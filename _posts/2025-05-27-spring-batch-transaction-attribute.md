---
title: Spring Batch - Configuring a Step (Transaction Attribute)
description: 
author: laze
date: 2025-05-27 00:00:02 +0900
categories: [Dev, SpringBatch]
tags: [SpringBatch]
---
## 트랜잭션 속성 (Transaction Attributes)

트랜잭션 속성을 사용하여 격리 수준(isolation), 전파 방식(propagation), 그리고 타임아웃(timeout) 설정을 제어할 수 있습니다.

트랜잭션 속성 설정에 대한 더 자세한 정보는 Spring 핵심 문서(Spring core documentation)에서 찾아볼 수 있습니다.

다음 예제는 자바에서 격리 수준, 전파 방식, 그리고 타임아웃 트랜잭션 속성을 설정하는 방법을 보여줍니다:

**자바 설정 (Java Configuration)**

```java
@Bean
public Step step1(JobRepository jobRepository, PlatformTransactionManager transactionManager) {
	DefaultTransactionAttribute attribute = new DefaultTransactionAttribute();
	attribute.setPropagationBehavior(Propagation.REQUIRED.value());
	attribute.setIsolationLevel(Isolation.DEFAULT.value());
	attribute.setTimeout(30); // 초 단위

	return new StepBuilder("step1", jobRepository)
				.<String, String>chunk(2, transactionManager) // <Input, Output> 타입, 청크 크기, 트랜잭션 매니저
				.reader(itemReader())
				.writer(itemWriter())
				.transactionAttribute(attribute) // 설정된 트랜잭션 속성 적용
				.build();
}
```

---

**학습 목표:**

이번 챕터를 통해 여러분은 다음을 이해하고 배울 수 있습니다:

1. **Spring Batch Step에서 트랜잭션 속성을 설정하는 방법을 이해합니다.**
2. **주요 트랜잭션 속성인 격리 수준(Isolation), 전파 방식(Propagation), 타임아웃(Timeout)의 개념과 역할을 파악합니다.**
3. **Java 기반 설정에서 `DefaultTransactionAttribute` 객체를 사용하여 Step에 원하는 트랜잭션 속성을 적용하는 과정을 설명할 수 있게 됩니다.**

---

### 핵심 개념 설명

우리가 Spring Batch에서 작업을 처리할 때, 특히 데이터를 읽고 쓰는 과정(Chunk-Oriented Processing)은 하나의 **트랜잭션(Transaction)** 안에서 이루어지는 경우가 많습니다.

트랜잭션은 "모 아니면 도"처럼, 관련된 작업들이 모두 성공하거나 모두 실패하도록 보장하는 중요한 메커니즘이죠.

마치 은행에서 돈을 이체할 때, 출금과 입금이 모두 성공해야만 거래가 완료되는 것과 같습니다.

이번 챕터에서 다루는 **트랜잭션 속성(Transaction Attributes)**은 이러한 트랜잭션이 어떻게 동작할지를 세밀하게 제어하는 설정값들이라고 생각하시면 됩니다.

우리가 음식점에서 주문할 때 "덜 맵게 해주세요", "소스는 따로 주세요" 와 같이 요청하는 것처럼, 트랜잭션에게도 "이렇게 동작해줘!" 라고 구체적인 지시를 내리는 것이죠.

이 챕터에서는 특히 다음 세 가지 중요한 속성에 대해 이야기하고 있습니다:

1. **격리 수준 (Isolation Level):**
  - 여러 트랜잭션이 동시에 실행될 때, 서로에게 얼마나 영향을 미칠지를 결정하는 수준입니다.
  - **비유:** 여러 사람이 동시에 같은 문서를 편집한다고 상상해 보세요.
    - **가장 낮은 격리 수준:** 다른 사람이 수정 중인 내용을 바로바로 볼 수 있지만, 아직 확정되지 않은 내용이라 혼란스러울 수 있습니다. (예: `READ_UNCOMMITTED`)
    - **중간 격리 수준:** 다른 사람이 수정을 완료(커밋)한 내용만 볼 수 있습니다. 하지만, 내가 읽는 동안 다른 사람이 새로운 내용을 추가하거나 수정할 수 있어서, 같은 데이터를 두 번 읽었을 때 결과가 다를 수 있습니다. (예: `READ_COMMITTED`, `REPEATABLE_READ`)
    - **가장 높은 격리 수준:** 내가 문서를 다 읽을 때까지 다른 사람은 해당 문서를 수정할 수 없습니다. 가장 안전하지만, 다른 사람들은 기다려야 하므로 성능에 영향을 줄 수 있습니다. (예: `SERIALIZABLE`)
  - Spring Batch에서는 기본적으로 데이터베이스의 기본 격리 수준을 따르지만, 필요에 따라 Step별로 다르게 설정할 수 있습니다.
2. **전파 방식 (Propagation Behavior):**
  - 이미 진행 중인 트랜잭션이 있을 때, 새로운 트랜잭션이 어떻게 동작할지를 결정하는 방식입니다.
  - **비유:** 팀 프로젝트를 진행하는데, 이미 팀장님이 시작한 큰 작업(외부 트랜잭션)이 있다고 가정해 봅시다. 내가 맡은 작은 작업(내부 트랜잭션)을 시작할 때,
    - **`REQUIRED` (기본값):** 팀장님이 시작한 작업에 합류합니다. 팀장님 작업이 실패하면 내 작업도 롤백됩니다. 만약 팀장님 작업이 없다면, 내가 새롭게 작업을 시작합니다. (가장 일반적인 방식)
    - **`REQUIRES_NEW`:** 팀장님 작업과는 별개로, 나만의 독립적인 새 작업을 시작합니다. 내 작업의 성공/실패는 팀장님 작업에 영향을 주지 않고, 팀장님 작업의 성공/실패도 내 작업에 직접적인 영향을 주지 않습니다. (예: 로그를 남기는 작업처럼, 원래 작업의 성공 여부와 관계없이 항상 기록되어야 하는 경우)
    - **`SUPPORTS`:** 팀장님 작업이 있으면 합류하고, 없으면 그냥 작업 없이 진행합니다.
    - **`NOT_SUPPORTED`:** 팀장님 작업이 있더라도, 나는 트랜잭션 없이 작업을 진행합니다.
    - **`MANDATORY`:** 반드시 팀장님 작업이 있어야만 내 작업을 시작할 수 있습니다. 없으면 오류가 발생합니다.
    - **`NEVER`:** 팀장님 작업이 있으면 오류가 발생하고, 없을 때만 트랜잭션 없이 작업을 진행합니다.
    - **`NESTED`:** 팀장님 작업 안에 중첩된 형태로 작업을 시작합니다. 내 작업이 실패해도 팀장님 작업 전체가 롤백되지는 않고, 특정 지점까지만 롤백될 수 있습니다. (일부 데이터베이스에서만 지원)
  - Spring Batch의 Step은 보통 하나의 트랜잭션 단위로 실행되므로, Step을 호출하는 외부의 트랜잭션이 없다면 `REQUIRED`는 새로운 트랜잭션을 시작하게 됩니다.
3. **타임아웃 (Timeout):**
  - 트랜잭션이 지정된 시간 안에 완료되지 않으면, 시스템이 강제로 롤백시키는 시간제한입니다.
  - **비유:** 식당에서 음식을 주문했는데, 너무 오래 걸리면 "죄송하지만, 주문 취소할게요" 라고 하는 것과 비슷합니다.
  - 예상치 못하게 오래 걸리는 작업을 방지하여 시스템 자원이 고갈되는 것을 막고, 무한정 대기하는 상황을 피하기 위해 사용합니다. 단위는 보통 초(seconds)입니다.

### 주요 용어 해설

- **`Transaction Attribute` (트랜잭션 속성):** 트랜잭션의 동작 방식을 정의하는 설정값들의 묶음입니다. (격리 수준, 전파 방식, 타임아웃, 읽기 전용 여부 등)
- **`DefaultTransactionAttribute`:** Spring 프레임워크에서 제공하는 `TransactionAttribute` 인터페이스의 기본 구현 클래스입니다. 이 클래스의 객체를 생성하고 필요한 속성들을 설정하여 사용합니다.
- **`Propagation` (전파 방식):** Spring에서 트랜잭션의 경계를 어떻게 설정할지, 즉 기존 트랜잭션에 참여할지, 새로운 트랜잭션을 시작할지 등을 결정하는 규칙입니다. 예제 코드의 `Propagation.REQUIRED.value()`는 `REQUIRED` 전파 방식을 사용하겠다는 의미입니다.
- **`Isolation` (격리 수준):** 여러 트랜잭션이 동시에 데이터를 접근할 때 발생하는 문제(Dirty Read, Non-Repeatable Read, Phantom Read 등)를 방지하기 위해, 트랜잭션 간의 격리 정도를 나타내는 값입니다. 예제 코드의 `Isolation.DEFAULT.value()`는 데이터베이스의 기본 격리 수준을 사용하겠다는 의미입니다.
- **`PlatformTransactionManager`:** Spring에서 트랜잭션을 관리하는 핵심 인터페이스입니다. 실제 트랜잭션 처리는 이 인터페이스의 구현체(예: `DataSourceTransactionManager` for JDBC)가 담당합니다. Step을 정의할 때 트랜잭션 매니저를 주입받아 사용합니다.
- **`JobRepository`:** Spring Batch의 메타데이터(Job 실행 정보, Step 실행 정보 등)를 저장하고 관리하는 역할을 합니다. Step을 빌드할 때 필요합니다.
- **`StepBuilder`:** Step을 쉽게 생성할 수 있도록 도와주는 빌더 클래스입니다.
- **`chunk(int, PlatformTransactionManager)`:** Chunk 지향 처리를 설정하는 메소드입니다. 첫 번째 인자는 커밋 간격(commit interval, 청크 크기)이고, 두 번째 인자는 해당 청크 처리를 위한 트랜잭션 매니저입니다.
- **`transactionAttribute(TransactionAttribute)`:** `StepBuilder`에서 생성 중인 Step에 특정 트랜잭션 속성을 적용하는 메소드입니다.

### 코드 예제 및 분석

제공된 예제 코드를 다시 한번 자세히 살펴보겠습니다.

```java
@Bean
public Step step1(JobRepository jobRepository, PlatformTransactionManager transactionManager) {
    // 1. 트랜잭션 속성 객체 생성
	DefaultTransactionAttribute attribute = new DefaultTransactionAttribute();

    // 2. 전파 방식 설정: REQUIRED
    // 이미 진행 중인 트랜잭션이 있으면 거기에 참여하고, 없으면 새로운 트랜잭션을 시작합니다.
    // Spring Batch의 Step은 보통 자체적으로 트랜잭션을 시작하므로, 대부분의 경우 새로운 트랜잭션이 시작됩니다.
	attribute.setPropagationBehavior(Propagation.REQUIRED.value());

    // 3. 격리 수준 설정: DEFAULT
    // 사용하는 데이터베이스의 기본 격리 수준을 따릅니다.
    // (예: MySQL의 기본값은 REPEATABLE_READ, Oracle은 READ_COMMITTED)
	attribute.setIsolationLevel(Isolation.DEFAULT.value());

    // 4. 타임아웃 설정: 30초
    // 이 Step의 트랜잭션은 30초 이내에 완료되어야 합니다.
    // 그렇지 않으면 강제로 롤백됩니다.
	attribute.setTimeout(30);

    // 5. Step 빌더를 사용하여 Step 생성
	return new StepBuilder("step1", jobRepository) // Step 이름과 JobRepository 주입
				.<String, String>chunk(2, transactionManager) // 입력 타입 String, 출력 타입 String, 청크 크기 2, 트랜잭션 매니저 주입
				.reader(itemReader()) // ItemReader 설정 (여기서는 itemReader() 메소드 호출로 가정)
				.writer(itemWriter()) // ItemWriter 설정 (여기서는 itemWriter() 메소드 호출로 가정)
				.transactionAttribute(attribute) // 1~4에서 설정한 트랜잭션 속성(attribute)을 이 Step에 적용
				.build(); // Step 객체 생성 완료
}
```

**코드 분석:**

1. `DefaultTransactionAttribute attribute = new DefaultTransactionAttribute();`
  - Spring에서 제공하는 `DefaultTransactionAttribute` 클래스의 인스턴스를 만듭니다. 이 객체에 우리가 원하는 트랜잭션 설정을 담을 것입니다.
2. `attribute.setPropagationBehavior(Propagation.REQUIRED.value());`
  - `attribute` 객체의 전파 방식을 `REQUIRED`로 설정합니다.
  - `Propagation.REQUIRED`는 열거형(enum)이고, `.value()` 메소드는 해당 열거형 상수가 나타내는 정수 값을 반환합니다. Spring 내부적으로 이 정수 값을 사용하여 전파 방식을 식별합니다.
  - **왜 `REQUIRED`일까요?** 대부분의 배치 작업 Step은 독립적인 작업 단위로 트랜잭션 처리가 되어야 하기 때문에, 기존 트랜잭션이 있든 없든 해당 Step의 작업은 트랜잭션 내에서 실행되도록 보장하는 `REQUIRED`가 자주 사용됩니다.
3. `attribute.setIsolationLevel(Isolation.DEFAULT.value());`
  - `attribute` 객체의 격리 수준을 `DEFAULT`로 설정합니다.
  - `Isolation.DEFAULT` 역시 열거형이며, `.value()`를 통해 정수 값을 얻습니다.
  - **왜 `DEFAULT`일까요?** 일반적으로 데이터베이스 드라이버나 데이터 소스에 설정된 기본 격리 수준을 따르는 것이 안전하고 예측 가능하기 때문입니다. 특정 비즈니스 요구사항에 따라 명시적으로 다른 격리 수준(예: `READ_COMMITTED`, `SERIALIZABLE` 등)을 설정할 수도 있습니다.
4. `attribute.setTimeout(30);`
  - `attribute` 객체의 타임아웃을 30초로 설정합니다. 만약 이 Step에서 처리하는 청크 하나의 트랜잭션이 30초를 넘기면, Spring은 해당 트랜잭션을 롤백시키고 예외를 발생시킵니다.
  - **왜 타임아웃을 설정할까요?** 특정 작업이 예기치 않게 길어져 시스템 전체에 영향을 주는 것을 방지하기 위함입니다. 예를 들어, 외부 API 호출이 응답이 없거나 데이터베이스 락(lock) 경합으로 인해 작업이 지연될 때 유용합니다.
5. `.transactionAttribute(attribute)`
  - `StepBuilder`를 통해 Step을 구성할 때, 위에서 설정한 `attribute` 객체를 `transactionAttribute()` 메소드의 인자로 전달합니다. 이렇게 함으로써 `step1`이라는 이름의 Step은 우리가 정의한 트랜잭션 속성(전파 방식 `REQUIRED`, 격리 수준 `DEFAULT`, 타임아웃 `30초`)을 가지고 동작하게 됩니다.

### "왜?" 라는 질문에 대한 답변

**Q: 왜 트랜잭션 속성을 직접 설정해야 할까요? 기본값을 사용하면 안 되나요?**

A: 물론 많은 경우 기본값을 사용해도 괜찮습니다. 하지만, 애플리케이션의 특성이나 처리하는 데이터의 중요도, 동시성 요구 수준 등에 따라 기본값만으로는 부족한 경우가 발생합니다.

- **데이터 일관성 요구 수준이 매우 높을 때:** 예를 들어 금융 거래 처리와 같이 데이터 정합성이 극도로 중요한 경우, 기본 격리 수준보다 더 높은 격리 수준(예: `SERIALIZABLE`)을 설정하여 동시성 문제를 원천적으로 차단해야 할 수 있습니다.
- **특정 작업의 독립성 보장이 필요할 때:** 위에서 언급한 로그 기록처럼, 주 작업의 성공/실패와 관계없이 항상 실행되어야 하는 작업은 `REQUIRES_NEW` 전파 방식을 사용하여 독립적인 트랜잭션으로 처리해야 합니다.
- **성능 최적화가 필요할 때:** 때로는 격리 수준을 낮추거나(주의해서 사용해야 함), 불필요한 트랜잭션 전파를 막음으로써 성능을 개선할 수도 있습니다.
- **특정 작업이 오래 걸리는 것을 방지하고 싶을 때:** 타임아웃 설정을 통해 특정 Step이나 청크 처리가 시스템 전체를 느리게 만드는 것을 방지할 수 있습니다.

결국, 트랜잭션 속성을 세밀하게 제어할 수 있다는 것은 개발자에게 **더 많은 통제권**을 주고, **다양한 상황에 유연하게 대처**할 수 있도록 해줍니다.

### 주의사항 및 Best Practice

- **격리 수준은 신중하게 선택하세요:** 격리 수준을 너무 낮게 설정하면 데이터 부정합 문제가 발생할 수 있고, 너무 높게 설정하면 동시성이 떨어져 성능 저하를 유발할 수 있습니다. 애플리케이션의 요구사항과 데이터베이스의 특성을 고려하여 적절한 수준을 선택해야 합니다.
- **`REQUIRES_NEW`는 주의해서 사용하세요:** `REQUIRES_NEW`는 새로운 커넥션을 사용하게 될 수 있으며, 너무 자주 사용하면 커넥션 풀 고갈의 원인이 될 수 있습니다. 꼭 필요한 경우에만 사용하는 것이 좋습니다.
- **타임아웃 값은 적절하게 설정하세요:** 너무 짧은 타임아웃은 정상적인 작업도 실패하게 만들 수 있고, 너무 긴 타임아웃은 문제 상황을 늦게 인지하게 만듭니다. 평균 작업 시간과 예상되는 최대 작업 시간을 고려하여 설정하세요.
- **Spring Core 문서 참고:** 트랜잭션 속성에 대한 더 깊이 있는 이해를 위해서는 Spring Core 문서의 트랜잭션 관리 부분을 함께 학습하는 것이 매우 도움이 됩니다. 이 챕터의 원문에서도 언급되었죠.
- **테스트는 필수입니다:** 트랜잭션 관련 설정 변경은 애플리케이션의 동작에 큰 영향을 미칠 수 있으므로, 변경 후에는 반드시 충분한 테스트를 통해 의도한 대로 동작하는지 확인해야 합니다. 특히 동시성 환경에서의 테스트가 중요합니다.
- **XML 설정 방식:** 이 챕터에서는 Java 설정 예제만 보여줬지만, XML을 사용하여 배치 Job을 설정하는 경우에도 유사하게 트랜잭션 속성을 정의할 수 있습니다. `<tx:attributes>` 와 같은 태그를 사용하게 됩니다.
