---
title: Spring Batch - Configuring a Step (Restart)
description: 
author: laze
date: 2025-05-22 00:00:02 +0900
categories: [Dev, SpringBatch]
tags: [SpringBatch]
---
**재시작을 위한 Step 설정하기**

"Job 설정 및 실행" 섹션에서 Job 재시작에 대해 논의했습니다. 재시작은 Step에 수많은 영향을 미치며, 결과적으로 몇 가지 특정 구성이 필요할 수 있습니다.

**시작 제한 설정하기 (Setting a Start Limit)**

`Step`을 시작할 수 있는 횟수를 제어하고 싶은 시나리오가 많습니다.

예를 들어, 특정 `Step`이 다시 실행되기 전에 수동으로 수정해야 하는 일부 리소스를 무효화하기 때문에 한 번만 실행되도록 구성해야 할 수 있습니다.

다른 Step은 다른 요구 사항을 가질 수 있으므로 이는 Step 수준에서 구성 가능합니다.

한 번만 실행할 수 있는 `Step`은 무한정 실행할 수 있는 `Step`과 동일한 `Job`의 일부로 존재할 수 있습니다.

Java에서 시작 제한 구성의 예시

```java
@Bean
public Step step1(JobRepository jobRepository, PlatformTransactionManager transactionManager) {
	return new StepBuilder("step1", jobRepository)
				.<String, String>chunk(10, transactionManager)
				.reader(itemReader()) // itemReader(), itemWriter()는 실제 빈을 참조해야 합니다.
				.writer(itemWriter())
				.startLimit(1) // 이 Step은 최대 1번만 시작 가능
				.build();
}
```

위 예제의 Step은 한 번만 실행할 수 있습니다. 다시 실행하려고 하면 `StartLimitExceededException`이 발생합니다.

`start-limit`의 기본값은 `Integer.MAX_VALUE`입니다.

**완료된 Step 재시작하기 (Restarting a Completed Step)**

재시작 가능한 Job의 경우, 처음 성공 여부와 관계없이 항상 실행되어야 하는 하나 이상의 Step이 있을 수 있습니다.

예를 들어 유효성 검사 Step 또는 처리 전에 리소스를 정리하는 `Step`이 있을 수 있습니다.

재시작된 Job의 정상적인 처리 중에 `COMPLETED` 상태(즉, 이미 성공적으로 완료됨)인 모든 Step은 건너뜁니다.

`allow-start-if-complete`를 `true`로 설정하면 이를 재정의하여 Step이 항상 실행되도록 합니다.

다음 코드 조각은 Java에서 재시작 가능한 Job을 정의하는 방법을 보여줍니다 (정확히는, 완료된 Step도 항상 다시 실행되도록 설정하는 방법입니다).

```java
@Bean
public Step step1(JobRepository jobRepository, PlatformTransactionManager transactionManager) {
	return new StepBuilder("step1", jobRepository)
				.<String, String>chunk(10, transactionManager)
				.reader(itemReader())
				.writer(itemWriter())
				.allowStartIfComplete(true) // 이 Step은 완료되었더라도 Job 재시작 시 항상 실행
				.build();
}
```

**Step 재시작 구성 예제**

다음 Java 예제는 재시작할 수 있는 Step을 가진 Job을 구성하는 방법을 보여줍니다.

```java
@Bean
public Job footballJob(JobRepository jobRepository, Step playerLoad, Step gameLoad, Step playerSummarization) {
	return new JobBuilder("footballJob", jobRepository)
				.start(playerLoad)
				.next(gameLoad)
				.next(playerSummarization)
				.build();
}

@Bean
public Step playerLoad(JobRepository jobRepository, PlatformTransactionManager transactionManager) {
	return new StepBuilder("playerLoad", jobRepository)
			.<String, String>chunk(10, transactionManager)
			.reader(playerFileItemReader()) // 각 reader/writer는 실제 빈을 참조해야 합니다.
			.writer(playerWriter())
			.build();
}

@Bean
public Step gameLoad(JobRepository jobRepository, PlatformTransactionManager transactionManager) {
	return new StepBuilder("gameLoad", jobRepository)
			.allowStartIfComplete(true) // gameLoad는 항상 실행
			.<String, String>chunk(10, transactionManager)
			.reader(gameFileItemReader())
			.writer(gameWriter())
			.build();
}

@Bean
public Step playerSummarization(JobRepository jobRepository, PlatformTransactionManager transactionManager) {
	return new StepBuilder("playerSummarization", jobRepository)
			.startLimit(2) // playerSummarization은 최대 2번만 시작 가능
			.<String, String>chunk(10, transactionManager)
			.reader(playerSummarizationSource())
			.writer(summaryWriter())
			.build();
}
```

위 예제 구성은 축구 게임에 대한 정보를 로드하고 요약하는 Job에 대한 것입니다.

여기에는 `playerLoad`, `gameLoad`, `playerSummarization`의 세 가지 Step이 포함됩니다.

`playerLoad` Step은 플랫 파일에서 선수 정보를 로드하고, `gameLoad` Step은 게임에 대해 동일한 작업을 수행합니다.

마지막 Step인 `playerSummarization`은 제공된 게임을 기반으로 각 선수의 통계를 요약합니다.

`playerLoad`에서 로드한 파일은 한 번만 로드해야 하지만, `gameLoad`는 특정 디렉토리에서 찾은 모든 게임을 로드하고 데이터베이스에 성공적으로 로드된 후 삭제할 수 있다고 가정합니다.

결과적으로 `playerLoad` Step에는 추가 구성이 없습니다.

몇 번이든 시작할 수 있으며 완료되면 건너뜁니다.

그러나 `gameLoad` Step은 마지막 실행 이후 추가 파일이 추가되었을 경우를 대비하여 매번 실행되어야 합니다.

항상 시작되도록 `allow-start-if-complete`가 `true`로 설정되어 있습니다. (게임이 로드되는 데이터베이스 테이블에는 요약 Step이 새 게임을 올바르게 찾을 수 있도록 프로세스 표시기가 있다고 가정합니다.)

Job에서 가장 중요한 요약 Step인 `playerSummarization`은 시작 제한이 2로 구성되어 있습니다.

이는 Step이 계속 실패하는 경우 Job 실행을 제어하는 운영자에게 새 종료 코드가 반환되고 수동 개입이 있을 때까지 다시 시작할 수 없기 때문에 유용합니다.

*이 Job은 이 문서를 위한 예제이며 샘플 프로젝트에 있는 footballJob과는 다릅니다.*

이 섹션의 나머지 부분에서는 `footballJob` 예제의 세 번의 실행 각각에 대해 발생하는 상황을 설명합니다.

**실행 1:**

- `playerLoad`가 실행되고 성공적으로 완료되어 `PLAYERS` 테이블에 400명의 선수를 추가합니다.
- `gameLoad`가 실행되고 11개 파일 분량의 게임 데이터를 처리하여 해당 내용을 `GAMES` 테이블에 로드합니다.
- `playerSummarization`이 처리를 시작하고 5분 후에 실패합니다.

**실행 2:**

- `playerLoad`는 이미 성공적으로 완료되었고 `allow-start-if-complete`가 `false`(기본값)이므로 실행되지 않습니다.
- `gameLoad`가 다시 실행되고 다른 2개의 파일을 처리하여 해당 내용을 `GAMES` 테이블에도 로드합니다 (아직 처리되지 않았음을 나타내는 프로세스 표시기와 함께).
- `playerSummarization`이 나머지 모든 게임 데이터(프로세스 표시기를 사용하여 필터링) 처리를 시작하고 30분 후에 다시 실패합니다.

**실행 3:**

- `playerLoad`는 이미 성공적으로 완료되었고 `allow-start-if-complete`가 `false`(기본값)이므로 실행되지 않습니다.
- `gameLoad`가 다시 실행되고 다른 2개의 파일을 처리하여 해당 내용을 `GAMES` 테이블에도 로드합니다 (아직 처리되지 않았음을 나타내는 프로세스 표시기와 함께).
- `playerSummarization`은 `playerSummarization`의 세 번째 실행이고 제한이 2뿐이므로 시작되지 않고 Job이 즉시 중단됩니다. 제한을 늘리거나 Job을 새 `JobInstance`로 실행해야 합니다.

---

## Step 재시작, 내 마음대로 조종하기! 🕹️

`Job` 전체에 대한 재시작 가능 여부(`restartable` 속성)는 이미 배웠죠? 이번에는 그 `Job` 안에 있는 **개별 `Step`들의 재시작 동작**을 좀 더 세밀하게 제어하는 두 가지 마법 주문을 배울 거예요.

1. **`startLimit(횟수)`**: "이 `Step`은 최대 O번까지만 시작할 수 있어!"
2. **`allowStartIfComplete(true/false)`**: "이 `Step`이 이미 성공했어도, `Job` 재시작 시 또 실행할까 말까?"

이 설정들은 `StepBuilder`를 통해 Java 코드로 쉽게 설정할 수 있어요.

---

### 1. `startLimit(횟수)`: Step 시작 횟수 제한! 🚫🔢

어떤 `Step`들은 너무 자주 실행되면 곤란하거나, 특정 횟수 이상 실패하면 사람이 직접 확인해야 하는 경우가 있을 수 있어요. 이럴 때 `startLimit`을 사용해요!

- **기능:** 특정 `Step`이 시작될 수 있는 **최대 횟수를 제한**합니다.
- **설정 방법 (Java):**

    ```java
    @Bean
    public Step crucialStep(JobRepository jobRepository, PlatformTransactionManager transactionManager,
                            ItemReader<Data> reader, ItemWriter<Data> writer) {
        return new StepBuilder("crucialStep", jobRepository)
                .<Data, Data>chunk(10, transactionManager)
                .reader(reader)
                .writer(writer)
                .startLimit(3) // 이 Step은 최대 3번까지만 시작될 수 있다!
                .build();
    }
    
    ```

- **동작:**
  - 만약 `startLimit`으로 설정된 횟수를 초과하여 해당 `Step`이 다시 시작되려고 하면, Spring Batch는 **`StartLimitExceededException`*이라는 예외를 발생시키면서 `Job`을 중단시켜요. "더 이상 이 `Step`은 못 돌려! 한계 초과야!" 하는 거죠.
  - **기본값:** `Integer.MAX_VALUE` (거의 무한대). 즉, 따로 설정하지 않으면 횟수 제한 없이 `Step`을 시작할 수 있어요.
- **언제 유용할까요?**
  - **중요한 리소스를 변경하는 `Step`:** 한 번 실행되면 되돌리기 어려운 작업을 하는 `Step`의 경우, 실수로 여러 번 실행되는 것을 막고 싶을 때. (예: `startLimit(1)`)
  - **반복적으로 실패하는 `Step`:** 특정 `Step`이 계속 실패한다면, 무한정 재시도하는 것보다 몇 번만 시도하고 관리자에게 알리는 것이 나을 수 있어요. `startLimit`을 넘어서면 `Job`이 중단되므로, 운영자가 문제를 파악하고 조치할 시간을 벌 수 있죠.

---

### 2. `allowStartIfComplete(true)`: 성공한 Step도 다시 실행! ✅➡️🔄

`Job`이 재시작될 때, 기본적으로 **이미 성공적으로 완료된 (`COMPLETED` 상태인) `Step`들은 건너뛰어요.** "이미 끝난 일인데 또 할 필요 없잖아?" 하는 거죠. 이게 대부분의 경우에 합리적이에요.

하지만 때로는 **이미 성공한 `Step`이라도 `Job`이 재시작될 때마다 항상 다시 실행**하고 싶을 때가 있어요. 이럴 때 `allowStartIfComplete(true)`를 사용해요!

- **기능:** `Job` 재시작 시, 해당 `Step`이 이전에 `COMPLETED` 상태였더라도 **무조건 다시 실행**하도록 설정합니다.
- **설정 방법 (Java):**

    ```java
    @Bean
    public Step cleanupStep(JobRepository jobRepository, PlatformTransactionManager transactionManager,
                            Tasklet cleanupTasklet) {
        return new StepBuilder("cleanupStep", jobRepository)
                .tasklet(cleanupTasklet, transactionManager)
                .allowStartIfComplete(true) // 이 Step은 Job 재시작 시 항상 다시 실행!
                .build();
    }
    ```

- **동작:**
  - `allowStartIfComplete(true)`로 설정된 `Step`은, `Job`이 재시작될 때 이전 실행 상태가 `COMPLETED`였는지 여부와 상관없이 항상 실행됩니다.
  - **기본값:** `false`. 즉, 따로 설정하지 않으면 완료된 `Step`은 재시작 시 건너뜁니다.
- **언제 유용할까요?**
  - **리소스 정리/준비 `Step`:** 매번 `Job` 실행 전에 특정 임시 파일을 삭제하거나, 필요한 디렉토리를 만들어야 하는 `Step`.
  - **데이터 유효성 검사 `Step`:** 항상 최신 상태의 데이터로 유효성 검사를 다시 수행해야 하는 `Step`.
  - **외부 상태에 의존하는 `Step`:** `Step` 외부의 어떤 조건(예: 특정 파일의 존재 유무, 새로운 데이터 도착 여부)에 따라 매번 다른 작업을 수행해야 하는 `Step`. (예제에서 `gameLoad` Step처럼 새로운 게임 파일이 추가되었는지 매번 확인하고 로드하는 경우)

---

### 시나리오 예제: `footballJob` 파헤치기! ⚽📊

문서에 나온 `footballJob` 예제를 통해 이 설정들이 실제 어떻게 동작하는지 살펴봅시다.

**Job 구성:**

- `playerLoad` (Step 1): 선수 정보 로드 (기본 설정: `startLimit=무한대`, `allowStartIfComplete=false`)
- `gameLoad` (Step 2): 게임 정보 로드 (`allowStartIfComplete=true` 설정)
- `playerSummarization` (Step 3): 선수 통계 요약 (`startLimit=2` 설정)

**실행 시나리오:**

1. **첫 번째 실행:**
  - `playerLoad`: 실행 후 성공 (400명 선수 로드)
  - `gameLoad`: 실행 후 성공 (11개 게임 파일 로드)
  - `playerSummarization`: 실행 중 **실패** (5분 후)
  - ➡️ `JobExecution` 상태: `FAILED`
2. **두 번째 실행 (Job 재시작):**
  - `playerLoad`: **건너뜀** (이전에 성공했고, `allowStartIfComplete=false` 이므로)
  - `gameLoad`: **다시 실행** (`allowStartIfComplete=true` 이므로). 새로운 게임 파일 2개 추가 로드.
  - `playerSummarization`: **다시 실행** (첫 번째 실패 후 재시도). 하지만 또 **실패** (30분 후).
  - ➡️ `JobExecution` 상태: `FAILED`
  - ➡️ `playerSummarization` Step의 시작 횟수: **2회** (첫 번째 실행 + 두 번째 실행)
3. **세 번째 실행 (Job 재시작):**
  - `playerLoad`: **건너뜀**
  - `gameLoad`: **다시 실행**. 새로운 게임 파일 2개 추가 로드.
  - `playerSummarization`: **시작조차 못함!** 왜? `startLimit`이 `2`로 설정되어 있는데, 이미 두 번 시작했기 때문이죠. `StartLimitExceededException`이 발생하면서 `Job`이 즉시 중단돼요.
  - ➡️ `JobExecution` 상태: `FAILED` (또는 `STOPPED` - 예외 종류에 따라 다름)

**이 예제가 보여주는 것:**

- `allowStartIfComplete=true`는 `Job`이 재시작될 때마다 특정 `Step`을 항상 실행시키고 싶을 때 유용해요 (예: 새로운 입력 파일 확인).
- `startLimit`은 특정 `Step`의 무한 재시도를 막고, 문제가 계속될 경우 운영자의 개입을 유도하는 데 효과적이에요.
- 이런 `Step` 레벨의 재시작 설정을 통해 `Job` 전체의 안정성과 관리 용이성을 높일 수 있어요!

---
