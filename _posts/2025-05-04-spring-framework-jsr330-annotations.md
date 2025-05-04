---
title: Spring Framework JSR 330 Standard Annotations
description: 
author: laze
date: 2025-05-04 00:00:05 +0900
categories: [Dev, SpringBoot]
tags: [SpringBoot]
---
**Using JSR 330 Standard Annotations**

스프링은 JSR-330 표준 어노테이션 (의존성 주입)에 대한 지원을 제공합니다.

해당 어노테이션들은 스프링 어노테이션과 동일한 방식으로 스캔됩니다. 이를 사용하려면 클래스패스에 관련 jar 파일들이 있어야 합니다.

Maven을 사용하는 경우, `jakarta.inject` 아티팩트는 표준 Maven 저장소 ( https://repo.maven.apache.org/maven2/jakarta/inject/jakarta.inject-api/2.0.0/ )에서 사용할 수 있습니다.

`pom.xml` 파일에 다음 의존성을 추가할 수 있습니다:

```xml
<dependency>
	<groupId>jakarta.inject</groupId>
	<artifactId>jakarta.inject-api</artifactId>
	<version>2.0.0</version>
</dependency>
```

**@Inject 및 @Named를 사용한 의존성 주입**

`@Autowired` 대신 다음과 같이 `@jakarta.inject.Inject`를 사용할 수 있습니다:

```java
// Java
import jakarta.inject.Inject;

public class SimpleMovieLister {

	private MovieFinder movieFinder;

	@Inject
	public void setMovieFinder(MovieFinder movieFinder) {
		this.movieFinder = movieFinder;
	}

	public void listMovies() {
		this.movieFinder.findMovies(...);
		// ...
	}
}
```

```kotlin
// Kotlin
import jakarta.inject.Inject

class SimpleMovieLister {
    private var movieFinder: MovieFinder? = null

    @Inject
    fun setMovieFinder(movieFinder: MovieFinder) {
        this.movieFinder = movieFinder
    }

    fun listMovies() {
        this.movieFinder?.findMovies(...) // Use safe call
        // ...
    }
}
```

`@Autowired`와 마찬가지로, `@Inject`를 필드 레벨, 메소드 레벨 및 생성자 인수 레벨에서 사용할 수 있습니다.

주입 지점을 `Provider`로 선언하여 더 짧은 스코프의 빈에 대한 온디맨드(on-demand) 접근 또는 `Provider.get()` 호출을 통해 다른 빈에 대한 지연 접근(lazy access)을 허용할 수 있습니다.

```java
// Java
import jakarta.inject.Inject;
import jakarta.inject.Provider;

public class SimpleMovieLister {

	private Provider<MovieFinder> movieFinder;

	@Inject
	public void setMovieFinder(Provider<MovieFinder> movieFinder) {
		this.movieFinder = movieFinder;
	}

	public void listMovies() {
		this.movieFinder.get().findMovies(...);
		// ...
	}
}
```

```kotlin
// Kotlin
import jakarta.inject.Inject
import jakarta.inject.Provider

class SimpleMovieLister {
    private var movieFinder: Provider<MovieFinder>? = null

    @Inject
    fun setMovieFinder(movieFinder: Provider<MovieFinder>) {
        this.movieFinder = movieFinder
    }

    fun listMovies() {
        this.movieFinder?.get()?.findMovies(...) // Use safe calls
        // ...
    }
}
```

주입되어야 할 의존성에 대해 정규화된(qualified) 이름을 사용하려면, 다음 예제와 같이 `@Named` 어노테이션을 사용해야 합니다:

```java
// Java
import jakarta.inject.Inject;
import jakarta.inject.Named;

public class SimpleMovieLister {

	private MovieFinder movieFinder;

	@Inject
	public void setMovieFinder(@Named("main") MovieFinder movieFinder) {
		this.movieFinder = movieFinder;
	}

	// ...
}
```

```kotlin
// Kotlin
import jakarta.inject.Inject
import jakarta.inject.Named

class SimpleMovieLister {
    private var movieFinder: MovieFinder? = null

    @Inject
    fun setMovieFinder(@Named("main") movieFinder: MovieFinder) {
        this.movieFinder = movieFinder
    }

    // ...
}
```

`@Autowired`와 마찬가지로, `@Inject`는 `java.util.Optional` 또는 `@Nullable`과 함께 사용될 수도 있습니다.

`@Inject`에는 `required` 속성이 없으므로 여기서 훨씬 더 적용 가능합니다. 다음 한 쌍의 예제는 `@Inject`와 `@Nullable`을 사용하는 방법을 보여줍니다:

```java
public class SimpleMovieLister {

	@Inject
	public void setMovieFinder(Optional<MovieFinder> movieFinder) {
		// ...
	}
}
```

```java
// Java
public class SimpleMovieLister {

	@Inject
	public void setMovieFinder(@Nullable MovieFinder movieFinder) {
		// ...
	}
}
```

```kotlin
// Kotlin
class SimpleMovieLister {
    @Inject
    fun setMovieFinder(movieFinder: MovieFinder?) { // Nullable type implies optionality
        // ...
    }
}
```

**@Named 및 @ManagedBean: @Component 어노테이션의 표준 등가물**

`@Component` 대신 다음과 같이 `@jakarta.inject.Named` 또는 `jakarta.annotation.ManagedBean`을 사용할 수 있습니다:

```java
// Java
import jakarta.inject.Inject;
import jakarta.inject.Named;

@Named("movieListener")  // @ManagedBean("movieListener")을 사용할 수도 있음
public class SimpleMovieLister {

	private MovieFinder movieFinder;

	@Inject
	public void setMovieFinder(MovieFinder movieFinder) {
		this.movieFinder = movieFinder;
	}

	// ...
}
```

```kotlin
// Kotlin
import jakarta.inject.Inject
import jakarta.inject.Named

@Named("movieListener") // @ManagedBean("movieListener")을 사용할 수도 있음
class SimpleMovieLister {
    private var movieFinder: MovieFinder? = null

    @Inject
    fun setMovieFinder(movieFinder: MovieFinder) {
        this.movieFinder = movieFinder
    }

    // ...
}
```

컴포넌트 이름을 지정하지 않고 `@Component`를 사용하는 것이 매우 일반적입니다. 다음 예제와 같이 `@Named`도 유사한 방식으로 사용할 수 있습니다:

```java
// Java
import jakarta.inject.Inject;
import jakarta.inject.Named;

@Named // No name specified, default name will be generated
public class SimpleMovieLister {

	private MovieFinder movieFinder;

	@Inject
	public void setMovieFinder(MovieFinder movieFinder) {
		this.movieFinder = movieFinder;
	}

	// ...
}
```

```kotlin
// Kotlin
import jakarta.inject.Inject
import jakarta.inject.Named

@Named // No name specified, default name will be generated
class SimpleMovieLister {
    private var movieFinder: MovieFinder? = null

    @Inject
    fun setMovieFinder(movieFinder: MovieFinder) {
        this.movieFinder = movieFinder
    }

    // ...
}
```

`@Named` 또는 `@ManagedBean`을 사용할 때, 다음 예제와 같이 스프링 어노테이션을 사용할 때와 정확히 동일한 방식으로 컴포넌트 스캔을 사용할 수 있습니다:

```java
// Java
@Configuration
@ComponentScan(basePackages = "org.example")
public class AppConfig  {
	// ...
}
```

```kotlin
// Kotlin
@Configuration
@ComponentScan(basePackages = ["org.example"])
class AppConfig {
    // ...
}
```

`*@Component`와 대조적으로, JSR-330 `@Named` 및 JSR-250 `@ManagedBean` 어노테이션은 조합 가능(composable)하지 않습니다.*

*커스텀 컴포넌트 어노테이션을 빌드하려면 스프링의 스테레오타입 모델을 사용해야 합니다.*

**JSR-330 표준 어노테이션의 한계**

표준 어노테이션으로 작업할 때, 다음 표에서 보여주듯이 일부 중요한 기능을 사용할 수 없다는 것을 알아야 합니다:

표 1. 스프링 컴포넌트 모델 요소 대 JSR-330 변형

| 스프링 | `jakarta.inject.*` | `jakarta.inject` 제한 사항 / 주석 |
| --- | --- | --- |
| `@Autowired` | `@Inject` | `@Inject`에는 'required' 속성이 없습니다. 대신 Java 8의 `Optional`과 함께 사용할 수 있습니다. |
| `@Component` | `@Named` / `@ManagedBean` | JSR-330은 조합 가능한 모델을 제공하지 않으며, 단지 이름 붙여진 컴포넌트를 식별하는 방법만 제공합니다. |
| `@Scope("singleton")` | `@Singleton` | JSR-330 기본 스코프는 스프링의 프로토타입과 같습니다. 그러나 스프링의 일반적인 기본값과 일관성을 유지하기 위해, 스프링 컨테이너에 선언된 JSR-330 빈은 기본적으로 싱글톤입니다. 싱글톤 외의 스코프를 사용하려면 스프링의 `@Scope` 어노테이션을 사용해야 합니다. `jakarta.inject`는 또한 `jakarta.inject.Scope` 어노테이션을 제공하지만, 이것은 커스텀 어노테이션 생성을 위해서만 사용되도록 의도되었습니다. |
| `@Qualifier` | `@Qualifier` / `@Named` | `jakarta.inject.Qualifier`는 단지 커스텀 퀄리파이어 빌드를 위한 메타 어노테이션입니다. 구체적인 문자열 퀄리파이어(예: 값이 있는 스프링의 `@Qualifier`)는 `jakarta.inject.Named`를 통해 연관될 수 있습니다. |
| `@Value` | - | 등가물 없음 |
| `@Lazy` | - | 등가물 없음 |
| `ObjectFactory` | `Provider` | `jakarta.inject.Provider`는 스프링의 `ObjectFactory`에 대한 직접적인 대안이며, 더 짧은 `get()` 메소드 이름을 가집니다. 스프링의 `@Autowired` 또는 어노테이션 없는 생성자 및 세터 메소드와 함께 사용할 수도 있습니다. |

---

**전체 주제: JSR-330 표준 어노테이션 사용하기**

이 부분은 `@Autowired`, `@Component` 같은 스프링 어노테이션 대신, 자바 표준 명세(JSR-330)에 정의된 `@Inject`, `@Named` 등의 어노테이션을 스프링에서 어떻게 사용할 수 있는지, 그리고 스프링 어노테이션과의 차이점 및 한계는 무엇인지 설명합니다.

**핵심 아이디어:** 특정 프레임워크(스프링)에 덜 의존적인 코드를 작성하기 위해 자바 표준 어노테이션을 사용해보자!

---

**1. 준비: 의존성 추가**

- JSR-330 표준 어노테이션(`@Inject`, `@Named` 등)을 사용하려면 해당 어노테이션 정의가 포함된 **라이브러리가 필요**합니다. 이는 JDK에 기본 포함되어 있지 않습니다.
- **Maven:** `pom.xml`에 `jakarta.inject:jakarta.inject-api` 의존성을 추가해야 합니다. (버전 명시 필요)

    ```xml
    <dependency>
        <groupId>jakarta.inject</groupId>
        <artifactId>jakarta.inject-api</artifactId>
        <version>2.0.0</version> <!-- 예시 버전 -->
    </dependency>
    ```

- **Gradle:** `build.gradle`에 의존성을 추가합니다.

    ```
    implementation 'jakarta.inject:jakarta.inject-api:2.0.0' // 예시 버전
    ```


---

**2. `@Inject` 사용하기 (`@Autowired` 대안)**

- `@jakarta.inject.Inject` 어노테이션은 스프링의 `@Autowired`와 **거의 동일한 역할**을 합니다. 즉, 의존성을 자동으로 주입하도록 지시합니다.
- **사용 위치:** `@Autowired`와 마찬가지로 필드, 메소드, 생성자 파라미터에 사용할 수 있습니다.

    ```java
    import jakarta.inject.Inject;
    
    public class SimpleMovieLister {
        private MovieFinder movieFinder;
    
        @Inject // @Autowired 대신 사용
        public void setMovieFinder(MovieFinder movieFinder) {
            this.movieFinder = movieFinder;
        }
        // ...
    }
    ```

- **`Provider<T>`를 사용한 지연/온디맨드 주입:** `@Inject`는 JSR-330의 `Provider<T>` 인터페이스와 함께 사용하여, 빈을 **필요한 시점에 `get()` 메소드를 통해 얻어오거나**(지연 로딩), **더 짧은 스코프의 빈(예: request)을 안전하게 주입**받는 데 사용할 수 있습니다. 이는 스프링의 `ObjectFactory<T>`나 `ObjectProvider<T>`와 유사한 역할을 합니다.

    ```java
    import jakarta.inject.Inject;
    import jakarta.inject.Provider;
    
    public class SimpleMovieLister {
        private Provider<MovieFinder> movieFinderProvider; // Provider로 선언
    
        @Inject
        public void setMovieFinder(Provider<MovieFinder> movieFinderProvider) {
            this.movieFinderProvider = movieFinderProvider;
        }
    
        public void listMovies() {
            // 실제로 필요한 시점에 get() 호출하여 빈 얻기
            MovieFinder finder = this.movieFinderProvider.get();
            finder.findMovies(...);
        }
    }
    ```


---

**3. `@Named` 사용하기 (`@Qualifier` 또는 이름 지정 `@Component` 대안)**

`@Inject`도 `@Autowired`처럼 타입 기반으로 주입을 시도하다가 같은 타입의 빈이 여러 개 있으면 모호성 문제가 발생할 수 있습니다. 이때 특정 빈을 지정하기 위해 `@Named`를 사용합니다.

- **`@jakarta.inject.Named`의 역할:**
  1. **퀄리파이어 역할:** 주입 지점에서 `@Inject`와 함께 사용하여 **주입할 빈의 이름(ID)** 을 명시적으로 지정합니다. 스프링의 `@Qualifier("이름")`과 유사하게 동작합니다.

      ```java
      import jakarta.inject.Inject;
      import jakarta.inject.Named;
      
      public class SimpleMovieLister {
          private MovieFinder movieFinder;
      
          @Inject
          public void setMovieFinder(@Named("main") MovieFinder movieFinder) { // "main" 이라는 이름의 빈 주입
              this.movieFinder = movieFinder;
          }
          // ...
      }
      ```

  2. **컴포넌트 이름 지정 역할:** 클래스 레벨에 사용하여 스프링의 `@Component("이름")`처럼 **빈의 이름(ID)을 지정**하는 역할을 합니다. (다음 섹션에서 설명)

---

**4. 선택적 의존성 처리 (`@Inject` + `Optional` / `@Nullable` / Kotlin Null-Safety)**

- `@Inject` 어노테이션 자체에는 `@Autowired`의 `required` 속성 같은 것이 **없습니다.**
- 따라서 의존성이 선택 사항임을 나타내려면, **Java 8의 `Optional<T>`** 이나 **`@Nullable` 어노테이션** 또는 **코틀린의 Nullable 타입 (`T?`)** 을 사용해야 합니다. `@Autowired`에서 설명한 방식과 동일하게 동작합니다.

    ```java
    // Optional 사용
    @Inject
    public void setMovieFinder(Optional<MovieFinder> movieFinder) { ... }
    
    // @Nullable 사용 (javax.annotation 또는 다른 패키지)
    @Inject
    public void setMovieFinder(@Nullable MovieFinder movieFinder) { ... }
    
    // Kotlin Nullable 타입 사용
    @Inject
    fun setMovieFinder(movieFinder: MovieFinder?) { ... }
    ```


---

**5. `@Named` / `@ManagedBean` 사용하기 (`@Component` 대안)**

- 스프링의 `@Component` 어노테이션처럼 클래스를 **스프링 빈으로 등록**하도록 표시하는 표준 어노테이션으로 `@jakarta.inject.Named`와 `@jakarta.annotation.ManagedBean`을 사용할 수 있습니다.
- **사용법:** 클래스 위에 붙입니다. 어노테이션의 `value` 속성으로 빈의 이름(ID)을 지정할 수 있습니다. 이름을 생략하면 스프링의 `@Component`처럼 클래스 이름을 기반으로 기본 이름이 생성됩니다.

    ```java
    import jakarta.inject.Inject;
    import jakarta.inject.Named;
    // import jakarta.annotation.ManagedBean; // @ManagedBean 사용 시
    
    @Named("movieListener") // 빈 이름 "movieListener" 지정. @ManagedBean("movieListener")도 가능
    // @Named // 이름 생략 시 기본 이름 생성 (simpleMovieLister)
    public class SimpleMovieLister {
        private MovieFinder movieFinder;
    
        @Inject
        public void setMovieFinder(MovieFinder movieFinder) {
            this.movieFinder = movieFinder;
        }
        // ...
    }
    ```

- **컴포넌트 스캔:** `@Named`나 `@ManagedBean`이 붙은 클래스도 스프링의 컴포넌트 스캔(`@ComponentScan` 또는 `<context:component-scan>`) 대상에 포함되어 자동으로 빈으로 등록됩니다.

---

**6. JSR-330 표준 어노테이션의 한계점 (스프링 어노테이션 대비)**

표준 어노테이션을 사용하면 특정 프레임워크에 대한 의존성을 줄일 수 있지만, 스프링 고유의 어노테이션이 제공하는 몇 가지 편리한 기능은 사용할 수 없습니다.

- **`@Inject` vs `@Autowired`:** `@Inject`에는 `required` 속성이 없습니다. (선택적 의존성 처리를 위해 `Optional` 등 사용 필요)
- **`@Named`/`@ManagedBean` vs `@Component`:** 표준 어노테이션은 `@Component`처럼 다른 어노테이션과 **조합하여 커스텀 어노테이션을 만드는 기능(Composability)이 없습니다.** 스프링의 `@Service`, `@Repository` 같은 스테레오타입 어노테이션을 만들려면 스프링의 `@Component`를 메타 어노테이션으로 사용해야 합니다.
- **스코프 지정:** JSR-330에는 `@Singleton` 스코프 어노테이션이 있지만, 그 외의 스코프(prototype, request, session 등)를 지정하려면 **스프링의 `@Scope` 어노테이션을 사용해야 합니다.** (JSR-330의 `@Scope`는 커스텀 스코프 어노테이션 정의용 메타 어노테이션)
  - **주의:** JSR-330의 기본 스코프는 스프링의 `prototype`과 유사하지만, 스프링 컨테이너는 호환성을 위해 `@Named`/`@ManagedBean` 빈을 기본적으로 **`singleton`** 으로 취급합니다.
- **퀄리파이어:** JSR-330의 `@Qualifier`는 커스텀 퀄리파이어를 만들기 위한 **메타 어노테이션**일 뿐입니다. 특정 **문자열 값**으로 퀄리파이어를 지정하려면 `@Named`를 사용해야 합니다. 스프링의 `@Qualifier`는 자체적으로 문자열 값을 가질 수 있고 커스텀 퀄리파이어 생성도 지원하여 더 유연합니다.
- **`@Value` 대안 없음:** 외부 프로퍼티 값이나 SpEL 표현식을 주입하는 `@Value`에 해당하는 표준 어노테이션은 없습니다.
- **`@Lazy` 대안 없음:** 빈의 지연 초기화를 지정하는 `@Lazy`에 해당하는 표준 어노테이션은 없습니다.
- **`Provider<T>` vs `ObjectFactory<T>/ObjectProvider<T>`:** JSR-330의 `Provider<T>`는 스프링의 `ObjectFactory<T>`와 유사한 역할을 하지만, 스프링의 `ObjectProvider<T>`가 제공하는 추가적인 편의 메소드(예: `getIfAvailable()`, `getIfUnique()`)는 없습니다.

**요약:**

스프링은 JSR-330 표준 의존성 주입 어노테이션(`@Inject`, `@Named`, `@ManagedBean`, `@Singleton`, `@Provider`)을 지원하여 프레임워크에 덜 종속적인 코드를 작성할 수 있도록 합니다. `@Inject`는 `@Autowired`와 유사하게 동작하고, `@Named`/`@ManagedBean`은 `@Component`와 유사하게 동작하며 컴포넌트 스캔 대상이 됩니다. 하지만 표준 어노테이션은 스프링 고유 어노테이션이 제공하는 일부 편의 기능(예: `required` 속성, 조합 가능성, `@Value`, `@Lazy`, 다양한 스코프 지정)을 지원하지 않는 한계가 있습니다. 따라서 프로젝트의 요구사항과 표준 준수 필요성에 따라 적절히 선택하여 사용하는 것이 중요합니다.
