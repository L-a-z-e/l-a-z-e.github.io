---
title: Spring Web MVC - Annotated Controllers (Type Conversion)
description: 
author: laze
date: 2025-05-27 00:00:01 +0900
categories: [Dev, SpringWebMVC]
tags: [SpringWebMVC]
---
## 타입 변환 (Type Conversion)

[Reactive 스택의 동일 기능 보기](https://docs.spring.io/spring-framework/docs/current/reference/html/web-reactive.html#webflux-ann-arguments-type-conversion)

문자열 기반의 요청 입력(예: `@RequestParam`, `@RequestHeader`, `@PathVariable`, `@MatrixVariable`, `@CookieValue` 애노테이션이 붙은 인자)을 나타내는 일부 애노테이션 컨트롤러 메서드 인자는, 해당 인자가 `String` 이외의 타입으로 선언된 경우 타입 변환이 필요할 수 있습니다.

이러한 경우, 설정된 변환기에 따라 타입 변환이 자동으로 적용됩니다.

기본적으로 단순 타입(int, long, Date 등)이 지원됩니다.

`WebDataBinder`([DataBinder](https://docs.spring.io/spring-framework/docs/current/reference/html/core.html#validation-databinding) 참조)를 통해 또는 `FormattingConversionService`에 `Formatter`를 등록하여 타입 변환을 사용자 정의할 수 있습니다. ([스프링 필드 포맷팅](https://docs.spring.io/spring-framework/docs/current/reference/html/core.html#format) 참조).

타입 변환에서 실제적인 이슈 중 하나는 빈 문자열 소스 값의 처리입니다. 이러한 값은 타입 변환 결과 `null`이 되면 누락된 것으로 처리됩니다.

이는 `Long`, `UUID` 및 기타 대상 타입의 경우에 해당될 수 있습니다.

`null` 주입을 허용하려면 인수 애노테이션의 `required` 플래그를 사용하거나 인수를 `@Nullable`로 선언하십시오.

5.3 버전부터는 타입 변환 후에도 null이 아닌 인수가 강제됩니다.

만약 핸들러 메서드가 `null` 값도 수용하도록 하려면, 인수를 `@Nullable`로 선언하거나 해당 `@RequestParam` 등의 애노테이션에서 `required=false`로 표시하십시오.

이는 모범 사례이며 5.3 업그레이드 시 발생하는 회귀 문제에 권장되는 해결책입니다.

또는, 예를 들어 필수 `@PathVariable`의 경우 결과적으로 발생하는 `MissingPathVariableException`을 특별히 처리할 수도 있습니다.

변환 후 `null` 값은 원래의 빈 값처럼 처리되므로 해당 `Missing…Exception` 변형 예외들이 발생합니다.

---

**✨ 이 챕터의 학습 목표 ✨**

1. **스프링 MVC에서 요청 파라미터가 컨트롤러 메서드의 인자로 전달될 때, 어떻게 자동으로 타입이 변환되는지 이해합니다.** (예: URL의 문자열 "123"이 숫자 `int id`로 변환되는 과정)
2. **빈 문자열이나 `null` 값이 전달될 경우, 스프링 MVC가 이를 어떻게 처리하는지 배우고, `@Nullable`과 `required` 속성을 사용하여 이를 제어하는 방법을 익힙니다.**
3. **기본적인 타입 변환 외에, 사용자 정의 타입 변환이 필요할 때 `WebDataBinder` 또는 `Formatter`를 사용할 수 있다는 점을 인지합니다.** (이번 챕터에서 깊게 다루진 않지만, 개념을 알아둡니다.)

### 핵심 개념 설명

우리가 웹 애플리케이션을 만들 때, 사용자가 웹 브라우저를 통해 서버로 보내는 정보는 대부분 **문자열(String)** 형태입니다.

예를 들어, 사용자가 게시글의 ID를 URL에 `/boards/123` 이렇게 넣어서 요청한다고 해봅시다.

여기서 `123`은 우리 눈에는 숫자지만, HTTP 프로토콜을 통해 서버로 전달될 때는 "123"이라는 문자열로 전달됩니다.

하지만 우리 자바 코드에서는 이 `123`을 문자열 "123"이 아니라 숫자 `int` 타입의 `id` 변수로 받고 싶을 수 있습니다.

그래야 숫자 연산을 하거나 데이터베이스에서 해당 ID를 가진 데이터를 찾기가 편하겠죠.

바로 이럴 때 **타입 변환(Type Conversion)**이 일어납니다.

스프링 MVC는 똑똑하게도, 컨트롤러 메서드의 파라미터 타입을 보고 "아, 개발자가 이 문자열 '123'을 `int` 타입으로 받고 싶어 하는구나!" 라고 알아채고 자동으로 변환해줍니다.

**비유를 들어볼까요?**

- 여러분이 해외여행을 갔는데, 현지 상점에서 물건값을 달러($)로 알려줍니다. 하지만 여러분 지갑에는 원화(₩)만 있어요. 이때 환전소에 가서 달러를 원화로 바꾸는 과정이 필요하죠? 여기서 **환전소**가 스프링 MVC의 **타입 변환기(Converter)** 역할이고, **달러**는 사용자가 보낸 **문자열 데이터**, **원화**는 우리 자바 코드에서 사용할 **실제 타입**이라고 생각할 수 있습니다.

스프링 MVC는 기본적인 타입들( `int`, `long`, `double`, `boolean`, `Date` 등)에 대해서는 별도의 설정 없이도 알아서 잘 변환해줍니다.

이것이 가능한 이유는 스프링 내부에 이미 이러한 기본 타입들에 대한 변환기가 등록되어 있기 때문입니다.

하지만 만약 우리가 직접 만든 특수한 객체 타입으로 변환하고 싶다면 (예: "2023-03-15" 라는 문자열을 우리가 만든 `LocalDate` 비슷한 `MyDate` 객체로 변환),

그때는 우리가 직접 "이런 문자열은 이렇게 `MyDate` 객체로 바꿔줘" 라고 알려주는 **커스텀 변환기(Custom Converter)**나 **포맷터(Formatter)**를 만들어 등록해야 합니다.

### 주요 용어 해설

- **`@RequestParam`**: HTTP 요청 파라미터를 컨트롤러 메서드의 인자로 받을 때 사용하는 애노테이션입니다. 예를 들어, URL이 `/search?keyword=spring` 일 때, `keyword` 파라미터의 값 "spring"을 받습니다.

    ```java
    @GetMapping("/search")
    public String search(@RequestParam String keyword) {
        // keyword 변수에는 "spring"이 담겨있습니다.
        return "searchResults";
    }
    ```

- **`@RequestHeader`**: HTTP 요청 헤더 값을 컨트롤러 메서드의 인자로 받을 때 사용합니다.

    ```java
    @GetMapping("/info")
    public String getInfo(@RequestHeader("User-Agent") String userAgent) {
        // userAgent 변수에는 요청을 보낸 클라이언트의 정보 (예: "Chrome/90.0...")가 담깁니다.
        return "infoPage";
    }
    ```

- **`@PathVariable`**: URL 경로의 일부를 동적으로 변하는 값으로 받아올 때 사용합니다. RESTful API에서 자주 사용됩니다.

    ```java
    @GetMapping("/users/{userId}") // URL: /users/123
    public String getUserProfile(@PathVariable Long userId) {
        // userId 변수에는 숫자 123이 담깁니다. (문자열 "123"에서 Long 타입으로 자동 변환)
        return "userProfile";
    }
    ```

- **`@MatrixVariable`**: 경로 변수(Path Variable) 내에 키-값 쌍으로 데이터를 전달할 때 사용합니다. (세미콜론 `;`으로 구분) 잘 사용되지는 않지만 이런 것도 있다는 정도로 알아두세요.
  예: `/cars/Ford;color=blue;year=2023`
- **`@CookieValue`**: HTTP 요청에 포함된 쿠키 값을 컨트롤러 메서드의 인자로 받을 때 사용합니다.

    ```java
    @GetMapping("/welcome")
    public String welcome(@CookieValue(value = "username", required = false) String username) {
        if (username != null) {
            // username 쿠키가 있다면 해당 값을 사용
        }
        return "welcomePage";
    }
    ```

- **`WebDataBinder`**: HTTP 요청 파라미터를 자바 객체에 바인딩(연결)하고 유효성 검사를 수행하는 역할을 합니다. 타입 변환 규칙을 커스터마이징할 때 사용됩니다.
- **`Formatter`**: 특정 타입의 객체를 문자열로 변환하거나, 문자열을 특정 타입의 객체로 변환하는 역할을 합니다. `Converter`보다 더 풍부한 기능을 제공하며, 주로 사용자 인터페이스(UI)에 표시될 값의 형식을 지정하거나, 사용자로부터 입력받은 문자열을 특정 객체 타입으로 변환할 때 사용합니다. (예: 날짜를 "yyyy-MM-dd" 형식으로 보여주거나, 이 형식의 문자열을 `Date` 객체로 변환)
- **`FormattingConversionService`**: `Formatter`와 `Converter`를 등록하고 관리하는 서비스입니다. 스프링은 이 서비스를 통해 적절한 변환기를 찾아 타입 변환을 수행합니다.
- **`@Nullable`**: 해당 파라미터가 `null` 값을 가질 수 있음을 나타내는 애노테이션입니다. (Java 8의 `Optional<T>`와 비슷한 역할을 합니다.)
- **`required` 속성**: `@RequestParam(required = false)` 와 같이 사용되며, 해당 요청 파라미터가 필수가 아님을 나타냅니다. `required = true` (기본값)이면 해당 파라미터가 요청에 없거나 빈 문자열이면 예외가 발생합니다.

### 코드 예제 및 분석

원문에는 직접적인 코드 예제가 명시되어 있지는 않지만, 설명에 기반하여 예제를 만들어보겠습니다.

**예제 1: 기본 타입 변환 (문자열 -> 숫자)**

```java
import org.springframework.stereotype.Controller;
import org.springframework.web.bind.annotation.GetMapping;
import org.springframework.web.bind.annotation.PathVariable;
import org.springframework.web.bind.annotation.RequestParam;
import org.springframework.web.bind.annotation.ResponseBody;

@Controller
public class TypeConversionController {

    // 예시 1: @RequestParam으로 숫자 받기
    // 요청 URL: /calculate?num1=10&num2=20
    @GetMapping("/calculate")
    @ResponseBody // 결과를 HTTP 응답 본문에 바로 쓰기 위함 (뷰를 사용하지 않음)
    public String calculate(@RequestParam int num1, @RequestParam int num2) {
        // num1에는 숫자 10, num2에는 숫자 20이 들어옴 (문자열 "10", "20"에서 자동 변환)
        int sum = num1 + num2;
        return "Sum: " + sum; // "Sum: 30" 반환
    }

    // 예시 2: @PathVariable로 숫자 받기
    // 요청 URL: /product/777
    @GetMapping("/product/{productId}")
    @ResponseBody
    public String getProduct(@PathVariable long productId) {
        // productId에는 숫자 777L이 들어옴 (문자열 "777"에서 자동 변환)
        return "Product ID: " + productId; // "Product ID: 777" 반환
    }
}
```

**분석:**

- `calculate` 메서드:
  - `@RequestParam int num1`, `@RequestParam int num2`: URL의 쿼리 파라미터 `num1`과 `num2`의 값을 받습니다. 스프링은 문자열 "10"과 "20"을 각각 `int` 타입으로 자동 변환하여 `num1`과 `num2` 변수에 할당합니다.
- `getProduct` 메서드:
  - `@PathVariable long productId`: URL 경로 `/product/{productId}`에서 `{productId}` 부분의 값을 받습니다. 만약 `/product/777`로 요청이 오면, 문자열 "777"이 `long` 타입으로 자동 변환되어 `productId` 변수에 할당됩니다.

**예제 2: 빈 문자열과 `null` 처리**

원문에서 언급된 "빈 문자열 소스 값의 처리" 부분을 이해하기 위한 예제입니다.

```java
import org.springframework.lang.Nullable; // @Nullable 애노테이션
import org.springframework.stereotype.Controller;
import org.springframework.web.bind.annotation.GetMapping;
import org.springframework.web.bind.annotation.RequestParam;
import org.springframework.web.bind.annotation.ResponseBody;

@Controller
public class NullableParamController {

    // 1. 기본: required=true (기본값)
    // 요청 URL: /process?value=123 -> value: 123
    // 요청 URL: /process?value=   -> MissingServletRequestParameterException 발생! (빈 문자열도 오류)
    // 요청 URL: /process          -> MissingServletRequestParameterException 발생! (파라미터 없음)
    @GetMapping("/process")
    @ResponseBody
    public String processRequired(@RequestParam Long value) { // Long 타입은 빈 문자열을 null로 변환 시도
        return "Value: " + value;
    }

    // 2. required=false 사용
    // 요청 URL: /process-optional?value=123 -> value: 123
    // 요청 URL: /process-optional?value=   -> value: null (Long 타입은 빈 문자열을 null로 변환)
    // 요청 URL: /process-optional          -> value: null (파라미터가 없어도 null)
    @GetMapping("/process-optional")
    @ResponseBody
    public String processOptional(@RequestParam(required = false) Long value) {
        return "Optional Value: " + value;
    }

    // 3. @Nullable 사용 (Spring 5.0 이상 권장)
    // 요청 URL: /process-nullable?value=123 -> value: 123
    // 요청 URL: /process-nullable?value=   -> value: null (Long 타입은 빈 문자열을 null로 변환)
    // 요청 URL: /process-nullable          -> MissingServletRequestParameterException 발생!
    //                                     (파라미터 자체가 없는 경우는 @Nullable로 커버 안됨, required=false 필요)
    @GetMapping("/process-nullable")
    @ResponseBody
    public String processNullable(@RequestParam @Nullable Long value) {
        return "Nullable Value: " + value;
    }

    // 4. required=false 와 @Nullable 함께 사용 (가장 유연)
    // 요청 URL: /process-flexible?value=123 -> value: 123
    // 요청 URL: /process-flexible?value=   -> value: null
    // 요청 URL: /process-flexible          -> value: null
    @GetMapping("/process-flexible")
    @ResponseBody
    public String processFlexible(@RequestParam(required = false) @Nullable Long value) {
        return "Flexible Value: " + value;
    }
}
```

**분석:**

- **`processRequired` 메서드**: `Long value`는 기본적으로 `required=true`입니다.
  - 만약 `?value=` (빈 문자열) 또는 파라미터가 아예 없는 경우, `Long` 타입으로 변환 시 `null`이 되려고 하는데, `required=true`이기 때문에 스프링은 이를 "필수 파라미터가 누락되었다"고 판단하여 `MissingServletRequestParameterException` 예외를 발생시킵니다. (원문: "Such a value is treated as missing if it becomes null as a result of type conversion.")
- **`processOptional` 메서드**: `@RequestParam(required = false)`
  - `required = false`로 설정하면, 파라미터가 없거나 빈 문자열일 때 예외가 발생하지 않고 `value` 변수에 `null`이 할당됩니다.
  - `Long` 타입은 빈 문자열을 `null`로 변환합니다. (원문: "This can be the case for Long, UUID, and other target types.")
- **`processNullable` 메서드**: `@RequestParam @Nullable Long value`
  - `@Nullable`은 해당 파라미터가 `null`을 가질 수 있음을 명시적으로 나타냅니다.
  - `?value=` (빈 문자열)인 경우, `Long` 타입으로 변환 시 `null`이 되고, `@Nullable`이 있으므로 `value`에 `null`이 할당됩니다.
  - 하지만 파라미터 자체가 없는 경우 (`/process-nullable` 요청)에는 여전히 `MissingServletRequestParameterException`이 발생합니다. `@Nullable`은 "파라미터는 존재하지만 그 값이 `null`일 수 있다"는 의미이지, "파라미터 자체가 없어도 된다"는 의미는 아니기 때문입니다.
- **`processFlexible` 메서드**: `@RequestParam(required = false) @Nullable Long value`
  - `required = false`와 `@Nullable`을 함께 사용하면, 파라미터가 없거나, 빈 문자열이거나, 유효한 값이거나 모두 수용 가능하며, 없는 경우와 빈 문자열인 경우는 `null`로 처리됩니다. 이것이 가장 유연한 방법입니다.
  - 원문에서 "As of 5.3, non-null arguments will be enforced even after type conversion. If your handler method intends to accept a null value as well, either declare your argument as `@Nullable` or mark it as `required=false`... This is a best practice..." 라고 언급된 부분이 이와 관련됩니다. 즉, `null`을 허용하고 싶다면 `@Nullable`이나 `required=false`를 명시하라는 것입니다.

### "왜?" 라는 질문에 대한 답변

**Q: 왜 문자열로만 받지 않고 굳이 다른 타입으로 변환해야 하나요?**

A: 몇 가지 중요한 이유가 있습니다.

1. **타입 안정성(Type Safety):** 개발자가 의도한 타입으로 데이터를 사용함으로써, 런타임에 발생할 수 있는 타입 관련 오류를 컴파일 시점이나 애플리케이션 시작 시점에 미리 방지할 수 있습니다. 예를 들어, 숫자여야 하는 값을 문자열로 다루다가 실수로 문자열 연산을 해버리는 등의 오류를 막을 수 있습니다.
2. **코드 가독성 및 유지보수성 향상:** `int age` 라고 선언하면, 이 변수가 나이를 나타내는 숫자라는 것을 명확히 알 수 있습니다. 만약 `String ageStr` 이라고 하고 매번 `Integer.parseInt(ageStr)` 같은 코드를 반복한다면 코드가 지저분해지고 이해하기 어려워집니다.
3. **비즈니스 로직 처리 용이:** 숫자 계산, 날짜 비교 등 각 타입에 맞는 연산을 직접 수행할 수 있습니다. 문자열 상태로는 이러한 로직을 구현하기 번거롭습니다.
4. **프레임워크 기능 활용:** 스프링 MVC는 타입 변환뿐만 아니라 유효성 검증(Validation) 같은 기능도 제공합니다. 이러한 기능들은 특정 타입으로 변환된 후에 동작하는 경우가 많습니다.

**Q: 스프링은 어떻게 이 많은 타입 변환을 자동으로 해줄 수 있는 건가요?**

A: 스프링 내부에는 `ConversionService`라는 것이 있고, 이 서비스 안에는 수많은 기본 `Converter` (변환기)들이 미리 등록되어 있습니다. 예를 들어 `String`을 `Integer`로, `String`을 `Date`로 바꾸는 변환기들이죠. 스프링은 컨트롤러 메서드의 파라미터 타입을 보고, 입력된 문자열을 해당 타입으로 변환할 수 있는 적절한 `Converter`를 찾아 실행합니다. 만약 적절한 변환기가 없다면 에러가 발생하겠죠.

**Q: 빈 문자열이 `null`로 처리되는 경우가 있고 안되는 경우가 있는 것 같은데, 왜 그런가요? (예: `String` vs `Long`)**

A: 좋은 질문입니다! 이것은 각 타입 변환기가 빈 문자열을 어떻게 해석하도록 설계되었는지에 따라 다릅니다.

- **`String` 타입:** 빈 문자열(`""`)은 그 자체로 유효한 `String` 값입니다. 따라서 `String` 타입 파라미터에 빈 문자열이 오면, `null`이 아니라 빈 문자열 `""`로 그대로 전달됩니다.
- **`Integer`, `Long`, `UUID` 등:** 이러한 숫자나 객체 타입의 경우, 빈 문자열은 해당 타입으로 표현할 수 있는 유효한 값이 아닙니다. (예: 빈 문자열은 숫자가 아니죠.) 그래서 스프링의 기본 변환기들은 이런 타입들에 대해 빈 문자열을 `null`로 취급하는 경향이 있습니다. "이건 숫자로 바꿀 수 없으니, 값이 없는 것(null)으로 보자" 라는 논리입니다.

그래서 원문에서도 "This can be the case for Long, UUID, and other target types." 라고 언급하며, 이런 타입들은 빈 문자열이 `null`로 변환될 수 있으니 주의하라고 알려주는 것입니다.

### 주의사항 및 Best Practice

1. **`required=false` 와 `@Nullable`의 차이 명확히 이해하기:**
  - `@RequestParam(required = false)`: 파라미터 자체가 요청에 존재하지 않아도 예외가 발생하지 않고, 해당 변수에는 `null` (또는 기본 타입의 경우 기본값)이 할당됩니다. 빈 문자열이 전달될 경우, 해당 타입 변환기의 정책에 따라 `null` 또는 다른 값으로 변환됩니다.
  - `@RequestParam @Nullable Type param`: 파라미터는 반드시 요청에 존재해야 하지만 (없으면 `MissingServletRequestParameterException` 발생 가능), 그 값이 `null`일 수 있음을 나타냅니다. 빈 문자열이 전달되어 `null`로 변환되는 경우를 수용합니다.
  - **Best Practice:** `null` 값을 허용하는 파라미터라면 `@Nullable`을 명시적으로 사용하고, 파라미터 자체가 선택 사항이라면 `required = false`를 사용하는 것이 좋습니다. 둘 다 필요한 경우 함께 사용할 수 있습니다. (원문의 "As of 5.3, non-null arguments will be enforced... either declare your argument as `@Nullable` or mark it as `required=false`" 참고)
2. **기본 타입(Primitive Type) vs 참조 타입(Reference Type) 사용 시 `null` 처리 주의:**
  - `int`, `long`, `boolean` 같은 기본 타입은 `null`을 가질 수 없습니다. 만약 `required=false`이고 파라미터가 전달되지 않으면, `int`는 `0`, `boolean`은 `false` 같은 기본값이 할당되려고 시도하다가 에러가 발생할 수 있습니다 (특히 파라미터가 아예 없는 경우).
  - `null` 가능성을 다루려면 `Integer`, `Long`, `Boolean` 같은 래퍼(Wrapper) 클래스 타입을 사용하는 것이 좋습니다. 이들은 `null`을 가질 수 있습니다.

    ```java
    // 좋지 않은 예 (파라미터 없을 때 문제 발생 가능성)
    // @RequestParam(required = false) int count;
    
    // 좋은 예
    @RequestParam(required = false) Integer count;
    ```

3. **커스텀 타입 변환이 필요할 경우:**
  - 문자열 "true", "false" 외에 "Y", "N"을 `boolean`으로 변환하고 싶거나, "20230315" 같은 문자열을 `LocalDate`로 변환하고 싶다면, 직접 `Converter`나 `Formatter`를 만들어 등록해야 합니다. 이는 나중에 더 자세히 다룰 내용입니다.
4. **예외 처리:**
  - 필수 파라미터가 누락되어 `MissingServletRequestParameterException`이 발생하거나, 타입 변환 자체가 실패하여 `MethodArgumentTypeMismatchException` 등이 발생할 수 있습니다. 이러한 예외들을 적절히 처리하여 사용자에게 친절한 오류 메시지를 보여주는 것이 좋습니다. (예: `@ExceptionHandler` 사용)
  - 원문에서도 `@PathVariable`이 `null`로 변환되면 `MissingPathVariableException`이 발생할 수 있다고 언급합니다.
