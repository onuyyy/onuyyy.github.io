---
title: "서버 운영 — 메모리 · CPU · Swap · Docker 실무 정리"
date: 2026-06-17 09:00:00 +0900
categories: [Infrastructure]
tags: [Server, Memory, CPU, Swap, Docker, Linux]
---

> 어떤 서버(AWS EC2, RAM 1.9GB)에서 "도커로 Java/Python/프론트 띄웠는데
> 메모리가 자주 찬다"를 진단하며 배운 실무 내용 정리.
> 핵심 질문: **"서버 메모리에 뭐가 올라가고, 꽉 찼다는 건 어떻게 판단하나"**

---

## 0. 출발점 — 실제 상황

- EC2 인스턴스 RAM **1.9GB**(약 2GB, t3.small급)
- Docker 컨테이너 4개: `my-app`(Java), `my-ai`(Python), `postgres`, `minio`
- 증상: 메모리가 자주 차는 것 같음 → 진단 시작

---

## 1. 서버 RAM에 뭐가 올라가나

| 구분 | 내용 |
|---|---|
| **프로세스 실사용 메모리** | 각 앱(JVM, Python, …)이 실제 쓰는 메모리 (코드+힙+스택) |
| **페이지 캐시 (buff/cache)** | 한번 읽은 파일을 OS가 RAM에 캐싱. **"쓰는 중"이 아니라 "빌려쓰는 중"** → 앱이 필요하면 즉시 회수 |
| **커널 / OS** | 리눅스 커널, 네트워크 버퍼, 도커 데몬 등 |

> 메모리 계층 관점: 여기서 보는 "메모리"는 전부 **RAM(DRAM)**. CPU 캐시(L1~L3)는 CPU 안이라 안 보이고,
> RAM 부족 시 디스크의 **swap**으로 밀려난다. (→ CPU·메모리 계층 글)

---

## 2. `free -h` 읽는 법 — 착시 걷어내기 (가장 중요)

```
       total  used  free  shared  buff/cache  available
Mem:   1.9Gi  1.3Gi 64Mi  22Mi    710Mi       573Mi
Swap:  0B     0B    0B
```

| 컬럼 | 의미 | 함정 |
|---|---|---|
| `total` | 전체 RAM | — |
| `used` | 사용 중 | **이게 높다고 위험한 거 아님** |
| `free` | 완전히 빈 것 | 거의 항상 작음(정상). OS가 남는 RAM을 캐시로 쓰니까 |
| `buff/cache` | 파일 캐시 | **회수 가능한 메모리.** "빌려쓰는 중"일 뿐 |
| **`available`** | **앱이 실제로 더 쓸 수 있는 양** | ✅ **이걸 봐야 한다** (free + 회수가능 캐시) |

**판단 기준:**
- `free`가 작아도 → 정상 (OS가 캐시로 활용 중)
- **`available`이 0에 가까워지거나 / `swap used`가 쌓이면** → 그게 진짜 부족 신호

위 예: `used 1.3Gi`라 놀라기 쉽지만 `buff/cache 710Mi`는 회수 가능 → 실제 여유 `available 573Mi`. **아직 위험 아님.**

---

## 3. Swap — RAM 넘칠 때의 안전망

### 개념
- **RAM이 부족할 때, 당장 안 쓰는 데이터를 디스크로 잠깐 내보내 RAM을 비우는 것.**
- 그 디스크 공간 = swap. **"느린 예비 메모리".**
- 비유: RAM = 책상, swap = 책장. 책상 꽉 차면 안 보는 책을 책장에 잠깐 꽂아둠(꺼낼 때 느림).

### 왜 필요한가 / swap=0이면?
- swap이 0이면 완충 공간이 없다 → RAM 꽉 차는 순간 리눅스 **OOM Killer**가 프로세스를 강제 종료(컨테이너가 갑자기 툭 꺼짐).
- swap 있으면 "느려질지언정 죽지는 않게" → **죽는 것보단 잠깐 느린 게 낫다.**

### 한계 (만능 아님)
- swap은 디스크라 **느리다.** 자주 쓰이면(thrashing) 서버가 기어간다.
- RAM + swap **둘 다** 차면 결국 OOM. swap은 **시간 벌기/응급처치**일 뿐, 반복되면 근본 해결(RAM 증설) 필요.

### Swap 2GB 만드는 법 (실제로 한 명령)
```bash
sudo fallocate -l 2G /swapfile      # ① 2GB 빈 파일 생성
sudo chmod 600 /swapfile            # ② root만 접근(보안 — swap엔 민감정보 들어갈 수 있음)
sudo mkswap /swapfile               # ③ 파일을 swap 형식으로 포맷
sudo swapon /swapfile               # ④ 지금 즉시 활성화 (재부팅하면 사라짐)
echo '/swapfile none swap sw 0 0' | sudo tee -a /etc/fstab   # ⑤ 재부팅에도 유지(영구 등록)
free -h                             # 확인: Swap 2.0Gi 잡혔는지
```
흐름: 빈 땅 사기(①) → 울타리(②) → 정비(③) → 입주(④) → 등기(⑤)

### swappiness — swap을 얼마나 적극적으로 쓸지 (0~100, 기본 60)
```bash
sudo sysctl vm.swappiness=10                          # 즉시 적용
echo 'vm.swappiness=10' | sudo tee -a /etc/sysctl.conf  # 영구
```
- 서버는 보통 낮게(10): RAM 최대한 쓰고 정말 부족할 때만 swap.

---

## 4. Docker 메모리 — 컨테이너별 한도(칸막이)

### 핵심: 서버 RAM을 컨테이너들이 나눠 쓴다
```
┌─────────────────────────────────────┐
│      서버 전체 RAM = 1.9GB            │
│  ┌──────────┐ 한도 512MB (app, 376↑) │
│  ┌──────────┐ 한도 512MB (ai,  65↑)  │
│  ┌──────────┐ 한도 256MB (postgres)  │
│  ┌──────────┐ 한도 256MB (minio)     │
│  ┌──────────┐ OS·도커 데몬 (~0.4~0.7) │
└─────────────────────────────────────┘
```
- `docker stats`의 `MEM USAGE / LIMIT`에서 LIMIT = **그 컨테이너 한 개의 한도** (서버 전체 아님!)
- 한도(`mem_limit`)는 **칸막이**: 한 컨테이너(특히 JVM)가 RAM을 독식해 남을 굶기는 걸 막는다.
- 한도를 **안 걸면** 위험. 거는 게 정상.

### 진단 명령
```bash
free -h                                    # 서버 전체
docker stats                               # 컨테이너별 실시간 (CPU%, MEM USAGE/LIMIT, MEM%)
docker stats --no-stream                   # 스냅샷 1회
docker exec <컨테이너> java -version        # Java 컨테이너인지 확인
dmesg | grep -i oom                        # OOM Kill 흔적 (갑자기 죽었을 때)
```

### docker-compose 한도 설정
```yaml
services:
  app:
    mem_limit: 512m     # 컨테이너 RAM 상한
    # JVM이면 JAVA_TOOL_OPTIONS=-XX:MaxRAMPercentage=70.0 도 함께 (→ jvm 노트)
```

---

## 5. 오버커밋(Overcommit) — 진짜 문제

```
한도 합 = 512(app) + 512(ai) + 256(pg) + 256(minio) = 1,536MB (1.5GB)
        + OS·도커 (~0.4~0.7GB)
        ≈ 1.9~2.2GB  ←  서버 RAM 1.9GB에 빠듯하거나 살짝 초과
```

- **"한도의 합 > 실제 RAM"** = 오버커밋 상태.
- 지금은 컨테이너들이 한도까지 안 차서 버티지만, 동시에 다 차면 터진다.
- 즉 메모리가 빠듯했던 **진짜 원인 = 개별 앱 잘못이 아니라 "작은 서버 + 오버커밋".**

### 처방 우선순위
| 처방 | 효과 | 상태 |
|---|---|---|
| swap 2GB 추가 | 비상 완충(OOM 방지) | ✅ 완료 |
| 여유 큰 `my-ai`(65/512) 한도 → 256m 축소 | 오버커밋 완화 | 권장 다음 |
| 인스턴스 업그레이드(2GB→4GB, t3.medium) | 근본 해결 | RAM 부족 반복 시 |

---

## 6. CPU 보는 법 (메모리와 함께 자주 보는 것)

- `docker stats`의 `CPU %`: 컨테이너별 CPU 사용률 (이 경우 0.1~0.2%로 한가 → CPU는 문제 아님)
- `top` / `htop`: 프로세스별 CPU·메모리, **load average**(1·5·15분 평균 대기 작업 수)
  - load average가 **코어 수보다 크면** CPU가 밀리는 중. (예: 2코어인데 load 4 → 과부하)
- `nproc`: 코어 수 확인
- CPU 계층 관점(레지스터/캐시/RAM)은 → CPU·메모리 계층 글

---

## 7. 진단 플로우 (메모리 "꽉 찼다" 신고 들어오면)

```
1. free -h
   → available 충분? swap used 쌓임?   (used만 보고 패닉 금지)
2. docker stats --no-stream
   → 어느 컨테이너가 LIMIT에 붙어있나? MEM%?
3. (Java면) java -XX:+PrintFlagsFinal -version | grep MaxHeapSize
   → JVM이 컨테이너 한도 제대로 인식? (→ jvm 노트)
4. dmesg | grep -i oom
   → 실제로 OOM Kill 당한 적 있나?
5. 판단:
   - 특정 컨테이너 폭주 → 그 앱 튜닝/한도 조정
   - 전체적으로 빠듯 + 오버커밋 → swap / 한도 재배분 / 인스턴스 증설
```

---

## 8. 요약 (한 장)

| 항목 | 핵심 |
|---|---|
| 서버 RAM에 올라가는 것 | ① 프로세스 메모리 ② 페이지 캐시(회수가능) ③ 커널/OS |
| `free` 판단 기준 | `used` 아님! **`available`** 과 **`swap used`** 를 봐라 |
| `buff/cache` | 회수 가능한 캐시. 높아도 정상 |
| swap | RAM 넘칠 때 디스크로 대피 = 안전망. 느림. 0이면 OOM 위험 |
| swappiness | 서버는 낮게(10): RAM 우선, swap은 비상시만 |
| docker LIMIT | 서버 전체 아님. **컨테이너 한 개의 칸막이** |
| 오버커밋 | 한도 합 > RAM. 메모리 압박의 진짜 원인 |
| 처방 | swap 추가 ✅ → ai 한도 축소 → 필요시 인스턴스 증설 |

> 교훈: **`used` 높다고 패닉 금지. `available`/`swap`을 봐라.**
> 그리고 "메모리 부족"의 원인은 개별 앱보다 **전체 용량 설계(오버커밋)**인 경우가 많다.

---

## 관련 노트
- CPU·메모리 계층 글 — RAM이 왜 빠른가, 메모리 계층/캐시 원리
- JVM 메모리·튜닝 글 — JVM 메모리 구조, 컨테이너 인식, 튜닝
