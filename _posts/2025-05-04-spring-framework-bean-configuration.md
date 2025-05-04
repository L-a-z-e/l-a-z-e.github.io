---
title: Spring Framework @Bean and @Configuration
description: 
author: laze
date: 2025-05-04 00:00:04 +0900
categories: [Dev, SpringBoot]
tags: [SpringBoot]
---
**Basic Concepts: @Bean and @Configuration**

스프링의 자바 설정 지원에서 핵심적인 요소(artifacts)는 `@Configuration` 어노테이션이 달린 클래스와 `@Bean` 어노테이션이 달린 메소드입니다.

`@Bean` 어노테이션은 메소드가 스프링 IoC 컨테이너에 의해 관리될 새로운 객체를 인스턴스화하고, 설정하고, 초기화한다는 것을 나타내는 데 사용됩니다.

스프링의 `<beans/>` XML 설정에 익숙한 사람들에게 `@Bean` 어노테이션은 `<bean/>` 요소와 동일한 역할을 합니다.

`@Bean` 어노테이션이 달린 메소드는 모든 스프링 `@Component`와 함께 사용할 수 있습니다.

그러나 대부분 `@Configuration` 빈과 함께 사용됩니다.

클래스에 `@Configuration` 어노테이션을 다는 것은 그 주된 목적이 빈 정의의 소스(source)임을 나타냅니다.

더욱이, `@Configuration` 클래스는 동일한 클래스 내의 다른 `@Bean` 메소드를 호출함으로써 빈 간의 의존성(inter-bean dependencies)이 정의되도록 합니다.

가장 간단한 `@Configuration` 클래스는 다음과 같습니다:

```java
// Java
@Configuration
public class AppConfig {

	@Bean
	public MyServiceImpl myService() {
		return new MyServiceImpl();
	}
}
```

```kotlin
// Kotlin
@Configuration
class AppConfig {

    @Bean
    fun myService(): MyServiceImpl {
        return MyServiceImpl()
    }
}
// Assuming MyServiceImpl class exists
```

앞의 `AppConfig` 클래스는 다음 스프링 `<beans/>` XML과 동일합니다:

```xml
<beans>
	<bean id="myService" class="com.acme.services.MyServiceImpl"/>
</beans>
```

**@Configuration 클래스에서 @Bean 메소드 간의 로컬 호출 유무**

일반적인 시나리오에서, `@Bean` 메소드는 `@Configuration` 클래스 내에 선언되어야 하며,

이는 완전한 구성 클래스 처리(full configuration class processing)가 적용되고 따라서 메소드 간 참조(cross-method references)가 컨테이너의 생명주기 관리로 리디렉션되도록 보장합니다.

이는 동일한 `@Bean` 메소드가 일반적인 자바 메소드 호출을 통해 우연히 호출되는 것을 방지하며, 추적하기 어려운 미묘한 버그를 줄이는 데 도움이 됩니다.

`@Bean` 메소드가 `@Configuration`으로 어노테이션되지 않은 클래스 내에 선언되거나 `@Configuration(proxyBeanMethods=false)`가 선언된 경우, "라이트(lite)" 모드로 처리된다고 합니다.

이러한 시나리오에서 `@Bean` 메소드는 특별한 런타임 처리(즉, CGLIB 하위 클래스 생성 없음) 없이 효과적으로 범용 팩토리 메소드 메커니즘입니다.

이러한 메소드에 대한 커스텀 자바 호출은 컨테이너에 의해 가로채지지 않으므로 일반 메소드 호출처럼 동작하며, 주어진 빈에 대해 기존 싱글톤(또는 스코프 지정된) 인스턴스를 재사용하는 대신 매번 새로운 인스턴스를 생성합니다.

결과적으로, 런타임 프록시가 없는 클래스의 `@Bean` 메소드는 빈 간 의존성을 전혀 선언하도록 의도되지 않았습니다.

대신, 포함하는 컴포넌트의 필드에서 작동하고, 선택적으로 자동 와이어링된 협력자를 받기 위해 팩토리 메소드가 선언할 수 있는 인수에서 작동할 것으로 예상됩니다.

따라서 이러한 `@Bean` 메소드는 다른 `@Bean` 메소드를 호출할 필요가 전혀 없습니다.

모든 그러한 호출은 대신 팩토리 메소드 인수를 통해 표현될 수 있습니다.

여기서 긍정적인 부작용은 런타임 시 CGLIB 하위 클래스 생성을 적용할 필요가 없어 오버헤드와 메모리 사용량(footprint)을 줄인다는 것입니다.

`@Bean` 및 `@Configuration` 어노테이션은 다음 섹션에서 자세히 논의됩니다. 그러나 먼저 자바 기반 설정을 사용하여 스프링 컨테이너를 생성하는 다양한 방법을 다룹니다.

---

**전체 주제: 기본 개념 - `@Bean`과 `@Configuration`**

이 부분은 XML 설정 파일(`beans.xml`)을 대체하여, **자바 클래스와 메소드를 사용하여 스프링 빈을 정의하고 의존 관계를 설정**하는 방법에 대한 기본적인 소개입니다.

**핵심 아이디어:** XML 대신 자바 코드로 스프링 설정을 작성하자! `@Configuration` 클래스와 `@Bean` 메소드가 그 중심 역할을 한다.

---

**1. `@Bean` 어노테이션: 빈(Bean) 정의**

- **역할:** 특정 **메소드** 위에 붙여서, **"이 메소드가 반환하는 객체는 스프링 컨테이너가 관리해야 할 빈(Bean)이다"** 라고 알려주는 역할을 합니다.
- **동작:**
  1. 스프링 컨테이너는 `@Bean`이 붙은 메소드를 **호출**합니다.
  2. 메소드 내부 로직에 따라 **객체가 생성되고, 필요한 설정(예: 세터 호출)이 이루어집니다.**
  3. 메소드가 **반환하는 객체**를 스프링 컨테이너가 **빈으로 등록하고 관리**합니다.
  4. 기본적으로 **메소드 이름**이 해당 빈의 **이름(ID)** 이 됩니다. (예: `myService()` 메소드 -> 빈 이름: `myService`) `@Bean(name = "다른이름")` 처럼 이름을 지정할 수도 있습니다.
- **XML과의 비교:** `@Bean` 메소드는 XML 설정의 **`<bean/>` 태그와 동일한 역할**을 합니다. 객체 생성 및 설정을 정의합니다.
- **사용 위치:** `@Bean` 어노테이션은 `@Configuration` 클래스 내에서 가장 흔하게 사용되지만, `@Component` 등 다른 스프링 스테레오타입 어노테이션이 붙은 클래스 안에서도 사용할 수 있습니다. (단, 동작 방식에 약간의 차이가 있을 수 있음 - 다음 섹션에서 설명)

---

**2. `@Configuration` 어노테이션: 설정 정보의 근원**

- **역할:** 특정 **클래스** 위에 붙여서, **"이 클래스는 스프링 빈 설정 정보를 정의하는 특별한 클래스이다"** 라고 알려주는 역할을 합니다.
- **핵심 기능:**
  1. **빈 정의 소스 표시:** 이 클래스 안에 있는 `@Bean` 메소드들이 스프링 빈을 정의한다는 것을 명확히 합니다.
  2. **빈 간 의존성 처리 활성화 (중요!):** `@Configuration` 클래스 내에서는 한 `@Bean` 메소드가 **다른 `@Bean` 메소드를 직접 호출**하여 그 반환값을 의존성으로 사용할 수 있습니다. 스프링은 이 호출을 **가로채서(intercept)** 항상 올바른 빈 인스턴스(예: 싱글톤이면 항상 같은 인스턴스)를 반환하도록 보장합니다. (내부적으로 CGLIB 프록시 사용)
- **가장 간단한 예시:**

    ```java
    @Configuration // ★ 이 클래스가 설정 정보 클래스임을 나타냄 ★
    public class AppConfig {
    
        @Bean // ★ myService 빈 정의 ★
        public MyServiceImpl myService() {
            // 객체를 생성하고 설정하여 반환
            return new MyServiceImpl();
        }
    }
    ```

  위 자바 코드는 아래 XML 설정과 동일한 의미를 가집니다:

    ```xml
    <beans>
        <bean id="myService" class="com.acme.services.MyServiceImpl"/>
    </beans>
    ```


---

**3. `@Configuration` 클래스 내 `@Bean` 메소드 간의 호출 ("Full" 모드 vs "Lite" 모드)**

이 부분은 `@Bean` 메소드를 `@Configuration` 클래스 안에서 사용하는 것과 그렇지 않은 경우의 중요한 차이점을 설명합니다.

- **"Full" 모드 (일반적인 `@Configuration` 클래스):**
  - 클래스에 `@Configuration` 어노테이션이 붙어 있고, `proxyBeanMethods=true` (기본값)인 경우입니다.
  - **핵심:** 스프링은 이 클래스에 대해 **CGLIB 프록시**를 생성하여 런타임에 동작을 향상시킵니다.
  - **`@Bean` 메소드 간 호출:** 만약 한 `@Bean` 메소드(예: `beanA()`)가 다른 `@Bean` 메소드(예: `beanB()`)를 직접 호출하면 (`beanB()` 호출), 스프링의 CGLIB 프록시가 이 호출을 **가로채서** `beanB`라는 이름의 빈이 이미 컨테이너에 존재하면 **기존 인스턴스(싱글톤)** 를 반환하고, 없으면 생성하여 등록하고 반환합니다. **절대로 `beanB()` 메소드 본문이 여러 번 실행되어 새 객체가 계속 만들어지지 않습니다.**
  - **장점:** 빈 간의 의존 관계를 자연스러운 자바 메소드 호출 방식으로 정의할 수 있고, 스프링이 싱글톤 스코프 등을 정확하게 관리해줍니다.
- **"Lite" 모드 (`@Component` 내 `@Bean` 또는 `@Configuration(proxyBeanMethods=false)`):**
  - `@Bean` 메소드가 `@Configuration`이 아닌 클래스(예: `@Component`) 안에 있거나, `@Configuration(proxyBeanMethods=false)`로 프록시 생성을 껐을 때 해당됩니다.
  - **핵심:** 스프링은 이 클래스에 대해 CGLIB 프록시를 **생성하지 않습니다.** `@Bean` 메소드는 일반적인 **팩토리 메소드**처럼 동작합니다.
  - **`@Bean` 메소드 간 호출:** 만약 한 `@Bean` 메소드가 다른 `@Bean` 메소드를 직접 호출하면, 이는 **평범한 자바 메소드 호출**과 같습니다. 스프링이 가로채지 않으므로, 호출될 때마다 **매번 새로운 객체가 생성될 수 있습니다.** (싱글톤 보장 안 됨)
  - **용도:** 이 방식에서는 `@Bean` 메소드 간 호출로 의존성을 정의하는 것을 **피해야 합니다.** 대신, 필요한 의존성은 메소드 **파라미터**를 통해 `@Autowired` 등으로 주입받아 사용해야 합니다.
  - **장점:** CGLIB 프록시 생성 오버헤드가 없으므로 약간의 성능 향상 및 메모리 절약 효과가 있을 수 있습니다. 하지만 "Full" 모드의 안정성과 명확성을 잃을 수 있습니다.

**결론:**

`@Configuration`과 `@Bean`은 자바 코드로 스프링 설정을 작성하는 핵심 어노테이션입니다. `@Configuration`은 설정 클래스임을 나타내고 빈 간의 의존성 관리를 위한 프록시 기능을 활성화하며(Full 모드), `@Bean`은 해당 메소드가 반환하는 객체를 스프링 빈으로 등록하도록 지시합니다. "Full" 모드는 안정적인 의존성 관리를 제공하며, "Lite" 모드는 프록시 오버헤드가 없지만 `@Bean` 메소드 간 호출에 주의해야 합니다. 일반적으로는 "Full" 모드(`@Configuration` 사용)가 권장됩니다.

---
