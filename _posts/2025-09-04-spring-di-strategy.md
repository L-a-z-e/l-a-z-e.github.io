---
title: Docker/Gitlab CI 트러블슈팅 - Pipeline Cancel 후 네트워크가 죽는 현상
description: 
author: laze
date: 2025-09-04 00:00:01 +0900
categories: [Dev, SpringBoot]
tags: [SpringBoot, DI, StrategyPattern]
---
# Spring DI  `Map`에 Bean이 자동으로 주입되는 마법- 전략 패턴의 완성

## 1. 서론: "저는 `Map.put()`을 호출한 적이 없는데요?"

Spring 프레임워크로 개발을 하다 보면, 때로 마법처럼 느껴지는 순간들을 마주하게 됩니다. 그중 하나가 바로 `Map`에 Bean이 자동으로 주입되는 현상입니다.

```java
@Service
public class MyStrategyService {

    private final Map<String, StrategyInterface> strategyMap;

    // 생성자에 Map<String, StrategyInterface>를 선언했을 뿐인데...
    public MyStrategyService(Map<String, StrategyInterface> strategyMap) {
        this.strategyMap = strategyMap;
    }

    public void execute(String type) {
        // 이 Map 안에 어떻게 모든 Strategy Bean들이 들어가 있을까?
        StrategyInterface strategy = strategyMap.get(type);
        strategy.run();
    }
}
```

분명히 `Map`에 `put()`을 호출한 적이 없는데도, 애플리케이션이 실행되면 `strategyMap` 안에는 해당 인터페이스를 구현한 모든 Bean들이 신기하게도 가득 채워져 있습니다.

이것은 버그나 우연이 아닙니다. Spring의 의존성 주입(DI) 컨테이너가 제공하는, 매우 강력하고 의도된 기능입니다. 이 글에서는 이 '마법'의 원리를 파헤치고, 이 기능을 활용하여 어떻게 `if-else` 지옥에서 벗어나 우아한 **전략 패턴(Strategy Pattern)**을 완성할 수 있는지 심층적으로 알아봅니다.

## 2. Spring DI의 기본 원리: 타입, 그리고 이름

이 마법을 이해하기 위한 첫걸음은 Spring IoC 컨테이너가 의존성을 주입하는 기본 원리를 아는 것입니다.

1. **Bean 수집:** 애플리케이션이 시작될 때, Spring은 `@Component`, `@Service`, `@Bean` 등이 붙은 모든 클래스를 스캔하여 객체(Bean)를 생성하고, 자신의 내부 저장소(컨테이너)에 보관합니다. 이때 각 Bean은 고유한 **이름(Bean Name)**과 **타입(Type)**을 갖게 됩니다.
2. **의존성 주입:** 어떤 클래스가 생성자를 통해 의존성을 요청하면, Spring 컨테이너는 주입할 Bean을 찾습니다.
  - **1순위 (타입 매칭):** 먼저, 요청된 **타입과 일치하는** Bean을 찾습니다.
  - **2순위 (이름 매칭):** 만약 동일한 타입의 Bean이 여러 개 발견되어 모호한 경우, 이번에는 **변수 이름과 일치하는** Bean 이름을 가진 것을 찾아 주입합니다.

## 3. 마법의 정체: 컬렉션 타입에 대한 특별 대우

Spring은 `List<T>`나 `Map<String, T>`와 같은 특정 컬렉션 타입의 의존성 주입을 요청받으면, 위 기본 원리를 확장하여 매우 특별한 방식으로 동작합니다.

### `List<MyInterface>`를 주입받을 때

- **Spring의 해석:** "개발자가 `MyInterface` 타입의 Bean을 **'전부 다(all)'** 달라고 하는구나!"
- **동작:** 컨테이너 내부를 모두 뒤져서, `MyInterface` 타입으로 할당 가능한 모든 Bean을 찾아서 `List`에 담아 주입합니다. `@Order` 어노테이션을 통해 주입되는 순서를 제어할 수도 있습니다.

### `Map<String, MyInterface>`를 주입받을 때

- **Spring의 해석:** "개발자가 `MyInterface` 타입의 Bean을 **'전부 다'** 달라고 하는데, **'이름표(Bean 이름)'와 함께** 달라고 하는구나!"
- **동작:**
  1. 컨테이너 내부를 모두 뒤져서, `MyInterface` 타입으로 할당 가능한 모든 Bean을 찾습니다.
  2. 찾아낸 각 Bean에 대해, **Bean의 이름(ID)을 Key**로, **Bean 객체 인스턴스를 Value**로 하는 `Map`을 동적으로 생성합니다.
  3. 이 `Map`을 요청한 곳에 주입합니다.

이제 아래 코드가 어떻게 동작하는지 명확해집니다.

**`StrategyConfig.java` (Bean 등록)**

```java
@Configuration
public class StrategyConfig {

    @Bean(name = "strategyA") // Bean 이름을 "strategyA"로 명시
    public StrategyInterface strategyA() {
        return new StrategyAImpl();
    }

    @Bean(name = "strategyB") // Bean 이름을 "strategyB"로 명시
    public StrategyInterface strategyB() {
        return new StrategyBImpl();
    }
}
```

**`MyStrategyService.java` (의존성 주입)**

```java
@Service
public class MyStrategyService {
    private final Map<String, StrategyInterface> strategyMap;

    public MyStrategyService(Map<String, StrategyInterface> strategyMap) {
        this.strategyMap = strategyMap;
    }
    // ...
}
```

Spring은 `MyStrategyService`를 생성할 때, `Map<String, StrategyInterface>` 주입 요청을 보고, 컨테이너에 등록된 모든 `StrategyInterface` 타입의 Bean들을 찾아 다음과 같은 `Map`을 **자동으로 만들어** 주입합니다.

```java
// Spring이 MyStrategyService에 주입하는 Map의 실제 형태
{
  "strategyA": (StrategyAImpl 객체 인스턴스),
  "strategyB": (StrategyBImpl 객체 인스턴스)
}
```

이것이 바로 "내가 `put`한 적이 없는데 `Map`에 모든 Bean이 담겨 들어오는" 마법의 정체입니다.

## 4. 실전 활용: 전략 패턴(Strategy Pattern)의 우아한 완성

이 기능은 단순히 여러 Bean을 주입받는 것을 넘어, `if-else`나 `switch` 분기문을 제거하고 코드를 유연하게 만드는 **전략 패턴**을 매우 우아하게 구현하는 데 사용됩니다.

### Before: `if-else` 지옥과 OCP 위반

- **상황:** 결제 타입(`"CARD"`, `"CASH"`, `"POINT"`)에 따라 각기 다른 처리 로직을 실행해야 합니다.

```java
@Service
public class BadPaymentService {
    public void processPayment(String paymentType, long amount) {
        if ("CARD".equals(paymentType)) {
            // 신용카드 결제 로직...
            System.out.println(amount + "원, 카드로 결제합니다.");
        } else if ("CASH".equals(paymentType)) {
            // 현금 결제 로직...
            System.out.println(amount + "원, 현금으로 결제합니다.");
        } else if ("POINT".equals(paymentType)) {
            // 포인트 결제 로직...
            System.out.println(amount + "포인트, 포인트로 결제합니다.");
        } else {
            throw new IllegalArgumentException("지원하지 않는 결제 방식입니다.");
        }
    }
}
```

이 코드는 새로운 결제 방식(`"GIFT_CARD"`)이 추가될 때마다 `BadPaymentService` 클래스의 코드를 직접 수정해야 합니다. 이는 **OCP(개방-폐쇄 원칙: 확장에 열려있고, 수정에 닫혀있어야 한다)**를 위반하는 경직된 설계입니다.

### After: 전략 패턴과 `Map` 자동 주입으로 리팩토링

**1단계: 전략 인터페이스 정의**

```java
public interface PaymentStrategy {
    void pay(long amount);
    String getType(); // 자신의 타입을 반환하는 메서드 추가
}

```

**2단계: 각 로직을 별도의 전략 클래스(Bean)로 구현**

```java
@Component // Bean으로 등록
public class CardPaymentStrategy implements PaymentStrategy {
    @Override
    public void pay(long amount) {
        System.out.println(amount + "원, 카드로 결제합니다.");
    }
    @Override
    public String getType() { return "CARD"; }
}

@Component
public class CashPaymentStrategy implements PaymentStrategy {
    @Override
    public void pay(long amount) {
        System.out.println(amount + "원, 현금으로 결제합니다.");
    }
    @Override
    public String getType() { return "CASH"; }
}

// ... PointPaymentStrategy 등 추가

```

**3.단계: `Map`으로 모든 전략을 주입받고, 선택하여 사용**

```java
@Service
public class GoodPaymentService {
    private final Map<String, PaymentStrategy> strategyMap;

    // 생성자에서 Map<String, PaymentStrategy>를 요청하면,
    // Spring이 "cardPaymentStrategy" -> CardPaymentStrategy 객체,
    // "cashPaymentStrategy" -> CashPaymentStrategy 객체...
    // 와 같은 Map을 자동으로 주입해준다.
    // (Bean 이름은 클래스 이름의 첫 글자를 소문자로 바꾼 것이 기본값)
    public GoodPaymentService(Map<String, PaymentStrategy> strategyMap) {
        this.strategyMap = strategyMap;
    }

    public void processPayment(String paymentType, long amount) {
        // if-else 없이, paymentType에 해당하는 Bean 이름(e.g., "cardPaymentStrategy")을 찾아 전략을 선택
        // 실제로는 getType()으로 만든 Map을 사용하는 것이 더 좋다. (아래 코드 참조)
        String strategyBeanName = paymentType.toLowerCase() + "PaymentStrategy";
        PaymentStrategy strategy = strategyMap.get(strategyBeanName);

        if (strategy == null) {
            throw new IllegalArgumentException("지원하지 않는 결제 방식입니다.");
        }
        strategy.pay(amount);
    }
}
```

> Tip: Bean 이름 대신 paymentType 자체를 Key로 사용하고 싶다면, GoodPaymentService의 생성자를 다음과 같이 수정할 수 있습니다.
>
>
> ```java
> private final Map<String, PaymentStrategy> strategyMap;
> 
> public GoodPaymentService(List<PaymentStrategy> strategies) {
>     this.strategyMap = strategies.stream()
>             .collect(Collectors.toMap(PaymentStrategy::getType, Function.identity()));
> }
> ```
>
> 이 방식은 `List`로 모든 전략을 받은 뒤, `getType()`의 반환 값을 키로 하는 `Map`으로 직접 변환하여 더 유연하게 대처할 수 있습니다.
>

이 구조의 가장 큰 장점은, 새로운 결제 방식(`GiftCardPaymentStrategy`)이 추가되더라도 `GoodPaymentService`의 코드는 **단 한 줄도 수정할 필요가 없다**는 것입니다. 그저 새로운 `@Component` 클래스를 추가하기만 하면, Spring이 알아서 `Map`에 포함시켜 주기 때문입니다. 이것이 바로 OCP를 만족하는 유연하고 확장 가능한 설계입니다.

## 5. 결론: Spring DI, 알고 쓰면 더 강력하다

Spring의 `Map` 자동 주입 기능은 단순히 여러 Bean을 한 번에 받는 편리한 기능을 넘어, **전략 패턴과 같은 핵심 디자인 패턴을 매우 우아하게 구현할 수 있도록 돕는** 강력한 도구입니다.

`if-else`나 `switch`가 반복적으로 나타나는 코드를 발견했다면, 이 '마법' 같은 기능을 활용하여 더 유연하고, 확장 가능하며, 유지보수하기 쉬운 애플리케이션을 설계해 보시기 바랍니다.
