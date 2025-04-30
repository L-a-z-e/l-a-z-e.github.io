---
title: Spring Framework Bean
description: 
author: laze
date: 2025-05-01 00:00:01 +0900
categories: [Dev, SpringBoot]
tags: [SpringBoot]
---
# Bean Overview

**Bean 개요**

스프링 IoC 컨테이너는 하나 이상의 빈(beans)을 관리합니다. 이 빈들은 컨테이너에 제공하는 설정 메타데이터(예: XML `<bean/>` 정의 형식)를 사용하여 생성됩니다.

컨테이너 자체 내부에서는 이러한 빈 정의들이 `BeanDefinition` 객체로 표현되며, 이 객체는 (다른 정보들 중에서) 다음 메타데이터를 포함합니다:

- **패키지 정규화된 클래스 이름:** 일반적으로 정의되는 빈의 실제 구현 클래스입니다.
- **빈 행동 설정 요소:** 빈이 컨테이너 내에서 어떻게 행동해야 하는지를 명시합니다 (스코프(scope), 생명주기 콜백(lifecycle callbacks) 등).
- **빈이 작업을 수행하는 데 필요한 다른 빈들에 대한 참조:** 이러한 참조는 협력자(collaborators) 또는 의존성(dependencies)이라고도 합니다.
- **새로 생성된 객체에 설정할 다른 구성 설정:** 예를 들어, 커넥션 풀을 관리하는 빈에서 사용할 풀의 크기 제한이나 연결 수.

이 메타데이터는 각 빈 정의를 구성하는 속성들의 집합으로 변환됩니다. 다음 표는 이러한 속성들을 설명합니다:

빈 정의

| 속성 | 설명 |
| --- | --- |
| Class | 빈 인스턴스화 |
| Name | 빈 이름 지정 |
| Scope | 빈 스코프 |
| Constructor arguments | 의존성 주입 |
| Properties | 의존성 주입 |
| Autowiring mode | 협력자 자동 와이어링 |
| Lazy initialization mode | 지연 초기화된 빈 |
| Initialization method | 초기화 콜백 |
| Destruction method | 소멸 콜백 |

특정 빈을 생성하는 방법에 대한 정보를 포함하는 빈 정의 외에도,

`ApplicationContext` 구현체들은 컨테이너 외부에서 (사용자에 의해) 생성된 기존 객체의 등록도 허용합니다.

이는 `ApplicationContext`의 BeanFactory에 `getAutowireCapableBeanFactory()` 메소드를 통해 접근하여 수행되며, 이 메소드는 `DefaultListableBeanFactory` 구현체를 반환합니다.

`DefaultListableBeanFactory`는 `registerSingleton(..)` 및 `registerBeanDefinition(..)` 메소드를 통해 이 등록을 지원합니다.

그러나 일반적인 애플리케이션은 오직 정규 빈 정의 메타데이터를 통해 정의된 빈들로만 작동합니다.

빈 메타데이터와 수동으로 제공된 싱글톤 인스턴스는 가능한 한 빨리 등록되어야 하며, 이는 컨테이너가 자동 와이어링(autowiring) 및 다른 내부 검사(introspection) 단계 동안 그것들에 대해 적절히 추론(reason)할 수 있도록 하기 위함입니다.

기존 메타데이터와 기존 싱글톤 인스턴스를 오버라이딩하는 것은 어느 정도 지원되지만, 런타임에 새로운 빈을 등록하는 것(팩토리에 대한 실시간 접근과 동시에)은 공식적으로 지원되지 않으며

동시 접근 예외, 빈 컨테이너의 불일치 상태 또는 둘 다를 초래할 수 있습니다.

**빈 오버라이딩 (Bean Overriding)**

빈 오버라이딩은 이미 할당된 식별자를 사용하여 빈이 등록될 때 발생합니다.

빈 오버라이딩은 가능하지만, 설정을 읽기 어렵게 만듭니다.

*빈 오버라이딩은 향후 릴리스에서 폐기(deprecated)될 예정입니다.*
빈 오버라이딩을 완전히 비활성화하려면, `ApplicationContext`가 리프레시(refresh)되기 전에 `allowBeanDefinitionOverriding` 플래그를 `false`로 설정할 수 있습니다.

이러한 설정에서는 빈 오버라이딩이 사용되면 예외가 발생합니다.

기본적으로 컨테이너는 빈을 오버라이드하려는 모든 시도를 `INFO` 레벨로 로깅하여 설정을 적절히 조정할 수 있도록 합니다.

권장되지는 않지만, `allowBeanDefinitionOverriding` 플래그를 `true`로 설정하여 해당 로그를 무시할 수 있습니다.

*자바 설정(Java Configuration)*
자바 설정을 사용하는 경우, 해당 `@Bean` 메소드는 반환 타입이 해당 빈 클래스와 일치하는 한, 동일한 컴포넌트 이름을 가진 스캔된 빈 클래스를 항상 자동으로(silently) 오버라이드합니다.

이는 단순히 컨테이너가 빈 클래스의 미리 선언된 생성자보다 `@Bean` 팩토리 메소드를 우선적으로 호출한다는 것을 의미합니다.

**빈 이름 지정 (Naming Beans)**

모든 빈은 하나 이상의 식별자를 가집니다.

이 식별자들은 빈을 호스팅하는 컨테이너 내에서 고유해야 합니다.

빈은 보통 하나의 식별자만 가집니다.

그러나 하나 이상이 필요한 경우, 추가적인 것들은 별칭(aliases)으로 간주될 수 있습니다.

XML 기반 설정 메타데이터에서는 `id` 속성, `name` 속성 또는 둘 다를 사용하여 빈 식별자를 지정합니다.

`id` 속성을 사용하면 정확히 하나의 id를 지정할 수 있습니다.

관례적으로 이러한 이름들은 영숫자('myBean', 'someService' 등)이지만, 특수 문자도 포함할 수 있습니다.

빈에 다른 별칭을 도입하려면 `name` 속성에 쉼표(,), 세미콜론(;), 또는 공백으로 구분하여 지정할 수도 있습니다.

`id` 속성은 `xsd:string` 타입으로 정의되지만, 빈 id의 고유성은 XML 파서가 아닌 컨테이너에 의해 강제됩니다.

빈에 이름이나 id를 제공할 필요는 없습니다.

이름이나 id를 명시적으로 제공하지 않으면, 컨테이너는 해당 빈에 대한 고유한 이름을 생성합니다.

그러나 `ref` 요소를 사용하거나 서비스 로케이터(Service Locator) 스타일 조회를 통해 이름으로 해당 빈을 참조하려면 이름을 제공해야 합니다.

이름을 제공하지 않는 동기는 내부 빈(inner beans) 사용 및 협력자 자동 와이어링(autowiring collaborators)과 관련이 있습니다.

*빈 이름 지정 관례 (Bean Naming Conventions)*
관례는 빈 이름을 지정할 때 인스턴스 필드 이름에 대한 표준 자바 관례를 사용하는 것입니다.

즉, 빈 이름은 소문자로 시작하고 거기서부터 카멜 케이스(camel-cased)를 사용합니다.

이러한 이름의 예로는 `accountManager`, `accountService`, `userDao`, `loginController` 등이 있습니다.

빈 이름을 일관되게 지정하면 설정을 더 쉽게 읽고 이해할 수 있습니다.

또한, 스프링 AOP를 사용하는 경우 이름으로 관련된 빈 집합에 어드바이스(advice)를 적용할 때 많은 도움이 됩니다.

*클래스패스에서 컴포넌트 스캔(component scanning)을 사용하면, 스프링은 이름 없는 컴포넌트에 대해 앞서 설명한 규칙에 따라 빈 이름을 생성합니다:*

*기본적으로 단순 클래스 이름을 가져와 첫 글자를 소문자로 바꿉니다.*

*그러나 (드문) 특수한 경우로, 두 글자 이상이고 첫 글자와 두 번째 글자가 모두 대문자일 때는 원래의 대소문자가 유지됩니다.*

*이는 `java.beans.Introspector.decapitalize` (스프링이 여기서 사용함)에서 정의된 것과 동일한 규칙입니다.*

**빈 정의 외부에서 빈 별칭 지정 (Aliasing a Bean outside the Bean Definition)**

빈 정의 자체에서는 `id` 속성으로 지정된 최대 하나의 이름과 `name` 속성의 다른 이름들을 조합하여 빈에 대해 둘 이상의 이름을 제공할 수 있습니다.

이 이름들은 동일한 빈에 대한 동등한 별칭이 될 수 있으며, 애플리케이션의 각 컴포넌트가 해당 컴포넌트 자체에 특정한 빈 이름을 사용하여 공통 의존성을 참조하게 하는 것과 같은 일부 상황에서 유용합니다.

그러나 빈이 실제로 정의된 곳에서 모든 별칭을 지정하는 것이 항상 적절하지는 않습니다.

때로는 다른 곳에 정의된 빈에 대한 별칭을 도입하는 것이 바람직합니다.

이는 일반적으로 구성이 각 하위 시스템 간에 분할되고 각 하위 시스템이 자체 객체 정의 세트를 갖는 대규모 시스템에서 흔히 발생합니다.

XML 기반 설정 메타데이터에서는 `<alias/>` 요소를 사용하여 이를 수행할 수 있습니다. 다음 예제는 이를 수행하는 방법을 보여줍니다:

```xml
<alias name="fromName" alias="toName"/>
```

이 경우, (동일한 컨테이너 내의) `fromName`이라는 이름의 빈은 이 별칭 정의를 사용한 후 `toName`으로도 참조될 수 있습니다.

예를 들어, 하위 시스템 A의 설정 메타데이터는 `subsystemA-dataSource`라는 이름으로 DataSource를 참조할 수 있습니다. 하위 시스템 B의 설정 메타데이터는 `subsystemB-dataSource`라는 이름으로 DataSource를 참조할 수 있습니다. 이 두 하위 시스템을 모두 사용하는 주 애플리케이션을 구성할 때, 주 애플리케이션은 `myApp-dataSource`라는 이름으로 DataSource를 참조합니다. 세 이름 모두 동일한 객체를 참조하게 하려면 설정 메타데이터에 다음 별칭 정의를 추가할 수 있습니다:

```xml
<alias name="myApp-dataSource" alias="subsystemA-dataSource"/>
<alias name="myApp-dataSource" alias="subsystemB-dataSource"/>
```

이제 각 컴포넌트와 주 애플리케이션은 다른 정의와 충돌하지 않음이 보장되는 고유한 이름(효과적으로 네임스페이스 생성)을 통해 `dataSource`를 참조하면서도 동일한 빈을 참조합니다.

*자바 설정(Java-configuration)*
자바 설정을 사용하는 경우, `@Bean` 어노테이션을 사용하여 별칭을 제공할 수 있습니다.

**빈 인스턴스화 (Instantiating Beans)**

빈 정의는 본질적으로 하나 이상의 객체를 생성하기 위한 레시피입니다.

컨테이너는 요청받을 때 이름 붙여진 빈에 대한 레시피를 보고 해당 빈 정의에 캡슐화된 설정 메타데이터를 사용하여 실제 객체를 생성(또는 획득)합니다.

XML 기반 설정 메타데이터를 사용하는 경우, 인스턴스화될 객체의 타입(또는 클래스)을 `<bean/>` 요소의 `class` 속성에 지정합니다.

이 `class` 속성(내부적으로는 `BeanDefinition` 인스턴스의 `Class` 속성임)은 일반적으로 필수입니다.

`Class` 속성은 다음 두 가지 방법 중 하나로 사용할 수 있습니다:

- **일반적으로, 컨테이너 자체가 생성자를 리플렉션(reflectively) 방식으로 호출하여 빈을 직접 생성하는 경우 생성할 빈 클래스를 지정합니다.** 이는 `new` 연산자를 사용하는 자바 코드와 다소 유사합니다.
- **덜 일반적인 경우로, 컨테이너가 클래스의 정적(static) 팩토리 메소드를 호출하여 빈을 생성하는 경우 객체를 생성하기 위해 호출될 정적 팩토리 메소드를 포함하는 실제 클래스를 지정합니다.**
  정적 팩토리 메소드 호출에서 반환되는 객체 타입은 동일한 클래스일 수도 있고 전혀 다른 클래스일 수도 있습니다.

*중첩 클래스 이름 (Nested class names)*
중첩 클래스에 대한 빈 정의를 설정하려면 중첩 클래스의 바이너리 이름이나 소스 이름 중 하나를 사용할 수 있습니다.

예를 들어, `com.example` 패키지에 `SomeThing`이라는 클래스가 있고 이 `SomeThing` 클래스에 `OtherThing`이라는 정적 중첩 클래스가 있는 경우, 달러 기호($) 또는 점(.)으로 구분할 수 있습니다.

따라서 빈 정의의 `class` 속성 값은 `com.example.SomeThing$OtherThing` 또는 `com.example.SomeThing.OtherThing`이 됩니다.

**생성자를 사용한 인스턴스화 (Instantiation with a Constructor)**

생성자 방식으로 빈을 생성할 때, 모든 일반 클래스는 스프링에서 사용 가능하고 호환됩니다.

즉, 개발 중인 클래스가 특정 인터페이스를 구현하거나 특정 방식으로 코딩될 필요가 없습니다.

단순히 빈 클래스를 지정하는 것으로 충분합니다.

그러나 해당 특정 빈에 어떤 유형의 IoC를 사용하는지에 따라 기본(빈) 생성자가 필요할 수 있습니다.

스프링 IoC 컨테이너는 관리하려는 거의 모든 클래스를 관리할 수 있습니다.

실제 JavaBeans 관리에만 국한되지 않습니다.

대부분의 스프링 사용자는 컨테이너의 속성을 모델링한 적절한 setter 및 getter와 기본(인수 없는) 생성자만 있는 실제 JavaBeans를 선호합니다.

컨테이너에 non-bean 스타일 클래스를 가질 수도 있습니다.

예를 들어, JavaBean 명세를 전혀 따르지 않는 레거시 커넥션 풀을 사용해야 하는 경우, 스프링은 그것도 관리할 수 있습니다.

XML 기반 설정 메타데이터를 사용하여 다음과 같이 빈 클래스를 지정할 수 있습니다:

```xml
<bean id="exampleBean" class="examples.ExampleBean"/>

<bean name="anotherExample" class="examples.ExampleBeanTwo"/>
```

생성자에 인수(필요한 경우)를 제공하고 객체가 생성된 후 객체 인스턴스 속성을 설정하는 메커니즘에 대한 자세한 내용은 의존성 주입(Injecting Dependencies)을 학습하면 알 수 있습니다.

*생성자 인수의 경우, 컨테이너는 여러 오버로드된 생성자 중에서 해당하는 생성자를 선택할 수 있습니다. 그렇지만 모호함을 피하기 위해 생성자 시그니처를 가능한 한 간단하게 유지하는 것이 좋습니다.*

**정적 팩토리 메소드를 사용한 인스턴스화 (Instantiation with a Static Factory Method)**

정적(static) 팩토리 메소드로 생성하는 빈을 정의할 때는, `class` 속성을 사용하여 정적 팩토리 메소드를 포함하는 클래스를 지정하고 `factory-method`라는 이름의 속성을 사용하여 팩토리 메소드 자체의 이름을 지정합니다.

이 메소드를 (나중에 설명될 선택적 인수와 함께) 호출하여 살아있는(live) 객체를 반환할 수 있어야 하며, 이 객체는 이후 생성자를 통해 생성된 것처럼 취급됩니다.

이러한 빈 정의의 한 가지 용도는 레거시 코드에서 정적 팩토리를 호출하는 것입니다.

다음 빈 정의는 팩토리 메소드를 호출하여 빈이 생성될 것임을 지정합니다.

이 정의는 반환된 객체의 타입(클래스)을 지정하는 것이 아니라, 팩토리 메소드를 포함하는 클래스를 지정합니다.

이 예제에서 `createInstance()` 메소드는 정적 메소드여야 합니다. 다음 예제는 팩토리 메소드를 지정하는 방법을 보여줍니다:

```xml
<bean id="clientService"
	class="examples.ClientService"
	factory-method="createInstance"/>

```

다음 예제는 앞의 빈 정의와 함께 작동할 클래스를 보여줍니다:

```java
// Java
public class ClientService {
	private static ClientService clientService = new ClientService();
	private ClientService() {}

	public static ClientService createInstance() {
		return clientService;
	}
}
```

```kotlin
// Kotlin
class ClientService private constructor() {
    companion object {
        private val clientService = ClientService()

        @JvmStatic // Ensure the method is static for Java reflection
        fun createInstance(): ClientService {
            return clientService
        }
    }
}
```

팩토리 메소드에 (선택적) 인수를 제공하고 객체가 팩토리에서 반환된 후 객체 인스턴스 속성을 설정하는 메커니즘에 대한 자세한 내용은 상세 의존성 및 설정(Dependencies and Configuration in Detail)에서 학습할 수 있습니다.

*팩토리 메소드 인수의 경우, 컨테이너는 동일한 이름의 여러 오버로드된 메소드 중에서 해당하는 메소드를 선택할 수 있습니다. 그렇지만 모호함을 피하기 위해 팩토리 메소드 시그니처를 가능한 한 간단하게 유지하는 것이 좋습니다.*

*팩토리 메소드 오버로딩의 일반적인 문제 사례는 Mockito의 `mock` 메소드에 대한 많은 오버로드입니다. 가능한 가장 구체적인 `mock` 변형을 선택하십시오:*

```xml
<bean id="clientService" class="org.mockito.Mockito" factory-method="mock">
	<constructor-arg type="java.lang.Class" value="examples.ClientService"/>
	<constructor-arg type="java.lang.String" value="clientService"/>
</bean>
```

**인스턴스 팩토리 메소드를 사용한 인스턴스화 (Instantiation by Using an Instance Factory Method)**

정적 팩토리 메소드를 통한 인스턴스화와 유사하게, 인스턴스 팩토리 메소드를 사용한 인스턴스화는 컨테이너의 기존 빈의 비정적(non-static) 메소드를 호출하여 새로운 빈을 생성합니다.

이 메커니즘을 사용하려면 `class` 속성을 비워두고, `factory-bean` 속성에 객체를 생성하기 위해 호출될 인스턴스 메소드를 포함하는 현재 (또는 부모 또는 조상) 컨테이너의 빈 이름을 지정합니다.

팩토리 메소드 자체의 이름은 `factory-method` 속성으로 설정합니다. 다음 예제는 이러한 빈을 설정하는 방법을 보여줍니다:

```xml
<!-- createClientServiceInstance() 메소드를 포함하는 팩토리 빈 -->
<bean id="serviceLocator" class="examples.DefaultServiceLocator">
	<!-- 이 로케이터 빈에 필요한 모든 의존성 주입 -->
</bean>

<!-- 팩토리 빈을 통해 생성될 빈 -->
<bean id="clientService"
	factory-bean="serviceLocator"
	factory-method="createClientServiceInstance"/>

```

다음 예제는 해당 클래스를 보여줍니다:

```java
// Java
public class DefaultServiceLocator {

	private static ClientService clientService = new ClientServiceImpl();

	public ClientService createClientServiceInstance() {
		return clientService;
	}
}

```

```kotlin
// Kotlin
class DefaultServiceLocator {
    companion object {
        private val clientService = ClientServiceImpl()
    }

    fun createClientServiceInstance(): ClientService {
        return clientService
    }
}

```

하나의 팩토리 클래스는 다음 예제와 같이 둘 이상의 팩토리 메소드를 가질 수도 있습니다:

```xml
<bean id="serviceLocator" class="examples.DefaultServiceLocator">
	<!-- 이 로케이터 빈에 필요한 모든 의존성 주입 -->
</bean>

<bean id="clientService"
	factory-bean="serviceLocator"
	factory-method="createClientServiceInstance"/>

<bean id="accountService"
	factory-bean="serviceLocator"
	factory-method="createAccountServiceInstance"/>

```

다음 예제는 해당 클래스를 보여줍니다:

```java
// Java
public class DefaultServiceLocator {

	private static ClientService clientService = new ClientServiceImpl();

	private static AccountService accountService = new AccountServiceImpl();

	public ClientService createClientServiceInstance() {
		return clientService;
	}

	public AccountService createAccountServiceInstance() {
		return accountService;
	}
}

```

```kotlin
// Kotlin
class DefaultServiceLocator {
    companion object {
        private val clientService = ClientServiceImpl()
        private val accountService = AccountServiceImpl()
    }

    fun createClientServiceInstance(): ClientService {
        return clientService
    }

    fun createAccountServiceInstance(): AccountService {
        return accountService
    }
}

```

이 접근 방식은 팩토리 빈 자체가 의존성 주입(DI)을 통해 관리되고 설정될 수 있음을 보여줍니다.

*스프링 문서에서 "팩토리 빈(factory bean)"은 스프링 컨테이너에 설정되고 인스턴스 또는 정적 팩토리 메소드를 통해 객체를 생성하는 빈을 나타냅니다.*

*반면, `FactoryBean`은 스프링 특정의 `FactoryBean` 구현 클래스를 나타냅니다.*

**빈의 런타임 타입 결정 (Determining a Bean’s Runtime Type)**

특정 빈의 런타임 타입을 결정하는 것은 간단하지 않습니다(non-trivial).

빈 메타데이터 정의에 지정된 클래스는 단지 초기 클래스 참조일 뿐이며, 선언된 팩토리 메소드와 결합되거나, 빈의 다른 런타임 타입을 초래할 수 있는 `FactoryBean` 클래스일 수 있거나,

인스턴스 수준 팩토리 메소드의 경우 전혀 설정되지 않을 수도 있습니다 (대신 지정된 `factory-bean` 이름을 통해 해결됨).

추가적으로, AOP 프록시는 빈 인스턴스를 인터페이스 기반 프록시로 감싸서 대상 빈의 실제 타입 노출을 제한할 수 있습니다.

특정 빈의 실제 런타임 타입을 알아내는 권장 방법은 지정된 빈 이름에 대해 `BeanFactory.getType`을 호출하는 것입니다.

이는 위의 모든 경우를 고려하고 동일한 빈 이름에 대해 `BeanFactory.getBean` 호출이 반환할 객체의 타입을 반환합니다.

---

Service Locator

```java
// 서비스 인터페이스
public interface Service {
    String getName();
    void execute();
}

// 서비스 구현체
public class ServiceA implements Service {
    public String getName() { return "ServiceA"; }
    public void execute() { System.out.println("Executing ServiceA"); }
}

// 서비스 로케이터
public class ServiceLocator {
    private static Map<String, Service> services = new HashMap<>();

    public static Service getService(String serviceName) {
        return services.get(serviceName);
    }

    public static void addService(Service service) {
        services.put(service.getName(), service);
    }
}

// 사용 예시
public class Client {
    public static void main(String[] args) {
        Service service = new ServiceA();
        ServiceLocator.addService(service);

        Service retrievedService = ServiceLocator.getService("ServiceA");
        retrievedService.execute();
    }
}
```

### 🔧 작동 원리

1. **서비스 인터페이스 정의**: 공통된 기능을 정의하는 인터페이스를 생성합니다.
2. **서비스 구현 등록**: 구체적인 서비스 구현체를 서비스 로케이터에 등록합니다.
3. **서비스 조회 및 사용**: 클라이언트는 서비스 로케이터를 통해 필요한 서비스를 조회하고 사용합니다.

---

### ✅ 장점

- **의존성 관리 용이**: 클라이언트는 서비스의 구체적인 구현에 대한 정보를 알 필요 없이 서비스를 사용할 수 있습니다.
- **유연성 향상**: 서비스 구현을 변경하더라도 클라이언트 코드를 수정할 필요가 없습니다.

---

### ⚠️ 단점

- **의존성 숨김**: 클라이언트의 의존성이 명시적으로 드러나지 않아 코드의 이해와 유지보수가 어려울 수 있습니다.
- **테스트 어려움**: 서비스 로케이터를 사용하면 테스트 시 모의 객체를 주입하기 어려워질 수 있습니다.

---

### 🔁 Service Locator vs. Dependency Injection
| 특징 | Service Locator | Dependency Injection |
| --- | --- | --- |
| 의존성 주입 방식 | 클라이언트가 서비스 로케이터를 통해 조회 | 외부에서 클라이언트에 의존성을 주입 |
| 의존성 명시성 | 낮음 | 높음 |
| 테스트 용이성 | 낮음 | 높음 |
| 구현 복잡성 | 낮음 | 높음 |
