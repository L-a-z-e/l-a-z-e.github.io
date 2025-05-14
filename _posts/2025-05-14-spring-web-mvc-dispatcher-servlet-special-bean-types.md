---
title: Spring Web MVC - Dispatcher Servlet (Context Hierarchy)
description: 
author: laze
date: 2025-05-14 00:00:05 +0900
categories: [Dev, SpringWebMVC]
tags: [SpringWebMVC]
---
**특별한 빈 타입 (Special Bean Types)**

`DispatcherServlet`은 요청을 처리하고 적절한 응답을 렌더링하기 위해 특별한 빈(special beans)들에게 위임합니다.

"특별한 빈"이란 프레임워크 계약(framework contracts)을 구현하는 Spring이 관리하는 객체 인스턴스를 의미합니다.

이러한 빈들은 대개 내장된 계약(built-in contracts)을 가지고 있지만, 여러분은 그 속성을 사용자 정의하거나 확장 또는 교체할 수 있습니다.

다음 표는 `DispatcherServlet`에 의해 탐지되는 특별한 빈들을 나열합니다:

| 빈 타입 (Bean type) | 설명 (Explanation) |
| --- | --- |
| `HandlerMapping` | 전처리(pre-processing) 및 후처리(post-processing)를 위한 인터셉터 목록과 함께 요청을 핸들러(handler)에 매핑합니다. 매핑은 특정 기준에 따라 이루어지며, 세부 사항은 `HandlerMapping` 구현체마다 다릅니다. <br><br> 두 가지 주요 `HandlerMapping` 구현체는 `RequestMappingHandlerMapping`(@RequestMapping 어노테이션이 붙은 메소드 지원)과 `SimpleUrlHandlerMapping`(URI 경로 패턴과 핸들러 간의 명시적 등록 유지)입니다. |
| `HandlerAdapter` | `DispatcherServlet`이 요청에 매핑된 핸들러를 호출하는 것을 돕습니다. 핸들러가 실제로 어떻게 호출되는지와는 무관합니다. 예를 들어, 어노테이션이 달린 컨트롤러를 호출하려면 어노테이션을 해석해야 합니다. `HandlerAdapter`의 주요 목적은 `DispatcherServlet`을 이러한 세부 사항으로부터 보호하는 것입니다. |
| `HandlerExceptionResolver` | 예외를 해결하기 위한 전략으로, 예외를 핸들러, HTML 에러 뷰 또는 다른 대상으로 매핑할 수 있습니다. |
| `ViewResolver` | 핸들러로부터 반환된 논리적인 문자열 기반 뷰 이름을 실제 `View` (응답을 렌더링하는 데 사용됨)로 해석합니다.  |
| `LocaleResolver`, `LocaleContextResolver` | 국제화된 뷰를 제공할 수 있도록 클라이언트가 사용하고 있는 로케일(Locale) 및 경우에 따라 시간대(time zone)를 해석합니다. |
| `ThemeResolver` | 웹 애플리케이션이 사용할 수 있는 테마(예: 개인화된 레이아웃 제공)를 해석합니다.  |
| `MultipartResolver` | 멀티파트 파싱 라이브러리의 도움을 받아 멀티파트 요청(예: 브라우저 폼 파일 업로드)을 파싱하기 위한 추상화입니다. |
| `FlashMapManager` | 리다이렉트(redirect)를 통해 한 요청에서 다른 요청으로 속성을 전달하는 데 사용될 수 있는 "입력(input)" 및 "출력(output)" `FlashMap`을 저장하고 검색합니다. |

---

## DispatcherServlet의 조력자들: 특별한 빈 타입 이해하기

`DispatcherServlet`은 웹 요청 처리의 총괄 지휘자이지만, 모든 일을 혼자 다 하지는 않습니다. 마치 오케스트라 지휘자가 각 악기 연주자들에게 지시를 내리듯, `DispatcherServlet`도 다양한 전문적인 역할을 수행하는 **특별한 빈(Special Beans)**들에게 작업을 위임합니다.

여기서 "특별한 빈"이란, Spring 프레임워크가 미리 정해놓은 특정 인터페이스(계약, contract)를 구현한 Spring 관리 객체들을 말합니다. Spring은 이러한 인터페이스를 구현한 빈들을 자동으로 감지하고, 웹 요청 처리 흐름의 적절한 시점에 호출하여 사용합니다.

개발자는 이러한 특별한 빈들의 기본 구현체를 그대로 사용하거나, 필요에 따라 설정을 변경하거나, 직접 구현체를 만들어 교체할 수도 있습니다.

### DispatcherServlet이 사용하는 주요 특별한 빈들

아래 표는 `DispatcherServlet`이 사용하는 핵심적인 특별한 빈들과 그 역할을 요약한 것입니다. 각 빈에 대해서는 앞으로 더 자세히 배우게 될 텐데, 지금은 각 빈이 **어떤 종류의 일을 하는지** 큰 그림을 이해하는 데 초점을 맞추시면 됩니다.

| 빈 타입 (인터페이스) | 주요 역할 | 핵심 설명 |
| --- | --- | --- |
| **`HandlerMapping`** | 요청 URL과 핸들러(컨트롤러) 매핑 | 클라이언트 요청(URL, HTTP 메소드 등)을 분석하여, 이 요청을 처리할 적절한 **핸들러(Handler)**, 즉 컨트롤러의 메소드를 찾아줍니다. 요청 전후로 실행될 인터셉터 목록도 함께 결정합니다. |
| **`HandlerAdapter`** | 핸들러(컨트롤러 메소드) 실행 | `HandlerMapping`이 찾아준 핸들러를 실제로 호출(실행)하는 방법을 알고 있습니다. `DispatcherServlet`은 핸들러를 어떻게 실행해야 하는지 구체적인 방법을 몰라도 되도록, 이 어댑터에게 실행을 위임합니다. |
| **`HandlerExceptionResolver`** | 예외 처리 | 요청 처리 과정에서 발생한 예외를 어떻게 처리할지 결정합니다. 특정 예외를 다른 페이지로 안내하거나, 특정 HTTP 상태 코드를 반환하도록 할 수 있습니다. |
| **`ViewResolver`** | 뷰 이름(논리적)을 실제 뷰 객체로 변환 | 컨트롤러가 반환한 논리적인 뷰 이름(예: "home", "user/list")을 실제 렌더링 가능한 `View` 객체(예: `/WEB-INF/views/home.jsp`)로 변환해줍니다. 이전에 잠깐 언급되었죠? |
| **`LocaleResolver`** | 클라이언트의 지역(Locale) 정보 결정 | 다국어 지원(i18n) 웹사이트를 만들 때, 클라이언트가 사용하는 언어 및 지역 설정을 파악하여 그에 맞는 뷰를 제공할 수 있도록 합니다. (예: 한국어 사용자에게는 한글 페이지, 영어 사용자에게는 영문 페이지) |
| `LocaleContextResolver` | (선택) `LocaleResolver`의 확장 | 로케일 정보뿐만 아니라 시간대(Time Zone) 정보까지 함께 관리할 수 있는 확장된 인터페이스입니다. |
| `ThemeResolver` | 웹 애플리케이션 테마 결정 | 웹사이트의 디자인 테마(예: 밝은 테마, 어두운 테마, 사용자별 맞춤 테마)를 결정하고 적용할 수 있도록 합니다. |
| **`MultipartResolver`** | 멀티파트 요청(파일 업로드) 처리 | `<form>` 태그를 통해 파일이 업로드될 때, 이러한 멀티파트 요청을 파싱하여 업로드된 파일 데이터에 접근할 수 있도록 도와줍니다. (예: `<input type="file">`로 전송된 파일 처리) |
| `FlashMapManager` | 리다이렉션 시 데이터 전달 관리 (Flash Attributes) | 주로 리다이렉트(redirect) 상황에서, 한 번의 요청에서 다음 요청으로 데이터를 안전하게 전달하는 데 사용됩니다. (예: 게시글 작성 후 목록 페이지로 리다이렉트하면서 "게시글이 성공적으로 등록되었습니다." 같은 메시지 전달) |

**굵게 표시된 빈들 (`HandlerMapping`, `HandlerAdapter`, `HandlerExceptionResolver`, `ViewResolver`, `LocaleResolver`, `MultipartResolver`)은 Spring MVC를 사용한다면 거의 필수적으로 이해하고 사용하게 되는 매우 중요한 컴포넌트들입니다.**

### 각 빈의 역할에 대한 좀 더 자세한 설명 및 예시 (개념적)

1. **`HandlerMapping` (핸들러 매핑)**
  - **상황:** 사용자가 브라우저에 `/myapp/users/1` 이라는 URL을 입력하고 엔터를 칩니다.
  - **역할:** `DispatcherServlet`은 이 요청을 받고 `HandlerMapping`에게 "이 `/myapp/users/1` 요청을 처리할 컨트롤러가 누구니?"라고 묻습니다.
  - `HandlerMapping`은 자신의 설정(예: `@RequestMapping` 어노테이션 정보, XML 설정 등)을 뒤져서 이 URL을 처리하도록 등록된 특정 컨트롤러의 특정 메소드를 찾아냅니다.
  - **주요 구현체:**
    - `RequestMappingHandlerMapping`: `@Controller` 클래스 내의 `@RequestMapping`, `@GetMapping`, `@PostMapping` 등의 어노테이션을 분석하여 핸들러를 매핑합니다. (가장 흔하게 사용됨)
    - `SimpleUrlHandlerMapping`: 개발자가 직접 URL 패턴과 컨트롤러 빈 이름을 명시적으로 매핑합니다. (덜 사용됨)
2. **`HandlerAdapter` (핸들러 어댑터)**
  - **상황:** `HandlerMapping`이 `UserController`의 `getUserById(Long id)` 메소드가 `/myapp/users/1` 요청을 처리할 것이라고 알려줬습니다.
  - **역할:** `DispatcherServlet`은 이 `getUserById` 메소드를 어떻게 실행해야 할지 모릅니다. (예: `@PathVariable` 어노테이션을 어떻게 처리하고, 파라미터는 어떻게 주입해야 하는지 등)
  - `HandlerAdapter`는 다양한 유형의 핸들러(어노테이션 기반 컨트롤러, 특정 인터페이스를 구현한 컨트롤러 등)를 실행하는 방법을 알고 있습니다. `DispatcherServlet`은 해당 핸들러에 맞는 `HandlerAdapter`를 찾아 실행을 위임합니다.
  - **핵심:** `DispatcherServlet`이 핸들러 호출의 세부 사항으로부터 분리될 수 있게 합니다.
3. **`HandlerExceptionResolver` (핸들러 예외 리졸버)**
  - **상황:** `UserController`의 `getUserById` 메소드 실행 중, 데이터베이스 연결 오류로 `SQLException`이 발생했습니다.
  - **역할:** `DispatcherServlet`은 이 예외를 `HandlerExceptionResolver`에게 넘깁니다.
  - `HandlerExceptionResolver`는 이 `SQLException`을 어떻게 처리할지 결정합니다. (예: 사용자에게 "죄송합니다. 오류가 발생했습니다."라는 에러 페이지를 보여주거나, 특정 HTTP 상태 코드를 반환하거나, 로그를 남기는 등)
4. **`ViewResolver` (뷰 리졸버)**
  - **상황:** `UserController`의 `getUserById` 메소드가 성공적으로 사용자 정보를 조회했고, 이제 이 정보를 `user-detail`이라는 뷰를 통해 보여주려고 합니다. 컨트롤러는 `"user-detail"`이라는 문자열(논리적 뷰 이름)을 반환합니다.
  - **역할:** `DispatcherServlet`은 `"user-detail"`이라는 문자열을 `ViewResolver`에게 전달합니다.
  - `ViewResolver`는 설정된 규칙(예: 접두사 `/WEB-INF/views/`, 접미사 `.jsp`)에 따라 이 논리적 뷰 이름을 실제 뷰 파일 경로(예: `/WEB-INF/views/user-detail.jsp`)로 변환하고, 해당 뷰를 렌더링할 수 있는 `View` 객체를 반환합니다.
5. **`LocaleResolver` (로케일 리졸버)**
  - **상황:** 웹사이트가 한국어와 영어를 모두 지원합니다. 미국에서 접속한 사용자가 있습니다.
  - **역할:** `LocaleResolver`는 요청 헤더(예: `Accept-Language`), 쿠키, 세션 등을 통해 클라이언트가 선호하는 로케일(여기서는 `en_US`)을 결정합니다.
  - 이 정보는 이후 뷰를 선택하거나, 메시지를 해당 언어로 보여주는 데 사용됩니다.
6. **`MultipartResolver` (멀티파트 리졸버)**
  - **상황:** 사용자가 게시글 작성 폼에서 제목, 내용과 함께 첨부파일(이미지 파일)을 선택하고 '전송' 버튼을 눌렀습니다.
  - **역할:** `DispatcherServlet`은 이 요청이 파일 업로드를 포함하는 멀티파트 요청임을 감지하고 `MultipartResolver`에게 처리를 위임합니다.
  - `MultipartResolver`는 요청 스트림을 파싱하여 텍스트 데이터와 파일 데이터를 분리하고, 컨트롤러에서 업로드된 파일에 쉽게 접근할 수 있도록 `MultipartFile` 객체 형태로 제공합니다.

### 왜 이렇게 많은 빈들이 필요할까요?

- **관심사의 분리 (SoC, Separation of Concerns):** 각 빈은 특정 기능에만 집중합니다. 이로 인해 코드가 더 모듈화되고, 유지보수 및 테스트가 용이해집니다.
- **유연성 및 확장성:** Spring은 각 인터페이스에 대한 기본 구현체를 제공하지만, 개발자는 필요에 따라 이러한 구현체를 교체하거나 확장하여 자신만의 로직을 추가할 수 있습니다. 예를 들어, 특별한 방식으로 핸들러를 매핑하고 싶다면 자신만의 `HandlerMapping` 구현체를 만들어 등록할 수 있습니다.
- **`DispatcherServlet`의 단순화:** `DispatcherServlet`은 요청 처리의 전체 흐름만 제어하고, 구체적인 작업은 전문적인 빈들에게 위임함으로써 자신의 코드를 단순하게 유지할 수 있습니다.

---

### `HandlerAdapter`가 필요한 이유: 다양한 핸들러 유형 처리

Spring MVC는 매우 유연해서 다양한 방식으로 핸들러(요청 처리기)를 작성할 수 있습니다.

1. **`@Controller` 어노테이션과 `@RequestMapping` 어노테이션을 사용하는 방식 (가장 일반적):**

    ```java
    @Controller
    public class MyController {
        @GetMapping("/hello/{name}")
        @ResponseBody // HTTP 응답 본문에 직접 문자열을 쓴다.
        public String sayHello(@PathVariable String name, @RequestParam String greeting) {
            return greeting + ", " + name + "!";
        }
    }
    ```

   이 경우, `DispatcherServlet`은 `sayHello` 메소드를 실행하기 위해 다음 작업들을 해야 합니다:

  - URL 경로에서 `name` 값을 추출하여 `@PathVariable`이 붙은 `name` 파라미터에 주입해야 합니다.
  - 요청 파라미터에서 `greeting` 값을 추출하여 `@RequestParam`이 붙은 `greeting` 파라미터에 주입해야 합니다.
  - 메소드가 반환한 문자열을 `@ResponseBody` 어노테이션에 따라 HTTP 응답 본문에 직접 써야 합니다.
2. **`HttpRequestHandler` 인터페이스를 구현하는 방식:**

    ```java
    public class MyHttpRequestHandler implements HttpRequestHandler {
        @Override
        public void handleRequest(HttpServletRequest request, HttpServletResponse response)
                throws ServletException, IOException {
            response.getWriter().write("Hello from HttpRequestHandler!");
        }
    }
    ```

   이 경우, `DispatcherServlet`은 `handleRequest` 메소드를 직접 호출하고, `HttpServletRequest`와 `HttpServletResponse` 객체를 넘겨주기만 하면 됩니다. 어노테이션 처리 같은 복잡한 과정이 없습니다.

3. **(과거 방식) `Controller` 인터페이스 ( `org.springframework.web.servlet.mvc.Controller` )를 구현하는 방식:**

    ```java
    public class LegacyController implements org.springframework.web.servlet.mvc.Controller {
        @Override
        public ModelAndView handleRequest(HttpServletRequest request, HttpServletResponse response) throws Exception {
            ModelAndView mav = new ModelAndView("someView");
            mav.addObject("message", "Hello from legacy Controller!");
            return mav;
        }
    }
    ```

   이 경우, `DispatcherServlet`은 `handleRequest` 메소드를 호출하고, 반환된 `ModelAndView` 객체를 처리해야 합니다.


**핵심은 `DispatcherServlet`이 이 모든 다양한 핸들러 유형을 직접 알고 처리하려고 하면 코드가 매우 복잡해진다는 것입니다.** `@PathVariable`은 어떻게 처리하고, `@RequestParam`은 어떻게 하며, `HttpRequestHandler`는 또 어떻게 호출하고... 이 모든 세부 사항을 `DispatcherServlet`이 다 떠안게 됩니다.

### `HandlerAdapter`의 역할: "어댑터 패턴"의 적용

여기서 **어댑터 패턴(Adapter Pattern)**이 등장합니다. `HandlerAdapter`는 서로 다른 인터페이스(호출 방식)를 가진 다양한 핸들러들을 `DispatcherServlet`이 일관된 방식으로 호출할 수 있도록 중간에서 '번역' 또는 '조정'해주는 역할을 합니다.

- `DispatcherServlet`은 `HandlerMapping`으로부터 실행할 핸들러 객체(예: `MyController`의 `sayHello` 메소드 정보)를 받습니다.
- `DispatcherServlet`은 등록된 `HandlerAdapter`들에게 "이 핸들러를 지원하는 어댑터가 누구냐?" (`supports(Object handler)` 메소드 호출)라고 묻습니다.
- 해당 핸들러를 지원하는 `HandlerAdapter`(예: 어노테이션 기반 컨트롤러를 위한 `RequestMappingHandlerAdapter`)가 선택됩니다.
- `DispatcherServlet`은 선택된 `HandlerAdapter`에게 "이 핸들러를 실행하고 결과를 `ModelAndView` 형태로 달라" (`handle(HttpServletRequest request, HttpServletResponse response, Object handler)` 메소드 호출)고 요청합니다.

**`HandlerAdapter`가 구체적으로 하는 일 (예: `RequestMappingHandlerAdapter`의 경우):**

`RequestMappingHandlerAdapter`는 어노테이션 기반 컨트롤러 메소드를 호출하기 위해 다음과 같은 복잡한 작업들을 내부적으로 수행합니다:

1. **인자 해석 (Argument Resolving):**
  - 컨트롤러 메소드의 파라미터들을 분석합니다. (예: `@PathVariable String name`, `@RequestParam String greeting`, `Model model`, `HttpServletRequest request` 등)
  - 각 파라미터에 어떤 어노테이션이 붙어 있는지, 타입은 무엇인지 등을 확인합니다.
  - `@PathVariable`: URL 경로에서 값을 추출합니다.
  - `@RequestParam`: 요청 파라미터(쿼리 스트링 또는 form data)에서 값을 추출합니다.
  - `@RequestBody`: 요청 본문(JSON, XML 등)을 객체로 변환합니다.
  - `Model`, `HttpServletRequest`, `HttpServletResponse` 등: 필요한 객체를 생성하거나 전달받아 주입합니다.
  - 이러한 작업을 위해 내부적으로 `HandlerMethodArgumentResolver`라는 인터페이스의 구현체들을 사용합니다.
2. **메소드 호출:**
  - 해석된 인자들을 사용하여 실제 컨트롤러 메소드를 호출합니다. (예: `sayHello("John", "Hi")`)
3. **반환 값 처리 (Return Value Handling):**
  - 컨트롤러 메소드가 반환한 값을 분석합니다. (예: `String`, `ModelAndView`, `@ResponseBody`가 붙은 객체 등)
  - `String`: 뷰 이름으로 간주하고 `ModelAndView` 객체를 생성합니다.
  - `ModelAndView`: 그대로 사용합니다.
  - `@ResponseBody` + 객체/문자열: HTTP 응답 본문에 직접 쓸 수 있도록 `HttpMessageConverter`를 사용하여 변환합니다. (예: 객체를 JSON 문자열로)
  - `void` (메소드 반환 타입이 없는 경우): 응답이 이미 직접 처리되었거나 (예: `response.getWriter().write(...)`) 또는 뷰 이름이 암묵적으로 결정될 수 있는 경우.
  - 이러한 작업을 위해 내부적으로 `HandlerMethodReturnValueHandler`라는 인터페이스의 구현체들을 사용합니다.
4. **결과 반환:**
  - 최종적으로 `DispatcherServlet`이 이해할 수 있는 형태인 `ModelAndView` 객체를 만들어 반환합니다. (만약 `@ResponseBody` 등으로 직접 응답이 처리되었다면 `null`을 반환할 수도 있습니다.)

### 코드 예시 (개념적인 흐름)

실제 Spring 내부 코드는 훨씬 복잡하지만, 개념적인 흐름을 보여드리면 다음과 같습니다.

**1. DispatcherServlet의 `doDispatch` 메소드 (일부)**

```java
// DispatcherServlet.java (간략화된 개념 코드)
protected void doDispatch(HttpServletRequest request, HttpServletResponse response) throws Exception {
    HttpServletRequest processedRequest = request;
    HandlerExecutionChain mappedHandler = null;
    ModelAndView mv = null;
    Exception dispatchException = null;

    try {
        // 1. HandlerMapping을 통해 핸들러 찾기
        mappedHandler = getHandler(processedRequest);
        if (mappedHandler == null) {
            noHandlerFound(processedRequest, response);
            return;
        }

        // 2. 해당 핸들러를 실행할 HandlerAdapter 찾기
        HandlerAdapter ha = getHandlerAdapter(mappedHandler.getHandler());

        // (생략) 인터셉터 전처리 (preHandle)

        // 3. HandlerAdapter를 통해 핸들러 실행
        // HandlerAdapter가 @PathVariable, @RequestParam, @RequestBody 등을 처리하고 메소드를 호출함
        mv = ha.handle(processedRequest, response, mappedHandler.getHandler());

        // (생략) 인터셉터 후처리 (postHandle)

    } catch (Exception ex) {
        dispatchException = ex;
    } catch (Throwable err) {
        // ...
    }
    // 4. 결과 처리 (뷰 렌더링 또는 예외 처리)
    processDispatchResult(processedRequest, response, mappedHandler, mv, dispatchException);
}

```

**2. `RequestMappingHandlerAdapter`의 `handle` 메소드 (매우 간략화된 개념 코드)**

```java
// RequestMappingHandlerAdapter.java (매우 간략화된 개념 코드)
public ModelAndView handle(HttpServletRequest request, HttpServletResponse response, Object handler) throws Exception {
    // handler는 실제로는 HandlerMethod 객체 (컨트롤러 객체 + 메소드 정보)
    HandlerMethod handlerMethod = (HandlerMethod) handler;

    // 1. 메소드 인자 준비 (Argument Resolvers 사용)
    Object[] args = resolveHandlerArguments(handlerMethod, request, response, ...);

    // 2. 실제 컨트롤러 메소드 호출
    Object returnValue = invokeHandlerMethod(handlerMethod, args);

    // 3. 반환 값 처리 (Return Value Handlers 사용)
    ModelAndView mav = processReturnValue(returnValue, handlerMethod, request, response, ...);

    return mav;
}

// 내부적으로 사용될 수 있는 메소드들 (개념)
private Object[] resolveHandlerArguments(HandlerMethod handlerMethod, /*...*/) {
    // 각 파라미터에 대해 적절한 HandlerMethodArgumentResolver를 찾아 값 준비
    // 예: @PathVariable -> UrlPathHelper, AntPathMatcher 등으로 경로에서 값 추출
    // 예: @RequestParam -> request.getParameter() 등 사용
    // 예: @RequestBody -> HttpMessageConverter를 사용해 요청 본문을 객체로 변환
    // ...
    return resolvedArguments;
}

private Object invokeHandlerMethod(HandlerMethod handlerMethod, Object[] args) {
    // Reflection을 사용하여 실제 컨트롤러 객체의 메소드를 args와 함께 호출
    return handlerMethod.getMethod().invoke(handlerMethod.getBean(), args);
}

private ModelAndView processReturnValue(Object returnValue, HandlerMethod handlerMethod, /*...*/) {
    // 반환 값의 타입, 어노테이션 등을 보고 처리
    // 예: String 반환 -> new ModelAndView(returnValue)
    // 예: @ResponseBody -> HttpMessageConverter로 응답 본문에 직접 쓰도록 설정 (ModelAndView는 null일 수 있음)
    // ...
    return modelAndView;
}

```

**`DispatcherServlet`은 `ha.handle(...)` 한 줄만 호출하면 됩니다.** `@PathVariable`을 파싱하고, `@RequestParam`을 가져오고, JSON을 객체로 변환하고, 메소드를 호출하고, 그 결과를 다시 `ModelAndView`로 만들거나 HTTP 응답에 직접 쓰는 등의 **복잡한 작업은 모두 `RequestMappingHandlerAdapter`가 알아서 처리**해주는 것입니다.

만약 `HttpRequestHandler` 타입의 핸들러였다면, `DispatcherServlet`은 `HttpRequestHandlerAdapter`를 찾을 것이고, 이 어댑터는 단순히 `handler.handleRequest(request, response)`만 호출하고 `null` (이미 응답이 직접 처리되었으므로)을 반환할 것입니다.

이처럼 `HandlerAdapter`는 `DispatcherServlet`과 다양한 유형의 핸들러 사이에서 **"어떻게 이 핸들러를 실행할 것인가?"** 라는 문제를 해결해주는 매우 중요한 역할을 합니다. `DispatcherServlet`은 "누가 처리할 것인가?"(`HandlerMapping`)와 "어떻게 실행할 것인가?"(`HandlerAdapter`)만 알면 되는 것입니다.

---

### 1. 왜 나는 설정하지도 않았는데 내부에서 동작하고 있는가? (자동 설정의 마법)

이것이 바로 **Spring Boot의 자동 설정(Auto-Configuration)** 또는 Spring MVC의 **기본 설정(@EnableWebMvc)** 덕분입니다.

- **Spring Boot 사용 시:**
  - `pom.xml` (Maven)이나 `build.gradle` (Gradle) 파일에 `spring-boot-starter-web` 의존성을 추가하면, Spring Boot는 "아, 이 사용자는 웹 애플리케이션을 만들려나 보다!"라고 판단합니다.
  - 그리고 웹 애플리케이션에 필요한 수많은 기본 설정들을 **자동으로** 해줍니다. 여기에는 다음이 포함됩니다:
    - `DispatcherServlet` 등록 및 설정
    - **`RequestMappingHandlerMapping` 빈 등록** (여러분이 `@GetMapping` 등을 사용하면 URL과 메소드를 연결해주는 역할)
    - **`RequestMappingHandlerAdapter` 빈 등록** (여러분이 `@PathVariable`, `@RequestParam` 등을 사용하면 파라미터 값을 주입하고 메소드를 실행해주는 역할)
    - 기본적인 `ViewResolver` (예: `InternalResourceViewResolver`의 기본 설정)
    - `HttpMessageConverter` (JSON, XML 등을 처리하기 위한) 다수 등록
    - 기타 등등...
  - 이 모든 것이 "Convention over Configuration (설정보다 관례/규약)" 철학에 따라, 가장 일반적이고 많이 사용되는 방식으로 미리 설정되어 제공되는 것입니다. 개발자는 특별히 바꾸고 싶지 않으면 그냥 가져다 쓰면 됩니다.
- **Spring Boot 없이 순수 Spring MVC 사용 시 (예: `WebApplicationInitializer` 사용):**
  - 설정 클래스(예: `AppConfig.java`)에 `@EnableWebMvc` 어노테이션을 붙이면, Spring MVC는 웹 요청 처리에 필요한 핵심 빈들을 **기본값으로 등록**해줍니다.
  - `@EnableWebMvc`가 바로 `RequestMappingHandlerMapping`, `RequestMappingHandlerAdapter` 등을 포함한 여러 필수 빈들을 자동으로 등록해주는 역할을 합니다.

**결론:** 학생님이 직접 `HandlerMapping`이나 `HandlerAdapter` 같은 빈들을 XML이나 Java 코드로 명시적으로 "이거 써!"라고 등록하지 않았더라도, Spring Boot나 `@EnableWebMvc`가 **"대부분의 경우 이게 필요할 테니, 내가 알아서 기본 좋은 녀석들로 준비해둘게!"** 라고 해서 이미 내부적으로 존재하고 동작하고 있는 것입니다. 그래서 Controller, Service, DAO만 작성해도 웹 애플리케이션이 잘 돌아가는 것이죠.

### 2. 이 기본 `HandlerMapping`이나 `HandlerAdapter`를 언제, 왜 바꿔야 할까?

대부분의 일반적인 웹 애플리케이션에서는 Spring이 제공하는 기본 `RequestMappingHandlerMapping`과 `RequestMappingHandlerAdapter`만으로도 충분합니다. 하지만 다음과 같은 특별한 요구사항이 있을 때 커스터마이징하거나 교체하는 것을 고려할 수 있습니다:

- **`HandlerMapping`을 바꾸는 경우:**
  - **매우 특수한 URL 매핑 전략이 필요한 경우:** 예를 들어, URL 패턴이 아니라 요청 헤더의 특정 값, 또는 특정 IP 대역에서 오는 요청에 따라 다른 컨트롤러를 매핑하고 싶을 때. `@RequestMapping`으로는 표현하기 어려운 복잡한 매핑 규칙이 필요할 때.
  - **레거시 시스템과의 통합:** 기존 시스템이 사용하는 독특한 핸들러 결정 방식을 Spring MVC에 통합해야 할 때.
  - **커스텀 어노테이션 기반 매핑:** `@RequestMapping` 외에 자신만의 커스텀 어노테이션을 만들고, 그 어노테이션을 기준으로 핸들러를 매핑하고 싶을 때 (매우 고급 활용 사례).
  - **성능 최적화를 위한 특수 매핑:** 극단적인 성능 요구사항 하에서 특정 매핑 알고리즘을 사용하고 싶을 때 (매우 드문 경우).
- **`HandlerAdapter`를 바꾸는 경우:**
  - **표준적이지 않은 핸들러(컨트롤러) 타입을 지원해야 할 때:** `@Controller` 어노테이션 방식이나 `HttpRequestHandler` 인터페이스 방식이 아닌, 아주 독특한 형태의 핸들러 객체를 만들어서 요청을 처리하게 하고 싶을 때. 예를 들어, 특정 DSL(Domain Specific Language)로 작성된 스크립트를 실행하여 요청을 처리하는 핸들러를 지원하고 싶다면, 그 스크립트를 해석하고 실행할 수 있는 커스텀 `HandlerAdapter`가 필요합니다.
  - **핸들러 메소드 호출 전후로 매우 특수한 로직을 추가하고 싶을 때:** 물론 인터셉터(Interceptor)나 AOP(Aspect-Oriented Programming)로도 많은 것을 할 수 있지만, 핸들러 호출 방식 자체를 근본적으로 변경해야 한다면 `HandlerAdapter`를 커스터마이징할 수 있습니다. (예: 특정 타입의 파라미터에 대해 일괄적인 전처리 로직을 수행하거나, 반환 값을 특별한 방식으로 가공하는 등)
  - **다른 프레임워크의 요청 처리 모델을 Spring MVC 내에 통합할 때:** (매우 드문 경우)

**일반적으로, Spring MVC가 제공하는 확장 포인트(예: `HandlerMethodArgumentResolver` 추가, `HttpMessageConverter` 추가, `Interceptor` 사용 등)를 활용하면 대부분의 요구사항을 해결할 수 있으므로, `HandlerMapping`이나 `HandlerAdapter` 자체를 완전히 교체하는 경우는 흔하지 않습니다.**

### 3. 진짜 바꿔야 한다면 어떻게 해야 할까? (개념적 접근)

만약 정말로 기본 `HandlerMapping`이나 `HandlerAdapter`를 커스터마이징하거나 교체해야 한다면, 다음과 같은 방식으로 접근할 수 있습니다 (구체적인 코드는 상황에 따라 매우 달라집니다):

**방법 1: 기존 빈의 프로퍼티 수정 (제한적)**

일부 `HandlerMapping`이나 `HandlerAdapter` 구현체는 설정 가능한 프로퍼티를 제공합니다. Spring 설정 파일(Java Config 또는 XML)에서 해당 빈을 가져와 프로퍼티를 수정할 수 있습니다. 하지만 이는 제공되는 프로퍼티 내에서만 커스터마이징이 가능합니다.

**방법 2: 새로운 구현체 만들고 등록하기 (가장 일반적인 교체/확장 방식)**

1. **인터페이스 구현:**
  - `org.springframework.web.servlet.HandlerMapping` 인터페이스 또는 `org.springframework.web.servlet.HandlerAdapter` 인터페이스를 구현하는 새로운 클래스를 만듭니다.
  - 또는 기존 구현체(예: `RequestMappingHandlerMapping`)를 상속받아 필요한 부분만 오버라이드할 수도 있습니다.
2. **새로운 빈 등록:**
  - **Java Config:** `@Configuration` 클래스 내에 `@Bean` 메소드를 사용하여 직접 만든 `HandlerMapping` 또는 `HandlerAdapter` 구현체를 빈으로 등록합니다.

      ```java
      @Configuration
      @EnableWebMvc // 기본 MVC 설정을 가져오되, 우리가 정의한 빈으로 일부를 대체/추가
      public class MyWebMvcConfig implements WebMvcConfigurer {
      
          @Bean
          public HandlerMapping customHandlerMapping() {
              // MyCustomHandlerMapping은 우리가 직접 만든 HandlerMapping 구현체
              MyCustomHandlerMapping mapping = new MyCustomHandlerMapping();
              // 필요한 설정 (예: 우선순위 설정 등)
              mapping.setOrder(0); // 다른 매핑보다 먼저 시도하도록
              return mapping;
          }
      
          @Bean
          public HandlerAdapter customHandlerAdapter() {
              // MyCustomHandlerAdapter는 우리가 직접 만든 HandlerAdapter 구현체
              return new MyCustomHandlerAdapter();
          }
      
          // 만약 기존 RequestMappingHandlerMapping, RequestMappingHandlerAdapter를
          // 완전히 비활성화하고 우리 것만 쓰고 싶다면 @EnableWebMvc를 사용하지 않거나
          // 더 복잡한 설정이 필요할 수 있습니다.
          // 보통은 기존 것과 함께 사용하거나, 우선순위를 조절하여 우리 것이 먼저 선택되도록 합니다.
      }
      
      ```

  - **XML Config:** `<bean>` 태그를 사용하여 등록합니다.
3. **우선순위 설정 (중요):**
   Spring MVC는 여러 개의 `HandlerMapping`이나 `HandlerAdapter`가 등록되어 있을 수 있습니다. 이때 어떤 것을 먼저 사용할지는 `Ordered` 인터페이스를 구현하거나 `@Order` 어노테이션을 사용하여 우선순위를 지정함으로써 결정됩니다. 숫자가 낮을수록 우선순위가 높습니다. 직접 만든 빈이 기본 빈보다 먼저 사용되게 하려면 더 낮은 숫자로 우선순위를 설정해야 합니다.

**주의사항:**

- `HandlerMapping`이나 `HandlerAdapter`를 직접 만드는 것은 Spring MVC의 내부 동작에 대한 깊은 이해를 필요로 하는 고급 작업입니다.
- 대부분의 경우, `@Controller`, `@Service`, `@Repository`를 사용하고, 필요에 따라 `Interceptor`, `HandlerMethodArgumentResolver`, `HttpMessageConverter` 등을 추가하는 방식으로 원하는 기능을 구현할 수 있습니다.
- 이러한 핵심 컴포넌트를 잘못 건드리면 애플리케이션 전체가 오동작할 수 있으므로 신중해야 합니다.

**결론적으로, 학생님이 Controller, Service, DAO만 작성해도 잘 동작했던 이유는 Spring (특히 Spring Boot)이 나머지를 "알아서 잘" 해주기 때문입니다.** 그리고 이 "알아서 잘" 해주는 기본 컴포넌트들을 직접 바꾸는 경우는 매우 드물며, 특별하고 복잡한 요구사항이 있을 때 고려하는 고급 주제입니다.
