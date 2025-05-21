---
title: Spring Batch - Configuring a Step (Commit Interval)
description: 
author: laze
date: 2025-05-21 00:00:04 +0900
categories: [Dev, SpringBatch]
tags: [SpringBatch]
---
**커밋 간격 (The Commit Interval)**

앞서 언급했듯이, `Step`은 제공된 `PlatformTransactionManager`를 사용하여 주기적으로 커밋하면서 항목을 읽고 씁니다.

커밋 간격이 1이면 각 개별 항목을 쓴 후 커밋합니다.

이는 트랜잭션을 시작하고 커밋하는 데 비용이 많이 들기 때문에 많은 상황에서 이상적이지 않습니다.

이상적으로는 각 트랜잭션에서 가능한 한 많은 항목을 처리하는 것이 바람직하며, 이는 전적으로 처리되는 데이터 유형과 `Step`이 상호 작용하는 리소스에 따라 달라집니다.

이러한 이유로 커밋 내에서 처리되는 항목 수를 구성할 수 있습니다.

다음 예는 Java로 정의될 때 `tasklet`의 `commit-interval` 값이 10인 `Step`을 보여줍니다.
*(문서에서는 `<tasklet>` 요소 내부의 `<chunk>`를 언급하고 있지만, Java 설정에서는 `StepBuilder`의 `chunk()` 메서드에 이 값을 직접 지정합니다. 또한, `ItemReader`와 `ItemWriter` 빈 정의는 생략되어 있습니다.)*

```java
@Bean
public Job sampleJob(JobRepository jobRepository, Step step1) {
    return new JobBuilder("sampleJob", jobRepository)
                     .start(step1)
                     .build();
}

@Bean
public Step step1(JobRepository jobRepository, PlatformTransactionManager transactionManager,
                  ItemReader<String> itemReader, ItemWriter<String> itemWriter) { // ItemReader와 ItemWriter를 주입받도록 수정
	return new StepBuilder("step1", jobRepository)
				.<String, String>chunk(10, transactionManager) // 커밋 간격(청크 크기)을 10으로 설정
				.reader(itemReader) // itemReader() 메서드 호출 대신 주입받은 빈 사용
				.writer(itemWriter) // itemWriter() 메서드 호출 대신 주입받은 빈 사용
				.build();
}

// ItemReader 및 ItemWriter 빈 정의 (예시)
// @Bean
// public ItemReader<String> itemReader() { ... }
// @Bean
// public ItemWriter<String> itemWriter() { ... }

```

위 예에서 각 트랜잭션 내에서 10개의 항목이 처리됩니다. 처리 시작 시 트랜잭션이 시작됩니다. 또한 `ItemReader`에서 `read`가 호출될 때마다 카운터가 증가합니다. 카운터가 10에 도달하면 집계된 항목 목록이 `ItemWriter`에 전달되고 트랜잭션이 커밋됩니다.

---

## 청크 Step의 엔진오일, 커밋 간격(Commit Interval) 제대로 알기! ⚙️⛽

우리가 청크 지향 `Step`이 데이터를 "덩어리(chunk)"로 묶어서 처리한다고 배웠죠? 이때 **"한 덩어리에 몇 개의 아이템을 담을 것이냐?"**를 결정하는 것이 바로 **커밋 간격(Commit Interval)** 또는 **청크 크기(Chunk Size)**입니다.

Java 설정에서는 `StepBuilder`의 `chunk()` 메서드를 통해 이 값을 지정해요.

```java
@Bean
public Step step1(JobRepository jobRepository,
                  PlatformTransactionManager transactionManager, // 트랜잭션 관리를 위해 필요!
                  ItemReader<String> reader,
                  ItemWriter<String> writer) {
    return new StepBuilder("step1", jobRepository)
            .<String, String>chunk(10, transactionManager) // <-- 바로 이 부분! 청크 크기를 10으로 설정!
            .reader(reader)
            .writer(writer)
            .build();
}
```

위 예시에서 `.chunk(10, transactionManager)`은 "아이템 10개를 하나의 청크로 묶어서 처리하고, 이 과정은 `transactionManager`를 통해 하나의 트랜잭션으로 관리해줘!" 라는 의미예요.

---

### 커밋 간격이 하는 일 (동작 원리)

커밋 간격(여기서는 10)이 설정되면 `Step`은 다음과 같이 동작해요:

1. **트랜잭션 시작:** `Step`이 청크 처리를 시작하기 전에, Spring Batch는 `PlatformTransactionManager`를 사용해서 새로운 트랜잭션을 시작해요.
2. **아이템 읽기 (Read):** `ItemReader`가 `read()` 메서드를 호출하여 데이터를 **하나씩** 읽어와요.
3. **카운터 증가:** 아이템을 하나 읽을 때마다 내부적으로 카운터가 1씩 증가해요. (만약 `ItemProcessor`가 있다면, 읽은 아이템은 `ItemProcessor`를 거쳐 가공돼요.)
4. **청크 완성 확인:** 카운터가 설정된 커밋 간격(여기서는 10)에 도달했는지 확인해요.
  - **아직 10개가 안 됐다면?** -> 2번으로 돌아가서 다음 아이템을 계속 읽어요.
  - **드디어 10개가 됐다면? (또는 더 이상 읽을 아이템이 없다면?)** -> 다음 단계로!
5. **아이템 쓰기 (Write):** 지금까지 모인 10개의 아이템 (또는 남아있는 모든 아이템) 덩어리(청크)를 `ItemWriter`에게 전달해서 한꺼번에 쓰기 작업을 수행해요.
6. **트랜잭션 커밋:** `ItemWriter`의 쓰기 작업이 성공적으로 끝나면, 1번에서 시작했던 트랜잭션을 **커밋(commit)**해요. "이번 덩어리 처리 성공! 데이터베이스에 확정!" 하는 거죠.
7. **반복 또는 종료:** 아직 처리할 데이터가 남아있다면, 1번으로 돌아가서 새로운 트랜잭션을 시작하고 다음 청크 처리를 계속해요. 더 이상 읽을 데이터가 없으면 `Step`이 종료돼요.

**만약 커밋 간격이 1이라면?**

`chunk(1, transactionManager)`로 설정하면, 아이템 하나를 읽고, (처리하고), 쓰고, 바로 커밋하는 과정이 매 아이템마다 반복돼요.

---

### 커밋 간격, 왜 중요할까요?

- **성능 (Performance):**
  - **너무 작은 커밋 간격 (예: 1):** 아이템 하나 처리할 때마다 트랜잭션을 시작하고 커밋하는 것은 데이터베이스에 상당한 부하를 줘요. 네트워크 통신 비용, 디스크 I/O 비용 등이 매번 발생하므로 전체적인 처리 속도가 매우 느려질 수 있어요.
  - **적절한 커밋 간격:** 여러 아이템을 묶어서 한 번의 트랜잭션으로 처리하면, 이러한 오버헤드를 크게 줄여 성능을 향상시킬 수 있어요. (예: 1000건의 데이터를 1건씩 1000번 커밋하는 것보다, 1000건을 한 번에 커밋하는 것이 훨씬 빠르겠죠?)
- **메모리 사용량 (Memory Usage):**
  - 커밋 간격만큼의 아이템들이 메모리에 잠시 보관되었다가 `ItemWriter`로 전달돼요.
  - **너무 큰 커밋 간격:** 한 번에 처리해야 할 아이템 수가 너무 많아지면, 메모리를 많이 사용하게 되어 `OutOfMemoryError`가 발생할 위험이 커져요.
- **오류 발생 시 복구 (Recovery):**
  - **적절한 커밋 간격:** 만약 특정 청크 처리 중 오류가 발생하면, 해당 청크 내의 작업만 롤백(rollback)돼요. 이전에 성공적으로 커밋된 청크들은 안전하게 유지되죠. 재시작 시에도 실패한 청크부터 (또는 해당 청크의 처음부터) 다시 시작하면 되므로 복구 범위가 작아져요.
  - **너무 큰 커밋 간격:** 하나의 청크가 매우 크다면, 오류 발생 시 롤백해야 할 데이터 양이 많아지고, 재시작 시 다시 처리해야 할 작업량도 커져서 복구 시간이 길어질 수 있어요.
- **데이터베이스 락 (Database Locking):**
  - 하나의 트랜잭션이 오래 지속될수록 데이터베이스의 특정 데이터에 대한 락(lock) 점유 시간도 길어질 수 있어요. 이는 다른 트랜잭션과의 충돌 가능성을 높이고, 시스템 전체의 동시성을 저해할 수 있어요.
  - **너무 큰 커밋 간격**은 트랜잭션 시간을 길게 만들어 이런 문제를 유발할 수 있어요.

---

### 최적의 커밋 간격은 어떻게 찾을까요?

아쉽게도 "이 숫자가 최고야!" 하는 만능 정답은 없어요. 다음과 같은 요소들을 고려해서 **실제 환경에서 테스트를 통해 조율**해나가는 것이 가장 좋아요.

- **처리하는 데이터의 크기와 복잡성:** 아이템 하나하나가 얼마나 많은 정보를 담고 있는지, `ItemProcessor`의 로직이 얼마나 복잡한지 등.
- **메모리 자원:** 애플리케이션 서버가 사용할 수 있는 메모리 크기.
- **데이터베이스 성능 및 종류:** 사용 중인 데이터베이스의 처리 능력, 락킹 메커니즘 등.
- **트랜잭션의 중요도 및 복구 시간 요구사항:** 얼마나 빠른 복구가 필요한지, 한 번에 롤백될 수 있는 데이터의 허용 범위는 어느 정도인지.
- **네트워크 환경:** 데이터베이스 서버와의 네트워크 지연 시간 등.

**일반적인 접근 방법:**

1. **작은 값(예: 10, 100)으로 시작해서 점차 늘려가며 테스트**해보세요.
2. 처리 시간, 메모리 사용량, CPU 사용량 등을 모니터링하면서 성능 변화를 관찰하세요.
3. 특정 값을 넘어서면 오히려 성능이 저하되거나 메모리 문제가 발생하는 지점을 찾을 수 있을 거예요.
4. 너무 큰 값보다는 **안정적으로 좋은 성능을 내는 적절한 크기**를 선택하는 것이 중요해요. 보통 수십에서 수천 사이의 값을 많이 사용하지만, 상황에 따라 달라질 수 있습니다.

**핵심:** 커밋 간격은 청크 지향 `Step`의 **성능과 안정성을 결정짓는 중요한 튜닝 포인트**예요. "읽고-쓰고-커밋"하는 주기를 조절함으로써, 시스템 자원을 효율적으로 사용하고 안정적인 대량 데이터 처리를 가능하게 한답니다!

---

## 데이터 지킴이, `PlatformTransactionManager` 파헤치기! 🛡️🚦

**트랜잭션(Transaction)이란 무엇일까요?**

먼저 트랜잭션의 개념부터 간단히 짚고 넘어갈게요. 트랜잭션은 **"더 이상 쪼갤 수 없는 최소한의 작업 단위"**를 의미해요. 이 작업 단위는 다음 네 가지 중요한 특성(ACID)을 만족해야 합니다.

- **원자성 (Atomicity):** 트랜잭션 내의 모든 작업이 전부 성공하거나, 하나라도 실패하면 전부 실패(롤백)해야 합니다. "All or Nothing!"
- **일관성 (Consistency):** 트랜잭션이 성공적으로 완료되면 데이터베이스는 항상 일관된 상태를 유지해야 합니다. (예: 계좌 이체 시 총액은 변하지 않음)
- **고립성 (Isolation):** 여러 트랜잭션이 동시에 실행될 때, 각 트랜잭션은 다른 트랜잭션의 작업에 영향을 받지 않고 독립적으로 실행되는 것처럼 보여야 합니다. (다른 사람의 작업에 끼어들지 마!)
- **지속성 (Durability):** 성공적으로 완료된 트랜잭션의 결과는 시스템 장애가 발생하더라도 영구적으로 저장되어야 합니다.

**`PlatformTransactionManager`는 바로 이 트랜잭션을 관리하는 역할을 합니다.**

---

### `PlatformTransactionManager`의 역할과 중요성

Spring 프레임워크에서 `PlatformTransactionManager`는 다양한 트랜잭션 API(예: JDBC, JPA, JTA 등)에 대한 **추상화 계층**을 제공하는 인터페이스입니다. 이게 무슨 말이냐면, 우리가 어떤 기술(JDBC, JPA 등)을 사용하든 상관없이 **일관된 방식으로 트랜잭션을 제어**할 수 있게 해준다는 뜻이에요.

**Spring Batch에서 `PlatformTransactionManager`가 하는 일:**

1. **트랜잭션 시작 (Begin Transaction):**
  - 청크 지향 `Step`에서 청크 처리가 시작될 때, `PlatformTransactionManager`는 새로운 트랜잭션을 시작합니다.
  - "자, 지금부터 하는 작업들은 하나의 묶음이야!" 라고 선언하는 거죠.
2. **트랜잭션 커밋 (Commit Transaction):**
  - 청크 내의 모든 아이템에 대한 `ItemWriter`의 쓰기 작업이 성공적으로 완료되면, `PlatformTransactionManager`는 해당 트랜잭션을 커밋합니다.
  - "이번 묶음 작업 성공! 데이터베이스에 확정 저장!"
3. **트랜잭션 롤백 (Rollback Transaction):**
  - 청크 처리 중 (`ItemReader`, `ItemProcessor`, `ItemWriter` 어느 단계에서든) 예외가 발생하여 작업이 실패하면, `PlatformTransactionManager`는 해당 트랜잭션을 롤백합니다.
  - "이런! 이번 묶음 작업 실패! 모든 변경 사항을 원래대로 되돌려!"
  - 이렇게 롤백되면, 해당 청크에서 데이터베이스에 변경하려던 내용들이 모두 취소되어 데이터 일관성이 유지됩니다.
4. **트랜잭션 상태 관리:**
  - 현재 트랜잭션이 진행 중인지, 새로 시작해야 하는지, 기존 트랜잭션에 참여해야 하는지 등의 상태를 관리하고 결정합니다. (트랜잭션 전파 속성 - Propagation Behavior)
  - 트랜잭션 격리 수준(Isolation Level) 같은 고급 설정도 관리합니다.

**왜 `PlatformTransactionManager`가 필요할까요?**

- **데이터 무결성 보장:** 배치 작업은 대량의 데이터를 다루기 때문에, 중간에 오류가 발생했을 때 데이터가 뒤죽박죽되거나 일부만 반영되는 것을 막아야 합니다. `PlatformTransactionManager`는 트랜잭션을 통해 이를 보장합니다.
- **재시작 용이성:** 청크 단위로 트랜잭션이 커밋되므로, 실패 시 문제가 발생한 청크만 롤백되고 이전 청크의 결과는 유지됩니다. 따라서 `JobRepository`에 기록된 정보를 바탕으로 실패한 지점부터 안전하게 재시작할 수 있습니다.
- **유연한 트랜잭션 전략:** 다양한 데이터 접근 기술(JDBC, JPA, Hibernate 등)에 맞는 `PlatformTransactionManager` 구현체를 선택하여 사용할 수 있습니다. 개발자는 특정 기술에 종속되지 않고 일관된 방식으로 트랜잭션을 관리할 수 있습니다.

---

### `PlatformTransactionManager`의 종류 (구현체)

`PlatformTransactionManager`는 인터페이스이기 때문에, 실제로는 이 인터페이스를 구현한 클래스들이 사용됩니다. 어떤 데이터 접근 기술을 사용하느냐에 따라 주로 사용되는 구현체가 다릅니다.

- **`DataSourceTransactionManager`:**
  - **JDBC**를 직접 사용하거나, MyBatis 같은 SQL 매퍼 프레임워크를 사용할 때 주로 사용됩니다.
  - 하나의 `DataSource`에 대한 트랜잭션을 관리합니다.
  - Spring Batch 예제에서 `JdbcTransactionManager`라는 이름으로 빈을 등록하는 것을 보셨을 텐데, 이것이 `DataSourceTransactionManager`의 하위 클래스이거나 동일한 역할을 하는 클래스입니다. (정확히는 `DataSourceTransactionManager`를 사용합니다.)
- **`JpaTransactionManager`:**
  - *JPA(Java Persistence API)**를 사용할 때 (예: Hibernate, EclipseLink 등을 JPA 구현체로 사용할 때) 사용됩니다.
  - JPA의 `EntityManagerFactory`를 통해 트랜잭션을 관리합니다.
- **`JtaTransactionManager`:**
  - *분산 트랜잭션(Distributed Transaction)**을 처리해야 할 때 사용됩니다. 예를 들어, 여러 개의 데이터베이스나 메시지 큐 같은 여러 리소스에 걸쳐 하나의 트랜잭션을 유지해야 할 때 JTA(Java Transaction API)를 사용하며, 이때 이 트랜잭션 매니저를 사용합니다. (매우 복잡한 시나리오)
- **기타:** Hibernate를 직접 사용할 때는 `HibernateTransactionManager` 등 특정 기술에 특화된 구현체들도 있습니다.

**Spring Batch 설정 시:**

우리가 `@EnableBatchProcessing`을 사용하면, Spring 컨테이너 내에 `PlatformTransactionManager` 타입의 빈이 (보통 `transactionManager`라는 이름으로) 등록되어 있어야 한다고 했죠? 이때, 우리가 사용하는 데이터 접근 기술에 맞는 적절한 `PlatformTransactionManager` 구현체를 빈으로 등록해주면 됩니다.

```java
@Configuration
@EnableBatchProcessing // Spring Batch 기본 설정 활성화
public class BatchConfig {

    // 1. DataSource 빈 정의 (예: H2 인메모리 DB)
    @Bean
    public DataSource dataSource() {
        return new EmbeddedDatabaseBuilder()
                .setType(EmbeddedDatabaseType.H2)
                .addScript("classpath:org/springframework/batch/core/schema-h2.sql") // Spring Batch 메타데이터 테이블 생성
                .build();
    }

    // 2. PlatformTransactionManager 빈 정의 (JDBC 사용 가정)
    @Bean
    public PlatformTransactionManager transactionManager(DataSource dataSource) { // 위에서 정의한 dataSource를 주입받음
        return new DataSourceTransactionManager(dataSource); // DataSourceTransactionManager 사용
    }

    // ... Job, Step, ItemReader, ItemWriter 등 정의 ...
}

```

**핵심:** `PlatformTransactionManager`는 Spring Batch에서 **청크 단위 처리의 데이터 일관성과 안정성을 보장하는 핵심적인 역할**을 합니다. 우리가 어떤 데이터베이스 기술을 사용하든, Spring은 이 인터페이스를 통해 일관된 트랜잭션 관리 기능을 제공해줍니다.
