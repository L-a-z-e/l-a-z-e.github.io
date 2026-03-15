---
title: "Redis EXPIRE 호출 시 StackOverflowError — Redisson 버그 우회기"
description: "Redisson + Spring Data Redis 조합에서 expire() 호출 시 발생하는 무한 재귀 StackOverflowError의 원인과 SETEX/Lua Script 이중 전략 우회 기록"
author: laze
date: 2026-03-12 09:30:00 +0900
categories: [Dev, Redis]
tags: [Redis, Redisson, LuaScript]
---

## 문제 상황

쇼핑 서비스의 Redis 키(쿠폰 재고, 타임딜 재고, 구매 기록)에 TTL이 설정되어 있지 않았습니다. 쿠폰이 만료되거나 타임딜이 종료되어도 Redis 키는 남아 있어 메모리가 계속 증가하는 구조였습니다.

```
coupon:stock:42        → "150"     (TTL 없음, 영구 보존)
coupon:issued:42       → {userId1, userId2, ...}  (TTL 없음)
timedeal:stock:7:101   → "50"     (TTL 없음)
```

TTL을 설정하면 간단히 해결될 문제였습니다. `StringRedisTemplate.expire()`를 호출하면 끝일 줄 알았는데, 실행하자마자 `StackOverflowError`가 발생했습니다.

```
java.lang.StackOverflowError
  at DefaultedRedisConnection.pExpire(DefaultedRedisConnection.java:220)
  at DefaultedRedisConnection.expire(DefaultedRedisConnection.java:213)
  at DefaultedRedisConnection.pExpire(DefaultedRedisConnection.java:220)
  ... (무한 재귀)
```

## 근본 원인

스택 트레이스를 보면 `expire()` → `pExpire()` → `expire()` → `pExpire()`가 반복되고 있습니다. 두 가지 요인이 결합된 결과입니다.

> 확인된 버전: Spring Boot 3.5.5, Redisson 3.45.1, Spring Data Redis 3.5.x

### 1. RedissonConnection의 미구현

Spring Data Redis의 `DefaultedRedisConnection`은 `expire()`와 `pExpire()`에 default 구현을 제공합니다. 문제는 이 default 구현이 서로를 호출하는 구조라는 점입니다.

```java
// DefaultedRedisConnection (Spring Data Redis)
default Boolean expire(byte[] key, long seconds) {
    return pExpire(key, seconds * 1000);  // → pExpire 호출
}

default Boolean pExpire(byte[] key, long millis) {
    return expire(key, millis / 1000);    // → expire 호출
}
```

`RedissonConnection`이 이 두 메서드를 직접 override하지 않았기 때문에, default 메서드끼리 무한 재귀에 빠집니다.

### 2. micrometer-tracing-bridge-otel의 개입

`micrometer-tracing-bridge-otel` 트레이싱 라이브러리가 Connection 객체를 프록시로 래핑하면서, 메서드 dispatch가 원래 구현체가 아닌 `DefaultedRedisConnection`의 default 메서드로 라우팅됩니다. 트레이싱 없이는 Redisson 내부에서 다르게 처리될 수 있는 경로가, 프록시 계층을 거치면서 무한 재귀로 빠지게 된 것입니다.

## 대안 검토

| 대안 | 장점 | 단점 | 선택 |
|------|------|------|:----:|
| **SETEX** (`set(key, value, timeout, unit)`) | `expire()` 미경유, 단일 Redis 명령어 | 값 설정 시에만 사용 가능 | String 키 |
| **Lua Script** (`redis.call('EXPIRE')`) | Redis 서버에서 직접 실행, Java Connection 계층 우회 | 스크립트 변경 시 SHA 캐시 무효화 처리 필요, 디버깅 시 Redis 서버 로그 확인 필요 | Set/기존 키 |
| Lettuce로 교체 | 근본 해결 | 분산 락/동기화 기능 상실, 대규모 변경 | 미선택 |
| RedissonConnection 패치 | 근본 해결 | 외부 라이브러리 포크 필요, 유지보수 부담 | 미선택 |

Redisson의 분산 락 기능을 쿠폰 발급과 타임딜 구매의 동시성 제어에 사용하고 있어서, Lettuce 교체나 포크 패치는 비용이 큽니다. `expire()` 호출 자체를 피하는 우회 전략을 선택했습니다.

## 해결: 이중 전략

### 전략 1: String 키 — SETEX

재고처럼 값을 초기화하면서 동시에 TTL을 설정할 수 있는 경우, `opsForValue().set(key, value, timeout, unit)`을 사용합니다. 이 메서드는 내부적으로 Redis의 `SET key value EX seconds` 명령어를 사용하므로, `expire()`를 거치지 않습니다.

```java
// CouponRedisService
public void initializeCouponStock(Long couponId, int quantity, long ttlSeconds) {
    String stockKey = COUPON_STOCK_KEY + couponId;
    stringRedisTemplate.opsForValue()
        .set(stockKey, String.valueOf(quantity), ttlSeconds, TimeUnit.SECONDS);
}

// TimeDealRedisService
public void initializeStock(Long timeDealId, Long productId, int quantity, long ttlSeconds) {
    String stockKey = buildStockKey(timeDealId, productId);
    stringRedisTemplate.opsForValue()
        .set(stockKey, String.valueOf(quantity), ttlSeconds, TimeUnit.SECONDS);
}
```

값 설정과 TTL 설정이 하나의 Redis 명령어로 원자적으로 처리되므로, 중간에 TTL 없이 키가 존재하는 구간도 없습니다.

### 전략 2: Set/기존 키 — Lua Script

`coupon:issued:{id}`처럼 `SADD`로 이미 생성된 Set 타입 키에는 SETEX를 사용할 수 없습니다. 이 경우 Lua Script로 Redis 서버에서 `EXPIRE` 명령을 직접 실행합니다.

```java
private static final String EXPIRE_LUA = "return redis.call('EXPIRE', KEYS[1], ARGV[1])";
private static final DefaultRedisScript<Long> EXPIRE_SCRIPT;

static {
    EXPIRE_SCRIPT = new DefaultRedisScript<>(EXPIRE_LUA, Long.class);
}

private void expireViaLua(String key, long ttlSeconds) {
    stringRedisTemplate.execute(
        EXPIRE_SCRIPT,
        Collections.singletonList(key),
        String.valueOf(ttlSeconds)
    );
}

public void setIssuedKeyExpiration(Long couponId, long ttlSeconds) {
    String issuedKey = COUPON_ISSUED_KEY + couponId;
    expireViaLua(issuedKey, ttlSeconds);
}
```

Lua Script는 Redis 서버에서 실행되기 때문에, Java 쪽의 `RedissonConnection.expire()` 메서드를 전혀 거치지 않습니다. `StringRedisTemplate.execute()`는 스크립트를 EVALSHA로 전송할 뿐이므로, expire/pExpire 재귀 경로와 무관합니다.

### TTL 정책

TTL 값은 쿠폰/타임딜의 만료 시점을 기준으로 계산하되, 1일의 버퍼를 둡니다.

```
TTL = expiresAt - now + 1일
```

타임딜과 쿠폰의 유효기간은 수일에서 수주 단위입니다. 1일 버퍼는 전체 TTL 대비 미미한 수준이면서, 만료 직전 구매/발급 트랜잭션이 진행 중일 때 키가 먼저 삭제되는 것을 방지하기에 충분합니다. 버퍼가 지나면 자동으로 삭제되므로 메모리 누수는 발생하지 않습니다.

## 적용 결과

| 키 패턴 | Redis 타입 | 우회 방식 | 메서드 |
|---------|-----------|----------|--------|
| `coupon:stock:{id}` | String | SETEX | `initializeCouponStock(id, qty, ttl)` |
| `coupon:issued:{id}` | Set | Lua Script | `setIssuedKeyExpiration(id, ttl)` |
| `timedeal:stock:{dealId}:{prodId}` | String | SETEX | `initializeStock(dealId, prodId, qty, ttl)` |
| `timedeal:purchased:{dealId}:{prodId}:{userId}` | String | Lua Script | `setPurchasedKeyExpiration(dealId, prodId, userId, ttl)` |

## 한계와 후속 과제

이 방식에는 두 가지 한계가 있습니다.

**1. `expire()` 버그가 코드베이스에 잠재적으로 남아 있다**

현재 우회한 곳은 문제가 없지만, 향후 다른 개발자가 `expire()`를 직접 호출하면 동일한 에러가 발생합니다. 각 메서드에 주석으로 Lua Script 사용 이유를 명시해 두었지만, 근본 해결은 아닙니다.

**2. Lua Script 상수가 두 서비스 클래스에 중복 정의되어 있다**

`CouponRedisService`와 `TimeDealRedisService`에 동일한 `EXPIRE_LUA`와 `EXPIRE_SCRIPT`가 각각 정의되어 있습니다. 공통 유틸로 추출할 수 있지만, 현재는 두 곳에만 사용되므로 유지하고 있습니다.

Redisson이 `expire()`/`pExpire()`를 제대로 override하는 버전이 나오면, Lua Script를 걷어내고 표준 API로 돌아갈 수 있습니다. Lua Script는 그 시점에 제거해도 동작에는 영향이 없습니다.
