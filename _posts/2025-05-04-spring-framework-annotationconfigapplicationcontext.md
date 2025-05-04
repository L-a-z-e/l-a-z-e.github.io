---
title: Spring Framework AnnotationConfigApplicationContext
description: 
author: laze
date: 2025-05-04 00:00:06 +0900
categories: [Dev, SpringBoot]
tags: [SpringBoot]
---
**Instantiating the Spring Container by Using AnnotationConfigApplicationContext**

`ApplicationContext` 구현체인 `AnnotationConfigApplicationContext`는  `@Configuration` 클래스뿐만 아니라 일반 `@Component` 클래스 및 JSR-330 메타데이터로 어노테이션된 클래스도 입력으로 받을 수 있습니다.

`@Configuration` 클래스가 입력으로 제공될 때, `@Configuration` 클래스 자체는 빈 정의로 등록되고 클래스 내의 모든 선언된 `@Bean` 메소드도 빈 정의로 등록됩니다.

`@Component` 및 JSR-330 클래스가 제공될 때, 그것들은 빈 정의로 등록되며, 해당 클래스 내에서 필요한 경우 `@Autowired` 또는 `@Inject`와 같은 DI 메타데이터가 사용된다고 가정합니다.

**간단한 생성 (Simple Construction)**

`ClassPathXmlApplicationContext`를 인스턴스화할 때 스프링 XML 파일이 입력으로 사용되는 것과 거의 동일한 방식으로, `AnnotationConfigApplicationContext`를 인스턴스화할 때 `@Configuration` 클래스를 입력으로 사용할 수 있습니다. 이는 다음 예제와 같이 스프링 컨테이너의 완전한 XML 없는 사용을 가능하게 합니다:

```java
// Java
public static void main(String[] args) {
	ApplicationContext ctx = new AnnotationConfigApplicationContext(AppConfig.class);
	MyService myService = ctx.getBean(MyService.class);
	myService.doStuff();
}
```

```kotlin
// Kotlin
fun main(args: Array<String>) {
    val ctx = AnnotationConfigApplicationContext(AppConfig::class.java) // Use ::class.java
    val myService = ctx.getBean(MyService::class.java)
    myService.doStuff()
}
// Assuming AppConfig and MyService exist
```

`AnnotationConfigApplicationContext`는 `@Configuration` 클래스만 사용하는 데 국한되지 않습니다.

다음 예제와 같이 모든 `@Component` 또는 JSR-330 어노테이션된 클래스가 생성자에 입력으로 제공될 수 있습니다:

```java
// Java
public static void main(String[] args) {
	ApplicationContext ctx = new AnnotationConfigApplicationContext(MyServiceImpl.class, Dependency1.class, Dependency2.class);
	MyService myService = ctx.getBean(MyService.class);
	myService.doStuff();
}
```

```kotlin
// Kotlin
fun main(args: Array<String>) {
    val ctx = AnnotationConfigApplicationContext(MyServiceImpl::class.java, Dependency1::class.java, Dependency2::class.java)
    val myService = ctx.getBean(MyService::class.java)
    myService.doStuff()
}
// Assuming MyServiceImpl, Dependency1, Dependency2 exist
```

앞의 예제는 `MyServiceImpl`, `Dependency1`, `Dependency2`가 `@Autowired`와 같은 스프링 의존성 주입 어노테이션을 사용한다고 가정합니다.

**`register(Class<?>...)`를 사용하여 프로그래밍 방식으로 컨테이너 빌드하기 (Building the Container Programmatically by Using `register(Class<?>…)`)**

인수 없는 생성자를 사용하여 `AnnotationConfigApplicationContext`를 인스턴스화한 다음 `register()` 메소드를 사용하여 구성할 수 있습니다.

이 접근 방식은 프로그래밍 방식으로 `AnnotationConfigApplicationContext`를 빌드할 때 특히 유용합니다.

```java
// Java
public static void main(String[] args) {
	AnnotationConfigApplicationContext ctx = new AnnotationConfigApplicationContext();
	ctx.register(AppConfig.class, OtherConfig.class);
	ctx.register(AdditionalConfig.class);
	ctx.refresh(); // Important: Refresh the context after registration
	MyService myService = ctx.getBean(MyService.class);
	myService.doStuff();
}
```

```kotlin
// Kotlin
fun main(args: Array<String>) {
    val ctx = AnnotationConfigApplicationContext()
    ctx.register(AppConfig::class.java, OtherConfig::class.java)
    ctx.register(AdditionalConfig::class.java)
    ctx.refresh() // Important: Refresh the context after registration
    val myService = ctx.getBean(MyService::class.java)
    myService.doStuff()
}
// Assuming AppConfig, OtherConfig, AdditionalConfig, MyService exist
```

**`scan(String…)`으로 컴포넌트 스캐닝 활성화하기 (Enabling Component Scanning with `scan(String…)`)**

컴포넌트 스캐닝을 활성화하려면 다음과 같이 `@Configuration` 클래스에 어노테이션을 달 수 있습니다:

```java
// Java
@Configuration
@ComponentScan(basePackages = "com.acme") // ①
public class AppConfig  {
	// ...
}
```

① 이 어노테이션은 컴포넌트 스캐닝을 활성화합니다.

```kotlin
// Kotlin
@Configuration
@ComponentScan(basePackages = ["com.acme"]) // ① Use array syntax
class AppConfig {
    // ...
}
```

경험 많은 스프링 사용자는 다음 예제에 표시된 스프링의 `context:` 네임스페이스의 XML 선언 등가물에 익숙할 수 있습니다:

```xml
<beans>
	<context:component-scan base-package="com.acme"/>
</beans>
```

앞의 예제에서, `com.acme` 패키지는 `@Component`-어노테이션된 클래스를 찾기 위해 스캔되며, 해당 클래스는 컨테이너 내에서 스프링 빈 정의로 등록됩니다.

`AnnotationConfigApplicationContext`는 다음 예제와 같이 동일한 컴포넌트 스캐닝 기능을 허용하기 위해 `scan(String…)` 메소드를 노출합니다:

```java
// Java
public static void main(String[] args) {
	AnnotationConfigApplicationContext ctx = new AnnotationConfigApplicationContext();
	ctx.scan("com.acme");
	ctx.refresh(); // Important: Refresh after scan
	MyService myService = ctx.getBean(MyService.class);
}
```

```kotlin
// Kotlin
fun main(args: Array<String>) {
    val ctx = AnnotationConfigApplicationContext()
    ctx.scan("com.acme")
    ctx.refresh() // Important: Refresh after scan
    val myService = ctx.getBean(MyService::class.java)
    // ... use myService
}
```

`*@Configuration` 클래스는 `@Component`로 메타-어노테이션되므로 컴포넌트 스캐닝 후보라는 것을 기억하십시오.*

*앞의 예제에서 `AppConfig`가 `com.acme` 패키지(또는 그 아래의 모든 패키지) 내에 선언되었다고 가정하면, `scan()` 호출 중에 선택됩니다.*

`*refresh()` 시 모든 `@Bean` 메소드가 처리되고 컨테이너 내에서 빈 정의로 등록됩니다.*

**AnnotationConfigWebApplicationContext를 사용한 웹 애플리케이션 지원 (Support for Web Applications with AnnotationConfigWebApplicationContext)**

`AnnotationConfigApplicationContext`의 `WebApplicationContext` 변형은 `AnnotationConfigWebApplicationContext`와 함께 사용할 수 있습니다.

스프링 `ContextLoaderListener` 서블릿 리스너, 스프링 MVC `DispatcherServlet` 등을 구성할 때 이 구현을 사용할 수 있습니다.

다음 `web.xml` 스니펫은 일반적인 스프링 MVC 웹 애플리케이션을 구성합니다 (`contextClass` `context-param` 및 `init-param` 사용 주목):

```xml
<web-app>
	<!-- ContextLoaderListener가 기본 XmlWebApplicationContext 대신
		AnnotationConfigWebApplicationContext를 사용하도록 구성 -->
	<context-param>
		<param-name>contextClass</param-name>
		<param-value>
			org.springframework.web.context.support.AnnotationConfigWebApplicationContext
		</param-value>
	</context-param>

	<!-- 구성 위치는 하나 이상의 쉼표 또는 공백으로 구분된
		정규화된 @Configuration 클래스로 구성되어야 합니다.
		컴포넌트 스캐닝을 위해 정규화된 패키지도 지정할 수 있습니다 -->
	<context-param>
		<param-name>contextConfigLocation</param-name>
		<param-value>com.acme.AppConfig</param-value>
	</context-param>

	<!-- ContextLoaderListener를 사용하여 평소처럼 루트 애플리케이션 컨텍스트 부트스트랩 -->
	<listener>
		<listener-class>org.springframework.web.context.ContextLoaderListener</listener-class>
	</listener>

	<!-- 평소처럼 스프링 MVC DispatcherServlet 선언 -->
	<servlet>
		<servlet-name>dispatcher</servlet-name>
		<servlet-class>org.springframework.web.servlet.DispatcherServlet</servlet-class>
		<!-- DispatcherServlet이 기본 XmlWebApplicationContext 대신
			AnnotationConfigWebApplicationContext를 사용하도록 구성 -->
		<init-param>
			<param-name>contextClass</param-name>
			<param-value>
				org.springframework.web.context.support.AnnotationConfigWebApplicationContext
			</param-value>
		</init-param>
		<!-- 다시 한번, 구성 위치는 하나 이상의 쉼표 또는 공백으로 구분되고
			정규화된 @Configuration 클래스로 구성되어야 합니다 -->
		<init-param>
			<param-name>contextConfigLocation</param-name>
			<param-value>com.acme.web.MvcConfig</param-value>
		</init-param>
	</servlet>

	<!-- /app/*에 대한 모든 요청을 dispatcher 서블릿에 매핑 -->
	<servlet-mapping>
		<servlet-name>dispatcher</servlet-name>
		<url-pattern>/app/*</url-pattern>
	</servlet-mapping>
</web-app>
```

*프로그래밍 방식 사용 사례의 경우, `GenericWebApplicationContext`를 `AnnotationConfigWebApplicationContext`의 대안으로 사용할 수 있습니다.*

---

**전체 주제: AnnotationConfigApplicationContext를 사용하여 스프링 컨테이너 인스턴스화**

이 부분은 XML 설정 파일 대신 **자바 설정 클래스(`@Configuration` 등)나 컴포넌트 클래스(`@Component` 등)를 입력으로 사용하여 스프링 컨테이너(`ApplicationContext`)를 만드는 방법**을 설명합니다.

XML 없는(XML-free) 스프링 설정을 가능하게 하는 핵심적인 방법입니다.

**핵심 아이디어:** XML 파일 경로 대신 자바 클래스 이름을 넘겨주면 스프링 컨테이너를 만들 수 있다!

---

**1. `AnnotationConfigApplicationContext` 소개**

- **역할:** `org.springframework.context.annotation.AnnotationConfigApplicationContext`는 **어노테이션 기반 설정**을 읽어들여 스프링 컨테이너를 생성하는 `ApplicationContext`의 주요 구현체입니다.
- **입력:** 이 클래스는 다음과 같은 종류의 클래스들을 입력으로 받아 처리할 수 있습니다:
  - **`@Configuration` 클래스:** 이 클래스 자체와 그 안에 정의된 모든 `@Bean` 메소드가 빈 정의로 등록됩니다.
  - **`@Component` 클래스 (및 `@Service`, `@Repository`, `@Controller`):** 이 클래스 자체가 빈 정의로 등록됩니다. 내부의 `@Autowired`나 `@Inject` 같은 DI 어노테이션도 처리됩니다.
  - **JSR-330 어노테이션 클래스 (`@Named`, `@ManagedBean`):** 이 클래스들도 `@Component`처럼 빈 정의로 등록되고 DI 어노테이션(`@Inject`)이 처리됩니다.

---

**2. 간단한 생성 방법 (생성자 인수 사용)**

- 가장 기본적인 방법은 `AnnotationConfigApplicationContext` 생성자에 **설정 클래스(들)의 `.class` 객체를 직접 전달**하는 것입니다. 이는 `ClassPathXmlApplicationContext` 생성자에 XML 파일 경로를 전달하는 것과 매우 유사합니다.

    ```java
    // Java
    // AppConfig.class (@Configuration 클래스)를 기반으로 컨테이너 생성
    ApplicationContext ctx = new AnnotationConfigApplicationContext(AppConfig.class);
    MyService myService = ctx.getBean(MyService.class); // 빈 가져오기
    myService.doStuff();
    
    // 여러 설정 클래스 또는 컴포넌트 클래스 전달 가능
    ApplicationContext ctx2 = new AnnotationConfigApplicationContext(
        MyServiceImpl.class, // @Component 클래스
        Dependency1.class,   // @Component 클래스
        Dependency2.class    // @Component 클래스
    );
    MyService myService2 = ctx2.getBean(MyService.class);
    ```

    ```kotlin
    // Kotlin
    val ctx = AnnotationConfigApplicationContext(AppConfig::class.java) // Kotlin에서는 ::class.java 사용
    val myService = ctx.getBean(MyService::class.java)
    myService.doStuff()
    
    val ctx2 = AnnotationConfigApplicationContext(
        MyServiceImpl::class.java,
        Dependency1::class.java,
        Dependency2::class.java
    )
    val myService2 = ctx2.getBean(MyService::class.java)
    ```


---

**3. 프로그래밍 방식으로 컨테이너 빌드하기 (`register()` 메소드 사용)**

- 때로는 컨테이너를 먼저 만들고, **나중에 설정 클래스들을 동적으로 등록**하고 싶을 수 있습니다. (예: 조건에 따라 다른 설정을 로드할 때)
- 이 경우, `AnnotationConfigApplicationContext`의 **기본 생성자**로 객체를 먼저 만들고, `register(Class<?>...)` 메소드를 사용하여 설정 클래스들을 **추가**한 다음, **반드시 `refresh()` 메소드를 호출하여 컨테이너를 초기화**해야 합니다.

    ```java
    // Java
    AnnotationConfigApplicationContext ctx = new AnnotationConfigApplicationContext(); // 기본 생성자로 생성
    
    // 설정 클래스들 등록
    ctx.register(AppConfig.class, OtherConfig.class);
    ctx.register(AdditionalConfig.class); // 여러 번 호출 가능
    
    // ★★★ 중요: 등록 후 refresh() 호출 필수 ★★★
    ctx.refresh();
    
    MyService myService = ctx.getBean(MyService.class);
    myService.doStuff();
    ```

    ```kotlin
    // Kotlin
    val ctx = AnnotationConfigApplicationContext()
    ctx.register(AppConfig::class.java, OtherConfig::class.java)
    ctx.register(AdditionalConfig::class.java)
    ctx.refresh() // ★★★ 중요 ★★★
    val myService = ctx.getBean(MyService::class.java)
    myService.doStuff()
    ```


---

**4. 컴포넌트 스캐닝 활성화하기 (`scan()` 메소드 사용)**

- `@Configuration` 클래스에 `@ComponentScan` 어노테이션을 붙이는 대신, `AnnotationConfigApplicationContext`의 `scan(String...)` 메소드를 사용하여 **프로그래밍 방식으로 컴포넌트 스캐닝을 활성화**할 수도 있습니다.
- **사용법:** 기본 생성자로 컨텍스트를 만들고, `scan()` 메소드에 스캔을 시작할 **패키지 이름 문자열**을 전달한 후, `refresh()`를 호출합니다.

    ```java
    // Java
    AnnotationConfigApplicationContext ctx = new AnnotationConfigApplicationContext();
    ctx.scan("com.acme"); // "com.acme" 패키지 하위를 스캔하도록 지시
    ctx.refresh(); // 스캔 및 빈 등록, 초기화 수행
    MyService myService = ctx.getBean(MyService.class);
    ```

    ```kotlin
    // Kotlin
    val ctx = AnnotationConfigApplicationContext()
    ctx.scan("com.acme")
    ctx.refresh()
    val myService = ctx.getBean(MyService::class.java)
    ```

- **`scan()`과 `@Configuration` 클래스:** `scan()` 메소드가 지정된 패키지를 스캔하다가 `@Configuration` 클래스를 발견하면, 그 클래스도 빈으로 등록하고 내부의 `@Bean` 메소드들도 처리합니다.

---

**5. 웹 애플리케이션 지원 (`AnnotationConfigWebApplicationContext`)**

- 웹 애플리케이션 환경에서는 `AnnotationConfigApplicationContext` 대신 **`AnnotationConfigWebApplicationContext`** 라는 웹 전용 구현체를 사용합니다.
- **설정 방법 (`web.xml`):**
  - 기존의 XML 기반 웹 애플리케이션(`web.xml`)에서 Java Config를 사용하려면, `web.xml` 파일 수정을 통해 스프링 컨텍스트 로딩 방식을 변경해야 합니다.
  - **`ContextLoaderListener` 설정:**
    - `<context-param>`의 `contextClass` 파라미터 값을 `org.springframework.web.context.support.AnnotationConfigWebApplicationContext`로 지정합니다. (기본값은 `XmlWebApplicationContext`)
    - `contextConfigLocation` 파라미터 값에는 **XML 파일 경로 대신**, 로드할 `@Configuration` 클래스의 **정규화된 클래스 이름**(쉼표나 공백으로 여러 개 지정 가능) 또는 **컴포넌트 스캔할 패키지 이름**을 지정합니다.
  - **`DispatcherServlet` 설정:**
    - `<servlet>`의 `<init-param>`으로 `contextClass`와 `contextConfigLocation`을 `ContextLoaderListener`와 유사하게 지정하여 `DispatcherServlet`이 사용할 컨텍스트도 어노테이션 기반으로 설정합니다.

**요약:**

`AnnotationConfigApplicationContext`는 XML 파일 없이 `@Configuration`, `@Component`, JSR-330 어노테이션이 붙은 자바 클래스를 직접 사용하여 스프링 컨테이너를 생성하고 설정하는 핵심 클래스입니다.

생성자에 클래스를 전달하거나, `register()` 및 `scan()` 메소드를 사용하여 프로그래밍 방식으로 컨테이너를 구성하고 `refresh()`를 통해 초기화할 수 있습니다.

웹 애플리케이션에서는 `AnnotationConfigWebApplicationContext`를 사용하며, `web.xml` 수정을 통해 적용할 수 있습니다.
