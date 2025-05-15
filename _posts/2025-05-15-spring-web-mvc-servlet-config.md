---
title: Spring Web MVC - Dispatcher Servlet (Web MVC Config)
description: 
author: laze
date: 2025-05-15 00:00:01 +0900
categories: [Dev, SpringWebMVC]
tags: [SpringWebMVC]
---
**서블릿 설정 (Servlet Config)**

서블릿 환경에서는 `web.xml` 파일의 대안으로 또는 조합하여 프로그래밍 방식으로 서블릿 컨테이너를 설정할 수 있는 옵션이 있습니다. 다음 예제는 `DispatcherServlet`을 등록합니다:

**Java**

```java
import org.springframework.web.WebApplicationInitializer;

public class MyWebApplicationInitializer implements WebApplicationInitializer {

	@Override
	public void onStartup(ServletContext container) {
		XmlWebApplicationContext appContext = new XmlWebApplicationContext();
		appContext.setConfigLocation("/WEB-INF/spring/dispatcher-config.xml");

		ServletRegistration.Dynamic registration = container.addServlet("dispatcher", new DispatcherServlet(appContext));
		registration.setLoadOnStartup(1);
		registration.addMapping("/");
	}
}
```

`WebApplicationInitializer`는 Spring MVC에서 제공하는 인터페이스로, 여러분의 구현체가 감지되어 모든 서블릿 3 컨테이너를 초기화하는 데 자동으로 사용되도록 보장합니다.

`WebApplicationInitializer`의 추상 기본 클래스 구현인 `AbstractDispatcherServletInitializer`는 서블릿 매핑과

`DispatcherServlet` 설정 위치를 지정하는 메소드를 오버라이드함으로써 `DispatcherServlet`을 훨씬 쉽게 등록할 수 있게 해줍니다.

이는 다음 예제와 같이 Java 기반 Spring 설정을 사용하는 애플리케이션에 권장됩니다:

**Java**

```java
public class MyWebAppInitializer extends AbstractAnnotationConfigDispatcherServletInitializer {

	@Override
	protected Class<?>[] getRootConfigClasses() {
		return null;
	}

	@Override
	protected Class<?>[] getServletConfigClasses() {
		return new Class<?>[] { MyWebConfig.class };
	}

	@Override
	protected String[] getServletMappings() {
		return new String[] { "/" };
	}
}
```

XML 기반 Spring 설정을 사용하는 경우, 다음 예제와 같이 `AbstractDispatcherServletInitializer`로부터 직접 확장해야 합니다:

**Java**

```java
public class MyWebAppInitializer extends AbstractDispatcherServletInitializer {

	@Override
	protected WebApplicationContext createRootApplicationContext() {
		return null;
	}

	@Override
	protected WebApplicationContext createServletApplicationContext() {
		XmlWebApplicationContext cxt = new XmlWebApplicationContext();
		cxt.setConfigLocation("/WEB-INF/spring/dispatcher-config.xml");
		return cxt;
	}

	@Override
	protected String[] getServletMappings() {
		return new String[] { "/" };
	}
}
```

`AbstractDispatcherServletInitializer`는 또한 다음 예제와 같이 `Filter` 인스턴스를 추가하고 `DispatcherServlet`에 자동으로 매핑되도록 하는 편리한 방법을 제공합니다:

**Java**

```java
public class MyWebAppInitializer extends AbstractDispatcherServletInitializer {

	// ...

	@Override
	protected Filter[] getServletFilters() {
		return new Filter[] {
			new HiddenHttpMethodFilter(), new CharacterEncodingFilter() };
	}
}

```

각 필터는 구체적인 타입에 기반한 기본 이름으로 추가되며 `DispatcherServlet`에 자동으로 매핑됩니다.

`AbstractDispatcherServletInitializer`의 `isAsyncSupported` protected 메소드는 `DispatcherServlet`과 여기에 매핑된 모든 필터에 대해 비동기 지원을 활성화하는 단일 지점을 제공합니다.

기본적으로 이 플래그는 `true`로 설정되어 있습니다.

마지막으로, `DispatcherServlet` 자체를 추가적으로 사용자 정의해야 하는 경우, `createDispatcherServlet` 메소드를 오버라이드할 수 있습니다.

---

## `web.xml` 없는 Spring MVC 설정: Servlet Config 이해하기

과거에는 Java 웹 애플리케이션을 배포할 때 `web.xml` (배포 서술자, Deployment Descriptor) 파일에 서블릿, 필터, 리스너 등을 설정했습니다.

하지만 서블릿 3.0 명세부터는 이러한 설정을 Java 코드로도 할 수 있게 되었고, Spring MVC는 이를 적극적으로 지원합니다.

이렇게 하면 XML 없이 순수 Java 코드로만 애플리케이션 전체 설정을 관리할 수 있는 이점이 있습니다.

### 1. `WebApplicationInitializer` 인터페이스: Java 기반 설정의 시작점

- **핵심 역할:** Spring MVC는 `org.springframework.web.WebApplicationInitializer` 라는 인터페이스를 제공합니다. 이 인터페이스를 구현한 클래스가 클래스패스에 존재하면, 서블릿 3.0 이상을 지원하는 컨테이너(예: Tomcat 7 이상)는 웹 애플리케이션 시작 시 이 구현체의 `onStartup(ServletContext servletContext)` 메소드를 **자동으로 호출**합니다.
- **`onStartup` 메소드:** 이 메소드 안에서 개발자는 `ServletContext` 객체를 사용하여 프로그래밍 방식으로 `DispatcherServlet`, 필터, 리스너 등을 등록하고 설정할 수 있습니다.

**가. `WebApplicationInitializer` 직접 구현 (기본적인 방법)**

```java
// MyWebApplicationInitializer.java
import javax.servlet.ServletContext;
import javax.servlet.ServletException;
import javax.servlet.ServletRegistration;
import org.springframework.web.WebApplicationInitializer;
import org.springframework.web.context.support.XmlWebApplicationContext; // XML 설정 사용 시
// import org.springframework.web.context.support.AnnotationConfigWebApplicationContext; // Java 설정 사용 시
import org.springframework.web.servlet.DispatcherServlet;

public class MyWebApplicationInitializer implements WebApplicationInitializer {

    @Override
    public void onStartup(ServletContext servletCxt) throws ServletException {

        // 1. DispatcherServlet을 위한 Spring 설정 로드 (예: XML 기반)
        XmlWebApplicationContext appContext = new XmlWebApplicationContext();
        appContext.setConfigLocation("/WEB-INF/spring/dispatcher-config.xml");
        // 만약 Java Config를 사용한다면:
        // AnnotationConfigWebApplicationContext appContext = new AnnotationConfigWebApplicationContext();
        // appContext.register(AppConfig.class); // AppConfig는 @Configuration 클래스

        // 2. DispatcherServlet 생성 및 등록
        DispatcherServlet dispatcherServlet = new DispatcherServlet(appContext);
        ServletRegistration.Dynamic registration = servletCxt.addServlet("dispatcher", dispatcherServlet);
        // "dispatcher"는 서블릿의 이름 (임의 지정 가능)

        // 3. DispatcherServlet 설정
        registration.setLoadOnStartup(1); // 웹 애플리케이션 시작 시 우선 로드
        registration.addMapping("/");     // 모든 요청 ("/")을 이 DispatcherServlet이 처리하도록 매핑
    }
}
```

- 이 방식은 서블릿 등록의 모든 과정을 직접 코딩해야 하지만, 가장 기본적인 형태를 보여줍니다.

### 2. `AbstractDispatcherServletInitializer` 추상 클래스: 더 간편한 등록

`WebApplicationInitializer`를 직접 구현하는 것보다 `DispatcherServlet` 등록을 더 쉽게 할 수 있도록 Spring MVC는 여러 추상 기본 클래스를 제공합니다.

그중 대표적인 것이 `AbstractDispatcherServletInitializer`와 그 자식 클래스들입니다.

이 클래스들은 `DispatcherServlet` 등록과 관련된 상용구 코드(boilerplate code)를 많이 줄여주고, 개발자는 몇 가지 핵심 메소드만 오버라이드하면 됩니다.

### 가. `AbstractAnnotationConfigDispatcherServletInitializer` (Java 기반 Spring 설정 시 권장)

- 루트 애플리케이션 컨텍스트와 서블릿 애플리케이션 컨텍스트를 **Java `@Configuration` 클래스**를 사용하여 설정할 때 사용합니다.
- 이전에 "Context Hierarchy" 챕터에서 봤던 바로 그 클래스입니다!

```java
// MyWebAppInitializer.java (Java Config 사용)
import org.springframework.web.servlet.support.AbstractAnnotationConfigDispatcherServletInitializer;

public class MyWebAppInitializer extends AbstractAnnotationConfigDispatcherServletInitializer {

    // 루트 애플리케이션 컨텍스트 설정 (예: 서비스, DAO 빈)
    @Override
    protected Class<?>[] getRootConfigClasses() {
        // return new Class<?>[] { RootConfig.class }; // 루트 설정 클래스가 있다면
        return null; // 없다면 null
    }

    // 서블릿 애플리케이션 컨텍스트 설정 (DispatcherServlet용, 예: 컨트롤러, 뷰 리졸버 빈)
    @Override
    protected Class<?>[] getServletConfigClasses() {
        return new Class<?>[] { MyWebConfig.class }; // MyWebConfig는 @Configuration 클래스
    }

    // DispatcherServlet URL 매핑
    @Override
    protected String[] getServletMappings() {
        return new String[] { "/" }; // 모든 요청을 처리
    }
}
```

- `MyWebConfig.class`는 `@Configuration`과 `@EnableWebMvc` 등이 포함된 Spring MVC 설정 클래스입니다.

### 나. `AbstractDispatcherServletInitializer` (XML 기반 Spring 설정 시)

- 루트 애플리케이션 컨텍스트와 서블릿 애플리케이션 컨텍스트를 **XML 파일**을 사용하여 설정할 때 사용합니다.
- `createRootApplicationContext()`와 `createServletApplicationContext()` 메소드를 오버라이드하여 `XmlWebApplicationContext` 객체를 생성하고 설정 파일 위치를 지정합니다.

```java
// MyWebAppInitializer.java (XML Config 사용)
import org.springframework.web.context.WebApplicationContext;
import org.springframework.web.context.support.XmlWebApplicationContext;
import org.springframework.web.servlet.support.AbstractDispatcherServletInitializer;

public class MyWebAppInitializer extends AbstractDispatcherServletInitializer {

    // 루트 애플리케이션 컨텍스트 생성 (XML 기반)
    @Override
    protected WebApplicationContext createRootApplicationContext() {
        // XmlWebApplicationContext rootContext = new XmlWebApplicationContext();
        // rootContext.setConfigLocation("/WEB-INF/spring/root-context.xml");
        // return rootContext;
        return null; // 루트 컨텍스트가 없다면 null
    }

    // 서블릿 애플리케이션 컨텍스트 생성 (XML 기반)
    @Override
    protected WebApplicationContext createServletApplicationContext() {
        XmlWebApplicationContext servletAppContext = new XmlWebApplicationContext();
        servletAppContext.setConfigLocation("/WEB-INF/spring/dispatcher-config.xml");
        return servletAppContext;
    }

    // DispatcherServlet URL 매핑
    @Override
    protected String[] getServletMappings() {
        return new String[] { "/" };
    }
}
```

### 3. `AbstractDispatcherServletInitializer`의 추가 기능

이 추상 클래스들은 `DispatcherServlet` 등록 외에도 다음과 같은 편리한 기능을 제공합니다:

### 가. 필터(Filter) 등록

- `getServletFilters()` 메소드를 오버라이드하여 `DispatcherServlet`에 자동으로 매핑될 필터들을 반환할 수 있습니다.
- Spring에서 제공하는 유용한 필터들(예: `HiddenHttpMethodFilter`, `CharacterEncodingFilter`)을 쉽게 추가할 수 있습니다.

```java
// MyWebAppInitializer.java (필터 추가 예시)
import javax.servlet.Filter;
import org.springframework.web.filter.CharacterEncodingFilter;
import org.springframework.web.filter.HiddenHttpMethodFilter;
import org.springframework.web.servlet.support.AbstractAnnotationConfigDispatcherServletInitializer; // 또는 AbstractDispatcherServletInitializer

public class MyWebAppInitializer extends AbstractAnnotationConfigDispatcherServletInitializer {

    // ... (getRootConfigClasses, getServletConfigClasses, getServletMappings 생략)

    @Override
    protected Filter[] getServletFilters() {
        CharacterEncodingFilter characterEncodingFilter = new CharacterEncodingFilter();
        characterEncodingFilter.setEncoding("UTF-8");
        characterEncodingFilter.setForceEncoding(true);

        return new Filter[] {
            new HiddenHttpMethodFilter(), // HTML form에서 PUT, DELETE 등 다른 HTTP 메소드 사용 가능하게 함
            characterEncodingFilter      // 요청/응답 인코딩 설정
        };
    }

    @Override
    protected Class<?>[] getRootConfigClasses() { return null; } // 예시를 위해 간단히 null 처리

    @Override
    protected Class<?>[] getServletConfigClasses() { return new Class<?>[] { MyWebConfig.class }; } // 예시를 위해 간단히 MyWebConfig 사용

    @Override
    protected String[] getServletMappings() { return new String[] { "/" }; }
}

// MyWebConfig.java (간단한 예시)
import org.springframework.context.annotation.Configuration;
import org.springframework.web.servlet.config.annotation.EnableWebMvc;

@Configuration
@EnableWebMvc
public class MyWebConfig {
    // 필요한 MVC 설정
}
```

- 각 필터는 기본적으로 자신의 클래스 타입에 기반한 이름으로 등록되고, `DispatcherServlet`의 매핑 경로(예: `/`)에 함께 매핑됩니다.

### 나. 비동기(Async) 지원 설정

- `isAsyncSupported` protected 메소드를 오버라이드하여 `DispatcherServlet`과 여기에 매핑된 모든 필터에 대해 비동기 지원을 활성화/비활성화할 수 있습니다. 기본값은 `true`입니다.

    ```java
    // @Override
    // protected boolean isAsyncSupported() {
    //     return true; // 기본값
    // }
    ```


### 다. `DispatcherServlet` 자체 커스터마이징

- 만약 `DispatcherServlet` 인스턴스를 생성하는 방식 자체를 변경하거나, 생성된 `DispatcherServlet`에 추가적인 설정을 하고 싶다면 `createDispatcherServlet(WebApplicationContext servletAppContext)` 메소드를 오버라이드할 수 있습니다.

    ```java
    // @Override
    // protected DispatcherServlet createDispatcherServlet(WebApplicationContext servletAppContext) {
    //     DispatcherServlet ds = new DispatcherServlet(servletAppContext);
    //     // ds.setDetectAllHandlerAdapters(false); // 예시: DispatcherServlet의 특정 프로퍼티 변경
    //     return ds;
    // }
    ```


### 4. 왜 Java 기반 설정을 사용하는가? (vs `web.xml`)

- **타입 안전성(Type Safety):** Java 코드로 작성되므로 컴파일 시점에 오류를 잡을 수 있습니다. XML은 오타가 있어도 런타임에 발견되는 경우가 많습니다.
- **리팩토링 용이성:** 클래스 이름이나 메소드 이름 변경 시 IDE의 리팩토링 기능을 활용할 수 있습니다.
- **더 나은 가독성 및 유지보수성 (주관적일 수 있음):** 복잡한 XML 구조보다 Java 코드가 더 익숙한 개발자에게는 가독성이 높을 수 있습니다.
- **XML 제거:** 프로젝트에서 XML 설정 파일을 완전히 제거하고 Java 코드로만 관리할 수 있습니다.
- **Spring Boot와의 일관성:** Spring Boot는 기본적으로 Java 기반 설정을 사용하므로, 전체적인 설정 방식에 일관성을 가져갈 수 있습니다.

**주의:** Spring Boot를 사용한다면, `WebApplicationInitializer`를 직접 구현할 필요가 거의 없습니다. Spring Boot는 `spring-boot-starter-web` 의존성이 있으면 `DispatcherServlet`과 기본적인 필터들을 자동으로 등록하고 설정해주기 때문입니다. 이 챕터의 내용은 주로 Spring Boot 없이 순수 Spring MVC를 사용하거나, Spring Boot 환경에서도 매우 세부적인 서블릿 컨테이너 레벨의 설정을 하고 싶을 때 유용합니다.

---

- **Servlet Config (`WebApplicationInitializer` 계열):**
  - **대상:** **서블릿 컨테이너 수준**의 설정. `DispatcherServlet` 자체를 서블릿 컨테이너(예: Tomcat)에 **등록하고 매핑**하는 역할, 필터 등록, 리스너 등록 등.
  - **역할:** Spring MVC 애플리케이션이 웹 환경에서 **동작하기 위한 가장 기본적인 틀**을 마련합니다. `web.xml`이 하던 역할을 Java 코드로 대체하는 것입니다.
  - **주요 인터페이스/클래스:** `WebApplicationInitializer`, `AbstractDispatcherServletInitializer`, `AbstractAnnotationConfigDispatcherServletInitializer`.
- **Web MVC Config (`WebMvcConfigurer` 계열):**
  - **대상:** 이미 등록된 **`DispatcherServlet` 내부의 동작 방식**을 설정. `DispatcherServlet`이 사용하는 특별한 빈들(HandlerMapping, HandlerAdapter, ViewResolver 등)의 **세부 동작을 커스터마이징**하거나, 인터셉터, 포맷터, 정적 리소스 처리 등을 설정합니다.
  - **역할:** `DispatcherServlet`이 요청을 **어떻게 처리하고 응답을 생성할지**에 대한 세부 규칙과 컴포넌트를 설정합니다.
  - **주요 인터페이스/클래스:** `WebMvcConfigurer`, `@EnableWebMvc`.

이 둘의 관계를 집 짓는 과정에 비유해볼게요:

**1. Servlet Config (집의 골격과 주소 설정)**

- `WebApplicationInitializer` (또는 그 하위 클래스들)은 마치 **집을 짓기 위한 기초 공사를 하고, 그 집에 주소(`DispatcherServlet`의 URL 매핑)를 부여하는 과정**과 같습니다.
- **하는 일:**
  - "우리 동네(서블릿 컨테이너)에 'Spring MVC 레스토랑(`DispatcherServlet`)'이라는 가게를 열겠다!" 라고 신고하고 등록합니다. (`servletContext.addServlet(...)`)
  - "이 레스토랑은 '모든 손님(/)'을 받겠다!" 라고 간판을 답니다. (`registration.addMapping("/")`)
  - 레스토랑의 총괄 매니저(`DispatcherServlet`)에게 어떤 메뉴판(Spring 설정 파일 - `AppConfig.class` 또는 `dispatcher-config.xml`)을 보고 일해야 하는지 알려줍니다.
  - 레스토랑 입구에 보안 요원(Filter)을 배치합니다. (`getServletFilters()`)
- **결과:** `DispatcherServlet`이라는 서블릿이 서블릿 컨테이너에 성공적으로 등록되고, 특정 URL 요청을 받을 준비가 완료됩니다. **하지만 아직 `DispatcherServlet`이 내부적으로 어떻게 요청을 처리할지에 대한 세부 설정은 되어있지 않습니다.**

**2. Web MVC Config (레스토랑 내부 인테리어 및 운영 규칙 설정)**

- `WebMvcConfigurer` (또는 XML의 `<mvc:...>` 태그들)는 이미 문을 연 'Spring MVC 레스토랑(`DispatcherServlet`)'의 **내부 인테리어를 꾸미고, 주방 시스템을 정비하며, 손님 응대 매뉴얼을 만드는 과정**과 같습니다.
- **하는 일:**
  - "손님이 '메인 요리 보여줘(URL 요청)'라고 하면, 어떤 주방장(Controller)에게 전달할지 결정하는 규칙(HandlerMapping)을 이렇게 설정하겠다." (실제로는 `@EnableWebMvc`가 기본 `HandlerMapping`을 제공하고, 필요시 `WebMvcConfigurer`로 커스텀)
  - "주방장이 요리를 다 만들면(Controller가 결과 반환), 어떤 그릇(View)에 담아낼지 결정하는 방법(ViewResolver)은 이걸로 하겠다." (`viewResolver()` 빈 등록)
  - "손님이 특정 음료를 주문하면(특정 요청 경로), 중간에 특별한 서비스를 제공하겠다(Interceptor)." (`addInterceptors()`)
  - "레스토랑 한쪽에는 셀프바(정적 리소스)를 마련하겠다." (`addResourceHandlers()`)
- **결과:** `DispatcherServlet`이 실제로 요청을 받았을 때, 그 요청을 효율적으로 처리하고 적절한 응답을 생성하기 위한 내부 컴포넌트들과 동작 방식이 구체적으로 설정됩니다.

**표로 비교해볼까요?**

| 구분 | Servlet Config (`WebApplicationInitializer` 등) | Web MVC Config (`WebMvcConfigurer` 등) |
| --- | --- | --- |
| **주요 목적** | `DispatcherServlet` 자체를 **서블릿 컨테이너에 등록** 및 초기 설정. `web.xml` 대체. | `DispatcherServlet` **내부의 요청 처리 방식** 및 컴포넌트 설정. |
| **설정 수준** | **서블릿 컨테이너 수준** (Servlet, Filter, Listener 등록 및 매핑) | **Spring MVC 프레임워크 수준** (`DispatcherServlet`의 특별한 빈들, 인터셉터, 뷰 리졸루션 등) |
| **핵심 역할** | Spring MVC 애플리케이션의 **진입점(Entry Point) 마련**. | `DispatcherServlet`의 **동작 커스터마이징** 및 확장. |
| **설정 대상** | `DispatcherServlet` 인스턴스, `Filter` 인스턴스, `ServletContext` 등. | `HandlerMapping`, `HandlerAdapter`, `ViewResolver`, `Interceptor`, `Formatter`, `HttpMessageConverter` 등. |
| **주요 도구** | `WebApplicationInitializer`, `AbstractDispatcherServletInitializer` 계열 클래스들. | `WebMvcConfigurer` 인터페이스, `@EnableWebMvc` 어노테이션, `<mvc:...>` XML 네임스페이스. |
| **예시** | `registration.addMapping("/");`, `getServletConfigClasses()` 반환. | `addInterceptors()`, `viewResolver()` 빈 정의, `addResourceHandlers()`. |
- **Servlet Config**에서는 `DispatcherServlet`이 사용할 **`WebApplicationContext` 인스턴스를 생성**하고, 그 컨텍스트가 어떤 설정 파일(Java Config 클래스 또는 XML 파일)을 읽어야 하는지 알려주는 역할을 합니다. (`getRootConfigClasses()`, `getServletConfigClasses()` 또는 `createServletApplicationContext()`). 이 컨텍스트가 바로 `DispatcherServlet`이 자신의 특별한 빈들을 찾아올 "곳"이 됩니다.
- **Web MVC Config**는 바로 그 **`WebApplicationContext` 안에 어떤 빈들을 채워 넣을 것인가, 그리고 그 빈들의 동작을 어떻게 조정할 것인가**에 대한 구체적인 내용을 정의합니다.

**흐름으로 보면:**

1. **서블릿 컨테이너 시작**
2. `WebApplicationInitializer` 구현체의 `onStartup` 메소드 (또는 `AbstractDispatcherServletInitializer`의 내부 로직) 호출 (**Servlet Config 단계**)
  - 루트 `WebApplicationContext` 생성 (만약 있다면)
  - `DispatcherServlet`용 `WebApplicationContext` 생성 (이때 `getServletConfigClasses()`에서 반환된 `MyWebConfig.class` 같은 클래스를 사용)
  - `DispatcherServlet` 인스턴스 생성 및 서블릿 컨테이너에 등록, URL 매핑
3. `DispatcherServlet` 초기화
  - 자신의 `WebApplicationContext` (2번에서 생성된) 안에서 특별한 빈들을 찾음.
  - 이때 `MyWebConfig.class` ( **Web MVC Config 단계**에서 정의된 내용)에 따라 `ViewResolver` 빈, `Interceptor` 설정 등이 로드되고 적용됨.

결국, **Servlet Config는 `DispatcherServlet`이라는 배우를 무대에 올리고 어떤 대본(`ApplicationContext`와 그 설정 파일)을 줄지 결정하는 역할**이라면, **Web MVC Config는 그 대본의 구체적인 내용(어떤 장면, 어떤 소품, 어떤 특수효과)을 작성하는 역할**이라고 볼 수 있습니다.
