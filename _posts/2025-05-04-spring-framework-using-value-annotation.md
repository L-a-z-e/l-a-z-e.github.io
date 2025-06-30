---
title: Spring Framework Using @Value
description: 
author: laze
date: 2025-05-04 00:00:01 +0900
categories: [Dev, SpringBoot]
tags: [SpringBoot]
---
**Using @Value**

`@Value`는 일반적으로 외부화된(externalized) 속성을 주입하는 데 사용됩니다:

```java
// Java
@Component
public class MovieRecommender {

	private final String catalog;

	public MovieRecommender(@Value("${catalog.name}") String catalog) {
		this.catalog = catalog;
	}
}
```

```kotlin
// Kotlin
@Component
class MovieRecommender(
    @Value("\\${catalog.name}") private val catalog: String // Use \\ to escape $ in Kotlin
)
```

다음 구성과 함께:

```java
// Java
@Configuration
@PropertySource("classpath:application.properties")
public class AppConfig { }
```

```kotlin
// Kotlin
@Configuration
@PropertySource("classpath:application.properties")
class AppConfig
```

그리고 다음 `application.properties` 파일과 함께:

```
catalog.name=MovieCatalog
```

이 경우, `catalog` 파라미터와 필드는 `MovieCatalog` 값과 동일하게 됩니다.

스프링은 기본적으로 관대한(lenient) 내장 값 해석기(embedded value resolver)를 제공합니다.

이는 속성 값을 해석하려고 시도하며, 해석될 수 없는 경우 속성 이름(예: `${catalog.name}`)이 값으로 주입됩니다.

존재하지 않는 값에 대한 엄격한 제어를 유지하려면, 다음 예제와 같이 `PropertySourcesPlaceholderConfigurer` 빈을 선언해야 합니다:

```java
// Java
@Configuration
public class AppConfig {

	@Bean
	public static PropertySourcesPlaceholderConfigurer propertyPlaceholderConfigurer() {
		return new PropertySourcesPlaceholderConfigurer();
	}
}
```

```kotlin
// Kotlin
@Configuration
class AppConfig {
    companion object { // Bean methods for post-processors must be static
        @Bean
        @JvmStatic // Ensure the method is static
        fun propertyPlaceholderConfigurer(): PropertySourcesPlaceholderConfigurer {
            return PropertySourcesPlaceholderConfigurer()
        }
    }
}
```

*JavaConfig를 사용하여 `PropertySourcesPlaceholderConfigurer`를 구성할 때, `@Bean` 메소드는 `static`이어야 합니다.*

위의 구성을 사용하면 `${}` 플레이스홀더를 해결할 수 없는 경우 스프링 초기화가 실패하도록 보장합니다.

또한 `setPlaceholderPrefix`, `setPlaceholderSuffix`, `setValueSeparator` 또는 `setEscapeCharacter`와 같은 메소드를 사용하여 플레이스홀더를 사용자 정의할 수도 있습니다.

*스프링 부트는 기본적으로 `application.properties` 및 `application.yml` 파일에서 속성을 가져올 `PropertySourcesPlaceholderConfigurer` 빈을 구성합니다.*

스프링에서 제공하는 내장 변환기 지원은 간단한 타입 변환(예: `Integer` 또는 `int`로)이 자동으로 처리되도록 합니다.

여러 개의 쉼표로 구분된 값은 추가 노력 없이 자동으로 `String` 배열로 변환될 수 있습니다.

```java
// Java
@Component
public class MovieRecommender {

	private final String catalog;

	public MovieRecommender(@Value("${catalog.name:defaultCatalog}") String catalog) {
		this.catalog = catalog;
	}
}
```

```kotlin
// Kotlin
@Component
class MovieRecommender(
    // If catalog.name property is not found, "defaultCatalog" will be used
    @Value("\\${catalog.name:defaultCatalog}") private val catalog: String
)
```

스프링 `BeanPostProcessor`는 `@Value`의 `String` 값을 대상 타입으로 변환하는 프로세스를 처리하기 위해 내부적으로 `ConversionService`를 사용합니다.

자신만의 커스텀 타입에 대한 변환 지원을 제공하려면, 다음 예제와 같이 자신만의 `ConversionService` 빈 인스턴스를 제공할 수 있습니다:

```java
// Java
@Configuration
public class AppConfig {

	@Bean
	public ConversionService conversionService() {
		DefaultFormattingConversionService conversionService = new DefaultFormattingConversionService();
		conversionService.addConverter(new MyCustomConverter());
		return conversionService;
	}
}
```

```kotlin
// Kotlin
@Configuration
class AppConfig {
    @Bean
    fun conversionService(): ConversionService {
        val conversionService = DefaultFormattingConversionService()
        conversionService.addConverter(MyCustomConverter()) // Assuming MyCustomConverter exists
        return conversionService
    }
}
```

`@Value`에 SpEL 표현식(expression)이 포함된 경우, 다음 예제와 같이 런타임 시 값이 동적으로 계산됩니다:

```java
// Java
@Component
public class MovieRecommender {

	private final String catalog;

	public MovieRecommender(@Value("#{systemProperties['user.catalog'] + 'Catalog' }") String catalog) {
		this.catalog = catalog;
	}
}
```

```kotlin
// Kotlin
@Component
class MovieRecommender(
    // Concatenates system property 'user.catalog' with 'Catalog'
    @Value("#{systemProperties['user.catalog'] + 'Catalog' }") private val catalog: String
)
```

SpEL은 또한 더 복잡한 데이터 구조 사용을 가능하게 합니다:
{% raw %}
```java
// Java
@Component
public class MovieRecommender {

	private final Map<String, Integer> countOfMoviesPerCatalog;

	public MovieRecommender(
			@Value("#{{'Thriller': 100, 'Comedy': 300}}") Map<String, Integer> countOfMoviesPerCatalog) {
		this.countOfMoviesPerCatalog = countOfMoviesPerCatalog;
	}
}
```
{% endraw %}
{% raw %}
```kotlin
// Kotlin
@Component
class MovieRecommender(
    // Creates a Map directly using SpEL map literal syntax
    @Value("#{{'Thriller': 100, 'Comedy': 300}}")
    private val countOfMoviesPerCatalog: Map<String, Int>
)
```
{% endraw %}
---

**전체 주제: `@Value` 사용하기**

이 부분은 스프링 빈의 속성 값을 코드나 XML에 직접 하드코딩하는 대신, 외부 프로퍼티 파일이나 시스템 환경 변수 등에서 값을 가져와 **동적으로 주입**하는 방법에 대한 내용입니다.

특히 **설정 값 외부화(externalized configuration)** 에 매우 유용합니다.

**핵심 아이디어:** 빈이 사용할 설정 값(문자열, 숫자 등)을 외부 파일에서 읽어와서 코드에 자동으로 넣어주자!

---

**1. 기본적인 `@Value` 사용법 (프로퍼티 값 주입):**

- **`@Value` 어노테이션:** 필드, 생성자 파라미터, 메소드 파라미터 등에 붙여서 사용합니다.
- **플레이스홀더(Placeholder) 사용:** `@Value("${프로퍼티.키}")` 와 같이 `${...}` 구문을 사용하여 외부 설정 파일에 정의된 키(key)를 지정합니다.
- **동작 방식:**
  1. 스프링 컨테이너는 `@Value("${catalog.name}")` 어노테이션을 발견합니다.
  2. 스프링 환경(Environment)에 등록된 프로퍼티 소스(예: `application.properties` 파일, 시스템 환경 변수 등)에서 `catalog.name`이라는 키를 찾습니다.
  3. 찾은 값("MovieCatalog")을 해당 필드나 파라미터(`catalog`)에 주입합니다.
- **예시:**
  - **application.properties 파일:**

      ```
      catalog.name=MovieCatalog
      ```

  - **Java Config (프로퍼티 소스 지정):** `@PropertySource` 어노테이션으로 읽어올 파일을 지정합니다. (스프링 부트에서는 기본적으로 `application.properties`를 읽으므로 생략 가능)

      ```java
      @Configuration
      @PropertySource("classpath:application.properties")
      public class AppConfig { }
      ```

  - **빈 코드 (값 주입):**

      ```java
      @Component
      public class MovieRecommender {
          private final String catalog;
      
          // 생성자 파라미터에 @Value 사용
          public MovieRecommender(@Value("${catalog.name}") String catalog) {
              this.catalog = catalog; // "MovieCatalog" 값이 주입됨
          }
      }
      ```

      ```kotlin
      @Component
      class MovieRecommender(
          // Kotlin에서는 $ 문자를 이스케이프해야 함 (\\\\$)
          @Value("\\\\${catalog.name}") private val catalog: String
      )
      ```


---

**2. 값 해석 실패 시 동작 및 엄격 모드:**

- **기본 동작 (관대함):** 만약 스프링이 `${catalog.name}` 같은 플레이스홀더를 해석(resolve)하지 못하면 (즉, 해당 키의 프로퍼티 값이 없으면), 기본적으로는 오류를 내지 않고 **플레이스홀더 문자열 자체 (`"${catalog.name}"`)** 를 값으로 주입합니다.
- **엄격 모드 (Strict Mode):** 만약 프로퍼티 값을 찾지 못했을 때 오류를 발생시켜 애플리케이션 시작을 중단시키고 싶다면 (더 안전한 방식), **`PropertySourcesPlaceholderConfigurer` 빈을 명시적으로 등록**해야 합니다.
  - 이 빈은 이전에 설명한 `BeanFactoryPostProcessor`로, 플레이스홀더 처리를 담당합니다. 명시적으로 등록하면 기본 관대함 대신 엄격하게 동작합니다.
  - **주의:** Java Config에서 이 빈을 등록할 때는 `@Bean` 메소드를 **`static`** 으로 선언해야 합니다. (다른 빈들이 생성되기 전에 먼저 실행되어야 하므로)

    ```java
    @Configuration
    public class AppConfig {
        @Bean
        public static PropertySourcesPlaceholderConfigurer propertyPlaceholderConfigurer() {
            // 이 빈을 등록하면 플레이스홀더를 못 찾을 때 오류 발생
            return new PropertySourcesPlaceholderConfigurer();
        }
    }
    ```

  (스프링 부트는 기본적으로 엄격 모드처럼 동작하는 설정을 제공합니다.)

- **플레이스홀더 커스터마이징:** `PropertySourcesPlaceholderConfigurer` 빈을 직접 설정하면 접두사(`${`), 접미사(`}`), 값 구분자(:) 등을 변경할 수도 있습니다.

---

**3. 타입 변환 및 기본값:**

- **자동 타입 변환:** 스프링은 주입받으려는 타입(예: `int`, `boolean`, `Integer`)에 맞춰 프로퍼티 값(문자열)을 **자동으로 변환**해줍니다. 쉼표(`,`)로 구분된 값은 `String[]` (문자열 배열)로 자동 변환됩니다.
- **기본값 제공:** 만약 프로퍼티 키가 존재하지 않을 경우 사용할 **기본값**을 지정할 수 있습니다. 플레이스홀더 안에 콜론(`:`)을 사용합니다: `${프로퍼티.키:기본값}`

    ```java
    // catalog.name 프로퍼티가 없으면 "defaultCatalog" 문자열 사용
    @Value("${catalog.name:defaultCatalog}")
    private String catalog;
    
    ```


---

**4. 커스텀 타입 변환 (`ConversionService`):**

- 만약 스프링이 기본적으로 지원하지 않는 커스텀 타입(예: 직접 만든 `MyCustomType` 클래스)으로 프로퍼티 값을 변환해야 한다면, `Converter` 인터페이스를 구현한 커스텀 변환기를 만들고 이를 `ConversionService` 빈으로 등록하면 됩니다.
- 스프링은 컨테이너에 `ConversionService` 빈이 등록되어 있으면 `@Value` 처리 시 이 서비스를 사용하여 타입 변환을 시도합니다.

    ```java
    @Configuration
    public class AppConfig {
        @Bean
        public ConversionService conversionService() {
            DefaultFormattingConversionService conversionService = new DefaultFormattingConversionService();
            // 직접 만든 MyCustomConverter 등록
            conversionService.addConverter(new MyCustomConverter());
            return conversionService;
        }
    }
    ```


---

**5. SpEL(Spring Expression Language) 사용:**

`@Value` 어노테이션 안에는 단순히 프로퍼티 플레이스홀더뿐만 아니라, 더 강력한 **SpEL 표현식**을 사용할 수도 있습니다. SpEL 표현식은 `#{ ... }` 구문을 사용합니다.

- **동적 값 계산:** 런타임에 값을 동적으로 계산하여 주입할 수 있습니다.

    ```java
    // 시스템 프로퍼티 'user.catalog' 값 뒤에 "Catalog" 문자열을 붙여서 주입
    @Value("#{systemProperties['user.catalog'] + 'Catalog' }")
    private String catalog;
    ```

- **복잡한 데이터 구조 생성:** SpEL을 사용하여 리스트나 맵 같은 데이터 구조를 직접 만들어서 주입할 수도 있습니다.
{% raw %}
    ```java
    // SpEL의 맵 리터럴 구문을 사용하여 Map 객체를 생성하고 주입
    @Value("#{{'Thriller': 100, 'Comedy': 300}}")
    private Map<String, Integer> countOfMoviesPerCatalog;
    ```
{% endraw %}


**요약:**

`@Value` 어노테이션은 주로 **외부 설정 값(프로퍼티, 시스템 변수 등)** 을 스프링 빈의 필드나 파라미터에 주입하는 데 사용됩니다.

`${...}` 플레이스홀더 구문을 사용하며, 기본값 지정, 자동 타입 변환 기능을 제공합니다.

값을 찾지 못했을 때 오류를 내도록 엄격 모드를 설정할 수 있으며, 커스텀 타입 변환도 가능합니다. 또한, `#{...}` 구문을 사용하여 강력한 SpEL 표현식을 통해 동적인 값을 계산하거나 복잡한 데이터를 주입할 수도 있습니다.
