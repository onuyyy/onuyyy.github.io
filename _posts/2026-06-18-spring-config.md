---
title: "Spring (Legacy) 설정 파일 정리"
date: 2026-06-18 12:00:00 +0900
categories: [Spring]
tags: [Spring, Config, XML, Legacy]
---

전통적인(legacy) Spring MVC 프로젝트의 XML 설정 파일들과 그 구조, 네임스페이스, 그리고 **Spring Boot에서는 무엇이 이 역할을 대신하는지**를 정리한다.

> 핵심 한 줄: 이 XML들은 "톰캣이 뜰 때 스프링 웹앱을 어떻게 구동할지"를 손으로 적어준 것이고, **Spring Boot는 이걸 어노테이션 설정 + 자동 구성(auto-configuration)으로 흡수**했다.

---

## 1. 설정 파일 3종과 역할

| 파일 | 누가 읽나 | 역할 | 들어가는 빈 |
|------|----------|------|------------|
| `web.xml` | 톰캣(서블릿 컨테이너) | 웹앱 구동 시동 (스프링 띄우기) | (설정만, 빈 없음) |
| `root-context.xml` | `ContextLoaderListener` | **Root 컨텍스트(부모)** — 공통/비즈니스 | DataSource, MyBatis, Service, DAO |
| `servlet-context.xml` | `DispatcherServlet` | **Servlet 컨텍스트(자식)** — 웹/MVC | Controller, ViewResolver, 인터셉터 |

### web.xml — 진입점 (서블릿 표준 파일)
톰캣이 앱 시작 시 가장 먼저 읽는다. 스프링 파일이 아니라 서블릿 표준. 하는 일 2가지:

1. **`ContextLoaderListener` 등록** → `root-context.xml`을 읽어 Root 컨텍스트 생성
   ```xml
   <listener>
     <listener-class>org.springframework.web.context.ContextLoaderListener</listener-class>
   </listener>
   <context-param>
     <param-name>contextConfigLocation</param-name>
     <param-value>WEB-INF/spring/root-context.xml</param-value>
   </context-param>
   ```
2. **`DispatcherServlet` 등록** → `servlet-context.xml`을 읽어 Servlet 컨텍스트 생성, 모든 요청(`/`)을 받음
   ```xml
   <servlet>
     <servlet-name>appServlet</servlet-name>
     <servlet-class>org.springframework.web.servlet.DispatcherServlet</servlet-class>
     <init-param>
       <param-name>contextConfigLocation</param-name>
       <param-value>WEB-INF/spring/servlet-context.xml</param-value>
     </init-param>
     <load-on-startup>1</load-on-startup>   <!-- 톰캣 뜰 때 같이 로딩, 숫자 낮을수록 먼저 -->
   </servlet>
   <servlet-mapping>
     <servlet-name>appServlet</servlet-name>
     <url-pattern>/</url-pattern>            <!-- / : .jsp 직접접근 제외 전부 / *: .jsp 포함 전부 -->
   </servlet-mapping>
   ```

### root-context.xml — 공통 빈 (부모)
웹과 무관한 공통 빈. 예를 들어: DataSource(HikariCP), SqlSessionFactory(MyBatis), 매퍼 스캔. (보통 Service/DAO/트랜잭션도 여기)

### servlet-context.xml — 웹 빈 (자식)
MVC 전용 빈. 예시:
```xml
<context:component-scan base-package="com.example.demo.controller"/>
<mvc:annotation-driven />
```

---

## 2. 컨텍스트 구조 — 왜 root / servlet 둘로 나누나

두 컨텍스트는 **부모-자식 관계**다.

```
       [Root ApplicationContext]      ← root-context.xml
        (DataSource, MyBatis, Service, DAO)
                  ▲ 부모
                  │  자식은 부모 빈을 참조 가능 / 부모는 자식 빈 못 봄
        [Servlet ApplicationContext]  ← servlet-context.xml
        (Controller, ViewResolver)
```

- 자식(Controller) → 부모(Service/DAO) 주입 **가능** ✅
- 부모(Service) → 자식(Controller) 참조 **불가** ❌

**나누는 이유:**
1. 관심사 분리 (웹 계층 vs 비즈니스 계층)
2. 재사용성 — 공통 빈을 웹 외 진입점(배치/스케줄러)에서도 공유
3. `DispatcherServlet`을 여러 개 둘 때 공통 빈(Root)은 하나로 공유

### 구동 + 요청 흐름
```
톰캣 시작 → web.xml
   → ContextLoaderListener: root-context.xml → Root 컨텍스트 (DB/MyBatis/Service)
   → DispatcherServlet: servlet-context.xml → Servlet 컨텍스트 (Controller)

요청 (GET /users)
   → url-pattern "/" → DispatcherServlet
   → Controller 실행 (필요 시 Root의 Service/DAO 사용)
   → View 반환 → 응답
```

---

## 3. XML 구조 & 네임스페이스(namespace)

세 설정 XML(특히 root/servlet)은 **같은 `<beans>` 기반 XML**이다. 단, 파일마다 **선언한 `xmlns`(네임스페이스)가 달라서 쓸 수 있는 태그가 달라진다.**

### xmlns 한 줄의 구조
```xml
xmlns:mvc="http://www.springframework.org/schema/mvc"
      └┬┘  └────────────── 네임스페이스 식별자(정체/출처) ──────────────┘
    접두사(이 파일 안에서 쓰는 별명)
```
- 왼쪽 접두사 = 이 파일에서만 쓰는 별명 (바꿔도 됨)
- 오른쪽 URL = 네임스페이스 고유 식별자 (단순 식별용, 실제 접속 안 함)
- `xsi:schemaLocation`이 각 네임스페이스 ↔ **문법 규칙서(XSD)** 위치를 짝지어 줌

### 파일별 선언된 네임스페이스 (= 쓸 수 있는 태그)

| 파일 | 선언한 xmlns | 쓸 수 있는 커스텀 태그 |
|------|-------------|---------------------|
| root-context | beans, context, **mybatis-spring** | `<context:component-scan>`, `<mybatis-spring:scan>` |
| servlet-context | beans, context, **mvc** | `<context:component-scan>`, `<mvc:annotation-driven>` |

→ 쓸 수 있는 태그 = **선언한 xmlns들의 합집합**. root엔 `mvc:`가 없어 `<mvc:...>` 못 쓰고, servlet엔 `mybatis-spring:`이 없어 `<mybatis-spring:scan>` 못 씀.
→ 새 커스텀 태그(예: `<tx:...>`)를 쓰려면 그 `xmlns`와 `schemaLocation`을 **직접 추가**해야 함. 안 하면 "지원되지 않는 태그" 에러.

### 커스텀 태그 동작 원리 (`<mvc:>`, `<context:>`, `<mybatis-spring:>` 공통)
1. 접두사 → `xmlns`에 선언된 네임스페이스로 연결
2. 스프링이 그 네임스페이스를 처리할 **NamespaceHandler** 클래스를 찾음
3. 핸들러가 태그를 읽어 실제 빈들로 변환
   > 앞서 `UnsupportedClassVersionError: ...mybatis.spring.config.NamespaceHandler`가 난 게 바로 이 3단계 — 핸들러 로드 시 자바 버전이 안 맞아 깨졌던 것.

### 자주 쓰는 커스텀 태그
- `<context:component-scan base-package="..."/>` : `@Component/@Service/@Repository/@Controller` 스캔해 빈 등록
- `<mvc:annotation-driven/>` : 어노테이션 기반 MVC 활성화 (핸들러 매핑/어댑터, 데이터 바인딩, 메시지 컨버터(JSON), 검증 등 한 번에)
- `<mybatis-spring:scan base-package="..."/>` : 매퍼 인터페이스 스캔해 빈 등록
- `<bean>` : 외부 라이브러리 등 어노테이션 못 붙이는 객체를 수동 빈 등록

> 참고: `<beans>`/`<bean>` 같은 스프링 XML 네임스페이스 문법은 **EL(Expression Language, `${}`/`#{}`)과는 무관**. 별개 개념.

---

## 4. Spring Boot에서는? (역할 대응 + 자동화)

Boot는 위 XML 3개를 전부 **어노테이션 설정 클래스 + 자동 구성**으로 대체한다. 그래서 이 파일들이 안 보인다.

| 레거시 | Spring Boot 대응 | 비고 |
|--------|-----------------|------|
| **web.xml** | 없음 — **내장 톰캣** + `main()`의 `SpringApplication.run()` | 서블릿 컨테이너가 내장되어 web.xml 불필요 |
| `ContextLoaderListener` / `DispatcherServlet` 등록 | **자동 등록** (`DispatcherServletAutoConfiguration`) | 부트가 알아서 띄움 |
| **root-context.xml** | `@Configuration` 클래스 + **자동 구성** | DataSource/MyBatis 등은 의존성만 넣으면 자동 |
| └ DataSource 설정 | `application.properties` + `DataSourceAutoConfiguration` | URL/계정만 적으면 됨 |
| └ MyBatis(SqlSessionFactory, 매퍼 스캔) | `mybatis-spring-boot-starter` + `@MapperScan` | 스타터가 자동 구성 |
| **servlet-context.xml** | **자동 구성 + `WebMvcConfigurer` 클래스** | 커스터마이징만 코드로 |
| └ `<context:component-scan>` | `@SpringBootApplication` (안에 `@ComponentScan`) | 메인 클래스 패키지 하위 자동 스캔 |
| └ `<mvc:annotation-driven>` | **자동 구성** (`WebMvcAutoConfiguration`) | 별도 선언 불필요 |
| └ ViewResolver(prefix/suffix) | `application.properties` | 아래 예시 |
| └ 인터셉터/정적리소스/CORS | `WebMvcConfigurer` 구현 클래스 | 아래 예시 |

### Boot 대응 코드 예시
```java
// web.xml + DispatcherServlet 등록 + component-scan  → 이 한 클래스로 흡수
@SpringBootApplication
@MapperScan("com.example.demo.mappers")   // <mybatis-spring:scan> 대응
public class App {
    public static void main(String[] args) {
        SpringApplication.run(App.class, args);
    }
}
```
```java
// servlet-context.xml의 "커스터마이징" 부분 대응
@Configuration
public class WebConfig implements WebMvcConfigurer {
    @Override
    public void addInterceptors(InterceptorRegistry registry) { /* ... */ }
}
```
```properties
# root/servlet의 간단 설정은 properties로
spring.datasource.url=jdbc:mariadb://localhost:3307/springdb
spring.datasource.username=springdbuser
spring.datasource.password=springdbuser
spring.mvc.view.prefix=/WEB-INF/views/
spring.mvc.view.suffix=.jsp
```

### Boot가 "자동으로 해주는" 것 정리
- 내장 톰캣 구동 (web.xml·배포 불필요)
- `DispatcherServlet`, `ContextLoaderListener` 자동 등록
- `<mvc:annotation-driven>` 상당의 MVC 기본 설정
- 컴포넌트 스캔(`@SpringBootApplication` 위치 기준)
- 의존성(starter)만 추가하면 DataSource/MyBatis/Jackson 등 자동 구성
- → 개발자는 **"기본과 다르게 할 부분"만** properties나 설정 클래스로 작성

---

## 한눈 요약

- **web.xml** = 시동 (톰캣이 스프링 띄움) → Boot: 내장 톰캣 + `run()`
- **root-context.xml** = 공통 빈(부모, DB/MyBatis/Service) → Boot: 자동 구성 + properties
- **servlet-context.xml** = 웹 빈(자식, Controller/MVC) → Boot: 자동 구성 + `WebMvcConfigurer`
- 파일마다 다른 건 **선언한 네임스페이스(xmlns)**뿐, 기본은 같은 `<beans>` XML
- 커스텀 태그(`<mvc:>` 등)는 namespace + XSD + NamespaceHandler로 동작
- Boot의 본질: 이 XML들을 **어노테이션 + 자동 구성으로 대체**, 다른 점만 직접 설정