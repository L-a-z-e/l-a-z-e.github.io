---
title: Comparator, 함수형 인터페이스
description: 
author: laze
date: 2025-07-25 00:00:13 +0900
categories: [Dev, Java]
tags: [Java, Comparator, Functional Interface]
---
# [Java] `Comparator` , 함수형 인터페이스

## 1. 서론: 정렬, 어떻게 하고 계신가요?

Java 8의 Stream API를 사용하여 객체 리스트를 정렬할 때, 우리는 `sorted()` 메서드와 함께 `Comparator` 인터페이스를 사용합니다. 이때, 많은 예제 코드에서 다음과 같은 람다 표현식을 흔히 볼 수 있습니다.

```java
import java.util.Comparator;

List<Transaction> transactions = ...;

// 거래 금액(value)을 기준으로 오름차순 정렬
transactions.stream()
        .sorted((t1, t2) -> t1.getValue() - t2.getValue());
```

이 코드는 잘 동작하지만, 몇 가지 근본적인 질문을 던지게 합니다.

1. `t1.getValue() - t2.getValue()`는 정확히 어떻게 정렬을 수행하는 걸까? `t1`과 `t2`를 바꾸면 어떻게 될까?
2. `Comparator` 인터페이스를 열어보면 `compare()` 외에 `equals()` 등 다른 메서드도 있는데, 어떻게 람다를 사용할 수 있는 함수형 인터페이스(Functional Interface)가 되는 걸까?

이 글에서는 이 두 가지 핵심 질문에 대한 답을 통해 `Comparator`의 동작 원리를 완벽하게 이해하고, 실무에서 더 안전하고 우아하게 사용하는 방법을 알아봅니다.

## 2. `compare` 메서드의 계약: 음수, 0, 양수의 의미

`Comparator`의 핵심은 `int compare(T o1, T o2)`라는 단 하나의 추상 메서드에 있습니다. 이 메서드는 정렬 기준을 정의하기 위해 호출자와 다음과 같은 **'계약(Contract)'**을 맺습니다.

- **반환값이 음수 (e.g., -1):** 첫 번째 인자(`o1`)가 두 번째 인자(`o2`)보다 **"작다"** (앞에 와야 한다).
- **반환값이 0:** 두 인자가 **"같다"** (순서를 바꿀 필요 없다).
- **반환값이 양수 (e.g., 1):** 첫 번째 인자(`o1`)가 두 번째 인자(`o2`)보다 **"크다"** (뒤에 와야 한다).

`sorted()` 메서드는 이 계약에 따라 요소들의 위치를 계속 바꾸어 나가며 최종적으로 리스트를 정렬합니다.

### 오름차순 (Ascending Order): `t1 - t2`

```java
.sorted((t1, t2) -> t1.getValue() - t2.getValue())
```

- **가정:** `t1.getValue()`가 `50`, `t2.getValue()`가 `100`이라고 해봅시다.
- **계산:** `50 - 100 = -50` (음수)
- **해석:** `sorted()`는 "t1이 t2보다 작다"고 판단합니다. 따라서 정렬된 결과에서 `t1`은 `t2`의 **앞에** 위치하게 됩니다. `[... 50, ..., 100, ...]`
- **결과:** 작은 값이 앞으로 오므로, **오름차순**으로 정렬됩니다.

### 내림차순 (Descending Order): `t2 - t1`

```java
.sorted((t1, t2) -> t2.getValue() - t1.getValue())
```

- **가정:** `t1.getValue()`가 `50`, `t2.getValue()`가 `100`이라고 해봅시다.
- **계산:** `100 - 50 = 50` (양수)
- **해석:** `sorted()`는 "t1이 t2보다 크다"고 판단합니다. 따라서 정렬된 결과에서 `t1`은 `t2`의 **뒤에** 위치하게 됩니다. `[... 100, ..., 50, ...]`
- **결과:** 큰 값이 앞으로 오므로, **내림차순**으로 정렬됩니다.

## 3. 더 안전하고 우아하게: `Comparator.comparing()`

단순 뺄셈(`-`) 방식은 코드가 간결하지만, 두 값의 차이가 `Integer.MAX_VALUE`를 초과할 경우 **정수 오버플로우(Integer Overflow)**가 발생하여 정렬 순서가 깨질 수 있는 잠재적 위험을 안고 있습니다.

Java 8부터는 이 문제를 해결하고, 훨씬 더 가독성 높은 코드를 작성할 수 있도록 `Comparator` 인터페이스에 유용한 정적/디폴트 메서드들이 추가되었습니다.

### `Comparator.comparing()`과 메서드 참조

`Comparator.comparing()`은 정렬의 '키'가 되는 값을 추출하는 함수를 인자로 받습니다.

```java
import static java.util.Comparator.comparing;

// 오름차순 (가장 안전하고 권장되는 방식)
.sorted(comparing(Transaction::getValue))

// 내림차순
.sorted(comparing(Transaction::getValue).reversed())
```

`comparing(Transaction::getValue)`는 "Transaction 객체에서 `getValue()` 메서드를 호출한 결과를 기준으로 오름차순 정렬하라"는 의미입니다. 이 방식은 내부적으로 오버플로우 없이 안전하게 비교를 수행하며, 코드의 '의도'가 훨씬 명확하게 드러납니다.

### 다중 조건 정렬: `thenComparing()`

여러 개의 기준으로 정렬해야 할 때는 `thenComparing()`을 체인처럼 이어붙이면 됩니다.

```java
// 1차: 연도(year) 오름차순, 2차: 금액(value) 내림차순
.sorted(comparing(Transaction::getYear)
        .thenComparing(Transaction::getValue, Comparator.reverseOrder()))
```

## 4. `Comparator`는 왜 함수형 인터페이스일까?

함수형 인터페이스의 정확한 정의는 **"단 하나의 추상 메서드(abstract method)만을 가진 인터페이스"** 입니다. 그런데 `Comparator` 인터페이스의 소스 코드를 열어보면 `compare()` 외에 `equals()`, `reversed()`, `comparing()` 등 여러 메서드가 보입니다. 어떻게 된 일일까요?

```java
@FunctionalInterface
public interface Comparator<T> {
    // 1. 유일한 추상 메서드
    int compare(T o1, T o2);

    // 2. Object 클래스의 public 메서드 (추상 메서드 카운트에서 제외됨)
    boolean equals(Object obj);

    // 3. 디폴트 메서드 (추상 메서드가 아님)
    default Comparator<T> reversed() { ... }

    // 4. 정적 메서드 (추상 메서드가 아님)
    public static <T, U> Comparator<T> comparing(...) { ... }
}
```

- **`compare(T o1, T o2)`:** 이것이 바로 `Comparator`가 가진 **유일한 추상 메서드**입니다. 우리가 람다로 구현해야 할 대상이 바로 이 메서드입니다.
- **`equals(Object obj)`:** Java 언어 명세에 따라, 인터페이스가 `Object` 클래스의 `public` 메서드와 동일한 시그니처의 추상 메서드를 선언하더라도, **함수형 인터페이스 여부를 판단할 때 이 메서드는 추상 메서드 개수에 포함되지 않습니다.**
- **`default` 및 `static` 메서드:** Java 8부터 인터페이스는 구현부를 가진 `default` 및 `static` 메서드를 포함할 수 있습니다. 이들은 추상 메서드가 아니므로, 함수형 인터페이스의 조건에 아무런 영향을 미치지 않습니다.

결론적으로 `Comparator`는 구현해야 할 추상 메서드가 `compare()` 단 하나뿐이므로, **함수형 인터페이스의 정의를 완벽하게 만족합니다.**

## 5. 결론: `Comparator` 제대로 이해하고 사용하기

`Comparator`를 사용한 정렬의 핵심은 `compare` 메서드가 반환하는 '음수, 0, 양수'라는 계약을 이해하는 것입니다. 이 원리를 알면 어떤 복잡한 정렬 조건도 자신감 있게 구현할 수 있습니다.

하지만 실무에서는,

- 오버플로우 위험이 있는 `t1 - t2` 방식보다는,
- **`Comparator.comparing()`과 메서드 참조(`::`)를 함께 사용하는 것**이 더 안전하고, 가독성이 높으며, 현대적인 Java 스타일입니다.

`Comparator`가 함수형 인터페이스이기 때문에, 우리는 이처럼 강력하고 유연한 기능을 Stream API의 파이프라인 안에서 자유자재로 활용할 수 있는 것입니다.
