---
title: "테스트 코드 — 유닛 · 통합 · 어노테이션 · 작성법 (레거시 Spring + JUnit5)"
date: 2026-06-26 09:00:00 +0900
categories: [Spring]
tags: [Spring, Test, JUnit5, Mockito, Legacy]
---

> 레거시 Spring(비-Boot) + JUnit5 + MyBatis 환경 기준 테스트 정리.
> 핵심 질문: **유닛/통합이 뭐가 다른가? 어노테이션은 왜 갈리는가? 실제로 어떻게 쓰나?**
> (예시는 `MemberService` / `MemberMapper` / `MemberVO` 같은 일반 도메인으로 든다.)

---

## 0. 가장 큰 한 줄

> **`@ExtendWith`에 `MockitoExtension`이 오면 유닛, `SpringExtension`이 오면 통합.**
> 여기서 모든 게 갈린다.

---

## 1. 유닛 vs 통합 — 기준은 "진짜 인프라를 쓰냐"

| | 유닛 테스트 | 통합 테스트 |
|---|---|---|
| 대상 | **한 클래스만** | **여러 계층 + 실제 인프라** |
| 의존성 | mock(가짜)으로 대체 | 진짜 빈 |
| 스프링 컨테이너 | ❌ 안 띄움 | ⭕ 띄움 |
| DB | ❌ 안 씀 | ⭕ 진짜 DB |
| 속도 | 빠름 | 느림 |
| 도구 | Mockito | Spring TestContext |

> 비유: 유닛 = **부품 하나만** 떼서 시험대에서 / 통합 = **조립한 채로** 전원 켜고.

---

## 2. 공통 — JUnit 5 (유닛·통합 안 가림)

테스트의 뼈대. 어떤 테스트든 쓴다.

| 어노테이션 | 역할 |
|---|---|
| `@Test` | 테스트 메서드 표시 |
| `@BeforeEach` / `@AfterEach` | **각** 테스트 전/후 실행 (준비·정리) |
| `@BeforeAll` / `@AfterAll` | 클래스 전체에서 **한 번** (static 메서드) |
| `@DisplayName("...")` | 테스트 이름을 한글 등으로 표시 |
| `@Disabled` | 그 테스트 잠시 끔 |
| `@Nested` | 테스트를 그룹으로 묶음 |
| `@ParameterizedTest` + `@ValueSource`/`@CsvSource` | 값 바꿔가며 반복 실행 |

```java
@DisplayName("회원 목록 조회")
@Test
void getList() { ... }
```

---

## 3. 유닛 테스트 — Mockito 계열 (스프링·DB 안 띄움)

| 어노테이션 | 역할 |
|---|---|
| `@ExtendWith(MockitoExtension.class)` | Mockito 켜기 (**SpringExtension 대신**) |
| `@Mock` | 가짜 의존성 (예: 가짜 `MemberMapper`) |
| `@InjectMocks` | mock들을 주입받을 **진짜 테스트 대상** (예: `MemberService`) |
| `@Spy` | 진짜 객체인데 일부 메서드만 가짜 |
| `@Captor` | 메서드에 넘어간 인자를 캡처해 검증 |

```java
@ExtendWith(MockitoExtension.class)        // 스프링 안 띄움 → 빠름
class MemberServiceTest {

    @Mock MemberMapper mapper;             // 가짜 매퍼
    @InjectMocks MemberService service;    // 진짜 서비스(가짜 매퍼가 주입됨)

    @Test
    void 가입_중복이메일이면_예외() {
        // given: 가짜 매퍼가 "이미 있다"고 답하도록
        given(mapper.existsByEmail("a@a.com")).willReturn(true);

        // when & then: 비즈니스 로직이 예외를 던지는가
        assertThrows(IllegalStateException.class,
                     () -> service.register("a@a.com"));
    }
}
```

> **언제 쓰나:** 서비스에 **비즈니스 로직(분기·규칙)** 이 있을 때. DB 없이 빠르게 분기만 검증.
> 매퍼를 그대로 호출만 하는 **얇은 래퍼**라면 유닛 가치가 낮다(검증할 로직이 없음).

> 💡 Mockito 핵심 동사
> - `given(...).willReturn(...)` / `when(...).thenReturn(...)` — 가짜가 이렇게 답하라(stubbing)
> - `verify(mapper).save(...)` — 그 메서드를 호출했는지 검증
> - `any()`, `eq(...)` — 인자 매처

---

## 4. 통합 테스트 — Spring TestContext 계열 (컨테이너·DB 띄움)

| 어노테이션 | 역할 |
|---|---|
| `@ExtendWith(SpringExtension.class)` | 스프링 테스트 컨텍스트 켜기 (**Mockito 대신**) |
| `@ContextConfiguration(...)` | **어떤 설정으로 컨테이너를 띄울지** ← 통합의 핵심 |
| `@Autowired` | 컨테이너에서 진짜 빈 주입 (진짜 DB 연결됨) |
| `@Transactional` | 테스트 끝나면 **자동 롤백** → DB 안 더럽힘 ⭐ |
| `@Sql("classpath:...sql")` | 테스트 전에 SQL 실행 (데이터 세팅) |
| `@ActiveProfiles("test")` | 프로파일 지정 (test용 DB 등) |
| `@TestPropertySource` | 테스트 전용 프로퍼티 주입 |
| `@DirtiesContext` | 이 테스트가 컨텍스트를 망치니 다음엔 새로 만들라 |

`@ContextConfiguration`은 XML/자바컨피그 둘 다 가능:
```java
@ContextConfiguration("classpath:applicationContext.xml")     // XML 설정
@ContextConfiguration(classes = AppConfig.class)              // 자바 설정
```

```java
@ExtendWith(SpringExtension.class)                 // 통합
@ContextConfiguration("classpath:applicationContext.xml")
@Transactional                                     // 끝나면 롤백 (INSERT 테스트해도 DB 깨끗)
class MemberMapperTest {

    @Autowired MemberMapper mapper;                // 진짜 DB 친다

    @Test
    void 단건조회() {
        MemberVO vo = mapper.get(1L);              // 실제 SELECT 실행
        assertNotNull(vo);
    }
}
```

> 💡 **왜 Mapper/Repository는 통합이어야 하나:** SQL·컬럼매핑은 **진짜 DB가 있어야** 검증된다.
> mock으론 SQL 오타나 매핑 누락을 못 잡는다.

### `@Transactional`이 테스트에선 "롤백"인 이유
- 일반 코드에서 `@Transactional` = commit/rollback 트랜잭션 경계.
- **테스트 클래스에 붙으면 스프링이 기본적으로 끝나고 rollback** → INSERT/UPDATE를 돌려도 DB가 원상복구.
- 진짜 커밋이 필요하면 `@Commit` 또는 `@Rollback(false)`.

---

## 5. 컨트롤러 테스트 — MockMvc

컨트롤러는 **MockMvc**로 가짜 HTTP 요청을 보내 검증한다. 짜는 방식에 따라 유닛/통합 둘 다.

| 방식 | 어떻게 | 성격 |
|---|---|---|
| `MockMvcBuilders.standaloneSetup(controller)` | 컨트롤러 1개 + **mock 서비스** | 유닛 성격 (빠름) |
| `MockMvcBuilders.webAppContextSetup(ctx)` | `@WebAppConfiguration` + 전체 컨텍스트 | 통합 |

| 어노테이션 | 역할 |
|---|---|
| `@WebAppConfiguration` | 웹 컨텍스트(ServletContext) 환경 띄우기 |
| `@ContextConfiguration({root, servlet})` | 컨트롤러까지 띄우려면 web 설정도 포함 |

```java
mockMvc.perform(get("/members"))
       .andExpect(status().isOk())
       .andExpect(model().attributeExists("list"))
       .andExpect(view().name("member/list"));
```

---

## 6. ⚠️ Boot 전용 — 레거시(비-Boot)에선 못 씀

인터넷 예제 대부분이 Boot라 아래를 쓰는데, **비-Boot 환경엔 없다.**

| Boot 어노테이션 | Boot에서 역할 | 레거시 대응 |
|---|---|---|
| `@SpringBootTest` | 전체 컨텍스트 통합테스트 | `@ExtendWith(SpringExtension)` + `@ContextConfiguration` |
| `@WebMvcTest` | 웹 계층만 슬라이스 | MockMvc standalone 수동 |
| `@DataJpaTest` | DB 계층만 슬라이스 | Mapper/Repository 통합 수동 |
| `@MockBean` | 컨텍스트의 빈을 mock으로 교체 | (없음 — Mockito 수동) |
| `@AutoConfigureMockMvc` | MockMvc 자동 주입 | 수동 빌더 |

> 레거시 = 어노테이션 **직접 조합** / Boot = `@SpringBootTest` **한 방**. 마이그레이션하면 확 줄어든다.

---

## 7. 계층별 테스트 전략

| 계층 | 뭘로? | 이유 |
|---|---|---|
| **Mapper / Repository** | **통합** (진짜 DB) | SQL·매핑 검증. mock으론 못 잡음 |
| **Service** | **유닛** (mock 의존성) | 비즈니스 로직(분기·규칙)을 DB 없이 빠르게 |
| **Controller** | **MockMvc** (standalone=유닛 / 전체=통합) | URL매핑·model·뷰이름·상태코드 |

---

## 8. 테스트 코드 작성하는 법 (실전)

### 8-1. 기본 골격 — given / when / then (AAA 패턴)
모든 테스트는 **준비 → 실행 → 검증** 3단계.

```java
@Test
void 메서드_상황_기대결과() {
    // given (준비): 입력·전제 세팅
    Long id = 1L;

    // when (실행): 테스트 대상 호출
    MemberVO vo = mapper.get(id);

    // then (검증): 결과가 기대와 같은지 단언(assert)
    assertNotNull(vo);
    assertEquals("kim", vo.getName());
}
```

> 핵심: **한 테스트는 한 가지만 검증.** "조회하면 null 아니다"와 "이름이 kim이다"는 의도가 다르면 나눈다.

### 8-2. 단언(assertion) — 결과를 확인하는 도구

**(A) JUnit5 기본** (`org.junit.jupiter.api.Assertions`)

| 메서드 | 의미 |
|---|---|
| `assertEquals(기대, 실제)` | 같은가 |
| `assertNotNull(x)` / `assertNull(x)` | null 아닌가 / null인가 |
| `assertTrue(조건)` / `assertFalse(조건)` | 참/거짓 |
| `assertThrows(예외.class, () -> ...)` | 그 예외가 터지는가 |
| `assertAll(...)` | 여러 단언을 묶어 한 번에 (하나 실패해도 나머지 다 확인) |

```java
import static org.junit.jupiter.api.Assertions.*;

assertEquals(30, vo.getAge());
assertThrows(IllegalArgumentException.class, () -> service.get(null));
```

**(B) AssertJ** — 더 읽기 좋은 단언 (의존성 `assertj-core` 추가 시)
```java
assertThat(list).hasSize(3);
assertThat(vo.getName()).isEqualTo("kim");
assertThat(vo.getAge()).isGreaterThan(0);
```

### 8-3. 네이밍 — 테스트 이름은 "문장"으로
- 메서드명: `대상_상황_기대` (예: `get_없는id_null반환`)
- 또는 `@DisplayName("없는 id로 조회하면 null")` 로 한글 문장.
- 좋은 이름 = 실패했을 때 **이름만 봐도 뭐가 깨졌는지** 안다.

### 8-4. 실행하는 법
```bash
./gradlew test                            # 전체 테스트
./gradlew test --tests "*MemberMapper*"   # 특정 클래스만
```
- IDE(IntelliJ): 테스트 클래스/메서드 옆 ▶ 버튼.
- 결과 리포트: `build/reports/tests/test/index.html`
- ⚠️ 통합테스트는 **테스트용 DB가 떠 있어야** 한다.

### 8-5. 완성 예제 ① — Mapper 통합테스트
```java
@ExtendWith(SpringExtension.class)
@ContextConfiguration("classpath:applicationContext.xml")
@Transactional
class MemberMapperTest {

    @Autowired MemberMapper mapper;

    @Test
    @DisplayName("회원 목록 조회 - 비어있지 않다")
    void getList() {
        List<MemberVO> list = mapper.getList();    // 실제 SELECT
        assertNotNull(list);
    }

    @Test
    @DisplayName("단건 조회 - 1번 회원이 있다")
    void get() {
        MemberVO vo = mapper.get(1L);
        assertNotNull(vo);
        assertEquals(1L, vo.getId());
    }
}
```

### 8-6. 완성 예제 ② — Service 유닛테스트 (Mockito)
```java
@ExtendWith(MockitoExtension.class)
class MemberServiceTest {

    @Mock MemberMapper mapper;
    @InjectMocks MemberService service;

    @Test
    @DisplayName("가입 - 정상이면 매퍼 save 호출")
    void register() {
        // given
        given(mapper.existsByEmail("a@a.com")).willReturn(false);

        // when
        service.register("a@a.com");

        // then
        verify(mapper).save(any());        // save가 불렸는지
    }

    @Test
    @DisplayName("가입 - 중복 이메일이면 예외")
    void register_중복() {
        given(mapper.existsByEmail("a@a.com")).willReturn(true);
        assertThrows(IllegalStateException.class,
                     () -> service.register("a@a.com"));
    }
}
```

### 8-7. 작성 순서 팁
1. **실패하는 시나리오부터** 떠올린다 ("없는 id면 null이어야 해").
2. given/when/then 골격 깔고 → **then(단언)부터** 적으면 "뭘 검증할지"가 명확해진다.
3. 한 번에 다 짜지 말고 **메서드 하나씩** 초록불 확인.
4. 순서: Mapper(통합) → Service(유닛) → Controller(MockMvc).

---

## 한 줄 요약

> **갈리는 핵심:** `@ExtendWith(MockitoExtension)` = 유닛(DB 없음) / `@ExtendWith(SpringExtension)` = 통합(진짜 DB).
> 공통은 JUnit5(`@Test`·`@BeforeEach`), 통합엔 `@ContextConfiguration`+`@Transactional`(롤백), 컨트롤러는 MockMvc.
> `@SpringBootTest` 같은 건 **Boot 전용**이라 레거시에선 못 쓰고, 마이그레이션하면 한 방으로 줄어든다.
> 작성은 **given-when-then** 골격에 단언으로 검증 — **Mapper는 통합 / Service는 유닛 / Controller는 MockMvc**.
