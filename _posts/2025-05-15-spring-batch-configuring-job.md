---
title: Spring Batch - Configuring a Job
description: 
author: laze
date: 2025-05-15 00:00:03 +0900
categories: [Dev, SpringBatch]
tags: [SpringBatch]
---
**Job 설정하기**

`Job` 인터페이스에는 여러 구현체가 있습니다. 그러나 이러한 구현체들은 제공되는 빌더(Java 구성용) 또는 XML 네임스페이스(XML 기반 구성용) 뒤에 추상화되어 있습니다. 다음 예는 Java 및 XML 구성을 모두 보여줍니다.

```java
@Bean
public Job footballJob(JobRepository jobRepository, Step playerLoad, Step gameLoad, Step playerSummarization) { // Step들을 파라미터로 받아옵니다.
    return new JobBuilder("footballJob", jobRepository)
                     .start(playerLoad)
                     .next(gameLoad)
                     .next(playerSummarization)
                     .build();
}
```

`Job`(그리고 일반적으로 그 안의 모든 `Step`)은 `JobRepository`가 필요합니다.

`JobRepository`의 구성은 Java 구성을 통해 처리됩니다.

위 예는 세 개의 `Step` 인스턴스로 구성된 `Job`을 보여줍니다.

Job 관련 빌더는 병렬화(Split), 선언적 흐름 제어(Decision), 흐름 정의 외부화(Flow)에 도움이 되는 다른 요소도 포함할 수 있습니다.

**재시작 가능성 (Restartability)**

배치 작업을 실행할 때 중요한 문제 중 하나는 특정 `JobInstance`에 대해 `JobExecution`이 이미 존재할 때 해당 `Job`이 재시작될 때의 동작입니다.

이상적으로는 모든 Job이 중단된 지점에서 시작할 수 있어야 하지만, 이것이 불가능한 시나리오도 있습니다.

이 시나리오에서는 새로운 `JobInstance`가 생성되도록 하는 것은 전적으로 개발자의 책임입니다.

그러나 Spring Batch는 약간의 도움을 제공합니다.

`Job`이 절대로 재시작되어서는 안 되지만 항상 새로운 `JobInstance`의 일부로 실행되어야 하는 경우, `restartable` 속성을 `false`로 설정할 수 있습니다.

다음 예는 Java에서 `restartable` 필드를 `false`로 설정하는 방법을 보여줍니다.

[Java 구성]

```java
@Bean
public Job footballJob(JobRepository jobRepository) {
    return new JobBuilder("footballJob", jobRepository)
                     .preventRestart() // 이 부분이 restartable=false 와 동일
                     ...
                     .build();
}
```

다른 말로 표현하면, `restartable`을 `false`로 설정하는 것은 "이 Job은 다시 시작되는 것을 지원하지 않습니다"를 의미합니다.

재시작할 수 없는 Job을 재시작하려고 하면 `JobRestartException`이 발생합니다. 다음 JUnit 코드는 예외를 발생시킵니다.

```java
Job job = new SimpleJob("nonRestartableJob"); // Job 이름을 생성자에 전달해야 합니다.
job.setRestartable(false);

JobParameters jobParameters = new JobParameters();
// JobRepository 를 통해 JobExecution 생성
JobExecution firstExecution = jobRepository.createJobExecution("nonRestartableJob", jobParameters); // Job 이름 전달
// firstExecution.setJobInstance(...); // JobInstance 설정이 필요할 수 있음 (실제 코드에서는 JobLauncher가 처리)
jobRepository.update(firstExecution); // BATCH_JOB_EXECUTION 테이블에 저장

try {
    // 동일한 Job 이름과 파라미터로 JobExecution 재생성 시도
    jobRepository.createJobExecution("nonRestartableJob", jobParameters);
    fail();
}
catch (JobRestartException e) {
    // 예상된 예외
}
```

*(역자 주: JUnit 예제 코드는 문서에서 제공된 것과 약간 다를 수 있으며, 실제 동작을 위해서는 `JobRepository`의 구체적인 사용 방식과 Job 이름, JobInstance 설정 등이 필요합니다. `jobRepository.createJobExecution(job, jobParameters)` 형태의 직접 호출보다는 `JobLauncher`를 통해 실행하는 것이 일반적입니다.)*

재시작 불가능한 Job에 대해 `JobExecution`을 생성하려는 첫 번째 시도는 문제가 없습니다.

그러나 두 번째 시도에서는 `JobRestartException`이 발생합니다.

**Job 실행 가로채기 (Intercepting Job Execution)**

`Job` 실행 과정에서 라이프사이클의 다양한 이벤트에 대한 알림을 받아 사용자 정의 코드를 실행하는 것이 유용할 수 있습니다.

`SimpleJob`은 적절한 시점에 `JobListener`를 호출하여 이를 허용합니다.

```java
public interface JobExecutionListener {

    void beforeJob(JobExecution jobExecution);

    void afterJob(JobExecution jobExecution);
}
```

`JobListener`는 Job에 리스너를 설정하여 `SimpleJob`에 추가할 수 있습니다.

다음 예는 Java Job 정의에 리스너 메서드를 추가하는 방법을 보여줍니다.

```java
@Bean
public Job footballJob(JobRepository jobRepository, JobExecutionListener sampleListener) { // Listener를 파라미터로 받아옵니다.
    return new JobBuilder("footballJob", jobRepository)
                     .listener(sampleListener)
                     ...
                     .build();
}
```

`afterJob` 메서드는 `Job`의 성공 또는 실패 여부에 관계없이 호출됩니다.

성공 또는 실패 여부를 확인해야 하는 경우, `JobExecution`에서 해당 정보를 얻을 수 있습니다.

```java
public void afterJob(JobExecution jobExecution){
    if (jobExecution.getStatus() == BatchStatus.COMPLETED ) {
        //Job 성공
    }
    else if (jobExecution.getStatus() == BatchStatus.FAILED) {
        //Job 실패
    }
}
```

이 인터페이스에 해당하는 어노테이션은 다음과 같습니다.

- `@BeforeJob`
- `@AfterJob`

**부모 Job으로부터 상속 (Inheriting from a Parent Job)**

여러 Job 그룹이 유사하지만 동일하지는 않은 구성을 공유하는 경우, 구체적인 `Job` 인스턴스가 속성을 상속할 수 있는 "부모" Job을 정의하는 것이 도움이 될 수 있습니다.

Java의 클래스 상속과 유사하게, "자식" Job은 부모의 요소 및 속성과 자신의 요소 및 속성을 결합합니다.

다음 예에서 `baseJob`은 리스너 목록만 정의하는 추상 `Job` 정의입니다.

`Job`(job1)은 `baseJob`에서 리스너 목록을 상속하고 자체 리스너 목록과 병합하여 두 개의 리스너와 하나의 `Step`(step1)을 가진 `Job`을 생성하는 구체적인 정의입니다.

```xml
<job id="baseJob" abstract="true">
    <listeners>
        <listener ref="listenerOne"/>
    </listeners>
</job>

<job id="job1" parent="baseJob">
    <step id="step1" parent="standaloneStep"/>

    <listeners merge="true"> <!-- merge="true"로 부모와 자식 리스너 모두 사용 -->
        <listener ref="listenerTwo"/>
    </listeners>
</job>
```

**JobParametersValidator**

XML 네임스페이스에 선언되거나 `AbstractJob`의 하위 클래스를 사용하는 Job은 런타임에 Job 매개변수에 대한 유효성 검사기(validator)를 선택적으로 선언할 수 있습니다.

예를 들어, Job이 모든 필수 매개변수로 시작되었는지 확인해야 할 때 유용합니다.

간단한 필수 및 선택적 매개변수의 조합을 제한하는 데 사용할 수 있는 `DefaultJobParametersValidator`가 있습니다.

더 복잡한 제약 조건의 경우 인터페이스를 직접 구현할 수 있습니다.

유효성 검사기 구성은 Java 빌더를 통해 지원됩니다.

```java
@Bean
public Job job1(JobRepository jobRepository, JobParametersValidator parametersValidator) { // Validator를 파라미터로 받아옵니다.
    return new JobBuilder("job1", jobRepository)
                     .validator(parametersValidator)
                     ...
                     .build();
}
```

---

## Spring Batch Job 똑똑하게 설정하기: 고급 기능 활용법! 🛠️⚙️

우리가 이전 시간에 `Job`이 무엇인지, 그리고 `JobInstance`, `JobExecution` 같은 핵심 용어들을 배웠죠? 이번에는 이 `Job`을 실제로 만들 때 사용할 수 있는 유용한 설정 방법들에 대해 알아볼 거예요.

### 1. Job 만들기: Java 빌더 vs XML 네임스페이스

`Job`을 만드는 방법은 크게 두 가지가 있어요. 마치 요리 레시피를 글로 쓰느냐, 그림으로 그리느냐의 차이처럼 생각할 수 있어요.

- **Java 구성 (Builders 사용):**
  - Java 코드로 직접 `Job`을 만드는 방식이에요. `JobBuilderFactory`나 `JobBuilder` 같은 "건축가(Builder)" 객체를 사용해서 `Job`의 이름, 포함될 `Step`들, 순서 등을 차곡차곡 쌓아 올리듯 정의해요.
  - **장점:** 타입 체크가 가능하고, 리팩토링이 쉬우며, 복잡한 로직을 코드로 표현하기 좋아요. 최근에는 이 방식을 더 선호하는 추세예요.

    ```java
    @Bean // 이 메서드가 만드는 객체를 Spring이 관리해줘!
    public Job footballJob(JobRepository jobRepository, Step playerLoad, Step gameLoad, Step playerSummarization) {
        return new JobBuilder("footballJob", jobRepository) // "footballJob"이라는 이름의 Job을 jobRepository와 함께 만들게.
                         .start(playerLoad)         // 첫 번째 스텝은 playerLoad로 시작하고,
                         .next(gameLoad)            // 그 다음은 gameLoad,
                         .next(playerSummarization) // 마지막으로 playerSummarization 스텝을 실행할 거야.
                         .build();                  // 자, Job 완성!
    }
    ```

- **XML 기반 구성 (Namespace 사용):**
  - XML 파일에 `<job>`, `<step>` 같은 특별한 태그(네임스페이스)를 사용해서 `Job`을 선언적으로 정의해요.
  - **장점:** 코드를 직접 수정하지 않고 설정을 변경할 수 있고, 전체적인 구조를 한눈에 보기 편할 수 있어요. 예전 프로젝트에서 많이 사용되었어요.

    ```xml
    <job id="footballJob">
        <step id="playerLoadStep" next="gameLoadStep">
            <tasklet ref="playerLoadTasklet"/>
        </step>
        <step id="gameLoadStep" next="playerSummarizationStep">
            <tasklet ref="gameLoadTasklet"/>
        </step>
        <step id="playerSummarizationStep">
            <tasklet ref="playerSummarizationTasklet"/>
        </step>
    </job>
    ```


**핵심:** 어떤 방식을 사용하든, `Job`은 **하나 이상의 `Step`으로 구성**되며, 실행 정보를 기록하기 위해 **`JobRepository`가 필요**하다는 점은 같아요!

---

### 2. 이 Job, 다시 시작해도 될까? (Restartability 설정) 🔄

배치 작업이 중간에 실패했을 때, 처음부터 다시 시작해야 할까요, 아니면 멈춘 부분부터 이어서 할 수 있을까요? 이게 바로 **재시작 가능성(Restartability)** 문제예요.

- **기본 동작:** Spring Batch는 기본적으로 `JobInstance`가 이미 존재하면 재시작으로 간주하고, 중단된 `Step`부터 이어서 실행하려고 해요. (상태 정보는 `JobRepository`에 저장되어 있겠죠?)
- **"이 Job은 절대 재시작하면 안 돼!"**
  - 어떤 `Job`들은 특성상 재시작을 지원하지 않거나, 항상 새로운 `JobInstance`로 실행되어야 할 수 있어요. (예: 매번 완전히 새로운 데이터 세트를 처리해야 하는 경우)
  - 이럴 때는 `Job` 설정에서 **"재시작 불가(restartable=false)"** 옵션을 줄 수 있어요.
  - **Java 설정:** `.preventRestart()` 메서드를 사용해요.

      ```java
      @Bean
      public Job footballJob(JobRepository jobRepository, Step someStep) {
          return new JobBuilder("footballJob", jobRepository)
                           .preventRestart() // 이 Job은 재시작 못하게 할 거야!
                           .start(someStep)
                           .build();
      }
      ```

  - **XML 설정:** `<job restartable="false">` 속성을 사용해요.
- **재시작 불가 Job을 재시작하려고 하면?**
  - `JobRestartException`이라는 오류가 발생하면서 "이건 재시작할 수 없는 Job이야!"라고 알려줘요.

**핵심:** `Job`의 특성에 맞게 **재시작 가능 여부를 명확히 설정**해주는 것이 중요해요. 대부분의 경우는 재시작 가능하게 만들지만, 필요에 따라 재시작을 막을 수도 있어요.

---

### 3. Job 실행 전후에 뭔가 하고 싶어! (JobExecutionListener) 🔔

`Job`이 시작되기 직전이나, `Job`이 (성공이든 실패든) 끝난 직후에 특정 코드를 실행하고 싶을 때가 있어요. 예를 들면,

- `Job` 시작 전에: 필요한 리소스 준비, 로그 남기기 등
- `Job` 종료 후에: 사용한 리소스 정리, 결과 알림 메일 보내기, 작업 성공/실패에 따른 후처리 등

이럴 때 사용하는 것이 바로 **`JobExecutionListener`** 예요!

- **인터페이스:**

    ```java
    public interface JobExecutionListener {
        void beforeJob(JobExecution jobExecution); // Job 시작 전에 호출
        void afterJob(JobExecution jobExecution);  // Job 종료 후에 호출
    }
    ```

- **등록 방법:**
  - **Java 설정:** `.listener()` 메서드를 사용해서 우리가 만든 `JobExecutionListener` 구현체를 등록해요.

      ```java
      @Bean
      public Job footballJob(JobRepository jobRepository, Step someStep, JobExecutionListener myListener) {
          return new JobBuilder("footballJob", jobRepository)
                           .listener(myListener) // Job 실행 전후에 myListener를 호출해줘!
                           .start(someStep)
                           .build();
      }
      ```

  - **XML 설정:** `<listeners><listener ref="myListenerBean"/></listeners>` 태그를 사용해요.
- **`afterJob`에서의 상태 확인:** `afterJob` 메서드는 `Job`이 성공했든 실패했든 항상 호출돼요. 그래서 `Job`의 최종 상태를 확인하려면 `jobExecution.getStatus()`를 사용해야 해요.

    ```java
    public void afterJob(JobExecution jobExecution) {
        if (jobExecution.getStatus() == BatchStatus.COMPLETED) {
            System.out.println("Job 성공! 축하합니다!");
        } else if (jobExecution.getStatus() == BatchStatus.FAILED) {
            System.out.println("Job 실패... 원인을 찾아봅시다.");
            // 실패 원인: jobExecution.getFailureExceptions() 등으로 확인 가능
        }
    }
    ```

- **어노테이션 사용:** `@BeforeJob`, `@AfterJob` 어노테이션을 메서드에 직접 붙여서 리스너를 더 간편하게 만들 수도 있어요.

**핵심:** `JobExecutionListener`를 사용하면 **`Job`의 실행 흐름에 개입하여 부가적인 로직을 수행**할 수 있어요. 마치 공연 시작 전 안내방송, 공연 후 커튼콜 같은 역할이죠!

---

### 4. 비슷한 Job 설정, 재사용하고 싶어! (부모 Job으로부터 상속 - XML 전용) 👨‍👩‍👧

여러 `Job`들이 대부분 비슷한 설정을 공유하는데, 몇 가지만 다르다면? 매번 똑같은 설정을 반복하는 건 비효율적이겠죠. 이럴 때 **"부모 Job"**을 만들어두고, **"자식 Job"**들이 그 설정을 **상속**받도록 할 수 있어요. (주로 XML 설정에서 사용되는 기능이에요.)

- **부모 Job 만들기:** `abstract="true"` 속성을 줘서 "이건 직접 실행할 Job이 아니라, 다른 Job들이 상속하기 위한 템플릿이야"라고 표시해요.
- **자식 Job에서 상속받기:** `parent="부모Job이름"` 속성을 사용해요.
- **설정 병합(Merge):** 자식 Job에서 부모와 같은 종류의 설정(예: listeners)을 추가할 때, `merge="true"` 옵션을 주면 부모의 설정과 자식의 설정을 합칠 수 있어요. `false`이거나 생략하면 자식의 설정이 부모의 설정을 덮어써요.

```xml
<!-- 부모 Job (템플릿) -->
<job id="baseJob" abstract="true">
    <listeners>
        <listener ref="listenerOne"/> <!-- 공통 리스너 1 -->
    </listeners>
</job>

<!-- 자식 Job -->
<job id="job1" parent="baseJob"> <!-- baseJob의 설정을 상속받겠다! -->
    <step id="step1" parent="standaloneStep"/> <!-- 이 Job만의 Step -->

    <listeners merge="true"> <!-- 부모의 리스너와 나의 리스너를 합쳐줘! -->
        <listener ref="listenerTwo"/> <!-- 이 Job만의 추가 리스너 2 -->
    </listeners>
</job>
```

- 위 예시에서 `job1`은 `listenerOne`과 `listenerTwo` 두 개의 리스너를 가지게 됩니다.

**핵심:** (XML 설정에서) **부모 Job 상속**을 이용하면 **공통 설정을 재사용**하여 중복을 줄이고, 설정 관리를 더 효율적으로 할 수 있어요.

---

### 5. Job 실행 전에 파라미터 검사! (JobParametersValidator) 🧐

`Job`을 실행할 때 `JobParameters`를 전달한다고 배웠죠? 그런데 만약 꼭 필요한 파라미터가 빠졌거나, 잘못된 형식의 값이 들어오면 어떻게 될까요? `Job`이 엉뚱하게 동작하거나 오류가 날 수 있어요.

이런 문제를 미리 방지하기 위해 **`JobParametersValidator`**를 사용할 수 있어요. `Job`이 실제로 시작되기 전에 전달된 `JobParameters`가 올바른지 검사하는 역할을 해요.

- **사용 시점:** `Job` 실행 직전에 `JobParameters`의 유효성을 검사.
- **기본 제공 Validator:** `DefaultJobParametersValidator`를 사용하면 간단한 필수/선택 파라미터 조합을 검증할 수 있어요.
- **커스텀 Validator:** 더 복잡한 검증 로직이 필요하면 `JobParametersValidator` 인터페이스를 직접 구현해서 만들 수 있어요.
- **등록 방법:**
  - **Java 설정:** `.validator()` 메서드를 사용해서 등록해요.

      ```java
      @Bean
      public Job job1(JobRepository jobRepository, Step someStep, JobParametersValidator myValidator) {
          return new JobBuilder("job1", jobRepository)
                           .validator(myValidator) // Job 실행 전에 myValidator로 파라미터 검사해줘!
                           .start(someStep)
                           .build();
      }
      ```

  - **XML 설정:** `<job validator="myValidatorBean">` 속성을 사용해요.
- **검증 실패 시:** `JobParametersInvalidException` 예외가 발생하면서 `Job` 실행이 중단돼요.

**핵심:** `JobParametersValidator`를 사용하면 **잘못된 파라미터로 `Job`이 실행되는 것을 미리 막아** 안정성을 높일 수 있어요. 마치 공항에서 비행기 타기 전에 여권 검사하는 것과 같아요!

---
