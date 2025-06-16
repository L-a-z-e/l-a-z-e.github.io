---
title: Spring Web MVC - Model
description: 
author: laze
date: 2025-06-16 00:00:01 +0900
categories: [Dev, SpringWebMVC]
tags: [SpringWebMVC]
---
### Model (모델)

`@ModelAttribute` 어노테이션은 다음과 같이 사용할 수 있습니다:

1. `@RequestMapping` 메소드의 인자(argument)에 사용하여 모델로부터 객체를 생성하거나 접근하고, `WebDataBinder`를 통해 요청에 바인딩합니다. (이전에 커맨드 객체 배울 때 봤던 방식)
2. `@Controller` 또는 `@ControllerAdvice` 클래스 내 메소드 레벨 어노테이션으로 사용하여, 어떤 `@RequestMapping` 메소드가 호출되기 전에 모델을 초기화하는 데 도움을 줍니다.
3. `@RequestMapping` 메소드에 사용하여 반환 값이 모델 속성(model attribute)임을 표시합니다.

이 섹션은 위 목록의 두 번째 항목인 `@ModelAttribute` 메소드에 대해 설명합니다.

컨트롤러는 여러 개의 `@ModelAttribute` 메소드를 가질 수 있습니다.

이러한 모든 메소드는 동일한 컨트롤러 내의 `@RequestMapping` 메소드보다 먼저 호출됩니다.

`@ModelAttribute` 메소드는 `@ControllerAdvice`를 통해 여러 컨트롤러 간에 공유될 수도 있습니다.

자세한 내용은 컨트롤러 어드바이스(Controller Advice) 섹션을 참고하세요.

`@ModelAttribute` 메소드는 유연한 메소드 시그니처를 가집니다.

`@ModelAttribute` 자체나 요청 본문과 관련된 것을 제외하고, `@RequestMapping` 메소드와 동일한 많은 인자들을 지원합니다.

다음 예제는 `@ModelAttribute` 메소드를 보여줍니다:

```java
@ModelAttribute // 이 메소드는 @RequestMapping 메소드 실행 전에 호출됨
public void populateModel(@RequestParam String number, Model model) { // Model 객체를 받아 속성 추가
	model.addAttribute(accountRepository.findAccount(number)); // number로 계좌 조회 후 모델에 추가
	// 더 많은 속성 추가 가능 ...
}
```

다음 예제는 단 하나의 속성만 추가합니다 (메소드 반환 값을 모델 속성으로):

```java
@ModelAttribute // 이 메소드는 @RequestMapping 메소드 실행 전에 호출됨
public Account addAccount(@RequestParam String number) { // 반환되는 Account 객체가 모델에 추가됨
	return accountRepository.findAccount(number);
}
```

이름이 명시적으로 지정되지 않으면, `Conventions`의 javadoc에 설명된 대로 객체 타입을 기반으로 기본 이름이 선택됩니다.

오버로드된 `addAttribute` 메소드를 사용하거나 (반환 값의 경우) `@ModelAttribute`의 `name` 속성을 통해 항상 명시적인 이름을 할당할 수 있습니다.

또한 `@ModelAttribute`를 `@RequestMapping` 메소드 레벨 어노테이션으로 사용할 수도 있으며, 이 경우 `@RequestMapping` 메소드의 반환 값이 모델 속성으로 해석됩니다.

반환 값이 뷰 이름으로 해석될 수 있는 `String`이 아닌 한, HTML 컨트롤러에서는 이것이 기본 동작이므로 일반적으로 필요하지 않습니다.

`@ModelAttribute`는 다음 예제와 같이 모델 속성 이름을 사용자 정의할 수도 있습니다:

**Java**

```java
@GetMapping("/accounts/{id}")
@ModelAttribute("myAccount") // 이 메소드의 반환 값은 "myAccount"라는 이름으로 모델에 추가됨
public Account handle() {
	// ...
	Account account = ... ;
	return account;
}

```

---

### **학습 목표 제시**

이 챕터를 학습하고 나면, 학생은 다음을 이해하고 수행할 수 있게 됩니다:

1. **`@ModelAttribute` 메소드의 핵심 역할 및 실행 시점 이해:** `@Controller` 내에서 `@ModelAttribute` 어노테이션이 붙은 메소드가 `@RequestMapping` 메소드 실행 전에 호출되어 모델을 초기화하는 데 사용된다는 것을 이해합니다.
2. **다양한 방식으로 모델에 데이터 추가하는 방법 숙지:** `@ModelAttribute` 메소드에서 `Model` 객체를 직접 사용하거나, 메소드의 반환 값을 이용하여 모델에 속성을 추가하는 두 가지 주요 방법을 이해하고 적용할 수 있습니다.
3. **모델 속성 이름 지정 규칙 이해:** `@ModelAttribute` 메소드를 통해 추가되는 모델 속성의 이름이 어떻게 결정되는지(기본 규칙, 명시적 지정) 이해합니다. (`@ModelAttribute`를 `@RequestMapping` 메소드에 사용하는 경우는 이전 내용과 중복되므로, 주로 메소드 레벨 `@ModelAttribute`에 집중합니다.)

---

### **핵심 개념 설명**

**`@ModelAttribute` 메소드란 무엇일까요?**

여러분이 어떤 웹사이트의 특정 컨트롤러(예: 상품 관련 기능을 담당하는 `ProductController`)를 만든다고 상상해봅시다. 이 컨트롤러 안에는 여러 개의 요청 처리 메소드(`@RequestMapping` 또는 `@GetMapping`, `@PostMapping` 등)가 있을 수 있습니다. 예를 들어, 상품 목록 보기, 상품 상세 정보 보기, 상품 추천 목록 보기 등.

이때, 이 여러 요청 처리 메소드들이 화면(뷰)을 렌더링할 때 **공통적으로 필요한 데이터**가 있을 수 있습니다. 예를 들면, 모든 상품 관련 페이지 상단에 표시될 "현재 진행 중인 이벤트 목록"이나, 상품 카테고리 드롭다운 목록에 필요한 "전체 카테고리 리스트" 같은 것들이죠.

`@ModelAttribute` 어노테이션이 붙은 메소드(이하 `@ModelAttribute` 메소드)는 바로 이러한 **공통 모델 데이터를 준비하는 역할**을 합니다. 이 메소드는 해당 컨트롤러 내의 **어떤 `@RequestMapping` 메소드가 호출되기 전에 항상 먼저 실행**됩니다. 그리고 `@ModelAttribute` 메소드에서 `Model` 객체에 추가한 데이터는 이후에 실행될 `@RequestMapping` 메소드에서 그대로 사용할 수 있게 됩니다.

**비유:**

여러분이 연극 공연을 준비한다고 생각해봅시다.

- **컨트롤러:** 연극 연출가
- **`@RequestMapping` 메소드들:** 각 장(scene)을 연기하는 배우들 (예: 1장 배우, 2장 배우)
- **`@ModelAttribute` 메소드:** 무대 담당 스태프
- **`Model` 객체:** 무대 위에 놓일 소품들

배우들이 각 장을 연기하기 전에( `@RequestMapping` 메소드가 실행되기 전에), 무대 담당 스태프(`@ModelAttribute` 메소드)가 먼저 무대에 올라가서 모든 장에서 공통적으로 필요한 소품들(예: 배경 그림, 기본 가구)을 미리 준비해둡니다(`Model`에 데이터 추가). 그러면 각 장의 배우들은 이미 준비된 소품들을 활용하여 연기를 펼칠 수 있습니다.

**`@ModelAttribute` 메소드의 주요 특징 및 실행 흐름:**

1. **선행 실행:** 동일한 컨트롤러 내에서, 어떤 `@RequestMapping` 메소드가 특정 URL 요청을 받아 실행되기 직전에, 해당 컨트롤러 내에 있는 모든 `@ModelAttribute` 메소드들이 먼저 호출됩니다. (만약 여러 개의 `@ModelAttribute` 메소드가 있다면, 그들 사이의 실행 순서는 보장되지 않으므로 서로 의존하지 않도록 작성하는 것이 좋습니다.)
2. **모델 초기화:** 주된 목적은 `Model` 객체에 데이터를 추가하여, 이후 실행될 `@RequestMapping` 메소드와 그 메소드가 반환할 뷰(View)에서 사용할 수 있도록 하는 것입니다.
3. **유연한 시그니처:** `@RequestMapping` 메소드와 유사하게 다양한 타입의 매개변수를 가질 수 있습니다. (예: `HttpServletRequest`, `@RequestParam`, `@PathVariable`, 그리고 `Model` 객체 자체 등). 단, `@ModelAttribute` 어노테이션이 붙은 매개변수나 요청 본문 관련 매개변수(`@RequestBody`, `HttpEntity`)는 가질 수 없습니다.
4. **`@ControllerAdvice`를 통한 전역 적용:** `@ModelAttribute` 메소드를 `@ControllerAdvice` 어노테이션이 붙은 클래스 내에 정의하면, 특정 컨트롤러뿐만 아니라 애플리케이션 전역의 여러 컨트롤러에 걸쳐 공통 모델 데이터를 제공할 수도 있습니다. (이 부분은 "Controller Advice" 챕터에서 더 자세히 다룹니다.)

### **주요 용어 해설**

- **`@ModelAttribute` 어노테이션:**
  - **메소드 레벨:** 해당 메소드가 모델을 초기화하는 역할을 함을 나타냅니다. `@RequestMapping` 메소드 실행 전에 호출됩니다.
  - **매개변수 레벨:** `@RequestMapping` 메소드의 매개변수에 사용하여, HTTP 요청 파라미터를 해당 매개변수 객체(커맨드 객체)에 바인딩하거나, 모델에 이미 있는 객체를 가져오는 역할을 합니다. (이번 챕터의 주 내용은 아님)
  - **메소드 반환 값 레벨 (on `@RequestMapping` method):** `@RequestMapping` 메소드의 반환 값을 특정 이름의 모델 속성으로 추가할 때 사용합니다. (이번 챕터의 주 내용은 아님)
- **`Model` 인터페이스:** 컨트롤러에서 뷰로 데이터를 전달하는 데 사용되는 객체입니다. `addAttribute(String name, Object value)` 메소드를 통해 데이터를 추가합니다.
- **`Conventions` 클래스:** 스프링 내부에서 사용되는, 이름 생성 등에 대한 규칙(convention)을 정의한 클래스. `@ModelAttribute`로 추가되는 속성의 기본 이름을 결정할 때 이 규칙이 사용됩니다.

### **코드 예제 및 분석**

원문의 예제들을 중심으로 살펴보겠습니다.

**1. `Model` 객체를 직접 사용하여 여러 속성 추가**

```java
import org.springframework.stereotype.Controller;
import org.springframework.ui.Model;
import org.springframework.web.bind.annotation.GetMapping;
import org.springframework.web.bind.annotation.ModelAttribute;
import org.springframework.web.bind.annotation.RequestParam;

// 가상의 Account 및 AccountRepository
class Account { public String id; public String name; public Account(String id, String name){ this.id=id; this.name=name;} }
class AccountRepository { public Account findAccount(String number){ return new Account(number, "User " + number); } }

@Controller
public class AccountController {

    private AccountRepository accountRepository = new AccountRepository(); // 예시용

    // 1. @ModelAttribute 메소드: populateModel
    @ModelAttribute
    public void populateModel(@RequestParam(required = false) String accountNumber, // accountNumber 파라미터가 있다면 사용
                              Model model) { // Model 객체를 매개변수로 받음
        System.out.println("populateModel called. accountNumber: " + accountNumber);
        if (accountNumber != null) {
            // 2. accountNumber로 계좌를 찾아 모델에 "account"라는 이름으로 추가 (기본 이름 규칙)
            model.addAttribute(accountRepository.findAccount(accountNumber));
        }
        // 3. 다른 공통 데이터도 모델에 추가할 수 있음
        model.addAttribute("commonInfo", "This is common information for all account views.");
    }

    // 4. @RequestMapping 메소드: showAccount
    @GetMapping("/accountDetails")
    public String showAccountDetails(Model model) {
        System.out.println("showAccountDetails called.");
        // populateModel에서 추가한 "account"와 "commonInfo"를 여기서 사용할 수 있음
        // 만약 accountNumber 파라미터 없이 /accountDetails만 호출했다면, "account"는 없을 수 있음
        // (populateModel의 if (accountNumber != null) 조건 때문)
        if (model.containsAttribute("account")) {
            Account acc = (Account) model.getAttribute("account");
            System.out.println("Account from model: " + acc.name);
        }
        System.out.println("Common Info from model: " + model.getAttribute("commonInfo"));
        return "accountView"; // 뷰 이름 반환
    }
}

```

**코드 분석:**

1. `@ModelAttribute public void populateModel(...)`:
  - 이 메소드는 `@GetMapping("/accountDetails")` 메소드인 `showAccountDetails`가 호출되기 **전에** 실행됩니다.
  - `@RequestParam(required = false) String accountNumber`: URL에 `accountNumber` 파라미터가 있다면 그 값을 받습니다 (없어도 오류는 아님).
  - `Model model`: 스프링 MVC가 `Model` 객체를 주입해줍니다.
2. `model.addAttribute(accountRepository.findAccount(accountNumber));`:
  - 만약 `accountNumber`가 존재하면, 해당 계좌 정보를 조회하여 `Account` 객체를 `Model`에 추가합니다.
  - 이때 `model.addAttribute(Object)` 형태로 객체만 전달했으므로, 모델에 추가되는 속성의 이름은 **기본 규칙**에 따라 결정됩니다. `Account` 타입의 객체이므로, 기본적으로 "account"라는 이름으로 추가됩니다 (클래스 이름의 첫 글자를 소문자로 바꾼 형태).
3. `model.addAttribute("commonInfo", "This is common information for all account views.");`: "commonInfo"라는 이름으로 문자열 데이터를 모델에 명시적으로 추가합니다.
4. `@GetMapping("/accountDetails") public String showAccountDetails(Model model)`:
  - 이 메소드가 실행될 때는 `populateModel` 메소드에 의해 모델에 "account"(조건부)와 "commonInfo"가 이미 추가된 상태입니다.
  - 따라서 `showAccountDetails` 메소드 내부나, 이 메소드가 반환하는 뷰("accountView")에서 해당 데이터들을 사용할 수 있습니다.

**2. 메소드 반환 값을 모델 속성으로 추가 (단일 속성 추가 시 간결)**

```java
// ... (Account, AccountRepository, AccountController 클래스 상단은 위와 동일) ...
@Controller
public class AnotherAccountController { // 클래스 이름 변경하여 구분

    private AccountRepository accountRepository = new AccountRepository();

    // 1. @ModelAttribute 메소드: addAccountToModel
    // 이 메소드가 반환하는 Account 객체가 모델에 추가됨
    @ModelAttribute // ("account" 라는 이름으로 모델에 추가됨 - 기본 규칙)
    public Account addAccountToModel(@RequestParam String accountNumber) {
        System.out.println("addAccountToModel called with accountNumber: " + accountNumber);
        return accountRepository.findAccount(accountNumber);
    }

    // 또는, 모델 속성 이름을 명시적으로 지정하고 싶다면:
    // @ModelAttribute("myCustomAccount")
    // public Account addCustomNamedAccountToModel(@RequestParam String accountNumber) {
    //     return accountRepository.findAccount(accountNumber);
    // }

    @GetMapping("/displayAccount")
    public String displayAccountPage(Model model) { // accountNumber 파라미터와 함께 호출되어야 함
        System.out.println("displayAccountPage called.");
        // addAccountToModel에서 반환한 Account 객체가 "account"라는 이름으로 모델에 이미 들어있음
        Account acc = (Account) model.getAttribute("account");
        if (acc != null) {
            System.out.println("Account from model (via return value): " + acc.name);
        }
        return "accountDisplayView";
    }
}

```

**코드 분석:**

1. `@ModelAttribute public Account addAccountToModel(@RequestParam String accountNumber)`:
  - 이 메소드는 `String` 타입의 `accountNumber` 요청 파라미터를 필수로 받습니다. (만약 `accountNumber`가 없으면 오류 발생)
  - 메소드가 `Account` 객체를 반환합니다.
  - `@ModelAttribute` 어노테이션에 `name`이나 `value` 속성으로 이름을 지정하지 않았으므로, 반환된 `Account` 객체는 **기본 이름 규칙**에 따라 "account"라는 이름으로 `Model`에 자동으로 추가됩니다.
  - 만약 `@ModelAttribute("myCustomAccount")`와 같이 이름을 명시했다면, "myCustomAccount"라는 이름으로 모델에 추가됩니다.

**모델 속성 이름 결정 규칙 (복습 및 정리):**

- **`model.addAttribute("이름", 값)` 사용 시:** 명시적으로 지정한 "이름"으로 추가됩니다.
- **`model.addAttribute(값)` 사용 시 (객체만 전달):**
  - 객체 타입이 `Collection`이고 비어있지 않은 경우: 컬렉션의 첫 번째 요소 타입 이름에 "List"를 붙인 형태 (예: `List<Account>` -> `accountList`).
  - 그 외의 경우: 객체의 클래스 이름에서 첫 글자만 소문자로 바꾼 형태 (예: `User` -> `user`, `OrderDetail` -> `orderDetail`). (이는 `org.springframework.core.Conventions.getVariableName(Object)` 메소드의 로직을 따릅니다.)
- **`@ModelAttribute` 메소드가 값을 반환하는 경우:**
  - `@ModelAttribute("이름")`으로 이름을 명시하면 그 이름으로 추가됩니다.
  - 이름을 명시하지 않으면 위 `model.addAttribute(값)`과 동일한 기본 이름 규칙이 적용됩니다.

**3. `@RequestMapping` 메소드에 `@ModelAttribute`를 사용하여 반환 값을 모델 속성으로 지정 (주로 이름 커스터마이징 목적)**

원문의 마지막 예제:

```java
@Controller
public class YetAnotherAccountController {

    @GetMapping("/accounts/{id}")
    @ModelAttribute("myAccount") // 1. 이 메소드의 반환 값을 "myAccount"라는 이름의 모델 속성으로 추가
    public Account handle(@PathVariable String id) {
        Account account = new Account(id, "User " + id); // 가상 데이터
        // ... 로직 ...
        return account; // 2. 이 Account 객체가 "myAccount"로 모델에 담김
    }

    // 만약 위 메소드가 String을 반환하고, 그 String이 뷰 이름이 아니라
    // 모델에 "myAccount"라는 이름으로 담고 싶다면 @ModelAttribute가 의미 있음.
    // 하지만 객체를 반환하는 경우, @ResponseBody가 없다면
    // 어차피 반환된 객체는 기본 이름(여기서는 "account")으로 모델에 담기므로,
    // 이름을 바꾸고 싶을 때만 @ModelAttribute("이름")을 씀.
}
```

**코드 분석:**

1. `@ModelAttribute("myAccount")`: `@GetMapping` 메소드 자체에 `@ModelAttribute`를 적용했습니다. 이는 `handle` 메소드가 반환하는 `Account` 객체를 `Model`에 추가할 때, 기본 이름("account") 대신 "myAccount"라는 이름으로 추가하라는 의미입니다.
2. `return account;`: `Account` 객체를 반환합니다.
  - **중요:** 이 컨트롤러가 `@RestController`가 아니고, 메소드에 `@ResponseBody`도 없다면, 스프링 MVC는 기본적으로 반환된 객체(`account`)를 모델에 추가하고 (이때 `@ModelAttribute("myAccount")`에 의해 이름이 "myAccount"로 지정됨), 뷰를 렌더링하려고 시도합니다. (만약 뷰 이름이 명시적으로 반환되지 않으면, 요청 경로 기반으로 뷰 이름을 추론하려 할 수 있습니다 - `RequestToViewNameTranslator`.)
  - 원문 설명처럼, 객체를 반환하는 HTML 컨트롤러(뷰 렌더링 컨트롤러)에서는 반환 값이 기본적으로 모델에 추가됩니다. 따라서 `@ModelAttribute`를 `@RequestMapping` 메소드에 붙이는 경우는 주로 **모델 속성의 이름을 기본 규칙과 다르게 지정하고 싶을 때** 사용됩니다.
  - 만약 반환 타입이 `String`이라면, 스프링은 기본적으로 그 문자열을 뷰 이름으로 해석하려 합니다. 이때 `@ModelAttribute`를 붙이면 "이건 뷰 이름이 아니라 모델 속성이야!"라고 알려주는 효과가 있습니다. (하지만 이런 경우는 `Model` 객체를 직접 매개변수로 받아 `addAttribute`하는 것이 더 명확할 수 있습니다.)

### **"왜?" 라는 질문에 대한 답변**

**`@ModelAttribute` 메소드는 왜 필요할까요? 그냥 각 `@RequestMapping` 메소드 안에서 필요한 데이터를 모델에 추가하면 안 되나?**

1. **코드 중복 제거 (DRY - Don't Repeat Yourself):** 여러 `@RequestMapping` 메소드에서 동일한 데이터를 조회하거나 생성하여 모델에 추가해야 하는 경우, 각 메소드마다 중복된 코드를 작성해야 합니다. `@ModelAttribute` 메소드를 사용하면 이 공통 로직을 한 곳에 모아 중복을 제거하고 코드를 더 깔끔하게 유지할 수 있습니다.
2. **관심사의 분리 (Separation of Concerns):**
  - `@RequestMapping` 메소드는 주로 요청 처리, 비즈니스 로직 호출, 다음 행동(뷰 반환, 리다이렉트 등) 결정이라는 핵심 관심사에 집중할 수 있습니다.
  - 모델에 필요한 공통 데이터를 준비하는 책임은 `@ModelAttribute` 메소드에게 위임함으로써 각 메소드의 역할을 더 명확하게 분리할 수 있습니다.
3. **가독성 및 유지보수성 향상:** 공통 데이터 준비 로직이 한 곳에 모여 있으므로, 코드를 이해하기 쉽고, 나중에 공통 데이터에 변경이 필요할 때도 해당 `@ModelAttribute` 메소드만 수정하면 되므로 유지보수성이 향상됩니다.
4. **컨트롤러 레벨의 기본 모델 설정:** 특정 컨트롤러의 모든 뷰 페이지에서 항상 필요한 데이터(예: 웹사이트 공통 메뉴, 사용자 로그인 상태 정보 등 특정 범위)를 일관되게 제공하는 데 유용합니다.

물론, 아주 간단한 경우나 특정 `@RequestMapping` 메소드에서만 필요한 데이터라면 해당 메소드 내에서 직접 모델에 추가하는 것이 더 직관적일 수 있습니다. `@ModelAttribute` 메소드는 **"여러 곳에서 반복적으로 사용되는 모델 데이터"** 를 처리할 때 그 진가를 발휘합니다.

### **주의사항 및 Best Practice**

1. **실행 순서:** `@ModelAttribute` 메소드는 `@RequestMapping` 메소드보다 먼저 실행됩니다. 여러 `@ModelAttribute` 메소드 간의 실행 순서는 보장되지 않으므로, 서로 의존적인 로직을 작성하는 것은 피해야 합니다.
2. **성능 고려:** `@ModelAttribute` 메소드는 매 요청마다 실행될 수 있습니다. 만약 이 메소드 내부에서 무거운 DB 조회나 복잡한 연산을 수행한다면 성능에 영향을 줄 수 있습니다. 이런 경우에는 캐싱 전략을 함께 고려하거나, 정말 매번 필요한 데이터인지 다시 한번 생각해보는 것이 좋습니다.
3. **과도한 사용 지양:** 모든 것을 `@ModelAttribute` 메소드로 빼내려고 하기보다는, 정말 공통적으로 필요한 데이터에 대해서만 사용하는 것이 좋습니다. 과도하게 사용하면 오히려 코드 흐름을 파악하기 어려워질 수 있습니다.
4. **이름 충돌 주의:** `@ModelAttribute` 메소드를 통해 추가되는 속성의 이름과 `@RequestMapping` 메소드에서 모델에 추가하는 속성의 이름이 같다면, 나중에 추가된 값으로 덮어쓰여질 수 있습니다. 명확한 이름을 사용하거나, `@ModelAttribute("이름")`으로 명시적으로 지정하여 충돌을 피하는 것이 좋습니다.
5. **`@ControllerAdvice`와의 조합:** 전역적으로 필요한 모델 데이터는 `@ControllerAdvice`와 함께 `@ModelAttribute` 메소드를 사용하여 관리하는 것이 효과적입니다.

### **이전 학습 내용과의 연관성**

- **`Model` 인터페이스:** `@ModelAttribute` 메소드는 결국 `Model` 객체에 데이터를 추가하는 작업을 수행합니다. `Model`의 역할과 사용법을 이해하고 있어야 합니다.
- **`@RequestParam`, `@PathVariable` 등:** `@ModelAttribute` 메소드의 매개변수로 이러한 어노테이션들을 사용하여 요청으로부터 값을 받아 모델 데이터를 준비하는 데 활용할 수 있습니다.
- **커맨드 객체 (`@ModelAttribute` on parameter):** 이전에 배운, `@RequestMapping` 메소드의 매개변수에 `@ModelAttribute` (또는 생략 가능)를 사용하여 폼 데이터를 객체에 바인딩하는 것과는 다른 용법입니다. 이번 챕터는 **메소드 자체에 `@ModelAttribute`를 붙여 모델을 초기화하는 기능**에 초점을 맞췄습니다. 이 두 가지 용법을 잘 구분하는 것이 중요합니다.
- **`@ControllerAdvice` (미리보기):** `@ModelAttribute` 메소드를 `@ControllerAdvice` 클래스에 정의하면 전역적인 모델 초기화가 가능해집니다. 이는 나중에 배울 내용과 연결됩니다.

---
