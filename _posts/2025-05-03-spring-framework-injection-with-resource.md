---
title: Spring Framework Injection with @Resource
description: 
author: laze
date: 2025-05-03 00:00:05 +0900
categories: [Dev, SpringBoot]
tags: [SpringBoot]
---
**Injection with @Resource**

스프링은 또한 필드 또는 빈 프로퍼티 세터 메소드에 JSR-250 `@Resource` 어노테이션 (`jakarta.annotation.Resource`)을 사용한 주입도 지원합니다.

이는 Jakarta EE에서 일반적인 패턴입니다: 예를 들어, JSF 관리 빈 및 JAX-WS 엔드포인트에서 사용됩니다. 스프링은 스프링 관리 객체에 대해서도 이 패턴을 지원합니다.

`@Resource`는 `name` 속성을 가집니다. 기본적으로 스프링은 그 값을 주입될 빈의 이름으로 해석합니다. 즉, 다음 예제에서 보여주듯이 이름 기반(by-name) 의미론을 따릅니다:

```java
// Java
public class SimpleMovieLister {

	private MovieFinder movieFinder;

	@Resource(name="myMovieFinder") // ①
	public void setMovieFinder(MovieFinder movieFinder) {
		this.movieFinder = movieFinder;
	}
}
```

① 이 라인은 `@Resource`를 주입합니다.

```kotlin
// Kotlin
class SimpleMovieLister {
    private var movieFinder: MovieFinder? = null

    @Resource(name = "myMovieFinder") // ①
    fun setMovieFinder(movieFinder: MovieFinder) {
        this.movieFinder = movieFinder
    }
}
```

이름이 명시적으로 지정되지 않은 경우, 기본 이름은 필드 이름 또는 세터 메소드 이름에서 파생됩니다. 필드의 경우에는 필드 이름을 사용합니다. 세터 메소드의 경우에는 빈 프로퍼티 이름을 사용합니다. 다음 예제는 `movieFinder`라는 이름의 빈을 해당 세터 메소드에 주입하게 됩니다:

```java
// Java
public class SimpleMovieLister {

	private MovieFinder movieFinder;

	@Resource
	public void setMovieFinder(MovieFinder movieFinder) {
		this.movieFinder = movieFinder;
	}
}
```

```kotlin
// Kotlin
class SimpleMovieLister {
    private var movieFinder: MovieFinder? = null

    @Resource
    fun setMovieFinder(movieFinder: MovieFinder) {
        this.movieFinder = movieFinder
    }
}
```

어노테이션과 함께 제공된 이름은 `CommonAnnotationBeanPostProcessor`가 인식하는 `ApplicationContext`에 의해 빈 이름으로 해석됩니다.

스프링의 `SimpleJndiBeanFactory`를 명시적으로 설정하면 JNDI를 통해 이름을 해석(resolve)할 수 있습니다.

그러나 기본 동작에 의존하고 간접 참조(indirection) 수준을 유지하기 위해 스프링의 JNDI 룩업(lookup) 기능을 사용하는 것을 권장합니다.

명시적인 이름 지정 없이 `@Resource`를 사용하는 **배타적인 경우에만**, 그리고 `@Autowired`와 유사하게, `@Resource`는 특정 이름의 빈 대신 기본 타입 일치(primary type match)를 찾고 잘 알려진 해석 가능한 의존성들을 해석합니다: `BeanFactory`, `ApplicationContext`, `ResourceLoader`, `ApplicationEventPublisher`, `MessageSource` 인터페이스.

따라서 다음 예제에서 `customerPreferenceDao` 필드는 먼저 `customerPreferenceDao`라는 이름의 빈을 찾고, 그 다음 `CustomerPreferenceDao` 타입에 대한 기본 타입 일치로 대체됩니다(falls back).

```java
// Java
public class MovieRecommender {

	@Resource
	private CustomerPreferenceDao customerPreferenceDao;

	@Resource
	private ApplicationContext context; // ①

	public MovieRecommender() {
	}

	// ...
}
```

① `context` 필드는 알려진 해석 가능한 의존성 타입인 `ApplicationContext`를 기반으로 주입됩니다.

```kotlin
// Kotlin
class MovieRecommender {
    @Resource
    private lateinit var customerPreferenceDao: CustomerPreferenceDao

    @Resource
    private lateinit var context: ApplicationContext // ①

    // ...
}
```

---

**전체 주제: `@Resource`를 사용한 주입**

이 부분은 JSR-250 표준 어노테이션인 `@Resource` (`jakarta.annotation.Resource` 또는 이전 버전에서는 `javax.annotation.Resource`)를 사용하여 스프링 빈의 의존성을 주입하는 방법에 대한 내용입니다.

`@Autowired`가 주로 **타입** 기반으로 동작하는 것과 달리, `@Resource`는 주로 **이름** 기반으로 동작하는 특징이 있습니다.

**핵심 아이디어:** "이 필드나 세터에는 '이런 이름'을 가진 빈을 찾아서 연결해줘!" 라고 명시적으로 지시하자.

---

**1. `@Resource` 사용법 및 기본 동작 (이름 기반):**

- **적용 위치:** `@Resource`는 **필드(field)** 또는 **단일 파라미터를 가진 빈 프로퍼티 세터(setter) 메소드**에 붙일 수 있습니다. (`@Autowired`와 달리 생성자나 여러 파라미터를 가진 메소드에는 사용 불가)
- **`name` 속성:** `@Resource` 어노테이션은 `name`이라는 속성을 가지고 있습니다. 이 속성으로 **주입받고 싶은 빈의 이름(ID)** 을 명시적으로 지정할 수 있습니다.

    ```java
    public class SimpleMovieLister {
        private MovieFinder movieFinder;
    
        // ★ "myMovieFinder" 라는 이름의 빈을 찾아서 주입 ★
        @Resource(name="myMovieFinder")
        public void setMovieFinder(MovieFinder movieFinder) {
            this.movieFinder = movieFinder;
        }
    }
    ```

    ```kotlin
    class SimpleMovieLister {
        private var movieFinder: MovieFinder? = null
    
        @Resource(name = "myMovieFinder")
        fun setMovieFinder(movieFinder: MovieFinder) {
            this.movieFinder = movieFinder
        }
    }
    ```

- **이름 미지정 시 기본 동작:** 만약 `name` 속성을 **생략**하면, 스프링(`CommonAnnotationBeanPostProcessor`)은 다음 규칙에 따라 주입할 빈의 이름을 **추론**합니다:
  - **필드에 적용 시:** **필드 이름**과 동일한 이름의 빈을 찾습니다.

      ```java
      @Resource // name 생략
      private MovieFinder movieFinder; // 필드 이름 "movieFinder"와 같은 이름의 빈을 찾음
      ```

  - **세터 메소드에 적용 시:** **세터 메소드 이름에서 파생된 프로퍼티 이름**과 동일한 이름의 빈을 찾습니다. (예: `setMovieFinder` -> 프로퍼티 이름 "movieFinder")

      ```java
      @Resource // name 생략
      public void setMovieFinder(MovieFinder movieFinder) {
          // 프로퍼티 이름 "movieFinder"와 같은 이름의 빈을 찾음
          this.movieFinder = movieFinder;
      }
      
      ```


---

**2. 이름 매칭 실패 시 대체 동작 (Fallback - 타입 기반):**

- **중요:** `@Resource`가 이름으로 빈을 찾으려고 시도했는데, **명시적으로 지정된 이름이나 추론된 이름으로 빈을 찾지 못했을 경우**, `@Resource`는 `@Autowired`처럼 **타입(Type)** 기반으로 매칭을 시도하는 **대체(fallback)** 동작을 수행합니다.
- **동작:**
  1. 먼저 이름으로 빈을 찾는다. (실패)
  2. 주입 지점의 **타입**과 일치하는 빈을 찾는다.
  3. 만약 타입이 일치하는 빈이 **정확히 하나** 있다면, 그 빈을 주입한다.
  4. 만약 타입이 일치하는 빈이 여러 개 있다면, 그 중에서 `@Primary`로 지정된 빈이 있는지 확인하고 있다면 주입한다.
  5. 그래도 결정할 수 없다면 오류가 발생한다.
- **예시:**

    ```java
    public class MovieRecommender {
        // 1. "customerPreferenceDao" 이름의 빈을 찾음 (없다고 가정)
        // 2. CustomerPreferenceDao 타입의 빈을 찾음 (하나 있다고 가정)
        // 3. 타입이 일치하는 그 빈을 주입함
        @Resource
        private CustomerPreferenceDao customerPreferenceDao;
    }
    
    ```


---

**3. 잘 알려진 프레임워크 인터페이스 주입:**

- `@Autowired`와 마찬가지로, `@Resource`를 사용하여 이름 지정 없이도 스프링의 핵심 인터페이스(`ApplicationContext`, `BeanFactory`, `Environment` 등)를 주입받을 수 있습니다. 이때는 타입 기반으로 자동 해석됩니다.

    ```java
    @Resource
    private ApplicationContext context; // ApplicationContext 타입으로 자동 주입됨
    
    ```


---

**4. `@Resource` vs `@Autowired` 요약:**

| 특징 | `@Resource` (JSR-250) | `@Autowired` (Spring) |
| --- | --- | --- |
| **주요 매칭 기준** | **이름 (Name)** | **타입 (Type)** |
| **이름 매칭 실패 시** | **타입** 기반 매칭으로 대체(Fallback) | (해당 없음) |
| **타입 매칭 실패 시** | 오류 발생 | 오류 발생 |
| **타입 후보 여러 개일 때** | `@Primary` 고려 (대체 동작 시) | `@Qualifier`, `@Primary`, `@Fallback`, 이름 매칭 등으로 해결 |
| **적용 가능 위치** | 필드, 단일 파라미터 세터 메소드 | 필드, 생성자, 모든 메소드 (파라미터 개수 무관) |
| **`required` 속성** | 없음 (기본적으로 필수, 타입 대체 실패 시 오류) | 있음 (`required = false`로 선택적 의존성 가능) |
| **표준** | 자바 표준 (JSR-250) | 스프링 고유 |

**언제 `@Resource`를 쓸까?**

- 주입받을 빈을 **이름**으로 명확하게 지정하고 싶을 때.
- 자바 표준 어노테이션을 선호할 때.
- 필드나 간단한 세터 주입만 필요할 때.

**언제 `@Autowired`를 쓸까?**

- **타입** 기반 주입이 더 자연스러울 때 (일반적으로 더 많이 사용됨).
- **생성자 주입**을 사용하고 싶을 때 (권장 방식).
- 여러 후보 중 선택하기 위해 `@Qualifier`나 `@Primary` 등의 세밀한 제어가 필요할 때.
- 선택적 의존성(`required = false`) 처리가 필요할 때.

**결론:**

`@Resource`는 주로 **이름**을 기반으로 의존성을 주입하는 자바 표준 어노테이션입니다. 이름으로 빈을 찾지 못하면 타입 기반으로 대체 동작을 수행합니다. 필드와 단일 파라미터 세터에만 적용 가능하며, `@Autowired`와는 다른 의미론과 적용 범위를 가집니다. 어떤 것을 사용할지는 상황과 선호하는 방식에 따라 선택할 수 있습니다.
