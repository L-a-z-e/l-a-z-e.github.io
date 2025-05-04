---
title: Spring Framework Environment Abstraction
description: 
author: laze
date: 2025-05-04 00:00:09 +0900
categories: [Dev, SpringBoot]
tags: [SpringBoot]
---
**Environment Abstraction**

`Environment` 인터페이스는 애플리케이션 환경의 두 가지 핵심 측면인 **프로파일(profiles)**과 **속성(properties)**을 모델링하는, 컨테이너에 통합된 추상화입니다.

**프로파일**은 주어진 프로파일이 활성 상태인 경우에만 컨테이너에 등록될 빈 정의들의 명명된 논리적 그룹입니다.

빈은 XML로 정의되든 어노테이션으로 정의되든 프로파일에 할당될 수 있습니다.

프로파일과 관련된 `Environment` 객체의 역할은 현재 어떤 프로파일(있는 경우)이 활성 상태인지, 그리고 기본적으로 어떤 프로파일(있는 경우)이 활성 상태여야 하는지를 결정하는 것입니다.

**속성**은 거의 모든 애플리케이션에서 중요한 역할을 하며 다양한 소스에서 비롯될 수 있습니다: 프로퍼티 파일, JVM 시스템 속성, 시스템 환경 변수, JNDI, 서블릿 컨텍스트 파라미터, 임시 `Properties` 객체, `Map` 객체 등.

속성과 관련된 `Environment` 객체의 역할은 사용자에게 속성 소스(property sources)를 구성하고 그들로부터 속성을 해석(resolving)하기 위한 편리한 서비스 인터페이스를 제공하는 것입니다.

**빈 정의 프로파일 (Bean Definition Profiles)**

빈 정의 프로파일은 핵심 컨테이너에서 다른 환경에서 다른 빈의 등록을 허용하는 메커니즘을 제공합니다.

"환경"이라는 단어는 사용자마다 다른 의미를 가질 수 있으며, 이 기능은 다음을 포함한 많은 사용 사례에 도움이 될 수 있습니다:

- 개발 환경에서는 인메모리 데이터 소스(in-memory datasource)를 사용하는 반면, QA 또는 프로덕션 환경에서는 동일한 데이터 소스를 JNDI에서 조회하는 것.
- 성능 환경에 애플리케이션을 배포할 때만 모니터링 인프라를 등록하는 것.
- 고객 A 배포 대 고객 B 배포를 위해 빈의 커스터마이징된 구현을 등록하는 것.

`DataSource`가 필요한 실제 애플리케이션에서 첫 번째 사용 사례

```java
// Java
@Bean
public DataSource dataSource() {
	return new EmbeddedDatabaseBuilder()
		.setType(EmbeddedDatabaseType.HSQL)
		.addScript("my-schema.sql")
		.addScript("my-test-data.sql")
		.build();
}
```

```kotlin
// Kotlin
@Bean
fun dataSource(): DataSource {
    return EmbeddedDatabaseBuilder()
        .setType(EmbeddedDatabaseType.HSQL)
        .addScript("my-schema.sql")
        .addScript("my-test-data.sql")
        .build()
}
// Assuming necessary imports and EmbeddedDatabaseBuilder class
```

이제 이 애플리케이션이 QA 또는 프로덕션 환경에 어떻게 배포될 수 있는 지 확인해 봅시다.

애플리케이션의 데이터 소스가 프로덕션 애플리케이션 서버의 JNDI 디렉토리에 등록되어 있다고 가정합니다.

이제 `dataSource` 빈은 다음 목록과 같습니다:

```java
// Java
@Bean(destroyMethod = "") // Prevent Spring from calling close() on DataSource from JNDI
public DataSource dataSource() throws Exception {
	Context ctx = new InitialContext();
	return (DataSource) ctx.lookup("java:comp/env/jdbc/datasource");
}
```

```kotlin
// Kotlin
@Bean(destroyMethod = "") // Prevent Spring from calling close() on DataSource from JNDI
@Throws(Exception::class)
fun dataSource(): DataSource {
    val ctx = InitialContext()
    return ctx.lookup("java:comp/env/jdbc/datasource") as DataSource
}
// Assuming necessary imports
```

문제는 현재 환경에 따라 이 두 변형 사용 간을 어떻게 전환하느냐입니다.

시간이 지남에 따라 스프링 사용자들은 이를 수행하는 여러 가지 방법을 고안했으며, 일반적으로 시스템 환경 변수와 환경 변수 값에 따라 올바른 구성 파일 경로로 해석되는 `${placeholder}` 토큰을 포함하는 XML `<import/>` 문의 조합에 의존합니다.

빈 정의 프로파일은 이 문제에 대한 해결책을 제공하는 핵심 컨테이너 기능입니다.

환경 특정 빈 정의의 앞선 예제에서 보여준 사용 사례를 일반화하면, 특정 컨텍스트에서는 특정 빈 정의를 등록하고 다른 컨텍스트에서는 등록하지 않아야 할 필요성에 도달합니다.

상황 A에서는 특정 프로파일의 빈 정의를 등록하고 상황 B에서는 다른 프로파일을 등록하고 싶다고 말할 수 있습니다. 이 필요성을 반영하도록 구성을 업데이트하는 것으로 시작합니다.

**@Profile 사용하기 (Using @Profile)**

`@Profile` 어노테이션을 사용하면 하나 이상의 지정된 프로파일이 활성 상태일 때 컴포넌트가 등록 대상임을 나타낼 수 있습니다.

앞의 예제를 사용하여 다음과 같이 `dataSource` 구성을 다시 작성할 수 있습니다:

```java
// Java
@Configuration
@Profile("development")
public class StandaloneDataConfig {

	@Bean
	public DataSource dataSource() {
		return new EmbeddedDatabaseBuilder()
			.setType(EmbeddedDatabaseType.HSQL)
			.addScript("classpath:com/bank/config/sql/schema.sql")
			.addScript("classpath:com/bank/config/sql/test-data.sql")
			.build();
	}
}
```

```kotlin
// Kotlin
@Configuration
@Profile("development")
class StandaloneDataConfig {

    @Bean
    fun dataSource(): DataSource {
        return EmbeddedDatabaseBuilder()
            .setType(EmbeddedDatabaseType.HSQL)
            .addScript("classpath:com/bank/config/sql/schema.sql")
            .addScript("classpath:com/bank/config/sql/test-data.sql")
            .build()
    }
}
```

```java
// Java
@Configuration
@Profile("production")
public class JndiDataConfig {

	@Bean(destroyMethod = "") // ①
	public DataSource dataSource() throws Exception {
		Context ctx = new InitialContext();
		return (DataSource) ctx.lookup("java:comp/env/jdbc/datasource");
	}
}
```

① `@Bean(destroyMethod = "")`은 기본 소멸 메소드 추론을 비활성화합니다.

```kotlin
// Kotlin
@Configuration
@Profile("production")
class JndiDataConfig {

    @Bean(destroyMethod = "") // ①
    @Throws(Exception::class)
    fun dataSource(): DataSource {
        val ctx = InitialContext()
        return ctx.lookup("java:comp/env/jdbc/datasource") as DataSource
    }
}
```

*앞서 언급했듯이, `@Bean` 메소드를 사용하면 일반적으로 스프링의 `JndiTemplate`/`JndiLocatorDelegate` 헬퍼 또는 앞서 보여준 직접적인 JNDI `InitialContext` 사용을 통해 프로그래밍 방식 JNDI 룩업을 선택하며,*

`*JndiObjectFactoryBean` 변형은 사용하지 않습니다.*

*이는 반환 타입을 `FactoryBean` 타입으로 선언하도록 강제하여, 여기서 제공된 리소스를 참조하려는 다른 `@Bean` 메소드의 상호 참조 호출에 사용하기 어렵게 만듭니다.*

프로파일 문자열은 단순 프로파일 이름(예: `production`) 또는 프로파일 표현식(profile expression)을 포함할 수 있습니다.

프로파일 표현식은 더 복잡한 프로파일 로직(예: `production & us-east`)을 표현할 수 있도록 합니다. 프로파일 표현식에서는 다음 연산자가 지원됩니다:

- `!`: 프로파일의 논리적 NOT
- `&`: 프로파일들의 논리적 AND
- `|`: 프로파일들의 논리적 OR

*괄호를 사용하지 않고 `&`와 `|` 연산자를 혼합할 수 없습니다.*

*예를 들어, `production & us-east | eu-central`은 유효한 표현식이 아닙니다.*

*이는 `production & (us-east | eu-central)`로 표현되어야 합니다.*

커스텀 조합 어노테이션(custom composed annotation)을 생성할 목적으로 `@Profile`을 메타 어노테이션으로 사용할 수 있습니다.

다음 예제는 `@Profile("production")`의 즉시 대체(drop-in replacement)로 사용할 수 있는 커스텀 `@Production` 어노테이션을 정의합니다:

```java
// Java
@Target(ElementType.TYPE)
@Retention(RetentionPolicy.RUNTIME)
@Profile("production")
public @interface Production {
}
```

```kotlin
// Kotlin
@Target(AnnotationTarget.CLASS) // Apply to classes
@Retention(AnnotationRetention.RUNTIME)
@Profile("production") // Meta-annotation
annotation class Production
```

`@Configuration` 클래스가 `@Profile`로 표시되면, 지정된 프로파일 중 하나 이상이 활성 상태가 아닌 한 해당 클래스와 연관된 모든 `@Bean` 메소드 및 `@Import` 어노테이션은 우회(bypassed)됩니다.

`@Component` 또는 `@Configuration` 클래스가 `@Profile({"p1", "p2"})`로 표시되면, 해당 클래스는 프로파일 'p1' 또는 'p2'가 활성화되지 않은 한 등록되거나 처리되지 않습니다.

주어진 프로파일이 NOT 연산자(`!`)로 접두사가 붙으면, 어노테이션된 요소는 프로파일이 활성 상태가 아닌 경우에만 등록됩니다.

예를 들어, `@Profile({"p1", "!p2"})`가 주어지면, 프로파일 'p1'이 활성 상태이거나 프로파일 'p2'가 활성 상태가 아닌 경우 등록이 발생합니다.

`@Profile`은 또한 다음 예제와 같이 구성 클래스의 특정 빈 하나만 포함하기 위해(예: 특정 빈의 대체 변형) 메소드 레벨에서 선언될 수도 있습니다:

```java
// Java
@Configuration
public class AppConfig {

	@Bean("dataSource")
	@Profile("development") // ①
	public DataSource standaloneDataSource() {
		// ...
	}

	@Bean("dataSource")
	@Profile("production") // ②
	public DataSource jndiDataSource() throws Exception {
		// ...
	}
}
```

① `standaloneDataSource` 메소드는 `development` 프로파일에서만 사용 가능합니다.
② `jndiDataSource` 메소드는 `production` 프로파일에서만 사용 가능합니다.

```kotlin
// Kotlin
@Configuration
class AppConfig {

    @Bean("dataSource")
    @Profile("development") // ①
    fun standaloneDataSource(): DataSource {
        // ...
        return EmbeddedDatabaseBuilder()./* ... */.build() // Example
    }

    @Bean("dataSource")
    @Profile("production") // ②
    @Throws(Exception::class)
    fun jndiDataSource(): DataSource {
        // ...
        val ctx = InitialContext()
        return ctx.lookup("java:comp/env/jdbc/datasource") as DataSource // Example
    }
}
```

`*@Bean` 메소드에 `@Profile`을 사용할 때 특별한 시나리오가 적용될 수 있습니다: 동일한 자바 메소드 이름의 오버로드된 `@Bean` 메소드(생성자 오버로딩과 유사)의 경우, `@Profile` 조건은 모든 오버로드된 메소드에 일관되게 선언되어야 합니다.*

*조건이 일치하지 않으면 오버로드된 메소드 중 첫 번째 선언의 조건만 중요합니다.*

*따라서 `@Profile`은 다른 특정 인수 시그니처를 가진 오버로드된 메소드를 선택하는 데 사용할 수 없습니다.*

*동일한 빈에 대한 모든 팩토리 메소드 간의 해석은 생성 시점에 스프링의 생성자 해석 알고리즘을 따릅니다.*

*다른 프로파일 조건으로 대체 빈을 정의하려면, 앞의 예제에서 보여준 것처럼 `@Bean` `name` 속성을 사용하여 동일한 빈 이름을 가리키는 고유한 자바 메소드 이름을 사용하십시오.*

*인수 시그니처가 모두 동일한 경우(예: 모든 변형이 인수 없는 팩토리 메소드를 가짐), 이는 유효한 자바 클래스에서 이러한 배열을 나타내는 유일한 방법입니다(특정 이름과 인수 시그니처를 가진 메소드는 하나만 있을 수 있으므로).*

**XML 빈 정의 프로파일 (XML Bean Definition Profiles)**

XML 상대물은 `<beans>` 요소의 `profile` 속성입니다. 앞의 샘플 구성은 다음과 같이 두 개의 XML 파일로 다시 작성될 수 있습니다:

```xml
<beans profile="development"
	xmlns="<http://www.springframework.org/schema/beans>"
	xmlns:xsi="<http://www.w3.org/2001/XMLSchema-instance>"
	xmlns:jdbc="<http://www.springframework.org/schema/jdbc>"
	xsi:schemaLocation="...">

	<jdbc:embedded-database id="dataSource">
		<jdbc:script location="classpath:com/bank/config/sql/schema.sql"/>
		<jdbc:script location="classpath:com/bank/config/sql/test-data.sql"/>
	</jdbc:embedded-database>
</beans>
```

```xml
<beans profile="production"
	xmlns="<http://www.springframework.org/schema/beans>"
	xmlns:xsi="<http://www.w3.org/2001/XMLSchema-instance>"
	xmlns:jee="<http://www.springframework.org/schema/jee>"
	xsi:schemaLocation="...">

	<jee:jndi-lookup id="dataSource" jndi-name="java:comp/env/jdbc/datasource"/>
</beans>
```

다음 예제와 같이 분할을 피하고 동일한 파일 내에 `<beans/>` 요소를 중첩하는 것도 가능합니다:

```xml
<beans xmlns="<http://www.springframework.org/schema/beans>"
	xmlns:xsi="<http://www.w3.org/2001/XMLSchema-instance>"
	xmlns:jdbc="<http://www.springframework.org/schema/jdbc>"
	xmlns:jee="<http://www.springframework.org/schema/jee>"
	xsi:schemaLocation="...">

	<!-- 다른 빈 정의들 -->

	<beans profile="development">
		<jdbc:embedded-database id="dataSource">
			<jdbc:script location="classpath:com/bank/config/sql/schema.sql"/>
			<jdbc:script location="classpath:com/bank/config/sql/test-data.sql"/>
		</jdbc:embedded-database>
	</beans>

	<beans profile="production">
		<jee:jndi-lookup id="dataSource" jndi-name="java:comp/env/jdbc/datasource"/>
	</beans>
</beans>
```

`*spring-bean.xsd`는 이러한 요소를 파일의 마지막 요소로만 허용하도록 제한되었습니다.*

*이는 XML 파일에 혼란(clutter)을 야기하지 않으면서 유연성을 제공하는 데 도움이 될 것입니다.XML*

*상대물은 앞서 설명한 프로파일 표현식을 지원하지 않습니다.*

*그러나 `!` 연산자를 사용하여 프로파일을 부정하는 것은 가능합니다. 또한 다음 예제와 같이 프로파일을 중첩하여 논리적 "and"를 적용하는 것도 가능합니다:*

```xml
<beans xmlns="<http://www.springframework.org/schema/beans>"
	xmlns:xsi="<http://www.w3.org/2001/XMLSchema-instance>"
	xmlns:jdbc="<http://www.springframework.org/schema/jdbc>"
	xmlns:jee="<http://www.springframework.org/schema/jee>"
	xsi:schemaLocation="...">

	<!-- 다른 빈 정의들 -->

	<beans profile="production">
		<beans profile="us-east"> <!-- Nested profile -->
			<jee:jndi-lookup id="dataSource" jndi-name="java:comp/env/jdbc/datasource"/>
		</beans>
	</beans>
</beans>
```

앞의 예제에서 `dataSource` 빈은 `production` 및 `us-east` 프로파일이 모두 활성 상태인 경우 노출됩니다.

**프로파일 활성화하기 (Activating a Profile)**

이제 구성을 업데이트했으므로, 어떤 프로파일이 활성 상태인지 스프링에 지시해야 합니다.

샘플 애플리케이션을 지금 바로 시작하면 `dataSource`라는 이름의 스프링 빈을 찾을 수 없기 때문에 `NoSuchBeanDefinitionException`이 발생하는 것을 볼 수 있습니다.

프로파일 활성화는 여러 가지 방법으로 수행할 수 있지만, 가장 간단한 방법은 `ApplicationContext`를 통해 사용 가능한 `Environment` API에 대해 프로그래밍 방식으로 수행하는 것입니다.

```java
// Java
AnnotationConfigApplicationContext ctx = new AnnotationConfigApplicationContext();
ctx.getEnvironment().setActiveProfiles("development"); // Activate "development" profile
ctx.register(SomeConfig.class, StandaloneDataConfig.class, JndiDataConfig.class);
ctx.refresh();
```

```kotlin
// Kotlin
val ctx = AnnotationConfigApplicationContext()
ctx.environment.setActiveProfiles("development") // Activate "development" profile
ctx.register(SomeConfig::class.java, StandaloneDataConfig::class.java, JndiDataConfig::class.java)
ctx.refresh()
// Assuming SomeConfig, StandaloneDataConfig, JndiDataConfig exist
```

또한, 시스템 환경 변수, JVM 시스템 속성, `web.xml`의 서블릿 컨텍스트 파라미터 또는 JNDI의 항목(속성 소스 추상화(PropertySource Abstraction) 참조)을 통해 지정될 수 있는

`spring.profiles.active` 속성을 통해 선언적으로 프로파일을 활성화할 수도 있습니다.

통합 테스트에서는 `spring-test` 모듈의 `@ActiveProfiles` 어노테이션을 사용하여 활성 프로파일을 선언할 수 있습니다.

프로파일은 "양자택일(either-or)" 제안이 아니라는 점에 유의하십시오.

한 번에 여러 프로파일을 활성화할 수 있습니다.

프로그래밍 방식으로, `String… varargs`를 받는 `setActiveProfiles()` 메소드에 여러 프로파일 이름을 제공할 수 있습니다. 다음 예제는 여러 프로파일을 활성화합니다:

```java
// Java
ctx.getEnvironment().setActiveProfiles("profile1", "profile2");
```

```kotlin
// Kotlin
ctx.environment.setActiveProfiles("profile1", "profile2")
```

선언적으로, `spring.profiles.active`는 다음 예제와 같이 쉼표로 구분된 프로파일 이름 목록을 받을 수 있습니다:

- `Dspring.profiles.active="profile1,profile2"`

**기본 프로파일 (Default Profile)**

기본 프로파일은 활성 프로파일이 없는 경우 활성화되는 프로파일을 나타냅니다.

```java
// Java
@Configuration
@Profile("default")
public class DefaultDataConfig {

	@Bean
	public DataSource dataSource() {
		// ... define default DataSource
	}
}
```

```kotlin
// Kotlin
@Configuration
@Profile("default")
class DefaultDataConfig {

    @Bean
    fun dataSource(): DataSource {
        // ... define default DataSource
        return EmbeddedDatabaseBuilder()./*...*/.build() // Example
    }
}
```

활성 프로파일이 없으면 `dataSource`가 생성됩니다.

이를 하나 이상의 빈에 대한 기본 정의를 제공하는 방법으로 볼 수 있습니다.

어떤 프로파일이라도 활성화되면 기본 프로파일은 적용되지 않습니다.

기본 프로파일의 이름은 `default`입니다. `Environment`에서 `setDefaultProfiles()`를 사용하거나 선언적으로 `spring.profiles.default` 속성을 사용하여 기본 프로파일의 이름을 변경할 수 있습니다.

**속성 소스 추상화 (PropertySource Abstraction)**

스프링의 `Environment` 추상화는 구성 가능한 속성 소스 계층 구조에 대한 검색 작업을 제공합니다.

```java
// Java
ApplicationContext ctx = new GenericApplicationContext();
Environment env = ctx.getEnvironment();
boolean containsMyProperty = env.containsProperty("my-property");
System.out.println("Does my environment contain the 'my-property' property? " + containsMyProperty);
```

```kotlin
// Kotlin
val ctx = GenericApplicationContext()
val env = ctx.environment
val containsMyProperty = env.containsProperty("my-property")
println("Does my environment contain the 'my-property' property? $containsMyProperty")
```

앞의 스니펫에서, 현재 환경에 `my-property` 속성이 정의되어 있는지 스프링에 묻는 높은 수준의 방법을 봅니다.

이 질문에 답하기 위해 `Environment` 객체는 `PropertySource` 객체 집합에 대한 검색을 수행합니다.

`PropertySource`는 모든 키-값 쌍 소스에 대한 간단한 추상화이며,

스프링의 `StandardEnvironment`는 두 개의 `PropertySource` 객체로 구성됩니다 - 하나는 JVM 시스템 속성 집합(`System.getProperties()`)을 나타내고 다른 하나는 시스템 환경 변수 집합(`System.getenv()`)을 나타냅니다.

*이러한 기본 속성 소스는 독립 실행형 애플리케이션에서 사용하기 위해 `StandardEnvironment`에 존재합니다.*

`*StandardServletEnvironment`는 서블릿 구성, 서블릿 컨텍스트 파라미터 및 JNDI가 사용 가능한 경우 `JndiPropertySource`를 포함한 추가 기본 속성 소스로 채워집니다.*

구체적으로, `StandardEnvironment`를 사용할 때 `env.containsProperty("my-property")` 호출은 런타임 시 `my-property` 시스템 속성 또는 `my-property` 환경 변수가 존재하는 경우 `true`를 반환합니다.

수행되는 검색은 계층적입니다.

기본적으로 시스템 속성은 환경 변수보다 우선합니다.

따라서 `env.getProperty("my-property")` 호출 중에 `my-property` 속성이 두 곳 모두에 설정되어 있는 경우 시스템 속성 값이 "우선"하여 반환됩니다.

속성 값은 병합되지 않고 선행 항목에 의해 완전히 오버라이드된다는 점에 유의하십시오.

일반적인 `StandardServletEnvironment`의 경우 전체 계층 구조는 다음과 같으며 가장 높은 우선순위 항목이 맨 위에 있습니다:

1. `ServletConfig` 파라미터 (해당되는 경우 - 예: `DispatcherServlet` 컨텍스트의 경우)
2. `ServletContext` 파라미터 (`web.xml` `context-param` 항목)
3. JNDI 환경 변수 (`java:comp/env/` 항목)
4. JVM 시스템 속성 (`D` 명령줄 인수)
5. JVM 시스템 환경 (운영 체제 환경 변수)

가장 중요한 것은 전체 메커니즘을 구성할 수 있다는 것입니다.

이 검색에 통합하려는 속성의 커스텀 소스가 있을 수 있습니다.

그렇게 하려면 자신만의 `PropertySource`를 구현하고 인스턴스화한 다음 현재 `Environment`의 `PropertySources` 집합에 추가합니다.

```java
// Java
ConfigurableApplicationContext ctx = new GenericApplicationContext();
MutablePropertySources sources = ctx.getEnvironment().getPropertySources();
sources.addFirst(new MyPropertySource()); // Add with highest precedence
```

```kotlin
// Kotlin
val ctx = GenericApplicationContext()
val sources = ctx.environment.propertySources
sources.addFirst(MyPropertySource()) // Add with highest precedence
// Assuming MyPropertySource exists and implements PropertySource
```

앞의 코드에서 `MyPropertySource`는 검색에서 가장 높은 우선순위로 추가되었습니다.

`my-property` 속성을 포함하는 경우 해당 속성은 다른 `PropertySource`의 모든 `my-property` 속성보다 우선하여 감지되고 반환됩니다.

`MutablePropertySources` API는 속성 소스 집합의 정밀한 조작을 허용하는 여러 메소드를 노출합니다.

**@PropertySource 사용하기 (Using @PropertySource)**

`@PropertySource` 어노테이션은 스프링의 `Environment`에 `PropertySource`를 추가하는 편리하고 선언적인 메커니즘을 제공합니다.

`testbean.name=myTestBean` 키-값 쌍을 포함하는 `app.properties`라는 파일이 주어졌을 때, 다음 `@Configuration` 클래스는 `testBean.getName()` 호출이 `myTestBean`을 반환하도록 `@PropertySource`를 사용합니다:

```java
// Java
@Configuration
@PropertySource("classpath:/com/myco/app.properties")
public class AppConfig {

 @Autowired
 Environment env;

 @Bean
 public TestBean testBean() {
  TestBean testBean = new TestBean();
  testBean.setName(env.getProperty("testbean.name"));
  return testBean;
 }
}
```

```kotlin
// Kotlin
@Configuration
@PropertySource("classpath:/com/myco/app.properties")
class AppConfig {

    @Autowired
    lateinit var env: Environment

    @Bean
    fun testBean(): TestBean {
        val testBean = TestBean()
        testBean.name = env.getProperty("testbean.name") // Assuming TestBean has a name property
        return testBean
    }
}
// Assuming TestBean exists
```

다음 예제와 같이 `@PropertySource` 리소스 위치에 있는 모든 `${...}` 플레이스홀더는 환경에 이미 등록된 속성 소스 집합에 대해 해석됩니다:

```java
// Java
@Configuration
@PropertySource("classpath:/com/${my.placeholder:default/path}/app.properties")
public class AppConfig {

 @Autowired
 Environment env;

 // ... bean definition using env.getProperty("...")
}
```

```kotlin
// Kotlin
@Configuration
// Use \\ to escape $ if needed, or use string templates carefully
@PropertySource("classpath:/com/\\${my.placeholder:default/path}/app.properties")
class AppConfig {

    @Autowired
    lateinit var env: Environment

    // ... bean definition using env.getProperty("...")
}
```

`my.placeholder`가 이미 등록된 속성 소스 중 하나(예: 시스템 속성 또는 환경 변수)에 존재한다고 가정하면, 플레이스홀더는 해당 값으로 해석됩니다.

그렇지 않으면 `default/path`가 기본값으로 사용됩니다. 기본값이 지정되지 않고 속성을 해석할 수 없는 경우 `IllegalArgumentException`이 발생합니다.

`*@PropertySource`는 반복 가능한(repeatable) 어노테이션으로 사용될 수 있습니다.*

`*@PropertySource`는 또한 속성 오버라이드가 있는 커스텀 조합 어노테이션을 생성하기 위해 메타 어노테이션으로 사용될 수도 있습니다.*

**문장에서의 플레이스홀더 해석 (Placeholder Resolution in Statements)**

역사적으로 요소의 플레이스홀더 값은 JVM 시스템 속성 또는 환경 변수에 대해서만 해석될 수 있었습니다.

이것은 더 이상 그렇지 않습니다.

`Environment` 추상화가 컨테이너 전체에 통합되어 있으므로 플레이스홀더 해석을 그것을 통해 라우팅하기 쉽습니다.

이는 즉, 원하는 방식으로 해석 프로세스를 구성할 수 있다는 것을 의미합니다. 시스템 속성 및 환경 변수를 통한 검색 우선순위를 변경하거나 완전히 제거할 수 있습니다.

또한 적절하게 자신만의 속성 소스를 혼합에 추가할 수도 있습니다.

구체적으로, 다음 문장은 `customer` 속성이 `Environment`에서 사용 가능한 한 어디에 정의되었는지에 관계없이 작동합니다:

```xml
<beans>
	<import resource="com/bank/service/${customer}-config.xml"/>
</beans>
```

---

**전체 주제: 환경 추상화 (Environment Abstraction)**

이 부분은 스프링 애플리케이션이 실행되는 **"환경"** 에 따라 설정을 다르게 적용하거나, 외부 설정 값들을 체계적으로 관리할 수 있도록 스프링이 제공하는 **`Environment` 인터페이스**와 그 핵심 기능(프로파일, 속성)에 대한 설명입니다.

**핵심 아이디어:** 애플리케이션의 실행 환경(개발, 테스트, 운영 등)을 인식하고, 그 환경에 맞는 설정을 적용하며, 외부 설정 값들을 일관된 방식으로 관리하자!

---

**첫 번째 파트: `Environment` 인터페이스 소개 (프로파일과 속성)**

- **`Environment` 인터페이스:** 스프링 컨테이너에 통합되어 있는 중요한 인터페이스입니다. 애플리케이션이 실행되는 환경에 대한 정보를 담고 있으며, 두 가지 핵심적인 측면을 다룹니다.
  1. **프로파일 (Profiles):** 특정 조건(활성화된 프로파일)에 따라 빈(Bean) 정의를 선택적으로 등록하거나 제외하는 메커니즘입니다. 즉, **환경별로 다른 빈 구성을 가능하게 합니다.**
  2. **속성 (Properties):** 애플리케이션 설정 값들(예: DB 접속 정보, API 키, 서버 포트 등)을 관리하고 접근하는 방법을 제공합니다. 이 값들은 다양한 소스(파일, 시스템 변수 등)로부터 올 수 있습니다. 즉, **설정 값 관리를 위한 통합 인터페이스**입니다.
- **역할 요약:** `Environment` 객체는 "현재 어떤 프로파일이 활성화되어 있는가?" 그리고 "특정 속성(property)의 값은 무엇인가?" 라는 질문에 답해주는 역할을 합니다.

---

**두 번째 파트: 빈 정의 프로파일 (Bean Definition Profiles)**

이 부분은 **"프로파일(Profiles)"** 기능을 사용하여 특정 환경에서만 특정 빈들을 활성화하는 방법에 대해 자세히 설명합니다.

- **왜 필요한가? (사용 사례)**
  - 개발 중에는 빠르고 간편한 **인메모리 DB**를 사용하고, 실제 운영 환경에서는 **운영 DB (JNDI 룩업)** 를 사용하고 싶을 때.
  - 성능 테스트 환경에서만 **모니터링 관련 빈**들을 추가로 등록하고 싶을 때.
  - 특정 고객사(A 고객, B 고객) 배포 환경에 따라 **다른 기능 구현 빈**을 등록하고 싶을 때.
  - 즉, 동일한 코드베이스를 다른 환경에 배포할 때, **환경에 맞는 빈 구성**을 동적으로 적용하기 위해 사용합니다.
- **기존 방식의 문제점:** 과거에는 환경 변수와 XML의 `<import>` 및 플레이스홀더(`${...}`)를 조합하여 복잡하게 처리해야 했습니다.
- **프로파일의 해결책:** 스프링의 프로파일 기능은 이 문제를 표준화되고 깔끔하게 해결하는 방법을 제공합니다.
- **핵심:** 특정 빈 정의(설정)에 **프로파일 이름을 지정**해두고, 애플리케이션 실행 시 **활성화할 프로파일을 지정**하면, 해당 프로파일에 속한 빈들만 컨테이너에 등록됩니다.

---

**세 번째 파트: `@Profile` 어노테이션 사용하기**

Java Config 방식에서 프로파일을 어떻게 사용하는지 설명합니다.

- **`@Profile` 어노테이션:** 클래스 또는 `@Bean` 메소드 레벨에 붙여서 해당 설정이나 빈이 **어떤 프로파일에서 활성화될지** 지정합니다.
- **클래스 레벨 적용:** `@Configuration` 클래스 위에 `@Profile("프로파일이름")`을 붙이면, 해당 프로파일이 활성화되지 않으면 그 클래스 안의 **모든 `@Bean` 메소드와 `@Import` 설정이 무시**됩니다.

    ```java
    @Configuration
    @Profile("development") // "development" 프로파일이 활성화될 때만 이 설정 사용
    public class StandaloneDataConfig {
        @Bean
        public DataSource dataSource() { /* 개발용 인메모리 DB 설정 */ }
    }
    
    @Configuration
    @Profile("production") // "production" 프로파일이 활성화될 때만 이 설정 사용
    public class JndiDataConfig {
        @Bean(destroyMethod="")
        public DataSource dataSource() throws Exception { /* 운영용 JNDI DB 설정 */ }
    }
    ```

- **메소드 레벨 적용:** `@Bean` 메소드 위에 `@Profile("프로파일이름")`을 붙이면, 해당 프로파일이 활성화될 때만 그 `@Bean` 메소드가 실행되어 빈이 등록됩니다. 이를 이용해 **같은 이름의 빈을 프로파일별로 다르게 정의**할 수 있습니다. (단, 이때 Java 메소드 이름은 달라야 함)

    ```java
    @Configuration
    public class AppConfig {
        @Bean("dataSource") // 빈 이름은 동일하게 "dataSource"
        @Profile("development")
        public DataSource standaloneDataSource() { /* 개발용 DB */ }
    
        @Bean("dataSource") // 빈 이름은 동일하게 "dataSource"
        @Profile("production")
        public DataSource jndiDataSource() throws Exception { /* 운영용 DB */ }
    }
    ```

- **프로파일 표현식:** `@Profile` 안에는 단순 이름 외에 논리 연산자를 사용한 표현식도 가능합니다.
  - `!`: NOT (예: `@Profile("!production")` -> production이 아닐 때 활성)
  - `&`: AND (예: `@Profile("production & us-east")` -> production이면서 us-east일 때 활성)
  - `|`: OR (예: `@Profile("development | test")` -> development 또는 test일 때 활성)
  - **주의:** `&`와 `|`를 괄호 없이 혼용할 수 없습니다. `production & (us-east | eu-central)` 처럼 괄호 필요.
- **커스텀 프로파일 어노테이션:** `@Profile`을 메타 어노테이션으로 사용하여 `@Production` 같은 커스텀 어노테이션을 만들어 사용할 수도 있습니다 (가독성 향상).

---

**네 번째 파트: XML 빈 정의 프로파일**

XML 설정 방식에서 프로파일을 어떻게 사용하는지 설명합니다.

- **`<beans profile="프로파일이름">` 속성:** 최상위 `<beans>` 요소에 `profile` 속성을 사용하여 해당 XML 파일 전체 또는 특정 `<beans>` 블록이 어떤 프로파일에 속하는지 지정합니다.
  - 별도의 XML 파일 사용: `dev-beans.xml`에는 `<beans profile="development">`, `prod-beans.xml`에는 `<beans profile="production">` 처럼 사용.
  - 하나의 XML 파일 내 중첩: `<beans>` 요소 안에 다른 `<beans profile="...">` 요소를 중첩하여 사용할 수 있습니다.

      ```xml
      <beans ...>
          <!-- 공통 빈 정의 -->
          <beans profile="development">
              <!-- 개발용 빈 정의 -->
          </beans>
          <beans profile="production">
              <!-- 운영용 빈 정의 -->
          </beans>
      </beans>
      ```

- **XML 제약:**
  - 프로파일 표현식(`&`, `|`)은 지원하지 않습니다.
  - `!` (NOT) 연산자는 사용 가능합니다 (예: `profile="!production"`).
  - `<beans>` 요소를 중첩하면 **AND** 조건으로 동작합니다. (예: `<beans profile="production"><beans profile="us-east">...</beans></beans>` 는 production AND us-east)

---

**다섯 번째 파트: 프로파일 활성화하기**

정의된 프로파일 중 실제로 어떤 것을 사용할지 스프링에게 알려주는 방법입니다.

- **프로그래밍 방식:** `ApplicationContext` 객체의 `Environment`를 통해 `setActiveProfiles("프로파일이름1", "프로파일이름2", ...)` 메소드를 호출합니다. 컨텍스트를 `refresh()` 하기 전에 호출해야 합니다.

    ```java
    AnnotationConfigApplicationContext ctx = new AnnotationConfigApplicationContext();
    ctx.getEnvironment().setActiveProfiles("development"); // "development" 프로파일 활성화
    ctx.register(...);
    ctx.refresh();
    ```

- **선언적 방식 (`spring.profiles.active` 속성):** 가장 일반적인 방법입니다. 다음 위치 중 하나에 `spring.profiles.active` 속성을 설정합니다.
  - **JVM 시스템 속성:** `Dspring.profiles.active="production,us-east"`
  - **시스템 환경 변수:** `SPRING_PROFILES_ACTIVE=production,us-east`
  - **`web.xml` 서블릿 컨텍스트 파라미터:** `<context-param>`으로 설정.
  - **JNDI 항목:** JNDI에 등록.
  - **프로퍼티 파일:** `application.properties` 등에 `spring.profiles.active=...` 설정 (스프링 부트에서 주로 사용)
  - 쉼표(`,`)로 구분하여 여러 프로파일을 동시에 활성화할 수 있습니다.
- **통합 테스트:** `@ActiveProfiles` 어노테이션 (in `spring-test`)

---

**여섯 번째 파트: 기본 프로파일 (Default Profile)**

- **개념:** **활성화된 프로파일이 아무것도 없을 때** 자동으로 활성화되는 프로파일입니다.
- **기본 이름:** `default` (문자열)
- **사용법:** 특정 빈 정의에 `@Profile("default")` (또는 XML에서 `profile="default"`)를 지정하면, 다른 프로파일이 명시적으로 활성화되지 않았을 때 해당 빈이 등록됩니다.
- **이름 변경:** `Environment`의 `setDefaultProfiles()` 메소드나 `spring.profiles.default` 속성을 사용하여 기본 프로파일의 이름을 변경할 수 있습니다.
- **주의:** 어떤 프로파일이라도 하나 이상 활성화되면, 기본 프로파일은 적용되지 않습니다.

---

**일곱 번째 파트: 속성 소스 추상화 (PropertySource Abstraction)**

이제 `Environment`의 두 번째 핵심 기능인 **"속성(Properties)"** 관리 방식을 설명합니다.

- **`PropertySource`:** 스프링은 설정 값(키-값 쌍)을 제공하는 모든 소스(예: `.properties` 파일, 시스템 환경 변수, JVM 시스템 속성 등)를 **`PropertySource`** 라는 객체로 추상화합니다.
- **`Environment`의 역할:** `Environment` 객체는 이러한 `PropertySource` 객체들의 **계층 구조(목록)** 를 관리합니다.
- **속성 값 검색:** `environment.getProperty("my-property")` 처럼 특정 속성 값을 요청하면, `Environment`는 자신이 관리하는 `PropertySource` 목록을 **정해진 순서(우선순위)** 대로 검색하여 **가장 먼저 찾은 값을 반환**합니다.
- **기본 속성 소스 및 우선순위:**
  - **`StandardEnvironment` (일반 자바 앱):**
    1. JVM 시스템 속성 (`System.getProperties()`) - 가장 높음
    2. 시스템 환경 변수 (`System.getenv()`)
  - **`StandardServletEnvironment` (웹 앱):**
    1. 서블릿 설정 파라미터 (`ServletConfig` init-params) - 가장 높음
    2. 서블릿 컨텍스트 파라미터 (`ServletContext` context-params)
    3. JNDI 환경 변수 (`java:comp/env/`)
    4. JVM 시스템 속성
    5. 시스템 환경 변수
- **커스텀 `PropertySource` 추가:** 개발자는 자신만의 `PropertySource` 구현체를 만들고, `environment.getPropertySources().addFirst(...)` (가장 높은 우선순위로 추가) 또는 `addLast(...)` 등을 사용하여 기존 계층 구조에 **추가**할 수 있습니다. 이를 통해 원하는 소스에서 속성 값을 가져오도록 확장할 수 있습니다.

---

**여덟 번째 파트: `@PropertySource` 어노테이션 사용하기**

Java Config에서 프로퍼티 파일 같은 외부 속성 소스를 `Environment`에 **쉽게 추가**하는 방법을 설명합니다.

- **`@PropertySource` 어노테이션:** `@Configuration` 클래스 위에 붙여서, 지정된 리소스(주로 `.properties` 파일)를 읽어 `PropertySource`로 만들어 `Environment`에 등록하도록 지시합니다.
- **사용법:**

    ```java
    @Configuration
    @PropertySource("classpath:/com/myco/app.properties") // 클래스패스의 app.properties 파일을 로드
    public class AppConfig {
        @Autowired
        Environment env; // Environment 객체 주입받기
    
        @Bean
        public TestBean testBean() {
            TestBean testBean = new TestBean();
            // Environment를 통해 프로퍼티 값 사용
            testBean.setName(env.getProperty("testbean.name"));
            return testBean;
        }
    }
    ```

- **위치 플레이스홀더:** `@PropertySource`의 파일 경로 자체에도 플레이스홀더를 사용할 수 있습니다. 이 플레이스홀더는 **이미 등록된 다른 속성 소스** (예: 시스템 속성)의 값으로 해석됩니다. 기본값을 지정하거나 해석 실패 시 예외 처리도 가능합니다.

    ```java
    // ${my.placeholder} 값을 찾아서 경로 완성, 없으면 default/path 사용
    @PropertySource("classpath:/com/${my.placeholder:default/path}/app.properties")
    ```

- **반복 사용:** `@PropertySource`는 여러 번 사용하여 여러 파일을 등록할 수 있습니다. 커스텀 조합 어노테이션으로 만들 수도 있습니다.

---

**아홉 번째 파트: 문장에서의 플레이스홀더 해석**

- 과거에는 XML 설정 등의 `${...}` 플레이스홀더가 주로 JVM 시스템 속성이나 환경 변수만 참조할 수 있었습니다.
- 하지만 이제 `Environment` 추상화 덕분에, XML의 `<import resource="...">`나 다른 설정 값에 사용된 `${...}` 플레이스홀더도 **`Environment`가 관리하는 모든 속성 소스(PropertySource)를 대상으로 해석**됩니다. 즉, `.properties` 파일에 정의된 값도 플레이스홀더로 사용할 수 있게 되어 훨씬 유연해졌습니다.

**요약:**

`Environment`는 스프링에서 **프로파일(환경별 빈 구성)** 과 **속성(설정 값 관리)** 이라는 두 가지 중요한 개념을 다루는 핵심 추상화입니다. `@Profile` (Java Config) 또는 `<beans profile="...">` (XML)을 사용하여 환경별로 다른 빈 설정을 활성화할 수 있으며, `spring.profiles.active` 속성으로 활성화할 프로파일을 지정합니다. 속성 값은 다양한 `PropertySource`로부터 계층적으로 관리되며, `@PropertySource` 어노테이션으로 프로퍼티 파일을 쉽게 추가하고 `Environment` 객체나 `${...}` 플레이스홀더를 통해 값을 사용할 수 있습니다.

---

**1. JNDI (Java Naming and Directory Interface)**

- **개념:** 자바 애플리케이션이 **이름(Naming)** 을 사용하여 **객체(Object)** 나 **데이터(Data)** 를 **찾아오는(Lookup)** 방법을 표준화한 **API (명세)** 입니다. 마치 전화번호부 같은 역할을 합니다. 전화번호부에 이름(예: "홍길동")을 찾으면 전화번호(데이터)를 알려주듯이, JNDI는 특정 이름(예: "jdbc/myDataSource")을 찾으면 해당 객체(예: 데이터베이스 커넥션 풀 객체)를 돌려줍니다.
- **주요 사용처:** 주로 **애플리케이션 서버(WAS - Web Application Server, 예: Tomcat, JBoss, WebSphere)** 환경에서 사용됩니다. WAS 관리자가 미리 서버 설정에 데이터 소스(DB 연결 정보), 메시징 큐, 외부 서비스 연결 정보 등을 **특정 JNDI 이름으로 등록**해 놓으면, 애플리케이션 코드에서는 복잡한 설정 없이 그 **이름만 가지고 해당 자원을 찾아서 사용**할 수 있습니다.
  - **예시:** 코드에서 `Context ctx = new InitialContext(); DataSource ds = (DataSource) ctx.lookup("java:comp/env/jdbc/datasource");` 와 같이 사용합니다. 여기서 `"java:comp/env/jdbc/datasource"`가 JNDI 이름입니다.
- **스프링과의 관계:** 스프링은 JNDI를 통해 등록된 자원(특히 `DataSource`)을 쉽게 찾아 빈으로 등록하는 기능을 제공합니다. (`<jee:jndi-lookup>` XML 태그, 또는 `JndiObjectFactoryBean`, `JndiTemplate` 등)
- **핵심:** "자바 환경에서 **이름으로 자원을 찾아 쓰는 표준 방법**" 정도로 이해하시면 됩니다. 주로 WAS 환경에서 설정된 자원을 가져올 때 사용됩니다.

**2. CGLIB (Code Generation Library)**

- **개념:** 바이트코드(자바 클래스 파일의 내용)를 **조작**하여 **런타임(애플리케이션 실행 중)** 에 **새로운 클래스를 동적으로 생성**해주는 강력한 **라이브러리**입니다.
- **주요 기능:** 어떤 클래스가 주어지면, CGLIB은 그 클래스를 **상속(extends)하는 자식 클래스**를 즉석에서 만들어낼 수 있습니다. 이 자식 클래스는 부모 클래스의 메소드를 오버라이드하여 **추가적인 기능(부가 기능)** 을 넣거나 동작을 변경할 수 있습니다.
- **스프링에서의 사용:** 스프링은 다음과 같은 경우에 내부적으로 CGLIB을 **많이** 사용합니다.
  - **AOP 프록시 생성:** `@Transactional`, `@Async` 등 AOP 기능이 적용된 빈을 만들 때, 원본 클래스가 인터페이스를 구현하지 않았다면 CGLIB을 사용하여 프록시(자식 클래스)를 만듭니다.
  - **`@Configuration` 클래스 처리:** `@Configuration(proxyBeanMethods=true)`일 때, `@Bean` 메소드 간 호출 시 싱글톤을 보장하기 위해 CGLIB 프록시(자식 클래스)를 만듭니다.
  - **스코프 프록시 생성:** `@Scope(proxyMode = ScopedProxyMode.TARGET_CLASS)` 설정 시 CGLIB 프록시(자식 클래스)를 만듭니다.
- **핵심:** "스프링이 **실행 중에 클래스를 상속하는 가짜 자식 클래스(프록시)** 를 몰래 만들 때 사용하는 기술 라이브러리" 정도로 이해하시면 됩니다. (그래서 원본 클래스가 `final`이면 안 되는 제약이 생김)

**3. JSR (Java Specification Request)**

- **개념:** **자바 기술에 대한 표준 명세(Specification)를 만들기 위한 공식적인 프로세스 및 그 결과 문서**를 의미합니다. JCP(Java Community Process)라는 과정을 통해 전문가 그룹이 모여 특정 기술(예: 의존성 주입, 어노테이션)에 대한 표준 API와 동작 방식을 정의하고, 그 결과로 JSR 문서와 번호가 부여됩니다.
- **예시:**
  - **JSR-330 (Dependency Injection for Java):** `@Inject`, `@Named`, `@Qualifier`, `@Scope`, `@Singleton`, `Provider` 등 **표준 의존성 주입 관련 어노테이션**들을 정의했습니다. 스프링은 이 표준 어노테이션들을 지원합니다.
  - **JSR-250 (Common Annotations for the Java Platform):** `@PostConstruct`, `@PreDestroy`, `@Resource`, `@ManagedBean` 등 **자바 플랫폼 전반에서 공통적으로 사용될 수 있는 어노테이션**들을 정의했습니다. 스프링 역시 이 표준 어노테이션들을 지원합니다.
- **숫자를 외워야 할까? -> 아니요!**
  - **JSR 뒤에 붙는 숫자(330, 250 등)는 해당 명세의 고유 번호일 뿐, 개발자가 이 숫자를 외울 필요는 전혀 없습니다.**
  - **중요한 것은 그 JSR이 어떤 기술에 대한 표준이고, 어떤 주요 어노테이션이나 인터페이스를 정의했는지 아는 것입니다.** 예를 들어, "아, `@Inject`는 JSR-330 표준 DI 어노테이션이지" 또는 "`@PostConstruct`는 JSR-250 표준 생명주기 어노테이션이야" 정도로 이해하면 충분합니다.
  - 문서나 자료에서 JSR 번호가 언급되는 것은 "이것은 특정 회사의 기술이 아니라 자바 표준 기술이다"라는 것을 명확히 하기 위함인 경우가 많습니다.
- **핵심:** "자바 기술의 **표준 규격(Specification)** 이며, 특정 번호로 식별된다. **표준 어노테이션** 등이 여기서 정의되었다." 정도로 이해하시면 됩니다.
