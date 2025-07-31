---
title: 무한 스트림 만들기 - `iterate` vs `generate`, 언제 무엇을 써야 할까?
description: 
author: laze
date: 2025-07-31 00:00:01 +0900
categories: [Dev, Java]
tags: [Java, Stream, iterate, generate]
---
# [Java Stream] 무한 스트림 만들기: `iterate` vs `generate`, 언제 무엇을 써야 할까?

## 1. 서론: 무한(Unbounded) 스트림의 매력

Java Stream API는 `List`나 `Set`처럼 이미 존재하는 컬렉션으로부터 스트림을 만드는 것 외에도, **무한한(unbounded)** 데이터의 스트림을 생성하는 두 가지 정적 팩토리 메서드, `Stream.iterate`와 `Stream.generate`를 제공합니다.

'무한'이라는 말에 겁먹을 필요는 없습니다. 이 스트림들은 **게으른 연산(Lazy Evaluation)** 방식으로 동작하여, 최종 연산에서 필요하다고 요청할 때마다 값을 하나씩 생성합니다. 따라서 `limit()`와 같은 단락(short-circuiting) 연산자와 함께 사용하면, 메모리 걱정 없이 원하는 만큼의 데이터 시퀀스를 만들어낼 수 있습니다.

```java
// 0, 2, 4, 6, 8 ...
Stream.iterate(0, n -> n + 2).limit(5).forEach(System.out::println);

// 5개의 랜덤 숫자
Stream.generate(Math::random).limit(5).forEach(System.out::println);
```

두 메서드 모두 무한 스트림을 만들지만, 그 내부 철학과 사용 용도는 완전히 다릅니다.

"둘 다 비슷한 것 같은데, 언제 무엇을 써야 할까?" 이 글에서는 이 질문에 대한 명확한 답을 찾아보겠습니다.

## 2. 핵심 기준: '이전 값'에 의존하는가?

두 메서드를 구분하는 가장 중요한 기준은 바로 **'상태 의존성(State Dependency)'**입니다.

- **`iterate`:** 이전 값을 바탕으로 다음 값을 **계산**한다. (상태 의존적)
- **`generate`:** 이전 값과 상관없이 각 값을 **독립적으로** 생성한다. (상태 비의존적)

이 차이점이 두 메서드의 사용처를 결정합니다.

## 3. `Stream.iterate()`: 순차적인 '도미노' 스트림

- **핵심 철학:** **"이전 요소를 바탕으로 다음 요소를 계산한다."**
- **시그니처:** `static <T> Stream<T> iterate(T seed, UnaryOperator<T> f)`
- **동작 원리:**
  1. `seed` (초기값)를 스트림의 첫 번째 요소로 사용한다. (`stream[0] = seed`)
  2. 두 번째 요소는 `f(seed)`를 통해 계산한다. (`stream[1] = f(stream[0])`)
  3. 세 번째 요소는 `f(두 번째 요소)`를 통해 계산한다. (`stream[2] = f(stream[1])`)
  4. 이 과정을 계속 반복한다.
- **비유:** **도미노.** 첫 번째 도미노(`seed`)가 넘어지면, 그 힘이 다음 도미노를 넘어뜨리고, 그 다음 도미노를 넘어뜨리는 것처럼 앞선 결과가 다음 결과에 필연적으로 영향을 줍니다. 각 요소는 순차적으로 생성될 수밖에 없습니다.

### Best Cases: `iterate`는 이럴 때 사용하세요

`iterate`는 이전 결과에 기반하여 다음 값을 예측할 수 있는 순차적인 데이터 시퀀스를 만드는 데 이상적입니다.

### 예제 1: 짝수 무한 스트림 생성

```java
Stream.iterate(0, n -> n + 2)
      .limit(10)
      .forEach(System.out::println); // 0, 2, 4, 6, 8, 10, 12, 14, 16, 18
```

### 예제 2: 피보나치 수열 생성

피보나치 수열은 `F(n) = F(n-1) + F(n-2)`로 정의되므로, 이전 두 항의 '상태'가 반드시 필요합니다. `iterate`는 이 상태를 배열(`int[]`)을 통해 완벽하게 표현할 수 있습니다.

```java
Stream.iterate(new int[]{0, 1}, t -> new int[]{t[1], t[0] + t[1]})
      .limit(10)
      .map(t -> t[0]) // 각 배열에서 첫 번째 요소만 추출
      .forEach(System.out::println); // 0, 1, 1, 2, 3, 5, 8, 13, 21, 34
```

## 4. `Stream.generate()`: 독립적인 '주사위' 스트림

- **핵심 철학:** **"각 요소는 독립적으로, 이전 요소와 아무 상관 없이 생성된다."**
- **시그니처:** `static <T> Stream<T> generate(Supplier<T> s)`
- **동작 원리:** 스트림에서 요소가 필요할 때마다, 주어진 `Supplier<T>`(공급자)를 반복적으로 호출하여 새로운 값을 생성합니다.
- **비유:** **주사위 던지기.** 이전에 어떤 숫자가 나왔든, 다음에 던질 주사위의 결과에는 아무런 영향을 주지 않습니다. 모든 시도는 완전히 독립적입니다.

### Best Cases: `generate`는 이럴 때 사용하세요

`generate`는 각 요소가 이전 요소와 아무런 관계가 없는 데이터를 생성할 때 적합합니다. 각 요소가 독립적이므로 **병렬 처리(`parallel()`)**에 매우 유리합니다.

### 예제 1: 상수 스트림 생성

```java
Stream.generate(() -> "Hello World")
      .limit(3)
      .forEach(System.out::println);
// Hello World
// Hello World
// Hello World
```

### 예제 2: 난수 스트림 생성

```java
Stream.generate(Math::random)
      .limit(5)
      .forEach(System.out::println);
// 0.123...
// 0.987...
// ... (매번 다른 5개의 난수)
```

### 예제 3: 새로운 객체 스트림 생성

```java
Stream.generate(User::new)
      .limit(5)
      .forEach(System.out::println);
// User@1f32e575
// User@2ff4f00f
// ... (5개의 새로운 User 객체)
```

## 5. 한눈에 보는 비교 테이블

| 구분 | `Stream.iterate(seed, f)` | `Stream.generate(s)` |
| --- | --- | --- |
| **핵심 철학** | 이전 값으로 다음 값 계산 (상태 의존적) | 각 값을 독립적으로 생성 (상태 비의존적) |
| **파라미터** | `T seed` (초기값), `UnaryOperator<T> f` | `Supplier<T> s` (공급자) |
| **병렬 처리** | 순차적 의존성 때문에 병렬화 어려움 | 각 요소가 독립적이므로 병렬화에 **매우 적합** |
| **주요 용도** | 수열, 순차적 상태 변화, 재귀적 구조 | 상수, 난수, 독립적인 객체 생성 |

## 6. `generate`의 함정: 상태를 가진 `Supplier`의 위험성

`generate`에 `Supplier`를 구현하면서 내부에 상태(필드)를 가지게 만들 수도 있습니다. 예를 들어, 피보나치 수열을 `generate`로 구현하면 다음과 같습니다.

```java
// 안티 패턴: generate에 상태를 가진 Supplier 사용
IntSupplier fib = new IntSupplier() {
    private int previous = 0;
    private int current = 1;

    public int getAsInt() {
        int oldPrevious = this.previous;
        int nextValue = this.previous + this.current;
        this.previous = this.current;
        this.current = nextValue;
        return oldPrevious;
    }
};

IntStream.generate(fib).limit(10).forEach(System.out::println);
```

이 코드는 순차 실행 시에는 올바르게 동작하는 것처럼 보입니다.

하지만 이 스트림을 **병렬로 실행(`parallel()`)**하면 어떻게 될까요? 여러 스레드가 `fib`라는 단 하나의 인스턴스에 동시에 접근하여 `previous`와 `current` 값을 마구잡이로 변경하려고 시도할 것입니다.

이는 **데이터 경쟁(Data Race)**을 유발하여 결과가 완전히 깨지게 됩니다.

**"상태를 기반으로 순차적인 값을 생성하고 싶다면, `iterate`가 훨씬 더 안전하고 올바른 선택이다."**

## 7. 결론: 올바른 도구를 올바른 문제에 사용하기

`iterate`와 `generate`는 무한 스트림을 생성하는 강력한 도구지만, 그 용도는 명확하게 구분됩니다. 선택의 기준은 간단합니다.

- **연속적인 값, 수열처럼 이전 항의 결과가 다음 항에 영향을 주는가?**
  - → **`iterate`*를 사용하라.
- **각 값이 완전히 독립적이고, 이전 항과 아무런 관련이 없는가?**
  - → **`generate`*를 사용하라. (특히 병렬 처리가 필요할 때)

두 메서드의 철학적 차이를 이해하고 문제의 성격에 맞는 도구를 선택한다면, 더 안전하고 효율적인 스트림 코드를 작성할 수 있을 것입니다.
