---
title: Spring Batch - JSON Item Readers and Writers
description: 
author: laze
date: 2025-06-18 00:00:02 +0900
categories: [Dev, SpringBatch]
tags: [SpringBatch]
---
### JSON Item Readers And Writers

Spring Batch는 다음 형식의 JSON 리소스를 읽고 쓰는 기능을 지원합니다:

```json
[
  {
    "isin": "123",
    "quantity": 1,
    "price": 1.2,
    "customer": "foo"
  },
  {
    "isin": "456",
    "quantity": 2,
    "price": 1.4,
    "customer": "bar"
  }
]
```

JSON 리소스는 개별 아이템에 해당하는 JSON 객체의 배열이라고 가정합니다.

Spring Batch는 특정 JSON 라이브러리에 얽매이지 않습니다.

### JsonItemReader

`JsonItemReader`는 JSON 파싱 및 바인딩을 `org.springframework.batch.item.json.JsonObjectReader` 인터페이스의 구현체에 위임합니다.

이 인터페이스는 스트리밍 API를 사용하여 JSON 객체를 청크 단위로 읽도록 구현될 것을 의도합니다.

현재 두 가지 구현체가 제공됩니다:

- `org.springframework.batch.item.json.JacksonJsonObjectReader`를 통한 Jackson
- `org.springframework.batch.item.json.GsonJsonObjectReader`를 통한 Gson

JSON 레코드를 처리할 수 있으려면 다음이 필요합니다:

- **리소스(Resource):** 읽을 JSON 파일을 나타내는 Spring `Resource`입니다.
- **JsonObjectReader:** JSON 객체를 아이템으로 파싱하고 바인딩하는 JSON 객체 리더입니다.

다음 예제는 이전 JSON 리소스 `org/springframework/batch/item/json/trades.json`과 Jackson 기반의 `JsonObjectReader`로 작동하는 `JsonItemReader`를 정의하는 방법을 보여줍니다:

```java
@Bean
public JsonItemReader<Trade> jsonItemReader() {
   return new JsonItemReaderBuilder<Trade>()
                 .jsonObjectReader(new JacksonJsonObjectReader<>(Trade.class)) // Jackson 리더 사용, Trade 클래스로 바인딩
                 .resource(new ClassPathResource("trades.json")) // 입력 JSON 파일 리소스
                 .name("tradeJsonItemReader") // ItemReader 이름
                 .build();
}
```

### JsonFileItemWriter

`JsonFileItemWriter`는 아이템의 마샬링을 `org.springframework.batch.item.json.JsonObjectMarshaller` 인터페이스에 위임합니다.

이 인터페이스의 계약은 객체를 받아 JSON 문자열로 마샬링하는 것입니다. 현재 두 가지 구현체가 제공됩니다:

- `org.springframework.batch.item.json.JacksonJsonObjectMarshaller`를 통한 Jackson
- `org.springframework.batch.item.json.GsonJsonObjectMarshaller`를 통한 Gson

JSON 레코드를 쓸 수 있으려면 다음이 필요합니다:

- **리소스(Resource):** 쓸 JSON 파일을 나타내는 Spring `Resource`입니다.
- **JsonObjectMarshaller:** 객체를 JSON 형식으로 마샬링하는 JSON 객체 마샬러입니다.

다음 예제는 `JsonFileItemWriter`를 정의하는 방법을 보여줍니다:

```java
@Bean
public JsonFileItemWriter<Trade> jsonFileItemWriter(Resource outputResource) { // Resource 주입 방식 선호
   return new JsonFileItemWriterBuilder<Trade>()
                 .jsonObjectMarshaller(new JacksonJsonObjectMarshaller<>()) // Jackson 마샬러 사용
                 .resource(outputResource) // 출력 JSON 파일 리소스
                 .name("tradeJsonFileItemWriter") // ItemWriter 이름
                 .build();
}
```

---

### **학습 목표 제시**

이번 "JSON Item Readers And Writers" 챕터를 통해 학생분은 다음을 이해하고 배울 수 있습니다:

1. **Spring Batch JSON 처리의 기본 형식 및 라이브러리 독립성 이해:** Spring Batch가 기대하는 JSON 입력 형식(객체 배열)을 인지하고, 특정 JSON 라이브러리(Jackson, Gson 등)에 종속되지 않고 유연하게 처리할 수 있음을 이해합니다.
2. **`JsonItemReader`를 사용한 JSON 읽기 방법 습득:** `JsonObjectReader` (예: `JacksonJsonObjectReader`)를 사용하여 JSON 파일로부터 데이터를 읽고 자바 객체로 변환하는 과정을 이해하고 설정하는 방법을 익힙니다.
3. **`JsonFileItemWriter`를 사용한 JSON 쓰기 방법 습득:** `JsonObjectMarshaller` (예: `JacksonJsonObjectMarshaller`)를 사용하여 자바 객체를 JSON 형식으로 변환하고 파일에 쓰는 과정을 이해하고 설정하는 방법을 익힙니다.

---

### **핵심 개념 설명**

이 챕터의 핵심은 **"JSON 형식의 데이터를 어떻게 읽고 자바 객체로 변환하며, 반대로 자바 객체를 어떻게 JSON 형식으로 변환하여 파일에 쓸 것인가?"** 입니다. 이를 위해 Spring Batch는 Jackson이나 Gson과 같은 유명한 JSON 라이브러리를 활용할 수 있는 추상화 계층을 제공합니다.

### 1. Spring Batch가 기대하는 JSON 형식

- Spring Batch의 JSON 리더/라이터는 기본적으로 **JSON 객체들의 배열(JSON array of JSON objects)** 형식을 기대합니다.
- 각 JSON 객체는 하나의 아이템(레코드)에 해당합니다.
- 예시 (문서 제공 형식):

    ```json
    [  // 배열 시작
      {  // 첫 번째 아이템 (JSON 객체)
        "isin": "123",
        "quantity": 1,
        "price": 1.2,
        "customer": "foo"
      },
      {  // 두 번째 아이템 (JSON 객체)
        "isin": "456",
        "quantity": 2,
        "price": 1.4,
        "customer": "bar"
      }
    ]  // 배열 끝
    ```


### 2. 라이브러리 독립성

- Spring Batch 자체는 특정 JSON 처리 라이브러리(예: Jackson, Gson)에 직접적으로 묶여있지 않습니다.
- 대신, `JsonObjectReader` (읽기용) 와 `JsonObjectMarshaller` (쓰기용) 라는 **자체 인터페이스**를 제공하고, 이 인터페이스의 구현체를 통해 실제 JSON 처리를 위임합니다.
- 이를 통해 개발자는 자신이 선호하거나 프로젝트에서 이미 사용 중인 JSON 라이브러리(Jackson 또는 Gson)를 선택하여 Spring Batch와 함께 사용할 수 있는 유연성을 갖게 됩니다. (현재 Jackson과 Gson 구현체가 기본 제공됩니다.)

### 3. `JsonItemReader` (JSON 읽기)

- **역할:** JSON 파일로부터 데이터를 읽어와 각 JSON 객체를 지정된 자바 객체로 변환하여 제공하는 `ItemReader` 구현체입니다.
- **핵심 구성 요소:**
  1. **`resource` (Resource)**: 읽어올 JSON 파일 리소스.
  2. **`jsonObjectReader` (org.springframework.batch.item.json.JsonObjectReader)**: 실제 JSON 파싱 및 객체 바인딩(변환)을 담당하는 인터페이스.
    - 이 인터페이스는 스트리밍 API를 사용하여 JSON 객체를 효율적으로(청크 단위로) 읽도록 설계되었습니다.
- **제공되는 `JsonObjectReader` 구현체:**
  - **`JacksonJsonObjectReader<T>`**: Jackson 라이브러리를 사용하여 JSON을 객체 `T`로 변환합니다. 생성 시 대상 클래스(`T.class`)를 지정해야 합니다.
  - **`GsonJsonObjectReader<T>`**: Gson 라이브러리를 사용하여 JSON을 객체 `T`로 변환합니다. 마찬가지로 대상 클래스 지정이 필요합니다.
- **설정 예시 (`JsonItemReaderBuilder` 사용):**

    ```java
    @Bean
    public JsonItemReader<Trade> tradeJsonItemReader(
            @Value("classpath:input/trades.json") Resource inputResource) { // 리소스 주입
    
       // Jackson 라이브러리를 사용하여 JSON을 Trade 객체로 변환할 JsonObjectReader 생성
       JacksonJsonObjectReader<Trade> jsonObjectReader = new JacksonJsonObjectReader<>(Trade.class);
       // jsonObjectReader.setMapper(customObjectMapper); // (선택) 커스텀 ObjectMapper 설정 가능
    
       return new JsonItemReaderBuilder<Trade>()
                     .name("tradeJsonItemReader")       // Reader 이름
                     .jsonObjectReader(jsonObjectReader) // 위에서 만든 JsonObjectReader 설정
                     .resource(inputResource)           // 읽을 JSON 파일
                     .build();
    }
    ```

  - `JacksonJsonObjectReader<>(Trade.class)`: JSON 객체를 `Trade` 클래스의 인스턴스로 변환하도록 Jackson 리더를 설정합니다. Jackson은 필드 이름(JSON 키)과 클래스의 프로퍼티(게터/세터 또는 필드)를 자동으로 매칭하려고 시도합니다. (필요시 Jackson의 어노테이션 `@JsonProperty` 등으로 매핑 조정 가능)

### 4. `JsonFileItemWriter` (JSON 쓰기)

- **역할:** 자바 객체(아이템)들을 받아 각 객체를 JSON 형식의 문자열로 변환하고, 이를 지정된 파일에 기록하는 `ItemWriter` 구현체입니다. 출력 형식도 기본적으로 JSON 객체의 배열입니다.
- **핵심 구성 요소:**
  1. **`resource` (Resource)**: 데이터를 기록할 출력 JSON 파일 리소스.
  2. **`jsonObjectMarshaller` (org.springframework.batch.item.json.JsonObjectMarshaller)**: 자바 객체를 JSON 문자열로 변환(마샬링)하는 역할을 담당하는 인터페이스.
- **제공되는 `JsonObjectMarshaller` 구현체:**
  - **`JacksonJsonObjectMarshaller<T>`**: Jackson 라이브러리를 사용하여 객체 `T`를 JSON 문자열로 변환합니다.
  - **`GsonJsonObjectMarshaller<T>`**: Gson 라이브러리를 사용하여 객체 `T`를 JSON 문자열로 변환합니다.
- **설정 예시 (`JsonFileItemWriterBuilder` 사용):**

    ```java
    @Bean
    public JsonFileItemWriter<Trade> tradeJsonFileItemWriter(
            @Value("file:output/output_trades.json") Resource outputResource) { // 리소스 주입
    
        // Jackson 라이브러리를 사용하여 Trade 객체를 JSON 문자열로 변환할 JsonObjectMarshaller 생성
        JacksonJsonObjectMarshaller<Trade> jsonObjectMarshaller = new JacksonJsonObjectMarshaller<>();
        // jsonObjectMarshaller.setObjectMapper(customObjectMapper); // (선택) 커스텀 ObjectMapper 설정 가능
    
        return new JsonFileItemWriterBuilder<Trade>()
                     .name("tradeJsonFileItemWriter")         // Writer 이름
                     .jsonObjectMarshaller(jsonObjectMarshaller) // 위에서 만든 JsonObjectMarshaller 설정
                     .resource(outputResource)                // 출력할 JSON 파일
                     // .lineSeparator("\\n") // (선택) 기본값은 시스템 줄바꿈, JSON 배열 형식 유지를 위해 보통 기본값 사용
                     // .headerCallback(...) // (선택) 배열 시작 '[' 문자 쓰기
                     // .footerCallback(...) // (선택) 배열 끝 ']' 문자 쓰기
                     // .append(false) // (선택) 기본적으로 덮어쓰기
                     .build();
    }
    ```

  - `JacksonJsonObjectMarshaller<>()`: 객체를 JSON으로 변환할 때 Jackson 마샬러를 사용합니다. Jackson은 객체의 필드를 기반으로 JSON을 생성합니다.
  - **출력 형식 제어:** `JsonFileItemWriter`는 기본적으로 각 JSON 객체를 쓰고 그 사이에 쉼표와 줄바꿈을 추가하여 JSON 배열 형식을 만듭니다. 배열의 시작(`[`)과 끝(`]`) 문자는 `headerCallback`과 `footerCallback` (또는 `JsonFileItemWriter`의 `JsonFileItemWriterBuilder.headerCallback`/`footerCallback` 통해 간접적으로 설정 가능하거나, 내부적으로 관리)을 통해 처리될 수 있습니다. (Spring Batch 5부터는 빌더에서 `.transactional(false)` 등으로 설정 시 내부적으로 JSON 배열 포맷을 더 잘 관리해주는 경향이 있습니다.)

---

### **주요 용어 해설**

- **JSON (JavaScript Object Notation):** 속성-값 쌍 또는 정렬된 리스트(배열)로 이루어진, 사람이 읽기 쉬운 텍스트 기반의 데이터 교환 형식.
- **JSON 객체 (JSON Object):** 중괄호(`{}`)로 둘러싸이며, 쉼표로 구분된 0개 이상의 이름/값 쌍(필드)들의 순서 없는 집합. (예: `{"name": "John", "age": 30}`)
- **JSON 배열 (JSON Array):** 대괄호(`[]`)로 둘러싸이며, 쉼표로 구분된 0개 이상의 값들의 순서 있는 리스트. 값은 문자열, 숫자, 불리언, 객체, 배열 등이 될 수 있음. (예: `[1, "apple", {"id": 100}]`)
- **Jackson:** 매우 인기 있는 고성능 Java용 JSON 처리 라이브러리. 데이터 바인딩, 트리 모델, 스트리밍 API 등 다양한 기능 제공.
- **Gson:** Google에서 개발한 Java용 JSON 처리 라이브러리. 사용이 간편하고 강력한 기능을 제공.
- **마샬링/언마샬링 (Marshalling/Unmarshalling - JSON 문맥에서):**
  - JSON 마샬링: 자바 객체 -> JSON 문자열 변환.
  - JSON 언마샬링: JSON 문자열 -> 자바 객체 변환. (XML에서의 용어와 유사)

---

### **코드 예제 분석**

### `JsonItemReader` 설정 예시 (문서 코드 기반)

```java
// Trade.java (도메인 객체 - XML 예제와 유사하게 필드 가정)
// public class Trade {
//    private String isin;
//    private int quantity; // 또는 Long
//    private double price;  // 또는 BigDecimal
//    private String customer;
//    // Getters and Setters
// }

@Configuration
public class JsonReaderConfig {

    @Bean
    public JsonItemReader<Trade> tradeJsonItemReader(
            @Value("classpath:com/example/trades.json") Resource inputJsonResource) { // 리소스 경로 예시

        // Jackson 라이브러리를 사용하여 Trade.class 타입으로 JSON 객체를 변환
        JacksonJsonObjectReader<Trade> jsonObjectReader = new JacksonJsonObjectReader<>(Trade.class);

        // (선택 사항) 만약 Jackson ObjectMapper에 특별한 설정을 하고 싶다면:
        // ObjectMapper objectMapper = new ObjectMapper();
        // objectMapper.configure(DeserializationFeature.FAIL_ON_UNKNOWN_PROPERTIES, false); // 알 수 없는 프로퍼티 무시
        // jsonObjectReader.setMapper(objectMapper);

        return new JsonItemReaderBuilder<Trade>()
                .name("tradeJsonItemReader")
                .resource(inputJsonResource)           // 읽을 JSON 파일
                .jsonObjectReader(jsonObjectReader) // 사용할 JsonObjectReader (Jackson 기반)
                .build();
    }
}
```

- `JacksonJsonObjectReader<>(Trade.class)`: 이 부분이 핵심입니다. Jackson에게 "읽어들인 각 JSON 객체를 `Trade` 클래스의 인스턴스로 만들어줘"라고 알려주는 것입니다. Jackson은 `Trade` 클래스의 필드(또는 게터/세터) 이름과 JSON 객체의 키 이름을 비교하여 자동으로 값을 매핑합니다.
  - 예를 들어 JSON에 `{"isin": "XYZ"}`가 있다면, Jackson은 `Trade` 객체의 `isin` 필드에 "XYZ"를 설정하려고 합니다.

### `JsonFileItemWriter` 설정 예시 (문서 코드 기반)

```java
@Configuration
public class JsonWriterConfig {

    @Bean
    public JsonFileItemWriter<Trade> tradeJsonFileItemWriter(
            @Value("file:output/processed_trades.json") Resource outputJsonResource) { // 리소스 경로 예시

        JacksonJsonObjectMarshaller<Trade> jsonObjectMarshaller = new JacksonJsonObjectMarshaller<>();

        // (선택 사항) 만약 Jackson ObjectMapper에 특별한 설정을 하고 싶다면:
        // ObjectMapper objectMapper = new ObjectMapper();
        // objectMapper.enable(SerializationFeature.INDENT_OUTPUT); // 보기 좋게 들여쓰기 된 JSON 출력
        // jsonObjectMarshaller.setObjectMapper(objectMapper);

        return new JsonFileItemWriterBuilder<Trade>()
                .name("tradeJsonFileItemWriter")
                .resource(outputJsonResource)            // 출력할 JSON 파일
                .jsonObjectMarshaller(jsonObjectMarshaller) // 사용할 JsonObjectMarshaller (Jackson 기반)
                // .shouldDeleteIfExists(true) // (선택) 파일이 존재하면 덮어쓰기 전 삭제
                .build();
    }
}
```

- `JacksonJsonObjectMarshaller<>()`: `Trade` 객체를 JSON 문자열로 변환할 때 Jackson을 사용하겠다는 의미입니다.
- 출력 파일은 기본적으로 다음과 같은 형식을 가집니다 (별도의 헤더/푸터 콜백 설정이 없다면):
  각 객체가 JSON으로 변환되고, 그 사이에 쉼표가 들어가며, 전체가 대괄호(`[]`)로 묶인 배열 형태가 됩니다.

    ```json
    [
    {"isin":"XYZ0001", ...},
    {"isin":"XYZ0002", ...},
    {"isin":"XYZ0003", ...}
    ]
    ```


---

### **"왜?" 라는 질문에 대한 답변**

- **왜 Spring Batch는 자체 JSON 파서를 만들지 않고 Jackson이나 Gson 같은 외부 라이브러리를 사용할까?**
  - **성숙도와 기능 풍부성:** Jackson과 Gson은 이미 매우 성숙하고, 광범위하게 테스트되었으며, 수많은 기능을 (커스텀 직렬화/역직렬화, 어노테이션 기반 매핑, 트리 모델, 성능 최적화 등) 제공하는 라이브러리입니다. Spring Batch가 이를 처음부터 다시 만드는 것은 비효율적입니다.
  - **생태계 호환성:** 많은 자바 개발자들이 이미 Jackson이나 Gson에 익숙하고, 기존 프로젝트에서도 널리 사용되고 있습니다. 이를 그대로 활용함으로써 학습 곡선을 낮추고 기존 코드와의 통합을 용이하게 합니다.
  - **유지보수 및 발전:** 외부 라이브러리는 해당 커뮤니티나 개발 주체에 의해 지속적으로 발전하고 유지보수됩니다. Spring Batch는 이러한 발전에 편승할 수 있습니다.
  - **Spring의 철학 (위임):** 스프링 프레임워크는 종종 특정 기능에 대해 잘 만들어진 외부 라이브러리가 있다면, 이를 직접 구현하기보다는 추상화 계층을 통해 위임하는 방식을 선호합니다. Spring OXM이 그 예시이고, JSON 처리도 유사한 접근 방식을 취합니다.
- **XML 처리와 JSON 처리의 유사점과 차이점은?**
  - **유사점:**
    - 둘 다 구조화된 데이터를 표현합니다.
    - Spring Batch는 둘 다 스트리밍 방식으로 처리하려고 노력합니다 (XML은 StAX, JSON은 `JsonObjectReader`의 스트리밍 API 구현).
    - 객체 <-> 데이터 변환을 위해 외부 라이브러리/기술(OXM, Jackson/Gson)을 활용하는 추상화 계층을 제공합니다.
    - `ItemReader`와 `ItemWriter` 인터페이스를 통해 일관된 방식으로 배치 처리 흐름에 통합됩니다.
  - **차이점:**
    - **구문과 구조:** XML은 태그 기반의 계층적 구조, 네임스페이스, 속성 등을 가지는 반면, JSON은 키-값 쌍의 객체와 값의 배열로 더 단순한 구조를 가집니다.
    - **파싱/매핑 메커니즘:** XML은 `Unmarshaller`/`Marshaller` (OXM)를, JSON은 `JsonObjectReader`/`JsonObjectMarshaller` (Jackson/Gson 기반)를 사용합니다. 설정 방식이나 매핑 전략에 차이가 있을 수 있습니다. (예: XML은 별칭/어노테이션, JSON은 필드명 매칭/어노테이션)
    - **데이터 표현 단위:** XML은 "프래그먼트"라는 개념을 사용했지만, JSON은 더 직관적으로 "JSON 객체"를 하나의 아이템 단위로 봅니다.

---

### **주의사항 및 Best Practice**

1. **JSON 라이브러리 의존성 추가:** 프로젝트에서 Jackson이나 Gson 중 사용할 라이브러리의 의존성을 `pom.xml`이나 `build.gradle`에 추가해야 합니다. (예: `spring-boot-starter-json`은 Jackson을 기본으로 포함)
2. **객체-JSON 필드 매핑:**
  - 기본적으로 Jackson/Gson은 자바 객체의 필드명(또는 getter/setter에서 파생된 프로퍼티명)과 JSON 객체의 키를 이름으로 매칭합니다.
  - 이름이 다르거나 특별한 처리가 필요하면 각 라이브러리가 제공하는 어노테이션(예: Jackson의 `@JsonProperty("json_key_name")`, `@JsonIgnore` 등)을 활용할 수 있습니다.
  - 날짜 형식, 특정 값의 (비)직렬화 등 세부적인 제어는 `ObjectMapper`(Jackson)나 `GsonBuilder`(Gson)를 커스터마이징하여 `JsonObjectReader/Marshaller`에 설정함으로써 가능합니다.
3. **JSON 파일 형식 확인:** `JsonItemReader`는 기본적으로 JSON 객체의 배열(`[{}, {}, ...]`)을 기대합니다. 만약 파일이 단일 JSON 객체이거나, 각 줄이 독립적인 JSON 객체인 경우(JSON Lines 형식) 등 다른 형식이면 추가적인 설정이나 커스텀 리더가 필요할 수 있습니다.
4. **`JsonFileItemWriter` 출력 형식:** 기본적으로 JSON 배열 형식을 만듭니다. `headerCallback`으로 `[`를, `footerCallback`으로 `]`를, 그리고 각 아이템 사이에 쉼표를 찍어주는 로직이 내장되어 있거나 빌더를 통해 쉽게 설정 가능합니다. (Spring Batch 버전에 따라 내부 구현이 약간씩 개선됨)

---

### **이전 학습 내용과의 연관성**

- **`ItemReader` / `ItemWriter` 인터페이스:** `JsonItemReader`와 `JsonFileItemWriter`는 이들 인터페이스의 JSON 특화 구현체입니다.
- **`ItemStream` 인터페이스:** 이들도 `ItemStream`을 구현하여 파일 리소스 관리와 (필요시) 상태 저장을 지원합니다.
- **`Resource` (스프링 코어):** JSON 파일 위치를 지정하는 데 사용됩니다.
- **빌더 패턴:** `JsonItemReaderBuilder`와 `JsonFileItemWriterBuilder`를 통해 간결하고 가독성 높게 객체를 설정할 수 있습니다.
- **XML 처리와의 비교:** 데이터 형식은 다르지만, "외부 데이터 -> 객체" 또는 "객체 -> 외부 데이터"라는 배치 처리의 기본 흐름과 추상화 개념은 유사합니다.

---
