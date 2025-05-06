---
title: Using the ProxyFactoryBean to Create AOP Proxies
description: 
author: laze
date: 2025-05-06 00:00:09 +0900
categories: [Dev, SpringBoot]
tags: [SpringBoot]
---
**Using the ProxyFactoryBean to Create AOP Proxies**

비즈니스 객체에 대해 스프링 IoC 컨테이너(`ApplicationContext` 또는 `BeanFactory`)를 사용한다면 (그리고 그래야 합니다!),

스프링의 AOP `FactoryBean` 구현 중 하나를 사용하고 싶을 것입니다. (`FactoryBean`이 다른 타입의 객체를 생성할 수 있게 해주는 간접 참조(indirection) 계층을 도입한다는 것을 기억하십시오.)

*스프링 AOP 지원은 또한 내부적으로(under the covers) 팩토리 빈을 사용합니다.*

스프링에서 AOP 프록시를 생성하는 기본적인 방법은 `org.springframework.aop.framework.ProxyFactoryBean`을 사용하는 것입니다.

이는 포인트컷, 적용되는 모든 어드바이스 및 그 순서에 대한 완전한 제어를 제공합니다.

그러나 이러한 제어가 필요하지 않은 경우 선호되는 더 간단한 옵션들이 있습니다.

**기본 사항 (Basics)**

다른 스프링 `FactoryBean` 구현과 마찬가지로 `ProxyFactoryBean`은 간접 참조 수준을 도입합니다.

`foo`라는 이름의 `ProxyFactoryBean`을 정의하면, `foo`를 참조하는 객체는 `ProxyFactoryBean` 인스턴스 자체가 아니라 `ProxyFactoryBean`의 `getObject()` 메소드 구현에 의해 생성된 객체를 봅니다.

이 메소드는 대상 객체를 래핑(wrap)하는 AOP 프록시를 생성합니다.

AOP 프록시를 생성하기 위해 `ProxyFactoryBean` 또는 다른 IoC 인식 클래스를 사용하는 가장 중요한 이점 중 하나는 어드바이스와 포인트컷도 IoC에 의해 관리될 수 있다는 것입니다.

이것은 다른 AOP 프레임워크로는 달성하기 어려운 특정 접근 방식을 가능하게 하는 강력한 기능입니다.

예를 들어, 어드바이스 자체는 (모든 AOP 프레임워크에서 사용 가능해야 하는 대상 외에) 애플리케이션 객체를 참조할 수 있으며, 의존성 주입에 의해 제공되는 모든 연결성(pluggability)의 이점을 누릴 수 있습니다.

**JavaBean 속성 (JavaBean Properties)**

스프링과 함께 제공되는 대부분의 `FactoryBean` 구현과 마찬가지로, `ProxyFactoryBean` 클래스 자체는 JavaBean입니다. 그 속성은 다음을 위해 사용됩니다:

- 프록시하려는 대상 지정.
- CGLIB 사용 여부 지정 (나중에 설명되며 JDK 및 CGLIB 기반 프록시도 참조).

일부 핵심 속성은 `org.springframework.aop.framework.ProxyConfig`(스프링의 모든 AOP 프록시 팩토리의 슈퍼클래스)에서 상속됩니다. 이러한 핵심 속성에는 다음이 포함됩니다:

- `proxyTargetClass`: 대상 클래스의 인터페이스 대신 대상 클래스를 프록시하려면 `true`. 이 속성 값이 `true`로 설정되면 CGLIB 프록시가 생성됩니다 (하지만 JDK 및 CGLIB 기반 프록시도 참조).
- `optimize`: CGLIB를 통해 생성된 프록시에 공격적인 최적화를 적용할지 여부를 제어합니다. 관련 AOP 프록시가 최적화를 어떻게 처리하는지 완전히 이해하지 않는 한 이 설정을 경솔하게(blithely) 사용해서는 안 됩니다. 이것은 현재 CGLIB 프록시에만 사용됩니다. JDK 동적 프록시에는 효과가 없습니다.
- `frozen`: 프록시 구성이 동결(frozen)되면 구성 변경이 더 이상 허용되지 않습니다. 이는 약간의 최적화와 프록시 생성 후 호출자가 프록시(`Advised` 인터페이스를 통해)를 조작할 수 없도록 하려는 경우 모두 유용합니다. 이 속성의 기본값은 `false`이므로 변경(예: 추가 어드바이스 추가)이 허용됩니다.
- `exposeProxy`: 현재 프록시가 대상에 의해 접근될 수 있도록 `ThreadLocal`에 노출되어야 하는지 여부를 결정합니다. 대상이 프록시를 얻어야 하고 `exposeProxy` 속성이 `true`로 설정된 경우, 대상은 `AopContext.currentProxy()` 메소드를 사용할 수 있습니다.

`ProxyFactoryBean`에 특정한 다른 속성에는 다음이 포함됩니다:

- `proxyInterfaces`: `String` 인터페이스 이름의 배열. 이것이 제공되지 않으면 대상 클래스에 대한 CGLIB 프록시가 사용됩니다 (하지만 JDK 및 CGLIB 기반 프록시도 참조).
- `interceptorNames`: 적용할 `Advisor`, 인터셉터 또는 기타 어드바이스 이름의 `String` 배열. 순서는 선착순으로 중요합니다. 즉, 목록의 첫 번째 인터셉터가 호출을 가로챌 수 있는 첫 번째입니다.
  - 이름은 조상 팩토리의 빈 이름을 포함하여 현재 팩토리의 빈 이름입니다. 여기서 빈 참조를 언급할 수 없습니다. 그렇게 하면 `ProxyFactoryBean`이 어드바이스의 싱글톤 설정을 무시하게 됩니다.
  - 인터셉터 이름 뒤에 별표()를 추가할 수 있습니다. 그렇게 하면 별표 앞 부분으로 시작하는 이름을 가진 모든 어드바이저 빈이 적용되는 결과가 발생합니다. 이 기능 사용 예는 "전역" 어드바이저 사용하기(Using “Global” Advisors)에서 찾을 수 있습니다.
- `singleton`: `getObject()` 메소드가 몇 번 호출되든 관계없이 팩토리가 단일 객체를 반환해야 하는지 여부. 여러 `FactoryBean` 구현이 이러한 메소드를 제공합니다. 기본값은 `true`입니다. 상태 저장 어드바이스(예: 상태 저장 믹스인)를 사용하려면 `false`의 `singleton` 값과 함께 프로토타입 어드바이스를 사용하십시오.

**JDK 및 CGLIB 기반 프록시 (JDK- and CGLIB-based proxies)**

프록시될 대상 객체의 클래스(이하 단순히 대상 클래스라고 함)가 인터페이스를 구현하지 않는 경우, CGLIB 기반 프록시가 생성됩니다.

이것이 가장 쉬운 시나리오입니다.

왜냐하면 JDK 프록시는 인터페이스 기반이며 인터페이스가 없다는 것은 JDK 프록시가 불가능하다는 것을 의미하기 때문입니다.

대상 빈을 연결하고 `interceptorNames` 속성을 설정하여 인터셉터 목록을 지정할 수 있습니다.

`ProxyFactoryBean`의 `proxyTargetClass` 속성이 `false`로 설정된 경우에도 CGLIB 기반 프록시가 생성된다는 점에 유의하십시오. (그렇게 하는 것은 의미가 없으며 빈 정의에서 제거하는 것이 가장 좋습니다. 왜냐하면 기껏해야 중복되고 최악의 경우 혼란스럽기 때문입니다.)

대상 클래스가 하나 (또는 그 이상)의 인터페이스를 구현하는 경우, 생성되는 프록시 타입은 `ProxyFactoryBean`의 구성에 따라 달라집니다.

- `ProxyFactoryBean`의 `proxyTargetClass` 속성이 `true`로 설정된 경우, CGLIB 기반 프록시가 생성됩니다. 이는 합리적이며 최소 놀람의 원칙(principle of least surprise)과 일치합니다. `ProxyFactoryBean`의 `proxyInterfaces` 속성이 하나 이상의 정규화된 인터페이스 이름으로 설정된 경우에도, `proxyTargetClass` 속성이 `true`로 설정된 사실 때문에 CGLIB 기반 프록시가 적용됩니다.
- `ProxyFactoryBean`의 `proxyInterfaces` 속성이 하나 이상의 정규화된 인터페이스 이름으로 설정된 경우, JDK 기반 프록시가 생성됩니다. 생성된 프록시는 `proxyInterfaces` 속성에 지정된 모든 인터페이스를 구현합니다. 대상 클래스가 `proxyInterfaces` 속성에 지정된 것보다 훨씬 많은 인터페이스를 구현하는 경우에도 괜찮지만, 해당 추가 인터페이스는 반환된 프록시에 의해 구현되지 않습니다.
- `ProxyFactoryBean`의 `proxyInterfaces` 속성이 설정되지 않았지만 대상 클래스가 하나 (또는 그 이상)의 인터페이스를 구현하는 경우, `ProxyFactoryBean`은 대상 클래스가 실제로 최소한 하나의 인터페이스를 구현한다는 사실을 자동 감지하고 JDK 기반 프록시가 생성됩니다. 실제로 프록시되는 인터페이스는 대상 클래스가 구현하는 모든 인터페이스입니다. 사실상, 이는 대상 클래스가 구현하는 모든 인터페이스 목록을 `proxyInterfaces` 속성에 제공하는 것과 동일합니다. 그러나 이는 훨씬 적은 작업이며 오타가 발생할 가능성이 적습니다.

**인터페이스 프록시하기 (Proxying Interfaces)**

`ProxyFactoryBean` 작동의 간단한 예제를 고려해 봅시다. 이 예제에는 다음이 포함됩니다:

- 프록시될 대상 빈. 예제의 `personTarget` 빈 정의입니다.
- 어드바이스를 제공하는 데 사용되는 `Advisor` 및 `Interceptor`.
- 대상 객체(`personTarget` 빈), 프록시할 인터페이스 및 적용할 어드바이스를 지정하는 AOP 프록시 빈 정의.

다음 목록은 예제를 보여줍니다:

```xml
<!-- Target Bean -->
<bean id="personTarget" class="com.mycompany.PersonImpl">
	<property name="name" value="Tony"/>
	<property name="age" value="51"/>
</bean>

<!-- Advisor Bean -->
<bean id="myAdvisor" class="com.mycompany.MyAdvisor"> <!-- Assuming MyAdvisor implements Advisor -->
	<property name="someProperty" value="Custom string property value"/>
</bean>

<!-- Interceptor Bean -->
<bean id="debugInterceptor" class="org.springframework.aop.interceptor.DebugInterceptor"/>

<!-- Proxy Factory Bean -->
<bean id="person"
	class="org.springframework.aop.framework.ProxyFactoryBean">
	<property name="proxyInterfaces" value="com.mycompany.Person"/> <!-- Interface to proxy -->

	<property name="target" ref="personTarget"/> <!-- Target object -->
	<property name="interceptorNames"> <!-- Names of interceptors/advisors -->
		<list>
			<value>myAdvisor</value> <!-- Advisor bean name -->
			<value>debugInterceptor</value> <!-- Interceptor bean name -->
		</list>
	</property>
</bean>
```

`interceptorNames` 속성은 현재 팩토리의 인터셉터 또는 어드바이저의 빈 이름을 보유하는 `String` 목록을 받습니다.

어드바이저, 인터셉터, before, after returning 및 throws 어드바이스 객체를 사용할 수 있습니다.

어드바이저 순서는 중요합니다.

*목록이 빈 참조를 보유하지 않는 이유가 궁금할 수 있습니다.*

*그 이유는 `ProxyFactoryBean`의 `singleton` 속성이 `false`로 설정된 경우 독립적인 프록시 인스턴스를 반환할 수 있어야 하기 때문입니다.*

*어드바이저 중 하나라도 자체가 프로토타입인 경우 독립적인 인스턴스를 반환해야 하므로 팩토리에서 프로토타입 인스턴스를 얻을 수 있어야 합니다.*

*참조를 보유하는 것만으로는 충분하지 않습니다.*

앞서 보여준 `person` 빈 정의는 다음과 같이 `Person` 구현 대신 사용할 수 있습니다:

```java
// Java
import org.springframework.context.ApplicationContext;
// Assuming Person interface exists and context is initialized

Person person = (Person) factory.getBean("person"); // Get the proxied bean
```

```kotlin
// Kotlin
import org.springframework.context.ApplicationContext
// Assuming Person interface exists and context is initialized

val person = factory.getBean("person") as Person // Get the proxied bean with Kotlin cast
```

동일한 IoC 컨텍스트의 다른 빈들은 일반 자바 객체처럼 강력한 타입 지정 의존성을 표현할 수 있습니다. 다음 예제는 이를 수행하는 방법을 보여줍니다:

```xml
<bean id="personUser" class="com.mycompany.PersonUser"> <!-- Assuming PersonUser class exists -->
	<property name="person"><ref bean="person"/></property> <!-- Inject the proxied bean -->
</bean>
```

이 예제의 `PersonUser` 클래스는 `Person` 타입의 속성을 노출합니다.

그것에 관한 한, AOP 프록시는 "실제" person 구현 대신 투명하게 사용될 수 있습니다.

그러나 해당 클래스는 동적 프록시 클래스가 됩니다. 이를 `Advised` 인터페이스(나중에 논의됨)로 캐스팅하는 것이 가능할 것입니다.

익명 내부 빈(anonymous inner bean)을 사용하여 대상과 프록시 간의 구분을 숨길 수 있습니다.

`ProxyFactoryBean` 정의만 다릅니다. 어드바이스는 완전성을 위해서만 포함됩니다. 다음 예제는 익명 내부 빈 사용 방법을 보여줍니다:

```xml
<bean id="myAdvisor" class="com.mycompany.MyAdvisor">
	<property name="someProperty" value="Custom string property value"/>
</bean>

<bean id="debugInterceptor" class="org.springframework.aop.interceptor.DebugInterceptor"/>

<bean id="person" class="org.springframework.aop.framework.ProxyFactoryBean">
	<property name="proxyInterfaces" value="com.mycompany.Person"/>
	<!-- 로컬 타겟 참조 대신 내부 빈 사용 -->
	<property name="target">
		<bean class="com.mycompany.PersonImpl"> <!-- Define target bean inline -->
			<property name="name" value="Tony"/>
			<property name="age" value="51"/>
		</bean>
	</property>
	<property name="interceptorNames">
		<list>
			<value>myAdvisor</value>
			<value>debugInterceptor</value>
		</list>
	</property>
</bean>

```

익명 내부 빈을 사용하는 것의 장점은 `Person` 타입의 객체가 하나만 있다는 것입니다.

이는 애플리케이션 컨텍스트 사용자가 어드바이스받지 않은 객체에 대한 참조를 얻는 것을 방지하거나 스프링 IoC 자동 와이어링과의 모호함을 피해야 하는 경우 유용합니다.

또한 `ProxyFactoryBean` 정의가 독립적(self-contained)이라는 장점도 있습니다.

그러나 팩토리에서 어드바이스받지 않은 대상을 얻을 수 있는 것이 실제로 이점일 때도 있습니다 (예: 특정 테스트 시나리오에서).

**클래스 프록시하기 (Proxying Classes)**

하나 이상의 인터페이스 대신 클래스를 프록시해야 하는 경우는 어떨까요?

이전 예제에서 `Person` 인터페이스가 없다고 상상해 봅시다. 비즈니스 인터페이스를 구현하지 않은 `Person`이라는 클래스를 어드바이스해야 했습니다.

이 경우, 동적 프록시 대신 CGLIB 프록시를 사용하도록 스프링을 구성할 수 있습니다.

그렇게 하려면 앞서 보여준 `ProxyFactoryBean`의 `proxyTargetClass` 속성을 `true`로 설정합니다.

클래스보다는 인터페이스를 대상으로 프로그래밍하는 것이 가장 좋지만, 인터페이스를 구현하지 않는 클래스를 어드바이스하는 기능은 레거시 코드로 작업할 때 유용할 수 있습니다. (일반적으로 스프링은 규범적이지 않습니다. 좋은 관행을 쉽게 적용할 수 있게 하면서도 특정 접근 방식을 강요하는 것을 피합니다.)

원한다면 인터페이스가 있는 경우에도 어떤 경우든 CGLIB 사용을 강제할 수 있습니다.

CGLIB 프록시는 런타임 시 대상 클래스의 하위 클래스를 생성하여 작동합니다.

스프링은 이 생성된 하위 클래스가 메소드 호출을 원본 대상으로 위임하도록 구성합니다.

하위 클래스는 데코레이터(Decorator) 패턴을 구현하고 어드바이스를 위빙(weaving)하는 데 사용됩니다.

CGLIB 프록시는 일반적으로 사용자에게 투명해야 합니다. 그러나 고려해야 할 몇 가지 문제가 있습니다:

- `final` 클래스는 확장될 수 없으므로 프록시될 수 없습니다.
- `final` 메소드는 오버라이드될 수 없으므로 어드바이스될 수 없습니다.
- `private` 메소드는 오버라이드될 수 없으므로 어드바이스될 수 없습니다.
- 일반적으로 다른 패키지의 부모 클래스에 있는 패키지 전용(package private) 메소드와 같이 보이지 않는 메소드는 효과적으로 `private`이므로 어드바이스될 수 없습니다.

클래스패스에 CGLIB를 추가할 필요가 없습니다. CGLIB는 재패키징되어 `spring-core` JAR에 포함됩니다.

즉, JDK 동적 프록시와 마찬가지로 CGLIB 기반 AOP는 "즉시(out of the box)" 작동합니다.

CGLIB 프록시와 동적 프록시 사이에는 성능 차이가 거의 없습니다. 이 경우 성능이 결정적인 고려 사항이 되어서는 안 됩니다.

**"전역" 어드바이저 사용하기 (Using “Global” Advisors)**

인터셉터 이름에 별표(`*`)를 추가하면, 별표 앞 부분과 일치하는 빈 이름을 가진 모든 어드바이저가 어드바이저 체인에 추가됩니다.

이는 표준 "전역" 어드바이저 세트를 추가해야 할 때 유용할 수 있습니다. 다음 예제는 두 개의 전역 어드바이저를 정의합니다:

```xml
<bean id="proxy" class="org.springframework.aop.framework.ProxyFactoryBean">
	<property name="target" ref="service"/> <!-- Assume 'service' bean exists -->
	<property name="interceptorNames">
		<list>
			<value>global*</value> <!-- Add all beans starting with 'global' -->
		</list>
	</property>
</bean>

<!-- Global advisors -->
<bean id="global_debug" class="org.springframework.aop.interceptor.DebugInterceptor"/>
<bean id="global_performance" class="org.springframework.aop.interceptor.PerformanceMonitorInterceptor"/>
```

---

**전체 주제: ProxyFactoryBean을 사용하여 AOP 프록시 생성하기**

이 부분은 스프링 IoC 컨테이너 환경에서 **`org.springframework.aop.framework.ProxyFactoryBean`** 이라는 특별한 종류의 빈(FactoryBean)을 사용하여 **AOP 프록시를 수동으로 생성하고 구성**하는 방법을 설명합니다.

**핵심 아이디어:** 스프링의 자동 프록시 생성 기능 대신, `ProxyFactoryBean`이라는 빈을 직접 설정하여 어떤 대상 객체에 어떤 어드바이스(부가 기능)들을 어떤 순서로 적용할지 세밀하게 제어하여 AOP 프록시를 만들자.

---

**1. `ProxyFactoryBean`이란? (기본 개념)**

- **FactoryBean:** `ProxyFactoryBean`은 스프링의 `FactoryBean` 인터페이스 구현체입니다. `FactoryBean`은 자신이 직접 빈 인스턴스가 되는 것이 아니라, **다른 객체를 생성하고 관리하는 팩토리 역할**을 하는 특별한 빈입니다.
- **AOP 프록시 생성 팩토리:** `ProxyFactoryBean`의 역할은 설정된 정보를 바탕으로 **AOP 프록시 객체를 생성**하여 반환하는 것입니다.
- **간접 참조:** 스프링 컨테이너에 `id="myProxy"`로 `ProxyFactoryBean`을 정의하면, 다른 빈에서 `myProxy`를 주입받을 때 `ProxyFactoryBean` 객체 자체가 아니라, `ProxyFactoryBean`의 `getObject()` 메소드가 **생성한 AOP 프록시 객체**를 받게 됩니다. (스프링이 내부적으로 처리해 줌)

---

**2. 왜 `ProxyFactoryBean`을 사용할까? (장점)**

- **IoC 관리의 이점:** `ProxyFactoryBean`을 사용하는 가장 큰 장점은 **어드바이스(Advice)와 포인트컷(Pointcut) 자체도 스프링 IoC 컨테이너에 의해 빈으로 관리**될 수 있다는 것입니다.
  - 즉, 어드바이스 빈에 다른 빈들을 의존성으로 주입하거나, 포인트컷 설정을 외부화하는 등 스프링 IoC의 모든 장점을 활용할 수 있습니다. (다른 AOP 프레임워크에서는 어려울 수 있음)
- **세밀한 제어:** 어떤 인터페이스를 프록시할지, 어떤 어드바이스들을 어떤 순서로 적용할지, CGLIB 프록시를 강제할지 등을 **매우 상세하게 직접 제어**할 수 있습니다.
- **언제 고려할까?:** 자동 프록시 생성(`@EnableAspectJAutoProxy`, `<aop:aspectj-autoproxy>`)이 대부분의 경우 더 간편하지만, 다음과 같은 경우 `ProxyFactoryBean` 사용을 고려할 수 있습니다.
  - 매우 복잡하고 동적인 AOP 설정을 프로그래밍 방식으로 제어해야 할 때 (하지만 `ProxyFactory` 직접 사용이 더 나을 수 있음).
  - 특정 빈에 대해서만 매우 특수한 프록시 설정을 적용하고 싶을 때.
  - (과거) 스프링 초기 버전에서는 이 방식이 주요 AOP 설정 방법 중 하나였습니다. (현재는 덜 선호됨)

---

**3. 주요 JavaBean 속성:**

`ProxyFactoryBean`은 다양한 속성을 통해 프록시 생성 방식을 제어합니다.

- **상위 클래스(`ProxyConfig`) 상속 속성:**
  - **`proxyTargetClass` (boolean):** `true`로 설정하면 인터페이스 구현 여부와 상관없이 **항상 CGLIB 프록시**(클래스 상속 기반)를 생성합니다. `false`(기본값)이면 인터페이스가 있을 경우 JDK 동적 프록시를 사용합니다.
  - `optimize` (boolean): CGLIB 프록시 생성 시 공격적인 최적화를 적용할지 여부 (잘 모르면 사용하지 않는 것이 좋음).
  - `frozen` (boolean): `true`로 설정하면 프록시 생성 후 설정을 변경할 수 없도록 동결합니다 (약간의 최적화 및 안정성). 기본값은 `false`.
  - `exposeProxy` (boolean): `true`로 설정하면 대상 객체 내부에서 `AopContext.currentProxy()`를 통해 현재 프록시 객체에 접근할 수 있도록 허용합니다. (자가 호출 문제 해결 방법 중 하나, 비추천)
- **`ProxyFactoryBean` 고유 속성:**
  - **`target` 또는 `targetName`:** **프록시할 원본 대상 객체**를 지정합니다.
    - `target`: 대상 객체 빈에 대한 직접 참조 (`<property name="target" ref="..."/>`).
    - `targetName`: 대상 객체 빈의 이름 (`<property name="targetName" value="..."/>`). `singleton=false`일 때 주로 사용.
  - **`proxyInterfaces` (String[]):** **프록시가 구현해야 할 인터페이스**들의 정규화된 이름 배열입니다. 지정하면 JDK 동적 프록시가 생성됩니다 (단, `proxyTargetClass=true`가 아니어야 함). 지정하지 않고 대상 객체가 인터페이스를 구현하면 자동으로 감지하여 JDK 프록시를 만듭니다.
  - **`interceptorNames` (String[]):** **적용할 어드바이스(Advice), 인터셉터(Interceptor), 어드바이저(Advisor) 빈들의 이름**을 순서대로 나열한 문자열 배열입니다. **순서가 중요**하며, 이 순서대로 인터셉터 체인이 구성됩니다.
    - **주의:** 여기에 빈 **참조(`ref`)가 아닌 빈 이름(문자열)** 을 사용해야 합니다. (프로토타입 어드바이스 지원 등을 위해)
    - **와일드카드() 지원:** 이름 끝에 를 붙이면 해당 패턴으로 시작하는 모든 어드바이저 빈이 적용됩니다 (예: `global*`).
  - `singleton` (boolean): 이 팩토리가 항상 동일한 프록시 인스턴스를 반환할지(기본값 `true`), 아니면 매번 새로운 프록시 인스턴스를 생성할지(프로토타입 프록시, `false`) 결정합니다. 상태 저장(stateful) 어드바이스(예: 인트로덕션 믹스인)를 사용할 때 `false`로 설정해야 할 수 있습니다.

---

**4. JDK 동적 프록시 vs CGLIB 프록시 결정 로직:**

`ProxyFactoryBean`이 어떤 종류의 프록시를 생성할지는 다음 규칙에 따라 결정됩니다:

1. **대상 클래스가 인터페이스를 구현하지 않으면:** **무조건 CGLIB 프록시** 생성.
2. **대상 클래스가 인터페이스를 구현하는 경우:**
  - **`proxyTargetClass` 속성이 `true`이면:** **무조건 CGLIB 프록시** 생성. (`proxyInterfaces` 설정은 무시됨)
  - **`proxyInterfaces` 속성이 설정되어 있으면:** **JDK 동적 프록시** 생성. 프록시는 `proxyInterfaces`에 명시된 인터페이스들만 구현합니다.
  - **`proxyInterfaces` 속성이 없고 `proxyTargetClass`도 `false`(기본값)이면:** 스프링이 대상 클래스가 구현한 **모든 인터페이스를 자동으로 감지**하여 **JDK 동적 프록시**를 생성합니다.

---

**5. 사용 예시:**

- **인터페이스 프록시하기:** 대상 빈(`personTarget`), 어드바이저(`myAdvisor`), 인터셉터(`debugInterceptor`)를 각각 빈으로 정의하고, `ProxyFactoryBean`(`person`)에서 `proxyInterfaces`, `target`, `interceptorNames`를 설정하여 프록시를 생성합니다. 다른 빈(`personUser`)에서는 생성된 프록시 빈(`person`)을 주입받아 사용합니다.
- **클래스 프록시하기:** 대상 클래스가 인터페이스를 구현하지 않을 경우, `proxyTargetClass="true"` 속성을 설정하여 CGLIB 프록시를 생성하도록 지시합니다. (인터페이스가 없으면 자동으로 CGLIB이 사용되므로 사실 이 속성은 필수는 아님)
- **익명 내부 빈 사용:** `ProxyFactoryBean`의 `target` 속성 내부에 `<bean>` 태그를 사용하여 대상 객체를 직접 정의할 수 있습니다. 이렇게 하면 어드바이스되지 않은 원본 객체가 컨테이너에 노출되지 않는 장점이 있습니다.
- **"전역" 어드바이저 사용:** `interceptorNames`에 `global*` 와 같이 와일드카드를 사용하여 특정 패턴의 이름을 가진 모든 어드바이저 빈들을 한 번에 적용할 수 있습니다.

**요약:**

`ProxyFactoryBean`은 스프링 IoC 컨테이너 내에서 AOP 프록시를 **수동으로, 상세하게 구성**하기 위한 `FactoryBean`입니다. 대상 객체, 프록시할 인터페이스(선택), 적용할 어드바이스/인터셉터/어드바이저 목록 및 순서, 프록시 생성 방식(JDK/CGLIB), 프록시 자체의 설정 등을 속성을 통해 제어할 수 있습니다. 어드바이스와 포인트컷도 빈으로 관리할 수 있다는 장점이 있지만, 설정이 `@AspectJ`나 스키마 기반 방식보다 번거로울 수 있어 현재는 **덜 선호되는 방식**입니다. 하지만 스프링 AOP의 내부 동작 방식을 이해하고 세밀한 제어가 필요할 때 유용할 수 있습니다.
