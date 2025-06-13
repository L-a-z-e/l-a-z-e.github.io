---
title: Spring Web MVC - Annotated Controllers (@ResponseBody)
description: 
author: laze
date: 2025-06-13 00:00:01 +0900
categories: [Dev, SpringWebMVC]
tags: [SpringWebMVC]
---
### @ResponseBody

메소드에 `@ResponseBody` 어노테이션을 사용하면 반환 값이 `HttpMessageConverter`를 통해 응답 본문(response body)으로 직렬화(serialized)되도록 할 수 있습니다.

**Java**

```java
@GetMapping("/accounts/{id}")
@ResponseBody // 이 메소드의 반환 값(Account 객체)이 응답 본문이 됨
public Account handle() {
	// ... Account 객체를 조회하거나 생성하는 로직 ...
	return new Account(); // 이 Account 객체가 JSON/XML 등으로 변환되어 응답 본문에 쓰여짐
}
```

`@ResponseBody`는 클래스 레벨에서도 지원되며, 이 경우 모든 컨트롤러 메소드에 상속됩니다.

이것이 바로 `@RestController`의 효과인데, `@RestController`는 `@Controller`와 `@ResponseBody`로 마킹된 메타 어노테이션에 지나지 않습니다.

파일 내용을 위해 `Resource` 객체를 반환할 수 있으며, 제공된 리소스의 `InputStream` 내용이 응답 `OutputStream`으로 복사됩니다.

`InputStream`은 응답으로 복사된 후 안정적으로 닫히기 위해 리소스 핸들에 의해 지연(lazily) 검색되어야 한다는 점에 유의하세요.

이러한 목적으로 `InputStreamResource`를 사용하는 경우, (실제 `InputStream`을 검색하는 람다 표현식 등을 통해) 필요시점에 `InputStream`을 가져오는 `InputStreamSource`로 생성해야 합니다.

`@ResponseBody`는 반응형 타입(reactive types)과 함께 사용할 수 있습니다.

MVC 설정의 메시지 컨버터(Message Converters) 옵션을 사용하여 메시지 변환을 구성하거나 사용자 정의할 수 있습니다.

`@ResponseBody` 메소드를 JSON 직렬화 뷰(JSON serialization views)와 결합할 수 있습니다.

---

### **학습 목표**

이 챕터를 학습하고 나면, 학생은 다음을 이해하고 수행할 수 있게 됩니다:

1. **`@ResponseBody` 어노테이션의 핵심 역할 이해:** 컨트롤러 메소드의 반환 값을 HTTP 응답 본문으로 직접 변환(직렬화)하는 `@ResponseBody`의 기능과 주된 사용 사례(특히 JSON/XML 기반의 API 응답)를 이해합니다.
2. **`HttpMessageConverter`와의 관계 파악 (응답 측면):** `@ResponseBody`가 자바 객체를 응답 본문으로 변환할 때 `HttpMessageConverter`가 어떤 역할을 하는지, 그리고 이 변환 과정을 어떻게 커스터마이징할 수 있는지 개념적으로 이해합니다.
3. **`@RestController` 어노테이션의 의미와 효과 이해:** `@RestController`가 `@Controller`와 `@ResponseBody`의 조합이라는 것을 이해하고, 클래스 레벨에 적용 시 모든 핸들러 메소드에 `@ResponseBody`가 적용되는 효과를 인지합니다.
4. **다양한 반환 타입 처리 가능성 인지:** 단순 객체뿐만 아니라 `Resource` 객체(파일 다운로드) 등을 반환하여 응답 본문을 구성할 수 있음을 이해합니다. (반응형 타입, JSON 직렬화 뷰는 심화 내용으로 간단히 인지만 합니다.)

---

### **핵심 개념 설명**

**`@ResponseBody`란 무엇일까요?**

우리가 웹 애플리케이션을 만들 때, 컨트롤러 메소드는 보통 뷰(View)의 이름(예: `"userProfilePage"`)을 반환하여 해당 뷰가 사용자에게 보여지도록 합니다.

그러면 스프링 MVC는 모델(Model)에 담긴 데이터를 사용하여 HTML 페이지 등을 렌더링해서 응답합니다.

하지만, 항상 HTML 페이지를 응답으로 보내는 것은 아닙니다.

특히 REST API와 같이 프로그램 간 통신을 할 때는, **데이터 자체(예: JSON, XML 형식의 데이터)를 HTTP 응답 본문(response body)에 직접 담아 보내야 하는 경우**가 많습니다.

`@ResponseBody` 어노테이션은 바로 이러한 요구사항을 해결해줍니다.

컨트롤러 메소드 또는 클래스 레벨에 이 어노테이션을 붙이면, 해당 메소드가 반환하는 **자바 객체나 값이 뷰를 통해 렌더링되지 않고, HTTP 응답 본문에 직접 쓰여지도록** 합니다.

이때, 반환된 자바 객체는 `HttpMessageConverter`에 의해 적절한 형식(예: JSON, XML, 또는 단순 텍스트)으로 **직렬화(serialization)**됩니다.

**비유:**

여러분이 친구에게 선물을 보낸다고 상상해봅시다.

- **일반적인 컨트롤러 응답 (뷰 이름 반환):** 친구에게 "선물 포장지 디자인 번호 3번(`뷰 이름`)으로 포장해서, 안에 이 편지(`Model` 데이터)를 넣어줘" 라고 배송업체(스프링 MVC)에 요청하는 것과 같습니다. 배송업체는 디자인에 맞춰 예쁘게 포장된 선물(HTML 응답)을 친구에게 전달합니다.
- **`@ResponseBody` 사용 시:** 친구에게 **선물 내용물 자체(반환된 자바 객체)**를 투명한 비닐봉투(HTTP 응답 본문)에 담아 바로 보내는 것과 같습니다. 포장(뷰 렌더링) 과정 없이 내용물(데이터)이 그대로 전달되는 것이죠. 친구는 비닐봉투를 열어 내용물(JSON/XML 데이터)을 바로 확인할 수 있습니다.

**예시:**

클라이언트가 `/api/users/1` 로 GET 요청을 보내 특정 사용자 정보를 JSON 형태로 받고 싶다고 가정해봅시다.

**User 자바 클래스:**

```java
public class User {
    private Long id;
    private String username;
    private String email;
    // Getters and Setters
}
```

**컨트롤러 메소드:**

```java
import org.springframework.stereotype.Controller;
import org.springframework.web.bind.annotation.GetMapping;
import org.springframework.web.bind.annotation.PathVariable;
import org.springframework.web.bind.annotation.ResponseBody;

@Controller // 1. 일반 컨트롤러로 선언
public class UserController {

    // 가상의 UserService
    private UserService userService = new UserServiceImpl();

    @GetMapping("/api/users/{id}")
    @ResponseBody // 2. 이 메소드의 반환 값은 응답 본문으로 직접 직렬화됨
    public User getUserById(@PathVariable Long id) {
        User user = userService.findById(id);
        // 3. 여기서 반환된 User 객체가 JSON 등으로 변환되어 응답 본문에 담김
        return user;
    }
}
```

만약 `id`가 1인 사용자의 정보가 `User(id=1, username="springFan", email="fan@example.com")`이라면, 클라이언트는 다음과 유사한 JSON 응답을 받게 됩니다 (실제 HTTP 응답 헤더에는 `Content-Type: application/json` 등이 포함됩니다):

```json
{
  "id": 1,
  "username": "springFan",
  "email": "fan@example.com"
}
```

**핵심 도우미: `HttpMessageConverter` (다시 등장!)**

`@RequestBody`와 마찬가지로, `@ResponseBody`의 마법 뒤에도 **`HttpMessageConverter`** 가 있습니다.

- `@ResponseBody`가 붙은 메소드가 자바 객체를 반환하면, 스프링 MVC는 클라이언트가 요청 시 보낸 `Accept` 헤더(예: `application/json`, `application/xml`)와 반환되는 객체의 타입, 그리고 등록된 `HttpMessageConverter`들을 종합적으로 고려합니다.
- **적절한 `HttpMessageConverter`를 선택하여, 반환된 자바 객체를 클라이언트가 요청한 형식(예: JSON, XML)의 문자열로 직렬화**하고, 이를 HTTP 응답 본문에 씁니다.
- 이때 응답 헤더의 `Content-Type`도 선택된 `HttpMessageConverter`에 의해 적절하게 설정됩니다. (예: JSON으로 변환되면 `Content-Type: application/json`)

**`@RestController` 어노테이션**

매번 메소드마다 `@ResponseBody`를 붙이는 것이 번거로울 수 있습니다. 만약 컨트롤러의 **모든 메소드**가 뷰 이름 대신 응답 본문에 직접 데이터를 쓰는 REST API 스타일이라면, 클래스 레벨에 `@RestController` 어노테이션을 사용할 수 있습니다.

`@RestController`는 사실 다음과 같이 두 어노테이션을 합쳐놓은 편리한 메타 어노테이션입니다:

- `@Controller`: 이 클래스가 스프링 MVC 컨트롤러임을 나타냅니다.
- `@ResponseBody`: 이 클래스 내의 모든 핸들러 메소드에 `@ResponseBody`의 효과가 적용되도록 합니다. (즉, 모든 메소드의 반환 값은 응답 본문으로 직렬화됩니다.)

```java
import org.springframework.web.bind.annotation.GetMapping;
import org.springframework.web.bind.annotation.PathVariable;
import org.springframework.web.bind.annotation.RestController; // @RestController 사용

@RestController // 1. @Controller + @ResponseBody
public class UserRestController {

    private UserService userService = new UserServiceImpl();

    @GetMapping("/api/v2/users/{id}")
    // 2. @ResponseBody를 생략해도 동일하게 동작
    public User getUserById(@PathVariable Long id) {
        User user = userService.findById(id);
        return user;
    }

    @GetMapping("/api/v2/users")
    public List<User> getAllUsers() {
        List<User> users = userService.findAll();
        // List<User> 객체가 JSON 배열 등으로 변환되어 응답 본문에 담김
        return users;
    }
}
```

### **주요 용어 해설 (복습 및 추가)**

- **응답 본문 (Response Body):** HTTP 응답 메시지에서 헤더 다음에 오는 부분으로, 서버가 클라이언트에게 전달하고자 하는 실제 데이터를 담고 있습니다.
- **직렬화 (Serialization):** 메모리상의 객체(자바 객체)를 특정 형식(예: JSON, XML 문자열)으로 변환하는 과정. `@ResponseBody`에서 일어나는 핵심 작업입니다.
- **`HttpMessageConverter`:** (다시 한번 강조!) HTTP 요청/응답 본문과 자바 객체 간의 변환을 담당합니다. `@ResponseBody`에서는 객체 -> 특정 형식 문자열(또는 바이트 스트림) 변환을 수행합니다.
- **`@RestController`:** `@Controller`와 `@ResponseBody` 어노테이션을 결합한 편의 어노테이션. 클래스 내 모든 핸들러 메소드의 반환 값을 응답 본문으로 처리합니다. 주로 REST API 컨트롤러를 작성할 때 사용됩니다.
- **`Resource`:** 스프링에서 파일, URL, 클래스패스 상의 리소스 등 다양한 종류의 저수준 리소스에 접근하기 위한 추상화 인터페이스. 파일 다운로드와 같은 기능을 구현할 때 `@ResponseBody`와 함께 사용될 수 있습니다.
- **`InputStreamResource`:** `InputStream`을 감싸서 스프링 `Resource`로 만들어주는 구현체. 파일 다운로드 시 유용하지만, 스트림을 지연 로딩하고 사용 후 잘 닫도록 주의해야 합니다.

### **코드 예제 및 분석**

**1. `@ResponseBody`와 `@RestController`**

**2. 파일 다운로드를 위한 `Resource` 반환**

만약 서버에 있는 특정 파일을 클라이언트가 다운로드하도록 하고 싶다면, 컨트롤러 메소드가 `Resource` 타입의 객체를 반환하고 `@ResponseBody`를 사용할 수 있습니다.

```java
import org.springframework.core.io.InputStreamResource;
import org.springframework.core.io.Resource;
import org.springframework.http.HttpHeaders;
import org.springframework.http.MediaType;
import org.springframework.http.ResponseEntity;
import org.springframework.web.bind.annotation.GetMapping;
import org.springframework.web.bind.annotation.PathVariable;
import org.springframework.web.bind.annotation.RestController;

import java.io.File;
import java.io.FileInputStream;
import java.io.FileNotFoundException;
import java.nio.file.Path;
import java.nio.file.Paths;

@RestController
public class FileDownloadController {

    private final Path fileStorageLocation = Paths.get("./uploads").toAbsolutePath().normalize();

    @GetMapping("/downloadFile/{fileName:.+}") // :.+ 는 파일 확장자까지 포함하기 위함
    public ResponseEntity<Resource> downloadFile(@PathVariable String fileName) {
        try {
            Path filePath = this.fileStorageLocation.resolve(fileName).normalize();
            File file = filePath.toFile();

            if (!file.exists() || !file.isFile()) {
                // 파일을 찾을 수 없는 경우 404 응답
                return ResponseEntity.notFound().build();
            }

            // InputStream을 지연 로딩하기 위해 InputStreamSource 사용 (람다 표현식)
            InputStreamResource resource = new InputStreamResource(() -> {
                try {
                    return new FileInputStream(file);
                } catch (FileNotFoundException e) {
                    // 실제로는 더 적절한 예외 처리 필요
                    throw new RuntimeException("Error reading file", e);
                }
            });

            // 다운로드될 파일명 설정 (Content-Disposition 헤더)
            // 한글 파일명 등을 위해 인코딩 처리 필요할 수 있음
            String contentType = "application/octet-stream"; // 일반적인 바이너리 파일 타입
            // 파일 확장자에 따라 적절한 MediaType 설정 가능 (예: Files.probeContentType(filePath))

            return ResponseEntity.ok()
                    .contentType(MediaType.parseMediaType(contentType))
                    .header(HttpHeaders.CONTENT_DISPOSITION, "attachment; filename=\\"" + file.getName() + "\\"") // 다운로드 대화상자 표시
                    .contentLength(file.length()) // 파일 크기 설정
                    .body(resource); // Resource 객체를 응답 본문으로

        } catch (Exception ex) {
            // 오류 처리
            return ResponseEntity.internalServerError().build();
        }
    }
}

```

**코드 분석:**

- `@GetMapping("/downloadFile/{fileName:.+}")`: 파일 다운로드 요청을 처리합니다.
- `ResponseEntity<Resource>`: 응답 상태 코드, 헤더, 그리고 본문(`Resource`)을 함께 제어하기 위해 `ResponseEntity`를 사용했습니다. `@ResponseBody`는 `ResponseEntity`의 본문 처리에도 동일하게 적용됩니다.
- `InputStreamResource`: 실제 파일로부터 `FileInputStream`을 생성하고, 이를 `InputStreamResource`로 감싸서 반환합니다. 원문에서 언급된 것처럼, `InputStream`은 지연 로딩(필요할 때 생성)하고 사용 후 안정적으로 닫히도록 하는 것이 중요합니다. 람다 표현식을 사용한 `InputStreamSource`가 이를 돕습니다.
- `Content-Disposition` 헤더: `attachment; filename="파일명"`으로 설정하면 브라우저가 이 응답을 파일로 다운로드하도록 합니다.
- `body(resource)`: `Resource` 객체가 응답 본문으로 사용됩니다. 스프링은 이 `Resource`의 내용을 읽어 응답 스트림으로 복사합니다.

### **"왜?" 라는 질문에 대한 답변**

**`@ResponseBody`는 왜 필요할까요? 그냥 서블릿의 `response.getWriter().print(...)`를 쓰면 안 되나?**

1. **추상화 및 편의성:** 서블릿 API를 직접 사용하면 `Content-Type` 설정, 문자 인코딩, 객체의 문자열 변환(JSON/XML 직렬화) 등을 모두 수동으로 처리해야 합니다. `@ResponseBody`와 `HttpMessageConverter`는 이 모든 복잡한 과정을 추상화하여 개발자가 비즈니스 로직에만 집중할 수 있도록 해줍니다. 자바 객체만 반환하면 나머지는 스프링이 알아서 해줍니다.
2. **`Content Negotiation` (내용 협상) 지원:** 클라이언트는 `Accept` 헤더를 통해 자신이 원하는 응답 형식을 서버에 알릴 수 있습니다 (예: `Accept: application/json` 또는 `Accept: application/xml`). `@ResponseBody`는 등록된 `HttpMessageConverter`들과 `Accept` 헤더를 고려하여, 클라이언트가 원하는 형식(또는 서버가 지원하는 최적의 형식)으로 데이터를 변환하여 제공할 수 있는 기반을 마련합니다. 수동으로는 이런 유연한 처리가 어렵습니다.
3. **테스트 용이성:** 컨트롤러 메소드가 특정 뷰 기술이나 서블릿 API에 직접 의존하지 않고, 순수 자바 객체를 반환하게 되므로 단위 테스트가 더 용이해집니다. 반환된 객체의 내용만 검증하면 됩니다.
4. **RESTful API 구현의 표준 방식:** 현대적인 REST API는 대부분 JSON이나 XML 형태로 데이터를 주고받습니다. `@ResponseBody`는 이러한 API를 스프링 MVC로 구현하는 표준적이고 효율적인 방법을 제공합니다.

**`@RestController`는 왜 등장했을까요?**

REST API를 개발할 때는 거의 모든 컨트롤러 메소드가 `@ResponseBody`의 기능을 필요로 합니다. 매번 메소드마다 `@ResponseBody`를 붙이는 것은 반복적이고 코드를 번잡하게 만들 수 있습니다. `@RestController`는 이러한 반복을 줄이고, "이 컨트롤러는 REST API를 위한 것이며, 모든 메소드의 반환 값은 응답 본문으로 직접 처리된다"는 의도를 명확하게 표현하기 위해 등장한 편의 어노테이션입니다.

### **주의사항 및 Best Practice**

1. **`Accept` 헤더와 `HttpMessageConverter`:** 클라이언트가 보내는 `Accept` 헤더와 서버에 등록된 `HttpMessageConverter`가 서로 호환되어야 원하는 형식으로 응답을 받을 수 있습니다. 예를 들어, 클라이언트가 `Accept: application/xml`을 보냈는데 서버에 XML 컨버터가 없다면 오류가 발생하거나 원치 않는 형식(예: JSON)으로 응답이 갈 수 있습니다.
2. **반환 객체의 직렬화 가능성:** 반환하는 자바 객체가 선택된 `HttpMessageConverter`에 의해 제대로 직렬화될 수 있어야 합니다. 예를 들어 Jackson을 사용하여 JSON으로 변환한다면, 객체의 필드에 접근 가능해야 하고(public 필드 또는 getter), 순환 참조 등이 없도록 주의해야 합니다.
3. **에러 응답 처리:** REST API에서는 성공 응답뿐만 아니라 에러 상황에 대한 응답도 명확한 형식(예: JSON 에러 객체)으로 제공하는 것이 중요합니다. `@ControllerAdvice`와 `@ExceptionHandler`를 사용하여 예외를 처리하고, `@ResponseBody`를 통해 일관된 에러 응답 본문을 생성할 수 있습니다.
4. **`ResponseEntity` 활용:** 응답 상태 코드, 헤더, 본문을 보다 세밀하게 제어하고 싶을 때는 단순히 객체를 반환하는 대신 `ResponseEntity` 객체를 생성하여 반환하는 것이 좋습니다. `ResponseEntity`의 본문도 `@ResponseBody`와 동일한 방식으로 처리됩니다.

### **이전 학습 내용과의 연관성**

- **`@RequestBody`:** `@RequestBody`가 요청 본문을 객체로 변환하는 것이었다면, `@ResponseBody`는 반대로 객체를 응답 본문으로 변환합니다. 둘은 데이터 흐름의 방향만 다를 뿐, `HttpMessageConverter`를 사용한다는 핵심 원리는 동일합니다. 이 둘은 REST API 개발의 핵심적인 한 쌍입니다.
- **`HttpMessageConverter`:** 이전에 `@RequestBody`와 `@RequestPart`에서 중요하게 다루어졌던 `HttpMessageConverter`가 `@ResponseBody`에서도 동일하게 핵심적인 역할을 수행합니다. 요청 처리와 응답 처리 모두에서 데이터 변환을 담당하는 만능 해결사 같은 존재입니다.
- **`HttpEntity`와 `ResponseEntity`:** `HttpEntity`가 요청의 헤더와 본문을 캡슐화했다면, `ResponseEntity`는 `HttpEntity`를 상속받아 응답의 상태 코드까지 추가로 포함합니다. `@ResponseBody`는 `ResponseEntity`의 본문 부분을 직렬화하는 데 사용됩니다.

---
