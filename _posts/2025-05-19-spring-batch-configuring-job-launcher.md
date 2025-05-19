---
title: Spring Batch - Configuring JobLauncher
description: 
author: laze
date: 2025-05-19 00:00:04 +0900
categories: [Dev, SpringBatch]
tags: [SpringBatch]
---
**JobLauncher 설정하기**

`@EnableBatchProcessing`을 사용하면 `JobRegistry`가 제공됩니다.

이 섹션에서는 직접 `JobLauncher`를 구성하는 방법을 설명합니다.

`JobLauncher` 인터페이스의 가장 기본적인 구현은 `TaskExecutorJobLauncher`입니다.

이 구현체의 유일한 필수 의존성은 `JobRepository`입니다 (실행(Execution)을 얻기 위해 필요합니다).

다음 예는 Java에서 `TaskExecutorJobLauncher`를 보여줍니다.
*(이 예제는 `@EnableBatchProcessing`을 사용하지 않고 `JobLauncher` 빈을 수동으로 구성하는 경우입니다. `@EnableBatchProcessing`을 사용하면 `jobLauncher`라는 이름의 빈이 자동으로 생성됩니다.)*

[Java 구성]

```java
// ... JobRepository 빈이 이미 정의되어 있다고 가정 (예: jobRepository) ...

@Configuration
public class MyJobLauncherConfig {

    @Autowired // JobRepository 빈을 주입받음
    private JobRepository jobRepository;

    @Bean
    public JobLauncher jobLauncher() throws Exception {
        TaskExecutorJobLauncher jobLauncher = new TaskExecutorJobLauncher();
        jobLauncher.setJobRepository(jobRepository); // 주입받은 JobRepository 설정
        jobLauncher.afterPropertiesSet(); // 빈 초기화 완료 알림
        return jobLauncher;
    }
}
// ...
```

`JobExecution`이 얻어지면, `Job`의 `execute` 메서드로 전달되어 궁극적으로 `JobExecution`을 호출자에게 반환합니다.

이 순서는 간단하며 스케줄러에서 시작될 때 잘 작동합니다. 그러나 HTTP 요청에서 시작하려고 할 때 문제가 발생합니다.

이 시나리오에서는 `TaskExecutorJobLauncher`가 호출자에게 즉시 반환되도록 실행이 비동기적으로 수행되어야 합니다.

이는 배치 작업과 같이 오래 실행되는 프로세스에 필요한 시간 동안 HTTP 요청을 열어두는 것이 좋은 관행이 아니기 때문입니다.

`TaskExecutor`를 구성하여 이 시나리오를 허용하도록 `TaskExecutorJobLauncher`를 구성할 수 있습니다.

다음 Java 예제는 즉시 반환하도록 `TaskExecutorJobLauncher`를 구성합니다.
*(이 또한 `@EnableBatchProcessing`을 사용하지 않고 수동으로 구성하는 예입니다. `@EnableBatchProcessing`이 제공하는 `jobLauncher`를 커스터마이징 하려면,*

`*JobLauncher` 빈을 직접 재정의하거나 `JobLauncherApplicationRunner` 같은 다른 메커니즘을 고려해야 할 수 있습니다.)*

```java
@Configuration
public class AsyncJobLauncherConfig {

    @Autowired
    private JobRepository jobRepository; // JobRepository 빈 주입

    @Bean
    public JobLauncher asyncJobLauncher() throws Exception { // 빈 이름을 다르게 하거나 @Primary 등으로 기본 JobLauncher를 대체
        TaskExecutorJobLauncher jobLauncher = new TaskExecutorJobLauncher();
        jobLauncher.setJobRepository(jobRepository);
        jobLauncher.setTaskExecutor(new SimpleAsyncTaskExecutor()); // TaskExecutor 설정 (비동기 실행)
        jobLauncher.afterPropertiesSet();
        return jobLauncher;
    }
}
```

Spring `TaskExecutor` 인터페이스의 모든 구현을 사용하여 Job이 비동기적으로 실행되는 방식을 제어할 수 있습니다.

---

## Spring Batch 작업의 시동키, `JobLauncher` 설정하기! 🚀🔑

우리가 만든 멋진 `Job`을 실제로 실행하려면 누군가가 "출발!" 신호를 줘야겠죠? 그 역할을 하는 것이 바로 **`JobLauncher`** 예요.

`JobLauncher`는 우리가 전달한 `Job`과 `JobParameters`를 가지고 `JobRepository`와 통신하며 실제 `JobExecution`을 만들어내고 `Job`을 실행시켜요.

`@EnableBatchProcessing` 어노테이션을 사용하면 `jobLauncher`라는 이름으로 `JobLauncher`의 기본 구현체가 자동으로 빈(Bean)으로 등록돼요. 그래서 대부분의 경우 우리가 직접 `JobLauncher`를 만들 필요는 없어요.

하지만 이 섹션에서는 만약 우리가 직접 `JobLauncher`를 만들거나, 기본 `JobLauncher`의 동작 방식을 바꾸고 싶을 때 어떻게 하는지, 특히 **동기적 실행**과 **비동기적 실행**에 초점을 맞춰서 알아볼 거예요.

---

### 1. `JobLauncher`의 기본 선수: `TaskExecutorJobLauncher`

Spring Batch가 제공하는 `JobLauncher` 인터페이스의 가장 기본적인 구현체는 **`TaskExecutorJobLauncher`** 예요.

이 녀석이 `Job`을 실행하는 실제 선수라고 생각하면 돼요.

- **필수 준비물:** `TaskExecutorJobLauncher`가 제대로 일하려면 딱 하나, **`JobRepository`*가 꼭 필요해요. `JobRepository`를 통해 `JobExecution` 정보를 얻고 저장해야 하거든요.
- **설정 예시 (만약 직접 만든다면 - `@EnableBatchProcessing` 안 쓸 때):***(실제로는 `@EnableBatchProcessing`을 쓰면 `jobLauncher` 빈이 자동으로 만들어지니, 이렇게 직접 만들 일은 드물어요. 이 예제는 `TaskExecutorJobLauncher`의 구조를 이해하기 위한 것이라고 생각해주세요.)*

    ```java
    @Configuration
    public class MyManualJobLauncherConfig {
    
        @Autowired // JobRepository는 어딘가에 빈으로 등록되어 있어야겠죠?
        private JobRepository jobRepository;
    
        @Bean
        public JobLauncher customJobLauncher() throws Exception {
            TaskExecutorJobLauncher jobLauncher = new TaskExecutorJobLauncher();
            jobLauncher.setJobRepository(jobRepository); // JobRepository 연결!
            jobLauncher.afterPropertiesSet(); // "나 준비 다 됐어!" 라고 알려주는 초기화 메서드 호출
            return jobLauncher;
        }
    }
    ```


---

### 2. `JobLauncher`의 기본 동작: 동기(Synchronous) 실행 ⏳

`TaskExecutorJobLauncher`의 기본 동작 방식은 **동기(Synchronous) 실행**이에요.

- **동기 실행이란?**
  - `JobLauncher`의 `run()` 메서드를 호출하면, **해당 `Job`이 완전히 끝날 때까지 기다렸다가** `JobExecution` 결과를 반환해요.
  - 호출한 쪽(예: 스케줄러, 테스트 코드)은 `Job`이 완료될 때까지 아무것도 못 하고 대기해야 해요.
  - Job Launcher 실행 순서 (동기)
    1. 호출자(Client)가 `JobLauncher.run(job, jobParameters)` 호출.
    2. `JobLauncher`는 `JobRepository`에서 `JobExecution`을 가져옴(또는 생성).
    3. `JobLauncher`가 `Job.execute(jobExecution)`을 호출하여 `Job` 실행.
    4. **(`Job`이 끝날 때까지 여기서 대기!)**
    5. `Job`이 완료되면, `JobLauncher`는 최종 `JobExecution` 상태를 `JobRepository`에 업데이트.
    6. `JobLauncher`가 호출자에게 `JobExecution` 반환.
- **언제 유용할까?**
  - **스케줄러**에서 배치 작업을 실행할 때: 스케줄러는 보통 작업이 끝날 때까지 기다렸다가 다음 작업을 진행하거나 성공/실패 여부를 기록해요. 동기 방식이 딱 맞죠!
  - 간단한 테스트 코드에서 `Job`을 실행하고 바로 결과를 확인할 때.

---

### 3. `JobLauncher`를 비동기(Asynchronous)로! ⚡

가끔은 `Job`이 끝날 때까지 마냥 기다릴 수 없는 상황이 있어요. 특히 **웹 애플리케이션에서 HTTP 요청으로 배치 작업을 시작시킬 때**가 대표적이에요.

- **왜 비동기가 필요할까? (HTTP 요청의 경우)**
  - 배치 작업은 보통 몇 분에서 몇 시간까지 오래 걸릴 수 있어요.
  - 만약 HTTP 요청을 처리하는 스레드가 배치 작업이 끝날 때까지 동기적으로 기다린다면, 그동안 해당 HTTP 요청은 응답을 받지 못하고 계속 대기 상태로 남아있게 돼요.
  - 이건 사용자 경험에도 안 좋고, 웹 서버의 자원도 낭비하는 일이에요. (HTTP 타임아웃 발생 가능성도!)
  - 그래서 이럴 때는 `JobLauncher`가 `Job` 실행을 **"시작만 시켜놓고 바로 응답"**을 해주는 비동기 방식이 필요해요. 실제 `Job`은 백그라운드에서 다른 스레드가 처리하도록 하는 거죠.
- **비동기 실행이란?**
  - `JobLauncher.run()` 메서드를 호출하면, `Job` 실행을 **별도의 스레드에 맡기고 즉시 호출자에게 반환**해요. 호출자는 `Job`이 실제로 끝났는지 여부와 상관없이 바로 다음 작업을 진행할 수 있어요.
  - 비동기 Job Launcher 실행 순서
    1. 호출자(Client)가 `JobLauncher.run(job, jobParameters)` 호출.
    2. `JobLauncher`는 `JobRepository`에서 `JobExecution`을 가져옴(또는 생성).
    3. `JobLauncher`가 **`TaskExecutor`에게 `Job` 실행을 요청** (별도 스레드에서 `Job.execute` 실행).
    4. `JobLauncher`는 `Job` 실행이 시작되었다는 정보(아직 완료되지 않은 `JobExecution`)를 **즉시 호출자에게 반환**.
    5. (백그라운드에서 `Job`이 계속 실행됨)
    6. 백그라운드 스레드에서 `Job`이 완료되면, 최종 `JobExecution` 상태가 `JobRepository`에 업데이트됨.
- **어떻게 설정할까? (`TaskExecutor` 사용)**
  - `TaskExecutorJobLauncher`에 `TaskExecutor`라는 것을 설정해주면 비동기로 동작하게 만들 수 있어요.
  - `TaskExecutor`는 Spring 프레임워크에서 비동기 작업 실행을 위한 인터페이스예요. `SimpleAsyncTaskExecutor` (가장 간단한 구현체, 요청마다 새 스레드 생성)나 `ThreadPoolTaskExecutor` (스레드 풀 사용) 등을 사용할 수 있어요.
  - **설정 예시 (만약 직접 만든다면 - `@EnableBatchProcessing` 안 쓸 때):**

      ```java
      @Configuration
      public class MyAsyncJobLauncherConfig {
      
          @Autowired
          private JobRepository jobRepository;
      
          @Bean // 빈 이름을 다르게 하거나 @Primary 등으로 기본 JobLauncher와 구분 필요
          public JobLauncher asyncJobLauncher() throws Exception {
              TaskExecutorJobLauncher jobLauncher = new TaskExecutorJobLauncher();
              jobLauncher.setJobRepository(jobRepository);
              // 여기에 TaskExecutor를 설정!
              jobLauncher.setTaskExecutor(new SimpleAsyncTaskExecutor()); // 이제 비동기로 동작!
              jobLauncher.afterPropertiesSet();
              return jobLauncher;
          }
      }
      
      ```


**핵심:** `TaskExecutorJobLauncher`에 `TaskExecutor`를 설정해주면, `Job` 실행을 **비동기적으로 처리**하여 호출자가 즉시 응답을 받을 수 있게 돼요. 특히 **HTTP 요청으로 배치 작업을 트리거**할 때 매우 유용해요!

---

**`@EnableBatchProcessing`을 사용할 때 `JobLauncher` 커스터마이징은?**

`@EnableBatchProcessing`을 사용하면 `jobLauncher`라는 이름의 `TaskExecutorJobLauncher` 빈이 **기본적으로 동기 방식**으로 생성돼요. 만약 이 기본 `jobLauncher`를 비동기로 바꾸고 싶다면,

1. 위의 예시처럼 **별도의 `JobLauncher` 빈을 다른 이름으로 등록**하고 (예: `asyncJobLauncher`), 비동기가 필요한 곳에서 그 빈을 주입받아 사용하는 방법이 있어요.
2. 또는, Spring Boot 환경이라면 `spring.batch.job.enabled=false`로 자동 구성을 끄고 직접 `JobLauncher`를 포함한 모든 것을 수동으로 구성하거나, `JobLauncherApplicationRunner`의 `TaskExecutor`를 설정하는 등 좀 더 Spring Boot 친화적인 방법을 고려해볼 수 있습니다. (이 부분은 지금 단계에서는 조금 복잡할 수 있어요.)

일반적으로는 **스케줄러에서는 기본 동기 `jobLauncher`를 사용**하고, **웹 요청 등 비동기 실행이 필요한 경우에는 별도의 `TaskExecutor`가 설정된 `JobLauncher` 빈을 만들어 사용**하는 경우가 많습니다.

---
