---
title: Stream flatMap & boxed 데이터 타입 흐름 추적하기
description: 
author: laze
date: 2025-07-25 00:00:11 +0900
categories: [Dev, Java]
tags: [Java, Stream, Data Type, flatMap, map]
---
# [Java Stream 심층 분석] `flatMap`과 `boxed`의 미로: 복잡한 데이터 타입 흐름 추적하기

## 1. 서론: Stream, 한 줄인데 왜 이렇게 어렵지?

Java 8의 Stream API는 컬렉션 처리를 위한 강력하고 우아한 방법을 제공합니다. `map`, `filter` 같은 간단한 연산들은 직관적이지만, `flatMap`과 `IntStream` 같은 기본형 스트림이 섞이기 시작하면, 단 한 줄의 코드임에도 불구하고 그 흐름을 따라가기가 매우 어려워집니다.

다음은 "피타고라스 수"(a² + b² = c²을 만족하는 세 자연수 a,b,c)를 찾는, 전형적인 중첩 스트림 예제입니다.

```java
Stream<int[]> pythagoreanTriples = IntStream.rangeClosed(1, 100).boxed()
    .flatMap(a -> IntStream.rangeClosed(a, 100)
        .filter(b -> Math.sqrt(a * a + b * b) % 1 == 0).boxed()
        .map(b -> new int[] { a, b, (int) Math.sqrt(a * a + b * b) }));
```

이 코드를 처음 보고 "각 `.`(점) 연산자를 지날 때마다 데이터의 타입이 어떻게 변하는지" 한눈에 파악하기란 쉽지 않습니다.

이 글에서는 이처럼 복잡한 Stream 코드의 미로를 헤쳐나가기 위해, 데이터의 '형태' 변화를 단계별로 추적하는 방법을 심층적으로 분석해 보겠습니다.

## 2. 기초 다지기: 두 개의 스트림 세계와 그 사이의 다리

Java Stream API에는 크게 두 종류의 스트림 세계가 존재합니다.

- **객체형 스트림 (`Stream<T>`):** `Stream<Integer>`, `Stream<String>`, `Stream<User>`처럼 제네릭(`T`)을 사용하여 모든 종류의 **'객체'**를 다룹니다.
- **기본형 특화 스트림 (`IntStream`, `LongStream`, `DoubleStream`):** `int`, `long`, `double`과 같은 **기본형(primitive type) 데이터**를 다루기 위해 특별히 설계되었습니다. `sum()`, `average()` 같은 숫자 특화 연산을 제공하며, 불필요한 객체 포장(Boxing/Unboxing)을 피할 수 있어 성능상 이점이 있습니다.

이 두 세계를 오가는 다리 역할을 하는 메서드가 있습니다.

- **`.boxed()`:** 기본형 스트림 → 객체형 스트림 (`IntStream` -> `Stream<Integer>`)
- **`.mapToInt()` / `.mapToLong()` / `.mapToDouble()`:** 객체형 스트림 → 기본형 스트림 (`Stream<Integer>` -> `IntStream`)

이 '다리'가 언제 놓이는지를 아는 것이 타입 흐름을 추적하는 첫걸음입니다.

## 3. `flatMap`의 본질: "봉지를 까서 내용물만 꺼내줘"

`flatMap`은 `map`과 비슷해 보이지만, 그 역할은 근본적으로 다릅니다.

- **`map(function)`:** 스트림의 각 요소에 함수를 적용하여 **1:1로 변환**합니다. `Stream<T>`는 `Stream<R>`이 됩니다. 스트림의 개수는 변하지 않습니다.
  - **비유:** 계란(`T`) 10개를 받아서, 각각을 계란 후라이(`R`) 10개로 만드는 것.
- **`flatMap(function)`:** 스트림의 각 요소에 함수를 적용하여 **새로운 스트림**을 생성한 뒤, **생성된 모든 스트림들을 하나의 평평한 스트림으로 이어붙입니다.**
  - **비유:** 여러 개의 계란판(`Stream<List<Egg>>`)을 받아서, 모든 계란판을 까고 **계란들만(`Stream<Egg>`)** 하나의 큰 바구니에 담는 것. 즉, '평탄화(flattening)' 작업을 수행합니다.

`flatMap`은 Stream으로 이중 `for` 루프와 같은 효과를 낼 때 주로 사용됩니다.

## 4. 실전 코드 추적: 피타고라스 수를 찾는 여정

이제 처음 봤던 피타고라스 코드의 데이터 타입 흐름을 각 단계별로 추적해 보겠습니다.

```java
// 최종 목표 타입: Stream<int[]>
Stream<int[]> pythagoreanTriples =

    // [Step 1] IntStream: 1, 2, 3, ..., 100
    // int 값들이 흐르는 기본형 스트림
    IntStream.rangeClosed(1, 100)

    // [Step 2] Stream<Integer>: Integer(1), Integer(2), ...
    // 각 int를 Integer 객체로 포장. 객체형 스트림으로 전환.
    // flatMap은 객체형 스트림에서 동작하기 때문.
    .boxed()

    // [Step 3] Stream<int[]>: 최종적으로 int[] 배열들이 흐르는 스트림
    // Stream<Integer>의 각 요소 a에 대해, 새로운 스트림을 생성하고 그 결과들을 모두 합친다.
    .flatMap(a -> // a의 타입은 Integer. (예: a = 3)

        // --- flatMap 내부의 람다 시작 (반드시 Stream을 반환해야 함) ---

        // [Step 3a] IntStream: 3, 4, 5, ..., 100
        // a부터 100까지의 int 값들이 흐르는 내부 스트림 생성
        IntStream.rangeClosed(a, 100)

        // [Step 3b] IntStream: 4, ... (피타고라스 조건을 만족하는 b만 남음)
        // a와 조합하여 피타고라스 수가 되는 b만 필터링
        .filter(b -> Math.sqrt(a * a + b * b) % 1 == 0)

        // [Step 3c] Stream<Integer>: Integer(4), ...
        // 필터링된 int 값 b들을 Integer 객체로 포장
        .boxed()

        // [Step 3d] Stream<int[]>: Stream 내부에 {3, 4, 5} 배열이 담김
        // 각 Integer b를 사용하여, a, b, c로 구성된 int[] 배열을 생성.
        // 이 map 연산의 결과로 Stream<int[]>가 생성된다.
        .map(b -> new int[] { a, b, (int) Math.sqrt(a * a + b * b) })

        // --- flatMap 내부의 람다 종료 ---
    );

```

`flatMap`은 `a`가 `3`일 때 생성된 `Stream` (원소: `{3,4,5}`), `a`가 `5`일 때 생성된 `Stream` (원소: `{5,12,13}`) 등, `a`값 마다 생성된 모든 `Stream<int[]>`들을 모두 까서, 내용물인 `int[]` 배열들만 모아 **하나의 최종 `Stream<int[]>`**를 만들어냅니다.

## 5. 더 나은 코드를 위한 리팩토링: `mapToObj` 활용

위 코드의 `flatMap` 내부를 보면 `.filter().boxed().map()` 이라는 3단계의 다소 번잡한 과정이 있습니다. `IntStream`은 이 과정을 더 효율적으로 처리할 수 있는 `mapToObj` 메서드를 제공합니다.

- `mapToObj(IntToObjectFunction mapper)`: `IntStream`의 각 `int` 요소를 받아서 **객체(`Object`)로 변환**하는, `map`의 기본형 스트림 버전입니다.

```java
Stream<int[]> pythagoreanTriples2 = IntStream.rangeClosed(1, 100).boxed()
    .flatMap(a -> IntStream.rangeClosed(a, 100)
        // 1. int b를 받아 double[] 배열(객체)로 변환
        .mapToObj(b -> new double[]{a, b, Math.sqrt(a * a + b * b)})
        // 2. double[] 배열을 받아 c가 정수인지 필터링
        .filter(t -> t[2] % 1 == 0))
    // 3. 필터링된 Stream<double[]>을 받아 최종 Stream<int[]>로 변환
    .map(array -> Arrays.stream(array).mapToInt(d -> (int) d).toArray());

```

이 두 번째 접근 방식은 계산 순서를 변경하여, 불필요한 `boxed()` 호출을 줄이고 `double` 타입으로 중간 계산을 수행하여 가독성과 효율성을 약간 개선한 버전입니다.

## 6. 복잡한 타입을 다루는 팁: `var`와 메서드 분리

### 팁 1: `var` 활용 (Java 10+)

복잡한 Stream의 최종 타입을 직접 작성하는 것은 번거롭고 실수하기 쉽습니다. `var` 키워드를 사용하면 컴파일러에게 타입 추론을 위임할 수 있습니다.

```java
var pythagoreanTriples = IntStream.rangeClosed(1, 100).boxed()
        .flatMap( ... ); // 가독성 향상!
```

### 팁 2: 메서드 분리

`flatMap` 내부의 복잡한 람다 블록은 가독성을 크게 해칩니다. 이를 별도의 private 메서드로 추출하면, 코드가 훨씬 깔끔해지고 테스트하기도 쉬워집니다.

```java
public Stream<int[]> findPythagoreanTriples() {
    return IntStream.rangeClosed(1, 100).boxed()
            .flatMap(this::findTriplesForA);
}

private Stream<int[]> findTriplesForA(int a) {
    return IntStream.rangeClosed(a, 100)
            .filter(b -> Math.sqrt(a * a + b * b) % 1 == 0)
            .mapToObj(b -> new int[]{a, b, (int) Math.sqrt(a * a + b * b)});
}
```

## 7. 결론

복잡한 Stream 코드를 읽는 핵심은, 각 연산자(`.`)를 경계로 **데이터의 '형태'와 '타입'이 어떻게 변하는지**를 차분히 추적하는 능력입니다.

- **`boxed()`**: 기본형의 세계에서 객체의 세계로 넘어가는 다리.
- **`flatMap()`**: 여러 개의 봉지(스트림)를 까서 내용물만 하나의 큰 봉지에 담는 평탄화 작업.

이 두 가지 핵심 개념과 각 연산자의 역할을 명확히 이해한다면, 어떤 복잡한 스트림이라도 자신감 있게 분석하고 작성할 수 있게 될 것입니다.
