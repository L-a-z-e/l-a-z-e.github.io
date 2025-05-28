---
title: Spring Web MVC - Annotated Controllers (Mapping Requests)
description: 
author: laze
date: 2025-05-24 00:00:01 +0900
categories: [Dev, SpringWebMVC]
tags: [SpringWebMVC]
---
---
**메서드 인수(Method Arguments)**
Reactive 스택에서의 동등한 기능 참조

다음 표는 지원되는 컨트롤러 메서드 인수를 설명합니다. Reactive 타입은 어떤 인수에도 지원되지 않습니다.

JDK 8의 `java.util.Optional`은 `required` 속성을 가진 어노테이션(예: `@RequestParam`, `@RequestHeader` 등)과 함께 메서드 인수로 지원되며, 이는 `required=false`와 동일합니다.

| 컨트롤러 메서드 인수 | 설명 |
| --- | --- |
| `WebRequest`, `NativeWebRequest` | Servlet API를 직접 사용하지 않고 요청 파라미터, 요청 및 세션 속성에 대한 일반적인 접근을 제공합니다. |
| `jakarta.servlet.ServletRequest`, `jakarta.servlet.ServletResponse` | 특정 요청 또는 응답 타입을 선택합니다 — 예를 들어, `ServletRequest`, `HttpServletRequest`, 또는 Spring의 `MultipartRequest`, `MultipartHttpServletRequest`가 있습니다. |
| `jakarta.servlet.http.HttpSession` | 세션의 존재를 강제합니다. 결과적으로 이러한 인수는 절대 `null`이 아닙니다. 세션 접근은 스레드로부터 안전하지 않다는 점에 유의하십시오. 여러 요청이 동시에 세션에 접근할 수 있는 경우 `RequestMappingHandlerAdapter` 인스턴스의 `synchronizeOnSession` 플래그를 `true`로 설정하는 것을 고려하십시오. |
| `jakarta.servlet.http.PushBuilder` | 프로그래밍 방식의 HTTP/2 리소스 푸시를 위한 Servlet 4.0 푸시 빌더 API입니다. Servlet 명세에 따라, 클라이언트가 해당 HTTP/2 기능을 지원하지 않으면 주입된 `PushBuilder` 인스턴스는 `null`일 수 있다는 점에 유의하십시오. |
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

## Spring MVC 컨트롤러 메소드의 다양한 인자들: 요청 정보 활용의 모든 것

Spring MVC의 어노테이션 기반 컨트롤러는 메소드 시그니처(파라미터 목록)가 매우 유연합니다.

개발자는 특정 인터페이스를 구현하거나 기본 클래스를 상속받을 필요 없이, 필요한 정보에 따라 다양한 타입의 인자를 메소드에 선언하여 사용할 수 있습니다.

Spring MVC는 `HandlerMethodArgumentResolver`라는 내부 컴포넌트를 통해 이러한 다양한 인자들을 해석하고 적절한 값을 주입해줍니다.

### 1. 주요 컨트롤러 메소드 인자 유형

다음은 컨트롤러 메소드에서 자주 사용되거나 중요한 인자 유형들입니다.

**가. HTTP 요청 및 응답 직접 접근**

- **`jakarta.servlet.ServletRequest` / `jakarta.servlet.http.HttpServletRequest`**:
  - **역할:** HTTP 요청에 대한 저수준(low-level) 정보에 직접 접근할 때 사용합니다.
  - **예시:** `request.getHeader("User-Agent")`, `request.getParameter("name")`, `request.getSession()` 등 서블릿 API를 직접 활용할 수 있습니다. `MultipartHttpServletRequest`를 사용하면 멀티파트 요청(파일 업로드)도 처리 가능합니다.
  - **사용 시점:** Spring MVC가 제공하는 고수준 추상화(예: `@RequestParam`, `@RequestHeader`)로 처리하기 어렵거나, 서블릿 API의 특정 기능을 직접 사용해야 할 때 유용합니다.
- **`jakarta.servlet.ServletResponse` / `jakarta.servlet.http.HttpServletResponse`**:
  - **역할:** HTTP 응답에 대한 저수준 제어가 필요할 때 사용합니다.
  - **예시:** `response.setContentType("application/json")`, `response.addHeader("Custom-Header", "value")`, `response.getWriter().println("Hello")` 등 응답 헤더나 본문을 직접 조작할 수 있습니다.
  - **주의:** 대부분의 경우 `@ResponseBody`, `ResponseEntity`, 또는 뷰 리졸루션을 통해 응답을 처리하는 것이 권장됩니다. `HttpServletResponse`를 직접 사용하면 Spring MVC의 일부 기능(예: `HttpMessageConverter` 자동 적용)을 우회할 수 있습니다.
- **`java.io.InputStream` / `java.io.Reader`**:
  - **역할:** HTTP 요청 본문(body)의 내용을 원시(raw) 스트림 형태로 직접 읽어올 때 사용합니다.
  - **사용 시점:** 매우 큰 데이터를 스트리밍 방식으로 처리하거나, Spring의 `HttpMessageConverter`가 지원하지 않는 특수한 형식의 데이터를 직접 파싱해야 할 때 유용합니다.
- **`java.io.OutputStream` / `java.io.Writer`**:
  - **역할:** HTTP 응답 본문(body)에 원시 스트림 형태로 직접 데이터를 쓸 때 사용합니다.
  - **사용 시점:** 대용량 파일을 다운로드하거나, 특수한 형식으로 응답을 생성해야 할 때 유용합니다. `HttpServletResponse`와 마찬가지로, 고수준 추상화를 사용하는 것이 일반적입니다.

**나. HTTP 요청 세부 정보 접근 (어노테이션 기반)**

- **`@PathVariable`**:
  - **역할:** URI 템플릿 변수(Path Variable)의 값을 메소드 인자로 받습니다.
  - **예시:** `@GetMapping("/users/{userId}")` 일 때, `public String getUser(@PathVariable Long userId, ...)`
  - 타입 변환이 자동으로 수행됩니다 (예: 문자열 "123" -> `Long` 타입 123).
- **`@RequestParam`**:
  - **역할:** HTTP 요청 파라미터(쿼리 스트링 또는 폼 데이터)의 값을 메소드 인자로 받습니다.
  - **예시:** URL이 `/search?keyword=spring` 일 때, `public String search(@RequestParam String keyword, ...)`
  - **속성:**
    - `value` 또는 `name`: 파라미터 이름 지정 (생략 시 메소드 인자 이름 사용).
    - `required`: 해당 파라미터가 필수인지 여부 (기본값 `true`). `false`로 설정하면 해당 파라미터가 없어도 오류 발생 안 함.
    - `defaultValue`: 파라미터가 없거나 비어있을 경우 사용할 기본값.
  - `MultipartFile` 타입과 함께 사용하여 업로드된 파일도 받을 수 있습니다.
- **`@RequestHeader`**:
  - **역할:** 특정 HTTP 요청 헤더의 값을 메소드 인자로 받습니다.
  - **예시:** `public String process(@RequestHeader("User-Agent") String userAgent, ...)`
  - `@RequestParam`과 유사하게 `value`, `required`, `defaultValue` 속성을 가집니다.
- **`@CookieValue`**:
  - **역할:** 특정 쿠키의 값을 메소드 인자로 받습니다.
  - **예시:** `public String welcome(@CookieValue("sessionId") String sessionId, ...)`
  - `@RequestParam`과 유사하게 `value`, `required`, `defaultValue` 속성을 가집니다.
- **`@RequestBody`**:
  - **역할:** HTTP 요청 본문(body)의 전체 내용을 특정 Java 객체로 변환(역직렬화)하여 메소드 인자로 받습니다.
  - **사용 시점:** 주로 JSON, XML 등의 데이터를 요청 본문에 담아 보낼 때 사용합니다 (예: REST API에서 POST, PUT 요청).
  - `HttpMessageConverter`가 이 변환 작업을 수행합니다.
  - **예시:** `public String createUser(@RequestBody UserDto userDto, ...)`
- **`HttpEntity<B>`**:
  - **역할:** HTTP 요청 헤더와 본문(`B` 타입의 객체) 모두에 접근할 수 있게 해줍니다.
  - 요청 본문은 `HttpMessageConverter`를 통해 `B` 타입으로 변환됩니다.
  - `HttpEntity`는 헤더 정보(`getHeaders()`)와 본문 정보(`getBody()`)를 모두 포함합니다.
  - **응답 시에도 사용 가능:** `ResponseEntity<B>`는 `HttpEntity<B>`를 상속받아 HTTP 상태 코드까지 포함하여 응답을 구성할 때 사용됩니다.
- **`@RequestPart`**:
  - **역할:** `multipart/form-data` 요청의 개별 파트(part)에 접근할 때 사용합니다. 주로 파일 업로드와 함께 텍스트 데이터를 받을 때 유용합니다.
  - 각 파트의 본문은 `HttpMessageConverter`를 통해 변환될 수 있습니다.
  - **예시 (파일과 JSON 데이터를 함께 받는 경우):**

      ```java
      @PostMapping(value = "/upload-with-json", consumes = MediaType.MULTIPART_FORM_DATA_VALUE)
      public ResponseEntity<String> uploadWithJson(
              @RequestPart("file") MultipartFile file,
              @RequestPart("jsonData") MyJsonDto jsonData) { // jsonData 파트는 JSON 문자열로 오고, HttpMessageConverter가 MyJsonDto로 변환
          // ... 파일 처리 ...
          // ... jsonData 처리 ...
          return ResponseEntity.ok("Uploaded!");
      }
      
      ```


**다. 모델(Model), 세션(Session), 리다이렉트(Redirect) 관련**

- **`java.util.Map`, `org.springframework.ui.Model`, `org.springframework.ui.ModelMap`**:
  - **역할:** 뷰(View)에 전달할 데이터를 담는 모델 객체에 접근합니다.
  - 메소드에서 이 타입의 인자를 선언하면 Spring MVC가 자동으로 해당 객체를 주입해줍니다. 여기에 데이터를 추가하면 뷰 템플릿에서 사용할 수 있습니다.
  - `Model`이 가장 일반적으로 사용되는 인터페이스입니다.
- **`RedirectAttributes`**:
  - **역할:** 리다이렉트(redirect) 시 데이터를 전달하는 데 사용됩니다.
  - **두 가지 종류의 속성 추가 가능:**
    - **일반 속성 (Query Parameters):** `addAttribute(String name, Object value)`를 사용하면 리다이렉트 URL의 쿼리 파라미터로 추가됩니다. (예: `/target?name=value`)
    - **플래시 속성 (Flash Attributes):** `addFlashAttribute(String name, Object value)`를 사용하면, 데이터가 세션에 일시적으로 저장되었다가 리다이렉트된 후 **첫 번째 요청에서만** 모델에 자동으로 추가되고 세션에서 제거됩니다. (예: "저장 성공!" 같은 1회성 메시지 전달에 유용)
- **`@ModelAttribute`**:
  - **여러 용도로 사용됩니다:**
    1. **메소드 인자:** HTTP 요청 파라미터를 특정 객체(커맨드 객체 또는 폼 객체)에 바인딩(데이터 채우기)하고, 해당 객체를 모델에도 자동으로 추가합니다. 유효성 검사(`@Valid`)와 함께 자주 사용됩니다.

        ```java
        // UserForm.java (폼 객체)
        public class UserForm { private String name; private int age; /* getter, setter */ }
        
        // Controller
        @PostMapping("/users")
        public String processUserForm(@ModelAttribute("user") UserForm userForm, BindingResult result, Model model) {
            // userForm 객체에는 요청 파라미터가 바인딩되어 있음
            // "user"라는 이름으로 모델에도 추가됨
            if (result.hasErrors()) { return "userForm"; }
            // ...
        }
        
        ```

    2. **메소드 레벨:** `@RequestMapping` 메소드가 호출되기 전에 먼저 실행되어 모델에 특정 데이터를 미리 추가하는 역할을 합니다. (이 부분은 "메소드 인자"가 아닌 "어노테이션" 자체의 기능입니다.)
  - 단순 타입이 아닌 객체를 요청 파라미터로부터 바인딩 받을 때 유용합니다.
- **`Errors`, `BindingResult`**:
  - **역할:** `@ModelAttribute`로 바인딩된 커맨드 객체 또는 `@RequestBody`, `@RequestPart`로 바인딩된 객체에 대한 **데이터 바인딩 오류나 유효성 검사(validation) 오류 정보**를 담습니다.
  - **중요:** 반드시 **검사 대상이 되는 인자 바로 뒤에** 선언해야 합니다.
  - `hasErrors()` 메소드로 오류 유무를 확인하고, `getFieldErrors()` 등으로 구체적인 오류 내용을 가져올 수 있습니다.
- **`jakarta.servlet.http.HttpSession`**:
  - **역할:** HTTP 세션 객체에 직접 접근합니다.
  - 이 인자를 선언하면 세션이 존재함이 보장됩니다 (없으면 생성될 수 있음).
  - **주의:** 세션 객체 자체는 스레드 안전하지 않으므로, 동시에 여러 요청이 동일 세션에 접근할 가능성이 있다면 `RequestMappingHandlerAdapter`의 `synchronizeOnSession` 플래그를 `true`로 설정하는 것을 고려해야 합니다. (일반적으로는 `@SessionAttributes`나 Spring Session을 사용하는 것이 더 권장됩니다.)
- **`SessionStatus`**:
  - **역할:** 클래스 레벨에 `@SessionAttributes` 어노테이션으로 세션에 저장하도록 지정된 모델 속성들을 **세션에서 제거**할 때 사용합니다.
  - 주로 폼 처리 과정이 완료되었을 때 `setComplete()` 메소드를 호출하여 세션 데이터를 정리합니다.
- **`@SessionAttribute`**:
  - **역할:** `@SessionAttributes`를 통해 모델에서 세션으로 이동된 속성이 아닌, **세션에 직접 저장된 어떤 속성이라도** 이름으로 접근할 수 있게 해줍니다.
  - 예: `public String showCart(@SessionAttribute("cart") ShoppingCart cart, ...)` (세션에 "cart"라는 이름으로 저장된 `ShoppingCart` 객체를 가져옴)
- **`@RequestAttribute`**:
  - **역할:** `HttpServletRequest`의 속성(attribute)에 직접 접근합니다. (요청 파라미터가 아님)
  - 주로 필터나 인터셉터에서 요청 객체에 저장해둔 값을 컨트롤러에서 사용할 때 유용합니다.
  - 예: `public String displayInfo(@RequestAttribute("userInfo") UserInfo userInfo, ...)`

**라. 기타 유용한 인자들**

- **`java.security.Principal`**:
  - **역할:** 현재 인증된 사용자의 정보를 나타내는 `Principal` 객체를 받습니다.
  - Spring Security와 함께 사용될 때, 현재 로그인한 사용자의 정보를 가져오는 데 유용합니다.
  - `@AuthenticationPrincipal` 어노테이션과 함께 사용하면 Spring Security의 `Authentication` 객체로부터 `Principal`을 더 유연하게 주입받을 수 있습니다.
- **`HttpMethod`**:
  - **역할:** 현재 요청의 HTTP 메소드(GET, POST, PUT 등)를 `HttpMethod` 열거형 타입으로 받습니다.
- **`java.util.Locale`**:
  - **역할:** 현재 요청에 대해 결정된 로케일 정보를 받습니다. (이전에 배운 `LocaleResolver`에 의해 결정됨)
- **`java.util.TimeZone` / `java.time.ZoneId`**:
  - **역할:** 현재 요청과 관련된 시간대 정보를 받습니다. (`LocaleContextResolver`가 시간대 정보도 함께 제공할 경우)
- **`jakarta.servlet.http.PushBuilder` (Servlet 4.0+)**:
  - **역할:** HTTP/2 서버 푸시(Server Push) 기능을 프로그래밍 방식으로 사용하기 위한 API입니다. 클라이언트가 요청하지 않은 리소스를 서버가 미리 보내줄 수 있습니다. (클라이언트가 HTTP/2를 지원해야 함)
- **`UriComponentsBuilder`**:
  - **역할:** 현재 요청의 컨텍스트(호스트, 포트, 스킴, 컨텍스트 경로 등)를 기준으로 상대적인 또는 절대적인 URL을 안전하고 편리하게 생성할 수 있도록 도와주는 빌더 클래스입니다. (예: HATEOAS 링크 생성 시 유용)

**마. 그 외 다른 인수 (Any other argument)**

- 만약 메소드 인자가 위 목록의 어떤 특별한 타입이나 어노테이션과도 일치하지 않는 경우, Spring MVC는 다음과 같이 처리합니다:
  - **단순 타입 (Simple Type)** (예: `int`, `String`, `boolean` 등, `BeanUtils.isSimpleProperty()`로 판단): **`@RequestParam`*으로 간주합니다. 즉, 해당 인자 이름과 동일한 요청 파라미터를 찾아서 값을 주입하려고 시도합니다. 이 경우 `@RequestParam` 어노테이션을 생략할 수 있습니다.

      ```java
      // @RequestParam String name 과 동일하게 동작 (단, required=true가 기본)
      public String greet(String name, Model model) { ... }
      
      ```

  - **복합 타입 (Complex Type, POJO):** **`@ModelAttribute`*로 간주합니다. Spring MVC는 해당 타입의 객체를 생성하고, 요청 파라미터들을 객체의 필드에 바인딩하려고 시도하며, 생성된 객체를 모델에 자동으로 추가합니다.

      ```java
      // @ModelAttribute UserForm userForm 과 동일하게 동작
      public String processForm(UserForm userForm, BindingResult result) { ... }
      
      ```


**JDK 8 `java.util.Optional` 지원:**

- `@RequestParam`, `@RequestHeader` 등 `required` 속성을 가지는 어노테이션과 함께 `Optional<String> paramName`과 같이 사용할 수 있습니다.
- 이는 `required=false`와 동일한 효과를 가집니다. 즉, 해당 파라미터나 헤더가 없어도 오류가 발생하지 않고 `Optional.empty()`가 주입됩니다.

    ```java
    @GetMapping("/search")
    public String search(@RequestParam Optional<String> keyword, Model model) {
        if (keyword.isPresent()) {
            model.addAttribute("searchTerm", keyword.get());
        } else {
            model.addAttribute("searchTerm", "N/A");
        }
        return "searchResults";
    }
    
    ```


---
