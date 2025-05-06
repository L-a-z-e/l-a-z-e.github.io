---
title: Using AspectJ with Spring Applications
description: 
author: laze
date: 2025-05-06 00:00:05 +0900
categories: [Dev, SpringBoot]
tags: [SpringBoot]
---
**Using AspectJ with Spring Applications**

스프링 AOP만으로 제공되는 기능 이상의 요구 사항이 있을 경우, 스프링 AOP 대신 또는 추가적으로 AspectJ 컴파일러나 위버(weaver)를 어떻게 사용할 수 있는지 살펴봅니다.

스프링은 작은 AspectJ 애스펙트 라이브러리와 함께 제공되며, 이는 배포판에서 `spring-aspects.jar`로 독립적으로 사용할 수 있습니다.

그 안에 있는 애스펙트를 사용하려면 이것을 클래스패스에 추가해야 합니다.

**스프링으로 도메인 객체 의존성 주입에 AspectJ 사용하기 (Using AspectJ to Dependency Inject Domain Objects with Spring)**

스프링 컨테이너는 애플리케이션 컨텍스트에 정의된 빈을 인스턴스화하고 구성합니다.

적용될 구성을 포함하는 빈 정의의 이름을 지정하여, 미리 존재하는 객체를 구성하도록 빈 팩토리에 요청하는 것도 가능합니다.

`spring-aspects.jar`는 이 기능을 활용하여 모든 객체의 의존성 주입을 허용하는 어노테이션 기반 애스펙트를 포함합니다.

이 지원은 컨테이너의 제어 외부에서 생성된 객체에 사용하기 위한 것입니다.

도메인 객체는 종종 `new` 연산자를 사용하여 프로그래밍 방식으로 생성되거나 데이터베이스 쿼리의 결과로 ORM 도구에 의해 생성되기 때문에 이 범주에 속합니다.

`@Configurable` 어노테이션은 클래스가 스프링 주도 구성 대상임을 표시합니다.

가장 간단한 경우, 다음 예제와 같이 순수하게 마커(marker) 어노테이션으로 사용할 수 있습니다:

```java
// Java
package com.xyz.domain;

import org.springframework.beans.factory.annotation.Configurable;

@Configurable // Mark class for Spring configuration
public class Account {
	// ...
}
```

```kotlin
// Kotlin
package com.xyz.domain

import org.springframework.beans.factory.annotation.Configurable

@Configurable // Mark class for Spring configuration
class Account {
    // ...
}
```

이러한 방식으로 마커 인터페이스로 사용될 때, 스프링은 어노테이션된 타입(이 경우 `Account`)의 새 인스턴스를 정규화된 타입 이름(`com.xyz.domain.Account`)과 동일한 이름을 가진 빈 정의(일반적으로 프로토타입 스코프)를 사용하여 구성합니다. XML을 통해 정의된 빈의 기본 이름은 해당 타입의 정규화된 이름이므로, 프로토타입 정의를 선언하는 편리한 방법은 다음 예제와 같이 `id` 속성을 생략하는 것입니다:

```xml
<bean class="com.xyz.domain.Account" scope="prototype"> <!-- No id needed if it matches class name -->
	<property name="fundsTransferService" ref="fundsTransferService"/>
</bean>
```

사용할 프로토타입 빈 정의의 이름을 명시적으로 지정하려면, 다음 예제와 같이 어노테이션에서 직접 수행할 수 있습니다:

```java
// Java
package com.xyz.domain;

import org.springframework.beans.factory.annotation.Configurable;

@Configurable("account") // Specify the bean name to use for configuration
public class Account {
	// ...
}
```

```kotlin
// Kotlin
package com.xyz.domain

import org.springframework.beans.factory.annotation.Configurable

@Configurable("account") // Specify the bean name to use for configuration
class Account {
    // ...
}
```

이제 스프링은 `account`라는 이름의 빈 정의를 찾고 이를 새 `Account` 인스턴스를 구성하는 정의로 사용합니다.

전용 빈 정의를 전혀 지정할 필요가 없도록 자동 와이어링(autowiring)을 사용할 수도 있습니다.

스프링이 자동 와이어링을 적용하도록 하려면 `@Configurable` 어노테이션의 `autowire` 속성을 사용합니다.

타입별 자동 와이어링 또는 이름별 자동 와이어링을 위해 각각 `@Configurable(autowire=Autowire.BY_TYPE)` 또는 `@Configurable(autowire=Autowire.BY_NAME)`을 지정할 수 있습니다.

대안으로, 필드 또는 메소드 레벨에서 `@Autowired` 또는 `@Inject`를 통해 `@Configurable` 빈에 대한 명시적, 어노테이션 기반 의존성 주입을 지정하는 것이 바람직합니다.

마지막으로, `dependencyCheck` 속성을 사용하여 새로 생성되고 구성된 객체의 객체 참조에 대한 스프링 의존성 검사를 활성화할 수 있습니다 (예: `@Configurable(autowire=Autowire.BY_NAME,dependencyCheck=true)`).

이 속성이 `true`로 설정되면, 스프링은 구성 후 모든 속성(기본 타입이나 컬렉션이 아님)이 설정되었는지 검증합니다.

*어노테이션 자체만 사용하는 것은 아무런 효과가 없다는 점에 유의하십시오.*

*어노테이션의 존재에 따라 작동하는 것은 `spring-aspects.jar`의 `AnnotationBeanConfigurerAspect`입니다.*

*본질적으로, 애스펙트는 "@Configurable로 어노테이션된 타입의 새 객체 초기화에서 반환된 후, 어노테이션의 속성에 따라 스프링을 사용하여 새로 생성된 객체를 구성하라"고 말합니다.*

*이 컨텍스트에서 "초기화"는 새로 인스턴스화된 객체(예: `new` 연산자로 인스턴스화된 객체)뿐만 아니라 역직렬화(deserialization) 중인 `Serializable` 객체(예: `readResolve()`를 통해)도 나타냅니다.*

*위 단락의 핵심 구절 중 하나는 "본질적으로"입니다. 대부분의 경우, "새 객체 초기화에서 반환된 후"의 정확한 의미론은 괜찮습니다.*

*이 컨텍스트에서 "초기화 후"는 객체가 생성된 후 의존성이 주입됨을 의미합니다. 이는 즉, 의존성이 클래스의 생성자 본문에서 사용할 수 없다는 것을 의미합니다.*

*생성자 본문이 실행되기 전에 의존성이 주입되어 생성자 본문에서 사용할 수 있기를 원한다면, 다음과 같이 `@Configurable` 선언에 이를 정의해야 합니다:*

```java
// Java
@Configurable(preConstruction = true) // Inject dependencies before constructor execution
```

```kotlin
// Kotlin
@Configurable(preConstruction = true) // Inject dependencies before constructor execution
```

이것이 작동하려면 어노테이션된 타입이 AspectJ 위버(weaver)로 위빙되어야 합니다.

이를 위해 빌드 타임 Ant 또는 Maven 태스크를 사용하거나 로드 타임 위빙을 사용할 수 있습니다.

`AnnotationBeanConfigurerAspect` 자체는 스프링에 의해 구성되어야 합니다 (새 객체를 구성하는 데 사용될 빈 팩토리에 대한 참조를 얻기 위해). 관련 구성을 다음과 같이 정의할 수 있습니다:

**Java**

```java
import org.springframework.context.annotation.Configuration;
import org.springframework.context.annotation.EnableLoadTimeWeaving;
import org.springframework.context.annotation.aspectj.EnableSpringConfigured; // Import for @Configurable

@Configuration
@EnableSpringConfigured // Enable support for @Configurable aspect
@EnableLoadTimeWeaving // Often needed in conjunction with @Configurable if LTW is used
public class ApplicationConfiguration {
    // ... other bean definitions
}
```

**Kotlin**

```kotlin
import org.springframework.context.annotation.Configuration
import org.springframework.context.annotation.EnableLoadTimeWeaving
import org.springframework.context.annotation.aspectj.EnableSpringConfigured // Import for @Configurable

@Configuration
@EnableSpringConfigured // Enable support for @Configurable aspect
@EnableLoadTimeWeaving // Often needed in conjunction with @Configurable if LTW is used
class ApplicationConfiguration {
    // ... other bean definitions
}
```

**Xml**

```xml
<?xml version="1.0" encoding="UTF-8"?>
<beans xmlns="<http://www.springframework.org/schema/beans>"
    xmlns:xsi="<http://www.w3.org/2001/XMLSchema-instance>"
    xmlns:context="<http://www.springframework.org/schema/context>"
    xsi:schemaLocation="
        <http://www.springframework.org/schema/beans> <https://www.springframework.org/schema/beans/spring-beans.xsd>
        <http://www.springframework.org/schema/context> <https://www.springframework.org/schema/context/spring-context.xsd>">

    <!-- Enables the AnnotationBeanConfigurerAspect -->
    <context:spring-configured/>

    <!-- Enable LTW if needed for @Configurable weaving -->
    <context:load-time-weaver aspectj-weaving="on"/>

</beans>
```

애스펙트가 구성되기 전에 생성된 `@Configurable` 객체의 인스턴스는 디버그 로그에 메시지가 발행되고 객체 구성이 이루어지지 않는 결과를 낳습니다.

예로는 스프링에 의해 초기화될 때 도메인 객체를 생성하는 스프링 구성의 빈이 있을 수 있습니다.

이 경우, `depends-on` 빈 속성을 사용하여 빈이 구성 애스펙트에 의존함을 수동으로 지정할 수 있습니다. 다음 예제는 `depends-on` 속성 사용 방법을 보여줍니다:

```xml
<bean id="myService"
		class="com.xyz.service.MyService"
		depends-on="org.springframework.context.config.internalBeanConfigurerAspect"> <!-- Correct aspect bean name -->

	<!-- ... -->

</bean>
```

*런타임 시 해당 의미론에 의존하려는 의도가 아니라면 빈 구성자 애스펙트를 통해 `@Configurable` 처리를 활성화하지 마십시오.*

*특히, 컨테이너에 일반 스프링 빈으로 등록된 빈 클래스에 `@Configurable`을 사용하지 않도록 하십시오.*

*그렇게 하면 컨테이너를 통해 한 번, 애스펙트를 통해 한 번, 이중 초기화가 발생합니다.*

*@Configurable 객체 단위 테스트하기 (Unit Testing @Configurable Objects)*`@Configurable` 지원의 목표 중 하나는 하드코딩된 룩업과 관련된 어려움 없이 도메인 객체의 독립적인 단위 테스트를 가능하게 하는 것입니다.

`@Configurable` 타입이 AspectJ에 의해 위빙되지 않았다면, 어노테이션은 단위 테스트 중에 아무런 효과가 없습니다.

테스트 대상 객체에 모의(mock) 또는 스텁(stub) 속성 참조를 설정하고 평소처럼 진행할 수 있습니다.

`@Configurable` 타입이 AspectJ에 의해 위빙되었다면, 평소처럼 컨테이너 외부에서 단위 테스트를 할 수 있지만, `@Configurable` 객체를 생성할 때마다 스프링에 의해 구성되지 않았음을 나타내는 경고 메시지가 표시됩니다.

*여러 애플리케이션 컨텍스트 작업하기 (Working with Multiple Application Contexts)*`@Configurable` 지원을 구현하는 데 사용되는 `AnnotationBeanConfigurerAspect`는 AspectJ 싱글톤 애스펙트입니다.

싱글톤 애스펙트의 스코프는 정적 멤버의 스코프와 동일합니다: 타입을 정의하는 클래스로더당 하나의 애스펙트 인스턴스가 있습니다.

이는 즉, 동일한 클래스로더 계층 구조 내에 여러 애플리케이션 컨텍스트를 정의하는 경우, `@EnableSpringConfigured` 빈을 어디에 정의하고 클래스패스에 `spring-aspects.jar`를 어디에 배치할지 고려해야 함을 의미합니다.

공통 비즈니스 서비스, 해당 서비스를 지원하는 데 필요한 모든 것,

그리고 각 서블릿에 대한 자식 애플리케이션 컨텍스트(해당 서블릿에 특정된 정의 포함)를 정의하는 공유 부모 애플리케이션 컨텍스트를 가진 일반적인 스프링 웹 애플리케이션 구성을 고려하십시오.

이러한 모든 컨텍스트는 동일한 클래스로더 계층 구조 내에 공존하므로 `AnnotationBeanConfigurerAspect`는 그중 하나에 대한 참조만 보유할 수 있습니다.

이 경우, 공유 (부모) 애플리케이션 컨텍스트에 `@EnableSpringConfigured` 빈을 정의하는 것이 좋습니다.

이는 도메인 객체에 주입하고 싶을 가능성이 높은 서비스를 정의합니다.

결과적으로, `@Configurable` 메커니즘을 사용하여 자식 (서블릿 특정) 컨텍스트에 정의된 빈에 대한 참조로 도메인 객체를 구성할 수 없습니다.

동일한 컨테이너 내에 여러 웹 애플리케이션을 배포할 때, 각 웹 애플리케이션이 자체 클래스로더를 사용하여 `spring-aspects.jar`의 타입을 로드하도록 하십시오 (예: `spring-aspects.jar`를 `WEB-INF/lib`에 배치). 만약 `spring-aspects.jar`가 컨테이너 전체 클래스패스에만 추가되고 (따라서 공유 부모 클래스로더에 의해 로드됨), 모든 웹 애플리케이션은 동일한 애스펙트 인스턴스를 공유하게 됩니다.

**AspectJ를 위한 기타 스프링 애스펙트 (Other Spring aspects for AspectJ)**

`@Configurable` 애스펙트 외에도, `spring-aspects.jar`는 `@Transactional` 어노테이션으로 어노테이션된 타입 및 메소드에 대한 스프링의 트랜잭션 관리를 구동하는 데 사용할 수 있는 AspectJ 애스펙트를 포함합니다.

이는 주로 스프링 컨테이너 외부에서 스프링 프레임워크의 트랜잭션 지원을 사용하려는 사용자를 위한 것입니다.

`@Transactional` 어노테이션을 해석하는 애스펙트는 `AnnotationTransactionAspect`입니다.

이 애스펙트를 사용할 때, 클래스가 구현하는 인터페이스(있는 경우)가 아니라 구현 클래스(또는 해당 클래스 내의 메소드 또는 둘 다)에 어노테이션을 달아야 합니다.

AspectJ는 인터페이스의 어노테이션이 상속되지 않는다는 자바의 규칙을 따릅니다.

클래스의 `@Transactional` 어노테이션은 클래스의 모든 public 작업 실행에 대한 기본 트랜잭션 의미론을 지정합니다.

클래스 내 메소드의 `@Transactional` 어노테이션은 (존재하는 경우) 클래스 어노테이션에 의해 주어진 기본 트랜잭션 의미론을 오버라이드합니다.

private 메소드를 포함하여 모든 가시성의 메소드에 어노테이션을 달 수 있습니다.

비-public 메소드에 직접 어노테이션을 다는 것이 해당 메소드 실행에 대한 트랜잭션 경계 설정(transaction demarcation)을 얻는 유일한 방법입니다.

*스프링 프레임워크 4.2부터 `spring-aspects`는 표준 `jakarta.transaction.Transactional` 어노테이션에 대해 정확히 동일한 기능을 제공하는 유사한 애스펙트를 제공합니다.*

스프링 구성 및 트랜잭션 관리 지원을 사용하고 싶지만 어노테이션을 사용하고 싶지 않거나 사용할 수 없는 AspectJ 프로그래머를 위해, `spring-aspects.jar`는 또한 자신만의 포인트컷 정의를 제공하기 위해 확장할 수 있는 추상 애스펙트를 포함합니다. 정규화된 클래스 이름과 일치하는 프로토타입 빈 정의를 사용하여 도메인 모델에 정의된 객체의 모든 인스턴스를 구성하는 애스펙트를 작성하는 방법을 보여줍니다:

```java
// Example extending AbstractBeanConfigurerAspect (Conceptual)
import org.aspectj.lang.annotation.Aspect;
import org.aspectj.lang.annotation.Pointcut;
import org.springframework.beans.factory.aspectj.AbstractBeanConfigurerAspect;
import org.springframework.beans.factory.wiring.ClassNameBeanWiringInfoResolver;

@Aspect
public aspect DomainObjectConfiguration extends AbstractBeanConfigurerAspect {

	public DomainObjectConfiguration() {
		// Use class name to find the prototype bean definition
		setBeanWiringInfoResolver(new ClassNameBeanWiringInfoResolver());
	}

	// Define pointcut for new object initialization in the domain model package
	@Override // Override the abstract pointcut
	@Pointcut("initialization(new(..)) && CommonPointcuts.inDomainModel() && this(beanInstance)")
	protected void beanCreation(Object beanInstance) {} // Match initialization of domain objects
}
// Assuming CommonPointcuts.inDomainModel() exists
```

**스프링 IoC를 사용하여 AspectJ 애스펙트 구성하기 (Configuring AspectJ Aspects by Using Spring IoC)**

스프링 애플리케이션과 함께 AspectJ 애스펙트를 사용할 때, 스프링으로 이러한 애스펙트를 구성할 수 있기를 원하고 기대하는 것이 자연스럽습니다.

AspectJ 런타임 자체는 애스펙트 생성을 책임지며, 스프링을 통해 AspectJ 생성 애스펙트를 구성하는 수단은 애스펙트가 사용하는 AspectJ 인스턴스화 모델( `per-xxx` 절)에 따라 달라집니다.

대부분의 AspectJ 애스펙트는 싱글톤 애스펙트입니다.

이러한 애스펙트 구성은 쉽습니다. 평소처럼 애스펙트 타입을 참조하는 빈 정의를 생성하고 `factory-method="aspectOf"` 빈 속성을 포함할 수 있습니다.

이는 스프링이 자체적으로 인스턴스를 생성하려고 시도하는 대신 AspectJ에 요청하여 애스펙트 인스턴스를 얻도록 보장합니다. 다음 예제는 `factory-method="aspectOf"` 속성 사용 방법을 보여줍니다:

```xml
<bean id="profiler" class="com.xyz.profiler.Profiler"
		factory-method="aspectOf"> <!-- ① Tell Spring to ask AspectJ for the instance -->

	<property name="profilingStrategy" ref="jamonProfilingStrategy"/>
</bean>
```

① `factory-method="aspectOf"` 속성을 주목하세요

비-싱글톤 애스펙트는 구성하기 더 어렵습니다.

그러나 프로토타입 빈 정의를 생성하고 `spring-aspects.jar`의 `@Configurable` 지원을 사용하여 AspectJ 런타임에 의해 생성된 후 애스펙트 인스턴스를 구성함으로써 가능합니다.

AspectJ로 위빙하려는 일부 `@AspectJ` 애스펙트(예: 도메인 모델 타입에 로드 타임 위빙 사용)와 스프링 AOP와 함께 사용하려는 다른 `@AspectJ` 애스펙트가 있고,

이러한 모든 애스펙트가 스프링에서 구성되는 경우, 구성에 정의된 `@AspectJ` 애스펙트 중 정확히 어떤 하위 집합이 자동 프록시에 사용되어야 하는지 스프링 AOP `@AspectJ` 자동 프록시 지원에 알려야 합니다.

`<aop:aspectj-autoproxy/>` 선언 내부에 하나 이상의 `<include/>` 요소를 사용하여 이를 수행할 수 있습니다.

각 `<include/>` 요소는 이름 패턴을 지정하며, 패턴 중 하나 이상과 일치하는 이름을 가진 빈만 스프링 AOP 자동 프록시 구성에 사용됩니다. 다음 예제는 `<include/>` 요소 사용 방법을 보여줍니다:

```xml
<aop:aspectj-autoproxy>
	<aop:include name="thisBean"/> <!-- Only include 'thisBean' for AOP proxying -->
	<aop:include name="thatBean"/> <!-- Also include 'thatBean' -->
</aop:aspectj-autoproxy>
```

`*<aop:aspectj-autoproxy/>` 요소의 이름에 속지 마십시오.*

*이를 사용하면 스프링 AOP 프록시가 생성됩니다.*

*여기서 `@AspectJ` 스타일의 애스펙트 선언이 사용되고 있지만, AspectJ 런타임은 관련되지 않습니다.*

**스프링 프레임워크에서의 AspectJ 로드 타임 위빙 (Load-time Weaving with AspectJ in the Spring Framework)**

로드 타임 위빙(LTW - Load-time weaving)은 AspectJ 애스펙트를 애플리케이션의 클래스 파일이 자바 가상 머신(JVM)에 로드될 때 위빙하는 프로세스를 나타냅니다.

스프링 프레임워크가 AspectJ LTW에 제공하는 가치는 위빙 프로세스에 대해 훨씬 더 세분화된 제어를 가능하게 하는 데 있습니다.

'순수한(Vanilla)' AspectJ LTW는 JVM 시작 시 VM 인수를 지정하여 켜는 자바 (5+) 에이전트를 사용하여 수행됩니다. 따라서 이는 JVM 전체 설정이며, 어떤 상황에서는 괜찮을 수 있지만 종종 너무 광범위합니다.

스프링 활성화 LTW를 사용하면 클래스로더별로 LTW를 켤 수 있으며, 이는 더 세분화되고 '단일 JVM-다중 애플리케이션' 환경(일반적인 애플리케이션 서버 환경에서 발견됨)에서 더 의미 있을 수 있습니다.

또한 특정 환경에서는 이 지원이 `-javaagent:path/to/aspectjweaver.jar` 또는 `-javaagent:path/to/spring-instrument.jar`를 추가하는 데 필요한 애플리케이션 서버의 시작 스크립트를 수정하지 않고도 로드 타임 위빙을 가능하게 합니다. 개발자는 일반적으로 시작 스크립트와 같은 배포 구성을 담당하는 관리자에게 의존하는 대신 로드 타임 위빙을 활성화하도록 애플리케이션 컨텍스트를 구성합니다.

*첫 번째 예제 (A First Example)*
시스템의 일부 성능 문제의 원인을 진단하는 임무를 맡은 애플리케이션 개발자라고 가정해 봅시다.

프로파일링 도구를 꺼내는 대신, 몇 가지 성능 메트릭을 신속하게 얻을 수 있는 간단한 프로파일링 애스펙트를 켤 것입니다.

그런 다음 즉시 해당 특정 영역에 더 세분화된 프로파일링 도구를 적용할 수 있습니다.

*여기서 제시된 예제는 XML 구성을 사용합니다.*

*자바 구성으로 `@AspectJ`를 구성하고 사용할 수도 있습니다. 특히, `<context:load-time-weaver/>`의 대안으로 `@EnableLoadTimeWeaving` 어노테이션을 사용할 수 있습니다 .*

다음 예제는 화려하지 않은 프로파일링 애스펙트를 보여줍니다. 이는 `@AspectJ` 스타일 애스펙트 선언을 사용하는 시간 기반 프로파일러입니다:

```java
// Java - Profiling Aspect
package com.xyz;

import org.aspectj.lang.ProceedingJoinPoint;
import org.aspectj.lang.annotation.Aspect;
import org.aspectj.lang.annotation.Around;
import org.aspectj.lang.annotation.Pointcut;
import org.springframework.util.StopWatch;
import org.springframework.core.annotation.Order; // Can be used for ordering aspects

@Aspect
public class ProfilingAspect {

	@Around("methodsToBeProfiled()") // Apply around advice to the pointcut
	public Object profile(ProceedingJoinPoint pjp) throws Throwable {
		StopWatch sw = new StopWatch(getClass().getSimpleName());
		try {
			sw.start(pjp.getSignature().getName());
			return pjp.proceed();
		} finally {
			sw.stop();
			System.out.println(sw.prettyPrint());
		}
	}

	// Define the pointcut for public methods in com.xyz packages
	@Pointcut("execution(public * com.xyz..*.*(..))")
	public void methodsToBeProfiled(){}
}
```

```kotlin
// Kotlin - Profiling Aspect
package com.xyz

import org.aspectj.lang.ProceedingJoinPoint
import org.aspectj.lang.annotation.Around
import org.aspectj.lang.annotation.Aspect
import org.aspectj.lang.annotation.Pointcut
import org.springframework.util.StopWatch

@Aspect
class ProfilingAspect {

    @Around("methodsToBeProfiled()") // Apply around advice to the pointcut
    @Throws(Throwable::class)
    fun profile(pjp: ProceedingJoinPoint): Any? {
        val sw = StopWatch(javaClass.simpleName)
        return try {
            sw.start(pjp.signature.name)
            pjp.proceed()
        } finally {
            sw.stop()
            println(sw.prettyPrint())
        }
    }

    // Define the pointcut for public methods in com.xyz packages
    @Pointcut("execution(public * com.xyz..*.*(..))")
    fun methodsToBeProfiled() {}
}
```

또한 클래스에 `ProfilingAspect`를 위빙하고 싶다는 것을 AspectJ 위버에게 알리기 위해 `META-INF/aop.xml` 파일을 생성해야 합니다.

이 파일 규칙, 즉 `META-INF/aop.xml`이라는 이름의 파일(들)이 자바 클래스패스에 존재하는 것은 표준 AspectJ입니다. 다음 예제는 `aop.xml` 파일을 보여줍니다:

```xml
<!DOCTYPE aspectj PUBLIC "-//AspectJ//DTD//EN" "<https://www.eclipse.org/aspectj/dtd/aspectj.dtd>">
<aspectj>

	<weaver>
		<!-- 애플리케이션 특정 패키지 및 하위 패키지의 클래스만 위빙 -->
		<include within="com.xyz..*"/>
	</weaver>

	<aspects>
		<!-- 이 애스펙트만 위빙 -->
		<aspect name="com.xyz.ProfilingAspect"/>
	</aspects>

</aspectj>
```

*AspectJ 덤프 파일 및 경고와 같은 부작용을 피하기 위해 특정 클래스(일반적으로 위 aop.xml 예제에서 보여준 것처럼 애플리케이션 패키지에 있는 것들)만 위빙하는 것이 좋습니다.*

*이는 또한 효율성 관점에서도 모범 사례입니다.*

이제 구성의 스프링 특정 부분으로 넘어갈 수 있습니다.

`LoadTimeWeaver`를 구성해야 합니다.

이 로드 타임 위버는 하나 이상의 `META-INF/aop.xml` 파일의 애스펙트 구성을 애플리케이션의 클래스에 위빙하는 책임이 있는 필수 구성 요소입니다.

좋은 점은 다음 예제에서 볼 수 있듯이 많은 구성이 필요하지 않다는 것입니다 (지정할 수 있는 몇 가지 추가 옵션이 있지만 이는 나중에 자세히 설명됨):

```xml
<?xml version="1.0" encoding="UTF-8"?>
<beans xmlns="<http://www.springframework.org/schema/beans>"
	xmlns:xsi="<http://www.w3.org/2001/XMLSchema-instance>"
	xmlns:context="<http://www.springframework.org/schema/context>"
	xsi:schemaLocation="
		<http://www.springframework.org/schema/beans>
		<https://www.springframework.org/schema/beans/spring-beans.xsd>
		<http://www.springframework.org/schema/context>
		<https://www.springframework.org/schema/context/spring-context.xsd>">

	<!-- 서비스 객체; 해당 메소드를 프로파일링할 것임 -->
	<bean id="entitlementCalculationService"
			class="com.xyz.StubEntitlementCalculationService"/> <!-- Assuming this class exists -->

	<!-- 로드 타임 위빙을 켭니다 -->
	<context:load-time-weaver/>

</beans>
```

이제 필요한 모든 아티팩트(애스펙트, `META-INF/aop.xml` 파일 및 스프링 구성)가 준비되었으므로, LTW가 작동하는 것을 보여주기 위해 `main(..)` 메소드가 있는 다음 드라이버 클래스를 생성할 수 있습니다:

```java
// Java
package com.xyz;

import org.springframework.context.ApplicationContext;
import org.springframework.context.support.ClassPathXmlApplicationContext;
// Assuming EntitlementCalculationService interface and Stub implementation exist

public class Main {

	public static void main(String[] args) {
		ApplicationContext ctx = new ClassPathXmlApplicationContext("beans.xml"); // Load Spring config

		EntitlementCalculationService service =
				ctx.getBean(EntitlementCalculationService.class);

		// 프로파일링 애스펙트가 이 메소드 실행 주위에 '위빙'됩니다
		service.calculateEntitlement(); // Call the method
	}
}
```

```kotlin
// Kotlin
package com.xyz

import org.springframework.context.support.ClassPathXmlApplicationContext
// Assuming EntitlementCalculationService interface and Stub implementation exist

fun main(args: Array<String>) {
    val ctx = ClassPathXmlApplicationContext("beans.xml") // Load Spring config

    val service = ctx.getBean(EntitlementCalculationService::class.java)

    // 프로파일링 애스펙트가 이 메소드 실행 주위에 '위빙'됩니다
    service.calculateEntitlement() // Call the method
}
```

스프링을 사용하여 클래스로더별로 선택적으로 LTW를 켤 수 있다고 말했으며 이는 사실입니다.

그러나 이 예제에서는 LTW를 켜기 위해 (스프링과 함께 제공되는) 자바 에이전트를 사용합니다.

앞서 보여준 `Main` 클래스를 실행하기 위해 다음 명령을 사용합니다:

`java -javaagent:C:/projects/xyz/lib/spring-instrument.jar com.xyz.Main`

- `javaagent`는 JVM에서 실행되는 프로그램을 계측(instrument)하기 위해 에이전트를 지정하고 활성화하는 플래그입니다. 스프링 프레임워크는 이러한 에이전트인 `InstrumentationSavingAgent`를 제공하며, 이는 앞의 예제에서 `javaagent` 인수의 값으로 제공된 `spring-instrument.jar`에 패키징되어 있습니다.

`Main` 프로그램 실행의 출력은 다음 예제와 유사합니다. (`calculateEntitlement()` 구현에 `Thread.sleep(..)` 문을 도입하여 프로파일러가 0밀리초 외의 것을 실제로 캡처하도록 했습니다 (01234 밀리초는 AOP에 의해 도입된 오버헤드가 아님). 다음 목록은 프로파일러를 실행했을 때 얻은 출력을 보여줍니다:

```
Calculating entitlement

StopWatch 'ProfilingAspect': running time (millis) = 1234
-----------------------------------------
ms     %     Task name
-----------------------------------------
01234  100%  calculateEntitlement

```

이 LTW는 완전한 AspectJ를 사용하여 수행되므로 스프링 빈 어드바이스에만 국한되지 않습니다. `Main` 프로그램의 다음 약간의 변형은 동일한 결과를 산출합니다:

```java
// Java
package com.xyz;

import org.springframework.context.support.ClassPathXmlApplicationContext;
// Assuming EntitlementCalculationService interface and Stub implementation exist

public class Main {

	public static void main(String[] args) {
		new ClassPathXmlApplicationContext("beans.xml"); // Bootstrap Spring container (loads LTW config)

		EntitlementCalculationService service =
				new StubEntitlementCalculationService(); // Create instance outside Spring

		// 프로파일링 애스펙트가 이 메소드 실행 주위에 '위빙'됩니다
		service.calculateEntitlement(); // Advice is still woven due to LTW agent
	}
}
```

```kotlin
// Kotlin
package com.xyz

import org.springframework.context.support.ClassPathXmlApplicationContext
// Assuming EntitlementCalculationService interface and Stub implementation exist

fun main(args: Array<String>) {
    ClassPathXmlApplicationContext("beans.xml") // Bootstrap Spring container (loads LTW config)

    val service = StubEntitlementCalculationService() // Create instance outside Spring

    // 프로파일링 애스펙트가 이 메소드 실행 주위에 '위빙'됩니다
    service.calculateEntitlement() // Advice is still woven due to LTW agent
}
```

앞의 프로그램에서 스프링 컨테이너를 부트스트랩한 다음 스프링 컨텍스트 외부에서 완전히 `StubEntitlementCalculationService`의 새 인스턴스를 생성하는 방식에 주목하십시오.

프로파일링 어드바이스는 여전히 위빙됩니다.

*이 예제에서 사용된 `ProfilingAspect`는 기본적일 수 있지만 매우 유용합니다.*

*개발자가 개발 중에 사용할 수 있고 UAT 또는 프로덕션에 배포되는 애플리케이션 빌드에서 쉽게 제외할 수 있는 개발 시간 애스펙트의 좋은 예입니다.*

*애스펙트 (Aspects)*
LTW에서 사용하는 애스펙트는 AspectJ 애스펙트여야 합니다.

AspectJ 언어 자체로 작성하거나 `@AspectJ` 스타일로 애스펙트를 작성할 수 있습니다.

그러면 애스펙트는 유효한 AspectJ 및 스프링 AOP 애스펙트 모두입니다. 또한 컴파일된 애스펙트 클래스는 클래스패스에서 사용 가능해야 합니다.

*META-INF/aop.xml*
AspectJ LTW 인프라는 자바 클래스패스에 있는 (직접 또는 더 일반적으로는 jar 파일 내) 하나 이상의 `META-INF/aop.xml` 파일을 사용하여 구성됩니다. 예를 들어:

```xml
<!DOCTYPE aspectj PUBLIC "-//AspectJ//DTD//EN" "<https://www.eclipse.org/aspectj/dtd/aspectj.dtd>">
<aspectj>

	<weaver>
		<!-- 애플리케이션 특정 패키지 및 하위 패키지의 클래스만 위빙 -->
		<include within="com.xyz..*"/>
	</weaver>

</aspectj>
```

*AspectJ 덤프 파일 및 경고와 같은 부작용을 피하기 위해 특정 클래스(일반적으로 위 aop.xml 예제에서 보여준 것처럼 애플리케이션 패키지에 있는 것들)만 위빙하는 것이 좋습니다.*

*이는 또한 효율성 관점에서도 모범 사례입니다.*

*필수 라이브러리 (JARS) (Required libraries (JARS))*
최소한 스프링 프레임워크의 AspectJ LTW 지원을 사용하려면 다음 라이브러리가 필요합니다:

- `spring-aop.jar`
- `aspectjweaver.jar`

계측(instrumentation)을 활성화하기 위해 스프링 제공 에이전트를 사용하는 경우 다음도 필요합니다:

- `spring-instrument.jar`

*스프링 구성 (Spring Configuration)*
스프링의 LTW 지원의 핵심 구성 요소는 `LoadTimeWeaver` 인터페이스(`org.springframework.instrument.classloading` 패키지)와 스프링 배포판과 함께 제공되는 수많은 구현체입니다.

`LoadTimeWeaver`는 런타임 시 `ClassLoader`에 하나 이상의 `java.lang.instrument.ClassFileTransformer`를 추가하는 책임이 있으며, 이는 모든 종류의 흥미로운 애플리케이션의 문을 엽니다.

그중 하나가 애스펙트의 LTW입니다.

특정 `ApplicationContext`에 대해 `LoadTimeWeaver`를 구성하는 것은 한 줄을 추가하는 것만큼 쉬울 수 있습니다.

(`ApplicationContext`를 스프링 컨테이너로 거의 확실하게 사용해야 한다는 점에 유의하십시오 - 일반적으로 LTW 지원은 `BeanFactoryPostProcessor`를 사용하기 때문에 `BeanFactory`로는 충분하지 않습니다.)

스프링 프레임워크의 LTW 지원을 활성화하려면 다음과 같이 `LoadTimeWeaver`를 구성해야 합니다:

**Java**

```java
import org.springframework.context.annotation.Configuration;
import org.springframework.context.annotation.EnableLoadTimeWeaving;

@Configuration
@EnableLoadTimeWeaving // Enable LTW support
public class ApplicationConfiguration {
    // ...
}
```

**Kotlin**

```kotlin
import org.springframework.context.annotation.Configuration
import org.springframework.context.annotation.EnableLoadTimeWeaving

@Configuration
@EnableLoadTimeWeaving // Enable LTW support
class ApplicationConfiguration {
    // ...
}
```

**Xml**

```xml
<?xml version="1.0" encoding="UTF-8"?>
<beans xmlns="<http://www.springframework.org/schema/beans>"
    xmlns:xsi="<http://www.w3.org/2001/XMLSchema-instance>"
    xmlns:context="<http://www.springframework.org/schema/context>"
    xsi:schemaLocation="
        <http://www.springframework.org/schema/beans> <https://www.springframework.org/schema/beans/spring-beans.xsd>
        <http://www.springframework.org/schema/context> <https://www.springframework.org/schema/context/spring-context.xsd>">

    <context:load-time-weaver/> <!-- Enable LTW support -->

</beans>
```

앞의 구성은 `LoadTimeWeaver` 및 `AspectJWeavingEnabler`와 같은 여러 LTW 특정 인프라 빈을 자동으로 정의하고 등록합니다. 기본 `LoadTimeWeaver`는 `DefaultContextLoadTimeWeaver` 클래스이며, 이는 자동으로 감지된 `LoadTimeWeaver`를 장식(decorate)하려고 시도합니다. "자동으로 감지된" `LoadTimeWeaver`의 정확한 타입은 런타임 환경에 따라 다릅니다. 다음 표는 다양한 `LoadTimeWeaver` 구현을 요약합니다:

표 1. `DefaultContextLoadTimeWeaver` LoadTimeWeavers

| 런타임 환경 | `LoadTimeWeaver` 구현 |
| --- | --- |
| Apache Tomcat에서 실행 중 | `TomcatLoadTimeWeaver` |
| GlassFish에서 실행 중 (EAR 배포로 제한됨) | `GlassFishLoadTimeWeaver` |
| Red Hat의 JBoss AS 또는 WildFly에서 실행 중 | `JBossLoadTimeWeaver` |
| 스프링 InstrumentationSavingAgent로 시작된 JVM (`java -javaagent:path/to/spring-instrument.jar`) | `InstrumentationLoadTimeWeaver` |
| 폴백(Fallback), 기본 `ClassLoader`가 공통 관례(즉, `addTransformer` 및 선택적으로 `getThrowawayClassLoader` 메소드)를 따를 것으로 예상 | `ReflectiveLoadTimeWeaver` |

표에는 `DefaultContextLoadTimeWeaver`를 사용할 때 자동 감지되는 `LoadTimeWeavers`만 나열되어 있다는 점에 유의하십시오.

사용할 `LoadTimeWeaver` 구현을 정확히 지정할 수 있습니다.

특정 `LoadTimeWeaver`를 구성하려면 `LoadTimeWeavingConfigurer` 인터페이스를 구현하고 `getLoadTimeWeaver()` 메소드를 오버라이드합니다 (또는 XML 등가물 사용).

다음 예제는 `ReflectiveLoadTimeWeaver`를 지정합니다:

**Java**

```java
import org.springframework.context.annotation.Configuration;
import org.springframework.context.annotation.EnableLoadTimeWeaving;
import org.springframework.instrument.classloading.LoadTimeWeaver;
import org.springframework.instrument.classloading.ReflectiveLoadTimeWeaver;
import org.springframework.context.LoadTimeWeavingConfigurer; // Import interface

@Configuration
@EnableLoadTimeWeaving
public class CustomWeaverConfiguration implements LoadTimeWeavingConfigurer { // Implement interface

	@Override // Override the method
	public LoadTimeWeaver getLoadTimeWeaver() {
		return new ReflectiveLoadTimeWeaver(); // Specify the weaver
	}
}
```

**Kotlin**

```kotlin
import org.springframework.context.annotation.Configuration
import org.springframework.context.annotation.EnableLoadTimeWeaving
import org.springframework.instrument.classloading.LoadTimeWeaver
import org.springframework.instrument.classloading.ReflectiveLoadTimeWeaver
import org.springframework.context.LoadTimeWeavingConfigurer // Import interface

@Configuration
@EnableLoadTimeWeaving
class CustomWeaverConfiguration : LoadTimeWeavingConfigurer { // Implement interface

    override fun getLoadTimeWeaver(): LoadTimeWeaver { // Override the method
        return ReflectiveLoadTimeWeaver() // Specify the weaver
    }
}
```

**Xml**

```xml
<beans>
    <!-- Define the specific weaver bean -->
    <bean id="loadTimeWeaver"
        class="org.springframework.instrument.classloading.ReflectiveLoadTimeWeaver"/>

    <!-- Enable LTW using the defined weaver -->
    <context:load-time-weaver weaver-class="org.springframework.instrument.classloading.ReflectiveLoadTimeWeaver"/>
    <!-- OR reference the bean directly if needed elsewhere -->
    <!-- <context:load-time-weaver weaver-ref="loadTimeWeaver"/> -->
</beans>
```

구성에 의해 정의되고 등록된 `LoadTimeWeaver`는 나중에 잘 알려진 이름인 `loadTimeWeaver`를 사용하여 스프링 컨테이너에서 검색될 수 있습니다.

`LoadTimeWeaver`는 스프링의 LTW 인프라가 하나 이상의 `ClassFileTransformer`를 추가하는 메커니즘으로만 존재한다는 것을 기억하십시오.

LTW를 수행하는 실제 `ClassFileTransformer`는 `ClassPreProcessorAgentAdapter` (`org.aspectj.weaver.loadtime` 패키지) 클래스입니다.

구성의 마지막 속성 하나가 남았습니다: `aspectjWeaving` 속성 (또는 XML을 사용하는 경우 `aspectj-weaving`). 이 속성은 LTW가 활성화되었는지 여부를 제어합니다.

세 가지 가능한 값 중 하나를 받으며, 속성이 없는 경우 기본값은 `autodetect`입니다. 다음 표는 세 가지 가능한 값을 요약합니다:

AspectJ 위빙 속성 값

| 어노테이션 값 | XML 값 | 설명 |
| --- | --- | --- |
| `ENABLED` | `on` | AspectJ 위빙이 켜져 있으며, 애스펙트가 로드 타임에 적절하게 위빙됩니다. |
| `DISABLED` | `off` | LTW가 꺼져 있습니다. 로드 타임에 애스펙트가 위빙되지 않습니다. |
| `AUTODETECT` | `autodetect` | 스프링 LTW 인프라가 최소 하나 이상의 `META-INF/aop.xml` 파일을 찾을 수 있으면 AspectJ 위빙이 켜집니다. 그렇지 않으면 꺼집니다. 이것이 기본값입니다. |

*환경 특정 구성 (Environment-specific Configuration)*
이 마지막 섹션에는 애플리케이션 서버 및 웹 컨테이너와 같은 환경에서 스프링의 LTW 지원을 사용할 때 필요한 추가 설정 및 구성이 포함됩니다.

*Tomcat, JBoss, WildFly*
Tomcat 및 JBoss/WildFly는 로컬 계측이 가능한 일반 앱 `ClassLoader`를 제공합니다.

스프링의 네이티브 LTW는 AspectJ 위빙을 제공하기 위해 해당 `ClassLoader` 구현을 활용할 수 있습니다.

앞서 설명한 대로 로드 타임 위빙을 간단히 활성화할 수 있습니다. 구체적으로, `-javaagent:path/to/spring-instrument.jar`를 추가하기 위해 JVM 시작 스크립트를 수정할 필요가 없습니다.

*JBoss에서는 애플리케이션이 실제로 시작되기 전에 클래스를 로드하는 것을 방지하기 위해 앱 서버 스캔을 비활성화해야 할 수 있다는 점에 유의하십시오.*

*빠른 해결 방법은 아티팩트에 다음 내용을 포함하는 `WEB-INF/jboss-scanning.xml`이라는 이름의 파일을 추가하는 것입니다:*`<scanning xmlns="urn:jboss:scanning:1.0"/>`

*일반 자바 애플리케이션 (Generic Java Applications)*
특정 `LoadTimeWeaver` 구현에서 지원하지 않는 환경에서 클래스 계측이 필요한 경우 JVM 에이전트가 일반적인 해결책입니다.

이러한 경우를 위해 스프링은 스프링 특정(하지만 매우 일반적인) JVM 에이전트인 `spring-instrument.jar`가 필요한 `InstrumentationLoadTimeWeaver`를 제공하며,

이는 일반적인 `@EnableLoadTimeWeaving` 및 `<context:load-time-weaver/>` 설정에 의해 자동 감지됩니다.

이를 사용하려면 다음 JVM 옵션을 제공하여 스프링 에이전트로 가상 머신을 시작해야 합니다:

- `javaagent:/path/to/spring-instrument.jar`

이는 JVM 시작 스크립트 수정이 필요하며, 이로 인해 애플리케이션 서버 환경(서버 및 운영 정책에 따라 다름)에서 사용하지 못할 수 있다는 점에 유의하십시오.

그렇긴 하지만, 독립 실행형 스프링 부트 애플리케이션과 같은 JVM당 하나의 앱 배포의 경우, 일반적으로 어쨌든 전체 JVM 설정을 제어합니다.

---

**전체 주제: 스프링 애플리케이션과 AspectJ 함께 사용하기 (Using AspectJ with Spring Applications)**

이 부분은 스프링 AOP의 한계(메소드 실행 조인 포인트만 지원, 스프링 빈만 대상 등)를 넘어서, **AspectJ의 더 강력한 기능(다양한 조인 포인트, 모든 객체 대상)을 스프링 애플리케이션에 통합**하는 여러 방법들을 소개합니다.

**핵심 아이디어:** 스프링 AOP만으로 부족할 때, 강력한 AspectJ의 기능을 스프링과 함께 사용하자! 특히 AspectJ의 로드 타임 위빙을 스프링 환경에서 편리하게 설정하고 사용해보자.

---

**첫 번째 파트: 왜 AspectJ를 함께 사용할까?**

- **스프링 AOP의 한계 복습:**
  - 메소드 실행 시점만 지원 (필드 접근, 객체 생성 등 미지원).
  - 주로 스프링 빈 객체만 대상 (일반 도메인 객체 적용 어려움).
  - 프록시 기반으로 인한 자가 호출 문제 등.
- **AspectJ의 강점:**
  - 다양한 조인 포인트 지원 (필드, 생성자, 정적 초기화 등).
  - 모든 자바 객체 대상 가능.
  - 다양한 위빙 시점 지원 (컴파일, 로드, 런타임 - 단, 스프링 통합 시 주로 로드 타임 활용).
- **통합의 목표:** 스프링 IoC 컨테이너의 편리함과 관리 기능을 유지하면서, AspectJ의 강력한 AOP 기능을 활용하여 스프링 AOP만으로는 해결하기 어려운 요구사항(예: 도메인 객체에 직접 의존성 주입, 필드 접근 로깅)을 구현합니다.

---

**두 번째 파트: 스프링으로 도메인 객체에 의존성 주입하기 (`@Configurable`)**

이 부분은 스프링 컨테이너의 제어 **외부에서 생성된 객체**(예: `new`로 직접 생성, ORM이 생성한 도메인 객체)에 **스프링 빈을 의존성으로 주입**하는 방법을 AspectJ와 `@Configurable` 어노테이션을 사용하여 설명합니다.

- **문제점:** `new Account()` 처럼 직접 생성된 객체는 스프링 컨테이너가 관리하지 않으므로, `@Autowired` 등을 사용해도 스프링 빈이 주입되지 않습니다.
- **해결책 (`@Configurable` + AspectJ):**
  1. **`spring-aspects.jar` 추가:** AspectJ 애스펙트 구현체(`AnnotationBeanConfigurerAspect`)가 포함된 이 라이브러리를 클래스패스에 추가합니다.
  2. **`@Configurable` 어노테이션 부착:** 의존성 주입을 받고 싶은 도메인 클래스(예: `Account`) 위에 `@Configurable` 어노테이션을 붙입니다. 이는 "이 클래스의 객체가 생성되면 스프링 설정을 적용해주세요" 라고 표시하는 마커입니다.
  3. **AspectJ 위빙 설정:** 이 클래스가 **AspectJ 위버(컴파일 타임 또는 로드 타임)** 에 의해 **위빙**되도록 설정합니다. 위빙 과정에서 `AnnotationBeanConfigurerAspect` 코드가 `Account` 클래스의 생성자 등에 끼어들어갑니다.
  4. **스프링 설정 활성화:** 스프링 설정(XML 또는 Java Config)에서 `@Configurable` 처리를 **활성화**합니다.
    - **XML:** `<context:spring-configured/>` 태그 추가.
    - **Java Config:** `@EnableSpringConfigured` 어노테이션 추가.
    - (로드 타임 위빙 사용 시) 로드 타임 위빙 활성화 설정도 필요 (`<context:load-time-weaver>` 또는 `@EnableLoadTimeWeaving`)
  5. **(선택) 빈 정의 또는 자동 와이어링 설정:**
    - `@Configurable("빈이름")`: 특정 이름의 스프링 빈 정의(보통 프로토타입 스코프)를 찾아서 해당 설정으로 객체를 구성합니다.
    - `@Configurable(autowire=Autowire.BY_TYPE)` 또는 `BY_NAME`: 자동 와이어링을 사용합니다.
    - (권장) `@Configurable`만 붙이고, **클래스 내부에 `@Autowired` 또는 `@Inject`** 를 사용하여 의존성을 명시적으로 선언합니다.
- **동작:**
  1. `new Account()` 가 호출됩니다.
  2. (위빙된 코드 실행) AspectJ에 의해 삽입된 코드가 `AnnotationBeanConfigurerAspect`를 호출합니다.
  3. `AnnotationBeanConfigurerAspect`는 스프링 컨테이너(`BeanFactory`) 참조를 가지고 있습니다.
  4. 설정된 방식(`@Configurable` 속성 또는 내부 `@Autowired`)에 따라 스프링 컨테이너에서 필요한 빈을 찾아서 **새로 생성된 `Account` 객체에 주입**해줍니다.
  5. (선택: `preConstruction=true`) 생성자 실행 전에 주입할 수도 있습니다.
- **결과:** `new`로 생성된 객체임에도 불구하고 스프링 빈 의존성을 주입받을 수 있게 됩니다.
- **주의:** `@Configurable`은 스프링 빈으로 등록된 클래스에는 사용하면 안 됩니다 (이중 초기화 발생). 단위 테스트 시 위빙되지 않으면 어노테이션은 무시됩니다. 여러 컨텍스트 사용 시 애스펙트 스코프에 주의해야 합니다.

---

**세 번째 파트: AspectJ를 위한 기타 스프링 애스펙트 (`@Transactional`)**

- `spring-aspects.jar`에는 `@Configurable` 처리 애스펙트 외에도 **`@Transactional` 어노테이션을 처리하는 AspectJ 애스펙트(`AnnotationTransactionAspect`)** 도 포함되어 있습니다.
- **용도:** 스프링 컨테이너를 사용하지 않는 환경이나, 스프링 AOP 프록시 방식이 아닌 **AspectJ 위빙 방식**으로 트랜잭션을 적용하고 싶을 때 사용합니다.
- **사용법:**
  1. `spring-aspects.jar`와 AspectJ 위버 설정.
  2. 트랜잭션을 적용할 **구현 클래스 또는 메소드**에 `@Transactional` 어노테이션을 붙입니다. (인터페이스 어노테이션은 상속 안 됨)
  3. 스프링 트랜잭션 관리자 설정을 구성합니다.
  4. AspectJ 위버가 클래스 로드/컴파일 시 `AnnotationTransactionAspect` 코드를 위빙하여 트랜잭션 로직을 적용합니다. (표준 Jakarta EE의 `@jakarta.transaction.Transactional`을 처리하는 `JtaAnnotationTransactionAspect`도 있음)
- **추상 애스펙트:** 개발자가 직접 포인트컷을 정의하여 빈 구성이나 트랜잭션을 적용할 수 있는 `AbstractBeanConfigurerAspect`, `AbstractTransactionAspect` 같은 추상 클래스도 제공됩니다.

---

**네 번째 파트: 스프링 IoC를 사용하여 AspectJ 애스펙트 구성하기**

이 부분은 **AspectJ 런타임이 직접 생성하고 관리하는 애스펙트 객체**에 **스프링의 의존성 주입(DI)** 기능을 적용하는 방법에 대한 내용입니다. (반대 방향의 통합)

- **싱글톤 애스펙트 구성:**
  - 대부분의 AspectJ 애스펙트는 싱글톤입니다.
  - 스프링 설정 파일에 해당 애스펙트 클래스에 대한 **`<bean>` 정의**를 만듭니다.
  - **중요:** `factory-method="aspectOf"` 속성을 추가합니다. 이는 스프링이 직접 `new`로 애스펙트 인스턴스를 만들지 않고, **AspectJ 런타임에게 해당 애스펙트의 싱글톤 인스턴스를 요청**하도록 지시합니다.
  - 이제 이 `<bean>` 정의에 `<property>` 등을 사용하여 다른 스프링 빈을 **의존성으로 주입**할 수 있습니다.

    ```xml
    <bean id="profilerAspect" class="com.xyz.profiler.Profiler" factory-method="aspectOf">
        <property name="profilingStrategy" ref="someStrategyBean"/>
    </bean>
    ```

- **비-싱글톤 애스펙트 구성:**
  - `perthis` 등 비-싱글톤 애스펙트는 구성이 더 복잡합니다.
  - 해당 애스펙트 클래스에 `@Configurable`을 붙이고, 스프링에 프로토타입 스코프의 빈 정의를 만든 다음, AspectJ 런타임이 애스펙트 인스턴스를 생성할 때 `@Configurable` 메커니즘을 통해 스프링 설정을 적용받도록 하는 방식을 사용합니다.
- **스프링 AOP 자동 프록시 대상 필터링:**
  - 만약 일부 `@AspectJ` 애스펙트는 AspectJ 위빙용으로, 다른 일부는 스프링 AOP 자동 프록시용으로 사용하고 싶다면, `<aop:aspectj-autoproxy>` 태그 내부에 **`<aop:include name="빈이름패턴"/>`** 요소를 사용하여 **스프링 AOP가 자동 프록시 대상으로 삼을 애스펙트 빈들을 명시적으로 지정**해야 합니다. 지정되지 않은 `@AspectJ` 빈은 스프링 AOP 자동 프록시에서 제외됩니다.

---

**다섯 번째 파트: 스프링 프레임워크에서의 AspectJ 로드 타임 위빙 (LTW)**

이 부분은 AspectJ의 **로드 타임 위빙(Load-Time Weaving)** 기능을 스프링 환경에서 **설정하고 활용**하는 방법을 상세히 설명합니다.

- **로드 타임 위빙(LTW)이란?** 클래스 로더가 클래스 파일을 JVM에 로드하는 시점에 AspectJ 위버가 바이트코드를 동적으로 변경하여 애스펙트를 적용하는 방식입니다.
- **스프링 LTW의 장점 (순수 AspectJ LTW 대비):**
  - **세분화된 제어:** JVM 전체에 적용하는 대신, **클래스로더별**로 LTW를 선택적으로 켤 수 있습니다. (단일 JVM 내 여러 애플리케이션 환경에 유리)
  - **간편한 설정:** 애플리케이션 서버의 **시작 스크립트를 수정하지 않고도** (즉, `javaagent` JVM 옵션 없이도 특정 환경에서는) 스프링 설정만으로 LTW를 활성화할 수 있는 경우가 있습니다. (예: Tomcat, JBoss 등 특정 WAS 지원)
- **LTW 활성화 및 설정 단계 (예시):**
  1. **애스펙트 작성:** AspectJ 애스펙트를 작성합니다 (`@AspectJ` 스타일 또는 코드 스타일).
  2. **`META-INF/aop.xml` 파일 작성:** AspectJ LTW 설정 파일입니다. 어떤 애스펙트를 사용할지, 어떤 클래스들을 위빙 대상으로 할지 등을 정의합니다. (AspectJ 표준 방식)

      ```xml
      <!DOCTYPE aspectj PUBLIC ...>
      <aspectj>
          <weaver options="-verbose"> <!-- 위빙 옵션 -->
              <include within="com.xyz..*"/> <!-- 특정 패키지만 위빙 대상 -->
          </weaver>
          <aspects>
              <aspect name="com.xyz.ProfilingAspect"/> <!-- 사용할 애스펙트 지정 -->
          </aspects>
      </aspectj>
      ```

  3. **스프링 설정 (`LoadTimeWeaver` 등록):** 스프링 설정(XML 또는 Java Config)에서 로드 타임 위빙 기능을 **활성화**합니다. 이는 스프링에게 LTW를 수행할 `LoadTimeWeaver` 빈을 준비시키고 클래스 로더에 AspectJ 변환기를 등록하도록 지시합니다.
    - **Java Config:** `@EnableLoadTimeWeaving` 어노테이션 사용.
    - **XML Config:** `<context:load-time-weaver/>` 태그 사용.
    - 스프링은 실행 환경(Tomcat, JBoss, 일반 JVM 에이전트 등)을 **자동으로 감지**하여 적절한 `LoadTimeWeaver` 구현체(예: `TomcatLoadTimeWeaver`, `InstrumentationLoadTimeWeaver`)를 선택합니다. 필요하다면 특정 구현체를 명시적으로 지정할 수도 있습니다 (`LoadTimeWeavingConfigurer` 또는 `weaver-class` 속성 사용).
  4. **필수 라이브러리:** `spring-aop.jar`, `aspectjweaver.jar` 가 필요합니다. (스프링 계측 에이전트 사용 시 `spring-instrument.jar` 추가)
  5. **실행 환경 설정 (필요시):**
    - Tomcat, JBoss 등 특정 WAS에서는 추가 설정 없이 스프링 설정만으로 LTW가 동작할 수 있습니다.
    - **일반 자바 애플리케이션**이나 지원되지 않는 환경에서는 JVM 시작 시 **`javaagent:/path/to/spring-instrument.jar`** 옵션을 사용하여 스프링 계측 에이전트를 활성화해야 합니다. 이 에이전트가 클래스 로딩 시점에 `LoadTimeWeaver`가 작동할 수 있도록 훅(hook)을 제공합니다.
- **LTW 효과:** LTW가 활성화되면, 스프링 컨테이너 외부에서 `new`로 생성된 객체라도 `aop.xml`의 위빙 대상에 포함된다면 AspectJ 애스펙트가 적용됩니다. (예제 코드 참고)

**요약:**

스프링은 AspectJ와의 강력한 통합을 제공합니다. `@Configurable`과 AspectJ 위빙을 사용하여 **스프링 외부에서 생성된 도메인 객체에 의존성을 주입**할 수 있습니다. `@Transactional` 처리 등 미리 정의된 AspectJ 애스펙트를 활용할 수도 있습니다. AspectJ 런타임이 생성한 애스펙트 객체에 **스프링 IoC를 통해 의존성을 주입**하는 것도 가능합니다(`factory-method="aspectOf"`). 마지막으로, 스프링은 AspectJ의 **로드 타임 위빙(LTW)** 기능을 스프링 환경에 맞게 **더욱 편리하고 세밀하게 제어**할 수 있도록 지원하며, `@EnableLoadTimeWeaving` 또는 `<context:load-time-weaver/>`를 통해 활성화하고 특정 환경(Tomcat 등)에서는 JVM 에이전트 없이도 사용할 수 있습니다. LTW는 스프링 AOP의 프록시 방식 한계를 넘어 모든 객체와 다양한 조인 포인트에 AOP를 적용할 수 있게 해줍니다.

---

위빙(Weaving)은 AOP(관점 지향 프로그래밍)에서 **매우 핵심적인 개념**입니다. "짜다", "엮다"라는 뜻처럼, **분리된 코드 조각들을 하나로 합치는 과정**을 의미합니다.

**위빙(Weaving)이란?**

- **개념:** AOP에서 **애스펙트(Aspect, 부가 기능 모듈)** 를 **타겟 객체(Target Object, 핵심 로직을 담은 객체)** 에 **적용(application)하여 최종적으로 어드바이스된 객체(Advised Object)** 를 만드는 **과정 전체**를 말합니다.
- **목표:** 핵심 로직 코드에는 손대지 않으면서, 마치 원래부터 있었던 것처럼 부가 기능(어드바이스)이 타겟 객체의 특정 시점(조인 포인트)에 실행되도록 만드는 것입니다.
- **비유:**
  - 씨실(부가 기능)과 날실(핵심 로직)을 엮어서 하나의 옷감(최종 객체)을 짜는 과정.
  - 기본 자동차(타겟 객체)에 튜닝 부품(애스펙트)을 장착하여 튜닝된 자동차(어드바이스된 객체)를 만드는 과정.
  - 연극 대본(타겟 로직)에 조명, 음향 효과(애스펙트)를 추가하여 실제 공연(어드바이스된 객체 실행)을 완성하는 과정.

**위빙이 일어나는 시점:**

위빙은 언제 애스펙트와 타겟 객체를 연결하느냐에 따라 크게 세 가지 시점으로 나뉩니다.

1. **컴파일 타임 위빙 (Compile-time Weaving):**
  - **시점:** 자바 소스 코드를 **컴파일하는 시점**에 위빙이 일어납니다.
  - **방법:** AspectJ 컴파일러(`ajc`)를 사용하여 소스 코드와 애스펙트 코드를 함께 컴파일합니다. 컴파일 결과로 생성되는 `.class` 파일(바이트코드) 자체에 이미 부가 기능 코드가 **삽입**되어 있습니다.
  - **장점:** 실행 시점 성능 오버헤드가 가장 적습니다. 모든 종류의 조인 포인트(필드 접근 등)에 적용 가능합니다.
  - **단점:** 별도의 AspectJ 컴파일러 설정이 필요하고 빌드 과정이 복잡해집니다. 원본 소스 코드를 수정하는 것과 유사한 결과를 낳습니다(바이트코드 레벨에서).
2. **로드 타임 위빙 (Load-time Weaving, LTW):**
  - **시점:** 컴파일된 `.class` 파일이 **JVM의 클래스 로더에 의해 메모리로 로드되는 시점**에 위빙이 일어납니다.
  - **방법:** JVM 실행 시 특별한 **위빙 에이전트(Weaving Agent)** (예: AspectJ 위빙 에이전트, `spring-instrument.jar`)를 사용하여 클래스 로더가 클래스 바이트코드를 읽을 때 중간에서 가로채 내용을 **동적으로 수정(변환, transform)** 하여 부가 기능 코드를 삽입합니다.
  - **장점:** 컴파일 시점 위빙처럼 모든 종류의 조인 포인트를 지원하면서도, 원본 소스 코드나 컴파일된 바이트코드를 직접 수정하지 않습니다. 애플리케이션 서버 환경 등에서 유연하게 적용 가능합니다.
  - **단점:** JVM 에이전트 설정이 필요하며, 클래스 로딩 시 약간의 오버헤드가 발생할 수 있습니다. 설정이 다소 복잡할 수 있습니다. (스프링은 이 과정을 좀 더 쉽게 할 수 있도록 지원)
3. **런타임 위빙 (Runtime Weaving):**
  - **시점:** 애플리케이션이 **실행되는 도중**에, 스프링 컨테이너가 빈 객체를 생성할 때 위빙이 일어납니다.
  - **방법:** **스프링 AOP가 사용하는 방식**입니다. 원본 대상 객체(Target)를 직접 수정하는 대신, 대상 객체를 **감싸는 프록시(Proxy) 객체**를 런타임에 동적으로 생성합니다. 이 프록시 객체가 부가 기능(어드바이스) 실행을 담당하고 내부적으로 원본 객체의 메소드를 호출합니다.
  - **장점:** 별도의 컴파일러나 JVM 에이전트 설정 없이 순수 자바만으로 AOP를 구현할 수 있어 **가장 간단**합니다. 스프링 IoC 컨테이너와 긴밀하게 통합됩니다.
  - **단점:** **메소드 실행 조인 포인트만** 지원합니다. 프록시 방식 자체의 한계(자가 호출 문제 등)가 있습니다. 약간의 런타임 성능 오버헤드가 있을 수 있습니다 (현대 JVM에서는 미미할 수 있음).

**결론:**

**위빙(Weaving)** 은 분리된 **부가 기능(애스펙트)** 을 **핵심 로직(타겟 객체)** 에 **적용하여 최종적으로 실행될 객체를 만드는 과정**입니다. 어떤 시점(컴파일, 로드, 런타임)에 이 작업을 수행하느냐에 따라 방식과 장단점이 달라집니다. 스프링 AOP는 기본적으로 **런타임 위빙(프록시 생성)** 방식을 사용하며, 필요하다면 AspectJ와 통합하여 **로드 타임 위빙**이나 **컴파일 타임 위빙**을 활용할 수도 있습니다.
