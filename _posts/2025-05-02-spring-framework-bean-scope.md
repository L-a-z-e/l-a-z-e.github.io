---
title: Spring Framework Bean Scopes
description: 
author: laze
date: 2025-05-02 00:00:01 +0900
categories: [Dev, SpringBoot]
tags: [SpringBoot]
---
빈 정의를 생성할 때, 해당 빈 정의에 의해 정의된 클래스의 실제 인스턴스를 생성하기 위한 레시피를 만드는 것입니다.

빈 정의가 레시피라는 아이디어는 중요합니다.

왜냐하면 클래스와 마찬가지로 단일 레시피에서 많은 객체 인스턴스를 만들 수 있다는 의미이기 때문입니다.

특정 빈 정의에서 생성되는 객체에 연결될 다양한 의존성 및 구성 값뿐만 아니라, 특정 빈 정의에서 생성되는 객체의 스코프(scope)도 제어할 수 있습니다.

이 접근 방식은 강력하고 유연합니다.

왜냐하면 자바 클래스 레벨에서 객체의 스코프를 고정(bake in)할 필요 없이 구성을 통해 생성하는 객체의 스코프를 선택할 수 있기 때문입니다.

빈은 여러 스코프 중 하나에 배포되도록 정의될 수 있습니다.

스프링 프레임워크는 6가지 스코프를 지원하며, 그중 4가지는 웹 인식 `ApplicationContext`를 사용하는 경우에만 사용할 수 있습니다.

커스텀 스코프(custom scope)를 생성할 수도 있습니다.

| 스코프 | 설명 |
| --- | --- |
| `singleton` | (기본값) 각 스프링 IoC 컨테이너당 단일 빈 정의를 단일 객체 인스턴스로 스코프 지정합니다. |
| `prototype` | 단일 빈 정의를 임의 개수의 객체 인스턴스로 스코프 지정합니다. |
| `request` | 단일 빈 정의를 단일 HTTP 요청의 생명주기로 스코프 지정합니다. 즉, 각 HTTP 요청은 단일 빈 정의로부터 생성된 자체 빈 인스턴스를 가집니다. 웹 인식 스프링 `ApplicationContext`의 컨텍스트에서만 유효합니다. |
| `session` | 단일 빈 정의를 HTTP 세션의 생명주기로 스코프 지정합니다. 웹 인식 스프링 `ApplicationContext`의 컨텍스트에서만 유효합니다. |
| `application` | 단일 빈 정의를 `ServletContext`의 생명주기로 스코프 지정합니다. 웹 인식 스프링 `ApplicationContext`의 컨텍스트에서만 유효합니다. |
| `websocket` | 단일 빈 정의를 웹소켓(WebSocket)의 생명주기로 스코프 지정합니다. 웹 인식 스프링 `ApplicationContext`의 컨텍스트에서만 유효합니다. |

**싱글톤 스코프 (The Singleton Scope)**

싱글톤 빈의 공유 인스턴스는 하나만 관리되며, 해당 빈 정의와 일치하는 ID 또는 ID들을 가진 빈에 대한 모든 요청은 스프링 컨테이너에 의해 해당 특정 빈 인스턴스가 반환되는 결과를 낳습니다.

다르게 말하면, 빈 정의를 정의하고 그것이 싱글톤으로 스코프 지정될 때, 스프링 IoC 컨테이너는 해당 빈 정의에 의해 정의된 객체의 인스턴스를 정확히 하나 생성합니다.

이 단일 인스턴스는 이러한 싱글톤 빈들의 캐시에 저장되며, 해당 이름 붙여진 빈에 대한 모든 후속 요청 및 참조는 캐시된 객체를 반환합니다.

스프링의 싱글톤 빈 개념은 Gang of Four(GoF) 패턴 책에서 정의된 싱글톤 패턴과는 다릅니다.

GoF 싱글톤은 특정 클래스의 인스턴스가 클래스로더(ClassLoader)당 하나만 생성되도록 객체의 스코프를 하드코딩합니다.

스프링 싱글톤의 스코프는 컨테이너별 및 빈별(per-container and per-bean)로 가장 잘 설명됩니다.

이는 단일 스프링 컨테이너에서 특정 클래스에 대해 하나의 빈을 정의하면 스프링 컨테이너가 해당 빈 정의에 의해 정의된 클래스의 인스턴스를 하나만 생성한다는 것을 의미합니다.

싱글톤 스코프는 스프링의 기본 스코프입니다.

XML에서 빈을 싱글톤으로 정의하려면 다음 예제와 같이 빈을 정의할 수 있습니다:

```xml
<bean id="accountService" class="com.something.DefaultAccountService"/>

<!-- 다음은 동일하지만 불필요합니다 (싱글톤 스코프가 기본값이므로) -->
<bean id="accountService" class="com.something.DefaultAccountService" scope="singleton"/>
```

**프로토타입 스코프 (The Prototype Scope)**

비-싱글톤 프로토타입 스코프의 빈 배포는 해당 특정 빈에 대한 요청이 있을 때마다 새로운 빈 인스턴스를 생성하는 결과를 낳습니다.

즉, 빈이 다른 빈에 주입되거나 컨테이너에서 `getBean()` 메소드 호출을 통해 요청할 때입니다.

일반적으로 모든 상태 저장(stateful) 빈에는 프로토타입 스코프를 사용하고 상태 비저장(stateless) 빈에는 싱글톤 스코프를 사용해야 합니다.
(데이터 접근 객체(DAO)는 일반적으로 프로토타입으로 구성되지 않습니다.

왜냐하면 일반적인 DAO는 대화 상태(conversational state)를 보유하지 않기 때문입니다. 싱글톤 다이어그램의 핵심을 재사용하는 것이 더 쉬웠습니다.)

다음 예제는 XML에서 빈을 프로토타입으로 정의합니다:

```xml
<bean id="accountService" class="com.something.DefaultAccountService" scope="prototype"/>
```

다른 스코프와 대조적으로, 스프링은 프로토타입 빈의 완전한 생명주기를 관리하지 않습니다.

컨테이너는 프로토타입 객체를 인스턴스화하고, 설정하고, 조립한 다음 클라이언트에게 전달하며, 해당 프로토타입 인스턴스에 대한 추가 기록은 없습니다.

따라서 초기화 생명주기 콜백 메소드는 스코프에 관계없이 모든 객체에서 호출되지만, 프로토타입의 경우 구성된 소멸 생명주기 콜백(destruction lifecycle callbacks)은 호출되지 않습니다.

클라이언트 코드는 프로토타입 스코프 객체를 정리하고 프로토타입 빈이 보유한 값비싼 리소스를 해제해야 합니다.

스프링 컨테이너가 프로토타입 스코프 빈이 보유한 리소스를 해제하게 하려면, 정리해야 할 빈에 대한 참조를 보유하는 커스텀 빈 후처리기(bean post-processor)를 사용해 보십시오.

어떤 면에서 프로토타입 스코프 빈에 대한 스프링 컨테이너의 역할은 자바 `new` 연산자를 대체하는 것입니다.

그 이후의 모든 생명주기 관리는 클라이언트가 처리해야 합니다.

**프로토타입 빈 의존성을 가진 싱글톤 빈 (Singleton Beans with Prototype-bean Dependencies)**

프로토타입 빈에 대한 의존성을 가진 싱글톤 스코프 빈을 사용할 때, 의존성은 인스턴스화 시점에 해결된다는 점에 유의하십시오.

따라서 프로토타입 스코프 빈을 싱글톤 스코프 빈에 의존성 주입하면 새로운 프로토타입 빈이 인스턴스화된 다음 싱글톤 빈에 의존성 주입됩니다.

이 프로토타입 인스턴스는 싱글톤 스코프 빈에 제공되는 유일한 인스턴스입니다.

그러나 싱글톤 스코프 빈이 런타임 시 프로토타입 스코프 빈의 새 인스턴스를 반복적으로 획득하기를 원한다고 가정해 봅시다.

프로토타입 스코프 빈을 싱글톤 빈에 의존성 주입할 수 없습니다.

왜냐하면 해당 주입은 스프링 컨테이너가 싱글톤 빈을 인스턴스화하고 그 의존성을 해결하고 주입할 때 단 한 번만 발생하기 때문입니다.

런타임 시 프로토타입 빈의 새 인스턴스가 두 번 이상 필요하다면, 메소드 주입(Method Injection)을 사용해야 합니다.

**Request, Session, Application, WebSocket 스코프**

`request`, `session`, `application`, `websocket` 스코프는 웹 인식 스프링 `ApplicationContext` 구현체(예: `XmlWebApplicationContext`)를 사용하는 경우에만 사용할 수 있습니다.

이러한 스코프를 `ClassPathXmlApplicationContext`와 같은 일반 스프링 IoC 컨테이너와 함께 사용하면 알 수 없는 빈 스코프에 대해  `IllegalStateException`이 발생합니다.

*초기 웹 구성 (Initial Web Configuration)*
request, session, application, websocket 레벨(웹 스코프 빈)에서 빈의 스코프 지정을 지원하려면, 빈을 정의하기 전에 약간의 초기 구성이 필요합니다. (이 초기 설정은 표준 스코프인 싱글톤 및 프로토타입에는 필요하지 않습니다.)

이 초기 설정을 수행하는 방법은 특정 서블릿 환경에 따라 다릅니다.

스프링 웹 MVC 내에서, 즉 스프링 `DispatcherServlet`에 의해 처리되는 요청 내에서 스코프 빈에 접근하는 경우 특별한 설정이 필요하지 않습니다.

`DispatcherServlet`은 이미 모든 관련 상태를 노출합니다.

스프링의 `DispatcherServlet` 외부에서 처리되는 요청과 함께 서블릿 웹 컨테이너를 사용하는 경우(예: JSF 사용 시), `org.springframework.web.context.request.RequestContextListener` `ServletRequestListener`를 등록해야 합니다.

이는 `WebApplicationInitializer` 인터페이스를 사용하여 프로그래밍 방식으로 수행할 수 있습니다. 또는 웹 애플리케이션의 `web.xml` 파일에 다음 선언을 추가합니다:

```xml
<web-app>
	...
	<listener>
		<listener-class>
			org.springframework.web.context.request.RequestContextListener
		</listener-class>
	</listener>
	...
</web-app>
```

또는 리스너 설정에 문제가 있는 경우 스프링의 `RequestContextFilter` 사용을 고려하십시오. 필터 매핑은 주변 웹 애플리케이션 구성에 따라 달라지므로 적절하게 변경해야 합니다. 다음 목록은 웹 애플리케이션의 필터 부분을 보여줍니다:

```xml
<web-app>
	...
	<filter>
		<filter-name>requestContextFilter</filter-name>
		<filter-class>org.springframework.web.filter.RequestContextFilter</filter-class>
	</filter>
	<filter-mapping>
		<filter-name>requestContextFilter</filter-name>
		<url-pattern>/*</url-pattern>
	</filter-mapping>
	...
</web-app>
```

`DispatcherServlet`, `RequestContextListener`, `RequestContextFilter`는 모두 정확히 동일한 작업을 수행합니다.

즉, HTTP 요청 객체를 해당 요청을 서비스하는 스레드(Thread)에 바인딩합니다. 이렇게 하면 request 및 session 스코프 빈이 호출 체인의 하위 단계에서 사용 가능해집니다.

*Request 스코프*

```xml
<bean id="loginAction" class="com.something.LoginAction" scope="request"/>
```

스프링 컨테이너는 각각의 모든 HTTP 요청에 대해 `loginAction` 빈 정의를 사용하여 `LoginAction` 빈의 새 인스턴스를 생성합니다.

즉, `loginAction` 빈은 HTTP 요청 레벨에서 스코프 지정됩니다.

생성된 인스턴스의 내부 상태는 원하는 만큼 변경할 수 있습니다.

왜냐하면 동일한 `loginAction` 빈 정의에서 생성된 다른 인스턴스는 상태의 이러한 변경 사항을 보지 못하기 때문입니다.

그것들은 개별 요청에 특정합니다. 요청 처리가 완료되면 요청에 스코프 지정된 빈은 폐기됩니다.

어노테이션 기반 컴포넌트 또는 자바 구성을 사용할 때 `@RequestScope` 어노테이션을 사용하여 컴포넌트를 request 스코프에 할당할 수 있습니다.

```java
// Java
@RequestScope
@Component
public class LoginAction {
	// ...
}
```

```kotlin
// Kotlin
@RequestScope
@Component
class LoginAction {
    // ...
}
```

*Session 스코프*
빈 정의에 대한 다음 XML 구성을 고려하십시오:

```xml
<bean id="userPreferences" class="com.something.UserPreferences" scope="session"/>
```

스프링 컨테이너는 단일 HTTP 세션의 수명 동안 `userPreferences` 빈 정의를 사용하여 `UserPreferences` 빈의 새 인스턴스를 생성합니다.

즉, `userPreferences` 빈은 효과적으로 HTTP 세션 레벨에서 스코프 지정됩니다.

request 스코프 빈과 마찬가지로, 생성된 인스턴스의 내부 상태는 원하는 만큼 변경할 수 있습니다.

동일한 `userPreferences` 빈 정의에서 생성된 인스턴스를 사용하는 다른 HTTP 세션 인스턴스는 이러한 상태 변경을 보지 못한다는 것을 알기 때문입니다.

왜냐하면 그것들은 개별 HTTP 세션에 특정하기 때문입니다.

HTTP 세션이 결국 폐기되면 해당 특정 HTTP 세션에 스코프 지정된 빈도 폐기됩니다.

어노테이션 기반 컴포넌트 또는 자바 구성을 사용할 때 `@SessionScope` 어노테이션을 사용하여 컴포넌트를 session 스코프에 할당할 수 있습니다.

```java
// Java
@SessionScope
@Component
public class UserPreferences {
	// ...
}
```

```kotlin
// Kotlin
@SessionScope
@Component
class UserPreferences {
    // ...
}
```

*Application 스코프*
빈 정의에 대한 다음 XML 구성을 고려하십시오:

```xml
<bean id="appPreferences" class="com.something.AppPreferences" scope="application"/>
```

스프링 컨테이너는 전체 웹 애플리케이션에 대해 한 번 `appPreferences` 빈 정의를 사용하여 `AppPreferences` 빈의 새 인스턴스를 생성합니다.

즉, `appPreferences` 빈은 `ServletContext` 레벨에서 스코프 지정되고 일반 `ServletContext` 속성으로 저장됩니다.

이는 스프링 싱글톤 빈과 다소 유사하지만 두 가지 중요한 방식으로 다릅니다

특정 웹 애플리케이션에 여러 개 있을 수 있는 스프링 `ApplicationContext`당 싱글톤이 아니라 `ServletContext`당 싱글톤이며, 실제로 노출되어 `ServletContext` 속성으로 보입니다.

어노테이션 기반 컴포넌트 또는 자바 구성을 사용할 때 `@ApplicationScope` 어노테이션을 사용하여 컴포넌트를 application 스코프에 할당할 수 있습니다. 다음 예제는 이를 수행하는 방법을 보여줍니다:

```java
// Java
@ApplicationScope
@Component
public class AppPreferences {
	// ...
}

```

```kotlin
// Kotlin
@ApplicationScope
@Component
class AppPreferences {
    // ...
}
```

*WebSocket 스코프*
웹소켓 스코프는 웹소켓 세션의 생명주기와 연관되며 STOMP over WebSocket 애플리케이션에 적용됩니다.

**의존성으로서의 스코프 빈 (Scoped Beans as Dependencies)**

스프링 IoC 컨테이너는 객체(빈)의 인스턴스화뿐만 아니라 협력자(또는 의존성)의 연결(wiring up)도 관리합니다.

(예를 들어) HTTP request 스코프 빈을 더 오래 지속되는 스코프의 다른 빈에 주입하려면, 스코프 빈 대신 AOP 프록시를 주입하도록 선택할 수 있습니다.

즉, 스코프 객체와 동일한 공개 인터페이스를 노출하지만 관련 스코프(예: HTTP 요청)에서 실제 대상 객체를 검색하고 실제 객체에 메소드 호출을 위임할 수도 있는 프록시 객체를 주입해야 합니다.

`<aop:scoped-proxy/>`를 싱글톤으로 스코프 지정된 빈들 사이에서도 사용할 수 있으며, 그러면 참조는 직렬화 가능(serializable)하고 따라서 역직렬화(deserialization) 시 대상 싱글톤 빈을 다시 얻을 수 있는 중간 프록시를 통하게 됩니다.
`prototype` 스코프의 빈에 대해 `<aop:scoped-proxy/>`를 선언하면 공유 프록시에 대한 모든 메소드 호출은 새로운 대상 인스턴스 생성을 유발하며, 호출은 그 인스턴스로 전달됩니다.

또한, 스코프 프록시는 수명주기 안전(lifecycle-safe) 방식으로 더 짧은 스코프의 빈에 접근하는 유일한 방법은 아닙니다.

주입 지점(즉, 생성자 또는 세터 인수 또는 자동 와이어링된 필드)을 `ObjectFactory<MyTargetBean>`으로 선언하여 필요할 때마다 `getObject()` 호출을 통해 현재 인스턴스를 온디맨드(on demand)로 검색할 수도 있습니다

- 인스턴스를 보유하거나 별도로 저장하지 않고
  확장된 변형으로, `ObjectProvider<MyTargetBean>`을 선언하여 `getIfAvailable` 및 `getIfUnique`를 포함한 여러 추가 접근 변형을 제공할 수 있습니다.
  이것의 JSR-330 변형은 `Provider`라고 불리며 `Provider<MyTargetBean>` 선언과 각 검색 시도에 대한 해당 `get()` 호출과 함께 사용됩니다.

다음 예제의 구성은 단 한 줄이지만, 그 이면의 "이유"와 "방법"을 이해하는 것이 중요합니다:

```xml
<?xml version="1.0" encoding="UTF-8"?>
<beans xmlns="<http://www.springframework.org/schema/beans>"
	xmlns:xsi="<http://www.w3.org/2001/XMLSchema-instance>"
	xmlns:aop="<http://www.springframework.org/schema/aop>"
	xsi:schemaLocation="<http://www.springframework.org/schema/beans>
		<https://www.springframework.org/schema/beans/spring-beans.xsd>
		<http://www.springframework.org/schema/aop>
		<https://www.springframework.org/schema/aop/spring-aop.xsd>">

	<!-- 프록시로 노출된 HTTP Session 스코프 빈 -->
	<bean id="userPreferences" class="com.something.UserPreferences" scope="session">
		<!-- 컨테이너에게 주변 빈을 프록시하도록 지시 -->
		<aop:scoped-proxy/> <!-- ① -->
	</bean>

	<!-- 위의 빈에 대한 프록시가 주입된 싱글톤 스코프 빈 -->
	<bean id="userService" class="com.something.SimpleUserService">
		<!-- 프록시된 userPreferences 빈에 대한 참조 -->
		<property name="userPreferences" ref="userPreferences"/>
	</bean>
</beans>
```

① 프록시를 정의하는 라인.

이러한 프록시를 생성하려면, 스코프 빈 정의에 자식 `<aop:scoped-proxy/>` 요소를 삽입합니다 (생성할 프록시 타입 선택 및 XML 스키마 기반 구성 참조).

일반적인 시나리오에서 request, session 및 커스텀 스코프 레벨에서 스코프 지정된 빈 정의에 `<aop:scoped-proxy/>` 요소가 필요한 이유는 무엇일까요?

다음 싱글톤 빈 정의를 고려하고 앞서 언급한 스코프에 대해 정의해야 하는 것과 대조해 보십시오 (다음 `userPreferences` 빈 정의는 현재 상태로는 불완전합니다):

```xml
<bean id="userPreferences" class="com.something.UserPreferences" scope="session"/>

<bean id="userManager" class="com.something.UserManager">
	<property name="userPreferences" ref="userPreferences"/>
</bean>
```

앞의 예제에서 싱글톤 빈(`userManager`)은 HTTP Session 스코프 빈(`userPreferences`)에 대한 참조로 주입됩니다.

여기서 중요한 점은 `userManager` 빈이 싱글톤이라는 것입니다: 컨테이너당 정확히 한 번 인스턴스화되며, 그 의존성(이 경우 `userPreferences` 빈 하나뿐)도 단 한 번만 주입됩니다.

이는 `userManager` 빈이 정확히 동일한 `userPreferences` 객체(즉, 원래 주입된 객체)에서만 작동한다는 것을 의미합니다.

이는 더 짧은 수명의 스코프 빈을 더 긴 수명의 스코프 빈에 주입할 때(예: HTTP Session 스코프 협력 빈을 싱글톤 빈의 의존성으로 주입) 원하는 동작이 아닙니다.

오히려 단일 `userManager` 객체가 필요하고, HTTP 세션의 수명 동안에는 해당 HTTP 세션에 특정한 `userPreferences` 객체가 필요합니다.

따라서 컨테이너는 `UserPreferences` 클래스와 정확히 동일한 공개 인터페이스를 노출하는 객체(이상적으로는 `UserPreferences` 인스턴스인 객체)를 생성하며,

이 객체는 스코핑 메커니즘(HTTP request, Session 등)에서 실제 `UserPreferences` 객체를 가져올 수 있습니다.

컨테이너는 이 프록시 객체를 `userManager` 빈에 주입하며, `userManager`는 이 `UserPreferences` 참조가 프록시라는 것을 알지 못합니다.

이 예제에서 `UserManager` 인스턴스가 의존성 주입된 `UserPreferences` 객체에 대해 메소드를 호출하면 실제로는 프록시에 대해 메소드를 호출하는 것입니다.

그러면 프록시는 (이 경우) HTTP 세션에서 실제 `UserPreferences` 객체를 가져오고 검색된 실제 `UserPreferences` 객체에 메소드 호출을 위임합니다.

따라서 다음 예제와 같이 request 및 session 스코프 빈을 협력 객체에 주입할 때 다음과 같은 (올바르고 완전한) 구성이 필요합니다:

```xml
<bean id="userPreferences" class="com.something.UserPreferences" scope="session">
	<aop:scoped-proxy/>
</bean>

<bean id="userManager" class="com.something.UserManager">
	<property name="userPreferences" ref="userPreferences"/>
</bean>
```

*생성할 프록시 타입 선택 (Choosing the Type of Proxy to Create)*
기본적으로 스프링 컨테이너가 `<aop:scoped-proxy/>` 요소로 표시된 빈에 대한 프록시를 생성할 때 CGLIB 기반 클래스 프록시가 생성됩니다.

*CGLIB 프록시는 private 메소드를 가로채지(intercept) 않습니다. 이러한 프록시에서 private 메소드를 호출하려고 시도하면 실제 스코프 대상 객체로 위임되지 않습니다.*
또는, `<aop:scoped-proxy/>` 요소의 `proxy-target-class` 속성 값을 `false`로 지정하여 스프링 컨테이너가 이러한 스코프 빈에 대해 표준 JDK 인터페이스 기반 프록시를 생성하도록 구성할 수 있습니다.

JDK 인터페이스 기반 프록시를 사용하면 이러한 프록시 생성에 영향을 미치기 위해 애플리케이션 클래스패스에 추가 라이브러리가 필요하지 않다는 것을 의미합니다.

그러나 이는 또한 스코프 빈의 클래스가 최소한 하나의 인터페이스를 구현해야 하고 스코프 빈이 주입되는 모든 협력자가 인터페이스 중 하나를 통해 빈을 참조해야 함을 의미합니다.

다음 예제는 인터페이스 기반 프록시를 보여줍니다:

```xml
<!-- DefaultUserPreferences는 UserPreferences 인터페이스를 구현함 -->
<bean id="userPreferences" class="com.stuff.DefaultUserPreferences" scope="session">
	<aop:scoped-proxy proxy-target-class="false"/>
</bean>

<bean id="userManager" class="com.stuff.UserManager">
	<property name="userPreferences" ref="userPreferences"/>
</bean>

```

*Request/Session 참조 직접 주입 (Injecting Request/Session References Directly)*
팩토리 스코프의 대안으로, 스프링 `WebApplicationContext`는 다른 빈에 대한 일반적인 주입 지점 옆에 단순히 타입 기반 자동 와이어링을 통해
`HttpServletRequest`, `HttpServletResponse`, `HttpSession`, `WebRequest` 및 (JSF가 있는 경우) `FacesContext`와 `ExternalContext`를 스프링 관리 빈에 주입하는 것도 지원합니다.
스프링은 일반적으로 이러한 request 및 session 객체에 대한 프록시를 주입하며, 이는 팩토리 스코프 빈에 대한 스코프 프록시와 유사하게 싱글톤 빈 및 직렬화 가능 빈에서도 작동하는 이점이 있습니다.

**커스텀 스코프 (Custom Scopes)**

빈 스코핑 메커니즘은 확장 가능합니다. 자신만의 스코프를 정의하거나 기존 스코프를 재정의할 수도 있습니다.
후자는 좋지 않은 관행으로 간주되며 내장된 싱글톤 및 프로토타입 스코프는 오버라이드할 수 없습니다.

*커스텀 스코프 생성하기 (Creating a Custom Scope)*
커스텀 스코프를 스프링 컨테이너에 통합하려면 이 섹션에서 설명하는 `org.springframework.beans.factory.config.Scope` 인터페이스를 구현해야 합니다.

`Scope` 인터페이스에는 스코프에서 객체를 가져오고, 스코프에서 제거하고, 소멸되도록 하는 네 가지 메소드가 있습니다.

예를 들어, session 스코프 구현은 session 스코프 빈을 반환합니다 (존재하지 않으면, 향후 참조를 위해 세션에 바인딩한 후 빈의 새 인스턴스를 반환합니다).

```java
// Java
Object get(String name, ObjectFactory<?> objectFactory)
```

```kotlin
// Kotlin
fun get(name: String, objectFactory: ObjectFactory<*>): Any
```

예를 들어, session 스코프 구현은 기본 세션에서 session 스코프 빈을 제거합니다.

객체는 반환되어야 하지만, 지정된 이름의 객체를 찾을 수 없는 경우 null을 반환할 수 있습니다. 다음 메소드는 기본 스코프에서 객체를 제거합니다:

```java
// Java
Object remove(String name)
```

```kotlin
// Kotlin
fun remove(name: String): Any?
```

다음 메소드는 스코프가 소멸되거나 스코프 내의 지정된 객체가 소멸될 때 스코프가 호출해야 하는 콜백을 등록합니다:

```java
// Java
void registerDestructionCallback(String name, Runnable destructionCallback)
```

```kotlin
// Kotlin
fun registerDestructionCallback(name: String, callback: Runnable)
```

다음 메소드는 기본 스코프에 대한 대화 식별자(conversation identifier)를 얻습니다:

```java
// Java
String getConversationId()
```

```kotlin
// Kotlin
fun getConversationId(): String?
```

이 식별자는 각 스코프마다 다릅니다. session 스코프 구현의 경우 이 식별자는 세션 식별자일 수 있습니다.

3

*커스텀 스코프 사용하기 (Using a Custom Scope)*
하나 이상의 커스텀 `Scope` 구현을 작성하고 테스트한 후, 스프링 컨테이너가 새로운 스코프를 인식하도록 해야 합니다. 다음 메소드는 스프링 컨테이너에 새로운 `Scope`를 등록하는 핵심 메소드입니다:

```java
// Java
void registerScope(String scopeName, Scope scope);
```

```kotlin
// Kotlin
fun registerScope(scopeName: String, scope: Scope)
```

이 메소드는 `ConfigurableBeanFactory` 인터페이스에 선언되어 있으며, 스프링과 함께 제공되는 대부분의 구체적인 `ApplicationContext` 구현체의 `BeanFactory` 속성을 통해 사용할 수 있습니다.

`registerScope(..)` 메소드의 첫 번째 인수는 스코프와 연관된 고유한 이름입니다.

스프링 컨테이너 자체의 이러한 이름의 예로는 `singleton`과 `prototype`이 있습니다. `registerScope(..)` 메소드의 두 번째 인수는 등록하고 사용하려는 커스텀 `Scope` 구현의 실제 인스턴스입니다.

커스텀 `Scope` 구현을 작성한 다음 다음 예제와 같이 등록한다고 가정해 봅시다.

*다음 예제는 스프링에 포함되어 있지만 기본적으로 등록되지 않은 `SimpleThreadScope`를 사용합니다. 지침은 사용자 정의 `Scope` 구현에 대해서도 동일합니다.*

```java
// Java
Scope threadScope = new SimpleThreadScope();
beanFactory.registerScope("thread", threadScope);
```

```kotlin
// Kotlin
val threadScope: Scope = SimpleThreadScope()
beanFactory.registerScope("thread", threadScope)
```

그런 다음 다음과 같이 커스텀 `Scope`의 스코핑 규칙을 따르는 빈 정의를 생성할 수 있습니다:

```xml
<bean id="..." class="..." scope="thread">
```

커스텀 `Scope` 구현을 사용하면 스코프의 프로그래밍 방식 등록에만 국한되지 않습니다. 다음 예제와 같이 `CustomScopeConfigurer` 클래스를 사용하여 선언적으로 스코프 등록을 수행할 수도 있습니다:

```xml
<?xml version="1.0" encoding="UTF-8"?>
<beans xmlns="<http://www.springframework.org/schema/beans>"
	xmlns:xsi="<http://www.w3.org/2001/XMLSchema-instance>"
	xmlns:aop="<http://www.springframework.org/schema/aop>"
	xsi:schemaLocation="<http://www.springframework.org/schema/beans>
		<https://www.springframework.org/schema/beans/spring-beans.xsd>
		<http://www.springframework.org/schema/aop>
		<https://www.springframework.org/schema/aop/spring-aop.xsd>">

	<bean class="org.springframework.beans.factory.config.CustomScopeConfigurer">
		<property name="scopes">
			<map>
				<entry key="thread">
					<bean class="org.springframework.context.support.SimpleThreadScope"/>
				</entry>
			</map>
		</property>
	</bean>

	<bean id="thing2" class="x.y.Thing2" scope="thread">
		<property name="name" value="Rick"/>
		<aop:scoped-proxy/>
	</bean>

	<bean id="thing1" class="x.y.Thing1">
		<property name="thing2" ref="thing2"/>
	</bean>

</beans>
```

`*FactoryBean` 구현에 대한 `<bean>` 선언 내에 `<aop:scoped-proxy/>`를 배치하면, `getObject()`에서 반환되는 객체가 아니라 팩토리 빈 자체가 스코프 지정됩니다.*

---

**1. 웹 애플리케이션과 서블릿 컨테이너(Servlet Container)란?**

- 우리가 만드는 웹사이트나 웹 서비스는 보통 **웹 서버(Web Server)** 위에서 돌아갑니다. 대표적인 예로 **톰캣(Tomcat)** 이나 제티(Jetty) 같은 것들이 있죠.
- 이런 서버들은 단순히 웹 페이지만 보여주는 것을 넘어, 자바 기반의 웹 애플리케이션을 실행하고 관리하는 특별한 환경을 제공하는데, 이를 **서블릿 컨테이너(Servlet Container)** 라고 부릅니다. (톰캣은 웹 서버 기능과 서블릿 컨테이너 기능을 모두 가지고 있습니다.)

**2. `ServletContext` 란 무엇인가?**

- 서블릿 컨테이너가 **하나의 웹 애플리케이션을 실행할 때**, 그 웹 애플리케이션 **전체**를 위한 **공용 저장소** 또는 **환경 정보**를 제공하는 객체가 있습니다. 이것이 바로 `ServletContext` 입니다.
- **하나의 웹 애플리케이션당 딱 하나**의 `ServletContext` 객체가 생성됩니다. 웹 애플리케이션이 시작될 때 만들어져서 종료될 때 사라집니다.
- **역할:**
  - 웹 애플리케이션 전체에서 공유해야 하는 정보나 객체를 저장하고 꺼내 쓸 수 있는 **공간**을 제공합니다. (마치 애플리케이션 전용 전역 변수 저장소 같은 느낌)
  - 웹 애플리케이션의 설정 정보(web.xml 등)를 읽어올 수 있습니다.
  - 서버 정보나 다른 웹 자원에 접근하는 통로가 되기도 합니다.
- **"ServletContext 레벨"**: 이 말은 어떤 것이 `ServletContext` 안에 저장되어서, **해당 웹 애플리케이션의 어디서든 접근 가능하고, 웹 애플리케이션 전체의 생명주기와 함께하는 범위**를 의미합니다.

**3. 스프링의 `application` 스코프**

- 이제 스프링 설정에서 `<bean ... scope="application"/>` 또는 `@ApplicationScope` 어노테이션을 사용하면 어떤 일이 벌어지는지 보겠습니다.
- **동작 방식:**
  1. 스프링 컨테이너는 해당 빈(`appPreferences` 예시)의 **인스턴스를 딱 하나만 생성**합니다.
  2. 중요한 점은, 스프링이 이 **하나의 인스턴스를 `ServletContext` 안에 속성(attribute)으로 저장**한다는 것입니다. 마치 `servletContext.setAttribute("appPreferences", theAppPreferencesInstance);` 와 같은 코드가 내부적으로 실행되는 셈입니다.
- **결과:**
  - `appPreferences` 빈은 이제 **`ServletContext` 레벨**에 존재하게 됩니다.
  - 이 빈은 웹 애플리케이션이 살아있는 동안 **계속 유일한 객체로 존재**하며, 웹 애플리케이션 내의 **어디서든** (스프링 컨테이너를 통하거나, 경우에 따라 `ServletContext`에서 직접 꺼내서) **동일한 인스턴스**를 참조할 수 있습니다.

**4. 스프링 싱글톤(Singleton) 스코프와의 차이점 (매우 중요!)**

스프링의 기본 스코프인 `singleton`과 `application` 스코프는 비슷해 보이지만 중요한 차이가 있습니다.

| 특징 | 싱글톤 (Singleton) 스코프 (기본) | 애플리케이션 (Application) 스코프 (`scope="application"`) |
| --- | --- | --- |
| **인스턴스 개수** | **하나의 스프링 컨테이너(ApplicationContext)당** 1개 | **하나의 웹 애플리케이션(ServletContext)당** 1개 |
| **저장 위치** | 스프링 컨테이너 내부 메모리 | `ServletContext`의 속성(attribute)으로 저장 |
| **가시성/노출** | 스프링 컨테이너 외부에서는 직접 접근 어려움 | `ServletContext`를 통해 접근 가능 (노출됨) |
| **웹 환경 의존성** | 웹 환경이 아니어도 사용 가능 | **반드시 웹 환경**이어야 함 (`ServletContext` 필요) |

**"두 가지 중요한 방식으로 다릅니다" 부분 설명:**

1. **"ApplicationContext당 싱글톤이 아니라 ServletContext당 싱글톤이다"**:
  - 하나의 웹 애플리케이션 안에도 **여러 개의 스프링 컨테이너(`ApplicationContext`)** 가 존재할 수 있습니다. (예: Root 컨텍스트, Servlet 컨텍스트 등).
  - 만약 어떤 빈이 그냥 `singleton` 스코프라면, 각 스프링 컨테이너마다 그 빈의 인스턴스가 하나씩 생길 수도 있습니다 (총 여러 개 존재 가능).
  - 하지만 `application` 스코프 빈은 **웹 애플리케이션 전체에 걸쳐 단 하나 존재하는 `ServletContext`를 기준으로** 딱 하나만 생성됩니다. 어떤 스프링 컨테이너에서 요청하든 항상 그 유일한 인스턴스를 참조하게 됩니다. 웹 애플리케이션 레벨에서 진정한 의미의 '단 하나'를 보장합니다.
2. **"실제로 노출되어 ServletContext 속성으로 보인다"**:
  - `singleton` 빈은 기본적으로 스프링 컨테이너가 내부적으로 관리하며, 외부에서 `ServletContext`를 통해 직접 꺼내 쓸 수는 없습니다.
  - `application` 스코프 빈은 스프링이 `ServletContext`에 `setAttribute`를 해주기 때문에, 스프링을 통하지 않고도 `ServletContext` 객체만 있다면 `getAttribute("빈이름")`으로 해당 빈 객체를 꺼내 쓸 수 있습니다. (물론 보통은 스프링의 DI를 통해 주입받아 사용합니다.)

**요약:**

`application` 스코프는 스프링 빈을 웹 애플리케이션 전체에서 유일하게 존재하고 어디서든 공유할 수 있도록, 웹 애플리케이션의 **공용 저장소인 `ServletContext`** 에 저장하는 방식입니다. 이는 스프링 컨테이너별로 싱글톤인 기본 스코프와 달리, 웹 애플리케이션 레벨에서의 싱글톤을 보장하며 `ServletContext`를 통해 외부에 노출된다는 차이점이 있습니다.

이 설명이 `ServletContext` 레벨과 `application` 스코프를 이해하는 데 도움이 되었기를 바랍니다!

---

Scope Bean 을 의존성으로 사용하기

**1. 문제 상황: 수명이 다른 빈들 간의 의존성**

- **빈의 수명(스코프):** 스프링 빈은 각자 다른 '수명'을 가집니다.
  - **싱글톤(Singleton):** 애플리케이션 전체에 딱 하나 존재하며, 애플리케이션 시작부터 끝까지 살아있습니다. (가장 긴 수명)
  - **세션(Session):** 각 사용자(웹 브라우저 세션)마다 하나씩 존재하며, 해당 사용자의 세션이 유지되는 동안 살아있습니다.
  - **리퀘스트(Request):** 각 웹 요청마다 하나씩 존재하며, 그 요청이 처리되는 동안만 살아있습니다. (가장 짧은 수명)
  - **프로토타입(Prototype):** 요청할 때마다 매번 새로운 객체가 만들어집니다.
- **문제:** **수명이 긴 빈(예: 싱글톤 `userService`)** 이 **수명이 짧은 빈(예: 세션 스코프 `userPreferences`)** 을 필요로 할 때 문제가 발생합니다.
  - 싱글톤 `userService` 빈은 애플리케이션 시작 시 **딱 한 번** 만들어지고, 이때 필요한 의존성(`userPreferences`)도 **딱 한 번** 주입받습니다.
  - 만약 이때 그냥 세션 스코프 `userPreferences` 빈을 주입하면, `userService`는 **자신이 만들어질 때 활성화되어 있던 (또는 처음으로 생성된) 딱 그 하나의 `userPreferences` 객체만 계속** 가지게 됩니다.
  - **이러면 안 됩니다!** `userService`는 여러 사용자가 동시에 사용할 수 있는 싱글톤 객체인데, 각 사용자의 요청을 처리할 때는 **그 사용자의 세션에 맞는 `userPreferences`** 를 사용해야 합니다.
    A 사용자의 요청을 처리할 때는 A의 `userPreferences`를, B 사용자의 요청을 처리할 때는 B의 `userPreferences`를 써야 하죠. 그런데 처음 주입받은 것만 계속 쓰면 잘못된 정보(다른 사용자의 정보)를 사용하게 될 수 있습니다.

**2. 해결책: 스코프 프록시 (`<aop:scoped-proxy/>`)**

이 문제를 해결하기 위해 스프링은 **"스코프 프록시(Scoped Proxy)"** 라는 아주 영리한 방법을 제공합니다.

- **아이디어:** 수명이 긴 빈(`userService`)에게 실제 수명이 짧은 빈(`userPreferences`)을 직접 주입하는 대신, **가짜(대리인) 객체인 프록시(Proxy)** 를 주입합니다.
- **프록시의 역할:**
  1. 이 프록시 객체는 겉보기에는 **실제 `UserPreferences` 객체와 똑같이 생겼습니다.** (같은 메소드를 가지고 있음)
  2. `userService`는 자신이 받은 것이 진짜 `userPreferences`인지 프록시인지 **전혀 모릅니다.** 그냥 평소처럼 그 객체의 메소드를 호출합니다.
  3. `userService`가 프록시 객체의 메소드를 호출하면, **그때 프록시가 진짜 일을 시작합니다:**
    - 프록시는 현재 요청이 어떤 사용자의 세션(또는 어떤 웹 요청)에서 온 것인지 **파악합니다.**
    - 현재 세션(또는 요청)에 **해당하는 실제 `userPreferences` 객체**를 스프링 컨테이너로부터 **찾아옵니다.** (만약 없으면 새로 만듭니다).
    - 찾아온 **진짜 `userPreferences` 객체에게 원래 요청했던 메소드 호출을 대신 전달(위임)** 합니다.
    - 그 결과를 다시 `userService`에게 돌려줍니다.
- **결과:** `userService`는 항상 자신에게 주입된 (프록시) 객체만 사용하지만, 그 프록시가 뒤에서 똑똑하게 움직여서 **매번 현재 상황에 맞는 진짜 `userPreferences` 객체**를 찾아 연결해주기 때문에, 결과적으로 `userService`는 항상 올바른 사용자의 정보를 사용하게 됩니다.

**3. 설정 방법 (`<aop:scoped-proxy/>`)**

XML 설정에서 이 프록시를 사용하라고 스프링에게 알려주는 방법은 아주 간단합니다. 수명이 짧은 빈(세션, 리퀘스트 등)을 정의할 때, 그 안에 **`<aop:scoped-proxy/>`** 라는 자식 태그를 한 줄 추가해주면 됩니다.

```xml
<!-- 세션 스코프 빈 정의 -->
<bean id="userPreferences" class="com.something.UserPreferences" scope="session">
    <!-- ★★★ 이 빈을 주입할 때는 실제 객체 대신 프록시를 주입해줘! ★★★ -->
    <aop:scoped-proxy/>
</bean>

<!-- 싱글톤 스코프 빈 정의 -->
<bean id="userService" class="com.something.SimpleUserService">
    <!-- userPreferences를 주입받지만, 실제로는 프록시 객체가 주입됨 -->
    <property name="userPreferences" ref="userPreferences"/>
</bean>
```

- 이렇게 하면, 스프링은 `userPreferences` 빈 대신 그 빈을 위한 **프록시 객체**를 생성합니다. 그리고 `userService`가 `userPreferences`를 요청하면, 스프링은 이 **프록시 객체**를 주입해줍니다.
- `<aop:scoped-proxy/>`는 내부적으로 AOP(관점 지향 프로그래밍) 기술과 프록시 패턴을 사용하여 이 마법 같은 일을 처리합니다.

**4. 다른 스코프에서의 사용**

- **프로토타입 스코프:** `prototype` 빈에 `<aop:scoped-proxy/>`를 사용하면, 프록시의 메소드를 호출할 때마다 **매번 새로운 `prototype` 빈 인스턴스**가 생성되어 사용됩니다. (룩업 메소드 주입과 유사한 효과)
- **싱글톤 스코프:** 싱글톤 빈끼리도 `<aop:scoped-proxy/>`를 사용할 수 있습니다. 이때는 약간 다른 목적인데, 객체를 직렬화(serialize)했다가 역직렬화(deserialize)할 때 프록시를 통해 다시 올바른 싱글톤 빈을 안전하게 참조할 수 있게 도와주는 역할을 하기도 합니다. (일반적인 사용 사례는 아님)

**5. 프록시 외 다른 방법 (간단 언급)**

스코프 프록시가 가장 일반적이지만, 비슷한 문제를 해결하는 다른 방법도 있습니다.

- **`ObjectFactory<T>`, `ObjectProvider<T>`, `Provider<T>` (JSR-330):**
  - 수명이 짧은 빈 자체를 직접 주입받는 대신, 그 빈을 **만들어주는 공장(Factory)이나 제공자(Provider)** 객체를 주입받는 방식입니다.
  - `private ObjectProvider<UserPreferences> userPreferencesProvider;` 와 같이 선언하고,
  - 실제로 `UserPreferences` 객체가 필요한 시점에 `userPreferencesProvider.getObject()` (또는 `.getIfAvailable()`) 메소드를 **호출**하여 현재 스코프에 맞는 객체를 얻어옵니다.
  - 프록시 방식보다 코드가 약간 더 복잡해질 수 있지만, 프록시 생성에 따른 오버헤드가 없다는 장점이 있을 수 있습니다.

`<aop:scoped-proxy/>`

1. **`<aop:scoped-proxy/>`:** 이 태그는 스프링 컨테이너에게 보내는 특별한 지시어입니다. "이 빈(`userPreferences`)을 위한 **프록시(대리인) 객체를 자동으로 만들어줘!**" 라고 말하는 것과 같습니다.
2. **스프링의 AOP와 동적 프록시 기술:** 스프링은 이 지시를 받으면 내부적으로 **AOP(관점 지향 프로그래밍)** 개념과 **동적 프록시(Dynamic Proxy)** 생성 기술을 사용합니다.
   핵심은 개발자가 프록시 클래스 코드를 직접 작성하지 않아도, 스프링이 **런타임(애플리케이션 실행 중)** 에 **자동으로 프록시 클래스를 만들고 그 객체를 생성**한다는 것입니다.
3. **어떤 기술을 사용할까? (두 가지 주요 방법)**
  - **JDK 동적 프록시 (기본 선호):** 만약 원본 클래스(`com.something.UserPreferences`)가 **하나 이상의 인터페이스를 구현(implements)** 하고 있다면, 스프링은 JDK(자바 개발 키트)에서 제공하는 동적 프록시 생성 방식을 사용합니다.
    - **작동 방식:** 스프링은 원본 클래스가 구현한 **모든 인터페이스를 똑같이 구현하는 새로운 프록시 클래스를 런타임에 생성**합니다. 이 프록시 객체는 원본 클래스와 같은 타입(인터페이스 타입)으로 취급될 수 있습니다.
    - **예시:** `UserPreferences`가 `Preferences` 인터페이스를 구현했다면, 프록시도 `Preferences` 인터페이스를 구현합니다.
  - **CGLIB 라이브러리:** 만약 원본 클래스가 **인터페이스를 구현하지 않은 일반 클래스**라면, 스프링은 CGLIB이라는 외부 라이브러리를 사용합니다.
    - **작동 방식:** CGLIB은 원본 클래스(`UserPreferences`)를 **상속(extends)하는 새로운 자식 클래스를 런타임에 생성**합니다. 이 자식 클래스가 프록시 역할을 합니다.
    - **제약:** 이 방식이 작동하려면 원본 클래스나 프록시가 오버라이드해야 하는 메소드가 `final`로 선언되어 있으면 안 됩니다. (상속/오버라이드가 불가능하므로)
4. **자동 생성된 프록시 객체의 내용물 (개념적으로):**
   스프링이 자동으로 만든 프록시 객체는 대략 다음과 같은 일을 하도록 프로그래밍되어 있습니다. (실제 코드는 더 복잡합니다)
  - **겉모습은 원본과 동일:** 원본 클래스가 구현한 인터페이스를 똑같이 구현하거나, 원본 클래스를 상속받았기 때문에, 외부에서 볼 때는 원본 `UserPreferences` 객체와 구분할 수 없습니다. (`userService`가 주입받을 때 타입 문제가 발생하지 않습니다.)
  - **실제 객체 찾는 로직 내장:** 이 프록시 객체는 내부에 "현재 요청에 맞는 실제 `UserPreferences` 객체를 어떻게 찾아야 하는지" 아는 로직을 가지고 있습니다. (예: 현재 HTTP 세션을 가져와서 그 세션에 저장된 `userPreferences` 빈을 스프링 컨테이너에 요청하는 방법)
  - **메소드 호출 가로채기 및 위임:** 누군가 이 프록시 객체의 메소드(예: `getTheme()`)를 호출하면, 프록시는 이 호출을 가로챕니다. 그리고 내장된 로직을 사용해 현재 스코프(여기서는 세션)에 맞는 **실제 `UserPreferences` 객체를 찾습니다.** 찾은 실제 객체에게 **원래 요청받았던 `getTheme()` 메소드 호출을 대신 전달(위임)** 하고, 그 결과를 받아 호출자에게 반환합니다.

Custom Scope

**1. 핵심 아이디어: 나만의 빈 수명 규칙 만들기**

- 스프링은 기본적으로 여러 종류의 빈 스코프(수명 규칙)를 제공합니다:
  - `singleton`: 딱 하나 만들어서 계속 씀
  - `prototype`: 요청할 때마다 새로 만듦
  - `request`: 웹 요청마다 새로 만듦
  - `session`: 웹 세션마다 새로 만듦
  - `application`: 웹 애플리케이션 전체에 하나 만듦
- **그런데 만약, 이런 기본 규칙들로는 내가 원하는 동작을 만들 수 없다면 어떡할까요?** 예를 들어, 다음과 같은 특별한 규칙이 필요할 수 있습니다:
  - **스레드(Thread) 스코프:** 각 스레드마다 빈 인스턴스를 따로 가지고 싶다. (여러 스레드가 동시에 작업할 때 유용)
  - **특정 비즈니스 프로세스 스코프:** 어떤 긴 작업(예: 주문 처리 과정)이 시작될 때 빈을 만들고, 그 작업이 끝날 때까지만 유지하고 싶다.
- 이럴 때 사용하는 것이 **커스텀 스코프**입니다. 즉, 스프링의 스코프 메커니즘은 **확장 가능(extensible)** 해서, 개발자가 **자신만의 새로운 빈 수명 규칙(스코프)을 정의**해서 사용할 수 있다는 뜻입니다.

**2. 커스텀 스코프는 어떻게 만드나? (`Scope` 인터페이스)**

- 새로운 스코프 규칙을 만들려면, 스프링이 정해놓은 **"설계도" 또는 "계약서"** 인 `org.springframework.beans.factory.config.Scope` 인터페이스를 따라야 합니다.
- 개발자는 이 `Scope` 인터페이스를 **구현(implements)** 하는 새로운 클래스를 만듭니다. 이 클래스가 바로 우리가 만들 커스텀 스코프의 **동작 방식**을 정의하는 곳입니다.
- `Scope` 인터페이스에는 구현해야 할 중요한 메소드가 4가지 있습니다. 이 메소드들을 어떻게 구현하느냐에 따라 커스텀 스코프가 어떻게 동작할지가 결정됩니다.

**3. `Scope` 인터페이스의 4가지 핵심 메소드 (각각의 역할)**

스프링 컨테이너는 빈을 관리할 때, 특정 빈이 커스텀 스코프에 해당하면 우리가 만든 `Scope` 구현 클래스의 이 메소드들을 호출해서 빈을 관리합니다.

- **`Object get(String name, ObjectFactory<?> objectFactory)`**
  - **역할:** "이 스코프에서 `name`이라는 이름의 빈 객체를 **내놔!**" 라는 요청을 처리하는 메소드입니다.
  - **동작 방식 (구현 내용):**
    1. 현재 커스텀 스코프의 "저장 공간"(예: 스레드 로컬 변수, 특정 맵 등)에서 `name`에 해당하는 객체가 **이미 있는지 확인**합니다.
    2. **만약 있다면:** 그 객체를 즉시 반환합니다.
    3. **만약 없다면:**
      - 함께 전달된 `objectFactory` (객체 생성기)를 사용해서 **새로운 빈 객체를 생성**합니다. (`objectFactory.getObject()` 호출)
      - 생성된 새 객체를 현재 커스텀 스코프의 **"저장 공간"에 보관**합니다. (나중에 다시 찾을 수 있도록)
      - 새로 생성하고 보관한 객체를 **반환**합니다.
  - *비유:* 도서관 사서에게 특정 책(`name`)을 달라고 요청하는 것과 비슷합니다. 사서는 서가(스코프 저장 공간)를 확인해서 책이 있으면 바로 주고, 없으면 새로 주문해서(`objectFactory`) 서가에 꽂아놓은 뒤(`스코프에 보관`) 빌려줍니다(`반환`).
- **`Object remove(String name)`**
  - **역할:** "이 스코프에서 `name`이라는 이름의 빈 객체를 **제거해줘!**" 라는 요청을 처리합니다. (더 이상 사용하지 않거나 스코프가 끝날 때)
  - **동작 방식 (구현 내용):**
    1. 현재 커스텀 스코프의 "저장 공간"에서 `name`에 해당하는 객체를 찾습니다.
    2. **찾았다면:** 저장 공간에서 **그 객체를 제거**하고, 제거된 객체를 **반환**합니다.
    3. **못 찾았다면:** `null`을 반환할 수 있습니다.
  - *비유:* 도서관에 빌렸던 책(`name`)을 반납하면, 사서가 그 책을 서가(`스코프 저장 공간`)에서 빼내는 것과 비슷합니다.
- **`void registerDestructionCallback(String name, Runnable callback)`**
  - **역할:** "나중에 이 스코프가 없어지거나 `name` 빈이 파괴될 때, **이 작업(`callback`)을 꼭 실행해줘!**" 라고 **예약**하는 메소드입니다. (주로 빈이 사용하던 리소스를 해제하는 등 뒷정리 작업을 등록할 때 사용)
  - **동작 방식 (구현 내용):**
    1. `name` 빈과 그 빈이 소멸될 때 실행해야 할 작업(`callback`, 보통 `Runnable` 객체)을 내부적으로 **기록**해 둡니다.
    2. 나중에 스코프 자체가 소멸되거나 `remove()` 메소드 등으로 빈이 제거될 때, 기록해 둔 해당 `callback`을 실행시켜 줍니다.
  - *비유:* 도서관에 책을 기증하면서 "이 책이 폐기될 때는 꼭 이 메모(`callback`)에 적힌 대로 처리해주세요"라고 부탁하는 것과 같습니다.
- **`String getConversationId()`**
  - **역할:** 현재 활성화된 스코프 **인스턴스**를 식별할 수 있는 **고유한 ID(이름표)** 를 반환합니다.
  - **동작 방식 (구현 내용):**
    1. 현재 스코프를 나타내는 고유 ID를 반환합니다. 예를 들어,
      - 세션 스코프라면 현재 세션 ID를 반환할 수 있습니다.
      - 스레드 스코프라면 현재 스레드 ID를 반환할 수 있습니다.
      - 만약 스코프가 딱 하나만 존재하는 개념이라면 (고정된 ID) `null`을 반환할 수도 있습니다.
  - *비유:* 도서관의 특정 열람실 번호나 회원 카드 번호처럼, 현재 내가 속한 스코프의 고유 식별 정보를 알려주는 것입니다. (로깅이나 추적 등에 사용될 수 있음)

**4. 커스텀 스코프 사용하기 (중요!)**

- 이렇게 `Scope` 인터페이스를 구현한 클래스를 만들었다고 해서 바로 쓸 수 있는 것은 아닙니다.
- 만든 커스텀 스코프를 **스프링 컨테이너에 등록**하는 과정이 추가로 필요합니다. (예: `CustomScopeConfigurer` 사용) 등록해야 스프링이 "아, 'thread'라는 이름의 스코프는 이 클래스가 처리하는 거구나!" 하고 인지할 수 있습니다.
- 등록 후에는 XML이나 어노테이션에서 `<bean scope="나만의스코프이름"/>` 처럼 사용할 수 있게 됩니다.

**요약:**

커스텀 스코프는 스프링의 기본 빈 수명 규칙(스코프) 외에 **개발자가 직접 새로운 규칙을 정의**할 수 있게 해주는 확장 기능입니다. 이를 위해 `Scope` 인터페이스를 구현해야 하며, 이 인터페이스의 `get`, `remove`, `registerDestructionCallback`, `getConversationId` 메소드를 통해 **"언제 빈을 만들고, 어떻게 보관하고, 언제 제거하고, 어떻게 식별할지"** 등의 구체적인 동작 방식을 정의하게 됩니다.

ConversationId

**핵심 아이디어: 현재 활성화된 스코프 인스턴스에 "이름표" 붙여주기**

- **스코프 인스턴스:** 스프링 스코프는 '규칙'이지만, 실제로 애플리케이션이 실행될 때는 그 규칙에 따라 여러 개의 **스코프 인스턴스(실체)** 가 동시에 존재할 수 있습니다.
  - 예를 들어, `session` 스코프는 규칙이지만, 사용자가 10명 접속해 있다면 **10개의 서로 다른 세션 스코프 인스턴스**가 메모리에 동시에 존재합니다. 각 사용자의 세션이 바로 스코프 인스턴스입니다.
  - `request` 스코프라면, 동시에 100개의 웹 요청이 들어오면 **100개의 서로 다른 리퀘스트 스코프 인스턴스**가 잠시 동안 존재합니다.
  - 만약 여러분이 '스레드(thread)' 스코프를 직접 만들었다면, 여러 스레드가 동시에 실행될 때 각 스레드가 자신만의 스레드 스코프 인스턴스를 가집니다.
- **`getConversationId()`의 역할:** 이 메소드는 현재 코드가 실행되고 있는 **바로 그 스코프 인스턴스**를 다른 인스턴스들과 **구별**할 수 있는 **고유한 식별자(ID)**, 즉 "이름표"를 반환하는 역할을 합니다.

**"대화 식별자 (Conversation Identifier)" 라는 이름의 의미**

- "Conversation(대화)"이라는 단어가 약간 혼란을 줄 수 있습니다. 꼭 사용자와 시스템 간의 긴 대화를 의미하는 것은 아닙니다.
- 여기서는 좀 더 넓은 의미로, **하나의 독립적인 컨텍스트(context) 또는 작업 단위**를 나타낸다고 생각하면 좋습니다.
  - **웹 세션:** 한 사용자가 로그인해서 로그아웃할 때까지의 연속적인 상호작용 = 하나의 "대화" 단위 = 고유한 세션 ID로 식별 가능.
  - **웹 요청:** 클라이언트가 서버에 요청을 보내고 응답을 받을 때까지의 짧은 상호작용 = 하나의 "대화" 단위 = 고유한 요청 ID (있다면)로 식별 가능.
  - **스레드:** 하나의 스레드가 시작해서 끝날 때까지 실행되는 작업 흐름 = 하나의 "대화" 단위 = 고유한 스레드 ID로 식별 가능.
- 즉, `getConversationId()`는 현재 내가 속한 **스코프 '대화' (또는 컨텍스트, 인스턴스)의 고유 ID**를 알려달라는 요청입니다.

**"이 식별자는 각 스코프마다 다릅니다."**

- 이 말은 각 **활성화된 스코프 인스턴스마다** 다른 ID를 가져야 한다는 뜻입니다.
- 내 세션 ID는 당신의 세션 ID와 달라야 합니다. 이 스레드의 ID는 저 스레드의 ID와 달라야 합니다. 그래야 서로 구별이 가능하겠죠.

**"session 스코프 구현의 경우 이 식별자는 세션 식별자일 수 있습니다."**

- 이것은 아주 좋은 예시입니다. 웹 애플리케이션에서 각 사용자 세션은 이미 고유한 ID(예: `JSESSIONID` 값)를 가지고 있습니다.
- 따라서 `session` 스코프를 구현한다면, `getConversationId()` 메소드가 그냥 **현재 HTTP 세션의 ID를 반환하도록** 만들면 가장 자연스럽고 쉽습니다. 이미 존재하는 고유 ID를 활용하는 것이죠.

**왜 이 ID가 필요할까?**

- **로깅 및 디버깅:** 어떤 로그 메시지나 오류가 **어떤 세션, 어떤 요청, 어떤 스레드**에서 발생했는지 추적할 때 유용합니다. ID를 함께 기록하면 문제 파악이 쉬워집니다.
- **고급 기능:** 경우에 따라 특정 스코프 인스턴스와 관련된 데이터를 관리하거나, 같은 스코프 내의 여러 객체 간의 상호작용을 조정해야 할 때 이 ID를 사용할 수도 있습니다. (흔한 경우는 아닙니다.)
- **필수는 아님:** 모든 커스텀 스코프가 반드시 의미 있는 `ConversationId`를 가질 필요는 없습니다. 예를 들어, 스코프 인스턴스가 개념적으로 딱 하나만 존재한다면 (예: 애플리케이션 전체 프로세스 스코프), ID가 필요 없을 수도 있고 이때는 `null`을 반환할 수도 있습니다. (코틀린 반환 타입이 `String?`인 이유)

**요약:**

`getConversationId()` 메소드는 현재 실행 컨텍스트가 속해 있는 **스코프 인스턴스(예: 현재 세션, 현재 스레드)의 고유한 식별자(이름표)** 를 반환하는 역할을 합니다. 이 ID는 여러 개의 활성화된 스코프 인스턴스들을 서로 구별하는 데 사용될 수 있으며, 특히 로깅이나 디버깅에 유용합니다. 세션 스코프의 경우, 세션 ID를 이 메소드의 반환값으로 사용하는 것이 자연스러운 예시입니다.
