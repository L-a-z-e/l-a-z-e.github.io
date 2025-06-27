---
title: Spring Core - AOP Practice
description: 
author: laze
date: 2025-06-27 00:00:01 +0900
categories: [Dev, SpringBoot]
tags: [SpringBoot]
---
### 성능 측정 Logger

### 간단한 서비스 구현

```java
package com.laze.aoppractice.service;

import org.springframework.stereotype.Service;

@Service
public class MySimpleService {
    /**
     * 간단한 메시지를 반환하는 메소드
     * @param message
     * @return
     */
    public String doSomething(String message) {
        System.out.println("MySimpleService.doSomething() called with message : " + message);
        return "Processed: " + message;
    }

    /**
     * 실행 시간이 오래 걸리는 작업을 시뮬레이션하는 메소드
     */
    public void doSomethingHeavy() {
        System.out.println("MySimpleService.doSomethingHeavy() started...");
        try {
            Thread.sleep(2000);
        } catch (InterruptedException e) {
            Thread.currentThread().interrupt();
        }
        System.out.println("MySimpleService.doSomethingHeavy() finished.");
    }
}
```

### Aspect 구현

```java
package com.laze.aoppractice.aop;

import org.aspectj.lang.ProceedingJoinPoint;
import org.aspectj.lang.annotation.Around;
import org.aspectj.lang.annotation.Aspect;
import org.aspectj.lang.annotation.Pointcut;
import org.springframework.stereotype.Component;

@Aspect
@Component
public class PerformanceLoggerAspect {

    @Pointcut("execution(public * com.laze.aoppractice.service..*.*(..))")
    public void serviceLayerMethods() {}

    @Around("serviceLayerMethods")
    public Object loggerPerformance(ProceedingJoinPoint proceedingJoinPoint) throws Throwable {

        long startTime = System.currentTimeMillis();

        try {
            Object result = proceedingJoinPoint.proceed();
            return result;
        } finally {
            long endTime = System.currentTimeMillis();
            long executionTime = endTime - startTime;

            String className = proceedingJoinPoint.getTarget().getClass().getSimpleName();
            String methodName = proceedingJoinPoint.getSignature().getName();

            System.out.println("[Performance Log] " + className + "." + methodName + " executed in " + executionTime + "ms");
        }
    }
}
```

**1. `@Aspect` 어노테이션과 스프링 AOP vs. AspectJ**

- `@Aspect` 어노테이션을 사용하는 것은 **AspectJ의 어노테이션 스타일을 "차용"**하여 **스프링 AOP를 구현**하는 것입니다. 순수한 AspectJ 컴파일러나 위빙(weaving)을 사용하는 것이 아닙니다.
- **상세 설명:**
  - **AspectJ:** AOP를 구현하는 **프레임워크**이자 **언어 확장**입니다. 바이트코드를 직접 조작하거나 컴파일 시점에 부가 기능 코드를 끼워넣는 등(위빙, Weaving) 매우 강력하고 정교한 기능을 제공합니다. 필드 접근, 객체 생성 등 메소드 실행 외의 다양한 지점(Join Point)에도 개입할 수 있습니다.
  - **스프링 AOP:** 순수한 AspectJ와는 다르게, **런타임(runtime)에 프록시(Proxy)를 생성하여 AOP를 구현**합니다. 즉, 스프링 컨테이너가 빈을 생성할 때, 대상 빈을 감싸는 프록시 객체를 만들고, 이 프록시 객체가 부가 기능(어드바이스)을 먼저 수행한 후 실제 대상 객체의 메소드를 호출하는 방식입니다.
  - **관계:** 스프링 AOP는 AspectJ의 모든 기능을 제공하지는 않습니다. 하지만 AOP를 더 쉽게 사용하기 위해 **AspectJ의 `@Aspect`, `@Pointcut`, `@Before`, `@Around`와 같은 어노테이션 문법과 포인트컷 표현식을 그대로 가져와서 사용**합니다. 따라서 개발자는 AspectJ의 편리한 문법으로 애스펙트를 정의하고, 실제 동작은 스프링 AOP의 프록시 기반 방식으로 이루어지는 것입니다.

> @Aspect 코드는 AspectJ의 문법이지만, 실행되는 방식은 스프링 AOP의 프록시 방식입니다.
>

---

**2. 포인트컷 `execution` 표현식 상세 설명**

`execution(public * com.example.aoppractice.service..*.*(..))`

- **`execution`**: 지시자(designator) 중 하나로, "메소드 실행" 조인 포인트를 의미합니다.
- **`public`**: 접근 제어자 패턴입니다. `public` 메소드를 대상으로 합니다. 생략하면 모든 접근 제어자(public, protected, default, private)를 의미합니다.
- **(첫 번째 )**: 리턴 타입 패턴입니다. 는 모든 리턴 타입을 의미합니다. `String`, `void`, `com.example.MyObject` 등으로 특정할 수도 있습니다.
- **`com.example.aoppractice.service..*`**: 클래스/타입 패턴입니다.
  - `com.example.aoppractice.service`: 패키지 이름입니다.
  - `..` (점 두 개): **해당 패키지 및 그 모든 하위 패키지**를 의미합니다. 즉, `com.example.aoppractice.service.impl`나 `com.example.aoppractice.service.util` 같은 패키지도 모두 포함됩니다.
  - (패키지 뒤의 ): 해당 패키지 내의 **모든 클래스(또는 인터페이스)**를 의미합니다. `My*` 와 같이 와일드카드를 사용하여 특정 패턴의 클래스만 지정할 수도 있습니다.
- **`.*` (두 번째  그룹)**: 메소드 이름 패턴입니다.
  - `.` (점 하나): 클래스와 메소드를 구분합니다.
  - (메소드 이름 ): **모든 메소드 이름**을 의미합니다. `do*` 와 같이 특정 패턴의 메소드만 지정할 수도 있습니다.
- **`(..)`**: 파라미터 패턴입니다.
  - `()`: 파라미터가 없는 경우.
  - `(*)`: 모든 타입의 파라미터가 하나인 경우.
  - `(String)`: 파라미터가 `String` 타입 하나인 경우.
  - `(..)`: **파라미터의 개수와 타입에 상관없이 모든 경우**를 의미합니다 (0개 이상).

> execution(public * com.example.aoppractice.service..*.*(..))는 "public 접근 제어자를 가지는, 모든 리턴 타입의, com.example.aoppractice.service 패키지 및 그 하위 패키지에 있는 모든 클래스의, 파라미터 개수와 타입에 상관없이 모든 메소드 실행" 이라는 매우 포괄적인 포인트컷입니다.
>
>
> **`execution( [수식어] 리턴타입 [클래스/타입 패턴]메소드이름패턴(파라미터패턴) [예외패턴] )`**
>
> 이 구조를 바탕으로, **메소드 이름 패턴 바로 뒤에 오는 소괄호 `()`가 메소드 선언의 시작**을 알리는 명확한 구분자 역할을 합니다.
>
> `com.example.service..*.*(..)` 표현식을 다시 한번 이 구조에 맞춰 분해해 보겠습니다.
>
> 1. **리턴타입 패턴:**
     >     - (이 예제에서는 생략되었지만, 보통 앞에 가 붙습니다. 예: `execution(* ...)` )
> 2. **클래스/타입 패턴 + 메소드이름 패턴:**
     >     - `com.example.service..*.*`
>     - 스프링 AOP 파서(parser)는 이 부분을 해석할 때, **가장 마지막에 있는 `.` (점)**을 클래스와 메소드 이름을 구분하는 경계로 인식합니다.
        >         - `com.example.service..*` -> 여기까지가 클래스/타입 패턴
>         - `.` -> 구분자
>         - -> 여기가 메소드 이름 패턴
> 3. **파라미터 패턴:**
     >     - **(..)**
>     - **바로 이 소괄호 `()`가 "이제부터는 메소드에 대한 설명입니다"라는 명확한 신호**가 됩니다. 파서는 이 소괄호를 만나면 "아, 바로 앞까지가 클래스와 메소드 이름 패턴이었고, 이제부터는 파라미터 패턴이구나"라고 인식합니다.
>
> **구별하는 순서(파서의 입장):**
>
> 1. `execution(` 으로 표현식이 시작된다.
> 2. 수식어(옵션), 리턴타입을 순서대로 파싱한다.
> 3. 그다음부터 나오는 문자열은 **소괄호 `()`가 나타나기 전까지** 모두 클래스/타입 및 메소드 이름 패턴으로 간주한다.
     >     - 이때, 마지막 `.`을 기준으로 앞부분은 클래스/타입 패턴으로, 뒷부분은 메소드 이름 패턴으로 나눈다.
> 4. 소괄호 `()`를 만나면, 그 안의 내용은 파라미터 패턴으로 해석한다.
> 5. 그 뒤에 `throws` 키워드가 나오면 예외 패턴으로 해석한다(옵션).
>
> **다른 예시로 확인해보기:**
>
> - `execution(* com.example.service.MyService.getGreeting(String))`
    >     - `(` 를 기준으로 앞부분: `com.example.service.MyService.getGreeting`
>     - 가장 마지막 `.`을 기준으로 나눔:
        >         - 클래스/타입 패턴: `com.example.service.MyService`
>         - 메소드 이름 패턴: `getGreeting`
>     - 파라미터 패턴: `(String)`
> - `execution(* *(..))`
    >     - `(` 를 기준으로 앞부분: `*`
>     - 가장 마지막 (여기서는 유일한)  이 메소드 이름 패턴.
>     - 그 앞의 은 타입 패턴... 이 아니라 리턴 타입 패턴이 되고, 클래스/타입 패턴은 생략된 것으로 간주되어 모든 타입을 대상으로 하게 됩니다.
>     - 파라미터 패턴: `(..)`
>     - 결과적으로 "모든 리턴 타입의, 모든 클래스에 있는, 모든 이름의, 모든 파라미터를 가진 메소드"가 됩니다.
>
> **결론적으로,** "어디서부터가 메소드인지"에 대한 답은 **파라미터를 감싸는 소괄호 `()`가 결정적인 기준점**이 됩니다. 소괄호 바로 앞까지가 클래스와 메소드 이름에 대한 패턴이고, 그중에서도 마지막 `.`이 둘을 가르는 경계선 역할을 합니다.
>

---

**3. `JoinPoint` vs. `ProceedingJoinPoint`**

- **`JoinPoint` (조인 포인트):**
  - **역할:** 어드바이스가 적용되는 조인 포인트(대상 메소드)에 대한 **정적인 정보**를 제공하는 인터페이스입니다.
  - **제공 정보:**
    - `getSignature()`: 실행되는 메소드의 시그니처(이름, 파라미터 타입 등) 정보.
    - `getTarget()`: 실제 대상 객체.
    - `getArgs()`: 메소드에 전달된 인자(파라미터) 배열.
  - **사용 어드바이스:** `@Before`, `@After`, `@AfterReturning`, `@AfterThrowing` 과 같이 **대상 메소드 실행 흐름에 직접 개입하지 않는 어드바이스**에서 사용됩니다. 이들은 대상 메소드 실행 전이나 후에 단순히 부가 기능을 추가할 뿐, 실행 자체를 제어하지는 못합니다.
- **`ProceedingJoinPoint` (진행 중인 조인 포인트):**
  - **역할:** `JoinPoint`의 하위 인터페이스로, `JoinPoint`가 제공하는 모든 기능에 더해, **대상 메소드의 실행을 직접 제어할 수 있는 강력한 기능**을 추가로 제공합니다.
  - **핵심 메소드:** **`proceed()`**
    - 이 메소드를 호출해야만 실제 대상 메소드가 실행됩니다.
    - `proceed()`를 여러 번 호출할 수도 있고 (권장되지 않음), 아예 호출하지 않아서 대상 메소드 실행을 막을 수도 있습니다.
    - `proceed(Object[] args)`를 사용하여 대상 메소드에 전달될 인자를 변경할 수도 있습니다.
  - **사용 어드바이스:** **오직 `@Around` 어드바이스에서만** 사용할 수 있습니다. `@Around` 어드바이스는 대상 메소드를 감싸서 실행 전/후에 모두 개입하고, 실행 자체를 제어해야 하기 때문입니다.

> 이번 예제에서 ProceedingJoinPoint를 사용한 이유: 메소드 실행 시간을 측정하려면 **"메소드 실행 직전"**에 시작 시간을 기록하고, **"메소드 실행 직후"**에 종료 시간을 기록해야 합니다. 이처럼 대상 메소드 실행의 전과 후에 모두 개입해야 하므로, @Around 어드바이스가 필요했고, @Around 어드바이스에서는 ProceedingJoinPoint를 사용해야만 대상 메소드를 실행시킬 수 있습니다.
>

---

**4. `throws Throwable`이 필요한 이유**

- `ProceedingJoinPoint.proceed()` 메소드의 시그니처는 `public Object proceed() throws Throwable;` 입니다.
- 이는 `proceed()` 메소드를 통해 실행되는 **실제 대상 메소드에서 어떤 종류의 예외(`Exception` 이나 `Error`를 포함하는 최상위 `Throwable`)라도 발생할 수 있음**을 의미합니다.
- 자바의 예외 처리 규칙에 따라, 호출하는 메소드(`logPerformance`)는 `proceed()`가 던질 수 있는 `Throwable`을 처리(catch)하거나, 자신을 호출한 쪽으로 다시 던져야(throws) 합니다.
- `@Around` 어드바이스는 보통 대상 메소드에서 발생한 예외를 그대로 전파하여, 다른 예외 처리 메커니즘(`@ExceptionHandler` 등)이 처리할 수 있도록 하는 것이 일반적입니다. 따라서 `throws Throwable`을 메소드 시그니처에 추가하여 예외를 다시 던지도록 선언하는 것입니다.

---

**5. `pjp.proceed()`의 반환 값 (`Object`)**

- `pjp.proceed()`를 호출하면 실제 대상 메소드가 실행됩니다.
- 이 메소드는 대상 메소드가 반환하는 값을 **`Object` 타입으로 반환**합니다.
- **예시:**
  - 만약 대상 메소드가 `String`을 반환하는 `MySimpleService.doSomething("hello")` 였다면, `pjp.proceed()`는 `"Processed: hello"` 라는 `String` 객체를 `Object` 타입으로 반환합니다.
  - 만약 대상 메소드가 `void`를 반환하는 `MySimpleService.doSomethingHeavy()` 였다면, `pjp.proceed()`는 `null`을 반환합니다.
- `@Around` 어드바이스에서는 이 반환 값을 가로채서 다른 값으로 바꾸거나, 로깅하는 등의 추가적인 작업을 할 수 있습니다. 우리 예제에서는 받은 `result`를 그대로 다시 반환(`return result;`)하여 대상 메소드의 원래 반환 값이 호출자에게 전달되도록 했습니다.

---

**6. `pjp` 객체의 유용한 메소드들 (`getTarget`, `getSignature` 등)**

`ProceedingJoinPoint` (또는 `JoinPoint`) 객체는 현재 실행되는 조인 포인트에 대한 풍부한 메타데이터를 제공합니다.

- **`pjp.getTarget()`:**
  - **반환:** **AOP 프록시가 감싸고 있는 실제 대상 객체**를 반환합니다.
  - **예시:** `MySimpleService`의 프록시 객체가 AOP를 수행하고 있다면, `pjp.getTarget()`는 원본 `MySimpleServiceImpl` 인스턴스를 반환합니다.
  - **활용:** 대상 객체의 클래스 정보를 얻거나(`pjp.getTarget().getClass()`), 대상 객체의 상태에 접근할 때 사용할 수 있습니다.
- **`pjp.getSignature()`:**
  - **반환:** **실행되는 메소드의 시그니처 정보**를 담고 있는 `Signature` 객체를 반환합니다.
  - `Signature` 인터페이스는 `MethodSignature` (메소드 시그니처)나 `ConstructorSignature` (생성자 시그니처) 등의 하위 인터페이스를 가집니다.
  - **주요 활용:**
    - `getSignature().getName()`: 메소드의 **이름**을 문자열로 반환합니다. (예: "doSomethingHeavy")
    - `getSignature().toShortString()`: 메소드 시그니처를 축약된 문자열로 반환합니다. (예: "MySimpleService.doSomethingHeavy()")
    - `getSignature().toLongString()`: `public ...` 등 수식어까지 포함된 완전한 시그니처를 문자열로 반환합니다.
    - `((MethodSignature) pjp.getSignature()).getMethod()`: 실제 `java.lang.reflect.Method` 객체를 가져올 수 있어, 어노테이션 정보 등에 접근할 수 있습니다.
- **`pjp.getArgs()`:**
  - **반환:** 대상 메소드에 전달된 **인자들을 `Object` 배열(`Object[]`)로 반환**합니다.
  - **활용:** 메소드에 어떤 값들이 전달되었는지 로그로 남기거나, 인자 값을 검사하는 등의 용도로 사용할 수 있습니다.

> 정리: 우리 예제의 System.out.println("[Performance Log] " + className + "." + methodName + ...) 부분에서 className은 pjp.getTarget().getClass().getSimpleName()을 통해 얻었고, methodName은 pjp.getSignature().getName()을 통해 얻었습니다. 이처럼 ProceedingJoinPoint 객체를 활용하면 어떤 메소드가 실행되는지 동적으로 파악하고 로그를 남길 수 있습니다.
>

---

```java
package com.laze.aoppractice;

import com.laze.aoppractice.service.MySimpleService;
import org.springframework.boot.CommandLineRunner;
import org.springframework.boot.SpringApplication;
import org.springframework.boot.autoconfigure.SpringBootApplication;
import org.springframework.context.annotation.Bean;

@SpringBootApplication
public class AopPracticeApplication {

    public static void main(String[] args) {
        SpringApplication.run(AopPracticeApplication.class, args);
    }

    @Bean
    public CommandLineRunner run(MySimpleService service) {
        return args -> {
            System.out.println("\n==== Executing Service Methods via CommandLineRunner ====");
            service.doSomething("Hello AOP!");
            System.out.println("--------------------------------");
            service.doSomethingHeavy();
            System.out.println("==== Finished Executing Service Methods ====\n");
        };
    }
}
```

CommandLineRunner 를 통해 테스트

```
==== Executing Service Methods via CommandLineRunner ====
MySimpleService.doSomething() called with message : Hello AOP!
[Performance Log] MySimpleService.doSomething executed in 0ms
--------------------------------
MySimpleService.doSomethingHeavy() started...
MySimpleService.doSomethingHeavy() finished.
[Performance Log] MySimpleService.doSomethingHeavy executed in 2005ms
==== Finished Executing Service Methods ====
```
