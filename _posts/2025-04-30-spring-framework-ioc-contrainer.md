---
title: Spring Framework IoC Container
description: 
author: laze
date: 2025-04-30 00:00:03 +0900
categories: [Dev, SpringBoot]
tags: [SpringBoot]
---
# IoC Container

제어의 역전(IoC) 원칙에 대한 스프링 프레임워크의 구현

의존성 주입(DI)은 IoC의 특수한 형태로 객체들이 자신의 의존성(즉, 함께 작동하는 다른 객체들)을  생성자 인수, 팩토리 메소드의 인수, 또는 객체 인스턴스가 생성되거나

팩토리 메소드에서 반환된 후 설정되는 속성들을 통해서만 정의하는 방식입니다.

그러면 IoC 컨테이너는 빈(bean)을 생성할 때 해당 의존성들을 주입합니다.

이 과정은 빈 자체가 클래스의 직접 생성이나 서비스 로케이터(Service Locator) 패턴과 같은 메커니즘을 사용하여 자신의 의존성을 인스턴스화하거나 위치를 찾는 것을 제어하는 방식과는

근본적으로 반대(역전, 그래서 제어의 역전이라는 이름이 붙음)입니다.

org.springframework.beans와 org.springframework.context 패키지는 스프링 프레임워크의 IoC 컨테이너의 기반입니다. 

BeanFactory 인터페이스는 모든 유형의 객체를 관리할 수 있는 고급 설정 메커니즘을 제공합니다. 

ApplicationContext는 BeanFactory의 하위 인터페이스입니다. 이는 다음을 추가합니다:

- 스프링의 AOP 기능과의 더 쉬운 통합
- 메시지 리소스 처리(국제화에 사용)
- 이벤트 발행
- 웹 애플리케이션에서 사용하기 위한 WebApplicationContext와 같은 애플리케이션 계층 특정 컨텍스트

요약하자면, BeanFactory는 설정 프레임워크와 기본 기능을 제공하고, ApplicationContext는 더 많은 엔터프라이즈 관련 기능을 추가합니다. 

스프링에서, 애플리케이션의 중추를 형성하고 스프링 IoC 컨테이너에 의해 관리되는 객체들을 빈(beans)이라고 부릅니다.

빈(Bean)은 스프링 IoC 컨테이너에 의해 인스턴스화되고, 조립되며, 관리되는 객체입니다.

(컨테이너에 의해 관리되지 않는다면) 빈은 단순히 애플리케이션에 있는 많은 객체 중 하나일 뿐입니다.

빈들과 그들 사이의 의존성은 컨테이너가 사용하는 설정 메타데이터에 반영됩니다.
