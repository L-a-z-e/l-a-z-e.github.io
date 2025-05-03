---
title: Spring Framework Using Generics as Autowiring Qualifiers
description: 
author: laze
date: 2025-05-03 00:00:03 +0900
categories: [Dev, SpringBoot]
tags: [SpringBoot]
---
**Using Generics as Autowiring Qualifiers**

`@Qualifier` 어노테이션 외에도, 자바 제네릭 타입을 암시적인 형태의 퀄리피케이션(qualification)으로 사용할 수 있습니다.

```java
// Java
@Configuration
public class MyConfiguration {

	@Bean
	public StringStore stringStore() {
		return new StringStore();
	}

	@Bean
	public IntegerStore integerStore() {
		return new IntegerStore();
	}
}
```

```kotlin
// Kotlin
@Configuration
class MyConfiguration {

    @Bean
    fun stringStore(): StringStore {
        return StringStore()
    }

    @Bean
    fun integerStore(): IntegerStore {
        return IntegerStore()
    }
}
```

앞의 빈들이 제네릭 인터페이스(즉, `Store<String>` 및 `Store<Integer>`)를 구현한다고 가정하면, `Store` 인터페이스를 `@Autowire`할 수 있으며 제네릭이 퀄리파이어(qualifier)로 사용됩니다. 다음 예제를 참조하십시오:

```java
// Java
@Autowired
private Store<String> s1; // <String> 퀄리파이어, stringStore 빈 주입

@Autowired
private Store<Integer> s2; // <Integer> 퀄리파이어, integerStore 빈 주입
```

```kotlin
// Kotlin
@Autowired
private lateinit var s1: Store<String> // <String> 퀄리파이어, stringStore 빈 주입

@Autowired
private lateinit var s2: Store<Integer> // <Integer> 퀄리파이어, integerStore 빈 주입
```

제네릭 퀄리파이어는 리스트(lists), `Map` 인스턴스 및 배열을 자동 와이어링할 때도 적용됩니다. 다음 예제는 제네릭 `List`를 자동 와이어링합니다:

```java
// Java
// <Integer> 제네릭을 가진 모든 Store 빈 주입
// Store<String> 빈은 이 리스트에 나타나지 않음
@Autowired
private List<Store<Integer>> s;
```

```kotlin
// Kotlin
// <Integer> 제네릭을 가진 모든 Store 빈 주입
// Store<String> 빈은 이 리스트에 나타나지 않음
@Autowired
private lateinit var s: List<Store<Integer>>
```

---

**전체 주제: 제네릭을 자동 와이어링 퀄리파이어로 사용하기**

이 부분은 같은 기본 인터페이스(raw type)를 구현하지만 **제네릭 타입 파라미터가 다른** 빈들이 여러 개 있을 때, `@Autowired`가 이 **제네릭 타입 정보를 마치 퀄리파이어처럼 사용**하여 정확한 빈을 찾아 주입하는 방법에 대한 내용입니다.

**핵심 아이디어:** 같은 `Store` 인터페이스라도, `Store<String>`과 `Store<Integer>`는 다르다! 주입받을 때 원하는 제네릭 타입을 명시하면 스프링이 알아서 구분해준다!

---

**1. 문제 상황 (비슷하지만 다른):**

- 다음과 같은 설정이 있다고 가정해 봅시다:
  - `Store<T>` 라는 제네릭 인터페이스가 있습니다. (`T`는 타입 파라미터)
  - `StringStore` 클래스는 `Store<String>`을 구현합니다.
  - `IntegerStore` 클래스는 `Store<Integer>`를 구현합니다.
  - 이 두 클래스를 모두 스프링 빈으로 등록했습니다 (`stringStore`, `integerStore`).
  - 어떤 빈에서 `Store<String>` 타입의 빈과 `Store<Integer>` 타입의 빈을 각각 주입받으려고 합니다.
- **만약 제네릭 정보가 없다면?** 그냥 `@Autowired private Store store;` 라고 하면, `stringStore`와 `integerStore` 둘 다 `Store` 타입이므로 여전히 모호성 문제가 발생합니다.

---

**2. 해결책: 제네릭 타입 자체가 퀄리파이어 역할!**

- 스프링의 `@Autowired`는 단순히 기본 타입(`Store`)만 보는 것이 아니라, 주입 지점에 선언된 **제네릭 타입 파라미터(예: `<String>`, `<Integer>`)** 정보까지 확인합니다.
- **사용법:** 주입받으려는 필드나 파라미터의 타입을 원하는 **제네릭 타입으로 정확하게 명시**하면 됩니다. `@Qualifier`를 따로 붙일 필요가 없습니다!

    ```java
    // Java
    @Configuration
    public class MyConfiguration {
        @Bean
        public StringStore stringStore() { // Store<String> 구현체 빈
            return new StringStore();
        }
        @Bean
        public IntegerStore integerStore() { // Store<Integer> 구현체 빈
            return new IntegerStore();
        }
    }
    
    // 다른 빈에서 주입받기
    public class MyService {
        @Autowired
        private Store<String> s1; // ★ <String> 제네릭이 퀄리파이어 역할! ★
    
        @Autowired
        private Store<Integer> s2; // ★ <Integer> 제네릭이 퀄리파이어 역할! ★
    }
    ```

    ```kotlin
    // Kotlin
    @Configuration
    class MyConfiguration {
        @Bean
        fun stringStore(): StringStore { // Store<String> 구현체 빈
            return StringStore()
        }
        @Bean
        fun integerStore(): IntegerStore { // Store<Integer> 구현체 빈
            return IntegerStore()
        }
    }
    
    // 다른 빈에서 주입받기
    class MyService {
        @Autowired
        private lateinit var s1: Store<String> // ★ <String> 제네릭이 퀄리파이어 역할! ★
    
        @Autowired
        private lateinit var s2: Store<Integer> // ★ <Integer> 제네릭이 퀄리파이어 역할! ★
    }
    ```

- **동작 방식:**
  1. `s1` 필드에 주입할 때: 스프링은 `Store<String>` 타입을 찾습니다. 컨테이너 내의 빈들 중 `stringStore` 빈이 `Store<String>` 타입과 일치하므로 (제네릭 정보까지 확인), `stringStore` 빈을 `s1`에 주입합니다. `integerStore` 빈은 `Store<Integer>` 타입이므로 후보에서 제외됩니다.
  2. `s2` 필드에 주입할 때: 스프링은 `Store<Integer>` 타입을 찾습니다. `integerStore` 빈이 `Store<Integer>` 타입과 일치하므로, `integerStore` 빈을 `s2`에 주입합니다. `stringStore` 빈은 `Store<String>` 타입이므로 후보에서 제외됩니다.

---

**3. 컬렉션/배열과 제네릭 퀄리파이어:**

- 이 제네릭 기반의 필터링은 여러 빈을 컬렉션이나 배열로 주입받을 때도 똑같이 적용됩니다.
- **예시:**
  위 코드에서는 `integerStore` 빈만 리스트에 포함되고, `stringStore` 빈은 제네릭 타입이 다르므로 포함되지 않습니다. 제네릭 타입이 컬렉션의 **요소들을 필터링하는 기준**이 됩니다.

    ```java
    // Java
    // Store<Integer> 타입과 일치하는 모든 빈들을 List로 주입받기
    @Autowired
    private List<Store<Integer>> integerStores;
    ```

    ```kotlin
    // Kotlin
    @Autowired
    private lateinit var integerStores: List<Store<Integer>>
    ```


**요약:**

`@Qualifier`를 명시적으로 사용하는 것 외에도, **주입 지점에 선언된 제네릭 타입 파라미터**를 이용하여 `@Autowired`의 대상을 좁힐 수 있습니다. 스프링은 타입 매칭 시 제네릭 정보까지 고려하여, 해당 제네릭 타입을 가진 빈들만 후보로 간주합니다. 이는 코드를 더 타입 안전하고 명확하게 만들어주며, 특히 제네릭 인터페이스나 클래스를 많이 사용하는 경우 매우 유용합니다. 컬렉션이나 배열을 주입받을 때도 제네릭 정보가 필터링 기준으로 작용합니다.
