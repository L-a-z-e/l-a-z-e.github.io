---
title: Integrations - Localization
description: 
author: laze
date: 2025-05-10 00:00:05 +0900
categories: [Dev, SpringSecurity]
tags: [SpringSecurity]
---
## 지역화 (Localization)

다른 로케일(locale, 지역/언어 설정)을 지원해야 하는 경우, 알아야 할 모든 내용이 이 섹션에 포함되어 있습니다.

인증 실패 및 접근 거부(권한 부여 실패)와 관련된 메시지를 포함하여 모든 예외 메시지는 지역화될 수 있습니다.

개발자나 시스템 배포자에게 초점을 맞춘 예외 및 로깅 메시지(잘못된 속성, 인터페이스 계약 위반, 잘못된 생성자 사용, 시작 시간 유효성 검사, 디버그 수준 로깅 포함)는 지역화되지 않고 대신 Spring Security 코드 내에 영어로 하드 코딩됩니다.

`spring-security-core-xx.jar` 파일에는 `org.springframework.security` 패키지가 포함되어 있으며, 이 패키지에는 `messages.properties` 파일과 몇 가지 일반적인 언어에 대한 지역화된 버전이 포함되어 있습니다.

Spring Security 클래스는 Spring의 `MessageSourceAware` 인터페이스를 구현하고 애플리케이션 컨텍스트 시작 시 메시지 리졸버가 의존성 주입될 것으로 예상하므로, 이는 애플리케이션 컨텍스트에서 참조되어야 합니다.

일반적으로 메시지를 참조하기 위해 애플리케이션 컨텍스트 내에 빈을 등록하기만 하면 됩니다. 예는 다음과 같습니다:

```xml
<bean id="messageSource"
	class="org.springframework.context.support.ReloadableResourceBundleMessageSource">
<property name="basename" value="classpath:org/springframework/security/messages"/>
</bean>
```

`messages.properties`는 표준 리소스 번들에 따라 이름이 지정되며 Spring Security 메시지에서 지원하는 기본 언어를 나타냅니다.

이 기본 파일은 영어입니다.

`messages.properties` 파일을 사용자 정의하거나 다른 언어를 지원하려면 파일을 복사하고 적절하게 이름을 변경한 다음 위 빈 정의 내에 등록해야 합니다.

이 파일에는 많은 수의 메시지 키가 없으므로 지역화는 주요 작업으로 간주되어서는 안 됩니다.

이 파일을 지역화하는 경우, JIRA 작업을 기록하고 적절하게 이름이 지정된 지역화된 버전의 `messages.properties`를 첨부하여 커뮤니티와 작업을 공유하는 것을 고려해 주십시오.

Spring Security는 실제로 적절한 메시지를 조회하기 위해 Spring의 지역화 지원에 의존합니다.

이것이 작동하려면 들어오는 요청의 로케일이 Spring의 `org.springframework.context.i18n.LocaleContextHolder`에 저장되어 있는지 확인해야 합니다.

Spring MVC의 `DispatcherServlet`은 애플리케이션을 위해 이를 자동으로 수행하지만, Spring Security의 필터가 이전에 호출되므로 필터가 호출되기 전에 `LocaleContextHolder`가 올바른 로케일을 포함하도록 설정해야 합니다.

`web.xml`에서 Spring Security 필터보다 먼저 와야 하는 필터에서 직접 이 작업을 수행하거나 Spring의 `RequestContextFilter`를 사용할 수 있습니다.

---

### 🌍 "안녕, Hello, Hola!" (Spring Security 메시지 다국어 지원하기)

Spring Security는 보안 관련 여러 상황에서 사용자에게 메시지를 보여줘요. 예를 들어, 비밀번호를 틀렸을 때, 접근 권한이 없는 페이지에 들어가려고 할 때 등이죠.

이런 메시지들을 **사용자의 언어에 맞게 보여주는 것**을 "지역화(Localization)" 또는 "다국어 지원(i18n - internationalization)"이라고 합니다.

### 1. 어떤 메시지가 다국어 지원될까요? 🗣️

- **사용자에게 보여지는 예외 메시지:**
  - 인증 실패 메시지 (예: "잘못된 사용자 이름 또는 비밀번호입니다.")
  - 권한 부여 실패 메시지 (예: "이 페이지에 접근할 권한이 없습니다.")
- **개발자나 시스템 관리자용 메시지는 제외:**
  - 설정이 잘못되었을 때 나는 오류 메시지, 디버그 로그 등은 영어로 고정되어 있어요. 이런 건 일반 사용자가 볼 일이 거의 없으니까요.

### 2. 다국어 메시지 파일은 어디에? 📂 (`messages.properties`)

Spring Security는 이미 기본적인 다국어 메시지 파일들을 가지고 있어요. `spring-security-core-xx.jar` 라는 파일 안에 `org.springframework.security` 패키지를 찾아보면,

- `messages.properties`: **기본 언어(영어)** 메시지들이 들어있어요.
- `messages_ko.properties`: 한국어 메시지 (만약 있다면)
- `messages_fr.properties`: 프랑스어 메시지 (만약 있다면)
- ... 등등 몇몇 주요 언어에 대한 번역 파일이 있을 수 있습니다.

이 파일들은 "키(key) = 값(value)" 형태로 되어 있어요. 예를 들어,

```
# messages.properties (영어)
AbstractUserDetailsAuthenticationProvider.badCredentials=Bad credentials

# messages_ko.properties (한국어)
AbstractUserDetailsAuthenticationProvider.badCredentials=잘못된 자격 증명입니다.
```

여기서 `AbstractUserDetailsAuthenticationProvider.badCredentials`가 "키"이고, 그 뒤에 나오는 문장이 실제 메시지 "값"입니다.

### 3. Spring에게 "여기 다국어 메시지 파일 있어!" 알려주기 📢 (`MessageSource` 빈 등록)

Spring Security는 Spring 프레임워크의 다국어 지원 기능을 그대로 사용해요.

그래서 우리 애플리케이션의 Spring 설정에 `MessageSource`라는 특별한 빈(Bean)을 등록해서, "다국어 메시지 파일은 여기에 있으니 참고해!" 라고 알려줘야 합니다.

**XML 설정 예시:**

```xml
<bean id="messageSource"
    class="org.springframework.context.support.ReloadableResourceBundleMessageSource">
    <property name="basename" value="classpath:org/springframework/security/messages"/>
</bean>
```

**Java Configuration 예시:**

```java
import org.springframework.context.MessageSource;
import org.springframework.context.annotation.Bean;
import org.springframework.context.annotation.Configuration;
import org.springframework.context.support.ReloadableResourceBundleMessageSource;

@Configuration
public class AppConfig {

    @Bean
    public MessageSource messageSource() {
        ReloadableResourceBundleMessageSource messageSource = new ReloadableResourceBundleMessageSource();
        // Spring Security 기본 메시지 파일 경로
        messageSource.setBasename("classpath:org/springframework/security/messages");
        messageSource.setDefaultEncoding("UTF-8");
        return messageSource;
    }
}
```

- `ReloadableResourceBundleMessageSource`: 메시지 파일을 관리하는 Spring 클래스예요.
- `basename`: 메시지 파일들의 기본 경로와 이름을 알려줘요. `classpath:org/springframework/security/messages`라고 하면, Spring Security JAR 파일 안에 있는 `messages.properties`, `messages_ko.properties` 등을 찾게 됩니다.

### 4. 나만의 메시지로 바꾸거나, 새로운 언어 추가하기 ✍️

- **기존 메시지 수정:** Spring Security가 제공하는 기본 영어 메시지가 마음에 안 들거나, 한국어 번역이 어색하다면?
  1. `messages.properties` (영어) 또는 `messages_ko.properties` (한국어) 파일을 복사합니다.
  2. 내 프로젝트의 `src/main/resources` 폴더 아래에 같은 이름으로 붙여넣고 내용을 수정합니다.
  3. `MessageSource` 빈 설정에서 `basename`을 수정된 파일이 있는 경로로 바꿔주거나, 여러 경로를 지정할 수 있도록 설정합니다. (예: `messageSource.setBasenames("classpath:my-custom-messages", "classpath:org/springframework/security/messages");` 이렇게 하면 `my-custom-messages`를 먼저 찾고 없으면 Spring Security 기본 메시지를 찾습니다.)
- **새로운 언어 지원 (예: 일본어):**
  1. `messages.properties` 파일을 복사해서 `messages_ja.properties` (ja는 일본어 로케일 코드)로 이름을 바꿉니다.
  2. 파일 안의 영어 메시지들을 일본어로 번역합니다.
  3. 이 파일을 내 프로젝트의 `src/main/resources` 폴더 (또는 `basename`으로 지정한 경로)에 넣어둡니다.
  4. 이제 사용자의 브라우저 언어 설정이 일본어이면, `messages_ja.properties` 파일의 메시지가 자동으로 사용됩니다.

**팁:** Spring Security의 메시지 키 개수는 그렇게 많지 않아서, 직접 번역하거나 수정하는 것이 크게 부담스럽지는 않을 거예요. 만약 멋지게 번역했다면 Spring Security 커뮤니티에 공유해보는 것도 좋겠죠!

### 5. "지금 사용자는 어떤 언어를 쓸까?" 알아내기 🌐 (`LocaleContextHolder`와 `RequestContextFilter`)

Spring이 어떤 언어의 메시지를 보여줄지 결정하려면, **"현재 사용자가 어떤 언어(로케일)를 사용하고 있는지"** 알아야 해요. 이 정보는 Spring의 `LocaleContextHolder`라는 곳에 저장됩니다.

- **Spring MVC (`DispatcherServlet`) 사용자:** `DispatcherServlet`이 알아서 사용자의 요청(예: 브라우저의 `Accept-Language` 헤더)을 보고 `LocaleContextHolder`에 적절한 로케일 정보를 설정해줘요.
- **문제점:** Spring Security 필터는 `DispatcherServlet`보다 **먼저** 실행되는 경우가 많아요. 그래서 Spring Security 필터가 메시지를 보여줘야 할 시점에는 아직 `LocaleContextHolder`에 로케일 정보가 없을 수도 있어요!
- **해결책:**
  1. **`RequestContextFilter` 사용 (권장):** Spring 프레임워크가 제공하는 `RequestContextFilter`를 Spring Security 필터 **앞에** 오도록 `web.xml`이나 Java 기반 웹 설정에 등록해주세요. 이 필터가 HTTP 요청 정보를 바탕으로 `LocaleContextHolder`를 미리 설정해줍니다.

      ```java
      // WebApplicationInitializer를 사용하는 Java 설정 예시
      public class MyWebAppInitializer extends AbstractAnnotationConfigDispatcherServletInitializer {
          // ... 다른 설정들 ...
      
          @Override
          protected Filter[] getServletFilters() {
              // RequestContextFilter를 CharacterEncodingFilter 다음에, Spring Security 필터보다 앞에 위치
              CharacterEncodingFilter characterEncodingFilter = new CharacterEncodingFilter();
              characterEncodingFilter.setEncoding("UTF-8");
              characterEncodingFilter.setForceEncoding(true);
      
              RequestContextFilter requestContextFilter = new RequestContextFilter();
      
              // Spring Security 필터 (FilterRegistrationBean 또는 DelegatingFilterProxy 사용)
              // ...
      
              return new Filter[] { characterEncodingFilter, requestContextFilter, /* springSecurityFilterChain */ };
          }
      }
      
      ```

  2. **직접 필터 만들기:** 필요하다면 직접 필터를 만들어서 `LocaleContextHolder`를 설정하고, 이 필터를 Spring Security 필터 앞에 배치할 수도 있습니다.

이 설정을 통해 Spring Security도 현재 사용자의 로케일을 정확히 알고, 그에 맞는 언어의 메시지를 보여줄 수 있게 됩니다.

---

**오늘의 핵심 요약:**

1. **Spring Security의 사용자 대상 예외 메시지는 다국어 지원 가능!** (인증 실패, 접근 거부 등)
2. **`messages.properties` (기본 영어) 및 `messages_언어코드.properties` 파일 사용!**
3. **`MessageSource` 빈을 Spring 설정에 등록해야 함!** (Spring Security 기본 메시지 경로는 `classpath:org/springframework/security/messages`)
4. **메시지 커스터마이징 또는 새 언어 추가 가능!** (파일 복사 후 수정/번역하고 `basename` 설정)
5. **`LocaleContextHolder`에 현재 사용자 로케일 정보가 있어야 함!** (Spring MVC는 `DispatcherServlet`이 처리, 그 외에는 `RequestContextFilter` 등을 Spring Security 필터 앞에 설정)
