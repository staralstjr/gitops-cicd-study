# 개념 03 — Dockerfile 해부 (레이어 캐싱 · 멀티스테이지 · 경량 이미지)

> 실제 두 서비스(Python 백엔드 / Next.js 프론트엔드)의 Dockerfile을 분석하며
> 개념 01의 "레이어"가 실물로 어떻게 적용되는지, CI가 무엇을 실행하게 되는지 이해한다.
> (코드는 일반화하여 구조와 원리만 기록)

선행 개념: [concept-01-container-image-registry.md](concept-01-container-image-registry.md)

---

## 1. 빌드 시 명령 vs 실행 시 명령 (RUN vs CMD)

| 지시어 | 언제 실행 | 결과 | 예 |
|--------|-----------|------|-----|
| `RUN` | **이미지 빌드 시** | 레이어로 굳어짐 | 라이브러리 설치 |
| `CMD` | **컨테이너 시작 시** | 컨테이너의 메인 프로세스 (하나만) | 서버 실행 |
| `COPY` | 빌드 시 | 파일을 이미지로 복사, 레이어 생성 | 소스 복사 |
| `ENV` | 빌드/실행 | 환경 변수 설정 | 런타임 설정 |
| `WORKDIR` | 빌드/실행 | 작업 디렉터리 지정 | `/app` |
| `EXPOSE` | (문서화) | 사용 포트 명시(실제 개방 아님) | 8000 |

`RUN pip install`은 빌드하면서 미리 설치해 레이어로 굳히고,
`CMD ["uvicorn", ...]`은 컨테이너가 뜰 때마다 서버를 실행한다.

---

## 2. 단일 스테이지 예 (Python 백엔드)

```dockerfile
FROM python:3.10-slim              # 1층: 베이스 이미지 (경량)
ENV PYTHONDONTWRITEBYTECODE=1 ...  # 런타임 환경 변수
WORKDIR /app
COPY requirements.txt ./           # 2층: 의존성 목록만 먼저 복사
RUN pip install -r requirements.txt # 3층: 라이브러리 설치 (무거운 층)
COPY . .                           # 4층: 애플리케이션 소스 복사
ENV PYTHONPATH=/app
EXPOSE 8000
RUN useradd -m appuser             # 비루트 사용자 생성
USER appuser                       # 권한 낮춰 실행
CMD ["uvicorn", "main:app", "--host", "0.0.0.0", "--port", "8000"]
```

### 포인트 1 — 레이어 캐싱 최적화가 실제로 적용됨

```dockerfile
COPY requirements.txt ./     # 의존성 목록만 먼저
RUN pip install -r ...       # 설치
COPY . .                     # 소스는 그 다음
```

- 소스코드는 자주 바뀌지만 `requirements.txt`는 거의 안 바뀐다.
- 의존성 설치를 **앞 레이어**에 두면, 소스만 변경됐을 때 무거운 설치 레이어를 **캐시로 재사용**한다.
- 순서를 반대로 두면 코드 한 줄만 고쳐도 매번 전체 의존성을 재설치하게 된다.

→ 개념 01의 "자주 바뀌는 것을 뒤에 둔다"가 실물로 구현된 사례.

### 포인트 2 — 경량 베이스 이미지 (`-slim`)

| 태그 | 크기(대략) | 특징 |
|------|-----------|------|
| `python:3.10` | ~1GB | 각종 도구 포함, 무겁다 |
| `python:3.10-slim` | ~150MB | 필수만 남긴 경량 |
| `python:3.10-alpine` | ~50MB | 극단적 경량(호환성 주의) |

이미지가 작을수록 push/pull이 빠르고 배포가 빨라지며 공격 표면도 줄어든다.

### 포인트 3 — 비루트 실행 (보안)

```dockerfile
RUN useradd -m appuser
USER appuser
```

컨테이너는 기본적으로 root로 프로세스를 실행한다. 앱이 침해될 경우 피해를 줄이기 위해
**권한이 낮은 일반 사용자로 실행**하는 것이 관례다. (최소 권한 원칙)

---

## 3. 멀티스테이지 빌드 예 (Next.js 프론트엔드)

`FROM`이 세 번 등장한다. 각 `FROM ... AS <name>`이 하나의 **스테이지**다.

```dockerfile
FROM node:22-alpine AS deps      # 1단계: 의존성 설치
...
FROM node:22-alpine AS builder   # 2단계: 프로덕션 빌드
COPY --from=deps /app/node_modules ./node_modules
COPY . .
RUN npm run build
...
FROM node:22-alpine AS runner    # 3단계: 최종 실행 이미지
COPY --from=builder /app/.next/standalone ./       # 빌드 결과물만 가져옴
COPY --from=builder /app/.next/static ./.next/static
COPY --from=builder /app/public ./public
USER nextjs
CMD ["node", "server.js"]
```

### 핵심 아이디어 — "짓는 곳과 사는 곳을 분리"

- **빌드 시점**: Node 전체 + 개발 의존성 + 소스코드 + 빌드 도구가 필요 (무겁다)
- **실행 시점**: 빌드 결과물만 있으면 된다 (가볍다)

멀티스테이지는 앞 단계에서 무거운 작업을 수행하고,
**마지막 단계에는 `--from=<stage>`로 결과물만 복사**한다.
소스코드·개발 의존성·빌드 도구는 최종 이미지에 포함되지 않는다.

| | 멀티스테이지 미사용 | 멀티스테이지 사용 |
|--|---------------------|-------------------|
| 최종 이미지 크기 | 큼 | **작음** |
| 포함물 | 빌드 도구·소스 전부 | **실행 결과물만** |
| 보안 | 소스/도구 노출 | 최소 포함 |

### 관련 기법

- **`alpine`**: 매우 작은 리눅스 배포판(~5MB). 극단적 경량화에 사용.
  일부 라이브러리 호환 문제로 `libc6-compat` 등 보완이 필요할 수 있다.
- **`standalone` 출력**: Next.js가 실행에 필요한 최소 파일만 담은 독립 패키지를 생성하는 기능.
  `node_modules` 전체를 담지 않아 이미지가 크게 줄어든다.

---

## 4. 두 Dockerfile 비교

| 항목 | 백엔드(Python) | 프론트(Next.js) |
|------|----------------|-----------------|
| 베이스 | `python:3.10-slim` | `node:22-alpine` |
| 구조 | 단일 스테이지 | 멀티스테이지(3단계) |
| 캐시 최적화 | 의존성 먼저 복사 | 의존성 먼저 복사 |
| 비루트 실행 | `appuser` | `nextjs` |
| 실행 명령 | `uvicorn main:app` | `node server.js` |
| 포트 | 8000 | 3000 |

---

## 5. CI와의 연결

이 Dockerfile들이 곧 **CI(GitHub Actions)가 실행할 대상**이다. Phase 1 워크플로의 뼈대는:

```
1. 레포 코드 체크아웃
2. docker build  (이 Dockerfile 사용, --platform linux/amd64)
3. 커밋 SHA로 태깅
4. 레지스트리에 push
```

- **`--platform linux/amd64`**: 개발 노트북(ARM64)과 운영 노드(x86_64)의
  아키텍처 차이로 인한 실행 오류를 방지 (개념 01 참조).
- 잘 구성된 Dockerfile(레이어 캐싱)은 CI 빌드 시간을 직접 단축시킨다.

---

## 6. 요약

1. `RUN`은 빌드 시, `CMD`는 실행 시. 둘의 시점이 다르다.
2. **의존성 설치를 소스 복사보다 앞에** 두어 레이어 캐시를 활용한다.
3. `-slim` / `alpine` 등 **경량 베이스**로 이미지 크기·공격 표면을 줄인다.
4. **비루트 사용자**로 실행해 침해 시 피해를 최소화한다.
5. **멀티스테이지 빌드**로 빌드 환경과 실행 환경을 분리해 최종 이미지를 경량·안전하게 만든다.
6. 이 Dockerfile들이 CI 파이프라인의 빌드 대상이며, `--platform linux/amd64`가 필요하다.
