---
title: Spring Web MVC - Annotated Controllers (Multipart)
description: 
author: laze
date: 2025-06-10 00:00:01 +0900
categories: [Dev, SpringWebMVC]
tags: [SpringWebMVC]
---
### Multipart (멀티파트)

> Reactive 스택에서의 동일 기능은 해당 문서를 참고하세요.
>

`MultipartResolver`가 활성화된 후에는, `multipart/form-data` 타입의 POST 요청 내용은 파싱되어 일반적인 요청 파라미터처럼 접근할 수 있게 됩니다.

다음 예제는 일반적인 폼 필드 하나와 업로드된 파일 하나에 접근하는 방법을 보여줍니다:

**Java**

```java
@Controller
public class FileUploadController {

	@PostMapping("/form")
	public String handleFormUpload(@RequestParam("name") String name,
			@RequestParam("file") MultipartFile file) {

		if (!file.isEmpty()) {
			byte[] bytes = file.getBytes(); // 파일 내용을 바이트 배열로 가져옴
			// 어딘가에 바이트 데이터를 저장합니다
			return "redirect:uploadSuccess";
		}
		return "redirect:uploadFailure";
	}
}
```

- 인자 타입을 `List<MultipartFile>`로 선언하면 동일한 파라미터 이름으로 여러 파일을 처리할 수 있습니다.

`@RequestParam` 어노테이션이 어노테이션에 파라미터 이름이 명시되지 않은 채 `Map<String, MultipartFile>` 또는 `MultiValueMap<String, MultipartFile>`로 선언되면,

해당 맵은 각 주어진 파라미터 이름에 대한 멀티파트 파일들로 채워집니다.

서블릿 멀티파트 파싱을 사용하면, 메소드 인자나 컬렉션 값 타입으로 스프링의 `MultipartFile` 대신 `jakarta.servlet.http.Part`를 선언할 수도 있습니다.

멀티파트 내용을 커맨드 객체(command object)에 데이터 바인딩하는 일부로 사용할 수도 있습니다.

예를 들어, 이전 예제의 폼 필드와 파일은 다음 예제와 같이 폼 객체의 필드가 될 수 있습니다:

**Java**

```java
class MyForm {

	private String name;

	private MultipartFile file;

	// ... getter, setter 등
}

@Controller
public class FileUploadController {

	@PostMapping("/form")
	public String handleFormUpload(MyForm form, BindingResult errors) { // MyForm 객체로 데이터 바인딩
		if (!form.getFile().isEmpty()) {
			byte[] bytes = form.getFile().getBytes();
			// 어딘가에 바이트 데이터를 저장합니다
			return "redirect:uploadSuccess";
		}
		return "redirect:uploadFailure";
	}
}
```

멀티파트 요청은 RESTful 서비스 시나리오에서 브라우저가 아닌 클라이언트로부터 제출될 수도 있습니다.

다음 예제는 JSON과 함께 파일을 보여줍니다:

```
POST /someUrl
Content-Type: multipart/mixed

--edt7Tfrdusa7r3lNQc79vXuhIIMlatb7PQg7Vp
Content-Disposition: form-data; name="meta-data"
Content-Type: application/json; charset=UTF-8
Content-Transfer-Encoding: 8bit

{
	"name": "value"
}
--edt7Tfrdusa7r3lNQc79vXuhIIMlatb7PQg7Vp
Content-Disposition: form-data; name="file-data"; filename="file.properties"
Content-Type: text/xml
Content-Transfer-Encoding: 8bit
... 파일 데이터 ...

```

"meta-data" 부분을 `@RequestParam`을 사용하여 `String`으로 접근할 수 있지만, 아마도 `@RequestBody`와 유사하게 JSON으로부터 역직렬화(deserialized)되기를 원할 것입니다.

`@RequestPart` 어노테이션을 사용하면 `HttpMessageConverter`를 통해 변환된 후 멀티파트에 접근할 수 있습니다:

**Java**

```java
@PostMapping("/")
public String handle(@RequestPart("meta-data") MetaData metadata, // "meta-data" 부분을 MetaData 객체로 변환
		@RequestPart("file-data") MultipartFile file) { // "file-data" 부분을 MultipartFile로 받음
	// ...
}
```

`@RequestPart`는 `jakarta.validation.Valid` 또는 스프링의 `@Validated` 어노테이션과 함께 사용할 수 있으며, 둘 다 표준 빈 검증(Standard Bean Validation)이 적용되도록 합니다.

기본적으로 검증 오류는 `MethodArgumentNotValidException`을 발생시키며, 이는 400 (BAD_REQUEST) 응답으로 변환됩니다.

또는, 다음 예제와 같이 `Errors` 또는 `BindingResult` 인자를 통해 컨트롤러 내에서 지역적으로 검증 오류를 처리할 수 있습니다:

**Java**

```java
@PostMapping("/")
public String handle(@Valid @RequestPart("meta-data") MetaData metadata, Errors errors) {
	// ... errors 객체를 통해 검증 오류 확인
}
```

만약 다른 파라미터에 `@Constraint` 어노테이션이 있어 메소드 검증이 적용되면, 대신 `HandlerMethodValidationException`이 발생합니다. 자세한 내용은 검증(Validation) 섹션을 참고하세요.

---

### **학습 목표 제시**

이 챕터를 학습하고 나면, 학생은 다음을 이해하고 수행할 수 있게 됩니다:

1. **스프링 MVC에서 멀티파트 요청 처리 방법 이해:** `multipart/form-data` 요청이 무엇이며, 스프링 MVC에서 이를 처리하기 위해 `MultipartResolver`가 어떻게 사용되는지, 그리고 업로드된 파일을 `MultipartFile` 객체로 어떻게 받는지 이해합니다.
2. **다양한 방식으로 멀티파트 데이터 접근:** `@RequestParam`을 사용하여 개별 파일 및 폼 필드에 접근하는 방법, 커맨드 객체에 파일 데이터를 바인딩하는 방법, 그리고 여러 파일을 한 번에 받는 방법을 이해합니다.
3. **`@RequestPart` 어노테이션의 활용 이해:** 단순 파일 데이터 외에 JSON 등 복합적인 데이터를 멀티파트 요청의 일부로 받아 처리할 때 `@RequestPart`를 어떻게 사용하며, `@RequestBody`와의 차이점 및 검증(Validation)과의 연동을 이해합니다.

---

### **핵심 개념 설명**

**Multipart 요청이란 무엇일까요?**

우리가 웹에서 게시글을 작성할 때 텍스트뿐만 아니라 이미지 파일, 동영상 파일 등을 함께 올리곤 합니다.

이렇게 **하나의 HTTP 요청 안에 텍스트 데이터와 파일 데이터 등 여러 종류의 데이터를 여러 부분(part)으로 나누어 함께 보내는 방식**을 **멀티파트(Multipart) 요청**이라고 합니다.

이때 HTTP 요청 헤더의 `Content-Type`은 주로 `multipart/form-data`로 설정됩니다.

이 타입은 웹 브라우저가 폼(form)을 통해 파일과 다른 데이터를 함께 서버로 전송할 때 사용되는 표준 방식입니다.

**스프링 MVC에서의 Multipart 처리 과정**

1. **`MultipartResolver` 활성화:**
  - 스프링 MVC가 멀티파트 요청을 처리하려면, 먼저 `MultipartResolver`라는 특별한 컴포넌트가 스프링 설정에 등록(활성화)되어 있어야 합니다. 이 친구가 바로 `multipart/form-data`로 들어오는 복잡한 요청을 스프링이 이해하기 쉬운 형태로 풀어주는 역할을 합니다.
  - 스프링 부트(Spring Boot)를 사용한다면, 기본적으로 `MultipartResolver`가 자동 설정되어 있어 별도의 설정 없이 바로 사용할 수 있는 경우가 많습니다. (내부적으로 아파치 커먼즈 파일업로드(Apache Commons FileUpload)나 서블릿 3.0+ 표준 멀티파트 지원을 사용합니다.)
2. **요청 파싱:**
  - `MultipartResolver`가 활성화되어 있으면, `multipart/form-data` 타입의 POST 요청이 들어왔을 때 디스패처 서블릿(DispatcherServlet)이 이를 감지하고 `MultipartResolver`를 호출합니다.
  - `MultipartResolver`는 요청 본문을 파싱하여, 각 파트(텍스트 데이터, 파일 데이터 등)를 분리합니다.
3. **데이터 접근:**
  - 파싱된 데이터는 컨트롤러에서 일반적인 요청 파라미터처럼, 또는 특별한 객체(`MultipartFile`)를 통해 접근할 수 있게 됩니다.

**`MultipartFile` 인터페이스**

스프링에서 업로드된 파일을 표현하는 핵심 인터페이스입니다. 개발자는 이 객체를 통해 업로드된 파일에 대한 다양한 정보(원본 파일 이름, 파일 크기, 파일 타입 등)를 얻고, 파일 내용을 읽거나 원하는 위치에 저장할 수 있습니다.

주요 메소드:

- `String getName()`: 파라미터 이름을 반환합니다 (HTML `<input type="file" name="X">` 에서의 "X").
- `String getOriginalFilename()`: 사용자가 업로드한 원본 파일 이름을 반환합니다.
- `String getContentType()`: 파일의 컨텐츠 타입(MIME 타입)을 반환합니다 (예: `image/jpeg`, `text/plain`).
- `boolean isEmpty()`: 업로드된 파일이 비어있는지 (선택은 했지만 실제 파일이 없거나 내용이 없는 경우) 확인합니다.
- `long getSize()`: 파일의 크기를 바이트 단위로 반환합니다.
- `byte[] getBytes()`: 파일 내용을 바이트 배열로 읽어옵니다. (메모리에 한 번에 로드하므로 큰 파일에는 주의)
- `InputStream getInputStream()`: 파일 내용을 읽을 수 있는 `InputStream`을 반환합니다. (스트리밍 방식 처리 가능)
- `void transferTo(File dest)`: 업로드된 파일을 지정된 `File` 객체가 가리키는 위치로 저장(복사)합니다. (가장 일반적인 저장 방법)
- `void transferTo(Path dest)`: (Java 7 NIO.2) 업로드된 파일을 지정된 `Path` 객체가 가리키는 위치로 저장합니다.

### **주요 용어 해설**

- **`multipart/form-data`:** HTTP `Content-Type`의 한 종류로, 여러 종류의 데이터를 각각의 파트(part)로 나누어 하나의 HTTP 요청으로 전송할 때 사용됩니다. 주로 파일 업로드를 포함하는 폼 전송에 사용됩니다.
- **`MultipartResolver`:** 스프링 MVC에서 `multipart/form-data` 요청을 파싱하여 컨트롤러가 파일 및 기타 데이터를 쉽게 처리할 수 있도록 변환해주는 역할을 하는 인터페이스 및 그 구현체.
- **`MultipartFile`:** 스프링에서 업로드된 파일을 나타내는 인터페이스. 파일 정보 접근 및 내용 처리를 위한 메소드 제공.
- **`jakarta.servlet.http.Part`:** 서블릿 3.0 스펙부터 표준으로 제공되는, 멀티파트 요청의 각 부분을 나타내는 인터페이스. 스프링 `MultipartFile` 대신 사용할 수도 있습니다.
- **커맨드 객체 (Command Object) / 폼 객체 (Form Object):** HTML 폼 필드의 값들을 담기 위해 사용되는 자바 객체(POJO). 스프링 MVC는 요청 파라미터를 이 객체의 필드에 자동으로 바인딩해줍니다.
- **`@RequestParam`:** 요청 파라미터(URL 쿼리 스트링 또는 폼 데이터)를 컨트롤러 메소드의 매개변수에 바인딩하는 어노테이션.
- **`@RequestPart`:** 멀티파트 요청의 특정 파트(part)를 컨트롤러 메소드의 매개변수에 바인딩하는 어노테이션. 파일뿐만 아니라 JSON, XML 등의 복합적인 데이터 파트를 `HttpMessageConverter`를 통해 객체로 변환하는 데 유용합니다.
- **`HttpMessageConverter`:** HTTP 요청/응답 본문을 자바 객체로 변환하거나 그 반대의 작업을 수행하는 스프링 인터페이스. (예: JSON을 객체로, 객체를 JSON으로)

### **코드 예제 및 분석**

**1. 기본적인 파일 업로드 (`@RequestParam` 사용)**

HTML 폼 예시:

```html
<form method="POST" action="/form" enctype="multipart/form-data">
    Name: <input type="text" name="name" /><br/>
    File: <input type="file" name="file" /><br/>
    <input type="submit" value="Upload" />
</form>
```

컨트롤러 코드:

```java
@Controller
public class FileUploadController {

	@PostMapping("/form") // 1. /form 경로로 POST 요청 처리
	public String handleFormUpload(@RequestParam("name") String name, // 2. "name" 폼 필드 값 바인딩
			@RequestParam("file") MultipartFile file) { // 3. "file" 파라미터로 업로드된 파일을 MultipartFile로 받음

		System.out.println("Name: " + name);
		System.out.println("Uploaded File Name: " + file.getOriginalFilename());
		System.out.println("Uploaded File Size: " + file.getSize());

		if (!file.isEmpty()) { // 4. 파일이 비어있지 않은지 확인
			try {
				byte[] bytes = file.getBytes(); // 5. 파일 내용을 바이트 배열로 가져옴
				// 어딘가에 바이트 데이터를 저장합니다 (예: 파일 시스템, 데이터베이스)
				// 예시: Files.write(Paths.get("/uploads/" + file.getOriginalFilename()), bytes);
				// 더 권장되는 방식: file.transferTo(new File("/uploads/" + file.getOriginalFilename()));
				System.out.println("File successfully uploaded and content read.");
				return "redirect:uploadSuccess"; // 6. 성공 시 리다이렉트
			} catch (IOException e) {
				e.printStackTrace();
				// 오류 처리
				return "redirect:uploadFailure";
			}
		} else {
			System.out.println("Uploaded file is empty.");
		}
		return "redirect:uploadFailure"; // 7. 실패 시 리다이렉트
	}
}

```

**코드 분석:**

1. `@PostMapping("/form")`: `/form` 경로로 오는 HTTP POST 요청을 이 메소드가 처리합니다. HTML 폼의 `action` 속성과 일치해야 합니다.
2. `@RequestParam("name") String name`: HTML 폼에서 `name="name"`으로 지정된 텍스트 입력 필드의 값을 `name` 문자열 변수에 바인딩합니다.
3. `@RequestParam("file") MultipartFile file`: HTML 폼에서 `name="file"`으로 지정된 파일 입력 필드를 통해 업로드된 파일을 `MultipartFile` 타입의 `file` 변수에 바인딩합니다. 스프링이 `MultipartResolver`를 통해 이 변환을 자동으로 처리해줍니다.
4. `if (!file.isEmpty())`: 실제로 파일이 업로드되었는지 확인합니다. 사용자가 파일 선택을 하지 않거나, 빈 파일을 올릴 수도 있기 때문입니다.
5. `byte[] bytes = file.getBytes();`: 파일의 내용을 바이트 배열로 읽어옵니다. 이 바이트 배열을 파일 시스템에 저장하거나, 데이터베이스에 저장하는 등의 처리를 할 수 있습니다.
  - **실제 저장 시 권장:** `file.transferTo(new File("저장경로/파일명"));` 또는 `file.transferTo(Paths.get("저장경로/파일명"));`을 사용하는 것이 더 효율적이고 일반적입니다. `getBytes()`는 파일 전체를 메모리에 로드하므로 매우 큰 파일의 경우 `OutOfMemoryError`가 발생할 수 있습니다.
6. `return "redirect:uploadSuccess";`: 업로드 성공 시 `/uploadSuccess` URL로 리다이렉트합니다.
7. `return "redirect:uploadFailure";`: 파일이 비어있거나 업로드 중 오류 발생 시 `/uploadFailure` URL로 리다이렉트합니다.

**여러 파일 업로드:**

- **같은 이름으로 여러 파일:** HTML에서 `<input type="file" name="files" multiple />` 와 같이 하고, 컨트롤러에서 `@RequestParam("files") List<MultipartFile> files` 로 받으면 됩니다.
- **다른 이름으로 여러 파일:** HTML에서 `<input type="file" name="file1" />`, `<input type="file" name="file2" />` 와 같이 하고, 컨트롤러에서 `@RequestParam("file1") MultipartFile file1, @RequestParam("file2") MultipartFile file2` 로 각각 받거나, `@RequestParam Map<String, MultipartFile> fileMap` 으로 받으면 맵에 `"file1": (MultipartFile 객체)`, `"file2": (MultipartFile 객체)` 형태로 담겨 옵니다. (맵으로 받을 때는 `@RequestParam`에 이름을 명시하지 않아야 합니다.)

**2. 커맨드 객체(폼 객체)로 데이터 바인딩**

폼 객체 정의:

```java
class MyForm {
	private String name;
	private MultipartFile file; // 폼 필드 이름과 동일한 이름의 필드

	// Getter와 Setter (스프링이 값을 바인딩하려면 필요)
	public String getName() { return name; }
	public void setName(String name) { this.name = name; }
	public MultipartFile getFile() { return file; }
	public void setFile(MultipartFile file) { this.file = file; }
}
```

컨트롤러 코드:

```java
@Controller
public class FileUploadController {

	@PostMapping("/formWithObject")
	public String handleFormUpload(MyForm form, BindingResult errors) { // 1. MyForm 객체를 매개변수로 받음
		// errors 객체는 데이터 바인딩 또는 검증 오류 정보를 담음 (여기서는 간단히 생략)

		if (form.getFile() != null && !form.getFile().isEmpty()) { // 2. 폼 객체를 통해 파일 접근
			try {
				// file.transferTo() 등을 사용하여 파일 저장
				System.out.println("Name from form object: " + form.getName());
				System.out.println("File from form object: " + form.getFile().getOriginalFilename());
				return "redirect:uploadSuccess";
			} catch (Exception e) {
				// 오류 처리
				return "redirect:uploadFailure";
			}
		}
		return "redirect:uploadFailure";
	}
}
```

**코드 분석:**

1. `MyForm form`: 컨트롤러 메소드의 매개변수로 `MyForm` 타입의 객체를 선언합니다. 스프링 MVC는 HTTP 요청 파라미터들(여기서는 `name`과 `file`)을 `MyForm` 객체의 해당 필드(set메소드를 통해)에 자동으로 바인딩해줍니다. `file`이라는 이름의 요청 파라미터(업로드된 파일 데이터)가 `MyForm` 객체의 `MultipartFile` 타입 `file` 필드에 바인딩됩니다.
2. `if (form.getFile() != null && !form.getFile().isEmpty())`: 폼 객체의 `getFile()` 메소드를 통해 업로드된 `MultipartFile` 객체에 접근합니다. 이후 처리는 `@RequestParam`을 사용했을 때와 유사합니다.
   이 방식은 폼 데이터가 많을 때 코드를 더 깔끔하게 관리할 수 있는 장점이 있습니다.

**3. `@RequestPart` 사용 (JSON 등 복합 데이터와 파일 함께 처리)**

RESTful API에서 클라이언트가 JSON 데이터와 파일을 함께 보내는 경우를 생각해봅시다.
예를 들어, "상품 메타데이터(JSON)"와 "상품 이미지 파일"을 한 번의 요청으로 보내는 것입니다.

요청 예시 (원문 참조): `Content-Type: multipart/mixed` (또는 `multipart/form-data`)

```
--boundary_string
Content-Disposition: form-data; name="meta-data"
Content-Type: application/json

{ "productName": "Awesome Gadget", "price": 99.99 }
--boundary_string
Content-Disposition: form-data; name="image-file"; filename="gadget.jpg"
Content-Type: image/jpeg

... (이미지 파일의 바이너리 데이터) ...
--boundary_string--
```

`MetaData` 자바 클래스:

```java
class MetaData {
    private String productName;
    private double price;
    // Getters and Setters
}
```

컨트롤러 코드:

```java
@PostMapping("/productWithJson")
public String handleProductUpload(
        @RequestPart("meta-data") MetaData metadata, // 1. "meta-data" 파트를 MetaData 객체로 자동 변환
        @RequestPart("image-file") MultipartFile imageFile, // 2. "image-file" 파트를 MultipartFile로 받음
        BindingResult errors) {

    // metadata 객체와 imageFile 객체 사용
    System.out.println("Product Name: " + metadata.getProductName());
    System.out.println("Price: " + metadata.getPrice());
    System.out.println("Image File: " + imageFile.getOriginalFilename());

    if (errors.hasErrors()) {
        // 검증 오류 처리
        return "errorPage";
    }

    // 파일 저장 및 비즈니스 로직 처리
    return "redirect:productUploadSuccess";
}
```

**코드 분석:**

1. `@RequestPart("meta-data") MetaData metadata`:
  - `@RequestPart` 어노테이션은 멀티파트 요청의 특정 "파트"를 지정합니다. 여기서는 `name="meta-data"`인 파트를 가리킵니다.
  - 이 파트의 `Content-Type`이 `application/json`이므로, 스프링은 등록된 `HttpMessageConverter` (예: `Jackson2HttpMessageConverter`)를 사용하여 JSON 문자열을 `MetaData` 자바 객체로 자동으로 역직렬화(변환)합니다. 이는 `@RequestBody`가 요청 본문 전체를 객체로 변환하는 것과 유사한 방식이지만, `@RequestPart`는 멀티파트 요청의 *일부*에 대해 이 작업을 수행합니다.
2. `@RequestPart("image-file") MultipartFile imageFile`:
  - `name="image-file"`인 파트를 `MultipartFile` 객체로 받습니다. 파일 파트의 경우 `@RequestParam`과 유사하게 동작하지만, `@RequestPart`는 복합적인 파트 처리에 더 특화되어 있습니다.

**`@RequestPart`와 검증 (Validation):**
원문에서 언급된 것처럼, `@RequestPart`로 받은 객체에 대해 `@Valid` 또는 `@Validated` 어노테이션을 사용하여 빈 검증(Bean Validation)을 적용할 수 있습니다.

```java
@PostMapping("/validatedProduct")
public String handleValidatedProductUpload(
        @Valid @RequestPart("meta-data") MetaData metadata, Errors errors, // @Valid 추가
        @RequestPart("image-file") MultipartFile imageFile) {

    if (errors.hasErrors()) {
        // metadata 객체에 대한 검증 오류가 errors 객체에 담김
        // 예: metadata의 productName이 @NotEmpty인데 비어있는 경우
        System.out.println("Validation errors: " + errors.getAllErrors());
        return "redirect:uploadFailureWithValidationErrors";
    }
    // 검증 통과 시 로직
    return "redirect:productUploadSuccess";
}
```

### **"왜?" 라는 질문에 대한 답변**

**왜 `MultipartResolver`가 필요하고, `MultipartFile`이라는 인터페이스를 사용할까요?**

1. **표준 HTTP 요청 처리의 복잡성:** `multipart/form-data` 요청은 일반적인 `application/x-www-form-urlencoded` 요청보다 구조가 복잡합니다. 각 파트는 헤더(Content-Disposition, Content-Type 등)와 본문을 가지며, 이들을 수동으로 파싱하는 것은 번거롭고 오류가 발생하기 쉽습니다. `MultipartResolver`는 이 복잡한 파싱 작업을 추상화하여 개발자가 쉽게 멀티파트 데이터를 다룰 수 있도록 해줍니다.
2. **파일 처리를 위한 표준화된 API 제공:** 업로드된 파일은 단순한 바이트 스트림이 아니라 파일명, 컨텐츠 타입, 크기 등의 메타데이터를 가집니다. `MultipartFile` 인터페이스는 이러한 파일 관련 정보와 기능(내용 읽기, 저장 등)에 접근할 수 있는 일관되고 편리한 방법을 제공합니다. 개발자는 서블릿 API의 저수준 `Part` 객체를 직접 다루는 것보다 `MultipartFile`을 사용하는 것이 더 쉽고 생산적입니다.
3. **다양한 구현체 지원 및 유연성:** `MultipartResolver`는 인터페이스이므로, 아파치 커먼즈 파일업로드, 서블릿 표준 멀티파트 지원 등 다양한 실제 구현체를 사용할 수 있는 유연성을 제공합니다. 개발자는 환경에 맞는 구현체를 선택하거나 교체할 수 있습니다.

**`@RequestParam` 대신 `@RequestPart`는 언제, 왜 사용할까요?**

- **단순 파일/텍스트 vs. 복합 객체:**
  - `@RequestParam`: 주로 단순 텍스트 폼 필드나 단일 파일을 받을 때 편리합니다. 파일 자체를 `MultipartFile`로 받습니다.
  - `@RequestPart`: 멀티파트 요청의 각 파트가 단순 파일이 아니라, JSON이나 XML과 같이 구조화된 데이터일 때 유용합니다. 해당 파트의 내용을 `HttpMessageConverter`를 통해 특정 자바 객체로 변환(역직렬화)하고자 할 때 사용합니다. `@RequestParam`은 이러한 자동 객체 변환 기능을 제공하지 않습니다 (파일을 제외하고는 주로 문자열로 받음).
- **Content-Type 기반 변환:** `@RequestPart`는 각 파트의 `Content-Type` 헤더를 보고 적절한 `HttpMessageConverter`를 선택하여 변환을 시도합니다. 이는 `@RequestBody`가 요청 전체의 `Content-Type`을 보고 변환하는 방식과 유사합니다.

결국, 스프링 MVC의 멀티파트 지원 기능들은 개발자가 복잡한 멀티파트 요청을 쉽고, 안전하며, 유연하게 처리할 수 있도록 돕기 위해 설계되었습니다.

### **주의사항 및 Best Practice**

1. **`MultipartResolver` 설정 확인:** (스프링 부트가 아닌 환경에서는) `MultipartResolver` 빈이 제대로 설정되어 있는지 확인해야 합니다. 일반적으로 `CommonsMultipartResolver` (Apache Commons FileUpload 사용) 또는 `StandardServletMultipartResolver` (Servlet 3.0+ 표준 사용) 중 하나를 사용합니다.
2. **파일 크기 제한 설정:** 업로드될 파일의 최대 크기를 제한하는 것이 중요합니다. 너무 큰 파일이 업로드되면 서버 메모리나 디스크 공간을 고갈시켜 서비스 거부(DoS) 공격에 취약해질 수 있습니다. 스프링 또는 서블릿 컨테이너 수준에서 파일 크기 제한을 설정할 수 있습니다. (예: `spring.servlet.multipart.max-file-size`, `spring.servlet.multipart.max-request-size` in Spring Boot)
3. **`MultipartFile.transferTo()` 사용 권장:** 파일 저장 시 `getBytes()`로 전체 내용을 메모리에 읽는 것보다 `transferTo()` 메소드를 사용하여 직접 파일 시스템으로 스트리밍하는 것이 메모리 효율성 면에서 좋습니다.
4. **임시 파일 관리:** 파일 업로드 시 `MultipartResolver`는 임시 파일을 생성할 수 있습니다. 애플리케이션 종료 시 또는 특정 조건에서 이 임시 파일들이 제대로 정리되는지 확인하거나, `transferTo()`를 사용한 후에는 원본 임시 파일이 어떻게 처리되는지 이해하고 있어야 합니다. (대부분의 `MultipartResolver` 구현체는 처리 후 임시 파일을 정리합니다.)
5. **보안 고려:**
  - **파일명 검증:** 사용자가 업로드한 파일명(`getOriginalFilename()`)을 그대로 사용하여 서버에 저장할 경우, 경로 조작(path traversal) 공격 (`../../../../../etc/passwd` 같은 파일명)에 취약할 수 있습니다. 파일명을 안전하게 필터링하거나, 서버에서 새로운 고유 파일명을 생성하여 사용하는 것이 좋습니다.
  - **파일 확장자 및 내용 검증 (MIME 타입):** 업로드된 파일의 확장자뿐만 아니라 실제 내용(MIME 타입, 매직 넘버 등)을 검증하여 악성 파일(예: 실행 파일) 업로드를 차단해야 합니다.
6. **오류 처리:** 파일 업로드 과정(네트워크 오류, 디스크 공간 부족, 권한 문제 등)에서 다양한 오류가 발생할 수 있으므로, `try-catch` 블록을 사용하여 적절히 예외 처리를 해야 합니다.
7. **`@RequestPart`와 `@RequestParam`의 구분 사용:**
  - HTML 폼에서 단순 파일과 텍스트 필드를 받는다면 `@RequestParam`이 더 간결하고 직관적일 수 있습니다.
  - 하나의 멀티파트 요청에 JSON/XML 데이터와 파일이 함께 오는 REST API의 경우 `@RequestPart`가 강력한 기능을 제공합니다.

### **이전 학습 내용과의 연관성**

- **`@RequestParam`:** 이전에 일반적인 요청 파라미터를 받을 때 사용했던 어노테이션입니다. 멀티파트 요청에서도 텍스트 폼 필드나 파일을 받는 데 동일하게 사용될 수 있다는 점을 연결할 수 있습니다.
- **데이터 바인딩:** 커맨드 객체(폼 객체)를 사용하여 멀티파트 데이터를 받는 것은 스프링 MVC의 강력한 데이터 바인딩 기능의 확장입니다.
- **`HttpMessageConverter`:** `@RequestPart`가 JSON/XML 등의 파트를 객체로 변환할 때 `HttpMessageConverter`를 사용한다는 점은, 이전에 `@RequestBody`나 `@ResponseBody`를 학습할 때 접했던 `HttpMessageConverter`의 역할을 다시 한번 상기시켜 줍니다.
- **검증 (Validation):** `@RequestPart`로 받은 객체에 `@Valid`를 적용하여 검증하는 것은 스프링 MVC의 통합된 검증 메커니즘을 활용하는 예입니다.

---
