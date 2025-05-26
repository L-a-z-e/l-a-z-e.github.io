---
title: Spring Batch - Configuring a Step (Retry Logic)
description: 
author: laze
date: 2025-05-26 00:00:01 +0900
categories: [Dev, SpringBatch]
tags: [SpringBatch]
---
**Retry 로직 설정하기**

대부분의 경우, 예외가 발생하면 Skip을 하거나 `Step` 실패를 유발하기를 원할 것입니다.

그러나 모든 예외가 결정론적인(deterministic) 것은 아닙니다.

읽기 중에 `FlatFileParseException`이 발생하면 해당 레코드에 대해 항상 해당 예외가 발생합니다.

`ItemReader`를 재설정해도 도움이 되지 않습니다.

그러나 다른 예외(예: `DeadlockLoserDataAccessException`, 이는 현재 프로세스가 다른 프로세스가 잠금을 보유한 레코드를 업데이트하려고 시도했음을 나타냄)의 경우, 기다렸다가 다시 시도하면 성공할 수 있습니다.

Java에서 Retry는 다음과 같이 구성해야 합니다.

```java
@Bean
public Step step1(JobRepository jobRepository, PlatformTransactionManager transactionManager) {
	return new StepBuilder("step1", jobRepository)
				.<String, String>chunk(2, transactionManager) // 청크 크기 예시
				.reader(itemReader())      // itemReader()는 실제 빈을 참조해야 합니다.
				.writer(itemWriter())      // itemWriter()도 실제 빈을  참조해야 합니다.
				.faultTolerant()           // 내결함성 기능 활성화
				.retryLimit(3)             // 개별 아이템당 최대 3번까지 재시도 허용
				.retry(DeadlockLoserDataAccessException.class) // 이 예외 발생 시 재시도
				.build();
}
```

`Step`은 개별 항목을 재시도할 수 있는 횟수 제한과 "재시도 가능한(retryable)" 예외 목록을 허용합니다.

---

## 실패해도 다시 한번! Step의 끈기, Retry 로직! 🦾🔁

우리가 앞에서 Skip 로직을 배웠죠? "이런 오류는 그냥 건너뛰자!" 하는 거였어요. 그런데 어떤 오류들은 **지금 당장은 실패했지만, 잠깐 기다렸다가 다시 시도하면 성공할 수도 있는 경우**가 있어요.

**예시:**

- **일시적인 네트워크 문제:** 데이터베이스에 데이터를 저장하려고 하는데, 아주 잠깐 네트워크 연결이 불안정해서 실패했어요. 몇 초 뒤에 다시 시도하면 성공할 가능성이 높겠죠?
- **데이터베이스 잠금(Lock) 충돌:** 내가 수정하려는 데이터를 다른 작업이 잠깐 동안 잠그고 있어서 실패했어요. (문서 예제의 `DeadlockLoserDataAccessException`이 이런 경우예요!) 다른 작업의 잠금이 풀린 후에 다시 시도하면 성공할 수 있어요.
- **외부 API 호출 실패:** 외부 서비스 API를 호출했는데, 그 서비스가 아주 잠깐 응답이 없었어요. 다시 호출하면 정상 응답을 받을 수도 있겠죠?

이런 **"일시적일 수 있는" 오류**들에 대해서는 무조건 `Step`을 실패시키거나, 데이터를 건너뛰는 것보다, **몇 번 더 시도해보는 것(Retry)**이 훨씬 효율적일 수 있어요.

---

### Retry 로직 설정의 핵심 듀오!

Skip 로직과 마찬가지로 Retry 로직도 `StepBuilder`를 사용할 때 설정하며, 먼저 **`.faultTolerant()`**를 호출해서 내결함성 기능을 활성화해야 해요.

1. **`.faultTolerant()`**: 내결함성 기능 활성화! Skip, Retry 같은 고급 오류 처리 기능을 사용하기 위한 필수 선언이에요.
2. **`.retryLimit(횟수)`**: "이 아이템, 최대 이만큼만 다시 시도해볼게!"
  - **개별 아이템(item) 하나**에 대해 재시도할 수 있는 **최대 횟수**를 설정해요.
  - 예를 들어, `.retryLimit(3)`으로 설정하면, 특정 아이템을 처리(Read, Process, Write 중 어느 단계든)하다가 재시도 가능한 오류가 발생했을 때, 최대 3번까지 그 아이템에 대한 처리를 다시 시도해요.
  - 만약 3번을 모두 재시도했는데도 계속 실패하면, 그때는 이 아이템에 대해 최종적으로 실패 처리를 해요. (이후 Skip 로직이 적용될 수도 있고, 아니면 `Step` 전체가 실패할 수도 있어요.)
3. **`.retry(Exception클래스)`**: "이런 종류의 오류는 다시 시도해보자!"
  - 재시도할 **특정 예외(Exception) 클래스**를 지정해요.
  - 여기에 지정된 예외 또는 그 예외의 하위 클래스에 해당하는 오류가 발생하면, 해당 아이템 처리를 재시도해요.
  - 여러 번 호출해서 다양한 예외를 재시도 대상으로 추가할 수 있어요.
  - 예시: `.retry(DeadlockLoserDataAccessException.class)` - 데이터베이스 락 충돌 오류는 재시도하겠다!

---

### Retry 로직 작동 방식 및 예시

**기본적인 Retry 설정 예시 (Java):**

```java
@Bean
public Step resilientStep(JobRepository jobRepository, PlatformTransactionManager transactionManager,
                          ItemReader<Order> reader, ItemProcessor<Order, ProcessedOrder> processor, ItemWriter<ProcessedOrder> writer) {
    return new StepBuilder("resilientStep", jobRepository)
            .<Order, ProcessedOrder>chunk(5, transactionManager) // 청크 크기는 5
            .reader(reader)
            .processor(processor)
            .writer(writer)
            .faultTolerant() // 1. 내결함성 기능 활성화!
            .retryLimit(3)   // 2. 아이템당 최대 3번 재시도
            .retry(OptimisticLockingFailureException.class) // 3. OptimisticLockingFailureException 발생 시 재시도
            .retry(TransientDataAccessResourceException.class) //    TransientDataAccessResourceException 발생 시에도 재시도
            .build();
}

```

**작동 시나리오 (한 아이템에 대해):**

1. `ItemProcessor`가 특정 `Order` 아이템을 처리하다가 `OptimisticLockingFailureException`이 발생했어요.
  - 이 예외는 `.retry()`에 지정된 재시도 대상 예외예요.
  - 현재 이 아이템에 대한 재시도 횟수는 0번이에요. `.retryLimit(3)`보다 작죠?
  - 따라서 Spring Batch는 이 `Order` 아이템에 대한 처리를 **다시 시도**해요. (보통 현재 청크를 롤백하고, 해당 아이템부터 다시 읽어서 처리하는 방식으로 동작할 수 있어요. 세부 동작은 복잡할 수 있습니다.)
2. 두 번째 시도에서도 또 `OptimisticLockingFailureException`이 발생했어요.
  - 여전히 재시도 대상 예외이고, 재시도 횟수는 이제 1번이에요. `.retryLimit(3)`보다 작죠?
  - 다시 한번 이 `Order` 아이템 처리를 **재시도**해요.
3. 세 번째 시도에서도 또! `OptimisticLockingFailureException`이 발생했어요.
  - 재시도 대상 예외이고, 재시도 횟수는 이제 2번이에요. `.retryLimit(3)`보다 작죠?
  - 마지막으로 한번 더 이 `Order` 아이템 처리를 **재시도**해요.
4. 네 번째 시도에서도 또!! `OptimisticLockingFailureException`이 발생했어요. (총 3번 재시도 후 네 번째 원래 시도)
  - 이제 이 아이템에 대한 재시도 횟수가 `retryLimit(3)`에 도달했어요.
  - 더 이상 이 아이템에 대해 재시도하지 않아요.
  - 이때, 이 `OptimisticLockingFailureException`이 `.skip()` 로직에도 해당한다면 Skip 처리될 것이고, 그렇지 않다면 `Step` 전체가 실패하게 될 수 있어요. (Retry와 Skip은 함께 사용될 수 있으며, Retry가 모두 실패한 후 Skip 여부가 결정돼요.)

**중요한 점:**

- **`retryLimit`은 아이템 하나에 대한 재시도 횟수 제한**이에요. `Step` 전체에 대한 제한이 아니에요.
- 재시도는 청크 처리의 **Read, Process, Write 어느 단계에서든** 발생할 수 있는 재시도 가능 예외에 대해 동작해요.
- **재시도를 한다는 것은, 현재 진행 중이던 청크에 문제가 생겼다는 의미일 수 있어요.** 그래서 Spring Batch는 재시도 시 현재 청크를 롤백하고, 문제가 발생한 아이템(또는 그 이전 아이템부터) 다시 처리하는 복잡한 과정을 내부적으로 수행할 수 있어요. (정확한 동작은 예외 발생 지점과 설정에 따라 달라질 수 있습니다.)

---

### Skip 로직과의 관계

Retry와 Skip은 함께 사용될 수 있어요. 순서는 보통 다음과 같아요:

1. 아이템 처리 중 오류 발생!
2. 이 오류가 **Retry 대상**인가?
  - **Yes** -> `retryLimit`까지 재시도. 모두 실패하면 다음 단계로.
  - **No** -> 다음 단계로.
3. (Retry가 모두 실패했거나, Retry 대상이 아니었다면) 이 오류가 **Skip 대상**인가?
  - **Yes** -> `skipLimit`까지 Skip. 모두 실패(Skip 한도 초과)하면 `Step` 실패.
  - **No** -> `Step` 즉시 실패.

즉, **Retry가 Skip보다 먼저 시도된다고 생각할 수 있어요.** "일단 다시 해보고, 그래도 안 되면 건너뛰거나 실패 처리하자!"는 거죠.

---

### Retry 로직, 언제 사용하면 좋을까요?

- **일시적인 문제로 인해 발생할 가능성이 높은 예외**를 처리할 때.
  - 네트워크 타임아웃, 데이터베이스 데드락, 외부 서비스의 일시적인 응답 없음 등.
- **결정론적이지 않은(non-deterministic) 오류:** 지금은 실패했지만, 조금 후에 다시 하면 성공할 수도 있는 오류들.
  - 파일 파싱 오류(`FlatFileParseException`)처럼 데이터 자체에 문제가 있어서 항상 실패하는 오류는 Retry 대상이 아니에요. 이런 건 Skip 대상이거나 그냥 실패 처리해야겠죠.

**주의사항:**

- **무한 재시도 방지:** `retryLimit`을 적절히 설정해서 무한 루프에 빠지지 않도록 해야 해요.
- **재시도 대상 예외의 신중한 선택:** 모든 예외를 재시도 대상으로 하면, 정말 심각한 문제를 계속 재시도하느라 작업이 끝나지 않을 수 있어요.
- **멱등성(Idempotence) 고려:** 재시도되는 작업이 여러 번 실행되어도 결과가 동일하게 유지되는지(멱등성) 확인해야 할 수 있어요. (특히 Write 단계에서)

**핵심:** Retry 로직은 **일시적인 오류를 극복하고 배치 작업의 안정성을 높이는 데 매우 유용한 기능**이에요. "혹시나 될까?" 하는 희망을 가지고 몇 번 더 시도해보는 끈기를 우리 `Step`에게 부여하는 거죠!

---
