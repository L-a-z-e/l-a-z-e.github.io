---
title: Redis 캐시 설정 - CacheConfig 가이드 (TTL, Serializer, 보안)
description: 
author: laze
date: 2025-07-21 00:00:01 +0900
categories: [Dev, Redis]
tags: [Redis, RedisCacheManager, Cache, Cacheable]
---
# [Spring Boot] 실전 Redis 캐시 설정: `CacheConfig` 완벽 가이드 (TTL, Serializer, 보안)

## 1. 서론: `@Cacheable`, 그 뒤에서는 무슨 일이 일어날까?

Spring Boot에서 `@EnableCaching`을 선언하고, 서비스 메서드에 `@Cacheable("myCache")` 어노테이션 하나만 붙이면 마법처럼 캐시 기능이 동작한다.

하지만 실무에서는 곧바로 몇 가지 질문에 부딪히게 된다.

- "모든 캐시의 유효시간(TTL)이 똑같아도 될까?"
- "Redis에는 데이터가 대체 어떤 형태로 저장되는 거지?"
- "`null` 값도 캐싱되면 문제가 생기지 않을까?"

이러한 문제들을 해결하고, 애플리케이션의 요구사항에 맞는 정교한 캐시 전략을 구현하기 위해서는 `CacheConfig`를 직접 작성해야 한다.

이 글에서는 단순히 코드를 복사-붙여넣기 하는 것을 넘어, 각 설정이 어떤 의미를 가지며 왜 필요한지 심층적으로 분석한다.

## 2. 캐시 설정의 시작: `RedisCacheManagerBuilderCustomizer`

Spring Boot는 `spring-boot-starter-data-redis` 의존성이 있으면, 기본적인 `RedisCacheManager`를 자동으로 설정해준다.

우리는 이 자동 설정을 완전히 무시하고 처음부터 새로 만드는 대신, **`RedisCacheManagerBuilderCustomizer`**를 사용하여 기존 자동 설정에 우리만의 규칙을 덧붙이는 방식을 사용할 것이다.

이는 더 적은 코드로 원하는 설정을 적용할 수 있는, Spring Boot가 권장하는 방식이다.

```java
import org.springframework.boot.autoconfigure.cache.RedisCacheManagerBuilderCustomizer;
import org.springframework.context.annotation.Bean;
import org.springframework.context.annotation.Configuration;

@Configuration
public class CacheConfig {

    @Bean
    public RedisCacheManagerBuilderCustomizer redisCacheManagerBuilderCustomizer() {
        // 이 메서드 안에서 커스텀 설정을 정의한다.
        return builder -> {
            // ... (3, 4번 항목의 설정 코드가 여기에 들어감)
        };
    }
}
```

## 3. 핵심 1: 직렬화(Serialization) 설정 - 안전하고 읽기 쉬운 캐시 데이터 만들기

Java 객체를 Redis에 저장하려면 '직렬화' 과정이 필수적이다.

이때 어떤 직렬화 방식을 선택하느냐에 따라 캐시 데이터의 가독성, 호환성, 보안 수준이 결정된다.

우리의 목표는 **"사람이 읽을 수 있고(JSON), 타입 정보가 안전하게 보존되는"** 직렬화 방식을 설정하는 것이다.

```java
// ... CacheConfig 내부 ...
import com.fasterxml.jackson.annotation.JsonTypeInfo;
import com.fasterxml.jackson.databind.ObjectMapper;
import com.fasterxml.jackson.databind.jsontype.impl.LaissezFaireSubTypeValidator;
import org.springframework.data.redis.serializer.GenericJackson2JsonRedisSerializer;
import org.springframework.data.redis.serializer.RedisSerializationContext.SerializationPair;
import org.springframework.data.redis.serializer.StringRedisSerializer;

// ... redisCacheManagerBuilderCustomizer() 메서드 내부 ...

// 1. ObjectMapper 커스터마이징 (JSON 변환 규칙 정의)
ObjectMapper objectMapper = new ObjectMapper();
// Java 8 날짜/시간 API(LocalDateTime 등) 지원
objectMapper.registerModule(new JavaTimeModule());
// 역직렬화 시 JSON에 없는 필드가 있어도 에러를 내지 않음 (하위 호환성)
objectMapper.configure(DeserializationFeature.FAIL_ON_UNKNOWN_PROPERTIES, false);
// [중요] 직렬화 시 클래스 타입 정보를 JSON에 함께 저장하여, 역직렬화 시 정확한 타입으로 복원
objectMapper.activateDefaultTyping(
    LaissezFaireSubTypeValidator.instance, // 모든 타입 허용 (보안에 유의, 실무에서는 특정 패키지 지정 권장)
    ObjectMapper.DefaultTyping.NON_FINAL,
    JsonTypeInfo.As.PROPERTY
);

// 2. 직렬화기(Serializer) 생성
GenericJackson2JsonRedisSerializer jsonRedisSerializer = new GenericJackson2JsonRedisSerializer(objectMapper);

// 3. RedisCacheConfiguration에 직렬화 규칙 적용
RedisCacheConfiguration redisCacheConfiguration = RedisCacheConfiguration.defaultCacheConfig()
    .serializeKeysWith(SerializationPair.fromSerializer(new StringRedisSerializer())) // Key는 String으로
    .serializeValuesWith(SerializationPair.fromSerializer(jsonRedisSerializer)); // Value는 JSON으로
```

### 직렬화 설정 요약

- **Key Serializer (`StringRedisSerializer`):** Redis의 키는 항상 문자열로 저장하는 것이 좋다. `redis-cli` 등에서 `GET "user:1"`처럼 직접 조회하여 디버깅하기 매우 편리해진다.
- **Value Serializer (`GenericJackson2JsonRedisSerializer`):** 어떤 Java 객체든 JSON으로 변환해준다.
- **`activateDefaultTyping`:** 이 설정 덕분에, 캐시된 JSON 데이터는 `{"@class":"com.example.UserDto", "id":1, ...}` 와 같이 타입 정보를 포함하게 된다. Spring은 이 `@class` 정보를 보고 JSON을 다시 정확한 DTO 객체로 복원할 수 있다.
- **`PolymorphicTypeValidator`:** `activateDefaultTyping` 사용 시 발생할 수 있는 역직렬화 보안 취약점을 막기 위한 **필수 보안 설정**이다. 실무에서는 `LaissezFaireSubTypeValidator.instance`(모든 타입 허용) 대신 `BasicPolymorphicTypeValidator.builder().allowIfSubType("com.myapp.dto").build()`처럼 허용할 패키지를 명시적으로 지정하는 것이 안전하다.

## 4. 핵심 2: 캐시별 정책 설정 - TTL과 동작 제어

이제 잘 만들어진 직렬화 규칙을 기반으로, 각 캐시의 이름별로 유효시간(TTL, Time-To-Live)과 동작 방식을 다르게 설정할 수 있다.

```java
// ... CacheConfig 클래스 내부 ...
import java.time.Duration;

public static final String CACHE_USERS = "users";
public static final String CACHE_PRODUCTS_POPULAR = "products:popular";

// ... redisCacheManagerBuilderCustomizer() 메서드 내부 ...

// (앞선 직렬화 설정 코드 이후)
builder
    // 기본 캐시 설정: 위에서 만든 직렬화 규칙을 모든 캐시에 공통으로 적용
    .cacheDefaults(redisCacheConfiguration)

    // "users" 캐시에 대한 개별 설정
    .withCacheConfiguration(CACHE_USERS,
        redisCacheConfiguration
            .entryTtl(Duration.ofHours(1)) // 1시간 TTL
            .disableCachingNullValues() // null 값은 캐싱하지 않음
    )

    // "products:popular" 캐시에 대한 개별 설정
    .withCacheConfiguration(CACHE_PRODUCTS_POPULAR,
        redisCacheConfiguration
            .entryTtl(Duration.ofMinutes(10)) // 10분 TTL
    );
```

### 캐시 정책 설정 요약

- `.cacheDefaults(redisCacheConfiguration)`: 모든 캐시에 적용될 **기본 설정**을 지정한다. 여기에 공통 직렬화 규칙을 넣어두면 코드가 깔끔해진다.
- `.withCacheConfiguration("캐시이름", 설정)`: 특정 이름의 캐시에 대해 **개별 정책**을 덧씌운다.
- `.entryTtl(Duration)`: 캐시의 유효시간을 `java.time.Duration` 객체로 설정한다. `Duration.ofMinutes(10)`, `Duration.ofDays(1)` 등 직관적인 설정이 가능하다.
- `.disableCachingNullValues()`: **매우 중요한 옵션.** DB 조회 결과가 `null`일 때, 이 `null` 자체를 캐싱하지 않도록 막는다. 만약 `null`이 캐시되면, 캐시가 만료되기 전까지는 DB에 실제 데이터가 생성되어도 애플리케이션은 계속 캐시된 `null`만 반환하는 '캐시 오염' 문제가 발생할 수 있다.

## 5. 실전 `CacheConfig` 전체 코드와 사용법

위 내용들을 종합한 최종 `CacheConfig` 파일은 다음과 같다.

```java
@EnableCaching
@Configuration
public class CacheConfig {

    // 캐시 이름 상수 정의
    public static final String CACHE_USERS = "users";
    public static final String CACHE_PRODUCTS_POPULAR = "products:popular";

    @Bean
    public RedisCacheManagerBuilderCustomizer redisCacheManagerBuilderCustomizer() {
        return builder -> {
            // 1. ObjectMapper 커스터마이징
            ObjectMapper objectMapper = new ObjectMapper()
                    .registerModule(new JavaTimeModule())
                    .configure(DeserializationFeature.FAIL_ON_UNKNOWN_PROPERTIES, false)
                    .activateDefaultTyping(
                        LaissezFaireSubTypeValidator.instance,
                        ObjectMapper.DefaultTyping.NON_FINAL,
                        JsonTypeInfo.As.PROPERTY
                    );

            // 2. 기본 캐시 설정 (직렬화 규칙)
            RedisCacheConfiguration defaultConfig = RedisCacheConfiguration.defaultCacheConfig()
                    .serializeKeysWith(SerializationPair.fromSerializer(new StringRedisSerializer()))
                    .serializeValuesWith(SerializationPair.fromSerializer(new GenericJackson2JsonRedisSerializer(objectMapper)));

            // 3. 캐시별 개별 설정
            builder
                .cacheDefaults(defaultConfig)
                .withCacheConfiguration(CACHE_USERS,
                    defaultConfig
                        .entryTtl(Duration.ofHours(1))
                        .disableCachingNullValues()
                )
                .withCacheConfiguration(CACHE_PRODUCTS_POPULAR,
                    defaultConfig
                        .entryTtl(Duration.ofMinutes(10))
                );
        };
    }
}
```

### 사용법: `@Cacheable`에서 캐시 이름 지정하기

이제 서비스 계층에서 `@Cacheable` 어노테이션을 사용할 때, `value` 속성에 `CacheConfig`에서 정의한 캐시 이름을 지정하면 해당 정책이 자동으로 적용된다.

```java
@Service
public class UserService {
    @Cacheable(value = CacheConfig.CACHE_USERS, key = "#id")
    public UserDto findUserById(Long id) {
        // 이 메서드의 결과는 "users" 캐시에 1시간 동안 저장된다.
        // ... DB 조회 로직 ...
    }
}

@Service
public class ProductService {
    @Cacheable(value = CacheConfig.CACHE_PRODUCTS_POPULAR)
    public List<ProductDto> findPopularProducts() {
        // 이 메서드의 결과는 "products:popular" 캐시에 10분 동안 저장된다.
        // ... DB 조회 로직 ...
    }
}
```

## 6. 결론: 좋은 `CacheConfig`란 무엇인가?

`CacheConfig`는 단순한 설정 파일이 아니라, 애플리케이션의 성능과 안정성을 좌우하는 중요한 설계의 일부다. 좋은 캐시 설정은 다음 요소를 만족해야 한다.

- **가독성:** Redis에 저장된 Key와 Value가 사람이 쉽게 읽고 디버깅할 수 있는 형태인가?
- **안정성 & 보안:** 예기치 않은 데이터 변경(필드 추가)에 유연하게 대처하고, 역직렬화 공격으로부터 안전한가?
- **유연성:** 각기 다른 데이터 성격에 맞춰 캐시별로 다른 TTL과 정책을 적용할 수 있는가?

의미를 이해하고 작성한 `CacheConfig` 하나가, 미래에 발생할 수많은 버그와 디버깅 시간을 줄여주는 든든한 방어막이 되어줄 것이다.
