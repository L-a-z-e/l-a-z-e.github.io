---
title: Spring Webflux 시작하기 - Servlet Stack부터 Reactor까지
description: 
author: laze
date: 2025-07-25 00:00:05 +0900
categories: [Dev, SpringWebflux]
tags: [SpringWebflux, Reactive, Mono, Flux]
---
# [개념 정리] Spring Webflux 시작하기: Servlet Stack부터 Reactor까지

## 1. 서론: 왜 Spring은 MVC에 만족하지 않고 Webflux를 만들었을까?

수년 동안 Spring MVC는 Java 웹 개발의 표준이었습니다.

서블릿(Servlet) 기술을 기반으로 한 이 프레임워크는 '요청 당 스레드(Thread-per-Request)'라는 직관적인 모델을 통해 안정적이고 생산적인 개발 환경을 제공해왔습니다.

하지만 수많은 마이크로서비스가 서로 통신하고, 실시간 스트리밍과 대규모 동시 접속 처리가 필수가 된 현대적인 웹 환경에서, 기존 방식은 한계에 부딪히기 시작했습니다.

바로 **I/O 작업으로 인한 스레드 블로킹(Blocking)** 문제 때문입니다.

Spring Webflux는 이러한 한계를 극복하기 위해 등장한, 완전히 새로운 패러다임의 **'비동기(Asynchronous), 논블로킹(Non-Blocking)'** 리액티브(Reactive) 웹 프레임워크입니다.

이 글에서는 Webflux를 이해하기 위한 첫걸음으로, 전통적인 Servlet Stack과 새로운 Reactive Stack의 근본적인 차이점부터 그 핵심 엔진인 Project Reactor까지 알아보겠습니다.

## 2. 두 개의 세계: Servlet Stack vs Reactive Stack

Spring 5부터, 개발자는 애플리케이션의 성격에 따라 두 가지 웹 스택 중 하나를 선택할 수 있게 되었습니다.

| 구분 | **Servlet Stack (Spring MVC)** | **Reactive Stack (Spring Webflux)** |
| --- | --- | --- |
| **기반 모델** | Thread-per-Request (요청 당 스레드) | Event Loop (이벤트 루프) |
| **핵심 서버** | **Tomcat**, Jetty (서블릿 컨테이너) | **Netty**, Undertow |
| **I/O 모델** | **Blocking I/O** (동기-블로킹) | **Non-Blocking I/O** (비동기-논블로킹) |
| **주요 의존성** | `spring-boot-starter-web` | `spring-boot-starter-webflux` |
| **데이터 접근** | **JDBC, JPA** (Blocking) | **R2DBC**, Reactive Mongo/Redis (Non-Blocking) |
| **적합한 상황** | - 대부분의 전통적인 웹 애플리케이션
- **JPA 사용이 필수적인 경우**
- 코드가 직관적이고 개발 속도가 중요할 때 | - **높은 동시성 처리**가 필요한 경우
- 다수의 외부 API 호출 등 I/O 작업이 지배적일 때
- 스트리밍, 실시간 통신 |

두 스택은 **함께 사용할 수 없으며**, 프로젝트 시작 시 하나를 선택해야 합니다.

## 3. Reactive Stack의 심장: Netty와 이벤트 루프

Servlet Stack의 '요청 하나당 스레드 하나' 모델은, 스레드가 DB 응답을 기다리는 동안 아무 일도 하지 않고 자원만 차지하는 비효율을 낳습니다.

Reactive Stack은 이 문제를 해결하기 위해 **이벤트 루프(Event Loop)** 라는 완전히 다른 모델을 사용합니다.

- **비유:** 한 명의 유능한 보안 요원(이벤트 루프 스레드)이 수백 개의 CCTV 모니터(네트워크 연결)를 동시에 감시하는 상황을 상상해 봅시다. 요원은 모든 모니터를 계속 쳐다보지 않습니다. 대신, 특정 모니터에서 **움직임(I/O 이벤트)**이 감지될 때만 그쪽으로 가서 처리하고, 다시 전체를 감시하는 상태로 돌아갑니다.

이벤트 루프 모델은 이와 같습니다.

1. 클라이언트 요청이 들어오면, **이벤트 루프 스레드**는 이 요청을 **이벤트 큐(Event Queue)**에 등록하고 즉시 다른 일을 하러 갑니다 (Non-Blocking).
2. DB 조회나 외부 API 호출 같은 I/O 작업은 OS 커널이나 별도의 워커 스레드에 위임하고, "작업이 끝나면 알려줘(Callback)"라고 약속합니다.
3. I/O 작업이 완료되면 '완료 이벤트'가 큐에 등록되고, 이벤트 루프 스레드는 이 이벤트를 감지하여 후속 처리(Response 전송 등)를 수행합니다.

이 모델 덕분에, **소수의 고정된 스레드(보통 CPU 코어 개수만큼)만으로 수만 개의 동시 연결을 효율적으로 처리**할 수 있게 됩니다. 스레드는 더 이상 기다리지 않고, 항상 '일하는' 상태를 유지합니다.

## 4. 비동기 데이터 흐름의 표준: Reactive Streams

이러한 비동기, 논블로킹 데이터 흐름을 코드 수준에서 어떻게 우아하게 표현하고 제어할 수 있을까요?

이를 위해 등장한 것이 바로 **Reactive Streams**라는 표준 사양(Specification)입니다.

이 표준은 4개의 핵심 인터페이스로 구성됩니다.

- **`Publisher<T>` (발행자):** 데이터 스트림을 생성하고 발행하는 주체입니다. (e.g., DB 조회 결과, 외부 API 응답 스트림)
- **`Subscriber<T>` (구독자):** `Publisher`가 발행하는 데이터를 소비(처리)하는 주체입니다.
- **`Subscription` (구독 정보):** `Publisher`와 `Subscriber` 사이의 연결고리. 데이터 요청 개수 등을 제어하는 통로입니다.
- **`Processor<T, R>` (처리자):** `Publisher`이면서 동시에 `Subscriber`인 중간 처리자. 데이터를 받아서 가공(`map`, `filter` 등)한 뒤 다시 발행합니다.

데이터는 `Publisher`에서 시작하여 여러 `Processor`를 거쳐 최종적으로 `Subscriber`에게 전달되는, 하나의 거대한 파이프라인을 통해 흐르게 됩니다.

## 5. 핵심 기능: Back Pressure - "넘치지 않게, 감당할 만큼만!"

Reactive Streams의 가장 중요한 특징 중 하나는 **'Back Pressure'**입니다.

- **문제 상황:** `Publisher`(발행자)가 초당 1,000개의 데이터를 쏟아내는데, `Subscriber`(구독자)는 초당 100개밖에 처리하지 못한다면? `Subscriber`는 처리하지 못한 데이터에 압도되어 결국 시스템 장애로 이어질 수 있습니다.
- **해결책 (Back Pressure):** 데이터 흐름의 주도권을 **`Subscriber`(소비자)가 갖습니다.** `Subscriber`는 자신이 처리할 수 있는 만큼만 `Subscription`을 통해 `Publisher`에게 **"데이터 N개만 요청(request)"**합니다. `Publisher`는 요청받은 만큼만 데이터를 전달하고, 다음 요청이 올 때까지 기다립니다.

이 메커니즘 덕분에, 소비자의 처리 속도에 맞춰 데이터 흐름이 자동으로 조절되므로, 시스템이 안정적으로 유지될 수 있습니다.

## 6. Spring의 선택: Project Reactor (`Mono`와 `Flux`)

Reactive Streams는 표준 '인터페이스'일 뿐, 실제 구현체가 필요합니다. Spring Webflux는 여러 구현체 중 **Project Reactor**를 표준 라이브러리로 채택했습니다.

Reactor는 Reactive Streams의 `Publisher`를 두 가지 구체적인 타입으로 제공합니다.

### `Mono<T>` (0-1개)

**0 또는 1개**의 데이터 아이템을 처리하기 위한 `Publisher` 구현체입니다. 비동기 작업의 결과가 단 하나(또는 없을 경우)일 때 사용합니다.

- **주요 사용처:**
  - ID로 단 건 조회: `findById(id)`
  - 데이터 생성/수정/삭제 후 결과 반환
  - HTTP 요청에 대한 최종 응답 객체

### `Flux<T>` (0-N개)

**0개 이상의 (N개의) 데이터 아이템 스트림**을 처리하기 위한 `Publisher` 구현체입니다.

여러 개의 데이터가 연속적으로 발생하는 스트림을 표현할 때 사용합니다.

- **주요 사용처:**
  - 목록 조회: `findAll()`
  - 실시간 데이터 스트리밍 (e.g., 주식 시세, 채팅 메시지)
  - 파일 처리

우리가 Webflux 컨트롤러에서 `Mono`나 `Flux`를 반환하는 것은, "이 메서드는 비동기적인 데이터 스트림을 생성하는 발행자(Publisher)입니다. 관심 있는 구독자(Subscriber)는 구독해서 데이터를 받아가세요" 라고 선언하는 것과 같습니다.

```java
@RestController
public class UserController {

    private final UserRepository userRepository;

    // ... 생성자 ...

    // ID로 사용자 한 명을 찾아 Mono<User>로 반환
    @GetMapping("/users/{id}")
    public Mono<User> getUserById(@PathVariable String id) {
        return userRepository.findById(id);
    }

    // 모든 사용자 목록을 Flux<User> 스트림으로 반환
    @GetMapping("/users")
    public Flux<User> getAllUsers() {
        return userRepository.findAll();
    }
}
```

## 7. 결론: 패러다임의 전환을 이해하기

Spring Webflux는 단순히 Spring MVC의 더 빠른 버전이 아닙니다. 그것은 웹 요청을 처리하는 근본적인 패러다임의 전환입니다.

- **Blocking → Non-Blocking:** 스레드가 기다리지 않고, 이벤트 기반으로 동작합니다.
- **명령형 → 선언형/함수형:** `for` 루프 대신, `map`, `filter`, `flatMap`과 같은 연산자를 사용하여 데이터 파이프라인을 선언적으로 구성합니다.

이러한 패러다임의 전환을 이해하는 것이 Webflux를 제대로 사용하는 첫걸음입니다. 모든 시스템에 Webflux가 정답은 아니지만, 높은 동시성과 효율적인 리소스 사용이 중요한 현대적인 마이크로서비스 환경에서 Webflux는 강력한 무기가 되어줄 것입니다.
