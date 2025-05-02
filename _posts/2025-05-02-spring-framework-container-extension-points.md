---
title: Spring Framework Container Extension Points
description: 
author: laze
date: 2025-05-02 00:00:04 +0900
categories: [Dev, SpringBoot]
tags: [SpringBoot]
---
**Container Extension Points**

일반적으로 애플리케이션 개발자는 `ApplicationContext` 구현 클래스를 하위 클래스로 만들 필요가 없습니다.

대신, 특별한 통합 인터페이스의 구현을 연결(plugging in)하여 스프링 IoC 컨테이너를 확장할 수 있습니다.

**`BeanPostProcessor`를 사용하여 빈 커스터마이징 (Customizing Beans by Using a `BeanPostProcessor`)**

`BeanPostProcessor` 인터페이스는 자신만의 (또는 컨테이너의 기본) 인스턴스화 로직, 의존성 해석 로직 등을 제공하기 위해 구현할 수 있는 콜백 메소드를 정의합니다.

스프링 컨테이너가 빈의 인스턴스화, 설정 및 초기화를 완료한 후 일부 커스텀 로직을 구현하려면 하나 이상의 커스텀 `BeanPostProcessor` 구현을 연결할 수 있습니다.

여러 `BeanPostProcessor` 인스턴스를 구성할 수 있으며, `order` 속성을 설정하여 이러한 `BeanPostProcessor` 인스턴스가 실행되는 순서를 제어할 수 있습니다.

이 속성은 `BeanPostProcessor`가 `Ordered` 인터페이스를 구현하는 경우에만 설정할 수 있습니다. 자신만의 `BeanPostProcessor`를 작성하는 경우, `Ordered` 인터페이스도 구현하는 것을 고려해야 합니다.

자세한 내용은 `BeanPostProcessor` 및 `Ordered` 인터페이스의 javadoc을 참조하십시오.

`*BeanPostProcessor` 인스턴스는 빈 (또는 객체) 인스턴스에서 작동합니다.*

*즉, 스프링 IoC 컨테이너가 빈 인스턴스를 인스턴스화한 다음 `BeanPostProcessor` 인스턴스가 작업을 수행합니다.*

`*BeanPostProcessor` 인스턴스는 컨테이너별로 스코프 지정됩니다.*

*이는 컨테이너 계층 구조를 사용하는 경우에만 관련이 있습니다. 한 컨테이너에서 `BeanPostProcessor`를 정의하면 해당 컨테이너의 빈만 후처리(post-processes)합니다.*

*즉, 한 컨테이너에 정의된 빈은 다른 컨테이너에 정의된 `BeanPostProcessor`에 의해 후처리되지 않습니다.*

*두 컨테이너가 동일한 계층 구조의 일부인 경우에도 마찬가지입니다.실제 빈 정의(즉, 빈을 정의하는 청사진)를 변경하려면, 대신 `BeanFactoryPostProcessor`를 사용해야 합니다.*

*이는 `BeanFactoryPostProcessor`로 구성 메타데이터 커스터마이징(Customizing Configuration Metadata with a `BeanFactoryPostProcessor`)에서 설명됩니다.*

`org.springframework.beans.factory.config.BeanPostProcessor` 인터페이스는 정확히 두 개의 콜백 메소드로 구성됩니다.

이러한 클래스가 컨테이너에 후처리기(post-processor)로 등록되면, 컨테이너가 생성하는 각 빈 인스턴스에 대해 후처리기는 컨테이너 초기화 메소드(예: `InitializingBean.afterPropertiesSet()` 또는 선언된 모든 init 메소드)가 호출되기 전과 모든 빈 초기화 콜백 이후 모두 컨테이너로부터 콜백을 받습니다.

후처리기는 콜백을 완전히 무시하는 것을 포함하여 빈 인스턴스로 모든 작업을 수행할 수 있습니다.

빈 후처리기는 일반적으로 콜백 인터페이스를 확인하거나 빈을 프록시로 래핑(wrap)할 수 있습니다.

일부 스프링 AOP 인프라 클래스는 프록시 래핑 로직을 제공하기 위해 빈 후처리기(bean post-processors)로 구현됩니다.

`ApplicationContext`는 `BeanPostProcessor` 인터페이스를 구현하는 구성 메타데이터에 정의된 모든 빈을 자동으로 감지합니다.

`ApplicationContext`는 이러한 빈들을 후처리기(post-processors)로 등록하여 빈 생성 시 나중에 호출될 수 있도록 합니다.

빈 후처리기는 다른 빈과 동일한 방식으로 컨테이너에 배포될 수 있습니다.

*설정 클래스의 `@Bean` 팩토리 메소드를 사용하여 `BeanPostProcessor`를 선언할 때,*

*팩토리 메소드의 반환 타입은 구현 클래스 자체이거나 적어도 `org.springframework.beans.factory.config.BeanPostProcessor` 인터페이스여야 하며, 해당 빈의 후처리기 특성을 명확하게 나타내야 합니다.*

*그렇지 않으면 `ApplicationContext`는 완전히 생성하기 전에 타입별로 자동 감지할 수 없습니다.*

`*BeanPostProcessor`는 컨텍스트의 다른 빈 초기화에 적용하기 위해 조기에 인스턴스화되어야 하므로 이 조기 타입 감지는 중요합니다.*

`*BeanPostProcessor` 인스턴스 프로그래밍 방식 등록*`BeanPostProcessor` 등록에 권장되는 접근 방식은 `ApplicationContext` 자동 감지(앞서 설명됨)를 통하는 것이지만,

`ConfigurableBeanFactory`의 `addBeanPostProcessor` 메소드를 사용하여 프로그래밍 방식으로 등록할 수도 있습니다.

이는 등록 전에 조건부 로직을 평가해야 하거나 계층 구조의 컨텍스트 간에 빈 후처리기를 복사해야 할 때 유용할 수 있습니다.

그러나 프로그래밍 방식으로 추가된 `BeanPostProcessor` 인스턴스는 `Ordered` 인터페이스를 따르지 않고 등록 순서가 실행 순서를 결정합니다.

또한 프로그래밍 방식으로 등록된 `BeanPostProcessor` 인스턴스는 명시적인 순서 지정에 관계없이 자동 감지를 통해 등록된 인스턴스보다 항상 먼저 처리됩니다.

`*BeanPostProcessor` 인스턴스 및 AOP 자동 프록시 (AOP auto-proxying)*`BeanPostProcessor` 인터페이스를 구현하는 클래스는 특별하며 컨테이너에 의해 다르게 처리됩니다.

모든 `BeanPostProcessor` 인스턴스와 직접 참조하는 빈들은 `ApplicationContext`의 특별한 시작 단계의 일부로서 시작 시점에 인스턴스화됩니다.

다음으로, 모든 `BeanPostProcessor` 인스턴스는 정렬된 방식으로 등록되고 컨테이너의 모든 추가 빈에 적용됩니다.

AOP 자동 프록시는 `BeanPostProcessor` 자체로 구현되기 때문에, `BeanPostProcessor` 인스턴스나 직접 참조하는 빈 모두 자동 프록시 대상이 아니므로 애스펙트(aspects)가 위빙(woven)되지 않습니다.

*이러한 빈의 경우 정보성 로그 메시지가 표시되어야 합니다:*

*빈 `someBean`은 모든 `BeanPostProcessor` 인터페이스에 의해 처리될 자격이 없습니다 (예: 자동 프록시 대상 아님).*

*자동 와이어링이나 `@Resource`(자동 와이어링으로 대체될 수 있음)를 사용하여 `BeanPostProcessor`에 빈을 와이어링하는 경우, 스프링은 타입 일치 의존성 후보를 검색할 때 예기치 않은 빈에 접근할 수 있으며, 따라서 자동 프록시 또는 다른 종류의 빈 후처리 대상에서 제외될 수 있습니다.*

*예를 들어, 필드 또는 세터 이름이 빈의 선언된 이름과 직접 일치하지 않고 `name` 속성이 사용되지 않은 `@Resource`로 어노테이션된 의존성이 있는 경우, 스프링은 타입별로 일치시키기 위해 다른 빈에 접근합니다.*

*예제: Hello World, `BeanPostProcessor` 스타일*
이 첫 번째 예제는 기본적인 사용법을 보여줍니다. 이 예제는 컨테이너가 생성할 때 각 빈의 `toString()` 메소드를 호출하고 결과 문자열을 시스템 콘솔에 출력하는 커스텀 `BeanPostProcessor` 구현을 보여줍니다.

다음 목록은 커스텀 `BeanPostProcessor` 구현 클래스 정의를 보여줍니다:

```java
// Java
package scripting;

import org.springframework.beans.factory.config.BeanPostProcessor;

public class InstantiationTracingBeanPostProcessor implements BeanPostProcessor {

	// 단순히 인스턴스화된 빈을 그대로 반환
	@Override
	public Object postProcessBeforeInitialization(Object bean, String beanName) {
		return bean; // 여기서 잠재적으로 어떤 객체 참조든 반환할 수 있음...
	}

	@Override
	public Object postProcessAfterInitialization(Object bean, String beanName) {
		System.out.println("Bean '" + beanName + "' created : " + bean.toString());
		return bean;
	}
}
```

```kotlin
// Kotlin
package scripting

import org.springframework.beans.factory.config.BeanPostProcessor
import org.springframework.beans.BeansException

class InstantiationTracingBeanPostProcessor : BeanPostProcessor {

    // 단순히 인스턴스화된 빈을 그대로 반환
    @Throws(BeansException::class)
    override fun postProcessBeforeInitialization(bean: Any, beanName: String): Any {
        return bean // 여기서 잠재적으로 어떤 객체 참조든 반환할 수 있음...
    }

    @Throws(BeansException::class)
    override fun postProcessAfterInitialization(bean: Any, beanName: String): Any {
        println("Bean '$beanName' created : $bean")
        return bean
    }
}
```

다음 `beans` 요소는 `InstantiationTracingBeanPostProcessor`를 사용합니다:

```xml
<?xml version="1.0" encoding="UTF-8"?>
<beans xmlns="<http://www.springframework.org/schema/beans>"
	xmlns:xsi="<http://www.w3.org/2001/XMLSchema-instance>"
	xmlns:lang="<http://www.springframework.org/schema/lang>"
	xsi:schemaLocation="<http://www.springframework.org/schema/beans>
		<https://www.springframework.org/schema/beans/spring-beans.xsd>
		<http://www.springframework.org/schema/lang>
		<https://www.springframework.org/schema/lang/spring-lang.xsd>">

	<lang:groovy id="messenger"
			script-source="classpath:org/springframework/scripting/groovy/Messenger.groovy">
		<lang:property name="message" value="Fiona Apple Is Just So Dreamy."/>
	</lang:groovy>

	<!--
	위의 빈 (messenger)이 인스턴스화될 때, 이 커스텀
	BeanPostProcessor 구현은 그 사실을 시스템 콘솔에 출력합니다
	-->
	<bean class="scripting.InstantiationTracingBeanPostProcessor"/>

</beans>

```

`InstantiationTracingBeanPostProcessor`가 단순히 정의되기만 했고, 이름조차 없으며, 빈이기 때문에 다른 빈처럼 의존성 주입될 수 있습니다.

다음 자바 애플리케이션은 앞의 코드와 구성을 실행합니다:

```java
// Java
import org.springframework.context.ApplicationContext;
import org.springframework.context.support.ClassPathXmlApplicationContext;
import org.springframework.scripting.Messenger;

public final class Boot {

	public static void main(final String[] args) throws Exception {
		ApplicationContext ctx = new ClassPathXmlApplicationContext("scripting/beans.xml");
		Messenger messenger = ctx.getBean("messenger", Messenger.class);
		System.out.println(messenger);
	}

}
```

```kotlin
// Kotlin
import org.springframework.context.ApplicationContext
import org.springframework.context.support.ClassPathXmlApplicationContext
import org.springframework.scripting.Messenger

fun main(args: Array<String>) {
    val ctx: ApplicationContext = ClassPathXmlApplicationContext("scripting/beans.xml")
    val messenger = ctx.getBean("messenger", Messenger::class.java)
    println(messenger)
}
```

앞의 애플리케이션의 출력은 다음과 유사합니다:

```
Bean 'messenger' created : org.springframework.scripting.groovy.GroovyMessenger@272961
org.springframework.scripting.groovy.GroovyMessenger@272961
```

*예제: `AutowiredAnnotationBeanPostProcessor`*
커스텀 `BeanPostProcessor` 구현과 함께 콜백 인터페이스나 어노테이션을 사용하는 것은 스프링 IoC 컨테이너를 확장하는 일반적인 수단입니다.

한 예는 스프링 배포판과 함께 제공되며 어노테이션된 필드, 세터 메소드 및 임의 구성 메소드를 자동 와이어링하는 `BeanPostProcessor` 구현인 스프링의 `AutowiredAnnotationBeanPostProcessor`입니다.

**`BeanFactoryPostProcessor`로 구성 메타데이터 커스터마이징 (Customizing Configuration Metadata with a `BeanFactoryPostProcessor`)**

다음으로 살펴볼 확장 지점은 `org.springframework.beans.factory.config.BeanFactoryPostProcessor`입니다.

이 인터페이스의 의미론은 `BeanPostProcessor`의 의미론과 유사하지만 한 가지 주요 차이점이 있습니다: `BeanFactoryPostProcessor`는 빈 구성 메타데이터에서 작동합니다.

즉, 스프링 IoC 컨테이너는 `BeanFactoryPostProcessor`가 구성 메타데이터를 읽고 `BeanFactoryPostProcessor` 인스턴스 외의 다른 빈을 컨테이너가 인스턴스화하기 전에 잠재적으로 변경하도록 허용합니다.

여러 `BeanFactoryPostProcessor` 인스턴스를 구성할 수 있으며, `order` 속성을 설정하여 이러한 `BeanFactoryPostProcessor` 인스턴스가 실행되는 순서를 제어할 수 있습니다.

그러나 이 속성은 `BeanFactoryPostProcessor`가 `Ordered` 인터페이스를 구현하는 경우에만 설정할 수 있습니다.

자신만의 `BeanFactoryPostProcessor`를 작성하는 경우, `Ordered` 인터페이스도 구현하는 것을 고려해야 합니다.

*실제 빈 인스턴스(즉, 구성 메타데이터에서 생성된 객체)를 변경하려면, 대신 `BeanPostProcessor`(앞서 `BeanPostProcessor`를 사용하여 빈 커스터마이징(Customizing Beans by Using a `BeanPostProcessor`)에서 설명됨)를 사용해야 합니다.*

*기술적으로 `BeanFactoryPostProcessor` 내에서 빈 인스턴스로 작업하는 것이 가능하지만(예: `BeanFactory.getBean()` 사용), 그렇게 하면 조기 빈 인스턴스화(premature bean instantiation)가 발생하여 표준 컨테이너 생명주기를 위반합니다.*

*이는 빈 후처리(bean post processing)를 우회하는 것과 같은 부정적인 부작용을 유발할 수 있습니다.*

*또한, `BeanFactoryPostProcessor` 인스턴스는 컨테이너별로 스코프 지정됩니다.*

*이는 컨테이너 계층 구조를 사용하는 경우에만 관련이 있습니다.*

*한 컨테이너에서 `BeanFactoryPostProcessor`를 정의하면 해당 컨테이너의 빈 정의에만 적용됩니다.*

*한 컨테이너의 빈 정의는 다른 컨테이너의 `BeanFactoryPostProcessor` 인스턴스에 의해 후처리되지 않습니다.*

*두 컨테이너가 동일한 계층 구조의 일부인 경우에도 마찬가지입니다.*

빈 팩토리 후처리기는 `ApplicationContext` 내부에 선언될 때 자동으로 실행되어 컨테이너를 정의하는 구성 메타데이터에 변경 사항을 적용합니다.

스프링에는 `PropertyOverrideConfigurer` 및 `PropertySourcesPlaceholderConfigurer`와 같은 여러 사전 정의된 빈 팩토리 후처리기가 포함되어 있습니다.

커스텀 속성 편집기(property editors)를 등록하는 것과 같이 커스텀 `BeanFactoryPostProcessor`를 사용할 수도 있습니다.

`ApplicationContext`는 `BeanFactoryPostProcessor` 인터페이스를 구현하는 배포된 모든 빈을 자동으로 감지합니다.

이 빈들을 빈 팩토리 후처리기(bean factory post-processors)로 사용하여 적절한 시점에 적용합니다.

다른 빈과 마찬가지로 이러한 후처리기 빈을 배포할 수 있습니다.

`*BeanPostProcessor`와 마찬가지로, 일반적으로 `BeanFactoryPostProcessor`를 지연 초기화하도록 구성하고 싶지 않을 것입니다.*

*다른 빈이 `Bean(Factory)PostProcessor`를 참조하지 않으면 해당 후처리기는 전혀 인스턴스화되지 않습니다.*

*따라서 지연 초기화로 표시하면 무시되고, `<beans />` 요소 선언에 `default-lazy-init` 속성을 `true`로 설정하더라도 `Bean(Factory)PostProcessor`는 즉시 인스턴스화됩니다.*

*예제: 클래스 이름 대체 `PropertySourcesPlaceholderConfigurer*PropertySourcesPlaceholderConfigurer`를 사용하여 표준 자바 `Properties` 형식을 사용하여 빈 정의의 속성 값을 별도 파일로 외부화(externalize)할 수 있습니다.

이렇게 하면 애플리케이션을 배포하는 사람이 컨테이너의 주 XML 정의 파일(들)을 수정하는 복잡성이나 위험 없이 데이터베이스 URL 및 암호와 같은 환경 특정 속성을 커스터마이징할 수 있습니다.

자리 표시자(placeholder) 값이 정의된 `DataSource`가 있는 다음 XML 기반 구성 메타데이터 조각을 고려하십시오:

```xml
<bean class="org.springframework.context.support.PropertySourcesPlaceholderConfigurer">
	<property name="locations" value="classpath:com/something/jdbc.properties"/>
</bean>

<bean id="dataSource" class="org.apache.commons.dbcp.BasicDataSource" destroy-method="close">
	<property name="driverClassName" value="${jdbc.driverClassName}"/>
	<property name="url" value="${jdbc.url}"/>
	<property name="username" value="${jdbc.username}"/>
	<property name="password" value="${jdbc.password}"/>
</bean>
```

이 예제는 외부 `Properties` 파일에서 구성된 속성을 보여줍니다.

런타임 시 `PropertySourcesPlaceholderConfigurer`가 `DataSource`의 일부 속성을 대체하는 메타데이터에 적용됩니다.

대체할 값은 Ant 및 log4j 및 JSP EL 스타일을 따르는 `${property-name}` 형식의 자리 표시자로 지정됩니다.

실제 값은 표준 자바 `Properties` 형식의 다른 파일에서 가져옵니다:

```
jdbc.driverClassName=org.hsqldb.jdbcDriver
jdbc.url=jdbc:hsqldb:hsql://production:9002
jdbc.username=sa
jdbc.password=root
```

따라서 `${jdbc.username}` 문자열은 런타임 시 값 'sa'로 대체되며, 프로퍼티 파일의 키와 일치하는 다른 자리 표시자 값에도 동일하게 적용됩니다.

`PropertySourcesPlaceholderConfigurer`는 빈 정의의 대부분 속성 및 속성에서 자리 표시자를 확인합니다.

또한 자리 표시자 접두사 및 접미사를 커스터마이징할 수 있습니다.

스프링 2.5에서 도입된 `context` 네임스페이스를 사용하면 전용 구성 요소를 사용하여 속성 자리 표시자를 구성할 수 있습니다.

다음 예제와 같이 `location` 속성에 쉼표로 구분된 목록으로 하나 이상의 위치를 제공할 수 있습니다:

```xml
<context:property-placeholder location="classpath:com/something/jdbc.properties"/>
```

`PropertySourcesPlaceholderConfigurer`는 지정한 `Properties` 파일에서만 속성을 찾는 것이 아닙니다.

기본적으로 지정된 프로퍼티 파일에서 속성을 찾을 수 없는 경우 스프링 환경(Environment) 속성 및 일반 자바 시스템(System) 속성을 확인합니다.

*필요한 속성을 가진 주어진 애플리케이션에 대해 이러한 요소는 하나만 정의되어야 합니다.*

*고유한 자리 표시자 구문(`${...}`)을 가지는 한 여러 속성 자리 표시자를 구성할 수 있습니다.*
*대체에 사용되는 속성의 소스를 모듈화해야 하는 경우 여러 속성 자리

---

**전체 주제: 컨테이너 확장 지점 (Container Extension Points)**

이 부분은 스프링 IoC 컨테이너의 **기본 동작 방식을 개발자가 직접 확장하거나 변경**할 수 있는 방법들에 대해 설명합니다. 스프링의 핵심 기능을 건드리지 않고도, 특정 인터페이스를 구현한 클래스를 '플러그인'처럼 끼워 넣어 컨테이너의 작동 방식에 영향을 줄 수 있다는 것이 핵심입니다.

**핵심 아이디어:** 스프링 컨테이너가 정해진 대로만 작동하는 것이 아니라, 내가 만든 특별한 규칙이나 추가 작업을 중간중간에 끼워 넣을 수 있다!

---

**첫 번째 파트: `BeanPostProcessor`를 사용하여 빈 커스터마이징**

이 부분은 스프링 컨테이너가 **빈(Bean) 객체를 생성하고 초기화하는 과정 중간에 끼어들어** 추가적인 작업을 수행하는 방법에 대해 설명합니다. `BeanPostProcessor`는 이미 **만들어진 빈 인스턴스**를 대상으로 작동합니다.

**핵심 아이디어:** 빈 객체가 만들어진 후, 초기화되기 전이나 후에 내가 원하는 추가 작업을 하자! (예: 특정 어노테이션 처리, 빈을 프록시로 감싸기 등)

**`BeanPostProcessor`란 무엇인가?**

- `org.springframework.beans.factory.config.BeanPostProcessor`는 스프링 인터페이스입니다.
- 이 인터페이스를 구현한 클래스는 스프링 컨테이너가 **모든 빈**을 생성하고 초기화할 때마다 **중간에 개입**할 기회를 얻습니다.
- 마치 공장에서 제품(빈)이 만들어진 후, 포장(초기화)하기 전이나 후에 **품질 검사**를 하거나 **특별한 스티커**를 붙이는 추가 공정 담당자와 같습니다.

**어떻게 동작하는가? (두 개의 콜백 메소드)**

`BeanPostProcessor` 인터페이스에는 두 개의 메소드가 있습니다:

1. **`postProcessBeforeInitialization(Object bean, String beanName)`:**
  - **호출 시점:** 빈 객체가 생성되고 모든 의존성 주입이 끝난 후, **초기화 콜백(`@PostConstruct`, `init-method` 등)이 호출되기 *직전*** 에 호출됩니다.
  - **역할:** 초기화 전에 빈 인스턴스를 검사하거나 변경할 수 있습니다. 예를 들어, 특정 인터페이스를 구현했는지 확인하고 추가 설정을 할 수 있습니다.
  - **반환값:** 처리된 빈 객체를 반환해야 합니다. (보통은 받은 `bean`을 그대로 반환하거나, 필요하면 완전히 다른 객체를 반환할 수도 있습니다.)
2. **`postProcessAfterInitialization(Object bean, String beanName)`:**
  - **호출 시점:** 빈의 **초기화 콜백(`@PostConstruct`, `init-method` 등)이 모두 완료된 *직후*** 에 호출됩니다.
  - **역할:** 완전히 초기화된 빈을 대상으로 마지막 추가 작업을 수행합니다. **스프링 AOP 프록시를 만드는 작업**이 주로 이 시점에서 이루어집니다. (즉, 원본 빈을 프록시 객체로 감싸서 반환합니다.)
  - **반환값:** 최종적으로 컨테이너에 등록될 빈 객체를 반환해야 합니다. (원본 `bean` 또는 프록시 객체 등)

**주요 특징 및 사용법:**

- **등록:** `BeanPostProcessor` 구현 클래스를 스프링 설정(XML, Java Config 등)에 **일반 빈처럼 등록**하기만 하면, `ApplicationContext`가 **자동으로 감지**해서 컨테이너의 후처리기 목록에 추가합니다.
- **실행 순서:** 여러 개의 `BeanPostProcessor`가 등록될 수 있습니다. 실행 순서가 중요하다면, `BeanPostProcessor` 구현 클래스가 `org.springframework.core.Ordered` 인터페이스를 함께 구현하고 `getOrder()` 메소드로 순서 값을 반환하도록 해야 합니다. (낮은 숫자가 먼저 실행됨)
- **작동 대상:** `BeanPostProcessor`는 **빈 인스턴스(객체)** 를 대상으로 작동합니다. 빈의 설정 정보(메타데이터) 자체를 바꾸지는 않습니다.
- **스코프:** `BeanPostProcessor`는 정의된 **컨테이너 내부**에서만 작동합니다. 컨테이너 계층 구조가 있을 때, 부모 컨테이너의 `BeanPostProcessor`는 자식 컨테이너의 빈에 영향을 주지 않습니다 (그 반대도 마찬가지).
- **AOP와의 관계:** 스프링의 AOP 기능(자동 프록시 생성) 자체가 `BeanPostProcessor`를 통해 구현됩니다. 이 때문에 `BeanPostProcessor` 빈 자체나 그 빈이 직접 참조하는 다른 빈들은 **자동 프록시 대상에서 제외**됩니다. (왜냐하면 AOP 관련 `BeanPostProcessor`가 완전히 준비되기 전에 이들이 먼저 생성되어야 하기 때문입니다.) 관련해서는 로그 메시지가 출력될 수 있습니다.
- **주의:** `@Bean` 메소드로 `BeanPostProcessor`를 정의할 때는 반환 타입을 인터페이스(`BeanPostProcessor`)나 실제 구현 클래스로 명확히 지정해야 스프링이 조기에 인식할 수 있습니다. (다른 빈들보다 먼저 생성되어야 하므로)

**예제 (`InstantiationTracingBeanPostProcessor`):**

- 제공된 예제는 `postProcessAfterInitialization` 메소드에서 단순히 생성된 빈의 이름과 `toString()` 결과를 콘솔에 출력하는 간단한 `BeanPostProcessor`입니다.
- 이것을 빈으로 등록하면, 컨테이너가 관리하는 모든 빈(여기서는 `messenger` 빈)이 초기화된 후에 해당 정보가 출력됩니다.

**실제 사용 예 (`AutowiredAnnotationBeanPostProcessor`):**

- 스프링 내부적으로 `@Autowired` 어노테이션을 처리하여 의존성을 자동으로 주입해주는 `AutowiredAnnotationBeanPostProcessor`가 바로 `BeanPostProcessor`의 대표적인 실제 사용 예입니다.

---

**두 번째 파트: `BeanFactoryPostProcessor`로 구성 메타데이터 커스터마이징**

이 부분은 스프링 컨테이너가 빈 객체를 **만들기 전에**, 빈을 어떻게 만들지에 대한 **설계도(설정 정보, 메타데이터)** 자체를 **읽고 수정**할 수 있는 방법에 대한 내용입니다. `BeanPostProcessor`가 이미 만들어진 '제품(빈 인스턴스)'을 다루었다면, `BeanFactoryPostProcessor`는 '제품 설계도(빈 정의)'를 다룹니다.

**핵심 아이디어:** 빈 객체를 만들기 전에, 빈 설정 정보 자체를 프로그램적으로 변경하자! (예: 설정 파일의 특정 값을 외부 파일 값으로 바꾸기)

**`BeanFactoryPostProcessor`란 무엇인가?**

- `org.springframework.beans.factory.config.BeanFactoryPostProcessor`는 스프링 인터페이스입니다.
- 이 인터페이스를 구현한 클래스는 스프링 컨테이너가 **모든 빈 정의(설정 정보)를 로드한 후, 아직 어떤 빈 인스턴스도 생성하기 전** 시점에 개입할 기회를 얻습니다.
- 마치 집(빈)을 짓기 전에, **설계 도면(빈 정의)** 을 검토하고 필요하면 수정하는 건축 감리자와 같습니다.

**어떻게 동작하는가? (하나의 콜백 메소드)**

`BeanFactoryPostProcessor` 인터페이스에는 **`postProcessBeanFactory(ConfigurableListableBeanFactory beanFactory)`** 라는 메소드가 하나 있습니다.

- **호출 시점:** 스프링 컨테이너가 모든 설정 정보(XML, Java Config 등)를 읽어서 내부적인 빈 정의(BeanDefinition) 객체들로 변환한 직후, 그리고 **어떤 빈 객체도 인스턴스화하기 전**에 호출됩니다.
- **역할:** 이 메소드 안에서 개발자는 `beanFactory` 파라미터를 통해 **모든 빈 정의 정보에 접근**할 수 있습니다. 이 정보를 읽거나, 심지어 **변경**할 수도 있습니다. 예를 들어, 특정 빈 정의의 클래스 이름을 바꾸거나, 속성 값을 변경하거나, 스코프를 바꾸는 등의 작업이 가능합니다.
- **주의:** 이 단계에서는 아직 빈 **인스턴스(객체)** 가 없습니다. 여기서 `beanFactory.getBean()` 같은 메소드를 호출해서 빈 인스턴스를 강제로 만들려고 하면, 컨테이너의 정상적인 생명주기를 방해하고 예기치 않은 문제를 일으킬 수 있으므로 **절대로 해서는 안 됩니다.** (조기 인스턴스화 문제) 작업 대상은 **빈 정의(설정 정보)** 입니다.

**주요 특징 및 사용법:**

- **등록:** `BeanPostProcessor`와 마찬가지로, `BeanFactoryPostProcessor` 구현 클래스를 스프링 설정에 **일반 빈처럼 등록**하면 `ApplicationContext`가 **자동으로 감지**하고 적절한 시점에 실행합니다.
- **실행 순서:** 여러 개의 `BeanFactoryPostProcessor`가 등록될 수 있으며, `Ordered` 인터페이스를 구현하여 실행 순서를 제어할 수 있습니다. (낮은 숫자가 먼저 실행됨)
- **스코프:** `BeanFactoryPostProcessor`도 정의된 **컨테이너 내부**에서만 작동합니다. 컨테이너 계층 구조에서 부모/자식 컨테이너 간에 서로 영향을 주지 않습니다.
- **지연 초기화 (Lazy Initialization):** `BeanFactoryPostProcessor`는 다른 빈들이 생성되기 전에 먼저 실행되어야 하므로, **지연 초기화 대상으로 설정해도 무시되고 즉시 인스턴스화됩니다.**
- **`BeanPostProcessor`와의 차이점:**
  - `BeanFactoryPostProcessor`: **빈 정의(설정 정보)** 를 대상으로 작동. 빈 인스턴스 생성 **전** 실행.
  - `BeanPostProcessor`: **빈 인스턴스(객체)** 를 대상으로 작동. 빈 인스턴스 생성 **후** 실행.

**실제 사용 예:**

- **`PropertySourcesPlaceholderConfigurer` (가장 흔한 예):**
  - XML 설정 파일 같은 곳에 `${db.url}` 처럼 **자리 표시자(placeholder)** 를 사용하고, 실제 값은 별도의 `.properties` 파일에 정의해 놓는 경우가 많습니다.
  - `PropertySourcesPlaceholderConfigurer`는 `BeanFactoryPostProcessor`의 구현체로서, 빈 정의들을 읽다가 `${...}` 형태의 자리 표시자를 발견하면, 지정된 프로퍼티 파일이나 시스템 환경 변수 등에서 실제 값을 찾아서 **빈 정의 자체를 수정하여 값을 바꿔치기** 해줍니다.
  - 이렇게 하면 실제 빈 객체가 생성될 때는 이미 올바른 값(예: 실제 DB URL)이 설정 정보에 반영되어 있게 됩니다.
  - XML 설정에서는 `<context:property-placeholder location="..."/>` 와 같은 편리한 태그로 쉽게 사용할 수 있습니다.
- **커스텀 속성 편집기(PropertyEditor) 등록:** 특정 타입의 속성 값을 어떻게 변환할지 커스텀 로직을 등록하는 데 사용될 수도 있습니다.

**요약:**

`BeanFactoryPostProcessor`는 스프링 컨테이너가 빈 객체를 만들기 전에, **빈 설정 정보(메타데이터)** 자체를 프로그램적으로 **읽고 수정**할 수 있게 해주는 강력한 확장 지점입니다. 주로 설정 값 외부화(`PropertySourcesPlaceholderConfigurer`) 등에 사용되며, 빈 인스턴스가 아닌 **빈 정의**를 대상으로 작동한다는 점이 `BeanPostProcessor`와의 가장 큰 차이점입니다.

---

**세 번째 파트: 예제 -  `PropertySourcesPlaceholderConfigurer`**

이 부분은 `BeanFactoryPostProcessor`가 실제로 어떻게 활용되는지 보여주는 대표적인 사례입니다. 설정 파일(XML 등) 안에 직접 값을 적는 대신, **자리 표시자(placeholder)** 를 사용하고 실제 값은 **외부 프로퍼티(.properties) 파일**에 보관하여 관리하는 방법을 설명합니다.

**핵심 아이디어:** 설정 정보 중 자주 바뀌거나 환경마다 다른 값들(예: DB 접속 정보, 외부 API 키)을 설정 파일과 분리해서 관리하자!

**왜 필요한가? (장점)**

- **환경 분리:** 개발 환경, 테스트 환경, 운영 환경마다 다른 DB 접속 정보나 설정을 사용해야 할 때, 각 환경에 맞는 프로퍼티 파일만 준비하면 XML 설정 자체는 수정할 필요가 없습니다.
- **보안:** DB 비밀번호 같은 민감한 정보를 XML 설정 파일에 직접 노출하지 않고 별도 파일로 관리하여 접근 제어를 용이하게 할 수 있습니다.
- **유지보수:** 여러 곳에서 사용되는 공통 설정을 프로퍼티 파일 한 곳에서 관리하면 변경이 쉬워집니다.

**사용법 및 동작 방식:**

1. **프로퍼티 파일 준비 (`.properties`):**
  - `key=value` 형식으로 설정 값들을 정의합니다.

    ```
    # 예시: jdbc.properties 파일
    jdbc.driverClassName=org.hsqldb.jdbcDriver
    jdbc.url=jdbc:hsqldb:hsql://production:9002
    jdbc.username=sa
    jdbc.password=root
    ```

2. **XML 설정 파일:**
  - **`PropertySourcesPlaceholderConfigurer` 빈 등록:** 이 빈 자체가 `BeanFactoryPostProcessor` 역할을 합니다. 스프링에게 프로퍼티 파일을 읽어서 자리 표시자를 처리하라고 알려주는 것입니다.
  - **`locations` 속성:** 읽어올 프로퍼티 파일의 위치를 지정합니다. (`classpath:`는 클래스패스 상의 경로를 의미)
  - **자리 표시자 사용:** 실제 값이 필요한 속성에는 `${프로퍼티키}` 형식으로 자리 표시자를 사용합니다.

    ```xml
    <!-- 1. PropertySourcesPlaceholderConfigurer 빈을 등록 -->
    <bean class="org.springframework.context.support.PropertySourcesPlaceholderConfigurer">
        <property name="locations" value="classpath:com/something/jdbc.properties"/>
    </bean>
    
    <!-- 2. DataSource 빈 정의 시 자리 표시자 사용 -->
    <bean id="dataSource" class="org.apache.commons.dbcp.BasicDataSource" destroy-method="close">
        <property name="driverClassName" value="${jdbc.driverClassName}"/> <!-- 여기! -->
        <property name="url" value="${jdbc.url}"/>                         <!-- 여기! -->
        <property name="username" value="${jdbc.username}"/>                 <!-- 여기! -->
        <property name="password" value="${jdbc.password}"/>                 <!-- 여기! -->
    </bean>
    ```

3. **스프링 컨테이너의 동작 과정 (중요!)**
  1. 스프링 컨테이너가 시작되면서 XML 설정 파일을 읽습니다.
  2. `PropertySourcesPlaceholderConfigurer` 빈 정의를 발견합니다. 이 빈은 `BeanFactoryPostProcessor`이므로 다른 일반 빈들보다 **먼저 인스턴스화**됩니다.
  3. `PropertySourcesPlaceholderConfigurer`의 `postProcessBeanFactory` 메소드가 **호출**됩니다. (아직 `dataSource` 빈은 만들어지지 않았습니다!)
  4. 이 메소드는 `locations` 속성에 지정된 `jdbc.properties` 파일을 **찾아서 읽어들입니다.**
  5. 컨테이너에 로드된 **모든 빈 정의(설계도)** 를 살펴봅니다.
  6. `dataSource` 빈 정의에서 `${jdbc.driverClassName}` 같은 자리 표시자를 발견합니다.
  7. 읽어들인 프로퍼티 파일에서 `jdbc.driverClassName` 키에 해당하는 값(`org.hsqldb.jdbcDriver`)을 찾습니다.
  8. `dataSource` 빈 정의의 `driverClassName` 속성 값을 `${jdbc.driverClassName}` 에서 실제 값 `org.hsqldb.jdbcDriver` 로 **수정(치환)** 합니다. 다른 자리 표시자들도 동일하게 처리합니다.
  9. 모든 자리 표시자 처리가 끝나면, 이제 **수정된 빈 정의**를 바탕으로 실제 `dataSource` 빈 **객체를 생성**합니다. 이때 `dataSource` 객체는 이미 실제 값들(DB 접속 정보)을 가지고 생성됩니다.

**추가 기능 및 편리한 설정:**

- **기본 검색 순서:** `PropertySourcesPlaceholderConfigurer`는 지정된 프로퍼티 파일에서 값을 못 찾으면, 기본적으로 **스프링 환경(Environment) 변수**나 **자바 시스템 속성(System properties)** 에서도 해당 키를 찾아봅니다.
- **접두사/접미사 변경:** `${...}` 대신 다른 기호(예: `#{...}`)를 사용하고 싶다면 관련 속성을 통해 변경할 수 있습니다.
- **`<context:property-placeholder>` 네임스페이스:** XML 설정에서 더 간결하게 사용하기 위해 `context` 네임스페이스를 사용할 수 있습니다. 아래 설정은 위의 `<bean>` 정의와 동일한 역할을 합니다.

    ```xml
    xmlns:context="<http://www.springframework.org/schema/context>"
    xsi:schemaLocation="... <http://www.springframework.org/schema/context> <https://www.springframework.org/schema/context/spring-context.xsd>"
    
    <!-- context 네임스페이스를 사용한 간결한 설정 -->
    <context:property-placeholder location="classpath:com/something/jdbc.properties"/>
    ```

  여러 파일을 지정하려면 쉼표(`,`)로 구분된 목록을 `location` 속성에 제공하면 됩니다.

- **주의:** 보통 하나의 애플리케이션 컨텍스트에는 하나의 `PropertySourcesPlaceholderConfigurer` (또는 `<context:property-placeholder>`)만 정의하는 것이 혼란을 피하는 방법입니다.

**요약:**

`PropertySourcesPlaceholderConfigurer`는 `BeanFactoryPostProcessor`의 대표적인 활용 사례로, 스프링 설정 파일 내의 **자리 표시자(`${...}`)** 를 **외부 프로퍼티 파일이나 시스템 환경의 실제 값으로 치환**해주는 역할을 합니다. 이를 통해 설정 값 관리가 유연해지고 보안성이 향상됩니다. 핵심은 빈 객체가 생성되기 전에 **빈 정의(설정 정보) 자체를 수정**한다는 점입니다.

---
