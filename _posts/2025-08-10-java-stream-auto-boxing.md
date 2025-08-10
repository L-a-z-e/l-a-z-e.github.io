---
title: Java Stream 문제 해결 - map 연산 후의 타입 에러, 오토박싱의 함정을 파헤치다
description: 
author: laze
date: 2025-08-10 00:00:02 +0900
categories: [Dev, Git]
tags: [Git, Staging, Tracked, Untracked]
---
# [Java Stream 문제 해결] `map` 연산 후의 타입 에러, 오토박싱의 함정을 파헤치다

## 1. 서론: "분명히 객체였는데, 왜 메서드를 못 찾지?"

Java Stream API는 코드를 간결하고 우아하게 만들어주는 강력한 도구입니다. 하지만 그 편리함 이면에는 데이터의 '타입 흐름'이라는 중요한 개념이 숨어있습니다. 이를 간과하면, 우리는 종종 다음과 같은 당황스러운 컴파일 에러와 마주하게 됩니다.

```java
// Point 클래스 정의
class Point {
    private int x, y;
    // getter, constructor...
    public int getX() { return x; }
}

List<Point> points = Arrays.asList(new Point(10, 20), new Point(30, 40));

points.stream()
      .map(Point::getX)
      .forEach(p -> System.out.println(p.getX())); // <-- COMPILE ERROR!
```

에러 메시지는 명확합니다: `cannot find symbol: method getX() in Integer`.
`p`가 `Integer` 타입이라서 `getX()` 메서드를 찾을 수 없다는 뜻입니다. 하지만 우리는 분명히 `Stream<Point>`에서 시작했는데, 왜 갑자기 `p`가 `Integer`가 되어버린 걸까요?

이 글에서는 이 문제의 원인이 되는 **`map` 연산자의 '변환' 특성**과 Java의 **'오토박싱(Autoboxing)'**을 심층적으로 분석하고, Stream의 타입 흐름을 추적하여 안정적인 코드를 작성하는 방법을 알아봅니다.

## 2. Stream 파이프라인: 데이터는 어떻게 흘러가는가?

Stream을 '데이터가 흐르는 파이프라인'에 비유할 수 있습니다. 데이터는 파이프라인의 각 단계를 거치면서 그 '형태'와 '타입'이 변할 수 있습니다.

**`stream()` (생성) → `filter`, `map` (중간 연산) → `forEach`, `collect` (최종 연산)**

문제의 원인을 찾으려면, 우리 코드의 파이프라인 각 단계에서 데이터가 어떤 타입으로 흐르고 있는지 추적해야 합니다.

## 3. 문제 분석: `map`은 '변환기'다

`map` 연산자는 스트림 파이프라인에서 가장 중요한 '변환기' 역할을 합니다. 스트림의 각 요소를 받아서, 다른 형태나 타입의 요소로 변환하여 다음 단계로 흘려보냅니다.

이제 문제의 코드를 단계별로 다시 추적해 봅시다.

**`List<Point> points = Arrays.asList(new Point(12, 2), null);`**

**`[Stage 1]` `points.stream()`**

- **흐르는 데이터 타입:** `Stream<Point>`
- **설명:** 이 파이프라인에는 `Point` 객체(또는 `null`)들이 흐르기 시작합니다.

**`[Stage 2]` `.map(Point::getX)`**

- **`map`의 역할:** `map`은 `Function<T, R>`을 인자로 받아, `T`타입의 각 요소를 `R`타입으로 변환합니다.
- **`Point::getX`:** 이 메서드 참조는 `(Point p) -> p.getX()` 람다와 동일합니다. `Point` 객체(`T`)를 받아서 **`int` 타입**의 x좌표 값을 반환하는 함수입니다.
- **오토박싱(Autoboxing) 발생:** Stream은 제네릭(`Stream<T>`)을 기반으로 동작하므로, 원시 타입(primitive type)인 `int`를 직접 담을 수 없습니다. 따라서 Java는 이 `int` 값을 자동으로 그것의 래퍼 클래스(wrapper class)인 **`Integer` 객체**로 변환(포장)합니다. 이것이 바로 **오토박싱**입니다.
- **결과 데이터 타입:** `Stream<Integer>`
- **설명:** `map` 연산을 통과한 스트림에는 더 이상 `Point` 객체가 흐르지 않습니다. 대신, 각 `Point` 객체의 x좌표 값이 담긴 **`Integer` 객체**들이 흐르고 있습니다.

**`[Stage 3]` `.forEach(p -> System.out.println(p.getX()))`**

- **`p`의 타입:** `forEach`는 이전 단계로부터 `Stream<Integer>`를 전달받았습니다. 따라서 `forEach` 람다의 파라미터 `p`는 **`Integer` 타입**이 됩니다.
- **컴파일 에러의 원인:** `Integer` 클래스에는 `getX()`라는 메서드가 존재하지 않습니다. 개발자는 `p`가 `Point`일 것이라고 착각했지만, `map` 변환기를 거치며 이미 `Integer`로 바뀌었기 때문에 컴파일러는 "그런 메서드는 찾을 수 없다"고 알려주는 것입니다.

## 4. 또 다른 함정: `NullPointerException`

위 코드는 `null` 요소 때문에 컴파일 에러가 아니더라도 런타임에 실패할 운명이었습니다.
`map(Point::getX)` 연산은 `points` 리스트의 각 요소에 대해 `getX()`를 호출합니다. `List`의 두 번째 요소인 `null`에 대해 `null.getX()`가 호출되는 순간, `NullPointerException`이 발생하여 스트림 전체가 중단됩니다.

> Best Practice: null일 가능성이 있는 요소를 다루는 스트림에서는, 본격적인 연산 이전에 filter를 통해 null을 반드시 제거해야 합니다. Objects::nonNull 메서드 참조를 사용하면 매우 간결하게 처리할 수 있습니다.
>

## 5. 올바른 해결책: 의도에 맞는 코드 작성하기

문제의 원인을 파악했으니, 이제 원래 의도에 맞게 코드를 수정해 봅시다.

### 해결책 1: 각 `Point`의 x좌표만 출력하고 싶은 경우

`map`을 통해 `Integer`로 변환된 x좌표 값을 그대로 사용하면 됩니다.

```java
List<Point> points = Arrays.asList(new Point(12, 2), null, new Point(30, 40));

points.stream()
      .filter(Objects::nonNull) // 1. null을 안전하게 제거한다.
      .map(Point::getX)         // 2. Stream<Point>를 Stream<Integer>로 변환한다.
      .forEach(x -> System.out.println("X coordinate: " + x)); // 3. Integer 타입의 x를 그대로 사용한다.

// 출력:
// X coordinate: 12
// X coordinate: 30
```

### 해결책 2: `Point` 객체 자체를 사용하고 싶은 경우

`map`으로 변환할 필요 없이, `Point` 객체를 `forEach`까지 그대로 흘려보내면 됩니다.

```java
List<Point> points = Arrays.asList(new Point(12, 2), null, new Point(30, 40));

points.stream()
      .filter(Objects::nonNull) // 1. null을 안전하게 제거한다.
      .forEach(p -> { // 2. p는 여전히 Point 타입이다.
          System.out.println("Point's X coordinate: " + p.getX());
          // p.getY() 등 Point의 다른 메서드도 자유롭게 사용 가능
      });

// 출력:
// Point's X coordinate: 12
// Point's X coordinate: 30
```

## 6. 성능까지 생각한다면: 기본형 스트림 `mapToInt`

`map` 연산의 결과가 `int`, `long`, `double`과 같은 숫자 타입이라면, 불필요한 오토박싱으로 인한 성능 저하를 피하고 숫자 특화 연산을 사용하기 위해 **기본형 특화 스트림**을 사용하는 것이 좋습니다.

- **`mapToInt(ToIntFunction f)`:** `map`과 유사하지만, 결과를 `Integer` 객체가 아닌 **원시 타입 `int`*로 변환하여 **`IntStream`*을 반환합니다.

```java
List<Point> points = Arrays.asList(new Point(12, 2), null, new Point(30, 40));

int sumOfX = points.stream()
      .filter(Objects::nonNull)
      .mapToInt(Point::getX) // Stream<Point> -> IntStream (오토박싱 없음)
      .sum(); // IntStream이 제공하는 편리한 집계 메서드

System.out.println("Sum of all X coordinates: " + sumOfX); // 42
```

`IntStream`은 `sum()`, `average()`, `summaryStatistics()` 등 유용한 숫자 집계 메서드를 바로 사용할 수 있어 코드가 더 간결해집니다. 성능이 중요한 대용량 데이터 처리 시에는 기본형 스트림을 사용하는 것이 더 좋은 선택입니다.

## 7. 결론

Stream API를 사용할 때는 각 연산자를 지날 때마다 **"지금 이 파이프라인에는 어떤 타입의 데이터가 흐르고 있는가?"**를 항상 염두에 두어야 합니다.

- `map`은 스트림의 요소를 다른 타입으로 **변환**하는 강력한 도구이며, 그 결과로 스트림의 타입이 바뀔 수 있음을 기억해야 합니다.
- Java의 **오토박싱**은 편리하지만, `map`과 만났을 때 원시 타입이 래퍼 객체로 변환되어 예상치 못한 타입 에러를 유발할 수 있음을 인지해야 합니다.
- 스트림 파이프라인에서 **`null`은 잠재적인 시한폭탄**입니다. 본격적인 처리 전에 `filter(Objects::nonNull)`로 먼저 제거하는 습관을 들이는 것이 안전합니다.

---
