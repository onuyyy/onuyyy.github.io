---
title: "Spring Controller 정리"
date: 2026-06-18 09:00:00 +0900
categories: [Spring]
tags: [Spring, MVC, Controller]
---

`@Controller`가 붙은 클래스가 요청을 처리한다. `DispatcherServlet`이 요청을 받아
`HandlerMapping`으로 어느 컨트롤러 메서드인지 찾고, 실행 결과(리턴값)를 보고 다음 동작을 결정한다.

예를 들어 레거시 Spring에서는 `servlet-context.xml`에 `<context:component-scan base-package="com.example.demo.controller"/>`
가 있어야 `@Controller`가 빈으로 등록된다.
```
spring-boot → 컴포넌트 자동 등록
spring(레거시) → 해당 패키지를 직접 스캔해줘야 인식됨
```

## 1. Controller return type (리턴 타입)

리턴 타입에 따라 `DispatcherServlet`이 결과를 다르게 해석한다.

| 리턴 타입 | 의미 | 비고 |
|-----------|------|------|
| `void` | **요청 경로**를 그대로 뷰 이름으로 사용 | `@GetMapping("/sample/basic")` → 뷰 `sample/basic` |
| `String` | 반환 문자열이 **뷰 이름** | `ViewResolver`가 실제 파일로 변환 |
| `VO/DTO` (자바 객체) | 객체 자체가 응답 데이터 | 보통 `@ResponseBody`/`@RestController`와 함께 JSON으로 변환 (HttpMessageConverter) |
| `ResponseEntity<>` | HTTP **header + body + status code**를 다 담음 | REST API에서 자주 씀. 상태코드를 직접 제어 |
| `Model` / `ModelAndView` / `HttpHeaders` | 뷰 이름 + 데이터를 함께 | 옛날 레거시 스타일. 요즘은 Model을 파라미터로 받고 String을 리턴하는 방식 선호 |

### void 리턴 — 경로 = 뷰 이름
리턴 타입이 `void`면 따로 뷰 이름을 안 주므로, **요청 경로가 곧 뷰 이름**이 된다.
ViewResolver 설정(`servlet-context.xml`)이 prefix/suffix를 붙여 실제 파일을 찾는다.
```xml
<bean class="org.springframework.web.servlet.view.InternalResourceViewResolver">
  <property name="prefix" value="/WEB-INF/views/"/>
  <property name="suffix" value=".jsp"/>
</bean>
```
```java
@GetMapping("/basic")          // @RequestMapping("/sample") + "/basic"
public void basic() { ... }
// 뷰 이름 "sample/basic"
// → /WEB-INF/views/sample/basic.jsp
```

> 즉 매핑 경로와 JSP 위치를 맞춰놓으면 리턴값을 생략할 수 있다.
> 위치가 안 맞으면 `String`으로 뷰 이름을 명시해야 한다.

## 2. Model — 뷰로 데이터 전달

데이터를 뷰로 넘기고 싶을 때 메서드 파라미터에 `Model model`을 추가하고
`model.addAttribute("이름", 값)`으로 담는다. 담긴 데이터는 JSP에서 EL `${이름}`로 꺼낸다.

```java
@GetMapping("/ex")
public void ex(
    @ModelAttribute("dto") SampleDto dto,   // 모델에 "dto"로 자동 담김
    @ModelAttribute("page") int page,        // "page"로 전달
    Model model) {
  model.addAttribute("dto", dto);
  model.addAttribute("list", new String[]{"AAA", "BBB", "CCC"});
}
```
```jsp
<%-- /WEB-INF/views/sample/ex.jsp --%>
<h1>${dto}</h1>     <%-- ⚠️ {dto} 가 아니라 ${dto} 여야 EL이 동작 --%>
<hr>${list}
```

### @ModelAttribute
- 파라미터에 붙이면: 요청 파라미터를 객체에 바인딩 + **자동으로 모델에 담겨** 뷰로 전달된다.
- 이름을 안 주면 타입명(첫 글자 소문자)이 모델 키가 된다. `@ModelAttribute("dto")`처럼 키를 지정 가능.
- 메서드에 붙이면: 그 컨트롤러의 모든 핸들러 실행 전에 공통으로 모델에 값을 채워준다.

## 3. @ControllerAdvice — 공통 예외 처리

> 컨트롤러에 문제(예외)가 생겼을 때 **대신 처리**하는 역할을 따로 빼놓은 것.

여러 컨트롤러에 공통으로 적용되는 **전역(global) 예외 처리기 / 공통 모델 설정**을 모아두는 곳.
컨트롤러마다 try-catch를 반복하지 않고 한 곳에서 처리할 수 있다.

```java
@ControllerAdvice               // (JSON 응답이면 @RestControllerAdvice)
public class GlobalExceptionAdvice {

  @ExceptionHandler(IllegalArgumentException.class)
  public String handleIllegalArg(IllegalArgumentException e, Model model) {
    model.addAttribute("msg", e.getMessage());
    return "error/badRequest";   // 에러 뷰
  }
}
```

### @ExceptionHandler
- **특정 예외 타입을 잡아서 처리**하는 메서드에 붙이는 어노테이션.
- 같은 컨트롤러 안에 두면 그 컨트롤러 한정, `@ControllerAdvice` 안에 두면 전역 적용.
- 더 구체적인 예외 타입을 먼저 매칭한다(부모 타입보다 자식 타입 우선).

| 어노테이션 | 범위 | 역할 |
|------------|------|------|
| `@ExceptionHandler` | 메서드 | 특정 예외 잡아서 처리 |
| `@ControllerAdvice` | 클래스 | 전역 예외 처리/공통 모델을 한곳에 모음 |

## 정리
- 리턴 타입(`void`/`String`/객체/`ResponseEntity`)에 따라 DispatcherServlet의 후속 처리가 달라진다.
- 화면 응답은 보통 뷰 이름 + `Model`, REST 응답은 객체/`ResponseEntity`로 JSON.
- 예외는 컨트롤러마다 처리하지 말고 `@ControllerAdvice` + `@ExceptionHandler`로 모은다.
- JSP에서는 `Model`에 담은 값을 EL `${...}`로 사용한다.