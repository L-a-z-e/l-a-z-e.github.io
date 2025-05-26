---
title: Spring Batch - Configuring a Step (Controlling Rollback)
description: 
author: laze
date: 2025-05-26 00:00:03 +0900
categories: [Dev, SpringBatch]
tags: [SpringBatch]
---
**롤백 제어하기 (Controlling Rollback)**

기본적으로 Retry나 Skip 설정과 관계없이, `ItemWriter`에서 발생하는 모든 예외는 `Step`에 의해 제어되는 트랜잭션을 롤백시킵니다.

앞에서 설명한 대로 Skip이 구성된 경우, `ItemReader`에서 발생하는 예외는 롤백을 유발하지 않습니다.

그러나 트랜잭션을 무효화하는 어떠한 작업도 발생하지 않았기 때문에 `ItemWriter`에서 발생하는 예외가 롤백을 유발해서는 안 되는 시나리오가 많습니다.

이러한 이유로, 롤백을 유발하지 않아야 하는 예외 목록으로 `Step`을 구성할 수 있습니다.

Java에서는 다음과 같이 롤백을 제어할 수 있습니다.

```java
@Bean
public Step step1(JobRepository jobRepository, PlatformTransactionManager transactionManager) {
	return new StepBuilder("step1", jobRepository)
				.<String, String>chunk(2, transactionManager) // 청크 크기 예시
				.reader(itemReader())      // itemReader()는 실제 빈을 참조해야 합니다.
				.writer(itemWriter())      // itemWriter()도 실제 빈을 참조해야 합니다.
				.faultTolerant()           // 내결함성 기능 활성화 (noRollback 사용 시 필요할 수 있음)
				.noRollback(ValidationException.class) // ValidationException 발생 시 롤백하지 않음
				.build();
}
```

**트랜잭션 리더 (Transactional Readers)**

`ItemReader`의 기본 계약은 앞으로만 이동(forward-only)한다는 것입니다.

`Step`은 리더 입력을 버퍼링하여, 롤백 발생 시 리더에서 항목을 다시 읽을 필요가 없도록 합니다.

그러나 JMS 큐와 같이 리더가 트랜잭션 리소스 위에 구축되는 특정 시나리오가 있습니다.

이 경우 큐가 롤백되는 트랜잭션에 연결되어 있으므로 큐에서 가져온 메시지가 다시 큐로 돌아갑니다.

이러한 이유로 `Step`이 항목을 버퍼링하지 않도록 구성할 수 있습니다.

다음 예는 Java에서 항목을 버퍼링하지 않는 리더를 만드는 방법을 보여줍니다.

```java
@Bean
public Step step1(JobRepository jobRepository, PlatformTransactionManager transactionManager) {
	return new StepBuilder("step1", jobRepository)
				.<String, String>chunk(2, transactionManager)
				.reader(itemReader()) // itemReader는 트랜잭션 기반 리소스(예: JMS)를 사용하는 구현이어야 함
				.writer(itemWriter())
				.readerIsTransactionalQueue() // 리더가 트랜잭션 큐임을 명시 (버퍼링 X)
				.build();
}
```

---

## 트랜잭션 되돌리기, 내 마음대로! - 롤백 제어와 트랜잭션 리더 🔄🛡️

우리가 청크 지향 `Step`에서 `ItemWriter`가 데이터를 쓸 때, 그 작업은 하나의 트랜잭션 안에서 이루어진다고 했죠?

만약 `ItemWriter`에서 데이터를 쓰는 도중에 오류가 발생하면, 기본적으로 Spring Batch는 "이 청크 작업은 문제가 생겼으니 없었던 일로 하자!" 하면서 해당 트랜잭션을 **롤백(Rollback)**시켜요.

즉, 그 청크에서 데이터베이스 등에 변경하려던 내용을 모두 취소하는 거죠. 이게 대부분의 경우 안전하고 올바른 동작이에요.

하지만 때로는 `ItemWriter`에서 오류가 발생했더라도, **"이 정도 오류는 롤백까지 할 필요는 없어. 그냥 이 아이템만 문제 삼고 넘어가자!"** 하고 싶을 때가 있어요.

---

### 1. 특정 예외는 롤백 안 하기! - `.noRollback(Exception클래스)`

- **기본 동작:**
  - `ItemReader`에서 오류 발생 시: 보통 트랜잭션 롤백이 발생하지 않아요. (문제가 된 아이템만 Skip되거나 `Step`이 실패할 수 있음)
  - `ItemProcessor`에서 오류 발생 시: 보통 트랜잭션 롤백이 발생하지 않아요. (마찬가지로 Skip 또는 `Step` 실패)
  - **`ItemWriter`에서 오류 발생 시: 기본적으로 현재 청크에 대한 트랜잭션이 롤백돼요!**
- **`noRollback(Exception클래스)` 기능:**
  - `ItemWriter`에서 특정 예외가 발생했을 때, **트랜잭션 롤백을 막고 싶을 때** 사용해요.
  - 즉, "이 예외가 발생해도, 이미 이 청크에서 다른 아이템들은 DB에 잘 써졌다면 그건 그대로 두고, 문제 된 아이템만 처리(예: Skip)하자!" 라는 의미예요.
  - 이 기능을 사용하려면 보통 `.faultTolerant()`를 함께 선언해야 할 수 있어요 (Skip 로직과 연계될 수 있기 때문).
- **설정 방법 (Java):**

    ```java
    @Bean
    public Step noRollbackStep(JobRepository jobRepository, PlatformTransactionManager transactionManager,
                               ItemReader<Data> reader, ItemWriter<Data> writer) {
        return new StepBuilder("noRollbackStep", jobRepository)
                .<Data, Data>chunk(10, transactionManager)
                .reader(reader)
                .writer(writer)
                .faultTolerant() // 내결함성 기능 (Skip/Retry 등과 연계될 수 있음)
                .skipLimit(5) // 예: 최대 5개까지 Skip 허용
                .skip(ValidationException.class) // ValidationException 발생 시 Skip
                .noRollback(ValidationException.class) // 그리고 ValidationException 발생 시 롤백도 하지 않음!
                .build();
    }
    
    ```

  위 예시에서 `ItemWriter` 실행 중 `ValidationException`이 발생하면:

  1. 해당 아이템은 `.skip(ValidationException.class)`에 의해 Skip 처리될 수 있어요.
  2. 그리고 `.noRollback(ValidationException.class)` 설정 때문에, 현재 청크의 트랜잭션은 **롤백되지 않아요.** 즉, `ValidationException`이 발생하기 전까지 이 청크에서 `ItemWriter`가 성공적으로 DB에 썼던 다른 데이터들은 그대로 유지(커밋될 수 있음)되고, 문제가 된 아이템만 Skip되는 거죠.
- **언제 유용할까요?**
  - 하나의 청크 내에 여러 아이템을 쓰는 상황에서, 일부 아이템에만 사소한 문제가 있고(예: 단순 유효성 검사 실패), 그 문제 때문에 이미 성공적으로 처리된 다른 아이템들의 작업까지 취소하고 싶지 않을 때.
  - **주의!** 이 기능을 사용할 때는 데이터 일관성이 깨지지 않도록 신중하게 설계해야 해요. 어떤 오류에 대해 롤백을 하지 않을지는 비즈니스 규칙과 데이터의 특성을 잘 이해하고 결정해야 합니다. 잘못 사용하면 데이터가 꼬일 수 있어요!

**핵심:** `.noRollback()`은 `ItemWriter`에서 발생하는 **특정 예외에 대해 트랜잭션 롤백을 선택적으로 방지**하여, 유연한 오류 처리를 가능하게 하지만, 데이터 일관성을 해치지 않도록 주의해서 사용해야 해요!

---

### 2. 특별한 `ItemReader`를 위한 배려 - `.readerIsTransactionalQueue()`

Spring Batch의 청크 지향 `Step`은 기본적으로 다음과 같이 동작한다고 했어요:

1. `ItemReader`가 청크 크기만큼 아이템들을 읽어서 **내부 버퍼(buffer)에 담아둬요.**
2. 이 버퍼에 담긴 아이템들을 `ItemProcessor`로 처리하고, `ItemWriter`로 써요.
3. 만약 쓰기 과정에서 오류가 나서 트랜잭션이 롤백되면, **이미 버퍼에 담아둔 아이템들이 있기 때문에 `ItemReader`가 처음부터 다시 데이터를 읽어올 필요가 없어요.** 버퍼에 있는 아이템들을 재처리하거나 Skip 할 수 있죠. (효율성!)

그런데 어떤 `ItemReader`들은 이런 방식과 잘 맞지 않을 수 있어요. 대표적인 예가 **트랜잭션 기반의 메시지 큐(Message Queue, 예: JMS Queue)에서 메시지를 읽어오는 `ItemReader`**예요.

- **트랜잭션 큐의 특징:**
  - 메시지 큐에서 메시지를 "꺼내오는(consume)" 작업 자체가 트랜잭션의 일부로 동작해요.
  - 만약 해당 트랜잭션이 롤백되면, 꺼내왔던 메시지들이 **다시 큐로 되돌아가는(put back)** 것이 일반적인 동작이에요. (메시지 유실 방지!)
- **문제점:**
  - 만약 Spring Batch가 이런 트랜잭션 큐에서 읽어온 메시지들을 내부 버퍼에 저장하고 있는데, 청크 처리 중 오류가 나서 트랜잭션이 롤백됐다고 해봐요.
  - 트랜잭션 롤백으로 인해 메시지들은 다시 큐로 돌아갔어요.
  - 그런데 Spring Batch의 버퍼에는 이미 그 메시지들이 남아있죠?
  - 다음에 이 버퍼에 있는 메시지들을 재처리하려고 하면, 이미 큐로 돌아간 메시지들이기 때문에 문제가 생기거나, 또는 큐에서 다시 같은 메시지를 중복으로 읽어올 수도 있어요.
- **해결책: `.readerIsTransactionalQueue()`**
  - `ItemReader`가 JMS 큐처럼 트랜잭션 리소스와 직접 연동되어, 트랜잭션 롤백 시 읽었던 아이템이 자동으로 원래 위치로 복구되는 경우에 이 설정을 사용해요.
  - 이 설정을 하면, Spring Batch는 `ItemReader`로부터 읽어온 아이템들을 **내부 버퍼에 저장하지 않아요.**
  - 대신, 트랜잭션이 롤백되면 `ItemReader`가 다음번 `read()` 호출 시 (큐로 되돌아간) 메시지를 다시 읽어올 것이라고 기대해요.
- **설정 방법 (Java):**

    ```java
    @Bean
    public Step jmsStep(JobRepository jobRepository, PlatformTransactionManager transactionManager,
                        ItemReader<Message> jmsItemReader, ItemWriter<Message> itemWriter) {
        return new StepBuilder("jmsStep", jobRepository)
                .<Message, Message>chunk(5, transactionManager)
                .reader(jmsItemReader) // 이 ItemReader는 JMS 큐에서 메시지를 읽어오는 구현체여야 함
                .writer(itemWriter)
                .readerIsTransactionalQueue() // "내 ItemReader는 트랜잭션 큐에서 읽어오니까 버퍼링 하지마!"
                .build();
    }
    
    ```


**핵심:** `.readerIsTransactionalQueue()`는 `ItemReader`가 **트랜잭션 리소스(주로 메시지 큐)를 사용할 때, Spring Batch의 기본 버퍼링 동작을 끄고, 트랜잭션 롤백 시 리소스 자체의 복구 메커니즘에 의존**하도록 하는 특별한 설정이에요.

---
