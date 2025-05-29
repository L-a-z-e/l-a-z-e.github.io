---
title: Spring Batch - Configuring a Step (Intercepting Step Execution)
description: 
author: laze
date: 2025-05-29 00:00:02 +0900
categories: [Dev, SpringBatch]
tags: [SpringBatch]
---
## Step 실행 가로채기 (Intercepting Step Execution)

Job과 마찬가지로, Step 실행 중에는 사용자가 특정 기능을 수행해야 할 수 있는 많은 이벤트 지점이 있습니다.

예를 들어, 바닥글(footer)이 필요한 플랫 파일에 쓰기 위해서는 `ItemWriter`가 Step 완료 시점을 통지받아 바닥글을 쓸 수 있어야 합니다.

이는 여러 Step 범위의 리스너(Step scoped listeners) 중 하나를 사용하여 달성할 수 있습니다.

`StepListener`의 확장 중 하나를 구현하는 모든 클래스(단, `StepListener` 인터페이스 자체는 비어있으므로 해당 인터페이스는 아님)를 `listeners` 요소를 통해 Step에 적용할 수 있습니다.

`listeners` 요소는 step, tasklet, 또는 chunk 선언 내에서 유효합니다.

리스너는 그 기능이 적용되는 수준에서 선언하거나, 다중 기능(예: `StepExecutionListener`와 `ItemReadListener`를 모두 구현)을 가졌다면 적용되는 가장 세분화된 수준에서 선언하는 것을 권장합니다.

다음 예제는 자바에서 chunk 수준에 리스너를 적용하는 방법을 보여줍니다:

**자바 설정 (Java Configuration)**

```java
@Bean
public Step step1(JobRepository jobRepository, PlatformTransactionManager transactionManager) {
	return new StepBuilder("step1", jobRepository)
				.<String, String>chunk(10, transactionManager)
				.reader(reader())
				.writer(writer())
				.listener(chunkListener()) // chunkListener()를 리스너로 등록
				.build();
}
```

`ItemReader`, `ItemWriter`, 또는 `ItemProcessor` 자체가 `StepListener` 인터페이스 중 하나를 구현하는 경우, 네임스페이스 `<step>` 요소 또는 `*StepFactoryBean` 팩토리 중 하나를 사용한다면 Step에 자동으로 등록됩니다.

이는 Step에 직접 주입된 컴포넌트에만 적용됩니다. 만약 리스너가 다른 컴포넌트 내부에 중첩되어 있다면, (이전에 "Step에 ItemStream 등록하기"에서 설명한 것처럼) 명시적으로 등록해야 합니다.

`StepListener` 인터페이스 외에도, 동일한 문제를 해결하기 위한 어노테이션이 제공됩니다.

일반 자바 객체(POJO)는 이러한 어노테이션이 붙은 메소드를 가질 수 있으며, 이는 해당 `StepListener` 타입으로 변환됩니다.

또한 `ItemReader`, `ItemWriter`, 또는 `Tasklet`과 같은 청크 컴포넌트의 커스텀 구현체에 어노테이션을 사용하는 것도 일반적입니다.

어노테이션은 `<listener/>` 요소에 대한 XML 파서에 의해 분석될 뿐만 아니라 빌더의 리스너 메소드에도 등록되므로, XML 네임스페이스나 빌더를 사용하여 리스너를 Step에 등록하기만 하면 됩니다.

### `StepExecutionListener`

`StepExecutionListener`는 Step 실행에 대한 가장 일반적인 리스너를 나타냅니다.

다음 예제와 같이 Step이 시작되기 전과 종료된 후(정상 종료 또는 실패 여부에 관계없이) 알림을 받을 수 있도록 합니다:

```java
public interface StepExecutionListener extends StepListener {

    void beforeStep(StepExecution stepExecution);

    ExitStatus afterStep(StepExecution stepExecution);

}
```

`afterStep`은 리스너가 Step 완료 시 반환되는 종료 코드(exit code)를 수정할 기회를 주기 위해 `ExitStatus` 반환 타입을 가집니다.

이 인터페이스에 해당하는 어노테이션은 다음과 같습니다:

- `@BeforeStep`
- `@AfterStep`

### `ChunkListener`

"청크(chunk)"는 트랜잭션 범위 내에서 처리되는 아이템들로 정의됩니다. 각 커밋 간격(commit interval)에서 트랜잭션을 커밋하는 것은 청크를 커밋하는 것입니다.

다음 인터페이스 정의와 같이, `ChunkListener`를 사용하여 청크 처리가 시작되기 전이나 청크가 성공적으로 완료된 후에 로직을 수행할 수 있습니다:

```java
public interface ChunkListener extends StepListener {

    void beforeChunk(ChunkContext context);
    void afterChunk(ChunkContext context);
    void afterChunkError(ChunkContext context); // 청크 처리 중 오류 발생 시 호출

}
```

`beforeChunk` 메소드는 트랜잭션이 시작된 후, `ItemReader`에서 읽기가 시작되기 전에 호출됩니다.

반대로, `afterChunk`는 청크가 커밋된 후에 호출됩니다 (롤백이 있는 경우 전혀 호출되지 않음).

이 인터페이스에 해당하는 어노테이션은 다음과 같습니다:

- `@BeforeChunk`
- `@AfterChunk`
- `@AfterChunkError`

청크 선언이 없는 경우에도 `ChunkListener`를 적용할 수 있습니다.

`TaskletStep`이 `ChunkListener`를 호출할 책임이 있으므로, 아이템 지향이 아닌 tasklet에도 적용됩니다 (tasklet 실행 전후에 호출됨).

`ChunkListener`는 검사 예외(checked exceptions)를 던지도록 설계되지 않았습니다. 오류는 구현 내에서 처리되어야 하며, 그렇지 않으면 step이 종료됩니다.

### `ItemReadListener`

이전에 스킵 로직(skip logic)을 논의할 때, 스킵된 레코드를 로깅하여 나중에 처리할 수 있도록 하는 것이 유용할 수 있다고 언급했습니다.

읽기 오류의 경우, 다음 인터페이스 정의와 같이 `ItemReadListener`를 사용하여 이를 수행할 수 있습니다:

```java
public interface ItemReadListener<T> extends StepListener {

    void beforeRead();
    void afterRead(T item);
    void onReadError(Exception ex);

}
```

`beforeRead` 메소드는 `ItemReader`에서 `read`를 호출하기 직전에 매번 호출됩니다.

`afterRead` 메소드는 `read`가 성공적으로 호출된 후 매번 호출되며 읽힌 아이템이 전달됩니다.

읽는 동안 오류가 발생하면 `onReadError` 메소드가 호출됩니다. 발생한 예외가 제공되므로 로깅할 수 있습니다.

이 인터페이스에 해당하는 어노테이션은 다음과 같습니다:

- `@BeforeRead`
- `@AfterRead`
- `@OnReadError`

### `ItemProcessListener`

`ItemReadListener`와 마찬가지로, 다음 인터페이스 정의와 같이 아이템의 처리를 "수신(listen)"할 수 있습니다:

```java
public interface ItemProcessListener<T, S> extends StepListener {

    void beforeProcess(T item);
    void afterProcess(T item, S result);
    void onProcessError(T item, Exception e);

}
```

`beforeProcess` 메소드는 `ItemProcessor`에서 `process`가 호출되기 전에 호출되며 처리될 아이템이 전달됩니다.

`afterProcess` 메소드는 아이템이 성공적으로 처리된 후에 호출됩니다.

처리 중 오류가 발생하면 `onProcessError` 메소드가 호출됩니다.

발생한 예외와 처리를 시도했던 아이템이 제공되므로 로깅할 수 있습니다.

이 인터페이스에 해당하는 어노테이션은 다음과 같습니다:

- `@BeforeProcess`
- `@AfterProcess`
- `@OnProcessError`

### `ItemWriteListener`

다음 인터페이스 정의와 같이 `ItemWriteListener`를 사용하여 아이템 쓰기를 "수신(listen)"할 수 있습니다:

```java
public interface ItemWriteListener<S> extends StepListener {

    void beforeWrite(List<? extends S> items);
    void afterWrite(List<? extends S> items);
    void onWriteError(Exception exception, List<? extends S> items);

}
```

`beforeWrite` 메소드는 `ItemWriter`에서 `write`가 호출되기 전에 호출되며 쓰여질 아이템 목록이 전달됩니다.

`afterWrite` 메소드는 아이템이 성공적으로 쓰여진 후, 청크 처리와 관련된 트랜잭션이 커밋되기 전에 호출됩니다.

쓰기 중 오류가 발생하면 `onWriteError` 메소드가 호출됩니다.

발생한 예외와 쓰기를 시도했던 아이템이 제공되므로 로깅할 수 있습니다.

이 인터페이스에 해당하는 어노테이션은 다음과 같습니다:

- `@BeforeWrite`
- `@AfterWrite`
- `@OnWriteError`

### `SkipListener`

`ItemReadListener`, `ItemProcessListener`, `ItemWriteListener`는 모두 오류 알림 메커니즘을 제공하지만, 레코드가 실제로 스킵되었음을 알려주지는 않습니다.

예를 들어, `onWriteError`는 아이템이 재시도되어 성공하더라도 호출됩니다. 이러한 이유로, 다음 인터페이스 정의와 같이 스킵된 아이템을 추적하기 위한 별도의 인터페이스가 있습니다:

```java
public interface SkipListener<T,S> extends StepListener {

    void onSkipInRead(Throwable t); // 읽기 중 스킵 발생 시
    void onSkipInProcess(T item, Throwable t); // 처리 중 스킵 발생 시
    void onSkipInWrite(S item, Throwable t); // 쓰기 중 스킵 발생 시

}
```

`onSkipInRead`는 읽는 동안 아이템이 스킵될 때마다 호출됩니다.

롤백으로 인해 동일한 아이템이 두 번 이상 스킵된 것으로 등록될 수 있음에 유의해야 합니다.

`onSkipInWrite`는 쓰는 동안 아이템이 스킵될 때 호출됩니다.

아이템이 성공적으로 읽혔기 때문에(그리고 스킵되지 않았기 때문에), 아이템 자체가 인수로 제공됩니다.

이 인터페이스에 해당하는 어노테이션은 다음과 같습니다:

- `@OnSkipInRead`
- `@OnSkipInWrite`
- `@OnSkipInProcess`

### `SkipListener`와 트랜잭션

`SkipListener`의 가장 일반적인 사용 사례 중 하나는 스킵된 아이템을 로그로 남겨, 다른 배치 프로세스나 심지어 사람이 문제를 평가하고 수정할 수 있도록 하는 것입니다.

원래 트랜잭션이 롤백될 수 있는 경우가 많기 때문에 Spring Batch는 두 가지를 보장합니다:

1. 적절한 스킵 메소드(오류 발생 시점에 따라 다름)는 아이템당 한 번만 호출됩니다.
2. `SkipListener`는 항상 트랜잭션이 커밋되기 직전에 호출됩니다. 이는 리스너에 의해 호출되는 모든 트랜잭션 리소스가 `ItemWriter` 내의 실패로 인해 롤백되지 않도록 하기 위함입니다.

---

**학습 목표:**

이번 챕터를 통해 여러분은 다음을 이해하고 배울 수 있습니다:

1. **Spring Batch Step 실행 중 특정 이벤트 발생 시점에 개입하여 추가 로직을 수행할 수 있는 다양한 `StepListener`의 종류와 역할을 이해합니다.** (예: `StepExecutionListener`, `ChunkListener`, `ItemReadListener` 등)
2. **`StepBuilder`의 `listener()` 메소드 또는 어노테이션(@BeforeStep, @AfterChunk 등)을 사용하여 Step에 리스너를 등록하고 활용하는 방법을 설명할 수 있게 됩니다.**
3. **각 리스너 인터페이스의 주요 메소드(예: `beforeStep`, `afterChunk`, `onReadError`, `onSkipInWrite` 등)가 언제 호출되고 어떤 파라미터를 받는지 파악하여, 실제 상황에 맞는 리스너를 선택하고 구현할 수 있는 기초를 다집니다.**

---

### 핵심 개념 설명

Spring Batch에서 **리스너(Listener)** 는 `Job`이나 `Step`의 실행 생명주기 동안 발생하는 특정 이벤트에 반응하여 우리가 정의한 코드를 실행할 수 있게 해주는 강력한 도구입니다.

이번 챕터에서는 특히 `Step` 레벨에서 동작하는 다양한 리스너들을 다루고 있습니다.

**왜 리스너가 필요할까요?**

배치 작업은 단순한 데이터 처리 흐름 이상으로, 실행 전후에 준비 작업이나 마무리 작업이 필요할 때가 많습니다. 예를 들어,

- **파일 푸터(footer) 추가:** 대량의 데이터를 파일로 쓰고 나서, 파일의 마지막에 총 레코드 수나 요약 정보를 담은 푸터를 추가해야 할 수 있습니다. `Step` 실행이 완전히 끝난 시점을 알아야 이 작업을 할 수 있겠죠?
- **자원 해제:** `Step` 실행 중에 사용했던 데이터베이스 커넥션이나 파일 핸들 같은 자원들을 `Step` 종료 시점에 안전하게 해제해야 합니다.
- **오류 로깅 및 알림:** 데이터 처리 중 특정 아이템에서 오류가 발생했을 때, 해당 아이템 정보와 오류 내용을 로그로 남기거나 관리자에게 알림을 보내야 할 수 있습니다.
- **처리 통계 집계:** `Step`이 시작될 때, 각 청크(chunk)가 처리될 때, 또는 `Step`이 종료될 때 처리된 아이템 수, 성공/실패 건수 등을 집계하여 모니터링하거나 리포팅할 수 있습니다.
- **조건부 로직 수행:** `Step` 실행 결과에 따라 다음 `Step`의 실행 여부를 결정하거나, 다른 작업을 트리거해야 할 수도 있습니다.

이처럼 `Step` 실행의 특정 "순간"에 개입해서 원하는 동작을 추가하고 싶을 때 리스너가 아주 유용하게 사용됩니다.

**다양한 리스너, 다양한 역할**

Spring Batch는 `Step` 실행의 여러 지점에서 사용할 수 있도록 다양한 종류의 리스너 인터페이스를 제공합니다. 각 리스너는 특정 이벤트에 특화된 메소드를 가지고 있어서, 개발자는 필요한 리스너를 선택하고 해당 메소드를 구현하기만 하면 됩니다.

- **`StepExecutionListener`**: `Step` 전체의 시작과 끝을 감지합니다. 마치 마라톤 경기의 시작 총성과 결승선 통과 순간을 알려주는 심판 같아요.
- **`ChunkListener`**: 청크(일정량의 데이터 묶음) 처리의 시작과 끝, 그리고 오류 발생 시점을 감지합니다. 컨베이어 벨트에서 한 박스의 제품 포장이 시작될 때와 끝날 때, 그리고 불량품이 발견되었을 때 알려주는 센서와 비슷합니다.
- **`ItemReadListener`**: 아이템 하나하나를 읽기 직전, 읽은 직후, 읽기 오류 발생 시점을 감지합니다. 도서관에서 책을 한 권씩 대출하기 전, 대출한 후, 또는 책을 찾지 못했을 때 알려주는 사서와 같아요.
- **`ItemProcessListener`**: 아이템 하나하나를 처리하기 직전, 처리한 직후, 처리 오류 발생 시점을 감지합니다. 공장에서 부품을 가공하기 전, 가공한 후, 또는 가공 중 불량이 발생했을 때 알려주는 검수원과 비슷합니다.
- **`ItemWriteListener`**: 아이템 묶음(청크)을 쓰기 직전, 쓴 직후, 쓰기 오류 발생 시점을 감지합니다. 우체국에서 한 묶음의 편지를 발송하기 전, 발송한 후, 또는 발송 중 문제가 생겼을 때 알려주는 직원과 같습니다.
- **`SkipListener`**: 아이템이 읽기, 처리, 또는 쓰기 과정에서 "스킵(skip)"되었을 때를 감지합니다. 다른 리스너의 오류 알림과 달리, 실제로 해당 아이템이 건너뛰어졌을 때만 호출됩니다. 예를 들어, 특정 조건에 맞지 않아 건너뛰기로 결정된 주문 건을 알려주는 시스템과 같습니다.

**리스너 등록 방법: Java 설정과 어노테이션**

리스너를 사용하려면 `Step`에 등록해야 합니다. Spring Batch는 두 가지 주요 방법을 제공합니다:

1. **Java 기반 설정:** `StepBuilder`를 사용하여 `Step`을 정의할 때 `.listener()` 메소드를 통해 리스너 구현체를 직접 등록합니다.
2. **어노테이션:** 리스너 인터페이스를 직접 구현하는 클래스를 만들거나, 일반 클래스(POJO)의 메소드에 `@BeforeStep`, `@AfterChunk` 같은 어노테이션을 붙여서 리스너처럼 동작하게 할 수 있습니다. 이렇게 어노테이션이 붙은 객체를 `Step`에 리스너로 등록하면 Spring Batch가 알아서 해당 메소드를 적절한 시점에 호출해줍니다.

---

### 주요 용어 해설

- **리스너 (Listener):** 특정 이벤트 발생을 "듣고(listen)" 있다가, 해당 이벤트가 발생하면 미리 정의된 동작을 수행하는 객체입니다. Spring Batch에서는 `Job`이나 `Step`의 생명주기 이벤트를 감지합니다.
- **`StepListener`:** 모든 `Step` 관련 리스너 인터페이스의 부모 인터페이스입니다. 실제로는 비어있는 마커(marker) 인터페이스이며, `StepExecutionListener`, `ChunkListener` 등이 이를 상속받습니다.
- **`StepExecution`:** `Step`의 한 번 실행에 대한 정보를 담고 있는 객체입니다. `Step` 이름, 시작/종료 시간, 상태(시작됨, 완료됨, 실패함 등), `ExecutionContext` 등을 포함합니다. `StepExecutionListener`의 메소드에 파라미터로 전달되어 현재 `Step` 실행 상태를 파악하고 조작하는 데 사용됩니다.
- **`ExitStatus`:** `Step` 또는 `Job`의 실행 결과를 나타내는 문자열입니다. 기본적으로 `COMPLETED`, `FAILED` 등이 있지만, 개발자가 커스텀 `ExitStatus`를 정의하여 보다 세밀한 흐름 제어를 할 수도 있습니다. `StepExecutionListener`의 `afterStep` 메소드는 `ExitStatus`를 반환하여 `Step`의 최종 종료 상태를 변경할 수 있습니다.
- **청크 (Chunk):** 아이템 기반 처리(`ItemReader` -> `ItemProcessor` -> `ItemWriter`)에서 한 번의 트랜잭션으로 처리되는 아이템의 묶음을 의미합니다. 예를 들어, `chunk(10)`으로 설정하면 10개의 아이템을 읽고, 처리하고, 한꺼번에 쓰는 작업이 하나의 트랜잭션으로 묶입니다.
- **`ChunkContext`:** 현재 청크 처리와 관련된 정보를 담고 있는 객체입니다. `ChunkListener`의 메소드에 파라미터로 전달됩니다. `StepContext`에 접근할 수 있어 `StepExecution` 등의 정보를 얻을 수 있습니다.
- **네임스페이스 `<step>` 요소:** XML 기반 설정에서 `Step`을 정의할 때 사용하는 태그입니다. 여기에 리스너를 등록할 수 있습니다.
- **`StepFactoryBean`:** Java 기반 설정 이전 또는 XML 설정에서 `Step` 객체를 생성하는 데 사용되던 팩토리 빈 클래스들입니다. (최근에는 `StepBuilder`를 더 많이 사용합니다.)
- **POJO (Plain Old Java Object):** 특정 프레임워크나 기술에 종속되지 않은 순수 자바 객체를 의미합니다. Spring Batch에서는 리스너 인터페이스를 직접 구현하지 않고도, POJO 클래스의 메소드에 어노테이션을 붙여 리스너로 활용할 수 있는 유연성을 제공합니다.
- **스킵 로직 (Skip Logic):** 배치 처리 중 특정 아이템에서 오류가 발생했을 때, 해당 아이템 처리를 건너뛰고 다음 아이템으로 계속 진행할 수 있도록 하는 기능입니다. `SkipListener`는 이 스킵 로직과 밀접하게 연관됩니다.
- **트랜잭션 리소스 (Transactional Resources):** 데이터베이스 커넥션처럼 트랜잭션의 영향을 받는 자원들을 의미합니다. `SkipListener`가 트랜잭션 커밋 직전에 호출되는 이유는, 리스너가 사용하는 이러한 트랜잭션 리소스들이 `ItemWriter`의 실패로 인해 롤백되지 않도록 보장하기 위함입니다.

---

### 코드 예제 및 분석

**1. Java Configuration에서 ChunkListener 등록 예제**

```java
@Bean
public Step step1(JobRepository jobRepository, PlatformTransactionManager transactionManager) {
	return new StepBuilder("step1", jobRepository) // (1)
				.<String, String>chunk(10, transactionManager) // (2)
				.reader(reader()) // (3)
				.writer(writer()) // (4)
				.listener(chunkListener()) // (5)
				.build(); // (6)
}
```

- **(1) `new StepBuilder("step1", jobRepository)`**: `"step1"`이라는 이름으로 새로운 `Step`을 생성하기 시작합니다. `JobRepository`는 `Step`의 실행 상태 등을 저장하는 데 필요합니다.
- **(2) `.<String, String>chunk(10, transactionManager)`**: 이 `Step`이 청크 지향 `Step`임을 정의합니다.
  - `<String, String>`: `ItemReader`가 읽어오는 아이템의 타입과 `ItemWriter`가 쓰는 아이템의 타입이 모두 `String`임을 나타냅니다 (실제로는 `ItemProcessor`의 출력 타입).
  - `10`: 커밋 간격(commit interval)을 10으로 설정합니다. 즉, 10개의 아이템이 처리될 때마다 하나의 청크로 묶여 트랜잭션이 커밋됩니다.
  - `transactionManager`: 이 청크 처리에 사용될 트랜잭션 매니저를 지정합니다.
- **(3) `.reader(reader())`**: 이 `Step`에서 사용할 `ItemReader`를 설정합니다. `reader()` 메소드는 `ItemReader` 빈(bean)을 반환해야 합니다.
- **(4) `.writer(writer())`**: 이 `Step`에서 사용할 `ItemWriter`를 설정합니다. `writer()` 메소드는 `ItemWriter` 빈을 반환해야 합니다. (필요하다면 `.processor(processor())`도 추가될 수 있습니다.)
- **(5) `.listener(chunkListener())`**: 이 `Step`의 청크 처리에 `ChunkListener`를 등록합니다. `chunkListener()` 메소드는 `ChunkListener` 인터페이스를 구현한 빈을 반환해야 합니다. 이 리스너는 각 청크가 시작되기 전(`beforeChunk`), 성공적으로 완료된 후(`afterChunk`), 또는 오류 발생 시(`afterChunkError`)에 호출됩니다.
- **(6) `.build()`**: `Step` 설정을 완료하고 `Step` 객체를 생성합니다.

**2. `StepExecutionListener` 인터페이스**

```java
public interface StepExecutionListener extends StepListener {

    void beforeStep(StepExecution stepExecution);

    ExitStatus afterStep(StepExecution stepExecution);

}
```

- **`beforeStep(StepExecution stepExecution)`**: `Step`이 실행되기 *직전에* 호출됩니다.
  - `stepExecution`: 현재 실행될 `Step`에 대한 정보를 담고 있습니다. 예를 들어, `stepExecution.getExecutionContext().put("startTime", System.currentTimeMillis());` 와 같이 `Step` 실행 컨텍스트에 정보를 저장하여 `afterStep`에서 사용할 수 있습니다.
- **`afterStep(StepExecution stepExecution)`**: `Step`이 실행된 *직후에* (성공이든 실패든) 호출됩니다.
  - `stepExecution`: 현재 실행된 `Step`에 대한 정보를 담고 있습니다. 성공 여부(`stepExecution.getStatus()`), 처리 건수 등을 여기서 확인할 수 있습니다.
  - **`ExitStatus` 반환 타입**: 이 메소드에서 반환하는 `ExitStatus`는 해당 `Step`의 최종 종료 상태가 됩니다. 만약 `null`을 반환하면 `Step`의 실제 실행 결과에 따른 `ExitStatus`(예: `COMPLETED`, `FAILED`)가 사용됩니다. 특정 조건에 따라 `Step`의 종료 상태를 프로그래밍 방식으로 변경하고 싶을 때 유용합니다. 예를 들어, 특정 오류가 발생했지만 `Step` 자체는 `COMPLETED`로 표시하고 싶을 때 `new ExitStatus("COMPLETED_WITH_WARNINGS")`와 같이 반환할 수 있습니다.

**3. `ChunkListener` 인터페이스**

```java
public interface ChunkListener extends StepListener {

    void beforeChunk(ChunkContext context);
    void afterChunk(ChunkContext context);
    void afterChunkError(ChunkContext context);

}
```

- **`beforeChunk(ChunkContext context)`**: 청크 트랜잭션이 시작된 후, `ItemReader`가 첫 번째 아이템을 읽기 *전에* 호출됩니다. 이 시점에서 현재 청크에 대한 준비 작업을 할 수 있습니다.
- **`afterChunk(ChunkContext context)`**: 청크가 성공적으로 처리되고 트랜잭션이 커밋된 *후에* 호출됩니다. 만약 청크 처리 중 오류가 발생하여 롤백되면 이 메소드는 호출되지 않습니다.
- **`afterChunkError(ChunkContext context)`**: 청크 처리 중 예외가 발생하여 트랜잭션이 롤백될 때 호출됩니다. `context.getAttribute(ChunkListener.ROLLBACK_EXCEPTION_KEY)`를 통해 발생한 예외 객체를 가져올 수 있습니다. 오류 로깅 등에 활용됩니다.

**간단한 `StepExecutionListener` 구현 예제 (POJO + 어노테이션)**

```java
// MyStepListener.java
public class MyStepListener {

    private static final Logger log = LoggerFactory.getLogger(MyStepListener.class);

    @BeforeStep
    public void initialize(StepExecution stepExecution) {
        log.info("===== {} 시작합니다! =====", stepExecution.getStepName());
        stepExecution.getExecutionContext().put("customData", "Step 시작 시 저장된 데이터");
    }

    @AfterStep
    public ExitStatus finalizeStep(StepExecution stepExecution) {
        log.info("===== {} 종료합니다! 상태: {} =====", stepExecution.getStepName(), stepExecution.getStatus());
        log.info("Step 실행 컨텍스트에서 가져온 데이터: {}", stepExecution.getExecutionContext().getString("customData"));

        if (stepExecution.getStatus() == BatchStatus.COMPLETED) {
            log.info("푸터 정보를 기록합니다...");
            // 실제 푸터 작성 로직 (예: 파일에 쓰기)
        } else if (stepExecution.getStatus() == BatchStatus.FAILED) {
            log.error("Step 실패! 원인: {}", stepExecution.getFailureExceptions());
            return ExitStatus.FAILED.addExitDescription("치명적인 오류 발생!"); // ExitStatus 변경 예시
        }
        return stepExecution.getExitStatus(); // 기본 ExitStatus 반환
    }
}

// Spring Batch 설정 (Java Config)
@Bean
public Step sampleStep(JobRepository jobRepository, PlatformTransactionManager transactionManager,
                       ItemReader<String> reader, ItemWriter<String> writer,
                       MyStepListener myStepListener) { // (1) MyStepListener 주입
    return new StepBuilder("sampleStep", jobRepository)
            .<String, String>chunk(10, transactionManager)
            .reader(reader)
            .writer(writer)
            .listener(myStepListener) // (2) 리스너로 등록
            .build();
}

@Bean
public MyStepListener myStepListener() { // (3) 리스너 빈 등록
    return new MyStepListener();
}
```

- `MyStepListener`는 어떤 인터페이스도 구현하지 않은 POJO입니다.
- `@BeforeStep` 어노테이션이 붙은 `initialize` 메소드는 `Step` 시작 전에, `@AfterStep` 어노테이션이 붙은 `finalizeStep` 메소드는 `Step` 종료 후에 호출됩니다.
- **(1)** `Step`을 정의하는 `sampleStep` 메소드에 `MyStepListener` 타입의 빈을 주입받습니다.
- **(2)** `.listener(myStepListener)`를 통해 주입받은 리스너를 `Step`에 등록합니다. Spring Batch는 이 객체 내의 어노테이션을 스캔하여 적절한 시점에 해당 메소드를 호출합니다.
- **(3)** `MyStepListener`를 스프링 빈으로 등록합니다.

---

### "왜?" 라는 질문에 대한 답변

**Q: 왜 이렇게 다양한 종류의 리스너가 필요한가요? 하나의 만능 리스너로 처리할 수는 없나요?**

A: 물론 이론적으로는 하나의 리스너 인터페이스에 모든 가능한 콜백 메소드를 다 넣어둘 수도 있었을 겁니다. 하지만 그렇게 하면 다음과 같은 문제들이 생길 수 있습니다.

1. **인터페이스 오염 (Interface Pollution):** 개발자는 자신이 관심 있는 특정 이벤트에만 반응하고 싶을 수 있습니다. 예를 들어, `Step` 시작/종료 시점에만 관심 있는데, `ItemReadListener`의 모든 메소드까지 구현해야 한다면 매우 번거롭고 불필요한 코드가 많아집니다. 각 리스너는 특정 관심사에만 집중하도록 설계되어 인터페이스가 깔끔하게 유지됩니다. (관심사의 분리, Separation of Concerns 원칙)
2. **가독성 및 유지보수성 저하:** 하나의 리스너에 너무 많은 기능이 섞여 있으면 코드를 이해하고 유지보수하기 어려워집니다. "이 리스너는 `Step`의 시작과 끝을 처리하고, 동시에 아이템 읽기 오류도 처리하며, 청크 완료 시점에도 뭔가 한다" 와 같이 되면 복잡도가 증가합니다.
3. **유연성 및 재사용성 감소:** 특정 기능(예: 아이템 처리 오류 로깅)만 재사용하고 싶을 때, 만능 리스너를 사용하면 불필요한 다른 기능들까지 딸려오게 됩니다. 각 리스너가 명확한 역할을 가지면 필요한 리스너만 조합하여 유연하게 사용할 수 있습니다.

따라서 Spring Batch는 `Step` 생명주기의 각기 다른 단계와 관심사에 맞춰 세분화된 리스너 인터페이스를 제공함으로써, 개발자가 **필요한 기능만 선택적으로 구현**하고 코드를 더 **명확하고 유지보수하기 쉽게** 만들도록 돕는 것입니다. 마치 공구함에 망치, 드라이버, 펜치 등 다양한 공구가 각자의 역할에 맞게 준비되어 있는 것과 같습니다. 모든 작업을 하나의 "만능 공구"로 하려고 하면 비효율적이고 불편하겠죠.

**Q: `ItemStream`을 등록하는 것과 리스너를 등록하는 것은 어떤 차이가 있나요?**

A: 좋은 질문입니다! `ItemStream`과 리스너는 `Step` 실행 중에 특정 시점에 개입한다는 점은 유사하지만, 그 목적과 사용 방식에 차이가 있습니다.

- **`ItemStream`**: 주로 `ItemReader`, `ItemProcessor`, `ItemWriter`와 같이 상태를 가지는 컴포넌트가 자신의 **상태를 저장하고 복원**하기 위해 사용됩니다. 예를 들어, 파일에서 데이터를 읽는 `ItemReader`는 `Step`이 중단되었다가 재시작될 때 이전에 어디까지 읽었는지 알아야 합니다. 이때 `ItemStream`의 `open()`, `update()`, `close()` 메소드를 구현하여 `ExecutionContext`에 현재 상태를 저장하고, 재시작 시 `open()`에서 복원합니다. **주요 목적은 상태 관리와 재시작 기능 지원입니다.**
- **리스너 (Listener)**: `Step` 실행 생명주기 중 발생하는 **다양한 이벤트에 대한 응답**으로 추가적인 로직(로깅, 알림, 자원 정리, 푸터 쓰기 등)을 수행하기 위해 사용됩니다. `ItemStream`처럼 상태 저장/복원에 직접 관여하기보다는, 실행 흐름의 특정 지점에서 "뭔가를 하기 위해" 사용됩니다.

비유하자면,

- `ItemStream`은 장거리 여행 중 중간에 쉬었다 갈 때, "어디까지 왔었지?" 하고 **현재 위치를 기록해두는 여행 일지**와 같습니다. 다음에 다시 출발할 때 그 지점부터 이어갈 수 있도록요.
- 리스너는 여행 중에 "사진 찍기 좋은 명소에 도착했으니 사진을 찍자!", "점심시간이니 밥을 먹자!", "목적지에 도착했으니 기념품을 사자!" 와 같이 **특정 이벤트가 발생했을 때 수행하는 행동**과 같습니다.

물론, `ItemReader`나 `ItemWriter`가 `ItemStream` 인터페이스를 구현하면서 동시에 `StepExecutionListener`와 같은 리스너 인터페이스를 구현할 수도 있습니다. 이 경우, 해당 컴포넌트는 상태 관리 기능과 함께 `Step` 생명주기 이벤트에 대한 로직도 수행하게 됩니다.

---

### 주의사항 및 Best Practice

1. **리스너는 가볍게 유지하세요:** 리스너 내의 로직이 너무 무거워지면 전체 배치 성능에 영향을 줄 수 있습니다. 복잡한 작업은 별도의 서비스나 컴포넌트로 분리하고, 리스너는 해당 컴포넌트를 호출하는 정도로 유지하는 것이 좋습니다.
2. **적절한 리스너 선택:** `Step` 전체에 대한 작업은 `StepExecutionListener`, 청크 단위 작업은 `ChunkListener`, 개별 아이템 관련 작업은 `ItemRead/Process/WriteListener`를 사용하는 등, 수행하려는 작업의 범위와 시점에 맞는 리스너를 선택하세요.
3. **어노테이션 활용의 장점:** POJO에 어노테이션을 사용하면 코드가 간결해지고 특정 인터페이스에 대한 직접적인 의존성을 줄일 수 있습니다. 테스트도 용이해집니다.
4. **리스너 등록 위치:** 문서에서도 언급되었듯이, 리스너는 해당 기능이 적용되는 가장 적절한 수준에 선언하는 것이 좋습니다. `Step` 전체에 영향을 미치는 리스너는 `Step` 레벨에, 청크 관련 리스너는 `chunk` 레벨에 등록하는 것이 일반적입니다.
5. **`ChunkListener`의 예외 처리:** `ChunkListener`의 메소드들은 체크 예외(checked exception)를 던지도록 설계되지 않았습니다. 만약 리스너 내에서 예외가 발생하여 `Step`을 중단시키고 싶다면 `RuntimeException`을 던지거나, 예외를 잡아서 적절히 처리해야 합니다. 그렇지 않으면 예기치 않게 `Step`이 종료될 수 있습니다.
6. **`SkipListener`와 트랜잭션:** `SkipListener`는 트랜잭션이 커밋되기 *직전에* 호출됩니다. 이는 `SkipListener` 내에서 수행하는 작업(예: DB에 스킵 정보 로깅)이 주 트랜잭션의 롤백에 영향을 받지 않도록 하기 위함입니다. 만약 `SkipListener`의 작업이 주 트랜잭션에 포함되어야 한다면, 다른 방법을 고려해야 합니다. (이 경우는 일반적이지 않습니다)
7. **자동 등록 vs. 명시적 등록:**
  - `ItemReader`, `ItemWriter`, `ItemProcessor`가 직접 `StepListener` 인터페이스를 구현하고, XML 네임스페이스 `<step>`이나 `StepBuilder`를 통해 `Step`에 직접 주입되면 자동으로 리스너로 등록됩니다.
  - 하지만, 리스너가 다른 빈(bean) 내부에 중첩되어 있거나 (예: `ItemProcessor`가 내부적으로 `MyCustomListener`를 사용하는 경우), 또는 POJO 어노테이션 방식이 아닌 리스너 인터페이스를 구현한 별도의 빈을 사용하고 싶다면 명시적으로 `.listener()`를 통해 등록해주어야 합니다.
8. **`ExecutionContext` 활용:** `StepExecutionListener` 등에서 `StepExecution` 객체를 통해 `ExecutionContext`에 접근할 수 있습니다. `Step` 실행 중에 필요한 데이터를 임시로 저장하거나 `Step` 간에 간단한 데이터를 전달하는 데 유용하게 사용할 수 있습니다. (단, 너무 많은 데이터를 `ExecutionContext`에 저장하는 것은 피해야 합니다.)

---

### 이전 학습 내용과의 연관성

(이번이 첫 챕터이므로, 이전 학습 내용과의 직접적인 연관성을 언급하기는 어렵습니다. 하지만 앞으로 배우게 될 내용들과 어떻게 연결될지 예상해볼 수 있습니다.)

- **Job 실행 흐름 제어:** `StepExecutionListener`의 `afterStep`에서 반환하는 `ExitStatus`는 `Job` 내에서 `Step`들의 실행 순서를 제어하는 데 사용될 수 있습니다. (추후 `Job`의 흐름 제어 챕터에서 더 자세히 다룰 것입니다.)
- **오류 처리 및 재시도/스킵:** `ItemRead/Process/WriteListener` 및 `SkipListener`는 `Step`의 오류 처리 전략(retry, skip)과 밀접하게 연관됩니다. 오류 발생 시 리스너를 통해 로깅하거나 특정 조치를 취하고, Spring Batch의 재시도/스킵 기능과 함께 사용하여 견고한 배치 애플리케이션을 만들 수 있습니다.
- **청크 지향 처리:** `ChunkListener`는 청크 지향 `Step`의 핵심 동작과 직접적으로 관련됩니다. 청크의 생명주기에 맞춰 동작하므로, 청크 처리 메커니즘을 이해하는 데 도움이 됩니다.

---
