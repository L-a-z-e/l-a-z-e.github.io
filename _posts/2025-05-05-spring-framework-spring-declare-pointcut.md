---
title: Declaring a Pointcut
description: 
author: laze
date: 2025-05-05 00:00:14 +0900
categories: [Dev, SpringBoot]
tags: [SpringBoot]
---
**Declaring a Pointcut**

포인트컷(Pointcuts)은 관심 있는 조인 포인트(join points)를 결정하며, 따라서 어드바이스(advice)가 언제 실행될지 제어할 수 있게 합니다.

스프링 AOP는 스프링 빈에 대한 메소드 실행 조인 포인트만 지원하므로, 포인트컷을 스프링 빈의 메소드 실행과 일치시키는 것으로 생각할 수 있습니다.

포인트컷 선언은 두 부분으로 구성됩니다: 이름과 모든 파라미터를 포함하는 시그니처(signature), 그리고 정확히 어떤 메소드 실행에 관심 있는지 결정하는 포인트컷 표현식(pointcut expression).

AOP의 `@AspectJ` 어노테이션 스타일에서는, 포인트컷 시그니처는 일반 메소드 정의로 제공되고, 포인트컷 표현식은 `@Pointcut` 어노테이션을 사용하여 표시됩니다 (포인트컷 시그니처 역할을 하는 메소드는 `void` 반환 타입을 가져야 함).

예제는 포인트컷 시그니처와 포인트컷 표현식 간의 이 구분을 명확하게 하는 데 도움이 될 수 있습니다.

다음 예제는 `transfer`라는 이름의 모든 메소드 실행과 일치하는 `anyOldTransfer`라는 이름의 포인트컷을 정의합니다:

```java
// Java
import org.aspectj.lang.annotation.Pointcut;

@Pointcut("execution(* transfer(..))") // 포인트컷 표현식
private void anyOldTransfer() {} // 포인트컷 시그니처 (메소드 바디 없음)
```

```kotlin
// Kotlin
import org.aspectj.lang.annotation.Pointcut

@Pointcut("execution(* transfer(..))") // 포인트컷 표현식
private fun anyOldTransfer() {} // 포인트컷 시그니처 (메소드 바디 없음)
```

`@Pointcut` 어노테이션의 값을 형성하는 포인트컷 표현식은 일반적인 AspectJ 포인트컷 표현식입니다.

**지원되는 포인트컷 지정자 (Supported Pointcut Designators)**

스프링 AOP는 포인트컷 표현식에서 사용할 다음 AspectJ 포인트컷 지정자(PCD - pointcut designators)를 지원합니다:

- `execution`: 메소드 실행 조인 포인트 매칭용. 스프링 AOP 작업 시 사용할 기본 포인트컷 지정자입니다.
- `within`: 특정 타입 내의 조인 포인트로 매칭 제한 (스프링 AOP 사용 시 매칭 타입 내에 선언된 메소드 실행).
- `this`: 빈 참조(스프링 AOP 프록시)가 주어진 타입의 인스턴스인 조인 포인트(스프링 AOP 사용 시 메소드 실행)로 매칭 제한.
- `target`: 대상 객체(프록시되는 애플리케이션 객체)가 주어진 타입의 인스턴스인 조인 포인트(스프링 AOP 사용 시 메소드 실행)로 매칭 제한.
- `args`: 인수가 주어진 타입의 인스턴스인 조인 포인트(스프링 AOP 사용 시 메소드 실행)로 매칭 제한.
- `@target`: 실행 객체의 클래스가 주어진 타입의 어노테이션을 가진 조인 포인트(스프링 AOP 사용 시 메소드 실행)로 매칭 제한.
- `@args`: 전달된 실제 인수의 런타임 타입이 주어진 타입의 어노테이션을 가진 조인 포인트(스프링 AOP 사용 시 메소드 실행)로 매칭 제한.
- `@within`: 주어진 어노테이션을 가진 타입 내의 조인 포인트로 매칭 제한 (스프링 AOP 사용 시 주어진 어노테이션을 가진 타입에 선언된 메소드 실행).
- `@annotation`: 조인 포인트의 주체(스프링 AOP에서 실행 중인 메소드)가 주어진 어노테이션을 가진 조인 포인트로 매칭 제한.

*다른 포인트컷 타입*
전체 AspectJ 포인트컷 언어는 스프링에서 지원되지 않는 추가 포인트컷 지정자를 지원합니다: `call`, `get`, `set`, `preinitialization`, `staticinitialization`, `initialization`, `handler`, `adviceexecution`, `withincode`, `cflow`, `cflowbelow`, `if`, `@this`, `@withincode`.

스프링 AOP에 의해 해석되는 포인트컷 표현식에서 이러한 포인트컷 지정자를 사용하면 `IllegalArgumentException`이 발생합니다.

*스프링 AOP에서 지원하는 포인트컷 지정자 집합은 향후 릴리스에서 더 많은 AspectJ 포인트컷 지정자를 지원하도록 확장될 수 있습니다.*

스프링 AOP는 매칭을 메소드 실행 조인 포인트로만 제한하기 때문에, 앞의 포인트컷 지정자 논의는 AspectJ 프로그래밍 가이드에서 찾을 수 있는 것보다 좁은 정의를 제공합니다.

또한 AspectJ 자체는 타입 기반 의미론을 가지며, 실행 조인 포인트에서 `this`와 `target` 모두 동일한 객체, 즉 메소드를 실행하는 객체를 참조합니다.

스프링 AOP는 프록시 기반 시스템이며 프록시 객체 자체(`this`에 바인딩됨)와 프록시 뒤의 대상 객체(`target`에 바인딩됨)를 구별합니다.

*스프링 AOP 프레임워크의 프록시 기반 특성으로 인해, 대상 객체 내의 호출은 정의상 가로채지지(intercepted) 않습니다.*

*JDK 프록시의 경우, 프록시에 대한 public 인터페이스 메소드 호출만 가로챌 수 있습니다.*

*CGLIB를 사용하면 프록시에 대한 public 및 protected 메소드 호출이 가로채집니다 (필요한 경우 패키지 가시성 메소드까지).*

*그러나 프록시를 통한 일반적인 상호 작용은 항상 public 시그니처를 통해 설계되어야 합니다.*

*포인트컷 정의는 일반적으로 가로채진 모든 메소드에 대해 매치된다는 점에 유의하십시오.*

*포인트컷이 엄격하게 public 전용으로 의도된 경우, 잠재적인 비-public 상호 작용이 있는 CGLIB 프록시 시나리오에서도 그에 따라 정의되어야 합니다.*

*가로채기 요구 사항에 대상 클래스 내의 메소드 호출이나 생성자까지 포함되는 경우, 스프링의 프록시 기반 AOP 프레임워크 대신 스프링 주도 네이티브 AspectJ 위빙 사용을 고려하세요.*

*이는 다른 특성을 가진 다른 AOP 사용 모드를 구성하므로, 결정을 내리기 전에 위빙에 대해 반드시 숙지하십시오.*

스프링 AOP는 `bean`이라는 이름의 추가 PCD도 지원합니다.

이 PCD를 사용하면 조인 포인트 매칭을 특정 이름의 스프링 빈 또는 (와일드카드 사용 시) 이름 붙여진 스프링 빈 집합으로 제한할 수 있습니다.

`bean` PCD는 다음 형식을 가집니다: `bean(idOrNameOfBean) idOrNameOfBean` 토큰은 모든 스프링 빈의 이름일 수 있습니다.

`*` 문자를 사용하는 제한된 와일드카드 지원이 제공되므로, 스프링 빈에 대한 어떤 명명 규칙을 설정하면 그것들을 선택하기 위한 `bean` PCD 표현식을 작성할 수 있습니다.

다른 포인트컷 지정자와 마찬가지로 `bean` PCD는 `&&`(and), `||`(or), `!`(negation) 연산자와 함께 사용할 수도 있습니다.

`*bean` PCD는 스프링 AOP에서만 지원되며 네이티브 AspectJ 위빙에서는 지원되지 않습니다.*

*이는 AspectJ가 정의하는 표준 PCD에 대한 스프링 특정 확장이며, 따라서 `@Aspect` 모델에서 선언된 애스펙트에는 사용할 수 없습니다.*

`*bean` PCD는 (위빙 기반 AOP가 제한되는) 타입 레벨만이 아닌 인스턴스 레벨(스프링 빈 이름 개념 기반)에서 작동합니다.*

*인스턴스 기반 포인트컷 지정자는 이름으로 특정 빈을 식별하는 것이 자연스럽고 간단한 스프링 빈 팩토리와의 긴밀한 통합과 스프링의 프록시 기반 AOP 프레임워크의 특별한 기능입니다.*

**포인트컷 표현식 결합하기 (Combining Pointcut Expressions)**

`&&`, `||`, `!`를 사용하여 포인트컷 표현식을 결합할 수 있습니다. 이름으로 포인트컷 표현식을 참조할 수도 있습니다. 다음 예제는 세 개의 포인트컷 표현식을 보여줍니다:

```java
// Java
import org.aspectj.lang.annotation.Aspect;
import org.aspectj.lang.annotation.Pointcut;

@Aspect // Need @Aspect to define pointcuts in this style
public class Pointcuts {

	@Pointcut("execution(public * *(..))") // ①
	public void publicMethod() {}

	@Pointcut("within(com.xyz.trading..*)") // ②
	public void inTrading() {}

	@Pointcut("publicMethod() && inTrading()") // ③ Combine by name
	public void tradingOperation() {}
}

```

① `publicMethod`는 메소드 실행 조인 포인트가 모든 public 메소드의 실행을 나타내는 경우 매치됩니다.
② `inTrading`은 메소드 실행이 trading 모듈 내에 있는 경우 매치됩니다.
③ `tradingOperation`은 메소드 실행이 trading 모듈의 모든 public 메소드를 나타내는 경우 매치됩니다.

```kotlin
// Kotlin
import org.aspectj.lang.annotation.Aspect
import org.aspectj.lang.annotation.Pointcut

@Aspect // Need @Aspect to define pointcuts in this style
class Pointcuts {

    @Pointcut("execution(public * *(..))") // ①
    fun publicMethod() {}

    @Pointcut("within(com.xyz.trading..*)") // ②
    fun inTrading() {}

    @Pointcut("publicMethod() && inTrading()") // ③ Combine by name
    fun tradingOperation() {}
}

```

위에서 보여준 것처럼 더 작은 이름 붙여진 포인트컷으로 더 복잡한 포인트컷 표현식을 구축하는 것이 가장 좋은 방법입니다. 이름으로 포인트컷을 참조할 때 일반적인 자바 가시성 규칙이 적용됩니다 (동일한 타입의 private 포인트컷, 계층 구조의 protected 포인트컷, 어디서든 public 포인트컷 등을 볼 수 있음). 가시성은 포인트컷 매칭에 영향을 미치지 않습니다.

**이름 붙여진 포인트컷 정의 공유하기 (Sharing Named Pointcut Definitions)**

엔터프라이즈 애플리케이션 작업 시, 개발자는 종종 여러 애스펙트 내에서 애플리케이션의 모듈과 특정 작업 집합을 참조해야 할 필요가 있습니다. 이 목적을 위해 일반적으로 사용되는 이름 붙여진 포인트컷 표현식을 캡처하는 전용 클래스를 정의하는 것이 좋습니다. 이러한 클래스는 일반적으로 다음 `CommonPointcuts` 예제와 유사합니다 (클래스 이름은 원하는 대로 지정할 수 있음):

```java
// Java
package com.xyz;

import org.aspectj.lang.annotation.Pointcut;

public class CommonPointcuts {

	/**
	 * 조인 포인트가 웹 계층에 있는 경우는 메소드가
	 * com.xyz.web 패키지 또는 그 아래 하위 패키지의 타입에 정의된 경우입니다.
	 */
	@Pointcut("within(com.xyz.web..*)")
	public void inWebLayer() {}

	/**
	 * 조인 포인트가 서비스 계층에 있는 경우는 메소드가
	 * com.xyz.service 패키지 또는 그 아래 하위 패키지의 타입에 정의된 경우입니다.
	 */
	@Pointcut("within(com.xyz.service..*)")
	public void inServiceLayer() {}

	/**
	 * 조인 포인트가 데이터 접근 계층에 있는 경우는 메소드가
	 * com.xyz.dao 패키지 또는 그 아래 하위 패키지의 타입에 정의된 경우입니다.
	 */
	@Pointcut("within(com.xyz.dao..*)")
	public void inDataAccessLayer() {}

	/**
	 * 비즈니스 서비스는 서비스 인터페이스에 정의된 모든 메소드의 실행입니다.
	 * 이 정의는 인터페이스가 "service" 패키지에 배치되고 구현 타입이
	 * 하위 패키지에 있다고 가정합니다.
	 *
	 * 서비스 인터페이스를 기능 영역별로 그룹화하는 경우 (예:
	 * com.xyz.abc.service 및 com.xyz.def.service 패키지),
	 * 대신 포인트컷 표현식 "execution(* com.xyz..service.*.*(..))"을
	 * 사용할 수 있습니다.
	 *
	 * 또는 'bean' PCD를 사용하여 표현식을 작성할 수 있습니다.
	 * 예: "bean(*Service)". (이는 스프링 서비스 빈의 이름을
	 * 일관된 방식으로 지정했다고 가정합니다.)
	 */
	@Pointcut("execution(* com.xyz..service.*.*(..))")
	public void businessService() {}

	/**
	 * 데이터 접근 작업은 DAO 인터페이스에 정의된 모든 메소드의 실행입니다.
	 * 이 정의는 인터페이스가 "dao" 패키지에 배치되고 구현 타입이
	 * 하위 패키지에 있다고 가정합니다.
	 */
	@Pointcut("execution(* com.xyz.dao.*.*(..))")
	public void dataAccessOperation() {}

}
```

```kotlin
// Kotlin
package com.xyz

import org.aspectj.lang.annotation.Pointcut

class CommonPointcuts {

	/**
	 * 조인 포인트가 웹 계층에 있는 경우는 메소드가
	 * com.xyz.web 패키지 또는 그 아래 하위 패키지의 타입에 정의된 경우입니다.
	 */
	@Pointcut("within(com.xyz.web..*)")
	fun inWebLayer() {}

	/**
	 * 조인 포인트가 서비스 계층에 있는 경우는 메소드가
	 * com.xyz.service 패키지 또는 그 아래 하위 패키지의 타입에 정의된 경우입니다.
	 */
	@Pointcut("within(com.xyz.service..*)")
	fun inServiceLayer() {}

	/**
	 * 조인 포인트가 데이터 접근 계층에 있는 경우는 메소드가
	 * com.xyz.dao 패키지 또는 그 아래 하위 패키지의 타입에 정의된 경우입니다.
	 */
	@Pointcut("within(com.xyz.dao..*)")
	fun inDataAccessLayer() {}

	/**
	 * 비즈니스 서비스는 서비스 인터페이스에 정의된 모든 메소드의 실행입니다.
	 * (설명은 Java 버전과 동일)
	 */
	@Pointcut("execution(* com.xyz..service.*.*(..))")
	fun businessService() {}

	/**
	 * 데이터 접근 작업은 DAO 인터페이스에 정의된 모든 메소드의 실행입니다.
	 * (설명은 Java 버전과 동일)
	 */
	@Pointcut("execution(* com.xyz.dao.*.*(..))")
	fun dataAccessOperation() {}

}
```

클래스의 정규화된 이름과 `@Pointcut` 메소드 이름을 결합하여 참조함으로써 포인트컷 표현식이 필요한 곳 어디든 이러한 클래스에 정의된 포인트컷을 참조할 수 있습니다.

예를 들어, 서비스 계층을 트랜잭션 가능하게 만들려면 `com.xyz.CommonPointcuts.businessService()` 이름 붙여진 포인트컷을 참조하는 다음을 작성할 수 있습니다:

```xml
<aop:config>
	<aop:advisor
		pointcut="com.xyz.CommonPointcuts.businessService()"
		advice-ref="tx-advice"/>
</aop:config>

<tx:advice id="tx-advice">
	<tx:attributes>
		<tx:method name="*" propagation="REQUIRED"/>
	</tx:attributes>
</tx:advice>
```

**예제 (Examples)**

스프링 AOP 사용자는 `execution` 포인트컷 지정자를 가장 자주 사용할 가능성이 높습니다. `execution` 표현식의 형식은 다음과 같습니다:

`execution(modifiers-pattern? ret-type-pattern declaring-type-pattern?name-pattern(param-pattern) throws-pattern?)`

반환 타입 패턴(`ret-type-pattern`), 이름 패턴, 파라미터 패턴을 제외한 모든 부분은 선택 사항입니다.

반환 타입 패턴은 조인 포인트가 매치되기 위해 메소드의 반환 타입이 무엇이어야 하는지 결정합니다.

`*`는 반환 타입 패턴으로 가장 자주 사용됩니다.

이는 모든 반환 타입을 매치합니다.

정규화된 타입 이름은 메소드가 주어진 타입을 반환할 때만 매치됩니다.

이름 패턴은 메소드 이름을 매치합니다.

이름 패턴의 전부 또는 일부로 `*` 와일드카드를 사용할 수 있습니다.

선언 타입 패턴을 지정하는 경우, 이름 패턴 구성 요소에 연결하기 위해 후행 `.`을 포함합니다.

파라미터 패턴은 약간 더 복잡합니다: `()`는 파라미터가 없는 메소드를 매치하고, `(..)`는 임의 개수(0개 이상)의 파라미터를 매치합니다.

`(*)` 패턴은 임의 타입의 파라미터 하나를 받는 메소드를 매치합니다.

`(*,String)`은 두 개의 파라미터를 받는 메소드를 매치합니다.

첫 번째는 임의 타입일 수 있지만 두 번째는 `String`이어야 합니다.

다음 예제는 몇 가지 일반적인 포인트컷 표현식을 보여줍니다:

- 모든 public 메소드 실행:
  `execution(public * *(..))`
- `set`으로 시작하는 이름의 모든 메소드 실행:
  `execution(* set*(..))`
- `AccountService` 인터페이스에 의해 정의된 모든 메소드 실행:
  `execution(* com.xyz.service.AccountService.*(..))`
- `service` 패키지에 정의된 모든 메소드 실행:
  `execution(* com.xyz.service.*.*(..))`
- `service` 패키지 또는 그 하위 패키지 중 하나에 정의된 모든 메소드 실행:
  `execution(* com.xyz.service..*.*(..))`
- `service` 패키지 내의 모든 조인 포인트 (스프링 AOP에서는 메소드 실행만):
  `within(com.xyz.service.*)`
- `service` 패키지 또는 그 하위 패키지 중 하나 내의 모든 조인 포인트 (스프링 AOP에서는 메소드 실행만):
  `within(com.xyz.service..*)`
- 프록시가 `AccountService` 인터페이스를 구현하는 모든 조인 포인트 (스프링 AOP에서는 메소드 실행만):
  `this(com.xyz.service.AccountService)*this`는 바인딩 형태로 더 일반적으로 사용됩니다. 어드바이스 본문에서 프록시 객체를 사용 가능하게 만드는 방법에 대한 섹션은 어드바이스 선언하기(Declaring Advice)를 참조하십시오.*
- 대상 객체가 `AccountService` 인터페이스를 구현하는 모든 조인 포인트 (스프링 AOP에서는 메소드 실행만):
  `target(com.xyz.service.AccountService)*target`은 바인딩 형태로 더 일반적으로 사용됩니다. 어드바이스 본문에서 대상 객체를 사용 가능하게 만드는 방법에 대한 섹션은 어드바이스 선언하기(Declaring Advice)를 참조하십시오.*
- 단일 파라미터를 받고 런타임 시 전달된 인수가 `Serializable`인 모든 조인 포인트 (스프링 AOP에서는 메소드 실행만):
  `args(java.io.Serializable)*args`는 바인딩 형태로 더 일반적으로 사용됩니다. 어드바이스 본문에서 메소드 인수를 사용 가능하게 만드는 방법에 대한 섹션은 어드바이스 선언하기(Declaring Advice)를 참조하십시오.이 예제에서 주어진 포인트컷은 `execution(* *(java.io.Serializable))`과 다르다는 점에 유의하십시오. `args` 버전은 런타임 시 전달된 인수가 `Serializable`인 경우 매치되고, `execution` 버전은 메소드 시그니처가 `Serializable` 타입의 단일 파라미터를 선언하는 경우 매치됩니다.*
- 대상 객체에 `@Transactional` 어노테이션이 있는 모든 조인 포인트 (스프링 AOP에서는 메소드 실행만):
  `@target(org.springframework.transaction.annotation.Transactional)`*바인딩 형태로 `@target`을 사용할 수도 있습니다. 어드바이스 본문에서 어노테이션 객체를 사용 가능하게 만드는 방법에 대한 섹션은 어드바이스 선언하기(Declaring Advice)를 참조하십시오.*
- 대상 객체의 선언된 타입에 `@Transactional` 어노테이션이 있는 모든 조인 포인트 (스프링 AOP에서는 메소드 실행만):
  `@within(org.springframework.transaction.annotation.Transactional)`*바인딩 형태로 `@within`을 사용할 수도 있습니다. 어드바이스 본문에서 어노테이션 객체를 사용 가능하게 만드는 방법에 대한 섹션은 어드바이스 선언하기(Declaring Advice)를 참조하십시오.*
- 실행 중인 메소드에 `@Transactional` 어노테이션이 있는 모든 조인 포인트 (스프링 AOP에서는 메소드 실행만):
  `@annotation(org.springframework.transaction.annotation.Transactional)`*바인딩 형태로 `@annotation`을 사용할 수도 있습니다. 어드바이스 본문에서 어노테이션 객체를 사용 가능하게 만드는 방법에 대한 섹션은 어드바이스 선언하기(Declaring Advice)를 참조하십시오.*
- 단일 파라미터를 받고, 전달된 인수의 런타임 타입에 `@Classified` 어노테이션이 있는 모든 조인 포인트 (스프링 AOP에서는 메소드 실행만):
  `@args(com.xyz.security.Classified)`*바인딩 형태로 `@args`를 사용할 수도 있습니다. 어드바이스 본문에서 어노테이션 객체(들)를 사용 가능하게 만드는 방법에 대한 섹션은 어드바이스 선언하기(Declaring Advice)를 참조하십시오.*
- `tradeService`라는 이름의 스프링 빈에서의 모든 조인 포인트 (스프링 AOP에서는 메소드 실행만):
  `bean(tradeService)`
- 와일드카드 표현식 `Service`와 일치하는 이름을 가진 스프링 빈에서의 모든 조인 포인트 (스프링 AOP에서는 메소드 실행만):
  `bean(*Service)`

**좋은 포인트컷 작성하기 (Writing Good Pointcuts)**

컴파일 중에 AspectJ는 매칭 성능을 최적화하기 위해 포인트컷을 처리합니다. 코드를 검사하고 각 조인 포인트가 주어진 포인트컷과 (정적 또는 동적으로) 일치하는지 확인하는 것은 비용이 많이 드는 프로세스입니다. (동적 매치는 정적 분석에서 매치를 완전히 결정할 수 없으며 코드가 실행될 때 실제 매치가 있는지 확인하기 위해 코드에 테스트가 배치됨을 의미합니다). 포인트컷 선언을 처음 접할 때 AspectJ는 매칭 프로세스를 위한 최적 형태로 재작성합니다. 이것은 무엇을 의미할까요? 기본적으로 포인트컷은 DNF(Disjunctive Normal Form)로 재작성되고 포인트컷의 구성 요소는 평가 비용이 저렴한 구성 요소가 먼저 확인되도록 정렬됩니다. 이는 다양한 포인트컷 지정자의 성능을 이해하는 것에 대해 걱정할 필요가 없으며 포인트컷 선언에서 어떤 순서로든 제공할 수 있음을 의미합니다.

그러나 AspectJ는 알려준 것만으로 작업할 수 있습니다. 최적의 매칭 성능을 위해 무엇을 달성하려고 하는지 생각하고 정의에서 매치 검색 공간을 가능한 한 좁혀야 합니다. 기존 지정자는 자연스럽게 세 그룹 중 하나로 분류됩니다: 종류 지정(kinded), 범위 지정(scoping), 문맥 지정(contextual):

- **종류 지정(Kinded) 지정자**는 특정 종류의 조인 포인트를 선택합니다: `execution`, `get`, `set`, `call`, `handler`.
- **범위 지정(Scoping) 지정자**는 관심 있는 조인 포인트 그룹(아마도 여러 종류)을 선택합니다: `within`, `withincode`.
- **문맥 지정(Contextual) 지정자**는 컨텍스트를 기반으로 매치(및 선택적으로 바인딩)합니다: `this`, `target`, `@annotation`.

잘 작성된 포인트컷은 최소한 처음 두 가지 타입(종류 지정 및 범위 지정)을 포함해야 합니다. 조인 포인트 컨텍스트를 기반으로 매치하거나 어드바이스에서 사용하기 위해 해당 컨텍스트를 바인딩하기 위해 문맥 지정 지정자를 포함할 수 있습니다. 종류 지정 지정자만 제공하거나 문맥 지정 지정자만 제공하는 것은 작동하지만, 추가 처리 및 분석으로 인해 위빙 성능(사용된 시간 및 메모리)에 영향을 미칠 수 있습니다. 범위 지정 지정자는 매치 속도가 매우 빠르며, 이를 사용하면 AspectJ가 더 이상 처리해서는 안 되는 조인 포인트 그룹을 매우 빠르게 기각(dismiss)할 수 있음을 의미합니다. 좋은 포인트컷은 가능하다면 항상 하나를 포함해야 합니다.

---

**`execution` 지정자의 기본 구조:**

`execution( [수식어 패턴]? [반환 타입 패턴] [선언 타입 패턴]?.[메소드 이름 패턴]([파라미터 패턴]) [예외 패턴]? )`

- `?`가 붙은 부분은 **선택적(Optional)** 으로 사용할 수 있다는 의미입니다.
- **띄어쓰기:** 각 패턴 사이는 **띄어쓰기**로 구분합니다. (매우 중요!)

**각 부분 상세 설명:**

1. **`수식어 패턴 (modifiers-pattern?)` (선택 사항):**
  - 메소드의 접근 제한자(modifier)를 지정합니다.
  - `public`, `private`, `protected` 등을 사용할 수 있습니다.
  - 와일드카드는 사용할 수 **없습니다**. (예: `public *` 은 안 됨)
  - 생략하면 모든 접근 제한자를 의미합니다.
  - **예시:**
    - `public`: public 메소드만
    - (생략): 모든 접근 제한자
2. **`반환 타입 패턴 (ret-type-pattern)` (필수):**
  - 메소드가 반환하는 타입 패턴을 지정합니다.
  - : 모든 반환 타입을 의미합니다. (가장 흔하게 사용됨)
  - `void`: 반환 타입이 없는 메소드.
  - `String`: 반환 타입이 정확히 `String`인 메소드.
  - `com.example.MyClass`: 반환 타입이 `com.example.MyClass`인 메소드.
  - `com.example.*`: `com.example` 패키지의 모든 타입을 반환하는 메소드.
  - `com.example..*`: `com.example` 패키지 및 모든 하위 패키지의 타입을 반환하는 메소드 (`..` 사용).
  - **예시:**
    - : 모든 반환 타입 매칭
    - `void`: void 반환 타입 매칭
    - `java.lang.String`: String 반환 타입 매칭
3. **`선언 타입 패턴 (declaring-type-pattern?)` (선택 사항):**
  - 메소드가 **선언된 클래스 또는 인터페이스**의 패턴을 지정합니다. 패키지 경로를 포함할 수 있습니다.
  - 이 부분을 생략하면 모든 클래스/인터페이스를 의미합니다.
  - 패턴 뒤에는 반드시 **점(`.`)** 을 찍어서 메소드 이름 패턴과 구분해야 합니다.
  - 와일드카드를 사용할 수 있습니다:
    - : 이름에서 임의의 문자열 부분 (예: `Service`)
    - `..`: 0개 이상의 패키지 레벨 (예: `com.example..service.*`)
  - **예시:**
    - `com.example.service.MyService`: `MyService` 클래스에 선언된 메소드.
    - `com.example.service.*`: `com.example.service` 패키지의 모든 타입에 선언된 메소드.
    - `com.example.service..*`: `com.example.service` 패키지 및 모든 하위 패키지의 모든 타입에 선언된 메소드.
    - `..MyService`: 이름이 `MyService`로 끝나는 모든 패키지의 `MyService` 클래스.
    - (생략 시): 모든 타입.
4. **`메소드 이름 패턴 (name-pattern)` (필수):**
  - 매칭할 **메소드의 이름** 패턴을 지정합니다.
  - 와일드카드를 사용할 수 있습니다.
  - **예시:**
    - `myMethod`: 이름이 정확히 "myMethod"인 메소드.
    - `set*`: 이름이 "set"으로 시작하는 모든 메소드 (세터 메소드).
    - : 모든 이름의 메소드.
    - `ByName`: 이름이 "ByName"으로 끝나는 모든 메소드.
5. **`파라미터 패턴 (param-pattern)` (필수):**
  - 메소드가 받는 **파라미터(인수)의 타입 패턴**을 괄호 `()` 안에 지정합니다.
  - **`()`**: 파라미터가 **없는** 메소드.
  - **`(..)`**: **0개 이상의 모든 타입**의 파라미터를 의미합니다. (가장 흔하게 사용됨)
  - **`(*)`**: **정확히 하나의 파라미터**를 가지며, 그 파라미터의 타입은 **무엇이든 상관없음**을 의미합니다.
  - **`(String)`**: 정확히 하나의 파라미터를 가지며, 그 타입이 `String`이어야 함을 의미합니다.
  - **`(String, *)`**: 정확히 두 개의 파라미터를 가지며, 첫 번째는 `String` 타입, 두 번째는 **임의의 타입**이어야 함을 의미합니다.
  - **`(com.example.User, ..)`**: 첫 번째 파라미터는 `com.example.User` 타입이고, 그 뒤에는 **0개 이상의 모든 타입**의 파라미터가 올 수 있음을 의미합니다.
  - **타입 패턴 사용:** 파라미터 타입에도 와일드카드(, `..`)를 사용할 수 있습니다. (예: `(com.example..*, int)`)
6. **`예외 패턴 (throws-pattern?)` (선택 사항):**
  - 메소드가 던질 수 있는(선언된) **예외(Exception)** 의 타입 패턴을 지정합니다.
  - `throws ExceptionTypePattern` 형식으로 사용합니다.
  - 일반적으로 잘 사용되지 않습니다.

**예시 해석:**

- **`execution(public * *(..))`**
  - `public`: 접근 제한자가 public인
  - : 모든 반환 타입의
  - (선언 타입 생략): 모든 클래스에 선언된
  - : 모든 이름의
  - `(..)`: 0개 이상의 모든 파라미터를 갖는 메소드 실행
  - **=> 모든 public 메소드 실행**
- **`execution(* set*(..))`**
  - (수식어 생략): 모든 접근 제한자의
  - : 모든 반환 타입의
  - (선언 타입 생략): 모든 클래스에 선언된
  - `set*`: 이름이 "set"으로 시작하는
  - `(..)`: 0개 이상의 모든 파라미터를 갖는 메소드 실행
  - **=> 모든 세터(setter) 메소드 실행**
- **`execution(* com.xyz.service.AccountService.*(..))`**
  - (수식어 생략): 모든 접근 제한자의
  - : 모든 반환 타입의
  - `com.xyz.service.AccountService`: `AccountService` 인터페이스(또는 클래스)에 선언된
  - `.*`: 모든 이름의
  - `(..)`: 0개 이상의 모든 파라미터를 갖는 메소드 실행
  - **=> `AccountService` 인터페이스/클래스에 정의된 모든 메소드 실행**
- **`execution(* com.xyz.service..*.*(..))`**
  - (수식어 생략): 모든 접근 제한자의
  - : 모든 반환 타입의
  - `com.xyz.service..*`: `com.xyz.service` 패키지 및 그 하위 패키지의 모든 타입에 선언된
  - `.*`: 모든 이름의
  - `(..)`: 0개 이상의 모든 파라미터를 갖는 메소드 실행
  - **=> `service` 패키지 및 하위 패키지의 모든 메소드 실행**

**핵심 와일드카드 및 기호:**

- **(별표):** 이름(메소드명, 타입명 일부, 패키지명 일부)이나 반환 타입, 단일 파라미터 타입에서 **"모든 것" 또는 "임의의 문자열"** 을 의미합니다.
- **`..` (점 두 개):**
  - **패키지 경로**에서 사용될 때: **0개 이상의 하위 패키지**를 의미합니다. (예: `com.example..`)
  - **파라미터 패턴**에서 사용될 때: **0개 이상의 모든 타입**의 파라미터를 의미합니다. (예: `(..)`)
- **띄어쓰기:** 각 패턴(수식어, 반환타입, 선언타입.메소드명, 파라미터, 예외)을 구분하는 중요한 구분자입니다.
- **`.` (점):** 선언 타입 패턴과 메소드 이름 패턴을 구분합니다.
- **`()` (괄호):** 파라미터 패턴을 감쌉니다.

이 설명을 통해 포인트컷 표현식의 구조와 와일드카드 사용법이 좀 더 명확하게 이해되셨기를 바랍니다!
