---
title: Spring Batch - ItemStream
description: 
author: laze
date: 2025-06-12 00:00:02 +0900
categories: [Dev, SpringBatch]
tags: [SpringBatch]
---
### 아이템 스트림 (ItemStream)

`ItemReader`와 `ItemWriter`는 각각의 개별적인 목적을 잘 수행하지만, 이들 모두에게 공통적인 관심사가 있어 또 다른 인터페이스가 필요합니다.

일반적으로 배치 잡(job)의 범위 내에서, 리더(reader)와 라이터(writer)는 열리고(opened), 닫혀야(closed) 하며, 상태를 영속화(persisting state)하기 위한 메커니즘이 필요합니다.

`ItemStream` 인터페이스가 바로 그 목적을 수행하며, 다음 예제와 같습니다:

```java
public interface ItemStream {

    void open(ExecutionContext executionContext) throws ItemStreamException;

    void update(ExecutionContext executionContext) throws ItemStreamException;

    void close() throws ItemStreamException;
}
```

각 메서드를 설명하기 전에, `ExecutionContext`에 대해 언급해야 합니다.

`ItemStream`을 구현하는 `ItemReader`의 클라이언트(사용자)는 `read`를 호출하기 전에 `open`을 호출하여 파일과 같은 리소스를 열거나 연결을 확보해야 합니다.

`ItemStream`을 구현하는 `ItemWriter`에도 유사한 제약 조건이 적용됩니다.

`ExecutionContext`에서 예상되는 데이터가 발견되면, 이를 사용하여 `ItemReader` 또는 `ItemWriter`를 초기 상태가 아닌 다른 위치에서 시작할 수 있습니다.

반대로, `close`는 `open` 중에 할당된 모든 리소스가 안전하게 해제되도록 호출됩니다.

`update`는 주로 현재 보유 중인 모든 상태가 제공된 `ExecutionContext`에 로드되도록 호출됩니다.

이 메서드는 커밋(commit) 전에 호출되어, 현재 상태가 커밋 전에 데이터베이스에 영속화되도록 보장합니다.

`ItemStream`의 클라이언트가 (Spring Batch Core의) `Step`인 특별한 경우, 각 `StepExecution`에 대해 `ExecutionContext`가 생성되어 사용자가 특정 실행의 상태를 저장할 수 있도록 하며,

동일한 `JobInstance`가 다시 시작될 경우 이 상태가 반환될 것으로 예상됩니다.

---

### **학습 목표 제시**

이번 "ItemStream" 챕터를 통해 학생분은 다음을 이해하고 배울 수 있습니다:

1. **`ItemStream` 인터페이스의 필요성과 역할 이해:** `ItemReader`와 `ItemWriter`가 공통적으로 필요로 하는 리소스 관리(열기/닫기) 및 상태 영속화(재시작 지원) 기능을 `ItemStream`이 어떻게 제공하는지 이해합니다.
2. **`ItemStream`의 생명주기 메서드(`open`, `update`, `close`) 파악:** 각 메서드가 언제, 왜 호출되는지, 그리고 `ExecutionContext`와 어떤 관계를 맺으며 상태를 저장하고 복원하는지 이해합니다.
3. **`ExecutionContext`와의 연관성을 통한 상태 관리 이해:** `ItemStream`이 `ExecutionContext`를 활용하여 스텝(Step)의 상태를 저장하고, 잡(Job) 재시작 시 이전 상태에서부터 작업을 이어갈 수 있도록 하는 원리를 파악합니다.

---

### **핵심 개념 설명**

이 챕터의 핵심은 **"`ItemReader`와 `ItemWriter`가 사용하는 리소스를 어떻게 관리하고, 처리 중인 상태를 어떻게 저장하여 재시작을 가능하게 할 것인가?"** 입니다.

### 1. `ItemStream`이란 무엇인가?

- **개념:** `ItemReader`나 `ItemWriter`가 파일, 데이터베이스 연결 등 외부 리소스를 사용하거나, 처리 진행 상황(예: "파일의 몇 번째 라인까지 읽었음")과 같은 내부 상태를 유지해야 할 때, 이러한 리소스의 생명주기 관리(열기, 닫기)와 상태의 영속화 및 복원을 위한 표준 방법을 제공하는 인터페이스입니다.
- **비유:**
  - **장거리 달리기 선수와 물품 보관소:** 선수가 달리기 전에 물통(리소스)을 챙기고(`open`), 중간 지점에서 현재 페이스(상태)를 기록하며 물을 보충하고(`update`), 달리기가 끝나면 물통을 반납하는(`close`) 것과 같습니다. 만약 중간에 달리기를 멈췄다가 다시 시작하면, 기록된 페이스 지점부터 다시 뛸 수 있습니다(재시작).
  - **요리사와 주방 도구 및 레시피 진행 상황:** 요리사가 요리를 시작하기 전에 필요한 도구(리소스)를 꺼내고(`open`), 특정 단계까지 요리한 후 잠시 멈출 때 현재까지의 진행 상황(상태)을 레시피에 표시해두고(`update`), 요리가 끝나면 도구를 정리하는(`close`) 것과 같습니다. 다시 요리를 시작할 때는 표시된 지점부터 이어갈 수 있습니다.
- **필요성:**
  - **리소스 관리:** 파일 스트림, 데이터베이스 커넥션 등은 사용 전에 열고 사용 후에는 반드시 닫아야 합니다. `ItemStream`은 이를 위한 `open()`과 `close()` 메서드를 제공합니다.
  - **상태 저장 및 재시작 (Restartability):** 대용량 배치 잡은 실행 시간이 길어 중간에 실패할 가능성이 있습니다. 처음부터 다시 시작하는 것은 매우 비효율적이므로, 실패 지점부터 다시 시작할 수 있는 기능이 중요합니다. `ItemStream`은 `update()` 메서드와 `ExecutionContext`를 통해 현재 처리 상태를 저장하고, `open()` 메서드에서 이 상태를 복원하여 재시작을 지원합니다.
- **"왜 필요한가?"**:
  - `ItemReader`와 `ItemWriter` 인터페이스 자체에는 리소스를 열고 닫거나 상태를 저장하는 표준적인 방법이 없습니다. `ItemStream`은 이러한 공통적인 관심사를 별도의 인터페이스로 분리하여 제공함으로써, 코드의 재사용성을 높이고 프레임워크가 일관된 방식으로 리소스와 상태를 관리할 수 있게 합니다.
  - 배치 잡의 견고성(robustness)과 신뢰성(reliability)을 높이는 데 필수적입니다.

### 2. `ItemStream` 인터페이스 살펴보기

```java
public interface ItemStream {

    void open(ExecutionContext executionContext) throws ItemStreamException;

    void update(ExecutionContext executionContext) throws ItemStreamException;

    void close() throws ItemStreamException;
}
```

- **`open(ExecutionContext executionContext)` 메서드:**
  - **호출 시점:** 해당 `ItemReader` 또는 `ItemWriter`가 스텝(Step)에서 사용되기 직전에, 즉 `read()`나 `write()`가 처음 호출되기 전에 단 한 번 호출됩니다.
  - **역할:**
    1. **리소스 초기화/열기:** 파일 스트림 열기, 데이터베이스 연결 획득, 필요한 설정 초기화 등.
    2. **상태 복원 (재시작 시):** `executionContext` 파라미터를 확인하여 이전에 저장된 상태가 있는지 확인합니다. 만약 있다면, 그 상태를 기반으로 내부 상태를 설정하여 중단된 지점부터 작업을 재개할 수 있도록 합니다. (예: 파일의 특정 라인 번호부터 읽기 시작, 데이터베이스 커서 특정 위치로 이동)
  - **`ItemStreamException`:** `open()` 작업 중 오류 발생 시 던져집니다.
- **`update(ExecutionContext executionContext)` 메서드:**
  - **호출 시점:** 주로 청크(Chunk) 처리가 완료되고 트랜잭션이 커밋되기 **직전**에 호출됩니다. 이는 현재까지의 처리 상태를 영속화하기 위함입니다.
  - **역할:**
    1. **현재 상태 저장:** `ItemReader`나 `ItemWriter`의 현재 내부 상태(예: 현재 읽은 아이템 수, 다음 처리할 오프셋 등)를 `executionContext` 파라미터에 저장합니다. 이렇게 저장된 정보는 잡이 실패 후 재시작될 때 `open()` 메서드에서 사용됩니다.
  - **`ItemStreamException`:** `update()` 작업 중 오류 발생 시 던져집니다.
- **`close()` 메서드:**
  - **호출 시점:** 해당 `ItemReader` 또는 `ItemWriter`의 사용이 완전히 종료될 때 (보통 스텝이 성공적으로 완료되거나 실패로 종료될 때) 단 한 번 호출됩니다.
  - **역할:**
    1. **리소스 해제/닫기:** `open()`에서 할당받거나 열었던 리소스(파일 스트림, DB 연결 등)를 안전하게 해제합니다. 리소스 누수(leak)를 방지하기 위해 매우 중요합니다.
  - **`ItemStreamException`:** `close()` 작업 중 오류 발생 시 던져집니다.

### 3. `ExecutionContext` 와의 관계

- **`ExecutionContext`란?**: Spring Batch에서 잡(Job) 또는 스텝(Step)의 실행 상태를 저장하는 데 사용되는 키-값 쌍의 컬렉션입니다. 마치 `Map<String, Object>`와 유사합니다.
  - `JobExecutionContext`: 잡 전체 범위에서 유효한 상태 저장소.
  - `StepExecutionContext`: 해당 스텝 범위에서 유효한 상태 저장소. `ItemStream`의 `open()`, `update()` 메서드에 전달되는 것이 바로 이 `StepExecutionContext`입니다.
- **`ItemStream`과 `ExecutionContext`의 상호작용 (재시작 시나리오):**
  1. **(최초 실행 시 `open`)**: `StepExecutionContext`는 비어있거나 초기 상태입니다. `ItemReader`는 처음부터 데이터를 읽기 시작합니다.
  2. **(`update`)**: 각 청크 처리 후, `ItemReader`는 현재까지 읽은 위치 정보(예: `processed.record.count = 100`)를 `StepExecutionContext`에 저장합니다. 이 정보는 데이터베이스(Spring Batch 메타데이터 테이블)에 영속화됩니다.
  3. **(잡 실패)**: 만약 150번째 아이템 처리 중 잡이 실패했다고 가정합니다.
  4. **(재시작 시 `open`)**: 동일한 `JobInstance`로 잡이 재시작되면, Spring Batch는 이전 실행의 `StepExecutionContext`를 데이터베이스에서 가져와 `open()` 메서드에 전달합니다. `ItemReader`는 `executionContext.getLong("processed.record.count")` 와 같이 저장된 값을 읽어, "아, 100번째까지는 이미 처리했구나. 그럼 101번째부터 시작해야지!"라고 판단하고 내부 상태를 조정합니다.
  5. 이후 정상적으로 `read()`, `update()`가 반복됩니다.
- **Quartz의 `JobDataMap`과의 유사성:** `ExecutionContext`는 Quartz 스케줄러에서 잡의 상태나 파라미터를 전달하는 데 사용되는 `JobDataMap`과 개념적으로 매우 유사합니다. 둘 다 실행 간에 상태를 유지하고 전달하는 메커니즘을 제공합니다.

---

### **주요 용어 해설**

- **상태 영속화 (Persisting State):** 프로그램의 현재 상태(데이터, 변수 값 등)를 비휘발성 저장소(주로 데이터베이스)에 저장하여, 프로그램이 종료되거나 시스템이 재시작되어도 해당 상태를 잃지 않고 나중에 복원할 수 있도록 하는 것. `ItemStream`과 `ExecutionContext`는 이를 위해 협력합니다.
- **리소스 누수 (Resource Leak):** 프로그램이 사용한 시스템 리소스(메모리, 파일 핸들, 네트워크 연결 등)를 제대로 해제하지 않아, 해당 리소스를 더 이상 사용할 수 없게 되거나 시스템 성능 저하를 유발하는 문제. `ItemStream`의 `close()` 메서드는 이를 방지하는 데 중요합니다.
- **`ItemStreamException`:** `ItemStream` 인터페이스의 메서드들(`open`, `update`, `close`) 실행 중 발생할 수 있는 예외의 기본 타입.

---

### **코드 예제 및 분석 (인터페이스 및 사용 흐름)**

```java
// ItemStream 인터페이스
public interface ItemStream {
    void open(ExecutionContext executionContext) throws ItemStreamException;
    void update(ExecutionContext executionContext) throws ItemStreamException;
    void close() throws ItemStreamException;
}

// ItemReader가 ItemStream을 구현하는 예시 (의사 코드)
public class MyFileReader implements ItemReader<String>, ItemStream { // 두 인터페이스 구현

    private BufferedReader reader;
    private String filePath;
    private ExecutionContext executionContext; // open에서 받은 executionContext 저장용
    private long lineCount = 0; // 현재까지 읽은 라인 수 (상태)
    private static final String LINE_COUNT_KEY = "myFileReader.lineCount"; // ExecutionContext 키

    public MyFileReader(String filePath) {
        this.filePath = filePath;
    }

    @Override
    public void open(ExecutionContext executionContext) throws ItemStreamException {
        this.executionContext = executionContext; // 나중에 update에서 사용하기 위해 저장
        try {
            File file = new File(filePath);
            this.reader = new BufferedReader(new FileReader(file));

            // 재시작 시 상태 복원
            if (executionContext.containsKey(LINE_COUNT_KEY)) {
                this.lineCount = executionContext.getLong(LINE_COUNT_KEY);
                // 이전에 읽은 라인만큼 건너뛰는 로직 (예시)
                for (long i = 0; i < this.lineCount; i++) {
                    if (reader.readLine() == null) { // 파일 끝에 도달하면 중단
                        break;
                    }
                }
                System.out.println("Resuming from line: " + (this.lineCount + 1));
            } else {
                this.lineCount = 0; // 처음 실행이면 0부터 시작
                System.out.println("Starting from beginning of file.");
            }
        } catch (IOException e) {
            throw new ItemStreamException("Failed to open file: " + filePath, e);
        }
    }

    @Override
    public String read() throws Exception, UnexpectedInputException, ParseException, NonTransientResourceException {
        if (reader == null) {
            throw new IllegalStateException("Reader not opened. Call open() first.");
        }
        String line = reader.readLine();
        if (line != null) {
            lineCount++; // 실제 아이템을 읽었을 때만 카운트 증가 (update와 동기화)
        }
        return line; // line이 null이면 데이터 끝
    }

    @Override
    public void update(ExecutionContext executionContext) throws ItemStreamException {
        // 현재까지 성공적으로 처리(read)된 lineCount를 저장
        executionContext.putLong(LINE_COUNT_KEY, this.lineCount);
        System.out.println("State updated. Current line count: " + this.lineCount);
    }

    @Override
    public void close() throws ItemStreamException {
        if (reader != null) {
            try {
                reader.close();
                System.out.println("File closed.");
            } catch (IOException e) {
                throw new ItemStreamException("Failed to close file reader", e);
            }
        }
    }
}

```

- **`MyFileReader`**: `ItemReader<String>`과 `ItemStream`을 모두 구현합니다.
- **`open()`**:
  - 파일을 열고 `BufferedReader`를 생성합니다.
  - `ExecutionContext`에 `LINE_COUNT_KEY`로 저장된 값이 있는지 확인합니다.
  - 값이 있다면 (재시작 상황), 해당 `lineCount`만큼 파일의 라인을 건너뛰어 중단된 지점부터 읽도록 준비합니다.
  - 값이 없다면 (최초 실행), `lineCount`를 0으로 초기화합니다.
- **`read()`**:
  - 파일에서 한 줄을 읽습니다.
  - 라인을 성공적으로 읽으면 내부 `lineCount`를 증가시킵니다.
  - 읽은 라인 (또는 파일 끝이면 `null`)을 반환합니다.
- **`update()`**:
  - 현재 `lineCount` 값을 `LINE_COUNT_KEY`와 함께 `ExecutionContext`에 저장합니다. 이 작업은 보통 청크 커밋 직전에 호출되어, 여기까지의 진행 상황이 안전하게 저장되도록 합니다.
- **`close()`**:
  - `BufferedReader`를 닫아 파일 리소스를 해제합니다.

**실행 흐름 (Spring Batch 프레임워크가 호출)**

1. 스텝 시작 -> `MyFileReader.open(executionContext)` 호출
2. (반복)
  1. `MyFileReader.read()` 호출 (청크 크기만큼 반복 또는 `null` 반환까지)
  2. (만약 `ItemProcessor`가 있다면 processor 호출)
  3. (청크 완성)
  4. `MyFileWriter.write(chunk)` 호출 (가상의 `ItemWriter`)
  5. `MyFileReader.update(executionContext)` 호출 (커밋 전 상태 저장)
  6. `MyFileWriter.update(executionContext)` 호출 (커밋 전 상태 저장)
  7. (트랜잭션 커밋)
3. 스텝 종료 -> `MyFileReader.close()` 호출
4. 스텝 종료 -> `MyFileWriter.close()` 호출

---

### **"왜?" 라는 질문에 대한 답변**

- **왜 `ItemReader`와 `ItemWriter` 인터페이스에 `open/update/close`가 직접 포함되지 않고 별도의 `ItemStream` 인터페이스로 분리되었을까?**
  - **선택적 구현 (Optional Implementation):** 모든 `ItemReader`나 `ItemWriter`가 리소스 관리나 상태 저장이 필요한 것은 아닙니다. 예를 들어, 메모리 내의 리스트에서 데이터를 읽는 간단한 `ItemReader`는 `ItemStream` 기능이 필요 없을 수 있습니다. 인터페이스를 분리함으로써, 필요한 경우에만 `ItemStream`을 구현하도록 하여 유연성을 높입니다.
  - **단일 책임 원칙 (Single Responsibility Principle):** `ItemReader`는 "읽기"에, `ItemWriter`는 "쓰기"에 집중하고, `ItemStream`은 "리소스 관리 및 상태 영속화"라는 별도의 책임에 집중하도록 역할을 분담할 수 있습니다.
  - **프레임워크의 일관된 처리:** Spring Batch 프레임워크는 빈이 `ItemStream` 타입인지 확인하고, 그렇다면 생명주기에 맞춰 `open`, `update`, `close`를 자동으로 호출해줍니다. 이를 통해 개발자는 반복적인 보일러플레이트 코드를 줄일 수 있습니다.
- **`update()`는 왜 커밋 전에 호출될까?**
  - 만약 `update()`가 커밋 *후에* 호출된다고 가정해봅시다. `update()`에서 `ExecutionContext`에 상태를 저장하는 도중 오류가 발생하면, 데이터는 이미 커밋되었지만 상태는 제대로 저장되지 않는 불일치가 발생할 수 있습니다.
  - 커밋 *전에* `update()`를 호출하여 상태를 `ExecutionContext`에 먼저 기록하고, 이 `ExecutionContext`의 변경사항이 트랜잭션과 함께 (Spring Batch 메타데이터 테이블에) 커밋되도록 하면, 데이터 처리와 상태 저장이 원자적으로 이루어지는 효과를 낼 수 있습니다. 즉, "여기까지 처리했고, 그 상태도 확실히 저장했어"라는 것을 보장하기 위함입니다.

---

### **주의사항 및 Best Practice**

1. **`ItemStream` 구현 시 세 메서드 모두 구현:** `ItemStream`을 구현한다면 `open()`, `update()`, `close()` 세 메서드 모두 의미에 맞게 제대로 구현해야 합니다. 특히 `close()`에서 리소스 해제를 누락하면 심각한 문제를 야기할 수 있습니다.
2. **`ExecutionContext` 키 이름 충돌 주의:** `ExecutionContext`에 상태를 저장할 때 사용하는 키 이름이 다른 `ItemStream` 구현체나 스텝 내 다른 로직에서 사용하는 키와 충돌하지 않도록 고유한 이름을 사용하는 것이 좋습니다. (예: `[빈이름].[상태변수명]`)
3. **`update()`는 주기적으로 호출됨:** `update()`는 각 청크 처리 후 커밋 전에 호출되므로, 너무 많은 데이터를 `ExecutionContext`에 자주 저장하면 성능에 부담을 줄 수 있습니다. 꼭 필요한 최소한의 상태 정보만 저장하는 것이 좋습니다.
4. **멱등성(Idempotency) 고려:** `open()`이나 `close()` 같은 메서드는 여러 번 호출되어도 문제가 없도록 멱등성을 가지도록 설계하는 것이 안전할 수 있습니다 (프레임워크가 정확히 한 번씩 호출하지만, 예기치 않은 상황을 대비). 예를 들어, `close()`에서 이미 닫힌 리소스를 또 닫으려고 할 때 오류가 발생하지 않도록 처리하는 것입니다.
5. **`ItemStream` 등록:** Spring Batch는 스텝에 등록된 `ItemReader`와 `ItemWriter` (그리고 `ItemProcessor`도 `ItemStream`을 구현할 수 있음)가 `ItemStream` 인터페이스를 구현하고 있는지 자동으로 감지하고, 해당 메서드들을 호출해줍니다. 별도로 "등록"하는 절차는 필요 없습니다.

---

### **이전 학습 내용과의 연관성**

- **`ItemReader` 및 `ItemWriter`:** `ItemStream`은 이 두 인터페이스의 기능을 보조하고 확장하는 역할을 합니다. `ItemReader`나 `ItemWriter`가 상태를 가져야 하거나 외부 리소스를 사용해야 할 때 `ItemStream`을 함께 구현합니다.
- **`@StepScope`:** `ItemStream`을 구현하는 컴포넌트들은 대부분 상태를 가지므로, 각 스텝 실행마다 독립적인 인스턴스를 갖도록 `@StepScope`로 선언하는 것이 일반적입니다. 이를 통해 상태가 다른 스텝 실행에 영향을 주지 않도록 격리할 수 있습니다.
- **`ExecutionContext` (지연 바인딩 챕터에서도 언급):** 지연 바인딩에서는 `ExecutionContext`의 값을 읽어와 빈의 속성을 설정하는 데 사용했다면, `ItemStream`에서는 반대로 빈의 현재 상태를 `ExecutionContext`에 기록하여 영속화하는 데 주로 사용합니다. 물론, `open()` 시에는 값을 읽어와 복원하는 데도 사용합니다.

---
