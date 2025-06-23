---
title: Spring Batch - Review(2)
description: 
author: laze
date: 2025-06-23 00:00:02 +0900
categories: [Dev, SpringBatch]
tags: [SpringBatch]
---
**1. `JobParameters`의 데이터 타입**

- **`String`, `Long`, `Double`, `Date` (java.util.Date) 타입을 값으로 가질 수 있습니다.** (Spring Batch 5부터는 `java.time.LocalDate`, `java.time.LocalDateTime` 등 Java 8의 날짜/시간 타입도 지원합니다.)
- 모든 파라미터는 Key(문자열)-Value 쌍으로 저장되며, 각 Value는 위에 언급된 특정 타입 중 하나여야 합니다.
- 내부적으로는 `JobParameter`라는 클래스가 실제 값을 감싸고 있으며, 이 `JobParameter` 객체들의 맵이 `JobParameters` 객체입니다.
- 값을 설정할 때는 예를 들어 `new JobParametersBuilder().addString("fileName", "input.csv").addLong("runId", 123L).toJobParameters()` 와 같이 빌더를 사용하며 각 타입에 맞는 `addXXX` 메서드를 사용합니다.

**2. `JobParameters`와 `JobInstance`의 관계**

- **`JobInstance` = `Job` 이름 + 고유 식별 `JobParameters`**
- 즉, 동일한 `Job` 이름으로 여러 번 실행하더라도, `JobParameters`의 내용이 (하나라도) 다르면 Spring Batch는 이를 완전히 **별개의 `JobInstance`*로 간주합니다.
- 이는 특정 파라미터 조합에 대한 잡 실행을 고유하게 추적하고, 재시작 정책 등을 적용하는 기준이 됩니다.

**3. `JobParameters`의 활용 용도**

- **주요 활용 용도:**
  1. **잡 실행의 고유성 식별:**
    - 매일 실행되는 배치 잡의 경우, 실행 날짜(`processingDate=2023-01-15`)를 파라미터로 전달하여 각 날짜별 실행을 별개의 `JobInstance`로 관리할 수 있습니다. 이를 통해 특정 날짜의 잡만 재실행하거나 실행 이력을 조회하는 것이 용이해집니다.
  2. **동적인 입력 값 제공:**
    - 처리할 입력 파일의 경로 (`inputFile=data/input_20230115.csv`)
    - 데이터베이스 쿼리의 조건 값 (`startDate=2023-01-01`, `endDate=2023-01-31`)
    - 특정 처리 모드 플래그 (`mode=FULL`, `mode=INCREMENTAL`)
    - 위와 같이 잡 실행 시점에 외부에서 동적으로 결정되는 값을 `Job` 내부로 전달하여, 동일한 `Job` 로직을 다양한 입력에 대해 재사용할 수 있도록 합니다.
  3. **재실행 방지 또는 허용 제어:**
    - 기본적으로 동일 `JobParameters`로 성공한 `JobInstance`는 재실행되지 않습니다.
    - 만약 재실행을 원한다면, `JobParametersIncrementer`를 사용하여 매번 `JobParameters`에 약간의 변화(예: 실행 카운터, 타임스탬프)를 주어 새로운 `JobInstance`를 생성하도록 유도할 수 있습니다.
  4. **실행 컨텍스트 초기화:**
    - `JobParameters`의 값은 `@StepScope`나 `@JobScope` 빈에서 SpEL (`#{jobParameters['keyName']}`)을 통해 직접 주입받아 사용할 수 있으며, 이는 스텝이나 잡의 초기 설정값으로 활용될 수 있습니다.

**`JobParameters` 사용 시 주의점:**

- **읽기 전용 (Read-only):** `JobParameters`는 한번 생성되어 `Job` 실행이 시작되면 변경할 수 없는 읽기 전용 값입니다.
- **식별 파라미터 vs 비식별 파라미터:**
  - `JobParametersBuilder` 사용 시, 각 파라미터를 추가할 때 "식별 파라미터"인지 여부를 지정할 수 있습니다 (`addString(key, value, isIdentifying)`의 세 번째 인자, 기본값은 `true`).
  - **식별 파라미터 (`isIdentifying=true`)**: `JobInstance`를 고유하게 구분하는 데 사용됩니다.
  - **비식별 파라미터 (`isIdentifying=false`)**: `JobInstance` 식별에는 영향을 주지 않지만, 잡 실행 중 참조할 수 있는 값입니다. 예를 들어, 매번 실행 시 다른 로그 레벨을 지정하고 싶지만, 이것이 `JobInstance`를 구분하는 기준이 되어서는 안 될 때 사용합니다. (하지만 대부분의 경우 모든 파라미터를 식별 파라미터로 사용하는 것이 일반적입니다.)

---

**1. `commit-interval`의 의미**

- `commit-interval`은 **하나의 트랜잭션으로 처리될 아이템의 개수, 즉 청크(Chunk)의 크기**를 의미합니다.
- `StepBuilder`에서 `.chunk(int commitInterval, PlatformTransactionManager transactionManager)` 형태로 설정합니다.
- 예를 들어 `chunk(10, ...)`으로 설정하면, 10개의 아이템이 하나의 청크를 이루고, 이 10개 아이템에 대한 읽기-처리-쓰기 작업이 완료된 후 트랜잭션이 커밋(또는 롤백)됩니다.

**2. `commit-interval`과 각 컴포넌트의 상호작용 및 트랜잭션 관리**

- **`ItemReader<I>`:**
  - `read()` 메서드는 호출될 때마다 **아이템을 하나씩** 읽어옵니다.
  - Spring Batch 프레임워크(정확히는 `ChunkOrientedTasklet`)는 `commit-interval`에 도달할 때까지 또는 `read()`가 `null`을 반환할 때까지 이 `read()` 메서드를 반복적으로 호출하여 아이템을 내부 버퍼(리스트)에 쌓습니다.
- **`ItemProcessor<I, O>` (선택 사항):**
  - `ItemReader`가 읽어온 각 아이템은 (만약 `ItemProcessor`가 설정되어 있다면) `process()` 메서드로 전달되어 **하나씩 처리(변환/필터링)**됩니다.
  - 처리된 결과(`O` 타입 객체 또는 `null`)는 다시 프레임워크의 내부 버퍼에 (다른 리스트일 수 있음) 쌓입니다. `null`이 반환되면 해당 아이템은 버려집니다.
- **`ItemWriter<O>`:**
  - 내부 버퍼에 `commit-interval` 만큼의 처리된 아이템(`O` 타입 객체)이 쌓이거나, `ItemReader`가 `null`을 반환하여 더 이상 읽을 아이템이 없어 현재까지 모인 아이템들로 마지막 청크를 구성해야 할 때,
  - Spring Batch 프레임워크는 이 아이템들의 묶음을 **`Chunk<? extends O>` 객체로 만들어 `ItemWriter`의 `write(Chunk<? extends O> items)` 메서드를 호출**합니다.
  - `ItemWriter`는 이 `Chunk` 안에 있는 **모든 아이템들을 한 번의 `write()` 호출 내에서** 목적지에 기록합니다. 학생분이 "버퍼에 쌓아놨다가 write 한다"고 표현하신 부분이 바로 이 지점입니다. `ItemWriter`는 "묶음"을 전달받는 것이죠.
- **트랜잭션 관리 (`PlatformTransactionManager`):**
  - 하나의 청크에 대한 **`ItemReader.read()` (여러 번) -> `ItemProcessor.process()` (여러 번) -> `ItemWriter.write()` (한 번)** 이 모든 과정이 **하나의 트랜잭션 범위 내에서 실행됩니다.**
  - `ItemWriter.write()`까지 성공적으로 완료되면, 해당 트랜잭션이 **커밋(commit)**됩니다. 이때 `JobRepository`를 통해 메타데이터(예: `StepExecution`의 읽은 수, 쓴 수, `ExecutionContext`의 변경 사항 등)도 함께 커밋됩니다.
  - 만약 이 과정 중 어디선가 (Reader, Processor, Writer 내부 또는 프레임워크 레벨에서) 예외가 발생하여 트랜잭션이 롤백(rollback)되어야 한다면, 해당 청크에서 처리하려 했던 모든 아이템에 대한 작업(DB 변경 등)이 취소됩니다.
  - 그리고 설정된 Skip/Retry 정책에 따라 다음 동작(건너뛰기, 재시도, 스텝 실패 등)이 결정됩니다.

**흐름 요약 (commit-interval = 3 가정):**

1. **트랜잭션 시작**
2. `Reader.read()` -> 아이템1 반환
3. `Processor.process(아이템1)` -> 처리된_아이템1 반환 (버퍼에 추가)
4. `Reader.read()` -> 아이템2 반환
5. `Processor.process(아이템2)` -> 처리된_아이템2 반환 (버퍼에 추가)
6. `Reader.read()` -> 아이템3 반환
7. `Processor.process(아이템3)` -> 처리된_아이템3 반환 (버퍼에 추가)
  - 이제 버퍼에 3개의 아이템이 모였으므로 (`commit-interval` 도달)
8. `Writer.write(Chunk[처리된_아이템1, 처리된_아이템2, 처리된_아이템3])` 호출
9. (Writer가 성공적으로 쓰기 완료)
10. **트랜잭션 커밋** (메타데이터 업데이트 포함)
11. (다음 청크를 위해 1번부터 반복)

만약 8번 `Writer.write()`에서 예외가 발생하면 10번은 커밋이 아니라 롤백이 됩니다.

---

**1. `TaskletStep`의 사용 사례**

- `TaskletStep`은 `ItemReaderItemProcessorItemWriter`로 이어지는 반복적인 청크 지향 처리보다는, **스텝 내에서 하나의 응집된 작업(single task)을 수행**하고자 할 때 사용합니다.
- **주요 사용 사례:**
  - **초기화/정리 작업:** 배치 잡 시작 전 필요한 임시 테이블 생성, 또는 잡 완료 후 임시 파일 삭제 등.
  - **간단한 SQL 실행:** 특정 업데이트 쿼리나 DDL 실행, 저장 프로시저 호출 등.
  - **시스템 명령어 실행:** 외부 셸 스크립트 실행 등.
  - **간단한 비즈니스 로직 호출:** 특정 서비스 메서드를 한 번 호출하여 작업 완료.
  - **리소스 유효성 검사:** 처리할 파일이 존재하는지, 필요한 DB 연결이 가능한지 등 스텝 시작 전 검증 작업.

**2. `TaskletStep`과 메모리 사용량**

- **메모리 사용량은 `Tasklet` 구현체 내에서 어떤 작업을 하느냐에 따라 결정됩니다.**
  - 만약 `Tasklet`이 대용량 파일을 한 번에 읽어 메모리에서 처리하는 로직을 가지고 있다면, 해당 `TaskletStep`은 OOM을 유발할 수 있습니다.
  - 반대로, `Tasklet`이 단순히 파일 하나를 삭제하거나, 간단한 SQL 쿼리 하나를 실행하는 것이라면 메모리 사용량은 매우 낮을 것입니다.
- **청크 지향 스텝과의 비교:** 일반적으로 **대용량 아이템 처리** 시에는 청크 지향 스텝이 `ItemReader`를 통해 데이터를 스트리밍 방식으로 조금씩 읽어오고, `commit-interval` 단위로 처리 후 메모리에서 해제될 수 있도록 설계되므로 **OOM 방지에 더 유리**합니다. `TaskletStep`에서 대용량 데이터를 직접 다루려면 개발자가 메모리 관리에 매우 신경 써야 합니다.
- **결론:** `TaskletStep`은 "간단한 단일 작업"에 적합하며, 이 작업 자체가 대규모 데이터를 메모리에 올리는 것이 아니라면 메모리 문제는 크게 걱정하지 않아도 됩니다. 만약 `Tasklet` 내에서 대량의 데이터를 처리해야 한다면, 청크 지향 스텝으로 전환하거나 `Tasklet` 내부 로직을 매우 신중하게 설계해야 합니다.

**3. `Tasklet` 인터페이스의 핵심 메서드와 `RepeatStatus`**

`Tasklet` 인터페이스에는 단 하나의 메서드만 정의되어 있습니다.

- **핵심 메서드:** `RepeatStatus execute(StepContribution contribution, ChunkContext chunkContext) throws Exception;`
  - `contribution`: 현재 스텝 실행에 대한 통계 정보(읽은 수, 쓴 수 등)를 업데이트하는 데 사용.
  - `chunkContext`: 스텝 실행 컨텍스트 정보(JobParameters, ExecutionContext 등)에 접근하는 데 사용.
  - 이 메서드 안에 실제 수행할 작업을 구현합니다.
- **`RepeatStatus` 반환값의 의미:**
  - `execute()` 메서드는 `RepeatStatus`라는 열거형 값을 반환해야 합니다. 이 값은 **해당 `Tasklet`의 실행을 계속 반복할지, 아니면 완료할지를 프레임워크에 알려주는 신호**입니다.
  - **`RepeatStatus.FINISHED`**:
    - **의미:** `Tasklet`의 작업이 **완료되었으며, 더 이상 반복 실행할 필요가 없음**을 나타냅니다.
    - **동작:** Spring Batch는 이 값을 받으면 해당 `TaskletStep`의 실행을 성공적으로 종료합니다.
    - **대부분의 `Tasklet` 구현은 이 값을 반환합니다.** "단일 작업"을 수행하는 목적에 부합하기 때문입니다.
  - **`RepeatStatus.CONTINUABLE`**:
    - **의미:** `Tasklet`의 작업이 아직 **완료되지 않았으며, 프레임워크가 `execute()` 메서드를 다시 호출해야 함**을 나타냅니다.
    - **동작:** Spring Batch는 이 값을 받으면, 현재 트랜잭션을 커밋하고 (기본 설정 시) **즉시 동일한 `Tasklet`의 `execute()` 메서드를 다시 호출**합니다.
    - **사용 사례 (매우 드묾):**
      - 하나의 `Tasklet`이 내부적으로 어떤 상태를 가지고 있고, 여러 번의 `execute()` 호출을 통해 점진적으로 작업을 완료해야 하는 매우 특수한 경우에 사용될 수 있습니다. (예: 매우 긴 시간 동안 폴링하며 특정 조건이 만족될 때까지 기다리는 작업, 하지만 이런 경우는 보통 별도의 스텝으로 나누거나 다른 방식으로 설계하는 것이 좋습니다.)
      - **주의:** `CONTINUABLE`을 반환하는 로직을 잘못 설계하면 **무한 루프**에 빠질 수 있으므로 매우 신중하게 사용해야 합니다. `Tasklet` 내부에서 언젠가는 `FINISHED`를 반환하도록 조건을 변경하는 로직이 반드시 포함되어야 합니다.

**일반적으로, 우리가 만드는 대부분의 `Tasklet`은 한 번의 `execute()` 호출로 작업을 마치고 `RepeatStatus.FINISHED`를 반환하게 됩니다.**

```java
public class MySimpleTasklet implements Tasklet {

    @Override
    public RepeatStatus execute(StepContribution contribution, ChunkContext chunkContext) throws Exception {
        System.out.println("MySimpleTasklet is executing its single task!");
        // 여기에 실제 작업 수행 (예: 파일 삭제, DB 쿼리 실행 등)

        // 작업이 완료되었음을 알림
        return RepeatStatus.FINISHED;
    }
}
```

---

### **`RepeatStatus.CONTINUABLE` 상세 설명 및 예시**

`Tasklet`의 `execute()` 메서드가 `RepeatStatus.CONTINUABLE`을 반환하면, Spring Batch 프레임워크는 다음과 같이 동작합니다:

1. 현재 `execute()` 메서드 호출까지의 작업 내용에 대해 **트랜잭션을 커밋**합니다. (이는 `TaskletStep`도 기본적으로 트랜잭션 내에서 실행되기 때문입니다. `PlatformTransactionManager`가 설정되어 있다면요.)
2. `StepContribution`에 기록된 통계 (읽은 수, 쓴 수 등)를 `StepExecution`에 반영합니다.
3. **즉시 동일한 `Tasklet` 인스턴스의 `execute()` 메서드를 다시 호출합니다.**
4. 이 과정은 `Tasklet`이 `RepeatStatus.FINISHED`를 반환할 때까지 반복됩니다.

**왜 `CONTINUABLE`이 필요할까? (매우 드문 사용 사례)**

`CONTINUABLE`은 `Tasklet`이 **하나의 논리적인 작업을 여러 번의 작은 실행 단위로 나누어 점진적으로 수행**하고 싶을 때, 그리고 각 작은 실행 단위가 **개별적인 트랜잭션으로 커밋되어야 할 때** 이론적으로 사용될 수 있습니다.

**가상 시나리오 예시: "긴급 데이터 동기화 폴러(Poller) Tasklet"**

외부 시스템에 특정 데이터가 준비되었는지 주기적으로 확인하고, 준비되었으면 해당 데이터를 가져와 로컬 DB에 반영하는 `Tasklet`이 있다고 가정해봅시다. 이 작업은 외부 시스템의 응답이 느릴 수 있고, 한 번의 `execute()` 호출로 모든 확인 및 동기화 작업을 마치기 어려울 수 있습니다.

```java
public class EmergencyDataSyncPollerTasklet implements Tasklet, StepExecutionListener {

    private static final Logger LOGGER = LoggerFactory.getLogger(EmergencyDataSyncPollerTasklet.class);
    private static final int MAX_POLL_ATTEMPTS = 5; // 최대 폴링 시도 횟수
    private int currentPollAttempt = 0; // 현재 폴링 시도 횟수 (StepExecution의 ExecutionContext에 저장/로드)
    private static final String POLL_ATTEMPT_KEY = "poller.currentAttempt";

    private ExternalSystemService externalService; // 외부 시스템과 통신하는 서비스
    private LocalDbService localDbService;       // 로컬 DB에 쓰는 서비스

    public EmergencyDataSyncPollerTasklet(ExternalSystemService externalService, LocalDbService localDbService) {
        this.externalService = externalService;
        this.localDbService = localDbService;
    }

    @Override
    public void beforeStep(StepExecution stepExecution) {
        // 스텝 시작 시 ExecutionContext에서 이전 시도 횟수 로드
        ExecutionContext stepContext = stepExecution.getExecutionContext();
        if (stepContext.containsKey(POLL_ATTEMPT_KEY)) {
            currentPollAttempt = stepContext.getInt(POLL_ATTEMPT_KEY);
        }
        LOGGER.info("EmergencyDataSyncPollerTasklet: Starting poll attempt #{}", (currentPollAttempt + 1));
    }

    @Override
    public RepeatStatus execute(StepContribution contribution, ChunkContext chunkContext) throws Exception {
        currentPollAttempt++;
        LOGGER.info("Executing poll attempt #{}", currentPollAttempt);

        // 1. 외부 시스템에 데이터 준비 상태 확인
        boolean isDataReady = externalService.isEmergencyDataReady();

        if (isDataReady) {
            LOGGER.info("Emergency data is ready!");
            // 2. 데이터 가져오기
            List<SomeData> dataToSync = externalService.fetchEmergencyData();
            // 3. 로컬 DB에 데이터 반영
            localDbService.saveData(dataToSync);
            contribution.incrementWriteCount(dataToSync.size()); // 통계 업데이트
            LOGGER.info("Emergency data synchronized successfully. Processed {} items.", dataToSync.size());
            // 작업 완료, FINISHED 반환
            return RepeatStatus.FINISHED;
        } else {
            LOGGER.info("Emergency data is not ready yet (Attempt {}/{})", currentPollAttempt, MAX_POLL_ATTEMPTS);
            if (currentPollAttempt < MAX_POLL_ATTEMPTS) {
                // 아직 최대 시도 횟수에 도달하지 않았으면, 잠시 후 다시 시도하도록 CONTINUABLE 반환
                // (실제로는 Thread.sleep() 등으로 지연을 줄 수도 있지만, 여기서는 즉시 재호출 가정)
                // 현재까지의 폴링 시도 횟수를 ExecutionContext에 저장하여 다음 execute 호출 시 사용
                chunkContext.getStepContext().getStepExecution().getExecutionContext().putInt(POLL_ATTEMPT_KEY, currentPollAttempt);
                return RepeatStatus.CONTINUABLE;
            } else {
                // 최대 시도 횟수 초과, 작업 실패 또는 다른 처리 후 FINISHED
                LOGGER.warn("Max poll attempts reached. Emergency data not synchronized.");
                // 이 경우, 잡을 실패시키거나, 경고만 남기고 FINISHED로 처리할 수 있음
                // throw new RuntimeException("Failed to sync emergency data after " + MAX_POLL_ATTEMPTS + " attempts.");
                return RepeatStatus.FINISHED; // 또는 다른 ExitStatus와 함께 종료
            }
        }
    }

    @Override
    public ExitStatus afterStep(StepExecution stepExecution) {
        // 스텝 종료 시 정리 (필요하다면)
        LOGGER.info("EmergencyDataSyncPollerTasklet finished.");
        if (stepExecution.getStatus() == BatchStatus.COMPLETED && currentPollAttempt >= MAX_POLL_ATTEMPTS && !externalService.isEmergencyDataReady()) {
            // 최대 시도 후에도 데이터가 준비되지 않았다면, ExitStatus를 변경하여 잡 흐름 제어 가능
            // return new ExitStatus("SYNC_TIMEOUT");
        }
        return null; // null을 반환하면 기본 ExitStatus 사용
    }
}

// --- 가상의 서비스들 ---
// interface ExternalSystemService {
//    boolean isEmergencyDataReady();
//    List<SomeData> fetchEmergencyData();
// }
// interface LocalDbService {
//    void saveData(List<SomeData> data);
// }
// class SomeData { /* ... */ }

```

**위 예시의 동작 흐름 (CONTINUABLE 반환 시):**

1. **`beforeStep`**: `currentPollAttempt`를 `ExecutionContext`에서 읽어와 초기화합니다 (재시작 시 이전 값 복원).
2. **`execute` (첫 번째 호출)**:
  - `currentPollAttempt`가 1이 됩니다.
  - `externalService.isEmergencyDataReady()`를 호출하여 데이터 준비 상태를 확인합니다.
  - **만약 `false`이고 `currentPollAttempt < MAX_POLL_ATTEMPTS` 라면:**
    - `ExecutionContext`에 `currentPollAttempt` (현재 값: 1)를 저장합니다.
    - `RepeatStatus.CONTINUABLE`을 반환합니다.
3. **프레임워크 동작:**
  - `CONTINUABLE`을 받았으므로, 현재까지의 `StepContribution` (여기서는 특별히 변경된 것 없음)과 `ExecutionContext` 변경 사항(POLL\_ATTEMPT\_KEY=1)을 포함하여 트랜잭션을 커밋합니다.
  - **즉시 `EmergencyDataSyncPollerTasklet`의 `execute()` 메서드를 다시 호출합니다.**
4. **`execute` (두 번째 호출)**:
  - `currentPollAttempt`가 2가 됩니다. (이전 `execute`에서 증가된 상태 유지)
  - 다시 `isEmergencyDataReady()`를 호출합니다.
  - **만약 또 `false`이고 `currentPollAttempt < MAX_POLL_ATTEMPTS` 라면:**
    - `ExecutionContext`에 `currentPollAttempt` (현재 값: 2)를 저장합니다.
    - `RepeatStatus.CONTINUABLE`을 반환합니다.
5. 이 과정이 `isEmergencyDataReady()`가 `true`를 반환하거나, `currentPollAttempt`가 `MAX_POLL_ATTEMPTS`에 도달할 때까지 반복됩니다.
6. **`isEmergencyDataReady()`가 `true`를 반환하면:**
  - 데이터를 가져와 로컬 DB에 저장합니다.
  - `contribution.incrementWriteCount()` 등으로 통계를 업데이트합니다.
  - `RepeatStatus.FINISHED`를 반환합니다.
7. **프레임워크 동작:**
  - `FINISHED`를 받았으므로, 현재 `StepContribution`과 `ExecutionContext` 변경 사항을 포함하여 트랜잭션을 커밋하고, 스텝 실행을 종료합니다.

**주의사항 및 대안:**

- **무한 루프 위험:** `CONTINUABLE`을 반환하는 `Tasklet`은 반드시 언젠가는 `FINISHED`를 반환할 수 있는 종료 조건을 가져야 합니다. 그렇지 않으면 스텝이 끝나지 않고 무한 루프에 빠집니다.
- **트랜잭션 단위:** `CONTINUABLE` 반환 시마다 트랜잭션이 커밋됩니다. 이는 각 반복이 독립적인 작업 단위로 처리되어야 할 때 유용할 수 있지만, 여러 반복이 하나의 원자적인 작업으로 묶여야 한다면 적합하지 않습니다.
- **실제 사용 빈도:** `RepeatStatus.CONTINUABLE`은 Spring Batch에서 매우 드물게 사용됩니다. 대부분의 "반복"적인 작업은 청크 지향 스텝으로 처리하거나, 여러 개의 개별 스텝으로 나누어 플로우로 제어하는 것이 더 일반적이고 관리하기 쉽습니다.
  - 위 예시 같은 폴링 작업도, `Tasklet` 내부에서 `Thread.sleep()`을 사용하며 루프를 돌고 한 번의 `execute()` 내에서 `FINISHED`를 반환하거나, 아예 Spring Integration 같은 다른 프레임워크를 연동하는 것이 더 나은 설계일 수 있습니다.

**결론적으로, `RepeatStatus.CONTINUABLE`은 `Tasklet`의 `execute()` 메서드를 (각 호출 후 트랜잭션 커밋과 함께) 반복적으로 실행시키고 싶을 때 사용하지만, 매우 신중하게, 그리고 명확한 종료 조건과 함께 사용해야 하는 고급 기능입니다.** 대부분의 일반적인 `Tasklet`은 한 번의 작업 후 `RepeatStatus.FINISHED`를 반환합니다.

---
