---
title: Advice API
description: 
author: laze
date: 2025-05-06 00:00:07 +0900
categories: [Dev, SpringBoot]
tags: [SpringBoot]
---
**Advice API in Spring**

**어드바이스 생명주기 (Advice Lifecycles)**

각 어드바이스는 스프링 빈입니다.

어드바이스 인스턴스는 모든 어드바이스받는(advised) 객체 간에 공유되거나 각 어드바이스받는 객체마다 고유할 수 있습니다.

이는 클래스별(per-class) 또는 인스턴스별(per-instance) 어드바이스에 해당합니다.

클래스별 어드바이스가 가장 자주 사용됩니다. 트랜잭션 어드바이저(transaction advisors)와 같은 일반적인 어드바이스에 적합합니다.

이것들은 프록시된 객체의 상태에 의존하거나 새로운 상태를 추가하지 않습니다.

단지 메소드와 인수에 따라 작동합니다.

인스턴스별 어드바이스는 믹스인(mixins)을 지원하기 위한 인트로덕션(introductions)에 적합합니다.

이 경우, 어드바이스는 프록시된 객체에 상태를 추가합니다.

동일한 AOP 프록시에서 공유 및 인스턴스별 어드바이스를 혼합하여 사용할 수 있습니다.

**스프링의 어드바이스 타입 (Advice Types in Spring)**

스프링은 여러 어드바이스 타입을 제공하며 임의의 어드바이스 타입을 지원하도록 확장 가능합니다.

*Interception Around 어드바이스*
스프링에서 가장 기본적인 어드바이스 타입은 interception around 어드바이스입니다.

스프링은 메소드 가로채기(method interception)를 사용하는 around 어드바이스에 대한 AOP 얼라이언스(AOP Alliance) 인터페이스를 준수합니다.

따라서 around 어드바이스를 구현하는 클래스는 `org.aopalliance.intercept` 패키지의 다음 `MethodInterceptor` 인터페이스를 구현해야 합니다:

```java
public interface MethodInterceptor extends Interceptor {

	Object invoke(MethodInvocation invocation) throws Throwable;
}
```

`invoke()` 메소드의 `MethodInvocation` 인수는 호출되는 메소드, 대상 조인 포인트, AOP 프록시 및 메소드 인수를 노출합니다.

`invoke()` 메소드는 호출의 결과를 반환해야 합니다: 일반적으로 조인 포인트의 반환 값입니다.

다음 예제는 간단한 `MethodInterceptor` 구현을 보여줍니다:

```java
// Java
import org.aopalliance.intercept.MethodInterceptor;
import org.aopalliance.intercept.MethodInvocation;

public class DebugInterceptor implements MethodInterceptor {

	@Override
	public Object invoke(MethodInvocation invocation) throws Throwable {
		System.out.println("Before: invocation=[" + invocation + "]");
		Object result = invocation.proceed(); // Proceed to the next interceptor or target method
		System.out.println("Invocation returned");
		return result;
	}
}
```

```kotlin
// Kotlin
import org.aopalliance.intercept.MethodInterceptor
import org.aopalliance.intercept.MethodInvocation

class DebugInterceptor : MethodInterceptor {

    @Throws(Throwable::class)
    override fun invoke(invocation: MethodInvocation): Any? {
        println("Before: invocation=[$invocation]")
        val result: Any? = invocation.proceed() // Proceed to the next interceptor or target method
        println("Invocation returned")
        return result
    }
}
```

`MethodInvocation`의 `proceed()` 메소드 호출에 주목하십시오.

이것은 인터셉터 체인을 따라 조인 포인트 방향으로 진행합니다.

대부분의 인터셉터는 이 메소드를 호출하고 그 반환 값을 반환합니다. 그러나 `MethodInterceptor`는 모든 around 어드바이스와 마찬가지로 `proceed` 메소드를 호출하는 대신 다른 값을 반환하거나 예외를 던질 수 있습니다.

그러나 타당한 이유 없이 이렇게 해서는 안 됩니다.

`*MethodInterceptor` 구현은 다른 AOP 얼라이언스 호환 AOP 구현과의 상호 운용성을 제공합니다.*

*이 섹션의 나머지 부분에서 논의되는 다른 어드바이스 타입은 일반적인 AOP 개념을 구현하지만 스프링 특정 방식으로 구현합니다.*

*가장 구체적인 어드바이스 타입을 사용하는 데 이점이 있지만, 다른 AOP 프레임워크에서 애스펙트를 실행하고 싶을 가능성이 있다면 `MethodInterceptor` around 어드바이스를 고수하십시오.*

*포인트컷은 현재 프레임워크 간에 상호 운용 가능하지 않으며 AOP 얼라이언스는 현재 포인트컷 인터페이스를 정의하지 않는다는 점에 유의하십시오.*

*Before 어드바이스*
더 간단한 어드바이스 타입은 before 어드바이스입니다. 이것은 메소드 진입 전에만 호출되므로 `MethodInvocation` 객체가 필요하지 않습니다.

before 어드바이스의 주요 장점은 `proceed()` 메소드를 호출할 필요가 없으므로 인터셉터 체인을 따라 진행하는 것을 실수로 실패할 가능성이 없다는 것입니다.

다음 목록은 `MethodBeforeAdvice` 인터페이스를 보여줍니다:

```java
public interface MethodBeforeAdvice extends BeforeAdvice {

	void before(Method method, Object[] args, Object target) throws Throwable;
}
```

반환 타입이 `void`라는 점에 유의하십시오.

Before 어드바이스는 조인 포인트가 실행되기 전에 커스텀 동작을 삽입할 수 있지만 반환 값을 변경할 수는 없습니다.

before 어드바이스가 예외를 던지면 인터셉터 체인의 추가 실행을 중지합니다.

예외는 인터셉터 체인을 따라 다시 전파됩니다.

확인되지 않은(unchecked) 예외이거나 호출된 메소드의 시그니처에 있는 경우 클라이언트에게 직접 전달됩니다.

그렇지 않으면 AOP 프록시에 의해 확인되지 않은 예외로 래핑됩니다.

다음 예제는 모든 메소드 호출을 계산하는 스프링의 before 어드바이스를 보여줍니다:

```java
// Java
import org.springframework.aop.MethodBeforeAdvice;
import java.lang.reflect.Method;

public class CountingBeforeAdvice implements MethodBeforeAdvice {

	private int count;

	@Override
	public void before(Method method, Object[] args, Object target) throws Throwable {
		++count; // Increment count before method execution
	}

	public int getCount() {
		return count;
	}
}
```

```kotlin
// Kotlin
import org.springframework.aop.MethodBeforeAdvice
import java.lang.reflect.Method

class CountingBeforeAdvice : MethodBeforeAdvice {
    var count: Int = 0 // Use var for mutable property
        private set // Make setter private if needed

    @Throws(Throwable::class)
    override fun before(method: Method, args: Array<Any?>, target: Any?) {
        ++count // Increment count before method execution
    }
}
```

Before 어드바이스는 모든 포인트컷과 함께 사용할 수 있습니다.

*Throws 어드바이스*
Throws 어드바이스는 조인 포인트가 예외를 던져 반환된 후 호출됩니다.

스프링은 타입 지정된 throws 어드바이스(typed throws advice)를 제공합니다.

이는 즉, `org.springframework.aop.ThrowsAdvice` 인터페이스가 어떤 메소드도 포함하지 않는다는 것을 의미합니다.

주어진 객체가 하나 이상의 타입 지정된 throws 어드바이스 메소드를 구현함을 식별하는 마커 인터페이스입니다.

이것들은 다음 형식이어야 합니다: `afterThrowing([Method, args, target], subclassOfThrowable)`

마지막 인수만 필수입니다.

어드바이스 메소드가 메소드와 인수에 관심 있는지 여부에 따라 메소드 시그니처는 하나 또는 네 개의 인수를 가질 수 있습니다.

다음 두 목록은 throws 어드바이스의 예인 클래스를 보여줍니다.

다음 어드바이스는 `RemoteException`이 던져지면 호출됩니다 (`RemoteException`의 하위 클래스 포함):

```java
// Java
import org.springframework.aop.ThrowsAdvice;
import java.rmi.RemoteException;

public class RemoteThrowsAdvice implements ThrowsAdvice { // Implement marker interface

	// Method name must be 'afterThrowing'
	// Parameter type determines which exceptions trigger this
	public void afterThrowing(RemoteException ex) throws Throwable {
		// 원격 예외로 무언가 수행
	}
}
```

```kotlin
// Kotlin
import org.springframework.aop.ThrowsAdvice
import java.rmi.RemoteException

class RemoteThrowsAdvice : ThrowsAdvice { // Implement marker interface

    // Method name must be 'afterThrowing'
    // Parameter type determines which exceptions trigger this
    @Throws(Throwable::class) // Declare potential exceptions
    fun afterThrowing(ex: RemoteException) {
        // 원격 예외로 무언가 수행
    }
}
```

앞의 어드바이스와 달리 다음 예제는 호출된 메소드, 메소드 인수 및 대상 객체에 접근할 수 있도록 네 개의 인수를 선언합니다. 다음 어드바이스는 `ServletException`이 던져지면 호출됩니다:

```java
// Java
import org.springframework.aop.ThrowsAdvice;
import java.lang.reflect.Method;
import jakarta.servlet.ServletException; // Assuming Jakarta Servlet API

public class ServletThrowsAdviceWithArguments implements ThrowsAdvice {

	// Method signature with Method, args, target, and specific Exception
	public void afterThrowing(Method method, Object[] args, Object target, ServletException ex) {
		// 모든 인수로 무언가 수행
	}
}
```

```kotlin
// Kotlin
import org.springframework.aop.ThrowsAdvice
import java.lang.reflect.Method
import jakarta.servlet.ServletException // Assuming Jakarta Servlet API

class ServletThrowsAdviceWithArguments : ThrowsAdvice {

    // Method signature with Method, args, target, and specific Exception
    fun afterThrowing(method: Method?, args: Array<Any?>?, target: Any?, ex: ServletException?) {
        // 모든 인수로 무언가 수행
        // Consider null checks for parameters
    }
}
```

마지막 예제는 `RemoteException`과 `ServletException` 모두를 처리하는 단일 클래스에서 이 두 메소드를 어떻게 사용할 수 있는지 보여줍니다.

임의 개수의 throws 어드바이스 메소드를 단일 클래스에 결합할 수 있습니다. 다음 목록은 마지막 예제를 보여줍니다:

```java
// Java
import org.springframework.aop.ThrowsAdvice;
import java.lang.reflect.Method;
import java.rmi.RemoteException;
import jakarta.servlet.ServletException;

public static class CombinedThrowsAdvice implements ThrowsAdvice { // Static inner class example

	// Handles RemoteException
	public void afterThrowing(RemoteException ex) throws Throwable {
		// 원격 예외로 무언가 수행
	}

	// Handles ServletException
	public void afterThrowing(Method m, Object[] args, Object target, ServletException ex) {
		// 모든 인수로 무언가 수행
	}
}
```

```kotlin
// Kotlin
import org.springframework.aop.ThrowsAdvice
import java.lang.reflect.Method
import java.rmi.RemoteException
import jakarta.servlet.ServletException

class CombinedThrowsAdvice : ThrowsAdvice { // Can be a top-level class

    // Handles RemoteException
    @Throws(Throwable::class)
    fun afterThrowing(ex: RemoteException) {
        // 원격 예외로 무언가 수행
    }

    // Handles ServletException
    fun afterThrowing(m: Method?, args: Array<Any?>?, target: Any?, ex: ServletException?) {
        // 모든 인수로 무언가 수행
    }
}
```

*throws-advice 메소드가 자체적으로 예외를 던지면 원본 예외를 오버라이드합니다 (즉, 사용자에게 던져지는 예외를 변경함). 오버라이딩 예외는 일반적으로 모든 메소드 시그니처와 호환되는 `RuntimeException`입니다.*

*그러나 throws-advice 메소드가 확인된(checked) 예외를 던지면 대상 메소드의 선언된 예외와 일치해야 하므로 특정 대상 메소드 시그니처에 어느 정도 결합됩니다.*

*대상 메소드의 시그니처와 호환되지 않는 선언되지 않은 확인된 예외를 던지지 마십시오!*

Throws 어드바이스는 모든 포인트컷과 함께 사용할 수 있습니다.

*After Returning 어드바이스*
스프링의 after returning 어드바이스는 다음 목록에 표시된 `org.springframework.aop.AfterReturningAdvice` 인터페이스를 구현해야 합니다:

```java
public interface AfterReturningAdvice extends Advice {

	void afterReturning(@Nullable Object returnValue, Method method, Object[] args, @Nullable Object target)
			throws Throwable;
}
```

after returning 어드바이스는 반환 값(수정할 수 없음), 호출된 메소드, 메소드의 인수 및 대상에 접근할 수 있습니다.

다음 after returning 어드바이스는 예외를 던지지 않은 모든 성공적인 메소드 호출을 계산합니다:

```java
// Java
import org.springframework.aop.AfterReturningAdvice;
import java.lang.reflect.Method;
import org.springframework.lang.Nullable;

public class CountingAfterReturningAdvice implements AfterReturningAdvice {

	private int count;

	@Override
	public void afterReturning(@Nullable Object returnValue, Method method, Object[] args, @Nullable Object target)
			throws Throwable {
		++count; // Increment count on successful return
	}

	public int getCount() {
		return count;
	}
}
```

```kotlin
// Kotlin
import org.springframework.aop.AfterReturningAdvice
import java.lang.reflect.Method

class CountingAfterReturningAdvice : AfterReturningAdvice {
    var count: Int = 0
        private set

    @Throws(Throwable::class)
    override fun afterReturning(returnValue: Any?, method: Method, args: Array<Any?>, target: Any?) {
        ++count // Increment count on successful return
    }
}
```

이 어드바이스는 실행 경로를 변경하지 않습니다.

예외를 던지면 반환 값 대신 인터셉터 체인을 따라 위로 던져집니다.

After returning 어드바이스는 모든 포인트컷과 함께 사용할 수 있습니다.

*인트로덕션 어드바이스 (Introduction Advice)*
스프링은 인트로덕션 어드바이스(introduction advice)를 특별한 종류의 interception 어드바이스로 취급합니다.

인트로덕션은 `IntroductionAdvisor`와 다음 인터페이스를 구현하는 `IntroductionInterceptor`가 필요합니다:

```java
public interface IntroductionInterceptor extends MethodInterceptor {

	boolean implementsInterface(Class<?> intf);
}
```

AOP 얼라이언스 `MethodInterceptor` 인터페이스에서 상속받은 `invoke()` 메소드는 인트로덕션을 구현해야 합니다.

즉, 호출된 메소드가 도입된(introduced) 인터페이스에 있는 경우, 인트로덕션 인터셉터는 메소드 호출을 처리할 책임이 있습니다 - `proceed()`를 호출할 수 없습니다.

인트로덕션 어드바이스는 메소드 레벨이 아닌 클래스 레벨에서만 적용되므로 모든 포인트컷과 함께 사용할 수 없습니다.

다음과 같은 메소드를 가진 `IntroductionAdvisor`와 함께만 인트로덕션 어드바이스를 사용할 수 있습니다:

```java
public interface IntroductionAdvisor extends Advisor, IntroductionInfo {

	ClassFilter getClassFilter();

	void validateInterfaces() throws IllegalArgumentException;
}

public interface IntroductionInfo {

	Class<?>[] getInterfaces();
}
```

`MethodMatcher`가 없으므로 인트로덕션 어드바이스와 연관된 `Pointcut`도 없습니다.

클래스 필터링만 논리적입니다.

`getInterfaces()` 메소드는 이 어드바이저에 의해 도입된 인터페이스들을 반환합니다.

`validateInterfaces()` 메소드는 도입된 인터페이스가 구성된 `IntroductionInterceptor`에 의해 구현될 수 있는지 여부를 확인하기 위해 내부적으로 사용됩니다.

스프링 테스트 스위트의 예제를 고려하고 다음 인터페이스를 하나 이상의 객체에 도입하고 싶다고 가정해 봅시다:

```java
// Java
public interface Lockable {
	void lock();
	void unlock();
	boolean locked();
}
```

```kotlin
// Kotlin
interface Lockable {
    fun lock()
    fun unlock(): Boolean // Changed to return Boolean like in example below
    fun locked(): Boolean
}
```

이것은 믹스인(mixin)을 보여줍니다.

타입에 관계없이 어드바이스받는 객체를 `Lockable`로 캐스팅하고 `lock` 및 `unlock` 메소드를 호출할 수 있기를 원합니다.

`lock()` 메소드를 호출하면 모든 세터 메소드가 `LockedException`을 던지기를 원합니다.

따라서 객체가 그것에 대해 전혀 알지 못해도 객체를 불변(immutable)으로 만드는 기능을 제공하는 애스펙트를 추가할 수 있습니다: AOP의 좋은 예입니다.

먼저, 힘든 일을 하는 `IntroductionInterceptor`가 필요합니다. 이 경우, `org.springframework.aop.support.DelegatingIntroductionInterceptor` 편의 클래스를 확장합니다.

`IntroductionInterceptor`를 직접 구현할 수도 있지만, 대부분의 경우 `DelegatingIntroductionInterceptor`를 사용하는 것이 가장 좋습니다.

`DelegatingIntroductionInterceptor`는 인트로덕션을 도입된 인터페이스의 실제 구현에 위임하여 그렇게 하기 위해 가로채기(interception) 사용을 은폐하도록 설계되었습니다.

생성자 인수를 사용하여 델리게이트(delegate)를 모든 객체로 설정할 수 있습니다.

기본 델리게이트(인수 없는 생성자 사용 시)는 `this`입니다.

따라서 다음 예제에서 델리게이트는 `DelegatingIntroductionInterceptor`의 `LockMixin` 하위 클래스입니다.

델리게이트(기본적으로 자체)가 주어지면, `DelegatingIntroductionInterceptor` 인스턴스는 델리게이트가 구현하는 모든 인터페이스(`IntroductionInterceptor` 제외)를 찾고 그 중 어느 것에 대해서든 인트로덕션을 지원합니다.

`LockMixin`과 같은 하위 클래스는 노출되어서는 안 되는 인터페이스를 억제(suppress)하기 위해 `suppressInterface(Class intf)` 메소드를 호출할 수 있습니다.

그러나 `IntroductionInterceptor`가 지원할 준비가 된 인터페이스 수에 관계없이 사용된 `IntroductionAdvisor`가 실제로 노출되는 인터페이스를 제어합니다.

도입된 인터페이스는 대상에 의한 동일한 인터페이스 구현을 은폐합니다.

따라서 `LockMixin`은 `DelegatingIntroductionInterceptor`를 확장하고 `Lockable` 자체를 구현합니다.

슈퍼클래스는 `Lockable`이 인트로덕션에 대해 지원될 수 있음을 자동으로 선택하므로 지정할 필요가 없습니다. 이런 식으로 임의 개수의 인터페이스를 도입할 수 있습니다.

`locked` 인스턴스 변수 사용에 주목하십시오. 이는 효과적으로 대상 객체에 보유된 상태에 추가 상태를 더합니다.

다음 예제는 예제 `LockMixin` 클래스를 보여줍니다:

```java
// Java
import org.aopalliance.intercept.MethodInvocation;
import org.springframework.aop.support.DelegatingIntroductionInterceptor;
// Assuming Lockable interface and LockedException class exist
class LockedException extends RuntimeException {}

public class LockMixin extends DelegatingIntroductionInterceptor implements Lockable {

	private boolean locked;

	@Override
	public void lock() {
		this.locked = true;
	}

	@Override
	public boolean unlock() { // Changed to return boolean for consistency
		this.locked = false;
        return true; // Example return
	}

	@Override
	public boolean locked() {
		return this.locked;
	}

	@Override // Override invoke from MethodInterceptor (via DelegatingIntroductionInterceptor)
	public Object invoke(MethodInvocation invocation) throws Throwable {
		if (locked() && invocation.getMethod().getName().startsWith("set")) { // Check if locked and method is a setter
			throw new LockedException();
		}
		// Important: Call super.invoke() to delegate to the target or other interceptors
		return super.invoke(invocation);
	}
}
```

```kotlin
// Kotlin
import org.aopalliance.intercept.MethodInvocation
import org.springframework.aop.support.DelegatingIntroductionInterceptor
// Assuming Lockable interface and LockedException class exist
class LockedException : RuntimeException()

class LockMixin : DelegatingIntroductionInterceptor(), Lockable { // Extend and Implement

    private var locked: Boolean = false

    override fun lock() {
        this.locked = true
    }

    override fun unlock(): Boolean { // Changed to return Boolean
        this.locked = false
        return true // Example return
    }

    override fun locked(): Boolean {
        return this.locked
    }

    @Throws(Throwable::class)
    override fun invoke(invocation: MethodInvocation): Any? { // Override invoke
        if (locked() && invocation.method.name.startsWith("set")) { // Check if locked and method is a setter
            throw LockedException()
        }
        // Important: Call super.invoke()
        return super.invoke(invocation)
    }
}
```

종종 `invoke()` 메소드를 오버라이드할 필요가 없습니다.

`DelegatingIntroductionInterceptor` 구현(메소드가 도입된 경우 델리게이트 메소드를 호출하고, 그렇지 않으면 조인 포인트를 향해 진행함)은 일반적으로 충분합니다.

현재 경우, 검사를 추가해야 합니다: 잠금 모드(locked mode)인 경우 세터 메소드를 호출할 수 없습니다.

필요한 인트로덕션은 고유한 `LockMixin` 인스턴스를 보유하고 도입된 인터페이스(이 경우 `Lockable`만)를 지정하기만 하면 됩니다. 더

복잡한 예제는 인트로덕션 인터셉터(프로토타입으로 정의됨)에 대한 참조를 받을 수 있습니다. 이 경우 `LockMixin`에 관련된 구성이 없으므로 `new`를 사용하여 생성합니다.

다음 예제는 `LockMixinAdvisor` 클래스를 보여줍니다:

```java
// Java
import org.springframework.aop.support.DefaultIntroductionAdvisor;
// Assuming Lockable and LockMixin exist

public class LockMixinAdvisor extends DefaultIntroductionAdvisor {

	// Constructor takes the interceptor (mixin implementation) and the interface to introduce
	public LockMixinAdvisor() {
		super(new LockMixin(), Lockable.class);
	}
}
```

```kotlin
// Kotlin
import org.springframework.aop.support.DefaultIntroductionAdvisor
// Assuming Lockable and LockMixin exist

class LockMixinAdvisor() : DefaultIntroductionAdvisor(LockMixin(), Lockable::class.java) {
    // Constructor automatically calls super
}
```

구성이 필요하지 않으므로 이 어드바이저를 매우 간단하게 적용할 수 있습니다. (그러나 `IntroductionAdvisor` 없이 `IntroductionInterceptor`를 사용하는 것은 불가능합니다.)

인트로덕션과 마찬가지로 어드바이저는 상태 저장(stateful)이므로 인스턴스별이어야 합니다.

각 어드바이스받는 객체에 대해 다른 `LockMixinAdvisor` 인스턴스, 따라서 `LockMixin`이 필요합니다.

어드바이저는 어드바이스받는 객체 상태의 일부를 구성합니다.

`Advised.addAdvisor()` 메소드를 사용하여 프로그래밍 방식으로 또는 (권장되는 방식) 다른 어드바이저처럼 XML 구성에서 이 어드바이저를 적용할 수 있습니다.

"자동 프록시 생성자(auto proxy creators)"를 포함하여 아래에서 논의되는 모든 프록시 생성 선택 사항은 인트로덕션과 상태 저장 믹스인을 올바르게 처리합니다.

---

**전체 주제: 스프링의 어드바이스 API (Advice API in Spring)**

이 부분은 `@Before`, `@After` 등의 어노테이션 대신, **스프링 AOP 프레임워크가 직접 제공하는 특정 인터페이스들(`MethodInterceptor`, `MethodBeforeAdvice`, `ThrowsAdvice` 등)** 을 구현하여 **어드바이스(부가 기능 로직)** 를 만드는 방법을 설명합니다.

**핵심 아이디어:** 어노테이션 없이, 스프링 AOP가 정의한 특정 인터페이스들을 구현하여 어드바이스 종류(Around, Before, After Throwing 등)별 로직을 직접 작성할 수도 있다.

---

**1. 어드바이스 생명주기 (Advice Lifecycles)**

어드바이스(부가 기능 로직을 담은 객체) 자체도 스프링 빈으로 관리됩니다. 이 어드바이스 빈의 생명주기는 두 가지 방식이 가능합니다.

- **클래스별 어드바이스 (Per-class Advice) - 가장 일반적:**
  - **개념:** 하나의 어드바이스 빈 인스턴스가 **여러 대상 객체(Target Object)에 걸쳐 공유**됩니다.
  - **특징:** 어드바이스가 **상태를 가지지 않거나(stateless)**, 모든 대상 객체에 걸쳐 공유되어야 하는 상태를 가질 때 적합합니다. (예: 트랜잭션 관리, 로깅)
  - 스프링 빈의 기본 스코프인 **싱글톤(singleton)** 으로 어드바이스 빈을 정의하면 이 방식이 됩니다.
- **인스턴스별 어드바이스 (Per-instance Advice):**
  - **개념:** 어드바이스가 적용되는 **각 대상 객체마다 별도의 어드바이스 빈 인스턴스**가 생성됩니다.
  - **특징:** 어드바이스가 대상 객체별로 **독립적인 상태를 유지**해야 할 때 사용됩니다. **인트로덕션(Introduction)** 기능(믹스인)을 구현할 때 주로 이 방식이 필요합니다. (어드바이스가 대상 객체에 상태를 추가하므로)
- **혼합 사용:** 하나의 AOP 프록시 내에서 클래스별 어드바이스와 인스턴스별 어드바이스를 함께 사용할 수도 있습니다.

---

**2. 스프링의 어드바이스 타입 (API 기반)**

스프링은 다양한 종류의 어드바이스를 구현하기 위한 인터페이스들을 제공합니다.

- **(1) Interception Around 어드바이스 (`MethodInterceptor`):**
  - **핵심 인터페이스:** `org.aopalliance.intercept.MethodInterceptor` (AOP 얼라이언스 표준 인터페이스)
  - **역할:** `@Around` 어드바이스와 동일하게, 대상 메소드 실행 **전후 모두**에 개입하고 **실행 자체를 제어**할 수 있습니다.
  - **구현 메소드:** `Object invoke(MethodInvocation invocation) throws Throwable;`
    - `MethodInvocation`: 호출된 메소드 정보, 대상 객체, 인수, 그리고 **다음 인터셉터나 대상 메소드로 진행하는 `proceed()` 메소드**를 제공합니다.
    - `invoke()` 메소드는 `invocation.proceed()`를 호출하여 체인을 계속 진행시키고 그 결과를 반환하는 것이 일반적입니다. 하지만 `proceed()`를 호출하지 않거나 다른 값을 반환하거나 예외를 던질 수도 있습니다.
  - **장점:** AOP 얼라이언스 표준이므로 다른 호환 프레임워크와 상호 운용성이 좋습니다.
  - **예시 (`DebugInterceptor`):** 메소드 실행 전후에 간단한 로그를 출력합니다.
- **(2) Before 어드바이스 (`MethodBeforeAdvice`):**
  - **핵심 인터페이스:** `org.springframework.aop.MethodBeforeAdvice`
  - **역할:** `@Before` 어드바이스와 동일하게, 대상 메소드가 실행되기 **전**에만 호출됩니다.
  - **구현 메소드:** `void before(Method method, Object[] args, Object target) throws Throwable;`
    - `method`: 호출된 메소드 정보.
    - `args`: 메소드에 전달된 인수 배열.
    - `target`: 원본 대상 객체.
  - **특징:** 반환 타입이 `void`이므로 대상 메소드의 반환 값을 변경할 수 없습니다. `proceed()` 호출이 없으므로 실수로 체인을 중단시킬 위험이 없습니다.
  - **예시 (`CountingBeforeAdvice`):** 메소드가 호출될 때마다 카운트를 증가시킵니다.
- **(3) Throws 어드바이스 (`ThrowsAdvice`):**
  - **핵심 인터페이스:** `org.springframework.aop.ThrowsAdvice` (마커 인터페이스 - 메소드 없음)
  - **역할:** `@AfterThrowing` 어드바이스와 동일하게, 대상 메소드 실행 중 **예외가 발생**했을 때 호출됩니다.
  - **구현 방법:** 이 인터페이스를 구현하는 클래스 안에 **특정 형식의 메소드를 정의**하는 방식입니다. 메소드 이름은 반드시 **`afterThrowing`** 이어야 하며, 파라미터를 통해 어떤 종류의 예외를 처리할지 결정합니다.
    - `public void afterThrowing(SpecificException ex)`: 특정 예외(`SpecificException` 또는 그 하위 타입)가 발생했을 때 호출됩니다. 예외 객체를 받을 수 있습니다.
    - `public void afterThrowing(Method method, Object[] args, Object target, SpecificException ex)`: 메소드 정보, 인수, 대상 객체 정보까지 추가로 받을 수 있습니다.
  - **특징:** 하나의 클래스 안에 여러 개의 `afterThrowing` 메소드를 정의하여 **다양한 종류의 예외를 처리**할 수 있습니다.
  - **주의:** 이 어드바이스 메소드 자체가 예외를 던지면 원래 예외를 덮어씁니다. 확인된(checked) 예외를 던질 경우 대상 메소드 시그니처와 호환되어야 합니다.
- **(4) After Returning 어드바이스 (`AfterReturningAdvice`):**
  - **핵심 인터페이스:** `org.springframework.aop.AfterReturningAdvice`
  - **역할:** `@AfterReturning` 어드바이스와 동일하게, 대상 메소드가 **예외 없이 정상적으로 값을 반환**했을 때 호출됩니다.
  - **구현 메소드:** `void afterReturning(@Nullable Object returnValue, Method method, Object[] args, @Nullable Object target) throws Throwable;`
    - `returnValue`: 대상 메소드의 반환 값 (수정 불가).
    - `method`, `args`, `target`: 메소드, 인수, 대상 객체 정보.
  - **특징:** 실행 경로를 변경할 수 없습니다. 여기서 예외가 발생하면 원래 반환 값 대신 예외가 위로 전파됩니다.
  - **예시 (`CountingAfterReturningAdvice`):** 성공적인 메소드 호출 횟수를 계산합니다.
- **(5) 인트로덕션 어드바이스 (`IntroductionInterceptor`):**
  - **역할:** `@DeclareParents` (인트로덕션) 기능의 **실제 구현 로직**을 담당하는 특별한 종류의 인터셉션 어드바이스입니다. 대상 객체에 새로운 인터페이스의 동작을 추가합니다.
  - **핵심 인터페이스:** `org.springframework.aop.IntroductionInterceptor` (이 인터페이스는 `MethodInterceptor`를 상속함)
  - **추가 메소드:** `boolean implementsInterface(Class<?> intf)`: 이 인터셉터가 주어진 인터페이스(`intf`)를 구현(도입)하는지 여부를 반환합니다.
  - **`invoke()` 메소드 구현:** 호출된 메소드가 **도입된 인터페이스의 메소드**라면, 이 인터셉터가 직접 처리해야 합니다 (`proceed()` 호출 안 함). 만약 원래 대상 객체의 메소드라면 `proceed()`를 호출하여 다음 인터셉터나 대상에게 넘겨야 합니다.
  - **편의 클래스 (`DelegatingIntroductionInterceptor`):** 직접 `IntroductionInterceptor`를 구현하는 것은 복잡하므로, 스프링은 `DelegatingIntroductionInterceptor`라는 편리한 추상 클래스를 제공합니다.
    - 이 클래스를 상속받고, **도입하려는 인터페이스를 직접 구현**합니다.
    - `invoke()` 메소드는 기본적으로 호출된 메소드가 도입된 인터페이스의 것이면 델리게이트(보통 `this`, 즉 하위 클래스 자신)의 해당 메소드를 호출하고, 아니면 `super.invoke()` (결국 `proceed()`와 유사)를 호출하도록 구현되어 있습니다.
    - 개발자는 주로 도입할 인터페이스의 메소드 로직만 구현하고, 필요하다면 `invoke()`를 오버라이드하여 추가 로직(예: 잠금 상태 확인)을 넣을 수 있습니다.
  - **어드바이저 필요 (`IntroductionAdvisor`):** 인트로덕션 어드바이스는 단독으로 사용될 수 없고, 어떤 클래스에 어떤 인터페이스를 도입할지 정의하는 **`IntroductionAdvisor`** 와 함께 사용되어야 합니다.
    - `DefaultIntroductionAdvisor`: 간단한 인트로덕션 어드바이저 구현체. 생성자에 인트로덕션 인터셉터(믹스인 구현 객체)와 도입할 인터페이스 클래스를 전달합니다.
  - **상태 저장 (Stateful):** 인트로덕션은 대상 객체에 상태를 추가하는 경우가 많으므로, 일반적으로 **인스턴스별 어드바이스(Per-instance Advice)** 로 사용되어야 합니다 (각 대상 객체마다 별도의 어드바이저/인터셉터 인스턴스 필요).

**요약:**

스프링 AOP는 어노테이션 스타일 외에도 `MethodInterceptor`(Around), `MethodBeforeAdvice`(Before), `ThrowsAdvice`(After Throwing), `AfterReturningAdvice`(After Returning), `IntroductionInterceptor`(Introduction)와 같은 **로우 레벨 API 인터페이스**를 통해 어드바이스를 구현하는 방법을 제공합니다. 각 인터페이스는 특정 어드바이스 타입의 동작과 필요한 정보를 정의합니다. Around 어드바이스는 AOP 얼라이언스 표준을 따르며, 인트로덕션은 상태 저장이 필요하여 인스턴스별로 적용되는 경우가 많고 `DelegatingIntroductionInterceptor`와 `IntroductionAdvisor`를 함께 사용하는 것이 일반적입니다. 이 API 기반 방식은 스프링 내부 동작을 이해하거나 다른 AOP 프레임워크와의 호환성이 필요할 때 유용할 수 있지만, 일반적인 애플리케이션 개발에서는 어노테이션 기반 방식이 더 간결하고 편리합니다.
