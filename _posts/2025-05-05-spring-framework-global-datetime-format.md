---
title: Spring Framework Configuring a Global Date and Time Format
description: 
author: laze
date: 2025-05-05 00:00:07 +0900
categories: [Dev, SpringBoot]
tags: [SpringBoot]
---
**Configuring a Global Date and Time Format**

기본적으로 `@DateTimeFormat`으로 어노테이션되지 않은 날짜 및 시간 필드는 `DateFormat.SHORT` 스타일을 사용하여 문자열에서 변환됩니다.

원한다면, 자신만의 전역 형식을 정의하여 이를 변경할 수 있습니다.

그렇게 하려면, 스프링이 기본 포맷터를 등록하지 않도록 해야 합니다. 대신, 다음의 도움을 받아 포맷터를 수동으로 등록할 수 있습니다.

- `org.springframework.format.datetime.standard.DateTimeFormatterRegistrar`
- `org.springframework.format.datetime.DateFormatterRegistrar`

예를 들어, 다음 구성은 전역 `yyyyMMdd` 형식을 등록합니다:

**Java**

```java
import org.springframework.context.annotation.Bean;
import org.springframework.context.annotation.Configuration;
import org.springframework.format.datetime.DateFormatter;
import org.springframework.format.datetime.DateFormatterRegistrar;
import org.springframework.format.datetime.standard.DateTimeFormatterRegistrar;
import org.springframework.format.number.NumberFormatAnnotationFormatterFactory;
import org.springframework.format.support.DefaultFormattingConversionService;
import org.springframework.format.support.FormattingConversionService;

import java.time.format.DateTimeFormatter;

@Configuration
public class ApplicationConfiguration {

	@Bean
	public FormattingConversionService conversionService() {

		// DefaultFormattingConversionService를 사용하되 기본값은 등록하지 않음
		DefaultFormattingConversionService conversionService =
				new DefaultFormattingConversionService(false);

		// @NumberFormat이 여전히 지원되도록 보장
		conversionService.addFormatterForFieldAnnotation(
				new NumberFormatAnnotationFormatterFactory());

		// 특정 전역 형식으로 JSR-310 날짜 변환 등록
		DateTimeFormatterRegistrar dateTimeRegistrar = new DateTimeFormatterRegistrar();
		dateTimeRegistrar.setDateFormatter(DateTimeFormatter.ofPattern("yyyyMMdd"));
		dateTimeRegistrar.registerFormatters(conversionService);

		// 특정 전역 형식으로 날짜 변환 등록
		DateFormatterRegistrar dateRegistrar = new DateFormatterRegistrar();
		dateRegistrar.setFormatter(new DateFormatter("yyyyMMdd"));
		dateRegistrar.registerFormatters(conversionService);

		return conversionService;
	}
}
```

**Kotlin**

```kotlin
import org.springframework.context.annotation.Bean
import org.springframework.context.annotation.Configuration
import org.springframework.format.datetime.DateFormatter
import org.springframework.format.datetime.DateFormatterRegistrar
import org.springframework.format.datetime.standard.DateTimeFormatterRegistrar
import org.springframework.format.number.NumberFormatAnnotationFormatterFactory
import org.springframework.format.support.DefaultFormattingConversionService
import org.springframework.format.support.FormattingConversionService
import java.time.format.DateTimeFormatter

@Configuration
class ApplicationConfiguration {

    @Bean
    fun conversionService(): FormattingConversionService {

        // DefaultFormattingConversionService를 사용하되 기본값은 등록하지 않음
        val conversionService = DefaultFormattingConversionService(false)

        // @NumberFormat이 여전히 지원되도록 보장
        conversionService.addFormatterForFieldAnnotation(NumberFormatAnnotationFormatterFactory())

        // 특정 전역 형식으로 JSR-310 날짜 변환 등록
        val dateTimeRegistrar = DateTimeFormatterRegistrar()
        dateTimeRegistrar.setDateFormatter(DateTimeFormatter.ofPattern("yyyyMMdd"))
        dateTimeRegistrar.registerFormatters(conversionService)

        // 특정 전역 형식으로 날짜 변환 등록
        val dateRegistrar = DateFormatterRegistrar()
        dateRegistrar.setFormatter(DateFormatter("yyyyMMdd"))
        dateRegistrar.registerFormatters(conversionService)

        return conversionService
    }
}
```

**Xml** (참고: XML 방식은 직접 Registrar를 빈으로 등록하고 conversionService에 주입하는 방식으로 구현합니다.)

```xml
<bean id="conversionService" class="org.springframework.format.support.FormattingConversionServiceFactoryBean">
    <property name="registerDefaultFormatters" value="false" /> <!-- 기본 포맷터 등록 안함 -->
    <property name="formatters">
        <set>
            <!-- @NumberFormat 지원 추가 -->
            <bean class="org.springframework.format.number.NumberFormatAnnotationFormatterFactory"/>
        </set>
    </property>
    <property name="formatterRegistrars">
        <set>
             <!-- JSR-310 전역 포맷 등록 -->
            <bean class="org.springframework.format.datetime.standard.DateTimeFormatterRegistrar">
                <property name="dateFormatter">
                    <bean class="java.time.format.DateTimeFormatter" factory-method="ofPattern">
                         <constructor-arg value="yyyyMMdd"/>
                    </bean>
                </property>
            </bean>
             <!-- java.util.Date 전역 포맷 등록 -->
            <bean class="org.springframework.format.datetime.DateFormatterRegistrar">
                <property name="formatter">
                    <bean class="org.springframework.format.datetime.DateFormatter">
                        <constructor-arg value="yyyyMMdd"/>
                    </bean>
                </property>
            </bean>
        </set>
    </property>
</bean>
```

---

**전체 주제: 전역 날짜 및 시간 형식 구성하기**

이 부분은 개별 필드마다 `@DateTimeFormat` 어노테이션을 붙이는 대신, 애플리케이션 전체에서 **어노테이션이 없는** 날짜/시간 타입 필드에 대해 **기본적으로 사용할 포맷(Format)** 을 지정하는 방법을 설명합니다.

**핵심 아이디어:** 매번 `@DateTimeFormat` 붙이기 귀찮다! 우리 앱에서는 기본적으로 날짜는 'yyyyMMdd' 형식으로 통일하자!

---

**1. 기본 동작 방식:**

- 스프링은 기본적으로 `@DateTimeFormat` 어노테이션이 **없는** 날짜/시간 관련 필드(예: `java.util.Date`, `java.time.LocalDate`)를 문자열과 변환할 때, **`DateFormat.SHORT` 스타일** (로케일에 따라 다르지만 보통 "yy. M. d." 또는 "M/d/yy" 같은 매우 짧은 형식)을 사용합니다.
- 이 기본 동작은 대부분의 애플리케이션 요구사항과 맞지 않을 수 있습니다.

---

**2. 전역 형식 설정 방법: 기본 포맷터 등록 비활성화 및 수동 등록**

전역 날짜/시간 형식을 설정하려면 다음 단계를 따릅니다:

1. **기본 포맷터 등록 비활성화:** 스프링이 내부적으로 등록하는 기본 날짜/시간 포맷터들의 등록을 **막아야 합니다.** 이렇게 하지 않으면 우리가 설정하려는 전역 형식이 우선순위에서 밀릴 수 있습니다.
  - **Java Config:** `DefaultFormattingConversionService` 객체를 생성할 때 생성자 인자로 `false`를 전달합니다 (`new DefaultFormattingConversionService(false)`). 이 `false` 값은 "기본 포맷터를 등록하지 말라"는 의미입니다.
  - **XML Config:** `FormattingConversionServiceFactoryBean` 빈을 정의할 때 `registerDefaultFormatters` 속성을 `false`로 설정합니다 (`<property name="registerDefaultFormatters" value="false"/>`).
2. **`FormatterRegistrar`를 사용하여 전역 포맷터 수동 등록:** 스프링은 날짜/시간 포맷터를 편리하게 등록할 수 있도록 도와주는 두 가지 `FormatterRegistrar` 구현체를 제공합니다. 이들을 사용하여 원하는 전역 형식을 가진 포맷터를 등록합니다.
  - **`org.springframework.format.datetime.standard.DateTimeFormatterRegistrar`:**
    - **대상:** **JSR-310 `java.time`** 패키지의 날짜/시간 타입들 (예: `LocalDate`, `LocalTime`, `LocalDateTime`, `ZonedDateTime`, `OffsetDateTime` 등)
    - **설정 방법:** `setDateFormatter()`, `setTimeFormatter()`, `setDateTimeFormatter()` 등의 메소드를 사용하여 `java.time.format.DateTimeFormatter` 객체(원하는 패턴으로 생성)를 설정합니다.
    - **등록:** `registerFormatters(conversionService)` 메소드를 호출하여 설정된 포맷터들을 `ConversionService`(정확히는 `FormatterRegistry`)에 등록합니다.
  - **`org.springframework.format.datetime.DateFormatterRegistrar`:**
    - **대상:** 구형 **`java.util.Date`**, `java.util.Calendar`, `Long` (타임스탬프) 타입.
    - **설정 방법:** `setFormatter()` 메소드를 사용하여 원하는 패턴으로 생성된 `org.springframework.format.datetime.DateFormatter` 객체를 설정합니다.
    - **등록:** `registerFormatters(conversionService)` 메소드를 호출하여 등록합니다.
3. **(선택 사항) 다른 기본 포맷터 재등록:** 기본 포맷터 등록을 비활성화했기 때문에, 날짜/시간 외의 다른 기본 포맷터(예: `@NumberFormat` 처리기)들이 필요하다면 **수동으로 다시 등록**해주어야 합니다.
  - 예시 코드에서는 `@NumberFormat`을 처리하는 `NumberFormatAnnotationFormatterFactory`를 `conversionService.addFormatterForFieldAnnotation(...)`을 통해 다시 등록해주고 있습니다.

---

**3. 예시 코드 분석 (Java Config):**

```java
@Configuration
public class ApplicationConfiguration {

    @Bean
    public FormattingConversionService conversionService() { // ★ 빈 이름은 "conversionService" ★

        // 1. 기본 포맷터 등록 비활성화
        DefaultFormattingConversionService conversionService =
                new DefaultFormattingConversionService(false); // false 전달

        // 3. (선택) 필요한 다른 기본 포맷터(예: @NumberFormat 처리기) 수동 등록
        conversionService.addFormatterForFieldAnnotation(
                new NumberFormatAnnotationFormatterFactory());

        // 2a. JSR-310 타입용 전역 포맷 등록 (yyyyMMdd)
        DateTimeFormatterRegistrar dateTimeRegistrar = new DateTimeFormatterRegistrar();
        dateTimeRegistrar.setDateFormatter(DateTimeFormatter.ofPattern("yyyyMMdd")); // LocalDate 등에 적용될 포맷
        // dateTimeRegistrar.setTimeFormatter(...); // 필요시 LocalTime 포맷 설정
        // dateTimeRegistrar.setDateTimeFormatter(...); // 필요시 LocalDateTime 포맷 설정
        dateTimeRegistrar.registerFormatters(conversionService); // ConversionService에 등록

        // 2b. java.util.Date 타입용 전역 포맷 등록 (yyyyMMdd)
        DateFormatterRegistrar dateRegistrar = new DateFormatterRegistrar();
        dateRegistrar.setFormatter(new DateFormatter("yyyyMMdd")); // Date 등에 적용될 포맷
        dateRegistrar.registerFormatters(conversionService); // ConversionService에 등록

        return conversionService;
    }
}
```

**위 설정의 효과:**

- 이제 애플리케이션 전체에서 `@DateTimeFormat` 어노테이션이 **없는** `java.time.LocalDate` 필드나 `java.util.Date` 필드는 기본적으로 `"yyyyMMdd"` 형식의 문자열과 상호 변환됩니다.
- 예를 들어, 웹 폼에서 "20231225"라고 입력된 문자열이 `LocalDate`나 `Date` 타입 필드에 자동으로 바인딩될 수 있습니다.
- 반대로 `LocalDate`나 `Date` 객체를 뷰에 출력할 때도 기본적으로 "20231225" 형식의 문자열로 변환됩니다.

---

**4. XML 설정 방식:**

XML 방식은 `FormatterRegistrar` 빈들을 직접 정의하고, `FormattingConversionServiceFactoryBean`의 `formatterRegistrars` 속성에 주입하는 방식으로 동일한 목표를 달성합니다. (예시 코드 참조)

---

**5. 웹 애플리케이션 고려 사항:**

- 스프링 MVC나 WebFlux 같은 웹 프레임워크를 사용할 때는, 단순히 `conversionService` 빈을 등록하는 것 외에 웹 계층에서 이 `ConversionService`를 사용하도록 추가적인 설정이 필요할 수 있습니다. (예: `WebMvcConfigurer`의 `addFormatters` 메소드 활용) 이는 각 웹 프레임워크 문서에서 자세히 다룹니다.

**요약:**

스프링의 기본 날짜/시간 포맷(`DateFormat.SHORT`) 대신 애플리케이션 **전체에 적용될 기본(전역) 형식**을 지정하려면, `conversionService` 빈을 구성할 때 **기본 포맷터 등록을 비활성화**하고(`DefaultFormattingConversionService(false)` 또는 `registerDefaultFormatters="false"`), **`DateTimeFormatterRegistrar`(JSR-310용)와 `DateFormatterRegistrar`(Date용)를 사용하여 원하는 형식의 포맷터를 수동으로 등록**해야 합니다. 이렇게 하면 `@DateTimeFormat` 어노테이션이 없는 날짜/시간 필드에도 지정된 전역 형식이 적용됩니다.
