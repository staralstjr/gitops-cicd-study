# 개념 02 — GitOps와 Argo CD

> "Git에 적힌 것이 정답이고, 클러스터를 그 상태에 맞춘다"는 배포 방식(GitOps)과
> 그것을 수행하는 엔진 Argo CD. 이 프로젝트에서 **수동 restart를 제거하는 핵심**이 여기 있다.

선행 개념: [concept-01-container-image-registry.md](concept-01-container-image-registry.md)

---

## 1. 한 문장 정의

**Argo CD** = Git 저장소를 지속적으로 감시하다가, 클러스터의 실제 상태를
Git에 선언된 상태와 동일하게 맞춰주는 **GitOps 배포 엔진**.

---

## 2. 배경 문제 — 명령형 배포의 한계

Kubernetes에 배포하는 전통적 방식은 사람이 직접 명령을 내리는 것이다.

```bash
kubectl apply -f deployment.yaml   # 사람이 클러스터에 직접 명령
```

팀 규모가 커지면 문제가 발생한다.

- **현재 클러스터 상태를 아무도 정확히 모른다** — 누가 언제 무엇을 apply 했는지 기록이 없다.
- **사람마다 클러스터 상태가 달라진다** — 각자 다른 명령을 실행.
- **수동 변경을 되돌릴 근거가 없다** — "원래 어떤 상태였는지"에 대한 단일 기준이 없다.

> 이것은 "손으로 `aws` 명령을 치지 말고 Terraform 코드로 관리하자"던 IaC의 문제의식과 동일하다.
> 명령형(imperative) → 선언형(declarative)으로의 전환이라는 같은 흐름이다.

---

## 3. 해결책 — GitOps

**핵심 원칙**

> 클러스터에 무엇을 띄울지를 전부 Git에 선언(YAML)해 두고,
> **Git을 유일한 정답(single source of truth)** 으로 삼는다.
> 실제 클러스터는 그 Git 상태에 **자동으로 수렴**한다.

```
[ 명령형 ]  사람 → kubectl apply → 클러스터        (기록 없음, 표류)

[ GitOps ] 사람 → Git에 YAML 커밋 → [Argo CD 감시] → 클러스터 자동 반영
                  └ 모든 변경이 Git 히스토리에 남음 (누가/언제/무엇을/왜)
```

**GitOps의 이점**

| 이점 | 설명 |
|------|------|
| 추적성 | 모든 배포 변경이 Git 커밋으로 남는다 |
| 재현성 | Git 상태만 있으면 동일한 클러스터를 복원 가능 |
| 롤백 | 이전 커밋으로 되돌리면 이전 배포 상태로 복귀 |
| 일관성 | 사람이 아니라 Git이 기준이므로 상태 표류가 없다 |

> Terraform이 "코드 → 클라우드"를 수렴시킨다면, Argo CD는 "Git → Kubernetes"를 수렴시킨다.
> 둘 다 **선언형 + 원하는 상태로 수렴(desired state reconciliation)** 이라는 동일한 철학이다.

---

## 4. Argo CD의 동작 원리 (Reconciliation Loop)

Argo CD는 클러스터 안에서 상주하며 다음 루프를 반복한다.

```
   ┌──────────────────────────────────────────────┐
   │ 1. Git 저장소를 주기적으로 확인                 │
   │    (매니페스트가 바뀌었는가?)                   │
   │                                              │
   │ 2. Desired State (Git에 선언된 상태)           │
   │    vs Live State (클러스터의 실제 상태) 비교     │
   │                                              │
   │ 3. 다르면(OutOfSync) → 클러스터를 Git에 맞춤(Sync)│
   └──────────────────────────────────────────────┘
                     ↑ 지속 반복
```

| 용어 | 의미 |
|------|------|
| **Desired State** | Git의 YAML에 선언된 "있어야 할" 상태 |
| **Live State** | 클러스터에 실제로 떠 있는 상태 |
| **Synced** | 둘이 일치 — 아무 작업도 하지 않음 |
| **OutOfSync** | 둘이 불일치 — Sync를 수행해 Git 상태로 맞춤 |

### 이 프로젝트에 대입

```
CI가 새 이미지 빌드/푸시
  → Git 매니페스트의 이미지 태그 변경 (:1f3b8ed → :217988d)
  → Argo CD가 OutOfSync 감지
  → 클러스터에 새 이미지로 롤링 업데이트
  → 사람의 수동 restart 불필요
```

**`latest` 태그가 문제였던 이유가 여기서 완결된다:**
이미지를 새로 push해도 매니페스트 텍스트(`:latest`)가 변하지 않으면
Argo CD는 계속 **Synced**로 판단해 아무 작업도 하지 않는다.
→ 그래서 사람이 수동 restart를 해야 했다.
→ **커밋 SHA 태그**로 매니페스트 텍스트를 바꾸면 OutOfSync가 발생해 자동 배포된다.

---

## 5. 동기화 정책 (syncPolicy)

Argo CD Application에 자동화 정책을 지정할 수 있다.

```yaml
syncPolicy:
  automated:
    selfHeal: true
    prune: true
```

| 옵션 | 의미 |
|------|------|
| **자동 동기화(automated)** | OutOfSync를 감지하면 사람 개입 없이 자동으로 Sync |
| **selfHeal** | 누군가 클러스터를 손으로 바꿔도 Git 상태로 **되돌린다** |
| **prune** | Git에서 삭제된 리소스를 클러스터에서도 제거한다 |

> **selfHeal은 "원하는 상태를 선언하면 시스템이 그 상태를 스스로 유지한다"는 패턴의 배포 버전이다.**
> AWS Auto Scaling Group이 인스턴스가 죽으면 자동으로 재생성해 desired capacity를 유지하던 것과
> 정확히 같은 사상이다. 이 self-healing 패턴은 클라우드 인프라 전반을 관통한다.

---

## 6. Argo vs Argo CD

- **Argo** = Kubernetes용 오픈소스 **도구 패밀리**의 이름.
- 그 패밀리 구성:

| 도구 | 역할 |
|------|------|
| **Argo CD** | GitOps 기반 지속적 배포(Continuous Delivery) — **이 프로젝트에서 사용** |
| Argo Workflows | 병렬 작업/파이프라인 실행 엔진 |
| Argo Rollouts | 카나리/블루그린 등 고급 배포 전략 |
| Argo Events | 이벤트 기반 트리거 |

이 프로젝트에서는 **Argo CD**만 사용한다. "CD"는 Continuous Delivery(지속적 배포)를 뜻한다.

---

## 7. Application 리소스 — 배포 명세서

Argo CD에게 "무엇을, 어디서, 어디로, 어떻게 배포할지" 알려주는 리소스.

```yaml
apiVersion: argoproj.io/v1alpha1
kind: Application
spec:
  source:                        # ① 감시할 Git 위치
    repoURL: <infra-repo>
    targetRevision: main
    path: apps/<project>/<app>   #    이 경로의 매니페스트를
  destination:                   # ② 배포 대상 클러스터/네임스페이스
    server: <k8s-api-server>
    namespace: <namespace>
  syncPolicy:                    # ③ 동기화 방식
    automated:
      selfHeal: true
      prune: true
```

### 핵심 함정

앱 디렉터리에 매니페스트(YAML)만 추가한다고 자동 배포되지 **않는다.**
Argo CD가 그 경로를 감시하도록 **Application 리소스를 별도로 등록**해야 한다.

- **앱 디렉터리** = 배포할 내용(Deployment/Service/ConfigMap/...)
- **Application 리소스** = "이 경로를 감시해 배포하라"는 등록

둘 다 존재해야 배포가 이루어진다.

---

## 8. 요약

1. **GitOps** = Git을 유일한 정답으로 삼고 클러스터를 Git 상태에 수렴시키는 방식.
2. **Argo CD** = Git을 감시하다 클러스터를 Git 상태로 자동 동기화하는 엔진.
3. 동작 = **Desired(Git) vs Live(클러스터) 비교 → 다르면 Sync**.
4. **selfHeal** = ASG 자가치유의 배포 버전 (원하는 상태를 스스로 유지).
5. **Argo**는 도구 패밀리, **Argo CD**는 그중 배포 담당.
6. **Application 리소스**가 있어야 해당 경로가 감시·배포된다 (디렉터리만으론 안 됨).
7. 이 프로젝트: **SHA 태그로 매니페스트를 변경 → OutOfSync → 자동 배포**로 수동 restart 제거.
