---
title: Advisor API
description: 
author: laze
date: 2025-05-06 00:00:08 +0900
categories: [Dev, SpringBoot]
tags: [SpringBoot]
---
**The Advisor API in Spring**

스프링에서 어드바이저(Advisor)는 포인트컷 표현식(pointcut expression)과 연관된 단일 어드바이스 객체(single advice object)만 포함하는 애스펙트(aspect)입니다.

인트로덕션(introductions)의 특별한 경우를 제외하고, 모든 어드바이저는 모든 어드바이스(any advice)와 함께 사용될 수 있습니다.

`org.springframework.aop.support.DefaultPointcutAdvisor`는 가장 일반적으로 사용되는 어드바이저 클래스입니다.

이것은 `MethodInterceptor`, `BeforeAdvice`, 또는 `ThrowsAdvice`와 함께 사용될 수 있습니다.

동일한 AOP 프록시 내에서 스프링의 어드바이저와 어드바이스 타입들을 혼합하는 것이 가능합니다.

예를 들어, 하나의 프록시 구성에서 interception around 어드바이스, throws 어드바이스, before 어드바이스를 사용할 수 있습니다. 스프링은 자동으로 필요한 인터셉터 체인(interceptor chain)을 생성합니다.

---

**전체 주제: 스프링의 어드바이저 API (The Advisor API in Spring)**

이 부분은 스프링 AOP에서 사용되는 **어드바이저(Advisor)** 라는 개념과 그 역할, 그리고 주요 구현 클래스에 대해 설명합니다. 어드바이저는 애스펙트보다 작은 단위의 AOP 설정 요소입니다.

**핵심 아이디어:** 어드바이저는 딱 하나의 부가 기능(어드바이스)과 그 기능이 적용될 대상(포인트컷)을 묶어놓은 작은 AOP 패키지다.

---

**1. 어드바이저(Advisor)란 무엇인가?**

- **개념:** 어드바이저는 **단 하나의 어드바이스(Advice)** 와 그 어드바이스가 적용될 위치를 결정하는 **단 하나의 포인트컷(Pointcut)** 을 **가지고 있는 객체**입니다.
- **애스펙트(Aspect)와의 비교:**
  - **애스펙트:** 여러 개의 포인트컷과 여러 종류의 어드바이스(Before, After, Around 등)를 포함할 수 있는 **더 크고 포괄적인 AOP 모듈**입니다.
  - **어드바이저:** 딱 **하나의 어드바이스**와 **하나의 포인트컷**만을 가집니다. 훨씬 **단순하고 작은 단위**입니다.
- **비유:**
  - 애스펙트가 여러 도구(망치, 드라이버, 펜치)와 사용 설명서(포인트컷들)가 들어있는 **"공구함 세트"** 라면,
  - 어드바이저는 **"망치(어드바이스)와 그 망치를 언제 써야 하는지 적힌 쪽지(포인트컷)"** 가 들어있는 **작은 주머니**와 같습니다.
- **용도:** 특정 부가 기능(어드바이스)을 특정 대상(포인트컷)에 적용하는 비교적 **간단한 AOP 설정**에 유용합니다. 특히 스프링에서는 미리 만들어진 특정 어드바이스(예: 트랜잭션 관리 어드바이스)를 특정 포인트컷(예: 서비스 계층 메소드)에 연결할 때 많이 사용됩니다.

---

**2. 어드바이저의 구성 요소:**

- **어드바이스(Advice):** 실제 부가 기능 로직을 담고 있는 객체입니다. 스프링 API 기반 어드바이스의 경우 `org.aopalliance.aop.Advice` 인터페이스 (또는 그 하위 인터페이스: `MethodInterceptor`, `BeforeAdvice`, `ThrowsAdvice`, `AfterReturningAdvice`)를 구현한 빈(Bean) 객체입니다.
- **포인트컷(Pointcut):** 어드바이스가 적용될 조인 포인트(메소드 실행)를 결정하는 규칙입니다. `org.springframework.aop.Pointcut` 인터페이스를 구현한 객체입니다. (예: `AspectJExpressionPointcut`, `NameMatchMethodPointcut`)

---

**3. 주요 어드바이저 구현체: `DefaultPointcutAdvisor`**

- **`org.springframework.aop.support.DefaultPointcutAdvisor`**: 스프링에서 **가장 일반적으로 사용되는** `Advisor` 인터페이스의 구현 클래스입니다.
- **특징:** 생성자나 세터 메소드를 통해 **하나의 `Pointcut` 객체**와 **하나의 `Advice` 객체**를 받아 저장합니다.
- **활용:** `MethodInterceptor`(Around), `MethodBeforeAdvice`(Before), `ThrowsAdvice`(After Throwing), `AfterReturningAdvice`(After Returning) 등 **대부분의 표준 스프링 어드바이스 타입**과 함께 사용할 수 있습니다. (단, 인트로덕션 어드바이스는 특별한 `IntroductionAdvisor` 사용)
- **설정 예시 (XML - 이전에 본 트랜잭션 예시):**
  위 설정에서 `<aop:advisor>`는 `DefaultPointcutAdvisor` 빈을 생성하고, `pointcut-ref`로 지정된 포인트컷과 `advice-ref`로 지정된 어드바이스 빈(`tx-advice`)을 연결해주는 역할을 합니다.

    ```xml
    <aop:config>
        <!-- 포인트컷 정의 -->
        <aop:pointcut id="businessService" expression="execution(* com.xyz.service.*.*(..))"/>
    
        <!-- 어드바이저 정의 -->
        <aop:advisor
            pointcut-ref="businessService" <!-- 사용할 포인트컷 참조 -->
            advice-ref="tx-advice" />     <!-- 사용할 어드바이스 빈 참조 -->
    </aop:config>
    
    <!-- 트랜잭션 어드바이스 빈 정의 (tx 네임스페이스 사용) -->
    <tx:advice id="tx-advice" transaction-manager="transactionManager">
        <!-- ... -->
    </tx:advice>
    
    ```


---

**4. 어드바이저와 어드바이스 혼합 사용:**

- 하나의 AOP 프록시 설정 내에서 `@AspectJ` 스타일로 정의된 어드바이스, XML 스키마로 정의된 애스펙트의 어드바이스, 그리고 `<aop:advisor>`로 정의된 어드바이저들을 **자유롭게 혼합하여 사용**할 수 있습니다.
- **예시:** 트랜잭션은 `<aop:advisor>`를 사용하고, 로깅은 `@AspectJ` 애스펙트를 사용하고, 특정 예외 처리는 XML 기반 애스펙트를 사용하는 것이 가능합니다.
- **동작:** 스프링 AOP는 이렇게 다양한 방식으로 정의된 모든 AOP 설정들을 모아서 **내부적으로는 동일한 인터셉터 체인(Interceptor Chain) 메커니즘**으로 통합하여 처리합니다. 따라서 서로 다른 스타일의 설정들이 충돌 없이 함께 작동할 수 있습니다.

**요약:**

스프링의 **어드바이저(Advisor)** 는 **단 하나의 어드바이스(부가 기능)** 와 **단 하나의 포인트컷(적용 대상)** 을 묶어놓은 **간단한 AOP 설정 단위**입니다. 여러 어드바이스와 포인트컷을 가질 수 있는 애스펙트(Aspect)보다 작은 개념입니다. 가장 일반적인 구현체는 `DefaultPointcutAdvisor`이며, 주로 미리 정의된 어드바이스 빈(특히 트랜잭션 어드바이스)을 특정 포인트컷에 적용할 때 사용됩니다. 스프링에서는 어노테이션 기반 애스펙트, XML 기반 애스펙트, 어드바이저 등 다양한 방식의 AOP 설정을 **함께 혼합하여 사용**할 수 있습니다.
