---
title: Spring Web MVC - Annotated Controllers (Declaration)
description: 
author: laze
date: 2025-05-22 00:00:01 +0900
categories: [Dev, SpringWebMVC]
tags: [SpringWebMVC]
---
**선언 (Declaration)**

서블릿의 `WebApplicationContext`에 표준 Spring 빈 정의를 사용하여 컨트롤러 빈을 정의할 수 있습니다.

`@Controller` 스테레오타입은 클래스패스에서 `@Component` 클래스를 감지하고 이에 대한 빈 정의를 자동 등록하는 Spring의 일반적인 지원과 마찬가지로 자동 감지를 허용합니다.

또한 어노테이션이 달린 클래스에 대한 스테레오타입으로 작용하여 웹 컴포넌트로서의 역할을 나타냅니다.

이러한 `@Controller` 빈의 자동 감지를 활성화하려면, 다음 예제와 같이 Java 설정에 컴포넌트 스캐닝을 추가할 수 있습니다:

**Java**

```java
@Configuration
@ComponentScan("org.example.web")
public class WebConfiguration {

	// ...
}
```

`@RestController`는 `@Controller`와 `@ResponseBody`로 메타 어노테이트된 합성 어노테이션(composed annotation)으로,

모든 메소드가 타입 레벨의 `@ResponseBody` 어노테이션을 상속받아 HTML 템플릿을 사용한 뷰 리졸루션 및 렌더링 대신 응답 본문에 직접 쓰는 컨트롤러임을 나타냅니다.

**AOP 프록시 (AOP Proxies)**

경우에 따라 런타임에 컨트롤러를 AOP 프록시로 꾸며야 할 수도 있습니다.

한 가지 예는 컨트롤러에 직접 `@Transactional` 어노테이션을 사용하기로 선택한 경우입니다.

이러한 경우, 특히 컨트롤러에 대해서는 클래스 기반 프록시를 사용하는 것이 좋습니다.

컨트롤러에 직접 이러한 어노테이션을 사용하면 자동으로 그렇게 됩니다.

만약 컨트롤러가 인터페이스를 구현하고 AOP 프록시가 필요한 경우, 명시적으로 클래스 기반 프록시를 설정해야 할 수 있습니다.

예를 들어, `@EnableTransactionManagement`의 경우 `@EnableTransactionManagement(proxyTargetClass = true)`로 변경하고,

`<tx:annotation-driven/>`의 경우 `<tx:annotation-driven proxy-target-class="true"/>`로 변경할 수 있습니다.

6.0 버전부터 인터페이스 프록시를 사용하는 경우, Spring MVC는 더 이상 인터페이스의 타입 레벨 `@RequestMapping` 어노테이션만으로는 컨트롤러를 감지하지 않는다는 점을 유의하십시오.

클래스 기반 프록시를 활성화하거나, 그렇지 않으면 인터페이스에도 `@Controller` 어노테이션이 있어야 합니다.

---

## Spring MVC 컨트롤러 선언하기: `@Controller`와 컴포넌트 스캔

Spring MVC에서 웹 요청을 처리하는 로직을 담는 클래스를 만들 때는 주로 **`@Controller`** 어노테이션을 사용합니다.

### 1. `@Controller` 스테레오타입 어노테이션

- **역할:**
  1. 해당 클래스가 **웹 컨트롤러 역할**을 수행함을 나타내는 표식(stereotype)입니다.
  2. Spring의 **컴포넌트 스캔(Component Scan)** 대상이 되어, 자동으로 감지되고 Spring 애플리케이션 컨텍스트에 **빈(Bean)으로 등록**됩니다. (내부적으로 `@Component` 어노테이션을 포함하고 있습니다.)
- **특징:**
  - 특별한 기본 클래스를 상속받거나 인터페이스를 구현할 필요가 없습니다. 일반적인 POJO(Plain Old Java Object) 클래스에 이 어노테이션만 붙이면 됩니다.
  - 메소드 시그니처가 매우 유연하여, 다양한 타입의 파라미터를 받고 다양한 타입의 값을 반환할 수 있습니다. (이는 다음 챕터들에서 자세히 다룹니다.)

### 2. 컴포넌트 스캔 (Component Scan) 활성화

`@Controller` 어노테이션이 붙은 클래스를 Spring이 자동으로 감지하고 빈으로 등록하게 하려면, Spring 설정에서 **컴포넌트 스캔**을 활성화해야 합니다.

- **Java Config 방식:** `@Configuration` 어노테이션이 붙은 설정 클래스에 `@ComponentScan` 어노테이션을 사용합니다.

    ```java
    // WebConfig.java (또는 AppConfig.java 등)
    import org.springframework.context.annotation.ComponentScan;
    import org.springframework.context.annotation.Configuration;
    import org.springframework.web.servlet.config.annotation.EnableWebMvc;
    
    @Configuration
    @EnableWebMvc // Spring MVC 기능 활성화 (HandlerMapping, HandlerAdapter 등 등록)
    @ComponentScan(basePackages = "com.example.web.controller") // 이 패키지 하위를 스캔하여 @Controller 등을 찾음
    // 또는 @ComponentScan("com.example.web") 과 같이 상위 패키지를 지정할 수도 있습니다.
    // 여러 패키지를 지정하려면: @ComponentScan(basePackages = {"com.example.web.controller", "com.example.another.package"})
    public class WebConfig {
        // ... (ViewResolver, Interceptor 등 다른 설정)
    }
    
    ```

  - `@ComponentScan`의 `basePackages` 속성에 지정된 패키지(와 그 하위 패키지들) 내에서 `@Controller`, `@Service`, `@Repository`, `@Component` 등의 스테레오타입 어노테이션이 붙은 클래스들을 찾아 빈으로 등록합니다.
- **XML Config 방식:** `<context:component-scan>` 태그를 사용합니다.

    ```xml
    <!-- dispatcher-servlet.xml 또는 applicationContext.xml -->
    <beans xmlns="<http://www.springframework.org/schema/beans>"
           xmlns:context="<http://www.springframework.org/schema/context>"
           xmlns:mvc="<http://www.springframework.org/schema/mvc>"
           xsi:schemaLocation="
               <http://www.springframework.org/schema/beans> <http://www.springframework.org/schema/beans/spring-beans.xsd>
               <http://www.springframework.org/schema/context> <http://www.springframework.org/schema/context/spring-context.xsd>
               <http://www.springframework.org/schema/mvc> <http://www.springframework.org/schema/mvc/spring-mvc.xsd>">
    
        <!-- Spring MVC 어노테이션 기반 기능 활성화 -->
        <mvc:annotation-driven />
    
        <!-- 컴포넌트 스캔 설정 -->
        <context:component-scan base-package="com.example.web.controller" />
    
        <!-- ... (다른 빈 정의) -->
    </beans>
    ```


**컴포넌트 스캔이 활성화되면,** `DispatcherServlet`이 사용하는 `WebApplicationContext` 내에 `@Controller`가 붙은 빈들이 등록되고,

`HandlerMapping`은 이 빈들 중에서 `@RequestMapping` (또는 `@GetMapping`, `@PostMapping` 등) 어노테이션이 붙은 메소드를 찾아 URL과 매핑하게 됩니다.

### 3. `@RestController` 어노테이션

`@RestController`는 RESTful 웹 서비스를 만들 때 자주 사용되는 편리한 어노테이션입니다.

- **합성 어노테이션 (Composed Annotation):** `@RestController`는 다음과 같은 두 어노테이션을 합쳐놓은 것입니다:
  - **`@Controller`**: 해당 클래스가 컨트롤러임을 나타내고, 컴포넌트 스캔 대상이 되도록 합니다.
  - **`@ResponseBody`**: 이 어노테이션이 클래스 레벨에 붙으면, 해당 컨트롤러의 **모든 핸들러 메소드**가 반환하는 값은 뷰(View)를 통해 렌더링되는 것이 아니라, **HTTP 응답 본문(body)에 직접 쓰여지게 됩니다.** (주로 JSON, XML, 또는 단순 문자열 데이터 반환 시 사용)
- **사용 예시:**

    ```java
    // ItemController.java
    import org.springframework.web.bind.annotation.GetMapping;
    import org.springframework.web.bind.annotation.PathVariable;
    import org.springframework.web.bind.annotation.PostMapping;
    import org.springframework.web.bind.annotation.RequestBody;
    import org.springframework.web.bind.annotation.RestController;
    import java.util.List;
    import java.util.ArrayList;
    
    @RestController // @Controller + @ResponseBody
    @RequestMapping("/api/items")
    public class ItemController {
    
        private List<String> items = new ArrayList<>();
    
        @GetMapping
        public List<String> getAllItems() {
            // 이 List<String> 객체는 HttpMessageConverter에 의해 JSON 배열 등으로 변환되어 응답 본문에 쓰여짐
            return items;
        }
    
        @PostMapping
        public String addItem(@RequestBody String item) {
            // @RequestBody: 요청 본문의 데이터를 item 파라미터로 받음 (예: JSON 문자열)
            // 반환된 String "Item added"는 HttpMessageConverter에 의해 text/plain 등으로 응답 본문에 쓰여짐
            items.add(item);
            return "Item added: " + item;
        }
    
        @GetMapping("/{index}")
        public String getItem(@PathVariable int index) {
            if (index >= 0 && index < items.size()) {
                return items.get(index);
            }
            return "Item not found"; // 이 문자열도 응답 본문에 직접 쓰여짐
        }
    }
    ```

- **`@Controller`와의 차이점:**
  - `@Controller`: 메소드가 주로 **뷰 이름(String)이나 `ModelAndView` 객체**를 반환하여, 뷰 리졸버를 통해 HTML 페이지 등을 렌더링합니다. 응답 본문에 직접 데이터를 쓰려면 메소드 레벨에 `@ResponseBody`를 추가해야 합니다.
  - `@RestController`: 모든 메소드가 기본적으로 `@ResponseBody` 효과를 가지므로, 반환 값이 바로 응답 본문이 됩니다. REST API에서 JSON이나 XML 데이터를 주고받을 때 매우 편리합니다.

### 4. AOP 프록시 (AOP Proxies)와 컨트롤러

- *AOP(Aspect-Oriented Programming, 관점 지향 프로그래밍)**는 로깅, 트랜잭션 관리, 보안 등 여러 모듈에 걸쳐 공통적으로 나타나는 부가 기능(횡단 관심사, cross-cutting concerns)을 분리하여 관리하는 프로그래밍 패러다임입니다. Spring은 AOP를 강력하게 지원하며, 주로 **프록시(Proxy)** 기반으로 동작합니다.

**컨트롤러에 AOP 프록시가 필요한 경우:**

- **`@Transactional` 어노테이션 사용:** 컨트롤러 메소드에 직접 트랜잭션 관리를 위한 `@Transactional` 어노테이션을 붙이는 경우. (일반적으로 트랜잭션은 서비스 계층에서 관리하는 것이 더 권장되지만, 경우에 따라 컨트롤러에 사용할 수도 있습니다.)
- 기타 커스텀 AOP Advice를 컨트롤러에 적용하는 경우.

**프록시 생성 방식과 주의사항:**

Spring AOP는 기본적으로 두 가지 방식의 프록시를 생성할 수 있습니다:

1. **JDK 동적 프록시 (JDK Dynamic Proxy):**
  - **인터페이스 기반**으로 프록시를 생성합니다. 즉, 대상 객체(여기서는 컨트롤러)가 인터페이스를 구현하고 있어야 합니다.
  - 프록시 객체는 해당 인터페이스 타입으로 캐스팅될 수 있습니다.
2. **CGLIB 프록시 (CGLIB Proxy):**
  - **클래스 기반**으로 프록시를 생성합니다. 대상 객체의 하위 클래스를 동적으로 만들어서 프록시로 사용합니다.
  - 인터페이스 구현 여부와 관계없이 적용 가능합니다. (final 클래스나 final 메소드에는 적용 불가)

**Spring MVC 컨트롤러와 AOP 프록시 권장 사항:**

- **컨트롤러에 AOP 프록시를 적용해야 할 경우 (예: `@Transactional`), 일반적으로 클래스 기반 프록시(CGLIB)를 사용하는 것이 권장됩니다.**
  - 이유: Spring MVC는 컨트롤러를 찾을 때 주로 구체적인 클래스 타입을 기준으로 동작하는 경우가 많습니다. JDK 동적 프록시를 사용하면 프록시 객체가 인터페이스 타입으로만 인식되어, Spring MVC가 컨트롤러의 어노테이션(예: `@RequestMapping`이 클래스 레벨에만 있는 경우)을 제대로 감지하지 못하는 문제가 발생할 수 있었습니다.
- **자동 설정:** 컨트롤러 클래스에 직접 `@Transactional`과 같은 AOP 관련 어노테이션을 붙이면, Spring은 보통 자동으로 클래스 기반 프록시를 생성하려고 시도합니다.
- **명시적 설정:**
  - 만약 컨트롤러가 인터페이스를 구현하고 있고, JDK 동적 프록시가 기본으로 생성되는데 클래스 기반 프록시가 필요하다면, AOP 설정을 명시적으로 변경해야 할 수 있습니다.
    - Java Config (`@EnableTransactionManagement`):

        ```java
        @Configuration
        @EnableTransactionManagement(proxyTargetClass = true) // 클래스 기반 프록시 사용 강제
        public class AppConfig { ... }
        ```

    - XML Config (`<tx:annotation-driven/>`):

        ```xml
        <tx:annotation-driven proxy-target-class="true"/> <!-- 클래스 기반 프록시 사용 강제 -->
        
        ```


**Spring Framework 6.0부터의 변경 사항:**

- 만약 인터페이스 기반 프록시(JDK 동적 프록시)를 사용하고, 컨트롤러의 `@RequestMapping` 어노테이션이 구현 클래스가 아닌 **인터페이스에만 정의되어 있다면, Spring MVC는 더 이상 이를 컨트롤러로 감지하지 않습니다.**
- **해결책:**
  1. **클래스 기반 프록시를 사용하도록 설정**합니다. (가장 권장)
  2. 또는, 해당 **인터페이스에도 `@Controller` 어노테이션을 추가**합니다.

**요약:** 컨트롤러는 일반적인 Spring 빈이므로 AOP 적용이 가능하지만, 프록시 생성 방식에 따라 Spring MVC의 컨트롤러 감지 메커니즘에 영향을 줄 수 있으므로 주의해야 합니다. 특히 인터페이스를 구현하는 컨트롤러에 AOP를 적용할 때는 클래스 기반 프록시 사용을 고려하는 것이 좋습니다.

---
