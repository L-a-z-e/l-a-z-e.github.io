---
title: Pointcut API
description: 
author: laze
date: 2025-05-06 00:00:06 +0900
categories: [Dev, SpringBoot]
tags: [SpringBoot]
---
**Pointcut API in Spring**

이 섹션에서는 스프링이 중요한 포인트컷(pointcut) 개념을 어떻게 처리하는지 설명합니다.

**개념 (Concepts)**

스프링의 포인트컷 모델은 어드바이스(advice) 타입과 독립적으로 포인트컷 재사용을 가능하게 합니다.

동일한 포인트컷으로 다른 어드바이스를 타겟팅할 수 있습니다.

`org.springframework.aop.Pointcut` 인터페이스는 특정 클래스와 메소드에 어드바이스를 타겟팅하는 데 사용되는 핵심 인터페이스입니다. 전체 인터페이스는 다음과 같습니다:

```java
public interface Pointcut {

	ClassFilter getClassFilter();

	MethodMatcher getMethodMatcher();
}
```

`Pointcut` 인터페이스를 두 부분으로 나누면 클래스 및 메소드 매칭 부분의 재사용과 세분화된 구성 작업(예: 다른 메소드 매처와의 "합집합(union)")이 가능해집니다.

`ClassFilter` 인터페이스는 포인트컷을 주어진 대상 클래스 집합으로 제한하는 데 사용됩니다.

`matches()` 메소드가 항상 `true`를 반환하면 모든 대상 클래스가 매치됩니다.

다음 목록은 `ClassFilter` 인터페이스 정의를 보여줍니다:

```java
public interface ClassFilter {

	boolean matches(Class<?> clazz); // Check if the class matches
}
```

`MethodMatcher` 인터페이스는 일반적으로 더 중요합니다. 전체 인터페이스는 다음과 같습니다:

```java
public interface MethodMatcher {

	// Static check: Does this matcher *ever* match this method on this class?
	boolean matches(Method m, @Nullable Class<?> targetClass);

	// Is this a runtime matcher? (If not, the 3-arg matches method is never called)
	boolean isRuntime();

	// Runtime check: Does this matcher match this method execution with these arguments?
	boolean matches(Method m, @Nullable Class<?> targetClass, Object... args);
}
```

`matches(Method, Class)` 메소드는 이 포인트컷이 대상 클래스의 주어진 메소드와 일치하는지 여부를 테스트하는 데 사용됩니다.

이 평가는 AOP 프록시가 생성될 때 수행될 수 있어 모든 메소드 호출 시 테스트 필요성을 피할 수 있습니다.

주어진 메소드에 대해 2-인수 `matches` 메소드가 `true`를 반환하고 `MethodMatcher`의 `isRuntime()` 메소드가 `true`를 반환하면, 3-인수 `matches` 메소드가 모든 메소드 호출 시 호출됩니다.

이를 통해 포인트컷은 대상 어드바이스가 시작되기 직전에 메소드 호출에 전달된 인수를 살펴볼 수 있습니다.

대부분의 `MethodMatcher` 구현은 정적(static)이며, 이는 `isRuntime()` 메소드가 `false`를 반환함을 의미합니다.

이 경우 3-인수 `matches` 메소드는 절대 호출되지 않습니다.

*가능하다면 포인트컷을 정적으로 만들어 AOP 프레임워크가 AOP 프록시 생성 시 포인트컷 평가 결과를 캐시할 수 있도록 하십시오.*

**포인트컷 연산 (Operations on Pointcuts)**

스프링은 포인트컷에 대한 연산(특히, 합집합 및 교집합)을 지원합니다.

합집합(Union)은 두 포인트컷 중 하나라도 매치되는 메소드를 의미합니다.

교집합(Intersection)은 두 포인트컷 모두 매치되는 메소드를 의미합니다.

합집합이 일반적으로 더 유용합니다. `org.springframework.aop.support.Pointcuts` 클래스의 정적 메소드를 사용하거나 동일한 패키지의 `ComposablePointcut` 클래스를 사용하여 포인트컷을 구성할 수 있습니다.

그러나 AspectJ 포인트컷 표현식을 사용하는 것이 일반적으로 더 간단한 접근 방식입니다.

**AspectJ 표현식 포인트컷 (AspectJ Expression Pointcuts)**

2.0 이후로 스프링에서 사용하는 가장 중요한 타입의 포인트컷은 `org.springframework.aop.aspectj.AspectJExpressionPointcut`입니다.

이는 AspectJ 포인트컷 표현식 문자열을 파싱하기 위해 AspectJ 제공 라이브러리를 사용하는 포인트컷입니다.

**편의 포인트컷 구현체 (Convenience Pointcut Implementations)**

스프링은 여러 편리한 포인트컷 구현체를 제공합니다.

일부는 직접 사용할 수 있습니다.

다른 것들은 애플리케이션 특정 포인트컷에서 하위 클래스로 만들어 사용하도록 의도되었습니다.

*정적 포인트컷 (Static Pointcuts)*
정적 포인트컷은 메소드와 대상 클래스를 기반으로 하며 메소드의 인수를 고려할 수 없습니다.

정적 포인트컷은 대부분의 사용 사례에 충분하며 가장 좋습니다.

스프링은 메소드가 처음 호출될 때 정적 포인트컷을 한 번만 평가할 수 있습니다.

그 이후에는 각 메소드 호출 시 포인트컷을 다시 평가할 필요가 없습니다.

*정규 표현식 포인트컷 (Regular Expression Pointcuts)*
정적 포인트컷을 지정하는 한 가지 명백한 방법은 정규 표현식입니다.

스프링 외의 여러 AOP 프레임워크도 이를 가능하게 합니다.

`org.springframework.aop.support.JdkRegexpMethodPointcut`은 JDK의 정규 표현식 지원을 사용하는 일반적인 정규 표현식 포인트컷입니다.

`JdkRegexpMethodPointcut` 클래스를 사용하면 패턴 문자열 목록을 제공할 수 있습니다.

이 중 하나라도 일치하면 포인트컷은 `true`로 평가됩니다. (결과적으로 결과 포인트컷은 지정된 패턴들의 효과적인 합집합입니다.)

다음 예제는 `JdkRegexpMethodPointcut` 사용 방법을 보여줍니다:

**Java**

```java
import org.springframework.aop.support.JdkRegexpMethodPointcut;
import org.springframework.context.annotation.Bean;
import org.springframework.context.annotation.Configuration;

@Configuration
public class JdkRegexpConfiguration {

	@Bean
	public JdkRegexpMethodPointcut settersAndAbsquatulatePointcut() {
		JdkRegexpMethodPointcut pointcut = new JdkRegexpMethodPointcut();
		// Match methods starting with 'set' or named 'absquatulate'
		pointcut.setPatterns(".*set.*", ".*absquatulate");
		return pointcut;
	}
}
```

**Kotlin**

```kotlin
import org.springframework.aop.support.JdkRegexpMethodPointcut
import org.springframework.context.annotation.Bean
import org.springframework.context.annotation.Configuration

@Configuration
class JdkRegexpConfiguration {

    @Bean
    fun settersAndAbsquatulatePointcut(): JdkRegexpMethodPointcut {
        val pointcut = JdkRegexpMethodPointcut()
        // Match methods starting with 'set' or named 'absquatulate'
        pointcut.setPatterns(".*set.*", ".*absquatulate")
        return pointcut
    }
}
```

스프링은 `RegexpMethodPointcutAdvisor`라는 편의 클래스를 제공하여, `Advice`도 참조할 수 있게 합니다 (`Advice`는 인터셉터, before 어드바이스, throws 어드바이스 등이 될 수 있음을 기억하십시오).

내부적으로 스프링은 `JdkRegexpMethodPointcut`을 사용합니다. 다음 예제에서 보여주듯이, 하나의 빈이 포인트컷과 어드바이스 모두를 캡슐화하므로 `RegexpMethodPointcutAdvisor`를 사용하면 와이어링이 단순화됩니다:

**Java**

```java
import org.aopalliance.aop.Advice; // Import Advice interface
import org.springframework.aop.support.RegexpMethodPointcutAdvisor;
import org.springframework.context.annotation.Bean;
import org.springframework.context.annotation.Configuration;

@Configuration
public class RegexpConfiguration {

	// Assume an Advice bean named 'myInterceptor' is defined elsewhere
	@Bean
	public RegexpMethodPointcutAdvisor settersAndAbsquatulateAdvisor(Advice myInterceptor) {
		RegexpMethodPointcutAdvisor advisor = new RegexpMethodPointcutAdvisor();
		advisor.setAdvice(myInterceptor); // Set the advice bean
		advisor.setPatterns(".*set.*", ".*absquatulate"); // Set the patterns
		return advisor;
	}
}
```

**Kotlin**

```kotlin
import org.aopalliance.aop.Advice // Import Advice interface
import org.springframework.aop.support.RegexpMethodPointcutAdvisor
import org.springframework.context.annotation.Bean
import org.springframework.context.annotation.Configuration

@Configuration
class RegexpConfiguration {

    // Assume an Advice bean named 'myInterceptor' is defined elsewhere
    @Bean
    fun settersAndAbsquatulateAdvisor(myInterceptor: Advice): RegexpMethodPointcutAdvisor {
        val advisor = RegexpMethodPointcutAdvisor()
        advisor.advice = myInterceptor // Set the advice bean
        advisor.setPatterns(".*set.*", ".*absquatulate") // Set the patterns
        return advisor
    }
}
```

모든 `Advice` 타입과 함께 `RegexpMethodPointcutAdvisor`를 사용할 수 있습니다.

*속성 기반 포인트컷 (Attribute-driven Pointcuts)*
정적 포인트컷의 중요한 타입은 메타데이터 기반 포인트컷입니다. 이는 메타데이터 속성(일반적으로 소스 레벨 메타데이터)의 값을 사용합니다.

*동적 포인트컷 (Dynamic pointcuts)*
동적 포인트컷은 정적 포인트컷보다 평가 비용이 더 비쌉니다.

정적 정보뿐만 아니라 메소드 인수도 고려합니다.

이는 즉, 모든 메소드 호출 시 평가되어야 하며 인수가 달라지므로 결과를 캐시할 수 없다는 것을 의미합니다.

주요 예는 제어 흐름 포인트컷입니다.

*제어 흐름 포인트컷 (Control Flow Pointcuts)*
스프링 제어 흐름 포인트컷은 개념적으로 AspectJ `cflow` 포인트컷과 유사하지만 덜 강력합니다. (현재 다른 포인트컷과 일치하는 조인 포인트 아래에서 포인트컷이 실행되도록 지정할 방법은 없습니다.)

제어 흐름 포인트컷은 현재 호출 스택(call stack)과 일치합니다.

예를 들어, 조인 포인트가 `com.mycompany.web` 패키지의 메소드나 `SomeCaller` 클래스에 의해 호출된 경우 발생할 수 있습니다.

제어 흐름 포인트컷은 `org.springframework.aop.support.ControlFlowPointcut` 클래스를 사용하여 지정됩니다.

*제어 흐름 포인트컷은 런타임 시 다른 동적 포인트컷보다 훨씬 더 평가 비용이 비쌉니다. 자바 1.4에서는 다른 동적 포인트컷 비용의 약 5배입니다.*

**포인트컷 슈퍼클래스 (Pointcut Superclasses)**

스프링은 자신만의 포인트컷을 구현하는 데 도움이 되는 유용한 포인트컷 슈퍼클래스를 제공합니다.

정적 포인트컷이 가장 유용하므로, 아마도 `StaticMethodMatcherPointcut`을 하위 클래스로 만들어야 할 것입니다.

이는 단 하나의 추상 메소드만 구현하면 됩니다 (비록 동작을 사용자 정의하기 위해 다른 메소드를 오버라이드할 수 있지만). 다음 예제는 `StaticMethodMatcherPointcut`을 하위 클래스로 만드는 방법을 보여줍니다:

```java
// Java
import org.springframework.aop.support.StaticMethodMatcherPointcut;
import java.lang.reflect.Method;

class TestStaticPointcut extends StaticMethodMatcherPointcut {

	@Override
	public boolean matches(Method method, Class<?> targetClass) {
		// 커스텀 기준이 일치하면 true 반환
		// Example: Check if method name starts with "get"
		return method.getName().startsWith("get");
	}
}
```

```kotlin
// Kotlin
import org.springframework.aop.support.StaticMethodMatcherPointcut
import java.lang.reflect.Method

class TestStaticPointcut : StaticMethodMatcherPointcut() { // Extend the class

    override fun matches(method: Method, targetClass: Class<*>?): Boolean {
        // 커스텀 기준이 일치하면 true 반환
        // Example: Check if method name starts with "get"
        return method.name.startsWith("get")
    }
}
```

동적 포인트컷을 위한 슈퍼클래스도 있습니다. 모든 어드바이스 타입과 함께 커스텀 포인트컷을 사용할 수 있습니다.

**커스텀 포인트컷 (Custom Pointcuts)**

스프링 AOP의 포인트컷은 (AspectJ에서처럼) 언어 기능이 아닌 자바 클래스이므로, 정적이든 동적이든 커스텀 포인트컷을 선언할 수 있습니다.

스프링의 커스텀 포인트컷은 임의로 복잡할 수 있습니다.

그러나 가능하다면 AspectJ 포인트컷 표현식 언어를 사용하는 것이 좋습니다.

*향후 버전의 스프링은 JAC에서 제공하는 것과 같은 "의미론적 포인트컷(semantic pointcuts)" 지원을 제공할 수 있습니다 - 예를 들어, "대상 객체의 인스턴스 변수를 변경하는 모든 메소드."*

---

**전체 주제: 스프링의 포인트컷 API (Pointcut API in Spring)**

이 부분은 스프링 프레임워크가 내부적으로 포인트컷을 어떻게 **모델링(인터페이스 정의)** 하고, 어떤 **기본 구현체**들을 제공하며, 개발자가 어떻게 **자신만의 커스텀 포인트컷**을 만들 수 있는지 설명합니다. 주로 스프링 프레임워크 자체나 고급 AOP 활용 시 관련되는 내용입니다.

**핵심 아이디어:** 포인트컷은 단순히 AspectJ 표현식 문자열만이 아니라, 스프링 내부에서는 `Pointcut`, `ClassFilter`, `MethodMatcher` 같은 인터페이스로 정의된 객체이다. 스프링은 이러한 인터페이스의 구현체들을 제공하며, 개발자도 직접 구현하여 커스텀 포인트컷 로직을 만들 수 있다.

---

**1. 개념 (Concepts)**

- **포인트컷 재사용:** 스프링의 포인트컷 모델은 **어드바이스 타입과 독립적**입니다. 즉, 하나의 포인트컷 정의(어디에 적용할지)를 여러 다른 종류의 어드바이스(무엇을 할지)에 재사용할 수 있습니다.
- **핵심 인터페이스 (`Pointcut`):** `org.springframework.aop.Pointcut` 인터페이스는 스프링 AOP에서 포인트컷의 가장 기본적인 정의입니다.
  - **두 부분으로 구성:**
    1. **`ClassFilter getClassFilter()`:** 어떤 **클래스**가 대상이 될 수 있는지 필터링합니다.
    2. **`MethodMatcher getMethodMatcher()`:** 해당 클래스 내의 어떤 **메소드**가 대상이 될 수 있는지 필터링합니다.
  - 이렇게 클래스 필터와 메소드 매처를 분리함으로써, 각각을 재사용하거나 조합(예: 같은 클래스 필터에 다른 메소드 매처 연결)하는 것이 가능해집니다.

---

**2. `ClassFilter` 인터페이스**

- **역할:** 포인트컷이 적용될 **대상 클래스**를 제한(필터링)합니다.
- **메소드:** `boolean matches(Class<?> clazz)`
  - 주어진 `clazz`가 이 포인트컷의 대상 클래스 조건에 맞으면 `true`, 아니면 `false`를 반환합니다.
  - 만약 모든 클래스를 대상으로 하고 싶다면 항상 `true`를 반환하는 구현체를 사용합니다 (예: `ClassFilter.TRUE`).

---

**3. `MethodMatcher` 인터페이스**

- **역할:** 특정 클래스 내에서 포인트컷이 적용될 **대상 메소드**를 식별합니다. (더 중요하고 복잡함)
- **메소드:**
  - **`boolean matches(Method method, @Nullable Class<?> targetClass)` (정적 검사):**
    - **핵심:** 이 메소드 매처가 주어진 `targetClass`의 `method`와 **일치할 가능성이 있는지** 정적으로 검사합니다. 메소드 이름, 파라미터 타입 등 시그니처 정보만으로 판단합니다.
    - **시점:** AOP 프록시가 생성될 때 미리 호출될 수 있습니다. 여기서 `false`가 반환되면, 해당 메소드는 절대 어드바이스 대상이 되지 않으므로 런타임 검사를 피할 수 있습니다.
  - **`boolean isRuntime()`:**
    - **역할:** 이 메소드 매처가 **런타임(메소드 실제 호출 시점)** 에 인수를 확인하는 동적 검사를 필요로 하는지 여부를 반환합니다.
    - `false` 반환 (대부분의 경우): 정적 매처. 3-인수 `matches`는 호출되지 않습니다. 성능상 유리.
    - `true` 반환: 동적 매처. 3-인수 `matches`가 매번 호출됩니다.
  - **`boolean matches(Method method, @Nullable Class<?> targetClass, Object... args)` (동적 검사):**
    - **호출 조건:** 2-인수 `matches`가 `true`이고 `isRuntime()`이 `true`일 때만, **매 메소드 호출 시마다** 호출됩니다.
    - **역할:** 메소드 시그니처뿐만 아니라, 실제로 메소드에 **전달된 인수(`args`)** 까지 확인하여 최종적으로 이 호출에 어드바이스를 적용할지 결정합니다. (예: 첫 번째 인수가 특정 값일 때만 적용)
    - **성능:** 매번 호출되므로 정적 매처보다 비용이 더 큽니다.
- **정적 포인트컷 권장:** 가능하면 `isRuntime()`이 `false`인 **정적 포인트컷**을 사용하는 것이 좋습니다. 스프링 AOP 프레임워크가 프록시 생성 시점에 평가 결과를 캐시하여 런타임 성능을 향상시킬 수 있기 때문입니다.

---

**4. 포인트컷 연산 (Operations on Pointcuts)**

- 스프링은 여러 포인트컷을 조합하는 연산을 지원합니다.
  - **합집합 (Union):** 두 포인트컷 중 **하나라도** 매치되면 참. (`Pointcuts.union()`)
  - **교집합 (Intersection):** 두 포인트컷 **모두** 매치되어야 참. (`Pointcuts.intersection()`)
- `org.springframework.aop.support.Pointcuts` 유틸리티 클래스나 `ComposablePointcut` 클래스를 사용하여 프로그래밍 방식으로 조합할 수 있습니다.
- **하지만:** 일반적으로 **AspectJ 포인트컷 표현식 (`&&`, `||`, `!`)** 을 사용하는 것이 훨씬 간단하고 강력합니다.

---

**5. AspectJ 표현식 포인트컷 (`AspectJExpressionPointcut`)**

- **핵심:** 스프링 AOP에서 **가장 중요하고 널리 사용되는** `Pointcut` 구현체입니다.
- **역할:** `org.springframework.aop.aspectj.AspectJExpressionPointcut` 클래스는 **AspectJ 포인트컷 표현식 문자열**을 입력받아 파싱하고, 그 표현식에 맞는 클래스와 메소드를 매칭하는 `Pointcut` 객체를 생성합니다.
- **내부 동작:** AspectJ 라이브러리를 사용하여 표현식을 해석하고 매칭 로직을 수행합니다.
- **사용:** `@AspectJ` 스타일의 `@Pointcut` 어노테이션이나 XML의 `expression` 속성에 지정된 문자열이 내부적으로 이 클래스(또는 유사한 메커니즘)를 통해 처리됩니다.

---

**6. 편의 포인트컷 구현체 (Convenience Pointcut Implementations)**

스프링은 AspectJ 표현식 외에도 몇 가지 미리 정의된 `Pointcut` 구현 클래스를 제공합니다. (하지만 AspectJ 표현식이 훨씬 강력해서 잘 사용되지는 않습니다.)

- **정적 포인트컷 구현체:**
  - **`JdkRegexpMethodPointcut`:** **정규 표현식(Regular Expression)** 을 사용하여 메소드 이름 패턴을 매칭하는 정적 포인트컷입니다. 여러 패턴을 지정하면 OR 조건으로 합쳐집니다.
  - **`NameMatchMethodPointcut`:** 메소드 이름을 직접 지정하여 매칭합니다.
  - **`ControlFlowPointcut` (동적이지만 참고):** 특정 메소드 호출 스택 하위에서 실행되는지 확인합니다. (비용이 매우 큼)
- **`RegexpMethodPointcutAdvisor`:** 정규 표현식 포인트컷(`JdkRegexpMethodPointcut`)과 어드바이스(`Advice` 빈)를 **하나로 묶어주는 편리한 어드바이저(Advisor)** 클래스입니다. 포인트컷과 어드바이스를 별도로 설정하는 대신 이 어드바이저 빈 하나만 정의하면 됩니다.
- **속성 기반 포인트컷 (`Attribute-driven Pointcuts`):** 메타데이터(주로 어노테이션)를 기반으로 매칭하는 포인트컷입니다. (`@annotation` 지정자와 유사한 역할)

---

**7. 포인트컷 슈퍼클래스 (직접 구현 시)**

개발자가 자신만의 커스텀 포인트컷 로직을 구현해야 할 때, 스프링은 편리한 추상 클래스를 제공합니다.

- **`StaticMethodMatcherPointcut`:** **정적 포인트컷**을 만들 때 상속받는 추상 클래스입니다. `matches(Method method, Class<?> targetClass)` 추상 메소드만 구현하면 됩니다.
- **`DynamicMethodMatcherPointcut`:** **동적 포인트컷**을 만들 때 상속받습니다. `matches(Method method, Class<?> targetClass, Object... args)` 메소드를 구현해야 합니다. (`isRuntime()`은 기본적으로 `true`)

---

**8. 커스텀 포인트컷 (Custom Pointcuts)**

- 스프링 AOP의 포인트컷은 자바 클래스이므로, 개발자는 필요에 따라 `Pointcut`, `ClassFilter`, `MethodMatcher` 인터페이스를 직접 구현하거나 제공된 슈퍼클래스를 상속하여 매우 복잡하고 특수한 조건의 **커스텀 포인트컷**을 만들 수 있습니다.
- **하지만:** 대부분의 경우 강력하고 표준화된 **AspectJ 포인트컷 표현식 언어**를 사용하는 것이 더 간단하고 효율적입니다. 커스텀 포인트컷 구현은 정말 특별한 경우가 아니면 필요하지 않을 수 있습니다.

**요약:**

스프링 AOP에서 포인트컷은 단순히 표현식 문자열이 아니라, 내부적으로 **`Pointcut` 인터페이스(와 `ClassFilter`, `MethodMatcher`)** 로 모델링됩니다. 스프링은 **`AspectJExpressionPointcut`** 을 통해 AspectJ 표현식을 사용하는 것을 핵심으로 지원하며, 정규 표현식 기반 포인트컷 등 몇 가지 편의 구현체도 제공합니다. 개발자는 필요시 `StaticMethodMatcherPointcut` 등을 상속하여 **커스텀 포인트컷**을 만들 수도 있지만, 대부분의 경우 AspectJ 표현식을 사용하는 것이 권장됩니다. 포인트컷은 정적(성능 유리)과 동적(인수 확인 가능)으로 나눌 수 있습니다.
