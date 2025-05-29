---
title: Spring Web MVC - Annotated Controllers (@CookieValue)
description: 
author: laze
date: 2025-05-29 00:00:01 +0900
categories: [Dev, SpringWebMVC]
tags: [SpringWebMVC]
---
### @CookieValue

> Reactive 스택에서의 동일 기능은 해당 문서를 참고하세요.
>

`@CookieValue` 어노테이션을 사용하면 HTTP 쿠키의 값을 컨트롤러의 메소드 매개변수에 바인딩할 수 있습니다.

다음과 같은 쿠키를 가진 요청을 생각해 봅시다:

```
JSESSIONID=415A4AC178C59DACE0B2C9CA727CDD84
```

다음 예제는 쿠키 값을 가져오는 방법을 보여줍니다:

**Java**

```java
@GetMapping("/demo")
public void handle(@CookieValue("JSESSIONID") String cookie) {
	//...
}
```

- `JSESSIONID` 쿠키의 값을 가져옵니다.

만약 대상 메소드 매개변수의 타입이 `String`이 아니라면, 타입 변환이 자동으로 적용됩니다.

---

### **학습 목표 제시**

이 챕터를 학습하고 나면, 학생은 다음을 이해하고 수행할 수 있게 됩니다:

1. **`@CookieValue` 어노테이션의 역할과 사용법:** HTTP 요청에 포함된 쿠키의 특정 값을 컨트롤러 메소드에서 어떻게 가져와 활용하는지 이해합니다.
2. **쿠키 값의 자동 타입 변환 이해:** 쿠키 값을 `String` 뿐만 아니라 다른 타입으로 변환하여 받는 방법과 스프링의 자동 타입 변환 기능을 이해합니다.
3. **필수 쿠키와 선택적 쿠키 처리 방법:** 특정 쿠키가 요청에 반드시 포함되어야 하는 경우와 그렇지 않은 경우를 구분하여 처리하는 방법을 이해합니다.

---

### **핵심 개념 설명**

**`@CookieValue`란 무엇일까요?**

우리가 웹사이트를 이용하다 보면, "로그인 정보 유지", "장바구니 상품 기억", "오늘 하루 이 창 보지 않기" 같은 편리한 기능들을 자주 접하게 됩니다. 이런 기능들은 대부분 **쿠키(Cookie)**라는 기술을 사용해서 구현됩니다.

**쿠키는 웹사이트가 사용자의 웹 브라우저에 저장하는 작은 텍스트 조각**입니다. 사용자가 웹사이트를 다시 방문했을 때, 브라우저는 저장된 쿠키를 서버로 다시 보내줍니다. 그러면 서버는 이 쿠키 정보를 보고 "아, 이 사용자는 지난번에 로그인했던 그 사람이구나!" 또는 "장바구니에 이런 상품들을 담아뒀었네!" 하고 사용자를 기억하거나 이전 상태를 이어갈 수 있게 됩니다.

`@CookieValue` 어노테이션은 이렇게 **클라이언트(웹 브라우저)가 요청과 함께 보내온 HTTP 쿠키들 중에서 특정 쿠키의 값을 스프링 MVC 컨트롤러의 메소드 안으로 쉽게 가져올 수 있도록 도와주는 역할**을 합니다.

**비유:**

여러분이 단골 카페에 자주 간다고 상상해 보세요.

- **첫 방문:**
  - **손님 (클라이언트):** "아메리카노 한 잔 주세요."
  - **점원 (서버):** 주문을 받고, 손님에게 "혹시 저희 카페 쿠폰 있으세요? 없으시면 하나 만들어 드릴게요. 여기에 도장 찍어두시면 다음에 할인받으실 수 있어요." 라며 **쿠폰(쿠키)**을 줍니다.
- **재방문:**
  - **손님 (클라이언트):** "안녕하세요, 아메리카노 한 잔이요." (이때, 지난번에 받은 **쿠폰(쿠키)**을 함께 제시합니다.)
  - **점원 (서버의 컨트롤러):** 손님이 제시한 쿠폰을 보고, "아, 단골손님이시네요! 쿠폰에 도장이 몇 개 찍혔는지 볼까요?" 라고 합니다.

여기서 `@CookieValue`는 점원이 손님이 제시한 쿠폰(쿠키)에서 "도장이 몇 개 찍혔는지" 또는 "손님 ID가 무엇인지"와 같은 특정 정보를 쏙 빼내어 확인하는 것과 같습니다.

예를 들어, 웹사이트에 로그인하면 서버는 `sessionId=사용자고유ID`와 같은 쿠키를 브라우저에 저장하도록 합니다. 이후 사용자가 다른 페이지를 요청할 때마다 브라우저는 이 `sessionId` 쿠키를 함께 보내고, 서버는 `@CookieValue("sessionId") String sessionId`와 같은 코드를 통해 이 값을 읽어 사용자를 식별할 수 있습니다.

### **주요 용어 해설**

- **쿠키 (Cookie):** 웹 서버가 사용자의 웹 브라우저에 저장하고, 이후 해당 브라우저가 서버에 요청을 보낼 때마다 함께 전송되는 작은 데이터 조각입니다. 상태 유지(로그인 정보, 장바구니), 사용자 추적, 개인화 설정 등에 사용됩니다.
- **`@CookieValue`:** 스프링 MVC 어노테이션으로, HTTP 요청의 쿠키 값을 컨트롤러 메소드의 매개변수에 바인딩합니다.
- **`JSESSIONID`:** 자바 기반 웹 애플리케이션에서 세션 관리를 위해 기본적으로 사용되는 쿠키의 이름입니다. 각 사용자별 세션을 식별하는 고유한 ID 값을 가집니다. (이름은 서버 설정에 따라 변경될 수 있습니다.)
- **세션 (Session):** 사용자가 웹사이트에 접속해서 브라우저를 닫을 때까지 일련의 요청과 응답을 하나의 상태로 관리하는 기술입니다. 쿠키는 이 세션 ID를 클라이언트에 저장하는 일반적인 방법 중 하나입니다.

### **코드 예제 및 분석**

제공된 원문의 Java 코드를 다시 살펴보겠습니다.

```java
@GetMapping("/demo") // 1. "/demo" 경로로 GET 요청이 오면 이 메소드가 처리합니다.
public void handle(@CookieValue("JSESSIONID") String cookie) { // 2. "JSESSIONID" 쿠키 값을 cookie 변수에 담습니다.
	// 3. 여기서 cookie 변수(JSESSIONID 값)를 사용하여
	//    쿠키 값에 따른 로직을 처리할 수 있습니다.
	//    예: System.out.println("JSESSIONID: " + cookie);
}
```

**코드 분석:**

1. `@GetMapping("/demo")`: 이 어노테이션은 HTTP GET 요청 중에서 URL 경로가 `/demo`인 요청을 `handle` 메소드가 처리하도록 지정합니다.
2. `@CookieValue("JSESSIONID") String cookie`:
  - `@CookieValue("JSESSIONID")`: "클라이언트가 보낸 HTTP 요청에 포함된 쿠키 중에서 `JSESSIONID`라는 이름의 쿠키를 찾아서..."
  - `String cookie`: "... 그 쿠키의 값을 `String` 타입의 `cookie`라는 매개변수에 넣어줘!" 라는 의미입니다.
  - 만약 클라이언트가 `Cookie: JSESSIONID=415A4AC178C59DACE0B2C9CA727CDD84` 라는 헤더를 포함하여 요청을 보냈다면, `cookie` 변수에는 `"415A4AC178C59DACE0B2C9CA727CDD84"` 라는 문자열이 할당됩니다.
3. 메소드 본문 `//...`: 이 부분에서는 `cookie` 변수에 담긴 `JSESSIONID` 값을 사용하여 다양한 로직을 수행할 수 있습니다. 예를 들어, 이 세션 ID를 기반으로 사용자의 로그인 상태를 확인하거나, 사용자별 데이터를 조회하는 등의 작업을 할 수 있습니다.

**타입 변환:**

`@RequestHeader`와 마찬가지로 `@CookieValue`도 자동 타입 변환을 지원합니다. 만약 쿠키 값이 숫자로 저장되어 있고 이를 `int`나 `long` 타입으로 받고 싶다면, 매개변수 타입을 해당 타입으로 지정하기만 하면 됩니다.

```java
@GetMapping("/userPrefs")
public void handleUserPrefs(
    @CookieValue("fontSize") int fontSize, // "fontSize" 쿠키 값을 int 타입으로 받음
    @CookieValue(name = "darkMode", defaultValue = "false") boolean darkMode) { // "darkMode" 쿠키 값을 boolean 타입으로 받음 (없으면 false)
    // fontSize와 darkMode 값을 사용하여 사용자 맞춤 설정 등을 처리
    System.out.println("Font Size: " + fontSize);
    System.out.println("Dark Mode: " + darkMode);
}
```

만약 `fontSize=16` 쿠키와 `darkMode=true` 쿠키가 전달되었다면, `fontSize` 변수에는 숫자 `16`이, `darkMode` 변수에는 불리언 `true`가 할당됩니다.

### **"왜?" 라는 질문에 대한 답변**

**`@CookieValue`는 왜 필요할까요? 어떤 문제를 해결하기 위해 등장했을까요?**

1. **표준화되고 선언적인 방법으로 쿠키 값 접근:** `@CookieValue`가 없다면, 서블릿 API의 `HttpServletRequest` 객체를 사용하고, `request.getCookies()` 메소드로 모든 쿠키 배열을 가져온 다음, 반복문을 돌면서 원하는 이름의 쿠키를 찾아 그 값을 얻어야 했습니다. 이는 코드를 길고 복잡하게 만듭니다.

    ```java
    // @CookieValue 사용 안 할 경우 (번거로운 방식)
    Cookie[] cookies = request.getCookies();
    String sessionId = null;
    if (cookies != null) {
        for (Cookie c : cookies) {
            if ("JSESSIONID".equals(c.getName())) {
                sessionId = c.getValue();
                break;
            }
        }
    }
    ```

   `@CookieValue`는 어노테이션 하나로 이 모든 과정을 단순화하여 **코드의 가독성과 생산성을 크게 향상**시킵니다.

2. **자동 타입 변환의 편리함:** `Cookie.getValue()` 메소드는 쿠키 값을 항상 `String`으로 반환합니다. 만약 숫자로 된 쿠키 값을 사용하려면 `Integer.parseInt()` 등의 변환 코드가 필요합니다. `@CookieValue`는 스프링의 타입 변환 서비스를 통해 **자동으로 원하는 타입으로 변환**해주므로, 개발자는 이러한 변환 로직을 신경 쓸 필요가 없습니다.
3. **필수 쿠키/선택적 쿠키 처리 용이성:** 특정 쿠키가 반드시 있어야 하는지, 아니면 없어도 되는지, 없을 경우 기본값을 사용할지 등을 `@CookieValue`의 속성(`required`, `defaultValue`)을 통해 쉽게 정의할 수 있습니다. 이는 `if (cookie == null)`과 같은 null 체크 로직을 줄여줍니다.
4. **테스트 용이성 증대:** 컨트롤러 메소드가 `HttpServletRequest`에 직접 의존하는 것보다, `@CookieValue`를 통해 쿠키 값을 단순 매개변수로 주입받으면 단위 테스트 작성이 더 쉬워집니다.

결국 `@CookieValue`는 `@RequestHeader`와 유사하게, 개발자가 HTTP 요청의 특정 부분(여기서는 쿠키)에 더 쉽고, 안전하며, 스프링다운 방식으로 접근할 수 있도록 돕는 편리한 도구입니다.

### **주의사항 및 Best Practice**

1. **필수 쿠키 vs 선택적 쿠키 (Default 값 처리):**
  - 기본적으로 `@CookieValue`로 지정된 쿠키가 요청에 없으면 스프링 MVC는 오류를 발생시킵니다 (MissingRequestCookieException).
  - 만약 특정 쿠키가 필수가 아니라 선택적으로 존재할 수 있다면, `required` 속성을 `false`로 설정하고 필요하다면 `defaultValue`를 지정할 수 있습니다.

      ```java
      @GetMapping("/settings")
      public void handleSettings(
              @CookieValue(name = "theme", required = false) String theme, // "theme" 쿠키는 선택적
              @CookieValue(name = "trackingConsent", defaultValue = "false") boolean trackingConsent) { // "trackingConsent" 쿠키가 없으면 false
          // theme 쿠키가 없으면 theme 변수는 null이 됩니다.
          System.out.println("Theme: " + theme);
          System.out.println("Tracking Consent: " + trackingConsent);
      }
      
      ```

2. **쿠키 값의 보안:**
  - 쿠키는 클라이언트 측(브라우저)에 저장되므로, 사용자가 임의로 수정하거나 내용을 볼 수 있습니다. 따라서 **민감한 정보(예: 비밀번호, 개인 식별 정보)를 쿠키에 직접 저장하는 것은 매우 위험합니다.**
  - 만약 민감한 정보를 다뤄야 한다면, 암호화하거나, 서버 측 세션에 저장하고 세션 ID만 쿠키로 사용하는 것이 좋습니다.
  - `HttpOnly` 플래그가 설정된 쿠키는 JavaScript를 통해 접근할 수 없으므로 XSS(Cross-Site Scripting) 공격을 일부 방어하는 데 도움이 됩니다. 스프링에서 쿠키를 생성할 때 이 플래그를 설정할 수 있습니다.
  - `Secure` 플래그가 설정된 쿠키는 HTTPS 연결을 통해서만 전송됩니다.
3. **쿠키의 크기 및 개수 제한:** 브라우저마다 쿠키에 저장할 수 있는 데이터의 크기(보통 도메인당 약 4KB)와 총 쿠키 개수(보통 도메인당 수십 개)에 제한이 있습니다. 너무 많은 정보나 큰 데이터를 쿠키에 저장하려고 하면 문제가 발생할 수 있습니다.
4. **쿠키 이름:** 쿠키 이름은 일반적으로 대소문자를 구분합니다. `@CookieValue`에 지정하는 쿠키 이름이 실제 쿠키의 이름과 정확히 일치하는지 확인해야 합니다.
5. **모든 쿠키 값 가져오기 (고급):**`@RequestHeader`처럼 모든 쿠키를 `Map`으로 한 번에 받는 직접적인 어노테이션 지원은 `@CookieValue`에는 명시적으로 없습니다. 만약 모든 쿠키를 확인해야 한다면, 여전히 `HttpServletRequest` 객체를 사용하거나 `@RequestHeader("Cookie") String allCookiesString`와 같이 Cookie 헤더 전체를 문자열로 받은 후 파싱해야 할 수 있습니다.
   하지만 보통은 특정 이름의 쿠키 값만 필요한 경우가 대부분입니다.

    ```java
    // 모든 쿠키를 보고 싶을 때 (HttpServletRequest 사용)
    @GetMapping("/allCookies")
    public void showAllCookies(HttpServletRequest request) {
        Cookie[] cookies = request.getCookies();
        if (cookies != null) {
            for (Cookie cookie : cookies) {
                System.out.println(cookie.getName() + "=" + cookie.getValue());
            }
        }
    }
    ```


### **이전 학습 내용과의 연관성**

- **`@RequestHeader` 와의 유사점 및 차이점:**
  - **유사점:** `@CookieValue`와 `@RequestHeader`는 모두 클라이언트 요청으로부터 특정 정보를 추출하여 컨트롤러 메소드 매개변수에 바인딩하는 어노테이션입니다. 둘 다 `required`, `defaultValue` 속성을 제공하며, 자동 타입 변환을 지원합니다.
  - **차이점:** `@RequestHeader`는 일반적인 HTTP 헤더 값을 대상으로 하는 반면, `@CookieValue`는 HTTP 헤더 중에서도 특히 `Cookie` 헤더 내에 있는 특정 이름의 쿠키 값을 대상으로 합니다. 쿠키는 헤더보다 좀 더 구조화된 (이름=값 쌍들의 집합) 형태를 가집니다.
- **HTTP 요청 구조의 이해:** 이번 챕터를 통해 HTTP 요청 메시지가 단순히 URL과 본문(body)으로만 구성된 것이 아니라, 헤더(Header)라는 영역에 다양한 부가 정보(쿠키 포함)를 담아 전달한다는 것을 다시 한번 확인할 수 있습니다. `@RequestHeader`는 그 헤더 영역을, `@CookieValue`는 헤더 중에서도 쿠키 부분을 다루는 도구인 셈이죠.

---
