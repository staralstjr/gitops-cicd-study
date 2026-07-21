# GitOps 기반 CI/CD 파이프라인 구축 — 학습 및 설계 기록

컨테이너 기반 웹 서비스(Next.js 프론트엔드 + FastAPI 백엔드)의 배포를
**수동 → 자동(GitOps)** 으로 전환하는 과정에서 정리한 개념·설계 문서.

> ⚠️ **기록 원칙**
> 이 저장소는 **개념과 파이프라인 로직만** 기록한다.
> 실제 레지스트리 주소·저장소 URL·도메인·자격증명 등 식별 정보는 모두 일반화(`<placeholder>`)했으며,
> 시크릿은 어떤 형태로도 기록하지 않는다. 실제 작업은 별도의 업무용 저장소에서 진행된다.

---

## 🎯 목표

**"PR을 머지하면 프로덕션까지 자동 배포되는 파이프라인"** 을 만든다.

### 현재 (수동)

```
개발 완료
  → 개발자가 로컬에서 docker build
  → :latest 태그로 레지스트리에 push
  → Argo CD UI에서 수동으로 서비스 restart
  → 눈으로 배포 확인
```

**문제점**

1. 빌드/푸시가 개발자 로컬 환경에 의존 (사람마다 환경 차이, 실수 여지)
2. `:latest` 태그는 **덮어쓰기 가능**하므로 매니페스트 내용이 변하지 않음
   → Argo CD가 변경을 감지하지 못함 → **수동 restart가 필요한 근본 원인**
3. 현재 운영 중인 버전이 **어느 커밋인지 추적 불가**, 롤백도 어려움

### 목표 (자동)

```
PR 머지 (main)
  → CI(GitHub Actions)가 이미지 빌드
  → 커밋 SHA 태그로 레지스트리에 push        ← 불변 태그
  → 인프라 저장소의 매니페스트 이미지 태그 갱신
  → Argo CD가 Git 변경을 감지해 자동 동기화
  → 롤링 업데이트로 신규 버전 반영
```

---

## 🗺️ 아키텍처 (일반화)

```
┌──────────────────────────────────────────────────────────┐
│ ① 애플리케이션 저장소  (web-frontend / web-api)             │
│    소스코드 → 이미지 빌드 → 레지스트리 push        = CI     │
└──────────────────────────────────────────────────────────┘
                     │  이미지(고유 태그)
                     ▼
        ┌──────────────────────────────┐
        │  컨테이너 레지스트리 (private) │
        └──────────────────────────────┘
                     │  pull (imagePullSecret 필요)
                     ▼
┌──────────────────────────────────────────────────────────┐
│ ② 인프라 저장소  (k8s-infra)                               │
│    Deployment / Service / ConfigMap / SealedSecret        │
│    + Kustomization + Argo CD Application         = CD     │
└──────────────────────────────────────────────────────────┘
                     │  Argo CD가 Git을 감시 → 클러스터 동기화
                     ▼
              ┌─────────────────┐
              │  Kubernetes     │
              └─────────────────┘
```

**핵심 원칙 (GitOps):** 클러스터를 사람이 직접 조작하지 않는다.
**Git 저장소의 선언이 "정답"이고, Argo CD가 클러스터를 그 상태에 맞춘다.**

---

## 🚦 진행 계획

| 단계 | 내용 | 프로덕션 영향 | 상태 |
|------|------|----------------|------|
| **개념 학습** | 컨테이너·이미지·레지스트리·K8s·GitOps | 없음 | 진행 중 |
| **Phase 1 — CI** | 머지 시 이미지 자동 빌드 + 레지스트리 push | **없음** (배포는 기존 수동 유지) | 예정 |
| **Phase 2 — CD** | 매니페스트 태그 자동 갱신 → Argo CD 자동 동기화 | 있음 (신중히, 승인 후) | 예정 |

> 위험도가 낮은 **CI부터** 붙인다. Phase 1은 이미지를 창고에 쌓기만 하므로
> 기존 배포 흐름을 전혀 건드리지 않는다.

---

## 🧩 구성 요소

| 이름 | 역할 |
|------|------|
| **컨테이너 레지스트리** | 빌드된 이미지를 보관하는 비공개 저장소 |
| **Kubernetes** | 컨테이너가 실제로 실행되는 오케스트레이션 플랫폼 |
| **Argo CD** | Git을 감시해 클러스터를 선언된 상태로 동기화하는 GitOps 엔진 |
| **Kustomize** | 여러 매니페스트를 하나의 배포 단위로 조합 |
| **Sealed Secrets** | 민감 정보를 암호화해 Git에 안전하게 보관 |
| **GitHub Actions** | CI 자동화 (빌드·태깅·푸시) |

---

## 📚 문서

| 문서 | 내용 |
|------|------|
| [concept-01-container-image-registry.md](docs/concept-01-container-image-registry.md) | 컨테이너 · 이미지 · 레이어 · 레지스트리 · 태그 |
| [concept-02-gitops-argocd.md](docs/concept-02-gitops-argocd.md) | GitOps · Argo CD · 동기화 루프 · selfHeal · Application 리소스 |
| [concept-03-dockerfile-anatomy.md](docs/concept-03-dockerfile-anatomy.md) | Dockerfile 해부 · 레이어 캐싱 · 멀티스테이지 · 경량 이미지 |
| [concept-04-deploy-topology-and-environments.md](docs/concept-04-deploy-topology-and-environments.md) | 앱/인프라 레포 분리 · 환경 · 빌드≠배포 · 브랜치의 의미 · 이름으로 환경 넘겨짚지 않기 |
| [concept-05-kubernetes-resources.md](docs/concept-05-kubernetes-resources.md) | Pod · Deployment · Service · ConfigMap · Secret/SealedSecret · AWS 대응표 |
| [concept-06-kustomize.md](docs/concept-06-kustomize.md) | Kustomize · base+overlay · 이미지 태그 치환(Phase 2 핵심) · Argo CD 연결 |

*(이후 GitHub Actions 워크플로 상세 문서 추가 예정)*
