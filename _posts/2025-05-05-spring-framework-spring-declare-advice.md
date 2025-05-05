---
title: Declaring Advice
description: 
author: laze
date: 2025-05-05 00:00:15 +0900
categories: [Dev, SpringBoot]
tags: [SpringBoot]
---
**Declaring Advice**

어드바이스(Advice)는 포인트컷 표현식(pointcut expression)과 연관되며, 포인트컷에 의해 매치되는 메소드 실행 전, 후 또는 주위(around)에서 실행됩니다.

포인트컷 표현식은 인라인 포인트컷(inline pointcut)이거나 이름 붙여진 포인트컷(named pointcut)에 대한 참조일 수 있습니다.

**Before 어드바이스 (Before Advice)**

`@Before` 어노테이션을 사용하여 애스펙트(aspect)에서 before 어드바이스를 선언할 수 있습니다.

다음 예제는 인라인 포인트컷 표현식을 사용합니다.

```java
// Java
import org.aspectj.lang.annotation.Aspect;
import org.aspectj.lang.annotation.Before;

@Aspect
public class BeforeExample {

	@Before("execution(* com.xyz.dao.*.*(..))") // Pointcut expression directly in the annotation
	public void doAccessCheck() {
		// ... 접근 확인 로직 등
	}
}
```

```kotlin
// Kotlin
import org.aspectj.lang.annotation.Aspect
import org.aspectj.lang.annotation.Before

@Aspect
class BeforeExample {

    @Before("execution(* com.xyz.dao.*.*(..))") // Pointcut expression directly in the annotation
    fun doAccessCheck() {
        // ... 접근 확인 로직 등
    }
}
```

이름 붙여진 포인트컷을 사용하는 경우, 앞의 예제를 다음과 같이 다시 작성할 수 있습니다:

```java
// Java
import org.aspectj.lang.annotation.Aspect;
import org.aspectj.lang.annotation.Before;
// Assuming CommonPointcuts class with dataAccessOperation pointcut exists

@Aspect
public class BeforeExample {

	@Before("com.xyz.CommonPointcuts.dataAccessOperation()") // Referencing a named pointcut
	public void doAccessCheck() {
		// ...
	}
}
```

```kotlin
// Kotlin
import org.aspectj.lang.annotation.Aspect
import org.aspectj.lang.annotation.Before
// Assuming CommonPointcuts class with dataAccessOperation pointcut exists

@Aspect
class BeforeExample {

    @Before("com.xyz.CommonPointcuts.dataAccessOperation()") // Referencing a named pointcut
    fun doAccessCheck() {
        // ...
    }
}
```

**After Returning 어드바이스 (After Returning Advice)**

After returning 어드바이스는 매치된 메소드 실행이 정상적으로 반환될 때 실행됩니다. `@AfterReturning` 어노테이션을 사용하여 선언할 수 있습니다.

```java
// Java
import org.aspectj.lang.annotation.Aspect;
import org.aspectj.lang.annotation.AfterReturning;

@Aspect
public class AfterReturningExample {

	@AfterReturning("execution(* com.xyz.dao.*.*(..))")
	public void doAccessCheck() {
		// ...
	}
}
```

```kotlin
// Kotlin
import org.aspectj.lang.annotation.Aspect
import org.aspectj.lang.annotation.AfterReturning

@Aspect
class AfterReturningExample {

    @AfterReturning("execution(* com.xyz.dao.*.*(..))")
    fun doAccessCheck() {
        // ...
    }
}
```

*동일한 애스펙트 내에 여러 어드바이스 선언(및 다른 멤버들)을 가질 수 있습니다.*

*이 예제들에서는 각각의 효과에 집중하기 위해 단일 어드바이스 선언만 보여줍니다.*

때로는 어드바이스 본문에서 실제 반환된 값에 접근해야 할 필요가 있습니다.

다음 예제와 같이 반환 값을 바인딩하는 `@AfterReturning` 형식을 사용하여 해당 접근 권한을 얻을 수 있습니다:

```java
// Java
import org.aspectj.lang.annotation.Aspect;
import org.aspectj.lang.annotation.AfterReturning;

@Aspect
public class AfterReturningExample {

	@AfterReturning(
		pointcut="execution(* com.xyz.dao.*.*(..))",
		returning="retVal") // Bind return value to 'retVal' parameter
	public void doAccessCheck(Object retVal) { // Parameter name matches 'returning' attribute
		// ... 반환 값(retVal) 사용
	}
}
```

```kotlin
// Kotlin
import org.aspectj.lang.annotation.Aspect
import org.aspectj.lang.annotation.AfterReturning

@Aspect
class AfterReturningExample {

    @AfterReturning(
        pointcut = "execution(* com.xyz.dao.*.*(..))",
        returning = "retVal" // Bind return value to 'retVal' parameter
    )
    fun doAccessCheck(retVal: Any?) { // Parameter name matches 'returning', type can be more specific or Any?
        // ... 반환 값(retVal) 사용
    }
}
```

`returning` 속성에 사용된 이름은 어드바이스 메소드의 파라미터 이름과 일치해야 합니다.

메소드 실행이 반환될 때, 반환 값은 해당 인수 값으로 어드바이스 메소드에 전달됩니다.

`returning` 절은 또한 지정된 타입의 값(이 경우 `Object`이며 모든 반환 값과 일치함)을 반환하는 메소드 실행에만 매칭을 제한합니다.

*after returning 어드바이스를 사용할 때는 완전히 다른 참조를 반환하는 것이 불가능하다는 점에 유의하십시오.*

**After Throwing 어드바이스 (After Throwing Advice)**

After throwing 어드바이스는 매치된 메소드 실행이 예외를 던져 종료될 때 실행됩니다. 다음 예제와 같이 `@AfterThrowing` 어노테이션을 사용하여 선언할 수 있습니다:

```java
// Java
import org.aspectj.lang.annotation.Aspect;
import org.aspectj.lang.annotation.AfterThrowing;

@Aspect
public class AfterThrowingExample {

	@AfterThrowing("execution(* com.xyz.dao.*.*(..))")
	public void doRecoveryActions() {
		// ... 복구 로직 등
	}
}
```

```kotlin
// Kotlin
import org.aspectj.lang.annotation.Aspect
import org.aspectj.lang.annotation.AfterThrowing

@Aspect
class AfterThrowingExample {

    @AfterThrowing("execution(* com.xyz.dao.*.*(..))")
    fun doRecoveryActions() {
        // ... 복구 로직 등
    }
}
```

종종 주어진 타입의 예외가 던져질 때만 어드바이스가 실행되기를 원하며, 어드바이스 본문에서 던져진 예외에 접근해야 할 필요도 종종 있습니다.

`throwing` 속성을 사용하여 매칭을 제한하고(원하는 경우 - 그렇지 않으면 예외 타입으로 `Throwable` 사용) 던져진 예외를 어드바이스 파라미터에 바인딩할 수 있습니다. 다음 예제는 이를 수행하는 방법을 보여줍니다:

```java
// Java
import org.aspectj.lang.annotation.Aspect;
import org.aspectj.lang.annotation.AfterThrowing;
import org.springframework.dao.DataAccessException; // Specific exception type

@Aspect
public class AfterThrowingExample {

	@AfterThrowing(
		pointcut="execution(* com.xyz.dao.*.*(..))",
		throwing="ex") // Bind exception to 'ex' parameter
	public void doRecoveryActions(DataAccessException ex) { // Parameter name matches 'throwing', type restricts match
		// ... 예외(ex) 사용
	}
}
```

```kotlin
// Kotlin
import org.aspectj.lang.annotation.Aspect
import org.aspectj.lang.annotation.AfterThrowing
import org.springframework.dao.DataAccessException // Specific exception type

@Aspect
class AfterThrowingExample {

    @AfterThrowing(
        pointcut = "execution(* com.xyz.dao.*.*(..))",
        throwing = "ex" // Bind exception to 'ex' parameter
    )
    fun doRecoveryActions(ex: DataAccessException) { // Parameter name matches 'throwing', type restricts match
        // ... 예외(ex) 사용
    }
}
```

`throwing` 속성에 사용된 이름은 어드바이스 메소드의 파라미터 이름과 일치해야 합니다.

메소드 실행이 예외를 던져 종료될 때, 예외는 해당 인수 값으로 어드바이스 메소드에 전달됩니다.

`throwing` 절은 또한 지정된 타입(`DataAccessException`, 이 경우)의 예외를 던지는 메소드 실행에만 매칭을 제한합니다.

`*@AfterThrowing`은 일반적인 예외 처리 콜백을 나타내지 않는다는 점에 유의하십시오.*

*구체적으로, `@AfterThrowing` 어드바이스 메소드는 조인 포인트(사용자 선언 대상 메소드) 자체에서 발생하는 예외만 받아야 하며, 동반되는 `@After`/`@AfterReturning` 메소드에서 발생하는 예외는 받지 않습니다.*

**After (Finally) 어드바이스 (After (Finally) Advice)**

After (finally) 어드바이스는 매치된 메소드 실행이 종료될 때 실행됩니다.

`@After` 어노테이션을 사용하여 선언됩니다. After 어드바이스는 정상 및 예외 반환 조건 모두를 처리할 준비가 되어 있어야 합니다.

일반적으로 리소스 해제 및 유사한 목적에 사용됩니다. 다음 예제는 after finally 어드바이스 사용 방법을 보여줍니다:

```java
// Java
import org.aspectj.lang.annotation.Aspect;
import org.aspectj.lang.annotation.After;

@Aspect
public class AfterFinallyExample {

	@After("execution(* com.xyz.dao.*.*(..))") // Runs regardless of outcome
	public void doReleaseLock() {
		// ... 락 해제 등
	}
}
```

```kotlin
// Kotlin
import org.aspectj.lang.annotation.Aspect
import org.aspectj.lang.annotation.After

@Aspect
class AfterFinallyExample {

    @After("execution(* com.xyz.dao.*.*(..))") // Runs regardless of outcome
    fun doReleaseLock() {
        // ... 락 해제 등
    }
}
```

*AspectJ의 `@After` 어드바이스는 `try-catch` 문의 `finally` 블록과 유사하게 "after finally 어드바이스"로 정의된다는 점에 유의하십시오.*

*이는 성공적인 정상 반환에만 적용되는 `@AfterReturning`과 대조적으로, 조인 포인트(사용자 선언 대상 메소드)에서 발생하는 정상 반환 또는 예외 발생 등 모든 결과에 대해 호출됩니다.*

**Around 어드바이스 (Around Advice)**

마지막 종류의 어드바이스는 around 어드바이스입니다. Around 어드바이스는 매치된 메소드 실행 "주위(around)"에서 실행됩니다.

메소드 실행 전후에 작업을 수행하고, 메소드가 실제로 실행될 시기, 방법, 심지어 실행 여부까지 결정할 기회를 가집니다.

Around 어드바이스는 스레드 안전(thread-safe) 방식으로 메소드 실행 전후에 상태를 공유해야 하는 경우(예: 타이머 시작 및 중지) 종종 사용됩니다.

*항상 요구 사항을 충족하는 가장 덜 강력한 형태의 어드바이스를 사용하십시오.*

*예를 들어, before 어드바이스가 요구 사항에 충분하다면 around 어드바이스를 사용하지 마십시오.*

Around 어드바이스는 메소드에 `@Around` 어노테이션을 달아 선언됩니다.

메소드는 반환 타입으로 `Object`를 선언해야 하며, 메소드의 첫 번째 파라미터는 `ProceedingJoinPoint` 타입이어야 합니다.

어드바이스 메소드 본문 내에서 기본 메소드가 실행되도록 하려면 `ProceedingJoinPoint`에서 `proceed()`를 호출해야 합니다.

인수를 사용하지 않고 `proceed()`를 호출하면 호출자의 원본 인수가 호출될 때 기본 메소드에 제공됩니다.

고급 사용 사례를 위해, 인수 배열(`Object[]`)을 받는 `proceed()` 메소드의 오버로드된 변형이 있습니다.

배열의 값은 호출될 때 기본 메소드의 인수로 사용됩니다.

`*Object[]`로 호출될 때 `proceed`의 동작은 AspectJ 컴파일러에 의해 컴파일된 around 어드바이스의 `proceed` 동작과 약간 다릅니다.*

*전통적인 AspectJ 언어를 사용하여 작성된 around 어드바이스의 경우, `proceed`에 전달되는 인수의 수는 around 어드바이스에 전달되는 인수의 수(기본 조인 포인트가 받는 인수의 수가 아님)와 일치해야 하며,*

*주어진 인수 위치에서 `proceed`에 전달된 값은 값이 바인딩된 엔티티에 대한 조인 포인트의 원본 값을 대체합니다 .*

*스프링이 취하는 접근 방식은 더 간단하며 프록시 기반, 실행 전용 의미론과 더 잘 일치합니다.*

*스프링용으로 작성된 `@AspectJ` 애스펙트를 컴파일하고 AspectJ 컴파일러 및 위버와 함께 인수를 사용하여 `proceed`를 사용하는 경우에만 이 차이점을 인지해야 합니다.*

*스프링 AOP와 AspectJ 모두에서 100% 호환되는 방식으로 이러한 애스펙트를 작성하는 방법이 있으며, 이는 어드바이스 파라미터에 대한 다음 섹션에서 논의됩니다.*

around 어드바이스가 반환하는 값은 메소드 호출자가 보는 반환 값입니다.

예를 들어, 간단한 캐싱 애스펙트는 캐시에 값이 있으면 캐시에서 값을 반환하거나, 없으면 `proceed()`를 호출(하고 해당 값을 반환)할 수 있습니다.

`proceed`는 around 어드바이스 본문 내에서 한 번, 여러 번 또는 전혀 호출되지 않을 수 있다는 점에 유의하십시오. 이 모든 것이 합법적입니다.

*around 어드바이스 메소드의 반환 타입을 `void`로 선언하면 호출자에게 항상 `null`이 반환되어 `proceed()` 호출 결과가 효과적으로 무시됩니다.*

*따라서 around 어드바이스 메소드는 `Object`의 반환 타입을 선언하는 것이 좋습니다.*

*어드바이스 메소드는 일반적으로 기본 메소드의 반환 타입이 `void`이더라도 `proceed()` 호출에서 반환된 값을 반환해야 합니다.*

*그러나 어드바이스는 사용 사례에 따라 선택적으로 캐시된 값, 래핑된 값 또는 다른 값을 반환할 수 있습니다.*

다음 예제는 around 어드바이스 사용 방법을 보여줍니다:

```java
// Java
import org.aspectj.lang.ProceedingJoinPoint;
import org.aspectj.lang.annotation.Around;
import org.aspectj.lang.annotation.Aspect;
import org.springframework.util.StopWatch; // Example for timing

@Aspect
public class AroundExample {

	@Around("execution(* com.xyz..service.*.*(..))") // Pointcut for service methods
	public Object doBasicProfiling(ProceedingJoinPoint pjp) throws Throwable {
		// 타이머 시작
        StopWatch sw = new StopWatch(getClass().getSimpleName());
        try {
            sw.start(pjp.getSignature().getName());
            Object retVal = pjp.proceed(); // Proceed with the original method execution
            return retVal;
        } finally {
            sw.stop();
            System.out.println(sw.prettyPrint()); // 타이머 중지 및 출력
        }
	}
}
```

```kotlin
// Kotlin
import org.aspectj.lang.ProceedingJoinPoint
import org.aspectj.lang.annotation.Around
import org.aspectj.lang.annotation.Aspect
import org.springframework.util.StopWatch // Example for timing

@Aspect
class AroundExample {

    @Around("execution(* com.xyz..service.*.*(..))") // Pointcut for service methods
    @Throws(Throwable::class) // Declare potential exceptions from proceed()
    fun doBasicProfiling(pjp: ProceedingJoinPoint): Any? { // Return type Any?
        // 타이머 시작
        val sw = StopWatch(javaClass.simpleName)
        return try {
            sw.start(pjp.signature.name)
            pjp.proceed() // Proceed with the original method execution
        } finally {
            sw.stop()
            println(sw.prettyPrint()) // 타이머 중지 및 출력
        }
    }
}
```

**어드바이스 파라미터 (Advice Parameters)**

스프링은 완전한 타입 지정 어드바이스(fully typed advice)를 제공합니다.

즉, 항상 `Object[]` 배열로 작업하는 대신 어드바이스 시그니처에서 필요한 파라미터를 선언합니다 (반환 및 throwing 예제에서 앞서 보았듯이).

이 섹션의 뒷부분에서 인수 및 기타 컨텍스트 값을 어드바이스 본문에서 사용 가능하게 만드는 방법을 봅니다.

먼저, 현재 어드바이스가 어드바이스하고 있는 메소드에 대해 알 수 있는 일반적인 어드바이스를 작성하는 방법을 살펴보겠습니다.

*현재 JoinPoint에 접근하기 (Access to the Current JoinPoint)*
모든 어드바이스 메소드는 첫 번째 파라미터로 `org.aspectj.lang.JoinPoint` 타입의 파라미터를 선언할 수 있습니다.

around 어드바이스는 `JoinPoint`의 하위 클래스인 `ProceedingJoinPoint` 타입의 첫 번째 파라미터를 선언해야 한다는 점에 유의하십시오.

`JoinPoint` 인터페이스는 여러 유용한 메소드를 제공합니다:

- `getArgs()`: 메소드 인수를 반환합니다.
- `getThis()`: 프록시 객체를 반환합니다.
- `getTarget()`: 대상 객체를 반환합니다.
- `getSignature()`: 어드바이스받는 메소드의 설명을 반환합니다.
- `toString()`: 어드바이스받는 메소드의 유용한 설명을 출력합니다.

*어드바이스에 파라미터 전달하기 (Passing Parameters to Advice)*
반환 값 또는 예외 값을 바인딩하는 방법(after returning 및 after throwing 어드바이스 사용)은 이미 보았습니다.

인수 값을 어드바이스 본문에서 사용 가능하게 만들려면 `args`의 바인딩 형태를 사용할 수 있습니다.

`args` 표현식에서 타입 이름 대신 파라미터 이름을 사용하면, 어드바이스가 호출될 때 해당 인수의 값이 파라미터 값으로 전달됩니다.

첫 번째 파라미터로 `Account` 객체를 받는 DAO 작업 실행을 어드바이스하고 어드바이스 본문에서 계정(account)에 접근해야 한다고 가정해 봅시다.

```java
// Java
import org.aspectj.lang.annotation.Aspect;
import org.aspectj.lang.annotation.Before;
// Assuming Account class exists

@Aspect
public class ParameterPassingExample {

    // Matches methods in DAO packages taking at least one arg, where the first is Account
    // Binds the first argument (which must be an Account) to the 'account' parameter
    @Before("execution(* com.xyz.dao.*.*(..)) && args(account,..)")
    public void validateAccount(Account account) { // Parameter name matches 'account' in args()
        // ... 계정(account) 사용
    }
}
```

```kotlin
// Kotlin
import org.aspectj.lang.annotation.Aspect
import org.aspectj.lang.annotation.Before
// Assuming Account data class exists

@Aspect
class ParameterPassingExample {

    // Matches methods in DAO packages taking at least one arg, where the first is Account
    // Binds the first argument (which must be an Account) to the 'account' parameter
    @Before("execution(* com.xyz.dao.*.*(..)) && args(account,..)")
    fun validateAccount(account: Account) { // Parameter name matches 'account' in args()
        // ... 계정(account) 사용
    }
}
```

포인트컷 표현식의 `args(account, ..)` 부분은 두 가지 목적을 제공합니다.

첫째, 메소드가 최소한 하나의 파라미터를 받고 해당 파라미터에 전달된 인수가 `Account`의 인스턴스인 메소드 실행에만 매칭을 제한합니다.

둘째, `account` 파라미터를 통해 실제 `Account` 객체를 어드바이스에서 사용 가능하게 만듭니다.

이를 작성하는 또 다른 방법은 조인 포인트와 일치할 때 `Account` 객체 값을 "제공하는" 포인트컷을 선언한 다음, 어드바이스에서 이름 붙여진 포인트컷을 참조하는 것입니다.

```java
// Java
import org.aspectj.lang.annotation.Aspect;
import org.aspectj.lang.annotation.Before;
import org.aspectj.lang.annotation.Pointcut;
// Assuming Account class exists

@Aspect
public class ParameterBindingExample {

    // Pointcut that matches the execution and binds the Account argument
    @Pointcut("execution(* com.xyz.dao.*.*(..)) && args(account,..)")
    private void accountDataAccessOperation(Account account) {} // Parameter binding in pointcut

    @Before("accountDataAccessOperation(account)") // Reference named pointcut, pass bound arg
    public void validateAccount(Account account) { // Receive the bound argument
        // ...
    }
}
```

```kotlin
// Kotlin
import org.aspectj.lang.annotation.Aspect
import org.aspectj.lang.annotation.Before
import org.aspectj.lang.annotation.Pointcut
// Assuming Account data class exists

@Aspect
class ParameterBindingExample {

    // Pointcut that matches the execution and binds the Account argument
    @Pointcut("execution(* com.xyz.dao.*.*(..)) && args(account,..)")
    private fun accountDataAccessOperation(account: Account) {} // Parameter binding in pointcut

    @Before("accountDataAccessOperation(account)") // Reference named pointcut, pass bound arg
    fun validateAccount(account: Account) { // Receive the bound argument
        // ...
    }
}
```

프록시 객체(`this`), 대상 객체(`target`), 어노테이션(`@within`, `@target`, `@annotation`, `@args`)은 모두 유사한 방식으로 바인딩될 수 있습니다.

다음 예제 세트는 `@Auditable` 어노테이션으로 어노테이션된 메소드 실행을 매치하고 감사 코드(audit code)를 추출하는 방법을 보여줍니다:

다음은 `@Auditable` 어노테이션의 정의를 보여줍니다:

```java
// Java
import java.lang.annotation.*;
// Assuming AuditCode enum exists
enum AuditCode { CODE1, CODE2 }

@Retention(RetentionPolicy.RUNTIME)
@Target(ElementType.METHOD)
public @interface Auditable {
	AuditCode value();
}
```

```kotlin
// Kotlin
import kotlin.annotation.AnnotationRetention.RUNTIME
import kotlin.annotation.AnnotationTarget.FUNCTION // Use FUNCTION for methods
// Assuming AuditCode enum exists
enum class AuditCode { CODE1, CODE2 }

@Retention(RUNTIME)
@Target(FUNCTION)
annotation class Auditable(val value: AuditCode)
```

다음은 `@Auditable` 메소드 실행과 일치하는 어드바이스를 보여줍니다:

```java
// Java
import org.aspectj.lang.annotation.Aspect;
import org.aspectj.lang.annotation.Before;
// Assuming Auditable annotation and Pointcuts class exist

@Aspect
public class AnnotationBindingExample {

    // Match public methods with @Auditable annotation, bind the annotation instance
    @Before("com.xyz.Pointcuts.publicMethod() && @annotation(auditable)") // ①
    public void audit(Auditable auditable) { // ② Parameter name matches annotation binding variable
        AuditCode code = auditable.value(); // Access annotation value
        // ... 감사 로직
    }
}
```

① `Combining Pointcut Expressions`에서 정의된 `publicMethod` 이름 붙여진 포인트컷을 참조합니다.
② 어노테이션 인스턴스를 `auditable` 파라미터에 바인딩합니다.

```kotlin
// Kotlin
import org.aspectj.lang.annotation.Aspect
import org.aspectj.lang.annotation.Before
// Assuming Auditable annotation and Pointcuts class exist

@Aspect
class AnnotationBindingExample {

    // Match public methods with @Auditable annotation, bind the annotation instance
    @Before("com.xyz.Pointcuts.publicMethod() && @annotation(auditable)") // ①
    fun audit(auditable: Auditable) { // ② Parameter name matches annotation binding variable
        val code = auditable.value // Access annotation value using property syntax
        // ... 감사 로직
    }
}
```

*어드바이스 파라미터 및 제네릭 (Advice Parameters and Generics)*
스프링 AOP는 클래스 선언 및 메소드 파라미터에 사용된 제네릭을 처리할 수 있습니다. 다음과 같은 제네릭 타입이 있다고 가정해 봅시다:

```java
// Java
import java.util.Collection;

public interface Sample<T> {
	void sampleGenericMethod(T param);
	void sampleGenericCollectionMethod(Collection<T> param);
}
```

```kotlin
// Kotlin
interface Sample<T> {
    fun sampleGenericMethod(param: T)
    fun sampleGenericCollectionMethod(param: Collection<T>)
}

```

어드바이스 파라미터를 가로채려는 파라미터 타입에 연결하여 메소드 타입의 가로채기를 특정 파라미터 타입으로 제한할 수 있습니다:

```java
// Java
import org.aspectj.lang.annotation.Aspect;
import org.aspectj.lang.annotation.Before;
// Assuming Sample interface and MyType class exist

class MyType {} // Example type

@Aspect
public class GenericParameterBindingExample {

    // Intercept sampleGenericMethod when the parameter is MyType
    @Before("execution(* ..Sample+.sampleGenericMethod(*)) && args(param)")
    public void beforeSampleMethod(MyType param) { // Advice parameter tied to specific type
        // 어드바이스 구현
        System.out.println("Intercepted method with MyType parameter: " + param);
    }
}
```

```kotlin
// Kotlin
import org.aspectj.lang.annotation.Aspect
import org.aspectj.lang.annotation.Before
// Assuming Sample interface and MyType class exist

class MyType // Example type

@Aspect
class GenericParameterBindingExample {

    // Intercept sampleGenericMethod when the parameter is MyType
    @Before("execution(* ..Sample+.sampleGenericMethod(*)) && args(param)")
    fun beforeSampleMethod(param: MyType) { // Advice parameter tied to specific type
        // 어드바이스 구현
        println("Intercepted method with MyType parameter: $param")
    }
}
```

이 접근 방식은 제네릭 컬렉션에는 작동하지 않습니다. 따라서 다음과 같이 포인트컷을 정의할 수 없습니다:

```java
// Java - This does NOT work as intended for generic collections
@Before("execution(* ..Sample+.sampleGenericCollectionMethod(*)) && args(param)")
public void beforeSampleMethod(Collection<MyType> param) {
	// Advice implementation
}
```

```kotlin
// Kotlin - This does NOT work as intended for generic collections
@Before("execution(* ..Sample+.sampleGenericCollectionMethod(*)) && args(param)")
fun beforeSampleMethod(param: Collection<MyType>) {
    // Advice implementation
}
```

이것이 작동하게 하려면 컬렉션의 모든 요소를 검사해야 하는데, 이는 일반적으로 `null` 값을 어떻게 처리할지 결정할 수 없기 때문에 합리적이지 않습니다.

이와 유사한 것을 달성하려면 파라미터를 `Collection<?>`으로 타입 지정하고 요소의 타입을 수동으로 확인해야 합니다.

*인수 이름 결정하기 (Determining Argument Names)*
어드바이스 호출에서의 파라미터 바인딩은 포인트컷 표현식에서 사용된 이름과 어드바이스 및 포인트컷 메소드 시그니처에 선언된 파라미터 이름을 매칭하는 것에 의존합니다.

*이 섹션에서는 AspectJ API가 파라미터 이름을 인수 이름으로 참조하므로 인수와 파라미터 용어를 혼용하여 사용합니다.*

스프링 AOP는 파라미터 이름을 결정하기 위해 다음 `ParameterNameDiscoverer` 구현들을 사용합니다.

각 디스커버러는 파라미터 이름을 발견할 기회를 가지며, 첫 번째 성공적인 디스커버러가 우선합니다.

등록된 디스커버러 중 어느 것도 파라미터 이름을 결정할 수 없는 경우 예외가 발생합니다.

1. **`AspectJAnnotationParameterNameDiscoverer`**
   해당 어드바이스 또는 포인트컷 어노테이션의 `argNames` 속성을 통해 사용자가 명시적으로 지정한 파라미터 이름을 사용합니다.
2. **`KotlinReflectionParameterNameDiscoverer`**
   Kotlin 리플렉션 API를 사용하여 파라미터 이름을 결정합니다. 이 디스커버러는 해당 API가 클래스패스에 있는 경우에만 사용됩니다.
3. **`StandardReflectionParameterNameDiscoverer`**
   표준 `java.lang.reflect.Parameter` API를 사용하여 파라미터 이름을 결정합니다. 코드가 `javac`의 `parameters` 플래그로 컴파일되어야 합니다. Java 8+에서 권장되는 접근 방식입니다.
4. **`AspectJAdviceParameterNameDiscoverer`**
   포인트컷 표현식, `returning` 및 `throwing` 절에서 파라미터 이름을 추론합니다. 사용된 알고리즘에 대한 자세한 내용은 javadoc을 참조하십시오.

*명시적 인수 이름 (Explicit Argument Names)*`@AspectJ` 어드바이스 및 포인트컷 어노테이션에는 어노테이션된 메소드의 인수 이름을 지정하는 데 사용할 수 있는 선택적 `argNames` 속성이 있습니다.

`*@AspectJ` 애스펙트가 디버그 정보 없이도 AspectJ 컴파일러(ajc)에 의해 컴파일된 경우, 컴파일러가 필요한 정보를 유지하므로 `argNames` 속성을 추가할 필요가 없습니다.*

*유사하게, `@AspectJ` 애스펙트가 `-parameters` 플래그를 사용하여 `javac`로 컴파일된 경우, 컴파일러가 필요한 정보를 유지하므로 `argNames` 속성을 추가할 필요가 없습니다.*

다음 예제는 `argNames` 속성 사용 방법을 보여줍니다:

```java
// Java
import org.aspectj.lang.annotation.Aspect;
import org.aspectj.lang.annotation.Before;
// Assuming Pointcuts, Auditable exist

@Aspect
public class ArgNamesExample {

    // Bind target object to 'bean', annotation to 'auditable'
    @Before(
        value = "com.xyz.Pointcuts.publicMethod() && target(bean) && @annotation(auditable)", // ①
        argNames = "bean,auditable") // ② Explicitly name the bound parameters
    public void audit(Object bean, Auditable auditable) { // Parameter names match argNames
        AuditCode code = auditable.value();
        // ... code 및 bean 사용
    }
}

```

① `Combining Pointcut Expressions`에서 정의된 `publicMethod` 이름 붙여진 포인트컷을 참조합니다.
② `bean`과 `auditable`을 인수 이름으로 선언합니다.

```kotlin
// Kotlin
import org.aspectj.lang.annotation.Aspect
import org.aspectj.lang.annotation.Before
// Assuming Pointcuts, Auditable exist

@Aspect
class ArgNamesExample {

    // Bind target object to 'bean', annotation to 'auditable'
    @Before(
        value = "com.xyz.Pointcuts.publicMethod() && target(bean) && @annotation(auditable)", // ①
        argNames = "bean,auditable" // ② Explicitly name the bound parameters
    )
    fun audit(bean: Any, auditable: Auditable) { // Parameter names match argNames
        val code = auditable.value
        // ... code 및 bean 사용
    }
}
```

첫 번째 파라미터가 `JoinPoint`, `ProceedingJoinPoint` 또는 `JoinPoint.StaticPart` 타입인 경우, `argNames` 속성 값에서 해당 파라미터 이름을 생략할 수 있습니다.

예를 들어, 조인 포인트 객체를 받도록 앞의 어드바이스를 수정하면 `argNames` 속성에 이를 포함할 필요가 없습니다:

```java
// Java
import org.aspectj.lang.JoinPoint; // Import JoinPoint
import org.aspectj.lang.annotation.Aspect;
import org.aspectj.lang.annotation.Before;
// Assuming Pointcuts, Auditable exist

@Aspect
public class ArgNamesWithJoinPointExample {

    @Before(
        value = "com.xyz.Pointcuts.publicMethod() && target(bean) && @annotation(auditable)", // ①
        argNames = "bean,auditable") // ② jp is omitted from argNames
    public void audit(JoinPoint jp, Object bean, Auditable auditable) { // jp is the first parameter
        AuditCode code = auditable.value();
        // ... code, bean, jp 사용
    }
}
```

① `Combining Pointcut Expressions`에서 정의된 `publicMethod` 이름 붙여진 포인트컷을 참조합니다.
② `bean`과 `auditable`을 인수 이름으로 선언합니다.

```kotlin
// Kotlin
import org.aspectj.lang.JoinPoint // Import JoinPoint
import org.aspectj.lang.annotation.Aspect
import org.aspectj.lang.annotation.Before
// Assuming Pointcuts, Auditable exist

@Aspect
class ArgNamesWithJoinPointExample {

    @Before(
        value = "com.xyz.Pointcuts.publicMethod() && target(bean) && @annotation(auditable)", // ①
        argNames = "bean,auditable" // ② jp is omitted from argNames
    )
    fun audit(jp: JoinPoint, bean: Any, auditable: Auditable) { // jp is the first parameter
        val code = auditable.value
        // ... code, bean, jp 사용
    }
}
```

`JoinPoint`, `ProceedingJoinPoint` 또는 `JoinPoint.StaticPart` 타입의 첫 번째 파라미터에 주어지는 특별한 처리는 다른 조인 포인트 컨텍스트를 수집하지 않는 어드바이스 메소드에 특히 편리합니다.

이러한 상황에서는 `argNames` 속성을 생략할 수 있습니다. 예를 들어, 다음 어드바이스는 `argNames` 속성을 선언할 필요가 없습니다:

```java
// Java
import org.aspectj.lang.JoinPoint;
import org.aspectj.lang.annotation.Aspect;
import org.aspectj.lang.annotation.Before;
// Assuming Pointcuts exist

@Aspect
public class SimpleAuditExample {

    @Before("com.xyz.Pointcuts.publicMethod()") // ① No other context needed besides JoinPoint
    public void audit(JoinPoint jp) { // argNames attribute omitted
        // ... jp 사용
    }
}
```

① `Combining Pointcut Expressions`에서 정의된 `publicMethod` 이름 붙여진 포인트컷을 참조합니다.

```kotlin
// Kotlin
import org.aspectj.lang.JoinPoint
import org.aspectj.lang.annotation.Aspect
import org.aspectj.lang.annotation.Before
// Assuming Pointcuts exist

@Aspect
class SimpleAuditExample {

    @Before("com.xyz.Pointcuts.publicMethod()") // ① No other context needed besides JoinPoint
    fun audit(jp: JoinPoint) { // argNames attribute omitted
        // ... jp 사용
    }
}
```

*인수로 진행하기 (Proceeding with Arguments)*
스프링 AOP와 AspectJ에서 일관되게 작동하는 인수를 사용한 `proceed` 호출을 작성하는 방법을 설명하겠다고 앞서 언급했습니다.

해결책은 어드바이스 시그니처가 각 메소드 파라미터를 순서대로 바인딩하도록 보장하는 것입니다.

```java
// Java
import org.aspectj.lang.ProceedingJoinPoint;
import org.aspectj.lang.annotation.Around;
import org.aspectj.lang.annotation.Aspect;
import java.util.List;
// Assuming Account, CommonPointcuts exist

@Aspect
public class ArgumentProceedExample {

    // Match find methods in data access layer taking one String argument
    @Around("execution(List<Account> find*(..)) && " +
            "com.xyz.CommonPointcuts.inDataAccessLayer() && " + // ① Reference shared pointcut
            "args(accountHolderNamePattern)") // Bind the argument
    public Object preProcessQueryPattern(ProceedingJoinPoint pjp,
            String accountHolderNamePattern) throws Throwable { // Receive bound argument
        String newPattern = preProcess(accountHolderNamePattern); // Modify the argument
        return pjp.proceed(new Object[] {newPattern}); // Proceed with modified argument
    }

    private String preProcess(String pattern) {
        // Example preprocessing
        return pattern.toUpperCase();
    }
}
```

① `Sharing Named Pointcut Definitions`에서 정의된 `inDataAccessLayer` 이름 붙여진 포인트컷을 참조합니다.

```kotlin
// Kotlin
import org.aspectj.lang.ProceedingJoinPoint
import org.aspectj.lang.annotation.Around
import org.aspectj.lang.annotation.Aspect
// Assuming Account, CommonPointcuts exist

@Aspect
class ArgumentProceedExample {

    // Match find methods in data access layer taking one String argument
    @Around(
        "execution(List<Account> find*(..)) && " +
                "com.xyz.CommonPointcuts.inDataAccessLayer() && " + // ① Reference shared pointcut
                "args(accountHolderNamePattern)" // Bind the argument
    )
    @Throws(Throwable::class)
    fun preProcessQueryPattern(pjp: ProceedingJoinPoint,
                               accountHolderNamePattern: String): Any? { // Receive bound argument
        val newPattern = preProcess(accountHolderNamePattern) // Modify the argument
        return pjp.proceed(arrayOf<Any>(newPattern)) // Proceed with modified argument array
    }

    private fun preProcess(pattern: String): String {
        // Example preprocessing
        return pattern.uppercase()
    }
}
```

많은 경우, (앞의 예제처럼) 어쨌든 이 바인딩을 수행합니다.

**어드바이스 순서 지정 (Advice Ordering)**

여러 어드바이스 조각이 모두 동일한 조인 포인트에서 실행되기를 원할 때 어떤 일이 발생할까요? 스프링 AOP는 어드바이스 실행 순서를 결정하기 위해 AspectJ와 동일한 우선순위 규칙을 따릅니다.

가장 높은 우선순위의 어드바이스가 "진입 시(on the way in)" 먼저 실행됩니다 (따라서, 두 개의 before 어드바이스가 주어진 경우 가장 높은 우선순위를 가진 것이 먼저 실행됨).

조인 포인트에서 "나갈 때(On the way out)", 가장 높은 우선순위의 어드바이스가 마지막에 실행됩니다 (따라서, 두 개의 after 어드바이스가 주어진 경우 가장 높은 우선순위를 가진 것이 두 번째로 실행됨).

다른 애스펙트에 정의된 두 개의 어드바이스 조각이 모두 동일한 조인 포인트에서 실행되어야 할 때, 달리 지정하지 않는 한 실행 순서는 정의되지 않습니다.

우선순위를 지정하여 실행 순서를 제어할 수 있습니다.

이는 애스펙트 클래스에서 `org.springframework.core.Ordered` 인터페이스를 구현하거나 `@Order` 어노테이션으로 어노테이션하는 일반적인 스프링 방식으로 수행됩니다.

두 애스펙트가 주어진 경우, `Ordered.getOrder()`(또는 어노테이션 값)에서 더 낮은 값을 반환하는 애스펙트가 더 높은 우선순위를 가집니다.

*특정 애스펙트의 각 고유한 어드바이스 타입은 개념적으로 조인 포인트에 직접 적용되도록 의도되었습니다.*

*결과적으로, `@AfterThrowing` 어드바이스 메소드는 동반되는 `@After`/`@AfterReturning` 메소드에서 예외를 받아서는 안 됩니다.*

동일한 `@Aspect` 클래스에 정의된 동일한 타입의 두 어드바이스 조각(예: 두 개의 `@After` 어드바이스 메소드)이 모두 동일한 조인 포인트에서 실행되어야 할 때, 순서는 정의되지 않습니다 (javac 컴파일 클래스에 대해 리플렉션을 통해 소스 코드 선언 순서를 검색할 방법이 없으므로).

이러한 어드바이스 메소드를 각 `@Aspect` 클래스 내의 조인 포인트당 하나의 어드바이스 메소드로 축소하거나, `Ordered` 또는 `@Order`를 통해 애스펙트 레벨에서 순서를 지정할 수 있는 별도의 `@Aspect` 클래스로 어드바이스 조각을 리팩토링하는 것을 고려하십시오.

---

**전체 주제: 어드바이스 선언하기 (Declaring Advice)**

이 부분은 `@Aspect` 클래스 내에서 **실제 부가 기능 로직**을 담고 있는 **어드바이스 메소드**를 어떻게 정의하고, 이 메소드를 **어떤 포인트컷(조인 포인트)** 과 연결하며, 필요한 경우 **조인 포인트의 정보(메소드 인수, 반환 값, 예외 등)를 어떻게 어드바이스 메소드에서 사용할 수 있는지** 설명합니다.

**핵심 아이디어:** `@Aspect` 클래스 안에 부가 기능을 수행할 메소드를 만들고, `@Before`, `@AfterReturning`, `@Around` 등 어노테이션과 포인트컷 표현식을 사용하여 "언제, 어디서" 이 메소드를 실행할지 지정하고, 필요하면 실행 중인 메소드의 정보도 가져와 사용하자!

---

**1. 어드바이스와 포인트컷의 연결:**

- 어드바이스는 항상 특정 **포인트컷 표현식**과 연결됩니다. 이 표현식은 어드바이스가 실행될 **조인 포인트(메소드 실행)** 를 결정합니다.
- 포인트컷 표현식은 두 가지 방식으로 제공될 수 있습니다:
  - **인라인(Inline) 방식:** 어드바이스 어노테이션(`@Before`, `@After` 등)의 `value` 속성에 직접 포인트컷 표현식 문자열을 작성합니다. (간단한 경우 편리)
  - **이름 붙여진 포인트컷 참조 방식:** `@Pointcut`으로 미리 정의해둔 포인트컷 시그니처(메소드 이름)를 어드바이스 어노테이션의 `value` 속성에 지정합니다. (표현식이 복잡하거나 재사용될 때 권장)

---

**2. 어드바이스 종류별 선언 방법:**

- **(1) Before 어드바이스 (`@Before`):**
  - **역할:** 조인 포인트(대상 메소드)가 실행되기 **전**에 실행됩니다.
  - **특징:** 대상 메소드의 실행 흐름 자체를 막을 수는 없습니다. (예외를 던지면 중단 가능) 접근 제어, 로깅 시작 등에 사용됩니다.
  - **선언:** `@Before("포인트컷표현식 또는 이름참조")` 어노테이션을 부가 기능 로직을 담은 메소드 위에 붙입니다.

      ```java
      @Before("execution(* com.xyz.dao.*.*(..))") // 인라인 포인트컷
      public void doAccessCheck() {
          System.out.println("DAO 접근 전 체크 수행...");
      }
      
      @Before("com.xyz.CommonPointcuts.dataAccessOperation()") // 이름 붙여진 포인트컷 참조
      public void logDataAccess() {
          System.out.println("데이터 접근 시도...");
      }
      ```

- **(2) After Returning 어드바이스 (`@AfterReturning`):**
  - **역할:** 조인 포인트(대상 메소드)가 **예외 없이 정상적으로 실행 완료되고 값을 반환한 후**에 실행됩니다.
  - **특징:** 메소드 실행 결과(반환 값)에 접근하여 추가 작업을 할 수 있습니다. (예: 결과 로깅, 캐시 업데이트) 하지만 반환 값 자체를 변경할 수는 없습니다.
  - **선언:** `@AfterReturning("포인트컷")`
  - **반환 값 접근:** `@AfterReturning` 어노테이션의 `returning` 속성에 파라미터 이름을 지정하고, 어드바이스 메소드에 **동일한 이름과 호환되는 타입의 파라미터**를 선언하면, 대상 메소드의 반환 값이 해당 파라미터로 전달됩니다. `pointcut` 속성도 함께 사용해야 합니다.

      ```java
      @AfterReturning(
          pointcut = "execution(* com.xyz.dao.*.*(..))",
          returning = "returnValue" // "returnValue"라는 이름으로 반환 값을 바인딩
      )
      public void logResult(Object returnValue) { // 파라미터 이름 일치, 타입은 Object 또는 더 구체적으로
          System.out.println("메소드 반환 값: " + returnValue);
      }
      ```

- **(3) After Throwing 어드바이스 (`@AfterThrowing`):**
  - **역할:** 조인 포인트(대상 메소드) 실행 중 **예외가 발생하여 종료**되었을 때 실행됩니다.
  - **특징:** 발생한 예외 객체에 접근하여 예외 로깅, 특정 예외에 대한 복구 로직 등을 수행할 수 있습니다.
  - **선언:** `@AfterThrowing("포인트컷")`
  - **예외 객체 접근 및 타입 제한:** `@AfterThrowing` 어노테이션의 `throwing` 속성에 파라미터 이름을 지정하고, 어드바이스 메소드에 **동일한 이름과 특정 예외 타입의 파라미터**를 선언하면, 해당 타입의 예외가 발생했을 때만 어드바이스가 실행되고 예외 객체가 파라미터로 전달됩니다.

      ```java
      @AfterThrowing(
          pointcut = "execution(* com.xyz.dao.*.*(..))",
          throwing = "dataAccessEx" // "dataAccessEx"라는 이름으로 예외를 바인딩
      )
      // DataAccessException 또는 그 하위 타입 예외 발생 시에만 실행됨
      public void handleException(DataAccessException dataAccessEx) {
          System.err.println("데이터 접근 오류 발생: " + dataAccessEx.getMessage());
          // 예외 복구 또는 알림 로직...
      }
      ```

  - **주의:** `@AfterThrowing`은 대상 메소드 자체에서 발생한 예외만 처리합니다. 다른 어드바이스(`@After`, `@AfterReturning`)에서 발생한 예외는 처리하지 않습니다.
- **(4) After (Finally) 어드바이스 (`@After`):**
  - **역할:** 조인 포인트(대상 메소드)가 **정상적으로 종료되든, 예외로 종료되든 상관없이 항상** 실행됩니다. (자바의 `finally` 블록과 유사)
  - **특징:** 주로 리소스 해제(예: 락 해제)와 같이 성공/실패 여부와 관계없이 반드시 수행되어야 하는 후처리 로직에 사용됩니다.
  - **선언:** `@After("포인트컷")`

      ```java
      @After("execution(* com.xyz.dao.*.*(..))")
      public void releaseResourceLock() {
          System.out.println("DAO 작업 후 리소스 락 해제 시도...");
          // ... 락 해제 로직 ...
      }
      ```

- **(5) Around 어드바이스 (`@Around`):**
  - **역할:** 조인 포인트(대상 메소드) 실행 **전, 후 모두**에 개입하며, **대상 메소드 호출 자체를 제어**할 수 있는 가장 강력한 어드바이스입니다.
  - **특징:**
    - 메소드 실행 시간을 측정하는 프로파일링.
    - 캐싱: 메소드 실행 전에 캐시를 확인하고, 있으면 실제 메소드를 호출하지 않고 캐시된 값 반환.
    - 트랜잭션 관리: 메소드 실행 전에 트랜잭션 시작, 실행 후 커밋/롤백.
    - 메소드 호출 자체를 막거나, 반환 값을 변경하거나, 다른 예외를 던지는 등의 작업 가능.
  - **선언:** `@Around("포인트컷")`
  - **필수 파라미터:** 어드바이스 메소드의 **첫 번째 파라미터는 반드시 `ProceedingJoinPoint` 타입**이어야 합니다.
  - **`ProceedingJoinPoint.proceed()` 호출:** 어드바이스 로직 내에서 **실제 대상 메소드를 실행**하려면 반드시 `pjp.proceed()` 또는 `pjp.proceed(args)` 메소드를 호출해야 합니다. 이 호출을 생략하면 대상 메소드는 실행되지 않습니다. `proceed()` 호출 전후로 원하는 로직을 추가할 수 있습니다.
  - **반환 값:** 어드바이스 메소드의 반환 타입은 `Object`여야 합니다. `pjp.proceed()`의 반환 값을 받아서 그대로 반환하거나, 필요에 따라 가공하여 반환할 수 있습니다. `void` 반환 타입 메소드라도 `proceed()` 결과는 반환해주는 것이 좋습니다. `void`로 선언하면 호출자는 항상 `null`을 받게 됩니다.

      ```java
      import org.aspectj.lang.ProceedingJoinPoint;
      import org.aspectj.lang.annotation.Around;
      // ...
      @Around("com.xyz.CommonPointcuts.businessService()")
      public Object profileServiceMethod(ProceedingJoinPoint pjp) throws Throwable { // Throwable 선언 필요
          long start = System.currentTimeMillis();
          System.out.println(pjp.getSignature().getName() + " 실행 시작");
          try {
              // ★ 대상 메소드 실행 및 결과 받기 ★
              Object result = pjp.proceed();
              return result; // ★ 결과 반환 ★
          } finally {
              long end = System.currentTimeMillis();
              System.out.println(pjp.getSignature().getName() + " 실행 종료. 소요 시간: " + (end - start) + "ms");
          }
      }
      ```

  - **주의:** 가장 강력하지만 복잡하므로, 다른 어드바이스 타입으로 해결 가능하다면 그것을 먼저 사용하는 것이 좋습니다.

---

**4. 어드바이스 파라미터: 조인 포인트 정보 활용하기**

어드바이스 메소드 내에서 현재 실행 중인 조인 포인트(대상 메소드)에 대한 다양한 정보에 접근할 수 있습니다.

- **`JoinPoint` 파라미터:** **모든** 어드바이스 메소드는 **첫 번째 파라미터**로 `org.aspectj.lang.JoinPoint` 타입의 객체를 선언하여 받을 수 있습니다. (Around 어드바이스는 `ProceedingJoinPoint` 사용)
  - `jp.getArgs()`: 대상 메소드에 전달된 인수(파라미터 값) 배열 반환.
  - `jp.getThis()`: 현재 실행 중인 **프록시 객체** 반환.
  - `jp.getTarget()`: 프록시가 감싸고 있는 **원본 대상 객체** 반환.
  - `jp.getSignature()`: 대상 메소드의 시그니처 정보(이름, 파라미터 타입 등) 반환.
  - `jp.toString()`: 대상 메소드에 대한 유용한 설명 문자열 반환.
- **특정 파라미터 값 바인딩 (`args()`):** 포인트컷 표현식의 `args()` 지정자를 사용하여 특정 타입의 메소드 인수를 어드바이스 메소드의 파라미터로 직접 전달받을 수 있습니다.
  - **포인트컷:** `args(account, ..)` - 첫 번째 인수가 `Account` 타입이고, 그 값을 `account`라는 이름으로 바인딩.
  - **어드바이스:** `public void validateAccount(Account account)` - 포인트컷에서 바인딩한 `account` 값을 파라미터로 받음.
- **어노테이션 값 바인딩 (`@annotation`, `@target`, `@within`, `@args`):** 포인트컷 표현식에서 어노테이션 지정자를 사용하여 매칭된 어노테이션 객체 자체를 어드바이스 메소드의 파라미터로 전달받을 수 있습니다.
  - **포인트컷:** `@annotation(auditable)` - `@Auditable` 어노테이션이 붙은 메소드를 찾고, 해당 어노테이션 인스턴스를 `auditable`이라는 이름으로 바인딩.
  - **어드바이스:** `public void audit(Auditable auditable)` - 바인딩된 `@Auditable` 어노테이션 객체를 파라미터로 받아서 그 속성 값(`auditable.value()`) 등을 사용할 수 있음.
- **제네릭 파라미터:** 어드바이스 파라미터에도 제네릭을 사용하여 특정 타입의 인수를 받을 수 있습니다. (단, 제네릭 컬렉션 자체를 직접 바인딩하는 데는 한계가 있음)
- **파라미터 이름 결정:** 스프링 AOP는 어드바이스/포인트컷 파라미터 이름을 결정하기 위해 여러 전략을 사용합니다 (`argNames` 속성, Kotlin 리플렉션, Java 8+ `parameters` 플래그, 이름 추론 등). 명시적으로 `argNames` 속성을 사용하는 것이 가장 확실할 수 있습니다.
  - `@Before(value = "...", argNames = "bean,auditable")` 처럼 지정하고, 어드바이스 메소드 파라미터 이름을 `bean`, `auditable`로 맞춥니다.
  - 첫 번째 파라미터가 `JoinPoint` 계열이면 `argNames`에서 생략 가능합니다.
- **Around 어드바이스에서 인수 변경하여 진행 (`proceed(args)`):** `ProceedingJoinPoint`의 `proceed(Object[] args)` 메소드를 사용하면, 원본 메소드를 호출할 때 **인수 값을 변경하여 전달**할 수도 있습니다.

---

**5. 어드바이스 순서 지정 (`@Order` 또는 `Ordered`)**

- **문제:** **동일한 조인 포인트**에 여러 애스펙트의 여러 어드바이스가 동시에 적용될 때, 어떤 순서로 실행될지 보장되지 않을 수 있습니다.
- **해결책:** 애스펙트 클래스에 **`@Order(숫자)`** 어노테이션을 붙이거나 **`org.springframework.core.Ordered` 인터페이스를 구현**하여 순서를 지정합니다.
- **규칙:**
  - **숫자가 낮을수록 우선순위가 높습니다.**
  - **Before 어드바이스:** 우선순위 높은 것이 먼저 실행됩니다.
  - **After (Returning/Throwing/Finally) 어드바이스:** 우선순위 높은 것이 **나중에** 실행됩니다. (가장 안쪽에서 실행됨)
  - **Around 어드바이스:** 우선순위 높은 것이 가장 바깥쪽을 감싸게 됩니다 (먼저 시작하고 가장 나중에 끝남).
- **동일 애스펙트 내 순서:** 같은 `@Aspect` 클래스 내에 정의된 같은 타입의 어드바이스들(예: `@Before` 두 개) 간의 순서는 **보장되지 않습니다.** 필요하다면 별도의 애스펙트 클래스로 분리하고 `@Order`를 적용해야 합니다.

**요약:**

어드바이스는 `@Aspect` 클래스 내에서 부가 기능 로직을 담는 메소드이며, `@Before`, `@AfterReturning`, `@AfterThrowing`, `@After`, `@Around` 어노테이션과 포인트컷 표현식을 통해 실행 시점과 대상을 지정합니다. `JoinPoint` 파라미터나 포인트컷의 바인딩 기능(`args`, `@annotation` 등)을 사용하여 조인 포인트의 컨텍스트 정보(인수, 반환 값, 예외, 어노테이션 등)를 어드바이스 내에서 활용할 수 있습니다. 여러 어드바이스가 같은 조인 포인트에 적용될 때는 `@Order`나 `Ordered` 인터페이스를 사용하여 실행 순서를 제어할 수 있습니다.
