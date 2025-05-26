---
title: Spring Web MVC - Annotated Controllers (Return Values)
description: 
author: laze
date: 2025-05-26 00:00:02 +0900
categories: [Dev, SpringWebMVC]
tags: [SpringWebMVC]
---
**반환 값 (Return Values)**

다음 표는 지원되는 컨트롤러 메소드 반환 값들을 설명합니다.

| 컨트롤러 메소드 반환 값 | 설명 (Description) |
| --- | --- |
| `jakarta.servlet.ServletRequest`, `jakarta.servlet.ServletResponse` | 특정 요청 또는 응답 타입을 선택합니다. 예를 들어, `ServletRequest`, `HttpServletRequest`, 또는 Spring의 `MultipartRequest`, `MultipartHttpServletRequest`. |
| `jakarta.servlet.http.HttpSession` | 세션의 존재를 강제합니다. 결과적으로 이러한 인자는 절대 `null`이 아닙니다. 세션 접근은 스레드 안전하지 않습니다. 여러 요청이 동시에 세션에 접근할 수 있도록 허용된 경우 `RequestMappingHandlerAdapter` 인스턴스의 `synchronizeOnSession` 플래그를 `true`로 설정하는 것을 고려하십시오. |
| `jakarta.servlet.http.PushBuilder` | 프로그램 방식의 HTTP/2 리소스 푸시를 위한 서블릿 4.0 푸시 빌더 API입니다. 서블릿 명세에 따라, 클라이언트가 해당 HTTP/2 기능을 지원하지 않으면 주입된 `PushBuilder` 인스턴스는 `null`일 수 있습니다. |
| `java.security.Principal` | 현재 인증된 사용자 — 알려진 경우 특정 `Principal` 구현 클래스일 수 있습니다.<br><br>참고: 이 인수는 `HttpServletRequest#getUserPrincipal`을 통한 기본 해석(resolution)으로 대체되기 전에 사용자 정의 해석기(resolver)가 이를 해석할 수 있도록 어노테이션이 지정된 경우, 즉시 해석되지 않습니다. 예를 들어, Spring Security의 `Authentication`은 `Principal`을 구현하며, `@AuthenticationPrincipal`로도 어노테이트되어 `Authentication#getPrincipal`을 통해 사용자 정의 Spring Security 해석기에 의해 해석되는 경우가 아니라면 `HttpServletRequest#getUserPrincipal`을 통해 주입됩니다. |
| `HttpMethod` | 요청의 HTTP 메서드입니다. |
| `java.util.Locale` | 현재 요청 로케일로, 사용 가능한 가장 구체적인 `LocaleResolver`(실질적으로 구성된 `LocaleResolver` 또는 `LocaleContextResolver`)에 의해 결정됩니다. |
| `java.util.TimeZone` + `java.time.ZoneId` | `LocaleContextResolver`에 의해 결정되는 현재 요청과 관련된 시간대입니다. |
| `java.io.InputStream`, `java.io.Reader` | Servlet API에 의해 노출된 원시(raw) 요청 본문에 접근하기 위함입니다. |
| `java.io.OutputStream`, `java.io.Writer` | Servlet API에 의해 노출된 원시(raw) 응답 본문에 접근하기 위함입니다. |
| `@PathVariable` | URI 템플릿 변수에 접근하기 위함입니다. URI 패턴을 참조하십시오. |
| `@MatrixVariable` | URI 경로 세그먼트의 이름-값 쌍에 접근하기 위함입니다. 매트릭스 변수를 참조하십시오. |
| `@RequestParam` | 멀티파트 파일을 포함한 Servlet 요청 파라미터에 접근하기 위함입니다. 파라미터 값은 선언된 메서드 인수 타입으로 변환됩니다. `@RequestParam` 및 멀티파트를 참조하십시오.<br><br>참고: 단순 파라미터 값의 경우 `@RequestParam` 사용은 선택 사항입니다. 이 표의 끝에 있는 "그 외 다른 인수"를 참조하십시오. |
| `@RequestHeader` | 요청 헤더에 접근하기 위함입니다. 헤더 값은 선언된 메서드 인수 타입으로 변환됩니다. `@RequestHeader`를 참조하십시오. |
| `@CookieValue` | 쿠키에 접근하기 위함입니다. 쿠키 값은 선언된 메서드 인수 타입으로 변환됩니다. `@CookieValue`를 참조하십시오. |
| `@RequestBody` | HTTP 요청 본문에 접근하기 위함입니다. 본문 내용은 `HttpMessageConverter` 구현을 사용하여 선언된 메서드 인수 타입으로 변환됩니다. `@RequestBody`를 참조하십시오. |
| `HttpEntity<B>` | 요청 헤더와 본문에 접근하기 위함입니다. 본문은 `HttpMessageConverter`로 변환됩니다. `HttpEntity`를 참조하십시오. |
| `@RequestPart` | `multipart/form-data` 요청의 파트(part)에 접근하고, `HttpMessageConverter`로 파트의 본문을 변환하기 위함입니다. 멀티파트를 참조하십시오. |
| `java.util.Map`, `org.springframework.ui.Model`, `org.springframework.ui.ModelMap` | HTML 컨트롤러에서 사용되고 뷰 렌더링의 일부로 템플릿에 노출되는 모델에 접근하기 위함입니다. |
| `RedirectAttributes` | 리다이렉트 시 사용할 속성(즉, 쿼리 문자열에 추가될 속성)과 리다이렉트 후의 요청까지 일시적으로 저장될 플래시(flash) 속성을 지정합니다. 리다이렉트 속성 및 플래시 속성을 참조하십시오. |
| `@ModelAttribute` | 모델의 기존 속성(없는 경우 인스턴스화됨)에 접근하며, 데이터 바인딩 및 유효성 검사가 적용됩니다. `@ModelAttribute` 및 모델과 `DataBinder`를 참조하십시오.<br><br>참고: `@ModelAttribute`의 사용은 선택 사항입니다(예: 속성을 설정하기 위해). 이 표의 끝에 있는 "그 외 다른 인수"를 참조하십시오. |
| `Errors`, `BindingResult` | 커맨드 객체(즉, `@ModelAttribute` 인수)에 대한 유효성 검사 및 데이터 바인딩 오류 또는 `@RequestBody` 또는 `@RequestPart` 인수의 유효성 검사 오류에 접근하기 위함입니다. 유효성 검사를 받는 메서드 인수 바로 뒤에 `Errors` 또는 `BindingResult` 인수를 선언해야 합니다. |
| `SessionStatus` + 클래스 수준 `@SessionAttributes` | 폼 처리가 완료되었음을 표시하여, 클래스 수준 `@SessionAttributes` 어노테이션을 통해 선언된 세션 속성의 정리를 트리거합니다. 자세한 내용은 `@SessionAttributes`를 참조하십시오. |
| `UriComponentsBuilder` | 현재 요청의 호스트, 포트, 스킴, 컨텍스트 경로 및 서블릿 매핑의 리터럴 부분에 상대적인 URL을 준비하기 위함입니다. URI 링크를 참조하십시오. |
| `@SessionAttribute` | 모든 세션 속성에 접근하기 위함이며, 이는 클래스 수준 `@SessionAttributes` 선언의 결과로 세션에 저장된 모델 속성과는 대조됩니다. 자세한 내용은 `@SessionAttribute`를 참조하십시오. |
| `@RequestAttribute` | 요청 속성에 접근하기 위함입니다. 자세한 내용은 `@RequestAttribute`를 참조하십시오. |
| 그 외 다른 인수 | 메서드 인수가 이 표의 이전 값 중 어느 것과도 일치하지 않고 단순 타입(`BeanUtils#isSimpleProperty`에 의해 결정됨)인 경우, `@RequestParam`으로 해석(resolve)됩니다. 그렇지 않으면 `@ModelAttribute`로 해석됩니다. |

---

## Spring MVC 컨트롤러 메소드의 다양한 반환 값과 그 의미

Spring MVC 컨트롤러의 핸들러 메소드는 다양한 타입의 값을 반환할 수 있으며, Spring MVC는 이 반환 값을 해석하여 적절한 HTTP 응답을 생성합니다.

이는 웹 페이지를 보여주는 것부터 REST API에서 데이터를 직접 전송하는 것까지 매우 유연한 응답 처리를 가능하게 합니다.

### 1. 주요 컨트롤러 메소드 반환 값 유형 및 설명

**가. HTTP 응답 본문에 직접 쓰기 (주로 REST API)**

- **`@ResponseBody` 어노테이션이 적용된 반환 값 (또는 `@RestController` 클래스의 메소드 반환 값):**
  - **역할:** 메소드의 반환 값을 뷰를 통해 렌더링하지 않고, `HttpMessageConverter`를 사용하여 **HTTP 응답 본문에 직접 쓰도록** 지시합니다.
  - **반환 타입 예시:** `String`, POJO (Plain Old Java Object), `Collection` (예: `List<User>`), `Map` 등.
  - **처리 과정:**
    1. 컨트롤러 메소드가 Java 객체나 기본 타입을 반환합니다.
    2. `RequestMappingHandlerAdapter`는 이 메소드에 `@ResponseBody` 효과가 적용된 것을 확인합니다.
    3. 클라이언트의 `Accept` 헤더와 반환된 객체의 타입을 고려하여 적절한 `HttpMessageConverter` (예: `MappingJackson2HttpMessageConverter` for JSON, `StringHttpMessageConverter` for `String`)를 선택합니다.
    4. 선택된 컨버터가 반환된 객체를 특정 미디어 타입(예: JSON, XML, 텍스트)의 데이터로 변환(직렬화)합니다.
    5. 변환된 데이터가 HTTP 응답 본문에 쓰여지고, 적절한 `Content-Type` 헤더가 설정됩니다.
  - **예시:**

      ```java
      @RestController
      public class ApiController {
          @GetMapping("/api/data")
          public MyData getData() {
              // MyData 객체가 JSON 등으로 변환되어 응답 본문에 쓰여짐
              return new MyData("value1", 123);
          }
      }
      ```

- **`HttpEntity<B>`, `ResponseEntity<B>`:**
  - **역할:** HTTP 응답의 **헤더(Headers), 상태 코드(Status Code), 본문(Body)**을 모두 완벽하게 제어하여 반환할 수 있게 합니다.
  - `B`는 본문의 타입이며, 이 본문은 `HttpMessageConverter`를 통해 변환됩니다.
  - `ResponseEntity`는 `HttpEntity`를 상속받아 HTTP 상태 코드를 더 편리하게 설정할 수 있는 기능을 제공합니다.
  - **예시:**

      ```java
      @GetMapping("/api/resource/{id}")
      public ResponseEntity<Resource> getResource(@PathVariable String id) {
          Resource resource = resourceService.findById(id);
          if (resource == null) {
              return ResponseEntity.notFound().build(); // 404 Not Found, 본문 없음
          }
          HttpHeaders headers = new HttpHeaders();
          headers.add("Custom-Header", "SomeValue");
          // Resource 객체가 JSON 등으로 변환되어 본문에 들어가고,
          // 상태 코드는 200 OK, 커스텀 헤더가 추가됨
          return new ResponseEntity<>(resource, headers, HttpStatus.OK);
      }
      ```

- **`HttpHeaders`**:
  - **역할:** HTTP 응답 본문 없이 **헤더만** 반환하고 싶을 때 사용합니다.
  - 주로 HTTP HEAD 요청을 처리하거나, 특정 헤더 정보만으로 응답을 구성할 때 사용될 수 있습니다.
  - 상태 코드는 기본적으로 200 OK가 되지만, `@ResponseStatus` 등으로 변경할 수 있습니다.
  - **예시:**

      ```java
      @GetMapping("/check-availability")
      public HttpHeaders checkAvailability() {
          HttpHeaders headers = new HttpHeaders();
          headers.set("X-Resource-Available", "true");
          return headers; // 본문 없이 헤더만 응답
      }
      ```

- **`ErrorResponse` / `ProblemDetail` (RFC 7807, RFC 9457 표준 에러 응답):**
  - **역할:** 표준화된 형식("Problem Details for HTTP APIs")으로 에러 응답 본문을 생성하여 반환합니다. REST API에서 일관된 에러 응답 구조를 제공하는 데 매우 유용합니다.
  - 이 객체들은 `HttpMessageConverter`에 의해 JSON이나 XML 등으로 변환되어 응답 본문에 포함됩니다.
  - **예시:**

      ```java
      @GetMapping("/find-error")
      public ResponseEntity<ProblemDetail> triggerError() {
          ProblemDetail problemDetail = ProblemDetail.forStatusAndDetail(HttpStatus.BAD_REQUEST, "잘못된 요청 파라미터입니다.");
          problemDetail.setType(URI.create("<https://example.com/probs/invalid-parameter>"));
          problemDetail.setTitle("유효하지 않은 파라미터");
          problemDetail.setProperty("invalid-params", List.of("param1", "param2"));
          return ResponseEntity.status(HttpStatus.BAD_REQUEST).body(problemDetail);
      }
      ```


**나. 뷰(View) 렌더링 관련**

- **`String` (뷰 이름):**
  - **역할:** 렌더링할 뷰의 **논리적인 이름**을 반환합니다.
  - **처리 과정:** `DispatcherServlet`은 이 문자열을 `ViewResolver`에게 전달하여 실제 `View` 객체를 찾고, 컨트롤러 메소드의 `Model` 인자 등을 통해 전달된 데이터와 함께 뷰를 렌더링합니다.
  - **예시:** `return "user/profile";` ( `user/profile.jsp` 또는 `user/profile.html` 등을 찾음)
- **`View` 객체 직접 반환:**
  - **역할:** 특정 `View` 구현체 인스턴스를 직접 반환하여 *명시적으로* 해당 뷰를 사용하여 렌더링하도록 합니다.
  - `ViewResolver`를 거치지 않거나, 특정 `View` 객체를 동적으로 생성하여 사용하고 싶을 때 유용합니다.
  - **예시:** `return new MyCustomPdfView(reportData);`
- **`java.util.Map`, `org.springframework.ui.Model`, `org.springframework.ui.ModelMap`:**
  - **역할:** 뷰에 전달할 모델 데이터 자체를 반환합니다.
  - **처리 과정:** 반환된 `Map`이나 `Model` 객체의 내용이 모델로 사용되며, 뷰 이름은 `RequestToViewNameTranslator`에 의해 현재 요청 URL을 기반으로 암묵적으로 결정됩니다.
  - **예시:**

      ```java
      @GetMapping("/info")
      public Model getInfo(Model model) {
          model.addAttribute("version", "1.0");
          return model; // 뷰 이름은 "/info" 등으로 추론됨
      }
      ```

- **`@ModelAttribute` 어노테이션이 붙은 객체 (또는 어노테이션 없는 POJO 반환 시):**
  - **역할:** 메소드가 반환하는 객체가 모델 속성(attribute)으로 추가됩니다. 뷰 이름은 `RequestToViewNameTranslator`에 의해 암묵적으로 결정됩니다.
  - 모델 속성의 이름은 기본적으로 클래스 이름의 첫 글자를 소문자로 바꾼 형태입니다.
  - **예시:**

      ```java
      @GetMapping("/current-user")
      public User getCurrentUser() { // User 객체가 "user"라는 이름으로 모델에 추가됨
          return userService.getCurrentUser();
      }
      ```

- **`ModelAndView` 객체:**
  - **역할:** 모델 데이터(Model)와 뷰 정보(View 이름 또는 `View` 객체)를 **하나의 객체로 묶어서** 반환합니다.
  - **예시:**

      ```java
      @GetMapping("/details")
      public ModelAndView showDetails() {
          ModelAndView mav = new ModelAndView("itemDetails"); // 뷰 이름 설정
          mav.addObject("item", itemService.getDetails()); // 모델 데이터 추가
          return mav;
      }
      ```

- **`FragmentsRendering`, `Collection<ModelAndView>`:**
  - HTML 조각(Fragment) 렌더링과 관련된 반환 타입입니다. 여러 뷰와 모델을 조합하여 응답을 구성할 때 사용될 수 있습니다 (예: 서버 사이드 인클루드(SSI)나 HTMX 같은 기술과 연동 시).

**다. `void` 반환 타입 및 비동기 처리**

- **`void` (또는 `null` 반환):**
  - **다양한 의미로 해석될 수 있습니다:**
    1. **직접 응답 처리:** 메소드 인자로 `HttpServletResponse`나 `OutputStream`을 받아 개발자가 직접 응답을 작성하고 완료한 경우.
    2. **`@ResponseStatus` 사용:** 메소드에 `@ResponseStatus` 어노테이션이 있어 특정 HTTP 상태 코드로 응답이 완료된 것으로 간주되는 경우 (보통 본문 없음).
    3. **ETag/Last-Modified 검사 성공:** HTTP 304 Not Modified 응답이 나가는 경우.
    4. **REST 컨트롤러에서 본문 없음:** `@RestController` 또는 `@ResponseBody`가 적용된 메소드가 `void`를 반환하면, 내용 없는 응답(예: HTTP 204 No Content)을 의미할 수 있습니다.
    5. **기본 뷰 이름 사용 (HTML 컨트롤러):** `RequestToViewNameTranslator`가 요청 URL로부터 뷰 이름을 추론하여 해당 뷰를 렌더링합니다.
- **비동기 처리 반환 타입:**
  - `DeferredResult<V>`
  - `Callable<V>`
  - `ListenableFuture<V>`, `java.util.concurrent.CompletionStage<V>`, `java.util.concurrent.CompletableFuture<V>`
  - `ResponseBodyEmitter`, `SseEmitter` (HTTP 스트리밍)
  - `StreamingResponseBody` (대용량 비동기 스트리밍)
  - **역할:** 이 타입들은 컨트롤러 메소드가 즉시 결과를 반환하지 않고, 별도의 스레드에서 비동기적으로 작업을 수행한 후 그 결과를 나중에 HTTP 응답으로 보낼 수 있도록 합니다. (자세한 내용은 "Asynchronous Requests" 챕터에서 다룹니다.)

**라. 그 외 다른 반환 값 (Any other return value)**

- 위에서 언급된 어떤 특정 처리 방식에도 해당하지 않는 객체를 반환하는 경우:
  - 만약 반환된 객체가 **단순 타입이 아니라면 (POJO 등)**, 그 객체는 **모델 속성(`@ModelAttribute`)**으로 간주됩니다. 모델 속성 이름은 클래스 이름(첫 글자 소문자)을 따르고, 뷰 이름은 `RequestToViewNameTranslator`에 의해 결정됩니다.
  - 만약 반환된 객체가 **단순 타입(String, int 등)**이라면, 기본적으로는 특별히 처리되지 않고 해석 불가(unresolved) 상태로 남을 수 있습니다. (이 경우 명시적인 `@ResponseBody`나 뷰 이름 반환이 필요할 수 있습니다.)

---
