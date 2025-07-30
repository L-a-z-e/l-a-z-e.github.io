---
title: Fork/Join 프레임워크는 언제, 어떻게 사용해야 할까?
description: 
author: laze
date: 2025-07-30 00:00:01 +0900
categories: [Dev, Java]
tags: [Java, Stream, parallelStream, ForkJoin Framework]
---
# [Java] 병렬 처리의 정석: Fork/Join 프레임워크는 언제, 어떻게 사용해야 할까?

## 1. 서론: `parallelStream()`의 엔진을 만나다

Java 8에서 등장한 병렬 스트림(`parallelStream()`)은 많은 개발자들에게 간단한 병렬 처리의 길을 열어주었습니다.

하지만 이 편리한 기능의 심장부에는 Java 7부터 존재해 온, 더 강력하고 근본적인 기술인 **`Fork/Join 프레임워크`**가 뛰고 있습니다.

`Fork/Join 프레임워크`는 단순히 작업을 여러 스레드에 분배하는 것을 넘어, **'분할 정복(Divide and Conquer)'** 알고리즘과 **'작업 훔치기(Work Stealing)'**라는 정교한 메커니즘을 통해 CPU 코어를 최대한 효율적으로 활용하도록 설계되었습니다.

이 글에서는 Fork/Join 프레임워크의 핵심 원리를 파헤치고, `RecursiveTask` 예제 코드를 상세히 분석하며, 가장 중요한 **실무에서 어떤 종류의 문제에 이 기술을 적용해야 최대의 효과를 볼 수 있는지**에 대한 명확한 가이드를 제시합니다.

## 2. Fork/Join 프레임워크란? "쪼개고, 합치고, 훔쳐라!"

이 프레임워크의 철학은 **"어떻게 하면 CPU를 최대한 놀지 않게 할까?"** 라는 질문에 대한 답입니다. 이를 위해 두 가지 핵심 전략을 사용합니다.

### 2.1. 분할 정복 (Fork & Join)

'분할 정복'은 복잡한 문제를 해결하는 고전적인 알고리즘 기법입니다. Fork/Join 프레임워크는 이를 병렬 처리에 접목했습니다.

- **Fork (분기):** 주어진 큰 문제를 더 이상 쪼갤 수 없을 만큼 작은 단위(임계값, `THRESHOLD`)가 될 때까지 재귀적으로 계속 쪼갭니다.
- **Join (결합):** 쪼개진 작은 문제들의 해결 결과를 다시 재귀적으로 합쳐서, 최종적으로 원래 문제의 답을 구합니다.

### 2.2. 작업 훔치기 (Work Stealing)

이것이 바로 Fork/Join 프레임워크의 진짜 비밀 병기이자, 전통적인 스레드 풀과의 가장 큰 차이점입니다.

`ForkJoinPool`의 각 스레드는 자신만의 작업 큐(Deque, 양방향 큐)를 가지고 작업을 처리합니다.

만약 어떤 스레드가 자신의 일을 모두 끝내고 놀게 되면(idle), 그 스레드는 **다른 바쁜 스레드의 작업 큐의 꼬리(tail)에서 가장 오래된 작업을 '훔쳐와서(steal)'** 처리합니다.

이 '작업 훔치기' 메커니즘 덕분에, `ForkJoinPool` 내의 모든 스레드가 최대한 쉬지 않고 일하게 되어, CPU 활용률을 극대화하고 병렬 처리의 효율을 높일 수 있습니다.

## 3. 핵심 컴포넌트와 동작 흐름: 거대 배열 합계 구하기

Fork/Join 프레임워크를 구성하는 핵심 컴포넌트들을 실제 예제 코드를 통해 알아보겠습니다.

- **`ForkJoinPool`:** '작업 훔치기' 기능이 내장된 특별한 스레드 풀.
- **`RecursiveTask<V>`:** '결과값(V)이 있는' 작업을 정의하기 위한 추상 클래스.
- **`compute()`:** 우리가 직접 "쪼갤 것인가, 풀 것인가"를 결정하는 로직을 구현해야 하는 핵심 메서드.

### 예제: 1부터 10,000,000까지의 합계를 병렬로 구하기

### `SumTask.java`

```java
import java.util.concurrent.RecursiveTask;

public class SumTask extends RecursiveTask<Long> {
    private final long[] numbers;
    private final int start;
    private final int end;

    // 이 크기보다 작아지면 더 이상 쪼개지 않고 직접 계산한다.
    private static final long THRESHOLD = 10_000;

    public SumTask(long[] numbers, int start, int end) {
        this.numbers = numbers;
        this.start = start;
        this.end = end;
    }

    @Override
    protected Long compute() {
        int length = end - start;

        // 1. 기본 사례(Base Case): 작업이 충분히 작으면 직접 계산한다.
        if (length <= THRESHOLD) {
            long sum = 0;
            for (int i = start; i < end; i++) {
                sum += numbers[i];
            }
            return sum;
        }

        // 2. 재귀 사례(Recursive Case): 작업을 두 개의 하위 작업으로 쪼갠다.
        int mid = start + length / 2;
        SumTask leftTask = new SumTask(numbers, start, mid);
        SumTask rightTask = new SumTask(numbers, mid, end);

        // 3. 한쪽은 비동기로 실행하고, 다른 쪽은 현재 스레드에서 직접 실행 (효율성 최적화)
        leftTask.fork(); // 왼쪽 작업은 다른 스레드에 맡긴다 (비동기).
        Long rightResult = rightTask.compute(); // 오른쪽 작업은 현재 스레드에서 즉시 실행한다.
        Long leftResult = leftTask.join(); // 왼쪽 작업이 끝날 때까지 기다렸다가 결과를 가져온다.

        // 4. 두 하위 작업의 결과를 합친다.
        return leftResult + rightResult;
    }
}
```

### `Main.java` (실행 코드)

```java
import java.util.concurrent.ForkJoinPool;
import java.util.stream.LongStream;

public class Main {
    public static void main(String[] args) {
        // CPU 코어 수에 맞춰 최적화된 공용 스레드 풀을 사용할 수도 있다.
        // ForkJoinPool commonPool = ForkJoinPool.commonPool();

        ForkJoinPool forkJoinPool = new ForkJoinPool();

        long[] numbers = LongStream.rangeClosed(1, 10_000_000).toArray();
        SumTask task = new SumTask(numbers, 0, numbers.length);

        // invoke()는 작업이 완료될 때까지 기다렸다가 최종 결과를 반환한다.
        long result = forkJoinPool.invoke(task);
        System.out.println("최종 합계: " + result);
    }
}
```

**`leftTask.fork()` → `rightTask.compute()` → `leftTask.join()`** 순서는 중요한 최적화 기법입니다.

두 작업(`left`, `right`)을 모두 `fork()`하고 각각 `join()`하면, 현재 스레드는 두 작업이 모두 끝날 때까지 아무 일도 하지 않고 기다리게 됩니다.

하지만 위 방식은, 한쪽 작업이 다른 스레드에서 실행되는 동안 현재 스레드도 놀지 않고 나머지 절반의 작업을 직접 처리하므로, 스레드 자원을 더 효율적으로 사용하게 됩니다.

## 4. 실무 가이드: 언제 Fork/Join을 써야 할까?

Fork/Join 프레임워크는 모든 병렬 처리 문제의 만병통치약이 아닙니다.

문제의 성격을 정확히 파악하고, 그에 맞는 도구를 사용하는 것이 핵심입니다.

### 👍 Best Cases: Fork/Join이 빛을 발하는 경우

Fork/Join 프레임워크는 **CPU를 최대한 활용해야 하는 계산 집약적(CPU-Bound) 작업**에 가장 적합합니다.

1. **순수 계산 작업:**
  - **설명:** 외부 I/O 없이, 주어진 데이터만으로 복잡한 연산을 수행하는 작업.
  - **실무 예시:**
    - 거대한 인메모리 배열/리스트의 데이터 처리 (합계, 평균, 필터링, 변환).
    - 이미지 렌더링 또는 필터 적용.
    - 과학/금융 시뮬레이션 및 복잡한 수학적 모델링.
    - 대규모 데이터의 암호화 또는 압축.
2. **명확한 분할 정복(Divide and Conquer) 구조:**
  - **설명:** 문제가 재귀적으로 자기 자신과 똑같은 형태의 더 작은 문제로 쉽게 나뉠 수 있는 경우.
  - **실무 예시:**
    - 커스텀 정렬 알고리즘 구현 (병합 정렬, 퀵 정렬).
    - 대규모 파일 시스템의 트리 구조 순회 및 파일 검색.
    - 재귀적으로 정의된 데이터 구조(e.g., 조직도) 분석.

### 👎 Worst Cases: Fork/Join을 피해야 하는 경우

Fork/Join 프레임워크는 **스레드가 대기(Blocking) 상태에 빠지는 I/O 집약적(I/O-Bound) 작업**에는 매우 비효율적이며, 오히려 성능을 저하시킬 수 있습니다.

1. **I/O 집약적 작업:**
  - **설명:** 작업 시간의 대부분을 네트워크 응답이나 디스크 읽기를 기다리는 데 사용하는 작업.
  - **이유:** Fork/Join 스레드가 I/O 작업으로 인해 블로킹(대기) 상태에 빠지면, **다른 작업을 훔쳐올 수 없어(Work Stealing 불가)** 스레드 풀의 핵심 장점이 무력화됩니다. 결국 스레드는 일도 안 하고 자원만 차지하는 최악의 상황이 발생합니다.
  - **실무 예시:**
    - **데이터베이스 조회 (JDBC 호출)**
    - **외부 REST API 호출**
    - **파일 시스템 읽기/쓰기**
    - 메시지 큐(Kafka, RabbitMQ)에서 메시지를 기다리는 작업
  - **대안:** 이런 종류의 작업에는 **`CompletableFuture`*나 **리액티브 프로그래밍 (Spring Webflux, Project Reactor)**을 사용하는 것이 훨씬 더 적합하고 효율적입니다.
2. **공유된 가변 상태(Shared Mutable State)에 크게 의존하는 작업:**
  - **설명:** 여러 스레드가 하나의 공유된 객체나 변수를 동시에 수정해야 하는 작업.
  - **이유:** 스레드 안전성을 보장하기 위해 `synchronized` 블록이나 `Lock`을 사용해야 합니다. 이는 스레드 간의 심각한 경합을 유발하여 병렬 처리의 이점을 상쇄하고, 순차 처리보다 오히려 성능이 크게 저하될 수 있습니다.

## 5. 결론: 올바른 도구를 올바른 문제에 사용하기

Fork/Join 프레임워크는 모든 병렬 처리 문제를 해결하는 만능 도구가 아닙니다. 그것은 **CPU를 한계까지 활용해야 하는 계산 집약적인 문제**를 **분할 정복**이라는 우아한 방식으로 해결하기 위해 태어난 고도로 특화된 전문 도구입니다.

실무에서 병렬 처리 요구사항을 마주했을 때, 가장 먼저 해야 할 일은 **"이 작업은 CPU-Bound인가, I/O-Bound인가?"**를 분석하는 것입니다. 문제의 성격을 정확히 파악하고 그에 맞는 병렬 처리 기술을 선택하는 것이, 성공적인 고성능 애플리케이션을 만드는 첫걸음입니다.
