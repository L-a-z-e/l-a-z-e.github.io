---
title: Spring Core - Review(5)
description: 
author: laze
date: 2025-06-26 00:00:01 +0900
categories: [Dev, SpringBoot]
tags: [SpringBoot]
---
### **스프링 AOP - `@AspectJ` 지원 및 어드바이스 종류**

**1. `@AspectJ` 스타일 AOP 활성화 어노테이션**

- **`@EnableAspectJAutoProxy`**
- **피드백 및 추가 설명:**
  - 자바 기반 설정(`@Configuration` 클래스)에서 `@AspectJ` 스타일의 AOP를 활성화하려면 `@EnableAspectJAutoProxy` 어노테이션을 설정 클래스에 추가해야 합니다.
  - 이 어노테이션은 스프링 컨테이너에게 `@Aspect` 어노테이션이 붙은 빈들을 찾아서, 해당 애스펙트가 정의한 포인트컷에 매칭되는 다른 빈들에 대해 프록시(proxy)를 생성하고 어드바이스를 적용하도록 지시합니다.
  - **코드 예시 (Java Config):**

      ```java
      import org.springframework.context.annotation.ComponentScan;
      import org.springframework.context.annotation.Configuration;
      import org.springframework.context.annotation.EnableAspectJAutoProxy;
      
      @Configuration
      @ComponentScan(basePackages = "com.example") // @Aspect 빈 등을 스캔하기 위해
      @EnableAspectJAutoProxy // AspectJ 기반 AOP 자동 프록시 생성 활성화
      public class AppConfig {
          // ... 다른 빈 정의 ...
      }
      ```

  - 만약 XML 기반 설정을 사용한다면 `<aop:aspectj-autoproxy/>` 태그를 사용합니다.
  - 스프링 부트(Spring Boot)를 사용하는 경우, `spring-boot-starter-aop` 의존성이 추가되어 있으면 AOP가 자동 구성되므로 `@EnableAspectJAutoProxy`를 명시적으로 추가하지 않아도 되는 경우가 많습니다. (부트의 자동 설정 마법!)

**2. 애스펙트(Aspect) 선언 어노테이션 및 빈 등록 여부**

- **`@Aspect`:** 해당 클래스가 하나 이상의 애스펙트(어드바이스 + 포인트컷)를 정의하는 클래스임을 나타냅니다.
- **스프링 빈으로 등록 필요:** `@Aspect` 어노테이션 자체는 클래스를 스프링 빈으로 만들지 않습니다. 따라서 애스펙트 클래스가 스프링 컨테이너에 의해 감지되고 AOP 로직을 적용하려면, 반드시 **스프링 빈으로 등록되어야 합니다.** 이를 위해 `@Component` (또는 `@Service`, `@Repository` 등) 어노테이션을 함께 사용하거나, 자바 설정 클래스의 `@Bean` 메소드를 통해 직접 빈으로 등록해야 합니다.
- **코드 예시:**

    ```java
    import org.aspectj.lang.annotation.Aspect;
    import org.springframework.stereotype.Component;
    
    @Aspect    // 이 클래스는 애스펙트입니다.
    @Component // 이 애스펙트를 스프링 빈으로 등록합니다.
    public class LoggingAspect {
        // ... 포인트컷과 어드바이스 정의 ...
    }
    ```


**3. 포인트컷(Pointcut) 정의 어노테이션 및 표현식 형식**

- **표현식 형식:** `@Pointcut` 어노테이션의 값으로는 **AspectJ 포인트컷 표현식(AspectJ Pointcut Expression Language)**을 사용합니다. 이 표현식은 어드바이스를 적용할 조인 포인트를 매우 정교하게 지정할 수 있는 강력한 문법을 제공합니다.
- **주요 표현식 지시자 (Designators):**
  - **`execution`:** 가장 일반적으로 사용되며, **메소드 실행 조인 포인트**를 매칭합니다. (스프링 AOP의 핵심)
    - 형식: `execution(수식어패턴? 리턴타입패턴 선언타입패턴?메소드이름패턴(파라미터패턴) 예외패턴?)`
    - 예: `execution(public * com.example.service.*.*(..))`
      - `public`: public 접근 제어자 (생략 가능, 생략 시 모든 접근 제어자)
      - : 모든 리턴 타입
      - `com.example.service.*`: `com.example.service` 패키지의 모든 클래스
      - `.*`: 모든 메소드 이름
      - `(..)`: 0개 이상의 모든 파라미터
  - `within`: 특정 타입 내의 모든 조인 포인트를 매칭합니다.
    - 예: `within(com.example.service.*)`
  - `this`: 지정된 타입의 프록시 객체를 대상으로 하는 조인 포인트를 매칭합니다.
  - `target`: 지정된 타입의 실제 대상 객체를 대상으로 하는 조인 포인트를 매칭합니다.
  - `args`: 특정 타입의 인자를 받는 조인 포인트를 매칭합니다.
  - `@target`: 특정 어노테이션이 붙은 타입의 객체를 대상으로 하는 조인 포인트를 매칭합니다.
  - `@within`: 특정 어노테이션이 붙은 타입 내의 조인 포인트를 매칭합니다.
  - `@annotation`: 특정 어노테이션이 붙은 메소드를 매칭합니다.
  - `@args`: 특정 어노테이션이 붙은 타입의 인자를 받는 조인 포인트를 매칭합니다.
  - `bean`: 특정 이름을 가진 스프링 빈의 조인 포인트를 매칭합니다.
    - 예: `bean(myUserService)`
- **코드 예시:**

    ```java
    @Aspect
    @Component
    public class MyPointcutAspect {
    
        // com.example.service 패키지 내의 모든 클래스의 모든 public 메소드 실행을 포인트컷으로 정의
        @Pointcut("execution(public * com.example.service..*.*(..))")
        public void serviceLayerExecution() {} // 포인트컷 시그니처 (이름만 사용됨)
    
        // MyService 인터페이스를 구현한 모든 빈의 메소드 실행을 포인트컷으로 정의
        @Pointcut("within(com.example.service.MyService+)") // +는 해당 타입 및 하위 타입 모두 포함
        public void myServiceMethods() {}
    
        // @Loggable 어노테이션이 붙은 메소드 실행을 포인트컷으로 정의
        @Pointcut("@annotation(com.example.Loggable)")
        public void loggableOperation() {}
    
        // 다른 어드바이스에서 이 포인트컷들을 참조하여 사용 가능
        // 예: @Before("serviceLayerExecution()")
    }
    ```


**4. 주요 어드바이스(Advice) 종류 5가지와 실행 시점**

- **스프링 AOP에서 사용하는 주요 어드바이스 종류 5가지와 실행 시점:**
  1. **`@Before` (Before advice):**
    - **실행 시점:** 조인 포인트(대상 메소드)가 **실행되기 전**에 실행됩니다.
    - **특징:** 대상 메소드의 실행 자체를 막을 수는 없습니다 (예외를 던지지 않는 한). 대상 메소드에 전달될 인자에는 접근할 수 있습니다.
  2. **`@AfterReturning` (After returning advice):**
    - **실행 시점:** 조인 포인트(대상 메소드)가 **정상적으로 실행을 완료하고 결과를 반환한 후**에 실행됩니다.
    - **특징:** 대상 메소드가 반환하는 결과 값에 접근할 수 있습니다 (`returning` 속성 사용). 대상 메소드에서 예외가 발생하면 실행되지 않습니다.
  3. **`@AfterThrowing` (After throwing advice):**
    - **실행 시점:** 조인 포인트(대상 메소드) 실행 중 **예외가 발생했을 때** 실행됩니다.
    - **특징:** 발생한 예외 객체에 접근할 수 있습니다 (`throwing` 속성 사용). 예외 복구 로직 등을 수행할 수 있지만, 발생한 예외 자체를 막을 수는 없습니다 (다른 예외로 바꿔 던질 수는 있음).
  4. **`@After` (After (finally) advice):**
    - **실행 시점:** 조인 포인트(대상 메소드)의 실행 결과(정상 종료 또는 예외 발생)에 **상관없이 항상 실행**됩니다. (마치 `try-catch-finally` 블록의 `finally`와 유사)
    - **특징:** 주로 자원 해제와 같이 반드시 수행되어야 하는 로직에 사용됩니다.
  5. **`@Around` (Around advice):**
    - **실행 시점:** 조인 포인트(대상 메소드) **실행 전후에 모두 개입**할 수 있습니다.
    - **특징:** 가장 강력한 어드바이스입니다. 대상 메소드의 **실행 여부를 직접 제어**할 수 있으며 (`ProceedingJoinPoint.proceed()` 호출), **반환 값을 변경**하거나, **예외를 가로채서 다른 동작을 수행**할 수도 있습니다. `ProceedingJoinPoint` 타입의 매개변수를 반드시 첫 번째로 받아야 합니다. 성능에 민감한 경우 신중하게 사용해야 합니다.
- **코드 예시 (간략):**

    ```java
    @Aspect
    @Component
    public class ComprehensiveAspect {
    
        @Pointcut("execution(* com.example.TargetService.doSomething(String)) && args(message)")
        public void targetMethodExecution(String message) {}
    
        @Before("targetMethodExecution(message)")
        public void beforeAdvice(String message) {
            System.out.println("[Before] Method called with message: " + message);
        }
    
        @AfterReturning(pointcut = "targetMethodExecution(message)", returning = "result")
        public void afterReturningAdvice(String message, Object result) {
            System.out.println("[AfterReturning] Method returned: " + result + " for message: " + message);
        }
    
        @AfterThrowing(pointcut = "targetMethodExecution(message)", throwing = "ex")
        public void afterThrowingAdvice(String message, Exception ex) {
            System.err.println("[AfterThrowing] Exception: " + ex.getMessage() + " for message: " + message);
        }
    
        @After("targetMethodExecution(message)")
        public void afterFinallyAdvice(String message) {
            System.out.println("[After] Method execution finished for message: " + message);
        }
    
        @Around("targetMethodExecution(message)")
        public Object aroundAdvice(ProceedingJoinPoint pjp, String message) throws Throwable {
            System.out.println("[Around] Before method execution for message: " + message);
            // 대상 메소드 실행 전 작업
            Object result = pjp.proceed(); // 대상 메소드 실행 (필수 호출)
            // 대상 메소드 실행 후 작업
            System.out.println("[Around] After method execution. Result: " + result);
            return "Modified: " + result; // 반환 값 변경 가능
        }
    }
    ```


---

### **AspectJ 포인트컷 표현식 주요 지시자 및 예시 코드**

**서비스 인터페이스 및 구현체 예시:**

```java
package com.example.service;

import org.springframework.stereotype.Service;

// 애스펙트 적용 대상이 될 서비스 인터페이스
public interface MyService {
    String getGreeting(String name);
    void processData(String data, int count) throws IllegalArgumentException;
    String getSecretData();
}

// 서비스 구현체
@Service("mySimpleService") // 빈 이름 "mySimpleService"
public class MySimpleServiceImpl implements MyService {
    @Override
    public String getGreeting(String name) {
        System.out.println("MySimpleServiceImpl: getGreeting called with " + name);
        return "Hello, " + name + "!";
    }

    @Override
    public void processData(String data, int count) throws IllegalArgumentException {
        if (count < 0) {
            throw new IllegalArgumentException("Count cannot be negative");
        }
        System.out.println("MySimpleServiceImpl: processData called with data=" + data + ", count=" + count);
    }

    @Override
    public String getSecretData() {
        System.out.println("MySimpleServiceImpl: getSecretData called");
        return "This is secret data.";
    }
}

// 또 다른 서비스 구현체 (다른 패키지에 있다고 가정)
package com.example.another.service;

import org.springframework.stereotype.Service;
import com.example.service.MyService; // MyService 인터페이스 임포트

@Service("anotherService")
public class AnotherServiceImpl implements MyService {
    @Override
    public String getGreeting(String name) {
        System.out.println("AnotherServiceImpl: getGreeting called with " + name);
        return "Greetings, " + name + "!";
    }
    // processData, getSecretData 메소드는 MySimpleServiceImpl과 동일하다고 가정 (생략)
    @Override public void processData(String data, int count) throws IllegalArgumentException {}
    @Override public String getSecretData() { return "Another secret"; }
}
```

**커스텀 어노테이션 예시 (나중에 `@annotation` 지시자에서 사용):**

```java
package com.example.annotation;

import java.lang.annotation.ElementType;
import java.lang.annotation.Retention;
import java.lang.annotation.RetentionPolicy;
import java.lang.annotation.Target;

@Target(ElementType.METHOD) // 메소드에 적용 가능한 어노테이션
@Retention(RetentionPolicy.RUNTIME) // 런타임까지 어노테이션 정보 유지
public @interface LogExecutionTime {
}
```

**애스펙트 클래스 예시 (여기에 포인트컷들을 정의):**

```java
package com.example.aop;

import org.aspectj.lang.JoinPoint;
import org.aspectj.lang.annotation.Aspect;
import org.aspectj.lang.annotation.Before;
import org.aspectj.lang.annotation.Pointcut;
import org.springframework.stereotype.Component;

@Aspect
@Component
public class PointcutExampleAspect {

    // --- 1. execution 지시자 ---
    // 가장 많이 사용되고 강력한 지시자. 메소드 실행 조인 포인트를 매칭.
    // 형식: execution(수식어? 리턴타입 선언타입?메소드이름(파라미터) 예외?)
    // ?는 생략 가능, *는 모든 값, ..는 0개 이상

    // public 접근 제어자, 리턴 타입은 String, com.example.service.MyService 인터페이스의
    // getGreeting 메소드, 파라미터는 String 하나인 경우
    @Pointcut("execution(public String com.example.service.MyService.getGreeting(String))")
    public void greetingExecution() {}

    // 리턴 타입은 모든 타입(*), com.example.service 패키지의 모든 클래스(*),
    // 모든 메소드 이름(*), 파라미터는 0개 이상(..)
    @Pointcut("execution(* com.example.service.*.*(..))")
    public void allMethodsInServicePackage() {}

    // com.example 패키지 및 그 하위 패키지(..)의 모든 클래스, 모든 메소드
    @Pointcut("execution(* com.example..*.*(..))")
    public void allMethodsInExamplePackageAndSubpackages() {}

    // MyService 인터페이스의 모든 메소드 (구현체 클래스가 아닌 인터페이스 기준)
    @Pointcut("execution(* com.example.service.MyService.*(..))")
    public void allMethodsInMyService() {}

    // --- 2. within 지시자 ---
    // 특정 타입 내의 모든 조인 포인트(메소드 실행)를 매칭. execution보다 간단.

    // com.example.service.MySimpleServiceImpl 클래스 내의 모든 메소드
    @Pointcut("within(com.example.service.MySimpleServiceImpl)")
    public void methodsWithinMySimpleServiceImpl() {}

    // com.example.service 패키지 내의 모든 클래스의 모든 메소드
    @Pointcut("within(com.example.service.*)")
    public void methodsWithinServicePackage() {}

    // com.example 패키지 및 그 하위 패키지 내의 모든 클래스의 모든 메소드
    @Pointcut("within(com.example..*)")
    public void methodsWithinExamplePackageAndSubpackages() {}

    // --- 3. this 지시자 ---
    // 스프링 AOP에서 프록시 객체의 타입이 지정된 타입과 일치하는 조인 포인트를 매칭.
    // (주의: target과 유사하지만, 프록시 자체를 대상으로 함)

    // 프록시 객체가 MyService 인터페이스를 구현한 경우
    @Pointcut("this(com.example.service.MyService)")
    public void thisMyServiceProxy() {}

    // --- 4. target 지시자 ---
    // 실제 대상(target) 객체의 타입이 지정된 타입과 일치하는 조인 포인트를 매칭.
    // (주의: this와 유사하지만, 실제 대상 객체를 대상으로 함)

    // 대상 객체가 MyService 인터페이스를 구현한 경우
    @Pointcut("target(com.example.service.MyService)")
    public void targetMyServiceObject() {}

    // --- 5. args 지시자 ---
    // 메소드의 인자 타입이 지정된 타입과 일치하는 조인 포인트를 매칭.
    // 어드바이스에서 인자 값을 직접 받을 때도 유용.

    // 첫 번째 인자가 String 타입인 모든 메소드
    @Pointcut("args(String)") // 파라미터가 String 하나인 경우
    public void methodsWithSingleStringArg() {}

    // 첫 번째 인자가 String, 두 번째 인자가 int인 모든 메소드
    @Pointcut("args(String, int)")
    public void methodsWithStringAndIntArgs() {}

    // 모든 파라미터를 받고 싶을 때 (execution의 (..) 와 유사한 효과는 아님, 타입 지정)
    // @Pointcut("args(..)") // 이런 사용은 잘 안함. execution에서 파라미터 패턴으로 제어.

    // 어드바이스에서 인자 값 사용 예시 (아래 Before 어드바이스 참고)
    @Pointcut("execution(* com.example.service.MyService.processData(String, int)) && args(data, count)")
    public void processDataArgs(String data, int count) {}

    // --- 6. @target 지시자 ---
    // 대상 객체의 클래스에 특정 어노테이션이 있는 조인 포인트를 매칭.

    // 대상 객체의 클래스에 @org.springframework.stereotype.Service 어노테이션이 있는 경우
    @Pointcut("@target(org.springframework.stereotype.Service)")
    public void methodsInServiceAnnotatedClass() {}

    // --- 7. @within 지시자 ---
    // 지정된 어노테이션이 있는 타입 내의 조인 포인트를 매칭.
    // @target과 유사하지만, 상속 관계 등에서 미묘한 차이가 있을 수 있음.

    // @org.springframework.stereotype.Service 어노테이션이 선언된 클래스 내부의 모든 메소드
    @Pointcut("@within(org.springframework.stereotype.Service)")
    public void methodsWithinServiceAnnotatedType() {}

    // --- 8. @annotation 지시자 ---
    // 특정 어노테이션이 붙은 메소드 실행 조인 포인트를 매칭.

    // com.example.annotation.LogExecutionTime 어노테이션이 붙은 메소드
    @Pointcut("@annotation(com.example.annotation.LogExecutionTime)")
    public void loggableMethods() {}

    // --- 9. @args 지시자 ---
    // 전달되는 인자의 런타임 타입에 특정 어노테이션이 있는 조인 포인트를 매칭.
    // (자주 사용되지는 않음)

    // 첫 번째 인자의 런타임 타입에 MyArgAnnotation 어노테이션이 있는 경우 (MyArgAnnotation 정의 필요)
    // @Pointcut("@args(com.example.annotation.MyArgAnnotation, ..)")
    // public void methodsWithAnnotatedArg() {}

    // --- 10. bean 지시자 ---
    // 스프링 빈의 이름(ID)으로 조인 포인트를 매칭.

    // 빈 이름이 "mySimpleService"인 빈의 모든 public 메소드
    @Pointcut("bean(mySimpleService)")
    public void mySimpleServiceBeanMethods() {}

    // 와일드카드 사용 가능: 이름이 "Service"로 끝나는 모든 빈
    @Pointcut("bean(*Service)")
    public void allServiceBeansMethods() {}

    // --- 포인트컷 조합 (&&, ||, !) ---
    @Pointcut("allMethodsInServicePackage() && methodsWithSingleStringArg()")
    public void serviceMethodsWithSingleStringArg() {}

    @Pointcut("greetingExecution() || processDataArgs(String, int)") // processDataArgs는 인자를 받도록 정의했으므로 타입 명시
    public void greetingOrProcessData() {}

    @Pointcut("allMethodsInServicePackage() && !greetingExecution()")
    public void serviceMethodsExceptGreeting() {}

    // --- 어드바이스 예시 (위 포인트컷들을 사용) ---
    @Before("greetingExecution()")
    public void beforeGreeting(JoinPoint joinPoint) {
        System.out.println("[ASPECT LOG - greetingExecution] Before: " + joinPoint.getSignature().toShortString());
    }

    @Before("allMethodsInServicePackage()")
    public void beforeAllServiceMethods(JoinPoint joinPoint) {
        System.out.println("[ASPECT LOG - allMethodsInServicePackage] Before: " + joinPoint.getSignature().toShortString());
    }

    // args()로 정의된 포인트컷을 사용하여 어드바이스에서 인자 값 직접 받기
    @Before("processDataArgs(data, count)")
    public void beforeProcessDataWithArgs(String data, int count) {
        System.out.println("[ASPECT LOG - processDataArgs] Before processData. Data: " + data + ", Count: " + count);
    }

    @Before("mySimpleServiceBeanMethods()")
    public void beforeMySimpleServiceBean(JoinPoint joinPoint) {
        System.out.println("[ASPECT LOG - mySimpleServiceBeanMethods] Before method in mySimpleService: " + joinPoint.getSignature().getName());
    }
}
```

**각 지시자에 대한 설명 및 실행 확인 방법:**

위 `PointcutExampleAspect` 클래스와 서비스 클래스들을 스프링 컨텍스트에 빈으로 등록하고, `MySimpleServiceImpl`이나 `AnotherServiceImpl`의 메소드들을 호출해보면, 각 포인트컷에 매칭되는 시점에 `@Before` 어드바이스가 실행되어 콘솔에 로그가 찍히는 것을 확인할 수 있습니다.

1. **`execution`:**
  - `greetingExecution()`: `MySimpleServiceImpl.getGreeting("...")` 호출 시 `[ASPECT LOG - greetingExecution]` 로그 출력.
  - `allMethodsInServicePackage()`: `com.example.service` 패키지 내의 모든 빈(예: `MySimpleServiceImpl`)의 public 메소드 호출 시 `[ASPECT LOG - allMethodsInServicePackage]` 로그 출력. `AnotherServiceImpl`은 다른 패키지이므로 적용 안 됨.
  - `allMethodsInExamplePackageAndSubpackages()`: `com.example` 및 하위 패키지(예: `com.example.service`, `com.example.another.service`)의 모든 빈의 public 메소드 호출 시 로그 출력.
  - `allMethodsInMyService()`: `MyService` 인터페이스를 구현한 모든 빈(`MySimpleServiceImpl`, `AnotherServiceImpl`)의 public 메소드 호출 시 로그 출력.
2. **`within`:**
  - `methodsWithinMySimpleServiceImpl()`: 오직 `MySimpleServiceImpl` 클래스 내부의 메소드 호출 시에만 로그 출력 (인터페이스 기반이 아닌 클래스 기반).
  - `methodsWithinServicePackage()`: `execution`의 패키지 지정과 유사하게 동작.
  - `methodsWithinExamplePackageAndSubpackages()`: `execution`의 패키지 및 하위 패키지 지정과 유사하게 동작.
3. **`this` vs. `target`:**
  - 스프링 AOP는 기본적으로 JDK 동적 프록시 (인터페이스 기반) 또는 CGLIB 프록시 (클래스 기반)를 사용합니다.
  - `this(com.example.service.MyService)`: 프록시 객체 자체가 `MyService` 타입인 경우에 매칭.
  - `target(com.example.service.MyService)`: 프록시가 감싸고 있는 실제 대상 객체가 `MyService` 타입인 경우에 매칭.
  - 대부분의 경우 두 지시자는 유사하게 동작하지만, 프록시와 타겟을 엄격히 구분해야 하는 고급 시나리오에서 차이가 드러날 수 있습니다. 일반적으로는 `execution`이나 `within`을 더 많이 사용합니다.
4. **`args`:**
  - `methodsWithSingleStringArg()`: `getGreeting(String name)` 호출 시 매칭.
  - `methodsWithStringAndIntArgs()`: `processData(String data, int count)` 호출 시 매칭.
  - `processDataArgs(data, count)`와 함께 `@Before("processDataArgs(data, count)") public void beforeProcessDataWithArgs(String data, int count)` 어드바이스를 사용하면, 어드바이스 메소드에서 `data`와 `count` 파라미터 값을 직접 사용할 수 있습니다. 매우 편리한 기능입니다.
5. **`@target` vs. `@within`:**
  - `@target(org.springframework.stereotype.Service)`: `MySimpleServiceImpl`이나 `AnotherServiceImpl` (둘 다 `@Service` 어노테이션 가짐)의 메소드 호출 시 매칭.
  - `@within(org.springframework.stereotype.Service)`: `@target`과 유사하게 동작. `@target`은 대상 객체의 런타임 타입을 기준으로 어노테이션을 확인하고, `@within`은 대상 객체의 클래스(또는 상위 클래스/인터페이스)에 선언된 어노테이션을 기준으로 합니다. 상속 관계에서 미묘한 차이가 발생할 수 있습니다.
6. **`@annotation`:**
  - 만약 `MySimpleServiceImpl`의 특정 메소드에 `@com.example.annotation.LogExecutionTime` 어노테이션을 붙였다면, 해당 메소드 호출 시 `loggableMethods()` 포인트컷이 매칭됩니다.
7. **`bean`:**
  - `mySimpleServiceBeanMethods()`: `MySimpleServiceImpl` 빈 (이름이 "mySimpleService")의 메소드 호출 시 `[ASPECT LOG - mySimpleServiceBeanMethods]` 로그 출력.
  - `allServiceBeansMethods()`: 이름이 "Service"로 끝나는 모든 빈(예: "mySimpleService", "anotherService")의 메소드 호출 시 로그 출력.

**포인트컷 조합:**
논리 연산자 `&&` (AND), `||` (OR), `!` (NOT)을 사용하여 기존에 정의된 포인트컷들을 조합하여 더 복잡하고 정교한 포인트컷을 만들 수 있습니다.

---
