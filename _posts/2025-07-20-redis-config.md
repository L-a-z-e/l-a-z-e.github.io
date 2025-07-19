---
title: RedisConfig 설정 가이드
description: 
author: laze
date: 2025-07-20 00:00:01 +0900
categories: [Dev, Redis]
tags: [Redis, RedisConfig]
---
# [Spring/Redis] `RedisConfig`, 복붙은 이제 그만! (ObjectMapper와 Serializer 완벽 해부)

## 1. 서론: `RedisConfig` 설정

Spring Boot 프로젝트에 Redis를 연동할 때, 많은 개발자들이 인터넷에서 `RedisConfig` 코드를 찾아 그대로 복사-붙여넣기 하곤 한다.

코드는 잘 동작하는 것처럼 보이지만, 그 안에 담긴 각 설정의 의미를 정확히 이해하는 경우는 드물다.

만약 아무런 설정을 하지 않고 `RedisTemplate`을 사용한다면 어떻게 될까? Redis CLI로 데이터를 조회했을 때, 우리는 다음과 같은 외계어와 마주치게 된다.

```
redis-cli> GET user:1
"\\xac\\xed\\x00\\x05t\\x00\\x1fcom.example.User\\x00\\x00..."
```

이것은 Spring의 기본 직렬화 방식(`JdkSerializationRedisSerializer`)이 만들어낸 결과물이다.

이런 데이터는 디버깅이 불가능하고, 오직 Java 애플리케이션만 이해할 수 있어 다른 언어로 만든 서비스와는 연동할 수 없다.

이 글에서는 단순히 Redis를 '연결'하는 것을 넘어, **Java 객체를 어떻게 Redis에 안전하고 효율적으로 저장하고 읽어올 것인가**에 대한 해답을 `RedisConfig`의 각 설정들을 하나씩 분해하며 찾아본다.

## 2. 1단계: JSON 번역기(`ObjectMapper`) 길들이기

우리의 목표는 Java 객체를 **"사람도 읽을 수 있고, 다른 언어와도 호환되는"** JSON 형식으로 저장하는 것이다.

이 '번역' 작업은 Jackson 라이브러리의 핵심 클래스인 `ObjectMapper`가 담당한다. `RedisConfig`의 첫 단계는 이 `ObjectMapper`에게 구체적인 번역 규칙을 알려주는 것이다.

```java
import com.fasterxml.jackson.databind.DeserializationFeature;
import com.fasterxml.jackson.databind.ObjectMapper;
import com.fasterxml.jackson.databind.SerializationFeature;
import com.fasterxml.jackson.datatype.jsr310.JavaTimeModule;
import org.springframework.context.annotation.Bean;
import org.springframework.context.annotation.Configuration;

@Configuration
public class RedisConfig {

    @Bean
    public ObjectMapper redisObjectMapper() {
        var objectMapper = new ObjectMapper();

        // 1. JSON에 없는 필드가 DTO에 있어도 에러를 내지 않고 무시
        objectMapper.configure(DeserializationFeature.FAIL_ON_UNKNOWN_PROPERTIES, false);

        // 2. LocalDateTime 등 Java 8 날짜/시간 API 지원
        objectMapper.registerModule(new JavaTimeModule());

        // 3. 날짜/시간을 숫자 타임스탬프 대신 ISO-8601 문자열로 변환
        objectMapper.disable(SerializationFeature.WRITE_DATES_AS_TIMESTAMPS);

        return objectMapper;
    }
}
```

### `.configure(FAIL_ON_UNKNOWN_PROPERTIES, false)`

- **역할:** JSON을 Java 객체로 변환(역직렬화)할 때, Java 객체에 **존재하지 않는 필드가 JSON에 있어도 에러 없이 무시**한다.
- **왜 필수인가? (실무 사례):** API 버전이 업데이트되어 Redis에 저장된 JSON에 새로운 필드(`"address"`)가 추가되었다고 가정해보자. 이 설정이 없다면, 아직 `address` 필드가 없는 구버전 애플리케이션이 이 JSON을 읽는 순간, "알 수 없는 프로퍼티 'address'가 있다"며 에러를 내고 시스템이 중단된다. 이 설정을 `false`로 두면 **시스템 전체의 하위 호환성과 안정성**을 크게 높일 수 있다.

### `.registerModule(new JavaTimeModule())`

- **역할:** Java 8부터 도입된 `LocalDateTime`, `LocalDate` 같은 `java.time` 패키지의 클래스들을 `ObjectMapper`가 올바르게 JSON으로 변환할 수 있도록 확장 모듈을 등록한다.
- **왜 필수인가?:** 이 모듈이 없으면, `ObjectMapper`는 `LocalDateTime`을 어떻게 처리해야 할지 몰라 에러를 발생시킨다. 이 모듈을 등록하면 `LocalDateTime` 객체는 `"2024-01-01T12:00:00"` 와 같은 표준 ISO-8601 형식의 문자열로 변환된다.

### `.disable(SerializationFeature.WRITE_DATES_AS_TIMESTAMPS)`

- **역할:** `LocalDateTime` 등을 숫자로 된 타임스탬프(e.g., `1672534800000`)가 아닌, 사람이 읽을 수 있는 날짜/시간 문자열 형식으로 변환하도록 강제한다.
- **왜 필수인가?:** 위 `JavaTimeModule`과 함께 사용하여, 모든 날짜/시간 관련 데이터를 디버깅하기 쉽고 표준화된 문자열 형태로 저장하도록 보장한다.

## 3. 2단계: `RedisTemplate`에 번역기 장착하기

이제 잘 설정된 `ObjectMapper`를 실제 Redis와 통신하는 `RedisTemplate`에 장착해줄 차례다.

```java
import org.springframework.data.redis.connection.RedisConnectionFactory;
import org.springframework.data.redis.core.RedisTemplate;
import org.springframework.data.redis.serializer.Jackson2JsonRedisSerializer;
import org.springframework.data.redis.serializer.StringRedisSerializer;

// ... RedisConfig 클래스 내부 ...

@Bean
public RedisTemplate<String, Object> redisTemplate(RedisConnectionFactory connectionFactory, ObjectMapper redisObjectMapper) {
    var template = new RedisTemplate<String, Object>();

    // 1. Redis 서버와의 연결 설정
    template.setConnectionFactory(connectionFactory);

    // 2. Key-Value 직렬화 규칙 설정
    template.setKeySerializer(new StringRedisSerializer());
    template.setValueSerializer(new Jackson2JsonRedisSerializer<>(redisObjectMapper, Object.class));

    // 3. Hash 직렬화 규칙 설정
    template.setHashKeySerializer(new StringRedisSerializer());
    template.setHashValueSerializer(new Jackson2JsonRedisSerializer<>(redisObjectMapper, Object.class));

    return template;
}
```

### `setKeySerializer(new StringRedisSerializer())`

- **역할:** Redis에 저장될 키(Key)를 일반적인 UTF-8 문자열로 직렬화한다.
- **왜 필수인가?:** **매우 중요하다.** 키를 문자열로 저장하면, Redis CLI나 GUI 툴에서 `GET user:123` 처럼 사람이 직접 키를 입력하여 데이터를 쉽게 조회하고 디버깅할 수 있다. 만약 이 설정을 하지 않으면 키도 바이너리 데이터로 저장되어 유지보수가 극도로 어려워진다.

### `setValueSerializer(new Jackson2JsonRedisSerializer<>(...))`

- **역할:** Redis에 저장될 값(Value)을 JSON으로 직렬화한다. 이때 우리가 1단계에서 세심하게 설정한 `redisObjectMapper`를 번역 도구로 사용하도록 지정한다.

### `setHashKeySerializer` / `setHashValueSerializer`

- **역할:** Redis의 해시(Hash) 데이터 구조를 사용할 때, 그 내부의 필드(Hash Key)와 값(Hash Value)에 대한 직렬화 규칙을 별도로 지정한다. 일반적으로는 Key/Value 직렬화 규칙과 동일하게 맞춰주는 것이 좋다.

## 4. 실무 Best Practice: 범용 `RedisTemplate` 만들기

위 예제에서 `RedisTemplate<String, Object>`처럼 값의 타입을 `Object`로 지정하면, `User` 객체든 `Product` 객체든 어떤 객체든 저장할 수 있는 범용 템플릿을 만들 수 있다. 하지만 이 경우, Redis에서 데이터를 다시 읽어올 때 Spring이 그 JSON이 원래 `User`였는지, `Product`였는지 알 수 없는 문제가 생긴다.

이때 사용하는 것이 **`GenericJackson2JsonRedisSerializer`** 다.

```java
import com.fasterxml.jackson.annotation.JsonTypeInfo;
import com.fasterxml.jackson.databind.jsontype.impl.LaissezFaireSubTypeValidator;
import org.springframework.data.redis.serializer.GenericJackson2JsonRedisSerializer;

// ... RedisConfig 클래스 내부 ...

@Bean
public ObjectMapper redisObjectMapper() {
    var objectMapper = new ObjectMapper();
    // ... 기존 설정 ...

    // [중요] 직렬화 시 클래스 타입 정보를 함께 저장
    objectMapper.activateDefaultTyping(
        LaissezFaireSubTypeValidator.instance,
        ObjectMapper.DefaultTyping.NON_FINAL,
        JsonTypeInfo.As.PROPERTY
    );

    return objectMapper;
}

@Bean
public RedisTemplate<String, Object> redisTemplate(RedisConnectionFactory connectionFactory, ObjectMapper redisObjectMapper) {
    var template = new RedisTemplate<String, Object>();
    template.setConnectionFactory(connectionFactory);

    // 모든 Key/HashKey는 String으로
    var stringSerializer = new StringRedisSerializer();
    template.setKeySerializer(stringSerializer);
    template.setHashKeySerializer(stringSerializer);

    // 모든 Value/HashValue는 범용 JSON Serializer로
    var jsonSerializer = new GenericJackson2JsonRedisSerializer(redisObjectMapper);
    template.setValueSerializer(jsonSerializer);
    template.setHashValueSerializer(jsonSerializer);

    return template;
}
```

`activateDefaultTyping` 설정을 추가하면, `ObjectMapper`는 객체를 JSON으로 변환할 때 `{"@class":"com.example.User", "id":1, "name":"laze"}` 와 같이 **클래스의 전체 경로를 `@class` 프로퍼티에 함께 저장**한다.

`GenericJackson2JsonRedisSerializer`는 이 `@class` 프로퍼티를 읽어, JSON을 다시 정확한 원본 Java 객체 타입으로 복원해준다.

이로써 우리는 타입 캐스팅의 걱정 없이 어떤 객체든 안전하게 저장하고 읽어올 수 있는 진정한 범용 `RedisTemplate`을 완성할 수 있다.

## 5. 결론: 좋은 `RedisConfig`의 조건

좋은 `RedisConfig`는 다음 질문에 "예"라고 답할 수 있어야 한다.

- **가독성:** Redis에 저장된 Key와 Value가 사람이 쉽게 읽고 디버깅할 수 있는 형태인가? (문자열, JSON)
- **안정성:** 예기치 않은 데이터 변경(필드 추가 등)에 유연하게 대처할 수 있는가? (`FAIL_ON_UNKNOWN_PROPERTIES`)
- **확장성:** 여러 타입의 객체를 저장하고 읽어올 수 있는 범용적인 설정인가? (`GenericJackson2JsonRedisSerializer`)

`RedisConfig`는 단순한 연결 설정이 아니라, 데이터의 안정성과 확장성, 그리고 애플리케이션의 유지보수성을 결정하는 중요한 아키텍처 설계의 일부다. 의미를 이해하고 작성한 단 몇 줄의 설정이, 미래의 수많은 버그와 디버깅 시간을 줄여줄 것이다.

---
