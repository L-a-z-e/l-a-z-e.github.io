---
title: Spring Web MVC - Annotated Controllers (@RequestBody)
description: 
author: laze
date: 2025-06-11 00:00:01 +0900
categories: [Dev, SpringWebMVC]
tags: [SpringWebMVC]
---
### @RequestBody

> Reactive 스택에서의 동일 기능은 해당 문서를 참고하세요.
>

`@RequestBody` 어노테이션을 사용하면 요청 본문(request body)을 `HttpMessageConverter`를 통해 읽어와서 `Object`로 역직렬화(deserialized)할 수 있습니다.

다음 예제는 `@RequestBody` 인자를 사용합니다:

**Java**

```java
@PostMapping("/accounts")
public void handle(@RequestBody Account account) { // 요청 본문을 Account 객체로 변환
	// ...
}
```

MVC 설정의 메시지 컨버터(Message Converters) 옵션을 사용하여 메시지 변환을 구성하거나 사용자 정의할 수 있습니다.

폼 데이터(Form data)는 `@RequestBody`가 아니라 `@RequestParam`을 사용하여 읽어야 합니다.

`@RequestBody`는 항상 안정적으로 사용할 수 없는데, 이는 서블릿 API에서 요청 파라미터 접근이 요청 본문을 파싱하도록 유발하며, 한번 파싱된 요청 본문은 다시 읽을 수 없기 때문입니다.

`@RequestBody`는 `jakarta.validation.Valid` 또는 스프링의 `@Validated` 어노테이션과 함께 사용할 수 있으며, 둘 다 표준 빈 검증(Standard Bean Validation)이 적용되도록 합니다.

기본적으로 검증 오류는 `MethodArgumentNotValidException`을 발생시키며, 이는 400 (BAD_REQUEST) 응답으로 변환됩니다.

또는, 다음 예제와 같이 `Errors` 또는 `BindingResult` 인자를 통해 컨트롤러 내에서 지역적으로 검증 오류를 처리할 수 있습니다:

**Java**

```java
@PostMapping("/accounts")
public String handle(@Valid @RequestBody Account account, Errors errors) { // Account 객체에 대한 검증 수행
	// ... errors 객체를 통해 검증 오류 확인
}
```

만약 다른 파라미터에 `@Constraint` 어노테이션이 있어 메소드 검증이 적용되면, 대신 `HandlerMethodValidationException`이 발생합니다.

---

### **학습 목표 제시**

이 챕터를 학습하고 나면, 학생은 다음을 이해하고 수행할 수 있게 됩니다:

1. **`@RequestBody` 어노테이션의 핵심 역할 이해:** HTTP 요청 본문 전체를 특정 자바 객체로 변환(역직렬화)하는 `@RequestBody`의 기능과 주된 사용 사례(특히 JSON/XML 기반의 API)를 이해합니다.
2. **`HttpMessageConverter`와의 관계 파악:** `@RequestBody`가 요청 본문을 객체로 변환할 때 `HttpMessageConverter`가 어떤 역할을 하는지, 그리고 이 변환 과정을 어떻게 커스터마이징할 수 있는지 개념적으로 이해합니다.
3. **`@RequestParam`과의 차이점 및 사용 구분 명확화:** `@RequestBody`와 `@RequestParam`이 각각 어떤 종류의 요청 데이터를 처리하는 데 적합한지, 그리고 왜 폼 데이터 처리에 `@RequestBody`를 사용하면 안 되는지 그 이유를 이해합니다.
4. **`@RequestBody`와 데이터 검증 연동 방법 숙지:** `@RequestBody`로 받은 객체에 대해 `@Valid` 또는 `@Validated` 어노테이션을 사용하여 데이터 유효성 검사를 수행하고, 오류를 처리하는 방법을 이해합니다.

---

### **핵심 개념 설명**

**`@RequestBody`란 무엇일까요?**

우리가 지금까지 배운 `@RequestParam`, `@RequestHeader`, `@CookieValue`, 커맨드 객체 등은 주로 HTTP 요청의 파라미터(URL 쿼리 스트링, 폼 데이터)나 헤더, 쿠키 등에서 특정 값들을 추출하거나 바인딩하는 방식이었습니다.

하지만 클라이언트가 서버로 데이터를 보낼 때, 특히 REST API와 같이 프로그램 간 통신을 할 때는, 요청 파라미터 형태가 아니라 **HTTP 요청 본문(request body)에 직접 데이터를 담아 보내는 경우**가 많습니다.

이때 데이터 형식은 주로 **JSON**이나 **XML**과 같은 구조화된 텍스트 형식을 사용합니다.

`@RequestBody` 어노테이션은 바로 이렇게 **HTTP 요청 본문에 담겨 오는 전체 내용을 통째로 읽어서, 지정된 자바 객체 타입으로 변환(역직렬화, deserialization)해주는 역할**을 합니다.

**비유:**

여러분이 해외에서 소포(HTTP 요청)를 받는다고 상상해봅시다.

- **@RequestParam, 커맨드 객체 방식:** 소포 겉면의 송장(요청 파라미터, 헤더)에 적힌 발신인 이름, 주소, 내용물 종류(카테고리) 등을 보고 각각 필요한 정보를 얻는 것과 비슷합니다.
- **@RequestBody 방식:** 소포를 통째로 받아서 열어보니, 그 안에 **하나의 완성된 제품 설명서(JSON 또는 XML 데이터)**가 들어있고, 여러분은 그 설명서를 보고 **완제품 모형(자바 객체)**을 조립하는 것과 같습니다. 요청 본문 전체가 하나의 의미 있는 데이터 덩어리인 셈이죠.

예를 들어, 클라이언트가 다음과 같은 JSON 데이터를 요청 본문에 담아 `/users` 라는 URL로 POST 요청을 보냈다고 합시다.

**요청 본문 (JSON):**

```json
{
  "username": "john.doe",
  "email": "john.doe@example.com",
  "age": 30
}
```

서버의 스프링 MVC 컨트롤러에서는 다음과 같이 `@RequestBody`를 사용하여 이 JSON 데이터를 `User` 자바 객체로 바로 받을 수 있습니다.

**User 자바 클래스:**

```java
public class User {
    private String username;
    private String email;
    private int age;
    // Getters and Setters
}
```

**컨트롤러 메소드:**

```java
@PostMapping("/users")
public ResponseEntity<String> createUser(@RequestBody User user) {
    // 이제 user 객체에는 JSON 데이터가 매핑되어 있습니다.
    // user.getUsername() 은 "john.doe"
    // user.getEmail() 은 "john.doe@example.com"
    // user.getAge() 는 30
    userService.save(user);
    return ResponseEntity.status(HttpStatus.CREATED).body("User created successfully");
}
```

마법 같죠? `@RequestBody` 어노테이션 하나로 이 모든 변환이 일어납니다.

**핵심 도우미: `HttpMessageConverter`**

이 마법의 뒤에는 **`HttpMessageConverter`** 라는 중요한 인터페이스와 그 구현체들이 있습니다.

- `HttpMessageConverter`는 HTTP 요청 본문을 자바 객체로 변환하거나 (역직렬화, `@RequestBody`의 경우), 자바 객체를 HTTP 응답 본문으로 변환하는 (직렬화, `@ResponseBody`의 경우) 역할을 담당합니다.
- 스프링 MVC에는 다양한 종류의 `HttpMessageConverter`가 기본적으로 등록되어 있습니다.
  - **JSON 처리:** `MappingJackson2HttpMessageConverter` (Jackson 라이브러리 사용), `GsonHttpMessageConverter` (Gson 라이브러리 사용) 등
  - **XML 처리:** `Jaxb2RootElementHttpMessageConverter` (JAXB 라이브러리 사용) 등
  - **문자열 처리:** `StringHttpMessageConverter`
  - **바이트 배열 처리:** `ByteArrayHttpMessageConverter`
- `@RequestBody`가 붙은 매개변수가 있으면, 스프링 MVC는 들어오는 요청의 `Content-Type` 헤더(예: `application/json`, `application/xml`)와 해당 매개변수의 자바 객체 타입(`User` 클래스)을 보고, **이 변환을 처리할 수 있는 적절한 `HttpMessageConverter`를 찾아서 사용**합니다.
- 예를 들어, `Content-Type`이 `application/json`이고 `User` 객체로 변환해야 한다면, `MappingJackson2HttpMessageConverter`가 선택되어 JSON 문자열을 `User` 객체의 필드에 매핑해줍니다.

**`@RequestParam` vs. `@RequestBody`**

이 둘의 차이를 명확히 이해하는 것이 매우 중요합니다.

| 특징 | `@RequestParam` | `@RequestBody` |
| --- | --- | --- |
| **처리 대상** | 개별 요청 파라미터 (URL 쿼리 스트링, 폼 데이터의 각 필드) | HTTP 요청 본문(body) 전체 |
| **데이터 형식** | 주로 키-값 쌍의 문자열 (예: `name=john&age=30`) | 주로 JSON, XML 등 구조화된 텍스트 형식 (예: `{"name":"john", "age":30}`) |
| **HTTP 메소드** | GET, POST (application/x-www-form-urlencoded) 등 | POST, PUT, PATCH 등 요청 본문을 포함하는 메소드 |
| **개수 제한** | 여러 개 사용 가능 | 한 메소드에 **하나만** 사용 가능 (요청 본문은 하나이므로) |
| **폼 데이터 처리** | **적합** (HTML 폼 데이터 처리용) | **부적합** (아래 설명 참조) |
| **내부 동작** | 서블릿 API의 `request.getParameter()` 계열 사용 | `HttpMessageConverter`를 통해 요청 본문 직접 읽고 변환 |

**폼 데이터(Form Data)와 `@RequestBody`**

원문에서도 강조했듯이, **HTML 폼 데이터(`application/x-www-form-urlencoded` 또는 `multipart/form-data`)를 처리할 때는 `@RequestBody`를 사용하면 안 됩니다.**

- **이유:** 서블릿 API 스펙상, `request.getParameter()` 메소드(또는 관련 메소드들)가 호출되면 요청 본문을 이미 소비(파싱)하게 됩니다. `@RequestParam`이나 커맨드 객체 바인딩은 내부적으로 이러한 서블릿 API를 사용합니다.
- 만약 `@RequestBody`를 사용하여 폼 데이터를 읽으려고 하면, 요청 본문이 이미 다른 곳(서블릿 파라미터 처리 과정)에서 읽혔을 수 있기 때문에 `@RequestBody`는 본문을 제대로 읽지 못하거나 오류가 발생할 수 있습니다. (요청 본문은 일반적으로 한 번만 읽을 수 있는 스트림 형태이기 때문입니다.)
- **결론:**
  - **폼 데이터 (키-값 쌍):** `@RequestParam` 또는 커맨드 객체 사용.
  - **JSON, XML 등 요청 본문 전체 데이터:** `@RequestBody` 사용.

### **주요 용어 해설 (복습 및 추가)**

- **요청 본문 (Request Body):** HTTP 요청 메시지에서 헤더 다음에 오는 부분으로, 클라이언트가 서버로 전송하고자 하는 실제 데이터를 담고 있습니다. (GET 요청은 보통 본문이 없습니다.)
- **역직렬화 (Deserialization):** 특정 형식(예: JSON, XML 문자열)으로 표현된 데이터를 메모리상의 객체(자바 객체) 구조로 변 Özellikle.
- **직렬화 (Serialization):** 메모리상의 객체(자바 객체)를 특정 형식(예: JSON, XML 문자열)으로 변환하는 과정. (@ResponseBody에서 사용)
- **`HttpMessageConverter`:** HTTP 요청/응답 본문과 자바 객체 간의 변환을 담당하는 스프링의 핵심 인터페이스.
- **Jackson, Gson, JAXB:** 각각 JSON, JSON, XML 처리를 위한 유명한 자바 라이브러리들. 스프링은 이들을 `HttpMessageConverter` 구현체로 활용합니다.

### **코드 예제 및 분석 (검증 포함)**

**1. `@RequestBody`와 데이터 검증 (`@Valid`, `Errors`)**

`Account` 자바 클래스 (검증 어노테이션 포함):

```java
import jakarta.validation.constraints.Email;
import jakarta.validation.constraints.NotEmpty;
import jakarta.validation.constraints.Size;

public class Account {

    @NotEmpty(message = "Username cannot be empty")
    @Size(min = 3, max = 20, message = "Username must be between 3 and 20 characters")
    private String username;

    @NotEmpty(message = "Password cannot be empty")
    private String password;

    @Email(message = "Email should be valid")
    private String email;

    // Getters and Setters
}
```

컨트롤러 메소드:

```java
import org.springframework.http.HttpStatus;
import org.springframework.http.ResponseEntity;
import org.springframework.stereotype.Controller;
import org.springframework.validation.Errors;
import org.springframework.web.bind.annotation.PostMapping;
import org.springframework.web.bind.annotation.RequestBody;
import jakarta.validation.Valid; // 또는 org.springframework.validation.annotation.Validated

@Controller
public class AccountController {

    @PostMapping("/accounts")
    public ResponseEntity<Object> createAccount(@Valid @RequestBody Account account, Errors errors) { // 1. @Valid, Errors 추가
        if (errors.hasErrors()) { // 2. 검증 오류 확인
            // 오류 응답 생성 (예: 오류 메시지를 포함한 400 Bad Request)
            return ResponseEntity.badRequest().body(
                errors.getAllErrors().stream()
                      .map(error -> error.getDefaultMessage())
                      .collect(Collectors.toList()) // 오류 메시지 리스트로 반환
            );
        }

        // 검증 통과: account 객체 사용 로직 (예: DB에 저장)
        System.out.println("Account created: " + account.getUsername());
        return ResponseEntity.status(HttpStatus.CREATED).body("Account created successfully");
    }
}
```

**코드 분석:**

1. `@Valid @RequestBody Account account, Errors errors`:
  - `@RequestBody Account account`: 이전 예제와 동일하게 요청 본문(JSON 등)을 `Account` 객체로 변환합니다.
  - `@Valid`: 스프링에게 이 `account` 객체에 대해 빈 검증(Bean Validation)을 수행하라고 지시합니다. `Account` 클래스에 정의된 `@NotEmpty`, `@Size`, `@Email` 등의 어노테이션에 따라 검증이 이루어집니다.
  - `Errors errors` (또는 `BindingResult errors`): 검증 결과를 담는 객체입니다. 만약 검증 오류가 발생하면, 이 `errors` 객체에 오류 정보가 채워집니다. **`@Valid` 어노테이션 바로 뒤에 `Errors` 또는 `BindingResult` 타입의 매개변수가 와야 합니다.**
2. `if (errors.hasErrors())`: `errors` 객체의 `hasErrors()` 메소드를 호출하여 검증 오류가 있었는지 확인합니다.
  - 오류가 있다면, 적절한 오류 응답(예: 400 Bad Request와 함께 오류 메시지)을 클라이언트에게 반환합니다. 위 예제에서는 모든 오류 메시지를 리스트로 만들어 응답 본문에 담았습니다.
  - 오류가 없다면, `account` 객체를 사용하여 비즈니스 로직을 수행합니다.

**기본 오류 처리:** 만약 `Errors` 매개변수를 선언하지 않고 `@Valid @RequestBody Account account`만 사용했는데 검증 오류가 발생하면, 스프링은 기본적으로 `MethodArgumentNotValidException`을 발생시키고, 이는 HTTP 400 (Bad Request) 응답으로 자동 변환되어 클라이언트에게 전달됩니다. `Errors` 객체를 사용하는 것은 오류를 좀 더 세밀하게 제어하고 커스텀 응답을 만들고 싶을 때 유용합니다.

### **"왜?" 라는 질문에 대한 답변**

**`@RequestBody`는 왜 필요할까요? 어떤 문제를 해결하기 위해 등장했을까요?**

1. **구조화된 데이터 전송의 표준화:** REST API와 같이 프로그램 간 통신에서는 단순한 키-값 쌍의 파라미터보다는 JSON이나 XML과 같이 계층적이고 구조화된 데이터를 주고받는 것이 일반적입니다. `@RequestBody`는 이러한 구조화된 데이터를 요청 본문에 담아 보내고, 서버에서는 이를 쉽게 자바 객체로 변환하여 사용할 수 있는 표준적인 방법을 제공합니다.
2. **복잡한 데이터 표현 용이:** 객체 안에 다른 객체가 있거나, 리스트를 포함하는 등 복잡한 데이터 구조를 표현할 때, URL 파라미터나 폼 데이터로는 한계가 있습니다. JSON이나 XML은 이러한 복잡한 데이터 구조를 표현하기에 매우 적합하며, `@RequestBody`는 이를 매끄럽게 처리합니다.
3. **요청 데이터와 객체 모델의 직접적인 매핑:** 클라이언트가 보내는 데이터 구조와 서버의 자바 객체 모델(예: `Account` 클래스)을 직접적으로 매핑할 수 있어 개발 생산성이 향상됩니다. 개발자는 문자열 파싱이나 수동 객체 변환 로직을 작성할 필요 없이 비즈니스 로직에 집중할 수 있습니다.
4. **`Content-Type` 기반의 유연한 처리:** `HttpMessageConverter`를 통해 요청의 `Content-Type`에 따라 다른 데이터 형식(JSON, XML 등)을 동일한 컨트롤러 메소드에서 처리할 수 있는 유연성을 제공합니다. (클라이언트가 `Content-Type: application/json`으로 보내면 JSON 컨버터가, `Content-Type: application/xml`로 보내면 XML 컨버터가 동작)

**폼 데이터에 `@RequestBody`를 사용하면 안 되는 이유는 무엇일까요? (다시 한번 강조)**

이는 서블릿 API의 요청 본문 처리 방식 때문입니다.

- `application/x-www-form-urlencoded` (일반 폼 데이터)나 `multipart/form-data` (파일 업로드 포함 폼 데이터)와 같은 `Content-Type`의 요청이 들어오면, 서블릿 컨테이너는 `request.getParameter()`와 같은 메소드를 통해 폼 필드 값을 읽을 수 있도록 요청 본문을 미리 파싱합니다.
- HTTP 요청 본문은 일반적으로 한 번만 읽을 수 있는 스트림(stream)입니다.
- 만약 `@RequestBody`가 이 폼 데이터 본문을 읽으려고 시도하면, 이미 서블릿 파라미터 처리 메커니즘에 의해 본문이 "소비"되었기 때문에, `@RequestBody`는 빈 본문을 읽거나 예외가 발생할 수 있습니다.
- 따라서 폼 데이터는 `@RequestParam`이나 커맨드 객체 바인딩(내부적으로 `request.getParameter()` 사용)을 통해 접근해야 하며, `@RequestBody`는 JSON/XML처럼 요청 본문 전체를 하나의 데이터 덩어리로 사용하는 경우에 적합합니다.

### **주의사항 및 Best Practice**

1. **`Content-Type` 헤더의 중요성:** 클라이언트는 반드시 요청 헤더에 올바른 `Content-Type`(예: `application/json`)을 명시해야 합니다. 서버는 이 헤더를 보고 적절한 `HttpMessageConverter`를 선택합니다. `Content-Type`이 없거나 잘못되면 변환 오류가 발생할 수 있습니다.
2. **적절한 `HttpMessageConverter` 등록:** 사용하려는 데이터 형식(JSON, XML 등)에 맞는 `HttpMessageConverter`가 스프링 설정에 등록되어 있어야 합니다. (스프링 부트에서는 Jackson 라이브러리가 클래스패스에 있으면 `MappingJackson2HttpMessageConverter`가 자동 등록됩니다.)
3. **객체 매핑 규칙 이해:** JSON/XML 데이터의 필드 이름과 자바 객체의 필드 이름(또는 getter/setter 이름)이 일치해야 기본적으로 매핑이 잘 이루어집니다. Jackson 같은 라이브러리는 `@JsonProperty` 어노테이션 등을 통해 필드 이름이 다른 경우도 매핑할 수 있는 방법을 제공합니다.
4. **한 메소드에 하나의 `@RequestBody`:** 하나의 HTTP 요청에는 하나의 본문만 있으므로, 컨트롤러 메소드에서도 `@RequestBody` 어노테이션은 하나의 매개변수에만 사용할 수 있습니다.
5. **검증 활용:** 클라이언트로부터 받은 데이터는 항상 신뢰할 수 없으므로, `@Valid`와 `BindingResult` (또는 `Errors`)를 사용하여 서버 측에서 반드시 유효성 검사를 수행하는 것이 좋습니다.
6. **예외 처리:** 역직렬화 실패(예: JSON 형식이 잘못됨), 검증 실패 등으로 인해 예외가 발생할 수 있습니다. `@ControllerAdvice`와 `@ExceptionHandler`를 사용하여 전역적으로 예외를 처리하거나, `Errors` 객체를 통해 지역적으로 처리할 수 있습니다.

### **이전 학습 내용과의 연관성**

- **커맨드 객체 vs. `@RequestBody`:**
  - 커맨드 객체는 요청 파라미터(키-값 쌍)를 객체에 바인딩합니다. (주로 폼 데이터 처리)
  - `@RequestBody`는 요청 본문 전체(주로 JSON/XML)를 객체에 바인딩합니다.
  - 둘 다 자바 객체로 데이터를 받는다는 점은 유사하지만, 데이터의 출처와 형식이 다릅니다.
- **`HttpMessageConverter`:** 멀티파트 요청에서 `@RequestPart`가 JSON/XML 파트를 객체로 변환할 때도 `HttpMessageConverter`가 사용되었습니다. `@RequestBody`는 이 `HttpMessageConverter`의 가장 대표적인 사용 사례 중 하나입니다.
- **검증 (Validation):** 커맨드 객체나 `@RequestPart`로 받은 객체에 검증을 적용했던 것처럼, `@RequestBody`로 받은 객체에도 동일한 방식으로 `@Valid`를 사용하여 빈 검증을 적용할 수 있습니다. 스프링의 일관된 검증 지원을 보여줍니다.

---
