---
title: Spring Web MVC - Validation
description: 
author: laze
date: 2025-06-18 00:00:01 +0900
categories: [Dev, SpringWebMVC]
tags: [SpringWebMVC]
---
### Validation

스프링 MVC는 자바 빈 검증(Java Bean Validation)을 포함하여 `@RequestMapping` 메소드에 대한 내장 검증 기능을 가지고 있습니다. 검증은 다음 두 가지 레벨 중 하나에서 적용될 수 있습니다:

1. **메소드 인자 레벨 검증:**
  - `@ModelAttribute`, `@RequestBody`, 그리고 `@RequestPart` 인자 리졸버는 메소드 파라미터가 Jakarta의 `@Valid` 또는 스프링의 `@Validated` 어노테이션으로 표시되어 있고, **그리고** 바로 뒤에 `Errors` 또는 `BindingResult` 파라미터가 없으며, **그리고** 메소드 검증이 필요하지 않은 경우(다음에 설명)에 해당 메소드 인자를 개별적으로 검증합니다.
  - 이 경우 발생하는 예외는 `MethodArgumentNotValidException`입니다.
2. **메소드 레벨 검증:**
  - `@Min`, `@NotBlank` 등과 같은 `@Constraint` 어노테이션이 메소드 파라미터에 직접 선언되거나, 메소드(반환 값에 대해)에 선언된 경우, 메소드 검증이 적용되어야 하며, 이는 메소드 파라미터 제약 조건과 `@Valid`를 통한 중첩 제약 조건 모두를 포괄하기 때문에 메소드 인자 레벨 검증을 대체합니다.
  - 이 경우 발생하는 예외는 `HandlerMethodValidationException`입니다.

애플리케이션은 컨트롤러 메소드 시그니처에 따라 둘 중 어느 것이든 발생할 수 있으므로 `MethodArgumentNotValidException`과 `HandlerMethodValidationException` 모두를 처리해야 합니다.

그러나 이 두 예외는 매우 유사하게 설계되었으며 거의 동일한 코드로 처리될 수 있습니다.

주요 차이점은 전자가 단일 객체에 대한 것인 반면, 후자는 메소드 파라미터 목록에 대한 것이라는 점입니다.

`@Valid`는 제약 조건 어노테이션이 아니라, 객체 내의 중첩된 제약 조건(nested constraints)을 위한 것입니다.

따라서 `@Valid` 자체만으로는 메소드 검증을 유발하지 않습니다.

반면에 `@NotNull`은 제약 조건이며, `@Valid` 파라미터에 `@NotNull`을 추가하면 메소드 검증을 유발합니다.

특히 null 허용 여부에 대해서는 `@RequestBody` 또는 `@ModelAttribute`의 `required` 플래그를 사용할 수도 있습니다.

메소드 검증은 `Errors` 또는 `BindingResult` 메소드 파라미터와 함께 사용될 수 있습니다.

그러나 컨트롤러 메소드는 모든 검증 오류가 바로 뒤에 `Errors`가 있는 메소드 파라미터에 있는 경우에만 호출됩니다.

다른 메소드 파라미터에 검증 오류가 있는 경우 `HandlerMethodValidationException`이 발생합니다.

`WebMvc` 설정을 통해 전역적으로 `Validator`를 구성하거나, `@Controller` 또는 `@ControllerAdvice`의 `@InitBinder` 메소드를 통해 지역적으로 구성할 수 있습니다.

여러 `Validator`를 사용할 수도 있습니다.

컨트롤러에 클래스 레벨 `@Validated`가 있는 경우, 메소드 검증은 AOP 프록시를 통해 적용됩니다.

스프링 프레임워크 6.1에 추가된 스프링 MVC 내장 메소드 검증 지원을 활용하려면 컨트롤러에서 클래스 레벨 `@Validated` 어노테이션을 제거해야 합니다.

오류 응답(Error Responses) 섹션에서는 `MethodArgumentNotValidException`과 `HandlerMethodValidationException`이 어떻게 처리되는지,

그리고 `MessageSource`와 로케일 및 언어별 리소스 번들을 통해 렌더링을 어떻게 사용자 정의할 수 있는지에 대한 자세한 내용을 제공합니다.

메소드 검증 오류에 대한 추가적인 사용자 정의 처리를 위해, `ResponseEntityExceptionHandler`를 확장하거나

컨트롤러 또는 `@ControllerAdvice`에서 `@ExceptionHandler` 메소드를 사용하여 `HandlerMethodValidationException`을 직접 처리할 수 있습니다.

이 예외에는 메소드 파라미터별로 검증 오류를 그룹화하는 `ParameterValidationResult` 목록이 포함되어 있습니다.

이를 반복하거나, 컨트롤러 메소드 파라미터 타입별 콜백 메소드를 가진 방문자(visitor)를 제공할 수 있습니다:

```java
HandlerMethodValidationException ex = ... ;

ex.visitResults(new HandlerMethodValidationException.Visitor() {

	@Override
	public void requestHeader(RequestHeader requestHeader, ParameterValidationResult result) {
		// ...
	}

	@Override
	public void requestParam(@Nullable RequestParam requestParam, ParameterValidationResult result) {
		// ...
	}

	@Override
	public void modelAttribute(@Nullable ModelAttribute modelAttribute, ParameterErrors errors) {
		// ...
	}

	@Override
	public void other(ParameterValidationResult result) {
		// ...
	}
});
```

---

### **학습 목표 제시**

이 챕터를 학습하고 나면, 다음을 이해하고 수행할 수 있게 됩니다:

1. **스프링 MVC에서의 두 가지 주요 검증 레벨 이해:** 메소드 인자 레벨 검증과 메소드 레벨 검증의 차이점, 각각 어떤 상황에서 적용되며 어떤 예외(`MethodArgumentNotValidException`, `HandlerMethodValidationException`)가 발생하는지 이해합니다.
2. **`@Valid`와 `@Validated` 어노테이션의 사용법 및 차이점 숙지:** Jakarta Bean Validation의 `@Valid`와 스프링의 `@Validated`를 사용하여 객체 및 메소드 파라미터에 대한 유효성 검사를 어떻게 적용하는지 이해하고, 이들의 미묘한 차이점을 인지합니다.
3. **검증 오류 처리 방법 이해:** `Errors` 또는 `BindingResult` 객체를 사용하여 컨트롤러 내에서 지역적으로 검증 오류를 처리하는 방법과, 예외 처리를 통해 전역적으로 또는 사용자 정의 방식으로 오류를 처리하는 방법을 이해합니다.
4. **`Validator` 구성 방법 인지:** `@InitBinder`를 통한 지역적 `Validator` 구성 또는 MVC 설정을 통한 전역적 `Validator` 구성 방법을 인지합니다. (클래스 레벨 `@Validated`와 AOP 프록시에 대한 내용은 심화 내용으로 간단히 인지만 합니다.)

---

### **핵심 개념 설명**

**왜 검증이 필요할까요?**

사용자로부터 입력을 받는 모든 애플리케이션은 그 입력값이 유효한지 확인해야 합니다. 예를 들어:

- 회원가입 시 이메일 형식이 올바른가?
- 게시글 작성 시 제목이 비어있지 않은가?
- 상품 주문 시 수량이 0보다 큰가?
- 나이 입력 시 음수나 너무 큰 값이 들어오지 않았는가?

이러한 검증을 제대로 하지 않으면, 잘못된 데이터가 시스템에 저장되어 데이터 무결성을 해치거나, 예기치 않은 오류를 발생시켜 애플리케이션의 안정성을 떨어뜨릴 수 있습니다. 심지어 보안 취약점으로 이어질 수도 있습니다.

**스프링 MVC의 검증 지원**

스프링 MVC는 **자바 빈 검증(Java Bean Validation, 현재는 Jakarta Bean Validation)** 표준을 기반으로 하는 강력한 내장 검증 기능을 제공합니다.

이를 통해 개발자는 어노테이션을 사용하여 간편하게 검증 규칙을 정의하고 적용할 수 있습니다.

**주요 검증 어노테이션 (Jakarta Bean Validation 표준):**

- `@NotNull`: 값이 null이 아니어야 함.
- `@NotEmpty`: (문자열, 컬렉션, 맵, 배열 등에 사용) null이 아니고 비어있지 않아야 함 (문자열의 경우 길이가 0보다 커야 함).
- `@NotBlank`: (문자열에 사용) null이 아니고, 공백 문자(whitespace)만으로 이루어져 있지 않아야 함.
- `@Size(min=, max=)`: 문자열, 컬렉션 등의 크기가 지정된 범위 내에 있어야 함.
- `@Min(value)`: 숫자 값이 지정된 최소값 이상이어야 함.
- `@Max(value)`: 숫자 값이 지정된 최대값 이하이어야 함.
- `@Email`: 문자열이 유효한 이메일 형식이어야 함.
- `@Pattern(regexp=)`: 문자열이 지정된 정규 표현식과 일치해야 함.
- 등등 매우 다양합니다.

**스프링 MVC에서의 검증 적용 레벨**

원문에서 설명한 것처럼, 스프링 MVC에서는 검증이 주로 두 가지 레벨에서 적용될 수 있습니다.

1. **메소드 인자 레벨 검증 (Argument-Level Validation):**
  - **대상:** 컨트롤러 메소드의 매개변수 중 `@ModelAttribute` (커맨드 객체), `@RequestBody` (요청 본문 객체), `@RequestPart` (멀티파트 요청의 일부) 어노테이션이 붙은 객체.
  - **트리거:** 해당 매개변수에 **Jakarta의 `@Valid`** 또는 **스프링의 `@Validated`** 어노테이션을 붙이면 활성화됩니다.
  - **조건:**
    - 바로 뒤에 `Errors` 또는 `BindingResult` 파라미터가 **없어야 합니다.** (만약 있다면, 예외가 바로 터지는 대신 `Errors` 객체에 오류 정보가 담기고 컨트롤러 메소드는 계속 실행됩니다.)
    - 그리고, 더 우선순위가 높은 "메소드 레벨 검증"이 필요하지 않아야 합니다.
  - **발생 예외:** 검증 실패 시 `MethodArgumentNotValidException` 발생. (이 예외는 주로 HTTP 400 Bad Request 응답으로 이어집니다.)
2. **메소드 레벨 검증 (Method-Level Validation):**
  - **대상:**
    - 컨트롤러 메소드의 일반 파라미터(객체가 아닌 단순 타입 등)에 직접 `@Min`, `@NotBlank` 같은 **제약 조건 어노테이션(`@Constraint` 어노테이션)이 붙어 있을 때**.
    - 또는, 메소드 자체(반환 값 검증을 위해)에 제약 조건 어노테이션이 붙어 있을 때.
  - **우선순위:** 이 "메소드 레벨 검증"이 필요하다고 판단되면, 위에서 설명한 "메소드 인자 레벨 검증"보다 우선적으로 적용됩니다 (이를 대체합니다).
  - **포함 범위:** 메소드 파라미터 자체의 제약 조건뿐만 아니라, 해당 파라미터가 객체이고 그 객체에 `@Valid`가 붙어있다면 그 객체 내부의 중첩된 제약 조건까지 모두 검증합니다.
  - **발생 예외:** 검증 실패 시 `HandlerMethodValidationException` 발생. (이 예외도 주로 HTTP 400 Bad Request 응답으로 이어집니다.)

**`@Valid` vs. `@Validated`**

- **`@Valid` (Jakarta Bean Validation):** 표준 자바 스펙입니다. 주로 객체 내부의 필드에 정의된 제약 조건들을 검증하도록 지시합니다 (중첩 검증).
- **`@Validated` (Spring):** `@Valid`의 기능을 포함하면서, 추가적으로 **검증 그룹(validation groups)**을 지정할 수 있는 스프링 고유의 어노테이션입니다. (검증 그룹은 특정 상황에서만 특정 제약 조건들을 검증하고 싶을 때 사용합니다. 예를 들어, "생성 시 검증 규칙"과 "수정 시 검증 규칙"을 다르게 가져가고 싶을 때.)

**검증 오류 처리 방법**

1. **`Errors` 또는 `BindingResult` 사용 (지역적 처리):**
  - `@Valid` 또는 `@Validated` 어노테이션이 붙은 매개변수 **바로 뒤에** `Errors` 또는 `BindingResult` 타입의 매개변수를 선언하면, 검증 오류가 발생해도 예외가 바로 터지지 않고, 대신 오류 정보가 이 `Errors` (또는 `BindingResult`) 객체에 담깁니다.
  - 컨트롤러 메소드 내에서 `errors.hasErrors()` 등을 호출하여 오류 발생 여부를 확인하고, 분기 처리를 할 수 있습니다. (예: 오류가 있으면 다시 폼 화면으로 돌려보내면서 오류 메시지 표시)
  - 이 방식은 "메소드 인자 레벨 검증"에서 주로 사용됩니다.
  - "메소드 레벨 검증"의 경우, 원문 설명처럼 **모든** 검증 오류가 `Errors`가 바로 뒤에 오는 파라미터에 대한 것이라면 컨트롤러 메소드가 호출되지만, 그렇지 않은 파라미터(예: `@RequestParam @Min(1) int age`)에서 오류가 발생하면 `HandlerMethodValidationException`이 터집니다.
2. **예외 처리 (전역적 또는 컨트롤러 레벨 처리):**
  - `MethodArgumentNotValidException`이나 `HandlerMethodValidationException`이 발생했을 때, 이를 처리하는 `@ExceptionHandler` 메소드를 컨트롤러 내에 만들거나, `@ControllerAdvice` 클래스에 만들어 전역적으로 처리할 수 있습니다.
  - 이를 통해 일관된 형식의 오류 응답(예: JSON 오류 메시지 객체)을 클라이언트에게 반환할 수 있습니다. (스프링 부트는 기본적으로 이러한 예외들을 적절한 HTTP 400 응답으로 변환해줍니다.)

**`Validator` 구성**

- **지역적 구성 (`@InitBinder`):** 이전 챕터에서 배운 `@InitBinder` 메소드 내에서 `WebDataBinder`의 `setValidator(Validator validator)` 메소드를 사용하여 특정 컨트롤러 또는 특정 커맨드 객체에만 적용될 `Validator`를 등록할 수 있습니다.
- **전역적 구성 (MVC Config):** `WebMvcConfigurer`를 구현한 설정 클래스에서 `getValidator()` 메소드를 오버라이드하여 애플리케이션 전반에 사용될 기본 `Validator`를 제공할 수 있습니다. (스프링 부트에서는 `spring-boot-starter-validation` 의존성이 있으면 `LocalValidatorFactoryBean`이 자동으로 구성됩니다.)

### **주요 용어 해설**

- **자바 빈 검증 (Java Bean Validation) / Jakarta Bean Validation:** 자바 객체의 유효성을 검증하기 위한 표준 스펙 및 API. 어노테이션 기반으로 검증 규칙을 정의합니다.
- **`@Valid`:** Jakarta Bean Validation 표준 어노테이션. 객체의 중첩된 필드에 대한 검증을 활성화합니다.
- **`@Validated`:** 스프링 프레임워크에서 제공하는 어노테이션. `@Valid`의 기능을 포함하며, 검증 그룹 지정 기능을 추가로 제공합니다.
- **제약 조건 어노테이션 (`@Constraint` 어노테이션):** `@NotNull`, `@Size`, `@Min` 등과 같이 실제 검증 규칙을 정의하는 어노테이션들.
- **`Errors` / `BindingResult`:** 스프링에서 데이터 바인딩 및 검증 과정에서 발생한 오류 정보를 담는 인터페이스. (`BindingResult`는 `Errors`를 상속받으며, 주로 폼 객체와 함께 사용될 때 더 많은 기능을 제공합니다.)
- **`MethodArgumentNotValidException`:** `@Valid` 또는 `@Validated`가 적용된 메소드 인자(주로 `@ModelAttribute`나 `@RequestBody` 객체)의 검증이 실패했을 때 발생하는 예외.
- **`HandlerMethodValidationException`:** 메소드 파라미터에 직접 적용된 제약 조건이나 메소드 자체의 반환 값 검증 등이 실패했을 때 발생하는 예외 (스프링 6.1부터 도입된 더 포괄적인 메소드 검증 예외).
- **검증 그룹 (Validation Groups):** 특정 상황(예: 객체 생성 시, 수정 시)에 따라 다른 검증 규칙들을 선택적으로 적용할 수 있게 하는 메커니즘. `@Validated` 어노테이션과 함께 사용됩니다.

### **코드 예제 및 분석**

**1. `@RequestBody` 객체 검증 (메소드 인자 레벨 검증) 및 `Errors` 사용**

`Item` 클래스 (검증 규칙 포함):

```java
import jakarta.validation.constraints.Min;
import jakarta.validation.constraints.NotBlank;
import jakarta.validation.constraints.NotNull;
import jakarta.validation.constraints.Size;

public class Item {
    @NotNull
    private Long id;

    @NotBlank(message = "Item name cannot be blank")
    @Size(min = 2, max = 50, message = "Item name must be between 2 and 50 characters")
    private String name;

    @Min(value = 1, message = "Quantity must be at least 1")
    private int quantity;

    // Getters and Setters
}

```

컨트롤러:

```java
import org.springframework.http.ResponseEntity;
import org.springframework.validation.BindingResult; // 또는 org.springframework.validation.Errors
import org.springframework.web.bind.annotation.PostMapping;
import org.springframework.web.bind.annotation.RequestBody;
import org.springframework.web.bind.annotation.RestController;
import jakarta.validation.Valid; // Jakarta Bean Validation 사용

@RestController
public class ItemController {

    @PostMapping("/items")
    public ResponseEntity<?> createItem(@Valid @RequestBody Item item, BindingResult bindingResult) {
        if (bindingResult.hasErrors()) {
            // 검증 오류가 있으면, 오류 정보와 함께 400 Bad Request 응답
            // (실제로는 오류 메시지를 좀 더 예쁘게 가공해서 보내는 것이 좋음)
            return ResponseEntity.badRequest().body(bindingResult.getAllErrors());
        }

        // 검증 통과: item 객체를 사용하여 아이템 생성 로직 수행
        System.out.println("Item created: " + item.getName());
        return ResponseEntity.ok("Item created successfully: " + item.getName());
    }
}

```

**코드 분석:**

- `@Valid @RequestBody Item item`: 클라이언트가 보낸 JSON 요청 본문을 `Item` 객체로 변환하고, `Item` 클래스에 정의된 검증 규칙(`@NotNull`, `@NotBlank`, `@Size`, `@Min`)에 따라 유효성 검사를 수행합니다.
- `BindingResult bindingResult`: `@Valid` 어노테이션 바로 뒤에 선언되어, 검증 오류가 발생하면 해당 오류 정보가 `bindingResult` 객체에 담깁니다. 예외가 즉시 발생하지 않습니다.
- `if (bindingResult.hasErrors())`: 오류 발생 여부를 확인하고, 오류가 있다면 `400 Bad Request`와 함께 오류 정보를 응답합니다.

**2. 메소드 파라미터 직접 검증 (메소드 레벨 검증)**

```java
import org.springframework.http.ResponseEntity;
import org.springframework.validation.annotation.Validated; // 스프링의 @Validated 사용 가능
import org.springframework.web.bind.annotation.GetMapping;
import org.springframework.web.bind.annotation.RequestParam;
import org.springframework.web.bind.annotation.RestController;
import jakarta.validation.constraints.Min;
import jakarta.validation.constraints.NotBlank;

@RestController
@Validated // 클래스 레벨 @Validated는 메소드 파라미터 검증을 위해 필요할 수 있음 (스프링 버전에 따라)
           // 하지만 스프링 6.1부터는 클래스 레벨 @Validated 없이도 메소드 파라미터 검증이 잘 동작.
           // 여기서는 설명을 위해 포함.
public class QueryParamValidationController {

    @GetMapping("/search")
    public ResponseEntity<String> searchItems(
            @NotBlank @RequestParam String keyword, // keyword는 비어있을 수 없음
            @Min(1) @RequestParam(defaultValue = "1") int page) { // page는 최소 1 이상

        // 이 메소드는 keyword와 page 파라미터가 위의 제약 조건을 통과해야만 실행됨.
        // 만약 통과하지 못하면 HandlerMethodValidationException 발생 (기본적으로 400 응답).
        return ResponseEntity.ok("Searching for: " + keyword + " on page " + page);
    }
}

```

**코드 분석:**

- `@NotBlank @RequestParam String keyword`: `keyword` 요청 파라미터는 반드시 존재하고 공백이 아닌 문자열이어야 합니다.
- `@Min(1) @RequestParam(defaultValue = "1") int page`: `page` 요청 파라미터는 최소 1이어야 합니다.
- 만약 클라이언트가 `/search?keyword=` (빈 문자열) 또는 `/search?keyword=test&page=0` 과 같이 유효하지 않은 요청을 보내면, 이 컨트롤러 메소드가 실행되기 전에 `HandlerMethodValidationException`이 발생하고, 기본적으로 HTTP 400 응답이 나갑니다.
- **클래스 레벨 `@Validated`:** 과거 버전의 스프링에서는 메소드 파라미터에 대한 직접적인 제약 조건 검증을 활성화하기 위해 컨트롤러 클래스에 `@Validated`를 붙여야 했습니다 (AOP 프록시를 통해 동작). 하지만 스프링 프레임워크 6.1부터는 이러한 클래스 레벨 어노테이션 없이도 메소드 파라미터 검증이 잘 지원됩니다. 원문에서는 "클래스 레벨 `@Validated` 어노테이션을 제거해야" 내장 지원을 활용할 수 있다고 언급하고 있습니다. 이는 버전별 차이를 인지해야 함을 시사합니다.

### **"왜?" 라는 질문에 대한 답변**

**왜 스프링은 이렇게 두 가지 레벨의 검증 방식을 제공할까요?**

1. **다양한 입력 형태 지원:**
  - **메소드 인자 레벨 검증:** 주로 복잡한 객체(커맨드 객체, `@RequestBody`로 받는 JSON/XML 객체) 내부의 여러 필드에 대한 검증에 적합합니다. 객체 지향적인 방식으로 검증 규칙을 모델 클래스 자체에 정의할 수 있습니다.
  - **메소드 레벨 검증:** 단순한 요청 파라미터(`@RequestParam`, `@PathVariable` 등)나 헤더 값(`@RequestHeader`) 등 객체가 아닌 개별 값들에 대한 직접적인 검증에 유용합니다. 또는, 여러 파라미터 간의 관계를 검증하거나 메소드 자체의 조건을 검증하는 데 사용될 수 있습니다 (더 고급 사용법).
2. **유연성 및 제어:**
  - `Errors`/`BindingResult`를 사용하면 검증 오류를 프로그램적으로 처리하고 사용자에게 친화적인 피드백을 제공할 수 있는 유연성을 줍니다.
  - 예외 기반 처리는 공통적인 오류 처리 로직을 중앙에서 관리(예: `@ControllerAdvice`)할 수 있게 해줍니다.
3. **표준 준수 및 확장성:** Jakarta Bean Validation 표준을 따르므로, 다른 프레임워크나 라이브러리와의 호환성이 좋고, 커스텀 제약 조건 어노테이션을 만들어 확장하기도 용이합니다.

**`MethodArgumentNotValidException`과 `HandlerMethodValidationException`은 왜 구분되어 있을까요?**

- **`MethodArgumentNotValidException`:** 주로 `@Valid`나 `@Validated`를 통해 단일 "객체" 인자의 검증이 실패했을 때 발생합니다. 이 예외는 그 "객체"에 대한 오류 정보(`BindingResult`)를 가지고 있습니다.
- **`HandlerMethodValidationException`:** (스프링 6.1부터 강조) 메소드 시그니처 자체에 정의된 여러 제약 조건(개별 파라미터, 반환 값 등) 중 하나라도 실패하면 발생합니다. 이 예외는 여러 파라미터에 대한 검증 결과를 가질 수 있으며, `ParameterValidationResult` 목록을 통해 각 파라미터별 오류를 확인할 수 있도록 더 구조화되어 있습니다.
  두 예외는 발생 원인과 담고 있는 오류 정보의 범위에 차이가 있지만, 클라이언트에게는 대체로 "잘못된 요청(400)"으로 전달된다는 점에서 유사하게 처리될 수 있습니다. 스프링은 이 둘을 거의 동일한 코드로 처리할 수 있도록 유사하게 설계했다고 언급합니다.

### **주의사항 및 Best Practice**

1. **`@Valid`는 중첩 검증용:** `@Valid` 자체는 제약 조건이 아닙니다. 객체 필드에 붙여서 "이 객체 내부도 검증해줘"라는 의미로 사용됩니다. `@Valid`가 붙은 객체 파라미터 자체의 null 여부를 검증하려면 `@NotNull @Valid MyObject obj`처럼 `@NotNull` 같은 제약 조건 어노테이션을 함께 사용해야 메소드 레벨 검증이 트리거될 수 있습니다.
2. **`Errors`/`BindingResult` 위치:** `@Valid`/`@Validated`로 검증되는 매개변수 **바로 뒤에** 선언해야 합니다. 순서가 중요합니다.
3. **예외 처리 전략:** 애플리케이션의 요구사항에 맞게 `BindingResult`를 사용한 지역적 오류 처리와 `@ExceptionHandler`(또는 `@ControllerAdvice`)를 사용한 전역적 오류 처리 중 적절한 방식을 선택하거나 조합해야 합니다. 일관된 오류 응답 형식을 제공하는 것이 중요합니다.
4. **메시지 국제화 (`MessageSource`):** 검증 오류 메시지를 사용자 친화적으로 제공하고 다국어 지원을 하려면, `MessageSource`를 설정하고 프로퍼티 파일에 오류 메시지를 정의하는 것이 좋습니다. (원문에서 "Error Responses" 섹션 참고 언급)
5. **버전별 차이 인지 (특히 메소드 레벨 검증):** 스프링 프레임워크 버전에 따라 메소드 레벨 검증의 동작 방식이나 클래스 레벨 `@Validated`의 필요 여부에 차이가 있을 수 있습니다. (스프링 6.1에서 개선됨)

### **이전 학습 내용과의 연관성**

- **`@ModelAttribute`, `@RequestBody`, `@RequestPart`:** 이 어노테이션들로 바인딩된 객체들이 주로 "메소드 인자 레벨 검증"의 대상이 됩니다.
- **`@RequestParam`, `@PathVariable`, `@RequestHeader`:** 이 어노테이션들로 받는 개별 파라미터들이 주로 "메소드 레벨 검증"의 대상이 됩니다.
- **`@InitBinder`:** `WebDataBinder`에 커스텀 `Validator`를 등록하여 특정 컨트롤러나 커맨드 객체에 대한 검증 로직을 커스터마이징할 수 있습니다.
- **`@ControllerAdvice`, `@ExceptionHandler` (미리보기):** 검증 실패 시 발생하는 예외들(`MethodArgumentNotValidException`, `HandlerMethodValidationException`)을 전역적으로 처리하여 일관된 오류 응답을 제공하는 데 사용됩니다.

---
