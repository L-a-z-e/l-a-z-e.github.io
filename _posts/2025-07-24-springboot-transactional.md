---
title: Transactional(readOnly = true) 동일 테이블 조회 시 성능이 저하되는 이유
description: 
author: laze
date: 2025-07-24 00:00:01 +0900
categories: [Dev, SpringBoot]
tags: [SpringBoot, Transactional, readOnly, DBConnection]
---
# `@Transactional(readOnly = true)`의 함정: 동일 테이블 조회 시 성능이 저하되는 이유

## 1. 서론: "읽기 전용인데 왜 느려지죠?" - 미스터리한 성능 저하 현상

Spring 기반의 애플리케이션에서 데이터 조회 성능을 최적화하기 위해 우리는 흔히 서비스 메서드에 `@Transactional(readOnly = true)` 어노테이션을 사용한다. 그런데 가끔 이 어노테이션의 상식을 벗어나는 것처럼 보이는 미스터리한 현상을 마주칠 때가 있다.

> 문제 상황:@Transactional(readOnly = true)가 적용된 하나의 서비스 메서드 안에서 여러 개의 조회(SELECT) 쿼리를 실행한다. 각 쿼리가 서로 다른 테이블을 조회할 때는 매우 빠른데, 동일한 테이블을 조회하는 쿼리가 두 번 이상 포함되면 갑자기 전체 메서드의 실행 시간이 눈에 띄게 느려진다. 더 이상한 점은, 이 느려지는 로직을 별도의 Controller/Service로 분리하여 호출하면 다시 빨라진다는 것이다.
>

이는 단순한 쿼리 튜닝의 문제가 아니다. 이 현상의 이면에는 **Spring의 트랜잭션 관리 방식, DB 커넥션 풀, 그리고 데이터베이스의 내부 동작 원리**가 복합적으로 얽혀 있다. 이 글에서는 이 미스터리한 성능 저하의 원인을 심층적으로 분석하고, 근본적인 해결책을 찾아본다.

## 2. `@Transactional(readOnly = true)`의 오해와 진실

먼저 이 어노테이션이 '마법의 성능 향상 스위치'가 아니라는 점을 명확히 해야 한다. `readOnly = true`의 진짜 역할은 다음과 같다.

- **JPA/Hibernate에서의 효과:** 영속성 컨텍스트의 Dirty Checking(변경 감지)을 생략하여 성능을 향상시킨다. **(MyBatis 환경에서는 이 효과가 없다.)**
- **DB 드라이버 레벨에서의 효과:** DB 커넥션에 "이 트랜잭션은 읽기 전용입니다"라는 힌트를 전달한다. 일부 DB(특히 Master/Slave 구조)는 이 힌트를 보고 읽기 작업을 부하가 적은 Slave DB로 보내는 등의 최적화를 수행할 수 있다.

**가장 중요한 오해:** `readOnly = true`가 DB의 락(Lock) 발생을 막아주는 것은 아니다. `SELECT` 쿼리라 할지라도, 데이터베이스의 트랜잭션 격리 수준(Isolation Level)이나 쿼리 종류에 따라 공유 락(Shared Lock) 등이 발생할 수 있다. 문제의 원인은 더 깊은 곳에 있다.

## 3. 가장 유력한 용의자: '동일 커넥션' 내에서의 암묵적 리소스 경합

이 현상의 가장 가능성 높은 원인은, **하나의 트랜잭션(하나의 DB 커넥션) 내에서 동일한 리소스(테이블)에 반복적으로 접근할 때 발생하는 DB 레벨의 미세한 경합이나 대기 현상**이다.

Spring 트랜잭션의 기본 동작 원리를 따라가 보자.

1. **하나의 서비스 = 하나의 트랜잭션 = 하나의 DB 커넥션:** `@Transactional`이 붙은 서비스 메서드가 호출되면, Spring은 커넥션 풀에서 DB 커넥션을 하나 빌려와 트랜잭션을 시작한다. 이 메서드가 종료될 때까지, 그 안에서 실행되는 **모든 DB 작업은 이 동일한 커넥션을 재사용한다.**
2. **첫 번째 쿼리 실행:** `mapperA.selectFromTableX()`가 호출되고, 이 커넥션을 통해 쿼리가 DB로 전송된다. DB는 쿼리를 실행하고 결과를 반환하기 시작한다.
3. **두 번째 쿼리 실행:** 첫 번째 쿼리의 결과 처리가 완전히 끝나기 전에, **동일한 커넥션**을 통해 `mapperB.selectFromTableX()`가 DB로 전송된다.
4. **병목 발생 지점:**
  - **DB 레벨:** DB는 동일한 테이블에 대해 동일한 세션(커넥션)에서 두 개의 요청이 거의 동시에 들어왔을 때, 데이터 일관성을 보장하기 위해 내부적으로 매우 짧은 시간 동안 **암묵적인 래치(Latch)나 내부 잠금(Internal Lock)**을 걸 수 있다. 이는 "첫 번째 작업이 완전히 정리될 때까지 두 번째 작업은 잠시만 기다려"라는 신호와 같다.
  - **JDBC 드라이버/MyBatis 레벨:** 하나의 커넥션은 한 번에 하나의 `ResultSet`을 처리하는 것이 가장 효율적이다. 첫 번째 쿼리의 `ResultSet`이 아직 열려 있고 데이터를 스트리밍하는 중에 두 번째 쿼리를 실행하면, 드라이버나 MyBatis 내부에서 리소스 정리를 위한 미세한 대기 시간이 발생할 수 있다.

### 왜 다른 테이블은 빠르고, 컨트롤러를 분리하면 빨라질까?

- **다른 테이블 조회:** 조회하는 테이블(리소스)이 다르면, DB는 두 쿼리가 서로에게 영향을 주지 않는다고 판단하여 락이나 대기 없이 즉시 병렬적으로 처리한다.
- **컨트롤러 분리:** 컨트롤러를 분리하면 각 호출은 **별개의 HTTP 요청**이 된다. 이는 **별개의 스레드, 별개의 트랜잭션, 그리고 커넥션 풀에서 빌려온 별개의 DB 커넥션**을 사용한다는 의미다. 즉, '동일 커넥션 내에서의 리소스 경합' 문제가 원천적으로 발생하지 않는다.

## 4. 문제 해결을 위한 실전 전략

"컨트롤러를 분리하니 빨라졌다"는 것은 문제 해결의 단서일 뿐, 근본적인 해결책이 될 수 없다. 더 나은 설계를 통해 이 문제를 해결해야 한다.

### 전략 1 (가장 권장): 서비스 책임 분리 (SRP)

애초에 하나의 서비스 메서드가 너무 많은 DB 조회를 하는 것은 단일 책임 원칙(SRP)에 위배될 수 있다. 특히 서로 다른 도메인의 테이블을 함께 조회한다면, 책임을 분리하는 것이 좋다.

**Before**

```java
@Service
@Transactional(readOnly = true)
@RequiredArgsConstructor
public class ComplexService {
    private final TableXMapper tableXMapper;

    public void doSomething() {
        // ... 로직 ...
        tableXMapper.selectFirstFromTableX(); // 느려지는 원인 1
        // ... 로직 ...
        tableXMapper.selectSecondFromTableX(); // 느려지는 원인 2
        // ... 로직 ...
    }
}
```

**After (리팩토링)**

```java
@Service
@RequiredArgsConstructor
public class ComplexService {
    private final TableXService tableXService; // 별도의 서비스 주입

    public void doSomething() {
        // ... 로직 ...
        tableXService.getFirstData();
        // ... 로직 ...
        tableXService.getSecondData();
        // ... 로직 ...
    }
}

@Service
@Transactional(readOnly = true) // 트랜잭션은 실제 DB I/O가 있는 곳에
@RequiredArgsConstructor
public class TableXService {
    private final TableXMapper tableXMapper;

    public void getFirstData() {
        tableXMapper.selectFirstFromTableX();
    }

    public void getSecondData() {
        tableXMapper.selectSecondFromTableX();
    }
}
```

`ComplexService`가 `TableXService`의 메서드를 호출할 때, Spring AOP는 각 호출에 대해 **새로운 트랜잭션(과 새로운 커넥션)**을 시작하거나 기존 트랜잭션에 참여시킨다. 만약 `TableXService`의 두 메서드를 각각 호출한다면, 각 호출이 별도의 트랜잭션으로 처리될 가능성이 높아져(설정에 따라 다름) 커넥션 공유 문제를 피할 수 있다.

### 전략 2 (차선책): `TransactionTemplate`을 이용한 수동 트랜잭션 분리

클래스 분리가 어려운 레거시 코드 등에서는, 하나의 메서드 내에서 프로그래밍적으로 트랜잭션을 분리할 수 있다.

```java
@Service
@RequiredArgsConstructor
public class ComplexService {
    private final TableXMapper tableXMapper;
    private final TransactionTemplate transactionTemplate;

    public void doSomething() {
        // ... 로직 ...
        transactionTemplate.execute(status -> tableXMapper.selectFirstFromTableX());
        // ... 로직 ...
        transactionTemplate.execute(status -> tableXMapper.selectSecondFromTableX());
        // ... 로직 ...
    }
}
```

`transactionTemplate.execute()` 블록은 각각 독립적인 트랜잭션(과 커넥션)으로 실행되므로, 리소스 경합을 피할 수 있다.

### 전략 3 (주의 필요): `Propagation.REQUIRES_NEW`

내부 호출되는 메서드에 `@Transactional(propagation = Propagation.REQUIRES_NEW)`를 붙여 강제로 새로운 트랜잭션을 시작하게 할 수 있다. 하지만 이 방법은 **자기 호출(self-invocation)**, 즉 동일 클래스 내의 메서드 호출 시에는 Spring의 프록시를 우회하여 **동작하지 않는다**는 치명적인 함정이 있다. 반드시 별도의 클래스로 분리된 Bean을 호출할 때만 의미가 있다.

## 5. 결론

`@Transactional(readOnly = true)` 환경에서 발생한 미스터리한 성능 저하의 원인은 쿼리 자체가 아니라, **단일 트랜잭션(단일 커넥션) 내에서의 동일 리소스 접근으로 인한 미세한 경합**일 가능성이 매우 높다.

`@Transactional`은 매우 편리한 추상화 기술이지만, 그 이면에서 동작하는 트랜잭션과 DB 커넥션의 생명주기를 이해하지 못하면 예기치 못한 성능 문제에 부딪힐 수 있다.

이 문제에 대한 가장 근본적인 해결책은 결국 좋은 설계로 돌아간다. **메서드와 클래스의 책임을 명확히 분리(SRP)**하여, 트랜잭션의 범위를 필요한 만큼 최소화하고 불필요한 리소스 경합을 피하는 것이 가장 현명한 접근법이다.
