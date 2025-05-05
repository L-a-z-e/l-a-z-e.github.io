---
title: Spring Framework Field Formatting
description: 
author: laze
date: 2025-05-05 00:00:06 +0900
categories: [Dev, SpringBoot]
tags: [SpringBoot]
---
**Spring Field Formatting**

이전 섹션에서 논의했듯이, `core.convert`는 범용 타입 변환 시스템입니다. 통합된 `ConversionService` API와 한 타입에서 다른 타입으로의 변환 로직을 구현하기 위한 강력한 타입 지정 `Converter` SPI를 제공합니다.

스프링 컨테이너는 이 시스템을 사용하여 빈 속성 값을 바인딩합니다.

또한, 스프링 표현 언어(SpEL)와 `DataBinder` 모두 이 시스템을 사용하여 필드 값을 바인딩합니다.

예를 들어, SpEL이 `expression.setValue(Object bean, Object value)` 시도를 완료하기 위해 `Short`를 `Long`으로 강제 변환(coerce)해야 할 때, `core.convert` 시스템이 강제 변환을 수행합니다.

이제 웹 또는 데스크톱 애플리케이션과 같은 일반적인 클라이언트 환경의 타입 변환 요구 사항을 고려해 봅시다.

이러한 환경에서는 일반적으로 클라이언트 포스트백(postback) 프로세스를 지원하기 위해 `String`에서 변환하고, 뷰 렌더링 프로세스를 지원하기 위해 다시 `String`으로 변환합니다.

또한 종종 `String` 값을 지역화(localize)해야 합니다.

더 일반적인 `core.convert` `Converter` SPI는 이러한 포매팅 요구 사항을 직접적으로 다루지 않습니다.

이를 직접적으로 다루기 위해, 스프링은 클라이언트 환경을 위한 `PropertyEditor` 구현의 간단하고 견고한 대안을 제공하는 편리한 `Formatter` SPI를 제공합니다.

일반적으로 범용 타입 변환 로직을 구현해야 할 때(예: `java.util.Date`와 `Long` 간 변환) `Converter` SPI를 사용할 수 있습니다.

클라이언트 환경(예: 웹 애플리케이션)에서 작업하고 지역화된 필드 값을 파싱하고 출력(print)해야 할 때 `Formatter` SPI를 사용할 수 있습니다. `ConversionService`는 두 SPI 모두에 대한 통합된 타입 변환 API를 제공합니다.

**Formatter SPI**

필드 포매팅 로직을 구현하기 위한 `Formatter` SPI는 간단하고 강력한 타입 지정을 가집니다. 다음 목록은 `Formatter` 인터페이스 정의를 보여줍니다:

```java
package org.springframework.format;

public interface Formatter<T> extends Printer<T>, Parser<T> {
}
```

`Formatter`는 `Printer` 및 `Parser` 구성 요소(building-block) 인터페이스로부터 확장됩니다. 다음 목록은 해당 두 인터페이스의 정의를 보여줍니다:

```java
public interface Printer<T> {

	String print(T fieldValue, Locale locale);
}
```

```java
import java.text.ParseException;

public interface Parser<T> {

	T parse(String clientValue, Locale locale) throws ParseException;
}
```

자신만의 `Formatter`를 생성하려면, 앞서 보여준 `Formatter` 인터페이스를 구현합니다.

T를 포매팅하려는 객체의 타입(예: `java.util.Date`)으로 파라미터화합니다.

클라이언트 로케일에서 표시하기 위해 T의 인스턴스를 출력하는 `print()` 연산을 구현합니다.

클라이언트 로케일에서 반환된 포맷된 표현으로부터 T의 인스턴스를 파싱하는 `parse()` 연산을 구현합니다.

파싱 시도가 실패하면 `Formatter`는 `ParseException` 또는 `IllegalArgumentException`을 던져야 합니다.

`Formatter` 구현이 스레드 안전한지 확인하는 데 주의하십시오.

`format` 하위 패키지는 편의를 위해 여러 `Formatter` 구현을 제공합니다.

`number` 패키지는 `java.text.NumberFormat`을 사용하는 `Number` 객체를 포맷하기 위한 `NumberStyleFormatter`, `CurrencyStyleFormatter`, `PercentStyleFormatter`를 제공합니다.

`datetime` 패키지는 `java.text.DateFormat`으로 `java.util.Date` 객체를 포맷하기 위한 `DateFormatter`와, `@DurationFormat.Style` enum(포맷 어노테이션 API 참조)에 정의된 다른 스타일로 `Duration` 객체를 포맷하기 위한 `DurationFormatter`를 제공합니다.

다음 `DateFormatter`는 예제 `Formatter` 구현입니다:

```java
// Java
package org.springframework.format.datetime;

import java.text.DateFormat;
import java.text.ParseException;
import java.text.SimpleDateFormat;
import java.util.Date;
import java.util.Locale;
import org.springframework.format.Formatter;

public final class DateFormatter implements Formatter<Date> {

	private String pattern;

	public DateFormatter(String pattern) {
		this.pattern = pattern;
	}

	@Override
	public String print(Date date, Locale locale) {
		if (date == null) {
			return "";
		}
		return getDateFormat(locale).format(date);
	}

	@Override
	public Date parse(String formatted, Locale locale) throws ParseException {
		if (formatted.isEmpty()) { // Use isEmpty() for strings
			return null;
		}
		return getDateFormat(locale).parse(formatted);
	}

	protected DateFormat getDateFormat(Locale locale) {
		DateFormat dateFormat = new SimpleDateFormat(this.pattern, locale);
		dateFormat.setLenient(false);
		return dateFormat;
	}
}
```

```kotlin
// Kotlin
package org.springframework.format.datetime

import java.text.DateFormat
import java.text.ParseException
import java.text.SimpleDateFormat
import java.util.Date
import java.util.Locale
import org.springframework.format.Formatter

class DateFormatter(private val pattern: String) : Formatter<Date> {

    @Throws(ParseException::class)
    override fun parse(text: String, locale: Locale): Date? {
        return if (text.isEmpty()) {
            null
        } else getDateFormat(locale).parse(text)
    }

    override fun print(date: Date, locale: Locale): String {
        return getDateFormat(locale).format(date) ?: ""
    }

    protected fun getDateFormat(locale: Locale): DateFormat {
        val dateFormat = SimpleDateFormat(this.pattern, locale)
        dateFormat.isLenient = false
        return dateFormat
    }
}
```

**어노테이션 기반 포매팅 (Annotation-driven Formatting)**

필드 포매팅은 필드 타입 또는 어노테이션으로 구성될 수 있습니다.

어노테이션을 `Formatter`에 바인딩하려면 `AnnotationFormatterFactory`를 구현합니다.

다음 목록은 `AnnotationFormatterFactory` 인터페이스의 정의를 보여줍니다:

```java
package org.springframework.format;

import java.lang.annotation.Annotation;
import java.util.Set;

public interface AnnotationFormatterFactory<A extends Annotation> {

	Set<Class<?>> getFieldTypes();

	Printer<?> getPrinter(A annotation, Class<?> fieldType);

	Parser<?> getParser(A annotation, Class<?> fieldType);
}
```

구현을 생성하려면:

1. A를 포매팅 로직을 연관시키려는 필드 `annotationType`(예: `org.springframework.format.annotation.DateTimeFormat`)으로 파라미터화합니다.
2. `getFieldTypes()`가 어노테이션을 사용할 수 있는 필드의 타입을 반환하도록 합니다.
3. `getPrinter()`가 어노테이션된 필드의 값을 출력할 `Printer`를 반환하도록 합니다.
4. `getParser()`가 어노테이션된 필드에 대한 `clientValue`를 파싱할 `Parser`를 반환하도록 합니다.

다음 예제 `AnnotationFormatterFactory` 구현은 `@NumberFormat` 어노테이션을 포맷터에 바인딩하여 숫자 스타일이나 패턴을 지정할 수 있도록 합니다:

```java
// Java
import java.lang.annotation.Annotation;
import java.math.BigDecimal;
import java.math.BigInteger;
import java.util.Set;
import org.springframework.context.support.EmbeddedValueResolutionSupport;
import org.springframework.format.AnnotationFormatterFactory;
import org.springframework.format.Formatter;
import org.springframework.format.Parser;
import org.springframework.format.Printer;
import org.springframework.format.annotation.NumberFormat;
import org.springframework.format.annotation.NumberFormat.Style;
import org.springframework.format.number.CurrencyStyleFormatter;
import org.springframework.format.number.NumberStyleFormatter;
import org.springframework.format.number.PercentStyleFormatter;
import org.springframework.util.StringUtils;

public final class NumberFormatAnnotationFormatterFactory
		extends EmbeddedValueResolutionSupport implements AnnotationFormatterFactory<NumberFormat> { // ①

	private static final Set<Class<?>> FIELD_TYPES = Set.of(Short.class, // ②
			Integer.class, Long.class, Float.class, Double.class,
			BigDecimal.class, BigInteger.class);

	@Override
	public Set<Class<?>> getFieldTypes() {
		return FIELD_TYPES;
	}

	@Override
	public Printer<Number> getPrinter(NumberFormat annotation, Class<?> fieldType) { // ③
		return configureFormatterFrom(annotation, fieldType);
	}

	@Override
	public Parser<Number> getParser(NumberFormat annotation, Class<?> fieldType) { // ④
		return configureFormatterFrom(annotation, fieldType);
	}

	private Formatter<Number> configureFormatterFrom(NumberFormat annotation, Class<?> fieldType) {
		String pattern = resolveEmbeddedValue(annotation.pattern()); // Resolve pattern if needed
		if (StringUtils.hasText(pattern)) {
			return new NumberStyleFormatter(pattern);
		}
		// else
		Style style = annotation.style();
		if (style == Style.PERCENT) {
			return new PercentStyleFormatter();
		}
		else if (style == Style.CURRENCY) {
			return new CurrencyStyleFormatter();
		}
		// else
		return new NumberStyleFormatter(); // Default style
	}
}
```

① `AnnotationFormatterFactory`를 구현합니다.
② `@NumberFormat`을 지원하는 필드 타입을 반환합니다.
③ 어노테이션된 필드의 값을 출력하기 위한 `Printer`를 반환합니다.
④ 어노테이션된 필드의 클라이언트 값을 파싱하기 위한 `Parser`를 반환합니다.

```kotlin
// Kotlin
import org.springframework.context.support.EmbeddedValueResolutionSupport
import org.springframework.format.AnnotationFormatterFactory
import org.springframework.format.Formatter
import org.springframework.format.Parser
import org.springframework.format.Printer
import org.springframework.format.annotation.NumberFormat
import org.springframework.format.annotation.NumberFormat.Style
import org.springframework.format.number.CurrencyStyleFormatter
import org.springframework.format.number.NumberStyleFormatter
import org.springframework.format.number.PercentStyleFormatter
import org.springframework.util.StringUtils
import java.lang.annotation.Annotation // Import needed for explicit type
import java.math.BigDecimal
import java.math.BigInteger

class NumberFormatAnnotationFormatterFactory :
    EmbeddedValueResolutionSupport(), AnnotationFormatterFactory<NumberFormat> { // ①

    companion object {
        private val FIELD_TYPES: Set<Class<*>> = setOf( // ② Use setOf for immutable set
            Short::class.javaObjectType, Integer::class.javaObjectType, Long::class.javaObjectType,
            Float::class.javaObjectType, Double::class.javaObjectType,
            BigDecimal::class.java, BigInteger::class.java
        )
    }

    override fun getFieldTypes(): Set<Class<*>> {
        return FIELD_TYPES
    }

    override fun getPrinter(annotation: NumberFormat, fieldType: Class<*>): Printer<Number> { // ③
        return configureFormatterFrom(annotation, fieldType)
    }

    override fun getParser(annotation: NumberFormat, fieldType: Class<*>): Parser<Number> { // ④
        return configureFormatterFrom(annotation, fieldType)
    }

    private fun configureFormatterFrom(annotation: NumberFormat, fieldType: Class<*>): Formatter<Number> {
        val pattern = resolveEmbeddedValue(annotation.pattern) // Resolve pattern if needed
        if (StringUtils.hasText(pattern)) {
            return NumberStyleFormatter(pattern)
        }

        return when (annotation.style) { // Use when expression
            Style.PERCENT -> PercentStyleFormatter()
            Style.CURRENCY -> CurrencyStyleFormatter()
            else -> NumberStyleFormatter() // Default style
        }
    }
}
```

포매팅을 트리거하려면 다음 예제와 같이 필드에 `@NumberFormat`으로 어노테이션을 달 수 있습니다:

```java
// Java
public class MyModel {

	@NumberFormat(style=Style.CURRENCY)
	private BigDecimal decimal;
}
```

```kotlin
// Kotlin
class MyModel {
    // Apply the annotation to the property
    @field:NumberFormat(style = Style.CURRENCY) // Use annotation use-site target for properties
    var decimal: BigDecimal? = null
}
```

**포맷 어노테이션 API (Format Annotation API)**

`org.springframework.format.annotation` 패키지에 이식 가능한 포맷 어노테이션 API가 존재합니다.

`@NumberFormat`을 사용하여 `Double` 및 `Long`과 같은 `Number` 필드를 포맷하고, `@DurationFormat`을 사용하여 ISO-8601 및 단순화된 스타일로 `Duration` 필드를 포맷하고, `@DateTimeFormat`을 사용하여 `java.util.Date`, `java.util.Calendar`, `Long` (밀리초 타임스탬프용)과 같은 필드 및 JSR-310 `java.time` 타입을 포맷할 수 있습니다.

다음 예제는 `@DateTimeFormat`을 사용하여 `java.util.Date`를 ISO 날짜(`yyyy-MM-dd`)로 포맷합니다:

```java
// Java
import java.util.Date;
import org.springframework.format.annotation.DateTimeFormat;
import org.springframework.format.annotation.DateTimeFormat.ISO;

public class MyModel {

	@DateTimeFormat(iso=ISO.DATE)
	private Date date;
}
```

```kotlin
// Kotlin
import java.util.Date
import org.springframework.format.annotation.DateTimeFormat
import org.springframework.format.annotation.DateTimeFormat.ISO

class MyModel {
    // Apply annotation to the property
    @field:DateTimeFormat(iso = ISO.DATE)
    var date: Date? = null
}
```

*스타일 기반 포매팅 및 파싱은 자바 런타임에 따라 변경될 수 있는 로케일 민감(locale-sensitive) 패턴에 의존합니다.*

*특히, 날짜, 시간 또는 숫자 파싱 및 포매팅에 의존하는 애플리케이션은 JDK 20 이상에서 실행될 때 호환되지 않는 동작 변경을 겪을 수 있습니다.*

*ISO 표준화된 형식이나 제어하는 구체적인 패턴을 사용하면 시스템 독립적이고 로케일 독립적인 날짜, 시간 및 숫자 값의 안정적인 파싱 및 포매팅이 가능합니다.*

`*@DateTimeFormat`의 경우, 폴백 패턴 사용도 호환성 문제를 해결하는 데 도움이 될 수 있습니다.*

**FormatterRegistry SPI**

`FormatterRegistry`는 포맷터와 변환기를 등록하기 위한 SPI입니다.

`FormattingConversionService`는 대부분의 환경에 적합한 `FormatterRegistry` 구현입니다.

예를 들어, `FormattingConversionServiceFactoryBean`을 사용하여 프로그래밍 방식이나 선언적으로 이 변형을 스프링 빈으로 구성할 수 있습니다.

이 구현은 또한 `ConversionService`를 구현하므로 스프링의 `DataBinder` 및 스프링 표현 언어(SpEL)와 함께 사용하도록 직접 구성할 수 있습니다.

다음 목록은 `FormatterRegistry` SPI를 보여줍니다:

```java
package org.springframework.format;

import java.lang.annotation.Annotation;
import org.springframework.core.convert.converter.ConverterRegistry;

public interface FormatterRegistry extends ConverterRegistry { // Extends ConverterRegistry

	void addPrinter(Printer<?> printer);

	void addParser(Parser<?> parser);

	void addFormatter(Formatter<?> formatter);

	void addFormatterForFieldType(Class<?> fieldType, Formatter<?> formatter);

	void addFormatterForFieldType(Class<?> fieldType, Printer<?> printer, Parser<?> parser);

	void addFormatterForFieldAnnotation(AnnotationFormatterFactory<? extends Annotation> annotationFormatterFactory);
}
```

앞의 목록에서 보여주듯이, 필드 타입 또는 어노테이션별로 포맷터를 등록할 수 있습니다.

`FormatterRegistry` SPI를 사용하면 이러한 구성을 컨트롤러 전체에 복제하는 대신 포매팅 규칙을 중앙에서 구성할 수 있습니다.

예를 들어, 모든 날짜 필드가 특정 방식으로 포맷되도록 하거나 특정 어노테이션이 있는 필드가 특정 방식으로 포맷되도록 할 수 있습니다.

공유 `FormatterRegistry`를 사용하면 이러한 규칙을 한 번 정의하면 포매팅이 필요할 때마다 적용됩니다.

**FormatterRegistrar SPI**

`FormatterRegistrar`는 `FormatterRegistry`를 통해 포맷터와 변환기를 등록하기 위한 SPI입니다. 다음 목록은 해당 인터페이스 정의를 보여줍니다:

```java
package org.springframework.format;

public interface FormatterRegistrar {

	void registerFormatters(FormatterRegistry registry);
}
```

`FormatterRegistrar`는 날짜 포매팅과 같은 주어진 포매팅 범주에 대해 여러 관련 변환기 및 포맷터를 등록할 때 유용합니다.

또한 선언적 등록이 불충분한 경우(예: 포맷터가 자체 `<T>`와 다른 특정 필드 타입 아래에 인덱싱되어야 하거나 `Printer`/`Parser` 쌍을 등록할 때)에도 유용할 수 있습니다.

---

**전체 주제: 스프링 필드 포매팅 (Spring Field Formatting)**

이 부분은 `core.convert`의 `Converter`만으로는 부족한 **클라이언트 환경(웹, 데스크톱 등)에서의 요구사항**, 즉, 객체 데이터를 **지역화된(Locale-specific) 문자열로 변환(출력, print)** 하거나, 사용자가 입력한 **지역화된 문자열을 다시 객체로 변환(파싱, parse)** 하는 기능을 제공하는 **`Formatter` SPI** 와 관련 어노테이션에 대한 내용입니다.

**핵심 아이디어:** `Converter`는 객체 타입 간의 일반 변환을 담당하고, `Formatter`는 객체를 사용자가 보거나 입력하는 **문자열** 형태로, 특히 **지역화(다국어, 숫자/날짜 형식 등)** 를 고려하여 변환하는 역할을 담당한다!

---

**1. 왜 별도의 포매팅(Formatting)이 필요한가? (`Converter`의 한계)**

- **`Converter<S, T>`:** 주로 **객체 타입 간의 변환**(예: `Date` -> `Long`, `String` -> `Integer`)이나 **내부적인 데이터 표현 변환**에 초점을 맞춥니다. 사용자가 직접 보거나 입력하는 형태, 특히 지역화(Locale)를 고려한 형식 변환은 직접 다루지 않습니다.
- **클라이언트 환경의 요구사항:** 웹이나 데스크톱 애플리케이션에서는 다음과 같은 변환이 필수적입니다.
  - **객체 -> 문자열 (출력, Printing):** 객체(예: `Date`, `BigDecimal`, `Duration`)를 특정 **로케일(언어/국가)** 에 맞는 **문자열 형식**(예: 날짜 형식 'yyyy년 MM월 dd일', 숫자 형식 '#,###.##', 통화 기호 등)으로 변환하여 화면에 보여줘야 합니다.
  - **문자열 -> 객체 (파싱, Parsing):** 사용자가 입력한 **문자열**(예: "2023-12-25", "1,234.56")을 다시 원래의 **객체 타입**(`Date`, `BigDecimal`)으로 변환해야 합니다. 이때도 사용자의 로케일을 고려해야 할 수 있습니다.
- **`Formatter`의 역할:** 스프링의 `Formatter` SPI는 바로 이러한 **객체 <-> 지역화된 문자열** 간의 변환 요구사항을 충족시키기 위해 만들어졌습니다. `PropertyEditor`의 더 현대적이고 강력한 대안입니다.

---

**2. `Formatter` SPI (Service Provider Interface)**

- `org.springframework.format.Formatter<T>` 인터페이스는 포매팅 로직을 구현하기 위한 SPI입니다. (`T`는 포매팅할 객체의 타입)
- 이 인터페이스는 두 개의 다른 인터페이스를 상속(extends)합니다:
  - **`Printer<T>`:**
    - `String print(T fieldValue, Locale locale)` 메소드를 가집니다.
    - **역할:** 주어진 객체(`fieldValue`)를 지정된 `locale`에 맞는 **문자열**로 변환(출력)합니다.
  - **`Parser<T>`:**
    - `T parse(String clientValue, Locale locale) throws ParseException` 메소드를 가집니다.
    - **역할:** 클라이언트(사용자)가 입력한 `clientValue` 문자열을 지정된 `locale`을 고려하여 다시 **객체 `T`** 로 변환(파싱)합니다. 파싱 실패 시 `ParseException` 등을 던져야 합니다.
- **구현:** 개발자는 `Formatter<T>` 인터페이스를 구현하고, `print()`와 `parse()` 메소드에 해당 타입(`T`)에 대한 문자열 변환 및 객체 변환 로직을 작성합니다. 구현체는 **스레드 안전(thread-safe)** 해야 합니다.
- **내장 Formatter:** 스프링은 `org.springframework.format.number` 및 `org.springframework.format.datetime` 패키지에 자주 사용되는 포맷터들을 미리 제공합니다.
  - **숫자:** `NumberStyleFormatter` (일반 숫자), `CurrencyStyleFormatter` (통화), `PercentStyleFormatter` (백분율) - 내부적으로 `java.text.NumberFormat` 사용.
  - **날짜/시간:** `DateFormatter` (`java.util.Date`용, `java.text.DateFormat` 사용), `DurationFormatter` (`java.time.Duration`용) 등.

---

**3. 어노테이션 기반 포매팅 (`AnnotationFormatterFactory`)**

필드 값에 특정 어노테이션을 붙여서 포매팅 규칙을 적용하는 방식입니다. (예: 필드 위에 `@NumberFormat(style = Style.CURRENCY)` 붙이기)

- **`AnnotationFormatterFactory<A extends Annotation>` 인터페이스:** 특정 **어노테이션(A)** 과 해당 어노테이션이 적용될 수 있는 **필드 타입들**을 연결하고, 어노테이션의 속성 값에 따라 적절한 **`Printer`와 `Parser`를 생성**해주는 팩토리 인터페이스입니다.
- **구현:**
  1. `A`를 포매팅 규칙을 지정할 어노테이션 타입으로 지정합니다. (예: `NumberFormat.class`)
  2. `getFieldTypes()`: 이 어노테이션이 적용될 수 있는 필드의 `Class` 객체 목록(`Set`)을 반환합니다. (예: `Integer.class`, `BigDecimal.class` 등)
  3. `getPrinter(A annotation, Class<?> fieldType)`: 주어진 어노테이션(`annotation`) 정보와 필드 타입(`fieldType`)을 보고 적절한 `Printer` 객체를 생성하여 반환합니다.
  4. `getParser(A annotation, Class<?> fieldType)`: 주어진 어노테이션 정보와 필드 타입을 보고 적절한 `Parser` 객체를 생성하여 반환합니다.
- **예시 (`NumberFormatAnnotationFormatterFactory`):** 스프링에 내장된 이 팩토리는 `@NumberFormat` 어노테이션을 처리합니다. `@NumberFormat`의 `style` 속성(CURRENCY, PERCENT 등)이나 `pattern` 속성 값을 보고 그에 맞는 `CurrencyStyleFormatter`, `PercentStyleFormatter`, `NumberStyleFormatter` 등을 생성하여 반환합니다.

---

**4. 포맷 어노테이션 API (`org.springframework.format.annotation`)**

스프링은 필드에 직접 붙여서 포매팅 규칙을 지정할 수 있는 편리한 표준 어노테이션들을 제공합니다.

- **`@NumberFormat`:** 숫자 필드(`Integer`, `Long`, `BigDecimal` 등)의 포맷을 지정합니다. `style` (CURRENCY, PERCENT, NUMBER) 또는 `pattern` (예: `#,###.##`) 속성을 사용합니다.
- **`@DateTimeFormat`:** 날짜/시간 관련 필드(`java.util.Date`, `java.time.LocalDate`, `Long` 등)의 포맷을 지정합니다. `iso` (미리 정의된 ISO 형식), `pattern` (예: `yyyy-MM-dd HH:mm:ss`), `style` (예: "S-" - Short Date) 등의 속성을 사용합니다.
- **`@DurationFormat`:** `java.time.Duration` 필드의 포맷을 지정합니다. `iso`, `style`, `pattern` 속성을 사용합니다.
- **사용법:** 포매팅을 적용하고 싶은 필드 선언 위에 해당 어노테이션을 붙이고 필요한 속성 값을 설정합니다.

    ```java
    public class MyModel {
        @NumberFormat(style = Style.CURRENCY) // 통화 형식으로 포맷
        private BigDecimal price;
    
        @DateTimeFormat(iso = ISO.DATE) // ISO 날짜 형식 (yyyy-MM-dd)으로 포맷
        private Date registerDate;
    }
    ```

- **주의 (JDK 20+):** 자바 런타임 버전에 따라 기본 로케일 민감 패턴이 변경될 수 있으므로, 안정적인 포매팅/파싱을 위해서는 **ISO 표준 형식이나 명시적인 패턴을 사용하는 것이 권장**됩니다.

---

**5. FormatterRegistry SPI: 포맷터와 변환기 등록소**

- **`FormatterRegistry` 인터페이스:** `Formatter`와 `Converter`를 **등록**하고 관리하는 역할을 하는 SPI입니다. (`ConverterRegistry` 인터페이스를 상속합니다.)
- **등록 방법:** 필드 타입별(`addFormatterForFieldType`), 어노테이션별(`addFormatterForFieldAnnotation`) 등으로 포맷터를 등록할 수 있습니다. 일반 `Converter`도 등록 가능합니다.
- **구현체 (`FormattingConversionService`):** 스프링은 `FormattingConversionService`라는 클래스를 제공하는데, 이 클래스는 `FormatterRegistry`와 `ConversionService` 인터페이스를 **모두 구현**합니다. 따라서 이 구현체를 사용하면 **포맷터와 컨버터를 한 곳에서 등록하고, 통합된 `ConversionService` API를 통해 사용할 수 있습니다.**
- **구성:** `FormattingConversionServiceFactoryBean` 등을 사용하여 스프링 빈으로 구성할 수 있습니다. 이렇게 구성된 `ConversionService` (실제로는 `FormattingConversionService`)는 스프링 MVC의 데이터 바인딩이나 SpEL 등에서 자동으로 사용됩니다.

---

**6. FormatterRegistrar SPI: 등록 로직 캡슐화**

- **`FormatterRegistrar` 인터페이스:** 특정 포매팅 규칙 그룹(예: 날짜 관련 포맷터들)이나 복잡한 등록 로직을 **하나의 클래스로 캡슐화**하기 위한 SPI입니다.
- **구현 메소드:** `registerFormatters(FormatterRegistry registry)` 메소드를 구현하여, 파라미터로 전달받은 `FormatterRegistry`에 필요한 포맷터나 컨버터를 등록하는 코드를 작성합니다.
- **활용:**
  - 관련된 포맷터/컨버터들을 모듈화하여 관리하기 좋습니다.
  - `FormattingConversionServiceFactoryBean`의 `formatterRegistrars` 속성이나 스프링 MVC의 `WebMvcConfigurer` 등을 통해 등록하여 사용할 수 있습니다.

---

**7. 스프링 MVC에서 포매팅 구성하기**

- 스프링 MVC 환경에서는 `WebMvcConfigurer` 인터페이스의 `addFormatters(FormatterRegistry registry)` 메소드를 구현하여 커스텀 `Formatter`, `Converter`, `FormatterRegistrar` 등을 **전역적으로 등록**하는 것이 일반적입니다. 이렇게 등록된 포맷터와 컨버터는 컨트롤러의 데이터 바인딩 과정에서 자동으로 사용됩니다.

**요약:**

스프링 필드 포매팅은 `core.convert`의 타입 변환 시스템을 확장하여, 특히 **클라이언트 환경(웹 등)에서의 객체 <-> 지역화된 문자열 변환**을 위한 `Formatter` SPI를 제공합니다. `@NumberFormat`, `@DateTimeFormat` 같은 어노테이션을 사용하여 필드에 직접 포매팅 규칙을 적용할 수 있으며, 이러한 어노테이션은 `AnnotationFormatterFactory`를 통해 처리됩니다. `FormatterRegistry`(주로 `FormattingConversionService` 구현체)에 커스텀 `Formatter`나 `Converter`를 등록하여 시스템 전체의 포매팅/변환 규칙을 설정할 수 있으며, `FormatterRegistrar`를 사용하여 등록 로직을 캡슐화할 수 있습니다. 이 모든 것은 스프링 MVC의 데이터 바인딩 과정 등에서 활용되어 사용자 입력 처리 및 화면 출력을 편리하게 만듭니다.
