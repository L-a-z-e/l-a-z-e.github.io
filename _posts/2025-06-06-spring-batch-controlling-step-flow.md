---
title: Spring Batch - Configuring a Step (TaskletStep)
description: 
author: laze
date: 2025-06-06 00:00:03 +0900
categories: [Dev, SpringBatch]
tags: [SpringBatch]
---
### 스텝 플로우 제어하기 (Controlling Step Flow)

여러 스텝을 하나의 잡(Job) 내에 그룹화할 수 있게 되면서, 한 스텝에서 다른 스텝으로 잡(Job)이 "흘러가는" 방식을 제어할 필요성이 생겼습니다.

스텝(Step)의 실패가 반드시 잡(Job)의 실패를 의미하는 것은 아닙니다. 더욱이, 다음에 어떤 스텝(Step)이 실행되어야 할지를 결정하는 "성공"에는 여러 유형이 있을 수 있습니다.

스텝(Step) 그룹이 어떻게 구성되느냐에 따라, 특정 스텝들은 전혀 처리되지 않을 수도 있습니다.

### 플로우 정의에서 스텝 빈 메서드 프록시 (Step bean method proxying in flow definitions)

스텝 인스턴스는 플로우 정의 내에서 유일해야 합니다.

스텝이 플로우 정의에서 여러 결과를 가질 때, 동일한 스텝 인스턴스가 플로우 정의 메서드(start, from 등)에 전달되는 것이 중요합니다.

그렇지 않으면 플로우 실행이 예기치 않게 동작할 수 있습니다.

다음 예제들에서, 스텝들은 플로우 또는 잡 빈 정의 메서드에 매개변수로 주입됩니다.

이러한 의존성 주입 스타일은 플로우 정의에서 스텝의 유일성을 보장합니다.

그러나 만약 `@Bean`으로 어노테이션된 스텝 정의 메서드를 호출하여 플로우가 정의된다면, 빈 메서드 프록시가 비활성화된 경우(예: `@Configuration(proxyBeanMethods = false)`) 스텝들이 유일하지 않을 수 있습니다.

빈 간 주입 스타일을 선호한다면, 빈 메서드 프록시가 활성화되어야 합니다.

Spring 프레임워크에서의 빈 메서드 프록시에 대한 자세한 내용은 [Using the @Configuration annotation](https://docs.spring.io/spring-framework/docs/current/reference/html/core.html#beans-java-configuration-bean-proxying) 섹션을 참조하십시오.

### 순차적 플로우 (Sequential Flow)

가장 간단한 플로우 시나리오는 모든 스텝이 순차적으로 실행되는 잡(Job)이며, 다음 이미지와 같습니다:

**순차적 플로우**

이는 스텝에서 `next`를 사용하여 달성할 수 있습니다.

다음 예제는 Java에서 `next()` 메서드를 사용하는 방법을 보여줍니다:

**Java 설정 (Java Configuration)**

```java
@Bean
public Job job(JobRepository jobRepository, Step stepA, Step stepB, Step stepC) {
	return new JobBuilder("job", jobRepository)
				.start(stepA)
				.next(stepB)
				.next(stepC)
				.build();
}
```

위 시나리오에서는, `stepA`가 목록의 첫 번째 스텝(Step)이므로 먼저 실행됩니다.

`stepA`가 정상적으로 완료되면 `stepB`가 실행되고, 이런 식으로 계속됩니다. 그러나 `stepA`가 실패하면, 전체 잡(Job)이 실패하고 `stepB`는 실행되지 않습니다.

Spring Batch XML 네임스페이스를 사용하면, 설정에 나열된 첫 번째 스텝이 항상 잡(Job)에 의해 실행되는 첫 번째 스텝입니다.

다른 스텝 요소들의 순서는 중요하지 않지만, 첫 번째 스텝은 항상 XML에서 가장 먼저 나타나야 합니다.

### 조건부 플로우 (Conditional Flow)

이전 예제에서는 두 가지 가능성만 있었습니다:

1. 스텝이 성공하고, 다음 스텝이 실행되어야 한다.
2. 스텝이 실패했고, 따라서 잡(Job)이 실패해야 한다.

많은 경우에 이것으로 충분할 수 있습니다.

하지만 스텝의 실패가 실패를 야기하는 대신 다른 스텝을 트리거해야 하는 시나리오는 어떨까요?

**조건부 플로우**

Java API는 플로우를 지정하고 스텝이 실패했을 때 무엇을 할지 지정할 수 있는 유연한 메서드 세트를 제공합니다.

다음 예제는 하나의 스텝(`stepA`)을 지정한 다음, `stepA`의 성공 여부에 따라 두 개의 다른 스텝(`stepB` 또는 `stepC`) 중 하나로 진행하는 방법을 보여줍니다:

**Java 설정 (Java Configuration)**

```java
@Bean
public Job job(JobRepository jobRepository, Step stepA, Step stepB, Step stepC) {
	return new JobBuilder("job", jobRepository)
				.start(stepA)
				.on("*").to(stepB)
				.from(stepA).on("FAILED").to(stepC)
				.end()
				.build();
}
```

Java 설정을 사용할 때, `on()` 메서드는 스텝(Step) 실행 결과로 나오는 `ExitStatus`와 일치시키기 위해 간단한 패턴 매칭 체계를 사용합니다.

패턴에는 두 가지 특수 문자만 허용됩니다:

- 는 0개 이상의 문자와 일치합니다.
- `?`는 정확히 하나의 문자와 일치합니다.

예를 들어, `c*t`는 `cat`과 `count`에 일치하지만, `c?t`는 `cat`에는 일치하지만 `count`에는 일치하지 않습니다.

스텝(Step)에 대한 전환(transition) 요소의 수에는 제한이 없지만, 스텝 실행 결과가 요소에 의해 커버되지 않는 `ExitStatus`를 반환하면 프레임워크는 예외를 발생시키고 잡(Job)은 실패합니다.

프레임워크는 자동으로 전환을 가장 구체적인 것에서 가장 덜 구체적인 순서로 정렬합니다.

이것은 이전 예제에서 `stepA`에 대한 순서가 바뀌었더라도, `FAILED`의 `ExitStatus`는 여전히 `stepC`로 간다는 것을 의미합니다.

### 배치 상태(Batch Status) 대 종료 상태(Exit Status)

조건부 플로우를 위해 잡(Job)을 구성할 때, `BatchStatus`와 `ExitStatus`의 차이를 이해하는 것이 중요합니다.

`BatchStatus`는 `JobExecution`과 `StepExecution` 모두의 속성인 열거형이며, 프레임워크가 잡(Job) 또는 스텝(Step)의 상태를 기록하는 데 사용됩니다.

다음 값 중 하나일 수 있습니다: `COMPLETED`, `STARTING`, `STARTED`, `STOPPING`, `STOPPED`, `FAILED`, `ABANDONED`, 또는 `UNKNOWN`. 대부분은 자명합니다:

`COMPLETED`는 스텝이나 잡이 성공적으로 완료되었을 때 설정되는 상태이고, `FAILED`는 실패했을 때 설정되는 상태 등입니다.

다음 예제는 Java 설정을 사용할 때의 `on` 요소를 포함합니다:

```java
...
.from(stepA).on("FAILED").to(stepB)
...
```

언뜻 보면, `on`이 속한 스텝(Step)의 `BatchStatus`를 참조하는 것처럼 보일 수 있습니다.

그러나 실제로는 스텝(Step)의 `ExitStatus`를 참조합니다.

이름에서 알 수 있듯이, `ExitStatus`는 스텝(Step)이 실행을 마친 후의 상태를 나타냅니다.

Java 설정을 사용할 때, 이전 Java 설정 예제에 표시된 `on()` 메서드는 `ExitStatus`의 종료 코드를 참조합니다.

영어로 말하면, "종료 코드가 `FAILED`이면 `stepB`로 가라"는 의미입니다. 기본적으로 종료 코드는 항상 스텝(Step)의 `BatchStatus`와 동일하므로 이전 항목이 작동합니다.

그러나 종료 코드가 달라야 한다면 어떨까요? 좋은 예는 샘플 프로젝트 내의 `skip` 샘플 잡(job)에서 나옵니다:

다음 예제는 Java에서 다른 종료 코드로 작업하는 방법을 보여줍니다:

**Java 설정 (Java Configuration)**

```java
@Bean
public Job job(JobRepository jobRepository, Step step1, Step step2, Step errorPrint1) {
	return new JobBuilder("job", jobRepository)
			.start(step1).on("FAILED").end()
			.from(step1).on("COMPLETED WITH SKIPS").to(errorPrint1)
			.from(step1).on("*").to(step2)
			.end()
			.build();
}
```

`step1`에는 세 가지 가능성이 있습니다:

1. 스텝(Step)이 실패했고, 이 경우 잡(Job)이 실패해야 합니다.
2. 스텝(Step)이 성공적으로 완료되었습니다.
3. 스텝(Step)이 성공적으로 완료되었지만 종료 코드가 `COMPLETED WITH SKIPS`입니다. 이 경우 오류를 처리하기 위해 다른 스텝을 실행해야 합니다.

이전 구성은 작동합니다. 그러나 다음 예제와 같이 레코드를 건너뛴 실행 조건에 따라 종료 코드를 변경하는 무언가가 필요합니다:

```java
public class SkipCheckingListener implements StepExecutionListener {
    @Override
    public ExitStatus afterStep(StepExecution stepExecution) {
        String exitCode = stepExecution.getExitStatus().getExitCode();
        if (!exitCode.equals(ExitStatus.FAILED.getExitCode()) &&
            stepExecution.getSkipCount() > 0) {
            return new ExitStatus("COMPLETED WITH SKIPS");
        } else {
            return null; // null을 반환하면 프레임워크가 기본 ExitStatus를 사용
        }
    }
}
```

위 코드는 `StepExecutionListener`로, 먼저 스텝(Step)이 성공했는지 확인한 다음 `StepExecution`의 `skipCount`가 0보다 큰지 확인합니다.

두 조건이 모두 충족되면 `COMPLETED WITH SKIPS`라는 종료 코드를 가진 새로운 `ExitStatus`가 반환됩니다.

### 중지를 위한 설정 (Configuring for Stop)

`BatchStatus`와 `ExitStatus`에 대한 논의 후, 잡(Job)에 대한 `BatchStatus`와 `ExitStatus`가 어떻게 결정되는지 궁금할 수 있습니다.

이러한 상태는 실행되는 코드에 의해 스텝(Step)에 대해 결정되지만, 잡(Job)에 대한 상태는 구성에 따라 결정됩니다.

지금까지 논의된 모든 잡(Job) 구성에는 전환(transition)이 없는 하나 이상의 최종 스텝(Step)이 있었습니다.

다음 Java 예제에서 스텝이 실행된 후 잡(Job)이 종료됩니다:

```java
@Bean
public Job job(JobRepository jobRepository, Step step1) {
	return new JobBuilder("job", jobRepository)
				.start(step1)
				.build();
}
```

스텝(Step)에 대해 전환(transition)이 정의되지 않은 경우, 잡(Job)의 상태는 다음과 같이 정의됩니다:

- 스텝(Step)이 `FAILED`의 `ExitStatus`로 끝나면, 잡(Job)의 `BatchStatus`와 `ExitStatus`는 모두 `FAILED`입니다.
- 그렇지 않으면, 잡(Job)의 `BatchStatus`와 `ExitStatus`는 모두 `COMPLETED`입니다.

배치 잡(batch job)을 종료하는 이 방법은 간단한 순차 스텝 잡(job)과 같은 일부 배치 잡(job)에는 충분하지만, 사용자 정의된 잡(job) 중지 시나리오가 필요할 수 있습니다.

이를 위해 Spring Batch는 잡(Job)을 중지하기 위한 세 가지 전환 요소(이전에 논의한 `next` 요소 외에)를 제공합니다. 이러한 각 중지 요소는 특정 `BatchStatus`로 잡(Job)을 중지합니다.

중지 전환 요소는 잡(Job) 내의 어떤 스텝(Step)의 `BatchStatus`나 `ExitStatus`에도 영향을 미치지 않는다는 점에 유의하는 것이 중요합니다.

이러한 요소는 잡(Job)의 최종 상태에만 영향을 미칩니다. 예를 들어, 잡(job)의 모든 스텝(step)이 `FAILED` 상태를 갖지만 잡(job)은 `COMPLETED` 상태를 가질 수 있습니다.

### 스텝에서 종료하기 (Ending at a Step)

스텝 종료를 구성하면 잡(Job)이 `COMPLETED`의 `BatchStatus`로 중지하도록 지시합니다.

`COMPLETED` 상태로 완료된 잡(Job)은 다시 시작할 수 없습니다 (프레임워크는 `JobInstanceAlreadyCompleteException`을 발생시킵니다).

Java 설정을 사용할 때, 이 작업에는 `end` 메서드가 사용됩니다. `end` 메서드는 또한 잡(Job)의 `ExitStatus`를 사용자 정의하는 데 사용할 수 있는 선택적 `exitStatus` 매개변수를 허용합니다.

`exitStatus` 값이 제공되지 않으면, `ExitStatus`는 기본적으로 `COMPLETED`가 되어 `BatchStatus`와 일치합니다.

다음 시나리오를 고려하십시오: `step2`가 실패하면, 잡(Job)은 `COMPLETED`의 `BatchStatus`와 `COMPLETED`의 `ExitStatus`로 중지되고 `step3`은 실행되지 않습니다.

그렇지 않으면 실행은 `step3`으로 이동합니다. `step2`가 실패하면 잡(Job)은 (상태가 `COMPLETED`이므로) 재시작할 수 없습니다.

다음 예제는 Java에서의 시나리오를 보여줍니다:

```java
@Bean
public Job job(JobRepository jobRepository, Step step1, Step step2, Step step3) {
	return new JobBuilder("job", jobRepository)
				.start(step1)
				.next(step2)
				.on("FAILED").end() // step2가 FAILED이면 Job을 COMPLETED 상태로 종료
				.from(step2).on("*").to(step3) // step2가 FAILED가 아니면 step3로
				.end() // 플로우 정의 종료
				.build();
}
```

### 스텝 실패시키기 (Failing a Step)

주어진 지점에서 스텝이 실패하도록 구성하면 잡(Job)이 `FAILED`의 `BatchStatus`로 중지하도록 지시합니다.

`end`와 달리, 잡(Job)의 실패는 잡(Job)이 재시작되는 것을 막지 않습니다.

XML 설정을 사용할 때, `fail` 요소는 잡(Job)의 `ExitStatus`를 사용자 정의하는 데 사용할 수 있는 선택적 `exit-code` 속성을 허용합니다.

`exit-code` 속성이 주어지지 않으면, `ExitStatus`는 기본적으로 `FAILED`가 되어 `BatchStatus`와 일치합니다.

다음 시나리오를 고려하십시오: `step2`가 실패하면, 잡(Job)은 `FAILED`의 `BatchStatus`와 `EARLY TERMINATION`의 `ExitStatus`로 중지되고 `step3`은 실행되지 않습니다.

그렇지 않으면 실행은 `step3`으로 이동합니다.

또한, `step2`가 실패하고 잡(Job)이 재시작되면 실행은 `step2`에서 다시 시작됩니다.

다음 예제는 Java에서의 시나리오를 보여줍니다:

**Java 설정 (Java Configuration)**

```java
@Bean
public Job job(JobRepository jobRepository, Step step1, Step step2, Step step3) {
	return new JobBuilder("job", jobRepository)
			.start(step1)
			.next(step2).on("FAILED").fail() // step2가 FAILED이면 Job을 FAILED 상태로 종료
			.from(step2).on("*").to(step3) // step2가 FAILED가 아니면 step3로
			.end() // 플로우 정의 종료
			.build();
}
```

### 주어진 스텝에서 잡(Job) 중지하기 (Stopping a Job at a Given Step)

특정 스텝에서 잡(Job)을 중지하도록 구성하면 잡(Job)이 `STOPPED`의 `BatchStatus`로 중지하도록 지시합니다.

잡(Job)을 중지하면 운영자가 잡(Job)을 재시작하기 전에 어떤 조치를 취할 수 있도록 처리 중에 일시적인 중단을 제공할 수 있습니다.

Java 설정을 사용할 때, `stopAndRestart` 메서드는 잡(Job)이 재시작될 때 실행이 시작되어야 하는 스텝을 지정하는 `restart` 속성을 필요로 합니다.

다음 시나리오를 고려하십시오: `step1`이 `COMPLETED`로 완료되면, 잡(Job)은 중지됩니다. 재시작되면 실행은 `step2`에서 시작됩니다.

다음 예제는 Java에서의 시나리오를 보여줍니다:

```java
@Bean
public Job job(JobRepository jobRepository, Step step1, Step step2) {
	return new JobBuilder("job", jobRepository)
			.start(step1).on("COMPLETED").stopAndRestart(step2) // step1이 COMPLETED이면 Job을 STOPPED 상태로 중지하고, 재시작 시 step2부터 시작
			.end() // 플로우 정의 종료
			.build();
}
```

### 프로그래밍 방식의 플로우 결정 (Programmatic Flow Decisions)

경우에 따라, 다음에 어떤 스텝을 실행할지 결정하기 위해 `ExitStatus`보다 더 많은 정보가 필요할 수 있습니다.

이 경우, 다음 예제와 같이 `JobExecutionDecider`를 사용하여 결정을 도울 수 있습니다:

```java
public class MyDecider implements JobExecutionDecider {
    public FlowExecutionStatus decide(JobExecution jobExecution, StepExecution stepExecution) {
        String status;
        if (someCondition()) { // 어떤 조건에 따라
            status = "FAILED";
        }
        else {
            status = "COMPLETED";
        }
        return new FlowExecutionStatus(status); // FlowExecutionStatus를 반환하여 다음 흐름을 결정
    }
}
```

다음 예제에서, `JobExecutionDecider`를 구현하는 빈이 Java 설정을 사용할 때 `next` 호출에 직접 전달됩니다:

**Java 설정 (Java Configuration)**

```java
@Bean
public Job job(JobRepository jobRepository, MyDecider decider, Step step1, Step step2, Step step3) {
	return new JobBuilder("job", jobRepository)
			.start(step1)
			.next(decider) // step1 다음에 Decider 실행
			    .on("FAILED").to(step2) // Decider 결과가 FAILED이면 step2로
			.from(decider).on("COMPLETED").to(step3) // Decider 결과가 COMPLETED이면 step3로
			.end() // 플로우 정의 종료
			.build();
}
```

### 분할 플로우 (Split Flows)

지금까지 설명된 모든 시나리오는 스텝을 한 번에 하나씩 선형 방식으로 실행하는 잡(Job)과 관련이 있었습니다.

이러한 일반적인 스타일 외에도 Spring Batch는 병렬 플로우로 잡(Job)을 구성할 수도 있습니다.

Java 기반 구성을 사용하면 제공된 빌더를 통해 분할을 구성할 수 있습니다.

다음 예제와 같이, `split` 요소에는 하나 이상의 `flow` 요소가 포함되며, 여기서 완전히 별개의 플로우를 정의할 수 있습니다.

`split` 요소에는 이전에 논의된 `next` 속성 또는 `next`, `end`, `fail` 요소와 같은 전환 요소도 포함될 수 있습니다.

```java
@Bean
public Flow flow1(Step step1, Step step2) {
	return new FlowBuilder<SimpleFlow>("flow1")
			.start(step1)
			.next(step2)
			.build();
}

@Bean
public Flow flow2(Step step3) {
	return new FlowBuilder<SimpleFlow>("flow2")
			.start(step3)
			.build();
}

@Bean
public Job job(JobRepository jobRepository, Flow flow1, Flow flow2, Step step4) {
	return new JobBuilder("job", jobRepository)
				.start(flow1) // flow1 시작
				.split(new SimpleAsyncTaskExecutor()) // TaskExecutor를 사용하여 병렬 실행 설정
				.add(flow2) // flow1과 병렬로 flow2 추가
				.next(step4) // flow1과 flow2가 모두 완료된 후 step4 실행
				.end() // 플로우 정의 종료
				.build();
}
```

### 플로우 정의 외부화 및 잡(Job) 간의 종속성 (Externalizing Flow Definitions and Dependencies Between Jobs)

잡(Job)의 플로우 일부는 별도의 빈 정의로 외부화한 다음 재사용할 수 있습니다.

이렇게 하는 방법에는 두 가지가 있습니다. 첫 번째는 다른 곳에 정의된 플로우에 대한 참조로 플로우를 선언하는 것입니다.

다음 Java 예제는 다른 곳에 정의된 플로우에 대한 참조로 플로우를 선언하는 방법을 보여줍니다:

**Java 설정 (Java Configuration)**

```java
@Bean
public Job job(JobRepository jobRepository, Flow flow1, Step step3) {
	return new JobBuilder("job", jobRepository)
				.start(flow1) // 미리 정의된 flow1을 시작
				.next(step3)
				.end()
				.build();
}

@Bean
public Flow flow1(Step step1, Step step2) { // flow1 정의
	return new FlowBuilder<SimpleFlow>("flow1")
			.start(step1)
			.next(step2)
			.build();
}
```

이전 예제와 같이 외부 플로우를 정의하는 효과는 외부 플로우의 스텝들을 마치 인라인으로 선언된 것처럼 잡(Job)에 삽입하는 것입니다.

이런 식으로 많은 잡(Job)들이 동일한 템플릿 플로우를 참조하고 이러한 템플릿들을 다른 논리적 플로우로 구성할 수 있습니다.

이것은 또한 개별 플로우의 통합 테스트를 분리하는 좋은 방법입니다.

외부화된 플로우의 다른 형태는 `JobStep`을 사용하는 것입니다.

`JobStep`은 `FlowStep`과 유사하지만 지정된 플로우의 스텝들에 대해 실제로 별도의 잡 실행(job execution)을 생성하고 시작합니다.

다음 예제는 Java에서 `JobStep`의 예를 보여줍니다:

**Java 설정 (Java Configuration)**

```java
// jobStepJob 이라는 이름의 Job 정의 (JobStep에 의해 실행될 Job)
@Bean
public Job jobStepJob(JobRepository jobRepository, Step jobStepJobStep1) {
	return new JobBuilder("jobStepJob", jobRepository)
				.start(jobStepJobStep1)
				.build();
}

// jobStepJob의 Step 정의
@Bean
public Step jobStepJobStep1(JobRepository jobRepository, JobLauncher jobLauncher, Job job, JobParametersExtractor jobParametersExtractor) {
	return new StepBuilder("jobStepJobStep1", jobRepository) // Step 이름
				.job(job) // 실행할 Job (여기서는 'job'이라는 이름의 Job을 참조하게 되는데, 실제로는 'jobStepJob'을 실행해야 함. 예제 코드의 혼동 가능성 있음. job() 메서드의 job을 참조)
				.launcher(jobLauncher) // Job을 실행할 Launcher
				.parametersExtractor(jobParametersExtractor) // JobParameters 추출기
				.build();
}

// 메인 Job 정의
@Bean
public Job job(JobRepository jobRepository, Step jobStepJobStep1) { // jobStepJobStep1은 실제로는 JobStep을 참조해야 함. 아래 주석 참고
	return new JobBuilder("job", jobRepository) // 메인 Job의 이름
				// ... 여기에 JobStep을 사용하는 Step이 와야 함 ...
				// .start(jobStep()) // 예: jobStep() 메서드가 JobStep을 반환한다고 가정
				.build();
}
// 위 Job 설정에서 jobStepJobStep1을 직접 참조하는 것은 JobStep의 일반적인 사용법과 다릅니다.
// 보통은 아래와 같이 JobStep 자체를 빈으로 만들고, 이를 메인 Job의 스텝으로 사용합니다.
// 예시:
// @Bean
// public Step actualJobStep(JobLauncher jobLauncher, JobRepository jobRepository, Job jobToRun, JobParametersExtractor extractor) {
//     return new StepBuilder("myJobStep", jobRepository)
//                .job(jobToRun) // 'jobStepJob' 빈을 주입받아 사용
//                .launcher(jobLauncher)
//                .parametersExtractor(extractor)
//                .build();
// }
// 그리고 메인 Job에서는:
// .start(actualJobStep)

@Bean
public DefaultJobParametersExtractor jobParametersExtractor() {
	DefaultJobParametersExtractor extractor = new DefaultJobParametersExtractor();
	// 실행될 Job에 전달할 파라미터 키 설정
	extractor.setKeys(new String[]{"input.file"});
	return extractor;
}

```

잡 파라미터 추출기는 스텝(Step)의 `ExecutionContext`가 실행될 잡(Job)의 `JobParameters`로 변환되는 방식을 결정하는 전략입니다.

`JobStep`은 잡(Job) 및 스텝(Step)에 대한 모니터링 및 보고에 대해 좀 더 세분화된 옵션을 갖고 싶을 때 유용합니다.

`JobStep`을 사용하는 것은 종종 "잡(Job) 간의 종속성을 어떻게 생성합니까?"라는 질문에 대한 좋은 답변이기도 합니다. 큰 시스템을 작은 모듈로 나누고 잡(Job)의 흐름을 제어하는 좋은 방법입니다.

---

### **학습 목표 제시**

이번 "Controlling Step Flow" 챕터를 통해 학생분은 다음을 이해하고 배울 수 있습니다:

1. **다양한 스텝 흐름 구성 방법 이해:** 단순한 순차 실행부터 시작하여, 특정 조건에 따라 다른 스텝으로 분기하는 조건부 흐름을 구성하는 방법을 배웁니다.
2. **`BatchStatus`와 `ExitStatus`의 차이점 및 활용법 파악:** 스텝과 잡의 상태를 나타내는 `BatchStatus`와 플로우 제어에 직접 사용되는 `ExitStatus`의 개념을 구분하고, `ExitStatus`를 커스터마이징하여 정교한 흐름 제어를 하는 방법을 익힙니다.
3. **고급 플로우 제어 기법 학습:** `JobExecutionDecider`를 사용한 프로그래밍 방식의 동적 흐름 결정, `Split Flow`를 이용한 병렬 스텝 실행, 그리고 `Flow`나 `JobStep`을 통해 플로우를 모듈화하고 재사용하는 방법을 배웁니다.

---

### **핵심 개념 설명**

이 챕터의 핵심은 **"잡(Job) 내에서 스텝(Step)들의 실행 순서와 조건을 어떻게 제어할 것인가?"** 입니다. 단순히 A 스텝 다음에 B 스텝, B 스텝 다음에 C 스텝으로 순서대로 실행되는 것 외에도, 특정 상황에 따라 다른 길로 가거나, 특정 스텝에서 멈추거나, 여러 스텝을 동시에 실행하는 등 다양한 시나리오가 필요할 수 있습니다.

### 1. 플로우(Flow)의 기본: 순차적 흐름 (Sequential Flow)

- **개념:** 가장 기본적인 형태로, 스텝들이 정의된 순서대로 하나씩 실행됩니다. 마치 기차놀이처럼 앞선 스텝이 끝나면 다음 스텝이 출발하는 거죠.
- **예시:** `stepA` -> `stepB` -> `stepC`
- **핵심:** `JobBuilder`의 `start()` 메서드로 첫 스텝을 지정하고, `next()` 메서드로 다음 스텝을 연결합니다.
- **"왜?"**: 가장 간단하고 직관적인 배치 처리 방식입니다. 데이터 추출(A) -> 데이터 가공(B) -> 데이터 저장(C)과 같이 순차적인 작업에 적합합니다.

### 2. 상황에 따른 길 안내: 조건부 흐름 (Conditional Flow)

- **개념:** 특정 스텝의 실행 결과(성공, 실패 등)에 따라 다음에 실행될 스텝을 다르게 지정할 수 있습니다. 마치 갈림길에서 표지판을 보고 다음 목적지를 정하는 것과 같아요.
- **예시:**
  - `stepA` 성공 시 -> `stepB` 실행
  - `stepA` 실패 시 -> `stepC` (예: 에러 처리 스텝) 실행
- **핵심:** `on()` 메서드와 `to()` 메서드를 사용합니다. `on()`에는 스텝의 `ExitStatus` 패턴을, `to()`에는 다음에 실행할 스텝을 지정합니다. `from()`을 사용해 특정 스텝으로부터의 분기를 명시할 수도 있습니다.
- **"왜?"**: 실제 업무에서는 성공/실패 외에도 다양한 상황이 발생할 수 있습니다. 예를 들어, "데이터가 일부 누락되었지만 처리는 계속하고 싶다"와 같은 시나리오에 대응하기 위해 필요합니다.

### 3. 스텝의 진짜 속마음: `ExitStatus` vs. `BatchStatus`

- **`BatchStatus`**: 스텝이나 잡의 **최종 실행 상태**를 나타내는 프레임워크 수준의 상태입니다. (예: `COMPLETED`, `FAILED`, `STOPPED`). 주로 모니터링이나 잡 재시작 여부 판단에 사용됩니다.
  - 비유: 학생의 최종 성적표 (A학점, F학점 등)
- **`ExitStatus`**: 스텝 실행 후 **플로우 제어를 위해 사용되는 상태 코드**입니다. 기본적으로 `BatchStatus`와 같지만, 개발자가 커스텀한 값을 가질 수 있습니다.
  - 비유: 시험을 본 후 "만점", "부분 정답", "오답"과 같이 좀 더 구체적인 결과. 이 결과에 따라 다음 학습 계획이 달라질 수 있습니다.
- **핵심:** `on()` 메서드는 `ExitStatus`를 기준으로 다음 스텝을 결정합니다. `ExitStatus`를 커스터마이징하면 (예: `StepExecutionListener` 사용) 더욱 세밀한 플로우 제어가 가능합니다.
- **"왜?"**: `BatchStatus`만으로는 단순 성공/실패만 구분 가능하지만, `ExitStatus`를 활용하면 "성공했지만 경고 있음", "특정 조건 만족 시 다른 흐름" 등 복잡한 비즈니스 로직을 플로우에 반영할 수 있습니다.

### 4. 잡(Job)의 마무리 방식 결정: `end()`, `fail()`, `stopAndRestart()`

- **개념:** 스텝의 결과에 따라 잡(Job) 전체를 어떻게 종료할지 결정합니다.
  - `end()`: 잡을 `COMPLETED` 상태로 종료. (재시작 불가)
    - 비유: 프로젝트 완료! 더 이상 할 일 없음.
  - `fail()`: 잡을 `FAILED` 상태로 종료. (재시작 가능)
    - 비유: 프로젝트 중단! 원인 파악 후 다시 시작해야 함.
  - `stopAndRestart(step)`: 잡을 `STOPPED` 상태로 중지. 재시작 시 지정된 스텝부터 시작.
    - 비유: 잠시 멈춤! 중간 점검 후 특정 지점부터 다시 시작.
- **핵심:** 이 메서드들은 잡(Job)의 최종 `BatchStatus`에 영향을 줍니다.
- **"왜?"**: 잡의 성공/실패/중지 상태를 명확히 하고, 재시작 정책을 정의하기 위해 필요합니다. 예를 들어, 치명적인 오류가 발생하면 `fail()`로 잡을 실패 처리하고, 운영자의 개입이 필요한 경우 `stopAndRestart()`로 특정 지점에서 멈추게 할 수 있습니다.

### 5. 똑똑한 길잡이: `JobExecutionDecider`

- **개념:** `ExitStatus`만으로는 부족한, 더 복잡한 조건으로 다음 스텝을 결정해야 할 때 사용합니다. `JobExecutionDecider` 인터페이스를 구현한 클래스가 동적으로 다음 상태를 결정합니다.
- **예시:** 데이터베이스의 특정 값, 외부 시스템의 응답 등 스텝 실행 결과 외의 정보를 바탕으로 분기합니다.
- **핵심:** `decide()` 메서드가 `FlowExecutionStatus` (내부에 `ExitStatus`와 유사한 문자열 상태 포함)를 반환하여 다음 흐름을 결정합니다.
- **"왜?"**: 스텝의 `ExitStatus`만으로는 표현하기 어려운 복잡한 비즈니스 규칙이나 외부 요인에 따라 플로우를 제어해야 할 때 유용합니다. 의사결정 로직을 별도의 컴포넌트로 분리하여 관리할 수 있습니다.

### 6. 동시에 여러 작업 처리: 분할 플로우 (Split Flows)

- **개념:** 여러 개의 플로우(Flow)를 동시에 병렬로 실행할 수 있게 합니다. 각 플로우는 독립적으로 실행되며, 모든 병렬 플로우가 완료되어야 다음 스텝으로 진행합니다.
- **예시:** `flow1` (A -> B)과 `flow2` (C -> D)를 동시에 실행하고, 둘 다 끝나면 `stepE`를 실행합니다.
- **핵심:** `split()` 메서드에 `TaskExecutor`를 지정하여 병렬 실행을 설정하고, `add()`로 병렬 실행할 플로우들을 추가합니다.
- **"왜?"**: 서로 의존성이 없는 작업들을 병렬로 처리하여 전체 배치 처리 시간을 단축시킬 수 있습니다. 예를 들어, 여러 개의 파일을 독립적으로 처리하는 경우 유용합니다.

### 7. 플로우 재사용 및 모듈화: 외부화된 플로우와 `JobStep`

- **외부화된 플로우 (Externalized Flow Definitions)**:
  - **개념:** 공통적으로 사용되는 스텝들의 묶음(Flow)을 별도의 빈(Bean)으로 정의하고, 여러 잡(Job)에서 재사용합니다.
  - **핵심:** `FlowBuilder`를 사용해 `Flow` 빈을 만들고, 잡 정의 시 이 `Flow` 빈을 참조합니다.
  - **"왜?"**: 중복 코드를 줄이고, 플로우 로직을 모듈화하여 관리하기 용이합니다. 테스트도 용이해집니다. 마치 함수나 메서드를 만들어 재사용하는 것과 같습니다.
- **`JobStep`**:
  - **개념:** 하나의 스텝처럼 보이지만, 내부적으로는 **별개의 잡(Job)을 실행**하는 특별한 스텝입니다.
  - **핵심:** `StepBuilder`의 `job()` 메서드에 실행할 잡(Job)을 지정합니다. 부모 잡의 `ExecutionContext`에서 자식 잡의 `JobParameters`를 추출하기 위해 `JobParametersExtractor`가 사용됩니다.
  - **"왜?"**:
    - **잡(Job) 간의 의존성 관리**: "A 잡이 성공해야 B 잡을 실행한다"와 같은 시나리오를 구현할 수 있습니다.
    - **모니터링 및 관리 단위 세분화**: 큰 규모의 배치 시스템을 작은 단위의 잡으로 나누어 관리하고 모니터링할 수 있습니다. 각 잡은 독립적인 실행 기록을 가집니다.
    - 마치 큰 프로젝트를 여러 개의 작은 하위 프로젝트로 나누어 진행하고, 각 하위 프로젝트의 완료 여부에 따라 다음 단계를 진행하는 것과 유사합니다.

---

### **주요 용어 해설**

- **`Flow`**: 하나 이상의 스텝(Step)과 그들 사이의 전환(transition) 규칙을 정의하는 객체입니다. 잡(Job)은 하나 이상의 플로우로 구성될 수 있습니다.
- **`JobExecutionDecider`**: 현재 `JobExecution` 및 `StepExecution` 정보를 바탕으로 다음 플로우 상태(`FlowExecutionStatus`)를 프로그래밍 방식으로 결정하는 인터페이스입니다.
- **`FlowExecutionStatus`**: `JobExecutionDecider`가 반환하는 객체로, 다음 전환을 위한 상태 문자열을 가지고 있습니다. `ExitStatus`와 유사한 역할을 합니다.
- **`TaskExecutor`**: `Split Flow`에서 병렬 실행을 담당하는 스프링 인터페이스입니다. `SimpleAsyncTaskExecutor`는 각 작업을 별도의 스레드에서 실행하는 간단한 구현체입니다.
- **`JobStep`**: 다른 잡(Job)을 실행하는 스텝(Step)입니다. 잡(Job)을 모듈화하고 잡(Job) 간의 종속성을 정의하는 데 사용됩니다.
- **`JobParametersExtractor`**: `JobStep`이 부모 잡(Job)의 `ExecutionContext`에서 자식 잡(Job)을 실행하기 위한 `JobParameters`를 추출하는 방법을 정의하는 인터페이스입니다. `DefaultJobParametersExtractor`는 키를 기반으로 값을 추출하는 기본 구현체입니다.
- **`@Configuration(proxyBeanMethods = false)`**: 스프링 설정 클래스에서 `@Bean` 메서드 호출 시 항상 새로운 인스턴스를 반환하도록 하는 설정입니다. Spring Batch 플로우 정의 시, 스텝 빈의 유일성을 보장하기 위해 기본값인 `proxyBeanMethods = true` (또는 생략) 상태를 유지하는 것이 좋습니다. 만약 `false`로 설정하면, 플로우 정의 내에서 동일 스텝을 여러 번 참조할 때 서로 다른 인스턴스가 사용되어 예상치 못한 동작을 유발할 수 있습니다.
  - **"왜?"**: 플로우 정의에서 `.from(stepA).on("FAILED").to(stepC)`와 같이 `stepA`를 여러 번 참조할 때, `proxyBeanMethods = true`이면 항상 동일한 `stepA` 빈 인스턴스를 참조하게 되어 플로우가 올바르게 구성됩니다. `false`이면, 각 참조가 새로운 인스턴스를 만들 수 있어 "어떤 `stepA`로부터의 분기인가?"라는 혼란이 발생할 수 있습니다.

---

### **코드 예제 및 분석**

### 1. 순차적 플로우 (Sequential Flow) - Java Configuration

```java
@Bean
public Job job(JobRepository jobRepository, Step stepA, Step stepB, Step stepC) {
	return new JobBuilder("job", jobRepository) // "job"이라는 이름의 Job 생성 시작, JobRepository 주입
				.start(stepA)        // stepA를 첫 번째 스텝으로 시작
				.next(stepB)         // stepA가 성공적으로 완료되면 stepB 실행
				.next(stepC)         // stepB가 성공적으로 완료되면 stepC 실행
				.build();            // Job 생성 완료
}

```

- **`new JobBuilder("job", jobRepository)`**: `JobBuilder`를 사용하여 "job"이라는 이름의 잡을 만듭니다. `jobRepository`는 잡의 메타데이터(실행 상태 등)를 저장하는 데 사용됩니다.
- **`.start(stepA)`**: `stepA`를 이 잡의 시작 스텝으로 지정합니다.
- **`.next(stepB)`**: 바로 이전에 지정된 스텝(`stepA`)이 `COMPLETED` 상태로 끝나면 `stepB`를 실행하도록 설정합니다.
- **`.next(stepC)`**: 마찬가지로, `stepB`가 `COMPLETED` 상태로 끝나면 `stepC`를 실행합니다.
- **`.build()`**: 설정된 내용으로 `Job` 객체를 생성합니다.

### 2. 조건부 플로우 (Conditional Flow) - Java Configuration

```java
@Bean
public Job job(JobRepository jobRepository, Step stepA, Step stepB, Step stepC) {
	return new JobBuilder("job", jobRepository)
				.start(stepA)                  // stepA를 시작 스텝으로 지정
				.on("*").to(stepB)            // stepA의 ExitStatus가 무엇이든 ('*'는 모든 문자열과 일치) 기본적으로 stepB로 이동
				.from(stepA).on("FAILED").to(stepC) // 하지만, stepA의 ExitStatus가 "FAILED"이면 stepC로 이동
				                                   // (더 구체적인 조건이 우선 적용됨)
				.end()                         // 플로우 정의의 한 부분을 마무리 (더 복잡한 플로우를 위해 사용되며, 여기서는 .build() 전에 오는 관례적 표현으로 볼 수 있음)
				                               // .from() 이후에는 .to(), .next(), .fail(), .end(), .stopAndRestart() 등이 올 수 있고,
				                               // 이들을 연결한 후 다음 from()을 쓰거나 최종 .build()를 하기 전에 .end()로 해당 from() 블록을 닫아주는 개념.
				                               // Job 전체의 끝이 아니라, 특정 분기 흐름의 끝을 명시.
				.build();
}
```

- **`.start(stepA).on("*").to(stepB)`**: `stepA`를 시작하고, `stepA`의 `ExitStatus`가 어떤 패턴(는 모든 것을 의미)이든 `stepB`로 가도록 기본 경로를 설정합니다.
- **`.from(stepA).on("FAILED").to(stepC)`**: `stepA`로부터의 전환을 다시 정의합니다. 만약 `stepA`의 `ExitStatus`가 "FAILED"와 정확히 일치하면, `stepB` 대신 `stepC`로 가도록 재정의합니다. Spring Batch는 더 구체적인 패턴 매칭을 우선합니다.
- **`.end()`**: 이 메서드는 플로우 빌더 API에서 특정 조건부 흐름의 정의를 마치고, 다른 흐름을 정의하거나 잡 빌드를 마무리할 준비가 되었음을 나타냅니다. 여기서는 `from(stepA)`로 시작된 조건부 흐름 정의가 끝났음을 의미합니다. 이후 다른 `.from()`을 사용하거나, `.build()`로 잡 생성을 완료할 수 있습니다. 만약 `stepB`나 `stepC` 이후에 더 이상 스텝이 없다면, 해당 스텝에서 잡이 종료됩니다. (종료 상태는 해당 스텝의 `BatchStatus`를 따름)
  - **`end()`의 다른 의미**: `.on("FAILED").end()` 와 같이 `to()` 대신 사용되면, 해당 조건에서 Job을 `COMPLETED` 상태로 **종료**시키라는 의미도 가집니다. (문맥에 따라 해석) 위 예제에서는 `from(stepA)`로 시작된 *분기 정의의 끝*을 나타냅니다.

### 3. `ExitStatus` 커스터마이징 - `StepExecutionListener`

```java
public class SkipCheckingListener implements StepExecutionListener {
    @Override
    public void beforeStep(StepExecution stepExecution) {
        // 스텝 시작 전 로직 (필요시 구현)
    }

    @Override
    public ExitStatus afterStep(StepExecution stepExecution) {
        String originalExitCode = stepExecution.getExitStatus().getExitCode(); // 스텝 실행 후의 기본 ExitStatus 가져오기
        // 스텝 자체가 FAILED가 아니고, skip된 항목이 있다면
        if (!originalExitCode.equals(ExitStatus.FAILED.getExitCode()) &&
            stepExecution.getSkipCount() > 0) {
            return new ExitStatus("COMPLETED WITH SKIPS"); // 새로운 ExitStatus 반환
        } else {
            return null; // null을 반환하면 프레임워크는 원래의 ExitStatus를 사용합니다.
                         // 또는 return stepExecution.getExitStatus(); 와 같이 명시적으로 기존 것을 반환해도 됩니다.
        }
    }
}
```

- **`StepExecutionListener`**: 스텝 실행 생명주기(시작 전, 완료 후)에 개입하여 추가 로직을 수행할 수 있는 인터페이스입니다.
- **`afterStep(StepExecution stepExecution)`**: 스텝이 완료된 후 호출됩니다.
- **`stepExecution.getExitStatus().getExitCode()`**: 현재 스텝의 `ExitStatus`에서 `exitCode` (문자열)를 가져옵니다.
- **`stepExecution.getSkipCount()`**: 스텝 실행 중 skip된 아이템의 수를 가져옵니다. (ItemProcessor나 ItemWriter에서 예외 발생 시 skip 처리 설정 가능)
- **`return new ExitStatus("COMPLETED WITH SKIPS");`**: 조건(실패하지 않았고, skip된 아이템이 있음)을 만족하면, 새로운 `ExitStatus` 객체를 "COMPLETED WITH SKIPS"라는 커스텀 코드로 생성하여 반환합니다. 이 값이 플로우 제어에 사용됩니다.
- **`return null;`**: 만약 `null`을 반환하면, Spring Batch는 해당 스텝의 원래 `ExitStatus`를 그대로 사용합니다. 변경할 필요가 없을 때 이렇게 합니다.

    ```java
    // 이 리스너를 사용하는 Job 설정 예시
    @Bean
    public Job job(JobRepository jobRepository, Step step1, Step step2, Step errorPrint1, SkipCheckingListener skipListener) {
        return new JobBuilder("job", jobRepository)
                .start(step1)
                    .listener(skipListener) // step1에 리스너 등록
                    .on("FAILED").end() // step1의 ExitStatus가 FAILED (기본 또는 리스너에 의해 FAILED로 유지된 경우)이면 Job을 COMPLETED로 종료
                    .from(step1).on("COMPLETED WITH SKIPS").to(errorPrint1) // 리스너가 ExitStatus를 "COMPLETED WITH SKIPS"로 변경한 경우 errorPrint1 실행
                    .from(step1).on("*").to(step2) // 그 외의 경우 (예: "COMPLETED") step2 실행
                .end() // from(step1) 블록 종료
                .build();
    }
    ```


### 4. `JobExecutionDecider` 사용 예시

```java
// Decider 구현체
public class MyDecider implements JobExecutionDecider {
    private boolean someCondition() {
        // 실제로는 여기서 데이터베이스 조회, 시스템 상태 확인 등
        // 복잡한 로직을 수행하여 조건을 판단합니다.
        // 예시로 간단히 true/false를 번갈아 반환하도록 합니다.
        return System.currentTimeMillis() % 2 == 0;
    }

    @Override
    public FlowExecutionStatus decide(JobExecution jobExecution, StepExecution stepExecution) {
        // jobExecution: 현재 실행 중인 Job의 정보 (JobParameters, JobId 등)
        // stepExecution: decider 바로 이전에 실행된 Step의 정보 (null일 수도 있음 - Job 시작 직후 Decider 사용 시)
        String status;
        if (someCondition()) {
            status = "MY_CUSTOM_STATUS_A"; // 사용자가 정의한 상태값
            System.out.println("Decider: 조건 만족! " + status + " 반환");
        } else {
            status = "MY_CUSTOM_STATUS_B"; // 사용자가 정의한 다른 상태값
            System.out.println("Decider: 조건 불만족! " + status + " 반환");
        }
        return new FlowExecutionStatus(status); // 이 status 문자열을 기반으로 .on()에서 분기
    }
}

// Job 설정
@Bean
public Job deciderJob(JobRepository jobRepository, MyDecider myDecider, Step firstStep, Step pathAStep, Step pathBStep) {
    return new JobBuilder("deciderJob", jobRepository)
            .start(firstStep)
            .next(myDecider) // firstStep 실행 후 myDecider 실행
                .on("MY_CUSTOM_STATUS_A").to(pathAStep) // Decider가 "MY_CUSTOM_STATUS_A" 반환 시 pathAStep 실행
            .from(myDecider).on("MY_CUSTOM_STATUS_B").to(pathBStep) // Decider가 "MY_CUSTOM_STATUS_B" 반환 시 pathBStep 실행
            .from(myDecider).on("*").fail() // 혹시 모를 다른 상태값 반환 시 Job 실패 처리 (안전장치)
            .end() // myDecider 분기 정의 종료
            .build();
}
```

- `MyDecider`는 `JobExecutionDecider` 인터페이스를 구현합니다.
- `decide()` 메서드 내에서 `someCondition()` (예시 로직)의 결과에 따라 `FlowExecutionStatus`를 "MY\_CUSTOM\_STATUS\_A" 또는 "MY\_CUSTOM\_STATUS\_B"로 반환합니다.
- 잡 설정에서 `firstStep` 다음에 `myDecider`를 `next()`로 연결합니다.
- `myDecider`의 `FlowExecutionStatus` 결과에 따라 `.on("...")`으로 분기하여 `pathAStep` 또는 `pathBStep`으로 이동합니다.
- `.from(myDecider).on("*").fail()`: `MyDecider`가 예상치 못한 상태를 반환할 경우, 잡을 `FAILED` 시키는 방어적인 코드입니다.

---

### **"왜?" 라는 질문에 대한 답변**

- **왜 스텝 플로우 제어가 필요한가?**
  - **유연성**: 모든 배치 작업이 단순 순차적이지 않습니다. 오류 발생 시 대체 경로를 타거나, 특정 조건에 따라 다른 작업을 수행해야 하는 경우가 많습니다. 예를 들어, 파일 처리 중 특정 형식의 오류가 발생하면 해당 파일을 별도로 백업하고 다음 파일 처리를 계속하거나, 모든 처리가 끝난 후 관리자에게 알림을 보내는 등의 시나리오가 있을 수 있습니다.
  - **견고성**: 스텝 실패가 전체 잡의 실패를 의미하지 않도록 만들 수 있습니다. 경미한 오류는 로깅만 하고 넘어가거나, 보정 스텝을 실행하여 데이터 정합성을 맞추는 등의 처리가 가능합니다.
  - **효율성**: `Split Flow`를 통해 독립적인 작업들을 병렬로 처리하여 전체 수행 시간을 단축할 수 있습니다.
  - **모듈화 및 재사용성**: `Flow`나 `JobStep`을 통해 공통 로직을 분리하고 재사용하여 개발 생산성과 유지보수성을 높일 수 있습니다.
- **왜 `BatchStatus` 외에 `ExitStatus`가 필요한가?**
  - `BatchStatus`는 `COMPLETED`, `FAILED` 등 프레임워크가 정의한 몇 가지 상태만 나타낼 수 있습니다. 하지만 실제 비즈니스 로직에서는 "완료되었지만 경고 발생", "데이터 없음", "부분 성공" 등 더 세분화된 결과 상태에 따라 다른 흐름을 타고 싶을 수 있습니다. `ExitStatus`는 이러한 사용자 정의 상태를 문자열로 표현하고, 이를 기반으로 플로우를 제어할 수 있게 해줍니다. 즉, 플로우 제어의 **"결정 키(decision key)"** 역할을 합니다.

---

### **주의사항 및 Best Practice**

1. **`ExitStatus` 패턴의 명확성**: `on()` 메서드에 사용되는 패턴은 명확해야 합니다. 와일드카드(, `?`)를 과도하게 사용하면 의도치 않은 흐름으로 빠질 수 있습니다. 가장 구체적인 패턴부터 덜 구체적인 패턴 순으로 정의하는 것이 좋습니다 (Spring Batch가 내부적으로 그렇게 정렬하긴 하지만, 가독성을 위해서도 좋습니다).
2. **모든 `ExitStatus` 처리**: 특정 스텝에서 발생할 수 있는 모든 `ExitStatus`에 대해 전환을 정의하는 것이 좋습니다. 정의되지 않은 `ExitStatus`가 발생하면 잡은 예외를 던지고 실패합니다. `on("*")`를 사용하여 "그 외 모든 경우"에 대한 기본 경로를 설정하는 것이 안전합니다.
3. **`Decider`의 책임 분리**: `JobExecutionDecider`는 플로우 결정 로직만 담당해야 합니다. 비즈니스 로직(데이터 처리 등)은 스텝 내에 구현하는 것이 좋습니다. Decider는 간결하고 테스트하기 쉽게 유지하세요.
4. **`Split Flow`와 트랜잭션**: `Split Flow` 내의 각 플로우(또는 스텝)는 일반적으로 자체 트랜잭션 컨텍스트에서 실행됩니다. 병렬 실행 시 데이터 일관성 문제를 고려해야 합니다.
5. **`JobStep`과 파라미터 전달**: `JobStep`을 통해 자식 잡으로 파라미터를 전달할 때 `JobParametersExtractor`를 신중하게 구성해야 합니다. 자식 잡이 필요로 하는 모든 파라미터가 정확히 전달되는지 확인하세요.
6. **스텝 빈의 유일성 (Bean Method Proxying)**: Java 설정 사용 시, `@Configuration` 클래스의 `proxyBeanMethods` 속성이 `true`(기본값)인지 확인하세요. `false`로 설정되면 플로우 정의 시 스텝 빈이 싱글톤으로 관리되지 않아 문제가 발생할 수 있습니다. 특별한 이유가 없다면 기본값을 유지하는 것이 좋습니다.
  - 만약 `proxyBeanMethods = false`를 사용해야 하는 상황이라면, 플로우 정의 메서드(예: `job()`)의 파라미터로 스텝 빈들을 직접 주입받아 사용하는 것이 안전합니다. 이렇게 하면 주입 시점에서 스프링이 빈의 유일성을 관리해줍니다. 본문의 예제 코드들이 이 방식을 따르고 있습니다.
7. **가독성 있는 플로우 정의**: 복잡한 플로우는 주석을 잘 활용하고, 가능하다면 여러 `Flow` 빈으로 분리하여 가독성을 높이세요.

---
