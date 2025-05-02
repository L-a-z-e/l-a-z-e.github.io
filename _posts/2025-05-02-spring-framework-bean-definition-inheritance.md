---
title: Spring Framework Bean Definition Inheritance
description: 
author: laze
date: 2025-05-02 00:00:03 +0900
categories: [Dev, SpringBoot]
tags: [SpringBoot]
---
**Bean Definition Inheritance**

빈 정의는 생성자 인수, 속성 값, 그리고 초기화 메소드, 정적 팩토리 메소드 이름 등과 같은 컨테이너 특정 정보를 포함하여 많은 구성 정보를 포함할 수 있습니다.

자식 빈 정의는 부모 정의로부터 구성 데이터를 상속받습니다.

자식 정의는 필요에 따라 일부 값을 오버라이드하거나 다른 값을 추가할 수 있습니다.

부모 및 자식 빈 정의를 사용하면 많은 타이핑을 절약할 수 있습니다.

효과적으로, 이것은 템플릿팅(templating)의 한 형태입니다.

프로그래밍 방식으로 `ApplicationContext` 인터페이스를 사용하는 경우, 자식 빈 정의는 `ChildBeanDefinition` 클래스로 표현됩니다.

대부분의 사용자는 이 레벨에서 작업하지 않습니다. 대신, `ClassPathXmlApplicationContext`와 같은 클래스에서 선언적으로 빈 정의를 구성합니다.

XML 기반 설정 메타데이터를 사용할 때, `parent` 속성을 사용하여 자식 빈 정의를 나타낼 수 있으며, 이 속성의 값으로 부모 빈을 지정합니다.

```xml
<bean id="inheritedTestBean" abstract="true"
		class="org.springframework.beans.TestBean">
	<property name="name" value="parent"/>
	<property name="age" value="1"/>
</bean>

<bean id="inheritsWithDifferentClass"
		class="org.springframework.beans.DerivedTestBean"
		parent="inheritedTestBean" init-method="initialize">
	<property name="name" value="override"/>
</bean>
```

자식 빈 정의는 지정되지 않은 경우 부모 정의의 빈 클래스를 사용하지만 오버라이드할 수도 있습니다.

후자의 경우, 자식 빈 클래스는 부모와 호환되어야 합니다(즉, 부모의 속성 값을 받아들여야 함).

자식 빈 정의는 새로운 값을 추가하는 옵션과 함께 부모로부터 스코프, 생성자 인수 값, 속성 값 및 메소드 오버라이드를 상속받습니다.

지정하는 모든 스코프, 초기화 메소드, 소멸 메소드 또는 정적 팩토리 메소드 설정은 해당하는 부모 설정을 오버라이드합니다.

나머지 설정은 항상 자식 정의에서 가져옵니다: `depends on`, 자동 와이어링 모드, 의존성 검사, 싱글톤 및 지연 초기화.

앞의 예제는 `abstract` 속성을 사용하여 부모 빈 정의를 명시적으로 추상(abstract)으로 표시합니다.

부모 정의가 클래스를 지정하지 않는 경우, 다음 예제와 같이 부모 빈 정의를 명시적으로 추상으로 표시해야 합니다:

```xml
<bean id="inheritedTestBeanWithoutClass" abstract="true">
	<property name="name" value="parent"/>
	<property name="age" value="1"/>
</bean>

<bean id="inheritsWithClass" class="org.springframework.beans.DerivedTestBean"
		parent="inheritedTestBeanWithoutClass" init-method="initialize">
	<property name="name" value="override"/>
</bean>
```

부모 빈은 불완전하고 명시적으로 추상으로 표시되었기 때문에 자체적으로 인스턴스화될 수 없습니다.

정의가 추상일 때, 자식 정의의 부모 정의 역할을 하는 순수한 템플릿 빈 정의로만 사용 가능합니다.

이러한 추상 부모 빈을 다른 빈의 `ref` 속성으로 참조하거나 부모 빈 ID로 명시적인 `getBean()` 호출을 하여 자체적으로 사용하려고 시도하면 오류가 반환됩니다.

유사하게, 컨테이너의 내부 `preInstantiateSingletons()` 메소드는 추상으로 정의된 빈 정의를 무시합니다.

`*ApplicationContext`는 기본적으로 모든 싱글톤을 미리 인스턴스화합니다.*

*따라서 (적어도 싱글톤 빈의 경우) 템플릿으로만 사용하려는 (부모) 빈 정의가 있고 이 정의가 클래스를 지정한다면, `abstract` 속성을 `true`로 설정해야 합니다.*

*그렇지 않으면 애플리케이션 컨텍스트가 실제로 추상 빈을 미리 인스턴스화하려고 시도할 것입니다.*
