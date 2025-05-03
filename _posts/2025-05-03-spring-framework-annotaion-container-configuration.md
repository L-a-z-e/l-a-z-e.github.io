---
title: Spring Framework Annotation-based Container Configuration
description: 
author: laze
date: 2025-05-03 00:00:01 +0900
categories: [Dev, SpringBoot]
tags: [SpringBoot]
---
**Annotation-based Container Configuration**

스프링은 관련 클래스, 메소드 또는 필드 선언에 어노테이션을 사용하여 컴포넌트 클래스 자체의 메타데이터에 따라 작동하는 어노테이션 기반 설정에 대한 포괄적인 지원을 제공합니다.

스프링은 핵심 IOC 컨테이너가 특정 어노테이션을 인식하도록 하기 위해 어노테이션과 함께 `BeanPostProcessor`를 사용합니다.

예를 들어, `@Autowired` 어노테이션은 협력자 자동 와이어링(Autowiring Collaborators)에서 설명된 것과 동일한 기능을 제공하지만, 더 세분화된 제어와 더 넓은 적용 가능성을 가집니다.

또한, 스프링은 `@PostConstruct` 및 `@PreDestroy`와 같은 어노테이션뿐만 아니라, `@Inject` 및 `@Named`와 같이 `jakarta.inject` 패키지에 포함된 어노테이션도 지원합니다.

*어노테이션 주입은 외부 속성 주입 전에 수행됩니다.*

*따라서, 외부 설정(예: XML 지정 빈 속성)은 혼합된 접근 방식을 통해 와이어링될 때 속성에 대한 어노테이션을 효과적으로 오버라이드합니다.*

*기술적으로, 후처리기들을 개별 빈 정의로 등록할 수 있지만, 그것들은 이미 `AnnotationConfigApplicationContext`에 암묵적으로 등록되어 있습니다.*

XML 기반 스프링 설정에서는, 어노테이션 기반 설정과 혼합하여 사용 가능하도록 다음 구성 태그를 포함할 수 있습니다:

```xml
<?xml version="1.0" encoding="UTF-8"?>
<beans xmlns="<http://www.springframework.org/schema/beans>"
	xmlns:xsi="<http://www.w3.org/2001/XMLSchema-instance>"
	xmlns:context="<http://www.springframework.org/schema/context>"
	xsi:schemaLocation="<http://www.springframework.org/schema/beans>
		<https://www.springframework.org/schema/beans/spring-beans.xsd>
		<http://www.springframework.org/schema/context>
		<https://www.springframework.org/schema/context/spring-context.xsd>">

	<context:annotation-config/>

</beans>
```

`<context:annotation-config/>` 요소는 다음 후처리기들을 암묵적으로 등록합니다:

- `ConfigurationClassPostProcessor`
- `AutowiredAnnotationBeanPostProcessor`
- `CommonAnnotationBeanPostProcessor`
- `PersistenceAnnotationBeanPostProcessor`
- `EventListenerMethodProcessor`

`*<context:annotation-config/>`는 그것이 정의된 동일한 애플리케이션 컨텍스트 내의 빈에서만 어노테이션을 찾습니다.*

*이는 즉, 만약 `<context:annotation-config/>`를 `DispatcherServlet`용 `WebApplicationContext`에 넣는다면, 그것은 여러분의 컨트롤러 내의 `@Autowired` 빈만 확인하고, 서비스 내의 빈은 확인하지 않는다는 것을 의미합니다.*

---

**첫 번째 파트: 기본 개념 및 동작 원리**

- **어노테이션이란?** `@Override`, `@Deprecated`처럼 코드에 붙이는 특별한 주석 같은 것입니다. 단순 주석과 달리, 컴파일러나 프레임워크가 이 어노테이션을 읽고 특별한 의미로 해석하여 추가적인 동작을 할 수 있습니다.
- **스프링의 활용:** 스프링은 `@Autowired`, `@Component`, `@Service`, `@Repository`, `@Configuration`, `@Bean`, `@PostConstruct`, `@PreDestroy` 등 다양한 어노테이션을 제공합니다.
  개발자는 이 어노테이션들을 클래스, 메소드, 필드 등에 붙여서 "이 클래스는 스프링 빈이야", "이 필드에는 다른 빈을 자동으로 주입해줘", "이 메소드는 빈 초기화 후에 호출해줘" 등의 지시를 내릴 수 있습니다.
- **동작 원리 (`BeanPostProcessor` 재등장!):** 스프링 컨테이너는 내부적으로 **어노테이션을 찾아 처리하는 특별한 `BeanPostProcessor` 구현체들**을 가지고 있습니다.
  - 예를 들어, `AutowiredAnnotationBeanPostProcessor`는 빈들을 생성하고 초기화하는 과정에서 `@Autowired` 어노테이션이 붙은 필드나 메소드를 찾아서 자동으로 의존성을 주입해주는 역할을 합니다.
  - `CommonAnnotationBeanPostProcessor`는 `@PostConstruct`, `@PreDestroy` 같은 표준 어노테이션을 처리하여 생명주기 콜백을 호출해줍니다.
- **결론:** 개발자는 어노테이션을 붙이기만 하면, 스프링에 내장된 (또는 등록된) `BeanPostProcessor`들이 알아서 그 의미를 해석하고 필요한 작업을 처리해줍니다.

---

**두 번째 파트: 어노테이션 주입 순서 및 XML과의 관계**

- **주입 순서:**
  - **어노테이션 기반 주입(`@Autowired` 등)이 먼저 실행됩니다.**
  - 그 후에 **XML 설정(`<property>` 태그 등)에 의한 외부 속성 주입이 실행됩니다.**
  - **결과:** 만약 같은 필드에 대해 `@Autowired`로 빈을 주입하고, XML의 `<property>` 태그로 또 다른 빈이나 값을 주입하려고 하면, **XML 설정이 어노테이션 설정을 덮어쓰게 됩니다.** (나중에 실행된 것이 이김) 이는 두 방식을 혼합해서 사용할 때 주의해야 할 점입니다.
- **후처리기 등록 방식:**
  - `AnnotationConfigApplicationContext` (Java Config 기반 컨텍스트)를 사용하면, 어노테이션 처리에 필요한 주요 `BeanPostProcessor`들이 **자동으로, 암묵적으로 등록**됩니다. 개발자가 신경 쓸 필요가 없습니다.
  - **XML 기반 설정에서 어노테이션 사용하기 (핵심 내용!)**: 만약 XML 기반으로 스프링 설정을 하면서 어노테이션 기능도 함께 사용하고 싶다면, **XML 설정 파일 안에 어노테이션 처리를 활성화하라는 특별한 태그를 추가**해야 합니다. 이것이 바로 `<context:annotation-config/>` 입니다.

---

**세 번째 파트: `<context:annotation-config/>` 태그 설명**

이 부분은 XML 기반 설정에서 어노테이션 기능을 어떻게 활성화하는지에 대한 내용입니다.

- **역할:** XML 설정 파일 안에 `<context:annotation-config/>` 태그를 한 줄 추가하면, 스프링 컨테이너에게 **"이 설정 파일을 사용하는 컨텍스트 내에서는 어노테이션 기반 설정을 처리할 준비를 해라!"** 라고 지시하는 것과 같습니다. 마치 어노테이션 처리 기능을 켜는 스위치 같은 역할입니다.
- **동작 방식 (암묵적 등록):** 이 태그는 단순히 스위치를 켜는 것을 넘어, 어노테이션 처리에 필요한 **핵심적인 `BeanPostProcessor`들을 스프링 컨테이너에 자동으로 등록**해줍니다. 개발자가 XML에 이 프로세서들을 일일이 `<bean>`으로 정의하지 않아도 됩니다.
- **자동 등록되는 주요 후처리기들:**
  - `ConfigurationClassPostProcessor`: `@Configuration`, `@Bean`, `@ComponentScan`, `@Import` 등 **Java Config 스타일 어노테이션**을 처리합니다.
  - `AutowiredAnnotationBeanPostProcessor`: `@Autowired`, `@Value`, `@Inject`(표준) 어노테이션을 처리하여 **의존성 주입**을 수행합니다.
  - `CommonAnnotationBeanPostProcessor`: `@PostConstruct`, `@PreDestroy`, `@Resource`(표준) 등 **JSR-250 표준 어노테이션**을 처리합니다. (생명주기 콜백, 이름 기반 주입 등)
  - `PersistenceAnnotationBeanPostProcessor`: `@PersistenceContext`, `@PersistenceUnit` 등 **JPA 관련 어노테이션**을 처리합니다. (JPA 사용 시)
  - `EventListenerMethodProcessor`: `@EventListener` 어노테이션을 처리하여 **스프링 이벤트 리스너**를 등록합니다.
- **스코프 제한:**
  - `<context:annotation-config/>` 태그는 **자신이 정의된 바로 그 `ApplicationContext` 내부에 있는 빈들**에 대해서만 어노테이션을 찾아서 처리합니다.
  - **예시:** 웹 애플리케이션에서 부모 컨텍스트(Root Context, 보통 서비스/DAO 빈 관리)와 자식 컨텍스트(Servlet Context, 보통 컨트롤러 빈 관리)를 사용하는 경우, 만약 `<context:annotation-config/>`를 **자식 컨텍스트 설정 파일(예: servlet-context.xml)에만** 넣었다면,
    - 자식 컨텍스트에 정의된 컨트롤러(`@Controller`) 내부의 `@Autowired`는 처리됩니다.
    - 하지만 **부모 컨텍스트에 정의된 서비스(`@Service`) 내부의 `@Autowired`는 처리되지 않습니다.** 왜냐하면 자식 컨텍스트의 `<context:annotation-config/>`는 부모 컨텍스트까지 들여다보지 않기 때문입니다.
  - **해결책:** 이런 구조에서는 보통 부모 컨텍스트 설정 파일(예: root-context.xml)에도 `<context:annotation-config/>`를 넣어주거나, (더 일반적으로는) `<context:component-scan>`을 사용하여 컴포넌트 스캔과 어노테이션 처리를 함께 활성화합니다. (`<context:component-scan>`은 내부적으로 `<context:annotation-config/>`의 기능을 포함합니다.)

**요약:**

어노테이션 기반 설정은 자바 코드 내에 `@Autowired`, `@Component` 등의 어노테이션을 사용하여 빈 설정 및 의존성 주입을 하는 방식입니다. 스프링은 내부적으로 `BeanPostProcessor`를 사용하여 이 어노테이션들을 처리합니다. XML 기반 설정에서 어노테이션 기능을 사용하려면 `<context:annotation-config/>` 태그를 추가해야 하며, 이 태그는 필요한 `BeanPostProcessor`들을 자동으로 등록해줍니다. 단, 이 태그는 자신이 정의된 컨텍스트 내의 빈에만 영향을 미친다는 점에 유의해야 합니다.
