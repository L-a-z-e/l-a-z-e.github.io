---
title: Proxying Mechanisms
description: 
author: laze
date: 2025-05-06 00:00:03 +0900
categories: [Dev, SpringBoot]
tags: [SpringBoot]
---
**Proxying Mechanisms**

스프링 AOP는 주어진 대상 객체에 대한 프록시를 생성하기 위해 JDK 동적 프록시 또는 CGLIB 중 하나를 사용합니다.

JDK 동적 프록시는 JDK에 내장되어 있는 반면, CGLIB는 일반적인 오픈 소스 클래스 정의 라이브러리입니다 (`spring-core`에 재패키징됨).

프록시될 대상 객체가 최소한 하나의 인터페이스를 구현하는 경우, JDK 동적 프록시가 사용되며 대상 타입에 의해 구현된 모든 인터페이스가 프록시됩니다.

대상 객체가 인터페이스를 구현하지 않는 경우, 대상 타입의 런타임 생성 하위 클래스인 CGLIB 프록시가 생성됩니다.

CGLIB 프록시 사용을 강제하려면 (예: 대상 객체에 대해 정의된 모든 메소드를 프록시하기 위해, 인터페이스에 의해 구현된 것뿐만 아니라), 그렇게 할 수 있습니다.

그러나 다음 문제들을 고려해야 합니다:

- `final` 클래스는 확장될 수 없으므로 프록시될 수 없습니다.
- `final` 메소드는 오버라이드될 수 없으므로 어드바이스될 수 없습니다.
- `private` 메소드는 오버라이드될 수 없으므로 어드바이스될 수 없습니다.
- 보이지 않는 메소드 – 예를 들어, 다른 패키지의 부모 클래스에 있는 패키지 전용(package-private) 메소드 – 는 효과적으로 `private`이므로 어드바이스될 수 없습니다.
- CGLIB 프록시 인스턴스는 Objenesis를 통해 생성되므로 프록시된 객체의 생성자는 두 번 호출되지 않습니다. 그러나 JVM이 생성자 우회(constructor bypassing)를 허용하지 않는 경우, 스프링 AOP 지원에서 이중 호출과 해당 디버그 로그 항목을 볼 수 있습니다.
- CGLIB 프록시 사용은 Java 모듈 시스템에서 제한에 직면할 수 있습니다. 일반적인 경우로, 모듈 경로에 배포할 때 `java.lang` 패키지의 클래스에 대한 CGLIB 프록시를 생성할 수 없습니다. 이러한 경우에는 JVM 부트스트랩 플래그 `-add-opens=java.base/java.lang=ALL-UNNAMED`가 필요하며, 이는 모듈에는 사용할 수 없습니다.

CGLIB 프록시 사용을 강제하려면, 다음 예제와 같이 `<aop:config>` 요소의 `proxy-target-class` 속성 값을 `true`로 설정합니다:

```xml
<aop:config proxy-target-class="true"> <!-- Force CGLIB proxying -->
	<!-- 다른 빈들이 여기에 정의됨... -->
</aop:config>
```

`@AspectJ` 자동 프록시 지원을 사용할 때 CGLIB 프록시를 강제하려면, 다음 예제와 같이 `<aop:aspectj-autoproxy>` 요소의 `proxy-target-class` 속성을 `true`로 설정합니다:

```xml
<aop:aspectj-autoproxy proxy-target-class="true"/> <!-- Force CGLIB proxying for @AspectJ -->
```

*여러 `<aop:config/>` 섹션은 런타임 시 단일 통합 자동 프록시 생성기로 축소(collapsed)되며, 이는 `<aop:config/>` 섹션 (일반적으로 다른 XML 빈 정의 파일에서) 중 어느 것이든 지정한 가장 강력한 프록시 설정을 적용합니다.*

*이는 `<tx:annotation-driven/>` 및 `<aop:aspectj-autoproxy/>` 요소에도 적용됩니다.*

*명확히 하자면, `<tx:annotation-driven/>`, `<aop:aspectj-autoproxy/>` 또는 `<aop:config/>` 요소에 `proxy-target-class="true"`를 사용하면 세 가지 모두에 대해 CGLIB 프록시 사용을 강제합니다.*

**AOP 프록시 이해하기 (Understanding AOP Proxies)**

스프링 AOP는 프록시 기반입니다.

자신만의 애스펙트를 작성하거나 스프링 프레임워크와 함께 제공되는 스프링 AOP 기반 애스펙트를 사용하기 전에 그 마지막 문장이 실제로 무엇을 의미하는지에 대한 의미론을 파악하는 것이 매우 중요합니다.

먼저, 다음 코드 스니펫에서 보여주듯이, 평범하고(plain-vanilla) 프록시되지 않은(un-proxied) 객체 참조를 가진 시나리오를 보면

```java
// Java
interface Pojo {
    void foo();
    void bar();
}

public class SimplePojo implements Pojo {

	@Override
	public void foo() {
		// 다음 메소드 호출은 'this' 참조에 대한 직접 호출입니다
		this.bar();
	}

	@Override
	public void bar() {
		// 일부 로직...
		System.out.println("bar");
	}
}
```

```kotlin
// Kotlin
interface Pojo {
    fun foo()
    fun bar()
}

class SimplePojo : Pojo {
    override fun foo() {
        // 다음 메소드 호출은 'this' 참조에 대한 직접 호출입니다
        this.bar() // Direct call on 'this'
    }

    override fun bar() {
        // 일부 로직...
        println("bar")
    }
}
```

객체 참조에서 메소드를 호출하면, 메소드는 해당 객체 참조에서 직접 호출됩니다:

```java
// Java
import org.springframework.aop.framework.ProxyFactory;
// Assuming RetryAdvice implementing MethodInterceptor exists

public class Main {

	public static void main(String[] args) {
		Pojo pojo = new SimplePojo();
		// 이것은 'pojo' 참조에 대한 직접 메소드 호출입니다
		pojo.foo(); // Calls SimplePojo.foo() directly
	}
}
```

```kotlin
// Kotlin
import org.springframework.aop.framework.ProxyFactory
// Assuming RetryAdvice implementing MethodInterceptor exists

fun main(args: Array<String>) {
    val pojo: Pojo = SimplePojo()
    // 이것은 'pojo' 참조에 대한 직접 메소드 호출입니다
    pojo.foo() // Calls SimplePojo.foo() directly
}
```

클라이언트 코드가 가진 참조가 프록시일 때 상황은 약간 달라집니다.

```java
// Java
import org.aopalliance.intercept.MethodInterceptor;
import org.aopalliance.intercept.MethodInvocation;
import org.springframework.aop.framework.ProxyFactory;

// Example Advice
class RetryAdvice implements MethodInterceptor {
    @Override
    public Object invoke(MethodInvocation invocation) throws Throwable {
        System.out.println("Advice executing before method: " + invocation.getMethod().getName());
        try {
            return invocation.proceed();
        } finally {
            System.out.println("Advice executing after method: " + invocation.getMethod().getName());
        }
    }
}

public class Main {

	public static void main(String[] args) {
		ProxyFactory factory = new ProxyFactory(new SimplePojo()); // Target object
		factory.addInterface(Pojo.class); // Interface to proxy
		factory.addAdvice(new RetryAdvice()); // Add advice

		Pojo pojo = (Pojo) factory.getProxy(); // Get the proxy object
		// 이것은 프록시에 대한 메소드 호출입니다!
		pojo.foo(); // Call goes through the proxy and advice
	}
}
```

```kotlin
// Kotlin
import org.aopalliance.intercept.MethodInterceptor
import org.aopalliance.intercept.MethodInvocation
import org.springframework.aop.framework.ProxyFactory

// Example Advice
class RetryAdvice : MethodInterceptor {
    override fun invoke(invocation: MethodInvocation): Any? {
        println("Advice executing before method: ${invocation.method.name}")
        try {
            return invocation.proceed()
        } finally {
            println("Advice executing after method: ${invocation.method.name}")
        }
    }
}

fun main(args: Array<String>) {
    val factory = ProxyFactory(SimplePojo()) // Target object
    factory.addInterface(Pojo::class.java) // Interface to proxy
    factory.addAdvice(RetryAdvice()) // Add advice

    val pojo = factory.proxy as Pojo // Get the proxy object (Kotlin style cast)
    // 이것은 프록시에 대한 메소드 호출입니다!
    pojo.foo() // Call goes through the proxy and advice
}
```

여기서 이해해야 할 핵심 사항은 `Main` 클래스의 `main(..)` 메소드 내의 클라이언트 코드가 프록시에 대한 참조를 가지고 있다는 것입니다.

이는 즉, 해당 객체 참조에 대한 메소드 호출은 프록시에 대한 호출이라는 것을 의미합니다.

결과적으로, 프록시는 해당 특정 메소드 호출과 관련된 모든 인터셉터(어드바이스)에 위임할 수 있습니다.

그러나 호출이 마침내 대상 객체(`SimplePojo` 참조, 이 경우)에 도달하면, `this.bar()` 또는 `this.foo()`와 같이 자체적으로 수행할 수 있는 모든 메소드 호출은 프록시가 아닌 `this` 참조에 대해 호출됩니다.

이것은 중요한 의미를 가집니다.

이는 즉, **자가 호출(self invocation)은 메소드 호출과 연관된 어드바이스가 실행될 기회를 얻지 못하게 된다는 것을 의미합니다.**

즉, 명시적 또는 암시적 `this` 참조를 통한 자가 호출은 어드바이스를 우회합니다.

이를 해결하려면 다음 옵션이 있습니다.

1. **자가 호출 피하기 (Avoid self invocation)**
   가장 좋은 접근 방식("가장 좋은"이라는 용어는 여기서는 느슨하게 사용됨)은 자가 호출이 발생하지 않도록 코드를 리팩토링하는 것입니다. 이는 어느 정도의 작업이 필요하지만, 가장 좋고 가장 덜 침습적인 접근 방식입니다.
2. **자가 참조 주입하기 (Inject a self reference)**
   대안적인 접근 방식은 자가 주입(self injection)을 사용하고, `this` 대신 자가 참조를 통해 프록시에서 메소드를 호출하는 것입니다.
3. **`AopContext.currentProxy()` 사용하기 (Use `AopContext.currentProxy()`)**
   이 마지막 접근 방식은 매우 권장되지 않으며, 이전 옵션들을 선호하여 지적하기를 주저합니다. 그러나 최후의 수단으로 다음 예제와 같이 클래스 내의 로직을 스프링 AOP에 묶도록 선택할 수 있습니다.

```java
// Java
import org.springframework.aop.framework.AopContext; // Import AopContext

public class SimplePojo implements Pojo {

	@Override
	public void foo() {
		// 이것은 작동하지만 가능하다면 피해야 합니다.
		((Pojo) AopContext.currentProxy()).bar(); // Get current proxy and call bar() on it
	}

	@Override
	public void bar() {
		// 일부 로직...
		System.out.println("bar called via AopContext");
	}
}
```

```kotlin
// Kotlin
import org.springframework.aop.framework.AopContext // Import AopContext

class SimplePojo : Pojo {
    override fun foo() {
        // 이것은 작동하지만 가능하다면 피해야 합니다.
        (AopContext.currentProxy() as Pojo).bar() // Get current proxy and call bar() on it
    }

    override fun bar() {
        // 일부 로직...
        println("bar called via AopContext")
    }
}
```

`AopContext.currentProxy()` 사용은 코드를 스프링 AOP에 완전히 결합시키고, 클래스 자체가 AOP 컨텍스트에서 사용되고 있다는 사실을 인식하게 만들어 AOP의 일부 이점을 감소시킵니다.

또한 다음 예제와 같이 프록시를 노출하도록 `ProxyFactory`를 구성해야 합니다:

```java
// Java
public class Main {

	public static void main(String[] args) {
		ProxyFactory factory = new ProxyFactory(new SimplePojo());
		factory.addInterface(Pojo.class);
		factory.addAdvice(new RetryAdvice());
		factory.setExposeProxy(true); // <<-- 프록시 노출 설정

		Pojo pojo = (Pojo) factory.getProxy();
		// 이것은 프록시에 대한 메소드 호출입니다!
		pojo.foo(); // foo() -> AopContext.currentProxy().bar() -> advice -> SimplePojo.bar()
	}
}
```

```kotlin
// Kotlin
fun main(args: Array<String>) {
    val factory = ProxyFactory(SimplePojo())
    factory.addInterface(Pojo::class.java)
    factory.addAdvice(RetryAdvice())
    factory.isExposeProxy = true // <<-- 프록시 노출 설정 (Kotlin property access)

    val pojo = factory.proxy as Pojo
    // 이것은 프록시에 대한 메소드 호출입니다!
    pojo.foo() // foo() -> AopContext.currentProxy().bar() -> advice -> SimplePojo.bar()
}
```

AspectJ 컴파일 타임 위빙 및 로드 타임 위빙은 프록시 대신 바이트코드 내에 어드바이스를 적용하므로 이 자가 호출 문제가 없습니다.

---

**전체 주제: 프록시 메커니즘 및 AOP 프록시 이해하기**

이 부분은 스프링 AOP가 어떤 기술로 프록시를 만드는지 다시 한번 정리하고, CGLIB 프록시 사용 시의 제약 조건과 설정 방법을 설명하며, 가장 중요하게는 **프록시 기반 AOP의 동작 방식과 그로 인한 "자가 호출(self-invocation)" 문제** 및 해결 방법을 다룹니다.

**핵심 아이디어:** 스프링 AOP는 프록시(대리인)를 사용한다. 어떤 프록시 기술(JDK 또는 CGLIB)을 쓸지 알자. 그리고 프록시 방식 때문에 객체 내부에서 자기 자신의 다른 메소드를 호출할 때는 AOP가 적용되지 않는다는 점을 이해하고 해결 방법을 알자.

---

**첫 번째 파트: 프록시 메커니즘 (Proxying Mechanisms)**

- **두 가지 선택지:** 스프링 AOP는 AOP 프록시를 만들기 위해 다음 두 기술 중 하나를 사용합니다.
  1. **JDK 동적 프록시 (기본):** 대상 객체가 **인터페이스를 구현**할 때 사용됩니다. JDK 내장 기능입니다. 프록시는 대상 객체가 구현한 인터페이스들을 똑같이 구현합니다.
  2. **CGLIB 프록시:** 대상 객체가 **인터페이스를 구현하지 않을 때** 사용됩니다. 대상 클래스를 **상속**하여 프록시를 만듭니다. `spring-core`에 포함된 CGLIB 라이브러리를 사용합니다.
- **CGLIB 사용 강제:** 기본적으로 인터페이스가 있으면 JDK 동적 프록시가 사용되지만, 다음과 같은 이유로 **CGLIB 사용을 명시적으로 강제**할 수 있습니다.
  - **인터페이스에 없는 메소드 어드바이스:** 대상 클래스에만 정의된 메소드(인터페이스에는 없는)에 AOP를 적용하고 싶을 때.
  - **구체 클래스 타입 필요:** 프록시 객체를 원본 클래스 타입으로 직접 캐스팅해야 할 때.
- **CGLIB 사용 강제 방법:**
  - **XML:** `<aop:config proxy-target-class="true">` 또는 `<aop:aspectj-autoproxy proxy-target-class="true"/>` 속성을 설정합니다.
  - **Java Config:** `@EnableAspectJAutoProxy(proxyTargetClass = true)` 속성을 설정합니다.
- **CGLIB 사용 시 제약 조건:**
  - `final` 클래스는 프록시 불가 (상속 불가).
  - `final` 메소드는 어드바이스 불가 (오버라이드 불가).
  - `private` 또는 패키지 외부에서 보이지 않는 메소드는 어드바이스 불가.
  - (내부 동작) 생성자가 두 번 호출되는 것처럼 보일 수 있으나, 보통 Objenesis 라이브러리로 이를 회피합니다.
  - 자바 모듈 시스템(JPMS) 사용 시 제약이 있을 수 있습니다 (특히 `java.lang` 패키지 클래스 프록시).
- **프록시 설정 통합:** 여러 AOP 설정(`aop:config`, `tx:annotation-driven`, `aop:aspectj-autoproxy`)이 혼재할 경우, **가장 강력한 프록시 설정(`proxy-target-class="true"`가 하나라도 있으면)** 이 전체 컨텍스트에 적용됩니다.

---

**두 번째 파트: AOP 프록시 이해하기 (Understanding AOP Proxies) - 자가 호출 문제**

이것이 프록시 기반 AOP를 이해하는 데 **가장 중요한 부분**입니다.

- **프록시 기반 동작 방식 복습:**
  - 클라이언트 코드는 **프록시 객체**에 대한 참조를 갖습니다.
  - 클라이언트가 프록시의 메소드(`pojo.foo()`)를 호출하면, 호출은 **프록시를 통해** 이루어지고 설정된 어드바이스(부가 기능)가 실행된 후, 프록시가 **내부적으로 가지고 있는 원본 대상 객체**의 메소드(`target.foo()`)를 호출합니다.
- **문제점: 자가 호출 (Self Invocation)**
  - 만약 **대상 객체(`SimplePojo`) 내부의 한 메소드(`foo`)** 에서 **같은 객체의 다른 메소드(`bar`)** 를 **`this` 참조**를 통해 직접 호출하면 어떻게 될까요? (`this.bar()`)
  - 이 호출은 **프록시를 거치지 않습니다.** `this`는 프록시 객체가 아닌 **원본 대상 객체 자신**을 가리키기 때문입니다.
  - 결과적으로 `bar()` 메소드에 적용되어야 할 **AOP 어드바이스(예: 트랜잭션, 로깅)가 실행되지 않습니다!** AOP가 적용되지 않고 원본 메소드만 직접 호출됩니다. 이것이 **자가 호출 문제**입니다.

    ```
    Client -> pojoProxy.foo() -> Advice -> target.foo() -> target.bar() // bar()에 대한 Advice 실행 안됨!
                                                        // (this.bar() 호출 시)
    ```

- **자가 호출 문제 해결 방법:**
  1. **리팩토링 (가장 권장):** 자가 호출이 발생하지 않도록 코드를 **재설계(리팩토링)** 합니다. 예를 들어, `bar()` 메소드의 로직을 별도의 다른 빈으로 분리하고, `foo()` 메소드에서는 그 다른 빈을 주입받아 호출하도록 변경합니다. 이것이 가장 깔끔하고 객체지향적인 해결책입니다.
  2. **자가 참조 주입 (Self Reference Injection):** 빈이 자기 자신(의 프록시)을 주입받아서 `this` 대신 주입받은 참조를 통해 메소드를 호출합니다.

      ```java
      @Service
      public class MyServiceImpl implements MyService {
          @Autowired // 자기 자신(의 프록시) 주입
          private MyService self;
      
          @Override
          @Transactional
          public void foo() {
              // this.bar(); // 이렇게 하면 bar()의 @Transactional 적용 안됨
              self.bar(); // ★ 프록시를 통해 호출해야 함 ★
          }
      
          @Override
          @Transactional(propagation = Propagation.REQUIRES_NEW)
          public void bar() { ... }
      }
      
      ```

    - **주의:** 이 방식은 약간 부자연스러워 보일 수 있고, 빈 생성 시 순환 참조 문제가 발생하지 않도록 주의해야 합니다 (예: `@Lazy` 사용 고려).
  3. **`AopContext.currentProxy()` 사용 (비추천):** 코드 내에서 `AopContext.currentProxy()`를 호출하여 현재 실행 중인 **프록시 객체를 명시적으로 얻어온 후**, 그 프록시를 통해 다른 메소드를 호출합니다.

      ```java
      public class SimplePojo implements Pojo {
          @Override
          public void foo() {
              // ★ 현재 프록시를 직접 가져와서 호출 ★
              ((Pojo) AopContext.currentProxy()).bar();
          }
          // ... bar() 메소드 ...
      }
      ```

    - **단점:** 코드가 스프링 AOP API에 **완전히 종속**됩니다. AOP 적용 사실이 코드에 노출되어 AOP의 투명성이 깨집니다. 최후의 수단으로만 사용해야 합니다.
    - **추가 설정 필요:** 이 방식이 작동하려면 AOP 프록시를 생성할 때 **프록시를 노출하도록 설정**해야 합니다 (`ProxyFactory.setExposeProxy(true)` 또는 XML/Java Config의 해당 옵션 설정).
- **AspectJ 위빙과의 차이:** AspectJ 컴파일 타임 또는 로드 타임 위빙은 프록시를 사용하지 않고 직접 바이트코드를 수정하므로 이러한 **자가 호출 문제가 발생하지 않습니다.**

**요약:**

스프링 AOP는 기본적으로 **JDK 동적 프록시**(인터페이스 기반) 또는 **CGLIB 프록시**(클래스 기반)를 사용하여 런타임에 프록시 객체를 생성합니다. CGLIB 사용 시 `final` 제약 등이 있으며 설정을 통해 강제할 수 있습니다. 가장 중요한 것은 프록시 기반이기 때문에, 빈 객체 내부에서 **`this`를 사용한 메소드 자가 호출** 시에는 **AOP 어드바이스가 적용되지 않는다**는 점입니다. 이 문제를 해결하기 위해 코드 리팩토링, 자가 참조 주입, 또는 (권장하지 않지만) `AopContext.currentProxy()` 사용 등의 방법을 고려해야 합니다.
