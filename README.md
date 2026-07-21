# 컨테이너 배포 파이프라인 학습 기록 (Docker · Kubernetes · GitOps)

컨테이너 기반 웹 서비스(Next.js 프론트엔드 + FastAPI 백엔드)의 **배포 파이프라인을
이해하고 직접 수행**하면서 정리한 개념·실습 기록. 나아가 이 흐름을 자동화(CI/CD)한다면
어떻게 설계할 수 있는지까지 **학습 차원에서** 탐구한다.

> ⚠️ **기록 원칙**
> 이 저장소는 **개념과 파이프라인 로직만** 기록한다.
> 실제 레지스트리 주소·저장소 URL·도메인·자격증명 등 식별 정보는 모두 일반화(`<placeholder>`)했으며,
> 시크릿은 어떤 형태로도 기록하지 않는다. 실제 작업은 별도의 업무용 저장소에서 진행된다.

---

## 🎯 범위

**① 현재 운영 중인 수동 배포 흐름을 이해하고 직접 수행한다.** (실제 업무)

```
개발 완료
  → docker build (linux/amd64)
  → :latest 태그로 레지스트리에 push
  → Argo CD에서 서비스 restart → 새 Pod가 최신 이미지 pull
  → 배포 확인
```

이 팀은 **의도적으로 수동 배포**를 사용한다(배포 통제·단순성). "자동화하지 않는 것"도 합리적 선택이다.

**② 이 흐름을 자동화한다면 어떻게 설계할 수 있는지 학습한다.** (개인 학습/포트폴리오, 실제 적용 X)

- `:latest`는 덮어쓰기 가능 → 매니페스트가 안 변함 → Argo CD가 감지 못 함 → 수동 restart의 근본 원인
- 커밋 SHA 불변 태그 + 매니페스트 자동 갱신으로 자동 배포가 가능해지는 원리
- → 상세: [concept-07](docs/concept-07-github-actions-ci-design.md)

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

## 🚦 진행 상황

| 단계 | 내용 | 상태 |
|------|------|------|
| **개념 학습** | 컨테이너·이미지·레지스트리·Dockerfile·K8s·GitOps·Kustomize | ✅ (concept 01~06) |
| **수동 배포 수행** | 백엔드 build → push → Argo CD restart 직접 경험 | ✅ (백엔드) / 프론트 예정 |
| **자동화 설계 탐구** | CI/CD로 자동화한다면? (학습용, 실제 적용 X) | ✅ 문서화 (concept 07) |

> 팀은 **수동 배포를 의도적으로 사용**한다. 실제 업무는 수동 배포를 능숙히 수행하는 것이고,
> 자동화(Phase 1/2)는 **"이렇게 자동화할 수 있다"는 학습·설계 기록**으로만 남긴다.

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
| [concept-07-github-actions-ci-design.md](docs/concept-07-github-actions-ci-design.md) | GitHub Actions CI 자동화 **설계 탐구**(학습용) · SHA 태그 · Phase 1/2 · 자동화 판단 |
