---
title: Spring Framework Annotation-based Container Configuration
description: 
author: laze
date: 2025-05-03 00:00:02 +0900
categories: [Dev, SpringBoot]
tags: [SpringBoot]
---
**Fine-tuning Annotation-based Autowiring with @Primary or @Fallback**

타입에 의한 자동 와이어링은 여러 후보를 발생시킬 수 있으므로, 선택 프로세스에 대한 더 많은 제어가 필요한 경우가 종종 있습니다.

이를 달성하는 한 가지 방법은 스프링의 `@Primary` 어노테이션을 사용하는 것입니다. `@Primary`는 여러 빈이 단일 값 의존성에 자동 와이어링될 후보일 때 특정 빈에 우선권이 주어져야 함을 나타냅니다.

후보들 중에 정확히 하나의 기본(primary) 빈이 존재하면, 그것이 자동 와이어링된 값이 됩니다.

`firstMovieCatalog`를 기본 `MovieCatalog`로 정의하는 경우

```java
// Java
@Configuration
public class MovieConfiguration {

	@Bean
	@Primary
	public MovieCatalog firstMovieCatalog() { ... }

	@Bean
	public MovieCatalog secondMovieCatalog() { ... }

	// ...
}
```

```kotlin
// Kotlin
@Configuration
class MovieConfiguration {

    @Bean
    @Primary
    fun firstMovieCatalog(): MovieCatalog { ... }

    @Bean
    fun secondMovieCatalog(): MovieCatalog { ... }

    // ...
}
```

또는, 6.2 버전부터 주입될 일반적인 빈 외의 다른 빈들을 구분하기 위한 `@Fallback` 어노테이션이 있습니다. 만약 하나의 일반 빈만 남는다면, 그것 또한 효과적으로 기본(primary)이 됩니다:

```java
// Java
@Configuration
public class MovieConfiguration {

	@Bean
	public MovieCatalog firstMovieCatalog() { ... }

	@Bean
	@Fallback
	public MovieCatalog secondMovieCatalog() { ... }

	// ...
}
```

```kotlin
// Kotlin
@Configuration
class MovieConfiguration {

    @Bean
    fun firstMovieCatalog(): MovieCatalog { ... }

    @Bean
    @Fallback
    fun secondMovieCatalog(): MovieCatalog { ... }

    // ...
}
```

앞의 구성의 두 변형 모두에서, 다음 `MovieRecommender`는 `firstMovieCatalog`로 자동 와이어링됩니다:

```java
// Java
public class MovieRecommender {

	@Autowired
	private MovieCatalog movieCatalog;

	// ...
}
```

```kotlin
// Kotlin
class MovieRecommender {
    @Autowired
    private lateinit var movieCatalog: MovieCatalog

    // ...
}
```

해당 빈 정의는 다음과 같습니다:

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

	<bean class="example.SimpleMovieCatalog" primary="true"> <!-- ① -->
		<!-- 이 빈에 필요한 모든 의존성 주입 -->
	</bean>

	<bean class="example.SimpleMovieCatalog">
		<!-- 이 빈에 필요한 모든 의존성 주입 -->
	</bean>

	<bean id="movieRecommender" class="example.MovieRecommender"/>

</beans>

```

① `primary` 속성을 `true`로 설정하여 기본 빈으로 지정

---

**전체 주제: 어노테이션 기반 자동 와이어링 세부 조정 (`@Primary`, `@Fallback`)**

이 부분은 `@Autowired`가 **타입**으로 빈을 찾다 보니, 같은 타입의 후보 빈이 여러 개 발견될 경우 **"그래서 뭘 주입해야 하는데?"** 하고 혼란에 빠지는 문제를 어떻게 해결하는지에 대한 내용입니다.

**핵심 아이디어:** 여러 후보 빈 중에서 어떤 놈이 "진짜 주인공(Primary)" 또는 "예비 선수(Fallback)"인지 표시해서 스프링의 선택을 도와주자!

---

**1. 문제 상황: 모호한 의존성 (Ambiguous Dependency)**

- 다음과 같이 설정되어 있다고 가정해 봅시다:
  - `MovieCatalog` 인터페이스가 있습니다.
  - `FirstMovieCatalogImpl` 클래스와 `SecondMovieCatalogImpl` 클래스가 모두 `MovieCatalog` 인터페이스를 구현합니다.
  - 이 두 클래스를 모두 스프링 빈으로 등록했습니다.
  - 어떤 다른 빈(`MovieRecommender`)에서 `@Autowired private MovieCatalog movieCatalog;` 처럼 `MovieCatalog` 타입의 빈을 주입받으려고 합니다.
- **문제:** `MovieRecommender`는 `MovieCatalog` 타입의 빈을 원하는데, 스프링 컨테이너에는 `FirstMovieCatalogImpl`과 `SecondMovieCatalogImpl`이라는 두 개의 후보가 있습니다. 스프링은 둘 중 **어떤 것을 주입해야 할지 결정할 수 없어서** 오류를 발생시킵니다. (NoUniqueBeanDefinitionException)

---

**2. 해결책 (1): `@Primary` - "내가 주인공이야!"**

- **개념:** 여러 후보 빈 중에서 **"이 빈이 기본(기본값, 대표)으로 사용될 빈이야!"** 라고 명확하게 표시해주는 어노테이션입니다.
- **사용법:** 우선적으로 선택되기를 원하는 빈의 정의 부분(클래스 레벨 또는 `@Bean` 메소드)에 `@Primary` 어노테이션을 붙여줍니다.
- **동작 방식:**
  - `@Autowired`가 `MovieCatalog` 타입의 빈을 찾습니다. 후보로 `firstMovieCatalog`와 `secondMovieCatalog` 두 개를 발견합니다.
  - 스프링은 후보들 중에 `@Primary`가 붙은 빈이 있는지 확인합니다.
  - `firstMovieCatalog`에 `@Primary`가 붙어 있으므로, 스프링은 **"아하! 얘가 대표구나!"** 라고 판단하고 `firstMovieCatalog`를 `MovieRecommender`에 주입합니다.
  - 만약 후보 중에 `@Primary`가 붙은 빈이 **오직 하나만** 있다면, 그 빈이 최종 선택됩니다. (만약 여러 개에 `@Primary`가 붙어 있다면 또다시 모호성 오류 발생)
- **예시 (Java Config):**

    ```java
    @Configuration
    public class MovieConfiguration {
        @Bean
        @Primary // ★ 얘가 대표! ★
        public MovieCatalog firstMovieCatalog() { /* ... */ }
    
        @Bean
        public MovieCatalog secondMovieCatalog() { /* ... */ }
    }
    ```

- **예시 (XML Config):**

    ```xml
    <bean class="example.SimpleMovieCatalog" primary="true"> <!-- ★ primary 속성 사용 ★ -->
        <!-- ... -->
    </bean>
    <bean class="example.SimpleMovieCatalog">
        <!-- ... -->
    </bean>
    ```


---

**3. 해결책 (2): `@Fallback` - "나는 예비 선수야!" (Spring 6.2+)**

- **개념:** `@Primary`와 반대되는 개념으로, **"이 빈은 다른 일반적인 후보가 없을 때만 고려될 예비(대체) 빈이야!"** 라고 표시하는 어노테이션입니다.
- **사용법:** 예비 후보로 취급되기를 원하는 빈의 정의 부분에 `@Fallback` 어노테이션을 붙여줍니다.
- **동작 방식:**
  - `@Autowired`가 `MovieCatalog` 타입의 빈을 찾습니다. 후보로 `firstMovieCatalog`와 `secondMovieCatalog` 두 개를 발견합니다.
  - 스프링은 후보들 중에 `@Fallback`이 붙은 빈을 확인하고, 이 빈들은 **일단 제외**합니다.
  - `secondMovieCatalog`가 `@Fallback`이므로 제외되고, 남은 일반 후보는 `firstMovieCatalog` **하나**입니다.
  - 따라서 스프링은 남은 유일한 일반 후보인 `firstMovieCatalog`를 주입합니다. (결과적으로 `@Primary`와 동일한 효과를 내지만, 의도가 다름)
  - 만약 모든 후보가 `@Fallback`이거나, `@Fallback`이 아닌 일반 후보가 여러 개 남으면 여전히 모호성 오류가 발생할 수 있습니다.
- **예시 (Java Config):**

    ```java
    @Configuration
    public class MovieConfiguration {
        @Bean
        public MovieCatalog firstMovieCatalog() { /* ... */ } // 일반 후보
    
        @Bean
        @Fallback // ★ 나는 예비! ★
        public MovieCatalog secondMovieCatalog() { /* ... */ }
    }
    ```


---

**4. `@Primary` vs `@Fallback`**

- **`@Primary`:** "내가 기본값이야!" (적극적)
- **`@Fallback`:** "다른 애들 없으면 나를 써." (소극적)
- 둘 다 결과적으로 하나의 빈을 선택하게 만들 수 있지만, 설정의 의도를 다르게 표현합니다. 일반적으로 `@Primary`가 더 직관적이고 많이 사용됩니다. `@Fallback`은 특정 시나리오(예: 테스트용 대체 빈)에서 유용할 수 있습니다.

**결론:**

`@Autowired` 사용 시 같은 타입의 빈이 여러 개 있어서 발생하는 모호성 문제를 해결하기 위해 `@Primary` 또는 `@Fallback` 어노테이션을 사용합니다. `@Primary`는 여러 후보 중 우선적으로 선택될 **대표 빈**을 지정하는 방식이고, `@Fallback`은 다른 일반 후보가 없을 때만 고려될 **예비 빈**을 지정하는 방식입니다. 이를 통해 스프링 컨테이너가 정확히 어떤 빈을 주입해야 할지 명확하게 알려줄 수 있습니다. XML 설정에서는 `<bean primary="true">` 속성을 사용합니다.
