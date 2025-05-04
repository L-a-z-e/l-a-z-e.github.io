---
title: Spring Framework LoadTimeWeaver
description: 
author: laze
date: 2025-05-04 00:00:10 +0900
categories: [Dev, SpringBoot]
tags: [SpringBoot]
---
**Registering a LoadTimeWeaver**

`LoadTimeWeaver`는 스프링이 클래스를 자바 가상 머신(JVM)에 로드할 때 동적으로 변환(transform)하는 데 사용됩니다.

로드 타임 위빙(load-time weaving)을 활성화하려면, 다음 예제와 같이 `@Configuration` 클래스 중 하나에 `@EnableLoadTimeWeaving`을 추가할 수 있습니다:

```java
// Java
@Configuration
@EnableLoadTimeWeaving
public class AppConfig {
}
```

```kotlin
// Kotlin
@Configuration
@EnableLoadTimeWeaving
class AppConfig
```

또는, XML 설정의 경우, `context:load-time-weaver` 요소를 사용할 수 있습니다:

```xml
<beans>
	<context:load-time-weaver/>
</beans>
```

`ApplicationContext`에 대해 일단 설정되면, 해당 `ApplicationContext` 내의 모든 빈은 `LoadTimeWeaverAware`를 구현할 수 있으며, 이를 통해 로드 타임 위버 인스턴스에 대한 참조를 받게 됩니다.

이는 스프링의 JPA 지원과 함께 특히 유용한데, JPA 클래스 변환을 위해 로드 타임 위빙이 필요할 수 있기 때문입니다.

---

**전체 주제: `LoadTimeWeaver` 등록하기**

이 부분은 스프링이 자바 **클래스 파일(.class)을 JVM(자바 가상 머신)에 로드하는 시점**에 해당 클래스의 **바이트코드를 동적으로 변경(위빙, Weaving)** 할 수 있도록 하는 **로드 타임 위빙** 기능을 어떻게 활성화하는지에 대한 내용입니다.

**핵심 아이디어:** 클래스가 메모리에 올라가는 바로 그 순간에 클래스 내용을 살짝 바꿔치기(위빙)해서 추가 기능을 넣자! 이걸 가능하게 해주는 도구(`LoadTimeWeaver`)를 스프링에 등록하자.

---

**1. 로드 타임 위빙 (Load-Time Weaving, LTW)이란?**

- **위빙(Weaving):** AOP(관점 지향 프로그래밍) 용어로, 핵심 로직(Core Concern) 코드에 부가 기능(Cross-cutting Concern, 예: 로깅, 트랜잭션) 코드를 **끼워 넣는(삽입하는)** 과정을 말합니다.
- **로드 타임(Load-Time):** 위빙이 일어나는 시점을 의미합니다. 즉, **클래스 로더(Class Loader)가 클래스 파일(.class)을 읽어서 JVM 메모리에 올리는 바로 그 시점**에 바이트코드를 변경하여 부가 기능 코드를 삽입합니다.
- **LTW 장점:**
  - 컴파일 시점이나 런타임(프록시 생성) 시점 위빙과 달리, 원본 코드나 설계 변경 없이 클래스 로딩 시점에 AOP를 적용할 수 있습니다.
  - 프록시 방식으로는 적용하기 어려운 경우(예: `final` 클래스, `private` 메소드 호출 등)에도 AOP 적용이 가능할 수 있습니다. (AspectJ LTW의 경우)
  - JPA 같은 기술에서 엔티티 클래스의 특정 필드 접근을 추적하거나 지연 로딩(Lazy Loading)을 구현하기 위해 바이트코드 조작이 필요한 경우가 있는데, LTW가 이를 가능하게 합니다.
- **LTW 단점:**
  - 설정이 다소 복잡할 수 있습니다. (JVM 실행 시 특별한 에이전트(agent) 설정이 필요할 수 있음)
  - 클래스 로딩 시간이 약간 증가할 수 있습니다.
  - 디버깅이 조금 더 어려워질 수 있습니다.

---

**2. `LoadTimeWeaver` 인터페이스:**

- 스프링에서 로드 타임 위빙 기능을 추상화한 인터페이스입니다.
- 실제 위빙 작업은 특정 환경(예: Tomcat, JBoss, AspectJ 에이전트)에 맞는 구체적인 `LoadTimeWeaver` 구현체가 담당합니다. 스프링은 환경을 감지하여 적절한 구현체를 자동으로 선택하거나 설정할 수 있도록 도와줍니다.

---

**3. 로드 타임 위빙 활성화 및 `LoadTimeWeaver` 등록 방법:**

스프링 `ApplicationContext`에 로드 타임 위빙 기능을 활성화하고 관련 `LoadTimeWeaver` 빈을 등록하는 방법은 두 가지입니다.

- **(1) Java Config 방식 (`@EnableLoadTimeWeaving`):**
  - `@Configuration` 클래스 중 하나에 `@EnableLoadTimeWeaving` 어노테이션을 추가합니다.
  - 이 어노테이션 하나만으로 스프링은 현재 실행 환경에 맞는 `LoadTimeWeaver` 구현체를 찾아서 빈으로 등록하고 로드 타임 위빙 기능을 활성화합니다. (매우 간편)

    ```java
    @Configuration
    @EnableLoadTimeWeaving // ★ 로드 타임 위빙 활성화 ★
    public class AppConfig {
        // ... 다른 빈 설정들 ...
    }
    
    ```

- **(2) XML 설정 방식 (`<context:load-time-weaver/>`):**
  - XML 설정 파일에 `<context:load-time-weaver/>` 태그를 추가합니다.
  - 이 태그 역시 `@EnableLoadTimeWeaving`과 동일하게 환경에 맞는 `LoadTimeWeaver` 빈을 자동으로 등록하고 기능을 활성화합니다.

    ```xml
    <beans xmlns:context="<http://www.springframework.org/schema/context>" ...>
        <context:load-time-weaver/> <!-- ★ 로드 타임 위빙 활성화 ★ -->
        <!-- ... 다른 빈 설정들 ... -->
    </beans>
    ```


---

**4. `LoadTimeWeaverAware` 인터페이스:**

- 일단 로드 타임 위빙이 활성화되면(`@EnableLoadTimeWeaving` 또는 `<context:load-time-weaver/>` 사용), 해당 `ApplicationContext` 내의 **모든 빈**은 `org.springframework.context.LoadTimeWeaverAware` 인터페이스를 구현할 수 있습니다.
- 이 인터페이스를 구현한 빈에게는 스프링이 자동으로 **활성화된 `LoadTimeWeaver` 인스턴스에 대한 참조**를 주입해줍니다. (`setLoadTimeWeaver` 메소드 호출)
- **주요 사용처:**
  - **스프링 JPA:** `LocalContainerEntityManagerFactoryBean` 같은 JPA 설정 빈은 내부적으로 `LoadTimeWeaver`를 사용하여 JPA 엔티티 클래스를 동적으로 변환(transform)해야 할 수 있습니다 (JPA Provider에 따라). 이때 `LoadTimeWeaverAware`를 통해 `LoadTimeWeaver`를 주입받아 사용합니다.
  - **AspectJ LTW 연동:** AspectJ를 사용하여 로드 타임 위빙을 직접 제어해야 하는 고급 시나리오에서 사용될 수 있습니다.

**요약:**

로드 타임 위빙(LTW)은 JVM이 클래스를 로드하는 시점에 클래스 바이트코드를 동적으로 변경하여 AOP 등의 기능을 적용하는 기술입니다. 스프링에서는 `@EnableLoadTimeWeaving` 어노테이션(Java Config) 또는 `<context:load-time-weaver/>` 태그(XML)를 사용하여 이 기능을 활성화하고 환경에 맞는 `LoadTimeWeaver` 빈을 등록할 수 있습니다. 일단 활성화되면, 다른 빈들은 `LoadTimeWeaverAware` 인터페이스를 구현하여 등록된 `LoadTimeWeaver` 인스턴스를 주입받아 사용할 수 있으며, 이는 특히 스프링 JPA 연동 시 유용하게 사용됩니다.
