---
title: Defining New Advice Types
description: 
author: laze
date: 2025-05-06 00:00:12 +0900
categories: [Dev, SpringBoot]
tags: [SpringBoot]
---
**Defining New Advice Types**

스프링 AOP는 확장 가능하도록 설계되었습니다.

현재 내부적으로는 가로채기(interception) 구현 전략이 사용되지만, 가로채기 around 어드바이스, before, throws 어드바이스, after returning 어드바이스 외에도 임의의 어드바이스 타입을 지원하는 것이 가능합니다.

`org.springframework.aop.framework.adapter` 패키지는 핵심 프레임워크를 변경하지 않고 새로운 커스텀 어드바이스 타입에 대한 지원을 추가할 수 있게 해주는 SPI 패키지입니다.

커스텀 `Advice` 타입에 대한 유일한 제약 조건은 `org.aopalliance.aop.Advice` 마커(marker) 인터페이스를 구현해야 한다는 것입니다.

---

**전체 주제: 새로운 어드바이스 타입 정의하기 (Defining New Advice Types)**

이 부분은 스프링 AOP가 얼마나 유연하고 확장 가능한지를 보여주는 내용입니다. 개발자는 스프링이 미리 정의해놓은 어드바이스 타입들 외에도, **자신만의 독자적인 어드바이스 실행 시점이나 방식을 정의**하고 이를 스프링 AOP 프레임워크와 연동시킬 수 있습니다.

**핵심 아이디어:** 스프링 AOP는 정해진 틀만 제공하는 것이 아니다! 필요하다면 완전히 새로운 방식의 부가 기능(어드바이스) 타입을 만들어서 스프링 AOP 시스템에 끼워 넣을 수 있다!

---

**1. 스프링 AOP의 확장성:**

- 스프링 AOP는 내부적으로 **메소드 가로채기(Method Interception)** 를 기반으로 동작하지만(주로 Around 어드바이스 방식), 아키텍처 자체는 특정 구현 방식에 얽매이지 않고 **확장 가능**하도록 설계되었습니다.
- 이는 즉, 개발자가 스프링이 제공하는 Before, After Returning, After Throwing, Around 외에 **완전히 새로운 개념의 어드바이스 타입**을 정의하고 사용할 수 있다는 의미입니다. (예: 'Before Validation Advice', 'After Commit Advice' 등 상상할 수 있는 모든 종류)

---

**2. 확장 메커니즘 (`org.springframework.aop.framework.adapter` 패키지):**

- 스프링은 새로운 어드바이스 타입을 추가하기 위한 **SPI(Service Provider Interface)** 를 `org.springframework.aop.framework.adapter` 패키지를 통해 제공합니다.
- 이 패키지의 클래스들(특히 `AdvisorAdapter` 인터페이스와 그 구현체들)은 **커스텀 어드바이스 타입을 스프링 AOP의 내부 인터셉터 체인 메커니즘과 연결**해주는 **어댑터(Adapter)** 역할을 합니다.
- 개발자는 자신만의 커스텀 어드바이스 타입을 만들고, 이 어드바이스 타입을 스프링 AOP 프레임워크가 이해하고 실행할 수 있도록 변환해주는 **`AdvisorAdapter` 구현체**를 함께 만들어서 등록하면 됩니다.

---

**3. 커스텀 어드바이스 타입의 최소 요구사항:**

- 개발자가 정의하는 새로운 커스텀 어드바이스 타입(클래스 또는 인터페이스)은 딱 한 가지 요구사항만 만족하면 됩니다: 바로 **`org.aopalliance.aop.Advice`** 라는 **마커(Marker) 인터페이스**를 구현(또는 상속)해야 한다는 것입니다.
- `Advice` 인터페이스는 아무 메소드도 가지고 있지 않으며, 단순히 "이것은 AOP 어드바이스의 한 종류이다" 라고 표시하는 역할만 합니다.
- 이 마커 인터페이스를 구현함으로써, 스프링 AOP 프레임워크는 해당 객체를 어드바이스로 인식하고 관련 처리(예: `AdvisorAdapter` 찾기)를 시도할 수 있게 됩니다.

**결론:**

스프링 AOP는 매우 유연하여, 기본 제공되는 어드바이스 타입 외에 **개발자가 정의한 완전히 새로운 종류의 어드바이스 타입을 시스템에 추가**할 수 있습니다. 이를 위해서는 커스텀 어드바이스 타입이 `org.aopalliance.aop.Advice` 마커 인터페이스를 구현해야 하며, 해당 커스텀 어드바이스를 스프링의 내부 인터셉터 메커니즘과 연결해주는 **`AdvisorAdapter` 구현체를 만들어 등록**해야 합니다. 이것은 스프링 프레임워크 자체를 확장하는 매우 고급 기능에 해당합니다.
