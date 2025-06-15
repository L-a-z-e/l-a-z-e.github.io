---
title: Spring Web MVC - Annotated Controllers (Jackson Json)
description: 
author: laze
date: 2025-06-15 00:00:01 +0900
categories: [Dev, SpringWebMVC]
tags: [SpringWebMVC]
---
### Jackson JSON

스프링은 Jackson JSON 라이브러리에 대한 지원을 제공합니다.

### JSON Views (JSON 뷰)

스프링 MVC는 Jackson의 직렬화 뷰(Serialization Views)에 대한 내장 지원을 제공하며, 이를 통해 객체의 모든 필드 중 일부만 렌더링할 수 있습니다.

`@ResponseBody` 또는 `ResponseEntity` 컨트롤러 메소드와 함께 사용하려면, 다음 예제와 같이 Jackson의 `@JsonView` 어노테이션을 사용하여 직렬화 뷰 클래스를 활성화할 수 있습니다:

```java
@RestController
public class UserController {

	@GetMapping("/user")
	@JsonView(User.WithoutPasswordView.class) // User.WithoutPasswordView 활성화
	public User getUser() {
		return new User("eric", "7!jd#h23");
	}
}

public class User {

	// 뷰 인터페이스 정의
	public interface WithoutPasswordView {};
	public interface WithPasswordView extends WithoutPasswordView {}; // WithoutPasswordView 상속

	private String username;
	private String password;

	public User() {
	}

	public User(String username, String password) {
		this.username = username;
		this.password = password;
	}

	@JsonView(WithoutPasswordView.class) // 이 필드는 WithoutPasswordView 또는 이를 상속한 뷰에 포함됨
	public String getUsername() {
		return this.username;
	}

	@JsonView(WithPasswordView.class) // 이 필드는 WithPasswordView에만 포함됨
	public String getPassword() {
		return this.password;
	}
}
```

`@JsonView`는 뷰 클래스의 배열을 허용하지만, 컨트롤러 메소드당 하나만 지정할 수 있습니다. 여러 뷰를 활성화해야 하는 경우, 복합 인터페이스(composite interface)를 사용할 수 있습니다.

위 작업을 `@JsonView` 어노테이션을 선언하는 대신 프로그래밍 방식으로 수행하려면, 반환 값을 `MappingJacksonValue`로 감싸고 이를 사용하여 직렬화 뷰를 제공합니다:

**Java**

```java
@RestController
public class UserController {

	@GetMapping("/user")
	public MappingJacksonValue getUser() {
		User user = new User("eric", "7!jd#h23");
		MappingJacksonValue value = new MappingJacksonValue(user); // User 객체를 MappingJacksonValue로 감쌈
		value.setSerializationView(User.WithoutPasswordView.class); // 직렬화 뷰 설정
		return value;
	}
}
```

뷰 리졸루션(view resolution)에 의존하는 컨트롤러의 경우, 다음 예제와 같이 직렬화 뷰 클래스를 모델에 추가할 수 있습니다:

**Java**

```java
@Controller
public class UserController extends AbstractController { // AbstractController는 예시일 뿐, 일반 @Controller에서도 가능

	@GetMapping("/user")
	public String getUser(Model model) {
		model.addAttribute("user", new User("eric", "7!jd#h23"));
		model.addAttribute(JsonView.class.getName(), User.WithoutPasswordView.class); // 모델에 뷰 클래스 추가
		return "userView"; // 이 뷰는 JSON 렌더링을 지원해야 함 (예: MappingJackson2JsonView)
	}
}
```

---

### **학습 목표 제시**

이 챕터를 학습하고 나면, 학생은 다음을 이해하고 수행할 수 있게 됩니다:

1. **Jackson JSON 뷰의 개념과 필요성 이해:** 동일한 자바 객체에 대해 상황에 따라 다른 필드들을 선택적으로 JSON 응답에 포함시켜야 하는 이유와, Jackson JSON 뷰가 이를 어떻게 해결하는지 이해합니다.
2. **`@JsonView` 어노테이션을 사용한 직렬화 뷰 적용 방법 숙지:** 뷰 인터페이스를 정의하고, 자바 객체의 필드(또는 getter 메소드)에 `@JsonView`를 적용하며, 컨트롤러 메소드에서 특정 뷰를 활성화하여 응답을 제어하는 방법을 이해하고 적용할 수 있습니다.
3. **프로그래밍 방식 및 뷰 리졸루션 방식에서의 JSON 뷰 사용법 인지:** `@JsonView` 어노테이션 외에 `MappingJacksonValue`를 사용하거나 모델에 뷰 정보를 추가하여 JSON 뷰를 적용하는 다른 방법들을 인지합니다.

---

### **핵심 개념 설명**

**Jackson JSON Views란 무엇이고, 왜 필요할까요?**

우리가 자바 객체를 JSON으로 변환하여 클라이언트에게 응답할 때, 항상 객체의 모든 필드 정보를 다 보여줘야 하는 것은 아닙니다. 때로는 상황에 따라 특정 필드는 숨기고, 특정 필드만 보여주고 싶을 수 있습니다.

**예시:**

- **사용자 정보 API:**
  - 일반 사용자가 자신의 프로필을 조회할 때: 사용자 이름, 이메일, 가입일 등 대부분의 정보를 보여줍니다.
  - 관리자가 사용자 목록을 조회할 때: 각 사용자의 이름, 이메일, 역할(role) 정도만 간략히 보여주고, 비밀번호 같은 민감 정보는 제외합니다.
  - 외부 공개 API에서 사용자 정보를 제공할 때: 사용자 이름과 공개 프로필 URL 정도만 노출하고, 이메일이나 개인 정보는 숨깁니다.

이처럼 **동일한 `User` 객체라도, 어떤 API 요청이냐, 또는 어떤 사용자가 요청하느냐에 따라 JSON으로 변환될 때 포함되는 필드가 달라져야 하는 경우**가 발생합니다.

이런 문제를 해결하기 위해 매번 다른 DTO(Data Transfer Object) 클래스를 여러 개 만드는 것은 번거롭고 유지보수가 어려울 수 있습니다. `UserSummaryDto`, `UserDetailDto`, `UserAdminViewDto` 등등...

**Jackson JSON Views**는 이러한 DTO를 여러 개 만들 필요 없이, **하나의 자바 객체(`User`)를 사용하면서도, 직렬화(객체 -> JSON 변환) 시점에 어떤 필드를 포함할지를 동적으로 제어할 수 있게 해주는 기능**입니다.

**동작 원리:**

1. **뷰 인터페이스 정의:** 먼저, 어떤 필드들을 묶어서 보여줄지를 나타내는 "뷰(View)"를 인터페이스로 정의합니다. 이 인터페이스는 마커(marker) 인터페이스 역할을 하며, 특별한 메소드를 가질 필요는 없습니다. 뷰 간의 상속도 가능합니다.
2. **필드에 뷰 지정 (`@JsonView` on fields/getters):** 자바 객체의 각 필드(또는 getter 메소드)에 `@JsonView` 어노테이션을 사용하여, 해당 필드가 어떤 뷰(들)에 속하는지를 지정합니다.
3. **컨트롤러에서 뷰 활성화 (`@JsonView` on method):** `@ResponseBody`나 `ResponseEntity`를 사용하는 컨트롤러 메소드에 `@JsonView` 어노테이션을 사용하여, 이번 응답에서는 어떤 뷰를 기준으로 JSON을 생성할지를 지정합니다.
4. **선택적 직렬화:** Jackson은 컨트롤러 메소드에 지정된 뷰를 기준으로, 객체의 필드들 중에서 해당 뷰에 속하는 필드들만 선택하여 JSON으로 직렬화합니다.

**비유:**

여러분이 사진 앨범(자바 객체)을 가지고 있다고 상상해봅시다. 앨범에는 다양한 사진(필드)들이 들어있습니다.

- **뷰 인터페이스:** "가족 공개용 필터", "친구 공개용 필터", "전체 공개용 필터" 와 같은 "필터(뷰)"를 정의하는 것과 같습니다.
- **필드에 뷰 지정:** 각 사진마다 "이 사진은 가족 공개용 필터에 해당", "이 사진은 친구 공개용 필터와 전체 공개용 필터 모두에 해당" 과 같이 태그(뷰 지정)를 붙이는 것입니다.
- **컨트롤러에서 뷰 활성화:** 누군가에게 앨범을 보여줄 때, "이번에는 '가족 공개용 필터'를 적용해서 보여줄게!" 라고 선언하는 것입니다.
- **선택적 직렬화:** 선언된 필터('가족 공개용 필터')에 해당하는 태그가 붙은 사진들만 골라서 보여주는(JSON으로 변환) 과정입니다.

### **주요 용어 해설**

- **직렬화 뷰 (Serialization View):** Jackson 라이브러리에서 객체를 JSON으로 직렬화할 때, 특정 조건(뷰)에 따라 포함될 필드를 선택적으로 제어하는 메커니즘.
- **`@JsonView`:** Jackson 어노테이션.
  - 자바 객체의 필드나 getter 메소드에 사용하여 해당 요소가 어떤 뷰(들)에 속하는지 지정합니다.
  - 스프링 MVC 컨트롤러 메소드에 사용하여 해당 요청에 대한 JSON 응답 생성 시 어떤 뷰를 활성화할지 지정합니다.
- **마커 인터페이스 (Marker Interface):** 특별한 메소드 없이, 단순히 타입을 표시하거나 그룹화하는 목적으로 사용되는 빈 인터페이스. JSON 뷰 정의 시 주로 사용됩니다.
- **`MappingJacksonValue`:** 프로그래밍 방식으로 직렬화 뷰를 설정하거나 다른 Jackson 설정을 적용해야 할 때, 반환될 객체를 감싸는 래퍼(wrapper) 클래스.
- **뷰 리졸루션 (View Resolution):** 스프링 MVC에서 컨트롤러가 반환한 뷰 이름(또는 뷰 객체)을 실제 뷰 구현체로 해석하고 렌더링하는 과정.

### **코드 예제 및 분석**

원문의 Java 코드를 다시 한번 자세히 살펴보겠습니다.

**1. 뷰 인터페이스 정의 및 객체 필드에 `@JsonView` 적용**

```java
public class User {

	// 1. 뷰 인터페이스 정의
	public interface WithoutPasswordView {}; // 비밀번호를 제외한 뷰
	public interface WithPasswordView extends WithoutPasswordView {}; // 비밀번호를 포함하는 뷰 (WithoutPasswordView의 모든 필드도 포함)

	private String username;
	private String password;

	public User() {
	}

	public User(String username, String password) {
		this.username = username;
		this.password = password;
	}

	// 2. 각 getter 메소드에 @JsonView 적용
	@JsonView(WithoutPasswordView.class) // username은 WithoutPasswordView에 속함
	public String getUsername() {
		return this.username;
	}

	@JsonView(WithPasswordView.class) // password는 WithPasswordView에만 속함
	public String getPassword() {
		return this.password;
	}
}

```

**코드 분석:**

1. **뷰 인터페이스 정의:**
  - `WithoutPasswordView`: 사용자 이름(`username`)만 포함시키고 비밀번호(`password`)는 제외하려는 의도를 가진 뷰입니다.
  - `WithPasswordView`: `WithoutPasswordView`를 상속받습니다. 이는 `WithPasswordView`가 활성화되면 `WithoutPasswordView`에 속한 모든 필드(여기서는 `username`)와 더불어 `WithPasswordView`에만 명시적으로 속한 필드(여기서는 `password`)까지 모두 포함된다는 의미입니다.
2. **getter 메소드에 `@JsonView` 적용:**
  - `getUsername()` 메소드에는 `@JsonView(WithoutPasswordView.class)`가 적용되었습니다. 따라서 `WithoutPasswordView` 또는 이를 상속받는 `WithPasswordView`가 활성화되면 `username` 필드가 JSON에 포함됩니다.
  - `getPassword()` 메소드에는 `@JsonView(WithPasswordView.class)`가 적용되었습니다. 따라서 오직 `WithPasswordView`가 활성화될 때만 `password` 필드가 JSON에 포함됩니다.

   > 참고: @JsonView는 필드 자체에 직접 적용할 수도 있고, getter 메소드에 적용할 수도 있습니다. getter에 적용하면 좀 더 세밀한 제어가 가능할 수 있습니다 (예: 특정 조건에 따라 getter에서 다른 값을 반환하고 싶을 때). 일반적으로는 둘 중 일관된 방식을 사용합니다.
>

**2. 컨트롤러 메소드에서 `@JsonView`로 뷰 활성화**

```java
@RestController // @Controller + @ResponseBody
public class UserController {

	@GetMapping("/user/profile") // 사용자 프로필 조회 (비밀번호 제외)
	@JsonView(User.WithoutPasswordView.class) // 1. User.WithoutPasswordView 활성화
	public User getUserProfile() {
		return new User("eric", "7!jd#h23");
	}

	@GetMapping("/user/admin-details") // 관리자용 상세 정보 조회 (비밀번호 포함)
	@JsonView(User.WithPasswordView.class) // 2. User.WithPasswordView 활성화
	public User getUserAdminDetails() {
		return new User("eric", "7!jd#h23");
	}
}
```

**코드 분석:**

1. `getUserProfile()` 메소드:
  - `@JsonView(User.WithoutPasswordView.class)`가 적용되었습니다.
  - 따라서 `User` 객체가 JSON으로 변환될 때, `User.WithoutPasswordView`에 속한 필드만 포함됩니다. 위 `User` 클래스 정의에 따르면 `username` 필드만 해당됩니다.
  - **예상 응답 (JSON):** `{"username": "eric"}` (`password` 필드는 제외됨)
2. `getUserAdminDetails()` 메소드:
  - `@JsonView(User.WithPasswordView.class)`가 적용되었습니다.
  - `User.WithPasswordView`는 `User.WithoutPasswordView`를 상속했으므로, `username` 필드와 `WithPasswordView`에 직접 속한 `password` 필드가 모두 포함됩니다.
  - **예상 응답 (JSON):** `{"username": "eric", "password": "7!jd#h23"}`

**3. 프로그래밍 방식으로 뷰 설정 (`MappingJacksonValue` 사용)**

때로는 컨트롤러 메소드 내의 로직에 따라 동적으로 어떤 뷰를 적용할지 결정해야 할 수 있습니다. 이때 `@JsonView` 어노테이션 대신 `MappingJacksonValue`를 사용할 수 있습니다.

```java
@RestController
public class UserController {

	@GetMapping("/user/dynamic-view")
	public MappingJacksonValue getUserDynamicView(@RequestParam(required = false) boolean includePassword) {
		User user = new User("eric", "7!jd#h23");
		MappingJacksonValue valueWrapper = new MappingJacksonValue(user); // 1. User 객체를 MappingJacksonValue로 감싼다

		if (includePassword) {
			valueWrapper.setSerializationView(User.WithPasswordView.class); // 2. 조건에 따라 WithPasswordView 설정
		} else {
			valueWrapper.setSerializationView(User.WithoutPasswordView.class); // 2. 조건에 따라 WithoutPasswordView 설정
		}
		return valueWrapper; // 3. MappingJacksonValue 객체 반환
	}
}
```

**코드 분석:**

1. `MappingJacksonValue valueWrapper = new MappingJacksonValue(user);`: 반환할 `User` 객체를 `MappingJacksonValue`로 한번 감쌉니다.
2. `valueWrapper.setSerializationView(...)`: `includePassword` 요청 파라미터 값에 따라 동적으로 사용할 직렬화 뷰를 설정합니다.
3. `return valueWrapper;`: `MappingJacksonValue` 객체를 반환합니다. 스프링 MVC는 이 객체를 보고 설정된 뷰를 적용하여 JSON으로 변환합니다.

**4. 뷰 리졸루션 방식에서의 JSON 뷰 사용**

만약 `@RestController`가 아니라 일반 `@Controller`를 사용하고, 뷰 리졸루션을 통해 JSON 응답을 생성하는 경우 (예: `MappingJackson2JsonView`를 사용하는 경우), 모델에 뷰 정보를 추가할 수 있습니다.

```java
@Controller
public class UserViewController { // 일반 @Controller

	@GetMapping("/user/view-resolved")
	public String getUserViewResolved(Model model) {
		model.addAttribute("user", new User("eric", "7!jd#h23")); // 1. 모델에 User 객체 추가
		// 2. 모델에 사용할 JsonView 클래스 정보 추가
		model.addAttribute(JsonView.class.getName(), User.WithoutPasswordView.class);
		// 또는 더 간단하게: model.addAttribute("jsonView", User.WithoutPasswordView.class); (뷰 설정에 따라 다를 수 있음)

		return "userJsonView"; // 3. 이 뷰는 JSON 렌더링을 지원하고, 모델의 JsonView 정보를 사용하도록 설정되어야 함
	}
}

```

**코드 분석:**

1. `model.addAttribute("user", ...)`: 일반적인 방식처럼 모델에 `User` 객체를 추가합니다.
2. `model.addAttribute(JsonView.class.getName(), User.WithoutPasswordView.class);`: 모델에 특별한 키 (`JsonView.class.getName()` 또는 뷰 설정에 따른 다른 약속된 키)로 사용할 뷰 클래스(`User.WithoutPasswordView.class`)를 추가합니다.
3. `return "userJsonView";`: `userJsonView`라는 이름의 뷰를 반환합니다. 이 뷰는 `MappingJackson2JsonView`와 같이 설정되어 있어야 하며, 모델에 담긴 `JsonView` 정보를 읽어 해당 뷰를 적용하여 `user` 객체를 JSON으로 렌더링합니다. (이 방식은 XML 설정이나 Java Config에서 `MappingJackson2JsonView` 빈을 설정하고 관련 프로퍼티를 구성해야 하므로, `@ResponseBody` 방식보다 설정이 더 필요할 수 있습니다.)

### **"왜?" 라는 질문에 대한 답변**

**Jackson JSON Views는 왜 필요할까요? 그냥 상황에 맞는 DTO 여러 개 만들면 안 되나?**

1. **코드 중복 감소 및 유지보수 용이성:**
  - DTO를 여러 개 만들면, 동일한 필드를 여러 DTO에서 중복으로 선언해야 할 수 있습니다. 만약 필드 하나가 변경되면 관련된 모든 DTO를 수정해야 하는 번거로움이 발생합니다.
  - JSON Views를 사용하면 원본 객체(`User`)는 하나만 유지하고, 뷰 인터페이스와 어노테이션만으로 다양한 표현을 만들어낼 수 있어 중복이 줄고 유지보수가 용이해집니다.
2. **객체 모델의 일관성 유지:** 핵심 도메인 객체는 하나로 유지하면서 표현(presentation) 계층에서만 보여지는 필드를 제어할 수 있습니다. 이는 도메인 모델의 설계를 깔끔하게 유지하는 데 도움이 됩니다.
3. **유연한 응답 제어:** 클라이언트의 권한, 요청의 종류, 또는 특정 조건에 따라 동적으로 다른 데이터 셋을 응답으로 보낼 수 있는 유연성을 제공합니다.
4. **계층형 뷰 정의 가능:** 뷰 인터페이스 간의 상속을 통해 공통적인 뷰를 정의하고, 이를 확장하여 더 구체적인 뷰를 만들 수 있습니다. (예: `PublicView` -> `UserView` -> `AdminView`)

물론, DTO를 사용하는 것이 더 적합한 상황도 있습니다. 예를 들어, 여러 도메인 객체의 정보를 조합하여 전혀 다른 구조의 응답을 만들어야 하거나, 입력(요청)과 출력(응답)의 객체 모델이 크게 다른 경우에는 DTO 패턴이 여전히 유용합니다. JSON Views는 주로 **하나의 객체를 다양한 관점(view)에서 보여주고 싶을 때** 강력한 효과를 발휘합니다.

### **주의사항 및 Best Practice**

1. **뷰 인터페이스 설계의 중요성:** 어떤 필드들을 어떤 상황에서 보여줄 것인지에 대한 명확한 요구사항 분석을 바탕으로 뷰 인터페이스를 잘 설계해야 합니다. 너무 많은 뷰 인터페이스는 오히려 관리를 어렵게 만들 수 있습니다.
2. **상속 관계 활용:** 뷰 인터페이스 간의 상속을 적절히 활용하면 중복을 줄이고 뷰 계층을 효과적으로 관리할 수 있습니다.
3. **`@JsonView` 어노테이션 위치:** 필드 또는 getter 메소드 중 일관된 위치에 적용하는 것이 좋습니다.
4. **컨트롤러 메소드당 하나의 `@JsonView` (기본):** 원문에서 언급했듯이, 컨트롤러 메소드에 `@JsonView`를 사용할 때 기본적으로 하나의 뷰 클래스(또는 그것들을 대표하는 하나의 복합 인터페이스)만 지정할 수 있습니다. 만약 여러 뷰를 조합해야 한다면, 그 조합을 나타내는 새로운 인터페이스를 만들거나 `MappingJacksonValue`를 사용하는 것을 고려해야 합니다.
5. **객체 그래프와 뷰:** 만약 객체가 다른 객체를 참조하고 있고(객체 그래프), 그 참조된 객체에도 JSON 뷰를 적용하고 싶다면, 참조된 객체의 해당 필드에도 `@JsonView`가 적절히 설정되어 있어야 합니다. 뷰는 객체 그래프를 따라 전파될 수 있습니다.
6. **성능 고려:** 매우 복잡한 객체 구조와 많은 뷰를 사용하는 경우, 직렬화 시 약간의 성능 영향이 있을 수 있습니다. 하지만 대부분의 일반적인 경우에는 큰 문제가 되지 않습니다.
7. **문서화:** API를 사용하는 클라이언트가 어떤 뷰가 어떤 필드를 포함하는지 알 수 있도록 API 문서를 잘 작성하는 것이 중요합니다.

### **이전 학습 내용과의 연관성**

- **`@ResponseBody` 및 `ResponseEntity`:** JSON Views는 이 어노테이션들과 함께 사용되어, 반환되는 객체가 JSON으로 직렬화될 때 어떤 필드를 포함할지 제어합니다.
- **`@RestController`:** `@RestController`를 사용한 컨트롤러의 메소드에서도 `@JsonView`를 동일하게 적용할 수 있습니다.
- **`HttpMessageConverter` (Jackson):** 이 모든 마법의 배경에는 Jackson 라이브러리를 사용하는 `HttpMessageConverter` (예: `MappingJackson2HttpMessageConverter`)가 있습니다. 이 컨버터가 `@JsonView` 어노테이션을 인지하고 선택적 직렬화를 수행합니다.
- **DTO (Data Transfer Object) 패턴:** JSON Views는 DTO 패턴의 대안 또는 보완재로 사용될 수 있습니다. 상황에 따라 어떤 방식이 더 적합할지 판단할 수 있어야 합니다.

---
