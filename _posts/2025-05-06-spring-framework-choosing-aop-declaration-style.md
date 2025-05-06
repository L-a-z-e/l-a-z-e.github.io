---
title: Choosing AOP Declaration Style
description: 
author: laze
date: 2025-05-06 00:00:02 +0900
categories: [Dev, SpringBoot]
tags: [SpringBoot]
---
**Choosing which AOP Declaration Style to Use**

주어진 요구 사항을 구현하는 데 애스펙트(aspect)가 최상의 접근 방식이라고 결정했다면,

스프링 AOP 또는 AspectJ 사용 여부와 애스펙트 언어(코드) 스타일, `@AspectJ` 어노테이션 스타일 또는 스프링 XML 스타일 중 어떤 것을 사용할지 어떻게 결정할까요?

이러한 결정은 애플리케이션 요구 사항, 개발 도구, 팀의 AOP 숙련도를 포함한 여러 요인에 의해 영향을 받습니다.

**스프링 AOP 또는 완전한 AspectJ? (Spring AOP or Full AspectJ?)**

작동할 수 있는 가장 간단한 것을 사용하십시오.

스프링 AOP는 개발 및 빌드 프로세스에 AspectJ 컴파일러/위버(weaver)를 도입할 필요가 없으므로 완전한 AspectJ를 사용하는 것보다 간단합니다.

스프링 빈에 대한 작업 실행만 어드바이스(advise)하면 된다면 스프링 AOP가 올바른 선택입니다.

스프링 컨테이너에서 관리하지 않는 객체(예: 일반적으로 도메인 객체)를 어드바이스해야 하는 경우 AspectJ를 사용해야 합니다.

또한 단순한 메소드 실행 이외의 조인 포인트(예: 필드 get 또는 set 조인 포인트 등)를 어드바이스하려는 경우에도 AspectJ를 사용해야 합니다.

AspectJ를 사용할 때 AspectJ 언어 구문( "코드 스타일"이라고도 함) 또는 `@AspectJ` 어노테이션 스타일 중 하나를 선택할 수 있습니다.

애스펙트가 디자인에서 큰 역할을 하고 Eclipse용 AspectJ 개발 도구(AJDT) 플러그인을 사용할 수 있는 경우, AspectJ 언어 구문이 선호되는 옵션입니다.

언어가 애스펙트 작성을 위해 의도적으로 설계되었기 때문에 더 깔끔하고 간단합니다.

Eclipse를 사용하지 않거나 애플리케이션에서 주요 역할을 하지 않는 몇 개의 애스펙트만 있는 경우, IDE에서 일반 자바 컴파일을 고수하고 빌드 스크립트에 애스펙트 위빙 단계를 추가하는 `@AspectJ` 스타일 사용을 고려할 수 있습니다.

**스프링 AOP를 위한 @AspectJ 또는 XML? (@AspectJ or XML for Spring AOP?)**

스프링 AOP를 사용하기로 선택했다면 `@AspectJ` 또는 XML 스타일 중 하나를 선택할 수 있습니다.

고려해야 할 다양한 트레이드오프(tradeoffs)가 있습니다.

XML 스타일은 기존 스프링 사용자에게 가장 익숙할 수 있으며 진정한 POJO에 의해 지원됩니다.

엔터프라이즈 서비스를 구성하기 위한 도구로 AOP를 사용할 때 XML은 좋은 선택일 수 있습니다 (좋은 테스트는 포인트컷 표현식을 독립적으로 변경하고 싶을 수 있는 구성의 일부로 간주하는지 여부입니다).

XML 스타일을 사용하면 시스템에 어떤 애스펙트가 존재하는지 구성에서 틀림없이 더 명확합니다.

XML 스타일에는 두 가지 단점이 있습니다. 첫째, 해결하는 요구 사항의 구현을 단일 장소에 완전히 캡슐화하지 못합니다.

DRY 원칙(DRY principle)은 시스템 내 모든 지식 조각에 대해 단일하고 명확하며 권위 있는 표현이 있어야 한다고 말합니다.

XML 스타일을 사용할 때 요구 사항이 구현되는 방식에 대한 지식은 백킹 빈 클래스 선언과 구성 파일의 XML로 분할됩니다.

`@AspectJ` 스타일을 사용하면 이 정보는 단일 모듈인 애스펙트에 캡슐화됩니다.

둘째, XML 스타일은 `@AspectJ` 스타일보다 표현할 수 있는 것이 약간 더 제한적입니다: "싱글톤" 애스펙트 인스턴스화 모델만 지원되며 XML에 선언된 이름 붙여진 포인트컷을 결합하는 것이 불가능합니다.

예를 들어, `@AspectJ` 스타일에서는 다음과 같이 작성할 수 있습니다:

```java
// Java
import org.aspectj.lang.annotation.Aspect;
import org.aspectj.lang.annotation.Pointcut;

@Aspect
public class PointcutCombinationExample {

    @Pointcut("execution(* get*())")
    public void propertyAccess() {}

    // Assuming Account class exists in com.xyz package or subpackages
    @Pointcut("execution(com.xyz.Account+ *(..))") // '+' indicates Account and its subclasses
    public void operationReturningAnAccount() {}

    // Combine named pointcuts
    @Pointcut("propertyAccess() && operationReturningAnAccount()")
    public void accountPropertyAccess() {}
}
```

```kotlin
// Kotlin
import org.aspectj.lang.annotation.Aspect
import org.aspectj.lang.annotation.Pointcut

@Aspect
class PointcutCombinationExample {

    @Pointcut("execution(* get*())")
    fun propertyAccess() {}

    // Assuming Account class exists in com.xyz package or subpackages
    @Pointcut("execution(com.xyz.Account+ *(..))") // '+' indicates Account and its subclasses
    fun operationReturningAnAccount() {}

    // Combine named pointcuts
    @Pointcut("propertyAccess() && operationReturningAnAccount()")
    fun accountPropertyAccess() {}
}
```

XML 스타일에서는 처음 두 포인트컷을 선언할 수 있습니다:

```xml
<aop:pointcut id="propertyAccess"
		expression="execution(* get*())"/>

<aop:pointcut id="operationReturningAnAccount"
		expression="execution(com.xyz.Account+ *(..))"/>
		
```

XML 접근 방식의 단점은 이러한 정의들을 결합하여 `accountPropertyAccess` 포인트컷을 정의할 수 없다는 것입니다.

`@AspectJ` 스타일은 추가적인 인스턴스화 모델과 더 풍부한 포인트컷 구성을 지원합니다.

애스펙트를 모듈 단위로 유지하는 장점이 있습니다. 또한 `@AspectJ` 애스펙트는 스프링 AOP와 AspectJ 모두에서 이해(따라서 소비)될 수 있다는 장점도 있습니다.

따라서 나중에 추가 요구 사항을 구현하기 위해 AspectJ의 기능이 필요하다고 결정하면 클래식 AspectJ 설정으로 쉽게 마이그레이션할 수 있습니다.

일반적으로 스프링 팀은 엔터프라이즈 서비스의 단순한 구성을 넘어서는 커스텀 애스펙트에 대해 `@AspectJ` 스타일을 선호합니다.
