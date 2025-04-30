---
title: Spring Framework IoC Container
description: 
author: laze
date: 2025-04-30 00:00:02 +0900
categories: [Dev, SpringBoot]
tags: [SpringBoot]
---
# Container Overview

**컨테이너 개요**

`org.springframework.context.ApplicationContext` 인터페이스는 스프링 IoC 컨테이너를 나타내며, 빈(bean)을 인스턴스화하고, 설정하고, 조립하는 책임을 가집니다.

컨테이너는 설정 메타데이터를 읽어 인스턴스화하고, 설정하고, 조립할 컴포넌트에 대한 지시를 받습니다.

설정 메타데이터는 어노테이션이 달린 컴포넌트 클래스, 팩토리 메소드가 있는 설정 클래스, 또는 외부 XML 파일이나 Groovy 스크립트로 표현될 수 있습니다.

어떤 형식이든, 애플리케이션과 해당 컴포넌트들 간의 풍부한 상호 의존성을 구성할 수 있습니다.

`ApplicationContext` 인터페이스의 여러 구현체는 코어 스프링의 일부입니다.

독립 실행형 애플리케이션에서는 `AnnotationConfigApplicationContext` 또는 `ClassPathXmlApplicationContext`의 인스턴스를 생성하는 것이 일반적입니다.

대부분의 애플리케이션 시나리오에서는 스프링 IoC 컨테이너의 인스턴스를 하나 이상 인스턴스화하기 위해 명시적인 사용자 코드가 필요하지 않습니다.

예를 들어, 일반적인 웹 애플리케이션 시나리오에서는 애플리케이션의 `web.xml` 파일에 있는 간단한 상용구(boilerplate) 웹 기술자(descriptor) XML만으로 충분합니다 (웹 애플리케이션을 위한 편리한 ApplicationContext 인스턴스화 참조).

스프링 부트 시나리오에서는 일반적인 설정 관례에 따라 애플리케이션 컨텍스트가 암묵적으로 bootstrapped됩니다.

다음 다이어그램은 스프링이 어떻게 작동하는지에 대한 높은 수준의 뷰를 보여줍니다.

애플리케이션 클래스들은 설정 메타데이터와 결합되어, `ApplicationContext`가 생성되고 초기화된 후에는 완전히 설정되고 실행 가능한 시스템 또는 애플리케이션을 가지게 됩니다.

![container-overview](/assets/img/container-overview.png)

**설정 메타데이터**

앞의 다이어그램에서 보여주듯이, 스프링 IoC 컨테이너는 특정 형태의 설정 메타데이터를 소비(consumes)합니다.

이 설정 메타데이터는 애플리케이션 개발자가 스프링 컨테이너에게 애플리케이션의 컴포넌트를 어떻게 인스턴스화하고, 설정하고, 조립할지 알려주는 방식을 나타냅니다.

스프링 IoC 컨테이너 자체는 이 설정 메타데이터가 실제로 작성되는 형식과 완전히 분리(decoupled)되어 있습니다.

요즘 많은 개발자들은 스프링 애플리케이션을 위해 자바 기반 설정을 선택합니다:

- **어노테이션 기반 설정:** 애플리케이션의 컴포넌트 클래스에 있는 어노테이션 기반 설정 메타데이터를 사용하여 빈을 정의합니다.
- **자바 기반 설정:** 자바 기반 설정 클래스를 사용하여 애플리케이션 클래스 외부에 빈을 정의합니다. (`@Configuration`, `@Bean`, `@Import`, `@DependsOn`)

스프링 설정은 컨테이너가 관리해야 하는 최소 하나, 일반적으로는 하나 이상의 빈 정의로 구성됩니다.

자바 설정은 일반적으로 `@Configuration` 클래스 내의 `@Bean` 어노테이션이 달린 메소드를 사용하며, 각 메소드는 하나의 빈 정의에 해당합니다.

이 빈 정의들은 애플리케이션을 구성하는 실제 객체에 해당합니다.

일반적으로 서비스 계층 객체, 리포지토리나 데이터 접근 객체(DAO) 같은 영속성 계층 객체, 웹 컨트롤러 같은 프레젠테이션 객체, JPA `EntityManagerFactory`나 JMS 큐 같은 인프라 객체 등을 정의합니다.

일반적으로 세밀한(fine-grained) 도메인 객체는 컨테이너에서 설정하지 않는데, 도메인 객체를 생성하고 로드하는 것은 보통 리포지토리와 비즈니스 로직의 책임이기 때문입니다.

**외부 설정 DSL로서의 XML**

XML 기반 설정 메타데이터는 이러한 빈들을 최상위 `<beans/>` 요소 내부의 `<bean/>` 요소로 설정합니다. 다음 예제는 XML 기반 설정 메타데이터의 기본 구조를 보여줍니다:

```xml
<?xml version="1.0" encoding="UTF-8"?>
<beans xmlns="<http://www.springframework.org/schema/beans>"
	xmlns:xsi="<http://www.w3.org/2001/XMLSchema-instance>"
	xsi:schemaLocation="<http://www.springframework.org/schema/beans>
		<https://www.springframework.org/schema/beans/spring-beans.xsd>">

	<bean id="..." class="...">
		<!-- 이 빈을 위한 collaborators 및 configuration 이 여기에 들어갑니다 -->
	</bean>

	<bean id="..." class="...">
		<!-- 이 빈을 위한 collaborators 및 configuration 이 여기에 들어갑니다 -->
	</bean>

</beans>

```

- `id` 속성은 개별 빈 정의를 식별하는 문자열입니다.
- `class` 속성은 빈의 타입을 정의하며 정규화된(fully qualified) 클래스 이름을 사용합니다.

`id` 속성의 값은 협력하는 객체를 참조하는 데 사용될 수 있습니다. 협력 객체를 참조하는 XML은 이 예제에는 표시되지 않았습니다. 자세한 내용은 의존성(Dependencies)을 참조하십시오.

컨테이너를 인스턴스화하기 위해, XML 리소스 파일의 위치 경로(들)는 `ClassPathXmlApplicationContext` 생성자에 제공되어야 하며, 이는 컨테이너가 로컬 파일 시스템, 자바 CLASSPATH 등 다양한 외부 리소스로부터 설정 메타데이터를 로드하게 합니다.

```java
// Java
ApplicationContext context = new ClassPathXmlApplicationContext("services.xml", "daos.xml");

```

```kotlin
// Kotlin
val context = ClassPathXmlApplicationContext("services.xml", "daos.xml")

```

스프링의 IoC 컨테이너에 대해 배운 후에는, URI 구문으로 정의된 위치에서 `InputStream`을 읽는 편리한 메커니즘을 제공하는 스프링의 리소스 추상화(Resource abstraction, 리소스(Resources)에서 설명됨)에 대해 더 알고 싶을 수 있습니다.

특히, 리소스 경로는 애플리케이션 컨텍스트와 리소스 경로(Application Contexts and Resource Paths)에서 설명된 대로 애플리케이션 컨텍스트를 구성하는 데 사용됩니다.

다음 예제는 서비스 계층 객체 (`services.xml`) 설정 파일을 보여줍니다:

```xml
<?xml version="1.0" encoding="UTF-8"?>
<beans xmlns="<http://www.springframework.org/schema/beans>"
	xmlns:xsi="<http://www.w3.org/2001/XMLSchema-instance>"
	xsi:schemaLocation="<http://www.springframework.org/schema/beans>
		<https://www.springframework.org/schema/beans/spring-beans.xsd>">

	<!-- services -->

	<bean id="petStore" class="org.springframework.samples.jpetstore.services.PetStoreServiceImpl">
		<property name="accountDao" ref="accountDao"/>
		<property name="itemDao" ref="itemDao"/>
	</bean>

	<!-- 서비스에 대한 더 많은 빈 정의가 여기에 들어갑니다 -->

</beans>

```

다음 예제는 데이터 접근 객체 (`daos.xml`) 파일을 보여줍니다:

```xml
<?xml version="1.0" encoding="UTF-8"?>
<beans xmlns="<http://www.springframework.org/schema/beans>"
	xmlns:xsi="<http://www.w3.org/2001/XMLSchema-instance>"
	xsi:schemaLocation="<http://www.springframework.org/schema/beans>
		<https://www.springframework.org/schema/beans/spring-beans.xsd>">

	<bean id="accountDao"
		class="org.springframework.samples.jpetstore.dao.jpa.JpaAccountDao">
	</bean>

	<bean id="itemDao" class="org.springframework.samples.jpetstore.dao.jpa.JpaItemDao">
	</bean>

</beans>

```

앞의 예제에서, 서비스 계층은 `PetStoreServiceImpl` 클래스와 (JPA 객체-관계 매핑 표준 기반의) `JpaAccountDao` 및 `JpaItemDao` 타입의 두 데이터 접근 객체로 구성됩니다.

`property name` 요소는 JavaBean 속성의 이름을 참조하고, `ref` 요소는 다른 빈 정의의 이름을 참조합니다. `id`와 `ref` 요소 간의 이 연결은 협력 객체 간의 의존성을 표현합니다.

**XML 기반 설정 메타데이터 구성하기**

빈 정의를 여러 XML 파일에 걸쳐 나누는 것이 유용할 수 있습니다. 종종 각 개별 XML 설정 파일은 아키텍처의 논리적 계층이나 모듈을 나타냅니다.

`ClassPathXmlApplicationContext` 생성자를 사용하여 XML 조각들로부터 빈 정의를 로드할 수 있습니다.

이 생성자는 이전 섹션에서 보여준 것처럼 여러 `Resource` 위치를 받습니다. 또는, `<import/>` 요소를 하나 이상 사용하여 다른 파일(들)로부터 빈 정의를 로드할 수 있습니다.

다음 예제는 이를 수행하는 방법을 보여줍니다:

```xml
<beans>
	<import resource="services.xml"/>
	<import resource="resources/messageSource.xml"/>
	<import resource="/resources/themeSource.xml"/>

	<bean id="bean1" class="..."/>
	<bean id="bean2" class="..."/>
</beans>

```

앞의 예제에서, 외부 빈 정의는 `services.xml`, `messageSource.xml`, `themeSource.xml` 세 파일에서 로드됩니다.

모든 위치 경로는 임포트하는 정의 파일에 상대적이므로, `services.xml`은 임포트하는 파일과 동일한 디렉토리 또는 클래스패스 위치에 있어야 하며,

`messageSource.xml`과 `themeSource.xml`은 임포트하는 파일 위치 아래의 `resources` 위치에 있어야 합니다. 보시다시피, 선행 슬래시(`/`)는 무시됩니다.

그러나 이러한 경로가 상대적이므로 슬래시를 전혀 사용하지 않는 것이 더 좋은 형식입니다. 최상위 `<beans/>` 요소를 포함하여 임포트되는 파일의 내용은 스프링 스키마(Spring Schema)에 따라 유효한 XML 빈 정의여야 합니다.

상대 경로 `../`를 사용하여 상위 디렉토리의 파일을 참조하는 것은 가능하지만 권장되지 않습니다.

그렇게 하면 현재 애플리케이션 외부의 파일에 대한 의존성이 생성됩니다.

특히, 이 참조는 `classpath:` URL(예: `classpath:../services.xml`)에 권장되지 않는데, 런타임 해석 프로세스가 "가장 가까운" 클래스패스 루트를 선택한 다음 해당 부모 디렉토리를 보기 때문입니다.

클래스패스 구성 변경으로 인해 다른, 잘못된 디렉토리가 선택될 수 있습니다.

상대 경로 대신 항상 정규화된 리소스 위치를 사용할 수 있습니다: 예를 들어, `file:C:/config/services.xml` 또는 `classpath:/config/services.xml`.

그러나 애플리케이션의 설정을 특정 절대 위치에 결합(coupling)시킨다는 점에 유의하십시오.

일반적으로 이러한 절대 위치에 대한 간접 참조(indirection)를 유지하는 것이 바람직합니다 - 예를 들어, 런타임 시 JVM 시스템 속성에 대해 해석되는 `${...}` 플레이스홀더를 통해.

네임스페이스 자체는 임포트 지시문 기능을 제공합니다. 단순한 빈 정의를 넘어서는 추가 설정 기능은 스프링에서 제공하는 선택된 XML 네임스페이스(예: `context` 및 `util` 네임스페이스)에서 사용할 수 있습니다.

**Groovy 빈 정의 DSL**

외부화된 설정 메타데이터의 추가 예로서, 빈 정의는 Grails 프레임워크에서 알려진 스프링의 Groovy 빈 정의 DSL로도 표현될 수 있습니다. 일반적으로 이러한 설정은 다음 예제와 같은 구조를 가진 ".groovy" 파일에 존재합니다:

```groovy
beans {
	dataSource(BasicDataSource) {
		driverClassName = "org.hsqldb.jdbcDriver"
		url = "jdbc:hsqldb:mem:grailsDB"
		username = "sa"
		password = ""
		settings = [mynew:"setting"]
	}
	sessionFactory(SessionFactory) {
		dataSource = dataSource // dataSource 빈 참조
	}
	myService(MyService) {
		nestedBean = { AnotherBean bean -> // 중첩 빈 정의
			dataSource = dataSource
		}
	}
}

```

이 설정 스타일은 XML 빈 정의와 거의 동일하며 스프링의 XML 설정 네임스페이스도 지원합니다.

또한 `importBeans` 지시문을 통해 XML 빈 정의 파일을 임포트할 수도 있습니다.

**컨테이너 사용하기**

`ApplicationContext`는 다른 빈들과 그 의존성들의 레지스트리를 유지할 수 있는 고급 팩토리 인터페이스입니다. `T getBean(String name, Class<T> requiredType)` 메소드를 사용하여 빈의 인스턴스를 검색할 수 있습니다.

`ApplicationContext`를 사용하면 빈 정의를 읽고 접근할 수 있습니다. 다음 예제를 참조하십시오:

```java
// Java
// 빈 생성 및 설정
ApplicationContext context = new ClassPathXmlApplicationContext("services.xml", "daos.xml");

// 설정된 인스턴스 검색
PetStoreService service = context.getBean("petStore", PetStoreService.class);

// 설정된 인스턴스 사용
List<String> userList = service.getUsernameList();

```

```kotlin
// Kotlin
// 빈 생성 및 설정
val context = ClassPathXmlApplicationContext("services.xml", "daos.xml")

// 설정된 인스턴스 검색
val service = context.getBean("petStore", PetStoreService::class.java)

// 설정된 인스턴스 사용
val userList = service.usernameList

```

Groovy 설정의 경우, 부트스트래핑은 매우 유사합니다. Groovy를 인식하는 (하지만 XML 빈 정의도 이해하는) 다른 컨텍스트 구현 클래스를 사용합니다. 다음 예제는 Groovy 설정을 보여줍니다:

```java
// Java
ApplicationContext context = new GenericGroovyApplicationContext("services.groovy", "daos.groovy");

```

```kotlin
// Kotlin
val context = GenericGroovyApplicationContext("services.groovy", "daos.groovy")

```

가장 유연한 변형은 리더 델리게이트(reader delegates)와 결합된 `GenericApplicationContext`입니다 - 예를 들어, XML 파일의 경우 `XmlBeanDefinitionReader`를 사용합니다. 다음 예제를 참조하십시오:

```java
// Java
GenericApplicationContext context = new GenericApplicationContext();
new XmlBeanDefinitionReader(context).loadBeanDefinitions("services.xml", "daos.xml");
context.refresh();

```

```kotlin
// Kotlin
val context = GenericApplicationContext()
XmlBeanDefinitionReader(context).loadBeanDefinitions("services.xml", "daos.xml")
context.refresh()

```

Groovy 파일의 경우 `GroovyBeanDefinitionReader`를 사용할 수도 있습니다. 다음 예제를 참조하십시오:

```java
// Java
GenericApplicationContext context = new GenericApplicationContext();
new GroovyBeanDefinitionReader(context).loadBeanDefinitions("services.groovy", "daos.groovy");
context.refresh();

```

```kotlin
// Kotlin
val context = GenericApplicationContext()
GroovyBeanDefinitionReader(context).loadBeanDefinitions("services.groovy", "daos.groovy")
context.refresh()

```

동일한 `ApplicationContext`에서 이러한 리더 델리게이트를 혼합하여 다양한 설정 소스로부터 빈 정의를 읽을 수 있습니다.

그런 다음 `getBean`을 사용하여 빈의 인스턴스를 검색할 수 있습니다. `ApplicationContext` 인터페이스에는 빈을 검색하는 다른 몇 가지 메소드가 있지만, 이상적으로는 애플리케이션 코드가 이를 사용해서는 안 됩니다. 실제로, 애플리케이션 코드는 `getBean()` 메소드를 전혀 호출하지 않아야 하며 따라서 스프링 API에 대한 의존성이 전혀 없어야 합니다. 예를 들어, 스프링의 웹 프레임워크 통합은 컨트롤러 및 JSF 관리 빈과 같은 다양한 웹 프레임워크 컴포넌트에 대한 의존성 주입을 제공하여, 메타데이터(예: 오토와이어링 어노테이션)를 통해 특정 빈에 대한 의존성을 선언할 수 있게 합니다.

요약

1. **핵심 역할:** 스프링 프레임워크의 중심에는 **IoC 컨테이너** (ApplicationContext 인터페이스가 대표적)가 있습니다. 이 컨테이너는 애플리케이션의 객체( **빈(Beans)** )를 **생성, 설정, 조립하고 생명주기를 관리**합니다.
2. **작동 방식:** 컨테이너는 **설정 메타데이터**를 읽어 어떤 빈을 만들고 어떻게 연결(의존성 주입)할지 지시받습니다.
3. **설정 메타데이터 종류:**
  - **어노테이션 기반:** @Component, @Service 등 클래스에 직접 어노테이션 사용.
  - **자바 기반:** @Configuration 클래스와 @Bean 메소드 사용.
  - **XML 기반:** <beans>, <bean> 태그 사용 (과거 방식, 여전히 지원됨).
4. **빈(Beans)이란?** 컨테이너가 생성하고 관리하는 애플리케이션의 핵심 객체들입니다 (서비스, DAO, 컨트롤러 등).
5. **ApplicationContext vs BeanFactory:** ApplicationContext가 더 많은 기능(AOP 통합, 메시지 처리, 이벤트 발행 등)을 제공하며 일반적으로 권장됩니다.
6. **컨테이너 생성:** 보통 웹 애플리케이션(web.xml 설정)이나 스프링 부트 환경에서는 컨테이너가 자동으로 생성 및 관리되므로 개발자가 직접 생성할 필요는 적습니다. (독립 실행형 앱에서는 ClassPathXmlApplicationContext 나 AnnotationConfigApplicationContext 등을 직접 생성 가능)
7. **빈 사용:** 주로 **의존성 주입(DI)**을 통해 필요한 빈을 자동으로 주입받아 사용합니다. context.getBean() 메소드로 직접 빈을 가져올 수도 있지만, 애플리케이션 코드에서 직접 사용하는 것은 권장되지 않습니다 (스프링 API 의존성 발생).
8. **XML 설정 조합:** 여러 XML 파일을 <import> 태그를 사용하여 논리적으로 분리하고 조합할 수 있습니다.
