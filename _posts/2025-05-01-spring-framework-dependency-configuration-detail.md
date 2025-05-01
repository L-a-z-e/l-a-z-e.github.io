---
title: Spring Framework Dependencies and Configuration
description: 
author: laze
date: 2025-05-01 00:00:04 +0900
categories: [Dev, SpringBoot]
tags: [SpringBoot]
---
# **Dependencies and Configuration in Detail**

## **의존성 및 설정 상세 (Dependencies and Configuration in Detail)**

빈 속성 및 생성자 인수를 다른 관리되는 빈(협력자)에 대한 참조 또는 인라인으로 정의된 값으로 정의할 수 있습니다.

스프링의 XML 기반 설정 메타데이터는 이를 위해 `<property/>` 및 `<constructor-arg/>` 요소 내의 하위 요소 타입을 지원합니다.

**직접 값 (기본 타입, 문자열 등) (Straight Values (Primitives, Strings, and so on))**

`<property/>` 요소의 `value` 속성은 속성 또는 생성자 인수를 사람이 읽을 수 있는 문자열 표현으로 지정합니다.

스프링의 변환 서비스(conversion service)는 이러한 값들을 문자열에서 속성 또는 인수의 실제 타입으로 변환하는 데 사용됩니다.

```xml
<bean id="myDataSource" class="org.apache.commons.dbcp.BasicDataSource" destroy-method="close">
	<!-- setDriverClassName(String) 호출로 이어짐 -->
	<property name="driverClassName" value="com.mysql.jdbc.Driver"/>
	<property name="url" value="jdbc:mysql://localhost:3306/mydb"/>
	<property name="username" value="root"/>
	<property name="password" value="misterkaoli"/>
</bean>
```

다음 예제는 훨씬 더 간결한 XML 구성을 위해 p-네임스페이스(p-namespace)를 사용합니다:

```xml
<beans xmlns="<http://www.springframework.org/schema/beans>"
	xmlns:xsi="<http://www.w3.org/2001/XMLSchema-instance>"
	xmlns:p="<http://www.springframework.org/schema/p>"
	xsi:schemaLocation="<http://www.springframework.org/schema/beans>
	<https://www.springframework.org/schema/beans/spring-beans.xsd>">

	<bean id="myDataSource" class="org.apache.commons.dbcp.BasicDataSource"
		destroy-method="close"
		p:driverClassName="com.mysql.jdbc.Driver"
		p:url="jdbc:mysql://localhost:3306/mydb"
		p:username="root"
		p:password="misterkaoli"/>
</beans>
```

앞의 XML은 더 간결합니다. 그러나 빈 정의를 생성할 때 자동 속성 완성을 지원하는 IDE(예: IntelliJ IDEA 또는 Spring Tools for Eclipse)를 사용하지 않는 한, 오타는 디자인 타임이 아닌 런타임에 발견됩니다.

다음과 같이 `java.util.Properties` 인스턴스를 구성할 수도 있습니다:

```xml
<bean id="mappings"
	class="org.springframework.context.support.PropertySourcesPlaceholderConfigurer">

	<!-- java.util.Properties 타입으로 지정됨 -->
	<property name="properties">
		<value>
			jdbc.driver.className=com.mysql.jdbc.Driver
			jdbc.url=jdbc:mysql://localhost:3306/mydb <!-- 줄바꿈을 통해 개별 프로퍼티로 인식함 -->
		</value>
	</property>
</bean>
```

스프링 컨테이너는 JavaBeans `PropertyEditor` 메커니즘을 사용하여 `<value/>` 요소 내부의 텍스트를 `java.util.Properties` 인스턴스로 변환합니다.

이는 편리한 단축 방법이며, 스프링 팀이 `value` 속성 스타일보다 중첩된 `<value/>` 요소 사용을 선호하는 몇 안 되는 경우 중 하나입니다.

**idref 요소 (The idref element)**

`idref` 요소는 컨테이너 내의 다른 빈의 id(참조가 아닌 문자열 값)를 `<constructor-arg/>` 또는 `<property/>` 요소에 전달하는 간단하고 오류 방지적인 방법입니다. 다음 예제는 사용 방법을 보여줍니다:

```xml
<bean id="theTargetBean" class="..."/>

<bean id="theClientBean" class="...">
	<property name="targetName">
		<idref bean="theTargetBean"/>
	</property>
</bean>
```

앞의 빈 정의 스니펫은 (런타임 시) 다음 스니펫과 정확히 동일합니다:

```xml
<bean id="theTargetBean" class="..." />

<bean id="theClientBean" class="...">
	<property name="targetName" value="theTargetBean"/>
</bean>
```

첫 번째 형식이 두 번째 형식보다 선호되는데, `idref` 태그를 사용하면 컨테이너가 배포 시점에 참조된 이름의 빈이 실제로 존재하는지 검증할 수 있기 때문입니다.

두 번째 변형에서는 클라이언트 빈의 `targetName` 속성에 전달되는 값에 대한 검증이 수행되지 않습니다.

오타는 클라이언트 빈이 실제로 인스턴스화될 때 (대부분 치명적인 결과와 함께) 발견됩니다.

클라이언트 빈이 프로토타입 빈인 경우, 이 오타와 결과적인 예외는 컨테이너가 배포된 후 오랜 시간이 지나서야 발견될 수 있습니다. **

`<idref/>` 요소가 가치를 제공하는 일반적인 장소(적어도 Spring 2.0 이전 버전에서)는 `ProxyFactoryBean` 빈 정의에서 AOP 인터셉터 구성입니다.

인터셉터 이름을 지정할 때 `<idref/>` 요소를 사용하면 인터셉터 ID를 잘못 입력하는 것을 방지할 수 있습니다.

**다른 빈(협력자)에 대한 참조 (References to Other Beans (Collaborators))**

`ref` 요소는 `<constructor-arg/>` 또는 `<property/>` 정의 요소 내부의 마지막 요소입니다.

여기서는 빈의 지정된 속성 값을 컨테이너가 관리하는 다른 빈(협력자)에 대한 참조로 설정합니다.

참조된 빈은 속성이 설정될 빈의 의존성이며, 속성이 설정되기 전에 필요에 따라 초기화됩니다. (협력자가 싱글톤 빈인 경우, 컨테이너에 의해 이미 초기화되었을 수 있습니다.)

모든 참조는 궁극적으로 다른 객체에 대한 참조입니다.

스코프 지정 및 유효성 검사는 다른 객체의 ID 또는 이름을 `bean` 또는 `parent` 속성을 통해 지정하는지에 따라 달라집니다.

`<ref/>` 태그의 `bean` 속성을 통해 대상 빈을 지정하는 것이 가장 일반적인 형태이며, 동일한 XML 파일에 있는지 여부에 관계없이 동일한 컨테이너 또는 부모 컨테이너의 모든 빈에 대한 참조 생성을 허용합니다.

`bean` 속성의 값은 대상 빈의 `id` 속성과 같거나 대상 빈의 `name` 속성 값 중 하나와 같을 수 있습니다. 다음 예제는 `ref` 요소를 사용하는 방법을 보여줍니다:

```xml
<ref bean="someBean"/>
```

`parent` 속성을 통해 대상 빈을 지정하면 현재 컨테이너의 부모 컨테이너에 있는 빈에 대한 참조를 생성합니다.

`parent` 속성의 값은 대상 빈의 `id` 속성 또는 대상 빈의 `name` 속성 값 중 하나와 같을 수 있습니다.

대상 빈은 현재 컨테이너의 부모 컨테이너에 있어야 합니다.

이 빈 참조 변형은 주로 컨테이너 계층 구조가 있고 부모 컨테이너의 기존 빈을 부모 빈과 동일한 이름을 가진 프록시로 감싸고 싶을 때 사용해야 합니다.

다음 한 쌍의 목록은 `parent` 속성을 사용하는 방법을 보여줍니다:

```xml
<!-- 부모 컨텍스트에서 -->
<bean id="accountService" class="com.something.SimpleAccountService">
	<!-- 필요에 따라 여기에 의존성 삽입 -->
</bean>
```

```xml
<!-- 자식 (하위) 컨텍스트에서, 빈 이름은 부모 빈과 동일함 -->
<bean id="accountService"
	class="org.springframework.aop.framework.ProxyFactoryBean">
	<property name="target">
		<ref parent="accountService"/> <!-- 부모 빈을 어떻게 참조하는지 주목 -->
	</property>
	<!-- 필요에 따라 여기에 다른 구성 및 의존성 삽입 -->
</bean>
```

**내부 빈 (Inner Beans)**

`<property/>` 또는 `<constructor-arg/>` 요소 내부의 `<bean/>` 요소는 내부 빈(inner bean)을 정의합니다. 다음 예제를 참조하십시오:

```xml
<bean id="outer" class="...">
	<!-- 대상 빈에 대한 참조를 사용하는 대신, 단순히 대상 빈을 인라인으로 정의 -->
	<property name="target">
		<bean class="com.example.Person"> <!-- 이것이 내부 빈 -->
			<property name="name" value="Fiona Apple"/>
			<property name="age" value="25"/>
		</bean>
	</property>
</bean>
```

내부 빈 정의는 정의된 ID나 이름이 필요하지 않습니다.

지정되더라도 컨테이너는 해당 값을 식별자로 사용하지 않습니다.

컨테이너는 또한 생성 시 스코프 플래그를 무시하는데, 내부 빈은 항상 익명이며 항상 외부 빈과 함께 생성되기 때문입니다.

내부 빈에 독립적으로 접근하거나 둘러싸는 빈 외의 협력 빈에 주입하는 것은 불가능합니다.

*특수한 경우로, 커스텀 스코프에서 소멸 콜백(destruction callbacks)을 받는 것이 가능합니다 - 예를 들어, 싱글톤 빈 내에 포함된 요청 스코프(request-scoped) 내부 빈의 경우.*

*내부 빈 인스턴스의 생성은 포함하는 빈에 연결되어 있지만, 소멸 콜백은 요청 스코프의 생명주기에 참여하게 합니다. 이는 일반적인 시나리오는 아닙니다. 내부 빈은 일반적으로 단순히 포함하는 빈의 스코프를 공유합니다.*

**컬렉션 (Collections)**

`<list/>`, `<set/>`, `<map/>`, `<props/>` 요소는 각각 자바 컬렉션 타입인 List, Set, Map, Properties의 속성 및 인수를 설정합니다. 다음 예제는 사용 방법을 보여줍니다:

```xml
<bean id="moreComplexObject" class="example.ComplexObject">
	<!-- setAdminEmails(java.util.Properties) 호출로 이어짐 -->
	<property name="adminEmails">
		<props>
			<prop key="administrator">administrator@example.org</prop>
			<prop key="support">support@example.org</prop>
			<prop key="development">development@example.org</prop>
		</props>
	</property>
	<!-- setSomeList(java.util.List) 호출로 이어짐 -->
	<property name="someList">
		<list>
			<value>a list element followed by a reference</value>
			<ref bean="myDataSource" />
		</list>
	</property>
	<!-- setSomeMap(java.util.Map) 호출로 이어짐 -->
	<property name="someMap">
		<map>
			<entry key="an entry" value="just some string"/>
			<entry key="a ref" value-ref="myDataSource"/>
		</map>
	</property>
	<!-- setSomeSet(java.util.Set) 호출로 이어짐 -->
	<property name="someSet">
		<set>
			<value>just some string</value>
			<ref bean="myDataSource" />
		</set>
	</property>
</bean>
```

**컬렉션 병합 (Collection Merging)**

스프링 컨테이너는 컬렉션 병합도 지원합니다.

애플리케이션 개발자는 부모 `<list/>`, `<map/>`, `<set/>` 또는 `<props/>` 요소를 정의하고 자식 `<list/>`, `<map/>`, `<set/>` 또는 `<props/>` 요소가 부모 컬렉션의 값을 상속하고 오버라이드하도록 할 수 있습니다.

즉, 자식 컬렉션의 값은 부모와 자식 컬렉션의 요소를 병합한 결과이며, 자식 컬렉션의 요소가 부모 컬렉션에 지정된 값을 오버라이드합니다.

다음 예제는 컬렉션 병합을 보여줍니다:

```xml
<beans>
	<bean id="parent" abstract="true" class="example.ComplexObject">
		<property name="adminEmails">
			<props>
				<prop key="administrator">administrator@example.com</prop>
				<prop key="support">support@example.com</prop>
			</props>
		</property>
	</bean>
	<bean id="child" parent="parent">
		<property name="adminEmails">
			<!-- 병합은 자식 컬렉션 정의에 지정됨 -->
			<props merge="true">
				<prop key="sales">sales@example.com</prop>
				<prop key="support">support@example.co.uk</prop>
			</props>
		</property>
	</bean>
<beans>
```

자식 빈 정의의 `adminEmails` 속성의 `<props/>` 요소에 `merge=true` 속성을 사용한 점에 주목하십시오. 컨테이너가 자식 빈을 해석하고 인스턴스화할 때, 결과 인스턴스는 부모의 `adminEmails` 컬렉션과 자식의 `adminEmails` 컬렉션을 병합한 결과를 포함하는 `adminEmails` Properties 컬렉션을 가집니다. 다음 목록은 결과를 보여줍니다:

```
administrator=administrator@example.com
sales=sales@example.com
support=support@example.co.uk
```

자식 Properties 컬렉션의 값 집합은 부모 `<props/>`의 모든 속성 요소를 상속하며, `support` 값에 대한 자식의 값은 부모 컬렉션의 값을 오버라이드합니다.

이 병합 동작은 `<list/>`, `<map/>`, `<set/>` 컬렉션 타입에도 유사하게 적용됩니다.

`<list/>` 요소의 특정 경우, List 컬렉션 타입과 관련된 의미론(즉, 값의 순서 있는 컬렉션 개념)이 유지됩니다.

부모의 값은 모든 자식 리스트의 값보다 앞에 옵니다.

Map, Set, Properties 컬렉션 타입의 경우 순서가 없습니다.

따라서 컨테이너가 내부적으로 사용하는 관련 Map, Set, Properties 구현 타입의 기반이 되는 컬렉션 타입에는 순서 의미론이 적용되지 않습니다.

*컬렉션 병합의 제한 사항 (Limitations of Collection Merging)*
다른 컬렉션 타입(예: Map과 List)을 병합할 수 없습니다. 시도하면 적절한 예외가 발생합니다.

`merge` 속성은 하위의, 상속된, 자식 정의에 지정되어야 합니다.

부모 컬렉션 정의에 `merge` 속성을 지정하는 것은 중복되며 원하는 병합 결과를 얻지 못합니다.

*강력한 타입의 컬렉션 (Strongly-typed collection)*
자바의 제네릭 타입 지원 덕분에 강력한 타입의 컬렉션을 사용할 수 있습니다.

즉, (예를 들어) String 요소만 포함할 수 있도록 컬렉션 타입을 선언하는 것이 가능합니다.

스프링을 사용하여 강력한 타입의 컬렉션을 빈에 의존성 주입하는 경우, 스프링의 타입 변환 지원을 활용하여 강력한 타입의 컬렉션 인스턴스 요소들이 컬렉션에 추가되기 전에 적절한 타입으로 변환되도록 할 수 있습니다.

다음 자바 클래스와 빈 정의는 이를 수행하는 방법을 보여줍니다:

```java
// Java
public class SomeClass {

	private Map<String, Float> accounts;

	public void setAccounts(Map<String, Float> accounts) {
		this.accounts = accounts;
	}
}

```

```kotlin
// Kotlin
class SomeClass {
    var accounts: Map<String, Float>? = null
}

```

```xml
<beans>
	<bean id="something" class="x.y.SomeClass">
		<property name="accounts">
			<map>
				<entry key="one" value="9.99"/>
				<entry key="two" value="2.75"/>
				<entry key="six" value="3.99"/>
			</map>
		</property>
	</bean>
</beans>

```

`something` 빈의 `accounts` 속성이 주입 준비될 때, 강력한 타입의 `Map<String, Float>`의 요소 타입에 대한 제네릭 정보는 리플렉션을 통해 사용할 수 있습니다.

따라서 스프링의 타입 변환 인프라는 다양한 값 요소들이 `Float` 타입임을 인식하고, 문자열 값(`9.99`, `2.75`, `3.99`)은 실제 `Float` 타입으로 변환됩니다.

**Null 및 빈 문자열 값 (Null and Empty String Values)**

스프링은 속성 등에 대한 빈 인수를 빈 문자열(empty Strings)로 처리합니다. 다음 XML 기반 설정 메타데이터 스니펫은 `email` 속성을 빈 문자열 값("")으로 설정합니다.

```xml
<bean class="ExampleBean">
	<property name="email" value=""/>
</bean>
```

앞의 예제는 다음 자바 코드와 동일합니다:

```java
// Java
exampleBean.setEmail("");
```

```kotlin
// Kotlin
exampleBean.email = ""
```

`<null/>` 요소는 null 값을 처리합니다.

```xml
<bean class="ExampleBean">
	<property name="email">
		<null/>
	</property>
</bean>

```

```java
// Java
exampleBean.setEmail(null);
```

```kotlin
// Kotlin
exampleBean.email = null
```

**p-네임스페이스를 사용한 XML 단축 (XML Shortcut with the p-namespace)**

p-네임스페이스를 사용하면 빈 요소의 속성(중첩된 `<property/>` 요소 대신)을 사용하여 속성 값, 협력 빈 또는 둘 다를 설명할 수 있습니다.

스프링은 XML 스키마 정의에 기반한 네임스페이스를 사용하여 확장 가능한 구성 형식을 지원합니다.

`beans` 구성 형식은 XML 스키마 문서에 정의되어 있습니다.

그러나 p-네임스페이스는 XSD 파일에 정의되어 있지 않으며 스프링 코어에만 존재합니다.

다음 예제는 동일한 결과를 내는 두 개의 XML 스니펫(첫 번째는 표준 XML 형식, 두 번째는 p-네임스페이스 사용)을 보여줍니다:

```xml
<beans xmlns="<http://www.springframework.org/schema/beans>"
	xmlns:xsi="<http://www.w3.org/2001/XMLSchema-instance>"
	xmlns:p="<http://www.springframework.org/schema/p>"
	xsi:schemaLocation="<http://www.springframework.org/schema/beans>
		<https://www.springframework.org/schema/beans/spring-beans.xsd>">

	<bean name="classic" class="com.example.ExampleBean">
		<property name="email" value="someone@somewhere.com"/>
	</bean>

	<bean name="p-namespace" class="com.example.ExampleBean"
		p:email="someone@somewhere.com"/>
</beans>

```

이 예제는 빈 정의에 `email`이라는 p-네임스페이스의 속성을 보여줍니다. 이는 스프링에게 속성 선언을 포함하도록 지시합니다. 앞서 언급했듯이, p-네임스페이스에는 스키마 정의가 없으므로 속성 이름을 속성 이름으로 설정할 수 있습니다.

다음 예제는 다른 빈에 대한 참조를 가진 두 개의 빈 정의를 더 포함합니다:

```xml
<beans xmlns="<http://www.springframework.org/schema/beans>"
	xmlns:xsi="<http://www.w3.org/2001/XMLSchema-instance>"
	xmlns:p="<http://www.springframework.org/schema/p>"
	xsi:schemaLocation="<http://www.springframework.org/schema/beans>
		<https://www.springframework.org/schema/beans/spring-beans.xsd>">

	<bean name="john-classic" class="com.example.Person">
		<property name="name" value="John Doe"/>
		<property name="spouse" ref="jane"/>
	</bean>

	<bean name="john-modern"
		class="com.example.Person"
		p:name="John Doe"
		p:spouse-ref="jane"/>

	<bean name="jane" class="com.example.Person">
		<property name="name" value="Jane Doe"/>
	</bean>
</beans>

```

이 예제는 p-네임스페이스를 사용한 속성 값뿐만 아니라 속성 참조를 선언하기 위한 특수 형식도 사용합니다.

첫 번째 빈 정의는 `john` 빈에서 `jane` 빈으로의 참조를 생성하기 위해 `<property name="spouse" ref="jane"/>`를 사용하는 반면, 두 번째 빈 정의는 정확히 동일한 작업을 수행하기 위해 `p:spouse-ref="jane"`을 속성으로 사용합니다.

이 경우 `spouse`는 속성 이름이고 `ref` 부분은 이것이 직접 값이 아니라 다른 빈에 대한 참조임을 나타냅니다.

*p-네임스페이스는 표준 XML 형식만큼 유연하지 않습니다.*

*예를 들어, 속성 참조를 선언하는 형식은 `Ref`로 끝나는 속성과 충돌하지만 표준 XML 형식은 그렇지 않습니다. 접근 방식을 신중하게 선택하고 팀원들과 소통하여 세 가지 접근 방식을 동시에 사용하는 XML 문서를 생성하지 않도록 권장합니다.*

**c-네임스페이스를 사용한 XML 단축 (XML Shortcut with the c-namespace)**

p-네임스페이스를 사용한 XML 단축과 유사하게, 스프링 3.1에서 도입된 c-네임스페이스는 중첩된 `constructor-arg` 요소 대신 생성자 인수를 구성하기 위한 인라인 속성을 허용합니다.

다음 예제는 생성자 기반 의존성 주입에서와 동일한 작업을 수행하기 위해 `c:` 네임스페이스를 사용합니다:

```xml
<beans xmlns="<http://www.springframework.org/schema/beans>"
	xmlns:xsi="<http://www.w3.org/2001/XMLSchema-instance>"
	xmlns:c="<http://www.springframework.org/schema/c>"
	xsi:schemaLocation="<http://www.springframework.org/schema/beans>
		<https://www.springframework.org/schema/beans/spring-beans.xsd>">

	<bean id="beanTwo" class="x.y.ThingTwo"/>
	<bean id="beanThree" class="x.y.ThingThree"/>

	<!-- 선택적 인수 이름을 사용한 전통적인 선언 -->
	<bean id="beanOne" class="x.y.ThingOne">
		<constructor-arg name="thingTwo" ref="beanTwo"/>
		<constructor-arg name="thingThree" ref="beanThree"/>
		<constructor-arg name="email" value="something@somewhere.com"/>
	</bean>

	<!-- 인수 이름을 사용한 c-네임스페이스 선언 -->
	<bean id="beanOne" class="x.y.ThingOne" c:thingTwo-ref="beanTwo"
		c:thingThree-ref="beanThree" c:email="something@somewhere.com"/>

</beans>
```

`c:` 네임스페이스는 이름으로 생성자 인수를 설정하기 위해 `p:` 네임스페이스와 동일한 관례(빈 참조를 위한 후행 `-ref`)를 사용합니다. 유사하게, XSD 스키마에 정의되어 있지 않더라도(스프링 코어 내부에 존재함) XML 파일에 선언되어야 합니다.

생성자 인수 이름을 사용할 수 없는 드문 경우(일반적으로 바이트코드가 `-parameters` 플래그 없이 컴파일된 경우)에는 다음과 같이 인수 인덱스로 대체할 수 있습니다:

```xml
<!-- c-네임스페이스 인덱스 선언 -->
<bean id="beanOne" class="x.y.ThingOne" c:_0-ref="beanTwo" c:_1-ref="beanThree"
	c:_2="something@somewhere.com"/>
```

XML 문법 때문에 인덱스 표기법은 선행 `_`의 존재를 요구하는데, XML 속성 이름은 숫자로 시작할 수 없기 때문입니다 (일부 IDE는 허용하더라도).

`<constructor-arg>` 요소에 대해서도 해당 인덱스 표기법을 사용할 수 있지만, 일반적으로 선언 순서만으로 충분하기 때문에 흔히 사용되지 않습니다.
*실제로 생성자 해석 메커니즘은 인수를 매칭하는 데 매우 효율적이므로, 정말로 필요하지 않다면 구성 전체에 걸쳐 이름 표기법을 사용하는 것이 좋습니다.*

**복합 속성 이름 (Compound Property Names)**

빈 속성을 설정할 때 마지막 속성 이름을 제외한 경로의 모든 구성 요소가 null이 아닌 한, 복합 또는 중첩된 속성 이름을 사용할 수 있습니다.

```xml
<bean id="something" class="things.ThingOne">
	<property name="fred.bob.sammy" value="123" />
</bean>
```

`something` 빈에는 `fred` 속성이 있고, 이 속성에는 `bob` 속성이 있으며, 이 속성에는 `sammy` 속성이 있고, 그 마지막 `sammy` 속성이 `123` 값으로 설정됩니다. 이것이 작동하려면 빈이 생성된 후 `something`의 `fred` 속성과 `fred`의 `bob` 속성이 null이 아니어야 합니다. 그렇지 않으면 `NullPointerException`이 발생합니다.
