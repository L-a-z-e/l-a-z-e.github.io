---
title: Spring Web MVC - Annotated Controllers (Mapping Requests)
description: 
author: laze
date: 2025-05-23 00:00:01 +0900
categories: [Dev, SpringWebMVC]
tags: [SpringWebMVC]
---
---

**요청 매핑 (Mapping Requests)**

`@RequestMapping` 어노테이션을 사용하여 요청을 컨트롤러 메소드에 매핑할 수 있습니다.

이 어노테이션은 URL, HTTP 메소드, 요청 파라미터, 헤더 및 미디어 타입별로 매칭하기 위한 다양한 속성을 가지고 있습니다.

클래스 레벨에서 사용하여 공유 매핑을 표현하거나, 메소드 레벨에서 사용하여 특정 엔드포인트 매핑으로 범위를 좁힐 수 있습니다.

`@RequestMapping`의 HTTP 메소드 특정 바로 가기 변형들도 있습니다:

- `@GetMapping`
- `@PostMapping`
- `@PutMapping`
- `@DeleteMapping`
- `@PatchMapping`

이러한 바로 가기들은 커스텀 어노테이션(Custom Annotations)이며, 대부분의 컨트롤러 메소드가 기본적으로 모든 HTTP 메소드와 매칭되는 `@RequestMapping`을 사용하는 대신

특정 HTTP 메소드에 매핑되어야 한다는 주장에 따라 제공됩니다.

공유 매핑을 표현하기 위해서는 여전히 클래스 레벨에 `@RequestMapping`이 필요합니다.

`@RequestMapping`은 동일한 요소(클래스, 인터페이스 또는 메소드)에 선언된 다른 `@RequestMapping` 어노테이션과 함께 사용할 수 없습니다.

동일한 요소에 여러 개의 `@RequestMapping` 어노테이션이 감지되면 경고가 기록되고 첫 번째 매핑만 사용됩니다.

이는 `@GetMapping`, `@PostMapping` 등과 같은 합성 `@RequestMapping` 어노테이션에도 적용됩니다.

다음 예제는 타입 및 메소드 레벨 매핑을 보여줍니다:

**Java**

```java
@RestController
@RequestMapping("/persons")
class PersonController {

	@GetMapping("/{id}")
	public Person getPerson(@PathVariable Long id) {
		// ...
	}

	@PostMapping
	@ResponseStatus(HttpStatus.CREATED)
	public void add(@RequestBody Person person) {
		// ...
	}
}
```

**URI 패턴 (URI patterns)**

`@RequestMapping` 메소드는 URL 패턴을 사용하여 매핑될 수 있습니다. 두 가지 대안이 있습니다:

- **`PathPattern`**: URL 경로(역시 `PathContainer`로 미리 파싱됨)에 대해 매칭되는 미리 파싱된 패턴입니다. 웹 사용을 위해 설계된 이 솔루션은 인코딩 및 경로 파라미터를 효과적으로 처리하고 효율적으로 매칭합니다.
- **`AntPathMatcher`**: 문자열 경로에 대해 문자열 패턴을 매칭합니다. 이는 클래스패스, 파일 시스템 및 기타 위치에서 리소스를 선택하기 위해 Spring 설정에서도 사용되는 원래 솔루션입니다. 덜 효율적이며 문자열 경로 입력은 URL의 인코딩 및 기타 문제들을 효과적으로 처리하는 데 어려움이 있습니다.

`PathPattern`은 웹 애플리케이션에 권장되는 솔루션이며 Spring WebFlux에서는 유일한 선택입니다.

Spring MVC에서는 버전 5.3부터 사용할 수 있게 되었고 버전 6.0부터 기본적으로 활성화됩니다.

경로 매칭 옵션의 사용자 정의는 MVC 설정(MVC config)을 참조하세요.

`PathPattern`은 `AntPathMatcher`와 동일한 패턴 구문을 지원합니다.

또한, 경로 끝에서 0개 이상의 경로 세그먼트를 매칭하기 위한 캡처링 패턴(예: `{*spring}`)도 지원합니다.

`PathPattern`은 또한 여러 경로 세그먼트를 매칭하기 위한 `**`의 사용을 패턴 끝에서만 허용하도록 제한합니다.

이는 주어진 요청에 가장 적합한 매칭 패턴을 선택할 때 발생하는 많은 모호성을 제거합니다.

몇 가지 예시 패턴:

- `"/resources/ima?e.png"` - 경로 세그먼트에서 한 문자 매칭
- `"/resources/*.png"` - 경로 세그먼트에서 0개 이상의 문자 매칭
- `"/resources/**"` - 여러 경로 세그먼트 매칭
- `"/projects/{project}/versions"` - 경로 세그먼트를 매칭하고 변수로 캡처
- `"/projects/{project:[a-z]+}/versions"` - 정규식을 사용하여 변수 매칭 및 캡처

캡처된 URI 변수는 `@PathVariable`을 사용하여 접근할 수 있습니다. 예:

```java
@GetMapping("/owners/{ownerId}/pets/{petId}")
public Pet findPet(@PathVariable Long ownerId, @PathVariable Long petId) {
	// ...
}
```

다음 예제와 같이 클래스 및 메소드 레벨에서 URI 변수를 선언할 수 있습니다:

```java
@Controller
@RequestMapping("/owners/{ownerId}")
public class OwnerController {

	@GetMapping("/pets/{petId}")
	public Pet findPet(@PathVariable Long ownerId, @PathVariable Long petId) {
		// ...
	}
}
```

URI 변수는 자동으로 적절한 타입으로 변환되거나 `TypeMismatchException`이 발생합니다.

단순 타입(`int`, `long`, `Date` 등)은 기본적으로 지원되며 다른 데이터 타입에 대한 지원을 등록할 수 있습니다.

URI 변수의 이름을 명시적으로 지정할 수 있지만(예: `@PathVariable("customId")`), 이름이 동일하고 코드가 `-parameters` 컴파일러 플래그로 컴파일된 경우에는 해당 세부 정보를 생략할 수 있습니다.

`{varName:regex}` 구문은 `{varName:regex}` 구문을 가진 정규식으로 URI 변수를 선언합니다.

예를 들어, URL "/spring-web-3.0.5.jar"이 주어졌을 때 다음 메소드는 이름, 버전 및 파일 확장자를 추출합니다:

```java
@GetMapping("/{name:[a-z-]+}-{version:\\\\d\\\\.\\\\d\\\\.\\\\d}{ext:\\\\.[a-z]+}")
public void handle(@PathVariable String name, @PathVariable String version, @PathVariable String ext) {
	// ...
}
```

URI 경로 패턴에는 로컬, 시스템, 환경 및 기타 프로퍼티 소스에 대해 `PropertySourcesPlaceholderConfigurer`를 사용하여 시작 시 해석되는 내장된 `${…}` 플레이스홀더가 있을 수도 있습니다.

예를 들어, 일부 외부 설정을 기반으로 기본 URL을 매개변수화하는 데 사용할 수 있습니다.

**패턴 비교 (Pattern Comparison)**

여러 패턴이 URL과 일치할 때 가장 적합한 일치 항목을 선택해야 합니다.

이는 파싱된 `PathPattern` 사용이 활성화되었는지 여부에 따라 다음 중 하나로 수행됩니다:

- `PathPattern.SPECIFICITY_COMPARATOR`
- `AntPathMatcher.getPatternComparator(String path)`

둘 다 더 구체적인 패턴을 위로 정렬하는 데 도움이 됩니다.

패턴은 URI 변수(1로 계산됨), 단일 와일드카드(1로 계산됨) 및 이중 와일드카드(2로 계산됨)의 수가 적을수록 더 구체적입니다.

점수가 같으면 더 긴 패턴이 선택됩니다.

점수와 길이가 같으면 와일드카드보다 URI 변수가 더 많은 패턴이 선택됩니다.

기본 매핑 패턴(`/**`)은 점수 계산에서 제외되며 항상 마지막으로 정렬됩니다.

또한, 접두사 패턴(예: `/public/**`)은 이중 와일드카드가 없는 다른 패턴보다 덜 구체적인 것으로 간주됩니다.

**접미사 매칭 (Suffix Match)**
5.3부터 Spring MVC는 기본적으로 `/person`에 매핑된 컨트롤러가 암묵적으로 `/person.*`에도 매핑되는 `.*` 접미사 패턴 매칭을 더 이상 수행하지 않습니다.

결과적으로 경로 확장은 더 이상 응답에 대해 요청된 콘텐츠 타입을 해석하는 데 사용되지 않습니다(예: `/person.pdf`, `/person.xml` 등).

이러한 방식으로 파일 확장자를 사용하는 것은 브라우저가 일관성 있게 해석하기 어려운 `Accept` 헤더를 보내던 시절에 필요했습니다.

현재는 더 이상 필요하지 않으며 `Accept` 헤더를 사용하는 것이 선호되는 선택입니다.

시간이 지남에 따라 파일 이름 확장자 사용은 다양한 방식으로 문제가 있는 것으로 입증되었습니다.

URI 변수, 경로 파라미터 및 URI 인코딩 사용과 중첩될 때 모호성을 유발할 수 있습니다.

URL 기반 권한 부여 및 보안(자세한 내용은 다음 섹션 참조)에 대해 추론하는 것도 더 어려워집니다.

5.3 이전 버전에서 경로 확장자 사용을 완전히 비활성화하려면 다음을 설정하십시오:

- `useSuffixPatternMatching(false)`, PathMatchConfigurer 참조
- `favorPathExtension(false)`, ContentNegotiationConfigurer 참조

"Accept" 헤더 이외의 방법으로 콘텐츠 타입을 요청하는 방법은 여전히 유용할 수 있습니다(예: 브라우저에 URL을 입력할 때). 경로 확장에 대한 안전한 대안은 쿼리 파라미터 전략을 사용하는 것입니다.

파일 확장자를 사용해야 하는 경우, `ContentNegotiationConfigurer`의 `mediaTypes` 속성을 통해 명시적으로 등록된 확장자 목록으로 제한하는 것을 고려하십시오.

**접미사 매칭과 RFD (Suffix Match and RFD)**
반사 파일 다운로드(RFD, Reflected File Download) 공격은 요청 입력(예: 쿼리 파라미터 및 URI 변수)이 응답에 반영되는 것에 의존한다는 점에서 XSS와 유사합니다.

그러나 HTML에 JavaScript를 삽입하는 대신 RFD 공격은 브라우저가 다운로드를 수행하도록 전환하고 나중에 더블 클릭했을 때 응답을 실행 가능한 스크립트로 처리하는 것에 의존합니다.

Spring MVC에서 `@ResponseBody` 및 `ResponseEntity` 메소드는 클라이언트가 URL 경로 확장을 통해 요청할 수 있는 다양한 콘텐츠 타입을 렌더링할 수 있기 때문에 위험합니다.

접미사 패턴 매칭을 비활성화하고 콘텐츠 협상을 위해 경로 확장을 사용하는 것은 위험을 낮추지만 RFD 공격을 막기에는 충분하지 않습니다.

RFD 공격을 방지하기 위해, 응답 본문을 렌더링하기 전에 Spring MVC는 고정되고 안전한 다운로드 파일 이름을 제안하기 위해 `Content-Disposition:inline;filename=f.txt` 헤더를 추가합니다.

이는 URL 경로에 안전한 것으로 허용되지 않거나 콘텐츠 협상을 위해 명시적으로 등록되지 않은 파일 확장자가 포함된 경우에만 수행됩니다.

그러나 URL을 브라우저에 직접 입력할 때 부작용이 있을 수 있습니다.

많은 일반적인 경로 확장자는 기본적으로 안전한 것으로 허용됩니다.

사용자 정의 `HttpMessageConverter` 구현이 있는 애플리케이션은 해당 확장에 대해 `Content-Disposition` 헤더가 추가되지 않도록 콘텐츠 협상을 위해 파일 확장자를 명시적으로 등록할 수 있습니다.

**소비 가능한 미디어 타입 (Consumable Media Types)**

다음 예제와 같이 요청의 `Content-Type`을 기반으로 요청 매핑의 범위를 좁힐 수 있습니다:

```java
@PostMapping(path = "/pets", consumes = "application/json")
public void addPet(@RequestBody Pet pet) {
	// ...
}
```

콘텐츠 타입으로 매핑 범위를 좁히기 위해 `consumes` 속성 사용.

`consumes` 속성은 부정 표현식도 지원합니다.

예를 들어, `!text/plain`은 `text/plain` 이외의 모든 콘텐츠 타입을 의미합니다.

클래스 레벨에서 공유 `consumes` 속성을 선언할 수 있습니다.

그러나 대부분의 다른 요청 매핑 속성과 달리 클래스 레벨에서 사용될 때 메소드 레벨 `consumes` 속성은 클래스 레벨 선언을 확장하는 대신 재정의합니다.

`MediaType`은 `APPLICATION_JSON_VALUE` 및 `APPLICATION_XML_VALUE`와 같이 일반적으로 사용되는 미디어 타입에 대한 상수를 제공합니다.

**생산 가능한 미디어 타입 (Producible Media Types)**

다음 예제와 같이 `Accept` 요청 헤더와 컨트롤러 메소드가 생산하는 콘텐츠 타입 목록을 기반으로 요청 매핑의 범위를 좁힐 수 있습니다:

```java
@GetMapping(path = "/pets/{petId}", produces = "application/json")
@ResponseBody
public Pet getPet(@PathVariable String petId) {
	// ...
}
```

콘텐츠 타입으로 매핑 범위를 좁히기 위해 `produces` 속성 사용.

미디어 타입은 문자 집합을 지정할 수 있습니다.

부정 표현식이 지원됩니다. 예를 들어, `!text/plain`은 "text/plain" 이외의 모든 콘텐츠 타입을 의미합니다.

클래스 레벨에서 공유 `produces` 속성을 선언할 수 있습니다.

그러나 대부분의 다른 요청 매핑 속성과 달리 클래스 레벨에서 사용될 때 메소드 레벨 `produces` 속성은 클래스 레벨 선언을 확장하는 대신 재정의합니다.

`MediaType`은 `APPLICATION_JSON_VALUE` 및 `APPLICATION_XML_VALUE`와 같이 일반적으로 사용되는 미디어 타입에 대한 상수를 제공합니다.

**파라미터, 헤더 (Parameters, headers)**

요청 파라미터 조건을 기반으로 요청 매핑의 범위를 좁힐 수 있습니다.

요청 파라미터의 존재(`myParam`), 부재(`!myParam`) 또는 특정 값(`myParam=myValue`)을 테스트할 수 있습니다.

다음 예제는 특정 값을 테스트하는 방법을 보여줍니다:

**Java**

```java
@GetMapping(path = "/pets/{petId}", params = "myParam=myValue")
public void findPet(@PathVariable String petId) {
	// ...
}

```

`myParam`이 `myValue`와 같은지 테스트.

다음 예제와 같이 요청 헤더 조건에도 동일하게 사용할 수 있습니다:

**Java**

```java
@GetMapping(path = "/pets/{petId}", headers = "myHeader=myValue")
public void findPet(@PathVariable String petId) {
	// ...
}
```

`myHeader`가 `myValue`와 같은지 테스트.

`headers` 조건으로 `Content-Type` 및 `Accept`를 매칭할 수 있지만, 대신 `consumes` 및 `produces`를 사용하는 것이 좋습니다.

**HTTP HEAD, OPTIONS**

`@GetMapping` (및 `@RequestMapping(method=HttpMethod.GET)`)은 요청 매핑을 위해 HTTP HEAD를 투명하게 지원합니다.

컨트롤러 메소드는 변경할 필요가 없습니다.

`jakarta.servlet.http.HttpServlet`에 적용된 응답 래퍼는 `Content-Length` 헤더가 (응답에 실제로 쓰지 않고) 쓰여진 바이트 수로 설정되도록 보장합니다.

기본적으로 HTTP OPTIONS는 일치하는 URL 패턴을 가진 모든 `@RequestMapping` 메소드에 나열된 HTTP 메소드 목록으로 `Allow` 응답 헤더를 설정하여 처리됩니다.

HTTP 메소드 선언이 없는 `@RequestMapping`의 경우, `Allow` 헤더는 `GET,HEAD,POST,PUT,PATCH,DELETE,OPTIONS`로 설정됩니다.

컨트롤러 메소드는 항상 지원되는 HTTP 메소드를 선언해야 합니다 (예: HTTP 메소드 특정 변형 사용: `@GetMapping`, `@PostMapping` 등).

`@RequestMapping` 메소드를 HTTP HEAD 및 HTTP OPTIONS에 명시적으로 매핑할 수 있지만, 일반적인 경우에는 필요하지 않습니다.

**커스텀 어노테이션 (Custom Annotations)**

Spring MVC는 요청 매핑을 위해 합성 어노테이션(composed annotations) 사용을 지원합니다.

이는 `@RequestMapping`으로 메타 어노테이트되고 더 좁고 구체적인 목적으로 `@RequestMapping` 속성의 하위 집합(또는 전체)을 재선언하도록 구성된 어노테이션입니다.

`@GetMapping`, `@PostMapping`, `@PutMapping`, `@DeleteMapping`, `@PatchMapping`은 합성 어노테이션의 예입니다.

이는 대부분의 컨트롤러 메소드가 기본적으로 모든 HTTP 메소드와 매칭되는 `@RequestMapping`을 사용하는 대신 특정 HTTP 메소드에 매핑되어야 한다는 주장에 따라 제공됩니다.

합성 어노테이션을 구현하는 방법에 대한 예제가 필요한 경우 해당 어노테이션이 어떻게 선언되었는지 살펴보십시오.

`@RequestMapping`은 동일한 요소(클래스, 인터페이스 또는 메소드)에 선언된 다른 `@RequestMapping` 어노테이션과 함께 사용할 수 없습니다.

동일한 요소에 여러 개의 `@RequestMapping` 어노테이션이 감지되면 경고가 기록되고 첫 번째 매핑만 사용됩니다.

이는 `@GetMapping`, `@PostMapping` 등과 같은 합성 `@RequestMapping` 어노테이션에도 적용됩니다.

Spring MVC는 또한 사용자 정의 요청 매칭 로직을 가진 사용자 정의 요청 매핑 속성을 지원합니다.

이는 `RequestMappingHandlerMapping`을 하위 클래스화하고 `getCustomMethodCondition` 메소드를 재정의해야 하는 더 고급 옵션으로,

여기서 사용자 정의 속성을 확인하고 자신만의 `RequestCondition`을 반환할 수 있습니다.

**명시적 등록 (Explicit Registrations)**

핸들러 메소드를 프로그래밍 방식으로 등록할 수 있으며, 이는 동적 등록이나 다른 URL에서 동일한 핸들러의 다른 인스턴스와 같은 고급 사례에 사용할 수 있습니다.

다음 예제는 핸들러 메소드를 등록합니다:

```java
@Configuration
public class MyConfig {

	@Autowired
	public void setHandlerMapping(RequestMappingHandlerMapping mapping, UserHandler handler)
			throws NoSuchMethodException {

		RequestMappingInfo info = RequestMappingInfo
				.paths("/user/{id}").methods(RequestMethod.GET).build();

		Method method = UserHandler.class.getMethod("getUser", Long.class);

		mapping.registerMapping(info, handler, method);
	}
}
```

1. 대상 핸들러와 컨트롤러용 핸들러 매핑을 주입합니다.
2. 요청 매핑 메타데이터를 준비합니다.
3. 핸들러 메소드를 가져옵니다.
4. 등록을 추가합니다.

`@HttpExchange`의 주요 목적은 생성된 프록시로 HTTP 클라이언트 코드를 추상화하는 것이지만, 이러한 어노테이션이 배치되는 HTTP 인터페이스는 클라이언트 대 서버 사용에 중립적인 계약입니다.

클라이언트 코드를 단순화하는 것 외에도 HTTP 인터페이스가 서버가 클라이언트 액세스를 위해 API를 노출하는 편리한 방법일 수 있는 경우도 있습니다.

이는 클라이언트와 서버 간의 결합도를 높이며, 특히 공용 API의 경우 종종 좋은 선택이 아니지만 내부 API의 경우 정확히 목표일 수 있습니다.

이는 Spring Cloud에서 일반적으로 사용되는 접근 방식이며, 이것이 `@HttpExchange`가 컨트롤러 클래스에서 서버 측 처리를 위해 `@RequestMapping`의 대안으로 지원되는 이유입니다.

```java
@HttpExchange("/persons")
interface PersonService {

	@GetExchange("/{id}")
	Person getPerson(@PathVariable Long id);

	@PostExchange
	void add(@RequestBody Person person);
}

@RestController
class PersonController implements PersonService {

	public Person getPerson(@PathVariable Long id) {
		// ...
	}

	@ResponseStatus(HttpStatus.CREATED)
	public void add(@RequestBody Person person) {
		// ...
	}
}
```

`@HttpExchange`와 `@RequestMapping`에는 차이점이 있습니다.

`@RequestMapping`은 경로 패턴, HTTP 메소드 등으로 여러 요청에 매핑할 수 있는 반면, `@HttpExchange`는 구체적인 HTTP 메소드, 경로 및 콘텐츠 타입을 가진 단일 엔드포인트를 선언합니다.

메소드 파라미터 및 반환 값의 경우, 일반적으로 `@HttpExchange`는 `@RequestMapping`이 지원하는 메소드 파라미터의 하위 집합을 지원합니다.

특히 서버 측 특정 파라미터 유형은 제외합니다.

`@HttpExchange`는 또한 클라이언트 측에서 `@RequestMapping(headers={})`과 같이 "이름=값"과 유사한 쌍을 허용하는 `headers()` 파라미터를 지원합니다.

서버 측에서는 이것이 `@RequestMapping`이 지원하는 전체 구문으로 확장됩니다.

---

## Part 1: `@RequestMapping` 기본과 URI 패턴 매칭

Spring MVC 컨트롤러에서 특정 URL 요청을 어떤 메소드가 처리할지 지정하는 데 사용되는 가장 핵심적인 어노테이션은 **`@RequestMapping`** 입니다.

### 1. `@RequestMapping` 어노테이션

- **역할:** 들어오는 HTTP 요청을 특정 핸들러 메소드(컨트롤러의 메소드)에 매핑(연결)합니다.
- **사용 위치:**
  - **클래스 레벨:** 해당 클래스 내의 모든 핸들러 메소드에 대한 공통적인 기본 경로(base path)를 지정할 때 사용합니다.
  - **메소드 레벨:** 특정 핸들러 메소드가 처리할 구체적인 경로 및 조건을 지정합니다.
- **주요 속성:**
  - `value` 또는 `path`: 매핑할 URL 경로 패턴을 지정합니다. (예: `"/users"`, `"/items/{itemId}"`) 여러 개를 지정할 수도 있습니다: `path = {"/hello", "/greeting"}`
  - `method`: 요청을 처리할 HTTP 메소드를 지정합니다. (예: `RequestMethod.GET`, `RequestMethod.POST`). 지정하지 않으면 모든 HTTP 메소드와 매칭됩니다.
  - `consumes`: 요청 본문의 `Content-Type`을 기준으로 매핑을 제한합니다. (예: `consumes = "application/json"`)
  - `produces`: 응답으로 생성할 `Content-Type`을 기준으로 매핑을 제한합니다. (클라이언트의 `Accept` 헤더와 비교) (예: `produces = "application/json"`)
  - `params`: 특정 요청 파라미터가 존재하거나 특정 값을 가질 때만 매핑합니다. (예: `params = "myParam=myValue"`)
  - `headers`: 특정 요청 헤더가 존재하거나 특정 값을 가질 때만 매핑합니다. (예: `headers = "myHeader=myValue"`)

### 2. HTTP 메소드 특정 단축 어노테이션

매번 `method` 속성을 지정하는 것이 번거로울 수 있어, Spring은 각 HTTP 메소드에 대한 단축 어노테이션을 제공합니다. 이들은 내부적으로 `@RequestMapping`을 사용하며 해당 HTTP 메소드가 미리 설정되어 있습니다.

- **`@GetMapping`**: HTTP GET 요청을 매핑 (`@RequestMapping(method = RequestMethod.GET)`과 동일)
- **`@PostMapping`**: HTTP POST 요청을 매핑 (`@RequestMapping(method = RequestMethod.POST)`와 동일)
- **`@PutMapping`**: HTTP PUT 요청을 매핑 (`@RequestMapping(method = RequestMethod.PUT)`와 동일)
- **`@DeleteMapping`**: HTTP DELETE 요청을 매핑 (`@RequestMapping(method = RequestMethod.DELETE)`와 동일)
- **`@PatchMapping`**: HTTP PATCH 요청을 매핑 (`@RequestMapping(method = RequestMethod.PATCH)`와 동일)

**권장 사항:** 대부분의 경우, 특정 HTTP 메소드에만 응답하도록 컨트롤러 메소드를 작성하는 것이 좋으므로, `@RequestMapping`보다는 이러한 단축 어노테이션을 사용하는 것이 더 명확하고 권장됩니다.

클래스 레벨의 공유 매핑에는 여전히 `@RequestMapping`을 사용할 수 있습니다.

**주의:** 하나의 메소드나 클래스에 `@RequestMapping`과 그 단축 어노테이션들을 **중복해서 사용할 수 없습니다.** (예: `@GetMapping`과 `@RequestMapping`을 한 메소드에 같이 쓰면 안 됨).

중복 시 경고가 발생하고 첫 번째 어노테이션만 적용됩니다.

**예시 (클래스 레벨 및 메소드 레벨 매핑):**

```java
import org.springframework.http.HttpStatus;
import org.springframework.web.bind.annotation.*;

@RestController // @Controller + @ResponseBody
@RequestMapping("/persons") // 클래스 레벨: 이 컨트롤러의 모든 메소드는 "/persons"를 기본 경로로 가짐
public class PersonController {

    // HTTP GET /persons/{id} 요청 처리
    @GetMapping("/{id}") // 메소드 레벨: "/{id}" 경로 추가, GET 메소드 지정
    public Person getPerson(@PathVariable Long id) {
        // ... id로 Person 객체를 찾는 로직 ...
        System.out.println("Fetching person with id: " + id);
        return new Person("John Doe " + id, 30); // 예시 Person 객체 반환 (JSON으로 변환됨)
    }

    // HTTP POST /persons 요청 처리
    @PostMapping // 메소드 레벨: 경로 추가 없음 (클래스 레벨 경로 "/persons" 사용), POST 메소드 지정
    @ResponseStatus(HttpStatus.CREATED) // 응답 상태 코드를 201 Created로 설정
    public void addPerson(@RequestBody Person person) {
        // ... person 객체를 저장하는 로직 ...
        System.out.println("Adding person: " + person.getName());
    }

    // HTTP PUT /persons/{id} 요청 처리
    @PutMapping("/{id}")
    public Person updatePerson(@PathVariable Long id, @RequestBody Person personDetails) {
        System.out.println("Updating person with id: " + id + ", Details: " + personDetails.getName());
        // ... 업데이트 로직 ...
        personDetails.setId(id); // 예시
        return personDetails;
    }

    // HTTP DELETE /persons/{id} 요청 처리
    @DeleteMapping("/{id}")
    @ResponseStatus(HttpStatus.NO_CONTENT) // 응답 상태 코드를 204 No Content로 설정
    public void deletePerson(@PathVariable Long id) {
        System.out.println("Deleting person with id: " + id);
        // ... 삭제 로직 ...
    }
}

// Person.java (간단한 DTO)
class Person {
    private Long id;
    private String name;
    private int age;
    public Person(String name, int age) { this.name = name; this.age = age; }
    // Getter, Setter, 기본 생성자 등 필요
    public Long getId() { return id; }
    public void setId(Long id) { this.id = id; }
    public String getName() { return name; }
    public void setName(String name) { this.name = name; }
    public int getAge() { return age; }
    public void setAge(int age) { this.age = age; }
}
```

위 예시에서:

- `getPerson` 메소드는 `GET /persons/{어떤ID}` 요청을 처리합니다.
- `addPerson` 메소드는 `POST /persons` 요청을 처리합니다.

### 3. URI 패턴 (URI Patterns)

`@RequestMapping` 계열 어노테이션은 URL 경로를 매칭하기 위해 패턴을 사용합니다. Spring MVC는 두 가지 패턴 매칭 전략을 제공합니다:

1. **`PathPattern` (권장, Spring 6.0부터 기본값):**
  - 웹 사용에 최적화되어 있으며, URL 경로를 미리 파싱된 `PathContainer`와 비교합니다.
  - 인코딩 문제 및 경로 변수 처리에 효과적이며, 매칭 효율이 좋습니다.
  - Spring WebFlux에서는 유일한 선택지입니다.
  - `AntPathMatcher`와 유사한 패턴 구문을 지원하며, 추가적으로 경로 끝에서 0개 이상의 세그먼트를 매칭하는 `{*변수명}` (예: `/{*filepath}`) 같은 캡처링 패턴도 지원합니다.
  - `*` (여러 경로 세그먼트 매칭) 사용을 패턴의 가장 마지막에만 허용하여 모호성을 줄입니다.
2. **`AntPathMatcher` (전통적인 방식):**
  - 문자열 경로와 문자열 패턴을 비교합니다.
  - Spring 설정 파일에서 리소스 경로를 지정할 때도 사용되는 방식입니다.
  - `PathPattern`보다 효율성이 떨어지고, URL 인코딩이나 경로 구조 문제 처리에 어려움이 있을 수 있습니다.

**일반적인 URI 패턴 예시:**

- `"/resources/ima?e.png"`: `?`는 한 글자 매칭 (예: `/resources/image.png`, `/resources/imaze.png`)
- `"/resources/*.png"`: 는 한 경로 세그먼트 내에서 0개 이상의 글자 매칭 (예: `/resources/photo.png`, `/resources/.png`)
- `"/resources/**"`: `*`는 0개 이상의 경로 세그먼트 매칭 (예: `/resources/images/icon.png`, `/resources/css/style.css`, `/resources/`)
- `"/projects/{project}/versions"`: `{project}`는 경로 변수(Path Variable). 해당 세그먼트의 값을 `project`라는 이름의 변수로 캡처합니다.
- `"/projects/{project:[a-z]+}/versions"`: 경로 변수에 정규 표현식(`[a-z]+`)을 사용하여 매칭 조건을 추가할 수 있습니다. (예: `project` 변수는 소문자 알파벳으로만 구성되어야 함)

### 4. `@PathVariable`을 사용한 URI 변수 추출

URI 패턴에서 `{}`로 캡처된 변수 값은 컨트롤러 메소드의 파라미터에서 `@PathVariable` 어노테이션을 사용하여 가져올 수 있습니다.

```java
@GetMapping("/owners/{ownerId}/pets/{petId}")
public Pet findPet(@PathVariable Long ownerId, @PathVariable String petId) {
    // ownerId에는 URL의 {ownerId} 부분 값이 Long 타입으로 변환되어 들어옴
    // petId에는 URL의 {petId} 부분 값이 String 타입으로 들어옴
    System.out.println("Owner ID: " + ownerId + ", Pet ID: " + petId);
    // ... 로직 ...
    return new Pet(petId, "Buddy"); // 예시 Pet 객체
}

```

- **타입 변환:** URI 변수는 메소드 파라미터 타입에 맞게 자동으로 변환됩니다 (예: `String` -> `Long`). 변환할 수 없으면 `TypeMismatchException`이 발생합니다. (타입 변환에 대해서는 "Type Conversion" 챕터에서 자세히 다룸)
- **이름 일치:** 기본적으로 `@PathVariable`의 변수명과 URI 패턴의 변수명(`{ownerId}`)이 같아야 합니다. 만약 다르다면 `@PathVariable("ownerId") Long id` 와 같이 명시적으로 지정해야 합니다. (Java 코드를 `parameters` 플래그로 컴파일하면 변수명이 같을 경우 생략 가능)
- **클래스 및 메소드 레벨 사용:** `@PathVariable`은 클래스 레벨 `@RequestMapping`의 경로 변수와 메소드 레벨 `@RequestMapping`의 경로 변수를 모두 받을 수 있습니다.

    ```java
    @Controller
    @RequestMapping("/owners/{ownerId}") // 클래스 레벨 경로 변수
    public class OwnerController {
    
        @GetMapping("/pets/{petId}") // 메소드 레벨 경로 변수
        public Pet findPet(@PathVariable Long ownerId, @PathVariable Long petId) {
            // ownerId와 petId 모두 사용 가능
            // ...
        }
    }
    ```


### 5. 정규 표현식을 사용한 URI 변수

`{변수명:정규표현식}` 구문을 사용하여 경로 변수가 특정 정규 표현식 패턴과 일치할 때만 매핑되도록 할 수 있습니다.

```java
// 예: /files/spring-web-6.0.5.jar
@GetMapping("/{name:[a-z-]+}-{version:\\\\d\\\\.\\\\d\\\\.\\\\d}{extension:\\\\.[a-z]+}")
public void getFileInfo(@PathVariable String name,
                        @PathVariable String version,
                        @PathVariable String extension) {
    // name = "spring-web"
    // version = "6.0.5"
    // extension = ".jar"
    System.out.println("Name: " + name + ", Version: " + version + ", Extension: " + extension);
}

```

- `name:[a-z-]+`: `name` 변수는 소문자 알파벳과 하이픈으로 구성.
- `version:\\\\d\\\\.\\\\d\\\\.\\\\d`: `version` 변수는 `숫자.숫자.숫자` 형식 (정규식에서 `\\`는 이스케이프).
- `ext:\\\\.[a-z]+`: `extension` 변수는 `.`으로 시작하고 소문자 알파벳으로 구성.

### 6. URI 패턴 내의 플레이스홀더 `${...}`

`@RequestMapping`의 경로 패턴에는 `${...}` 형태의 플레이스홀더를 사용할 수 있습니다. 이 플레이스홀더는 애플리케이션 시작 시 `PropertySourcesPlaceholderConfigurer`에 의해 프로퍼티 파일( `application.properties` 등), 시스템 프로퍼티, 환경 변수 등의 값으로 치환됩니다.

```java
// application.properties
// api.base.path=/api/v1

// Controller
@RestController
@RequestMapping("${api.base.path}/items") // 시작 시 "/api/v1/items"로 치환됨
public class ItemController {
    @GetMapping("/{id}")
    public String getItem(@PathVariable String id) {
        return "Item " + id + " from " + System.getProperty("api.base.path"); // 프로퍼티 값 사용 예시
    }
}
```

- 이를 통해 외부 설정에 따라 기본 URL 경로 등을 동적으로 변경할 수 있습니다.

## Part 2: 고급 요청 매핑 - 패턴 비교, 접미사, 조건부 매칭

### 7. 패턴 비교 (Pattern Comparison) - 어떤 매핑이 우선인가?

하나의 요청 URL에 대해 여러 `@RequestMapping` 패턴이 동시에 일치할 수 있습니다.

예를 들어, `/users/all`이라는 요청에 대해 `@GetMapping("/users/*")`와 `@GetMapping("/users/all")` 두 패턴 모두 일치할 수 있습니다.

이때 Spring MVC는 **가장 구체적인(most specific)** 매핑을 선택해야 합니다.

**비교 기준 ( `PathPattern.SPECIFICITY_COMPARATOR` 또는 `AntPathMatcher.getPatternComparator()` 사용):**

Spring은 내부적으로 비교기(Comparator)를 사용하여 어떤 패턴이 더 구체적인지 결정하고, 더 구체적인 패턴을 우선적으로 사용합니다. 비교 기준은 대략 다음과 같습니다 (정확한 로직은 각 비교기 구현 참조):

1. **URI 변수 및 와일드카드 개수:**
  - 경로 변수 (`{var}`): 1점으로 계산
  - 단일 와일드카드 (): 1점으로 계산
  - 이중 와일드카드 (`*`): 2점으로 계산
  - **점수가 낮을수록 (즉, 변수나 와일드카드가 적을수록) 더 구체적입니다.**
  - 예: `/users/{id}` (1점)는 `/users/*` (1점)와 점수가 같지만, `/users/specific` (0점)보다는 덜 구체적입니다.
2. **패턴 길이:**
  - 위 점수가 동일하다면, **더 긴 패턴이 더 구체적인 것으로 간주됩니다.**
  - 예: `/users/detail`은 `/users/*`보다 깁니다 (점수는 동일하다고 가정 시).
3. **경로 변수 vs 와일드카드 (점수와 길이가 모두 같을 때):**
  - 와일드카드보다 **경로 변수를 더 많이 가진 패턴이 더 구체적인 것으로 간주됩니다.**

**특별 규칙:**

- **기본 매핑 패턴 (`/**`):** 이 패턴은 점수 계산에서 제외되며 항상 가장 마지막 순위로 정렬됩니다. (가장 덜 구체적)
- **접두사 패턴 (예: `/public/**`):** `*`를 포함하지 않는 다른 패턴들보다 덜 구체적인 것으로 간주됩니다.

**예시:** `/products/123` 요청에 대해 다음 매핑들이 있다면:

1. `@GetMapping("/products/{id}")`
2. `@GetMapping("/products/*")`
3. `@GetMapping("/**")`

Spring MVC는 가장 구체적인 `1번: /products/{id}`를 선택합니다.

### 8. 접미사 매칭 (Suffix Match) - 더 이상 기본이 아님!

- **과거 (Spring 5.3 이전):** 기본적으로 `.*` 접미사 패턴 매칭이 활성화되어, 예를 들어 `@GetMapping("/person")`으로 매핑된 컨트롤러는 `/person.json`, `/person.xml`, `/person.pdf`와 같은 요청도 암묵적으로 처리했습니다. 이때 경로 확장자(`.json`, `.xml`)는 요청된 응답의 콘텐츠 타입을 해석하는 데 사용되기도 했습니다 (컨텐츠 협상).
- **현재 (Spring 5.3부터 기본 비활성화, 6.0부터 관련 옵션 폐기):**
  - **기본적으로 접미사 패턴 매칭을 더 이상 수행하지 않습니다.** 즉, `/person`은 `/person` 요청만 처리하고, `/person.json`은 별도의 매핑이 없으면 처리하지 않습니다.
  - **이유:**
    1. **`Accept` 헤더 사용 권장:** 콘텐츠 협상은 클라이언트의 `Accept` 요청 헤더를 사용하는 것이 표준적이고 더 나은 방법입니다. 과거에는 브라우저의 `Accept` 헤더 해석이 일관되지 않아 경로 확장자가 필요했지만, 현재는 그렇지 않습니다.
    2. **모호성 및 보안 문제:** 경로 확장자 사용은 URI 변수, 경로 파라미터, URI 인코딩 등과 함께 사용될 때 모호성을 유발할 수 있습니다. 또한 URL 기반 보안 규칙 적용을 더 어렵게 만들 수 있습니다.
    3. **RFD(Reflected File Download) 공격 위험:** 경로 확장을 통해 응답 콘텐츠 타입을 변경할 수 있는 경우, 악의적인 입력이 응답에 반영되어 브라우저가 이를 실행 가능한 파일로 다운로드하게 만드는 RFD 공격에 취약해질 수 있습니다.
- **과거 버전에서 비활성화 방법 (5.3 이전):**
  - `PathMatchConfigurer`에서 `useSuffixPatternMatching(false)` 설정.
  - `ContentNegotiationConfigurer`에서 `favorPathExtension(false)` 설정.
- **현재 대안:**
  - `Accept` 헤더를 사용한 콘텐츠 협상.
  - URL에 명시적으로 타입을 나타내고 싶다면, 경로 확장자 대신 **쿼리 파라미터** (예: `/person?format=json`)를 사용하는 것이 더 안전한 대안입니다.
  - 만약 파일 확장자를 꼭 사용해야 한다면, `ContentNegotiationConfigurer`의 `mediaTypes` 속성을 통해 명시적으로 등록된 확장자 목록으로 제한하는 것이 좋습니다.
- **RFD 공격 방어 (Spring MVC의 노력):**
  - Spring MVC는 RFD 공격을 방지하기 위해, 응답 본문 렌더링 전에 `Content-Disposition: inline; filename=f.txt`와 같은 헤더를 추가하여 안전한 다운로드 파일 이름을 제안하려고 시도합니다. (URL 경로에 허용되지 않거나 명시적으로 등록되지 않은 파일 확장자가 포함된 경우)

### 9. 소비 가능한 미디어 타입 (Consumable Media Types) - `consumes` 속성

컨트롤러 메소드가 처리할 수 있는 **요청 본문의 `Content-Type`을 기준으로 매핑을 제한**합니다.

- **사용법:** `@RequestMapping` (또는 그 변형) 어노테이션의 `consumes` 속성에 미디어 타입을 지정합니다.

    ```java
    // 이 메소드는 요청의 Content-Type이 "application/json"일 때만 호출됨
    @PostMapping(path = "/pets", consumes = "application/json")
    public void addPet(@RequestBody Pet pet) {
        // ... (pet 객체는 JSON 요청 본문으로부터 변환됨)
    }
    ```

- **여러 타입 지정 가능:** `consumes = {"application/json", "application/xml"}`
- **부정 표현식 지원:** `consumes = "!text/plain"` ( `text/plain`을 제외한 모든 타입)
- **클래스 레벨 vs 메소드 레벨:**
  - 클래스 레벨에 `consumes`를 선언하면 하위 모든 메소드에 적용됩니다.
  - 하지만 메소드 레벨에 `consumes`를 선언하면, 클래스 레벨 설정을 **확장하는 것이 아니라 재정의(override)**합니다.
- **`MediaType` 상수 사용:** `MediaType.APPLICATION_JSON_VALUE` (문자열 `"application/json"`)와 같이 상수를 사용하면 오타를 줄일 수 있습니다.

### 10. 생산 가능한 미디어 타입 (Producible Media Types) - `produces` 속성

컨트롤러 메소드가 생성할 수 있는 **응답 본문의 `Content-Type`을 기준으로 매핑을 제한**합니다. 이는 클라이언트가 보낸 **`Accept` 요청 헤더의 값과 비교**됩니다.

- **사용법:** `@RequestMapping` (또는 그 변형) 어노테이션의 `produces` 속성에 미디어 타입을 지정합니다.

    ```java
    // 이 메소드는 클라이언트가 "application/json" 타입을 Accept 할 때 호출될 가능성이 높음
    // (실제 선택은 ContentNegotiatingViewResolver나 HttpMessageConverter 선택 로직에 따름)
    @GetMapping(path = "/pets/{petId}", produces = "application/json")
    @ResponseBody // 이 어노테이션이 있어야 produces가 의미를 가짐 (또는 ResponseEntity 반환)
    public Pet getPet(@PathVariable String petId) {
        // ... Pet 객체를 반환하면 JSON으로 변환되어 응답 ...
        return new Pet(petId, "Fluffy");
    }
    ```

- **문자 집합 지정 가능:** `produces = "application/json;charset=UTF-8"`
- **부정 표현식 지원:** `produces = "!text/plain"`
- **클래스 레벨 vs 메소드 레벨:** `consumes`와 마찬가지로 메소드 레벨이 클래스 레벨을 재정의합니다.
- **`MediaType` 상수 사용:** `MediaType.APPLICATION_JSON_VALUE` 등.

### 11. 파라미터, 헤더 조건 매핑 - `params` 및 `headers` 속성

요청에 **특정 파라미터나 헤더가 존재하거나 특정 값을 가질 때만** 컨트롤러 메소드가 매핑되도록 조건을 걸 수 있습니다.

- **`params` 속성:**
  - `params = "myParam"`: `myParam`이라는 요청 파라미터가 존재해야 함.
  - `params = "!myParam"`: `myParam`이라는 요청 파라미터가 존재하지 않아야 함.
  - `params = "myParam=myValue"`: `myParam` 파라미터의 값이 `myValue`여야 함.
  - `params = {"myParam=myValue", "anotherParam"}`: 여러 조건 조합 가능.

    ```java
    @GetMapping(path = "/pets/{petId}", params = "action=display")
    public void findPetForDisplay(@PathVariable String petId) {
        // 요청 URL에 ?action=display 파라미터가 있을 때만 이 메소드 호출
    }
    ```

- **`headers` 속성:**
  - `params`와 유사한 방식으로 특정 요청 헤더의 존재 유무나 값을 검사합니다.
  - `headers = "myHeader"`
  - `headers = "!myHeader"`
  - `headers = "myHeader=myValue"`

    ```java
    @GetMapping(path = "/pets/{petId}", headers = "X-Custom-Header=SpecialValue")
    public void findPetWithCustomHeader(@PathVariable String petId) {
        // 요청 헤더에 X-Custom-Header: SpecialValue 가 있을 때만 이 메소드 호출
    }
    ```

- **주의:** `headers` 조건으로 `Content-Type`이나 `Accept` 헤더를 매칭할 수도 있지만, 이 경우에는 각각 `consumes`와 `produces` 속성을 사용하는 것이 더 명확하고 권장되는 방법입니다.

---

## Part 3: 특수 HTTP 메소드 처리, 커스텀 매핑, `@HttpExchange`

### 12. HTTP `HEAD`, `OPTIONS` 메소드 처리

Spring MVC는 HTTP `HEAD`와 `OPTIONS` 메소드에 대해 다음과 같은 기본 처리 방식을 제공합니다.

- **HTTP `HEAD`:**
  - `@GetMapping` (또는 `@RequestMapping(method = HttpMethod.GET)`)으로 매핑된 핸들러 메소드는 **자동으로 HTTP `HEAD` 요청도 처리**합니다. 개발자가 별도로 `HEAD` 요청을 위한 핸들러를 만들 필요가 없습니다.
  - **동작 방식:** `HEAD` 요청은 `GET` 요청과 동일하게 처리되지만, **응답 본문(body)은 전송되지 않습니다.** Spring MVC는 내부적으로 `jakarta.servlet.http.HttpServlet`의 응답 래퍼를 사용하여, `Content-Length` 헤더는 `GET` 요청 시 전송될 바이트 수로 설정하지만 실제 응답 본문은 쓰지 않도록 보장합니다.
  - 클라이언트는 `HEAD` 요청을 통해 리소스의 메타데이터(헤더 정보, 크기 등)만 확인할 수 있습니다.
- **HTTP `OPTIONS`:**
  - 기본적으로, `OPTIONS` 요청이 들어오면 Spring MVC는 해당 URL 패턴과 일치하는 모든 `@RequestMapping` 메소드들을 찾아, 그들이 지원하는 HTTP 메소드 목록을 `Allow` 응답 헤더에 담아 반환합니다.
    - 예: `/items` URL에 `@GetMapping`과 `@PostMapping`이 있다면, `OPTIONS /items` 요청에 대해 `Allow: GET, HEAD, POST, OPTIONS` 와 같은 응답을 보냅니다. (`HEAD`는 `GET`에 의해 자동으로 포함, `OPTIONS` 자신도 포함)
  - 만약 `@RequestMapping`에 `method` 속성이 지정되지 않았다면 (기본적으로 모든 HTTP 메소드와 매칭), `Allow` 헤더는 `GET, HEAD, POST, PUT, PATCH, DELETE, OPTIONS`로 설정됩니다.
  - **권장 사항:** 컨트롤러 메소드는 항상 지원하는 HTTP 메소드를 명시적으로 선언하는 것이 좋습니다 (예: `@GetMapping`, `@PostMapping` 사용).
  - 일반적으로 `@RequestMapping`을 `OPTIONS` 메소드에 명시적으로 매핑할 필요는 없습니다. Spring MVC가 알아서 잘 처리해줍니다.

### 13. 커스텀 어노테이션 (Custom Annotations) for Request Mapping

Spring MVC는 **합성 어노테이션(Composed Annotations)**을 사용하여 요청 매핑을 더욱 특수화하고 가독성을 높일 수 있도록 지원합니다.

- **합성 어노테이션이란?** 여러 어노테이션을 조합하여 새로운 어노테이션을 만드는 것입니다. 이때 메타 어노테이션(`@interface`로 정의된 어노테이션에 붙는 어노테이션)으로 `@RequestMapping` 등을 사용합니다.
- **예시:** `@GetMapping`, `@PostMapping` 등이 바로 `@RequestMapping`을 메타 어노테이션으로 사용하여 만들어진 합성 어노테이션입니다.

    ```java
    // @GetMapping 어노테이션의 실제 정의 (유사한 형태)
    @Target(ElementType.METHOD)
    @Retention(RetentionPolicy.RUNTIME)
    @Documented
    @RequestMapping(method = RequestMethod.GET) // @RequestMapping을 메타 어노테이션으로 사용
    public @interface GetMapping {
        @AliasFor(annotation = RequestMapping.class)
        String name() default "";
    
        @AliasFor(annotation = RequestMapping.class)
        String[] value() default {}; // path의 별칭
    
        @AliasFor(annotation = RequestMapping.class)
        String[] path() default {};
    
        // ... 기타 RequestMapping 속성들 ...
    }
    
    ```

- **장점:**
  - 자주 사용되는 `@RequestMapping` 속성 조합을 미리 정의하여 코드를 간결하게 만들 수 있습니다.
  - 어노테이션 이름 자체로 의도를 더 명확하게 표현할 수 있습니다. (예: `@GetJsonResource` 같이 특정 목적의 어노테이션 생성 가능)
- **주의:** 하나의 요소(클래스, 메소드)에 `@RequestMapping`과 이를 합성한 다른 어노테이션을 중복해서 사용하면 안 됩니다. (예: `@GetMapping`과 `@RequestMapping`을 한 메소드에 같이 사용)

**커스텀 요청 매칭 속성 (고급):**
더 나아가, 개발자가 자신만의 요청 매칭 조건을 만들고 이를 어노테이션 속성으로 사용하고 싶다면, `RequestMappingHandlerMapping`을 상속받아 `getCustomMethodCondition` 메소드를 오버라이드하는 고급 기능을 사용할 수도 있습니다. (매우 드문 경우)

### 14. 명시적 핸들러 등록 (Explicit Registrations)

일반적으로 `@Controller`와 `@RequestMapping`을 사용하는 선언적인 방식 외에, **프로그래밍 방식으로 핸들러 메소드를 직접 등록**할 수도 있습니다.

- **사용 시나리오:**
  - 동적으로 핸들러를 등록해야 하는 경우 (예: 플러그인 시스템)
  - 하나의 핸들러 객체 내의 동일한 메소드를 서로 다른 URL이나 조건으로 여러 번 등록해야 하는 고급 사례.
- **방법:** `RequestMappingHandlerMapping` 빈을 주입받아 `registerMapping` 메소드를 사용합니다.

    ```java
    // MyConfig.java (Java Config)
    import org.springframework.beans.factory.annotation.Autowired;
    import org.springframework.context.annotation.Configuration;
    import org.springframework.web.bind.annotation.RequestMethod;
    import org.springframework.web.servlet.mvc.method.RequestMappingInfo;
    import org.springframework.web.servlet.mvc.method.annotation.RequestMappingHandlerMapping;
    import java.lang.reflect.Method;
    
    // UserHandler.java (핸들러 로직을 담은 일반 클래스, @Controller가 아닐 수도 있음)
    // 이 예시에서는 간단히 일반 빈으로 가정
    // @Component 또는 다른 방식으로 빈으로 등록되어 있어야 함
    public class UserHandler {
        public User getUser(Long id) {
            System.out.println("UserHandler.getUser called with id: " + id);
            return new User("UserFromHandler", id.intValue() + 20);
        }
    }
    
    @Configuration
    public class MyConfig {
    
        @Autowired
        public void registerCustomHandlerMapping(
                RequestMappingHandlerMapping requestMappingHandlerMapping, // 주입받을 HandlerMapping
                UserHandler userHandler                                   // 주입받을 핸들러 객체
        ) throws NoSuchMethodException {
    
            // 1. 요청 매핑 정보 (URL, HTTP 메소드 등) 생성
            RequestMappingInfo requestMappingInfo = RequestMappingInfo
                    .paths("/users/handler/{id}")       // URL 패턴
                    .methods(RequestMethod.GET)        // HTTP GET 메소드
                    // .params("myParam=myValue")      // 기타 조건 추가 가능
                    .build();
    
            // 2. 실제 실행될 핸들러 객체와 메소드 정보 가져오기
            // UserHandler 클래스의 getUser 메소드 (파라미터 타입 Long)
            Method handlerMethod = UserHandler.class.getMethod("getUser", Long.class);
    
            // 3. HandlerMapping에 등록
            // requestMappingInfo 조건에 맞는 요청이 오면,
            // userHandler 객체의 handlerMethod를 실행하라.
            requestMappingHandlerMapping.registerMapping(requestMappingInfo, userHandler, handlerMethod);
    
            System.out.println("Custom handler mapping registered for /users/handler/{id}");
        }
    }
    
    ```

  - 위 코드가 실행되면, `/users/handler/{어떤ID}` 형태의 GET 요청이 올 경우 `UserHandler`의 `getUser` 메소드가 호출됩니다.
  - 이 방식은 매우 유연하지만, 대부분의 경우 어노테이션 기반 매핑이 더 간결하고 유지보수하기 좋습니다.

### 15. `@HttpExchange` 어노테이션 (Spring 6.0+)

`@HttpExchange` 어노테이션은 주로 선언적인 HTTP 클라이언트( Declarative HTTP Client, 예: `RestClient`와 함께 사용될 프록시 생성)를 만들기 위한 것이지만, **서버 측 컨트롤러에서도 `@RequestMapping`의 대안으로 사용될 수 있습니다.**

- **배경:** HTTP 인터페이스(어노테이션이 붙은 Java 인터페이스)는 클라이언트와 서버 간의 계약을 정의합니다. 이 인터페이스를 클라이언트 측에서는 API 호출 코드를 자동 생성하는 데 사용하고, 서버 측에서는 해당 API를 구현하는 컨트롤러의 매핑 정의로 사용할 수 있습니다. (주로 내부 API나 Spring Cloud 환경에서 사용 고려)
- **장점 (서버 측에서):** API 계약(인터페이스)과 구현(컨트롤러)을 명확히 분리하고, 인터페이스를 통해 API 스펙을 공유할 수 있습니다.
- **단점 (서버 측에서):** 클라이언트와 서버 간의 결합도를 높일 수 있어, 특히 외부로 공개되는 Public API에는 적합하지 않을 수 있습니다.

**예시:**

```java
// PersonService.java (HTTP 인터페이스 - 계약)
import org.springframework.web.bind.annotation.PathVariable;
import org.springframework.web.bind.annotation.RequestBody;
import org.springframework.web.service.annotation.GetExchange; // 클라이언트용 어노테이션과 유사
import org.springframework.web.service.annotation.HttpExchange;
import org.springframework.web.service.annotation.PostExchange;

@HttpExchange("/persons-api") // 기본 경로 (서버/클라이언트 공통)
public interface PersonService {

    @GetExchange("/{id}") // GET /persons-api/{id}
    Person getPersonById(@PathVariable Long id);

    @PostExchange // POST /persons-api
    void addNewPerson(@RequestBody Person person);
}

// PersonController.java (서버 측 구현)
import org.springframework.http.HttpStatus;
import org.springframework.web.bind.annotation.PathVariable; // Spring MVC 어노테이션
import org.springframework.web.bind.annotation.RequestBody;
import org.springframework.web.bind.annotation.ResponseStatus;
import org.springframework.web.bind.annotation.RestController;

@RestController // 이 컨트롤러가 PersonService 인터페이스를 구현
public class PersonControllerWithHttpExchange implements PersonService { // 인터페이스 구현

    @Override // PersonService의 getPersonById 구현
    public Person getPersonById(@PathVariable Long id) { // @PathVariable 등은 Spring MVC 것 사용
        System.out.println("Fetching person via @HttpExchange with id: " + id);
        return new Person("PersonFromHttpExchange " + id, 25);
    }

    @Override // PersonService의 addNewPerson 구현
    @ResponseStatus(HttpStatus.CREATED) // 응답 상태 코드 설정
    public void addNewPerson(@RequestBody Person person) { // @RequestBody 등은 Spring MVC 것 사용
        System.out.println("Adding person via @HttpExchange: " + person.getName());
    }
}

```

**`@HttpExchange` vs `@RequestMapping` (서버 측에서):**

- `@RequestMapping`: 경로 패턴, HTTP 메소드, 파라미터 조건 등 매우 다양한 조건으로 여러 요청에 유연하게 매핑 가능.
- `@HttpExchange` (인터페이스에 사용 시): 주로 단일 엔드포인트에 대한 구체적인 HTTP 메소드, 경로, 콘텐츠 타입을 선언. (더 명시적이지만 덜 유연)
- **메소드 파라미터 및 반환 값:** `@HttpExchange`는 `@RequestMapping`이 지원하는 파라미터의 하위 집합을 지원합니다 (주로 서버 측에 특화된 파라미터 타입 제외).
- `headers()` 파라미터: `@HttpExchange`도 헤더 조건을 지정할 수 있으며, 서버 측에서는 `@RequestMapping`과 동일한 전체 구문을 지원합니다.

`@HttpExchange`를 서버 측 컨트롤러에서 사용하는 것은 아직 일반적이지는 않지만, 특정 아키텍처(예: Spring Cloud의 선언적 Feign 클라이언트와 유사한 내부 API 구현)에서는 유용할 수 있습니다.

---
