---
title: Introductions
description: 
author: laze
date: 2025-05-05 00:00:17 +0900
categories: [Dev, SpringBoot]
tags: [SpringBoot]
---
**Introductions**

인트로덕션(AspectJ에서는 타입 간 선언(inter-type declarations)으로 알려짐)은 애스펙트(aspect)가 어드바이스받는(advised) 객체들이 주어진 인터페이스를 구현한다고 선언하고,

해당 객체들을 대신하여 그 인터페이스의 구현을 제공할 수 있게 합니다.

`@DeclareParents` 어노테이션을 사용하여 인트로덕션을 만들 수 있습니다.

이 어노테이션은 매칭되는 타입들이 새로운 부모를 가진다고 선언하는 데 사용됩니다(그래서 이름이 이렇습니다).

예를 들어, `UsageTracked`라는 이름의 인터페이스와 `DefaultUsageTracked`라는 이름의 해당 인터페이스 구현이 주어졌을 때,

다음 애스펙트는 서비스 인터페이스의 모든 구현자들이 `UsageTracked` 인터페이스도 구현한다고 선언합니다 (예: JMX를 통한 통계 목적):

```java
// Java
import org.aspectj.lang.annotation.Aspect;
import org.aspectj.lang.annotation.Before;
import org.aspectj.lang.annotation.DeclareParents;

// Assuming UsageTracked interface and DefaultUsageTracked implementation exist
interface UsageTracked { void incrementUseCount(); }
class DefaultUsageTracked implements UsageTracked {
    private int count;
    @Override public void incrementUseCount() { this.count++; }
    // ... other methods
}

@Aspect
public class UsageTracking {

    // Declare that types matching "com.xyz.service.*+" should implement UsageTracked
    // and provide DefaultUsageTracked as the implementation.
	@DeclareParents(value="com.xyz.service.*+", defaultImpl=DefaultUsageTracked.class)
	public static UsageTracked mixin; // The field type determines the interface to introduce

    // Advice that runs before methods in the service layer
    // Binds the 'this' proxy (which now implements UsageTracked) to the usageTracked parameter
	@Before("execution(* com.xyz..service.*.*(..)) && this(usageTracked)")
	public void recordUsage(UsageTracked usageTracked) { // Access the introduced interface
		usageTracked.incrementUseCount(); // Call the introduced method
	}

}
```

```kotlin
// Kotlin
import org.aspectj.lang.annotation.Aspect
import org.aspectj.lang.annotation.Before
import org.aspectj.lang.annotation.DeclareParents

// Assuming UsageTracked interface and DefaultUsageTracked implementation exist
interface UsageTracked { fun incrementUseCount() }
class DefaultUsageTracked : UsageTracked {
    private var count: Int = 0
    override fun incrementUseCount() { this.count++ }
    // ... other methods
}

@Aspect
class UsageTracking {

    companion object { // Declare static field in companion object for @DeclareParents
        // Declare that types matching "com.xyz.service.*+" should implement UsageTracked
        // and provide DefaultUsageTracked as the implementation.
        @JvmField // Expose as a field for AspectJ processing
        @DeclareParents(value = "com.xyz.service.*+", defaultImpl = DefaultUsageTracked::class)
        var mixin: UsageTracked? = null // Type determines interface, initialize as null or use lateinit
    }

    // Advice that runs before methods in the service layer
    // Binds the 'this' proxy (which now implements UsageTracked) to the usageTracked parameter
    @Before("execution(* com.xyz..service.*.*(..)) && this(usageTracked)")
    fun recordUsage(usageTracked: UsageTracked) { // Access the introduced interface
        usageTracked.incrementUseCount() // Call the introduced method
    }

}
```

구현될 인터페이스는 어노테이션된 필드의 타입에 의해 결정됩니다.

`@DeclareParents` 어노테이션의 `value` 속성은 AspectJ 타입 패턴입니다. 매칭되는 타입의 모든 빈은 `UsageTracked` 인터페이스를 구현합니다.

앞의 예제의 before 어드바이스에서 서비스 빈이 `UsageTracked` 인터페이스의 구현으로 직접 사용될 수 있다는 점에 유의하십시오. 빈에 프로그래밍 방식으로 접근하는 경우, 다음을 작성할 것입니다:

```java
// Java
import org.springframework.context.ApplicationContext;
// Assuming context is an initialized ApplicationContext and "myService" bean exists

UsageTracked usageTracked = context.getBean("myService", UsageTracked.class);
// usageTracked now refers to the proxy which implements both the original service interface
// and the introduced UsageTracked interface.
usageTracked.incrementUseCount(); // Can call the introduced method
```

```kotlin
// Kotlin
import org.springframework.context.ApplicationContext
// Assuming context is an initialized ApplicationContext and "myService" bean exists

val usageTracked = context.getBean("myService", UsageTracked::class.java)
// usageTracked now refers to the proxy which implements both the original service interface
// and the introduced UsageTracked interface.
usageTracked?.incrementUseCount() // Can call the introduced method (use safe call if applicable)
```

---

**전체 주제: 인트로덕션 (Introductions)**

이 부분은 AOP 애스펙트를 사용하여, 기존 클래스의 **코드를 전혀 수정하지 않고** 그 클래스로 만들어진 빈(Bean) 객체에게 **새로운 인터페이스를 구현하게 만들고, 해당 인터페이스의 실제 구현(메소드 등)까지 제공**하는 방법에 대한 내용입니다.

**핵심 아이디어:** 원래는 할 줄 몰랐던 일(특정 인터페이스 구현)을 AOP를 통해 대상 객체에게 가르쳐서(능력을 추가하여) 할 수 있게 만들자!

---

**1. 인트로덕션이란? (타입 간 선언)**

- **개념:** 기존 클래스(타입)에 **새로운 멤버(메소드, 필드)나 인터페이스 구현을 선언적으로 추가**하는 AOP 기능입니다. AspectJ에서는 이를 타입 간 선언(Inter-type declaration)이라고 부릅니다.
- **스프링 AOP의 지원:** 스프링 AOP는 전체 타입 간 선언 기능 중 **"새로운 인터페이스를 구현하도록 만드는" 인트로덕션** 기능을 지원합니다.
- **목적:** 여러 객체에 공통적인 상태나 행위를 추가하고 싶을 때 유용합니다. 예를 들어, 여러 서비스 빈들이 자신의 사용 횟수를 추적하는 기능을 공통적으로 갖게 하고 싶을 때 사용할 수 있습니다.

---

**2. `@DeclareParents` 어노테이션 사용법**

스프링 AOP에서 인트로덕션을 구현하는 방법은 `@Aspect` 클래스 내부에 `@DeclareParents` 어노테이션이 붙은 **정적(static) 필드**를 선언하는 것입니다.

- **`@DeclareParents` 어노테이션:**
  - **`value` 속성:** **어떤 타입들**에게 새로운 인터페이스를 도입할지 지정합니다. **AspectJ 타입 패턴**을 사용합니다. (예: `"com.xyz.service.*+"` -> `com.xyz.service` 패키지의 모든 타입 및 그 하위 타입들)
  - **`defaultImpl` 속성:** **도입할 인터페이스의 실제 구현**을 제공할 클래스를 지정합니다. 이 클래스는 도입될 인터페이스를 구현하고 있어야 합니다.
- **정적 필드 선언:**
  - `@DeclareParents` 어노테이션은 **`static` 필드** 위에 붙여야 합니다. (Kotlin에서는 `companion object` 내부에 `@JvmField`와 함께 선언)
  - 이 필드의 **타입**은 **도입(Introduce)하려는 인터페이스 타입**이어야 합니다. 스프링 AOP는 이 필드 타입을 보고 어떤 인터페이스를 추가해야 하는지 결정합니다.
  - 필드 이름 자체는 중요하지 않습니다 (예제에서는 `mixin`). 필드 값도 사용되지 않으므로 `null`로 두거나 초기화하지 않아도 됩니다.
- **예시 분석:**

    ```java
    @Aspect
    public class UsageTracking {
    
        // ★ 인트로덕션 선언 ★
        @DeclareParents(
            value = "com.xyz.service.*+",         // 대상: com.xyz.service 패키지 및 하위 타입들
            defaultImpl = DefaultUsageTracked.class // 구현 제공: DefaultUsageTracked 클래스 사용
        )
        public static UsageTracked mixin; // ★ 도입할 인터페이스 타입: UsageTracked ★
                                          // static 필드여야 함
    
        // ... (어드바이스 등) ...
    }
    ```

  - **의미:** "`com.xyz.service` 패키지 또는 그 하위 패키지에 속하는 **모든 타입**의 스프링 빈들은 이제 **`UsageTracked` 인터페이스를 구현**하게 된다. 그리고 `UsageTracked` 인터페이스의 메소드 구현은 **`DefaultUsageTracked` 클래스의 인스턴스**가 대신 제공한다."

---

**3. 인트로덕션의 동작 방식 (AOP 프록시 활용)**

인트로덕션은 스프링 AOP의 **프록시** 메커니즘을 통해 동작합니다.

1. 스프링 컨테이너는 `@DeclareParents` 설정이 적용될 대상 빈(예: `MyServiceImpl` 빈, `com.xyz.service` 패키지에 속함)을 찾습니다.
2. 해당 빈에 대한 **AOP 프록시**를 생성할 때, 스프링은 이 프록시가 **원래 빈의 타입**뿐만 아니라 `@DeclareParents`에서 지정한 **새로운 인터페이스(`UsageTracked`)** 도 **함께 구현**하도록 만듭니다.
3. 프록시 객체 내부에 `@DeclareParents`의 `defaultImpl`로 지정된 클래스(`DefaultUsageTracked`)의 **인스턴스를 생성하여 가지고 있습니다.** (이를 '믹스인(Mixin)'이라고도 합니다.)
4. 클라이언트가 프록시 객체에 대해 **원래 빈의 메소드**를 호출하면, 일반적인 AOP 처리 후 원본 대상 객체에게 위임됩니다.
5. 클라이언트가 프록시 객체에 대해 **새롭게 도입된 인터페이스(`UsageTracked`)의 메소드**(예: `incrementUseCount()`)를 호출하면, 프록시는 이 호출을 자신이 가지고 있는 **`DefaultUsageTracked` 인스턴스에게 전달**하여 처리합니다.

---

**4. 인트로덕션 사용 예시 (어드바이스 및 직접 접근)**

- **어드바이스에서 사용:**
  - 인트로덕션이 적용된 빈은 이제 새 인터페이스(`UsageTracked`) 타입으로도 취급될 수 있습니다.
  - 어드바이스(예: `@Before`)의 포인트컷에서 `this(인터페이스타입)` 지정자를 사용하여 **프록시 객체 자체가 해당 인터페이스를 구현하는지 확인**하고, 그 인터페이스 타입으로 **프록시 객체를 바인딩**하여 어드바이스 메소드 파라미터로 받을 수 있습니다.

      ```java
      // 포인트컷: 서비스 계층 메소드 실행 && 프록시 객체(this)가 UsageTracked 타입인가?
      //            -> 그렇다면 프록시 객체를 usageTracked 파라미터로 바인딩하라.
      @Before("execution(* com.xyz..service.*.*(..)) && this(usageTracked)")
      public void recordUsage(UsageTracked usageTracked) { // ★ UsageTracked 타입으로 프록시 받음 ★
          // 프록시에 대해 도입된 인터페이스의 메소드 호출 가능
          usageTracked.incrementUseCount();
      }
      ```

- **직접 접근:** 빈을 컨테이너에서 직접 가져올 때(`getBean`), 원래 타입뿐만 아니라 **새롭게 도입된 인터페이스 타입**으로도 요청하여 가져올 수 있습니다. 반환되는 것은 항상 **프록시 객체**입니다.

    ```java
    // "myService" 빈을 UsageTracked 타입으로 요청
    UsageTracked usageTracked = context.getBean("myService", UsageTracked.class);
    // 프록시 객체를 통해 도입된 메소드 호출 가능
    usageTracked.incrementUseCount();
    ```


**요약:**

인트로덕션(`@DeclareParents`)은 AOP를 사용하여 기존 클래스 코드 수정 없이 **새로운 인터페이스 구현 능력을 동적으로 추가**하는 기능입니다. `@Aspect` 클래스 내의 `@DeclareParents` 어노테이션이 붙은 `static` 필드를 통해 선언하며, 대상 타입 패턴(`value`)과 실제 구현을 제공할 클래스(`defaultImpl`)를 지정합니다. 스프링 AOP는 프록시를 생성할 때 해당 인터페이스도 함께 구현하도록 만들고, 도입된 인터페이스 메소드 호출은 `defaultImpl` 인스턴스에게 위임하여 처리합니다. 이를 통해 여러 빈에 공통적인 상태나 행위를 효과적으로 추가할 수 있습니다.
