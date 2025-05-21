---
title: Spring Web MVC - Http Message Converter
description: 
author: laze
date: 2025-05-21 00:00:03 +0900
categories: [Dev, SpringWebMVC]
tags: [SpringWebMVC]
---
**HTTP 메시지 변환 (HTTP Message Conversion)**

Reactive 스택에서의 동등한 내용을 확인하세요.

`spring-web` 모듈은 `InputStream`과 `OutputStream`을 통해 HTTP 요청 및 응답의 본문을 읽고 쓰는 `HttpMessageConverter` 인터페이스를 포함합니다.

`HttpMessageConverter` 인스턴스는 클라이언트 측(예: `RestClient`)과 서버 측(예: Spring MVC REST 컨트롤러)에서 사용됩니다.

프레임워크에는 주요 미디어(MIME) 타입에 대한 구체적인 구현체들이 제공되며, 기본적으로 클라이언트 측에서는 `RestClient` 및 `RestTemplate`에,

서버 측에서는 `RequestMappingHandlerAdapter`에 등록됩니다 (메시지 컨버터 설정(Configuring Message Converters) 참조).

여러 `HttpMessageConverter` 구현체들이 아래에 설명되어 있습니다.

전체 목록은 `HttpMessageConverter` Javadoc을 참조하세요.

모든 컨버터에 대해 기본 미디어 타입이 사용되지만, `supportedMediaTypes` 속성을 설정하여 이를 재정의할 수 있습니다.

**표 1. `HttpMessageConverter` 구현체들**

| 메시지 컨버터 (MessageConverter) | 설명 (Description) |
| --- | --- |
| `StringHttpMessageConverter` | HTTP 요청 및 응답으로부터 `String` 인스턴스를 읽고 쓸 수 있는 `HttpMessageConverter` 구현체입니다. 기본적으로 이 컨버터는 모든 텍스트 미디어 타입(`text/*`)을 지원하고 `text/plain`의 `Content-Type`으로 씁니다. |
| `FormHttpMessageConverter` | HTTP 요청 및 응답으로부터 폼 데이터를 읽고 쓸 수 있는 `HttpMessageConverter` 구현체입니다. 기본적으로 이 컨버터는 `application/x-www-form-urlencoded` 미디어 타입을 읽고 씁니다. 폼 데이터는 `MultiValueMap<String, String>`으로 읽고 쓰여집니다. 이 컨버터는 `MultiValueMap<String, Object>`에서 읽은 멀티파트 데이터를 쓸 수도 있습니다(읽기는 불가). 기본적으로 `multipart/form-data`가 지원됩니다. 폼 데이터 쓰기를 위해 추가적인 멀티파트 하위 타입이 지원될 수 있습니다. 자세한 내용은 `FormHttpMessageConverter` javadoc을 참조하세요. |
| `ByteArrayHttpMessageConverter` | HTTP 요청 및 응답으로부터 바이트 배열을 읽고 쓸 수 있는 `HttpMessageConverter` 구현체입니다. 기본적으로 이 컨버터는 모든 미디어 타입(`*/*`)을 지원하고 `application/octet-stream`의 `Content-Type`으로 씁니다. `supportedMediaTypes` 속성을 설정하고 `getContentType(byte[])`을 재정의하여 이를 변경할 수 있습니다. |
| `MarshallingHttpMessageConverter` | `org.springframework.oxm` 패키지의 Spring의 `Marshaller` 및 `Unmarshaller` 추상화를 사용하여 XML을 읽고 쓸 수 있는 `HttpMessageConverter` 구현체입니다. 이 컨버터는 사용되기 전에 `Marshaller`와 `Unmarshaller`가 필요합니다. 생성자나 빈 속성을 통해 이를 주입할 수 있습니다. 기본적으로 이 컨버터는 `text/xml` 및 `application/xml`을 지원합니다. |
| `MappingJackson2HttpMessageConverter` | Jackson의 `ObjectMapper`를 사용하여 JSON을 읽고 쓸 수 있는 `HttpMessageConverter` 구현체입니다. Jackson이 제공하는 어노테이션을 사용하여 필요에 따라 JSON 매핑을 사용자 정의할 수 있습니다. 특정 타입에 대해 사용자 정의 JSON 직렬 변환기/역직렬 변환기를 제공해야 하는 경우와 같이 추가적인 제어가 필요할 때는 `ObjectMapper` 속성을 통해 사용자 정의 `ObjectMapper`를 주입할 수 있습니다. 기본적으로 이 컨버터는 `application/json`을 지원합니다. `com.fasterxml.jackson.core:jackson-databind` 의존성이 필요합니다. |
| `MappingJackson2XmlHttpMessageConverter` | Jackson XML 확장의 `XmlMapper`를 사용하여 XML을 읽고 쓸 수 있는 `HttpMessageConverter` 구현체입니다. JAXB 또는 Jackson이 제공하는 어노테이션을 사용하여 필요에 따라 XML 매핑을 사용자 정의할 수 있습니다. 특정 타입에 대해 사용자 정의 XML 직렬 변환기/역직렬 변환기를 제공해야 하는 경우와 같이 추가적인 제어가 필요할 때는 `ObjectMapper` 속성을 통해 사용자 정의 `XmlMapper`를 주입할 수 있습니다. 기본적으로 이 컨버터는 `application/xml`을 지원합니다. `com.fasterxml.jackson.dataformat:jackson-dataformat-xml` 의존성이 필요합니다. |
| `MappingJackson2CborHttpMessageConverter` | `com.fasterxml.jackson.dataformat:jackson-dataformat-cbor` (의존성) |
| `SourceHttpMessageConverter` | HTTP 요청 및 응답으로부터 `javax.xml.transform.Source`를 읽고 쓸 수 있는 `HttpMessageConverter` 구현체입니다. `DOMSource`, `SAXSource`, `StreamSource`만 지원됩니다. 기본적으로 이 컨버터는 `text/xml` 및 `application/xml`을 지원합니다. |
| `GsonHttpMessageConverter` | "Google Gson"을 사용하여 JSON을 읽고 쓸 수 있는 `HttpMessageConverter` 구현체입니다. `com.google.code.gson:gson` 의존성이 필요합니다. |
| `JsonbHttpMessageConverter` | Jakarta JSON Bind API를 사용하여 JSON을 읽고 쓸 수 있는 `HttpMessageConverter` 구현체입니다. `jakarta.json.bind:jakarta.json.bind-api` 의존성과 사용 가능한 구현체가 필요합니다. |
| `ProtobufHttpMessageConverter` | "application/x-protobuf" 콘텐츠 타입으로 바이너리 형식의 Protobuf 메시지를 읽고 쓸 수 있는 `HttpMessageConverter` 구현체입니다. `com.google.protobuf:protobuf-java` 의존성이 필요합니다. |
| `ProtobufJsonFormatHttpMessageConverter` | Protobuf 메시지와 JSON 문서 간에 읽고 쓸 수 있는 `HttpMessageConverter` 구현체입니다. `com.google.protobuf:protobuf-java-util` 의존성이 필요합니다. |

---

## Spring MVC의 데이터 마법사: HttpMessageConverter 이해하기

웹 애플리케이션, 특히 REST API에서는 클라이언트와 서버가 다양한 형식의 데이터를 HTTP 요청/응답 본문을 통해 주고받습니다.

예를 들어 클라이언트는 JSON 형식으로 데이터를 서버에 보낼 수 있고, 서버는 Java 객체를 JSON이나 XML 형식으로 변환하여 클라이언트에게 응답할 수 있습니다.

Spring MVC는 이러한 **데이터 형식 변환**을 **`HttpMessageConverter`** 인터페이스를 통해 처리합니다.

### 1. `HttpMessageConverter<T>` 인터페이스

- **역할:** HTTP 요청 본문(`InputStream`)을 특정 타입의 Java 객체(`T`)로 읽어오거나(역직렬화, Deserialization), Java 객체(`T`)를 HTTP 응답 본문(`OutputStream`)으로 쓰는(직렬화, Serialization) 역할을 하는 전략 인터페이스입니다.
- **주요 메소드:**
  - `boolean canRead(Class<?> clazz, MediaType mediaType)`: 주어진 Java 클래스 타입(`clazz`)과 미디어 타입(`mediaType`)의 요청 본문을 읽을 수 있는지(역직렬화 가능한지) 여부를 반환합니다.
  - `boolean canWrite(Class<?> clazz, MediaType mediaType)`: 주어진 Java 클래스 타입(`clazz`)의 객체를 특정 미디어 타입(`mediaType`)의 응답 본문으로 쓸 수 있는지(직렬화 가능한지) 여부를 반환합니다.
  - `List<MediaType> getSupportedMediaTypes()`: 이 컨버터가 지원하는 미디어 타입 목록을 반환합니다.
  - `T read(Class<? extends T> clazz, HttpInputMessage inputMessage) throws IOException, HttpMessageNotReadableException`: `HttpInputMessage` (HTTP 요청 헤더 및 본문 접근 가능)로부터 데이터를 읽어 `clazz` 타입의 객체로 변환합니다.
  - `void write(T t, MediaType contentType, HttpOutputMessage outputMessage) throws IOException, HttpMessageNotWritableException`: 객체 `t`를 `contentType`으로 지정된 미디어 타입으로 변환하여 `HttpOutputMessage` (HTTP 응답 헤더 및 본문 접근 가능)에 씁니다.
- **사용처:**
  - **서버 측 (Spring MVC REST 컨트롤러):**
    - `@RequestBody` 어노테이션이 붙은 파라미터: 요청 본문을 해당 파라미터 타입의 Java 객체로 변환할 때 사용됩니다.
    - `@ResponseBody` 어노테이션이 붙은 메소드 또는 `ResponseEntity`를 반환하는 메소드: 메소드가 반환한 Java 객체를 응답 본문으로 변환할 때 사용됩니다.
    - 이러한 작업은 주로 `RequestMappingHandlerAdapter` 내부에서 `HttpMessageConverter`들을 통해 이루어집니다.
  - **클라이언트 측 (`RestTemplate`, `WebClient`, `RestClient`):**
    - HTTP 요청 시 Java 객체를 요청 본문으로 변환하거나, HTTP 응답 본문을 Java 객체로 변환할 때 사용됩니다.

### 2. 주요 `HttpMessageConverter` 구현체들

Spring 프레임워크는 다양한 데이터 형식과 라이브러리를 지원하기 위해 여러 `HttpMessageConverter` 구현체들을 기본으로 제공합니다. 이들은 특정 의존성이 클래스패스에 존재하면 자동으로 등록되기도 합니다.

| `HttpMessageConverter` 구현체 | 주요 역할 및 특징 | 기본 지원 미디어 타입 | 필요 의존성 (예시) |
| --- | --- | --- | --- |
| `StringHttpMessageConverter` | `String` 타입과 HTTP 텍스트 본문 간 변환. | `text/*` (기본: `text/plain`) | 없음 (Spring 기본) |
| `FormHttpMessageConverter` | `application/x-www-form-urlencoded` 형식의 폼 데이터를 `MultiValueMap<String, String>`으로 변환. `multipart/form-data` 쓰기도 지원( `MultiValueMap<String, Object>` 사용). | `application/x-www-form-urlencoded`, `multipart/form-data` (쓰기) | 없음 (Spring 기본) |
| `ByteArrayHttpMessageConverter` | `byte[]` (바이트 배열)과 HTTP 본문 간 변환. 주로 바이너리 데이터(이미지, 파일 등) 처리에 사용. | `*/*` (기본: `application/octet-stream`) | 없음 (Spring 기본) |
| `MappingJackson2HttpMessageConverter` | **(JSON 처리 시 가장 중요)** Jackson 라이브러리의 `ObjectMapper`를 사용하여 Java 객체와 JSON 간 변환. `@JsonIgnore`, `@JsonProperty` 등 Jackson 어노테이션 지원. | `application/json` | `com.fasterxml.jackson.core:jackson-databind` |
| `MappingJackson2XmlHttpMessageConverter` | Jackson XML 확장의 `XmlMapper`를 사용하여 Java 객체와 XML 간 변환. JAXB 또는 Jackson XML 어노테이션 지원. | `application/xml` | `com.fasterxml.jackson.dataformat:jackson-dataformat-xml` |
| `MarshallingHttpMessageConverter` | Spring의 `Marshaller`/`Unmarshaller` (JAXB, Castor, XStream 등 지원)를 사용하여 Java 객체와 XML 간 변환. | `text/xml`, `application/xml` | `org.springframework:spring-oxm` 및 해당 OXM 라이브러리 |
| `GsonHttpMessageConverter` | Google Gson 라이브러리를 사용하여 Java 객체와 JSON 간 변환. | `application/json` | `com.google.code.gson:gson` |
| `JsonbHttpMessageConverter` | Jakarta JSON Bind API (JSON-B)를 사용하여 Java 객체와 JSON 간 변환. | `application/json` | `jakarta.json.bind:jakarta.json.bind-api` 및 구현체 |
| `ProtobufHttpMessageConverter` | Google Protocol Buffers 메시지를 바이너리 형식(`application/x-protobuf`)으로 변환. | `application/x-protobuf` | `com.google.protobuf:protobuf-java` |
| `ProtobufJsonFormatHttpMessageConverter` | Protocol Buffers 메시지를 JSON 형식으로 변환. (Protobuf의 JSON 매핑 표준 사용) | `application/json` | `com.google.protobuf:protobuf-java-util` |
| `SourceHttpMessageConverter` | `javax.xml.transform.Source` (XML 처리용) 객체와 XML 본문 간 변환. (`DOMSource`, `SAXSource`, `StreamSource` 지원) | `text/xml`, `application/xml` | 없음 (JDK 기본) |

**컨버터 선택 및 등록:**

- `RequestMappingHandlerAdapter` (서버 측)나 `RestTemplate` (클라이언트 측)은 여러 `HttpMessageConverter` 인스턴스를 목록으로 가지고 있습니다.
- 요청/응답을 처리할 때, 이 목록을 순회하며 각 컨버터의 `canRead()` 또는 `canWrite()` 메소드를 호출하여 현재 상황(변환할 Java 타입, 요청/응답의 `Content-Type`)에 가장 적합한 컨버터를 선택하여 사용합니다.
- Spring Boot를 사용하면 클래스패스에 특정 라이브러리(예: Jackson)가 존재할 경우 관련된 `HttpMessageConverter`가 **자동으로 등록**됩니다.
- 필요하다면 `WebMvcConfigurer`의 `configureMessageConverters(List<HttpMessageConverter<?>> converters)` 또는 `extendMessageConverters(List<HttpMessageConverter<?>> converters)` 메소드를 오버라이드하여 기본 컨버터 목록을 수정하거나 새로운 커스텀 컨버터를 추가/교체할 수 있습니다. (메시지 컨버터 설정 챕터에서 자세히 다룸)

### 3. `HttpMessageConverter` 사용 예시 (서버 측 REST 컨트롤러)

**요청 본문을 Java 객체로 변환 (`@RequestBody`):**

```java
// User.java (데이터 객체)
public class User {
    private String username;
    private int age;
    // 생성자, getter, setter 생략
}

// UserController.java
import org.springframework.http.HttpStatus;
import org.springframework.http.ResponseEntity;
import org.springframework.web.bind.annotation.PostMapping;
import org.springframework.web.bind.annotation.RequestBody;
import org.springframework.web.bind.annotation.RestController;

@RestController
public class UserController {

    @PostMapping("/users")
    public ResponseEntity<String> createUser(@RequestBody User user) { // 요청 본문의 JSON/XML이 User 객체로 변환됨
        // 1. 클라이언트가 요청 헤더에 Content-Type: application/json 을 보내고,
        //    본문에 {"username":"john", "age":30} 과 같은 JSON 데이터를 보냈다고 가정.
        // 2. RequestMappingHandlerAdapter는 등록된 HttpMessageConverter 목록을 확인.
        // 3. MappingJackson2HttpMessageConverter가 User.class 타입과 application/json 미디어 타입을
        //    canRead() 할 수 있다고 판단.
        // 4. MappingJackson2HttpMessageConverter의 read() 메소드가 호출되어 JSON 본문을 User 객체로 변환.
        // 5. 변환된 User 객체가 createUser 메소드의 user 파라미터로 전달됨.

        System.out.println("Received user: " + user.getUsername() + ", Age: " + user.getAge());
        return new ResponseEntity<>("User created: " + user.getUsername(), HttpStatus.CREATED);
    }
}
```

**Java 객체를 응답 본문으로 변환 (`@ResponseBody` 또는 `ResponseEntity`):**

```java
// UserController.java (계속)
import org.springframework.web.bind.annotation.GetMapping;
import org.springframework.web.bind.annotation.PathVariable;

@RestController // 클래스 레벨에 @RestController를 사용하면 모든 메소드에 @ResponseBody가 적용된 효과
public class UserController {

    // ... createUser 메소드 ...

    @GetMapping("/users/{id}")
    public User getUser(@PathVariable String id) { // 반환된 User 객체가 JSON/XML 등으로 변환되어 응답 본문이 됨
        // 1. 이 메소드가 User 객체를 반환.
        // 2. RequestMappingHandlerAdapter는 클라이언트의 Accept 헤더 (예: application/json)와
        //    User.class 타입을 보고 적절한 HttpMessageConverter를 찾음.
        // 3. MappingJackson2HttpMessageConverter가 User.class 타입을 application/json 미디어 타입으로
        //    canWrite() 할 수 있다고 판단.
        // 4. MappingJackson2HttpMessageConverter의 write() 메소드가 호출되어 User 객체를 JSON 문자열로 변환.
        // 5. 변환된 JSON 문자열이 HTTP 응답 본문으로 클라이언트에게 전송됨.

        User user = new User(); // 실제로는 DB 등에서 조회
        user.setUsername("User" + id);
        user.setAge(Integer.parseInt(id) + 20);
        return user;
    }

    @GetMapping("/users/rawstring")
    public String getRawString() {
        // StringHttpMessageConverter가 사용되어 "text/plain"으로 응답.
        return "This is a raw string response.";
    }
}
```

### 4. `supportedMediaTypes` 재정의

모든 `HttpMessageConverter`는 기본적으로 지원하는 미디어 타입이 있지만, `setSupportedMediaTypes(List<MediaType> supportedMediaTypes)` 메소드를 통해 이를 변경하거나 추가할 수 있습니다.

예를 들어, `StringHttpMessageConverter`가 기본적으로 `text/plain` 외에 `application/custom-text`도 처리하도록 하고 싶다면, 빈 설정 시 이 프로퍼티를 지정할 수 있습니다.

```java
// WebConfig.java (StringHttpMessageConverter 커스터마이징 예시)
@Configuration
public class WebConfig implements WebMvcConfigurer {

    @Override
    public void configureMessageConverters(List<HttpMessageConverter<?>> converters) {
        StringHttpMessageConverter stringConverter = new StringHttpMessageConverter();
        stringConverter.setSupportedMediaTypes(Arrays.asList(
                new MediaType("text", "plain", StandardCharsets.UTF_8),
                new MediaType("application", "custom-text", StandardCharsets.UTF_8)
        ));
        converters.add(stringConverter); // 기존 컨버터 목록에 추가 (또는 교체)
    }
}
```

`HttpMessageConverter`는 Spring MVC에서 데이터를 다양한 형식으로 유연하게 주고받을 수 있도록 하는 매우 강력하고 핵심적인 메커니즘입니다. 어떤 컨버터들이 있고, 각각 어떤 역할을 하는지, 그리고 언제 사용되는지를 이해하면 REST API 등을 개발할 때 큰 도움이 됩니다.
