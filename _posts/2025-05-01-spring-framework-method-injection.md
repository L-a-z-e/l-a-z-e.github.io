---
title: Spring Framework Method Injection
description: 
author: laze
date: 2025-05-01 00:00:06 +0900
categories: [Dev, SpringBoot]
tags: [SpringBoot]
---
# **Method Injection**

대부분의 애플리케이션 시나리오에서 컨테이너의 대부분 빈은 싱글톤입니다.

싱글톤 빈이 다른 싱글톤 빈과 협력해야 하거나, 비-싱글톤 빈이 다른 비-싱글톤 빈과 협력해야 할 때, 일반적으로 한 빈을 다른 빈의 속성으로 정의하여 의존성을 처리합니다.

빈 생명주기가 다를 때 문제가 발생합니다.

싱글톤 빈 A가 비-싱글톤(프로토타입) 빈 B를 사용해야 한다고 가정해 봅시다.

아마도 A에 대한 각 메소드 호출마다 말이죠.

컨테이너는 싱글톤 빈 A를 한 번만 생성하므로 속성을 설정할 기회가 한 번뿐입니다.

컨테이너는 필요할 때마다 빈 A에게 빈 B의 새로운 인스턴스를 제공할 수 없습니다.

한 가지 해결책은 제어의 역전 일부를 포기하는 것입니다.

`ApplicationContextAware` 인터페이스를 구현하여 빈 A가 컨테이너를 인식하게 하고, 빈 A가 필요할 때마다 컨테이너에 `getBean("B")` 호출을 하여 (일반적으로 새로운) 빈 B 인스턴스를 요청하도록 할 수 있습니다.

다음 예제는 이 접근 방식을 보여줍니다:

```java
// Java
package fiona.apple;

// Spring-API 임포트
import org.springframework.beans.BeansException;
import org.springframework.context.ApplicationContext;
import org.springframework.context.ApplicationContextAware;

/**
 * 상태를 가지는 Command 스타일 클래스를 사용하여
 * 일부 처리를 수행하는 클래스.
 */
public class CommandManager implements ApplicationContextAware {

	private ApplicationContext applicationContext;

	public Object process(Map commandState) {
		// 적절한 Command의 새 인스턴스를 가져옵니다
		Command command = createCommand();
		// Command 인스턴스에 상태를 설정합니다
		command.setState(commandState);
		return command.execute();
	}

	protected Command createCommand() {
		// Spring API 의존성 주목!
		return this.applicationContext.getBean("command", Command.class);
	}

	public void setApplicationContext(
			ApplicationContext applicationContext) throws BeansException {
		this.applicationContext = applicationContext;
	}
}
```

```kotlin
// Kotlin
package fiona.apple

// Spring-API 임포트
import org.springframework.beans.BeansException
import org.springframework.context.ApplicationContext
import org.springframework.context.ApplicationContextAware

/**
 * 상태를 가지는 Command 스타일 클래스를 사용하여
 * 일부 처리를 수행하는 클래스.
 */
class CommandManager : ApplicationContextAware {

    private var applicationContext: ApplicationContext? = null

    fun process(commandState: Map<*, *>): Any? {
        // 적절한 Command의 새 인스턴스를 가져옵니다
        val command = createCommand()
        // (새것이길 바라는) Command 인스턴스에 상태를 설정합니다
        command.state = commandState // Assuming Command has a state property
        return command.execute()
    }

    protected fun createCommand(): Command {
        // Spring API 의존성 주목!
        return this.applicationContext!!.getBean("command", Command::class.java)
    }

    override fun setApplicationContext(applicationContext: ApplicationContext) {
        this.applicationContext = applicationContext
    }
}
```

앞의 코드는 비즈니스 코드가 스프링 프레임워크를 인식하고 결합(coupled)되기 때문에 바람직하지 않습니다.

스프링 IoC 컨테이너의 다소 고급 기능인 메소드 주입(Method Injection)을 사용하면 이 사용 사례를 깔끔하게 처리할 수 있습니다.

*메소드 주입의 동기에 대해 이 블로그 게시물에서 더 자세히 읽을 수 있습니다.*

**Lookup Method Injection**

룩업 메소드 주입은 컨테이너가 컨테이너 관리 빈의 메소드를 오버라이드(override)하고 컨테이너 내의 다른 이름 붙여진 빈에 대한 룩업(lookup) 결과를 반환하는 기능입니다.

룩업은 일반적으로 앞 섹션에서 설명한 시나리오처럼 프로토타입 빈을 포함합니다.

스프링 프레임워크는 이 메소드 주입을 CGLIB 라이브러리의 바이트코드 생성을 사용하여 메소드를 오버라이드하는 하위 클래스를 동적으로 생성함으로써 구현합니다.

*이 동적 하위 클래스 생성이 작동하려면, 스프링 빈 컨테이너가 하위 클래스로 만들 클래스는 `final`이 아니어야 하며, 오버라이드될 메소드도 `final`이 아니어야 합니다.*

*추상 메소드가 있는 클래스를 단위 테스트하려면 직접 클래스를 하위 클래스로 만들고 추상 메소드의 스텁(stub) 구현을 제공해야 합니다.*

*또 다른 주요 제한 사항은 룩업 메소드가 팩토리 메소드, 특히 설정 클래스의 `@Bean` 메소드와 함께 작동하지 않는다는 것입니다.*

*왜냐하면 그 경우 컨테이너가 인스턴스 생성을 책임지지 않으므로 런타임 생성 하위 클래스를 즉석에서 생성할 수 없기 때문입니다.*

이전 코드 스니펫의 `CommandManager` 클래스의 경우, 스프링 컨테이너는 `createCommand()` 메소드의 구현을 동적으로 오버라이드합니다. 재작업된 예제가 보여주듯이 `CommandManager` 클래스는 스프링 의존성을 가지지 않습니다:

```java
// Java
package fiona.apple;

// 더 이상 Spring 임포트 없음!

public abstract class CommandManager {

	public Object process(Object commandState) {
		// 적절한 Command 인터페이스의 새 인스턴스를 가져옵니다
		Command command = createCommand();
		// (새것이길 바라는) Command 인스턴스에 상태를 설정합니다
		command.setState(commandState);
		return command.execute();
	}
	
	protected abstract Command createCommand();
}
```

```kotlin
// Kotlin
package fiona.apple

// 더 이상 Spring 임포트 없음!

abstract class CommandManager {

    fun process(commandState: Any): Any? {
        // 적절한 Command 인터페이스의 새 인스턴스를 가져옵니다
        val command = createCommand()
        // (새것이길 바라는) Command 인스턴스에 상태를 설정합니다
        command.state = commandState // Assuming Command has a state property
        return command.execute()
    }

    protected abstract fun createCommand(): Command
}
```

주입될 메소드를 포함하는 클라이언트 클래스(이 경우 `CommandManager`)에서, 주입될 메소드는 다음 형식의 시그니처(signature)를 요구합니다:

`<public|protected> [abstract] <return-type> theMethodName(no-arguments);`

메소드가 추상(abstract)이면 동적으로 생성된 하위 클래스가 메소드를 구현합니다.

그렇지 않으면 동적으로 생성된 하위 클래스가 원본 클래스에 정의된 구체적인(concrete) 메소드를 오버라이드합니다.

```xml
<!-- 프로토타입(비-싱글톤)으로 배포된 상태 저장 빈 -->
<bean id="myCommand" class="fiona.apple.AsyncCommand" scope="prototype">
	<!-- 필요에 따라 여기에 의존성 주입 -->
</bean>

<!-- commandManager는 myCommand 프로토타입 빈을 사용 -->
<bean id="commandManager" class="fiona.apple.CommandManager">
	<lookup-method name="createCommand" bean="myCommand"/>
</bean>
```

`commandManager`로 식별된 빈은 `myCommand` 빈의 새 인스턴스가 필요할 때마다 자신의 `createCommand()` 메소드를 호출합니다.

실제로 필요한 경우 `myCommand` 빈을 프로토타입으로 설정 합니다. 만약 싱글톤이면 매번 동일한 `myCommand` 빈 인스턴스가 반환됩니다.

또는, 어노테이션 기반 컴포넌트 모델 내에서 `@Lookup` 어노테이션을 통해 룩업 메소드를 선언할 수 있습니다.

```java
// Java
public abstract class CommandManager {

	public Object process(Object commandState) {
		Command command = createCommand();
		command.setState(commandState);
		return command.execute();
	}

	@Lookup("myCommand")
	protected abstract Command createCommand();
}

```

```kotlin
// Kotlin
abstract class CommandManager {

    fun process(commandState: Any): Any? {
        val command = createCommand()
        command.state = commandState // Assuming Command has a state property
        return command.execute()
    }

    @Lookup("myCommand")
    protected abstract fun createCommand(): Command
}

```

또는, 더 관용적인(idiomatically) 방식으로, 룩업 메소드의 선언된 반환 타입에 대해 대상 빈이 해석(resolved)되는 것에 의존할 수 있습니다:

```java
// Java
public abstract class CommandManager {

	public Object process(Object commandState) {
		Command command = createCommand();
		command.setState(commandState);
		return command.execute();
	}

	@Lookup
	protected abstract Command createCommand();
}
```

```kotlin
// Kotlin
abstract class CommandManager {

    fun process(commandState: Any): Any? {
        val command = createCommand()
        command.state = commandState // Assuming Command has a state property
        return command.execute()
    }

    @Lookup // Return type (Command) is used to resolve the bean
    protected abstract fun createCommand(): Command
}
```

*스코프가 다른 대상 빈에 접근하는 또 다른 방법은 `ObjectFactory`/`Provider` 주입 지점입니다.*

*의존성으로서의 스코프 빈(Scoped Beans as Dependencies)을 참조하십시오.`org.springframework.beans.factory.config` 패키지의 `ServiceLocatorFactoryBean`도 유용할 수 있습니다.*

**임의 메소드 교체 (Arbitrary Method Replacement)**

빈의 임의 메소드를 다른 메소드 구현으로 교체하는 기능입니다.

XML 기반 설정 메타데이터를 사용하면, 배포된 빈에 대해 기존 메소드 구현을 다른 것으로 교체하기 위해 `replaced-method` 요소를 사용할 수 있습니다.

오버라이드하려는 `computeValue` 메서드가 있는 원본 클래스

```java
// Java
public class MyValueCalculator {

	public String computeValue(String input) {
		// 실제 코드 일부...
	}

	// 다른 메소드들...
}
```

```kotlin
// Kotlin
class MyValueCalculator {
    fun computeValue(input: String): String {
        // 실제 코드 일부...
        return "" // Placeholder
    }

    // 다른 메소드들...
}
```

`org.springframework.beans.factory.support.MethodReplacer` 인터페이스를 구현하는 클래스

```java
// Java
/**
 * MyValueCalculator의 기존 computeValue(String) 구현을
 * 오버라이드하는 데 사용하기 위한 것임
 */
public class ReplacementComputeValue implements MethodReplacer {

	public Object reimplement(Object o, Method m, Object[] args) throws Throwable {
		// 입력 값을 가져와 작업하고 계산된 결과를 반환
		String input = (String) args[0];
		...
		return ...;
	}
}
```

```kotlin
// Kotlin
/**
 * MyValueCalculator의 기존 computeValue(String) 구현을
 * 오버라이드하는 데 사용하기 위한 것임
 */
class ReplacementComputeValue : MethodReplacer {
    override fun reimplement(o: Any, m: Method, args: Array<Any?>): Any {
        // 입력 값을 가져와 작업하고 계산된 결과를 반환
        val input = args[0] as String
        // ...
        return "..." // Placeholder
    }
}
```

원본 클래스를 배포하고 메소드 오버라이드를 지정하는 빈 정의는 다음 예제와 유사합니다:

```xml
<bean id="myValueCalculator" class="x.y.z.MyValueCalculator">
	<!-- 임의 메소드 교체 -->
	<replaced-method name="computeValue" replacer="replacementComputeValue">
		<arg-type>String</arg-type>
	</replaced-method>
</bean>

<bean id="replacementComputeValue" class="a.b.c.ReplacementComputeValue"/>
```

`<replaced-method/>` 요소 내에서 하나 이상의 `<arg-type/>` 요소를 사용하여 오버라이드되는 메소드의 메소드 시그니처를 나타낼 수 있습니다. 인수에 대한 시그니처는 메소드가 오버로드되어 클래스 내에 여러 변형이 존재하는 경우에만 필요합니다. 편의상, 인수에 대한 타입 문자열은 정규화된 타입 이름의 하위 문자열일 수 있습니다. 예를 들어, 다음 모두는 `java.lang.String`과 일치합니다:

- `java.lang.String`
- `String`
- `Str`

인수의 개수만으로도 각 가능한 선택지를 구별하기에 충분한 경우가 많으므로, 이 단축 방법은 인수 타입과 일치하는 가장 짧은 문자열만 입력하게 하여 많은 타이핑을 절약할 수 있습니다.

---

### **Lookup Method Injection 보충 설명**

**1. 문제 상황: 싱글톤 빈이 프로토타입 빈을 계속 새로 필요로 할 때**

- **싱글톤 빈 (Singleton Bean):** 스프링에서 빈을 만들 때 기본 설정입니다. 컨테이너 안에 **딱 한 개만** 만들어지고, 어디서든 이 빈을 필요로 하면 항상 **같은 그 한 개의 객체**를 받습니다. (예: `CommandManager` 처럼 상태를 가지지 않고 기능만 제공하는 빈)
- **프로토타입 빈 (Prototype Bean):** 이 빈을 요청할 때마다 스프링 컨테이너는 **매번 새로운 객체**를 만들어서 줍니다. (예: `Command` 처럼 각 요청마다 고유한 상태를 가져야 하는 빈)

**문제는 이렇습니다:** 싱글톤 빈 (`CommandManager`)이 내부 로직을 처리할 때마다 **새로운** 프로토타입 빈 (`Command`) 객체가 필요하다고 가정해 봅시다.

만약 일반적인 의존성 주입(생성자나 세터 주입)으로 `CommandManager`에게 `Command`를 주입하면 어떻게 될까요? `CommandManager`는 싱글톤이라 **한 번만** 만들어집니다. 이때 주입받은 `Command` 객체도 **그때 딱 한 번**만 (새로 만들어져서) 주입됩니다. 그 후로는 `CommandManager`는 계속 **처음에 주입받았던 그 똑같은 `Command` 객체만** 사용하게 됩니다. 매번 새로운 `Command` 객체를 얻을 수가 없죠!

**2. 해결책: 룩업 메소드 주입 (Lookup Method Injection)**

이 문제를 해결하기 위해 스프링이 제공하는 방법이 '룩업 메소드 주입'입니다. 핵심 아이디어는 이렇습니다:

"싱글톤 빈(`CommandManager`)아, 네 안에 **특정 메소드(예: `createCommand()`)** 를 하나 정의해둬. 평소에는 네가 그 메소드를 직접 구현할 필요는 없어 (추상 메소드로 둬도 돼). 대신 **내가(스프링 컨테이너) 몰래 네 클래스를 상속받는 자식 클래스를 만들어서, 그 자식 클래스에서 그 메소드를 오버라이드(override)할게.** 오버라이드된 메소드는 호출될 때마다 **항상 새로운 프로토타입 빈(`myCommand`)을 찾아서(lookup) 반환하도록** 만들어줄게. 그리고 실제로 컨테이너에 등록되는 빈은 네 원래 클래스가 아니라, 내가 만든 이 **비밀 자식 클래스**의 객체야."

**3. 어떻게 동작하는가? (CGLIB 마법)**

- 스프링은 **CGLIB**라는 라이브러리를 사용해서 이 작업을 수행합니다. CGLIB는 자바 클래스의 바이트코드를 조작해서, 실행 시점에 (런타임에) **동적으로 새로운 자식 클래스를 만들어내는** 능력이 있습니다.
- `CommandManager` 클래스 정의에 `createCommand()` 라는 메소드가 있고, XML이나 어노테이션으로 "이 메소드는 `myCommand` 빈을 찾아주는 룩업 메소드야" 라고 지정하면, 스프링은 다음과 같이 동작합니다:
  1. `CommandManager`를 상속받는 `CommandManager$EnhancerBySpringCGLIB$$...` 같은 이름의 **새로운 클래스를 런타임에 생성**합니다.
  2. 이 새 클래스 안에서 `createCommand()` 메소드를 **오버라이드**합니다. 오버라이드된 메소드의 내용은 대략 "스프링 컨테이너에게 `myCommand`라는 이름의 빈을 새로 달라고 요청해서 반환한다"는 코드로 채워집니다.
  3. 우리가 스프링 컨테이너에게 `commandManager` 빈을 달라고 요청하면, 스프링은 원본 `CommandManager` 객체가 아니라, 이 **CGLIB으로 만든 자식 클래스의 객체**를 줍니다.
- 따라서, 우리가 `commandManager.createCommand()`를 호출하면, 실제로는 **CGLIB이 만든 자식 클래스의 오버라이드된 메소드**가 실행되고, 이 메소드는 스프링 컨테이너에게 "야, `myCommand` 프로토타입 빈 새로 하나 줘!"라고 요청해서 **매번 새로운 `Command` 객체**를 받아 반환하게 됩니다.

**4. 예시 코드 설명 (`CommandManager`)**

```java
public abstract class CommandManager { // abstract 키워드 주목!

    public Object process(Object commandState) {
        // commandManager.createCommand() 를 호출하면,
        // 실제로는 CGLIB 자식 클래스의 오버라이드된 메소드가 실행됨
        Command command = createCommand(); // ★ 매번 새로운 Command 객체를 얻음 ★
        command.setState(commandState);
        return command.execute();
    }

    // 이 메소드를 직접 구현할 필요 없음 (abstract)
    // 스프링이 CGLIB 자식 클래스에서 이 메소드를 구현(또는 오버라이드) 해줌.
    // 이 메소드는 반드시 인자(argument)가 없어야 함.
    // protected 또는 public 이어야 함.
    protected abstract Command createCommand();
}

```

- `CommandManager`가 `abstract` 클래스이고 `createCommand()`가 `abstract` 메소드인 이유는, "나는 이 메소드를 어떻게 구현할지 몰라. 스프링 네가 알아서 해줘!"라는 의미를 명확히 하기 좋습니다. (꼭 `abstract`일 필요는 없고, 구체적인 메소드여도 스프링이 오버라이드합니다.)
- `process()` 메소드 안에서 `createCommand()`를 호출할 때마다, 결과적으로 스프링 컨테이너를 통해 **새로운 `Command` 프로토타입 빈**을 얻어오게 됩니다.

**5. 설정 방법**

- **XML:**

    ```xml
    <bean id="myCommand" class="fiona.apple.AsyncCommand" scope="prototype"/>
    
    <bean id="commandManager" class="fiona.apple.CommandManager">
        <!-- commandManager 빈의 createCommand 메소드는 myCommand 빈을 찾아서 반환하도록 설정 -->
        <lookup-method name="createCommand" bean="myCommand"/>
    </bean>
    
    ```

- **Annotation:**

    ```java
    @Component // commandManager를 빈으로 등록
    public abstract class CommandManager {
        // ... process 메소드 ...
    
        // createCommand 메소드는 "myCommand" 빈을 찾아서 반환하도록 설정
        @Lookup("myCommand")
        protected abstract Command createCommand();
        // 또는, 반환 타입(Command)으로 스프링이 찾을 빈을 유추할 수 있다면 이름 생략 가능
        // @Lookup
        // protected abstract Command createCommand();
    }
    
    @Component("myCommand") // myCommand 빈 등록
    @Scope("prototype")     // 프로토타입 스코프 설정
    public class AsyncCommand implements Command { ... }
    
    ```


**6. 제약 조건 (주의사항)**

- **`final` 금지:** CGLIB은 클래스를 상속받아 자식 클래스를 만들어야 하므로, 원본 클래스(`CommandManager`)나 룩업 메소드(`createCommand`)가 `final`로 선언되어 있으면 안 됩니다. (상속/오버라이드가 불가능해짐)
- **`@Bean` 메소드와 사용 불가:** 설정 클래스(`@Configuration`) 안의 `@Bean` 메소드로 빈을 정의하는 방식에서는 룩업 메소드 주입이 동작하지 않습니다. 왜냐하면 `@Bean` 메소드는 개발자가 직접 `new` 등을 사용해서 객체를 생성하고 반환하는데, 스프링이 중간에 끼어들어 CGLIB 자식 클래스를 만들 기회가 없기 때문입니다.
- **단위 테스트:** `abstract` 메소드가 있는 클래스를 테스트하려면, 테스트 코드에서 직접 해당 클래스를 상속받고 `abstract` 메소드를 임시로 구현(stub)해줘야 할 수 있습니다.

**요약:**

룩업 메소드 주입은 **싱글톤 빈이 필요할 때마다 프로토타입 빈의 새로운 인스턴스를 얻고 싶을 때** 사용하는 특별한 기술입니다. 스프링이 CGLIB이라는 마법을 부려 원본 클래스 대신 **특정 메소드가 오버라이드된 비밀 자식 클래스**를 만들어서 이 기능을 구현해줍니다. 덕분에 싱글톤 빈의 코드 자체는 깔끔하게 유지하면서 원하는 동작을 얻을 수 있습니다.

### Overloading 된 메서드를 교체할때

**핵심 아이디어:** 스프링에게 **정확히 어떤 메소드를 바꿔치기할지** 알려줘야 할 때가 있다는 것입니다.

**상황:** 만약 `MyValueCalculator` 클래스 안에 메소드가 이렇게 여러 개 있다고 가정하면

```java
public class MyValueCalculator {
    // 1번 메소드: String 하나를 받음
    public String computeValue(String input) { ... }

    // 2번 메소드: int 하나를 받음
    public String computeValue(int count) { ... }

    // 3번 메소드: 파라미터가 없음
    public String computeValue() { ... }

    // 4번 메소드: String 두 개를 받음
    public String computeValue(String input1, String input2) { ... }
}

```

이렇게 **이름은 같지만 받는 값(파라미터)의 종류나 개수가 다른 메소드**들이 여러 개 있는 것을 **메소드 오버로딩(Method Overloading)** 이라고 부릅니다.

**문제:** 이 상황에서 스프링 설정 파일에 그냥 이렇게만 쓰면 어떻게 되는가?

```xml
<bean id="myValueCalculator" class="x.y.z.MyValueCalculator">
	<replaced-method name="computeValue" replacer="replacementComputeValue" />
	<!-- ??? 어떤 computeValue를 바꾸라는 거지 ??? -->
</bean>
```

스프링은 **"어? `computeValue`라는 이름의 메소드가 여러 개인데, 도대체 몇 번 메소드를 바꿔치기 하라는 거야?"** 라고 혼란스러워 할 수 있습니다.

**해결책: `<arg-type>` 태그 사용**

이 혼란을 없애주기 위해 `<replaced-method>` 태그 안에 **`<arg-type>` 태그를 사용해서 바꿔치기할 메소드가 받는 값(파라미터)의 타입을 정확히 알려주는 것**입니다.

- **만약 1번 `computeValue(String input)` 메소드를 바꾸고 싶다면:**

    ```xml
    <replaced-method name="computeValue" replacer="replacementComputeValue">
        <arg-type>String</arg-type> <!-- "String 하나 받는 메서드로 바꿔줘!" -->
    </replaced-method>
    ```

- **만약 2번 `computeValue(int count)` 메소드를 바꾸고 싶다면:**

    ```xml
    <replaced-method name="computeValue" replacer="replacementComputeValue">
        <arg-type>int</arg-type> <!-- "int 하나 받는 메서드로 바꿔줘!" -->
    </replaced-method>
    
    ```

- **만약 3번 `computeValue()` 메소드 (파라미터 없는 것)를 바꾸고 싶다면:**

    ```xml
    <replaced-method name="computeValue" replacer="replacementComputeValue">
        <!-- 파라미터가 없으므로 <arg-type>을 안 쓰거나 비워둠 -->
    </replaced-method>
    
    ```

- **만약 4번 `computeValue(String input1, String input2)` 메소드를 바꾸고 싶다면:**

    ```xml
    <replaced-method name="computeValue" replacer="replacementComputeValue">
        <arg-type>String</arg-type> <!-- 첫 번째 파라미터는 String -->
        <arg-type>String</arg-type> <!-- 두 번째 파라미터도 String -->
        <!-- 파라미터 순서대로 <arg-type> 태그를 써줘야 함 -->
    </replaced-method>
    
    ```


**"인수에 대한 시그니처는 ... 오버로드된 경우에만 필요합니다."**

- 이 말은, 만약 `MyValueCalculator` 클래스에 `computeValue`라는 이름의 메소드가 **딱 하나만 있다면**, 굳이 `<arg-type>`을 써서 파라미터 타입을 알려주지 않아도 스프링이 알아서 그 메소드를 찾을 수 있다는 뜻입니다. (혼동할 여지가 없으니까요.)
- 하지만 메소드 이름이 같은 것이 여러 개(오버로딩) 있을 때는 **반드시** `<arg-type>`으로 구분해줘야 합니다.

**"타입 문자열은 ... 하위 문자열일 수 있습니다." (편의 기능)**

- 파라미터 타입을 쓸 때, 꼭 정식 이름(`java.lang.String`)을 다 쓰지 않아도 된다는 뜻입니다.
- 예를 들어 `java.lang.String` 대신 그냥 `String`이라고 써도 스프링이 알아듣습니다.
- 심지어 `Str`처럼 일부만 써도, 다른 타입과 헷갈리지만 않는다면 스프링이 "아, 이거 String 말하는 거구나" 하고 알아채 줄 수도 있다는 것입니다. (하지만 너무 줄여 쓰면 나중에 코드를 볼 때 헷갈릴 수 있으니 `String` 정도가 적당합니다.)

**"인수의 개수만으로도 ... 많은 타이핑을 절약할 수 있습니다."**

- 어떤 경우에는 파라미터의 **개수**만 달라도 스프링이 구분할 수 있다는 의미입니다. 예를 들어 `computeValue()` (0개)와 `computeValue(String)` (1개)가 있다면, `<arg-type>String</arg-type>` 하나만 써줘도 스프링은 "아, 파라미터 1개짜리 `computeValue`를 말하는구나" 하고 알아챌 가능성이 높습니다.
- 하지만 이것 역시 명확성을 위해 타입을 정확히 써주는 것이 더 좋은 습관입니다.

**결론:**

`<arg-type>`은 `<replaced-method>`를 사용할 때, **같은 이름의 메소드가 여러 개 있을 경우(오버로딩), 정확히 어떤 메소드를 교체할지 파라미터 타입 정보를 통해 스프링에게 명확하게 알려주는 역할**을 합니다.

오버로딩이 없다면 필수는 아니지만, 명확성을 위해 써주는 것이 좋습니다. 타입 이름은 편리하게 줄여 쓸 수도 있습니다.
