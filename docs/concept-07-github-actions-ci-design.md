# 개념 07 — GitHub Actions로 CI 자동화 (설계 탐구)

> **학습/설계용 문서.** 현재 팀은 배포를 **수동**(build → push → Argo CD restart)으로 운영하며,
> 이는 배포 통제·단순성 측면에서 의도된 선택이다. 이 문서는 "만약 이 흐름을 자동화한다면
> 어떻게 설계할 수 있는가"를 정리한 것으로, **실제 업무 레포에는 적용하지 않았다.**
> (일반화: 실제 레지스트리/조직명은 `<placeholder>`로 표기)

선행 개념: [concept-01](concept-01-container-image-registry.md), [concept-06-kustomize.md](concept-06-kustomize.md)

---

## 0. 자동화할 vs 하지 않을 판단

먼저 전제. **자동화가 항상 정답은 아니다.** 수동 배포를 유지하는 합리적 이유:

- **배포 통제:** 언제 무엇이 나갈지 사람이 직접 확인 후 실행 (의도치 않은 자동 배포 방지)
- **낮은 배포 빈도:** 자주 배포하지 않으면 파이프라인 유지 비용이 더 클 수 있음
- **단순성:** 자동화 파이프라인 자체도 관리 대상

→ "자동화하지 않기로 하는 것"도 엔지니어링 판단이다. 아래는 **자동화를 택했을 때의 설계**다.

---

## 1. 수동 흐름 → 자동화 매핑

현재 수동 절차와, 자동화 시 무엇이 대체되는지.

| 수동 | 자동화 후 |
|------|-----------|
| 개발자 로컬에서 `docker build` | GitHub Actions 러너(임시 리눅스)가 빌드 |
| 개발자가 키 입력해 `docker login` | GitHub Secrets의 키로 자동 로그인 |
| `:latest` 태그 | **커밋 SHA 태그** (불변, 추적 가능) |
| 개발자가 `docker push` | 자동 push |
| Argo CD에서 수동 restart | (Phase 2) 매니페스트 태그 자동 갱신 → Argo CD 자동 동기화 |

---

## 2. Phase 1 — CI 워크플로 (빌드 + 푸시)

`.github/workflows/build-and-push.yml` (일반화):

```yaml
name: Build and Push

on:
  push:
    branches: [<deploy-branch>]   # 통합 브랜치에 머지될 때
  workflow_dispatch:              # 수동 실행 버튼(테스트용)

env:
  REGISTRY: <registry>.kr.ncr.ntruss.com
  IMAGE_NAME: <image-name>

jobs:
  build-and-push:
    runs-on: ubuntu-latest        # GitHub 제공 임시 리눅스 러너
    steps:
      - name: Checkout
        uses: actions/checkout@v4

      - name: Set short SHA
        id: vars
        run: echo "short_sha=$(git rev-parse --short HEAD)" >> "$GITHUB_OUTPUT"

      - name: Set up Docker Buildx
        uses: docker/setup-buildx-action@v3

      - name: Log in to registry
        uses: docker/login-action@v3
        with:
          registry: ${{ env.REGISTRY }}
          username: ${{ secrets.NCR_USERNAME }}   # Access Key (코드에 없음)
          password: ${{ secrets.NCR_PASSWORD }}   # Secret Key (코드에 없음)

      - name: Build and push
        uses: docker/build-push-action@v6
        with:
          context: .
          platforms: linux/amd64          # 운영 노드(x86_64) 호환
          provenance: false
          push: true
          tags: |
            ${{ env.REGISTRY }}/${{ env.IMAGE_NAME }}:${{ steps.vars.outputs.short_sha }}
            ${{ env.REGISTRY }}/${{ env.IMAGE_NAME }}:latest
          cache-from: type=gha            # 레이어 캐시 재사용(빌드 가속)
          cache-to: type=gha,mode=max
```

### 핵심 포인트

- **트리거(`on`)**: 통합 브랜치 push 시 실행 + 수동 실행 버튼.
- **러너**: 매번 새 리눅스 → 로컬 환경 의존 제거(재현성). 매번 깨끗하므로 캐시를 명시적으로 살린다.
- **SHA 태그**: `:latest`와 함께 커밋 SHA 태그를 부여 → **불변·추적 가능**(개념 01의 핵심).
- **Secrets**: 자격증명을 코드에 넣지 않고 `${{ secrets.* }}`로 주입(Terraform tfvars와 같은 원칙).
- **캐시**: `type=gha`로 레이어 캐시를 GitHub에 저장/복원(개념 03의 레이어 캐싱과 직결).
- **`platforms: linux/amd64`**: 개발 ARM ↔ 운영 x86_64 아키텍처 차이 방지(개념 01).

---

## 3. Phase 2 — CD 다리 (매니페스트 태그 자동 갱신)

Phase 1만으로는 이미지가 레지스트리에 쌓일 뿐 배포되지 않는다.
자동 배포까지 가려면 **인프라 레포의 이미지 태그를 새 SHA로 자동 갱신**해야 한다.

```
CI가 이미지 빌드/푸시 (SHA 태그)
  → 인프라 레포의 kustomization.yaml `images.newTag`를 새 SHA로 수정
  → 인프라 레포에 커밋/푸시
  → 매니페스트 텍스트가 변경됨
  → Argo CD가 OutOfSync 감지 → 자동 동기화(배포)
```

핵심은 개념 06의 이 부분:

```yaml
# 인프라 레포의 kustomization.yaml
images:
  - name: <registry>/<image-name>
    newTag: <old-sha>   →   <new-sha>   # CI가 이 줄을 자동 갱신
```

- 이 방식이면 `:latest` + 수동 restart가 사라진다(개념 02, 04).
- 다만 앱 레포 CI가 **인프라 레포를 수정**해야 하므로, 크로스 레포 권한/토큰 설계가 필요하다.
- 태그가 매번 바뀌므로 **어느 커밋이 배포됐는지 추적·롤백**이 쉬워진다.

---

## 4. 프론트엔드의 추가 고려 (NEXT_PUBLIC)

백엔드는 설정을 런타임에 주입(ConfigMap/Secret)하므로 이미지가 환경 무관하다.
반면 Next.js의 `NEXT_PUBLIC_*` 변수는 **빌드 시점에 번들에 baked-in** 된다.

- 환경이 여러 개이고 환경별 값이 다르면, 프론트는 환경별로 빌드하거나 런타임 주입 기법이 필요하다.
- 단일 환경이라면 이 고려는 단순해진다.

---

## 5. 요약

1. 자동화는 선택이다 — 수동 배포도 통제·단순성 면에서 합리적 선택.
2. Phase 1(CI): 러너에서 빌드 → **SHA 태그** → 레지스트리 push. Secrets·캐시·platform이 핵심.
3. Phase 2(CD): 인프라 레포의 `images.newTag`를 자동 갱신 → Argo CD 자동 동기화.
4. 프론트는 `NEXT_PUBLIC`의 빌드타임 특성을 추가로 고려해야 한다.
