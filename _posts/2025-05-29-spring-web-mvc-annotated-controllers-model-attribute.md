---
title: Spring Web MVC - Annotated Controllers (@ModelAttribute)
description: 
author: laze
date: 2025-05-29 00:00:03 +0900
categories: [Dev, SpringWebMVC]
tags: [SpringWebMVC]
---
## @ModelAttribute

[Reactive 스택의 동일 기능 보기](https://docs.spring.io/spring-framework/docs/current/reference/html/web-reactive.html#webflux-ann-modelattribute)

`@ModelAttribute` 메서드 파라미터 애노테이션은 요청 파라미터, URI 경로 변수, 그리고 요청 헤더를 모델 객체에 바인딩합니다. 예를 들어:

**Java**

```java
@PostMapping("/owners/{ownerId}/pets/{petId}/edit")
public String processSubmit(@ModelAttribute Pet pet) { // ①
    // 메서드 로직...
}
```

①`Pet`의 인스턴스에 바인딩합니다.

**Kotlin**

```kotlin
@PostMapping("/owners/{ownerId}/pets/{petId}/edit")
fun processSubmit(@ModelAttribute pet: Pet): String { // ①
    // 메서드 로직...
    return "someView"
}

```

① `Pet`의 인스턴스에 바인딩합니다.

요청 파라미터는 요청 본문의 폼 데이터와 쿼리 파라미터를 포함하는 서블릿 API 개념입니다. URI 변수와 헤더도 포함되지만, 동일한 이름의 요청 파라미터를 덮어쓰지 않는 경우에만 해당됩니다.

헤더 이름에서 대시는 제거됩니다.

위의 `Pet` 인스턴스는 다음 중 하나일 수 있습니다:

- `@ModelAttribute` 메서드에 의해 추가되었을 수 있는 모델에서 접근합니다.
- 클래스 수준의 `@SessionAttributes` 애노테이션에 모델 속성이 나열된 경우 HTTP 세션에서 접근합니다.
- 모델 속성 이름이 경로 변수나 요청 파라미터와 같은 요청 값의 이름과 일치하는 경우 `Converter`를 통해 얻어옵니다 (예제는 아래에 나옵니다).
- 기본 생성자를 통해 인스턴스화됩니다.
- 서블릿 요청 파라미터와 일치하는 인자를 가진 "주 생성자(primary constructor)"를 통해 인스턴스화됩니다. 인자 이름은 바이트코드에 런타임 시 유지되는 파라미터 이름을 통해 결정됩니다.

위에서 언급했듯이, 모델 속성 이름이 경로 변수나 요청 파라미터와 같은 요청 값의 이름과 일치하고 호환되는 `Converter<String, T>`가 있는 경우, `Converter<String, T>`를 사용하여 모델 객체를 얻을 수 있습니다.

아래 예제에서 모델 속성 이름 `account`는 URI 경로 변수 `account`와 일치하며, 아마도 영속성 저장소에서 이를 검색하는 등록된 `Converter<String, Account>`가 있습니다:

**Java**

```java
@PutMapping("/accounts/{account}")
public String save(@ModelAttribute("account") Account account) {
    // ...
}
```

**Kotlin**

```kotlin
@PutMapping("/accounts/{account}")
fun save(@ModelAttribute("account") account: Account): String {
    // ...
    return "someView"
}
```

기본적으로 생성자 바인딩과 프로퍼티 데이터 바인딩이 모두 적용됩니다.

그러나 모델 객체 설계는 신중한 고려가 필요하며, 보안상의 이유로 웹 바인딩을 위해 특별히 맞춤화된 객체를 사용하거나 생성자 바인딩만 적용하는 것이 좋습니다.

여전히 프로퍼티 바인딩을 사용해야 하는 경우, 설정할 수 있는 프로퍼티를 제한하기 위해 `allowedFields` 패턴을 설정해야 합니다.

이에 대한 자세한 내용과 예제 구성은 [모델 디자인](https://docs.spring.io/spring-framework/docs/current/reference/html/web.html#mvc-ann-modelattrib-design)을 참조하십시오.

생성자 바인딩을 사용할 때, `@BindParam` 애노테이션을 통해 요청 파라미터 이름을 사용자 정의할 수 있습니다. 예를 들어:

**Java**

```java
class Account {

    private final String firstName;

    public Account(@BindParam("first-name") String firstName) {
        this.firstName = firstName;
    }
}
```

**Kotlin**

```kotlin
class Account(@BindParam("first-name") val firstName: String)
```

`@BindParam`은 생성자 파라미터에 해당하는 필드에도 위치할 수 있습니다.

`@BindParam`은 기본적으로 지원되지만, `DataBinder`에 `DataBinder.NameResolver`를 설정하여 다른 애노테이션을 사용할 수도 있습니다.

생성자 바인딩은 단일 문자열(예: 쉼표로 구분된 리스트)에서 변환되거나, `accounts[2].name` 또는 `account[KEY].name`과 같은 인덱싱된 키를 기반으로 하는 `List`, `Map`, 배열 인자를 지원합니다.

경우에 따라 데이터 바인딩 없이 모델 속성에 접근하고 싶을 수 있습니다.

이러한 경우, `Model`을 컨트롤러에 주입하여 직접 접근하거나, 다음 예제와 같이 `@ModelAttribute(binding=false)`를 설정할 수 있습니다:

**Java**

```java
@ModelAttribute
public AccountForm setUpForm() {
    return new AccountForm();
}

@ModelAttribute
public Account findAccount(@PathVariable String accountId) {
    return accountRepository.findOne(accountId);
}

@PostMapping("update")
public String update(AccountForm form, BindingResult result,
        @ModelAttribute(binding=false) Account account) { // ①
    // ...
}
```

① `@ModelAttribute(binding=false)` 설정

**Kotlin**

```kotlin
@ModelAttribute
fun setUpForm(): AccountForm {
    return AccountForm()
}

@ModelAttribute
fun findAccount(@PathVariable accountId: String): Account {
    return accountRepository.findOne(accountId)
}

@PostMapping("update")
fun update(form: AccountForm, result: BindingResult,
        @ModelAttribute(binding = false) account: Account): String { // ①
    // ...
    return "someView"
}
```

① `@ModelAttribute(binding=false)` 설정

데이터 바인딩 결과 오류가 발생하면 기본적으로 `MethodArgumentNotValidException`이 발생하지만, 컨트롤러 메서드에서 이러한 오류를 처리하기 위해 `@ModelAttribute` 바로 옆에 `BindingResult` 인자를 추가할 수도 있습니다.

예를 들어:

**Java**

```java
@PostMapping("/owners/{ownerId}/pets/{petId}/edit")
public String processSubmit(@ModelAttribute("pet") Pet pet, BindingResult result) { // ①
    if (result.hasErrors()) {
        return "petForm";
    }
    // ...
}
```

① `@ModelAttribute` 옆에 `BindingResult` 추가

**Kotlin**

```kotlin
@PostMapping("/owners/{ownerId}/pets/{petId}/edit")
fun processSubmit(@ModelAttribute("pet") pet: Pet, result: BindingResult): String { // ①
    if (result.hasErrors()) {
        return "petForm"
    }
    // ...
    return "someView"
}
```

① `@ModelAttribute` 옆에 `BindingResult` 추가

`jakarta.validation.Valid` 애노테이션이나 스프링의 `@Validated` 애노테이션을 추가하여 데이터 바인딩 후 자동으로 유효성 검사를 적용할 수 있습니다.

[Bean 유효성 검사](https://docs.spring.io/spring-framework/docs/current/reference/html/core.html#validation) 및 [스프링 유효성 검사](https://docs.spring.io/spring-framework/docs/current/reference/html/web.html#mvc-config-validation)를 참조하십시오.

예를 들어:

**Java**

```java
@PostMapping("/owners/{ownerId}/pets/{petId}/edit")
public String processSubmit(@Valid @ModelAttribute("pet") Pet pet, BindingResult result) { // ①
    if (result.hasErrors()) {
        return "petForm";
    }
    // ...
}
```

① `Pet` 인스턴스를 유효성 검사합니다.

**Kotlin**

```kotlin
@PostMapping("/owners/{ownerId}/pets/{petId}/edit")
fun processSubmit(@Valid @ModelAttribute("pet") pet: Pet, result: BindingResult): String { // ①
    if (result.hasErrors()) {
        return "petForm"
    }
    // ...
    return "someView"
}
```

① `Pet` 인스턴스를 유효성 검사합니다.

`@ModelAttribute` 뒤에 `BindingResult` 파라미터가 없으면 유효성 검사 오류와 함께 `MethodArgumentNotValidException`이 발생합니다.

그러나 다른 파라미터에 `@jakarta.validation.Constraint` 애노테이션이 있어 메서드 유효성 검사가 적용되는 경우, 대신 `HandlerMethodValidationException`이 발생합니다.

자세한 내용은 [유효성 검사 섹션](https://docs.spring.io/spring-framework/docs/current/reference/html/web.html#mvc-ann-validation)을 참조하십시오.

`@ModelAttribute` 사용은 선택 사항입니다.

기본적으로, ([BeanUtils#isSimpleProperty](https://docs.spring.io/spring-framework/docs/current/javadoc-api/org/springframework/beans/BeanUtils.html#isSimpleProperty-java.lang.Class-)에 의해 결정된) 단순 값 타입이 아니고 다른 인자 해석기(argument resolver)에 의해 해석되지 않는 모든 파라미터는 암시적인 `@ModelAttribute`로 처리됩니다.

GraalVM으로 네이티브 이미지로 컴파일할 때, 위에서 설명한 암시적 `@ModelAttribute` 지원은 관련된 데이터 바인딩 리플렉션 힌트의 적절한 사전 추론을 허용하지 않습니다.

결과적으로 GraalVM 네이티브 이미지에서 사용하기 위해 메서드 파라미터에 `@ModelAttribute`를 명시적으로 애노테이트하는 것이 좋습니다.

---

**✨ 이 챕터의 학습 목표 ✨**

1. **`@ModelAttribute` 애노테이션이 컨트롤러 메서드의 파라미터에 사용될 때, 어떻게 요청 데이터(폼 데이터, 쿼리 파라미터 등)를 객체에 자동으로 바인딩하는지 이해합니다.** (핵심 기능!)
2. **`@ModelAttribute`가 파라미터가 아닌 메서드에 사용될 때, 해당 메서드가 반환하는 값을 모델에 자동으로 추가하는 기능을 이해하고, 이것이 뷰에 데이터를 전달하는 데 어떻게 활용되는지 배웁니다.**
3. **데이터 바인딩 과정에서 발생할 수 있는 오류를 `BindingResult`를 사용하여 처리하고, `@Valid` 또는 `@Validated` 애노테이션을 통해 유효성 검사를 수행하는 방법을 익힙니다.**
4. **`@ModelAttribute`의 `binding=false` 속성을 사용하여 데이터 바인딩 없이 모델 객체를 가져오는 경우를 이해하고, `@SessionAttributes`와의 연관 가능성을 인지합니다.** (이 챕터에서 `@SessionAttributes`를 깊게 다루진 않지만, 연관성을 언급합니다.)

---

### `@ModelAttribute`의 두 가지 주요 사용법

`@ModelAttribute`는 크게 두 가지 상황에서 사용됩니다.

1. **메서드 파라미터에 사용될 때:** 요청 데이터를 특정 객체에 자동으로 바인딩(채워 넣기)하는 역할을 합니다. (가장 흔한 사용법)
2. **메서드 자체에 사용될 때 (파라미터가 아님):** 해당 메서드가 반환하는 객체를 자동으로 모델(Model)에 추가하여 뷰(View)에서 사용할 수 있도록 합니다.

이번 챕터의 원문은 주로 첫 번째 경우(메서드 파라미터)에 초점을 맞추고 있지만, 두 번째 경우도 매우 중요하므로 함께 설명드리겠습니다.

---

### 1. 메서드 파라미터에 사용될 때: 요청 데이터 바인딩

이것이 `@ModelAttribute`의 가장 핵심적인 기능입니다.

사용자가 웹 브라우저를 통해 폼(form)을 제출하거나, URL에 쿼리 파라미터를 여러 개 붙여서 보낼 때, 그 값들을 우리가 만든 자바 객체(주로 DTO 또는 Command 객체라고 불립니다)의 필드에 알아서 착착 채워주는 마법 같은 역할을 합니다.

**핵심 개념 설명:**

- **데이터 바인딩(Data Binding):** 요청으로 들어온 문자열 값들을 객체의 특정 타입 필드( `String`, `int`, `boolean`, 심지어 중첩된 객체까지)에 맞게 변환하여 할당해주는 과정입니다.
- **어떤 요청 데이터를 바인딩하는가?**
  - **쿼리 파라미터:** URL 뒤에 `?name=john&age=30` 처럼 붙는 값들
  - **폼 데이터 (POST 요청 시):** HTML `<form>` 태그를 통해 `application/x-www-form-urlencoded` 방식으로 전송되는 데이터
  - **URI 경로 변수 (`@PathVariable`):** 경로 변수 이름과 객체의 필드 이름이 같으면 바인딩될 수 있습니다.
  - **요청 헤더:** 헤더 이름(대시 제거)과 객체 필드 이름이 같으면 바인딩될 수 있습니다. (단, 요청 파라미터와 이름이 겹치면 요청 파라미터가 우선)

**비유를 들어볼까요?**

- 여러분이 온라인 쇼핑몰에서 회원가입을 한다고 생각해보세요. 아이디, 비밀번호, 이름, 주소 등을 입력하고 '가입' 버튼을 누릅니다.
- 이때 여러분이 입력한 각 정보들(아이디, 비밀번호 등)이 **요청 파라미터**입니다.
- 서버에서는 이 정보들을 받아서 `User`라는 회원 정보 객체에 담아야 합니다.
  - `user.setId(요청된 아이디);`
  - `user.setPassword(요청된 비밀번호);`
  - `user.setName(요청된 이름);`
  - ...
- 이런 코드를 일일이 작성하는 대신, `@ModelAttribute User user` 라고만 컨트롤러 메서드 파라미터에 선언해주면, 스프링이 알아서 요청 파라미터 이름과 `User` 객체의 필드 이름을 비교해서 값을 채워줍니다. 마치 **자동으로 양식을 채워주는 로봇**과 같습니다.

**어떻게 `Pet` 인스턴스를 얻어오는가? (원문 내용)**

원문에서는 `@ModelAttribute Pet pet`과 같이 사용했을 때, 이 `pet` 객체가 어디서 왔는지 몇 가지 경우를 설명합니다.

1. **`@ModelAttribute` 메서드에서 이미 모델에 추가된 경우:** (이것은 `@ModelAttribute`의 두 번째 사용법과 관련) 만약 다른 메서드에 `@ModelAttribute`가 붙어 있고, 그 메서드가 "pet"이라는 이름으로 `Pet` 객체를 모델에 미리 추가해두었다면, 그 객체를 가져와서 요청 파라미터로 덮어쓸 수 있습니다.

    ```java
    @ModelAttribute("pet") // 모델에 "pet"이라는 이름으로 Pet 객체 추가
    public Pet setUpPet() {
        return new Pet(); // 또는 DB에서 기본 Pet 정보 로드
    }
    
    @PostMapping("/pets/edit")
    public String processSubmit(@ModelAttribute("pet") Pet petFromModel) {
        // petFromModel은 setUpPet()이 반환한 Pet 객체에 요청 파라미터가 덮어씌워진 상태
        // ...
    }
    ```

2. **HTTP 세션에서 가져오는 경우 (`@SessionAttributes`):** 만약 컨트롤러 클래스 레벨에 `@SessionAttributes("pet")`과 같이 선언되어 있고, 이전에 "pet"이라는 이름의 객체가 세션에 저장된 적이 있다면, 세션에서 가져온 객체에 요청 파라미터를 덮어씁니다. (주로 여러 페이지에 걸쳐 폼을 작성할 때 사용)
3. **`Converter`를 통해 얻어오는 경우:** (원문 예제 `save(@ModelAttribute("account") Account account)` 부분)

더 정확히 원문의 예제에 맞추려면:
- 만약 `@ModelAttribute("account")`의 이름("account")이 `@PathVariable` 이름 (`/accounts/{account}`)과 같고, `String`을 `Account` 타입으로 변환하는 `Converter`가 등록되어 있다면, 해당 `Converter`를 통해 `Account` 객체를 먼저 조회한 후, 그 객체에 나머지 요청 파라미터를 바인딩합니다.
- 주로 기존 데이터를 수정하는 폼을 보여주거나 처리할 때, ID를 기반으로 DB에서 엔티티를 먼저 로드하고, 그 엔티티에 사용자가 수정한 값을 덮어쓰는 시나리오에서 유용합니다.

    ```java
    // 요청: PUT /accounts/123?name=newName
    @PutMapping("/accounts/{accountId}") // accountId는 "123"
    public String save(@ModelAttribute("accountId") Account accountFromDB, // Converter<String, Account>가 "123"으로 DB에서 Account 로드
                       @RequestParam String name) { // "name" 요청 파라미터는 accountFromDB 객체의 name 필드에 바인딩 시도
        // accountFromDB는 ID "123"으로 DB에서 조회된 Account 객체이고,
        // 만약 Account 객체에 setName(String name) 메서드가 있다면,
        // 이 요청에서는 name="newName"이 accountFromDB.setName("newName") 형태로 바인딩될 수 있음.
        // (원문에서는 @ModelAttribute("account") Account account 형태로 한 번에 처리)
        // ...
    }
    ```
    
    ```java
    // 요청: PUT /accounts/123?email=new@email.com
    // Converter<String, Account>는 "123"을 Account 객체로 변환 (DB에서 조회)
    @PutMapping("/accounts/{account}") // {account} 경로 변수 이름이 "account"
    public String save(@ModelAttribute("account") Account loadedAccount) {
        // loadedAccount는 ID "123"으로 DB에서 조회된 Account 객체.
        // 만약 URL에 ?email=new@email.com 파라미터가 있다면,
        // loadedAccount.setEmail("new@email.com")이 시도됨.
        // ...
    }
    ```

4. **기본 생성자를 통해 인스턴스화:** 위 경우들에 해당하지 않으면, 스프링은 `@ModelAttribute`로 지정된 타입(예: `Pet`)의 기본 생성자(`new Pet()`)를 호출하여 새 객체를 만듭니다. 그리고 그 객체에 요청 파라미터를 바인딩합니다. (가장 일반적인 경우)
5. **"주 생성자(Primary Constructor)"를 통해 인스턴스화:** (주로 Kotlin의 data class나 Java Record와 관련) 만약 클래스에 모든 필드를 인자로 받는 생성자가 있고, 요청 파라미터 이름과 생성자 인자 이름이 일치하면, 해당 생성자를 통해 객체를 만들면서 값을 바로 주입합니다. (자바에서는 컴파일 시 `parameters` 옵션 필요)

**`@ModelAttribute` 이름 생략:**

- `@ModelAttribute Pet pet`: 이 경우 모델에 추가될 때의 이름은 타입명의 첫 글자를 소문자로 바꾼 "pet"이 됩니다.
- `@ModelAttribute("myCustomPet") Pet pet`: 모델에 "myCustomPet"이라는 이름으로 추가됩니다.

**`@RequestParam`과의 차이점:**

- `@RequestParam`: 주로 단일 값(문자열, 숫자, 불리언 등)을 받을 때 사용합니다.
- `@ModelAttribute`: 여러 요청 파라미터를 객체의 필드에 한 번에 바인딩할 때 사용합니다.

---

### 2. 메서드 자체에 사용될 때: 모델 데이터 추가

`@ModelAttribute`가 메서드에 붙으면, 해당 컨트롤러의 어떤 요청 처리 메서드(`@GetMapping`, `@PostMapping` 등)가 호출되든지 간에, **이 `@ModelAttribute`가 붙은 메서드가 먼저 실행**됩니다. 그리고 그 메서드가 반환하는 값은 자동으로 모델(Model) 객체에 추가됩니다.

```java
@Controller
public class MyController {

    @ModelAttribute("globalMessage") // "globalMessage"라는 이름으로 모델에 추가
    public String addGlobalMessageToModel() {
        return "Welcome to our site!"; // 이 문자열이 모델에 추가됨
    }

    @ModelAttribute("categories") // "categories"라는 이름으로 모델에 추가
    public List<String> populateCategories() {
        return Arrays.asList("Java", "Spring", "Database"); // 이 리스트가 모델에 추가됨
    }

    @GetMapping("/home")
    public String homePage(Model model) {
        // 이 메서드가 호출되기 전에, addGlobalMessageToModel()과 populateCategories()가 먼저 실행됨.
        // 따라서 model 객체에는 이미 "globalMessage"와 "categories"가 들어있음.
        // 뷰 (예: home.html)에서는 ${globalMessage}나 ${categories}로 접근 가능.
        return "homeView";
    }

    @GetMapping("/anotherPage")
    public String anotherPage(Model model) {
        // 이 메서드가 호출될 때도 마찬가지로 "globalMessage"와 "categories"가 모델에 있음.
        return "anotherView";
    }
}
```

**왜 필요할까요?**

- **공통 모델 데이터 제공:** 여러 뷰 페이지에서 공통적으로 사용되는 데이터(예: 웹사이트 전체 카테고리 목록, 현재 로그인한 사용자 정보 등)를 매번 컨트롤러 메서드마다 `model.addAttribute(...)` 코드를 반복하지 않고, 한 곳에서 관리하여 자동으로 모든 뷰에 전달하고 싶을 때 매우 유용합니다.
- **폼을 위한 초기 객체 제공:** 폼을 보여주는 GET 요청 시, 폼을 채울 빈 객체나 기본값이 설정된 객체를 미리 모델에 넣어둘 때 사용됩니다.

    ```java
    // 폼을 보여주기 위한 메서드
    @GetMapping("/pets/new")
    public String initCreationForm(Model model) {
        Pet pet = new Pet(); // 빈 Pet 객체 생성
        model.addAttribute("pet", pet); // "pet"이라는 이름으로 모델에 추가
        return "petCreateForm"; // 뷰에서 th:object="${pet}" 등으로 사용
    }
    
    // 위 코드를 @ModelAttribute 메서드로 대체 가능
    @ModelAttribute("pet") // "pet"이라는 이름으로 모델에 Pet 객체 추가
    public Pet setUpPetForm() {
        return new Pet(); // 새 Pet 객체 또는 기본값 설정된 Pet 객체
    }
    
    @GetMapping("/pets/new")
    public String initCreationForm() {
        // setUpPetForm()이 먼저 호출되어 "pet"이 모델에 이미 추가됨
        return "petCreateForm";
    }
    ```


---

### 데이터 바인딩 심화 및 주의사항 (원문 내용)

- **보안 (Model Design, `allowedFields`):**
  - 요청 파라미터는 사용자가 임의로 조작해서 보낼 수 있습니다. 만약 객체에 민감한 필드(예: `isAdmin`, `role`)가 있는데, 사용자가 악의적으로 `isAdmin=true` 같은 파라미터를 보내면, `@ModelAttribute`는 기본적으로 해당 필드가 객체에 존재하면 값을 바인딩하려고 시도합니다. 이를 **대량 할당 취약점(Mass Assignment Vulnerability)** 이라고 합니다.
  - **권장 사항:**
    1. **웹 바인딩 전용 객체(DTO) 사용:** 실제 도메인 객체/엔티티 대신, 웹 요청을 받기 위한 별도의 객체(Data Transfer Object)를 만들고, 꼭 필요한 필드만 정의합니다.
    2. **생성자 바인딩 우선 사용:** Setter를 통한 프로퍼티 바인딩보다 생성자를 통한 바인딩을 사용하면, 객체가 생성될 때 필요한 모든 값이 명시적으로 전달되므로 좀 더 안전할 수 있습니다. (특히 불변 객체로 만들 수 있음)
    3. **`allowedFields` 사용 (프로퍼티 바인딩 시):** `WebDataBinder`의 `setAllowedFields("field1", "field2", ...)`를 사용하여 특정 필드들만 바인딩되도록 허용 목록을 지정할 수 있습니다. (또는 `setDisallowedFields`로 금지 목록 지정)

        ```java
        @InitBinder // 컨트롤러 내에서 데이터 바인더 초기화
        public void initBinder(WebDataBinder binder) {
            binder.setAllowedFields("name", "age", "species"); // Pet 객체의 이 필드들만 바인딩 허용
        }
        ```

- **생성자 바인딩과 `@BindParam`:** (원문 Java 예제)
  - 기본적으로 생성자 인자 이름과 요청 파라미터 이름이 같아야 바인딩됩니다.
  - 만약 요청 파라미터 이름이 `first-name` (대시 포함)인데 생성자 인자 이름이 `firstName` (카멜케이스)이라면, `@BindParam("first-name")` 애노테이션을 생성자 인자에 붙여 매핑할 수 있습니다.

    ```java
    public Account(@BindParam("first-name") String firstName, String lastName) {
        // "first-name" 요청 파라미터는 firstName 인자에,
        // "lastName" 요청 파라미터는 lastName 인자에 바인딩
        this.firstName = firstName;
        this.lastName = lastName;
    }
    ```

- **데이터 바인딩 없이 모델 속성 접근 (`binding=false`):**
  - 때로는 모델에 이미 있는 객체를 가져오되, 현재 요청의 파라미터로 그 객체의 값을 덮어쓰고 싶지 않을 때가 있습니다. (예: DB에서 조회한 원본 데이터를 그대로 사용하고 싶을 때)
  - 이때 `@ModelAttribute(binding=false)`를 사용하면, 해당 객체를 모델(또는 세션, `@ModelAttribute` 메서드 등)에서 찾아오기만 하고, 현재 요청 파라미터로 데이터 바인딩을 수행하지 않습니다.

    ```java
    // 가정: update 메서드 호출 전에 어떤 방식으로든 "account"라는 이름의 Account 객체가 모델에 이미 존재
    // (예: @ModelAttribute 메서드나 @SessionAttributes를 통해)
    @PostMapping("/update")
    public String update(
            AccountForm form, // form 데이터는 AccountForm 객체에 바인딩
            BindingResult result,
            @ModelAttribute(binding=false) Account account // 모델에서 "account" 객체를 가져오지만, 바인딩은 안 함
    ) {
        // form의 내용으로 account 객체를 수동으로 업데이트하거나,
        // account 객체는 참조용으로만 사용
        if (account.isLocked()) {
            // ...
        }
        // ...
    }
    ```


---

### 데이터 바인딩 오류 처리 및 유효성 검사 (`BindingResult`, `@Valid`)

- **`BindingResult`:**
  - `@ModelAttribute`로 객체에 데이터를 바인딩할 때, 타입이 맞지 않거나(예: 숫자 필드에 문자열 입력) 형식이 잘못되면 오류가 발생합니다.
  - 이때 `@ModelAttribute` 파라미터 **바로 뒤에** `BindingResult` 타입의 파라미터를 선언하면, 바인딩 오류 정보가 이 `BindingResult` 객체에 담기고, 컨트롤러 메서드 내에서 오류를 직접 처리할 수 있습니다. (예외가 바로 터지지 않음)
  - `result.hasErrors()`: 오류가 있는지 확인합니다.
  - `result.getFieldErrors()`: 필드별 오류 목록을 가져옵니다.

    ```java
    @PostMapping("/pets/edit")
    public String processSubmit(@ModelAttribute("pet") Pet pet, BindingResult result, Model model) {
        if (result.hasErrors()) {
            // 바인딩 오류 발생 시 (예: age에 문자 입력)
            // 오류 메시지와 함께 다시 폼 페이지로 이동
            model.addAttribute("errors", result.getAllErrors());
            return "petForm"; // 다시 입력 폼을 보여줌
        }
        // 오류 없으면 정상 처리
        // ...
        return "redirect:/pets/" + pet.getId();
    }
    ```

  **중요:** `BindingResult`는 반드시 `@ModelAttribute` 파라미터 *바로 다음에* 와야 합니다!

- **`@Valid` 또는 `@Validated` (유효성 검사):**
  - 데이터 바인딩이 성공적으로 끝난 후에, 객체의 값들이 비즈니스 규칙에 맞는지(예: 이름은 비어있으면 안됨, 나이는 0 이상이어야 함 등) 검증하고 싶을 때 사용합니다.
  - 검증할 객체 파라미터 앞에 `@Valid` (Jakarta Bean Validation 표준) 또는 `@Validated` (스프링 제공) 애노테이션을 붙입니다.
  - 해당 객체의 클래스 필드에는 `@NotNull`, `@Size(min=2, max=10)`, `@Min(0)` 등과 같은 유효성 검사 애노테이션이 미리 정의되어 있어야 합니다.

    ```java
    // Pet.java
    public class Pet {
        @NotEmpty // 이름은 비어있을 수 없음
        private String name;
    
        @Min(0) // 나이는 0 이상이어야 함
        private int age;
        // ... getters and setters ...
    }
    
    // Controller.java
    @PostMapping("/pets/edit")
    public String processSubmit(@Valid @ModelAttribute("pet") Pet pet, BindingResult result) {
        if (result.hasErrors()) {
            // 바인딩 오류 또는 유효성 검사 오류 발생 시
            return "petForm";
        }
        // ...
    }
    ```

  - 유효성 검사 오류도 `BindingResult`에 담깁니다.
  - `BindingResult`가 없으면, 데이터 바인딩 오류 시 `MethodArgumentTypeMismatchException` (또는 유사 예외), 유효성 검사 오류 시 `MethodArgumentNotValidException` (또는 `HandlerMethodValidationException` - 원문 참고) 예외가 발생하여 프로그램이 중단될 수 있습니다.

---

### `@ModelAttribute` 생략 (암시적 사용)

원문 마지막 부분에 언급된 내용입니다.

- 컨트롤러 메서드의 파라미터가 **단순 타입이 아니고** (즉, `int`, `String` 같은 기본/문자열 타입이 아니라 우리가 만든 `Pet`, `User` 같은 복합 객체 타입이고)
- 다른 특별한 애노테이션( `@PathVariable`, `@RequestParam`, `@RequestBody` 등)이 붙어있지 않다면,
- 스프링 MVC는 이 파라미터를 **마치 `@ModelAttribute`가 붙은 것처럼 자동으로 처리**합니다.

    ```java
    // 아래 두 코드는 동일하게 동작 (Pet이 단순 타입이 아니라면)
    public String processSubmit(@ModelAttribute Pet pet, BindingResult result) { /* ... */ }
    public String processSubmit(Pet pet, BindingResult result) { /* ... */ } // @ModelAttribute 생략 가능
    
    ```

- **주의 (GraalVM):** 네이티브 이미지로 컴파일할 때는 이 암시적 기능을 사용하면 리플렉션 정보를 제대로 추론하기 어려울 수 있으므로, 명시적으로 `@ModelAttribute`를 붙이는 것이 권장됩니다.

---

### 이전 학습 내용과의 연관성

- **타입 변환(Type Conversion):** `@ModelAttribute`를 통해 객체에 값을 바인딩할 때, 요청 파라미터(문자열)를 객체 필드의 실제 타입( `int`, `Date` 등)으로 변환하는 과정이 내부적으로 일어납니다. 여기서 이전 챕터에서 배운 타입 변환기가 사용됩니다.
- **`@RequestParam`:** `@RequestParam`은 단일 파라미터를 받는 데 중점을 두는 반면, `@ModelAttribute`는 여러 파라미터를 객체 단위로 묶어서 받는 데 사용됩니다. 하지만 둘 다 요청 파라미터를 처리한다는 공통점이 있습니다.
- **컨트롤러와 요청 매핑 (`@Controller`, `@GetMapping`, `@PostMapping`):** `@ModelAttribute`는 이러한 컨트롤러의 요청 처리 메서드 내에서 파라미터로 사용되거나, 컨트롤러 내의 다른 메서드에 붙어 모델 데이터를 준비하는 역할을 합니다.

---

### `Converter`란 무엇인가?

- `Converter<S, T>`는 소스 타입 `S`를 타겟 타입 `T`로 변환하는 역할을 하는 스프링의 인터페이스입니다.
- 예를 들어, `Converter<String, Integer>`는 문자열을 정수로 변환하고, `Converter<String, LocalDate>`는 특정 형식의 문자열을 `LocalDate` 객체로 변환할 수 있습니다.
- 스프링은 이러한 `Converter`들을 `ConversionService`에 등록해두고, 필요할 때 적절한 `Converter`를 찾아 사용합니다.

### `@ModelAttribute`와 `Converter`의 만남: ID로 객체 로드 시나리오

원문에서 설명하는 시나리오는 다음과 같습니다:

1. **URL에 객체의 ID가 포함되어 있습니다.** (예: `/accounts/{accountId}`에서 `accountId`가 ID)
2. 컨트롤러 메서드 파라미터에 `@ModelAttribute("someName") Account account`와 같이 선언되어 있습니다.
3. 이때 `@ModelAttribute`의 이름("someName")이 URL의 경로 변수 이름(`accountId`)과 **일치**하고,
4. `String` 타입(경로 변수는 문자열로 들어옴)을 `Account` 타입으로 변환하는 `Converter<String, Account>`가 스프링에 등록되어 있다면,
5. 스프링은 해당 `Converter`를 사용하여 경로 변수로 들어온 ID 문자열을 가지고 `Account` 객체를 **먼저 조회(또는 생성)**합니다.
6. 그렇게 얻어진 `Account` 객체에 나머지 요청 파라미터(쿼리 파라미터나 폼 데이터)를 추가로 바인딩합니다.

이것은 주로 **기존 데이터를 수정하는 폼을 처리할 때** 매우 유용합니다.

- **수정 폼 보여주기 (GET 요청):** ID를 받아서 DB에서 해당 데이터를 조회한 후, 그 데이터를 폼에 채워서 보여줍니다.
- **수정 내용 저장 (PUT 또는 POST 요청):** ID를 받아서 DB에서 원본 데이터를 조회하고, 사용자가 폼에서 수정한 내용으로 원본 데이터를 업데이트한 후 저장합니다.

### 실제 코드 예시

간단한 `Account` 엔티티와 이를 처리하는 `Converter`, 그리고 컨트롤러를 만들어보겠습니다. (JPA나 특정 ORM을 사용하지 않고, 간단한 Map 기반의 "가짜" 리포지토리를 사용하겠습니다.)

**1. `Account` 클래스 (모델 객체)**

```java
package com.example.demo.model;

public class Account {
    private Long id;
    private String username;
    private String email;
    private boolean active;

    // 기본 생성자 (스프링이 객체 생성 시 사용 가능)
    public Account() {
    }

    public Account(Long id, String username, String email, boolean active) {
        this.id = id;
        this.username = username;
        this.email = email;
        this.active = active;
    }

    // Getters and Setters
    public Long getId() { return id; }
    public void setId(Long id) { this.id = id; }
    public String getUsername() { return username; }
    public void setUsername(String username) { this.username = username; }
    public String getEmail() { return email; }
    public void setEmail(String email) { this.email = email; }
    public boolean isActive() { return active; }
    public void setActive(boolean active) { this.active = active; }

    @Override
    public String toString() {
        return "Account{" +
               "id=" + id +
               ", username='" + username + '\\'' +
               ", email='" + email + '\\'' +
               ", active=" + active +
               '}';
    }
}
```

**2. `AccountRepository` (가짜 데이터 저장소)**

```java
package com.example.demo.repository;

import com.example.demo.model.Account;
import org.springframework.stereotype.Repository;

import java.util.HashMap;
import java.util.Map;
import java.util.Optional;

@Repository // 스프링 빈으로 등록
public class AccountRepository {
    private final Map<Long, Account> store = new HashMap<>();
    private long sequence = 0L;

    public AccountRepository() {
        // 초기 데이터
        save(new Account(null, "user1", "user1@example.com", true));
        save(new Account(null, "user2", "user2@example.com", false));
    }

    public Account save(Account account) {
        if (account.getId() == null) {
            account.setId(++sequence);
        }
        store.put(account.getId(), account);
        return account;
    }

    public Optional<Account> findById(Long id) {
        return Optional.ofNullable(store.get(id));
    }
}
```

**3. `StringToAccountConverter` (핵심!)**

이것이 바로 `String` 타입의 ID를 받아서 `Account` 객체를 DB(여기서는 `AccountRepository`)에서 조회해오는 `Converter`입니다.

```java
package com.example.demo.converter;

import com.example.demo.model.Account;
import com.example.demo.repository.AccountRepository;
import org.springframework.beans.factory.annotation.Autowired;
import org.springframework.core.convert.converter.Converter;
import org.springframework.stereotype.Component;

@Component // 스프링 빈으로 등록되어 ConversionService에 자동으로 추가됨
public class StringToAccountConverter implements Converter<String, Account> {

    private final AccountRepository accountRepository;

    @Autowired // AccountRepository 주입
    public StringToAccountConverter(AccountRepository accountRepository) {
        this.accountRepository = accountRepository;
    }

    @Override
    public Account convert(String sourceId) {
        System.out.println("StringToAccountConverter activated for ID: " + sourceId);
        try {
            Long id = Long.parseLong(sourceId);
            // ID로 Account 객체를 찾아서 반환, 없으면 예외 발생 또는 null 반환 (여기선 예외)
            return accountRepository.findById(id)
                    .orElseThrow(() -> new IllegalArgumentException("Account not found with id: " + sourceId));
        } catch (NumberFormatException e) {
            throw new IllegalArgumentException("Invalid account ID format: " + sourceId, e);
        }
    }
}
```

- `@Component`로 선언하면 스프링이 이 `Converter`를 자동으로 인식하고 `ConversionService`에 등록합니다.
- `convert(String sourceId)` 메서드에서 문자열 ID를 `Long`으로 변환하고, `accountRepository`를 사용해 `Account` 객체를 조회합니다.
- 만약 해당 ID의 `Account`가 없으면 예외를 발생시킵니다. (실제로는 `null`을 반환하거나 다른 방식으로 처리할 수도 있습니다.)

**4. `WebConfig` (Converter를 명시적으로 등록하는 방법 - 선택 사항)**

`@Component`를 사용하면 자동 등록되지만, 명시적으로 `WebMvcConfigurer`를 통해 등록할 수도 있습니다.

```java
package com.example.demo.config;

import com.example.demo.converter.StringToAccountConverter; // Converter 임포트
import org.springframework.context.annotation.Configuration;
import org.springframework.format.FormatterRegistry; // FormatterRegistry 임포트
import org.springframework.web.servlet.config.annotation.WebMvcConfigurer;

@Configuration
public class WebConfig implements WebMvcConfigurer {

    private final StringToAccountConverter stringToAccountConverter;

    // 생성자 주입 (만약 StringToAccountConverter가 @Component가 아니라면 여기서 new로 생성)
    public WebConfig(StringToAccountConverter stringToAccountConverter) {
        this.stringToAccountConverter = stringToAccountConverter;
    }

    @Override
    public void addFormatters(FormatterRegistry registry) {
        registry.addConverter(stringToAccountConverter); // Converter 등록
        // 또는 registry.addConverter(new StringToAccountConverter(accountRepository));
        System.out.println("StringToAccountConverter registered manually in WebConfig.");
    }
}
```

- 만약 `StringToAccountConverter`에 `@Component` 애노테이션이 있다면 이 `WebConfig` 설정은 필수는 아닙니다. 스프링 부트 환경에서는 `@Component`로 된 `Converter`를 자동으로 찾아 등록해줍니다. 하지만 일반 스프링 MVC 환경이거나 명시적인 관리를 원한다면 이렇게 등록할 수 있습니다.

**5. `AccountController` (컨트롤러)**

```java
package com.example.demo.controller;

import com.example.demo.model.Account;
import com.example.demo.repository.AccountRepository;
import org.springframework.beans.factory.annotation.Autowired;
import org.springframework.stereotype.Controller;
import org.springframework.ui.Model;
import org.springframework.web.bind.annotation.*;

@Controller
@RequestMapping("/accounts")
public class AccountController {

    private final AccountRepository accountRepository;

    @Autowired
    public AccountController(AccountRepository accountRepository) {
        this.accountRepository = accountRepository;
    }

    // 계정 수정 폼을 보여주는 핸들러
    // 요청 예: GET /accounts/1/edit
    @GetMapping("/{accountId}/edit")
    public String showEditForm(@ModelAttribute("accountId") Account account, Model model) {
        // 1. @PathVariable "accountId" (예: "1")가 StringToAccountConverter에 전달됩니다.
        // 2. StringToAccountConverter는 ID "1"로 AccountRepository에서 Account 객체를 조회합니다.
        // 3. 조회된 Account 객체가 account 파라미터에 바인딩됩니다.
        // 4. 이 account 객체는 자동으로 모델에 "account"라는 이름으로 추가됩니다.
        //    (ModelAttribute의 이름이 "accountId"이므로, 모델에 추가될 때의 이름은 타입명인 "account"가 됨.
        //     만약 @ModelAttribute("loadedAccount") Account account 였다면 "loadedAccount"로 추가됨)
        //    명시적으로 모델에 추가하려면 model.addAttribute("accountForForm", account); 처럼 할 수 있습니다.
        //    하지만 @ModelAttribute는 그 자체로 모델에 추가하는 기능이 있으므로, 여기서는 "account"로 추가됩니다.

        System.out.println("Editing Account (from Converter): " + account);
        // model.addAttribute("account", account); // @ModelAttribute가 이미 모델에 넣어주므로 생략 가능
        return "account-edit-form"; // Thymeleaf 템플릿 이름 (아래에서 간단히 만듦)
    }

    // 계정 수정 내용을 처리하는 핸들러
    // 요청 예: POST /accounts/1/edit?email=updated@example.com&active=false
    @PostMapping("/{accountId}/edit")
    public String processEditForm(@ModelAttribute("accountId") Account account, // ①
                                  @RequestParam(required = false) String email,  // ②
                                  @RequestParam(required = false) Boolean active) { // ②
        // ① @ModelAttribute("accountId") Account account:
        //    - GET 요청과 마찬가지로, "accountId" 경로 변수 값으로 StringToAccountConverter가 동작하여
        //      DB(여기서는 Repository)에서 원본 Account 객체를 로드합니다.
        // ② @RequestParam ... email, ... active:
        //    - 그 다음, URL의 쿼리 파라미터 (또는 폼 데이터) "email"과 "active" 값이
        //      로드된 account 객체의 해당 필드(setEmail, setActive)에 바인딩(덮어쓰기) 됩니다.
        //    - 만약 Account 객체에 username 필드도 있고, 요청에 username 파라미터가 있다면 그것도 바인딩됩니다.

        System.out.println("Account loaded by Converter: " + account.getId() + ", original email: " + account.getEmail());

        // 요청 파라미터로 account 객체의 값을 업데이트 (이 부분은 @ModelAttribute가 자동으로 해줌)
        // 만약 명시적으로 하고 싶다면, 또는 @ModelAttribute(binding=false)를 썼다면 아래처럼 수동으로 해야 함
        // if (email != null) account.setEmail(email);
        // if (active != null) account.setActive(active);

        System.out.println("Account after binding request params: " + account);

        accountRepository.save(account); // 변경된 내용 저장
        return "redirect:/accounts/" + account.getId() + "/edit"; // 수정 후 다시 폼으로 리다이렉트
    }
}
```

- **`showEditForm` 메서드:**
  - `@GetMapping("/{accountId}/edit")`: `accountId`라는 경로 변수를 가집니다.
  - `@ModelAttribute("accountId") Account account`:
    - `@ModelAttribute`의 이름 "accountId"가 경로 변수 이름과 같습니다.
    - 스프링은 `StringToAccountConverter`를 찾아서 실행합니다. `accountId` 경로 변수 값(예: "1")이 `Converter`의 `sourceId`로 전달됩니다.
    - `Converter`는 `AccountRepository`를 사용해 ID 1번 `Account` 객체를 조회하여 반환합니다.
    - 이 반환된 `Account` 객체가 `account` 파라미터에 할당됩니다.
    - 또한, 이 `account` 객체는 모델에 "account"라는 이름으로 (타입명의 첫 글자 소문자) 자동으로 추가되어 뷰에서 사용할 수 있게 됩니다.
- **`processEditForm` 메서드:**
  - `@PostMapping("/{accountId}/edit")`: 역시 `accountId` 경로 변수를 가집니다.
  - `@ModelAttribute("accountId") Account account`: `showEditForm`과 동일한 방식으로 `Converter`를 통해 DB에서 원본 `Account` 객체를 먼저 로드합니다.
  - **중요한 부분:** 이렇게 로드된 `account` 객체에, **이어서** 요청으로 들어온 다른 파라미터들(예: `email=updated@example.com&active=false`)이 자동으로 바인딩됩니다. 즉, `account.setEmail("updated@example.com")`과 `account.setActive(false)`가 내부적으로 호출됩니다.
  - 그래서 `Converter`를 사용하면 "DB에서 기존 객체 로드 -> 로드된 객체에 요청 파라미터 덮어쓰기" 과정이 `@ModelAttribute` 하나로 깔끔하게 처리됩니다.

**6. `account-edit-form.html` (간단한 Thymeleaf 템플릿 예시)**

`src/main/resources/templates/account-edit-form.html`

```html
<!DOCTYPE html>
<html xmlns:th="<http://www.thymeleaf.org>">
<head>
    <title>Edit Account</title>
</head>
<body>
    <h1>Edit Account</h1>
    <!-- th:object는 모델에서 "account"라는 이름의 객체를 찾음 -->
    <form th:action="@{/accounts/{id}/edit(id=${account.id})}" th:object="${account}" method="post">
        <input type="hidden" th:field="*{id}" />
        <div>
            <label for="username">Username:</label>
            <!-- username은 수정 불가로 가정하고 disabled 처리 (예시) -->
            <input type="text" id="username" th:field="*{username}" readonly />
        </div>
        <div>
            <label for="email">Email:</label>
            <input type="text" id="email" th:field="*{email}" />
        </div>
        <div>
            <label for="active">Active:</label>
            <input type="checkbox" id="active" th:field="*{active}" />
        </div>
        <div>
            <button type="submit">Save</button>
        </div>
    </form>
    <hr/>
    Current Account Details: <span th:text="${account}"></span>
</body>
</html>
```

### 실행 흐름 요약

1. **사용자가 `GET /accounts/1/edit` 요청:**
  - `AccountController`의 `showEditForm` 메서드 호출.
  - `@ModelAttribute("accountId") Account account` 부분에서:
    - 경로 변수 `accountId` ("1")가 `StringToAccountConverter`로 전달됨.
    - `Converter`는 ID 1번 `Account` 객체를 DB에서 조회하여 반환.
    - 조회된 `Account` 객체가 `account` 파라미터에 할당되고, 모델에 "account"로 추가됨.
  - `account-edit-form.html` 뷰가 렌더링되고, 폼에는 DB에서 조회된 1번 계정의 정보가 채워짐.
2. **사용자가 폼에서 이메일을 "[new@mail.com](mailto:new@mail.com)"으로 수정하고 'Save' 버튼 클릭 (`POST /accounts/1/edit` 요청, 폼 데이터: `email=new@mail.com&active=true` 등):**
  - `AccountController`의 `processEditForm` 메서드 호출.
  - `@ModelAttribute("accountId") Account account` 부분에서:
    - 경로 변수 `accountId` ("1")가 `StringToAccountConverter`로 전달됨.
    - `Converter`는 ID 1번 `Account` 객체를 DB에서 **다시 조회**하여 반환. (이것이 원본 객체)
    - 조회된 원본 `Account` 객체가 `account` 파라미터에 할당됨.
  - **이어서**, HTTP 요청의 파라미터들(`email=new@mail.com`, `active=true` 등)이 이 `account` 객체의 해당 필드에 **바인딩(덮어쓰기)** 됨.
    - `account.setEmail("new@mail.com")` 호출
    - `account.setActive(true)` 호출
  - `accountRepository.save(account)`로 변경된 `account` 객체 저장.
  - `redirect:/accounts/1/edit`로 리다이렉트.

이처럼 `Converter`와 `@ModelAttribute`를 함께 사용하면, ID를 통해 DB에서 엔티티를 로드하고, 그 엔티티에 요청 파라미터를 바인딩하는 과정을 매우 간결하게 처리할 수 있습니다. 특히 **"읽기-수정-쓰기(read-modify-write)"** 패턴에서 빛을 발합니다.
