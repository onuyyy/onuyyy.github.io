---
title: "Spring 파라미터 바인딩 정리"
date: 2026-06-10 09:00:00 +0900
categories: [Spring]
tags: [Spring, MVC, Binding, DTO]
---

> 핵심 질문: **요청 파라미터가 컨트롤러 메서드로 어떻게 들어오고, DTO로 받을 때 getter/setter/기본 생성자가 왜 필요한가?**

---

## 0. 핵심 한 줄

> **개별 파라미터로 받으면 Spring이 값 하나를 바로 변환해서 넣고, DTO로 받으면 Spring이 DTO 객체를 만든 뒤 필드에 값을 채운다.**

그래서 일반적인 DTO 바인딩에서는 보통 아래 조합이 필요하다.

```java
@Getter
@Setter
@NoArgsConstructor
public class SearchDto {
    private String name;
    private Integer age;
}
```

- 기본 생성자: Spring이 DTO 객체를 만들 때 필요
- setter: 요청 파라미터 값을 DTO 필드에 넣을 때 필요
- getter: 바인딩 자체에는 필수는 아니지만, 값을 꺼내 쓰거나 응답 JSON으로 내보낼 때 보통 필요

---

## 1. Spring과 Spring Boot 차이

파라미터 바인딩의 핵심 원리는 **Spring MVC**가 처리한다.

- Spring Framework: MVC의 실제 바인딩 기능 제공
- Spring Boot: Spring MVC, Jackson, 메시지 컨버터 등을 자동 설정해줌

즉, Boot라서 완전히 다른 방식으로 DTO에 값을 넣는 것이 아니다. Boot는 설정을 편하게 해줄 뿐이고, 컨트롤러 파라미터 바인딩의 기본 원리는 Spring MVC 쪽이다.

---

## 2. 개별 파라미터로 받을 때

요청:

```http
GET /users?name=kim&age=20
```

컨트롤러:

```java
@GetMapping("/users")
public String users(@RequestParam String name,
                    @RequestParam Integer age) {
    return "users";
}
```

이 경우 DTO가 없다. Spring은 요청 파라미터 `"kim"`, `"20"`을 각각 `String`, `Integer`로 변환해서 메서드 인자에 바로 넣는다.

```java
name = "kim";
age = 20;
```

따라서 getter/setter/기본 생성자 같은 DTO 규칙과 상관없다.

---

## 3. 쿼리 파라미터를 DTO로 받을 때

요청:

```http
GET /users?name=kim&age=20
```

컨트롤러:

```java
@GetMapping("/users")
public String users(SearchDto dto) {
    return "users";
}
```

위 코드는 보통 아래와 같은 의미로 동작한다.

```java
@GetMapping("/users")
public String users(@ModelAttribute SearchDto dto) {
    return "users";
}
```

Spring은 대략 이런 식으로 처리한다.

```java
SearchDto dto = new SearchDto();
dto.setName("kim");
dto.setAge(20);
```

그래서 일반적인 JavaBean 방식 DTO라면 다음이 필요하다.

| 필요 요소 | 이유 |
|---|---|
| 기본 생성자 | `new SearchDto()`처럼 객체를 먼저 만들기 위해 |
| setter | `setName(...)`, `setAge(...)`처럼 값을 넣기 위해 |
| getter | 바인딩 자체보다는 이후 값을 읽거나 응답으로 내보내기 위해 |

DTO:

```java
@Getter
@Setter
@NoArgsConstructor
public class SearchDto {
    private String name;
    private Integer age;
}
```

정리하면:

```java
@RequestParam String name
@RequestParam Integer age
```

이렇게 개별로 받으면 DTO가 필요 없고,

```java
SearchDto dto
```

또는

```java
@ModelAttribute SearchDto dto
```

이렇게 DTO로 받으면 Spring이 DTO를 생성하고 값을 채운다.

---

## 4. `@RequestParam DTO`가 아니라 `@ModelAttribute DTO`

헷갈리기 쉬운 부분:

```java
@GetMapping("/users")
public String users(@RequestParam SearchDto dto) {
    return "users";
}
```

이런 식으로 DTO에 `@RequestParam`을 붙이는 것은 일반적인 쿼리 파라미터 DTO 바인딩 방식이 아니다.

`@RequestParam`은 보통 값 하나를 받을 때 쓴다.

```java
@RequestParam String name
@RequestParam Integer age
```

여러 쿼리 파라미터를 DTO 필드에 묶어서 받고 싶으면 `@ModelAttribute` 방식이다.

```java
@ModelAttribute SearchDto dto
```

다만 Spring MVC에서는 복합 객체 타입이면 `@ModelAttribute`를 생략할 수 있어서 아래처럼 많이 쓴다.

```java
public String users(SearchDto dto)
```

---

## 5. 폼 파라미터도 같은 방식

HTML form 전송도 쿼리 파라미터와 비슷하게 `@ModelAttribute`로 DTO에 들어갈 수 있다.

```html
<form method="post" action="/users">
  <input name="name">
  <input name="age">
  <button type="submit">저장</button>
</form>
```

```java
@PostMapping("/users")
public String save(UserForm form) {
    return "redirect:/users";
}
```

이것도 대략:

```java
UserForm form = new UserForm();
form.setName(request.getParameter("name"));
form.setAge(...);
```

처럼 이해하면 된다.

---

## 6. JSON 요청 바디를 DTO로 받을 때

요청:

```http
POST /users
Content-Type: application/json

{
  "name": "kim",
  "age": 20
}
```

컨트롤러:

```java
@PostMapping("/users")
public void save(@RequestBody UserCreateRequest request) {
}
```

이건 `@ModelAttribute`가 아니라 `@RequestBody`다. 쿼리 파라미터나 form 파라미터가 아니라 **HTTP body의 JSON**을 읽는다.

이때는 Spring MVC가 직접 setter로 넣는다기보다, 내부적으로 Jackson 같은 JSON 라이브러리와 `HttpMessageConverter`가 객체를 만든다.

가장 무난한 DTO:

```java
@Getter
@Setter
@NoArgsConstructor
public class UserCreateRequest {
    private String name;
    private Integer age;
}
```

Jackson도 기본적으로 아래와 유사하게 처리할 수 있다.

```java
UserCreateRequest request = new UserCreateRequest();
request.setName("kim");
request.setAge(20);
```

---

## 7. 바인딩 종류별 정리

| 요청 값 위치 | 컨트롤러 예시 | 처리 방식 | DTO에 보통 필요한 것 |
|---|---|---|---|
| 쿼리 파라미터 | `@RequestParam String name` | 값 하나를 바로 주입 | DTO 아님 |
| 쿼리 파라미터 | `SearchDto dto` | `@ModelAttribute` 생략 | 기본 생성자 + setter |
| 쿼리 파라미터 | `@ModelAttribute SearchDto dto` | DTO 생성 후 필드 주입 | 기본 생성자 + setter |
| form 파라미터 | `UserForm form` | `@ModelAttribute` 생략 | 기본 생성자 + setter |
| JSON body | `@RequestBody UserRequest dto` | Jackson이 body를 객체로 변환 | 기본 생성자 + setter가 가장 무난 |
| URL 경로 | `@PathVariable Long id` | 경로 일부를 값 하나로 변환 | DTO 아님 |
| Header | `@RequestHeader String token` | 헤더 값을 바로 주입 | DTO 아님 |
| Cookie | `@CookieValue String sessionId` | 쿠키 값을 바로 주입 | DTO 아님 |

---

## 8. 자동 주입이 안 되거나 주의할 예외

### 8.1 DTO에 기본 생성자가 없고 setter만 있는 경우

```java
@Getter
@Setter
@AllArgsConstructor
public class SearchDto {
    private String name;
    private Integer age;
}
```

이렇게 `@AllArgsConstructor`만 있고 `@NoArgsConstructor`가 없으면, 일반적인 `@ModelAttribute` 바인딩에서 실패할 수 있다.

Spring이 먼저 객체를 만들어야 하는데:

```java
new SearchDto()
```

를 호출할 수 없기 때문이다.

### 8.2 기본 생성자는 있는데 setter가 없는 경우

```java
@Getter
@NoArgsConstructor
public class SearchDto {
    private String name;
    private Integer age;
}
```

객체 생성은 가능하지만 값을 넣을 setter가 없다. 일반적인 JavaBean 방식 바인딩에서는 값이 안 들어가거나 바인딩이 실패할 수 있다.

### 8.3 파라미터 이름과 필드명이 다를 때

요청:

```http
GET /users?user_name=kim
```

DTO:

```java
private String name;
```

요청 파라미터 이름은 `user_name`인데 DTO 필드는 `name`이면 자동으로 못 맞춘다.

자동 바인딩은 기본적으로 이름이 맞아야 한다.

```http
GET /users?name=kim
```

```java
private String name;
```

이렇게 맞아야 한다.

### 8.4 타입 변환이 안 될 때

요청:

```http
GET /users?age=abc
```

DTO:

```java
private Integer age;
```

`"abc"`는 `Integer`로 변환할 수 없으므로 바인딩 에러가 난다.

컨트롤러에서 `BindingResult`를 같이 받으면 에러를 직접 처리할 수 있다.

```java
@GetMapping("/users")
public String users(@ModelAttribute SearchDto dto, BindingResult bindingResult) {
    if (bindingResult.hasErrors()) {
        return "users/form";
    }
    return "users/list";
}
```

주의: `BindingResult`는 검증/바인딩 대상 바로 뒤에 와야 한다.

```java
public String users(@ModelAttribute SearchDto dto, BindingResult bindingResult)
```

### 8.5 중첩 객체는 이름 규칙이 필요

DTO:

```java
public class UserSearchDto {
    private TeamDto team;
}
```

```java
public class TeamDto {
    private String name;
}
```

요청:

```http
GET /users?team.name=dev
```

이런 식으로 `team.name` 형태의 파라미터 이름이 필요하다.

### 8.6 컬렉션은 인덱스나 반복 파라미터 규칙이 필요

예:

```http
GET /users?ids=1&ids=2&ids=3
```

```java
@RequestParam List<Long> ids
```

DTO 안에 리스트로 받을 수도 있지만, 복잡한 객체 리스트는 이름 규칙이 더 중요하다.

```http
items[0].name=a&items[1].name=b
```

처럼 인덱스를 써야 하는 경우가 많다.

### 8.7 `@RequestBody`와 `@ModelAttribute`를 헷갈릴 때

JSON body:

```json
{
  "name": "kim"
}
```

이걸 받으려면:

```java
@RequestBody UserRequest request
```

쿼리 파라미터나 form:

```http
GET /users?name=kim
```

이걸 DTO로 받으려면:

```java
@ModelAttribute UserSearchDto dto
```

둘은 데이터가 들어오는 위치도 다르고, 처리하는 내부 방식도 다르다.

### 8.8 `@RequestParam`은 기본적으로 필수값

```java
@RequestParam String name
```

이렇게 쓰면 기본적으로 `name` 파라미터가 필요하다. 없으면 에러가 날 수 있다.

선택값이면:

```java
@RequestParam(required = false) String name
```

또는 기본값:

```java
@RequestParam(defaultValue = "1") int page
```

처럼 쓴다.

반면 DTO 바인딩에서는 해당 필드 파라미터가 없으면 보통 `null` 또는 기본값으로 남는다.

### 8.9 primitive 타입은 null을 못 담는다

```java
private int age;
```

요청에 `age`가 없으면 `int`는 `null`이 될 수 없다. 기본값 `0`이 들어가거나 상황에 따라 의도와 다르게 동작할 수 있다.

검색 조건, 선택 입력값이면 보통 wrapper 타입이 안전하다.

```java
private Integer age;
```

### 8.10 `record`는 setter 방식이 아니라 생성자 방식

요즘 DTO를 `record`로 많이 만든다.

```java
public record SearchDto(String name, Integer age) {
}
```

`record`는 불변 객체다.

- 필드는 사실상 `private final`
- setter가 없다
- 기본 생성자도 없다
- 대신 전체 필드를 받는 생성자가 있다

그래서 일반 class DTO처럼 아래 방식으로 바인딩할 수 없다.

```java
SearchDto dto = new SearchDto();
dto.setName("kim");
dto.setAge(20);
```

`record`는 setter가 없으므로 Spring/Jackson이 값을 넣으려면 생성자를 호출해야 한다.

```java
SearchDto dto = new SearchDto("kim", 20);
```

즉 DTO 바인딩 방식은 크게 두 종류로 나뉜다.

| DTO 형태 | 바인딩 방식 |
|---|---|
| 일반 class DTO | 기본 생성자 호출 후 setter로 값 주입 |
| record DTO | 생성자 파라미터에 값을 맞춰 생성 |

Spring 6 / Spring Boot 3 계열에서는 조건이 맞으면 `@ModelAttribute`에서도 `record` 생성자 바인딩이 가능하다.

```java
@GetMapping("/users")
public String users(@ModelAttribute SearchDto dto) {
    return "users";
}
```

요청 파라미터 이름과 record 컴포넌트 이름이 맞아야 한다.

```http
GET /users?name=kim&age=20
```

```java
public record SearchDto(String name, Integer age) {
}
```

다만 레거시 Spring MVC나 Spring 5 이하에서는 `@ModelAttribute record` 바인딩이 기대처럼 안 될 수 있다. `record` 자체가 Java 16부터 정식이고, Spring의 record 지원도 최신 버전 쪽이 좋다.

`@RequestBody` JSON에서는 record DTO가 더 흔하게 잘 쓰인다.

```java
public record UserCreateRequest(String name, Integer age) {
}
```

```java
@PostMapping("/users")
public void save(@RequestBody UserCreateRequest request) {
}
```

JSON:

```json
{
  "name": "kim",
  "age": 20
}
```

Jackson이 생성자를 호출해서 만든다.

```java
new UserCreateRequest("kim", 20);
```

record가 아닌 일반 class도 생성자 바인딩이 가능할 수 있다.

```java
public class SearchDto {
    private final String name;
    private final Integer age;

    public SearchDto(String name, Integer age) {
        this.name = name;
        this.age = age;
    }

    public String getName() {
        return name;
    }

    public Integer getAge() {
        return age;
    }
}
```

이런 방식도 setter 없이 될 수 있다. 다만 생성자 파라미터 이름 인식, Java 버전, Spring 버전, 컴파일 옵션 등에 영향을 받을 수 있다.

입문 단계나 레거시 Spring MVC 프로젝트에서는 일단 아래 방식이 제일 덜 헷갈린다.

```java
@Getter
@Setter
@NoArgsConstructor
public class SearchDto {
    private String name;
    private Integer age;
}
```

---

## 9. 자주 쓰는 Lombok 조합

### 요청 DTO, 검색 DTO

```java
@Getter
@Setter
@NoArgsConstructor
public class UserSearchDto {
    private String keyword;
    private Integer page;
    private Integer size;
}
```

### JSON 요청 DTO도 무난하게

```java
@Getter
@Setter
@NoArgsConstructor
public class UserCreateRequest {
    private String name;
    private String email;
}
```

### 응답 DTO

응답으로만 내보내는 DTO는 setter가 꼭 필요하지 않을 수 있다.

```java
@Getter
@AllArgsConstructor
public class UserResponse {
    private Long id;
    private String name;
}
```

응답은 값을 "받는" 게 아니라 JSON으로 "내보내는" 쪽이라 getter가 중요하다.

---

## 10. 최종 정리

이번 대화에서 정리한 핵심:

> **쿼리 파라미터를 개별 파라미터로 받으면 그냥 받으면 된다. DTO가 아니므로 getter/setter/기본 생성자와 상관없다.**

```java
public String users(@RequestParam String name,
                    @RequestParam Integer age)
```

> **쿼리 파라미터를 DTO로 받으면 Spring이 DTO 객체를 만들고 setter로 값을 넣는 방식이 기본이다. 그래서 보통 기본 생성자와 setter가 필요하다.**

```java
public String users(@ModelAttribute SearchDto dto)
```

또는 생략형:

```java
public String users(SearchDto dto)
```

가장 안전한 DTO 형태:

```java
@Getter
@Setter
@NoArgsConstructor
public class SearchDto {
    private String name;
    private Integer age;
}
```

getter는 주입 자체보다는 DTO 값을 읽거나 응답으로 내려줄 때 필요하다고 보면 된다.
