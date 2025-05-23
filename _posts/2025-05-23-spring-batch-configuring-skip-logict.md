---
title: Spring Batch - Configuring a Step (Skip Logic)
description: 
author: laze
date: 2025-05-23 00:00:02 +0900
categories: [Dev, SpringBatch]
tags: [SpringBatch]
---
**Skip 로직 설정하기**

처리 중 발생한 오류가 `Step` 실패로 이어져서는 안 되고 대신 건너뛰어야 하는 시나리오가 많습니다.

이는 일반적으로 데이터 자체와 그 의미를 이해하는 사람이 내려야 하는 결정입니다.

예를 들어, 금융 데이터는 돈이 이체되는 결과를 낳기 때문에 건너뛸 수 없을 수 있으며, 이는 완전히 정확해야 합니다.

반면에 공급업체 목록을 로드하는 경우 건너뛰기를 허용할 수 있습니다.

공급업체가 잘못된 형식으로 지정되었거나 필요한 정보가 누락되어 로드되지 않은 경우 문제가 없을 가능성이 높습니다.

일반적으로 이러한 잘못된 레코드는 기록되며, 이는 나중에 리스너를 논의할 때 다룹니다.

다음 Java 예제는 Skip 제한을 사용하는 예를 보여줍니다.

```java
@Bean
public Step step1(JobRepository jobRepository, PlatformTransactionManager transactionManager) {
	return new StepBuilder("step1", jobRepository)
				.<String, String>chunk(10, transactionManager)
				.reader(flatFileItemReader()) // flatFileItemReader()는 실제 빈을 참조해야 합니다.
				.writer(itemWriter())         // itemWriter()도 실제 빈을 참조해야 합니다.
				.faultTolerant()              // 내결함성 기능 활성화
				.skipLimit(10)                // 최대 10개까지 Skip 허용
				.skip(FlatFileParseException.class) // FlatFileParseException 발생 시 Skip
				.build();
}
```

*참고: `skipLimit`은 `skipLimit()` 메서드를 사용하여 명시적으로 설정할 수 있습니다. 지정하지 않으면 기본 Skip 제한은 10으로 설정됩니다.*

위 예제에서는 `FlatFileItemReader`가 사용됩니다. 어느 시점에서든 `FlatFileParseException`이 발생하면 해당 항목은 건너뛰어지고 총 Skip 제한 10개에 대해 계산됩니다.

선언된 예외(및 해당 하위 클래스)는 청크 처리의 모든 단계(읽기, 처리 또는 쓰기) 중에 발생할 수 있습니다.

Step 실행 내에서 읽기, 처리 및 쓰기에 대한 Skip 횟수는 별도로 계산되지만, 제한은 모든 Skip에 걸쳐 적용됩니다.

Skip 제한에 도달하면 다음에 발견되는 예외로 인해 Step이 실패합니다.

즉, 열 번째 Skip이 아니라 열한 번째 Skip이 예외를 트리거합니다.

위 예제의 한 가지 문제점은 `FlatFileParseException` 이외의 다른 예외는 `Job` 실패를 유발한다는 것입니다.

특정 시나리오에서는 이것이 올바른 동작일 수 있습니다.

그러나 다른 시나리오에서는 어떤 예외가 실패를 유발해야 하는지 식별하고 나머지는 모두 건너뛰는 것이 더 쉬울 수 있습니다.

다음 Java 예제는 특정 예외를 제외하는 예를 보여줍니다.

```java
@Bean
public Step step1(JobRepository jobRepository, PlatformTransactionManager transactionManager) {
	return new StepBuilder("step1", jobRepository)
				.<String, String>chunk(10, transactionManager)
				.reader(flatFileItemReader())
				.writer(itemWriter())
				.faultTolerant()
				.skipLimit(10)
				.skip(Exception.class) // 모든 Exception을 기본적으로 Skip 대상으로 지정
				.noSkip(FileNotFoundException.class) // 단, FileNotFoundException은 Skip하지 않음 (즉, 발생 시 실패)
				.build();
}
```

*참고: `skipLimit`은 `skipLimit()` 메서드를 사용하여 명시적으로 설정할 수 있습니다. 지정하지 않으면 기본 Skip 제한은 10으로 설정됩니다.*

`java.lang.Exception`을 건너뛸 수 있는 예외 클래스로 식별함으로써, 이 구성은 모든 `Exception`이 건너뛸 수 있음을 나타냅니다.

그러나 `java.io.FileNotFoundException`을 "제외"함으로써, 이 구성은 건너뛸 수 있는 예외 클래스 목록을 `FileNotFoundException`을 제외한 모든 `Exception`으로 구체화합니다.

제외된 예외 클래스는 발생하면 치명적입니다 (즉, 건너뛰지 않습니다).

발생하는 모든 예외에 대해, 건너뛸 수 있는지 여부는 클래스 계층 구조에서 가장 가까운 상위 클래스에 의해 결정됩니다. 분류되지 않은 예외는 '치명적'으로 처리됩니다.

`skip` 및 `noSkip` 메서드 호출 순서는 중요하지 않습니다.

---

## 오류에도 굴하지 않는 Step 만들기: Skip 로직 활용법! 🤸‍♂️🚧

배치 작업, 특히 대량의 데이터를 다루다 보면 데이터 몇 건이 형식이 잘못되었거나, 특정 아이템 처리 중에 일시적인 문제가 생길 수 있어요.

**상황 예시:**

- **금융 데이터 처리:** 고객 계좌 이체 같은 중요한 작업이라면, 단 하나의 오류도 용납할 수 없겠죠? 이런 경우는 오류 발생 즉시 `Step`이 실패해야 해요. (Skip 사용 X)
- **외부 업체 목록 로드:** 수천 개의 업체 정보를 파일에서 읽어오는데, 그중 한두 개 업체 정보의 특정 필드가 빠져있거나 형식이 살짝 안 맞는다고 해봐요. 이 한두 건 때문에 전체 로드 작업을 중단시키는 건 너무 비효율적일 수 있어요. 이런 경우, 문제가 있는 데이터는 건너뛰고(Skip), 나머지는 정상적으로 처리한 후, 나중에 건너뛴 데이터만 따로 확인하는 것이 더 나을 수 있어요. (Skip 사용 O)

이처럼 "이 오류는 건너뛰어도 괜찮아" 또는 "이 오류는 절대 건너뛰면 안 돼!"를 결정하는 것은 **데이터의 중요도와 비즈니스 규칙**에 따라 달라져요.

Spring Batch의 Skip 로직은 바로 이런 상황에서 유연하게 대처할 수 있도록 도와줍니다.

---

### Skip 로직 설정의 핵심 삼총사!

Skip 로직을 설정하려면 `StepBuilder`를 사용할 때 다음과 같은 메서드들을 활용해요. (그리고 이 기능들을 사용하려면 먼저 **`.faultTolerant()`**를 호출해서 "이 `Step`은 오류에 좀 더 너그러워질 준비가 됐어!" 라고 알려줘야 해요.)

1. **`.faultTolerant()`**: 내결함성 기능 활성화! Skip, Retry 같은 고급 오류 처리 기능을 사용하기 위한 필수 선언이에요.
2. **`.skipLimit(횟수)`**: "최대 이만큼만 건너뛸 거야!"
  - `Step` 전체에서 건너뛸 수 있는 아이템의 **최대 허용 개수**를 설정해요.
  - 예를 들어, `.skipLimit(10)`으로 설정하면, 최대 10개의 아이템까지는 오류가 발생해도 건너뛰고 계속 진행해요.
  - 만약 11번째 아이템에서도 건너뛸 만한 오류가 발생하면, 그때는 "더 이상은 못 참아!" 하면서 `Step`이 최종적으로 실패해요.
  - **기본값:** 명시적으로 설정하지 않으면, `.skipLimit(10)`이 기본값으로 적용될 수 있습니다. (문서에는 10이라고 명시되어 있지만, 버전이나 세부 설정에 따라 다를 수 있으니 명시적으로 지정하는 것이 좋습니다.)
3. **`.skip(Exception클래스)`**: "이런 종류의 오류는 건너뛰자!"
  - 건너뛸 **특정 예외(Exception) 클래스**를 지정해요.
  - 여기에 지정된 예외 또는 그 예외의 하위 클래스에 해당하는 오류가 발생하면, 해당 아이템은 Skip 대상으로 간주돼요.
  - 여러 번 호출해서 다양한 예외를 Skip 대상으로 추가할 수 있어요.
  - 예시: `.skip(FlatFileParseException.class)` - 파일 파싱 오류는 건너뛰겠다!
4. **`.noSkip(Exception클래스)`**: "이런 종류의 오류는 절대 건너뛰면 안 돼! (즉시 실패!)"
  - `.skip(Exception.class)`처럼 광범위한 예외를 Skip 대상으로 지정했을 때, 그중에서도 **특정 예외만큼은 건너뛰지 않고 즉시 `Step`을 실패시키고 싶을 때** 사용해요.
  - 여러 번 호출해서 Skip하지 않을 예외를 여러 개 지정할 수 있어요.
  - 예시: `.skip(Exception.class).noSkip(FileNotFoundException.class)` - 대부분의 예외는 건너뛰지만, 파일을 아예 못 찾는 오류는 즉시 실패시키겠다!

---

### Skip 로직 작동 방식 및 예시

**기본적인 Skip 설정 예시 (Java):**

```java
@Bean
public Step fileProcessingStep(JobRepository jobRepository, PlatformTransactionManager transactionManager,
                               ItemReader<String> flatFileItemReader, ItemWriter<String> itemWriter) {
    return new StepBuilder("fileProcessingStep", jobRepository)
            .<String, String>chunk(10, transactionManager) // 청크 크기는 10
            .reader(flatFileItemReader)
            .writer(itemWriter)
            .faultTolerant() // 1. 내결함성 기능 활성화!
            .skipLimit(5)    // 2. 최대 5번까지만 건너뛰기 허용
            .skip(FlatFileParseException.class) // 3. FlatFileParseException 발생하면 건너뛰기
            // .skip(NumberFormatException.class) // 다른 예외도 추가 가능
            .build();
}

```

**작동 시나리오:**

- `flatFileItemReader`가 파일을 읽다가 `FlatFileParseException`이 발생하면:
  1. 해당 아이템은 Skip 처리됩니다.
  2. Skip된 아이템 수가 카운트됩니다 (현재 Skip 횟수: 1).
  3. `skipLimit` (5)을 아직 넘지 않았으므로 `Step`은 계속 진행됩니다.
- 이런 식으로 `FlatFileParseException`이 계속 발생해서 Skip된 아이템 수가 5개가 되면,
- 그다음 또 `FlatFileParseException`이 발생하면 (즉, 6번째 Skip 시도), `skipLimit`을 초과했으므로 `Step`은 최종적으로 실패합니다.
- 만약 `FlatFileParseException`이 아닌 다른 종류의 예외 (예: `NullPointerException`)가 발생하면, `.skip()`에 해당 예외가 지정되지 않았으므로 `Step`은 즉시 실패합니다.

**Skip 대상 예외를 더 넓게, 특정 예외는 제외하는 예시 (Java):**

```java
@Bean
public Step advancedSkippingStep(JobRepository jobRepository, PlatformTransactionManager transactionManager,
                                 ItemReader<String> reader, ItemWriter<String> writer) {
    return new StepBuilder("advancedSkippingStep", jobRepository)
            .<String, String>chunk(10, transactionManager)
            .reader(reader)
            .writer(writer)
            .faultTolerant()
            .skipLimit(20)
            .skip(Exception.class) // 1. 일단 모든 Exception 종류의 오류를 건너뛰기 대상으로 설정!
            .noSkip(FileNotFoundException.class) // 2. 하지만 FileNotFoundException은 건너뛰지 말고 즉시 실패!
            .noSkip(MyCriticalBusinessException.class) // 3. 그리고 내가 만든 MyCriticalBusinessException도 즉시 실패!
            .build();
}
```

**작동 시나리오:**

- 이 `Step`에서는 대부분의 `Exception` (그리고 그 하위 예외들)이 발생하면 Skip 처리됩니다 (최대 20번까지).
- 하지만 만약 `FileNotFoundException`이나 `MyCriticalBusinessException` (또는 이들의 하위 예외)이 발생하면, `.noSkip()`에 의해 즉시 `Step`이 실패합니다.
- `.skip()`과 `.noSkip()`의 선언 순서는 중요하지 않아요. Spring Batch가 알아서 잘 판단해 줍니다.

**예외 계층 구조와 Skip 판단:**

어떤 예외가 발생했을 때, Spring Batch는 그 예외와 가장 가까운 상위 클래스 중에서 `.skip()` 또는 `.noSkip()`에 지정된 것이 있는지 확인해서 Skip 여부를 결정해요. 만약 아무런 규칙에도 해당하지 않는 (분류되지 않은) 예외라면, 기본적으로 '치명적(fatal)'으로 간주되어 `Step`이 실패합니다.

---

### Skip 로직은 언제, 어디서 적용될까요?

Skip 로직은 청크 지향 `Step`의 **모든 단계 (Read, Process, Write)**에서 발생할 수 있는 예외에 대해 적용될 수 있어요.

- `ItemReader`에서 데이터 읽다가 오류 발생 시
- `ItemProcessor`에서 데이터 가공하다가 오류 발생 시
- `ItemWriter`에서 데이터 쓰다가 오류 발생 시

각 단계(Read, Process, Write)에서 Skip된 횟수는 `StepExecution` 내에 별도로 기록되지만, **`skipLimit`은 이 모든 단계에서 발생한 Skip 횟수의 총합에 대해 적용**돼요.

---

### Skip된 아이템은 어떻게 처리될까요?

기본적으로 Skip된 아이템은 그냥 "건너뛰고" 다음 아이템 처리로 넘어가요. 하지만 그냥 버려두면 안 되겠죠?

보통 **리스너(`SkipListener`)**를 함께 사용해서 Skip된 아이템의 정보와 오류 원인 등을 **로그로 남기거나, 별도의 파일/DB에 기록**해서 나중에 수동으로 처리하거나 원인을 분석할 수 있도록 해요. (리스너는 우리가 이전에 배웠죠? 😊)

---

**Skip 로직 사용 시 고려사항:**

- **정확한 예외 지정:** 어떤 예외를 Skip하고, 어떤 예외를 실패로 처리할지 명확하게 정의해야 해요. 너무 광범위하게 `Exception.class`를 Skip 대상으로 하면 중요한 오류를 놓칠 수 있어요.
- **적절한 `skipLimit` 설정:** `skipLimit`을 너무 높게 잡으면, 문제가 있는 데이터가 너무 많이 무시될 수 있어요. 반대로 너무 낮게 잡으면 Skip 로직의 유연성이 떨어질 수 있어요.
- **Skip된 데이터의 후처리 방안:** Skip된 데이터를 어떻게 추적하고 관리할지에 대한 계획이 필요해요. `SkipListener`를 활용하는 것이 일반적이에요.
- **비즈니스 영향도 분석:** 특정 데이터를 Skip했을 때 비즈니스적으로 어떤 영향을 미치는지 반드시 고려해야 해요.

**핵심:** Skip 로직은 배치 작업의 **안정성과 유연성을 높여주는 강력한 기능**이에요. 하지만 무분별하게 사용하기보다는, 데이터의 특성과 비즈니스 요구사항을 충분히 고려해서 신중하게 설정해야 합니다!

---
