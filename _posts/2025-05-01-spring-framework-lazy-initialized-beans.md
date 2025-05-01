---
title: Spring Framework Lazy Initialized Beans
description: 
author: laze
date: 2025-05-01 00:00:05 +0900
categories: [Dev, SpringBoot]
tags: [SpringBoot]
---
# **Lazy-initialized Beans**

기본적으로, `ApplicationContext` 구현체들은 초기화 프로세스의 일부로서 모든 싱글톤 빈을 즉시(eagerly) 생성하고 설정합니다.

일반적으로 이러한 사전 인스턴스화(pre-instantiation)는 바람직한데, 설정이나 주변 환경의 오류가 몇 시간 또는 며칠 후가 아닌 즉시 발견되기 때문입니다.

이 동작이 바람직하지 않을 때, 빈 정의를 지연 초기화(lazy-initialized)되도록 표시하여 싱글톤 빈의 사전 인스턴스화를 방지할 수 있습니다.

지연 초기화된 빈은 IoC 컨테이너에게 시작 시점이 아닌, 처음 요청될 때 빈 인스턴스를 생성하도록 지시합니다.

이 동작은 `@Lazy` 어노테이션 또는 XML에서 `<bean/>` 요소의 `lazy-init` 속성에 의해 제어됩니다. 다음 예제를 참조하십시오:

**Java**

```java
@Bean
@Lazy
ExpensiveToCreateBean lazy() {
	return new ExpensiveToCreateBean();
}

@Bean
AnotherBean notLazy() {
	return new AnotherBean();
}
```

**Kotlin**

```kotlin
@Bean
@Lazy
fun lazy(): ExpensiveToCreateBean {
    return ExpensiveToCreateBean()
}

@Bean
fun notLazy(): AnotherBean {
    return AnotherBean()
}
```

**Xml**

```xml
<bean id="lazy" class="ExpensiveToCreateBean" lazy-init="true"/>
<bean name="notLazy" class="AnotherBean"/>
```

앞의 구성이 `ApplicationContext`에 의해 소비(consumed)될 때, `lazy` 빈은 `ApplicationContext`가 시작될 때 즉시 사전 인스턴스화되지 않는 반면, `notLazy` 빈은 즉시 사전 인스턴스화됩니다.

그러나 지연 초기화된 빈이 지연 초기화되지 않은 싱글톤 빈의 의존성일 경우, `ApplicationContext`는 싱글톤의 의존성을 만족시켜야 하므로 시작 시점에 지연 초기화된 빈을 생성합니다.

지연 초기화된 빈은 지연 초기화되지 않은 다른 곳의 싱글톤 빈에 주입됩니다.

또한 `@Configuration` 어노테이션이 달린 클래스에 `@Lazy` 어노테이션을 사용하거나 XML에서 `<beans/>` 요소의 `default-lazy-init` 속성을 사용하여 빈 집합에 대한 지연 초기화를 제어할 수도 있습니다.

**Java**

```java
@Configuration
@Lazy
public class LazyConfiguration {
	// 어떤 빈도 사전 인스턴스화되지 않음
}

```

**Kotlin**

```kotlin
@Configuration
@Lazy
class LazyConfiguration {
    // 어떤 빈도 사전 인스턴스화되지 않음
}

```

**Xml**
