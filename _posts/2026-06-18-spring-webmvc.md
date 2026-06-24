---
title: "Spring WebMVC 정리"
date: 2026-06-18 13:00:00 +0900
categories: [Spring]
tags: [Spring, MVC]
---

Spring MVC는 자바 서블릿(Servlet) 위에서 동작하는 웹 프레임워크다. 서블릿을 직접 다룰 때의 번거로움을 추상화하고, 어노테이션 기반으로 요청을 처리하게 해 준다.

## 1. Front Controller 패턴 (= Model2 구조)

**모든 요청이 하나의 진입점을 반드시 거치게 하는 구조.**

- 그 진입점이 Spring MVC의 `DispatcherServlet`이다.
- 클라이언트의 모든 HTTP 요청은 먼저 `DispatcherServlet`으로 들어오고, 여기서 적절한 컨트롤러로 분배(dispatch)된다.
- 공통 처리(인증, 로깅, 인코딩, 예외 처리 등)를 진입점 한 곳에 모을 수 있다는 게 핵심 장점. → **파사드(Facade) 패턴**의 성격 (복잡한 내부를 단일 창구로 감춤).

### Model1 vs Model2

| 구분 | 처리 방식 |
|------|-----------|
| Model1 | JSP가 요청 처리 + 화면 출력을 다 담당 (로직과 뷰가 섞임) |
| **Model2** | 요청은 Controller, 데이터는 Model, 화면은 View로 **역할 분리** (MVC) |

Spring MVC는 Front Controller(`DispatcherServlet`)를 둔 **Model2 구조**다.

### 요청 처리 흐름 (대략)
```
요청 → DispatcherServlet → HandlerMapping(어느 컨트롤러?)
     → Controller 실행 → Model에 데이터 담고 View 이름 반환
     → ViewResolver(어느 화면?) → View 렌더링 → 응답
```

## 2. HttpServletRequest / HttpServletResponse 추상화

순수 서블릿에서는 요청·응답을 `HttpServletRequest`, `HttpServletResponse` 객체로 직접 다뤄야 한다.

- 파라미터 꺼내기: `request.getParameter("name")` — 문자열만, 형변환 직접
- 응답 쓰기: `response.getWriter().write(...)` — 번거로움

Spring MVC는 이 과정을 **추상화**해서, 개발자가 서블릿 API를 직접 만지지 않아도 되게 한다.
```java
// 순수 서블릿
String name = request.getParameter("name");
int age = Integer.parseInt(request.getParameter("age"));

// Spring MVC — 메서드 파라미터로 자동 수집 + 형변환
@GetMapping("/user")
public String user(@RequestParam String name, @RequestParam int age) { ... }
```
필요하면 여전히 `HttpServletRequest`를 파라미터로 받을 수도 있다(선택).

## 3. 파라미터 수집 & 리턴 타입 — 자바 빈(JavaBean) 기반

### 파라미터 자동 수집(바인딩)
요청 파라미터를 **자바 빈 객체에 자동으로 채워준다**. 파라미터 이름 ↔ 객체의 setter/필드명을 맞춰 매핑한다.
```java
// 요청: /join?name=kim&age=20
public class UserDTO {     // 자바 빈 (기본 생성자 + getter/setter)
    private String name;
    private int age;
}

@PostMapping("/join")
public String join(UserDTO dto) { ... }   // name, age가 dto에 자동 주입
```
- 자바 빈 규약: 기본 생성자, getter/setter, 필드. (Lombok의 `@Data` 등으로 간단히)
- 문자열 → 숫자/날짜 등 **형변환도 자동** 처리.

#### ⚠️ "자바 빈"과 "스프링 빈"은 다른 것
이름이 비슷해 헷갈리지만 별개다. **DTO는 스프링 빈으로 등록할 필요 없다** (`@Component`/`<bean>` 불필요).

| 구분 | 스프링 빈 (Spring Bean) | 자바 빈 (JavaBean) |
|------|----------------------|-------------------|
| 정체 | 스프링 컨테이너가 관리하는 객체 | 자바 객체 **작성 규약(관례)** |
| 등록 | `@Component`/`@Service`/`<bean>` 등 | 등록 개념 없음, 규약대로 만들면 끝 |
| 생성 | 보통 싱글톤 1개 | 요청마다 `new`로 새로 생성 |
| 예 | Controller, Service, DAO | **DTO, VO** |

DTO에서 말하는 "자바 빈"은 후자(JavaBean 규약)다.

#### 자바 빈(JavaBean) 규약 — 자바에 정해진 관례
1. **기본 생성자** (파라미터 없는 `public` 생성자)
2. **private 필드 + public getter/setter** — 명명규칙: 필드 `name` → `getName()`/`setName()` (boolean은 `isXxx()`)
3. `Serializable` 구현 (권장, 실무에선 자주 생략)

> 강제 문법은 아니지만 다들 따르는 약속. "이런 모양이면 프레임워크가 알아서 다뤄준다"는 공통 규칙으로, 스프링 MVC 파라미터 수집·JSP `<jsp:useBean>`·Jackson JSON 변환 등이 모두 이 규약에 기댄다.

#### 왜 기본 생성자 + setter가 필요한가 (동작 원리)
스프링 MVC가 DTO를 채울 때 내부적으로 이렇게 한다:
```java
UserDTO dto = new UserDTO();                 // ① 기본 생성자로 객체 생성
dto.setName(request.getParameter("name"));   // ② setter로 값 주입
dto.setAge(Integer.parseInt(...));           // ③ 형변환도 자동
```
- 기본 생성자 없으면 → ① `new` 불가 → 바인딩 실패
- setter 없으면 → ② 값 주입 불가 → 필드가 빈 채(null/0)
- **getter/setter 이름이 필드명과 규칙대로 맞아야** 매칭됨 (필드 `age` → 반드시 `setAge`, 오타 시 그 필드는 안 채워짐)

파라미터 자동 수집에 실무상 필요한 건 **기본 생성자 + getter/setter** 둘이면 충분(`Serializable`은 보통 생략). Lombok이면:
```java
@Data @NoArgsConstructor   // getter/setter + 기본 생성자 자동 생성
public class UserDTO { private String name; private int age; }
```

### 리턴 타입의 유연함
컨트롤러 메서드가 반환하는 타입에 따라 동작이 달라진다.

| 리턴 타입 | 의미 |
|-----------|------|
| `String` | 뷰(View) 이름 → ViewResolver가 화면 찾음 |
| 객체/DTO + `@ResponseBody` | 객체를 JSON 등으로 변환해 응답 본문에 |
| `ModelAndView` | 모델 데이터 + 뷰를 함께 |
| `void` | 요청 경로 기반으로 뷰 결정 |

## 4. 어노테이션 기반

XML로 일일이 매핑하던 것을 **어노테이션으로 선언**한다. (설정 < 관례)

| 어노테이션 | 역할 |
|-----------|------|
| `@Controller` | 이 클래스가 웹 컨트롤러임을 표시 |
| `@RequestMapping` | URL ↔ 메서드 매핑 (공통 경로) |
| `@GetMapping` / `@PostMapping` | HTTP 메서드별 매핑 (GET/POST…) |
| `@RequestParam` | 요청 파라미터를 메서드 인자로 |
| `@PathVariable` | URL 경로의 일부를 인자로 (`/user/{id}`) |
| `@RequestBody` | 요청 본문(JSON 등)을 객체로 |
| `@ResponseBody` | 반환 객체를 응답 본문(JSON 등)으로 |
| `@RestController` | `@Controller` + `@ResponseBody` (REST API용) |

```java
@Controller
@RequestMapping("/users")
public class UserController {

    @GetMapping("/{id}")
    public String detail(@PathVariable Long id, Model model) {
        model.addAttribute("user", service.find(id));
        return "user/detail";   // 뷰 이름
    }
}
```

## 5. 컨트롤러 & 뷰 매핑 동작 원리

### @Controller 와 컴포넌트 스캔
`@Controller`는 "이 클래스가 웹 컨트롤러"라는 표시일 뿐, 그것만으로 등록되지는 않는다.
- **스프링 부트**: `@Controller`가 붙으면 자동 스캔되어 빈 등록.
- **레거시 스프링**: `<context:component-scan base-package="...">`로 **해당 패키지를 스캔해야** 인식된다.

→ 스캔 범위 밖에 컨트롤러를 두면 빈 등록 자체가 안 되고, 매핑도 안 생겨 404가 난다.

### @RequestMapping + @GetMapping — 경로 조합
클래스 레벨 `@RequestMapping`과 메서드 레벨 `@GetMapping`은 **합쳐져서** 최종 URL이 된다.
```java
@RequestMapping("/sample")   // 클래스 공통 경로
@GetMapping("/basic")        // 메서드 경로  →  최종: GET /sample/basic
```
- `@RequestMapping`: HTTP 메서드 구분 없이 공통 경로(주로 클래스에).
- `@GetMapping`/`@PostMapping`: 특정 HTTP 메서드에만 매핑(주로 메서드에).
- 경로가 어긋나면(오타 등) HandlerMapping이 못 찾아 404.

### 리턴 타입 void — 뷰 이름을 경로에서 가져온다
컨트롤러가 **아무 값도 반환하지 않으면(void)**, 스프링은 **요청 경로를 그대로 뷰 이름으로 사용**한다.
```
요청 /sample/basic  →  (void)  →  뷰 이름 "sample/basic"
```
- 그래서 void인 경우, "URL 경로 = 뷰 이름"이 되도록 경로를 잘 지정해 줘야 한다.
- `String`을 반환하면 그 문자열이 뷰 이름이 된다 (`return "sample/basic";` → void와 동일 결과, 단 명시적).
- `@ResponseBody`가 붙으면 반환값은 뷰 이름이 아니라 **응답 본문 그 자체**가 된다(뷰 안 거침).

### ViewResolver — 뷰 이름을 실제 파일로 변환
컨트롤러는 "뷰 이름"만 정하고, 그 이름을 실제 파일 경로로 바꾸는 건 ViewResolver의 몫이다.
```xml
<bean class="org.springframework.web.servlet.view.InternalResourceViewResolver">
  <property name="prefix" value="/WEB-INF/views/"/>   <!-- 뷰 이름 앞에 붙임 -->
  <property name="suffix" value=".jsp"/>              <!-- 뷰 이름 뒤에 붙임 -->
</bean>
```
변환 공식: **prefix + 뷰이름 + suffix**
```
/WEB-INF/views/ + sample/basic + .jsp  →  /WEB-INF/views/sample/basic.jsp
```
- 이 경로에 JSP가 있으면 렌더링, 없으면 뷰 못 찾음 에러.
- JSP를 `WEB-INF/` 아래 두는 이유: URL로 **직접 접근 못 하게** 막아 반드시 컨트롤러를 거치게 하기 위함.

### 전체 매핑 흐름
```
요청 → DispatcherServlet → HandlerMapping(@RequestMapping으로 메서드 탐색)
     → 컨트롤러 메서드 실행 → 뷰 이름 결정(void면 경로 기반)
     → ViewResolver(prefix+suffix로 파일 경로 완성) → JSP 렌더링 → 응답
```

> 부트에서는 이 ViewResolver를 XML로 만들지 않고 `application.properties`에
> `spring.mvc.view.prefix=/WEB-INF/views/`, `spring.mvc.view.suffix=.jsp` 두 줄로 대체한다.

---

## 요약

| 특징 | 핵심 |
|------|------|
| Front Controller (Model2) | 모든 요청이 `DispatcherServlet` 단일 진입점을 거침 (파사드) |
| 서블릿 추상화 | `HttpServletRequest/Response`를 직접 안 다뤄도 됨 |
| 파라미터/리턴 | 자바 빈 기반 자동 수집·형변환, 리턴 타입에 따라 뷰/JSON 등 결정 |
| 어노테이션 기반 | `@Controller`, `@GetMapping` 등으로 선언적 매핑 |