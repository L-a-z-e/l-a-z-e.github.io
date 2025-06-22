---
title: Spring Batch - Review(1)
description: 
author: laze
date: 2025-06-22 00:00:02 +0900
categories: [Dev, SpringBatch]
tags: [SpringBatch]
---
**Spring Batch란?**

Spring Batch는 엔터프라이즈 시스템 운영에 있어 **대용량의 데이터를 주기적으로, 혹은 특정 조건에 따라 자동으로 처리(일괄 처리, Batch Processing)하기 위한 경량의 포괄적인 프레임워크**입니다. 단순히 데이터를 읽고 쓰는 것뿐만 아니라, 대규모 작업에서 필수적인 로깅, 추적, 트랜잭션 관리, 작업 처리 통계, 작업 흐름 관리, 실패 지점에서의 재시작 등 다양한 기능을 제공하여 개발자가 비즈니스 로직에 더 집중할 수 있도록 셔틀 역할을 합니다.

**Spring Batch의 주요 목표 및 필요성:**

- **대용량 데이터 처리의 자동화 및 효율화:** 수동 개입 없이 대규모 데이터를 안정적으로 처리합니다.
- **신뢰성 및 안정성 확보:** 트랜잭션 관리, 오류 처리(Skip/Retry), 재시작 기능을 통해 데이터 정합성을 유지하고 작업 실패 시 복구를 용이하게 합니다.
- **개발 생산성 향상:** 반복적이고 정형화된 배치 작업 패턴을 프레임워크 수준에서 지원하여 개발자가 보일러플레이트 코드를 줄이고 핵심 로직에 집중할 수 있게 합니다.
- **확장성 제공:** 병렬 처리, 파티셔닝 등을 통해 처리 성능을 향상시킬 수 있습니다.

사용 예시

1. **정산 시스템 (예: 일일/월간 거래 정산)**
  - **특징:** 대량의 거래 내역 데이터를 읽어와서, 수수료 계산, 판매자/구매자 간 금액 정산, 결과 데이터 생성 등의 복잡한 로직을 수행해야 합니다. 데이터의 정확성이 매우 중요합니다.
  - **Spring Batch 활용 포인트:**
    - **대용량 처리:** 수많은 거래 내역을 청크 단위로 효율적으로 처리합니다.
    - **트랜잭션 관리:** 각 정산 단위(또는 청크)가 원자적으로 처리되어, 중간에 오류가 발생해도 데이터 불일치를 최소화합니다.
    - **재시작 기능:** 정산 작업 중 오류로 중단되더라도, 실패 지점부터 안전하게 재시작하여 전체 작업을 다시 수행하는 비효율을 막습니다.
    - **로깅 및 모니터링:** 어떤 데이터가 어떻게 처리되었고, 오류는 무엇이었는지 상세한 로그를 남겨 추적 및 감사에 용이합니다.
2. **대출 이자 계산 (예: 매월 말 대출 잔액에 대한 이자 계산 및 부과)**
  - **특징:** 많은 고객의 대출 계좌 정보를 읽어와, 각 계좌별로 이자율, 기간, 잔액 등을 고려하여 이자를 계산하고, 이자 내역을 생성하거나 고객 계정에 반영합니다.
  - **Spring Batch 활용 포인트:**
    - **반복적인 계산 로직:** 각 고객 계좌에 대해 동일한 이자 계산 로직을 적용하는 ItemProcessor를 활용할 수 있습니다.
    - **데이터베이스 연동:** 고객 정보 및 대출 정보를 데이터베이스에서 효율적으로 읽어오고(ItemReader), 계산된 이자 정보를 다시 데이터베이스에 업데이트하거나 별도 테이블에 저장(ItemWriter)합니다.
    - **스케줄링 연동:** 매월 특정일에 자동으로 실행되도록 스케줄러(예: Spring Scheduler, Quartz)와 쉽게 연동할 수 있습니다.
3. **주문 처리 시스템 (예: 하루 동안 들어온 온라인 주문 일괄 처리)**
  - **특징:** 실시간으로 들어온 주문들을 특정 시점(예: 매일 자정)에 모아서 재고 확인, 결제 처리, 배송 정보 생성, 고객 알림 등의 후속 작업을 일괄적으로 수행합니다.
  - **Spring Batch 활용 포인트:**
    - **다단계 처리:** 주문 데이터를 읽어오는 스텝, 재고를 확인하고 업데이트하는 스텝, 결제를 처리하는 스텝, 배송 정보를 생성하는 스텝 등 여러 단계(Step)로 나누어 잡(Job)을 구성할 수 있습니다. 각 스텝의 성공/실패에 따라 다음 스텝의 실행 여부를 제어할 수 있습니다.
    - **다양한 입출력:** 주문 데이터는 DB에서, 재고 정보는 별도 시스템 API 호출로, 배송 정보는 파일로 생성하는 등 다양한 데이터 소스와 목적지를 유연하게 처리할 수 있습니다.
    - **오류 처리:** 특정 주문 처리 중 오류 발생 시 해당 주문만 스킵하거나(skip), 재시도(retry)하는 등의 세밀한 오류 제어가 가능합니다.
- **데이터 마이그레이션:** 기존 시스템의 데이터를 새로운 시스템으로 대량 이전.
- **리포트 생성:** 대량의 원시 데이터를 집계하고 분석하여 주기적인 통계 리포트 생성.
- **알림 발송:** 특정 조건에 맞는 고객들에게 대량의 이메일이나 SMS 발송.
- **데이터 동기화:** 여러 시스템 간의 데이터를 주기적으로 동기화.

### **Spring Batch의 핵심 아키텍처를 설명할 때 가장 중요한 네 가지 계층 또는 컴포넌트 그룹은 무엇이며, 이들은 각각 어떤 책임을 가지고 유기적으로 동작하는지?**

---

Spring Batch 아키텍처는 크게 다음과 같은 **세 가지 주요 계층(Layer)**으로 나눌 수 있습니다. (네 가지로 보기도 하지만, 핵심은 이 세 가지의 상호작용입니다.)

1. **애플리케이션 (Application) 계층:**
  - **역할:** 개발자가 **직접 작성하는 모든 배치 관련 코드와 비즈니스 로직**이 이 계층에 속합니다.
  - **포함 요소:**
    - `Job` 설정 (우리가 `@Bean`으로 만드는 `Job` 객체들)
    - `Step` 설정 (우리가 `@Bean`으로 만드는 `Step` 객체들)
    - `ItemReader` 구현체 (예: `FlatFileItemReader` 설정, 커스텀 Reader)
    - `ItemProcessor` 구현체 (비즈니스 로직, 데이터 변환 로직)
    - `ItemWriter` 구현체 (예: `JdbcBatchItemWriter` 설정, 커스텀 Writer)
    - 기타 도메인 객체, 서비스, DAO 등
  - **핵심:** "어떤 데이터를 가지고, 어떤 순서로, 어떤 처리를 할 것인가?"에 대한 구체적인 내용을 담습니다.
2. **코어 (Core) 계층:**
  - **역할:** 배치 잡을 **실행하고 제어하는 데 필요한 핵심 런타임 컴포넌트**들을 제공합니다. 애플리케이션 계층에서 정의된 배치 작업을 실제로 구동시키는 엔진과 같습니다.
  - **포함 요소:**
    - **`JobLauncher`**: `Job`을 실행시키는 인터페이스.
    - **`Job`**: 배치 작업의 전체 실행 단위 (인터페이스 및 구현체 - 예: `SimpleJob`).
    - **`Step`**: `Job` 내의 독립적인 실행 단계 (인터페이스 및 구현체 - 예: `TaskletStep`, `SimpleStep`).
    - **`JobRepository`**: `Job` 및 `Step`의 실행 메타데이터(상태, 통계 등)를 **저장하고 조회**하는 역할. (이전에 "일기장" 또는 "작업 기록부"로 비유했었죠!)
  - **핵심:** 배치 실행의 생명주기 관리, 흐름 제어, 상태 관리 등 프레임워크의 핵심 기능을 담당합니다.
3. **인프라스트럭처 (Infrastructure) 계층:**
  - **역할:** 애플리케이션 계층과 코어 계층 모두에게 공통적으로 필요한 **기반 서비스 및 추상화**를 제공합니다. 프레임워크가 원활하게 동작하기 위한 지원 부대와 같습니다.
  - **포함 요소:**
    - **`ItemReader`, `ItemProcessor`, `ItemWriter` 인터페이스**: 애플리케이션 계층에서 구현해야 할 계약을 정의합니다.
    - **다양한 `ItemReader` 및 `ItemWriter` 구현체**: (예: `FlatFileItemReader`, `JdbcCursorItemReader`, `StaxEventItemWriter` 등) 바로 사용할 수 있는 편리한 도구들을 제공합니다.
    - **재시작/스킵 처리 로직**: 오류 발생 시 작업을 어떻게 이어갈지에 대한 기반 로직.
    - **트랜잭션 관리 추상화**: `PlatformTransactionManager`를 통해 일관된 트랜잭션 처리를 지원.
    - **리소스 관리 추상화**: 스프링의 `Resource` 인터페이스 등을 활용.
  - **핵심:** 반복적으로 사용되는 공통 기능, 외부 시스템과의 연동, 저수준의 기술적인 처리 등을 담당하여 개발자가 비즈니스 로직에 더 집중할 수 있도록 돕습니다.

**간단한 관계도:**

```
+---------------------+
|     Application     |  <-- 개발자가 주로 작업하는 영역 (Job/Step 정의, R/P/W 구현)
| (배치 작업 명세)    |
+---------------------+
        ^
        | (실행 요청, 설정 제공)
        v
+---------------------+
|        Core         |  <-- 배치 실행 엔진 (JobLauncher, Job, Step, JobRepository)
| (배치 실행 및 제어) |
+---------------------+
        ^
        | (공통 기능 사용, 인터페이스 구현)
        v
+---------------------+
|   Infrastructure    |  <-- 지원 도구 및 기반 (Readers, Writers, 트랜잭션, 리소스)
| (공통 서비스 및 추상화) |
+---------------------+

```

이 세 가지 계층이 서로 유기적으로 상호작용하면서 하나의 Spring Batch 애플리케이션이 완성되는 것입니다.

---

### Spring Batch의 3대 핵심 컴포넌트라고 할 수 있는 **Job, Step, 그리고 JobExecution (또는 JobInstance)** 이 세 가지의 관계

---

**1. `Job`**

- **정의:** `Job`은 Spring Batch에서 **하나의 완전한 배치 처리 작업**을 나타내는 최상위 개념이자 실행 단위입니다. 마치 하나의 "프로젝트"나 "업무"와 같습니다.
- **구성:** `Job`은 순서대로 또는 조건에 따라 실행되는 **하나 이상의 `Step`들로 구성**됩니다. 이 `Step`들의 실행 흐름(Flow)을 정의하는 것이 `Job` 설정의 핵심입니다.
- **특징:**
  - 고유한 이름을 가집니다.
  - 재시작 가능 여부 등을 설정할 수 있습니다.
  - `JobParameters`를 받아 실행될 수 있습니다.

**2. `Step`**

- **정의:** `Step`은 `Job` 내부에서 **실질적인 배치 처리가 이루어지는 독립적인 단계 또는 국면**입니다. `Job`이라는 큰 프로젝트 내의 작은 "작업 패키지" 또는 "공정 단계"와 같습니다.
- **구성 (주로 청크 지향 스텝의 경우):**
  - **`ItemReader`**: 데이터를 읽어옵니다.
  - **`ItemProcessor`** (선택 사항): 읽어온 데이터를 가공/변환/필터링합니다.
  - **`ItemWriter`**: 처리된 데이터를 씁니다.
- **추가 기능:**
  - **`ItemStream`**: `Reader`, `Processor`(드물지만 가능), `Writer`가 `ItemStream`을 구현하여 리소스 관리 및 상태 저장/복원(재시작 지원) 기능을 가질 수 있습니다.
  - **오류 처리**: `skip` (특정 예외 발생 시 아이템 건너뛰기), `retry` (일시적 오류 발생 시 작업 재시도) 등의 정책을 설정할 수 있습니다.
  - **트랜잭션 관리**: 청크 단위로 트랜잭션이 관리됩니다.
  - 리스너 (`StepExecutionListener`, `ChunkListener` 등)를 통해 스텝 실행의 다양한 시점에 부가적인 로직을 실행할 수 있습니다.
- **유형:** 크게 **청크 지향 스텝(Chunk-Oriented Step)**과 **태스크릿 스텝(Tasklet Step)**으로 나뉩니다.

**3. `JobExecution`**

- **정의:** `JobExecution`은 특정 `JobInstance`에 대한 **한 번의 실행 시도(attempt)**를 나타내는 객체입니다. 즉, `JobLauncher`가 `Job`을 `run()` 시킬 때마다 새로운 `JobExecution`이 생성됩니다 (실패 후 재시작하는 경우에도 이전 `JobInstance`에 대한 새로운 `JobExecution`이 생성됨).
- **포함 정보 (메타데이터):**
  - 실행 상태 (`BatchStatus`: `STARTING`, `STARTED`, `COMPLETED`, `FAILED`, `STOPPED`, `ABANDONED`, `UNKNOWN`)
  - 시작 시간, 종료 시간, 마지막 업데이트 시간
  - 생성 파라미터 (`JobParameters`) - 어떤 파라미터로 실행되었는지
  - `ExecutionContext` (`JobExecutionContext`) - 해당 잡 실행 범위 내에서 공유되는 데이터
  - 해당 `JobExecution`에 속한 각 `StepExecution`들의 목록
  - `JobExecution`은 전체 잡의 성공/실패 여부와 전체적인 실행 정보를 담고, 각 스텝의 상세 진행 상황은 `StepExecution`에 기록됩니다.

**4. `JobInstance`**

- **정의:** `JobInstance`는 논리적으로 **한 번의 "배치 작업 실행"**을 의미합니다. 이것은 **`Job`의 이름**과 해당 잡을 실행하는 데 사용된 **고유한 식별 `JobParameters`의 조합**으로 정의됩니다.
- **예시:**
  - `dailySalesReportJob`이라는 `Job`이 있고,
  - `date=2023-01-01` 이라는 `JobParameters`로 실행했다면, 이것이 하나의 `JobInstance`입니다.
  - 다음 날 `date=2023-01-02` 라는 `JobParameters`로 동일한 `dailySalesReportJob`을 실행했다면, 이것은 **또 다른 새로운 `JobInstance`*입니다.
- **관계:**
  - 하나의 `Job` 정의(설계도)는 여러 개의 `JobInstance`(다른 재료로 만든 제품들)를 가질 수 있습니다.
  - 하나의 `JobInstance`는 여러 번의 `JobExecution`(제품을 만드는 시도들 - 첫 시도에 실패하고 재시도할 수 있음)을 가질 수 있습니다.
    - 예: `dailySalesReportJob` + `date=2023-01-01` (하나의 `JobInstance`)
      - 1차 실행 시도 (`JobExecution` 1) -> FAILED
      - 2차 실행 시도 (`JobExecution` 2) -> COMPLETED
- **`JobRepository`의 역할:** `JobRepository`는 이러한 `JobInstance`와 `JobExecution`들의 정보를 저장하고 관리하여, 동일한 `JobInstance`에 대한 재시작을 가능하게 하고 실행 이력을 추적합니다.

**정리하면, 이들의 관계는 다음과 같습니다:**

```
  Job (설계도, 예: "일일 판매 보고서 생성 작업")
   |
   +-- JobInstance 1 (식별 파라미터: date=2023-01-01)
   |    |
   |    +-- JobExecution 1 (첫 번째 실행 시도 - FAILED)
   |    |    |
   |    |    +-- StepExecution 1 (stepA - COMPLETED)
   |    |    +-- StepExecution 2 (stepB - FAILED)
   |    |
   |    +-- JobExecution 2 (재시작 시도 - COMPLETED)
   |         |
   |         +-- StepExecution 1 (stepA - 이미 COMPLETED이므로 스킵될 수 있음)
   |         +-- StepExecution 2 (stepB - FAILED 지점부터 재시작하여 COMPLETED)
   |
   +-- JobInstance 2 (식별 파라미터: date=2023-01-02)
        |
        +-- JobExecution 1 (첫 번째 실행 시도 - COMPLETED)
             |
             +-- StepExecution 1 (stepA - COMPLETED)
             +-- StepExecution 2 (stepB - COMPLETED)
```

---

**Spring Batch에서 `Job`과 `Step`을 설정(정의)할 때, 최근 주로 사용되는 방식은 무엇이며(예: XML vs Java Config), Java Config 방식을 사용할 때 주로 어떤 빌더(Builder) 클래스들을 활용하게 되나요?**

---

**1. 설정 방식: Java Config의 대세**

- 과거에는 Spring 프레임워크 전반적으로 XML 기반 설정이 많이 사용되었고, Spring Batch 역시 XML 네임스페이스를 통한 설정을 지원했습니다.
- 하지만 최근 Spring (특히 Spring Boot) 생태계에서는 **Java 기반 설정(Java Configuration)**이 훨씬 더 선호되고 널리 사용됩니다.
  - **타입 안전성(Type Safety):** 컴파일 시점에 설정 오류를 발견하기 용이합니다.
  - **리팩토링 용이성:** IDE의 지원을 받아 설정 변경이나 리팩토링이 쉽습니다.
  - **유연성:** 일반 Java 코드를 활용하여 동적인 설정 구성이 가능합니다.
  - **가독성:** (개인차는 있지만) 많은 개발자들이 XML보다 Java 코드를 더 직관적으로 이해합니다.
- 따라서 Spring Batch 잡을 설정할 때도, 특별한 이유가 없다면 **Java Config 방식**을 사용하는 것이 현재의 모범 사례(Best Practice)입니다.

**2. 핵심 빌더: `JobBuilder` 와 `StepBuilder`**

- Java Config 방식으로 `Job`과 `Step`을 정의할 때, 핵심적인 역할을 하는 것이 바로 이 빌더 클래스들입니다.
- **`JobBuilder`**:
  - **역할:** `Job` 객체를 생성하고 설정하기 위한 빌더입니다.
  - **사용법 (Spring Batch 5.x 기준):**
    - 과거 `JobBuilderFactory`를 주입받아 `factory.get("jobName")`으로 시작하는 방식도 있었지만, Spring Batch 5부터는 `JobRepository`를 직접 주입받는 생성자를 가진 `JobBuilder`를 사용하는 것이 권장됩니다.
    - `new JobBuilder("jobName", jobRepository)`: 잡의 이름과 `JobRepository`를 전달하여 빌더를 생성합니다.
    - 이후 `.start(step)`, `.next(step)`, `.listener(listener)`, `.incrementer(incrementer)` 등 다양한 메서드를 체이닝하여 `Job`을 구성하고, 마지막에 `.build()`를 호출하여 `Job` 객체를 생성합니다.
- **`StepBuilder`**:
  - **역할:** `Step` 객체를 생성하고 설정하기 위한 빌더입니다.
  - **사용법 (Spring Batch 5.x 기준):**
    - 과거 `StepBuilderFactory`를 주입받아 `factory.get("stepName")`으로 시작하는 방식도 있었지만, Spring Batch 5부터는 `JobRepository`와 `PlatformTransactionManager`를 직접 주입받는 생성자를 가진 `StepBuilder`를 사용하는 것이 권장됩니다.
    - `new StepBuilder("stepName", jobRepository)`: 스텝의 이름과 `JobRepository`를 전달하여 빌더를 생성합니다.
    - 이후 스텝의 유형에 따라 메서드를 선택합니다.
      - **청크 지향 스텝:** `.<InputType, OutputType>chunk(commitInterval, transactionManager)`를 호출하고, 이어서 `.reader(itemReader)`, `.processor(itemProcessor)`, `.writer(itemWriter)`, `.listener(listener)`, `.stream(itemStream)` 등을 체이닝하여 구성합니다.
      - **태스크릿 스텝:** `.tasklet(tasklet, transactionManager)`를 호출하여 `Tasklet` 구현체를 설정합니다.
    - 마지막에 `.build()`를 호출하여 `Step` 객체를 생성합니다.

**간단한 Java Config 예시:**

```java
@Configuration
public class MyBatchConfiguration {

    private final JobRepository jobRepository; // Spring Boot가 자동 구성
    private final PlatformTransactionManager transactionManager; // Spring Boot가 자동 구성

    // 생성자 주입
    public MyBatchConfiguration(JobRepository jobRepository, PlatformTransactionManager transactionManager) {
        this.jobRepository = jobRepository;
        this.transactionManager = transactionManager;
    }

    @Bean
    public ItemReader<String> myReader() {
        // ... ItemReader 구현 ...
        return new ListItemReader<>(Arrays.asList("a", "b", "c"));
    }

    @Bean
    public ItemWriter<String> myWriter() {
        // ... ItemWriter 구현 ...
        return items -> items.forEach(System.out::println);
    }

    @Bean
    public Step myStep(ItemReader<String> myReader, ItemWriter<String> myWriter) {
        return new StepBuilder("myStepName", jobRepository)
                .<String, String>chunk(10, transactionManager) // 청크 설정
                .reader(myReader)
                .writer(myWriter)
                .build();
    }

    @Bean
    public Job myJob(Step myStep) {
        return new JobBuilder("myJobName", jobRepository)
                .start(myStep)
                .build();
    }
}
```

이처럼 `@Configuration` 어노테이션이 붙은 클래스 내에 `@Bean` 어노테이션을 사용하여 각 컴포넌트(Reader, Writer, Step, Job 등)를 정의하고, `JobBuilder`와 `StepBuilder`를 통해 이들을 조립하여 하나의 배치 잡을 완성합니다.

---

**`JobLauncher`를 사용하여 `Job`을 실행할 때, 최소한 어떤 두 가지 정보가 `run()` 메서드에 전달되어야 할까요?**

**그리고 `JobRepository`는 기본적으로 어떤 방식으로 메타데이터를 저장하며, Spring Boot 환경에서는 이 저장소 설정을 위해 개발자가 특별히 많은 것을 해야 할까요, 아니면 어느 정도 자동화되어 있을까요?**

---

**1. `JobLauncher`의 `run()` 메서드에 필요한 정보**

- **`Job` 객체 (또는 `Job`의 이름 - 하지만 보통 `Job` 객체 자체를 전달):** 실행할 **`Job`의 정의**가 필요합니다. "어떤 잡을 실행할 것인가?"에 대한 정보죠. (해당 이름으로 스프링 컨테이너에서 찾아온 `Job` 빈 객체를 전달합니다.)
- **`JobParameters` 객체:** 해당 `Job` 실행을 위한 **파라미터들의 묶음**입니다. "이 잡을 어떤 조건/값으로 실행할 것인가?"에 대한 정보입니다. 이 `JobParameters`는 `JobInstance`를 고유하게 식별하는 데도 사용됩니다.

**따라서 `jobLauncher.run(Job job, JobParameters jobParameters)` 형태로 최소한 `Job` 객체와 `JobParameters` 객체가 필요합니다.**

---

**2. `JobRepository`의 메타데이터 저장 방식 및 Spring Boot 환경에서의 설정**

아주 중요한 포인트를 정확히 알고 계십니다! "기본적으로 DB에 대부분 자동화돼서 저장된다"는 말씀이 맞습니다.

- **`JobRepository`의 역할 (다시 한번):**
  - `Job` 빈 자체를 "등록"하거나 저장하는 곳이 아닙니다.
  - `Job` 및 `Step`의 **실행 상태 정보(메타데이터)**, 예를 들어 `JobInstance`, `JobExecution`, `StepExecution`, `ExecutionContext` 등을 **영속적으로 저장하고 관리**하는 컴포넌트입니다.
  - 이를 통해 잡의 재시작, 실행 이력 추적, 동시 실행 제어 등이 가능해집니다.
- **기본 저장 방식:**
  - Spring Batch는 `JobRepository`의 기본 구현체로 **관계형 데이터베이스(RDBMS)를 사용하는 방식**을 제공합니다. (`JdbcJobRepository` 또는 이를 기반으로 하는 구현체)
  - 기본적으로 DB에 저장됩니다.
  - 이때 사용되는 테이블들은 `BATCH_JOB_INSTANCE`, `BATCH_JOB_EXECUTION`, `BATCH_STEP_EXECUTION`, `BATCH_JOB_EXECUTION_PARAMS`, `BATCH_STEP_EXECUTION_CONTEXT`, `BATCH_JOB_EXECUTION_CONTEXT` 등과 같이 `BATCH_` 접두사를 가지는 Spring Batch 표준 메타데이터 테이블들입니다.
- **Spring Boot 환경에서의 설정 편의성:**
  - **자동 설정 (Auto-configuration):** Spring Boot는 `spring-boot-starter-batch` 의존성이 있으면, 애플리케이션에 설정된 `DataSource`를 자동으로 감지하여 `JobRepository`를 위한 설정을 **대부분 자동으로 수행**해줍니다.
    - 즉, 개발자가 별도로 `JobRepository` 빈을 복잡하게 설정할 필요가 거의 없습니다.
    - `application.properties`에 `spring.datasource.url` 등 DB 연결 정보만 제대로 제공하면, Spring Boot가 알아서 `DataSource` 빈을 만들고, 이 `DataSource`를 사용하는 `JobRepository` (및 `PlatformTransactionManager` 등)를 구성해줍니다.
  - **메타데이터 테이블 자동 생성:**
    - `spring.batch.jdbc.initialize-schema=always` (또는 `embedded` - 임베디드 DB 사용 시) 속성을 `application.properties`에 설정하면, 애플리케이션 시작 시 Spring Batch 메타데이터 테이블들이 **자동으로 생성**됩니다. (개발 시 매우 편리)
    - 운영 환경에서는 보통 `initialize-schema=never`로 설정하고, DBA가 미리 스키마를 생성/관리하는 것이 일반적입니다.
  - **인메모리 DB 지원:** 만약 별도의 DB 설정 없이 빠르게 테스트하고 싶다면, H2, HSQLDB, Derby 같은 임베디드 DB 의존성을 추가하고 `spring.batch.jdbc.initialize-schema=embedded`로 설정하면, 애플리케이션 실행 시 인메모리 DB에 메타데이터 테이블이 생성되고 잡 실행 정보가 저장됩니다.

Spring Boot 환경에서는 `JobRepository` 설정이 매우 간편하며, DB에 메타데이터가 저장되는 것이 기본 방식입니다.

개발자는 주로 `DataSource` 설정에만 신경 쓰면 되는 경우가 많습니다.

물론, 필요하다면 `JobRepository`의 트랜잭션 격리 수준이나 테이블 접두사 등을 커스터마이징할 수도 있지만, 대부분의 경우에는 기본 설정을 그대로 사용해도 충분합니다.
