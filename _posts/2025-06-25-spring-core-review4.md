---
title: Spring Core - Review(4)
description: 
author: laze
date: 2025-06-25 00:00:01 +0900
categories: [Dev, SpringBoot]
tags: [SpringBoot]
---
AOP(Aspect Oriented Programming, 관점 지향 프로그래밍)는 OOP(객체 지향 프로그래밍)를 보완하여 애플리케이션 전체에 흩어져 있는 **공통 관심사(Cross-cutting Concerns)**를 효과적으로 모듈화하는 방법입니다.

1. AOP에서 말하는 **"공통 관심사(Cross-cutting Concerns)"**란 무엇이며, 이러한 공통 관심사를 OOP만으로 처리하려고 할 때 발생할 수 있는 문제점은 무엇일까요? (예: 로깅, 트랜잭션 관리, 보안 등)
2. AOP의 핵심 구성 요소에는 **애스펙트(Aspect), 어드바이스(Advice), 포인트컷(Pointcut), 조인 포인트(Join Point)** 등이 있습니다.
  - 각 용어가 무엇을 의미하는지 간략하게 설명해주실 수 있나요?
  - 이들 간의 관계를 간단한 비유를 들어 설명해주시면 더욱 좋겠습니다. (예: "누가(Aspect), 언제 어디서(Pointcut at Join Point), 무엇을(Advice) 할 것인가")

**1. 공통 관심사(Cross-cutting Concerns)와 OOP의 한계**

- **공통 관심사 (Cross-cutting Concerns)란?**
  - 애플리케이션의 핵심 비즈니스 로직(예: 주문 처리, 상품 조회)과는 직접적인 관련이 없지만, 시스템의 여러 모듈(클래스, 메소드)에 걸쳐 **공통적으로 적용되어야 하는 부가 기능**들을 의미합니다.
  - **로깅(Logging)**, **전처리/후처리, 트랜잭션 관리(Transaction Management)**, **보안(Security)**, **성능 모니터링(Performance Monitoring)**, **캐싱(Caching)** 등이 대표적인 공통 관심사입니다.
- **OOP만으로 처리 시 문제점**
  1. **코드 중복 (Code Duplication):** 공통 기능을 모든 관련 모듈에 반복적으로 작성해야 합니다. (예: 모든 서비스 메소드 시작과 끝에 로그 남기는 코드)
  2. **핵심 로직의 오염 (Business Logic Scattering and Tangling):** 핵심 비즈니스 로직 코드 사이에 부가 기능 코드가 섞여 들어가, 코드의 가독성과 응집도를 떨어뜨립니다. 핵심 로직과 부가 기능 코드가 서로 얽히게(tangling) 됩니다.
  3. **유지보수 어려움:** 공통 기능에 변경이 필요할 경우, 관련된 모든 코드를 찾아 수정해야 하므로 유지보수가 어렵고 실수가 발생하기 쉽습니다.
  4. **모듈성 저하:** 부가 기능이 핵심 기능 모듈에 흩어져(scattering) 존재하게 되어, 각 모듈의 독립성과 재사용성이 저하됩니다.

AOP는 이러한 문제점들을 해결하기 위해 등장했습니다. 즉, **공통 관심사를 핵심 비즈니스 로직으로부터 분리하여 별도의 모듈(애스펙트)로 만들고, 필요한 시점에 동적으로 적용**함으로써 코드의 모듈성을 높이고 중복을 제거합니다.

**2. AOP 핵심 용어 (Aspect, Advice, Pointcut, Join Point)**

1. **애스펙트 (Aspect):**
  - **의미:** 여러 객체에 공통으로 적용되는 **관심사(concern)의 모듈화 단위**입니다. 즉, 우리가 분리하고자 하는 공통 관심사(예: 로깅, 트랜잭션)와, 이 관심사를 언제 어디에 적용할지에 대한 정보(어드바이스 + 포인트컷)를 하나로 묶은 것입니다.
  - **구현:** 스프링 AOP에서는 보통 `@Aspect` 어노테이션이 붙은 자바 클래스로 표현됩니다. 이 클래스 안에는 어드바이스와 포인트컷이 정의됩니다.
  - **비유:** "특정 임무를 수행하는 특수 요원 팀"이라고 생각할 수 있습니다. 이 팀은 "언제, 어디서(포인트컷)", "어떤 작전(어드바이스)"을 펼칠지 계획을 가지고 있습니다.
2. **조인 포인트 (Join Point):**
  - **의미:** 애플리케이션 실행 흐름 중 애스펙트가 **끼어들 수 있는 모든 지점 또는 시점**을 의미합니다. 즉, 어드바이스가 적용될 수 있는 모든 "후보" 위치입니다.
  - **예시:**
    - 메소드 호출 시점 (가장 일반적)
    - 메소드 실행 시점
    - 필드 값 변경 시점
    - 예외 발생 시점
    - 객체 생성 시점 등
  - 스프링 AOP는 프록시 기반이므로, **메소드 실행 조인 포인트만 지원**합니다. 즉, 스프링 빈의 public 메소드 호출 시점에만 개입할 수 있습니다.
  - **비유:** "요원이 작전을 펼칠 수 있는 모든 가능한 장소나 시간들" (예: 건물의 모든 출입구, 모든 복도, 특정 시간대 등)
3. **포인트컷 (Pointcut):**
  - **의미:** 여러 조인 포인트 중에서, **실제로 어드바이스가 적용될 조인 포인트를 선별하는 표현식 또는 규칙**입니다. 즉, "어디에" 부가 기능을 적용할지를 정의합니다.
  - **구현:** 스프링 AOP에서는 AspectJ 포인트컷 표현식을 사용하여 정의합니다. (예: `execution(* com.example.service.*.*(..))`)
  - **역할:** 어드바이스가 적용될 특정 메소드들을 정확하게 지정합니다.
  - **비유:** "요원 팀이 실제로 작전을 펼치기로 결정한 구체적인 장소나 시간" (예: "3층 로비의 정문", "오후 2시 회의실")
4. **어드바이스 (Advice):**
  - **의미:** 특정 조인 포인트에서 애스펙트에 의해 **실행되는 실제 부가 기능 로직**입니다. 즉, "무엇을" 할 것인지를 정의합니다.
  - **종류 (스프링 AOP 기준):**
    - **`@Before`:** 조인 포인트 실행 전에 실행되는 어드바이스.
    - **`@AfterReturning`:** 조인 포인트가 정상적으로 실행 완료된 후 실행되는 어드바이스 (결과 값 접근 가능).
    - **`@AfterThrowing`:** 조인 포인트 실행 중 예외가 발생했을 때 실행되는 어드바이스 (예외 객체 접근 가능).
    - **`@After` (finally):** 조인 포인트 실행 결과(정상/예외)에 상관없이 항상 실행되는 어드바이스.
    - **`@Around`:** 조인 포인트 실행 전후에 모두 개입하여, 조인 포인트의 실행 여부 자체를 제어하거나 반환 값을 변경할 수도 있는 가장 강력한 어드바이스.
  - **비유:** "요원 팀이 특정 장소/시간에 도착해서 수행하는 구체적인 작전 내용" (예: "문 앞에서 신원 확인", "회의 내용 기록", "비상 상황 시 경보 발령")
  - 학생분이 "포인트컷 전후 등에서 처리할 것들"이라고 하신 부분이 어드바이스의 개념과 정확히 일치합니다!

**이들 간의 관계 정리 (비유 확장):**

- **애스펙트 (Aspect - 특수 요원 팀 "알파"):** "알파" 팀은 "중요 정보 유출 방지"라는 임무(공통 관심사)를 가지고 있다.
- **조인 포인트 (Join Point - 가능한 모든 작전 지점):** 회사 내의 모든 회의실, 모든 임원실, 모든 문서고 등이 작전 가능 지점이다.
- **포인트컷 (Pointcut - 실제 작전 지점 선정):** "알파" 팀은 "매주 월요일 오전 10시, 3층 대회의실에서 열리는 임원 회의 시작 전" (구체적인 조인 포인트 선별)에 작전을 수행하기로 결정했다.
- **어드바이스 (Advice - 실제 작전 내용):** "알파" 팀은 해당 시점에 "회의실 입구에서 모든 참석자의 휴대폰을 수거한다"는 작전(부가 기능 로직)을 실행한다.

**코드 레벨에서의 관계:**

```java
import org.aspectj.lang.annotation.Aspect;
import org.aspectj.lang.annotation.Before;
import org.aspectj.lang.annotation.Pointcut;
import org.springframework.stereotype.Component;

@Aspect // 1. 이 클래스가 애스펙트임을 선언
@Component // 스프링 빈으로 등록되어야 AOP가 동작
public class LoggingAspect {

    // 3. 포인트컷 정의: com.example.service 패키지 내의 모든 클래스의 모든 메소드 실행
    @Pointcut("execution(* com.example.service..*.*(..))")
    private void allServiceMethods() {} // 포인트컷 시그니처 (이름만 중요)

    // 4. 어드바이스 정의: allServiceMethods() 포인트컷에 해당하는 조인 포인트 실행 전에 이 로직 실행
    @Before("allServiceMethods()")
    public void logBefore(org.aspectj.lang.JoinPoint joinPoint) { // 2. JoinPoint 객체를 통해 실행되는 메소드 정보 등에 접근 가능
        System.out.println("LOG: Executing " + joinPoint.getSignature().getName() + " method.");
    }

    // 추가적인 어드바이스들 (@AfterReturning, @Around 등)과 포인트컷들을 여기에 정의 가능
}
```

- **애스펙트:** `LoggingAspect` 클래스 전체.
- **조인 포인트:** `com.example.service` 패키지 내 모든 클래스의 모든 public 메소드가 실행되는 시점들 (스프링 AOP는 메소드 실행 조인 포인트만 지원).
- **포인트컷:** `@Pointcut("execution(* com.example.service..*.*(..))")` 이라는 표현식. `allServiceMethods()`는 이 포인트컷을 참조하기 위한 이름.
- **어드바이스:** `@Before("allServiceMethods()")` 어노테이션이 붙은 `logBefore()` 메소드. 이 메소드의 내용이 실제 부가 기능 로직입니다.
  - `org.aspectj.lang.JoinPoint joinPoint` 매개변수는 어드바이스가 적용되는 조인 포인트에 대한 정보(예: 실행되는 메소드 시그니처, 전달된 인자 등)를 담고 있습니다.

---

Aspect 어노테이션은 꼭 필요한가?

- **왜 꼭 필요할까요?**
  - 스프링 AOP는 애플리케이션 컨텍스트 내의 빈들 중에서 @Aspect 어노테이션이 붙은 클래스를 찾아서, 그 안에 정의된 @Pointcut과 @Before, @After 등의 어드바이스 어노테이션들을 해석하고 AOP 프록시를 생성하는 데 사용합니다.
  - 만약 @Aspect 어노테이션이 없다면, 스프링 AOP는 어떤 클래스가 애스펙트 역할을 하는지, 그리고 그 안에 정의된 메소드들이 어드바이스나 포인트컷인지 알 수 없습니다. 단순히 일반 자바 클래스로 취급하게 됩니다.
  - **따라서, @Pointcut이나 어드바이스 관련 어노테이션(@Before, @After 등)이 정상적으로 AOP 기능으로 동작하려면, 해당 어노테이션들을 포함하는 클래스에는 반드시 @Aspect 어노테이션이 붙어 있어야 합니다.**
- **추가적으로:**
  - @Aspect가 붙은 클래스는 일반적으로 @Component 어노테이션도 함께 사용하여 스프링 빈으로 등록되어야 스프링 컨테이너가 관리하고 AOP를 적용할 수 있습니다.
  - 또는, Java Config 클래스에서 @Bean 메소드를 통해 애스펙트 클래스의 인스턴스를 빈으로 등록할 수도 있습니다.
  - 그리고 AOP 기능을 사용하려면 스프링 설정에 @EnableAspectJAutoProxy 어노테이션을 추가하여 AspectJ 스타일의 AOP 자동 프록시 생성을 활성화해야 합니다. (스프링 부트에서는 특정 스타터 의존성이 있으면 자동 설정되기도 합니다.

---

### **`JoinPoint` 객체에 대한 구체적인 설명과 코드 예시**

- `org.aspectj.lang.JoinPoint` 인터페이스(및 그 하위 인터페이스 `ProceedingJoinPoint`)는 어드바이스 메소드가 실행될 때, **현재 실행 중인 조인 포인트(예: 특정 메소드 호출)에 대한 다양한 정적 정보와 동적인 상태에 접근할 수 있도록 해주는 객체**입니다.
- 어드바이스 메소드의 매개변수로 `JoinPoint` 타입을 선언하면, 스프링 AOP가 해당 조인 포인트의 컨텍스트 정보를 담은 `JoinPoint` 객체를 자동으로 주입해줍니다.

**`JoinPoint` 객체를 통해 얻을 수 있는 주요 정보:**

- `getSignature()`: 실행되는 조인 포인트(주로 메소드)의 시그니처 정보를 담은 `Signature` 객체를 반환합니다.
  - `Signature.getName()`: 메소드의 이름을 반환합니다.
  - `Signature.toLongString()`: 메소드의 전체 시그니처(수식어, 반환 타입, 클래스명, 메소드명, 파라미터 타입 등)를 문자열로 반환합니다.
  - `Signature.toShortString()`: 메소드의 간략한 시그니처(메소드명과 파라미터 타입)를 문자열로 반환합니다.
  - `Signature.getDeclaringTypeName()`: 메소드가 선언된 클래스/인터페이스의 이름을 반환합니다.
- `getArgs()`: 조인 포인트(메소드)에 전달된 인자(argument)들의 배열(`Object[]`)을 반환합니다.
- `getTarget()`: 어드바이스가 적용된 대상 객체(target object)를 반환합니다. (프록시가 아닌 실제 대상 객체)
- `getThis()`: 어드바이스를 실행하고 있는 현재 프록시 객체(proxy object)를 반환합니다.
- `getKind()`: 조인 포인트의 종류를 나타내는 문자열을 반환합니다 (예: "method-execution").
- `getSourceLocation()`: 조인 포인트의 소스 코드 위치 정보를 반환합니다 (컴파일 옵션에 따라 사용 가능 여부 다름).
- `getStaticPart()`: 조인 포인트의 정적인 부분을 나타내는 객체를 반환합니다.

**`ProceedingJoinPoint` (주로 `@Around` 어드바이스에서 사용):**

- `JoinPoint`를 상속받는 인터페이스로, `@Around` 어드바이스에서만 사용 가능합니다.
- `proceed()`: 실제 대상 조인 포인트(메소드)를 실행합니다. 이 메소드를 호출하지 않으면 대상 메소드가 실행되지 않습니다.
- `proceed(Object[] args)`: 대상 조인 포인트에 새로운 인자를 전달하여 실행합니다.

**구체적인 코드 예시:**

```java
import org.aspectj.lang.JoinPoint;
import org.aspectj.lang.ProceedingJoinPoint;
import org.aspectj.lang.annotation.Around;
import org.aspectj.lang.annotation.Aspect;
import org.aspectj.lang.annotation.Before;
import org.aspectj.lang.annotation.Pointcut;
import org.springframework.stereotype.Component;
import java.util.Arrays;

// 예시 서비스 인터페이스 및 구현체
interface SampleService {
    String greet(String name, int count);
    void processData(Object data);
}

@Component("sampleServiceImpl")
class SampleServiceImpl implements SampleService {
    @Override
    public String greet(String name, int count) {
        System.out.println("SampleServiceImpl: greet() method called with name=" + name + ", count=" + count);
        if (count < 0) {
            throw new IllegalArgumentException("Count cannot be negative");
        }
        return "Hello, " + name + "! You called " + count + " times.";
    }

    @Override
    public void processData(Object data) {
        System.out.println("SampleServiceImpl: processData() method called with data=" + data);
    }
}

// AOP 설정
@Aspect
@Component
public class DetailedLoggingAspect {

    // com.example.aop 패키지 내의 SampleService 인터페이스를 구현한 모든 클래스의 모든 public 메소드를 대상으로 함
    @Pointcut("execution(public * com.example.aop.SampleService+.*(..))")
    public void sampleServicePublicMethods() {}

    // @Before 어드바이스: 대상 메소드 실행 전에 호출됨
    @Before("sampleServicePublicMethods()")
    public void logMethodCallDetails(JoinPoint joinPoint) {
        String methodName = joinPoint.getSignature().getName(); // 메소드 이름
        String className = joinPoint.getTarget().getClass().getSimpleName(); // 대상 객체의 클래스 이름
        Object[] args = joinPoint.getArgs(); // 전달된 인자들

        System.out.println("--------------------------------------------------");
        System.out.println("[Before Advice] Executing: " + className + "." + methodName + "()");
        System.out.println("[Before Advice] Arguments: " + Arrays.toString(args));
        System.out.println("[Before Advice] Full Signature: " + joinPoint.getSignature().toLongString());
        System.out.println("--------------------------------------------------");
    }

    // @Around 어드바이스: 대상 메소드 실행 전후에 개입, 실행 시간 측정 예시
    @Around("execution(* com.example.aop.SampleService.greet(..))") // greet 메소드만 대상으로
    public Object measureGreetExecutionTime(ProceedingJoinPoint proceedingJoinPoint) throws Throwable {
        long startTime = System.currentTimeMillis();

        // 실제 대상 메소드 실행 (proceed() 호출)
        // proceed()를 호출하지 않으면 greet() 메소드는 실행되지 않음
        Object result = proceedingJoinPoint.proceed();

        long endTime = System.currentTimeMillis();
        String methodName = proceedingJoinPoint.getSignature().getName();

        System.out.println("**************************************************");
        System.out.println("[Around Advice] " + methodName + " execution time: " + (endTime - startTime) + "ms");
        System.out.println("[Around Advice] Return value: " + result);
        System.out.println("**************************************************");

        // 필요하다면 여기서 반환 값을 변경할 수도 있음
        return result;
    }
}
```

**위 AOP를 테스트하기 위한 간단한 실행 코드 (스프링 컨텍스트 로드 필요):**

```java
import org.springframework.context.ApplicationContext;
import org.springframework.context.annotation.AnnotationConfigApplicationContext;
import org.springframework.context.annotation.ComponentScan;
import org.springframework.context.annotation.Configuration;
import org.springframework.context.annotation.EnableAspectJAutoProxy;

// 스프링 설정 클래스
@Configuration
@ComponentScan(basePackages = "com.example.aop") // SampleServiceImpl, DetailedLoggingAspect 스캔
@EnableAspectJAutoProxy // AOP 활성화
class AppConfigForAop {}

// 실행 클래스
public class AopTestMain {
    public static void main(String[] args) {
        ApplicationContext context = new AnnotationConfigApplicationContext(AppConfigForAop.class);
        SampleService sampleService = context.getBean("sampleServiceImpl", SampleService.class);

        System.out.println("\\nCalling greet method...");
        String greeting = sampleService.greet("SpringFan", 3);
        System.out.println("Main: Received greeting - " + greeting);

        System.out.println("\\nCalling processData method...");
        sampleService.processData(new Object()); // 간단한 객체 전달
    }
}
// (com.example.aop 패키지 내에 위 클래스들을 위치시켜야 함)
```

**실행 결과 예상 (콘솔 출력 순서):**

1. `DetailedLoggingAspect`의 `@Before` 어드바이스가 `greet` 메소드 실행 전에 호출되어 메소드 이름, 인자 등을 출력.
2. `DetailedLoggingAspect`의 `@Around` 어드바이스가 `greet` 메소드 실행 전후로 실행되어 시간 측정.
3. `SampleServiceImpl`의 `greet` 메소드 본문 실행 및 메시지 출력.
4. `@Around` 어드바이스의 나머지 부분 실행 (시간 및 결과 출력).
5. `main` 메소드에서 `greet` 결과 출력.
6. `DetailedLoggingAspect`의 `@Before` 어드바이스가 `processData` 메소드 실행 전에 호출.
7. `SampleServiceImpl`의 `processData` 메소드 본문 실행.

이 예시를 통해 `JoinPoint` 객체가 어드바이스 메소드에 전달되어, 현재 실행되는 조인 포인트(메소드)에 대한 다양한 정보를 제공하고, `@Around` 어드바이스에서는 `ProceedingJoinPoint`를 통해 실제 메소드 실행을 제어할 수 있음을 확인할 수 있습니다.

---

### **1. `ApplicationContext` 구현체 선택 기준**

- `ApplicationContext`는 인터페이스이고, 스프링은 이 인터페이스에 대한 다양한 구현체를 제공합니다. 어떤 구현체를 사용할지는 주로 **스프링 설정 정보를 어떤 방식으로 제공할 것인가**와 **애플리케이션이 실행되는 환경**에 따라 결정됩니다.

  **주요 `ApplicationContext` 구현체 및 선택 기준:**

  1. **`AnnotationConfigApplicationContext`:**
    - **설정 방식:** **자바 기반 설정 (`@Configuration` 어노테이션이 붙은 클래스와 `@Bean` 메소드)**을 사용하여 빈을 정의하고 구성할 때 사용합니다.
    - **사용 시점:** 주로 독립 실행형 애플리케이션(non-web applications)이나, 스프링 부트가 아닌 환경에서 프로그래밍 방식으로 컨텍스트를 생성할 때 사용됩니다.
    - **예시:** `new AnnotationConfigApplicationContext(AppConfig.class, AnotherConfig.class);` 또는 `new AnnotationConfigApplicationContext("com.example.configpackage");` (패키지 스캔)
    - **선택 기준:** XML 없이 순수 자바 코드로만 스프링 설정을 관리하고 싶을 때 선택합니다. (현대 스프링에서 권장)
  2. **`ClassPathXmlApplicationContext`:**
    - **설정 방식:** **클래스패스(classpath)에 있는 XML 파일**을 사용하여 빈을 정의하고 구성할 때 사용합니다.
    - **사용 시점:** 주로 레거시 프로젝트나 XML 설정을 선호하는 경우, 또는 독립 실행형 애플리케이션에서 사용됩니다.
    - **예시:** `new ClassPathXmlApplicationContext("applicationContext.xml", "services.xml");`
    - **선택 기준:** 기존 XML 설정 파일이 많거나, XML 방식의 명시적인 빈 정의를 선호할 때 선택합니다.
  3. **`FileSystemXmlApplicationContext`:**
    - **설정 방식:** **파일 시스템의 특정 경로에 있는 XML 파일**을 사용하여 빈을 정의하고 구성할 때 사용합니다.
    - **사용 시점:** 클래스패스가 아닌 특정 파일 시스템 위치에서 XML 설정을 로드해야 할 때 사용됩니다. (일반적으로 `ClassPathXmlApplicationContext`보다 덜 사용됨)
    - **예시:** `new FileSystemXmlApplicationContext("C:/myapp/config/applicationContext.xml");`
    - **선택 기준:** 설정 파일이 클래스패스 외부에 위치해야 하는 특별한 경우에 선택합니다.
  4. **`AnnotationConfigServletWebServerApplicationContext` (주로 스프링 부트 웹 환경):**
    - **설정 방식:** 자바 기반 설정을 사용하며, **내장 서블릿 웹 서버(Tomcat, Jetty, Undertow 등)를 시작하고 웹 환경에 필요한 구성**을 자동으로 처리합니다.
    - **사용 시점:** 스프링 부트로 웹 애플리케이션을 개발할 때, `SpringApplication.run()` 메소드 내부에서 주로 이 타입(또는 유사한 웹 환경용 컨텍스트)의 `ApplicationContext`가 생성되고 사용됩니다. 개발자가 직접 `new`로 생성하는 경우는 드뭅니다.
    - **선택 기준:** 스프링 부트를 사용하여 웹 애플리케이션을 만들 때 자동으로 선택됩니다.
  5. **`GenericApplicationContext`, `StaticApplicationContext` 등:**
    - 더 저수준이거나 테스트, 특수한 목적을 위한 구현체들도 있습니다. `GenericApplicationContext`는 프로그래밍 방식으로 빈 정의를 등록하는 데 유연하게 사용될 수 있습니다.

  **결론적으로 선택 기준은 다음과 같습니다:**

  - **설정 메타데이터 형식:**
    - 자바 클래스 기반 설정 (`@Configuration`, `@Bean`) -> `AnnotationConfigApplicationContext`
    - XML 파일 기반 설정 -> `ClassPathXmlApplicationContext` 또는 `FileSystemXmlApplicationContext`
  - **애플리케이션 실행 환경:**
    - 독립 실행형 자바 애플리케이션 -> 위에 언급된 것들 중 선택
    - 웹 애플리케이션 (특히 스프링 부트) -> `AnnotationConfigServletWebServerApplicationContext` (또는 유사한 `WebApplicationContext` 구현체)가 주로 내부적으로 사용됨. 개발자가 직접 선택하기보다는 프레임워크가 관리.

  우리가 지금까지 진행한 예제에서는 주로 자바 기반 설정을 사용했기 때문에 `AnnotationConfigApplicationContext`를 사용한 것입니다.


---

### **2. `@EnableAspectJAutoProxy`의 정확한 동작**

- `@EnableAspectJAutoProxy` 어노테이션은 스프링 AOP를 활성화하고, **AspectJ 스타일의 어노테이션(`@Aspect`, `@Pointcut`, `@Before` 등)으로 정의된 애스펙트들을 스프링 컨테이너가 자동으로 감지하여 해당 애스펙트가 적용될 빈들에 대해 프록시(Proxy) 객체를 생성하도록 지시**하는 역할을 합니다.

  **구체적인 동작 과정:**

  1. **AOP 관련 빈 등록:** `@EnableAspectJAutoProxy` 어노테이션은 스프링 컨테이너에 AOP 처리에 필요한 핵심 빈들을 등록합니다. 가장 중요한 빈 중 하나는 `AnnotationAwareAspectJAutoProxyCreator`라는 `BeanPostProcessor` 구현체입니다.
  2. **`BeanPostProcessor`의 역할:** 이 `AnnotationAwareAspectJAutoProxyCreator`는 컨테이너 내의 모든 빈이 초기화된 후 (즉, `postProcessAfterInitialization` 단계에서) 각 빈을 검사합니다.
  3. **애스펙트 감지 및 프록시 생성 판단:**
    - 먼저 컨테이너 내에서 `@Aspect` 어노테이션이 붙은 빈들(애스펙트들)을 찾습니다.
    - 각 애스펙트 내에 정의된 `@Pointcut` 표현식을 사용하여, 현재 처리 중인 빈의 메소드들 중 어떤 메소드가 해당 포인트컷에 매칭되는지(즉, 어드바이스가 적용되어야 하는지) 확인합니다.
    - 만약 현재 빈의 메소드 중 하나라도 어떤 애스펙트의 포인트컷과 매칭된다면, 스프링은 해당 빈에 대해 **AOP 프록시를 생성**합니다.
  4. **프록시 반환:** 원래의 빈 인스턴스 대신, 이렇게 생성된 AOP 프록시 객체가 컨테이너에 최종적으로 빈으로 등록되거나 다른 곳에 주입됩니다.
  5. **프록시를 통한 어드바이스 실행:** 이후 해당 빈의 메소드가 호출되면, 직접 원본 객체의 메소드가 호출되는 것이 아니라 AOP 프록시 객체의 메소드가 먼저 호출됩니다. 이 프록시는 포인트컷 조건에 따라, 정의된 어드바이스(예: `@Before` 로직)를 실행하고, 그 다음에 실제 원본 객체의 메소드를 호출합니다 (`@Around` 어드바이스의 경우 `proceed()` 호출).

  **간단히 말해, `@EnableAspectJAutoProxy`는 "스프링아, 이제부터 `@Aspect` 붙은 클래스들을 잘 살펴보고, 걔네들이 끼어들어야 할 빈이 있으면 알아서 중간 대리인(프록시)을 만들어서 연결해줘!" 라고 지시하는 스위치와 같습니다.**

  이 어노테이션이 없으면, `@Aspect` 어노테이션을 클래스에 붙여도 스프링 AOP가 동작하지 않습니다. (스프링 부트에서는 `spring-boot-starter-aop` 의존성이 있으면 이 기능이 자동 설정될 수 있습니다.)


---

---

### **3. 스프링 AOP 프록시 종류: JDK Dynamic Proxy vs. CGLIB Proxy**

스프링 AOP는 **런타임 위빙(Runtime Weaving)** 방식을 사용하며, 이를 위해 **프록시(Proxy) 패턴**을 기반으로 동작합니다.

즉, 원본 대상 객체(Target Object) 대신 프록시 객체를 빈으로 등록하고, 이 프록시 객체가 부가 기능(어드바이스) 실행 후 실제 대상 객체의 메소드를 호출하는 방식입니다.

스프링 AOP에서 프록시를 생성하는 방법에는 크게 두 가지가 있습니다:

1. **JDK Dynamic Proxy (JDK 동적 프록시):**
  - **기반:** 자바 표준 JDK에서 제공하는 `java.lang.reflect.Proxy` 클래스와 `InvocationHandler` 인터페이스를 사용합니다.
  - **조건:** 프록시를 생성하려는 **대상 객체가 반드시 하나 이상의 인터페이스를 구현**하고 있어야 합니다. JDK 동적 프록시는 인터페이스를 기반으로 프록시를 생성하기 때문입니다.
  - **동작:** 생성된 프록시는 대상 객체가 구현한 인터페이스와 동일한 타입을 가지게 됩니다. 클라이언트는 이 인터페이스 타입으로 프록시를 사용합니다. 프록시의 메소드가 호출되면 `InvocationHandler`의 `invoke()` 메소드가 실행되고, 여기서 어드바이스 로직 실행 후 실제 대상 객체의 메소드를 리플렉션을 통해 호출합니다.
  - **장점:** 자바 표준 기술이므로 별도의 라이브러리가 필요 없습니다.
  - **단점:** 인터페이스가 없는 클래스에는 적용할 수 없습니다. (클래스 자체를 프록시할 수 없음)
2. **CGLIB (Code Generation Library) Proxy:**
  - **기반:** 외부 라이브러리인 CGLIB을 사용합니다. (스프링에는 기본적으로 내장되어 있거나, 필요시 의존성 추가)
  - **조건:** 대상 객체가 인터페이스를 구현하지 않아도 사용할 수 있습니다. CGLIB는 **대상 클래스를 상속받는 방식**으로 프록시를 생성합니다.
  - **동작:** CGLIB는 대상 클래스의 하위 클래스를 동적으로 생성하여 프록시로 사용합니다. 이 하위 클래스는 대상 클래스의 메소드를 오버라이드하고, 오버라이드된 메소드 내에서 어드바이스 로직 실행 후 `super.메소드()`를 호출하여 실제 대상 객체의 로직을 실행합니다.
  - **장점:** 인터페이스가 없는 클래스에도 AOP를 적용할 수 있습니다. 일반적으로 JDK 동적 프록시보다 약간 더 빠르다는 평가도 있었으나, 현대 JVM에서는 그 차이가 미미할 수 있습니다.
  - **단점:**
    - 대상 클래스가 `final`로 선언되었거나, 프록시할 메소드가 `final`이면 상속 및 오버라이드가 불가능하므로 CGLIB 프록시를 적용할 수 없습니다.
    - 생성자를 두 번 호출하는 등의 약간의 기술적 제약이 있을 수 있습니다.

**스프링 AOP의 프록시 선택 전략:**

- **기본 전략:**
  - 만약 대상 객체가 **하나 이상의 인터페이스를 구현**하고 있다면, 스프링은 기본적으로 **JDK Dynamic Proxy**를 사용합니다.
  - 만약 대상 객체가 **인터페이스를 구현하고 있지 않다면 (클래스만 있다면)**, 스프링은 **CGLIB Proxy**를 사용합니다.
- **강제 CGLIB 사용:** `@EnableAspectJAutoProxy(proxyTargetClass = true)`와 같이 설정하면, 대상 객체가 인터페이스를 구현했는지 여부와 상관없이 **항상 CGLIB Proxy**를 사용하도록 강제할 수 있습니다. (스프링 부트에서는 `spring.aop.proxy-target-class=true`가 기본값인 경우가 많아, 인터페이스 유무와 관계없이 CGLIB를 사용하는 것이 일반적일 수 있습니다.)
