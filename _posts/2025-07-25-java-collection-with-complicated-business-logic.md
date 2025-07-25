---
title: Transactional(readOnly = true) 동일 테이블 조회 시 성능이 저하되는 이유
description: 
author: laze
date: 2025-07-25 00:00:01 +0900
categories: [Dev, Java]
tags: [Java, List, Stream, Map, Performance Optimization]
---
# [Java] 이중 for문을 `Stream`과 `Map`으로 최적화하는 실전 패턴 (날짜 구간 계산)

## 1. 서론: 복잡한 비즈니스 로직과 이중 `for` 루프의 한계

실무에서 개발을 하다 보면, 두 개의 다른 데이터 목록을 비교하며 복잡한 계산을 수행해야 하는 경우가 많습니다.

특히 "특정 기간 동안의 유효한 할인율 목록"과 "사용자의 구매 이력 목록"을 비교하여 최종 할인 금액을 계산하는 것과 같은 '구간 겹침' 로직은 매우 흔한 요구사항입니다.

이런 문제에 직면했을 때, 가장 먼저 떠올리는 방법은 이중 `for` 루프입니다.

```java
// 날짜 구간별 할인율 정보
List<DiscountPolicy> policies = ...; // (startDate, endDate, rate)
// 사용자의 구매 이력
List<Purchase> purchases = ...;      // (purchaseDate, amount)

for (Purchase purchase : purchases) {
    for (DiscountPolicy policy : policies) {
        // 구매일이 할인 정책 기간에 포함되는지 확인
        if (isBetween(purchase.getPurchaseDate(), policy.getStartDate(), policy.getEndDate())) {
            // ... 할인 계산 로직 ...
        }
    }
}
```

이 코드는 직관적이지만, 치명적인 성능 문제를 안고 있습니다. `purchases` 목록의 크기가 N, `policies` 목록의 크기가 M일 때, 이 로직의 시간 복잡도는 **O(N * M)**이 됩니다.

만약 구매 이력이 10,000건, 할인 정책이 1,000개라면, 비교 연산은 무려 10,000,000번이나 수행됩니다.

이 글에서는 이처럼 비효율적인 이중 루프를 `Stream`과 `Map`을 활용한 **전처리(Preprocessing)** 기법을 통해 어떻게 비약적으로 최적화할 수 있는지 단계별로 알아보겠습니다.

## 2. 1단계: `Stream`으로 리팩토링하기 (가독성 개선)

먼저, 이중 `for` 루프를 Java 8의 `Stream` API를 사용하여 리팩토링해 보겠습니다.

### 예제 DTO 정의

```java
// 날짜 구간별 일일 고정 금액 정보
record Policy(LocalDate startDate, LocalDate endDate, BigDecimal dailyAmount) {}

// 계산 대상 날짜 구간 정보
@Data
class TargetPeriod {
    private LocalDate startDate;
    private LocalDate endDate;
    private BigDecimal calculatedAmount = BigDecimal.ZERO;

    public TargetPeriod(LocalDate startDate, LocalDate endDate) {
        this.startDate = startDate;
        this.endDate = endDate;
    }
}
```

### 중첩 `Stream`을 사용한 로직

```java
public void calculateWithNestedStream(List<Policy> policies, List<TargetPeriod> targets) {
    targets.forEach(target -> {
        BigDecimal totalAmount = policies.stream() // <-- 내부 스트림 (이중 루프와 동일)
            .filter(policy -> isOverlap(policy, target)) // 1. 겹치는 구간 필터링
            .map(policy -> { // 2. 겹치는 일수만큼 금액 계산
                long overlapDays = getOverlapDays(policy, target);
                return policy.dailyAmount().multiply(BigDecimal.valueOf(overlapDays));
            })
            .reduce(BigDecimal.ZERO, BigDecimal::add); // 3. 합산

        target.setCalculatedAmount(totalAmount);
    });
}

// 두 날짜 구간이 겹치는지 확인하는 헬퍼 메서드
private boolean isOverlap(Policy policy, TargetPeriod target) {
    return !policy.endDate().isBefore(target.getStartDate()) &&
           !policy.startDate().isAfter(target.getEndDate());
}

// 겹치는 일수를 계산하는 헬퍼 메서드
private long getOverlapDays(Policy policy, TargetPeriod target) {
    LocalDate overlapStart = policy.startDate().isAfter(target.getStartDate()) ? policy.startDate() : target.getStartDate();
    LocalDate overlapEnd = policy.endDate().isBefore(target.getEndDate()) ? policy.endDate() : target.getEndDate();
    return ChronoUnit.DAYS.between(overlapStart, overlapEnd) + 1;
}
```

- **장점:** `for-if` 구조가 `filter-map-reduce`라는 선언적인 코드로 바뀌어 가독성이 향상되고, 코드의 '의도'가 더 명확해졌습니다.
- **한계:** 이 코드는 여전히 내부적으로 `targets` 목록의 각 요소에 대해 `policies` 목록 전체를 순회합니다. 시간 복잡도는 여전히 **O(N * M)**으로, 근본적인 성능 문제는 해결되지 않았습니다.

## 3. 2단계 (핵심): 전처리(Preprocessing)와 `Map`을 이용한 성능 최적화

성능 문제의 근본 원인은 **반복적으로 수행되는 비효율적인 '탐색' 과정**에 있습니다. `TargetPeriod` 하나를 계산할 때마다, 매번 `Policy` 목록 전체를 뒤져 겹치는 구간을 찾아야 합니다.

이 문제를 해결하기 위한 가장 강력한 전략은, 탐색 대상인 `Policy` 목록을 **조회에 최적화된 데이터 구조**로 미리 가공해두는 **"전처리(Preprocessing)"** 입니다. 여기서는 `Map<LocalDate, BigDecimal>`을 사용합니다.

**핵심 전략:** `Policy` 목록의 모든 날짜 구간을 하루 단위로 펼쳐서, `Map<날짜, 해당 날짜의 금액>` 형태로 미리 계산해 둡니다.

### 3.1. 전처리 코드: `dateToAmountMap` 생성

`flatMap`을 사용하여 각 `Policy`의 날짜 구간을 `LocalDate` 스트림으로 펼치고, `Collectors.toMap`으로 `Map`을 생성합니다.

```java
private Map<LocalDate, BigDecimal> preprocessPolicies(List<Policy> policies) {
    return policies.stream()
        .flatMap(policy -> {
            // 각 Policy의 시작일부터 종료일까지의 모든 날짜 Stream을 생성
            return Stream.iterate(policy.startDate(), date -> date.plusDays(1))
                         .limit(ChronoUnit.DAYS.between(policy.startDate(), policy.endDate()) + 1)
                         .map(date -> Map.entry(date, policy.dailyAmount()));
        })
        .collect(Collectors.toMap(
            Map.Entry::getKey,
            Map.Entry::getValue,
            BigDecimal::add // 만약 동일 날짜에 여러 정책이 겹치면 금액을 합산한다.
        ));
}
```

### 3.2. 계산 코드: `Map`을 이용한 빠른 조회

이제 `TargetPeriod`를 계산할 때, 더 이상 `Policy` 목록 전체를 순회할 필요가 없습니다. `TargetPeriod`의 시작일부터 종료일까지 하루씩 `Map`에서 금액을 O(1) 속도로 조회하여 더하기만 하면 됩니다.

```java
public void calculateWithPreprocessedMap(Map<LocalDate, BigDecimal> dateToAmountMap, List<TargetPeriod> targets) {
    targets.forEach(target -> {
        BigDecimal totalAmount = Stream.iterate(target.getStartDate(), date -> date.plusDays(1))
            .limit(ChronoUnit.DAYS.between(target.getStartDate(), target.getEndDate()) + 1)
            .map(date -> dateToAmountMap.getOrDefault(date, BigDecimal.ZERO)) // Map에서 O(1) 조회
            .reduce(BigDecimal.ZERO, BigDecimal::add);

        target.setCalculatedAmount(totalAmount);
    });
}
```

### 3.3. 성능 비교

- **전처리 시간 복잡도:** O(P_days), 여기서 P_days는 `Policy` 목록의 모든 날짜 구간의 총 일수.
- **계산 시간 복잡도:** O(T_days), 여기서 T_days는 `TargetPeriod` 목록의 모든 날짜 구간의 총 일수.
- **전체 시간 복잡도:** O(P_days + T_days). 이는 이중 루프의 **O(N * M)에 비해 비약적으로 빠른 성능**을 보장합니다.

## 4. 3단계 (심화): 데이터 구조 최적화와 병렬 처리

더 높은 수준의 성능이 필요하다면, 다음과 같은 기법을 고려할 수 있습니다.

- **메모리 최적화 (`int` 키 사용):** `LocalDate` 객체는 비교적 무겁습니다. `yyyyMMdd` 형식의 `int`를 `Map`의 키로 사용하면 메모리 사용량과 해시 충돌 가능성을 줄여 성능을 더 높일 수 있습니다. (e.g., `fastutil` 라이브러리의 `Int2ObjectOpenHashMap` 사용)
- **알고리즘 최적화 (`TreeMap` 사용):** `HashMap` 대신 `TreeMap`으로 전처리 `Map`을 만들면, `target`의 날짜 구간을 `subMap()` 메서드로 한 번에 추출할 수 있습니다. 이를 통해 `Stream.iterate`로 날짜를 일일이 생성하는 오버헤드를 줄일 수 있습니다.
- **병렬 처리 (`parallelStream`):** 전처리된 `Map`은 사실상 불변(effectively final) 데이터입니다. 따라서 `TargetPeriod`를 계산하는 로직에 `targets.parallelStream()`을 안전하게 적용하여 멀티코어 프로세서의 성능을 최대한 활용할 수 있습니다.

### 병렬 처리 적용 예시

```java
public void calculateInParallel(Map<LocalDate, BigDecimal> dateToAmountMap, List<TargetPeriod> targets) {
    targets.parallelStream().forEach(target -> { // .parallelStream() 으로 변경
        // ... 내부 계산 로직은 동일 ...
        BigDecimal totalAmount = Stream.iterate(...)
            .map(...)
            .reduce(...);
        target.setCalculatedAmount(totalAmount);
    });
}
```

## 5. 결론: 어떤 방법을 선택해야 할까?

| 접근 방식 | 시간 복잡도 | 장점 | 단점 | 추천 상황 |
| --- | --- | --- | --- | --- |
| **이중 `for` 루프** | O(N * M) | 직관적, 구현 용이 | 데이터가 많으면 성능 최악 | 프로토타입, 데이터가 매우 적을 때 |
| **중첩 `Stream`** | O(N * M) | 선언적, 가독성 | `for`문과 성능 동일 | `for`문을 함수형으로 바꾸고 싶을 때 |
| **`Map` 전처리** | O(P_days + T_days) | **성능 비약적 향상** | 전처리 오버헤드, 메모리 사용 | **대부분의 실무 상황 (Best Practice)** |
| **심화 최적화** | O(P_days + T_days) 이하 | 성능 극대화 | 구현 복잡도 증가 | 초고성능이 요구되는 특수 상황 |

복잡한 집계 및 계산 로직을 다룰 때, `Stream`을 맹신하는 것만으로는 충분하지 않을 때가 많습니다. 그보다 **적절한 자료구조(`Map` 등)를 선택하고, 데이터를 조회에 최적화된 형태로 '전처리'하는 알고리즘적 사고**가 훨씬 더 중요합니다. 이 전처리 패턴을 잘 익혀두면, 복잡한 비즈니스 요구사항 속에서도 높은 성능과 좋은 가독성을 모두 만족시키는 코드를 작성할 수 있을 것입니다.
