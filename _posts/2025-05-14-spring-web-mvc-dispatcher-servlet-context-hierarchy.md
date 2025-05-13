---
title: Spring Web MVC - Dispatcher Servlet (Context Hierarchy)
description: 
author: laze
date: 2025-05-14 00:00:01 +0900
categories: [Dev, SpringWebMVC]
tags: [SpringWebMVC]
---
**컨텍스트 계층구조 (Context Hierarchy)**

`DispatcherServlet`은 자체 설정을 위해 `WebApplicationContext`(일반 `ApplicationContext`의 확장)를 필요로 합니다.

`WebApplicationContext`는 `ServletContext` 및 자신이 연관된 서블릿(Servlet)에 대한 링크를 가지고 있습니다.

또한, 애플리케이션이 `WebApplicationContext`에 접근해야 할 경우 `RequestContextUtils`의 정적 메소드를 사용하여 조회할 수 있도록 `ServletContext`에 바인딩됩니다.

많은 애플리케이션의 경우, 단일 `WebApplicationContext`를 갖는 것이 간단하고 충분합니다.

그러나 하나의 루트 `WebApplicationContext`가 여러 `DispatcherServlet` (또는 다른 서블릿) 인스턴스 간에 공유되고,

각 인스턴스는 자체적인 자식 `WebApplicationContext` 설정을 갖는 컨텍스트 계층구조를 가질 수도 있습니다.

컨텍스트 계층구조 기능에 대한 자세한 내용은 애플리케이션 컨텍스트의 추가 기능(Additional Capabilities of the ApplicationContext)을 참조하세요.

루트 `WebApplicationContext`는 일반적으로 여러 서블릿 인스턴스 간에 공유되어야 하는 데이터 저장소(data repositories) 및

비즈니스 서비스(business services)와 같은 인프라 빈(infrastructure beans)을 포함합니다.

이러한 빈들은 효과적으로 상속되며, 특정 서블릿에 국한된 빈들을 일반적으로 포함하는 서블릿별 자식 `WebApplicationContext`에서 오버라이드(즉, 재선언)될 수 있습니다.

다음 예제는 `WebApplicationContext` 계층구조를 설정합니다:

**Java**

```java
public class MyWebAppInitializer extends AbstractAnnotationConfigDispatcherServletInitializer {

	@Override
	protected Class<?>[] getRootConfigClasses() {
		return new Class<?>[] { RootConfig.class };
	}

	@Override
	protected Class<?>[] getServletConfigClasses() {
		return new Class<?>[] { App1Config.class };
	}

	@Override
	protected String[] getServletMappings() {
		return new String[] { "/app1/*" };
	}
}
```

만약 애플리케이션 컨텍스트 계층구조가 필요하지 않은 경우, 애플리케이션은 `getRootConfigClasses()`를 통해 모든 설정을 반환하고 `getServletConfigClasses()`에서는 `null`을 반환할 수 있습니다.

다음 예제는 `web.xml`에서의 동일한 설정을 보여줍니다:

```xml
<web-app>

	<listener>
		<listener-class>org.springframework.web.context.ContextLoaderListener</listener-class>
	</listener>

	<context-param>
		<param-name>contextConfigLocation</param-name>
		<param-value>/WEB-INF/root-context.xml</param-value>
	</context-param>

	<servlet>
		<servlet-name>app1</servlet-name>
		<servlet-class>org.springframework.web.servlet.DispatcherServlet</servlet-class>
		<init-param>
			<param-name>contextConfigLocation</param-name>
			<param-value>/WEB-INF/app1-context.xml</param-value>
		</init-param>
		<load-on-startup>1</load-on-startup>
	</servlet>

	<servlet-mapping>
		<servlet-name>app1</servlet-name>
		<url-pattern>/app1/*</url-pattern>
	</servlet-mapping>

</web-app>

```

만약 애플리케이션 컨텍스트 계층구조가 필요하지 않은 경우, 애플리케이션은 "루트" 컨텍스트만 설정하고 `contextConfigLocation` 서블릿 파라미터를 비워둘 수 있습니다.

---

## Spring Web MVC의 컨텍스트 계층구조 이해하기

Spring 애플리케이션에서 **`ApplicationContext`**는 빈(Bean)들의 생명주기를 관리하고 의존성을 주입하는 핵심 컨테이너입니다.

웹 애플리케이션에서는 이 `ApplicationContext`를 확장한 **`WebApplicationContext`**를 사용합니다.

`DispatcherServlet`은 바로 이 `WebApplicationContext`를 통해 자신이 필요로 하는 빈들(컨트롤러, 뷰 리졸버 등)을 관리하고 사용합니다.

### 1. WebApplicationContext란?

- **`ApplicationContext`의 확장:** 일반적인 Spring의 `ApplicationContext`가 제공하는 모든 기능(빈 관리, DI, AOP 등)을 포함합니다.
- **웹 환경 특화 기능:**
  - **`ServletContext` 접근:** 웹 애플리케이션의 표준 컨텍스트인 `ServletContext`에 접근할 수 있는 기능을 제공합니다. 이를 통해 웹 리소스 경로를 얻거나, `ServletContext`에 정보를 저장/조회할 수 있습니다.
  - **`ServletContext`에 바인딩:** `WebApplicationContext` 자체는 `ServletContext`의 속성(attribute)으로 저장되어, `RequestContextUtils.findWebApplicationContext(...)`와 같은 유틸리티 메소드를 통해 어디서든 현재 `WebApplicationContext`에 접근할 수 있게 됩니다.
- **`DispatcherServlet`과의 관계:** 각 `DispatcherServlet`은 자신만의 `WebApplicationContext`를 가집니다. 이 컨텍스트는 해당 `DispatcherServlet`과 관련된 웹 컴포넌트들(컨트롤러, 핸들러 매핑, 뷰 리졸버 등)을 설정하고 관리합니다.

### 2. 컨텍스트 계층구조 (Context Hierarchy)

대부분의 간단한 웹 애플리케이션에서는 하나의 `WebApplicationContext`만으로 충분합니다. 하지만 더 복잡한 애플리케이션에서는 여러 `DispatcherServlet` 인스턴스가 존재하거나, 웹 계층과 비즈니스 계층의 설정을 분리하고 싶을 때 컨텍스트 계층구조를 사용할 수 있습니다.

- **개념:** 부모-자식 관계를 갖는 여러 `ApplicationContext`를 구성하는 것입니다.
  - **루트 WebApplicationContext (Root WebApplicationContext):**
    - 계층구조의 최상위에 위치하는 부모 컨텍스트입니다.
    - 일반적으로 애플리케이션 전반에서 공유되어야 하는 빈들을 설정합니다.
      - **예시:** 데이터베이스 연동 설정(DataSource), 트랜잭션 관리자(TransactionManager), 서비스(Service) 빈, 데이터 접근 객체(DAO/Repository) 빈 등.
    - `ContextLoaderListener` (web.xml 사용 시) 또는 `getRootConfigClasses()` (Java 설정 시 `AbstractAnnotationConfigDispatcherServletInitializer` 사용 시)를 통해 생성됩니다.
  - **서블릿 WebApplicationContext (Servlet WebApplicationContext):**
    - 루트 `WebApplicationContext`를 부모로 하는 자식 컨텍스트입니다.
    - 각 `DispatcherServlet` 인스턴스마다 하나씩 생성됩니다.
    - 해당 `DispatcherServlet`과 관련된 웹 계층 빈들만 설정합니다.
      - **예시:** 컨트롤러(@Controller), 핸들러 매핑(HandlerMapping), 핸들러 어댑터(HandlerAdapter), 뷰 리졸버(ViewResolver), 인터셉터(Interceptor) 등.
    - `DispatcherServlet` 자체에 의해 생성되며, `contextConfigLocation` 초기화 파라미터 (web.xml 사용 시) 또는 `getServletConfigClasses()` (Java 설정 시)를 통해 설정 파일이 지정됩니다.
- **특징:**
  - **빈 상속:** 자식 컨텍스트는 부모 컨텍스트에 정의된 빈을 참조할 수 있습니다. (예: 서블릿 컨텍스트의 컨트롤러는 루트 컨텍스트의 서비스 빈을 주입받을 수 있습니다.)
  - **빈 격리:** 부모 컨텍스트는 자식 컨텍스트에 정의된 빈을 참조할 수 없습니다. (예: 루트 컨텍스트는 특정 서블릿 컨텍스트의 컨트롤러를 알 수 없습니다.)
  - **빈 오버라이딩:** 자식 컨텍스트에서 부모 컨텍스트와 동일한 이름으로 빈을 정의하면, 자식 컨텍스트의 빈이 우선적으로 사용됩니다 (오버라이딩).
  - **관심사 분리:** 웹 관련 설정(자식)과 비즈니스 로직/인프라 설정(부모)을 분리하여 관리할 수 있어 애플리케이션 구조가 명확해집니다.
  - **공유 빈 관리:** 여러 서블릿에서 공통으로 사용되는 빈들을 루트 컨텍스트에 한 번만 정의하여 공유할 수 있습니다.

**[컨텍스트 계층구조 시각화]**

```
                  [ Root WebApplicationContext ]
                  (DataSource, Service, Repository 등 공유 빈)
                          | (부모)
          -----------------------------------
          |                                 |
 (자식) |                               (자식) |
[ Servlet WebApplicationContext 1 ]  [ Servlet WebApplicationContext 2 ]
(DispatcherServlet 1 용)             (DispatcherServlet 2 용)
(Controller, ViewResolver 등)        (Controller, ViewResolver 등)
(URL: /app1/*)                       (URL: /app2/*)

```

### 3. 컨텍스트 계층구조 설정 방법

### 가. Java 설정 ( `AbstractAnnotationConfigDispatcherServletInitializer` 사용)

이 클래스는 루트 컨텍스트와 서블릿 컨텍스트 설정을 위한 메소드를 명확하게 제공합니다.

```java
// MyWebAppInitializer.java
public class MyWebAppInitializer extends AbstractAnnotationConfigDispatcherServletInitializer {

    // 루트 WebApplicationContext 설정
    @Override
    protected Class<?>[] getRootConfigClasses() {
        // RootConfig.class에 DataSource, Service 등 공통 빈 설정
        return new Class<?>[] { RootConfig.class };
    }

    // 서블릿 WebApplicationContext 설정 (DispatcherServlet용)
    @Override
    protected Class<?>[] getServletConfigClasses() {
        // App1Config.class에 Controller, ViewResolver 등 웹 관련 빈 설정
        return new Class<?>[] { App1Config.class };
    }

    // 이 DispatcherServlet이 처리할 URL 매핑
    @Override
    protected String[] getServletMappings() {
        return new String[] { "/app1/*" }; // /app1/로 시작하는 모든 요청을 이 DispatcherServlet이 처리
    }
}

// RootConfig.java (루트 컨텍스트 설정 예시)
@Configuration
@ComponentScan(basePackages = "com.example.service", "com.example.repository")
// @EnableTransactionManagement 등
public class RootConfig {
    // DataSource, TransactionManager 빈 정의 등
}

// App1Config.java (서블릿 컨텍스트 설정 예시)
@Configuration
@EnableWebMvc
@ComponentScan(basePackages = "com.example.controller.app1") // app1 관련 컨트롤러
public class App1Config {
    // ViewResolver 빈 정의 등
}

```

- **계층구조가 필요 없는 경우:**
  모든 설정을 루트 컨텍스트에만 하고 싶다면, `getRootConfigClasses()`에 모든 설정 클래스를 지정하고, `getServletConfigClasses()`는 `null` 또는 빈 배열을 반환하면 됩니다.
  이 경우, `DispatcherServlet`은 루트 `WebApplicationContext`를 직접 사용하게 됩니다.

    ```java
    @Override
    protected Class<?>[] getRootConfigClasses() {
        return new Class<?>[] { AppConfig.class }; // 모든 설정 포함
    }
    
    @Override
    protected Class<?>[] getServletConfigClasses() {
        return null; // 서블릿 컨텍스트 설정 사용 안 함
    }
    
    ```


### 나. web.xml 설정

`ContextLoaderListener`와 `DispatcherServlet`의 설정을 통해 계층구조를 만듭니다.

```xml
<!-- web.xml -->
<web-app>

    <!-- 1. 루트 WebApplicationContext 설정 -->
    <listener>
        <listener-class>org.springframework.web.context.ContextLoaderListener</listener-class>
    </listener>
    <context-param>
        <param-name>contextConfigLocation</param-name>
        <!-- 루트 컨텍스트 설정 파일 (예: 서비스, DAO 빈 정의) -->
        <param-value>/WEB-INF/root-context.xml</param-value>
    </context-param>

    <!-- 2. 서블릿 WebApplicationContext 설정 (DispatcherServlet용) -->
    <servlet>
        <servlet-name>app1</servlet-name>
        <servlet-class>org.springframework.web.servlet.DispatcherServlet</servlet-class>
        <init-param>
            <param-name>contextConfigLocation</param-name>
            <!-- app1 DispatcherServlet용 컨텍스트 설정 파일 (예: 컨트롤러, 뷰 리졸버 빈 정의) -->
            <param-value>/WEB-INF/app1-context.xml</param-value>
        </init-param>
        <load-on-startup>1</load-on-startup>
    </servlet>

    <servlet-mapping>
        <servlet-name>app1</servlet-name>
        <url-pattern>/app1/*</url-pattern>
    </servlet-mapping>

    <!-- (만약 다른 DispatcherServlet이 있다면) -->
    <!--
    <servlet>
        <servlet-name>app2</servlet-name>
        <servlet-class>org.springframework.web.servlet.DispatcherServlet</servlet-class>
        <init-param>
            <param-name>contextConfigLocation</param-name>
            <param-value>/WEB-INF/app2-context.xml</param-value>
        </init-param>
        <load-on-startup>1</load-on-startup>
    </servlet>
    <servlet-mapping>
        <servlet-name>app2</servlet-name>
        <url-pattern>/app2/*</url-pattern>
    </servlet-mapping>
    -->
</web-app>

```

- **계층구조가 필요 없는 경우 (루트 컨텍스트만 사용):**`ContextLoaderListener`를 통해 루트 컨텍스트를 설정하고, `DispatcherServlet`의 `contextConfigLocation` 초기화 파라미터를 비워두거나 아예 생략합니다. 이 경우 `DispatcherServlet`은 부모 컨텍스트(즉, `ContextLoaderListener`가 생성한 루트 컨텍스트)를 자신의 컨텍스트로 사용합니다.

    ```xml
    <!-- web.xml - 루트 컨텍스트만 사용하는 경우 -->
    <web-app>
        <listener>
            <listener-class>org.springframework.web.context.ContextLoaderListener</listener-class>
        </listener>
        <context-param>
            <param-name>contextConfigLocation</param-name>
            <param-value>/WEB-INF/applicationContext.xml</param-value> <!-- 모든 설정 포함 -->
        </context-param>
    
        <servlet>
            <servlet-name>app</servlet-name>
            <servlet-class>org.springframework.web.servlet.DispatcherServlet</servlet-class>
            <!-- contextConfigLocation을 비워두면 루트 컨텍스트를 사용 -->
            <init-param>
                <param-name>contextConfigLocation</param-name>
                <param-value></param-value>
            </init-param>
            <load-on-startup>1</load-on-startup>
        </servlet>
        <servlet-mapping>
            <servlet-name>app</servlet-name>
            <url-pattern>/</url-pattern>
        </servlet-mapping>
    </web-app>
    
    ```


### 4. 언제 컨텍스트 계층구조를 사용할까?

- **여러 `DispatcherServlet`이 서로 다른 URL 패턴을 처리하지만, 공통의 서비스/데이터 계층을 공유해야 할 때.**
  - 예: `/api/*` 요청은 REST API를 처리하는 `DispatcherServlet`이, `/web/*` 요청은 웹 페이지를 처리하는 다른 `DispatcherServlet`이 담당하지만, 둘 다 동일한 사용자 서비스, 상품 서비스를 사용해야 하는 경우.
- **애플리케이션의 규모가 커서 웹 계층과 비즈니스/데이터 계층의 설정을 명확히 분리하고 싶을 때.** (관심사 분리)
- **일반적으로 Spring Boot를 사용하면 단일 `ApplicationContext`를 사용하는 것이 기본이며, 대부분의 경우 충분합니다.** Spring Boot는 컴포넌트 스캔과 자동 설정을 통해 빈들을 관리하므로, 명시적인 계층구조 설정의 필요성이 줄어듭니다. 하지만 특정 요구사항(예: 여러 `DispatcherServlet` 사용)이 있다면 Spring Boot 환경에서도 구성할 수 있습니다.
