---
title: Spring Batch - Domain Language
description: 
author: laze
date: 2025-05-14 00:00:04 +0900
categories: [Dev, SpringBatch]
tags: [SpringBatch]
---
**배치의 도메인 언어**

경험 많은 배치 아키텍트라면 Spring Batch에서 사용되는 배치 처리의 전반적인 개념이 익숙하고 편안할 것입니다. "Job"과 "Step"이 있고, 개발자가 제공하는 처리 단위인 `ItemReader`와 `ItemWriter`가 있습니다.

그러나 Spring의 패턴, 연산, 템플릿, 콜백, 관용구로 인해 다음과 같은 기회가 있습니다:

- 명확한 관심사 분리(separation of concerns) 준수 능력의 상당한 향상.
- 명확하게 구분된 아키텍처 계층 및 인터페이스로 제공되는 서비스.
- 즉시 사용 가능한(out of the box) 빠르고 쉬운 채택을 가능하게 하는 간단하고 기본적(default)인 구현.
- 상당히 향상된 확장성.

Spring Batch는 단순한 배치 애플리케이션부터 매우 복잡한 처리 요구를 해결하기 위한 인프라와 확장을 갖춘 복잡한 배치 애플리케이션 생성에 이르기까지,

견고하고 유지보수 가능한 시스템에서 흔히 볼 수 있는 계층, 구성 요소 및 기술 서비스의 물리적 구현을 제공합니다.

위 다이어그램은 Spring Batch의 도메인 언어를 구성하는 핵심 개념을 강조합니다.

`Job`은 하나 이상의 `Step`을 가지며, 각 `Step`은 정확히 하나의 `ItemReader`, 하나의 `ItemProcessor`, 그리고 하나의 `ItemWriter`를 가집니다.

`Job`은 (`JobLauncher`를 통해) 실행되어야 하며, 현재 실행 중인 프로세스에 대한 메타데이터는 (`JobRepository`에) 저장되어야 합니다.

**Job**

이 섹션에서는 배치 작업(batch job)의 개념과 관련된 스테레오타입을 설명합니다.

`Job`은 전체 배치 프로세스를 캡슐화하는 엔티티입니다.

다른 Spring 프로젝트와 마찬가지로, `Job`은 XML 구성 파일 또는 Java 기반 구성을 통해 연결(wired)됩니다.

이 구성을 "Job 구성(job configuration)"이라고 할 수 있습니다.

그러나 `Job`은  전체 계층 구조의 최상위에 불과합니다.

Spring Batch에서 `Job`은 단순히 `Step` 인스턴스를 위한 컨테이너입니다.

논리적으로 함께 속하는 여러 `Step`들을 하나의 흐름으로 결합하고, 재시작 가능성(restartability)과 같이 모든 `Step`에 공통적인 속성을 구성할 수 있도록 합니다.

Job 구성에는 다음이 포함됩니다:

- Job의 이름.
- `Step` 인스턴스의 정의 및 순서.
- Job의 재시작 가능 여부.

Java 구성을 사용하는 사람들을 위해 Spring Batch는 `Job` 인터페이스의 기본 구현으로 `SimpleJob` 클래스를 제공하며, 이는 `Job` 위에 몇 가지 표준 기능을 만듭니다.

Java 기반 구성을 사용할 때, 다음 예제와 같이 `Job` 인스턴스화를 위한 빌더 컬렉션이 제공됩니다:

```java
@Bean
public Job footballJob(JobRepository jobRepository) {
    return new JobBuilder("footballJob", jobRepository)
                     .start(playerLoad())
                     .next(gameLoad())
                     .next(playerSummarization())
                     .build();
}
```

**JobInstance**

`JobInstance`는 논리적인 Job 실행(run)의 개념을 나타냅니다.

앞의 다이어그램에 있는 `EndOfDay` Job과 같이 하루가 끝날 때 한 번 실행되어야 하는 배치 작업을 생각해 보십시오.

`EndOfDay` Job은 하나지만, `Job`의 각 개별 실행은 별도로 추적되어야 합니다.

이 Job의 경우, 하루에 하나의 논리적 `JobInstance`가 있습니다.

예를 들어, 1월 1일 실행, 1월 2일 실행 등이 있습니다.

만약 1월 1일 실행이 처음 실패하고 다음 날 다시 실행되더라도, 그것은 여전히 1월 1일 실행입니다. (보통 이것은 처리하는 데이터와도 일치하며, 즉 1월 1일 실행은 1월 1일의 데이터를 처리합니다).

따라서 각 `JobInstance`는 여러 번의 실행(`JobExecution`은 이 장의 뒷부분에서 더 자세히 설명합니다)을 가질 수 있으며,

특정 `Job`과 식별용 `JobParameters`에 해당하는 단 하나의 `JobInstance`만이 주어진 시간에 실행될 수 있습니다.

`JobInstance`의 정의는 로드할 데이터와 전혀 관련이 없습니다.

데이터를 로드하는 방법은 전적으로 `ItemReader` 구현에 달려 있습니다.

예를 들어, `EndOfDay` 시나리오에서 데이터가 속한 유효 날짜 또는 스케줄 날짜를 나타내는 열이 데이터에 있을 수 있습니다.

따라서 1월 1일 실행은 1일의 데이터만 로드하고, 1월 2일 실행은 2일의 데이터만 사용합니다.

이러한 결정은 비즈니스 결정일 가능성이 높으므로 `ItemReader`가 결정하도록 남겨둡니다.

그러나 동일한 `JobInstance`를 사용하면 이전 실행의 "상태"(`ExecutionContext`)를 사용할지 여부가 결정됩니다.

새로운 `JobInstance`를 사용하는 것은 "처음부터 시작"을 의미하며, 기존 인스턴스를 사용하는 것은 일반적으로 "중단된 부분부터 시작"을 의미합니다.

**JobParameters**

`JobInstance`에 대해 논의하고 `Job`과의 차이점을 설명했으므로, 자연스럽게 드는 질문은 "하나의 `JobInstance`는 다른 `JobInstance`와 어떻게 구별되는가?"입니다.

답은 `JobParameters`입니다. `JobParameters` 객체는 배치 작업을 시작하는 데 사용되는 매개변수 세트를 보유합니다.

식별용으로 사용되거나 실행 중 참조 데이터로 사용될 수도 있습니다.

앞의 예에서 1월 1일과 1월 2일에 대한 두 개의 인스턴스가 있는 경우, 실제로는 `Job`은 하나뿐이지만 두 개의 `JobParameter` 객체를 가집니다:

하나는 01-01-2017이라는 Job 매개변수로 시작되었고, 다른 하나는 01-02-2017이라는 매개변수로 시작되었습니다.

따라서 계약은 다음과 같이 정의될 수 있습니다: `JobInstance = Job + 식별용 JobParameters`.

이를 통해 개발자는 전달되는 매개변수를 제어하므로 `JobInstance`가 어떻게 정의되는지를 효과적으로 제어할 수 있습니다.

모든 Job 매개변수가 `JobInstance` 식별에 기여해야 하는 것은 아닙니다. 기본적으로는 그렇게 합니다.

그러나 프레임워크는 `JobInstance`의 ID에 기여하지 않는 매개변수로 `Job`을 제출하는 것도 허용합니다.

**JobExecution**

`JobExecution`은 `Job`을 실행하려는 단일 시도에 대한 기술적인 개념을 나타냅니다.

실행은 실패 또는 성공으로 끝날 수 있지만, 특정 실행에 해당하는 `JobInstance`는 실행이 성공적으로 완료되지 않는 한 완료된 것으로 간주되지 않습니다.

앞에서 설명한 `EndOfDay` Job을 예로 들어, 처음 실행했을 때 실패한 01-01-2017의 `JobInstance`를 생각해 보십시오.

만약 첫 번째 실행과 동일한 식별용 Job 매개변수(01-01-2017)로 다시 실행되면, 새로운 `JobExecution`이 생성됩니다.

그러나 여전히 `JobInstance`는 하나뿐입니다.

`Job`은 작업이 무엇이며 어떻게 실행되어야 하는지를 정의하고, `JobInstance`는 주로 올바른 재시작 의미 체계를 가능하게 하기 위해 실행들을 함께 그룹화하는 순전히 조직적인 객체입니다.

그러나 `JobExecution`은 실행 중에 실제로 발생한 일에 대한 기본 저장 메커니즘이며, 다음 표와 같이 제어하고 영속화해야 하는 훨씬 더 많은 속성을 포함합니다.

JobExecution 속성

| 속성 | 정의 |
| --- | --- |
| Status | 실행 상태를 나타내는 `BatchStatus` 객체. 실행 중에는 `BatchStatus#STARTED`. 실패하면 `BatchStatus#FAILED`. 성공적으로 완료되면 `BatchStatus#COMPLETED` |
| startTime | 실행이 시작된 현재 시스템 시간을 나타내는 `java.time.LocalDateTime`. 작업이 아직 시작되지 않았으면 이 필드는 비어 있습니다. |
| endTime | 성공 여부에 관계없이 실행이 완료된 현재 시스템 시간을 나타내는 `java.time.LocalDateTime`. 작업이 아직 완료되지 않았으면 이 필드는 비어 있습니다. |
| exitStatus | 실행 결과를 나타내는 `ExitStatus`. 호출자에게 반환되는 종료 코드를 포함하므로 가장 중요합니다. 자세한 내용은 5장을 참조하십시오. 작업이 아직 완료되지 않았으면 이 필드는 비어 있습니다. |
| createTime | `JobExecution`이 처음 영속화된 현재 시스템 시간을 나타내는 `java.time.LocalDateTime`. 작업이 아직 시작되지 않았을 수 있지만(따라서 시작 시간이 없음), 항상 `createTime`을 가지며, 이는 Job 수준 `ExecutionContext`를 관리하기 위해 프레임워크에 필요합니다. |
| lastUpdated | `JobExecution`이 마지막으로 영속화된 시간을 나타내는 `java.time.LocalDateTime`. 작업이 아직 시작되지 않았으면 이 필드는 비어 있습니다. |
| executionContext | 실행 간에 영속화해야 하는 사용자 데이터를 포함하는 "속성 가방(property bag)". |
| failureExceptions | `Job` 실행 중 발생한 예외 목록. `Job` 실패 시 둘 이상의 예외가 발생한 경우 유용할 수 있습니다. |

이러한 속성은 영속화되며 실행 상태를 완전히 결정하는 데 사용될 수 있기 때문에 중요합니다.

예를 들어, 01-01의 `EndOfDay` 작업이 오후 9시에 실행되어 오후 9시 30분에 실패하면 배치 메타데이터 테이블에 다음 항목이 만들어집니다:

BATCH_JOB_INSTANCE

| JOB_INST_ID | JOB_NAME |
| --- | --- |
| 1 | EndOfDayJob |

BATCH_JOB_EXECUTION_PARAMS

| JOB_EXECUTION_ID | TYPE_CD | KEY_NAME | DATE_VAL | IDENTIFYING |
| --- | --- | --- | --- | --- |
| 1 | DATE | schedule.Date | 2017-01-01 | TRUE |

BATCH_JOB_EXECUTION

| JOB_EXEC_ID | JOB_INST_ID | START_TIME | END_TIME | STATUS |
| --- | --- | --- | --- | --- |
| 1 | 1 | 2017-01-01 21:00 | 2017-01-01 21:30 | FAILED |

이제 작업이 실패했으므로 문제를 파악하는 데 밤새도록 걸렸다고 가정하고, "배치 창"은 이제 닫혔습니다.

또한 창이 오후 9시에 시작한다고 가정하면, 01-01 작업이 중단된 지점부터 다시 시작되어 오후 9시 30분에 성공적으로 완료됩니다.

이제 다음 날이므로 01-02 작업도 실행해야 하며, 바로 뒤인 오후 9시 31분에 시작되어 정상적인 1시간 후인 오후 10시 30분에 완료됩니다.

두 작업이 동일한 데이터에 액세스하려고 시도하여 데이터베이스 수준에서 잠금 문제를 일으킬 가능성이 없는 한, 한 `JobInstance`가 다른 `JobInstance` 이후에 시작되어야 한다는 요구 사항은 없습니다.

`Job`이 언제 실행되어야 하는지는 전적으로 스케줄러가 결정합니다.

별도의 `JobInstance`이므로 Spring Batch는 동시에 실행되는 것을 막으려고 시도하지 않습니다. (이미 실행 중인 `JobInstance`를 실행하려고 하면 `JobExecutionAlreadyRunningException`이 발생합니다).

이제 `JobInstance` 및 `JobParameters` 테이블에 추가 항목이 하나씩 있어야 하고 `JobExecution` 테이블에는 두 개의 추가 항목이 있어야 합니다. 다음 표와 같습니다:

BATCH_JOB_INSTANCE

| JOB_INST_ID | JOB_NAME |
| --- | --- |
| 1 | EndOfDayJob |
| 2 | EndOfDayJob |

BATCH_JOB_EXECUTION_PARAMS

| JOB_EXECUTION_ID | TYPE_CD | KEY_NAME | DATE_VAL | IDENTIFYING |
| --- | --- | --- | --- | --- |
| 1 | DATE | schedule.Date | 2017-01-01 00:00:00 | TRUE |
| 2 | DATE | schedule.Date | 2017-01-01 00:00:00 | TRUE |
| 3 | DATE | schedule.Date | 2017-01-02 00:00:00 | TRUE |

BATCH_JOB_EXECUTION

| JOB_EXEC_ID | JOB_INST_ID | START_TIME | END_TIME | STATUS |
| --- | --- | --- | --- | --- |
| 1 | 1 | 2017-01-01 21:00 | 2017-01-01 21:30 | FAILED |
| 2 | 1 | 2017-01-02 21:00 | 2017-01-02 21:30 | COMPLETED |
| 3 | 2 | 2017-01-02 21:31 | 2017-01-02 22:29 | COMPLETED |

**Step**

`Step`은 배치 작업의 독립적이고 순차적인 단계를 캡슐화하는 도메인 객체입니다.

따라서 모든 `Job`은 전적으로 하나 이상의 `Step`으로 구성됩니다.

`Step`은 실제 배치 처리를 정의하고 제어하는 데 필요한 모든 정보를 포함합니다.

특정 `Step`의 내용은 `Job`을 작성하는 개발자의 재량에 달려 있기 때문에 이는 필연적으로 모호한 설명입니다.

`Step`은 개발자가 원하는 만큼 단순하거나 복잡할 수 있습니다.

간단한 `Step`은 파일에서 데이터베이스로 데이터를 로드할 수 있으며, 코드(사용된 구현에 따라 다름)가 거의 또는 전혀 필요하지 않을 수 있습니다.

더 복잡한 `Step`은 처리의 일부로 적용되는 복잡한 비즈니스 규칙을 가질 수 있습니다.

`Job`과 마찬가지로 `Step`은 다음 이미지와 같이 고유한 `JobExecution`과 관련된 개별 `StepExecution`을 가집니다.

**StepExecution**

`StepExecution`은 `Step`을 실행하려는 단일 시도를 나타냅니다.

`JobExecution`과 유사하게 `Step`이 실행될 때마다 새로운 `StepExecution`이 생성됩니다.

그러나 이전 `Step`이 실패하여 `Step`이 실행되지 않으면 해당 `Step`에 대한 실행은 영속화되지 않습니다.

`StepExecution`은 해당 `Step`이 실제로 시작될 때만 생성됩니다.

`Step` 실행은 `StepExecution` 클래스의 객체로 표현됩니다.

각 실행은 해당 `Step` 및 `JobExecution`에 대한 참조와 커밋 및 롤백 횟수, 시작 및 종료 시간과 같은 트랜잭션 관련 데이터를 포함합니다.

또한 각 `Step` 실행에는 통계 또는 재시작에 필요한 상태 정보와 같이 개발자가 배치 실행 간에 영속화해야 하는 모든 데이터를 포함하는 `ExecutionContext`가 포함됩니다.

다음 표는 `StepExecution`의 속성을 나열합니다.

StepExecution 속성

| 속성 | 정의 |
| --- | --- |
| Status | 실행 상태를 나타내는 `BatchStatus` 객체. 실행 중에는 `BatchStatus.STARTED`. 실패하면 `BatchStatus.FAILED`. 성공적으로 완료되면 `BatchStatus.COMPLETED`. |
| startTime | 실행이 시작된 현재 시스템 시간을 나타내는 `java.time.LocalDateTime`. Step이 아직 시작되지 않았으면 이 필드는 비어 있습니다. |
| endTime | 성공 여부에 관계없이 실행이 완료된 현재 시스템 시간을 나타내는 `java.time.LocalDateTime`. Step이 아직 종료되지 않았으면 이 필드는 비어 있습니다. |
| exitStatus | 실행 결과를 나타내는 `ExitStatus`. 호출자에게 반환되는 종료 코드를 포함하므로 가장 중요합니다. 자세한 내용은 5장을 참조하십시오. Job이 아직 종료되지 않았으면 이 필드는 비어 있습니다. |
| executionContext | 실행 간에 영속화해야 하는 사용자 데이터를 포함하는 "속성 가방(property bag)". |
| readCount | 성공적으로 읽은 항목 수. |
| writeCount | 성공적으로 쓴 항목 수. |
| commitCount | 이 실행에 대해 커밋된 트랜잭션 수. |
| rollbackCount | `Step`에 의해 제어되는 비즈니스 트랜잭션이 롤백된 횟수. |
| readSkipCount | 읽기 실패로 인해 항목을 건너뛴 횟수. |
| processSkipCount | 처리 실패로 인해 항목을 건너뛴 횟수. |
| filterCount | `ItemProcessor`에 의해 "필터링된" 항목 수. |
| writeSkipCount | 쓰기 실패로 인해 항목을 건너뛴 횟수. |

**ExecutionContext**

`ExecutionContext`는 개발자가 `StepExecution` 객체 또는 `JobExecution` 객체 범위에 영속적인 상태를 저장할 수 있는 장소를 제공하기 위해 프레임워크에 의해 영속화되고 제어되는 키/값 쌍의 컬렉션을 나타냅니다.

(Quartz에 익숙한 사람들에게는 `JobDataMap`과 매우 유사합니다.)

가장 좋은 사용 예는 재시작을 용이하게 하는 것입니다.

플랫 파일 입력을 예로 들면, 개별 라인을 처리하는 동안 프레임워크는 커밋 지점에서 주기적으로 `ExecutionContext`를 영속화합니다.

이렇게 하면 실행 중 치명적인 오류가 발생하거나 정전이 발생하더라도 `ItemReader`가 상태를 저장할 수 있습니다.

다음 예와 같이 읽은 현재 라인 수를 컨텍스트에 넣기만 하면 프레임워크가 나머지를 수행합니다.

```java
executionContext.putLong(getKey(LINES_READ_COUNT), reader.getPosition());
```

Job 스테레오타입 섹션의 `EndOfDay` 예제를 사용하여 하나의 Step, 즉 `loadData`가 파일을 데이터베이스에 로드한다고 가정합니다.

첫 번째 실패한 실행 후 메타데이터 테이블은 다음 예와 같습니다.

BATCH_JOB_INSTANCE

| JOB_INST_ID | JOB_NAME |
| --- | --- |
| 1 | EndOfDayJob |

BATCH_JOB_EXECUTION_PARAMS

| JOB_INST_ID | TYPE_CD | KEY_NAME | DATE_VAL |
| --- | --- | --- | --- |
| 1 | DATE | schedule.Date | 2017-01-01 |

BATCH_JOB_EXECUTION

| JOB_EXEC_ID | JOB_INST_ID | START_TIME | END_TIME | STATUS |
| --- | --- | --- | --- | --- |
| 1 | 1 | 2017-01-01 21:00 | 2017-01-01 21:30 | FAILED |

BATCH_STEP_EXECUTION

| STEP_EXEC_ID | JOB_EXEC_ID | STEP_NAME | START_TIME | END_TIME | STATUS |
| --- | --- | --- | --- | --- | --- |
| 1 | 1 | loadData | 2017-01-01 21:00 | 2017-01-01 21:30 | FAILED |

BATCH_STEP_EXECUTION_CONTEXT

| STEP_EXEC_ID | SHORT_CONTEXT |
| --- | --- |
| 1 | {piece.count=40321} |

앞의 경우, `Step`은 30분 동안 실행되었고 40,321개의 "조각(pieces)"을 처리했으며, 이 시나리오에서는 파일의 라인을 나타냅니다.

이 값은 프레임워크에 의해 각 커밋 직전에 업데이트되며 `ExecutionContext` 내의 항목에 해당하는 여러 행을 포함할 수 있습니다.

커밋 전에 알림을 받으려면 다양한 `StepListener` 구현(또는 `ItemStream`) 중 하나가 필요하며, 이 가이드의 뒷부분에서 더 자세히 설명합니다.

이전 예와 마찬가지로 `Job`이 다음 날 재시작된다고 가정합니다.

재시작되면 마지막 실행의 `ExecutionContext` 값이 데이터베이스에서 재구성됩니다.

`ItemReader`가 열리면 컨텍스트에 저장된 상태가 있는지 확인하고 다음 예와 같이 거기에서 자신을 초기화할 수 있습니다.

```java
if (executionContext.containsKey(getKey(LINES_READ_COUNT))) {
    log.debug("Initializing for restart. Restart data is: " + executionContext);

    long lineCount = executionContext.getLong(getKey(LINES_READ_COUNT));

    LineReader reader = getReader();

    Object record = "";
    while (reader.getPosition() < lineCount && record != null) {
        record = readLine();
    }
}
```

이 경우 위의 코드가 실행된 후 현재 라인은 40,322이므로 `Step`이 중단된 지점부터 다시 시작할 수 있습니다.

실행 자체에 대해 영속화해야 하는 통계에 대해 `ExecutionContext`를 사용할 수도 있습니다.

예를 들어, 플랫 파일에 여러 라인에 걸쳐 존재하는 처리 대상 주문이 포함된 경우, 처리된 주문 수를 저장해야 할 수 있습니다(읽은 라인 수와는 매우 다름).

그래야 `Step` 종료 시 본문에 처리된 총 주문 수가 포함된 이메일을 보낼 수 있습니다.

프레임워크는 개발자를 위해 이를 저장하여 개별 `JobInstance` 범위에 올바르게 지정합니다.

기존 `ExecutionContext`를 사용해야 하는지 여부를 알기가 매우 어려울 수 있습니다.

예를 들어, 위의 `EndOfDay` 예제를 사용할 때 01-01 실행이 두 번째로 다시 시작되면 프레임워크는 동일한 `JobInstance`임을 인식하고 개별 `Step` 기준으로

데이터베이스에서 `ExecutionContext`를 가져와 `Step` 자체에 (`StepExecution`의 일부로) 전달합니다.

반대로 01-02 실행의 경우 프레임워크는 다른 인스턴스임을 인식하므로 빈 컨텍스트를 `Step`에 전달해야 합니다.

개발자를 위해 프레임워크가 수행하는 이러한 유형의 결정이 많으므로 올바른 시점에 상태가 제공되도록 보장합니다.

또한 주어진 시간에 `StepExecution`당 정확히 하나의 `ExecutionContext`가 존재한다는 점에 유의하는 것이 중요합니다.

`ExecutionContext`의 클라이언트는 공유 키 공간을 생성하므로 주의해야 합니다.

결과적으로 값을 넣을 때 데이터가 덮어쓰이지 않도록 주의해야 합니다.

그러나 `Step`은 컨텍스트에 데이터를 전혀 저장하지 않으므로 프레임워크에 부정적인 영향을 미칠 방법이 없습니다.

`JobExecution`당 최소 하나의 `ExecutionContext`가 있고 모든 `StepExecution`에 대해 하나씩 있다는 점에 유의하십시오.

```java
ExecutionContext ecStep = stepExecution.getExecutionContext();
ExecutionContext ecJob = jobExecution.getExecutionContext();
//ecStep은 ecJob과 같지 않음
```

주석에서 언급했듯이 `ecStep`은 `ecJob`과 같지 않습니다.

이들은 두 개의 다른 `ExecutionContext`입니다.

`Step` 범위의 컨텍스트는 `Step`의 모든 커밋 지점에서 저장되는 반면, `Job` 범위의 컨텍스트는 모든 `Step` 실행 사이에 저장됩니다.

`ExecutionContext`에서 모든 비-일시적(non-transient) 항목은 직렬화 가능(Serializable)해야 합니다.

실행 컨텍스트의 적절한 직렬화는 Step 및 Job의 재시작 기능을 뒷받침합니다.

기본적으로 직렬화할 수 없는 키 또는 값을 사용하는 경우 맞춤형 직렬화 접근 방식을 사용해야 합니다.

실행 컨텍스트를 직렬화하지 못하면 상태 영속화 프로세스가 위태로워져 실패한 작업을 제대로 복구할 수 없게 될 수 있습니다.

**JobRepository**

`JobRepository`는 앞서 언급한 모든 스테레오타입에 대한 영속성 메커니즘입니다.

`JobLauncher`, `Job`, `Step` 구현에 대한 CRUD(Create, Read, Update, Delete) 작업을 제공합니다.

`Job`이 처음 시작되면 리포지토리에서 `JobExecution`을 가져옵니다.

또한 실행 과정에서 `StepExecution` 및 `JobExecution` 구현은 리포지토리에 전달되어 영속화됩니다.

Java 구성을 사용할 때 `@EnableBatchProcessing` 어노테이션은 자동으로 구성되는 구성 요소 중 하나로 `JobRepository`를 제공합니다.

**JobLauncher**

`JobLauncher`는 다음 예와 같이 지정된 `JobParameters` 세트로 `Job`을 시작하기 위한 간단한 인터페이스를 나타냅니다.

```java
public interface JobLauncher {

public JobExecution run(Job job, JobParameters jobParameters)
            throws JobExecutionAlreadyRunningException, JobRestartException,
                   JobInstanceAlreadyCompleteException, JobParametersInvalidException;
}
```

구현은 `JobRepository`에서 유효한 `JobExecution`을 가져와 `Job`을 실행할 것으로 예상됩니다.

**ItemReader**

`ItemReader`는 `Step`에 대한 입력을 한 번에 하나씩 검색하는 것을 나타내는 추상화입니다.

`ItemReader`가 제공할 수 있는 항목을 모두 소진하면 `null`을 반환하여 이를 나타냅니다.

**ItemWriter**

`ItemWriter`는 `Step`의 출력을 한 번에 항목의 배치 또는 청크(chunk) 단위로 나타내는 추상화입니다.

일반적으로 `ItemWriter`는 다음에 받아야 할 입력에 대해 알지 못하며 현재 호출에서 전달된 항목만 알고 있습니다.

**ItemProcessor**

`ItemProcessor`는 항목의 비즈니스 처리를 나타내는 추상화입니다.

`ItemReader`는 하나의 항목을 읽고, `ItemWriter`는 하나의 항목을 쓰지만, `ItemProcessor`는 변환하거나 다른 비즈니스 처리를 적용할 수 있는 액세스 포인트를 제공합니다.

항목을 처리하는 동안 해당 항목이 유효하지 않다고 판단되면 `null`을 반환하면 해당 항목을 쓰지 않아야 함을 나타냅니다.

`ItemProcessor` 인터페이스에 대한 자세한 내용은 Readers And Writers에서 찾을 수 있습니다.

**배치 네임스페이스**

앞서 나열된 많은 도메인 개념은 Spring `ApplicationContext`에서 구성해야 합니다.

표준 빈 정의에서 사용할 수 있는 위의 인터페이스 구현이 있지만, 다음 예와 같이 구성을 쉽게 하기 위해 네임스페이스가 제공되었습니다.

```xml
<beans:beans xmlns="<http://www.springframework.org/schema/batch>"
xmlns:beans="<http://www.springframework.org/schema/beans>"
xmlns:xsi="<http://www.w3.org/2001/XMLSchema-instance>"
xsi:schemaLocation="
   <http://www.springframework.org/schema/beans>
   <https://www.springframework.org/schema/beans/spring-beans.xsd>
   <http://www.springframework.org/schema/batch>
   <https://www.springframework.org/schema/batch/spring-batch.xsd>">

<job id="ioSampleJob">
    <step id="step1">
        <tasklet>
            <chunk reader="itemReader" writer="itemWriter" commit-interval="2"/>
        </tasklet>
    </step>
</job>

</beans:beans>
```

배치 네임스페이스가 선언되어 있는 한 모든 요소를 사용할 수 있습니다.

`Job` 구성에 대한 자세한 정보는 Configuring and Running a Job에서 찾을 수 있습니다.

`Step` 구성에 대한 자세한 정보는 Configuring a Step에서 찾을 수 있습니다.

---

## Spring Batch의 언어 배우기: 핵심 용어 정복! 🗣️🧱

우리가 어떤 분야를 배울 때 그 분야에서만 쓰는 특별한 단어들이 있잖아요? Spring Batch도 마찬가지예요. 이 용어들을 알면 Spring Batch가 어떻게 돌아가는지 훨씬 쉽게 이해할 수 있답니다.

---

### 1. Job: "오늘 할 일 묶음!" 📦

- **개념:** `Job`은 우리가 실행하고 싶은 **하나의 완전한 배치 작업 전체**를 의미해요. 여러 개의 작은 단계(Step)들이 모여서 하나의 `Job`을 이루죠.
- **특징:**
  - 이름을 가져요 (예: "월말정산작업", "고객데이터백업작업").
  - 어떤 `Step`들을 어떤 순서로 실행할지 정의해요.
  - 작업이 실패했을 때 다시 시작할 수 있는지(restartable) 같은 전체적인 설정을 할 수 있어요.
- **설정 방법:** XML 파일이나 Java 코드로 "이 `Job`은 이런 `Step`들로 구성되어 있고, 이런 특징을 가져요" 라고 알려줘야 해요.

    ```java
    // Java 코드로 Job 설정 예시
    @Bean
    public Job footballJob(JobRepository jobRepository, Step playerLoad, Step gameLoad, Step playerSummarization) {
        return new JobBuilder("footballJob", jobRepository) // "footballJob"이라는 이름의 Job을 만들겠다!
                         .start(playerLoad)         // 첫 번째 Step은 playerLoad
                         .next(gameLoad)            // 다음 Step은 gameLoad
                         .next(playerSummarization) // 그 다음 Step은 playerSummarization
                         .build();                  // 자, Job 완성!
    }
    ```

- **핵심:** `Job`은 여러 `Step`들을 담는 **가장 큰 바구니**라고 생각하세요!

---

### 2. JobInstance: "오늘 할 일 중, 오늘치!" (논리적 실행 단위) 🗓️

- **개념:** `JobInstance`는 **논리적으로 구분되는 한 번의 Job 실행**을 의미해요.
- **예시:**
  - 매일 밤 실행되는 "일일정산"이라는 `Job`이 있다고 해봐요.
  - `Job` 자체는 "일일정산" 하나지만, 1월 1일에 실행된 "일일정산", 1월 2일에 실행된 "일일정산"은 서로 다른 `JobInstance`가 되는 거예요.
  - 만약 1월 1일 "일일정산"(`JobInstance`)이 실패해서 다음 날 다시 실행해도, 그건 여전히 1월 1일의 "일일정산"(`JobInstance`)으로 취급돼요. 처리해야 할 데이터의 기준 날짜가 같기 때문이죠.
- **핵심:** `Job`은 설계도, `JobInstance`는 **"특정 날짜(또는 조건)의 작업"**이라고 생각하면 쉬워요. "2024년 5월 15일자 일일정산 작업" 처럼요.

---

### 3. JobParameters: "오늘 할 일에 필요한 준비물!" (식별 정보) 🏷️

- **개념:** `JobParameters`는 `Job`을 시작할 때 전달하는 **여러 가지 매개변수(파라미터)들의 묶음**이에요.
- **역할:**
  - **`JobInstance`를 구분하는 열쇠! 🔑:** "1월 1일자 작업"과 "1월 2일자 작업"을 어떻게 구분할까요? 바로 `JobParameters`에 날짜 정보(예: `run.date=2024-01-01`)를 넣어서 구분해요.
  - 작업 실행 중에 필요한 참고 데이터로도 사용될 수 있어요.
- **공식:** **`JobInstance = Job + (식별용) JobParameters`**
  - 같은 `Job`이라도 `JobParameters`가 다르면 서로 다른 `JobInstance`가 돼요.
- **주의!** 모든 `JobParameters`가 `JobInstance`를 식별하는 데 사용되는 건 아니에요. 어떤 파라미터를 식별용으로 쓸지는 설정할 수 있어요.
- **핵심:** `JobParameters`는 **"이 `JobInstance`는 어떤 조건으로 실행되는 작업이다"**를 알려주는 이름표 같은 거예요.

---

### 4. JobExecution: "오늘 할 일, 첫 번째 시도!" (물리적 실행 단위) 🏃‍♂️

- **개념:** `JobExecution`은 `JobInstance`를 **실제로 한 번 실행하려고 시도한 기록**이에요.
- **예시:**
  - "1월 1일자 일일정산"(`JobInstance`)을 처음 실행했어요. 이게 하나의 `JobExecution`이에요.
  - 만약 이 실행이 실패해서, 같은 "1월 1일자 일일정산"(`JobInstance`)을 다시 실행하면, 새로운 `JobExecution`이 또 생겨요.
  - 즉, 하나의 `JobInstance`는 여러 번의 `JobExecution`을 가질 수 있어요 (실패 후 재시도 등).
- **`JobInstance`와의 관계:** `JobInstance`는 "이 작업은 성공해야 해!"라는 목표 같은 거고, `JobExecution`은 그 목표를 달성하기 위한 각각의 '시도'들이에요. `JobInstance`는 마지막 `JobExecution`이 성공해야 완료된 것으로 간주돼요.
- **저장되는 정보 (중요!):**
  - **Status:** 현재 실행 상태 (STARTED, COMPLETED, FAILED 등)
  - **startTime, endTime:** 시작 시간, 종료 시간
  - **exitStatus:** 최종 실행 결과 코드 (성공, 실패 등)
  - **createTime, lastUpdated:** 생성 시간, 마지막 업데이트 시간
  - **executionContext:** 실행 중 필요한 데이터를 임시로 저장하는 공간 (나중에 자세히!)
  - **failureExceptions:** 실패했을 때 발생한 오류 정보
- **핵심:** `JobExecution`은 **`JobInstance`의 실제 '시도' 한 번 한 번에 대한 모든 기록**을 담고 있어요.

**예시 시나리오 (표로 보는 `Job`, `JobInstance`, `JobExecution`):**

"EndOfDayJob"이라는 `Job`이 있다고 가정해봅시다.

1. **2017년 1월 1일, 오후 9시에 EndOfDayJob 실행 (JobParameters: `schedule.Date=2017-01-01`)**
  - `JobInstance` ID: 1 (EndOfDayJob + 2017-01-01)
  - `JobExecution` ID: 1 (JobInstance 1의 첫 번째 시도)
  - **결과:** 오후 9시 30분에 실패 (STATUS: FAILED)
2. **2017년 1월 2일, 오후 9시에 어제 실패한 EndOfDayJob 재시작 (JobParameters: `schedule.Date=2017-01-01` - 동일)**
  - `JobInstance` ID: 1 (여전히 동일한 논리적 작업)
  - `JobExecution` ID: 2 (JobInstance 1의 두 번째 시도)
  - **결과:** 오후 9시 30분에 성공 (STATUS: COMPLETED)
3. **2017년 1월 2일, 오후 9시 31분에 오늘의 EndOfDayJob 실행 (JobParameters: `schedule.Date=2017-01-02`)**
  - `JobInstance` ID: 2 (EndOfDayJob + 2017-01-02 - 새로운 논리적 작업)
  - `JobExecution` ID: 3 (JobInstance 2의 첫 번째 시도)
  - **결과:** 오후 10시 29분에 성공 (STATUS: COMPLETED)

이 모든 정보(JobInstance, JobExecution, JobParameters)는 `JobRepository`라는 곳에 저장돼요!

---

### 5. Step: "오늘 할 일 중, 작은 단위 업무!" 📝

- **개념:** `Step`은 `Job`을 구성하는 **독립적이고 순서대로 진행되는 하나의 단계**예요. 모든 `Job`은 하나 이상의 `Step`으로 이루어져 있죠.
- **역할:** 실제 배치 처리 로직(데이터를 읽고, 가공하고, 쓰는 등)을 정의하고 제어하는 모든 정보를 담고 있어요.
- **복잡도:** `Step`은 아주 간단할 수도 (예: 파일에서 DB로 단순 복사) 있고, 매우 복잡할 수도 (예: 복잡한 비즈니스 규칙 적용) 있어요. 개발자가 만들기 나름!
- **`Job`과의 관계:** `Job`은 `Step`들을 모아놓은 큰 틀이고, `Step`은 그 안에서 실제로 일을 하는 작은 단위들이에요.
- **핵심:** `Step`은 **`Job`의 실질적인 작업 단위**예요.

---

### 6. StepExecution: "작은 단위 업무, 첫 번째 시도!" 🏃‍♀️

- **개념:** `StepExecution`은 `Step`을 **실제로 한 번 실행하려고 시도한 기록**이에요. `JobExecution`과 비슷하죠?
- **생성 시점:** `Step`이 실제로 시작될 때만 `StepExecution`이 만들어져요. 만약 이전 `Step`이 실패해서 현재 `Step`이 아예 시작도 못 했다면, 현재 `Step`의 `StepExecution`은 생성되지 않아요.
- **저장되는 정보 (중요!):**
  - `JobExecution`의 정보와 유사 (Status, startTime, endTime, exitStatus)
  - **executionContext:** `Step` 실행 중 필요한 데이터를 임시로 저장하는 공간. (Job의 executionContext와는 별개!)
  - **readCount, writeCount:** 성공적으로 읽거나 쓴 아이템 개수.
  - **commitCount, rollbackCount:** 트랜잭션 커밋/롤백 횟수.
  - **skipCount (read, process, write):** 읽기/처리/쓰기 단계에서 건너뛴(skip) 아이템 개수.
  - **filterCount:** `ItemProcessor`에서 걸러진 아이템 개수.
- **핵심:** `StepExecution`은 **`Step`의 실제 '시도' 한 번 한 번에 대한 모든 기록**을 담고 있어요.

---

### 7. ExecutionContext: "작업 중 필요한 메모장!" 🗒️

- **개념:** `ExecutionContext`는 프레임워크가 관리하는 **키-값(key-value) 쌍으로 된 데이터 저장 공간**이에요. 개발자는 여기에 배치 실행 중에 유지해야 할 상태 정보(예: 어디까지 읽었는지, 통계 정보 등)를 저장할 수 있어요.
- **범위 (Scope):**
  - **`JobExecution`의 `ExecutionContext`:** `Job` 전체 범위에서 공유. `Step` 실행 사이에 저장돼요.
  - **`StepExecution`의 `ExecutionContext`:** 해당 `Step` 범위에서만 유효. `Step` 내의 커밋 시점마다 주기적으로 저장돼요.
  - **주의!** 이 둘은 서로 다른 `ExecutionContext`예요! (`ecJob != ecStep`)
  - **주요 용도: 재시작!**

```java
// ItemReader가 재시작 시 ExecutionContext에서 정보 가져오는 예시
if (executionContext.containsKey("lines.read.count")) {
long linesRead = executionContext.getLong("lines.read.count");
// linesRead 지점부터 다시 읽도록 처리...
}
```

- 파일을 읽다가 중간에 실패했다고 해봐요. `ExecutionContext`에 "마지막으로 40321번째 줄까지 읽었어" 라고 저장해두면, 다음에 재시작할 때 이 정보를 보고 40322번째 줄부터 다시 시작할 수 있어요!
- **저장 시 주의!** `ExecutionContext`에 저장하는 데이터는 **직렬화 가능(Serializable)**해야 해요. 그래야 DB에 제대로 저장하고 불러올 수 있어요.
- **핵심:** `ExecutionContext`는 **작업의 상태를 저장하여 재시작을 돕거나, 통계 정보를 공유하는 데 사용되는 '임시 저장소' 또는 '메모장'** 같은 거예요.

---

### 8. JobRepository: "모든 작업 기록 보관소!" 🗄️

- **개념:** `JobRepository`는 위에서 설명한 모든 배치 관련 정보들(`JobInstance`, `JobExecution`, `StepExecution`, `ExecutionContext` 등)을 **데이터베이스에 저장하고 관리하는 역할**을 해요.
- **주요 기능:**
  - 새로운 `JobExecution`을 생성하고 가져오기.
  - `JobExecution`과 `StepExecution`의 상태를 실행 중에 계속 업데이트(저장)하기.
  - `JobLauncher`, `Job`, `Step` 구현체들이 필요로 하는 CRUD(Create, Read, Update, Delete) 작업 제공.
- **설정:** Java 설정에서는 `@EnableBatchProcessing` 어노테이션을 사용하면 자동으로 `JobRepository`가 설정돼요.
- **핵심:** `JobRepository`는 **Spring Batch의 모든 실행 기록과 메타데이터를 보관하는 '데이터베이스 창고'**예요. 이게 있어야 재시작도 가능하고, 작업 상태도 추적할 수 있어요.

---

### 9. JobLauncher: "작업 시작 스위치!" 🚀

- **개념:** `JobLauncher`는 `Job`을 `JobParameters`와 함께 **실행시키는 간단한 인터페이스**예요.

    ```java
    public interface JobLauncher {
        public JobExecution run(Job job, JobParameters jobParameters) throws ... ;
    }
    ```

- **역할:** `JobRepository`에서 유효한 `JobExecution`을 얻어오고, 실제로 `Job`을 실행해요.
- **핵심:** `JobLauncher`는 우리가 만든 `Job`에 **"시작!" 명령을 내리는 '실행 버튼'** 같은 거예요.

---

### 10. ItemReader, ItemWriter, ItemProcessor: "데이터 처리 삼총사!" 🧑‍🍳📖✍️

이 셋은 `Step` 안에서 실제 데이터 처리를 담당하는 핵심 인터페이스들이에요. (보통 "Chunk 기반 처리"라는 방식에서 함께 사용돼요.)

- **`ItemReader`: "데이터 읽어오기 담당!" 📖**
  - `Step`에 필요한 입력 데이터를 **한 번에 하나씩** 읽어와요.
  - 더 이상 읽을 데이터가 없으면 `null`을 반환해서 알려줘요.
  - (예: 파일에서 한 줄씩 읽기, DB에서 한 로우(row)씩 읽기)
- **`ItemProcessor`: "데이터 가공/변환 담당!" 🧑‍🍳**
  - `ItemReader`가 읽어온 데이터를 **가공하거나 비즈니스 로직을 적용**해요.
  - 만약 처리 중에 이 데이터가 유효하지 않다고 판단되면, `null`을 반환해서 "이건 쓰지 마!" 라고 알려줄 수 있어요 (필터링).
  - (예: 고객 데이터를 읽어서 구매 금액에 따라 등급 부여하기, 날짜 형식 변환하기)
- **`ItemWriter`: "데이터 쓰기/저장 담당!" ✍️**
  - `ItemProcessor`가 처리한 (또는 `ItemReader`가 바로 넘겨준) 데이터를 **한 번에 묶음(chunk) 단위로** 저장해요.
  - (예: 처리된 고객 데이터를 DB에 여러 건씩 저장하기, 결과를 파일에 쓰기)
- **흐름:** **Read (ItemReader) -> Process (ItemProcessor) -> Write (ItemWriter)**
- **핵심:** 이 삼총사는 **`Step` 내에서 실제 데이터가 흘러가며 처리되는 과정을 담당**해요.

---

### ➕ 배치 네임스페이스 (XML 설정 시)

XML로 Spring Batch를 설정할 때는 `<batch:job>`, `<batch:step>`처럼 `batch:`라는 접두사가 붙은 특별한 태그들을 사용해요. 이걸 "배치 네임스페이스"라고 부르는데, 복잡한 빈(bean) 설정을 더 간편하게 할 수 있도록 도와줘요.

```xml
<job id="ioSampleJob"> <!-- Job 정의 -->
    <step id="step1">   <!-- Step 정의 -->
        <tasklet>
            <chunk reader="itemReader" writer="itemWriter" commit-interval="2"/> <!-- 청크 기반 처리 설정 -->
        </tasklet>
    </step>
</job>
```

---

### 1. 네임스페이스를 선언하면 어떤 동작을 하는 지?

Spring 설정 파일(주로 XML)에서 **네임스페이스(namespace)**를 선언한다는 것은, Spring 프레임워크에게 **"이제부터 이 XML 파일 안에서 특정 스키마(규칙 모음)에 정의된 특별한 XML 태그들을 사용할 거야!"**라고 알려주는 것과 같아요.

**Spring Batch의 `batch:` 네임스페이스를 예로 들어볼게요.**

```xml
<beans:beans xmlns="<http://www.springframework.org/schema/batch>"  <!-- 기본 네임스페이스를 batch로! -->
             xmlns:beans="<http://www.springframework.org/schema/beans>"
             xmlns:xsi="<http://www.w3.org/2001/XMLSchema-instance>"
             xsi:schemaLocation="
                <http://www.springframework.org/schema/beans>
                <https://www.springframework.org/schema/beans/spring-beans.xsd>
                <http://www.springframework.org/schema/batch>  <!-- batch 네임스페이스 선언 -->
                <https://www.springframework.org/schema/batch/spring-batch.xsd>"> <!-- batch 스키마 위치 -->

    <job id="ioSampleJob">
        <step id="step1">
            <tasklet>
                <chunk reader="itemReader" writer="itemWriter" commit-interval="2"/>
            </tasklet>
        </step>
    </job>

</beans:beans>

```

여기서 `xmlns="<http://www.springframework.org/schema/batch>"` 나 `xmlns:batch="<http://www.springframework.org/schema/batch>"` (접두사를 붙일 경우) 같은 선언이 바로 네임스페이스 선언이에요.

**네임스페이스를 선언하면 다음과 같은 동작이 일어납니다:**

1. **특별한 태그 사용 가능:**
  - `batch:` 네임스페이스를 선언하면, Spring Batch가 제공하는 `<job>`, `<step>`, `<chunk>`, `<listener>` 같은 **간결하고 직관적인 XML 태그**들을 사용할 수 있게 됩니다.
  - 만약 이 네임스페이스를 선언하지 않으면, Spring은 이 태그들을 이해하지 못하고 오류를 발생시킵니다.
2. **XML 파싱 및 빈(Bean) 등록 자동화:**
  - Spring 컨테이너는 XML 파일을 읽어들일 때(파싱할 때), 선언된 네임스페이스와 `xsi:schemaLocation`에 명시된 스키마 파일(.xsd)을 참조합니다.
  - 이 스키마 파일에는 `<job>` 태그가 어떤 속성을 가질 수 있고, 그 속성값이 어떤 의미인지 등이 정의되어 있습니다.
  - Spring은 이 정보를 바탕으로, 예를 들어 `<job id="myJob" ...>` 태그를 만나면, 내부적으로 `Job` 인터페이스를 구현하는 적절한 빈(Bean) 객체(예: `SimpleJob`)를 생성하고, `id` 속성값을 빈의 이름으로 설정하는 등 **복잡한 자바 객체 생성 및 설정을 자동으로 처리**해줍니다.
  - 만약 네임스페이스를 사용하지 않고 직접 `<bean class="org.springframework.batch.core.job.SimpleJob" ...>` 와 같이 모든 것을 일일이 설정하려면 코드가 매우 길고 복잡해질 것입니다.
3. **가독성 및 유지보수성 향상:**
  - 네임스페이스를 사용하면 XML 설정이 훨씬 **간결하고 의미가 명확**해집니다.
  - "아, 이건 Spring Batch의 Job을 설정하는 부분이구나!" 하고 바로 알 수 있게 되어 코드 가독성이 높아지고 유지보수도 쉬워집니다.

**요약하자면,** 네임스페이스 선언은 Spring에게 "이 XML 문서에서 사용할 특별한 문법(태그)이 있는데, 그 문법은 이 스키마에 정의되어 있으니 참고해서 알아서 잘 처리해줘!" 라고 말하는 것과 같습니다. 이를 통해 개발자는 복잡한 내부 구현을 몰라도, 간편한 태그를 사용하여 Spring Batch의 구성 요소들을 쉽게 설정할 수 있게 되는 것입니다. 일종의 **"설정을 위한 단축키"** 또는 **"고수준의 추상화된 설정 언어"**를 제공하는 것이라고 생각할 수 있습니다.

---

### 2. 직렬화가 가능해야 하는 이유가 뭔지?

- *직렬화(Serialization)**란 객체를 바이트 스트림(byte stream) 형태로 변환하는 과정을 말해요. 이렇게 변환된 바이트 스트림은 파일에 저장하거나, 네트워크를 통해 다른 시스템으로 전송하거나, 데이터베이스에 저장하는 등의 작업을 할 수 있게 됩니다. 반대로 바이트 스트림을 다시 객체로 복원하는 과정을 **역직렬화(Deserialization)**라고 합니다.

Spring Batch에서, 특히 `ExecutionContext`에 저장되는 객체들이 **직렬화 가능(Serializable 인터페이스 구현)**해야 하는 주된 이유는 다음과 같습니다:

1. **상태 저장 및 재시작(Restart) 기능 지원:**
  - Spring Batch의 가장 강력한 기능 중 하나는 작업 실패 시 **중단된 지점부터 다시 시작**할 수 있다는 것입니다.
  - 이를 가능하게 하려면, 작업이 실행되는 동안 중요한 상태 정보들(예: `ItemReader`가 어디까지 읽었는지, `Step`에서 처리한 건수 등)을 어딘가에 **영구적으로 저장**해두어야 합니다.
  - 이 "어딘가"가 바로 `JobRepository`를 통해 접근하는 **데이터베이스**입니다.
  - `ExecutionContext`에 저장된 객체들은 주기적으로 (예: `Step`의 커밋 시점마다) 데이터베이스에 저장됩니다. 객체를 데이터베이스에 저장하려면, 객체를 데이터베이스가 이해할 수 있는 형태, 즉 바이트 스트림(또는 그와 유사한 형태)으로 변환해야 합니다. 이때 **직렬화**가 사용됩니다.
  - 나중에 작업이 재시작될 때, 데이터베이스에서 이 바이트 스트림을 읽어와 **역직렬화**를 통해 원래 객체 상태로 복원하고, 그 상태를 기반으로 작업을 이어갈 수 있게 됩니다.
2. **데이터베이스 호환성:**
  - `ExecutionContext`는 보통 데이터베이스의 특정 컬럼(예: `SHORT_CONTEXT` 또는 `SERIALIZED_CONTEXT`라는 이름의 `VARCHAR` 또는 `BLOB` 타입 컬럼)에 저장됩니다.
  - 자바 객체를 이런 컬럼에 직접 저장할 수는 없으므로, 직렬화를 통해 바이트 배열이나 문자열 형태로 변환하여 저장하는 것입니다.
3. **분산 환경에서의 상태 공유 (고급 시나리오):**
  - 만약 배치 작업을 여러 서버에 분산하여 실행하는 경우(예: Remote Chunking, Partitioning), `ExecutionContext`에 담긴 상태 정보를 네트워크를 통해 다른 서버로 전송해야 할 수 있습니다. 이때도 객체를 전송 가능한 바이트 스트림 형태로 만들기 위해 직렬화가 필요합니다.

**만약 `ExecutionContext`에 저장하는 객체가 직렬화 가능하지 않다면?**

- Spring Batch 프레임워크는 해당 객체를 바이트 스트림으로 변환하여 데이터베이스에 저장하려고 시도할 때 오류(`NotSerializableException` 등)를 발생시킵니다.
- 결과적으로, 작업 상태가 제대로 저장되지 않아 재시작 기능이 올바르게 동작하지 않거나, 아예 작업 자체가 실패할 수 있습니다.

**따라서,** `ExecutionContext`에 저장하는 모든 키(key)와 값(value)은 `java.io.Serializable` 인터페이스를 구현해야 합니다. 만약 직접 만든 클래스를 `ExecutionContext`에 넣고 싶다면, 해당 클래스에 `implements Serializable`을 추가해주어야 합니다.

**요약하자면,** Spring Batch에서 `ExecutionContext`의 객체들이 직렬화 가능해야 하는 핵심 이유는 **작업의 상태를 데이터베이스에 안정적으로 저장하고, 이를 통해 강력한 재시작 기능을 지원하기 위함**입니다.
