---
title: Spring Web MVC - Dispatcher Servlet (Exceptions)
description: 
author: laze
date: 2025-05-19 00:00:02 +0900
categories: [Dev, SpringWebMVC]
tags: [SpringWebMVC]
---
**예외 (Exceptions)**

Reactive 스택에서의 동등한 내용을 확인하세요.

요청 매핑 중 예외가 발생하거나 요청 핸들러(예: `@Controller`)에서 예외가 던져지면, `DispatcherServlet`은 `HandlerExceptionResolver` 빈들의 체인에 위임하여 예외를 해결하고 대안적인 처리(일반적으로 에러 응답)를 제공합니다.

다음 표는 사용 가능한 `HandlerExceptionResolver` 구현체들을 나열합니다:

**`HandlerExceptionResolver` 구현체들**

| `HandlerExceptionResolver` | 설명 |
| --- | --- |
| `SimpleMappingExceptionResolver` | 예외 클래스 이름과 에러 뷰 이름 간의 매핑입니다. 브라우저 애플리케이션에서 에러 페이지를 렌더링하는 데 유용합니다. |
| `DefaultHandlerExceptionResolver` | Spring MVC에 의해 발생된 예외를 해결하고 HTTP 상태 코드로 매핑합니다. 대안적인 `ResponseEntityExceptionHandler` 및 에러 응답(Error Responses)도 참조하세요. |
| `ResponseStatusExceptionResolver` | `@ResponseStatus` 어노테이션이 있는 예외를 해결하고 어노테이션의 값에 따라 HTTP 상태 코드로 매핑합니다. |
| `ExceptionHandlerExceptionResolver` | `@Controller` 또는 `@ControllerAdvice` 클래스 내의 `@ExceptionHandler` 메소드를 호출하여 예외를 해결합니다. `@ExceptionHandler` 메소드(methods)를 참조하세요. |

**리졸버 체인 (Chain of Resolvers)**
Spring 설정에 여러 `HandlerExceptionResolver` 빈을 선언하고 필요에 따라 `order` 속성을 설정하여 예외 리졸버 체인을 구성할 수 있습니다.

`order` 속성 값이 높을수록 예외 리졸버는 나중에 위치합니다.

`HandlerExceptionResolver`의 계약은 다음을 반환할 수 있다고 명시합니다:

- 에러 뷰를 가리키는 `ModelAndView`.
- 예외가 리졸버 내에서 처리된 경우 빈 `ModelAndView`.
- 예외가 해결되지 않은 경우 `null`. 이 경우 후속 리졸버가 시도하며, 만약 예외가 끝까지 해결되지 않으면 서블릿 컨테이너로 버블 업(bubble up)되도록 허용됩니다.

MVC 설정(MVC Config)은 기본 Spring MVC 예외, `@ResponseStatus` 어노테이션이 달린 예외, 그리고 `@ExceptionHandler` 메소드 지원을 위한 내장 리졸버들을 자동으로 선언합니다.

이 목록을 사용자 정의하거나 교체할 수 있습니다.

**컨테이너 에러 페이지 (Container Error Page)**
만약 예외가 어떤 `HandlerExceptionResolver`에 의해서도 해결되지 않아 전파되도록 남겨지거나, 응답 상태가 에러 상태(즉, 4xx, 5xx)로 설정되면, 서블릿 컨테이너는 HTML 형식의 기본 에러 페이지를 렌더링할 수 있습니다.

컨테이너의 기본 에러 페이지를 사용자 정의하려면, `web.xml`에 에러 페이지 매핑을 선언할 수 있습니다. 다음 예제는 그 방법을 보여줍니다:

```xml
<error-page>
	<location>/error</location>
</error-page>
```

위 예제를 고려할 때, 예외가 버블 업되거나 응답에 에러 상태가 있으면, 서블릿 컨테이너는 설정된 URL(예: `/error`)로 컨테이너 내에서 ERROR 디스패치를 수행합니다.

이는 그런 다음 `DispatcherServlet`에 의해 처리되며, 다음 예제와 같이 모델과 함께 에러 뷰 이름을 반환하거나 JSON 응답을 렌더링하도록 구현될 수 있는 `@Controller`에 매핑될 수 있습니다:

**Java**

```java
@RestController
public class ErrorController {

	@RequestMapping(path = "/error")
	public Map<String, Object> handle(HttpServletRequest request) {
		Map<String, Object> map = new HashMap<>();
		map.put("status", request.getAttribute("jakarta.servlet.error.status_code"));
		map.put("reason", request.getAttribute("jakarta.servlet.error.message"));
		return map;
	}
}
```

서블릿 API는 Java에서 에러 페이지 매핑을 생성하는 방법을 제공하지 않습니다.

그러나 `WebApplicationInitializer`와 최소한의 `web.xml`을 모두 사용할 수 있습니다.

---

## Spring MVC의 예외 처리 전략: HandlerExceptionResolver

Spring MVC 애플리케이션에서 요청 처리 중 (예: 컨트롤러 메소드 실행 중, 또는 그 이전 요청 매핑 과정 중) 예외가 발생하면, `DispatcherServlet`은 이 예외를 그냥 방치하지 않습니다.

대신, 등록된 **`HandlerExceptionResolver` 구현체들에게 예외를 처리할 기회**를 줍니다.

`HandlerExceptionResolver`는 발생한 예외를 보고, 적절한 에러 응답(예: 특정 에러 페이지를 보여주거나, HTTP 상태 코드를 설정하는 등)을 생성하여 사용자에게 보다 친절한 방식으로 오류를 알리는 역할을 합니다.

### 1. `HandlerExceptionResolver` 인터페이스

- **핵심 메소드:** `resolveException(HttpServletRequest request, HttpServletResponse response, Object handler, Exception ex)`
  - 이 메소드는 발생한 예외(`ex`)와 현재 요청/응답 정보, 그리고 예외가 발생한 핸들러(`handler`) 정보를 받아 예외를 처리합니다.
  - **반환 값의 의미:**
    - **`ModelAndView` 객체:** 예외를 특정 에러 뷰로 렌더링하도록 지시합니다. `ModelAndView`에는 뷰 이름과 모델 데이터가 포함될 수 있습니다.
    - **빈 `ModelAndView` 객체 (예: `new ModelAndView()`):** 예외가 이 리졸버 내에서 완전히 처리되었고, 더 이상 다른 처리가 필요 없음을 의미합니다. (예: 응답 스트림에 직접 에러 메시지를 작성한 경우)
    - **`null`:** 이 리졸버는 해당 예외를 처리할 수 없음을 의미합니다. `DispatcherServlet`은 다음 `HandlerExceptionResolver`에게 처리를 위임하거나, 모든 리졸버가 `null`을 반환하면 예외는 서블릿 컨테이너로 전파됩니다.

### 2. 주요 `HandlerExceptionResolver` 구현체들

Spring MVC는 여러 유용한 `HandlerExceptionResolver` 기본 구현체들을 제공하며, 이들은 보통 MVC Config에 의해 자동으로 등록됩니다.

| 구현체 | 주요 역할 및 특징 |
| --- | --- |
| `ExceptionHandlerExceptionResolver` | **(가장 중요하고 많이 사용됨)** `@Controller` 또는 `@ControllerAdvice` 클래스 내에 정의된 **`@ExceptionHandler` 어노테이션이 붙은 메소드**를 찾아 호출하여 예외를 처리합니다. 이를 통해 개발자는 특정 예외 타입별로 매우 유연하게 예외 처리 로직을 작성할 수 있습니다. |
| `ResponseStatusExceptionResolver` | 예외 클래스에 **`@ResponseStatus` 어노테이션**이 붙어 있는 경우, 해당 어노테이션에 지정된 HTTP 상태 코드를 응답으로 설정합니다. 간단한 예외-상태 코드 매핑에 유용합니다. |
| `DefaultHandlerExceptionResolver` | Spring MVC 내부에서 발생하는 표준적인 예외들 (예: `HttpRequestMethodNotSupportedException`, `TypeMismatchException` 등)을 미리 정의된 HTTP 상태 코드 (예: 405, 400)로 변환합니다. 일종의 기본 방어선 역할을 합니다. |
| `SimpleMappingExceptionResolver` | **(과거에 많이 사용, 현재는 `@ExceptionHandler` 방식이 더 선호됨)** 예외 클래스 이름과 뷰 이름(에러 페이지)을 매핑하는 간단한 방법을 제공합니다. `exceptionMappings` 프로퍼티에 `java.lang.RuntimeException=error/runtime` 과 같이 설정합니다. |

**자동 등록:** `@EnableWebMvc` 또는 Spring Boot의 자동 설정을 사용하면, `ExceptionHandlerExceptionResolver`, `ResponseStatusExceptionResolver`, `DefaultHandlerExceptionResolver` 등이 **자동으로 등록되고 적절한 우선순위로 설정**됩니다. 개발자는 특별한 경우가 아니면 이들의 등록을 신경 쓸 필요가 없습니다.

### 3. 리졸버 체인 (Chain of Resolvers)

애플리케이션에는 여러 개의 `HandlerExceptionResolver` 빈이 등록될 수 있습니다. 이때 이들은 **체인(chain)** 형태로 동작합니다.

- **실행 순서:** `DispatcherServlet`은 등록된 리졸버들을 `order` 속성 값에 따라 순서대로 호출합니다. `order` 값이 낮을수록 먼저 실행됩니다. (만약 `order`가 명시되지 않으면, Spring의 일반적인 빈 정렬 규칙을 따릅니다.)
- **처리 방식:**
  1. 첫 번째 리졸버의 `resolveException()` 메소드가 호출됩니다.
  2. 만약 이 메소드가 `ModelAndView` 객체(빈 객체 포함)를 반환하면, 예외 처리는 거기서 종료되고 그 결과가 사용됩니다.
  3. 만약 `null`을 반환하면, `DispatcherServlet`은 다음 순서의 리졸버에게 예외 처리를 넘깁니다.
  4. 모든 리졸버가 `null`을 반환하면, 예외는 처리되지 않은 것으로 간주하고 서블릿 컨테이너로 전파됩니다.

### 4. `@ExceptionHandler` 메소드 (가장 일반적인 예외 처리 방식)

`ExceptionHandlerExceptionResolver`에 의해 지원되는 `@ExceptionHandler` 어노테이션은 컨트롤러 내부 또는 `@ControllerAdvice` 클래스 내에서 특정 예외 타입을 처리하는 메소드를 정의하는 매우 강력하고 유연한 방법입니다.

**예시 (`@ControllerAdvice` 사용):**

```java
import org.slf4j.Logger;
import org.slf4j.LoggerFactory;
import org.springframework.http.HttpStatus;
import org.springframework.http.ResponseEntity;
import org.springframework.web.bind.annotation.ControllerAdvice;
import org.springframework.web.bind.annotation.ExceptionHandler;
import org.springframework.web.bind.annotation.ResponseStatus;
import org.springframework.web.servlet.ModelAndView;

// 모든 컨트롤러에서 발생하는 예외를 전역적으로 처리
@ControllerAdvice
public class GlobalExceptionHandler {

    private static final Logger logger = LoggerFactory.getLogger(GlobalExceptionHandler.class);

    // 특정 예외 타입 (CustomBusinessException) 처리
    @ExceptionHandler(CustomBusinessException.class)
    public ModelAndView handleCustomBusinessException(CustomBusinessException ex) {
        logger.error("Custom Business Exception Occurred: {}", ex.getMessage());
        ModelAndView mav = new ModelAndView("error/businessError"); // error/businessError.jsp 또는 Thymeleaf 템플릿
        mav.addObject("errorMessage", ex.getMessage());
        mav.addObject("errorCode", ex.getErrorCode());
        return mav;
    }

    // 일반적인 RuntimeException 처리 (REST API 스타일 - JSON 응답)
    @ExceptionHandler(RuntimeException.class)
    @ResponseStatus(HttpStatus.INTERNAL_SERVER_ERROR) // 응답 상태 코드 설정 (이 어노테이션이 없으면 ResponseEntity로 상태 설정)
    public ResponseEntity<ErrorResponse> handleRuntimeException(RuntimeException ex) {
        logger.error("Runtime Exception Occurred: ", ex);
        ErrorResponse errorResponse = new ErrorResponse(HttpStatus.INTERNAL_SERVER_ERROR.value(), "서버 내부 오류가 발생했습니다.");
        return new ResponseEntity<>(errorResponse, HttpStatus.INTERNAL_SERVER_ERROR);
    }

    // 특정 예외에 대해 HTTP 상태 코드만 반환 (예: 리소스 없음)
    @ExceptionHandler(ResourceNotFoundException.class)
    @ResponseStatus(value = HttpStatus.NOT_FOUND, reason = "요청하신 리소스를 찾을 수 없습니다.")
    public void handleResourceNotFoundException(ResourceNotFoundException ex) {
        logger.warn("Resource not found: {}", ex.getMessage());
        // @ResponseStatus 어노테이션이 상태 코드와 reason을 설정해주므로 메소드 본문은 비워도 됨
    }
}

// ErrorResponse.java (간단한 DTO)
class ErrorResponse {
    private int status;
    private String message;
    // 생성자, getter, setter
    public ErrorResponse(int status, String message) { this.status = status; this.message = message; }
    public int getStatus() { return status; }
    public String getMessage() { return message; }
}

// CustomBusinessException.java (사용자 정의 예외)
class CustomBusinessException extends RuntimeException {
    private String errorCode;
    public CustomBusinessException(String message, String errorCode) { super(message); this.errorCode = errorCode; }
    public String getErrorCode() { return errorCode; }
}

// ResourceNotFoundException.java (사용자 정의 예외)
class ResourceNotFoundException extends RuntimeException {
    public ResourceNotFoundException(String message) { super(message); }
}

```

- `@ControllerAdvice`: 이 클래스가 여러 컨트롤러에 걸쳐 적용될 전역적인 예외 처리기, 데이터 바인더 등을 포함함을 나타냅니다.
- `@ExceptionHandler(ExceptionType.class)`: 어떤 타입의 예외를 이 메소드가 처리할지 지정합니다. 여러 타입을 배열로 지정할 수도 있습니다 (예: `@ExceptionHandler({IOException.class, SQLException.class})`).
- `@ExceptionHandler` 메소드는 `ModelAndView`를 반환하여 특정 뷰를 렌더링하거나, `@ResponseBody` (또는 `@RestControllerAdvice` 사용 시 자동으로)와 함께 사용하여 응답 본문을 직접 생성하거나, `ResponseEntity`를 반환하여 HTTP 상태 코드와 헤더, 본문을 모두 제어할 수 있습니다. `@ResponseStatus`를 사용하여 간단히 상태 코드만 지정할 수도 있습니다.

### 5. 서블릿 컨테이너의 기본 에러 페이지 처리

만약 `HandlerExceptionResolver` 체인에서 예외가 전혀 처리되지 않고 서블릿 컨테이너까지 전파되거나, 응답 상태 코드가 4xx 또는 5xx 에러로 설정된 경우 (예: `response.sendError(404)`) 서블릿 컨테이너는 자신이 가지고 있는 기본 에러 페이지를 보여주려고 시도합니다.

이 컨테이너의 기본 에러 페이지를 사용자 정의하고 싶다면, `web.xml`에 다음과 같이 `<error-page>` 매핑을 선언할 수 있습니다:

```xml
<!-- web.xml -->
<error-page>
    <error-code>404</error-code> <!-- 특정 HTTP 에러 코드에 대해 -->
    <location>/WEB-INF/views/error/404.jsp</location> <!-- 이 페이지를 보여줘라 -->
</error-page>

<error-page>
    <exception-type>java.lang.Throwable</exception-type> <!-- 특정 예외 타입에 대해 -->
    <location>/error</location> <!-- /error URL로 다시 요청을 보내라 (ERROR 디스패치) -->
</error-page>

<error-page>
    <location>/error</location> <!-- 모든 처리되지 않은 에러 (기본값) -->
</error-page>
```

- 만약 `<location>`이 `/error`와 같이 특정 URL로 지정되면, 서블릿 컨테이너는 해당 URL로 **ERROR 디스패치**라는 특수한 내부 재요청을 보냅니다.
- 이 `/error` 요청은 다시 `DispatcherServlet`에 의해 처리될 수 있으며, 여기에 매핑된 컨트롤러(예: 원문의 `ErrorController` 예시)가 HTTP 상태 코드, 에러 메시지 등의 정보를 `HttpServletRequest` 속성(`jakarta.servlet.error.status_code`, `jakarta.servlet.error.message` 등)으로부터 가져와 사용자 정의된 에러 페이지(HTML)나 JSON 응답을 생성할 수 있습니다.

**Java Config에서의 컨테이너 에러 페이지:**

서블릿 API는 Java 코드로 `<error-page>` 매핑을 직접 생성하는 표준 방법을 제공하지 않습니다. 따라서 이 기능이 필요하다면:

- Spring Boot에서는 `ErrorPageRegistrar` 인터페이스나 `WebServerFactoryCustomizer`를 사용하여 프로그래밍 방식으로 에러 페이지를 등록할 수 있습니다.
- 순수 Spring MVC 환경에서는 `WebApplicationInitializer`와 함께 최소한의 `web.xml`을 사용하여 `<error-page>` 태그를 정의하는 방식을 사용할 수 있습니다.

**결론적으로 Spring MVC의 예외 처리 흐름은 다음과 같습니다:**

1. 예외 발생.
2. `DispatcherServlet`이 등록된 `HandlerExceptionResolver` 체인을 순서대로 호출.
3. `@ExceptionHandler` 메소드 (주로 `ExceptionHandlerExceptionResolver`에 의해) 또는 다른 리졸버가 예외를 처리하여 `ModelAndView`를 반환하거나 응답을 직접 생성.
4. 만약 모든 리졸버가 예외를 처리하지 못하면(`null` 반환), 예외는 서블릿 컨테이너로 전파.
5. 서블릿 컨테이너는 `web.xml` (또는 Spring Boot 설정)의 `<error-page>` 매핑에 따라 기본 에러 페이지를 보여주거나, 특정 URL로 ERROR 디스패치를 수행.

이러한 다층적인 예외 처리 메커니즘을 통해 Spring MVC는 개발자가 다양한 상황에 맞는 정교한 예외 처리 전략을 구현할 수 있도록 지원합니다.

---
