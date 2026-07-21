# 개념 05 — 쿠버네티스 핵심 리소스 (Pod · Deployment · Service · ConfigMap · Secret)

> 인프라 레포(k8s-infra)의 매니페스트를 구성하는 핵심 리소스들.
> Argo CD가 관리하고, "서비스 restart" 시 실제로 움직이는 대상이다.
> AWS 3-tier(EC2/ASG/ALB/RDS)에서 배운 개념과 1:1로 대응된다.

선행 개념: [concept-02-gitops-argocd.md](concept-02-gitops-argocd.md), [concept-04-deploy-topology-and-environments.md](concept-04-deploy-topology-and-environments.md)

---

## 0. Kubernetes의 기본 사상

컨테이너를 대신 관리하는 오케스트레이터. "이런 상태였으면 좋겠다"를 YAML로 **선언**하면
K8s의 컨트롤러들이 **실제 상태를 그 선언에 수렴**시킨다. (Terraform·ASG와 같은 desired-state 사상)

---

## 1. Pod — 최소 실행 단위

- 컨테이너 1개(또는 긴밀히 결합된 여러 개)를 감싼 K8s의 최소 배포 단위.
- 빌드한 이미지가 실제로 실행되는 곳.
- **Ephemeral(일시적):** 언제든 죽고 교체될 수 있으며, 교체되면 IP도 바뀐다.

> 대응: **Pod ≈ EC2 인스턴스** (실행되는 하나의 단위). 컨테이너 이미지 ≈ AMI.

---

## 2. Deployment — Pod 무리 관리자

Pod를 직접 만들지 않고, Deployment에 "이 이미지로 replicas N개 유지"를 선언한다.

```yaml
spec:
  replicas: 3                       # 유지할 Pod 수
  selector:
    matchLabels: { app: web-api }
  template:                         # 찍어낼 Pod의 설계도
    metadata:
      labels: { app: web-api }
    spec:
      containers:
        - name: web-api
          image: <registry>/<image>:<tag>
```

- **개수 유지 + 자가 치유:** Pod가 죽으면 자동으로 새로 생성.
- **롤링 업데이트:** 이미지/설정이 바뀌면 Pod를 하나씩 교체(무중단).
- 내부 구조: **Deployment → ReplicaSet → Pod** (ReplicaSet이 개수 유지 담당).

> 대응: **Deployment = Auto Scaling Group.** desired 수 유지, 자가 치유, 롤링 업데이트가 동일.

### "서비스 restart"의 실체

새 이미지를 레지스트리에 push한 뒤 restart하면, Deployment가 Pod를 새로 생성하고
새 Pod가 레지스트리에서 최신 이미지를 pull해 실행한다 → 신규 버전 반영.
(`:latest`처럼 태그가 그대로면 매니페스트가 안 변하므로 수동 restart가 필요해진다 — 개념 01)

---

## 3. Service — 고정된 네트워크 진입점

Pod는 교체되며 IP가 바뀌므로, 안정적인 접속 지점이 필요하다. Service가 이를 제공한다.

```yaml
spec:
  selector:
    app: web-api        # 이 라벨을 가진 Pod들을 대상으로
  ports:
    - port: 8000
  type: ClusterIP       # 클러스터 내부 전용(기본)
```

- **고정 가상 IP + DNS 이름**을 제공하고, 대상 Pod들로 **부하 분산**한다.
- `selector`(라벨)로 대상 Pod 집합을 찾는다.
- 종류: **ClusterIP**(내부 전용, 기본) / **NodePort** / **LoadBalancer**(외부 노출).

> 대응: **Service = ALB + Target Group.** 고정 진입점 + 부하 분산.
> `selector`(라벨로 Pod 선택) ≈ Target Group의 대상 등록.

---

## 4. ConfigMap — 비민감 설정 주입

민감하지 않은 환경 변수/런타임 설정을 이미지 밖으로 분리해 Pod에 주입한다.

```yaml
# ConfigMap
data:
  ENVIRONMENT: "production"
  LOG_LEVEL: "info"
---
# Deployment에서 주입
envFrom:
  - configMapRef: { name: web-api-config }
```

- 같은 이미지가 환경별로 다르게 동작할 수 있게 한다(주입되는 값이 다르므로).
- **build once, inject config at runtime** 원칙의 실현.

> 대응: EC2 user_data / SSM 파라미터로 런타임 설정을 주입하던 것과 유사.

---

## 5. Secret & Sealed Secret — 민감 설정

ConfigMap과 같지만 민감 데이터(DB 비밀번호, API 키)를 다룬다.

- **Secret:** 민감 데이터용. 단, 기본 Secret은 **base64 인코딩일 뿐 암호화가 아님** → Git에 평문처럼 노출 위험.
- **Sealed Secret:** `kubeseal`로 **암호화**하여, 암호화된 리소스를 Git에 안전하게 커밋.
  클러스터 내부의 Sealed Secrets 컨트롤러만 복호화해 실제 Secret으로 복원한다.

```
일반 Secret YAML  --kubeseal(암호화)-->  SealedSecret YAML  --(Git 커밋 대상)
                                              │ 클러스터 내부
                                              ▼
                                       컨트롤러가 복호화 → 실제 Secret → Pod에 주입
```

> 대응: RDS 비밀번호를 tfstate에 평문으로 두지 않으려던 고민의 K8s/GitOps식 해법.
> (AWS Secrets Manager와 목적이 같다: 비밀을 코드/평문으로 남기지 않기.)

---

## 6. Namespace & imagePullSecrets

- **Namespace:** 클러스터 안에서 리소스를 논리적으로 나누는 칸.
  관례적으로 접두사로 용도를 구분한다(예: `app-*`=애플리케이션, `addon-*`=플랫폼 부가기능).
- **imagePullSecrets:** 비공개 레지스트리에서 이미지를 pull할 때 클러스터가 쓰는 자격증명(예: `regcred-ncr`).
  개발자/CI뿐 아니라 **클러스터도 레지스트리 인증이 필요**하다는 개념 01의 내용이 여기서 구현된다.

---

## 7. 전체 조합

```
Namespace: <app-namespace>
 ├─ Deployment → Pod × N (이미지 실행, 자가치유, 롤링)
 │    ├─ imagePullSecrets: <regcred>   (레지스트리 인증)
 │    ├─ envFrom ConfigMap            (일반 설정 주입)
 │    └─ envFrom Secret(Sealed)        (민감 설정 주입)
 └─ Service → 위 Pod들 앞의 고정 진입점 + 부하분산
```

이 리소스 묶음을 **Kustomize**(`kustomization.yaml`)로 조합하고,
**Argo CD Application**(개념 02)이 해당 경로를 감시해 클러스터에 동기화한다.

---

## 8. AWS ↔ K8s 대응표

| AWS | Kubernetes | 역할 |
|-----|------------|------|
| AMI | 컨테이너 이미지 | 실행할 내용물 |
| EC2 인스턴스 | **Pod** | 실행 단위 |
| Auto Scaling Group | **Deployment** | 개수 유지·자가치유·롤링 |
| ALB + Target Group | **Service** | 고정 진입점·부하분산 |
| user_data / SSM | **ConfigMap** | 비민감 설정 주입 |
| Secrets Manager | **Secret / SealedSecret** | 민감 설정 주입 |
| VPC(격리) | **Namespace** | 논리적 격리 |

---

## 9. 요약

1. **Pod** = 최소 실행 단위(이미지가 도는 곳), 일시적.
2. **Deployment** = Pod 개수 유지·자가치유·롤링 (= ASG). restart는 Pod를 새로 만들어 최신 이미지를 pull.
3. **Service** = 고정 진입점 + 부하분산 (= ALB/Target Group).
4. **ConfigMap** = 비민감 설정 주입 (build once, inject at runtime).
5. **Secret/SealedSecret** = 민감 설정. SealedSecret은 암호화해 Git에 안전 커밋.
6. **Namespace/imagePullSecrets** = 논리적 격리 / 레지스트리 인증.
7. 이 리소스들이 Kustomize로 묶이고 Argo CD가 동기화한다.
