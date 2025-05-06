---
title: TargetSource Implementations
description: 
author: laze
date: 2025-05-06 00:00:12 +0900
categories: [Dev, SpringBoot]
tags: [SpringBoot]
---
**Using TargetSource Implementations**

스프링은 `org.springframework.aop.TargetSource` 인터페이스로 표현되는 `TargetSource` 개념을 제공합니다.

이 인터페이스는 조인 포인트를 구현하는 "대상 객체(target object)"를 반환하는 책임을 가집니다.

`TargetSource` 구현체는 AOP 프록시가 메소드 호출을 처리할 때마다 대상 인스턴스를 요청받습니다.

스프링 AOP를 사용하는 개발자는 일반적으로 `TargetSource` 구현체와 직접 작업할 필요가 없지만, 이것은 풀링(pooling), 핫 스왑(hot swappable) 및 기타 정교한 대상을 지원하는 강력한 수단을 제공합니다.

예를 들어, 풀링 `TargetSource`는 인스턴스를 관리하기 위해 풀을 사용하여 각 호출마다 다른 대상 인스턴스를 반환할 수 있습니다.

`TargetSource`를 지정하지 않으면, 로컬 객체를 래핑하는 기본 구현이 사용됩니다. 예상대로 각 호출에 대해 동일한 대상이 반환됩니다.

*커스텀 대상 소스를 사용할 때, 대상은 일반적으로 싱글톤 빈 정의 대신 프로토타입이어야 합니다.*

*이는 스프링이 필요할 때 새 대상 인스턴스를 생성할 수 있도록 합니다.*

**핫 스왑 가능한 대상 소스 (Hot-swappable Target Sources)**

`org.springframework.aop.target.HotSwappableTargetSource`는 호출자가 참조를 유지하면서 AOP 프록시의 대상을 전환(switched)할 수 있도록 존재합니다.

대상 소스의 대상을 변경하는 것은 즉시 효력을 발생합니다. `HotSwappableTargetSource`는 스레드 안전(thread-safe)합니다.

다음 예제와 같이 `HotSwappableTargetSource`의 `swap()` 메소드를 사용하여 대상을 변경할 수 있습니다:

```java
// Java
import org.springframework.aop.target.HotSwappableTargetSource;
import org.springframework.beans.factory.BeanFactory;
// Assuming beanFactory is available and contains the "swapper" bean
// Assuming newTarget is the new target object instance

HotSwappableTargetSource swapper = (HotSwappableTargetSource) beanFactory.getBean("swapper");
Object oldTarget = swapper.swap(newTarget); // Swap the target
```

```kotlin
// Kotlin
import org.springframework.aop.target.HotSwappableTargetSource
import org.springframework.beans.factory.BeanFactory
// Assuming beanFactory is available and contains the "swapper" bean
// Assuming newTarget is the new target object instance

val swapper = beanFactory.getBean("swapper") as HotSwappableTargetSource
val oldTarget: Any? = swapper.swap(newTarget) // Swap the target
```

다음 예제는 필요한 XML 정의를 보여줍니다:

```xml
<!-- 초기 대상 빈 -->
<bean id="initialTarget" class="mycompany.OldTarget"/>

<!-- 핫 스왑 가능한 대상 소스 -->
<bean id="swapper" class="org.springframework.aop.target.HotSwappableTargetSource">
	<constructor-arg ref="initialTarget"/> <!-- 초기 대상 설정 -->
</bean>

<!-- 핫 스왑 대상 소스를 사용하는 프록시 빈 -->
<bean id="swappable" class="org.springframework.aop.framework.ProxyFactoryBean">
	<property name="targetSource" ref="swapper"/> <!-- TargetSource로 swapper 사용 -->
	<!-- Optional: Add interceptors if needed -->
	<!-- <property name="interceptorNames" value="..."/> -->
</bean>
```

앞의 `swap()` 호출은 `swappable` 빈의 대상을 변경합니다.

해당 빈에 대한 참조를 보유한 클라이언트는 변경 사항을 인식하지 못하지만 즉시 새 대상을 사용하기 시작합니다.

이 예제는 어드바이스를 추가하지 않지만 (`TargetSource`를 사용하는 데 어드바이스를 추가할 필요는 없음), 모든 `TargetSource`는 임의의 어드바이스와 함께 사용될 수 있습니다.

**풀링 대상 소스 (Pooling Target Sources)**

풀링 대상 소스를 사용하면 상태 비저장 세션 EJB(stateless session EJBs)와 유사한 프로그래밍 모델을 제공하며, 동일한 인스턴스 풀이 유지되고 메소드 호출은 풀의 유휴(free) 객체로 전달됩니다.

스프링 풀링과 SLSB 풀링 간의 중요한 차이점은 스프링 풀링이 모든 POJO에 적용될 수 있다는 것입니다. 일반적인 스프링과 마찬가지로 이 서비스는 비침투적인(non-invasive) 방식으로 적용될 수 있습니다.

스프링은 상당히 효율적인 풀링 구현을 제공하는 Commons Pool 2.2에 대한 지원을 제공합니다.

이 기능을 사용하려면 애플리케이션의 클래스패스에 `commons-pool` Jar가 필요합니다.

다른 풀링 API를 지원하기 위해 `org.springframework.aop.target.AbstractPoolingTargetSource`를 하위 클래스로 만들 수도 있습니다.

다음 목록은 예제 구성을 보여줍니다:

```xml
<!-- 풀링될 대상 빈 (반드시 prototype 스코프여야 함) -->
<bean id="businessObjectTarget" class="com.mycompany.MyBusinessObject"
		scope="prototype"> <!-- ① 프로토타입 스코프 -->
	... 속성 생략
</bean>

<!-- Commons Pool 2 기반의 풀링 대상 소스 -->
<bean id="poolTargetSource" class="org.springframework.aop.target.CommonsPool2TargetSource">
	<property name="targetBeanName" value="businessObjectTarget"/> <!-- 대상 빈 이름 참조 -->
	<property name="maxSize" value="25"/> <!-- 풀 최대 크기 -->
	<!-- 다른 풀링 관련 속성 설정 가능 -->
</bean>

<!-- 풀링 대상 소스를 사용하는 프록시 빈 -->
<bean id="businessObject" class="org.springframework.aop.framework.ProxyFactoryBean">
	<property name="targetSource" ref="poolTargetSource"/> <!-- TargetSource로 poolTargetSource 사용 -->
	<property name="interceptorNames" value="myInterceptor"/> <!-- 필요한 경우 인터셉터 추가 -->
</bean>
```

앞의 예제의 대상 객체(`businessObjectTarget`)는 프로토타입이어야 한다는 점에 유의하십시오.

이는 `PoolingTargetSource` 구현이 필요에 따라 풀을 확장하기 위해 대상의 새 인스턴스를 생성할 수 있도록 합니다.

`maxSize`는 가장 기본적이며 항상 존재함이 보장됩니다.

이 경우 `myInterceptor`는 동일한 IoC 컨텍스트에 정의되어야 하는 인터셉터의 이름입니다.

그러나 풀링을 사용하기 위해 인터셉터를 지정할 필요는 없습니다.

풀링만 원하고 다른 어드바이스는 원하지 않는 경우 `interceptorNames` 속성을 설정하지 마십시오.

인트로덕션(introduction)을 통해 풀의 구성 및 현재 크기에 대한 정보를 노출하는 `org.springframework.aop.target.PoolingConfig` 인터페이스로 모든 풀링된 객체를 캐스팅할 수 있도록 스프링을 구성할 수 있습니다.

다음과 유사한 어드바이저를 정의해야 합니다:

```xml
<bean id="poolConfigAdvisor" class="org.springframework.beans.factory.config.MethodInvokingFactoryBean">
	<property name="targetObject" ref="poolTargetSource"/> <!-- 풀링 대상 소스 빈 참조 -->
	<property name="targetMethod" value="getPoolingConfigMixin"/> <!-- 편의 메소드 호출 -->
</bean>
```

이 어드바이저는 `AbstractPoolingTargetSource` 클래스의 편의 메소드를 호출하여 얻어지므로 `MethodInvokingFactoryBean`을 사용합니다.

이 어드바이저의 이름(여기서는 `poolConfigAdvisor`)은 풀링된 객체를 노출하는 `ProxyFactoryBean`의 인터셉터 이름 목록에 있어야 합니다.

캐스트는 다음과 같이 정의됩니다:

```java
// Java
import org.springframework.aop.target.PoolingConfig;
import org.springframework.beans.factory.BeanFactory;
// Assuming beanFactory contains the "businessObject" proxy bean

PoolingConfig conf = (PoolingConfig) beanFactory.getBean("businessObject");
System.out.println("Max pool size is " + conf.getMaxSize());
```

```kotlin
// Kotlin
import org.springframework.aop.target.PoolingConfig
import org.springframework.beans.factory.BeanFactory
// Assuming beanFactory contains the "businessObject" proxy bean

val conf = beanFactory.getBean("businessObject") as PoolingConfig
println("Max pool size is ${conf.maxSize}")
```

*상태 비저장 서비스 객체 풀링은 일반적으로 필요하지 않습니다.*

*대부분의 상태 비저장 객체는 자연스럽게 스레드 안전하며 인스턴스 풀링은 리소스가 캐시되는 경우 문제가 될 수 있으므로 기본 선택이 되어서는 안 된다고 생각합니다.*

자동 프록시를 사용하여 더 간단한 풀링을 사용할 수 있습니다.

모든 자동 프록시 생성기에서 사용하는 `TargetSource` 구현을 설정할 수 있습니다.

**프로토타입 대상 소스 (Prototype Target Sources)**

"프로토타입" 대상 소스를 설정하는 것은 풀링 `TargetSource`를 설정하는 것과 유사합니다.

이 경우, 모든 메소드 호출 시 대상의 새 인스턴스가 생성됩니다.

최신 JVM에서 새 객체를 생성하는 비용은 높지 않지만, 새 객체를 연결(wire up)하는 비용(IoC 의존성 충족)은 더 비쌀 수 있습니다.

따라서 매우 타당한 이유 없이 이 접근 방식을 사용해서는 안 됩니다.

이를 수행하려면 앞서 보여준 `poolTargetSource` 정의를 다음과 같이 수정할 수 있습니다 (명확성을 위해 이름도 변경함):

```xml
<bean id="prototypeTargetSource" class="org.springframework.aop.target.PrototypeTargetSource">
	<property name="targetBeanName" ref="businessObjectTarget"/> <!-- 대상 빈 이름만 필요 -->
</bean>
```

유일한 속성은 대상 빈의 이름입니다.

일관된 명명을 보장하기 위해 `TargetSource` 구현에서 상속이 사용됩니다.

풀링 대상 소스와 마찬가지로 대상 빈은 프로토타입 빈 정의여야 합니다.

**ThreadLocal 대상 소스 (ThreadLocal Target Sources)**

들어오는 각 요청(즉, 스레드별)마다 객체를 생성해야 하는 경우 `ThreadLocal` 대상 소스가 유용합니다.

`ThreadLocal` 개념은 스레드와 함께 리소스를 투명하게 저장하는 JDK 전체 기능을 제공합니다.

`ThreadLocalTargetSource` 설정은 다음 예제와 같이 다른 유형의 대상 소스에 대해 설명된 것과 거의 동일합니다:

```xml
<bean id="threadlocalTargetSource" class="org.springframework.aop.target.ThreadLocalTargetSource">
	<property name="targetBeanName" value="businessObjectTarget"/> <!-- 각 스레드에 대해 생성될 대상 빈 -->
</bean>
```

`*ThreadLocal` 인스턴스는 다중 스레드 및 다중 클래스로더 환경에서 잘못 사용될 때 심각한 문제(잠재적으로 메모리 누수 발생)를 일으킵니다.*

`*ThreadLocal`을 항상 다른 클래스로 래핑하고 `ThreadLocal` 자체를 직접 사용하지 않는 것(래퍼 클래스 제외)을 고려해야 합니다.*

*또한 스레드 로컬 리소스를 올바르게 설정하고 설정 해제(후자는 `ThreadLocal.remove()` 호출 포함)하는 것을 항상 기억해야 합니다.*

*설정 해제하지 않으면 문제가 발생할 수 있으므로 어떤 경우든 설정 해제를 수행해야 합니다.*

*스프링의 `ThreadLocal` 지원은 이를 대신 수행하며 다른 적절한 처리 코드 없이 `ThreadLocal` 인스턴스를 사용하는 대신 항상 고려해야 합니다.*

---

**전체 주제: TargetSource 구현체 사용하기 (Using TargetSource Implementations)**

이 부분은 AOP 프록시가 내부적으로 대상 객체를 어떻게 가져오는지 제어하는 **`org.springframework.aop.TargetSource` 인터페이스**와 스프링이 제공하는 **다양한 `TargetSource` 구현체들** (핫 스왑, 풀링, 프로토타입, 스레드 로컬 등)을 어떻게 사용하고 설정하는지 설명합니다.

**핵심 아이디어:** AOP 프록시가 메소드 호출을 위임할 **대상 객체를 찾는 방식**을 직접 제어할 수 있다! 이를 통해 단순히 하나의 고정된 객체를 프록시하는 것을 넘어, 동적으로 대상을 바꾸거나 여러 인스턴스를 활용하는 고급 패턴을 구현할 수 있다.

---

**1. `TargetSource` 개념:**

- **인터페이스:** `org.springframework.aop.TargetSource`는 AOP 프록시가 **실제 작업을 위임할 대상 객체 인스턴스를 얻어오는 방법**을 정의하는 인터페이스입니다.
- **역할:** 프록시에 메소드 호출이 들어올 때마다, 프록시는 자신이 가지고 있는 `TargetSource`에게 **"지금 사용할 대상 객체를 내놔!"** 라고 요청합니다 (`getTarget()` 메소드 호출). `TargetSource`는 이 요청에 따라 적절한 대상 객체 인스턴스를 반환합니다.
- **기본 동작:** 개발자가 별도로 `TargetSource`를 지정하지 않으면, 스프링은 **단일 대상 객체를 고정적으로 래핑**하고 항상 **동일한 그 객체**를 반환하는 기본 `TargetSource` 구현체(예: `SingletonTargetSource`)를 사용합니다. 이것이 우리가 일반적으로 사용하는 싱글톤 빈에 대한 프록시 방식입니다.
- **확장성:** 개발자는 스프링이 제공하는 다른 `TargetSource` 구현체를 사용하거나 직접 구현하여, 프록시가 호출 시마다 다른 대상 객체를 사용하도록 만들 수 있습니다. (예: 풀에서 가져오기, 스레드별로 다른 객체 사용하기 등)

---

**2. 핫 스왑 가능한 대상 소스 (`HotSwappableTargetSource`)**

- **개념:** AOP 프록시가 바라보는 **대상 객체를 런타임 중에 동적으로 교체(swap)** 할 수 있게 해주는 `TargetSource` 구현체입니다.
- **동작:**
  1. 처음에는 초기 대상 객체를 참조합니다.
  2. `HotSwappableTargetSource` 빈의 `swap(newTarget)` 메소드를 호출하면, 프록시가 내부적으로 참조하는 대상 객체가 `newTarget`으로 즉시 교체됩니다.
  3. 프록시 객체 자체에 대한 참조를 가지고 있는 클라이언트들은 **아무런 변경 없이** 새로운 대상 객체의 메소드를 호출하게 됩니다. (프록시 참조는 그대로, 내부 대상만 바뀜)
- **사용 시나리오:** 애플리케이션 재시작 없이 특정 빈의 구현을 동적으로 업데이트하거나 변경해야 하는 경우. (주의해서 사용해야 함)
- **설정:**

    ```xml
    <!-- 초기 대상 빈 -->
    <bean id="initialTarget" class="mycompany.OldTarget"/>
    
    <!-- 핫 스왑 TargetSource 빈 (초기 대상 지정) -->
    <bean id="swapper" class="org.springframework.aop.target.HotSwappableTargetSource">
        <constructor-arg ref="initialTarget"/>
    </bean>
    
    <!-- ProxyFactoryBean 설정: target 대신 targetSource 속성 사용 -->
    <bean id="swappable" class="org.springframework.aop.framework.ProxyFactoryBean">
        <property name="targetSource" ref="swapper"/> <!-- ★ targetSource로 swapper 빈 지정 ★ -->
    </bean>
    ```

- **대상 교체 코드:**

    ```java
    HotSwappableTargetSource swapper = (HotSwappableTargetSource) beanFactory.getBean("swapper");
    Object oldTarget = swapper.swap(newTargetObject); // ★ 대상 교체 실행 ★
    ```

- **스레드 안전:** `HotSwappableTargetSource` 자체는 스레드 안전하게 설계되었습니다.

---

**3. 풀링 대상 소스 (Pooling Target Sources)**

- **개념:** 상태 비저장 세션 빈(SLSB)처럼, 미리 생성된 **대상 객체 인스턴스 풀(Pool)** 을 유지하고, 메소드 호출 시 풀에서 **사용 가능한(idle) 객체 하나를 꺼내서** 사용한 후 다시 풀에 반납하는 방식의 `TargetSource` 구현체입니다.
- **동작:**
  1. 미리 정의된 크기만큼 대상 객체 인스턴스들을 생성하여 풀에 보관합니다. (대상 빈은 반드시 **프로토타입 스코프**여야 함)
  2. 프록시에 메소드 호출이 오면, `TargetSource`는 풀에서 유휴 객체를 하나 가져와서 그 객체에게 작업을 위임합니다.
  3. 작업이 끝나면 대상 객체는 다시 풀로 반환됩니다.
- **장점:** 상태 비저장(Stateless) POJO 객체에도 EJB의 풀링과 유사한 모델을 적용할 수 있습니다. 객체 생성 비용이 비싸거나 리소스 관리가 필요할 때 유용할 수 있습니다.
- **구현체:** 스프링은 **Apache Commons Pool 2** 라이브러리를 기반으로 하는 `org.springframework.aop.target.CommonsPool2TargetSource`를 제공합니다. (해당 라이브러리 의존성 필요) 다른 풀링 API를 위한 커스텀 구현도 가능합니다 (`AbstractPoolingTargetSource` 상속).
- **설정:**

    ```xml
    <!-- 풀링될 대상 빈 (반드시 prototype!) -->
    <bean id="businessObjectTarget" class="com.mycompany.MyBusinessObject" scope="prototype"/>
    
    <!-- 풀링 TargetSource 빈 -->
    <bean id="poolTargetSource" class="org.springframework.aop.target.CommonsPool2TargetSource">
        <property name="targetBeanName" value="businessObjectTarget"/> <!-- ★ 프로토타입 대상 빈 이름 지정 ★ -->
        <property name="maxSize" value="25"/> <!-- 풀 최대 크기 -->
        <!-- 기타 풀링 관련 속성 설정 (예: minIdle, maxWait 등) -->
    </bean>
    
    <!-- ProxyFactoryBean 설정 -->
    <bean id="businessObject" class="org.springframework.aop.framework.ProxyFactoryBean">
        <property name="targetSource" ref="poolTargetSource"/> <!-- ★ targetSource로 poolTargetSource 지정 ★ -->
        <!-- 필요시 인터셉터 추가 -->
    </bean>
    ```

- **풀 정보 접근:** `PoolingConfig` 인터페이스를 통해 풀의 현재 상태(크기 등) 정보를 인트로덕션(Introduction) 방식으로 노출할 수 있습니다 (`MethodInvokingFactoryBean`과 어드바이저 설정 필요).
- **주의:** 상태 비저장 객체의 경우, 객체 자체는 스레드 안전하므로 **풀링이 항상 필요한 것은 아닙니다.** 리소스 캐싱 등 특정 이유가 있을 때 고려해야 합니다.

---

**4. 프로토타입 대상 소스 (`PrototypeTargetSource`)**

- **개념:** 프록시에 **메소드 호출이 올 때마다 매번 새로운 대상 객체 인스턴스를 생성**하여 사용하는 `TargetSource` 구현체입니다.
- **동작:**
  1. 프록시 메소드가 호출됩니다.
  2. `PrototypeTargetSource`는 지정된 대상 빈 이름(반드시 **프로토타입 스코프**)으로 스프링 컨테이너에게 **새로운 인스턴스를 요청**합니다 (`getBean()`).
  3. 새로 생성된 인스턴스에게 작업을 위임합니다. (사용한 인스턴스는 보통 바로 버려짐)
- **설정:**

    ```xml
    <bean id="businessObjectTarget" class="com.mycompany.MyBusinessObject" scope="prototype"/>
    
    <bean id="prototypeTargetSource" class="org.springframework.aop.target.PrototypeTargetSource">
        <property name="targetBeanName" ref="businessObjectTarget"/> <!-- ★ 프로토타입 대상 빈 이름 지정 ★ -->
    </bean>
    
    <bean id="businessObject" class="org.springframework.aop.framework.ProxyFactoryBean">
        <property name="targetSource" ref="prototypeTargetSource"/>
    </bean>
    ```

- **주의:** 객체 생성 자체는 빠르지만, **객체 생성 시마다 의존성 주입 등 스프링의 초기화 과정**이 수행되므로 비용이 클 수 있습니다. **매우 특별한 이유가 없다면 사용을 피하는 것이 좋습니다.** (풀링 방식이 더 효율적일 수 있음)

---

**5. 스레드 로컬 대상 소스 (`ThreadLocalTargetSource`)**

- **개념:** **각 스레드(Thread)마다 별도의 대상 객체 인스턴스를 생성하고 유지**하여 사용하는 `TargetSource` 구현체입니다.
- **동작:**
  1. 특정 스레드에서 프록시 메소드가 처음 호출되면, 해당 스레드를 위한 **새로운 대상 객체 인스턴스를 생성**하여 `ThreadLocal` 변수에 저장합니다.
  2. 이후 동일한 스레드에서 다시 호출되면, `ThreadLocal`에 저장된 **그 스레드 전용 객체**를 가져와 사용합니다.
  3. 다른 스레드에서 호출되면, 그 스레드를 위한 또 다른 인스턴스를 생성하거나 가져옵니다.
- **사용 시나리오:** 스레드별로 독립적인 상태를 가져야 하는 객체를 프록시 뒤에서 관리해야 할 때 유용합니다. (예: 스레드별 트랜잭션 컨텍스트, 스레드별 사용자 정보 등. 하지만 스프링에는 보통 더 나은 방법들이 있음)
- **설정:**

    ```xml
    <bean id="businessObjectTarget" class="com.mycompany.MyBusinessObject" scope="prototype"/> <!-- 대상은 프로토타입 -->
    
    <bean id="threadlocalTargetSource" class="org.springframework.aop.target.ThreadLocalTargetSource">
        <property name="targetBeanName" value="businessObjectTarget"/>
    </bean>
    
    <bean id="businessObject" class="org.springframework.aop.framework.ProxyFactoryBean">
        <property name="targetSource" ref="threadlocalTargetSource"/>
    </bean>
    ```

- **주의:** `ThreadLocal`은 잘못 사용하면 **메모리 누수**를 일으킬 수 있습니다. 스레드 풀 환경 등에서 스레드가 재사용될 때 이전 스레드의 `ThreadLocal` 값이 남아있지 않도록 반드시 **명시적으로 제거(`ThreadLocal.remove()`)** 해주는 관리가 필요합니다. (스프링의 `ThreadLocalTargetSource`는 이를 어느 정도 처리해주지만, 개념 자체의 위험성은 인지해야 함)

**요약:**

`TargetSource`는 AOP 프록시가 **대상 객체를 얻는 방식**을 정의하는 인터페이스입니다. 기본적으로는 하나의 싱글톤 객체를 가리키지만, 스프링은 다양한 구현체를 제공하여 고급 기능을 가능하게 합니다. `HotSwappableTargetSource`는 런타임 대상 교체를, `CommonsPool2TargetSource`는 객체 풀링을, `PrototypeTargetSource`는 매 호출 시 새 객체 생성을, `ThreadLocalTargetSource`는 스레드별 객체 생성을 지원합니다. 이러한 `TargetSource`들은 `ProxyFactoryBean`의 `targetSource` 속성을 통해 설정하며, 대상 빈은 보통 프로토타입 스코프로 정의해야 합니다.
