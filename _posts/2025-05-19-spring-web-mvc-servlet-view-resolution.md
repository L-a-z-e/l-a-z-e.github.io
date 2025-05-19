---
title: Spring Web MVC - Dispatcher Servlet (View Resolution)
description: 
author: laze
date: 2025-05-19 00:00:03 +0900
categories: [Dev, SpringWebMVC]
tags: [SpringWebMVC]
---
**뷰 리졸루션 (View Resolution)**

Spring MVC는 특정 뷰 기술에 얽매이지 않고 브라우저에 모델을 렌더링할 수 있도록 하는 `ViewResolver`와 `View` 인터페이스를 정의합니다.

`ViewResolver`는 뷰 이름과 실제 뷰 간의 매핑을 제공합니다.

`View`는 특정 뷰 기술로 넘어가기 전 데이터 준비를 담당합니다.

다음 표는 `ViewResolver` 계층 구조에 대한 자세한 내용을 제공합니다:

**표 1. `ViewResolver` 구현체들**

| `ViewResolver` | 설명 |
| --- | --- |
| `AbstractCachingViewResolver` | `AbstractCachingViewResolver`의 하위 클래스들은 그들이 해석한 뷰 인스턴스를 캐시합니다. 캐싱은 특정 뷰 기술의 성능을 향상시킵니다. `cache` 속성을 `false`로 설정하여 캐시를 끌 수 있습니다. 또한, 런타임에 특정 뷰를 새로 고쳐야 하는 경우(예: FreeMarker 템플릿이 수정된 경우), `removeFromCache(String viewName, Locale loc)` 메소드를 사용할 수 있습니다. |
| `UrlBasedViewResolver` | 명시적인 매핑 정의 없이 논리적 뷰 이름을 URL로 직접 해석하는 `ViewResolver` 인터페이스의 간단한 구현체입니다. 이는 논리적 이름이 임의의 매핑 없이 간단한 방식으로 뷰 리소스의 이름과 일치하는 경우에 적합합니다. |
| `InternalResourceViewResolver` | `InternalResourceView`(실질적으로 서블릿 및 JSP) 및 `JstlView`와 같은 하위 클래스를 지원하는 `UrlBasedViewResolver`의 편리한 하위 클래스입니다. `setViewClass(..)`를 사용하여 이 리졸버에 의해 생성된 모든 뷰에 대한 뷰 클래스를 지정할 수 있습니다. 자세한 내용은 `UrlBasedViewResolver` javadoc을 참조하세요. |
| `FreeMarkerViewResolver` | `FreeMarkerView` 및 그들의 사용자 정의 하위 클래스를 지원하는 `UrlBasedViewResolver`의 편리한 하위 클래스입니다. |
| `ContentNegotiatingViewResolver` | 요청 파일 이름 또는 `Accept` 헤더를 기반으로 뷰를 해석하는 `ViewResolver` 인터페이스의 구현체입니다. |
| `BeanNameViewResolver` | 뷰 이름을 현재 애플리케이션 컨텍스트의 빈 이름으로 해석하는 `ViewResolver` 인터페이스의 구현체입니다. 이는 서로 다른 뷰 이름을 기반으로 다양한 뷰 타입을 혼합하고 일치시킬 수 있는 매우 유연한 변형입니다. 각 `View`는 예를 들어 XML이나 설정 클래스에서 빈으로 정의될 수 있습니다. |

**처리 (Handling)**

여러 개의 리졸버 빈을 선언하고, 필요한 경우 `order` 속성을 설정하여 순서를 지정함으로써 뷰 리졸버를 체인으로 연결할 수 있습니다.

기억하세요, `order` 속성 값이 높을수록 뷰 리졸버는 체인에서 나중에 위치합니다.

`ViewResolver`의 계약은 뷰를 찾을 수 없음을 나타내기 위해 `null`을 반환할 수 있다고 명시합니다.

그러나 JSP와 `InternalResourceViewResolver`의 경우, JSP가 존재하는지 알아내는 유일한 방법은 `RequestDispatcher`를 통해 디스패치를 수행하는 것입니다.

따라서 `InternalResourceViewResolver`는 항상 전체 뷰 리졸버 순서에서 마지막에 구성해야 합니다.

뷰 리졸루션 설정은 Spring 설정에 `ViewResolver` 빈을 추가하는 것만큼 간단합니다.

MVC 설정(MVC Config)은 뷰 리졸버 및 컨트롤러 로직 없이 HTML 템플릿 렌더링에 유용한 로직 없는 뷰 컨트롤러(View Controllers) 추가를 위한 전용 설정 API를 제공합니다.

**리다이렉팅 (Redirecting)**

Reactive 스택에서의 동등한 내용을 확인하세요.

뷰 이름의 특별한 `redirect:` 접두사를 사용하면 리다이렉트를 수행할 수 있습니다.

`UrlBasedViewResolver` (및 그 하위 클래스)는 이를 리다이렉트가 필요하다는 지시로 인식합니다.

나머지 뷰 이름은 리다이렉트 URL입니다.

그 순 효과는 컨트롤러가 `RedirectView`를 반환한 것과 동일하지만, 이제 컨트롤러 자체는 논리적 뷰 이름으로 작동할 수 있습니다.

`redirect:/myapp/some/resource`와 같은 논리적 뷰 이름은 현재 서블릿 컨텍스트에 상대적으로 리다이렉트되는 반면, `redirect:<https://myhost.com/some/arbitrary/path`와> 같은 이름은 절대 URL로 리다이렉트됩니다.

**포워딩 (Forwarding)**
결국 `UrlBasedViewResolver` 및 하위 클래스에 의해 해석되는 뷰 이름에 대해 특별한 `forward:` 접두사를 사용할 수도 있습니다.

이는 `RequestDispatcher.forward()`를 수행하는 `InternalResourceView`를 생성합니다.

따라서 이 접두사는 `InternalResourceViewResolver` 및 `InternalResourceView`(JSP용)와 함께 사용하는 데는 유용하지 않지만,

다른 뷰 기술을 사용하지만 여전히 서블릿/JSP 엔진에 의해 처리될 리소스의 포워드를 강제하고 싶을 때 도움이 될 수 있습니다. 대신 여러 뷰 리졸버를 체인으로 연결할 수도 있습니다.

**콘텐츠 협상 (Content Negotiation)**

`ContentNegotiatingViewResolver`는 자체적으로 뷰를 해석하지 않고, 다른 뷰 리졸버에 위임하고 클라이언트가 요청한 표현과 유사한 뷰를 선택합니다.

표현은 `Accept` 헤더 또는 쿼리 파라미터(예: `/path?format=pdf`)에서 결정될 수 있습니다.

`ContentNegotiatingViewResolver`는 요청 미디어 타입과 각 `ViewResolver`와 연관된 `View`가 지원하는 미디어 타입(콘텐츠 타입으로도 알려짐)을 비교하여 요청을 처리할 적절한 `View`를 선택합니다.

호환되는 `Content-Type`을 가진 목록의 첫 번째 `View`가 클라이언트에 표현을 반환합니다.

`ViewResolver` 체인에서 호환 가능한 뷰를 제공할 수 없는 경우, `DefaultViews` 속성을 통해 지정된 뷰 목록을 참조합니다.

이 후자의 옵션은 논리적 뷰 이름에 관계없이 현재 리소스의 적절한 표현을 렌더링할 수 있는 단일 `View`에 적합합니다.

`Accept` 헤더는 와일드카드(예: `text/*`)를 포함할 수 있으며, 이 경우 `Content-Type`이 `text/xml`인 `View`는 호환되는 일치 항목입니다.

---

## Spring MVC의 화면 마법사: ViewResolver와 View 이해하기

Spring MVC의 핵심 목표 중 하나는 **특정 뷰 기술(View Technology, 예: JSP, Thymeleaf, FreeMarker 등)에 대한 의존성을 낮추는 것**입니다.

즉, 컨트롤러는 자신이 어떤 뷰 기술을 사용하여 화면을 그릴지 몰라도 되도록 설계되었습니다.

이를 가능하게 하는 두 가지 핵심 인터페이스가 바로 `ViewResolver`와 `View`입니다.

**기본 흐름:**

1. **컨트롤러:** 요청 처리를 완료하고, 렌더링할 뷰의 **논리적인 이름(Logical View Name)**을 문자열 형태로 반환합니다. (예: `"home"`, `"user/profile"`, `"product/list"`) 또는 `ModelAndView` 객체에 뷰 이름과 모델 데이터를 담아 반환합니다.
2. **`DispatcherServlet`:** 컨트롤러로부터 이 논리적 뷰 이름을 받습니다.
3. **`ViewResolver`:** `DispatcherServlet`은 등록된 `ViewResolver`들에게 이 논리적 뷰 이름을 전달하여, 실제 렌더링을 수행할 **`View` 객체**를 찾아달라고 요청합니다.
4. **`View`:** `ViewResolver`가 찾아낸 `View` 객체는 모델 데이터와 함께 특정 뷰 기술을 사용하여 최종 응답(보통 HTML)을 생성하고 클라이언트에게 전송합니다.

### 1. `View` 인터페이스

- **역할:** 특정 뷰 기술을 사용하여 모델 데이터를 렌더링하는 실제 작업을 담당합니다.
- **주요 메소드:** `render(Map<String, ?> model, HttpServletRequest request, HttpServletResponse response) throws Exception`
  - 이 메소드는 모델 데이터(`model`), 현재 요청/응답 객체를 받아 실제 뷰를 렌더링합니다.
- **구현체 예시:** `JstlView` (JSP용), `ThymeleafView` (Thymeleaf용), `FreeMarkerView` (FreeMarker용) 등. 각 뷰 기술마다 해당 기술을 사용하여 렌더링하는 `View` 구현체가 존재합니다.

### 2. `ViewResolver` 인터페이스

- **역할:** 컨트롤러가 반환한 **논리적 뷰 이름**을 실제 **`View` 객체로 변환(resolve)**하는 역할을 합니다.
- **주요 메소드:** `resolveViewName(String viewName, Locale locale) throws Exception`
  - 이 메소드는 논리적 뷰 이름(`viewName`)과 로케일 정보를 받아, 해당하는 `View` 객체를 반환합니다. 만약 해당 뷰를 찾을 수 없으면 `null`을 반환할 수 있습니다.
- Spring MVC는 다양한 `ViewResolver` 구현체들을 제공하여 여러 뷰 기술을 지원하고 유연한 설정을 가능하게 합니다.

### 3. 주요 `ViewResolver` 구현체들

| `ViewResolver` 구현체 | 주요 특징 및 용도 |
| --- | --- |
| `InternalResourceViewResolver` | **(JSP/Servlet 사용 시 가장 흔하게 사용)** `UrlBasedViewResolver`의 하위 클래스로, 주로 JSP나 서블릿과 같은 내부 리소스(Internal Resource)를 뷰로 사용하도록 지원합니다. `prefix` (경로 접두사)와 `suffix` (경로 접미사)를 설정하여 논리적 뷰 이름을 실제 파일 경로로 변환합니다. (예: 뷰 이름 "home" -> `/WEB-INF/views/home.jsp`) |
| `UrlBasedViewResolver` | 논리적 뷰 이름을 특정 URL로 직접 매핑하는 간단한 방식입니다. `InternalResourceViewResolver`, `FreeMarkerViewResolver` 등의 부모 클래스입니다. |
| `AbstractCachingViewResolver` | 해석된 `View` 인스턴스를 캐싱하여 성능을 향상시키는 `ViewResolver`들의 부모 클래스입니다. 대부분의 `ViewResolver` 구현체들이 이를 상속받습니다. 캐시를 사용하지 않도록 설정하거나, 런타임에 특정 뷰 캐시를 제거할 수 있습니다. |
| `FreeMarkerViewResolver` | FreeMarker 템플릿을 뷰로 사용하도록 지원하는 `UrlBasedViewResolver`의 하위 클래스입니다. |
| `ThymeleafViewResolver` (별도 의존성) | Thymeleaf 템플릿을 뷰로 사용하도록 지원합니다. (Spring MVC 코어에는 포함되지 않으며, `thymeleaf-spring` 라이브러리 추가 필요) |
| `ContentNegotiatingViewResolver` | **(하나의 URL로 다양한 응답 포맷 지원 시 중요)** 자체적으로 뷰를 해석하지 않고, 다른 `ViewResolver`들에게 위임합니다. 클라이언트의 요청 헤더(`Accept` 헤더)나 URL 파라미터(`?format=json`)를 보고, 클라이언트가 원하는 응답 포맷(HTML, JSON, XML, PDF 등)에 가장 적합한 뷰를 선택합니다. (예: `Accept: application/json`이면 JSON 뷰, `Accept: text/html`이면 HTML 뷰 선택) |
| `BeanNameViewResolver` | 논리적 뷰 이름을 Spring 애플리케이션 컨텍스트에 **빈(Bean)으로 등록된 `View` 객체의 이름**으로 간주하여 해당 빈을 사용합니다. 매우 유연하며, 다양한 종류의 `View` 객체를 빈으로 정의하고 이름으로 호출하여 사용할 수 있습니다. |

### 4. 뷰 리졸버 체인 (View Resolver Chaining)

애플리케이션에는 여러 개의 `ViewResolver`를 등록하고 체인으로 연결할 수 있습니다.

- **설정:** MVC Config에 여러 `ViewResolver` 빈을 선언하고, `order` 프로퍼티를 사용하여 실행 순서를 지정합니다. `order` 값이 낮을수록 먼저 시도됩니다 (원문에는 "높을수록 나중에 위치한다"고 되어 있으니, 결국 같은 의미입니다. 보통 0부터 시작하여 작은 숫자가 우선순위가 높습니다.)
- **동작 방식:**
  1. `DispatcherServlet`은 첫 번째 `ViewResolver`(가장 낮은 `order` 값)에게 뷰 해석을 요청합니다.
  2. 만약 해당 `ViewResolver`가 `View` 객체를 찾아 반환하면, 그 `View`를 사용하고 체인은 종료됩니다.
  3. 만약 `null`을 반환하면 (해당 뷰를 처리할 수 없다는 의미), 다음 순서의 `ViewResolver`에게 요청이 넘어갑니다.
  4. 모든 `ViewResolver`가 `null`을 반환하면, `DispatcherServlet`은 뷰를 찾지 못한 것으로 간주하고 예외를 발생시킬 수 있습니다.
- **`InternalResourceViewResolver`의 특별한 주의사항:**
  - `InternalResourceViewResolver`(주로 JSP용)는 해당 JSP 파일이 실제로 존재하는지 확인하는 유일한 방법이 `RequestDispatcher`를 통해 내부적으로 디스패치(포워드)를 시도해보는 것입니다.
  - 만약 `InternalResourceViewResolver`가 체인의 앞쪽에 있고 JSP를 찾지 못했을 때 `null`을 반환하는 대신, "일단 내가 처리하겠다"고 판단하고 내부 디스패치를 시도했는데 실제 파일이 없다면, 그 이후의 다른 `ViewResolver`들은 실행될 기회조차 얻지 못하고 404 에러가 발생할 수 있습니다.
  - 따라서, **`InternalResourceViewResolver`는 항상 뷰 리졸버 체인의 가장 마지막 순서로 설정해야 합니다.** 다른 `ViewResolver`들이 먼저 자신들이 처리할 수 있는 뷰인지 확인하고, 모두 처리하지 못했을 때 마지막으로 `InternalResourceViewResolver`가 시도하도록 하는 것입니다.

**예시 (Java Config - `WebMvcConfigurer`):**

```java
@Configuration
@EnableWebMvc
public class WebConfig implements WebMvcConfigurer {

    // ThymeleafViewResolver (order = 0, 기본값이라 생략 가능)
    @Bean
    public ThymeleafViewResolver thymeleafViewResolver() {
        ThymeleafViewResolver resolver = new ThymeleafViewResolver();
        // ... Thymeleaf 설정 ...
        resolver.setOrder(0); // 명시적으로 지정 (다른 리졸버와 순서 조정 시)
        return resolver;
    }

    // BeanNameViewResolver (order = 1)
    @Bean
    public BeanNameViewResolver beanNameViewResolver() {
        BeanNameViewResolver resolver = new BeanNameViewResolver();
        resolver.setOrder(1);
        return resolver;
    }

    // Excel 파일을 생성하는 커스텀 View 빈 (BeanNameViewResolver가 사용)
    @Bean(name = "excelReportView") // 이 이름으로 BeanNameViewResolver가 찾음
    public View excelReportView() {
        return new MyExcelView(); // AbstractXlsxView 등을 상속한 커스텀 뷰
    }

    // InternalResourceViewResolver (가장 마지막 순서, order = 2)
    @Bean
    public InternalResourceViewResolver internalResourceViewResolver() {
        InternalResourceViewResolver resolver = new InternalResourceViewResolver();
        resolver.setPrefix("/WEB-INF/views/");
        resolver.setSuffix(".jsp");
        resolver.setOrder(2); // 항상 다른 리졸버들보다 뒤에 오도록
        return resolver;
    }
}
```

### 5. 특별한 뷰 이름 접두사: `redirect:`와 `forward:`

`UrlBasedViewResolver`와 그 하위 클래스들(예: `InternalResourceViewResolver`)은 뷰 이름에 특정 접두사가 사용되면 특별한 방식으로 처리합니다.

- **`redirect:` 접두사:**
  - **기능:** 클라이언트에게 HTTP 리다이렉트 응답(302 상태 코드와 `Location` 헤더)을 보냅니다.
  - **사용법:** 컨트롤러가 뷰 이름으로 `"redirect:/some/other/url"` 와 같이 반환합니다.
  - **동작:** `UrlBasedViewResolver`는 이 접두사를 보고 `RedirectView` 객체를 생성하여 사용합니다. `DispatcherServlet`은 이 `RedirectView`를 통해 리다이렉트를 수행합니다.
  - **URL 종류:**
    - `redirect:/myapp/resource`: 현재 서블릿 컨텍스트에 상대적인 경로로 리다이렉트. (실제 URL은 `http://host/contextPath/myapp/resource`)
    - `redirect:<http://example.com/absolute/url`:> 완전한 절대 URL로 리다이렉트.
  - **장점:** 컨트롤러는 실제 `RedirectView` 객체를 직접 생성할 필요 없이, 논리적인 뷰 이름만으로 리다이렉트를 지시할 수 있습니다. (PRG 패턴 구현 시 유용)
- **`forward:` 접두사:**
  - **기능:** 현재 요청을 서버 내부의 다른 URL로 포워드(forward)합니다. `RequestDispatcher.forward()`를 사용하는 것과 같습니다.
  - **사용법:** 컨트롤러가 뷰 이름으로 `"forward:/some/internal/path"` 와 같이 반환합니다.
  - **동작:** `UrlBasedViewResolver`는 `InternalResourceView`를 생성하여 내부 포워드를 수행합니다.
  - **주의:** 이 접두사는 `InternalResourceViewResolver`와 함께 사용하는 것은 별 의미가 없습니다. 왜냐하면 `InternalResourceViewResolver` 자체가 이미 내부 리소스(JSP 등)로 포워드하는 방식으로 동작하기 때문입니다.
  - **용도:** 주로 다른 뷰 기술(예: Thymeleaf, FreeMarker)을 사용하면서, 특정 경우에만 서블릿/JSP 엔진이 처리하는 리소스(예: 다른 서블릿)로 요청을 넘기고 싶을 때 제한적으로 사용될 수 있습니다. 하지만 대부분의 경우 뷰 리졸버 체인을 구성하여 해결하는 것이 더 일반적입니다.

### 6. `ContentNegotiatingViewResolver`: 하나의 URL, 다양한 표현

이 `ViewResolver`는 매우 강력하며, 클라이언트의 요청(주로 `Accept` HTTP 헤더 또는 URL 파라미터)에 따라 동일한 데이터에 대해 다른 형식(HTML, JSON, XML, PDF 등)으로 응답을 제공할 수 있게 해줍니다. (RESTful API 설계 시 매우 유용)

- **동작 방식:**
  1. 자체적으로 뷰를 해석하지 않습니다.
  2. 대신, 체인에 등록된 다른 `ViewResolver`들에게 요청을 위임합니다.
  3. 클라이언트가 요청한 미디어 타입(예: `application/json`, `text/html`)과 각 `ViewResolver`가 제공할 수 있는 뷰의 미디어 타입을 비교합니다.
  4. 가장 적합한(클라이언트가 선호하는) 미디어 타입을 제공하는 `View`를 선택하여 반환합니다.
  5. 만약 적합한 뷰를 찾지 못하면, `DefaultViews` 프로퍼티에 설정된 기본 뷰를 사용할 수도 있습니다.

**예시:**

- 컨트롤러가 `/users/1` 요청에 대해 사용자 정보를 모델에 담아 논리적 뷰 이름 `"userDetail"`을 반환.
- 클라이언트가 `Accept: application/json` 헤더와 함께 요청: `ContentNegotiatingViewResolver`는 JSON을 생성할 수 있는 `View` (예: `MappingJackson2JsonView`를 사용하는 `BeanNameViewResolver`나 특정 설정이 된 `ViewResolver`가 찾아줌)를 선택.
- 클라이언트가 `Accept: text/html` 헤더와 함께 요청: HTML을 생성할 수 있는 `View` (예: JSP를 사용하는 `InternalResourceViewResolver`가 찾아줌)를 선택.

---
