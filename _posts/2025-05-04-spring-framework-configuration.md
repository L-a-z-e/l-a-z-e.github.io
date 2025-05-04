---
title: Spring Framework @Bean
description: 
author: laze
date: 2025-05-04 00:00:08 +0900
categories: [Dev, SpringBoot]
tags: [SpringBoot]
---
**Using the @Configuration annotation**

`@Configuration`은 객체가 빈 정의의 소스임을 나타내는 클래스 레벨 어노테이션입니다.

`@Configuration` 클래스는 `@Bean`-어노테이션된 메소드를 통해 빈을 선언합니다.

`@Configuration` 클래스의 `@Bean` 메소드 호출은 빈 간의 의존성(inter-bean dependencies)을 정의하는 데 사용될 수도 있습니다.

**빈 간 의존성 주입하기 (Injecting Inter-bean Dependencies)**

빈들이 서로 의존성을 가질 때, 그 의존성을 표현하는 것은 한 빈 메소드가 다른 메소드를 호출하는 것만큼 간단합니다.

```java
// Java
@Configuration
public class AppConfig {

	@Bean
	public BeanOne beanOne() {
		return new BeanOne(beanTwo()); // Call beanTwo() method
	}

	@Bean
	public BeanTwo beanTwo() {
		return new BeanTwo();
	}
}
```

```kotlin
// Kotlin
@Configuration
class AppConfig {

    @Bean
    fun beanOne(): BeanOne {
        return BeanOne(beanTwo()) // Call beanTwo() method
    }

    @Bean
    fun beanTwo(): BeanTwo {
        return BeanTwo()
    }
}
// Assuming BeanOne and BeanTwo exist
```

앞의 예제에서 `beanOne`은 생성자 주입을 통해 `beanTwo`에 대한 참조를 받습니다.

*이 빈 간 의존성 선언 방식은 `@Bean` 메소드가 `@Configuration` 클래스 내에 선언될 때만 작동합니다.*

*일반 `@Component` 클래스를 사용하여 빈 간 의존성을 선언할 수는 없습니다.*

**룩업 메소드 주입 (Lookup Method Injection)**

싱글톤 스코프 빈이 프로토타입 스코프 빈에 대한 의존성을 가지는 경우에 유용합니다.

이러한 유형의 구성에 자바를 사용하면 이 패턴을 구현하는 자연스러운 수단을 제공합니다.

```java
// Java
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
abstract class CommandManager {
    fun process(commandState: Any): Any? {
        // 적절한 Command 인터페이스의 새 인스턴스를 가져옵니다
        val command = createCommand()
        // (새것이길 바라는) Command 인스턴스에 상태를 설정합니다
        command.state = commandState // Assuming Command has state property
        return command.execute()
    }

    protected abstract fun createCommand(): Command
}
// Assuming Command interface exists with state property and execute method
```

자바 구성을 사용하여, 추상 `createCommand()` 메소드가 새로운 (프로토타입) command 객체를 찾는 방식으로 오버라이드되는 `CommandManager`의 하위 클래스를 생성할 수 있습니다. 다음 예제는 이를 수행하는 방법을 보여줍니다:

```java
// Java
@Bean
@Scope("prototype")
public AsyncCommand asyncCommand() {
	AsyncCommand command = new AsyncCommand();
	// 필요에 따라 여기에 의존성 주입
	return command;
}

@Bean
public CommandManager commandManager() {
	// createCommand()가 새로운 프로토타입 Command 객체를 반환하도록
	// 오버라이드된 CommandManager의 익명 구현 반환
	return new CommandManager() {
		@Override // Add Override annotation for clarity
		protected Command createCommand() {
			return asyncCommand(); // Calls the prototype bean factory method
		}
	};
}
```

```kotlin
// Kotlin
@Bean
@Scope("prototype")
fun asyncCommand(): AsyncCommand {
    val command = AsyncCommand()
    // 필요에 따라 여기에 의존성 주입
    return command
}

@Bean
fun commandManager(): CommandManager {
    // createCommand()가 새로운 프로토타입 Command 객체를 반환하도록
    // 오버라이드된 CommandManager의 익명 구현 반환
    return object : CommandManager() {
        override fun createCommand(): Command {
            return asyncCommand() // Calls the prototype bean factory method
        }
    }
}
// Assuming AsyncCommand exists and implements Command
```

**자바 기반 구성이 내부적으로 작동하는 방식에 대한 추가 정보 (Further Information About How Java-based Configuration Works Internally)**

`@Bean` 어노테이션이 달린 메소드가 두 번 호출되는 예제

```java
// Java
@Configuration
public class AppConfig {

	@Bean
	public ClientService clientService1() {
		ClientServiceImpl clientService = new ClientServiceImpl();
		clientService.setClientDao(clientDao()); // Calls clientDao()
		return clientService;
	}

	@Bean
	public ClientService clientService2() {
		ClientServiceImpl clientService = new ClientServiceImpl();
		clientService.setClientDao(clientDao()); // Calls clientDao() again
		return clientService;
	}

	@Bean
	public ClientDao clientDao() {
		return new ClientDaoImpl();
	}
}
```

```kotlin
// Kotlin
@Configuration
class AppConfig {

    @Bean
    fun clientService1(): ClientService {
        val clientService = ClientServiceImpl()
        clientService.clientDao = clientDao() // Calls clientDao()
        return clientService
    }

    @Bean
    fun clientService2(): ClientService {
        val clientService = ClientServiceImpl()
        clientService.clientDao = clientDao() // Calls clientDao() again
        return clientService
    }

    @Bean
    fun clientDao(): ClientDao {
        return ClientDaoImpl()
    }
}
// Assuming ClientService, ClientServiceImpl, ClientDao, ClientDaoImpl exist,
// and ClientServiceImpl has a clientDao property/setter.
```

`clientDao()`는 `clientService1()`에서 한 번, `clientService2()`에서 한 번 호출되었습니다.

이 메소드는 `ClientDaoImpl`의 새 인스턴스를 생성하고 반환하므로, 일반적으로 두 개의 인스턴스(각 서비스당 하나)를 가질 것으로 예상됩니다.

이는 확실히 문제가 될 것입니다: 스프링에서 인스턴스화된 빈은 기본적으로 싱글톤 스코프를 가집니다.

모든 `@Configuration` 클래스는 시작 시점에 CGLIB를 사용하여 하위 클래스로 만들어집니다(subclassed).

하위 클래스에서 자식 메소드는 부모 메소드를 호출하고 새 인스턴스를 생성하기 전에 먼저 컨테이너에서 캐시된 (스코프 지정된) 빈이 있는지 확인합니다.

*동작은 빈의 스코프에 따라 다를 수 있습니다. (현재 예시는 싱글톤)*

*CGLIB 클래스는 `org.springframework.cglib` 패키지 아래에 재패키징되어 `spring-core` JAR 내에 직접 포함되어 있으므로, CGLIB를 클래스패스에 추가할 필요는 없습니다.*

CGLIB가 시작 시점에 동적으로 기능을 추가한다는 사실 때문에 몇 가지 제한 사항이 있습니다.

특히, 구성 클래스는 `final`이 아니어야 합니다.

그러나 구성 클래스에서는 `@Autowired` 사용 또는 기본 주입을 위한 단일 비-기본 생성자 선언을 포함하여 모든 생성자가 허용됩니다.

CGLIB에 의해 부과되는 제한 사항을 피하려면 `@Bean` 메소드를 비-`@Configuration` 클래스(예: 대신 일반 `@Component` 클래스)에 선언하거나 구성 클래스에 `@Configuration(proxyBeanMethods = false)`로 어노테이션을 달아야합니다. `@Bean` 메소드 간의 상호 메소드 호출은 가로채지지 않으므로, 생성자 또는 메소드 레벨에서만 의존성 주입에 전적으로 의존해야 합니다.

---

**전체 주제: `@Configuration` 어노테이션 사용하기**

이 부분은 자바 기반 설정을 위한 핵심 어노테이션인 `@Configuration`의 역할과 특징, 그리고 다른 빈과의 의존성을 설정하는 방법 및 내부 동작 원리에 대해 자세히 설명합니다.

**핵심 아이디어:** `@Configuration` 클래스는 단순한 빈 정의 모음이 아니라, 빈 간의 관계 설정과 스프링의 생명주기 관리를 위한 특별한 메커니즘을 제공한다.

---

**1. `@Configuration`의 기본 역할**

- 클래스 레벨 어노테이션입니다.
- "이 클래스는 스프링 빈 정의의 **소스(Source)** 입니다" 라고 컨테이너에게 알려줍니다.
- 내부에 `@Bean` 어노테이션이 붙은 메소드를 통해 실제 빈들을 정의하고 생성합니다.

---

**2. 빈 간 의존성 주입하기 (`@Bean` 메소드 간 호출)**

- **가장 큰 특징:** `@Configuration` 클래스 내에서는 한 `@Bean` 메소드가 **다른 `@Bean` 메소드를 직접 호출**하여 그 결과를 의존성으로 사용할 수 있습니다.
- **동작:** 스프링은 이 호출을 특별하게 처리하여, 호출된 `@Bean` 메소드(예: `beanTwo()`)가 **항상 컨테이너가 관리하는 빈 인스턴스(싱글톤이면 항상 동일한 인스턴스)** 를 반환하도록 보장합니다. 메소드 본문이 여러 번 실행되어 새 객체가 계속 만들어지는 것을 방지합니다.

    ```java
    @Configuration
    public class AppConfig {
        @Bean
        public BeanOne beanOne() {
            // beanTwo() 메소드를 직접 호출하여 의존성 주입
            // 스프링이 이 호출을 가로채서 항상 동일한 beanTwo 빈 인스턴스를 반환함
            return new BeanOne(beanTwo());
        }
    
        @Bean
        public BeanTwo beanTwo() {
            return new BeanTwo();
        }
    }
    ```

- **주의:** 이 "마법 같은" 동작은 해당 `@Bean` 메소드가 **`@Configuration` 클래스 내에 있을 때만** (정확히는 `proxyBeanMethods=true`일 때) 작동합니다. 일반 `@Component` 클래스 내의 `@Bean` 메소드 간 호출은 단순한 자바 메소드 호출로 취급되어 매번 새 객체를 만들 수 있습니다.

---

**3. 룩업 메소드 주입 (Java Config 방식)**

- **상황:** 싱글톤 빈(`CommandManager`)이 프로토타입 빈(`AsyncCommand`)을 필요로 할 때
- **Java Config 해결책:** `@Configuration` 클래스 내에서 룩업 메소드를 **익명 내부 클래스(Anonymous Inner Class)** 나 람다(Lambda) 등을 사용하여 **직접 구현**할 수 있습니다.

    ```java
    @Configuration
    public class AppConfig {
    
        // 1. 프로토타입 스코프의 Command 빈 정의
        @Bean
        @Scope("prototype")
        public AsyncCommand asyncCommand() {
            return new AsyncCommand();
        }
    
        // 2. CommandManager 빈 정의 (룩업 메소드 구현 포함)
        @Bean
        public CommandManager commandManager() {
            // CommandManager의 추상 메소드를 구현하는 익명 하위 클래스 반환
            return new CommandManager() {
                @Override
                protected Command createCommand() {
                    // ★ 다른 @Bean 메소드(asyncCommand)를 호출하여 프로토타입 빈 얻기 ★
                    // @Configuration 클래스이므로 이 호출은 매번 새로운 AsyncCommand 인스턴스를 반환함
                    return asyncCommand();
                }
            };
        }
    }
    ```

- **동작:** `commandManager()` 빈 내부에서 `createCommand()`가 호출될 때마다, `@Configuration` 클래스의 특별한 기능 덕분에 `asyncCommand()` `@Bean` 메소드가 호출되고, `@Scope("prototype")` 설정에 따라 **매번 새로운 `AsyncCommand` 인스턴스**가 생성되어 반환됩니다. XML 설정 없이도 룩업 메소드 주입을 깔끔하게 구현할 수 있습니다.

---

**4. 자바 기반 구성의 내부 작동 방식 (CGLIB 마법)**

이 부분은 `@Configuration` 클래스 내에서 `@Bean` 메소드 간 호출이 어떻게 싱글톤을 보장하는지에 대한 비밀을 설명합니다.

- **문제 인식:** `clientDao()` 메소드는 분명 `new ClientDaoImpl()`을 반환합니다. 그런데 `clientService1()`과 `clientService2()`에서 각각 `clientDao()`를 호출했는데 어떻게 같은 `ClientDao` 인스턴스를 사용할 수 있을까요?

    ```java
    @Configuration
    public class AppConfig {
        @Bean
        public ClientService clientService1() {
            return new ClientServiceImpl(clientDao()); // clientDao() 호출 1
        }
        @Bean
        public ClientService clientService2() {
            return new ClientServiceImpl(clientDao()); // clientDao() 호출 2 (새 객체 만들어질까?)
        }
        @Bean
        public ClientDao clientDao() {
            return new ClientDaoImpl(); // 호출될 때마다 new!
        }
    }
    ```

- **마법의 비밀 (CGLIB 프록시):**
  1. 스프링 컨테이너는 시작 시점에 `@Configuration` 어노테이션이 붙은 클래스(`AppConfig`)에 대해 **CGLIB 라이브러리를 사용하여 동적으로 자식 클래스(프록시)** 를 만듭니다.
  2. 실제로 컨테이너에 등록되는 것은 원본 `AppConfig` 객체가 아니라 이 **프록시 객체**입니다.
  3. 이 프록시 객체 내부의 `@Bean` 메소드(예: 프록시의 `clientDao()` 메소드)는 **오버라이드**되어 특별한 로직을 갖습니다.
  4. 프록시의 `@Bean` 메소드가 호출되면 (예: `clientService1` 내부에서 `clientDao()` 호출 시), 오버라이드된 로직은 **먼저 스프링 컨테이너 내부 저장소(캐시)** 를 확인하여 해당 이름("clientDao")의 빈이 **이미 만들어져 있는지 검사**합니다.
  5. **만약 이미 있다면 (싱글톤):** 원본 메소드(`new ClientDaoImpl()`)를 실행하지 않고, **캐시된 기존 빈 인스턴스**를 즉시 반환합니다.
  6. **만약 없다면:** 그때서야 원본 `@Bean` 메소드 로직(`new ClientDaoImpl()`)을 실행하여 **새 인스턴스를 만들고, 컨테이너 캐시에 저장한 후 반환**합니다.
- **결과:** `clientService1`이 `clientDao()`를 호출하면 처음이므로 새 `ClientDaoImpl` 객체가 생성되어 캐시되고 반환됩니다. `clientService2`가 다시 `clientDao()`를 호출하면, 이미 캐시에 있으므로 **새 객체를 만들지 않고 캐시된 동일한 객체**를 반환받습니다. 따라서 싱글톤 스코프가 보장됩니다.
- **CGLIB 제약사항:**
  - 이런 CGLIB 프록시 방식 때문에 `@Configuration` 클래스는 `final`로 선언될 수 없습니다. (상속 불가)
  - 하지만 생성자는 자유롭게 사용할 수 있습니다 (`@Autowired` 사용 가능).
- **프록시 피하기 ("Lite" 모드):** 만약 이 CGLIB 프록시 기능을 원치 않거나(약간의 성능 오버헤드 발생 가능) 제약사항을 피하고 싶다면,
  - `@Bean` 메소드를 일반 `@Component` 클래스 안에 정의하거나,
  - `@Configuration(proxyBeanMethods = false)` 어노테이션을 사용하여 프록시 생성을 명시적으로 끌 수 있습니다.
  - 이 경우("Lite" 모드), `@Bean` 메소드 간의 직접 호출은 일반 자바 호출처럼 동작하므로(매번 새 객체 생성 가능), 의존성은 반드시 메소드 파라미터를 통한 주입 방식으로만 처리해야 합니다.

**요약:**

`@Configuration` 클래스는 단순한 빈 정의 컨테이너가 아니라, **CGLIB 프록시**를 통해 내부적으로 강화되어 `@Bean` 메소드 간의 호출 시에도 싱글톤 스코프와 같은 컨테이너의 생명주기 관리를 보장하는 특별한 메커니즘을 가지고 있습니다. 이를 통해 자바 코드만으로도 XML 설정과 동일하게 빈 간의 의존성을 안전하고 선언적으로 정의할 수 있습니다. 룩업 메소드 주입과 같은 고급 패턴도 자연스럽게 구현 가능합니다. 프록시 기능을 사용하지 않는 "Lite" 모드도 존재합니다.
