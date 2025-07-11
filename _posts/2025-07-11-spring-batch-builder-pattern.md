---
title: build().build() 빌더 체인의 변신 과정
description: 
author: laze
date: 2025-07-11 00:00:04 +0900
categories: [Dev, SpringBatch]
tags: [SpringBatch, Builder, BuilderChain]
---
# [Spring Batch] `build().build()`: 빌더 체인의 변신 과정

Spring Batch로 잡(Job)을 구성하다 보면, 특히 조건 분기를 추가할 때 `...build().build()`처럼 `build()`를 두 번 호출하는 기이한 코드를 마주하게 됩니다.

```java
return new JobBuilder("myJob", jobRepository)
    .start(step1)
    .next(myDecider) // 조건 분기 추가
    .on("COMPLETED").to(step2)
    .build() // <-- 왜 build()가 여기 있지?
    .build(); // <-- 마지막 build()는 알겠는데...
```

처음 이 코드를 보면 "버그인가?", "내가 뭘 잘못 썼나?" 하는 생각이 들기 마련입니다. 하지만 이것은 버그가 아니라, Spring Batch의 **'계층적 빌더(Hierarchical Builder)'** 패턴이 동작하는 방식이며, 그 설계에는 명확한 의도가 담겨 있습니다.

이 글에서는 왜 `build()`를 두 번 호출해야 하는지, 그 내부에서 벌어지는 **빌더들의 연쇄 변신 과정**을 상세하게 추적해 보겠습니다.

## 1. 전제: Lombok의 `@Builder`와 Spring Batch의 빌더는 다르다

먼저 우리가 흔히 아는 Lombok의 `@Builder`와 Spring Batch의 빌더는 동작 방식이 다릅니다.

- **Lombok `@Builder`:** 모든 `setter` 메서드는 **항상 동일한 빌더 객체 자신**을 반환합니다. 마지막 `build()`만 최종 객체를 생성합니다.
- **Spring Batch 빌더:** 메서드 체인의 **단계(상태)에 따라, 다른 역할을 가진 새로운 빌더 객체를 반환**하며 제어권을 넘깁니다.

이 '빌더의 변신'이 바로 `build().build()` 미스터리의 핵심 열쇠입니다.

## 2. 레스토랑 비유로 이해하는 빌더의 여정

복잡한 내부 클래스 구조를 파헤치기 전에, 레스토랑에서 코스 요리를 주문하는 상황에 비유해 봅시다.

- **`JobBuilder` (전체 Job 생성 담당):** 주문을 받는 '총괄 웨이터'
- **`FlowBuilder` (흐름 제어 담당):** 복잡한 요리 순서를 결정하는 '주방장'
1. **"코스 요리 주문할게요."**
  - `new JobBuilder(...)` → **총괄 웨이터**가 주문을 받기 시작합니다.
2. **"에피타이저로 스프 주세요."**
  - `.start(step)` → 웨이터가 간단한 주문을 받습니다. 아직 웨이터의 역할입니다.
3. **"메인 디쉬는 날씨 보고 결정해주세요."**
  - `.next(decider)` → 복잡한 요청이 들어왔습니다! 웨이터는 이 요청을 **주방장**에게 넘깁니다. **(빌더의 제어권 전환)**
4. **"날씨가 좋으면('ON COMPLETE') 스테이크를('TO steakStep') 주세요."**
  - `.on(...).to(...)` → 이제부터는 **주방장**의 전문 영역입니다. 주방장이 요리 순서(흐름)를 결정합니다.
5. **"메인 디쉬 결정 끝났습니다." (`.build()`)**
  - **첫 번째 `build()` 호출:** 주방장이 결정된 '메인 요리 흐름(Flow)'을 완성해서, 다시 웨이터에게 전달합니다. **(주방장의 임무 완료)**
6. **"전체 코스 요리 주문 끝났습니다." (`.build()`)**
  - **두 번째 `build()` 호출:** 웨이터가 에피타이저와 주방장에게 전달받은 메인 요리 흐름을 모두 취합하여, 최종 '코스 요리 주문서(Job)'를 완성합니다. **(웨이터의 임무 완료)**

이처럼 `build()`가 두 번 나오는 이유는, **각기 다른 책임(Flow, Job)을 가진 빌더가 자신의 임무를 끝낼 때마다 `build()`를 호출**하기 때문입니다.

## 3. 코드로 추적하는 빌더의 연쇄 변신 과정

이제 실제 코드가 어떤 타입의 빌더 객체를 반환하며 변신하는지 단계별로 추적해 보겠습니다.

**분석 대상 코드:**

```java
// Job 생성을 시작한다.
new JobBuilder("myJob", jobRepository)
    .start(step1)                      // (A)
    .next(myDecider)                   // (B) ★ 변신의 순간
    .on("COMPLETED").to(step2)         // (C)
    .build()                           // (D) 첫 번째 build()
    .build();                          // (E) 두 번째 build()
```

### [A단계] `.start(step)`

- **호출:** `new JobBuilder(...).start(step1)`
- **반환 타입:** `SimpleJobBuilder`
- **설명:** `JobBuilder`는 `start()`가 호출되는 순간, Step들의 흐름을 구성할 수 있는 `SimpleJobBuilder`를 반환하며 첫 번째 변신을 합니다. 이제부터 우리는 `Job`을 만드는 여정을 시작합니다.

### [B단계] `.next(decider)` - ★ 변신의 순간

- **호출:** `(SimpleJobBuilder 객체).next(myDecider)`
- **반환 타입:** `JobFlowBuilder`
- **설명:** `SimpleJobBuilder`는 `Decider`를 인자로 받는 `next()`가 호출되자, "아, 이제부터는 조건 분기(Flow)를 만들어야 하는구나!"라고 인지합니다. 그리고 **흐름(Flow)을 전문적으로 다루는 `JobFlowBuilder`에게 제어권을 넘깁니다.** 이제 우리는 `Job`의 세계에서 잠시 벗어나 `Flow`의 세계로 들어왔습니다.

### [C단계] `.on("COMPLETED").to(step2)`

- **호출:** `(JobFlowBuilder 객체).on(...).to(...)`
- **반환 타입:** `FlowBuilder` (정확히는 그 내부 상태를 관리하는 빌더)
- **설명:** `Flow`의 세계에서는 `.on(상태).to(대상)` 문법을 사용하여 "Decider가 'COMPLETED'를 반환하면, step2로 가라"는 식의 **전환(Transition) 규칙**을 정의합니다.

### [D단계] `build()` - 첫 번째 빌드: `Flow`의 완성

- **호출:** `(FlowBuilder 객체).build()`
- **반환 타입:** `SimpleJobBuilder` (제어권 복귀!)
- **설명:** `FlowBuilder`는 `build()`가 호출되면, 지금까지 정의된 모든 전환 규칙을 종합하여 하나의 응축된 **`Flow` 객체**를 만듭니다. 그리고 "내 임무는 끝났다"며, 다시 제어권을 자신을 불렀던 **`SimpleJobBuilder`에게 돌려줍니다.** 이때 생성된 `Flow`는 `SimpleJobBuilder`의 다음 실행 대상으로 등록됩니다.

### [E단계] `build()` - 두 번째 빌드: `Job`의 완성

- **호출:** `(SimpleJobBuilder 객체).build()`
- **반환 타입:** `Job`
- **설명:** `Flow`의 세계에서 다시 `Job`의 세계로 돌아온 `SimpleJobBuilder`는 `build()`가 호출되자, 지금까지 등록된 모든 Step과 Flow들을 순서대로 종합하여, 최종적으로 우리가 원했던 **`Job` 객체**를 생성합니다. 모든 빌더의 여정이 여기서 끝납니다.

## 4. 케이스별 정리: 언제 `build()`를 한 번/두 번 쓰는가?

### `build()` 한 번: 단순 순차 흐름

조건 분기나 병렬 처리 없이, `start-next-next`로만 이어지는 단순한 구조일 때는 `FlowBuilder`가 등장할 일이 없습니다. `SimpleJobBuilder`가 처음부터 끝까지 모든 것을 처리하므로 `build()`는 한 번만 필요합니다.

```java
@Bean
public Job sequentialJob() {
    return new JobBuilder("sequentialJob", jobRepository)
            .start(stepA())
            .next(stepB())
            .next(stepC())
            .build(); // SimpleJobBuilder가 Job을 생성하며 임무 완료!
}
```

### `build()` 두 번: 조건 분기 또는 병렬 흐름

`next(decider)`나 `split(taskExecutor)`처럼 복잡한 흐름 제어가 시작되면, `FlowBuilder`가 개입하여 `Flow`를 만들기 때문에 `build()`가 두 번 필요합니다.

```java
@Bean
public Job conditionalJob() {
    return new JobBuilder("conditionalJob", jobRepository)
            .start(stepA())
            .next(myDecider()) // --> Flow의 세계로 진입
            .on("CONTINUE").to(stepB())
            .from(myDecider()).on("STOP").to(stepC())
            .end() // .end()로 Flow를 마무리 짓고 JobBuilder로 복귀하는 방법도 있음
            // 또는 .build()를 호출
            .build(); // --> Job을 최종 생성
}
```

**참고:** `.end()`를 사용하면 `Flow` 정의를 마치고 `SimpleJobBuilder`로 복귀할 수 있습니다.

`.build().build()`는 `.end()`를 사용하지 않고 `Flow`를 독립된 객체로 만든 뒤 `Job`에 포함시키는 방식에 해당합니다. 둘 다 비슷한 맥락으로 이해할 수 있습니다.

## 5. 실전 팁: 가독성을 위한 `Flow` 분리

`build().build()` 체인이 너무 길고 복잡하게 느껴진다면, `Flow` 자체를 별도의 Bean이나 변수로 분리하여 가독성을 높일 수 있습니다.

```java
@Bean
public Flow myComplexFlow() {
    FlowBuilder<Flow> flowBuilder = new FlowBuilder<>("myComplexFlow");
    return flowBuilder
            .start(myDecider())
            .on("COMPLETED").to(successStep())
            .from(myDecider()).on("FAILED").to(failureStep())
            .build();
}

@Bean
public Job myJobWithFlow(Flow myComplexFlow) {
    return new JobBuilder("myJobWithFlow", jobRepository)
            .start(initialStep())
            .next(myComplexFlow) // 미리 정의된 Flow를 주입
            .end()
            .build(); // Job 생성은 한 번만!
}
```

## 6. 결론

Spring Batch의 `build().build()`는 버그나 문법 오류가 아니라, **'Job'이라는 큰 단위와 'Flow'라는 세부 흐름 단위를 계층적으로 분리하여 설정**하기 위한 매우 정교하고 의도된 설계입니다.

이 빌더 체인의 변신 과정을 이해하면, 왜 코드가 그렇게 작성되었는지 명확히 알 수 있으며, 더 복잡하고 유연한 배치 흐름을 자신감 있게 구성할 수 있게 될 것입니다.

---
