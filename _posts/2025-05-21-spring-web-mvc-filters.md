---
title: Spring Web MVC - Filters
description: 
author: laze
date: 2025-05-21 00:00:01 +0900
categories: [Dev, SpringWebMVC]
tags: [SpringWebMVC]
---
**필터 (Filters)**

`spring-web` 모듈은 몇 가지 유용한 필터를 제공합니다:

- 폼 데이터 (Form Data)
- 전달된 헤더 (Forwarded Headers)
- 얕은 ETag (Shallow ETag)
- CORS
- URL 핸들러 (URL Handler)

서블릿 필터는 `web.xml` 설정 파일 또는 서블릿 어노테이션을 사용하여 설정할 수 있습니다.

Spring Boot를 사용하는 경우, 빈으로 선언하고 애플리케이션의 일부로 설정할 수 있습니다.

**폼 데이터 (Form Data)**
브라우저는 HTTP GET 또는 HTTP POST를 통해서만 폼 데이터를 제출할 수 있지만, 비브라우저 클라이언트는 HTTP PUT, PATCH, DELETE도 사용할 수 있습니다.

서블릿 API는 `ServletRequest.getParameter*()` 메소드가 HTTP POST에 대해서만 폼 필드 접근을 지원하도록 요구합니다.

`spring-web` 모듈은 `application/x-www-form-urlencoded` 콘텐츠 타입의 HTTP PUT, PATCH, DELETE 요청을 가로채고,

요청 본문에서 폼 데이터를 읽어 `ServletRequest`를 래핑(wrapping)하여 `ServletRequest.getParameter*()` 계열의 메소드를 통해 폼 데이터를 사용할 수 있도록 하는 `FormContentFilter`를 제공합니다.

**전달된 헤더 (Forwarded Headers)**

요청이 로드 밸런서와 같은 프록시를 통과하면서 호스트, 포트, 스킴(scheme)이 변경될 수 있으며, 이는 클라이언트 관점에서 올바른 호스트, 포트, 스킴을 가리키는 링크를 생성하는 데 어려움을 줍니다.

RFC 7239는 프록시가 원본 요청에 대한 정보를 제공하는 데 사용할 수 있는 `Forwarded` HTTP 헤더를 정의합니다.

**비표준 헤더 (Non-standard Headers)**`X-Forwarded-Host`, `X-Forwarded-Port`, `X-Forwarded-Proto`, `X-Forwarded-Ssl`, `X-Forwarded-Prefix`를 포함한 다른 비표준 헤더들도 있습니다.

- **`X-Forwarded-Host`**
  표준은 아니지만, `X-Forwarded-Host: <host>`는 원본 호스트를 다운스트림 서버에 전달하는 데 사용되는 사실상의 표준 헤더입니다. 예를 들어, `example.com/resource` 요청이 `localhost:8080/resource`로 요청을 전달하는 프록시로 전송되면, `X-Forwarded-Host: example.com` 헤더를 보내 서버에 원본 호스트가 `example.com`이었음을 알릴 수 있습니다.
- **`X-Forwarded-Port`**
  표준은 아니지만, `X-Forwarded-Port: <port>`는 원본 포트를 다운스트림 서버에 전달하는 데 사용되는 사실상의 표준 헤더입니다. 예를 들어, `example.com/resource` 요청이 `localhost:8080/resource`로 요청을 전달하는 프록시로 전송되면, `X-Forwarded-Port: 443` 헤더를 보내 서버에 원본 포트가 443이었음을 알릴 수 있습니다.
- **`X-Forwarded-Proto`**
  표준은 아니지만, `X-Forwarded-Proto: (https|http)`는 원본 프로토콜(예: https / https)을 다운스트림 서버에 전달하는 데 사용되는 사실상의 표준 헤더입니다. 예를 들어, `example.com/resource` 요청이 `localhost:8080/resource`로 요청을 전달하는 프록시로 전송되면, `X-Forwarded-Proto: https` 헤더를 보내 서버에 원본 프로토콜이 https였음을 알릴 수 있습니다.
- **`X-Forwarded-Ssl`**
  표준은 아니지만, `X-Forwarded-Ssl: (on|off)`는 원본 프로토콜(예: https / https)을 다운스트림 서버에 전달하는 데 사용되는 사실상의 표준 헤더입니다. 예를 들어, `example.com/resource` 요청이 `localhost:8080/resource`로 요청을 전달하는 프록시로 전송되면, `X-Forwarded-Ssl: on` 헤더를 보내 서버에 원본 프로토콜이 https였음을 알릴 수 있습니다.
- **`X-Forwarded-Prefix`**
  표준은 아니지만, `X-Forwarded-Prefix: <prefix>`는 원본 URL 경로 접두사를 다운스트림 서버에 전달하는 데 사용되는 사실상의 표준 헤더입니다.

  `X-Forwarded-Prefix`의 사용은 배포 시나리오에 따라 다를 수 있으며, 대상 서버의 경로 접두사를 교체, 제거 또는 앞에 추가할 수 있도록 유연해야 합니다.

  - **시나리오 1: 경로 접두사 재정의**`https://example.com/api/{path}` -> `http://localhost:8080/app1/{path}`
    접두사는 캡처 그룹 `{path}` 이전의 경로 시작 부분입니다. 프록시의 경우 접두사는 `/api`이고 서버의 경우 접두사는 `/app1`입니다. 이 경우 프록시는 `X-Forwarded-Prefix: /api`를 보내 원본 접두사 `/api`가 서버 접두사 `/app1`을 재정의하도록 할 수 있습니다.
  - **시나리오 2: 경로 접두사 제거**
    때때로 애플리케이션은 접두사를 제거하기를 원할 수 있습니다. 예를 들어, 다음 프록시-서버 매핑을 고려해보십시오:
    `https://app1.example.com/{path}` -> `http://localhost:8080/app1/{path}https://app2.example.com/{path}` -> `http://localhost:8080/app2/{path}`
    프록시에는 접두사가 없지만, 애플리케이션 app1과 app2에는 각각 `/app1`과 `/app2`라는 경로 접두사가 있습니다. 프록시는 `X-Forwarded-Prefix:` (빈 값)를 보내 빈 접두사가 서버 접두사 `/app1`과 `/app2`를 재정의하도록 할 수 있습니다.

    이 배포 시나리오의 일반적인 경우는 프로덕션 애플리케이션 서버당 라이선스 비용을 지불하고, 비용을 줄이기 위해 서버당 여러 애플리케이션을 배포하는 것이 선호될 때입니다. 또 다른 이유는 서버 실행에 필요한 리소스를 공유하기 위해 동일한 서버에서 더 많은 애플리케이션을 실행하는 것입니다.

    이러한 시나리오에서 애플리케이션은 동일한 서버에 여러 애플리케이션이 있기 때문에 비어 있지 않은 컨텍스트 루트가 필요합니다. 그러나 이는 애플리케이션이 다음과 같은 이점을 제공하는 다른 하위 도메인을 사용할 수 있는 공용 API의 URL 경로에는 표시되어서는 안 됩니다:

    - 추가된 보안, 예를 들어 동일 출처 정책(same origin policy)
    - 애플리케이션의 독립적인 확장 (다른 도메인이 다른 IP 주소를 가리킴)
  - **시나리오 3: 경로 접두사 삽입**
    다른 경우에는 접두사를 앞에 추가해야 할 수도 있습니다. 예를 들어, 다음 프록시-서버 매핑을 고려해보십시오:
    `https://example.com/api/app1/{path}` -> `http://localhost:8080/app1/{path}`
    이 경우 프록시의 접두사는 `/api/app1`이고 서버의 접두사는 `/app1`입니다. 프록시는 `X-Forwarded-Prefix: /api/app1`를 보내 원본 접두사 `/api/app1`이 서버 접두사 `/app1`을 재정의하도록 할 수 있습니다.

**`ForwardedHeaderFilter**ForwardedHeaderFilter`는 a) `Forwarded` 헤더를 기반으로 호스트, 포트, 스킴을 변경하고, b) 추가적인 영향을 제거하기 위해 해당 헤더들을 제거하도록 요청을 수정하는 서블릿 필터입니다.

이 필터는 요청을 래핑하는 데 의존하므로, 원본 요청이 아닌 수정된 요청으로 작업해야 하는 `RequestContextFilter`와 같은 다른 필터보다 앞에 정렬되어야 합니다.

**보안 고려 사항 (Security Considerations)**
애플리케이션은 헤더가 의도한 대로 프록시에 의해 추가되었는지, 아니면 악의적인 클라이언트에 의해 추가되었는지 알 수 없으므로 전달된 헤더에 대한 보안 고려 사항이 있습니다.

이것이 바로 신뢰 경계에 있는 프록시가 외부에서 오는 신뢰할 수 없는 `Forwarded` 헤더를 제거하도록 설정되어야 하는 이유입니다.

`ForwardedHeaderFilter`를 `removeOnly=true`로 설정할 수도 있으며, 이 경우 헤더를 제거하지만 사용하지는 않습니다.

**디스패처 타입 (Dispatcher Types)**
비동기 요청 및 에러 디스패치를 지원하려면 이 필터는 `DispatcherType.ASYNC` 및 `DispatcherType.ERROR`로 매핑되어야 합니다.

Spring Framework의 `AbstractAnnotationConfigDispatcherServletInitializer` (서블릿 설정(Servlet Config) 참조)를 사용하는 경우 모든 필터는 모든 디스패치 유형에 대해 자동으로 등록됩니다.

그러나 `web.xml`을 통해 또는 Spring Boot에서 `FilterRegistrationBean`을 통해 필터를 등록하는 경우 `DispatcherType.REQUEST` 외에 `DispatcherType.ASYNC` 및 `DispatcherType.ERROR`를 포함해야 합니다.

**얕은 ETag (Shallow ETag)**`ShallowEtagHeaderFilter` 필터는 응답에 작성된 콘텐츠를 캐싱하고 이로부터 MD5 해시를 계산하여 "얕은" ETag를 생성합니다.

다음에 클라이언트가 요청을 보내면 동일한 작업을 수행하지만, 계산된 값을 `If-None-Match` 요청 헤더와 비교하여 두 값이 같으면 304 (NOT_MODIFIED)를 반환합니다.

이 전략은 네트워크 대역폭은 절약하지만 CPU는 절약하지 못합니다.

각 요청에 대해 전체 응답을 계산해야 하기 때문입니다. 상태 변경 HTTP 메소드 및 `If-Match`, `If-Unmodified-Since`와 같은 다른 HTTP 조건부 요청 헤더는 이 필터의 범위 밖에 있습니다.

컨트롤러 수준의 다른 전략은 계산을 피하고 HTTP 조건부 요청에 대한 더 광범위한 지원을 가질 수 있습니다.

이 필터에는 `W/"02a2d595e6ed9a0b24f027f2b63b134d6"` (RFC 7232 섹션 2.3에 정의됨)와 유사한 약한 ETag를 작성하도록 필터를 설정하는 `writeWeakETag` 파라미터가 있습니다.

비동기 요청을 지원하려면 이 필터는 `DispatcherType.ASYNC`로 매핑되어야 필터가 마지막 비동기 디스패치가 끝날 때까지 ETag 생성을 지연하고 성공적으로 생성할 수 있습니다.

Spring Framework의 `AbstractAnnotationConfigDispatcherServletInitializer` (서블릿 설정(Servlet Config) 참조)를 사용하는 경우 모든 필터는 모든 디스패치 유형에 대해 자동으로 등록됩니다.

그러나 `web.xml`을 통해 또는 Spring Boot에서 `FilterRegistrationBean`을 통해 필터를 등록하는 경우 `DispatcherType.ASYNC`를 포함해야 합니다.

**CORS**

Spring MVC는 컨트롤러의 어노테이션을 통해 CORS 설정에 대한 세밀한 지원을 제공합니다.

그러나 Spring Security와 함께 사용하는 경우, Spring Security 필터 체인보다 앞에 정렬되어야 하는 내장 `CorsFilter`를 사용하는 것이 좋습니다.

**URL 핸들러 (URL Handler)**

이전 Spring Framework 버전에서는 들어오는 요청을 컨트롤러 메소드에 매핑할 때 URL 경로의 후행 슬래시를 무시하도록 Spring MVC를 설정할 수 있었습니다.

이는 `PathMatchConfigurer`에서 `setUseTrailingSlashMatch` 옵션을 활성화하여 수행할 수 있었습니다.

즉, "GET /home/" 요청을 보내면 `@GetMapping("/home")`으로 어노테이트된 컨트롤러 메소드에 의해 처리되었습니다.

이 옵션은 폐기되었지만, 애플리케이션은 여전히 이러한 요청을 안전한 방식으로 처리해야 합니다.

`UrlHandlerFilter` 서블릿 필터는 이러한 목적으로 설계되었습니다. 다음과 같이 설정할 수 있습니다:

- 후행 슬래시가 있는 URL을 수신할 때 HTTP 리다이렉트 상태로 응답하여, 브라우저를 후행 슬래시가 없는 URL 변형으로 보냅니다.
- 요청이 후행 슬래시 없이 전송된 것처럼 요청을 래핑하고 요청 처리를 계속합니다.

다음은 블로그 애플리케이션을 위해 `UrlHandlerFilter`를 인스턴스화하고 설정하는 방법입니다:

**Java**

```java
UrlHandlerFilter urlHandlerFilter = UrlHandlerFilter
		// "/blog/my-blog-post/" -> "/blog/my-blog-post" 로 HTTP 308 영구 리다이렉트
		.trailingSlashHandler("/blog/**").redirect(HttpStatus.PERMANENT_REDIRECT)
		// "/admin/user/account/" 요청을 래핑하여 "/admin/user/account"처럼 만들고 처리 계속
		.trailingSlashHandler("/admin/**").wrapRequest()
		.build();
```

---

## Spring Web 모듈이 제공하는 유용한 필터들

Spring은 웹 애플리케이션 개발 시 자주 필요한 공통 기능들을 필터 형태로 제공하여, 개발자가 직접 구현하는 수고를 덜어주고 코드의 재사용성을 높입니다.

**필터 설정 방법:**

- **`web.xml`:** 전통적인 방식으로 서블릿 필터를 선언하고 매핑합니다.
- **서블릿 어노테이션:** `@WebFilter` 어노테이션을 사용하여 필터를 정의합니다. (서블릿 3.0 이상)
- **Spring Boot:** 필터 구현체를 `@Component` 또는 `@Bean`으로 등록하면 Spring Boot가 자동으로 감지하여 등록해줍니다. `FilterRegistrationBean`을 사용하여 더 세밀한 설정(순서, URL 패턴 등)도 가능합니다.
- **Spring MVC (Java Config):** `AbstractAnnotationConfigDispatcherServletInitializer`의 `getServletFilters()` 메소드를 오버라이드하여 등록하거나, `ServletContext`에 직접 등록할 수 있습니다.

### 1. 폼 데이터 (Form Data) - `FormContentFilter`

- **문제점:** HTML 폼은 기본적으로 HTTP `GET` 또는 `POST` 메소드만 사용하여 데이터를 제출할 수 있습니다. 하지만 RESTful API와 같이 비브라우저 클라이언트(예: `curl`, Postman, 다른 서버)는 `PUT`, `PATCH`, `DELETE`와 같은 HTTP 메소드를 사용하여 폼 데이터(`application/x-www-form-urlencoded` 형식)를 전송할 수 있습니다. 그러나 서블릿 API의 `ServletRequest.getParameter*()` 메소드는 **오직 HTTP `POST` 요청에 대해서만** 폼 필드 접근을 지원하도록 규정되어 있습니다.
- **`FormContentFilter`의 역할:**
  - `Content-Type`이 `application/x-www-form-urlencoded`인 HTTP `PUT`, `PATCH`, `DELETE` 요청을 가로챕니다.
  - 요청 본문(body)에서 폼 데이터를 읽어옵니다.
  - 원래의 `ServletRequest` 객체를 래핑(wrapping)하여, 래핑된 요청 객체에서는 `getParameter*()` 메소드를 통해 `PUT`, `PATCH`, `DELETE` 요청의 폼 데이터에도 접근할 수 있도록 만들어줍니다.
- **사용 이유:** `PUT`, `PATCH`, `DELETE` 메소드로 전송된 폼 데이터도 `POST`와 동일한 방식으로 컨트롤러에서 쉽게 처리할 수 있게 합니다.

### 2. 전달된 헤더 (Forwarded Headers) - `ForwardedHeaderFilter`

- **배경:** 웹 애플리케이션은 종종 로드 밸런서, 리버스 프록시 등의 중간 프록시 서버 뒤에서 실행됩니다. 이 경우, 애플리케이션 서버가 직접 받는 요청의 호스트(host), 포트(port), 스킴(scheme, http/https) 정보는 프록시 서버의 정보일 수 있으며, 클라이언트가 원래 요청한 정보와 다를 수 있습니다. 이는 애플리케이션이 리다이렉트 URL이나 절대 경로 링크를 생성할 때 문제를 일으킬 수 있습니다.
- **표준 헤더:** RFC 7239는 `Forwarded`라는 표준 HTTP 헤더를 정의하여, 프록시가 원본 요청에 대한 정보(호스트, 프로토콜 등)를 백엔드 서버에 전달할 수 있도록 합니다.
- **비표준 헤더 (사실상 표준처럼 사용됨):**
  - `X-Forwarded-Host`: 원본 요청의 호스트 이름.
  - `X-Forwarded-Port`: 원본 요청의 포트 번호.
  - `X-Forwarded-Proto`: 원본 요청의 프로토콜 (http 또는 https).
  - `X-Forwarded-Ssl`: 원본 요청이 SSL(https)이었는지 여부 (`on` 또는 `off`).
  - `X-Forwarded-Prefix`: 원본 URL의 경로 접두사. (다양한 시나리오에 따라 접두사를 재정의, 제거, 삽입하는 데 사용될 수 있어 유연한 설정이 필요합니다. 원문에서는 3가지 시나리오를 설명했습니다.)
- **`ForwardedHeaderFilter`의 역할:**
  - 이러한 `Forwarded` 또는 `X-Forwarded-*` 헤더들을 읽어옵니다.
  - `HttpServletRequest` 객체를 래핑하여, `getRequestURL()`, `getServerName()`, `getServerPort()`, `getScheme()`, `isSecure()` 등의 메소드가 **프록시 헤더에 담긴 원본 클라이언트의 요청 정보를 반환하도록 수정**합니다.
  - 처리 후에는 보안을 위해 해당 프록시 헤더들을 요청에서 제거하여, 후속 처리에서 오용되는 것을 방지합니다.
- **중요 설정 및 고려 사항:**
  - **필터 순서:** 이 필터는 요청 정보를 변경하므로, 다른 필터들(특히 `RequestContextFilter`처럼 수정된 요청 정보를 사용해야 하는 필터)보다 **먼저 실행되도록** 순서를 지정해야 합니다.
  - **보안:** 악의적인 클라이언트가 가짜 프록시 헤더를 삽입하여 보낼 수 있습니다. 따라서 **신뢰 경계(boundary of trust)에 있는 프록시 서버(예: 가장 바깥쪽 로드 밸런서)에서 외부로부터 들어오는 신뢰할 수 없는 프록시 헤더를 반드시 제거하도록 설정해야 합니다.** `ForwardedHeaderFilter` 자체도 `removeOnly=true`로 설정하여, 헤더를 사용하지 않고 제거만 하도록 할 수 있습니다 (이 경우 헤더 값 반영은 프록시 서버나 다른 메커니즘에 의존).
  - **디스패처 타입 (`DispatcherType`):** 비동기 요청(`ASYNC`)과 에러 디스패치(`ERROR`) 상황에서도 이 필터가 동작하도록 하려면, 필터 등록 시 해당 `DispatcherType`들을 포함해야 합니다. (`AbstractAnnotationConfigDispatcherServletInitializer` 사용 시 기본적으로 모든 타입에 등록됨)

### 3. 얕은 ETag (Shallow ETag) - `ShallowEtagHeaderFilter`

- **ETag (Entity Tag)란?** HTTP 응답 헤더 중 하나로, 특정 버전의 리소스를 식별하는 고유한 문자열입니다. 클라이언트는 이 ETag 값을 저장해두었다가 다음 요청 시 `If-None-Match` 헤더에 담아 보내, 만약 서버의 리소스가 변경되지 않았다면 서버는 실제 본문 대신 `304 Not Modified` 응답을 보내 네트워크 대역폭을 절약할 수 있습니다 (HTTP 캐싱).
- **`ShallowEtagHeaderFilter`의 역할 ("얕은" 방식):**
  - 요청을 처리하고 생성된 **응답 본문 전체의 내용**을 읽어옵니다 (캐싱).
  - 이 본문 내용으로부터 MD5 해시 값을 계산하여 ETag를 생성하고, 이를 `ETag` 응답 헤더에 담아 클라이언트에게 보냅니다.
  - 다음 요청 시, 클라이언트가 `If-None-Match` 헤더에 이전에 받은 ETag 값을 보내오면:
    - 다시 응답 본문을 생성하고 MD5 해시를 계산합니다.
    - 새로 계산된 해시 값과 클라이언트가 보낸 `If-None-Match` 값을 비교합니다.
    - 두 값이 동일하면 (리소스 변경 없음), `304 Not Modified` 상태 코드를 응답하고 본문은 보내지 않습니다.
- **장점:** 네트워크 대역폭 절약.
- **단점:** **CPU 자원은 절약하지 못합니다.** 매번 전체 응답 본문을 생성하고 해시를 계산해야 하기 때문입니다. 컨트롤러 레벨에서 더 효율적인 ETag 전략 (응답 본문을 미리 생성하지 않고도 변경 여부 판단)을 사용하는 것이 더 좋을 수 있습니다. (참고: HTTP Caching 챕터)
- **`writeWeakETag` 파라미터:** `true`로 설정하면 약한(Weak) ETag (예: `W/"02a2d..."`)를 생성합니다. 약한 ETag는 리소스의 내용이 의미상으로는 동일하지만 표현 방식이 약간 다를 수 있는 경우에도 같은 ETag로 간주될 수 있음을 나타냅니다.
- **디스패처 타입:** 비동기 요청을 지원하려면 `DispatcherType.ASYNC`로 매핑되어야 합니다.

### 4. CORS (Cross-Origin Resource Sharing) - `CorsFilter` (Spring Security와 함께 사용 시)

- **CORS란?** 웹 브라우저에서 실행되는 스크립트가 자신의 출처(origin: 프로토콜, 호스트, 포트의 조합)와 다른 출처의 리소스를 요청할 수 있도록 허용하는 메커니즘입니다. 보안상의 이유로 브라우저는 기본적으로 동일 출처 정책(Same-Origin Policy)을 따릅니다.
- **Spring MVC의 CORS 지원:** 컨트롤러의 `@CrossOrigin` 어노테이션 등을 통해 세밀한 CORS 설정을 지원합니다.
- **`CorsFilter` (내장 필터):** Spring Security와 함께 사용할 경우, Spring Security 필터 체인보다 **먼저 실행되도록 `CorsFilter`를 사용하는 것이 권장**됩니다. Spring Security는 요청을 일찍 거부할 수 있으므로, CORS 관련 헤더 처리가 그보다 먼저 이루어져야 하기 때문입니다.
- 자세한 내용은 CORS 관련 챕터에서 다룹니다.

### 5. URL 핸들러 (URL Handler) - `UrlHandlerFilter` (Spring Framework 6.0+ 에서 후행 슬래시 처리)

- **배경 (과거):** 이전 Spring 버전에서는 `PathMatchConfigurer`의 `setUseTrailingSlashMatch(true)` 옵션을 통해 URL 경로 끝의 후행 슬래시(`/`)를 무시하도록 설정할 수 있었습니다. (예: `/home`과 `/home/`을 동일하게 처리)
- **현재 (Spring 6.0+):** `setUseTrailingSlashMatch` 옵션은 **폐기(retired)**되었습니다. 하지만 애플리케이션은 여전히 후행 슬래시가 있는 요청(예: `/blog/my-post/`)을 안전하게 처리할 필요가 있습니다. (SEO 관점에서도 URL 정규화는 중요)
- **`UrlHandlerFilter`의 역할:** 이 필터는 후행 슬래시 문제를 처리하기 위해 설계되었습니다. 다음과 같이 설정 가능합니다:
  - **리다이렉트 (Redirect):** 후행 슬래시가 있는 URL을 받으면, 후행 슬래시가 없는 URL로 HTTP 30x 리다이렉트 응답을 보냅니다. (예: `/blog/my-post/` -> `/blog/my-post` 로 308 영구 리다이렉트) **(권장 방식)**
  - **요청 래핑 (Wrap Request):** 후행 슬래시가 있는 URL을 받으면, 요청 객체를 래핑하여 마치 후행 슬래시가 없는 것처럼 만들고 요청 처리를 계속 진행합니다. (내부적으로만 경로가 변경됨)
- **설정 예시 (Java):**

    ```java
    import org.springframework.http.HttpStatus;
    import org.springframework.web.filter.UrlHandlerFilter; // 정확한 패키지명 확인 필요 (예시)
    
    // ...
    // 이 필터를 빈으로 등록하거나 WebApplicationInitializer 등을 통해 등록
    UrlHandlerFilter urlHandlerFilter = UrlHandlerFilter
        // "/blog/**" 패턴의 URL에 후행 슬래시가 있으면, 슬래시 없는 URL로 영구 리다이렉트
        .trailingSlashHandler("/blog/**").redirect(HttpStatus.PERMANENT_REDIRECT)
        // "/admin/**" 패턴의 URL에 후행 슬래시가 있으면, 요청을 래핑하여 슬래시 없는 것처럼 처리
        .trailingSlashHandler("/admin/**").wrapRequest()
        .build();
    
    ```


이러한 필터들은 Spring MVC 애플리케이션의 다양한 측면(요청 데이터 처리, 보안, 캐싱, URL 정규화 등)에서 개발을 용이하게 하고 애플리케이션의 품질을 높이는 데 도움을 줍니다. 각 필터의 역할과 적절한 사용 시점을 이해하는 것이 중요합니다.
