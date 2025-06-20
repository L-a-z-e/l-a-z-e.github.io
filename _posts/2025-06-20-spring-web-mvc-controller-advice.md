---
title: Spring Web MVC - Controller Advice
description: 
author: laze
date: 2025-06-20 00:00:01 +0900
categories: [Dev, SpringWebMVC]
tags: [SpringWebMVC]
---
### Controller Advice

`@ExceptionHandler`, `@InitBinder`, 그리고 `@ModelAttribute` 메소드는 그것들이 선언된 `@Controller` 클래스 또는 클래스 계층 구조에만 적용됩니다.

만약 대신 `@ControllerAdvice` 또는 `@RestControllerAdvice` 클래스에 선언된다면, 모든 컨트롤러에 적용됩니다.

더욱이, 5.3 버전부터 `@ControllerAdvice` 내의 `@ExceptionHandler` 메소드는 모든 `@Controller` 또는 다른 핸들러로부터 발생하는 예외를 처리하는 데 사용될 수 있습니다.

`@ControllerAdvice`는 `@Component`로 메타 어노테이션되어 있으므로 컴포넌트 스캐닝을 통해 스프링 빈으로 등록될 수 있습니다.

`@RestControllerAdvice`는 `@ControllerAdvice`와 `@ResponseBody`를 결합한 단축 어노테이션으로, 실제로는 예외 핸들러 메소드가 응답 본문으로 렌더링되는 `@ControllerAdvice`일 뿐입니다.

시작 시, `RequestMappingHandlerMapping`과 `ExceptionHandlerExceptionResolver`는 컨트롤러 어드바이스 빈을 감지하고 런타임에 이를 적용합니다.

`@ControllerAdvice`로부터의 전역 `@ExceptionHandler` 메소드는 `@Controller`로부터의 지역적인 것들 이후에 적용됩니다.

반대로, 전역 `@ModelAttribute`와 `@InitBinder` 메소드는 지역적인 것들 이전에 적용됩니다.

기본적으로 `@ControllerAdvice`와 `@RestControllerAdvice` 모두 `@Controller`와 `@RestController`를 포함한 모든 컨트롤러에 적용됩니다.

어노테이션의 속성을 사용하여 적용될 컨트롤러 및 핸들러의 집합을 좁힐 수 있습니다.

```java
// @RestController로 어노테이션된 모든 컨트롤러 대상
@ControllerAdvice(annotations = RestController.class)
public class ExampleAdvice1 {}

// 특정 패키지 내의 모든 컨트롤러 대상
@ControllerAdvice("org.example.controllers")
public class ExampleAdvice2 {}

// 특정 클래스에 할당 가능한 모든 컨트롤러 대상 (ControllerInterface를 구현하거나 AbstractController를 상속한 컨트롤러)
@ControllerAdvice(assignableTypes = {ControllerInterface.class, AbstractController.class})
public class ExampleAdvice3 {}
```

위 예제의 선택자(selectors)들은 런타임에 평가되며, 광범위하게 사용될 경우 성능에 부정적인 영향을 미칠 수 있습니다.

---

### **학습 목표 제시**

이 챕터를 학습하고 나면, 다음을 이해하고 수행할 수 있게 됩니다:

1. **`@ControllerAdvice` 및 `@RestControllerAdvice`의 핵심 역할 이해:** 이 어노테이션들이 여러 컨트롤러에 걸쳐 공통적인 관심사(예외 처리, 모델 초기화, 데이터 바인딩 설정)를 중앙에서 관리하는 방법을 이해합니다.
2. **전역 `@ExceptionHandler`, `@InitBinder`, `@ModelAttribute` 메소드 적용 방법 숙지:** `@ControllerAdvice` 클래스 내에 이러한 어노테이션이 붙은 메소드를 정의하여, 특정 컨트롤러에 국한되지 않고 여러 컨트롤러에 일관된 로직을 적용하는 방법을 이해하고 사용할 수 있습니다.
3. **적용 순서 및 대상 컨트롤러 제한 방법 이해:** 전역 어드바이스와 지역 컨트롤러 설정 간의 적용 우선순위(특히 `@ExceptionHandler`와 `@ModelAttribute`/`@InitBinder`의 차이)를 이해하고, 어노테이션 속성을 사용하여 어드바이스가 적용될 컨트롤러의 범위를 제한하는 방법을 인지합니다.

---

### **핵심 개념 설명**

**`@ControllerAdvice`란 무엇일까요?**

우리가 이전에 배운 `@ExceptionHandler`, `@InitBinder`, 그리고 `@ModelAttribute` 메소드들은 기본적으로 그것들이 선언된 **특정 `@Controller` 클래스 내에서만 동작**했습니다.

하지만, 여러 컨트롤러에서 **공통적으로 처리해야 하는 예외**가 있거나, **모든 컨트롤러의 모델에 공통적으로 추가해야 하는 데이터**가 있거나, **여러 컨트롤러에 동일한 데이터 바인딩 규칙을 적용**하고 싶을 때가 많습니다.

이런 경우 각 컨트롤러마다 동일한 코드를 반복해서 작성하는 것은 매우 비효율적입니다.

`@ControllerAdvice` 어노테이션은 바로 이러한 **여러 컨트롤러에 걸쳐 공통적으로 적용하고자 하는 관심사(concerns)들을 하나의 클래스에 모아서 정의**할 수 있게 해주는 특별한 어노테이션입니다.

`@ControllerAdvice`가 붙은 클래스는 일종의 "전역 컨트롤러 조언자" 역할을 하며, 다른 컨트롤러들의 동작에 영향을 줄 수 있습니다.

**`@ControllerAdvice` 클래스 내에서 사용할 수 있는 주요 어노테이션:**

- **`@ExceptionHandler`:** 여러 컨트롤러에서 발생하는 특정 예외들을 한 곳에서 공통적으로 처리합니다.
- **`@InitBinder`:** 여러 컨트롤러에 적용될 공통적인 `WebDataBinder` 초기화 설정을 정의합니다. (예: 전역적인 날짜 형식 변환 규칙)
- **`@ModelAttribute`:** 여러 컨트롤러의 모델에 공통적으로 추가될 데이터를 준비합니다. (예: 모든 페이지에 표시될 현재 사용자 정보, 웹사이트 공통 설정 값)

**`@RestControllerAdvice`**

`@RestControllerAdvice`는 `@ControllerAdvice`와 `@ResponseBody`를 합쳐놓은 편리한 메타 어노테이션입니다.
즉, `@RestControllerAdvice` 클래스 내에 정의된 `@ExceptionHandler` 메소드들은 기본적으로 반환 값을 HTTP 응답 본문으로 직접 직렬화합니다 (주로 REST API의 오류 응답을 JSON 등으로 보낼 때 유용).

**`@ControllerAdvice`의 동작 원리:**

1. **컴포넌트 스캔:** `@ControllerAdvice`는 내부적으로 `@Component` 어노테이션을 포함하고 있으므로, 컴포넌트 스캔 대상이 되어 스프링 빈으로 등록됩니다.
2. **감지 및 적용:** 애플리케이션 시작 시, 스프링 MVC의 핵심 컴포넌트들(예: `RequestMappingHandlerMapping`, `ExceptionHandlerExceptionResolver`)은 등록된 `@ControllerAdvice` 빈들을 감지합니다.
3. **런타임 적용:** 실제 요청 처리 과정에서, 이 `@ControllerAdvice` 빈들에 정의된 메소드들(`@ExceptionHandler`, `@InitBinder`, `@ModelAttribute`)이 적절한 시점에 다른 컨트롤러들의 동작에 영향을 미치게 됩니다.

**적용 순서 (지역 vs. 전역):**

이 부분이 조금 헷갈릴 수 있는데, 중요합니다!

- **`@ExceptionHandler` (예외 처리):**
  1. **지역 `@ExceptionHandler` (컨트롤러 내) 우선:** 만약 예외가 발생한 컨트롤러 내에 해당 예외를 처리할 수 있는 `@ExceptionHandler` 메소드가 있다면, 그것이 먼저 실행됩니다.
  2. **전역 `@ExceptionHandler` (ControllerAdvice 내) 차선:** 지역 핸들러가 없거나 처리하지 못하면, 그때 `@ControllerAdvice`에 정의된 전역 핸들러가 실행됩니다.
  - 여러 `@ControllerAdvice` 빈이 있고, 여러 핸들러가 특정 예외와 매칭될 수 있다면, 더 구체적인 예외 매칭이나 `@Order` 어노테이션 등을 통한 우선순위 설정이 고려됩니다. (루트 예외 vs. 원인 예외 매칭 규칙도 적용됨)
- **`@ModelAttribute` 및 `@InitBinder` (모델 초기화 및 바인더 설정):**
  1. **전역 `@ModelAttribute`/`@InitBinder` (ControllerAdvice 내) 우선:** `@ControllerAdvice`에 정의된 `@ModelAttribute` 또는 `@InitBinder` 메소드가 먼저 실행됩니다.
  2. **지역 `@ModelAttribute`/`@InitBinder` (컨트롤러 내) 차선:** 그 후에 해당 컨트롤러 내에 정의된 지역 `@ModelAttribute` 또는 `@InitBinder` 메소드가 실행됩니다.
  - 이 순서 덕분에 전역적으로 공통 설정을 해두고, 특정 컨트롤러에서는 필요에 따라 추가 설정을 하거나 덮어쓸 수 있는 유연성을 가집니다.

**비유:**

여러분이 회사(애플리케이션)의 여러 부서(컨트롤러)에서 발생하는 문제(예외)나 공통 업무 지침(모델 데이터, 바인더 설정)을 관리한다고 생각해봅시다.

- **각 부서(컨트롤러) 내부 규칙:** 각 부서마다 자체적으로 처리하는 문제 해결 방식이나 업무 지침이 있을 수 있습니다 (지역 `@ExceptionHandler`, `@ModelAttribute`, `@InitBinder`).
- **회사 전체 공통 대응팀/지침서 (`@ControllerAdvice`):** 회사 전체적으로 적용되는 문제 해결 절차(전역 `@ExceptionHandler`)나 모든 부서원이 알아야 할 공통 정보(전역 `@ModelAttribute`), 또는 공통 업무 양식 작성법(전역 `@InitBinder`)을 만들어 배포합니다.

이때,

- **문제 발생 시(예외 처리):** 먼저 해당 부서에서 자체적으로 해결하려고 시도하고(지역 핸들러), 해결 못 하면 회사 전체 대응팀(전역 핸들러)에 알립니다.
- **업무 시작 전 준비(모델/바인더 초기화):** 먼저 회사 전체 공통 지침서(전역 어드바이스)를 따르고, 그 위에 각 부서의 특수한 지침(지역 어드바이스)을 추가로 적용합니다.

### **주요 용어 해설**

- **`@ControllerAdvice`:** 여러 컨트롤러에 걸쳐 공통 관심사를 적용하기 위한 어노테이션. `@Component`를 포함하여 빈으로 등록됨.
- **`@RestControllerAdvice`:** `@ControllerAdvice` + `@ResponseBody`. 주로 REST API의 전역 예외 처리 등에 사용.
- **전역(Global) vs. 지역(Local):**
  - **전역:** `@ControllerAdvice`에 정의되어 여러 컨트롤러에 영향을 미치는 설정.
  - **지역:** 특정 `@Controller` 클래스 내에 정의되어 해당 컨트롤러에만 영향을 미치는 설정.
- **선택자 (Selectors) 속성:** `@ControllerAdvice` 어노테이션에 사용하여 어드바이스가 적용될 컨트롤러의 범위를 제한하는 속성들 (예: `annotations`, `basePackages`, `assignableTypes`).

### **코드 예제 및 분석**

**1. 전역 예외 처리 (`@ControllerAdvice`와 `@ExceptionHandler`)**

`CustomException.java` (커스텀 예외):

```java
public class CustomException extends RuntimeException {
    public CustomException(String message) {
        super(message);
    }
}
```

`GlobalExceptionHandler.java` (`@ControllerAdvice` 사용):

```java
import org.springframework.http.HttpStatus;
import org.springframework.http.ResponseEntity;
import org.springframework.web.bind.annotation.ControllerAdvice;
import org.springframework.web.bind.annotation.ExceptionHandler;
import org.springframework.web.bind.annotation.ResponseBody; // 만약 @RestControllerAdvice를 안 쓴다면 필요

// @ControllerAdvice // 일반 컨트롤러와 REST 컨트롤러 모두에 대한 전역 예외 처리
@RestControllerAdvice // @ControllerAdvice + @ResponseBody, REST API에 더 적합
public class GlobalExceptionHandler {

    // 1. CustomException이 어떤 컨트롤러에서든 발생하면 이 핸들러가 처리
    @ExceptionHandler(CustomException.class)
    public ResponseEntity<ErrorResponse> handleCustomException(CustomException ex) {
        System.err.println("GlobalExceptionHandler caught CustomException: " + ex.getMessage());
        ErrorResponse errorResponse = new ErrorResponse("CustomError", ex.getMessage());
        return new ResponseEntity<>(errorResponse, HttpStatus.BAD_REQUEST); // 400 응답
    }

    // 2. 모든 종류의 Exception을 처리 (가장 마지막에 매칭될 가능성이 높음)
    @ExceptionHandler(Exception.class)
    public ResponseEntity<ErrorResponse> handleGenericException(Exception ex) {
        System.err.println("GlobalExceptionHandler caught Exception: " + ex.getMessage());
        ErrorResponse errorResponse = new ErrorResponse("ServerError", "An unexpected error occurred.");
        return new ResponseEntity<>(errorResponse, HttpStatus.INTERNAL_SERVER_ERROR); // 500 응답
    }
}

// 간단한 ErrorResponse POJO
class ErrorResponse {
    public String code;
    public String message;
    public ErrorResponse(String code, String message) { this.code = code; this.message = message; }
}
```

`MyController.java` (예외를 발생시키는 컨트롤러):

```java
import org.springframework.stereotype.Controller;
import org.springframework.web.bind.annotation.GetMapping;
import org.springframework.web.bind.annotation.ResponseBody;

@Controller
public class MyController {

    @GetMapping("/test-custom-exception")
    @ResponseBody
    public String testCustomException() {
        throw new CustomException("Something went wrong with custom logic!");
    }

    @GetMapping("/test-generic-exception")
    @ResponseBody
    public String testGenericException() {
        throw new RuntimeException("A generic runtime error!"); // CustomException이 아닌 다른 예외
    }
}
```

**코드 분석:**

- `GlobalExceptionHandler` 클래스에 `@RestControllerAdvice` (또는 `@ControllerAdvice` + 각 메소드에 `@ResponseBody`)를 붙여 전역 예외 처리기로 만듭니다.
- `handleCustomException()`: `MyController` (또는 다른 어떤 컨트롤러든)에서 `CustomException`이 발생하면 이 메소드가 호출되어 JSON 형태의 `ErrorResponse`와 함께 400 상태 코드를 반환합니다.
- `handleGenericException()`: 다른 특정 핸들러에서 잡지 못한 모든 `Exception` 타입(및 그 하위 타입)의 예외가 발생하면 이 메소드가 최후에 호출되어 500 상태 코드와 함께 일반적인 오류 메시지를 반환합니다.

**2. 전역 모델 속성 추가 (`@ControllerAdvice`와 `@ModelAttribute`)**

```java
import org.springframework.ui.Model;
import org.springframework.web.bind.annotation.ControllerAdvice;
import org.springframework.web.bind.annotation.ModelAttribute;

@ControllerAdvice
public class GlobalModelAttributes {

    // 이 메소드는 모든 컨트롤러의 @RequestMapping 메소드 실행 전에 호출됨
    @ModelAttribute
    public void addCommonAttributes(Model model) {
        model.addAttribute("appName", "My Awesome Application");
        model.addAttribute("appVersion", "1.0.2");
        System.out.println("GlobalModelAttributes: appName and appVersion added to model.");
    }
}
```

**코드 분석:**

- `GlobalModelAttributes` 클래스에 `@ControllerAdvice`를 붙입니다.
- `addCommonAttributes()` 메소드에 `@ModelAttribute`를 붙이면, 어떤 컨트롤러의 어떤 `@RequestMapping` 메소드가 실행되든지 간에, 그 전에 이 메소드가 먼저 실행되어 `Model` 객체에 "appName"과 "appVersion"이라는 공통 속성이 추가됩니다.
- 따라서 모든 뷰 페이지에서 `${appName}`이나 `${appVersion}`과 같이 이 값들을 사용할 수 있게 됩니다.

**3. 전역 `WebDataBinder` 초기화 (`@ControllerAdvice`와 `@InitBinder`)**

```java
import org.springframework.beans.propertyeditors.CustomDateEditor;
import org.springframework.web.bind.WebDataBinder;
import org.springframework.web.bind.annotation.ControllerAdvice;
import org.springframework.web.bind.annotation.InitBinder;

import java.text.SimpleDateFormat;
import java.util.Date;

@ControllerAdvice
public class GlobalDataBinderAdvice {

    // 이 메소드는 모든 컨트롤러의 데이터 바인딩 전에 호출됨
    @InitBinder
    public void globalInitBinder(WebDataBinder binder) {
        SimpleDateFormat dateFormat = new SimpleDateFormat("yyyy-MM-dd HH:mm:ss");
        dateFormat.setLenient(false);
        binder.registerCustomEditor(Date.class, new CustomDateEditor(dateFormat, true)); // 빈 문자열을 null로 허용
        System.out.println("GlobalDataBinderAdvice: CustomDateEditor registered globally.");
    }
}
```

**코드 분석:**

- `GlobalDataBinderAdvice` 클래스에 `@ControllerAdvice`를 붙입니다.
- `globalInitBinder()` 메소드에 `@InitBinder`를 붙이면, 어떤 컨트롤러에서 데이터 바인딩이 일어나든지 간에, 그 전에 이 메소드가 먼저 실행되어 `WebDataBinder`에 전역적인 설정을 적용합니다.
- 위 예제에서는 모든 `Date` 타입에 대해 "yyyy-MM-dd HH:mm:ss" 형식의 문자열을 파싱하는 `CustomDateEditor`를 등록했습니다. 따라서 모든 컨트롤러에서 이 형식의 날짜 문자열 파라미터를 `Date` 객체로 바인딩 받을 수 있게 됩니다.

**4. 적용 대상 컨트롤러 제한**

원문의 예제처럼 `@ControllerAdvice` 어노테이션의 속성을 사용하여 특정 조건에 맞는 컨트롤러에만 어드바이스를 적용할 수 있습니다.

- `@ControllerAdvice(annotations = RestController.class)`: `@RestController` 어노테이션이 붙은 컨트롤러에만 적용.
- `@ControllerAdvice("com.example.api.controllers")`: `com.example.api.controllers` 패키지 및 그 하위 패키지에 있는 컨트롤러에만 적용.
- `@ControllerAdvice(assignableTypes = {BaseApiController.class, AdminController.class})`: `BaseApiController`를 상속하거나 `AdminController` 타입인 컨트롤러에만 적용.

### **"왜?" 라는 질문에 대한 답변**

**`@ControllerAdvice`는 왜 필요할까요? 중복이 좀 있어도 각 컨트롤러에서 처리하면 안 되나?**

1. **DRY 원칙 (Don't Repeat Yourself - 반복하지 마라):** 가장 큰 이유입니다. 여러 컨트롤러에 걸쳐 동일하거나 유사한 예외 처리 로직, 모델 초기화 로직, 바인더 설정 로직이 반복된다면, 코드가 길어지고, 수정이 필요할 때 모든 곳을 찾아 바꿔야 하며, 실수가 발생할 가능성도 커집니다. `@ControllerAdvice`는 이러한 공통 로직을 한 곳에 중앙 집중화하여 코드 중복을 획기적으로 줄여줍니다.
2. **유지보수성 향상:** 공통 로직이 한 곳에 모여 있으므로, 수정이나 확장이 필요할 때 해당 `@ControllerAdvice` 클래스만 보면 됩니다. 이는 애플리케이션 전체의 유지보수성을 크게 향상시킵니다.
3. **애플리케이션 전체의 일관성 있는 정책 적용:** 전역 예외 처리 페이지의 통일, 모든 페이지에 공통적으로 필요한 데이터 제공, 전사적인 날짜 형식 통일 등 애플리케이션 전반에 걸쳐 일관된 정책을 적용하는 데 매우 효과적입니다.
4. **관심사의 명확한 분리:**
  - 개별 컨트롤러는 자신의 핵심 비즈니스 로직 처리에만 집중할 수 있습니다.
  - 공통적인 횡단 관심사(cross-cutting concerns)는 `@ControllerAdvice`에게 위임함으로써 역할 분담이 명확해집니다.

물론, 아주 작은 애플리케이션이거나 공통 로직이 거의 없다면 `@ControllerAdvice`의 필요성이 낮을 수 있습니다. 하지만 애플리케이션 규모가 커지고 복잡해질수록 `@ControllerAdvice`는 코드의 품질과 생산성을 높이는 데 매우 중요한 역할을 합니다.

### **주의사항 및 Best Practice**

1. **적용 순서 명확히 이해:** `@ExceptionHandler`는 지역 우선, `@ModelAttribute`/`@InitBinder`는 전역 우선이라는 적용 순서를 잘 이해하고 있어야 의도치 않은 동작을 피할 수 있습니다.
2. **적용 범위 신중히 결정:** 기본적으로 모든 컨트롤러에 적용되므로, 너무 광범위한 어드바이스는 예상치 못한 사이드 이펙트를 유발할 수 있습니다. 필요하다면 `annotations`, `basePackages`, `assignableTypes` 속성을 사용하여 적용 대상을 명확히 제한하는 것이 좋습니다.
3. **`@Order` 어노테이션 활용:** 여러 `@ControllerAdvice` 빈이 존재하고, 이들 간의 적용 순서가 중요하다면 `@Order` 어노테이션이나 `Ordered` 인터페이스를 구현하여 우선순위를 지정할 수 있습니다. (특히 예외 처리 순서에 중요)
4. **성능 영향 가능성 (선택자 사용 시):** 원문에서 언급했듯이, `@ControllerAdvice`의 선택자 속성들(특히 `assignableTypes`처럼 런타임 타입 체크가 필요한 경우)은 런타임에 평가되므로, 매우 광범위하게 사용되거나 복잡한 조건일 경우 약간의 성능 영향을 줄 수 있습니다. 하지만 대부분의 일반적인 경우에는 큰 문제가 되지 않습니다.
5. **`@RestControllerAdvice`의 적절한 사용:** API 엔드포인트에 대한 전역 예외 처리를 하고, 오류 응답을 JSON/XML 등으로 보내야 한다면 `@ControllerAdvice` + 각 핸들러에 `@ResponseBody`를 붙이는 것보다 `@RestControllerAdvice`를 사용하는 것이 더 간결하고 명확합니다.

### **이전 학습 내용과의 연관성**

- **`@ExceptionHandler`, `@InitBinder`, `@ModelAttribute` (메소드 레벨):** 이 세 가지 어노테이션은 원래 개별 컨트롤러 내에서 지역적으로 사용되던 기능들입니다. `@ControllerAdvice`는 이 기능들을 여러 컨트롤러에 걸쳐 "전역적으로" 또는 "선택적으로 그룹화하여" 적용할 수 있도록 확장해주는 역할을 합니다. 이들의 기본적인 사용법을 이해하고 있어야 `@ControllerAdvice`의 효과를 제대로 활용할 수 있습니다.
- **컴포넌트 스캔 (`@Component`):** `@ControllerAdvice`는 `@Component`를 포함하므로, 스프링의 컴포넌트 스캔 메커니즘에 의해 자동으로 빈으로 등록되고 관리됩니다.
- **AOP (Aspect-Oriented Programming) 개념과 유사성:** `@ControllerAdvice`는 AOP의 "어드바이스(Advice)" 개념과 유사하게, 여러 컨트롤러(조인 포인트)에 걸쳐 공통적인 부가 기능(횡단 관심사)을 적용하는 방식으로 동작합니다. (실제로 AOP 프록시가 사용되기도 합니다.)

---
