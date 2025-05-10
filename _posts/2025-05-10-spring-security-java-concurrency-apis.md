---
title: Integrations - Java's Concurrency APIs
description: 
author: laze
date: 2025-05-10 00:00:03 +0900
categories: [Dev, SpringSecurity]
tags: [SpringSecurity]
---
## 동시성 지원 (Concurrency Support)

대부분의 환경에서 보안 정보(Security)는 스레드별(per Thread)로 저장됩니다.

이는 새로운 스레드에서 작업이 수행될 때 `SecurityContext`가 손실된다는 것을 의미합니다.

Spring Security는 사용자가 이를 훨씬 쉽게 처리할 수 있도록 몇 가지 인프라를 제공합니다.

Spring Security는 멀티스레드 환경에서 Spring Security와 함께 작업하기 위한 저수준 추상화를 제공합니다.

실제로 이것은 Spring Security가 `AsyncContext.start(Runnable)` 및 Spring MVC 비동기 통합과의 통합을 구축하는 기반입니다.

### DelegatingSecurityContextRunnable

Spring Security의 동시성 지원 내에서 가장 기본적인 구성 요소 중 하나는 `DelegatingSecurityContextRunnable`입니다.

이는 위임할 `Runnable`을 위해 지정된 `SecurityContext`로 `SecurityContextHolder`를 초기화하기 위해 위임 `Runnable`을 감쌉니다.

그런 다음 위임 `Runnable`을 호출하고 나중에 `SecurityContextHolder`를 지우도록 보장합니다. `DelegatingSecurityContextRunnable`은 다음과 같습니다:

**Java**

```java
public void run() {
    try {
        SecurityContextHolder.setContext(securityContext);
        delegate.run();
    } finally {
        SecurityContextHolder.clearContext();
    }
}
```

**Kotlin**

```kotlin
override fun run() {
    try {
        SecurityContextHolder.setContext(securityContext)
        delegate.run()
    } finally {
        SecurityContextHolder.clearContext()
    }
}
```

매우 간단하지만, 한 스레드에서 다른 스레드로 `SecurityContext`를 원활하게 전송할 수 있게 해줍니다.

이는 대부분의 경우 `SecurityContextHolder`가 스레드별로 작동하기 때문에 중요합니다.

예를 들어, 서비스 중 하나를 보호하기 위해 Spring Security의 `<global-method-security>` 지원을 사용했을 수 있습니다.

이제 현재 스레드의 `SecurityContext`를 보호된 서비스를 호출하는 스레드로 쉽게 전송할 수 있습니다. 이를 수행하는 방법의 예는 다음과 같습니다:

**Java**

```java
Runnable originalRunnable = new Runnable() {
    public void run() {
        // invoke secured service
    }
};

SecurityContext context = SecurityContextHolder.getContext();
DelegatingSecurityContextRunnable wrappedRunnable =
    new DelegatingSecurityContextRunnable(originalRunnable, context);

new Thread(wrappedRunnable).start();
```

**Kotlin**

```kotlin
val originalRunnable = Runnable {
    // invoke secured service
}

val context = SecurityContextHolder.getContext()
val wrappedRunnable =
    DelegatingSecurityContextRunnable(originalRunnable, context)

Thread(wrappedRunnable).start()
```

위 코드는 다음 단계를 수행합니다:

1. 보호된 서비스를 호출할 `Runnable`을 만듭니다. Spring Security를 인식하지 못한다는 점에 유의하세요.
2. `SecurityContextHolder`에서 사용하려는 `SecurityContext`를 가져오고 `DelegatingSecurityContextRunnable`을 초기화합니다.
3. `DelegatingSecurityContextRunnable`을 사용하여 `Thread`를 만듭니다.
4. 생성한 `Thread`를 시작합니다.

`SecurityContextHolder`에서 `SecurityContext`를 사용하여 `DelegatingSecurityContextRunnable`을 만드는 것이 매우 일반적이므로 이에 대한 바로 가기 생성자가 있습니다.

다음 코드는 위 코드와 동일합니다:

**Java**

```java
Runnable originalRunnable = new Runnable() {
    public void run() {
        // invoke secured service
    }
};

DelegatingSecurityContextRunnable wrappedRunnable =
    new DelegatingSecurityContextRunnable(originalRunnable);

new Thread(wrappedRunnable).start();
```

**Kotlin**

```kotlin
val originalRunnable = Runnable {
    // invoke secured service
}

val wrappedRunnable =
    DelegatingSecurityContextRunnable(originalRunnable)

Thread(wrappedRunnable).start()
```

우리가 가진 코드는 사용하기 간단하지만, 사용하기 위해서는 Spring Security를 사용하고 있다는 지식이 여전히 필요합니다.

다음 섹션에서는 `DelegatingSecurityContextExecutor`를 활용하여 Spring Security를 사용하고 있다는 사실을 숨기는 방법을 살펴보겠습니다.

### DelegatingSecurityContextExecutor

이전 섹션에서는 `DelegatingSecurityContextRunnable`을 사용하는 것이 쉽다는 것을 알았지만, 사용하기 위해 Spring Security를 인식해야 했기 때문에 이상적이지는 않았습니다.

`DelegatingSecurityContextExecutor`가 Spring Security를 사용하고 있다는 지식으로부터 우리 코드를 어떻게 보호할 수 있는지 살펴보겠습니다.

`DelegatingSecurityContextExecutor`의 디자인은 위임 `Runnable` 대신 위임 `Executor`를 허용한다는 점을 제외하고는 `DelegatingSecurityContextRunnable`과 매우 유사합니다.

사용 방법의 예는 다음과 같습니다:

**Java**

```java
SecurityContext context = SecurityContextHolder.createEmptyContext();
Authentication authentication =
    UsernamePasswordAuthenticationToken.authenticated("user","doesnotmatter", AuthorityUtils.createAuthorityList("ROLE_USER"));
context.setAuthentication(authentication);

SimpleAsyncTaskExecutor delegateExecutor =
    new SimpleAsyncTaskExecutor();
DelegatingSecurityContextExecutor executor =
    new DelegatingSecurityContextExecutor(delegateExecutor, context);

Runnable originalRunnable = new Runnable() {
    public void run() {
        // invoke secured service
    }
};

executor.execute(originalRunnable);
```

**Kotlin**

```kotlin
val context = SecurityContextHolder.createEmptyContext()
val authentication =
    UsernamePasswordAuthenticationToken.authenticated("user","doesnotmatter", AuthorityUtils.createAuthorityList("ROLE_USER"))
context.setAuthentication(authentication)

val delegateExecutor =
    SimpleAsyncTaskExecutor()
val executor =
    DelegatingSecurityContextExecutor(delegateExecutor, context)

val originalRunnable = Runnable {
    // invoke secured service
}

executor.execute(originalRunnable)
```

코드는 다음 단계를 수행합니다:

1. `DelegatingSecurityContextExecutor`에 사용할 `SecurityContext`를 만듭니다. 이 예에서는 단순히 수동으로 `SecurityContext`를 만듭니다. 그러나 `SecurityContext`를 어디서 또는 어떻게 얻는지는 중요하지 않습니다 (즉, 원한다면 `SecurityContextHolder`에서 얻을 수 있음).
2. 제출된 `Runnable`을 실행하는 역할을 하는 `delegateExecutor`를 만듭니다.
3. 마지막으로 `execute` 메소드에 전달된 모든 `Runnable`을 `DelegatingSecurityContextRunnable`으로 래핑하는 역할을 하는 `DelegatingSecurityContextExecutor`를 만듭니다. 그런 다음 래핑된 `Runnable`을 `delegateExecutor`에 전달합니다. 이 경우 `DelegatingSecurityContextExecutor`에 제출된 모든 `Runnable`에 대해 동일한 `SecurityContext`가 사용됩니다. 이는 상승된 권한을 가진 사용자가 실행해야 하는 백그라운드 작업을 실행하는 경우에 유용합니다.

이 시점에서 "이것이 어떻게 내 코드에서 Spring Security에 대한 모든 지식을 보호합니까?"라고 자문할 수 있습니다. 우리 코드에서 `SecurityContext`와 `DelegatingSecurityContextExecutor`를 만드는 대신 이미 초기화된 `DelegatingSecurityContextExecutor` 인스턴스를 주입할 수 있습니다.

**Java**

```java
@Autowired
private Executor executor; // DelegatingSecurityContextExecutor의 인스턴스가 됩니다.

public void submitRunnable() {
    Runnable originalRunnable = new Runnable() {
        public void run() {
        // invoke secured service
        }
    };
    executor.execute(originalRunnable);
}
```

**Kotlin**

```kotlin
@Autowired
private lateinit var executor: Executor // DelegatingSecurityContextExecutor의 인스턴스가 됩니다.

fun submitRunnable() {
    val originalRunnable = Runnable {
        // invoke secured service
    }
    executor.execute(originalRunnable)
}
```

이제 우리 코드는 `SecurityContext`가 스레드로 전파되고, `originalRunnable`이 실행된 다음 `SecurityContextHolder`가 비워진다는 것을 인식하지 못합니다.

이 예에서는 각 스레드를 실행하는 데 동일한 사용자가 사용됩니다.

`executor.execute(Runnable)`을 호출한 시점의 `SecurityContextHolder`의 사용자(즉, 현재 로그인한 사용자)를 사용하여 `originalRunnable`을 처리하려면 어떻게 해야 할까요?

이는 `DelegatingSecurityContextExecutor` 생성자에서 `SecurityContext` 인수를 제거하여 수행할 수 있습니다. 예를 들어:

**Java**

```java
SimpleAsyncTaskExecutor delegateExecutor = new SimpleAsyncTaskExecutor();
DelegatingSecurityContextExecutor executor =
    new DelegatingSecurityContextExecutor(delegateExecutor);

```

**Kotlin**

```kotlin
val delegateExecutor = SimpleAsyncTaskExecutor()
val executor =
    DelegatingSecurityContextExecutor(delegateExecutor)

```

이제 `executor.execute(Runnable)`이 실행될 때마다 `SecurityContext`가 먼저 `SecurityContextHolder`에 의해 얻어지고, 그런 다음 해당 `SecurityContext`가 `DelegatingSecurityContextRunnable`을 만드는 데 사용됩니다.

즉, `executor.execute(Runnable)` 코드를 호출하는 데 사용된 동일한 사용자로 `Runnable`을 실행하고 있다는 의미입니다.

### Spring Security 동시성 클래스

Java 동시성 API 및 Spring Task 추상화 모두와의 추가 통합에 대한 자세한 내용은 Javadoc을 참조하세요. 이전 코드를 이해하면 매우 자명합니다.

- `DelegatingSecurityContextCallable`
- `DelegatingSecurityContextExecutor`
- `DelegatingSecurityContextExecutorService`
- `DelegatingSecurityContextRunnable`
- `DelegatingSecurityContextScheduledExecutorService`
- `DelegatingSecurityContextSchedulingTaskExecutor`
- `DelegatingSecurityContextAsyncTaskExecutor`
- `DelegatingSecurityContextTaskExecutor`
- `DelegatingSecurityContextTaskScheduler`

### 🕵️‍♂️ 일꾼(스레드)에게도 신분증을! (Spring Security 동시성 지원 완벽 이해)

우리가 웹사이트에서 어떤 요청을 처리할 때, 가끔은 시간이 오래 걸리는 작업을 "뒤에서 몰래"(백그라운드에서) 처리하거나, 여러 작업을 "동시에"(병렬적으로) 처리하고 싶을 때가 있어요. 이때 "새로운 일꾼(스레드)"을 만들어서 일을 시키게 되죠.

**문제 발생! 🚨**

Spring Security는 보통 **"각 일꾼(스레드)마다 자기만의 작업 공간(ThreadLocal)에 신분증(SecurityContext)을 보관"** 해요.
그런데 새로운 일꾼(스레드)을 만들면, 이 새로운 일꾼은 **원래 일꾼(메인 스레드)의 신분증을 자동으로 물려받지 못해요!** 즉, 새로운 일꾼은 "나는 누구? 여긴 어디?" 상태가 되어버려서, Spring Security가 보호하는 중요한 작업(예: 특정 권한이 필요한 서비스 호출)을 수행할 수 없게 됩니다.

**Spring Security의 해결책! 든든한 도우미들!**

Spring Security는 이런 문제를 해결하기 위해 몇 가지 똑똑한 도우미 클래스들을 제공해요. 이 도우미들은 원래 일꾼의 신분증을 새로운 일꾼에게 안전하게 복사해주고, 일이 끝나면 깨끗하게 정리까지 해줍니다.

### 1. 가장 기본적인 도우미: `DelegatingSecurityContextRunnable` 🏃‍♂️

`Runnable`은 Java에서 "어떤 작업을 수행하는 코드 덩어리"를 나타내는 인터페이스예요. 새로운 스레드에게 "이거 해!" 하고 시킬 때 많이 사용하죠.

`DelegatingSecurityContextRunnable`은 이 `Runnable`을 한번 감싸주는 역할을 해요.

- **어떻게 동작할까요?**
  1. 원래 일꾼(메인 스레드)의 신분증(`SecurityContext`)을 가져옵니다.
  2. `DelegatingSecurityContextRunnable`을 만들 때, 이 신분증과 우리가 실행하고 싶은 실제 작업(`originalRunnable`)을 함께 넣어줍니다.
  3. 이 `DelegatingSecurityContextRunnable`을 새로운 일꾼(스레드)에게 전달해서 실행시킵니다.
  4. 새로운 일꾼이 작업을 **시작하기 직전에**, `DelegatingSecurityContextRunnable`이 원래 일꾼의 신분증을 **새로운 일꾼의 작업 공간(`SecurityContextHolder`)에 설정**해줍니다. (이제 새로운 일꾼도 "내가 누군지 알겠어!")
  5. 새로운 일꾼이 실제 작업(`originalRunnable`)을 수행합니다. (이제 보안이 적용된 서비스도 호출 가능!)
  6. 작업이 **끝나면 (성공하든 실패하든)**, `DelegatingSecurityContextRunnable`이 새로운 일꾼의 작업 공간에서 신분증을 **깨끗하게 지워줍니다.** (다른 작업에 영향을 주지 않도록!)
- **예시 코드 (간단 버전):**

    ```java
    // 1. 실제 수행할 작업 (Spring Security를 전혀 모름)
    Runnable originalRunnable = new Runnable() {
        public void run() {
            // 여기서 중요한 보안 서비스 호출!
            System.out.println("새로운 일꾼: 현재 사용자? " + SecurityContextHolder.getContext().getAuthentication().getName());
            // mySecuredService.doSomething();
        }
    };
    
    // 2. 현재 스레드의 SecurityContext를 사용하여 DelegatingSecurityContextRunnable 생성
    // (생성자에 SecurityContext를 직접 안 넘겨주면, 알아서 현재 스레드 것을 가져옴)
    DelegatingSecurityContextRunnable wrappedRunnable =
        new DelegatingSecurityContextRunnable(originalRunnable);
    
    // 3. 새로운 스레드 만들어서 실행
    new Thread(wrappedRunnable).start();
    ```

  위 코드에서 `new Thread(wrappedRunnable).start()`가 실행되면, `originalRunnable` 내부에서도 현재 로그인한 사용자 정보를 정상적으로 가져올 수 있게 됩니다!

- **장점:** 간단하게 현재 보안 컨텍스트를 다른 스레드로 전달할 수 있어요.
- **단점:** 코드를 작성할 때 "아, Spring Security 쓰고 있으니까 `DelegatingSecurityContextRunnable`으로 감싸야지!" 하고 직접 신경 써야 해요.

### 2. 더 세련된 도우미: `DelegatingSecurityContextExecutor` 👨‍💼

`Executor`는 Java에서 작업을 비동기적으로 실행하는 좀 더 일반적이고 고급 방식이에요. (스레드를 직접 만드는 것보다 유연하고 관리가 편하죠.)

`DelegatingSecurityContextExecutor`는 이 `Executor`를 한번 감싸주는 역할을 해요.

- **어떻게 동작할까요?**
  1. 우리가 사용할 실제 `Executor` (예: `SimpleAsyncTaskExecutor`)를 만듭니다.
  2. `DelegatingSecurityContextExecutor`를 만들 때, 이 실제 `Executor`와 (선택적으로) 특정 `SecurityContext`를 함께 넣어줍니다.
  3. 이제부터 이 `DelegatingSecurityContextExecutor`에게 작업을 시키면(`execute(Runnable)`),
    - `DelegatingSecurityContextExecutor`는 전달받은 `Runnable`을 **자동으로 `DelegatingSecurityContextRunnable`으로 감싸줍니다.** (이때 어떤 `SecurityContext`를 사용할지 결정)
    - 그리고 이 감싸진 `Runnable`을 실제 `Executor`에게 전달해서 실행시킵니다.
- **두 가지 사용 시나리오:**
  1. **특정 사용자로 모든 작업 실행:**

      ```java
      // 1. 특정 사용자의 SecurityContext 만들기 (예: 관리자 권한)
      SecurityContext adminContext = SecurityContextHolder.createEmptyContext();
      Authentication adminAuth = UsernamePasswordAuthenticationToken.authenticated("admin", "N/A", AuthorityUtils.createAuthorityList("ROLE_ADMIN"));
      adminContext.setAuthentication(adminAuth);
      
      Executor actualExecutor = new SimpleAsyncTaskExecutor();
      // 2. adminContext를 사용하는 DelegatingSecurityContextExecutor 생성
      Executor securityAwareExecutor = new DelegatingSecurityContextExecutor(actualExecutor, adminContext);
      
      Runnable task = () -> { /* 관리자 권한으로 실행될 작업 */ };
      securityAwareExecutor.execute(task); // 이 task는 항상 adminContext로 실행됨
      ```

    - 이 경우, `securityAwareExecutor`에 제출되는 모든 작업은 `adminContext`를 사용해서 실행됩니다. 백그라운드에서 관리자 권한으로 뭔가를 처리해야 할 때 유용하겠죠?
  2. **작업을 요청한 사용자로 실행 (더 일반적):**

      ```java
      Executor actualExecutor = new SimpleAsyncTaskExecutor();
      // 2. SecurityContext를 넘기지 않고 DelegatingSecurityContextExecutor 생성
      Executor securityAwareExecutor = new DelegatingSecurityContextExecutor(actualExecutor);
      
      // 이 코드가 실행될 때의 현재 로그인 사용자 (예: 'user1')
      Runnable task = () -> { /* user1의 권한으로 실행될 작업 */ };
      securityAwareExecutor.execute(task); // 이 task는 'user1'의 SecurityContext로 실행됨
      
      // 만약 다른 요청에서 'user2'가 로그인한 상태로 이 코드를 실행하면,
      // 그 task는 'user2'의 SecurityContext로 실행됨
      ```

    - 생성자에 `SecurityContext`를 안 넘겨주면, `execute()` 메소드가 호출될 때의 **현재 스레드의 `SecurityContext`를 가져와서 사용**합니다. 이게 더 일반적이고 우리가 원하는 동작일 경우가 많아요. (요청한 사람의 권한으로 비동기 작업 수행)
- **진짜 장점! Spring의 DI(의존성 주입)와 함께 사용하면 마법이! ✨**

    ```java
    @Service
    public class MyAsyncService {
    
        @Autowired
        private Executor taskExecutor; // 여기에 DelegatingSecurityContextExecutor가 주입되도록 설정!
    
        public void doSomethingAsync() {
            User currentUser = (User) SecurityContextHolder.getContext().getAuthentication().getPrincipal();
            System.out.println(currentUser.getUsername() + "님이 비동기 작업을 요청하셨습니다.");
    
            Runnable originalTask = () -> {
                // 이 안에서도 currentUser의 정보나 권한을 사용할 수 있어야 함!
                Authentication authInAsyncTask = SecurityContextHolder.getContext().getAuthentication();
                if (authInAsyncTask != null) {
                    System.out.println("비동기 작업 중인 사용자: " + authInAsyncTask.getName());
                    // mySecuredService.processData();
                } else {
                    System.out.println("비동기 작업: 어라, 사용자가 없네?");
                }
            };
            taskExecutor.execute(originalTask); // 그냥 Executor 쓰듯이 쓰면 알아서 보안 컨텍스트가 넘어감!
        }
    }
    ```

  - `taskExecutor`를 `DelegatingSecurityContextExecutor`로 빈(Bean) 등록해두고 `@Autowired`로 주입받으면, `MyAsyncService` 코드는 Spring Security를 전혀 의식하지 않고 평소처럼 `Executor`를 사용할 수 있어요! `DelegatingSecurityContextExecutor`가 뒤에서 조용히 모든 보안 처리를 다 해주니까요.

### 3. 더 많은 도우미 친구들 🤝

`DelegatingSecurityContextRunnable`이나 `DelegatingSecurityContextExecutor`와 비슷한 원리로 동작하는 다른 도우미 클래스들도 많이 있어요. Java의 다양한 동시성 API나 Spring의 Task 관련 추상화 클래스들을 Spring Security와 함께 사용할 수 있도록 도와줍니다.

- `DelegatingSecurityContextCallable` (`Callable`을 위한 것)
- `DelegatingSecurityContextExecutorService` (`ExecutorService`를 위한 것)
- `DelegatingSecurityContextScheduledExecutorService` (스케줄링 작업을 위한 것)
- 등등...

---

**오늘의 핵심 요약:**

1. **새로운 스레드에서는 기본적으로 원래 스레드의 `SecurityContext` (로그인 정보 등)가 유실된다!**
2. **`DelegatingSecurityContextRunnable`**: `Runnable`을 감싸서 특정 `SecurityContext`를 새 스레드에 설정하고, 작업 후 정리해준다. (직접 사용)
3. **`DelegatingSecurityContextExecutor`**: `Executor`를 감싸서, 제출되는 `Runnable`들을 자동으로 `DelegatingSecurityContextRunnable`으로 만들어 실행한다. (DI와 함께 사용하면 매우 편리!)
  - 생성 시 특정 `SecurityContext`를 고정할 수도 있고, `execute()` 호출 시점의 현재 `SecurityContext`를 사용하게 할 수도 있다.
4. **핵심은 `SecurityContextHolder.setContext()`와 `SecurityContextHolder.clearContext()`를 올바른 시점에 호출해주는 것!** 이 도우미 클래스들이 그걸 대신 해준다.
