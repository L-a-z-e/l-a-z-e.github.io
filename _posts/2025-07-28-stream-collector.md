---
title: collect 완벽 정복 - Collector의 원리부터 실전 조합까지
description: 
author: laze
date: 2025-07-28 00:00:01 +0900
categories: [Dev, Java]
tags: [Java, Stream, Collector, groupingBy]
---
# [Java Stream 심층 분석] `collect` 완벽 정복: `Collector`의 원리부터 실전 조합까지

## 서론: `collect`의 미로에서 길을 잃다

Java Stream API는 `filter`, `map`과 같은 직관적인 중간 연산을 통해 우리를 함수형 프로그래밍의 세계로 안내합니다. 하지만 스트림의 여정이 끝나는 최종 연산, 특히 `collect()`를 마주하는 순간, 우리는 종종 복잡한 제네릭과 수많은 `Collectors` 팩토리 메서드의 미로 속에서 길을 잃곤 합니다.

```java
// "요리 타입별로 그룹핑하고, 각 그룹에서 요리 이름만 Set으로 모아줘, 결과는 타입 이름 순으로 정렬된 TreeMap에 담아줘"
TreeMap<Dish.Type, Set<String>> result = menu.stream()
    .collect(Collectors.groupingBy(
        Dish::getType,
        TreeMap::new,
        Collectors.mapping(Dish::getName, Collectors.toSet())
    ));
```

이 코드는 어떻게 동작할까요? `groupingBy`는 알겠는데 `mapping`은 무엇이며, `TreeMap::new`는 왜 들어갈까요?

이 글의 목표는 `collect`와 `Collector`의 구조를 원자 단위까지 완전히 분해하여, 어떤 상황에서 어떤 `Collector`를 조합해야 하는지 명확한 가이드를 제공하는 것입니다.

## 1. `Collector`의 본질: "어떻게 수집할 것인가"에 대한 레시피

`collect`는 '수집'이라는 행위를 합니다. `Collector`는 그 **'수집 방법'을 아주 상세하게 기술한 레시피**입니다. 이 레시피에는 크게 네 가지 필수 정보가 담겨 있습니다.

`Collector<T, A, R>`

- `T`: **재료** (스트림의 각 요소 타입)
- `A`: **믹싱볼** (수집 과정에서 사용할 중간 저장소, Accumulator)
- `R`: **최종 요리** (수집이 끝난 후의 최종 결과 타입)

그리고 이 레시피의 조리 과정은 다음과 같습니다.

1. **`supplier()` (믹싱볼 준비):** 수집을 시작할 때, 결과를 담을 새로운 '믹싱볼(`A`)'을 어떻게 만들 것인가?
  - `() -> new ArrayList<>()`
2. **`accumulator()` (재료 담기):** 스트림의 각 '재료(`T`)'를 '믹싱볼(`A`)'에 어떻게 담을 것인가?
  - `(list, element) -> list.add(element)`
3. **`combiner()` (여러 믹싱볼 합치기):** (스트림이 병렬 처리될 때) 여러 조리대에서 만들어진 '믹싱볼(`A`)'들을 어떻게 하나로 합칠 것인가?
  - `(list1, list2) -> { list1.addAll(list2); return list1; }`
4. **`finisher()` (요리 완성):** 모든 재료가 담긴 '믹싱볼(`A`)'을 최종 '요리(`R`)'로 어떻게 마감 처리할 것인가? (선택 사항)
  - `(list) -> Collections.unmodifiableList(list)`

우리가 사용하는 `Collectors.toList()`, `Collectors.toSet()` 등은 이 복잡한 레시피를 미리 만들어 놓은 편리한 '밀키트'입니다.

## 2. 제네릭의 비밀: `? super T`, `? extends K` (PECS 원칙)

`groupingBy`의 시그니처를 보면 `Function<? super T, ? extends K>` 와 같은 와일드카드가 등장합니다. 이는 **PECS(Producer Extends, Consumer Super)** 원칙에 따른 것으로, 제네릭을 더 유연하게 만들기 위한 장치입니다.

- **`? extends K` (Producer Extends):** "생산자는 `extends`를 사용한다."
  - **의미:** `classifier` 함수는 `K` 타입의 결과(Key)를 **생산(Produce)**합니다. `K`의 자식 타입(e.g., `SubK`)을 생산해도, 우리는 그것을 더 넓은 범위인 `K` 타입으로 안전하게 받을 수 있습니다.
  - **예시:** `String`을 반환해야 하는 자리에 `CharSequence`의 자식인 `String`을 반환해도 안전합니다.
- **`? super T` (Consumer Super):** "소비자는 `super`를 사용한다."
  - **의미:** `classifier` 함수는 `T` 타입의 요소(element)를 **소비(Consume)**합니다. `T`를 처리할 수 있는 함수라면, 당연히 `T`의 부모 타입(e.g., `SuperT`)도 안전하게 처리(소비)할 수 있습니다.
  - **예시:** `Apple`을 처리하는 함수 자리에, 모든 `Fruit`(`Apple`의 부모)을 처리하는 함수를 넣어도 `Apple`을 처리하는 데는 문제가 없습니다.

**결론:** 이 와일드카드 덕분에 `groupingBy`는 더 다양한 종류의 함수를 분류자로 받아들일 수 있는 유연성을 갖게 됩니다.

## 3. 레벨별 `groupingBy` 정복하기

`Collectors.groupingBy`는 세 가지 레벨의 오버로딩된 메서드를 제공합니다. 레벨이 올라갈수록 더 정교한 제어가 가능해집니다.

### 예제 데이터

```java
List<Dish> menu = ...; // (name, calories, type)
```

### 레벨 1: `groupingBy(classifier)` - 가장 기본적인 분류

- **역할:** 스트림의 요소를 특정 기준으로 그룹핑하여, 그 결과를 `List`에 담아 `Map`으로 반환합니다.
- **반환 타입:** `Map<K, List<T>>`
- **요구 부품:** `classifier` (분류자) `Function` 하나.

```java
// 요리를 타입(MEAT, FISH, OTHER)별로 그룹핑하기
Map<Dish.Type, List<Dish>> dishesByType = menu.stream()
    .collect(Collectors.groupingBy(Dish::getType));

/* 결과:
{
  FISH=[prawns, salmon],
  MEAT=[pork, beef, chicken],
  OTHER=[french fries, rice, seasonal fruit]
}
*/
```

`Dish::getType` 함수가 각 `Dish`를 `Type`이라는 `K`(Key)로 분류하고, 동일한 `K`를 가진 `Dish` 객체(`T`)들을 `List<T>`로 묶어줍니다.

### 레벨 2: `groupingBy(classifier, downstream)` - 그룹 내부에서 추가 작업하기

- **역할:** 그룹핑을 한 후, 각 그룹에 대해 추가적인 `Collector`를 적용하여 결과를 변환합니다.
- **반환 타입:** `Map<K, D>` (여기서 `D`는 `downstream` Collector의 결과 타입)
- **요구 부품:** `classifier`(분류자)와 **`downstream`(하위 수집기)** `Collector` 두 개.

`downstream`은 **"그룹으로 나뉜 각 묶음 안에서 무엇을 할 것인가?"**를 결정하는 또 다른 '레시피'입니다.

### 예제 1: 그룹별 개수 세기 (`counting`)

```java
// 요리 타입별로 몇 개의 요리가 있는지 세기
Map<Dish.Type, Long> dishesCountByType = menu.stream()
    .collect(Collectors.groupingBy(Dish::getType, Collectors.counting()));

// 결과: {FISH=2, MEAT=3, OTHER=3}
```

`downstream`으로 `Collectors.counting()`을 전달하여, 각 그룹의 `List<Dish>`를 `Long`(개수)으로 변환했습니다.

### 예제 2: 그룹별 데이터 변환 후 수집 (`mapping`)

`mapping`은 그룹화된 요소를 최종 수집하기 전에, **한 번 더 변환(map)**하는 기능을 제공하는 `downstream Collector`입니다.

```java
// 요리 타입별로 그룹핑하되, 각 그룹에는 요리의 "이름"만 Set으로 저장하기
Map<Dish.Type, Set<String>> dishNamesByType = menu.stream()
    .collect(Collectors.groupingBy(
        Dish::getType, // 1. 타입으로 그룹핑
        Collectors.mapping(Dish::getName, Collectors.toSet()) // 2. 각 Dish를 이름(String)으로 변환 후, Set으로 수집
    ));

/* 결과:
{
  FISH=[prawns, salmon],
  MEAT=[beef, pork, chicken],
  OTHER=[rice, seasonal fruit, french fries]
}
*/
```

### 레벨 3: `groupingBy(classifier, mapFactory, downstream)` - Map 구현체 직접 선택하기

- **역할:** 레벨 2의 기능에 더해, 최종 결과를 담을 `Map`의 구체적인 구현체(`HashMap`, `TreeMap` 등)를 직접 지정할 수 있습니다.
- **반환 타입:** `M` (개발자가 `mapFactory`로 제공한 `Map` 타입)
- **요구 부품:** `classifier`, `mapFactory`(Map 생성기), `downstream` Collector 세 개.

```java
// 요리 타입별 개수를 세되, 결과를 타입 이름 순서대로 정렬된 TreeMap에 담기
TreeMap<Dish.Type, Long> sortedDishesCount = menu.stream()
    .collect(Collectors.groupingBy(
        Dish::getType,
        TreeMap::new, // 결과를 담을 Map으로 TreeMap을 사용하도록 지정
        Collectors.counting()
    ));

// 결과: (키가 알파벳 순서로 정렬됨) {FISH=2, MEAT=3, OTHER=3}
```

## 4. `toMap` vs `groupingBy`: 언제 무엇을 써야 할까?

두 `Collector`는 모두 `Map`을 생성하지만, **키(Key) 중복**을 다루는 방식에 근본적인 차이가 있습니다.

- **`toMap(keyMapper, valueMapper)`**
  - **가정:** 스트림의 각 요소가 Map의 **고유한(unique) 엔트리 하나**가 될 것이라고 가정합니다.
  - **키 중복 시:** 기본적으로 `IllegalStateException`을 발생시킵니다.
  - **예시:** `name`을 키로 사용하여 `Dish` 객체를 `Map`으로 변환할 때. (이름은 고유하다고 가정)

      ```java
      Map<String, Dish> dishMapByName = menu.stream()
          .collect(Collectors.toMap(Dish::getName, Function.identity()));
      ```

- **`groupingBy(classifier)`**
  - **가정:** **키가 중복되는 것이 당연하다**고 가정합니다.
  - **키 중복 시:** 중복된 키를 가진 요소들을 `List` (또는 `downstream`이 정의한 다른 형태)로 묶어줍니다.
  - **예시:** `type`을 키로 사용하여 `Dish` 객체를 그룹핑할 때. (타입은 중복되므로)

### 키 충돌 해결하기

`toMap`에서 키 충돌이 예상될 경우, 세 번째 인자인 `mergeFunction`(`BinaryOperator`)을 제공하여 충돌을 해결할 수 있습니다.

```java
// 요리 타입(키)이 중복될 경우, 칼로리가 더 높은 Dish를 선택하여 Map을 생성
Map<Dish.Type, Dish> highestCalorieDishByType = menu.stream()
    .collect(Collectors.toMap(
        Dish::getType,                                                  // Key Mapper
        Function.identity(),                                            // Value Mapper
        (d1, d2) -> d1.getCalories() > d2.getCalories() ? d1 : d2     // Merge Function
    ));
```

## 5. 보너스: `Collector.Characteristics`의 비밀

`Collector` 인터페이스에는 `characteristics()`라는 메서드가 있습니다. 이는 해당 `Collector`의 동작 특성을 Stream 프레임워크에 알려주는 '힌트' 역할을 합니다.

- **`IDENTITY_FINISH`:** '믹싱볼(`A`)'과 '최종 요리(`R`)'가 동일하여 `finisher` 단계가 필요 없음을 의미. (`toList` 등)
- **`CONCURRENT`:** 여러 스레드가 동시에 `accumulator`에 접근해도 안전함을 의미. `parallelStream()`에서 성능 최적화에 사용됨.
- **`UNORDERED`:** 결과의 순서가 중요하지 않음을 의미. 역시 병렬 처리 시 성능 최적화에 사용됨.

이 힌트 덕분에 Stream API는 내부적으로 더 효율적인 수집 전략을 선택할 수 있습니다.

## 6. 결론: `Collector` 조립 가이드

복잡한 데이터 수집 요구사항을 만났을 때, 다음 의사결정 트리를 따라가면 어떤 `Collector`를 조합해야 할지 명확해집니다.

1. **최종 결과 형태가 `Map`인가?**
  - Yes → `toMap` 또는 `groupingBy`를 고려한다.
2. **분류하려는 기준(Key)이 스트림 내에서 중복될 수 있는가?**
  - **No (Key is unique)** → `toMap`을 사용한다. (중복 가능성 있으면 `mergeFunction` 추가)
  - **Yes (Key can be duplicated)** → `groupingBy`를 사용한다.
3. **그룹핑된 결과가 단순한 `List`가 아닌, 다른 형태(개수, 합계, `Set` 등)여야 하는가?**
  - Yes → `groupingBy`의 두 번째 인자인 **`downstream Collector`*를 적극적으로 활용한다.
4. **그룹핑하기 전에, 각 요소의 데이터를 변환(map)해야 하는가?**
  - Yes → `downstream Collector`로 **`mapping`*을 사용한다.
5. **결과 `Map`의 순서가 중요하거나 특정 `Map` 구현체가 필요한가?**
  - Yes → `groupingBy`의 세 번째 인자인 **`mapFactory`**(`TreeMap::new` 등)를 사용한다.

복잡한 제네릭 시그니처에 압도되지 말고, 각 `Collector`의 '역할'과 '입/출력'이라는 레고 블록에 집중하면, 어떤 복잡한 데이터 구조라도 자신감 있게 조립해낼 수 있을 것입니다.
