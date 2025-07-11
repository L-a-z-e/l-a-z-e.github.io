---
title: 다형성의 함정 - Bean 반환 타입을 인터페이스로 하면 안 되는 이유
description: 
author: laze
date: 2025-07-11 00:00:07 +0900
categories: [Dev, Java]
tags: [SpringBatch, Polymorphism]
---
# [Java/Spring] 다형성의 함정: `@Bean` 반환 타입을 인터페이스로 하면 안 되는 이유

## 1. 서론: 다형성에 대한 익숙한 오해

"구현보다는 인터페이스에 의존하라(Depend on interfaces, not on implementations)."

객체지향 프로그래밍(OOP)을 공부했다면 귀에 못이 박이도록 들었던 원칙일 것이다.

Spring 프레임워크 역시 의존성 주입(DI) 시, 구체적인 클래스보다는 인터페이스를 통해 의존성을 주입받으라고 강력하게 권장한다.

```java
@Service
public class MyService {
    private final MemberRepository memberRepository; // 구현체(JpaMemberRepository)가 아닌 인터페이스로 받는다.

    public MyService(MemberRepository memberRepository) {
        this.memberRepository = memberRepository;
    }
}
```

이 원칙은 코드를 유연하고, 테스트하기 쉽고, 결합도를 낮추는 훌륭한 방법이다.

하지만 많은 개발자들이 이 원칙을 맹신하여, **Bean을 생성하고 등록하는 `@Bean` 메서드**에까지 그대로 적용하려다 예기치 못한 오류와 마주치곤 한다.

Spring Batch의 `ItemReader`를 예로 들어보자.

```java
// MyBatchConfiguration.java

// '인터페이스에 의존하라'는 원칙에 따라 반환 타입을 ItemReader로 선언했다.
@Bean
public ItemReader<MyDto> mybatisItemReader() {
    return new MyBatisCursorItemReaderBuilder<MyDto>()
            .sqlSessionFactory(...)
            .queryId(...)
            .build();
}
```

위 코드는 겉보기에는 아무 문제가 없어 보인다. 하지만 이 코드는 특정 상황에서 `MyBatisCursorItemReader`가 제대로 동작하지 않는 원인이 될 수 있다.

대체 왜일까? 이 글에서는 이 다형성의 함정에 대해 심층적으로 분석해 본다.

## 2. "사용" 시점 vs "생성" 시점: 다형성을 바라보는 두 가지 관점

이 문제를 이해하려면, 다형성이 적용되는 시점을 **'사용(Consumption)'**과 **'생성(Production)'**이라는 두 가지 관점으로 나누어 생각해야 한다.

### "사용" 시점: 의존성 주입(DI)

이것은 우리가 흔히 아는, 인터페이스로 의존성을 주입받는 단계다.

```java
@Configuration
@RequiredArgsConstructor
public class MyStepConfiguration {
    // 이미 컨테이너에 완벽하게 생성된 Bean을 가져와 "사용"한다.
    private final ItemReader<MyDto> mybatisItemReader;

    @Bean
    public Step myStep() {
        return new StepBuilder("myStep", jobRepository)
                .<MyDto, MyDto>chunk(10, transactionManager)
                .reader(mybatisItemReader) // 사용할 때는 인터페이스 타입으로!
                .writer(...)
                .build();
    }
}
```

이 단계에서는 아무런 문제가 없다. Spring 컨테이너는 `ItemReader<MyDto>` 타입의 Bean을 찾아 `MyBatisCursorItemReader`의 **완성된 인스턴스**를 주입해준다.

`.reader()` 메서드 또한 `ItemReader` 인터페이스를 받도록 설계되어 있으므로 다형성의 이점을 온전히 누릴 수 있다.

### "생성" 시점: `@Bean` 메서드

문제는 바로 여기, Bean 객체를 **`new` 또는 빌더를 통해 '만들어서' 컨테이너에 등록하는** 바로 이 시점에서 발생한다.

```java
@Configuration
public class MyBatchConfiguration {
    @Bean
    // "이 메서드는 ItemReader 타입의 Bean을 생성합니다" 라고 Spring에게 알리는 중
    public ItemReader<MyDto> myReader() {
        return new MyBatisCursorItemReaderBuilder<MyDto>(...)
                .build();
    }
}
```

## 3. 문제의 핵심: 왜 `@Bean` 반환 타입을 인터페이스로 하면 위험한가?

`@Bean` 메서드의 반환 타입은 단순히 자바 문법의 반환 타입이 아니다.

이는 **"내가 지금부터 등록할 Bean의 공식적인 타입은 이것입니다"** 라고 Spring 컨테이너에게 알려주는 중요한 **메타데이터**다.

반환 타입을 상위 인터페이스로 지정하면, 이 과정에서 중요한 정보들이 손실되거나 오해를 불러일으킬 수 있다.

### 3.1. 빌더 패턴과 복잡한 초기화 로직

Spring Batch의 `...Builder` 클래스들은 단순히 `new` 키워드를 대신하는 것이 아니다. `build()` 메서드 내부에서는 다음과 같은 복잡한 일들이 벌어진다.

1. 구체 클래스(`MyBatisCursorItemReader`)의 인스턴스를 생성한다.
2. `.sqlSessionFactory()`, `.queryId()` 등으로 전달받은 설정값들을 생성된 객체의 내부 필드에 주입한다.
3. 모든 속성이 올바르게 설정되었는지 검증하고, 객체를 사용 가능한 상태로 만드는 **초기화 메서드(`afterPropertiesSet`)**를 호출한다.

이 모든 과정은 **`MyBatisCursorItemReader`라는 구체 클래스의 내부 구조와 생명주기를 정확히 알고 있어야만** 완벽하게 수행될 수 있다.

### 3.2. Spring의 프록시와 타입 정보 손실

Spring은 종종 AOP 등을 적용하기 위해 Bean을 프록시(Proxy) 객체로 감싸서 컨테이너에 등록한다.

이때 Spring은 `@Bean` 메서드에 선언된 반환 타입을 중요한 참고 자료로 사용한다.

만약 반환 타입을 `ItemReader`로 지정하면, Spring은 "아, 이 Bean의 본질은 `ItemReader`구나" 라고 생각하고, `ItemReader` 인터페이스에 선언된 메서드만을 기준으로 프록시를 만들 수 있다.

이 과정에서 **`MyBatisCursorItemReader`만이 가진 고유한 특성이나, 빌더가 수행해야 했던 정교한 초기화 로직의 일부가 무시되거나 누락될 가능성**이 생긴다.

특히 `MyBatisCursorItemReader`처럼 내부적으로 커서(Cursor)를 열고 닫는 등 **상태에 민감한(stateful) 객체**는, 프레임워크가 타입을 오해하여 잘못된 최적화나 프록시 처리를 적용했을 때 치명적인 오작동을 일으킬 수 있다.

## 4. 베스트 프랙티스: "생성은 구체적으로, 사용은 추상적으로"

이 모든 혼란을 피하고, 안정성과 유연성을 모두 잡는 방법은 간단하다. 생성과 사용의 시점에 따라 다른 전략을 취하는 것이다.

### **생성 (Configuration): 반환 타입은 가장 구체적인 클래스로**

`@Bean` 메서드에서는, Spring 컨테이너와 빌더가 Bean을 완벽하게 이해하고 설정할 수 있도록 **가장 구체적인 클래스 타입을 반환형으로 지정**한다.

```java
// MyBatchConfiguration.java
@Configuration
public class MyBatchConfiguration {

    @Bean
    // Spring에게 "이 Bean은 MyBatisCursorItemReader 타입입니다!" 라고 명확히 알려준다.
    public MyBatisCursorItemReader<MyDto> mybatisItemReader(SqlSessionFactory sqlSessionFactory) {
        return new MyBatisCursorItemReaderBuilder<MyDto>()
                .sqlSessionFactory(sqlSessionFactory)
                .queryId("my.query.id")
                .build();
    }
}
```

- **효과:** Spring은 이 Bean의 정확한 타입을 인지하고, 프록시 생성이나 초기화 과정에서 발생할 수 있는 잠재적 문제를 피한다. 빌더 역시 자신의 목표 타입이 명확하므로 모든 설정 작업을 안정적으로 수행할 수 있다.

### **사용 (Service, Step): 의존성은 인터페이스로**

의존성을 주입받아 사용할 때는, 다시 객체지향의 기본 원칙으로 돌아와 **상위 인터페이스 타입으로 선언**한다.

```java
// MyStepConfiguration.java
@Configuration
@RequiredArgsConstructor
public class MyStepConfiguration {
    // 사용할 때는 인터페이스 타입으로 주입받아 유연성을 확보한다.
    private final ItemReader<MyDto> mybatisItemReader;

    // ... Step 정의 로직 ...
}
```

- **효과:** `MyStepConfiguration`은 `mybatisItemReader`의 구체적인 구현체가 `Cursor` 방식인지, `Paging` 방식인지 알 필요가 없다. 나중에 `ItemReader`의 구현체를 교체하더라도 이 클래스는 수정할 필요가 없으므로, 유연하고 테스트하기 쉬운 코드가 유지된다.

## 5. 결론

다형성은 객체지향의 강력한 무기지만, 사용하는 '시점'과 '맥락'을 이해하는 것이 중요하다.

- **의존성을 주입받아 '사용'할 때:** 인터페이스에 의존하여 유연성을 극대화하라.
- **Bean을 '생성'하여 컨테이너에 등록할 때:** 구체 클래스를 반환 타입으로 지정하여, 프레임워크가 객체를 완벽하게 이해하고 안정적으로 설정할 수 있도록 하라.

`@Bean` 메서드의 반환 타입을 구체 클래스로 명시하는 것은 다형성 원칙을 위배하는 것이 아니라, **Spring 프레임워크와 소통하는 또 다른 방식**이다. "이 Bean을 만들고 초기화하는 데 필요한 모든 정보는 이 구체 클래스 안에 담겨 있습니다"라고 명확히 알려주는, 더 안전한 코드를 위한 현명한 선택이다.

---
