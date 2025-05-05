---
title: Aspect Instantiation Models
description: 
author: laze
date: 2025-05-05 00:00:18 +0900
categories: [Dev, SpringBoot]
tags: [SpringBoot]
---
**Aspect Instantiation Models**

기본적으로 애플리케이션 컨텍스트 내에는 각 애스펙트(aspect)의 단일 인스턴스가 존재합니다.

AspectJ는 이를 싱글톤(singleton) 인스턴스화 모델이라고 부릅니다.

대체 생명주기를 가진 애스펙트를 정의하는 것이 가능합니다.

스프링은 AspectJ의 `perthis`, `pertarget`, `pertypewithin` 인스턴스화 모델을 지원합니다

`@Aspect` 어노테이션에 `perthis` 절(clause)을 지정하여 `perthis` 애스펙트를 선언할 수 있습니다.

```java
// Java
import org.aspectj.lang.annotation.Aspect;
import org.aspectj.lang.annotation.Before;

// Specifies one aspect instance per service object matching the pointcut
@Aspect("perthis(execution(* com.xyz..service.*.*(..)))")
public class MyAspect {

	private int someState; // State specific to the service object instance

	// This advice runs for method executions on the specific service object
	// associated with this aspect instance.
	@Before("execution(* com.xyz..service.*.*(..))")
	public void recordServiceUsage() {
		// ... this.someState를 사용한 로직
	}
}
```

```kotlin
// Kotlin
import org.aspectj.lang.annotation.Aspect
import org.aspectj.lang.annotation.Before

// Specifies one aspect instance per service object matching the pointcut
@Aspect("perthis(execution(* com.xyz..service.*.*(..)))")
class MyAspect {

    private var someState: Int = 0 // State specific to the service object instance

    // This advice runs for method executions on the specific service object
    // associated with this aspect instance.
    @Before("execution(* com.xyz..service.*.*(..))")
    fun recordServiceUsage() {
        // ... this.someState를 사용한 로직
    }
}
```

앞의 예제에서 `perthis` 절의 효과는 비즈니스 서비스를 수행하는 각 고유 서비스 객체(포인트컷 표현식과 일치하는 조인 포인트에서 `this`에 바인딩된 각 고유 객체)마다 하나의 애스펙트 인스턴스가 생성된다는 것입니다.

애스펙트 인스턴스는 서비스 객체에서 메소드가 처음 호출될 때 생성됩니다.

애스펙트는 서비스 객체가 스코프(scope)를 벗어날 때 스코프를 벗어납니다.

애스펙트 인스턴스가 생성되기 전에는 그 안의 어드바이스(advice)는 실행되지 않습니다.

애스펙트 인스턴스가 생성되자마자 그 안에 선언된 어드바이스는 일치하는 조인 포인트에서 실행되지만, 서비스 객체가 이 애스펙트와 연관된 객체일 때만 실행됩니다.

`pertarget` 인스턴스화 모델은 `perthis`와 정확히 동일한 방식으로 작동하지만, 일치하는 조인 포인트에서 각 고유 대상 객체마다 하나의 애스펙트 인스턴스를 생성합니다.

---

**전체 주제: 애스펙트 인스턴스화 모델 (Aspect Instantiation Models)**

이 부분은 `@Aspect` 어노테이션이 붙은 애스펙트 클래스의 **객체(인스턴스)가 몇 개 만들어지고, 언제 만들어지며, 어떤 범위에서 유효한지** 결정하는 모델들에 대한 설명입니다. AspectJ에서 정의된 모델들을 스프링 AOP가 지원하는 내용입니다.

**핵심 아이디어:** 애스펙트 객체도 일반 빈처럼 딱 하나만 만들어서 계속 쓸 수도 있고(싱글톤 - 기본값), 특정 조건에 따라 여러 개를 만들어서 사용할 수도 있다!

---

**1. 기본 모델: 싱글톤 (Singleton)**

- **동작:** 특별히 다른 모델을 지정하지 않으면, 스프링 `ApplicationContext` 내에서 각 애스펙트 클래스(`@Aspect` 붙은 클래스)당 **단 하나의 인스턴스**만 생성됩니다. 모든 어드바이스 호출은 이 **공유된 단일 애스펙트 인스턴스**를 통해 이루어집니다.
- **AspectJ 용어:** AspectJ에서는 이것을 싱글톤(singleton) 인스턴스화 모델이라고 부릅니다.
- **특징:** 가장 일반적이고 단순한 모델입니다. 애스펙트가 상태(state)를 가지지 않거나, 모든 대상 객체에 걸쳐 공유되어야 하는 상태를 가질 때 적합합니다.

---

**2. `perthis` 인스턴스화 모델:**

- **개념:** 포인트컷 표현식과 일치하는 **각 고유한 프록시 객체(this)** 마다 **별도의 애스펙트 인스턴스**가 생성되는 모델입니다.
- **선언 방법:** `@Aspect` 어노테이션의 값으로 `perthis(포인트컷표현식)` 구문을 사용합니다. 이 포인트컷 표현식은 어떤 프록시 객체들이 각자의 애스펙트 인스턴스를 가질지 결정합니다.

    ```java
    // 포인트컷 표현식("execution(* com.xyz..service.*.*(..))")과 일치하는
    // *각각의 고유한 서비스 객체(프록시)* 마다 MyAspect 인스턴스가 하나씩 생성됨
    @Aspect("perthis(execution(* com.xyz..service.*.*(..)))")
    public class MyAspect {
        private int someState; // ★ 이 상태는 특정 서비스 객체(프록시) 인스턴스에 귀속됨 ★
    
        @Before("execution(* com.xyz..service.*.*(..))")
        public void recordServiceUsage() {
            // 이 어드바이스는 이 애스펙트 인스턴스와 연결된
            // *바로 그* 서비스 객체의 메소드가 호출될 때만 실행됨
            this.someState++;
            System.out.println("State for " + this + ": " + someState);
        }
    }
    ```

- **동작:**
  1. 어떤 서비스 빈(예: `serviceA`, `serviceB`)의 메소드가 **처음으로** `perthis` 포인트컷 표현식과 매칭되면, 해당 **프록시 객체(`serviceA` 프록시)만을 위한** 새로운 `MyAspect` 인스턴스(`aspectForA`)가 생성됩니다.
  2. 이후 `serviceA` 프록시의 메소드가 호출될 때는 **항상 `aspectForA`** 의 `recordServiceUsage` 어드바이스가 실행됩니다. `aspectForA`는 `serviceA` 프록시에 대한 상태(`someState`)를 독립적으로 관리할 수 있습니다.
  3. 만약 다른 서비스 빈(`serviceB`)의 메소드가 호출되면, 마찬가지로 해당 **프록시 객체(`serviceB` 프록시)만을 위한** 또 다른 `MyAspect` 인스턴스(`aspectForB`)가 생성됩니다.
  4. `serviceB` 프록시의 메소드가 호출될 때는 **`aspectForB`** 의 어드바이스가 실행됩니다. `aspectForA`와 `aspectForB`는 서로 다른 상태를 가집니다.
- **생명주기:** `perthis` 애스펙트 인스턴스는 해당 프록시 객체가 처음 어드바이스될 때 생성되고, 프록시 객체(보통은 싱글톤 빈이므로 컨테이너 종료 시)가 스코프를 벗어날 때 함께 사라집니다.
- **사용 시나리오:** 어드바이스가 각 **프록시 객체별로 독립적인 상태**를 유지해야 할 때 사용합니다. (예: 각 서비스 인스턴스의 호출 횟수 추적)

---

**3. `pertarget` 인스턴스화 모델:**

- **개념:** `perthis`와 매우 유사하지만, 기준이 프록시 객체(`this`)가 아니라 **원본 대상 객체(`target`)** 입니다. 포인트컷 표현식과 일치하는 **각 고유한 대상 객체**마다 **별도의 애스펙트 인스턴스**가 생성됩니다.
- **선언 방법:** `@Aspect("pertarget(포인트컷표현식)")`
- **동작:** `perthis`와 거의 동일하지만, 애스펙트 인스턴스가 프록시 객체가 아닌 원본 대상 객체와 1:1로 연결됩니다. 스프링 AOP에서는 프록시와 타겟이 대부분 1:1 관계이므로 `perthis`와 `pertarget`의 실질적인 차이가 크지 않은 경우가 많습니다. 하지만 개념적으로는 구별됩니다.
- **사용 시나리오:** 어드바이스가 각 **원본 대상 객체별로 독립적인 상태**를 유지해야 할 때 사용합니다.

---

**4. `pertypewithin` 인스턴스화 모델 (문서에는 언급되었지만 예제는 없음):**

- **개념:** `@Aspect("pertypewithin(타입패턴)")` 형태로 사용하며, 지정된 타입 패턴(`within` 지정자와 유사)과 **일치하는 각 타입**마다 **하나의 애스펙트 인스턴스**가 생성됩니다.
- **예시:** `@Aspect("pertypewithin(com.xyz.service..*)")` 라고 하면, `com.xyz.service` 패키지 및 하위 패키지에 속하는 **각 클래스 타입별로** 애스펙트 인스턴스가 하나씩 생성됩니다. (예: `ServiceA`용 애스펙트 인스턴스, `ServiceB`용 애스펙트 인스턴스 등) 동일한 타입의 여러 빈 인스턴스가 있더라도 해당 타입에 대해서는 애스펙트 인스턴스가 하나만 생성됩니다.
- **사용 시나리오:** 특정 타입 그룹 전체에 걸쳐 공유되지만, 다른 타입 그룹과는 분리되어야 하는 상태를 애스펙트가 가져야 할 때 사용될 수 있습니다. (상대적으로 덜 사용됨)

**결론:**

스프링 AOP 애스펙트는 기본적으로 **싱글톤**으로 생성되어 모든 대상 객체에 대해 공유됩니다. 하지만 `@Aspect` 어노테이션에 `perthis(...)`, `pertarget(...)`, `pertypewithin(...)` 절을 추가하여 **특정 프록시 객체별, 특정 대상 객체별, 또는 특정 타입별로 별도의 애스펙트 인스턴스**를 생성하도록 생명주기 모델을 변경할 수 있습니다. 이는 애스펙트가 각 대상별로 독립적인 상태를 유지해야 하는 고급 시나리오에서 사용될 수 있습니다.
