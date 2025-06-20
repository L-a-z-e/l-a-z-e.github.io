---
title: Spring Batch - Reusing Existing Services
description: 
author: laze
date: 2025-06-20 00:00:02 +0900
categories: [Dev, SpringBatch]
tags: [SpringBatch]
---
### Reusing Existing Services

배치 시스템은 종종 다른 애플리케이션 스타일과 함께 사용됩니다.

가장 일반적인 것은 온라인 시스템이지만, 각 애플리케이션 스타일이 사용하는 필요한 대량 데이터를 이동시킴으로써 통합 또는 심지어 씩 클라이언트(thick client) 애플리케이션을 지원할 수도 있습니다.

이러한 이유로 많은 사용자가 배치 잡 내에서 기존 DAO 또는 기타 서비스를 재사용하기를 원하는 것이 일반적입니다.

Spring 컨테이너 자체는 필요한 모든 클래스를 주입할 수 있도록 하여 이를 상당히 쉽게 만듭니다.

그러나 다른 Spring Batch 클래스의 의존성을 충족시키기 위해서나 또는 그것이 실제로 스텝의 주요 `ItemReader`이기 때문에 기존 서비스가 `ItemReader` 또는 `ItemWriter` 역할을 해야 하는 경우가 있을 수 있습니다.

래핑(wrapping)이 필요한 각 서비스에 대해 어댑터 클래스를 작성하는 것은 상당히 사소한 일이지만,

이것이 매우 일반적인 관심사이기 때문에 Spring Batch는 `ItemReaderAdapter`와 `ItemWriterAdapter`라는 구현을 제공합니다.

두 클래스 모두 델리게이트 패턴을 호출하여 표준 Spring 메서드를 구현하며 설정이 매우 간단합니다.

```java
@Bean
public ItemReaderAdapter<Foo> itemReader(FooService fooService) { // FooService 주입, Foo 타입 아이템 가정
	ItemReaderAdapter<Foo> reader = new ItemReaderAdapter<>();

	reader.setTargetObject(fooService); // 실제 작업을 위임할 대상 객체 설정
	reader.setTargetMethod("generateFoo"); // 대상 객체에서 호출할 메서드 이름 설정

	return reader;
}

// FooService는 기존 서비스라고 가정
@Bean
public FooService fooService() {
	return new FooService();
}
```

한 가지 중요한 점은 `targetMethod`의 계약이 `read`의 계약과 동일해야 한다는 것입니다:

소진되면(더 이상 아이템이 없으면) `null`을 반환해야 합니다.

그렇지 않으면 객체를 반환합니다.

다른 어떤 것이든 프레임워크가 처리를 언제 끝내야 하는지 알 수 없게 만들어, `ItemWriter`의 구현에 따라 무한 루프를 발생시키거나 잘못된 실패를 야기할 수 있습니다.

```java
@Bean
public ItemWriterAdapter<Foo> itemWriter(FooService fooService) { // FooService 주입, Foo 타입 아이템 가정
	ItemWriterAdapter<Foo> writer = new ItemWriterAdapter<>();

	writer.setTargetObject(fooService); // 실제 작업을 위임할 대상 객체 설정
	writer.setTargetMethod("processFoo"); // 대상 객체에서 호출할 메서드 이름 설정 (보통 아이템 하나를 인자로 받음)

	return writer;
}

// FooService는 기존 서비스라고 가정
@Bean
public FooService fooService() {
	return new FooService();
}
```

---

### **학습 목표 제시**

이번 "Reusing Existing Services" 챕터를 통해 학생분은 다음을 이해하고 배울 수 있습니다:

1. **기존 서비스 재사용의 필요성 및 이점 이해:** 배치 잡 개발 시 새로운 로직을 모두 작성하는 대신, 이미 검증된 기존 서비스(DAO, 비즈니스 서비스 등)를 활용함으로써 개발 효율성과 시스템 일관성을 높일 수 있음을 이해합니다.
2. **`ItemReaderAdapter` 및 `ItemWriterAdapter`의 역할과 사용법 학습:** 기존 서비스의 특정 메서드를 Spring Batch의 `ItemReader` 또는 `ItemWriter` 인터페이스에 맞게 중개(adapt)해주는 `ItemReaderAdapter`와 `ItemWriterAdapter`의 개념과 주요 설정 방법(`targetObject`, `targetMethod`)을 익힙니다.
3. **어댑터 사용 시 주의사항 파악 (특히 `ItemReaderAdapter`의 `targetMethod` 계약):** `ItemReaderAdapter`에 연결되는 대상 메서드가 `ItemReader`의 `read()` 메서드 계약(데이터 소진 시 `null` 반환)을 반드시 지켜야 하는 이유와 중요성을 이해합니다.

---

### **핵심 개념 설명**

이 챕터의 핵심은 **"기존에 만들어둔 일반 자바 클래스(서비스, DAO 등)의 메서드를 어떻게 Spring Batch의 `ItemReader`나 `ItemWriter`처럼 동작하도록 만들 수 있을까?"** 입니다.

이를 위해 Spring Batch는 **어댑터(Adapter) 패턴**을 활용한 `ItemReaderAdapter`와 `ItemWriterAdapter`를 제공합니다.

### 1. 왜 기존 서비스를 재사용해야 하는가?

- **개발 효율성:** 이미 존재하는 비즈니스 로직이나 데이터 접근 로직을 배치 잡을 위해 처음부터 다시 만들 필요가 없습니다. 이는 개발 시간과 비용을 절약해줍니다.
- **일관성 유지:** 온라인 시스템과 배치 시스템이 동일한 서비스 로직을 공유함으로써 데이터 처리 방식이나 비즈니스 규칙의 일관성을 유지할 수 있습니다.
- **검증된 로직 활용:** 이미 운영 환경에서 검증된 안정적인 서비스를 재사용함으로써 배치 잡의 신뢰성을 높일 수 있습니다.
- **Spring DI의 장점 활용:** 기존 서비스가 스프링 빈(Bean)으로 관리되고 있다면, Spring Batch 잡 설정에서도 쉽게 의존성 주입(DI)을 통해 해당 서비스를 가져와 사용할 수 있습니다.

### 2. 어댑터 패턴 (Adapter Pattern)이란?

- **개념:** 서로 호환되지 않는 인터페이스를 가진 클래스들이 함께 동작할 수 있도록, 하나의 인터페이스를 다른 인터페이스로 변환해주는 역할을 하는 패턴입니다.
- **비유:**
  - **해외여행용 멀티 어댑터 플러그:** 우리나라의 220V 플러그를 다른 나라의 전원 콘센트(다른 인터페이스)에 꽂을 수 있도록 중간에서 모양을 바꿔주는 어댑터.
  - **통역사:** 서로 다른 언어(다른 인터페이스)를 사용하는 두 사람이 대화할 수 있도록 중간에서 말을 번역해주는 사람.
- **Spring Batch에서의 어댑터:**
  - `ItemReaderAdapter`: 일반 자바 객체의 특정 메서드를 `ItemReader` 인터페이스의 `read()` 메서드처럼 보이게 만들어줍니다.
  - `ItemWriterAdapter`: 일반 자바 객체의 특정 메서드를 `ItemWriter` 인터페이스의 `write(Chunk items)` 메서드처럼 보이게 만들어줍니다. (정확히는 `write()`는 청크를 받지만, 어댑터는 보통 청크 내의 각 아이템에 대해 대상 메서드를 호출합니다.)

### 3. `ItemReaderAdapter<T>`

- **역할:** 기존 서비스 객체의 특정 메서드를 호출하여 그 반환값을 `ItemReader`의 `read()` 메서드가 반환하는 아이템처럼 사용하게 해줍니다.
- **주요 설정:**
  1. **`setTargetObject(Object targetObject)`**: 실제 작업을 위임할 대상 객체(기존 서비스 빈)를 설정합니다.
  2. **`setTargetMethod(String targetMethodName)`**: `targetObject`에서 호출될 메서드의 이름을 문자열로 설정합니다. 이 메서드가 아이템을 하나씩 반환해야 합니다.
- **매우 중요한 계약 조건 (`targetMethod`의 반환값):**
  - `ItemReaderAdapter`에 연결된 `targetMethod`는 `ItemReader`의 `read()` 메서드와 **동일한 계약**을 지켜야 합니다.
  - 즉, **더 이상 반환할 아이템이 없으면 반드시 `null`을 반환해야 합니다.**
  - 만약 `null`을 반환하지 않고 계속 객체를 반환하거나, 데이터가 없을 때 예외를 던지면, Spring Batch 프레임워크는 언제 읽기 작업이 끝나야 하는지 알 수 없습니다. 이는 **무한 루프**를 발생시키거나, `ItemWriter`의 구현에 따라 잘못된 실패를 야기할 수 있습니다.
- **예시 (문서 코드 기반):**

    ```java
    // 기존 서비스 (데이터를 하나씩 생성하거나 가져오는 메서드가 있다고 가정)
    public class FooService {
        private int counter = 0;
        private final int MAX_ITEMS = 5;
    
        // 이 메서드가 ItemReaderAdapter의 targetMethod가 됨
        public Foo generateFoo() {
            if (counter < MAX_ITEMS) {
                counter++;
                return new Foo("Item " + counter); // Foo 객체 반환
            } else {
                return null; // 더 이상 아이템이 없으면 null 반환! (매우 중요)
            }
        }
    }
    
    // Foo.java (간단한 도메인 객체)
    // public class Foo {
    //    private String name;
    //    public Foo(String name) { this.name = name; }
    //    @Override public String toString() { return "Foo{name='" + name + "'}"; }
    // }
    
    // Spring Batch 설정
    @Bean
    public FooService fooService() { // 기존 서비스 빈 등록
        return new FooService();
    }
    
    @Bean
    public ItemReaderAdapter<Foo> fooItemReaderAdapter(FooService fooService) { // FooService 주입
        ItemReaderAdapter<Foo> adapter = new ItemReaderAdapter<>();
        adapter.setTargetObject(fooService);        // 대상 객체 설정
        adapter.setTargetMethod("generateFoo");     // 호출할 메서드 이름 설정
        // adapter.setArguments(new Object[]{arg1, arg2}); // (선택) 대상 메서드에 인자를 전달해야 할 경우
        return adapter;
    }
    
    // 이 fooItemReaderAdapter를 스텝의 reader로 사용하면,
    // 스텝 실행 시 fooService.generateFoo()가 반복 호출됨.
    // generateFoo()가 null을 반환하면 읽기 종료.
    
    ```


### 4. `ItemWriterAdapter<T>`

- **역할:** 전달받은 아이템(또는 아이템 묶음)을 기존 서비스 객체의 특정 메서드에 인자로 전달하여 처리하도록 합니다.
- **주요 설정:**
  1. **`setTargetObject(Object targetObject)`**: 실제 작업을 위임할 대상 객체(기존 서비스 빈)를 설정합니다.
  2. **`setTargetMethod(String targetMethodName)`**: `targetObject`에서 호출될 메서드의 이름을 문자열로 설정합니다. 이 메서드는 보통 처리할 아이템을 인자로 받습니다.
- **동작 방식:** `ItemWriterAdapter`의 `write(Chunk<? extends T> items)` 메서드가 호출되면, 청크 내의 각 아이템에 대해 `targetObject.targetMethod(item)`을 호출합니다.
- **예시 (문서 코드 기반):**

    ```java
    // 기존 서비스 (아이템을 받아 처리하는 메서드가 있다고 가정)
    public class FooService {
        // ... (generateFoo 메서드는 위와 동일하다고 가정) ...
    
        // 이 메서드가 ItemWriterAdapter의 targetMethod가 됨
        public void processFoo(Foo item) { // Foo 타입의 아이템을 인자로 받음
            System.out.println("Processing in FooService: " + item);
            // 실제로는 DB에 저장하거나, 다른 시스템에 전송하는 등의 로직 수행
        }
    
        // 만약 청크 전체를 한 번에 처리하는 메서드가 있다면:
        // public void processFoos(List<Foo> items) {
        //    System.out.println("Processing foos in FooService: " + items.size() + " items.");
        // }
        // 이 경우, ItemWriterAdapter 대신 직접 ItemWriter를 구현하거나,
        // 또는 청크를 List로 변환하여 전달하는 커스텀 어댑터 로직이 필요할 수 있음.
        // (ItemWriterAdapter는 기본적으로 각 아이템에 대해 targetMethod를 호출)
    }
    
    // Spring Batch 설정
    // @Bean public FooService fooService() { ... } // 위와 동일
    
    @Bean
    public ItemWriterAdapter<Foo> fooItemWriterAdapter(FooService fooService) { // FooService 주입
        ItemWriterAdapter<Foo> adapter = new ItemWriterAdapter<>();
        adapter.setTargetObject(fooService);     // 대상 객체 설정
        adapter.setTargetMethod("processFoo");  // 호출할 메서드 이름 설정
                                                // (processFoo 메서드는 Foo 타입 인자를 하나 받아야 함)
        return adapter;
    }
    
    // 이 fooItemWriterAdapter를 스텝의 writer로 사용하면,
    // 스텝 실행 시 각 Foo 아이템에 대해 fooService.processFoo(item)이 호출됨.
    
    ```

  - **주의:** `ItemWriterAdapter`의 `targetMethod`는 기본적으로 **단일 아이템**을 인자로 받도록 기대합니다. 만약 기존 서비스 메서드가 `List<Foo>`처럼 아이템의 리스트를 인자로 받는다면, `ItemWriterAdapter`를 그대로 사용하기 어렵고, 커스텀 `ItemWriter`를 만들거나 다른 접근 방식(예: `ChunkOrientedTasklet` 직접 사용 등)을 고려해야 할 수 있습니다. (문서에서는 이 부분에 대해 상세히 다루지 않았지만, 일반적인 `ItemWriterAdapter` 사용법은 단일 아이템 처리입니다.)

---

### **주요 용어 해설**

- **어댑터 (Adapter):** 서로 다른 인터페이스를 연결해주는 중간 객체.
- **델리게이트 패턴 (Delegate Pattern):** 한 객체가 다른 객체에게 작업을 위임하는 패턴. 어댑터 패턴과 함께 자주 사용됩니다. `ItemReader/WriterAdapter`에서 `targetObject`가 델리게이트 역할을 합니다.
- **래핑 (Wrapping):** 기존 객체를 다른 객체로 감싸서 새로운 인터페이스를 제공하거나 부가 기능을 추가하는 것. 어댑터도 일종의 래퍼입니다.

---

### **코드 예제 분석 (문서 예제 기반으로 좀 더 구체화)**

문서의 예제는 `FooService`라는 가상의 서비스와 `generateFoo`, `processFoo`라는 메서드를 사용합니다. 위에서 제가 작성한 예시 코드가 이 시나리오를 좀 더 구체화한 것입니다.

**핵심 흐름:**

1. **`FooService` 빈 정의:** 기존에 사용하던 서비스 로직을 담고 있는 `FooService`를 스프링 빈으로 등록합니다.
2. **`ItemReaderAdapter` 설정:**
  - 새로운 `ItemReaderAdapter` 빈을 정의합니다.
  - `setTargetObject()`: `fooService` 빈을 주입하여 설정합니다.
  - `setTargetMethod()`: `fooService`의 `generateFoo` 메서드 이름을 문자열로 지정합니다.
  - **`generateFoo()` 메서드 구현 시 주의:** 반드시 데이터가 소진되면 `null`을 반환하도록 구현해야 합니다.
3. **`ItemWriterAdapter` 설정:**
  - 새로운 `ItemWriterAdapter` 빈을 정의합니다.
  - `setTargetObject()`: `fooService` 빈을 주입하여 설정합니다.
  - `setTargetMethod()`: `fooService`의 `processFoo` 메서드 이름을 문자열로 지정합니다.
  - **`processFoo(Foo item)` 메서드 구현 시 주의:** `ItemWriterAdapter`는 청크의 각 아이템에 대해 이 메서드를 호출하므로, 단일 `Foo` 아이템을 인자로 받도록 구현해야 합니다.
4. **스텝(Step)에 어댑터 등록:**
  - 이렇게 생성된 `fooItemReaderAdapter`를 스텝의 `reader()`로, `fooItemWriterAdapter`를 스텝의 `writer()`로 설정합니다.

이제 스텝이 실행되면, `ItemReaderAdapter`는 내부적으로 `fooService.generateFoo()`를 호출하여 아이템을 읽어오고, `ItemWriterAdapter`는 각 아이템에 대해 `fooService.processFoo(item)`을 호출하여 아이템을 처리(쓰기)합니다.

---

### **"왜?" 라는 질문에 대한 답변**

- **왜 직접 `ItemReader`/`ItemWriter`를 구현하지 않고 어댑터를 사용할까?**
  - **간편함:** 기존 서비스 메서드의 시그니처가 `ItemReader.read()` (데이터 없으면 null 반환) 또는 `ItemWriter`가 기대하는 단일 아이템 처리 메서드와 유사하다면, 단순히 어댑터에 객체와 메서드 이름만 설정해주면 되므로 매우 간편합니다. `ItemReader`나 `ItemWriter` 인터페이스를 직접 구현하는 것보다 코드량이 훨씬 적습니다.
  - **기존 코드 변경 최소화:** 기존 서비스 코드를 Spring Batch 인터페이스에 맞게 수정할 필요가 없습니다. 기존 코드는 그대로 두고 어댑터를 통해 연결만 하면 됩니다. 이는 특히 수정하기 어려운 레거시 코드를 활용할 때 유용합니다.
- **`ItemReaderAdapter`에서 `targetMethod`가 `null`을 반환하는 것이 왜 그렇게 중요한가?**
  - Spring Batch의 청크 지향 스텝은 `ItemReader.read()`가 `null`을 반환할 때까지 아이템을 계속 읽으려고 시도합니다. 이것이 "데이터의 끝"을 의미하는 유일한 신호입니다.
  - 만약 `targetMethod`가 `null`을 반환하지 않으면, 스텝은 데이터가 끝없이 계속 나오는 것으로 착각하고 무한정 `read()`를 호출하게 되어 **무한 루프**에 빠집니다.
  - 또는, `targetMethod`가 데이터가 없을 때 예외를 던진다면, 스텝은 이를 실제 오류로 간주하여 잡을 실패시킬 수 있습니다. (skip 정책이 없다면)
  - 따라서 `ItemReader`의 계약을 정확히 따르는 것이 매우 중요합니다.

---

### **주의사항 및 Best Practice**

1. **`ItemReaderAdapter`의 `targetMethod` 계약 준수:** 반복해서 강조하지만, `targetMethod`는 데이터 소진 시 반드시 `null`을 반환해야 합니다. 이것이 가장 중요한 주의사항입니다.
2. **`targetMethod`의 시그니처:**
  - `ItemReaderAdapter`의 `targetMethod`: 보통 인자 없이 아이템 객체 또는 `null`을 반환하는 메서드.
  - `ItemWriterAdapter`의 `targetMethod`: 보통 단일 아이템 객체를 인자로 받고 반환값이 없는(`void`) 메서드. (만약 반환값이 있다면 무시됨)
3. **메서드 인자 전달:** `ItemReaderAdapter`나 `ItemWriterAdapter`의 `targetMethod`가 인자를 필요로 한다면, `setArguments(Object[] args)` 메서드를 사용하여 정적인 인자를 전달할 수 있습니다. 하지만 동적인 인자(예: `JobParameter`)를 전달하려면 추가적인 커스터마이징이나 `@StepScope`와 함께 SpEL을 사용하는 다른 접근 방식이 필요할 수 있습니다. (이 챕터에서는 정적 인자 전달까지만 암시)
4. **상태 관리 및 `ItemStream`:**
  - `ItemReaderAdapter`나 `ItemWriterAdapter` 자체는 `ItemStream`을 구현하지 않습니다.
  - 만약 어댑터가 참조하는 `targetObject`(기존 서비스)가 상태를 가지고 있고, 이 상태가 스텝 생명주기에 따라 관리(open, update, close)되어야 한다면, 해당 `targetObject`가 `ItemStream`을 구현하고, 이전 챕터에서 배운 것처럼 스텝에 `.stream(targetObject)`로 명시적으로 등록해주어야 합니다. 어댑터는 단순히 메서드 호출을 중개할 뿐, `ItemStream` 생명주기를 자동으로 위임하지는 않습니다.
5. **성능 고려:** 리플렉션을 통해 메서드를 호출하므로, 매우 민감한 고성능 환경에서는 직접 `ItemReader`/`ItemWriter`를 구현하는 것보다 약간의 오버헤드가 있을 수 있습니다. 하지만 대부분의 경우 그 차이는 미미하며, 개발 편의성이 더 큰 이점을 제공합니다.

---

### **이전 학습 내용과의 연관성**

- **델리게이트 패턴 및 스텝 등록:** 이 챕터는 델리게이트 패턴의 구체적인 활용 예시이며, 만약 `targetObject`가 `ItemStream` 등의 생명주기 관리가 필요하다면 이전 챕터에서 배운 명시적 스텝 등록이 필요할 수 있다는 점을 상기시킵니다.
- **`ItemReader` / `ItemWriter` 인터페이스:** 이 어댑터들은 기존 클래스를 이들 인터페이스처럼 보이게 만들어줍니다.
- **Spring DI:** 기존 서비스를 스프링 빈으로 등록하고 어댑터에 주입하여 사용하는 것은 Spring DI의 기본적인 활용입니다.

---
