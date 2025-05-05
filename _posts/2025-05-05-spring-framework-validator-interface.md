---
title: Spring Framework Validator Interface
description: 
author: laze
date: 2025-05-05 00:00:02 +0900
categories: [Dev, SpringBoot]
tags: [SpringBoot]
---
**Validation by Using Spring’s Validator Interface**

스프링은 객체를 검증하는 데 사용할 수 있는 `Validator` 인터페이스를 제공합니다.

`Validator` 인터페이스는 `Errors` 객체를 사용하여 작동하므로, 검증하는 동안 검증기(validators)는 검증 실패를 `Errors` 객체에 보고할 수 있습니다.

```java
// Java
public class Person {

	private String name;
	private int age;

	// 일반적인 getter 및 setter...
}
```

```kotlin
// Kotlin
data class Person(var name: String? = null, var age: Int = 0) // Using data class for brevity
```

다음 예제는 `org.springframework.validation.Validator` 인터페이스의 다음 두 메소드를 구현하여 `Person` 클래스에 대한 검증 동작을 제공합니다:

- `supports(Class)`: 이 `Validator`가 제공된 `Class`의 인스턴스를 검증할 수 있습니까?
- `validate(Object, org.springframework.validation.Errors)`: 주어진 객체를 검증하고, 검증 오류 발생 시 주어진 `Errors` 객체에 해당 오류를 등록합니다.

`Validator` 구현은 매우 간단하며, 특히 스프링 프레임워크가 제공하는 `ValidationUtils` 헬퍼 클래스를 알고 있다면 더욱 그렇습니다.

다음 예제는 `Person` 인스턴스에 대한 `Validator`를 구현합니다:

```java
// Java
public class PersonValidator implements Validator {

	/**
	 * 이 Validator는 Person 인스턴스만 검증합니다
	 */
	@Override
	public boolean supports(Class<?> clazz) {
		return Person.class.equals(clazz);
	}

	@Override
	public void validate(Object obj, Errors e) {
		ValidationUtils.rejectIfEmpty(e, "name", "name.empty");
		Person p = (Person) obj;
		if (p.getAge() < 0) {
			e.rejectValue("age", "negativevalue");
		} else if (p.getAge() > 110) {
			e.rejectValue("age", "too.darn.old");
		}
	}
}
```

```kotlin
// Kotlin
class PersonValidator : Validator {

    /**
     * 이 Validator는 Person 인스턴스만 검증합니다
     */
    override fun supports(clazz: Class<*>): Boolean {
        return Person::class.java == clazz
    }

    override fun validate(obj: Any, e: Errors) {
        ValidationUtils.rejectIfEmpty(e, "name", "name.empty")
        val p = obj as Person
        if (p.age < 0) {
            e.rejectValue("age", "negativevalue")
        } else if (p.age > 110) {
            e.rejectValue("age", "too.darn.old")
        }
    }
}
```

`ValidationUtils` 클래스의 정적 `rejectIfEmpty(..)` 메소드는 `name` 속성이 `null`이거나 빈 문자열인 경우 이를 거부(reject)하는 데 사용됩니다.

풍부한(rich) 객체의 각 중첩 객체를 검증하기 위해 단일 `Validator` 클래스를 구현하는 것이 확실히 가능하지만, 각 중첩 객체 클래스에 대한 검증 로직을 자체 `Validator` 구현에 캡슐화하는 것이 더 좋을 수 있습니다.

"풍부한" 객체의 간단한 예는 두 개의 문자열 속성(이름과 성)과 복잡한 `Address` 객체로 구성된 `Customer`입니다.

`Address` 객체는 `Customer` 객체와 독립적으로 사용될 수 있으므로 별개의 `AddressValidator`가 구현되었습니다.

복사-붙여넣기에 의존하지 않고 `CustomerValidator`가 `AddressValidator` 클래스에 포함된 로직을 재사용하려면, 다음 예제와 같이 `CustomerValidator` 내에 `AddressValidator`를 의존성 주입하거나 인스턴스화할 수 있습니다:

```java
// Java
public class CustomerValidator implements Validator {

	private final Validator addressValidator;

	public CustomerValidator(Validator addressValidator) {
		if (addressValidator == null) {
			throw new IllegalArgumentException("The supplied [Validator] is " +
				"required and must not be null.");
		}
		if (!addressValidator.supports(Address.class)) {
			throw new IllegalArgumentException("The supplied [Validator] must " +
				"support the validation of [Address] instances.");
		}
		this.addressValidator = addressValidator;
	}

	/**
	 * 이 Validator는 Customer 인스턴스 및 Customer의 모든 하위 클래스도 검증합니다
	 */
	@Override
	public boolean supports(Class<?> clazz) {
		return Customer.class.isAssignableFrom(clazz);
	}

	@Override
	public void validate(Object target, Errors errors) {
		ValidationUtils.rejectIfEmptyOrWhitespace(errors, "firstName", "field.required");
		ValidationUtils.rejectIfEmptyOrWhitespace(errors, "surname", "field.required");
		Customer customer = (Customer) target;
		try {
			errors.pushNestedPath("address"); // Navigate into address property
			ValidationUtils.invokeValidator(this.addressValidator, customer.getAddress(), errors);
		} finally {
			errors.popNestedPath(); // Return to previous path
		}
	}
}
```

```kotlin
// Kotlin
class CustomerValidator(private val addressValidator: Validator) : Validator {

    init {
        requireNotNull(addressValidator) { "The supplied [Validator] is required and must not be null." }
        require(addressValidator.supports(Address::class.java)) {
            "The supplied [Validator] must support the validation of [Address] instances."
        }
    }

    /**
     * 이 Validator는 Customer 인스턴스 및 Customer의 모든 하위 클래스도 검증합니다
     */
    override fun supports(clazz: Class<*>): Boolean {
        return Customer::class.java.isAssignableFrom(clazz)
    }

    override fun validate(target: Any, errors: Errors) {
        ValidationUtils.rejectIfEmptyOrWhitespace(errors, "firstName", "field.required")
        ValidationUtils.rejectIfEmptyOrWhitespace(errors, "surname", "field.required")
        val customer = target as Customer
        try {
            errors.pushNestedPath("address") // Navigate into address property
            ValidationUtils.invokeValidator(this.addressValidator, customer.address, errors) // Assuming Customer has address property
        } finally {
            errors.popNestedPath() // Return to previous path
        }
    }
}
// Assuming Customer and Address classes exist
```

검증 오류는 검증기에 전달된 `Errors` 객체에 보고됩니다.

스프링 웹 MVC의 경우, `<spring:bind/>` 태그를 사용하여 오류 메시지를 검사할 수 있지만, `Errors` 객체를 직접 검사할 수도 있습니다.

*검증기는 또한 바인딩 프로세스를 포함하지 않고 주어진 객체의 즉각적인 검증을 위해 로컬에서 호출될 수도 있습니다.*

*6.1 버전부터 이것은 이제 기본적으로 사용 가능한 새로운 `Validator.validateObject(Object)` 메소드를 통해 단순화되었으며, 검사할 수 있는 간단한 `Errors` 표현을 반환합니다:*

*일반적으로 `hasErrors()` 또는 오류 요약 메시지를 예외로 변환하기 위한 새로운 `failOnError` 메소드를 호출합니다 (예: `validator.validateObject(myObject).failOnError(IllegalArgumentException::new)`).*

---

**전체 주제: 스프링의 Validator 인터페이스를 사용한 검증**

이 부분은 스프링이 제공하는 표준 `Validator` 인터페이스를 사용하여 **객체의 데이터를 검증**하고, 만약 **문제가 있다면 오류 정보를 수집**하는 방법에 대한 설명입니다.

**핵심 아이디어:** 객체의 데이터가 올바른 형식과 값을 가지고 있는지 체계적으로 검사하고, 문제가 있으면 오류 메시지를 기록하여 사용자에게 알려주거나 후속 처리를 하자!

---

**1. 왜 검증이 필요한가?**

- 사용자 입력값(예: 웹 폼 데이터), 외부 시스템 연동 데이터, 또는 객체 내부 상태가 애플리케이션의 **비즈니스 규칙이나 제약 조건**을 만족하는지 확인해야 합니다.
- 예시:
  - 이름은 비어 있으면 안 된다.
  - 나이는 음수이거나 너무 큰 값이면 안 된다.
  - 이메일은 형식에 맞아야 한다.
  - 주문 수량은 0보다 커야 한다.
- 검증 없이 데이터를 처리하면 예기치 않은 오류가 발생하거나, 잘못된 데이터가 시스템에 저장될 수 있습니다.

---

**2. 스프링의 `Validator` 인터페이스**

- `org.springframework.validation.Validator` 인터페이스는 스프링에서 데이터 검증 로직을 구현하기 위한 **표준 계약(contract)** 입니다.
- 이 인터페이스에는 두 개의 메소드가 있습니다:
  - **`boolean supports(Class<?> clazz)`:**
    - **역할:** 이 `Validator` 구현체가 **주어진 `clazz` 타입의 객체를 검증할 수 있는지 여부**를 반환합니다. `true`를 반환해야 해당 타입의 객체에 대해 `validate` 메소드가 호출될 수 있습니다.
    - **구현:** 보통 `return TargetClass.class.isAssignableFrom(clazz);` 와 같이 작성하여, 검증 대상 클래스 또는 그 하위 클래스들을 지원함을 나타냅니다.
  - **`void validate(Object target, Errors errors)`:**
    - **역할:** **실제 검증 로직**을 수행하는 메소드입니다.
    - **`target` 파라미터:** 검증할 대상 객체입니다. (타입 캐스팅 필요)
    - **`errors` 파라미터:** **검증 오류를 기록**하기 위한 객체입니다. (`org.springframework.validation.Errors`) 만약 검증 실패 조건을 발견하면, 이 `Errors` 객체의 메소드(예: `rejectValue`, `reject`)를 호출하여 어떤 필드에서 어떤 이유로 오류가 발생했는지 등록합니다.

---

**3. `Errors` 객체: 오류 기록장**

- `Errors` 인터페이스(및 그 구현체, 예: `BeanPropertyBindingResult`)는 검증 과정에서 발생한 **오류 정보들을 담는 역할**을 합니다.
- `validate` 메소드 내에서 검증 로직을 수행하다가 오류를 발견하면, `Errors` 객체에 다음과 같은 정보를 기록합니다:
  - **어떤 필드에서 오류가 발생했는지:** (예: "name", "age")
  - **어떤 종류의 오류인지 (오류 코드):** (예: "name.empty", "negativevalue") 이 코드는 나중에 `MessageSource`와 연동하여 사용자에게 보여줄 실제 오류 메시지(예: "이름은 필수입니다", "나이는 음수일 수 없습니다")로 변환될 수 있습니다.
  - (선택) 기본 오류 메시지
  - (선택) 오류 메시지에 들어갈 인자(argument)
- 주요 메소드:
  - `rejectValue(String field, String errorCode, ...)`: 특정 **필드**에 대한 오류를 등록합니다.
  - `reject(String errorCode, ...)`: 특정 필드와 관련 없는 **전체 객체**에 대한 오류를 등록합니다.

---

**4. `ValidationUtils` 헬퍼 클래스: 검증 코드 간소화**

- 스프링은 자주 사용되는 검증 로직(예: 값이 비어있는지, 공백만 있는지 등)을 쉽게 구현할 수 있도록 `org.springframework.validation.ValidationUtils` 라는 **헬퍼(도우미) 클래스**를 제공합니다.
- **`rejectIfEmpty(Errors errors, String field, String errorCode)`:** 지정된 필드 값이 `null`이거나 빈 문자열(`""`)이면 자동으로 `errors.rejectValue(...)`를 호출해줍니다.
- **`rejectIfEmptyOrWhitespace(Errors errors, String field, String errorCode)`:** `rejectIfEmpty` 기능에 더해, 값이 공백 문자(whitespace)로만 이루어진 경우도 오류로 처리합니다.
- 이 클래스를 사용하면 `if (value == null || value.isEmpty()) { ... }` 같은 코드를 반복해서 작성할 필요 없이 간결하게 검증 로직을 구현할 수 있습니다.

**5. `Validator` 구현 예시 (`PersonValidator`)**

```java
public class PersonValidator implements Validator {
    @Override
    public boolean supports(Class<?> clazz) {
        // 이 Validator는 Person 클래스만 검증 가능
        return Person.class.equals(clazz);
    }

    @Override
    public void validate(Object target, Errors errors) {
        // 1. ValidationUtils를 사용하여 'name' 필드가 비어있는지 검사
        ValidationUtils.rejectIfEmptyOrWhitespace(errors, "name", "name.required"); // 오류 코드: "name.required"

        // 2. target 객체를 Person 타입으로 캐스팅
        Person p = (Person) target;

        // 3. 'age' 필드에 대한 직접 검증 로직
        if (p.getAge() < 0) {
            // 나이가 음수면 오류 등록
            errors.rejectValue("age", "age.negative"); // 필드: "age", 오류 코드: "age.negative"
        } else if (p.getAge() > 120) {
            // 나이가 너무 많으면 오류 등록
            errors.rejectValue("age", "age.too_old"); // 필드: "age", 오류 코드: "age.too_old"
        }
    }
}
```

---

**6. 중첩 객체 검증 및 Validator 재사용**

- 만약 객체 안에 다른 객체가 포함된 경우(예: `Customer` 객체 안에 `Address` 객체 포함), 각 객체 타입에 대한 `Validator`를 따로 만드는 것이 좋습니다 (`CustomerValidator`, `AddressValidator`).
- `CustomerValidator`는 `AddressValidator`를 **의존성으로 주입**받거나 직접 생성하여 사용할 수 있습니다.
- `CustomerValidator`의 `validate` 메소드 내에서 `Address` 객체를 검증해야 할 때:
  1. `errors.pushNestedPath("address")`: 현재 오류 경로를 중첩된 객체의 속성 이름("address")으로 변경합니다. 이렇게 하면 `AddressValidator` 내에서 발생하는 오류가 "address.street", "address.city" 처럼 올바른 필드 경로로 기록됩니다.
  2. `ValidationUtils.invokeValidator(this.addressValidator, customer.getAddress(), errors)`: 주입받은 `AddressValidator`를 사용하여 `customer`의 `address` 객체를 검증합니다. 오류는 동일한 `errors` 객체에 기록됩니다.
  3. `errors.popNestedPath()`: 중첩 경로에서 빠져나와 원래 경로(Customer 객체 수준)로 돌아옵니다. (반드시 `finally` 블록 등에서 호출하여 경로가 꼬이지 않도록 해야 함)

---

**7. 검증 결과 사용**

- 검증 로직이 실행된 후, `Errors` 객체에는 발생한 모든 오류 정보가 담겨 있습니다.
- **스프링 웹 MVC:** `@Valid` 어노테이션과 함께 사용하면 컨트롤러 메소드에서 `BindingResult` ( `Errors`의 하위 인터페이스) 객체를 받아 오류 정보를 확인할 수 있습니다. `<spring:bind>` 태그 등을 사용하여 JSP 같은 뷰에서 오류 메시지를 사용자에게 보여줄 수 있습니다.
- **직접 사용:** 코드 내에서 `errors.hasErrors()` 메소드로 오류 발생 여부를 확인하거나, `errors.getAllErrors()`, `errors.getFieldErrors()` 등으로 구체적인 오류 목록을 가져와서 직접 처리할 수도 있습니다.
- **v6.1+ 편의 메소드:** `Validator.validateObject(object)` 메소드를 사용하면 `Errors` 객체를 직접 다루지 않고도 검증을 수행하고 결과를 받아 `hasErrors()`, `failOnError(...)` 등으로 간단히 처리할 수 있습니다.

**요약:**

스프링의 `Validator` 인터페이스는 객체 데이터의 **유효성을 검증**하는 표준적인 방법을 제공합니다. `supports()` 메소드로 검증 대상을 지정하고, `validate()` 메소드에서 실제 검증 로직을 구현하며, 오류 발생 시 `Errors` 객체에 오류 정보를 등록합니다. `ValidationUtils`는 검증 코드 작성을 도와주며, 중첩 객체 검증 시에는 `Validator`를 조합하고 `push/popNestedPath`를 사용합니다. 검증 결과는 `Errors` 객체를 통해 확인하고 활용할 수 있습니다.
