---
title: Spring Batch - Configuring a Step (Late Binding of Job and Step Attributes)
description: 
author: laze
date: 2025-06-09 00:00:02 +0900
categories: [Dev, SpringBatch]
tags: [SpringBatch]
---
### 잡(Job) 및 스텝(Step) 속성의 지연 바인딩 (Late Binding of Job and Step Attributes)

앞서 보여드린 XML 및 플랫 파일 예제들은 파일을 얻기 위해 스프링의 `Resource` 추상화를 사용합니다.

이는 `Resource`가 `java.io.File`을 반환하는 `getFile` 메서드를 가지고 있기 때문에 작동합니다.

표준 스프링 구문을 사용하여 XML 및 플랫 파일 리소스를 모두 구성할 수 있습니다:

다음 예제는 Java에서의 지연 바인딩을 보여줍니다:

**Java 설정 (Java Configuration)**

```java
@Bean
public FlatFileItemReader flatFileItemReader() {
	FlatFileItemReader<Foo> reader = new FlatFileItemReaderBuilder<Foo>()
			.name("flatFileItemReader")
			.resource(new FileSystemResource("file://outputs/file.txt")) // 파일 시스템 경로를 하드코딩
			// ... (나머지 설정)
}
```

위의 `Resource`는 지정된 파일 시스템 위치에서 파일을 로드합니다.

절대 위치는 이중 슬래시(//)로 시작해야 함에 유의하십시오.

대부분의 스프링 애플리케이션에서는 이러한 리소스의 이름이 컴파일 타임에 알려져 있기 때문에 이 솔루션으로 충분합니다.

그러나 배치 시나리오에서는 파일 이름이 잡(Job)의 매개변수로서 런타임에 결정되어야 할 수 있습니다.

이는 시스템 속성을 읽기 위해 `-D` 매개변수를 사용하여 해결할 수 있습니다.

다음은 Java에서 속성(property)으로부터 파일 이름을 읽는 방법을 보여줍니다:

**Java 설정 (Java Configuration)**

```java
@Bean
public FlatFileItemReader flatFileItemReader(@Value("${input.file.name}") String name) { // 시스템 프로퍼티 또는 프로퍼티 파일 값 주입
	return new FlatFileItemReaderBuilder<Foo>()
			.name("flatFileItemReader")
			.resource(new FileSystemResource(name)) // 주입받은 이름으로 리소스 생성
			// ... (나머지 설정)
}
```

이 솔루션이 작동하기 위해 필요한 것은 시스템 인자(예: `-Dinput.file.name="file://outputs/file.txt"`)뿐입니다.

여기서 `PropertyPlaceholderConfigurer`를 사용할 수 있지만, 시스템 속성이 항상 설정되어 있다면 스프링의 `ResourceEditor`가 이미 시스템 속성에 대한 필터링 및 플레이스홀더 교체를 수행하므로 필수는 아닙니다.

배치 환경에서는 종종 (시스템 속성을 통하는 대신) 잡(Job)의 `JobParameters`에 파일 이름을 매개변수화하고 그 방식으로 접근하는 것이 선호됩니다.

이를 위해 Spring Batch는 다양한 잡(Job) 및 스텝(Step) 속성의 지연 바인딩을 허용합니다.

다음 예제는 Java에서 파일 이름을 매개변수화하는 방법을 보여줍니다:

**Java 설정 (Java Configuration)**

```java
@StepScope // 스텝 스코프 지정
@Bean
public FlatFileItemReader flatFileItemReader(@Value("#{jobParameters['input.file.name']}") String name) { // SpEL을 사용하여 JobParameters 값 주입
	return new FlatFileItemReaderBuilder<Foo>()
			.name("flatFileItemReader")
			.resource(new FileSystemResource(name))
			// ... (나머지 설정)
}
```

`JobExecution` 및 `StepExecution` 수준의 `ExecutionContext`에도 동일한 방식으로 접근할 수 있습니다.

다음 예제는 Java에서 `ExecutionContext`에 접근하는 방법을 보여줍니다:

**Java 설정 (Java Configuration)**

```java
@StepScope
@Bean
public FlatFileItemReader flatFileItemReader(@Value("#{jobExecutionContext['input.file.name']}") String name) { // JobExecutionContext 값 주입
	return new FlatFileItemReaderBuilder<Foo>()
			.name("flatFileItemReader")
			.resource(new FileSystemResource(name))
			// ... (나머지 설정)
}
```

**Java 설정 (Java Configuration)**

```java
@StepScope
@Bean
public FlatFileItemReader flatFileItemReader(@Value("#{stepExecutionContext['input.file.name']}") String name) { // StepExecutionContext 값 주입
	return new FlatFileItemReaderBuilder<Foo>()
			.name("flatFileItemReader")
			.resource(new FileSystemResource(name))
			// ... (나머지 설정)
}
```

지연 바인딩을 사용하는 모든 빈은 `scope="step"`으로 선언되어야 합니다. 자세한 내용은 [Step Scope](https://www.notion.so/Late-Binding-of-Job-and-Step-Attributes-20d3df01028f80cfacd4c58c7d213477?pvs=21)를 참조하십시오. 스텝(Step) 빈 자체는 스텝 스코프 또는 잡 스코프가 되어서는 안 됩니다.

스텝 정의에서 지연 바인딩이 필요한 경우, 해당 스텝의 구성 요소(태스크릿, 아이템 리더/라이터, 완료 정책 등)가 대신 스코프 지정되어야 합니다.

Spring 3.0 (또는 그 이상)을 사용하는 경우, 스텝 스코프 빈의 표현식은 강력하고 다양한 기능을 가진 범용 언어인 Spring Expression Language(SpEL)로 작성됩니다. 하

위 호환성을 제공하기 위해, Spring Batch가 이전 버전의 Spring을 감지하면 덜 강력하고 약간 다른 파싱 규칙을 가진 네이티브 표현 언어를 사용합니다.

주요 차이점은 위 예제의 맵 키가 Spring 2.5에서는 따옴표로 묶을 필요가 없지만, Spring 3.0에서는 따옴표가 필수라는 것입니다.

### 스텝 스코프 (Step Scope)

앞서 보여드린 모든 지연 바인딩 예제는 빈 정의에 `step` 스코프가 선언되어 있습니다.

다음 예제는 Java에서 스텝 스코프에 바인딩하는 예를 보여줍니다:

**Java 설정 (Java Configuration)**

```java
@StepScope
@Bean
public FlatFileItemReader flatFileItemReader(@Value("#{jobParameters[input.file.name]}") String name) {
	return new FlatFileItemReaderBuilder<Foo>()
			.name("flatFileItemReader")
			.resource(new FileSystemResource(name))
			// ... (나머지 설정)
}
```

지연 바인딩을 사용하려면 `Step` 스코프를 사용하는 것이 필수입니다. 왜냐하면 속성을 찾을 수 있도록 스텝(Step)이 시작될 때까지 빈이 실제로 인스턴스화될 수 없기 때문입니다.

기본적으로 스프링 컨테이너의 일부가 아니므로, 배치 네임스페이스를 사용하거나, `StepScope`에 대한 빈 정의를 명시적으로 포함하거나,

`@EnableBatchProcessing` 어노테이션을 사용하여 스코프를 명시적으로 추가해야 합니다.

이 방법들 중 하나만 사용하십시오.

다음 예제는 배치 네임스페이스를 사용합니다:

```xml
<beans xmlns="<http://www.springframework.org/schema/beans>"
       xmlns:batch="<http://www.springframework.org/schema/batch>"
       xmlns:xsi="<http://www.w3.org/2001/XMLSchema-instance>"
       xsi:schemaLocation="...">
<batch:job .../>
...
</beans>
```

다음 예제는 빈 정의를 명시적으로 포함합니다:

```xml
<bean class="org.springframework.batch.core.scope.StepScope" />
```

### 잡 스코프 (Job Scope)

Spring Batch 3.0에 도입된 잡 스코프는 구성 면에서는 스텝 스코프와 유사하지만 잡 컨텍스트에 대한 스코프이므로 실행 중인 잡당 이러한 빈의 인스턴스는 하나만 존재합니다.

또한, `#{..}` 플레이스홀더를 사용하여 `JobContext`에서 접근 가능한 참조의 지연 바인딩에 대한 지원이 제공됩니다.

이 기능을 사용하면 잡 또는 잡 실행 컨텍스트 및 잡 매개변수에서 빈 속성을 가져올 수 있습니다.

다음 예제는 Java에서 잡 스코프에 바인딩하는 예를 보여줍니다:

**Java 설정 (Java Configuration)**

```java
@JobScope
@Bean
public FlatFileItemReader flatFileItemReader(@Value("#{jobParameters[input]}") String name) {
	return new FlatFileItemReaderBuilder<Foo>()
			.name("flatFileItemReader")
			.resource(new FileSystemResource(name))
			// ... (나머지 설정)
}
```

**Java 설정 (Java Configuration)**

```java
@JobScope
@Bean
public FlatFileItemReader flatFileItemReader(@Value("#{jobExecutionContext['input.name']}") String name) {
	return new FlatFileItemReaderBuilder<Foo>()
			.name("flatFileItemReader")
			.resource(new FileSystemResource(name))
			// ... (나머지 설정)
}
```

기본적으로 스프링 컨테이너의 일부가 아니므로, 배치 네임스페이스를 사용하거나, `JobScope`에 대한 빈 정의를 명시적으로 포함하거나,

`@EnableBatchProcessing` 어노테이션을 사용하여 스코프를 명시적으로 추가해야 합니다 (하나의 접근 방식만 선택). 다음 예제는 배치 네임스페이스를 사용합니다:

```xml
<beans xmlns="<http://www.springframework.org/schema/beans>"
		  xmlns:batch="<http://www.springframework.org/schema/batch>"
		  xmlns:xsi="<http://www.w3.org/2001/XMLSchema-instance>"
		  xsi:schemaLocation="...">

<batch:job .../>
...
</beans>
```

다음 예제는 `JobScope`를 명시적으로 정의하는 빈을 포함합니다:

```xml
<bean class="org.springframework.batch.core.scope.JobScope" />
```

멀티스레드 또는 파티션된 스텝에서 잡 스코프 빈을 사용하는 데에는 몇 가지 실질적인 제한 사항이 있습니다.

Spring Batch는 이러한 사용 사례에서 생성된 스레드를 제어하지 않으므로 이러한 빈을 올바르게 사용하도록 설정할 수 없습니다.

따라서 멀티스레드 또는 파티션된 스텝에서 잡 스코프 빈을 사용하는 것은 권장하지 않습니다.

### ItemStream 구성 요소 스코핑 (Scoping ItemStream components)

잡 또는 스텝 스코프 `ItemStream` 빈을 정의하기 위해 Java 구성 스타일을 사용할 때, 빈 정의 메서드의 반환 타입은 최소한 `ItemStream`이어야 합니다.

이는 Spring Batch가 이 인터페이스를 구현하는 프록시를 올바르게 생성하고, 따라서 `open`, `update`, `close` 메서드를 예상대로 호출하여 계약을 준수하도록 하기 위해 필요합니다.

다음 예제와 같이 이러한 빈의 빈 정의 메서드가 가장 구체적으로 알려진 구현을 반환하도록 하는 것이 좋습니다:

가장 구체적인 반환 타입으로 스텝 스코프 빈 정의하기

```java
@Bean
@StepScope
public FlatFileItemReader flatFileItemReader(@Value("#{jobParameters['input.file.name']}") String name) {
	return new FlatFileItemReaderBuilder<Foo>()
			.resource(new FileSystemResource(name))
			// 아이템 리더의 다른 속성 설정
			.build();
}
```

---

### **학습 목표 제시**

이번 "Late Binding of Job and Step Attributes" 챕터를 통해 학생분은 다음을 이해하고 배울 수 있습니다:

1. **지연 바인딩(Late Binding)의 개념 및 필요성 이해:** 배치 잡 실행 시점에 동적으로 결정되는 값(예: 파일 경로, 파라미터)을 어떻게 스텝의 속성에 주입하여 유연성을 높이는지 이해합니다.
2. **`@StepScope` 및 `@JobScope`의 활용법 학습:** 스텝 또는 잡의 생명주기에 맞춰 빈(Bean)을 생성하고, `JobParameters`나 `ExecutionContext`의 값을 스프링 표현 언어(SpEL)를 통해 주입받는 방법을 익힙니다.
3. **스코프 빈 사용 시 주의사항 파악:** 스코프 빈, 특히 `ItemStream`을 구현하는 빈을 정의할 때의 주의사항과 권장 사항을 이해합니다.

---

### **핵심 개념 설명**

이 챕터의 핵심은 **"컴파일 시점이 아닌, 실행 시점에 결정되는 값을 어떻게 배치 컴포넌트(주로 ItemReader, ItemWriter 등)에 전달하고 사용할 것인가?"** 입니다. 이를 "지연 바인딩(Late Binding)"이라고 부릅니다.

### 1. 지연 바인딩(Late Binding)이란?

- **개념:** 프로그램이 컴파일될 때 값이 정해지는 것이 아니라, 프로그램이 실제로 실행될 때(런타임에) 값이 결정되어 바인딩(연결)되는 방식입니다.
- **비유:** 요리 레시피에 "오늘의 신선한 채소"라고 적혀있다고 상상해보세요. 레시피를 작성할 때는 어떤 채소가 될지 모르지만, 실제로 요리하는 날 시장에 가서 가장 신선한 채소를 골라 사용하는 것과 같습니다. 여기서 "오늘의 신선한 채소"라는 정보가 지연 바인딩되는 값입니다.
- **Spring Batch에서의 의미:** 배치 잡을 실행할 때마다 다른 입력 파일을 처리하거나, 다른 파라미터를 사용해야 하는 경우가 많습니다. 이때, 파일 경로 같은 정보가 코드에 고정되어 있다면 매번 코드를 수정하고 재배포해야 하는 번거로움이 있습니다. 지연 바인딩을 사용하면 이런 값들을 잡 실행 시점에 외부에서 주입받아 사용할 수 있게 됩니다.
- **"왜?"**:
  - **유연성 향상:** 동일한 배치 잡 로직을 다양한 입력 데이터나 조건에 따라 재사용할 수 있게 됩니다.
  - **재배포 감소:** 설정값 변경을 위해 코드를 수정하고 시스템을 재배포하는 번거로움을 줄일 수 있습니다.
  - **운영 용이성:** 외부 설정(예: 잡 파라미터)을 통해 배치 잡의 동작을 제어하기 쉬워집니다.

### 2. 지연 바인딩 방법들

문서에서는 주로 세 가지 방법을 언급하고, 그중 Spring Batch에서 권장하는 방식을 설명합니다.

- **방법 1: 고정된 값 사용 (Not Late Binding)**
  - `new FileSystemResource("file://outputs/file.txt")`처럼 코드에 파일 경로를 직접 하드코딩하는 방식입니다. 가장 간단하지만 유연성이 떨어집니다.
- **방법 2: 시스템 속성(System Properties) 또는 프로퍼티 파일 사용**
  - `@Value("${input.file.name}")` 와 같이 스프링의 `@Value` 어노테이션과 프로퍼티 플레이스홀더(`$ {...}`)를 사용하여 시스템 속성이나 `.properties` 파일에서 값을 읽어옵니다.
  - 실행 시 `Dinput.file.name=경로` 와 같이 JVM 옵션으로 값을 전달할 수 있습니다.
  - 일반 스프링 애플리케이션에서는 흔히 사용되지만, 배치 잡의 맥락에서는 `JobParameters`를 사용하는 것이 더 권장됩니다.
  - **"왜 JobParameters가 더 선호될까?"**: `JobParameters`는 각 잡 실행(JobInstance)을 고유하게 식별하는 데 사용되며, Spring Batch의 메타데이터 관리(재시작, 실행 이력 추적 등)와 밀접하게 연관되어 있습니다. 잡 실행과 관련된 파라미터를 `JobParameters`로 관리하면 이러한 프레임워크의 기능을 온전히 활용할 수 있습니다.
- **방법 3: `JobParameters` 및 `ExecutionContext`와 `@StepScope` / `@JobScope` 사용 (Spring Batch 권장 방식)**
  - 이것이 이 챕터의 핵심입니다!
  - `@Value("#{jobParameters['input.file.name']}")` 또는 `@Value("#{jobExecutionContext['some.key']}")` 와 같이 스프링 표현 언어(Spring Expression Language, SpEL - `#{...}`)를 사용합니다.
  - 이 방식을 사용하려면 해당 빈(Bean)이 `@StepScope` 또는 `@JobScope`로 선언되어야 합니다.

### 3. `@StepScope` 와 지연 바인딩

- **개념:** `@StepScope`는 빈의 생명주기를 해당 빈이 사용되는 **스텝(Step)의 실행 주기**에 맞추는 특별한 스코프입니다.
  - 즉, 스텝이 시작될 때 빈 인스턴스가 생성되고, 스텝이 종료될 때 소멸(또는 소멸 대기)됩니다.
  - 각 스텝 실행마다 새로운 인스턴스가 생성될 수 있습니다 (스텝이 여러 번 실행되는 경우, 예: 재시작).
- **동작 원리:**
  1. 스프링 컨테이너는 `@StepScope` 빈을 직접 생성하는 대신, **프록시(Proxy)** 객체를 먼저 생성하여 주입합니다.
  2. 실제로 해당 스텝이 실행되어 이 빈의 메서드가 호출될 때, 프록시는 현재 스텝 컨텍스트에서 실제 빈 인스턴스를 찾거나 생성합니다.
  3. 이때, `@Value("#{jobParameters['...']}")` 와 같은 SpEL 표현식이 평가되어 실제 런타임 값이 주입됩니다.
- **"왜 `@StepScope`가 필요한가?"**:
  - `JobParameters`나 `StepExecutionContext`의 값은 스텝이 시작되는 시점에야 확정됩니다. 일반적인 싱글톤 빈은 애플리케이션 컨텍스트 로딩 시점에 생성되므로, 이 시점에는 아직 해당 값들을 알 수 없습니다.
  - `@StepScope`를 사용하면 빈의 생성을 스텝 시작 시점까지 "지연"시켜서, 런타임 값을 안전하게 주입받을 수 있게 됩니다.
- **사용 대상:** 주로 `ItemReader`, `ItemWriter`, `ItemProcessor`, `Tasklet` 등 스텝 내에서 실행 시점의 데이터가 필요한 컴포넌트들에 사용됩니다.
- **주의!**: **`Step` 빈 자체는 `@StepScope`나 `@JobScope`로 만들면 안 됩니다.** `Step` 빈은 잡 정의의 일부로, 잡이 로드될 때 구성되어야 합니다. 지연 바인딩은 `Step`을 구성하는 *내부 컴포넌트들*에 적용합니다.

### 4. `@JobScope` 와 지연 바인딩

- **개념:** `@JobScope`는 빈의 생명주기를 해당 빈이 사용되는 **잡(Job)의 실행 주기**에 맞추는 스코프입니다.
  - 즉, 잡이 시작될 때 빈 인스턴스가 생성되고, 잡이 종료될 때 소멸됩니다.
  - 하나의 잡 실행(JobExecution) 내에서는 해당 빈의 인스턴스가 유일하게 존재합니다.
- **동작 원리:** `@StepScope`와 유사하게 프록시를 통해 지연 생성 및 값 주입이 이루어집니다. 다만, 컨텍스트가 스텝이 아닌 잡 전체입니다.
- **"왜 `@JobScope`가 필요한가?"**:
  - 스텝 간에 공유되어야 하지만, 잡 실행마다 값이 달라질 수 있는 컴포넌트에 유용합니다.
  - 예를 들어, 잡 전체에서 사용될 특정 설정값이나 리소스를 `JobParameters`로부터 받아와 잡 실행 동안 유지하고 싶을 때 사용합니다.
- **사용 대상:** 여러 스텝에서 공통으로 사용되지만 잡 실행 시점에 초기화되어야 하는 리소스 관리자, 설정 객체 등에 사용될 수 있습니다.
- **주의!**: 멀티스레드 또는 파티셔닝된 스텝에서는 `JobScope` 빈 사용을 권장하지 않습니다. Spring Batch가 해당 스레드들을 직접 제어하지 않아 스코프 빈의 정확한 동작을 보장하기 어렵기 때문입니다. 이러한 경우에는 `StepScope`를 사용하거나 다른 동기화 메커니즘을 고려해야 합니다.

### 5. 스프링 표현 언어(SpEL) 활용

- `@Value` 어노테이션과 함께 사용되는 `#{ ... }` 구문이 바로 SpEL입니다.
- **접근 가능한 컨텍스트 객체:**
  - `jobParameters['파라미터명']`: 현재 잡 실행의 `JobParameters`에 접근합니다.
  - `jobExecutionContext['키명']`: 현재 잡의 `JobExecutionContext`에 접근합니다. 잡 전체에 걸쳐 데이터를 공유할 때 사용됩니다.
  - `stepExecutionContext['키명']`: 현재 스텝의 `StepExecutionContext`에 접근합니다. 해당 스텝 내에서 또는 스텝 재시작 시 데이터를 공유할 때 사용됩니다.
- **따옴표 사용 규칙:** SpEL에서 맵의 키(예: `jobParameters`의 키)를 참조할 때는 일반적으로 따옴표로 감싸줍니다 (예: `jobParameters['input.file.name']`). Spring 2.5에서는 따옴표가 필수가 아니었지만, Spring 3.0 이상에서는 필수입니다. (현재는 대부분 Spring 3.0 이상을 사용하므로 따옴표를 사용하는 것이 표준입니다.)

### 6. `@StepScope` / `@JobScope` 등록

- 이 스코프들은 스프링 컨테이너에 기본적으로 등록되어 있지 않습니다. 따라서 명시적으로 추가해주어야 합니다.
- **방법 (셋 중 하나만 선택):**
  1. **`@EnableBatchProcessing` 사용 (Java Config 권장):** `@Configuration` 클래스에 `@EnableBatchProcessing` 어노테이션을 추가하면 `StepScope`와 `JobScope`가 자동으로 등록됩니다. 가장 간편한 방법입니다.
  2. **XML 설정에서 `<batch:job>` 등 사용:** XML 설정에서 Spring Batch 네임스페이스 (`xmlns:batch="<http://www.springframework.org/schema/batch>"`)를 사용하고 `<batch:job>` 같은 요소를 사용하면 관련 스코프가 등록됩니다.
  3. **스코프 빈 직접 등록 (XML):**

      ```xml
      <bean class="org.springframework.batch.core.scope.StepScope" />
      <bean class="org.springframework.batch.core.scope.JobScope" />
      ```


### 7. `ItemStream` 컴포넌트 스코핑 시 주의사항

- `ItemReader`, `ItemWriter` 등이 `ItemStream` 인터페이스를 구현하는 경우가 많습니다 (파일 처리, 데이터베이스 커서 사용 등). `ItemStream`은 `open()`, `update()`, `close()` 같은 생명주기 메서드를 가집니다.
- `@StepScope`나 `@JobScope`로 `ItemStream` 구현 빈을 정의할 때, 스프링은 이 빈의 프록시를 생성합니다. 이때 Spring Batch가 `open`, `update`, `close` 메서드를 올바르게 호출하도록 하려면, **빈 정의 메서드의 반환 타입을 최소한 `ItemStream` 인터페이스로 하거나, 가능하다면 가장 구체적인 구현 클래스 타입으로 지정**하는 것이 좋습니다.
  - **예시 (권장):**
    여기서 반환 타입이 `FlatFileItemReader` (구체적인 클래스)입니다. 이렇게 하면 스프링이 생성하는 프록시가 `FlatFileItemReader`의 모든 메서드(당연히 `ItemStream`의 메서드 포함)를 제대로 위임할 수 있습니다. 만약 반환 타입을 `Object`나 너무 일반적인 타입으로 하면, 프록시가 `ItemStream`의 생명주기 메서드를 인지하지 못해 호출되지 않을 수 있습니다.

      ```java
      @Bean
      @StepScope
      public FlatFileItemReader<MyData> myReader(@Value("#{jobParameters['inputFile']}") String filePath) {
          return new FlatFileItemReaderBuilder<MyData>()
                     .name("myReader")
                     .resource(new FileSystemResource(filePath))
                     // ...
                     .build();
      }
      ```


---

### **주요 용어 해설**

- **지연 바인딩 (Late Binding):** 프로그램 실행 시점에 값이 결정되어 속성에 연결되는 방식.
- **`@StepScope`:** 빈의 생명주기를 스텝 실행 주기에 맞추는 스프링 배치 커스텀 스코프. 스텝 시작 시 빈 생성, 스텝 종료 시 소멸.
- **`@JobScope`:** 빈의 생명주기를 잡 실행 주기에 맞추는 스프링 배치 커스텀 스코프. 잡 시작 시 빈 생성, 잡 종료 시 소멸.
- **Spring Expression Language (SpEL):** 스프링에서 제공하는 강력한 표현 언어. `@Value` 어노테이션과 함께 `#{ ... }` 구문으로 사용되어 런타임에 객체 그래프를 조회하고 조작할 수 있게 함.
- **`JobParameters`:** 잡 인스턴스를 시작할 때 전달되는 읽기 전용 파라미터 집합. 각 잡 인스턴스를 고유하게 식별하는 데도 사용됨.
- **`ExecutionContext`:** 잡 또는 스텝 실행 중에 상태를 저장하고 공유하기 위한 키-값 저장소. `JobExecutionContext` (잡 전체)와 `StepExecutionContext` (해당 스텝)가 있음.
- **프록시 (Proxy):** `@StepScope`나 `@JobScope` 빈의 경우, 실제 빈 인스턴스 대신 먼저 주입되는 대리 객체. 실제 메서드 호출 시점에 실제 빈을 찾아 연결해줌.
- **`ItemStream`:** 배치 처리 중 스트림(파일, 데이터베이스 커서 등)을 열고, 상태를 업데이트하고, 닫는 생명주기 관리가 필요한 컴포넌트를 위한 인터페이스. `open()`, `update()`, `close()` 메서드를 가짐.

---

### **코드 예제 및 분석**

### 1. `@StepScope`와 `jobParameters` 사용 예시

```java
@Configuration
@EnableBatchProcessing // StepScope, JobScope 자동 등록
public class MyJobConfig {

    @Bean
    @StepScope // 이 빈은 스텝 실행 시점에 생성됨
    public FlatFileItemReader<Transaction> transactionReader(
            @Value("#{jobParameters['input.file.path']}") String inputFilePath) { // SpEL을 통해 JobParameter 값 주입

        // inputFilePath는 잡 실행 시점에 JobParameters에 의해 결정된 실제 파일 경로를 가짐
        if (inputFilePath == null) {
            throw new IllegalArgumentException("input.file.path job parameter is required.");
        }

        return new FlatFileItemReaderBuilder<Transaction>()
                .name("transactionReader") // ItemReader 이름
                .resource(new FileSystemResource(inputFilePath)) // 주입받은 경로로 리소스 생성
                .delimited() // 구분자 기반 파일 형식 지정
                .names(new String[]{"transactionId", "amount", "timestamp"}) // 필드명
                .targetType(Transaction.class) // 매핑될 객체 타입
                .build();
    }

    @Bean
    public Step myStep(JobRepository jobRepository, PlatformTransactionManager transactionManager,
                       FlatFileItemReader<Transaction> reader, // transactionReader 빈 주입 (프록시가 주입됨)
                       ItemWriter<Transaction> writer) {
        return new StepBuilder("myStep", jobRepository)
                .<Transaction, Transaction>chunk(10, transactionManager)
                .reader(reader) // 스텝 실행 시, reader 프록시가 실제 인스턴스를 찾아 메서드 호출
                .writer(writer)
                .build();
    }

    @Bean
    public Job myJob(JobRepository jobRepository, Step myStep) {
        return new JobBuilder("myJob", jobRepository)
                .start(myStep)
                .build();
    }
}

// Transaction.java (예시 데이터 객체)
// public class Transaction {
//    private String transactionId;
//    private BigDecimal amount;
//    private String timestamp;
//    // getters and setters
// }

```

- **`@EnableBatchProcessing`**: `StepScope`와 `JobScope`를 사용할 수 있도록 해줍니다.
- **`@StepScope` on `transactionReader`**: `transactionReader` 빈은 각 스텝 실행 시 새로 생성(또는 컨텍스트에서 조회)됩니다.
- **`@Value("#{jobParameters['input.file.path']}") String inputFilePath`**:
  - `transactionReader` 빈이 생성될 때 (즉, `myStep`이 시작될 때), `JobParameters`에서 'input.file.path'라는 키로 전달된 값을 찾아 `inputFilePath` 문자열에 주입합니다.
  - 예를 들어, 잡을 실행할 때 `java -jar myapp.jar input.file.path=data/transactions_today.csv` 와 같이 파라미터를 전달하면, `inputFilePath`는 "data/transactions\_today.csv"가 됩니다.
- **`myStep`에서 `reader` 주입**: `myStep` 빈이 생성될 때는 `transactionReader`의 프록시 객체가 주입됩니다. 실제로 `myStep`이 실행되어 `reader.read()`가 호출될 때, 프록시가 현재 스텝 컨텍스트에 맞는 실제 `FlatFileItemReader` 인스턴스를 사용하게 됩니다.

### 2. `@JobScope`와 `jobExecutionContext` 사용 예시 (가상 시나리오)

```java
// 잡 실행 초기에 어떤 설정값을 JobExecutionContext에 저장하고,
// 이후 다른 스텝에서 이 값을 사용하는 시나리오를 가정해봅니다.

// 먼저 JobExecutionContext에 값을 쓰는 리스너 (예시)
public class SetupJobExecutionListener extends JobExecutionListenerSupport {
    @Override
    public void beforeJob(JobExecution jobExecution) {
        // 실제로는 JobParameters나 다른 로직을 통해 값을 결정할 수 있음
        jobExecution.getExecutionContext().putString("shared.config.value", "DynamicValueFromJobStart-" + jobExecution.getJobId());
        System.out.println("JobExecutionContext에 'shared.config.value' 설정됨.");
    }
}

@Configuration
@EnableBatchProcessing
public class AnotherJobConfig {

    @Bean
    public SetupJobExecutionListener setupListener() {
        return new SetupJobExecutionListener();
    }

    @Bean
    @JobScope // 이 빈은 잡 실행 시점에 생성됨
    public MyJobScopedService myJobScopedService(
            @Value("#{jobExecutionContext['shared.config.value']}") String sharedValue) { // SpEL을 통해 JobExecutionContext 값 주입
        System.out.println("MyJobScopedService 생성됨. sharedValue: " + sharedValue);
        return new MyJobScopedService(sharedValue);
    }

    @Bean
    public Step firstStepInJob(JobRepository jobRepository, PlatformTransactionManager transactionManager,
                             MyJobScopedService service) { // myJobScopedService 빈 주입 (프록시)
        return new StepBuilder("firstStepInJob", jobRepository)
                .tasklet((contribution, chunkContext) -> {
                    System.out.println("firstStepInJob 실행 중... MyJobScopedService 사용.");
                    service.doSomething(); // 서비스 사용
                    return RepeatStatus.FINISHED;
                }, transactionManager)
                .build();
    }

    // ... (다른 스텝에서 myJobScopedService를 또 주입받아 사용할 수 있음, 동일 인스턴스)

    @Bean
    public Job anotherJob(JobRepository jobRepository, Step firstStepInJob, SetupJobExecutionListener setupListener) {
        return new JobBuilder("anotherJob", jobRepository)
                .listener(setupListener) // 리스너 등록
                .start(firstStepInJob)
                // .next(anotherStepThatUsesMyJobScopedService)
                .build();
    }
}

// MyJobScopedService.java (예시 서비스)
// public class MyJobScopedService {
//    private final String configValue;
//    public MyJobScopedService(String configValue) {
//        this.configValue = configValue;
//    }
//    public void doSomething() {
//        System.out.println("MyJobScopedService가 작업 수행 중, 설정값: " + configValue);
//    }
// }
```

- **`SetupJobExecutionListener`**: 잡 시작 전에 `JobExecutionContext`에 "shared.config.value"라는 키로 값을 저장합니다.
- **`@JobScope` on `myJobScopedService`**: `myJobScopedService` 빈은 잡 실행 당 하나의 인스턴스만 생성됩니다.
- **`@Value("#{jobExecutionContext['shared.config.value']}") String sharedValue`**:
  - `myJobScopedService` 빈이 생성될 때 (즉, 잡이 시작될 때), `JobExecutionContext`에서 "shared.config.value" 키의 값을 찾아 `sharedValue`에 주입합니다.
- 이 `myJobScopedService` 빈은 `anotherJob` 내의 여러 스텝에서 주입받아 사용될 수 있으며, 모든 스텝은 동일한 `myJobScopedService` 인스턴스를 참조하게 됩니다.

---

### **"왜?" 라는 질문에 대한 답변**

- **왜 지연 바인딩이 배치에서 특히 중요한가?**
  - 배치 잡은 주로 정기적으로, 또는 필요에 따라 반복 실행됩니다. 매번 다른 입력 파일, 다른 날짜 범위, 다른 기준값 등을 사용해야 하는 경우가 빈번합니다. 이런 동적인 요구사항을 코드 변경 없이 외부 설정(주로 `JobParameters`)으로 처리할 수 있게 해 주기 때문에 매우 중요합니다. 이는 잡의 재사용성과 운영 효율성을 크게 높여줍니다.
- **왜 `@StepScope`나 `@JobScope`를 써야만 `jobParameters` 등을 SpEL로 주입받을 수 있는가?**
  - `jobParameters`나 `ExecutionContext`의 값들은 잡이나 스텝이 **실제로 실행되는 시점**에야 의미가 있고 접근 가능합니다. 일반적인 스프링 빈(싱글톤)은 애플리케이션 시작 시점에 한 번만 생성되는데, 이때는 아직 어떤 잡이 어떤 파라미터로 실행될지 알 수 없습니다.
  - `@StepScope`와 `@JobScope`는 빈의 생성을 해당 스코프(스텝 또는 잡)가 시작되는 시점까지 "지연"시킵니다. 그리고 스프링은 이 스코프 빈들을 위한 특별한 프록시 메커니즘을 제공하여, 실제 빈이 필요할 때 (즉, 메서드가 호출될 때) 해당 컨텍스트(현재 실행 중인 스텝 또는 잡)에 맞는 실제 인스턴스를 생성하거나 찾아내고, 이때 SpEL을 통해 `jobParameters` 등의 값을 주입합니다. 이 "지연"과 "프록시"가 핵심입니다.

---

### **주의사항 및 Best Practice**

1. **스코프 등록 확인:** `@EnableBatchProcessing`을 사용하거나 XML에 스코프 빈을 명시적으로 등록해야 `@StepScope`, `@JobScope`가 동작합니다. 잊기 쉬운 부분이니 주의하세요.
2. **`Step` 빈 자체는 스코프 지정 금지:** `Step`을 정의하는 `@Bean` 메서드에는 `@StepScope`나 `@JobScope`를 붙이지 마세요. 스텝의 *내부 컴포넌트*(Reader, Writer, Processor 등)에 적용합니다.
3. **`ItemStream` 반환 타입:** `@StepScope` 또는 `@JobScope`로 지정된 `ItemStream` 구현 빈의 `@Bean` 메서드 반환 타입은 최소 `ItemStream`이거나, 더 구체적인 해당 클래스 타입으로 지정해야 `open/update/close`가 올바르게 호출됩니다.
4. **`@JobScope`와 멀티스레딩:** 멀티스레드 스텝이나 파티셔닝 스텝에서는 `@JobScope` 빈 사용 시 동시성 문제가 발생할 수 있으므로 권장되지 않습니다. 각 스레드가 동일한 `@JobScope` 빈 인스턴스를 공유하려 할 때 문제가 생길 수 있습니다. 이 경우 `@StepScope`를 고려하거나 스레드 안전한 방식으로 빈을 설계해야 합니다.
5. **SpEL 표현식의 정확성:** `#{jobParameters['키']}`와 같이 SpEL 표현식에서 키 이름을 정확히 사용해야 합니다. 오타가 있으면 `null`이 주입되거나 오류가 발생할 수 있습니다.
6. **불필요한 스코프 사용 자제:** 모든 빈을 `@StepScope`로 만들 필요는 없습니다. 실행 시점에 값이 결정되어야 하는 경우에만 사용하세요. 스코프 빈은 프록시를 통해 관리되므로 약간의 오버헤드가 있을 수 있습니다.

---

### **이전 학습 내용과의 연관성**

- **`JobParameters` (이전 챕터에서 간략히 언급되었을 수 있음):** 이번 챕터에서는 `JobParameters`가 어떻게 실제 빈의 속성값으로 연결되어 사용되는지 구체적으로 배웁니다. `JobParameters`는 잡을 실행할 때 외부에서 값을 전달하는 표준 방법이며, 지연 바인딩의 주요 입력 소스입니다.
- **`ExecutionContext` (이전 챕터에서 간략히 언급되었을 수 있음):** 스텝 간 또는 잡 재시작 시 데이터를 공유하는 메커니즘인 `ExecutionContext`의 값을 어떻게 SpEL을 통해 빈에 주입하여 활용하는지 보여줍니다.
- **`ItemReader`, `ItemWriter` (앞으로 배울 내용 또는 이미 알고 있다면):** 이러한 핵심 배치 컴포넌트들이 어떻게 `@StepScope`를 통해 동적인 설정을 받아 유연하게 동작할 수 있는지 이해하는 데 중요합니다. 예를 들어, 매일 다른 이름의 입력 파일을 읽어야 하는 `ItemReader`는 지연 바인딩의 대표적인 활용 사례입니다.

---
