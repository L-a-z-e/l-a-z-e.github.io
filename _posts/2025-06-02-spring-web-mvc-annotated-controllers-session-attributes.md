---
title: Spring Web MVC - Annotated Controllers (@SessionAttributes)
description: 
author: laze
date: 2025-06-02 00:00:01 +0900
categories: [Dev, SpringWebMVC]
tags: [SpringWebMVC]
---
## @SessionAttributes

[Reactive 스택의 동일 기능 보기](https://docs.spring.io/spring-framework/docs/current/reference/html/web-reactive.html#webflux-ann-sessionattributes)

`@SessionAttributes`는 여러 요청 간에 HTTP 서블릿 세션에 모델 속성을 저장하는 데 사용됩니다.

이는 특정 컨트롤러에서 사용될 세션 속성을 선언하는 타입 수준(type-level)의 애노테이션입니다.

일반적으로 후속 요청에서 접근할 수 있도록 세션에 투명하게 저장되어야 하는 모델 속성의 이름 또는 모델 속성의 타입을 나열합니다.

다음 예제는 `@SessionAttributes` 애노테이션을 사용합니다:

**Java**

```java
@Controller
@SessionAttributes("pet") // ①
public class EditPetForm {
    // ...
}
```

① `@SessionAttributes` 애노테이션 사용.

**Kotlin**

```kotlin
@Controller
@SessionAttributes("pet") // ①
class EditPetForm {
    // ...
}
```

① `@SessionAttributes` 애노테이션 사용.

첫 번째 요청에서, `pet`이라는 이름의 모델 속성이 모델에 추가되면, 이는 자동으로 HTTP 서블릿 세션으로 승격되어 저장됩니다. 다음 예제와 같이 다른 컨트롤러 메서드가 `SessionStatus` 메서드 인자를 사용하여 저장소를 지울 때까지 세션에 남아있게 됩니다:

**Java**

```java
@Controller
@SessionAttributes("pet") // ①
public class EditPetForm {

    // ...

    @PostMapping("/pets/{id}")
    public String handle(Pet pet, BindingResult errors, SessionStatus status) {
        if (errors.hasErrors()) {
            // ...
        }
        status.setComplete(); // ②
        // ...
        return "redirect:/some/path"; // 예시 리다이렉션 경로
    }
}
```

① `Pet` 값을 서블릿 세션에 저장.
② 서블릿 세션에서 `Pet` 값을 지움.

**Kotlin**

```kotlin
@Controller
@SessionAttributes("pet") // ①
class EditPetForm {

    // ...

    @PostMapping("/pets/{id}")
    fun handle(pet: Pet, errors: BindingResult, status: SessionStatus): String {
        if (errors.hasErrors()) {
            // ...
        }
        status.setComplete() // ②
        // ...
        return "redirect:/some/path" // 예시 리다이렉션 경로
    }
}
```

① `Pet` 값을 서블릿 세션에 저장.
② 서블릿 세션에서 `Pet` 값을 지움.

---

**✨ 이 챕터의 학습 목표 ✨**

1. **`@SessionAttributes` 애노테이션이 컨트롤러 수준에서 특정 모델 속성을 HTTP 세션에 저장하는 역할을 한다는 것을 이해합니다.** (어떤 데이터를 여러 요청에 걸쳐 유지할지 지정)
2. **모델에 추가된 특정 이름의 속성이 어떻게 자동으로 세션에 저장되고, 이후 요청에서 `@ModelAttribute`를 통해 어떻게 세션의 값을 우선적으로 가져오는지 과정을 이해합니다.**
3. **`SessionStatus` 객체의 `setComplete()` 메서드를 사용하여 `@SessionAttributes`로 관리되던 세션 데이터를 언제, 어떻게 정리하는지 배웁니다.** (세션 데이터의 생명주기 관리)

---

### 핵심 개념 설명

**HTTP 세션이란?**

먼저 HTTP 세션에 대한 간단한 이해가 필요합니다.

HTTP 프로토콜 자체는 상태를 저장하지 않습니다(stateless).

즉, 각 요청은 이전 요청과 독립적입니다.

하지만 웹 애플리케이션에서는 사용자가 로그인한 상태를 유지하거나, 장바구니 정보를 기억하는 등 여러 요청에 걸쳐 특정 데이터를 유지해야 할 필요가 있습니다.

이때 사용되는 것이 **세션(Session)**입니다.

서버는 각 클라이언트(웹 브라우저)마다 고유한 세션 ID를 발급하고, 이 ID와 연결된 저장 공간에 데이터를 보관합니다.

클라이언트는 이후 요청 시 이 세션 ID를 함께 보내고, 서버는 ID를 통해 해당 클라이언트의 데이터를 찾아 사용할 수 있게 됩니다.

**`@SessionAttributes`의 역할**

스프링 MVC의 `@SessionAttributes`는 이러한 HTTP 세션을 좀 더 편리하게, 그리고 선언적으로 사용할 수 있도록 도와주는 애노테이션입니다.

- **컨트롤러 수준(Type-level) 애노테이션:** `@Controller` 바로 위에 클래스 레벨로 선언합니다.
- **특정 모델 속성을 세션에 저장:** `@SessionAttributes`에 지정된 이름의 모델 속성(또는 특정 타입의 모델 속성)이 모델에 추가되면, 스프링 MVC는 이를 자동으로 HTTP 세션에도 복사해둡니다.
- **투명한 저장 및 접근:** 개발자는 세션에 직접 데이터를 넣고 빼는 코드를 작성할 필요 없이, 평소처럼 모델에 데이터를 추가하고 `@ModelAttribute`로 접근하면 됩니다. 스프링이 중간에서 알아서 세션과의 동기화를 처리해줍니다.

**왜 필요할까요? (사용 시나리오)**

1. **여러 단계로 진행되는 폼 처리 (Wizard-style form):**
  - 예를 들어, 상품 주문 과정이 1단계: 배송지 입력, 2단계: 결제 정보 입력, 3단계: 주문 확인 및 완료로 나뉘어 있다고 해봅시다.
  - 1단계에서 입력한 배송지 정보를 2단계, 3단계에서도 계속 사용해야 합니다. 이때 `@SessionAttributes`를 사용하여 1단계에서 생성된 주문 객체(배송지 정보 포함)를 세션에 저장해두면, 다음 단계에서 해당 객체를 쉽게 가져와 추가 정보를 채워나갈 수 있습니다.
2. **임시 데이터 보관:**
  - 사용자가 어떤 작업을 하다가 잠시 다른 페이지로 이동했다가 다시 돌아왔을 때, 이전에 작업하던 내용을 유지하고 싶을 때.
3. **리다이렉트(Redirect) 시 데이터 전달의 한계 극복:**
  - 일반적으로 리다이렉트 시에는 모델 데이터가 유지되지 않습니다. 하지만 세션에 저장된 데이터는 리다이렉트 후에도 접근 가능합니다. (`RedirectAttributes`를 사용하는 것이 더 일반적이긴 하지만, 세션도 한 방법입니다.)

**비유를 들어볼까요?**

- 여러분이 큰 도서관에서 여러 권의 책을 빌리려고 합니다.
  - **일반적인 모델:** 책 한 권을 빌릴 때마다 대출 카드에 기록하고, 다음 책을 빌릴 때는 카드를 새로 쓰는 것과 같습니다. (요청마다 모델은 새로움)
  - **`@SessionAttributes` 사용:** 여러분이 도서관에 입장할 때 받은 바구니(`세션`)가 있다고 상상해보세요. 빌리고 싶은 책(`모델 속성`)을 발견하면, 일단 바구니에 담아둡니다(`세션에 저장`). 다른 코너에 가서 또 다른 책을 바구니에 추가할 수 있습니다. 나중에 대출 데스크에 가서 바구니에 있는 모든 책을 한 번에 처리할 수 있습니다. 도서관을 나갈 때 바구니를 반납하면(`status.setComplete()`), 다음 방문 시에는 빈 바구니를 받게 됩니다.

### `@SessionAttributes` 동작 방식

1. **선언:** 컨트롤러 클래스에 `@SessionAttributes("attributeName")` 또는 `@SessionAttributes(types = MyType.class)` 형태로 선언합니다.
  - `"attributeName"`: 모델에 이 이름으로 추가되는 속성을 세션에 저장합니다. (가장 일반적)
  - `types = MyType.class`: 모델에 `MyType` 클래스(또는 그 하위 클래스)의 인스턴스가 추가되면 세션에 저장합니다.
2. **모델에 속성 추가 시 세션 저장:**
  - 해당 컨트롤러 내의 어떤 메서드에서든 (예: `@GetMapping` 메서드나 `@ModelAttribute`가 붙은 메서드) `@SessionAttributes`에 지정된 이름이나 타입의 객체를 `model.addAttribute("pet", aPetObject);` 와 같이 모델에 추가하면,
  - 스프링 MVC는 이 "pet"이라는 모델 속성을 HTTP 세션에도 **자동으로 복사**하여 저장합니다.
3. **후속 요청에서 모델 속성 접근 시 세션 우선 조회:**
  - 이후 같은 컨트롤러로 다른 요청이 들어왔을 때, 컨트롤러 메서드의 파라미터에 `@ModelAttribute("pet") Pet petInSession`과 같이 선언하면,
  - 스프링 MVC는 먼저 HTTP 세션에서 "pet"이라는 이름의 속성을 찾습니다.
  - **세션에 해당 속성이 있다면:** 그 객체를 가져와서 `petInSession` 파라미터에 바인딩합니다. (그리고 현재 요청 파라미터로 그 객체의 값을 덮어쓸 수 있습니다 - 이전 `@ModelAttribute` 챕터 내용)
  - **세션에 해당 속성이 없다면 (또는 처음 요청이라면):** 일반적인 `@ModelAttribute` 처리 방식(기본 생성자로 새로 만들거나, 다른 `@ModelAttribute` 메서드에서 가져오거나 등)을 따릅니다.
4. **세션 데이터 정리 (`SessionStatus`):**
  - `@SessionAttributes`로 관리되는 세션 데이터는 "대화(conversation)"가 끝났다고 판단될 때 명시적으로 정리해주어야 합니다. 그렇지 않으면 계속 세션에 남아 메모리를 차지하게 됩니다.
  - 컨트롤러 메서드의 파라미터로 `SessionStatus` 타입을 선언하여 받을 수 있습니다.
  - `status.setComplete();`를 호출하면, 해당 컨트롤러의 `@SessionAttributes`에 의해 관리되던 세션 데이터들이 **모두 제거**됩니다. (특정 속성만 골라서 지울 수는 없습니다. 해당 컨트롤러가 관리하는 것 전체가 정리됩니다.)
  - 주로 폼 처리의 마지막 단계(저장 완료 후)나, 사용자가 특정 작업을 취소했을 때 호출합니다.

### 코드 예제 및 분석 (원문 예제 기반)

**예제 1: `@SessionAttributes` 선언 및 세션 저장 흐름**

```java
package com.example.demo.controller;

import com.example.demo.model.Pet; // Pet 모델 클래스 가정
import org.springframework.stereotype.Controller;
import org.springframework.ui.Model;
import org.springframework.validation.BindingResult;
import org.springframework.web.bind.annotation.*;
import org.springframework.web.bind.support.SessionStatus;

@Controller
@RequestMapping("/pets")
@SessionAttributes("pet") // "pet"이라는 이름의 모델 속성을 세션에 저장하도록 선언
public class EditPetFormController {

    // 가상의 PetRepository
    // private PetRepository petRepository;
    // public EditPetFormController(PetRepository petRepository) { this.petRepository = petRepository; }

    // 1. 폼을 처음 보여주는 요청 (예: GET /pets/1/edit)
    @GetMapping("/{id}/edit")
    public String initUpdateForm(@PathVariable int id, Model model) {
        System.out.println("initUpdateForm - 세션에 pet 있는지 확인 전");
        if (!model.containsAttribute("pet")) { // 모델에 "pet"이 아직 없으면 (첫 방문 또는 세션 만료 등)
            System.out.println("initUpdateForm - 세션에 pet 없음. DB에서 로드 또는 새로 생성.");
            // Pet pet = petRepository.findById(id); // 실제로는 DB에서 Pet 객체를 로드
            Pet pet = new Pet(); // 임시로 새 객체 생성
            pet.setId(id);
        pet.setName("Temp Pet Name for ID: " + id); // 임시 이름
            model.addAttribute("pet", pet); // "pet"이라는 이름으로 모델에 추가
                                           // -> 이 시점에 @SessionAttributes("pet")에 의해 세션에도 저장됨!
        } else {
            System.out.println("initUpdateForm - 모델(세션)에 이미 pet 객체 존재: " + model.getAttribute("pet"));
        }
        return "petForm"; // 뷰 이름
    }

    // 2. 폼 제출 처리 (예: POST /pets/1/edit)
    @PostMapping("/{id}/edit")
    public String processUpdateForm(
            @ModelAttribute("pet") Pet pet, // 세션에 "pet"이 있으면 가져오고, 요청 파라미터로 덮어씀
            BindingResult result,
            SessionStatus status,
            @PathVariable int id, Model model) {

        System.out.println("processUpdateForm - 전달받은 Pet (세션 + 요청 바인딩): " + pet);

        if (result.hasErrors()) {
            System.out.println("processUpdateForm - 바인딩/유효성 오류 발생");
            // 오류가 있으면 다시 폼으로
            // 이때 model.addAttribute("pet", pet); 코드가 없어도
            // pet 객체는 @ModelAttribute에 의해 이미 모델에 있고,
            // @SessionAttributes에 의해 세션에도 유지되고 있으므로 폼에 값이 채워져서 보임
            return "petForm";
        } else {
            System.out.println("processUpdateForm - 성공. DB에 저장 시도.");
            // petRepository.save(pet); // 실제 DB에 저장
            System.out.println("processUpdateForm - DB 저장 완료. 세션 정리.");
            status.setComplete(); // "pet" 속성을 현재 컨트롤러의 세션에서 제거
            return "redirect:/owners/" + id; // 예시: 주인 상세 페이지로 리다이렉트
        }
    }

    // (참고) 만약 @ModelAttribute 메서드가 있다면
    // @ModelAttribute("pet")
    // public Pet initializePet() {
    //     // 이 메서드가 먼저 호출되어 "pet"을 모델과 세션에 넣을 수도 있음
    //     // 그러면 initUpdateForm에서 DB 로드 로직이 달라질 수 있음
    //     System.out.println("@ModelAttribute initializePet() called");
    //     return new Pet();
    // }
}

```

(간단한 `Pet` 클래스)

```java
// Pet.java
package com.example.demo.model;
public class Pet {
    private Integer id;
    private String name;
    private String type;
    // Getters and Setters
    public Integer getId() { return id; }
    public void setId(Integer id) { this.id = id; }
    public String getName() { return name; }
    public void setName(String name) { this.name = name; }
    public String getType() { return type; }
    public void setType(String type) { this.type = type; }
    @Override public String toString() { return "Pet{id=" + id + ", name='" + name + "', type='" + type + "'}"; }
}

```

**분석:**

- **`@SessionAttributes("pet")`**: 이 컨트롤러에서 "pet"이라는 이름으로 모델에 추가되는 객체는 세션에도 저장됩니다.
- **`initUpdateForm` 메서드 (폼 보여주기):**
  - `@PathVariable int id`로 수정할 Pet의 ID를 받습니다.
  - `model.containsAttribute("pet")`로 모델(그리고 세션)에 "pet"이 이미 있는지 확인합니다.
    - **만약 없다면 (첫 방문 등):** DB에서 `Pet` 객체를 로드하거나 새로 생성해서 `model.addAttribute("pet", pet);`로 모델에 추가합니다. 이 순간, 이 "pet" 객체는 세션에도 저장됩니다.
    - **만약 있다면:** 세션에 있는 "pet" 객체가 이미 모델에 로드된 상태입니다.
  - `petForm.html` (가정) 뷰로 이동합니다. 이 뷰는 모델에 있는 "pet" 객체의 정보를 사용하여 폼을 채웁니다.
- **`processUpdateForm` 메서드 (폼 제출 처리):**
  - `@ModelAttribute("pet") Pet pet`:
    1. 스프링은 먼저 세션에서 "pet" 속성을 찾습니다. `initUpdateForm`에서 세션에 저장했으므로, 해당 객체를 가져옵니다.
    2. 가져온 세션의 "pet" 객체에 현재 POST 요청으로 들어온 폼 데이터(예: `name=newName&type=newType`)를 바인딩(덮어쓰기)합니다.
  - `BindingResult result`: 바인딩 또는 유효성 검사 오류를 확인합니다.
    - **오류 시:** `petForm`으로 다시 돌아갑니다. 이때 `pet` 객체는 사용자가 수정한 값을 그대로 가지고 있으며, 세션에도 그 상태로 유지되므로 폼에 수정된 값이 그대로 보입니다.
    - **성공 시:**
      - `pet` 객체를 DB에 저장합니다.
      - `status.setComplete();`: **매우 중요!** 이 컨트롤러가 `@SessionAttributes`를 통해 관리하던 "pet" 세션 데이터를 제거합니다. 이렇게 하지 않으면, 사용자가 다른 Pet을 수정하려고 하거나, 관련 없는 다른 페이지로 이동해도 세션에 이전 "pet" 정보가 계속 남아있을 수 있습니다.
      - 성공 페이지로 리다이렉트합니다.

**왜 `status.setComplete()`가 중요할까요?**

만약 `status.setComplete()`를 호출하지 않고 `EditPetFormController`에서 다른 애완동물을 수정하려고 `/pets/2/edit`로 접근하면, 세션에는 여전히 이전에 수정하던 (ID가 1인) "pet" 객체가 남아있을 수 있습니다. 그러면 `@ModelAttribute("pet")`은 세션에 있는 ID 1번 Pet 객체를 가져오고, 거기에 요청 파라미터를 덮어쓰려고 시도하여 의도치 않은 동작을 유발할 수 있습니다.

`setComplete()`는 "이 대화(특정 Pet 수정 작업)는 끝났으니, 관련된 세션 데이터는 정리해주세요" 라는 의미입니다.

### 주의사항 및 Best Practice

1. **`SessionStatus.setComplete()` 호출 시점:** 대화형 작업(예: 폼 마법사)이 완전히 종료되었을 때 반드시 호출하여 세션 데이터를 정리해야 합니다. 일반적으로 성공적인 저장 후 또는 취소 시에 호출합니다.
2. **세션 속성 이름의 명확성:** `@SessionAttributes`에 지정하는 이름은 모델에 추가하는 속성의 이름과 정확히 일치해야 합니다.
3. **과도한 세션 사용 지양:** 세션은 서버 메모리를 사용합니다. 너무 많은 데이터나 너무 큰 객체를 세션에 저장하면 서버 성능에 부담을 줄 수 있습니다. 꼭 필요한 최소한의 데이터만 세션에 유지하는 것이 좋습니다.
4. **`@SessionAttributes` 범위:** `@SessionAttributes`는 해당 컨트롤러 내에서만 유효합니다. 다른 컨트롤러에서 동일한 이름의 모델 속성을 사용해도 세션을 공유하지 않습니다 (물론 직접 HTTP 세션 API를 사용하면 가능하지만, `@SessionAttributes`의 범위는 컨트롤러 단위입니다).
5. **`@ModelAttribute`와의 관계:** `@SessionAttributes`는 `@ModelAttribute`와 긴밀하게 연관되어 동작합니다. 세션에서 값을 가져와 `@ModelAttribute` 파라미터에 바인딩하고, `@ModelAttribute` 메서드가 반환하는 값을 세션에 저장하는 등의 상호작용을 이해하는 것이 중요합니다.

### 이전 학습 내용과의 연관성

- **`@ModelAttribute`:**
  - `@SessionAttributes`에 지정된 이름의 객체가 모델에 추가될 때 (주로 `@ModelAttribute`를 통해 반환되거나 `model.addAttribute`로 추가될 때) 세션에 저장됩니다.
  - 컨트롤러 메서드 파라미터의 `@ModelAttribute`는 세션에 해당 속성이 있으면 우선적으로 세션에서 값을 가져와 사용합니다.
- **컨트롤러 (`@Controller`)**: `@SessionAttributes`는 컨트롤러 클래스 레벨에 선언됩니다.
- **모델 (`Model`)**: 세션에 저장될 데이터는 먼저 모델에 추가되어야 합니다. 모델은 컨트롤러와 뷰 사이의 데이터 전달 매개체이자, 세션 저장의 트리거가 됩니다.

---
