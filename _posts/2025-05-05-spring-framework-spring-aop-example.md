---
title: AOP Example
description: 
author: laze
date: 2025-05-05 00:00:19 +0900
categories: [Dev, SpringBoot]
tags: [SpringBoot]
---
**An AOP Example**

비즈니스 서비스 실행은 때때로 동시성 문제(예: 데드락 패자(deadlock loser))로 인해 실패할 수 있습니다.

작업을 재시도하면 다음 시도에 성공할 가능성이 높습니다.

이러한 조건에서 재시도하는 것이 적절한 비즈니스 서비스(충돌 해결을 위해 사용자에게 다시 돌아갈 필요가 없는 멱등(idempotent) 작업)의 경우,

클라이언트가 `PessimisticLockingFailureException`을 보지 않도록 작업을 투명하게 재시도하고 싶습니다.

이는 서비스 계층의 여러 서비스에 명확하게 걸쳐 있는 요구 사항이므로, 애스펙트(aspect)를 통해 구현하기에 이상적입니다.

작업을 재시도하고 싶기 때문에, `proceed`를 여러 번 호출할 수 있도록 around 어드바이스를 사용해야 합니다. 다음 목록은 기본적인 애스펙트 구현을 보여줍니다:

**Java**

```java
import org.aspectj.lang.ProceedingJoinPoint;
import org.aspectj.lang.annotation.Around;
import org.aspectj.lang.annotation.Aspect;
import org.springframework.core.Ordered;
import org.springframework.dao.PessimisticLockingFailureException; // Assuming this exception exists

@Aspect
public class ConcurrentOperationExecutor implements Ordered { // Implement Ordered for precedence

	private static final int DEFAULT_MAX_RETRIES = 2;

	private int maxRetries = DEFAULT_MAX_RETRIES;
	private int order = 1; // Default order

	public void setMaxRetries(int maxRetries) {
		this.maxRetries = maxRetries;
	}

	@Override // Implement getOrder() from Ordered
	public int getOrder() {
		return this.order;
	}

	public void setOrder(int order) {
		this.order = order;
	}

	// Use the shared pointcut
	@Around("com.xyz.CommonPointcuts.businessService()") // ①
	public Object doConcurrentOperation(ProceedingJoinPoint pjp) throws Throwable {
		int numAttempts = 0;
		PessimisticLockingFailureException lockFailureException;
		do {
			numAttempts++;
			try {
				return pjp.proceed(); // Try to execute the original method
			}
			catch(PessimisticLockingFailureException ex) {
				lockFailureException = ex; // Catch the specific exception
			}
		} while(numAttempts <= this.maxRetries); // Retry until maxRetries is reached
		// If all retries fail, throw the last caught exception
		throw lockFailureException;
	}
}
```

① `Sharing Named Pointcut Definitions`에서 정의된 `businessService` 이름 붙여진 포인트컷을 참조합니다.

**Kotlin**

```kotlin
import org.aspectj.lang.ProceedingJoinPoint
import org.aspectj.lang.annotation.Around
import org.aspectj.lang.annotation.Aspect
import org.springframework.core.Ordered
import org.springframework.dao.PessimisticLockingFailureException // Assuming this exception exists

@Aspect
class ConcurrentOperationExecutor : Ordered { // Implement Ordered for precedence

    companion object {
        private const val DEFAULT_MAX_RETRIES = 2
    }

    var maxRetries: Int = DEFAULT_MAX_RETRIES
    private var _order: Int = 1 // Backing field for order

    override fun getOrder(): Int { // Implement getOrder() from Ordered
        return this._order
    }

    fun setOrder(order: Int) { // Optional setter for order
        this._order = order
    }

    // Use the shared pointcut
    @Around("com.xyz.CommonPointcuts.businessService()") // ①
    @Throws(Throwable::class)
    fun doConcurrentOperation(pjp: ProceedingJoinPoint): Any? {
        var numAttempts = 0
        var lockFailureException: PessimisticLockingFailureException? = null // Nullable
        do {
            numAttempts++
            try {
                return pjp.proceed() // Try to execute the original method
            } catch (ex: PessimisticLockingFailureException) {
                lockFailureException = ex // Catch the specific exception
            }
        } while (numAttempts <= this.maxRetries) // Retry until maxRetries is reached
        // If all retries fail, throw the last caught exception (if any)
        throw lockFailureException ?: RuntimeException("Retry failed but no exception captured") // Handle null case
    }
}

```

애스펙트가 `Ordered` 인터페이스를 구현하여 트랜잭션 어드바이스보다 애스펙트의 우선순위를 높게 설정할 수 있다는 점에 유의하십시오(재시도할 때마다 새로운 트랜잭션을 원함).

`maxRetries`와 `order` 속성은 모두 스프링에 의해 구성됩니다. 주요 작업은 `doConcurrentOperation` around 어드바이스에서 발생합니다.

현재로서는 재시도 로직을 각 `businessService`에 적용한다는 점에 유의하십시오.

진행(proceed)을 시도하고 `PessimisticLockingFailureException`으로 실패하면, 모든 재시도 횟수를 소진하지 않은 한 다시 시도합니다.

해당 스프링 구성은 다음과 같습니다:

**Java**

```java
import org.springframework.context.annotation.Bean;
import org.springframework.context.annotation.Configuration;
import org.springframework.context.annotation.EnableAspectJAutoProxy;

@Configuration
@EnableAspectJAutoProxy // Don't forget to enable AspectJ auto-proxying
public class ApplicationConfiguration {

	@Bean
	public ConcurrentOperationExecutor concurrentOperationExecutor() {
		ConcurrentOperationExecutor executor = new ConcurrentOperationExecutor();
		executor.setMaxRetries(3); // Example: Set max retries to 3
		executor.setOrder(100);    // Example: Set order
		return executor;
	}

    // ... other bean definitions like CommonPointcuts if needed
    @Bean
    public CommonPointcuts commonPointcuts() {
        return new CommonPointcuts();
    }
}
```

**Kotlin**

```kotlin
import org.springframework.context.annotation.Bean
import org.springframework.context.annotation.Configuration
import org.springframework.context.annotation.EnableAspectJAutoProxy

@Configuration
@EnableAspectJAutoProxy // Don't forget to enable AspectJ auto-proxying
class ApplicationConfiguration {

    @Bean
    fun concurrentOperationExecutor(): ConcurrentOperationExecutor {
        val executor = ConcurrentOperationExecutor()
        executor.maxRetries = 3 // Example: Set max retries to 3
        executor.setOrder(100)   // Example: Set order
        return executor
    }

    // ... other bean definitions like CommonPointcuts if needed
    @Bean
    fun commonPointcuts(): CommonPointcuts {
        return CommonPointcuts()
    }
}
```

**Xml**

```xml
<?xml version="1.0" encoding="UTF-8"?>
<beans xmlns="<http://www.springframework.org/schema/beans>"
    xmlns:xsi="<http://www.w3.org/2001/XMLSchema-instance>"
    xmlns:aop="<http://www.springframework.org/schema/aop>"
    xsi:schemaLocation="
        <http://www.springframework.org/schema/beans> <https://www.springframework.org/schema/beans/spring-beans.xsd>
        <http://www.springframework.org/schema/aop> <https://www.springframework.org/schema/aop/spring-aop.xsd>">

    <!-- Enable @AspectJ support -->
    <aop:aspectj-autoproxy/>

    <!-- Define the aspect bean -->
    <bean id="concurrentOperationExecutor" class="example.aop.ConcurrentOperationExecutor">
        <property name="maxRetries" value="3"/>
        <property name="order" value="100"/>
    </bean>

    <!-- Define the CommonPointcuts bean if needed by the aspect -->
    <bean id="commonPointcuts" class="com.xyz.CommonPointcuts"/>

    <!-- other beans -->

</beans>
```

멱등(idempotent) 작업만 재시도하도록 애스펙트를 구체화하기 위해, 다음과 같은 `Idempotent` 어노테이션을 정의할 수 있습니다:

**Java**

```java
import java.lang.annotation.ElementType;
import java.lang.annotation.Retention;
import java.lang.annotation.RetentionPolicy;
import java.lang.annotation.Target;

@Retention(RetentionPolicy.RUNTIME)
@Target(ElementType.METHOD) // Apply to methods
// 마커 어노테이션 (속성 없음)
public @interface Idempotent {
}
```

**Kotlin**

```kotlin
import kotlin.annotation.AnnotationRetention.RUNTIME
import kotlin.annotation.AnnotationTarget.FUNCTION

@Retention(RUNTIME)
@Target(FUNCTION) // Apply to functions (methods)
// 마커 어노테이션 (속성 없음)
annotation class Idempotent
```

그런 다음 이 어노테이션을 사용하여 서비스 작업 구현에 어노테이션을 달 수 있습니다. 멱등 작업만 재시도하도록 애스펙트를 변경하는 것은 `@Idempotent` 작업만 일치하도록 포인트컷 표현식을 구체화하는 것을 포함합니다. 다음과 같습니다:

**Java**

```java
@Aspect // Assuming class level Aspect annotation
public class RefinedConcurrentOperationExecutor { // Renamed for clarity
    // ... other fields and methods ...

    // Refined pointcut: only methods annotated with @Idempotent in the service layer
    @Around("execution(* com.xyz..service.*.*(..)) && " +
            "@annotation(com.xyz.service.Idempotent)") // Check for @Idempotent annotation
    public Object doConcurrentOperation(ProceedingJoinPoint pjp) throws Throwable {
        // ... retry logic as before ...
        int numAttempts = 0;
        PessimisticLockingFailureException lockFailureException = null;
        do {
            numAttempts++;
            try {
                // Use getArgs() to pass original arguments if proceeding again
                return pjp.proceed(pjp.getArgs());
            } catch (PessimisticLockingFailureException ex) {
                lockFailureException = ex;
            }
        } while (numAttempts <= this.maxRetries); // Assuming maxRetries is defined
        throw lockFailureException;
    }
}
```

**Kotlin**

```kotlin
@Aspect // Assuming class level Aspect annotation
class RefinedConcurrentOperationExecutor { // Renamed for clarity
    // ... other fields and methods ...

    // Refined pointcut: only methods annotated with @Idempotent in the service layer
    @Around(
        "execution(* com.xyz..service.*.*(..)) && " +
                "@annotation(com.xyz.service.Idempotent)" // Check for @Idempotent annotation
    )
    @Throws(Throwable::class)
    fun doConcurrentOperation(pjp: ProceedingJoinPoint): Any? {
        // ... retry logic as before ...
        var numAttempts = 0
        var lockFailureException: PessimisticLockingFailureException? = null
        val maxRetries = 3 // Assuming maxRetries is defined or fetched
        do {
            numAttempts++
            try {
                // Use pjp.args to pass original arguments if proceeding again
                return pjp.proceed(pjp.args)
            } catch (ex: PessimisticLockingFailureException) {
                lockFailureException = ex
            }
        } while (numAttempts <= maxRetries)
        throw lockFailureException ?: RuntimeException("Retry failed")
    }
}
```

---

**전체 주제: AOP 예제 (An AOP Example)**

이 부분은 AOP의 개념과 선언 방법들을 활용하여, 애플리케이션의 여러 서비스 계층 메소드에 공통적으로 적용될 수 있는 **"낙관적 락 실패(Pessimistic Locking Failure) 시 작업 재시도"** 기능을 어떻게 애스펙트로 구현하고 설정하는지 구체적인 코드를 통해 보여줍니다.

**핵심 아이디어:** DB 동시성 문제 등으로 작업이 실패했을 때, 클라이언트 코드 수정 없이 AOP를 이용해 투명하게 해당 작업을 몇 번 더 재시도해보자!

---

**1. 문제 상황:**

- 여러 사용자가 동시에 데이터베이스에 접근하여 작업할 때, **동시성 제어 문제**로 인해 특정 작업이 **`PessimisticLockingFailureException`** (또는 유사한 예외)을 발생시키며 실패할 수 있습니다. (예: 데드락 상황에서 한 트랜잭션이 "패자(loser)"로 선택되어 롤백됨)
- 이런 예외는 일시적인 경우가 많아서, **잠시 후 동일한 작업을 다시 시도하면 성공**할 가능성이 높습니다.
- 하지만 모든 서비스 메소드 코드 안에 `try-catch`와 재시도 로직을 반복적으로 넣는 것은 매우 비효율적이고 코드를 지저분하게 만듭니다.

---

**2. AOP 해결책: 재시도 애스펙트 구현**

- 이 "재시도" 로직은 특정 비즈니스 로직과 관계없이 여러 서비스 메소드에 **공통적으로 적용**될 수 있는 **횡단 관심사(crosscutting concern)** 입니다. 따라서 AOP 애스펙트로 구현하기에 매우 적합합니다.
- **선택할 어드바이스 타입:** 작업을 재시도하려면 원래 메소드(`pjp.proceed()`)를 **여러 번 호출**할 수 있어야 하고, 특정 예외(`PessimisticLockingFailureException`)를 잡아서 처리해야 합니다. 따라서 **`@Around` 어드바이스**가 가장 적합합니다.

---

**3. 애스펙트 구현 (`ConcurrentOperationExecutor`)**

- **`@Aspect`**: 클래스가 애스펙트임을 선언합니다.
- **`implements Ordered`**: 이 애스펙트의 **실행 순서**를 제어하기 위해 `Ordered` 인터페이스를 구현합니다.
  - **이유:** 보통 재시도 로직은 **트랜잭션(Transaction) 어드바이스보다 먼저** 실행되어야 합니다. 즉, 재시도하는 각 시도마다 새로운 트랜잭션이 시작되도록 하는 것이 일반적입니다. 트랜잭션 어드바이스는 보통 낮은 순서값(높은 우선순위)을 가지므로, 이 재시도 애스펙트는 그보다 **높은 순서값(낮은 우선순위)** 을 갖도록 설정해야 트랜잭션 애스펙트보다 바깥쪽에서 실행됩니다. (`order` 속성으로 제어)
- **`maxRetries` 필드:** 최대 재시도 횟수를 저장합니다. (기본값 2, 외부 설정 가능)
- **`order` 필드 및 `getOrder()`/`setOrder()`:** `Ordered` 인터페이스 구현을 위한 필드와 메소드입니다. 순서 값을 저장하고 반환합니다. (외부 설정 가능)
- **`@Around("com.xyz.CommonPointcuts.businessService()")`:**
  - `@Around`: Around 어드바이스임을 선언합니다.
  - `"com.xyz.CommonPointcuts.businessService()"`: 이전에 정의된 **공통 포인트컷을 참조**합니다. (예: `execution(* com.xyz..service.*.*(..))` - 서비스 계층의 모든 메소드 실행) 즉, **서비스 계층의 모든 메소드**에 이 재시도 로직을 적용합니다.
- **`doConcurrentOperation(ProceedingJoinPoint pjp)` 메소드:** 실제 어드바이스 로직입니다.
  1. `numAttempts` 변수로 시도 횟수를 셉니다.
  2. `do-while` 루프를 사용하여 최대 `maxRetries` 횟수까지 반복합니다.
  3. `try` 블록 안에서 `pjp.proceed()`를 호출하여 **원본 서비스 메소드를 실행**합니다. 성공하면 그 결과를 즉시 반환하고 루프를 종료합니다.
  4. `catch (PessimisticLockingFailureException ex)`: 만약 실행 중 **특정 예외(`PessimisticLockingFailureException`)** 가 발생하면, 예외 객체를 `lockFailureException` 변수에 저장하고 루프를 계속 진행하여 재시도합니다. (다른 종류의 예외는 잡지 않고 그대로 위로 던져짐)
  5. 루프가 최대 재시도 횟수까지 돌았는데도 계속 실패했다면 (즉, `while` 조건을 빠져나오면), 마지막으로 잡았던 `lockFailureException`을 최종적으로 던집니다.

---

**4. 스프링 설정 (애스펙트 빈 등록 및 속성 설정)**

애스펙트 클래스를 만들었으면 스프링 컨테이너에 빈으로 등록하고 필요한 속성(최대 재시도 횟수, 실행 순서)을 설정해주어야 합니다.

- **Java Config:**

    ```java
    @Configuration
    @EnableAspectJAutoProxy // AOP 활성화 필수!
    public class ApplicationConfiguration {
        @Bean
        public ConcurrentOperationExecutor concurrentOperationExecutor() {
            ConcurrentOperationExecutor executor = new ConcurrentOperationExecutor();
            executor.setMaxRetries(3); // 최대 재시도 횟수 3으로 설정
            executor.setOrder(100);    // 실행 순서 100으로 설정 (값이 클수록 우선순위 낮음)
            return executor;
        }
        // CommonPointcuts 빈도 필요하면 등록
        @Bean public CommonPointcuts commonPointcuts() { return new CommonPointcuts(); }
        // ... 다른 빈들 ...
    }
    ```

- **XML Config:**

    ```xml
    <aop:aspectj-autoproxy/> <!-- AOP 활성화 필수! -->
    
    <bean id="concurrentOperationExecutor" class="example.aop.ConcurrentOperationExecutor">
        <property name="maxRetries" value="3"/> <!-- 속성 설정 -->
        <property name="order" value="100"/>    <!-- 속성 설정 -->
    </bean>
    
    <bean id="commonPointcuts" class="com.xyz.CommonPointcuts"/> <!-- 필요시 등록 -->
    <!-- ... 다른 빈들 ... -->
    ```


---

**5. 개선: 멱등(Idempotent) 작업에만 재시도 적용하기**

모든 서비스 작업을 재시도하는 것은 위험할 수 있습니다.

예를 들어, '사용자에게 이메일 발송' 같은 작업은 실패 시 재시도하면 같은 메일이 여러 번 발송될 수 있습니다.

재시도는 **여러 번 실행해도 결과가 동일한 멱등(idempotent) 작업**에만 적용하는 것이 안전합니다.

- **`@Idempotent` 어노테이션 정의:** 재시도가 안전한 메소드임을 표시하기 위한 **마커 어노테이션**을 직접 만듭니다. (속성 없는 간단한 어노테이션)

    ```java
    @Retention(RetentionPolicy.RUNTIME)
    @Target(ElementType.METHOD)
    public @interface Idempotent {}
    ```

- **서비스 메소드에 적용:** 재시도해도 되는 서비스 메소드 위에 `@Idempotent` 어노테이션을 붙입니다.

    ```java
    @Service
    public class MyIdempotentService {
        @Idempotent // 이 메소드는 재시도 가능
        public void updateCounter() { ... }
    
        public void sendNotification() { ... } // 이 메소드는 재시도하면 안됨
    }
    ```

- **애스펙트 포인트컷 수정:** `@Around` 어노테이션의 포인트컷 표현식을 수정하여, 기존 `businessService` 조건에 **`@annotation(com.xyz.service.Idempotent)` 조건을 AND(`&&`)로 추가**합니다. 이렇게 하면 서비스 계층 메소드 중에서도 **`@Idempotent` 어노테이션이 붙은 메소드에만** 재시도 로직이 적용됩니다.

    ```java
    @Around("com.xyz.CommonPointcuts.businessService() && @annotation(com.xyz.service.Idempotent)")
    public Object doConcurrentOperation(ProceedingJoinPoint pjp) throws Throwable {
        // ... 재시도 로직은 동일 ...
        // proceed() 호출 시 인수를 명시적으로 전달하는 것이 더 안전할 수 있음: pjp.proceed(pjp.getArgs())
    }
    ```


**요약:**

이 예제는 AOP 애스펙트를 사용하여 **여러 서비스 메소드에 공통적으로 필요한 "낙관적 락 실패 시 재시도" 로직**을 어떻게 구현하고 적용하는지 보여줍니다. `@Around` 어드바이스와 `ProceedingJoinPoint`를 사용하여 원본 메소드 호출을 제어하고 예외를 처리하며 재시도를 구현합니다. `Ordered` 인터페이스로 애스펙트 실행 순서를 제어하고, `@Idempotent` 같은 커스텀 어노테이션과 포인트컷 수정을 통해 재시도 대상을 더 안전하게 제한할 수도 있습니다. 이는 AOP가 어떻게 횡단 관심사를 효과적으로 분리하고 적용하는지를 보여주는 좋은 실용적인 예시입니다.
