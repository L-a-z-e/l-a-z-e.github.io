---
title: Spring Framework Dependency Injection
description: 
author: laze
date: 2025-05-01 00:00:01 +0900
categories: [Dev, SpringBoot]
tags: [SpringBoot]
---
# **Dependency Injection**

**의존성 주입 (Dependency Injection)**

의존성 주입(DI)은 객체들이 자신의 의존성(즉, 함께 작동하는 다른 객체들)을 생성자 인수, 팩토리 메소드의 인수, 또는 객체 인스턴스가 생성되거나 팩토리 메소드에서 반환된 후 설정되는 속성들을 통해서만 정의하는 프로세스입니다.

그러면 컨테이너는 빈(bean)을 생성할 때 해당 의존성들을 주입합니다.

이 과정은 빈 자체가 클래스의 직접 생성이나 서비스 로케이터(Service Locator) 패턴을 사용하여 자신의 의존성을 인스턴스화하거나 위치를 찾는 것을 직접 제어하는 방식과는 근본적으로 반대(역전, 그래서 제어의 역전이라는 이름이 붙음)입니다.

DI 원칙을 사용하면 코드가 더 깨끗해지고, 객체에 의존성이 제공될 때 분리(decoupling)가 더 효과적입니다.

객체는 자신의 의존성을 찾지 않으며 의존성의 위치나 클래스를 알지 못합니다.

결과적으로, 클래스를 테스트하기가 더 쉬워지며, 특히 의존성이 인터페이스나 추상 기본 클래스에 있을 때 단위 테스트에서 스텁(stub) 또는 모의(mock) 구현을 사용할 수 있게 됩니다.

DI는 생성자 기반 의존성 주입과 세터 기반 의존성 주입이라는 두 가지 주요 변형으로 존재합니다.

**생성자 기반 의존성 주입 (Constructor-based Dependency Injection)**

생성자 기반 DI는 컨테이너가 여러 개의 인수를 가진 생성자를 호출함으로써 이루어지며, 각 인수는 의존성을 나타냅니다.

빈을 생성하기 위해 특정 인수를 가진 정적 팩토리 메소드를 호출하는 것도 거의 동일하며, 이 논의에서는 생성자에 대한 인수와 정적 팩토리 메소드에 대한 인수를 유사하게 취급합니다.

다음 예제는 생성자 주입을 통해서만 의존성 주입될 수 있는 클래스를 보여줍니다:

```java
// Java
public class SimpleMovieLister {

	// SimpleMovieLister는 MovieFinder에 대한 의존성을 가집니다
	private final MovieFinder movieFinder;

	// 스프링 컨테이너가 MovieFinder를 주입할 수 있도록 하는 생성자
	public SimpleMovieLister(MovieFinder movieFinder) {
		this.movieFinder = movieFinder;
	}

	// 실제로 주입된 MovieFinder를 사용하는 비즈니스 로직은 생략...
}

```

```kotlin
// Kotlin
class SimpleMovieLister(
    // SimpleMovieLister는 MovieFinder에 대한 의존성을 가집니다
    private val movieFinder: MovieFinder
) {
    // 실제로 주입된 MovieFinder를 사용하는 비즈니스 로직은 생략...
}

```

컨테이너 특정 인터페이스, 기본 클래스 또는 어노테이션에 대한 의존성이 없는 POJO입니다.

*생성자 인수 해석 (Constructor Argument Resolution)*

생성자 인수 해석 매칭은 인수의 타입을 사용하여 발생합니다.

빈 정의의 생성자 인수에 잠재적인 모호성이 없다면, 빈 정의에서 생성자 인수가 정의된 순서가 빈이 인스턴스화될 때 해당 생성자에 인수가 제공되는 순서입니다.

```java
// Java
package x.y;

public class ThingOne {

	public ThingOne(ThingTwo thingTwo, ThingThree thingThree) {
		// ...
	}
}

```

```kotlin
// Kotlin
package x.y

class ThingOne(thingTwo: ThingTwo, thingThree: ThingThree) {
    // ...
}

```

`ThingTwo`와 `ThingThree` 클래스가 상속 관계로 관련되어 있지 않다고 가정하면, 잠재적인 모호성이 없습니다. 따라서 다음 구성은 잘 작동하며, `<constructor-arg/>` 요소에서 생성자 인수의 인덱스나 타입을 명시적으로 지정할 필요가 없습니다.

```xml
<beans>
	<bean id="beanOne" class="x.y.ThingOne">
		<constructor-arg ref="beanTwo"/>
		<constructor-arg ref="beanThree"/>
	</bean>

	<bean id="beanTwo" class="x.y.ThingTwo"/>

	<bean id="beanThree" class="x.y.ThingThree"/>
</beans>

```

다른 빈이 참조될 때는 타입이 알려져 있으므로 매칭이 발생할 수 있습니다 (앞의 예제와 같이). `<value>true</value>`와 같은 단순 타입이 사용될 때는, 스프링은 값의 타입을 결정할 수 없으므로 도움 없이는 타입으로 매칭할 수 없습니다.

```java
// Java
package examples;

public class ExampleBean {
	private final int years;

	private final String ultimateAnswer;

	public ExampleBean(int years, String ultimateAnswer) {
		this.years = years;
		this.ultimateAnswer = ultimateAnswer;
	}
}
```

```kotlin
// Kotlin
package examples

class ExampleBean(

    private val years: Int,
    private val ultimateAnswer: String
)
```

*생성자 인수 타입 매칭*
앞의 시나리오에서, 다음 예제와 같이 `type` 속성을 통해 생성자 인수의 타입을 명시적으로 지정하면 컨테이너는 단순 타입과 함께 타입 매칭을 사용할 수 있습니다:

```xml
<bean id="exampleBean" class="examples.ExampleBean">
	<constructor-arg type="int" value="7500000"/>
	<constructor-arg type="java.lang.String" value="42"/>
</bean>
```

*생성자 인수 인덱스*
다음 예제와 같이 `index` 속성을 사용하여 생성자 인수의 인덱스를 명시적으로 지정할 수 있습니다:

```xml
<bean id="exampleBean" class="examples.ExampleBean">
	<constructor-arg index="0" value="7500000"/>
	<constructor-arg index="1" value="42"/>
</bean>

```

여러 단순 값의 모호성을 해결하는 것 외에도, 인덱스를 지정하면 생성자가 동일한 타입의 두 인수를 가질 때의 모호성도 해결합니다.

*인덱스는 0부터 시작합니다.*

*생성자 인수 이름*
다음 예제와 같이 값의 모호성을 해결하기 위해 생성자 파라미터 이름을 사용할 수도 있습니다:

```xml
<bean id="exampleBean" class="examples.ExampleBean">
	<constructor-arg name="years" value="7500000"/>
	<constructor-arg name="ultimateAnswer" value="42"/>
</bean>

```

이것이 기본적으로 작동하게 하려면 코드가 `-parameters` 플래그를 활성화하여 컴파일되어야 한다는 점을 명심하십시오. 그래야 스프링이 생성자에서 파라미터 이름을 조회할 수 있습니다. 코드를 `-parameters` 플래그로 컴파일할 수 없거나 원하지 않는 경우, `@ConstructorProperties` JDK 어노테이션을 사용하여 생성자 인수를 명시적으로 명명할 수 있습니다. 샘플 클래스는 다음과 같아야 합니다:

```java
// Java
package examples;

public class ExampleBean {

	// 필드 생략

	@ConstructorProperties({"years", "ultimateAnswer"})
	public ExampleBean(int years, String ultimateAnswer) {
		this.years = years;
		this.ultimateAnswer = ultimateAnswer;
	}
}
```

```kotlin
// Kotlin
package examples

import java.beans.ConstructorProperties

class ExampleBean @ConstructorProperties("years", "ultimateAnswer") constructor(
    private val years: Int,
    private val ultimateAnswer: String
) {
    // ...
}
```

**세터 기반 의존성 주입 (Setter-based Dependency Injection)**

세터 기반 DI는 컨테이너가 인수 없는 생성자 또는 인수 없는 정적 팩토리 메소드를 호출하여 빈을 인스턴스화한 후, 빈의 세터(setter) 메소드를 호출함으로써 이루어집니다.

다음 예제는 순수 세터 주입만을 사용하여 의존성 주입될 수 있는 클래스를 보여줍니다.

이 클래스는 관례적인 자바입니다. 이는 컨테이너 특정 인터페이스, 기본 클래스 또는 어노테이션에 대한 의존성이 없는 POJO입니다.

```java
// Java
public class SimpleMovieLister {

	// SimpleMovieLister는 MovieFinder에 대한 의존성을 가집니다
	private MovieFinder movieFinder;

	// 스프링 컨테이너가 MovieFinder를 주입할 수 있도록 하는 세터 메소드
	public void setMovieFinder(MovieFinder movieFinder) {
		this.movieFinder = movieFinder;
	}

	// 실제로 주입된 MovieFinder를 사용하는 비즈니스 로직은 생략...
}

```

```kotlin
// Kotlin
class SimpleMovieLister {
    // SimpleMovieLister는 MovieFinder에 대한 의존성을 가집니다
    private var movieFinder: MovieFinder? = null

    // 스프링 컨테이너가 MovieFinder를 주입할 수 있도록 하는 세터 메소드
    fun setMovieFinder(movieFinder: MovieFinder) {
        this.movieFinder = movieFinder
    }

    // 실제로 주입된 MovieFinder를 사용하는 비즈니스 로직은 생략...
}

```

`ApplicationContext`는 관리하는 빈에 대해 생성자 기반 및 세터 기반 DI를 지원합니다.

또한 일부 의존성이 이미 생성자 방식을 통해 주입된 후 세터 기반 DI도 지원합니다.

의존성을 `BeanDefinition` 형태로 설정하며, 이는 `PropertyEditor` 인스턴스와 함께 사용하여 속성을 한 형식에서 다른 형식으로 변환합니다.

그러나 대부분의 스프링 사용자는 이러한 클래스를 직접(즉, 프로그래밍 방식으로) 다루지 않고, XML 빈 정의, 어노테이션 기반 컴포넌트(즉, `@Component`, `@Controller` 등으로 어노테이션된 클래스),

또는 자바 기반 `@Configuration` 클래스의 `@Bean` 메소드를 사용합니다. 이러한 소스들은 내부적으로 `BeanDefinition`의 인스턴스로 변환되어 전체 스프링 IoC 컨테이너 인스턴스를 로드하는 데 사용됩니다.

**생성자 기반 또는 세터 기반 DI? (Constructor-based or setter-based DI?)**

생성자 기반과 세터 기반 DI를 혼합할 수 있으므로, **필수 의존성에는 생성자**를 사용하고 **선택적 의존성에는 세터 메소드 또는 설정 메소드**를 사용하는 것이 좋은 경험 법칙입니다.

세터 메소드에 `@Autowired` 어노테이션을 사용하면 해당 속성을 필수 의존성으로 만들 수 있지만, 프로그래밍 방식의 인수 유효성 검증을 동반한 생성자 주입이 더 선호됩니다.

스프링 팀은 일반적으로 **생성자 주입을 권장**하는데, 이는 애플리케이션 컴포넌트를 불변 객체(immutable objects)로 구현할 수 있게 하고 필수 의존성이 null이 아님을 보장하기 때문입니다.

더욱이, 생성자 주입된 컴포넌트는 항상 완전히 초기화된 상태로 클라이언트(호출) 코드에 반환됩니다.

참고로, 생성자 인수가 많은 것은 좋지 않은 코드 냄새(bad code smell)이며, 이는 해당 클래스가 너무 많은 책임을 가지고 있어 관심사의 적절한 분리(separation of concerns)를 위해 리팩토링되어야 함을 시사합니다.

**세터 주입은 주로 클래스 내에서 합리적인 기본값을 할당할 수 있는 선택적 의존성에만 사용해야 합니다.**

그렇지 않으면 코드가 의존성을 사용하는 모든 곳에서 null이 아닌지 확인해야 합니다.

세터 주입의 한 가지 이점은 세터 메소드가 해당 클래스의 객체를 나중에 재설정(reconfiguration)하거나 재주입(re-injection)하기에 적합하게 만든다는 것입니다.

**의존성 해석 프로세스 (Dependency Resolution Process)**

컨테이너는 다음과 같이 빈 의존성 해석을 수행합니다:

1. `ApplicationContext`는 모든 빈을 설명하는 설정 메타데이터로 생성되고 초기화됩니다. 설정 메타데이터는 XML, 자바 코드 또는 어노테이션으로 지정될 수 있습니다.
2. 각 빈에 대해, 그 의존성은 속성, 생성자 인수 또는 정적 팩토리 메소드의 인수 (일반 생성자 대신 사용하는 경우) 형태로 표현됩니다. 이러한 의존성은 빈이 실제로 생성될 때 빈에 제공됩니다.
3. 각 속성 또는 생성자 인수는 설정할 값의 실제 정의이거나 컨테이너의 다른 빈에 대한 참조입니다.
4. 값인 각 속성 또는 생성자 인수는 지정된 형식에서 해당 속성 또는 생성자 인수의 실제 타입으로 변환됩니다. 기본적으로 스프링은 문자열 형식으로 제공된 값을 `int`, `long`, `String`, `boolean` 등 모든 내장 타입으로 변환할 수 있습니다.

스프링 컨테이너는 컨테이너가 생성될 때 각 빈의 구성을 검증합니다.

그러나 빈 속성 자체는 빈이 실제로 생성될 때까지 설정되지 않습니다.

싱글톤 스코프이고 미리 인스턴스화되도록 설정된(기본값) 빈들은 컨테이너가 생성될 때 생성됩니다.

스코프는 빈 스코프(Bean Scopes)에 정의되어 있습니다.

그렇지 않으면 빈은 요청될 때만 생성됩니다.

빈의 생성은 잠재적으로 빈 그래프(graph of beans) 생성을 유발할 수 있는데, 빈의 의존성과 그 의존성의 의존성(등등)이 생성되고 할당되기 때문입니다.

이러한 의존성 간의 해석 불일치는 영향을 받는 빈의 첫 생성 시점에 늦게 나타날 수 있습니다.

*순환 의존성 (Circular dependencies)*
주로 생성자 주입을 사용하는 경우, 해결할 수 없는 순환 의존성 시나리오를 만들 수 있습니다.

예를 들어: 클래스 A는 생성자 주입을 통해 클래스 B의 인스턴스를 필요로 하고, 클래스 B는 생성자 주입을 통해 클래스 A의 인스턴스를 필요로 합니다.

클래스 A와 B에 대한 빈들이 서로 주입되도록 설정하면, 스프링 IoC 컨테이너는 런타임 시 이 순환 참조를 감지하고 `BeanCurrentlyInCreationException`을 발생시킵니다.

한 가지 가능한 해결책은 일부 클래스의 소스 코드를 편집하여 생성자 대신 세터로 설정되도록 하는 것입니다.

또는 생성자 주입을 피하고 세터 주입만 사용하는 것입니다. 즉, 권장되지는 않지만 세터 주입으로 순환 의존성을 설정할 수 있습니다.

일반적인 경우(순환 의존성이 없는)와 달리, 빈 A와 빈 B 사이의 순환 의존성은 빈 중 하나가 완전히 초기화되기 전에 다른 빈에 주입되도록 강제합니다(전형적인 닭과 달걀 시나리오).

스프링은 존재하지 않는 빈에 대한 참조 및 순환 의존성과 같은 구성 문제를 컨테이너 로드 시점에 감지합니다.

스프링은 속성을 설정하고 의존성을 가능한 한 늦게, 빈이 실제로 생성될 때 해결합니다.

이는 올바르게 로드된 스프링 컨테이너가 나중에 객체를 요청할 때 해당 객체 또는 그 의존성 중 하나를 생성하는 데 문제가 있으면 예외를 발생시킬 수 있음을 의미합니다

예를 들어, 빈이 누락되거나 잘못된 속성으로 인해 예외를 발생시키는 경우.

일부 구성 문제의 이러한 잠재적으로 지연된 가시성 때문에 `ApplicationContext` 구현체는 기본적으로 싱글톤 빈을 미리 인스턴스화합니다.

실제로 필요하기 전에 이러한 빈을 생성하는 데 약간의 초기 시간과 메모리를 소비하는 대가로, `ApplicationContext`가 생성될 때 구성 문제를 발견하게 됩니다.

여전히 이 기본 동작을 오버라이드하여 싱글톤 빈이 즉시 미리 인스턴스화되는 대신 지연 초기화(lazily initialize)되도록 할 수 있습니다.

순환 의존성이 없다면, 하나 이상의 협력 빈이 종속 빈에 주입될 때, 각 협력 빈은 종속 빈에 주입되기 전에 완전히 구성됩니다.

이는 빈 A가 빈 B에 대한 의존성을 가지고 있다면, 스프링 IoC 컨테이너는 빈 A의 세터 메소드를 호출하기 전에 빈 B를 완전히 구성한다는 것을 의미합니다.

즉, 빈이 인스턴스화되고(미리 인스턴스화된 싱글톤이 아니라면), 그 의존성이 설정되고, 관련 생명주기 메소드(예: 설정된 init 메소드 또는 `InitializingBean` 콜백 메소드)가 호출됩니다.

**의존성 주입 예제 (Examples of Dependency Injection)**

다음 예제는 세터 기반 DI를 위한 XML 기반 설정 메타데이터를 사용합니다. 스프링 XML 설정 파일의 작은 부분은 다음과 같이 일부 빈 정의를 지정합니다:

```xml
<bean id="exampleBean" class="examples.ExampleBean">
	<!-- 중첩된 ref 요소를 사용한 세터 주입 -->
	<property name="beanOne">
		<ref bean="anotherExampleBean"/>
	</property>

	<!-- 더 간결한 ref 속성을 사용한 세터 주입 -->
	<property name="beanTwo" ref="yetAnotherBean"/>
	<property name="integerProperty" value="1"/>
</bean>

<bean id="anotherExampleBean" class="examples.AnotherBean"/>
<bean id="yetAnotherBean" class="examples.YetAnotherBean"/>

```

다음 예제는 해당 `ExampleBean` 클래스를 보여줍니다:

```java
// Java
public class ExampleBean {

	private AnotherBean beanOne;
	private YetAnotherBean beanTwo;
	private int i;

	public void setBeanOne(AnotherBean beanOne) {
		this.beanOne = beanOne;
	}

	public void setBeanTwo(YetAnotherBean beanTwo) {
		this.beanTwo = beanTwo;
	}

	public void setIntegerProperty(int i) {
		this.i = i;
	}
}

```

```kotlin
// Kotlin
class ExampleBean {
    var beanOne: AnotherBean? = null
    var beanTwo: YetAnotherBean? = null
    var i: Int = 0
}

```

앞의 예제에서 세터는 XML 파일에 지정된 속성과 일치하도록 선언되었습니다. 다음 예제는 생성자 기반 DI를 사용합니다:

```xml
<bean id="exampleBean" class="examples.ExampleBean">
	<!-- 중첩된 ref 요소를 사용한 생성자 주입 -->
	<constructor-arg>
		<ref bean="anotherExampleBean"/>
	</constructor-arg>

	<!-- 더 간결한 ref 속성을 사용한 생성자 주입 -->
	<constructor-arg ref="yetAnotherBean"/>

	<constructor-arg type="int" value="1"/>
</bean>

<bean id="anotherExampleBean" class="examples.AnotherBean"/>
<bean id="yetAnotherBean" class="examples.YetAnotherBean"/>

```

다음 예제는 해당 `ExampleBean` 클래스를 보여줍니다:

```java
// Java
public class ExampleBean {

	private AnotherBean beanOne;
	private YetAnotherBean beanTwo;
	private int i;

	public ExampleBean(
		AnotherBean anotherBean, YetAnotherBean yetAnotherBean, int i) {
		this.beanOne = anotherBean;
		this.beanTwo = yetAnotherBean;
		this.i = i;
	}
}

```

```kotlin
// Kotlin
class ExampleBean(
    private val beanOne: AnotherBean,
    private val beanTwo: YetAnotherBean,
    private val i: Int
)

```

빈 정의에 지정된 생성자 인수는 `ExampleBean`의 생성자에 대한 인수로 사용됩니다.

이제 이 예제의 변형을 고려해 보겠습니다. 여기서 스프링은 생성자를 사용하는 대신 정적 팩토리 메소드를 호출하여 객체의 인스턴스를 반환하도록 지시받습니다:

```xml
<bean id="exampleBean" class="examples.ExampleBean" factory-method="createInstance">
	<constructor-arg ref="anotherExampleBean"/>
	<constructor-arg ref="yetAnotherBean"/>
	<constructor-arg value="1"/>
</bean>

<bean id="anotherExampleBean" class="examples.AnotherBean"/>
<bean id="yetAnotherBean" class="examples.YetAnotherBean"/>

```

다음 예제는 해당 `ExampleBean` 클래스를 보여줍니다:

```java
// Java
public class ExampleBean {

	// private 생성자
	private ExampleBean(...) {
		...
	}

	// 정적 팩토리 메소드; 이 메소드의 인수는 반환되는 빈의
	// 의존성으로 간주될 수 있으며, 해당 인수가 실제로 어떻게
	// 사용되는지와는 관계없습니다.
	public static ExampleBean createInstance (
		AnotherBean anotherBean, YetAnotherBean yetAnotherBean, int i) {

		ExampleBean eb = new ExampleBean (...);
		// 몇 가지 다른 작업...
		return eb;
	}
}

```

```kotlin
// Kotlin
class ExampleBean private constructor(...) { // private 생성자
    companion object {
        // 정적 팩토리 메소드; 이 메소드의 인수는 반환되는 빈의
        // 의존성으로 간주될 수 있으며, 해당 인수가 실제로 어떻게
        // 사용되는지와는 관계없습니다.
        @JvmStatic // Java에서 정적 메소드로 보이도록 함
        fun createInstance(
            anotherBean: AnotherBean,
            yetAnotherBean: YetAnotherBean,
            i: Int
        ): ExampleBean {
            val eb = ExampleBean(...)
            // 몇 가지 다른 작업...
            return eb
        }
    }
}

```

정적 팩토리 메소드에 대한 인수는 실제로 생성자가 사용된 것처럼 정확히 동일하게 `<constructor-arg/>` 요소를 통해 제공됩니다. 팩토리 메소드에 의해 반환되는 클래스의 타입은 정적 팩토리 메소드를 포함하는 클래스의 타입과 동일할 필요는 없습니다(이 예제에서는 동일하지만). 인스턴스(비정적) 팩토리 메소드는 본질적으로 동일한 방식으로 사용될 수 있으므로( `class` 속성 대신 `factory-bean` 속성을 사용하는 것 제외), 해당 세부 정보는 여기서 논의하지 않습니다.

---

생성자 기반 의존성 주입(Constructor-based Dependency Injection)과 세터 기반 의존성 주입(Setter-based Dependency Injection)은 모두 스프링에서 의존성을 주입하는 방법입니다. 두 방식은 비슷해 보일 수 있지만, 실제로는 중요한 차이점이 있습니다.

### 🔧 생성자 기반 주입 vs 세터 기반 주입

| 구분 | 생성자 기반 주입 | 세터 기반 주입 |
| --- | --- | --- |
| **주입 시점** | 객체 생성 시점에 주입 | 객체 생성 후 세터 메서드를 통해 주입 |
| **필수 의존성 보장** | 모든 필수 의존성을 생성자에서 강제 | 선택적 의존성 주입에 적합 |
| **불변성(Immutable)** | 주입된 의존성을 변경할 수 없음 | 주입된 의존성을 변경할 수 있음 |
| **테스트 용이성** | 테스트 시 의존성을 명확히 설정 가능 | 의존성 누락 시 런타임 오류 가능 |
| **순환 참조 해결** | 순환 참조 문제 발생 가능 | 순환 참조 문제 해결 가능 |

---

### ✅ 언제 어떤 방식을 사용할까?

- **생성자 주입**: 필수 의존성이 있는 경우에 사용합니다. 객체가 생성될 때 모든 필수 의존성이 주입되어야 하며, 이후 변경되지 않아야 하는 경우에 적합합니다.
- **세터 주입**: 선택적 의존성이 있는 경우에 사용합니다. 객체 생성 후에 의존성을 설정하거나 변경할 수 있어야 하는 경우에 적합합니다.

Spring Framework에서 `PropertyEditor`는 문자열(String) 기반의 설정 값을 Java 객체로 변환하거나 그 반대로 변환하는 데 사용되는 인터페이스입니다. 이 기능은 JavaBeans 사양의 일부로, 특히 스프링에서 빈(bean)의 속성을 설정할 때 유용하게 활용됩니다.

### 🔧 PropertyEditor의 역할

스프링에서는 XML 설정 파일이나 애플리케이션 속성 파일에서 빈의 속성을 문자열로 지정하는 경우가 많습니다. 예를 들어:

```xml
xml
복사편집
<bean id="exampleBean" class="com.example.ExampleBean">
    <property name="someProperty" value="2025-05-01"/>
</bean>

```

이때, `someProperty`가 `java.util.Date` 타입이라면, 스프링은 이 문자열을 `Date` 객체로 변환해야 합니다. 이러한 변환 작업을 수행하는 것이 바로 `PropertyEditor`의 역할입니다.

---

### 🛠️ 기본 제공 PropertyEditor 예시

스프링은 다양한 기본 `PropertyEditor`를 제공합니다:

- **CustomDateEditor**: 문자열을 `Date` 객체로 변환합니다.
- **CustomNumberEditor**: 문자열을 `Integer`, `Long` 등의 숫자 타입으로 변환합니다.
- **ClassEditor**: 문자열을 `Class` 객체로 변환합니다.
- **FileEditor**: 문자열을 `File` 객체로 변환합니다.

이러한 편집기들은 스프링 컨테이너가 빈의 속성을 설정할 때 자동으로 사용됩니다.

---

### 🧩 커스텀 PropertyEditor 구현

특정한 변환 로직이 필요한 경우, 사용자 정의 `PropertyEditor`를 구현할 수 있습니다. 예를 들어, `ExoticType`이라는 사용자 정의 타입을 처리하려면:

```java
java
복사편집
public class ExoticTypeEditor extends PropertyEditorSupport {
    @Override
    public void setAsText(String text) throws IllegalArgumentException {
        ExoticType exoticType = new ExoticType(text);
        setValue(exoticType);
    }
}

```

그리고 이를 스프링 설정에 등록합니다:

```xml
xml
복사편집
<bean class="org.springframework.beans.factory.config.CustomEditorConfigurer">
    <property name="customEditors">
        <map>
            <entry key="com.example.ExoticType" value="com.example.ExoticTypeEditor"/>
        </map>
    </property>
</bean>

```

이렇게 하면 스프링은 `ExoticType` 속성을 설정할 때 `ExoticTypeEditor`를 사용하여 문자열을 객체로 변환합니다.[Medium](https://medium.com/%40ivan.tsupa111/enhancing-spring-applications-with-custom-property-editors-8a0af98a5f93?utm_source=chatgpt.com)

---

### 🔄 PropertyEditor와 Converter의 차이점

스프링 3 이후에는 `PropertyEditor`보다 더 유연한 `Converter` 인터페이스가 도입되었습니다. `Converter`는 제네릭을 사용하여 타입 안전성을 제공하며, 스레드에 안전합니다. 따라서 새로운 프로젝트에서는 `PropertyEditor`보다는 `Converter`를 사용하는 것이 권장됩니다.

---

### ✅ 요약

- **`PropertyEditor`**: 문자열과 객체 간의 변환을 처리합니다.
- **기본 제공 편집기**: 날짜, 숫자, 클래스 등 다양한 타입을 지원합니다.
- **커스텀 편집기**: 특정한 변환이 필요한 경우 사용자 정의 편집기를 구현하여 등록할 수 있습니다.
- **`Converter`**: 스프링 3 이후 도입된 더 유연하고 타입 안전한 변환 메커니즘입니다.

이러한 메커니즘을 통해 스프링은 다양한 타입의 속성을 유연하게 처리할 수 있습니다.
