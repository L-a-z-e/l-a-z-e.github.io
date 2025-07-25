---
title: ExecutorType 충돌, TransactionTemplate으로 Reader/Writer 트랜잭션 분리하기
description: 
author: laze
date: 2025-07-25 00:00:03 +0900
categories: [Dev, SpringBatch]
tags: [SpringBatch, ExecutorType, Transaction]
---
# [Spring Batch/MyBatis 심층 분석] `ExecutorType` 충돌 해결: `TransactionTemplate`으로 Reader/Writer 트랜잭션 분리하기

## 1. 서론: `CursorItemReader`와 `BatchItemWriter`을 하나의 Step에서 사용하기, 왜 문제인가?

Spring Batch로 대용량 데이터를 처리할 때, 우리는 종종 다음과 같은 이상적인 시나리오를 꿈꾼다.

- **조회(Read):** 대용량 데이터를 메모리 부담 없이 안전하게 읽어오기 위해 `MyBatisCursorItemReader`를 사용한다.
- **쓰기(Write):** 처리된 데이터를 데이터베이스에 최대한 빠르게 입력하기 위해 `MyBatisBatchItemWriter`를 사용한다.

하지만 이 두 컴포넌트를 하나의 Chunk-Oriented Step에 함께 배치하는 순간, 우리는 다음과 같은 낯선 예외와 마주치게 된다.

```
org.springframework.dao.TransientDataAccessResourceException:
Error changing ExecutorType. Cannot change the ExecutorType when there is an existing transaction.
```

"기존 트랜잭션이 있어서 실행기 타입을 바꿀 수 없다"는 이 메시지는 대체 무엇을 의미할까?

이 글에서는 이 문제의 근본 원인을 MyBatis의 `ExecutorType`과 Spring Batch의 트랜잭션 모델을 통해 심층적으로 분석하고, `TransactionTemplate`을 이용한 정교한 해결책을 제시한다.

## 2. `ExecutorType`의 두 얼굴: `SIMPLE` vs `BATCH`

이 문제의 핵심에는 MyBatis가 SQL을 실행하는 방식인 `ExecutorType`이 있다.

- **`ExecutorType.SIMPLE`:**
  - **동작 방식:** 각 SQL 문을 실행할 때마다 새로운 `PreparedStatement`를 생성하고 실행한다. 가장 기본적인 실행기 타입이다.
  - **주요 사용처:** `MyBatisCursorItemReader`가 이 방식을 사용한다. 대용량 데이터를 조회할 때 DB 커서를 사용하여 한 번에 하나의 레코드만 처리하는 것이 효율적이기 때문이다.
- **`ExecutorType.BATCH`:**
  - **동작 방식:** 여러 개의 `INSERT`, `UPDATE`, `DELETE` SQL 문을 JDBC 드라이버 레벨에서 하나의 묶음(Batch)으로 만들어 DB에 한 번에 전송한다. 네트워크 왕복 횟수를 획기적으로 줄여 대량 쓰기 작업의 성능을 극대화한다.
  - **주요 사용처:** `MyBatisBatchItemWriter`는 고성능 쓰기를 위해 반드시 이 방식을 사용해야 한다.

**근본적인 제약사항:** MyBatis의 `SqlSession`은 트랜잭션과 생명주기를 같이한다. 그리고 **하나의 `SqlSession`(하나의 트랜잭션)이 시작될 때, `ExecutorType`은 둘 중 하나로만 결정된다.** `ItemReader`가 먼저 실행되면서 현재 트랜잭션의 `SqlSession`은 `SIMPLE` 모드로 동작을 시작하는데, 그 직후 `ItemWriter`가 동일한 `SqlSession`에게 `BATCH` 모드로 동작하라고 요구하니 충돌이 발생하는 것이다.

## 3. 해결의 열쇠: 트랜잭션 전파(Propagation)와 분리

이 문제를 해결할 유일한 방법은 `Reader`와 `Writer`가 사용하는 트랜잭션을 분리하여, 각자에게 맞는 `ExecutorType`으로 동작할 수 있는 독립적인 환경을 만들어주는 것이다.

이때 필요한 것이 Spring의 **트랜잭션 전파(Transaction Propagation)** 개념이다.

- **`Propagation.REQUIRED` (기본값):** "이미 진행 중인 트랜잭션이 있으면 거기에 참여하고, 없으면 새로 시작해." 이 기본값으로는 `Writer`가 `Reader`가 시작한 기존 트랜잭션에 참여하므로 문제가 해결되지 않는다.
- **`Propagation.REQUIRES_NEW` (해결책):** "이미 진행 중인 트랜잭션이 있든 없든 상관없이, **무조건 새로운 트랜잭션을 시작해.** 만약 기존 트랜잭션이 있다면, 그건 잠시 일시정지(suspend)시켜둬."

이 `REQUIRES_NEW` 옵션을 사용하면 `Writer`는 `Reader`와 완전히 독립된 새로운 트랜잭션(과 새로운 `SqlSession`)을 할당받아 `ExecutorType.BATCH`로 안전하게 동작할 수 있다.

## 4. 실전 구현: `TransactionTemplate`으로 `ItemWriter` 트랜잭션 분리하기

`@Transactional` 어노테이션은 메서드 전체에 적용되므로, `write` 메서드의 일부만 다른 트랜잭션으로 묶기 어렵다. 이처럼 프로그래밍적으로 세밀한 트랜잭션 제어가 필요할 때 `TransactionTemplate`을 사용한다.

### Step 1: `SqlSessionTemplate` 분리하기

먼저, `SIMPLE`과 `BATCH` 모드로 동작할 `SqlSessionTemplate` Bean을 각각 생성한다.

**`MybatisConfig.java`**

```java
@Configuration
public class MybatisConfig {

    // ... SqlSessionFactory Bean 설정 ...

    @Bean(name = "simpleSqlSessionTemplate")
    public SqlSessionTemplate simpleSqlSessionTemplate(SqlSessionFactory sqlSessionFactory) {
        return new SqlSessionTemplate(sqlSessionFactory, ExecutorType.SIMPLE);
    }

    @Bean(name = "batchSqlSessionTemplate")
    public SqlSessionTemplate batchSqlSessionTemplate(SqlSessionFactory sqlSessionFactory) {
        return new SqlSessionTemplate(sqlSessionFactory, ExecutorType.BATCH);
    }
}
```

### Step 2: `ItemReader`와 `ItemWriter` 설정

이제 각 컴포넌트에 맞는 `SqlSessionTemplate`을 주입하고, `ItemWriter`를 `TransactionTemplate`으로 감싸 트랜잭션을 분리한다.

**`MyJobConfig.java`**

```java
@Configuration
@RequiredArgsConstructor
public class MyJobConfig {

    private final JobRepository jobRepository;
    private final PlatformTransactionManager transactionManager;

    @Qualifier("simpleSqlSessionTemplate")
    private final SqlSessionTemplate simpleSqlSessionTemplate;

    @Qualifier("batchSqlSessionTemplate")
    private final SqlSessionTemplate batchSqlSessionTemplate;

    @Bean
    public Job myJob() {
        return new JobBuilder("myJob", jobRepository)
                .start(myStep())
                .build();
    }

    @Bean
    public Step myStep() {
        return new StepBuilder("myStep", jobRepository)
                .<MyDto, MyDto>chunk(100, transactionManager)
                .reader(myCursorItemReader())
                .writer(transactionalBatchItemWriter()) // 트랜잭션이 분리된 Writer 사용
                .build();
    }

    @Bean
    @StepScope
    public MyBatisCursorItemReader<MyDto> myCursorItemReader() {
        return new MyBatisCursorItemReaderBuilder<MyDto>()
                .sqlSessionTemplate(simpleSqlSessionTemplate)
                .queryId("myMapper.selectData")
                .build();
    }

    @Bean
    @StepScope
    public ItemWriter<MyDto> transactionalBatchItemWriter() {
        // 실제 쓰기 작업을 수행할 Batch Writer 생성
        MyBatisBatchItemWriter<MyDto> delegateWriter = new MyBatisBatchItemWriterBuilder<MyDto>()
                .sqlSessionTemplate(batchSqlSessionTemplate)
                .statementId("myMapper.insertData")
                .build();

        // TransactionTemplate으로 감싸서 반환
        return items -> {
            TransactionTemplate txTemplate = new TransactionTemplate(transactionManager);
            txTemplate.setPropagationBehavior(TransactionDefinition.PROPAGATION_REQUIRES_NEW);

            txTemplate.execute(status -> {
                try {
                    delegateWriter.write(items);
                } catch (Exception e) {
                    status.setRollbackOnly();
                    throw new RuntimeException("Writer에서 오류 발생", e);
                }
                return null;
            });
        };
    }
}
```

위 `transactionalBatchItemWriter` 코드는 람다를 사용하여 `ItemWriter`의 익명 구현체를 만들었다.

`write` 메서드가 호출될 때마다, `TransactionTemplate`이 `REQUIRES_NEW` 전파 옵션으로 새로운 트랜잭션을 시작한다.

그 안에서 `MyBatisBatchItemWriter`가 `batchSqlSessionTemplate`을 사용하여 안전하게 배치 쓰기를 수행한다.

## 5. 대안과 고려사항: 다른 방법은 없는가?

### 대안: Step 분리 (가장 단순하고 확실한 방법)

하나의 Step에서 이 모든 것을 해결하려는 대신, Job의 흐름을 두 개의 Step으로 나누는 것도 훌륭한 방법이다.

1. **Step 1 (조회 및 중간 저장):** `MyBatisCursorItemReader`로 데이터를 읽어, 임시 테이블이나 파일에 저장한다.
2. **Step 2 (쓰기):** Step 1이 끝나면 실행된다. 임시 저장된 데이터를 읽어 `MyBatisBatchItemWriter`로 최종 목적지에 쓴다.
- **장점:** 각 Step이 하나의 책임만 가지므로 코드가 매우 단순하고 명확해진다.
- **단점:** Job의 흐름이 길어지고, 중간 데이터 저장을 위한 추가적인 리소스(임시 테이블 등)가 필요하다.

## 6. 결론

`MyBatisCursorItemReader`와 `MyBatisBatchItemWriter`의 `ExecutorType` 충돌은, **하나의 트랜잭션(하나의 `SqlSession`)** 내에서 상반된 실행 모드를 동시에 요구하기 때문에 발생하는 필연적인 문제다.

`TransactionTemplate`과 `Propagation.REQUIRES_NEW`를 사용한 프로그래밍 방식의 트랜잭션 분리는, Step을 나누지 않고 이 문제를 해결하는 가장 정교하고 효과적인 방법이다.

Spring Batch와 같은 프레임워크를 깊이 있게 사용하려면, 그 기반이 되는 트랜잭션과 같은 핵심 기술에 대한 이해가 필수적이다. 이 문제를 해결하는 과정은 복잡해 보일 수 있지만, Spring의 트랜잭션 관리 능력을 한 단계 더 깊이 이해하는 좋은 기회가 될 것이다.
