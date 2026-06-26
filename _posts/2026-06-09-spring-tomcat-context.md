---
title: "Spring(레거시 XML) + Tomcat + Context 기초 정리"
date: 2026-06-09 09:00:00 +0900
categories: [Spring]
tags: [Spring, Tomcat, Context, Legacy]
---

> Spring Boot만 써본 사람이 레거시 Spring MVC(XML 설정 + WAR + Tomcat) 구조를 이해하기 위한 정리.
> 핵심 질문: **설정 파일은 누가/어떻게 읽나? 규격이 맞아야 읽히나? Context는 뭐고 부모-자식은 뭐냐?**

---

## 0. 핵심 한 줄

> **톰캣은 "서블릿"이라는 규격의 객체만 알고, 스프링은 그 규격에 맞춰 자기 자신을 끼워넣는다.**

- Boot는 이 모든 과정을 자동화/은닉했기 때문에 안 보였을 뿐, 내부에서는 똑같은 일이 일어난다.

---

## 1. 등장인물 (역할 구분이 제일 중요)

```
┌─────────────────────────────────────────────────────────┐
│  WAS / 서블릿 컨테이너  =  톰캣(Tomcat)                    │
│   - HTTP 요청을 받아 처리하는 "엔진"                        │
│   - 자바 웹 표준 규격 = "서블릿 스펙(Servlet Spec)"을 구현   │
│   - 스프링을 1도 모름. 오직 '서블릿'이라는 규격만 앎         │
└─────────────────────────────────────────────────────────┘
                          ↑ 이 규격(계약)으로만 대화
┌─────────────────────────────────────────────────────────┐
│  Spring Framework                                         │
│   - 톰캣이 정한 규격(서블릿/리스너)에 자기를 "끼워넣음"      │
│   - 그 안에서 빈(Bean), DI, MVC, Security 등을 처리         │
└─────────────────────────────────────────────────────────┘
```

---

## 2. "규격이 맞아야 읽힌다"의 두 층위

설정 파일이 읽히는 과정은 **두 단계의 계약(contract)** 을 거친다.

### 계약 ① 톰캣 ↔ web.xml  (엄격한 표준 규격)

- `web.xml`은 **서블릿 스펙이 정한 표준 파일**이다.
- 이름·위치(`/WEB-INF/web.xml`)·태그(`<context-param>`, `<listener>`, `<servlet>` ...)가 **고정**. 마음대로 못 바꾼다.
- 톰캣은 앱이 뜰 때 **무조건 `/WEB-INF/web.xml`을 찾아 읽는다.**
- 톰캣이 이해하는 건 정해진 표준 태그뿐: "리스너 띄워라", "서블릿 매핑해라", "전역 파라미터 저장해라".

### 계약 ② 스프링 ↔ 설정 XML  (스프링이 정한 규칙, 톰캣과 무관)

- `root-context.xml`, `servlet-context.xml` 등은 **서블릿 표준이 아니다.** 톰캣은 이 파일들의 존재조차 모른다.
- 순전히 **스프링의 약속**. 파일 **이름도 자유**(`applicationContext.xml`이든 뭐든 OK).
- 단지 **그 경로를 스프링에게 알려주기만** 하면 된다 → 그 통로가 `contextConfigLocation`.

> **정리:** 위치/이름이 고정인 건 `web.xml` 하나뿐. 나머지 스프링 설정 파일은 이름·개수·위치 다 자유고, web.xml을 통해 "경로만" 스프링에게 전달된다.

---

## 3. 전체 부팅 흐름 (시간 순서)

```
[앱 시작 / 톰캣이 WAR을 띄움]
        │
        ▼
① 톰캣: "/WEB-INF/web.xml 있네? 읽자" (표준이라 무조건 읽음)
        │
        ▼
② 톰캣: web.xml의 <context-param> 값들을
        ServletContext(앱 전역 저장소)에 그냥 보관만 함
        (아직 스프링 1도 동작 안 함)
        │
        ▼
③ 톰캣: web.xml의 <listener>에 적힌
        ContextLoaderListener(스프링 객체)를 new 해서 실행
        ↑ 여기서 처음으로 스프링이 깨어남
        │
        ▼
④ ContextLoaderListener: ServletContext에서
        'contextConfigLocation' 값을 꺼냄
        = "root-context.xml, security-context.xml ..." 경로 목록
        │
        ▼
⑤ 스프링: 그 XML들을 읽어서
        ★ 루트 컨테이너(Root ApplicationContext) ★ 생성
        → DataSource, Service, MyBatis, Security 빈 등이 여기서 생성
        │
        ▼
⑥ 톰캣: web.xml의 <servlet>에 적힌
        DispatcherServlet(또 다른 스프링 객체)을 실행
        │
        ▼
⑦ DispatcherServlet: 자기 설정(servlet-context.xml)을 읽어서
        ★ 웹 컨테이너(Servlet ApplicationContext) ★ 생성
        → Controller, ViewResolver 등 화면 관련 빈이 생성
        │
        ▼
[준비 끝 — 이제 사용자 HTTP 요청을 받기 시작]
```

### "누가 부르나"에 대한 답

| 대상 | 읽는/실행하는 주체 |
|------|-------------------|
| web.xml | **톰캣** (표준이라 무조건) |
| 스프링 설정 xml | **스프링 객체** (ContextLoaderListener, DispatcherServlet) |
| 위 스프링 객체를 깨우는(실행하는) 것 | **톰캣** (web.xml에 등록돼 있으니까) |

> 구조 요약: **톰캣이 스프링을 깨우고 → 깨어난 스프링이 자기 설정파일을 읽는다** (릴레이 구조).

---

## 4. "Context" 개념 — 종류가 3개라서 헷갈린다

```
ServletContext          ← 톰캣(서블릿 스펙)의 것.  "앱 1개 = 1개"
ApplicationContext      ← 스프링의 것.  "스프링 컨테이너(빈 보관소)"
  ├─ Root ApplicationContext     (부모)
  └─ Servlet ApplicationContext  (자식) = WebApplicationContext
```

- `~Context`라는 단어가 붙었다고 다 같은 게 아니다.
- **ServletContext = 톰캣 세계**, **ApplicationContext = 스프링 세계.** 완전히 다른 동네.
- 둘이 만나는 접점은 딱 하나 → `contextConfigLocation` 전달 지점.

### (A) ServletContext — 톰캣이 만드는 "앱 전역 상자"

```
┌──────────────────────────────────────────┐
│  ServletContext  (웹앱 1개당 딱 1개)        │
│   - 톰캣이 앱 띄울 때 자동 생성             │
│   - "이 앱 전체에서 공유되는 전역 저장소"   │
│   - web.xml의 <context-param>이 여기 담김   │
│   - 모든 서블릿/리스너가 이걸 같이 봄        │
└──────────────────────────────────────────┘
```

- 누가 만드나? → **톰캣**
- 몇 개? → WAR 1개당 1개
- 역할? → 앱 전역 게시판. `contextConfigLocation` 같은 값이 붙어 있고, 나중에 스프링이 꺼내감.

### (B) ApplicationContext — 스프링이 만드는 "빈 컨테이너"

```
┌──────────────────────────────────────────┐
│  ApplicationContext (= 스프링 컨테이너)     │
│   - 스프링이 XML/어노테이션 읽어서 생성      │
│   - 빈(Bean)을 만들어 보관하고 DI 해주는 관리소│
│   - 흔히 "스프링 컨테이너"라 부르는 것       │
└──────────────────────────────────────────┘
```

- 누가 만드나? → **스프링 객체**(ContextLoaderListener, DispatcherServlet)
- 역할? → 빈 생성·보관·주입(DI). `@Service`, `<bean>` 한 것들이 사는 집.

### 둘의 연결 (접점)

```
   톰캣 세계                              스프링 세계
┌──────────────┐                  ┌───────────────────────┐
│ ServletContext│  ──①값 전달──▶   │  ApplicationContext    │
│  (전역 상자)   │  contextConfig   │  (빈 컨테이너)          │
│               │     Location     │                       │
│  ◀──②자기를 상자에 등록(setAttribute)──                    │
└──────────────┘                  └───────────────────────┘
```

- ① 스프링이 ServletContext에서 `contextConfigLocation`을 **꺼내** 컨테이너를 만들고
- ② 다 만든 스프링 컨테이너를 **다시 ServletContext 상자에 등록**(`setAttribute`)
  → 나중에 다른 서블릿/필터가 "스프링 컨테이너 어딨어?" 할 때 꺼내 쓸 수 있게.
- 즉 **ServletContext = 그릇, ApplicationContext = 그릇 안 내용물**.

---

## 5. 부모-자식 관계 (Root vs Servlet)

스프링 컨테이너가 **2개** 생기고, 이 둘이 부모-자식이다.

```
┌────────────────────────────────────────────────────┐
│              ServletContext (톰캣, 1개)              │
│                                                    │
│   ┌──────────────────────────────────────────┐     │
│   │   Root WebApplicationContext  (부모)       │     │
│   │   생성자: ContextLoaderListener            │     │
│   │   내용물: DataSource, Service, Mapper,     │     │
│   │           Security, Cache (뒷단 공통)       │     │
│   └──────────────────────────────────────────┘     │
│                     ▲                              │
│                     │ getParent() — 자식은 부모를 봄  │
│                     │ (부모는 자식 못 봄)            │
│   ┌──────────────────────────────────────────┐     │
│   │  Servlet WebApplicationContext (자식)      │     │
│   │  생성자: DispatcherServlet                 │     │
│   │  내용물: Controller, ViewResolver,         │     │
│   │          Interceptor (웹 앞단)             │     │
│   └──────────────────────────────────────────┘     │
└────────────────────────────────────────────────────┘
```

### 규칙 (딱 2가지)

1. **자식 → 부모 빈을 가져다 쓸 수 있다.**
   - 자식(Controller)이 부모(Service)를 `@Autowired` 가능.
2. **부모 → 자식 빈은 못 본다.**
   - 부모(Service)가 자식(Controller)을 주입받으려 하면 실패.

### 빈 검색 순서

- 자식이 빈을 찾을 때: 자기 안에서 먼저 찾고 → 없으면 부모로 올라가서 찾음.

### 왜 나눴나 (의도)

- **부모(Root):** 여러 DispatcherServlet이 공유하는 공통 뒷단. (서블릿이 여러 개여도 DataSource는 하나)
- **자식(Servlet):** 웹 요청 처리 전용. 서블릿마다 하나씩 가질 수 있음.

### "공유"의 정확한 의미 — 부모는 1개(공유), 자식은 서블릿마다 1개씩(개인 것)

> 헷갈림 포인트: "부모/자식 둘 다 서블릿이 공유하는 거 아냐?" → **아니다. 공유되는 건 부모뿐이다.**

```
┌─────────────────────────────────────────────────────────┐
│                   ServletContext (톰캣, 1개)              │
│        ┌────────────────────────────────────┐            │
│        │   Root 컨텍스트 (부모) — 딱 1개        │            │
│        │   DataSource, Service, TX ...        │            │
│        └────────────────────────────────────┘            │
│              ▲                      ▲                     │
│              │ getParent()         │ getParent()         │
│       ┌──────┴───────┐      ┌───────┴──────┐             │
│       │ 자식 A         │      │ 자식 B        │             │
│       │ (apiServlet)  │      │ (adminServlet)│            │
│       │ Controller A  │      │ Controller B  │            │
│       └───────────────┘      └───────────────┘            │
│         ↑ 형제끼리 서로 못 봄 (A는 B의 빈을 못 봄) ↑          │
└─────────────────────────────────────────────────────────┘
```

- **부모(Root) = 모든 자식이 공유** ⭕
  - DispatcherServlet이 몇 개든 부모는 하나, 다 같이 올려다본다(getParent).
  - DataSource·Service를 부모에 두면 → 모든 서블릿이 **같은 인스턴스 공유**.
- **자식(Servlet) = 그 서블릿 전용, 다른 서블릿과 공유 ❌**
  - 자식 A의 빈은 자식 A를 만든 그 DispatcherServlet만 본다.
  - 자식 B는 자식 A의 빈을 못 본다 (**형제끼리는 서로 안 보임**).

| | 공유되나? | 비유 |
|---|---------|------|
| 부모(Root) | ⭕ 모든 서블릿이 공유 | 공용 창고 |
| 자식(Servlet) | ❌ 그 서블릿만 사용 | 각 서블릿의 개인 사물함 |

→ "여러 서블릿이 같이 써야 하는 것(DB연결·서비스·트랜잭션)은 부모에 1번만,
  각 서블릿 전용 웹 설정(Controller·ViewResolver)은 각자 자식에" 라는
  **배치 규칙이 바로 이 공유 구조 때문에 나온 것.**

> 단, DispatcherServlet이 `appServlet` **하나뿐**이면 자식도 1개 → "형제끼리 공유 못 함" 문제가 없다.
> 서블릿을 추가하면 그때부터 "공유하려면 부모에 둬야 한다"가 실제로 중요해진다.

### ⭐ 헷갈림 총정리 — 서블릿 ≠ 컨트롤러, 왜 파일 나누나, 부모 표식은 어디

부모-자식을 이해할 때 꼭 같이 짚어야 하는 3가지.

#### (1) 서블릿 1개 = 컨트롤러 여러 개 (프론트 컨트롤러)

> 헷갈림 포인트: "서블릿이 곧 컨트롤러 하나 아냐?" → **아니다. 서블릿 1개가 컨트롤러 N개를 거느린다.**

```
[옛날] 기능마다 서블릿 1개          [Spring MVC] 서블릿은 딱 1개
/hello → HelloServlet              모든 요청 → DispatcherServlet(안내데스크 1개)
/login → LoginServlet                              ├─ BoardController
/board → BoardServlet                              ├─ ReplyController   (창구 여러 개)
                                                   └─ LectureController
```

- `DispatcherServlet` = **안내데스크 1개.** 모든 요청을 일단 받아서, 알맞은 `@Controller`로 분배(= **프론트 컨트롤러 패턴**).
- 그래서 **서블릿 1개 = 자식 컨테이너 1개 = 그 안에 컨트롤러 N개.**
- 앞에서 든 `apiServlet`/`adminServlet`은 "DispatcherServlet을 2개 둘 수도 있다"는 **예시**일 뿐, 대부분 프로젝트는 **디스패처 1개**다. (프로젝트의 `HelloServlet.java`가 바로 옛날 "서블릿=핸들러 하나" 방식의 잔재)

#### (2) 왜 컨트롤러/서비스를 다른 파일에 넣나 — "공유 자원 vs 서블릿 전용"

파일이 둘을 나눠주는 게 아니라, **어느 컨테이너에 살아야 하는지가 먼저**고 그 컨테이너가 읽는 파일에 넣는 것뿐.

`/api`, `/admin` 디스패처 둘 다 `BoardService`가 필요하다고 치면:
```
        Root(부모)  ── BoardService · DataSource   ← 여기 한 번만 두면 둘 다 공유
       /          \
  /api(자식)    /admin(자식)   ← 형제끼리 서로 못 봄
```
- `BoardService`를 `/api` **자식**에 두면 → `/admin`은 못 봄 → 똑같은 서비스를 또 만들어야 하고 **DB 커넥션풀도 2개**? 말이 안 됨. → **공유 자원이라 부모에.**
- 컨트롤러는 `/api`용·`/admin`용이 **애초에 서로 다른 놈** → 공유할 게 아님 → **각자 자식에.**

> 한 줄: **여러 서블릿이 같이 쓸 것(서비스·DB) = 부모, 그 서블릿만 쓸 것(컨트롤러·뷰) = 자식.**

그리고 **의존 방향**이 이 배치와 딱 맞아떨어진다:
- 자식 → 부모는 보임 / 부모 → 자식은 안 보임
- 컨트롤러는 서비스를 주입받아야 함(컨트롤러가 서비스에 의존)
- **컨트롤러(자식)가 서비스(부모)를 올려다보는 구조** → ✅ 주입됨. (반대로 서비스를 자식에 두면 다른 서블릿이 못 쓰고, 방향도 어긋남)

#### (3) "부모 관계"라는 표식은 어디 있나? → XML엔 없다. 자동이다.

> 헷갈림 포인트: "`<parent>` 같은 태그를 어디 적나?" → **그런 태그 없다. web.xml 두 줄 + 스프링 코드가 자동으로 맺는다.**

```
① <listener> ContextLoaderListener  → 부모(Root) 컨테이너 생성
   └ 만든 뒤 ServletContext에 저장: key = "...WebApplicationContext.ROOT"
                                   (ROOT_WEB_APPLICATION_CONTEXT_ATTRIBUTE)
② <servlet>  DispatcherServlet       → 자식 컨테이너 생성
   └ 그 key로 ServletContext에서 부모를 꺼내 child.setParent(parent)  ← 여기서 연결!
```

- 부모/자식을 **만드는** 표식 = web.xml의 `<listener>`(부모) + `<servlet>`(자식).
- 부모로 **연결하는** 표식 = `DispatcherServlet`(상위 `FrameworkServlet`)이 시작할 때 **ServletContext의 약속된 key**로 Root를 찾아 `setParent()` 호출. **이건 XML이 아니라 프레임워크 코드에 박혀 있고 자동**이다.

| 무엇 | 표식 위치 |
|---|---|
| 부모 컨테이너 생성 | web.xml `<listener>` ContextLoaderListener |
| 자식 컨테이너 생성 | web.xml `<servlet>` DispatcherServlet |
| 부모-자식 연결 | DispatcherServlet 코드의 `setParent()` (ServletContext의 ROOT key로 조회, **자동**) |
| `<parent>` 같은 XML 태그 | **없음** |

> 한 줄: **"부모다"라고 적는 곳은 없다. ContextLoaderListener가 만든 Root를 DispatcherServlet이 자동으로 자기 부모로 삼는다 — 그 연결고리가 ServletContext에 저장된 약속된 key다.**

### ⚠️ 흔한 실수

- `<context:component-scan>`을 부모/자식 양쪽에서 같은 패키지로 돌리면 **빈이 2번 생성**된다.
- 그래서 보통 **부모는 `@Service/@Repository`만, 자식은 `@Controller`만** 스캔하도록 분리한다.

### 빈을 어디 넣을지 — 규칙인가? 자율인가? (중요)

> 질문: "컨테이너 2개 만드는 건 자율이라며. 그럼 내용물을 꼭 저렇게 나눠야 해? 부모에 Controller 넣으면 안 돼?"

**문법적 강제 규칙은 아니다.** 스프링이 "컨트롤러는 부모에 넣으면 에러!" 하고 막지 않는다.
하지만 **부모에 컨트롤러를 넣으면 실제로 동작을 안 하는 경우가 많다.** 관례가 아니라 진짜 메커니즘 때문이다.

#### 왜 컨트롤러를 부모에 넣으면 안 먹히나 (진짜 이유)

요청 → 컨트롤러 연결을 담당하는 핵심 빈이 **`RequestMappingHandlerMapping`** 인데,
이건 보통 `servlet-context.xml`(자식)에 정의된다 (`<mvc:annotation-driven>` / `@EnableWebMvc`).
그리고 이 핸들러 매핑은 **기본적으로 "자기가 사는 컨텍스트(자식)의 `@Controller`만" 스캔**한다
(`detectHandlerMethodsInAncestorContexts = false` 가 기본값).

```
자식 컨텍스트: RequestMappingHandlerMapping 있음
   └ "내 컨텍스트의 @Controller 찾자" → 자식만 봄, 부모는 안 올라감
부모 컨텍스트: @Controller 넣어둠
   └ 아무도 안 찾아줌 → URL 매핑 등록 안 됨 → 404
```

- 즉 **빈 참조 방향(자식→부모만 가능)** + **핸들러 매핑 스캔 범위(자식만)** 가 겹쳐서,
  부모의 컨트롤러는 "있긴 한데 요청이 연결 안 되는" 상태가 된다.
- 반대로 Service를 부모에 두는 건 OK — 자식(Controller)이 부모(Service)를 올려다볼 수 있으니까.

#### "뭐가 어디 들어가야 한다"는 정해진 표준 목록은 없다

표준 목록은 없지만, 위 메커니즘 때문에 사실상 이렇게 굳어졌다:

| 빈 | 자식(Servlet)에 둬야 동작 | 부모(Root)에 둬야 적절 |
|----|--------------------------|------------------------|
| `@Controller` | ⭕ (핸들러 매핑이 자식만 스캔) | ❌ 매핑 안 됨 |
| `ViewResolver`, `Interceptor`, `HandlerMapping` | ⭕ DispatcherServlet이 자기 컨텍스트에서 찾음 | △ |
| `@Service`, `@Repository`, DataSource, TX, MyBatis | △ | ⭕ (여러 서블릿이 공유, 트랜잭션/AOP 안정) |

→ "규칙"이라기보다 **"이 위치에 둬야 그 빈이 제 역할을 한다"는 기술적 필연**.

#### "자율이라며?" 에 대한 정확한 답

- **컨테이너를 2개로 나눌지 1개로 둘지** = 자율 ⭕
- **2개로 나눴을 때 뭘 어디 넣느냐** = 자율 ❌, 위 메커니즘을 따라야 동작.

#### 반전: 1개로 합쳐도 된다

- 부모를 비우고 **모든 빈(Controller+Service)을 자식 `servlet-context.xml` 하나에** 다 넣으면 → 컨텍스트 1개로 다 돌아간다. (자식 하나뿐이면 부모-자식 문제 자체가 없음)
- **Spring Boot가 바로 이 방식** — 컨텍스트 1개에 전부. 그래서 부트에선 "어디 넣지?" 고민이 없었던 것.
- 결론: **나누는 순간 위 규칙을 지켜야 하고, 안 나누면 그 고민이 없다.**

---

## 5-2. 두 컨텍스트는 "자동"으로 생기나? — 그릇 vs 내용물

> 헷갈리기 딱 좋은 포인트: "스프링이 알아서 2개 만들어주나?" → **아니다. web.xml에 등록한 두 줄 때문에 생긴다.**

### 핵심: "그릇(컨테이너)"과 "내용물(빈)"을 분리해서 보기

```
                    그릇(컨테이너)을 만드는 것       내용물(빈)을 채우는 것
                    = "누가 new 하나"              = "무슨 설정파일 읽나"
─────────────────────────────────────────────────────────────────────
Root 컨텍스트(부모)   web.xml의 <listener>           contextConfigLocation
                    ContextLoaderListener           (root/servlet/security/ehcache)
                    ↑ 등록돼 있으니까 만든다

Servlet 컨텍스트(자식) web.xml의 <servlet>            init-param의
                    DispatcherServlet                servlet-context.xml
                    ↑ 등록돼 있으니까 만든다
```

- **그릇을 만드는 건 "자동"이 아니라 "등록 때문"이다.**
  - `<listener>`에 `ContextLoaderListener`를 적어놔서 → 톰캣이 실행 → 리스너가 **부모 컨테이너(빈 껍데기)를 new**.
  - `<servlet>`에 `DispatcherServlet`을 적어놔서 → 톰캣이 실행 → 서블릿이 **자식 컨테이너(빈 껍데기)를 new**.
  - 등록을 지우면 안 생긴다. (`<listener>`를 지우면 부모 컨텍스트 자체가 없음)
- **내용물을 채우는 건 설정파일 매핑이 맞다.**
  - 부모 그릇 ← `contextConfigLocation`의 파일들 / 자식 그릇 ← 서블릿 `init-param`의 파일.

### "등록"의 정확한 의미

이 두 줄은 *컨텍스트를 직접 등록*하는 게 아니라 **컨텍스트를 만들어줄 "일꾼"을 등록**하는 것이다.

```
<listener> ContextLoaderListener  → 톰캣이 실행 → 리스너가  부모 컨텍스트를 new
<servlet>  DispatcherServlet      → 톰캣이 실행 → 서블릿이  자식 컨텍스트를 new
```

- `ContextLoaderListener` = **부모 컨텍스트를 만드는 일꾼** (등록 → 부모 생김)
- `DispatcherServlet` = **자식 컨텍스트를 만드는 일꾼** (등록 → 자식 생김)

### 등록 줄 ↔ 컨텍스트 ↔ 설정파일 매핑표

| 등록 줄 | 만드는 컨텍스트 | 읽는 설정파일 | 파라미터 위치 |
|--------|---------------|--------------|--------------|
| `<listener>` ContextLoaderListener | **부모**(Root) | `contextConfigLocation`의 4개 (root/servlet/security/ehcache) | `<context-param>` (앱 전역) |
| `<servlet>` DispatcherServlet | **자식**(Servlet) | `servlet-context.xml` | `<init-param>` (서블릿 전용) |

> ⚠️ 헷갈림 포인트: `param-name`은 둘 다 똑같이 `contextConfigLocation`이다.
> **어디에 적혀 있느냐**로 갈린다 — `<context-param>`에 있으면 부모 것, `<servlet>`의 `<init-param>`에 있으면 자식 것.

### 개수도 고정이 아니다 (이해 확인 포인트)

- **부모(Root): 0개 또는 1개** — 리스너 등록하면 1개, 안 하면 0개.
- **자식(Servlet): 0개~N개** — DispatcherServlet을 여러 개 등록하면 자식도 여러 개.

```
<servlet> apiServlet   → 자식 컨텍스트 A (api 전용 빈)
<servlet> adminServlet → 자식 컨텍스트 B (admin 전용 빈)
        둘 다 같은 Root(부모) 하나를 공유
```
- DispatcherServlet이 `appServlet` 하나뿐이면 → **부모 1 + 자식 1** 구조.

### Boot와 비교 (왜 "자동"처럼 느꼈나)

- **Boot:** web.xml이 없어 등록할 곳도 없음 → `SpringApplication.run()`이 **코드로 컨텍스트를 자동 생성**해서 진짜 자동처럼 느껴짐 (게다가 보통 부모-자식 안 나누고 1개로 통합).
- **레거시:** "자동"이 아니라 **web.xml에 적은 등록 줄이 트리거**라서 눈에 직접 보였던 것.

### 한 문장 요약

> **리스너 등록 → 부모 컨텍스트 생성(context-param 파일들로 채움), DispatcherServlet 등록 → 자식 컨텍스트 생성(init-param의 servlet-context.xml로 채움).**

---

## 6. 전체 통합 그림 (흐름 + 컨텍스트)

```
[톰캣 시작]
   │
   ├─▶ ServletContext 생성 (톰캣, 앱당 1개) ────────────┐
   │                                                  │ 이 상자 안에
   ├─▶ web.xml 읽음                                    │ 모든 게 담김
   │     <context-param> → ServletContext에 값 저장     │
   │                                                  │
   ├─▶ ContextLoaderListener 실행 (스프링 깨어남)       │
   │     └ contextConfigLocation 꺼냄                  │
   │     └ ★Root ApplicationContext(부모) 생성★ ───────┤
   │        (DataSource, Service, Security, Cache)     │
   │     └ 만든 컨테이너를 ServletContext에 등록         │
   │                                                  │
   └─▶ DispatcherServlet 실행                          │
         └ servlet-context.xml 읽음                    │
         └ ★Servlet ApplicationContext(자식) 생성★ ────┘
            (Controller, ViewResolver)
            └ 부모를 getParent()로 연결

[요청 들어옴] → DispatcherServlet이 자식에서 Controller 찾고
              → Controller가 부모의 Service를 가져다 씀
```

---

## 7. web.xml 매핑 예시

```xml
<context-param>
  <param-name>contextConfigLocation</param-name>   <!-- ④ 스프링이 꺼낼 "약속된 키 이름" -->
  <param-value>
      /WEB-INF/config/root-context.xml             <!-- ┐ -->
      /WEB-INF/config/spring/servlet-context.xml   <!-- │ ⑤ 루트 컨테이너가 -->
      /WEB-INF/config/spring/security-context.xml  <!-- │    읽어들일 파일 목록 -->
      /WEB-INF/config/spring/ehcache-context.xml   <!-- ┘ -->
  </param-value>
</context-param>
```

| 파일 | 역할 |
|------|------|
| `root-context.xml` | 공통 뒷단 핵심 빈: DataSource, 트랜잭션, MyBatis, Service 계층 |
| `servlet-context.xml` | 웹/화면(MVC): Controller 스캔, ViewResolver, 인터셉터, 정적 리소스 |
| `security-context.xml` | Spring Security: 로그인/로그아웃, 인증·인가, 세션, CSRF |
| `ehcache-context.xml` | EhCache 캐시: 공통코드 등 자주 쓰는 데이터 메모리 캐싱 |

### 규칙 포인트

- `contextConfigLocation` **이름은 못 바꾼다** — ContextLoaderListener가 이 이름으로 찾기로 정해놨다(계약 ②). 오타 나면 스프링이 못 찾음.
- `<param-value>` 안의 **파일 경로/이름/개수는 자유** — 여기 적은 만큼만, 적은 그대로 읽는다.
- **하위 폴더 자동 탐색 안 함.** 더 끌어오려면:
  - 와일드카드 패턴: `classpath*:/spring/context-*.xml`, 재귀는 `**` (`classpath*:/config/**/*.xml`)
  - 또는 XML 내부에서 `<import resource="..."/>`로 연결.

### 참고: 환경 분기 파라미터 예시

```xml
<context-param>
  <param-name>app.dataSource</param-name>
  <param-value>primary</param-value>   <!-- primary, secondary 선택 -->
</context-param>
<context-param>
  <param-name>app.active</param-name>
  <param-value>local</param-value>  <!-- local, dev, prod 선택 -->
</context-param>
```
- 설정 파일 내부에서 **데이터소스 선택**, **로컬/개발/운영** 환경을 분기하는 데 사용되는 값.

---

## 8. Spring Boot와의 대조 (왜 안 보였나)

| 항목 | 레거시 Spring | Spring Boot |
|------|---------------------------|-------------|
| 톰캣 | 외부 톰캣에 WAR 배포 | 내장 톰캣 (jar 안에 포함) |
| web.xml | 직접 작성 (표준 파일) | **없음** (자바 코드로 대체) |
| 부팅 진입점 | 톰캣 → web.xml → 리스너/서블릿 | `main()` → `SpringApplication.run()` |
| Context 분리 | Root + Servlet 2개 | 보통 1개로 통합 |
| 설정 방식 | XML (`contextConfigLocation`) | 어노테이션 + `application.yml` |
| 빈 등록 | `<bean>`, `<context:component-scan>` | `@Component` 자동스캔 |

> Boot는 위 ①~⑦ 부팅 과정을 `@SpringBootApplication` + 내장 톰캣이 **코드로 자동 수행**한다.
> 그래서 Boot만 쓰면 ServletContext / 2개 컨텍스트 / web.xml을 직접 볼 일이 없었던 것.

---

## 9. web.xml 예시 직접 짚어보기 (1:1 매핑)

> 파일: `src/main/webapp/WEB-INF/web.xml`
> 앞에서 본 부팅 흐름 ①~⑦이 web.xml 어디에 해당하는지 줄 단위로 짚는다.

### 9-1. 전체 구조 한눈에

```
web.xml
├─ <context-param>      ← 톰캣이 ServletContext에 "값만 저장" (흐름 ②)
│   ├─ app.dataSource / app.active   (환경 분기값)
│   ├─ contextConfigLocation     (★루트 컨텍스트가 읽을 파일 목록)
│   └─ log4jConfiguration        (로그 설정 경로)
│
├─ <listener>           ← 톰캣이 "실행" → 스프링/기타 깨어남 (흐름 ③)
│   ├─ ContextLoaderListener     (★루트 ApplicationContext 생성 = 부모)
│   ├─ HttpSessionEventPublisher (세션 이벤트 → Security)
│   └─ EhCache 리스너 2개
│
├─ <filter>             ← 모든 요청 앞단에서 가로채는 필터들
│   ├─ requestContextFilter      (/login.do 한정)
│   ├─ springSecurityFilterChain (/* — Spring Security 진입점)
│   └─ encodingFilter            (/* — UTF-8)
│
├─ <servlet>            ← 톰캣이 "실행" → DispatcherServlet (흐름 ⑥⑦)
│   └─ appServlet = DispatcherServlet
│       └ contextConfigLocation = servlet-context.xml (★자식 컨텍스트)
│
├─ <session-config>     세션 타임아웃
├─ <error-page>         에러코드별 화면
└─ <security-constraint> HEAD/PUT/DELETE/TRACE/OPTIONS 차단
```

### 9-2. 흐름 ②  — `<context-param>` (라인 18~47)

```xml
<context-param>                                      <!-- 톰캣이 ServletContext에 -->
  <param-name>contextConfigLocation</param-name>     <!-- 값만 저장. 스프링은 아직 안 깸 -->
  <param-value>
      /WEB-INF/config/root-context.xml               <!-- 부모가 읽을 -->
      /WEB-INF/config/spring/servlet-context.xml     <!-- 파일 목록 -->
      /WEB-INF/config/spring/security-context.xml
      /WEB-INF/config/spring/ehcache-context.xml
  </param-value>
</context-param>
```

- 이 `contextConfigLocation`은 **리스너(ContextLoaderListener)가 꺼내 쓸 키**다 (흐름 ④).
- `app.dataSource=primary`, `app.active=local` 도 같은 ServletContext에 저장돼, 설정 XML 내부에서 데이터소스·로컬/개발/운영 분기에 사용.

### 9-3. 흐름 ③④⑤  — `<listener>` (라인 52~63)

```xml
<listener>
  <listener-class>org.springframework.web.context.ContextLoaderListener</listener-class>
</listener>
```
- **이 한 줄이 흐름 ③의 주인공.** 톰캣이 이 리스너를 `new` 해서 실행 → 깨어난 리스너가 `contextConfigLocation`(9-2)을 읽어 **★Root ApplicationContext(부모) 생성★** (흐름 ④⑤).
- 즉 위 4개 XML(root/servlet/security/ehcache)이 **전부 부모 컨텍스트로** 로딩됨.

```xml
<listener>HttpSessionEventPublisher</listener>   <!-- 세션 생성/소멸 이벤트를 Security에 알림 (동시로그인 제어 등) -->
<listener>EhCacheCacheManager</listener>         <!-- EhCache 캐시 매니저 -->
<listener>EhCacheManagerFactoryBean</listener>   <!-- EhCache 매니저 팩토리 -->
```
- 나머지 리스너는 스프링 컨텍스트와 별개로 톰캣이 실행하는 부가 리스너.

### 9-4. 필터 — `<filter>` (라인 64~91)

> 필터는 컨텍스트(빈 컨테이너)와 별개로, **요청이 서블릿에 닿기 전에** 톰캣이 먼저 태우는 관문이다.

| 필터 | url-pattern | 역할 |
|------|-------------|------|
| `requestContextFilter` | `/login.do` | 요청 스코프 바인딩 (로그인 처리 시 RequestContext 사용 가능하게) |
| `springSecurityFilterChain` | `/*` | **Spring Security 진입점.** 실제 보안 처리는 `security-context.xml`의 빈들이 담당하고, 이 필터는 그 체인으로 위임(`DelegatingFilterProxy`) |
| `encodingFilter` | `/*` | 요청 인코딩 UTF-8 강제 |

- `springSecurityFilterChain`이 `DelegatingFilterProxy`인 점이 포인트: **필터 껍데기는 web.xml에**, **실제 보안 빈은 스프링 컨텍스트에** 있고 둘을 연결한다. (= 톰캣 세계 ↔ 스프링 세계 접점의 또 다른 예)
- `RsaFilter`는 주석 처리됨(라인 100~111, 운영/개발 서버와 싱크 맞추려 비활성).

### 9-5. 흐름 ⑥⑦  — `<servlet>` (라인 113~126)

```xml
<servlet>
  <servlet-name>appServlet</servlet-name>
  <servlet-class>org.springframework.web.servlet.DispatcherServlet</servlet-class>
  <init-param>
    <param-name>contextConfigLocation</param-name>
    <param-value>/WEB-INF/config/spring/servlet-context.xml</param-value>  <!-- ★자식이 읽을 파일 -->
  </init-param>
  <load-on-startup>1</load-on-startup>   <!-- 톰캣 시작 시 바로 초기화(요청 안 기다림) -->
</servlet>
<servlet-mapping>
  <servlet-name>appServlet</servlet-name>
  <url-pattern>/</url-pattern>            <!-- 기본 매핑: 다른 서블릿이 안 잡는 요청 전부 -->
</servlet-mapping>
```
- **이게 흐름 ⑥⑦의 주인공.** 톰캣이 `DispatcherServlet`을 실행 → 자기 `init-param`의 `servlet-context.xml`을 읽어 **★Servlet ApplicationContext(자식) 생성★**.
- `load-on-startup=1` 이라 요청을 기다리지 않고 톰캣 시작 시점에 바로 자식 컨텍스트를 만든다.
- 여기서 `contextParam`이 아니라 `init-param`(서블릿 전용 파라미터)인 점에 주목: **부모 설정은 context-param, 자식 설정은 servlet의 init-param.**

### 9-6. ⚠️ 눈여겨볼 점 — servlet-context.xml 중복 로딩

```
부모(context-param, 라인 36)   → servlet-context.xml 포함
자식(servlet init-param, 118)  → servlet-context.xml
```
- **같은 `servlet-context.xml`이 부모와 자식 양쪽에 등록**되어 있다.
- 결과적으로 servlet-context.xml의 빈들이 **부모·자식 두 컨텍스트에 모두** 만들어질 수 있다(이론상 중복 생성).
- 보통 정석은 *부모 = root/security/ehcache*, *자식 = servlet* 로 깔끔히 나누는 것. 만약 부모 목록에도 servlet-context.xml을 넣으면, 5장에서 말한 **"양쪽 중복 스캔" 위험 패턴**에 해당한다.
  - 그래도 servlet-context.xml 내부의 컴포넌트 스캔 범위가 Controller 위주라면 충돌이 표면화되지 않을 수 있다.
  - 손댈 일이 생기면 *부모 목록에서 servlet-context.xml을 빼는 것*이 일반적 정리 방향(단, 실제 빈 의존관계 확인 후 신중히).

### 9-7. 나머지 (참고)

| 영역 | 라인 | 내용 |
|------|------|------|
| `<session-config>` | 128~130 | 세션 타임아웃 7200초 (주석엔 10분 가이드 언급되나 실제값은 7200초=2시간) |
| `<error-page>` | 131~158 | 404 → 전용 페이지, 400/401/402/500/501/503 → 공통 serviceError.jsp |
| `<security-constraint>` | 159~172 | HEAD/PUT/DELETE/TRACE/OPTIONS 메서드를 `empty_role`로 막아 사실상 차단(웹취약점 조치) |

### 9-8. 예시 기준 최종 정리

```
톰캣
 ├─(②) context-param 저장: contextConfigLocation = [root, servlet, security, ehcache]
 ├─(③) ContextLoaderListener 실행
 │       └(④⑤) ★Root 컨텍스트(부모)★ = 위 4개 XML 전부 로딩
 ├─     필터 3개 등록 (security 필터는 부모 컨텍스트의 보안 빈으로 위임)
 └─(⑥) DispatcherServlet(appServlet) 실행, load-on-startup=1
         └(⑦) ★Servlet 컨텍스트(자식)★ = servlet-context.xml 로딩
                └ 부모를 통해 Service/DataSource/Security 빈 사용

[요청] → encoding/security 필터 통과 → DispatcherServlet('/')
        → 자식의 Controller → 부모의 Service → ... → View(JSP)
```

---

## 10. 머릿속에 남길 3줄 요약

1. **web.xml만 표준 규격**(톰캣이 무조건 읽음). 나머지 스프링 xml은 이름·위치 자유, 경로만 web.xml로 알려줄 뿐.
2. **읽는 주체가 다르다:** web.xml = 톰캣, 스프링 설정 = 스프링 객체(Listener/DispatcherServlet). 톰캣이 스프링을 깨우고, 스프링이 자기 설정을 읽는 릴레이.
3. **컨테이너 2개:** 공통 뒷단(Root, 부모) + 웹 앞단(Servlet, 자식). 자식은 부모 빈을 쓸 수 있지만 반대는 안 됨. Boot는 이걸 자동화·통합해서 안 보였던 것.
