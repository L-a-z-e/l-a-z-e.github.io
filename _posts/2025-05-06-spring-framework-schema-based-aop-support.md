---
title: Schema-based AOP Support
description: 
author: laze
date: 2025-05-06 00:00:01 +0900
categories: [Dev, SpringBoot]
tags: [SpringBoot]
---
**Schema-based AOP Support**

XML 기반 형식을 선호하는 경우, 스프링은 `aop` 네임스페이스 태그를 사용하여 애스펙트(aspects)를 정의하는 지원도 제공합니다.

`@AspectJ` 스타일 사용 시와 정확히 동일한 포인트컷 표현식 및 어드바이스 종류가 지원됩니다.

스프링 구성 내에서 모든 애스펙트 및 어드바이저(advisor) 요소는 `<aop:config>` 요소 내에 배치되어야 합니다 (애플리케이션 컨텍스트 구성에 둘 이상의 `<aop:config>` 요소가 있을 수 있음).

`<aop:config>` 요소는 `pointcut`, `advisor`, `aspect` 요소를 포함할 수 있습니다 (이 순서대로 선언되어야 함에 유의).

`*<aop:config>` 스타일 구성은 스프링의 자동 프록시(auto-proxying) 메커니즘을 많이 사용합니다.*

`*BeanNameAutoProxyCreator` 또는 이와 유사한 것을 사용하여 명시적 자동 프록시를 이미 사용하고 있는 경우, 이로 인해 문제(예: 어드바이스가 위빙되지 않음)가 발생할 수 있습니다.*

*권장되는 사용 패턴은 `<aop:config>` 스타일만 사용하거나 `AutoProxyCreator` 스타일만 사용하고 절대 혼합하지 않는 것입니다.*

**애스펙트 선언하기 (Declaring an Aspect)**

스키마 지원을 사용할 때, 애스펙트는 스프링 애플리케이션 컨텍스트에서 빈으로 정의된 일반 자바 객체입니다. 상태와 동작은 객체의 필드와 메소드에 캡처되고, 포인트컷과 어드바이스 정보는 XML에 캡처됩니다.

`<aop:aspect>` 요소를 사용하여 애스펙트를 선언하고, 다음 예제와 같이 `ref` 속성을 사용하여 백킹 빈(backing bean)을 참조할 수 있습니다:

```xml
<aop:config>
	<aop:aspect id="myAspect" ref="aBean"> <!-- 'aBean'이라는 이름의 빈을 애스펙트로 사용 -->
		...
	</aop:aspect>
</aop:config>

<bean id="aBean" class="..."> <!-- 애스펙트 로직을 포함하는 실제 빈 -->
	...
</bean>
```

애스펙트를 지원하는 빈(`aBean`, 이 경우)은 물론 다른 스프링 빈처럼 구성되고 의존성 주입될 수 있습니다.

**포인트컷 선언하기 (Declaring a Pointcut)**

`<aop:config>` 요소 내에 이름 붙여진 포인트컷(named pointcut)을 선언하여 여러 애스펙트와 어드바이저 간에 포인트컷 정의를 공유할 수 있습니다.

서비스 계층의 모든 비즈니스 서비스 실행을 나타내는 포인트컷은 다음과 같이 정의될 수 있습니다:

```xml
<aop:config>

	<aop:pointcut id="businessService" <!-- 포인트컷 ID -->
		expression="execution(* com.xyz.service.*.*(..))" /> <!-- AspectJ 표현식 -->

</aop:config>
```

포인트컷 표현식 자체는 `@AspectJ` 지원에서 설명된 것과 동일한 AspectJ 포인트컷 표현식 언어를 사용한다는 점에 유의하십시오.

스키마 기반 선언 스타일을 사용하는 경우, 포인트컷 표현식 내에서 `@Aspect` 타입에 정의된 이름 붙여진 포인트컷을 참조할 수도 있습니다.

따라서 위 포인트컷을 정의하는 또 다른 방법은 다음과 같습니다:

```xml
<aop:config>

	<aop:pointcut id="businessService"
		expression="com.xyz.CommonPointcuts.businessService()" /> <!-- ① -->

</aop:config>
```

① `Sharing Named Pointcut Definitions`에서 정의된 `businessService` 이름 붙여진 포인트컷을 참조합니다.

애스펙트 내부에 포인트컷을 선언하는 것은 다음 예제와 같이 최상위 레벨 포인트컷을 선언하는 것과 매우 유사합니다:

```xml
<aop:config>

	<aop:aspect id="myAspect" ref="aBean">

		<aop:pointcut id="businessService" <!-- 애스펙트 내 포인트컷 -->
			expression="execution(* com.xyz.service.*.*(..))"/>

		...
	</aop:aspect>

</aop:config>
```

`@AspectJ` 애스펙트와 거의 동일한 방식으로, 스키마 기반 정의 스타일을 사용하여 선언된 포인트컷은 조인 포인트 컨텍스트를 수집할 수 있습니다.

예를 들어, 다음 포인트컷은 `this` 객체를 조인 포인트 컨텍스트로 수집하고 어드바이스에 전달합니다:

```xml
<aop:config>

	<aop:aspect id="myAspect" ref="aBean">

		<aop:pointcut id="businessService"
			expression="execution(* com.xyz.service.*.*(..)) &amp;&amp; this(service)"/> <!-- this(service)로 바인딩 -->

		<aop:before pointcut-ref="businessService" method="monitor"/> <!-- 메소드에 파라미터 필요 -->

		...
	</aop:aspect>

</aop:config>
```

어드바이스는 다음과 같이 일치하는 이름의 파라미터를 포함하여 수집된 조인 포인트 컨텍스트를 받도록 선언되어야 합니다:

```java
// Java
public void monitor(Object service) { // 'service' 파라미터로 this 객체 수신
	// ...
}
```

```kotlin
// Kotlin
fun monitor(service: Any) { // 'service' 파라미터로 this 객체 수신
    // ...
}
```

포인트컷 하위 표현식을 결합할 때 `&amp;&amp;`는 XML 문서 내에서 번거로우므로, 각각 `&&`, `||`, `!` 대신 `and`, `or`, `not` 키워드를 사용할 수 있습니다.

예를 들어, 이전 포인트컷은 다음과 같이 더 잘 작성될 수 있습니다:

```xml
<aop:config>

	<aop:aspect id="myAspect" ref="aBean">

		<aop:pointcut id="businessService"
			expression="execution(* com.xyz.service.*.*(..)) and this(service)"/> <!-- 'and' 키워드 사용 -->

		<aop:before pointcut-ref="businessService" method="monitor"/>

		...
	</aop:aspect>

</aop:config>
```

*이런 방식으로 정의된 포인트컷은 XML id로 참조되며 복합 포인트컷을 형성하기 위한 이름 붙여진 포인트컷으로 사용될 수 없다는 점에 유의하십시오.*

*따라서 스키마 기반 정의 스타일의 이름 붙여진 포인트컷 지원은 `@AspectJ` 스타일에서 제공하는 것보다 더 제한적입니다.*

**어드바이스 선언하기 (Declaring Advice)**

스키마 기반 AOP 지원은 `@AspectJ` 스타일과 동일한 다섯 가지 종류의 어드바이스를 사용하며, 정확히 동일한 의미론을 가집니다.

*Before 어드바이스*
Before 어드바이스는 매치된 메소드 실행 전에 실행됩니다. 다음 예제와 같이 `<aop:aspect>` 내에서 `<aop:before>` 요소를 사용하여 선언됩니다:

```xml
<aop:aspect id="beforeExample" ref="aBean">

	<aop:before
		pointcut-ref="dataAccessOperation" <!-- 이름 붙여진 포인트컷 참조 -->
		method="doAccessCheck"/> <!-- 실행할 어드바이스 메소드 -->

	...

</aop:aspect>
```

위 예제에서 `dataAccessOperation`은 최상위 (`<aop:config>`) 레벨에서 정의된 이름 붙여진 포인트컷의 id입니다 (포인트컷 선언하기 참조).

`*@AspectJ` 스타일 논의에서 언급했듯이, 이름 붙여진 포인트컷을 사용하면 코드의 가독성을 크게 향상시킬 수 있습니다.*

포인트컷을 인라인으로 정의하려면 `pointcut-ref` 속성을 `pointcut` 속성으로 바꿉니다.

```xml
<aop:aspect id="beforeExample" ref="aBean">

	<aop:before
		pointcut="execution(* com.xyz.dao.*.*(..))" <!-- 인라인 포인트컷 -->
		method="doAccessCheck"/>

	...

</aop:aspect>
```

`method` 속성은 어드바이스의 본문을 제공하는 메소드(`doAccessCheck`)를 식별합니다. 이 메소드는 어드바이스를 포함하는 aspect 요소가 참조하는 빈에 대해 정의되어야 합니다.

데이터 접근 작업(포인트컷 표현식과 일치하는 메소드 실행 조인 포인트)이 수행되기 전에, 애스펙트 빈의 `doAccessCheck` 메소드가 호출됩니다.

*After Returning 어드바이스*
After returning 어드바이스는 매치된 메소드 실행이 정상적으로 완료될 때 실행됩니다.

`<aop:aspect>` 내에서 before 어드바이스와 동일한 방식으로 선언됩니다. 다음 예제는 선언 방법을 보여줍니다:

```xml
<aop:aspect id="afterReturningExample" ref="aBean">

	<aop:after-returning
		pointcut="execution(* com.xyz.dao.*.*(..))"
		method="doAccessCheck"/>

	...
</aop:aspect>
```

`@AspectJ` 스타일에서와 마찬가지로 어드바이스 본문 내에서 반환 값을 얻을 수 있습니다.

그렇게 하려면 다음 예제와 같이 반환 값이 전달되어야 하는 파라미터의 이름을 지정하기 위해 `returning` 속성을 사용합니다:

```xml
<aop:aspect id="afterReturningExample" ref="aBean">

	<aop:after-returning
		pointcut="execution(* com.xyz.dao.*.*(..))"
		returning="retVal" <!-- 반환 값 바인딩 -->
		method="doAccessCheck"/>

	...
</aop:aspect>
```

`doAccessCheck` 메소드는 `retVal`이라는 이름의 파라미터를 선언해야 합니다.

이 파라미터의 타입은 `@AfterReturning`에 대해 설명된 것과 동일한 방식으로 매칭을 제한합니다. 예를 들어, 메소드 시그니처를 다음과 같이 선언할 수 있습니다:

```java
// Java
public void doAccessCheck(Object retVal) { // 'retVal' 파라미터로 반환 값 수신
    // ...
}
```

```kotlin
// Kotlin
fun doAccessCheck(retVal: Any?) { // 'retVal' 파라미터로 반환 값 수신
    // ...
}
```

*After Throwing 어드바이스*
After throwing 어드바이스는 매치된 메소드 실행이 예외를 던져 종료될 때 실행됩니다.

`<aop:aspect>` 내에서 `after-throwing` 요소를 사용하여 선언됩니다.

```xml
<aop:aspect id="afterThrowingExample" ref="aBean">

	<aop:after-throwing
		pointcut="execution(* com.xyz.dao.*.*(..))"
		method="doRecoveryActions"/>

	...
</aop:aspect>
```

`@AspectJ` 스타일에서와 마찬가지로 어드바이스 본문 내에서 던져진 예외를 얻을 수 있습니다.

그렇게 하려면 다음 예제와 같이 예외가 전달되어야 하는 파라미터의 이름을 지정하기 위해 `throwing` 속성을 사용합니다:

```xml
<aop:aspect id="afterThrowingExample" ref="aBean">

	<aop:after-throwing
		pointcut="execution(* com.xyz.dao.*.*(..))"
		throwing="dataAccessEx" <!-- 예외 바인딩 -->
		method="doRecoveryActions"/>

	...
</aop:aspect>
```

`doRecoveryActions` 메소드는 `dataAccessEx`라는 이름의 파라미터를 선언해야 합니다.

이 파라미터의 타입은 `@AfterThrowing`에 대해 설명된 것과 동일한 방식으로 매칭을 제한합니다.

```java
// Java
import org.springframework.dao.DataAccessException;

public void doRecoveryActions(DataAccessException dataAccessEx) { // 'dataAccessEx' 파라미터로 예외 수신
    // ...
}
```

```kotlin
// Kotlin
import org.springframework.dao.DataAccessException

fun doRecoveryActions(dataAccessEx: DataAccessException) { // 'dataAccessEx' 파라미터로 예외 수신
    // ...
}
```

*After (Finally) 어드바이스*
After (finally) 어드바이스는 매치된 메소드 실행이 어떻게 종료되든 실행됩니다.

`after` 요소를 사용하여 선언할 수 있습니다. 다음 예제를 참조하십시오:

```xml
<aop:aspect id="afterFinallyExample" ref="aBean">

	<aop:after <!-- finally 블록처럼 항상 실행 -->
		pointcut="execution(* com.xyz.dao.*.*(..))"
		method="doReleaseLock"/>

	...
</aop:aspect>

```

*Around 어드바이스*
마지막 종류의 어드바이스는 around 어드바이스입니다.

Around 어드바이스는 매치된 메소드 실행 "주위(around)"에서 실행됩니다.

메소드 실행 전후에 작업을 수행하고, 메소드가 실제로 실행될 시기, 방법, 심지어 실행 여부까지 결정할 기회를 가집니다.

Around 어드바이스는 스레드 안전 방식으로 메소드 실행 전후에 상태를 공유해야 하는 경우(예: 타이머 시작 및 중지) 종종 사용됩니다.

*항상 요구 사항을 충족하는 가장 덜 강력한 형태의 어드바이스를 사용하십시오.예를 들어, before 어드바이스가 요구 사항에 충분하다면 around 어드바이스를 사용하지 마십시오.*

`aop:around` 요소를 사용하여 around 어드바이스를 선언할 수 있습니다.

어드바이스 메소드는 반환 타입으로 `Object`를 선언해야 하며, 메소드의 첫 번째 파라미터는 `ProceedingJoinPoint` 타입이어야 합니다.

어드바이스 메소드 본문 내에서 기본 메소드가 실행되도록 하려면 `ProceedingJoinPoint`에서 `proceed()`를 호출해야 합니다.

인수를 사용하지 않고 `proceed()`를 호출하면 호출자의 원본 인수가 호출될 때 기본 메소드에 제공됩니다.

고급 사용 사례를 위해, 인수 배열(`Object[]`)을 받는 `proceed()` 메소드의 오버로드된 변형이 있습니다.

배열의 값은 호출될 때 기본 메소드의 인수로 사용됩니다. `Object[]`로 `proceed` 호출에 대한 참고 사항은 Around 어드바이스를 참조하십시오.

다음 예제는 XML에서 around 어드바이스를 선언하는 방법을 보여줍니다:

```xml
<aop:aspect id="aroundExample" ref="aBean">

	<aop:around
		pointcut="execution(* com.xyz.service.*.*(..))"
		method="doBasicProfiling"/> <!-- Around 어드바이스 메소드 지정 -->

	...
</aop:aspect>
```

`doBasicProfiling` 어드바이스의 구현은 다음 예제와 같이 `@AspectJ` 예제와 정확히 동일할 수 있습니다 (물론 어노테이션 제외):

```java
// Java
import org.aspectj.lang.ProceedingJoinPoint;
import org.springframework.util.StopWatch; // Example for timing

public Object doBasicProfiling(ProceedingJoinPoint pjp) throws Throwable {
	// 타이머 시작
    StopWatch sw = new StopWatch(getClass().getSimpleName());
    try {
        sw.start(pjp.getSignature().getName());
        Object retVal = pjp.proceed(); // 원래 메소드 실행 진행
        return retVal;
    } finally {
        sw.stop();
        System.out.println(sw.prettyPrint()); // 타이머 중지 및 출력
    }
}
```

```kotlin
// Kotlin
import org.aspectj.lang.ProceedingJoinPoint
import org.springframework.util.StopWatch // Example for timing

@Throws(Throwable::class) // Declare potential exceptions
fun doBasicProfiling(pjp: ProceedingJoinPoint): Any? { // Return Any?
    // 타이머 시작
    val sw = StopWatch(javaClass.simpleName)
    return try {
        sw.start(pjp.signature.name)
        pjp.proceed() // 원래 메소드 실행 진행
    } finally {
        sw.stop()
        println(sw.prettyPrint()) // 타이머 중지 및 출력
    }
}
```

**어드바이스 파라미터 (Advice Parameters)**

스키마 기반 선언 스타일은 `@AspectJ` 지원에 대해 설명된 것과 동일한 방식으로 완전한 타입 지정 어드바이스를 지원합니다 - 포인트컷 파라미터를 이름별로 어드바이스 메소드 파라미터와 매칭

어드바이스 메소드에 대한 인수 이름을 명시적으로 지정하려면 (앞서 설명한 탐지 전략에 의존하지 않고), 어드바이스 요소의 `arg-names` 속성을 사용할 수 있으며,

이는 어드바이스 어노테이션의 `argNames` 속성과 동일한 방식으로 처리됩니다 (인수 이름 결정하기(Determining Argument Names)에서 설명됨).

```xml
<aop:before
	pointcut="com.xyz.Pointcuts.publicMethod() and @annotation(auditable)" <!-- ① -->
	method="audit"
	arg-names="auditable" /> <!-- ② 인수 이름 명시 -->
```

① `Combining Pointcut Expressions`에서 정의된 `publicMethod` 이름 붙여진 포인트컷을 참조합니다.
② `arg-names` 속성은 쉼표로 구분된 파라미터 이름 목록을 받습니다.

다음 XSD 기반 접근 방식의 약간 더 복잡한 예제는 여러 강력한 타입 지정 파라미터와 함께 사용되는 일부 around 어드바이스를 보여줍니다:

```java
// Java - Service Interface and Implementation
package com.xyz.service;
// Assuming Person class exists
class Person {
    String name; int age;
    Person(String n, int a) { name=n; age=a;}
    // getters...
}

public interface PersonService {
	Person getPerson(String personName, int age);
}

public class DefaultPersonService implements PersonService {
	@Override
	public Person getPerson(String name, int age) {
		return new Person(name, age);
	}
}
```

```kotlin
// Kotlin - Service Interface and Implementation
package com.xyz.service
// Assuming Person data class exists
data class Person(val name: String, val age: Int)

interface PersonService {
    fun getPerson(personName: String, age: Int): Person
}

class DefaultPersonService : PersonService {
    override fun getPerson(name: String, age: Int): Person {
        return Person(name, age)
    }
}
```

다음은 애스펙트입니다. `profile(..)` 메소드가 여러 강력한 타입 지정 파라미터를 받습니다.

첫 번째는 메소드 호출을 진행하는 데 사용되는 조인 포인트입니다.

이 파라미터의 존재는 다음 예제에서 보여주듯이 `profile(..)`이 around 어드바이스로 사용될 것임을 나타냅니다:

```java
// Java - Profiling Aspect
package com.xyz;

import org.aspectj.lang.ProceedingJoinPoint;
import org.springframework.util.StopWatch;

public class SimpleProfiler {

	// Advice method receiving join point and bound arguments
	public Object profile(ProceedingJoinPoint call, String name, int age) throws Throwable {
		StopWatch clock = new StopWatch("Profiling for '" + name + "' and '" + age + "'");
		try {
			clock.start(call.toShortString());
			return call.proceed(); // Proceed with original call
		} finally {
			clock.stop();
			System.out.println(clock.prettyPrint());
		}
	}
}
```

```kotlin
// Kotlin - Profiling Aspect
package com.xyz

import org.aspectj.lang.ProceedingJoinPoint
import org.springframework.util.StopWatch

class SimpleProfiler {

    // Advice method receiving join point and bound arguments
    @Throws(Throwable::class)
    fun profile(call: ProceedingJoinPoint, name: String, age: Int): Any? {
        val clock = StopWatch("Profiling for '$name' and '$age'")
        return try {
            clock.start(call.toShortString())
            call.proceed() // Proceed with original call
        } finally {
            clock.stop()
            println(clock.prettyPrint())
        }
    }
}
```

마지막으로, 다음 예제 XML 구성은 특정 조인 포인트에 대해 앞의 어드바이스 실행을 수행합니다:

```xml
<beans xmlns="<http://www.springframework.org/schema/beans>"
	xmlns:xsi="<http://www.w3.org/2001/XMLSchema-instance>"
	xmlns:aop="<http://www.springframework.org/schema/aop>"
	xsi:schemaLocation="
		<http://www.springframework.org/schema/beans>
		<https://www.springframework.org/schema/beans/spring-beans.xsd>
		<http://www.springframework.org/schema/aop>
		<https://www.springframework.org/schema/aop/spring-aop.xsd>">

	<!-- 이것은 스프링 AOP 인프라에 의해 프록시될 객체입니다 -->
	<bean id="personService" class="com.xyz.service.DefaultPersonService"/>

	<!-- 이것은 실제 어드바이스 자체입니다 -->
	<bean id="profiler" class="com.xyz.SimpleProfiler"/>

	<aop:config>
		<aop:aspect ref="profiler"> <!-- Reference the profiler bean -->

			<!-- Pointcut matching getPerson(String, int) and binding arguments -->
			<aop:pointcut id="theExecutionOfSomePersonServiceMethod"
				expression="execution(* com.xyz.service.PersonService.getPerson(String,int))
				and args(name, age)"/> <!-- Bind args to 'name' and 'age' -->

			<!-- Apply around advice, referencing the pointcut -->
			<aop:around pointcut-ref="theExecutionOfSomePersonServiceMethod"
				method="profile"/> <!-- Advice method to call -->

		</aop:aspect>
	</aop:config>

</beans>
```

```java
// Java
import org.springframework.context.ApplicationContext;
import org.springframework.context.support.ClassPathXmlApplicationContext;
import com.xyz.service.PersonService; // Assuming PersonService is in this package

public class Boot {

	public static void main(String[] args) {
		ApplicationContext ctx = new ClassPathXmlApplicationContext("beans.xml"); // Load the XML config
		PersonService person = ctx.getBean(PersonService.class);
		person.getPerson("Pengo", 12); // Call the advised method
	}
}
```

```kotlin
// Kotlin
import org.springframework.context.support.ClassPathXmlApplicationContext
import com.xyz.service.PersonService // Assuming PersonService is in this package

fun main(args: Array<String>) {
    val ctx = ClassPathXmlApplicationContext("beans.xml") // Load the XML config
    val person = ctx.getBean(PersonService::class.java)
    person.getPerson("Pengo", 12) // Call the advised method
}

```

이러한 `Boot` 클래스를 사용하면 표준 출력에서 다음과 유사한 출력을 얻을 수 있습니다:

```
StopWatch 'Profiling for 'Pengo' and '12'': running time (millis) = 0
-----------------------------------------
ms     %     Task name
-----------------------------------------
00000  ?%    execution(getPerson)

```

*어드바이스 순서 지정 (Advice Ordering)*
여러 어드바이스 조각이 동일한 조인 포인트(실행 중인 메소드)에서 실행되어야 할 때 순서 지정 규칙은 어드바이스 순서 지정(Advice Ordering)에서 설명된 대로입니다.

애스펙트 간의 우선순위는 `<aop:aspect>` 요소의 `order` 속성을 통해 또는 애스펙트를 지원하는 빈에 `@Order` 어노테이션을 추가하거나 빈이 `Ordered` 인터페이스를 구현하도록 하여 결정됩니다.

*동일한 `@Aspect` 클래스에 정의된 어드바이스 메소드의 우선순위 규칙과 대조적으로, 동일한 `<aop:aspect>` 요소에 정의된 두 개의 어드바이스 조각이 모두 동일한 조인 포인트에서 실행되어야 할 때,*

*우선순위는 포함하는 `<aop:aspect>` 요소 내에서 어드바이스 요소가 선언된 순서(가장 높은 우선순위에서 가장 낮은 우선순위로)에 의해 결정됩니다.*

*예를 들어, 동일한 조인 포인트에 적용되는 동일한 `<aop:aspect>` 요소에 정의된 around 어드바이스와 before 어드바이스가 주어진 경우,*

*around 어드바이스가 before 어드바이스보다 높은 우선순위를 갖도록 하려면 `<aop:around>` 요소가 `<aop:before>` 요소보다 먼저 선언되어야 합니다.*

*일반적으로 동일한 조인 포인트에 적용되는 동일한 `<aop:aspect>` 요소에 여러 어드바이스 조각이 정의되어 있는 경우,*

*각 `<aop:aspect>` 요소 내의 조인 포인트당 하나의 어드바이스 메소드로 이러한 어드바이스 메소드를 축소하거나 애스펙트 레벨에서 순서를 지정할 수 있는 별도의 `<aop:aspect>` 요소로 어드바이스 조각을 리팩토링하는 것을 고려하십시오.*

*인트로덕션 (Introductions)*
인트로덕션(AspectJ에서는 타입 간 선언으로 알려짐)은 애스펙트가 어드바이스받는 객체들이 주어진 인터페이스를 구현한다고 선언하고 해당 객체들을 대신하여 그 인터페이스의 구현을 제공할 수 있게 합니다.

`<aop:aspect>` 내의 `aop:declare-parents` 요소를 사용하여 인트로덕션을 만들 수 있습니다.

`aop:declare-parents` 요소를 사용하여 매칭되는 타입들이 새로운 부모를 가진다고 선언할 수 있습니다(그래서 이름이 이렇습니다).

예를 들어, `UsageTracked`라는 이름의 인터페이스와 `DefaultUsageTracked`라는 이름의 해당 인터페이스 구현이 주어졌을 때,

다음 애스펙트는 서비스 인터페이스의 모든 구현자들이 `UsageTracked` 인터페이스도 구현한다고 선언합니다. (예를 들어 JMX를 통해 통계를 노출하기 위해.)

```xml
<aop:aspect id="usageTrackerAspect" ref="usageTracking"> <!-- Reference the bean containing advice -->

	<aop:declare-parents
		types-matching="com.xyz.service.*+" <!-- Target types -->
		implement-interface="com.xyz.service.tracking.UsageTracked" <!-- Interface to introduce -->
		default-impl="com.xyz.service.tracking.DefaultUsageTracked"/> <!-- Implementation class -->

	<aop:before
		pointcut="execution(* com.xyz..service.*.*(..))
			and this(usageTracked)" <!-- Pointcut + bind 'this' as UsageTracked -->
			method="recordUsage"/> <!-- Advice method to call -->

</aop:aspect>
```

`usageTracking` 빈을 지원하는 클래스는 다음 메소드를 포함합니다:

```java
// Java
import com.xyz.service.tracking.UsageTracked; // Assuming interface exists

public void recordUsage(UsageTracked usageTracked) { // Receive the introduced interface
	usageTracked.incrementUseCount(); // Call the method from the introduced interface
}
```

```kotlin
// Kotlin
import com.xyz.service.tracking.UsageTracked // Assuming interface exists

fun recordUsage(usageTracked: UsageTracked) { // Receive the introduced interface
    usageTracked.incrementUseCount() // Call the method from the introduced interface
}
```

구현될 인터페이스는 `implement-interface` 속성에 의해 결정됩니다. `types-matching` 속성의 값은 AspectJ 타입 패턴입니다. 매칭되는 타입의 모든 빈은 `UsageTracked` 인터페이스를 구현합니다. 앞의 예제의 before 어드바이스에서 서비스 빈이 `UsageTracked` 인터페이스의 구현으로 직접 사용될 수 있다는 점에 유의하십시오. 빈에 프로그래밍 방식으로 접근하려면 다음을 작성할 수 있습니다:

```java
// Java
UsageTracked usageTracked = context.getBean("myService", UsageTracked.class);
```

```kotlin
// Kotlin
val usageTracked = context.getBean("myService", UsageTracked::class.java)
```

*애스펙트 인스턴스화 모델 (Aspect Instantiation Models)*
스키마 정의 애스펙트에 대해 지원되는 유일한 인스턴스화 모델은 싱글톤 모델입니다.

**어드바이저 (Advisors)**

"어드바이저(advisors)" 개념은 스프링에 정의된 AOP 지원에서 비롯되었으며 AspectJ에는 직접적인 등가물이 없습니다.

어드바이저는 단일 어드바이스 조각을 가진 작은 독립적인 애스펙트와 같습니다.

어드바이스 자체는 빈으로 표현되며 스프링의 어드바이스 타입(Advice Types in Spring)에 설명된 어드바이스 인터페이스 중 하나를 구현해야 합니다.

어드바이저는 AspectJ 포인트컷 표현식을 활용할 수 있습니다.

스프링은 `<aop:advisor>` 요소로 어드바이저 개념을 지원합니다.

가장 일반적으로 트랜잭션 어드바이스와 함께 사용되는 것을 볼 수 있으며, 이는 스프링에서 자체 네임스페이스 지원도 가지고 있습니다. 다음 예제는 어드바이저를 보여줍니다:

```xml
<aop:config>

	<aop:pointcut id="businessService"
		expression="execution(* com.xyz.service.*.*(..))"/>

	<aop:advisor
		pointcut-ref="businessService" <!-- Reference pointcut -->
		advice-ref="tx-advice" /> <!-- Reference advice bean -->

</aop:config>

<!-- Transactional advice definition -->
<tx:advice id="tx-advice">
	<tx:attributes>
		<tx:method name="*" propagation="REQUIRED"/>
	</tx:attributes>
</tx:advice>
```

앞의 예제에서 사용된 `pointcut-ref` 속성 외에도, `pointcut` 속성을 사용하여 포인트컷 표현식을 인라인으로 정의할 수도 있습니다.

어드바이스가 순서 지정에 참여할 수 있도록 어드바이저의 우선순위를 정의하려면, `order` 속성을 사용하여 어드바이저의 `Ordered` 값을 정의합니다.

**AOP 스키마 예제 (An AOP Schema Example)**

이 섹션은 스키마 지원으로 다시 작성했을 때 AOP 예제(An AOP Example)의 동시 잠금 실패 재시도 예제가 어떻게 보이는지 보여줍니다.

비즈니스 서비스 실행은 때때로 동시성 문제(예: 데드락 패자)로 인해 실패할 수 있습니다. 작업을 재시도하면 다음 시도에 성공할 가능성이 높습니다.

이러한 조건에서 재시도하는 것이 적절한 비즈니스 서비스(충돌 해결을 위해 사용자에게 다시 돌아갈 필요가 없는 멱등 작업)의 경우, 클라이언트가 `PessimisticLockingFailureException`을 보지 않도록 작업을 투명하게 재시도하고 싶습니다.

이는 서비스 계층의 여러 서비스에 명확하게 걸쳐 있는 요구 사항이므로, 애스펙트를 통해 구현하기에 이상적입니다.

작업을 재시도하고 싶기 때문에, `proceed`를 여러 번 호출할 수 있도록 around 어드바이스를 사용해야 합니다.

다음 목록은 기본적인 애스펙트 구현(스키마 지원을 사용하는 일반 자바 클래스)을 보여줍니다:

```java
// Java (Same class as before, just used with XML config)
import org.aspectj.lang.ProceedingJoinPoint;
import org.springframework.core.Ordered;
import org.springframework.dao.PessimisticLockingFailureException;

public class ConcurrentOperationExecutor implements Ordered {

	private static final int DEFAULT_MAX_RETRIES = 2;
	private int maxRetries = DEFAULT_MAX_RETRIES;
	private int order = 1;

	public void setMaxRetries(int maxRetries) { this.maxRetries = maxRetries; }
	public int getOrder() { return this.order; }
	public void setOrder(int order) { this.order = order; }

	public Object doConcurrentOperation(ProceedingJoinPoint pjp) throws Throwable {
		int numAttempts = 0;
		PessimisticLockingFailureException lockFailureException = null;
		do {
			numAttempts++;
			try {
				return pjp.proceed();
			}
			catch(PessimisticLockingFailureException ex) {
				lockFailureException = ex;
			}
		} while(numAttempts <= this.maxRetries);
		throw lockFailureException;
	}
}
```

```kotlin
// Kotlin (Same class as before, just used with XML config)
import org.aspectj.lang.ProceedingJoinPoint
import org.springframework.core.Ordered
import org.springframework.dao.PessimisticLockingFailureException

class ConcurrentOperationExecutor : Ordered {
    companion object { private const val DEFAULT_MAX_RETRIES = 2 }
    var maxRetries: Int = DEFAULT_MAX_RETRIES
    private var _order: Int = 1
    override fun getOrder(): Int = this._order
    fun setOrder(order: Int) { this._order = order }

    @Throws(Throwable::class)
    fun doConcurrentOperation(pjp: ProceedingJoinPoint): Any? {
        var numAttempts = 0
        var lockFailureException: PessimisticLockingFailureException? = null
        do {
            numAttempts++
            try {
                return pjp.proceed()
            } catch (ex: PessimisticLockingFailureException) {
                lockFailureException = ex
            }
        } while (numAttempts <= this.maxRetries)
        throw lockFailureException ?: RuntimeException("Retry failed")
    }
}
```

애스펙트가 `Ordered` 인터페이스를 구현하여 트랜잭션 어드바이스보다 애스펙트의 우선순위를 높게 설정할 수 있다는 점에 유의하십시오(재시도할 때마다 새로운 트랜잭션을 원함).

`maxRetries`와 `order` 속성은 모두 스프링에 의해 구성됩니다. 주요 작업은 `doConcurrentOperation` around 어드바이스 메소드에서 발생합니다. 진행(proceed)을 시도합니다.

`PessimisticLockingFailureException`으로 실패하면, 모든 재시도 횟수를 소진하지 않은 한 다시 시도합니다.

*이 클래스는 `@AspectJ` 예제에서 사용된 것과 동일하지만 어노테이션이 제거되었습니다.*

해당 스프링 구성은 다음과 같습니다:

```xml
<aop:config>

	<aop:aspect id="concurrentOperationRetry" ref="concurrentOperationExecutor"> <!-- Reference the bean -->

		<!-- Define the pointcut -->
		<aop:pointcut id="idempotentOperation"
			expression="execution(* com.xyz.service.*.*(..))"/> <!-- Apply to all service methods initially -->

		<!-- Apply around advice -->
		<aop:around
			pointcut-ref="idempotentOperation"
			method="doConcurrentOperation"/>

	</aop:aspect>

</aop:config>

<!-- Define the aspect bean itself -->
<bean id="concurrentOperationExecutor"
	class="com.xyz.service.impl.ConcurrentOperationExecutor"> <!-- Adjust class path -->
		<property name="maxRetries" value="3"/>
		<property name="order" value="100"/>
</bean>
```

현재로서는 모든 비즈니스 서비스가 멱등(idempotent)하다고 가정한다는 점에 유의하십시오. 그렇지 않은 경우, `Idempotent` 어노테이션을 도입하고 다음 예제와 같이 서비스 작업 구현에 어노테이션을 사용하여 진정으로 멱등한 작업만 재시도하도록 애스펙트를 구체화할 수 있습니다:

```java
// Java - Idempotent Annotation (Same as before)
import java.lang.annotation.*;
@Retention(RetentionPolicy.RUNTIME)
@Target(ElementType.METHOD)
public @interface Idempotent {}
```

```kotlin
// Kotlin - Idempotent Annotation (Same as before)
import kotlin.annotation.*
@Retention(AnnotationRetention.RUNTIME)
@Target(AnnotationTarget.FUNCTION)
annotation class Idempotent
```

멱등 작업만 재시도하도록 애스펙트를 변경하는 것은 `@Idempotent` 작업만 일치하도록 포인트컷 표현식을 구체화하는 것을 포함합니다. 다음과 같습니다:

```xml
<aop:pointcut id="idempotentOperation"
		expression="execution(* com.xyz.service.*.*(..)) and
		@annotation(com.xyz.service.Idempotent)"/> <!-- Added @annotation check -->
```
