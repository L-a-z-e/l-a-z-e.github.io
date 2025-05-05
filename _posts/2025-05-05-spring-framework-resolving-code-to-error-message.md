---
title: Spring Framework Resolving Codes to Error Messages
description: 
author: laze
date: 2025-05-05 00:00:04 +0900
categories: [Dev, SpringBoot]
tags: [SpringBoot]
---
**Resolving Codes to Error Messages**

유효성 검사 오류에 해당하는 메시지를 출력하는 방법에 대해 학습해봅니다.

이전 포스팅에서 보여준 예제에서는 `name`과 `age` 필드를 거부(rejected)했습니다.

만약 `MessageSource`를 사용하여 오류 메시지를 출력하고 싶다면, 필드를 거부할 때 제공하는 오류 코드('name'과 'age'가 이 경우에 해당)를 사용하여 그렇게 할 수 있습니다.

사용자가 (직접적으로, 또는 예를 들어 `ValidationUtils` 클래스를 사용하여 간접적으로) `Errors` 인터페이스의 `rejectValue` 또는 다른 `reject` 메소드 중 하나를 호출하면,

내부 구현은 여러분이 전달한 코드뿐만 아니라 여러 추가 오류 코드도 등록합니다.

`MessageCodesResolver`는 `Errors` 인터페이스가 어떤 오류 코드를 등록할지 결정합니다.

기본적으로 `DefaultMessageCodesResolver`가 사용되며, 이는 (예를 들어) 여러분이 제공한 코드로 메시지를 등록할 뿐만 아니라, `reject` 메소드에 전달한 필드 이름을 포함하는 메시지도 등록합니다.

따라서, `rejectValue("age", "too.darn.old")`를 사용하여 필드를 거부하면, `too.darn.old` 코드 외에도 스프링은 `too.darn.old.age`(첫 번째는 필드 이름을 포함)와 `too.darn.old.age.int`(두 번째는 필드 타입을 포함)도 등록합니다.

이는 개발자가 오류 메시지를 타겟팅할 때 도움을 주기 위한 편의 기능으로 수행됩니다.

---

**전체 주제: 오류 코드를 오류 메시지로 해석하기 (Resolving Codes to Error Messages)**

이 부분은 스프링의 `Validator`가 `Errors` 객체에 **오류 코드**(예: "name.required", "age.negative")를 등록하면, 나중에 이 **코드를 기반으로 `MessageSource`를 통해 실제 사용자에게 보여줄 메시지**(예: "이름은 필수입니다.", "나이는 음수일 수 없습니다.")를 어떻게 찾아오는지 그 메커니즘을 설명합니다. `MessageCodesResolver`라는 인터페이스가 중요한 역할을 합니다.

**핵심 아이디어:** 검증 시에는 간단한 오류 코드만 기록하고, 나중에 보여줄 때는 이 코드를 사용하여 다국어 지원 등이 가능한 실제 메시지로 바꿔서 보여주자!

---

**상황 설정:**

이전 예제에서 사용했던 `Person` 클래스와 `PersonValidator`를 다시 사용하겠습니다. 그리고 사용자가 볼 메시지를 정의하는 프로퍼티 파일과 `MessageSource` 빈 설정이 필요합니다.

- **`Person.java` (데이터 객체)**

    ```java
    public class Person {
        private String name;
        private int age;
        // Getters and Setters...
        public String getName() { return name; }
        public void setName(String name) { this.name = name; }
        public int getAge() { return age; }
        public void setAge(int age) { this.age = age; }
    }
    
    ```

- **`PersonValidator.java` (검증 로직)**

    ```java
    import org.springframework.validation.Errors;
    import org.springframework.validation.ValidationUtils;
    import org.springframework.validation.Validator;
    
    public class PersonValidator implements Validator {
        @Override
        public boolean supports(Class<?> clazz) {
            return Person.class.equals(clazz);
        }
    
        @Override
        public void validate(Object target, Errors errors) {
            // 이름이 비어있으면 "name.required" 오류 코드 등록
            ValidationUtils.rejectIfEmptyOrWhitespace(errors, "name", "name.required");
    
            Person p = (Person) target;
    
            if (p.getAge() < 0) {
                // 나이가 음수이면 "age.negative" 오류 코드 등록
                errors.rejectValue("age", "age.negative");
            } else if (p.getAge() > 120) {
                // 나이가 너무 많으면 "age.too_old" 오류 코드 등록
                errors.rejectValue("age", "age.too_old");
            }
        }
    }
    
    ```

- **`messages.properties` (오류 메시지 정의 - 기본, 영어)**

    ```
    # --- 오류 코드에 대한 메시지 정의 ---
    name.required=Name is required.
    age.negative=Age cannot be negative.
    age.too_old=Age must be less than or equal to 120.
    
    # --- 필드 이름 포함 메시지 (나중에 설명) ---
    name.required.person.name=The name field of the person is required.
    age.negative.person.age=The age field of the person cannot be negative.
    age.too_old.person.age=The age field of the person must be less than or equal to 120.
    
    # --- 필드 타입 포함 메시지 (나중에 설명) ---
    typeMismatch.java.lang.Integer=Invalid number format. Please enter a valid integer.
    typeMismatch.int=Invalid number format. Please enter a valid integer.
    
    # --- 필드 레이블 (참고용) ---
    # field.label.name=User Name
    # field.label.age=User Age
    
    ```

- **`messages_ko.properties` (오류 메시지 정의 - 한국어)**

    ```
    name.required=이름은 필수 항목입니다.
    age.negative=나이는 음수일 수 없습니다.
    age.too_old=나이는 120세 이하여야 합니다.
    
    name.required.person.name=사람 객체의 이름 필드는 필수입니다.
    age.negative.person.age=사람 객체의 나이 필드는 음수일 수 없습니다.
    age.too_old.person.age=사람 객체의 나이 필드는 120 이하여야 합니다.
    
    typeMismatch.java.lang.Integer=잘못된 숫자 형식입니다. 유효한 정수를 입력해주세요.
    typeMismatch.int=잘못된 숫자 형식입니다. 유효한 정수를 입력해주세요.
    
    # field.label.name=사용자 이름
    # field.label.age=사용자 나이
    ```

- **스프링 설정 (`app-config.xml` 또는 Java Config)**

    ```xml
    <!-- MessageSource 빈 등록 -->
    <bean id="messageSource" class="org.springframework.context.support.ResourceBundleMessageSource">
        <property name="basenames">
            <list>
                <value>messages</value> <!-- messages.properties, messages_ko.properties 등을 찾음 -->
            </list>
        </property>
        <property name="defaultEncoding" value="UTF-8"/>
    </bean>
    
    <!-- Validator 빈 등록 (필요하다면) -->
    <bean id="personValidator" class="com.example.validation.PersonValidator"/>
    ```


---

**실제 사용 및 오류 메시지 해석 과정:**

이제 위 설정을 바탕으로 컨트롤러 등에서 어떻게 사용되고 메시지가 해석되는지 보겠습니다.

```java
import org.springframework.beans.factory.annotation.Autowired;
import org.springframework.context.MessageSource; // MessageSource 주입
import org.springframework.stereotype.Controller;
import org.springframework.ui.Model;
import org.springframework.validation.BindingResult; // Errors의 하위 인터페이스
import org.springframework.validation.DataBinder; // 데이터 바인딩 및 검증 실행
import org.springframework.web.bind.annotation.GetMapping;
import org.springframework.web.bind.annotation.PostMapping;
import java.util.Locale;

@Controller
public class PersonController {

    @Autowired
    private PersonValidator personValidator; // Validator 주입

    @Autowired
    private MessageSource messageSource; // MessageSource 주입

    @GetMapping("/person/add")
    public String showForm(Model model) {
        model.addAttribute("person", new Person());
        return "person-form";
    }

    @PostMapping("/person/add")
    public String processForm(Person person, BindingResult bindingResult, Model model, Locale locale) {

        // 1. 데이터 바인딩 (스프링 MVC가 자동으로 수행)
        //    만약 사용자가 age에 문자열 "abc"를 입력했다면, 이 단계에서 타입 변환 오류 발생
        //    스프링은 자동으로 Errors(BindingResult) 객체에 "typeMismatch" 같은 오류 코드 등록

        // 2. Validator를 사용하여 명시적으로 검증 수행
        personValidator.validate(person, bindingResult); // bindingResult가 Errors 역할

        // 3. 오류가 있는지 확인
        if (bindingResult.hasErrors()) {
            System.out.println("검증 오류 발생!");

            // 4. 오류 메시지 처리 및 모델에 추가 (예시)
            bindingResult.getAllErrors().forEach(error -> {
                // MessageSource를 사용하여 오류 코드를 실제 메시지로 변환
                // getMessage는 내부적으로 MessageCodesResolver 전략을 사용하여 여러 코드를 시도함
                String errorMessage = messageSource.getMessage(error, locale); // ★★★ 핵심 ★★★
                System.out.println("  - 오류 필드: " + (error instanceof org.springframework.validation.FieldError ? ((org.springframework.validation.FieldError)error).getField() : "전체 객체"));
                System.out.println("  - 오류 코드: " + error.getCode());
                System.out.println("  - 변환된 메시지 (" + locale + "): " + errorMessage);

                // 실제 웹 애플리케이션에서는 보통 <spring:message> 태그나 Thymeleaf의 #fields.errors() 등을 사용
                // 여기서는 콘솔 출력 예시
            });

            model.addAttribute("person", person); // 폼 데이터 유지를 위해 다시 모델에 추가
            return "person-form"; // 오류가 있으면 다시 폼 뷰로
        }

        // 검증 성공 시 로직
        System.out.println("검증 성공: " + person);
        return "person-success";
    }
}

```

**오류 코드 해석 과정 (`MessageCodesResolver`와 `DefaultMessageCodesResolver`)**

1. **`Validator`가 오류 등록:** `PersonValidator`의 `validate` 메소드에서 `errors.rejectValue("age", "age.negative");` 가 호출됩니다.
2. **`DefaultMessageCodesResolver` 동작:** `Errors` 객체(실제로는 `BindingResult` 등)는 내부적으로 `MessageCodesResolver` (기본 구현: `DefaultMessageCodesResolver`)를 사용합니다. `rejectValue("age", "age.negative")` 호출 시, 이 리졸버는 **하나가 아닌 여러 개의 오류 코드를 생성하여 `Errors` 객체에 등록**합니다. 생성되는 코드의 일반적인 규칙은 다음과 같습니다 (우선순위 순):
  - `errorCode + "." + objectName + "." + fieldName` (예: `age.negative.person.age`)
  - `errorCode + "." + fieldName` (예: `age.negative.age`)
  - `errorCode + "." + fieldType` (예: `age.negative.int`, `age.negative.java.lang.Integer`)
  - `errorCode` (예: `age.negative`)
  - (타입 불일치 오류의 경우) `typeMismatch.objectName.fieldName`, `typeMismatch.fieldName`, `typeMismatch.fieldType`, `typeMismatch`
3. **`MessageSource`가 메시지 검색:** 나중에 `messageSource.getMessage(error, locale)` 가 호출되면, `MessageSource`는 `error` 객체 안에 등록된 **여러 개의 오류 코드들을 `DefaultMessageCodesResolver`가 생성한 순서대로 하나씩 사용**하여 `messages_ko.properties`나 `messages.properties` 파일에서 **해당 코드를 키로 가진 메시지를 찾으려고 시도**합니다.
4. **가장 구체적인 메시지 우선:** 예를 들어 `age.negative.person.age` 코드가 메시지 파일에 정의되어 있다면 그 메시지가 사용됩니다. 없다면 다음 우선순위인 `age.negative.age` 코드를 찾고, 그것도 없으면 `age.negative.int`, 그리고 마지막으로 가장 일반적인 `age.negative` 코드를 찾아 사용합니다. 만약 모든 코드를 못 찾으면 예외가 발생하거나 기본 메시지(설정했다면)가 사용됩니다.

**왜 여러 개의 코드를 등록할까? (편의성)**

- 개발자는 매우 **구체적인 오류 메시지**(예: "사람 객체의 나이 필드는 음수일 수 없습니다." - 코드: `age.negative.person.age`)를 정의할 수도 있고,
- 좀 더 **일반적인 오류 메시지**(예: "나이는 음수일 수 없습니다." - 코드: `age.negative`)만 정의해 둘 수도 있습니다.
- `DefaultMessageCodesResolver`가 여러 코드를 생성해주기 때문에, 개발자는 필요한 수준의 구체성으로 메시지 파일에 키를 정의하기만 하면 스프링이 알아서 가장 적절한 메시지를 찾아줍니다. 모든 가능한 코드 조합을 미리 다 정의할 필요는 없습니다.

**결론:**

스프링의 검증 오류 처리 과정은 `Validator`가 **오류 코드**를 `Errors` 객체에 등록하고, `MessageSource`가 이 **코드를 기반으로 실제 사용자 메시지를 찾는** 방식으로 이루어집니다. 핵심은 `MessageCodesResolver`(주로 `DefaultMessageCodesResolver`)가 하나의 오류 발생에 대해 **여러 개의 후보 오류 코드를 생성**해주고, `MessageSource`는 이 코드들을 **우선순위에 따라** 메시지 파일에서 **검색**하여 가장 적합한 메시지를 찾아 반환한다는 것입니다. 이를 통해 개발자는 유연하고 체계적으로 오류 메시지를 관리하고 국제화할 수 있습니다.

---

**1. `BindingResult`가 왜 `processForm`의 파라미터로 먼저 들어오는가? (validate 끝나기 전에?)**

- **스프링 MVC의 자동 데이터 바인딩:** 사용자가 폼을 제출하면, 스프링 MVC는 `@PostMapping` 메소드를 호출하기 **전에** 다음과 같은 일을 먼저 시도합니다:
  1. HTTP 요청 파라미터(사용자가 폼에 입력한 값들)를 읽습니다.
  2. 메소드 파라미터에 있는 객체(여기서는 `Person person`)를 생성합니다.
  3. 읽어온 요청 파라미터 값들을 `Person` 객체의 해당 필드(name, age)에 **자동으로 채워 넣으려고 시도**합니다. 이것이 **자동 데이터 바인딩**입니다.
- **바인딩 오류 발생 가능성:** 이 자동 바인딩 과정에서 오류가 발생할 수 있습니다. 가장 흔한 오류는 **타입 불일치(Type Mismatch)** 입니다.
  - 예를 들어, `age` 필드는 `int` 타입인데 사용자가 숫자 대신 문자열 "abc"를 입력하면, 스프링은 "abc"를 `int`로 변환할 수 없어서 **바인딩 오류**가 발생합니다.
- **`BindingResult`의 역할 (1단계 - 자동 생성 및 오류 수집):**
  - 스프링 MVC는 이렇게 **자동 데이터 바인딩 과정에서 발생할 수 있는 오류들을 담기 위해**, 컨트롤러 메소드를 호출하기 **전에** `BindingResult` (또는 `Errors`) 객체를 **미리 생성**합니다.
  - 만약 자동 바인딩 중 타입 불일치 같은 오류가 발생하면, 스프링 MVC는 이 **미리 생성된 `BindingResult` 객체에 해당 오류 정보를 자동으로 등록**합니다.
  - **중요 컨벤션:** `BindingResult` 파라미터는 반드시 **검증 대상 객체(여기서는 `Person person`) 바로 뒤에 위치**해야 합니다. 그래야 스프링 MVC가 `person` 객체에 대한 바인딩 오류를 이 `bindingResult`에 담아줄 수 있습니다.
- **결론:** 따라서 `processForm` 메소드가 호출되는 시점에 파라미터로 전달되는 `bindingResult`는 단순히 비어있는 객체가 아니라, **자동 데이터 바인딩 과정에서 이미 발생했을 수 있는 오류들을 담고 있는 객체**일 수 있습니다. `validate` 메소드를 호출하기 전에도 이미 오류가 있을 수 있다는 뜻입니다!

**2. 왜 `personValidator.validate(person, bindingResult)`로 `BindingResult`가 또 매개변수로 들어가는가?**

이것은 `Validator` 인터페이스의 **설계(Contract)** 와 관련이 있습니다.

- **`Validator` 인터페이스의 `validate` 메소드 시그니처:** `void validate(Object target, Errors errors)` 입니다. 이 메소드는 검증 대상 객체(`target`)와 **오류를 기록할 `Errors` 객체**를 파라미터로 받도록 정의되어 있습니다.
- **`BindingResult`는 `Errors`의 자식:** 스프링 MVC에서 사용하는 `BindingResult` 인터페이스는 `Errors` 인터페이스를 **상속(extends)** 받습니다. 즉, `BindingResult` 객체는 `Errors` 타입으로 취급될 수 있습니다.
- **오류 통합:** `personValidator.validate(person, bindingResult)` 호출은 다음과 같은 의미입니다:
  - " `personValidator`야, `person` 객체를 네가 정의한 규칙에 따라 검증해줘."
  - "그리고 만약 검증하다가 **새로운 오류를 발견하면**, 내가 **미리 준비해서 전달하는 이 `bindingResult` 객체에다가 그 오류 정보들을 추가로 기록해줘.**"
- **결과:** 이렇게 함으로써,
  1. 스프링 MVC가 자동 데이터 바인딩 중에 발견한 **바인딩/타입 변환 오류**와
  2. 개발자가 직접 만든 `PersonValidator`가 비즈니스 규칙에 따라 발견한 **검증 오류**가
     **모두 하나의 `BindingResult` 객체에 통합되어 관리**됩니다.

**흐름 요약:**

1. HTTP 요청 수신.
2. 스프링 MVC가 `Person` 객체 생성 시도 및 요청 파라미터를 자동으로 바인딩 시도.
3. **(오류 발생 가능)** 타입 변환 실패 등 바인딩 오류 발생 시, **미리 생성된 `BindingResult` 객체에 오류 기록.**
4. 스프링 MVC가 `processForm` 메소드 호출. 이때 **오류가 기록되었을 수도 있는 `BindingResult` 객체**와 바인딩된(또는 실패한) `Person` 객체를 파라미터로 전달.
5. 개발자가 코드에서 `personValidator.validate(person, bindingResult)` 호출.
6. `PersonValidator`가 `person` 객체를 검증.
7. **(오류 발생 가능)** `PersonValidator`가 새로운 검증 오류 발견 시, **전달받은 동일한 `BindingResult` 객체에 오류 추가 기록.**
8. `if (bindingResult.hasErrors()) { ... }` 코드를 통해 **바인딩 오류와 검증 오류 모두**를 한 번에 확인하고 처리.

이제 `BindingResult`가 파라미터로 먼저 전달되고, 다시 `validate` 메소드에 전달되는 이유가 명확하게 이해되셨기를 바랍니다!
