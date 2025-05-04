---
title: Spring Framework Classpath Scanning and Managed Components
description: 
author: laze
date: 2025-05-04 00:00:03 +0900
categories: [Dev, SpringBoot]
tags: [SpringBoot]
---
**Classpath Scanning and Managed Components**

**@Component 및 추가 스테레오타입 어노테이션 (@Component and Further Stereotype Annotations)**

`@Repository` 어노테이션은 리포지토리(데이터 접근 객체 또는 DAO라고도 함)의 역할이나 스테레오타입(stereotype)을 충족하는 모든 클래스에 대한 마커입니다.

이 마커의 용도 중 하나는 예외 변환(Exception Translation)에서 설명된 대로 예외의 자동 변환입니다.

스프링은 추가적인 스테레오타입 어노테이션을 제공합니다: `@Component`, `@Service`, `@Controller`. `@Component`는 스프링 관리 컴포넌트에 대한 일반적인 스테레오타입입니다.

`@Repository`, `@Service`, `@Controller`는 더 구체적인 사용 사례(각각 영속성, 서비스, 프리젠테이션 계층)를 위한 `@Component`의 특수화입니다.

따라서 컴포넌트 클래스에 `@Component`로 어노테이션을 달 수 있지만, 대신 `@Repository`, `@Service` 또는 `@Controller`로 어노테이션을 달면 클래스가 도구에 의한 처리나 애스펙트와의 연관에 더 적합해집니다.

예를 들어, 이러한 스테레오타입 어노테이션은 포인트컷(pointcuts)의 이상적인 대상이 됩니다.

`@Repository`, `@Service`, `@Controller`는 스프링 프레임워크의 향후 릴리스에서 추가적인 의미를 가질 수도 있습니다.

따라서 서비스 계층에 `@Component` 또는 `@Service` 사용 중에서 선택하는 경우, `@Service`가 분명히 더 나은 선택입니다.

유사하게, 앞서 언급했듯이 `@Repository`는 이미 영속성 계층에서 자동 예외 변환을 위한 마커로 지원됩니다.

**메타 어노테이션 및 조합 어노테이션 사용하기 (Using Meta-annotations and Composed Annotations)**

스프링에서 제공하는 많은 어노테이션은 사용자 코드에서 메타 어노테이션(meta-annotations)으로 사용될 수 있습니다.

메타 어노테이션은 다른 어노테이션에 적용될 수 있는 어노테이션입니다. 예를 들어, 앞서 언급한 `@Service` 어노테이션은 다음 예제와 같이 `@Component`로 메타 어노테이션되어 있습니다:

```java
// Java
@Target(ElementType.TYPE)
@Retention(RetentionPolicy.RUNTIME)
@Documented
@Component // ①
public @interface Service {

	// ...
}
```

① `@Component`는 `@Service`가 `@Component`와 동일한 방식으로 처리되도록 합니다.

```kotlin
// Kotlin
@Target(AnnotationTarget.CLASS)
@Retention(AnnotationRetention.RUNTIME)
@MustBeDocumented
@Component // ①
annotation class Service {
    // ...
}
```

메타 어노테이션을 결합하여 "조합 어노테이션(composed annotations)"을 생성할 수도 있습니다.

예를 들어, 스프링 MVC의 `@RestController` 어노테이션은 `@Controller`와 `@ResponseBody`로 구성됩니다.

또한, 조합 어노테이션은 선택적으로 메타 어노테이션의 속성을 재선언하여 사용자 정의를 허용할 수 있습니다.

이는 메타 어노테이션 속성의 하위 집합만 노출하고 싶을 때 특히 유용할 수 있습니다.

예를 들어, 스프링의 `@SessionScope` 어노테이션은 스코프 이름을 `session`으로 하드코딩하지만 여전히 `proxyMode`의 사용자 정의를 허용합니다. 다음 목록은 `SessionScope` 어노테이션의 정의를 보여줍니다:

```java
// Java
@Target({ElementType.TYPE, ElementType.METHOD})
@Retention(RetentionPolicy.RUNTIME)
@Documented
@Scope(WebApplicationContext.SCOPE_SESSION)
public @interface SessionScope {

	/**
	 * {@link Scope#proxyMode}의 별칭.
	 * <p>기본값은 {@link ScopedProxyMode#TARGET_CLASS}.
	 */
	@AliasFor(annotation = Scope.class)
	ScopedProxyMode proxyMode() default ScopedProxyMode.TARGET_CLASS;

}
```

```kotlin
// Kotlin
@Target(AnnotationTarget.CLASS, AnnotationTarget.FUNCTION)
@Retention(AnnotationRetention.RUNTIME)
@MustBeDocumented
@Scope(WebApplicationContext.SCOPE_SESSION)
annotation class SessionScope(
    /**
     * {@link Scope#proxyMode}의 별칭.
     * <p>기본값은 {@link ScopedProxyMode#TARGET_CLASS}.
     */
    @get:AliasFor(annotation = Scope::class) // Use annotation use-site target
    val proxyMode: ScopedProxyMode = ScopedProxyMode.TARGET_CLASS
)
```

그런 다음 다음과 같이 `proxyMode`를 선언하지 않고 `@SessionScope`를 사용할 수 있습니다:

```java
// Java
@Service
@SessionScope
public class SessionScopedService {
	// ...
}
```

```kotlin
// Kotlin
@Service
@SessionScope
class SessionScopedService {
    // ...
}
```

다음 예제와 같이 `proxyMode` 값을 오버라이드할 수도 있습니다:

```java
// Java
@Service
@SessionScope(proxyMode = ScopedProxyMode.INTERFACES)
public class SessionScopedUserService implements UserService {
	// ...
}
```

```kotlin
// Kotlin
@Service
@SessionScope(proxyMode = ScopedProxyMode.INTERFACES)
class SessionScopedUserService : UserService {
    // ...
}
```

**클래스 자동 감지 및 빈 정의 등록 (Automatically Detecting Classes and Registering Bean Definitions)**

스프링은 스테레오타입 클래스를 자동으로 감지하고 해당 `BeanDefinition` 인스턴스를 `ApplicationContext`에 등록할 수 있습니다.

예를 들어, 다음 두 클래스는 이러한 자동 감지 대상입니다:

```java
// Java
@Service
public class SimpleMovieLister {

	private MovieFinder movieFinder;

	public SimpleMovieLister(MovieFinder movieFinder) {
		this.movieFinder = movieFinder;
	}
}
```

```kotlin
// Kotlin
@Service
class SimpleMovieLister(private val movieFinder: MovieFinder)
```

```java
// Java
@Repository
public class JpaMovieFinder implements MovieFinder {
	// 명확성을 위해 구현 생략
}
```

```kotlin
// Kotlin
@Repository
class JpaMovieFinder : MovieFinder {
    // 명확성을 위해 구현 생략
}
```

이러한 클래스를 자동 감지하고 해당 빈을 등록하려면 `@Configuration` 클래스에 `@ComponentScan`을 추가해야 하며, 여기서 `basePackages` 속성은 두 클래스의 공통 상위 패키지입니다.

(또는 각 클래스의 상위 패키지를 포함하는 쉼표, 세미콜론 또는 공백으로 구분된 목록을 지정할 수 있습니다.)

```java
// Java
@Configuration
@ComponentScan(basePackages = "org.example")
public class AppConfig  {
	// ...
}
```

```kotlin
// Kotlin
@Configuration
@ComponentScan(basePackages = ["org.example"]) // Use array syntax in Kotlin
class AppConfig {
    // ...
}
```

*간결함을 위해 앞의 예제는 어노테이션의 `value` 속성을 사용할 수 있었습니다 (즉, `@ComponentScan("org.example")`).*

```xml
<?xml version="1.0" encoding="UTF-8"?>
<beans xmlns="<http://www.springframework.org/schema/beans>"
	xmlns:xsi="<http://www.w3.org/2001/XMLSchema-instance>"
	xmlns:context="<http://www.springframework.org/schema/context>"
	xsi:schemaLocation="<http://www.springframework.org/schema/beans>
		<https://www.springframework.org/schema/beans/spring-beans.xsd>
		<http://www.springframework.org/schema/context>
		<https://www.springframework.org/schema/context/spring-context.xsd>">

	<context:component-scan base-package="org.example"/>

</beans>
```

`*<context:component-scan>` 사용은 암묵적으로 `<context:annotation-config>`의 기능을 활성화합니다.*

`*<context:component-scan>`을 사용할 때 `<context:annotation-config>` 요소를 포함할 필요는 일반적으로 없습니다.*

*클래스패스 패키지 스캔은 클래스패스에 해당 디렉토리 항목이 있어야 합니다.*

*Ant로 JAR을 빌드할 때 JAR 태스크의 `files-only` 스위치를 활성화하지 않도록 하십시오.*

*또한 일부 환경의 보안 정책에 따라 클래스패스 디렉토리가 노출되지 않을 수 있습니다 - 예를 들어, JDK 1.7.0_45 이상에서의 독립 실행형 앱 (매니페스트에 'Trusted-Library' 설정 필요*

*모듈 경로(Java Module System)에서 스프링의 클래스패스 스캔은 일반적으로 예상대로 작동합니다.*

*그러나 컴포넌트 클래스가 `module-info` 기술자(descriptor)에 `exports`되어 있는지 확인하십시오.*

*스프링이 클래스의 비-public 멤버를 호출할 것으로 예상하는 경우, 해당 멤버가 '열려'(opened) 있는지 확인하십시오 (즉, `module-info` 기술자에서 `exports` 선언 대신 `opens` 선언 사용).*

또한, `AutowiredAnnotationBeanPostProcessor`와 `CommonAnnotationBeanPostProcessor`는 `component-scan` 요소를 사용할 때 모두 암묵적으로 포함됩니다.

이는 즉, 두 컴포넌트가 자동 감지되고 함께 연결된다는 것을 의미합니다 - 모두 XML에서 제공되는 빈 구성 메타데이터 없이.

`*annotation-config` 속성을 `false` 값으로 포함하여 `AutowiredAnnotationBeanPostProcessor` 및 `CommonAnnotationBeanPostProcessor`의 등록을 비활성화할 수 있습니다.*

**필터를 사용하여 스캔 사용자 정의하기 (Using Filters to Customize Scanning)**

기본적으로 `@Component`, `@Repository`, `@Service`, `@Controller`, `@Configuration` 또는 `@Component`로 어노테이션된 커스텀 어노테이션으로 어노테이션된 클래스만이 감지된 후보 컴포넌트입니다.

그러나 커스텀 필터를 적용하여 이 동작을 수정하고 확장할 수 있습니다.

`@ComponentScan` 어노테이션의 `includeFilters` 또는 `excludeFilters` 속성으로 추가하거나 (또는 XML 구성의 `<context:component-scan>` 요소의 `<context:include-filter />` 또는 `<context:exclude-filter />` 자식 요소로) 추가합니다. 각 필터 요소는 `type` 및 `expression` 속성을 요구합니다. 다음 표는 필터링 옵션을 설명합니다:

표 1. 필터 타입

| 필터 타입 | 예제 표현식 | 설명 |
| --- | --- | --- |
| `annotation` (기본값) | `org.example.SomeAnnotation` | 대상 컴포넌트의 타입 레벨에 존재하거나 메타-존재하는 어노테이션. |
| `assignable` | `org.example.SomeClass` | 대상 컴포넌트가 할당 가능한(확장 또는 구현하는) 클래스 (또는 인터페이스). |
| `aspectj` | `org.example..*Service+` | 대상 컴포넌트와 일치하는 AspectJ 타입 표현식. |
| `regex` | `org\\.example\\.Default.*` | 대상 컴포넌트의 클래스 이름과 일치하는 정규식 표현식. |
| `custom` | `org.example.MyTypeFilter` | `org.springframework.core.type.TypeFilter` 인터페이스의 커스텀 구현. |

다음 예제는 모든 `@Repository` 어노테이션을 무시하고 대신 "stub" 리포지토리를 사용하는 구성을 보여줍니다:

```java
// Java
@Configuration
@ComponentScan(basePackages = "org.example",
		includeFilters = @Filter(type = FilterType.REGEX, pattern = ".*Stub.*Repository"),
		excludeFilters = @Filter(Repository.class))
public class AppConfig {
	// ...
}
```

```kotlin
// Kotlin
@Configuration
@ComponentScan(
    basePackages = ["org.example"],
    includeFilters = [ComponentScan.Filter(type = FilterType.REGEX, pattern = [".*Stub.*Repository"])],
    excludeFilters = [ComponentScan.Filter(type = FilterType.ANNOTATION, classes = [Repository::class])]
)
class AppConfig {
    // ...
}
```

XML로 작성시

```xml
<beans>
	<context:component-scan base-package="org.example">
		<context:include-filter type="regex"
				expression=".*Stub.*Repository"/>
		<context:exclude-filter type="annotation"
				expression="org.springframework.stereotype.Repository"/>
	</context:component-scan>
</beans>
```

*어노테이션에서 `useDefaultFilters=false`를 설정하거나 `<component-scan/>` 요소의 속성으로 `use-default-filters="false"`를 제공하여 기본 필터를 비활성화할 수도 있습니다.*

*이는 `@Component`, `@Repository`, `@Service`, `@Controller`, `@RestController` 또는 `@Configuration`으로 어노테이션되거나 메타-어노테이션된 클래스의 자동 감지를 효과적으로 비활성화합니다.*

**컴포넌트 내에서 빈 메타데이터 정의하기 (Defining Bean Metadata within Components)**

스프링 컴포넌트는 또한 컨테이너에 빈 정의 메타데이터를 기여할 수 있습니다.

`@Configuration` 어노테이션된 클래스 내에서 빈 메타데이터를 정의하는 데 사용되는 동일한 `@Bean` 어노테이션으로 이를 수행할 수 있습니다. 다음 예제는 이를 수행하는 방법을 보여줍니다:

```java
// Java
@Component
public class FactoryMethodComponent {

	@Bean
	@Qualifier("public")
	public TestBean publicInstance() {
		return new TestBean("publicInstance");
	}

	public void doWork() {
		// 컴포넌트 메소드 구현 생략
	}
}
```

```kotlin
// Kotlin
@Component
class FactoryMethodComponent {

    @Bean
    @Qualifier("public")
    fun publicInstance(): TestBean {
        return TestBean("publicInstance")
    }

    fun doWork() {
        // 컴포넌트 메소드 구현 생략
    }
}
```

앞의 클래스는 `doWork()` 메소드에 애플리케이션 특정 코드를 가진 스프링 컴포넌트입니다.

그러나 `publicInstance()` 메소드를 참조하는 팩토리 메소드를 가진 빈 정의도 기여합니다.

`@Bean` 어노테이션은 팩토리 메소드 및 `@Qualifier` 어노테이션을 통한 퀄리파이어 값과 같은 다른 빈 정의 속성을 식별합니다.

지정할 수 있는 다른 메소드 레벨 어노테이션은 `@Scope`, `@Lazy` 및 커스텀 퀄리파이어 어노테이션입니다.

*컴포넌트 초기화 역할 외에도, `@Autowired` 또는 `@Inject`로 표시된 주입 지점에 `@Lazy` 어노테이션을 배치할 수도 있습니다.*

*이 컨텍스트에서는 지연 해석 프록시(lazy-resolution proxy) 주입으로 이어집니다.*

*그러나 이러한 프록시 접근 방식은 다소 제한적입니다.*

*정교한 지연 상호 작용, 특히 선택적 의존성과 결합된 경우에는 대신 `ObjectProvider<MyTargetBean>`을 권장합니다.*

자동 와이어링된 필드와 메소드가 지원되며, `@Bean` 메소드의 자동 와이어링에 대한 추가 지원이 있습니다.

```java
// Java
@Component
public class FactoryMethodComponent {

	private static int i;

	@Bean
	@Qualifier("public")
	public TestBean publicInstance() {
		return new TestBean("publicInstance");
	}

	// 커스텀 퀄리파이어 사용 및 메소드 파라미터 자동 와이어링
	@Bean
	protected TestBean protectedInstance(
			@Qualifier("public") TestBean spouse,
			@Value("#{privateInstance.age}") String country) {
		TestBean tb = new TestBean("protectedInstance", 1);
		tb.setSpouse(spouse);
		tb.setCountry(country);
		return tb;
	}

	@Bean
	private TestBean privateInstance() {
		return new TestBean("privateInstance", i++);
	}

	@Bean
	@RequestScope
	public TestBean requestScopedInstance() {
		return new TestBean("requestScopedInstance", 3);
	}
}
```

```kotlin
// Kotlin
@Component
class FactoryMethodComponent {
    companion object { // For static field 'i'
        private var i = 0
    }

    @Bean
    @Qualifier("public")
    fun publicInstance(): TestBean {
        return TestBean("publicInstance")
    }

    // 커스텀 퀄리파이어 사용 및 메소드 파라미터 자동 와이어링
    @Bean
    protected fun protectedInstance(
        @Qualifier("public") spouse: TestBean,
        @Value("#{privateInstance.age}") country: String // Assuming TestBean has age property
    ): TestBean {
        val tb = TestBean("protectedInstance", 1)
        tb.spouse = spouse
        tb.country = country
        return tb
    }

    @Bean
    private fun privateInstance(): TestBean {
        return TestBean("privateInstance", i++)
    }

    @Bean
    @RequestScope
    fun requestScopedInstance(): TestBean {
        return TestBean("requestScopedInstance", 3)
    }
}
// Assuming TestBean class exists with necessary properties and constructors
```

이 예제는 문자열 메소드 파라미터 `country`를 `privateInstance`라는 다른 빈의 `age` 속성 값으로 자동 와이어링합니다.

스프링 표현 언어(Spring Expression Language) 요소는 `#{ <expression> }` 표기법을 통해 속성 값을 정의합니다.

`@Value` 어노테이션의 경우, 표현식 텍스트를 해석할 때 빈 이름을 찾도록 표현식 해석기(expression resolver)가 미리 구성됩니다.

스프링 프레임워크 4.3부터, 현재 빈 생성을 트리거하는 요청 주입 지점에 접근하기 위해 `InjectionPoint` 타입(또는 더 구체적인 하위 클래스: `DependencyDescriptor`)의 팩토리 메소드 파라미터를 선언할 수도 있습니다.

이는 기존 인스턴스 주입이 아닌 실제 빈 인스턴스 생성에만 적용된다는 점에 유의하십시오.

결과적으로 이 기능은 프로토타입 스코프 빈에 가장 적합합니다.

다른 스코프의 경우, 팩토리 메소드는 주어진 스코프에서 새 빈 인스턴스 생성을 트리거한 주입 지점(예: 지연 싱글톤 빈 생성을 트리거한 의존성)만 볼 수 있습니다.

이러한 시나리오에서는 제공된 주입 지점 메타데이터를 의미론적 주의와 함께 사용할 수 있습니다.

다음 예제는 `InjectionPoint` 사용 방법을 보여줍니다:

```java
// Java
@Component
public class FactoryMethodComponent {

	@Bean @Scope("prototype")
	public TestBean prototypeInstance(InjectionPoint injectionPoint) {
		return new TestBean("prototypeInstance for " + injectionPoint.getMember());
	}
}
```

```kotlin
// Kotlin
@Component
class FactoryMethodComponent {

    @Bean @Scope("prototype")
    fun prototypeInstance(injectionPoint: InjectionPoint): TestBean {
        // injectionPoint.member could be Field or Method
        val memberInfo = injectionPoint.member?.toString() ?: "unknown"
        return TestBean("prototypeInstance for $memberInfo")
    }
}
// Assuming TestBean class exists
```

일반 스프링 컴포넌트의 `@Bean` 메소드는 스프링 `@Configuration` 클래스 내부의 상대 메소드와 다르게 처리됩니다.

차이점은 `@Component` 클래스는 메소드 및 필드 호출을 가로채기(intercept) 위해 CGLIB로 향상(enhanced)되지 않는다는 것입니다.

CGLIB 프록시는 `@Configuration` 클래스의 `@Bean` 메소드 내에서 메소드나 필드를 호출하는 것이 협력 객체에 대한 빈 메타데이터 참조를 생성하는 수단입니다.

이러한 메소드는 일반적인 자바 의미론으로 호출되지 않고, `@Bean` 메소드에 대한 프로그래밍 방식 호출을 통해 다른 빈을 참조할 때조차도 스프링 빈의 일반적인 생명주기 관리 및 프록시를 제공하기 위해 컨테이너를 통과합니다.

대조적으로, 일반 `@Component` 클래스의 `@Bean` 메소드에서 메소드나 필드를 호출하는 것은 표준 자바 의미론을 가지며, 특별한 CGLIB 처리나 다른 제약 조건이 적용되지 않습니다.

`*@Bean` 메소드를 `static`으로 선언하여 포함하는 구성 클래스를 인스턴스로 생성하지 않고 호출할 수 있습니다.*

*이는 특히 후처리기 빈(예: `BeanFactoryPostProcessor` 또는 `BeanPostProcessor` 타입)을 정의할 때 의미가 있는데, 이러한 빈은 컨테이너 생명주기 초기에 초기화되고 해당 시점에 구성의 다른 부분을 트리거하는 것을 피해야 하기 때문입니다.*

*정적 `@Bean` 메소드 호출은 기술적 한계로 인해 `@Configuration` 클래스 내에서도 (이 섹션 앞부분에서 설명한 대로) 컨테이너에 의해 절대 가로채지지 않습니다: CGLIB 하위 클래스 생성은 비-정적 메소드만 오버라이드할 수 있습니다.*

*결과적으로, 다른 `@Bean` 메소드에 대한 직접 호출은 표준 자바 의미론을 가지며, 팩토리 메소드 자체에서 직접 독립적인 인스턴스가 반환됩니다.*

`*@Bean` 메소드의 자바 언어 가시성(visibility)은 스프링 컨테이너의 결과적인 빈 정의에 즉각적인 영향을 미치지 않습니다.*

*비-`@Configuration` 클래스 및 어디서든 정적 메소드에 대해 원하는 대로 자유롭게 팩토리 메소드를 선언할 수 있습니다.*

*그러나 `@Configuration` 클래스의 일반 `@Bean` 메소드는 오버라이드 가능해야 합니다 - 즉, `private` 또는 `final`로 선언되어서는 안 됩니다.*

`*@Bean` 메소드는 주어진 컴포넌트 또는 구성 클래스의 기본 클래스와 컴포넌트 또는 구성 클래스에 의해 구현된 인터페이스에 선언된 자바 8 기본 메소드에서도 발견됩니다.*

*이는 스프링 4.2부터 자바 8 기본 메소드를 통해 다중 상속까지 가능한 복잡한 구성 배열을 구성하는 데 많은 유연성을 제공합니다.*

*마지막으로, 단일 클래스는 런타임 시 사용 가능한 의존성에 따라 사용할 여러 팩토리 메소드의 배열로서 동일한 빈에 대해 여러 `@Bean` 메소드를 보유할 수 있습니다.*

*이는 다른 구성 시나리오에서 "가장 탐욕스러운(greediest)" 생성자 또는 팩토리 메소드를 선택하는 것과 동일한 알고리즘입니다:*

*생성 시점에 만족 가능한 의존성 수가 가장 많은 변형이 선택되며, 이는 컨테이너가 여러 `@Autowired` 생성자 중에서 선택하는 방식과 유사합니다.*

**자동 감지된 컴포넌트 이름 지정 (Naming Autodetected Components)**

컴포넌트가 스캔 프로세스의 일부로 자동 감지될 때, 해당 빈 이름은 해당 스캐너에 알려진 `BeanNameGenerator` 전략에 의해 생성됩니다.

기본적으로 `AnnotationBeanNameGenerator`가 사용됩니다.

스프링 스테레오타입 어노테이션의 경우, 어노테이션의 `value` 속성을 통해 이름을 제공하면 해당 이름이 해당 빈 정의의 이름으로 사용됩니다.

이 관례는 스프링 스테레오타입 어노테이션 대신 다음 JSR-250 및 JSR-330 어노테이션을 사용할 때도 적용됩니다: `@jakarta.annotation.ManagedBean`, `@javax.annotation.ManagedBean`, `@jakarta.inject.Named`, `@javax.inject.Named`.

*스프링 프레임워크 6.1부터 빈 이름을 지정하는 데 사용되는 어노테이션 속성의 이름은 더 이상 `value`일 필요가 없습니다.*

*커스텀 스테레오타입 어노테이션은 다른 이름(예: `name`)을 가진 속성을 선언하고 해당 속성에 `@AliasFor(annotation = Component.class, attribute = "value")`로 어노테이션을 달 수 있습니다.*

*결과적으로, 커스텀 스테레오타입 어노테이션은 `@Component`의 `value` 속성에 대한 명시적 별칭을 선언하기 위해 `@AliasFor`를 사용해야 합니다.*

*구체적인 예는 `Repository#value()` 및 `ControllerAdvice#name()`의 소스 코드 선언을 참조하십시오.*

이러한 어노테이션 또는 다른 감지된 컴포넌트(예: 커스텀 필터에 의해 발견된 것)에서 명시적인 빈 이름을 파생할 수 없는 경우, 기본 빈 이름 생성기는 대문자로 시작하지 않는 비-정규화(non-qualified) 클래스 이름을 반환합니다.

예를 들어, 다음 컴포넌트 클래스가 감지되면 이름은 `myMovieLister`와 `movieFinderImpl`이 됩니다.

```java
// Java
@Service("myMovieLister")
public class SimpleMovieLister {
	// ...
}
```

```kotlin
// Kotlin
@Service("myMovieLister")
class SimpleMovieLister {
    // ...
}
```

```java
// Java
@Repository
public class MovieFinderImpl implements MovieFinder {
	// ...
}
```

```kotlin
// Kotlin
@Repository // Default name will be 'movieFinderImpl'
class MovieFinderImpl : MovieFinder {
    // ...
}
```

기본 빈 이름 지정 전략에 의존하고 싶지 않다면, 커스텀 빈 이름 지정 전략을 제공할 수 있습니다.

먼저, `BeanNameGenerator` 인터페이스를 구현하고 기본 인수 없는 생성자를 포함해야 합니다.

그런 다음 스캐너를 구성할 때 정규화된 클래스 이름을 제공합니다.

*이름이 동일하지만 다른 패키지에 있는 여러 자동 감지된 컴포넌트로 인해 이름 충돌이 발생하는 경우, 생성된 빈 이름에 대해 정규화된 클래스 이름을 기본값으로 사용하는 `BeanNameGenerator`를 구성해야 할 수 있습니다. `org.springframework.context.annotation` 패키지에 있는 `FullyQualifiedAnnotationBeanNameGenerator`를 이러한 목적으로 사용할 수 있습니다.*

```java
// Java
@Configuration
@ComponentScan(basePackages = "org.example", nameGenerator = MyNameGenerator.class)
public class AppConfig {
	// ...
}
```

```kotlin
// Kotlin
@Configuration
@ComponentScan(basePackages = ["org.example"], nameGenerator = MyNameGenerator::class) // Use ::class in Kotlin
class AppConfig {
    // ...
}
// Assuming MyNameGenerator exists and implements BeanNameGenerator
```

```xml
<beans>
	<context:component-scan base-package="org.example"
		name-generator="org.example.MyNameGenerator" />
</beans>
```

일반적으로 다른 컴포넌트가 명시적으로 참조할 수 있는 경우 어노테이션으로 이름을 지정하는 것을 고려하십시오.

반면에 컨테이너가 와이어링을 책임지는 경우에는 자동 생성된 이름으로 충분합니다.

**자동 감지된 컴포넌트에 스코프 제공하기 (Providing a Scope for Autodetected Components)**

일반적으로 스프링 관리 컴포넌트와 마찬가지로, 자동 감지된 컴포넌트의 기본이자 가장 일반적인 스코프는 `singleton`입니다.

그러나 때로는 `@Scope` 어노테이션으로 지정할 수 있는 다른 스코프가 필요합니다. 다음 예제와 같이 어노테이션 내에 스코프 이름을 제공할 수 있습니다:

```java
// Java
@Scope("prototype")
@Repository
public class MovieFinderImpl implements MovieFinder {
	// ...
}
```

```kotlin
// Kotlin
@Scope("prototype")
@Repository
class MovieFinderImpl : MovieFinder {
    // ...
}
```

`*@Scope` 어노테이션은 구체적인 빈 클래스(어노테이션된 컴포넌트의 경우) 또는 팩토리 메소드(`@Bean` 메소드의 경우)에서만 내부 검사(introspected)됩니다.*

*XML 빈 정의와 대조적으로 빈 정의 상속 개념이 없으며, 클래스 레벨의 상속 계층 구조는 메타데이터 목적상 관련이 없습니다.*

스프링 컨텍스트에서 "request" 또는 "session"과 같은 웹 특정 스코프에 대한 자세한 내용은 Request, Session, Application 및 WebSocket 스코프를 참조하십시오.

해당 스코프에 대한 사전 빌드된 어노테이션과 마찬가지로, 스프링의 메타 어노테이션 접근 방식을 사용하여 자신만의 스코핑 어노테이션을 구성할 수도 있습니다:

예를 들어, `@Scope("prototype")`으로 메타 어노테이션된 커스텀 어노테이션, 잠재적으로 커스텀 스코프 프록시 모드도 선언 가능.

어노테이션 기반 접근 방식에 의존하는 대신 스코프 해석을 위한 커스텀 전략을 제공하려면 `ScopeMetadataResolver` 인터페이스를 구현할 수 있습니다.

기본 인수 없는 생성자를 포함해야 합니다.

그런 다음 어노테이션과 빈 정의 모두의 다음 예제와 같이 스캐너를 구성할 때 정규화된 클래스 이름을 제공할 수 있습니다:

```java
// Java
@Configuration
@ComponentScan(basePackages = "org.example", scopeResolver = MyScopeResolver.class)
public class AppConfig {
	// ...
}
```

```kotlin
// Kotlin
@Configuration
@ComponentScan(basePackages = ["org.example"], scopeResolver = MyScopeResolver::class) // Use ::class
class AppConfig {
    // ...
}
// Assuming MyScopeResolver exists and implements ScopeMetadataResolver
```

```xml
<beans>
	<context:component-scan base-package="org.example" scope-resolver="org.example.MyScopeResolver"/>
</beans>
```

특정 비-싱글톤 스코프를 사용할 때, 스코프 객체에 대한 프록시를 생성해야 할 수 있습니다.

그 이유는 의존성으로서의 스코프 빈(Scoped Beans as Dependencies)에 설명되어 있습니다.

이 목적을 위해 `component-scan` 요소에 `scoped-proxy` 속성을 사용할 수 있습니다.

세 가지 가능한 값은 `no`, `interfaces`, `targetClass`입니다. 예를 들어, 다음 구성은 표준 JDK 동적 프록시를 생성합니다:

```java
// Java
@Configuration
@ComponentScan(basePackages = "org.example", scopedProxy = ScopedProxyMode.INTERFACES)
public class AppConfig {
	// ...
}
```

```kotlin
// Kotlin
@Configuration
@ComponentScan(basePackages = ["org.example"], scopedProxy = ScopedProxyMode.INTERFACES)
class AppConfig {
    // ...
}
```

```xml
<beans>
	<context:component-scan base-package="org.example" scoped-proxy="interfaces"/>
</beans>
```

**어노테이션으로 퀄리파이어 메타데이터 제공하기 (Providing Qualifier Metadata with Annotations)**

`@Qualifier` 어노테이션은 `@Qualifier`로 어노테이션 기반 자동 와이어링 미세 조정(Fine-tuning Annotation-based Autowiring with Qualifiers)에서 논의됩니다.

해당 섹션의 예제는 자동 와이어링 후보를 해석할 때 세분화된 제어를 제공하기 위해 `@Qualifier` 어노테이션과 커스텀 퀄리파이어 어노테이션 사용을 보여줍니다.

해당 예제는 XML 빈 정의를 기반으로 했기 때문에, 퀄리파이어 메타데이터는 XML의 `bean` 요소의 `qualifier` 또는 `meta` 자식 요소를 사용하여 후보 빈 정의에 제공되었습니다.

컴포넌트의 자동 감지를 위해 클래스패스 스캐닝에 의존할 때, 후보 클래스의 타입 레벨 어노테이션으로 퀄리파이어 메타데이터를 제공할 수 있습니다. 다음 세 가지 예제는 이 기술을 보여줍니다:

```java
// Java
@Component
@Qualifier("Action")
public class ActionMovieCatalog implements MovieCatalog {
	// ...
}
```

```kotlin
// Kotlin
@Component
@Qualifier("Action")
class ActionMovieCatalog : MovieCatalog {
    // ...
}
```

```java
// Java
@Component
@Genre("Action") // Assuming @Genre is a custom qualifier annotation
public class ActionMovieCatalog implements MovieCatalog {
	// ...
}
```

```kotlin
// Kotlin
@Component
@Genre("Action") // Assuming @Genre is a custom qualifier annotation
class ActionMovieCatalog : MovieCatalog {
    // ...
}
```

```java
// Java
@Component
@Offline // Assuming @Offline is a custom qualifier annotation without a value
public class CachingMovieCatalog implements MovieCatalog {
	// ...
}
```

```kotlin
// Kotlin
@Component
@Offline // Assuming @Offline is a custom qualifier annotation without a value
class CachingMovieCatalog : MovieCatalog {
    // ...
}
```

*대부분의 어노테이션 기반 대안과 마찬가지로, 어노테이션 메타데이터는 클래스 정의 자체에 바인딩되는 반면, XML 사용은 동일한 타입의 여러 빈이 퀄리파이어 메타데이터에서 변형을 제공할 수 있도록 한다는 점을 명심하십시오.*

*왜냐하면 해당 메타데이터는 클래스별이 아닌 인스턴스별로 제공되기 때문입니다.*

---

**전체 주제: 클래스패스 스캐닝과 관리 컴포넌트 (Classpath Scanning and Managed Components)**

이 부분은 XML에 `<bean>` 태그를 일일이 적는 대신, 스프링이 **지정된 폴더(패키지)를 자동으로 탐색(스캔)** 하여 특정 어노테이션(`@Component`, `@Service` 등)이 붙은 클래스를 **스스로 찾아서 스프링 빈으로 등록**하게 만드는 방법에 대한 내용입니다. XML 설정을 획기적으로 줄여주는 강력한 기능이죠.

**핵심 아이디어:** 개발자는 클래스에 특정 어노테이션만 붙여주고 어디를 스캔할지만 알려주면, 스프링이 알아서 빈으로 만들어 관리하게 하자!

---

**첫 번째 파트: 배경 및 기본 개념**

- **XML의 한계:** 모든 빈을 XML에 등록하는 방식은 프로젝트가 커지면 설정 파일이 매우 길고 관리하기 어려워집니다. 빈을 하나 추가할 때마다 XML도 수정해야 합니다.
- **어노테이션의 발전:** 이전 섹션에서 배운 `@Autowired`, `@PostConstruct` 등은 의존성 주입이나 콜백을 위한 어노테이션이었습니다. 하지만 여전히 이 빈들을 스프링 컨테이너에 등록하는 과정(예: `<bean>` 정의)은 필요했습니다.
- **클래스패스 스캐닝:** 이 문제를 해결하기 위해 스프링은 **클래스패스(Classpath)** 를 스캔하여 **"후보 컴포넌트(Candidate Components)"** 를 자동으로 찾아내는 기능을 제공합니다.
  - **후보 컴포넌트:** 특정 조건(예: 특정 어노테이션이 붙어 있음)을 만족하여 스프링 빈으로 등록될 자격이 있는 클래스들을 의미합니다.
- **목표:** XML 없이, 어노테이션과 스캐닝 설정만으로 빈을 등록하고 관리하자!
- **대안 (Java Config):** XML 대신 순수 자바 코드로 빈을 설정하는 방법(`@Configuration`, `@Bean` 등)도 있다는 점을 언급합니다. 스캐닝은 이 Java Config 방식과 함께 많이 사용됩니다.

---

**두 번째 파트: `@Component` 및 추가 스테레오타입 어노테이션**

이 부분은 스프링이 스캔할 때 어떤 클래스를 빈으로 등록할지 **표시(Marking)** 하는 데 사용되는 주요 어노테이션들을 설명합니다. 이를 **스테레오타입(Stereotype)** 어노테이션이라고 부릅니다.

- **`@Component`:**
  - 가장 **기본적이고 일반적인** 스테레오타입 어노테이션입니다.
  - "이 클래스는 스프링이 관리해야 할 컴포넌트(구성 요소, 즉 빈)입니다" 라는 의미를 가집니다.
  - 특별한 역할 구분이 없는 일반적인 빈에 사용합니다.
- **특수화된 스테레오타입들 (`@Repository`, `@Service`, `@Controller`):**
  - 이 어노테이션들은 기술적으로 `@Component`와 **동일한 역할**(스캔 대상 표시)을 하지만, 클래스가 애플리케이션의 특정 **계층(Layer)** 이나 **역할(Role)** 을 담당한다는 **추가적인 의미**를 부여합니다.
  - **`@Repository`:** 주로 **데이터 접근 계층(DAO)** 에서 사용됩니다. 이 어노테이션이 붙은 빈에서는 스프링이 데이터 접근 시 발생하는 특정 예외들을 스프링 표준 예외(`DataAccessException`)로 자동 변환해주는 부가 기능이 있습니다.
  - **`@Service`:** 주로 **비즈니스 로직**을 처리하는 **서비스 계층**에서 사용됩니다. 현재로서는 특별한 추가 기능은 없지만, 가독성과 역할 구분을 위해 사용합니다. (미래에 기능 추가 가능성 있음)
  - **`@Controller`:** 주로 **프리젠테이션 계층(웹 요청 처리)** 에서 사용됩니다. 스프링 MVC 프레임워크에서 웹 컨트롤러임을 나타냅니다. (`@RestController`는 `@Controller` + `@ResponseBody`가 합쳐진 것)
- **왜 구분해서 쓸까?**
  - **가독성 및 의도 명확화:** 클래스의 역할을 명확히 드러냅니다.
  - **부가 기능:** `@Repository`처럼 특정 기능이 연관될 수 있습니다.
  - **AOP 대상 지정 용이:** 특정 계층의 빈들(예: 모든 `@Service` 빈)에 공통적인 부가 기능(AOP)을 적용할 때 편리합니다.
  - **결론:** 그냥 `@Component`를 써도 동작은 하지만, 각 계층에 맞는 어노테이션을 사용하는 것이 더 좋은 설계입니다.

---

**세 번째 파트: 메타 어노테이션 및 조합 어노테이션 사용하기**

이 부분은 스프링 어노테이션들을 **재사용**하거나 **조합**하여 커스텀 어노테이션을 만드는 방법에 대해 설명합니다.

- **메타 어노테이션:** 다른 어노테이션 위에 붙어서 그 어노테이션의 성격을 규정하는 어노테이션입니다.
  - **예시:** `@Service` 어노테이션 정의를 보면 그 위에 `@Component`가 붙어 있습니다. 이는 `@Service`가 `@Component`의 특성을 그대로 물려받아 스캔 대상이 됨을 의미합니다.
- **조합 어노테이션 (Composed Annotation):** 여러 메타 어노테이션을 조합하여 새로운 의미를 가진 어노테이션을 만드는 것입니다.
  - **예시:** 스프링 MVC의 `@RestController`는 `@Controller`와 `@ResponseBody`라는 두 어노테이션이 합쳐진 조합 어노테이션입니다.
- **속성 재선언 및 커스터마이징:** 조합 어노테이션을 만들 때, 원본 메타 어노테이션의 속성 중 일부를 다시 선언하여 기본값을 변경하거나 사용자 정의를 허용할 수 있습니다. `@AliasFor` 어노테이션을 사용하여 속성 간의 관계를 명시합니다.
  - **예시:** `@SessionScope` 어노테이션은 내부적으로 `@Scope("session")`을 사용하지만, `proxyMode` 속성은 개발자가 변경할 수 있도록 노출되어 있습니다.

---

**네 번째 파트: 클래스 자동 감지 및 빈 정의 등록**

이 부분은 실제로 클래스패스 스캐닝을 어떻게 설정하고 사용하는지에 대해 설명합니다.

- **자동 감지 대상:** `@Component`, `@Repository`, `@Service`, `@Controller` 등의 스테레오타입 어노테이션이 붙은 클래스들.
- **설정 방법:**
  - **Java Config:** `@Configuration` 클래스에 `@ComponentScan(basePackages = "스캔시작패키지")` 어노테이션을 추가합니다. `basePackages`에는 스캔을 시작할 최상위 패키지 이름을 지정합니다. (여러 개 지정 가능) `value` 속성으로 간단히 지정할 수도 있습니다 (`@ComponentScan("패키지")`).
  - **XML Config:** XML 설정 파일에 `<context:component-scan base-package="스캔시작패키지"/>` 태그를 추가합니다.
- **자동 어노테이션 처리 활성화:** **중요!** `@ComponentScan` (Java Config) 또는 `<context:component-scan>` (XML)을 사용하면, 이전에 설명한 `<context:annotation-config/>`의 기능(즉, `@Autowired`, `@PostConstruct` 등을 처리하는 `BeanPostProcessor` 등록)이 **자동으로 활성화됩니다.** 따라서 컴포넌트 스캔을 사용한다면 `<context:annotation-config/>`를 **별도로 추가할 필요가 없습니다.**
- **스캔 제약 조건:** 클래스패스 스캐닝이 제대로 동작하려면 해당 클래스 파일들이 클래스패스 상에 실제로 존재해야 하며, 특정 환경(보안 정책, 모듈 시스템)에서는 추가 설정이 필요할 수 있습니다.

---

**다섯 번째 파트: 필터를 사용하여 스캔 사용자 정의하기**

이 부분은 기본 스테레오타입 어노테이션 외에 추가적인 조건으로 스캔 대상을 포함시키거나 제외하는 방법에 대해 설명합니다.

- **기본 필터:** 기본적으로는 앞서 언급한 스테레오타입 어노테이션들이 붙은 클래스만 스캔합니다.
- **커스텀 필터:** `@ComponentScan`의 `includeFilters` / `excludeFilters` 속성 (또는 XML의 `<context:include-filter/>` / `<context:exclude-filter/>` 자식 태그)를 사용하여 필터를 추가할 수 있습니다.
- **필터 타입:**
  - `ANNOTATION` (기본): 특정 어노테이션이 붙은 클래스.
  - `ASSIGNABLE_TYPE`: 특정 클래스/인터페이스를 상속/구현하는 클래스.
  - `ASPECTJ`: AspectJ 패턴 표현식과 일치하는 클래스.
  - `REGEX`: 클래스 이름이 정규 표현식과 일치하는 클래스.
  - `CUSTOM`: 개발자가 직접 만든 `TypeFilter` 구현 클래스.
- **사용 예:** `@Repository`는 제외하고, 클래스 이름에 "Stub"이 포함된 것만 스캔하도록 필터를 설정할 수 있습니다.
- **기본 필터 비활성화:** `useDefaultFilters=false` 옵션을 주면 기본 스테레오타입 감지를 끄고 오직 커스텀 필터만 사용하게 할 수도 있습니다.

---

**여섯 번째 파트: 컴포넌트 내에서 빈 메타데이터 정의하기 (`@Bean` in `@Component`)**

이 부분은 `@Component` (또는 `@Service` 등)로 표시된 일반 컴포넌트 클래스 안에서도 `@Configuration` 클래스처럼 `@Bean` 메소드를 사용하여 추가적인 빈을 정의할 수 있다는 내용입니다.

- **사용법:** `@Component` 클래스 내부에 `@Bean` 어노테이션이 붙은 메소드를 만들면, 그 메소드가 반환하는 객체가 스프링 빈으로 등록됩니다. `@Qualifier`, `@Scope`, `@Lazy` 등 다른 빈 설정 어노테이션도 `@Bean` 메소드 위에 함께 사용할 수 있습니다.
- **@Bean 메소드 내 자동 와이어링:** `@Bean` 메소드의 파라미터에 `@Autowired`, `@Qualifier`, `@Value` 등을 사용하여 다른 빈이나 값을 주입받을 수 있습니다. SpEL (`#{...}`) 사용도 가능합니다.
- **`InjectionPoint` 파라미터 (프로토타입):** 프로토타입 스코프의 `@Bean` 메소드는 `InjectionPoint` 타입의 파라미터를 선언하여, 현재 이 빈 생성을 요청한 곳(주입 지점)에 대한 정보를 얻을 수 있습니다.
- **`@Configuration`과의 차이점 (중요!):**
  - `@Configuration` 클래스: 내부적으로 CGLIB 프록시로 강화(enhanced)됩니다. `@Bean` 메소드 간의 호출이 스프링 컨테이너를 통해 이루어져 싱글톤 스코프 등을 보장합니다. `@Bean` 메소드는 `private`이나 `final`이면 안 됩니다.
  - `@Component` 클래스: 기본적으로 CGLIB 강화가 없습니다. `@Bean` 메소드 간의 호출은 일반적인 자바 메소드 호출처럼 동작합니다. 특별한 제약 조건이 없습니다.
- **`static @Bean` 메소드:** `@Bean` 메소드를 `static`으로 선언하면, 해당 메소드가 속한 컴포넌트 클래스의 인스턴스 생성 없이도 빈을 정의할 수 있습니다. 특히 `BeanFactoryPostProcessor`나 `BeanPostProcessor`처럼 컨테이너 생명주기 초기에 생성되어야 하는 빈을 정의할 때 유용합니다.

---

**일곱 번째 파트: 자동 감지된 컴포넌트 이름 지정**

이 부분은 클래스패스 스캔으로 자동 등록된 빈들이 어떤 이름(ID)을 갖게 되는지에 대한 규칙을 설명합니다.

- **기본 전략 (`AnnotationBeanNameGenerator`):**
  1. 스테레오타입 어노테이션(`@Component`, `@Service`, `@Named` 등)에 `value` 속성(또는 `@AliasFor("value")` 처리된 속성)으로 이름이 지정되어 있으면 그 이름을 사용합니다. (예: `@Service("myMovieLister")` -> 빈 이름: `myMovieLister`)
  2. 이름이 지정되지 않았으면, 클래스 이름의 첫 글자를 소문자로 바꾼 이름을 사용합니다. (예: `SimpleMovieLister` 클래스 -> 빈 이름: `simpleMovieLister`)
- **커스텀 전략:** `BeanNameGenerator` 인터페이스를 구현하고 `@ComponentScan` 또는 `<context:component-scan>`에 `nameGenerator` 속성으로 지정하여 빈 이름 생성 규칙을 직접 만들 수 있습니다. (예: 이름 충돌 방지를 위해 패키지명을 포함시키는 `FullyQualifiedAnnotationBeanNameGenerator`)
- **언제 이름을 지정할까?** 다른 빈에서 명시적으로 이름으로 참조해야 할 경우 어노테이션에 이름을 지정하는 것이 좋습니다. 단순히 자동 와이어링으로만 사용된다면 자동 생성된 이름으로도 충분합니다.

---

**여덟 번째 파트: 자동 감지된 컴포넌트에 스코프 제공하기**

이 부분은 자동 감지된 빈의 스코프(기본값: 싱글톤)를 변경하는 방법을 설명합니다.

- **`@Scope` 어노테이션:** 빈으로 등록될 클래스 위에 `@Scope("스코프이름")` 어노테이션을 붙여서 스코프를 지정합니다. (예: `@Scope("prototype")`)
- **웹 스코프:** "request", "session" 등의 웹 스코프도 지정 가능합니다. 스프링은 `@RequestScope`, `@SessionScope` 같은 편리한 커스텀 스코프 어노테이션도 제공합니다.
- **커스텀 스코프 해석:** `@ComponentScan` 또는 `<context:component-scan>`에 `scopeResolver` 속성으로 커스텀 `ScopeMetadataResolver` 구현 클래스를 지정하여 스코프 결정 방식을 직접 제어할 수도 있습니다.
- **프록시 생성:** 싱글톤이 아닌 스코프(request, session, prototype 등)를 사용할 때는 종종 프록시가 필요합니다. `@ComponentScan` 또는 `<context:component-scan>`의 `scoped-proxy` 속성 (`interfaces` 또는 `targetClass`)을 사용하여 프록시 생성을 지시할 수 있습니다.

---

**아홉 번째 파트: 어노테이션으로 퀄리파이어 메타데이터 제공하기**

이 부분은 XML의 `<qualifier>`나 `<meta>` 태그 대신, **어노테이션을 사용하여** 자동 감지된 컴포넌트에 퀄리파이어 정보를 직접 부여하는 방법을 설명합니다.

- **방법:** 빈으로 스캔될 클래스 정의 **위에** `@Qualifier("식별자")` 또는 직접 만든 커스텀 퀄리파이어 어노테이션(`@Genre("Action")`, `@Offline` 등)을 붙입니다.

    ```java
    @Component
    @Qualifier("Action") // 이 빈은 "Action" 퀄리파이어를 가짐
    public class ActionMovieCatalog implements MovieCatalog { ... }
    
    @Component
    @Genre("Comedy") // 이 빈은 커스텀 @Genre 퀄리파이어("Comedy")를 가짐
    public class ComedyMovieCatalog implements MovieCatalog { ... }
    
    ```

- **동작:** 이렇게 하면 해당 클래스가 빈으로 등록될 때 지정된 퀄리파이어 정보도 함께 등록됩니다. 나중에 `@Autowired @Qualifier("Action")` 이나 `@Autowired @Genre("Comedy")` 처럼 주입을 시도할 때 이 정보가 사용됩니다.
- **XML과의 차이점:** XML에서는 같은 클래스의 빈이라도 `<bean>` 정의마다 다른 퀄리파이어를 줄 수 있었지만(인스턴스별 메타데이터), 어노테이션 방식은 클래스 자체에 퀄리파이어가 붙으므로 보통 해당 클래스로 만들어진 모든 빈 인스턴스가 동일한 퀄리파이어를 갖게 됩니다(클래스별 메타데이터).

**요약:**

클래스패스 스캐닝은 `@Component` 및 관련 스테레오타입 어노테이션을 사용하여 XML 없이 빈을 자동으로 등록하는 강력한 방법입니다. `@ComponentScan` 또는 `<context:component-scan>`으로 스캔을 활성화하고, 필터를 사용하여 대상을 제어할 수 있습니다. 스캔된 컴포넌트 내에서도 `@Bean` 메소드를 정의할 수 있으며, 빈 이름 생성 규칙과 스코프를 커스터마이징할 수 있습니다. 또한, 퀄리파이어 정보를 XML 대신 어노테이션으로 직접 클래스에 부여하여 자동 와이어링 시 활용할 수 있습니다. 이 기능들은 현대 스프링 애플리케이션 개발의 핵심적인 부분입니다.
