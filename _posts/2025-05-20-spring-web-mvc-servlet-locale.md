---
title: Spring Web MVC - Dispatcher Servlet (Locale)
description: 
author: laze
date: 2025-05-20 00:00:03 +0900
categories: [Dev, SpringWebMVC]
tags: [SpringWebMVC]
---
**로케일 (Locale)**

Spring 아키텍처의 대부분은 Spring 웹 MVC 프레임워크와 마찬가지로 국제화를 지원합니다.

`DispatcherServlet`을 사용하면 클라이언트의 로케일을 사용하여 메시지를 자동으로 해석할 수 있습니다.

이는 `LocaleResolver` 객체를 통해 수행됩니다.

요청이 들어오면, `DispatcherServlet`은 로케일 리졸버를 찾고, 만약 찾으면 이를 사용하여 로케일을 설정하려고 시도합니다.

`RequestContext.getLocale()` 메소드를 사용하면 로케일 리졸버에 의해 해석된 로케일을 항상 검색할 수 있습니다.

자동 로케일 해석 외에도, 특정 상황(예: 요청의 파라미터 기반)에서 로케일을 변경하기 위해 핸들러 매핑에 인터셉터를 연결할 수 있습니다 (핸들러 매핑 인터셉터에 대한 자세한 정보는 인터셉션(Interception) 참조).

로케일 리졸버와 인터셉터는 `org.springframework.web.servlet.i18n` 패키지에 정의되어 있으며, 일반적인 방식으로 애플리케이션 컨텍스트에 설정됩니다.

다음은 Spring에 포함된 로케일 리졸버 선택 항목입니다.

- 시간대 (Time Zone)
- 헤더 리졸버 (Header Resolver)
- 쿠키 리졸버 (Cookie Resolver)
- 세션 리졸버 (Session Resolver)
- 로케일 인터셉터 (Locale Interceptor)

**시간대 (Time Zone)**
클라이언트의 로케일을 얻는 것 외에도, 종종 그들의 시간대를 아는 것이 유용합니다.

`LocaleContextResolver` 인터페이스는 리졸버가 시간대 정보를 포함할 수 있는 더 풍부한 `LocaleContext`를 제공할 수 있도록 `LocaleResolver`에 대한 확장을 제공합니다.

사용 가능한 경우, 사용자의 `TimeZone`은 `RequestContext.getTimeZone()` 메소드를 사용하여 얻을 수 있습니다.

시간대 정보는 Spring의 `ConversionService`에 등록된 모든 날짜/시간 변환기(Converter) 및 포맷터(Formatter) 객체에 의해 자동으로 사용됩니다.

**헤더 리졸버 (Header Resolver)**
이 로케일 리졸버는 클라이언트(예: 웹 브라우저)가 보낸 요청의 `accept-language` 헤더를 검사합니다.

일반적으로 이 헤더 필드에는 클라이언트 운영 체제의 로케일이 포함됩니다.

이 리졸버는 시간대 정보를 지원하지 않습니다.

**쿠키 리졸버 (Cookie Resolver)**
이 로케일 리졸버는 클라이언트에 존재할 수 있는 쿠키를 검사하여 `Locale` 또는 `TimeZone`이 지정되었는지 확인합니다.

만약 그렇다면, 지정된 세부 정보를 사용합니다. 이 로케일 리졸버의 속성을 사용하여 쿠키 이름과 최대 수명을 지정할 수 있습니다.

다음 예제는 `CookieLocaleResolver`를 정의합니다:

```xml
<bean id="localeResolver" class="org.springframework.web.servlet.i18n.CookieLocaleResolver">

	<property name="cookieName" value="clientlanguage"/>

	<!-- 초 단위. -1로 설정하면 쿠키는 유지되지 않음 (브라우저 종료 시 삭제) -->
	<property name="cookieMaxAge" value="100000"/>

</bean>
```

다음 표는 `CookieLocaleResolver`의 속성을 설명합니다:

**표 1. `CookieLocaleResolver` 속성**

| 속성 (Property) | 기본값 (Default) | 설명 (Description) |
| --- | --- | --- |
| `cookieName` | 클래스 이름 + `LOCALE` | 쿠키의 이름 |
| `cookieMaxAge` | 서블릿 컨테이너 기본값 | 쿠키가 클라이언트에 유지되는 최대 시간입니다. -1로 지정하면 쿠키는 유지되지 않습니다. 클라이언트가 브라우저를 종료할 때까지만 사용할 수 있습니다. |
| `cookiePath` | `/` | 쿠키의 가시성을 사이트의 특정 부분으로 제한합니다. `cookiePath`가 지정되면 쿠키는 해당 경로와 그 하위 경로에만 표시됩니다. |

**세션 리졸버 (Session Resolver)**`SessionLocaleResolver`를 사용하면 사용자의 요청과 연관될 수 있는 세션에서 `Locale` 및 `TimeZone`을 검색할 수 있습니다.

`CookieLocaleResolver`와는 대조적으로, 이 전략은 로컬에서 선택된 로케일 설정을 서블릿 컨테이너의 `HttpSession`에 저장합니다.

결과적으로, 이러한 설정은 각 세션에 대해 일시적이며 각 세션이 종료될 때 손실됩니다.

Spring Session 프로젝트와 같은 외부 세션 관리 메커니즘과의 직접적인 관계는 없습니다.

이 `SessionLocaleResolver`는 현재 `HttpServletRequest`에 대해 해당 `HttpSession` 속성을 평가하고 수정합니다.

**로케일 인터셉터 (Locale Interceptor)**`HandlerMapping` 정의 중 하나에 `LocaleChangeInterceptor`를 추가하여 로케일 변경을 활성화할 수 있습니다.

이는 요청의 파라미터를 감지하고 그에 따라 로케일을 변경하며, 디스패처의 애플리케이션 컨텍스트에 있는 `LocaleResolver`의 `setLocale` 메소드를 호출합니다.

다음 예제는 `siteLanguage`라는 이름의 파라미터를 포함하는 모든 `*.view` 리소스에 대한 호출이 이제 로케일을 변경함을 보여줍니다.

예를 들어, `www.sf.net/home.view?siteLanguage=nl` URL에 대한 요청은 사이트 언어를 네덜란드어로 변경합니다. 다음 예제는 로케일을 가로채는 방법을 보여줍니다:

```xml
<bean id="localeChangeInterceptor"
		class="org.springframework.web.servlet.i18n.LocaleChangeInterceptor">
	<property name="paramName" value="siteLanguage"/>
</bean>

<bean id="localeResolver"
		class="org.springframework.web.servlet.i18n.CookieLocaleResolver"/>

<bean id="urlMapping"
		class="org.springframework.web.servlet.handler.SimpleUrlHandlerMapping">
	<property name="interceptors">
		<list>
			<ref bean="localeChangeInterceptor"/>
		</list>
	</property>
	<property name="mappings">
		<value>/**/*.view=someController</value>
	</property>
</bean>
```

---

## Spring MVC의 다국어 지원: LocaleResolver와 LocaleChangeInterceptor

Spring MVC는 애플리케이션의 여러 부분에서 국제화를 지원하도록 설계되었습니다.

`DispatcherServlet`은 클라이언트의 **로케일(Locale)**을 기반으로 메시지(예: UI 텍스트, 에러 메시지)를 자동으로 해석하고, 적절한 언어로 된 뷰를 제공하는 등의 기능을 수행할 수 있습니다.

이 모든 것은 **`LocaleResolver`** 객체를 통해 이루어집니다.

### 1. 로케일(Locale)이란?

로케일은 특정 지역의 언어, 국가, 문화적 관습 등을 나타내는 정보입니다. (예: `ko_KR` - 한국어/대한민국, `en_US` - 영어/미국, `fr_FR` - 프랑스어/프랑스) Spring MVC는 이 로케일 정보를 사용하여 사용자에게 맞는 언어로 콘텐츠를 제공합니다.

### 2. `LocaleResolver`의 역할

- **요청으로부터 로케일 결정:** `DispatcherServlet`은 요청이 들어올 때마다 등록된 `LocaleResolver`를 사용하여 해당 요청에 대한 로케일을 결정하려고 시도합니다.
- **로케일 설정 및 조회:** 결정된 로케일은 현재 요청 처리 과정에서 사용될 수 있도록 설정되며, `RequestContext.getLocale()`과 같은 메소드를 통해 언제든지 조회할 수 있습니다. 이 정보는 뷰 렌더링, 메시지 소스(MessageSource)를 통한 메시지 해석 등에 활용됩니다.

### 3. 주요 `LocaleResolver` 구현체들

Spring은 다양한 방식으로 로케일을 결정할 수 있는 여러 `LocaleResolver` 구현체들을 제공합니다. 이들은 `org.springframework.web.servlet.i18n` 패키지에 있으며, Spring 설정 파일(Java Config 또는 XML)에 빈으로 등록하여 사용합니다.

| `LocaleResolver` 구현체 | 로케일 결정 방식 | 시간대 지원 여부 |
| --- | --- | --- |
| `AcceptHeaderLocaleResolver` | **(기본값)** 클라이언트(웹 브라우저)가 보낸 HTTP 요청 헤더의 `Accept-Language` 값을 검사하여 로케일을 결정합니다. (예: 브라우저 언어 설정에 따라) | X |
| `CookieLocaleResolver` | 클라이언트의 쿠키에 저장된 로케일 정보를 읽어 사용합니다. 사용자가 선택한 로케일을 쿠키에 저장해두면 다음 방문 시에도 유지될 수 있습니다. 쿠키 이름, 유효 기간 등을 설정할 수 있습니다. | O (TimeZone도 가능) |
| `SessionLocaleResolver` | 사용자의 HTTP 세션(`HttpSession`)에 저장된 로케일 정보를 읽어 사용합니다. 세션이 유지되는 동안만 로케일 설정이 유효하며, 세션이 만료되면 사라집니다. | O (TimeZone도 가능) |
| `FixedLocaleResolver`  | 항상 고정된 특정 로케일을 사용하도록 설정합니다. (예: 항상 `ko_KR` 사용) | X |

**예시 (XML 기반 `CookieLocaleResolver` 설정):**

```xml
<bean id="localeResolver" class="org.springframework.web.servlet.i18n.CookieLocaleResolver">
    <property name="cookieName" value="clientLanguage"/> <!-- 쿠키 이름 설정 -->
    <property name="defaultLocale" value="en"/>          <!-- 기본 로케일 (쿠키에 값 없을 시) -->
    <property name="cookieMaxAge" value="360000"/>       <!-- 쿠키 유효 기간 (초 단위) -->
    <!-- <property name="cookiePath" value="/mypage"/> --> <!-- 쿠키 적용 경로 (옵션) -->
</bean>
```

**Java Config 예시 (`CookieLocaleResolver`):**

```java
import org.springframework.context.annotation.Bean;
import org.springframework.context.annotation.Configuration;
import org.springframework.web.servlet.LocaleResolver;
import org.springframework.web.servlet.i18n.CookieLocaleResolver;
import java.util.Locale;

@Configuration
public class WebConfig { // implements WebMvcConfigurer 등과 함께 사용될 수 있음

    @Bean
    public LocaleResolver localeResolver() {
        CookieLocaleResolver clr = new CookieLocaleResolver("clientLanguage"); // 생성자에서 쿠키 이름 직접 지정 가능
        clr.setDefaultLocale(Locale.ENGLISH); // 기본 로케일
        clr.setCookieMaxAge(360000); // 쿠키 유효 기간
        // clr.setCookiePath("/mypage");
        return clr;
    }
}
```

### 4. 시간대(Time Zone) 정보

로케일과 더불어 클라이언트의 시간대를 아는 것도 중요할 때가 많습니다 (특히 날짜/시간 데이터 처리 시).

- **`LocaleContextResolver` 인터페이스:** `LocaleResolver`를 확장한 인터페이스로, 로케일 정보뿐만 아니라 시간대 정보까지 포함하는 `LocaleContext` 객체를 제공할 수 있도록 합니다. `CookieLocaleResolver`와 `SessionLocaleResolver`는 이 인터페이스를 구현합니다.
- **시간대 조회:** `RequestContext.getTimeZone()` 메소드를 통해 현재 요청에 대한 시간대를 얻을 수 있습니다.
- **활용:** 이렇게 얻은 시간대 정보는 Spring의 `ConversionService`에 등록된 날짜/시간 관련 `Converter`나 `Formatter`에 의해 자동으로 사용되어, 사용자 시간대에 맞는 날짜/시간 표시를 가능하게 합니다.

### 5. `LocaleChangeInterceptor`: 사용자가 로케일을 변경하도록 허용

애플리케이션이 자동으로 로케일을 결정하는 것 외에, 사용자가 직접 웹사이트의 언어를 선택하여 변경할 수 있는 기능을 제공하고 싶을 때 `LocaleChangeInterceptor`를 사용합니다.

- **역할:** HTTP 요청 파라미터에 특정 이름(예: `lang`, `siteLanguage`)으로 로케일 값이 전달되면, 이를 감지하여 현재 `LocaleResolver`의 `setLocale` 메소드를 호출함으로써 로케일을 변경합니다.
- **설정:** `HandlerMapping`에 이 인터셉터를 등록해야 합니다. (MVC Config의 `addInterceptors` 사용)

**예시 (XML 기반 설정):**

```xml
<!-- 인터셉터 빈 정의 -->
<bean id="localeChangeInterceptor" class="org.springframework.web.servlet.i18n.LocaleChangeInterceptor">
    <property name="paramName" value="lang"/> <!-- "lang"이라는 요청 파라미터를 감지 -->
</bean>

<!-- LocaleResolver 빈 정의 (예: CookieLocaleResolver) -->
<bean id="localeResolver" class="org.springframework.web.servlet.i18n.CookieLocaleResolver">
    <property name="defaultLocale" value="en"/>
</bean>

<!-- HandlerMapping에 인터셉터 등록 (예: SimpleUrlHandlerMapping) -->
<!-- 실제로는 RequestMappingHandlerMapping에 등록하기 위해 WebMvcConfigurer를 더 많이 사용 -->
<bean class="org.springframework.web.servlet.mvc.method.annotation.RequestMappingHandlerMapping">
    <property name="interceptors">
        <list>
            <ref bean="localeChangeInterceptor"/>
        </list>
    </property>
</bean>
```

**Java Config 예시 (`WebMvcConfigurer` 사용):**

```java
import org.springframework.context.annotation.Bean;
import org.springframework.context.annotation.Configuration;
import org.springframework.web.servlet.LocaleResolver;
import org.springframework.web.servlet.config.annotation.InterceptorRegistry;
import org.springframework.web.servlet.config.annotation.WebMvcConfigurer;
import org.springframework.web.servlet.i18n.CookieLocaleResolver;
import org.springframework.web.servlet.i18n.LocaleChangeInterceptor;
import java.util.Locale;

@Configuration
public class WebConfig implements WebMvcConfigurer {

    @Bean
    public LocaleResolver localeResolver() {
        CookieLocaleResolver clr = new CookieLocaleResolver("clientLanguage");
        clr.setDefaultLocale(Locale.KOREAN); // 기본 한국어
        return clr;
    }

    @Bean
    public LocaleChangeInterceptor localeChangeInterceptor() {
        LocaleChangeInterceptor lci = new LocaleChangeInterceptor();
        lci.setParamName("lang"); // URL 파라미터 이름 "lang" (예: /some/path?lang=en)
        return lci;
    }

    @Override
    public void addInterceptors(InterceptorRegistry registry) {
        registry.addInterceptor(localeChangeInterceptor()); // 인터셉터 등록
    }
}
```

- 위 Java Config 설정 후, 만약 사용자가 `http://example.com/home?lang=en` 과 같이 `lang` 파라미터에 `en` 값을 넣어 요청하면, `LocaleChangeInterceptor`가 이를 감지하여 로케일을 영어로 변경합니다. `CookieLocaleResolver`를 사용했다면 이 변경된 로케일 정보가 쿠키에 저장됩니다.
- 뷰(JSP, Thymeleaf 등)에서는 변경된 로케일에 따라 다른 언어의 메시지를 보여줄 수 있습니다 (보통 Spring의 `<spring:message>` 태그나 Thymeleaf의 `#{...}` 표현식과 `MessageSource`를 함께 사용).

### 다국어 지원을 위한 전체적인 그림

1. **`LocaleResolver` 빈 등록:** 애플리케이션이 로케일을 결정하는 방식을 선택하고 설정합니다. (예: `CookieLocaleResolver`)
2. **(선택) `LocaleChangeInterceptor` 빈 등록 및 설정:** 사용자가 로케일을 직접 변경할 수 있는 기능을 제공하려면 인터셉터를 설정합니다.
3. **메시지 소스(MessageSource) 설정:** 각 언어별 메시지(텍스트)를 담고 있는 프로퍼티 파일들을 설정합니다. (예: `messages_en.properties`, `messages_ko.properties`)

    ```
    // messages_en.properties
    greeting = Hello
    // messages_ko.properties
    greeting = 안녕하세요
    ```

    ```java
    // WebConfig.java (MessageSource 설정 예시)
    @Bean
    public MessageSource messageSource() {
        ReloadableResourceBundleMessageSource messageSource = new ReloadableResourceBundleMessageSource();
        messageSource.setBasename("classpath:/messages/messages"); // messages_ko.properties, messages_en.properties 등을 찾음
        messageSource.setDefaultEncoding("UTF-8");
        return messageSource;
    }
    ```

4. **뷰에서 메시지 사용:** JSP의 `<spring:message code="greeting"/>` 태그나 Thymeleaf의 `<p th:text="#{greeting}"></p>` 와 같은 방식으로 현재 로케일에 맞는 메시지를 가져와 출력합니다.

Spring MVC의 로케일 처리 기능을 사용하면, 다양한 국가의 사용자를 대상으로 하는 웹 애플리케이션을 효과적으로 구축할 수 있습니다.
