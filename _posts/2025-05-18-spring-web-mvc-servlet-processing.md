---
title: Spring Web MVC - Dispatcher Servlet (Processing)
description: 
author: laze
date: 2025-05-18 00:00:01 +0900
categories: [Dev, SpringWebMVC]
tags: [SpringWebMVC]
---
**처리 과정 (Processing)**

`DispatcherServlet`은 다음과 같이 요청을 처리합니다:

1. `WebApplicationContext`를 검색하여 요청(request)에 속성(attribute)으로 바인딩합니다.
   이는 컨트롤러 및 처리 과정의 다른 요소들이 사용할 수 있도록 하기 위함입니다.
   기본적으로 `DispatcherServlet.WEB_APPLICATION_CONTEXT_ATTRIBUTE` 키 아래에 바인딩됩니다.
2. 로케일 리졸버(locale resolver)가 요청에 바인딩되어, 처리 과정의 요소들이 요청을 처리할 때 (뷰 렌더링, 데이터 준비 등) 사용할 로케일을 결정할 수 있게 합니다.
   로케일 결정이 필요 없다면 로케일 리졸버도 필요하지 않습니다.
3. 테마 리졸버(theme resolver)가 요청에 바인딩되어, 뷰와 같은 요소들이 사용할 테마를 결정할 수 있게 합니다. 테마를 사용하지 않는다면 무시해도 됩니다.
4. 멀티파트 파일 리졸버(multipart file resolver)를 지정한 경우, 요청에서 멀티파트가 있는지 검사합니다.
   멀티파트가 발견되면, 요청은 처리 과정의 다른 요소들에 의한 추가 처리를 위해 `MultipartHttpServletRequest`로 래핑(wrapping)됩니다.
5. 적절한 핸들러(handler)를 검색합니다. 핸들러가 발견되면, 핸들러와 연관된 실행 체인(전처리기, 후처리기, 컨트롤러)이 실행되어 렌더링을 위한 모델(model)을 준비합니다.
   또는, 어노테이션이 달린 컨트롤러의 경우, 뷰를 반환하는 대신 응답이 ( `HandlerAdapter` 내에서) 렌더링될 수 있습니다.
6. 모델이 반환되면 뷰가 렌더링됩니다. 모델이 반환되지 않으면 (아마도 전처리기나 후처리기가 요청을 가로챘기 때문에, 예를 들어 보안상의 이유로), 요청이 이미 이행되었을 수 있으므로 뷰는 렌더링되지 않습니다.
7. `WebApplicationContext`에 선언된 `HandlerExceptionResolver` 빈들이 요청 처리 중 발생한 예외를 해결하는 데 사용됩니다. 이러한 예외 리졸버들은 예외 처리 로직을 사용자 정의할 수 있게 해줍니다.
8. HTTP 캐싱 지원을 위해, 핸들러는 `WebRequest`의 `checkNotModified` 메소드를 사용할 수 있으며, 컨트롤러를 위한 HTTP 캐싱(HTTP Caching for Controllers)에 설명된 대로 어노테이션이 달린 컨트롤러를 위한 추가 옵션도 있습니다.

`web.xml` 파일의 서블릿 선언에 서블릿 초기화 파라미터(`init-param` 요소)를 추가하여 개별 `DispatcherServlet` 인스턴스를 사용자 정의할 수 있습니다.

다음 표는 지원되는 파라미터를 나열합니다:

**표 1. DispatcherServlet 초기화 파라미터**

| 파라미터 (Parameter) | 설명 (Explanation) |
| --- | --- |
| `contextClass` | 이 서블릿에 의해 인스턴스화되고 로컬로 설정될 `ConfigurableWebApplicationContext`를 구현하는 클래스입니다. 기본적으로 `XmlWebApplicationContext`가 사용됩니다. |
| `contextConfigLocation` | 컨텍스트 인스턴스(`contextClass`로 지정됨)에 전달되어 컨텍스트를 찾을 수 있는 위치를 나타내는 문자열입니다. 이 문자열은 여러 컨텍스트를 지원하기 위해 잠재적으로 여러 문자열(쉼표를 구분자로 사용)로 구성될 수 있습니다. 빈이 두 번 정의된 여러 컨텍스트 위치의 경우, 가장 마지막 위치가 우선합니다. |
| `namespace` | `WebApplicationContext`의 네임스페이스입니다. 기본값은 `[서블릿이름]-servlet` 입니다. |
| `throwExceptionIfNoHandlerFound` | 요청에 대한 핸들러를 찾지 못했을 때 `NoHandlerFoundException`을 발생시킬지 여부입니다. 이 예외는 `HandlerExceptionResolver` (예: `@ExceptionHandler` 컨트롤러 메소드 사용)로 잡아서 다른 예외처럼 처리할 수 있습니다. <br><br> 6.1 버전부터 이 속성은 `true`로 설정되고 사용 중단(deprecated)되었습니다. <br><br> 참고로, 기본 서블릿 처리(default servlet handling)도 구성된 경우, 해결되지 않은 요청은 항상 기본 서블릿으로 전달되며 404는 절대 발생하지 않습니다. |

---

## DispatcherServlet의 요청 처리 여정: 단계별 심층 분석

클라이언트(예: 웹 브라우저)가 특정 URL로 요청을 보내면, `DispatcherServlet`은 미리 정의된 일련의 과정을 거쳐 이 요청을 처리합니다. 이 과정은 매우 체계적이며, 각 단계에서 이전에 배웠던 "특별한 빈"들이 활약합니다.

### DispatcherServlet의 주요 요청 처리 단계

1. **`WebApplicationContext` 검색 및 요청(request)에 바인딩:**
  - **목적:** 현재 요청을 처리하는 동안 컨트롤러나 다른 컴포넌트들이 Spring의 빈 컨테이너(`WebApplicationContext`)에 접근하여 필요한 빈(예: 서비스 객체)을 사용할 수 있도록 하기 위함입니다.
  - **동작:** `DispatcherServlet`은 자신이 사용하는 `WebApplicationContext`를 찾아서, 현재 HTTP 요청 객체(`HttpServletRequest`)의 속성(attribute)으로 저장합니다.
  - **기본 키:** `DispatcherServlet.WEB_APPLICATION_CONTEXT_ATTRIBUTE` (상수 값: `org.springframework.web.servlet.FrameworkServlet.CONTEXT.dispatcher`)
  - **예시:** 컨트롤러나 뷰에서 `RequestContextUtils.findWebApplicationContext(request)`를 통해 현재 `WebApplicationContext`에 접근할 수 있습니다.
2. **로케일 리졸버(LocaleResolver) 바인딩:**
  - **목적:** 다국어 지원(i18n)을 위해 현재 요청에 대한 적절한 로케일(언어 및 국가 설정)을 결정하고, 이후 과정(뷰 렌더링, 데이터 포맷팅 등)에서 이 로케일 정보를 사용할 수 있도록 합니다.
  - **동작:** 등록된 `LocaleResolver`를 사용하여 현재 요청의 로케일을 결정하고, 이 정보를 요청 객체에 바인딩합니다.
  - **생략 가능:** 만약 애플리케이션이 다국어를 지원할 필요가 없다면, `LocaleResolver`를 설정하지 않아도 됩니다.
3. **테마 리졸버(ThemeResolver) 바인딩:**
  - **목적:** 웹사이트에 여러 테마(예: 밝은 테마, 어두운 테마)를 적용할 수 있도록 현재 요청에 대한 테마를 결정하고, 뷰에서 이 테마 정보를 사용할 수 있도록 합니다.
  - **동작:** 등록된 `ThemeResolver`를 사용하여 현재 요청의 테마를 결정하고, 이 정보를 요청 객체에 바인딩합니다.
  - **생략 가능:** 테마 기능을 사용하지 않는다면 무시해도 됩니다.
4. **멀티파트 파일 리졸버(MultipartResolver) 검사 및 요청 래핑:**
  - **목적:** 파일 업로드와 같은 멀티파트 요청을 처리하기 위함입니다.
  - **동작:**
    - `MultipartResolver`가 설정되어 있는지 확인합니다.
    - 설정되어 있다면, 현재 요청이 멀티파트 요청(예: `Content-Type`이 `multipart/form-data`인 경우)인지 검사합니다.
    - 멀티파트 요청으로 판단되면, 원래의 `HttpServletRequest` 객체를 `MultipartHttpServletRequest` 인터페이스를 구현한 객체로 **래핑(wrapping)**합니다.
    - `MultipartHttpServletRequest`는 업로드된 파일에 쉽게 접근할 수 있는 추가 메소드(예: `getFile(String name)`, `getFileMap()`)를 제공합니다.
  - **이후 처리:** 컨트롤러 등에서는 이 래핑된 `MultipartHttpServletRequest`를 통해 파일 데이터를 처리합니다. (자세한 내용은 "Multipart Resolver" 챕터에서)
5. **적절한 핸들러(Handler) 검색 (by `HandlerMapping`):**
  - **목적:** 현재 요청 URL 및 HTTP 메소드 등의 정보를 바탕으로, 이 요청을 실제로 처리할 **핸들러(일반적으로 컨트롤러의 특정 메소드)**를 찾는 것입니다.
  - **동작:** 등록된 `HandlerMapping` 구현체들(예: `RequestMappingHandlerMapping`)이 순서대로 호출되어, 요청에 가장 적합한 핸들러를 찾습니다.
  - **결과:** 핸들러가 찾아지면, 해당 핸들러와 그 핸들러에 적용될 인터셉터(pre-processors, post-processors) 목록을 포함하는 `HandlerExecutionChain` 객체가 반환됩니다.
6. **핸들러 실행 및 모델(Model) 준비 (by `HandlerAdapter`):**
  - **목적:** 5번 단계에서 찾은 핸들러를 실행하고, 뷰 렌더링에 필요한 데이터(모델)를 준비합니다.
  - **동작:**
    - `HandlerExecutionChain`에 포함된 **인터셉터의 전처리(preHandle) 메소드**들이 순서대로 실행됩니다. 만약 어떤 `preHandle` 메소드가 `false`를 반환하면, 요청 처리는 여기서 중단됩니다 (보통 보안 검사 등에 사용).
    - 적절한 `HandlerAdapter`(예: `RequestMappingHandlerAdapter`)가 선택되어, 핸들러(컨트롤러 메소드)를 **실행**합니다. 이때 `@PathVariable`, `@RequestParam` 등의 어노테이션 처리, 파라미터 바인딩 등이 이루어집니다.
    - 컨트롤러 메소드는 뷰에 전달할 데이터(모델)를 설정하고, 뷰의 논리적인 이름이나 `ModelAndView` 객체를 반환할 수 있습니다.
    - **어노테이션 기반 컨트롤러의 특수 경우:** 만약 컨트롤러 메소드가 `@ResponseBody` 어노테이션을 사용하거나 `ResponseEntity`를 반환하는 경우, 뷰를 통하지 않고 `HandlerAdapter` 내에서 직접 HTTP 응답 본문이 생성될 수 있습니다. 이 경우 모델과 뷰는 반환되지 않을 수 있습니다.
    - 핸들러 실행 후, **인터셉터의 후처리(postHandle) 메소드**들이 (역순으로) 실행됩니다. 이 단계에서 모델을 수정하거나 뷰에 추가적인 데이터를 전달할 수 있습니다.
7. **뷰(View) 렌더링 또는 요청 처리 완료:**
  - **모델이 반환된 경우 (뷰 렌더링 필요):**
    - 컨트롤러가 반환한 뷰 이름(또는 `ModelAndView` 객체에 포함된 뷰 정보)을 사용하여 `ViewResolver`가 실제 뷰(예: JSP, Thymeleaf 템플릿)를 찾거나 생성합니다.
    - 선택된 `View` 객체의 `render(model, request, response)` 메소드가 호출되어, 모델 데이터를 사용하여 최종 HTML 응답 등을 생성하고 클라이언트에게 전송합니다.
  - **모델이 반환되지 않은 경우 (요청이 이미 처리됨):**
    - 앞선 단계(예: 인터셉터의 `preHandle`, 또는 `@ResponseBody`를 사용한 컨트롤러)에서 이미 HTTP 응답이 직접 생성되어 전송되었을 수 있습니다.
    - 이 경우, 뷰 렌더링 과정은 생략됩니다. 요청은 이미 "이행(fulfilled)"된 것으로 간주합니다.
  - **완료 후 처리(afterCompletion):** 뷰 렌더링이 완료되거나 요청 처리가 완전히 끝난 후, `HandlerExecutionChain`에 포함된 인터셉터의 `afterCompletion` 메소드들이 (역순으로) 실행됩니다. 이 단계는 주로 리소스 정리 등의 작업을 수행합니다.
8. **예외 처리 (by `HandlerExceptionResolver`):**
  - 위의 1~7번 과정 중 어느 단계에서든 예외가 발생하면, `DispatcherServlet`은 등록된 `HandlerExceptionResolver` 빈들에게 이 예외를 처리할 기회를 줍니다.
  - `HandlerExceptionResolver`는 특정 예외를 미리 정의된 에러 페이지로 안내하거나, 특정 HTTP 상태 코드를 응답하거나, 로그를 남기는 등의 사용자 정의 예외 처리 로직을 수행할 수 있습니다. (자세한 내용은 "Exceptions" 챕터에서)
9. **HTTP 캐싱 지원:**
  - 컨트롤러는 `WebRequest`의 `checkNotModified` 메소드 등을 사용하여 HTTP `Last-Modified` 헤더나 `ETag`를 활용한 캐싱 전략을 구현할 수 있습니다. 이를 통해 불필요한 데이터 전송을 줄이고 성능을 향상시킬 수 있습니다. (자세한 내용은 "HTTP Caching for Controllers" 챕터에서)

### `DispatcherServlet` 초기화 파라미터 (`web.xml` 사용 시)

만약 `web.xml`을 사용하여 `DispatcherServlet`을 등록하는 경우 (Java Config 방식이 아닌), `<servlet>` 태그 내의 `<init-param>`을 통해 `DispatcherServlet`의 몇 가지 동작을 제어할 수 있습니다. (Spring Boot나 `WebApplicationInitializer`를 사용하면 이런 `init-param`을 직접 쓸 일은 거의 없습니다.)

| 파라미터 | 설명 |
| --- | --- |
| `contextClass` | `DispatcherServlet`이 사용할 `WebApplicationContext` 구현 클래스를 지정합니다. 기본값은 `XmlWebApplicationContext`입니다. (Java Config 시에는 보통 `AnnotationConfigWebApplicationContext`가 사용됩니다.) |
| `contextConfigLocation` | `contextClass`로 지정된 `WebApplicationContext` 인스턴스가 로드할 Spring 설정 파일의 위치를 지정합니다. (예: `/WEB-INF/spring/dispatcher-servlet.xml`). 쉼표로 여러 위치를 지정할 수 있습니다. |
| `namespace` | `WebApplicationContext`의 네임스페이스를 지정합니다. 기본값은 `[서블릿이름]-servlet` (예: 서블릿 이름이 `app`이면 `app-servlet`) 입니다. 이 네임스페이스는 빈 이름 충돌 방지 등에 사용될 수 있습니다. |
| `throwExceptionIfNoHandlerFound` | **(중요: 6.1부터 `true`로 기본 변경 및 deprecated)** 요청을 처리할 핸들러를 찾지 못했을 때 `NoHandlerFoundException`을 발생시킬지 여부를 설정합니다. `true`로 설정하면 이 예외를 `HandlerExceptionResolver`로 잡아서 404 에러 페이지 등을 보여줄 수 있습니다. 기본 서블릿 핸들링이 구성된 경우 이 설정은 무시될 수 있습니다. |

**`throwExceptionIfNoHandlerFound`의 변화:** 과거에는 이 값이 기본적으로 `false`여서, 핸들러를 못 찾으면 서블릿 컨테이너가 기본 404 페이지를 보여줬습니다. Spring 4.x 버전부터 `true`로 설정하여 Spring MVC 내에서 404 처리를 더 유연하게 할 수 있도록 권장되었고, Spring 6.1부터는 기본값이 `true`가 되고 이 파라미터 자체는 deprecated 되었습니다. 즉, 이제는 기본적으로 핸들러를 못 찾으면 `NoHandlerFoundException`이 발생한다고 생각하면 됩니다.
