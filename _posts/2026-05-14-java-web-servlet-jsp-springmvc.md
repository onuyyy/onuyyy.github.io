---
title: "Java Web · Servlet · JSP · Spring MVC · Tomcat 정리"
date: 2026-05-14 09:00:00 +0900
categories: [Spring]
tags: [Java, Servlet, Spring, Tomcat, JSP]
---

## Java 웹 애플리케이션의 기본 구조

Java 웹 애플리케이션은 기본적으로 **HTTP 요청을 Java 코드로 처리하고 HTTP 응답을 돌려주는 구조**다.

```text
Client
  -> HTTP 요청
  -> Tomcat
  -> Servlet
  -> Spring MVC Controller
  -> HTML 또는 JSON 응답
```

Java 자체는 범용 언어지만, 웹 요청/응답을 처리하기 위해 **Servlet API**라는 표준이 만들어졌다.

Servlet API는 다음을 정의한다.

```text
- HTTP 요청을 어떤 객체로 받을지
- HTTP 응답을 어떤 객체로 만들지
- 요청이 오면 어떤 메서드를 호출할지
- Servlet 객체의 생명주기를 누가 관리할지
```

그래서 Java 웹에서는 `HttpServletRequest`, `HttpServletResponse`, `HttpServlet`, `Filter`, `Listener` 같은 개념이 등장한다.

## Servlet이란?

Servlet은 **HTTP 요청과 응답을 처리하기 위한 Java 웹 컴포넌트**다.

정확히는:

```text
Servlet = Servlet 규약을 따르는 Java 클래스
```

보통은 `HttpServlet`을 상속해서 만든다.

```java
public class HelloServlet extends HttpServlet {

  @Override
  protected void doGet(HttpServletRequest request,
                       HttpServletResponse response) throws IOException {
    response.getWriter().write("hello");
  }
}
```

서블릿은 자바 클래스가 맞다.  
다만 일반 자바 클래스와 다른 점은 **내가 직접 생성하고 실행하지 않는다**는 것이다.

일반 자바 객체는 보통 이렇게 쓴다.

```java
HelloService service = new HelloService();
service.run();
```

하지만 Servlet은 Tomcat 같은 Servlet Container가 생성하고 실행한다.

```text
Tomcat
  -> Servlet 객체 생성
  -> init() 호출
  -> 요청마다 service() 호출
  -> doGet(), doPost() 실행
  -> 종료 시 destroy() 호출
```

즉 Servlet은 다음처럼 이해하면 된다.

```text
Servlet = Tomcat이 생성/실행/관리하는 HTTP 요청 처리용 Java 클래스
```

## Tomcat이란?

Tomcat은 Java 웹 애플리케이션을 실행하기 위한 서버다.

정확히는:

```text
Tomcat = HTTP 서버 + Servlet Container
```

Tomcat이 하는 일은 다음과 같다.

```text
- 서버 포트 열기
- TCP 연결 받기
- HTTP 요청 파싱
- WAR 배포
- web.xml 읽기
- Servlet 생성 및 실행
- Filter/Listener 실행
- JSP 실행
- Session 관리
- 요청 처리 스레드 관리
- HTTP 응답 전송
```

브라우저가 요청을 보내면 Tomcat은 HTTP 메시지를 읽고, Java 코드가 사용할 수 있도록 객체로 바꾼다.

```java
HttpServletRequest
HttpServletResponse
```

그리고 URL에 매핑된 Servlet을 찾아 실행한다.

```text
브라우저
  -> HTTP 요청
  -> Tomcat
  -> Servlet
  -> HTTP 응답
```

Tomcat은 Java로 만들어졌고 JVM 위에서 실행된다.  
그래서 Tomcat은 사실상 **Java 웹 애플리케이션 전용 서버**라고 볼 수 있다.

Tomcat에 올리는 대표적인 애플리케이션:

```text
- Servlet/JSP 앱
- Spring MVC 앱
- Spring Boot WAR
- 전자정부프레임워크
```

Tomcat이 직접 실행하는 것이 아닌 것:

```text
- Node.js 앱
- Python Django 앱
- PHP 앱
- Ruby on Rails 앱
```

## Servlet, JSP, Spring MVC의 발전 흐름

Java 웹의 흐름은 대략 이렇게 볼 수 있다.

```text
Servlet
  -> JSP
  -> Servlet + JSP MVC
  -> Spring MVC
  -> REST API + Frontend 분리
```

초기에는 Servlet이 직접 HTML을 만들었다.

```java
response.getWriter().println("<html>");
response.getWriter().println("<body>");
response.getWriter().println("<h1>Hello</h1>");
response.getWriter().println("</body>");
response.getWriter().println("</html>");
```

이 방식은 Java 코드 안에 HTML 문자열을 계속 써야 해서 불편했다.

그래서 JSP가 나왔다.

JSP는 HTML 안에 Java 코드를 섞어서 서버에서 HTML을 만들기 위한 기술이다.

```jsp
<html>
<body>
  <h1>Hello <%= request.getAttribute("name") %></h1>
</body>
</html>
```

중요한 점은 JSP도 내부적으로 Servlet으로 변환되어 실행된다는 것이다.

```text
home.jsp
  -> home_jsp.java
  -> home_jsp.class
  -> Servlet처럼 실행
  -> HTML 응답 생성
```

이후에는 역할을 나누기 시작했다.

```text
Servlet
  - 요청 처리
  - Service 호출
  - 결과를 request에 저장
  - JSP로 forward

JSP
  - 화면 출력
```

이게 전통적인 MVC 구조다.

```text
Model
  - 데이터, 비즈니스 결과

View
  - JSP

Controller
  - Servlet
```

Spring MVC는 이 구조를 더 편하게 만든 프레임워크다.

개발자가 Servlet을 직접 여러 개 만들지 않고, Spring이 제공하는 `DispatcherServlet` 하나가 모든 요청을 중앙에서 받는다.

```text
HTTP 요청
  -> Tomcat
  -> DispatcherServlet
  -> Controller
  -> Service
  -> View 또는 JSON 응답
```

Spring MVC에서 개발자는 보통 이렇게 Controller만 작성한다.

```java
@Controller
public class UserController {

  @GetMapping("/users")
  public String users(Model model) {
    model.addAttribute("users", userService.findAll());
    return "users/list";
  }
}
```

하지만 내부적으로는 여전히 Servlet 기반이다.

```text
DispatcherServlet
  -> HttpServlet
  -> Servlet
```

## REST API와 JSON도 왜 Servlet 기반인가?

REST API는 HTML 대신 JSON을 응답하는 방식이다.

```java
@RestController
public class UserApiController {

  @GetMapping("/api/users")
  public List<UserDto> users() {
    return userService.findAll();
  }
}
```

응답:

```json
[
  {"id": 1, "name": "kim"},
  {"id": 2, "name": "lee"}
]
```

여기서 헷갈리기 쉬운 점은 **JSON은 응답 형식이고, Servlet은 요청/응답 처리 기반**이라는 것이다.

```text
Servlet = Java 서버에서 HTTP 요청/응답을 처리하는 실행 구조
JSON = 응답 body에 담기는 데이터 형식
```

Spring MVC 기반 REST API의 실제 흐름은 이렇다.

```text
브라우저 / 프론트엔드 / 앱
  -> HTTP 요청
  -> Tomcat
  -> DispatcherServlet
  -> @RestController
  -> 객체 반환
  -> Jackson이 JSON으로 변환
  -> HttpServletResponse에 JSON body 작성
  -> Tomcat이 HTTP 응답 전송
```

즉 `@RestController`는 Servlet 클래스가 아니다.  
하지만 `@RestController`를 호출하는 입구가 `DispatcherServlet`이고, `DispatcherServlet`은 Servlet이다.

정리하면:

```text
JSON 응답이라서 Servlet이 아닌 것이 아니라,
Servlet 기반 HTTP 처리 흐름 위에서 JSON을 응답하는 것이다.
```

단, Spring WebFlux를 Netty로 실행하면 Servlet 기반이 아닐 수 있다.

```text
spring-boot-starter-web
  -> Spring MVC
  -> 기본 Tomcat
  -> Servlet 기반

spring-boot-starter-webflux
  -> WebFlux
  -> 기본 Reactor Netty
  -> Servlet 기반 아님
```

## WEB-INF와 web.xml

`WEB-INF`는 Spring 자체의 폴더라기보다, Java Servlet 기반 WAR 웹 애플리케이션에서 사용하는 표준 구조다.

```text
src/main/webapp/
  WEB-INF/
    web.xml
    views/
      home.jsp
```

`WEB-INF` 아래 파일은 브라우저에서 직접 접근할 수 없다.

```text
/WEB-INF/views/home.jsp
```

이 경로는 URL로 직접 열 수 없고, Controller나 Servlet이 내부적으로 forward해야만 렌더링된다.

그래서 JSP를 View로 사용할 때 보통 `WEB-INF/views` 아래에 둔다.

`web.xml`은 Spring이 먼저 읽는 파일이 아니라, Tomcat 같은 Servlet Container가 먼저 읽는 파일이다.

역할은 다음과 같다.

```text
- DispatcherServlet 등록
- ContextLoaderListener 등록
- Filter 등록
- Listener 등록
- Spring 설정 XML 위치 지정
- 세션, 에러 페이지 등 웹 애플리케이션 설정
```

전통적인 Spring MVC에서는 `web.xml`에 다음처럼 설정했다.

```xml
<servlet>
  <servlet-name>appServlet</servlet-name>
  <servlet-class>
    org.springframework.web.servlet.DispatcherServlet
  </servlet-class>
</servlet>

<servlet-mapping>
  <servlet-name>appServlet</servlet-name>
  <url-pattern>/</url-pattern>
</servlet-mapping>
```

그리고 Spring 설정 XML 파일도 `web.xml`에서 연결했다.

```xml
<context-param>
  <param-name>contextConfigLocation</param-name>
  <param-value>/WEB-INF/spring/root-context.xml</param-value>
</context-param>
```

전통적인 구조에서는 보통 설정을 이렇게 나눴다.

```text
web.xml
  - Tomcat이 읽음
  - DispatcherServlet, Listener, Filter 등록
  - Spring XML 위치 지정

root-context.xml
  - Service
  - Repository
  - DataSource
  - Transaction

servlet-context.xml
  - Controller
  - ViewResolver
  - HandlerMapping
  - static resource mapping
```

## Filter와 Listener

Filter는 Servlet에 요청이 도달하기 전후에 실행되는 Servlet 표준 컴포넌트다.

```text
요청
  -> Filter
  -> DispatcherServlet
  -> Controller
  -> DispatcherServlet
  -> Filter
  -> 응답
```

주요 용도:

```text
- 인코딩 처리
- 인증/인가
- CORS 처리
- 요청/응답 로깅
- 보안 헤더 추가
- Spring Security
```

전통 Spring에서는 `web.xml`에 등록했다.

```xml
<filter>
  <filter-name>encodingFilter</filter-name>
  <filter-class>
    org.springframework.web.filter.CharacterEncodingFilter
  </filter-class>
</filter>

<filter-mapping>
  <filter-name>encodingFilter</filter-name>
  <url-pattern>/*</url-pattern>
</filter-mapping>
```

Listener는 Servlet Container의 이벤트를 감지하는 객체다.

예를 들면:

```text
- 웹 애플리케이션 시작/종료
- 세션 생성/소멸
- 요청 생성/소멸
```

대표 인터페이스:

```java
ServletContextListener
HttpSessionListener
ServletRequestListener
```

전통 Spring에서는 Listener도 `web.xml`에 등록했다.

Spring Boot에서는 Filter와 Listener를 보통 다음 방식으로 등록한다.

```java
@Component
public class MyFilter implements Filter {
}
```

또는:

```java
@Bean
public FilterRegistrationBean<MyFilter> myFilter() {
  FilterRegistrationBean<MyFilter> bean = new FilterRegistrationBean<>();
  bean.setFilter(new MyFilter());
  bean.addUrlPatterns("/*");
  return bean;
}
```

## Spring Boot에서는 무엇이 달라졌나?

Spring Boot는 기존의 복잡한 Servlet/Tomcat/Spring MVC 설정을 자동화한다.

전통 Spring MVC에서는 다음을 직접 설정해야 했다.

```text
- 외부 Tomcat 설치
- WAR 배포
- web.xml 작성
- DispatcherServlet 등록
- Filter/Listener 등록
- Spring XML 설정 연결
```

Spring Boot에서는 보통 다음처럼 실행한다.

```bash
java -jar app.jar
```

그러면 Boot가 내부에서 많은 것을 자동으로 처리한다.

```text
Spring Boot 시작
  -> 내장 Tomcat 시작
  -> DispatcherServlet 자동 등록
  -> Filter/Listener Bean 감지
  -> application.yml 설정 적용
  -> 요청 처리
```

`web.xml`과 `application.yml`은 같은 파일이 아니다.

```text
web.xml
  - Servlet Container가 읽는 웹 애플리케이션 배포/실행 설정

application.yml
  - Spring Boot 애플리케이션 설정값 파일
```

Boot에서는 기존 `web.xml`의 역할이 다음으로 나뉜다.

```text
DispatcherServlet 등록
  -> Boot Auto Configuration

Filter/Listener 등록
  -> @Component
  -> @Bean
  -> FilterRegistrationBean
  -> ServletListenerRegistrationBean

context-param/init-param
  -> application.yml
  -> @ConfigurationProperties
  -> @Value

session/error/multipart 설정
  -> application.yml 또는 Java Config
```

즉 Boot는 Tomcat을 없앤 것이 아니라, **Tomcat을 내장하고 자동 설정으로 편하게 실행하게 만든 것**이다.

## WAR 배포와 JAR 배포

WAR는 외부 Tomcat/WAS에 올리는 전통적인 Java 웹 애플리케이션 패키지다.

```text
myapp.war
  WEB-INF/
    web.xml
    classes/
    lib/
    views/
  static/
  index.jsp
```

WAR 배포 흐름:

```text
1. 서버에 Tomcat 설치
2. Tomcat 실행
3. WAR 파일을 Tomcat webapps 아래에 배치
4. Tomcat이 WAR를 자동으로 풀어서 배포
5. URL로 접근
```

예:

```text
/opt/tomcat/webapps/myapp.war
```

Tomcat이 풀면:

```text
/opt/tomcat/webapps/myapp/
  WEB-INF/
    web.xml
    classes/
    lib/
```

접근 URL:

```text
http://서버IP:8080/myapp/
```

`myapp.war`의 파일명 `myapp`이 context path가 된다.

Spring Boot에서 흔한 JAR 배포는 외부 Tomcat 없이 실행한다.

```bash
java -jar app.jar
```

JAR 안에는 내장 Tomcat과 애플리케이션 코드가 같이 들어 있다.

```text
app.jar
  BOOT-INF/
    classes/
    lib/
```

차이는 다음과 같다.

```text
WAR
  - 외부 Tomcat/WAS에 배포
  - WEB-INF 구조 사용
  - 전통 Servlet/JSP/Spring MVC와 친함
  - 하나의 WAS에 여러 앱 배포 가능

JAR
  - java -jar로 단독 실행
  - 내장 Tomcat 포함
  - Spring Boot 기본 방식
  - Docker, 클라우드, MSA에 적합
```

핵심 차이:

```text
WAR
  Tomcat이 먼저 실행됨
  -> WAR를 배포함
  -> Spring 앱이 Tomcat 안에서 뜸

JAR
  Spring Boot 앱을 실행함
  -> 내장 Tomcat이 같이 뜸
  -> 웹 서버까지 포함한 하나의 앱처럼 동작
```

## 컴파일과 빌드

컴파일은 Java 소스 코드를 JVM이 실행할 수 있는 바이트코드로 바꾸는 단계다.

```text
.java
  -> javac
  -> .class
```

예:

```text
src/main/java/com/example/UserService.java
  -> build/classes/java/main/com/example/UserService.class
```

빌드는 컴파일을 포함해서 실행 또는 배포 가능한 결과물을 만드는 전체 과정이다.

빌드에는 보통 다음이 포함된다.

```text
1. 소스 컴파일
2. resources 복사
3. 테스트 코드 컴파일
4. 테스트 실행
5. 의존성 라이브러리 포함
6. 정적 파일/JSP 포함
7. jar 또는 war 패키징
```

WAR로 빌드하면 `src/main/java`의 `.java` 파일이 그대로 들어가는 것이 아니라, 컴파일된 `.class` 파일이 들어간다.

```text
src/main/java
  -> 컴파일
  -> WEB-INF/classes

src/main/resources
  -> 복사
  -> WEB-INF/classes

src/main/webapp
  -> WAR 루트
```

예:

```text
src/main/java/com/example/HomeController.java
```

빌드 후:

```text
WEB-INF/classes/com/example/HomeController.class
```

정리:

```text
컴파일 = .java -> .class 로 변환

빌드 = 컴파일 + 리소스 처리 + 테스트 + 의존성 처리 + 패키징
```

## 전체 관계 최종 정리

```text
Java
  - 언어와 플랫폼

Servlet API
  - Java로 웹 요청/응답을 처리하기 위한 표준 규약

Servlet
  - Servlet API 규약을 따르는 Java 클래스
  - Tomcat이 생성하고 실행함

Tomcat
  - Servlet/JSP를 실행해주는 Java 기반 서버
  - HTTP 요청을 받고 Servlet으로 연결함

JSP
  - 서버에서 HTML을 만들기 위한 View 기술
  - 내부적으로 Servlet으로 변환되어 실행됨

Spring MVC
  - Servlet 기반 위에서 더 편하게 웹 개발하게 해주는 프레임워크
  - DispatcherServlet이 모든 요청을 받아 Controller로 넘김

REST API
  - HTML 대신 JSON을 응답하는 방식
  - Spring MVC REST API는 여전히 DispatcherServlet 기반으로 동작함

Spring Boot
  - Tomcat, DispatcherServlet, 설정 등록을 자동화함
  - 내장 Tomcat으로 java -jar 실행 가능
```

최종적으로 이렇게 이해하면 된다.

```text
Tomcat은 Java 웹앱을 실행하기 위해 만들어진 Java 기반 서버이고,
Servlet/JSP 규칙을 실제로 실행해주는 컨테이너다.

Servlet은 Tomcat이 생성/실행/관리하는 HTTP 요청 처리용 Java 클래스다.

Spring MVC는 Servlet 구조 위에서 DispatcherServlet을 통해
Controller 방식으로 요청을 처리하게 해주는 프레임워크다.

REST API는 Servlet 기반 HTTP 처리 흐름 위에서
HTML 대신 JSON을 응답하는 방식이다.

Spring Boot는 이런 Tomcat/Servlet/Spring MVC 구성을 자동화하고,
내장 Tomcat을 통해 java -jar로 바로 실행할 수 있게 해준다.
```
