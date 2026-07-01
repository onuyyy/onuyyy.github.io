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

`@RestController` 정의를 까보면:

```java
@Controller        // ← @RestController 안에 @Controller가 들어있다!
@ResponseBody
public @interface RestController { }
```

그리고 `@ControllerAdvice(annotations = ...)`는 **메타 어노테이션까지 뒤져서** 매칭한다. 결과:

| 스코프 | 실제 매칭 |
|---|---|
| `annotations = RestController.class` | `@RestController`만 ✅ |
| `annotations = Controller.class` | `@Controller` **+ `@RestController`도** ❌ |

즉 뷰 어드바이스를 `annotations = Controller.class`로 좁히면 **REST 컨트롤러까지 딸려 잡혀서** 다시 충돌한다.

### 5-3. 안전한 구조 — @Order로 우선순위

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
  - `@RestController`는 `@Controller`이기도 해서 `annotations = Controller.class`는 REST까지 잡는 **함정**.
  - **REST 어드바이스에 `@Order` 우선권 + JSP는 전역 fallback**, 또는 **패키지로 분리**.

### 한 줄 결론

> REST와 뷰는 **에러 응답 포맷이 다르다(JSON vs HTML)**. 어드바이스를 분리하되, `@RestController`가 `@Controller`를 겸한다는 점 때문에 **스코프만으로는 안 되고 우선순위(@Order)까지** 잡아줘야 각자 자기 예외를 안전하게 처리한다.
