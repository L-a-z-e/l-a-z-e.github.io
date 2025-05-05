---
title: Spring Framework Type Conversion
description: 
author: laze
date: 2025-05-05 00:00:05 +0900
categories: [Dev, SpringBoot]
tags: [SpringBoot]
---
**Spring Type Conversion**

`core.convert` 패키지는 일반적인 타입 변환 시스템을 제공합니다.

이 시스템은 타입 변환 로직을 구현하기 위한 SPI(서비스 제공자 인터페이스)와 런타임 시 타입 변환을 수행하기 위한 API를 정의합니다.

스프링 컨테이너 내에서 이 시스템을 `PropertyEditor` 구현의 대안으로 사용하여 외부화된 빈 속성 값 문자열을 필요한 속성 타입으로 변환할 수 있습니다.

애플리케이션의 어느 곳에서든 타입 변환이 필요한 경우 이 공개 API를 사용할 수도 있습니다.

**Converter SPI**

타입 변환 로직을 구현하기 위한 SPI는 다음 인터페이스 정의에서 보여주듯이 간단하고 강력한 타입 지정(strongly typed)을 가집니다:

```java
package org.springframework.core.convert.converter;

public interface Converter<S, T> {

	T convert(S source);
}
```

자신만의 변환기(converter)를 생성하려면, `Converter` 인터페이스를 구현하고 S를 변환할 소스 타입으로, T를 변환될 타겟 타입으로 파라미터화합니다.

위임하는 배열 또는 컬렉션 변환기도 등록되어 있다면(기본적으로 `DefaultConversionService`가 수행함), S의 컬렉션이나 배열을 T의 배열이나 컬렉션으로 변환해야 하는 경우에도 이러한 변환기를 투명하게 적용할 수 있습니다.

`convert(S)`에 대한 각 호출에 대해, 소스 인수는 null이 아님이 보장됩니다.

변환이 실패하면 `Converter`는 모든 확인되지 않은(unchecked) 예외를 던질 수 있습니다.

특히, 잘못된 소스 값을 보고하기 위해 `IllegalArgumentException`을 던져야 합니다.

`Converter` 구현이 스레드 안전(thread-safe)한지 확인하는 데 주의하십시오.

편의를 위해 `core.convert.support` 패키지에 여러 변환기 구현이 제공됩니다.

여기에는 문자열에서 숫자 및 기타 일반적인 타입으로의 변환기가 포함됩니다.

다음 목록은 일반적인 `Converter` 구현인 `StringToInteger` 클래스를 보여줍니다:

```java
package org.springframework.core.convert.support;

final class StringToInteger implements Converter<String, Integer> {

	@Override
	public Integer convert(String source) {
		return Integer.valueOf(source);
	}
}
```

**ConverterFactory 사용하기 (Using ConverterFactory)**

전체 클래스 계층 구조에 대한 변환 로직을 중앙 집중화해야 하는 경우(예: `String`에서 `Enum` 객체로 변환할 때), 다음 예제와 같이 `ConverterFactory`를 구현할 수 있습니다:

```java
package org.springframework.core.convert.converter;

public interface ConverterFactory<S, R> {

	<T extends R> Converter<S, T> getConverter(Class<T> targetType);
}
```

S를 변환할 소스 타입으로, R을 변환할 수 있는 클래스 범위(range)를 정의하는 기본 타입으로 파라미터화합니다.

그런 다음 `getConverter(Class<T>)`를 구현합니다.

여기서 T는 R의 하위 클래스입니다.

예제로 `StringToEnumConverterFactory`를 고려하십시오:

```java
package org.springframework.core.convert.support;

final class StringToEnumConverterFactory implements ConverterFactory<String, Enum> {

	@Override
	public <T extends Enum> Converter<String, T> getConverter(Class<T> targetType) {
		return new StringToEnumConverter(targetType);
	}

	private static final class StringToEnumConverter<T extends Enum> implements Converter<String, T> {

		private final Class<T> enumType;

		public StringToEnumConverter(Class<T> enumType) {
			this.enumType = enumType;
		}

		@Override
		public T convert(String source) {
			return (T) Enum.valueOf(this.enumType, source.trim());
		}
	}
}
```

**GenericConverter 사용하기 (Using GenericConverter)**

정교한 `Converter` 구현이 필요한 경우 `GenericConverter` 인터페이스 사용을 고려하십시오.

`Converter`보다 더 유연하지만 덜 강력한 타입 지정 시그니처를 가진 `GenericConverter`는 여러 소스 및 대상 타입 간의 변환을 지원합니다.

또한 `GenericConverter`는 변환 로직을 구현할 때 사용할 수 있는 소스 및 대상 필드 컨텍스트(context)를 제공합니다.

이러한 컨텍스트를 통해 필드 어노테이션이나 필드 시그니처에 선언된 제네릭 정보에 의해 타입 변환이 구동될 수 있습니다. 다음 목록은 `GenericConverter`의 인터페이스 정의를 보여줍니다:

```java
package org.springframework.core.convert.converter;

import java.util.Set;
import org.springframework.core.convert.TypeDescriptor;

public interface GenericConverter {

	Set<ConvertiblePair> getConvertibleTypes();

	Object convert(Object source, TypeDescriptor sourceType, TypeDescriptor targetType);

	final class ConvertiblePair {
		// Constructor and methods omitted for brevity
	}
}
```

`GenericConverter`를 구현하려면, `getConvertibleTypes()`가 지원되는 소스→대상 타입 쌍을 반환하도록 합니다.

그런 다음 변환 로직을 포함하도록 `convert(Object, TypeDescriptor, TypeDescriptor)`를 구현합니다.

소스 `TypeDescriptor`는 변환되는 값을 보유하는 소스 필드에 대한 접근을 제공합니다.

대상 `TypeDescriptor`는 변환된 값이 설정될 대상 필드에 대한 접근을 제공합니다.

`GenericConverter`의 좋은 예는 자바 배열과 컬렉션 간에 변환하는 변환기입니다.

이러한 `ArrayToCollectionConverter`는 대상 컬렉션 타입의 필드를 내부 검사(introspects)하여 컬렉션의 요소 타입을 해석합니다.

이를 통해 소스 배열의 각 요소가 컬렉션이 대상 필드에 설정되기 전에 컬렉션 요소 타입으로 변환될 수 있습니다.

`*GenericConverter`는 더 복잡한 SPI 인터페이스이므로 필요할 때만 사용해야 합니다. 기본 타입 변환 요구 사항에는 `Converter` 또는 `ConverterFactory`를 선호하십시오.*

**ConditionalGenericConverter 사용하기 (Using ConditionalGenericConverter)**

때로는 특정 조건이 참일 때만 `Converter`를 실행하고 싶을 수 있습니다.

예를 들어, 대상 필드에 특정 어노테이션이 있는 경우에만 `Converter`를 실행하거나, 대상 클래스에 특정 메소드(예: 정적 `valueOf` 메소드)가 정의된 경우에만 `Converter`를 실행하고 싶을 수 있습니다.

`ConditionalGenericConverter`는 `GenericConverter`와 `ConditionalConverter` 인터페이스의 합집합으로, 이러한 커스텀 매칭 기준을 정의할 수 있게 합니다:

```java
public interface ConditionalConverter {

	boolean matches(TypeDescriptor sourceType, TypeDescriptor targetType);
}

public interface ConditionalGenericConverter extends GenericConverter, ConditionalConverter {
}
```

`ConditionalGenericConverter`의 좋은 예는 영속 엔티티 식별자와 엔티티 참조 간에 변환하는 `IdToEntityConverter`입니다.

이러한 `IdToEntityConverter`는 대상 엔티티 타입이 정적 파인더 메소드(예: `findAccount(Long)`)를 선언하는 경우에만 매치될 수 있습니다.

이러한 파인더 메소드 검사는 `matches(TypeDescriptor, TypeDescriptor)` 구현에서 수행할 수 있습니다.

**ConversionService API**

`ConversionService`는 런타임 시 타입 변환 로직을 실행하기 위한 통합된 API를 정의합니다.

변환기는 종종 다음 퍼사드(facade) 인터페이스 뒤에서 실행됩니다:

```java
package org.springframework.core.convert;

import org.springframework.core.convert.TypeDescriptor;
import org.springframework.lang.Nullable;

public interface ConversionService {

	boolean canConvert(@Nullable Class<?> sourceType, Class<?> targetType);

	@Nullable
	<T> T convert(@Nullable Object source, Class<T> targetType);

	boolean canConvert(@Nullable TypeDescriptor sourceType, TypeDescriptor targetType);

	@Nullable
	Object convert(@Nullable Object source, @Nullable TypeDescriptor sourceType, TypeDescriptor targetType);
}
```

대부분의 `ConversionService` 구현체는 변환기 등록을 위한 SPI를 제공하는 `ConverterRegistry`도 구현합니다.

내부적으로 `ConversionService` 구현체는 타입 변환 로직을 수행하기 위해 등록된 변환기들에 위임합니다.

견고한 `ConversionService` 구현은 `core.convert.support` 패키지에 제공됩니다.

`GenericConversionService`는 대부분의 환경에서 사용하기에 적합한 범용 구현입니다.

`ConversionServiceFactory`는 일반적인 `ConversionService` 구성을 생성하기 위한 편리한 팩토리를 제공합니다.

**ConversionService 구성하기 (Configuring a ConversionService)**

`ConversionService`는 애플리케이션 시작 시 인스턴스화되고 여러 스레드 간에 공유되도록 설계된 상태 비저장(stateless) 객체입니다.

스프링 애플리케이션에서는 일반적으로 각 스프링 컨테이너(또는 `ApplicationContext`)에 대해 `ConversionService` 인스턴스를 구성합니다.

스프링은 해당 `ConversionService`를 선택하고 프레임워크에 의해 타입 변환이 수행되어야 할 때마다 사용합니다.

이 `ConversionService`를 모든 빈에 주입하고 직접 호출할 수도 있습니다.

*스프링에 `ConversionService`가 등록되지 않으면, 원래의 `PropertyEditor` 기반 시스템이 사용됩니다.*

기본 `ConversionService`를 스프링에 등록하려면 `conversionService`라는 id를 가진 다음 빈 정의를 추가합니다:

```xml
<bean id="conversionService"
	class="org.springframework.context.support.ConversionServiceFactoryBean"/>
```

기본 `ConversionService`는 문자열, 숫자, 열거형, 컬렉션, 맵 및 기타 일반적인 타입 간에 변환할 수 있습니다.

기본 변환기를 자신만의 커스텀 변환기로 보충하거나 오버라이드하려면 `converters` 속성을 설정합니다. 속성 값은 `Converter`, `ConverterFactory` 또는 `GenericConverter` 인터페이스 중 하나를 구현할 수 있습니다.

```xml
<bean id="conversionService"
		class="org.springframework.context.support.ConversionServiceFactoryBean">
	<property name="converters">
		<set>
			<bean class="example.MyCustomConverter"/>
		</set>
	</property>
</bean>
```

스프링 MVC 애플리케이션 내에서 `ConversionService`를 사용하는 것도 일반적입니다.

**프로그래밍 방식으로 ConversionService 사용하기 (Using a ConversionService Programmatically)**

`ConversionService` 인스턴스를 프로그래밍 방식으로 사용하려면 다른 빈처럼 참조를 주입할 수 있습니다.

```java
// Java
@Service
public class MyService {

	private final ConversionService conversionService;

	public MyService(ConversionService conversionService) { // Inject ConversionService
		this.conversionService = conversionService;
	}

	public void doIt() {
		// Example: Convert an integer to a string
		String result = this.conversionService.convert(123, String.class);
		System.out.println(result);
	}
}
```

```kotlin
// Kotlin
@Service
class MyService( // Inject ConversionService via constructor
    private val conversionService: ConversionService
) {
    fun doIt() {
        // Example: Convert an integer to a string
        val result = conversionService.convert(123, String::class.java)
        println(result)
    }
}
```

대부분의 사용 사례에서는 `targetType`을 지정하는 `convert` 메소드를 사용할 수 있지만, 파라미터화된 요소의 컬렉션과 같은 더 복잡한 타입에는 작동하지 않습니다.

예를 들어, `Integer`의 `List`를 `String`의 `List`로 프로그래밍 방식으로 변환하려면 소스 및 대상 타입의 형식적인 정의를 제공해야 합니다.

다행히 `TypeDescriptor`는 다음 예제와 같이 이를 간단하게 만드는 다양한 옵션을 제공합니다:

```java
// Java
DefaultConversionService cs = new DefaultConversionService();

List<Integer> input = ... // Initialize your list
// Convert List<Integer> to List<String>
List<String> output = (List<String>) cs.convert(input,
	TypeDescriptor.forObject(input), // Source type descriptor (List<Integer>)
	TypeDescriptor.collection(List.class, TypeDescriptor.valueOf(String.class))); // Target type descriptor (List<String>)
```

```kotlin
// Kotlin
val cs = DefaultConversionService()

val input: List<Integer> = listOf(1, 2, 3) // Initialize your list
// Convert List<Integer> to List<String>
val output = cs.convert(input,
    TypeDescriptor.forObject(input), // Source type descriptor (List<Integer>)
    TypeDescriptor.collection(List::class.java, TypeDescriptor.valueOf(String::class.java))) as List<String>? // Target type descriptor (List<String>) and cast
```

`DefaultConversionService`는 대부분의 환경에 적합한 변환기를 자동으로 등록한다는 점에 유의하십시오.

여기에는 컬렉션 변환기, 스칼라 변환기 및 기본 `Object`-to-`String` 변환기가 포함됩니다. `DefaultConversionService` 클래스의 정적 `addDefaultConverters` 메소드를 사용하여 모든 `ConverterRegistry`에 동일한 변환기를 등록할 수 있습니다.

값 타입에 대한 변환기는 배열 및 컬렉션에 재사용되므로, 표준 컬렉션 처리가 적절하다고 가정하면 `S`의 `Collection`을 `T`의 `Collection`으로 변환하기 위해 특정 변환기를 생성할 필요가 없습니다.

---

**전체 주제: 스프링 타입 변환 (Spring Type Conversion)**

이 부분은 스프링이 **어떤 타입의 객체를 다른 타입의 객체로 변환**하는 과정을 어떻게 처리하는지,

그리고 개발자가 자신만의 변환 규칙을 어떻게 추가하여 사용할 수 있는지 설명합니다. 데이터 바인딩(`@Value`, 웹 입력)이나 SpEL 처리 등 다양한 곳에서 내부적으로 활용됩니다.

**핵심 아이디어:** 자바 객체의 타입을 다른 타입으로 변환하는 과정을 표준화하고 자동화하자! `PropertyEditor`보다 더 유연하고 강력한 새 변환 시스템을 사용하자!

---

**1. `core.convert` 패키지: 새로운 타입 변환 시스템**

- **도입 배경:** 기존의 `PropertyEditor` 기반 변환 시스템은 몇 가지 한계가 있었습니다 (예: `String` <-> 객체 변환만 지원, 스레드 안전성 문제, 상태 관리 필요성 등). 이를 극복하기 위해 스프링 3.0부터 **`org.springframework.core.convert`** 패키지에 새로운 타입 변환 시스템이 도입되었습니다.
- **구성:**
  - **SPI (Service Provider Interface):** 개발자가 자신만의 변환 로직을 구현하기 위한 인터페이스 (`Converter`, `ConverterFactory`, `GenericConverter`).
  - **API (Application Programming Interface):** 런타임 시 변환을 수행하기 위한 인터페이스 (`ConversionService`).
- **활용:** 이 시스템은 `PropertyEditor`를 대체하여 **외부화된 설정 값 주입(`@Value`, `PropertySourcesPlaceholderConfigurer`)**, **데이터 바인딩(웹 입력)**, **SpEL 표현식 평가** 등 스프링 내부의 다양한 타입 변환 작업에 사용됩니다. 또한 개발자가 필요할 때 직접 사용하기 위한 공개 API로도 제공됩니다.

---

**2. 변환기(Converter) 구현하기 (SPI)**

개발자가 자신만의 변환 로직을 만들려면 다음 인터페이스 중 하나를 구현합니다.

- **`Converter<S, T>` (가장 기본):**
  - **역할:** **하나의 소스 타입(S)** 을 **하나의 타겟 타입(T)** 으로 변환하는 가장 기본적인 변환기입니다.
  - **구현 메소드:** `T convert(S source)` 메소드 하나만 구현하면 됩니다.
  - **요구사항:** `source`는 `null`이 아님을 보장받습니다. 구현체는 **스레드 안전(thread-safe)** 해야 합니다.
  - **예시:** 문자열 "123"을 정수 123으로 변환하는 `StringToInteger` 같은 변환.
  - **자동 적용:** `List<S>`를 `List<T>`로 변환하는 경우 등에도 이 변환기가 투명하게 적용될 수 있습니다.
- **`ConverterFactory<S, R>` (클래스 계층 변환):**
  - **역할:** **하나의 소스 타입(S)** 을 **특정 기본 타입(R)의 하위 타입들(T extends R)** 로 변환하는 변환기들을 생성하는 팩토리입니다.
  - **구현 메소드:** `getConverter(Class<T> targetType)` 메소드를 구현하여 요청된 타겟 타입(`T`)에 맞는 `Converter`를 생성하고 반환합니다.
  - **예시:** 문자열을 모든 `Enum` 타입으로 변환하는 `StringToEnumConverterFactory`. 이 팩토리는 요청된 `Enum` 타입(예: `Color.class`)에 맞는 `StringToEnumConverter`를 생성하여 반환합니다.
- **`GenericConverter` (가장 유연하지만 복잡):**
  - **역할:** **여러 소스 타입**을 **여러 타겟 타입**으로 변환하거나, 변환 시 **필드 컨텍스트(Field Context)** (어떤 필드에 주입되는지, 필드의 어노테이션이나 제네릭 정보 등)가 필요할 때 사용합니다.
  - **구현 메소드:**
    - `Set<ConvertiblePair> getConvertibleTypes()`: 이 변환기가 지원하는 모든 소스→타겟 타입 쌍 목록을 반환합니다.
    - `Object convert(Object source, TypeDescriptor sourceType, TypeDescriptor targetType)`: 실제 변환 로직을 구현합니다. `TypeDescriptor` 객체를 통해 소스 및 타겟 필드의 상세 정보(어노테이션, 제네릭 타입 등)에 접근할 수 있습니다.
  - **예시:** 배열 <-> 컬렉션 변환 (`ArrayToCollectionConverter`), 특정 어노테이션이 붙은 필드에 대한 특별 변환 등.
  - **필요할 때만 사용:** 일반적인 변환 요구사항에는 `Converter` 또는 `ConverterFactory`를 사용하는 것이 더 간단합니다.
- **`ConditionalConverter` & `ConditionalGenericConverter` (조건부 변환):**
  - **역할:** 변환 로직 실행 전에 추가적인 **조건 검사**가 필요한 경우 사용합니다.
  - **구현 메소드:** `boolean matches(TypeDescriptor sourceType, TypeDescriptor targetType)` 메소드를 구현하여, 이 변환기를 사용할지 말지 결정하는 조건을 정의합니다 (예: 타겟 필드에 특정 어노테이션이 붙어 있는지, 특정 메소드가 존재하는지 등).
  - `ConditionalGenericConverter`는 `GenericConverter` + `ConditionalConverter` 입니다.

---

**3. `ConversionService` API: 변환 실행**

- **`ConversionService` 인터페이스:** 런타임 시 변환 로직을 **실행**하기 위한 **통합된 API**입니다. 등록된 모든 변환기들을 관리하고, 실제 변환 요청이 들어오면 적절한 변환기를 찾아 실행합니다.
- **주요 메소드:**
  - `boolean canConvert(Class<?> sourceType, Class<?> targetType)`: 특정 소스 타입을 타겟 타입으로 변환 가능한지 확인.
  - `<T> T convert(Object source, Class<T> targetType)`: 실제 변환을 수행.
  - `canConvert(TypeDescriptor sourceType, TypeDescriptor targetType)`: `TypeDescriptor`를 사용하여 더 복잡한 타입(제네릭, 어노테이션 포함) 변환 가능 여부 확인.
  - `convert(Object source, TypeDescriptor sourceType, TypeDescriptor targetType)`: `TypeDescriptor`를 사용하여 변환 수행.
- **구현체:** `org.springframework.core.convert.support.GenericConversionService`가 가장 일반적이고 견고한 구현체입니다. `DefaultConversionService`는 `GenericConversionService`를 상속하며, 자주 사용되는 기본 변환기들을 자동으로 등록해 둔 구현체입니다.

---

**4. `ConversionService` 구성하기 (스프링 컨테이너에 등록)**

- **등록:** 스프링 컨테이너에서 이 새로운 타입 변환 시스템을 활성화하고 사용하려면, **`conversionService`** 라는 **고정된 이름**으로 `ConversionService` 타입의 빈을 등록해야 합니다. (이 이름이 아니면 스프링이 자동으로 인식하지 못하고 `PropertyEditor` 시스템을 사용합니다.)
- **`ConversionServiceFactoryBean` 사용 (XML):**`converters` 속성에 커스텀 변환기들을 등록하면, `ConversionServiceFactoryBean`이 내부적으로 `GenericConversionService`를 생성하고 기본 변환기들과 함께 등록해줍니다.

    ```xml
    <bean id="conversionService"
        class="org.springframework.context.support.ConversionServiceFactoryBean">
        <property name="converters">
            <set>
                <!-- 여기에 직접 만든 Converter, ConverterFactory, GenericConverter 빈 또는 클래스 등록 -->
                <bean class="example.MyCustomConverter"/>
            </set>
        </property>
    </bean>
    ```

- **`DefaultConversionService` 사용 (Java Config):** Java Config에서는 `DefaultConversionService` 인스턴스를 직접 생성하고 커스텀 변환기를 추가한 후 빈으로 등록하는 것이 일반적입니다.

    ```java
    @Configuration
    public class AppConfig {
        @Bean
        public ConversionService conversionService() { // 빈 이름이 "conversionService"
            DefaultFormattingConversionService conversionService = new DefaultFormattingConversionService();
            // 커스텀 변환기 추가
            conversionService.addConverter(new MyCustomConverter());
            // Formatting 관련 변환기도 추가 가능 (FormatterRegistry 인터페이스 확장)
            // conversionService.addFormatter(...);
            return conversionService;
        }
    }
    ```

- **자동 활성화:** `conversionService`라는 이름의 빈이 등록되면, 스프링 프레임워크(예: `@Value` 처리기, 데이터 바인더 등)는 **자동으로 이 빈을 찾아서 타입 변환에 사용**합니다.

---

**5. `ConversionService` 프로그래밍 방식 사용하기**

- `ConversionService` 빈은 다른 빈들처럼 `@Autowired` 등으로 주입받아서 코드 내에서 **직접 타입 변환 기능**을 사용할 수도 있습니다.

    ```java
    @Service
    public class MyService {
        private final ConversionService conversionService;
    
        @Autowired
        public MyService(ConversionService conversionService) { // ConversionService 주입
            this.conversionService = conversionService;
        }
    
        public void processValue(String input) {
            // 문자열을 정수로 변환 (기본 변환기 사용)
            Integer number = conversionService.convert(input, Integer.class);
            System.out.println("변환된 숫자: " + number);
        }
    
        public void processComplexValue(List<String> stringList) {
             // List<String>을 List<Integer>로 변환 (컬렉션 변환기와 StringToInteger 변환기 조합)
            List<Integer> intList = (List<Integer>) conversionService.convert(stringList,
                TypeDescriptor.forObject(stringList), // 소스 타입 (List<String>)
                TypeDescriptor.collection(List.class, TypeDescriptor.valueOf(Integer.class))); // 타겟 타입 (List<Integer>)
             System.out.println("변환된 숫자 리스트: " + intList);
        }
    }
    ```

- `TypeDescriptor`를 사용하면 컬렉션, 배열, 제네릭 등 복잡한 타입 변환도 프로그래밍 방식으로 수행할 수 있습니다.

**요약:**

스프링의 새로운 타입 변환 시스템(`core.convert`)은 `Converter`, `ConverterFactory`, `GenericConverter` 같은 **SPI**를 통해 개발자가 변환 로직을 구현하고, `ConversionService` **API**를 통해 런타임에 변환을 실행합니다. `conversionService`라는 이름의 빈을 등록하면 스프링 프레임워크가 이를 자동으로 사용하며, 커스텀 변환기를 추가하여 자신만의 타입 변환 규칙을 정의할 수 있습니다. 이 시스템은 `PropertyEditor`보다 유연하고 스레드 안전하며, `@Value`, 데이터 바인딩, SpEL 등 다양한 곳에서 활용됩니다. 개발자도 필요에 따라 `ConversionService` 빈을 주입받아 직접 타입 변환을 수행할 수 있습니다.

---

**1. TypeDescriptor 란 무엇인가?**

`TypeDescriptor` (`org.springframework.core.convert.TypeDescriptor`)는 단순히 자바의 `Class` 객체보다 **훨씬 더 풍부한 타입 정보**를 담는 스프링의 클래스입니다. `Class` 객체는 기본적인 클래스 타입만 알려주지만, `TypeDescriptor`는 다음과 같은 추가적인 정보들을 함께 캡슐화합니다:

- **제네릭 타입(Generic Type) 정보:** `List<String>` 에서 `String`이 무엇인지, `Map<String, Integer>` 에서 키와 값의 타입이 무엇인지 등의 제네릭 정보를 정확하게 가지고 있습니다. (`Class` 객체만으로는 런타임에 이 정보(소거된 정보)를 완벽히 알기 어렵습니다.)
- **어노테이션(Annotation) 정보:** 타입이 선언된 필드나 메소드 파라미터에 붙어 있는 어노테이션 정보(예: `@NumberFormat`, `@DateTimeFormat`, 커스텀 어노테이션)를 가지고 있습니다. 변환 시 이 어노테이션 정보를 활용할 수 있습니다.
- **배열(Array) 및 컬렉션(Collection) 요소 타입 정보:** 배열이나 컬렉션의 경우, 그 안에 들어가는 요소(element)의 타입 정보를 가지고 있습니다.
- **객체 인스턴스 (선택적):** 실제 변환 대상 객체 인스턴스를 참조할 수도 있습니다.

**왜 `TypeDescriptor`가 필요할까?**

- **정확한 타입 변환:** 단순히 `List.class` 정보만으로는 그 리스트 안에 어떤 타입의 객체가 들어가는지 알 수 없습니다. `TypeDescriptor`는 `List<Integer>` 인지 `List<String>` 인지를 명확히 구분하여, 리스트의 각 요소를 올바른 타입으로 변환하는 데 필요한 정보를 제공합니다.
- **컨텍스트 기반 변환:** 필드에 붙은 어노테이션(예: `@NumberFormat(pattern="#,###")`)에 따라 숫자 형식을 다르게 변환해야 할 때, `TypeDescriptor`를 통해 해당 어노테이션 정보를 얻어와 변환 로직에 반영할 수 있습니다.

**`TypeDescriptor` 생성 방법:**

- `TypeDescriptor.forObject(obj)`: 주어진 객체 인스턴스로부터 `TypeDescriptor` 생성.
- `TypeDescriptor.valueOf(Class<?> type)`: 단순 `Class` 객체로부터 `TypeDescriptor` 생성 (제네릭 정보 등은 없음).
- `TypeDescriptor.collection(Class<?> collectionType, TypeDescriptor elementType)`: 컬렉션 타입과 요소 타입을 지정하여 `TypeDescriptor` 생성 (예: `List.class`와 `String` 타입의 `TypeDescriptor`).
- `TypeDescriptor.map(Class<?> mapType, TypeDescriptor keyType, TypeDescriptor valueType)`: 맵 타입과 키/값 타입을 지정하여 생성.
- `new TypeDescriptor(Field field)`, `new TypeDescriptor(MethodParameter methodParam)`: 필드나 메소드 파라미터 객체로부터 어노테이션, 제네릭 정보 등을 포함하는 `TypeDescriptor` 생성 (스프링 내부에서 주로 사용).

**결론 (`TypeDescriptor`):** `TypeDescriptor`는 단순 `Class` 객체보다 **더 상세한 타입 정보(제네릭, 어노테이션 등)** 를 담는 스프링 클래스로, 복잡하고 컨텍스트에 맞는 **정확한 타입 변환**을 수행하기 위해 필요합니다.

---

**2. 왜 `List<Integer>`를 `List<String>`으로 바꾸는데 복잡한 과정이 필요한가?**

겉보기에는 `convert(input, List.class)` 처럼 간단히 될 것 같지만, 실제로는 그렇지 않습니다. 왜 `TypeDescriptor`를 사용한 코드가 필요한지 설명드릴게요.

```java
// 복잡해 보이는 코드
List<String> output = (List<String>) cs.convert(input, // 소스 객체 (List<Integer>)
    TypeDescriptor.forObject(input), // ★ 소스 타입 정보 (List<Integer>) ★
    TypeDescriptor.collection(List.class, TypeDescriptor.valueOf(String.class))); // ★ 타겟 타입 정보 (List<String>) ★

```

**이유:**

1. **타입 소거 (Type Erasure):** 자바 제네릭의 중요한 특징 중 하나는 **컴파일 시점에는 타입 정보가 있지만, 런타임 시점에는 제네릭 타입 정보가 지워진다(erased)** 는 것입니다. 즉, 런타임에는 `List<Integer>`나 `List<String>`이나 모두 그냥 `List` 객체로 취급될 뿐, 그 안에 어떤 타입의 요소가 들어가는지에 대한 정보는 `Class` 객체만으로는 직접 알기 어렵습니다.
2. **`ConversionService`는 런타임에 동작:** `ConversionService`는 런타임 시점에 타입 변환을 수행합니다. 따라서 단순히 `convert(input, List.class)` 라고만 호출하면, `ConversionService`는 "아, `List` 타입으로 변환해 달라는 거구나" 라고만 알 뿐, 그 리스트 안에 최종적으로 **어떤 타입의 요소(`String`?)** 가 들어가야 하는지는 알 수 없습니다. 또한, 소스 객체(`input`)가 `List`라는 것은 알지만, 그 요소 타입이 **원래 무엇이었는지(`Integer`?)** 도 정확히 알기 어렵습니다.
3. **요소 단위 변환 필요:** `List<Integer>`를 `List<String>`으로 변환한다는 것은 실제로는 리스트 안의 **각각의 `Integer` 요소**를 **각각의 `String` 요소**로 변환한 다음, 그 결과들을 새로운 `List`에 담아야 한다는 의미입니다. 즉, `Integer` -> `String` 변환기가 필요하고, 이 변환기를 리스트의 모든 요소에 적용해야 합니다.
4. **`TypeDescriptor`의 역할:** `TypeDescriptor`는 바로 이 **타입 소거 문제를 해결**하고, 런타임에 `ConversionService`에게 **정확한 소스 및 타겟 타입 정보(제네릭 포함)** 를 전달하는 역할을 합니다.
  - `TypeDescriptor.forObject(input)`: `input` 객체(여기서는 `List<Integer>`)의 런타임 타입 정보를 분석하여 `TypeDescriptor`를 만듭니다. 이를 통해 "소스는 `Integer`를 요소로 가지는 List이다"라는 정보를 `ConversionService`에게 알려줍니다.
  - `TypeDescriptor.collection(List.class, TypeDescriptor.valueOf(String.class))`: "타겟은 `String`을 요소로 가지는 List이다"라는 정보를 명시적으로 만들어 전달합니다.
5. **`ConversionService`의 처리:** `ConversionService`는 이 두 `TypeDescriptor` 정보를 받아서,
  - "소스 요소 타입은 `Integer`이고, 타겟 요소 타입은 `String`이구나."
  - "내가 가지고 있는 변환기 중에 `Integer`를 `String`으로 변환하는 `Converter`를 찾아야겠다." (스프링 기본 변환기 또는 커스텀 변환기)
  - 찾은 `Converter<Integer, String>`을 사용하여 소스 리스트의 각 `Integer` 요소를 `String`으로 변환합니다.
  - 변환된 `String` 요소들을 새로운 `List`에 담아서 최종적으로 반환합니다.

**결론 (복잡해 보이는 이유):**

자바의 **타입 소거(Type Erasure)** 때문에 런타임에는 제네릭 타입 정보가 명확하지 않습니다. `List<Integer>`를 `List<String>`으로 변환하려면, 단순히 `List`로 변환하는 것이 아니라 **내부 요소의 타입(`Integer` -> `String`)** 까지 정확히 알려줘야 요소 단위 변환이 가능합니다. `TypeDescriptor`는 이처럼 **소거된 제네릭 정보를 포함한 상세한 타입 정보를 런타임에 전달**하여 `ConversionService`가 **정확하고 안전하게 복잡한 타입 변환(특히 컬렉션, 제네릭 관련)을 수행**할 수 있도록 돕기 때문에 코드가 다소 길어 보이는 것입니다. 간단한 스칼라 타입(예: `int` -> `String`) 변환에는 `convert(source, targetClass)`만으로도 충분하지만, 제네릭이 포함되면 `TypeDescriptor`가 필요합니다.
