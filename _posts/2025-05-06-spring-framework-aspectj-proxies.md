---
title: Programmatic Creation of @AspectJ Proxies
description: 
author: laze
date: 2025-05-06 00:00:04 +0900
categories: [Dev, SpringBoot]
tags: [SpringBoot]
---
**Programmatic Creation of @AspectJ Proxies**

`<aop:config>` 또는 `<aop:aspectj-autoproxy>`를 사용하여 구성에서 애스펙트(aspects)를 선언하는 것 외에도,

대상 객체를 어드바이스(advise)하는 프록시를 프로그래밍 방식으로 생성하는 것도 가능합니다.

스프링의 AOP API에 대한 전체 세부 내용은 다음 장을 참조하십시오. 여기서는 `@AspectJ` 애스펙트를 사용하여 프록시를 자동으로 생성하는 기능에 초점을 맞추고자 합니다.

`org.springframework.aop.aspectj.annotation.AspectJProxyFactory` 클래스를 사용하여 하나 이상의 `@AspectJ` 애스펙트에 의해 어드바이스되는 대상 객체에 대한 프록시를 생성할 수 있습니다.

이 클래스의 기본 사용법은 다음 예제에서 보여주듯이 매우 간단합니다:

```java
// Java
import org.springframework.aop.aspectj.annotation.AspectJProxyFactory;
// Assuming targetObject, SecurityManager (@Aspect class), usageTracker (@Aspect instance),
// and MyInterfaceType exist.

// 주어진 대상 객체에 대한 프록시를 생성할 수 있는 팩토리 생성
AspectJProxyFactory factory = new AspectJProxyFactory(targetObject);

// 애스펙트 추가, 클래스는 @AspectJ 애스펙트여야 함
// 다른 애스펙트로 필요한 만큼 여러 번 호출 가능
factory.addAspect(SecurityManager.class);

// 기존 애스펙트 인스턴스를 추가할 수도 있음, 제공된 객체의 타입은
// @AspectJ 애스펙트여야 함
factory.addAspect(usageTracker);

// 이제 프록시 객체 가져오기...
// Get proxy based on target's interfaces or target class
MyInterfaceType proxy = factory.getProxy();
// Or specify classloader / interfaces if needed:
// MyInterfaceType proxy = factory.getProxy(MyInterfaceType.class.getClassLoader());
// MyInterfaceType proxy = factory.getProxy(MyInterfaceType.class.getClassLoader(), MyInterfaceType.class);
```

```kotlin
// Kotlin
import org.springframework.aop.aspectj.annotation.AspectJProxyFactory
// Assuming targetObject, SecurityManager (@Aspect class), usageTracker (@Aspect instance),
// and MyInterfaceType exist.

// 주어진 대상 객체에 대한 프록시를 생성할 수 있는 팩토리 생성
val factory = AspectJProxyFactory(targetObject)

// 애스펙트 추가, 클래스는 @AspectJ 애스펙트여야 함
// 다른 애스펙트로 필요한 만큼 여러 번 호출 가능
factory.addAspect(SecurityManager::class.java)

// 기존 애스펙트 인스턴스를 추가할 수도 있음, 제공된 객체의 타입은
// @AspectJ 애스펙트여야 함
factory.addAspect(usageTracker)

// 이제 프록시 객체 가져오기...
// Get proxy based on target's interfaces or target class
val proxy: MyInterfaceType = factory.getProxy() // Type inference might work
// Or specify classloader / interfaces if needed:
// val proxy = factory.getProxy(MyInterfaceType::class.java.classLoader) as MyInterfaceType
// val proxy = factory.getProxy(MyInterfaceType::class.java.classLoader, MyInterfaceType::class.java)
```

---

**전체 주제: `@AspectJ` 프록시의 프로그래밍 방식 생성 (Programmatic Creation of @AspectJ Proxies)**

이 부분은 스프링 IoC 컨테이너의 설정(XML의 `<aop:config>`나 Java Config의 `@EnableAspectJAutoProxy`)을 통하지 않고, **자바 코드 내에서 특정 객체(Target)와 특정 `@AspectJ` 애스펙트들을 직접 지정하여 AOP 프록시를 만드는 방법**을 설명합니다.

**핵심 아이디어:** 스프링 설정 파일 없이, 코드 레벨에서 내가 원하는 객체에 내가 원하는 `@AspectJ` 애스펙트를 적용한 프록시를 직접 만들자!

---

**1. 언제 사용할까?**

- **일반적인 경우는 아닙니다.** 대부분의 경우, 스프링 컨테이너가 제공하는 자동 프록시 생성 기능 (`@EnableAspectJAutoProxy` 또는 `<aop:aspectj-autoproxy/>`)을 사용하는 것이 훨씬 편리하고 관리하기 좋습니다.
- **특수한 경우:**
  - 스프링 컨테이너를 사용하지 않는 환경에서 AOP 프록시를 생성해야 할 때.
  - 컨테이너 외부에서 생성된 객체에 동적으로 AOP를 적용해야 할 때.
  - 단위 테스트 등에서 특정 객체에 대한 프록시를 수동으로 생성하여 테스트해야 할 때.
  - 매우 동적인 방식으로 프록시 생성 및 애스펙트 적용을 제어해야 할 때.

---

**2. `AspectJProxyFactory` 사용법**

스프링은 `@AspectJ` 애스펙트를 사용하여 프록시를 프로그래밍 방식으로 생성하기 위한 `org.springframework.aop.aspectj.annotation.AspectJProxyFactory` 클래스를 제공합니다.

- **단계별 사용법:**
  1. **팩토리 생성:** 프록시를 만들 **대상 객체(Target Object)** 를 생성자의 인자로 전달하여 `AspectJProxyFactory` 인스턴스를 생성합니다.

      ```java
      Object targetObject = new MyTargetImplementation(); // 원본 대상 객체
      AspectJProxyFactory factory = new AspectJProxyFactory(targetObject);
      ```

  2. **애스펙트 추가:** 생성된 팩토리에 적용할 `@AspectJ` 애스펙트들을 추가합니다. 두 가지 방법이 있습니다.
    - **클래스 타입으로 추가:** `@Aspect` 어노테이션이 붙은 **애스펙트 클래스의 `.class` 객체**를 전달합니다. 팩토리가 내부적으로 해당 클래스의 인스턴스를 생성하여 사용합니다 (기본적으로 싱글톤).

        ```java
        // SecurityManager 클래스는 @Aspect 어노테이션이 있어야 함
        factory.addAspect(SecurityManager.class);
        ```

    - **인스턴스로 추가:** 이미 생성된 **애스펙트 객체 인스턴스**를 직접 전달합니다. 이 객체의 클래스에는 `@Aspect` 어노테이션이 있어야 합니다.

        ```java
        UsageTrackingAspect usageTracker = new UsageTrackingAspect(); // 미리 생성된 애스펙트 인스턴스
        // usageTracker 객체의 클래스는 @Aspect 어노테이션이 있어야 함
        factory.addAspect(usageTracker);
        ```

    - `addAspect()` 메소드는 여러 번 호출하여 **여러 개의 애스펙트**를 추가할 수 있습니다.
  3. **프록시 객체 얻기:** `getProxy()` 메소드를 호출하여 최종적으로 **애스펙트가 적용된 프록시 객체**를 얻습니다.

      ```java
      // 대상 객체가 구현한 인터페이스 기반 프록시 (JDK 동적 프록시) 또는
      // 대상 객체 클래스 기반 프록시 (CGLIB)를 자동으로 생성하여 반환
      MyInterfaceType proxy = factory.getProxy();
      
      // 필요하다면 ClassLoader나 프록시할 특정 인터페이스를 지정하여 getProxy() 호출 가능
      // MyInterfaceType proxy = factory.getProxy(classLoader);
      // MyInterfaceType proxy = factory.getProxy(classLoader, MyInterfaceType.class);
      ```

    - `getProxy()` 메소드는 대상 객체가 인터페이스를 구현했는지 여부에 따라 적절한 프록시 방식(JDK 또는 CGLIB)을 사용하여 프록시를 생성합니다. (이전 섹션의 프록시 메커니즘과 동일)

---

**3. 예시 코드 종합:**

```java
// 필요 클래스/인터페이스 정의 (가정)
interface MyInterfaceType { void doSomething(); }
class MyTargetImplementation implements MyInterfaceType { public void doSomething() { System.out.println("Target doing something"); } }
@Aspect class SecurityManager { @Before("execution(* doSomething())") public void check() { System.out.println("Security check..."); } }
@Aspect class UsageTrackingAspect { /* ... */ }

// --- 프로그래밍 방식 프록시 생성 ---
public class ProgrammaticProxyExample {
    public static void main(String[] args) {
        // 1. 대상 객체 생성
        MyInterfaceType targetObject = new MyTargetImplementation();

        // 2. AspectJProxyFactory 생성
        AspectJProxyFactory factory = new AspectJProxyFactory(targetObject);

        // 3. 애스펙트 추가 (클래스 타입)
        factory.addAspect(SecurityManager.class);

        // 3. 애스펙트 추가 (인스턴스)
        UsageTrackingAspect usageTracker = new UsageTrackingAspect();
        factory.addAspect(usageTracker);

        // 4. 프록시 객체 얻기
        MyInterfaceType proxy = factory.getProxy();

        // 5. 프록시 객체 사용
        proxy.doSomething();
        // 출력 결과 예상:
        // Security check...
        // (UsageTrackingAspect의 어드바이스 실행)
        // Target doing something
    }
}
```

**결론:**

`AspectJProxyFactory`를 사용하면 스프링 IoC 컨테이너 설정 없이도 **프로그래밍 방식으로 특정 대상 객체에 `@AspectJ` 애스펙트를 적용하여 AOP 프록시를 생성**할 수 있습니다.

팩토리를 생성하고, 적용할 애스펙트(클래스 또는 인스턴스)를 추가한 다음, `getProxy()` 메소드를 호출하여 프록시 객체를 얻는 간단한 과정을 따릅니다.

이는 단위 테스트나 특정 동적 AOP 적용 시나리오 등 특수한 경우에 유용할 수 있지만, 일반적인 애플리케이션 개발에서는 컨테이너의 자동 프록시 기능을 사용하는 것이 더 편리합니다.

---
위빙(Weaving)은 AOP(관점 지향 프로그래밍)에서 **매우 핵심적인 개념**입니다. "짜다", "엮다"라는 뜻처럼, **분리된 코드 조각들을 하나로 합치는 과정**을 의미합니다.

**위빙(Weaving)이란?**

- **개념:** AOP에서 **애스펙트(Aspect, 부가 기능 모듈)** 를 **타겟 객체(Target Object, 핵심 로직을 담은 객체)** 에 **적용(application)하여 최종적으로 어드바이스된 객체(Advised Object)** 를 만드는 **과정 전체**를 말합니다.
- **목표:** 핵심 로직 코드에는 손대지 않으면서, 마치 원래부터 있었던 것처럼 부가 기능(어드바이스)이 타겟 객체의 특정 시점(조인 포인트)에 실행되도록 만드는 것입니다.
- **비유:**
  - 씨실(부가 기능)과 날실(핵심 로직)을 엮어서 하나의 옷감(최종 객체)을 짜는 과정.
  - 기본 자동차(타겟 객체)에 튜닝 부품(애스펙트)을 장착하여 튜닝된 자동차(어드바이스된 객체)를 만드는 과정.
  - 연극 대본(타겟 로직)에 조명, 음향 효과(애스펙트)를 추가하여 실제 공연(어드바이스된 객체 실행)을 완성하는 과정.

**위빙이 일어나는 시점:**

위빙은 언제 애스펙트와 타겟 객체를 연결하느냐에 따라 크게 세 가지 시점으로 나뉩니다.

1. **컴파일 타임 위빙 (Compile-time Weaving):**
- **시점:** 자바 소스 코드를 **컴파일하는 시점**에 위빙이 일어납니다.
- **방법:** AspectJ 컴파일러(`ajc`)를 사용하여 소스 코드와 애스펙트 코드를 함께 컴파일합니다. 컴파일 결과로 생성되는 `.class` 파일(바이트코드) 자체에 이미 부가 기능 코드가 **삽입**되어 있습니다.
- **장점:** 실행 시점 성능 오버헤드가 가장 적습니다. 모든 종류의 조인 포인트(필드 접근 등)에 적용 가능합니다.
- **단점:** 별도의 AspectJ 컴파일러 설정이 필요하고 빌드 과정이 복잡해집니다. 원본 소스 코드를 수정하는 것과 유사한 결과를 낳습니다(바이트코드 레벨에서).
2. **로드 타임 위빙 (Load-time Weaving, LTW):**
- **시점:** 컴파일된 `.class` 파일이 **JVM의 클래스 로더에 의해 메모리로 로드되는 시점**에 위빙이 일어납니다.
- **방법:** JVM 실행 시 특별한 **위빙 에이전트(Weaving Agent)** (예: AspectJ 위빙 에이전트, `spring-instrument.jar`)를 사용하여 클래스 로더가 클래스 바이트코드를 읽을 때 중간에서 가로채 내용을 **동적으로 수정(변환, transform)** 하여 부가 기능 코드를 삽입합니다.
- **장점:** 컴파일 시점 위빙처럼 모든 종류의 조인 포인트를 지원하면서도, 원본 소스 코드나 컴파일된 바이트코드를 직접 수정하지 않습니다. 애플리케이션 서버 환경 등에서 유연하게 적용 가능합니다.
- **단점:** JVM 에이전트 설정이 필요하며, 클래스 로딩 시 약간의 오버헤드가 발생할 수 있습니다. 설정이 다소 복잡할 수 있습니다. (스프링은 이 과정을 좀 더 쉽게 할 수 있도록 지원)
3. **런타임 위빙 (Runtime Weaving):**
- **시점:** 애플리케이션이 **실행되는 도중**에, 스프링 컨테이너가 빈 객체를 생성할 때 위빙이 일어납니다.
- **방법:** **스프링 AOP가 사용하는 방식**입니다. 원본 대상 객체(Target)를 직접 수정하는 대신, 대상 객체를 **감싸는 프록시(Proxy) 객체**를 런타임에 동적으로 생성합니다. 이 프록시 객체가 부가 기능(어드바이스) 실행을 담당하고 내부적으로 원본 객체의 메소드를 호출합니다.
- **장점:** 별도의 컴파일러나 JVM 에이전트 설정 없이 순수 자바만으로 AOP를 구현할 수 있어 **가장 간단**합니다. 스프링 IoC 컨테이너와 긴밀하게 통합됩니다.
- **단점:** **메소드 실행 조인 포인트만** 지원합니다. 프록시 방식 자체의 한계(자가 호출 문제 등)가 있습니다. 약간의 런타임 성능 오버헤드가 있을 수 있습니다 (현대 JVM에서는 미미할 수 있음).

**결론:**

**위빙(Weaving)** 은 분리된 **부가 기능(애스펙트)** 을 **핵심 로직(타겟 객체)** 에 **적용하여 최종적으로 실행될 객체를 만드는 과정**입니다. 어떤 시점(컴파일, 로드, 런타임)에 이 작업을 수행하느냐에 따라 방식과 장단점이 달라집니다. 스프링 AOP는 기본적으로 **런타임 위빙(프록시 생성)** 방식을 사용하며, 필요하다면 AspectJ와 통합하여 **로드 타임 위빙**이나 **컴파일 타임 위빙**을 활용할 수도 있습니다.
