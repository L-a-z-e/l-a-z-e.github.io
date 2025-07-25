---
title: Spring Batch ORA-08177 완벽 해부 (원인과 해결책)
description: 
author: laze
date: 2025-07-25 00:00:03 +0900
categories: [Dev, SpringBatch]
tags: [SpringBatch, ExecutorType, Transaction]
---
# [심층 분석] Spring Batch의 유령 오류, ORA-08177 완벽 해부 (원인과 해결책)

## 1. 서론: 코드 수정 없이 나타났다 사라지는 ORA-08177

Spring Batch로 대용량 데이터를 처리하는 Job을 운영하다 보면, 개발자를 가장 괴롭히는 종류의 오류를 만날 때가 있습니다. 바로 **간헐적으로 발생했다가 사라지는 오류**입니다. 그중에서도 `ORA-08177: can't serialize access for this transaction`은 단연 악명 높은 문제 중 하나입니다.

```
Caused by: java.sql.SQLException: ORA-08177: can't serialize access for this transaction
```

코드는 전혀 수정한 것이 없는데, 어제는 성공했던 배치가 오늘은 실패하고, 재실행하니 또 성공합니다. 이처럼 예측 불가능한 '유령' 같은 오류의 원인은 대체 무엇일까요?

이 글에서는 이 문제가 단순한 코드 버그가 아니라, 데이터베이스의 **트랜잭션 격리 수준(Transaction Isolation Level)**과 **동시성(Concurrency)**이 복잡하게 얽혀 발생하는 현상임을 심층적으로 분석하고, 그 원인을 추적하는 방법과 실전적인 해결책을 제시합니다.

## 2. 오류의 정체: "당신이 보던 세상은 이미 변했다"

`ORA-08177` 오류의 의미를 한 문장으로 요약하면 다음과 같습니다.

> "당신의 트랜잭션이 시작된 이후, 당신이 읽었던 데이터가 다른 트랜잭션에 의해 변경 후 커밋(Commit)되었습니다. 지금 당신이 이 데이터를 수정하려고 하면 데이터 정합성이 깨지므로, 이 작업을 계속할 수 없습니다."
>

이 오류는 거의 항상 Oracle DB의 가장 엄격한 트랜잭션 격리 수준인 **`SERIALIZABLE`**에서만 발생합니다.

- **`SERIALIZABLE`의 약속:** 여러 트랜잭션이 동시에 실행되더라도, 마치 **모든 트랜잭션이 한 줄로 순서대로 실행된 것과 동일한 결과**를 보장하겠다는 매우 강력한 약속입니다.
- **오류 발생 시나리오:**
  1. **[내 배치 트랜잭션 A 시작]** `SELECT` 쿼리로 ID 100번 고객의 데이터를 읽습니다. DB는 이 시점의 데이터 상태를 '스냅샷'으로 기억합니다.
  2. **[다른 트랜잭션 B의 개입]** 내 배치가 아직 끝나지 않은 상태에서, 다른 API 호출이나 다른 배치 Job인 **트랜잭션 B**가 ID 100번 고객의 주소를 수정하고 **커밋(Commit)**합니다.
  3. **[내 배치 트랜잭션 A의 쓰기 시도]** 이제 내 배치가 자신이 읽었던 ID 100번 고객의 데이터를 `UPDATE`하려고 합니다.
  4. **[Oracle DB의 거부!]** DB는 이 시점에서 충돌을 감지합니다. "트랜잭션 A, 네가 ID 100번을 처음 읽었을 때와 지금 상태가 달라졌다. 만약 네 수정을 허용하면, '순서대로 실행된 것 같은 결과'라는 `SERIALIZABLE` 약속이 깨져버린다. 미안하지만 이 작업은 실패시켜야겠다."
  5. **[ORA-08177 발생]** DB는 트랜잭션 A에게 오류를 던지며 작업을 강제로 롤백시킵니다.

### 왜 '간헐적으로' 발생할까?

이 오류는 위 시나리오의 2번 단계, 즉 '다른 트랜잭션 B의 개입'이 트랜잭션 A가 데이터를 읽은 후 쓰기 전까지의 그 **아슬아슬한 타이밍**에 정확히 들어맞을 때만 발생합니다. 전형적인 **레이스 컨디션(Race Condition)** 문제입니다.

## 3. 원인 추적: 내 배치 Job의 범인은 누구인가?

이 유령의 정체를 밝히기 위해, 다음 세 가지를 체계적으로 점검해야 합니다.

### 1단계: 격리 수준 확인하기 (가장 유력한 용의자)

"우리 배치 Job이 정말 `SERIALIZABLE` 격리 수준으로 동작하고 있는가?"를 확인해야 합니다.

- **`@Transactional` 어노테이션:** `Job`이나 `Step` 설정에 `@Transactional(isolation = Isolation.SERIALIZABLE)`이 명시되어 있는지 확인합니다.
- **`DataSource` 설정:** `application.properties`나 `DataSource` Bean 설정에서 `spring.datasource.hikari.transaction-isolation=TRANSACTION_SERIALIZABLE` 과 같이 전역으로 설정되었는지 확인합니다.

### 2단계: 동시성 유발자 찾기

누가 '트랜잭션 B'의 역할을 하고 있는지 찾아야 합니다.

- **다른 배치 Job:** 문제의 Job과 동시에 실행되도록 스케줄링된 다른 Job이 동일한 테이블을 수정하는지 확인합니다.
- **실시간 API:** 운영 중인 웹 애플리케이션의 API가 배치 Job이 처리하는 테이블을 실시간으로 수정하는지 확인합니다.
- **DBA 또는 수동 쿼리:** DBA나 다른 개발자가 운영 DB에 직접 수정 쿼리를 실행하는 타이밍과 겹치는지 확인합니다. (로그 시간 교차 분석)

### 3단계: 배치 로직 패턴 분석하기

배치 Job 내부 로직이 **"읽고 나서 바로 쓰는(Read-Modify-Write)"** 패턴을 가지고 있는지 확인합니다. `ItemProcessor`에서 읽어온 데이터를 수정한 뒤 `ItemWriter`로 넘기는 대부분의 배치가 이 패턴에 해당하며, `SERIALIZABLE` 환경에서 `ORA-08177` 오류에 매우 취약합니다.

## 4. 문제 해결을 위한 3가지 실전 전략

원인을 파악했다면, 상황에 맞는 해결책을 적용할 수 있습니다.

### 전략 1 (가장 현실적인 해결책): 격리 수준 하향 조정

가장 먼저 자문해야 할 질문은 **"이 배치 Job에 정말 `SERIALIZABLE` 수준의 완벽한 정합성이 필요한가?"** 입니다. 99%의 경우, 답은 '아니오'입니다.

`SERIALIZABLE`은 동시성을 크게 저하시키므로, 매우 특수한 금융 정산 로직 등이 아니면 거의 사용하지 않습니다. Spring의 기본값인 `READ_COMMITTED`로 낮추는 것이 가장 일반적이고 효과적인 해결책입니다.

```java
import org.springframework.batch.core.Step;
import org.springframework.batch.core.repository.JobRepository;
import org.springframework.batch.core.step.builder.StepBuilder;
import org.springframework.transaction.PlatformTransactionManager;
import org.springframework.transaction.TransactionDefinition;
import org.springframework.transaction.interceptor.DefaultTransactionAttribute;

@Bean
public Step myStep(JobRepository jobRepository, PlatformTransactionManager transactionManager) {
    // Step의 트랜잭션 속성을 직접 설정
    DefaultTransactionAttribute attribute = new DefaultTransactionAttribute();
    // 격리 수준을 READ_COMMITTED로 명시 (Spring 기본값)
    attribute.setIsolationLevel(TransactionDefinition.ISOLATION_READ_COMMITTED);

    return new StepBuilder("myStep", jobRepository)
            .<Source, Target>chunk(100, transactionManager)
            .reader(reader())
            .writer(writer())
            .transactionAttribute(attribute) // Step에 커스텀 트랜잭션 속성 적용
            .build();
}
```

### 전략 2 (`SERIALIZABLE`이 필수일 때): 비관적 락(Pessimistic Lock)

`SERIALIZABLE` 격리 수준을 포기할 수 없는 비즈니스 로직이라면, 다른 트랜잭션이 내 데이터에 아예 접근하지 못하도록 원천적으로 막아야 합니다.

`ItemReader`의 조회 쿼리에 **`FOR UPDATE`** 절을 추가하여, 읽는 시점에 해당 로우(Row)에 배타적 락(Exclusive Lock)을 겁니다.

```xml
<!-- MyBatis Mapper XML -->
<select id="selectDataForBatch" resultType="com.example.MyDto">
  SELECT *
  FROM MY_TABLE
  WHERE status = 'PENDING'
  FOR UPDATE
</select>
```

- **효과:** 내 배치 트랜잭션이 `SELECT ... FOR UPDATE`로 데이터를 읽으면, 다른 트랜잭션은 내 트랜잭션이 끝날 때까지 해당 데이터의 수정을 시도조차 못하고 대기하게 됩니다. `ORA-08177`이 발생할 상황 자체가 만들어지지 않습니다.
- **단점:** 동시성이 크게 저하됩니다. 락을 건 트랜잭션이 길어지면 시스템 전체의 병목이 될 수 있습니다.

### 전략 3 (유연한 대안): 재시도(Retry) 로직 구현

`ORA-08177`은 "지금은 안되니, 나중에 다시 시도해봐" 라는 의미를 내포한, **재시도 가능한(retryable)** 오류입니다. Spring Batch는 청크(Chunk) 단위로 실패 시 재시도하는 강력한 기능을 제공합니다.

```java
import org.springframework.batch.core.step.builder.StepBuilder;
import org.springframework.dao.CannotSerializeTransactionException;

@Bean
public Step myRetryableStep(JobRepository jobRepository, PlatformTransactionManager transactionManager) {
    return new StepBuilder("myRetryableStep", jobRepository)
            .<Source, Target>chunk(100, transactionManager)
            .reader(reader())
            .writer(writer())
            .faultTolerant() // 내고장성 기능 활성화
            .retryLimit(3)   // 최대 3번 재시도
            .retry(CannotSerializeTransactionException.class) // ORA-08177에 매핑되는 Spring 예외
            .build();
}
```

- **효과:** 비관적 락보다 동시성을 덜 해치면서, 간헐적인 동시성 충돌을 유연하게 극복할 수 있습니다.
- **단점:** 재시도가 반복되면 배치 전체의 수행 시간이 길어질 수 있고, 재시도 시에도 동일한 문제가 계속 발생할 수 있습니다.

## 5. 결론: 트랜잭션 격리 수준은 트레이드오프(Trade-off)다

Spring Batch에서 발생하는 간헐적인 `ORA-08177` 오류는 코드의 버그가 아니라, **과도하게 높은 트랜잭션 격리 수준(정합성)**과 **동시성** 사이의 필연적인 충돌입니다.

이 문제를 마주쳤을 때, 해결의 첫걸음은 기술적인 기교가 아니라, **"우리 비즈니스에 정말 `SERIALIZALBE` 수준의 완벽한 정합성이 필요한가?"**를 분석하고, 요구사항에 맞는 **적절한 격리 수준을 선택**하는 것입니다. 트랜잭션 격리 수준은 단순한 이론적 개념이 아니라, 실제 시스템의 안정성과 성능에 지대한 영향을 미치는 실전적인 설계 요소임을 기억해야 합니다.
