---
title: Spring Web MVC - Annotated Controllers (@RequestAttribute)
description: 
author: laze
date: 2025-06-06 00:00:01 +0900
categories: [Dev, SpringWebMVC]
tags: [SpringWebMVC]
---
### @RequestAttribute

> Reactive 스택에서의 동일 기능은 해당 문서를 참고하세요.
>

`@SessionAttribute`와 유사하게, `@RequestAttribute` 어노테이션을 사용하여 이전에 (예를 들어, 서블릿 필터(Servlet Filter)나 핸들러 인터셉터(HandlerInterceptor)에 의해) 미리 생성된 요청 속성(request attributes)에 접근할 수 있습니다.

**Java**

```java
@GetMapping("/")
public String handle(@RequestAttribute Client client) {
	// ...
}
```

- `@RequestAttribute` 어노테이션 사용하기.

---

### **학습 목표 제시**

이 챕터를 학습하고 나면, 학생은 다음을 이해하고 수행할 수 있게 됩니다:

1. **`@RequestAttribute` 어노테이션의 역할과 사용법:** HTTP 요청 범위(request scope)에 저장된 속성 값을 컨트롤러 메소드에서 어떻게 가져와 활용하는지 이해합니다.
2. **요청 속성(Request Attribute)의 개념 이해:** 요청 속성이 무엇이며, 어떤 경우에 사용되는지 (특히 필터나 인터셉터와의 연관성) 이해합니다.
3. **`@SessionAttribute`와의 차이점 인지:** `@RequestAttribute`가 `@SessionAttribute`와 어떤 점에서 유사하고 다른지 간략하게나마 인지합니다. (비록 `@SessionAttribute`를 아직 배우지 않았더라도, 이 챕터에서 언급된 내용을 통해 기본적인 차이를 짐작할 수 있습니다.)

---

### **핵심 개념 설명**

**`@RequestAttribute`란 무엇일까요?**

웹 애플리케이션에서 하나의 HTTP 요청이 서버에 도착해서 클라이언트에게 응답이 돌아갈 때까지, 그 **단일 요청 처리 과정 동안에만 유지되는 데이터를 저장하는 공간**이 있습니다.

이를 **요청 범위(request scope)**라고 부릅니다. 마치 짧은 여행 가방과 같아서, 이번 여행(요청 처리)에만 필요한 물건들을 잠시 담아두는 곳이라고 생각할 수 있습니다.

`@RequestAttribute` 어노테이션은 바로 이 **요청 범위(request scope)에 저장된 특정 속성(attribute) 값을 스프링 MVC 컨트롤러의 메소드 매개변수로 편리하게 가져올 수 있도록 도와주는 역할**을 합니다.

**중요한 점:** `@RequestAttribute`는 클라이언트가 직접 HTTP 요청 메시지(헤더나 쿠키 등)에 담아 보내는 데이터를 가져오는 것이 아닙니다.

대신, **서버 내부에서 해당 요청을 처리하는 과정 중에 누군가(다른 컴포넌트)가 요청 범위에 미리 저장해 둔 데이터를 가져오는 것**입니다.

**누가 요청 범위에 데이터를 저장할까요?**

원문에서 언급된 것처럼, 주로 다음과 같은 컴포넌트들이 요청을 컨트롤러가 받기 *전에* 또는 *후에* 요청 범위에 데이터를 저장할 수 있습니다.

- **서블릿 필터 (Servlet Filter):** 모든 요청 또는 특정 패턴의 요청에 대해 공통적인 전처리나 후처리를 수행합니다. 예를 들어, 모든 요청에 대해 사용자 인증 정보를 확인하고, 그 결과를 요청 속성에 저장해두면 이후 컨트롤러나 뷰에서 사용할 수 있습니다.
- **핸들러 인터셉터 (HandlerInterceptor):** 스프링 MVC에서 컨트롤러의 특정 핸들러 메소드가 실행되기 전, 후, 또는 완료 시점에 개입하여 추가적인 로직을 수행합니다. 필터와 유사하게, 요청 처리 과정에서 필요한 데이터를 가공하거나 조회하여 요청 속성에 저장할 수 있습니다.

**비유:**

여러분이 놀이공원에 입장한다고 상상해봅시다.

1. **입구 (서블릿 필터 또는 핸들러 인터셉터):**
  - 직원이 여러분의 티켓을 확인하고, 손목에 **자유이용권 팔찌(요청 속성)**를 채워줍니다. 이 팔찌에는 오늘 날짜, 입장 시간 등의 정보가 기록되어 있을 수 있습니다.
2. **놀이기구 탑승 (컨트롤러):**
  - 각 놀이기구 앞의 직원은 여러분의 **자유이용권 팔찌(요청 속성)**를 보고 "아, 오늘 입장한 손님이 맞으시군요. 이 놀이기구는 이용 가능합니다."라고 판단하고 놀이기구에 태워줍니다.

여기서 `@RequestAttribute`는 놀이기구 직원이 손님의 팔찌(요청 속성)에 있는 정보를 쉽게 확인하는 것과 같습니다. 팔찌는 놀이공원 입구에서 채워준 것이지, 손님이 집에서부터 직접 만들어서 가져온 것이 아니라는 점이 중요합니다.

예를 들어, 어떤 필터가 모든 요청에 대해 사용자 정보를 데이터베이스에서 조회한 후, 그 사용자 객체를 `currentUser`라는 이름으로 요청 속성에 저장했다고 합시다. 그러면 컨트롤러에서는 `@RequestAttribute("currentUser") User user`와 같이 코드를 작성하여 해당 사용자 객체를 바로 주입받아 사용할 수 있습니다.

### **주요 용어 해설**

- **요청 속성 (Request Attribute):** 하나의 HTTP 요청이 처리되는 동안에만 데이터를 저장하고 공유하기 위한 메커니즘입니다. `HttpServletRequest` 객체의 `setAttribute(String name, Object value)` 메소드를 통해 저장하고, `getAttribute(String name)` 메소드를 통해 읽을 수 있습니다.
- **요청 범위 (Request Scope):** 데이터가 유효한 범위를 나타내는 용어 중 하나로, 요청 속성은 '요청 범위'에서 유효합니다. 즉, 해당 요청이 시작되어 응답이 완료될 때까지만 데이터가 살아있습니다. 다음 요청에서는 이전 요청의 속성 값을 사용할 수 없습니다.
- **서블릿 필터 (Servlet Filter):** J2EE(Java EE) 스펙의 일부로, 클라이언트 요청이 서블릿(또는 컨트롤러)에 도달하기 전이나 서블릿이 응답을 보낸 후에 특정 작업을 수행할 수 있는 컴포넌트입니다. 주로 로깅, 인증, 권한 부여, 데이터 압축, 인코딩 변환 등의 공통 관심사를 처리합니다.
- **핸들러 인터셉터 (HandlerInterceptor):** 스프링 MVC 프레임워크가 제공하는 기능으로, 디스패처 서블릿(DispatcherServlet)이 컨트롤러의 핸들러 메소드를 호출하기 전후에 특정 로직을 실행할 수 있게 합니다. 필터보다 좀 더 스프링 MVC에 특화된 기능을 제공하며, 핸들러 메소드 자체에 대한 정보(예: 어떤 컨트롤러, 어떤 메소드인지)에 접근하기 용이합니다.
- **`@SessionAttribute` :** 이 어노테이션은 요청 범위보다 더 긴 생명주기를 가지는 **세션 범위(session scope)**에 저장된 속성 값을 가져올 때 사용됩니다. 세션은 보통 사용자가 로그인해서 로그아웃할 때까지 유지됩니다. `@RequestAttribute`와 사용 방식은 유사하지만, 데이터를 가져오는 저장소(범위)가 다릅니다.

### **코드 예제 및 분석**

제공된 원문의 Java 코드를 다시 한번 살펴보겠습니다.

```java
@GetMapping("/")
public String handle(@RequestAttribute Client client) { // 1. "client"라는 이름의 요청 속성 값을 Client 타입으로 받습니다.
	// 2. 여기서 client 객체를 사용하여
	//    요청 속성 값에 따른 로직을 처리할 수 있습니다.
	//    예: if (client != null) { System.out.println("Client Name: " + client.getName()); }
	return "someView"; // 3. 뷰 이름을 반환합니다.
}
```

**코드 분석:**

1. `@GetMapping("/")`: 루트 경로(`/`)로 GET 요청이 오면 `handle` 메소드가 처리합니다.
2. `@RequestAttribute Client client`:
  - `@RequestAttribute`: "현재 HTTP 요청의 요청 범위(request scope)에서 속성을 찾아서..."
  - `Client client`: "... 그 속성의 이름이 `client` (매개변수 이름과 동일하게 기본 설정됨)인 것을 찾아 `Client` 타입의 `client` 매개변수에 넣어줘!" 라는 의미입니다.
  - 만약 속성 이름이 매개변수 이름과 다르다면 `@RequestAttribute("실제속성이름") Client client` 와 같이 명시적으로 지정할 수 있습니다. 예를 들어, 요청 속성에 `loggedInUser`라는 이름으로 `Client` 객체가 저장되어 있다면, `@RequestAttribute("loggedInUser") Client client` 와 같이 사용합니다.
  - 이 `Client` 객체는 이 `handle` 메소드가 호출되기 *이전에* 어떤 필터나 인터셉터에서 `request.setAttribute("client", new Client(...));` 와 같은 코드로 요청 범위에 저장해두었을 것입니다.
3. 메소드 본문 `// ...`: 이 부분에서는 주입받은 `client` 객체를 사용하여 필요한 로직을 수행합니다. 예를 들어, 클라이언트 정보를 화면에 표시하거나, 클라이언트 등급에 따라 다른 서비스를 제공하는 등의 작업을 할 수 있습니다.
4. `return "someView";`: 이 메소드는 뷰의 이름을 반환하고, 스프링 MVC는 해당 뷰를 렌더링하여 클라이언트에게 응답합니다. 이 뷰에서도 요청 속성에 접근하여 `client` 정보를 사용할 수 있습니다 (예: JSP라면 `<%= request.getAttribute("client") %>` 또는 JSTL/EL을 사용).

**어떻게 요청 속성이 설정되는가? (가상 시나리오)**

실제로 `Client` 객체가 요청 속성에 어떻게 저장될 수 있는지 간단한 인터셉터 예시를 들어보겠습니다. (인터셉터 설정은 지금 다루지 않고, 개념만 이해하시면 됩니다.)

```java
// 간단한 예시용 HandlerInterceptor
public class ClientInfoInterceptor implements HandlerInterceptor {

    @Override
    public boolean preHandle(HttpServletRequest request, HttpServletResponse response, Object handler) throws Exception {
        // 실제로는 DB 조회, 인증 시스템 연동 등을 통해 Client 정보를 가져옴
        Client client = new Client("John Doe", "Premium"); // 예시 Client 객체 생성

        // 요청 속성에 "client"라는 이름으로 Client 객체를 저장
        request.setAttribute("client", client);
        System.out.println("ClientInfoInterceptor: 'client' attribute set.");
        return true; // true를 반환하면 다음 단계(컨트롤러 등)로 요청 처리 계속
    }
}

```

위와 같은 인터셉터가 스프링 MVC에 등록되어 있고, `/` 경로의 요청에 대해 동작한다면, `ClientInfoInterceptor`의 `preHandle` 메소드가 컨트롤러의 `handle` 메소드보다 먼저 실행됩니다. 이때 `request.setAttribute("client", client);` 코드를 통해 `Client` 객체가 요청 속성에 저장되고, 이후 컨트롤러의 `handle` 메소드에서 `@RequestAttribute Client client`를 통해 이 객체를 성공적으로 주입받을 수 있는 것입니다.

### **"왜?" 라는 질문에 대한 답변**

**`@RequestAttribute`는 왜 필요할까요? 어떤 문제를 해결하기 위해 등장했을까요?**

1. **요청 처리 단계 간 데이터 공유의 표준화된 방법:** 하나의 HTTP 요청은 여러 컴포넌트(필터, 인터셉터, 컨트롤러, 뷰)를 거쳐 처리될 수 있습니다. 이러한 단계들 사이에서 데이터를 안전하고 일관되게 공유할 방법이 필요합니다. `HttpServletRequest`의 `setAttribute`/`getAttribute`는 이를 위한 표준적인 방법이지만, `@RequestAttribute`는 스프링 MVC 환경에서 이를 더욱 선언적이고 편리하게 사용할 수 있도록 해줍니다.
2. **컨트롤러와 전처리/후처리 로직의 분리:** 인증, 로깅, 공통 데이터 준비 등의 로직은 특정 컨트롤러에만 국한되지 않는 경우가 많습니다. 이러한 공통 로직을 필터나 인터셉터로 분리하고, 처리 결과를 요청 속성에 담아 컨트롤러에 전달하면, 컨트롤러는 핵심 비즈니스 로직에만 집중할 수 있게 됩니다. 이는 **관심사의 분리(Separation of Concerns)**라는 좋은 설계 원칙을 따르는 것입니다.
3. **타입 안정성 및 편의성 증대:** `request.getAttribute("name")`은 `Object` 타입을 반환하므로, 개발자가 직접 원하는 타입으로 형 변환(`(Client) request.getAttribute("client")`)을 해야 합니다. 이 과정에서 `ClassCastException`이 발생할 수 있습니다. `@RequestAttribute Client client`처럼 사용하면 스프링이 타입 변환을 (가능하다면) 시도하고, 타입이 맞지 않으면 오류를 발생시켜 문제를 더 빨리 발견할 수 있게 해줍니다. 또한, 형 변환 코드가 사라져 코드가 더 깔끔해집니다.
4. **테스트 용이성:** 컨트롤러가 `HttpServletRequest`에 직접 의존하는 것보다, `@RequestAttribute`를 통해 필요한 데이터만 주입받으면, 해당 데이터를 모의(mock) 객체로 만들어 테스트하기가 더 수월해집니다.

결국 `@RequestAttribute`는 요청 처리 과정에서 컴포넌트 간 데이터 전달을 더 스프링답고, 안전하며, 편리하게 만들어주는 역할을 합니다.

### **주의사항 및 Best Practice**

1. **`@RequestAttribute`는 요청 외부에서 오는 데이터를 위한 것이 아님:** `@RequestHeader`, `@RequestParam`, `@CookieValue`, `@RequestBody` 등은 클라이언트가 HTTP 요청을 통해 직접 전달하는 데이터를 받기 위한 어노테이션입니다. 반면, `@RequestAttribute`는 서버 내부의 다른 컴포넌트(필터, 인터셉터 등)가 **미리 요청 범위에 설정해 둔 데이터를 접근**하기 위한 것입니다. 이 차이점을 명확히 이해하는 것이 중요합니다.
2. **속성 이름 일치:**
  - 기본적으로 `@RequestAttribute`는 매개변수의 이름을 요청 속성의 이름으로 사용합니다.
  - 만약 요청 속성의 이름과 매개변수의 이름이 다르다면, `@RequestAttribute("실제속성이름")`과 같이 `name` (또는 `value`) 속성을 사용하여 명시적으로 지정해야 합니다.

      ```java
      // 요청 속성 이름이 "currentUser"인 경우
      public String handleUser(@RequestAttribute("currentUser") User user) { ... }
      ```

3. **필수 속성 vs 선택적 속성:**
  - 기본적으로 `@RequestAttribute`로 지정된 속성이 요청 범위에 없으면 스프링 MVC는 오류를 발생시킵니다.
  - 만약 속성이 선택적이라면 `required` 속성을 `false`로 설정할 수 있습니다. 이때 해당 속성이 없으면 매개변수에는 `null`이 주입됩니다.

      ```java
      @GetMapping("/optionalData")
      public String handleOptionalData(@RequestAttribute(name = "optionalInfo", required = false) String info) {
          if (info != null) {
              // info 사용 로직
          } else {
              // info가 없을 때의 로직
          }
          return "viewName";
      }
      ```

  - `@CookieValue`나 `@RequestHeader`와 달리 `@RequestAttribute`에는 `defaultValue` 속성이 없습니다. 선택적 속성이고 값이 없을 때 `null` 대신 기본값을 사용하고 싶다면, 메소드 내부에서 직접 처리해야 합니다.
4. **데이터의 생명주기 이해:** 요청 속성에 저장된 데이터는 해당 HTTP 요청이 처리되는 동안에만 유효합니다. 응답이 클라이언트에게 전달되고 나면 해당 데이터는 사라집니다. 만약 여러 요청에 걸쳐 데이터를 유지해야 한다면 세션(session)이나 다른 저장소를 사용해야 합니다.
5. **과도한 사용 지양:** 요청 속성은 편리하지만, 너무 많은 데이터를 요청 속성을 통해 전달하는 것은 코드의 흐름을 파악하기 어렵게 만들 수 있습니다. 꼭 필요한 데이터만, 명확한 이름으로 전달하는 것이 좋습니다.

### **이전 학습 내용과의 연관성**

- **`@RequestHeader`, `@CookieValue` 와의 차이점:**
  - 이전 챕터에서 배운 `@RequestHeader`와 `@CookieValue`는 클라이언트가 HTTP 요청 시 **외부에서 직접 전달하는 정보(헤더, 쿠키)**를 컨트롤러로 가져오는 방법입니다.
  - 반면, `@RequestAttribute`는 **서버 내부에서 요청 처리 흐름 중에 다른 컴포넌트에 의해 미리 준비되어 요청 객체 자체에 저장된 정보**를 가져오는 방법입니다. 데이터의 출처가 다릅니다.
- **스프링 MVC 처리 흐름의 이해:** `@RequestAttribute`를 이해하려면 스프링 MVC의 요청 처리 흐름(request processing lifecycle)에 대한 그림을 그릴 수 있어야 합니다. 클라이언트 요청이 들어오면, 서블릿 필터, 디스패처 서블릿, 핸들러 인터셉터, 컨트롤러, 뷰 등으로 이어지는 과정을 상상할 수 있어야 "아, 인터셉터에서 요청 속성을 세팅하면 컨트롤러에서 `@RequestAttribute`로 받을 수 있겠구나" 하고 연결 지을 수 있습니다.
- **`@SessionAttribute` 와의 비교 (미리보기):**
  - 원문에서 `@SessionAttribute`와 유사하다고 언급했습니다. 둘 다 서버 측에 저장된 데이터를 가져온다는 점은 비슷합니다.
  - 가장 큰 차이는 **데이터의 생명주기(scope)**입니다. `@RequestAttribute`는 **단일 요청 동안**만 유효한 데이터를 다루고, `@SessionAttribute`는 **여러 요청에 걸쳐 (사용자 세션 동안)** 유효한 데이터를 다룹니다. 마치 `@RequestAttribute`는 당일치기 여행 가방이고, `@SessionAttribute`는 좀 더 긴 여행을 위한 큰 트렁크와 같다고 비유할 수 있습니다.

---
