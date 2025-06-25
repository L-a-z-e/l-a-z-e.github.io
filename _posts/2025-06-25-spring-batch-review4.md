---
title: Spring Batch - Review(4)
description: 
author: laze
date: 2025-06-25 00:00:02 +0900
categories: [Dev, SpringBatch]
tags: [SpringBatch]
---
**1. `JobExecutionListener`**

- **호출 시점:**
  - **`beforeJob(JobExecution jobExecution)`**: `Job`이 실행되기 **직전**에 호출됩니다.
  - **`afterJob(JobExecution jobExecution)`**: `Job`이 성공적으로 완료되거나, 실패하거나, 중단되는 등 **어떤 이유로든 종료된 직후**에 호출됩니다. `jobExecution.getStatus()`를 통해 잡의 최종 상태를 알 수 있습니다.
- **사용 사례**
  - **초기화 작업 (`beforeJob`):**
    - `Job` 실행에 필요한 외부 리소스 준비 (예: 특정 디렉토리 생성, 연결 테스트).
    - `JobExecutionContext`에 초기값 설정.
    - `Job` 실행 시작 로깅.
  - **정리 작업 (`afterJob`):**
    - `Job` 실행 중 사용한 임시 리소스 정리 (예: 임시 파일 삭제).
    - `Job` 실행 결과(성공/실패)에 따른 알림 발송 (이메일, 슬랙 등).
    - `Job` 실행 완료 로깅 및 통계 요약.
    - 실패 시 후속 조치 트리거링.

**2. `StepExecutionListener`**

- **호출 시점:**
  - **`beforeStep(StepExecution stepExecution)`**: 각 `Step`이 실행되기 **직전**에 호출됩니다.
  - **`ExitStatus afterStep(StepExecution stepExecution)`**: 각 `Step`이 성공적으로 완료되거나, 실패하는 등 **종료된 직후**에 호출됩니다.
    - **중요:** 이 메서드는 `ExitStatus`를 반환할 수 있으며, 이 반환된 `ExitStatus`가 해당 스텝의 최종 `ExitStatus`가 되어 잡의 플로우 제어에 영향을 줄 수 있습니다. (만약 `null`을 반환하면, 스텝의 원래 `ExitStatus`가 사용됩니다.)
- **사용 사례**
  - **초기화 작업 (`beforeStep`):**
    - 해당 `Step` 실행에 필요한 리소스 준비.
    - `StepExecutionContext`에 초기값 설정 또는 `JobExecutionContext`에서 값 가져오기.
    - `Step` 실행 시작 로깅.
  - **정리 작업 (`afterStep`):**
    - 해당 `Step` 실행 중 사용한 리소스 정리.
    - `Step` 실행 결과에 따른 로직 수행 (예: 특정 `ExitStatus`로 변경하여 다음 스텝 흐름 제어).
    - `StepExecutionContext`에 최종 결과 저장.
    - `Step` 실행 완료/실패 로깅.

**구현 방법:**

- 해당 인터페이스(`JobExecutionListener`, `StepExecutionListener`)를 구현한 클래스를 만듭니다.
- `Job` 또는 `Step` 설정 시 `StepBuilder` 또는 `JobBuilder`의 `.listener()` 메서드를 사용하여 등록합니다.

```java
// 예시: JobExecutionListener 구현
public class MyJobListener implements JobExecutionListener {
    @Override
    public void beforeJob(JobExecution jobExecution) {
        System.out.println(jobExecution.getJobInstance().getJobName() + " is starting...");
        jobExecution.getExecutionContext().putString("jobSharedData", "Initial value for job");
    }

    @Override
    public void afterJob(JobExecution jobExecution) {
        System.out.println(jobExecution.getJobInstance().getJobName() + " finished with status: " + jobExecution.getStatus());
        System.out.println("Shared data from job: " + jobExecution.getExecutionContext().getString("jobSharedData"));
    }
}

// 예시: StepExecutionListener 구현
public class MyStepListener implements StepExecutionListener {
    @Override
    public void beforeStep(StepExecution stepExecution) {
        System.out.println(stepExecution.getStepName() + " is starting...");
        stepExecution.getExecutionContext().putLong("stepStartTime", System.currentTimeMillis());
    }

    @Override
    public ExitStatus afterStep(StepExecution stepExecution) {
        System.out.println(stepExecution.getStepName() + " finished with status: " + stepExecution.getStatus());
        long startTime = stepExecution.getExecutionContext().getLong("stepStartTime");
        long duration = System.currentTimeMillis() - startTime;
        System.out.println("Step duration: " + duration + "ms");

        // 예: 특정 조건에 따라 ExitStatus 변경
        if (stepExecution.getWriteCount() == 0 && stepExecution.getStatus() == BatchStatus.COMPLETED) {
            return new ExitStatus("COMPLETED_NO_DATA_WRITTEN");
        }
        return null; // 기본 ExitStatus 사용
    }
}

// Job 설정 시 리스너 등록
@Bean
public Job myJobWithListener(JobRepository jobRepository, Step step1, MyJobListener myJobListener) {
    return new JobBuilder("myJobWithListener", jobRepository)
            .listener(myJobListener) // Job 리스너 등록
            .start(step1)
            .build();
}

@Bean
public Step step1WithListener(JobRepository jobRepository, PlatformTransactionManager transactionManager,
                           ItemReader<String> reader, ItemWriter<String> writer, MyStepListener myStepListener) {
    return new StepBuilder("step1WithListener", jobRepository)
            .<String, String>chunk(10, transactionManager)
            .reader(reader)
            .writer(writer)
            .listener(myStepListener) // Step 리스너 등록
            .build();
}
```

**어노테이션 기반 리스너:**

Spring Batch는 더 간편하게 리스너를 등록할 수 있도록 어노테이션도 지원합니다.

- `@BeforeJob`, `@AfterJob` (메서드에 직접 사용, 해당 빈을 `.listener()`로 Job에 등록)
- `@BeforeStep`, `@AfterStep` (메서드에 직접 사용, 해당 빈을 `.listener()`로 Step에 등록)
- `@BeforeChunk`, `@AfterChunk`, `@AfterChunkError` (`ChunkListener` 관련)
- `@OnReadError`, `@OnProcessError`, `@OnWriteError` (아이템 처리 리스너 관련)

```java
// 예시: 어노테이션 기반 Step 리스너
public class MyAnnotationStepListener {

    @BeforeStep
    public void initializeStep(StepExecution stepExecution) {
        System.out.println("Annotation: " + stepExecution.getStepName() + " is starting...");
    }

    @AfterStep
    public ExitStatus finalizeStep(StepExecution stepExecution) {
        System.out.println("Annotation: " + stepExecution.getStepName() + " finished.");
        // ...
        return stepExecution.getExitStatus(); // 또는 새로운 ExitStatus
    }
}

// 스텝 설정 시
// .listener(new MyAnnotationStepListener())

```

---

**`ExitStatus 활용 방법`**

`ExitStatus`는 `Step` 실행 후의 **"결과 상태"**를 나타내는 문자열입니다. 단순히 "성공(COMPLETED)" 또는 "실패(FAILED)"를 넘어, 더 세분화된 상태를 표현하여 `Job`의 실행 흐름을 정교하게 제어하는 데 사용됩니다.

**1. `ExitStatus`의 기본값:**

- `Step`이 예외 없이 정상적으로 완료되면 기본 `ExitStatus`는 **"COMPLETED"** 입니다.
- `Step` 실행 중 처리되지 않은 예외가 발생하여 실패하면 기본 `ExitStatus`는 **"FAILED"** 입니다.

**2. 커스텀 `ExitStatus` 설정 방법:**

`Step`의 최종 `ExitStatus`를 개발자가 직접 제어하고 싶을 때가 있습니다. 이때 주로 다음 두 가지 방법을 사용합니다.

- **`StepExecutionListener`의 `afterStep` 메서드:**
  - `afterStep` 메서드는 `ExitStatus`를 반환할 수 있습니다. 여기서 반환된 `ExitStatus`가 해당 스텝의 최종 `ExitStatus`가 됩니다. (만약 `null`을 반환하면 스텝의 원래 기본 `ExitStatus`가 사용됩니다.)
  - **예시:** 스텝은 성공적으로 완료되었지만(데이터 처리 중 예외 없음), 처리한 데이터가 하나도 없는 경우 "COMPLETED\_NO\_DATA"라는 커스텀 `ExitStatus`를 설정하고 싶을 수 있습니다.

      ```java
      public class MyCustomStepListener implements StepExecutionListener {
          // ... beforeStep 생략 ...
      
          @Override
          public ExitStatus afterStep(StepExecution stepExecution) {
              if (stepExecution.getStatus() == BatchStatus.COMPLETED && stepExecution.getWriteCount() == 0) {
                  System.out.println("Step completed but no data was written.");
                  return new ExitStatus("COMPLETED_NO_DATA"); // 커스텀 ExitStatus 반환
              }
              return null; // 기본 ExitStatus 사용 (이 경우 COMPLETED)
          }
      }
      ```

- **`JobExecutionDecider` (Decider):**
  - `Decider`는 스텝 자체가 아니라, 스텝 실행 *후에* 호출되어 다음 플로우 상태를 결정합니다. 이때 `Decider`가 반환하는 `FlowExecutionStatus`의 문자열이 `JobBuilder`의 `.on()` 조건에 사용되는 `ExitStatus`처럼 활용됩니다.
  - `Decider`는 `StepExecution` 객체를 참조하여 이전 스텝의 `ExitStatus`뿐만 아니라 `ExecutionContext`, 읽은/쓴 아이템 수 등 더 다양한 정보를 바탕으로 분기 로직을 만들 수 있습니다.

**3. `JobBuilder`에서 `ExitStatus` 활용:**

이렇게 설정된 (또는 기본) `ExitStatus`는 `JobBuilder`의 플로우 제어 구문에서 사용됩니다.

```java
// ... JobBuilder 설정 ...
.start(stepA)
    .on("COMPLETED").to(stepB)          // stepA의 ExitStatus가 "COMPLETED"이면 stepB로
    .on("FAILED").to(errorHandlingStep) // stepA가 "FAILED"이면 errorHandlingStep으로
    .on("COMPLETED_NO_DATA").to(notificationStep) // stepA가 "COMPLETED_NO_DATA"이면 notificationStep으로
.from(stepA) // stepA로부터의 다른 모든 경우 (위 조건에 해당 안 될 시)
    .on("*").fail()                     // 그 외의 경우는 Job 실패 처리
// ...
```

**제대로 활용하는 방법 요약:**

1. **세분화된 상태 정의:** 단순 성공/실패 외에, 비즈니스적으로 의미 있는 다양한 결과 상태를 `ExitStatus` 문자열로 미리 정의합니다. (예: "VALIDATION\_ERROR", "PARTIAL\_SUCCESS", "REQUIRES\_MANUAL\_CHECK")
2. **적절한 시점에 `ExitStatus` 설정:**
  - 스텝 자체의 최종 결과로 `ExitStatus`를 바꾸고 싶다면 `StepExecutionListener`의 `afterStep`을 활용합니다.
  - 스텝 실행 후 더 복잡한 조건으로 다음 흐름을 결정하고 싶다면 `JobExecutionDecider`를 활용합니다.
3. **`JobBuilder`에서 명확한 플로우 정의:** 정의된 `ExitStatus` 패턴에 따라 각기 다른 다음 스텝이나 플로우 종료자(`.end()`, `.fail()`)로 연결되도록 `Job`의 흐름을 명확하게 구성합니다. `on("*")`을 사용하여 예상치 못한 `ExitStatus`에 대한 기본 처리 경로를 설정하는 것도 좋은 방법입니다.

---

**질문 2: 여러 스텝으로 가공한 데이터를 어떻게 다음 스텝에 넘겨주나? `JobExecution`이나 `StepExecution`에 다 들어있나?**

이것은 Spring Batch를 사용할 때 정말 중요한 이해 포인트입니다!

**결론부터 말씀드리면, Spring Batch의 기본 청크 지향 모델에서는 한 스텝에서 처리(가공)한 "아이템 데이터 자체"를 직접 다음 스텝으로 메모리를 통해 넘겨주는 메커니즘은 권장되지 않으며, 기본적으로 제공하지도 않습니다.**

대신, 다음과 같은 방식으로 스텝 간 "데이터 흐름" 또는 "상태/정보 전달"이 이루어집니다.

**1. 데이터베이스 또는 파일 시스템을 통한 중간 저장 (가장 일반적인 방식):**

- **개념:** 첫 번째 스텝(`step1`)이 데이터를 읽고 가공한 후, 그 결과를 **데이터베이스의 특정 테이블이나 임시 파일에 저장**합니다 (`ItemWriter`).
- 두 번째 스텝(`step2`)은 `step1`이 저장한 그 테이블이나 파일을 **입력으로 읽어서 (`ItemReader`)** 다음 처리를 진행합니다.
- **흐름:**
  - `Step1`: `Reader1` (원본 데이터 소스) -> `Processor1` (1차 가공) -> `Writer1` (중간 DB 테이블 또는 파일에 저장)
  - `Step2`: `Reader2` (Step1이 저장한 중간 DB/파일 읽기) -> `Processor2` (2차 가공) -> `Writer2` (최종 목적지에 저장)
- **장점:**
  - **대용량 데이터 처리 적합:** 각 스텝은 필요한 데이터만 스트리밍 방식으로 처리하므로 메모리 부담이 적습니다.
  - **재시작 용이성:** 각 스텝이 독립적으로 상태를 관리하고, 중간 결과가 영속화되어 있으므로 특정 스텝부터 재시작하기 용이합니다.
  - **명확한 책임 분리:** 각 스텝의 역할이 명확해집니다.
- **단점:**
  - 중간 저장소를 위한 I/O 오버헤드가 발생할 수 있습니다.
  - 중간 저장소 스키마 관리가 필요할 수 있습니다.

**2. `ExecutionContext`를 통한 "소량의" 상태 정보 또는 파라미터 전달:**

- **개념:** `StepExecutionListener`의 `afterStep`에서 현재 스텝의 중요한 결과 "요약 정보"나 다음 스텝에 필요한 "소량의 파라미터"를 `JobExecutionContext` (또는 다음 스텝의 `StepExecutionContext` - 이는 `beforeStep`에서 가져와야 함)에 저장합니다.
- 다음 스텝의 `beforeStep`이나 `ItemReader` (또는 다른 컴포넌트)의 `open` 메서드, 또는 `@StepScope` 빈에서 SpEL을 통해 이 값을 읽어와 활용합니다.
- **주의:** `ExecutionContext`는 **대용량의 아이템 데이터 자체를 전달하는 용도가 아닙니다.** 직렬화/역직렬화 오버헤드가 크고, `JobRepository` (DB)에 부담을 줄 수 있습니다. 주로 다음과 같은 정보를 전달합니다.
  - 처리된 파일 이름, 생성된 레코드 수, 특정 집계 값 등
  - 다음 스텝의 쿼리 조건으로 사용될 ID 목록 (매우 작은 경우)
  - 다음 스텝의 동작을 제어할 플래그 값
- **`JobExecution`이나 `StepExecution`에 내가 읽고 처리한게 다 들어있어서 이것만 넘겨주면 사용이 가능한거야 뭐야?**
  - `JobExecution`이나 `StepExecution` 객체 자체에는 처리된 **아이템 데이터 목록이 들어있지 않습니다.** 이 객체들은 주로 실행 상태(COMPLETED, FAILED), 시작/종료 시간, 읽은/쓴 아이템의 **개수(count)**, 커밋/롤백 횟수 등의 **메타데이터**와 `ExecutionContext`를 담고 있습니다.
  - 따라서 이 객체들을 다음 스텝에 "넘겨준다"는 개념보다는, `ExecutionContext`라는 공유 공간을 통해 "정보를 남기고 다음 스텝이 참조한다"는 개념에 가깝습니다.

**3. Spring Batch Integration 또는 인메모리 큐/메시징 (고급/특수 사례):**

- 매우 특별한 경우, 스텝 간에 데이터를 메모리상에서 직접 주고받고 싶다면, `BlockingQueue` 같은 것을 공유 빈으로 만들고 한 스텝에서는 쓰고 다른 스텝에서는 읽는 식으로 직접 구현하거나, Spring Integration과 연동하여 메시지 채널을 통해 데이터를 전달하는 등의 고급 기법을 사용할 수 있습니다.
- 하지만 이는 Spring Batch의 기본적인 청크 처리 모델을 벗어나는 것이며, 트랜잭션 관리나 재시작 로직을 매우 신중하게 직접 설계해야 하는 복잡성이 따릅니다. 대부분의 경우 1번 방식(중간 저장소 사용)이 권장됩니다.

**결론적으로, "read 와 process를 여러차례 거쳐야하는 복잡한 비즈니스 로직이 들어간 Job의 경우"에는 다음과 같은 접근을 고려합니다.**

- **각 "주요 가공 단계"를 별도의 `Step`으로 분리합니다.**
- **한 `Step`의 결과물(가공된 데이터)은 다음 `Step`의 입력이 되도록, 중간 저장소(DB 테이블, 파일)를 활용합니다.**
- 만약 스텝 간에 전달해야 할 정보가 데이터 자체가 아니라 **소량의 상태 정보나 제어 파라미터**라면, `ExecutionContext`를 활용합니다.

**예시 시나리오:**

"고객 주문 파일(CSV)을 읽어 -> 주문 유효성 검사 및 고객 등급 적용 -> 등급 적용된 주문을 DB에 저장 -> 저장된 주문 중 VIP 고객 주문만 추출하여 -> 특별 할인 계산 -> 할인 적용된 VIP 주문 내역을 별도 리포트 파일(TXT)로 생성"

1. **Step 1: 주문 파일 읽기 및 기본 가공/저장**
  - `Reader`: `FlatFileItemReader` (CSV 주문 파일 읽기)
  - `Processor`: 주문 유효성 검사, 고객 정보 조회 후 등급 적용 (결과는 `OrderWithGrade` 객체)
  - `Writer`: `JdbcBatchItemWriter` (`OrderWithGrade` 객체를 "STAGING\_ORDERS" 테이블에 저장)
  - `StepExecutionListener`: `afterStep`에서 처리된 주문 건수를 `JobExecutionContext`에 "processedOrderCount"로 저장.
2. **Step 2: VIP 주문 추출 및 할인 적용/리포트 생성**
  - `Reader`: `JdbcCursorItemReader` ("STAGING\_ORDERS" 테이블에서 VIP 등급 주문만 읽기. 이때 `JobExecutionContext`의 "processedOrderCount"를 참조하여, 만약 0건이면 이 스텝을 스킵하도록 `Decider`를 사용할 수도 있음)
  - `Processor`: VIP 주문에 대해 특별 할인 계산 (결과는 `VipOrderWithDiscount` 객체)
  - `Writer`: `FlatFileItemWriter` (`VipOrderWithDiscount` 객체를 TXT 리포트 파일로 쓰기)

이런 식으로 각 스텝은 자신의 입력 소스에서 데이터를 읽고, 처리하고, 자신의 출력 목적지에 쓰는 명확한 책임을 가집니다. 스텝 간 데이터는 DB나 파일을 통해 "간접적으로" 전달됩니다.

---

**문제인 상황 분석:**

1. **1단계:** "대상 여부" 정보 읽기 (예: `SourceA`에서 `대상고객ID` 목록)
2. **2단계:** 1단계에서 얻은 `대상고객ID`를 바탕으로 "현금흐름" 정보 가져오기 (예: `SourceB`에서 각 `대상고객ID`별 `현금흐름 데이터`)
3. **3단계:** 1단계의 "대상 정보"와 2단계의 "현금흐름 정보"를 조합하고, 추가적인 비즈니스 로직을 적용하여 최종 "상각스케줄" 계산.
4. **4단계:** 계산된 "상각스케줄"을 특정 테이블에 저장.

---

**해결 방안 제안:**

**방안 1: 단일 스텝, 복합적인 `ItemProcessor` 활용 (가장 일반적인 접근)**

이 방식은 "최초 대상이 되는 정보"를 `ItemReader`로 읽고, 나머지 데이터 조회 및 가공 로직을 `ItemProcessor` 내부에 집중시키는 방식입니다.

- **`ItemReader<대상고객ID>`:**
  - 1단계의 "대상 여부" 정보를 읽어옵니다. 예를 들어, `JdbcCursorItemReader`를 사용하여 `대상고객ID` 목록을 하나씩 읽어옵니다.
  - `read()` 메서드는 `대상고객ID` (또는 해당 ID를 포함한 간단한 객체)를 반환합니다.
- **`ItemProcessor<대상고객ID, 상각스케줄결과>`:**
  - `process(대상고객ID item)` 메서드 내에서 다음 작업들을 순차적으로 수행합니다.
    1. **2단계 로직 (현금흐름 정보 가져오기):** 전달받은 `대상고객ID`를 사용하여 `SourceB` (예: 다른 DB 테이블, 외부 API 등)에서 해당 고객의 "현금흐름 정보"를 조회합니다.
      - 이때 `ItemProcessor` 내부에 `JdbcTemplate`, `RestTemplate`, 또는 다른 서비스/DAO를 주입받아 사용할 수 있습니다.
    2. **3단계 로직 (데이터 조합 및 비즈니스 로직 적용):**
      - 1단계에서 넘어온 `대상고객ID` (및 관련 정보)와 2단계에서 조회한 "현금흐름 정보"를 조합합니다.
      - 필요한 비즈니스 로직을 적용하여 최종 "상각스케줄" 값을 계산합니다.
      - 이 결과로 `상각스케줄결과` 객체를 생성합니다.
    3. 생성된 `상각스케줄결과` 객체를 반환합니다. 만약 특정 `대상고객ID`에 대해 상각스케줄을 만들 수 없다면 `null`을 반환하여 필터링할 수도 있습니다.
- **`ItemWriter<상각스케줄결과>`:**
  - `ItemProcessor`가 반환한 `상각스케줄결과` 객체들을 `Chunk` 단위로 받아, 4단계의 특정 테이블에 저장합니다. (예: `JdbcBatchItemWriter` 사용)

**장점:**

- 하나의 스텝 내에서 모든 로직이 처리되므로 흐름이 비교적 단순해 보일 수 있습니다.
- 청크 기반 처리와 트랜잭션 관리가 `ItemReader`가 읽어온 `대상고객ID` 단위를 기준으로 자연스럽게 이루어집니다.
- 중간 저장소를 사용하지 않아도 됩니다 (단, `ItemProcessor` 내부에서 조회하는 데이터는 메모리에 로드될 수 있음).

**단점:**

- **`ItemProcessor`가 매우 비대해지고 복잡해질 수 있습니다.** 여러 데이터 소스에 대한 접근 로직과 비즈니스 로직이 한 곳에 집중됩니다.
- **성능 문제:** 각 `대상고객ID` 아이템마다 `ItemProcessor` 내에서 추가적인 DB 조회(2단계 현금흐름 정보)가 발생하므로, N+1 쿼리 문제와 유사한 성능 저하가 발생할 수 있습니다. (대상 고객이 100만 명이면, 현금흐름 조회 쿼리도 100만 번 실행될 수 있음)
- **재시작 단위:** `ItemProcessor` 내부의 여러 단계 중 어디서 실패하든, 해당 `대상고객ID` 아이템 처리부터 재시작됩니다. 2단계 현금흐름 조회까지는 성공했는데 3단계 로직에서 실패했다면, 재시작 시 현금흐름 조회를 다시 해야 할 수 있습니다.

**성능 개선 방안 (방안 1의 경우):**

- `ItemProcessor` 내에서 현금흐름 정보를 조회할 때, 여러 고객의 현금흐름 정보를 한 번의 쿼리로 가져와 메모리에 캐싱해두고 사용하는 방식을 고려할 수 있습니다 (단, 캐시 크기와 데이터 변경 빈도 고려).
- 또는, `ItemReader`에서 "대상고객 정보 + 해당 고객의 현금흐름 정보"를 JOIN 등을 통해 한 번에 읽어오는 방식으로 SQL을 최적화할 수 있다면 가장 좋습니다. (하지만 데이터 소스가 다르다면 불가능)

---

**방안 2: 여러 스텝 사용 + `ExecutionContext`를 통한 "제어 정보" 전달 (데이터 자체 전달 아님)**

이 방식은 각 주요 단계를 별도의 스텝으로 나누되, 스텝 간 직접적인 대용량 데이터 전달은 피하고, 필요한 "제어 정보"나 "소량의 키 값" 정도만 `ExecutionContext`를 통해 전달하는 방식입니다. 하지만 학생분의 시나리오에서는 "1단계 결과로 2단계 입력이 결정"되므로, 이 방식만으로는 완전한 해결이 어려울 수 있고, 결국 중간 저장소가 필요하게 될 가능성이 높습니다.

만약 "대상 고객 ID 목록" 자체가 매우 작고, 다음 스텝에서 이 ID 목록을 기반으로 새로운 `ItemReader`를 동적으로 구성할 수 있다면 시도해볼 수 있습니다.

- **Step 1: 대상 고객 ID 추출 및 `ExecutionContext`에 저장**
  - `ItemReader`: `대상고객ID` 읽기
  - `ItemProcessor`: (선택적)
  - `ItemWriter`: 읽어온 `대상고객ID`들을 `List`로 모아서, 스텝 종료 시 `StepExecutionListener`의 `afterStep`에서 `JobExecutionContext`에 이 `List<Long> targetCustomerIds`를 저장. (**주의: ID 목록이 매우 커지면 이 방식은 부적합!**)
- **Step 2: 현금흐름 정보 가져오기 및 상각스케줄 계산/저장**
  - `ItemReader`:
    - `@StepScope`로 선언.
    - `open()` 메서드 또는 생성자에서 `JobExecutionContext`로부터 `targetCustomerIds` 목록을 가져옵니다.
    - 이 ID 목록을 사용하여 `SourceB`에서 현금흐름 정보를 **하나의 `대상고객ID` (또는 그에 매칭되는 현금흐름 정보) 단위로** 읽어오는 `ItemReader`를 구현합니다. (예: `JdbcCursorItemReader`의 SQL을 동적으로 구성하거나, `ListItemReader`에 조회 결과를 채워 넣는 방식)
    - **이 부분이 까다로울 수 있습니다.** `ItemReader`는 스트리밍 방식으로 아이템을 하나씩 반환해야 하는데, 이전 스텝의 결과(ID 목록 전체)를 한 번에 알아야 다음 처리가 가능한 구조이기 때문입니다.
  - `ItemProcessor`: (1단계에서 대상 고객 정보도 `ExecutionContext`나 다른 방식으로 가져올 수 있다면) 대상 정보 + 현금흐름 정보 조합하여 상각스케줄 계산.
  - `ItemWriter`: 계산된 상각스케줄 저장.

**장점 (이론상):**

- 스텝별 책임 분리.

**단점:**

- **`ExecutionContext`를 통한 대량 데이터 전달은 부적합합니다.** `targetCustomerIds` 목록이 수십만 건 이상이면 `ExecutionContext`에 저장하고 읽는 것이 성능 문제 및 직렬화 문제를 일으킬 수 있습니다.
- **`Step 2`의 `ItemReader` 구현이 복잡해집니다.** 이전 스텝의 결과 전체를 알아야 다음 아이템을 읽을 수 있는 구조는 일반적인 `ItemReader`의 스트리밍 방식과 잘 맞지 않습니다. 결국 `Step 2` 시작 시점에 모든 현금흐름 정보를 한 번에 로드하거나, `Step 1`에서 중간 파일/테이블에 ID를 쓰고 `Step 2`에서 그 파일/테이블을 읽는 것이 더 현실적일 수 있습니다.

---

**방안 3: `Job` 파라미터를 활용한 동적 `ItemReader` 구성 (제한적)**

만약 "대상 고객 ID"가 `JobParameter`로 하나씩 전달될 수 있는 시나리오라면 (예: 각 고객별로 잡을 실행), `ItemReader`에서 `@Value("#{jobParameters['customerId']}")` 와 같이 고객 ID를 받아 해당 고객의 정보와 현금흐름 정보를 조합하여 읽는 것도 가능합니다. 하지만 이는 "일괄 처리"의 개념과는 거리가 멀어집니다.

---

중간 저장소를 사용하지 않는다는 제약 하에서는 **방안 1 (단일 스텝, 복합 `ItemProcessor`)** 이 가장 직접적인 방법

- **`ItemReader`**: 가장 기본적인 "단위"가 되는 정보(여기서는 `대상고객ID` 또는 대상고객 기본정보)를 하나씩 읽어옵니다.
- **`ItemProcessor`**:
  1. Reader로부터 받은 `대상고객ID`를 키로 사용하여, 필요한 추가 정보(현금흐름 정보)를 **내부에서 직접 조회**합니다. (이때 `Processor`에 `JdbcTemplate`이나 관련 서비스 주입)
  2. 조회된 정보와 기존 정보를 조합합니다.
  3. 또 다른 정보가 필요하면, **다시 내부에서 조회**합니다.
  4. 모든 정보가 모이면 비즈니스 로직을 수행하여 최종 결과를 만듭니다.
  5. 이 최종 결과를 `ItemWriter`로 넘깁니다.

**이 방식의 핵심은 `ItemProcessor`의 역할이 매우 커진다는 것입니다.** 마치 작은 서비스 레이어처럼 동작하게 됩니다.

**성능 고려사항 (방안 1 적용 시 반드시 고려):**

- **N+1 문제 최소화:** `ItemProcessor` 내부에서 각 아이템마다 DB 조회를 반복하면 성능이 크게 저하됩니다.
  - **일괄 조회 후 캐싱:** `beforeStep`에서 다음 청크 또는 일정량의 `대상고객ID`에 대한 현금흐름 정보를 미리 한 번의 쿼리로 가져와 `Map` 등에 캐싱해두고, `ItemProcessor`에서는 이 캐시를 참조하는 방법을 고려할 수 있습니다. (메모리 사용량 주의)
  - **데이터베이스 함수/프로시저 활용:** 복잡한 데이터 조합 및 계산 로직을 DB 함수나 프로시저로 만들고, `ItemProcessor`에서는 이를 호출하는 방식으로 DB의 처리 능력을 활용할 수도 있습니다.
  - **JOIN 최적화 (가능하다면):** 만약 대상 정보와 현금흐름 정보가 동일 DB에 있고 JOIN이 가능하다면, `ItemReader` 단계에서 최대한 JOIN을 통해 필요한 데이터를 한 번에 읽어오는 것이 가장 좋습니다. (학생분 상황에서는 데이터 소스가 다를 수 있어 어려울 수 있음)

**만약 중간 저장소를 "정말 정말" 사용할 수 없는 상황이 아니라면,** 복잡도가 일정 수준 이상으로 높아질 경우, 각 주요 데이터 변환 단계의 결과를 임시 테이블이나 파일에 저장하고 다음 스텝에서 읽어 처리하는 것이 장기적으로는 더 관리하기 쉽고, 재시작 및 성능 튜닝에도 유리할 수 있습니다.

---

Cursor 방식 문제점 해결 방안 (그룹화된 데이터가 Process 과정에서 필요할 때)

이것은 커서 방식의 `ItemReader`를 사용할 때 **"하나의 논리적인 아이템을 구성하기 위해 여러 개의 물리적인 DB 행(row)이 필요한 경우"** 에 발생하는 대표적인 문제입니다.

"24회차 상각스케줄"이라는 **하나의 최종 아이템**을 만들기 위해서는 0회차부터 24회차까지의 모든 현금흐름 데이터가 필요한데,

`JdbcCursorItemReader`는 기본적으로 DB 행 하나하나를 아이템으로 간주하고 스트리밍 방식으로 넘겨주기 때문에 발생하는 불일치입니다.

이러한 위험에 대응하기 위한 몇 가지 전략이 있습니다.

**전략 1: `ItemReader` 수준에서 "그룹화된 아이템" 읽기 (커스텀 `ItemReader` 또는 고급 설정)**

가장 이상적인 방법은 `ItemReader` 자체가 "0~24회차 현금흐름 데이터 묶음"을 하나의 아이템으로 인식하고 반환하도록 하는 것입니다.

- **커스텀 `ItemReader` 구현:**
  - `JdbcCursorItemReader`를 직접 확장하거나, `AbstractItemCountingItemStreamItemReader` 등을 기반으로 새로운 `ItemReader`를 만듭니다.
  - `read()` 메서드 내부에서, 특정 고객(또는 기준)의 현금흐름 데이터를 0회차부터 24회차까지 (또는 해당 그룹의 끝까지) **모두 읽어서 하나의 리스트나 복합 객체로 만든 후, 이 묶음을 하나의 아이템으로 반환**합니다.
  - 다음 `read()` 호출 시에는 다음 고객(또는 다음 그룹)의 데이터를 같은 방식으로 묶어서 반환합니다.
  - 이를 위해서는 현재 어떤 고객/그룹을 처리 중인지, 그리고 해당 그룹의 데이터를 어디까지 읽었는지 내부적으로 상태 관리가 필요합니다. `ItemStream` 인터페이스를 구현하여 재시작 시 이 상태를 복원할 수 있도록 해야 합니다.
  - **예시 (의사 코드):**

      ```java
      public class GroupedCashflowReader extends AbstractItemCountingItemStreamItemReader<List<Cashflow>> {
          private JdbcTemplate jdbcTemplate;
          private String currentCustomerId; // 현재 처리 중인 고객 ID
          private List<String> allCustomerIds; // 처리해야 할 전체 고객 ID 리스트 (open 시 로드)
          private int customerIndex = 0;
      
          @Override
          protected void doOpen() throws Exception {
              // 전체 대상 고객 ID 목록을 가져옴 (예시)
              this.allCustomerIds = jdbcTemplate.queryForList("SELECT DISTINCT customer_id FROM CASHFLOW_SOURCE ORDER BY customer_id", String.class);
              // 재시작 로직: executionContext에서 customerIndex 복원
          }
      
          @Override
          protected List<Cashflow> doRead() throws Exception {
              if (customerIndex >= allCustomerIds.size()) {
                  return null; // 모든 고객 처리 완료
              }
              this.currentCustomerId = allCustomerIds.get(customerIndex++);
              // 현재 customerId에 해당하는 모든 회차(0~24)의 현금흐름 데이터를 한 번에 조회
              List<Cashflow> cashflowsForCustomer = jdbcTemplate.query(
                  "SELECT * FROM CASHFLOW_SOURCE WHERE customer_id = ? ORDER BY 회차",
                  new CashflowRowMapper(),
                  this.currentCustomerId
              );
              if (cashflowsForCustomer.isEmpty()) { // 해당 고객의 데이터가 없으면 다음 고객으로 (또는 null 처리)
                  return doRead(); // 재귀 호출로 다음 고객 데이터 읽기 시도
              }
              // 여기서 24개 데이터가 다 있는지, 아니면 특정 개수만 있는지 등 비즈니스 규칙에 따라 추가 검증 가능
              return cashflowsForCustomer; // 한 고객의 전체 현금흐름 리스트를 하나의 아이템으로 반환
          }
      
          @Override
          protected void doClose() throws Exception { /* 리소스 정리 */ }
      
          // ItemStream의 update 메서드에서 customerIndex 등을 ExecutionContext에 저장
      }
      ```

- **`Driving Query ItemReader` 패턴 (Spring Batch Extensions 등에 유사한 개념 존재):**
  - 첫 번째 `ItemReader` (드라이빙 쿼리 리더)는 그룹의 키(예: `고객ID`)만 읽어옵니다.
  - 각 키가 읽힐 때마다, 해당 키를 파라미터로 사용하여 그룹 전체의 상세 데이터를 조회하는 두 번째 `ItemReader`나 서비스 로직을 실행합니다. (이 부분은 `ItemProcessor`나 `CompositeItemStreamReader` 같은 형태로 구현될 수 있습니다.)

**전략 2: `ItemProcessor`에서 데이터 축적 및 그룹 완성 시 반환 (상태 저장 프로세서)**

이 방식은 `ItemReader`는 여전히 개별 현금흐름 행을 하나씩 읽어오되, `ItemProcessor`가 특정 고객의 모든 회차 데이터가 모일 때까지 상태를 유지하며 데이터를 축적하고, 그룹이 완성되면 그 묶음을 다음 단계로 넘기는 방식입니다.

- **`ItemReader<Cashflow>`:** 개별 `Cashflow` 객체 (하나의 행, 특정 회차 데이터)를 순서대로 읽어옵니다. (예: 고객ID, 회차, 금액)
- **`StatefulItemProcessor<Cashflow, List<Cashflow>>` (커스텀 구현):**
  - 내부적으로 이전 아이템의 고객ID와 현재 아이템의 고객ID를 비교합니다.
  - **고객ID가 동일하면:** 현재 `Cashflow` 데이터를 내부 리스트(버퍼)에 계속 추가합니다. 그리고 `null`을 반환하여 `ItemWriter`로 아직 넘어가지 않도록 합니다 (필터링 효과).
  - **고객ID가 변경되면 (또는 특정 회차(예: 24회차)에 도달하면):** 이전 고객의 모든 현금흐름 데이터가 담긴 내부 리스트를 반환합니다. 그리고 새로운 고객의 첫 번째 `Cashflow` 데이터를 새로운 내부 리스트에 추가하기 시작합니다.
  - **마지막 아이템 처리:** `ItemReader`가 `null`을 반환했을 때, `ItemProcessor`는 마지막으로 축적하고 있던 고객의 데이터 묶음을 처리해야 합니다. 이를 위해서는 `ItemStream`을 구현하여 `read()`가 끝난 후 (또는 `ChunkListener`의 `afterChunk` 등) 특별한 로직이 필요할 수 있습니다. 또는, `ItemProcessor` 자체를 `ItemStream`으로 만들고 `flush()`와 유사한 메서드를 두어 `ItemWriter` 호출 전에 명시적으로 마지막 그룹을 반환하도록 할 수도 있습니다. (이 부분은 설계가 복잡해질 수 있습니다.)
- **`ItemWriter<List<Cashflow>>`:** `ItemProcessor`가 반환한 `List<Cashflow>` (한 고객의 전체 현금흐름 묶음)를 받아 상각스케줄 계산 및 저장을 수행합니다.

**장점 (전략 2):**

- `ItemReader`는 단순하게 유지될 수 있습니다.

**단점 (전략 2):**

- `ItemProcessor`가 상태를 가져야 하므로 스레드 안전성에 주의해야 합니다 (일반적으로 `@StepScope`로 해결).
- 마지막 그룹 처리가 까다로울 수 있습니다.
- 청크 경계와 그룹 경계가 일치하지 않으면 로직이 복잡해질 수 있습니다 (예: 한 고객의 데이터가 두 청크에 걸쳐 처리되는 경우).

**전략 3: 스텝 분리 및 중간 저장소 활용 (학생분이 피하고 싶어하는 방식이지만, 때로는 가장 명확)**

이전에 논의했던 것처럼, 각 고객별로 모든 회차의 현금흐름 데이터를 임시 테이블이나 파일에 먼저 모두 저장한 후, 다음 스텝에서 이 그룹화된 데이터를 읽어 처리하는 방식입니다. 중간 저장소를 사용할 수 없다는 제약이 없다면, 복잡한 그룹핑 로직에는 오히려 더 안정적이고 관리하기 쉬울 수 있습니다.

---

이 "중간에 잘리는" 위험은 **`ItemReader`가 반환하는 아이템의 단위**와 **`ItemProcessor` 또는 `ItemWriter`가 필요로 하는 데이터의 논리적 단위**가 일치하지 않을 때 발생합니다.

- **위험 발생 시나리오 (단순 `JdbcCursorItemReader` 사용 시):**
  1. `ItemReader`는 `Cashflow` 행을 하나씩 (예: 3회차 데이터) `ItemProcessor`에게 넘깁니다.
  2. `ItemProcessor`는 이 3회차 데이터만으로는 0~24회차 전체를 사용하는 계산을 할 수 없습니다.
  3. 만약 잡이 이 상태에서 실패하고 재시작하면, `ItemReader`는 (설정에 따라) 4회차 데이터부터 다시 읽어올 수 있습니다. 여전히 0~2회차 데이터는 누락된 상태로 다음 처리가 진행될 수 있습니다.
- **대응 방안:**
  - **가장 좋은 대응은 전략 1 (커스텀 `ItemReader` 또는 고급 설정)을 사용하는 것입니다.** `ItemReader` 자체가 "0~24회차 현금흐름 데이터 묶음"을 하나의 아이템으로 만들어서 `ItemProcessor`에게 전달하도록 설계하면, `ItemProcessor`는 항상 완전한 데이터 세트를 받게 됩니다.
    - 이 경우, `ItemReader`의 `read()`가 성공적으로 `List<Cashflow>` (0~24회차)를 반환했다면, 그 아이템은 완전한 것입니다.
    - 만약 `read()` 과정에서 (예: 0~24회차를 DB에서 읽어오는 도중) 오류가 발생하여 `read()`가 예외를 던지거나 `null`을 반환하면, 해당 "묶음 아이템"은 처리되지 않거나, 재시작 시 해당 "묶음 아이템"을 처음부터 다시 읽으려고 시도합니다. (커스텀 `ItemReader`의 상태 저장 로직에 따라)
  - **전략 2 (상태 저장 프로세서)를 사용한다면,** `ItemProcessor`가 고객 ID가 바뀔 때까지 데이터를 모으므로, `ItemProcessor`가 `List<Cashflow>`를 반환하는 시점에는 해당 고객의 (이론상) 모든 회차 데이터가 모인 상태입니다. 만약 `ItemReader`가 중간(예: 3회차)까지 읽고 잡이 실패했다면, 재시작 시 `ItemReader`는 4회차부터 읽겠지만, `ItemProcessor`는 이전 고객의 데이터를 아직 반환하지 않은 상태이므로, 새로운 고객 데이터가 들어올 때까지 기다리거나, 재시작 시 `ExecutionContext`를 통해 이전 고객의 데이터를 복원하는 로직이 필요할 수 있습니다 (매우 복잡해짐).

**결론적으로, "하나의 논리적 아이템을 만들기 위해 여러 DB 행이 필요한 경우"에는:**

1. **`ItemReader`가 그 여러 DB 행을 하나의 묶음(예: `List`나 복합 객체)으로 만들어서 반환하도록 커스터마이징하는 것이 가장 좋습니다.** 이렇게 하면 `ItemProcessor`는 항상 완전한 데이터를 받게 되고, Spring Batch의 청크 처리 및 재시작 메커니즘이 이 "묶음 아이템" 단위를 기준으로 동작하게 됩니다.
2. 이것이 어렵다면, `ItemProcessor`에서 상태를 관리하며 데이터를 축적하는 방식을 고려할 수 있지만, 구현 복잡도와 재시작 시 상태 관리가 어려워질 수 있습니다.

---
