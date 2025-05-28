---
title: Spring Batch - Configuring a Step (Registering ItemStream)
description: 
author: laze
date: 2025-05-28 00:00:02 +0900
categories: [Dev, SpringBatch]
tags: [SpringBatch]
---
## Step에 ItemStream 등록하기 (Registering ItemStream with a Step)

Step은 생명주기(lifecycle)의 필요한 시점에서 `ItemStream` 콜백을 처리해야 합니다. (`ItemStream` 인터페이스에 대한 더 자세한 정보는 [ItemStream](https://docs.spring.io/spring-batch/docs/current/reference/html/index-single.html#itemStream)을 참조하세요).

이는 Step이 실패하여 재시작해야 할 경우 매우 중요합니다.

왜냐하면 `ItemStream` 인터페이스는 Step이 실행 간의 영속적인 상태(persistent state)에 필요한 정보를 얻는 곳이기 때문입니다.

만약 `ItemReader`, `ItemProcessor`, 또는 `ItemWriter` 자체가 `ItemStream` 인터페이스를 구현한다면, 이들은 자동으로 등록됩니다.

그 외 다른 스트림들은 별도로 등록해야 합니다. 이는 종종 reader나 writer에 주입된 위임 객체(delegates)와 같은 간접적인 의존성이 있는 경우에 해당합니다.

`stream` 요소를 통해 Step에 스트림을 등록할 수 있습니다.

다음 예제는 자바에서 Step에 스트림을 등록하는 방법을 보여줍니다:

```java
@Bean
public Step step1(JobRepository jobRepository, PlatformTransactionManager transactionManager) {
	return new StepBuilder("step1", jobRepository)
				.<String, String>chunk(2, transactionManager)
				.reader(itemReader()) // itemReader()는 ItemStream을 구현했다고 가정
				.writer(compositeItemWriter()) // compositeItemWriter() 자체는 ItemStream이 아닐 수 있음
				.stream(fileItemWriter1()) // fileItemWriter1()을 명시적으로 스트림으로 등록
				.stream(fileItemWriter2()) // fileItemWriter2()를 명시적으로 스트림으로 등록
				.build();
}

/**
 * Spring Batch 4 버전에서 CompositeItemWriter는 ItemStream을 구현하므로
 * 이 예제와 같은 명시적 등록은 필요하지 않지만, 예시를 위해 사용되었습니다.
 * (역자 주: 하지만 여전히 ItemStream을 직접 구현하지 않는 Composite 패턴이나 커스텀 ItemWriter를
 * 사용할 경우 이 개념은 유효합니다.)
 */
@Bean
public CompositeItemWriter<String> compositeItemWriter() { // ItemWriter의 제네릭 타입 명시 권장
	List<ItemWriter<? super String>> writers = new ArrayList<>(2); // ItemWriter의 제네릭 타입 명시 권장
	writers.add(fileItemWriter1());
	writers.add(fileItemWriter2());

	CompositeItemWriter<String> itemWriter = new CompositeItemWriter<>(); // 제네릭 타입 명시 권장

	itemWriter.setDelegates(writers);

	return itemWriter;
}

// 예시를 위한 fileItemWriter1, fileItemWriter2 빈 정의 (실제 구현 필요)
// @Bean
// public ItemStreamWriter<String> fileItemWriter1() {
//     // ... 실제 파일 쓰기 로직 및 ItemStream 구현 ...
//     return new FlatFileItemWriter<>(); // 예시
// }
//
// @Bean
// public ItemStreamWriter<String> fileItemWriter2() {
//     // ... 실제 파일 쓰기 로직 및 ItemStream 구현 ...
//     return new FlatFileItemWriter<>(); // 예시
// }
```

위 예제에서 `CompositeItemWriter`는 `ItemStream`이 아니지만, 그것의 두 위임 객체(delegates)는 `ItemStream`입니다. 따라서 프레임워크가 이들을 올바르게 처리하려면 두 위임 writer 모두 스트림으로 명시적으로 등록되어야 합니다. `ItemReader`는 Step의 직접적인 속성이므로 스트림으로 명시적으로 등록할 필요가 없습니다 (이미 `ItemStream`을 구현하고 있다면 자동으로 처리됨). 이제 Step은 재시작 가능하며(restartable), 실패 시 reader와 writer의 상태가 올바르게 유지됩니다.

---

**학습 목표:**

이번 챕터를 통해 여러분은 다음을 이해하고 배울 수 있습니다:

1. **`ItemStream` 인터페이스의 역할과 Step의 재시작 기능에 있어 그 중요성을 이해합니다.** (특히 `open`, `update`, `close` 메소드의 역할)
2. **어떤 경우에 `ItemReader`, `ItemProcessor`, `ItemWriter`를 Step에 명시적으로 스트림으로 등록해야 하는지 판단할 수 있게 됩니다.** (자동 등록 vs. 수동 등록)
3. **`StepBuilder`의 `stream()` 메소드를 사용하여 커스텀 `ItemStream` 구현체를 Step에 등록하는 방법을 설명할 수 있게 됩니다.**

---

### 핵심 개념 설명

Spring Batch에서 Step이 데이터를 처리할 때, 때로는 현재까지 처리한 상태를 기억해야 할 필요가 있습니다. 예를 들어, 긴 파일을 읽다가 중간에 오류가 나서 Step이 중단되었다고 가정해봅시다. 다음에 이 Step을 재시작할 때, 처음부터 다시 읽는 것이 아니라 이전에 실패했던 지점부터 이어서 처리하고 싶을 것입니다. 바로 이럴 때 필요한 것이 **`ItemStream` 인터페이스**입니다.

**`ItemStream`이란?**

- `ItemStream`은 Step의 생명주기 동안 상태를 저장하고 복원하는 데 사용되는 인터페이스입니다. 마치 우리가 게임을 하다가 중간에 저장(save)하고, 나중에 그 지점부터 다시 시작(load)하는 것과 비슷합니다.
- 주로 `ItemReader`나 `ItemWriter`가 이 인터페이스를 구현하여, 자신이 어디까지 읽었는지, 또는 어디까지 썼는지 등의 상태 정보를 ExecutionContext에 저장하고, 재시작 시 이 정보를 바탕으로 작업을 이어갈 수 있도록 합니다.
- `ItemStream` 인터페이스는 세 가지 주요 메소드를 가지고 있습니다:
  1. **`open(ExecutionContext executionContext)`**:
    - Step이 시작될 때 (또는 스트림이 처음 열릴 때) 호출됩니다.
    - 필요한 리소스를 초기화하거나, 이전 실행에서 저장된 상태가 있다면 `executionContext`에서 해당 상태를 불러와 복원하는 역할을 합니다.
    - **비유:** 게임을 시작할 때 저장된 파일을 불러오거나, 새 게임을 위한 초기 설정을 하는 것과 같습니다.
  2. **`update(ExecutionContext executionContext)`**:
    - Chunk가 커밋되기 직전에 주기적으로 호출됩니다. (보통 각 Chunk 처리 후)
    - 현재까지의 처리 상태 (예: 현재 읽은 라인 수, 처리한 레코드 수 등)를 `executionContext`에 저장합니다. 이렇게 저장된 정보는 Step이 실패하고 재시작될 때 `open()` 메소드에서 사용됩니다.
    - **비유:** 게임 중간중간 자동 저장(autosave)되는 것과 같습니다.
  3. **`close()`**:
    - Step이 정상적으로 또는 비정상적으로 종료될 때 호출됩니다.
    - 사용했던 리소스를 해제하는 역할을 합니다 (예: 파일 닫기, DB 커넥션 반환 등).
    - **비유:** 게임을 종료할 때 사용했던 메모리나 파일 핸들을 정리하는 것과 같습니다.

**왜 Step에 `ItemStream`을 등록해야 할까요?**

Spring Batch의 Step은 이 `ItemStream`의 `open`, `update`, `close` 메소드를 적절한 시점에 호출해주는 역할을 합니다.

- **자동 등록:** 만약 여러분이 사용하는 `ItemReader`, `ItemProcessor`, 또는 `ItemWriter`가 직접 `ItemStream` 인터페이스를 구현하고 있다면, Spring Batch는 이를 자동으로 감지하고 관리해줍니다. 별도의 등록 절차가 필요 없습니다. 예를 들어, Spring Batch에서 제공하는 `FlatFileItemReader`, `JdbcCursorItemReader` 등은 대부분 `ItemStream`을 구현하고 있습니다.
- **수동 등록 (명시적 등록):** 하지만 다음과 같은 경우에는 `ItemStream`을 Step에 명시적으로 등록해주어야 합니다:
  - `ItemReader`, `ItemWriter` 등이 `ItemStream`을 직접 구현하지 않고, 내부적으로 `ItemStream`을 구현한 다른 객체(위임 객체, delegate)를 사용하는 경우. 이 경우 Step은 그 내부 객체가 `ItemStream`이라는 것을 알지 못합니다.
  - 여러분만의 커스텀 로직을 가진, `ItemStream`을 구현한 별도의 헬퍼(helper) 객체를 Step의 생명주기에 맞춰 관리하고 싶을 때.

**예제의 `CompositeItemWriter` 상황:**

원문 예제에서는 `CompositeItemWriter`를 사용하고 있습니다. `CompositeItemWriter`는 여러 개의 `ItemWriter`를 묶어서 하나의 `ItemWriter`처럼 동작하게 만드는 유용한 클래스입니다.

- `CompositeItemWriter` 자체는 (Spring Batch 4 이전 버전에서는) `ItemStream`을 구현하지 않았을 수 있습니다.
- 하지만 이 `CompositeItemWriter`가 내부적으로 사용하는 실제 `ItemWriter`들 (예제에서는 `fileItemWriter1`, `fileItemWriter2`)은 각각 파일을 다루므로 `ItemStream`을 구현하고 있을 가능성이 높습니다.
- 이런 경우, Step은 `compositeItemWriter`만 알고 있을 뿐, 그 안에 있는 `fileItemWriter1`과 `fileItemWriter2`가 `ItemStream`이라는 사실을 모릅니다. 따라서 이들의 `open`, `update`, `close` 메소드가 호출되지 않아 상태 관리가 제대로 이루어지지 않게 됩니다.
- 이를 해결하기 위해 `StepBuilder`의 `stream()` 메소드를 사용하여 `fileItemWriter1`과 `fileItemWriter2`를 Step에 명시적으로 "이들도 ItemStream이니 관리해줘!" 라고 알려주는 것입니다.

**결론적으로, `ItemStream`을 올바르게 등록하는 것은 Step의 재시작 기능을 보장하고 데이터 처리의 안정성을 높이는 데 매우 중요합니다.**

### 주요 용어 해설

- **`ItemStream`**: Step의 생명주기 동안 입력 또는 출력 스트림의 상태를 관리하기 위한 인터페이스. `open`, `update`, `close` 메소드를 정의합니다.
- **`ExecutionContext`**: Step 또는 Job 실행 중에 프레임워크나 개발자가 상태 정보를 저장하고 공유할 수 있는 키-값 쌍의 컬렉션입니다. `ItemStream`은 이 `ExecutionContext`를 사용하여 상태를 저장하고 복원합니다.
- **콜백 (Callback)**: 특정 이벤트가 발생했을 때 시스템에 의해 호출되도록 약속된 메소드입니다. `ItemStream`의 `open`, `update`, `close`가 대표적인 콜백 메소드입니다.
- **영속적인 상태 (Persistent State)**: 애플리케이션 실행이 종료되거나 시스템이 재시작되어도 유지되는 상태 정보입니다. `ItemStream`은 `ExecutionContext`를 통해 이 상태를 디스크(보통 데이터베이스의 배치 메타데이터 테이블)에 저장합니다.
- **위임 객체 (Delegate)**: 어떤 객체가 자신이 직접 처리해야 할 작업을 다른 객체에게 맡기는 경우, 그 작업을 실제로 수행하는 다른 객체를 말합니다. 예제의 `CompositeItemWriter`는 실제 쓰기 작업을 내부 `ItemWriter`들에게 위임합니다.
- **`stream()` 메소드**: `StepBuilder`에서 `ItemStream`을 구현한 객체를 Step에 명시적으로 등록할 때 사용하는 메소드입니다.
- **`CompositeItemWriter`**: 여러 `ItemWriter`를 하나의 단위로 묶어 처리할 수 있게 해주는 `ItemWriter` 구현체입니다. 각 아이템을 모든 내부 writer들에게 전달합니다. (주: 원문에서는 Spring Batch 4부터 `CompositeItemWriter`가 `ItemStream`을 구현한다고 언급되어 있지만, 직접 `ItemStream`을 구현하지 않는 커스텀 컴포짓 패턴이나, 내부 컴포넌트가 `ItemStream`인 복합 객체를 사용할 때 이 개념은 여전히 중요합니다.)

### 코드 예제 및 분석

제공된 예제 코드를 다시 한번 자세히 살펴보겠습니다.

```java
@Bean
public Step step1(JobRepository jobRepository, PlatformTransactionManager transactionManager) {
	return new StepBuilder("step1", jobRepository) // Step 이름과 JobRepository
				.<String, String>chunk(2, transactionManager) // 청크 설정
				.reader(itemReader()) // ItemReader 설정. itemReader()가 ItemStream을 구현했다면 자동 등록됨.
				.writer(compositeItemWriter()) // ItemWriter로 compositeItemWriter 사용
				// compositeItemWriter가 직접 ItemStream을 구현하지 않았고,
				// 내부의 fileItemWriter1, fileItemWriter2가 ItemStream일 경우,
				// 이들을 명시적으로 등록해야 open, update, close가 호출됨.
				.stream(fileItemWriter1()) // fileItemWriter1을 스트림으로 등록
				.stream(fileItemWriter2()) // fileItemWriter2를 스트림으로 등록
				.build();
}

/**
 * Spring Batch 4에서 CompositeItemWriter는 ItemStream을 구현하므로 이 명시적 등록은
 * 불필요할 수 있지만, 예시로 사용됨.
 * (하지만 ItemStream을 구현하지 않는 커스텀 CompositeWriter를 사용하거나,
 *  CompositeWriter가 ItemStream을 구현했더라도, 그 내부 delegate들이
 *  추가적인 상태 관리가 필요하다면 여전히 유효할 수 있습니다.)
 */
@Bean
public CompositeItemWriter<String> compositeItemWriter() {
	List<ItemWriter<? super String>> writers = new ArrayList<>(2);
	writers.add(fileItemWriter1()); // ItemStream을 구현한 writer라고 가정
	writers.add(fileItemWriter2()); // ItemStream을 구현한 writer라고 가정

	CompositeItemWriter<String> itemWriter = new CompositeItemWriter<>();
	itemWriter.setDelegates(writers); // 내부 writer들 설정

	return itemWriter;
}

// 가정: fileItemWriter1, fileItemWriter2는 ItemStream을 구현한 ItemWriter 빈 (Bean)
// 예시:
// @Bean
// public FlatFileItemWriter<String> fileItemWriter1() {
//     FlatFileItemWriter<String> writer = new FlatFileItemWriter<>();
//     // ... writer 설정 ...
//     writer.setName("fileItemWriter1"); // ItemStream은 이름을 가지는 것이 좋음 (ExecutionContext 키로 사용될 수 있음)
//     return writer;
// }
//
// @Bean
// public FlatFileItemWriter<String> fileItemWriter2() {
//     FlatFileItemWriter<String> writer = new FlatFileItemWriter<>();
//     // ... writer 설정 ...
//     writer.setName("fileItemWriter2");
//     return writer;
// }
//
// @Bean
// public ItemReader<String> itemReader() {
//     // ... ItemReader 구현 (ItemStream을 구현했을 수 있음) ...
//     // 예: return new ListItemReader<>(Arrays.asList("a", "b", "c", "d", "e"));
//     // 또는 FlatFileItemReader 등
// }

```

**코드 분석:**

1. **`reader(itemReader())`**:
  - `itemReader()` 메소드가 반환하는 `ItemReader` 객체를 Step의 reader로 설정합니다.
  - 만약 이 `itemReader()`가 반환하는 객체가 `ItemStream` 인터페이스를 직접 구현했다면 (예: `FlatFileItemReader`), Spring Batch가 자동으로 이를 인지하고 `open`, `update`, `close` 콜백을 관리합니다. 별도의 `stream()` 등록이 필요 없습니다.
2. **`writer(compositeItemWriter())`**:
  - `compositeItemWriter()` 메소드가 반환하는 `CompositeItemWriter` 객체를 Step의 writer로 설정합니다.
  - `CompositeItemWriter`는 여러 writer에게 작업을 위임하는 역할을 합니다.
3. **`stream(fileItemWriter1())`** 및 **`stream(fileItemWriter2())`**:
  - 이 부분이 핵심입니다. `compositeItemWriter()`가 직접 `ItemStream`이 아니거나, `ItemStream`이더라도 내부 위임 객체들의 상태 관리를 프레임워크에 명시적으로 알려주고 싶을 때 사용합니다.
  - `fileItemWriter1()`과 `fileItemWriter2()`가 각각 `ItemStream`을 구현한 실제 파일 쓰기 담당 객체라고 가정합니다.
  - `.stream()` 메소드를 통해 이들을 Step에 등록하면, Step은 `compositeItemWriter`의 `write()` 메소드를 호출하는 것 외에도, `fileItemWriter1`과 `fileItemWriter2`의 `open`, `update`, `close` 메소드를 적절한 시점에 호출해줍니다.
  - **왜 이렇게 할까요?** 만약 `fileItemWriter1`이 "output1.txt"에 쓰고, `fileItemWriter2`가 "output2.txt"에 쓴다고 해봅시다. Step이 중간에 실패하면, 재시작 시 `fileItemWriter1`은 "output1.txt"의 어디까지 썼는지, `fileItemWriter2`는 "output2.txt"의 어디까지 썼는지 알아야 합니다. `.stream()`으로 등록해야 각 writer의 `update` 메소드가 호출되어 현재 상태가 `ExecutionContext`에 저장되고, 재시작 시 `open` 메소드에서 이 상태를 복원할 수 있습니다.
4. **`compositeItemWriter()` 메소드 내부:**
  - `List<ItemWriter<? super String>> writers` : 여러 `ItemWriter`를 담을 리스트입니다.
  - `writers.add(fileItemWriter1()); writers.add(fileItemWriter2());`: `fileItemWriter1`과 `fileItemWriter2`를 리스트에 추가합니다. 이들이 실제 작업을 수행하는 위임 객체들입니다.
  - `itemWriter.setDelegates(writers);`: `CompositeItemWriter`에 위임할 writer 리스트를 설정합니다.

**원문의 주석에 대한 부연 설명:**

> "In Spring Batch 4, the CompositeItemWriter implements ItemStream so this isn't necessary, but used for an example."
>
- 이 말은 Spring Batch 4 버전부터는 `CompositeItemWriter` 자체가 `ItemStream`을 구현하기 때문에, `CompositeItemWriter`가 가진 위임 객체(delegates)들이 `ItemStream`이라면 `CompositeItemWriter`가 알아서 그들의 `open`, `update`, `close`를 호출해 줄 수 있다는 의미입니다.
- **하지만** 만약 여러분이 `CompositeItemWriter`를 사용하지 않거나, 직접 만든 커스텀 컴포짓 객체가 `ItemStream`을 제대로 위임 처리하지 않거나, 또는 `ItemStream`을 구현했지만 내부적으로 또 다른 복잡한 상태 관리가 필요한 `ItemStream` 컴포넌트를 가지고 있다면, 여전히 `.stream()`을 통한 명시적 등록이 필요할 수 있습니다.
- 또한, `ItemProcessor`의 경우에도 `ItemStream`을 구현했다면, `.processor()`로 등록될 때 자동으로 스트림 관리가 되지만, 만약 `ItemProcessor`가 내부적으로 `ItemStream`을 사용하는 헬퍼 객체를 가지고 있다면 그 헬퍼 객체는 `.stream()`으로 등록해줘야 합니다.

### "왜?" 라는 질문에 대한 답변

**Q: 왜 `ItemStream`이라는 복잡해 보이는 인터페이스와 등록 절차가 필요할까요? 그냥 reader나 writer가 알아서 상태 관리하면 안 되나요?**

A: 좋은 질문입니다! 몇 가지 이유가 있습니다:

1. **표준화된 라이프사이클 관리:** Spring Batch는 다양한 종류의 `ItemReader`, `ItemProcessor`, `ItemWriter`를 지원합니다 (파일, DB, 메시지 큐 등). 각기 다른 방식으로 상태를 관리한다면 프레임워크가 이를 일관되게 제어하기 어렵습니다. `ItemStream` 인터페이스는 `open`, `update`, `close`라는 표준화된 메소드를 제공함으로써, Step이 어떤 종류의 스트림이든 동일한 방식으로 생명주기를 관리하고 상태를 유지/복원할 수 있도록 합니다.
2. **재시작 기능의 핵심:** 배치 작업은 대용량 데이터를 다루는 경우가 많아 중간에 실패할 가능성이 항상 존재합니다. 처음부터 다시 시작하는 것은 매우 비효율적입니다. `ItemStream`과 `ExecutionContext`를 통한 상태 저장은 재시작 시 실패 지점부터 작업을 이어갈 수 있게 하는 핵심 메커니즘입니다. 이것이 없다면 "exactly once" 처리를 보장하기 어렵습니다.
3. **관심사의 분리 (Separation of Concerns):** `ItemReader`나 `ItemWriter`의 주된 책임은 데이터를 읽고 쓰는 것입니다. 상태 관리 로직은 부가적인 관심사일 수 있습니다. `ItemStream` 인터페이스를 통해 이 상태 관리 로직을 명확히 분리하고, Step이 이를 관장하도록 함으로써 코드의 모듈성과 테스트 용이성을 높일 수 있습니다.
4. **유연성 및 확장성:** 개발자는 자신만의 커스텀 `ItemStream` 구현체를 만들어서 Step의 생명주기에 참여시킬 수 있습니다. 예를 들어, 특정 리소스를 Step 시작 시 초기화하고 종료 시 해제해야 하는 경우, `ItemStream`을 구현한 헬퍼 빈을 만들어 `.stream()`으로 등록하면 편리하게 관리할 수 있습니다.

결국, `ItemStream`과 그 등록 과정은 Spring Batch가 **견고하고 신뢰할 수 있는(robust and reliable)** 배치 애플리케이션을 구축할 수 있도록 지원하는 중요한 기능 중 하나입니다.

### 주의사항 및 Best Practice

- **`ItemStream` 구현 시 `setName()` 호출:** `ItemStream`을 구현하는 컴포넌트(특히 Spring Batch 제공 클래스 사용 시)에는 `setName()` 메소드를 호출하여 고유한 이름을 지정하는 것이 좋습니다. 이 이름은 `ExecutionContext`에 상태를 저장할 때 키(key)의 일부로 사용되어, 여러 `ItemStream` 인스턴스가 있을 때 상태가 겹치거나 유실되는 것을 방지합니다.

    ```java
    @Bean
    public FlatFileItemReader<String> myItemReader() {
        FlatFileItemReader<String> reader = new FlatFileItemReader<>();
        // ... other configurations ...
        reader.setName("myItemReader"); // 고유한 이름 설정
        return reader;
    }
    
    ```

- **`ExecutionContext` 키 충돌 주의:** `update()` 메소드에서 `ExecutionContext`에 값을 저장할 때 사용하는 키가 다른 `ItemStream`이나 리스너 등에서 사용하는 키와 충돌하지 않도록 주의해야 합니다. 일반적으로 `ItemStream`의 이름과 관련된 접두사를 사용하여 키를 만드는 것이 좋습니다. Spring Batch 제공 `ItemStream` 구현체들은 내부적으로 이를 잘 처리합니다.
- **불필요한 명시적 등록은 피하세요:** `ItemReader`, `ItemProcessor`, `ItemWriter`가 이미 `ItemStream`을 구현하고 있다면 대부분 자동으로 등록됩니다. 꼭 필요한 경우에만 `.stream()`을 사용하여 명시적으로 등록하세요.
- **`CompositeItemWriter` 사용 시 버전 확인:** 만약 `CompositeItemWriter`를 사용하고 있고, 그 위임 객체들이 `ItemStream`이라면, 사용 중인 Spring Batch 버전에 따라 `CompositeItemWriter` 자체가 `ItemStream`을 적절히 처리하는지 확인하세요. 문서나 소스 코드를 통해 확인하는 것이 가장 정확합니다. (하지만 예제에서 보여주듯이, 학습 목적으로 또는 구버전과의 호환성을 위해 명시적으로 등록하는 방법을 알아두는 것은 유용합니다.)
- **상태 정보는 최소한으로:** `ExecutionContext`에 저장하는 상태 정보는 꼭 필요한 최소한의 정보만이어야 합니다. 너무 많은 데이터를 저장하면 성능에 영향을 줄 수 있고, 직렬화/역직렬화 오버헤드가 발생할 수 있습니다.

### 이전 학습 내용과의 연관성

- **Step의 생명주기:** `ItemStream`의 `open`, `update`, `close` 메소드는 Step의 특정 생명주기 시점(시작, 청크 커밋 전, 종료)에 호출됩니다.
- **Chunk-Oriented Processing:** `ItemStream`의 `update` 메소드는 주로 각 청크가 성공적으로 처리되고 트랜잭션이 커밋되기 직전에 호출되어 현재까지의 진행 상황을 저장합니다.
- **재시작 (Restartability):** `ItemStream`은 Step의 재시작 기능을 구현하는 데 핵심적인 역할을 합니다. 실패한 Step이 재시작될 때 `ExecutionContext`에 저장된 상태를 `open` 메소드에서 읽어와 이전 작업 지점부터 다시 시작할 수 있도록 합니다.

---

**그렇다면 왜 reader, processor, writer 외에 다른 ItemStream이 필요할까요?**

Step의 주된 데이터 처리 흐름(읽기-처리-쓰기) 외에도, Step의 생명주기(시작, 각 청크 처리 후, 종료)에 맞춰 특정 부가적인 작업을 수행하고, 그 작업의 상태를 관리해야 하는 경우가 있습니다. 이럴 때 독립적인 ItemStream (여기서는 helperStream으로 명명)이 유용하게 사용될 수 있습니다.

몇 가지 구체적인 시나리오를 들어보겠습니다:

1. **외부 리소스 관리:**
  - **시나리오:** Step이 시작될 때 특정 외부 시스템에 연결을 맺고, Step이 종료될 때 연결을 해제해야 합니다. 또한, 중간중간 연결 상태를 점검하거나 특정 정보를 해당 시스템에 주기적으로 업데이트해야 할 수도 있습니다.
  - **ItemStream 활용:**
    - open(): 외부 시스템 연결 초기화.
    - update(): 연결 상태 점검, 주기적인 정보 업데이트 (예: "현재까지 N건 처리 완료" 상태 전송).
    - close(): 외부 시스템 연결 해제.
  - 이런 연결 관리 로직은 ItemReader나 ItemWriter의 주된 책임과는 거리가 멀 수 있습니다. 별도의 ConnectionManagerStream 같은 클래스를 만들어 ItemStream을 구현하고 .stream()으로 등록하면, Step의 생명주기에 맞춰 깔끔하게 리소스 관리를 할 수 있습니다.
2. **커스텀 통계 및 감사 로깅:**
  - **시나리오:** Step 실행 중에 특정 비즈니스 규칙에 따라 아이템들을 분류하고, 각 분류별 아이템 개수를 집계하여 Step 종료 시 로그로 남기거나 별도의 통계 테이블에 저장하고 싶습니다. 이 집계 정보는 재시작 시에도 유지되어야 합니다.
  - **ItemStream 활용:**
    - CustomStatisticsStream을 만들고 ItemStream을 구현합니다.
    - open(): ExecutionContext에서 이전 집계 정보를 불러옵니다.
    - update(): 현재 청크까지의 집계 정보를 ExecutionContext에 저장합니다. (실제 집계 로직은 ItemProcessor나 ItemWriteListener 등에서 이 스트림의 메소드를 호출하여 수행할 수 있습니다.)
    - close(): 최종 집계 정보를 로그로 남기거나 DB에 저장합니다.
  - 이렇게 하면 통계 로직이 주된 데이터 처리 로직과 분리되어 코드가 더 명확해집니다.
3. **임시 파일 또는 디렉토리 관리:**
  - **시나리오:** Step 실행 중에 임시 파일을 생성하여 중간 데이터를 저장했다가, Step 종료 시 이 임시 파일을 삭제해야 합니다. 만약 Step이 중간에 실패하면, 재시작 시 이전 임시 파일의 내용을 참고해야 할 수도 있습니다.
  - **ItemStream 활용:**
    - TempFileManagerStream을 만들고 ItemStream을 구현합니다.
    - open(): 필요한 임시 디렉토리를 생성하거나, ExecutionContext에서 이전 임시 파일 정보를 가져옵니다.
    - update(): 현재 사용 중인 임시 파일의 상태나 경로를 ExecutionContext에 저장합니다.
    - close(): 임시 파일을 정리(삭제)합니다.
4. **복잡한 상태를 가진 헬퍼 객체의 생명주기 관리:**
  - **시나리오:** ItemProcessor나 ItemWriter가 내부적으로 복잡한 상태를 가진 헬퍼 객체를 사용하는데, 이 헬퍼 객체 자체의 초기화, 상태 업데이트, 자원 해제가 Step의 생명주기와 동기화되어야 합니다. ItemProcessor나 ItemWriter가 직접 ItemStream을 구현하기에는 너무 많은 책임을 가지게 될 때 유용합니다.
  - **ItemStream 활용:** 이 헬퍼 객체를 ItemStream으로 만들고 .stream()으로 등록하여 Step이 생명주기를 관리하도록 위임합니다.
