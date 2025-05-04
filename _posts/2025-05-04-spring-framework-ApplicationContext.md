---
title: Spring Framework ApplicationContext
description: 
author: laze
date: 2025-05-04 00:00:11 +0900
categories: [Dev, SpringBoot]
tags: [SpringBoot]
---
**Additional Capabilities of the ApplicationContext**

`org.springframework.beans.factory` 패키지는 프로그래밍 방식을 포함하여 빈을 관리하고 조작하기 위한 기본 기능을 제공합니다.

`org.springframework.context` 패키지는 `BeanFactory` 인터페이스를 확장하는 `ApplicationContext` 인터페이스를 추가하며, 더 나아가 다른 인터페이스들을 확장하여 더 애플리케이션 프레임워크 지향적인 스타일로 추가 기능을 제공합니다.

많은 사람들이 `ApplicationContext`를 완전히 선언적인 방식으로 사용하며, 심지어 프로그래밍 방식으로 생성하지 않고,

대신 Jakarta EE 웹 애플리케이션의 일반적인 시작 프로세스의 일부로 `ApplicationContext`를 자동으로 인스턴스화하는 `ContextLoader`와 같은 지원 클래스에 의존합니다.

더 프레임워크 지향적인 스타일로 `BeanFactory` 기능을 향상시키기 위해, context 패키지는 다음 기능도 제공합니다:

- `MessageSource` 인터페이스를 통한 i18n 스타일의 메시지 접근.
- `ResourceLoader` 인터페이스를 통한 URL 및 파일과 같은 리소스 접근.
- `ApplicationEventPublisher` 인터페이스 사용을 통한, `ApplicationListener` 인터페이스를 구현하는 빈에 대한 이벤트 발행.
- `HierarchicalBeanFactory` 인터페이스를 통한, 애플리케이션의 웹 계층과 같은 특정 계층에 각각 집중할 수 있는 다중 (계층적) 컨텍스트 로딩.

**Internationalization using MessageSource**

`ApplicationContext` 인터페이스는 `MessageSource`라는 인터페이스를 확장하므로 Internationalization("i18n") 기능을 제공합니다.

스프링은 또한 메시지를 계층적으로 해석(resolve)할 수 있는 `HierarchicalMessageSource` 인터페이스를 제공합니다.

함께, 이 인터페이스들은 스프링이 메시지 해석을 수행하는 기반을 제공합니다.

이 인터페이스들에 정의된 메소드는 다음과 같습니다:

- `String getMessage(String code, Object[] args, String default, Locale loc)`: `MessageSource`에서 메시지를 검색하는 데 사용되는 기본 메소드. 지정된 로케일(locale)에 대한 메시지를 찾을 수 없는 경우 기본 메시지가 사용됩니다. 전달된 모든 인수는 표준 라이브러리에서 제공하는 `MessageFormat` 기능을 사용하여 대체 값이 됩니다.
- `String getMessage(String code, Object[] args, Locale loc)`: 이전 메소드와 본질적으로 동일하지만 한 가지 차이점이 있습니다: 기본 메시지를 지정할 수 없습니다. 메시지를 찾을 수 없으면 `NoSuchMessageException`이 발생합니다.
- `String getMessage(MessageSourceResolvable resolvable, Locale locale)`: 이전 메소드에서 사용된 모든 속성은 `MessageSourceResolvable`이라는 클래스에 래핑(wrapped)되어 있으며, 이 메소드와 함께 사용할 수 있습니다.

`ApplicationContext`가 로드될 때, 컨텍스트에 정의된 `MessageSource` 빈을 자동으로 검색합니다.

빈의 이름은 `messageSource`여야 합니다. 이러한 빈이 발견되면, 이전 메소드에 대한 모든 호출은 메시지 소스에 위임됩니다.

메시지 소스를 찾을 수 없는 경우, `ApplicationContext`는 동일한 이름의 빈을 포함하는 부모를 찾으려고 시도합니다.

찾으면 해당 빈을 `MessageSource`로 사용합니다.

`ApplicationContext`가 메시지에 대한 소스를 찾을 수 없는 경우, 위에서 정의된 메소드 호출을 받을 수 있도록 빈 `DelegatingMessageSource`가 인스턴스화됩니다.

스프링은 세 가지 `MessageSource` 구현, `ResourceBundleMessageSource`, `ReloadableResourceBundleMessageSource` 및 `StaticMessageSource`를 제공합니다.

이들 모두는 중첩된 메시징(nested messaging)을 수행하기 위해 `HierarchicalMessageSource`를 구현합니다.

`StaticMessageSource`는 거의 사용되지 않지만 프로그래밍 방식으로 소스에 메시지를 추가하는 방법을 제공합니다. 다음 예제는 `ResourceBundleMessageSource`를 보여줍니다:

```xml
<beans>
	<bean id="messageSource"
			class="org.springframework.context.support.ResourceBundleMessageSource">
		<property name="basenames">
			<list>
				<value>format</value>
				<value>exceptions</value>
				<value>windows</value>
			</list>
		</property>
	</bean>
</beans>
```

이 예제는 클래스패스에 `format`, `exceptions`, `windows`라는 세 개의 리소스 번들(resource bundles)이 정의되어 있다고 가정합니다.

메시지 해석 요청은 `ResourceBundle` 객체를 통해 메시지를 해석하는 JDK 표준 방식으로 처리됩니다.

예제의 목적상, 위의 리소스 번들 파일 중 두 개의 내용은 다음과 같다고 가정합니다:

```
# format.properties 에서
message=Alligators rock!
```

```
# exceptions.properties 에서
argument.required=The {0} argument is required.
```

다음 예제는 `MessageSource` 기능을 실행하는 프로그램을 보여줍니다.

모든 `ApplicationContext` 구현은 `MessageSource` 구현이기도 하므로 `MessageSource` 인터페이스로 캐스팅될 수 있음을 기억하십시오.

```java
// Java
public static void main(String[] args) {
	MessageSource resources = new ClassPathXmlApplicationContext("beans.xml");
	String message = resources.getMessage("message", null, "Default", Locale.ENGLISH);
	System.out.println(message);
}
```

```kotlin
// Kotlin
fun main(args: Array<String>) {
    val resources: MessageSource = ClassPathXmlApplicationContext("beans.xml")
    val message = resources.getMessage("message", null, "Default", Locale.ENGLISH)
    println(message)
}
```

위 프로그램의 결과 출력은 다음과 같습니다:

```
Alligators rock!
```

요약하자면, `MessageSource`는 클래스패스의 루트에 존재하는 `beans.xml`이라는 파일에 정의됩니다.

`messageSource` 빈 정의는 `basenames` 속성을 통해 여러 리소스 번들을 참조합니다.

`basenames` 속성의 리스트에 전달된 세 개의 파일은 각각 클래스패스의 루트에 파일로 존재하며 이름은 `format.properties`, `exceptions.properties`, `windows.properties`입니다.

다음 예제는 메시지 조회에 전달되는 인수를 보여줍니다. 이 인수들은 `String` 객체로 변환되어 조회 메시지의 플레이스홀더에 삽입됩니다.

```xml
<beans>

	<!-- 이 MessageSource는 웹 애플리케이션에서 사용됨 -->
	<bean id="messageSource" class="org.springframework.context.support.ResourceBundleMessageSource">
		<property name="basename" value="exceptions"/>
	</bean>

	<!-- 위의 MessageSource를 이 POJO에 주입 -->
	<bean id="example" class="com.something.Example">
		<property name="messages" ref="messageSource"/>
	</bean>

</beans>
```

```java
// Java
public class Example {

	private MessageSource messages;

	public void setMessages(MessageSource messages) {
		this.messages = messages;
	}

	public void execute() {
		String message = this.messages.getMessage("argument.required",
			new Object [] {"userDao"}, "Required", Locale.ENGLISH);
		System.out.println(message);
	}
}
```

```kotlin
// Kotlin
class Example {
    private var messages: MessageSource? = null

    fun setMessages(messages: MessageSource) {
        this.messages = messages
    }

    fun execute() {
        val message = this.messages!!.getMessage(
            "argument.required",
            arrayOf<Any>("userDao"), // Need arrayOf for varargs
            "Required",
            Locale.ENGLISH
        )
        println(message)
    }
}
```

`execute()` 메소드 호출의 결과 출력은 다음과 같습니다:

```
The userDao argument is required.

```

Internationalization("i18n")과 관련하여, 스프링의 다양한 `MessageSource` 구현은 표준 JDK `ResourceBundle`과 동일한 로케일 해석 및 폴백(fallback) 규칙을 따릅니다.

간단히 말해서, 앞서 정의된 예제 `messageSource`를 계속 사용하여 영국(en-GB) 로케일에 대해 메시지를 해석하려면 각각 `format_en_GB.properties`, `exceptions_en_GB.properties`, `windows_en_GB.properties`라는 파일을 생성하면 됩니다.

일반적으로 로케일 해석은 애플리케이션의 주변 환경에 의해 관리됩니다.

다음 예제에서 (영국) 메시지가 해석되는 로케일은 수동으로 지정됩니다:

```
# exceptions_en_GB.properties 에서
argument.required=Ebagum lad, the ''{0}'' argument is required, I say, required.
```

```java
// Java
public static void main(final String[] args) {
	MessageSource resources = new ClassPathXmlApplicationContext("beans.xml");
	String message = resources.getMessage("argument.required",
		new Object [] {"userDao"}, "Required", Locale.UK);
	System.out.println(message);
}
```

```kotlin
// Kotlin
fun main(args: Array<String>) {
    val resources: MessageSource = ClassPathXmlApplicationContext("beans.xml")
    val message = resources.getMessage(
        "argument.required",
        arrayOf<Any>("userDao"),
        "Required",
        Locale.UK
    )
    println(message)
}
```

위 프로그램 실행의 결과 출력은 다음과 같습니다:

```
Ebagum lad, the 'userDao' argument is required, I say, required.
```

정의된 모든 `MessageSource`에 대한 참조를 얻기 위해 `MessageSourceAware` 인터페이스를 사용할 수도 있습니다.

`MessageSourceAware` 인터페이스를 구현하는 `ApplicationContext`에 정의된 모든 빈은 빈이 생성되고 구성될 때 애플리케이션 컨텍스트의 `MessageSource`로 주입됩니다.

*스프링의 `MessageSource`는 자바의 `ResourceBundle`을 기반으로 하므로 동일한 기본 이름을 가진 번들을 병합하지 않고 찾은 첫 번째 번들만 사용합니다.*

*동일한 기본 이름을 가진 후속 메시지 번들은 무시됩니다.*`ResourceBundleMessageSource`의 대안으로, 스프링은 `ReloadableResourceBundleMessageSource` 클래스를 제공합니다.

이 변형은 동일한 번들 파일 형식을 지원하지만 표준 JDK 기반 `ResourceBundleMessageSource` 구현보다 더 유연합니다.

특히, 모든 스프링 리소스 위치(클래스패스뿐만 아니라)에서 파일을 읽을 수 있으며 번들 프로퍼티 파일의 핫 리로딩(hot reloading)을 지원합니다 (그 사이에 효율적으로 캐싱하면서).

**표준 및 커스텀 이벤트 (Standard and Custom Events)**

`ApplicationContext`에서의 이벤트 처리는 `ApplicationEvent` 클래스와 `ApplicationListener` 인터페이스를 통해 제공됩니다.

`ApplicationListener` 인터페이스를 구현하는 빈이 컨텍스트에 배포되면, `ApplicationEvent`가 `ApplicationContext`에 게시(published)될 때마다 해당 빈이 알림을 받습니다.

본질적으로 이것은 표준 옵저버(Observer) 디자인 패턴입니다.

*스프링 4.2부터 이벤트 인프라가 크게 개선되었으며 어노테이션 기반 모델과 임의의 이벤트(즉, 반드시 `ApplicationEvent`에서 확장되지 않은 객체)를 게시하는 기능을 제공합니다.*

*이러한 객체가 게시되면, 그것을 이벤트로 래핑합니다.*

다음 표는 스프링이 제공하는 표준 이벤트를 설명합니다:

표 1. 내장 이벤트

| 이벤트 | 설명 |
| --- | --- |
| `ContextRefreshedEvent` | `ApplicationContext`가 초기화되거나 리프레시될 때 게시됩니다 (예: `ConfigurableApplicationContext` 인터페이스의 `refresh()` 메소드 사용). 여기서 "초기화됨"은 모든 빈이 로드되고, 후처리기 빈이 감지 및 활성화되고, 싱글톤이 사전 인스턴스화되고, `ApplicationContext` 객체가 사용 준비됨을 의미합니다. 컨텍스트가 닫히지 않은 한, 선택한 `ApplicationContext`가 실제로 이러한 "핫" 리프레시를 지원한다면 리프레시를 여러 번 트리거할 수 있습니다. 예를 들어, `XmlWebApplicationContext`는 핫 리프레시를 지원하지만 `GenericApplicationContext`는 지원하지 않습니다. |
| `ContextStartedEvent` | `ConfigurableApplicationContext` 인터페이스의 `start()` 메소드를 사용하여 `ApplicationContext`가 시작될 때 게시됩니다. 여기서 "시작됨"은 모든 `Lifecycle` 빈이 명시적인 시작 신호를 받음을 의미합니다. 일반적으로 이 신호는 명시적인 중지 후 빈을 재시작하는 데 사용되지만, 자동 시작(autostart)으로 구성되지 않은 컴포넌트(예: 초기화 시 이미 시작되지 않은 컴포넌트)를 시작하는 데 사용될 수도 있습니다. |
| `ContextStoppedEvent` | `ConfigurableApplicationContext` 인터페이스의 `stop()` 메소드를 사용하여 `ApplicationContext`가 중지될 때 게시됩니다. 여기서 "중지됨"은 모든 `Lifecycle` 빈이 명시적인 중지 신호를 받음을 의미합니다. 중지된 컨텍스트는 `start()` 호출을 통해 재시작될 수 있습니다. |
| `ContextClosedEvent` | `ConfigurableApplicationContext` 인터페이스의 `close()` 메소드를 사용하거나 JVM 종료 훅(shutdown hook)을 통해 `ApplicationContext`가 닫힐 때 게시됩니다. 여기서 "닫힘"은 모든 싱글톤 빈이 소멸될 것임을 의미합니다. 컨텍스트가 닫히면 수명이 다하여 리프레시하거나 재시작할 수 없습니다. |
| `RequestHandledEvent` | HTTP 요청이 서비스되었음을 모든 빈에게 알리는 웹 특정 이벤트입니다. 이 이벤트는 요청이 완료된 후 게시됩니다. 이 이벤트는 스프링의 `DispatcherServlet`을 사용하는 웹 애플리케이션에만 적용됩니다. |
| `ServletRequestHandledEvent` | `RequestHandledEvent`의 하위 클래스로, 서블릿 특정 컨텍스트 정보를 추가합니다. |

자신만의 커스텀 이벤트를 생성하고 게시할 수도 있습니다.

다음 예제는 스프링의 `ApplicationEvent` 기본 클래스를 확장하는 간단한 클래스 예시입니다.

```java
// Java
public class BlockedListEvent extends ApplicationEvent {

	private final String address;
	private final String content;

	public BlockedListEvent(Object source, String address, String content) {
		super(source);
		this.address = address;
		this.content = content;
	}

	// 접근자 및 기타 메소드들...

    public String getAddress() { return address; }
    public String getContent() { return content; }
}
```

```kotlin
// Kotlin
class BlockedListEvent(source: Any, val address: String, val content: String) : ApplicationEvent(source) {
    // Accessors are generated automatically for val properties
}
```

커스텀 `ApplicationEvent`를 게시하려면 `ApplicationEventPublisher`에서 `publishEvent()` 메소드를 호출합니다.

일반적으로 이는 `ApplicationEventPublisherAware`를 구현하는 클래스를 생성하고 스프링 빈으로 등록하여 수행됩니다.

```java
// Java
public class EmailService implements ApplicationEventPublisherAware {

	private List<String> blockedList;
	private ApplicationEventPublisher publisher;

	public void setBlockedList(List<String> blockedList) {
		this.blockedList = blockedList;
	}

	@Override
	public void setApplicationEventPublisher(ApplicationEventPublisher publisher) {
		this.publisher = publisher;
	}

	public void sendEmail(String address, String content) {
		if (blockedList.contains(address)) {
			publisher.publishEvent(new BlockedListEvent(this, address, content));
			return;
		}
		// 이메일 보내기...
	}
}
```

```kotlin
// Kotlin
class EmailService : ApplicationEventPublisherAware {
    private var blockedList: List<String>? = null
    private var publisher: ApplicationEventPublisher? = null

    fun setBlockedList(blockedList: List<String>) {
        this.blockedList = blockedList
    }

    override fun setApplicationEventPublisher(publisher: ApplicationEventPublisher) {
        this.publisher = publisher
    }

    fun sendEmail(address: String, content: String) {
        if (blockedList?.contains(address) == true) { // Use safe call
            publisher?.publishEvent(BlockedListEvent(this, address, content)) // Use safe call
            return
        }
        // 이메일 보내기...
    }
}
```

구성 시, 스프링 컨테이너는 `EmailService`가 `ApplicationEventPublisherAware`를 구현함을 감지하고 자동으로 `setApplicationEventPublisher()`를 호출합니다.

실제로는 전달되는 파라미터는 스프링 컨테이너 자체입니다.

`ApplicationEventPublisher` 인터페이스를 통해 애플리케이션 컨텍스트와 상호 작용하는 것입니다.

커스텀 `ApplicationEvent`를 받으려면, `ApplicationListener`를 구현하는 클래스를 생성하고 스프링 빈으로 등록할 수 있습니다.

```java
// Java
public class BlockedListNotifier implements ApplicationListener<BlockedListEvent> {

	private String notificationAddress;

	public void setNotificationAddress(String notificationAddress) {
		this.notificationAddress = notificationAddress;
	}

	@Override
	public void onApplicationEvent(BlockedListEvent event) {
		// notificationAddress를 통해 적절한 당사자에게 알림...
	}
}
```

```kotlin
// Kotlin
class BlockedListNotifier : ApplicationListener<BlockedListEvent> {
    private var notificationAddress: String? = null

    fun setNotificationAddress(notificationAddress: String) {
        this.notificationAddress = notificationAddress
    }

    override fun onApplicationEvent(event: BlockedListEvent) {
        // notificationAddress를 통해 적절한 당사자에게 알림...
        println("Blocked email to ${event.address} with content: ${event.content}") // Example
    }
}
```

`ApplicationListener`가 커스텀 이벤트 타입(앞의 예제에서는 `BlockedListEvent`)으로 제네릭하게 파라미터화된다는 점에 주목하십시오.

이는 `onApplicationEvent()` 메소드가 타입 안전(type-safe)하게 유지될 수 있으며 다운캐스팅(downcasting) 필요성을 피할 수 있음을 의미합니다.

원하는 만큼 많은 이벤트 리스너를 등록할 수 있지만, 기본적으로 이벤트 리스너는 동기적으로(synchronously) 이벤트를 받는다는 점에 유의하십시오.

이는 `publishEvent()` 메소드가 모든 리스너가 이벤트 처리를 완료할 때까지 블록(block)된다는 것을 의미합니다.

이 동기 및 단일 스레드 접근 방식의 한 가지 장점은 리스너가 이벤트를 받을 때 트랜잭션 컨텍스트를 사용할 수 있는 경우 게시자(publisher)의 트랜잭션 컨텍스트 내에서 작동한다는 것입니다.

다음 예제는 위의 각 클래스를 등록하고 구성하는 데 사용되는 빈 정의를 보여줍니다:

```xml
<bean id="emailService" class="example.EmailService">
	<property name="blockedList">
		<list>
			<value>known.spammer@example.org</value>
			<value>known.hacker@example.org</value>
			<value>john.doe@example.org</value>
		</list>
	</property>
</bean>

<bean id="blockedListNotifier" class="example.BlockedListNotifier">
	<property name="notificationAddress" value="blockedlist@example.org"/>
</bean>

<!-- 선택 사항: 커스텀 ApplicationEventMulticaster 정의 -->
<bean id="applicationEventMulticaster" class="org.springframework.context.event.SimpleApplicationEventMulticaster">
	<property name="taskExecutor" ref="..."/> <!-- For asynchronous execution -->
	<property name="errorHandler" ref="..."/> <!-- For handling listener exceptions -->
</bean>
```

모두 종합하면, `emailService` 빈의 `sendEmail()` 메소드가 호출될 때, 차단해야 할 이메일 메시지가 있는 경우 `BlockedListEvent` 타입의 커스텀 이벤트가 게시됩니다.

`blockedListNotifier` 빈은 `ApplicationListener`로 등록되어 `BlockedListEvent`를 받으며, 그 시점에서 적절한 당사자에게 알릴 수 있습니다.

*스프링의 이벤트 메커니즘은 동일한 애플리케이션 컨텍스트 내의 스프링 빈 간의 간단한 통신을 위해 설계되었습니다.*

*그러나 더 정교한 엔터프라이즈 통합 요구 사항의 경우, 별도로 유지 관리되는 스프링 통합(Spring Integration) 프로젝트는 잘 알려진 스프링 프로그래밍 모델을 기반으로 구축되는 경량, 패턴 지향, 이벤트 기반 아키텍처 구축을 위한 완전한 지원을 제공합니다.*

**어노테이션 기반 이벤트 리스너 (Annotation-based Event Listeners)**

`@EventListener` 어노테이션을 사용하여 관리되는 빈의 모든 메소드에 이벤트 리스너를 등록할 수 있습니다.

`BlockedListNotifier`는 다음과 같이 다시 작성될 수 있습니다:

```java
// Java
public class BlockedListNotifier {

	private String notificationAddress;

	public void setNotificationAddress(String notificationAddress) {
		this.notificationAddress = notificationAddress;
	}

	@EventListener
	public void processBlockedListEvent(BlockedListEvent event) {
		// notificationAddress를 통해 적절한 당사자에게 알림...
	}
}
```

```kotlin
// Kotlin
class BlockedListNotifier {
    private var notificationAddress: String? = null

    fun setNotificationAddress(notificationAddress: String) {
        this.notificationAddress = notificationAddress
    }

    @EventListener // Mark the method as an event listener
    fun processBlockedListEvent(event: BlockedListEvent) {
        // notificationAddress를 통해 적절한 당사자에게 알림...
        println("Blocked email to ${event.address} with content: ${event.content}") // Example
    }
}
```

*이러한 빈을 지연(lazy)으로 정의하지 마십시오.*

`*ApplicationContext`는 이를 존중하여 이벤트를 수신하도록 메소드를 등록하지 않습니다.*

메소드 시그니처는 다시 한번 수신하는 이벤트 타입을 선언하지만, 이번에는 유연한 이름과 특정 리스너 인터페이스를 구현하지 않고 선언합니다.

이벤트 타입은 실제 이벤트 타입이 구현 계층 구조에서 제네릭 파라미터를 해석하는 한 제네릭을 통해 좁혀질 수도 있습니다.

메소드가 여러 이벤트를 수신해야 하거나 매개변수 없이 정의하려는 경우, 이벤트 타입은 어노테이션 자체에 지정할 수도 있습니다. 다음 예제는 이를 수행하는 방법을 보여줍니다:

```java
// Java
@EventListener({ContextStartedEvent.class, ContextRefreshedEvent.class})
public void handleContextStart() {
	// ...
}
```

```kotlin
// Kotlin
@EventListener(ContextStartedEvent::class, ContextRefreshedEvent::class) // Specify event types
fun handleContextStart() {
    // ...
}
```

특정 이벤트에 대해 실제로 메소드를 호출하기 위해 일치해야 하는 SpEL 표현식을 정의하는 어노테이션의 `condition` 속성을 사용하여 추가적인 런타임 필터링을 추가하는 것도 가능합니다.

다음 예제는 이벤트의 `content` 속성이 `my-event`와 같을 경우에만 호출되도록 notifier를 다시 작성하는 방법을 보여줍니다:

```java
// Java
@EventListener(condition = "#blEvent.content == 'my-event'")
public void processBlockedListEvent(BlockedListEvent blEvent) {
	// notificationAddress를 통해 적절한 당사자에게 알림...
}
```

```kotlin
// Kotlin
// SpEL condition checks the content property of the BlockedListEvent
@EventListener(condition = "#blEvent.content == 'my-event'")
fun processBlockedListEvent(blEvent: BlockedListEvent) {
    // notificationAddress를 통해 적절한 당사자에게 알림...
}
```

각 SpEL 표현식은 전용 컨텍스트에 대해 평가됩니다.

다음 표는 조건부 이벤트 처리에 사용할 수 있도록 컨텍스트에 제공되는 항목을 나열합니다:

SpEL 표현식에서 사용 가능한 이벤트 메타데이터

| 이름 | 위치 | 설명 | 예제 |
| --- | --- | --- | --- |
| 이벤트 (Event) | 루트 객체 | 실제 `ApplicationEvent`. | `#root.event` 또는 `event` |
| 인수 배열 (Args) | 루트 객체 | 메소드 호출에 사용된 인수 (객체 배열). | `#root.args` 또는 `args`; 첫 번째 인수 접근: `args[0]` 등 |
| 인수 이름 | 평가 컨텍스트 | 특정 메소드 인수의 이름. 이름 사용 불가 시 (예: `-parameters` 플래그 없이 컴파일), `#a<#arg>` 구문으로 개별 인수 사용 가능 (<#arg>는 0부터 시작하는 인수 인덱스). | `#blEvent` 또는 `#a0` (별칭으로 `#p0` 또는 `#p<#arg>` 파라미터 표기법 사용 가능) |

*메소드 시그니처가 실제로 게시된 임의의 객체를 참조하더라도 `#root.event`는 기본 이벤트에 대한 접근을 제공한다는 점에 유의하십시오.*

다른 이벤트를 처리한 결과로 이벤트를 게시해야 하는 경우, 게시해야 할 이벤트를 반환하도록 메소드 시그니처를 변경할 수 있습니다. 다음 예제를 참조하십시오:

```java
// Java
@EventListener
public ListUpdateEvent handleBlockedListEvent(BlockedListEvent event) {
	// notificationAddress를 통해 적절한 당사자에게 알림 후
	// ListUpdateEvent 게시...
	return new ListUpdateEvent(this); // Example return
}
```

```kotlin
// Kotlin
@EventListener
fun handleBlockedListEvent(event: BlockedListEvent): ListUpdateEvent { // Return type is the event to publish
    // notificationAddress를 통해 적절한 당사자에게 알림 후
    // ListUpdateEvent 게시...
    return ListUpdateEvent(this) // Example return
}
// Assuming ListUpdateEvent exists
```

*이 기능은 비동기 리스너에서는 지원되지 않습니다.*

여러 이벤트를 게시해야 하는 경우, `Collection` 또는 이벤트 배열을 대신 반환할 수 있습니다.

*비동기 리스너 (Asynchronous Listeners)*
특정 리스너가 이벤트를 비동기적으로 처리하기를 원하면, 일반적인 `@Async` 지원을 재사용할 수 있습니다. 다음 예제는 이를 수행하는 방법을 보여줍니다:

```java
// Java
@EventListener
@Async
public void processBlockedListEvent(BlockedListEvent event) {
	// BlockedListEvent는 별도의 스레드에서 처리됩니다
}
```

```kotlin
// Kotlin
@EventListener
@Async // Requires @EnableAsync somewhere in the configuration
fun processBlockedListEvent(event: BlockedListEvent) {
    // BlockedListEvent는 별도의 스레드에서 처리됩니다
    println("Processing blocked list event asynchronously...")
}
```

비동기 이벤트를 사용할 때 다음 제한 사항에 유의하십시오:

- 비동기 이벤트 리스너가 `Exception`을 발생시키면 호출자에게 전파되지 않습니다.
- 비동기 이벤트 리스너 메소드는 값을 반환하여 후속 이벤트를 게시할 수 없습니다. 처리 결과로 다른 이벤트를 게시해야 하는 경우, `ApplicationEventPublisher`를 주입하여 이벤트를 수동으로 게시하십시오.
- `ThreadLocal` 및 로깅 컨텍스트는 기본적으로 이벤트 처리를 위해 전파되지 않습니다. 관찰 가능성 문제에 대한 자세한 내용은 `@EventListener` 관찰 가능성 섹션을 참조하십시오.

*리스너 순서 지정 (Ordering Listeners)*
한 리스너가 다른 리스너보다 먼저 호출되어야 하는 경우, 다음 예제와 같이 메소드 선언에 `@Order` 어노테이션을 추가할 수 있습니다:

```java
// Java
@EventListener
@Order(42) // Lower values have higher priority
public void processBlockedListEvent(BlockedListEvent event) {
	// notify appropriate parties via notificationAddress...
}
```

```kotlin
// Kotlin
@EventListener
@Order(42) // Lower values have higher priority
fun processBlockedListEvent(event: BlockedListEvent) {
    // notify appropriate parties via notificationAddress...
}
```

*제네릭 이벤트 (Generic Events)*
이벤트 구조를 더 정의하기 위해 제네릭을 사용할 수도 있습니다.

`T`가 생성된 실제 엔티티의 타입인 `EntityCreatedEvent<T>` 사용

예를 들어, `Person`에 대한 `EntityCreatedEvent`만 받도록 다음 리스너 정의를 생성할 수 있습니다:

```java
// Java
@EventListener
public void onPersonCreated(EntityCreatedEvent<Person> event) {
	// ...
}
```

```kotlin
// Kotlin
@EventListener
fun onPersonCreated(event: EntityCreatedEvent<Person>) { // Listen for specific generic type
    // ...
}
// Assuming EntityCreatedEvent<T> exists
```

타입 소거(type erasure)로 인해, 이는 발생한 이벤트가 이벤트 리스너가 필터링하는 제네릭 파라미터를 해석하는 경우에만 작동합니다 (즉, `class PersonCreatedEvent extends EntityCreatedEvent<Person> { ... }`와 같은 것).

특정 상황에서는 모든 이벤트가 동일한 구조를 따르는 경우(앞의 예제의 이벤트처럼) 상당히 지루해질 수 있습니다.

이러한 경우 `ResolvableTypeProvider`를 구현하여 런타임 환경이 제공하는 것 이상으로 프레임워크를 안내할 수 있습니다.

```java
// Java
public class EntityCreatedEvent<T> extends ApplicationEvent implements ResolvableTypeProvider {

	public EntityCreatedEvent(T entity) {
		super(entity);
	}

	@Override
	public ResolvableType getResolvableType() {
		return ResolvableType.forClassWithGenerics(getClass(), ResolvableType.forInstance(getSource()));
	}
}
```

```kotlin
// Kotlin
class EntityCreatedEvent<T>(entity: T) : ApplicationEvent(entity), ResolvableTypeProvider {

    override fun getResolvableType(): ResolvableType {
        // Helps Spring determine the generic type T at runtime
        return ResolvableType.forClassWithGenerics(javaClass, ResolvableType.forInstance(source))
    }
}
```

이는 `ApplicationEvent`뿐만 아니라 이벤트로 보내는 임의의 객체에도 작동합니다.

*마지막으로, 고전적인 `ApplicationListener` 구현과 마찬가지로 실제 멀티캐스팅(multicasting)은 런타임 시 컨텍스트 전체의 `ApplicationEventMulticaster`를 통해 발생합니다.*

*기본적으로 이것은 호출자 스레드에서 동기적 이벤트 게시를 수행하는 `SimpleApplicationEventMulticaster`입니다.*

*이는 예를 들어, 모든 이벤트를 비동기적으로 처리하고/하거나 리스너 예외를 처리하기 위해 "applicationEventMulticaster" 빈 정의를 통해 교체/사용자 정의될 수 있습니다:*

```java
@Bean
ApplicationEventMulticaster applicationEventMulticaster() {
	SimpleApplicationEventMulticaster multicaster = new SimpleApplicationEventMulticaster();
	multicaster.setTaskExecutor(...); // For async execution
	multicaster.setErrorHandler(...); // For exception handling
	return multicaster;
}
```

**저수준 리소스에 편리하게 접근하기 (Convenient Access to Low-level Resources)**

애플리케이션 컨텍스트를 최적으로 사용하고 이해하려면 리소스(Resources)에서 설명된 스프링의 `Resource` 추상화에 익숙해져야 합니다.

애플리케이션 컨텍스트는 `ResourceLoader`이며, `Resource` 객체를 로드하는 데 사용될 수 있습니다.

`Resource`는 본질적으로 JDK `java.net.URL` 클래스의 더 많은 기능을 가진 버전입니다.

실제로 `Resource` 구현은 적절한 경우 `java.net.URL`의 인스턴스를 래핑합니다.

`Resource`는 클래스패스, 파일 시스템 위치, 표준 URL로 설명할 수 있는 모든 위치 및 일부 다른 변형을 포함하여 거의 모든 위치에서 투명한 방식으로 저수준 리소스를 얻을 수 있습니다.

리소스 위치 문자열이 특별한 접두사 없는 단순 경로인 경우, 해당 리소스가 어디서 오는지는 실제 애플리케이션 컨텍스트 타입에 따라 구체적이고 적절합니다.

애플리케이션 컨텍스트에 배포된 빈이 특별한 콜백 인터페이스인 `ResourceLoaderAware`를 구현하도록 구성하여 초기화 시점에 `ResourceLoader`로 전달된 애플리케이션 컨텍스트 자체로 자동으로 콜백되도록 할 수 있습니다.

정적 리소스에 접근하는 데 사용될 `Resource` 타입의 속성을 노출할 수도 있습니다.

다른 속성처럼 주입됩니다.

이러한 `Resource` 속성을 단순한 문자열 경로로 지정하고 빈이 배포될 때 해당 텍스트 문자열에서 실제 `Resource` 객체로의 자동 변환에 의존할 수 있습니다.

`ApplicationContext` 생성자에 제공되는 위치 경로(들)는 실제로 리소스 문자열이며, 단순한 형태로는 특정 컨텍스트 구현에 따라 적절하게 처리됩니다.

예를 들어 `ClassPathXmlApplicationContext`는 단순 위치 경로를 클래스패스 위치로 처리합니다.

실제 컨텍스트 타입에 관계없이 클래스패스 또는 URL에서 정의 로드를 강제하기 위해 특별한 접두사가 있는 위치 경로(리소스 문자열)를 사용할 수도 있습니다.

**애플리케이션 시작 추적 (Application Startup Tracking)**

`ApplicationContext`는 스프링 애플리케이션의 생명주기를 관리하고 컴포넌트 주변의 풍부한 프로그래밍 모델을 제공합니다.

결과적으로 복잡한 애플리케이션은 동등하게 복잡한 컴포넌트 그래프와 시작 단계를 가질 수 있습니다.

특정 메트릭으로 애플리케이션 시작 단계를 추적하면 시작 단계 동안 시간이 어디에 소요되는지 이해하는 데 도움이 될 수 있지만, 전체적으로 컨텍스트 생명주기를 더 잘 이해하는 방법으로도 사용될 수 있습니다.

`AbstractApplicationContext` (및 그 하위 클래스)는 다양한 시작 단계에 대한 `StartupStep` 데이터를 수집하는 `ApplicationStartup`으로 계측(instrumented)됩니다:

- 애플리케이션 컨텍스트 생명주기 (기본 패키지 스캔, 구성 클래스 관리)
- 빈 생명주기 (인스턴스화, 스마트 초기화, 후처리)
- 애플리케이션 이벤트 처리

다음은 `AnnotationConfigApplicationContext`에서의 계측 예제입니다:

```java
// Java
// 시작 단계 생성 및 기록 시작
StartupStep scanPackages = getApplicationStartup().start("spring.context.base-packages.scan");
// 현재 단계에 태깅 정보 추가
scanPackages.tag("packages", () -> Arrays.toString(basePackages));
// 계측 중인 실제 단계 수행
this.scanner.scan(basePackages);
// 현재 단계 종료
scanPackages.end();
```

```kotlin
// Kotlin
// 시작 단계 생성 및 기록 시작
val scanPackages = applicationStartup.start("spring.context.base-packages.scan")
// 현재 단계에 태깅 정보 추가
scanPackages.tag("packages") { basePackages.contentToString() } // Use contentToString for arrays
// 계측 중인 실제 단계 수행
this.scanner.scan(*basePackages) // Use spread operator for varargs
// 현재 단계 종료
scanPackages.end()
```

애플리케이션 컨텍스트는 이미 여러 단계로 계측되어 있습니다.

일단 기록되면, 이러한 시작 단계는 특정 도구를 사용하여 수집, 표시 및 분석될 수 있습니다.

기본 `ApplicationStartup` 구현은 최소한의 오버헤드를 위한 no-op 변형입니다.

이는 즉, 기본적으로 애플리케이션 시작 중에 메트릭이 수집되지 않음을 의미합니다.

스프링 프레임워크는 Java Flight Recorder로 시작 단계를 추적하기 위한 구현인 `FlightRecorderApplicationStartup`을 제공합니다.

이 변형을 사용하려면 생성된 즉시 `ApplicationContext`에 해당 인스턴스를 구성해야 합니다.

개발자는 자체 `AbstractApplicationContext` 하위 클래스를 제공하거나 더 정밀한 데이터를 수집하고자 하는 경우 `ApplicationStartup` 인프라를 사용할 수도 있습니다.

`*ApplicationStartup`은 애플리케이션 시작 중 및 핵심 컨테이너에만 사용되도록 의도되었습니다. 이는 결코 Java 프로파일러나 Micrometer와 같은 메트릭 라이브러리를 대체하는 것이 아닙니다.*

커스텀 `StartupStep` 수집을 시작하려면, 컴포넌트는 애플리케이션 컨텍스트에서 직접 `ApplicationStartup` 인스턴스를 가져오거나, 컴포넌트가 `ApplicationStartupAware`를 구현하도록 하거나, 모든 주입 지점에서 `ApplicationStartup` 타입을 요청할 수 있습니다.

*개발자는 커스텀 시작 단계를 생성할 때 "spring.*" 네임스페이스를 사용해서는 안 됩니다. 이 네임스페이스는 내부 스프링 사용을 위해 예약되어 있으며 변경될 수 있습니다.*

**Convenient ApplicationContext Instantiation for Web Applications**

예를 들어, `ContextLoader`를 사용하여 선언적으로 `ApplicationContext` 인스턴스를 생성할 수 있습니다.

물론, `ApplicationContext` 구현 중 하나를 사용하여 프로그래밍 방식으로 `ApplicationContext` 인스턴스를 생성할 수도 있습니다.

다음 예제와 같이 `ContextLoaderListener`를 사용하여 `ApplicationContext`를 등록할 수 있습니다:

```xml
<context-param>
	<param-name>contextConfigLocation</param-name>
	<param-value>/WEB-INF/daoContext.xml /WEB-INF/applicationContext.xml</param-value>
</context-param>

<listener>
	<listener-class>org.springframework.web.context.ContextLoaderListener</listener-class>
</listener>
```

리스너는 `contextConfigLocation` 파라미터를 검사합니다.

파라미터가 존재하지 않으면 리스너는 `/WEB-INF/applicationContext.xml`을 기본값으로 사용합니다.

파라미터가 존재하면 리스너는 미리 정의된 구분 기호(쉼표, 세미콜론 및 공백)를 사용하여 문자열을 분리하고 해당 값을 애플리케이션 컨텍스트를 검색하는 위치로 사용합니다.

Ant 스타일 경로 패턴도 지원됩니다. 예로는 `/WEB-INF/*Context.xml` (`WEB-INF` 디렉토리에 있고 이름이 `Context.xml`로 끝나는 모든 파일) 및 `/WEB-INF/**/*Context.xml` (`WEB-INF`의 모든 하위 디렉토리에 있는 그러한 모든 파일)이 있습니다.

**스프링 ApplicationContext를 Jakarta EE RAR 파일로 배포하기 (Deploying a Spring ApplicationContext as a Jakarta EE RAR File)**

스프링 `ApplicationContext`를 RAR 파일로 배포하여 컨텍스트와 필요한 모든 빈 클래스 및 라이브러리 JAR을 Jakarta EE RAR 배포 단위에 캡슐화하는 것이 가능합니다.

이는 (Jakarta EE 환경에서만 호스팅되는) 독립 실행형 `ApplicationContext` 부트스트래핑과 동일하며 Jakarta EE 서버 시설에 접근할 수 있습니다.

RAR 배포는 헤드리스(headless) WAR 파일 배포 시나리오에 대한 더 자연스러운 대안입니다 - 사실상, Jakarta EE 환경에서 스프링 `ApplicationContext`를 부트스트래핑하는 데만 사용되는 HTTP 진입점이 없는 WAR 파일입니다.

RAR 배포는 HTTP 진입점이 필요하지 않고 메시지 엔드포인트와 스케줄된 작업으로만 구성된 애플리케이션 컨텍스트에 이상적입니다.

이러한 컨텍스트의 빈은 JTA 트랜잭션 관리자 및 JNDI 바인딩된 JDBC `DataSource` 인스턴스 및 JMS `ConnectionFactory` 인스턴스와 같은 애플리케이션 서버 리소스를 사용할 수 있으며, 플랫폼의 JMX 서버에도 등록할 수 있습니다

- 모두 스프링의 표준 트랜잭션 관리 및 JNDI 및 JMX 지원 기능을 통해 가능합니다.

애플리케이션 컴포넌트는 스프링의 `TaskExecutor` 추상화를 통해 애플리케이션 서버의 JCA `WorkManager`와 상호 작용할 수도 있습니다.

스프링 `ApplicationContext`를 Jakarta EE RAR 파일로 간단히 배포하려면:

1. 모든 애플리케이션 클래스를 RAR 파일(다른 파일 확장자를 가진 표준 JAR 파일)로 패키징합니다.
2. 필요한 모든 라이브러리 JAR을 RAR 아카이브의 루트에 추가합니다.
3. `META-INF/ra.xml` 배포 기술자 (`SpringContextResourceAdapter`의 javadoc에 표시된 대로)와 해당 스프링 XML 빈 정의 파일(들) (일반적으로 `META-INF/applicationContext.xml`)을 추가합니다.
4. 결과 RAR 파일을 애플리케이션 서버의 배포 디렉토리에 놓습니다.

이러한 RAR 배포 단위는 일반적으로 자체 포함(self-contained)됩니다.

외부 세계, 심지어 동일한 애플리케이션의 다른 모듈에도 컴포넌트를 노출하지 않습니다.

RAR 기반 `ApplicationContext`와의 상호 작용은 일반적으로 다른 모듈과 공유하는 JMS 대상을 통해 발생합니다.

RAR 기반 `ApplicationContext`는 예를 들어 일부 작업을 스케줄링하거나 파일 시스템(또는 이와 유사한 것)의 새 파일에 반응할 수도 있습니다.

외부로부터 동기적 접근을 허용해야 하는 경우, 예를 들어 동일한 시스템의 다른 애플리케이션 모듈에서 사용할 수 있는 RMI 엔드포인트를 내보낼(export) 수 있습니다.

---

**전체 주제: ApplicationContext의 추가 기능 (Additional Capabilities of the ApplicationContext)**

이 부분은 스프링 컨테이너의 핵심 엔진인 `BeanFactory`만 사용하는 것보다, 그 기능을 확장한 `ApplicationContext`를 사용하는 것이 왜 더 편리하고 강력한지를 보여줍니다.

`ApplicationContext`는 단순히 빈을 생성하고 관리하는 것을 넘어, **애플리케이션 개발에 필요한 다양한 프레임워크 수준의 기능**들을 통합적으로 제공합니다.

**핵심 아이디어:** `ApplicationContext`는 `BeanFactory`의 기능에 더해, 메시지 처리(국제화), 리소스 접근, 이벤트 발행, 계층 구조 지원 등 풍부한 추가 기능을 제공하여 풀 스택(full-stack) 애플리케이션 개발을 더 쉽게 만들어준다.

**주요 추가 기능 목록:**

1. **국제화 (Internationalization, i18n):** `MessageSource` 인터페이스를 통한 다국어 메시지 처리 지원.
2. **리소스 접근:** `ResourceLoader` 인터페이스를 통한 파일, URL 등 외부 자원 접근 기능.
3. **이벤트 발행:** `ApplicationEventPublisher` 인터페이스를 통한 빈들 간의 이벤트 기반 통신 지원.
4. **계층 구조:** 부모-자식 관계의 컨텍스트를 구성하여 설정을 모듈화하고 공유하는 기능.

---

**첫 번째 파트: MessageSource를 사용한 국제화 (i18n)**

이 부분은 `ApplicationContext`가 어떻게 **다국어 메시지 처리(Internationalization, 줄여서 i18n)** 기능을 지원하는지에 대해 설명합니다.

- **`MessageSource` 인터페이스:** `ApplicationContext`는 이 인터페이스를 상속(extends)합니다. 이 인터페이스는 메시지 코드(key), 로케일(locale, 언어 및 국가 정보) 등을 기반으로 해당 언어에 맞는 메시지를 찾아주는 기능을 정의합니다.
  - `HierarchicalMessageSource`: `MessageSource`를 확장하여, 메시지를 찾지 못했을 때 부모 컨텍스트의 `MessageSource`에서도 찾아보는 계층적 검색 기능을 추가합니다.
- **핵심 메소드 (`getMessage`):**
  - `getMessage(code, args, defaultMsg, locale)`: 가장 기본적인 메소드. `code`에 해당하는 메시지를 `locale`에 맞춰 찾습니다. 없으면 `defaultMsg`를 반환합니다. `args` 배열로 메시지 내 플레이스홀더(`{0}`, `{1}`)에 들어갈 값을 전달할 수 있습니다.
  - `getMessage(code, args, locale)`: 기본 메시지 지정 없이, 메시지를 못 찾으면 `NoSuchMessageException` 예외를 발생시킵니다.
  - `getMessage(resolvable, locale)`: 메시지 코드, 인수, 기본 메시지를 `MessageSourceResolvable` 객체 하나로 묶어서 전달하는 방식입니다.
- **자동 감지 및 위임:**
  - `ApplicationContext`가 로드될 때, 컨테이너 내에 이름이 **`messageSource`** 인 빈이 있는지 **자동으로 검색**합니다.
  - 만약 `messageSource` 빈이 있다면, `ApplicationContext`의 `getMessage` 메소드 호출은 **이 빈에게 위임**됩니다.
  - 만약 현재 컨텍스트에 `messageSource` 빈이 없고 부모 컨텍스트가 있다면, 부모 컨텍스트에서 `messageSource` 빈을 찾아 사용합니다.
  - 어디에서도 찾지 못하면, 빈 `DelegatingMessageSource`가 생성되어 기본 동작(예: 기본 메시지 반환)을 수행합니다.
- **스프링 구현체:**
  - `ResourceBundleMessageSource`: JDK 표준 `ResourceBundle`을 사용하여 클래스패스에 있는 `.properties` 파일에서 메시지를 읽어옵니다. (가장 기본적)
  - `ReloadableResourceBundleMessageSource`: `ResourceBundleMessageSource`보다 유연합니다. 클래스패스 외의 경로에서도 파일을 읽을 수 있고, 애플리케이션 재시작 없이 프로퍼티 파일을 다시 로드(hot reloading)하는 기능을 지원합니다.
  - `StaticMessageSource`: 거의 사용되지 않지만, 코드 내에서 프로그래밍 방식으로 메시지를 추가할 때 사용합니다.
- **XML 설정 예시 (`ResourceBundleMessageSource`):**

    ```xml
    <bean id="messageSource" class="org.springframework.context.support.ResourceBundleMessageSource">
        <property name="basenames">
            <list>
                <value>format</value>       <!-- format.properties, format_ko.properties, format_en_GB.properties 등 -->
                <value>exceptions</value>   <!-- exceptions.properties, exceptions_ko.properties 등 -->
                <value>windows</value>      <!-- windows.properties 등 -->
            </list>
        </property>
    </bean>
    ```

  - `basenames` 속성에 프로퍼티 파일들의 기본 이름 목록을 지정합니다. 스프링은 로케일에 따라 적절한 파일(예: `format_ko.properties`)을 찾아 사용합니다.
- **사용 예시:**

    ```java
    MessageSource resources = new ClassPathXmlApplicationContext("beans.xml");
    // 영어 로케일로 "message" 코드에 해당하는 메시지 요청
    String msg = resources.getMessage("message", null, "Default", Locale.ENGLISH);
    System.out.println(msg); // format.properties 또는 format_en.properties 내용 출력
    
    // 플레이스홀더 사용 예시 (exceptions.properties: argument.required=The {0} argument is required.)
    String errMsg = resources.getMessage("argument.required", new Object[]{"userDao"}, "Required", Locale.ENGLISH);
    System.out.println(errMsg); // "The userDao argument is required." 출력
    ```

- **로케일 처리:** JDK 표준 `ResourceBundle` 규칙에 따라 로케일(예: `en_GB`)에 맞는 파일을 찾고, 없으면 기본 언어 파일(예: `en`), 그것도 없으면 기본 파일(예: `format.properties`) 순서로 찾습니다(fallback).
- **`MessageSourceAware`:** 빈이 `MessageSourceAware` 인터페이스를 구현하면, 스프링은 해당 빈에게 `ApplicationContext`의 `MessageSource` 객체를 자동으로 주입해줍니다.

**요약 (i18n):** `ApplicationContext`는 `MessageSource` 기능을 내장하여 다국어 지원을 쉽게 구현할 수 있도록 합니다. `messageSource`라는 이름의 빈을 등록하면 (주로 `ResourceBundleMessageSource`나 `ReloadableResourceBundleMessageSource` 사용), `getMessage` 메소드를 통해 로케일에 맞는 메시지를 플레이스홀더 처리와 함께 가져올 수 있습니다.

MessageSource를 사용하는 **핵심적인 이유와 목적**은 다음과 같습니다:

1. **메시지의 중앙 관리 및 외부화 (Centralization & Externalization):**
  - **문제점:** 애플리케이션 코드(Java, Kotlin)나 화면 템플릿(HTML, JSP, Thymeleaf 등) 안에 직접 사용자에게 보여줄 메시지(예: "로그인 성공", "아이디는 필수입니다", "주문 완료")를 하드코딩하면 어떻게 될까요?
    - 메시지를 수정해야 할 때 (오타 수정, 문구 변경 등) 해당 메시지가 사용된 **모든 코드와 파일을 찾아 일일이 수정**해야 합니다. 매우 번거롭고 실수를 유발하기 쉽습니다.
    - 같은 의미의 메시지가 여러 곳에서 약간 다른 문구로 사용될 수 있어 **일관성이 떨어집니다.**
  - **해결책:** MessageSource는 애플리케이션에서 사용하는 모든 메시지를 **별도의 외부 파일(주로 .properties 파일)** 에 모아놓고 **중앙에서 관리**할 수 있게 해줍니다. 코드는 이 외부 파일을 직접 참조하지 않고, MessageSource를 통해 간접적으로 접근합니다.
  - **장점:** 메시지 수정이 필요하면 **외부 파일 한 곳만 수정**하면 됩니다. 코드 변경 없이 메시지 관리가 가능해집니다. 메시지의 일관성을 유지하기 쉽습니다.
2. **코드와 메시지의 분리 (Separation of Concerns):**
  - **문제점:** 개발자가 코드 로직과 사용자 메시지 문구를 동시에 신경 써야 합니다. 마케팅팀이나 기획팀에서 메시지 문구 변경을 요청할 때마다 개발자가 코드를 수정하고 재배포해야 할 수 있습니다.
  - **해결책:** MessageSource를 사용하면 개발자는 코드에서 메시지를 직접 다루는 대신, **메시지 코드(Key)** (예: "login.success", "validation.username.required")만 사용합니다. 실제 메시지 내용은 외부 파일에 정의됩니다.
  - **장점:** 개발자는 로직에 집중하고, 메시지 담당자(기획자, 마케터, 번역가 등)는 코드에 대한 지식 없이 외부 메시지 파일만 관리하면 됩니다. 역할 분리가 명확해지고 협업이 용이해집니다.
3. **국제화(i18n) 지원 (Internationalization):**
  - **문제점:** 하드코딩된 메시지는 다국어 지원이 거의 불가능합니다. 사용자 언어에 따라 if/else로 모든 메시지를 분기 처리하는 것은 끔찍한 일입니다.
  - **해결책:** MessageSource는 로케일(Locale - 언어 및 국가 정보)을 기반으로 적절한 메시지 파일을 자동으로 선택해줍니다.
    - messages_ko.properties (한국어 메시지)
    - messages_en.properties (영어 메시지)
    - messages_ja.properties (일본어 메시지)
    - ... 등을 준비해두면, 코드에서는 **동일한 메시지 코드("login.success")** 를 사용하더라도 사용자의 로케일에 맞는 언어의 메시지가 자동으로 반환됩니다.
  - **장점:** 다국어 지원을 매우 체계적이고 효율적으로 구현할 수 있습니다. 새로운 언어 추가도 해당 언어의 메시지 파일만 추가하면 됩니다.
4. **메시지 매개변수화 (Parameterization):**
  - 단순히 고정된 메시지만 보여주는 것이 아니라, 메시지 중간에 **동적인 값**을 넣어야 할 때가 많습니다. (예: "{0}님의 주문이 완료되었습니다.", "{0} 필드는 필수입니다.")
  - MessageSource는 getMessage 메소드의 args 파라미터를 통해 이러한 **매개변수(Parameter)** 를 전달받아 메시지의 {0}, {1} 같은 플레이스홀더에 안전하고 올바르게 채워 넣어주는 기능을 제공합니다. 이는 언어별 문법(어순 등) 차이도 고려할 수 있게 해줍니다.

**결론:**

MessageSource는 단순히 "언어 변환"만을 위한 기능이 아닙니다. 그것은 애플리케이션의 **유지보수성, 확장성, 협업 효율성, 코드 품질**을 높이기 위한 중요한 설계 패턴이자 도구입니다. 코드와 사용자 메시지를 분리하고 중앙에서 관리함으로써 얻는 이점은 매우 크며, 국제화(i18n)는 그 중요한 활용 사례 중 하나일 뿐입니다.

---

**두 번째 파트: 표준 및 커스텀 이벤트 (Standard and Custom Events)**

이 부분은 스프링 컨테이너 내에서 발생하는 특정 사건(이벤트)을 감지하고 이에 반응하거나, 개발자가 직접 정의한 커스텀 이벤트를 발행하고 수신하는 방법에 대해 설명합니다. **옵저버(Observer) 디자인 패턴**을 스프링 빈들 간의 통신에 적용한 것입니다.

**핵심 아이디어:** 특정 상황(이벤트)이 발생했을 때, 관심 있는 다른 빈들에게 "이런 일이 일어났어!" 라고 알리고 필요한 작업을 수행하게 하자. (느슨한 결합 방식)

**주요 구성 요소:**

1. **`ApplicationEvent`:** 모든 이벤트 객체의 **기본 부모 클래스**입니다. 특정 이벤트는 이 클래스를 상속받아 만들어집니다. 이벤트 발생 시 전달하고 싶은 데이터를 포함할 수 있습니다.
  - 스프링 4.2부터는 꼭 `ApplicationEvent`를 상속하지 않은 **임의의 객체**도 이벤트로 발행할 수 있습니다. 스프링이 내부적으로 래핑하여 처리합니다.
2. **`ApplicationListener<E extends ApplicationEvent>`:** 특정 타입(`E`)의 이벤트를 **수신(listen)** 하고 처리하는 역할을 하는 **인터페이스**입니다. 이 인터페이스를 구현한 빈은 해당 타입의 이벤트가 발생했을 때 `onApplicationEvent(E event)` 메소드가 자동으로 호출됩니다.
3. **`ApplicationEventPublisher`:** 이벤트를 **발행(publish)** 하는 역할을 하는 **인터페이스**입니다. `ApplicationContext` 자체가 이 인터페이스를 구현하고 있습니다. `publishEvent(Object event)` 메소드를 사용하여 이벤트를 발생시킵니다.

**스프링 내장 표준 이벤트:**

스프링 컨테이너는 자신의 생명주기나 특정 활동과 관련된 몇 가지 표준 이벤트를 자동으로 발행합니다. 개발자는 이 이벤트들을 수신하여 컨테이너 상태 변화에 대응할 수 있습니다.

| 이벤트 | 설명 |
| --- | --- |
| `ContextRefreshedEvent` | `ApplicationContext`가 **초기화되거나 리프레시**될 때 발행됩니다. 모든 빈이 로드되고 준비된 상태입니다. (핫 리프레시 지원 시 여러 번 발생 가능) |
| `ContextStartedEvent` | `ApplicationContext`의 `start()` 메소드가 호출되어 **시작**될 때 발행됩니다. 모든 `Lifecycle` 빈들이 시작 신호를 받습니다. |
| `ContextStoppedEvent` | `ApplicationContext`의 `stop()` 메소드가 호출되어 **중지**될 때 발행됩니다. 모든 `Lifecycle` 빈들이 중지 신호를 받습니다. (다시 시작 가능) |
| `ContextClosedEvent` | `ApplicationContext`의 `close()` 메소드가 호출되거나 JVM 종료 훅 등으로 **닫힐 때** 발행됩니다. 모든 싱글톤 빈이 소멸됩니다. (재시작 불가) |
| `RequestHandledEvent` | (웹 환경) HTTP 요청 처리가 **완료**되었을 때 발행됩니다. (스프링 MVC `DispatcherServlet` 사용 시) |
| `ServletRequestHandledEvent` | (웹 환경) `RequestHandledEvent`의 하위 클래스로, 서블릿 관련 정보를 더 포함합니다. |

**커스텀 이벤트 생성 및 사용:**

개발자가 직접 필요한 이벤트를 만들고 발행/수신할 수 있습니다.

1. **커스텀 이벤트 클래스 정의:** `ApplicationEvent`를 상속받거나, 그냥 일반 POJO 객체로 만듭니다. 이벤트 발생 시 전달할 데이터를 멤버 변수로 가집니다.

    ```java
    // 예: 이메일 차단 목록 관련 이벤트
    public class BlockedListEvent extends ApplicationEvent {
        private final String address;
        private final String content;
    
        public BlockedListEvent(Object source, String address, String content) {
            super(source); // source: 이벤트를 발생시킨 객체
            this.address = address;
            this.content = content;
        }
        // Getter 메소드들 ...
    }
    ```

2. **이벤트 발행자(Publisher) 구현:**
  - 이벤트를 발행할 빈 클래스가 `ApplicationEventPublisherAware` 인터페이스를 구현합니다.
  - 스프링은 `setApplicationEventPublisher` 메소드를 통해 `ApplicationEventPublisher` (실제로는 `ApplicationContext` 자신)를 주입해줍니다.
  - 이벤트 발생 시점에 주입받은 `publisher.publishEvent(new MyCustomEvent(...))` 를 호출합니다.

    ```java
    public class EmailService implements ApplicationEventPublisherAware {
        private ApplicationEventPublisher publisher;
    
        @Override
        public void setApplicationEventPublisher(ApplicationEventPublisher publisher) {
            this.publisher = publisher;
        }
    
        public void sendEmail(String address, String content) {
            if (isBlocked(address)) {
                // ★ 커스텀 이벤트 발행 ★
                publisher.publishEvent(new BlockedListEvent(this, address, content));
                return;
            }
            // ... 이메일 발송 로직 ...
        }
        // ...
    }
    ```

3. **이벤트 리스너(Listener) 구현:**
  - 이벤트를 수신할 빈 클래스가 `ApplicationListener<EventType>` 인터페이스를 구현합니다. 제네릭(`EventType`)으로 수신할 이벤트 타입을 지정하면 타입 안전성이 확보됩니다.
  - `onApplicationEvent(EventType event)` 메소드를 구현하여 이벤트 발생 시 처리할 로직을 작성합니다.

    ```java
    public class BlockedListNotifier implements ApplicationListener<BlockedListEvent> { // ★ BlockedListEvent만 수신 ★
        @Override
        public void onApplicationEvent(BlockedListEvent event) {
            // 이벤트 발생 시 처리할 로직
            System.out.println("차단된 이메일 알림: " + event.getAddress());
            // ... 알림 로직 ...
        }
    }
    ```

4. **빈 등록:** 이벤트 발행자(`EmailService`)와 리스너(`BlockedListNotifier`)를 모두 스프링 빈으로 등록해야 합니다.

**이벤트 처리 방식:**

- **동기 처리 (기본값):** 기본적으로 `publishEvent()` 메소드는 **동기적**으로 동작합니다. 즉, 이벤트를 발행하면 해당 이벤트를 수신하는 **모든 리스너의 `onApplicationEvent` 메소드가 순차적으로 실행 완료될 때까지 `publishEvent()` 메소드는 블록(대기)** 합니다.
  - **장점:** 발행자의 트랜잭션 컨텍스트 내에서 리스너가 실행될 수 있습니다.
  - **단점:** 리스너 중 하나라도 오래 걸리면 전체 프로세스가 지연될 수 있습니다.
- **비동기 처리 (설정 필요):** 이벤트 처리를 비동기적(별도 스레드)으로 수행하려면, `ApplicationEventMulticaster` 빈(보통 `SimpleApplicationEventMulticaster`)을 직접 설정하고 `taskExecutor` 속성에 비동기 실행기(Executor)를 지정해주어야 합니다.

**요약 (이벤트):** `ApplicationContext`는 옵저버 패턴 기반의 이벤트 메커니즘을 제공합니다. `ApplicationEvent` (또는 POJO), `ApplicationListener`, `ApplicationEventPublisher`를 사용하여 빈들 간의 느슨하게 결합된 통신을 구현할 수 있습니다. 스프링은 자체적인 생명주기 이벤트를 발행하며, 개발자는 커스텀 이벤트를 만들고 발행/수신할 수 있습니다. 기본적으로 동기 처리되며, 비동기 처리를 위한 설정도 가능합니다.

**스프링 이벤트를 사용하는 핵심 목적: 관심사의 분리(Separation of Concerns)와 느슨한 결합(Loose Coupling)**

이게 가장 중요합니다. 어떤 시스템에서 특정 작업(A)이 완료된 후, 그 결과에 따라 다른 여러 부가적인 작업들(B, C, D...)이 이어서 수행되어야 하는 경우가 많습니다.

**만약 이벤트를 사용하지 않는다면? (문제점)**

- 작업 A를 수행하는 컴포넌트(예: UserService)가 작업 B, C, D를 수행하는 다른 모든 컴포넌트들(예: EmailService, PointService, NotificationService)을 **직접 알고 있어야** 합니다.
- UserService는 사용자 등록(작업 A)을 완료한 후, 직접 EmailService의 환영 메일 발송 메소드, PointService의 가입 포인트 적립 메소드, NotificationService의 관리자 알림 메소드를 **순서대로 호출**해야 합니다.
- **문제점:**
  - **강한 결합(Tight Coupling):** UserService가 다른 서비스들에 너무 많이 의존하게 됩니다. EmailService의 인터페이스가 바뀌거나 PointService가 없어지면 UserService 코드도 수정해야 합니다.
  - **단일 책임 원칙(SRP) 위반:** UserService의 주 책임은 사용자 관리인데, 이메일 발송, 포인트 적립, 알림 등 부가적인 책임까지 떠안게 되어 코드가 복잡해집니다.
  - **확장성 저하:** 만약 나중에 사용자 등록 후 "추천 친구 목록 생성" (작업 E) 기능을 추가하려면, 또 UserService 코드를 **수정**해야 합니다. 새로운 기능 추가가 기존 코드를 계속 건드리게 됩니다 (개방-폐쇄 원칙 OCP 위반).

**이벤트 메커니즘을 사용하는 이유 (해결책 및 장점)**

스프링 이벤트는 이런 문제를 해결하기 위해 **옵저버(Observer) 패턴**을 적용한 것입니다.

1. **관심사 분리:**
  - **이벤트 발행자(Publisher, 예: UserService)**: 자신의 핵심 책임(사용자 등록 완료)에만 집중합니다. 작업 완료 후에는 단순히 **"사용자 등록 이벤트 발생!"** 이라고 **선언(발행)** 하기만 하면 됩니다. 어떤 다른 작업들이 뒤따라야 하는지는 **알 필요가 없습니다.**
  - **이벤트 리스너(Listener, 예: EmailService, PointService)**: 특정 이벤트(사용자 등록 이벤트)에 **관심**을 가지고 있다가, 해당 이벤트가 발생하면 **자신이 해야 할 일**(환영 메일 발송, 포인트 적립)만 수행합니다. 누가 이벤트를 발생시켰는지는 알 필요가 없습니다.
  - **이벤트 객체(Event Object, 예: UserRegisteredEvent)**: 이벤트 발생 시 필요한 정보(예: 등록된 사용자 정보)를 담아서 전달하는 매개체 역할을 합니다.
2. **느슨한 결합:**
  - 발행자와 리스너는 서로를 **직접 알지 못합니다.** 오직 스프링의 이벤트 시스템(ApplicationEventPublisher, ApplicationListener 또는 @EventListener)을 통해서만 상호작용합니다.
  - UserService는 EmailService나 PointService의 존재를 몰라도 되고, 반대로 EmailService 등도 UserService를 몰라도 됩니다. 오직 "사용자 등록 이벤트"라는 **약속(이벤트 타입)** 만 공유합니다.
3. **향상된 확장성 및 유지보수성:**
  - 사용자 등록 후 새로운 작업(예: 친구 추천 목록 생성)을 추가하고 싶다면? 그냥 해당 이벤트를 수신하는 **새로운 리스너(FriendRecommendationListener)만 만들어서 빈으로 등록**하면 됩니다. 기존 UserService 코드는 **전혀 수정할 필요가 없습니다.** (OCP 준수)
  - 기존 리스너의 로직 변경(예: 이메일 내용 변경)도 해당 리스너 클래스만 수정하면 되므로 영향 범위가 명확하고 유지보수가 쉬워집니다.

**다양한 활용 사례 (이메일 차단 외):**

- **사용자 가입 처리:**
  - (발행) UserService: 사용자 가입 성공 이벤트 발행
  - (수신) EmailService: 환영 이메일 발송
  - (수신) PointService: 가입 축하 포인트 적립
  - (수신) CouponService: 신규 가입 쿠폰 발급
- **주문 완료 처리:**
  - (발행) OrderService: 주문 완료 이벤트 발행
  - (수신) InventoryService: 재고 차감
  - (수신) NotificationService: 주문 확인 SMS/메일 발송
  - (수신) ShippingService: 배송 요청 등록
- **게시물 등록 처리:**
  - (발행) BoardService: 새 게시물 등록 이벤트 발행
  - (수신) NotificationService: 구독자에게 새 글 알림 발송
  - (수신) SearchIndexService: 검색 엔진에 새 글 색인 요청
- **스프링 내부 이벤트 활용:**
  - (수신) MyCacheInitializer: ContextRefreshedEvent (컨텍스트 초기화 완료 이벤트)를 수신하여 애플리케이션 시작 시 캐시 데이터 미리 로드
  - (수신) MyResourceCleaner: ContextClosedEvent (컨텍스트 종료 이벤트)를 수신하여 애플리케이션 종료 시 사용하던 외부 리소스 정리
- **시스템 상태 변경 알림:** 특정 설정값이 변경되거나, 중요한 시스템 상태가 변경되었을 때 관련 컴포넌트들에게 이벤트 발행하여 상태 동기화

**결론:**

스프링 이벤트는 단순히 알림을 보내는 것을 넘어, 시스템 내 컴포넌트 간의 **결합도를 낮추고**, 각 컴포넌트가 자신의 **책임에만 집중**하도록 하며, **유연하고 확장 가능한 구조**를 만드는 데 매우 유용한 도구입니다. 어떤 작업 후에 **부가적으로 다양한 후속 처리**가 필요하거나, **서로 직접 알 필요가 없는 컴포넌트 간의 상호작용**이 필요할 때 이벤트 메커니즘 사용을 고려해볼 수 있습니다.

**Aware 인터페이스의 역할: 빈에게 '능력' 또는 '정보' 부여하기**

스프링 컨테이너는 빈 객체를 생성하고 관리하는 역할을 합니다. 하지만 빈 객체 자체는 기본적으로 자신이 스프링 컨테이너 안에서 돌아가고 있다는 사실이나, 컨테이너가 어떤 다른 기능들을 제공하는지 **알지 못합니다.** 그냥 평범한 자바 객체(POJO)일 뿐이죠.

그런데 때로는 빈 객체가 다음과 같은 일을 하고 싶을 수 있습니다:

- "내가 속한 스프링 컨테이너(ApplicationContext) 객체 자체를 직접 사용하고 싶어!" (예: 다른 빈을 이름으로 찾거나, 이벤트를 발행하기 위해)
- "설정 파일에서 나에게 부여된 이름(빈 ID)이 뭔지 알고 싶어!" (예: 로그 출력 시 자신을 식별하기 위해)
- "메시지 번들 파일(.properties)에 접근해서 다국어 메시지를 가져오고 싶어!"
- "파일이나 클래스패스 같은 외부 리소스를 읽어오고 싶어!"
- "내가 어떤 클래스 로더에 의해 로드되었는지 알고 싶어!"
- "현재 웹 애플리케이션의 ServletContext 객체를 사용하고 싶어!" (웹 환경)

이런 요구사항들을 해결하기 위해 스프링은 Aware 라는 이름이 붙은 여러 인터페이스들을 제공합니다. (예: ApplicationContextAware, BeanNameAware, MessageSourceAware, ResourceLoaderAware, ServletContextAware 등)

**Aware 인터페이스의 동작 방식 (약속과 이행):**

1. **빈의 약속 (Interface Implementation):** 개발자는 특정 기능이나 정보가 필요한 빈 클래스에게 해당 Aware 인터페이스를 **구현(implements)** 하도록 만듭니다. 예를 들어, ApplicationContext가 필요하면 ApplicationContextAware를 구현합니다. 이는 "나는 ApplicationContext를 알고 싶고 사용할 준비가 되었어!" 라고 스프링에게 **약속**하는 것과 같습니다.
2. **스프링 컨테이너의 약속 이행 (Callback):** 스프링 컨테이너는 빈 객체를 생성하고 초기화하는 과정에서, 해당 빈이 어떤 Aware 인터페이스를 구현했는지 **확인(인지, aware)** 합니다.
3. 만약 빈이 ApplicationContextAware를 구현했다면, 스프링 컨테이너는 **약속대로** 그 빈의 setApplicationContext(ApplicationContext ctx) 메소드를 **호출**하면서 **컨테이너 자신의 참조(ctx)를 파라미터로 전달**해줍니다.
4. 마찬가지로, BeanNameAware를 구현했다면 setBeanName(String name) 메소드를 호출하며 빈의 이름을 전달해주고, ResourceLoaderAware를 구현했다면 setResourceLoader(ResourceLoader loader)를 호출하며 리소스 로더 객체를 전달해줍니다.
5. **빈의 능력 획득:** 이렇게 Aware 인터페이스의 메소드가 호출되면서 필요한 객체나 정보를 전달받은 빈은, 그 정보를 **내부적으로 저장**해 두었다가 자신이 필요할 때 **사용**할 수 있게 됩니다. 즉, 특정 환경 정보나 기능에 대해 **"인지(aware)"** 하고 **"활용(use)"** 할 수 있는 **능력**을 얻게 되는 것입니다.

**결론:**

스프링에서 Aware 인터페이스는 **빈(Bean)이 스프링 컨테이너나 실행 환경의 특정 객체, 정보, 또는 기능에 접근할 필요가 있을 때 사용되는 콜백(Callback) 메커니즘**입니다. 빈이 특정 Aware 인터페이스를 구현하면, 스프링 컨테이너는 빈 생성 및 초기화 과정 중 적절한 시점에 해당 인터페이스의 메소드를 호출하여 빈에게 필요한 **"인지 능력"** 과 **"활용 수단"** 을 제공해줍니다.

하지만 이 방식은 빈 코드가 스프링 API에 직접 의존하게 만들기 때문에(결합도 증가), 꼭 필요한 경우가 아니면 의존성 주입(@Autowired 등)을 통해 필요한 기능을 간접적으로 사용하는 것이 더 권장됩니다. Aware 인터페이스는 주로 프레임워크 내부 동작과 밀접하게 연관된 작업을 해야 하는 인프라스트럭처 빈 등에서 사용됩니다.

---

**세 번째 파트: 어노테이션 기반 이벤트 리스너 (`@EventListener`)**

이 부분은 스프링 4.2부터 도입된 `@EventListener` 어노테이션을 사용하여, **일반 메소드를 이벤트 리스너로 등록**하는 더 쉽고 유연한 방법을 설명합니다.

**핵심 아이디어:** 복잡하게 인터페이스 구현하지 말고, 이벤트 처리할 메소드 위에 `@EventListener` 어노테이션만 붙이자!

**사용법:**

1. **리스너 클래스:** 일반적인 스프링 빈 클래스(`@Component`, `@Service` 등)를 만듭니다. `ApplicationListener` 인터페이스를 구현할 **필요가 없습니다.**
2. **리스너 메소드:** 이벤트를 처리할 **메소드**를 만듭니다.
  - 메소드 이름은 자유롭게 지정할 수 있습니다.
  - 메소드의 **파라미터**로 수신하고 싶은 **이벤트 객체**를 선언합니다. (스프링이 알아서 해당 타입의 이벤트만 이 메소드로 전달해줍니다.)
3. **`@EventListener` 어노테이션:** 이벤트 처리 메소드 **위에** `@EventListener` 어노테이션을 붙입니다.

    ```java
    // Java
    import org.springframework.context.event.EventListener;
    import org.springframework.stereotype.Component;
    
    @Component // 일반 스프링 빈
    public class BlockedListNotifier {
        // ... (notificationAddress 필드 및 세터) ...
    
        // ★ @EventListener를 메소드 위에 붙임 ★
        // ★ 파라미터 타입(BlockedListEvent)으로 수신할 이벤트를 지정 ★
        @EventListener
        public void processBlockedListEvent(BlockedListEvent event) {
            System.out.println("어노테이션 리스너: 차단된 이메일 알림 - " + event.getAddress());
            // ... 알림 로직 ...
        }
    
        // 다른 이벤트 타입도 처리 가능
        @EventListener
        public void handleContextRefreshed(org.springframework.context.event.ContextRefreshedEvent event) {
            System.out.println("어노테이션 리스너: 컨텍스트 리프레시됨!");
        }
    }
    ```

    ```kotlin
    // Kotlin
    import org.springframework.context.event.EventListener
    import org.springframework.stereotype.Component
    
    @Component
    class BlockedListNotifier {
        // ... (notificationAddress 프로퍼티 및 세터) ...
    
        @EventListener // ★ 메소드 위에 붙임 ★
        fun processBlockedListEvent(event: BlockedListEvent) { // ★ 파라미터 타입으로 이벤트 지정 ★
            println("어노테이션 리스너: 차단된 이메일 알림 - ${event.address}")
            // ...
        }
    
        @EventListener
        fun handleContextRefreshed(event: org.springframework.context.event.ContextRefreshedEvent) {
            println("어노테이션 리스너: 컨텍스트 리프레시됨!")
        }
    }
    ```

- **지연 초기화 주의:** `@EventListener` 메소드가 있는 빈은 지연 초기화(lazy initialization) 대상으로 설정하면 안 됩니다. 지연 초기화될 경우 스프링이 해당 리스너 메소드를 등록하지 못해 이벤트를 수신할 수 없습니다.

**고급 기능:**

- **여러 이벤트 타입 처리:** `@EventListener` 어노테이션에 처리할 이벤트 클래스들을 배열 형태로 직접 명시할 수도 있습니다. 이 경우 메소드 파라미터는 생략하거나 `ApplicationEvent` 같은 상위 타입을 사용할 수 있습니다.

    ```java
    @EventListener({ContextStartedEvent.class, ContextRefreshedEvent.class})
    public void handleContextStartOrRefresh() { ... }
    ```

- **조건부 처리 (`condition` 속성):** `@EventListener`의 `condition` 속성에 **SpEL 표현식**을 사용하여 특정 조건이 만족될 때만 이벤트가 처리되도록 할 수 있습니다. SpEL 표현식 내에서는 이벤트 객체(`event` 또는 `#root.event`), 메소드 인수(`args` 또는 `#root.args`), 특정 인수 이름(예: `#blEvent`) 등을 참조할 수 있습니다.

    ```java
    // BlockedListEvent의 content 속성이 'my-event'일 때만 처리
    @EventListener(condition = "#blEvent.content == 'my-event'")
    public void processBlockedListEvent(BlockedListEvent blEvent) { ... }
    ```

- **이벤트 발행 (메소드 반환 값):** 리스너 메소드가 이벤트를 처리한 결과로 **또 다른 이벤트를 발행**해야 하는 경우, 메소드의 **반환 타입**을 발행할 이벤트 객체 타입으로 지정하고 해당 객체를 반환하면 됩니다. 스프링이 알아서 그 반환 객체를 새 이벤트로 발행해줍니다. (여러 개를 발행하려면 `Collection`이나 배열 반환)

    ```java
    @EventListener
    public ListUpdateEvent handleBlockedListEvent(BlockedListEvent event) {
        // ... 처리 로직 ...
        // 처리 결과로 ListUpdateEvent 발행
        return new ListUpdateEvent(this);
    }
    ```

  - **주의:** 이 기능은 **비동기 리스너에서는 지원되지 않습니다.** 비동기 리스너가 이벤트를 발행하려면 `ApplicationEventPublisher`를 직접 주입받아 사용해야 합니다.
- **비동기 처리 (`@Async`):** 특정 리스너 메소드를 **비동기적(별도 스레드)** 으로 실행하고 싶다면, 메소드 위에 `@Async` 어노테이션을 함께 붙여주면 됩니다. (단, 애플리케이션 설정 어딘가에 `@EnableAsync`가 활성화되어 있어야 함)

    ```java
    @EventListener
    @Async // 이 리스너는 별도 스레드에서 실행됨
    public void processBlockedListEvent(BlockedListEvent event) { ... }
    ```

  - **비동기 처리 제한:** 예외 전파가 다르고, 반환 값으로 이벤트 발행이 불가능하며, `ThreadLocal` 등이 기본적으로 전파되지 않는다는 점에 유의해야 합니다.
- **리스너 실행 순서 지정 (`@Order`):** 같은 이벤트를 처리하는 여러 `@EventListener` 메소드 간의 **실행 순서**를 제어하고 싶다면, 메소드 위에 `@Order(순서값)` 어노테이션을 붙입니다. 숫자가 낮을수록 우선순위가 높습니다 (먼저 실행됨).

    ```java
    @EventListener
    @Order(10) // 낮은 값, 먼저 실행될 가능성 높음
    public void firstListener(MyEvent event) { ... }
    
    @EventListener
    @Order(20) // 높은 값, 나중에 실행될 가능성 높음
    public void secondListener(MyEvent event) { ... }
    ```

- **제네릭 이벤트 처리:** 이벤트 클래스에 제네릭을 사용한 경우, 리스너 메소드의 파라미터 타입에 해당 제네릭 타입을 명시하여 특정 제네릭 타입의 이벤트만 수신하도록 필터링할 수 있습니다. (단, 이벤트 클래스가 `ResolvableTypeProvider`를 구현하거나, 제네릭 타입 정보가 명확한 하위 클래스 형태여야 타입 소거 문제를 피할 수 있음)

    ```java
    // EntityCreatedEvent<Person> 타입의 이벤트만 수신
    @EventListener
    public void onPersonCreated(EntityCreatedEvent<Person> event) { ... }
    ```


**요약:**

`@EventListener` 어노테이션은 `ApplicationListener` 인터페이스를 구현하는 번거로움 없이, **일반 메소드를 스프링 이벤트 리스너로 쉽게 등록**할 수 있게 해줍니다. 메소드 파라미터 타입을 통해 수신할 이벤트를 지정하며, SpEL을 이용한 조건부 처리, 반환 값을 통한 이벤트 발행, `@Async`를 이용한 비동기 처리, `@Order`를 이용한 순서 지정, 제네릭 이벤트 처리 등 다양하고 유연한 기능을 제공합니다.

---

**네 번째 파트: 저수준 리소스에 편리하게 접근하기 (`ResourceLoader`)**

이 부분은 스프링 애플리케이션이 파일 시스템의 파일, 클래스패스 상의 파일, 웹 URL 등 다양한 위치에 있는 **외부 자원(리소스)** 에 **일관된 방식**으로 접근할 수 있도록 스프링이 제공하는 **`Resource` 추상화**와 **`ResourceLoader`** 에 대한 내용입니다.

**핵심 아이디어:** 파일 위치가 어디든(내 컴퓨터, 서버, 클래스패스 안 등) 상관없이, 똑같은 방식으로 외부 자원을 읽어올 수 있게 하자!

---

**1. 문제점: 자바의 분산된 리소스 접근 방식**

- 자바 표준 라이브러리에서 외부 자원에 접근하는 방식은 자원의 위치에 따라 다릅니다.
  - 파일 시스템 파일: `java.io.File` 사용
  - 웹 URL: `java.net.URL` 사용
  - 클래스패스 리소스: `ClassLoader.getResourceAsStream()` 또는 `Class.getResourceAsStream()` 사용
- 이렇게 방식이 다르기 때문에 코드가 복잡해지고, 자원의 위치가 변경되면 코드 수정이 필요할 수 있습니다.

---

**2. 스프링의 해결책: `Resource` 인터페이스와 `ResourceLoader`**

- **`Resource` 인터페이스:** 스프링은 다양한 종류의 저수준 리소스(파일, URL, 클래스패스 리소스 등)를 **하나의 통일된 인터페이스(`org.springframework.core.io.Resource`)** 로 추상화했습니다. `Resource` 객체는 해당 리소스의 실제 존재 여부 확인, 내용 읽기(InputStream 얻기), 파일 객체 얻기(가능한 경우) 등의 기능을 제공합니다. JDK의 `java.net.URL` 클래스보다 더 많은 기능을 제공하며, 내부적으로 URL을 래핑하기도 합니다.
- **`ResourceLoader` 인터페이스:** 이 인터페이스는 **리소스 위치 문자열(location string)** 을 받아서 해당하는 **`Resource` 객체를 반환**해주는 역할을 합니다. "리소스 로더" 또는 "자원 탐색기"라고 생각할 수 있습니다.
  - **`ApplicationContext`는 `ResourceLoader`이다!** 모든 `ApplicationContext` 구현체는 기본적으로 `ResourceLoader` 인터페이스를 구현하고 있습니다. 따라서 `ApplicationContext` 객체를 사용하여 리소스를 로드할 수 있습니다.

---

**3. 리소스 위치 문자열 (Location Strings)과 접두사(Prefixes)**

`ResourceLoader` (즉, `ApplicationContext`)에게 리소스를 요청할 때는 **문자열 형태의 위치 경로**를 사용합니다. 스프링은 이 문자열 앞에 붙는 **특별한 접두사**를 보고 리소스의 종류를 판단하고 적절한 `Resource` 구현체를 반환합니다.

- **`classpath:` 접두사:** 클래스패스 경로에서 리소스를 찾습니다. (가장 흔하게 사용됨)
  - 예: `classpath:com/example/config.xml`
- **`file:` 접두사:** 파일 시스템 경로에서 리소스를 찾습니다.
  - 예: `file:/data/config.properties`, `file:C:/config/settings.xml`
- **`http:`, `https:` 접두사:** 웹 URL에서 리소스를 가져옵니다.
  - 예: `https://example.com/config.json`
- **접두사 없음 (Default):** 만약 접두사 없이 단순 경로만 주어지면, 리소스를 찾는 방식은 **사용하는 `ApplicationContext`의 구체적인 타입**에 따라 달라집니다.
  - `ClassPathXmlApplicationContext`: 기본적으로 클래스패스 경로로 간주합니다.
  - `FileSystemXmlApplicationContext`: 기본적으로 파일 시스템 경로로 간주합니다.
  - `WebApplicationContext`: 기본적으로 웹 애플리케이션 루트 경로(`ServletContext` 기준)로 간주합니다.
- **Ant 스타일 패턴:** `classpath*:` 접두사와 함께 이나 `*` 같은 Ant 스타일 와일드카드를 사용하여 여러 리소스를 한 번에 찾을 수도 있습니다.
  - 예: `classpath*:META-INF/spring/*.xml` (모든 JAR 파일 내의 해당 경로 XML 로드)

---

**4. 빈(Bean)에서 리소스 사용하기**

스프링 빈에서 `Resource`를 사용하는 방법은 여러 가지가 있습니다.

- **`ResourceLoaderAware` 사용:** 빈 클래스가 `ResourceLoaderAware` 인터페이스를 구현하면, 스프링이 자동으로 `ApplicationContext` (즉, `ResourceLoader`)를 주입해줍니다. 주입받은 로더를 사용하여 필요할 때 리소스를 직접 로드할 수 있습니다.

    ```java
    import org.springframework.context.ResourceLoaderAware;
    import org.springframework.core.io.Resource;
    import org.springframework.core.io.ResourceLoader;
    
    public class MyBean implements ResourceLoaderAware {
        private ResourceLoader resourceLoader;
    
        @Override
        public void setResourceLoader(ResourceLoader resourceLoader) {
            this.resourceLoader = resourceLoader;
        }
    
        public void loadResource() {
            Resource resource = this.resourceLoader.getResource("classpath:mydata.txt");
            // resource 사용 로직...
        }
    }
    ```

- **`@Autowired`로 `ResourceLoader` 주입:** `ResourceLoader` 자체도 빈처럼 취급될 수 있으므로, `@Autowired`를 사용하여 직접 주입받을 수도 있습니다. (더 선호되는 방식)

    ```java
    import org.springframework.beans.factory.annotation.Autowired;
    import org.springframework.core.io.ResourceLoader;
    // ...
    @Autowired
    private ResourceLoader resourceLoader;
    ```

- **`Resource` 타입 속성 직접 주입 (가장 편리):** 빈의 속성(필드 또는 세터 파라미터) 타입을 `Resource`로 선언하고, **설정(XML, Java Config, `@Value`)에서는 리소스 위치 문자열을 값으로 지정**하면, 스프링이 **자동으로 문자열을 해당 `Resource` 객체로 변환하여 주입**해줍니다.
  - **XML 설정:**

      ```xml
      <bean id="myBean" class="com.example.ResourceBean">
          <property name="template" value="classpath:templates/template.txt"/>
      </bean>
      ```

  - **Java 클래스:**

      ```java
      public class ResourceBean {
          private Resource template; // 타입은 Resource
      
          public void setTemplate(Resource template) { // 스프링이 문자열을 Resource 객체로 변환해줌
              this.template = template;
          }
      }
      ```

  - **`@Value` 어노테이션:**

      ```java
      @Value("classpath:templates/template.txt")
      private Resource template; // @Value로 리소스 경로 문자열 주입 시 자동으로 Resource 변환
      
      @Value("file:/data/config.xml")
      private Resource configFile;
      ```


**요약:**

`ApplicationContext`는 `ResourceLoader` 역할을 수행하여, 파일 시스템, 클래스패스, URL 등 다양한 위치의 외부 자원(리소스)에 **일관된 방식**으로 접근할 수 있게 해줍니다. 리소스 위치는 **접두사**가 붙은 문자열로 표현하며, 스프링은 이를 해석하여 적절한 `Resource` 객체를 반환합니다. 빈에서는 `ResourceLoaderAware`, `@Autowired ResourceLoader`, 또는 (가장 편리하게는) `Resource` 타입 속성에 리소스 위치 문자열을 직접 주입하는 방식으로 리소스를 사용할 수 있습니다.

**1. "저수준(Low-level)" vs "고수준(High-level)" 리소스 접근**

여기서 "저수준"과 "고수준"은 리소스를 다루는 **추상화의 정도**를 의미합니다.

- **저수준 리소스 접근 (Low-level Resource Access):**
  - **의미:** 리소스(파일, URL 등)의 **구체적인 위치나 종류(파일 시스템, 클래스패스, 웹 URL 등)를 직접 인지하고, 각 종류에 맞는 기본적인(low-level) API를 사용하여 리소스의 내용(주로 바이트 스트림)을 읽어오는 방식**을 의미합니다.
  - **예시:**
    - 파일 시스템 파일: java.io.FileInputStream을 직접 열어서 바이트를 읽는 것.
    - 클래스패스 리소스: ClassLoader.getResourceAsStream()을 호출하여 InputStream을 얻는 것.
    - URL 리소스: java.net.URL.openStream()을 호출하여 InputStream을 얻는 것.
  - **특징:** 프로토콜이나 위치에 따라 사용하는 API가 다르고, 주로 원시적인 데이터(바이트 스트림)를 다룹니다. 유연하지만 코드가 복잡해질 수 있습니다.
- **고수준 리소스 접근 (High-level Resource Access):**
  - **의미:** 리소스의 구체적인 위치나 종류보다는 **리소스가 담고 있는 '의미'나 '구조'에 더 초점을 맞춰** 다루는 방식입니다. 저수준 API를 직접 다루는 대신, 잘 만들어진 라이브러리나 프레임워크가 제공하는 편리한 기능을 사용합니다.
  - **예시:**
    - 스프링 설정 파일 로딩: ApplicationContext에 "classpath:beans.xml" 경로만 넘겨주면 스프링이 알아서 파일을 읽고 파싱하여 빈 객체를 만들어줍니다. 개발자는 파일 읽기 과정을 직접 코딩하지 않습니다.
    - MyBatis 매퍼 XML 로딩: MyBatis 설정에 매퍼 파일 경로만 지정하면 MyBatis가 내부적으로 파일을 읽고 SQL 매핑 정보를 로드합니다.
    - 템플릿 엔진 (Thymeleaf, Freemarker): 템플릿 파일 경로만 지정하면 엔진이 파일을 읽고 데이터를 바인딩하여 최종 HTML을 생성해줍니다.
  - **특징:** 개발자는 리소스의 내용을 어떻게 읽어올지에 대한 세부 사항보다는, 읽어온 데이터를 어떻게 사용할지에 더 집중할 수 있습니다. 코드가 간결해지고 생산성이 높아집니다.
- **결론:** 스프링의 Resource와 ResourceLoader는 **저수준 리소스 접근을 위한 통일된 방법을 제공**하는 인터페이스입니다. 이를 통해 개발자는 리소스의 구체적인 위치나 종류에 덜 신경 쓰면서 일관된 방식으로 리소스에 접근할 수 있게 되고, 스프링 프레임워크 자체나 다른 고수준 라이브러리들은 이 기반 위에서 편리한 기능들을 제공할 수 있습니다.

**2. 리소스(Resource)란 무엇을 지칭하는가?**

여기서 "리소스"는 단순히 로컬 파일 시스템의 파일만을 의미하는 것이 아니라, **애플리케이션이 접근해야 할 수 있는 모든 종류의 외부 자원**을 포괄적으로 의미합니다.

- **주요 예시:**
  - **파일 시스템 상의 파일:** (예: /etc/config.properties, C:\data\image.jpg)
  - **클래스패스 상의 파일:** JAR 파일 안에 포함되어 있거나, 빌드된 클래스 경로 내에 있는 파일 (예: 설정 파일 beans.xml, 프로퍼티 파일 messages.properties, SQL 스크립트 schema.sql) - **가장 흔하게 사용됩니다.**
  - **웹 URL 상의 자원:** (예: https://example.com/data.json, http://server/config)
  - **서블릿 컨텍스트 경로의 자원:** 웹 애플리케이션 루트 디렉토리 기준의 파일 (예: /WEB-INF/config.xml)
  - (이론적으로) FTP 서버의 파일, 데이터베이스의 BLOB 데이터 등 다양한 형태의 자원도 Resource 구현체를 통해 표현될 수 있습니다.

**3. Resource 객체는 무엇인가? (파일 자체를 담는 객체이자 인터페이스?)**

- **파일 자체를 담는 객체? -> 아니요!** Resource 객체는 파일의 **내용 전체를 메모리에 미리 로드해서 담고 있는 객체가 아닙니다.** (물론 내용은 읽어올 수 있습니다.)
- **인터페이스? -> 네, 맞습니다!** org.springframework.core.io.Resource는 **인터페이스**입니다.
- **역할:** Resource 인터페이스는 특정 리소스에 대한 **메타데이터(정보)와 접근 방법(API)** 을 **추상화**하여 제공합니다.
  - **메타데이터 제공:**
    - exists(): 리소스가 실제로 존재하는지 확인.
    - getFilename(): 리소스의 파일 이름 반환 (해당되는 경우).
    - getDescription(): 리소스에 대한 설명 문자열 반환 (예: "classpath resource [config.xml]").
    - contentLength(): 리소스의 크기 반환.
    - lastModified(): 마지막 수정 시간 반환.
  - **내용 접근 방법 제공:**
    - getInputStream(): 리소스의 내용을 읽을 수 있는 **InputStream** (바이트 스트림)을 반환합니다. **실제 내용을 읽는 것은 이 스트림을 사용할 때** 이루어집니다. (가장 중요하고 기본적인 방법)
    - getFile(): 만약 리소스가 파일 시스템 상의 파일이라면, 해당 java.io.File 객체를 반환합니다. (클래스패스 리소스 등은 파일 객체로 직접 접근 불가능할 수 있음)
    - getURL(): 리소스에 대한 java.net.URL 객체를 반환합니다.
    - isReadable(): 리소스 내용을 읽을 수 있는지 확인.
- **구현체:** 스프링은 리소스 종류별로 다양한 Resource 구현 클래스를 제공합니다. (예: ClassPathResource, FileSystemResource, UrlResource, ServletContextResource 등) ResourceLoader가 적절한 구현체를 반환해줍니다.

**4. ResourceLoader의 역할은 무엇인가?**

- **역할:** ResourceLoader는 **리소스 위치 문자열(예: "classpath:config.xml")을 해석하여 적절한 Resource 객체를 찾아 반환**해주는 **"리소스 로딩 전략"** 인터페이스입니다.
- **비유:** 마치 웹 브라우저 주소창에 URL을 입력하면 브라우저가 해당 웹 페이지(리소스)를 찾아서 보여주는 것처럼, ResourceLoader에게 리소스 경로를 알려주면 해당 Resource 객체를 찾아주는 역할을 합니다.
- **주요 메소드:** Resource getResource(String location)
  - location 문자열(접두사 포함)을 분석합니다.
  - 적절한 Resource 구현체(예: ClassPathResource, FileSystemResource)를 생성하고 설정하여 반환합니다.
- **ApplicationContext와의 관계:** 모든 ApplicationContext는 ResourceLoader 인터페이스를 구현하므로, ApplicationContext 자체가 리소스를 로드하는 기능을 가지고 있습니다. context.getResource("...") 처럼 사용할 수 있습니다.

**결론:**

- **저수준 리소스:** 파일 시스템, 클래스패스, URL 등 구체적인 위치에 있는 원시적인 형태의 외부 자원.
- **리소스:** 스프링에서 저수준 리소스를 추상화한 **인터페이스**. 리소스에 대한 정보와 내용에 접근하는 방법을 제공. (파일 내용 자체를 미리 담는 것은 아님)
- **ResourceLoader:** 리소스 위치 문자열을 받아 해당 **Resource 객체를 찾아 반환**해주는 역할. (ApplicationContext가 이 역할을 함)

---

**다섯 번째 파트: 애플리케이션 시작 추적 (Application Startup Tracking)**

이 부분은 스프링 애플리케이션이 **시작되는 동안 어떤 단계들을 거치고 각 단계에서 시간이 얼마나 걸리는지 측정하고 기록**하는 기능에 대한 설명입니다. 애플리케이션 시작 성능을 분석하고 병목 구간을 찾는 데 도움을 줄 수 있습니다.

**핵심 아이디어:** 스프링 부팅(Booting) 과정을 단계별로 기록해서 어디서 시간이 오래 걸리는지 알아보자!

---

**1. 왜 필요한가?**

- 복잡한 스프링 애플리케이션은 시작 시 많은 빈을 생성하고, 설정하고, 초기화하는 등 여러 단계를 거칩니다.
- 때로는 애플리케이션 시작 시간이 예상보다 오래 걸리는 경우가 있는데, **정확히 어떤 작업(빈 생성, 설정 처리 등)에서 시간이 많이 소요되는지 파악**하기 어려울 수 있습니다.
- 시작 단계 추적 기능은 이러한 **병목 지점을 식별**하고 성능 개선의 단서를 얻는 데 도움을 줍니다. 또한 컨테이너의 생명주기를 더 깊이 이해하는 데도 사용될 수 있습니다.

---

**2. 핵심 컴포넌트: ApplicationStartup 인터페이스와 StartupStep**

- **ApplicationStartup:** 스프링 컨테이너(AbstractApplicationContext 및 하위 클래스) 내부에 **시작 단계 데이터를 수집하는 역할**을 하는 컴포넌트입니다.
- **StartupStep:** 애플리케이션 시작 과정 중의 **개별적인 단계** (예: "패키지 스캔", "빈 인스턴스화", "이벤트 처리")를 나타내는 객체입니다. 각 StartupStep은 시작 시간, 종료 시간, 실행 시간, 그리고 추가적인 정보(태그, key-value)를 기록할 수 있습니다.
- **동작 방식:** 스프링 컨테이너는 내부적으로 중요한 작업들을 수행할 때 ApplicationStartup 객체를 사용하여 StartupStep을 시작(start())하고, 관련 정보를 태그(tag())로 추가하고, 작업이 끝나면 단계를 종료(end())합니다.

    ```java
    // 스프링 내부 코드 예시 (개념적)
    ApplicationStartup applicationStartup = getApplicationStartup(); // 컨텍스트로부터 가져옴
    StartupStep scanPackages = applicationStartup.start("spring.context.base-packages.scan"); // 단계 시작
    scanPackages.tag("packages", () -> Arrays.toString(basePackages)); // 관련 정보 태그 추가
    // ... 실제 패키지 스캔 작업 수행 ...
    scanPackages.end(); // 단계 종료 (실행 시간 기록됨)
    ```


---

**3. 스프링의 기본 계측(Instrumentation) 단계:**

스프링 컨테이너는 이미 다음과 같은 주요 단계들에 대해 StartupStep 데이터를 자동으로 기록하도록 계측되어 있습니다:

- **애플리케이션 컨텍스트 생명주기:** 기본 패키지 스캔, 구성 클래스 관리 등
- **빈 생명주기:** 인스턴스화, 스마트 초기화, 후처리 등
- **애플리케이션 이벤트 처리**

---

**4. ApplicationStartup 구현체 선택 및 활성화:**

- **기본 구현 (No-Op):** 기본적으로 스프링은 **아무 작업도 하지 않는(no-op)** ApplicationStartup 구현체를 사용합니다. 이는 성능 영향을 최소화하기 위함입니다. 따라서 **별도 설정을 하지 않으면 시작 단계 추적 데이터는 수집되지 않습니다.**
- **활성화 방법:** 시작 단계 데이터를 실제로 수집하고 사용하려면, **특정 ApplicationStartup 구현체를 사용하도록 설정**해야 합니다.
  - **FlightRecorderApplicationStartup:** 스프링 프레임워크에서 제공하는 구현체로, 수집된 데이터를 **Java Flight Recorder (JFR)** 형식으로 기록합니다. JFR은 JDK에 내장된 강력한 성능 분석 도구입니다.
  - **스프링 부트 (BufferingApplicationStartup 등):** 스프링 부트에서는 spring.application.admin.enabled=true 설정이나 특정 구현체 설정을 통해 데이터를 메모리에 버퍼링하고 나중에 액추에이터(/startup 엔드포인트) 등으로 조회하는 방식을 더 쉽게 제공합니다.
  - **커스텀 구현:** 필요하다면 직접 ApplicationStartup 인터페이스를 구현하여 데이터를 원하는 방식(로그 파일, 모니터링 시스템 등)으로 기록할 수도 있습니다.
- **설정 시점:** ApplicationStartup 구현체는 **ApplicationContext 객체가 생성된 직후, refresh()가 호출되기 전에 설정**해야 합니다.

    ```java
    // 예: Java Flight Recorder 사용 설정
    AnnotationConfigApplicationContext context = new AnnotationConfigApplicationContext();
    context.setApplicationStartup(new FlightRecorderApplicationStartup()); // ★ JFR 구현체 설정 ★
    context.register(AppConfig.class);
    context.refresh();
    ```

  **content_copydownload**Use code [**with caution**](https://support.google.com/legal/answer/13505487).Java


---

**5. 개발자의 커스텀 StartupStep 추가:**

- 개발자도 자신의 코드 내에서 중요한 작업 단계의 성능을 측정하고 싶다면, ApplicationStartup 객체를 얻어와서 커스텀 StartupStep을 기록할 수 있습니다.
- **ApplicationStartup 얻는 방법:**
  - ApplicationContext에서 직접 가져오기 (context.getApplicationStartup())
  - 빈 클래스가 ApplicationStartupAware 인터페이스를 구현하기
  - 빈의 생성자나 메소드 파라미터로 ApplicationStartup 타입 주입받기 (@Autowired)
- **사용 예시:**

    ```java
    @PostConstruct
    public void initialize() {
        StartupStep myStep = this.applicationStartup.start("my.custom.initialization"); // ★ 단계 시작
        myStep.tag("dataType", "configuration"); // ★ 태그 추가
        try {
            // ... 시간이 걸리는 초기화 작업 수행 ...
        } finally {
            myStep.end(); // ★ 단계 종료 ★
        }
    }
    ```

  **content_copydownload**Use code [**with caution**](https://support.google.com/legal/answer/13505487).Java

- **주의:** 커스텀 단계 이름에는 spring.* 네임스페이스를 사용하지 말아야 합니다. (스프링 내부용)

---

**6. 주의사항:**

- ApplicationStartup 기능은 애플리케이션 **시작 단계 분석**에 초점을 맞춘 기능입니다.
- 실행 중인 애플리케이션의 전반적인 성능 모니터링이나 상세한 프로파일링을 대체하는 기능은 아닙니다. 그러한 목적에는 **Micrometer** 같은 메트릭 라이브러리나 **Java 프로파일러**(VisualVM, JProfiler 등)를 사용하는 것이 적합합니다.

**요약:**

스프링의 애플리케이션 시작 추적 기능(ApplicationStartup)은 컨테이너가 시작되는 동안의 주요 단계(패키지 스캔, 빈 생성, 이벤트 처리 등)에 대한 **시간 및 관련 정보(StartupStep)를 기록**하는 메커니즘입니다. 기본적으로는 비활성화되어 있으며, FlightRecorderApplicationStartup 같은 구현체를 설정하여 활성화할 수 있습니다. 이를 통해 애플리케이션 **시작 성능 병목을 진단**하고 컨테이너 생명주기를 이해하는 데 도움을 받을 수 있습니다. 개발자도 커스텀 단계를 추가하여 추적할 수 있습니다.

**주요 확인 방법:**

1. **FlightRecorderApplicationStartup 사용 시 (스프링 프레임워크 표준 방식):**
  - **기록 방식:** 이 구현체는 수집된 StartupStep 데이터를 **Java Flight Recorder (JFR)** 시스템에 기록합니다. JFR은 실행 중인 JVM의 상세 정보를 담는 .jfr 파일을 생성합니다.
  - **결과 확인 방법:**
    - JVM 실행 시 JFR 기록을 활성화하는 옵션 (-XX:StartFlightRecording=... 등)을 주거나, 실행 중인 프로세스에서 jcmd 유틸리티를 사용하여 JFR 데이터를 덤프하여 **.jfr 파일을 생성**해야 합니다.
    - 생성된 .jfr 파일을 **Java Mission Control (JMC)** 이라는 도구나 IntelliJ IDEA Ultimate 같은 JFR 분석을 지원하는 IDE 또는 다른 프로파일링 도구로 열어봅니다.
    - JMC 등에서는 타임라인 뷰나 이벤트 브라우저 등에서 스프링 컨테이너 시작과 관련된 이벤트들(이름이 spring.* 또는 개발자가 정의한 커스텀 단계 이름으로 시작)을 찾을 수 있습니다. 각 단계의 **실행 시간, 계층 구조, 설정된 태그 정보** 등을 상세하게 분석할 수 있습니다.
2. **스프링 부트 환경 (BufferingApplicationStartup, Actuator 사용 시 - 더 일반적):**
  - **기록 방식:** 스프링 부트에서는 BufferingApplicationStartup 구현체를 사용하여 시작 단계 데이터를 **메모리(버퍼)** 에 저장하는 경우가 많습니다. (예: spring.application.admin.enabled=true 설정 시 활성화될 수 있음)
  - **결과 확인 방법:**
    - **Spring Boot Actuator의 /startup 엔드포인트:** Spring Boot Actuator 의존성을 추가하고 웹 애플리케이션으로 실행 중이라면, 웹 브라우저나 curl 같은 도구로 /actuator/startup 경로에 HTTP GET 요청을 보냅니다. (정확한 경로는 설정에 따라 다를 수 있음)
    - 응답으로 **JSON 형식**의 시작 단계 데이터가 반환됩니다. 여기에는 각 단계의 이름, ID, 부모-자식 관계, 시작/종료 시간, 실행 시간, 태그 등의 정보가 포함됩니다.
    - **Spring Boot Admin:** Spring Boot Admin 같은 관리 도구를 사용하고 있다면, 해당 UI에서 시작 단계 정보를 시각적으로 더 편리하게 확인할 수 있는 기능을 제공하는 경우가 많습니다.
3. **커스텀 ApplicationStartup 구현체 사용 시:**
  - **기록 방식:** 개발자가 직접 구현한 방식에 따라 다릅니다. 로그 파일에 기록하거나, 특정 데이터베이스에 저장하거나, 모니터링 시스템으로 전송할 수 있습니다.
  - **결과 확인 방법:** 개발자가 데이터를 저장하거나 전송하도록 **구현한 그 위치**(로그 파일, DB 테이블, 모니터링 대시보드 등)에서 확인해야 합니다.

**결론:**

StartupStep 데이터는 단순히 기록만 되는 것이 아니라, **선택하고 설정한 ApplicationStartup 구현체에 따라 특정 형식과 위치에 저장**됩니다. 그 결과를 확인하려면 **해당 구현체에 맞는 도구나 방법**(JMC, Actuator 엔드포인트, 로그 파일 등)을 사용해야 합니다. 어떤 구현체를 사용하도록 설정했는지 확인하는 것이 가장 중요합니다.

---

**여섯 번째 파트: 웹 애플리케이션을 위한 편리한 ApplicationContext 인스턴스화**

이 부분은 웹 애플리케이션 서버(WAS, 예: Tomcat)가 시작될 때 자동으로 스프링 `ApplicationContext`를 생성하고 초기화하도록 설정하는 **선언적인 방법**에 초점을 맞춥니다. 개발자가 직접 `ApplicationContext` 객체를 생성하는 코드를 작성할 필요 없이, `web.xml` 설정만으로 컨테이너를 구동할 수 있습니다.

**핵심 아이디어:** 웹 애플리케이션 시작 시, `web.xml` 설정을 통해 스프링 컨테이너를 자동으로 로딩하고 초기화하자!

---

**1. `ContextLoaderListener` 사용 (가장 일반적인 방법)**

- `org.springframework.web.context.ContextLoaderListener`는 **서블릿 표준 리스너(Servlet Listener)** 입니다. 서블릿 컨테이너(Tomcat 등)는 웹 애플리케이션이 시작되거나 종료될 때 특정 리스너 클래스에게 이벤트를 전달하는 기능을 제공합니다.
- **역할:** `ContextLoaderListener`는 웹 애플리케이션 **시작 이벤트**를 감지하여, **루트(Root) 웹 애플리케이션 컨텍스트 (Root WebApplicationContext)** 를 생성하고 초기화하는 역할을 합니다. 이 루트 컨텍스트는 웹 애플리케이션 전체에서 공유되며, 보통 서비스 계층(Service), 데이터 접근 계층(DAO) 등의 빈들을 포함합니다.
- **설정 (`web.xml`):**
  1. **리스너 등록:** `<listener>` 태그를 사용하여 `ContextLoaderListener`를 서블릿 컨테이너에 등록합니다.
  2. **설정 파일 위치 지정 (`contextConfigLocation`):** `<context-param>` 태그를 사용하여 `contextConfigLocation`이라는 이름의 파라미터를 설정합니다. 이 파라미터의 **값**으로 스프링 설정 파일의 위치를 지정합니다.
    - **XML 기반:** `/WEB-INF/applicationContext.xml`, `/WEB-INF/daoContext.xml` 처럼 **XML 파일 경로**를 공백, 쉼표(,), 세미콜론(;)으로 구분하여 여러 개 지정할 수 있습니다. Ant 스타일 패턴(예: `/WEB-INF/*Context.xml`)도 사용 가능합니다.
    - **Java 기반 (AnnotationConfigWebApplicationContext 사용 시):** 로드할 **`@Configuration` 클래스의 정규화된 이름**(예: `com.acme.AppConfig`) 또는 **컴포넌트 스캔할 패키지 이름**(예: `com.acme`)을 지정합니다.
  3. **(선택) 컨텍스트 클래스 지정 (`contextClass`):** 기본적으로 `ContextLoaderListener`는 XML 기반(`XmlWebApplicationContext`)을 로드하려고 합니다. 만약 **Java 기반 설정을 사용**한다면, `<context-param>`으로 `contextClass` 파라미터 값을 `org.springframework.web.context.support.AnnotationConfigWebApplicationContext`로 **명시적으로 지정**해야 합니다.
- **예시 (`web.xml` - XML 기반 설정 로드):**

    ```xml
    <web-app>
        <context-param>
            <param-name>contextConfigLocation</param-name>
            <!-- 여러 XML 파일 로드 -->
            <param-value>
                /WEB-INF/spring/root-context.xml
                /WEB-INF/spring/security-context.xml
            </param-value>
        </context-param>
    
        <listener>
            <listener-class>org.springframework.web.context.ContextLoaderListener</listener-class>
        </listener>
        <!-- ... (DispatcherServlet 등 다른 설정) ... -->
    </web-app>
    ```

- **예시 (`web.xml` - Java 기반 설정 로드):**

    ```xml
    <web-app>
        <!-- 사용할 컨텍스트 클래스를 AnnotationConfigWebApplicationContext로 지정 -->
        <context-param>
            <param-name>contextClass</param-name>
            <param-value>org.springframework.web.context.support.AnnotationConfigWebApplicationContext</param-value>
        </context-param>
    
        <!-- 로드할 @Configuration 클래스 또는 스캔할 패키지 지정 -->
        <context-param>
            <param-name>contextConfigLocation</param-name>
            <param-value>com.example.config.RootConfig, com.example.service</param-value>
        </context-param>
    
        <listener>
            <listener-class>org.springframework.web.context.ContextLoaderListener</listener-class>
        </listener>
        <!-- ... -->
    </web-app>
    ```


---

**2. 프로그래밍 방식 (참고):**

- 물론 `web.xml`을 사용하지 않고, 서블릿 3.0+ 환경이라면 `ServletContainerInitializer` 등을 사용하여 프로그래밍 방식으로 `ApplicationContext` (예: `AnnotationConfigWebApplicationContext`, `GenericWebApplicationContext`)를 생성하고 등록하는 것도 가능합니다. 스프링 부트는 내부적으로 이런 방식을 활용하여 `web.xml` 없는 구성을 가능하게 합니다.

**요약:**

웹 애플리케이션에서는 `web.xml`에 **`ContextLoaderListener`** 를 등록하고 **`contextConfigLocation`** 파라미터로 스프링 설정 파일(XML 경로 또는 `@Configuration` 클래스/패키지 이름)의 위치를 지정하는 것이 `ApplicationContext`를 **선언적으로, 자동으로 인스턴스화**하는 가장 일반적이고 편리한 방법입니다. Java 기반 설정을 사용할 경우 **`contextClass`** 파라미터를 `AnnotationConfigWebApplicationContext`로 지정해야 합니다. 이렇게 생성된 컨텍스트는 보통 애플리케이션 전체에서 공유되는 **루트 컨텍스트** 역할을 합니다.

**1. web.xml의 역할은 무엇인가요?**

web.xml 파일은 **자바 서블릿(Servlet) 표준 명세**에 따른 **웹 애플리케이션 배포 설명자(Deployment Descriptor)** 입니다. 쉽게 말해, **웹 서버(정확히는 서블릿 컨테이너, 예: Tomcat)** 에게 **"이 웹 애플리케이션을 어떻게 설정하고 실행해야 하는지 알려주는 공식 설명서"** 와 같습니다.

웹 서버는 웹 애플리케이션을 시작할 때 이 web.xml 파일을 읽어서 다음과 같은 중요한 정보들을 설정하고 구성합니다:

- **서블릿(Servlets) 선언 및 매핑:** 어떤 서블릿 클래스(예: 스프링 MVC의 DispatcherServlet)를 사용할 것이며, 어떤 URL 요청(예: /app/*)을 해당 서블릿으로 전달할지 정의합니다.
- **필터(Filters) 선언 및 매핑:** 요청(Request)이 서블릿에 도달하기 전이나 응답(Response)이 나가기 후에 공통적인 작업(예: 인코딩 설정, 보안 검사)을 수행할 필터를 정의하고 어떤 요청에 적용할지 설정합니다.
- **리스너(Listeners) 선언:** 웹 애플리케이션의 **생명주기 이벤트(시작, 종료)** 나 **세션(Session), 요청(Request)의 생성/소멸 이벤트**를 감지하고 특정 작업을 수행할 리스너 클래스를 등록합니다. **스프링의 ContextLoaderListener가 바로 여기에 해당합니다.**
- **컨텍스트 파라미터(Context Parameters):** 웹 애플리케이션 **전체에서 공유**될 설정 값들을 정의합니다. (<context-param> 태그). **스프링 설정 파일의 위치(contextConfigLocation)나 사용할 컨텍스트 클래스(contextClass)** 를 지정하는 데 주로 사용됩니다.
- **웰컴 파일(Welcome Files):** 사용자가 특정 경로(예: /)를 요청했을 때 기본적으로 보여줄 파일(예: index.html)을 지정합니다.
- **에러 페이지(Error Pages):** 특정 오류(예: 404 Not Found, 500 Internal Server Error) 발생 시 보여줄 페이지를 지정합니다.
- **세션 설정:** 세션 타임아웃 시간 등을 설정합니다.

**핵심:** web.xml은 웹 서버에게 **"이 웹 애플리케이션을 실행하기 위해 필요한 구성 요소들과 설정 값들은 이것들이니, 이걸 기반으로 애플리케이션을 준비하고 실행시켜줘"** 라고 지시하는 표준 방식의 설정 파일입니다. 스프링 프레임워크를 웹 환경에서 사용하기 위한 **초기 진입점 설정**이 여기서 이루어집니다.

**(참고)** 최신 스프링 부트(Spring Boot) 환경에서는 내장 웹 서버를 사용하고 자동 설정을 통해 web.xml 없이도 웹 애플리케이션을 구성하는 것이 일반적입니다. 하지만 전통적인 WAR 파일 배포 방식이나 스프링 부트 이전의 환경에서는 web.xml이 필수적이었습니다. 그 기본 개념을 이해하는 것은 여전히 중요합니다.

**스프링 부트(Spring Boot) 환경 - "마법"처럼 보이는 이유:**

- 스프링 부트는 개발 편의성을 극대화하기 위해 만들어졌습니다. 스프링 부트 애플리케이션을 실행하면 (main 메소드에서 SpringApplication.run(...) 호출) 다음과 같은 일들이 **내부적으로 자동**으로 일어납니다:
  - 클래스패스를 스캔하여 필요한 라이브러리(예: 웹 서버 관련 라이브러리)를 감지합니다.
  - 감지된 환경에 맞는 ApplicationContext 구현체(예: 웹 환경이면 AnnotationConfigServletWebServerApplicationContext)를 **자동으로 선택하고 생성**합니다.
  - @SpringBootApplication 어노테이션이 붙은 메인 클래스 위치를 기준으로 컴포넌트 스캔(@ComponentScan)을 **자동으로 수행**하여 @Component, @Service 등을 빈으로 등록합니다.
  - application.properties 또는 application.yml 파일을 **자동으로 로드**하여 설정 값으로 사용합니다.
- **결론:** 스프링 부트는 **개발자가 직접 컨테이너 생성 코드를 작성하거나 복잡한 설정을 하지 않아도**, SpringApplication.run() 한 줄로 이 모든 과정을 **자동으로 처리해주기 때문에** 마치 컨테이너가 저절로 만들어지는 것처럼 보이는 것입니다. 하지만 내부적으로는 스프링 부트가 대신 컨테이너를 **만들어주고 있는 것**입니다.

**2. 전통적인 스프링 환경 (스프링 부트 이전 또는 WAR 배포):**

- 스프링 부트가 나오기 전이나, 또는 스프링 부트를 사용하더라도 전통적인 방식인 WAR 파일로 톰캣 같은 외부 서블릿 컨테이너에 배포하는 경우에는 **자동으로 ApplicationContext가 생성되지 않습니다.**
- **이유:**
  - **독립 실행형 애플리케이션:** main 메소드를 가진 일반 자바 애플리케이션에서는, 개발자가 **직접 코드**로 new ClassPathXmlApplicationContext(...) 또는 new AnnotationConfigApplicationContext(...)를 호출하여 컨테이너를 생성해야 합니다.
  - **웹 애플리케이션 (WAR 배포):** 웹 서버(Tomcat 등)는 web.xml 파일을 읽어서 웹 애플리케이션을 구성합니다. 웹 서버는 **기본적으로 스프링의 존재를 알지 못합니다.** 따라서 개발자는 **web.xml 설정을 통해** 웹 서버에게 **"웹 애플리케이션이 시작될 때 스프링 컨테이너를 만들고 초기화해줘"** 라고 명시적으로 알려줘야 합니다.
    - 이때 사용하는 것이 바로 **ContextLoaderListener** 입니다. web.xml에 이 리스너를 등록하면, 웹 서버는 앱 시작 시 이 리스너를 실행합니다.
    - ContextLoaderListener는 web.xml의 contextConfigLocation 파라미터를 읽어서 **스프링 설정 파일을 찾고, 그 설정 파일을 기반으로 ApplicationContext를 생성**하는 역할을 합니다.
- **결론:** 스프링 부트가 아닌 환경에서는 개발자가 **코드(독립 실행형)나 설정(web.xml - 웹 앱)** 을 통해 ApplicationContext 생성을 **명시적으로 지시**해야 합니다. 여섯 번째 파트에서 설명하는 ContextLoaderListener 방식은 바로 이 **웹 애플리케이션 환경에서 컨테이너 생성을 지시하는 표준적이고 선언적인 방법**인 것입니다.

따라서, ApplicationContext는 원래 자동으로 만들어지는 것이 아니라, **스프링 부트가 그 과정을 매우 편리하게 자동화**해주거나, **전통적인 방식에서는 개발자가 직접 생성하거나 web.xml을 통해 생성을 지시**해야 하는 것입니다.

---

**일곱 번째 파트: 스프링 ApplicationContext를 Jakarta EE RAR 파일로 배포하기**

이 부분은 스프링 애플리케이션 컨텍스트와 관련 빈들, 라이브러리들을 **Jakarta EE (과거 Java EE) 표준 형식인 RAR (Resource ARchive) 파일**로 묶어서 **애플리케이션 서버(WAS)에 배포**하는 방법에 대한 내용입니다.

**핵심 아이디어:** 웹 인터페이스(HTTP 요청 처리)가 없는 스프링 애플리케이션(예: 메시지 처리, 스케줄링 작업 등)을 Jakarta EE 서버 환경에 배포하고 서버 자원을 활용하기 위한 표준적인 방법 중 하나이다.

---

**1. 왜 RAR 배포를 사용하는가?**

- **WAR 파일과의 비교:** 일반적으로 웹 애플리케이션은 WAR (Web ARchive) 파일로 배포됩니다. WAR 파일은 웹 리소스(HTML, CSS, JS)와 서블릿, JSP 등을 포함하며 HTTP 요청을 처리하는 것을 주 목적으로 합니다.
- **헤드리스(Headless) WAR의 대안:** 만약 스프링 애플리케이션이 HTTP 요청을 처리할 필요 없이, 백그라운드에서 동작하거나(예: 메시지 큐 리스너, 스케줄된 작업) 외부 시스템과 다른 방식으로 통신하는 경우, 빈 WAR 파일을 만들어 배포하는 것(헤드리스 WAR)보다 **RAR 파일 배포가 더 자연스럽고 표준적인 방법**일 수 있습니다. RAR은 원래 Jakarta EE Connector Architecture (JCA) 스펙의 일부로 외부 시스템과의 연결을 위한 리소스 어댑터를 배포하는 데 사용되지만, 스프링은 이를 활용하여 일반 애플리케이션 컨텍스트를 배포하는 방법도 제공합니다.
- **애플리케이션 서버 자원 활용:** RAR로 배포된 스프링 `ApplicationContext`는 해당 Jakarta EE 애플리케이션 서버가 제공하는 **다양한 자원과 서비스를 쉽게 활용**할 수 있습니다.
  - **JTA 트랜잭션 관리자:** 서버가 제공하는 분산 트랜잭션 관리 기능 사용.
  - **JNDI 자원:** 서버에 등록된 `DataSource` (DB 연결 풀), JMS `ConnectionFactory` (메시징) 등을 JNDI 룩업으로 사용.
  - **JMX 서버:** 스프링 빈들을 서버의 JMX 모니터링 시스템에 등록.
  - **JCA `WorkManager`:** 서버의 스레드 풀 관리 기능을 스프링의 `TaskExecutor`를 통해 사용.

---

**2. RAR 배포 방법 (간단 요약)**

1. **패키징:**
  - 애플리케이션의 모든 클래스 파일들과 필요한 라이브러리 JAR 파일들을 **하나의 아카이브 파일**로 묶습니다. 이 파일의 확장자를 `.rar`로 지정합니다. (기본적으로는 JAR 파일과 구조가 유사)
  - 필요한 라이브러리 JAR들은 보통 RAR 파일의 루트 디렉토리에 포함시킵니다.
2. **배포 기술자 (`ra.xml`):**
  - RAR 파일의 `META-INF` 디렉토리에 `ra.xml` 파일을 생성합니다. 이 파일은 RAR 배포의 표준 기술자 파일입니다.
  - `ra.xml` 안에는 스프링이 제공하는 **`SpringContextResourceAdapter`** 클래스를 리소스 어댑터로 지정하는 내용을 포함해야 합니다. 이 클래스가 RAR 배포 시 스프링 컨텍스트를 부트스트래핑하는 역할을 합니다.
3. **스프링 설정 파일:**
  - 일반적으로 `META-INF` 디렉토리 안에 스프링 빈 설정 파일(예: `applicationContext.xml`)을 위치시킵니다. `ra.xml`에서 이 파일의 위치를 참조하도록 설정할 수 있습니다.
4. **배포:**
  - 생성된 `.rar` 파일을 사용 중인 Jakarta EE 애플리케이션 서버의 배포 디렉토리에 복사하거나 관리 콘솔을 통해 배포합니다. 서버는 RAR 파일을 인식하고 `ra.xml` 설정에 따라 `SpringContextResourceAdapter`를 로드하여 스프링 `ApplicationContext`를 시작시킵니다.

---

**3. RAR 배포의 특징 및 상호 작용**

- **자체 포함 (Self-contained):** RAR 배포 단위는 일반적으로 필요한 모든 것을 포함하여 독립적으로 작동합니다.
- **외부 노출 제한:** 기본적으로 RAR 내의 컴포넌트들은 외부(다른 웹 애플리케이션 모듈 등)에 직접 노출되지 않습니다.
- **상호 작용 방식:**
  - **메시징:** 다른 모듈과 **JMS 큐(Queue)나 토픽(Topic)** 을 통해 메시지를 주고받는 방식으로 상호 작용하는 것이 일반적입니다.
  - **스케줄링/파일 감시 등:** 내부적으로 스케줄된 작업을 수행하거나, 파일 시스템의 변화를 감지하여 동작할 수 있습니다.
  - **원격 호출 (RMI 등):** 만약 외부에서 동기적인 호출이 필요하다면, RAR 내부의 빈이 RMI 엔드포인트 같은 원격 호출 인터페이스를 노출(export)할 수 있습니다.

**요약:**

스프링 `ApplicationContext`를 Jakarta EE 표준 형식인 **RAR 파일**로 배포하는 것은, 주로 **웹 인터페이스가 필요 없는 백그라운드 애플리케이션**을 Jakarta EE 서버 환경에 배포하고 서버 자원(JTA, JNDI, JMX, WorkManager 등)을 활용하기 위한 방법입니다. `ra.xml` 배포 기술자와 스프링 설정 파일을 포함하여 패키징하고 서버에 배포하면, 스프링 컨텍스트가 부트스트래핑됩니다. 외부와의 상호 작용은 주로 메시징이나 원격 호출 등을 통해 이루어집니다.

---
