---
title: parallelStream()은 어떻게 데이터를 분할할까? Spliterator의 모든 것
description: 
author: laze
date: 2025-07-29 00:00:01 +0900
categories: [Dev, Java]
tags: [Java, Stream, parallelStream, Spliterator]
---
# [Java Stream 심층 분석] `parallelStream()`은 어떻게 데이터를 분할할까? Spliterator의 모든 것

## 1. 서론: `parallelStream()` 한 줄에 숨겨진 비밀

Java 8에서 `parallelStream()`이 등장했을 때, 많은 개발자들은 열광했습니다. 단 한 줄의 코드 변경으로 멀티코어 CPU를 활용하여 데이터 처리를 병렬화할 수 있다는 것은 매우 매력적이었습니다.

```java
// 이 한 줄의 코드는 어떻게 마법처럼 병렬로 동작할까?
long count = myList.parallelStream()
                   .filter(i -> i > 100)
                   .count();
```

하지만 이 마법의 이면에는 **`Spliterator`**라는, 스트림의 데이터 소스를 '탐색'하고 **'분할(Split)'**하는 정교한 엔진이 숨어있습니다.

`Spliterator`를 이해하는 것은 `parallelStream()`이 언제 빠르고, 언제 오히려 느려지는지, 그리고 어떻게 최적화할 수 있는지에 대한 해답을 얻는 것과 같습니다.

이 글에서는 Java 컬렉션이 기본적으로 제공하는 `Spliterator`의 동작 원리부터, 필요할 때 직접 `Spliterator`를 구현하여 커스텀 데이터 소스를 병렬 처리하는 방법, 그리고 병렬 처리 성능을 극대화하는 `characteristics`의 비밀까지, Spliterator의 모든 것을 심층적으로 분석합니다.

## 2. Spliterator란 무엇인가? Iterator의 진화된 '분할 가능한' 형태

`Spliterator`는 **Splittable Iterator**의 줄임말입니다.

전통적인 `Iterator`가 `hasNext()`와 `next()`를 통해 데이터를 순차적으로만 탐색할 수 있었던 반면, `Spliterator`는 여기에 **'분할' 기능(`trySplit`)**을 추가하여 병렬 처리를 가능하게 만든, `Iterator`의 진화된 버전입니다.

## 3. "기성품" Spliterator 분석: `ArrayList`는 어떻게 분할되는가?

우리가 직접 `Spliterator`를 구현하지 않아도 `myList.parallelStream()`이 동작하는 이유는, `ArrayList`와 같은 Java의 핵심 컬렉션들이 이미 고도로 최적화된 `Spliterator`를 내장하고 있기 때문입니다.

`ArrayList`의 `spliterator()` 메서드가 반환하는 `ArrayListSpliterator`의 분할 원리는 매우 간단하고 효율적입니다.

- **`ArrayListSpliterator`의 `trySplit` 원리:**
  1. `ArrayList`는 내부적으로 **배열(array)**로 데이터를 관리하며, 현재 크기(size)를 정확히 알고 있습니다.
  2. `trySplit()`이 호출되면, `ArrayListSpliterator`는 자신이 담당하고 있는 배열의 남은 요소 개수를 정확히 **절반으로 나눕니다.** (`mid = low + (high - low) / 2`)
  3. 중간 인덱스(`mid`)를 기준으로, **앞쪽 절반**을 담당하는 새로운 `ArrayListSpliterator`를 생성하여 반환합니다.
  4. 기존 `Spliterator`는 **뒤쪽 절반**을 담당하도록 자신의 내부 상태(시작 인덱스)를 조정합니다.

이 과정이 재귀적으로 반복되면서, 하나의 거대한 `ArrayList`는 여러 개의 작은 조각으로 순식간에 분할되고, 이 조각들이 포크/조인 프레임워크의 각 스레드에 할당되어 병렬로 처리됩니다.

인덱스 기반으로 메모리 접근이 O(1)인 배열의 특성 덕분에, `ArrayList`의 병렬 처리 성능은 매우 뛰어납니다.

반면, `LinkedList`는 특정 인덱스에 접근하려면 처음부터 순회해야 하므로(`O(N)`), `trySplit`의 효율이 떨어져 병렬 처리 성능이 `ArrayList`보다 좋지 않습니다.

## 4. "수제품" Spliterator 제작: 왜 직접 만들어야 할까?

만약 우리의 데이터 소스가 `ArrayList`처럼 처음부터 분할에 최적화된 구조가 아니라면 어떻게 될까요?

예를 들어, 거대한 텍스트 파일의 각 라인, 네트워크 스트림, 또는 직접 만든 트리 구조 같은 경우입니다.

이러한 데이터 소스를 효율적으로 병렬 처리하기 위해서는, **"데이터의 특성에 맞는 최적의 분할 전략"**을 우리가 직접 코드로 작성하여 Stream 프레임워크에 알려주어야 합니다.

이것이 바로 커스텀 `Spliterator`를 구현하는 이유입니다.

## 5. 실전! 커스텀 `Spliterator` 구현: 병렬 단어 개수 세기

문자열에서 단어 개수를 세는 예제를 통해, 커스텀 `Spliterator`를 단계별로 구현해 보겠습니다.

### 5.1. `tryAdvance` (순차 탐색)

가장 먼저 구현할 것은 `Iterator`처럼 데이터를 순차적으로 하나씩 소비하는 `tryAdvance`입니다.

```java
public class WordCountSpliterator implements Spliterator<Character> {
    private final String string;
    private int currentChar = 0;

    // ... 생성자 ...

    @Override
    public boolean tryAdvance(Consumer<? super Character> action) {
        if (currentChar < string.length()) {
            // 처리할 문자가 남아있으면,
            // action(람다)에 현재 문자를 전달하고, 인덱스를 1 증가시킨다.
            action.accept(string.charAt(currentChar++));
            return true; // 아직 처리할 요소가 남았음을 알림
        }
        return false; // 더 이상 처리할 요소가 없음을 알림
    }
    // ...
}
```

### 5.2. `trySplit` (병렬 분할)

이것이 병렬 처리의 핵심입니다. 문자열을 아무 데서나 반으로 자르면 단어("apple")가 "ap"와 "ple"로 쪼개져 계산이 잘못될 수 있습니다. 따라서, **단어의 경계가 되는 공백(' ')**을 찾아 안전하게 분할해야 합니다.

```java
@Override
public Spliterator<Character> trySplit() {
    int currentSize = string.length() - currentChar;
    // 남은 문자열이 너무 작으면 분할하지 않는다. (오버헤드가 더 큼)
    if (currentSize < 10) {
        return null;
    }

    // 남은 문자열의 중간 지점 근처에서부터 공백을 찾는다.
    for (int splitPos = currentSize / 2 + currentChar; splitPos < string.length(); splitPos++) {
        if (Character.isWhitespace(string.charAt(splitPos))) {
            // 공백을 찾음
            // 1. 앞부분 조각을 담당할 새로운 Spliterator를 만든다.
            Spliterator<Character> newSpliterator =
                    new WordCountSpliterator(string.substring(currentChar, splitPos));

            // 2. 현재 Spliterator는 뒷부분 조각을 담당하도록 시작 위치를 갱신한다.
            this.currentChar = splitPos;

            // 3. 새로 만든 Spliterator를 반환한다.
            return newSpliterator;
        }
    }
    // 공백을 못 찾았으면 분할하지 않는다.
    return null;
}
```

### 5.3. `estimateSize` (크기 정보)

남은 데이터의 크기를 알려줍니다. 문자열은 크기가 명확하므로 정확한 값을 반환할 수 있습니다.

```java
@Override
public long estimateSize() {
    return string.length() - currentChar;
}
```

### 5.4. `characteristics` (특성 정보) - 성능 최적화의 핵심

`characteristics`는 이 `Spliterator`가 다루는 데이터 소스의 '특성'을 Stream 프레임워크에 알려주는 힌트입니다. 프레임워크는 이 힌트를 보고 최적화를 수행합니다.

```java
@Override
public int characteristics() {
    return ORDERED + SIZED + SUBSIZED + NONNULL + IMMUTABLE;
}
```

| 특성 | 의미 | 프레임워크에 주는 힌트 및 효과 |
| --- | --- | --- |
| **`ORDERED`** | 요소들이 정해진 순서를 가지고 있다. | 순서가 중요한 `findFirst()` 같은 연산의 결과를 보장한다. 순서를 유지하기 위한 추가 비용이 발생할 수 있다. |
| **`SIZED`** | `estimateSize()`가 정확한 크기를 반환한다. | 프레임워크는 `trySplit`을 할 때, 배열처럼 정확한 크기 기반의 분할 전략을 적용할 수 있다. `ArrayList`의 `Spliterator`가 이 특성을 가진다. |
| **`SUBSIZED`** | `trySplit()`으로 분할된 하위 `Spliterator`들도 `SIZED` 특성을 유지한다. | 분할된 모든 작업의 크기를 예측할 수 있으므로, 포크/조인 프레임워크가 작업을 더 효율적으로 분배할 수 있다. |
| **`NONNULL`** | `null`인 요소가 없다. | 스트림 처리 중 `null` 체크를 생략하여 약간의 성능 향상을 기대할 수 있다. |
| **`IMMUTABLE`** | 데이터 소스가 불변이다. (요소 추가/삭제/변경이 없음) | 데이터 경쟁(Data Race)에 대한 걱정 없이 안전하게 병렬 처리를 수행할 수 있다. 프레임워크는 더 공격적인 최적화를 시도할 수 있다. |
| **`CONCURRENT`** | 데이터 소스 자체가 동시성을 지원한다. (`ConcurrentHashMap` 등) | `Spliterator`를 분할하지 않고도, 여러 스레드가 동시에 데이터 소스에 접근하여 처리할 수 있음을 의미한다. |
| **`UNORDERED`** | 요소들의 순서가 중요하지 않다. | 프레임워크는 순서 유지 비용을 완전히 무시하고, 가장 효율적인 방식으로 데이터를 처리하고 결합할 수 있다. `HashSet`의 `Spliterator`가 이 특성을 가진다. 병렬 처리에서 상당한 성능 향상을 가져온다. |

`WordCountSpliterator`는 불변인 `String`을 순서대로, 정확한 크기를 기반으로 다루므로 위 5가지 특성을 모두 가집니다.

## 6. 조립 및 실행: `StreamSupport.stream()`으로 생명 불어넣기

직접 구현한 `Spliterator`는 `StreamSupport.stream()`을 통해 실제 스트림으로 변환됩니다. `parallel` 플래그를 `true`로 설정하면, Stream 프레임워크가 `trySplit()`을 재귀적으로 호출하여 작업을 분할하고, 이를 내부 스레드 풀에 분배하여 병렬로 처리합니다.

```java
public static int countWords(String sentence) {
    Spliterator<Character> spliterator = new WordCountSpliterator(sentence);
    // spliterator를 병렬 스트림으로 변환
    Stream<Character> stream = StreamSupport.stream(spliterator, true);

    // 이제 이 병렬 스트림을 사용하여 원하는 작업을 수행할 수 있다.
    // ...
}
```

## 7. 결론: `Spliterator`를 이해한다는 것

`Spliterator`는 `parallelStream()`의 마법 뒤에 숨겨진 정교한 엔진입니다. 그 동작 원리를 이해하는 것은 Java의 병렬 처리 메커니즘을 깊이 있게 이해하는 것과 같습니다.

- **대부분의 경우:** `ArrayList`, `HashSet` 등 Java 컬렉션 프레임워크가 제공하는 고도로 최적화된 내장 `Spliterator`만으로 충분합니다. 우리는 그저 `parallelStream()`을 호출하기만 하면 됩니다.
- **특별한 경우:** `String`, 파일 I/O, 직접 만든 커스텀 데이터 구조처럼, **데이터의 특성에 맞는 정교한 분할 전략이 필요한 경우**에 `Spliterator`를 직접 구현하는 것이 강력한 무기가 될 수 있습니다.

`Spliterator`를 직접 작성하는 것은 흔한 일은 아니지만, 그 내부 원리를 이해하고 나면 `parallelStream()`을 더 자신감 있고 효과적으로 사용하며, 때로는 그 성능의 한계를 스스로 돌파할 수 있는 개발자로 성장하게 될 것입니다.
