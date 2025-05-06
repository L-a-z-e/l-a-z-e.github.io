---
title: Manipulating Advised Objects
description: 
author: laze
date: 2025-05-06 00:00:10 +0900
categories: [Dev, SpringBoot]
tags: [SpringBoot]
---
**Manipulating Advised Objects**

AOP 프록시를 어떻게 생성하든, `org.springframework.aop.framework.Advised` 인터페이스를 사용하여 조작할 수 있습니다.

모든 AOP 프록시는 구현하는 다른 인터페이스에 관계없이 이 인터페이스로 캐스팅될 수 있습니다. 이 인터페이스는 다음 메소드를 포함합니다:

**Java**

```java
import org.aopalliance.aop.Advice;
import org.springframework.aop.Advisor;
import org.springframework.aop.AopConfigException;

public interface Advised {

	Advisor[] getAdvisors(); // 추가된 모든 어드바이저 배열 반환

	void addAdvice(Advice advice) throws AopConfigException; // 어드바이스 추가 (항상 참인 포인트컷으로 래핑됨)
	void addAdvice(int pos, Advice advice) throws AopConfigException; // 특정 위치에 어드바이스 추가

	void addAdvisor(Advisor advisor) throws AopConfigException; // 어드바이저 추가
	void addAdvisor(int pos, Advisor advisor) throws AopConfigException; // 특정 위치에 어드바이저 추가

	int indexOf(Advisor advisor); // 특정 어드바이저의 인덱스 반환

	boolean removeAdvisor(Advisor advisor) throws AopConfigException; // 어드바이저 제거
	void removeAdvisor(int index) throws AopConfigException; // 인덱스로 어드바이저 제거

	boolean replaceAdvisor(Advisor a, Advisor b) throws AopConfigException; // 어드바이저 교체

	boolean isFrozen(); // 프록시 구성이 동결(frozen)되었는지 여부 반환
}
```

**Kotlin**

```kotlin
import org.aopalliance.aop.Advice
import org.springframework.aop.Advisor
import org.springframework.aop.AopConfigException

interface Advised {
    val advisors: Array<Advisor> // Added advisors as a property

    @Throws(AopConfigException::class)
    fun addAdvice(advice: Advice)

    @Throws(AopConfigException::class)
    fun addAdvice(pos: Int, advice: Advice)

    @Throws(AopConfigException::class)
    fun addAdvisor(advisor: Advisor)

    @Throws(AopConfigException::class)
    fun addAdvisor(pos: Int, advisor: Advisor)

    fun indexOf(advisor: Advisor): Int

    @Throws(AopConfigException::class)
    fun removeAdvisor(advisor: Advisor): Boolean

    @Throws(AopConfigException::class)
    fun removeAdvisor(index: Int)

    @Throws(AopConfigException::class)
    fun replaceAdvisor(a: Advisor, b: Advisor): Boolean

    val isFrozen: Boolean // Frozen status as a property
}
```

`getAdvisors()` 메소드는 팩토리에 추가된 모든 어드바이저, 인터셉터 또는 다른 어드바이스 타입에 대해 `Advisor`를 반환합니다.

`Advisor`를 추가했다면 이 인덱스에서 반환된 어드바이저는 추가한 객체입니다.

인터셉터나 다른 어드바이스 타입을 추가했다면, 스프링은 이것을 항상 `true`를 반환하는 포인트컷을 가진 어드바이저로 래핑(wrapped)했습니다.

따라서 `MethodInterceptor`를 추가했다면 이 인덱스에 대해 반환된 어드바이저는 `MethodInterceptor`와 모든 클래스 및 메소드와 일치하는 포인트컷을 반환하는 `DefaultPointcutAdvisor`입니다.

`addAdvisor()` 메소드는 모든 `Advisor`를 추가하는 데 사용될 수 있습니다.

일반적으로 포인트컷과 어드바이스를 보유하는 어드바이저는 제네릭 `DefaultPointcutAdvisor`이며, 이는 모든 어드바이스 또는 포인트컷(인트로덕션 제외)과 함께 사용할 수 있습니다.

기본적으로 프록시가 생성된 후에도 어드바이저나 인터셉터를 추가하거나 제거하는 것이 가능합니다.

유일한 제한 사항은 팩토리의 기존 프록시가 인터페이스 변경을 보여주지 않기 때문에 인트로덕션 어드바이저를 추가하거나 제거하는 것이 불가능하다는 것입니다. (이 문제를 피하기 위해 팩토리에서 새 프록시를 얻을 수 있습니다.)

다음 예제는 AOP 프록시를 `Advised` 인터페이스로 캐스팅하고 해당 어드바이스를 검사하고 조작하는 것을 보여줍니다:

**Java**

```java
import org.springframework.aop.Advisor;
import org.springframework.aop.framework.Advised;
import org.springframework.aop.interceptor.DebugInterceptor;
import org.springframework.aop.support.DefaultPointcutAdvisor;
// Assuming myObject is an AOP proxy, mySpecialPointcut and myAdvice exist
import static org.junit.jupiter.api.Assertions.assertEquals; // For testing assertion

Advised advised = (Advised) myObject; // Cast proxy to Advised
Advisor[] advisors = advised.getAdvisors(); // Get current advisors
int oldAdvisorCount = advisors.length;
System.out.println(oldAdvisorCount + " advisors");

// 포인트컷 없이 인터셉터처럼 어드바이스 추가
// 모든 프록시된 메소드와 일치할 것임
// 인터셉터, before, after returning 또는 throws 어드바이스에 사용 가능
advised.addAdvice(new DebugInterceptor());

// 포인트컷을 사용하여 선택적 어드바이스 추가
advised.addAdvisor(new DefaultPointcutAdvisor(mySpecialPointcut, myAdvice));

assertEquals(oldAdvisorCount + 2, advised.getAdvisors().length, "Added two advisors"); // Verify addition
```

**Kotlin**

```kotlin
import org.springframework.aop.Advisor
import org.springframework.aop.Pointcut // Import Pointcut
import org.aopalliance.aop.Advice // Import Advice
import org.springframework.aop.framework.Advised
import org.springframework.aop.interceptor.DebugInterceptor
import org.springframework.aop.support.DefaultPointcutAdvisor
import org.junit.jupiter.api.Assertions.assertEquals // For testing assertion
// Assuming myObject is an AOP proxy, mySpecialPointcut and myAdvice exist

val advised = myObject as Advised // Cast proxy to Advised
val advisors: Array<Advisor> = advised.advisors // Get current advisors using property
val oldAdvisorCount = advisors.size
println("$oldAdvisorCount advisors")

// 포인트컷 없이 인터셉터처럼 어드바이스 추가
// 모든 프록시된 메소드와 일치할 것임
// 인터셉터, before, after returning 또는 throws 어드바이스에 사용 가능
advised.addAdvice(DebugInterceptor())

// 포인트컷을 사용하여 선택적 어드바이스 추가
// Assuming mySpecialPointcut is a Pointcut and myAdvice is an Advice
advised.addAdvisor(DefaultPointcutAdvisor(mySpecialPointcut, myAdvice))

assertEquals(oldAdvisorCount + 2, advised.advisors.size, "Added two advisors") // Verify addition
```

프로덕션 환경에서 비즈니스 객체의 어드바이스를 수정하는 것이 바람직한지(어떤 말장난도 의도하지 않음)는 의문이지만, 의심할 여지 없이 합법적인 사용 사례가 있습니다.

그러나 개발 중에는 (예: 테스트에서) 매우 유용할 수 있습니다.

테스트하려는 메소드 호출 내부에 들어가기 위해 인터셉터나 다른 어드바이스 형태로 테스트 코드를 추가하는 것이 매우 유용하다는 것을 때때로 발견했습니다. (예를 들어, 어드바이스는 해당 메소드에 대해 생성된 트랜잭션 내부에 들어가서, 트랜잭션을 롤백(roll back)으로 표시하기 전에 데이터베이스가 올바르게 업데이트되었는지 확인하기 위해 SQL을 실행할 수 있습니다.)

프록시를 생성한 방식에 따라 일반적으로 동결(frozen) 플래그를 설정할 수 있습니다.

이 경우 `Advised` `isFrozen()` 메소드는 `true`를 반환하며, 추가 또는 제거를 통해 어드바이스를 수정하려는 모든 시도는 `AopConfigException`을 발생시킵니다.

어드바이스된 객체의 상태를 동결하는 기능은 어떤 경우에는 유용합니다 (예: 호출 코드가 보안 인터셉터를 제거하는 것을 방지하기 위해).

---

**전체 주제: 어드바이스된 객체 조작하기 (Manipulating Advised Objects)**

이 부분은 스프링 AOP에 의해 생성된 **프록시 객체** 자체를 검사하고 **런타임에 어드바이스(부가 기능)를 추가, 제거, 교체**하는 등의 작업을 수행할 수 있도록 해주는 **`org.springframework.aop.framework.Advised`** 인터페이스에 대한 설명입니다.

**핵심 아이디어:** 일단 만들어진 AOP 프록시라도, 그 내부 구성을 들여다보거나 동적으로 변경할 수 있다! (단, 주의해서 사용해야 함)

---

**1. `Advised` 인터페이스란?**

- **모든 스프링 AOP 프록시가 구현하는 인터페이스:** 스프링 AOP 프레임워크에 의해 생성된 모든 프록시 객체는 (원래 구현하는 인터페이스나 상속하는 클래스와 더불어) **반드시 `Advised` 인터페이스를 구현**합니다.
- **역할:** 이 인터페이스는 해당 프록시 객체에 **현재 적용되어 있는 AOP 설정 정보(어드바이저, 어드바이스 등)에 접근**하고, **동적으로 이 설정을 조작**할 수 있는 메소드들을 제공합니다.
- **접근 방법:** AOP 프록시 객체에 대한 참조가 있다면, 이를 `(Advised)` 로 **캐스팅**하여 `Advised` 인터페이스의 메소드를 사용할 수 있습니다.

---

**2. `Advised` 인터페이스의 주요 메소드:**

- **`Advisor[] getAdvisors()`:** 현재 이 프록시에 적용되어 있는 **모든 어드바이저(Advisor) 객체의 배열**을 반환합니다.
  - **주의:** 만약 `MethodInterceptor`나 다른 어드바이스 타입만 추가했다면, 스프링은 내부적으로 이를 **항상 `true`를 반환하는 포인트컷을 가진 `DefaultPointcutAdvisor`로 감싸서** 관리합니다. 따라서 이 메소드는 항상 `Advisor` 배열을 반환합니다.
- **`void addAdvice(Advice advice)`:** 주어진 어드바이스 객체를 **인터셉터 체인의 끝**에 추가합니다. 이 어드바이스는 **모든 메소드 호출에 적용**됩니다 (항상 `true`인 포인트컷과 함께 `DefaultPointcutAdvisor`로 래핑됨). Before, After Returning, Throws, Around(MethodInterceptor) 어드바이스 모두 가능합니다.
- **`void addAdvice(int pos, Advice advice)`:** 특정 위치(인덱스 `pos`)에 어드바이스를 추가합니다.
- **`void addAdvisor(Advisor advisor)`:** 어드바이저(포인트컷 + 어드바이스) 객체를 **인터셉터 체인의 끝**에 추가합니다.
- **`void addAdvisor(int pos, Advisor advisor)`:** 특정 위치(인덱스 `pos`)에 어드바이저를 추가합니다.
- **`boolean removeAdvisor(Advisor advisor)`:** 주어진 어드바이저를 제거합니다. 성공하면 `true`.
- **`void removeAdvisor(int index)`:** 특정 위치의 어드바이저를 제거합니다.
- **`boolean replaceAdvisor(Advisor a, Advisor b)`:** 기존 어드바이저(`a`)를 새로운 어드바이저(`b`)로 교체합니다.
- **`int indexOf(Advisor advisor)`:** 특정 어드바이저의 현재 인덱스(순서)를 반환합니다.
- **`boolean isFrozen()`:** 이 프록시의 AOP 구성이 **동결(frozen)** 되어 더 이상 변경(추가/제거/교체)할 수 없는 상태인지 여부를 반환합니다.

---

**3. 사용 시나리오 및 주의사항:**

- **동적 어드바이스 조작:** 애플리케이션 실행 중에 특정 조건에 따라 AOP 부가 기능(어드바이스)을 **동적으로 추가하거나 제거**해야 하는 특별한 경우에 사용할 수 있습니다.
- **검사 및 디버깅:** 개발 중이나 테스트 시, 특정 프록시 객체에 **어떤 어드바이스들이 어떤 순서로 적용되어 있는지 확인**하는 용도로 매우 유용할 수 있습니다.
  - **테스트 예시:** 테스트 코드에서 특정 메소드 호출 시점에 임시 디버깅 인터셉터(`DebugInterceptor`)나 데이터 검증 어드바이스를 동적으로 추가하여 내부 상태를 확인하고, 테스트가 끝나면 다시 제거하는 방식으로 활용할 수 있습니다.
- **인트로덕션 제한:** 프록시가 생성된 이후에는 **인트로덕션 어드바이저(새로운 인터페이스 추가)를 추가하거나 제거할 수 없습니다.** (프록시의 타입 자체가 변경되어야 하므로) 만약 인트로덕션을 변경하려면 팩토리에서 새로운 프록시를 다시 생성해야 합니다.
- **동결(Frozen) 설정:** `ProxyFactoryBean` 이나 다른 프록시 생성 설정에서 `frozen=true`로 설정하면, `isFrozen()`이 `true`를 반환하고 `addAdvisor`, `removeAdvisor` 등의 조작 메소드는 `AopConfigException`을 발생시킵니다. 이는 프록시 구성을 변경 불가능하게 만들어 안정성을 높이거나 보안적인 이유(예: 보안 관련 어드바이스 제거 방지)로 사용될 수 있습니다.
- **프로덕션 환경 사용:** 프로덕션 환경에서 런타임에 AOP 설정을 동적으로 변경하는 것은 **매우 신중하게 접근**해야 합니다. 예기치 않은 부작용을 일으킬 수 있으며, 시스템의 동작을 예측하기 어렵게 만들 수 있습니다. 주로 **개발 및 테스트 단계**에서 활용하는 것이 일반적입니다.

**요약:**

스프링 AOP 프록시는 **`Advised` 인터페이스**를 구현하며, 이를 통해 개발자는 **런타임에 프록시에 적용된 어드바이저(AOP 설정) 목록을 확인**하거나, **새로운 어드바이스/어드바이저를 동적으로 추가, 제거, 교체**할 수 있습니다. 이 기능은 주로 **개발, 테스트, 디버깅** 목적 또는 매우 특수한 동적 AOP 요구사항에 유용합니다. 단, 인트로덕션 조작은 불가능하며, `frozen` 설정을 통해 조작을 막을 수도 있습니다. 프로덕션 환경에서의 동적 조작은 주의가 필요합니다.

---

**인터셉터 체인(Interceptor Chain)과 인덱스(Index)**

스프링 AOP 프록시가 동작할 때, 하나의 조인 포인트(메소드 실행)에 여러 개의 어드바이스(Before, Around, After 등)가 적용될 수 있습니다. 스프링은 이 어드바이스들을 **실행 순서에 따라 내부적으로 정렬**하여 **"인터셉터 체인(Interceptor Chain)"** 또는 **"어드바이저 체인(Advisor Chain)"** 이라는 목록(List) 형태로 관리합니다.

- **체인(Chain):** 여러 인터셉터(어드바이스를 실행 가능한 형태로 변환한 것)나 어드바이저들이 순서대로 연결되어 있는 구조를 의미합니다.
- **실행 흐름:** 메소드 호출이 프록시에 들어오면, 이 체인의 **첫 번째 인터셉터부터 순서대로 실행**됩니다. Around 어드바이스(`MethodInterceptor`)의 경우 `invocation.proceed()`를 호출하면 체인의 **다음 인터셉터**로 제어가 넘어갑니다. 마지막 인터셉터가 `proceed()`를 호출하면 **실제 대상 메소드**가 실행됩니다. 그 후 결과가 다시 체인을 거슬러 올라가면서 After 계열의 로직이 실행됩니다.

**`pos` (Position/Index)의 의미:**

- `pos` 파라미터는 바로 이 **인터셉터(어드바이저) 체인이라는 목록에서의 인덱스(위치 번호)** 를 의미합니다.
- **0부터 시작:** 자바의 배열이나 리스트처럼 인덱스는 **0부터 시작**합니다.
- **`addAdvisor(int pos, Advisor advisor)` 동작:**
  - `pos` 값으로 **0**을 주면, 새로운 어드바이저를 체인의 **가장 맨 앞(첫 번째)** 에 삽입합니다. 이 어드바이저는 다른 어떤 어드바이스보다 먼저 실행될 기회를 갖게 됩니다.
  - `pos` 값으로 **현재 체인의 크기(`advised.getAdvisors().length`)** 를 주면, 체인의 **가장 마지막**에 추가하는 것과 같습니다 (`addAdvisor(advisor)`와 동일).
  - `pos` 값으로 **0 < pos < 체인 크기** 인 값을 주면, 해당 인덱스 위치에 어드바이저를 **삽입**하고, 기존에 그 위치 및 그 뒤에 있던 어드바이저들은 **한 칸씩 뒤로 밀려납니다.**
- **`removeAdvisor(int index)` 동작:** 해당 인덱스 위치에 있는 어드바이저를 체인에서 **제거**합니다. 그 뒤에 있던 어드바이저들은 앞으로 당겨집니다.

**예시:**

현재 어드바이저 체인이 다음과 같다고 가정해 봅시다 (인덱스 0부터 시작):

`[ Advisor A, Advisor B, Advisor C ]` (총 3개, 크기 3)

1. **`advised.addAdvisor(1, newAdvisorX)` 호출 시:**
  - 인덱스 1 위치에 `newAdvisorX`가 삽입됩니다.
  - 결과 체인: `[ Advisor A, newAdvisorX, Advisor B, Advisor C ]`
2. **`advised.addAdvisor(0, newAdvisorY)` 호출 시:**
  - 인덱스 0 위치에 `newAdvisorY`가 삽입됩니다.
  - 결과 체인: `[ newAdvisorY, Advisor A, newAdvisorX, Advisor B, Advisor C ]`
3. **`advised.removeAdvisor(2)` 호출 시:**
  - 인덱스 2 위치의 `newAdvisorX`가 제거됩니다.
  - 결과 체인: `[ newAdvisorY, Advisor A, Advisor B, Advisor C ]`

**결론:**

`pos` 인자는 스프링 AOP 프록시 내부의 **어드바이저(또는 인터셉터) 실행 순서 목록(체인)** 에서 **특정 위치(인덱스, 0부터 시작)** 를 가리킵니다. `addAdvisor(int pos, ...)`는 해당 위치에 새로운 어드바이저를 **삽입**하고, `removeAdvisor(int index)`는 해당 위치의 어드바이저를 **제거**하는 데 사용됩니다. 이를 통해 어드바이스의 실행 순서를 동적으로 정밀하게 제어할 수 있습니다. (단, 이 기능은 고급 사용 사례에 해당하며 주의가 필요합니다.)
