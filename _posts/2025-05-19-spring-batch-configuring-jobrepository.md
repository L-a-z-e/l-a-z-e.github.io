---
title: Spring Batch - Configuring JobRepository
description: 
author: laze
date: 2025-05-19 00:00:01 +0900
categories: [Dev, SpringBatch]
tags: [SpringBatch]
---
**JobRepository 설정하기**

앞서 설명했듯이, `JobRepository`는 `JobExecution` 및 `StepExecution`과 같이 Spring Batch 내의 다양한 영속화된 도메인 객체에 대한 기본 CRUD(Create, Read, Update, Delete) 작업을 위해 사용됩니다.

이는 `JobLauncher`, `Job`, `Step`과 같은 프레임워크의 주요 기능 중 다수에 필요합니다.

`@EnableBatchProcessing`을 사용하면 `JobRepository`가 자동으로 제공됩니다.

이 섹션에서는 이를 사용자 정의하는 방법을 설명합니다.

Job 리포지토리의 구성 옵션은 다음 예와 같이 `@EnableBatchProcessing` 어노테이션의 속성을 통해 지정할 수 있습니다.

[Java 구성]

```java
@Configuration
@EnableBatchProcessing(
		dataSourceRef = "batchDataSource",
		transactionManagerRef = "batchTransactionManager",
		tablePrefix = "BATCH_", // 테이블 접두사
		maxVarCharLength = 1000, // VARCHAR 최대 길이
		isolationLevelForCreate = "SERIALIZABLE") // create* 메서드의 격리 수준
public class MyJobConfiguration {

   // Job 정의

}
```

여기에 나열된 구성 옵션은 필수가 아닙니다.

설정되지 않으면 앞서 보여준 기본값이 사용됩니다.

`max varchar length`의 기본값은 2500이며, 이는 샘플 스키마 스크립트의 긴 VARCHAR 열의 길이입니다.

**JobRepository의 트랜잭션 구성**

네임스페이스 또는 제공된 `FactoryBean`을 사용하는 경우, 리포지토리 주변에 트랜잭션 어드바이스(advice)가 자동으로 생성됩니다.

이는 실패 후 재시작에 필요한 상태를 포함한 배치 메타데이터가 올바르게 영속화되도록 하기 위한 것입니다.

리포지토리 메서드가 트랜잭션 방식이 아니면 프레임워크의 동작이 제대로 정의되지 않습니다.

`create*` 메서드 속성의 격리 수준은 Job이 시작될 때 두 프로세스가 동시에 동일한 Job을 시작하려고 시도하는 경우 하나만 성공하도록 보장하기 위해 별도로 지정됩니다.

해당 메서드의 기본 격리 수준은 `SERIALIZABLE`이며, 이는 매우 엄격합니다.

`READ_COMMITTED`도 일반적으로 잘 작동합니다.

두 프로세스가 이러한 방식으로 충돌할 가능성이 낮다면 `READ_UNCOMMITTED`도 괜찮습니다.

그러나 `create*` 메서드 호출은 매우 짧기 때문에 데이터베이스 플랫폼이 지원하는 한 `SERIALIZED`가 문제를 일으킬 가능성은 낮습니다.

그러나 이 설정을 재정의할 수 있습니다.

다음 예는 Java에서 격리 수준을 재정의하는 방법을 보여줍니다.

[Java 구성]

```java
@Configuration
@EnableBatchProcessing(isolationLevelForCreate = "ISOLATION_REPEATABLE_READ") // 원하는 격리 수준으로 변경 (예: REPEATABLE_READ)
public class MyJobConfiguration {

   // Job 정의

}
```

네임스페이스를 사용하지 않는 경우, AOP를 사용하여 리포지토리의 트랜잭션 동작도 구성해야 합니다.

다음 예는 Java에서 리포지토리의 트랜잭션 동작을 구성하는 방법을 보여줍니다.
*(이 예제는 `@EnableBatchProcessing`을 사용하지 않고 `JobRepository` 빈을 수동으로 설정하고 트랜잭션 프록시를 직접 구성하는 경우를 보여줍니다. `@EnableBatchProcessing`을 사용하면 프레임워크가 이 부분을 자동으로 처리해줍니다.)*

[Java 구성]

```java
@Bean
public JobRepository jobRepository(DataSource dataSource, PlatformTransactionManager transactionManager) throws Exception {
    JobRepositoryFactoryBean factory = new JobRepositoryFactoryBean();
    factory.setDataSource(dataSource);
    factory.setTransactionManager(transactionManager);
    // 필요시 다른 속성 설정 (예: factory.setTablePrefix("CUSTOM_BATCH_");)
    factory.afterPropertiesSet(); // 빈 초기화
    return factory.getObject();
}

@Bean // jobRepository 빈에 대한 트랜잭션 프록시 설정
public TransactionProxyFactoryBean jobRepositoryProxy(JobRepository jobRepository, PlatformTransactionManager transactionManager) {
	TransactionProxyFactoryBean transactionProxyFactoryBean = new TransactionProxyFactoryBean();
	Properties transactionAttributes = new Properties();
	transactionAttributes.setProperty("create*", "PROPAGATION_REQUIRED,ISOLATION_SERIALIZABLE"); // create* 메서드에 대한 설정
	transactionAttributes.setProperty("*", "PROPAGATION_REQUIRED"); // 나머지 메서드에 대한 설정
	transactionProxyFactoryBean.setTransactionAttributes(transactionAttributes);
	transactionProxyFactoryBean.setTarget(jobRepository); // jobRepository 빈을 타겟으로
	transactionProxyFactoryBean.setTransactionManager(transactionManager);
	return transactionProxyFactoryBean;
}
// JobLauncher 등에서 이 프록시된 jobRepository (jobRepositoryProxy)를 사용해야 함.
```

*(문서의 `TransactionProxyFactoryBean` 예제는 `jobRepository()`와 `transactionManager()` 메서드를 호출하고 있지만, 실제로는 빈으로 주입받거나 생성된 객체를 참조해야 합니다. 위 코드는 이를 반영하여 수정되었습니다.*

*하지만 `@EnableBatchProcessing`을 사용하면 이러한 복잡한 설정이 필요 없습니다.)*

**테이블 접두사 변경**

`JobRepository`의 또 다른 수정 가능한 속성은 메타데이터 테이블의 테이블 접두사입니다.

기본적으로 모두 `BATCH_`로 시작합니다.

`BATCH_JOB_EXECUTION` 및 `BATCH_STEP_EXECUTION`이 두 가지 예입니다.

그러나 이 접두사를 수정해야 하는 잠재적인 이유가 있습니다.

스키마 이름을 테이블 이름 앞에 추가해야 하거나 동일한 스키마 내에 둘 이상의 메타데이터 테이블 세트가 필요한 경우 테이블 접두사를 변경해야 합니다.

다음 예는 Java에서 테이블 접두사를 변경하는 방법을 보여줍니다.

[Java 구성]

```java
@Configuration
@EnableBatchProcessing(tablePrefix = "SYSTEM.TEST_")
public class MyJobConfiguration {

   // Job 정의

}
```

위 변경 사항을 적용하면 메타데이터 테이블에 대한 모든 쿼리에 `SYSTEM.TEST_`가 접두사로 붙습니다.

`BATCH_JOB_EXECUTION`은 `SYSTEM.TEST_JOB_EXECUTION`으로 참조됩니다.

테이블 접두사만 구성할 수 있습니다.

테이블 및 열 이름은 구성할 수 없습니다.

**리포지토리에서 비표준 데이터베이스 유형 사용**

지원되는 플랫폼 목록에 없는 데이터베이스 플랫폼을 사용하는 경우, SQL 변형이 충분히 유사하다면 지원되는 유형 중 하나를 사용할 수 있습니다.

이렇게 하려면 네임스페이스 바로 가기 대신 원시 `JobRepositoryFactoryBean`을 사용하고 이를 사용하여 데이터베이스 유형을 가장 유사한 항목으로 설정할 수 있습니다.

다음 예는 Java에서 `JobRepositoryFactoryBean`을 사용하여 데이터베이스 유형을 가장 유사한 항목으로 설정하는 방법을 보여줍니다.
*(이 예제는 `@EnableBatchProcessing`을 사용하지 않고 `JobRepository` 빈을 수동으로 구성하는 경우입니다.)*

[Java 구성]

```java
@Configuration
public class MyCustomDbJobRepositoryConfig {

    @Autowired
    private DataSource dataSource; // 애플리케이션 컨텍스트에서 DataSource 주입

    @Autowired
    private PlatformTransactionManager transactionManager; // 애플리케이션 컨텍스트에서 TransactionManager 주입

    @Bean
    public JobRepository jobRepository() throws Exception {
        JobRepositoryFactoryBean factory = new JobRepositoryFactoryBean();
        factory.setDataSource(dataSource);
        factory.setDatabaseType("db2"); // DB2와 유사한 DB를 사용한다고 가정
        factory.setTransactionManager(transactionManager);
        factory.afterPropertiesSet(); // 빈 초기화
        return factory.getObject();
    }

    // JobLauncher, Job 등 다른 빈 정의 시 이 jobRepository를 주입받아 사용
}
```

데이터베이스 유형이 지정되지 않으면 `JobRepositoryFactoryBean`은 `DataSource`에서 데이터베이스 유형을 자동 감지하려고 시도합니다.

플랫폼 간의 주요 차이점은 주로 기본 키 증가 전략에 있으므로, 종종 `incrementerFactory`도 재정의해야 합니다 (Spring Framework의 표준 구현 중 하나 사용).

그래도 작동하지 않거나 RDBMS를 사용하지 않는 경우, `SimpleJobRepository`가 의존하는 다양한 `Dao` 인터페이스를 구현하고 일반적인 Spring 방식으로 수동으로 연결하는 것이 유일한 옵션일 수 있습니다.

---

## Spring Batch의 기록 보관소, `JobRepository` 똑똑하게 설정하기! 🗄️🔧

우리가 앞에서 `JobRepository`가 `JobExecution`, `StepExecution` 같은 배치 작업의 모든 실행 정보(메타데이터)를 데이터베이스에 저장하고 관리하는 아주 중요한 역할을 한다고 배웠죠? 마치 학교의 학생 생활기록부처럼요!

`@EnableBatchProcessing` 어노테이션을 사용하면 이 `JobRepository`가 자동으로 뿅! 하고 만들어진다고 했어요. 하지만 때로는 이 `JobRepository`의 동작 방식을 조금 더 세밀하게 조정하고 싶을 때가 있어요. 예를 들면,

- 메타데이터를 저장할 **테이블 이름의 접두사**를 바꾸고 싶을 때
- 데이터베이스 **트랜잭션의 격리 수준**을 특별히 지정하고 싶을 때
- Spring Batch가 공식적으로 지원하지 않는 **특별한 데이터베이스**를 사용하고 싶을 때

이런 경우들을 어떻게 처리하는지 한번 알아봅시다!

---

### 1. `@EnableBatchProcessing`으로 `JobRepository` 맞춤 설정하기

가장 쉬운 방법은 `@EnableBatchProcessing` 어노테이션에 몇 가지 **속성(attributes)**을 추가하는 거예요. 마치 로봇 장난감에 여러 가지 부품을 갈아 끼우는 것과 비슷해요.

```java
@Configuration
@EnableBatchProcessing(
        dataSourceRef = "batchDataSource",               // 사용할 DataSource 빈 이름 (기본값: "dataSource")
        transactionManagerRef = "batchTransactionManager", // 사용할 TransactionManager 빈 이름 (기본값: "transactionManager")
        tablePrefix = "MYAPP_BATCH_",                    // 메타데이터 테이블 접두사 (기본값: "BATCH_")
        maxVarCharLength = 1000,                         // VARCHAR 컬럼 최대 길이 (기본값: 2500)
        isolationLevelForCreate = "ISOLATION_SERIALIZABLE" // JobExecution 생성 시 트랜잭션 격리 수준 (기본값: "ISOLATION_SERIALIZABLE")
)
public class MyJobConfiguration {
    // ... DataSource, TransactionManager, Job, Step 빈 정의 ...
}
```

**주요 설정 속성 살펴보기:**

- `dataSourceRef`, `transactionManagerRef`: 앞에서 배웠듯이, `JobRepository`가 사용할 `DataSource`와 `PlatformTransactionManager` 빈의 이름을 지정해요. 기본 이름(`dataSource`, `transactionManager`)이 아닌 다른 이름을 쓰고 싶을 때 사용해요.
- `tablePrefix`: Spring Batch 메타데이터 테이블 이름 앞에 붙는 접두사를 변경해요.
  - 기본값은 `BATCH_` 예요. 그래서 테이블 이름이 `BATCH_JOB_INSTANCE`, `BATCH_STEP_EXECUTION` 처럼 생겼죠.
  - 만약 `tablePrefix = "MYAPP_BATCH_"`로 바꾸면, 테이블 이름은 `MYAPP_BATCH_JOB_INSTANCE`가 돼요.
  - **언제 필요할까요?**
    - 하나의 데이터베이스 스키마 안에 여러 애플리케이션의 배치 메타데이터를 함께 저장해야 할 때 (이름 충돌 방지!)
    - 회사 규칙상 특정 접두사를 사용해야 할 때
- `maxVarCharLength`: 메타데이터 테이블의 `VARCHAR` 타입 컬럼들의 최대 길이를 지정해요. (예: `EXIT_MESSAGE` 컬럼) 기본값은 2500자인데, 혹시 더 긴 메시지를 저장해야 하거나 DB 제약사항이 있다면 조절할 수 있어요.
- `isolationLevelForCreate`: `JobRepository`가 새로운 `JobExecution`을 생성하는 `create*` 메서드들(예: `createJobExecution`)에 적용될 트랜잭션 격리 수준을 지정해요.
  - **왜 특별히 지정할까요?** 여러 애플리케이션이나 스케줄러가 동시에 같은 `JobInstance`를 실행하려고 할 때, 딱 하나만 성공적으로 `JobExecution`을 생성하도록 보장하기 위해서예요. (동시성 문제 방지!)
  - 기본값은 `ISOLATION_SERIALIZABLE`인데, 가장 높은 격리 수준으로 데이터 일관성을 매우 엄격하게 지켜요. 대부분의 경우 이 기본값이 안전하고 좋지만, 데이터베이스 종류나 상황에 따라 `ISOLATION_READ_COMMITTED` 등으로 변경할 수도 있어요. (예: `isolationLevelForCreate = "ISOLATION_REPEATABLE_READ"`)
  - **주의!** 여기서 지정한 격리 수준은 `create*` 메서드에만 적용돼요. 다른 `JobRepository` 메서드들은 보통 `PlatformTransactionManager`의 기본 설정을 따라요.

**핵심:** `@EnableBatchProcessing`의 속성들을 이용하면, 코드를 많이 바꾸지 않고도 `JobRepository`의 기본적인 동작 방식을 쉽게 커스터마이징할 수 있어요!

---

### 2. `JobRepository`의 트랜잭션, 왜 중요할까? (feat. 격리 수준)

Spring Batch 프레임워크는 `JobRepository`의 메서드들이 **트랜잭션 안에서 실행되는 것을 강력히 권장**해요. `@EnableBatchProcessing`을 사용하면 이런 트랜잭션 처리가 자동으로 설정돼요.

- **왜 트랜잭션이 필요할까요?**
  - 배치 작업이 실패했을 때, "여기까지는 성공적으로 처리했고, 여기서부터 다시 시작하면 돼"라는 **정확한 상태 정보**가 `JobRepository`에 저장되어야 해요.
  - 만약 상태 저장 중에 문제가 생겨서 일부만 저장되거나 꼬여버리면, 재시작이 불가능하거나 엉뚱하게 동작할 수 있어요.
  - 트랜잭션은 이런 상태 정보가 **원자적으로(All or Nothing)**, 그리고 **일관성 있게** 저장되도록 보장해줘요.
- **`create*` 메서드의 격리 수준 (`isolationLevelForCreate`) 다시 한번!**
  - 앞에서 잠깐 언급했듯이, `JobExecution`을 처음 만들 때(예: `JobLauncher`가 `Job`을 실행할 때) 이 격리 수준이 중요해요.
  - 만약 두 개의 서버에서 거의 동시에 "오늘의 일일정산 작업"을 실행하려고 한다고 상상해 보세요. `JobRepository`는 "어? 이 작업 이미 누가 시작하려고 하는데?" 하고 알아채서 둘 중 하나만 실제 `JobExecution`을 만들고, 다른 하나는 "이미 실행 중이야!" 하고 막아야겠죠?
  - `ISOLATION_SERIALIZABLE` 같은 높은 격리 수준은 이런 **동시 실행 시도 시 데이터 부정합을 막는 데 도움**을 줘요. `create*` 메서드 호출은 아주 짧은 순간이기 때문에, 성능에 큰 영향을 주지 않으면서 안정성을 높일 수 있어요.

**만약 `@EnableBatchProcessing`을 안 쓴다면? (수동 설정 - 참고만 하세요!)**

만약 `@EnableBatchProcessing`을 사용하지 않고 `JobRepository` 빈을 직접 만들어서 쓴다면, AOP(Aspect-Oriented Programming)를 이용해서 개발자가 직접 트랜잭션 설정을 해줘야 해요. (문서에 있는 `TransactionProxyFactoryBean` 예제가 그런 경우인데, 요즘엔 `@Transactional` 어노테이션이나 다른 방식으로 더 쉽게 할 수 있어요.) 하지만 대부분의 경우 `@EnableBatchProcessing`이 알아서 해주니 크게 걱정할 필요는 없어요!

**핵심:** `JobRepository`의 트랜잭션 설정은 배치 메타데이터의 **무결성과 정확한 재시작**을 위해 매우 중요해요. `@EnableBatchProcessing`이 대부분 알아서 잘 처리해준답니다!

---

### 3. 메타데이터 테이블 이름 바꾸기 (테이블 접두사 변경)

앞에서 `@EnableBatchProcessing(tablePrefix = "MYAPP_BATCH_")`처럼 테이블 접두사를 바꿀 수 있다고 했죠?

이렇게 하면 `JobRepository`가 데이터베이스에 쿼리를 날릴 때, 모든 테이블 이름 앞에 지정한 접두사를 붙여서 사용해요.

- 예시: `BATCH_JOB_EXECUTION` 테이블 조회 -> `MYAPP_BATCH_JOB_EXECUTION` 테이블 조회

**주의!**

- 오직 **테이블 접두사(prefix)만** 바꿀 수 있어요.
- 테이블 이름 자체(예: `JOB_EXECUTION`)나 컬럼 이름은 바꿀 수 없어요. 이건 Spring Batch 프레임워크 내부에 고정되어 있기 때문이에요.

**핵심:** 테이블 접두사 변경은 **스키마 공유 환경**이나 **명명 규칙 준수**를 위해 유용해요.

---

### 4. 특이한 데이터베이스 사용하기 (비표준 DB 지원)

Spring Batch는 Oracle, MySQL, PostgreSQL, SQLServer, H2, HSQLDB 등 많이 사용되는 데이터베이스들을 공식적으로 지원해요. 이 DB들은 각각 SQL 문법이나 기본 키 생성 방식 등이 조금씩 달라서, Spring Batch는 각 DB에 맞는 SQL 스크립트와 전략을 가지고 있어요.

**만약 공식 지원 목록에 없는 데이터베이스를 써야 한다면?**

1. **가장 비슷한 DB 타입으로 설정 시도:**
  - 사용하려는 DB가 공식 지원 DB 중 하나와 SQL 문법이 매우 유사하다면, 그 유사한 DB 타입을 지정해서 시도해볼 수 있어요.
  - 이때는 `@EnableBatchProcessing` 대신, `JobRepositoryFactoryBean`을 직접 빈으로 등록하고 `setDatabaseType("유사DB타입")` 메서드를 사용해야 해요.

    ```java
    @Configuration
    public class CustomDbConfig {
    
        @Autowired
        private DataSource dataSource;
    
        @Autowired
        private PlatformTransactionManager transactionManager;
    
        @Bean
        public JobRepository jobRepository() throws Exception {
            JobRepositoryFactoryBean factory = new JobRepositoryFactoryBean();
            factory.setDataSource(dataSource);
            factory.setTransactionManager(transactionManager);
            factory.setDatabaseType("db2"); // 예: 내 DB가 DB2와 거의 비슷하다고 가정
            // factory.setIncrementerFactory(...); // 필요하면 PK 생성 전략도 직접 지정
            factory.afterPropertiesSet(); // 초기화! 중요!
            return factory.getObject();
        }
        // ... JobLauncher, Job 등 다른 빈들도 이 jobRepository를 사용하도록 설정 ...
    }
    ```

  - `setDatabaseType()`을 지정하지 않으면, `JobRepositoryFactoryBean`이 `DataSource` 정보를 보고 "음... 이 DB는 MySQL인 것 같군!" 하고 자동으로 추측하려고 해요.
  - DB마다 기본 키(PK)를 자동으로 증가시키는 방식이 다른데, 이게 안 맞으면 `IncrementerFactory`도 직접 설정해줘야 할 수 있어요.
2. **`Dao` 인터페이스 직접 구현 (최후의 수단):**
  - 위 방법으로도 안 되거나, 아예 관계형 데이터베이스(RDBMS)가 아닌 NoSQL 같은 것을 `JobRepository` 저장소로 쓰고 싶다면 (매우 드문 경우), `SimpleJobRepository`가 내부적으로 사용하는 여러 `Dao` (Data Access Object) 인터페이스들(예: `JobInstanceDao`, `JobExecutionDao`, `StepExecutionDao`, `ExecutionContextDao`)을 직접 구현해야 해요. 이건 정말 많은 노력이 필요하고 Spring Batch 내부 구조에 대한 깊은 이해가 필요해요.

**핵심:** 대부분의 경우 공식 지원 DB를 사용하겠지만, 혹시 특수한 상황이라면 `JobRepositoryFactoryBean`을 통해 **유사 DB 타입을 지정**해보거나, 최악의 경우 **`Dao`를 직접 구현**하는 방법도 있다는 것을 알아두세요. (하지만 `@EnableBatchProcessing`이 최고!)

---
