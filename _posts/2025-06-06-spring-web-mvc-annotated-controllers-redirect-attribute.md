---
title: Spring Web MVC - Annotated Controllers (@RedirectAttribute)
description: 
author: laze
date: 2025-06-06 00:00:02 +0900
categories: [Dev, SpringWebMVC]
tags: [SpringWebMVC]
---
### Redirect Attributes (리다이렉트 속성)

기본적으로, 모든 모델 속성(model attributes)은 리다이렉트 URL에서 URI 템플릿 변수(URI template variables)로 노출되는 것으로 간주됩니다.

나머지 속성들 중, 기본 타입(primitive types)이거나 기본 타입의 컬렉션 또는 배열인 속성들은 자동으로 쿼리 파라미터(query parameters)로 추가됩니다.

모델 인스턴스가 특별히 리다이렉트를 위해 준비된 경우라면, 기본 타입 속성을 쿼리 파라미터로 추가하는 것이 원하는 결과일 수 있습니다.

그러나 어노테이션 기반 컨트롤러에서는, 모델이 렌더링 목적(예: 드롭다운 필드 값)으로 추가된 부가적인 속성들을 포함할 수 있습니다.

이러한 속성들이 URL에 나타날 가능성을 피하기 위해, `@RequestMapping` 메소드는 `RedirectAttributes` 타입의 인자를 선언하고 이를 사용하여 `RedirectView`에 제공할 정확한 속성들을 지정할 수 있습니다.

만약 메소드가 리다이렉트를 수행하면, `RedirectAttributes`의 내용이 사용됩니다.

그렇지 않으면(리다이렉트하지 않으면), 모델의 내용이 사용됩니다.

`RequestMappingHandlerAdapter`는 `ignoreDefaultModelOnRedirect`라는 플래그를 제공합니다.

이 플래그를 사용하면 컨트롤러 메소드가 리다이렉트할 경우 기본 `Model`의 내용을 절대 사용하지 않도록 지정할 수 있습니다.

대신, 컨트롤러 메소드는 `RedirectAttributes` 타입의 속성을 선언해야 하며, 만약 그렇지 않으면 어떤 속성도 `RedirectView`로 전달되지 않아야 합니다.

MVC 네임스페이스와 MVC Java 설정 모두 하위 호환성을 유지하기 위해 이 플래그를 `false`로 유지합니다.

그러나 새로운 애플리케이션의 경우, 이 값을 `true`로 설정하는 것을 권장합니다.

현재 요청의 URI 템플릿 변수들은 리다이렉트 URL을 확장할 때 자동으로 사용 가능하게 되며, `Model`이나 `RedirectAttributes`를 통해 명시적으로 추가할 필요가 없습니다.

다음 예제는 리다이렉트를 정의하는 방법을 보여줍니다:

**Java**

```java
@PostMapping("/files/{path}")
public String upload(...) {
	// ...
	return "redirect:files/{path}";
}
```

리다이렉트 대상에게 데이터를 전달하는 또 다른 방법은 플래시 속성(flash attributes)을 사용하는 것입니다.

다른 리다이렉트 속성들과 달리, 플래시 속성은 HTTP 세션에 저장됩니다 (따라서 URL에 나타나지 않습니다).

---

### **학습 목표 제시**

이 챕터를 학습하고 나면, 학생은 다음을 이해하고 수행할 수 있게 됩니다:

1. **리다이렉트 시 데이터 전달의 필요성 및 기본 동작 이해:** 컨트롤러에서 다른 URL로 리다이렉트할 때 왜 데이터 전달이 필요하며, 스프링 MVC에서 모델 속성이 기본적으로 어떻게 처리되는지 (URI 템플릿 변수, 쿼리 파라미터) 이해합니다.
2. **`RedirectAttributes`의 역할과 사용법 숙지:** URL에 원치 않는 데이터가 노출되는 것을 방지하고, 리다이렉트 대상에게 명시적으로 데이터를 전달하기 위해 `RedirectAttributes`를 어떻게 사용하는지 이해하고 적용할 수 있습니다.
3. **`ignoreDefaultModelOnRedirect` 플래그의 의미와 권장 설정 이해:** 해당 플래그가 리다이렉트 시 모델 데이터 처리에 어떤 영향을 미치는지, 그리고 새로운 애플리케이션에서 권장되는 설정이 무엇인지 이해합니다. (플래시 속성은 다음 챕터에서 더 자세히 다룰 예정이므로, 여기서는 간단히 언급만 하고 넘어갑니다.)

---

### **핵심 개념 설명**

**리다이렉트(Redirect)란 무엇이고, 왜 데이터 전달이 필요할까요?**

**리다이렉트**는 웹 서버가 클라이언트(웹 브라우저)에게 "지금 요청한 페이지 말고, 저쪽 다른 페이지로 다시 가보세요!"라고 알려주는 것입니다. 그러면 브라우저는 서버가 알려준 새로운 URL로 자동으로 다시 요청을 보냅니다.

**예시:**

- **게시글 작성 후:** 사용자가 게시글을 성공적으로 작성하면, `/write` 페이지에서 `/articles/123` (방금 작성한 글 보기 페이지)와 같이 상세 페이지로 이동시키고 싶을 수 있습니다.
- **로그인 성공 후:** 사용자가 `/login` 페이지에서 로그인을 성공하면, 이전에 보려던 페이지나 사용자의 메인 페이지 `/home` 등으로 이동시킵니다.
- **권한 없는 페이지 접근 시:** 사용자가 권한 없는 페이지에 접근하려고 하면, 로그인 페이지 `/login`으로 보내버릴 수 있습니다.

이때, 단순히 다른 페이지로 이동만 시키는 것이 아니라, **"게시글 작성이 성공적으로 완료되었습니다!"** 와 같은 메시지를 보여주거나, **방금 생성된 리소스의 ID** 같은 정보를 다음 페이지에 전달하고 싶을 때가 많습니다. 이것이 리다이렉트 시 데이터 전달이 필요한 이유입니다.

**스프링 MVC에서 리다이렉트 시 데이터 전달의 기본 동작**

스프링 MVC 컨트롤러 메소드에서 문자열 앞에 `redirect:` 접두사를 붙여 반환하면 리다이렉트가 일어납니다. (예: `return "redirect:/home";`)

이때, 컨트롤러 메소드 내에서 `Model` 객체에 추가된 데이터(속성)들은 기본적으로 다음과 같이 처리됩니다:

1. **URI 템플릿 변수로 사용:** 리다이렉트 URL에 `{변수명}`과 같은 형태로 정의된 부분이 있다면, 모델에 같은 이름의 속성이 있을 경우 그 값으로 치환됩니다.
  - 예: `model.addAttribute("articleId", 123); return "redirect:/articles/{articleId}";` -> 브라우저는 `/articles/123`으로 리다이렉트됩니다.
2. **쿼리 파라미터로 자동 추가:** 위에서 사용되지 않은 나머지 모델 속성들 중,
  - **기본 타입 (primitive types):** `int`, `long`, `String`, `boolean` 등
  - **기본 타입의 컬렉션 또는 배열:** `List<String>`, `String[]` 등
    이런 속성들은 URL 뒤에 `?이름=값&이름2=값2` 형태로 자동으로 추가됩니다 (쿼리 파라미터).
  - 예: `model.addAttribute("message", "success"); model.addAttribute("count", 10); return "redirect:/list";` -> 브라우저는 `/list?message=success&count=10`으로 리다이렉트될 수 있습니다.

**문제점: 원치 않는 데이터 노출**

"기본 동작 2번"이 편리할 때도 있지만, 문제가 될 수도 있습니다. 컨트롤러가 화면 렌더링을 위해 모델에 여러 데이터를 담아두는 경우가 많은데 (예: 드롭다운 목록에 필요한 코드 값들, 사용자 정보 객체 등), 리다이렉트 시 이 모든 정보가 URL에 쿼리 파라미터로 노출될 수 있습니다. 이는 URL을 지저분하게 만들고, 때로는 민감한 정보를 노출시킬 수도 있습니다.

**해결책: `RedirectAttributes` 사용**

`RedirectAttributes` 인터페이스는 이러한 문제를 해결하고, 리다이렉트 시 전달할 데이터를 명시적으로 제어할 수 있게 해줍니다. 컨트롤러 메소드의 매개변수로 `RedirectAttributes`를 선언하고, 여기에 추가한 속성들만 리다이렉트 시 사용됩니다.

- **`addAttribute(String name, Object value)`:** 이 메소드로 추가된 속성은 URI 템플릿 변수로 사용되거나, 기본 타입일 경우 쿼리 파라미터로 URL에 노출됩니다. (기본 동작과 유사하지만, 명시적으로 선택)
- **`addFlashAttribute(String name, Object value)`:** 이 메소드로 추가된 속성은 **URL에 노출되지 않습니다.** 대신, 일시적으로 HTTP 세션에 저장되었다가, 리다이렉트된 후 딱 한 번만 읽히고 세션에서 자동으로 제거됩니다. 이를 "플래시 속성(Flash Attribute)"이라고 하며, 주로 성공/실패 메시지 같이 일회성으로 보여줄 정보를 전달할 때 유용합니다. (이 챕터에서는 간단히 언급만 하고, 다음 챕터에서 자세히 다룹니다.)

**`RedirectAttributes` 사용 시 동작:**

- 메소드가 `redirect:`로 리다이렉트를 수행하면: `RedirectAttributes`에 추가된 내용만 사용됩니다. (기존 `Model` 내용은 무시될 수 있음 - 아래 `ignoreDefaultModelOnRedirect` 참고)
- 메소드가 리다이렉트하지 않고 뷰를 렌더링하면: 기존 `Model`의 내용이 사용됩니다. (`RedirectAttributes`에 추가한 내용도 `Model`에 포함되어 뷰로 전달됩니다.)

### **주요 용어 해설**

- **리다이렉트 (Redirect):** 클라이언트에게 다른 URL로 재요청하도록 지시하는 HTTP 응답 메커니즘 (주로 HTTP 상태 코드 302 Found 또는 301 Moved Permanently 사용).
- **모델 속성 (Model Attribute):** 컨트롤러에서 뷰로 데이터를 전달하기 위해 `Model` 객체에 저장하는 이름-값 쌍.
- **URI 템플릿 변수 (URI Template Variable):** URL 경로의 일부를 변수처럼 사용하여 동적으로 URL을 구성하는 방식. 예: `/users/{userId}`에서 `{userId}`.
- **쿼리 파라미터 (Query Parameter):** URL의 `?` 뒤에 `이름=값` 형태로 추가되어 서버에 부가적인 데이터를 전달하는 방식. 예: `/search?keyword=spring&page=1`.
- **`RedirectAttributes`:** 스프링 MVC 인터페이스로, 리다이렉트 시 전달할 속성을 명시적으로 관리하고, 플래시 속성을 사용할 수 있게 합니다.
- **플래시 속성 (Flash Attribute):** 리다이렉트 시 URL에 노출되지 않고 HTTP 세션을 통해 일시적으로 전달되는 데이터. 리다이렉트 후 딱 한 번 사용되고 자동 제거됩니다.
- **`RequestMappingHandlerAdapter`:** 스프링 MVC에서 `@RequestMapping` 어노테이션이 붙은 메소드를 처리하는 핵심 컴포넌트.
- **`ignoreDefaultModelOnRedirect`:** `RequestMappingHandlerAdapter`의 설정 플래그. `true`로 설정하면 리다이렉트 시 컨트롤러의 기본 `Model` 내용을 무시하고, `RedirectAttributes`에 명시된 내용만 사용하거나 아무것도 전달하지 않도록 강제합니다.
  - 기본값: `false` (하위 호환성 때문)
  - **권장값 (새 애플리케이션): `true`** (예측 가능하고 안전한 동작을 위해)

### **코드 예제 및 분석**

**1. 기본적인 리다이렉트 (URI 템플릿 변수 자동 사용)**

```java
@PostMapping("/files/{path}") // 1. 경로 변수 {path}를 사용
public String upload(@PathVariable String path, /* ... 다른 매개변수 ... */ Model model) {
	// 파일 업로드 로직 등 ...
	boolean success = true; // 업로드 성공 여부

	if (success) {
        // 2. 모델에 아무것도 추가하지 않아도, {path}는 리다이렉트 URL에 자동으로 사용됨
		return "redirect:files/{path}"; // 3. 예: /files/images 로 리다이렉트 (만약 path가 "images"였다면)
	} else {
		model.addAttribute("errorMessage", "Upload failed!");
		return "uploadForm"; // 리다이렉트 안 함, 뷰 렌더링
	}
}
```

**코드 분석:**

1. `@PostMapping("/files/{path}")`: `/files/어떤경로` 형태의 POST 요청을 처리합니다. `{path}`는 URI 템플릿 변수입니다.
2. 이 예제에서는 `Model`이나 `RedirectAttributes`에 `{path}`와 관련된 값을 명시적으로 추가하지 않았습니다.
3. `return "redirect:files/{path}";`: 리다이렉트 URL을 지정합니다. 스프링 MVC는 현재 요청의 URI 템플릿 변수(`{path}`) 값을 자동으로 인식하여, 리다이렉트 URL의 `{path}` 부분에 채워 넣습니다. 예를 들어, 클라이언트가 `/files/user-docs`로 요청했다면, 리다이렉트 URL은 `files/user-docs`가 됩니다.

**2. `RedirectAttributes`를 사용하여 데이터 전달하기**

```java
import org.springframework.stereotype.Controller;
import org.springframework.web.bind.annotation.PostMapping;
import org.springframework.web.bind.annotation.RequestParam;
import org.springframework.web.servlet.mvc.support.RedirectAttributes;

@Controller
public class OrderController {

    @PostMapping("/orders")
    public String createOrder(@RequestParam String product,
                              @RequestParam int quantity,
                              RedirectAttributes redirectAttributes) {
        // 주문 생성 로직 (예: DB에 저장)
        Long orderId = 123L; // 생성된 주문 ID라고 가정

        // 1. RedirectAttributes에 속성 추가 (쿼리 파라미터로 전달될 것)
        redirectAttributes.addAttribute("orderId", orderId);
        redirectAttributes.addAttribute("statusMessage", "OrderCreatedSuccessfully");

        // 2. 플래시 속성 추가 (URL에 노출되지 않고 세션으로 전달)
        redirectAttributes.addFlashAttribute("successMessage", "주문이 성공적으로 완료되었습니다!");

        // 3. 리다이렉트 URL. orderId는 URI 템플릿 변수로, statusMessage는 쿼리 파라미터로
        //    실제로는 orderId를 URI 템플릿으로 사용하는 것이 더 RESTful 할 수 있음
        //    예: return "redirect:/orders/" + orderId; (addAttribute로 orderId를 추가했으므로 자동 치환)
        return "redirect:/order/confirmation";
    }
}

```

리다이렉트 후 `/order/confirmation`을 처리하는 컨트롤러:

```java
import org.springframework.stereotype.Controller;
import org.springframework.ui.Model;
import org.springframework.web.bind.annotation.GetMapping;
import org.springframework.web.bind.annotation.RequestParam;
import java.util.Map;

@Controller
public class OrderConfirmationController {

    // RedirectAttributes.addAttribute로 추가한 값 받기 (쿼리 파라미터)
    @GetMapping("/order/confirmation")
    public String showConfirmation(@RequestParam Long orderId,
                                   @RequestParam String statusMessage,
                                   Model model) { // Model에는 FlashAttribute로 전달된 값도 자동으로 포함됨

        System.out.println("Order ID from query param: " + orderId);
        System.out.println("Status Message from query param: " + statusMessage);

        // FlashAttribute는 모델에 자동으로 추가되므로, 뷰에서 바로 사용 가능
        // @ModelAttribute("successMessage") String message 와 같이 받을 수도 있음
        // 또는 model.getAttribute("successMessage") 로 접근 가능

        if (model.containsAttribute("successMessage")) {
            System.out.println("Success Message from flash attribute: " + model.getAttribute("successMessage"));
        }

        model.addAttribute("confirmedOrderId", orderId); // 뷰에 전달할 데이터 추가

        return "orderConfirmationPage"; // 뷰 이름
    }
}

```

**코드 분석 (`OrderController`):**

1. `redirectAttributes.addAttribute("orderId", orderId);redirectAttributes.addAttribute("statusMessage", "OrderCreatedSuccessfully");`
  - `orderId`와 `statusMessage`를 `RedirectAttributes`에 추가합니다.
  - 만약 리다이렉트 URL이 `/order/confirmation/{orderId}`였다면, `orderId`는 URI 템플릿 변수로 치환됩니다.
  - 현재 예제에서는 리다이렉트 URL이 `/order/confirmation`이므로, `orderId`와 `statusMessage`는 기본 타입이므로 쿼리 파라미터로 추가됩니다.
  - 즉, 실제 리다이렉트되는 URL은 `/order/confirmation?orderId=123&statusMessage=OrderCreatedSuccessfully` 와 같이 됩니다.
2. `redirectAttributes.addFlashAttribute("successMessage", "주문이 성공적으로 완료되었습니다!");`
  - `successMessage`는 플래시 속성으로 추가됩니다. 이 값은 URL에 나타나지 않고, 세션을 통해 리다이렉트 대상에게 전달됩니다.
3. `return "redirect:/order/confirmation";`
  - 지정된 URL로 리다이렉트합니다.

**코드 분석 (`OrderConfirmationController`):**

- `@RequestParam Long orderId, @RequestParam String statusMessage`: `RedirectAttributes.addAttribute`로 추가되어 URL의 쿼리 파라미터로 넘어온 값들을 받습니다.
- `Model model`: 여기에 `RedirectAttributes.addFlashAttribute`로 추가했던 `successMessage`가 자동으로 담겨져 옵니다. 그래서 뷰 (예: `orderConfirmationPage.html`)에서는 특별한 작업 없이 `successMessage` 값을 바로 사용할 수 있습니다. (예: Thymeleaf라면 `th:text="${successMessage}"`)

### **"왜?" 라는 질문에 대한 답변**

**`RedirectAttributes`와 관련 기능들은 왜 필요할까요?**

1. **안전하고 깔끔한 URL 유지:** 가장 큰 이유는 리다이렉트 시 URL에 불필요하거나 민감한 정보가 쿼리 파라미터로 노출되는 것을 방지하기 위함입니다. `RedirectAttributes`를 사용하면 전달할 데이터를 명시적으로 선택할 수 있습니다.
2. **일회성 메시지 전달의 표준화된 방법 (플래시 속성):** "작업 성공/실패"와 같은 메시지는 리다이렉트된 페이지에서 한 번만 보여주고 사라지는 것이 자연스럽습니다. 플래시 속성은 이를 HTTP 세션을 이용하여 깔끔하게 구현할 수 있는 표준적인 방법을 제공합니다. 직접 세션을 다루고, 사용 후 제거하는 번거로운 로직을 작성할 필요가 없습니다.
3. **예측 가능한 데이터 전달:** `ignoreDefaultModelOnRedirect = true` 설정을 사용하면, 리다이렉트 시 어떤 데이터가 전달될지 명확하게 예측하고 제어할 수 있습니다. 의도치 않게 `Model`에 남아있던 데이터가 URL에 붙는 상황을 방지하여 버그 발생 가능성을 줄입니다.
4. **PRG 패턴 (Post/Redirect/Get) 지원 용이:** 웹 개발에서 폼 제출(POST) 후 새로고침 시 중복 제출을 방지하기 위해 PRG 패턴을 많이 사용합니다.
  - **P (POST):** 사용자가 폼을 제출하면 서버는 데이터를 처리합니다.
  - **R (Redirect):** 처리가 완료되면 서버는 클라이언트에게 다른 URL(주로 GET 요청을 위한 결과 페이지)로 리다이렉트하라고 응답합니다. 이때 `RedirectAttributes`를 사용하여 결과 메시지 등을 전달할 수 있습니다.
  - **G (GET):** 클라이언트는 새로운 URL로 GET 요청을 보내고, 서버는 결과 페이지를 보여줍니다. 이 페이지에서 새로고침해도 GET 요청이 반복될 뿐, 이전의 POST 요청이 중복 실행되지 않습니다.
    `RedirectAttributes`는 이 PRG 패턴에서 R 단계의 데이터 전달을 효과적으로 돕습니다.

### **주의사항 및 Best Practice**

1. **`ignoreDefaultModelOnRedirect = true` 사용 권장:** 새 애플리케이션에서는 이 플래그를 `true`로 설정하는 것이 좋습니다. 이렇게 하면 `Model`의 내용이 실수로 리다이렉트 URL에 포함되는 것을 막고, `RedirectAttributes`를 통해서만 데이터를 전달하도록 강제하여 코드의 명확성을 높입니다. (Spring Boot에서는 `spring.mvc.ignore-default-model-on-redirect=true`로 설정 가능)
2. **URI 템플릿 변수 vs. 쿼리 파라미터 vs. 플래시 속성 선택 기준:**
  - **URI 템플릿 변수 (`addAttribute` + URL에 `{}` 사용):** 리소스의 고유 식별자처럼, URL 경로 자체에 의미를 부여하는 데이터에 적합합니다. (예: `/orders/{orderId}`)
  - **쿼리 파라미터 (`addAttribute`):** 필터링, 정렬, 페이지네이션 정보 등 URL에 명시적으로 보여도 괜찮고, 북마크 가능해야 하는 데이터에 적합합니다. 너무 많은 데이터를 쿼리 파라미터로 전달하는 것은 피해야 합니다.
  - **플래시 속성 (`addFlashAttribute`):** 일회성 알림 메시지, 임시 데이터 등 URL에 노출시키고 싶지 않고, 리다이렉트 후 한 번만 사용될 데이터에 매우 유용합니다.
3. **플래시 속성의 저장 메커니즘:** 플래시 속성은 HTTP 세션에 저장됩니다. 따라서 세션이 유지되지 않는 환경이거나, 너무 많은 데이터를 플래시 속성으로 사용하면 세션 저장 공간에 부담을 줄 수 있다는 점을 인지해야 합니다. (일반적인 메시지 전달에는 문제없습니다.)
4. **리다이렉트 URL은 완전한 URL 또는 상대 경로:** `redirect:` 접두사 뒤에는 `/`로 시작하는 컨텍스트 상대 경로, 현재 요청의 상대 경로, 또는 `http://`로 시작하는 완전한 URL을 사용할 수 있습니다.
5. **데이터 전달 시 보안 고려:** 아무리 플래시 속성을 사용한다고 해도, 매우 민감한 정보(예: 개인 식별 번호 전체)를 리다이렉트를 통해 직접 전달하는 것은 신중해야 합니다. 가능하다면 ID만 전달하고, 리다이렉트된 페이지에서 해당 ID로 다시 데이터를 조회하는 것이 더 안전할 수 있습니다.

### **이전 학습 내용과의 연관성**

- **`Model` 객체:** 이전 컨트롤러 로직에서 뷰로 데이터를 전달하기 위해 `Model`을 사용했습니다. 리다이렉트 상황에서는 이 `Model`의 데이터가 어떻게 처리되는지, 그리고 `RedirectAttributes`가 `Model`과 어떻게 상호작용하는지를 이해하는 것이 중요합니다.
- **HTTP 요청/응답 흐름:** 리다이렉트는 클라이언트와 서버 간의 또 다른 요청/응답 사이클을 발생시킵니다. 첫 번째 요청에 대한 응답이 리다이렉트 지시이고, 클라이언트는 그 지시에 따라 두 번째 요청을 보내는 것입니다. 이 과정에서 데이터가 어떻게 유지되고 전달되는지를 파악해야 합니다.
- **세션 (Session):** 플래시 속성은 HTTP 세션을 사용합니다. 아직 세션에 대해 자세히 배우지 않았더라도, "사용자별로 서버에 잠시 데이터를 저장해두는 공간" 정도로 이해하고 넘어가면 됩니다. 나중에 `@SessionAttributes`나 세션 직접 사용을 배울 때 이 개념이 다시 등장할 것입니다.

---
