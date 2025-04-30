---
title: Spring Framework Architecture
description: 
author: laze
date: 2025-04-30 00:00:01 +0900
categories: [Dev, SpringBoot]
tags: [SpringBoot]
---
# Spring Framework Architecture

**코어 컨테이너 (Core Container)**

코어 컨테이너는 코어(Core), 빈(Beans), 컨텍스트(Context), 표현 언어(Expression Language) 모듈로 구성되며, 자세한 내용은 다음과 같습니다:

- **코어(Core) 모듈:** IoC와 의존성 주입(Dependency Injection) 기능을 포함하여 프레임워크의 기본적인 부분들을 제공합니다.
- **빈(Bean) 모듈:** 팩토리 패턴(factory pattern)의 정교한 구현체인 BeanFactory를 제공합니다.
- **컨텍스트(Context) 모듈:** 코어(Core) 및 빈(Beans) 모듈이 제공하는 견고한 기반 위에 구축되며, 정의되고 설정된 모든 객체에 접근하기 위한 수단입니다. ApplicationContext 인터페이스가 컨텍스트 모듈의 핵심입니다.
- **SpEL 모듈:** 런타임 시 객체 그래프(object graph)를 조회하고 조작하기 위한 강력한 표현 언어(expression language)를 제공합니다.

**데이터 접근/통합 (Data Access/Integration)**

데이터 접근/통합 계층은 JDBC, ORM, OXM, JMS, 트랜잭션(Transaction) 모듈로 구성되며, 자세한 내용은 다음과 같습니다:

- **JDBC 모듈:** 지루한 JDBC 관련 코딩의 필요성을 제거하는 JDBC 추상화 계층(JDBC-abstraction layer)을 제공합니다.
- **ORM 모듈:** JPA, JDO, Hibernate, iBatis를 포함한 인기 있는 객체-관계 매핑(object-relational mapping) API들을 위한 통합 계층(integration layers)을 제공합니다.
- **OXM 모듈:** JAXB, Castor, XMLBeans, JiBX, XStream을 위한 객체/XML 매핑(Object/XML mapping) 구현을 지원하는 추상화 계층(abstraction layer)을 제공합니다.
- **자바 메시징 서비스(Java Messaging Service) JMS 모듈:** 메시지를 생산(producing)하고 소비(consuming)하는 기능을 포함합니다.
- **트랜잭션(Transaction) 모듈:** 특별한 인터페이스를 구현하는 클래스들과 모든 POJO들을 위한 프로그래밍 방식 및 선언적 방식의 트랜잭션 관리를 지원합니다.

**웹 (Web)**

웹 계층은 웹(Web), 웹-MVC(Web-MVC), 웹소켓(Web-Socket), 웹-포틀릿(Web-Portlet) 모듈로 구성되며, 자세한 내용은 다음과 같습니다:

- **웹(Web) 모듈:** 멀티파트 파일 업로드(multipart file-upload) 기능, 서블릿 리스너(servlet listeners)를 사용한 IoC 컨테이너 초기화, 웹 지향 애플리케이션 컨텍스트(web-oriented application context)와 같은 기본적인 웹 중심의 통합 기능을 제공합니다.
- **웹-MVC(Web-MVC) 모듈:** 웹 애플리케이션을 위한 스프링의 모델-뷰-컨트롤러(Model-View-Controller, MVC) 구현을 포함합니다.
- **웹소켓(Web-Socket) 모듈:** 웹 애플리케이션에서 클라이언트와 서버 간의 웹소켓(WebSocket) 기반 양방향 통신(two-way communication)을 지원합니다.
- **웹-포틀릿(Web-Portlet) 모듈:** 포틀릿(portlet) 환경에서 사용될 MVC 구현을 제공하며, 웹-서블릿(Web-Servlet) 모듈의 기능을 반영(mirrors)합니다.

**기타 (Miscellaneous)**

그 외 AOP, Aspects, Instrumentation, Messaging, Test 모듈과 같은 몇 가지 다른 중요한 모듈들이 있으며, 자세한 내용은 다음과 같습니다:

- **AOP 모듈:** 관점 지향 프로그래밍 구현을 제공하여, 분리되어야 할 기능을 구현하는 코드를 깔끔하게 분리하기 위해 메소드 가로채기(method-interceptors)와 포인트컷(pointcuts)을 정의할 수 있게 해줍니다.
- **Aspects 모듈:** 강력하고 성숙한 AOP 프레임워크인 AspectJ와의 통합을 제공합니다.
- **Instrumentation 모듈:** 특정 애플리케이션 서버에서 사용될 클래스 계측(class instrumentation) 지원과 클래스 로더(class loader) 구현을 제공합니다.
- **메시징(Messaging) 모듈:** 애플리케이션에서 사용할 웹소켓 하위 프로토콜(WebSocket sub-protocol)로서 STOMP를 지원합니다. 또한 웹소켓 클라이언트로부터 오는 STOMP 메시지를 라우팅하고 처리하기 위한 어노테이션 기반 프로그래밍 모델도 지원합니다.
- **테스트(Test) 모듈:** JUnit이나 TestNG 프레임워크를 사용하여 스프링 컴포넌트 테스트를 지원합니다.
