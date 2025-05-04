---
title: Spring Framework BeanFactory Api
description: 
author: laze
date: 2025-05-04 00:00:12 +0900
categories: [Dev, SpringBoot]
tags: [SpringBoot]
---
**BeanFactory API**

BeanFactory API는 스프링의 IoC 기능의 기반을 제공합니다.

그 구체적인 계약(contracts)은 주로 스프링의 다른 부분 및 관련 서드파티 프레임워크와의 통합에 사용되며,

그 `DefaultListableBeanFactory` 구현은 더 높은 수준의 `GenericApplicationContext` 컨테이너 내에서 핵심적인 위임(delegate) 객체입니다.

`BeanFactory` 및 관련 인터페이스들(예: `BeanFactoryAware`, `InitializingBean`, `DisposableBean`)은 다른 프레임워크 컴포넌트들에게 중요한 통합 지점입니다.

어노테이션이나 리플렉션조차 요구하지 않음으로써, 컨테이너와 컴포넌트 간의 매우 효율적인 상호 작용을 가능하게 합니다.

애플리케이션 레벨의 빈들은 동일한 콜백 인터페이스를 사용할 수 있지만, 일반적으로 어노테이션을 통하거나 프로그래밍 방식 설정을 통한 선언적 의존성 주입을 선호합니다.

핵심 `BeanFactory` API 레벨과 그 `DefaultListableBeanFactory` 구현은 구성 형식이나 사용될 컴포넌트 어노테이션에 대해 어떠한 가정도 하지 않는다는 점에 유의하십시오.

이러한 모든 방식들은 확장(extensions) (예: `XmlBeanDefinitionReader` 및 `AutowiredAnnotationBeanPostProcessor`)을 통해 제공되며, 공유된 `BeanDefinition` 객체를 핵심 메타데이터 표현으로 사용하여 작동합니다.

이것이 스프링 컨테이너를 매우 유연하고 확장 가능하게 만드는 본질입니다.

**BeanFactory or ApplicationContext?**

`BeanFactory`와 `ApplicationContext` 컨테이너 레벨 간의 차이점과 부트스트래핑(bootstrapping)에 미치는 영향에 대해 알아봅니다.

특별한 이유가 없다면 `ApplicationContext`를 사용해야 하며, 커스텀 부트스트래핑을 위한 일반적인 구현으로는 `GenericApplicationContext`와 그 하위 클래스인 `AnnotationConfigApplicationContext`가 있습니다.

이것들은 모든 일반적인 목적을 위한 스프링 핵심 컨테이너의 주요 진입점입니다:

구성 파일 로딩, 클래스패스 스캔 트리거, 빈 정의 및 어노테이션 클래스의 프로그래밍 방식 등록, 그리고 (5.0부터) 함수형 빈 정의 등록.

`ApplicationContext`는 `BeanFactory`의 모든 기능을 포함하므로, 빈 처리에 대한 완전한 제어가 필요한 시나리오를 제외하고는 일반적으로 일반 `BeanFactory`보다 권장됩니다.

`ApplicationContext`(예: `GenericApplicationContext` 구현) 내에서는 여러 종류의 빈이 관례(즉, 빈 이름 또는 빈 타입 - 특히 후처리기)에 의해 감지되는 반면, 일반 `DefaultListableBeanFactory`는 어떤 특별한 빈에 대해서도 인식하지 못합니다(agnostic).

어노테이션 처리 및 AOP 프록싱과 같은 많은 확장된 컨테이너 기능의 경우, `BeanPostProcessor` 확장 지점은 필수적입니다.

일반 `DefaultListableBeanFactory`만 사용하는 경우, 이러한 후처리기는 기본적으로 감지되고 활성화되지 않습니다.

실제로 빈 구성에는 아무런 문제가 없기 때문에 이 상황은 혼란스러울 수 있습니다.

오히려, 그러한 시나리오에서는 컨테이너가 추가적인 설정을 통해 완전히 부트스트랩되어야 합니다.

다음 표는 `BeanFactory`와 `ApplicationContext` 인터페이스 및 구현에서 제공하는 기능 목록입니다.

표 1. 기능 매트릭스

| 기능 | BeanFactory | ApplicationContext |
| --- | --- | --- |
| 빈 인스턴스화/와이어링 | 예 | 예 |
| 통합된 생명주기 관리 | 아니오 | 예 |
| 자동 `BeanPostProcessor` 등록 | 아니오 | 예 |
| 자동 `BeanFactoryPostProcessor` 등록 | 아니오 | 예 |
| 편리한 `MessageSource` 접근 (국제화용) | 아니오 | 예 |
| 내장 `ApplicationEvent` 발행 메커니즘 | 아니오 | 예 |

`DefaultListableBeanFactory`에 빈 후처리기를 명시적으로 등록하려면, 다음 예제와 같이 프로그래밍 방식으로 `addBeanPostProcessor`를 호출해야 합니다:

```java
// Java
DefaultListableBeanFactory factory = new DefaultListableBeanFactory();
// 팩토리를 빈 정의로 채웁니다

// 이제 필요한 BeanPostProcessor 인스턴스를 등록합니다
factory.addBeanPostProcessor(new AutowiredAnnotationBeanPostProcessor());
factory.addBeanPostProcessor(new MyBeanPostProcessor());

// 이제 팩토리를 사용하기 시작합니다
```

```kotlin
// Kotlin
val factory = DefaultListableBeanFactory()
// 팩토리를 빈 정의로 채웁니다

// 이제 필요한 BeanPostProcessor 인스턴스를 등록합니다
factory.addBeanPostProcessor(AutowiredAnnotationBeanPostProcessor())
factory.addBeanPostProcessor(MyBeanPostProcessor()) // Assuming MyBeanPostProcessor exists

// 이제 팩토리를 사용하기 시작합니다
```

일반 `DefaultListableBeanFactory`에 `BeanFactoryPostProcessor`를 적용하려면, 다음 예제와 같이 해당 `postProcessBeanFactory` 메소드를 호출해야 합니다:

```java
// Java
DefaultListableBeanFactory factory = new DefaultListableBeanFactory();
XmlBeanDefinitionReader reader = new XmlBeanDefinitionReader(factory);
reader.loadBeanDefinitions(new FileSystemResource("beans.xml"));

// Properties 파일에서 일부 속성 값을 가져옵니다
PropertySourcesPlaceholderConfigurer cfg = new PropertySourcesPlaceholderConfigurer();
cfg.setLocation(new FileSystemResource("jdbc.properties"));

// 이제 실제로 대체를 수행합니다
cfg.postProcessBeanFactory(factory);
```

```kotlin
// Kotlin
val factory = DefaultListableBeanFactory()
val reader = XmlBeanDefinitionReader(factory)
reader.loadBeanDefinitions(FileSystemResource("beans.xml"))

// Properties 파일에서 일부 속성 값을 가져옵니다
val cfg = PropertySourcesPlaceholderConfigurer()
cfg.setLocation(FileSystemResource("jdbc.properties"))

// 이제 실제로 대체를 수행합니다
cfg.postProcessBeanFactory(factory)
```

두 경우 모두 명시적인 등록 단계는 불편하므로, 스프링 기반 애플리케이션에서는 일반적인 엔터프라이즈 설정에서 확장된 컨테이너 기능을 위해 `BeanFactoryPostProcessor` 및 `BeanPostProcessor` 인스턴스에 의존할 때

특히, 일반 `DefaultListableBeanFactory`보다 다양한 `ApplicationContext` 변형이 선호됩니다.

`AnnotationConfigApplicationContext`는 모든 일반적인 어노테이션 후처리기가 등록되어 있으며, `@EnableTransactionManagement`와 같은 구성 어노테이션을 통해 내부적으로 추가적인 프로세서를 가져올 수 있습니다.

스프링의 어노테이션 기반 구성 모델의 추상화 수준에서는, 빈 후처리기의 개념은 단순한 내부 컨테이너 세부 사항이 됩니다.

---

**전체 주제: BeanFactory API**

이 부분은 스프링 IoC 컨테이너의 **핵심이자 기반**이 되는 `BeanFactory` 인터페이스와 그 관련 기능들을 설명합니다. `ApplicationContext`는 이 `BeanFactory`를 **확장**하여 더 많은 기능을 제공하는 것입니다.

**핵심 아이디어:** `BeanFactory`는 빈 생성과 관리의 핵심 기능만 제공하는 기본 엔진이고, `ApplicationContext`는 여기에 다양한 애플리케이션 편의 기능을 추가한 더 강력하고 사용하기 쉬운 엔진이다.

---

**첫 번째 파트: `BeanFactory` API 소개**

- **`BeanFactory` 인터페이스:** 스프링 IoC 기능의 **가장 근본적인 계약(Contract)** 입니다. 빈 객체를 **생성하고, 설정하고, 의존성을 연결하고, 관리**하는 핵심적인 기능을 정의합니다.
- **주요 역할:**
  - 빈 정의(설정 정보)를 로드하고 관리합니다.
  - 요청 시 빈 인스턴스를 생성하고 반환합니다 (`getBean()` 메소드).
  - 빈의 생명주기를 기본적인 수준에서 관리합니다.
- **`DefaultListableBeanFactory`:** `BeanFactory` 인터페이스의 **주요 구현 클래스** 중 하나입니다. 실제로 빈들을 목록으로 관리하고 처리하는 핵심 로직을 담고 있습니다. 우리가 사용하는 `ApplicationContext` (예: `GenericApplicationContext`)도 **내부적으로 이 `DefaultListableBeanFactory`를 위임(delegate)하여 핵심 기능을 수행**합니다.
- **통합 지점:** `BeanFactory`와 관련된 인터페이스들 (`BeanFactoryAware`, `InitializingBean`, `DisposableBean` 등)은 스프링 내부의 다른 컴포넌트나 서드파티 프레임워크와의 **통합(integration)** 에 중요한 역할을 합니다. 이들은 어노테이션이나 리플렉션 없이도 컨테이너와 컴포넌트 간의 효율적인 상호작용을 가능하게 합니다.
- **애플리케이션 개발자의 관점:** 일반적인 애플리케이션 개발에서는 `BeanFactory`를 직접 사용하는 것보다 **어노테이션(`@Autowired` 등)이나 Java Config(`@Bean`)를 통한 선언적인 방식**을 사용하는 것이 더 편리하고 권장됩니다. `BeanFactory` API는 프레임워크 개발이나 아주 특별한 커스터마이징 시에 주로 사용됩니다.
- **유연성과 확장성:** `BeanFactory` 자체는 특정 설정 형식(XML, Java Config 등)이나 컴포넌트 어노테이션(@Component 등)에 대해 **알지 못합니다(agnostic)**. 이러한 기능들은 별도의 **확장 모듈**(`XmlBeanDefinitionReader`, `AnnotationConfigUtils` 등)을 통해 제공되며, 이들은 모두 **`BeanDefinition`** 이라는 표준화된 메타데이터 객체를 사용하여 작동합니다. 이것이 스프링 컨테이너가 매우 유연하고 확장 가능한 이유입니다.

---

**두 번째 파트: `BeanFactory` 또는 `ApplicationContext`? (차이점 및 권장 사항)**

이 부분은 언제 `BeanFactory`를 사용하고 언제 `ApplicationContext`를 사용해야 하는지, 그리고 왜 대부분의 경우 `ApplicationContext`가 권장되는지를 설명합니다.

- **권장 사항:** **특별한 이유가 없다면 항상 `ApplicationContext`를 사용해야 합니다.**
  - **일반적인 구현체:** `GenericApplicationContext`와 그 하위 클래스인 `AnnotationConfigApplicationContext`가 커스텀 부트스트래핑(컨테이너 시작 과정 제어) 및 일반적인 용도로 가장 많이 사용됩니다.
- **`ApplicationContext`의 장점 (`BeanFactory` 대비):** `ApplicationContext`는 `BeanFactory`의 모든 기능을 포함하면서 다음과 같은 **추가적인 기능들을 자동으로 처리하고 제공**합니다.
  - **통합된 생명주기 관리:** 컨테이너 시작/종료와 빈 생명주기를 더 체계적으로 관리합니다 (`Lifecycle` 인터페이스 등).
  - **자동 `BeanPostProcessor` 등록 및 실행:** 설정에 등록된 `BeanPostProcessor` 구현체들을 **자동으로 감지하고 활성화**하여 어노테이션 처리(@Autowired, @PostConstruct 등), AOP 프록시 생성 등의 후처리 작업을 수행합니다.
  - **자동 `BeanFactoryPostProcessor` 등록 및 실행:** 설정에 등록된 `BeanFactoryPostProcessor` 구현체들을 **자동으로 감지하고 실행**하여 빈 정의 메타데이터를 수정합니다 (예: 프로퍼티 플레이스홀더 처리).
  - **국제화 지원 (`MessageSource`):** `MessageSource` 기능을 내장하여 다국어 메시지 처리를 편리하게 제공합니다.
  - **이벤트 발행 메커니즘 (`ApplicationEventPublisher`):** 빈들 간의 이벤트 기반 통신 기능을 제공합니다.
  - **리소스 로딩:** `ResourceLoader` 기능을 통합하여 외부 리소스 접근을 용이하게 합니다.
  - **관례에 의한 빈 감지:** 특정 이름(예: "messageSource")이나 타입(예: `BeanPostProcessor`)을 가진 빈들을 자동으로 감지하고 특별하게 처리합니다.
- **`BeanFactory`의 한계:**
  - `DefaultListableBeanFactory`와 같은 순수 `BeanFactory` 구현체는 위에서 언급한 **자동 감지 및 자동 등록 기능을 제공하지 않습니다.**
  - 어노테이션 처리(`@Autowired` 등)나 AOP 프록시 생성, 프로퍼티 플레이스홀더 처리 등을 사용하려면, 해당 기능을 수행하는 `Bean(Factory)PostProcessor`들을 **개발자가 직접 수동으로 찾아서 등록하고 실행**해주어야 합니다. (아래 예시 코드 참조)
  - 이러한 수동 설정은 매우 번거롭고 오류가 발생하기 쉬우며, 스프링의 많은 편리한 기능들을 제대로 활용하지 못하게 됩니다.

**기능 비교 표 요약:**

| 기능 | `BeanFactory` (예: `DefaultListableBeanFactory`) | `ApplicationContext` (예: `GenericApplicationContext`) |
| --- | --- | --- |
| 빈 생성/설정/와이어링 | **예** | **예** |
| **통합된 생명주기 관리** | 아니오 | **예** |
| **자동 `BeanPostProcessor` 등록** | 아니오 | **예** |
| **자동 `BeanFactoryPostProcessor` 등록** | 아니오 | **예** |
| **편리한 `MessageSource` 접근** | 아니오 | **예** |
| **내장 `ApplicationEvent` 발행** | 아니오 | **예** |

**`BeanFactory`에 후처리기 수동 등록/실행 예시 (왜 불편한가):**

- **`BeanPostProcessor` 수동 등록:**

    ```java
    DefaultListableBeanFactory factory = new DefaultListableBeanFactory();
    // ... 빈 정의 로드 ...
    
    // ★ 수동으로 후처리기 인스턴스를 만들어서 등록해야 함 ★
    factory.addBeanPostProcessor(new AutowiredAnnotationBeanPostProcessor());
    factory.addBeanPostProcessor(new MyBeanPostProcessor());
    
    // 이제 팩토리 사용 시작...
    ```

- **`BeanFactoryPostProcessor` 수동 실행:**`ApplicationContext`를 사용하면 위의 `addBeanPostProcessor`나 `postProcessBeanFactory` 호출 코드가 **전혀 필요 없습니다.** 컨텍스트가 알아서 해줍니다.

    ```java
    DefaultListableBeanFactory factory = new DefaultListableBeanFactory();
    // ... 빈 정의 로드 ...
    
    PropertySourcesPlaceholderConfigurer cfg = new PropertySourcesPlaceholderConfigurer();
    cfg.setLocation(...); // 프로퍼티 파일 위치 설정
    
    // ★ 수동으로 후처리기 메소드를 호출하여 빈 정의를 수정해야 함 ★
    cfg.postProcessBeanFactory(factory);
    
    // 이제 팩토리 사용 시작...
    ```


**결론:**

`BeanFactory`는 스프링 IoC 컨테이너의 **핵심 엔진**이지만, 로우 레벨(low-level) API에 가깝습니다. 반면 `ApplicationContext`는 `BeanFactory`를 기반으로 **다양한 엔터프라이즈급 기능(자동 후처리기 등록, 국제화, 이벤트 등)을 추가하고 통합**하여 개발자가 **더 편리하고 선언적인 방식**으로 스프링을 사용할 수 있도록 만든 **고수준 컨테이너**입니다. 따라서 빈 생성/관리 외에 스프링의 풍부한 기능을 활용하는 대부분의 애플리케이션에서는 **`ApplicationContext`를 사용하는 것이 훨씬 효율적이고 권장**됩니다. `BeanFactory`를 직접 사용하는 경우는 매우 드물며, 주로 프레임워크 통합이나 아주 특수한 커스터마이징 시나리오에 해당합니다.
