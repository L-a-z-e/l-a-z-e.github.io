---
title: Spring Framework Customizing the Nature of a Bean
description: 
author: laze
date: 2025-05-02 00:00:02 +0900
categories: [Dev, SpringBoot]
tags: [SpringBoot]
---
**Customizing the Nature of a Bean**

스프링 프레임워크는 빈의 특성을 커스터마이징하는 데 사용할 수 있는 여러 인터페이스를 제공합니다.

- Lifecycle Callbacks
- `ApplicationContextAware` 및 `BeanNameAware`
- 기타 `Aware` 인터페이스

**Lifecycle Callbacks**

컨테이너의 빈 생명주기 관리와 상호 작용하기 위해, 스프링 `InitializingBean` 및 `DisposableBean` 인터페이스를 구현할 수 있습니다.

컨테이너는 전자에 대해 `afterPropertiesSet()`, 후자에 대해 `destroy()`를 호출하여 빈이 초기화 및 소멸 시 특정 작업을 수행하도록 합니다.

생명주기 콜백을 받는 가장 좋은 방법은 일반적으로 `@PostConstruct` 및 `@PreDestroy` 어노테이션을 사용하는 것으로 간주됩니다.

이러한 어노테이션을 사용하면 빈이 스프링 특정 인터페이스에 결합되지 않는다는 의미입니다.

어노테이션을 사용하고 싶지 않지만 여전히 결합을 제거하고 싶다면, `init-method` 및 `destroy-method` 빈 정의 메타데이터를 사용할 수 있습니다.

내부적으로 스프링 프레임워크는 찾을 수 있는 모든 콜백 인터페이스를 처리하고 적절한 메소드를 호출하기 위해 `BeanPostProcessor` 구현을 사용합니다.

스프링이 기본적으로 제공하지 않는 커스텀 기능이나 다른 생명주기 동작이 필요한 경우, 직접 `BeanPostProcessor`를 구현할 수 있습니다.

초기화 및 소멸 콜백 외에도, 스프링 관리 객체는 `Lifecycle` 인터페이스를 구현하여 컨테이너 자체의 생명주기에 의해 구동되는 시작 및 종료 프로세스에 참여할 수도 있습니다.

*초기화 콜백 (Initialization Callbacks)*`org.springframework.beans.factory.InitializingBean` 인터페이스는 컨테이너가 빈에 필요한 모든 속성을 설정한 후 빈이 초기화 작업을 수행할 수 있도록 합니다.

`InitializingBean` 인터페이스는 단일 메소드를 지정합니다:

```java
void afterPropertiesSet() throws Exception;
```

`InitializingBean` 인터페이스는 코드를 불필요하게 스프링에 결합시키므로 사용하지 않는 것이 좋습니다.

대신 `@PostConstruct` 어노테이션을 사용하거나 POJO 초기화 메소드를 지정하는 것을 제안합니다.

XML 기반 설정 메타데이터의 경우, `init-method` 속성을 사용하여 void 반환 타입과 인수 없는 시그니처를 가진 메소드의 이름을 지정할 수 있습니다.

자바 구성에서는 `@Bean`의 `initMethod` 속성을 사용할 수 있습니다. 생명주기 콜백 받기(Receiving Lifecycle Callbacks)를 참조하십시오.

```xml
<bean id="exampleInitBean" class="examples.ExampleBean" init-method="init"/>
```

```java
// Java
public class ExampleBean {

	public void init() {
		// 초기화 작업 수행
	}
}
```

```kotlin
// Kotlin
class ExampleBean {
    fun init() {
        // 초기화 작업 수행
    }
}
```

앞의 예제는 다음 예제(두 개의 목록으로 구성됨)와 거의 동일한 효과를 가집니다:

```xml
<bean id="exampleInitBean" class="examples.AnotherExampleBean"/>
```

```java
// Java
public class AnotherExampleBean implements InitializingBean {

	@Override
	public void afterPropertiesSet() {
		// 초기화 작업 수행
	}
}
```

```kotlin
// Kotlin
class AnotherExampleBean : InitializingBean {
    override fun afterPropertiesSet() {
        // 초기화 작업 수행
    }
}
```

그러나 앞의 두 예제 중 첫 번째는 코드를 스프링에 결합시키지 않습니다.

`*@PostConstruct` 및 일반적인 초기화 메소드는 컨테이너의 싱글톤 생성 잠금(lock) 내에서 실행된다는 점에 유의하십시오.*

*빈 인스턴스는 `@PostConstruct` 메소드에서 반환된 후에야 완전히 초기화되고 다른 곳에 게시될 준비가 된 것으로 간주됩니다.*

*이러한 개별 초기화 메소드는 구성 상태를 검증하고 주어진 구성을 기반으로 일부 데이터 구조를 준비하기 위한 것이며, 외부 빈 접근을 포함한 추가 활동은 없습니다. 그렇지 않으면 초기화 교착 상태(deadlock)의 위험이 있습니다.*

*예를 들어, 비동기 데이터베이스 준비 단계와 같이 비용이 많이 드는 후-초기화(post-initialization) 활동을 트리거해야 하는 시나리오의 경우,*

*빈은 `SmartInitializingSingleton.afterSingletonsInstantiated()`를 구현하거나 컨텍스트 리프레시 이벤트에 의존해야 합니다:*

`*ApplicationListener<ContextRefreshedEvent>`를 구현하거나 해당 어노테이션 등가물인 `@EventListener(ContextRefreshedEvent.class)`를 선언합니다.*

*이러한 변형은 모든 일반 싱글톤 초기화 이후에 발생하므로 싱글톤 생성 잠금 외부에서 이루어집니다.또는 (Smart)Lifecycle 인터페이스를 구현하고 자동 시작 메커니즘,*

*사전 소멸 중지 단계 및 잠재적인 중지/재시작 콜백(아래 참조)을 포함하여 컨테이너의 전반적인 생명주기 관리와 통합할 수 있습니다.*

*소멸 콜백 (Destruction Callbacks)*`org.springframework.beans.factory.DisposableBean` 인터페이스를 구현하면 빈을 포함하는 컨테이너가 소멸될 때 빈이 콜백을 받을 수 있습니다.

`DisposableBean` 인터페이스는 단일 메소드를 지정합니다:

```java
void destroy() throws Exception;
```

`DisposableBean` 콜백 인터페이스는 코드를 불필요하게 스프링에 결합시키므로 사용하지 않는 것이 좋습니다.

대신 `@PreDestroy` 어노테이션을 사용하거나 빈 정의에서 지원하는 일반 메소드를 지정하는 것을 제안합니다.

XML 기반 설정 메타데이터에서는 `<bean/>`에 `destroy-method` 속성을 사용할 수 있습니다.

자바 구성에서는 `@Bean`의 `destroyMethod` 속성을 사용할 수 있습니다. 생명주기 콜백 받기(Receiving Lifecycle Callbacks)를 참조하십시오.

```xml
<bean id="exampleDestructionBean" class="examples.ExampleBean" destroy-method="cleanup"/>
```

```java
// Java
public class ExampleBean {

	public void cleanup() {
		// 소멸 작업 수행 (예: 풀링된 연결 해제)
	}
}
```

```kotlin
// Kotlin
class ExampleBean {
    fun cleanup() {
        // 소멸 작업 수행 (예: 풀링된 연결 해제)
    }
}
```

앞의 정의는 다음 정의와 거의 동일한 효과를 가집니다:

```xml
<bean id="exampleDestructionBean" class="examples.AnotherExampleBean"/>
```

```java
// Java
public class AnotherExampleBean implements DisposableBean {

	@Override
	public void destroy() {
		// 소멸 작업 수행 (예: 풀링된 연결 해제)
	}
}
```

```kotlin
// Kotlin
class AnotherExampleBean : DisposableBean {
    override fun destroy() {
        // 소멸 작업 수행 (예: 풀링된 연결 해제)
    }
}
```

그러나 앞의 두 정의 중 첫 번째는 코드를 스프링에 결합시키지 않습니다.

*스프링은 또한 소멸 메소드 추론(inference)을 지원하여 public `close` 또는 `shutdown` 메소드를 감지한다는 점에 유의하십시오.*

*이는 자바 구성 클래스의 `@Bean` 메소드에 대한 기본 동작이며 `java.lang.AutoCloseable` 또는 `java.io.Closeable` 구현과 자동으로 일치하므로 소멸 로직을 스프링에 결합시키지 않습니다.*

*XML을 사용한 소멸 메소드 추론의 경우, `<bean>` 요소의 `destroy-method` 속성에 특별한 (추론된) 값을 할당할 수 있으며, 이는 스프링에게 특정 빈 정의에 대한 빈 클래스의 public `close` 또는 `shutdown` 메소드를 자동으로 감지하도록 지시합니다. `<beans>` 요소의 `default-destroy-method` 속성에 이 특별한 (추론된) 값을 설정하여 전체 빈 정의 집합에 이 동작을 적용할 수도 있습니다 (기본 초기화 및 소멸 메소드 참조).*

*확장된 종료 단계를 위해, `Lifecycle` 인터페이스를 구현하고 모든 싱글톤 빈의 소멸 메소드가 호출되기 전에 조기 중지 신호(early stop signal)를 받을 수 있습니다.*

*또한 컨테이너가 모든 이러한 중지 처리가 완료될 때까지 대기한 후 소멸 메소드로 이동하는 시간 제한이 있는 중지 단계를 위해 `SmartLifecycle`을 구현할 수도 있습니다.*

*기본 초기화 및 소멸 메소드 (Default Initialization and Destroy Methods)*
스프링 특정의 `InitializingBean` 및 `DisposableBean` 콜백 인터페이스를 사용하지 않는 초기화 및 소멸 메소드 콜백을 작성할 때, 일반적으로 `init()`, `initialize()`, `dispose()` 등과 같은 이름의 메소드를 작성합니다.

이상적으로는 이러한 생명주기 콜백 메소드의 이름이 프로젝트 전체에 걸쳐 표준화되어 모든 개발자가 동일한 메소드 이름을 사용하고 일관성을 보장하도록 합니다.

스프링 컨테이너가 모든 빈에서 명명된 초기화 및 소멸 콜백 메소드 이름을 "찾도록" 구성할 수 있습니다.

이는 애플리케이션 개발자인 여러분이 각 빈 정의에 `init-method="init"` 속성을 구성할 필요 없이 `init()`이라는 초기화 콜백을 사용하여 애플리케이션 클래스를 작성할 수 있음을 의미합니다.

스프링 IoC 컨테이너는 빈이 생성될 때 (그리고 이전에 설명한 표준 생명주기 콜백 계약에 따라) 해당 메소드를 호출합니다.

이 기능은 또한 초기화 및 소멸 메소드 콜백에 대한 일관된 이름 지정 규칙을 강제합니다.

초기화 콜백 메소드의 이름이 `init()`이고 소멸 콜백 메소드의 이름이 `destroy()`라고 가정해 봅시다.

```java
// Java
public class DefaultBlogService implements BlogService {

	private BlogDao blogDao;

	public void setBlogDao(BlogDao blogDao) {
		this.blogDao = blogDao;
	}

	public void init() {
		if (this.blogDao == null) {
			throw new IllegalStateException("The [blogDao] property must be set.");
		}
	}
}
```

```kotlin
// Kotlin
class DefaultBlogService : BlogService {
    private var blogDao: BlogDao? = null

    fun setBlogDao(blogDao: BlogDao) {
        this.blogDao = blogDao
    }

    fun init() {
        checkNotNull(blogDao) { "The [blogDao] property must be set." }
    }
}
```

그런 다음 다음과 유사한 빈에서 해당 클래스를 사용할 수 있습니다:

```xml
<beans default-init-method="init">

	<bean id="blogService" class="com.something.DefaultBlogService">
		<property name="blogDao" ref="blogDao" />
	</bean>

</beans>
```

최상위 `<beans/>` 요소 속성에 `default-init-method` 속성이 있으면 스프링 IoC 컨테이너가 빈 클래스의 `init`이라는 메소드를 초기화 메소드 콜백으로 인식하게 됩니다.

빈이 생성되고 조립될 때, 빈 클래스에 그러한 메소드가 있으면 적절한 시점에 호출됩니다.

최상위 `<beans/>` 요소의 `default-destroy-method` 속성을 사용하여 유사하게 (즉, XML에서) 소멸 메소드 콜백을 구성할 수 있습니다.

기존 빈 클래스에 이미 관례와 다른 이름의 콜백 메소드가 있는 경우, `<bean/>` 자체의 `init-method` 및 `destroy-method` 속성을 사용하여 (즉, XML에서) 메소드 이름을 지정하여 기본값을 오버라이드할 수 있습니다.

스프링 컨테이너는 구성된 초기화 콜백이 빈에 모든 의존성이 제공된 직후에 호출됨을 보장합니다.

따라서 초기화 콜백은 원시 빈 참조(raw bean reference)에서 호출되며, 이는 AOP 인터셉터 등이 아직 빈에 적용되지 않았음을 의미합니다.

대상 빈이 먼저 완전히 생성된 다음 해당 인터셉터 체인이 있는 AOP 프록시(예:)가 적용됩니다. 대상 빈과 프록시가 별도로 정의된 경우, 코드는 프록시를 우회하여 원시 대상 빈과 직접 상호 작용할 수도 있습니다.

따라서 `init` 메소드에 인터셉터를 적용하는 것은 일관성이 없습니다. 왜냐하면 그렇게 하면 대상 빈의 생명주기를 프록시 또는 인터셉터에 결합시키고 코드가 원시 대상 빈과 직접 상호 작용할 때 이상한 의미론을 남기기 때문입니다.

*생명주기 메커니즘 결합 (Combining Lifecycle Mechanisms)*
스프링 2.5부터 빈 생명주기 동작을 제어하는 세 가지 옵션이 있습니다:

- `InitializingBean` 및 `DisposableBean` 콜백 인터페이스
- 커스텀 `init()` 및 `destroy()` 메소드
- `@PostConstruct` 및 `@PreDestroy` 어노테이션

주어진 빈을 제어하기 위해 이러한 메커니즘들을 결합할 수 있습니다.

*빈에 대해 여러 생명주기 메커니즘이 구성되고 각 메커니즘이 다른 메소드 이름으로 구성된 경우, 각 구성된 메소드는 이 노트 다음에 나열된 순서대로 실행됩니다.*

*그러나 동일한 메소드 이름이 (예: 초기화 메소드의 경우 `init()`) 이러한 생명주기 메커니즘 중 둘 이상에 대해 구성된 경우, 해당 메소드는 앞 섹션에서 설명한 대로 한 번만 실행됩니다.*

다른 초기화 메소드로 동일한 빈에 대해 구성된 여러 생명주기 메커니즘은 다음과 같이 호출됩니다:

1. `@PostConstruct`로 어노테이션된 메소드
2. `InitializingBean` 콜백 인터페이스에서 정의된 `afterPropertiesSet()`
3. 커스텀 구성된 `init()` 메소드

소멸 메소드는 동일한 순서로 호출됩니다:

1. `@PreDestroy`로 어노테이션된 메소드
2. `DisposableBean` 콜백 인터페이스에서 정의된 `destroy()`
3. 커스텀 구성된 `destroy()` 메소드

*시작 및 종료 콜백 (Startup and Shutdown Callbacks)*`Lifecycle` 인터페이스는 자체 생명주기 요구 사항(예: 일부 백그라운드 프로세스 시작 및 중지)을 가진 모든 객체에 대한 필수 메소드를 정의합니다:

```java
public interface Lifecycle {

	void start();

	void stop();

	boolean isRunning();
}
```

모든 스프링 관리 객체는 `Lifecycle` 인터페이스를 구현할 수 있습니다.

그러면 `ApplicationContext` 자체가 시작 및 중지 신호(예: 런타임 시 중지/재시작 시나리오)를 받으면 해당 컨텍스트 내에 정의된 모든 `Lifecycle` 구현에 해당 호출을 연쇄적으로 전달합니다.

이는 다음 목록에 표시된 `LifecycleProcessor`에 위임하여 수행합니다:

```java
public interface LifecycleProcessor extends Lifecycle {

	void onRefresh();

	void onClose();
}
```

`LifecycleProcessor` 자체가 `Lifecycle` 인터페이스의 확장이라는 점에 주목하십시오.

또한 컨텍스트가 리프레시되고 닫힐 때 반응하는 두 가지 다른 메소드를 추가합니다.

*일반 `org.springframework.context.Lifecycle` 인터페이스는 명시적인 시작 및 중지 알림을 위한 단순한 계약이며 컨텍스트 리프레시 시 자동 시작을 의미하지 않는다는 점에 유의하십시오.*

*자동 시작에 대한 세분화된 제어 및 특정 빈의 정상적인 중지(시작 및 중지 단계 포함)를 위해서는 대신 확장된 `org.springframework.context.SmartLifecycle` 인터페이스 구현을 고려하십시오.*

*또한 중지 알림이 소멸 전에 온다는 보장은 없습니다. 일반 종료 시 모든 `Lifecycle` 빈은 일반적인 소멸 콜백이 전파되기 전에 먼저 중지 알림을 받습니다.*

*그러나 컨텍스트의 수명 동안 핫 리프레시(hot refresh) 또는 중지된 리프레시 시도 시에는 소멸 메소드만 호출됩니다.*

시작 및 종료 호출 순서가 중요할 수 있습니다. 두 객체 간에 "depends-on" 관계가 존재하는 경우, 종속 측은 의존성 이후에 시작하고 의존성 전에 중지합니다.

그러나 때로는 직접적인 의존성을 알 수 없습니다. 특정 타입의 객체가 다른 타입의 객체보다 먼저 시작해야 한다는 것만 알 수 있습니다.

이러한 경우 `SmartLifecycle` 인터페이스는 또 다른 옵션, 즉 슈퍼 인터페이스인 `Phased`에 정의된 `getPhase()` 메소드를 정의합니다. 다음 목록은 `Phased` 인터페이스의 정의를 보여줍니다:

```java
public interface Phased {

	int getPhase();
}
```

다음 목록은 `SmartLifecycle` 인터페이스의 정의를 보여줍니다:

```java
public interface SmartLifecycle extends Lifecycle, Phased {

	boolean isAutoStartup();

	void stop(Runnable callback);
}
```

시작할 때 가장 낮은 단계(phase)를 가진 객체가 먼저 시작합니다.

중지할 때는 역순이 따릅니다. 따라서 `SmartLifecycle`을 구현하고 `getPhase()` 메소드가 `Integer.MIN_VALUE`를 반환하는 객체는 가장 먼저 시작하고 가장 마지막에 중지하는 객체 중 하나가 됩니다.
스펙트럼의 다른 쪽 끝에서 `Integer.MAX_VALUE`의 단계 값은 객체가 마지막에 시작되고 먼저 중지되어야 함을 나타냅니다 (다른 프로세스가 실행 중이어야 의존하기 때문일 가능성이 높음).

단계 값을 고려할 때 `SmartLifecycle`을 구현하지 않는 모든 "일반" `Lifecycle` 객체의 기본 단계는 0이라는 것을 아는 것도 중요합니다.

따라서 음수 단계 값은 객체가 이러한 표준 구성 요소보다 먼저 시작해야 함(그리고 그 이후에 중지해야 함)을 나타냅니다. 양수 단계 값의 경우 그 반대입니다.

`SmartLifecycle`에서 정의된 `stop` 메소드는 콜백을 받습니다.

모든 구현은 해당 구현의 종료 프로세스가 완료된 후 해당 콜백의 `run()` 메소드를 호출해야 합니다.

이는 필요한 경우 비동기 종료를 가능하게 합니다. 왜냐하면 `LifecycleProcessor` 인터페이스의 기본 구현인 `DefaultLifecycleProcessor`는 각 단계 내의 객체 그룹이 해당 콜백을 호출할 때까지 타임아웃 값까지 대기하기 때문입니다.

기본 단계별 타임아웃은 30초입니다. 컨텍스트 내에 `lifecycleProcessor`라는 이름의 빈을 정의하여 기본 생명주기 프로세서 인스턴스를 오버라이드할 수 있습니다. 타임아웃만 수정하려면 다음을 정의하는 것으로 충분합니다:

```xml
<bean id="lifecycleProcessor" class="org.springframework.context.support.DefaultLifecycleProcessor">
	<!-- 밀리초 단위의 타임아웃 값 -->
	<property name="timeoutPerShutdownPhase" value="10000"/>
</bean>
```

앞서 언급했듯이 `LifecycleProcessor` 인터페이스는 컨텍스트의 리프레시 및 닫기에 대한 콜백 메소드도 정의합니다.

후자는 `stop()`이 명시적으로 호출된 것처럼 종료 프로세스를 구동하지만 컨텍스트가 닫힐 때 발생합니다. 반면 '리프레시' 콜백은 `SmartLifecycle` 빈의 또 다른 기능을 활성화합니다.

컨텍스트가 리프레시될 때(모든 객체가 인스턴스화되고 초기화된 후), 해당 콜백이 호출됩니다.

그 시점에서 기본 생명주기 프로세서는 각 `SmartLifecycle` 객체의 `isAutoStartup()` 메소드가 반환하는 부울 값을 확인합니다.

`true`이면 해당 객체는 컨텍스트나 자체 `start()` 메소드의 명시적 호출을 기다리지 않고 그 시점에 시작됩니다(컨텍스트 리프레시와 달리 표준 컨텍스트 구현의 경우 컨텍스트 시작은 자동으로 발생하지 않음).

단계 값 및 모든 "depends-on" 관계는 앞에서 설명한 대로 시작 순서를 결정합니다.

*비-웹 애플리케이션에서 스프링 IoC 컨테이너 정상적으로 종료하기 (Shutting Down the Spring IoC Container Gracefully in Non-Web Applications)이 섹션은 비-웹 애플리케이션에만 적용됩니다.*

*스프링의 웹 기반 `ApplicationContext` 구현체는 관련 웹 애플리케이션이 종료될 때 스프링 IoC 컨테이너를 정상적으로 종료하는 코드를 이미 가지고 있습니다.*

비-웹 애플리케이션 환경(예: 리치 클라이언트 데스크톱 환경)에서 스프링의 IoC 컨테이너를 사용하는 경우, JVM에 종료 훅(shutdown hook)을 등록하십시오.

이렇게 하면 정상적인 종료가 보장되고 싱글톤 빈의 관련 소멸 메소드가 호출되어 모든 리소스가 해제됩니다.

여전히 이러한 소멸 콜백을 올바르게 구성하고 구현해야 합니다.

종료 훅을 등록하려면 다음 예제와 같이 `ConfigurableApplicationContext` 인터페이스에 선언된 `registerShutdownHook()` 메소드를 호출합니다:

```java
// Java
import org.springframework.context.ConfigurableApplicationContext;
import org.springframework.context.support.ClassPathXmlApplicationContext;

public final class Boot {

	public static void main(final String[] args) throws Exception {
		ConfigurableApplicationContext ctx = new ClassPathXmlApplicationContext("beans.xml");

		// 위 컨텍스트에 대한 종료 훅 추가...
		ctx.registerShutdownHook();

		// 앱이 여기서 실행됨...

		// main 메소드 종료, 훅은 앱 종료 전에 호출됨...
	}
}
```

```kotlin
// Kotlin
import org.springframework.context.ConfigurableApplicationContext
import org.springframework.context.support.ClassPathXmlApplicationContext

fun main(args: Array<String>) {
    val ctx: ConfigurableApplicationContext = ClassPathXmlApplicationContext("beans.xml")

    // 위 컨텍스트에 대한 종료 훅 추가...
    ctx.registerShutdownHook()

    // 앱이 여기서 실행됨...

    // main 함수 종료, 훅은 앱 종료 전에 호출됨...
}
```

*스레드 안전성 및 가시성 (Thread Safety and Visibility)*
스프링 코어 컨테이너는 생성된 싱글톤 인스턴스를 스레드 안전(thread-safe)한 방식으로 게시(publish)하며, 싱글톤 잠금(singleton lock)을 통해 접근을 보호하고 다른 스레드에서의 가시성(visibility)을 보장합니다.

결과적으로, 애플리케이션에서 제공하는 빈 클래스는 초기화 상태의 가시성에 대해 걱정할 필요가 없습니다.

일반적인 구성 필드는 초기화 단계 동안에만 변경되는 한 `volatile`로 표시할 필요가 없으며, 해당 초기 단계 동안 변경 가능한 세터 기반 구성 상태에 대해서도 `final`과 유사한 가시성 보장을 제공합니다.

이러한 필드가 빈 생성 단계 및 후속 초기 게시 이후에 변경되면, 접근할 때마다 `volatile`로 선언하거나 공통 잠금으로 보호해야 합니다.

*싱글톤 빈 인스턴스의 이러한 구성 상태에 대한 동시 접근(예: 컨트롤러 인스턴스 또는 리포지토리 인스턴스)은 컨테이너 측에서 이러한 안전한 초기 게시 이후에 완벽하게 스레드 안전하다는 점에 유의하십시오.*

*이는 일반 싱글톤 잠금 내에서 처리되는 공통 싱글톤 `FactoryBean` 인스턴스도 포함합니다.*

*소멸 콜백의 경우 구성 상태는 스레드 안전하게 유지되지만, 초기화와 소멸 사이에 누적된 모든 런타임 상태는 일반적인 자바 지침에 따라 스레드 안전 구조(또는 간단한 경우 `volatile` 필드)에 보관해야 합니다.*

*위에 표시된 더 깊은 생명주기 통합은 `volatile`로 선언해야 하는 `runnable` 필드와 같은 런타임 변경 가능 상태를 포함합니다.*

*일반적인 생명주기 콜백은 특정 순서를 따르지만(예: 시작 콜백은 완전한 초기화 이후에만 발생하고 중지 콜백은 초기 시작 이후에만 발생함), 일반적인 소멸 전 중지 배열에는 특별한 경우가 있습니다:*

*취소된 부트스트랩 후 비정상적인 종료 또는 다른 빈으로 인한 중지 타임아웃의 경우 선행 중지 없이 즉각적인 소멸 콜백이 발생할 수 있으므로 이러한 빈의 내부 상태도 이를 허용하도록 강력히 권장됩니다.*

**`ApplicationContextAware` 및 `BeanNameAware`**

`ApplicationContext`가 `org.springframework.context.ApplicationContextAware` 인터페이스를 구현하는 객체 인스턴스를 생성할 때, 인스턴스에는 해당 `ApplicationContext`에 대한 참조가 제공됩니다.

```java
public interface ApplicationContextAware {

	void setApplicationContext(ApplicationContext applicationContext) throws BeansException;
}
```

따라서 빈은 `ApplicationContext` 인터페이스를 통해 또는 이 인터페이스의 알려진 하위 클래스(예: 추가 기능을 노출하는 `ConfigurableApplicationContext`)로

참조를 캐스팅하여 자신을 생성한 `ApplicationContext`를 프로그래밍 방식으로 조작할 수 있습니다. 한 가지 용도는 다른 빈의 프로그래밍 방식 검색입니다.

때로는 이 기능이 유용하지만 일반적으로는 코드를 스프링에 결합시키고 협력자가 빈에 속성으로 제공되는 제어의 역전 스타일을 따르지 않으므로 피해야 합니다.

`ApplicationContext`의 다른 메소드는 파일 리소스 접근, 애플리케이션 이벤트 게시 및 `MessageSource` 접근을 제공합니다.

이러한 추가 기능은 `ApplicationContext`의 추가 기능(Additional Capabilities of the ApplicationContext)에서 설명합니다.

자동 와이어링은 `ApplicationContext`에 대한 참조를 얻는 또 다른 대안입니다.

전통적인 `constructor` 및 `byType` 자동 와이어링 모드(협력자 자동 와이어링(Autowiring Collaborators)에서 설명됨)는 각각 생성자 인수 또는 세터 메소드 파라미터에 대해 `ApplicationContext` 타입의 의존성을 제공할 수 있습니다.

필드 및 여러 파라미터 메소드를 자동 와이어링하는 기능을 포함하여 더 많은 유연성을 위해 어노테이션 기반 자동 와이어링 기능을 사용하십시오.

그렇게 하면 해당 필드, 생성자 또는 메소드에 `@Autowired` 어노테이션이 있는 경우 필드, 생성자 인수 또는 메소드 파라미터가 `ApplicationContext` 타입을 기대할 때 `ApplicationContext`가 자동 와이어링됩니다.

자세한 내용은 `@Autowired` 사용하기를 참조하십시오.

`ApplicationContext`가 `org.springframework.beans.factory.BeanNameAware` 인터페이스를 구현하는 클래스를 생성할 때, 클래스에는 관련 객체 정의에 정의된 이름에 대한 참조가 제공됩니다.

다음 목록은 `BeanNameAware` 인터페이스의 정의를 보여줍니다:

```java
public interface BeanNameAware {

	void setBeanName(String name) throws BeansException;
}
```

콜백은 일반 빈 속성 채우기(population) 후, 하지만 `InitializingBean.afterPropertiesSet()` 또는 커스텀 `init-method`와 같은 초기화 콜백 전에 호출됩니다.

**기타 `Aware` 인터페이스 (Other Aware Interfaces)**

앞서 논의한 `ApplicationContextAware` 및 `BeanNameAware` 외에도, 스프링은 빈이 특정 인프라 의존성이 필요함을 컨테이너에 나타낼 수 있도록 하는 광범위한 `Aware` 콜백 인터페이스를 제공합니다.

일반적으로 이름은 의존성 타입을 나타냅니다. 다음 표는 가장 중요한 `Aware` 인터페이스를 요약합니다:

| 이름 | 주입된 의존성 | description |
| --- | --- | --- |
| `ApplicationContextAware` | `ApplicationContext` 선언. | `ApplicationContextAware` 및 `BeanNameAware` |
| `ApplicationEventPublisherAware` | 둘러싸는 `ApplicationContext`의 이벤트 발행자(Event publisher). | `ApplicationContext`의 추가 기능 |
| `BeanClassLoaderAware` | 빈 클래스를 로드하는 데 사용된 클래스 로더. | 빈 인스턴스화 |
| `BeanFactoryAware` | `BeanFactory` 선언. | BeanFactory API |
| `BeanNameAware` | 선언하는 빈의 이름. | `ApplicationContextAware` 및 `BeanNameAware` |
| `LoadTimeWeaverAware` | 로드 시점에 클래스 정의 처리를 위한 정의된 위버(weaver). | 스프링 프레임워크에서의 AspectJ를 사용한 로드 타임 위빙 |
| `MessageSourceAware` | 메시지 해석을 위한 구성된 전략 (파라미터화 및 국제화 지원 포함). | `ApplicationContext`의 추가 기능 |
| `NotificationPublisherAware` | 스프링 JMX 알림 발행자. | 알림(Notifications) |
| `ResourceLoaderAware` | 리소스에 대한 저수준 접근을 위한 구성된 로더. | 리소스(Resources) |
| `ServletConfigAware` | 컨테이너가 실행 중인 현재 `ServletConfig`. 웹 인식 스프링 `ApplicationContext`에서만 유효. | 스프링 MVC |
| `ServletContextAware` | 컨테이너가 실행 중인 현재 `ServletContext`. 웹 인식 스프링 `ApplicationContext`에서만 유효. | 스프링 MVC |
