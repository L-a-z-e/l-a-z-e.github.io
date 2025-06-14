---
title: Spring Web MVC - Annotated Controllers (ResponseEntity)
description: 
author: laze
date: 2025-06-14 00:00:01 +0900
categories: [Dev, SpringWebMVC]
tags: [SpringWebMVC]
---
### ResponseEntity

`ResponseEntity`는 `@ResponseBody`와 유사하지만 상태(status)와 헤더(headers)를 포함합니다.

**Java**

```java
@GetMapping("/something")
public ResponseEntity<String> handle() {
	String body = ... ;    // 응답 본문 내용
	String etag = ... ;     // ETag 값
	return ResponseEntity.ok().eTag(etag).body(body); // 상태 코드 200 OK, ETag 헤더, 본문 설정하여 반환
}

```

본문(body)은 일반적으로 등록된 `HttpMessageConverter` 중 하나에 의해 해당 응답 표현(예: JSON)으로 렌더링될 값 객체(value object)로 제공됩니다.

파일 내용을 위해 `ResponseEntity<Resource>`를 반환할 수 있으며, 제공된 리소스의 `InputStream` 내용이 응답 `OutputStream`으로 복사됩니다.

`InputStream`은 응답으로 복사된 후 안정적으로 닫히기 위해 리소스 핸들에 의해 지연(lazily) 검색되어야 한다는 점에 유의하세요.

이러한 목적으로 `InputStreamResource`를 사용하는 경우, (실제 `InputStream`을 검색하는 람다 표현식 등을 통해) 필요시점에 `InputStream`을 가져오는 `InputStreamSource`로 생성해야 합니다.

또한, `InputStreamResource`의 사용자 정의 하위 클래스는 해당 목적으로 스트림을 소비하는 것을 피하는 사용자 정의 `contentLength()` 구현과 함께 사용할 때만 지원됩니다.

스프링 MVC는 `ResponseEntity`를 비동기적으로 생성하기 위해 단일 값 반응형 타입(single value reactive type)을 사용하거나, 본문을 위해 단일 및 다중 값 반응형 타입(single and multi-value reactive types)을 사용하는 것을 지원합니다.

이를 통해 다음과 같은 유형의 비동기 응답이 가능합니다:

- `ResponseEntity<Mono<T>>` 또는 `ResponseEntity<Flux<T>>`: 응답 상태와 헤더를 즉시 알리고 본문은 나중에 비동기적으로 제공합니다. 본문이 0개 또는 1개의 값으로 구성되면 `Mono`를 사용하고, 여러 값을 생성할 수 있으면 `Flux`를 사용합니다.
- `Mono<ResponseEntity<T>>`: 응답 상태, 헤더, 본문 세 가지 모두를 나중에 비동기적으로 제공합니다. 이를 통해 비동기 요청 처리 결과에 따라 응답 상태와 헤더가 달라질 수 있습니다.

---

### **학습 목표 제시**

이 챕터를 학습하고 나면, 학생은 다음을 이해하고 수행할 수 있게 됩니다:

1. **`ResponseEntity`의 핵심 기능 및 `@ResponseBody`와의 관계 이해:** `ResponseEntity`가 HTTP 응답의 상태 코드, 헤더, 본문을 모두 제어할 수 있는 방법을 이해하고, `@ResponseBody`와의 유사점 및 `ResponseEntity`가 제공하는 추가적인 제어 능력을 설명할 수 있습니다.
2. **`ResponseEntity`를 사용한 다양한 HTTP 응답 생성 방법 숙지:** 정적 팩토리 메소드(예: `ok()`, `created()`, `notFound()`)와 빌더 패턴을 사용하여 원하는 상태 코드, 헤더, 본문을 가진 `ResponseEntity` 객체를 생성하고 반환하는 방법을 이해하고 적용할 수 있습니다.
3. **`ResponseEntity<Resource>`를 이용한 파일 다운로드 응답 처리 이해:** `ResponseEntity`를 사용하여 파일(`Resource`)을 다운로드하는 응답을 어떻게 구성하는지 이해하고, 관련 주의사항(지연 로딩 등)을 인지합니다. (반응형 타입 관련 내용은 심화 내용으로 간단히 인지만 합니다.)

---

### **핵심 개념 설명**

**`ResponseEntity<T>`란 무엇일까요?**

`ResponseEntity<T>`는 스프링 프레임워크에서 **HTTP 응답 전체를 표현하는 매우 강력하고 유연한 클래스**입니다.

여기서 `<T>`는 제네릭 타입으로, 응답 본문(body)의 내용을 나타내는 자바 타입을 의미합니다.

`ResponseEntity`는 다음 세 가지 주요 구성 요소를 모두 포함하고 제어할 수 있게 해줍니다:

1. **HTTP 상태 코드 (Status Code):** 요청 처리 결과를 나타내는 숫자 코드 (예: `200 OK`, `201 Created`, `404 Not Found`, `500 Internal Server Error` 등).
2. **HTTP 헤더 (Headers):** 응답에 대한 추가 정보(메타데이터)를 담고 있는 키-값 쌍들 (예: `Content-Type`, `Content-Length`, `ETag`, `Location` 등).
3. **HTTP 본문 (Body):** 클라이언트에게 전달될 실제 데이터 (`<T>` 타입의 객체).

**`@ResponseBody`와의 관계 및 차이점**

- **유사점:** `@ResponseBody` 어노테이션이 붙은 메소드가 객체를 반환하면 그 객체가 응답 본문으로 직렬화되는 것처럼, `ResponseEntity<T>`의 본문 부분(`T` 타입 객체)도 `HttpMessageConverter`에 의해 동일한 방식으로 직렬화되어 응답 본문에 쓰여집니다.
- **차이점 (핵심!):**
  - `@ResponseBody`만 사용하면, 반환된 객체가 응답 본문이 되고 HTTP 상태 코드는 대부분 성공 시 `200 OK`로 자동 설정되며, 헤더 제어는 제한적입니다.
  - `ResponseEntity<T>`를 사용하면, 개발자가 **프로그래밍 방식으로 상태 코드와 헤더를 명시적으로 설정**할 수 있습니다. 즉, 응답의 모든 측면을 완벽하게 제어할 수 있게 됩니다.

**`HttpEntity` 상속:** `ResponseEntity<T>`는 사실 이전에 배운 `HttpEntity<T>`를 상속받습니다. `HttpEntity`가 헤더와 본문을 가졌던 것처럼, `ResponseEntity`도 이를 가지며, 추가적으로 상태 코드를 더 가집니다.

**비유:**

여러분이 고객에게 제품(HTTP 응답)을 배송한다고 생각해봅시다.

- **`@ResponseBody` 사용 시 (객체 반환):** 고객에게 제품 내용물(응답 본문)만 보내는 것과 비슷합니다. 배송 상태(성공, 실패 등)나 특별 취급 주의사항(헤더)을 명시적으로 전달하기 어렵고, 보통 "성공적으로 배송됨"(200 OK) 정도로만 간주됩니다.
- **`ResponseEntity` 사용 시:** 고객에게 제품 내용물(응답 본문)과 함께, **배송 상태 확인서(상태 코드)**와 **취급 주의사항 및 상세 정보가 적힌 안내문(헤더)**까지 완벽하게 갖춰서 보내는 것과 같습니다. 예를 들어, "제품이 성공적으로 생성되어 배송 시작됨 (201 Created)", "신제품 안내 (Location 헤더)", "내용물 파손 주의 (커스텀 헤더)" 등을 명확히 전달할 수 있습니다.

**`ResponseEntity` 생성 방법**

`ResponseEntity` 객체는 주로 정적 팩토리 메소드와 빌더 패턴을 사용하여 생성합니다.

1. **정적 팩토리 메소드 활용:**
  - `ResponseEntity.ok()`: `200 OK` 상태. 본문과 헤더를 추가로 설정할 수 있습니다.

      ```java
      return ResponseEntity.ok().body("Success!");
      return ResponseEntity.ok().header("X-Custom", "value").body(userObject);
      ```

  - `ResponseEntity.status(HttpStatus status)`: 원하는 `HttpStatus` 열거형 상수로 상태 코드를 지정합니다.

      ```java
      return ResponseEntity.status(HttpStatus.CREATED).body(newUser); // 201 Created
      ```

  - `ResponseEntity.created(URI location)`: `201 Created` 상태와 함께, 새로 생성된 리소스의 위치를 나타내는 `Location` 헤더를 설정합니다.

      ```java
      URI location = new URI("/users/" + newUser.getId());
      return ResponseEntity.created(location).body(newUser);
      ```

  - `ResponseEntity.notFound()`: `404 Not Found` 상태의 응답을 쉽게 만듭니다. (본문 없이)

      ```java
      return ResponseEntity.notFound().build(); // .build()로 최종 생성
      ```

  - `ResponseEntity.badRequest()`: `400 Bad Request` 상태.
  - `ResponseEntity.noContent()`: `204 No Content` 상태. (본문 없이)
  - 등등 다양한 상태에 대한 정적 메소드가 제공됩니다.
2. **빌더 패턴 활용:**
   정적 팩토리 메소드 뒤에 체이닝(chaining) 방식으로 `header()`, `eTag()`, `contentType()`, `contentLength()`, `body()` 등을 호출하여 `ResponseEntity`를 구성합니다. 마지막에는 `build()` (본문이 없을 때) 또는 `body()` (본문이 있을 때 최종적으로 본문을 설정하며 반환)를 호출합니다.

   원문의 예시:

    ```java
    @GetMapping("/something")
    public ResponseEntity<String> handle() {
        String bodyContent = "Hello, ResponseEntity!";
        String etagValue = "\\"12345\\""; // ETag는 보통 따옴표로 감싸짐
    
        return ResponseEntity.ok() // 1. 200 OK 상태로 시작
                .eTag(etagValue)   // 2. ETag 헤더 설정 (내부적으로 "ETag" 헤더에 값 설정)
                .body(bodyContent); // 3. 본문 설정 및 ResponseEntity<String> 반환
    }
    ```


### **주요 용어 해설**

- **`ResponseEntity<T>`:** HTTP 응답의 상태 코드, 헤더, 본문을 모두 포함하는 스프링 클래스.
- **HTTP 상태 코드 (HTTP Status Code):** 클라이언트 요청에 대한 서버의 처리 결과를 나타내는 세 자리 숫자. (예: `200`, `201`, `400`, `404`, `500`)
- **`HttpStatus` 열거형:** 스프링에서 표준 HTTP 상태 코드들을 편리하게 사용하기 위해 제공하는 열거형 타입. (예: `HttpStatus.OK`, `HttpStatus.CREATED`)
- **빌더 패턴 (Builder Pattern):** 복잡한 객체의 생성 과정과 표현 방법을 분리하여, 동일한 생성 절차에서 서로 다른 표현 결과를 만들 수 있게 하는 디자인 패턴. `ResponseEntity` 생성 시 유연하고 가독성 높게 사용됩니다.
- **ETag (Entity Tag):** 특정 버전의 리소스를 식별하는 HTTP 응답 헤더. 캐싱 및 조건부 요청에 사용됩니다.

### **코드 예제 및 분석**

**1. 다양한 상태 코드와 헤더, 본문 설정 예제**

```java
import org.springframework.http.HttpHeaders;
import org.springframework.http.HttpStatus;
import org.springframework.http.MediaType;
import org.springframework.http.ResponseEntity;
import org.springframework.web.bind.annotation.GetMapping;
import org.springframework.web.bind.annotation.PathVariable;
import org.springframework.web.bind.annotation.PostMapping;
import org.springframework.web.bind.annotation.RequestBody;
import org.springframework.web.bind.annotation.RestController;

import java.net.URI;
import java.net.URISyntaxException;

@RestController
public class AdvancedResponseController {

    // 가상의 User 객체
    static class User {
        public Long id;
        public String name;
        public User(Long id, String name) { this.id = id; this.name = name; }
        public Long getId() { return id; }
        public String getName() { return name; }
    }

    // 200 OK와 함께 본문, 커스텀 헤더
    @GetMapping("/users/{id}")
    public ResponseEntity<User> getUser(@PathVariable Long id) {
        User user = findUserById(id); // 가상 메소드
        if (user == null) {
            return ResponseEntity.notFound().build(); // 404 Not Found
        }
        HttpHeaders headers = new HttpHeaders();
        headers.add("X-User-Role", "ADMIN"); // 커스텀 헤더 추가
        return new ResponseEntity<>(user, headers, HttpStatus.OK); // 생성자 직접 사용도 가능
        // 또는 빌더 사용: return ResponseEntity.ok().headers(headers).body(user);
    }

    // 201 Created와 함께 Location 헤더, 본문
    @PostMapping("/users")
    public ResponseEntity<User> createUser(@RequestBody User newUser) throws URISyntaxException {
        User createdUser = saveUser(newUser); // 가상 메소드, ID가 할당되었다고 가정
        URI location = new URI("/users/" + createdUser.getId());
        return ResponseEntity.created(location) // Location 헤더 자동 설정
                             .contentType(MediaType.APPLICATION_JSON) // Content-Type 명시
                             .body(createdUser);
    }

    // 400 Bad Request (예: 요청 데이터 유효성 검사 실패 시)
    @PostMapping("/validate")
    public ResponseEntity<String> validateData(@RequestBody String data) {
        if (data == null || data.isEmpty()) {
            return ResponseEntity.badRequest().body("Data cannot be empty");
        }
        return ResponseEntity.ok("Data is valid");
    }

    // 204 No Content (예: 삭제 성공 후 내용 없을 때)
    @GetMapping("/deleteResource")
    public ResponseEntity<Void> deleteResource() {
        // 리소스 삭제 로직 ...
        boolean deleted = true; // 삭제 성공했다고 가정
        if (deleted) {
            return ResponseEntity.noContent().build(); // 본문 없는 204 응답
        } else {
            return ResponseEntity.status(HttpStatus.INTERNAL_SERVER_ERROR).build();
        }
    }

    // 가상 메소드들
    private User findUserById(Long id) {
        if (id == 1L) return new User(1L, "Alice");
        return null;
    }
    private User saveUser(User user) {
        user.id = System.currentTimeMillis() % 1000; // 간단한 ID 할당
        return user;
    }
}
```

**코드 분석:**

- `getUser()`: ID로 사용자를 찾아 `200 OK`와 함께 사용자 정보(본문), 커스텀 헤더(`X-User-Role`)를 반환합니다. 사용자를 찾지 못하면 `404 Not Found`를 반환합니다.
- `createUser()`: 새 사용자를 생성하고, `201 Created` 상태 코드와 함께 생성된 리소스의 위치를 `Location` 헤더에 담아 반환합니다. `Content-Type`도 명시적으로 `application/json`으로 설정했습니다.
- `validateData()`: 간단한 데이터 유효성 검사를 수행하여, 데이터가 비어있으면 `400 Bad Request`와 함께 오류 메시지를 본문에 담아 반환합니다.
- `deleteResource()`: 리소스 삭제 후 성공 시 본문 없는 `204 No Content` 응답을 반환합니다.

**2. `ResponseEntity<Resource>`를 이용한 파일 다운로드 (이전 챕터 내용 복습 및 심화)**

이전 `@ResponseBody` 챕터에서 `ResponseEntity<Resource>`를 사용하여 파일 다운로드를 구현한 예제가 있었습니다. 그 예제에서 `ResponseEntity.ok().contentType(...).header(...).body(resource)` 와 같이 빌더 패턴을 사용하여 상태 코드, 헤더(Content-Type, Content-Disposition), 본문(`Resource`)을 설정한 것이 바로 `ResponseEntity`의 강력한 활용 예입니다.

```java
// (FileDownloadController 예제 코드 중 일부 발췌)
// return ResponseEntity.ok()
//         .contentType(MediaType.parseMediaType(contentType))
//         .header(HttpHeaders.CONTENT_DISPOSITION, "attachment; filename=\\"" + file.getName() + "\\"")
//         .contentLength(file.length())
//         .body(resource);
```

이 코드는 `200 OK` 상태, 적절한 `Content-Type`, 브라우저가 파일을 다운로드하도록 하는 `Content-Disposition` 헤더, 파일 크기, 그리고 실제 파일 데이터를 담은 `Resource` 객체를 응답으로 구성합니다.

### **"왜?" 라는 질문에 대한 답변**

**`ResponseEntity`는 왜 필요할까요? `@ResponseBody`만으로는 부족한가?**

1. **HTTP 프로토콜의 완전한 제어:** 웹 애플리케이션, 특히 REST API는 HTTP 프로토콜 위에서 동작합니다. HTTP는 단순한 데이터 전송 외에도 상태 코드와 헤더를 통해 매우 풍부한 의미를 전달할 수 있습니다. `ResponseEntity`는 개발자가 이러한 HTTP 프로토콜의 모든 요소를 완벽하게 제어하여, 보다 정확하고 명확하며 표준을 준수하는 API 응답을 만들 수 있게 해줍니다.
2. **클라이언트와의 명확한 커뮤니케이션:**
  - **상태 코드:** 단순히 "성공/실패"를 넘어, "성공적으로 생성됨(201)", "요청이 잘못됨(400)", "권한 없음(401/403)", "리소스를 찾을 수 없음(404)" 등 구체적인 처리 결과를 클라이언트에게 명확히 알릴 수 있습니다. 이는 클라이언트 측에서 적절한 후속 조치를 취하는 데 매우 중요합니다.
  - **헤더:** `Location` (새 리소스 위치), `ETag` (캐싱), `Content-Type` (본문 형식), `WWW-Authenticate` (인증 요구) 등 다양한 헤더를 통해 부가 정보를 전달하거나 프로토콜 수준의 상호작용을 할 수 있습니다.
3. **RESTful API 설계 원칙 준수:** 잘 설계된 REST API는 HTTP 메소드(GET, POST, PUT, DELETE 등)와 상태 코드를 적절히 사용하여 리소스의 상태를 표현하고 조작합니다. `ResponseEntity`는 이러한 RESTful 원칙을 따르는 API를 구축하는 데 필수적인 도구입니다.
4. **유연한 응답 구성:** 동일한 요청에 대해서도 처리 결과에 따라 다른 상태 코드, 다른 헤더, 다른 본문(또는 본문 없음)을 가진 응답을 동적으로 생성하여 반환할 수 있는 유연성을 제공합니다.

`@ResponseBody`는 주로 "성공적인 경우(200 OK)에 이 데이터를 본문으로 보내줘"라는 단순한 시나리오에 적합하지만, `ResponseEntity`는 이보다 훨씬 넓은 범위의 HTTP 응답 시나리오를 섬세하게 다룰 수 있게 해줍니다.

### **주의사항 및 Best Practice**

1. **적절한 상태 코드 사용:** 상황에 맞는 정확한 HTTP 상태 코드를 반환하는 것이 중요합니다. 이는 API의 의미를 명확하게 하고 클라이언트가 올바르게 동작하도록 돕습니다. (예: 생성 성공 시 201, 단순 조회 성공 시 200, 클라이언트 오류 시 4xx, 서버 오류 시 5xx)
2. **표준 헤더 및 커스텀 헤더 활용:** `Content-Type`, `Location`, `ETag` 등 표준 HTTP 헤더를 적극적으로 활용하고, 필요하다면 의미 있는 커스텀 헤더(`X-` 접두사 사용 권장)를 추가하여 부가 정보를 제공할 수 있습니다.
3. **본문 직렬화는 `HttpMessageConverter` 담당:** `ResponseEntity`의 본문 부분은 여전히 `HttpMessageConverter`에 의해 직렬화됩니다. 따라서 클라이언트의 `Accept` 헤더, 서버의 컨버터 설정 등이 중요합니다.
4. **빌더 패턴 적극 활용:** `ResponseEntity` 객체를 생성할 때는 가독성과 유연성을 위해 빌더 패턴을 사용하는 것이 일반적이고 권장됩니다.
5. **반응형 타입 지원 (고급):** 원문에서 언급된 `ResponseEntity<Mono<T>>`, `Mono<ResponseEntity<T>>` 등은 비동기 처리가 중요한 고성능 애플리케이션에서 유용하게 사용될 수 있습니다. (지금 단계에서는 이런 것이 가능하다는 정도만 인지)

### **이전 학습 내용과의 연관성**

- **`@ResponseBody`:** `ResponseEntity`는 `@ResponseBody`의 기능을 포함하면서 상태 코드와 헤더 제어 기능을 확장한 것입니다. `@ResponseBody`가 "본문만 신경 쓸게"라면, `ResponseEntity`는 "상태, 헤더, 본문 다 내가 책임질게!" 와 같습니다.
- **`HttpEntity`:** `ResponseEntity`는 `HttpEntity`를 상속받으므로, 헤더와 본문을 다루는 방식은 `HttpEntity`와 유사합니다.
- **`HttpMessageConverter`:** `ResponseEntity`의 본문도 `HttpMessageConverter`에 의해 직렬화되어 클라이언트에게 전달됩니다.
- **`Resource`:** 파일 다운로드 시 `ResponseEntity<Resource>` 형태로 사용되어, `Resource` 객체의 내용이 응답 본문으로 처리되는 것을 보았습니다.

---
