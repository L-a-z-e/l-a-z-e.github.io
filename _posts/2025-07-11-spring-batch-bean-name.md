---
title: 제네릭 타입이 다른데 왜 Bean 이름이 충돌할까? (Type Erasure와 Spring DI)
description: 
author: laze
date: 2025-07-11 00:00:08 +0900
categories: [Dev, Java]
tags: [SpringBatch, Bean, Generic, TypeErasure]
---
# [Spring/Java] 제네릭 타입이 다른데 왜 Bean 이름이 충돌할까? (Type Erasure와 Spring DI)

## 1. 서론: "분명히 타입이 다른데...?" - 흔하지만 당황스러운 Bean 충돌 에러

Spring, 특히 Spring Batch로 여러 개의 독립적인 Job 설정을 구성하다 보면, 다음과 같이 당황스러운 에러 메시지를 마주칠 때가 있다.

```
The bean 'reader', defined in class path resource [com/.../config/BJobConfig.class], could not be registered.
A bean with that name has already been defined in class path resource [com/.../config/AJobConfig.class] and overriding is disabled.
```

코드를 살펴보면 더 의아하다.

**`AJobConfig.java`**

```java
@Configuration
public class AJobConfig {
    @Bean
    public ItemReader<ADto> reader() {
        // ... A DTO를 읽는 ItemReader 구현 ...
    }
}
```

**`BJobConfig.java`**

```java
@Configuration
public class BJobConfig {
    @Bean
    public ItemReader<BDto> reader() {
        // ... B DTO를 읽는 ItemReader 구현 ...
    }
}
```

`ItemReader<ADto>`와 `ItemReader<BDto>`는 명백히 다른 타입이다. 그런데 왜 Spring은 이 둘을 같은 이름의 Bean으로 인식하고 충돌을 일으키는 것일까? 이 글에서는 이 문제의 근본적인 원인을 파헤치고, Spring의 Bean 관리 원리와 Java 제네릭의 한계를 통해 명쾌한 해답을 찾아본다.

## 2. Spring의 Bean 식별법: "이름이 유일한가?"

Spring IoC 컨테이너는 내부에 수많은 Bean들을 보관하는 거대한 창고와 같다. 이 창고에서 각 물건(Bean)을 구분하는 **유일한 식별자는 바로 '이름(Bean Name/ID)'**이다.

`@Bean` 어노테이션을 사용하여 Bean을 등록할 때, 개발자가 별도로 이름을 지정하지 않으면 Spring은 기본적으로 **메서드의 이름(method name)을 Bean의 이름으로 사용**한다.

위 예시에서,

- `AJobConfig`의 `reader()` 메서드는 **"reader"** 라는 이름의 Bean을 등록하려고 시도한다.
- `BJobConfig`의 `reader()` 메서드 또한 **"reader"** 라는 이름의 Bean을 등록하려고 시도한다.

하나의 컨테이너(창고)에 똑같은 이름표(`reader`)를 가진 두 개의 객체를 동시에 넣을 수 없기 때문에, Spring은 "이름이 중복되는데, 덮어쓰기는 비활성화되어 있어서 등록할 수 없습니다!" (`overriding is disabled`) 라며 오류를 발생시키는 것이다.

## 3. 제네릭의 함정: Java의 타입 소거(Type Erasure)

"그렇다면 왜 제네릭 타입 정보는 고려되지 않는가?" 라는 질문이 남는다. 이것이 바로 이 문제의 핵심이다.

Java의 제네릭은 컴파일 시점에 타입 안정성을 보장해주는 강력한 기능이지만, 한 가지 중요한 특징이 있다. 바로 **타입 소거(Type Erasure)**다.

**타입 소거란, 제네릭 타입 정보(`<>` 안에 있는 정보)가 컴파일 시점에만 타입 검사를 위해 사용되고, 런타임(실행 시점)에는 대부분 사라지는 현상**을 말한다.

즉, 코드가 컴파일되고 바이트코드로 변환되면, `ItemReader<ADto>`나 `ItemReader<BDto>`나 모두 그냥 `ItemReader`라는 원시 타입(Raw Type)처럼 취급된다.

Spring 컨테이너가 Bean을 등록하고 관리하는 시점은 바로 이 **런타임**이다. 런타임에 Spring이 바라보는 두 Bean의 타입은 그저 `ItemReader`일 뿐, `<ADto>`나 `<BDto>`와 같은 제네릭 정보로는 두 Bean을 구분할 수 없다.

**결론적으로, Spring은 오직 'Bean의 이름'으로만 유일성을 판단하며, 제네릭 타입 정보는 이 과정에서 사용되지 않는다.**

## 4. 해결책: 각 Bean에게 고유한 이름표 달아주기

문제의 원인이 '이름 중복'이므로, 해결책은 간단하다. 각 Bean에게 고유한 이름을 부여하면 된다.

### 해결책 1 (Best Practice): `@Bean(name = "...")`으로 명시적 이름 지정

가장 명확하고 권장되는 방법이다. `@Bean` 어노테이션에 `name` 속성을 사용하여 각 Bean에 고유한 ID를 직접 부여한다.

**`AJobConfig.java`**

```java
@Configuration
public class AJobConfig {
    @Bean(name = "aDtoReader") // 고유한 이름 "aDtoReader"를 부여
    public ItemReader<ADto> reader() {
        // ...
    }
}
```

**`BJobConfig.java`**

```java
@Configuration
public class BJobConfig {
    @Bean(name = "bDtoReader") // 고유한 이름 "bDtoReader"를 부여
    public ItemReader<BDto> reader() {
        // ...
    }
}
```

이제 두 Bean은 각각 "aDtoReader"와 "bDtoReader"라는 고유한 이름을 갖게 되어 충돌이 발생하지 않는다.

의존성을 주입받을 때는 `@Qualifier` 어노테이션을 사용하여 정확히 원하는 Bean을 지정할 수 있다.

```java
@Bean
public Step aStep(@Qualifier("aDtoReader") ItemReader<ADto> reader, ...) {
    // ...
}
```

- **장점:** 의도가 매우 명확하다. 나중에 메서드 이름을 리팩토링하더라도 Bean의 이름은 그대로 유지되어 안정적이다.

### 해결책 2 (간단한 방법): 메서드 이름 자체를 고유하게 변경

메서드 이름이 곧 Bean의 이름이 되므로, 메서드 이름 자체를 다르게 짓는 것만으로도 문제가 해결된다.

**`AJobConfig.java`**

```java
@Configuration
public class AJobConfig {
    @Bean
    public ItemReader<ADto> aDtoReader() { // 메서드 이름을 고유하게 변경
        // ...
    }
}
```

**`BJobConfig.java`**

```java
@Configuration
public class BJobConfig {
    @Bean
    public ItemReader<BDto> bDtoReader() { // 메서드 이름을 고유하게 변경
        // ...
    }
}
```

의존성을 주입받을 때는, Spring이 파라미터 이름과 Bean 이름을 자동으로 매칭해주므로 코드가 더 간결해질 수 있다.

```java
@Bean
public Step aStep(ItemReader<ADto> aDtoReader, ...) { // 파라미터 이름으로 자동 주입
    // ...
}
```

## 5. 심화 학습: `scopedTarget.reader`의 정체

에러 로그에 종종 등장하는 `scopedTarget.reader`는 무엇일까? 이는 `@StepScope`나 `@RequestScope`처럼 스코프가 지정된 Bean과 관련이 있다.

스코프가 지정된 Bean은 실제 Bean을 감싸는 **프록시(Proxy) 객체**로 먼저 컨테이너에 등록된다.

그리고 해당 스코프(e.g., Step 실행)가 활성화될 때 비로소 **실제 타겟(Target) Bean**이 생성된다.

이때 Spring이 내부적으로 관리하는 실제 타겟 Bean의 이름이 바로 `scopedTarget.원래빈이름`이다.

따라서 `scopedTarget.reader`에서 충돌이 발생했다는 로그는, **`@StepScope`가 적용된 `reader`라는 이름의 Bean이 중복**되었다는 확실한 증거다.

## 6. 결론

"제네릭 타입이 다른데 왜 Bean이 충돌할까?"라는 질문에 대한 답을 요약하면 다음과 같다.

- **원인:** Spring의 Bean 이름은 기본적으로 **메서드명**으로 결정되며, Java의 **타입 소거(Type Erasure)** 특성 때문에 제네릭 타입 정보는 런타임에 Bean을 구분하는 기준이 될 수 없다.
- **해결:** `@Bean(name=...)`이나 고유한 메서드명을 통해 각 Bean에 **유일한 이름**을 부여해야 한다.

이 문제는 단순히 Spring의 동작 방식을 넘어, Java 언어 자체의 특성을 함께 이해해야만 명확히 파악할 수 있다. 이 원리를 이해했다면, 당신은 프레임워크와 언어의 상호작용을 더 깊이 이해하는 개발자로 한 단계 성장한 것이다.

---
