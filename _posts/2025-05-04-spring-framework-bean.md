---
title: Spring Framework @Bean
description: 
author: laze
date: 2025-05-04 00:00:07 +0900
categories: [Dev, SpringBoot]
tags: [SpringBoot]
---
**Using the @Bean Annotation**

`@Bean`은 메소드 레벨 어노테이션이며 XML `<bean/>` 요소의 직접적인 유사체입니다.

이 어노테이션은 `<bean/>`이 제공하는 몇 가지 속성을 지원합니다.

- `init-method`
- `destroy-method`
- `autowiring`
- `name`.

`@Bean` 어노테이션은 `@Configuration`-어노테이션된 클래스 또는 `@Component`-어노테이션된 클래스에서 사용할 수 있습니다.

**빈 선언하기 (Declaring a Bean)**

빈을 선언하려면, `@Bean` 어노테이션으로 메소드에 어노테이션을 달 수 있습니다.

이 메소드를 사용하여 메소드의 반환 값으로 지정된 타입의 `ApplicationContext` 내에 빈 정의를 등록합니다.

기본적으로 빈 이름은 메소드 이름과 동일합니다. 다음 예제는 `@Bean` 메소드 선언을 보여줍니다:

```java
// Java
@Configuration
public class AppConfig {

	@Bean
	public TransferServiceImpl transferService() {
		return new TransferServiceImpl();
	}
}
```

```kotlin
// Kotlin
@Configuration
class AppConfig {

    @Bean
    fun transferService(): TransferServiceImpl {
        return TransferServiceImpl()
    }
}
// Assuming TransferServiceImpl exists
```

앞의 구성은 다음 스프링 XML과 정확히 동일합니다:

```xml
<beans>
	<bean id="transferService" class="com.acme.TransferServiceImpl"/>
</beans>
```

두 선언 모두 다음 텍스트 이미지에서 보여주듯이 `transferService`라는 이름의 빈을 `ApplicationContext`에서 사용할 수 있게 만들고, `TransferServiceImpl` 타입의 객체 인스턴스에 바인딩합니다:

```
transferService -> com.acme.TransferServiceImpl
```

기본 메소드(default methods)를 사용하여 빈을 정의할 수도 있습니다. 이는 기본 메소드에 빈 정의가 있는 인터페이스를 구현하여 빈 구성의 조합(composition)을 허용합니다.

```java
// Java
public interface BaseConfig {

	@Bean
	default TransferServiceImpl transferService() { // Default method in interface
		return new TransferServiceImpl();
	}
}

@Configuration
public class AppConfig implements BaseConfig { // Implements interface with default @Bean method
	// No need to redefine the bean here if the default is sufficient
}
```

다음 예제와 같이 인터페이스(또는 기본 클래스) 반환 타입으로 `@Bean` 메소드를 선언할 수도 있습니다:

```java
// Java
@Configuration
public class AppConfig {

	@Bean
	public TransferService transferService() { // Return type is interface
		return new TransferServiceImpl(); // Return implementation
	}
}
```

```kotlin
// Kotlin
@Configuration
class AppConfig {

    @Bean
    fun transferService(): TransferService { // Return type is interface
        return TransferServiceImpl() // Return implementation
    }
}
// Assuming TransferService interface and TransferServiceImpl implementation exist
```

그러나 이는 사전 타입 예측(advance type prediction)의 가시성을 지정된 인터페이스 타입(`TransferService`)으로 제한합니다.

그러면 영향을 받는 싱글톤 빈이 인스턴스화된 후에야 컨테이너가 전체 타입(`TransferServiceImpl`)을 알게 됩니다.

지연 초기화되지 않는(Non-lazy) 싱글톤 빈은 선언 순서에 따라 인스턴스화되므로, 다른 컴포넌트가 선언되지 않은 타입으로 매칭을 시도하는 시점에 따라 다른 타입 매칭 결과를 볼 수 있습니다 (예: `@Autowired TransferServiceImpl`은 `transferService` 빈이 인스턴스화된 후에만 해결됨).

*선언된 서비스 인터페이스로 타입을 일관되게 참조한다면, `@Bean` 반환 타입은 해당 설계 결정을 안전하게 따를 수 있습니다.*

*그러나 여러 인터페이스를 구현하는 컴포넌트나 구현 타입으로 잠재적으로 참조될 수 있는 컴포넌트의 경우, 가능한 가장 구체적인 반환 타입을 선언하는 것이 더 안전합니다 (적어도 빈을 참조하는 주입 지점에서 요구하는 만큼 구체적이어야 함).*

**빈 의존성 (Bean Dependencies)**

`@Bean`-어노테이션된 메소드는 해당 빈을 빌드하는 데 필요한 의존성을 설명하는 임의 개수의 파라미터를 가질 수 있습니다.

예를 들어, `TransferService`가 `AccountRepository`를 필요로 한다면, 다음 예제와 같이 메소드 파라미터로 해당 의존성을 구체화(materialize)할 수 있습니다:

```java
// Java
@Configuration
public class AppConfig {

	@Bean
	public TransferService transferService(AccountRepository accountRepository) { // Dependency as parameter
		return new TransferServiceImpl(accountRepository);
	}
}
```

```kotlin
// Kotlin
@Configuration
class AppConfig {

    @Bean
    fun transferService(accountRepository: AccountRepository): TransferService { // Dependency as parameter
        return TransferServiceImpl(accountRepository)
    }
}
// Assuming AccountRepository, TransferService, TransferServiceImpl exist
```

해석 메커니즘은 생성자 기반 의존성 주입과 거의 동일합니다.

**생명주기 콜백 받기 (Receiving Lifecycle Callbacks)**

`@Bean` 어노테이션으로 정의된 모든 클래스는 일반적인 생명주기 콜백을 지원하며 JSR-250의 `@PostConstruct` 및 `@PreDestroy` 어노테이션을 사용할 수 있습니다.

일반적인 스프링 생명주기 콜백도 완전히 지원됩니다.

빈이 `InitializingBean`, `DisposableBean` 또는 `Lifecycle`을 구현하면, 해당 메소드는 컨테이너에 의해 호출됩니다.

표준 `*Aware` 인터페이스 세트(예: `BeanFactoryAware`, `BeanNameAware`, `MessageSourceAware`, `ApplicationContextAware` 등)도 완전히 지원됩니다.

`@Bean` 어노테이션은 다음 예제와 같이 `bean` 요소의 스프링 XML `init-method` 및 `destroy-method` 속성과 마찬가지로 임의의 초기화 및 소멸 콜백 메소드 지정을 지원합니다:

```java
// Java
public class BeanOne {
	public void init() {
		// 초기화 로직
	}
}

public class BeanTwo {
	public void cleanup() {
		// 소멸 로직
	}
}

@Configuration
public class AppConfig {

	@Bean(initMethod = "init")
	public BeanOne beanOne() {
		return new BeanOne();
	}

	@Bean(destroyMethod = "cleanup")
	public BeanTwo beanTwo() {
		return new BeanTwo();
	}
}
```

```kotlin
// Kotlin
class BeanOne {
    fun init() {
        // 초기화 로직
    }
}

class BeanTwo {
    fun cleanup() {
        // 소멸 로직
    }
}

@Configuration
class AppConfig {

    @Bean(initMethod = "init")
    fun beanOne(): BeanOne {
        return BeanOne()
    }

    @Bean(destroyMethod = "cleanup")
    fun beanTwo(): BeanTwo {
        return BeanTwo()
    }
}
```

*기본적으로, public `close` 또는 `shutdown` 메소드를 가진 자바 구성으로 정의된 빈은 자동으로 소멸 콜백에 등록됩니다.*

*public `close` 또는 `shutdown` 메소드가 있고 컨테이너 종료 시 호출되기를 원하지 않는 경우, 빈 정의에 `@Bean(destroyMethod = "")`을 추가하여 기본 (추론된) 모드를 비활성화할 수 있습니다.*

*애플리케이션 외부에서 생명주기가 관리되므로 JNDI로 획득하는 리소스에 대해 기본적으로 이 작업을 수행할 수 있습니다.*

*특히, Jakarta EE 애플리케이션 서버에서 문제가 있는 것으로 알려져 있으므로 `DataSource`에 대해 항상 수행해야 합니다.*

다음 예제는 `DataSource`에 대한 자동 소멸 콜백을 방지하는 방법을 보여줍니다:

```java
// Java
@Bean(destroyMethod = "") // Disable inferred destroy method
public DataSource dataSource() throws NamingException {
	return (DataSource) jndiTemplate.lookup("MyDS");
}
```

```kotlin
// Kotlin
@Bean(destroyMethod = "") // Disable inferred destroy method
@Throws(NamingException::class)
fun dataSource(): DataSource {
    // Assuming jndiTemplate is available
    return jndiTemplate.lookup("MyDS") as DataSource
}
```

*또한, `@Bean` 메소드를 사용하면 일반적으로 스프링의 `JndiTemplate` 또는 `JndiLocatorDelegate` 헬퍼를 사용하거나 직접적인 JNDI `InitialContext` 사용을 통해 프로그래밍 방식 JNDI 룩업을 사용하며,*

`*JndiObjectFactoryBean` 변형은 사용하지 않습니다 (이는 반환 타입을 실제 대상 타입 대신 `FactoryBean` 타입으로 선언하도록 강제하여, 여기서 제공된 리소스를 참조하려는 다른 `@Bean` 메소드의 상호 참조 호출에 사용하기 어렵게 만듦).*

위의 `BeanOne` 예제의 경우, 다음 예제와 같이 생성 중에 `init()` 메소드를 직접 호출하는 것도 똑같이 유효합니다:

```java
// Java
@Configuration
public class AppConfig {

	@Bean
	public BeanOne beanOne() {
		BeanOne beanOne = new BeanOne();
		beanOne.init(); // Call init() directly
		return beanOne;
	}

	// ...
}
```

```kotlin
// Kotlin
@Configuration
class AppConfig {

    @Bean
    fun beanOne(): BeanOne {
        val beanOne = BeanOne()
        beanOne.init() // Call init() directly
        return beanOne
    }

    // ...
}
```

*자바에서 직접 작업할 때, 객체로 원하는 모든 작업을 수행할 수 있으며 항상 컨테이너 생명주기에 의존할 필요는 없습니다.*

**빈 스코프 지정하기 (Specifying Bean Scope)**

스프링은 빈의 스코프를 지정할 수 있도록 `@Scope` 어노테이션을 포함합니다.

*@Scope 어노테이션 사용하기 (Using the @Scope Annotation)*`@Bean` 어노테이션으로 정의된 빈이 특정 스코프를 가져야 함을 지정할 수 있습니다.

기본 스코프는 `singleton`이지만, 다음 예제와 같이 `@Scope` 어노테이션으로 이를 오버라이드할 수 있습니다:

```java
// Java
@Configuration
public class MyConfiguration {

	@Bean
	@Scope("prototype")
	public Encryptor encryptor() {
		// ...
	}
}
```

```kotlin
// Kotlin
@Configuration
class MyConfiguration {

    @Bean
    @Scope("prototype")
    fun encryptor(): Encryptor {
        // ...
        return Encryptor() // Assuming Encryptor exists
    }
}
```

*@Scope 및 scoped-proxy (@Scope and scoped-proxy)*
스프링은 스코프 프록시(scoped proxies)를 통해 스코프 의존성 작업을 위한 편리한 방법을 제공합니다.

XML 구성을 사용할 때 이러한 프록시를 생성하는 가장 쉬운 방법은 `<aop:scoped-proxy/>` 요소입니다.

`@Scope` 어노테이션을 사용하여 자바로 빈을 구성하는 것은 `proxyMode` 속성을 통해 동등한 지원을 제공합니다.

기본값은 `ScopedProxyMode.DEFAULT`이며, 이는 일반적으로 컴포넌트 스캔 지시 레벨에서 다른 기본값이 구성되지 않은 한 스코프 프록시가 생성되지 않아야 함을 나타냅니다.

`ScopedProxyMode.TARGET_CLASS`, `ScopedProxyMode.INTERFACES` 또는 `ScopedProxyMode.NO`를 지정할 수 있습니다.

```java
// Java
// 프록시로 노출된 HTTP Session 스코프 빈
@Bean
@SessionScope // Combines @Scope("session") and default proxyMode = TARGET_CLASS
public UserPreferences userPreferences() {
	return new UserPreferences();
}

@Bean
public Service userService() {
	UserService service = new SimpleUserService();
	// 프록시된 userPreferences 빈에 대한 참조
	service.setUserPreferences(userPreferences()); // Injects the proxy
	return service;
}
```

```kotlin
// Kotlin
// 프록시로 노출된 HTTP Session 스코프 빈
@Bean
@SessionScope // Combines @Scope("session") and default proxyMode = TARGET_CLASS
fun userPreferences(): UserPreferences {
    return UserPreferences()
}

@Bean
fun userService(): UserService { // Changed return type to UserService
    val service = SimpleUserService()
    // 프록시된 userPreferences 빈에 대한 참조
    service.userPreferences = userPreferences() // Assuming setter or property access
    return service
}
// Assuming UserPreferences, UserService, SimpleUserService exist
```

**빈 이름 사용자 정의하기 (Customizing Bean Naming)**

기본적으로 구성 클래스는 `@Bean` 메소드의 이름을 결과 빈의 이름으로 사용합니다. 그러나 다음 예제와 같이 `name` 속성으로 이 기능을 오버라이드할 수 있습니다:

```java
// Java
@Configuration
public class AppConfig {

	@Bean("myThing") // Specify the bean name explicitly
	public Thing thing() {
		return new Thing();
	}
}
```

```kotlin
// Kotlin
@Configuration
class AppConfig {

    @Bean("myThing") // Specify the bean name explicitly
    fun thing(): Thing {
        return Thing()
    }
}
// Assuming Thing exists
```

**빈 별칭 지정하기 (Bean Aliasing)**

빈 이름 지정(Naming Beans)에서 논의된 바와 같이, 때로는 단일 빈에 여러 이름, 즉 빈 별칭(bean aliasing)을 지정하는 것이 바람직합니다. `@Bean` 어노테이션의 `name` 속성은 이 목적을 위해 `String` 배열을 받습니다. 다음 예제는 빈에 여러 별칭을 설정하는 방법을 보여줍니다:

```java
// Java
@Configuration
public class AppConfig {

	@Bean({"dataSource", "subsystemA-dataSource", "subsystemB-dataSource"}) // Multiple names (aliases)
	public DataSource dataSource() {
		// DataSource 빈 인스턴스화, 구성 및 반환...
		return new BasicDataSource(); // Example implementation
	}
}
```

```kotlin
// Kotlin
@Configuration
class AppConfig {

    @Bean(name = ["dataSource", "subsystemA-dataSource", "subsystemB-dataSource"]) // Multiple names (aliases)
    fun dataSource(): DataSource {
        // DataSource 빈 인스턴스화, 구성 및 반환...
        return BasicDataSource() // Example implementation
    }
}
// Assuming DataSource and BasicDataSource exist
```

**빈 설명 (Bean Description)**

때로는 빈에 대한 더 자세한 텍스트 설명을 제공하는 것이 도움이 됩니다.

이는 빈이 모니터링 목적으로 (아마도 JMX를 통해) 노출될 때 특히 유용할 수 있습니다.

`@Bean`에 설명을 추가하려면 다음 예제와 같이 `@Description` 어노테이션을 사용할 수 있습니다:

```java
// Java
@Configuration
public class AppConfig {

	@Bean
	@Description("Provides a basic example of a bean")
	public Thing thing() {
		return new Thing();
	}
}
```

```kotlin
// Kotlin
@Configuration
class AppConfig {

    @Bean
    @Description("Provides a basic example of a bean")
    fun thing(): Thing {
        return Thing()
    }
}
// Assuming Thing exists
```

---

**전체 주제: `@Bean` 어노테이션 사용하기**

이 부분은 XML의 `<bean/>` 태그를 대체하여, 자바 메소드를 통해 스프링 빈을 **선언(정의), 생성, 설정**하는 방법에 대한 상세 설명입니다.

**핵심 아이디어:** 자바 메소드를 사용하여 스프링 빈을 만들고 설정하자!

---

**1. 빈 선언하기 (`@Bean`의 기본 역할)**

- * `@Bean` 어노테이션:** 메소드 레벨에 붙입니다.
- **역할:** 이 어노테이션이 붙은 메소드는 스프링 컨테이너에게 **"이 메소드가 실행된 결과(반환 객체)를 스프링 빈으로 등록하고 관리해줘!"** 라고 알려줍니다.
- **동작:** 스프링 컨테이너는 이 메소드를 호출하고, 그 반환 객체를 컨테이너에 등록합니다.
- **빈 이름:** 기본적으로 **메소드 이름**이 해당 빈의 ID가 됩니다. (예: `transferService()` -> 빈 ID: `transferService`)
- **XML 등가물:** `<bean id="메소드이름" class="반환타입클래스"/>` 와 거의 동일합니다.

    ```java
    @Configuration
    public class AppConfig {
        @Bean // 이 메소드가 transferService 빈을 정의함
        public TransferServiceImpl transferService() {
            // 1. 객체 생성
            TransferServiceImpl service = new TransferServiceImpl();
            // 2. (필요하다면) 객체 설정 (예: setter 호출)
            // service.setSomeProperty(...);
            // 3. 설정된 객체 반환 -> 이것이 빈으로 등록됨
            return service;
        }
    }
    ```

- **인터페이스 반환 타입:** `@Bean` 메소드의 반환 타입을 **구현 클래스(`TransferServiceImpl`)** 대신 **인터페이스(`TransferService`)** 로 선언할 수도 있습니다. 이는 좋은 설계 관행이지만, 스프링 컨테이너가 빈의 실제 타입을 미리 예측하는 데 제약을 줄 수 있습니다. 다른 빈에서 구체적인 구현 타입(`TransferServiceImpl`)으로 주입받으려고 할 때, 해당 `@Bean` 메소드가 실행(빈 생성)되기 전까지는 타입 매칭이 안 될 수도 있습니다. 가능하면 주입 지점에서 필요한 만큼 구체적인 타입을 반환하는 것이 안전합니다.
- **자바 8 기본 메소드:** 인터페이스의 `default` 메소드에도 `@Bean`을 사용하여 빈을 정의할 수 있습니다. 이를 통해 인터페이스를 구현하는 `@Configuration` 클래스에서 빈 설정을 조합할 수 있습니다.

---

**2. 빈 의존성 처리 (`@Bean` 메소드 파라미터)**

- `@Bean` 메소드가 정의하는 빈이 **다른 빈에 의존**할 경우, 그 의존성을 `@Bean` 메소드의 **파라미터**로 선언할 수 있습니다.
- **동작:** 스프링 컨테이너는 `@Bean` 메소드를 호출하기 전에, 해당 파라미터 타입과 일치하는 빈을 컨테이너 내에서 찾아서 **자동으로 주입(전달)** 해줍니다. (마치 `@Autowired` 생성자 주입과 유사하게 동작)

    ```java
    @Configuration
    public class AppConfig {
    
        @Bean // accountRepository 빈 정의 (가정)
        public AccountRepository accountRepository() {
            return new JdbcAccountRepository();
        }
    
        @Bean
        // ★ accountRepository 파라미터로 의존성 주입 요청 ★
        public TransferService transferService(AccountRepository accountRepository) {
            // 스프링이 accountRepository 빈을 찾아서 파라미터로 전달해줌
            return new TransferServiceImpl(accountRepository); // 생성자를 통해 의존성 주입
        }
    }
    ```


---

**3. 생명주기 콜백 받기**

`@Bean`으로 정의된 빈도 일반적인 스프링 빈과 동일하게 **생명주기 콜백**을 처리할 수 있습니다.

- **JSR-250 어노테이션:** 빈 클래스 내부에 `@PostConstruct` (초기화) 및 `@PreDestroy` (소멸) 어노테이션을 사용하면 스프링이 자동으로 인식하고 호출합니다. (가장 권장되는 방식)
- **스프링 인터페이스:** 빈 클래스가 `InitializingBean` (`afterPropertiesSet()`) 또는 `DisposableBean` (`destroy()`) 인터페이스를 구현하면 해당 메소드가 호출됩니다. (비추천)
- **`Aware` 인터페이스:** `BeanNameAware`, `ApplicationContextAware` 등 표준 Aware 인터페이스들도 지원됩니다.
- **`@Bean` 어노테이션 속성 (XML 유사):** `@Bean` 어노테이션 자체에 `initMethod`와 `destroyMethod` 속성을 사용하여, 빈 클래스 내의 특정 메소드를 초기화/소멸 콜백으로 지정할 수 있습니다.

    ```java
    @Configuration
    public class AppConfig {
        @Bean(initMethod = "init", destroyMethod = "cleanup") // ★ 속성으로 콜백 메소드 지정 ★
        public MyCustomBean myCustomBean() {
            return new MyCustomBean(); // MyCustomBean 클래스에는 init(), cleanup() 메소드가 있어야 함
        }
    }
    ```

- **자동 소멸 메소드 추론:** 기본적으로 `@Bean` 정의 시, 반환되는 객체 클래스에 `public void close()` 또는 `public void shutdown()` 메소드가 있다면 스프링이 **자동으로 이를 소멸 콜백으로 등록**합니다. 만약 이를 원치 않는다면 `@Bean(destroyMethod = "")` 처럼 빈 문자열을 지정하여 비활성화해야 합니다. (특히 JNDI 룩업으로 얻어온 `DataSource`처럼 외부에서 관리되는 리소스의 경우 필수)
- **직접 메소드 호출:** `@Bean` 메소드 내부에서 객체를 생성한 후, 반환하기 전에 직접 초기화 메소드(`beanOne.init()`)를 호출하는 것도 가능합니다. (자바 코드를 사용하므로 유연함)

---

**4. 빈 스코프 지정하기 (`@Scope`)**

- `@Bean`으로 정의된 빈의 **스코프**를 지정하려면, `@Bean` 어노테이션과 함께 `@Scope` 어노테이션을 사용합니다.
- 기본 스코프는 `singleton`입니다.

    ```java
    @Configuration
    public class MyConfiguration {
        @Bean
        @Scope("prototype") // ★ 프로토타입 스코프 지정 ★
        public Encryptor encryptor() {
            return new Encryptor();
        }
    }
    ```

- **스코프 프록시 (`@Scope`의 `proxyMode` 속성):** request, session, prototype 등 싱글톤이 아닌 스코프를 사용할 때 프록시가 필요한 경우, `@Scope` 어노테이션의 `proxyMode` 속성을 사용하여 프록시 생성 방식을 지정할 수 있습니다 (`ScopedProxyMode.TARGET_CLASS` 또는 `ScopedProxyMode.INTERFACES`).
  - 스프링은 `@RequestScope`, `@SessionScope` 등 편리한 조합 어노테이션도 제공합니다. (예: `@SessionScope`는 `@Scope("session", proxyMode = ScopedProxyMode.TARGET_CLASS)`와 유사)

    ```java
    @Bean
    @SessionScope // == @Scope("session", proxyMode=ScopedProxyMode.TARGET_CLASS)
    public UserPreferences userPreferences() {
        return new UserPreferences();
    }
    
    @Bean
    public UserService userService() {
        UserService service = new SimpleUserService();
        // userPreferences() 호출 시 프록시 객체가 주입됨
        service.setUserPreferences(userPreferences());
        return service;
    }
    ```


---

**5. 빈 이름 사용자 정의 및 별칭 지정**

- **이름 지정:** `@Bean` 어노테이션의 `name` (또는 `value`) 속성을 사용하여 기본값(메소드 이름) 대신 **다른 이름(ID)** 을 빈에 부여할 수 있습니다.

    ```java
    @Bean("myThing") // 빈 이름을 "myThing"으로 지정
    public Thing thing() {
        return new Thing();
    }
    ```

- **별칭(Alias) 지정:** `name` 속성에 **문자열 배열**을 제공하여 하나의 빈에 여러 개의 이름(첫 번째가 주 이름, 나머지는 별칭)을 부여할 수 있습니다.

    ```java
    @Bean(name = {"dataSource", "subsystemA-dataSource", "subsystemB-dataSource"})
    public DataSource dataSource() {
        // ...
    }
    ```


---

**6. 빈 설명 추가 (`@Description`)**

- `@Bean` 메소드 위에 `@Description` 어노테이션을 사용하여 해당 빈에 대한 **설명 문자열**을 추가할 수 있습니다. 이는 주로 JMX 등을 통한 모니터링 시 유용합니다.

    ```java
    @Bean
    @Description("Provides a basic example of a bean")
    public Thing thing() {
        return new Thing();
    }
    ```


**요약:**

`@Bean` 어노테이션은 자바 메소드를 사용하여 스프링 빈을 정의하는 핵심 메커니즘입니다. 메소드 이름이 빈 이름이 되고 반환 객체가 빈 인스턴스가 됩니다.

메소드 파라미터를 통해 의존성을 주입받을 수 있으며, 다양한 생명주기 콜백 메커니즘을 지원합니다. `@Scope`, `@Qualifier`, `@Lazy` 등 다른 어노테이션과 함께 사용하여 빈의 스코프, 이름, 별칭, 설명 등을 세밀하게 제어할 수 있습니다. `@Configuration` 클래스 내에서 사용하는 것이 일반적이며, 빈 간의 의존성 처리 방식에 차이가 있을 수 있습니다.

**1. 왜 `proxyMode`가 필요한가?**

- **문제:** **수명이 긴 빈(예: 싱글톤 `UserService`)** 이 **수명이 짧은 빈(예: 세션 스코프 `UserPreferences`)** 을 주입받아야 할 때 발생합니다.
- 싱글톤 `UserService`는 딱 한 번 생성되고, 이때 주입받은 `UserPreferences` 객체도 딱 하나입니다.
- 하지만 `UserService`는 여러 사용자가 공유하는데, 각 사용자 요청 시에는 **해당 사용자의 세션에 맞는 `UserPreferences` 객체**를 사용해야 합니다. 처음 주입받은 것만 계속 쓰면 안 됩니다.

**2. 해결책: 스코프 프록시**

- 스프링은 이 문제를 해결하기 위해 실제 빈(`UserPreferences`) 대신 **가짜 객체(프록시)** 를 `UserService`에 주입합니다.
- `UserService`는 이 프록시를 진짜 `UserPreferences`인 줄 알고 사용합니다.
- `UserService`가 프록시의 메소드를 호출하면, 프록시는 **현재 요청의 스코프(여기서는 세션)를 파악**하고, 그 스코프에 맞는 **실제 `UserPreferences` 객체를 찾아서** 메소드 호출을 **대신 전달(위임)** 해줍니다.

**3. `@Scope`의 `proxyMode` 속성: 프록시 생성 방법 지정**

`@Scope` 어노테이션의 `proxyMode` 속성은 이 스코프 프록시를 **어떻게 만들지** 스프링에게 알려주는 역할을 합니다. `ScopedProxyMode`라는 enum 타입을 값으로 사용하며, 주요 옵션은 다음과 같습니다:

- **`ScopedProxyMode.DEFAULT` (또는 `ScopedProxyMode.NO`):**
  - **기본값**입니다.
  - **프록시를 생성하지 않습니다.** 만약 컴포넌트 스캔 설정(`@ComponentScan`의 `scopedProxy` 속성)에서 다른 기본값이 지정되어 있지 않다면, 프록시 없이 실제 빈을 주입하려고 시도합니다. (싱글톤 빈이 세션/리퀘스트 빈을 주입받으면 문제가 발생할 수 있음)
- **`ScopedProxyMode.INTERFACES`:**
  - **JDK 동적 프록시**를 사용하여 프록시를 생성합니다.
  - **조건:** 프록시를 만들려는 빈 클래스(`UserPreferences`)가 **반드시 하나 이상의 인터페이스를 구현(implements)** 하고 있어야 합니다.
  - **동작:** 스프링은 해당 빈이 구현한 **모든 인터페이스를 똑같이 구현하는 프록시 클래스를 런타임에 생성**합니다. 주입받는 쪽에서는 인터페이스 타입으로 프록시를 받게 됩니다.
  - **장점:** CGLIB 라이브러리가 필요 없습니다. JDK 기본 기능입니다.
  - **단점:** 원본 클래스가 인터페이스를 구현해야만 사용할 수 있습니다.
- **`ScopedProxyMode.TARGET_CLASS`:**
  - **CGLIB 라이브러리**를 사용하여 프록시를 생성합니다.
  - **조건:** 특별한 조건은 없지만, 프록시 대상 클래스나 메소드가 `final`이면 안 됩니다.
  - **동작:** 스프링은 CGLIB을 사용하여 원본 빈 클래스(`UserPreferences`)를 **상속(extends)하는 자식 클래스 형태의 프록시 클래스를 런타임에 생성**합니다. 주입받는 쪽에서는 원본 클래스 타입 또는 그 부모/인터페이스 타입으로 프록시를 받을 수 있습니다.
  - **장점:** 원본 클래스가 인터페이스를 구현하지 않아도 프록시를 만들 수 있습니다. (더 유연함)
  - **단점:** CGLIB 라이브러리에 대한 의존성이 필요합니다. (스프링은 기본적으로 포함하고 있음) 클래스 상속 방식이므로 약간의 제약(final 금지)이 있을 수 있습니다.

**4. 사용 예시:**

```java
import org.springframework.context.annotation.Bean;
import org.springframework.context.annotation.Configuration;
import org.springframework.context.annotation.Scope;
import org.springframework.context.annotation.ScopedProxyMode;
import org.springframework.web.context.WebApplicationContext;

interface MySessionInterface { String getData(); }
class MySessionBean implements MySessionInterface { /* ... */ public String getData() { return "Session Data"; } }
class MySessionBeanWithoutInterface { /* ... */ public String getData() { return "Session Data No Interface"; } }

@Configuration
public class ScopeConfig {

    // MySessionBean은 MySessionInterface를 구현하므로 INTERFACES 방식 가능
    @Bean
    @Scope(value = WebApplicationContext.SCOPE_SESSION, proxyMode = ScopedProxyMode.INTERFACES)
    public MySessionInterface sessionBeanWithInterfaceProxy() {
        return new MySessionBean();
    }

    // MySessionBeanWithoutInterface는 인터페이스가 없으므로 TARGET_CLASS 방식 사용
    @Bean
    @Scope(value = WebApplicationContext.SCOPE_SESSION, proxyMode = ScopedProxyMode.TARGET_CLASS)
    public MySessionBeanWithoutInterface sessionBeanWithClassProxy() {
        return new MySessionBeanWithoutInterface();
    }

    // 프록시 모드를 지정하지 않으면 기본값(NO) 적용 (주의 필요)
    @Bean
    @Scope(value = WebApplicationContext.SCOPE_SESSION) // proxyMode = ScopedProxyMode.NO
    public MySessionBean sessionBeanNoProxy() {
        return new MySessionBean();
    }

    // 편리한 조합 어노테이션 (@SessionScope는 기본적으로 TARGET_CLASS 사용)
    @Bean
    @org.springframework.web.context.annotation.SessionScope // 기본 proxyMode=TARGET_CLASS
    public MySessionBean sessionBeanConvenienceAnnotation() {
        return new MySessionBean();
    }
}

// --- 주입 받는 쪽 ---
@Component
public class MySingletonService {

    @Autowired
    private MySessionInterface sessionBean1; // sessionBeanWithInterfaceProxy의 프록시 주입됨

    @Autowired
    private MySessionBeanWithoutInterface sessionBean2; // sessionBeanWithClassProxy의 프록시 주입됨

    // @Autowired
    // private MySessionBean sessionBean3; // sessionBeanNoProxy 주입 시도 시 문제 발생 가능성 높음

    @Autowired
    private MySessionBean sessionBean4; // sessionBeanConvenienceAnnotation의 프록시 주입됨

    public void useSessionData() {
        System.out.println("Data 1: " + sessionBean1.getData()); // 프록시 통해 실제 세션 빈 호출
        System.out.println("Data 2: " + sessionBean2.getData()); // 프록시 통해 실제 세션 빈 호출
        System.out.println("Data 4: " + sessionBean4.getData()); // 프록시 통해 실제 세션 빈 호출
    }
}

```

**결론:**

`@Scope` 어노테이션의 `proxyMode` 속성은 싱글톤이 아닌 스코프의 빈(주로 request, session)을 싱글톤 빈에 주입할 때 발생하는 문제를 해결하기 위해 **스코프 프록시를 생성하는 방식을 지정**합니다. `INTERFACES`는 JDK 동적 프록시(인터페이스 필요)를 사용하고, `TARGET_CLASS`는 CGLIB(클래스 상속, 인터페이스 불필요)를 사용합니다. 어떤 방식을 사용할지는 대상 빈이 인터페이스를 구현했는지 여부와 프로젝트의 요구사항에 따라 선택할 수 있습니다. 스프링의 `@SessionScope`, `@RequestScope` 같은 편리한 어노테이션들은 일반적으로 `TARGET_CLASS`를 기본값으로 사용합니다.
