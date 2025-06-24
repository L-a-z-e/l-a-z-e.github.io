---
title: Spring Batch - Review(3)
description: 
author: laze
date: 2025-06-24 00:00:02 +0900
categories: [Dev, SpringBatch]
tags: [SpringBatch]
---
**Job 내에서 여러 Step들의 실행 순서를 정의하는 가장 기본적인 방법은 무엇이며, 특정 Step의 실행 결과(ExitStatus)에 따라 다음에 실행될 Step을 다르게 하고 싶을 때 JobBuilder에서 주로 어떤 메서드들을 활용하게 되나요?**

### **`Job` 내 `Step` 실행 순서 제어 (Flow Control)**

`JobBuilder`를 사용하여 `Job`을 정의할 때, 여러 `Step`들의 실행 순서와 조건을 지정할 수 있습니다.

**1. 기본적인 순차 실행: `.start()` 와 `.next()`**

가장 기본적인 방법은 `Step`들을 순서대로 연결하는 것입니다.

- **`.start(Step step)`**: `Job`이 시작될 때 가장 먼저 실행될 `Step`을 지정합니다.
- **`.next(Step step)`**: 바로 이전에 지정된 `Step`이 **성공적으로 완료되면 (`ExitStatus`가 "COMPLETED"이면)** 다음에 실행될 `Step`을 지정합니다.

**예시 코드:**

```java
@Configuration
public class SequentialJobConfig {

    // ... (jobRepository, transactionManager, step1, step2, step3 빈 정의는 생략) ...

    @Bean
    public Job sequentialJob(JobRepository jobRepository, Step step1, Step step2, Step step3) {
        return new JobBuilder("sequentialJob", jobRepository)
                .start(step1)  // 1. step1 부터 시작
                .next(step2)   // 2. step1이 성공하면 step2 실행
                .next(step3)   // 3. step2가 성공하면 step3 실행
                .build();      // 4. step3이 성공하면 Job 전체가 성공으로 완료
    }
}
```

- **동작 흐름:**
  - `step1` 실행 -> 성공 시 `step2` 실행 -> 성공 시 `step3` 실행 -> 성공 시 `Job` 완료.
  - 만약 `step1`이 실패하면, `step2`와 `step3`은 실행되지 않고 `Job` 전체가 실패합니다.
  - 마찬가지로 `step2`가 실패하면 `step3`은 실행되지 않고 `Job`이 실패합니다.

**2. 조건부 플로우: `.on(String pattern).to(Step nextStep)` 와 `.from(Step currentStep)`**

특정 `Step`의 실행 결과(**`ExitStatus`**)에 따라 다음에 실행될 `Step`을 다르게 지정할 수 있습니다.

- **`ExitStatus`**: `Step` 실행 후의 상태를 나타내는 문자열입니다. 기본적으로 "COMPLETED", "FAILED" 등이 있지만, 개발자가 커스텀한 `ExitStatus`를 만들 수도 있습니다. (예: "COMPLETED WITH SKIPS", "NO_DATA")
- **`.on(String pattern)`**: 이전 `Step`의 `ExitStatus`가 주어진 `pattern`과 일치하는 경우를 지정합니다.
  - 패턴에는 와일드카드  (0개 이상의 문자)와 `?` (정확히 1개의 문자)를 사용할 수 있습니다.
  - 예: `.on("COMPLETED")`, `.on("FAILED")`, `.on("C*")`, `.on("*")` (모든 경우)
- **`.to(Step nextStep)`**: `.on()` 조건이 만족될 때 다음에 실행할 `Step`을 지정합니다.
- **`.from(Step currentStep)`**: (선택 사항이지만 가독성을 높임) 어떤 `Step`으로부터의 전환인지를 명시적으로 지정합니다. `.on()` 이전에 사용됩니다. `.start()` 이후 첫 번째 `.on()`에는 `.from()`이 필요 없지만, 두 번째 분기부터는 어떤 스텝으로부터의 분기인지 명시하기 위해 `.from()`을 사용합니다.
- **플로우 종료자 (Flow Enders):** `.on()` 조건 이후에 `.to(Step)` 대신 다음과 같은 메서드들을 사용하여 `Job`의 최종 상태를 결정하며 플로우를 종료할 수 있습니다.
  - **`.end()`**: `Job`을 `BatchStatus.COMPLETED` 상태로 종료시킵니다. (선택적으로 `ExitStatus` 지정 가능: `.end("CUSTOM_COMPLETED_STATUS")`)
  - **`.fail()`**: `Job`을 `BatchStatus.FAILED` 상태로 종료시킵니다.
  - **`.stopAndRestart(Step restartStep)`**: `Job`을 `BatchStatus.STOPPED` 상태로 중지시키고, 재시작 시 `restartStep`부터 실행되도록 합니다.

**예시 코드 (조건부 플로우):**

```java
@Configuration
public class ConditionalJobConfig {

    // ... (jobRepository, transactionManager, stepA, stepB, stepC, errorStep 빈 정의는 생략) ...

    @Bean
    public Job conditionalJob(JobRepository jobRepository,
                              Step stepA, Step stepB, Step stepC, Step errorStep) {
        return new JobBuilder("conditionalJob", jobRepository)
                .start(stepA)                                 // 1. stepA 시작
                    .on("FAILED").to(errorStep)               // 2. stepA가 "FAILED" ExitStatus로 끝나면 errorStep 실행
                    .on("COMPLETED_WITH_WARNINGS").to(stepC)  // 3. stepA가 "COMPLETED_WITH_WARNINGS"로 끝나면 stepC 실행
                .from(stepA)                                  // 4. stepA로부터의 다른 전환 정의 (위 조건에 해당 안 될 시)
                    .on("*").to(stepB)                       // 5. stepA가 그 외 어떤 ExitStatus로 끝나든 (예: "COMPLETED") stepB 실행
                .from(stepB)                                  // 6. stepB로부터의 전환 정의
                    .on("FAILED").fail()                      // 7. stepB가 "FAILED"로 끝나면 Job 전체를 FAILED로 종료
                    .on("*").to(stepC)                       // 8. stepB가 그 외 어떤 ExitStatus로 끝나든 stepC 실행
                .from(stepC)                                  // 9. stepC로부터의 전환 정의
                    .on("*").end()                           // 10. stepC가 어떤 ExitStatus로 끝나든 Job 전체를 COMPLETED로 종료
                .from(errorStep)                              // 11. errorStep으로부터의 전환 정의
                    .on("*").fail()                           // 12. errorStep이 어떤 ExitStatus로 끝나든 Job 전체를 FAILED로 종료
                .end() // JobBuilder의 플로우 정의 전체를 마무리하는 .end() (여러 .from()...end() 그룹 후 사용)
                .build();
    }
}
```

**동작 흐름 분석 (위 예시):**

1. `stepA`가 실행됩니다.
2. `stepA`의 `ExitStatus`를 확인합니다.
  - 만약 "FAILED"이면, `errorStep`으로 갑니다. `errorStep` 실행 후 (12번 규칙에 의해) `Job`은 `FAILED`로 끝납니다.
  - 만약 "COMPLETED\_WITH\_WARNINGS"이면, `stepC`로 갑니다. `stepC` 실행 후 (10번 규칙에 의해) `Job`은 `COMPLETED`로 끝납니다.
  - 만약 위 두 경우가 아니고 다른 `ExitStatus`(예: "COMPLETED")이면, `stepB`로 갑니다.
3. `stepB`가 실행되었다면, `stepB`의 `ExitStatus`를 확인합니다.
  - 만약 "FAILED"이면, (7번 규칙에 의해) `Job`은 즉시 `FAILED`로 끝납니다.
  - 만약 다른 `ExitStatus`이면, `stepC`로 갑니다. `stepC` 실행 후 (10번 규칙에 의해) `Job`은 `COMPLETED`로 끝납니다.

**중요 포인트:**

- **`ExitStatus` 매칭 순서:** Spring Batch는 `.on()` 조건들을 **가장 구체적인 패턴부터 덜 구체적인 패턴 순서로** 내부적으로 정렬하여 매칭을 시도합니다. 따라서 `on("*")`는 항상 가장 마지막에 평가됩니다. 위 예제에서 `from(stepA).on("FAILED")`가 `from(stepA).on("*")`보다 먼저 평가됩니다.
- **`.from(Step)`의 역할:**
  - `.start(stepA).on("A").to(stepB).on("B").to(stepC)` 와 같이 `.from()` 없이 `.on()`을 여러 번 연결하면, 모든 `.on()`은 바로 앞의 `.start(stepA)` 또는 `.next(stepX)`에 대한 분기 조건이 됩니다.
  - `.start(stepA).on("A").to(stepB).from(stepA).on("B").to(stepC)` 처럼 `.from(stepA)`를 사용하면, "stepA로부터의 전환"임을 명확히 하고, 이전 `.on()`과는 별개의 분기 그룹을 시작하는 것처럼 코드를 구성할 수 있습니다. 가독성을 높이고 복잡한 분기를 명확하게 표현하는 데 도움이 됩니다.
  - **JobBuilder의 Fluent API:** `JobBuilder`는 메서드 체이닝(fluent API)을 통해 플로우를 정의하는데, `.from(step)`은 이 체이닝의 "현재 컨텍스트 스텝"을 `step`으로 변경하는 역할을 합니다. 그 이후의 `.on().to()` 등은 이 새로운 컨텍스트 스텝으로부터의 전환을 정의하게 됩니다.
- **플로우 빌더의 `.end()`:**
  - `JobBuilder`에서 여러 `.from()... .to()... .end()` (여기서 `.end()`는 각 from 블록의 마침표 역할) 와 같은 분기 그룹을 정의한 후, 최종적으로 `.build()` 하기 전에 `JobBuilder` 자체의 플로우 정의를 마감하는 의미로 `.end()`를 한 번 더 사용할 수 있습니다. (위 예제의 마지막 `.end()`)
  - `.on("...").end()` 와 같이 `to()` 대신 사용되면, 해당 조건에서 Job을 `COMPLETED` 상태로 **종료**시키라는 의미도 가집니다.

**3. `JobExecutionDecider` (Decider) - 프로그래밍 방식의 분기**

`ExitStatus`만으로는 분기 로직을 표현하기 부족할 때, Java 코드로 직접 다음 실행할 흐름 상태를 결정할 수 있습니다.

- `JobExecutionDecider` 인터페이스를 구현한 빈을 만듭니다.
- `decide(JobExecution jobExecution, StepExecution stepExecution)` 메서드에서 로직을 구현하고 `FlowExecutionStatus` (내부에 다음 상태 문자열 포함)를 반환합니다.
- `JobBuilder`에서 `.next(myDeciderBean).on("DECIDER_STATUS_A").to(stepA)...` 와 같이 사용합니다.

**예시 코드 (Decider 사용):**

```java
// MyDecider.java
public class MyDecider implements JobExecutionDecider {
    @Override
    public FlowExecutionStatus decide(JobExecution jobExecution, StepExecution stepExecution) {
        // JobParameters, ExecutionContext, 또는 stepExecution 결과 등을 참조하여 조건 판단
        boolean someCondition = jobExecution.getJobParameters().getLong("count", 0L) > 10;
        if (someCondition) {
            return new FlowExecutionStatus("CONDITION_MET");
        } else {
            return new FlowExecutionStatus("CONDITION_NOT_MET");
        }
    }
}

// Job Configuration
@Configuration
public class DeciderJobConfig {
    // ... (jobRepository, stepX, stepY 빈 정의 및 MyDecider 빈 정의) ...

    @Bean
    public Job deciderJob(JobRepository jobRepository, Step initialStep, MyDecider myDecider, Step stepX, Step stepY) {
        return new JobBuilder("deciderJob", jobRepository)
                .start(initialStep)
                .next(myDecider) // initialStep 후 Decider 실행
                    .on("CONDITION_MET").to(stepX)       // Decider가 "CONDITION_MET" 반환 시 stepX 실행
                .from(myDecider)
                    .on("CONDITION_NOT_MET").to(stepY)   // Decider가 "CONDITION_NOT_MET" 반환 시 stepY 실행
                .from(stepX).on("*").end()
                .from(stepY).on("*").end()
                .end()
                .build();
    }
}
```

---

- **`.start(step)`**: 시작점.
- **`.next(step)`**: 이전 스텝 성공 시 다음 스텝 (기본 흐름).
- **`.on(exitStatusPattern).to(nextStepOrFlowEnder)`**: 조건부 분기.
- **`.from(currentStep)`**: 분기의 시작점을 명확히 함 (가독성 및 복잡한 분기).
- **`.end()`, `.fail()`, `.stopAndRestart()`**: 플로우의 종착점 및 Job 상태 결정.
- **`Decider`**: 프로그래밍 방식의 고급 분기.

**1. `skip` (건너뛰기) 전략**

- **개념:** `ItemReader`, `ItemProcessor`, `ItemWriter`에서 특정 아이템을 처리하는 도중 예외가 발생했을 때, 해당 예외가 "건너뛰기 가능한(skippable)" 것으로 설정되어 있다면, 문제가 된 아이템 처리를 포기(건너뛰고)하고 다음 아이템 처리를 계속 진행하는 전략입니다.
- **주요 설정 (`StepBuilder`의 `faultTolerant()` 이후):**
  - **`skippableExceptionClasses(Class<? extends Throwable>... types)`**: 건너뛸 예외 유형들을 지정합니다. 이 예외들이 발생하면 skip 로직이 동작합니다.
  - **`skipLimit(int skipLimit)`**: 스텝 전체에서 건너뛸 수 있는 최대 아이템 개수를 지정합니다. 이 한도를 초과하면 스텝은 실패합니다.
  - `noRollback(Class<? extends Throwable>... types)`: 특정 예외에 대해서는 롤백을 수행하지 않도록 지정할 수 있습니다. (주의해서 사용)
  - `skipPolicy(SkipPolicy policy)`: 더 복잡한 skip 조건을 프로그래밍 방식으로 정의할 수 있는 커스텀 `SkipPolicy`를 설정합니다.
- **동작:**
  - 읽기/처리/쓰기 중 `skippableExceptionClasses`에 등록된 예외가 발생하면,
  - 현재 청크는 롤백됩니다 (기본적으로).
  - 문제가 된 아이템을 식별하고, skip 카운트를 증가시킵니다.
  - skip 카운트가 `skipLimit`을 넘지 않았다면, 해당 아이템을 제외하고 다음 아이템부터 (또는 다음 청크부터) 처리를 재개합니다.
  - `SkipListener`를 통해 skip된 아이템과 발생한 예외에 대한 정보를 받아 로깅 등의 후속 처리를 할 수 있습니다.

**2. `retry` (재시도) 전략**

> retry는 보통 db연결 등 불안정한 상황에서 여러번 재시도 해서 성공가능한 것들을 처리하기 좋지
>
- **개념:** 특정 아이템 처리 중 일시적인 오류(예: 네트워크 지연, DB 락(lock) 경쟁, 일시적인 외부 서비스 오류 등)로 인해 예외가 발생했을 때, 동일한 아이템에 대한 작업을 즉시 또는 약간의 시간 간격을 두고 몇 번 더 재시도하는 전략입니다.
- **주요 설정 (`StepBuilder`의 `faultTolerant()` 이후):**
  - **`retryableExceptionClasses(Class<? extends Throwable>... types)`**: 재시도할 예외 유형들을 지정합니다.
  - **`retryLimit(int retryLimit)`**: 각 아이템당 최대 재시도 횟수를 지정합니다. 이 횟수를 초과해도 계속 실패하면, 그때는 skip 정책이 적용되거나 스텝이 최종 실패합니다.
  - **`backOffPolicy(BackOffPolicy policy)`**: 재시도 간의 대기 시간을 설정합니다. (예: `FixedBackOffPolicy` - 고정 시간 대기, `ExponentialBackOffPolicy` - 대기 시간 점진적 증가)
  - `retryPolicy(RetryPolicy policy)`: 더 복잡한 재시도 조건을 프로그래밍 방식으로 정의할 수 있는 커스텀 `RetryPolicy`를 설정합니다.
- **동작:**
  - 읽기/처리/쓰기 중 `retryableExceptionClasses`에 등록된 예외가 발생하면,
  - 현재 청크는 롤백됩니다 (기본적으로).
  - `retryLimit` 내에서, 그리고 `backOffPolicy`에 따라 대기 후, 문제가 된 아이템부터 **다시** 읽거나 처리하거나 쓰려고 시도합니다.
  - 재시도 중 성공하면 다음 아이템으로 넘어가고, `retryLimit`까지 모두 실패하면 해당 아이템은 최종적으로 실패 처리됩니다. (이후 skip 정책이 있다면 skip 시도, 없다면 스텝 실패)
  - `RetryListener`를 통해 재시도 시도, 성공, 최종 실패 등의 이벤트에 대한 정보를 받을 수 있습니다.

**Skip과 Retry의 관계:**

- 보통 `retryLimit`까지 재시도를 모두 소진하고도 실패한 예외가 `skippableExceptionClasses`에도 포함되어 있다면, 그제서야 `skip` 로직이 동작하게 됩니다. 즉, **Retry가 Skip보다 우선적으로 시도**됩니다.

---

**3. 동일 파라미터로 동일한 잡을 재실행하는 옵션**

네, 이 부분은 실무에서 테스트하거나 특정 상황에서 필요할 수 있는 기능이죠. Spring Batch는 기본적으로 성공적으로 완료된(`COMPLETED`) `JobInstance` (동일 잡 이름 + 동일 `JobParameters`)의 재실행을 막습니다. 이를 우회하거나 제어하는 몇 가지 방법이 있습니다.

1. **`JobParametersIncrementer` 사용 (권장되는 방식 중 하나):**
  - **개념:** `Job` 설정 시 `JobParametersIncrementer`를 등록하면, `JobLauncher`가 잡을 실행할 때마다 이 `Incrementer`가 기존 `JobParameters`에 **새로운 파라미터를 추가하거나 기존 파라미터 값을 변경**하여 항상 **새로운 `JobInstance`*가 생성되도록 합니다.
  - **예시 (`RunIdIncrementer`):** Spring Batch가 기본으로 제공하는 `RunIdIncrementer`는 "[run.id](http://run.id/)"라는 `Long` 타입 파라미터를 매번 1씩 증가시켜 추가합니다.

      ```java
      @Bean
      public Job myJob(JobRepository jobRepository, Step step1) {
          return new JobBuilder("myJob", jobRepository)
                  .incrementer(new RunIdIncrementer()) // JobParameters에 "run.id" 자동 추가 및 증가
                  .start(step1)
                  .build();
      }
      ```

  - **장점:** 매번 고유한 `JobInstance`가 생성되므로, Spring Batch의 재실행 방지 로직에 걸리지 않고 항상 잡을 실행할 수 있습니다. 각 실행 이력도 별도로 관리됩니다. 테스트 시 매우 유용합니다.
  - **주의:** "논리적으로 동일한 작업"을 여러 번 실행하는 것이 아니라, "매번 약간 다른 파라미터로 새로운 작업 인스턴스를 실행"하는 개념에 가깝습니다.
2. **실행 시점에 항상 다른 `JobParameters` 전달:**
  - `JobLauncher.run(job, jobParameters)`를 호출할 때, 애플리케이션 코드 레벨에서 매번 다른 값을 가진 `JobParameter` (예: 현재 타임스탬프)를 생성하여 전달하는 방식입니다. `JobParametersIncrementer`와 유사한 효과를 냅니다.
  - 예시:

      ```java
      JobParameters params = new JobParametersBuilder()
              .addDate("timestamp", new Date()) // 매번 현재 시간을 파라미터로 추가
              .addString("inputFile", "input.csv") // 다른 파라미터
              .toJobParameters();
      jobLauncher.run(myJob, params);
      ```

3. **`JobRepository`에서 이전 `JobExecution` 상태 변경 (고급, 주의 필요):**
  - **개념:** 이미 `COMPLETED` 상태인 `JobInstance`가 있다면, `JobExplorer`를 사용하여 해당 `JobInstance`의 마지막 `JobExecution` 상태를 `FAILED`나 `ABANDONED` 등으로 수동 변경한 후, 동일 `JobParameters`로 다시 실행하면 Spring Batch는 이를 실패한 잡의 재시작으로 간주하여 실행을 허용할 수 있습니다.
  - **방법:**
    - `JobExplorer`를 사용하여 `JobInstance` 및 `JobExecution` 조회.
    - `JobOperator`를 사용하여 `JobExecution`을 `abandon(executionId)` 하거나, DB에서 직접 `BATCH_JOB_EXECUTION` 테이블의 `STATUS`를 변경 (매우 비권장, 위험).
  - **주의:** 이 방법은 Spring Batch 메타데이터를 직접 조작하는 것과 유사하여 매우 신중하게 사용해야 하며, 데이터 정합성에 문제를 일으킬 수 있습니다. 테스트 환경이나 정말 특별한 복구 상황 외에는 권장되지 않습니다.
4. **`SimpleJob`의 `setRestartable(true)` (제한적 효과):**
  - `SimpleJob` 클래스에는 `setRestartable(boolean restartable)` 메서드가 있지만, 이것은 `JobInstanceAlreadyCompleteException`을 직접적으로 우회하는 기능이라기보다는, **실패한 잡(`FAILED` 또는 `STOPPED` 상태)을 재시작할 수 있는지 여부**를 설정하는 데 더 가깝습니다. `COMPLETED` 상태의 잡을 재실행하는 것과는 다른 맥락입니다. (Spring Batch 5에서는 이 속성이 `AbstractJob`으로 올라갔고 기본값은 `true`입니다.)

**결론적으로, "성공적으로 완료된 잡을 동일 파라미터로 다시 실행"하고 싶을 때 가장 일반적이고 안전한 방법은 `JobParametersIncrementer`를 사용하거나 실행 시점에 항상 유니크한 파라미터를 추가하여 새로운 `JobInstance`를 만드는 것입니다.** 이는 Spring Batch가 각 실행을 명확히 구분하고 이력을 관리하는 방식과도 잘 부합합니다.

---
