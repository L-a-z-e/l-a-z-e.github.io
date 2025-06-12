---
title: Spring Web MVC - Annotated Controllers (HttpEntity)
description: 
author: laze
date: 2025-06-12 00:00:01 +0900
categories: [Dev, SpringWebMVC]
tags: [SpringWebMVC]
---
### HttpEntity

> Reactive 스택에서의 동일 기능은 해당 문서를 참고하세요.
>

`HttpEntity`는 `@RequestBody`를 사용하는 것과 거의 동일하지만, 요청 헤더와 본문을 노출하는 컨테이너 객체에 기반합니다.

**Java**

```java
@PostMapping("/accounts")
public void handle(HttpEntity<Account> entity) { // 요청 헤더와 본문(Account 객체로 변환됨)을 HttpEntity로 받음
	// ...
}
```

---

### **학습 목표 제시**

이 챕터를 학습하고 나면, 학생은 다음을 이해하고 수행할 수 있게 됩니다:

1. **`HttpEntity`의 기본 역할과 `@RequestBody`와의 유사점 및 차이점 이해:** `HttpEntity`가 HTTP 요청의 본문과 헤더를 함께 캡슐화하는 방법을 이해하고, `@RequestBody`와의 주요 공통점과 핵심 차이점을 설명할 수 있습니다.
2. **`HttpEntity`를 사용하여 요청 본문 및 헤더 접근 방법 숙지:** 컨트롤러 메소드에서 `HttpEntity` 타입의 매개변수를 사용하여 요청 본문을 특정 객체로 변환하고, 동시에 요청 헤더 정보에 어떻게 접근하는지 이해합니다.
3. **`HttpEntity`의 활용 사례 인지:** 어떤 상황에서 `@RequestBody` 대신 `HttpEntity`를 사용하는 것이 더 유용할 수 있는지 인지합니다. (예: 요청 헤더 값을 기반으로 로직을 분기해야 할 때)

---

### **핵심 개념 설명**

**`HttpEntity<T>`란 무엇일까요?**

`HttpEntity<T>`는 스프링 프레임워크에서 **HTTP 요청 또는 응답 전체를 표현하는 객체**입니다. 여기서 `<T>`는 제네릭 타입으로, HTTP 본문(body)의 내용을 나타내는 자바 타입을 의미합니다.

- **HTTP 요청을 다룰 때:** `HttpEntity<T>`는 클라이언트가 보낸 요청의 **헤더(headers)와 본문(body)을 모두 포함**하는 컨테이너 역할을 합니다.
  - 본문 내용은 `@RequestBody`와 마찬가지로 `HttpMessageConverter`를 통해 `<T>` 타입의 객체로 자동 변환(역직렬화)됩니다.
  - 추가적으로 요청 헤더 정보에도 접근할 수 있습니다.
- **HTTP 응답을 생성할 때:** `ResponseEntity<T>`는 `HttpEntity<T>`를 상속받으며, 서버가 클라이언트에게 보낼 응답의 상태 코드(status code), 헤더, 그리고 본문(`<T>` 타입 객체)을 담는 데 사용됩니다.

**`@RequestBody`와의 유사점 및 차이점**

| 특징 | `@RequestBody MyType body` | `HttpEntity<MyType> entity` |
| --- | --- | --- |
| **본문 처리** | 요청 본문을 `MyType` 객체로 변환하여 `body`에 주입. | 요청 본문을 `MyType` 객체로 변환하여 `entity.getBody()`로 접근. |
| **헤더 접근** | 불가능 (별도로 `@RequestHeader` 사용 필요) | 가능 (`entity.getHeaders()` 사용) |
| **핵심 기능** | 요청 본문 객체 변환에 집중 | 요청 본문 객체 변환 + 요청 헤더 접근 기능 제공 |
| **내부 동작** | `HttpMessageConverter` 사용 | `HttpMessageConverter` 사용 (본문 변환 시) |
| **주 사용처** | 요청 본문 데이터만 필요할 때 | 요청 본문 데이터와 함께 특정 요청 헤더 값도 필요할 때 |

**간단히 말해, `HttpEntity<T>`는 `@RequestBody T object`와 `@RequestHeader HttpHeaders headers`를 하나로 합쳐놓은 듯한 편리함을 제공한다고 볼 수 있습니다.**

**비유:**

여러분이 레스토랑에서 특별 주문한 요리(HTTP 요청)를 받는다고 생각해봅시다.

- **`@RequestBody` 사용 시:** 주방장이 요리(요청 본문 객체)만 접시에 담아 가져다주는 것과 같습니다. 요리 자체에만 관심이 있을 때 유용합니다.
- **`HttpEntity` 사용 시:** 주방장이 요리(요청 본문 객체)가 담긴 접시와 함께, "이 요리는 특별히 '신선한 재료만 사용'이라는 요청(요청 헤더)에 따라 만들었습니다"라는 **메모(요청 헤더 정보)**까지 함께 가져다주는 것과 같습니다. 요리 자체와 함께 그 요리가 어떤 조건이나 요청 하에 만들어졌는지에 대한 부가 정보(헤더)도 함께 확인하고 싶을 때 유용합니다.

### **주요 용어 해설**

- **`HttpEntity<T>`:** HTTP 요청/응답을 나타내는 제네릭 클래스. 헤더와 본문을 포함합니다.
  - `T getBody()`: HTTP 본문을 `<T>` 타입 객체로 반환합니다.
  - `HttpHeaders getHeaders()`: HTTP 헤더 정보를 담고 있는 `HttpHeaders` 객체를 반환합니다.
- **`HttpHeaders`:** 스프링에서 HTTP 헤더 정보를 표현하고 다루기 위한 클래스. 다양한 헤더에 쉽게 접근하고 설정할 수 있는 메소드들을 제공합니다. (이전에 `@RequestHeader`에서 잠깐 봤었죠?)

### **코드 예제 및 분석**

원문의 Java 코드를 다시 살펴보겠습니다.

`Account` 자바 클래스 (예시):

```java
public class Account {
    private String accountNumber;
    private String ownerName;
    // Getters and Setters
}
```

컨트롤러 메소드:

```java
import org.springframework.http.HttpEntity;
import org.springframework.http.HttpHeaders;
import org.springframework.http.MediaType;
import org.springframework.stereotype.Controller;
import org.springframework.web.bind.annotation.PostMapping;

@Controller
public class AccountControllerWithHttpEntity {

    @PostMapping("/accountsWithHttpEntity")
    public void handle(HttpEntity<Account> entity) { // 1. HttpEntity<Account>를 매개변수로 받음
        Account account = entity.getBody(); // 2. 요청 본문(Account 객체) 가져오기
        HttpHeaders headers = entity.getHeaders(); // 3. 요청 헤더 가져오기

        // account 객체 사용 (예: 데이터베이스에 저장)
        if (account != null) {
            System.out.println("Account Number from body: " + account.getAccountNumber());
            System.out.println("Owner Name from body: " + account.getOwnerName());
        }

        // headers 객체 사용 (예: 특정 헤더 값 확인)
        MediaType contentType = headers.getContentType();
        long contentLength = headers.getContentLength();
        String customHeaderValue = headers.getFirst("X-Custom-Header");

        System.out.println("Request Content-Type: " + contentType);
        System.out.println("Request Content-Length: " + contentLength);
        if (customHeaderValue != null) {
            System.out.println("X-Custom-Header: " + customHeaderValue);
        }

        // 응답 생성 로직 (여기서는 void 반환, 실제로는 ResponseEntity 등을 사용할 수 있음)
    }
}
```

**코드 분석:**

1. `HttpEntity<Account> entity`: 컨트롤러 메소드의 매개변수로 `HttpEntity<Account>`를 선언합니다.
  - 스프링 MVC는 클라이언트 요청 본문(예: JSON)을 `HttpMessageConverter`를 사용하여 `Account` 타입 객체로 변환합니다.
  - 또한, 요청 헤더 정보도 함께 `HttpEntity` 객체 안에 담아줍니다.
2. `Account account = entity.getBody();`: `HttpEntity` 객체의 `getBody()` 메소드를 호출하여, 변환된 `Account` 객체를 가져옵니다. 이 `account` 객체는 `@RequestBody Account account`로 받았을 때의 `account` 객체와 동일한 것입니다.
3. `HttpHeaders headers = entity.getHeaders();`: `HttpEntity` 객체의 `getHeaders()` 메소드를 호출하여, 요청 헤더 정보를 담고 있는 `HttpHeaders` 객체를 가져옵니다. 이 `headers` 객체를 통해 `Content-Type`, `Content-Length` 또는 사용자 정의 헤더(`X-Custom-Header`) 등 다양한 헤더 값에 접근할 수 있습니다.

**`@RequestBody`와 `@RequestHeader`를 함께 사용하는 것과 비교:**

위 `HttpEntity`를 사용한 코드는 다음과 같이 `@RequestBody`와 `@RequestHeader`를 함께 사용한 것과 기능적으로 유사합니다.

```java
@PostMapping("/accountsWithRequestBodyAndHeader")
public void handleWithRequestBodyAndHeader(
        @RequestBody Account account, // 요청 본문만 받음
        @RequestHeader HttpHeaders headers) { // 모든 요청 헤더를 받음 (또는 특정 헤더만 지정 가능)

    // account 객체 사용
    // headers 객체 사용
}
```

`HttpEntity`는 이 두 가지를 하나의 객체로 묶어서 제공하므로 코드가 조금 더 간결해 보일 수 있고, 요청의 본문과 헤더가 논리적으로 함께 처리되어야 할 때 더 적합할 수 있습니다.

### **"왜?" 라는 질문에 대한 답변**

**`HttpEntity`는 왜 필요할까요? `@RequestBody`와 `@RequestHeader`를 각각 사용하면 되는데?**

1. **요청의 전체적인 표현:** `HttpEntity`는 HTTP 요청(또는 응답)의 본질적인 구성 요소인 헤더와 본문을 하나의 객체로 묶어서 표현합니다. 이는 요청 데이터를 좀 더 객체 지향적으로 다룰 수 있게 해줍니다.
2. **코드의 간결성 및 가독성 (경우에 따라):** 요청 본문과 헤더 정보가 모두 필요한 경우, 매개변수가 하나로 줄어들어 코드가 더 간결해 보일 수 있습니다. 특히 특정 헤더 값에 따라 본문 처리 로직이 달라져야 하는 경우, 관련된 정보가 하나의 객체 안에 있으므로 로직의 가독성이 높아질 수 있습니다.
3. **Spring RestTemplate과의 일관성:** 스프링에서 제공하는 HTTP 클라이언트 라이브러리인 `RestTemplate` (또는 최신의 `WebClient`)을 사용하여 외부 API를 호출할 때도 `HttpEntity` (또는 `RequestEntity`, `ResponseEntity`)를 사용하여 요청/응답을 구성하고 처리합니다. 컨트롤러에서도 유사한 `HttpEntity`를 사용함으로써 서버 측과 클라이언트 측 코드 간의 개념적 일관성을 유지할 수 있습니다.
4. **응답 생성 시의 유용성 (`ResponseEntity`):** 비록 이번 챕터는 요청 처리에 관한 것이지만, `HttpEntity`의 하위 클래스인 `ResponseEntity`는 응답을 생성할 때 상태 코드, 헤더, 본문을 한 번에 설정하여 반환할 수 있게 해주어 매우 유용합니다. `HttpEntity`를 이해하는 것은 `ResponseEntity`를 이해하는 데 도움이 됩니다.

**언제 `HttpEntity`를 사용하는 것이 좋을까요?**

- 요청 본문 데이터와 함께 특정 요청 헤더 값(예: `If-Match`, `If-None-Match` 같은 조건부 요청 헤더, 커스텀 인증 헤더 등)을 반드시 확인하고 로직을 처리해야 할 때.
- 요청 헤더와 본문을 하나의 단위로 묶어서 처리하는 것이 로직상 더 자연스러울 때.
- `RestTemplate` 등 스프링의 HTTP 클라이언트와 코드 스타일을 유사하게 가져가고 싶을 때.

반대로, **요청 본문 데이터만 필요하고 헤더 정보는 전혀 필요 없다면, 굳이 `HttpEntity`를 사용할 필요 없이 `@RequestBody`만 사용하는 것이 더 간단하고 명확**할 수 있습니다.

### **주의사항 및 Best Practice**

1. **본문 변환은 여전히 `HttpMessageConverter`:** `HttpEntity<T>`의 본문 부분(`T`)은 `@RequestBody`와 동일하게 `HttpMessageConverter`에 의해 변환됩니다. 따라서 `Content-Type` 헤더의 정확성, 적절한 컨버터 등록 등의 주의사항은 동일하게 적용됩니다.
2. **`@RequestBody`와 동시 사용 불가:** 하나의 컨트롤러 메소드 매개변수로 `HttpEntity<T>`와 `@RequestBody T`를 동시에 사용할 수는 없습니다. 둘 다 요청 본문을 소비하려고 하기 때문입니다. `HttpEntity`를 사용하면 이미 본문 정보가 그 안에 포함되어 있습니다.
3. **검증 (`@Valid`):** `@RequestBody`와 마찬가지로 `HttpEntity`의 본문 객체에 대해서도 검증을 적용할 수 있습니다. 다만, `@Valid` 어노테이션을 `HttpEntity<@Valid Account> entity` 와 같이 제네릭 타입에 직접 적용하는 것은 일반적인 사용법이 아니며, 보통은 `HttpEntity`를 받은 후 `entity.getBody()`로 객체를 꺼내어 별도로 검증하거나, 스프링의 고급 기능을 활용해야 할 수 있습니다. 가장 간단한 방법은 `HttpEntity`를 사용하지 않고 `@Valid @RequestBody Account account` 와 `Errors errors`를 사용하는 것입니다. 만약 `HttpEntity`를 꼭 써야 하고 검증도 필요하다면, 메소드 내부에서 수동으로 Validator를 호출하거나 AOP 등을 고려해야 할 수 있습니다. (이 부분은 스프링 버전에 따라 지원 방식이 다를 수 있으므로 공식 문서를 참고하는 것이 좋습니다. 일반적으로는 `@RequestBody`와 `@Valid` 조합이 더 흔합니다.)
4. **불변성(Immutability):** `HttpEntity` 객체는 생성된 후에는 일반적으로 불변(immutable)으로 취급됩니다. 즉, `getBody()`나 `getHeaders()`로 내용을 가져올 수는 있지만, `HttpEntity` 객체 자체의 내용을 직접 변경하는 것은 권장되지 않거나 불가능할 수 있습니다. (응답을 위한 `ResponseEntity`는 빌더를 통해 생성합니다.)

### **이전 학습 내용과의 연관성**

- **`@RequestBody`:** `HttpEntity`는 `@RequestBody`의 기능을 포함하면서 헤더 정보까지 추가로 제공합니다. `@RequestBody`를 확실히 이해했다면 `HttpEntity`의 본문 처리 부분은 쉽게 이해할 수 있습니다.
- **`@RequestHeader`:** `HttpEntity.getHeaders()`를 통해 얻는 `HttpHeaders` 객체는 `@RequestHeader HttpHeaders headers`로 모든 헤더를 받는 것과 유사한 정보를 제공합니다.
- **`HttpMessageConverter`:** `HttpEntity`의 본문이 객체로 변환되는 과정의 핵심에는 여전히 `HttpMessageConverter`가 있습니다.
- **`HttpHeaders` 클래스:** 이전에 `@RequestHeader`를 배울 때 언급되었던 `HttpHeaders` 클래스를 `HttpEntity`를 통해 다시 만나게 됩니다. 이 클래스는 HTTP 헤더를 편리하게 다룰 수 있는 다양한 메소드를 제공합니다.

---
