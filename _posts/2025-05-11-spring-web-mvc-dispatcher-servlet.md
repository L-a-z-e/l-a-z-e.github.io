---
title: Spring Web MVC - Dispatcher Servlet
description: 
author: laze
date: 2025-05-11 00:00:01 +0900
categories: [Dev, SpringWebMVC]
tags: [SpringWebMVC]
---
**DispatcherServlet (디스패처 서블릿)**

Spring MVC는 다른 많은 웹 프레임워크와 마찬가지로 프론트 컨트롤러 패턴(front controller pattern)을 중심으로 설계되었습니다.

이 패턴에서는 중앙 서블릿(Servlet)인 `DispatcherServlet`이 요청 처리를 위한 공유 알고리즘을 제공하고, 실제 작업은 설정 가능한 위임 컴포넌트(delegate components)에 의해 수행됩니다.

이 모델은 유연하며 다양한 워크플로우를 지원합니다.

`DispatcherServlet`은 다른 서블릿과 마찬가지로, Java 설정이나 `web.xml`을 사용하여 서블릿 명세(Servlet specification)에 따라 선언되고 매핑되어야 합니다.

결과적으로 `DispatcherServlet`은 Spring 설정을 사용하여 요청 매핑, 뷰 리졸루션(view resolution), 예외 처리 등에 필요한 위임 컴포넌트들을 탐색합니다.

다음 Java 설정 예제는 `DispatcherServlet`을 등록하고 초기화하며, 이는 서블릿 컨테이너에 의해 자동 감지됩니다 (Servlet Config 참조):

**Java**

```java
public class MyWebApplicationInitializer implements WebApplicationInitializer {

	@Override
	public void onStartup(ServletContext servletContext) {

		// Spring 웹 애플리케이션 설정 로드
		AnnotationConfigWebApplicationContext context = new AnnotationConfigWebApplicationContext();
		context.register(AppConfig.class);

		// DispatcherServlet 생성 및 등록
		DispatcherServlet servlet = new DispatcherServlet(context);
		ServletRegistration.Dynamic registration = servletContext.addServlet("app", servlet);
		registration.setLoadOnStartup(1);
		registration.addMapping("/app/*");
	}
}
```

`ServletContext` API를 직접 사용하는 것 외에도, `AbstractAnnotationConfigDispatcherServletInitializer`를 확장하고 특정 메소드를 오버라이드할 수도 있습니다 (Context Hierarchy 아래 예제 참조).
프로그래밍 방식의 사용 사례를 위해, `AnnotationConfigWebApplicationContext`의 대안으로 `GenericWebApplicationContext`를 사용할 수 있습니다.

자세한 내용은 `GenericWebApplicationContext` javadoc을 참조하세요.

다음 `web.xml` 설정 예제는 `DispatcherServlet`을 등록하고 초기화합니다:

```xml
<web-app>

	<listener>
		<listener-class>org.springframework.web.context.ContextLoaderListener</listener-class>
	</listener>

	<context-param>
		<param-name>contextConfigLocation</param-name>
		<param-value>/WEB-INF/app-context.xml</param-value>
	</context-param>

	<servlet>
		<servlet-name>app</servlet-name>
		<servlet-class>org.springframework.web.servlet.DispatcherServlet</servlet-class>
		<init-param>
			<param-name>contextConfigLocation</param-name>
			<param-value></param-value> <!-- 보통 비워두거나 DispatcherServlet 자체 설정을 지정 -->
		</init-param>
		<load-on-startup>1</load-on-startup>
	</servlet>

	<servlet-mapping>
		<servlet-name>app</servlet-name>
		<url-pattern>/app/*</url-pattern>
	</servlet-mapping>

</web-app>
```

Spring Boot는 다른 초기화 순서를 따릅니다.

서블릿 컨테이너의 생명주기에 연결하는 대신, Spring Boot는 Spring 설정을 사용하여 자체적으로 부트스트랩하고 내장 서블릿 컨테이너를 초기화합니다.

필터(Filter) 및 서블릿(Servlet) 선언은 Spring 설정에서 감지되어 서블릿 컨테이너에 등록됩니다.

---

## Spring Web MVC의 핵심: DispatcherServlet 이해하기

Spring Web MVC 프레임워크의 심장과 같은 역할을 하는 것이 바로 **DispatcherServlet**입니다.

마치 레스토랑의 총괄 매니저나 복잡한 교차로의 교통 경찰관처럼, 웹 애플리케이션으로 들어오는 모든 요청을 가장 먼저 받아 적절한 담당자에게 전달하는 역할을 합니다.

### 1. 프론트 컨트롤러 패턴 (Front Controller Pattern) 이란?

DispatcherServlet을 이해하기 위해서는 먼저 "프론트 컨트롤러 패턴"을 알아야 합니다.

- **개념:** 모든 클라이언트의 요청을 단일 진입점(여기서는 DispatcherServlet)에서 처리하는 디자인 패턴입니다.
- **장점:**
  - **중앙 집중 처리:** 요청 처리 로직이 한 곳에 집중되어 관리가 용이합니다.
  - **공통 작업 용이:** 인증, 로깅, 인코딩 설정 등 공통적으로 처리해야 할 작업들을 프론트 컨트롤러에서 일괄적으로 수행할 수 있습니다.
  - **유연성 및 확장성:** 실제 작업은 다른 '위임 컴포넌트'들에게 맡기므로, 새로운 기능을 추가하거나 기존 기능을 변경하기 쉽습니다.

DispatcherServlet은 바로 이 프론트 컨트롤러 패턴의 구현체입니다.

### 2. DispatcherServlet의 역할

- **요청 접수:** 웹 브라우저와 같은 클라이언트로부터 들어오는 HTTP 요청을 가장 먼저 받습니다.
- **요청 분석 및 위임:** 받은 요청을 분석하여 어떤 컨트롤러(Controller)에게 작업을 맡길지 결정하고, 해당 컨트롤러에게 요청을 전달합니다.
- **응답 처리 조정:** 컨트롤러가 처리한 결과를 받아, 적절한 뷰(View, 예를 들어 JSP, Thymeleaf 템플릿 파일 등)를 통해 사용자에게 보여줄 최종 응답을 생성하도록 조정합니다.
- **기타:** 예외 처리, 뷰 리졸루션(View Resolution) 등 다양한 웹 관련 작업을 총괄합니다.

**핵심:** DispatcherServlet 자체는 모든 일을 직접 하지 않습니다. 요청을 받고, 그 요청을 처리할 적절한 전문가(다른 컴포넌트들)에게 일을 분배하고 조율하는 역할을 합니다.

### 3. DispatcherServlet 설정 방법

DispatcherServlet을 사용하려면 서블릿 컨테이너(예: Tomcat)에게 "이 서블릿을 사용하겠다"라고 알려줘야 합니다. 설정 방법은 크게 두 가지가 있습니다.

### 가. Java 설정 (최신 방식, 권장)

XML 설정 없이 Java 클래스만으로 DispatcherServlet을 설정할 수 있습니다.

- **`WebApplicationInitializer` 인터페이스 활용:**
  - 서블릿 컨테이너가 시작될 때 `onStartup` 메소드가 호출됩니다.
  - 이 메소드 안에서 DispatcherServlet을 생성하고 등록합니다.

    ```java
    // MyWebApplicationInitializer.java
    public class MyWebApplicationInitializer implements WebApplicationInitializer {
    
        @Override
        public void onStartup(ServletContext servletContext) { // 서블릿 컨테이너가 시작될 때 호출됨
    
            // 1. Spring 설정을 로드할 컨텍스트(Context) 생성
            // AnnotationConfigWebApplicationContext는 @Configuration 어노테이션이 붙은 클래스에서 설정을 읽어옴
            AnnotationConfigWebApplicationContext context = new AnnotationConfigWebApplicationContext();
            context.register(AppConfig.class); // AppConfig.class 라는 설정 파일을 사용하겠다고 지정
    
            // 2. DispatcherServlet 생성 및 등록
            // 위에서 만든 Spring 컨텍스트를 DispatcherServlet에 주입
            DispatcherServlet servlet = new DispatcherServlet(context);
    
            // servletContext에 "app"이라는 이름으로 servlet(DispatcherServlet)을 등록
            ServletRegistration.Dynamic registration = servletContext.addServlet("app", servlet);
    
            // 3. DispatcherServlet 설정
            registration.setLoadOnStartup(1); // 웹 애플리케이션 시작 시 DispatcherServlet을 우선적으로 로드 (숫자가 작을수록 우선순위 높음)
            registration.addMapping("/app/*"); // "/app/"으로 시작하는 모든 URL 요청을 이 DispatcherServlet이 처리하도록 매핑
        }
    }
    ```

  - **`AppConfig.class` 예시 (간단한 형태):**

      ```java
      // AppConfig.java
      import org.springframework.context.annotation.ComponentScan;
      import org.springframework.context.annotation.Configuration;
      import org.springframework.web.servlet.config.annotation.EnableWebMvc;
      
      @Configuration // 이 클래스가 Spring 설정 파일임을 나타냄
      @EnableWebMvc  // Spring MVC 사용을 활성화 (다양한 기본 설정 자동 수행)
      @ComponentScan(basePackages = "com.example.controller") // @Controller 어노테이션이 붙은 클래스를 찾아 빈으로 등록
      public class AppConfig {
          // 여기에 뷰 리졸버, 인터셉터 등 다른 Spring MVC 관련 설정을 추가할 수 있습니다.
      }
      ```

  - **`AbstractAnnotationConfigDispatcherServletInitializer` 활용:**`WebApplicationInitializer`를 직접 구현하는 것보다 더 간편하게 DispatcherServlet을 설정할 수 있도록 Spring에서 제공하는 추상 클래스입니다. 아래와 같이 필요한 메소드만 오버라이드하면 됩니다.

      ```java
      public class MyWebAppInitializer extends AbstractAnnotationConfigDispatcherServletInitializer {
      
          @Override
          protected Class<?>[] getRootConfigClasses() {
              // 루트 애플리케이션 컨텍스트 설정 (일반적으로 서비스, 리포지토리 빈)
              // 여기서는 사용하지 않으므로 null 또는 빈 배열 반환
              return null;
          }
      
          @Override
          protected Class<?>[] getServletConfigClasses() {
              // DispatcherServlet의 애플리케이션 컨텍스트 설정 (웹 관련 빈: 컨트롤러, 뷰 리졸버 등)
              return new Class<?>[] { AppConfig.class };
          }
      
          @Override
          protected String[] getServletMappings() {
              // DispatcherServlet이 처리할 URL 패턴 지정
              return new String[] { "/app/*" };
          }
      }
      ```


### 나. web.xml 설정 (전통적인 방식)

과거에는 `web.xml` 파일을 사용하여 서블릿을 설정했습니다.

```xml
<!-- web.xml -->
<web-app>

    <!-- (선택) 루트 웹 애플리케이션 컨텍스트 설정을 위한 리스너 -->
    <!-- 보통 데이터베이스 연동, 서비스 계층 빈 등을 여기에 설정 -->
    <listener>
        <listener-class>org.springframework.web.context.ContextLoaderListener</listener-class>
    </listener>
    <context-param>
        <param-name>contextConfigLocation</param-name>
        <param-value>/WEB-INF/app-context.xml</param-value> <!-- 루트 컨텍스트 설정 파일 위치 -->
    </context-param>

    <!-- DispatcherServlet 정의 -->
    <servlet>
        <servlet-name>app</servlet-name> <!-- 서블릿 이름 (임의 지정) -->
        <servlet-class>org.springframework.web.servlet.DispatcherServlet</servlet-class> <!-- DispatcherServlet 클래스 경로 -->
        <!-- (선택) DispatcherServlet 자체의 Spring 설정 파일 위치 -->
        <!-- 비워두면 /WEB-INF/[서블릿이름]-servlet.xml (예: app-servlet.xml)을 기본으로 찾음 -->
        <init-param>
            <param-name>contextConfigLocation</param-name>
            <param-value></param-value> <!-- 또는 /WEB-INF/mvc-config.xml 등 지정 -->
        </init-param>
        <load-on-startup>1</load-on-startup> <!-- 웹 애플리케이션 시작 시 우선 로드 -->
    </servlet>

    <!-- DispatcherServlet URL 매핑 -->
    <servlet-mapping>
        <servlet-name>app</servlet-name> <!-- 위에서 정의한 서블릿 이름과 동일해야 함 -->
        <url-pattern>/app/*</url-pattern> <!-- 이 URL 패턴으로 오는 요청을 'app' 서블릿이 처리 -->
    </servlet-mapping>

</web-app>
```

- **`ContextLoaderListener`**: `DispatcherServlet`과는 별개로, 애플리케이션 전반에서 사용될 루트(Root) Spring 컨테이너를 생성하고 관리합니다. 보통 서비스(Service), DAO(Repository) 등의 비즈니스 로직 관련 빈들을 여기에 등록합니다.
- **`DispatcherServlet`의 `init-param` (contextConfigLocation)**: `DispatcherServlet` 자신만의 Spring 컨테이너(Servlet WebApplicationContext)를 위한 설정 파일을 지정합니다. 여기에는 주로 컨트롤러(Controller), 뷰 리졸버(ViewResolver), 인터셉터(Interceptor) 등 웹 계층과 관련된 빈들을 등록합니다. 만약 이 값을 비워두면, 기본적으로 `/WEB-INF/[서블릿이름]-servlet.xml` (예: `app-servlet.xml`) 파일을 찾습니다.

### 4. Spring Boot에서의 DispatcherServlet

Spring Boot를 사용하면 이러한 설정 과정이 훨씬 간단해집니다.

- **자동 설정 (Auto-configuration):** Spring Boot는 클래스패스에 `spring-boot-starter-web` 의존성이 있으면 자동으로 DispatcherServlet을 설정하고 등록합니다.
- **내장 서블릿 컨테이너:** 별도의 Tomcat을 설치하고 설정할 필요 없이, Spring Boot 애플리케이션을 실행하면 내장된 Tomcat (또는 Jetty, Undertow)이 함께 실행됩니다.
- 사용자는 `application.properties` 또는 `application.yml` 파일을 통해 간단히 설정을 변경할 수 있습니다.
- 위에서 설명한 `WebApplicationInitializer`나 `web.xml` 설정이 대부분 필요 없어집니다.

### 5. DispatcherServlet과 위임 컴포넌트

DispatcherServlet은 요청을 받으면, 실제 처리를 위해 다음과 같은 다양한 위임 컴포넌트들과 협력합니다. (이 컴포넌트들은 앞으로 자세히 배우게 됩니다.)

- **`HandlerMapping` (핸들러 매핑):** 요청 URL에 해당하는 컨트롤러(핸들러)를 찾아주는 역할.
- **`HandlerAdapter` (핸들러 어댑터):** 찾아낸 컨트롤러를 실행하는 역할.
- **`ViewResolver` (뷰 리졸버):** 컨트롤러가 반환한 뷰 이름을 실제 뷰 객체(예: JSP 파일 경로)로 변환하는 역할.
- **`LocaleResolver` (로케일 리졸버):** 클라이언트의 지역 정보를 결정하는 역할.
- **`ThemeResolver` (테마 리졸버):** 웹 애플리케이션의 테마를 결정하는 역할.
- **`MultipartResolver` (멀티파트 리졸버):** 파일 업로드 요청을 처리하는 역할.
- **`HandlerExceptionResolver` (핸들러 예외 리졸버):** 요청 처리 중 발생한 예외를 처리하는 역할.
