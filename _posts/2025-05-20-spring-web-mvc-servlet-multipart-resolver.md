---
title: Spring Web MVC - Dispatcher Servlet (Multipart Resolver)
description: 
author: laze
date: 2025-05-20 00:00:04 +0900
categories: [Dev, SpringWebMVC]
tags: [SpringWebMVC]
---
**멀티파트 리졸버 (Multipart Resolver)**

Reactive 스택에서의 동등한 내용을 확인하세요.

`org.springframework.web.multipart` 패키지의 `MultipartResolver`는 파일 업로드를 포함한 멀티파트 요청을 파싱하기 위한 전략입니다.

서블릿 멀티파트 요청 파싱을 위한 컨테이너 기반의 `StandardServletMultipartResolver` 구현체가 있습니다.

Apache Commons FileUpload에 기반한 오래된 `CommonsMultipartResolver`는 Servlet 5.0+ 기준을 따르는 Spring Framework 6.0부터 더 이상 사용할 수 없다는 점에 유의하세요.

멀티파트 처리를 활성화하려면, `DispatcherServlet` Spring 설정에 `multipartResolver`라는 이름으로 `MultipartResolver` 빈을 선언해야 합니다.

`DispatcherServlet`은 이를 감지하고 들어오는 요청에 적용합니다.

`multipart/form-data` 콘텐츠 타입의 POST 요청이 수신되면, 리졸버는 콘텐츠를 파싱하고 현재 `HttpServletRequest`를 `MultipartHttpServletRequest`로 래핑(wrapping)하여,

해석된 파일에 대한 접근을 제공하고 파트(parts)들을 요청 파라미터로 노출합니다.

**서블릿 멀티파트 파싱 (Servlet Multipart Parsing)**
서블릿 멀티파트 파싱은 서블릿 컨테이너 설정을 통해 활성화되어야 합니다. 이를 위해:

- Java에서는 서블릿 등록 시 `MultipartConfigElement`를 설정합니다.
- `web.xml`에서는 서블릿 선언에 `<multipart-config>` 섹션을 추가합니다.

다음 예제는 서블릿 등록 시 `MultipartConfigElement`를 설정하는 방법을 보여줍니다:

```java
public class AppInitializer extends AbstractAnnotationConfigDispatcherServletInitializer {

	// ...

	@Override
	protected void customizeRegistration(ServletRegistration.Dynamic registration) {

		// 선택적으로 maxFileSize, maxRequestSize, fileSizeThreshold도 설정
		registration.setMultipartConfig(new MultipartConfigElement("/tmp"));
	}

}
```

서블릿 멀티파트 설정이 완료되면, `multipartResolver`라는 이름으로 `StandardServletMultipartResolver` 타입의 빈을 추가할 수 있습니다.

이 리졸버 변형은 서블릿 컨테이너의 멀티파트 파서를 그대로 사용하며, 잠재적으로 애플리케이션을 컨테이너 구현 차이에 노출시킬 수 있습니다.

기본적으로, 모든 HTTP 메소드의 모든 `multipart/` 콘텐츠 타입을 파싱하려고 시도하지만, 이는 모든 서블릿 컨테이너에서 지원되지 않을 수 있습니다.

---

## Spring MVC의 파일 업로드 처리 마법사: MultipartResolver 이해하기

HTML 폼을 통해 파일을 업로드하면, HTTP 요청은 **`multipart/form-data`** 라는 특별한 `Content-Type`으로 전송됩니다.

이 요청은 일반적인 텍스트 데이터와 함께 파일 데이터를 포함하고 있으며, 이를 파싱(parsing, 분석하고 분리하는 작업)하는 것은 다소 복잡합니다.

Spring MVC는 **`MultipartResolver`** 인터페이스를 통해 이 멀티파트 요청 파싱 작업을 추상화하고, 개발자가 업로드된 파일에 쉽게 접근할 수 있도록 합니다.

**핵심 변경 사항 (Spring Framework 6.0 부터):**

- 과거에 많이 사용되던 Apache Commons FileUpload 기반의 `CommonsMultipartResolver`는 **더 이상 사용할 수 없습니다.**
- 현재는 **서블릿 3.0 표준**을 따르는 컨테이너 기반의 **`StandardServletMultipartResolver`*가 주된 구현체입니다.

### 1. `MultipartResolver`의 역할

- `DispatcherServlet`은 요청이 들어올 때 `multipartResolver`라는 이름의 빈이 등록되어 있는지 확인합니다.
- 만약 등록되어 있고, 들어온 요청의 `Content-Type`이 `multipart/form-data`라면, `MultipartResolver`가 활성화됩니다.
- `MultipartResolver`는 요청 본문을 파싱하여:
  - 텍스트 파라미터는 일반적인 요청 파라미터처럼 접근할 수 있도록 합니다.
  - 업로드된 파일 데이터는 별도로 추출합니다.
- 원래의 `HttpServletRequest` 객체를 **`MultipartHttpServletRequest`** 인터페이스를 구현한 객체로 **래핑(wrapping)**합니다.
  - `MultipartHttpServletRequest`는 `HttpServletRequest`의 모든 기능을 포함하면서, 업로드된 파일에 접근하기 위한 추가 메소드(예: `getFile(String name)`, `getFileMap()`)를 제공합니다.

### 2. 멀티파트 처리 활성화 단계

`StandardServletMultipartResolver`를 사용하기 위해서는 두 가지 주요 설정이 필요합니다.

### 단계 1: 서블릿 컨테이너 레벨에서 멀티파트 파싱 활성화

Spring MVC의 `MultipartResolver`가 동작하기 전에, 서블릿 컨테이너 자체가 멀티파트 요청을 처리할 수 있도록 설정되어야 합니다.

이는 서블릿 3.0 표준에 따른 설정입니다.

- **Java Config 방식 (`AbstractAnnotationConfigDispatcherServletInitializer` 사용 시):**
  - `customizeRegistration(ServletRegistration.Dynamic registration)` 메소드를 오버라이드하여 `MultipartConfigElement`를 설정합니다.

    ```java
    // AppInitializer.java
    import javax.servlet.MultipartConfigElement;
    import javax.servlet.ServletRegistration;
    import org.springframework.web.servlet.support.AbstractAnnotationConfigDispatcherServletInitializer;
    
    public class AppInitializer extends AbstractAnnotationConfigDispatcherServletInitializer {
    
        // ... (getRootConfigClasses, getServletConfigClasses, getServletMappings 생략)
    
        @Override
        protected void customizeRegistration(ServletRegistration.Dynamic registration) {
            // 업로드된 파일이 임시로 저장될 위치 (필수)
            // 실제 운영 환경에서는 적절한 경로로 변경해야 합니다.
            String tempUploadPath = System.getProperty("java.io.tmpdir"); // 시스템 임시 디렉토리 사용 예시
            if (tempUploadPath == null) {
                tempUploadPath = "/tmp"; // 기본값
            }
    
            // MultipartConfigElement 설정
            // 첫 번째 인자: 임시 파일 저장 위치 (location)
            // 두 번째 인자: 파일 하나의 최대 크기 (maxFileSize, 바이트 단위, -1L은 제한 없음)
            // 세 번째 인자: 전체 요청의 최대 크기 (maxRequestSize, 바이트 단위, -1L은 제한 없음)
            // 네 번째 인자: 파일 크기 임계값 (fileSizeThreshold, 이 크기를 넘으면 디스크에 임시 저장, 바이트 단위)
            MultipartConfigElement multipartConfigElement =
                    new MultipartConfigElement(tempUploadPath, 20971520, 41943040, 0); // 예: 파일당 20MB, 전체 40MB
            // 간단하게는 new MultipartConfigElement("/tmp") 와 같이 임시 저장 위치만 지정할 수도 있습니다.
            // 기본값: location="", maxFileSize=-1L, maxRequestSize=-1L, fileSizeThreshold=0
    
            registration.setMultipartConfig(multipartConfigElement);
        }
    
        // 예시를 위한 간단한 구현
        @Override
        protected Class<?>[] getRootConfigClasses() { return null; }
        @Override
        protected Class<?>[] getServletConfigClasses() { return new Class<?>[]{ WebConfig.class }; } // WebConfig는 Spring MVC 설정 클래스
        @Override
        protected String[] getServletMappings() { return new String[]{ "/" }; }
    }
    ```

  - **`MultipartConfigElement`의 주요 파라미터:**
    - `location`: 업로드된 파일이 임시로 저장될 디렉토리 경로. (필수 아님, 비워두면 컨테이너 기본 위치)
    - `maxFileSize`: 업로드 가능한 파일 하나의 최대 크기 (바이트 단위). 기본값 -1L (제한 없음).
    - `maxRequestSize`: 멀티파트 요청 전체의 최대 크기 (바이트 단위). 기본값 -1L (제한 없음).
    - `fileSizeThreshold`: 파일 데이터가 이 크기를 초과하면 디스크에 임시 저장되고, 그 이하면 메모리에 유지됩니다. 기본값 0 (항상 디스크에 저장).
- **`web.xml` 방식:**
  - `<servlet>` 선언 내에 `<multipart-config>` 섹션을 추가합니다.

    ```xml
    <!-- web.xml -->
    <servlet>
        <servlet-name>dispatcher</servlet-name>
        <servlet-class>org.springframework.web.servlet.DispatcherServlet</servlet-class>
        <!-- ... 기타 init-param ... -->
        <multipart-config>
            <location>/tmp</location>
            <max-file-size>20971520</max-file-size> <!-- 20MB -->
            <max-request-size>41943040</max-request-size> <!-- 40MB -->
            <file-size-threshold>0</file-size-threshold>
        </multipart-config>
    </servlet>
    ```


### 단계 2: Spring MVC 설정에 `MultipartResolver` 빈 등록

서블릿 컨테이너 설정이 완료되면, Spring MVC 설정 파일(Java Config 또는 XML)에 `StandardServletMultipartResolver` 빈을 **`multipartResolver`라는 이름으로** 등록합니다.

- **Java Config 방식:**

    ```java
    // WebConfig.java (Spring MVC 설정 클래스)
    import org.springframework.context.annotation.Bean;
    import org.springframework.context.annotation.ComponentScan;
    import org.springframework.context.annotation.Configuration;
    import org.springframework.web.multipart.MultipartResolver;
    import org.springframework.web.multipart.support.StandardServletMultipartResolver;
    import org.springframework.web.servlet.config.annotation.EnableWebMvc;
    
    @Configuration
    @EnableWebMvc
    @ComponentScan(basePackages = "com.example.controller")
    public class WebConfig {
    
        @Bean // 빈의 이름이 "multipartResolver" 여야 함
        public MultipartResolver multipartResolver() {
            StandardServletMultipartResolver multipartResolver = new StandardServletMultipartResolver();
            // StandardServletMultipartResolver는 추가적인 프로퍼티 설정이 거의 필요 없습니다.
            // 필요하다면 javadoc을 참조하여 resolveLazily 등의 옵션을 설정할 수 있습니다.
            // multipartResolver.setResolveLazily(true); // 요청 파라미터 접근 시 실제 파싱 (기본은 false)
            return multipartResolver;
        }
    
        // ... (다른 ViewResolver, Interceptor 등의 설정)
    }
    ```

- **XML Config 방식:**

    ```xml
    <!-- dispatcher-servlet.xml -->
    <bean id="multipartResolver" class="org.springframework.web.multipart.support.StandardServletMultipartResolver">
        <!-- <property name="resolveLazily" value="true" /> -->
    </bean>
    ```


**중요:** 빈의 이름은 반드시 `multipartResolver`여야 `DispatcherServlet`이 자동으로 감지하여 사용합니다.

### 3. 컨트롤러에서 업로드된 파일 사용하기

위 설정이 모두 완료되면, 컨트롤러에서 `MultipartHttpServletRequest` 또는 `@RequestParam("file") MultipartFile` 어노테이션을 사용하여 업로드된 파일에 접근할 수 있습니다.

**예시 컨트롤러:**

```java
// FileUploadController.java
import org.springframework.stereotype.Controller;
import org.springframework.ui.Model;
import org.springframework.web.bind.annotation.GetMapping;
import org.springframework.web.bind.annotation.PostMapping;
import org.springframework.web.bind.annotation.RequestParam;
import org.springframework.web.multipart.MultipartFile;
import org.springframework.web.multipart.MultipartHttpServletRequest; // 선택적 사용

import java.io.File;
import java.io.IOException;
import java.util.Iterator;

@Controller
public class FileUploadController {

    @GetMapping("/uploadForm")
    public String uploadForm() {
        return "uploadForm"; // uploadForm.jsp 또는 Thymeleaf 템플릿
    }

    // 방법 1: @RequestParam MultipartFile 사용 (권장, 더 간편)
    @PostMapping("/upload")
    public String handleFileUpload(@RequestParam("file") MultipartFile file,
                                   @RequestParam("description") String description,
                                   Model model) {
        if (!file.isEmpty()) {
            try {
                // 업로드된 파일 이름 가져오기
                String originalFileName = file.getOriginalFilename();
                model.addAttribute("fileName", originalFileName);
                model.addAttribute("fileSize", file.getSize());
                model.addAttribute("contentType", file.getContentType());
                model.addAttribute("description", description);

                // 파일 저장 (실제 저장 경로 및 로직은 애플리케이션에 맞게 구현)
                // 예: File newFile = new File("/path/to/uploads/" + originalFileName);
                // file.transferTo(newFile);

                logger.info("File uploaded: {}, Description: {}", originalFileName, description);
                model.addAttribute("message", "File uploaded successfully: " + originalFileName);

            } catch (Exception e) { // IOException 또는 IllegalStateException 등
                logger.error("File upload failed", e);
                model.addAttribute("message", "File upload failed: " + e.getMessage());
            }
        } else {
            model.addAttribute("message", "No file selected for upload.");
        }
        return "uploadResult"; // uploadResult.jsp 또는 Thymeleaf 템플릿
    }

    // 방법 2: MultipartHttpServletRequest 직접 사용 (여러 파일 처리 등에 유용)
    @PostMapping("/uploadMultiple")
    public String handleMultipleFileUpload(MultipartHttpServletRequest request, Model model) {
        String description = request.getParameter("description"); // 일반 텍스트 파라미터
        model.addAttribute("description", description);

        Iterator<String> fileNames = request.getFileNames(); // 업로드된 모든 파일의 input name 가져오기
        int count = 0;
        while (fileNames.hasNext()) {
            String inputName = fileNames.next(); // <input type="file" name="uploadFile1"> 에서 "uploadFile1"
            MultipartFile file = request.getFile(inputName);

            if (file != null && !file.isEmpty()) {
                try {
                    // 파일 처리 로직 (위와 유사)
                    String originalFileName = file.getOriginalFilename();
                    // file.transferTo(new File("/path/to/uploads/" + originalFileName));
                    logger.info("File [{}] uploaded: {}", count, originalFileName);
                    model.addAttribute("file" + count, originalFileName);
                    count++;
                } catch (Exception e) {
                    logger.error("File upload failed for input name: " + inputName, e);
                }
            }
        }
        model.addAttribute("message", count + " files uploaded successfully.");
        return "uploadResult";
    }
}

```

**`MultipartFile` 인터페이스의 주요 메소드:**

- `String getName()`: 폼 필드의 이름 ( `<input type="file" name="이름">` )
- `String getOriginalFilename()`: 클라이언트 시스템에서의 원래 파일 이름
- `String getContentType()`: 파일의 컨텐츠 타입 (MIME 타입)
- `boolean isEmpty()`: 업로드된 파일이 없는지 여부
- `long getSize()`: 파일 크기 (바이트)
- `byte[] getBytes()`: 파일 내용을 바이트 배열로 반환
- `InputStream getInputStream()`: 파일 내용을 읽을 수 있는 `InputStream` 반환
- `void transferTo(File dest)`: 업로드된 파일을 지정된 `File` 객체(목적지 파일)로 저장

### 4. `StandardServletMultipartResolver`의 특징

- **서블릿 컨테이너 의존:** 서블릿 컨테이너가 제공하는 멀티파트 파싱 기능을 그대로 사용합니다. 이는 컨테이너 구현체(Tomcat, Jetty, Undertow 등)에 따라 미세한 동작 차이가 있을 수 있음을 의미합니다.
- **단순함:** Apache Commons FileUpload 같은 외부 라이브러리에 대한 의존성이 없습니다.
- **HTTP 메소드 및 Content-Type:** 기본적으로 모든 HTTP 메소드(POST, PUT 등)와 `multipart/*` 형태의 모든 `Content-Type`을 파싱하려고 시도합니다. 하지만 모든 서블릿 컨테이너가 모든 조합을 지원하지 않을 수 있으므로, 주의가 필요합니다. (일반적으로 파일 업로드는 POST 메소드를 사용합니다.)

Spring MVC의 `MultipartResolver`를 사용하면, 복잡한 멀티파트 요청 파싱 로직을 직접 구현할 필요 없이, 컨트롤러에서 업로드된 파일과 다른 폼 데이터에 쉽게 접근하여 비즈니스 로직을 처리할 수 있습니다.

---
