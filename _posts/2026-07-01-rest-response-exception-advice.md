---
title: "REST 응답 설계와 예외 어드바이스 분리 — @ControllerAdvice가 JSON을 안 주는 문제"
date: 2026-07-01 09:00:00 +0900
categories: [Study, Trouble]
tags: [Spring, REST, ExceptionHandler, ControllerAdvice, Legacy]
---

> 레거시 Spring(비-Boot, Spring 5.2 / Java 11 / MyBatis) 환경 기준.
> `@RestController`에서 예외를 던졌는데 JSON이 아니라 **JSP 에러 페이지(HTML)** 가 나가는 문제를 파고, REST 응답을 어떻게 설계하는지까지 정리한다.
> 예시는 `RestLectureController` / `EnrollmentException` / `BaseException` 도메인으로 든다.

---

## 0. 가장 큰 한 줄

> **`@ControllerAdvice`는 REST든 JSP든 안 가리고 전역으로 예외를 잡는다.**
> 그래서 뷰 반환용 어드바이스 하나만 있으면, REST 예외도 그 어드바이스가 낚아채서 `error_page`(HTML)를 내보낸다. → REST 클라이언트는 JSON을 못 받는다.

해결의 방향은 두 가지다.
1. REST 전용 `@RestControllerAdvice`를 따로 두고 → JSON을 반환
2. 두 어드바이스가 **담당 구역을 나누도록** 스코프/우선순위를 지정

---

## 1. 먼저: REST 응답은 실제로 어떻게 나가나

`@RestController`에서 그냥 DTO를 반환하면 끝이다.

```java
@GetMapping("/{id}/enrollees")
public List<String> getEnrollees(@PathVariable Long id) {
    return lectureService.getEnrollees(id);
}
```

나가는 응답:

```
HTTP/1.1 200 OK
Content-Type: application/json

["홍길동","김철수","이영희"]
```

### 흐름

```
return List<String>
  → @RestController(= @Controller + @ResponseBody)라 "반환값 = 응답 바디"
  → HttpMessageConverter(Jackson)가 자바 객체 → JSON 변환
  → 200 OK + application/json + 배열 바디
```

### 반환 타입별로 나가는 모양

| 반환 | JSON |
|---|---|
| `List<String>` | `["a","b","c"]` |
| `EnrollmentVO` | `{"id":1,"userName":"홍길동","status":"PENDING"}` |
| `List<EnrollmentVO>` | `[{...},{...}]` |
| `"hello"` | `hello` (문자열은 그대로 텍스트) |

**성공 응답은 이렇게 raw로 둬도 아무 문제 없다.** 문제는 항상 **에러 응답**에서 터진다.

---

## 2. 성공 응답 — 두 학파

### A. 그냥 DTO 반환 (HTTP 상태코드로 의미 전달)

```java
public List<String> getEnrollees(@PathVariable Long id) {
    return lectureService.getEnrollees(id);   // 200 + 바디
}
```

- 순수 REST 정석. 성공/실패는 **HTTP 상태코드**로, 데이터는 바디로.

### B. 공통 응답 포맷(envelope)으로 감싸기 — 국내 실무에서 매우 흔함

```java
public ApiResponse<List<String>> getEnrollees(@PathVariable Long id) {
    return ApiResponse.success(lectureService.getEnrollees(id));
}
```

```json
{ "success": true, "code": "OK", "message": "요청 성공", "data": ["홍길동","김철수"] }
```

- 성공이든 실패든 **응답 껍데기 구조가 항상 동일** → 프론트가 처리하기 편함.
- 국내 SI·금융권에서 거의 표준.

> **어느 쪽이 정답?** 팀 컨벤션 따라간다. 정답은 없고 **"한 프로젝트 안에선 무조건 통일"** 이 핵심.
> 다만 `ApiResponse`(B)를 쓰더라도 **HTTP 상태코드는 제대로 줘야 한다.** "무조건 200 주고 바디의 `success`로만 구분"은 안티패턴 — REST 원칙 위반이고 캐싱·모니터링이 꼬인다.

---

## 3. HTTP 상태코드 관례

| 코드 | 언제 |
|---|---|
| 200 OK | 조회·수정 성공 (바디 있음) |
| 201 Created | 생성 성공 (POST) |
| 204 No Content | 성공 + **바디 없음** (삭제·취소) |
| 400 Bad Request | 잘못된 요청 (예: 이미 취소됨) |
| 401 / 403 | 인증 / 권한 |
| 404 Not Found | 리소스 없음 |
| 500 | 서버 에러 |

### 취소·삭제는 204가 정석 — 3가지 방법

```java
// ① void → 자동 200 OK (제일 간단)
@DeleteMapping("/enrollments/{id}")
public void cancel(@PathVariable Long id) { service.cancel(id); }

// ② @ResponseStatus(204) + void → 204 (항상 고정 코드일 때 담백)
@DeleteMapping("/enrollments/{id}")
@ResponseStatus(HttpStatus.NO_CONTENT)
public void cancel(@PathVariable Long id) { service.cancel(id); }

// ③ ResponseEntity<Void> → 상태코드를 코드로 제어 (명시적, 유연)
@DeleteMapping("/enrollments/{id}")
public ResponseEntity<Void> cancel(@PathVariable Long id) {
    service.cancel(id);
    return ResponseEntity.noContent().build();   // 204
}
```

| 방식 | 언제 |
|---|---|
| `@ResponseStatus` + void | 항상 같은 코드 → 간결 |
| `ResponseEntity` | 조건 따라 코드 분기 / 헤더 추가 → 유연, 코드에 상태코드가 보임 |

둘 다 정답. 취소가 항상 성공/204면 `@ResponseStatus`가 담백하고, 명시성·유연성을 원하면 `ResponseEntity`.

---

## 4. 트러블: 에러 응답이 JSON이 아니라 HTML로 나간다

여기가 이 글의 핵심.

### 현상

REST 컨트롤러에서 예외를 던졌는데 응답이 JSON이 아니라 **JSP 에러 페이지(HTML)** 로 나갔다.

### 원인 — 어드바이스가 뷰 반환용이었다

```java
@ControllerAdvice                          // ← REST 전용 아님(전역)
public class CommonExceptionAdvice {

  @ExceptionHandler(BaseException.class)
  public String baseException(BaseException ex, Model model, HttpServletResponse response) {
      BaseErrorCode ec = ex.getErrorCode();
      response.setStatus(ec.getStatus().value());
      model.addAttribute("message", ex.getMessage());
      return "error_page";                 // ← JSP 뷰 이름을 반환!
  }
}
```

이 핸들러는 **뷰 이름(`"error_page"`)을 반환**한다 = JSP 렌더링용이다. 그런데:

```
RestLectureController에서 EnrollmentException 발생
  ↓  (EnrollmentException extends BaseException)
@ControllerAdvice의 @ExceptionHandler(BaseException.class)가 잡음
  ↓
return "error_page"  →  JSP 뷰 렌더링
  ↓
REST 클라이언트(fetch/앱)에 JSON 대신 HTML이 감  ❌
```

> 포인트: **`@ControllerAdvice`는 `@Controller`든 `@RestController`든 가리지 않고 전역으로 예외를 잡는다.** 상태코드(404 등)는 제대로 세팅되지만 **바디가 HTML** 이라 API 응답으로는 깨진다.

---

## 5. 해결 — REST 전용 어드바이스 분리

### 5-1. REST용 `@RestControllerAdvice` 추가

`@RestControllerAdvice` = `@ControllerAdvice` + `@ResponseBody` → **반환값을 JSON 바디로** 내보낸다.

```java
@RestControllerAdvice(annotations = RestController.class)
public class ApiExceptionAdvice {

    @ExceptionHandler(BaseException.class)
    public ResponseEntity<ErrorResponse> handle(BaseException ex) {
        BaseErrorCode ec = ex.getErrorCode();
        log.error("[{}] {}", ec.getCode(), ex.getMessage(), ex);

        ErrorResponse body = new ErrorResponse(ec.getCode(), ex.getMessage());
        return ResponseEntity.status(ec.getStatus()).body(body);   // 상태코드 + JSON 바디
    }
}
```

```json
{ "code": "ER_02", "message": "이미 취소되었습니다." }
```

기존 `BaseErrorCode`(code / message / status) 구조가 그대로 JSON에 매핑된다 — 애초에 이 응답을 염두에 둔 설계.

### 5-2. ⚠️ 함정: `@RestController`는 사실 `@Controller`이기도 하다

`@RestController`와 `@Controller` 정의를 까보면:

```java
@Controller          // ← @RestController 안에 @Controller가 들어있다!
@ResponseBody
public @interface RestController { }

@Component           // ← 반대로 @Controller 안엔 @RestController가 없다
public @interface Controller { }
```

그리고 `@ControllerAdvice(annotations = X)`는 **메타 어노테이션까지 뒤져서**(`AnnotatedElementUtils.hasAnnotation`) "이 클래스의 어노테이션 계층에 X가 있나?"를 본다.

#### 관계가 일방통행이다 (RestController ⊂ Controller)

이게 핵심이다. **`@RestController`는 `@Controller`이지만, `@Controller`는 `@RestController`가 아니다.** 붕어빵은 빵이지만, 빵이 붕어빵은 아닌 것과 같다.

컨트롤러 두 개를 각각 검사해 보면:

| 검사 | `RestLectureController`<br>(@RestController) | `LectureController`<br>(@Controller) |
|---|---|---|
| `@RestController` 있나? | ✅ (직접 붙음) | ❌ **없음** |
| `@Controller` 있나? | ✅ (메타로 딸려옴) | ✅ (직접 붙음) |

그래서 스코프 결과가 갈린다:

| 스코프 | 실제 매칭 | 성격 |
|---|---|---|
| `annotations = RestController.class` | `@RestController`만 ✅ | 좁음, 정확 |
| `annotations = Controller.class` | `@Controller` **+ `@RestController`도** ❌ | 넓음, 함정 |

- **넓은 쪽(Controller)으로 필터** → 좁은 쪽(RestController)까지 딸려온다.
- **좁은 쪽(RestController)으로 필터** → 넓은 쪽(일반 Controller)은 안 딸려온다.

포함 관계가 한 방향이라, `@RestController`는 `@Controller`의 **부분집합**이다. 부분집합만 골라내면(=`RestController.class`) 나머지 일반 컨트롤러는 자연히 빠진다. 그래서 REST 어드바이스를 `annotations = RestController.class`로 잡으면 **일반 `@Controller`는 애초에 담당 대상이 아니다.**

반대로 뷰 어드바이스를 `annotations = Controller.class`로 좁히면 **REST 컨트롤러까지 딸려 잡혀서** 다시 충돌한다. → 그래서 뷰 쪽은 스코프를 좁히지 말고 **전역 fallback**으로 두는 게 낫다.

### 5-3. 안전한 구조 — 2단계 필터 + @Order 우선순위

#### 어드바이스 선택은 2단계로 걸러진다

```
1단계 (스코프 필터): "이 어드바이스가 이 컨트롤러를 담당하나?"  ← annotations/basePackages
2단계 (@Order):      1단계를 통과한 후보가 여럿일 때만 순서로 결정
```

`@Order`는 **1단계를 통과한 후보들 사이에서만** 작동한다. 스코프에서 걸러진 어드바이스는 순서와 무관하게 경기에 안 나온다. 이 점이 "일반 컨트롤러가 실수로 JSON을 반환하지 않을까?"라는 걱정을 해소한다 — 일반 `@Controller` 예외는 REST 어드바이스의 **후보에도 못 든다.**

REST 어드바이스에 **우선권**을 주고, 뷰 어드바이스는 **나머지를 받는 fallback**으로 둔다.

```java
@Order(Ordered.HIGHEST_PRECEDENCE)                       // 우선권 ↑
@RestControllerAdvice(annotations = RestController.class) // @RestController만
public class ApiExceptionAdvice { /* BaseException → JSON */ }

@ControllerAdvice                                        // 스코프 없음 = 전역 fallback
public class ViewExceptionAdvice { /* BaseException → error_page */ }
```

동작:

```
@RestController 예외
  → ApiExceptionAdvice 매칭 O (RestController)
  → ViewExceptionAdvice 매칭 O (전역)
  → ApiExceptionAdvice가 우선권(@Order) → JSON 승 ✅

@Controller(JSP) 예외
  → ApiExceptionAdvice 매칭 X (RestController 아님 → 필터에서 제외)
  → ViewExceptionAdvice만 매칭 → error_page ✅
```

> **왜 @Order가 필요한가?**
> REST 컨트롤러 예외는 두 어드바이스가 **둘 다 잡을 자격**이 있다(메타 어노테이션 때문). 스프링은 **@Order 순서대로 훑어서 그 예외를 처리할 첫 번째 어드바이스를 선택**한다. 그래서 REST를 앞에 둬야 JSON 핸들러가 먼저 가로챈다. 순서를 안 주면 뷰 어드바이스가 먼저 잡아 HTML이 나갈 위험이 있다.
> JSP 예외는 REST 어드바이스의 스코프 필터에서 애초에 제외되므로 순서와 무관하다.

#### "그럼 view로 가야 할 요청이 JSON으로 새지 않나?" — 안 샌다

`@Order`가 REST를 앞에 뒀으니 일반 컨트롤러 요청도 JSON이 나가는 것 아닐까? 아니다. **스코프(1단계)가 먼저 걸러서**, 일반 `@Controller` 예외는 REST 어드바이스의 후보에도 들지 못한다.

| 예외 던진 컨트롤러 | ApiAdvice 후보? | ViewAdvice 후보? | @Order 작동? | 결과 |
|---|---|---|---|---|
| **일반 `@Controller`** | ❌ (스코프 탈락) | ✅ | 불필요 | **error_page** ✅ |
| **`@RestController`** | ✅ | ✅ (겸직 함정) | ✅ 작동 | **JSON** ✅ |

- 일반 컨트롤러: 후보가 ViewAdvice 하나뿐이라, REST를 아무리 먼저 order 해도 **JSON이 나갈 수가 없다.** REST 어드바이스는 그 경기장에 입장조차 못 했다.
- `@Order`가 실제로 일하는 경우는 **@RestController 예외(둘 다 후보)일 때 딱 하나**, 그리고 거기선 JSON이 원래 정답이다.

#### 집합으로 보면

```
        ┌─────────── @Controller 로 필터 (넓음) ───────────┐
        │                                                  │
        │   일반 @Controller들          @RestController들   │
        │   (LectureController)        ┌──────────────────┐│
        │                             │ @RestController   ││ ← 이것만 필터 (좁음)
        │                             │ .class 로 필터     ││
        │                             └──────────────────┘│
        └──────────────────────────────────────────────────┘
```

> 결국 **"이 요청이 view로 가냐 json으로 가냐"는 예외를 던진 컨트롤러의 타입이 결정**하고, 어드바이스는 거기에 맞춰 선택될 뿐이다. 스코프가 컨트롤러 타입으로 이미 갈라놓기 때문에 뒤섞일 일이 없다.

### 5-4. (대안) 패키지로 분리 — 더 명확

REST 컨트롤러가 한 패키지에 모여 있으면 메타 어노테이션 함정 없이 깔끔하다.

```java
@RestControllerAdvice(basePackages = "com.org.legacyproject.xxx.controller")
```

---

## 6. 정리

- **REST 응답 기본**: `@RestController`가 반환값을 Jackson으로 JSON 직렬화 → `200 OK` + 바디. 성공은 raw여도 OK.
- **성공 응답 설계**: DTO 직접(REST 정석) vs `ApiResponse` envelope(국내 실무 흔함). 하나로 통일이 핵심. envelope 써도 **상태코드는 제대로**.
- **상태코드**: 200/201/204/400/404 의미대로. 취소·삭제는 **204**(`@ResponseStatus` or `ResponseEntity.noContent()`).
- **트러블**: `@ControllerAdvice`가 전역이라 REST 예외도 잡아 `error_page`(HTML)로 내보냄 → REST 클라이언트가 JSON을 못 받음.
- **해결**: REST 전용 `@RestControllerAdvice`로 JSON 반환.
  - **관계는 일방통행**: `@RestController` ⊂ `@Controller`. RestController는 Controller이지만, Controller는 RestController가 아니다.
  - 그래서 `annotations = RestController.class`는 **일반 컨트롤러를 안 잡고**(좁음), `annotations = Controller.class`는 **REST까지 딸려 잡는다**(넓음, 함정).
  - **REST 어드바이스에 `@Order` 우선권 + JSP는 전역 fallback**, 또는 **패키지로 분리**.
- **어드바이스 선택은 2단계**: ①스코프 필터(annotations/basePackages)로 후보를 거르고 → ②겹치는 후보가 있을 때만 `@Order`로 결정. 일반 컨트롤러 예외는 ①에서 REST 어드바이스가 탈락하므로 JSON으로 샐 일이 없다.

### 한 줄 결론

> REST와 뷰는 **에러 응답 포맷이 다르다(JSON vs HTML)**. 어드바이스를 분리하되, `@RestController`가 `@Controller`를 겸한다는 점(일방통행 포함관계) 때문에 **스코프(1단계)로 컨트롤러 타입을 가르고, 겹치는 REST 구간만 우선순위(@Order, 2단계)로** 정리하면 각자 자기 예외를 안전하게 처리한다. 예외를 던진 **컨트롤러 타입**이 view냐 json이냐를 결정한다.
