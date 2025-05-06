---
title: Using the "auto-proxy" facility
description: 
author: laze
date: 2025-05-06 00:00:11 +0900
categories: [Dev, SpringBoot]
tags: [SpringBoot]
---
**Using the "auto-proxy" facility**

지금까지는 `ProxyFactoryBean` 또는 유사한 팩토리 빈을 사용하여 AOP 프록시를 명시적으로 생성하는 것을 고려했습니다.

스프링은 또한 선택된 빈 정의를 자동으로 프록시할 수 있는 "자동 프록시(auto-proxy)" 빈 정의를 사용할 수 있게 합니다.

이것은 스프링의 "빈 후처리기(bean post processor)" 인프라에 구축되어 있으며, 컨테이너가 로드될 때 모든 빈 정의의 수정을 가능하게 합니다.

이 모델에서는 자동 프록시 인프라를 구성하기 위해 XML 빈 정의 파일에 몇 가지 특별한 빈 정의를 설정합니다.

이를 통해 자동 프록시 대상이 될 대상을 선언할 수 있습니다. `ProxyFactoryBean`을 사용할 필요가 없습니다.

이를 수행하는 두 가지 방법이 있습니다:

1. 현재 컨텍스트의 특정 빈을 참조하는 자동 프록시 생성기(auto-proxy creator) 사용.
2. 별도로 고려할 가치가 있는 자동 프록시 생성의 특별한 경우: 소스 레벨 메타데이터 속성에 의해 구동되는 자동 프록시 생성.

**자동 프록시 빈 정의 (Auto-proxy Bean Definitions)**

이 섹션에서는 `org.springframework.aop.framework.autoproxy` 패키지에서 제공하는 자동 프록시 생성기를 다룹니다.

*BeanNameAutoProxyCreator*`BeanNameAutoProxyCreator` 클래스는 리터럴 값 또는 와일드카드와 일치하는 이름을 가진 빈에 대해 자동으로 AOP 프록시를 생성하는 `BeanPostProcessor`입니다.

다음 예제는 `BeanNameAutoProxyCreator` 빈을 생성하는 방법을 보여줍니다:

```xml
<bean class="org.springframework.aop.framework.autoproxy.BeanNameAutoProxyCreator">
	<property name="beanNames" value="jdk*,onlyJdk"/> <!-- 프록시할 빈 이름 패턴 -->
	<property name="interceptorNames"> <!-- 적용할 인터셉터/어드바이저 이름 -->
		<list>
			<value>myInterceptor</value> <!-- 'myInterceptor' 빈 참조 -->
		</list>
	</property>
</bean>
```

`ProxyFactoryBean`과 마찬가지로, 프로토타입 어드바이저에 대한 올바른 동작을 허용하기 위해 인터셉터 목록 대신 `interceptorNames` 속성이 있습니다.

이름 붙여진 "인터셉터(interceptors)"는 어드바이저 또는 모든 어드바이스 타입일 수 있습니다.

일반적인 자동 프록시와 마찬가지로, `BeanNameAutoProxyCreator` 사용의 주요 요점은 최소한의 구성 볼륨으로 여러 객체에 동일한 구성을 일관되게 적용하는 것입니다.

여러 객체에 선언적 트랜잭션을 적용하는 데 널리 사용되는 선택입니다.

앞의 예제의 `jdkMyBean` 및 `onlyJdk`와 같이 이름이 일치하는 빈 정의는 대상 클래스를 가진 일반적인 오래된 빈 정의입니다.

AOP 프록시는 `BeanNameAutoProxyCreator`에 의해 자동으로 생성됩니다.

동일한 어드바이스가 모든 일치하는 빈에 적용됩니다. (앞의 예제의 인터셉터 대신) 어드바이저가 사용되는 경우, 포인트컷이 다른 빈에 다르게 적용될 수 있다는 점에 유의하십시오.

*DefaultAdvisorAutoProxyCreator*
더 일반적이고 매우 강력한 자동 프록시 생성기는 `DefaultAdvisorAutoProxyCreator`입니다.

이것은 자동 프록시 어드바이저의 빈 정의에 특정 빈 이름을 포함할 필요 없이 현재 컨텍스트의 적격한(eligible) 어드바이저를 자동으로(automagically) 적용합니다.

`BeanNameAutoProxyCreator`와 동일하게 일관된 구성 및 중복 회피의 장점을 제공합니다.

이 메커니즘을 사용하는 것은 다음을 포함합니다:

1. `DefaultAdvisorAutoProxyCreator` 빈 정의 지정.
2. 동일하거나 관련된 컨텍스트에 임의 개수의 어드바이저 지정. 이것들은 인터셉터나 다른 어드바이스가 아닌 어드바이저여야 한다는 점에 유의하십시오. 후보 빈 정의에 대한 각 어드바이스의 적격성을 확인하기 위해 평가할 포인트컷이 있어야 하기 때문에 이것이 필요합니다.

`DefaultAdvisorAutoProxyCreator`는 각 어드바이저에 포함된 포인트컷을 자동으로 평가하여 각 비즈니스 객체(예: 예제의 `businessObject1` 및 `businessObject2`)에 어떤 어드바이스(있는 경우)를 적용해야 하는지 확인합니다.

이는 즉, 임의 개수의 어드바이저가 각 비즈니스 객체에 자동으로 적용될 수 있음을 의미합니다. 어드바이저 중 어떤 포인트컷도 비즈니스 객체의 어떤 메소드와도 일치하지 않으면 객체는 프록시되지 않습니다.

새 비즈니스 객체에 대한 빈 정의가 추가되면 필요한 경우 자동으로 프록시됩니다.

일반적인 자동 프록시는 호출자나 의존성이 어드바이스받지 않은 객체를 얻는 것을 불가능하게 만드는 장점이 있습니다.

이 `ApplicationContext`에서 `getBean("businessObject1")`을 호출하면 대상 비즈니스 객체가 아닌 AOP 프록시가 반환됩니다. (앞서 보여준 "내부 빈" 관용구도 이 이점을 제공합니다.)

다음 예제는 `DefaultAdvisorAutoProxyCreator` 빈과 이 섹션에서 논의된 다른 요소들을 생성합니다:

```xml
<!-- 이 빈은 컨텍스트의 모든 Advisor를 찾아서 적용 가능한 빈에 자동으로 프록시를 생성합니다 -->
<bean class="org.springframework.aop.framework.autoproxy.DefaultAdvisorAutoProxyCreator"/>

<!-- 예: 트랜잭션 어드바이저 (포인트컷 내장 또는 별도 정의 가능) -->
<bean class="org.springframework.transaction.interceptor.TransactionAttributeSourceAdvisor">
	<property name="transactionInterceptor" ref="transactionInterceptor"/> <!-- TransactionInterceptor 빈 참조 -->
</bean>

<!-- 사용자 정의 어드바이저 -->
<bean id="customAdvisor" class="com.mycompany.MyAdvisor"/> <!-- Assuming MyAdvisor implements Advisor -->

<!-- 대상 빈 1 -->
<bean id="businessObject1" class="com.mycompany.BusinessObject1">
	<!-- 속성 생략 -->
</bean>

<!-- 대상 빈 2 -->
<bean id="businessObject2" class="com.mycompany.BusinessObject2"/>

<!-- TransactionInterceptor 빈 (TransactionAttributeSourceAdvisor에서 참조됨) -->
<bean id="transactionInterceptor" class="org.springframework.transaction.interceptor.TransactionInterceptor">
    <property name="transactionManager" ref="transactionManager"/> <!-- TransactionManager 빈 참조 -->
    <property name="transactionAttributeSource">
        <bean class="org.springframework.transaction.annotation.AnnotationTransactionAttributeSource"/>
    </property>
</bean>

<!-- TransactionManager 빈 (위에 참조됨) -->
<bean id="transactionManager" class="org.springframework.jdbc.datasource.DataSourceTransactionManager">
     <property name="dataSource" ref="dataSource"/> <!-- DataSource 빈 참조 -->
</bean>

<!-- 기타 필요한 빈들 (DataSource, MyAdvisor 구현 등) -->
</beans>
```

`DefaultAdvisorAutoProxyCreator`는 많은 비즈니스 객체에 동일한 어드바이스를 일관되게 적용하려는 경우 매우 유용합니다.

인프라 정의가 준비되면 특정 프록시 구성 없이 새 비즈니스 객체를 추가할 수 있습니다. 또한 추가적인 애스펙트(예: 추적 또는 성능 모니터링 애스펙트)를 최소한의 구성 변경으로 쉽게 추가(drop in)할 수 있습니다.

`DefaultAdvisorAutoProxyCreator`는 필터링(명명 규칙을 사용하여 특정 어드바이저만 평가되도록 함으로써 동일한 팩토리에서 여러 다르게 구성된 `AdvisorAutoProxyCreators` 사용 허용) 및 순서 지정을 지원합니다.

어드바이저는 이것이 문제인 경우 올바른 순서를 보장하기 위해 `org.springframework.core.Ordered` 인터페이스를 구현할 수 있습니다.

앞의 예제에서 사용된 `TransactionAttributeSourceAdvisor`는 구성 가능한 순서 값을 가집니다. 기본 설정은 순서 없음(unordered)입니다.

---

**전체 주제: "자동 프록시" 기능 사용하기 (Using the "auto-proxy" facility)**

이 부분은 스프링 IoC 컨테이너가 로드될 때 **특정 규칙에 맞는 빈들을 자동으로 찾아서 AOP 프록시를 생성**해주는 메커니즘에 대해 설명합니다.

개발자는 프록시 생성 자체를 신경 쓸 필요 없이, 어떤 빈에 어떤 부가 기능을 적용할지에 대한 규칙(어드바이저 등)만 정의하면 됩니다.

이 기능은 스프링의 **빈 후처리기(`BeanPostProcessor`) 인프라**를 기반으로 동작합니다.

**핵심 아이디어:** 일일이 `ProxyFactoryBean` 설정하기 귀찮다! 스프링에게 "이런 규칙에 맞는 빈들은 네가 알아서 찾아서 AOP 프록시로 만들어줘!" 라고 설정 한 번만 해두자!

---

**자동 프록시 생성 방법 (두 가지 주요 방식):**

스프링은 자동 프록시 생성을 위한 두 가지 주요 `BeanPostProcessor` 구현체를 제공합니다.

1. **이름 기반 자동 프록시 (`BeanNameAutoProxyCreator`):** 빈의 **이름**을 기준으로 프록시 대상을 결정합니다.
2. **어드바이저 기반 자동 프록시 (`DefaultAdvisorAutoProxyCreator`):** 컨텍스트 내의 모든 **어드바이저(Advisor)** 를 찾아서, 각 어드바이저의 **포인트컷**에 매칭되는 빈들을 프록시 대상으로 결정합니다.

---

**첫 번째 파트: `BeanNameAutoProxyCreator`**

- **역할:** 빈 **이름(ID)** 이 **지정된 패턴과 일치**하는 빈들에 대해 **자동으로 AOP 프록시를 생성**하는 `BeanPostProcessor`입니다.
- **사용법:** XML 설정 파일에 `org.springframework.aop.framework.autoproxy.BeanNameAutoProxyCreator` 타입의 빈을 정의하고 다음 속성들을 설정합니다.
  - **`beanNames` (필수):** 프록시를 생성할 대상 빈들의 **이름 목록 또는 이름 패턴**(와일드카드  사용 가능)을 지정합니다. (쉼표, 공백, 세미콜론으로 구분 가능)
  - **`interceptorNames` (필수):** `beanNames`에 매칭된 빈들에게 **적용할 어드바이스/인터셉터/어드바이저 빈들의 이름**을 순서대로 지정합니다. (`ProxyFactoryBean`과 동일)
- **예시:**

    ```xml
    <!-- BeanNameAutoProxyCreator 빈 정의 -->
    <bean class="org.springframework.aop.framework.autoproxy.BeanNameAutoProxyCreator">
        <!-- 이름이 "jdk"로 시작하거나 "onlyJdk"인 빈들을 프록시 대상으로 지정 -->
        <property name="beanNames" value="jdk*,onlyJdk"/>
        <!-- 이 빈들에게 "myInterceptor" 라는 이름의 어드바이스/어드바이저 빈을 적용 -->
        <property name="interceptorNames">
            <list>
                <value>myInterceptor</value>
            </list>
        </property>
    </bean>
    
    <!-- 어드바이스/어드바이저 빈 정의 -->
    <bean id="myInterceptor" class="com.example.MyLoggingInterceptor"/>
    
    <!-- 대상 빈 정의 -->
    <bean id="jdkMyService" class="com.example.MyService"/> <!-- 프록시 대상 (이름 패턴 "jdk*"과 일치) -->
    <bean id="anotherService" class="com.example.AnotherService"/> <!-- 프록시 대상 아님 -->
    <bean id="onlyJdk" class="com.example.SpecialService"/> <!-- 프록시 대상 (이름 "onlyJdk"과 일치) -->
    ```

- **동작:** 스프링 컨테이너가 로드될 때, `BeanNameAutoProxyCreator`는 `beanNames` 속성에 지정된 패턴("jdk*", "onlyJdk")과 이름이 일치하는 빈들(`jdkMyService`, `onlyJdk`)을 찾습니다. 그리고 해당 빈들이 생성된 후, `interceptorNames`에 지정된 어드바이스(`myInterceptor`)가 적용된 AOP 프록시 객체로 **자동으로 교체**합니다. `anotherService` 빈은 이름이 매칭되지 않으므로 프록시되지 않습니다.
- **장점:** 간단한 명명 규칙을 통해 여러 빈에 동일한 AOP 설정을 일괄 적용하기 쉽습니다.
- **단점:** 빈 이름에 의존하므로 유연성이 떨어질 수 있습니다. 포인트컷을 통한 세밀한 대상 지정이 어렵습니다.

---

**두 번째 파트: `DefaultAdvisorAutoProxyCreator` (더 일반적이고 강력)**

- **역할:** 스프링 컨테이너 내에 정의된 **모든 어드바이저(Advisor) 빈들을 자동으로 찾아서**, 각 어드바이저의 **포인트컷(Pointcut)** 이 매칭되는 **모든 빈**에 대해 **자동으로 AOP 프록시를 생성**하고 해당 어드바이스를 적용하는 매우 강력한 `BeanPostProcessor`입니다.
- **사용법:** 매우 간단합니다. XML 설정 파일에 `org.springframework.aop.framework.autoproxy.DefaultAdvisorAutoProxyCreator` 타입의 **빈 하나만 정의**하면 됩니다. 별도의 속성 설정은 거의 필요 없습니다.

    ```xml
    <!-- DefaultAdvisorAutoProxyCreator 빈 정의 (이것 하나로 자동 프록시 활성화) -->
    <bean class="org.springframework.aop.framework.autoproxy.DefaultAdvisorAutoProxyCreator"/>
    
    <!-- 적용될 어드바이저 빈들 정의 (포인트컷 + 어드바이스) -->
    <bean id="transactionAdvisor" class="org.springframework.transaction.interceptor.TransactionAttributeSourceAdvisor">
        <property name="transactionInterceptor" ref="txInterceptor"/>
        <!-- 포인트컷은 보통 내장되어 있거나 (예: @Transactional 검색) 명시적으로 설정 -->
    </bean>
    
    <bean id="myCustomAdvisor" class="com.example.MyCustomAdvisor">
        <!-- 이 어드바이저의 포인트컷과 어드바이스 설정 -->
    </bean>
    
    <!-- 대상 빈들 정의 -->
    <bean id="userService" class="com.example.UserServiceImpl"/>
    <bean id="orderService" class="com.example.OrderServiceImpl"/>
    <bean id="reportService" class="com.example.ReportService"/> <!-- AOP 적용 안 될 수도 있음 -->
    
    <!-- 어드바이저에서 참조하는 어드바이스 빈 등 정의 -->
    <bean id="txInterceptor" class="org.springframework.transaction.interceptor.TransactionInterceptor"> ... </bean>
    ```

- **동작:**
  1. 컨테이너 로드 시 `DefaultAdvisorAutoProxyCreator`가 활성화됩니다.
  2. 컨테이너는 모든 빈 생성을 시도합니다.
  3. 각 빈이 생성된 후, `DefaultAdvisorAutoProxyCreator`는 컨테이너 내에 등록된 **모든 `Advisor` 빈**들을 가져옵니다 (예: `transactionAdvisor`, `myCustomAdvisor`).
  4. 각 `Advisor`의 **포인트컷**을 **현재 생성된 빈**에 대해 **평가(evaluate)** 합니다.
  5. 만약 **하나 이상의 `Advisor`의 포인트컷이 현재 빈의 메소드와 매칭**된다면, `DefaultAdvisorAutoProxyCreator`는 해당 빈에 대한 **AOP 프록시를 생성**하고, 매칭된 모든 `Advisor`의 어드바이스를 프록시에 적용합니다.
  6. 만약 **어떤 `Advisor`의 포인트컷도 현재 빈과 매칭되지 않는다면**, 프록시가 생성되지 않고 원본 빈 객체가 그대로 사용됩니다. (예: `reportService`)
- **장점:**
  - **매우 유연함:** 빈 이름이 아닌, **포인트컷 규칙**에 따라 AOP 적용 대상을 결정하므로 훨씬 유연하고 강력합니다.
  - **설정 간결화:** 어떤 빈을 프록시할지 일일이 지정할 필요 없이, 어드바이저만 정의해두면 알아서 적용됩니다. 새로운 빈을 추가해도 포인트컷에 해당하면 자동으로 프록시됩니다.
  - **부가 기능 추가 용이:** 새로운 AOP 기능(예: 로깅, 모니터링)을 추가하고 싶으면 해당 기능을 수행하는 새로운 `Advisor` 빈만 등록하면 됩니다. 기존 설정 변경이 최소화됩니다.
- **필터링 및 순서 지정:**
  - `DefaultAdvisorAutoProxyCreator`도 특정 이름 패턴의 어드바이저만 사용하도록 필터링하는 기능을 지원합니다 (여러 개의 `DefaultAdvisorAutoProxyCreator`를 사용할 경우).
  - 적용되는 어드바이저들의 순서는 각 `Advisor` 빈이 `Ordered` 인터페이스를 구현하거나 `@Order` 어노테이션을 사용하여 지정할 수 있습니다.

**결론:**

스프링의 "자동 프록시" 기능은 `ProxyFactoryBean`을 직접 사용하는 것보다 훨씬 간편하게 AOP를 적용할 수 있게 해줍니다. `BeanNameAutoProxyCreator`는 빈 이름을 기준으로 대상을 찾고, `DefaultAdvisorAutoProxyCreator`는 컨테이너 내의 모든 어드바이저와 포인트컷을 기반으로 대상을 자동으로 찾아 프록시를 생성합니다. **`DefaultAdvisorAutoProxyCreator` 방식이 훨씬 강력하고 유연**하며, 대부분의 현대적인 스프링 애플리케이션에서 선호되는 자동 프록시 방식입니다. (`@EnableAspectJAutoProxy`나 `<aop:aspectj-autoproxy/>`도 내부적으로 이와 유사한 메커니즘을 사용합니다.)
