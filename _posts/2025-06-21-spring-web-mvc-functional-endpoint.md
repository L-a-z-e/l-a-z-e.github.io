---
title: Spring Web MVC - Functional Endpoints
description: 
author: laze
date: 2025-06-21 00:00:01 +0900
categories: [Dev, SpringWebMVC]
tags: [SpringWebMVC]
---
### Functional Endpoints (함수형 엔드포인트)

스프링 Web MVC는 함수를 사용하여 요청을 라우팅하고 처리하며, 불변성(immutability)을 위해 계약(contracts)이 설계된 경량의 함수형 프로그래밍 모델인 `WebMvc.fn`을 포함합니다.

이는 어노테이션 기반 프로그래밍 모델의 대안이지만, 그 외에는 동일한 `DispatcherServlet`에서 실행됩니다.

### Overview (개요)

`WebMvc.fn`에서 HTTP 요청은 `HandlerFunction`으로 처리됩니다: 이 함수는 `ServerRequest`를 받아 `ServerResponse`를 반환합니다.

요청과 응답 객체 모두 HTTP 요청 및 응답에 대한 JDK 8 친화적인 접근을 제공하는 불변 계약을 가지고 있습니다.

`HandlerFunction`은 어노테이션 기반 프로그래밍 모델의 `@RequestMapping` 메소드 본문에 해당합니다.

들어오는 요청은 `RouterFunction`을 통해 핸들러 함수로 라우팅됩니다: 이 함수는 `ServerRequest`를 받아 선택적인 `HandlerFunction`(즉, `Optional<HandlerFunction>`)을 반환합니다.

라우터 함수가 매칭되면 핸들러 함수가 반환되고, 그렇지 않으면 빈 `Optional`이 반환됩니다.

`RouterFunction`은 `@RequestMapping` 어노테이션에 해당하지만, 라우터 함수는 데이터뿐만 아니라 동작(behavior)도 제공한다는 주요 차이점이 있습니다.

`RouterFunctions.route()`는 다음 예제와 같이 라우터 생성을 용이하게 하는 라우터 빌더를 제공합니다:

**Java**

```java
import static org.springframework.http.MediaType.APPLICATION_JSON;
import static org.springframework.web.servlet.function.RequestPredicates.*;
import static org.springframework.web.servlet.function.RouterFunctions.route;

PersonRepository repository = ... ; // 레포지토리 가정
PersonHandler handler = new PersonHandler(repository); // 핸들러 클래스 인스턴스

RouterFunction<ServerResponse> route = route() // 1. route() 빌더로 라우터 생성 시작
	.GET("/person/{id}", accept(APPLICATION_JSON), handler::getPerson) // GET /person/{id}, JSON 요청 시 handler.getPerson 호출
	.GET("/person", accept(APPLICATION_JSON), handler::listPeople)   // GET /person, JSON 요청 시 handler.listPeople 호출
	.POST("/person", handler::createPerson)                        // POST /person 요청 시 handler.createPerson 호출
	.build(); // 라우터 빌드 완료

public class PersonHandler { // 요청을 처리하는 핸들러 클래스

	// ... (레포지토리 주입 등)

	public ServerResponse listPeople(ServerRequest request) {
		// ... (모든 사람 목록 반환 로직)
	}

	public ServerResponse createPerson(ServerRequest request) {
		// ... (사람 생성 로직)
	}

	public ServerResponse getPerson(ServerRequest request) {
		// ... (특정 ID의 사람 반환 로직)
	}
}
```

1. `route()`를 사용하여 라우터 생성하기.

`RouterFunction`을 빈으로 등록하면 (예: `@Configuration` 클래스에서 노출하여), 서버 실행(Running a Server) 섹션에서 설명한 대로 서블릿에 의해 자동 감지됩니다.

### HandlerFunction (핸들러 함수)

`ServerRequest`와 `ServerResponse`는 헤더, 본문, 메소드, 상태 코드를 포함하여 HTTP 요청 및 응답에 대한 JDK 8 친화적인 접근을 제공하는 불변 인터페이스입니다.

### ServerRequest

`ServerRequest`는 HTTP 메소드, URI, 헤더, 쿼리 파라미터에 대한 접근을 제공하며, 본문에 대한 접근은 `body` 메소드를 통해 제공됩니다.

다음 예제는 요청 본문을 `String`으로 추출합니다:

**Java**

```java
String string = request.body(String.class);
```

다음 예제는 본문을 `List<Person>`으로 추출하며, `Person` 객체는 JSON 또는 XML과 같은 직렬화된 형태에서 디코딩됩니다:

**Java**

```java
List<Person> people = request.body(new ParameterizedTypeReference<List<Person>>() {});
```

다음 예제는 파라미터에 접근하는 방법을 보여줍니다:

**Java**

```java
MultiValueMap<String, String> params = request.params();
```

### ServerResponse

`ServerResponse`는 HTTP 응답에 대한 접근을 제공하며, 불변이므로 빌드 메소드를 사용하여 생성할 수 있습니다. 빌더를 사용하여 응답 상태를 설정하거나, 응답 헤더를 추가하거나, 본문을 제공할 수 있습니다. 다음 예제는 JSON 내용을 가진 200 (OK) 응답을 생성합니다:

**Java**

```java
Person person = ... ;
ServerResponse.ok().contentType(MediaType.APPLICATION_JSON).body(person);
```

다음 예제는 `Location` 헤더와 본문 없는 201 (CREATED) 응답을 빌드하는 방법을 보여줍니다:

**Java**

```java
URI location = ... ;
ServerResponse.created(location).build();
```

`CompletableFuture`, `Publisher`, 또는 `ReactiveAdapterRegistry`에서 지원하는 다른 유형의 형태로 비동기 결과를 본문으로 사용할 수도 있습니다. 예를 들면:

**Java**

```java
Mono<Person> person = webClient.get().retrieve().bodyToMono(Person.class); // WebClient (Reactive) 사용 예시
ServerResponse.ok().contentType(MediaType.APPLICATION_JSON).body(person);
```

만약 본문뿐만 아니라 상태나 헤더도 비동기 타입에 기반한다면, `ServerResponse`의 정적 `async` 메소드를 사용할 수 있으며, 이는 `CompletableFuture<ServerResponse>`, `Publisher<ServerResponse>`, 또는 `ReactiveAdapterRegistry`에서 지원하는 다른 비동기 타입을 받습니다. 예를 들면:

**Java**

```java
Mono<ServerResponse> asyncResponse = webClient.get().retrieve().bodyToMono(Person.class)
  .map(p -> ServerResponse.ok().header("Name", p.name()).body(p)); // Person 객체에서 이름 추출
ServerResponse.async(asyncResponse);
```

서버-전송 이벤트(Server-Sent Events)는 `ServerResponse`의 정적 `sse` 메소드를 통해 제공될 수 있습니다. 해당 메소드에서 제공하는 빌더를 사용하면 `String` 또는 다른 객체를 JSON으로 보낼 수 있습니다. 예를 들면:

**Java**

```java
public RouterFunction<ServerResponse> sse() {
	return route(GET("/sse"), request -> ServerResponse.sse(sseBuilder -> {
				// sseBuilder 객체를 어딘가에 저장합니다..
			}));
}

// 다른 스레드에서, String 전송
// sseBuilder.send("Hello world"); // 실제로는 저장된 sseBuilder 참조를 사용해야 함

// 또는 객체, 이는 JSON으로 변환됨
// Person person = ... ;
// sseBuilder.send(person);

// 다른 메소드를 사용하여 이벤트 사용자 정의
// sseBuilder.id("42")
// 		.event("sse event")
// 		.data(person);

// 그리고 특정 시점에 완료
// sseBuilder.complete();
```

### Handler Classes (핸들러 클래스)

다음 예제와 같이 핸들러 함수를 람다로 작성할 수 있습니다:

**Java**

```java
HandlerFunction<ServerResponse> helloWorld =
  request -> ServerResponse.ok().body("Hello World");
```

이것은 편리하지만, 애플리케이션에서는 여러 함수가 필요하며 여러 인라인 람다는 지저분해질 수 있습니다. 따라서 관련된 핸들러 함수들을 핸들러 클래스로 그룹화하는 것이 유용하며, 이는 어노테이션 기반 애플리케이션의 `@Controller`와 유사한 역할을 합니다. 예를 들어, 다음 클래스는 반응형 `Person` 레포지토리를 노출합니다 (예제는 비반응형 코드로 보이지만, 개념 설명용):

**Java**

```java
import static org.springframework.http.MediaType.APPLICATION_JSON;
import static org.springframework.web.servlet.function.ServerResponse.ok; // RouterFunctions.route()와 ServerResponse.ok()는 다른 패키지일 수 있음

public class PersonHandler {

	private final PersonRepository repository;

	public PersonHandler(PersonRepository repository) {
		this.repository = repository;
	}

	// 1. listPeople은 레포지토리에서 찾은 모든 Person 객체를 JSON으로 반환하는 핸들러 함수입니다.
	public ServerResponse listPeople(ServerRequest request) {
		List<Person> people = repository.allPeople();
		return ok().contentType(APPLICATION_JSON).body(people);
	}

	// 2. createPerson은 요청 본문에 포함된 새로운 Person을 저장하는 핸들러 함수입니다.
	public ServerResponse createPerson(ServerRequest request) throws Exception { // body() 메소드는 Exception을 던질 수 있음
		Person person = request.body(Person.class);
		repository.savePerson(person);
		return ok().build(); // 본문 없는 OK 응답
	}

	// 3. getPerson은 id 경로 변수로 식별되는 단일 사람을 반환하는 핸들러 함수입니다.
	//    레포지토리에서 해당 Person을 검색하고, 찾으면 JSON 응답을 생성합니다.
	//    찾지 못하면 404 Not Found 응답을 반환합니다.
	public ServerResponse getPerson(ServerRequest request) {
		int personId = Integer.parseInt(request.pathVariable("id"));
		Person person = repository.getPerson(personId);
		if (person != null) {
			return ok().contentType(APPLICATION_JSON).body(person);
		}
		else {
			return ServerResponse.notFound().build();
		}
	}
}
```

1. `listPeople`은 레포지토리에서 찾은 모든 `Person` 객체를 JSON으로 반환하는 핸들러 함수입니다.
2. `createPerson`은 요청 본문에 포함된 새로운 `Person`을 저장하는 핸들러 함수입니다.
3. `getPerson`은 `id` 경로 변수로 식별되는 단일 사람을 반환하는 핸들러 함수입니다. 레포지토리에서 해당 `Person`을 검색하고, 찾으면 JSON 응답을 생성합니다. 찾지 못하면 404 Not Found 응답을 반환합니다.

### Validation (검증)

함수형 엔드포인트는 요청 본문에 검증을 적용하기 위해 스프링의 검증 기능을 사용할 수 있습니다. 예를 들어, `Person`에 대한 사용자 정의 스프링 `Validator` 구현이 주어졌다고 가정합니다:

**Java**

```java
public class PersonHandler {

	private final Validator validator = new PersonValidator(); // 1. Validator 인스턴스 생성

	// ...

	public ServerResponse createPerson(ServerRequest request) throws Exception {
		Person person = request.body(Person.class);
		validate(person); // 2. 검증 적용
		repository.savePerson(person);
		return ok().build();
	}

	private void validate(Person person) {
		Errors errors = new BeanPropertyBindingResult(person, "person");
		validator.validate(person, errors);
		if (errors.hasErrors()) {
			throw new ServerWebInputException(errors.toString()); // 3. 400 응답을 위한 예외 발생
		}
	}
}
```

1. `Validator` 인스턴스 생성.
2. 검증 적용.
3. 400 응답을 위한 예외 발생.

핸들러는 또한 `LocalValidatorFactoryBean`에 기반한 전역 `Validator` 인스턴스를 생성하고 주입하여 표준 빈 검증 API (JSR-303)를 사용할 수 있습니다. 스프링 검증(Spring Validation)을 참고하세요.

### RouterFunction (라우터 함수)

라우터 함수는 요청을 해당 `HandlerFunction`으로 라우팅하는 데 사용됩니다. 일반적으로 라우터 함수를 직접 작성하는 대신 `RouterFunctions` 유틸리티 클래스의 메소드를 사용하여 생성합니다. `RouterFunctions.route()` (파라미터 없음)는 라우터 함수 생성을 위한 플루언트 빌더(fluent builder)를 제공하는 반면, `RouterFunctions.route(RequestPredicate, HandlerFunction)`는 라우터를 직접 생성하는 방법을 제공합니다.

일반적으로 `route()` 빌더를 사용하는 것이 권장되며, 이는 찾기 어려운 정적 임포트 없이 일반적인 매핑 시나리오에 대한 편리한 단축키를 제공합니다. 예를 들어, 라우터 함수 빌더는 GET 요청에 대한 매핑을 생성하기 위해 `GET(String, HandlerFunction)` 메소드를 제공하고, POST 요청에는 `POST(String, HandlerFunction)`를 제공합니다.

HTTP 메소드 기반 매핑 외에도, 라우트 빌더는 요청에 매핑할 때 추가적인 술어(predicates)를 도입하는 방법을 제공합니다. 각 HTTP 메소드에는 `RequestPredicate`를 파라미터로 받는 오버로드된 변형이 있으며, 이를 통해 추가적인 제약 조건을 표현할 수 있습니다.

### Predicates (술어)

자신만의 `RequestPredicate`를 작성할 수 있지만, `RequestPredicates` 유틸리티 클래스는 요청 경로, HTTP 메소드, 컨텐츠 타입 등을 기반으로 일반적으로 사용되는 구현을 제공합니다. 다음 예제는 `Accept` 헤더를 기반으로 제약 조건을 생성하기 위해 요청 술어를 사용합니다:

**Java**

```java
RouterFunction<ServerResponse> route = RouterFunctions.route()
	.GET("/hello-world", accept(MediaType.TEXT_PLAIN), // Accept 헤더가 text/plain인 경우
		request -> ServerResponse.ok().body("Hello World")).build();
```

다음을 사용하여 여러 요청 술어를 함께 구성할 수 있습니다:

- `RequestPredicate.and(RequestPredicate)` – 둘 다 일치해야 함.
- `RequestPredicate.or(RequestPredicate)` – 둘 중 하나만 일치하면 됨.

`RequestPredicates`의 많은 술어들은 구성되어 있습니다. 예를 들어, `RequestPredicates.GET(String)`는 `RequestPredicates.method(HttpMethod)`와 `RequestPredicates.path(String)`로 구성됩니다. 위 예제는 빌더가 내부적으로 `RequestPredicates.GET`을 사용하고 이를 `accept` 술어와 구성하므로 두 개의 요청 술어를 사용합니다.

### Routes (경로)

라우터 함수는 순서대로 평가됩니다: 첫 번째 경로가 일치하지 않으면 두 번째 경로가 평가되는 식입니다. 따라서 일반적인 경로보다 더 구체적인 경로를 먼저 선언하는 것이 합리적입니다. 이는 나중에 설명할 라우터 함수를 스프링 빈으로 등록할 때도 중요합니다. 이 동작은 "가장 구체적인" 컨트롤러 메소드가 자동으로 선택되는 어노테이션 기반 프로그래밍 모델과는 다릅니다.

라우터 함수 빌더를 사용할 때, 정의된 모든 경로는 `build()`에서 반환되는 하나의 `RouterFunction`으로 구성됩니다. 여러 라우터 함수를 함께 구성하는 다른 방법도 있습니다:

- `RouterFunctions.route()` 빌더의 `add(RouterFunction)`
- `RouterFunction.and(RouterFunction)`
- `RouterFunction.andRoute(RequestPredicate, HandlerFunction)` – 중첩된 `RouterFunctions.route()`와 함께 `RouterFunction.and()`의 단축키.

다음 예제는 네 개의 경로 구성을 보여줍니다:

**Java**

```java
import static org.springframework.http.MediaType.APPLICATION_JSON;
import static org.springframework.web.servlet.function.RequestPredicates.*; // RequestPredicates 사용

PersonRepository repository = ... ;
PersonHandler handler = new PersonHandler(repository);

RouterFunction<ServerResponse> otherRoute = ... ; // 다른 곳에서 생성된 라우터 함수

RouterFunction<ServerResponse> route = route()
	.GET("/person/{id}", accept(APPLICATION_JSON), handler::getPerson) // 1
	.GET("/person", accept(APPLICATION_JSON), handler::listPeople)   // 2
	.POST("/person", handler::createPerson)                         // 3
	.add(otherRoute)                                               // 4
	.build();
```

1. `Accept` 헤더가 JSON과 일치하는 `GET /person/{id}`는 `PersonHandler.getPerson`으로 라우팅됩니다.
2. `Accept` 헤더가 JSON과 일치하는 `GET /person`은 `PersonHandler.listPeople`로 라우팅됩니다.
3. 추가적인 술어 없는 `POST /person`은 `PersonHandler.createPerson`으로 매핑됩니다.
4. `otherRoute`는 다른 곳에서 생성된 라우터 함수이며, 빌드된 경로에 추가됩니다.

### Nested Routes (중첩 경로)

라우터 함수 그룹이 공유된 술어, 예를 들어 공유된 경로를 갖는 것은 일반적입니다. 위 예제에서 공유된 술어는 세 개의 경로에서 사용된 `/person`과 일치하는 경로 술어입니다.

어노테이션을 사용할 때는 `/person`에 매핑되는 타입 레벨 `@RequestMapping` 어노테이션을 사용하여 이러한 중복을 제거합니다.

`WebMvc.fn`에서는 라우터 함수 빌더의 `path` 메소드를 통해 경로 술어를 공유할 수 있습니다.

예를 들어, 위 예제의 마지막 몇 줄은 중첩 경로를 사용하여 다음과 같이 개선될 수 있습니다:

**Java**

```java
RouterFunction<ServerResponse> route = route()
	.path("/person", builder -> builder // 1. "/person" 경로를 기준으로 중첩 시작
		.GET("/{id}", accept(APPLICATION_JSON), handler::getPerson)
		.GET(accept(APPLICATION_JSON), handler::listPeople) // 기본 경로 "/person"에 상대적
		.POST(handler::createPerson))
	.build();
```

1. `path`의 두 번째 파라미터는 라우터 빌더를 받는 소비자(consumer)임에 유의하세요.

경로 기반 중첩이 가장 일반적이지만, 빌더의 `nest` 메소드를 사용하여 모든 종류의 술어에 대해 중첩할 수 있습니다. 위 예제는 여전히 공유된 `Accept`-헤더 술어 형태의 중복을 포함하고 있습니다. `accept`와 함께 `nest` 메소드를 사용하여 더욱 개선할 수 있습니다:

**Java**

```java
RouterFunction<ServerResponse> route = route()
	.path("/person", b1 -> b1 // "/person" 경로
		.nest(accept(APPLICATION_JSON), b2 -> b2 // 하위 경로들은 application/json Accept 헤더 필요
			.GET("/{id}", handler::getPerson)
			.GET(handler::listPeople))
		.POST(handler::createPerson)) // POST는 application/json Accept 헤더 불필요
	.build();
```

### Serving Resources (리소스 제공)

WebMvc.fn은 리소스 제공을 위한 내장 지원을 제공합니다.

아래 설명된 기능 외에도 `RouterFunctions#resource(java.util.function.Function)` 덕분에 훨씬 더 유연한 리소스 처리를 구현할 수 있습니다.

### Redirecting to a resource (리소스로 리다이렉팅)

지정된 술어와 일치하는 요청을 리소스로 리다이렉팅할 수 있습니다. 이는 예를 들어 단일 페이지 애플리케이션(Single Page Applications)에서 리다이렉트를 처리하는 데 유용할 수 있습니다.

**Java**

```java
ClassPathResource index = new ClassPathResource("static/index.html");
List<String> extensions = List.of("js", "css", "ico", "png", "jpg", "gif");
// API, 에러, 특정 확장자 요청이 아니면 index.html로 리다이렉트 (SPA 라우팅 지원)
RequestPredicate spaPredicate = path("/api/**").or(path("/error")).or(pathExtension(extensions::contains)).negate();
RouterFunction<ServerResponse> redirectToIndex = route()
	.resource(spaPredicate, index)
	.build();
```

### Serving resources from a root location (루트 위치에서 리소스 제공)

주어진 패턴과 일치하는 요청을 주어진 루트 위치에 상대적인 리소스로 라우팅할 수도 있습니다.

**Java**

```java
Resource location = new FileUrlResource("public-resources/"); // "public-resources/" 디렉토리 지정
RouterFunction<ServerResponse> resources = RouterFunctions.resources("/resources/**", location);
// /resources/image.png 요청 -> public-resources/image.png 파일 제공
```

### Running a Server (서버 실행)

일반적으로 라우터 함수는 MVC 설정을 통해 `DispatcherHandler` 기반 설정에서 실행되며, 이 설정은 요청 처리에 필요한 컴포넌트를 선언하기 위해 스프링 구성을 사용합니다. MVC Java 구성은 함수형 엔드포인트를 지원하기 위해 다음 인프라 컴포넌트를 선언합니다:

- `RouterFunctionMapping`: 스프링 구성에서 하나 이상의 `RouterFunction<?>` 빈을 감지하고, 순서를 지정하고, `RouterFunction.andOther`를 통해 결합하고, 결과로 구성된 `RouterFunction`으로 요청을 라우팅합니다.
- `HandlerFunctionAdapter`: `DispatcherHandler`가 요청에 매핑된 `HandlerFunction`을 호출할 수 있도록 하는 간단한 어댑터입니다.

앞서 언급된 컴포넌트들은 함수형 엔드포인트가 `DispatcherServlet` 요청 처리 생명주기에 적합하도록 하며, 선언된 경우 어노테이션 기반 컨트롤러와도 (잠재적으로) 나란히 실행될 수 있도록 합니다. 또한 이것이 스프링 부트 웹 스타터에 의해 함수형 엔드포인트가 활성화되는 방식입니다.

다음 예제는 WebFlux Java 구성(이지만 Web MVC에도 유사하게 적용 가능)을 보여줍니다:

**Java**

```java
@Configuration
@EnableMvc // @EnableWebMvc 와 유사
public class WebConfig implements WebMvcConfigurer {

	@Bean
	public RouterFunction<?> routerFunctionA() {
		// ... (라우터 함수 A 정의)
	}

	@Bean
	public RouterFunction<?> routerFunctionB() {
		// ... (라우터 함수 B 정의)
	}

	// ...

	@Override
	public void configureMessageConverters(List<HttpMessageConverter<?>> converters) {
		// 메시지 변환 구성...
	}

	@Override
	public void addCorsMappings(CorsRegistry registry) {
		// CORS 구성...
	}

	@Override
	public void configureViewResolvers(ViewResolverRegistry registry) {
		// HTML 렌더링을 위한 뷰 리졸루션 구성...
	}
}
```

### Filtering Handler Functions (핸들러 함수 필터링)

라우팅 함수 빌더의 `before`, `after`, 또는 `filter` 메소드를 사용하여 핸들러 함수를 필터링할 수 있습니다. 어노테이션을 사용하면 `@ControllerAdvice`, `ServletFilter`, 또는 둘 다를 사용하여 유사한 기능을 구현할 수 있습니다. 필터는 빌더에 의해 빌드된 모든 경로에 적용됩니다. 즉, 중첩된 경로에 정의된 필터는 "최상위" 경로에는 적용되지 않습니다. 예를 들어, 다음 예제를 고려하십시오:

**Java**

```java
RouterFunction<ServerResponse> route = route()
	.path("/person", b1 -> b1
		.nest(accept(APPLICATION_JSON), b2 -> b2
			.GET("/{id}", handler::getPerson)
			.GET(handler::listPeople)
			.before(request -> ServerRequest.from(request) // 1. 이 before 필터는 두 GET 경로에만 적용
				.header("X-RequestHeader", "Value")
				.build()))
		.POST(handler::createPerson))
	.after((request, response) -> logResponse(response)) // 2. 이 after 필터는 중첩된 경로 포함 모든 경로에 적용
	.build();
```

1. 사용자 정의 요청 헤더를 추가하는 `before` 필터는 두 GET 경로에만 적용됩니다.
2. 응답을 로깅하는 `after` 필터는 중첩된 경로를 포함한 모든 경로에 적용됩니다.

라우터 빌더의 `filter` 메소드는 `HandlerFilterFunction`을 받습니다: 이 함수는 `ServerRequest`와 `HandlerFunction`을 받아 `ServerResponse`를 반환합니다. 핸들러 함수 파라미터는 체인의 다음 요소를 나타냅니다. 이것은 일반적으로 라우팅된 핸들러이지만, 여러 필터가 적용된 경우 다른 필터일 수도 있습니다.

이제 특정 경로가 허용되는지 여부를 결정할 수 있는 `SecurityManager`가 있다고 가정하고 경로에 간단한 보안 필터를 추가할 수 있습니다. 다음 예제는 그렇게 하는 방법을 보여줍니다:

```java
SecurityManager securityManager = ... ;

RouterFunction<ServerResponse> route = route()
	.path("/person", b1 -> b1
		.nest(accept(APPLICATION_JSON), b2 -> b2
			.GET("/{id}", handler::getPerson)
			.GET(handler::listPeople))
		.POST(handler::createPerson))
	.filter((request, next) -> { // 모든 /person/** 경로에 적용되는 필터
		if (securityManager.allowAccessTo(request.path())) {
			return next.handle(request); // 허용되면 다음 핸들러(또는 필터) 실행
		}
		else {
			return ServerResponse.status(HttpStatus.UNAUTHORIZED).build(); // 허용되지 않으면 401 응답
		}
	})
	.build();
```

앞의 예제는 `next.handle(ServerRequest)` 호출이 선택 사항임을 보여줍니다. 접근이 허용될 때만 핸들러 함수가 실행되도록 합니다.

라우터 함수 빌더에서 `filter` 메소드를 사용하는 것 외에도, `RouterFunction.filter(HandlerFilterFunction)`를 통해 기존 라우터 함수에 필터를 적용할 수 있습니다.

함수형 엔드포인트에 대한 CORS 지원은 전용 `CorsFilter`를 통해 제공됩니다.

---

### **학습 목표 제시**

이 챕터를 학습하고 나면, 학생은 다음을 이해하고 수행할 수 있게 됩니다:

1. **함수형 엔드포인트(WebMvc.fn)의 기본 개념 이해:** 어노테이션 기반 모델의 대안으로, 함수를 사용하여 요청 라우팅(`RouterFunction`)과 처리(`HandlerFunction`)를 수행하는 함수형 프로그래밍 모델의 핵심 아이디어를 이해합니다.
2. **`RouterFunction`과 `HandlerFunction` 작성 및 사용 방법 숙지:** `RouterFunctions.route()` 빌더를 사용하여 요청 경로, HTTP 메소드, 헤더 조건 등을 기반으로 요청을 특정 `HandlerFunction`으로 라우팅하는 방법을 이해하고, `ServerRequest`를 받아 `ServerResponse`를 반환하는 `HandlerFunction`을 작성할 수 있습니다.
3. **`ServerRequest`와 `ServerResponse`의 활용 이해:** 불변 객체인 `ServerRequest`를 통해 요청 정보(헤더, 본문, 파라미터 등)에 접근하고, `ServerResponse` 빌더를 사용하여 상태 코드, 헤더, 본문을 포함한 응답을 생성하는 방법을 이해합니다.
4. **함수형 엔드포인트에서의 검증, 필터링, 리소스 제공 방법 인지:** 함수형 스타일로 검증을 적용하고, 핸들러 함수 전후에 필터를 적용하며, 정적 리소스를 제공하는 기본적인 방법을 인지합니다. (비동기 처리, SSE 등은 심화 내용으로 간단히 인지만 합니다.)

---

### **핵심 개념 설명: Functional Endpoints (WebMvc.fn) 란?**

**기존 방식과의 비교:**

- **어노테이션 기반 모델 (우리가 지금까지 배운 것):**
  - `@Controller`, `@RestController` 클래스 안에 `@RequestMapping`, `@GetMapping` 등의 어노테이션이 붙은 메소드를 만들어 요청을 처리합니다.
  - 어노테이션이 "어떤 URL 요청이 오면 이 메소드를 실행해줘" 라고 스프링에게 알려주는 선언적인 방식입니다.
- **함수형 엔드포인트 모델 (WebMvc.fn):**
  - *함수(Function)**를 사용하여 요청을 라우팅하고 처리합니다.
  - **라우팅(Routing):** 어떤 요청이 들어왔을 때 어떤 함수가 처리할지를 결정하는 규칙을 함수 형태로 정의합니다 (`RouterFunction`).
  - **처리(Handling):** 실제 요청을 받아서 응답을 생성하는 로직을 함수 형태로 정의합니다 (`HandlerFunction`).
  - 어노테이션 대신 코드를 통해 명시적으로 라우팅 및 처리 로직을 구성합니다.
  - **불변성(Immutability):** 요청(`ServerRequest`)과 응답(`ServerResponse`) 객체가 불변 계약을 가지도록 설계되어, 상태 변경으로 인한 부작용을 줄이고 예측 가능성을 높입니다. (JDK 8+ 친화적)

**주요 구성 요소:**

1. **`HandlerFunction<T extends ServerResponse>`:**
  - **역할:** HTTP 요청을 실제로 처리하고 응답을 생성하는 함수입니다.
  - **시그니처:** `ServerRequest`를 입력으로 받아 `ServerResponse` (또는 그 하위 타입)를 반환합니다.
  - **비유:** 어노테이션 기반 모델의 `@RequestMapping` 메소드 **본문(body)**에 해당합니다. 실제 로직이 들어가는 부분이죠.
2. **`RouterFunction<T extends ServerResponse>`:**
  - **역할:** 들어오는 HTTP 요청을 적절한 `HandlerFunction`으로 라우팅(연결)하는 함수입니다.
  - **시그니처:** `ServerRequest`를 입력으로 받아 `Optional<HandlerFunction<T>>`를 반환합니다.
    - 요청이 특정 조건(경로, HTTP 메소드, 헤더 등)에 맞으면 해당 요청을 처리할 `HandlerFunction`을 담은 `Optional`을 반환합니다.
    - 맞는 `HandlerFunction`이 없으면 빈 `Optional`을 반환합니다.
  - **비유:** 어노테이션 기반 모델의 `@RequestMapping` **어노테이션 자체**에 해당합니다. 하지만 단순한 메타데이터가 아니라, 라우팅 로직(동작)을 직접 코드로 정의한다는 차이가 있습니다.

**`RouterFunctions.route()` 빌더:**

`RouterFunction`을 직접 처음부터 구현하는 것은 복잡할 수 있습니다. 그래서 스프링은 `RouterFunctions`라는 유틸리티 클래스와 `route()`라는 빌더 메소드를 제공하여, 보다 쉽고 읽기 편하게 라우팅 규칙을 정의할 수 있도록 도와줍니다.

```java
// 예시 (원문 발췌 및 일부 수정)
import static org.springframework.http.MediaType.APPLICATION_JSON;
import static org.springframework.web.servlet.function.RequestPredicates.*; // 요청 조건을 정의하는 Predicate들
import static org.springframework.web.servlet.function.RouterFunctions.route; // 라우터 빌더 시작
import static org.springframework.web.servlet.function.ServerResponse.ok; // 응답 생성을 위한 유틸리티

// ... PersonRepository, PersonHandler 정의는 아래에서 ...

RouterFunction<ServerResponse> personRoutes = route() // 빌더 시작
    .GET("/person/{id}", accept(APPLICATION_JSON), handler::getPerson) // (1)
    .GET("/person", accept(APPLICATION_JSON), handler::listPeople)    // (2)
    .POST("/person", handler::createPerson)                         // (3)
    .build();                                                       // (4)
```

1. **`GET("/person/{id}", accept(APPLICATION_JSON), handler::getPerson)`:**
  - HTTP GET 요청이 `/person/{id}` 경로로 오고,
  - `Accept` 헤더가 `application/json`을 포함하면 (클라이언트가 JSON 응답을 원하면),
  - `handler` 객체의 `getPerson` 메소드( `HandlerFunction` )를 실행하라. (`handler::getPerson`은 메소드 레퍼런스)
2. **`GET("/person", accept(APPLICATION_JSON), handler::listPeople)`:**
  - HTTP GET 요청이 `/person` 경로로 오고, `Accept` 헤더가 `application/json`이면, `handler.listPeople` 실행.
3. **`POST("/person", handler::createPerson)`:**
  - HTTP POST 요청이 `/person` 경로로 오면 (추가 조건 없음), `handler.createPerson` 실행.
4. **`.build()`:** 정의된 라우팅 규칙들을 모아 하나의 `RouterFunction` 객체를 생성합니다.

이 `RouterFunction` 빈을 스프링 설정(`@Configuration` 클래스)에 `@Bean`으로 등록하면, `DispatcherServlet`이 이를 감지하여 요청 처리에 사용합니다.

### **`HandlerFunction`의 핵심: `ServerRequest`와 `ServerResponse`**

`HandlerFunction`은 `ServerRequest`를 받아 `ServerResponse`를 반환합니다. 이 두 인터페이스는 함수형 엔드포인트에서 HTTP 요청과 응답을 다루는 핵심입니다.

### `ServerRequest` (불변 요청 객체)

- HTTP 요청의 모든 정보(메소드, URI, 헤더, 쿼리 파라미터, 본문 등)에 접근할 수 있는 메소드를 제공합니다.
- **불변(Immutable):** 한번 생성된 `ServerRequest` 객체의 상태는 변경되지 않습니다. 이는 스레드 안전성을 높이고 예측 가능성을 향상시킵니다.
- **주요 메소드:**
  - `String pathVariable(String name)`: 경로 변수 값 가져오기 (예: `/person/{id}`에서 `id` 값).
  - `Optional<String> param(String name)`: 쿼리 파라미터 값 가져오기.
  - `MultiValueMap<String, String> params()`: 모든 쿼리 파라미터 가져오기.
  - `Headers headers()`: 요청 헤더 정보 (`ServerRequest.Headers` 타입) 가져오기.
  - `<T> T body(Class<T> bodyType)`: 요청 본문을 지정된 클래스 타입(`T`)으로 변환 (예: `String.class`, `Person.class`). `HttpMessageConverter`가 사용됩니다.
  - `<T> T body(ParameterizedTypeReference<T> typeReference)`: 요청 본문을 제네릭 타입(예: `List<Person>`)으로 변환.
  - `Optional<URI> uri()`: 전체 요청 URI.
  - `HttpMethod method()`: HTTP 메소드 (GET, POST 등).

### `ServerResponse` (불변 응답 객체)

- HTTP 응답(상태 코드, 헤더, 본문)을 생성하는 데 사용됩니다.
- **불변(Immutable):** `ServerResponse` 객체도 불변이며, 빌더 패턴을 사용하여 생성합니다.
- **빌더 패턴:** `ServerResponse.ok()`, `ServerResponse.created(URI location)`, `ServerResponse.status(HttpStatus status)` 등과 같은 정적 팩토리 메소드로 빌더를 시작하고, `contentType()`, `header()`, `body()` 등을 체이닝하여 응답을 구성한 후, 최종적으로 `build()` (본문 없을 때) 또는 `body()` (본문 설정 시)를 호출하여 `ServerResponse` 객체를 생성합니다.
- **주요 빌더 메소드:**
  - `ok()`: 200 OK 상태.
  - `created(URI location)`: 201 Created 상태와 `Location` 헤더 설정.
  - `accepted()`: 202 Accepted 상태.
  - `noContent()`: 204 No Content 상태 (본문 없음).
  - `badRequest()`: 400 Bad Request 상태.
  - `notFound()`: 404 Not Found 상태.
  - `status(HttpStatus status)` 또는 `status(int rawStatus)`: 특정 상태 코드 설정.
  - `contentType(MediaType contentType)`: `Content-Type` 헤더 설정.
  - `header(String headerName, String... headerValues)`: 헤더 추가.
  - `<T> BodyBuilder body(T body)`: 응답 본문 설정. `HttpMessageConverter`가 `T` 타입 객체를 직렬화합니다. (이 메소드 자체가 `ServerResponse`를 반환)
  - `build()`: 본문 없이 응답을 최종 생성.

**비동기 처리 및 SSE 지원:**
원문에서 언급된 것처럼, `ServerResponse`는 `CompletableFuture`, Project Reactor의 `Mono`/`Flux` (주로 WebFlux에서 사용되지만, Web MVC에서도 `ReactiveAdapterRegistry`를 통해 일부 비동기 시나리오 지원 가능)와 같은 비동기 타입을 본문으로 지원합니다. 또한, 서버-전송 이벤트(Server-Sent Events, SSE)를 위한 `ServerResponse.sse()` 빌더도 제공합니다. (이 부분들은 고급 주제이므로, 지금은 이런 기능도 있다는 정도만 인지하고 넘어가도 좋습니다.)

### **Handler Classes (핸들러 클래스)**

람다(lambda)를 사용하여 간단한 `HandlerFunction`을 직접 정의할 수도 있지만, 여러 관련된 `HandlerFunction`들을 하나의 클래스로 _**그룹화**_하는 것이 일반적입니다. 이 "핸들러 클래스"는 어노테이션 기반 모델의 `@Controller`와 유사한 역할을 합니다.

```java
// PersonHandler.java (원문 예시)
public class PersonHandler {

    private final PersonRepository repository; // 의존성 주입 (생성자 통해)

    public PersonHandler(PersonRepository repository) {
        this.repository = repository;
    }

    // 각 메소드가 HandlerFunction의 시그니처를 따름 (ServerRequest -> ServerResponse)
    public ServerResponse listPeople(ServerRequest request) {
        List<Person> people = repository.allPeople();
        return ServerResponse.ok().contentType(APPLICATION_JSON).body(people);
    }

    public ServerResponse createPerson(ServerRequest request) throws Exception {
        Person person = request.body(Person.class); // 요청 본문을 Person 객체로 변환
        repository.savePerson(person);
        return ServerResponse.ok().build(); // 본문 없는 200 OK
    }

    public ServerResponse getPerson(ServerRequest request) {
        int personId = Integer.parseInt(request.pathVariable("id")); // 경로 변수 가져오기
        Person person = repository.getPerson(personId);
        if (person != null) {
            return ServerResponse.ok().contentType(APPLICATION_JSON).body(person);
        } else {
            return ServerResponse.notFound().build(); // 404 Not Found
        }
    }
}
```

이렇게 핸들러 클래스를 만들고, `RouterFunction`을 정의할 때 이 클래스의 메소드들을 메소드 레퍼런스(`handler::getPerson`) 형태로 참조하여 사용합니다.

### **Validation (검증)**

함수형 엔드포인트에서도 요청 본문에 대한 검증을 수행할 수 있습니다.

- **스프링 `Validator` 사용:**
  1. `org.springframework.validation.Validator` 인터페이스를 구현한 커스텀 Validator (예: `PersonValidator`)를 만듭니다.
  2. 핸들러 함수 내에서, `request.body(Person.class)`로 객체를 받은 후, 이 객체와 `BeanPropertyBindingResult` (또는 `Errors`) 객체를 사용하여 `validator.validate(person, errors)`를 직접 호출합니다.
  3. `errors.hasErrors()`를 확인하여 오류가 있으면, 적절한 예외(예: `ServerWebInputException` - WebFlux에서 주로 사용, Web MVC에서는 커스텀 예외나 `ResponseStatusException` 등 고려)를 발생시키거나 오류 `ServerResponse`를 직접 생성하여 반환합니다.
- **빈 검증 API (Bean Validation - JSR-303) 사용:**
  - `LocalValidatorFactoryBean` 기반의 전역 `Validator`를 주입받아 사용할 수 있습니다.
  - 사용 방식은 위 스프링 `Validator`와 유사합니다.

어노테이션 기반 모델처럼 `@Valid` 어노테이션만 붙이면 자동으로 검증되고 예외가 터지는 방식은 아니며, **핸들러 함수 내에서 명시적으로 검증 로직을 호출**해야 합니다.

### **`RouterFunction` 심화: Predicates, Routes, Nested Routes**

### Predicates (술어)

`RequestPredicate`는 특정 요청이 주어진 조건에 맞는지 여부를 판단하는 함수입니다 (`ServerRequest -> boolean`). `RouterFunctions.route()` 빌더에서 경로, HTTP 메소드 외에 추가적인 매칭 조건을 지정하는 데 사용됩니다.

`RequestPredicates` 유틸리티 클래스는 다양한 종류의 미리 정의된 `RequestPredicate`를 제공합니다:

- `path(String pattern)`: URL 경로 패턴 매칭.
- `method(HttpMethod httpMethod)`: HTTP 메소드 매칭.
- `contentType(MediaType... mediaTypes)`: 요청의 `Content-Type` 헤더 매칭.
- `accept(MediaType... mediaTypes)`: 요청의 `Accept` 헤더 매칭.
- `header(String name, String value)`: 특정 헤더 값 매칭.
- `param(String name, String value)`: 특정 쿼리 파라미터 값 매칭.
- `pathExtension(String extension)`: 경로 확장자 매칭.

이러한 `RequestPredicate`들은 `.and()` 또는 `.or()` 메소드를 사용하여 조합할 수 있습니다.

### Routes (경로)

- **순서 중요:** `RouterFunction`에 정의된 라우팅 규칙들은 **선언된 순서대로 평가**됩니다. 만약 첫 번째 규칙이 요청과 매칭되면 그 규칙에 연결된 `HandlerFunction`이 실행되고, 나머지 규칙들은 평가되지 않습니다. 따라서 **더 구체적인 경로 규칙을 일반적인 경로 규칙보다 먼저 선언**해야 합니다. (어노테이션 기반에서는 스프링이 "가장 구체적인" 핸들러를 자동으로 찾아주지만, 함수형에서는 순서가 중요합니다.)
- **구성(Composition):** 여러 `RouterFunction`들을 `add()` (빌더 사용 시) 또는 `and()` 메소드를 사용하여 하나의 큰 `RouterFunction`으로 결합할 수 있습니다.

### Nested Routes (중첩 경로)

여러 라우팅 규칙이 공통된 경로 접두사(prefix)나 다른 술어(predicate)를 공유할 때, 중첩 경로를 사용하여 코드를 더 간결하고 가독성 있게 만들 수 있습니다.

- `builder.path("/공통경로", nestedBuilder -> { ... })`: `/공통경로`를 기준으로 하위 경로들을 중첩 정의.
- `builder.nest(RequestPredicate, nestedBuilder -> { ... })`: 특정 `RequestPredicate` (예: `accept(APPLICATION_JSON)`)를 기준으로 하위 경로들을 중첩 정의.

```java
// 원문 예시 (중첩 경로 사용)
RouterFunction<ServerResponse> route = route()
    .path("/person", b1 -> b1 // (A) "/person" 경로 중첩 시작
        .nest(accept(APPLICATION_JSON), b2 -> b2 // (B) Accept 헤더가 JSON인 경우 중첩 시작
            .GET("/{id}", handler::getPerson)     // --> /person/{id} + Accept: JSON
            .GET(handler::listPeople))            // --> /person + Accept: JSON
        .POST(handler::createPerson))          // (C) --> /person (Accept 헤더 조건 없음)
    .build();

```

(A), (B), (C)는 각각 다른 수준의 경로/조건을 나타냅니다.

### **Serving Resources (리소스 제공)**

- **특정 요청을 리소스로 리다이렉트:** `route().resource(RequestPredicate, Resource).build()`를 사용하여, 특정 조건에 맞는 요청을 `index.html` 같은 정적 리소스로 리다이렉트할 수 있습니다. (SPA 라우팅 지원에 유용)
- **특정 경로 패턴 요청을 루트 위치의 리소스로 제공:** `RouterFunctions.resources("/경로패턴/**", Resource루트위치)`를 사용하여, 특정 URL 패턴 요청을 파일 시스템의 특정 디렉토리 내 리소스로 매핑하여 제공할 수 있습니다. (예: `/static/**` 요청을 `classpath:/static/` 디렉토리의 파일로)

### **Running a Server (서버 실행) 및 Filtering Handler Functions (필터링)**

- **`RouterFunctionMapping`과 `HandlerFunctionAdapter`:** `@Configuration` 클래스에 `RouterFunction` 타입의 빈을 등록하면, 스프링 MVC는 `RouterFunctionMapping`을 통해 이 빈들을 감지하고 결합하여 실제 라우팅에 사용합니다. `HandlerFunctionAdapter`는 `DispatcherServlet`이 매핑된 `HandlerFunction`을 호출할 수 있도록 셔주는 역할을 합니다. 이 덕분에 함수형 엔드포인트는 기존의 `DispatcherServlet` 요청 처리 생명주기 내에서 어노테이션 기반 컨트롤러와 함께 동작할 수 있습니다.
- **필터링:** 라우터 함수 빌더의 `before()`, `after()`, `filter()` 메소드를 사용하여 특정 라우팅 규칙 그룹에 필터를 적용할 수 있습니다.
  - `before(Function<ServerRequest, ServerRequest> requestProcessor)`: 핸들러 함수 실행 전에 요청을 가공.
  - `after(BiFunction<ServerRequest, ServerResponse, ServerResponse> responseProcessor)`: 핸들러 함수 실행 후에 응답을 가공.
  - `filter(HandlerFilterFunction filterFunction)`: `HandlerFilterFunction` ( `(ServerRequest, HandlerFunction) -> ServerResponse` )을 사용하여 요청 처리 체인에 개입. `next.handle(request)`를 호출하여 다음 필터나 핸들러로 제어를 넘길 수 있고, 호출하지 않고 직접 응답을 생성할 수도 있습니다 (예: 보안 필터).
  - 필터의 적용 범위는 빌더의 중첩 구조에 따라 달라집니다.

### **"왜?" 라는 질문에 대한 답변**

**왜 어노테이션 기반 모델 외에 함수형 엔드포인트 모델이 필요할까요?**

1. **프로그래밍 방식의 완전한 제어:** 어노테이션은 선언적이지만, 때로는 런타임 조건에 따라 동적으로 라우팅 규칙을 변경하거나 복잡한 라우팅 로직을 구현해야 할 수 있습니다. 함수형 모델은 모든 것을 코드로 명시적으로 정의하므로 이러한 완전한 제어가 가능합니다.
2. **함수형 프로그래밍 패러다임 선호:** 함수형 프로그래밍에 익숙하거나 선호하는 개발자들에게 더 자연스러운 코딩 스타일을 제공합니다. 불변성, 부작용 최소화 등의 함수형 특징을 활용할 수 있습니다.
3. **경량성 및 성능 (이론상):** 어노테이션 스캔 및 리플렉션 사용이 줄어들기 때문에, 이론적으로는 약간의 시작 시간 단축이나 런타임 성능 이점이 있을 수 있습니다. (실제 체감 효과는 애플리케이션 규모나 복잡도에 따라 다를 수 있습니다.)
4. **명시적인 의존성 및 구성:** 모든 라우팅과 핸들러가 코드로 명시되므로, 애플리케이션의 동작 방식을 이해하기 더 쉬울 수 있다는 관점도 있습니다. (어노테이션의 "마법"이 줄어듦)
5. **WebFlux와의 일관성:** 스프링의 반응형 웹 프레임워크인 WebFlux에서는 함수형 엔드포인트가 주요 프로그래밍 모델 중 하나입니다. WebMvc.fn은 WebFlux의 함수형 모델과 매우 유사한 API를 제공하므로, 두 프레임워크 간 전환이나 코드 공유가 용이해질 수 있습니다.

**어떤 경우에 함수형 엔드포인트를 사용하는 것이 좋을까요?**

- 간단한 마이크로서비스나 특정 기능에 대해 경량의 엔드포인트가 필요할 때.
- 라우팅 로직이 매우 동적이거나 복잡하여 어노테이션만으로는 표현하기 어려울 때.
- 팀이 함수형 프로그래밍 스타일을 선호하고, 애플리케이션 전체적으로 일관된 스타일을 유지하고 싶을 때.
- 어노테이션의 암묵적인 동작보다는 모든 것을 코드로 명시적으로 제어하는 것을 선호할 때.

하지만, 대부분의 일반적인 웹 애플리케이션에서는 **어노테이션 기반 모델이 여전히 매우 강력하고 생산적**입니다. 함수형 엔드포인트는 또 다른 "선택지"를 제공하는 것이며, 반드시 어노테이션 방식을 대체해야 하는 것은 아닙니다. 두 가지 방식을 함께 사용할 수도 있습니다.

### **주의사항 및 Best Practice**

1. **학습 곡선:** 어노테이션 기반 모델에 익숙한 개발자에게는 함수형 스타일이 초기에 낯설 수 있습니다. `RouterFunction`, `HandlerFunction`, `ServerRequest`, `ServerResponse` 등의 새로운 개념과 API에 익숙해질 필요가 있습니다.
2. **코드 가독성:** 간단한 경우에는 람다를 사용하여 간결하게 표현할 수 있지만, 라우팅 규칙이나 핸들러 로직이 복잡해지면 코드가 길어지고 가독성이 떨어질 수 있습니다. 핸들러 클래스로 로직을 적절히 분리하고, 중첩 경로 등을 현명하게 사용하는 것이 중요합니다.
3. **순서의 중요성 (라우팅):** 라우팅 규칙은 선언된 순서대로 평가되므로, 구체적인 규칙을 먼저 배치해야 합니다.
4. **에러 처리:** 핸들러 함수 내에서 발생하는 예외는 명시적으로 처리하거나, 함수형 엔드포인트용 예외 처리 필터 등을 구성해야 합니다. (어노테이션 기반의 `@ExceptionHandler`와는 다른 방식으로 접근)
5. **의존성 주입:** 핸들러 클래스에 필요한 의존성(예: 서비스, 레포지토리)은 생성자를 통해 주입받는 것이 일반적입니다.

### **이전 학습 내용과의 연관성**

- **`DispatcherServlet`:** 함수형 엔드포인트도 결국 동일한 `DispatcherServlet` 위에서 동작합니다. 요청 처리 생명주기의 일부로 통합됩니다.
- **`HttpMessageConverter`:** `ServerRequest.body()`로 요청 본문을 객체로 변환하거나, `ServerResponse.body()`로 객체를 응답 본문으로 직렬화할 때 여전히 `HttpMessageConverter`가 사용됩니다.
- **검증(Validation):** 어노테이션 기반 검증과는 방식이 다르지만, 스프링의 `Validator`나 빈 검증 API를 핸들러 함수 내에서 수동으로 호출하여 사용할 수 있습니다.
- **REST API 설계 원칙:** 상태 코드, 헤더, HTTP 메소드 등을 사용하여 RESTful API를 구축하는 원칙은 함수형 엔드포인트에서도 동일하게 중요합니다. `ServerResponse` 빌더를 통해 이를 명시적으로 제어할 수 있습니다.
- **`@Controller`, `@RequestMapping` (대안 관계):** 함수형 엔드포인트는 이러한 어노테이션 기반 방식의 "대안" 프로그래밍 모델입니다.

---
