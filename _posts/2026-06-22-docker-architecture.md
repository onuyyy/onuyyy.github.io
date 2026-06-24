---
title: "Docker 전체 구조 · 원리 · 명령어 정리"
date: 2026-06-22 09:00:00 +0900
categories: [DevOps]
tags: [Docker, Container, Compose]
---

> 서버 메모리/CD 디버깅을 하다가 "도커가 대체 어떻게 도는 건가"를
> 전체적으로 파고든 내용 정리. (CI/CD·GHCR 노트의 *기반 개념*)
> 핵심 질문: **"왜 도커를 쓰고, 컨테이너는 어떻게 격리되며, 코드가 어떻게 서버까지 가나"**

---

## 1. 왜 도커를 쓰나 — "내 컴에선 됐는데요" 박멸

도커 이전: 서버마다 Java 버전·라이브러리·설정이 제각각 → "내 PC는 되는데 서버는 안 됨".
도커: **앱 + 실행에 필요한 모든 것(런타임·라이브러리·설정)을 하나의 "이미지"로 포장** → 어디서든 똑같이 실행.

이점 4가지:
1. **일관성(이식성)** — 어디서 돌려도 동일
2. **격리** — 컨테이너끼리 간섭 없음 (app 죽어도 postgres 멀쩡)
3. **빠른 배포** — 이미지 받아 실행만
4. **자원 제어** — 컨테이너별 메모리/CPU 한도 (`mem_limit`)

---

## 2. VM과의 차이 (핵심)

```
[가상머신(VM)]                    [도커(컨테이너)]
┌──────┬──────┬──────┐          ┌──────┬──────┬──────┐
│ 앱A  │ 앱B  │ 앱C  │          │ 앱A  │ 앱B  │ 앱C  │
│ Guest OS×3 (무거움)│          ├──────┴──────┴──────┤
├──────┴──────┴──────┤          │  Docker 엔진       │
│   Hypervisor       │          ├────────────────────┤
├────────────────────┤          │  호스트 OS (공유!)  │  ← OS 하나만
│   호스트 OS         │          ├────────────────────┤
│   하드웨어          │          │   하드웨어          │
└────────────────────┘          └────────────────────┘
```
- VM: 컨테이너마다 OS 통째 → 무겁고(수 GB) 느림(분 단위 부팅)
- 도커: **호스트 OS 커널 공유** → 가볍고(수십~수백 MB) 빠름(초 단위)
- → 그래서 1.9GB 작은 EC2에 컨테이너 4개(app·postgres·minio…)가 뜸. VM이면 불가능.

---

## 3. 격리 원리 — 메모리 얘기와 직접 연결

도커는 마법이 아니라 **리눅스 커널 기능 2개**를 쓴다.

| 커널 기능 | 역할 | 연결 예 |
|---|---|---|
| **namespace** | "보이는 것" 격리 — 프로세스·네트워크·파일시스템을 컨테이너마다 따로 보게 | 컨테이너마다 자기 PID·네트워크 |
| **cgroups** | "쓸 수 있는 양" 제한 — CPU·메모리 한도 | **`mem_limit: 512m`이 곧 cgroup 설정** |

> JVM이 "컨테이너 한도 512MB를 인식"한 것 = **cgroup 값을 읽어서** 자기 방 크기를 안 것.
> namespace = 격벽, cgroups = 사용량 미터기.
> (→ JVM 메모리·튜닝 글, 서버 운영 글)

---

## 4. 도커 아키텍처 — 누가 일하나

```
┌──────────────┐  요청   ┌─────────────────────┐
│ docker CLI    │ ──────► │ Docker 데몬(dockerd) │  ← 백그라운드 서비스(진짜 일꾼)
│ (docker run…) │         │  - 이미지 관리        │
└──────────────┘         │  - 컨테이너 생성/실행  │
                         │  - 네트워크/볼륨 관리  │
                         └──────────┬──────────┘
                                    │ pull/push
                                    ▼
                         ┌─────────────────────┐
                         │ 레지스트리 (Hub/GHCR) │  ← 이미지 창고
                         └─────────────────────┘
```
- **docker CLI**: 내가 명령 치는 도구(심부름꾼)
- **dockerd(데몬)**: 실제로 이미지 받고 컨테이너 만드는 백그라운드 서비스
- **레지스트리**: 이미지 보관소

---

## 5. 핵심 구성요소 6개

```
이미지(Image)      앱 포장본(읽기전용 틀)        ← docker build / pull
   ↓ 실행
컨테이너(Container) 이미지를 돌린 실행체          ← docker run
레이어(Layer)       이미지를 이루는 변경 단위      ← 캐시·재사용 단위
볼륨(Volume)        컨테이너 밖 영구 저장소        ← DB 데이터(postgres_data)
네트워크(Network)   컨테이너끼리 통신하는 가상망    ← app-shared
레지스트리          이미지 창고                   ← docker.io / ghcr.io
```

### 볼륨 — "컨테이너는 휘발성"이라 필요
컨테이너를 지우면 안의 데이터도 사라짐 → DB는 큰일. 그래서:
```yaml
volumes:
  - postgres_data:/var/lib/postgresql/data   # 데이터를 컨테이너 밖에 보관
```
→ 컨테이너 지웠다 다시 만들어도 **데이터는 볼륨에 남음.**

### 네트워크 — 컨테이너 이름이 곧 주소

#### 왜 필요한가 — "컨테이너 1개면 사실 필요 없다"
- 컨테이너는 각자 **격리된 IP**를 가진 남남. 그냥 띄우면 서로 모름.
- 네트워크 = 컨테이너끼리 **대화시키는 가상 통로**. 그래서:
  - **컨테이너 1개**(혼자 실행)면 네트워크 신경 쓸 일 거의 없음.
  - **여러 컨테이너**(app + DB + minio)면 → 네트워크로 묶어야 통신됨.
- ⚠️ 헷갈림 주의: **"컨테이너 안에 이미지 여러 개"는 없음.** 컨테이너 1개 = 이미지 1개 = 역할 1개.
  여러 역할을 한 컨테이너에 다 넣을 수도 있지만(그럼 다 `localhost`라 네트워크 불필요),
  실무는 **일부러 역할별로 쪼개서** → 따로 재시작/업데이트/스케일 → 그래서 네트워크가 필요해짐.

#### 이름으로 통신되는 조건
| 네트워크 종류 | 이름 통신 |
|---|---|
| 기본 bridge (`docker run` 그냥) | ❌ IP로만 |
| 사용자 정의 네트워크 / compose | ✅ 도커 내장 DNS가 이름→IP 변환 |

같은 (사용자 정의) 네트워크면 **컨테이너 이름으로 통신**:
```
app → "my-postgres:5432" (DB)
app → "my-minio:9000"        (스토리지)
```
→ IP 몰라도 이름으로 찾음. **IP는 재시작 때마다 바뀌지만 이름은 안 바뀜** → 이름 통신이 안정적이라 실무 표준.
→ **JAR 단독 실행이 안 됐던 이유** = 이 네트워크 밖이면 이름 해석 불가.

> ⭐ **이름 통신은 "편의"가 아니라 사실상 "필수".**
> 컨테이너는 그냥 `docker run`만 해도 기본 bridge에 들어가 IP는 받음(=네트워크가 아예 없진 않음).
> 단 기본 bridge는 이름 해석이 안 돼 **IP로만** 통신 가능 → 근데 IP는 재시작/재생성마다 바뀜
> (`172.17.0.3 → 172.17.0.5`). 설정에 IP를 박으면 DB 한 번 재시작에 연결이 깨짐.
> 이름(`my-postgres`)은 안 바뀌므로, 사용자 정의 네트워크/compose를 쓰는 진짜 이유가 이것.
> 비유: **IP = 그때그때 바뀌는 좌석번호 / 이름 = 안 바뀌는 사람 이름.**

#### `external: true` 주의 ⚠️
```yaml
networks:
  app-shared:
    external: true   # "내가 미리 만들어 둘 테니 compose야 새로 만들지 말고 갖다 써"
```
- 보통 compose는 네트워크를 **자동 생성**하는데, `external: true`면 자동 생성 안 함.
- 그래서 **처음 한 번은 직접 만들어야** 함 (안 하면 `network ... not found` 에러):
  ```bash
  docker network create app-shared
  ```
- 왜? minio는 `shared` 프로필 = dev/prod가 **공유**. 공용 네트워크에 dev앱·prod앱·minio를 다 꽂으려는 의도.

#### `ports`(외부 노출) vs `네트워크`(내부 통신) — 헷갈리지 말 것
- `ports: "8080:8080"` = **호스트(외부)** → 컨테이너. 브라우저·외부 클라이언트 접속용.
- 네트워크 = **컨테이너끼리(내부)** 통신. 이름으로 접근.
- 예: app↔DB는 같은 네트워크라 `ports` 없어도 `my-postgres:5432`로 대화 가능.
  DB에 `ports: 5432`를 연 건 **호스트에서 직접 접속**(DBeaver 등)하려는 용도일 뿐.

---

## 6. 이미지 만들기 — Dockerfile & 레이어 캐시

`Dockerfile` = 이미지 만드는 레시피. **각 줄이 하나의 레이어.**
```dockerfile
FROM eclipse-temurin:21-jdk      # ① 베이스 이미지
COPY build.gradle ./             # ② 레이어
RUN ./gradlew dependencies       # ③ 레이어 (의존성 — 안 바뀌면 캐시)
COPY src ./src                   # ④ 자주 바뀜
RUN ./gradlew build              # ⑤
```
- **레이어 캐시**: 바뀐 줄부터 아래만 다시 실행 → 자주 바뀌는 `COPY src`를 아래에 둠.
- ⚠️ `build.gradle` 바뀌면 ③ 캐시 깨짐 → 의존성 전체 재다운로드.
- **멀티스테이지**: 빌드용(JDK)·실행용(JRE) 분리 → 최종 이미지 작고 이식성↑.

```
docker build → Dockerfile로 → 이미지
docker run   → 이미지로      → 컨테이너
```

---

## 7. Docker Compose — 여러 컨테이너 한 방에

컨테이너 여러 개 + 네트워크 + 볼륨을 YAML 하나로 선언:

```yaml
services:
  app: { build: .., ports: [ "8080:8080" ], mem_limit: 512m }
  postgres: { image: postgres:16, volumes: [ postgres_data:/... ] }
  minio: { image: minio/minio }
```
```bash
docker compose up -d    # 전부 한 번에 띄움
docker compose down     # 한 번에 정리
```
→ "여러 컨테이너"를 **코드로 관리**.

### profiles — 켤 서비스를 골라 띄우는 스위치
예를 들어 한 compose에 dev/prod/공용을 다 담고 `profiles`로 분기할 수 있다.
```yaml
app:      { profiles: ["app"] }            # app 프로필일 때만
postgres: { profiles: ["app", "local"] }   # 두 프로필 중 하나면 뜸
minio:    { profiles: ["shared"] }         # 공용(dev·prod 공유)
```
- profile 붙은 서비스는 **그냥 `up` 하면 안 뜸** → `--profile app` 처럼 명시해야 기동(명령어는 9번 참고).
- `external: true` 네트워크(`app-shared`)와 한 세트: 서로 다른 기동(dev앱·prod앱·minio)을
  **미리 만든 공용 네트워크**에 꽂아 이름으로 통신 (자세히는 5번 네트워크 섹션).

### compose 파일 작성법 — 뼈대와 주요 키
최상위는 `services` / `volumes` / `networks` 3개. (`version:`은 요즘 생략)
```yaml
services:                         # ① 컨테이너들(서비스 단위로 선언)
  app:                            #   서비스 이름 = 내부 통신용 호스트명
    image: ghcr.io/.../server:dev #   받아 쓸 이미지  (직접 빌드면 build: . )
    # build: { context: ., dockerfile: Dockerfile }  ← 이미지 대신 직접 빌드
    container_name: my-app   # 실제 컨테이너 이름(생략하면 자동 생성)
    environment:                  #   컨테이너 안 환경변수
      SPRING_PROFILES_ACTIVE: dev
      SPRING_DATASOURCE_URL: jdbc:postgresql://my-postgres:5432/mydb  # 이름통신
    env_file: .env                #   변수 파일로 분리도 가능
    ports: ['8080:8080']          #   호스트:컨테이너  (외부 노출용)
    volumes:                      #   데이터 영속/마운트
      - postgres_data:/var/lib/postgresql/data   # 명명 볼륨
      - /opt/logs:/app/logs                      # 호스트 경로 바인드
    depends_on: [postgres]        #   기동 순서(단, '준비완료'까진 보장X)
    restart: unless-stopped       #   재시작 정책
    mem_limit: 512m               #   메모리 한도 (cgroups)
    networks: [app-shared]     #   연결할 네트워크
    profiles: ["app"]             #   이 프로필일 때만 기동

volumes:                          # ② 명명 볼륨 선언(여기 적어야 services에서 씀)
  postgres_data:
  minio_data:

networks:                         # ③ 네트워크 선언
  app-shared:
    external: true                #   미리 만든 외부 네트워크 사용(자동생성X)
```

**자주 헷갈리는 키 정리**
| 키 | 뜻 | 주의 |
|---|---|---|
| `image` vs `build` | 받아쓰기 vs 직접빌드 | 둘 중 하나. 예: 운영=image(GHCR), 로컬=build 가능 |
| `ports` | 호스트↔컨테이너 노출 | **외부 접속용.** 내부 통신은 ports 없어도 됨(네트워크) |
| `expose` | 컨테이너끼리만 포트 공개 | 호스트엔 노출 안 함 (ports와 다름) |
| `volumes`(서비스) | 마운트 | 명명볼륨은 최상위 `volumes:`에도 선언해야 함 |
| `environment` vs `env_file` | 인라인 vs 파일 | 비밀값은 `.env`로 빼고 git 제외 |
| `depends_on` | 기동 순서 | DB *준비*까진 보장X → 앱에 재시도 로직 필요 |
| `${VAR:-기본값}` | 변수+기본값 | `.env`나 셸 환경에서 주입. 미설정시 기본값 |

> 작성 팁:
> - 변수 치환 결과는 `docker compose config`로 **최종 yaml을 눈으로 확인**하고 띄우기.
> - 한 서비스 = 한 역할(1컨테이너=1이미지). 역할 늘면 서비스를 추가하고 같은 네트워크로 묶기.
> - 비밀값(`DB_PASSWORD` 등)은 yaml에 직접 쓰지 말고 `.env`/시크릿으로.

---

## 8. 레지스트리 — Docker Hub vs GHCR (종류는 같다)

> 코드의 **GitHub vs GitLab**과 똑같은 관계. 둘 다 "컨테이너 레지스트리"(OCI 표준), `docker pull/push` 동일.

### `docker pull mysql`이 되는 이유 = 주소 생략
```
docker pull mysql
   = docker pull docker.io/library/mysql:latest
            └─레지스트리─┘└공식┘ └이미지┘ └태그┘
docker pull ghcr.io/my-org/my-app:latest
            └─레지스트리─┘ └─계정─┘  └─이미지─┘
```
레지스트리 도메인을 안 쓰면 도커가 **Docker Hub(`docker.io`)로 자동 간주.** 그래서 `mysql`만 쳐도 받아짐.

### 보통 둘 다 섞어 씀
```yaml
postgres: { image: 'postgres:16' }      # Docker Hub 공식 이미지
minio:    { image: minio/minio:latest } # Docker Hub minio 계정
app:      { build: . }                  # 직접 빌드 → (목표) GHCR에 push
```

### 비교 — 내 앱 이미지는 왜 GHCR?
| | Docker Hub | GHCR |
|---|---|---|
| 공식 이미지(mysql 등) | ✅ 본진 | ❌ |
| 내 private 이미지 | 무료 1개만 | **무제한 무료** |
| 인증 | 별도 계정·토큰 | **GITHUB_TOKEN 자동** |
| pull 횟수 제한 | 무료 6시간당 제한 ⚠️ | 넉넉 |
| 코드와 위치 | 분리 | **GitHub에 통합** |

→ **공식 이미지 = Docker Hub에서 pull, 내 앱 = GHCR에 push.** 섞어 쓰는 게 정상.
종류: Docker Hub / GHCR / GitLab Registry / AWS ECR / Google GCR … 전부 같은 방식, 플랫폼 통합 편의로 선택.

---

## 9. 명령어 치트시트 (빈도순)

```bash
# === 상태 ===
docker ps                    # 실행 중 컨테이너
docker ps -a                 # 멈춘 것 포함 전부
docker images                # 이미지 목록
docker stats                 # 실시간 CPU/메모리 (메모리 디버깅 때 씀)
docker logs -f <컨테이너>     # 로그 실시간 추적

# === 컨테이너 ===
docker run -d --name app -p 8080:8080 <이미지>   # 백그라운드 실행
docker exec -it <컨테이너> bash                   # 안으로 들어가기
docker stop / start / restart <컨테이너>
docker rm <컨테이너>

# === 이미지 ===
docker build -t <이름>:<태그> .   # Dockerfile로 빌드
docker pull / push <이미지>       # 레지스트리에서 받기/올리기
docker rmi <이미지>

# === Compose (기본) ===
docker compose up -d              # 전체 띄우기(백그라운드)
docker compose up -d --build      # 빌드까지 (CD에서 쓰는 그 --build)
docker compose up -d <서비스>      # 특정 서비스만 (예: app, postgres)
docker compose pull               # 이미지만 받기 (빌드 없이 GHCR에서)
docker compose down               # 전체 내리기 (네트워크도 제거)
docker compose down -v            # 볼륨까지 삭제 ⚠️ DB 데이터 날아감
docker compose restart <서비스>    # 특정 서비스만 재시작
docker compose stop / start       # 컨테이너 유지한 채 정지/기동

# === Compose (운영·디버깅) ===
docker compose ps                 # 이 compose가 띄운 컨테이너 목록·상태
docker compose logs -f <서비스>    # 특정 서비스 로그 추적
docker compose exec <서비스> bash  # 서비스 컨테이너 안으로 (예: app)
docker compose config             # 변수(.env) 다 치환된 *최종* yaml 확인 ⭐디버깅 필수
docker compose pull && docker compose up -d   # 새 이미지로 무중단 비슷하게 갱신

# === Compose (profiles) ===
# profile 붙은 서비스는 그냥 up 하면 안 뜸 → 반드시 명시해야 기동
docker compose --profile app up -d        # app + postgres (app프로필)
docker compose --profile shared up -d     # minio (shared, dev/prod 공용)
docker compose --profile local up -d      # postgres만 (로컬 DB만 띄울 때)
docker compose --profile app --profile shared up -d   # 여러 프로필 동시
# 환경 변수와 같이: SPRING_PROFILE=prod docker compose --profile app up -d
# ⚠️ down 도 같은 프로필 줘야 그 서비스들이 내려감
docker compose --profile app down

# === 청소 (디스크 찰 때) ===
docker system df             # 도커 디스크 사용량
docker system prune          # 안 쓰는 컨테이너/이미지/캐시 정리
docker volume prune          # 안 쓰는 볼륨 (⚠️ 데이터 주의)
```
자주 쓰는 플래그: `-d`(백그라운드) `-p`(포트 host:container) `-it`(터미널) `--name`(이름).

---

## 10. 전체 흐름 한 장 (코드 → 서버)

```
개발자 코드
   │ git push
   ▼
[Dockerfile 레시피] ─build─► [이미지] ─push─► [레지스트리(Hub/GHCR)]
                                                   │ pull
                                                   ▼
                                [서버] ─run─► [컨테이너 실행]
                                              ├ namespace (격리)
                                              ├ cgroups   (mem_limit)
                                              ├ volume    (데이터 영속)
                                              └ network   (이름으로 통신)
```

---

## 11. 한눈 요약

| 개념 | 한 줄 | 예시 |
|---|---|---|
| 왜 쓰나 | 포장해서 어디서든 동일 실행 | 1.9GB에 4개 |
| vs VM | OS 공유 → 가볍고 빠름 | — |
| 격리 원리 | namespace(격벽)+cgroups(미터기) | `mem_limit: 512m` |
| 아키텍처 | CLI→데몬(dockerd)→레지스트리 | — |
| 이미지/컨테이너 | 틀 / 실행체 | `build:` |
| 볼륨 | 컨테이너 밖 영구 저장 | `postgres_data` |
| 네트워크 | 이름으로 통신 | `app-shared` |
| Dockerfile | 이미지 레시피(=레이어) | 멀티스테이지 |
| Compose | 여러 컨테이너 YAML 관리 | `compose.yaml` |
| 레지스트리 | 이미지 창고 (Hub=공식, GHCR=내 앱) | postgres=Hub, app=GHCR |

> 한 줄: **"앱을 이미지로 포장(Dockerfile) → 창고에 보관(레지스트리) → 서버에서 격리 실행(namespace+cgroups)."**

---

## 관련 노트
- CI/CD·배포 흐름 → Docker CI/CD·GHCR 글
- 서버 메모리/스왑/오버커밋 → 서버 운영 글
- JVM 컨테이너 메모리 → JVM 메모리·튜닝 글
- CPU·메모리 계층 → CPU·메모리 계층 글
