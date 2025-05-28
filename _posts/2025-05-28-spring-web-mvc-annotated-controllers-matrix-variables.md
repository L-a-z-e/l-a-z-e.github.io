---
title: Spring Web MVC - Annotated Controllers (Matrix Variables)
description: 
author: laze
date: 2025-05-28 00:00:01 +0900
categories: [Dev, SpringWebMVC]
tags: [SpringWebMVC]
---
## 매트릭스 변수 (Matrix Variables)

[Reactive 스택의 동일 기능 보기](https://docs.spring.io/spring-framework/docs/current/reference/html/web-reactive.html#webflux-ann-matrix-vars)

[RFC 3986](https://tools.ietf.org/html/rfc3986)은 경로 세그먼트 내의 이름-값 쌍에 대해 논의합니다.

스프링 MVC에서는 팀 버너스리(Tim Berners-Lee)의 "오래된 게시물"에 기반하여 이를 "매트릭스 변수"라고 부르지만, URI 경로 파라미터라고도 할 수 있습니다.

매트릭스 변수는 모든 경로 세그먼트에 나타날 수 있으며, 각 변수는 세미콜론(;)으로 구분되고 여러 값은 쉼표(,)로 구분됩니다 (예: `/cars;color=red,green;year=2012`).

여러 값은 반복된 변수 이름을 통해서도 지정할 수 있습니다 (예: `color=red;color=green;color=blue`).

URL에 매트릭스 변수가 포함될 것으로 예상되는 경우, 컨트롤러 메서드의 요청 매핑은 해당 변수 내용을 가리기 위해 URI 변수를 사용해야 하며, 매트릭스 변수의 순서나 존재 여부와 관계없이 요청이 성공적으로 매칭되도록 보장해야 합니다.

다음 예제는 매트릭스 변수를 사용합니다:

**Java**

```java
// GET /pets/42;q=11;r=22

@GetMapping("/pets/{petId}")
public void findPet(@PathVariable String petId, @MatrixVariable int q) {

    // petId == 42
    // q == 11
}
```

**Kotlin**

```kotlin
// GET /pets/42;q=11;r=22

@GetMapping("/pets/{petId}")
fun findPet(@PathVariable petId: String, @MatrixVariable q: Int) {

    // petId == "42"
    // q == 11
}
```

모든 경로 세그먼트에 매트릭스 변수가 포함될 수 있으므로, 때로는 매트릭스 변수가 어떤 경로 변수에 속할 것으로 예상되는지 명확히 구분해야 할 필요가 있습니다. 다음 예제는 이를 수행하는 방법을 보여줍니다:

**Java**

```java
// GET /owners/42;q=11/pets/21;q=22

@GetMapping("/owners/{ownerId}/pets/{petId}")
public void findPet(
        @MatrixVariable(name="q", pathVar="ownerId") int q1,
        @MatrixVariable(name="q", pathVar="petId") int q2) {

    // q1 == 11
    // q2 == 22
}
```

**Kotlin**

```kotlin
// GET /owners/42;q=11/pets/21;q=22

@GetMapping("/owners/{ownerId}/pets/{petId}")
fun findPet(
        @MatrixVariable(name="q", pathVar="ownerId") q1: Int,
        @MatrixVariable(name="q", pathVar="petId") q2: Int) {

    // q1 == 11
    // q2 == 22
}
```

다음 예제와 같이 매트릭스 변수는 선택 사항으로 정의하고 기본값을 지정할 수 있습니다:

**Java**

```java
// GET /pets/42

@GetMapping("/pets/{petId}")
public void findPet(@MatrixVariable(required=false, defaultValue="1") int q) {

    // q == 1
}
```

**Kotlin**

```kotlin
// GET /pets/42

@GetMapping("/pets/{petId}")
fun findPet(@MatrixVariable(required = false, defaultValue = "1") q: Int) {

    // q == 1
}
```

모든 매트릭스 변수를 가져오려면 다음 예제와 같이 `MultiValueMap`을 사용할 수 있습니다:

**Java**

```java
// GET /owners/42;q=11;r=12/pets/21;q=22;s=23

@GetMapping("/owners/{ownerId}/pets/{petId}")
public void findPet(
        @MatrixVariable MultiValueMap<String, String> matrixVars,
        @MatrixVariable(pathVar="petId") MultiValueMap<String, String> petMatrixVars) {

    // matrixVars: ["q" : [11,22], "r" : [12], "s" : [23]] (문서 수정: r과 s도 List<String>으로 받는게 일반적)
    // petMatrixVars: ["q" : [22], "s" : [23]]
}
```

**Kotlin**

```kotlin
// GET /owners/42;q=11;r=12/pets/21;q=22;s=23

@GetMapping("/owners/{ownerId}/pets/{petId}")
fun findPet(
        @MatrixVariable matrixVars: MultiValueMap<String, String>,
        @MatrixVariable(pathVar="petId") petMatrixVars: MultiValueMap<String, String>) {

    // matrixVars: ["q" : [11,22], "r" : [12], "s" : [23]]
    // petMatrixVars: ["q" : [22], "s" : [23]]
}
```

매트릭스 변수 사용을 활성화해야 한다는 점에 유의하십시오. MVC Java 설정에서는 경로 매칭(Path Matching)을 통해 `removeSemicolonContent=false`로 설정된 `UrlPathHelper`를 설정해야 합니다.

MVC XML 네임스페이스에서는 `<mvc:annotation-driven enable-matrix-variables="true"/>`를 설정할 수 있습니다.

---

**✨ 이 챕터의 학습 목표 ✨**

1. **매트릭스 변수(Matrix Variable)가 무엇인지, URL 경로에서 어떻게 표현되는지 이해합니다.** (세미콜론 `;` 을 사용한 키-값 쌍)
2. **`@MatrixVariable` 애노테이션을 사용하여 컨트롤러 메서드에서 매트릭스 변수 값을 특정 경로 변수와 연관시켜 가져오는 방법을 배웁니다.** ( `pathVar` 속성 사용법 포함)
3. **매트릭스 변수를 사용하기 위해 스프링 MVC 설정을 어떻게 변경해야 하는지 인지합니다.** (`UrlPathHelper` 또는 XML 설정)

---

### 핵심 개념 설명

- *매트릭스 변수(Matrix Variable)**란 URL 경로의 각 세그먼트( `/`로 구분되는 부분)에 추가적인 파라미터를 키-값 형태로 전달하는 방법입니다. 우리가 흔히 사용하는 쿼리 파라미터(`?key=value`)와 비슷하지만, 쿼리 파라미터는 URL 전체의 끝에 붙는 반면, 매트릭스 변수는 URL 경로 중간중간에 들어갈 수 있다는 차이가 있습니다.

**표현 방식:**

- 세미콜론( `;` )으로 시작합니다.
- 키와 값은 등호( `=` )로 연결합니다. (예: `;key=value` )
- 하나의 경로 세그먼트에 여러 매트릭스 변수를 사용할 경우, 각 변수는 세미콜론( `;` )으로 구분합니다. (예: `;key1=value1;key2=value2` )
- 하나의 키에 여러 값을 전달할 때는 쉼표( `,` )를 사용하거나, 키를 반복해서 사용합니다.
  - 쉼표 사용: `/cars;color=red,green,blue`
  - 키 반복: `/cars;color=red;color=green;color=blue`

**비유를 들어볼까요?**

- 우리가 도서관에서 책을 찾는다고 상상해봅시다.
  - **쿼리 파라미터 방식:** "컴퓨터 과학 섹션에서 '자바'라는 단어가 포함된 책을 찾아주세요." → `/books/search?category=cs&keyword=java` (책 전체에서 찾는 느낌)
  - **매트릭스 변수 방식:** "컴퓨터 과학 섹션(이 섹션은 2020년 이후 출판된 책만)에서 '자바'라는 책(이 책은 초급자용)을 찾아주세요." → `/books;publishedAfter=2020/java;level=beginner` (각 경로 단계별로 조건을 거는 느낌)

  이렇게 매트릭스 변수를 사용하면 각 경로 세그먼트에 대한 좀 더 구체적인 정보를 명시적으로 표현할 수 있습니다.


**왜 필요할까요?**

- **RESTful API 설계의 유연성:** 경로의 각 리소스에 대해 좀 더 세부적인 식별자나 필터링 조건을 URL 자체에 표현하고 싶을 때 유용합니다.
- **계층적 데이터 표현:** `/resource1;criteria1=value1/resource2;criteria2=value2` 처럼 각 계층의 리소스에 독립적인 파라미터를 부여할 수 있습니다.

하지만 매트릭스 변수는 쿼리 파라미터만큼 널리 사용되지는 않습니다.

그 이유는 URL이 다소 복잡해 보일 수 있고, 모든 웹 서버나 프레임워크에서 기본적으로 완벽하게 지원하지 않을 수도 있기 때문입니다.

스프링 MVC에서는 명시적으로 활성화 설정을 해주어야 합니다.

### 주요 용어 해설

- **매트릭스 변수 (Matrix Variable / URI Path Parameter):** URL 경로 세그먼트 내에 세미콜론( `;` )으로 구분하여 키-값 쌍으로 전달되는 파라미터입니다.
- **경로 세그먼트 (Path Segment):** URL에서 슬래시( `/` )로 구분되는 각 부분을 의미합니다. 예를 들어 `/owners/42/pets/21` 에서 `owners`, `42`, `pets`, `21` 이 각각 경로 세그먼트입니다.
- **`@MatrixVariable`**: 컨트롤러 메서드의 인자에서 매트릭스 변수 값을 추출할 때 사용하는 애노테이션입니다.
  - `name` 속성: 가져올 매트릭스 변수의 이름을 지정합니다. (생략 시 인자 이름 사용)
  - `pathVar` 속성: 어떤 경로 변수( `@PathVariable`로 지정된)에 속한 매트릭스 변수를 가져올지 지정합니다. 여러 경로 세그먼트에 동일한 이름의 매트릭스 변수가 있을 때 구분하기 위해 사용됩니다.
  - `required` 속성: 해당 매트릭스 변수가 필수인지 여부를 지정합니다. (기본값: `true`)
  - `defaultValue` 속성: `required=false`일 때, 해당 매트릭스 변수가 없으면 사용할 기본값을 지정합니다.
- **`MultiValueMap<String, String>`**: 하나의 키에 여러 값을 가질 수 있는 맵입니다. 매트릭스 변수가 `key=v1,v2` 또는 `key=v1;key=v2` 형태로 전달될 때 유용하게 사용할 수 있습니다.
- **`UrlPathHelper`**: 스프링 MVC에서 URL 경로를 파싱하고 처리하는 데 사용되는 헬퍼 클래스입니다. 매트릭스 변수를 사용하려면 이 클래스의 `removeSemicolonContent` 속성을 `false`로 설정해야 합니다.
  - `removeSemicolonContent=true` (기본값): 경로에서 세미콜론( `;` ) 이후의 내용을 제거합니다. 즉, 매트릭스 변수를 인식하지 못합니다.
  - `removeSemicolonContent=false`: 세미콜론 이후의 내용(매트릭스 변수)을 유지하여 파싱합니다.
- **`<mvc:annotation-driven enable-matrix-variables="true"/>`**: XML 기반 설정에서 매트릭스 변수 사용을 활성화하는 방법입니다. Java 설정에서는 `WebMvcConfigurer`를 통해 `UrlPathHelper`를 직접 설정합니다.

### 코드 예제 및 분석

**매트릭스 변수 활성화 설정 (Java Config)**

가장 먼저, 매트릭스 변수를 사용하려면 스프링 MVC 설정을 변경해야 합니다.

```java
import org.springframework.context.annotation.Configuration;
import org.springframework.web.servlet.config.annotation.PathMatchConfigurer;
import org.springframework.web.servlet.config.annotation.WebMvcConfigurer;
import org.springframework.web.util.UrlPathHelper;

@Configuration
public class WebConfig implements WebMvcConfigurer {

    @Override
    public void configurePathMatch(PathMatchConfigurer configurer) {
        UrlPathHelper urlPathHelper = new UrlPathHelper();
        urlPathHelper.setRemoveSemicolonContent(false); // 세미콜론 내용 제거 비활성화
        configurer.setUrlPathHelper(urlPathHelper);
    }
}
```

**분석:**

- `WebMvcConfigurer` 인터페이스를 구현한 설정 클래스를 만듭니다.
- `configurePathMatch` 메서드를 오버라이드합니다.
- `UrlPathHelper` 객체를 생성하고, `setRemoveSemicolonContent(false)`를 호출하여 매트릭스 변수를 인식하도록 설정합니다.
- 이 `urlPathHelper`를 `PathMatchConfigurer`에 설정해줍니다.

이제 컨트롤러에서 `@MatrixVariable`을 사용할 준비가 되었습니다.

**예제 1: 기본적인 매트릭스 변수 사용** (원문 예제)

```java
import org.springframework.stereotype.Controller;
import org.springframework.web.bind.annotation.GetMapping;
import org.springframework.web.bind.annotation.MatrixVariable;
import org.springframework.web.bind.annotation.PathVariable;
import org.springframework.web.bind.annotation.ResponseBody;

@Controller
public class MatrixVarController {

    // 요청 URL: GET /pets/42;q=11;r=22
    @GetMapping("/pets/{petId}")
    @ResponseBody
    public String findPet(@PathVariable String petId, @MatrixVariable int q) {
        // petId에는 "42"가 할당됩니다. (경로 변수)
        // q에는 11이 할당됩니다. (매트릭스 변수, 타입 변환 자동 수행)
        // r=22는 여기서는 사용되지 않습니다.
        return "Pet ID: " + petId + ", q: " + q; // "Pet ID: 42, q: 11"
    }
}
```

**분석:**

- `@GetMapping("/pets/{petId}")`:  `/pets/` 다음에 오는 경로 세그먼트를 `petId`라는 경로 변수로 받습니다.
- `@PathVariable String petId`: `petId` 경로 변수의 값을 문자열로 받습니다. (예: "42")
- `@MatrixVariable int q`: `petId` 경로 세그먼트 (즉, `42`가 있는 부분)에 있는 매트릭스 변수 중 이름이 `q`인 값을 `int` 타입으로 받습니다. 스프링은 "11"이라는 문자열을 숫자 `11`로 자동 타입 변환합니다.
- URL의 `r=22`는 `@MatrixVariable`로 요청하지 않았으므로 무시됩니다.

**예제 2: 특정 경로 변수에 속한 매트릭스 변수 지정** (원문 예제)

```java
// 요청 URL: GET /owners/42;q=11/pets/21;q=22
@GetMapping("/owners/{ownerId}/pets/{petId}")
@ResponseBody
public String findOwnerPet(
        @PathVariable String ownerId,
        @PathVariable String petId,
        @MatrixVariable(name="q", pathVar="ownerId") int q1, // ownerId 세그먼트의 q
        @MatrixVariable(name="q", pathVar="petId") int q2) {  // petId 세그먼트의 q

    // ownerId == "42"
    // petId == "21"
    // q1 == 11 (owners/42;q=11 에서 가져옴)
    // q2 == 22 (pets/21;q=22 에서 가져옴)
    return "Owner ID: " + ownerId + ", q1: " + q1 + " | Pet ID: " + petId + ", q2: " + q2;
    // "Owner ID: 42, q1: 11 | Pet ID: 21, q2: 22"
}
```

**분석:**

- `@GetMapping("/owners/{ownerId}/pets/{petId}")`: 두 개의 경로 변수 `ownerId`와 `petId`를 정의합니다.
- `@MatrixVariable(name="q", pathVar="ownerId") int q1`:
  - `pathVar="ownerId"`: `ownerId`라는 이름의 경로 변수에 해당하는 경로 세그먼트(여기서는 `/owners/42;q=11` 부분)에서 매트릭스 변수를 찾습니다.
  - `name="q"`: 그중에서 이름이 `q`인 매트릭스 변수의 값을 가져옵니다. (`11`)
- `@MatrixVariable(name="q", pathVar="petId") int q2`:
  - `pathVar="petId"`: `petId`라는 이름의 경로 변수에 해당하는 경로 세그먼트(여기서는 `/pets/21;q=22` 부분)에서 매트릭스 변수를 찾습니다.
  - `name="q"`: 그중에서 이름이 `q`인 매트릭스 변수의 값을 가져옵니다. (`22`)
- 만약 `pathVar`를 명시하지 않으면, 스프링은 기본적으로 URL의 첫 번째 경로 변수에 해당하는 세그먼트에서 매트릭스 변수를 찾으려고 시도하거나, 이름이 유일하다면 모호하지 않게 찾을 수도 있습니다. 하지만 명확성을 위해 여러 경로 변수가 있을 때는 `pathVar`를 사용하는 것이 좋습니다.

**예제 3: 선택적 매트릭스 변수와 기본값** (원문 예제)

```java
// 요청 URL 1: GET /items/101;type=A
// 요청 URL 2: GET /items/101
@GetMapping("/items/{itemId}")
@ResponseBody
public String getItem(@PathVariable String itemId,
                      @MatrixVariable(name="type", required=false, defaultValue="DefaultType") String itemType) {
    // 요청 URL 1의 경우: itemId="101", itemType="A"
    // 요청 URL 2의 경우: itemId="101", itemType="DefaultType" (매트릭스 변수 없으므로 기본값 사용)
    return "Item ID: " + itemId + ", Type: " + itemType;
}
```

**분석:**

- `@MatrixVariable(name="type", required=false, defaultValue="DefaultType") String itemType`:
  - `name="type"`: `type`이라는 이름의 매트릭스 변수를 찾습니다.
  - `required=false`: 이 매트릭스 변수는 필수가 아닙니다. URL에 없어도 오류가 발생하지 않습니다.
  - `defaultValue="DefaultType"`: 만약 `type` 매트릭스 변수가 URL에 없다면, `itemType` 변수에는 "DefaultType"이라는 문자열이 할당됩니다.
- 이것은 이전 챕터에서 배운 `@RequestParam`의 `required` 및 `defaultValue` 속성과 동일한 방식으로 동작합니다.

**예제 4: 모든 매트릭스 변수 받기 (MultiValueMap)** (원문 예제)

```java
import org.springframework.util.MultiValueMap;
// ... 다른 import 생략 ...

// 요청 URL: GET /catalog/books;author=kim;lang=ko/journals;year=2023;lang=en
@GetMapping("/catalog/{section1}/{section2}")
@ResponseBody
public String getCatalogItems(
        @PathVariable String section1,
        @PathVariable String section2,
        @MatrixVariable(pathVar="section1") MultiValueMap<String, String> section1Vars,
        @MatrixVariable MultiValueMap<String, String> allMatrixVars) { // pathVar 없으면 모든 경로 세그먼트의 모든 매트릭스 변수

    // section1Vars: {"author" : ["kim"], "lang" : ["ko"]} (books 세그먼트의 변수들)
    // allMatrixVars: {"author" : ["kim"], "lang" : ["ko", "en"], "year" : ["2023"]} (모든 세그먼트의 변수들, lang은 중복 키)

    return "Section 1 Vars: " + section1Vars + "<br/>All Matrix Vars: " + allMatrixVars;
}
```

**분석:**

- `@MatrixVariable(pathVar="section1") MultiValueMap<String, String> section1Vars`:
  - `pathVar="section1"`: `section1` (여기서는 "books") 경로 세그먼트에 있는 모든 매트릭스 변수를 `MultiValueMap` 형태로 받습니다.
  - `MultiValueMap`은 키 하나에 여러 값을 가질 수 있어서 `lang=ko,en` 같은 경우나 `lang=ko;lang=en` 같은 경우를 모두 처리할 수 있습니다. (예제에서는 `lang`이 다른 세그먼트에 있지만, 만약 `/books;lang=ko;lang=en` 이었다면 `section1Vars`의 `lang` 키에 `["ko", "en"]`이 들어갑니다.)
- `@MatrixVariable MultiValueMap<String, String> allMatrixVars`:
  - `pathVar`를 지정하지 않으면, 요청 URL에 있는 **모든 경로 세그먼트의 모든 매트릭스 변수**를 하나의 `MultiValueMap`으로 받습니다.
  - 만약 서로 다른 경로 세그먼트에 동일한 이름의 매트릭스 변수가 있다면 (예제에서 `lang`), `MultiValueMap`에는 해당 키로 모든 값이 리스트 형태로 합쳐져서 들어갑니다. (예: `lang` 키에 `["ko", "en"]`이 저장됨)

### "왜?" 라는 질문에 대한 답변

**Q: 왜 굳이 매트릭스 변수를 써야 하나요? 쿼리 파라미터로도 충분하지 않나요?**

A: 네, 대부분의 경우 쿼리 파라미터로 충분합니다. 하지만 다음과 같은 상황에서 매트릭스 변수가 더 적합하거나 표현력이 좋을 수 있습니다.

1. **경로의 특정 리소스에 대한 세부 식별/필터링:**
   예를 들어, `/cars;color=red/engines;type=diesel` 처럼 "빨간색 자동차들 중에서 디젤 엔진을 가진 것"을 표현할 때, 각 리소스(`cars`, `engines`)에 대한 필터 조건이 명확하게 해당 리소스 경로 세그먼트에 붙어 있어 의미가 더 명확해질 수 있습니다. 쿼리 파라미터로 `/cars/engines?carColor=red&engineType=diesel` 로 표현할 수도 있지만, 매트릭스 변수는 각 조건이 어떤 리소스에 귀속되는지를 시각적으로 더 잘 보여줍니다.
2. **리소스 식별자의 일부로 사용될 때:**
   때로는 리소스 식별 자체가 여러 요소로 구성될 수 있습니다. 예를 들어, 특정 버전의 문서를 가리킬 때 `/documents/guide;version=1.2` 와 같이 사용될 수 있습니다.
3. **계층적 데이터에 대한 파라미터:**`/continents/asia;populationFilter=over10M/countries/korea;economy=developed`
   위와 같이 각 계층(대륙, 국가)마다 독립적인 파라미터를 적용하고 싶을 때 유용합니다.

하지만 앞서 언급했듯이, URL이 복잡해지고, 매트릭스 변수를 사용하기 위한 추가 설정이 필요하며, 널리 사용되지 않아 다른 개발자에게 생소할 수 있다는 단점도 있습니다. 따라서 꼭 필요한 경우가 아니라면 쿼리 파라미터가 더 간단하고 일반적인 선택일 수 있습니다. "이 기술을 사용해야만 하는가?"를 항상 고민해보는 것이 좋습니다.

**Q: `UrlPathHelper`의 `removeSemicolonContent`를 `false`로 설정하지 않으면 어떻게 되나요?**

A: 만약 `removeSemicolonContent`가 기본값인 `true`로 되어 있다면, 스프링 MVC는 URL 경로에서 세미콜론( `;` )과 그 이후의 모든 내용을 "의미 없는 내용"으로 간주하고 제거해버립니다.
예를 들어, 요청 URL이 `/pets/42;q=11` 이고 `removeSemicolonContent=true` 라면, 스프링 MVC는 이 경로를 `/pets/42` 로만 인식합니다. 따라서 `@MatrixVariable`은 아무런 값도 받지 못하거나, `required=true` (기본값)라면 매트릭스 변수를 찾지 못해 예외가 발생할 수 있습니다.

### 주의사항 및 Best Practice

1. **활성화 설정 필수:** 매트릭스 변수를 사용하려면 반드시 `UrlPathHelper`의 `removeSemicolonContent`를 `false`로 설정해야 합니다. (Java Config 또는 XML Config) 이 설정을 잊으면 매트릭스 변수가 전혀 동작하지 않습니다.
2. **URL 가독성 및 복잡도:** 매트릭스 변수를 과도하게 사용하면 URL이 매우 길고 복잡해져 가독성을 해칠 수 있습니다. 꼭 필요한 경우에만 간결하게 사용하는 것이 좋습니다.
3. **`pathVar`의 명확한 사용:** 여러 경로 변수 (`@PathVariable`)가 있는 URL에서 매트릭스 변수를 사용할 때는, 어떤 경로 변수에 속한 매트릭스 변수를 가져올지 `@MatrixVariable`의 `pathVar` 속성을 명시해주는 것이 혼동을 줄이고 코드의 의도를 명확하게 합니다.
4. **타입 변환:** `@RequestParam`과 마찬가지로 `@MatrixVariable`도 기본적인 타입 변환(문자열 -> 숫자, 불리언 등)을 지원합니다.
5. **대안 고려:** 매트릭스 변수가 정말 최선의 선택인지, 쿼리 파라미터나 다른 방식으로 동일한 목적을 더 간단하게 달성할 수 없는지 항상 고민해보세요. API 설계는 명확성과 단순성이 중요합니다.
6. **RFC 3986과의 관계:** 원문에서 언급된 RFC 3986은 URI의 일반적인 구문에 대한 표준 문서입니다. 매트릭스 변수는 이 표준에서 "path segment parameters"라는 개념으로 논의될 수 있지만, 모든 웹 환경에서 동일하게 해석되거나 지원되는 것은 아닙니다. 스프링 MVC는 이를 "매트릭스 변수"라는 이름으로 지원하는 것입니다.

### 이전 학습 내용과의 연관성

- **타입 변환(Type Conversion):** 바로 이전 챕터에서 배운 타입 변환 개념이 `@MatrixVariable`에도 그대로 적용됩니다. 예를 들어 `@MatrixVariable int q` 처럼 선언하면, URL의 문자열 매트릭스 값(예: "11")이 `int` 타입으로 자동 변환됩니다.
- **`@PathVariable`:** 매트릭스 변수는 특정 `@PathVariable`로 식별되는 경로 세그먼트에 종속되는 경우가 많습니다. 따라서 `@PathVariable`의 개념을 이해하고 있어야 `@MatrixVariable`의 `pathVar` 속성을 효과적으로 사용할 수 있습니다.
- **애노테이션 기반 컨트롤러:** `@GetMapping`, `@PathVariable`, `@RequestParam` 등과 마찬가지로 `@MatrixVariable`도 애노테이션을 통해 선언적으로 프로그래밍하는 스프링 MVC의 특징을 보여줍니다.
- **`required` 및 `defaultValue` 속성:** `@RequestParam`에서 사용했던 `required`와 `defaultValue` 속성이 `@MatrixVariable`에도 동일하게 존재하며, 비슷한 방식으로 동작합니다.

---
