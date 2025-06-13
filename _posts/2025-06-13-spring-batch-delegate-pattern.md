---
title: Spring Batch - Delegate Pattern and Registering with the Step
description: 
author: laze
date: 2025-06-13 00:00:02 +0900
categories: [Dev, SpringBatch]
tags: [SpringBatch]
---
### The Delegate Pattern and Registering with the Step

`CompositeItemWriter`는 Spring Batch에서 흔히 볼 수 있는 델리게이트 패턴의 한 예입니다.

델리게이트 자체는 `StepListener`와 같은 콜백 인터페이스를 구현할 수 있습니다.

만약 그렇고, 잡(Job) 내의 스텝(Step)의 일부로서 Spring Batch Core와 함께 사용된다면, 거의 확실하게 스텝(Step)에 수동으로 등록되어야 합니다.

스텝(Step)에 직접 연결된 리더(reader), 라이터(writer), 또는 프로세서(processor)는 `ItemStream` 또는 `StepListener` 인터페이스를 구현하는 경우 자동으로 등록됩니다.

그러나 델리게이트는 스텝(Step)에 알려져 있지 않으므로, 리스너(listeners) 또는 스트림(streams)으로 (또는 적절하다면 둘 다로) 주입되어야 합니다.

다음 예제는 Java에서 델리게이트를 스트림으로 주입하는 방법을 보여줍니다:

```java
@Bean
public Job ioSampleJob(JobRepository jobRepository, Step step1) {
	return new JobBuilder("ioSampleJob", jobRepository)
				.start(step1)
				.build();
}

@Bean
public Step step1(JobRepository jobRepository, PlatformTransactionManager transactionManager,
                  ItemReader<String> reader, // fooReader() 대신 직접 주입받는 형태로 변경 (예시 일관성)
                  ItemProcessor<String, String> processor, // fooProcessor() 대신 직접 주입받는 형태로 변경
                  CustomCompositeItemWriter compositeItemWriter, // compositeItemWriter() 빈 주입
                  BarWriter barWriterForStream) { // barWriter() 빈 주입 (스트림 등록용)
	return new StepBuilder("step1", jobRepository)
				.<String, String>chunk(2, transactionManager)
				.reader(reader) // reader 직접 사용
				.processor(processor) // processor 직접 사용
				.writer(compositeItemWriter) // CompositeItemWriter를 writer로 설정
				.stream(barWriterForStream) // barWriter를 스트림으로 명시적 등록
				.build();
}

// CustomCompositeItemWriter는 BarWriter에게 실제 쓰기 작업을 위임한다고 가정
@Bean
public CustomCompositeItemWriter compositeItemWriter(BarWriter barWriterDelegate) { // barWriter() 빈을 주입받아 delegate로 설정
	CustomCompositeItemWriter writer = new CustomCompositeItemWriter();
	writer.setDelegate(barWriterDelegate); // barWriter를 delegate로 설정
	return writer;
}

// 실제 쓰기 작업을 수행하는 BarWriter (ItemStream을 구현한다고 가정)
@Bean
public BarWriter barWriter() {
	return new BarWriter();
}

// 예시를 위한 ItemReader, ItemProcessor (문서에는 fooReader, fooProcessor로 되어 있으나,
// 일관성을 위해 주입받는 형태로 가정하여 추가)
@Bean
public ItemReader<String> fooReader() {
    // 실제 ItemReader 구현 반환
    return new ListItemReader<>(Arrays.asList("a", "b", "c", "d", "e")); // 간단한 예시
}

@Bean
public ItemProcessor<String, String> fooProcessor() {
    // 실제 ItemProcessor 구현 반환
    return item -> item.toUpperCase(); // 간단한 예시
}
```

---

### **학습 목표 제시**

이번 "The Delegate Pattern and Registering with the Step" 챕터를 통해 학생분은 다음을 이해하고 배울 수 있습니다:

1. **델리게이트 패턴(Delegate Pattern)의 개념 이해:** Spring Batch에서 특정 작업을 다른 객체(델리게이트)에게 위임하는 델리게이트 패턴이 어떻게 사용될 수 있는지 이해합니다. (예: `CompositeItemWriter`)
2. **델리게이트 객체의 생명주기 관리 필요성 인지:** 델리게이트 객체가 `ItemStream`이나 `StepListener`와 같은 생명주기 인터페이스를 구현할 경우, 스텝(Step)이 이를 인지하고 관리할 수 있도록 명시적인 등록이 필요함을 이해합니다.
3. **스텝(Step)에 델리게이트를 스트림 또는 리스너로 등록하는 방법 학습:** `StepBuilder`의 `stream()` 또는 `listener()` 메서드를 사용하여, 스텝에 직접 연결되지 않은 델리게이트 객체의 생명주기 메서드(`open`, `update`, `close` 또는 리스너 메서드)가 올바르게 호출되도록 설정하는 방법을 익힙니다.

---

### **핵심 개념 설명**

이 챕터의 핵심은 **"Spring Batch 스텝(Step)에 직접 연결되지 않았지만, 스텝의 생명주기에 참여해야 하는 객체(델리게이트)들을 어떻게 스텝에 알리고 관리할 것인가?"** 입니다.

### 1. 델리게이트 패턴 (Delegate Pattern)이란?

- **개념:** 하나의 객체(주 객체)가 자신이 직접 처리해야 할 작업의 일부 또는 전부를 다른 객체(델리게이트 객체)에게 위임하는 디자인 패턴입니다. 주 객체는 인터페이스 역할을 하고, 실제 작업은 델리게이트 객체가 수행합니다.
- **비유:**
  - **사장님(주 객체)과 비서(델리게이트 객체):** 사장님은 중요한 결정만 하고, 실제 스케줄 관리, 전화 응대 등의 업무는 비서에게 위임합니다.
  - **TV 리모컨(주 객체)과 TV 내부 부품(델리게이트 객체):** 리모컨의 버튼을 누르면, 리모컨은 실제 채널 변경이나 음량 조절 명령을 TV 내부의 특정 부품에게 전달하여 수행하게 합니다.
- **Spring Batch에서의 예시 (`CompositeItemWriter`):**
  - `CompositeItemWriter`는 여러 개의 다른 `ItemWriter`들에게 실제 쓰기 작업을 위임합니다. `CompositeItemWriter` 자체는 아이템을 받아서 등록된 델리게이트 `ItemWriter`들에게 순차적으로 전달하는 역할만 합니다.
  - 예를 들어, 처리된 데이터를 파일에도 쓰고, 데이터베이스에도 동시에 쓰고 싶을 때, `CompositeItemWriter`를 사용하고, 그 안에 `FlatFileItemWriter`와 `JdbcBatchItemWriter`를 델리게이트로 등록할 수 있습니다.

### 2. 왜 델리게이트를 스텝에 등록해야 하는가?

- Spring Batch의 스텝(`Step`)은 자신의 직접적인 구성 요소들, 즉 `ItemReader`, `ItemProcessor`, `ItemWriter`로 등록된 빈(Bean)들에 대해서는 특별한 관리를 합니다.
- 만약 이들 직접 구성 요소가 `ItemStream` (리소스 관리 및 상태 저장)이나 `StepListener` (스텝 생명주기 이벤트 처리) 인터페이스를 구현하고 있다면, 스텝은 이들의 `open()`, `update()`, `close()`나 리스너 메서드들을 **자동으로 호출**해줍니다.
- **문제점:** 하지만, 델리게이트 패턴을 사용할 경우, 스텝은 주 객체(예: `CompositeItemWriter`)만 알고 있고, 그 내부의 델리게이트 객체(예: `CompositeItemWriter`가 사용하는 실제 `FlatFileItemWriter`)의 존재를 **직접적으로 알지 못합니다.**
- **결과:** 만약 델리게이트 객체가 `ItemStream`이나 `StepListener`를 구현하고 있더라도, 스텝이 이를 모르기 때문에 해당 델리게이트의 생명주기 메서드들이 **자동으로 호출되지 않습니다.** 이는 리소스가 제대로 열리거나 닫히지 않거나, 상태가 저장/복원되지 않거나, 리스너가 동작하지 않는 문제를 야기할 수 있습니다.
- **"왜 이런 문제가 생길까?"**: 스텝은 설정된 `reader`, `processor`, `writer` 객체만 보고 "아, 네가 `ItemStream`이구나! 내가 `open`, `close` 챙겨줄게!" 라고 판단합니다. 하지만 그 객체가 내부적으로 다른 `ItemStream` 구현체를 숨기고 있다면, 스텝은 그 숨겨진 객체까지는 알 길이 없는 것이죠.

### 3. 스텝에 델리게이트 등록하기

이러한 문제를 해결하기 위해, 개발자는 스텝에게 "이봐, 스텝! 네가 직접 쓰는 건 아니지만, 이 녀석도 `ItemStream`이니까 네 생명주기에 맞춰서 `open`, `update`, `close` 좀 호출해줘!" 또는 "이 녀석은 `StepListener`니까 이벤트 발생 시 이 녀석도 좀 불러줘!" 라고 **명시적으로 알려줘야 합니다.**

Spring Batch는 `StepBuilder`를 통해 이를 수행할 수 있는 메서드를 제공합니다:

- **`.stream(ItemStream stream)`:** `ItemStream`을 구현한 델리게이트 객체를 스텝의 관리 대상으로 등록합니다. 이렇게 등록된 스트림은 스텝 생명주기에 따라 `open()`, `update()`, `close()` 메서드가 호출됩니다.
- **`.listener(Object listener)`:** `StepListener` (또는 `JobListener`, `ChunkListener` 등 리스너 계열 인터페이스)를 구현한 델리게이트 객체를 리스너로 등록합니다. 해당 이벤트 발생 시 리스너 메서드가 호출됩니다. 리스너는 다양한 타입이 올 수 있으므로 파라미터 타입이 `Object`입니다.

**핵심:** 스텝에 `reader`, `writer`, `processor`로 직접 설정되지 *않았지만*, 그들의 생명주기(리소스 관리, 상태 저장, 이벤트 처리)가 스텝과 함께 관리되어야 하는 객체들이 있다면, `.stream()`이나 `.listener()`를 통해 **수동으로 등록**해주어야 합니다.

---

### **주요 용어 해설**

- **델리게이트 패턴 (Delegate Pattern):** 한 객체가 처리해야 할 작업을 다른 객체에게 위임하는 디자인 패턴.
- **델리게이트 (Delegate):** 작업을 위임받아 실제로 수행하는 객체.
- **콜백 인터페이스 (Callback Interface):** 특정 이벤트가 발생했을 때 시스템(프레임워크)에 의해 호출될 메서드를 정의하는 인터페이스. `ItemStream`, `StepListener` 등이 이에 해당합니다.
- **명시적 등록 (Manual Registration):** 프레임워크가 자동으로 인지하지 못하는 컴포넌트를 개발자가 직접 프레임워크에 알려주는 행위.

---

### **코드 예제 및 분석**

```java
@Configuration
public class MyJobConfiguration {

    @Bean
    public Job ioSampleJob(JobRepository jobRepository, Step step1) {
        return new JobBuilder("ioSampleJob", jobRepository)
                .start(step1)
                .build();
    }

    @Bean
    public Step step1(JobRepository jobRepository,
                      PlatformTransactionManager transactionManager,
                      ItemReader<String> fooReader, // 스텝에 직접 등록될 ItemReader
                      ItemProcessor<String, String> fooProcessor, // 스텝에 직접 등록될 ItemProcessor
                      CustomCompositeItemWriter compositeItemWriter, // 스텝에 직접 등록될 ItemWriter (주 객체)
                      BarWriter barWriter // compositeItemWriter의 델리게이트이자, ItemStream을 구현
                      /* 여기에 BarWriter를 barWriterForStream처럼 별도 파라미터로 받아도 되고,
                         아래 compositeItemWriter 빈 정의 시 주입되는 barWriter와 동일한 빈을 참조해도 무방.
                         중요한 것은 StepBuilder의 .stream()에 해당 ItemStream 구현체를 전달하는 것. */
                      ) {
        return new StepBuilder("step1", jobRepository)
                .<String, String>chunk(2, transactionManager)
                .reader(fooReader)          // fooReader는 ItemStream을 구현했다면 자동으로 open/close 호출됨
                .processor(fooProcessor)    // fooProcessor도 마찬가지
                .writer(compositeItemWriter) // compositeItemWriter도 ItemStream 구현 시 자동 호출
                // 하지만 compositeItemWriter가 사용하는 barWriter는 스텝이 직접 모름!
                .stream(barWriter) // 그래서 barWriter를 ItemStream으로 명시적으로 등록!
                                   // 이제 스텝은 barWriter의 open, update, close도 호출해줌.
                .build();
    }

    // CustomCompositeItemWriter는 내부적으로 BarWriter에게 쓰기 작업을 위임
    @Bean
    public CustomCompositeItemWriter compositeItemWriter(BarWriter barWriterDelegate) { // BarWriter 빈을 주입받음
        CustomCompositeItemWriter writer = new CustomCompositeItemWriter();
        // 실제 쓰기 작업을 barWriterDelegate에게 위임하도록 설정
        writer.setDelegate(barWriterDelegate);
        return writer;
    }

    // 실제 쓰기 작업을 수행하며 ItemStream을 구현한 BarWriter
    @Bean
    public BarWriter barWriter() {
        return new BarWriter(); // 실제로는 ItemStream을 구현한 구체적인 Writer
    }

    // 예시 ItemReader (문서의 fooReader())
    @Bean
    public ItemReader<String> fooReader() {
        // 실제 ItemReader 구현 (예: ItemStream도 구현)
        // return new MyCustomReaderThatImplementsItemStream();
        // 간단한 예시:
        return new ListItemReader<>(Arrays.asList("item1", "item2", "item3", "item4"));
    }

    // 예시 ItemProcessor (문서의 fooProcessor())
    @Bean
    public ItemProcessor<String, String> fooProcessor() {
        // 실제 ItemProcessor 구현 (예: ItemStream도 구현 가능하지만 흔치 않음)
        return item -> "Processed-" + item;
    }
}

// 가상의 CustomCompositeItemWriter 와 BarWriter
// class CustomCompositeItemWriter implements ItemWriter<String> { // ItemStream을 구현할 수도 있음
//    private ItemWriter<String> delegate;
//
//    public void setDelegate(ItemWriter<String> delegate) {
//        this.delegate = delegate;
//    }
//
//    @Override
//    public void write(Chunk<? extends String> items) throws Exception {
//        if (delegate != null) {
//            delegate.write(items); // 실제 쓰기 작업을 델리게이트에게 위임
//        }
//    }
//    // 만약 CustomCompositeItemWriter 자체가 ItemStream을 구현한다면
//    // open, update, close 메서드에서 delegate의 해당 메서드들을 호출해줘야 함.
// }

// class BarWriter implements ItemWriter<String>, ItemStream { // ItemStream 구현
//    @Override
//    public void open(ExecutionContext executionContext) throws ItemStreamException {
//        System.out.println("BarWriter.open() called");
//        // 리소스 열기 또는 상태 복원 로직
//    }
//
//    @Override
//    public void update(ExecutionContext executionContext) throws ItemStreamException {
//        System.out.println("BarWriter.update() called");
//        // 상태 저장 로직
//    }
//
//    @Override
//    public void close() throws ItemStreamException {
//        System.out.println("BarWriter.close() called");
//        // 리소스 닫기 로직
//    }
//
//    @Override
//    public void write(Chunk<? extends String> items) throws Exception {
//        System.out.println("BarWriter writing items: " + items);
//        // 실제 쓰기 로직
//    }
// }

```

- **`step1` 빈 정의:**
  - `reader(fooReader)`, `processor(fooProcessor)`, `writer(compositeItemWriter)`: 스텝에 직접 연결된 주요 컴포넌트들입니다. 이들이 `ItemStream`이나 `StepListener`를 구현하면, Spring Batch가 자동으로 이들의 생명주기 메서드를 호출합니다.
  - **`stream(barWriter)`**:
    - `compositeItemWriter`는 내부적으로 `barWriter`에게 실제 쓰기 작업을 위임합니다.
    - 만약 `barWriter`가 `ItemStream`을 구현하여 `open()`, `update()`, `close()` 로직(예: 파일 열고 닫기, 상태 저장)을 가지고 있다면, `compositeItemWriter`만 스텝에 `writer`로 등록해서는 `barWriter`의 `ItemStream` 메서드들이 호출되지 않습니다.
    - 따라서 `barWriter`를 `.stream(barWriter)`를 통해 스텝에 명시적으로 "스트림"으로 등록해줍니다.
    - 이제 스텝은 `compositeItemWriter`의 생명주기 메서드뿐만 아니라, `barWriter`의 `open()`, `update()`, `close()`도 적절한 시점에 호출해줍니다.
- **`compositeItemWriter` 빈 정의:**
  - `BarWriter` 빈을 주입받아 자신의 `delegate`로 설정합니다. `CustomCompositeItemWriter`는 `write()` 메서드가 호출되면 이 `delegate` (즉, `barWriter`)에게 실제 작업을 넘길 것입니다.
- **`barWriter` 빈 정의:**
  - 실제 I/O 작업을 수행하고, `ItemStream` 인터페이스를 구현하여 리소스 관리 및 상태 저장 로직을 가질 수 있는 컴포넌트입니다.

**만약 `barWriter`가 `StepListener`도 구현했다면?**

```java
// ... StepBuilder ...
.stream(barWriter) // ItemStream으로서 등록
.listener(barWriter) // StepListener로서도 등록
// ...
```

이렇게 `listener()` 메서드를 사용해 리스너로도 등록할 수 있습니다.

---

### **"왜?" 라는 질문에 대한 답변**

- **왜 델리게이트 패턴을 사용하는가?**
  - **유연성 및 재사용성:** 여러 작은 책임을 가진 객체들을 조합하여 복잡한 기능을 만들 수 있습니다. `CompositeItemWriter`처럼, 여러 `ItemWriter`를 동적으로 조합하여 다양한 출력 요구사항에 대응할 수 있습니다.
  - **관심사 분리:** 주 객체는 조정 역할만 하고, 실제 작업은 델리게이트가 담당하므로 각 객체의 책임이 명확해집니다.
  - **단일 책임 원칙 준수:** 각 델리게이트는 하나의 특정 작업에만 집중할 수 있습니다.
- **왜 명시적 등록이 필요한가? (자동으로 안 되는 이유)**
  - Spring Batch 스텝은 자신이 직접 참조하는 `reader`, `processor`, `writer` 객체에 대해서만 생명주기를 관리하는 것이 기본 원칙입니다. 만약 주입된 객체가 내부적으로 사용하는 모든 객체의 생명주기까지 스텝이 자동으로 관리하려고 한다면, 스텝의 책임이 너무 커지고 내부 구현에 대한 의존성이 높아져 유연성이 떨어질 수 있습니다.
  - 또한, 어떤 내부 객체가 스텝 생명주기에 참여해야 하는지는 개발자의 의도에 따라 달라질 수 있습니다. 모든 내부 객체가 항상 생명주기 관리가 필요한 것은 아닐 수 있습니다.
  - 따라서, "이 델리게이트는 스텝 생명주기에 참여해야 해!"라는 것을 개발자가 명시적으로 알려주는 방식을 택한 것입니다.

---

### **주의사항 및 Best Practice**

1. **누락 주의:** 델리게이트가 `ItemStream`이나 `StepListener`를 구현하고 있는데 스텝에 `stream()`이나 `listener()`로 등록하는 것을 잊으면, 해당 생명주기 메서드들이 호출되지 않아 예기치 않은 문제(리소스 누수, 상태 불일치, 리스너 미동작 등)가 발생할 수 있습니다.
2. **중복 등록 피하기:** 스텝에 직접 `reader`, `writer`, `processor`로 등록된 컴포넌트가 `ItemStream`이나 `StepListener`를 구현한 경우, 이들은 자동으로 관리되므로 `stream()`이나 `listener()`로 *또* 등록할 필요는 없습니다. (중복 등록해도 큰 문제는 없을 수 있지만, 불필요한 설정입니다.)
3. **델리게이터(Delegator)의 역할:** 만약 주 객체(예: `CompositeItemWriter`) 자체도 `ItemStream`을 구현한다면, 주 객체의 `open()`, `update()`, `close()` 메서드 내에서 각 델리게이트들의 해당 메서드를 적절히 호출해주는 로직을 포함해야 할 수도 있습니다. 하지만, 각 델리게이트를 스텝에 직접 `stream`으로 등록하면 이러한 번거로움을 줄일 수 있습니다. (어떤 방식이 더 적합한지는 상황에 따라 다를 수 있습니다.)
4. **리스너 타입의 다양성:** `.listener()` 메서드는 `StepExecutionListener`, `ChunkListener`, `ItemReadListener`, `ItemProcessListener`, `ItemWriteListener` 등 다양한 Spring Batch 리스너 인터페이스를 구현한 객체를 등록할 수 있습니다.

---

### **이전 학습 내용과의 연관성**

- **`ItemStream`:** 이 챕터는 `ItemStream` 인터페이스의 실제 활용 예시와, 프레임워크가 이를 어떻게 관리하는지에 대한 더 깊은 이해를 제공합니다.
- **`StepListener` (아직 직접 배우지 않았다면 곧 배울 내용):** `ItemStream`과 유사하게, 스텝의 생명주기 이벤트(시작 전, 종료 후, 청크 처리 전/후 등)에 개입하여 특정 로직을 수행할 수 있게 하는 인터페이스입니다.
- **`CompositeItemWriter` / `CompositeItemProcessor` (Spring Batch 제공 구현체):** 이들은 델리게이트 패턴을 사용하는 대표적인 예시이며, 이들을 사용할 때 내부 델리게이트들의 생명주기 관리를 위해 이 챕터의 내용이 필요할 수 있습니다.
- **스텝 구성 (`StepBuilder`):** `StepBuilder`를 사용하여 스텝을 구성할 때, `.stream()`과 `.listener()` 메서드가 어떻게 사용되어 스텝의 기능을 확장하는지 보여줍니다.

---
