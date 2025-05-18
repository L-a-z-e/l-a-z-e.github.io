---
title: Spring Web MVC - Dispatcher Servlet (Path Matching)
description: 
author: laze
date: 2025-05-18 00:00:03 +0900
categories: [Dev, SpringWebMVC]
tags: [SpringWebMVC]
---
**경로 매칭 (Path Matching)**

서블릿 API는 전체 요청 경로를 `requestURI`로 노출하고, 이를 다시 `contextPath`, `servletPath`, `pathInfo`로 세분화합니다.

이 값들은 서블릿이 어떻게 매핑되느냐에 따라 달라집니다.

이러한 입력으로부터 Spring MVC는 핸들러 매핑에 사용할 조회 경로(lookup path)를 결정해야 하며, 이 조회 경로는 (해당하는 경우) `contextPath`와 모든 `servletMapping` 접두사를 제외해야 합니다.

`servletPath`와 `pathInfo`는 디코딩되어 제공되므로, 조회 경로를 도출하기 위해 전체 `requestURI`와 직접 비교하는 것이 불가능하며, 이로 인해 `requestURI`를 디코딩할 필요성이 생깁니다.

그러나 이는 그 자체로 문제를 야기하는데, 경로는 디코딩 후 경로 구조를 변경할 수 있는 인코딩된 예약 문자(예: "/" 또는 ";")를 포함할 수 있으며, 이는 또한 보안 문제로 이어질 수 있습니다.

또한, 서블릿 컨테이너는 `servletPath`를 다양한 정도로 정규화할 수 있어, `requestURI`에 대한 `startsWith` 비교를 더욱 불가능하게 만듭니다.

이것이 바로 접두사 기반 `servletPath` 매핑 타입과 함께 제공되는 `servletPath`에 대한 의존성을 피하는 것이 최선인 이유입니다.

만약 `DispatcherServlet`이 "/"로 기본 서블릿으로 매핑되거나, 또는 "/*"로 접두사 없이 매핑되고 서블릿 컨테이너가 4.0 이상이라면,

Spring MVC는 서블릿 매핑 타입을 감지하고 `servletPath`와 `pathInfo`의 사용을 완전히 피할 수 있습니다.

3.1 서블릿 컨테이너에서는 동일한 서블릿 매핑 타입을 가정할 때, MVC 설정의 경로 매칭(Path Matching)을 통해 `alwaysUseFullPath=true`로 설정된 `UrlPathHelper`를 제공함으로써 동등한 결과를 얻을 수 있습니다.

다행히 기본 서블릿 매핑 "/"은 좋은 선택입니다. 그러나 `requestURI`를 디코딩하여 컨트롤러 매핑과 비교할 수 있도록 해야 한다는 문제가 여전히 남아 있습니다.

이는 경로 구조를 변경하는 예약된 문자를 디코딩할 가능성 때문에 다시 바람직하지 않습니다.

만약 그러한 문자가 예상되지 않는다면, 이를 거부하거나(Spring Security HTTP 방화벽처럼), 또는 `UrlPathHelper`를 `urlDecode=false`로 설정할 수 있지만,

이 경우 컨트롤러 매핑은 인코딩된 경로와 일치해야 하며 이는 항상 잘 작동하지 않을 수 있습니다.

더욱이, 때때로 `DispatcherServlet`은 다른 서블릿과 URL 공간을 공유해야 할 수 있으며, 접두사로 매핑되어야 할 수도 있습니다.

위의 문제들은 `AntPathMatcher`를 사용한 문자열 경로 매칭의 대안으로 `PathPatternParser`와 파싱된 패턴을 사용할 때 해결됩니다.

`PathPatternParser`는 Spring MVC 버전 5.3부터 사용할 수 있으며, 버전 6.0부터는 기본적으로 활성화됩니다.

조회 경로를 디코딩하거나 컨트롤러 매핑을 인코딩해야 하는 `AntPathMatcher`와 달리, 파싱된 `PathPattern`은 한 번에 하나의 경로 세그먼트씩 `RequestPath`라는 파싱된 경로 표현과 일치합니다.

이를 통해 경로 구조를 변경할 위험 없이 경로 세그먼트 값을 개별적으로 디코딩하고 살균(sanitizing)할 수 있습니다.

파싱된 `PathPattern`은 또한 서블릿 경로 매핑이 사용되고 접두사가 단순하게 유지되는 한(즉, 인코딩된 문자가 없는 한) `servletPath` 접두사 매핑의 사용을 지원합니다.

---

## Spring MVC의 경로 매칭: 정확하고 안전한 URL 처리를 위한 여정

웹 애플리케이션에서 URL 경로를 올바르게 해석하는 것은 매우 중요합니다. Spring MVC는 클라이언트가 요청한 URL을 분석하여 어떤 컨트롤러가 이 요청을 처리해야 할지 결정하는데, 이 과정에서 여러 가지 복잡한 문제들을 고려해야 합니다.

### 1. 서블릿 API의 경로 정보와 그 한계

서블릿 API는 요청 URL에 대한 정보를 다음과 같이 여러 부분으로 나누어 제공합니다:

- **`requestURI`**: 전체 요청 경로 (예: `/myapp/user/info;jsessionid=123?name=John`)
- **`contextPath`**: 웹 애플리케이션의 컨텍스트 경로 (예: `/myapp`)
- **`servletPath`**: 서블릿 매핑에 사용된 경로 (예: `/user` - 만약 서블릿이 `/user/*`로 매핑되었다면)
- **`pathInfo`**: `servletPath` 이후의 나머지 경로 (예: `/info` - 위 예시에서)

Spring MVC는 이 정보들을 조합하여 **조회 경로(lookup path)**를 만들어야 합니다. 이 조회 경로는 실제로 컨트롤러 매핑에 사용될 순수한 경로 부분입니다. (일반적으로 `contextPath`와 `servletPath`의 일부를 제외한 부분)

**문제점들:**

1. **디코딩(Decoding) 문제:**
  - `servletPath`와 `pathInfo`는 서블릿 컨테이너에 의해 **이미 URL 디코딩된 상태**로 제공됩니다.
  - 반면, `requestURI`는 인코딩된 상태일 수 있습니다.
  - 조회 경로를 정확히 얻기 위해 `requestURI`를 디코딩해야 하는데, 이때 문제가 발생할 수 있습니다. 예를 들어, URL 경로에 인코딩된 슬래시 (`%2F`)나 세미콜론 (`%3B`) 같은 예약 문자가 포함되어 있다면, 디코딩 후 경로의 구조 자체가 변경될 수 있습니다. 이는 **보안 취약점(Path Traversal 등)**으로 이어질 수 있습니다.
  - 예시: `/path/..%2F..%2Fsecret.txt` -> 디코딩 후 -> `/path/../../secret.txt` (상위 디렉토리 접근 시도)
2. **`servletPath` 정규화(Normalization) 문제:**
  - 서블릿 컨테이너마다 `servletPath`를 정규화하는 방식이 조금씩 다를 수 있습니다. (예: 마지막 슬래시 처리 등)
  - 이 때문에 `requestURI.startsWith(contextPath + servletPath)`와 같은 단순한 문자열 비교로 조회 경로를 안정적으로 추출하기 어렵습니다.
3. **`servletPath` 의존성의 어려움:**
  - 특히 `DispatcherServlet`이 `/` (기본 서블릿)이 아닌, 특정 접두사(prefix)로 매핑된 경우 (예: `/app/*`), `servletPath`를 정확히 계산하고 제외하는 것이 까다롭습니다.

### 2. Spring MVC의 경로 매칭 전략 변화

이러한 문제들을 해결하고 더 정확하며 안전한 경로 매칭을 위해 Spring MVC는 다음과 같은 접근 방식을 취해왔습니다.

### 가. `UrlPathHelper`와 설정 옵션

- Spring MVC는 `UrlPathHelper`라는 유틸리티 클래스를 사용하여 URL 경로를 해석하고 조회 경로를 결정합니다.
- `UrlPathHelper`에는 다음과 같은 중요한 설정 옵션들이 있습니다 (주로 MVC Config를 통해 설정 가능):
  - **`alwaysUseFullPath`**:
    - `true`로 설정하면, `servletPath`를 무시하고 `requestURI`에서 `contextPath`만 제외하여 전체 경로를 조회 경로로 사용합니다.
    - 이는 `DispatcherServlet`이 `/` 또는 `/*`로 매핑되어 있고, 서블릿 컨테이너가 4.0 이상일 때 Spring MVC가 자동으로 감지하여 유사하게 동작합니다. (3.1 컨테이너에서는 이 설정을 명시적으로 켜야 할 수 있습니다.)
    - `servletPath` 해석의 복잡성을 피하는 데 도움이 됩니다.
  - **`urlDecode`**:
    - 기본값은 `true`입니다. 조회 경로를 가져올 때 URL 디코딩을 수행합니다.
    - `false`로 설정하면 URL 디코딩을 하지 않습니다. 이 경우 컨트롤러의 `@RequestMapping` 패턴도 인코딩된 상태로 매칭되어야 하므로, 항상 실용적인 것은 아닙니다. 하지만 특정 보안 시나리오에서는 유용할 수 있습니다.

**`DispatcherServlet`을 `/` (기본 서블릿 매핑)로 설정하는 것이 일반적으로 가장 간단하고 권장되는 방식입니다.** 이렇게 하면 `servletPath`와 관련된 많은 복잡성을 피할 수 있습니다.

### 나. `PathPatternParser`의 등장 (Spring Framework 5.3 이상)

전통적으로 Spring MVC는 `AntPathMatcher`를 사용하여 `@RequestMapping`의 경로 패턴과 실제 요청 URL을 비교했습니다. `AntPathMatcher`는 유연하지만, 위에서 언급된 디코딩 문제와 경로 구조 변경 위험에 완전히 자유롭지는 않았습니다.

**`PathPatternParser`와 `RequestPath`:**

- **Spring Framework 5.3부터 도입되었고, 6.0부터는 기본으로 활성화됩니다.**
- **작동 방식:**
  1. 컨트롤러의 `@RequestMapping` 패턴 (예: `/users/{id}`)을 미리 **파싱(parsing)**하여 내부적인 `PathPattern` 객체로 만들어 둡니다.
  2. 요청이 들어오면, 요청 URL도 파싱하여 `RequestPath`라는 내부 표현으로 만듭니다.
  3. `PathPattern`과 `RequestPath`를 **세그먼트(segment, `/`로 구분된 각 경로 조각) 단위로 비교**합니다.
- **장점:**
  - **안전한 디코딩 및 살균(Sanitization):** 각 경로 세그먼트 값을 개별적으로 디코딩하고 검증(예: `../` 같은 경로 조작 시도 방지)할 수 있습니다. 전체 경로를 한 번에 디코딩함으로써 발생할 수 있는 경로 구조 변경 위험을 크게 줄입니다.
  - **성능 향상:** 미리 파싱된 패턴을 사용하므로, 문자열 기반의 `AntPathMatcher`보다 일반적으로 성능이 더 좋습니다.
  - **명확한 구문:** `AntPathMatcher`보다 더 명확하고 예측 가능한 패턴 구문을 제공합니다. (자세한 내용은 공식 문서의 "Pattern Comparison" 참조)
  - **`servletPath` 접두사 지원:** `DispatcherServlet`이 접두사로 매핑된 경우에도, 접두사가 단순하다면(인코딩된 문자 없음) `PathPatternParser`는 이를 잘 처리할 수 있습니다.

**`PathPatternParser`로의 전환은 Spring MVC가 URL 경로를 더 안전하고 효율적으로 처리하기 위한 중요한 발전입니다.**

**흐름 요약 ( `PathPatternParser` 사용 시):**

1. 애플리케이션 시작 시: `@RequestMapping("/users/{id}")` -> `PathPatternParser`가 파싱 -> `PathPattern` 객체 생성 및 저장.
2. 요청 발생: `GET /myapp/users/123`
3. `DispatcherServlet`이 `UrlPathHelper` 등을 이용해 `contextPath` (`/myapp`)를 제외한 조회 경로 후보를 얻습니다.
4. 이 조회 경로 후보 (`/users/123`)를 `RequestPath` 객체로 파싱합니다.
5. 등록된 `PathPattern`들과 `RequestPath`를 세그먼트 단위로 비교하여 일치하는 핸들러(컨트롤러 메소드)를 찾습니다.
  - `/users` 세그먼트 일치.
  - `/{id}` 패턴과 `123` 세그먼트 일치. (`id` 변수에 `123` 값 바인딩 준비)

### 3. 개발자가 고려할 사항

- **`DispatcherServlet` 매핑:** 가능하면 `DispatcherServlet`을 `/`로 매핑하는 것이 가장 간단하고 문제를 줄이는 방법입니다.
- **URL 인코딩:** 클라이언트가 보내는 URL이 올바르게 인코딩되었는지, 서버 측에서 디코딩 시 발생할 수 있는 보안 문제는 없는지 항상 유의해야 합니다. Spring Security의 HTTP 방화벽 같은 도구를 함께 사용하면 도움이 될 수 있습니다.
- **Spring 버전:** Spring Framework 6.0 이상을 사용한다면 기본적으로 `PathPatternParser`가 활성화되어 더 안전하고 효율적인 경로 매칭의 이점을 누릴 수 있습니다. 5.3 ~ 5.x 버전에서는 명시적으로 활성화할 수 있습니다. (보통 `WebMvcConfigurer`의 `configurePathMatch` 메소드를 통해 설정)
  (Spring Boot에서는 `spring.mvc.pathmatch.matching-strategy=path_pattern_parser` 프로퍼티로 설정 가능)

    ```java
    // WebMvcConfigurer에서 PathPatternParser 활성화 (Spring 5.3.x 예시)
    @Override
    public void configurePathMatch(PathMatchConfigurer configurer) {
        configurer.setPatternParser(new PathPatternParser());
    }
    
    ```


이 챕터는 Spring MVC가 "단순해 보이는" URL 매칭 뒤에서 얼마나 많은 고민을 하고 있는지 보여줍니다. 특히 보안과 관련된 부분은 매우 중요합니다.

---

### 1. `requestURI`, `contextPath`, `servletPath`, `pathInfo` 란 무엇인가?

이 값들은 **서블릿 컨테이너(예: Tomcat, Jetty 등)**가 HTTP 요청을 받았을 때, 요청 URL을 분석하여 `HttpServletRequest` 객체를 통해 개발자에게 제공하는 정보입니다. Spring MVC가 세팅하는 것이 아니라, **서블릿 컨테이너가 생성하고 세팅**합니다.

**상황 예시:**

- 웹 애플리케이션 이름(컨텍스트): `myApp`
- `DispatcherServlet`이 `/service/*` 라는 URL 패턴으로 매핑되어 있음
- 클라이언트가 요청한 전체 URL: `http://localhost:8080/myApp/service/users/100?action=view`

이 경우, `HttpServletRequest` 객체에서 각 메소드를 호출하면 다음과 같은 값을 얻을 수 있습니다:

- **`request.getRequestURI()` -> `/myApp/service/users/100`**
  - **정의:** 프로토콜, 호스트, 포트, 쿼리 스트링(`?action=view`)을 **제외한** 전체 경로 부분입니다.
  - **특징:** URL 인코딩이 유지될 수도 있고, 부분적으로 디코딩될 수도 있습니다 (서블릿 컨테이너 구현에 따라 약간 다를 수 있지만, 일반적으로는 쿼리 스트링 앞까지의 경로를 나타냅니다).
  - 가장 "날 것(raw)"에 가까운 경로 정보라고 볼 수 있습니다.
- **`request.getContextPath()` -> `/myApp`**
  - **정의:** 현재 웹 애플리케이션이 서블릿 컨테이너 내에서 가지는 고유한 경로입니다. 웹 애플리케이션의 "루트" 디렉토리라고 생각할 수 있습니다.
  - **특징:**
    - 만약 웹 애플리케이션이 컨테이너의 루트에 직접 배포되었다면 (예: `http://localhost:8080/service/...`), `contextPath`는 빈 문자열(`""`)이 됩니다.
    - 항상 슬래시(`/`)로 시작하거나 (루트가 아닐 경우), 빈 문자열입니다.
    - 슬래시로 끝나지 않습니다 (루트이고 빈 문자열이 아닌 경우).
- **`request.getServletPath()` -> `/service`**
  - **정의:** 요청을 처리한 서블릿(여기서는 `DispatcherServlet`)의 매핑 경로 부분입니다.
  - **특징:**
    - 어떤 URL 패턴으로 서블릿을 매핑했느냐에 따라 값이 달라집니다.
      - 만약 `DispatcherServlet`이 `/` (기본 서블릿)로 매핑되었다면, `servletPath`는 `requestURI`에서 `contextPath`를 제외한 값과 거의 유사하게 됩니다 (정확한 값은 컨테이너와 매핑 방식에 따라 조금씩 다를 수 있습니다. 예를 들어 `/index.html`을 요청했다면 `/index.html`이 될 수 있습니다).
      - 만약 `.do` 와 같이 확장자 패턴으로 매핑되었다면, `servletPath`는 `/some/path/action.do` 와 같이 실제 요청된 경로가 됩니다.
      - 우리의 예시처럼 `/service/*`로 매핑되었다면, `servletPath`는 `/service`가 됩니다.
    - 일반적으로 슬래시(`/`)로 시작합니다.
    - **서블릿 컨테이너에 의해 URL 디코딩된 상태로 제공될 수 있습니다.** (이것이 중요한 포인트!)
- **`request.getPathInfo()` -> `/users/100`**
  - **정의:** `servletPath` 이후의 나머지 경로 부분입니다. 서블릿이 처리해야 할 구체적인 리소스나 액션을 나타냅니다.
  - **특징:**
    - `servletPath`가 요청 URI의 전체를 포함한다면 (예: 확장자 매핑), `pathInfo`는 `null`이 될 수 있습니다.
    - 우리의 예시처럼 `DispatcherServlet`이 `/service/*`로 매핑되어 있고, 실제 요청이 `/myApp/service/users/100` 이라면, `pathInfo`는 `/users/100`이 됩니다.
    - 슬래시(`/`)로 시작합니다 (존재하는 경우).
    - **서블릿 컨테이너에 의해 URL 디코딩된 상태로 제공될 수 있습니다.** (이것도 중요한 포인트!)

**요약:**

`requestURI` = `contextPath` + `servletPath` + `pathInfo` (항상 정확히 이 공식이 성립하는 것은 아니며, 특히 `pathInfo`가 `null`이거나 `servletPath`의 매핑 방식에 따라 약간의 변형이 있을 수 있습니다. 하지만 개념적으로는 이렇게 구성됩니다.)

### 2. 이 값들이 언제 세팅되는가?

클라이언트가 웹 서버로 HTTP 요청을 보내면, 웹 서버(예: Apache)는 이 요청을 WAS(Web Application Server, 서블릿 컨테이너 포함, 예: Tomcat)로 전달합니다.

1. **Tomcat과 같은 서블릿 컨테이너는 들어온 HTTP 요청 메시지를 파싱합니다.**
2. 이 과정에서 요청 라인(request line, 예: `GET /myApp/service/users/100?action=view HTTP/1.1`)의 URL 부분을 분석합니다.
3. 분석된 URL과 웹 애플리케이션의 배포 정보(`contextPath`), 그리고 `web.xml`이나 Java Config에 설정된 서블릿 매핑 정보를 바탕으로 `requestURI`, `contextPath`, `servletPath`, `pathInfo` 값을 **계산하고 결정합니다.**
4. 결정된 값들을 `HttpServletRequest` 객체의 필드(또는 내부 데이터 구조)에 **저장(세팅)**합니다.
5. 이렇게 값이 채워진 `HttpServletRequest` 객체가 필터 체인을 거쳐 최종적으로 `DispatcherServlet` (또는 다른 서블릿)에게 전달됩니다.

즉, 이 값들은 **Spring MVC가 요청을 받기 전, 서블릿 컨테이너 단에서 이미 세팅되어 `HttpServletRequest` 객체를 통해 넘어오는 정보**입니다.

### 3. 개발자가 이 과정을 얼마나 알아야 하는가?

**일반적인 웹 애플리케이션 개발 시나리오에서는 이 값들이 어떻게 계산되는지 그 내부 로직까지 상세히 알 필요는 없습니다.** 대부분의 경우 Spring MVC는 이러한 복잡성을 잘 추상화해주고, 개발자는 `@GetMapping("/{userId}")` 와 같이 컨트롤러 레벨에서 경로 변수만 신경 쓰면 됩니다.

**하지만 다음과 같은 경우에는 이 값들의 의미와 특징을 이해하는 것이 도움이 됩니다:**

1. **복잡한 URL 매핑 또는 필터링 로직 구현 시:**
  - `DispatcherServlet`이 `/`가 아닌 특정 접두사로 매핑되어 여러 서블릿/필터와 URL 공간을 공유해야 할 때.
  - 요청 URL을 기반으로 매우 세밀한 접근 제어 필터를 만들어야 할 때.
  - URL 재작성(Rewrite) 규칙을 적용해야 할 때.
2. **경로 관련 문제 해결(Troubleshooting) 시:**
  - 특정 URL이 예상대로 컨트롤러에 매핑되지 않을 때, `requestURI`, `servletPath`, `pathInfo` 값을 로그로 찍어보면 원인 파악에 도움이 될 수 있습니다.
  - URL 인코딩/디코딩과 관련된 보안 문제나 오작동을 분석할 때.
3. **Spring MVC의 내부 동작 원리 이해 심화:**
  - Spring MVC가 어떻게 `UrlPathHelper`를 사용하고, 왜 `PathPatternParser` 같은 새로운 매칭 전략이 도입되었는지 이해하려면, 이 기본 경로 정보들의 한계점을 아는 것이 중요합니다. (바로 이 챕터의 내용이죠!)
4. **서블릿 스펙 및 컨테이너 동작 이해:**
  - Spring MVC뿐만 아니라 일반적인 서블릿/JSP 개발에서도 이 값들은 기본적으로 제공되는 정보이므로, 웹 개발자로서 알아두면 좋습니다.

**결론:**

- `requestURI`, `contextPath`, `servletPath`, `pathInfo`는 **서블릿 컨테이너가 HTTP 요청을 분석하여 `HttpServletRequest` 객체에 설정해주는 값**입니다.
- Spring MVC는 이 값들을 **입력으로 받아** 컨트롤러 매핑에 사용할 "조회 경로(lookup path)"를 결정합니다.
- **일반적인 개발에서는 이 값들의 생성 과정까지 알 필요는 없지만, 그 의미와 특징, 그리고 Spring MVC가 이들을 어떻게 활용하려고 하는지에 대한 개념을 이해하고 있으면 고급 기능 활용이나 문제 해결 시 매우 유용합니다.**

---

### Spring MVC의 경로 매칭 도전 과제와 해결 노력

앞서 설명드렸듯이, `servletPath`와 `pathInfo`는 이미 디코딩되어 있을 수 있고, `requestURI`는 인코딩된 상태일 수 있으며, 여기에 `contextPath`까지 고려하여 "컨트롤러 매핑에 사용할 실제 경로" (이를 **조회 경로(lookup path)**라고 부릅니다)를 추출하는 것은 생각보다 까다로운 작업입니다.

### 1. 조회 경로(Lookup Path) 결정의 어려움

Spring MVC의 `HandlerMapping` (예: `RequestMappingHandlerMapping`)은 이 **조회 경로**를 기준으로 `@RequestMapping` 등으로 정의된 컨트롤러 메소드와 매칭을 시도합니다.

**이상적인 조회 경로:** `requestURI`에서 `contextPath`를 제외하고, 만약 `DispatcherServlet`이 특정 접두사(prefix)로 매핑되었다면 그 접두사(`servletPath`의 일부)까지 제외한 순수한 경로.

**예시:**

- `requestURI`: `/myApp/app1/users/100`
- `contextPath`: `/myApp`
- `DispatcherServlet`이 `/app1/*`로 매핑되어 `servletPath`가 `/app1` 이라고 가정.
- **원하는 조회 경로:** `/users/100` (이것이 `@GetMapping("/users/{id}")` 와 매칭되어야 함)

**문제 발생 시나리오 (전통적인 방식의 어려움):**

1. **디코딩 불일치:**
  - `requestURI`는 `/myApp/app1/users%2Fdetails/100` (인코딩된 슬래시 `%2F` 포함)
  - `servletPath`는 컨테이너에 의해 디코딩되어 `/app1`
  - `pathInfo`도 컨테이너에 의해 디코딩되어 `/users/details/100`
  - 만약 `requestURI`에서 `contextPath`와 `servletPath`를 문자열로 단순 제거하려고 하면, 인코딩 상태가 달라 정확한 조회 경로를 얻기 어렵습니다.
  - `requestURI` 전체를 디코딩하면? 경로 중간에 있던 `%2F`가 `/`로 바뀌면서 경로 구조가 변경될 수 있습니다. (`/users/details/100`이 되어야 하는데, `/users//details/100` 처럼 될 수도 있거나, `%2F`가 경로 구분자로 오인될 수 있음) 이는 **보안 문제**를 야기할 수 있습니다 (예: Path Traversal 공격).
2. **`servletPath`의 모호성 및 컨테이너별 차이:**
  - `DispatcherServlet`이 `/` (기본 서블릿)로 매핑된 경우, `servletPath`의 값이 컨테이너나 상황에 따라 조금씩 다를 수 있습니다. (예: 요청이 `/index.html`이면 `servletPath`가 `/index.html`이 될 수도 있고, 요청이 `/users/`이면 `/users/`가 될 수도 있음)
  - 이런 `servletPath`의 변동성 때문에 이를 기준으로 조회 경로를 일관되게 추출하는 것이 어렵습니다.

### 2. `UrlPathHelper`: 경로 해석 도우미

Spring MVC는 이러한 복잡성을 다루기 위해 `org.springframework.web.util.UrlPathHelper`라는 클래스를 내부적으로 사용합니다.

- **역할:** `HttpServletRequest` 객체로부터 컨트롤러 매핑에 사용할 **조회 경로(lookup path)**를 추출하는 로직을 담당합니다.
- **주요 기능 및 설정:**
  - **`getLookupPathForRequest(HttpServletRequest request)`:** 이 메소드가 핵심입니다. 요청 객체를 받아 조회 경로를 반환합니다.
  - **`alwaysUseFullPath` (boolean, 기본값 `false`):**
    - `true`로 설정 시: `servletPath`를 사용하지 않고, `requestURI`에서 `contextPath`만 제거한 전체 경로를 조회 경로로 사용합니다. 이는 `DispatcherServlet`이 `/` (기본 서블릿)로 매핑되었을 때 발생하는 `servletPath`의 모호성을 피하는 데 도움이 됩니다. (Spring 4.0 이후로는 `/` 또는 `/*` 매핑 시 자동으로 이와 유사하게 동작하려는 경향이 있습니다.)
    - `false`로 설정 시 (기본값): `servletPath`를 고려하여 조회 경로를 결정하려고 시도합니다.
  - **`urlDecode` (boolean, 기본값 `true`):**
    - `true`로 설정 시: 조회 경로를 반환하기 전에 URL 디코딩을 수행합니다. 대부분의 경우 컨트롤러 매핑은 디코딩된 경로를 기준으로 하기 때문에 기본값이 `true`입니다.
    - `false`로 설정 시: URL 디코딩을 하지 않고 경로를 반환합니다. 이 경우 컨트롤러의 `@RequestMapping` 패턴도 인코딩된 값과 일치해야 하므로 사용이 제한적입니다. (예: `@GetMapping("/users%2Fdetails")`)
  - **`removeSemicolonContent` (boolean, 기본값 `true`):**
    - `true`로 설정 시: 경로에서 세미콜론( `;` )과 그 뒤의 내용(일명 '경로 파라미터' 또는 '매트릭스 변수')을 제거합니다. (예: `/users;jsessionid=123` -> `/users`) Spring MVC는 기본적으로 매트릭스 변수를 지원하지 않으므로, 이를 제거하는 것이 일반적입니다.
  - 기타: `defaultEncoding` (URL 디코딩 시 사용할 기본 인코딩) 등.

`UrlPathHelper`는 MVC 설정을 통해 커스터마이징할 수 있습니다. (예: `WebMvcConfigurer`의 `configurePathMatch` 메소드 내에서 `pathHelper.setAlwaysUseFullPath(true);` 와 같이 설정)

**`UrlPathHelper`의 노력에도 불구하고 남는 문제:**`urlDecode=true`가 기본값이지만, 앞서 말했듯이 `requestURI`를 디코딩하는 과정에서 경로 구조가 변경될 위험은 여전히 존재합니다. 특히 악의적인 사용자가 인코딩된 예약 문자를 포함한 URL을 보낼 경우 문제가 될 수 있습니다.

### 3. `PathPatternParser`: 더 안전하고 효율적인 경로 매칭 (Spring 5.3+)

이러한 근본적인 디코딩 문제를 해결하고, 더 나은 성능과 명확성을 제공하기 위해 Spring Framework 5.3부터 `org.springframework.web.util.pattern.PathPatternParser`가 도입되었고, **Spring 6.0부터는 기본 경로 매칭 전략으로 사용됩니다.**

**`PathPatternParser`의 핵심 아이디어:**

1. **패턴 사전 파싱:** 애플리케이션 시작 시, `@RequestMapping` 등에 정의된 모든 URL 패턴(예: `/users/{id}/orders/{orderId}`)을 미리 `PathPatternParser`를 사용하여 파싱하고, 내부적인 `PathPattern` 객체 구조로 변환해 둡니다. 이 `PathPattern`은 URL을 `/` 기준으로 나눈 각 **세그먼트(segment)**의 정보와 변수 정보 등을 구조적으로 가지고 있습니다.
2. **요청 경로 파싱 (`RequestPath`):** 실제 HTTP 요청이 들어오면, `UrlPathHelper` 등을 통해 얻은 조회 경로 후보 문자열을 `RequestPath`라는 객체로 파싱합니다. `RequestPath` 역시 요청 URL을 세그먼트 단위로 분리하고, 각 세그먼트에 대한 정보(디코딩 전/후 값 등)를 가집니다.
3. **구조적 매칭 (세그먼트 단위):**
  - 미리 파싱된 `PathPattern`과 요청의 `RequestPath`를 **한 세그먼트씩 순서대로 비교**합니다.
  - **가장 큰 장점:** 각 세그먼트 값을 **개별적으로 디코딩하고 정제(sanitize)**할 수 있습니다.
    - 예를 들어, `/users/{id}` 패턴과 `/users/foo%2Fbar` 요청이 있을 때:
      - 첫 번째 세그먼트 `users`는 `users`와 일치.
      - 두 번째 세그먼트 `foo%2Fbar`는 `{id}` 변수에 해당.
      - `PathPatternParser`는 `foo%2Fbar`를 디코딩하여 `foo/bar`를 얻지만, 이것이 경로 구분자를 포함하고 있음을 인지하고 안전하게 처리합니다. (예: 변수 값 자체로만 사용하고, 경로 구조를 변경시키지 않음. 또는 특정 문자를 허용하지 않도록 설정 가능)
  - 전체 경로 문자열을 한 번에 디코딩함으로써 발생할 수 있는 경로 구조 변경 및 보안 위험을 효과적으로 방지합니다.

**`PathPatternParser`의 이점:**

- **향상된 보안:** 경로 조작(Path Traversal) 공격 등에 더 강인합니다. 각 세그먼트 값을 안전하게 처리합니다.
- **성능 개선:** 미리 파싱된 패턴과의 구조적 비교는 일반적으로 문자열 기반의 `AntPathMatcher`보다 빠릅니다.
- **명확한 패턴 구문:** `AntPathMatcher`의 일부 모호한 구문(예: `*`의 사용)을 개선하고 더 명확한 규칙을 제공합니다.
- **`servletPath` 접두사 지원:** `DispatcherServlet`이 특정 접두사로 매핑된 경우에도, 해당 접두사가 단순하다면(인코딩된 문자 미포함) 잘 처리합니다.

**전통적인 `AntPathMatcher`와의 비교:**

- `AntPathMatcher`: 문자열 기반 패턴 매칭. 조회 경로 전체를 디코딩하거나, 컨트롤러 패턴을 인코딩해야 하는 딜레마.
- `PathPatternParser`: 구조적 패턴 매칭. 요청 경로를 세그먼트 단위로 파싱하고, 각 세그먼트를 안전하게 디코딩/처리 후 `PathPattern`과 비교.

**결론적으로, Spring MVC는:**

- 서블릿 컨테이너가 제공하는 기본 경로 정보(`requestURI`, `contextPath` 등)의 한계를 인지하고,
- `UrlPathHelper`를 통해 조회 경로를 추출하려는 노력을 해왔으며,
- 궁극적으로 `PathPatternParser`라는 더 안전하고 효율적인 경로 매칭 메커니즘을 도입하여 URL 처리의 정확성과 보안성을 크게 향상시켰습니다.

개발자 입장에서는 Spring 버전이 올라갈수록 이러한 내부적인 경로 처리의 복잡성이 점점 더 잘 추상화되고 숨겨지므로, 컨트롤러의 `@RequestMapping` 패턴 정의에만 집중할 수 있게 됩니다. 하지만 이러한 배경을 이해하고 있으면, 예기치 않은 경로 관련 문제가 발생했을 때 원인을 파악하거나, 보안 설정을 구성할 때 도움이 될 수 있습니다.

---

### `%2F`는 무엇인가? (URL 인코딩의 세계)

`%2F`는 **URL 인코딩(URL Encoding, 또는 Percent-encoding)**된 **슬래시(`/`) 문자**를 의미합니다.

**URL 인코딩이란?**

URL은 인터넷에서 특정 리소스를 찾기 위한 주소입니다. 이 URL 주소에는 특정 규칙에 따라 사용할 수 있는 문자 집합이 정해져 있습니다 (주로 알파벳, 숫자, 그리고 일부 특수 문자 `-`, `.`, `_`, `~`).

하지만 URL 경로, 쿼리 파라미터 값 등에 다음과 같은 문자들을 포함해야 할 때가 있습니다:

- **예약된 문자(Reserved Characters):** URL에서 특별한 의미를 가지는 문자들 (예: `/`, `?`, `#`, `&`, `=`, `:`, `;` 등). 이 문자들을 데이터 값 자체로 사용하려면 인코딩해야 합니다.
- **안전하지 않은 문자(Unsafe Characters):** 공백 문자, `<`, `>`, `"`, `{`, `}` 등 URL에 직접 사용하면 문제를 일으킬 수 있는 문자들.
- **ASCII가 아닌 문자:** 한글, 일본어, 중국어 등 ASCII 문자 집합에 포함되지 않는 문자들.

이런 문자들을 URL에 안전하게 포함시키기 위해 **URL 인코딩**이라는 규칙을 사용합니다.

**인코딩 방식:**

1. 해당 문자를 특정 문자 인코딩(주로 UTF-8)을 사용하여 바이트(byte) 시퀀스로 변환합니다.
2. 변환된 각 바이트를 `%` 기호 뒤에 두 자리 16진수 값으로 표현합니다.

**예시:**

- **슬래시 (`/`) 문자:**
  - ASCII 코드로 `0x2F` 입니다. (16진수 2F)
  - URL 인코딩하면: **`%2F`**
- **공백 문자 ( )**:
  - ASCII 코드로 `0x20` 입니다.
  - URL 인코딩하면: **`%20`** (또는 전통적으로 `+` 기호로 대체되기도 합니다. 특히 쿼리 파라미터에서)
- **물음표 (`?`) 문자:**
  - ASCII 코드로 `0x3F` 입니다.
  - URL 인코딩하면: **`%3F`**
- **한글 "안녕" (UTF-8 인코딩 시):**
  - '안': `EC 95 88` (16진수) -> `%EC%95%88`
  - '녕': `EB 85 95` (16진수) -> `%EB%85%95`
  - "안녕" -> `%EC%95%88%EB%85%95`

### `%2F`는 왜 생기는가? (슬래시를 데이터로 사용하고 싶을 때)

일반적으로 슬래시(`/`)는 URL에서 **경로 구분자**로 사용됩니다. 예를 들어 `/users/search` 에서 `/`는 `users` 디렉토리(또는 리소스 그룹) 아래의 `search` 리소스(또는 액션)를 의미합니다.

하지만 만약 **경로의 일부로서, 즉 데이터 값 자체로서 슬래시 문자를 포함시켜야 하는 경우**가 있다면 어떻게 할까요?

예를 들어, 파일 이름이나 어떤 식별자(ID) 값에 슬래시가 포함될 수 있습니다.

- **상황:** 어떤 사용자의 아이디가 `john/doe` 라고 가정해봅시다.
- 이 사용자의 정보를 가져오는 URL을 `/users/{userId}` 와 같이 설계했다고 할 때,
- 만약 URL을 `/users/john/doe` 라고 만들면, 서버는 이 경로를 `/users` 아래의 `john` 디렉토리 아래의 `doe` 리소스로 해석할 가능성이 높습니다. 즉, `userId`가 `john`인지 `john/doe`인지 모호해집니다.

이런 경우, `userId` 값에 포함된 슬래시를 URL 경로 구분자가 아닌 **데이터의 일부**로 명확히 표현하기 위해 URL 인코딩을 사용합니다.

- `john/doe` 라는 값을 URL에 안전하게 포함시키려면:
  - `j`, `o`, `h`, `n`, `d`, `o`, `e`는 그대로 사용 가능.
  - 중간의 `/`는 `%2F`로 인코딩.
  - 결과: `john%2Fdoe`
- 따라서 이 사용자의 정보를 요청하는 URL은 다음과 같이 구성될 수 있습니다:
  - `http://example.com/users/john%2Fdoe`

클라이언트(예: 웹 브라우저나 API 호출 프로그램)는 서버로 요청을 보낼 때, 이렇게 특수 문자를 포함한 경로 세그먼트나 쿼리 파라미터 값을 **URL 인코딩하여 전송**합니다.

### `%2F`가 왜 경로 매칭에서 문제를 일으키는가?

앞서 설명드린 것처럼, 서버(서블릿 컨테이너 또는 Spring MVC)는 이 URL을 받아서 디코딩하는 과정을 거칩니다.

- 만약 `http://example.com/users/john%2Fdoe` 라는 요청을 서버가 받아서,
- 컨트롤러 매핑에 사용할 조회 경로를 얻기 위해 **전체 경로를 한 번에 디코딩**한다고 가정해봅시다.
- `john%2Fdoe`는 디코딩 후 `john/doe`가 됩니다.
- 그러면 조회 경로는 `/users/john/doe`가 됩니다.

이제 Spring MVC가 이 `/users/john/doe` 경로를 `@GetMapping("/users/{userId}")` 패턴과 매칭하려고 할 때 문제가 발생합니다.

- **`AntPathMatcher` (전통 방식):** 이 경우, `/users/john/doe`를 보고 `{userId}`에 `john`이 매칭되고, 그 뒤에 `/doe`가 남아서 패턴과 일치하지 않는다고 판단할 수 있습니다. 또는, `{userId}`가 어디까지인지 모호해집니다.
- **보안 문제:** 만약 사용자가 악의적으로 `product/..%2F..%2Fadmin_config` 와 같은 값을 `@PathVariable`에 주입하려고 시도하고, 이것이 디코딩되어 `product/../../admin_config`가 되면, 의도치 않은 경로 탐색(Path Traversal)이 발생할 수 있습니다.

**`PathPatternParser`가 이 문제를 해결하는 방법:**

`PathPatternParser`는 요청 경로를 세그먼트 단위로 처리합니다.

1. `@GetMapping("/users/{userId}")` 패턴을 보고, 두 번째 세그먼트가 `{userId}` 변수임을 인지합니다.
2. 요청 URL `/users/john%2Fdoe` 에서 두 번째 세그먼트 값으로 `john%2Fdoe`를 가져옵니다.
3. 이 `john%2Fdoe`를 `{userId}` 변수의 값으로 **디코딩하여** `john/doe`를 얻습니다.
4. 이때, `PathPatternParser`는 디코딩된 값 `john/doe`에 경로 구분자 `/`가 포함되어 있음을 알지만, 이를 **새로운 경로 세그먼트로 확장하여 해석하지 않고, `{userId}`라는 단일 변수의 값으로 안전하게 취급**합니다. (필요하다면 이러한 문자를 포함한 값을 거부하도록 설정할 수도 있습니다.)

**결론적으로 `%2F`는:**

- **URL 경로 구분자인 슬래시(`/`)를 데이터 값 자체로 사용하기 위해 URL 인코딩된 형태**입니다.
- 클라이언트가 URL을 구성할 때, 경로 세그먼트나 쿼리 파라미터 값에 `/` 문자가 포함되어야 하는 경우 `%2F`로 변환하여 전송합니다.
- 서버 측에서 이 값을 어떻게 디코딩하고 해석하느냐에 따라 경로 매칭의 정확성과 보안에 영향을 미칠 수 있으며, 이것이 Spring MVC가 `PathPatternParser`와 같은 정교한 매칭 전략을 도입하게 된 중요한 이유 중 하나입니다.

이제 `%2F`가 무엇이고 왜 생기는지, 그리고 이것이 경로 매칭에 어떤 영향을 미치는지에 대한 궁금증이 해결되셨을까요?
