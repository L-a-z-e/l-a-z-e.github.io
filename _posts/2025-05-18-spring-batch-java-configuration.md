---
title: Spring Batch - Java Configuration
description: 
author: laze
date: 2025-05-18 00:00:02 +0900
categories: [Dev, SpringBatch]
tags: [SpringBatch]
---
**Java 구성 (Java Configuration)**

Spring 3는 XML 대신 Java로 애플리케이션을 구성하는 기능을 도입했습니다.

Spring Batch 2.2.0부터 동일한 Java 구성을 사용하여 배치 작업을 구성할 수 있습니다.

Java 기반 구성에는 `@EnableBatchProcessing` 어노테이션과 두 개의 빌더라는 세 가지 구성 요소가 있습니다.

`@EnableBatchProcessing` 어노테이션은 Spring 제품군의 다른 `@Enable*` 어노테이션과 유사하게 작동합니다.

이 경우 `@EnableBatchProcessing`은 배치 작업을 구축하기 위한 기본 구성을 제공합니다.

이 기본 구성 내에서 `StepScope` 및 `JobScope`의 인스턴스가 생성되며, 자동 주입(autowired)될 수 있는 여러 빈(bean)도 사용할 수 있게 됩니다.

- `JobRepository`: `jobRepository`라는 이름의 빈
- `JobLauncher`: `jobLauncher`라는 이름의 빈
- `JobRegistry`: `jobRegistry`라는 이름의 빈
- `JobExplorer`: `jobExplorer`라는 이름의 빈
- `JobOperator`: `jobOperator`라는 이름의 빈

기본 구현은 위 목록에 언급된 빈을 제공하며, 컨텍스트 내에 `DataSource`와 `PlatformTransactionManager`가 빈으로 제공되어야 합니다.

데이터 소스와 트랜잭션 관리자는 `JobRepository` 및 `JobExplorer` 인스턴스에서 사용됩니다.

기본적으로 `dataSource`라는 이름의 데이터 소스와 `transactionManager`라는 이름의 트랜잭션 관리자가 사용됩니다.

`@EnableBatchProcessing` 어노테이션의 속성을 사용하여 이러한 빈을 사용자 정의할 수 있습니다.

다음 예는 사용자 정의 데이터 소스 및 트랜잭션 관리자를 제공하는 방법을 보여줍니다.

```java
@Configuration
@EnableBatchProcessing(dataSourceRef = "batchDataSource", transactionManagerRef = "batchTransactionManager")
public class MyJobConfiguration {

	@Bean
	public DataSource batchDataSource() {
		return new EmbeddedDatabaseBuilder().setType(EmbeddedDatabaseType.HSQL)
				.addScript("/org/springframework/batch/core/schema-hsqldb.sql")
				.generateUniqueName(true).build();
	}

	@Bean
	public JdbcTransactionManager batchTransactionManager(DataSource dataSource) { // dataSourceRef에 지정된 batchDataSource 빈이 주입됨
		return new JdbcTransactionManager(dataSource);
	}

	@Bean
	public Job job(JobRepository jobRepository) {
		return new JobBuilder("myJob", jobRepository)
				// 필요에 따라 Job 흐름 정의
				.build();
	}

}
```

오직 하나의 구성 클래스에만 `@EnableBatchProcessing` 어노테이션이 필요합니다.

이 어노테이션이 있는 클래스가 있으면 앞에서 설명한 모든 구성을 갖게 됩니다.

v5.0부터 기본 인프라 빈을 프로그래밍 방식으로 구성하는 대안으로 `DefaultBatchConfiguration` 클래스가 제공됩니다.

이 클래스는 `@EnableBatchProcessing`에서 제공하는 것과 동일한 빈을 제공하며 배치 작업을 구성하기 위한 기본 클래스로 사용할 수 있습니다.

다음 스니펫은 이를 사용하는 일반적인 예입니다.

```java
@Configuration
class MyJobConfiguration extends DefaultBatchConfiguration { // DefaultBatchConfiguration 상속

	@Bean
	public Job job(JobRepository jobRepository) {
		return new JobBuilder("job", jobRepository)
				// 필요에 따라 Job 흐름 정의
				.build();
	}

}
```

데이터 소스와 트랜잭션 관리자는 애플리케이션 컨텍스트에서 확인되어 Job 리포지토리와 Job 탐색기에 설정됩니다.

필요한 세터(setter)를 재정의하여 모든 인프라 빈의 구성을 사용자 정의할 수 있습니다.

다음 예는 인스턴스에 대한 문자 인코딩을 사용자 정의하는 방법을 보여줍니다.

```java
@Configuration
class MyJobConfiguration extends DefaultBatchConfiguration {

	@Bean
	public Job job(JobRepository jobRepository) {
		return new JobBuilder("job", jobRepository)
				// 필요에 따라 Job 흐름 정의
				.build();
	}

	@Override // DefaultBatchConfiguration의 메서드 재정의
	protected Charset getCharset() {
		return StandardCharsets.ISO_8859_1;
	}
}
```

`@EnableBatchProcessing`은 `DefaultBatchConfiguration`과 함께 사용해서는 안 됩니다.

`@EnableBatchProcessing`을 통한 선언적 방식의 Spring Batch 구성 또는 `DefaultBatchConfiguration`을 확장하는 프로그래밍 방식 중 하나를 사용해야 하며, 두 가지 방식을 동시에 사용해서는 안 됩니다.

---

## Spring Batch, Java 코드로 우아하게 설정하기! ☕✨

예전에는 Spring 설정을 주로 XML 파일로 했지만, 요즘에는 Java 코드로 직접 설정하는 방식이 대세예요. Spring Batch도 마찬가지랍니다! Java 코드로 설정하면 다음과 같은 장점들이 있어요.

- **타입 안전성:** 컴파일 시점에 오류를 잡을 수 있어요. (XML은 실행해봐야 알죠 😥)
- **리팩토링 용이:** 코드 변경이 훨씬 쉬워요.
- **IDE 지원:** 자동완성 같은 IDE의 편리한 기능을 마음껏 쓸 수 있어요.
- **가독성:** 복잡한 로직도 Java 코드로 명확하게 표현할 수 있어요.

Spring Batch를 Java 코드로 설정하는 핵심 요소는 크게 두 가지예요.

1. **`@EnableBatchProcessing` 어노테이션:** "자, 지금부터 Spring Batch 기본 설정 들어갑니다!" 라고 알려주는 마법의 주문!
2. **두 개의 빌더(Builders):** `JobBuilderFactory`와 `StepBuilderFactory` (또는 직접 `JobBuilder`, `StepBuilder` 사용) - `Job`과 `Step`을 조립하는 건축가들이죠. (이 빌더들은 `@EnableBatchProcessing`이 이미 만들어줘서 우리가 직접 생성할 필요는 없을 때가 많아요.)

---

### 1. `@EnableBatchProcessing`: Spring Batch 설정의 시작! 🎬

이 어노테이션 하나만 설정 클래스에 딱! 붙여주면, Spring Batch를 사용하는 데 필요한 기본적인 것들이 알아서 준비돼요. 마치 요리 프로그램에서 "자, 기본 재료 준비됐습니다!" 하는 것과 같아요.

**`@EnableBatchProcessing`이 자동으로 해주는 일:**

- **핵심 빈(Bean) 등록:** Spring Batch가 돌아가는 데 꼭 필요한 여러 객체(빈)들을 자동으로 만들어서 Spring 컨테이너에 등록해줘요. 우리가 이 빈들을 가져다 쓰기만 하면 돼요 (자동 주입 - Autowired).
  - `JobRepository` (`jobRepository`라는 이름으로): 배치 작업의 모든 메타데이터(실행 기록, 상태 등)를 저장하고 관리하는 저장소.
  - `JobLauncher` (`jobLauncher`라는 이름으로): 배치 작업을 시작시키는 실행기.
  - `JobRegistry` (`jobRegistry`라는 이름으로): 생성된 Job들을 등록하고 관리하는 곳.
  - `JobExplorer` (`jobExplorer`라는 이름으로): `JobRepository`에 저장된 메타데이터를 읽기 전용으로 조회하는 도구.
  - `JobOperator` (`jobOperator`라는 이름으로): Job 중단, 재시작, 요약 정보 조회 등 운영에 필요한 다양한 기능을 제공하는 인터페이스.
- **스코프(Scope) 생성:** `StepScope`와 `JobScope`라는 특별한 스코프를 사용할 수 있게 해줘요. (나중에 자세히 배울 텐데, `Step`이나 `Job` 실행 범위 내에서만 유효한 빈을 만들 때 유용해요.)

**`@EnableBatchProcessing` 사용 시 꼭 필요한 것!**

`@EnableBatchProcessing`이 이런 멋진 일들을 해주려면, 우리에게 두 가지를 준비해달라고 요청해요.

1. **`DataSource` (데이터 소스):** `JobRepository`가 배치 메타데이터를 저장할 데이터베이스에 연결하기 위해 필요해요. 어떤 데이터베이스를 쓸지 알려줘야겠죠?
2. **`PlatformTransactionManager` (트랜잭션 관리자):** 배치 작업 중 데이터 일관성을 지키기 위한 트랜잭션 처리를 위해 필요해요.

기본적으로 Spring Batch는 `dataSource`라는 이름의 `DataSource` 빈과 `transactionManager`라는 이름의 `PlatformTransactionManager` 빈을 찾아서 사용하려고 해요.

**만약 다른 이름의 `DataSource`나 `TransactionManager`를 쓰고 싶다면?**

`@EnableBatchProcessing` 어노테이션에 속성을 지정해서 알려줄 수 있어요.

```java
@Configuration
// "내가 쓸 DataSource 이름은 batchDataSource 이고, TransactionManager 이름은 batchTransactionManager 야!"
@EnableBatchProcessing(dataSourceRef = "batchDataSource", transactionManagerRef = "batchTransactionManager")
public class MyJobConfiguration {

    // 1. batchDataSource 라는 이름으로 DataSource 빈 만들기
    @Bean
    public DataSource batchDataSource() {
        // 예시: HSQL 임베디드 DB 사용 설정
        return new EmbeddedDatabaseBuilder()
                .setType(EmbeddedDatabaseType.HSQL) // HSQL DB 쓸 거야.
                .addScript("classpath:org/springframework/batch/core/schema-hsqldb.sql") // Spring Batch 메타데이터 테이블 생성 스크립트 실행
                .generateUniqueName(true) // DB 이름을 고유하게 생성
                .build();
    }

    // 2. batchTransactionManager 라는 이름으로 PlatformTransactionManager 빈 만들기
    @Bean
    public PlatformTransactionManager batchTransactionManager(DataSource batchDataSource) { // 위에서 만든 batchDataSource를 주입받음
        return new JdbcTransactionManager(batchDataSource); // JDBC 기반 트랜잭션 매니저 사용
    }

    // 3. 이제 Job, Step 등을 정의하면 돼요!
    @Bean
    public Job job(JobRepository jobRepository, Step myStep) { // @EnableBatchProcessing이 만들어준 jobRepository를 주입받음
        return new JobBuilder("myJob", jobRepository)
                 .start(myStep)
                 .build();
    }

    @Bean
    public Step myStep(/* 필요한 의존성 주입 */) {
        // Step 정의 (Tasklet 또는 Chunk 방식)
        // ...
        return null; // 실제 Step 객체를 반환해야 합니다.
    }
}

```

**중요!** `@EnableBatchProcessing` 어노테이션은 애플리케이션 전체에서 **단 하나의 설정 클래스에만** 붙여주면 됩니다. 여러 군데 붙일 필요 없어요!

**핵심:** `@EnableBatchProcessing` 어노테이션 하나로 Spring Batch의 **기본 골격이 자동으로 설정**돼요. 우리는 `DataSource`와 `TransactionManager`만 잘 준비해주면 된답니다!

---

### 2. `DefaultBatchConfiguration`: 또 다른 설정 방법 (Spring Batch 5.0 이상)

Spring Batch 5.0 버전부터는 `@EnableBatchProcessing` 어노테이션 대신, `DefaultBatchConfiguration`이라는 클래스를 **상속**받아서 기본 설정을 가져오는 방법도 생겼어요.

이 방법은 `@EnableBatchProcessing`이 해주는 것과 동일한 기본 빈들을 제공하지만, 어노테이션 방식보다 좀 더 **프로그래밍 방식(programmatic way)**으로 설정을 커스터마이징하고 싶을 때 유용해요.

```java
@Configuration
class MyJobConfiguration extends DefaultBatchConfiguration { // DefaultBatchConfiguration을 상속받아요!

    // Job, Step 등 정의는 동일
    @Bean
    public Job job(JobRepository jobRepository, Step myStep) {
        return new JobBuilder("job", jobRepository)
                 .start(myStep)
                 .build();
    }

    @Bean
    public Step myStep(/* 필요한 의존성 주입 */) {
        // Step 정의
        // ...
        return null;
    }

    // 만약 기본 설정을 바꾸고 싶다면, 부모 클래스의 메서드를 오버라이드(재정의)해요.
    // 예시: JobRepository가 사용할 문자 인코딩을 변경하고 싶다면?
    @Override
    protected Charset getCharset() {
        return StandardCharsets.ISO_8859_1; // 기본값은 UTF-8 인데, ISO_8859_1로 변경!
    }

    // 예시: JobRepository가 사용할 테이블 접두어(prefix)를 변경하고 싶다면?
    // @Override
    // protected String getTablePrefix() {
    //     return "MY_BATCH_"; // 기본값은 BATCH_ 인데, MY_BATCH_로 변경!
    // }
}

```

**`DefaultBatchConfiguration` 사용 시 주의사항:**

- `@EnableBatchProcessing` 어노테이션과 `DefaultBatchConfiguration` 클래스 상속 방식은 **함께 사용하면 안 돼요!** 둘 중 하나만 선택해야 합니다. (마치 짜장면과 짬뽕 중 하나만 고르는 것처럼요! 😉)
- `DataSource`와 `PlatformTransactionManager`는 여전히 애플리케이션 컨텍스트에 빈으로 등록되어 있어야 해요. `DefaultBatchConfiguration`이 알아서 찾아 쓸 거예요. (보통 `dataSource`와 `transactionManager`라는 이름으로)
- 기본 설정을 변경하고 싶을 때는, `DefaultBatchConfiguration` 클래스 내부에 있는 `protected` 접근 제한자의 `getter` 메서드들을 `@Override` 해서 원하는 값을 반환하도록 구현하면 됩니다. (예: `getCharset()`, `getTablePrefix()`, `getDataSource()`, `getTransactionManager()` 등)

**어떤 방식을 선택해야 할까?**

- **`@EnableBatchProcessing`:** 대부분의 경우 더 간편하고 선언적인 방식이라 추천돼요. 간단한 커스터마이징은 어노테이션 속성으로도 가능하고요.
- **`DefaultBatchConfiguration` 상속:** Spring Batch의 기본 인프라 빈 설정을 좀 더 세밀하게 프로그래밍 방식으로 제어하고 싶거나, 여러 설정을 한 클래스 내에서 상속 구조로 관리하고 싶을 때 유용할 수 있어요.

**핵심:** Java 코드로 Spring Batch를 설정하는 것은 **유연하고 강력하며, 타입 안전성**까지 챙길 수 있는 좋은 방법이에요. `@EnableBatchProcessing`을 사용하든 `DefaultBatchConfiguration`을 상속받든, 핵심은 `JobRepository`와 `JobLauncher` 같은 기본 빈들이 잘 준비되도록 하는 것이랍니다!
