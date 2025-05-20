---
title: Spring Batch - Advanced Metadata Usage
description: 
author: laze
date: 2025-05-20 00:00:02 +0900
categories: [Dev, SpringBatch]
tags: [SpringBatch]
---
**고급 메타데이터 활용**

지금까지 `JobLauncher`와 `JobRepository` 인터페이스에 대해 논의했습니다.

이 둘은 함께 Job의 단순한 실행과 `JobExecution` 및 `StepExecution`과 같은 배치 도메인 객체의 기본 CRUD 작업을 나타냅니다.

`JobLauncher`는 `JobRepository`를 사용하여 새로운 `JobExecution` 객체를 생성하고 실행합니다.

`Job` 및 `Step` 구현은 나중에 `Job` 실행 중에 동일한 실행에 대한 기본 업데이트를 위해 동일한 `JobRepository`를 사용합니다.

기본 작업은 간단한 시나리오에 충분합니다.

그러나 수백 개의 배치 작업과 복잡한 스케줄링 요구 사항이 있는 대규모 배치 환경에서는 메타데이터에 대한 더 고급 액세스가 필요합니다.

다음 섹션에서 논의할 `JobExplorer` 및 `JobOperator` 인터페이스는 메타데이터를 쿼리하고 제어하기 위한 추가 기능을 추가합니다.

**리포지토리 쿼리하기**

고급 기능 이전에 가장 기본적인 요구 사항은 기존 실행에 대해 리포지토리를 쿼리하는 기능입니다.

이 기능은 `JobExplorer` 인터페이스에 의해 제공됩니다.

```java
public interface JobExplorer {

    List<JobInstance> getJobInstances(String jobName, int start, int count);

    JobExecution getJobExecution(Long executionId);

    StepExecution getStepExecution(Long jobExecutionId, Long stepExecutionId);

    JobInstance getJobInstance(Long instanceId);

    List<JobExecution> getJobExecutions(JobInstance jobInstance);

    Set<JobExecution> findRunningJobExecutions(String jobName);
}

```

메서드 시그니처에서 알 수 있듯이, `JobExplorer`는 `JobRepository`의 읽기 전용 버전이며, `JobRepository`와 마찬가지로 팩토리 빈을 사용하여 쉽게 구성할 수 있습니다.

다음 예는 Java에서 `JobExplorer`를 구성하는 방법을 보여줍니다.
*(이 코드는 `DefaultBatchConfiguration`을 확장하는 클래스 내부에 있을 것이라고 가정하고 있습니다. `@EnableBatchProcessing`을 사용하면 `jobExplorer` 빈이 자동으로 등록됩니다.)*

```java
// ...
// 이 코드는 DefaultBatchConfiguration 확장 클래스 내에 위치합니다.
// this.dataSource는 DefaultBatchConfiguration이 가지고 있는 DataSource를 참조합니다.
@Bean
public JobExplorer jobExplorer() throws Exception {
	JobExplorerFactoryBean factoryBean = new JobExplorerFactoryBean();
	factoryBean.setDataSource(this.dataSource); // DefaultBatchConfiguration의 dataSource 사용
	return factoryBean.getObject();
}
// ...
```

이 장 앞부분에서, 다른 버전이나 스키마를 허용하기 위해 `JobRepository`의 테이블 접두사를 수정할 수 있다고 언급했습니다.

`JobExplorer`는 동일한 테이블에서 작동하므로 접두사를 설정하는 기능도 필요합니다.

다음 예는 Java에서 `JobExplorer`의 테이블 접두사를 설정하는 방법을 보여줍니다.
*(위와 마찬가지로 `DefaultBatchConfiguration` 확장 클래스 내의 코드 예시입니다.)*

```java
// ...
// 이 코드는 DefaultBatchConfiguration 확장 클래스 내에 위치합니다.
@Bean
public JobExplorer jobExplorer() throws Exception {
	JobExplorerFactoryBean factoryBean = new JobExplorerFactoryBean();
	factoryBean.setDataSource(this.dataSource);
	factoryBean.setTablePrefix("SYSTEM."); // 테이블 접두사 설정
	return factoryBean.getObject();
}
// ...
```

**JobRegistry**

`JobRegistry`(및 부모 인터페이스인 `JobLocator`)는 필수는 아니지만, 컨텍스트에서 사용 가능한 Job을 추적하려는 경우 유용할 수 있습니다.

또한 다른 곳(예: 자식 컨텍스트)에서 생성된 Job을 애플리케이션 컨텍스트에서 중앙 집중식으로 수집하는 데도 유용합니다.

사용자 정의 `JobRegistry` 구현을 사용하여 등록된 Job의 이름 및 기타 속성을 조작할 수도 있습니다.

프레임워크에서 제공하는 구현은 하나뿐이며, 이는 Job 이름에서 Job 인스턴스로의 간단한 맵을 기반으로 합니다.

`@EnableBatchProcessing`을 사용하면 `JobRegistry`가 자동으로 제공됩니다.

다음 예는 사용자 정의 `JobRegistry`를 구성하는 방법을 보여줍니다.
*(`@EnableBatchProcessing`을 사용하면 이미 `jobRegistry` 빈이 제공되므로, 이를 커스터마이징하려면 `DefaultBatchConfiguration`을 확장하고 해당 빈 정의 메서드를 오버라이드해야 합니다.)*

```java
// ...
// 이는 @EnableBatchProcessing을 통해 이미 제공되지만
// DefaultBatchConfiguration에서 빈을 오버라이드하여 사용자 정의할 수 있습니다.
@Override
@Bean
public JobRegistry jobRegistry() throws Exception {
	return new MapJobRegistry();
}
// ...
```

다음 방법 중 하나로 `JobRegistry`를 채울 수 있습니다: 빈 후처리기(bean post processor) 사용, 스마트 초기화 싱글톤(smart initializing singleton) 사용 또는 등록기관 라이프사이클 컴포넌트(registrar lifecycle component) 사용.

**JobRegistryBeanPostProcessor**

이는 생성될 때 모든 Job을 등록할 수 있는 빈 후처리기입니다.

다음 예는 Java로 정의된 Job에 `JobRegistryBeanPostProcessor`를 포함하는 방법을 보여줍니다.
*(`@EnableBatchProcessing` 환경에서는 일반적으로 `jobRegistry` 빈이 이미 존재하므로 이를 주입받아 사용합니다.)*

[Java 구성]

```java
@Bean
public JobRegistryBeanPostProcessor jobRegistryBeanPostProcessor(JobRegistry jobRegistry) { // jobRegistry 빈 주입
    JobRegistryBeanPostProcessor postProcessor = new JobRegistryBeanPostProcessor();
    postProcessor.setJobRegistry(jobRegistry);
    return postProcessor;
}
```

반드시 필요한 것은 아니지만, 예제의 후처리기는 ID가 부여되어 자식 컨텍스트(예: 부모 빈 정의)에 포함될 수 있으며, 거기서 생성된 모든 Job도 자동으로 등록되도록 합니다.

**폐기 예정 (Deprecation)**

버전 5.2부터 `JobRegistryBeanPostProcessor` 클래스는 `JobRegistrySmartInitializingSingleton`을 선호하여 폐기될 예정입니다.

**JobRegistrySmartInitializingSingleton**

이는 Job 레지스트리 내에 모든 싱글톤 Job을 등록하는 `SmartInitializingSingleton`입니다.

다음 예는 Java에서 `JobRegistrySmartInitializingSingleton`을 정의하는 방법을 보여줍니다.

```java
@Bean
public JobRegistrySmartInitializingSingleton jobRegistrySmartInitializingSingleton(JobRegistry jobRegistry) {
    return new JobRegistrySmartInitializingSingleton(jobRegistry);
}
```

**AutomaticJobRegistrar**

이는 자식 컨텍스트를 생성하고 해당 컨텍스트에서 생성될 때 Job을 등록하는 라이프사이클 구성 요소입니다.

이렇게 하는 한 가지 이점은 자식 컨텍스트의 Job 이름이 레지스트리에서 전역적으로 고유해야 하지만 해당 종속성은 "자연스러운" 이름을 가질 수 있다는 것입니다.

예를 들어, 각각 하나의 Job만 있지만 `reader`와 같이 동일한 빈 이름을 가진 `ItemReader`의 다른 정의를 모두 가진 XML 구성 파일 세트를 만들 수 있습니다.

모든 해당 파일이 동일한 컨텍스트로 가져온 경우 `reader` 정의가 충돌하여 서로를 재정의하지만 자동 등록기관을 사용하면 이를 피할 수 있습니다.

이렇게 하면 애플리케이션의 별도 모듈에서 제공된 Job을 더 쉽게 통합할 수 있습니다/

다음 예는 Java로 정의된 Job에 `AutomaticJobRegistrar`를 포함하는 방법을 보여줍니다.

```java
// ApplicationContextFactory 와 JobLoader 빈이 필요합니다.
@Bean
public ApplicationContextFactory[] applicationContextFactories() {
    ClassPathXmlApplicationContextFactory factory = new ClassPathXmlApplicationContextFactory();
    factory.setPath("classpath:/com/example/job1.xml"); // 예시 경로
    return new ApplicationContextFactory[] { factory };
}

@Bean
public JobLoader jobLoader(JobRegistry jobRegistry) { // jobRegistry 빈 주입
    DefaultJobLoader jobLoader = new DefaultJobLoader();
    jobLoader.setJobRegistry(jobRegistry);
    return jobLoader;
}

@Bean
public AutomaticJobRegistrar registrar(JobLoader jobLoader, ApplicationContextFactory[] applicationContextFactories) {
    AutomaticJobRegistrar registrar = new AutomaticJobRegistrar();
    registrar.setJobLoader(jobLoader);
    registrar.setApplicationContextFactories(applicationContextFactories);
    // registrar.afterPropertiesSet(); // @Bean 메서드에서는 Spring이 라이프사이클 관리
    return registrar;
}
```

등록기관에는 두 가지 필수 속성이 있습니다: `ApplicationContextFactory` 배열(앞의 예에서 편리한 팩토리 빈에서 생성됨)과 `JobLoader`.

`JobLoader`는 자식 컨텍스트의 라이프사이클을 관리하고 `JobRegistry`에 Job을 등록하는 역할을 합니다.

`ApplicationContextFactory`는 자식 컨텍스트를 생성하는 역할을 합니다.

가장 일반적인 사용법은 (앞의 예에서와 같이) `ClassPathXmlApplicationContextFactory`를 사용하는 것입니다.

이 팩토리의 기능 중 하나는 기본적으로 부모 컨텍스트에서 자식 컨텍스트로 일부 구성을 복사한다는 것입니다.

따라서 예를 들어, 부모와 동일해야 하는 경우 자식에서 `PropertyPlaceholderConfigurer` 또는 AOP 구성을 다시 정의할 필요가 없습니다.

`AutomaticJobRegistrar`는 (`DefaultJobLoader`도 사용하는 한) `JobRegistryBeanPostProcessor`와 함께 사용할 수 있습니다.

예를 들어, 주 부모 컨텍스트와 자식 위치 모두에 Job이 정의된 경우 바람직할 수 있습니다.

**JobOperator**

앞서 논의했듯이, `JobRepository`는 메타데이터에 대한 CRUD 작업을 제공하고, `JobExplorer`는 메타데이터에 대한 읽기 전용 작업을 제공합니다.

그러나 이러한 작업은 배치 운영자가 일반적으로 수행하는 것처럼 Job 중지, 재시작 또는 요약과 같은 일반적인 모니터링 작업을 수행하기 위해 함께 사용할 때 가장 유용합니다.

Spring Batch는 `JobOperator` 인터페이스에서 이러한 유형의 작업을 제공합니다.

```java
public interface JobOperator {

    List<Long> getExecutions(long instanceId) throws NoSuchJobInstanceException;

    List<Long> getJobInstances(String jobName, int start, int count)
          throws NoSuchJobException;

    Set<Long> getRunningExecutions(String jobName) throws NoSuchJobException;

    String getParameters(long executionId) throws NoSuchJobExecutionException;

    Long start(String jobName, String parameters) // JobParameters를 문자열로 받음
          throws NoSuchJobException, JobInstanceAlreadyExistsException;

    Long restart(long executionId)
          throws JobInstanceAlreadyCompleteException, NoSuchJobExecutionException,
                  NoSuchJobException, JobRestartException;

    Long startNextInstance(String jobName) // 다음 JobInstance 실행
          throws NoSuchJobException, JobParametersNotFoundException, JobRestartException,
                 JobExecutionAlreadyRunningException, JobInstanceAlreadyCompleteException;

    boolean stop(long executionId)
          throws NoSuchJobExecutionException, JobExecutionNotRunningException;

    String getSummary(long executionId) throws NoSuchJobExecutionException;

    Map<Long, String> getStepExecutionSummaries(long executionId)
          throws NoSuchJobExecutionException;

    Set<String> getJobNames();

}
```

위의 작업은 `JobLauncher`, `JobRepository`, `JobExplorer`, `JobRegistry`와 같은 여러 다른 인터페이스의 메서드를 나타냅니다.

이러한 이유로 제공된 `JobOperator` 구현(`SimpleJobOperator`)에는 많은 종속성이 있습니다.

다음 예는 Java에서 `SimpleJobOperator`의 일반적인 빈 정의를 보여줍니다.
*(`@EnableBatchProcessing`을 사용하면 `jobOperator` 빈이 자동으로 등록되므로, 수동으로 정의할 필요가 거의 없습니다.)*

```java
 /**
  * 이 빈에 대한 모든 주입된 종속성은 @EnableBatchProcessing
  * 인프라에서 즉시 제공됩니다.
  */
 @Bean
 public SimpleJobOperator jobOperator(JobExplorer jobExplorer,
                                JobRepository jobRepository,
                                JobRegistry jobRegistry,
                                JobLauncher jobLauncher) { // 필요한 빈들을 주입받음

	SimpleJobOperator jobOperator = new SimpleJobOperator();
	jobOperator.setJobExplorer(jobExplorer);
	jobOperator.setJobRepository(jobRepository);
	jobOperator.setJobRegistry(jobRegistry);
	jobOperator.setJobLauncher(jobLauncher);

	return jobOperator;
 }
```

버전 5.0부터 `@EnableBatchProcessing` 어노테이션은 애플리케이션 컨텍스트에 `jobOperator` 빈을 자동으로 등록합니다.

Job 리포지토리에 테이블 접두사를 설정한 경우 Job 탐색기에도 설정하는 것을 잊지 마십시오.

**JobParametersIncrementer**

`JobOperator`의 대부분의 메서드는 자명하며 인터페이스의 Javadoc에서 더 자세한 설명을 찾을 수 있습니다.

그러나 `startNextInstance` 메서드는 주목할 가치가 있습니다.

이 메서드는 항상 Job의 새 인스턴스를 시작합니다. 이는 `JobExecution`에 심각한 문제가 있고 Job을 처음부터 다시 시작해야 하는 경우 매우 유용할 수 있습니다.

`JobLauncher`(매개변수가 이전 매개변수 세트와 다른 경우 새 `JobInstance`를 트리거하는 새 `JobParameters` 객체가 필요함)와 달리

`startNextInstance` 메서드는 `Job`에 연결된 `JobParametersIncrementer`를 사용하여 Job을 새 인스턴스로 강제 실행합니다.

```java
public interface JobParametersIncrementer {

    JobParameters getNext(JobParameters parameters);

}

```

`JobParametersIncrementer`의 계약은 `JobParameters` 객체가 주어지면 포함할 수 있는 필요한 값을 증가시켜 "다음" `JobParameters` 객체를 반환한다는 것입니다.

이 전략은 프레임워크가 `JobParameters`의 어떤 변경 사항이 "다음" 인스턴스가 되는지 알 수 없기 때문에 유용합니다.

예를 들어, `JobParameters`의 유일한 값이 날짜이고 다음 인스턴스를 생성해야 하는 경우 해당 값을 1일 또는 1주(예: Job이 주간인 경우)만큼 증가시켜야 합니까?

다음 예와 같이 Job을 식별하는 데 도움이 되는 모든 숫자 값에 대해서도 마찬가지입니다.

```java
public class SampleIncrementer implements JobParametersIncrementer {

    public JobParameters getNext(JobParameters parameters) {
        if (parameters==null || parameters.isEmpty()) {
            return new JobParametersBuilder().addLong("run.id", 1L).toJobParameters();
        }
        long id = parameters.getLong("run.id",1L) + 1; // 기존 run.id에 1을 더함
        return new JobParametersBuilder().addLong("run.id", id).toJobParameters();
    }
}
```

이 예에서 `run.id` 키를 가진 값은 `JobInstance`를 구별하는 데 사용됩니다.

전달된 `JobParameters`가 `null`이면 Job이 이전에 실행된 적이 없다고 가정할 수 있으므로 초기 상태를 반환할 수 있습니다.

그러나 그렇지 않은 경우 이전 값을 가져와 1을 더한 후 반환합니다.

Java로 정의된 Job의 경우 다음과 같이 빌더에 제공된 `incrementer` 메서드를 통해 `Job`에 증분기(incrementer)를 연결할 수 있습니다.

```java
@Bean
public Job footballJob(JobRepository jobRepository, JobParametersIncrementer sampleIncrementer) { // Incrementer 주입
    return new JobBuilder("footballJob", jobRepository)
    				 .incrementer(sampleIncrementer) // Job에 Incrementer 설정
    				 ...
                     .build();
}
```

**Job 중지하기**

`JobOperator`의 가장 일반적인 사용 사례 중 하나는 Job을 정상적으로 중지하는 것입니다.

```java
Set<Long> executions = jobOperator.getRunningExecutions("sampleJob");
if (!executions.isEmpty()) { // 실행 중인 Job이 있는지 확인
    jobOperator.stop(executions.iterator().next());
}
```

종료는 즉각적이지 않습니다.

특히 실행이 현재 프레임워크가 제어할 수 없는 개발자 코드(예: 비즈니스 서비스)에 있는 경우 즉시 종료를 강제할 방법이 없기 때문입니다.

그러나 제어가 프레임워크로 돌아오자마자 현재 `StepExecution`의 상태를 `BatchStatus.STOPPED`로 설정하고 저장한 다음, 완료하기 전에 `JobExecution`에 대해서도 동일하게 수행합니다.

**Job 중단시키기 (Aborting a Job)**

`FAILED` 상태인 Job 실행은 (Job이 재시작 가능한 경우) 재시작할 수 있습니다.

상태가 `ABANDONED`인 Job 실행은 프레임워크에서 재시작할 수 없습니다.

`ABANDONED` 상태는 재시작된 Job 실행에서 건너뛸 수 있도록 표시하기 위해 Step 실행에서도 사용됩니다.

Job이 실행 중이고 이전 실패한 Job 실행에서 `ABANDONED`로 표시된 Step을 만나면 다음 Step으로 이동합니다 (Job 흐름 정의 및 Step 실행 종료 상태에 따라 결정됨).

프로세스가 중단된 경우(kill -9 또는 서버 장애), Job은 물론 실행 중이 아니지만, 프로세스가 중단되기 전에 아무도 알려주지 않았기 때문에 `JobRepository`는 알 방법이 없습니다.

실행이 실패했거나 중단된 것으로 간주해야 한다고 수동으로 알려야 합니다 (상태를 `FAILED` 또는 `ABANDONED`로 변경).

이는 비즈니스 결정이며 자동화할 방법이 없습니다.

재시작 가능하고 재시작 데이터가 유효하다는 것을 알고 있는 경우에만 상태를 `FAILED`로 변경하십시오.

---

## Spring Batch 고급 메타데이터 활용법: 탐험하고, 등록하고, 조작하기! 🗺️⚙️

우리가 `JobLauncher`로 `Job`을 실행하고, `JobRepository`가 그 모든 기록을 저장한다는 것은 이미 알고 있죠?

하지만 때로는 저장된 기록을 단순히 보고 싶거나, 실행 중인 `Job`을 멈추게 하고 싶거나, 어떤 `Job`들이 있는지 목록을 보고 싶을 때도 있을 거예요.

이런 고급 기능들을 위해 Spring Batch는 몇 가지 특별한 도구들을 제공합니다.

---

### 1. `JobExplorer`: 배치 기록 탐험가! 🕵️‍♂️

- **역할:** `JobRepository`에 저장된 배치 메타데이터를 **읽기 전용(read-only)**으로 조회하는 기능을 제공해요. 마치 도서관 사서에게 "이런 책 찾아주세요" 하는 것처럼, 배치 실행 기록을 안전하게 열람할 수 있게 해줘요.
- **주요 기능 (메서드 이름으로 유추해 보세요!):**
  - `getJobInstances(jobName, start, count)`: 특정 `Job` 이름으로 `JobInstance` 목록 조회 (페이징 가능)
  - `getJobExecution(executionId)`: 특정 `JobExecution` ID로 상세 정보 조회
  - `getStepExecution(jobExecutionId, stepExecutionId)`: 특정 `StepExecution` 상세 정보 조회
  - `getJobInstance(instanceId)`: 특정 `JobInstance` ID로 상세 정보 조회
  - `getJobExecutions(jobInstance)`: 특정 `JobInstance`에 속한 모든 `JobExecution` 목록 조회
  - `findRunningJobExecutions(jobName)`: 현재 실행 중인 특정 `Job`의 `JobExecution` 목록 조회
- **설정:**
  - `@EnableBatchProcessing`을 사용하면 `jobExplorer`라는 이름의 빈이 **자동으로 등록**돼요! (우리가 직접 만들 필요 거의 없음)
  - 만약 `JobRepository`처럼 테이블 접두사(`tablePrefix`)를 변경했다면, `JobExplorer`에도 동일한 접두사를 설정해줘야 해요. (같은 테이블을 봐야 하니까요!) `DefaultBatchConfiguration`을 상속받아 `jobExplorer()` 메서드를 오버라이드하고 `factoryBean.setTablePrefix(...)`를 설정할 수 있어요.

    ```java
    // DefaultBatchConfiguration 상속 시 JobExplorer 테이블 접두사 변경 예시
    @Override
    @Bean
    public JobExplorer jobExplorer() throws Exception {
        JobExplorerFactoryBean factoryBean = new JobExplorerFactoryBean();
        factoryBean.setDataSource(this.dataSource); // 부모 클래스의 DataSource 사용
        factoryBean.setTablePrefix("MYAPP_BATCH_"); // JobRepository와 동일한 접두사 설정
        return factoryBean.getObject();
    }
    
    ```


**핵심:** `JobExplorer`는 `JobRepository`의 **읽기 전용 창구**예요. 배치 실행 기록을 안전하게 조회하고 싶을 때 사용해요.

---

### 2. `JobRegistry`: 우리 동네 Job 명단 관리소! 📋

- **역할:** 애플리케이션 컨텍스트 내에 어떤 `Job`들이 "등록"되어 있는지 **추적하고 관리**하는 역할을 해요. 필수는 아니지만, 여러 곳에서 만들어진 `Job`들을 한 곳에서 파악하거나, 동적으로 `Job`을 찾아 실행해야 할 때 유용해요.
- **주요 기능:**
  - `Job` 이름으로 `Job` 객체 찾기 (`JobLocator` 인터페이스 기능)
  - 등록된 `Job`들의 이름 목록 가져오기
- **설정:**
  - `@EnableBatchProcessing`을 사용하면 `jobRegistry`라는 이름의 빈(보통 `MapJobRegistry` 구현체)이 **자동으로 등록**돼요.
- **`JobRegistry`에 Job을 등록하는 방법들:**
  1. **`JobRegistryBeanPostProcessor` (폐기 예정):** 예전 방식. Spring이 빈을 만들 때 `Job` 타입의 빈을 자동으로 `JobRegistry`에 등록해줬어요. (v5.2부터 폐기 예정)
  2. **`JobRegistrySmartInitializingSingleton` (권장):** 현재 권장되는 방식. Spring 컨테이너가 모든 싱글톤 빈 초기화를 마친 후, `Job` 타입의 빈들을 찾아서 `JobRegistry`에 등록해줘요.

      ```java
      // @EnableBatchProcessing을 쓰면 보통 자동으로 해주지만, 수동 설정 예시
      @Bean
      public JobRegistrySmartInitializingSingleton jobRegistrySmartInitializingSingleton(JobRegistry jobRegistry) {
          return new JobRegistrySmartInitializingSingleton(jobRegistry);
      }
      ```

  3. **`AutomaticJobRegistrar` (고급, 모듈화):** 여러 개의 XML 설정 파일이나 자식 컨텍스트에 각각 `Job`들이 정의되어 있을 때, 이들을 중앙 `JobRegistry`에 자동으로 등록해주는 복잡한 메커니즘이에요. 각 모듈이 독립적인 의존성을 가지면서도 `Job`들을 통합 관리할 수 있게 해줘요. (설정이 다소 복잡해서 특별한 경우에 사용)

**핵심:** `JobRegistry`는 애플리케이션에 **어떤 Job들이 있는지 목록을 관리**하는 역할을 해요. 대부분은 `@EnableBatchProcessing`이 알아서 잘 처리해준답니다.

---

### 3. `JobOperator`: 배치 작업 지휘관! 👨‍✈️

- **역할:** `JobRepository` (쓰기), `JobExplorer` (읽기), `JobLauncher` (실행), `JobRegistry` (목록)의 기능들을 조합해서, 배치 작업을 **운영하고 관리하는 데 필요한 다양한 명령**들을 제공해요. 마치 관제탑에서 비행기 이착륙을 지시하는 지휘관 같아요!
- **주요 기능 (엄청 많아요!):**
  - `getExecutions(instanceId)`: 특정 `JobInstance`의 모든 `JobExecution` ID 목록 조회
  - `getJobInstances(jobName, start, count)`: 특정 `Job`의 `JobInstance` ID 목록 조회
  - `getRunningExecutions(jobName)`: 실행 중인 `JobExecution` ID 목록 조회
  - `getParameters(executionId)`: 특정 `JobExecution`의 `JobParameters` 문자열 조회
  - `start(jobName, parameters)`: 특정 `Job`을 주어진 파라미터(문자열 형태)로 **새로운 `JobInstance`로 실행** (파라미터가 기존과 다르면)
  - `restart(executionId)`: 실패했거나 중단된 `JobExecution`을 **재시작**
  - `startNextInstance(jobName)`: 특정 `Job`의 **"다음" `JobInstance`를 실행** (아래 `JobParametersIncrementer`와 함께 사용)
  - `stop(executionId)`: 실행 중인 `JobExecution`을 **정상적으로 중지 요청**
  - `getSummary(executionId)`: `JobExecution`의 요약 정보 문자열 조회
  - `getStepExecutionSummaries(executionId)`: `JobExecution`에 속한 `StepExecution`들의 요약 정보 조회
  - `getJobNames()`: `JobRegistry`에 등록된 모든 `Job` 이름 목록 조회
- **설정:**
  - `@EnableBatchProcessing`을 사용하면 (Spring Batch 5.0부터) `jobOperator`라는 이름의 빈(`SimpleJobOperator` 구현체)이 **자동으로 등록**돼요! 필요한 의존성(`JobExplorer`, `JobRepository`, `JobRegistry`, `JobLauncher`)도 알아서 주입해줘요.
  - `JobRepository`나 `JobExplorer`에 테이블 접두사를 설정했다면, `SimpleJobOperator`를 수동으로 빈 등록할 때 해당 접두사가 설정된 `JobRepository`와 `JobExplorer`를 주입해줘야 해요.

**핵심:** `JobOperator`는 배치 작업을 **모니터링하고 제어하는 데 필요한 거의 모든 운영 기능을 제공**하는 강력한 도구예요. 외부 관리 도구나 API를 만들 때 아주 유용해요.

---

### 4. `JobParametersIncrementer`: "다음 실행 번호는 뭐지?" 🔢

`JobOperator`의 `startNextInstance(jobName)` 메서드는 "다음" `JobInstance`를 실행한다고 했죠? 그런데 "다음"이라는 게 뭘까요? 날짜를 하루 더해야 할까요, 아니면 어떤 숫자를 1 증가시켜야 할까요? Spring Batch는 이걸 알 수 없어요.

이때 사용하는 것이 바로 **`JobParametersIncrementer`** 예요!

- **역할:** 기존 `JobParameters`를 받아서, **"다음" `JobInstance`를 위한 새로운 `JobParameters`를 생성**해주는 역할을 해요.
- **인터페이스:**

    ```java
    public interface JobParametersIncrementer {
        JobParameters getNext(JobParameters parameters);
    }
    ```

- **구현 예시:** `run.id`라는 파라미터를 1씩 증가시키는 간단한 Incrementer

    ```java
    public class SampleIncrementer implements JobParametersIncrementer {
        public JobParameters getNext(JobParameters parameters) {
            if (parameters == null || parameters.isEmpty()) {
                // 처음 실행이면 run.id=1L 로 시작
                return new JobParametersBuilder().addLong("run.id", 1L).toJobParameters();
            }
            // 기존 run.id 값을 가져와서 1 증가
            long id = parameters.getLong("run.id", 0L) + 1L; // 기본값을 0L로 하여 null 처리 간소화
            return new JobParametersBuilder().addLong("run.id", id).toJobParameters();
        }
    }
    
    ```

- **Job에 연결하기:**
  - Java 설정: `JobBuilder`의 `.incrementer()` 메서드를 사용해요.

      ```java
      @Bean
      public Job footballJob(JobRepository jobRepository, JobParametersIncrementer sampleIncrementer) {
          return new JobBuilder("footballJob", jobRepository)
                           .incrementer(sampleIncrementer) // 이 Job은 sampleIncrementer를 사용해서 다음 파라미터를 생성!
                           // ...
                           .build();
      }
      ```

  - XML 설정: `<job incrementer="sampleIncrementerBean">` 속성을 사용해요.

**핵심:** `JobParametersIncrementer`를 사용하면 `JobOperator.startNextInstance()` 호출 시 **규칙적으로 `JobParameters`를 변경하여 새로운 `JobInstance`를 쉽게 실행**할 수 있어요. (예: 매일 실행되는 작업에 날짜 파라미터 자동 증가)

---

### 5. Job 중지하기 (Stopping a Job) vs. 중단시키기 (Aborting a Job)

`JobOperator`를 통해 실행 중인 `Job`을 제어할 수 있는데, 두 가지 상황을 구분해야 해요.

- **`jobOperator.stop(executionId)` (정상 중지 요청):**
  - 실행 중인 `JobExecution`에게 "이제 그만 멈춰줘" 라고 신호를 보내는 거예요.
  - **즉시 멈추는 건 아니에요!** 현재 `Step`의 진행 중인 청크(Chunk) 처리나 `Tasklet` 실행이 끝나고 프레임워크로 제어권이 돌아오면, 그때 `StepExecution`과 `JobExecution`의 상태를 `STOPPED`로 변경하고 안전하게 종료해요.
  - 마치 운전 중인 사람에게 "다음 휴게소에서 멈춰줘" 라고 하는 것과 같아요.
- **프로세스 강제 종료 (예: `kill -9`, 서버 다운) 후 상태 변경:**
  - 만약 배치 프로세스가 예기치 않게 죽어버리면, `JobRepository`는 이 사실을 몰라요. `JobExecution` 상태는 여전히 `STARTED`로 남아있겠죠.
  - 이런 경우, 운영자가 **수동으로** 해당 `JobExecution`의 상태를 변경해줘야 해요.
    - **`FAILED`:** 만약 작업이 재시작 가능하고, 저장된 재시작 데이터가 유효하다고 판단되면 `FAILED`로 변경해서 나중에 재시작할 수 있게 해요.
    - **`ABANDONED`:** 만약 재시작할 수 없거나, 해당 실행을 완전히 포기해야 한다면 `ABANDONED`로 변경해요. `ABANDONED` 상태의 `StepExecution`은 재시작 시 건너뛰게 돼요.
  - 이건 **비즈니스적인 판단**이 필요한 부분이라 자동화하기 어려워요.

**핵심:** `JobOperator.stop()`은 **정상적인 종료 요청**이고, 프로세스 강제 종료 시에는 **수동으로 상태를 `FAILED` 또는 `ABANDONED`로 변경**해야 해요.

---
