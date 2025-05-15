---
title: Spring Web MVC - Dispatcher Servlet (Web MVC Config)
description: 
author: laze
date: 2025-05-15 00:00:01 +0900
categories: [Dev, SpringWebMVC]
tags: [SpringWebMVC]
---
**웹 MVC 설정 (Web MVC Config)**

애플리케이션은 요청을 처리하는 데 필요한 특별한 빈 타입(Special Bean Types)에 나열된 인프라 빈(infrastructure beans)들을 선언할 수 있습니다.

`DispatcherServlet`은 각 특별한 빈에 대해 `WebApplicationContext`를 확인합니다.

만약 일치하는 빈 타입이 없다면, `DispatcherServlet.properties`에 나열된 기본 타입들을 사용합니다.

대부분의 경우, MVC 설정(MVC Config)이 가장 좋은 시작점입니다.

이는 Java 또는 XML로 필요한 빈들을 선언하고, 이를 사용자 정의하기 위한 더 높은 수준의 설정 콜백 API를 제공합니다.

Spring Boot는 Spring MVC를 설정하기 위해 MVC Java 설정에 의존하며, 많은 추가적인 편리한 옵션들을 제공합니다.

---

## Spring Web MVC 설정의 비밀: DispatcherServlet.properties와 MVC Config

이전 챕터에서 `DispatcherServlet`이 요청 처리를 위해 다양한 "특별한 빈(Special Beans)" (`HandlerMapping`, `HandlerAdapter` 등)을 사용한다고 배웠습니다.

이번 챕터에서는 Spring MVC가 이러한 특별한 빈들을 **어떻게 찾고, 만약 우리가 직접 설정하지 않으면 어떤 기본 빈들을 사용하는지**,

그리고 **우리가 이러한 빈들을 어떻게 효과적으로 설정할 수 있는지**에 대해 알아봅니다.

### 1. DispatcherServlet의 "특별한 빈" 탐색 과정

`DispatcherServlet`이 초기화될 때, 그리고 요청을 처리할 때마다 다음과 같은 순서로 필요한 특별한 빈들을 찾습니다:

1. **`WebApplicationContext`에서 먼저 찾는다:**
  - `DispatcherServlet`은 자신이 사용하는 `WebApplicationContext` (우리가 `RootConfig`나 `AppConfig` 등으로 설정한 Spring 컨테이너)에 해당 인터페이스(예: `HandlerMapping`)를 구현한 빈이 등록되어 있는지 확인합니다.
  - **만약 개발자가 명시적으로 해당 타입의 빈을 설정해두었다면 (예: `@Bean`으로 `HandlerMapping` 구현체를 등록), `DispatcherServlet`은 그 빈을 사용합니다.** 이것이 바로 우리가 설정을 커스터마이징하는 방법입니다.
2. **일치하는 빈이 없다면? -> `DispatcherServlet.properties`의 기본값 사용!**
  - 만약 `WebApplicationContext`에서 해당 타입의 빈을 찾지 못하면, `DispatcherServlet`은 **내부적으로 가지고 있는 `DispatcherServlet.properties` 파일**을 참조합니다.
  - `DispatcherServlet.properties` 이 파일에는 각 특별한 빈 인터페이스에 대한 **기본 구현 클래스(default implementation classes)**들이 정의되어 있습니다.
  - `DispatcherServlet`은 이 프로퍼티 파일에 명시된 기본 클래스의 인스턴스를 생성하여 사용합니다.

**`DispatcherServlet.properties` 내용 분석:**

```xml
# Default implementation classes for DispatcherServlet's strategy interfaces.
# Used as fallback when no matching beans are found in the DispatcherServlet context.
# Not meant to be customized by application developers.

# LocaleResolver의 기본 구현체는 AcceptHeaderLocaleResolver 이다.
org.springframework.web.servlet.LocaleResolver=org.springframework.web.servlet.i18n.AcceptHeaderLocaleResolver

# HandlerMapping의 기본 구현체는 BeanNameUrlHandlerMapping, RequestMappingHandlerMapping, RouterFunctionMapping 순서로 사용된다.
#  실제로는 RequestMappingHandlerMapping이 어노테이션 기반에서 주로 활성화됩니다.)
org.springframework.web.servlet.HandlerMapping=org.springframework.web.servlet.handler.BeanNameUrlHandlerMapping,\\
org.springframework.web.servlet.mvc.method.annotation.RequestMappingHandlerMapping,\\
org.springframework.web.servlet.function.support.RouterFunctionMapping

# HandlerAdapter의 기본 구현체는 HttpRequestHandlerAdapter, SimpleControllerHandlerAdapter, RequestMappingHandlerAdapter, HandlerFunctionAdapter 순서로 사용된다.
# (마찬가지로 RequestMappingHandlerAdapter가 어노테이션 기반에서 주로 활성화됩니다.)
org.springframework.web.servlet.HandlerAdapter=org.springframework.web.servlet.mvc.HttpRequestHandlerAdapter,\\
org.springframework.web.servlet.mvc.SimpleControllerHandlerAdapter,\\
org.springframework.web.servlet.mvc.method.annotation.RequestMappingHandlerAdapter,\\
org.springframework.web.servlet.function.support.HandlerFunctionAdapter

# HandlerExceptionResolver의 기본 구현체들
org.springframework.web.servlet.HandlerExceptionResolver=org.springframework.web.servlet.mvc.method.annotation.ExceptionHandlerExceptionResolver,\\
org.springframework.web.servlet.mvc.annotation.ResponseStatusExceptionResolver,\\
org.springframework.web.servlet.mvc.support.DefaultHandlerExceptionResolver

# RequestToViewNameTranslator의 기본 구현체 (요청 URL로부터 뷰 이름을 추론)
org.springframework.web.servlet.RequestToViewNameTranslator=org.springframework.web.servlet.view.DefaultRequestToViewNameTranslator

# ViewResolver의 기본 구현체는 InternalResourceViewResolver 이다.
org.springframework.web.servlet.ViewResolver=org.springframework.web.servlet.view.InternalResourceViewResolver

# FlashMapManager의 기본 구현체는 SessionFlashMapManager 이다.
org.springframework.web.servlet.FlashMapManager=org.springframework.web.servlet.support.SessionFlashMapManager
```

- **주석의 의미:**
  - `Default implementation classes for DispatcherServlet's strategy interfaces.`
    -> `DispatcherServlet`의 전략 인터페이스(특별한 빈들)에 대한 기본 구현 클래스들이다.
  - `Used as fallback when no matching beans are found in the DispatcherServlet context.`
    -> `DispatcherServlet` 컨텍스트(즉, `WebApplicationContext`)에서 일치하는 빈을 찾지 못했을 때 **대안으로 사용된다.**
  - `Not meant to be customized by application developers.`
    -> 이 `DispatcherServlet.properties` 파일 자체를 개발자가 직접 수정하는 것은 권장되지 않는다. (커스터마이징은 `WebApplicationContext`에 빈을 등록하는 방식으로 하라는 의미)
- **핵심:** 우리가 `@Controller`, `@GetMapping`, `@PathVariable` 등을 사용했을 때 `RequestMappingHandlerMapping`이나 `RequestMappingHandlerAdapter`가 "알아서" 동작했던 이유 중 하나가 바로 여기에 있습니다! 우리가 별도로 설정하지 않으면 Spring MVC는 이 `DispatcherServlet.properties`에 정의된 기본 구현체들을 사용하기 때문입니다. (정확히는 `@EnableWebMvc`나 Spring Boot 자동 설정이 이 목록에 있는 빈들을 활성화하고 기본값으로 등록해주는 역할을 합니다.)

### 2. MVC Config: 더 쉽고 효율적인 설정 방법

`DispatcherServlet.properties` 파일은 Spring MVC의 내부 동작 방식을 이해하는 데는 중요하지만, 우리가 직접 이 파일을 건드려서 설정을 바꾸지는 않습니다.

대신 Spring은 더 편리하고 높은 수준의 설정 방법을 제공하는데, 이것이 바로 **"MVC Config"**입니다.

"MVC Config"는 크게 두 가지 방식으로 제공됩니다:

- **Java 기반 설정 (권장):**
  - `@Configuration` 어노테이션이 붙은 클래스에서 `WebMvcConfigurer` 인터페이스를 구현하여 사용합니다.
  - `WebMvcConfigurer` 인터페이스는 다양한 `addXXX()` 또는 `configureXXX()` 형태의 콜백 메소드를 제공하여, 개발자가 Spring MVC의 여러 컴포넌트(인터셉터, 뷰 리졸버, 정적 리소스 핸들러, 메시지 컨버터 등)를 쉽게 커스터마이징하거나 추가할 수 있도록 합니다.
  - `@EnableWebMvc` 어노테이션과 함께 사용되는 경우가 많습니다. `@EnableWebMvc`는 기본적인 Spring MVC 설정을 활성화하고, `WebMvcConfigurer`를 통해 개발자가 세부 사항을 조정할 수 있도록 합니다.

  **예시 (Java Config):**

    ```java
    import org.springframework.context.annotation.Bean;
    import org.springframework.context.annotation.ComponentScan;
    import org.springframework.context.annotation.Configuration;
    import org.springframework.web.servlet.ViewResolver;
    import org.springframework.web.servlet.config.annotation.EnableWebMvc;
    import org.springframework.web.servlet.config.annotation.InterceptorRegistry;
    import org.springframework.web.servlet.config.annotation.ResourceHandlerRegistry;
    import org.springframework.web.servlet.config.annotation.WebMvcConfigurer;
    import org.springframework.web.servlet.view.InternalResourceViewResolver;
    
    @Configuration
    @EnableWebMvc // Spring MVC 기본 설정 활성화 + WebMvcConfigurer 사용 가능하게 함
    @ComponentScan(basePackages = "com.example.controller")
    public class WebConfig implements WebMvcConfigurer { // WebMvcConfigurer 인터페이스 구현
    
        // 1. ViewResolver 커스터마이징 (기본 InternalResourceViewResolver 대신 다른 설정을 사용하거나, 다른 ViewResolver 추가)
        @Bean
        public ViewResolver viewResolver() {
            InternalResourceViewResolver resolver = new InternalResourceViewResolver();
            resolver.setPrefix("/WEB-INF/views/"); // 뷰 파일 경로의 접두사
            resolver.setSuffix(".jsp");           // 뷰 파일 경로의 접미사
            return resolver;
        }
    
        // 2. 인터셉터 추가
        @Override
        public void addInterceptors(InterceptorRegistry registry) {
            registry.addInterceptor(new MyCustomInterceptor()) // 직접 만든 인터셉터
                    .addPathPatterns("/secure/*");        // "/secure/"로 시작하는 경로에만 적용
        }
    
        // 3. 정적 리소스 핸들러 설정 (예: CSS, JS, 이미지 파일)
        @Override
        public void addResourceHandlers(ResourceHandlerRegistry registry) {
            registry.addResourceHandler("/resources/**")      // "/resources/"로 시작하는 URL 요청은
                    .addResourceLocations("/public-resources/"); // "/public-resources/" 폴더에서 찾는다.
        }
    
        // 기타 HttpMessageConverter 설정, Formatter 설정 등 다양한 콜백 메소드 오버라이드 가능
        // @Override
        // public void configureMessageConverters(List<HttpMessageConverter<?>> converters) { ... }
    }
    
    ```

- **XML 기반 설정 (과거 방식):**
  - `<mvc:annotation-driven />` 태그를 사용하여 기본적인 어노테이션 기반 MVC 설정을 활성화합니다.
  - `<mvc:interceptors>`, `<mvc:resources>`, `<mvc:view-resolvers>` 같은 전용 네임스페이스 태그들을 사용하여 MVC 컴포넌트를 설정합니다.

  **예시 (XML Config):**

    ```xml
    <!-- dispatcher-servlet.xml 또는 mvc-config.xml -->
    <beans xmlns="<http://www.springframework.org/schema/beans>"
           xmlns:mvc="<http://www.springframework.org/schema/mvc>"
           xmlns:context="<http://www.springframework.org/schema/context>"
           xsi:schemaLocation="
               <http://www.springframework.org/schema/beans> <http://www.springframework.org/schema/beans/spring-beans.xsd>
               <http://www.springframework.org/schema/mvc> <http://www.springframework.org/schema/mvc/spring-mvc.xsd>
               <http://www.springframework.org/schema/context> <http://www.springframework.org/schema/context/spring-context.xsd>">
    
        <!-- 1. @Controller, @Service 등 어노테이션 스캔 -->
        <context:component-scan base-package="com.example.controller" />
    
        <!-- 2. 기본적인 어노테이션 기반 MVC 설정 활성화 (RequestMappingHandlerMapping, RequestMappingHandlerAdapter 등 등록) -->
        <mvc:annotation-driven />
    
        <!-- 3. ViewResolver 커스터마이징 -->
        <bean id="viewResolver" class="org.springframework.web.servlet.view.InternalResourceViewResolver">
            <property name="prefix" value="/WEB-INF/views/" />
            <property name="suffix" value=".jsp" />
        </bean>
    
        <!-- 4. 인터셉터 추가 -->
        <mvc:interceptors>
            <mvc:interceptor>
                <mvc:mapping path="/secure/*" />
                <bean class="com.example.interceptor.MyCustomInterceptor" />
            </mvc:interceptor>
        </mvc:interceptors>
    
        <!-- 5. 정적 리소스 핸들러 설정 -->
        <mvc:resources mapping="/resources/**" location="/public-resources/" />
    
    </beans>
    
    ```


**왜 MVC Config를 사용하는가?**

- **단순함:** `DispatcherServlet.properties`를 직접 건드리거나, 모든 특별한 빈들을 일일이 `@Bean`으로 등록하는 것보다 훨씬 간결하고 직관적입니다.
- **높은 수준의 API:** `WebMvcConfigurer`의 콜백 메소드들은 "무엇을 하고 싶은지"에 초점을 맞춘 API를 제공합니다. (예: "인터셉터를 추가하고 싶다", "뷰 리졸버를 설정하고 싶다")
- **모범 사례 통합:** `@EnableWebMvc`나 `<mvc:annotation-driven />`은 Spring 팀이 권장하는 많은 기본 설정들을 자동으로 포함하고 있어, 개발자가 세세한 부분까지 신경 쓰지 않아도 됩니다.

### 3. Spring Boot와 MVC Config

Spring Boot는 이러한 Java 기반 MVC Config를 더욱 발전시켜 사용합니다.

- **자동 설정의 확장:** `spring-boot-starter-web` 의존성이 있으면, Spring Boot는 `@EnableWebMvc`가 해주는 기본 설정에 더해, 훨씬 더 많은 편리한 자동 설정을 제공합니다. (예: 내장 톰캣 설정, 기본 에러 페이지, `JacksonHttpMessageConverter` 자동 등록 등)
- **`application.properties` 또는 `application.yml`을 통한 간편 설정:** 많은 MVC 관련 설정을 Java 코드 변경 없이 프로퍼티 파일에서 쉽게 변경할 수 있습니다. (예: `spring.mvc.view.prefix=/WEB-INF/jsp/`)
- **`WebMvcConfigurer` 여전히 유효:** Spring Boot 환경에서도 `WebMvcConfigurer`를 사용하여 세부적인 MVC 설정을 커스터마이징할 수 있습니다. Spring Boot는 개발자가 정의한 `WebMvcConfigurer`를 존중하여 자동 설정과 함께 적용합니다.

**결론:**

1. `DispatcherServlet`은 필요한 특별한 빈을 `WebApplicationContext`에서 먼저 찾습니다.
2. 없으면 `DispatcherServlet.properties`에 정의된 기본 구현체를 사용합니다 (하지만 이 파일을 직접 수정하진 않습니다).
3. 개발자는 **MVC Config (주로 Java 기반 `WebMvcConfigurer` 또는 XML 기반 `<mvc: ... />` 태그)**를 사용하여 이러한 빈들을 효과적으로 설정하고 커스터마이징합니다.
4. Spring Boot는 이 MVC Config를 기반으로 더욱 강력하고 편리한 자동 설정을 제공합니다.
