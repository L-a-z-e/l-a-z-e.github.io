---
title: Spring Web MVC - Exception
description: 
author: laze
date: 2025-06-19 00:00:01 +0900
categories: [Dev, SpringWebMVC]
tags: [SpringWebMVC]
---
### Exceptions (예외)

`@Controller`와 `@ControllerAdvice` 클래스는 다음 예제와 같이 컨트롤러 메소드에서 발생하는 예외를 처리하기 위한 `@ExceptionHandler` 메소드를 가질 수 있습니다:

**Java**

```java
import java.io.IOException;

import org.springframework.http.ResponseEntity;
import org.springframework.stereotype.Controller;
import org.springframework.web.bind.annotation.ExceptionHandler;

@Controller
public class SimpleController {

	// IOException 또는 그 하위 타입의 예외가 발생하면 이 메소드가 처리
	@ExceptionHandler(IOException.class)
	public ResponseEntity<String> handle() {
		// 내부 서버 오류 응답 생성
		return ResponseEntity.internalServerError().body("Could not read file storage");
	}
}
```

### Exception Mapping (예외 매핑)

예외는 전파되는 최상위 예외(예: 직접 발생한 `IOException`)와 매칭되거나, 래퍼 예외(wrapper exception) 내의 중첩된 원인(nested cause)과 매칭될 수 있습니다(예: `IllegalStateException` 내에 래핑된 `IOException`). 5.3 버전부터는 임의의 원인 레벨에서 매칭될 수 있으며, 이전에는 즉각적인 원인만 고려되었습니다.

예외 타입을 매칭할 때는, 위 예제와 같이 대상 예외를 메소드 인자로 선언하는 것이 좋습니다.

여러 예외 처리 메소드가 매칭될 경우, 일반적으로 원인 예외 매칭보다 루트 예외 매칭이 우선시됩니다.

더 구체적으로, `ExceptionDepthComparator`는 발생한 예외 타입으로부터의 깊이를 기준으로 예외를 정렬하는 데 사용됩니다.

또는, 다음 예제와 같이 어노테이션 선언에서 매칭할 예외 타입을 좁힐 수 있습니다:

**Java**

```java
// FileSystemException 또는 RemoteException 발생 시 이 메소드가 처리
@ExceptionHandler({FileSystemException.class, RemoteException.class})
public ResponseEntity<String> handleIoException(IOException ex) { // IOException 또는 그 하위 타입으로 받음
	return ResponseEntity.internalServerError().body(ex.getMessage());
}
```

다음 예제와 같이 매우 일반적인 인자 시그니처와 함께 특정 예외 타입 목록을 사용할 수도 있습니다:

**Java**

```java
// FileSystemException 또는 RemoteException 발생 시 이 메소드가 처리
@ExceptionHandler({FileSystemException.class, RemoteException.class})
public ResponseEntity<String> handleExceptions(Exception ex) { // 모든 예외의 최상위 타입인 Exception으로 받음
	return ResponseEntity.internalServerError().body(ex.getMessage());
}
```

루트 예외와 원인 예외 매칭 간의 구분은 놀라울 수 있습니다.

앞서 보여드린 `IOException` 변형 예제에서, 메소드는 일반적으로 실제 `FileSystemException` 또는 `RemoteException` 인스턴스를 인자로 받아 호출됩니다.

왜냐하면 이 둘 모두 `IOException`에서 확장되기 때문입니다.

그러나 만약 이러한 매칭되는 예외가 그 자체가 `IOException`인 래퍼 예외 내에서 전파된다면, 전달되는 예외 인스턴스는 해당 래퍼 예외가 됩니다.

`handle(Exception)` 변형 예제에서는 동작이 훨씬 간단합니다.

래핑 시나리오에서는 항상 래퍼 예외와 함께 호출되며, 이 경우 실제로 매칭되는 예외는 `ex.getCause()`를 통해 찾아야 합니다.

전달되는 예외는 이러한 예외들이 최상위 예외로 발생했을 때만 실제 `FileSystemException` 또는 `RemoteException` 인스턴스가 됩니다.

일반적으로 인자 시그니처에서 가능한 한 구체적으로 명시하여, 루트 예외 타입과 원인 예외 타입 간의 불일치 가능성을 줄이는 것을 권장합니다.

여러 예외를 매칭하는 메소드를 시그니처를 통해 단일 특정 예외 타입을 매칭하는 개별 `@ExceptionHandler` 메소드로 나누는 것을 고려하십시오.

여러 `@ControllerAdvice`가 배열된 경우, 주요 루트 예외 매핑을 해당 순서로 우선순위가 지정된 `@ControllerAdvice`에 선언하는 것이 좋습니다.

루트 예외 매치가 원인 매치보다 선호되지만, 이는 주어진 컨트롤러 또는 `@ControllerAdvice` 클래스의 메소드들 사이에서 정의됩니다.

즉, 우선순위가 높은 `@ControllerAdvice` 빈의 원인 매치가 우선순위가 낮은 `@ControllerAdvice` 빈의 어떤 매치(예: 루트)보다 선호됩니다.

마지막으로, `@ExceptionHandler` 메소드 구현은 주어진 예외 인스턴스 처리를 포기하고 원래 형태로 다시 던질(rethrow) 수 있습니다.

이는 루트 레벨 매치에만 관심이 있거나 정적으로 결정할 수 없는 특정 컨텍스트 내의 매치에만 관심이 있는 시나리오에서 유용합니다.

다시 던져진 예외는 마치 해당 `@ExceptionHandler` 메소드가 처음부터 매치되지 않은 것처럼 나머지 해결 체인(resolution chain)을 통해 전파됩니다.

스프링 MVC에서 `@ExceptionHandler` 메소드 지원은 `DispatcherServlet` 레벨의 `HandlerExceptionResolver` 메커니즘을 기반으로 구축됩니다.

### Media Type Mapping (미디어 타입 매핑)

예외 타입 외에도, `@ExceptionHandler` 메소드는 생성 가능한 미디어 타입(producible media types)을 선언할 수 있습니다.

이를 통해 HTTP 클라이언트가 요청한 미디어 타입(일반적으로 HTTP 요청 헤더의 "Accept" 헤더)에 따라 오류 응답을 세분화할 수 있습니다.

애플리케이션은 동일한 예외 타입에 대해 어노테이션에 직접 생성 가능한 미디어 타입을 선언할 수 있습니다:

**Java**

```java
// IllegalArgumentException 발생 시 클라이언트가 "application/json"을 요청하면 이 메소드 호출
@ExceptionHandler(produces = "application/json")
public ResponseEntity<ErrorMessage> handleJson(IllegalArgumentException exc) {
	return ResponseEntity.badRequest().body(new ErrorMessage(exc.getMessage(), 42));
}

// IllegalArgumentException 발생 시 클라이언트가 "text/html"을 요청(또는 기본값)하면 이 메소드 호출
@ExceptionHandler(produces = "text/html")
public String handle(IllegalArgumentException exc, Model model) {
	model.addAttribute("error", new ErrorMessage(exc.getMessage(), 42));
	return "errorView"; // HTML 오류 페이지로 이동
}
```

여기서 메소드들은 동일한 예외 타입을 처리하지만 중복으로 거부되지 않습니다.

대신, "application/json"을 요청하는 API 클라이언트는 JSON 오류를 받고, 브라우저는 HTML 오류 뷰를 받게 됩니다.

각 `@ExceptionHandler` 어노테이션은 여러 생성 가능한 미디어 타입을 선언할 수 있으며, 오류 처리 단계에서의 내용 협상(content negotiation)을 통해 어떤 컨텐츠 타입이 사용될지 결정됩니다.

### Method Arguments (메소드 인자)

`@ExceptionHandler` 메소드는 다음 인자들을 지원합니다:

| 메소드 인자 | 설명 |
| --- | --- |
| `Exception` 타입 | 발생한 예외에 접근하기 위함. |
| `HandlerMethod` | 예외를 발생시킨 컨트롤러 메소드에 접근하기 위함. |
| `WebRequest`, `NativeWebRequest` | 서블릿 API를 직접 사용하지 않고 요청 파라미터, 요청 및 세션 속성에 일반적인 방식으로 접근하기 위함. |
| `jakarta.servlet.ServletRequest`, `jakarta.servlet.ServletResponse` | 특정 요청 또는 응답 타입을 선택 (예: `ServletRequest` 또는 `HttpServletRequest` 또는 스프링의 `MultipartRequest` 또는 `MultipartHttpServletRequest`). |
| `jakarta.servlet.http.HttpSession` | 세션의 존재를 강제. 결과적으로, 이러한 인자는 절대 `null`이 아님.<br/>세션 접근은 스레드 안전하지 않음에 유의. 여러 요청이 동시에 세션에 접근하도록 허용된 경우 `RequestMappingHandlerAdapter` 인스턴스의 `synchronizeOnSession` 플래그를 `true`로 설정하는 것을 고려. |
| `java.security.Principal` | 현재 인증된 사용자 – 알려진 경우 특정 `Principal` 구현 클래스일 수 있음. |
| `HttpMethod` | 요청의 HTTP 메소드. |
| `java.util.Locale` | 현재 요청 로케일, 사용 가능한 가장 구체적인 `LocaleResolver`에 의해 결정됨 – 실제로는 구성된 `LocaleResolver` 또는 `LocaleContextResolver`. |
| `java.util.TimeZone`, `java.time.ZoneId` | `LocaleContextResolver`에 의해 결정된 현재 요청과 관련된 시간대. |
| `java.io.OutputStream`, `java.io.Writer` | 서블릿 API에 의해 노출된 원시 응답 본문에 접근하기 위함. |
| `java.util.Map`, `org.springframework.ui.Model`, `org.springframework.ui.ModelMap` | 오류 응답을 위한 모델에 접근하기 위함. 항상 비어있음. |
| `RedirectAttributes` | 리다이렉트 시 사용할 속성(쿼리 문자열에 추가될 것)과 리다이렉트 후 요청까지 임시 저장될 플래시 속성을 지정. Redirect Attributes 및 Flash Attributes 참고. |
| `@SessionAttribute` | 클래스 레벨 `@SessionAttributes` 선언의 결과로 세션에 저장된 모델 속성과는 대조적으로, 모든 세션 속성에 접근하기 위함. 자세한 내용은 @SessionAttribute 참고. |
| `@RequestAttribute` | 요청 속성에 접근하기 위함. 자세한 내용은 @RequestAttribute 참고. |

### Return Values (반환 값)

`@ExceptionHandler` 메소드는 다음 반환 값들을 지원합니다:

| 반환 값 | 설명 |
| --- | --- |
| `@ResponseBody` | 반환 값은 `HttpMessageConverter` 인스턴스를 통해 변환되어 응답에 쓰여짐. @ResponseBody 참고. |
| `HttpEntity<B>`, `ResponseEntity<B>` | 반환 값은 전체 응답(HTTP 헤더 및 본문 포함)이 `HttpMessageConverter` 인스턴스를 통해 변환되어 응답에 쓰여지도록 지정. ResponseEntity 참고. |
| `ErrorResponse`, `ProblemDetail` | 본문에 상세 내용을 포함하는 RFC 9457 오류 응답을 렌더링하기 위함. Error Responses 참고. |
| `String` | `ViewResolver` 구현체로 해석될 뷰 이름이며, 암묵적 모델(커맨드 객체 및 `@ModelAttribute` 메소드를 통해 결정됨)과 함께 사용됨. 핸들러 메소드는 (앞서 설명한) `Model` 인자를 선언하여 프로그래밍 방식으로 모델을 보강할 수 있음. |
| `View` | 렌더링에 사용할 `View` 인스턴스이며, 암묵적 모델(커맨드 객체 및 `@ModelAttribute` 메소드를 통해 결정됨)과 함께 사용됨. 핸들러 메소드는 (앞서 설명한) `Model` 인자를 선언하여 프로그래밍 방식으로 모델을 보강할 수 있음. |
| `java.util.Map`, `org.springframework.ui.Model` | 암묵적 모델에 추가될 속성들이며, 뷰 이름은 `RequestToViewNameTranslator`를 통해 암묵적으로 결정됨. |
| `@ModelAttribute` | 모델에 추가될 속성이며, 뷰 이름은 `RequestToViewNameTranslator`를 통해 암묵적으로 결정됨.<br/>`@ModelAttribute`는 선택 사항임에 유의. 이 표의 끝에 있는 "기타 모든 반환 값" 참고. |
| `ModelAndView` 객체 | 사용할 뷰와 모델 속성, 그리고 선택적으로 응답 상태. |
| `void` | `void` 반환 타입(또는 `null` 반환 값)을 가진 메소드는 `ServletResponse`, `OutputStream` 인자를 가지거나 `@ResponseStatus` 어노테이션이 있는 경우 응답을 완전히 처리한 것으로 간주됨. 컨트롤러가 ETag 또는 lastModified 타임스탬프 검사를 성공적으로 수행한 경우에도 마찬가지임 (자세한 내용은 Controllers 참고).<br/>위 중 어느 것도 해당되지 않는 경우, `void` 반환 타입은 REST 컨트롤러의 경우 "응답 본문 없음"을 나타내거나 HTML 컨트롤러의 경우 기본 뷰 이름 선택을 나타낼 수 있음. |
| 기타 모든 반환 값 | 반환 값이 위 중 어느 것과도 일치하지 않고 단순 타입(`BeanUtils#isSimpleProperty`에 의해 결정됨)이 아닌 경우, 기본적으로 모델에 추가될 모델 속성으로 처리됨. 단순 타입인 경우 해결되지 않은 상태로 남음. |

---

### **학습 목표 제시**

이 챕터를 학습하고 나면, 학생은 다음을 이해하고 수행할 수 있게 됩니다:

1. **`@ExceptionHandler` 어노테이션의 역할과 사용법 숙지:** 특정 컨트롤러 내에서 발생하는 예외를 처리하기 위해 `@ExceptionHandler` 메소드를 어떻게 정의하고 사용하는지, 그리고 어떤 예외 타입을 대상으로 매핑할 수 있는지 이해합니다.
2. **다양한 예외 매핑 전략 이해:** 단일 예외, 여러 예외, 루트 예외와 원인 예외 매칭의 차이점 등을 이해하고, 상황에 맞는 예외 매핑 방법을 선택할 수 있습니다. (컨트롤러 어드바이스는 다음 챕터에서 더 자세히)
3. **미디어 타입 기반 예외 처리 방법 인지:** 클라이언트의 `Accept` 헤더에 따라 동일한 예외에 대해 다른 형식(JSON, HTML 등)의 오류 응답을 제공하는 방법을 이해합니다.
4. **`@ExceptionHandler` 메소드의 유연한 인자 및 반환 타입 활용:** `@ExceptionHandler` 메소드가 다양한 요청 관련 정보(예외 객체, `WebRequest`, `Model` 등)를 인자로 받을 수 있고, 다양한 형태( `ResponseEntity`, `String` (뷰 이름), `@ResponseBody` 객체 등)로 응답을 생성할 수 있음을 이해하고 적용할 수 있습니다.

---

### **핵심 개념 설명**

**왜 예외 처리가 중요할까요?**

애플리케이션을 개발하다 보면 다양한 종류의 오류(예외)가 발생할 수 있습니다. 예를 들어:

- 데이터베이스 연결 실패
- 파일을 찾을 수 없음 (`FileNotFoundException`)
- 잘못된 사용자 입력으로 인한 데이터 변환 오류 (`NumberFormatException`)
- 권한 없는 작업 시도 (`SecurityException`)
- 프로그래밍 로직 오류 (`NullPointerException`, `ArrayIndexOutOfBoundsException` 등)

이러한 예외들이 제대로 처리되지 않고 사용자에게 그대로 노출되면, 사용자는 당황스러운 오류 화면을 보게 되고 서비스에 대한 신뢰를 잃을 수 있습니다. 또한, 시스템 내부적으로도 오류 상황을 제대로 기록하거나 복구하지 못하면 더 큰 문제로 이어질 수 있습니다.

**스프링 MVC의 `@ExceptionHandler`**

스프링 MVC는 `@ExceptionHandler` 어노테이션을 통해 컨트롤러 내에서 발생하는 예외를 효과적으로 처리할 수 있는 방법을 제공합니다.

- **역할:** `@ExceptionHandler` 어노테이션이 붙은 메소드는, 해당 컨트롤러(또는 `@ControllerAdvice`를 통해 여러 컨트롤러) 내의 다른 `@RequestMapping` 메소드가 실행되는 도중에 특정 타입의 예외가 발생했을 때, 그 예외를 "잡아서(catch)" 대신 처리하는 역할을 합니다.
- **동작 방식:** 마치 일반적인 `try-catch` 블록의 `catch` 부분과 유사하게 동작한다고 생각할 수 있습니다. 특정 예외가 발생하면, `DispatcherServlet`은 해당 예외 타입과 매칭되는 `@ExceptionHandler` 메소드를 찾아서 실행합니다.
- **목표:** 예외 발생 시 사용자에게는 친화적인 오류 페이지나 메시지를 보여주고, 서버는 내부적으로 로그를 기록하거나 필요한 후속 조치를 취할 수 있도록 합니다.

**`@ExceptionHandler`의 주요 특징:**

1. **컨트롤러 단위 또는 전역 단위 처리:**
  - **`@Controller` 클래스 내:** 해당 컨트롤러 내에서 발생하는 지정된 예외만 처리합니다.
  - **`@ControllerAdvice` 클래스 내:** 여러 컨트롤러 또는 애플리케이션 전역에서 발생하는 지정된 예외를 공통적으로 처리할 수 있습니다. (이 부분은 다음 챕터에서 더 자세히)
2. **유연한 예외 매핑:**
  - **단일 예외 타입 지정:** `@ExceptionHandler(IOException.class)` 와 같이 특정 예외 클래스를 지정합니다.
  - **여러 예외 타입 지정:** `@ExceptionHandler({FileSystemException.class, RemoteException.class})` 와 같이 중괄호 `{}` 안에 여러 예외 클래스를 나열할 수 있습니다. 이 중 하나라도 발생하면 해당 핸들러가 처리합니다.
  - **상위 예외 타입 지정:** `@ExceptionHandler(Exception.class)` 와 같이 상위 예외 타입을 지정하면, 해당 예외 타입 및 그 모든 하위 타입의 예외를 처리할 수 있습니다. (하지만 너무 광범위하게 잡는 것은 주의해야 합니다.)
3. **루트 예외 vs. 원인 예외 매핑:**
  - 예외는 종종 다른 예외를 감싸는 형태로 발생할 수 있습니다 (예: `ServiceLayerException` 안에 `IOException`이 원인으로 포함됨).
  - `@ExceptionHandler`는 전파되는 최상위 예외뿐만 아니라, 그 안에 중첩된 원인(cause) 예외와도 매칭될 수 있습니다.
  - 여러 핸들러가 매칭될 경우, 일반적으로 더 구체적인(발생한 예외와 더 가까운) 루트 예외 매칭이 우선시됩니다.
  - **권장 사항:** 혼란을 줄이기 위해 `@ExceptionHandler` 메소드의 인자 시그니처에서 처리할 예외 타입을 가능한 한 구체적으로 명시하는 것이 좋습니다.
4. **미디어 타입 기반 처리 (Content Negotiation):**
  - 클라이언트가 `Accept` 헤더를 통해 특정 미디어 타입(예: `application/json` 또는 `text/html`)의 응답을 요청했을 경우, 동일한 예외에 대해서도 다른 형식의 오류 응답을 제공할 수 있습니다.
  - `@ExceptionHandler(produces = "application/json")` 와 같이 `produces` 속성을 사용하여, 특정 `Accept` 헤더 요청에만 반응하는 핸들러를 만들 수 있습니다.
5. **유연한 메소드 인자 및 반환 타입:**
  - `@ExceptionHandler` 메소드는 `@RequestMapping` 메소드처럼 다양한 타입의 인자를 받을 수 있습니다 (예: 발생한 예외 객체, `WebRequest`, `Model`, `HttpServletResponse` 등). 이를 통해 예외 처리 시 필요한 다양한 정보에 접근할 수 있습니다.
  - 또한, 다양한 타입의 값을 반환하여 응답을 구성할 수 있습니다 (예: `ResponseEntity`, `String` (뷰 이름), `@ResponseBody`가 붙은 객체, `ModelAndView` 등).

**`HandlerExceptionResolver` 메커니즘:**

스프링 MVC에서 예외 처리는 내부적으로 `HandlerExceptionResolver` 인터페이스와 그 구현체들에 의해 이루어집니다. `DispatcherServlet`은 컨트롤러 실행 중 예외가 발생하면 등록된 `HandlerExceptionResolver`들을 순차적으로 호출하여 예외를 처리할 수 있는 핸들러를 찾습니다. `@ExceptionHandler` 어노테이션은 `ExceptionHandlerExceptionResolver`라는 `HandlerExceptionResolver` 구현체에 의해 처리됩니다.

### **주요 용어 해설**

- **`@ExceptionHandler`:** 특정 예외 타입을 처리하는 메소드임을 나타내는 어노테이션.
- **`@ControllerAdvice` (미리보기):** 여러 컨트롤러에 걸쳐 공통적으로 적용될 기능(예: 예외 처리, 모델 초기화, 데이터 바인딩 설정)을 정의하는 클래스에 사용되는 어노테이션.
- **루트 예외 (Root Exception):** 예외 연쇄(exception chain)에서 가장 바깥쪽, 즉 직접적으로 발생하거나 전파된 예외.
- **원인 예외 (Cause Exception):** 다른 예외의 발생 원인이 된, 내부에 중첩된 예외. (`Throwable.getCause()`로 접근 가능)
- **`ExceptionDepthComparator`:** 여러 `@ExceptionHandler` 메소드가 특정 예외와 매칭될 가능성이 있을 때, 어떤 핸들러를 우선적으로 선택할지 결정하기 위해 예외 계층 구조에서의 깊이를 비교하는 데 사용되는 내부 컴포넌트.
- **`HandlerExceptionResolver`:** 스프링 MVC에서 컨트롤러 실행 중 발생한 예외를 처리하는 전략을 정의하는 인터페이스.
- **내용 협상 (Content Negotiation):** 클라이언트와 서버가 서로 어떤 미디어 타입(Content Type)으로 데이터를 주고받을지 결정하는 과정. HTTP `Accept` 헤더(클라이언트가 받고 싶은 타입)와 `Content-Type` 헤더(실제 전송되는 타입)가 사용됩니다.

### **코드 예제 및 분석**

**1. 특정 예외 처리 (`IOException`)**

```java
import org.springframework.http.HttpStatus;
import org.springframework.http.ResponseEntity;
import org.springframework.stereotype.Controller;
import org.springframework.web.bind.annotation.ExceptionHandler;
import org.springframework.web.bind.annotation.GetMapping;

import java.io.IOException;

@Controller
public class FileOperationController {

    @GetMapping("/readFile")
    public String readFileOperation() throws IOException { // IOException 발생 가능성 시사
        // 실제 파일 읽기 로직 (예시로 예외 발생)
        if (true) { // 항상 예외 발생하도록
            throw new IOException("Simulated file read error");
        }
        return "fileContentView";
    }

    // 1. IOException 또는 그 하위 타입의 예외가 발생하면 이 핸들러가 호출됨
    @ExceptionHandler(IOException.class)
    public ResponseEntity<String> handleIOException(IOException ex) {
        System.err.println("IOException caught: " + ex.getMessage());
        // 2. 클라이언트에게는 500 Internal Server Error 상태와 함께 에러 메시지 반환
        return ResponseEntity.status(HttpStatus.INTERNAL_SERVER_ERROR)
                             .body("An error occurred while processing your file request: " + ex.getMessage());
    }
}

```

**코드 분석:**

1. `@ExceptionHandler(IOException.class)`: `readFileOperation()` 메소드 실행 중 `IOException` (또는 `FileNotFoundException`과 같은 `IOException`의 하위 클래스)이 발생하면 `handleIOException()` 메소드가 호출됩니다.
2. `ResponseEntity.status(HttpStatus.INTERNAL_SERVER_ERROR).body(...)`: 클라이언트에게는 HTTP 500 상태 코드와 함께 사용자 친화적인 오류 메시지가 담긴 응답 본문을 전달합니다. 발생한 예외 객체(`ex`)를 인자로 받아 상세 메시지를 활용할 수 있습니다.

**2. 여러 예외 타입을 하나의 핸들러에서 처리**

```java
// ... (imports 생략)
import org.springframework.web.bind.MissingServletRequestParameterException;
import java.nio.file.FileSystemException;

@Controller
public class AdvancedErrorController {

    @GetMapping("/processData")
    public String processData(@RequestParam String type) throws FileSystemException, MissingServletRequestParameterException {
        if ("file".equals(type)) {
            throw new FileSystemException("Simulated file system error");
        } else if (type == null || type.isEmpty()){ // type 파라미터가 없을 때 @RequestParam이 MissingServletRequestParameterException 발생시킴
            // 실제로는 @RequestParam(required=true)가 기본이라 이 코드는 도달 안 함.
            // 예시를 위해 type이 null이거나 비면 다른 예외를 던진다고 가정.
            // throw new MissingServletRequestParameterException("type", "String");
        }
        return "dataProcessedView";
    }

    // 1. FileSystemException 또는 MissingServletRequestParameterException 발생 시 이 핸들러 호출
    @ExceptionHandler({FileSystemException.class, MissingServletRequestParameterException.class})
    public ResponseEntity<String> handleSpecificErrors(Exception ex) { // 공통 상위 타입 Exception으로 받음
        String errorMessage;
        HttpStatus status = HttpStatus.INTERNAL_SERVER_ERROR; // 기본 상태

        if (ex instanceof FileSystemException) {
            errorMessage = "File system operation failed: " + ex.getMessage();
            status = HttpStatus.SERVICE_UNAVAILABLE; // 예시: 503 상태
        } else if (ex instanceof MissingServletRequestParameterException) {
            errorMessage = "Required parameter is missing: " + ((MissingServletRequestParameterException) ex).getParameterName();
            status = HttpStatus.BAD_REQUEST; // 400 상태
        } else {
            errorMessage = "An unexpected error occurred.";
        }
        System.err.println("Caught exception: " + ex.getClass().getSimpleName() + " - " + ex.getMessage());
        return ResponseEntity.status(status).body(errorMessage);
    }
}

```

**코드 분석:**

1. `@ExceptionHandler({FileSystemException.class, MissingServletRequestParameterException.class})`: `FileSystemException` 또는 `MissingServletRequestParameterException` 중 어느 하나라도 발생하면 `handleSpecificErrors()` 메소드가 호출됩니다.
2. 메소드 내부에서는 `instanceof` 연산자를 사용하여 실제 발생한 예외 타입에 따라 다른 오류 메시지나 상태 코드를 설정하여 응답할 수 있습니다.

**3. 미디어 타입 기반 예외 처리**

`IllegalArgumentException` 발생 시, 클라이언트가 JSON을 원하면 JSON 오류 응답, HTML을 원하면 HTML 오류 페이지를 보여주는 예제 (원문 코드 재활용):

```java
// ... (ErrorMessage 클래스는 간단한 오류 메시지 담는 POJO라고 가정)
// public class ErrorMessage { public String message; public int code; /* 생성자, getter */ }

@Controller
public class MediaTypeErrorController {

    @GetMapping("/riskyOperation")
    public String performRiskyOperation(@RequestParam int value) {
        if (value < 0) {
            throw new IllegalArgumentException("Value cannot be negative.");
        }
        return "operationSuccessView";
    }

    // 1. 클라이언트가 Accept: application/json 헤더를 보냈을 때 이 핸들러가 우선적으로 매칭됨
    @ExceptionHandler(value = IllegalArgumentException.class, produces = "application/json")
    @ResponseBody // JSON 응답을 위해 필요 (또는 클래스가 @RestController)
    public ResponseEntity<ErrorMessage> handleIllegalArgumentAsJson(IllegalArgumentException exc) {
        System.err.println("Handling IllegalArgumentException as JSON");
        ErrorMessage error = new ErrorMessage(exc.getMessage(), 4001); // 예시 에러 코드
        return ResponseEntity.badRequest().body(error);
    }

    // 2. 클라이언트가 Accept: text/html 헤더를 보냈거나, Accept 헤더가 없거나,
    //    JSON 핸들러와 매칭되지 않을 때 이 핸들러가 매칭됨
    @ExceptionHandler(value = IllegalArgumentException.class, produces = "text/html")
    public String handleIllegalArgumentAsHtml(IllegalArgumentException exc, Model model) {
        System.err.println("Handling IllegalArgumentException as HTML");
        ErrorMessage error = new ErrorMessage(exc.getMessage(), 4002);
        model.addAttribute("errorDetails", error);
        return "customErrorPage"; // customErrorPage.html 또는 .jsp 뷰로 이동
    }
}

```

**코드 분석:**

1. `@ExceptionHandler(value = IllegalArgumentException.class, produces = "application/json")`: `IllegalArgumentException`이 발생하고, 클라이언트 요청의 `Accept` 헤더가 `application/json`을 포함하면 `handleIllegalArgumentAsJson()` 메소드가 호출됩니다. 이 메소드는 `ErrorMessage` 객체를 JSON으로 변환하여 응답합니다.
2. `@ExceptionHandler(value = IllegalArgumentException.class, produces = "text/html")`: 동일한 `IllegalArgumentException`에 대해, 클라이언트 요청의 `Accept` 헤더가 `text/html`을 포함하거나 다른 핸들러와 매칭되지 않을 경우 `handleIllegalArgumentAsHtml()` 메소드가 호출됩니다. 이 메소드는 모델에 오류 정보를 담아 HTML 뷰 페이지("customErrorPage")로 이동합니다.

### **"왜?" 라는 질문에 대한 답변**

**`@ExceptionHandler`는 왜 필요할까요? 각 컨트롤러 메소드마다 `try-catch`를 쓰면 안 되나?**

1. **코드 중복 제거 및 관심사의 분리:** 여러 컨트롤러 메소드에서 동일한 종류의 예외(예: `IOException`, `SQLException`)가 발생할 수 있습니다. 각 메소드마다 동일한 `try-catch` 로직을 반복해서 작성하는 것은 매우 비효율적이고 코드 중복을 유발합니다. `@ExceptionHandler`를 사용하면 예외 처리 로직을 한 곳(또는 `@ControllerAdvice`를 통해 중앙 집중적인 곳)에 모아둘 수 있어, 컨트롤러 메소드는 핵심 비즈니스 로직에만 집중할 수 있게 됩니다. 이것이 바로 관심사의 분리(SoC)입니다.
2. **일관된 예외 처리 정책 적용:** 애플리케이션 전체적으로 또는 특정 컨트롤러 그룹에 대해 일관된 예외 처리 방식(예: 특정 형식의 오류 응답, 특정 로깅 방식)을 적용하기 용이합니다.
3. **선언적인 예외 처리:** `@ExceptionHandler`는 "이런 예외가 발생하면 이 메소드가 처리한다"고 선언적으로 명시하는 방식입니다. 이는 코드의 가독성을 높이고 의도를 명확하게 전달합니다.
4. **스프링 MVC의 풍부한 기능 활용:** `@ExceptionHandler` 메소드는 일반 컨트롤러 메소드처럼 다양한 매개변수(예외 객체, 요청 정보, 응답 객체, 모델 등)를 받을 수 있고, 다양한 방식(`ResponseEntity`, 뷰 이름, `@ResponseBody` 객체 등)으로 응답을 생성할 수 있어, 스프링 MVC의 다른 기능들과 자연스럽게 통합됩니다. 일반적인 `try-catch`로는 이런 유연성을 확보하기 어렵습니다.

### **주의사항 및 Best Practice**

1. **구체적인 예외 타입 우선 처리:** 가능한 한 구체적인 예외 타입을 처리하는 핸들러를 만들고, 가장 일반적인 `Exception.class`를 처리하는 핸들러는 최후의 수단(fallback)으로 사용하는 것이 좋습니다. 너무 광범위한 예외를 하나의 핸들러에서 모두 처리하려고 하면 로직이 복잡해질 수 있습니다.
2. **`@ControllerAdvice` 활용:** 여러 컨트롤러에 걸쳐 공통적으로 발생하는 예외(예: 인증/인가 오류, 데이터베이스 관련 공통 오류)는 `@ControllerAdvice` 클래스에 `@ExceptionHandler` 메소드를 정의하여 전역적으로 처리하는 것이 매우 효과적입니다.
3. **사용자 친화적인 오류 응답:** 실제 운영 환경에서는 사용자에게 스택 트레이스(stack trace)나 내부 오류 코드를 그대로 노출하는 것은 피해야 합니다. 의미 있는 오류 메시지와 함께 적절한 HTTP 상태 코드를 반환해야 합니다.
4. **로깅:** 예외 발생 시 상세한 내용을 로그로 남기는 것은 문제 해결에 매우 중요합니다. `@ExceptionHandler` 메소드 내에서 로깅 로직을 포함하는 것이 좋습니다.
5. **순환 참조 주의 (루트 vs. 원인):** 예외 매핑 시 루트 예외와 원인 예외 간의 관계를 이해하고, 의도치 않은 핸들러가 호출되지 않도록 주의해야 합니다. 가능하다면 가장 직접적인 예외 타입을 명시하는 것이 좋습니다.
6. **예외 다시 던지기 (Rethrowing):** 특정 조건에서는 `@ExceptionHandler` 메소드가 예외를 처리하지 않고 다시 던져서 다른 핸들러(예: 더 일반적인 핸들러 또는 서블릿 컨테이너의 기본 오류 페이지)가 처리하도록 할 수 있습니다.

### **이전 학습 내용과의 연관성**

- **`ResponseEntity`:** `@ExceptionHandler` 메소드가 반환 타입으로 `ResponseEntity`를 사용하여 상태 코드, 헤더, 본문을 포함한 상세한 오류 응답을 구성할 수 있습니다.
- **`@ResponseBody` / `@RestController`:** `@ExceptionHandler` 메소드에 `@ResponseBody`를 붙이거나, `@ControllerAdvice` 클래스가 `@RestControllerAdvice`이면, 반환된 객체가 JSON/XML 등으로 직렬화되어 응답 본문에 직접 쓰여집니다. (API 오류 응답 시 유용)
- **`Model`:** HTML 뷰를 통해 오류 페이지를 보여줄 경우, `@ExceptionHandler` 메소드에서 `Model` 객체에 오류 정보를 담아 뷰로 전달할 수 있습니다.
- **검증 (Validation) 예외:** 이전 챕터에서 배운 `MethodArgumentNotValidException`이나 `HandlerMethodValidationException` 같은 검증 관련 예외들도 `@ExceptionHandler`를 통해 효과적으로 처리하고 사용자 정의 응답을 만들 수 있습니다.

---
