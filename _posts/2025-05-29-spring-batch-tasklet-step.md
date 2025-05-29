---
title: Spring Batch - Configuring a Step (TaskletStep)
description: 
author: laze
date: 2025-05-29 00:00:04 +0900
categories: [Dev, SpringBatch]
tags: [SpringBatch]
---
### `TaskletStep`

청크(Chunk) 지향 처리가 `Step`에서 처리하는 유일한 방법은 아닙니다.

만약 `Step`이 저장 프로시저 호출로 구성되어야 한다면 어떨까요?

해당 호출을 `ItemReader`로 구현하고 프로시저가 완료된 후 `null`을 반환하도록 할 수 있습니다.

그러나 그렇게 하는 것은 아무 작업도 하지 않는(no-op) `ItemWriter`가 필요하기 때문에 다소 부자연스럽습니다.

Spring Batch는 이러한 시나리오를 위해 `TaskletStep`을 제공합니다.

`Tasklet` 인터페이스는 `execute`라는 하나의 메소드를 가지고 있으며, 이 메소드는 `RepeatStatus.FINISHED`를 반환하거나 실패를 알리는 예외를 던질 때까지 `TaskletStep`에 의해 반복적으로 호출됩니다.

`Tasklet`에 대한 각 호출은 트랜잭션으로 래핑됩니다.

`Tasklet` 구현체는 저장 프로시저, 스크립트 또는 SQL 업데이트 문을 호출할 수 있습니다.

Java에서 `TaskletStep`을 생성하려면, 빌더의 `tasklet` 메소드에 전달되는 빈(bean)이 `Tasklet` 인터페이스를 구현해야 합니다.

`TaskletStep`을 빌드할 때는 `chunk`에 대한 호출이 없어야 합니다.

다음 예제는 간단한 tasklet을 보여줍니다:

```java
@Bean
public Step step1(JobRepository jobRepository, PlatformTransactionManager transactionManager) {
    return new StepBuilder("step1", jobRepository)
    			.tasklet(myTasklet(), transactionManager) // myTasklet()은 Tasklet 인터페이스를 구현한 빈을 반환해야 합니다.
    			.build();
}
```

만약 `Tasklet` 구현체가 `StepListener` 인터페이스를 구현한다면, `TaskletStep`은 해당 tasklet을 `StepListener`로 자동 등록합니다.

### `TaskletAdapter`

`ItemReader` 및 `ItemWriter` 인터페이스에 대한 다른 어댑터와 마찬가지로, `Tasklet` 인터페이스는 기존 클래스에 자신을 적용(adapt)할 수 있는 구현체인 `TaskletAdapter`를 포함합니다.

이것이 유용할 수 있는 예로는 레코드 집합에 대한 플래그를 업데이트하는 데 사용되는 기존 DAO가 있습니다.

`TaskletAdapter`를 사용하면 `Tasklet` 인터페이스용 어댑터를 작성할 필요 없이 이 클래스를 호출할 수 있습니다.

다음 예제는 Java에서 `TaskletAdapter`를 정의하는 방법을 보여줍니다:

**Java Configuration**

```java
@Bean
public MethodInvokingTaskletAdapter myTasklet() { // 반환 타입이 MethodInvokingTaskletAdapter
	MethodInvokingTaskletAdapter adapter = new MethodInvokingTaskletAdapter();

	adapter.setTargetObject(fooDao()); // 호출할 대상 객체 (예: DAO)
	adapter.setTargetMethod("updateFoo"); // 호출할 대상 객체의 메소드 이름

	return adapter;
}
```

### `Tasklet` 구현 예제

많은 배치 작업에는 주 처리가 시작되기 전에 다양한 리소스를 설정하거나, 처리가 완료된 후 해당 리소스를 정리하기 위해 수행되어야 하는 스텝들이 포함됩니다.

파일 작업을 많이 하는 작업의 경우, 다른 위치로 성공적으로 업로드된 후 특정 파일을 로컬에서 삭제해야 하는 경우가 종종 있습니다.

다음 예제(Spring Batch 샘플 프로젝트에서 가져옴)는 바로 그러한 책임을 가진 `Tasklet` 구현입니다:

```java
public class FileDeletingTasklet implements Tasklet, InitializingBean {

    private Resource directory; // 삭제할 파일이 있는 디렉토리 리소스

    public RepeatStatus execute(StepContribution contribution,
                                ChunkContext chunkContext) throws Exception {
        File dir = directory.getFile();
        // 상태 검증: 제공된 리소스가 디렉토리인지 확인
        Assert.state(dir.isDirectory(), "The resource must be a directory");

        File[] files = dir.listFiles(); // 디렉토리 내 모든 파일 목록 가져오기
        for (int i = 0; i < files.length; i++) {
            boolean deleted = files[i].delete(); // 각 파일 삭제 시도
            if (!deleted) {
                // 파일 삭제 실패 시 예외 발생
                throw new UnexpectedJobExecutionException("Could not delete file " +
                                                          files[i].getPath());
            }
        }
        return RepeatStatus.FINISHED; // 작업 완료 후 FINISHED 상태 반환
    }

    public void setDirectoryResource(Resource directory) {
        this.directory = directory; // 디렉토리 리소스 설정 (setter 주입)
    }

    // InitializingBean 인터페이스 구현: 빈 초기화 시 호출됨
    public void afterPropertiesSet() throws Exception {
        // 필수 프로퍼티(directory)가 설정되었는지 확인
        Assert.state(directory != null, "Directory must be set");
    }
}
```

위의 tasklet 구현은 주어진 디렉토리 내의 모든 파일을 삭제합니다.

`execute` 메소드는 한 번만 호출된다는 점에 유의해야 합니다. 남은 것은 스텝에서 이 tasklet을 참조하는 것입니다.

다음 예제는 Java에서 스텝이 tasklet을 참조하는 방법을 보여줍니다:

**Java Configuration**

```java
@Bean
public Job taskletJob(JobRepository jobRepository, Step deleteFilesInDir) { // Job 정의
	return new JobBuilder("taskletJob", jobRepository)
				.start(deleteFilesInDir) // deleteFilesInDir 스텝으로 시작
				.build();
}

@Bean
public Step deleteFilesInDir(JobRepository jobRepository, PlatformTransactionManager transactionManager) { // Step 정의
	return new StepBuilder("deleteFilesInDir", jobRepository)
				.tasklet(fileDeletingTasklet(), transactionManager) // fileDeletingTasklet()을 tasklet으로 설정
				.build();
}

@Bean
public FileDeletingTasklet fileDeletingTasklet() { // Tasklet 빈 정의
	FileDeletingTasklet tasklet = new FileDeletingTasklet();

    // 삭제할 디렉토리 리소스 설정
	tasklet.setDirectoryResource(new FileSystemResource("target/test-outputs/test-dir"));

	return tasklet;
}
```

---

## 학습 목표 제시

이번 챕터를 통해 학생 여러분은 다음을 이해하고 배울 수 있습니다:

1. **청크 지향 처리가 아닌 단일 작업(Task)을 수행하는 `TaskletStep`의 개념과 사용 사례를 이해합니다.** (예: 시스템 명령어 실행, DB 업데이트, 파일 정리 등)
2. **`Tasklet` 인터페이스의 `execute` 메소드 역할과 `RepeatStatus`의 의미를 파악하고, 간단한 `Tasklet`을 직접 구현할 수 있습니다.**
3. **기존의 일반 Java 클래스 메소드를 `Tasklet`으로 쉽게 사용할 수 있도록 해주는 `TaskletAdapter`(특히 `MethodInvokingTaskletAdapter`)의 사용법을 익힙니다.**

---

### 핵심 개념 설명

**`TaskletStep`이란?**

`TaskletStep`은 Spring Batch에서 **단일 태스크(Task)** 를 실행하는 `Step`을 만들기 위한 간단하고 유연한 방법입니다.

우리가 이전 챕터에서 배운 `Step`은 주로 아이템을 읽고(Read), 처리하고(Process), 쓰는(Write) 청크(Chunk) 지향 방식이었죠? 하지만 `TaskletStep`은 이러한 아이템 기반의 반복적인 처리가 아니라, **한 번의 독립적인 작업**을 수행하는 데 초점을 맞춥니다.

**왜 `TaskletStep`이 필요할까요?**

- **간단한 작업 수행:**
  - 배치 작업 시작 전에 특정 디렉토리를 생성하거나, 작업 완료 후 임시 파일을 삭제하는 경우
  - 데이터베이스의 특정 테이블을 비우거나(truncate), 특정 조건에 맞는 데이터를 한 번에 업데이트하는 SQL을 실행하는 경우
  - 저장 프로시저(Stored Procedure)를 호출하는 경우
  - 외부 시스템 명령어를 실행하는 경우 (예: 셸 스크립트 실행)
  - 배치 작업의 성공/실패 여부를 외부 시스템에 알리는 API를 호출하는 경우
- **청크 모델이 부적합할 때:** 위와 같은 작업들은 여러 아이템을 반복적으로 읽고 쓰는 청크 모델로는 표현하기 어색하거나 비효율적입니다. 억지로 `ItemReader`가 한 번만 동작하고 `null`을 반환하게 하거나, 아무것도 안 하는 `ItemWriter`를 만드는 것은 자연스럽지 않죠. `TaskletStep`은 이런 상황에 딱 맞는 해결책입니다.

**`Tasklet` 인터페이스**

`TaskletStep`의 핵심은 `Tasklet` 인터페이스입니다. 이 인터페이스는 단 하나의 메소드, `execute`를 가지고 있습니다.

```java
public interface Tasklet {
    RepeatStatus execute(StepContribution contribution, ChunkContext chunkContext) throws Exception;
}
```

- **`execute(StepContribution contribution, ChunkContext chunkContext)`**: 이 메소드 안에 우리가 `Step`에서 수행하고 싶은 실제 로직을 구현합니다.
  - `StepContribution`: 현재 `Step` 실행에 대한 기여분(예: 읽은 수, 쓴 수 등)을 업데이트할 수 있는 객체입니다. `Tasklet`에서는 주로 작업 성공/실패 여부나 간단한 카운트 업데이트에 사용될 수 있습니다.
  - `ChunkContext`: 현재 청크(비록 `Tasklet`은 청크 지향이 아니지만, `Step` 실행 컨텍스트 정보를 공유하기 위해 사용됨)와 관련된 정보를 담고 있습니다. `StepExecution` 등에 접근할 수 있습니다.
- **반환 타입 `RepeatStatus`**:
  - `RepeatStatus.FINISHED`: `Tasklet`의 작업이 완료되었음을 나타냅니다. `TaskletStep`은 이 값을 받으면 해당 `Tasklet`의 실행을 종료합니다. 대부분의 경우 이 값을 반환하게 됩니다.
  - `RepeatStatus.CONTINUABLE`: (자주 사용되진 않지만) `Tasklet`의 작업이 아직 끝나지 않았고, `TaskletStep`이 `execute` 메소드를 다시 호출해야 함을 나타냅니다. 예를 들어, 어떤 조건이 충족될 때까지 반복적으로 뭔가를 확인해야 하는 경우에 사용할 수 있습니다.
- **트랜잭션**: `Tasklet`의 `execute` 메소드 호출은 **하나의 트랜잭션 내에서 실행됩니다.** 즉, `execute` 메소드가 시작될 때 트랜잭션이 시작되고, 메소드가 성공적으로 완료되면 커밋, 예외가 발생하면 롤백됩니다. 이는 데이터베이스 작업 등을 `Tasklet` 내에서 수행할 때 원자성을 보장해줍니다.

**`TaskletStep` 생성 방법 (Java Config)**

`StepBuilder`를 사용하여 `TaskletStep`을 만드는 것은 매우 간단합니다. `chunk()` 메소드 대신 `tasklet()` 메소드를 사용하면 됩니다.

```java
@Bean
public Step mySimpleTaskletStep(JobRepository jobRepository,
                               PlatformTransactionManager transactionManager,
                               MyTasklet myTaskletBean) { // (1) Tasklet 구현체 빈 주입
    return new StepBuilder("mySimpleTaskletStep", jobRepository)
                .tasklet(myTaskletBean, transactionManager) // (2) tasklet() 메소드 사용
                // .chunk()는 사용하지 않습니다!
                .build();
}

@Bean
public MyTasklet myTaskletBean() { // (3) Tasklet 인터페이스를 구현한 빈
    return new MyTasklet();
}
```

1. **(1) `MyTasklet myTaskletBean`**: `Tasklet` 인터페이스를 구현한 빈을 주입받습니다.
2. **(2) `.tasklet(myTaskletBean, transactionManager)`**: `StepBuilder`의 `tasklet()` 메소드에 `Tasklet` 구현체 빈과 `PlatformTransactionManager`를 전달합니다.
3. **(3) `MyTasklet` 빈 정의**: `Tasklet` 인터페이스를 구현한 클래스를 빈으로 등록합니다.

**`TaskletAdapter` (특히 `MethodInvokingTaskletAdapter`)**

만약 이미 존재하는 클래스의 특정 메소드를 `Tasklet`처럼 실행하고 싶다면 어떻게 해야 할까요?

예를 들어, 아주 잘 만들어진 `LegacyService` 클래스에 `cleanupOldData()` 라는 메소드가 있다고 해봅시다.

이 메소드를 `TaskletStep`에서 실행하기 위해 `LegacyService`를 수정하거나, `LegacyService`를 감싸는 새로운 `Tasklet` 구현 클래스를 만들 수도 있겠지만, 더 간단한 방법이 있습니다.

바로 `TaskletAdapter`를 사용하는 것입니다.

그중에서도 `MethodInvokingTaskletAdapter`는 특정 객체의 특정 메소드를 호출해주는 편리한 어댑터입니다.

```java
// 기존에 있는 서비스 클래스라고 가정
public class LegacyService {
    public void cleanupOldData() {
        System.out.println("오래된 데이터를 정리합니다...");
        // 실제 정리 로직
    }
}

// Spring Batch 설정 (Java Config)
@Bean
public MethodInvokingTaskletAdapter cleanupTasklet(LegacyService legacyService) { // (1) LegacyService 주입
    MethodInvokingTaskletAdapter adapter = new MethodInvokingTaskletAdapter();
    adapter.setTargetObject(legacyService); // (2) 호출할 대상 객체 설정
    adapter.setTargetMethod("cleanupOldData"); // (3) 호출할 메소드 이름 설정
    // adapter.setArguments(new Object[]{"argument1"}); // 메소드에 인자가 필요하다면 설정
    return adapter;
}

@Bean
public Step cleanupStep(JobRepository jobRepository,
                        PlatformTransactionManager transactionManager,
                        MethodInvokingTaskletAdapter cleanupTasklet) {
    return new StepBuilder("cleanupStep", jobRepository)
                .tasklet(cleanupTasklet, transactionManager)
                .build();
}

@Bean
public LegacyService legacyService() {
    return new LegacyService();
}
```

1. **(1) `LegacyService legacyService`**: 호출 대상 메소드를 가진 객체(`LegacyService`)를 주입받습니다.
2. **(2) `adapter.setTargetObject(legacyService)`**: `MethodInvokingTaskletAdapter`에 어떤 객체의 메소드를 호출할지 알려줍니다.
3. **(3) `adapter.setTargetMethod("cleanupOldData")`**: 호출할 메소드의 이름을 문자열로 지정합니다.

이렇게 하면 `cleanupOldData()` 메소드가 마치 `Tasklet`의 `execute` 메소드처럼 `TaskletStep`에 의해 실행됩니다. 기존 코드를 거의 수정하지 않고도 Spring Batch와 통합할 수 있어 매우 유용합니다.

---

### 주요 용어 해설

- **`Tasklet`:** `Step` 내에서 수행될 단일 작업을 정의하는 인터페이스입니다. `execute` 메소드 하나만을 가집니다.
- **`TaskletStep`:** `Tasklet` 인터페이스의 구현체를 실행하는 `Step`의 한 종류입니다. 청크 지향 처리가 아닌 작업에 사용됩니다.
- **`RepeatStatus`:** `Tasklet`의 `execute` 메소드가 반환하는 열거형(enum)입니다.
  - `FINISHED`: 작업이 완료되어 더 이상 반복할 필요가 없음을 나타냅니다.
  - `CONTINUABLE`: 작업이 아직 완료되지 않아 `execute` 메소드를 다시 호출해야 함을 나타냅니다. (일반적으로는 `FINISHED`가 주로 사용됩니다.)
- **`StepContribution`:** `Tasklet`이 `Step` 실행에 대한 정보를 업데이트할 수 있도록 하는 객체입니다. (예: 필터링된 아이템 수, 스킵된 아이템 수 등). `Tasklet`에서는 사용 빈도가 낮을 수 있지만, 청크 처리에서는 더 활발히 사용됩니다.
- **`ChunkContext`:** 현재 `Step` 실행과 관련된 컨텍스트 정보를 제공합니다. `StepExecution` 등에 접근하여 `JobParameters`나 `ExecutionContext`를 가져올 수 있습니다.
- **`TaskletAdapter`:** 기존 클래스를 `Tasklet` 인터페이스에 맞게 조정해주는 어댑터 클래스입니다.
- **`MethodInvokingTaskletAdapter`:** `TaskletAdapter`의 구체적인 구현체로, 지정된 객체의 특정 메소드를 호출하여 `Tasklet`처럼 동작하게 합니다.
- **`InitializingBean`:** 스프링 프레임워크 인터페이스로, 빈(bean)의 모든 프로퍼티가 설정된 후 초기화 로직을 수행해야 할 때 사용합니다. `afterPropertiesSet()` 메소드를 구현해야 합니다. 예제 코드의 `FileDeletingTasklet`에서 `directory` 리소스가 제대로 설정되었는지 검증하는 데 사용되었습니다.
- **`Resource`:** 스프링 프레임워크에서 파일, 클래스패스 상의 리소스, URL 등 다양한 종류의 리소스를 추상화한 인터페이스입니다. 예제에서는 삭제할 파일들이 있는 디렉토리를 나타내는 데 사용되었습니다 (`FileSystemResource`).

---

### 코드 예제 및 분석

**1. `FileDeletingTasklet` 구현 예제 분석**

```java
public class FileDeletingTasklet implements Tasklet, InitializingBean { // (1)

    private Resource directory; // (2)

    // Tasklet 인터페이스의 execute 메소드 구현
    public RepeatStatus execute(StepContribution contribution,
                                ChunkContext chunkContext) throws Exception { // (3)
        File dir = directory.getFile(); // (4)
        Assert.state(dir.isDirectory(), "The resource must be a directory"); // (5)

        File[] files = dir.listFiles();
        for (int i = 0; i < files.length; i++) {
            boolean deleted = files[i].delete(); // (6)
            if (!deleted) {
                throw new UnexpectedJobExecutionException("Could not delete file " +
                                                          files[i].getPath()); // (7)
            }
        }
        return RepeatStatus.FINISHED; // (8)
    }

    // Setter 메소드: Spring이 directory 프로퍼티를 주입할 때 사용
    public void setDirectoryResource(Resource directory) { // (9)
        this.directory = directory;
    }

    // InitializingBean 인터페이스의 afterPropertiesSet 메소드 구현
    public void afterPropertiesSet() throws Exception { // (10)
        Assert.state(directory != null, "Directory must be set"); // (11)
    }
}
```

- **(1) `implements Tasklet, InitializingBean`**: 이 클래스가 `Tasklet`으로서 `Step`에서 실행될 수 있고, `InitializingBean`으로서 빈 초기화 시점에 특정 로직을 수행할 수 있음을 나타냅니다.
- **(2) `private Resource directory;`**: 삭제할 파일들이 있는 디렉토리를 나타내는 스프링의 `Resource` 객체입니다. 외부(Spring 설정)에서 주입받게 됩니다.
- **(3) `public RepeatStatus execute(...)`**: `Tasklet`의 핵심 로직입니다. 이 `Step`이 실행될 때 이 메소드가 호출됩니다.
- **(4) `File dir = directory.getFile();`**: `Resource` 객체로부터 실제 `java.io.File` 객체를 얻습니다.
- **(5) `Assert.state(dir.isDirectory(), ...);`**: `directory`가 실제로 디렉토리인지 확인합니다. 아니면 `IllegalStateException`을 발생시켜 `Step`을 실패시킵니다. `Assert`는 유효성 검증에 유용한 스프링 유틸리티 클래스입니다.
- **(6) `boolean deleted = files[i].delete();`**: 디렉토리 내의 각 파일을 삭제합니다.
- **(7) `throw new UnexpectedJobExecutionException(...)`**: 파일 삭제에 실패하면 `UnexpectedJobExecutionException` (Spring Batch 예외)을 발생시켜 `Step`을 실패 처리합니다.
- **(8) `return RepeatStatus.FINISHED;`**: 모든 파일이 성공적으로 삭제되면 `RepeatStatus.FINISHED`를 반환하여 `Tasklet` 작업이 완료되었음을 알립니다.
- **(9) `public void setDirectoryResource(Resource directory)`**: Spring DI (Dependency Injection)를 통해 `directory` 프로퍼티 값을 설정받기 위한 setter 메소드입니다.
- **(10) `public void afterPropertiesSet() throws Exception`**: `InitializingBean` 인터페이스의 메소드입니다. 이 빈의 모든 프로퍼티(여기서는 `directory`)가 설정된 후 Spring에 의해 호출됩니다.
- **(11) `Assert.state(directory != null, ...);`**: `directory` 프로퍼티가 반드시 설정되어야 함을 검증합니다. 만약 설정되지 않았다면 `IllegalStateException`을 발생시켜 애플리케이션 컨텍스트 로딩 시점에 문제를 알립니다.

**2. `FileDeletingTasklet`을 사용하는 `Job` 및 `Step` 설정 예제 분석**

```java
@Bean
public Job taskletJob(JobRepository jobRepository, Step deleteFilesInDir) { // (A)
	return new JobBuilder("taskletJob", jobRepository)
				.start(deleteFilesInDir) // (B)
				.build();
}

@Bean
public Step deleteFilesInDir(JobRepository jobRepository, PlatformTransactionManager transactionManager,
                             FileDeletingTasklet fileDeletingTasklet) { // (C) Step에 FileDeletingTasklet 주입
	return new StepBuilder("deleteFilesInDir", jobRepository)
				.tasklet(fileDeletingTasklet, transactionManager) // (D) Tasklet 설정
				.build();
}

@Bean
public FileDeletingTasklet fileDeletingTasklet() { // (E) Tasklet 빈 정의
	FileDeletingTasklet tasklet = new FileDeletingTasklet();
	// 실제 파일 시스템 경로를 사용하여 Resource 객체 생성 및 설정
	tasklet.setDirectoryResource(new FileSystemResource("target/test-outputs/test-dir")); // (F)
	return tasklet;
}
```

- **(A) `public Job taskletJob(...)`**: `"taskletJob"`이라는 이름의 `Job`을 정의합니다.
- **(B) `.start(deleteFilesInDir)`**: 이 `Job`은 `deleteFilesInDir`이라는 `Step`으로 시작합니다.
- **(C) `public Step deleteFilesInDir(...)`**: `"deleteFilesInDir"`이라는 이름의 `Step`을 정의합니다. 이 `Step`은 `fileDeletingTasklet` 빈을 주입받습니다.
- **(D) `.tasklet(fileDeletingTasklet, transactionManager)`**: 주입받은 `fileDeletingTasklet`을 이 `Step`의 `Tasklet`으로 설정합니다. 이 `Step`이 실행되면 `fileDeletingTasklet`의 `execute` 메소드가 호출됩니다.
- **(E) `public FileDeletingTasklet fileDeletingTasklet()`**: `FileDeletingTasklet` 타입의 빈을 생성하고 설정합니다.
- **(F) `tasklet.setDirectoryResource(new FileSystemResource("target/test-outputs/test-dir"));`**: `FileDeletingTasklet`의 `directory` 프로퍼티에 실제 디렉토리 경로를 `FileSystemResource` 형태로 주입합니다. 즉, 이 `Tasklet`은 `"target/test-outputs/test-dir"` 디렉토리 내의 파일들을 삭제하게 됩니다.

**`Tasklet`과 `StepListener` 자동 등록**

문서에 "If it implements the StepListener interface, TaskletStep automatically registers the tasklet as a StepListener." 라는 부분이 있었습니다.

이것은 만약 `FileDeletingTasklet` 클래스가 `Tasklet` 인터페이스뿐만 아니라 `StepExecutionListener`와 같은 `StepListener` 인터페이스도 함께 구현했다면,

```java
public class FileDeletingTasklet implements Tasklet, InitializingBean, StepExecutionListener {
    // ... 기존 코드 ...

    @Override
    public void beforeStep(StepExecution stepExecution) {
        System.out.println("파일 삭제 작업 시작 전!");
    }

    @Override
    public ExitStatus afterStep(StepExecution stepExecution) {
        System.out.println("파일 삭제 작업 완료 후!");
        return null; // 또는 적절한 ExitStatus
    }
}
```

그리고 이 `FileDeletingTasklet` 빈을 `StepBuilder`의 `tasklet()` 메소드에 전달하면,

```java
@Bean
public Step deleteFilesInDir(JobRepository jobRepository, PlatformTransactionManager transactionManager,
                             FileDeletingTasklet fileDeletingTasklet) {
	return new StepBuilder("deleteFilesInDir", jobRepository)
				.tasklet(fileDeletingTasklet, transactionManager) // fileDeletingTasklet은 StepExecutionListener도 구현함
				.build();
}
```

별도로 `.listener(fileDeletingTasklet)`를 호출하지 않아도 `FileDeletingTasklet`의 `beforeStep`과 `afterStep` 메소드가 자동으로 호출된다는 의미입니다.

`Tasklet` 자체가 `Step`의 핵심 컴포넌트이기 때문에, 이것이 리스너 역할도 겸하고 있다면 Spring Batch가 알아서 처리해주는 편리한 기능입니다.

---

### "왜?" 라는 질문에 대한 답변

**Q: `Tasklet`의 `execute` 메소드는 왜 `RepeatStatus`를 반환하나요? 그냥 `void`로 하고, 예외가 없으면 성공, 있으면 실패로 처리하면 안 되나요?**

A: 좋은 질문입니다. 대부분의 `Tasklet`은 한 번 실행되고 `RepeatStatus.FINISHED`를 반환하는 것이 일반적입니다. 하지만 `RepeatStatus`를 사용하는 주된 이유는 `Tasklet`에게 **"아직 작업이 끝나지 않았으니, 나를 다시 한번 호출해줘!"** 라고 `TaskletStep`에게 알릴 수 있는 메커니즘을 제공하기 위해서입니다.

- **`RepeatStatus.CONTINUABLE`의 사용 사례 (이론적):**
  - **폴링(Polling) 작업:** 특정 외부 조건이 만족될 때까지 주기적으로 확인해야 하는 `Tasklet`을 상상해봅시다. 예를 들어, "외부 시스템에서 특정 파일이 생성될 때까지 1초마다 확인하고, 파일이 생성되면 다음 `Step`으로 진행" 같은 시나리오입니다. 이 경우 `Tasklet`은 파일이 아직 없으면 `RepeatStatus.CONTINUABLE`을 반환하고, 파일이 발견되면 `RepeatStatus.FINISHED`를 반환할 수 있습니다. (실제로는 이런 경우 Spring Integration 같은 다른 도구를 고려하거나, `Job` 레벨에서 루프를 도는 것이 더 일반적일 수 있습니다.)
  - **내부 상태에 따른 반복:** `Tasklet` 내부적으로 어떤 카운터가 특정 값에 도달할 때까지 동일한 로직을 여러 번 반복해야 할 때, `CONTINUABLE`을 사용하여 `execute` 메소드가 다시 호출되도록 할 수 있습니다.

물론, 말씀하신 것처럼 `void`로 하고 예외로만 실패를 알리는 방식도 가능했을 것입니다. 하지만 `RepeatStatus`를 통해 `Tasklet` 스스로 자신의 실행 흐름을 좀 더 제어할 수 있는 유연성을 부여한 것이라고 볼 수 있습니다. 하지만 현실적으로 대부분의 `Tasklet`은 한 번의 `execute` 호출로 작업이 완료되므로 `RepeatStatus.FINISHED`를 주로 사용하게 됩니다.

**Q: `TaskletStep`에서도 트랜잭션이 관리되는데, 청크 지향 `Step`의 트랜잭션 관리와 다른 점이 있나요?**

A: 네, 트랜잭션이 관리된다는 점은 같지만, 범위와 주기에 차이가 있습니다.

- **청크 지향 `Step`:**
  - 트랜잭션은 **청크 단위**로 관리됩니다. `commit-interval` 만큼의 아이템이 읽고, 처리되고, 쓰여진 후에 하나의 트랜잭션이 커밋됩니다.
  - 만약 청크 처리 중 오류가 발생하면, 해당 청크 내에서 이미 처리된 아이템들에 대한 작업도 함께 롤백됩니다 (일반적으로).
- **`TaskletStep`:**
  - 트랜잭션은 **`Tasklet`의 `execute` 메소드 한 번의 호출 전체**를 감쌉니다.
  - `execute` 메소드가 시작될 때 트랜잭션이 시작되고, 메소드가 성공적으로 완료되면 커밋됩니다. `execute` 메소드 내에서 예외가 발생하면 트랜잭션 전체가 롤백됩니다.
  - `RepeatStatus.CONTINUABLE`을 반환하여 `execute` 메소드가 여러 번 호출되는 경우, **각 `execute` 호출은 별도의 독립적인 트랜잭션**으로 처리됩니다. 첫 번째 `execute` 호출이 성공적으로 커밋되고, 두 번째 `execute` 호출에서 실패하여 롤백되더라도 첫 번째 호출의 결과는 유지됩니다.

비유하자면,

- **청크 지향 `Step`의 트랜잭션:** 여러 개의 작은 상자(아이템)를 하나의 큰 컨테이너(청크)에 담아 한 번에 배송(커밋)하는 것과 같습니다. 배송 중 문제(오류)가 생기면 컨테이너 전체가 반송(롤백)될 수 있습니다.
- **`TaskletStep`의 트랜잭션:** 하나의 큰 작업(Tasklet의 `execute` 메소드)을 시작부터 끝까지 하나의 묶음으로 처리하는 것과 같습니다. 작업 도중 문제(오류)가 생기면 그 작업 전체가 취소(롤백)됩니다. 만약 이 큰 작업을 여러 번 반복한다면(CONTINUABLE), 각 반복마다 새로운 묶음으로 처리됩니다.

---

### 주의사항 및 Best Practice

1. **`Tasklet`은 간단한 작업에 적합:** `TaskletStep`은 복잡한 데이터 처리 로직보다는 단일 목적의 간단한 작업(파일 정리, DB 초기화, 알림 발송 등)에 사용하는 것이 좋습니다. 대량 데이터 처리는 여전히 청크 지향 `Step`이 더 적합합니다.
2. **`Tasklet` 내 로직은 멱등성(Idempotence)을 고려:** 만약 `Job`이 실패 후 재시작될 때 `TaskletStep`이 다시 실행될 수 있습니다. `Tasklet`의 작업이 여러 번 실행되어도 문제가 없는지(멱등성) 고려하는 것이 좋습니다. 예를 들어, 파일을 삭제하는 `Tasklet`은 이미 삭제된 파일을 다시 삭제하려고 해도 오류가 발생하지 않도록 하거나, 이미 작업이 완료된 상태인지 확인하는 로직을 추가할 수 있습니다.
3. **`MethodInvokingTaskletAdapter`의 유용성:** 기존 코드를 재활용해야 할 때 `MethodInvokingTaskletAdapter`는 매우 강력한 도구입니다. Spring Batch에 맞게 코드를 새로 작성하는 수고를 덜어줍니다.
4. **`RepeatStatus.CONTINUABLE`은 신중하게 사용:** 대부분의 경우 `RepeatStatus.FINISHED`로 충분합니다. `CONTINUABLE`을 사용해야 한다면, `Tasklet`이 무한 루프에 빠지지 않도록 종료 조건을 명확히 해야 합니다.
5. **에러 핸들링:** `Tasklet`의 `execute` 메소드 내에서 발생하는 예외는 `Step` 실패로 이어집니다. 필요하다면 `try-catch`를 사용하여 예외를 적절히 처리하고, `Step`의 성공/실패 여부를 제어할 수 있습니다. (예: 특정 예외는 무시하고 `FINISHED` 반환)
6. **설정 주입:** `FileDeletingTasklet` 예제처럼 `Tasklet`에 필요한 설정값(예: 디렉토리 경로, DB 접속 정보 등)은 프로퍼티로 정의하고 Spring의 DI를 통해 주입받는 것이 좋습니다. 이렇게 하면 `Tasklet`의 재사용성과 테스트 용이성이 높아집니다.
7. **`InitializingBean` 활용:** `Tasklet`이 실행되기 전에 필요한 프로퍼티들이 제대로 설정되었는지 검증하고 싶을 때 `InitializingBean`의 `afterPropertiesSet()` 메소드를 활용하면 좋습니다.

---

### 이전 학습 내용과의 연관성

- **`Step`의 다양성:** 이전 챕터에서는 주로 청크 지향 `Step`에 대해 배웠지만, 이번 챕터를 통해 Spring Batch의 `Step`에는 `TaskletStep`이라는 또 다른 중요한 유형이 있다는 것을 알게 되었습니다. 이는 Spring Batch가 다양한 종류의 배치 작업을 처리할 수 있는 유연성을 가지고 있음을 보여줍니다.
- **`StepListener`의 자동 등록:** 이전 챕터에서 `ItemReader/Processor/Writer`가 `StepListener`를 구현하면 자동 등록될 수 있다는 것을 배웠습니다. 이번 챕터에서는 `Tasklet` 구현체가 `StepListener`를 구현하는 경우에도 동일하게 자동 등록된다는 것을 확인했습니다. 이는 Spring Batch의 일관된 설계 철학을 보여줍니다.
- **트랜잭션 관리:** 모든 `Step` 실행(청크 지향이든 `Tasklet` 기반이든)은 트랜잭션 컨텍스트 내에서 이루어진다는 점이 중요합니다. 다만, 그 트랜잭션의 범위와 단위가 `Step`의 종류에 따라 다를 수 있다는 것을 이해해야 합니다.

---
