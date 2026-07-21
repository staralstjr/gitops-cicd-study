# 개념 06 — Kustomize (매니페스트 조합 · 환경별 오버레이 · 이미지 태그 치환)

> 여러 K8s 매니페스트를 하나의 배포 단위로 묶고, 환경별 차이만 덮어쓰는 도구.
> `kustomization.yaml`의 이미지 태그 치환 기능이 **이 프로젝트 자동화(Phase 2)의 핵심 지점**이다.

선행 개념: [concept-05-kubernetes-resources.md](concept-05-kubernetes-resources.md), [concept-02-gitops-argocd.md](concept-02-gitops-argocd.md)

---

## 1. 배경 문제 (1) — 매니페스트가 여러 개다

앱 하나를 배포하려면 매니페스트가 여러 개 필요하다.

```
deployment.yaml
service.yaml
configmap.yaml
sealed-secrets-app.yaml
```

이를 매번 `kubectl apply -f ...`로 하나씩 적용하는 것은 번거롭고 실수하기 쉽다.

## 2. 배경 문제 (2) — 환경마다 대부분 같고 일부만 다르다

dev와 prod 매니페스트는 대부분 동일하고 일부만 다르다.

| 항목 | dev | prod |
|------|-----|------|
| 이미지 | 동일 | 동일 |
| 포트 | 8000 | 8000 |
| **replicas** | **1** | **4** |
| **ConfigMap 값** | dev 설정 | prod 설정 |

환경별로 YAML 전체를 복사하면 90%가 중복되어, 공통 부분 수정 시 모든 사본을 고쳐야 한다(DRY 위반).

---

## 3. Kustomize의 해법 — base + overlay

공통 부분은 `base`에 한 번만 작성하고, 환경별로 **다른 부분만** overlay에서 덮어쓴다.

```
base/
 ├─ deployment.yaml        # replicas: 1 (기본)
 ├─ service.yaml
 └─ kustomization.yaml     # 이 리소스들을 묶음

overlays/
 ├─ dev/
 │   └─ kustomization.yaml  # base 참조 + replicas=1
 └─ prod/
     └─ kustomization.yaml  # base 참조 + replicas=4 로 덮어씀
```

- 공통 수정은 `base`만 고치면 모든 환경에 반영된다.
- 환경별 차이는 각 overlay에만 존재한다.

> Terraform의 `variable`과 같은 목적(중복 제거). Kustomize는 "공통 base + 환경별 오버레이"로 이를 달성한다.
> (단, 이 프로젝트의 인프라 레포는 앱 디렉터리 단위 구조를 쓰며, overlay 구조는 팀 컨벤션에 따라 다를 수 있다.)

---

## 4. kustomization.yaml의 두 가지 핵심 역할

### ① 리소스 묶기

```yaml
resources:
  - deployment.yaml
  - service.yaml
  - configmap.yaml
  - sealed-secrets-app.yaml
```

여러 YAML을 하나의 배포 단위로 조합한다. (앱 디렉터리의 `kustomization.yaml`이 이 역할)

### ② 이미지 태그 치환 (이 프로젝트의 핵심)

```yaml
images:
  - name: <registry>/<image-name>
    newTag: 1f3b8ed        # 이미지 태그를 선언적으로 치환
```

Deployment의 이미지 태그를 여기서 덮어쓸 수 있다.

**Phase 2 자동화의 연결점:**
`:latest` 대신 커밋 SHA 태그를 쓰면, CI가 이미지를 빌드한 뒤 이 `newTag`를 **새 SHA로 자동 변경**하고
인프라 레포에 커밋한다. 그러면 매니페스트 텍스트가 바뀌어 **Argo CD가 변경을 감지 → 자동 배포**한다.
→ `:latest`로 인한 수동 restart가 사라진다(개념 01, 04와 직결).

---

## 5. Argo CD와의 연결

Argo CD Application의 `source.path`가 가리키는 디렉터리에 `kustomization.yaml`이 있으면:

1. Argo CD가 그 `kustomization.yaml`을 읽어 모든 리소스를 조합(render)한다.
2. 렌더링 결과를 클러스터의 실제 상태와 비교한다.
3. 차이가 있으면 동기화(Sync)한다.

> **Kustomize = 조립 설명서, Argo CD = 그 설명서대로 조립해 클러스터에 반영하는 엔진.**

---

## 6. 두 종류의 kustomization.yaml (혼동 주의)

이 프로젝트의 인프라 레포에는 목적이 다른 kustomization.yaml이 두 곳에 존재한다.

| 위치 | 역할 |
|------|------|
| `apps/<project>/<app>/kustomization.yaml` | **앱 하나**의 리소스(deployment/service/configmap...)를 묶음 |
| `addons/argocd/apps/kustomization.yaml` | **Argo CD Application 리소스들**을 묶음(앱마다 하나씩 등록) |

- 앞: "이 앱을 어떻게 조립하는가"
- 뒤: "Argo CD가 감시할 Application 목록"

앱 디렉터리만 만들고 뒤쪽(Application 등록)을 빠뜨리면 배포되지 않는다(개념 02의 함정).

---

## 7. 요약

1. Kustomize는 여러 매니페스트를 묶고(`resources`), 환경별 차이만 덮어쓰는(overlay) 도구다.
2. **base(공통) + overlay(환경별)** 로 중복을 제거한다(Terraform 변수와 같은 목적).
3. `kustomization.yaml`의 **`images.newTag`** 치환이 **Phase 2 자동 배포의 핵심 지점**이다.
4. Argo CD가 kustomization을 렌더링해 클러스터에 동기화한다.
5. 앱용 kustomization과 Argo CD Application용 kustomization은 역할이 다르다.
