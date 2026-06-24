---
title: "Docker CI/CD · GHCR 아키텍처 정리"
date: 2026-06-17 10:00:00 +0900
categories: [DevOps]
tags: [Docker, CICD, GHCR, GitHubActions]
---

> 어떤 서버의 CD(배포)가 5분 → 20분으로 느려진 걸 파다가
> "빌드를 어디서 하느냐"가 원인임을 발견하고, GHCR 기반 구조로 바꾸려고 정리한 내용.
> 핵심 질문: **"코드가 머지되면 무슨 일이 일어나서 서버에 올라가나"**

---

## 0. 출발점 — 왜 CD가 20분이나 걸렸나

예를 들어 어떤 `dev_cd.yml`은 EC2에 SSH로 들어가서 이 한 줄을 실행한다:
```bash
docker compose --profile app up -d --build app   # ← --build 가 범인
```
- `--build`가 **그 1.9GB짜리 빈약한 EC2 위에서** Gradle 빌드 + 도커 이미지 빌드를 돌림.
- 서버 메모리가 빠듯 → 스왑 → 빌드가 기어감 → 20분.
- → **결론: 빌드를 서버에서 하지 말고, 빵빵한 CI 러너(GitHub Actions / GitLab Runner)에서 하고 서버엔 완성품만 보내자.** 그 "완성품 전달 창고"가 레지스트리(GHCR 등).

> 참고: GitLab CI/CD도 같은 구조로 짤 수 있다. `build-and-image` 잡이 러너에서
> `./gradlew clean war` + `docker build`를 하고, `deploy` 잡이 `docker run` 하는 식이면
> 이미 "러너에서 빌드"에 가깝다. 반대로 "서버에서 빌드"하면 느려진다. 핵심 교훈은 동일.

---

## 1. 용어 먼저 — 이미지 / 컨테이너 / 레지스트리

| 용어 | 뜻 | 비유 |
|---|---|---|
| **이미지(Image)** | 앱+런타임을 통째로 묶은 **완성 패키지(읽기 전용 템플릿)** | 붕어빵 **틀** |
| **컨테이너(Container)** | 이미지를 실행한 **살아있는 프로세스** | 틀로 찍어낸 **붕어빵** |
| **레지스트리(Registry)** | 이미지를 올려두고 받아가는 **클라우드 저장소** | 붕어빵 틀 **보관 창고** |
| **GHCR** | GitHub이 운영하는 레지스트리(`ghcr.io`) | GitHub표 창고 |

```
코드  → GitHub/GitLab  (git push / git pull)
이미지 → 레지스트리      (docker push / docker pull)   ← 구조가 똑같다
```

### GHCR (GitHub Container Registry)란
- 주소 형식: `ghcr.io/<GitHub계정>/<이미지명>:<태그>`
  - 예: `ghcr.io/my-org/my-app:latest`, `...:abc1234`(커밋 SHA)
- GitHub에 통합 → **`GITHUB_TOKEN`(Actions가 자동 발급)으로 무료 인증.** 별도 가입·시크릿 불필요.
- private 저장소면 이미지도 private. 개인/소규모는 무료 범위 충분.
- 대안: Docker Hub(공용), AWS ECR, GitLab Container Registry 등. 쓰는 플랫폼에 통합된 걸 쓰면 편함.

---

## 2. CI vs CD — 헷갈리지 말 것

| | CI (Continuous Integration) | CD (Continuous Deployment) |
|---|---|---|
| 뜻 | 통합·검증 | 배포 |
| 언제 | PR이 **열릴 때** | PR이 **머지될 때** |
| 하는 일 | 빌드 + **테스트** + 규칙 검사 | 실제 **서버에 배포** |
| 어디서 | CI 러너 | (보통) 러너 → SSH로 서버 |
| 통과 못 하면 | 머지 막음(품질 게이트) | 배포 실패 |

> CI = "이 코드 합쳐도 돼?"를 검증, CD = "검증된 코드를 실제로 띄우기".

---

## 3. 현재 아키텍처 (AS-IS) — 빌드가 서버에 있음 ❌

```
PR 머지 (→ dev)
   │
   ▼
GitHub Actions 러너 (클라우드)   ← 거의 놀고 SSH만 함
   │  ssh ─────────────────┐
   ▼                       ▼
              ┌──────────────────────────────────┐
              │  EC2 (1.9GB, 빈약)                 │
              │  git reset --hard origin/dev       │
              │  docker compose up -d --build app  │
              │     ├─ ./gradlew build  ← 무거움 💥 │  ← 여기서 빌드
              │     └─ docker build     ← 무거움 💥 │     메모리부족→스왑→20분
              │  docker run                        │
              └──────────────────────────────────┘
```
문제: **무거운 빌드를 가장 약한 장비가 떠안음.** compose.yaml의 app 서비스가 `build:`로 정의돼 있어서 매 배포가 곧 빌드.

---

## 4. 목표 아키텍처 (TO-BE) — 레지스트리 경유 ✅

```
PR 머지 (→ dev)
   │
   ▼  ① CI 러너 (클라우드, 빵빵) ──────────────────────────────┐
   │     docker build  (멀티스테이지: Gradle 빌드까지 여기서)     │
   │     docker push   → ghcr.io/<org>/<image>:<태그>          │
   └──────────────────────────────┬───────────────────────────┘
                                   ▼
                       ┌────────────────────────┐
                       │   레지스트리 (GHCR)       │  ← 이미지 창고
                       │   <image>:latest        │     버전별 보관
                       │   <image>:<sha>         │  ← 옛 버전 = 롤백용
                       └───────────┬────────────┘
                                   │ pull
   ② 서버 (SSH, 가벼움) ◄──────────┘
        docker compose pull app      ← 완성 이미지 다운로드만
        docker compose up -d app     ← --build 없음! 실행만
```

### 비유
```
CI 러너 = 공장(제조)  →  레지스트리 = 물류창고(완제품·버전별)  →  서버 = 매장(받아서 진열만)
```
서버는 빌드 0회 → CD **20분 → 1~2분**, 빌드 중 메모리 폭증도 사라짐.

---

## 5. 핵심 개념 — 멀티스테이지 빌드 & 레이어 캐시

### 멀티스테이지 빌드 (Dockerfile)
```dockerfile
# Build stage — JDK 풀버전. 여기서 Gradle 빌드
FROM eclipse-temurin:21-jdk AS build
COPY gradle gradlew ... ./
RUN ./gradlew dependencies   # 의존성 (소스 안 바뀌면 캐시됨)
COPY src ./src
RUN ./gradlew build -x test  # JAR 생성

# Runtime stage — JRE alpine(가벼움). 빌드 결과 JAR만 복사
FROM eclipse-temurin:21-jre-alpine
COPY --from=build /app/build/libs/*.jar app.jar
ENTRYPOINT ["sh","-c","java $JAVA_OPTS -jar app.jar"]
```
- 빌드용(무거운 JDK)·실행용(가벼운 JRE) 분리 → 최종 이미지 작고, **어디서 빌드해도 결과 동일**(이식성↑).
- → 빌드 위치만 옮기면 됨(서버 → CI 러너).

### 레이어 캐시 (왜 이 순서로 COPY하나)
도커는 각 명령(레이어)을 캐시하고 **바뀐 줄부터 아래만 다시 실행**한다.
```
COPY gradle/gradlew      ← 거의 안 바뀜 → 거의 항상 캐시 hit
RUN  gradlew dependencies ← build.gradle 안 바뀌면 캐시 hit (의존성 재다운로드 X)
COPY src                 ← 자주 바뀜
RUN  gradlew build       ← 소스 바뀌면 여기만 다시
```
- **자주 바뀌는 걸 아래쪽에** 둬서 위쪽(무거운 의존성 다운로드)을 최대한 재사용.
- ⚠️ `build.gradle` 바뀌면 의존성 레이어 캐시 깨짐 → 전체 재다운로드 → 그 배포는 느려짐.

---

## 6. 무엇을 바꿔야 하나 (변경 포인트)

> 코드/Dockerfile은 그대로. **빌드 위치와 전달 방식만** 바꾼다.

1. **CD 워크플로우(러너 측)**: SSH 빌드 대신 러너에서 `docker build` → `docker push <레지스트리>`
   (GHCR 로그인은 `docker/login-action` + `GITHUB_TOKEN`)
2. **compose.yaml(app 서비스)**: `build:` → `image: ghcr.io/<org>/<image>:<tag>`
3. **서버 측 스크립트**: `--build` 제거 → `docker compose pull app` + `docker compose up -d app`
4. **인증**: private 이미지면 서버에서도 `docker login <레지스트리>` 1회(토큰) 필요.

### 왜 JAR scp가 아니라 레지스트리(이미지)인가
- app이 **compose 네트워크 안 컨테이너 이름으로 DB/MinIO와 통신** → JAR 단독 실행 불가.
- JAR만 보내면 결국 서버에서 다시 이미지를 말아야 함 → "서버 빌드 제거" 목표 위반.
- → **완성 이미지(레지스트리)** 가 유일하게 목표를 만족.

---

## 7. 요약 (한 장)

| 항목 | 핵심 |
|---|---|
| 문제 | CD가 빈약한 서버(1.9GB)에서 `--build` → 스왑 쓰래싱 → 20분 |
| 이미지/컨테이너 | 이미지=틀(완성 패키지), 컨테이너=실행체 |
| 레지스트리/GHCR | 이미지 보관 창고. GHCR=GitHub표, `GITHUB_TOKEN`으로 무료 인증 |
| CI vs CD | CI=검증(PR 열림), CD=배포(PR 머지) |
| AS-IS | 러너는 SSH만, 서버가 빌드+실행 다 함 ❌ |
| TO-BE | 러너가 build+push → 레지스트리 → 서버는 pull+run만 ✅ |
| 멀티스테이지 | 빌드(JDK)·실행(JRE) 분리 → 이식성↑, 빌드 위치 옮겨도 결과 동일 |
| 레이어 캐시 | 자주 바뀌는 건 아래로. build.gradle 바뀌면 의존성 재다운로드 |
| 효과 | 20분 → 1~2분, 서버 메모리 압박 제거, 태그로 롤백 |

> 한 줄: **"공장(러너)에서 만들어 창고(레지스트리)에 넣고, 매장(서버)은 받아서 진열만."**
> 빌드라는 무거운 일을 빈약한 서버에서 떼어내는 게 전부다.

---

## 관련
- 서버 메모리/스왑/오버커밋 배경 → 서버 운영 글
- JVM 컨테이너 메모리 → JVM 메모리·튜닝 글
- CPU·메모리 계층 → CPU·메모리 계층 글
