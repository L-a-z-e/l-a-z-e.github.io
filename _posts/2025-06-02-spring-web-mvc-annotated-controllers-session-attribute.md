---
title: Spring Web MVC - Annotated Controllers (@SessionAttribute)
description: 
author: laze
date: 2025-06-02 00:00:02 +0900
categories: [Dev, SpringWebMVC]
tags: [SpringWebMVC]
---
## @SessionAttribute

[Reactive 스택의 동일 기능 보기](https://docs.spring.io/spring-framework/docs/current/reference/html/web-reactive.html#webflux-ann-sessionattribute)

만약 전역적으로 관리되는 (즉, 컨트롤러 외부에서 - 예를 들어, 필터에 의해) 기존 세션 속성에 접근해야 하고, 해당 속성이 존재할 수도 있고 존재하지 않을 수도 있는 경우,

다음 예제와 같이 메서드 파라미터에 `@SessionAttribute` 애노테이션을 사용할 수 있습니다:

**Java**

```java
@RequestMapping("/")
public String handle(@SessionAttribute User user) { // ①
    // ...
}
```

① `@SessionAttribute` 애노테이션 사용.

**Kotlin**

```kotlin
@RequestMapping("/")
fun handle(@SessionAttribute user: User): String { // ①
    // ...
    return "someView"
}
```

① `@SessionAttribute` 애노테이션 사용.

세션 속성을 추가하거나 제거해야 하는 사용 사례의 경우, `org.springframework.web.context.request.WebRequest` 또는 `jakarta.servlet.http.HttpSession`을 컨트롤러 메서드에 주입하는 것을 고려하십시오.

컨트롤러 워크플로우의 일부로 세션에 모델 속성을 임시로 저장하는 경우, [@SessionAttributes에 설명된 대로 `@SessionAttributes`](https://docs.spring.io/spring-framework/docs/current/reference/html/web.html#mvc-ann-sessionattributes) 사용을 고려하십시오.

---

**✨ 이 챕터의 학습 목표 ✨**

1. **`@SessionAttribute` 애노테이션이 이미 HTTP 세션에 존재하는 특정 이름의 속성 값을 컨트롤러 메서드의 파라미터로 가져오는 역할을 한다는 것을 이해합니다.** (주로 읽기 전용 접근)
2. **`@SessionAttribute`는 해당 세션 속성이 존재하지 않을 경우 어떻게 동작하는지 (기본적으로 예외 발생) 및 `required=false` 속성을 통해 이를 처리하는 방법을 인지합니다.**
3. **`@SessionAttribute`가 `@SessionAttributes` (복수형)와 어떻게 다른지, 그리고 세션에 데이터를 직접 추가하거나 제거할 때는 어떤 방법을 사용해야 하는지 구분합니다.**

---

### 핵심 개념 설명

**`@SessionAttribute`의 주된 역할: 기존 세션 데이터 읽기**

`@SessionAttribute` 애노테이션은 컨트롤러 메서드의 파라미터에 사용하여, **이미 HTTP 세션에 저장되어 있는 특정 이름의 속성(attribute) 값**을 해당 파라미터에 바인딩(할당)하는 역할을 합니다.

- **컨트롤러 외부에서 관리되는 세션 데이터:** 이 애노테이션은 주로 현재 컨트롤러가 직접 관리하지 않는 (즉, `@SessionAttributes`로 선언하지 않은) 세션 데이터를 읽어올 때 유용합니다. 예를 들어, 로그인 처리 필터(Filter)나 인터셉터(Interceptor)에서 사용자 정보를 세션에 저장해두었을 경우, 컨트롤러에서는 `@SessionAttribute`를 사용해 그 사용자 정보를 간편하게 가져올 수 있습니다.
- **읽기 중심:** `@SessionAttribute`는 기본적으로 세션에서 값을 *읽어오는* 데 초점을 맞춥니다. 이 애노테이션 자체로는 세션에 새로운 값을 쓰거나 기존 값을 변경/삭제하는 기능은 제공하지 않습니다. (물론, 가져온 객체의 내부 상태를 변경할 수는 있지만, 세션 저장소 자체를 조작하는 것은 아닙니다.)

**`@SessionAttributes` (복수형)와의 차이점:**

| 특징 | `@SessionAttributes` (복수형) | `@SessionAttribute` (단수형) |
| --- | --- | --- |
| **선언 위치** | 컨트롤러 클래스 레벨 | 컨트롤러 메서드 파라미터 |
| **주요 목적** | 특정 모델 속성을 세션에 **저장하고 관리**(읽기/쓰기/제거 사이클) | 이미 세션에 있는 특정 속성을 **읽어오기** |
| **데이터 관리 주체** | 해당 컨트롤러가 관리하는 세션 데이터의 라이프사이클을 담당 | 컨트롤러 외부에서 관리되는 세션 데이터를 단순히 참조 |
| **세션 데이터 생성** | 모델에 해당 이름/타입의 객체가 추가될 때 세션에 **자동 생성/저장** | 세션에 해당 이름의 속성이 **이미 존재해야 함** (기본적으로) |
| **데이터 제거** | `SessionStatus.setComplete()`로 해당 컨트롤러 관리 데이터 제거 | 이 애노테이션 자체로는 세션 데이터 제거 기능 없음 (별도 방법 사용) |

**비유를 들어볼까요?**

- 여러분 집에 **개인 서재(`@SessionAttributes` 관리 영역)**와 **공용 거실 책장(전역 세션 영역)**이 있다고 상상해봅시다.
  - **`@SessionAttributes("myBook")`**: 여러분이 개인 서재에 "myBook"이라는 이름표를 붙인 책꽂이를 마련하고, 거기에 책을 넣고 빼고 관리하는 것과 같습니다. 이 책은 여러분만 주로 신경 씁니다.
  - **`@SessionAttribute("sharedMagazine")`**: 거실 책장에 이미 꽂혀 있는 "sharedMagazine"이라는 잡지를 잠시 가져와서 읽는 것과 같습니다. 이 잡지는 가족 누군가가 가져다 놓았을 수 있고(예: 필터), 여러분은 그냥 읽기만 합니다. 읽고 나서 다시 제자리에 둘 수도 있고, 다른 가족이 치울 수도 있습니다. 여러분이 이 잡지를 거실 책장에서 직접 없애거나 새로 가져다 놓는 주된 책임자는 아닙니다.

### `@SessionAttribute` 사용법 및 속성

```java
@Controller
public class ExampleController {

    // 1. 기본적인 사용: 세션에 "currentUser" 속성이 반드시 있어야 함
    @GetMapping("/user/info")
    public String getUserInfo(@SessionAttribute("currentUser") User user, Model model) {
        // 세션에서 "currentUser" 이름으로 저장된 User 객체를 가져와 user 파라미터에 바인딩
        model.addAttribute("userInfo", user);
        return "userInfoPage";
    }

    // 2. 속성 이름 생략: 파라미터 변수명을 세션 속성 이름으로 사용
    @GetMapping("/user/dashboard")
    public String showDashboard(@SessionAttribute User user, Model model) { // "user"라는 이름의 세션 속성을 찾음
        model.addAttribute("dashboardData", user.getDashboardSettings());
        return "dashboardPage";
    }

    // 3. 선택적 속성 처리: 세션에 해당 속성이 없어도 예외 발생 안 함
    @GetMapping("/cart/summary")
    public String getCartSummary(
            @SessionAttribute(name = "shoppingCart", required = false) Cart cart, // "shoppingCart" 세션 속성이 없으면 cart는 null
            Model model) {

        if (cart == null) {
            model.addAttribute("message", "장바구니가 비어있습니다.");
        } else {
            model.addAttribute("cartItems", cart.getItems());
        }
        return "cartSummaryPage";
    }
}
```

**주요 속성:**

- `name` (또는 `value`): 가져올 세션 속성의 이름을 지정합니다. 생략하면 메서드 파라미터의 변수 이름을 사용합니다.
- `required`: (기본값: `true`)
  - `true`: 세션에 해당 이름의 속성이 반드시 존재해야 합니다. 없으면 `HttpSessionRequiredException` (또는 유사 예외)이 발생합니다.
  - `false`: 세션에 해당 이름의 속성이 없어도 예외가 발생하지 않고, 해당 파라미터에는 `null`이 할당됩니다. (자바 기본 타입의 경우 예외 발생 가능성 있음, 래퍼 타입 사용 권장)

**타입 변환:**

세션에 저장된 객체가 `@SessionAttribute`로 선언된 파라미터 타입과 다를 경우, 스프링의 타입 변환 서비스(`ConversionService`)를 통해 호환 가능한 타입으로 변환을 시도합니다.

### 세션 속성 추가/제거 시 고려사항 (원문 내용)

원문에서는 `@SessionAttribute`가 주로 **읽기**에 사용된다고 강조하며, 세션 속성을 **추가하거나 제거**해야 하는 경우에는 다음 방법을 고려하라고 안내합니다.

1. **`org.springframework.web.context.request.WebRequest` 주입:**

    ```java
    @PostMapping("/session/add")
    public String addToSession(WebRequest request, @RequestParam String data) {
        request.setAttribute("myData", data, WebRequest.SCOPE_SESSION); // 세션에 "myData" 추가
        return "redirect:/somePage";
    }
    
    @PostMapping("/session/remove")
    public String removeFromSession(WebRequest request) {
        request.removeAttribute("myData", WebRequest.SCOPE_SESSION); // 세션에서 "myData" 제거
        return "redirect:/somePage";
    }
    ```

  - `WebRequest`는 현재 요청 및 세션 범위에 대한 일반적인 접근 인터페이스를 제공합니다.
  - `setAttribute(name, value, scope)`: 특정 범위( `SCOPE_REQUEST`, `SCOPE_SESSION` )에 속성을 설정합니다.
  - `removeAttribute(name, scope)`: 특정 범위에서 속성을 제거합니다.
2. **`jakarta.servlet.http.HttpSession` 직접 주입:**

    ```java
    import jakarta.servlet.http.HttpSession;
    
    @PostMapping("/session/addLegacy")
    public String addToSessionLegacy(HttpSession session, @RequestParam String data) {
        session.setAttribute("legacyData", data); // 세션에 "legacyData" 추가
        return "redirect:/somePage";
    }
    
    @PostMapping("/session/removeLegacy")
    public String removeFromSessionLegacy(HttpSession session) {
        session.removeAttribute("legacyData"); // 세션에서 "legacyData" 제거
        // session.invalidate(); // 세션 전체를 무효화할 수도 있음 (주의!)
        return "redirect:/somePage";
    }
    ```

  - 전통적인 서블릿 API인 `HttpSession` 객체를 직접 주입받아 사용할 수 있습니다.
  - `setAttribute(name, value)`와 `removeAttribute(name)` 메서드를 사용합니다.

**왜 `@SessionAttribute`는 쓰기/삭제 기능을 제공하지 않을까?**

`@SessionAttribute`의 설계 철학은 "이미 존재하는 (그리고 아마도 다른 곳에서 관리되는) 세션 데이터를 편리하게 읽어오자"는 것입니다. 만약 이 애노테이션이 쓰기/삭제 기능까지 갖게 되면, `@SessionAttributes` (복수형)와의 역할이 모호해지고, 세션 데이터 관리의 책임 소재가 불분명해질 수 있습니다. 따라서 명확한 역할 분담을 위해 읽기 기능에 집중하는 것입니다.

### 임시 저장 vs. 전역 세션 속성 (원문 내용)

원문 마지막 부분에서는 다시 한번 `@SessionAttributes` (복수형)와 `@SessionAttribute` (단수형)의 사용 목적을 구분해줍니다.

- **`@SessionAttributes` (복수형):** 컨트롤러 내의 특정 워크플로우(예: 여러 단계 폼 처리)를 위해 모델 속성을 **임시로 세션에 저장**하고, 해당 워크플로우가 끝나면 `SessionStatus.setComplete()`로 정리하는 용도에 적합합니다. (컨트롤러가 관리하는 "단기" 세션 데이터)
- **`@SessionAttribute` (단수형):** 컨트롤러 외부(예: 필터, 인터셉터, 다른 시스템)에서 이미 생성되어 **전역적으로 관리될 수 있는 세션 속성**에 접근(주로 읽기)하는 용도에 적합합니다. (컨트롤러가 직접 관리하지 않는 "장기 또는 외부" 세션 데이터)

### 주의사항 및 Best Practice

1. **`required = false` 사용 고려:** 접근하려는 세션 속성이 항상 존재한다는 보장이 없다면, `required = false`를 설정하여 예외 발생을 방지하고 `null` 체크를 통해 안전하게 처리하는 것이 좋습니다.
2. **타입 안전성:** `@SessionAttribute`로 가져올 객체의 타입과 실제 세션에 저장된 객체의 타입이 일치하거나 호환 가능해야 합니다. 타입 불일치 시 `ClassCastException` 등이 발생할 수 있습니다.
3. **세션 데이터의 생명주기 인지:** `@SessionAttribute`로 읽어오는 데이터는 누가, 언제 생성하고, 언제 제거하는지에 대한 이해가 필요합니다. 해당 컨트롤러가 그 생명주기를 직접 관리하지 않기 때문입니다.
4. **`@SessionAttributes`와의 명확한 구분:** 두 애노테이션의 이름이 비슷하여 혼동하기 쉽습니다. 's'의 유무에 따라 기능과 사용 목적이 완전히 다르다는 것을 명심해야 합니다.

### 이전 학습 내용과의 연관성

- **`@SessionAttributes` (복수형):** `@SessionAttribute` (단수형)의 가장 중요한 비교 대상입니다. 역할과 사용법의 차이를 명확히 이해해야 합니다.
- **HTTP 세션 기본:** HTTP 세션이 무엇이고 왜 필요한지에 대한 기본적인 이해가 `@SessionAttribute`의 사용 맥락을 파악하는 데 도움이 됩니다.
- **컨트롤러 메서드 파라미터 바인딩:** `@RequestParam`, `@PathVariable`, `@ModelAttribute` 등과 마찬가지로 `@SessionAttribute`도 특정 소스(여기서는 HTTP 세션)로부터 값을 가져와 메서드 파라미터에 바인딩하는 스프링 MVC의 일반적인 메커니즘을 따릅니다.

---
