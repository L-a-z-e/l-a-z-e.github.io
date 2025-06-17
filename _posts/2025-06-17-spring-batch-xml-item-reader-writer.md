---
title: Spring Batch - XML Item Readers and Writers
description: 
author: laze
date: 2025-06-17 00:00:02 +0900
categories: [Dev, SpringBatch]
tags: [SpringBatch]
---
### XML Item Readers and Writers

Spring Batch는 XML 레코드를 읽고 이를 Java 객체로 매핑하는 것과 Java 객체를 XML 레코드로 작성하는 것 모두에 대한 트랜잭션 인프라를 제공합니다.

### 스트리밍 XML의 제약 조건 (Constraints on streaming XML)

다른 표준 XML 파싱 API는 배치 처리 요구 사항에 맞지 않기 때문에 (DOM은 전체 입력을 한 번에 메모리에 로드하고 SAX는 사용자가 콜백만 제공하도록 허용하여 파싱 프로세스를 제어함) I/O에는 StAX API가 사용됩니다.

Spring Batch에서 XML 입력 및 출력이 어떻게 작동하는지 고려해야 합니다.

첫째, 파일 읽기 및 쓰기와는 다르지만 Spring Batch XML 처리 전반에 걸쳐 공통적인 몇 가지 개념이 있습니다.

XML 처리에서는 토큰화해야 하는 레코드 라인(`FieldSet` 인스턴스) 대신, XML 리소스가 다음 이미지와 같이 개별 레코드에 해당하는 '프래그먼트(fragment)'의 컬렉션이라고 가정합니다:

**XML 입력**
![Desktop View](assets/img/xml_item_reader_writer_1.png){: width="972" height="589" }

위 시나리오에서 'trade' 태그는 '루트 요소(root element)'로 정의됩니다.

`<trade>`와 `</trade>` 사이의 모든 것은 하나의 '프래그먼트'로 간주됩니다.

Spring Batch는 프래그먼트를 객체에 바인딩하기 위해 Object/XML 매핑(OXM)을 사용합니다.

그러나 Spring Batch는 특정 XML 바인딩 기술에 얽매이지 않습니다.

일반적인 사용은 가장 인기 있는 OXM 기술에 대한 균일한 추상화를 제공하는 Spring OXM에 위임하는 것입니다.

Spring OXM에 대한 의존성은 선택 사항이며, 원하는 경우 Spring Batch 특정 인터페이스를 구현하도록 선택할 수 있습니다.

OXM이 지원하는 기술과의 관계는 다음 이미지와 같습니다:

**OXM 바인딩**

![Desktop View](assets/img/xml_item_reader_writer_2.png){: width="972" height="589" }

OXM과 XML 프래그먼트를 사용하여 레코드를 나타내는 방법에 대한 소개와 함께, 이제 리더와 라이터를 더 자세히 살펴볼 수 있습니다.

### StaxEventItemReader

`StaxEventItemReader` 구성은 XML 입력 스트림에서 레코드를 처리하기 위한 일반적인 설정을 제공합니다.

```xml
<?xml version="1.0" encoding="UTF-8"?>
<records>
    <trade xmlns="<https://springframework.org/batch/sample/io/oxm/domain>">
        <isin>XYZ0001</isin>
        <quantity>5</quantity>
        <price>11.39</price>
        <customer>Customer1</customer>
    </trade>
    <trade xmlns="<https://springframework.org/batch/sample/io/oxm/domain>">
        <isin>XYZ0002</isin>
        <quantity>2</quantity>
        <price>72.99</price>
        <customer>Customer2c</customer>
    </trade>
    <trade xmlns="<https://springframework.org/batch/sample/io/oxm/domain>">
        <isin>XYZ0003</isin>
        <quantity>9</quantity>
        <price>99.99</price>
        <customer>Customer3</customer>
    </trade>
</records>
```

XML 레코드를 처리할 수 있으려면 다음이 필요합니다:

- **루트 요소 이름(Root Element Name):** 매핑할 객체를 구성하는 프래그먼트의 루트 요소 이름입니다. 예제 구성은 `trade` 값으로 이를 보여줍니다.
- **리소스(Resource):** 읽을 파일을 나타내는 Spring `Resource`입니다.
- **언마샬러(Unmarshaller):** XML 프래그먼트를 객체로 매핑하기 위해 Spring OXM에서 제공하는 언마샬링 기능입니다.

다음 예제는 `trade`라는 루트 요소, `data/iosample/input/input.xml` 리소스, 그리고 `tradeMarshaller`라는 언마샬러로 작동하는 `StaxEventItemReader`를 Java에서 정의하는 방법을 보여줍니다:

**Java Configuration**

```java
@Bean
public StaxEventItemReader<Trade> itemReader(Resource tradesXml, Unmarshaller tradeMarshaller) { // Resource와 Unmarshaller 주입
	return new StaxEventItemReaderBuilder<Trade>()
			.name("itemReader") // ItemReader 이름
			.resource(tradesXml) // 입력 XML 파일 리소스
			.addFragmentRootElements("trade") // 각 레코드(프래그먼트)의 루트 요소 이름
			.unmarshaller(tradeMarshaller) // 사용할 Unmarshaller (OXM)
			.build();
}
```

이 예제에서는 `XStreamMarshaller`를 사용하기로 선택했으며, 이는 첫 번째 키와 값이 프래그먼트의 이름(즉, 루트 요소)이고 객체 유형이 바인딩될 맵으로 전달된 별칭(alias)을 허용합니다.

그런 다음, `FieldSet`과 유사하게, 객체 유형 내의 필드에 매핑되는 다른 요소의 이름이 맵의 키/값 쌍으로 설명됩니다.

구성 파일에서는 필요한 별칭을 설명하기 위해 Spring 구성 유틸리티를 사용할 수 있습니다.
**Java Configuration**

```java
@Bean
public XStreamMarshaller tradeMarshaller() {
	Map<String, Class<?>> aliases = new HashMap<>(); // Class<?> 사용 권장
	aliases.put("trade", Trade.class); // XML의 <trade> 태그를 Trade 클래스에 매핑
	aliases.put("price", BigDecimal.class); // <price> 태그를 BigDecimal에 매핑
	aliases.put("isin", String.class);
	aliases.put("customer", String.class);
	aliases.put("quantity", Long.class); // 또는 int.class 등 실제 타입에 맞게

	XStreamMarshaller marshaller = new XStreamMarshaller();
	marshaller.setAliases(aliases); // 별칭 설정
	return marshaller;
}
```

입력 시, 리더는 새 프래그먼트가 시작되려고 하는 것을 인식할 때까지 XML 리소스를 읽습니다.

기본적으로 리더는 요소 이름을 일치시켜 새 프래그먼트가 시작되려고 하는 것을 인식합니다.

리더는 프래그먼트에서 독립 실행형 XML 문서를 만들고 해당 문서를 역직렬화기(일반적으로 Spring OXM `Unmarshaller` 주변의 래퍼)에 전달하여 XML을 Java 객체로 매핑합니다.

요약하자면, 이 절차는 Spring 구성에서 제공하는 주입을 사용하는 다음 Java 코드와 유사합니다:

```java
// 프로그래밍 방식 설정 및 사용 예시 (실제 잡에서는 빈으로 주입받아 사용)
StaxEventItemReader<Trade> xmlStaxEventItemReader = new StaxEventItemReader<>();
Resource resource = new ByteArrayResource(xmlResource.getBytes()); // xmlResource는 XML 문자열이라고 가정

Map<String, String> aliases = new HashMap<>(); // 원문은 String,String 이나 Class가 더 적합
aliases.put("trade","org.springframework.batch.samples.domain.trade.Trade");
aliases.put("price","java.math.BigDecimal");
// ... (나머지 별칭 설정)
XStreamMarshaller unmarshaller = new XStreamMarshaller();
unmarshaller.setAliases(aliases); // 실제로는 setAliasedClasses 또는 유사한 메서드 사용 가능
xmlStaxEventItemReader.setUnmarshaller(unmarshaller);
xmlStaxEventItemReader.setResource(resource);
xmlStaxEventItemReader.setFragmentRootElementName("trade"); // 원문에는 setFragmentRootElementName 이지만, 빌더에서는 addFragmentRootElements
xmlStaxEventItemReader.open(new ExecutionContext()); // ItemStream 생명주기

boolean hasNext = true;
Trade trade = null;

while (hasNext) {
    trade = xmlStaxEventItemReader.read();
    if (trade == null) {
        hasNext = false;
    }
    else {
        System.out.println(trade);
    }
}
```

### StaxEventItemWriter

출력은 입력과 대칭적으로 작동합니다. `StaxEventItemWriter`에는 `Resource`, 마샬러(marshaller) 및 `rootTagName`이 필요합니다.

Java 객체가 마샬러(일반적으로 표준 Spring OXM `Marshaller`)에 전달되며, 이 마샬러는 OXM 도구에 의해 각 프래그먼트에 대해 생성된 `StartDocument` 및 `EndDocument` 이벤트를 필터링하는 사용자 정의 이벤트 라이터를 사용하여 `Resource`에 씁니다.

다음 Java 예제는 `MarshallingEventWriterSerializer`를 사용합니다:

```java
@Bean
public StaxEventItemWriter<Trade> itemWriter(Resource outputResource, Marshaller tradeMarshaller) { // outputResource와 Marshaller 주입
	return new StaxEventItemWriterBuilder<Trade>()
			.name("tradesWriter") // ItemWriter 이름
			.marshaller(tradeMarshaller) // 사용할 Marshaller (OXM)
			.resource(outputResource) // 출력 XML 파일 리소스
			.rootTagName("trade") // 각 아이템 객체를 감싸는 XML 태그 이름
			.overwriteOutput(true) // 기존 파일 덮어쓰기 여부
			.build();
}
```

앞의 구성은 세 가지 필수 속성을 설정하고 이 챕터 앞부분에서 언급된, 기존 파일을 덮어쓸 수 있는지 여부를 지정하기 위한 선택적 `overwriteOutput=true` 속성을 설정합니다.

다음 Java 예제는 이 챕터 앞부분에 표시된 읽기 예제에서 사용된 것과 동일한 마샬러를 사용합니다:

```java
@Bean
public XStreamMarshaller tradeMarshaller() { // 읽기에서 사용한 tradeMarshaller 빈 재사용 또는 동일 구성
	XStreamMarshaller marshaller = new XStreamMarshaller();

	Map<String, Class<?>> aliases = new HashMap<>();
	aliases.put("trade", Trade.class);
	aliases.put("price", BigDecimal.class);
	aliases.put("isin", String.class);
	aliases.put("customer", String.class);
	aliases.put("quantity", Long.class);

	marshaller.setAliases(aliases);

	return marshaller;
}
```

Java 예제로 요약하자면, 다음 코드는 논의된 모든 사항을 보여주며, 필수 속성의 프로그래밍 방식 설정을 보여줍니다:

```java
// 프로그래밍 방식 설정 예시
FileSystemResource resource = new FileSystemResource("data/outputFile.xml");

Map<String, String> aliases = new HashMap<>(); // 원문은 String,String 이나 Class가 더 적합
aliases.put("trade","org.springframework.batch.samples.domain.trade.Trade");
// ... (나머지 별칭 설정)
Marshaller marshaller = new XStreamMarshaller();
marshaller.setAliases(aliases); // 실제로는 setAliasedClasses 또는 유사한 메서드 사용 가능

StaxEventItemWriter<Trade> staxItemWriter =
	new StaxEventItemWriterBuilder<Trade>()
				.name("tradesWriter")
				.marshaller(marshaller)
				.resource(resource)
				.rootTagName("trade") // 각 아이템을 이 태그로 감쌈 (주의: 전체 문서의 루트 태그와는 다름)
				.overwriteOutput(true)
				.build();

staxItemWriter.afterPropertiesSet(); // 빈 생명주기 메서드 (보통 프레임워크가 관리)

ExecutionContext executionContext = new ExecutionContext();
staxItemWriter.open(executionContext); // ItemStream 생명주기

Trade trade = new Trade();
trade.setPrice(new BigDecimal("11.39")); // BigDecimal로 생성
trade.setIsin("XYZ0001");
trade.setQuantity(5L);
trade.setCustomer("Customer1");
staxItemWriter.write(new Chunk<>(Arrays.asList(trade))); // Chunk로 전달해야 함
staxItemWriter.close(executionContext); // ItemStream 생명주기
```

---

### **학습 목표 제시**

이번 "XML Item Readers and Writers" 챕터를 통해 학생분은 다음을 이해하고 배울 수 있습니다:

1. **Spring Batch XML 처리의 기본 개념 이해:** StAX API 사용 이유, XML '프래그먼트' 개념, 그리고 Object/XML Mapping (OXM)과의 연관성을 파악합니다.
2. **`StaxEventItemReader`를 사용한 XML 읽기 방법 습득:** XML 파일에서 특정 프래그먼트(레코드)를 식별하고, Spring OXM의 `Unmarshaller` (예: `XStreamMarshaller`)를 사용하여 이를 자바 객체로 변환하는 과정을 이해하고 설정하는 방법을 익힙니다.
3. **`StaxEventItemWriter`를 사용한 XML 쓰기 방법 습득:** 자바 객체를 Spring OXM의 `Marshaller`를 사용하여 XML 프래그먼트로 변환하고, 이를 파일에 쓰는 과정을 이해하고 설정하는 방법을 익힙니다.
4. **Spring OXM (특히 `XStreamMarshaller`)의 기본 설정 이해:** XML 태그와 자바 클래스/필드를 매핑하기 위한 별칭(alias) 설정 방법을 이해합니다.

---

### **핵심 개념 설명**

이 챕터의 핵심은 **"XML 문서로부터 데이터를 어떻게 효율적으로 읽고 객체로 변환하며(Unmarshalling), 반대로 객체를 어떻게 XML 형식으로 변환하여 파일에 쓸 것인가(Marshalling)?"** 입니다.

이를 위해 Spring Batch는 StAX API와 Spring OXM 라이브러리를 주로 활용합니다.

### 1. 왜 StAX API를 사용하는가? (XML 스트리밍의 중요성)

- **DOM (Document Object Model)의 한계:** DOM 파서는 XML 문서 전체를 메모리에 트리 구조로 로드합니다. 작은 XML 파일에는 편리하지만, 대용량 배치 처리에는 **메모리 부족(Out Of Memory)** 문제를 일으킬 수 있습니다.
- **SAX (Simple API for XML)의 한계:** SAX 파서는 이벤트를 기반으로 XML을 순차적으로 읽어나가지만, 개발자가 콜백 핸들러를 직접 구현해야 하고, 파싱 과정을 완전히 제어하기 어렵다는 단점이 있습니다. (프레임워크가 주도하는 것이 아니라, 파서가 이벤트를 던지면 개발자가 반응하는 방식)
- **StAX (Streaming API for XML)의 장점:**
  - **스트리밍 방식:** SAX처럼 문서를 순차적으로 읽어 메모리 사용량이 적습니다.
  - **풀(Pull) 파서:** DOM이나 SAX(푸시 파서)와 달리, 개발자가 필요할 때 다음 이벤트를 "가져오는(pull)" 방식으로 파싱을 제어할 수 있습니다. 이는 배치 처리의 `read()` 메서드처럼 "하나의 아이템을 읽어온다"는 개념과 잘 맞습니다.
  - Spring Batch는 이러한 StAX의 장점을 활용하여 대용량 XML 파일을 효율적으로 처리합니다.

### 2. XML '프래그먼트(Fragment)' 개념

- 플랫 파일에서는 한 줄(line)이 하나의 레코드(데이터 단위)였지만, XML에서는 보통 **특정 태그로 둘러싸인 부분**을 하나의 레코드(아이템)로 취급합니다. 이를 **프래그먼트(fragment)**라고 부릅니다.
- 예시 (문서 그림 1 참고):

    ```xml
    <records> <!-- 전체 문서를 감싸는 루트 요소 (프래그먼트 아님) -->
        <trade>   <!-- 이 부분이 하나의 프래그먼트 (하나의 아이템) -->
            <isin>XYZ0001</isin>
            ...
        </trade>
        <trade>   <!-- 또 다른 프래그먼트 -->
            <isin>XYZ0002</isin>
            ...
        </trade>
    </records>
    ```

- `StaxEventItemReader`는 이 프래그먼트 단위로 XML을 읽어 객체로 변환합니다. `StaxEventItemWriter`는 객체를 이러한 프래그먼트 단위로 XML에 씁니다.
- **프래그먼트 루트 요소 (Fragment Root Element):** 각 프래그먼트를 시작하는 태그 이름입니다. 위 예시에서는 "trade"가 프래그먼트 루트 요소입니다. `ItemReader`와 `ItemWriter` 설정 시 이 이름을 지정해줘야 합니다.

### 3. Object/XML Mapping (OXM)

- **OXM이란?:** 자바 객체와 XML 문서 간의 변환(매핑)을 처리하는 기술입니다.
  - **언마샬링 (Unmarshalling):** XML 프래그먼트 -> 자바 객체 (읽을 때)
  - **마샬링 (Marshalling):** 자바 객체 -> XML 프래그먼트 (쓸 때)
- **Spring OXM:** Spring 프레임워크는 JAXB, XStream, Castor, JiBX 등 다양한 OXM 구현 기술들을 일관된 방식으로 사용할 수 있도록 추상화 계층인 Spring OXM을 제공합니다.
- **Spring Batch와의 관계:** Spring Batch는 XML 처리 시 주로 Spring OXM에 작업을 위임합니다. 이를 통해 개발자는 특정 OXM 기술에 종속되지 않고 유연하게 XML 바인딩을 처리할 수 있습니다. (의존성은 선택 사항이지만, 일반적으로 많이 사용됩니다.)

### 4. `StaxEventItemReader` (XML 읽기)

- **역할:** StAX API를 사용하여 XML 파일로부터 프래그먼트 단위로 데이터를 읽고, 이를 `Unmarshaller`를 통해 자바 객체로 변환하여 제공하는 `ItemReader` 구현체입니다.
- **주요 설정:**
  1. **`resource` (Resource)**: 읽어올 XML 파일 리소스.
  2. **`fragmentRootElementName` (또는 `addFragmentRootElements(String... names)`)**: 어떤 XML 태그가 하나의 아이템(레코드)에 해당하는 프래그먼트의 시작인지를 알려줍니다. (예: "trade") 여러 개의 루트 요소 이름을 지정할 수도 있습니다.
  3. **`unmarshaller` (org.springframework.oxm.Unmarshaller)**: XML 프래그먼트를 실제 자바 객체로 변환(언마샬링)하는 역할을 합니다. Spring OXM에서 제공하는 `Unmarshaller` 구현체(예: `XStreamMarshaller`, `Jaxb2Marshaller`)를 설정합니다.
- **동작 과정 (간략히):**
  1. `StaxEventItemReader`는 XML 파일을 StAX 이벤트 스트림으로 읽기 시작합니다.
  2. `fragmentRootElementName`으로 지정된 시작 태그를 만나면, 해당 프래그먼트의 끝 태그까지의 XML 조각을 추출합니다.
  3. 이 추출된 XML 조각(프래그먼트)을 설정된 `unmarshaller`에게 전달합니다.
  4. `unmarshaller`는 XML 조각을 해당 자바 객체로 변환하여 반환합니다.
  5. `StaxEventItemReader`는 이 변환된 객체를 `read()` 메서드의 결과로 반환합니다.
  6. 파일의 끝에 도달하거나 더 이상 지정된 프래그먼트가 없으면 `null`을 반환합니다.
- **예시 `XStreamMarshaller` 설정 (언마샬링용):**
  - XStream은 XML 태그 이름과 자바 클래스/필드 이름을 매핑하기 위해 **별칭(alias)**을 사용합니다.
  - `Map<String, Class<?>> aliases = new HashMap<>();`
  - `aliases.put("trade", Trade.class);` // XML의 `<trade>` 태그를 `Trade.java` 클래스로 매핑
  - `aliases.put("isin", String.class);` // `<trade>` 내부의 `<isin>` 태그를 `Trade` 클래스의 `isin` 필드(타입은 String)로 매핑 (필드 타입도 지정 가능)
  - `marshaller.setAliases(aliases);` 또는 `marshaller.setAliasedClasses(Map<String, Class<?>>);` (API 버전에 따라 약간 다를 수 있음)

### 5. `StaxEventItemWriter` (XML 쓰기)

- **역할:** 자바 객체(아이템)를 받아 `Marshaller`를 통해 XML 프래그먼트로 변환하고, 이를 StAX API를 사용하여 지정된 파일에 기록하는 `ItemWriter` 구현체입니다.
- **주요 설정:**
  1. **`resource` (Resource)**: 데이터를 기록할 출력 XML 파일 리소스.
  2. **`marshaller` (org.springframework.oxm.Marshaller)**: 자바 객체를 XML 프래그먼트로 변환(마샬링)하는 역할을 합니다. (예: `XStreamMarshaller`, `Jaxb2Marshaller`)
  3. **`rootTagName` (String)**: 각 아이템 객체를 XML로 변환할 때, 그 객체에 해당하는 최상위 XML 태그(프래그먼트 루트 요소)의 이름을 지정합니다. (예: "trade")
  4. **`overwriteOutput` (boolean)**: (기본값 `false`) `true`로 설정하면, 출력 파일이 이미 존재할 경우 덮어씁니다.
  5. **`headerCallback` / `footerCallback`**: XML 문서 전체를 감싸는 루트 요소(예: `<records> ... </records>`)나 XML 선언부(`<?xml ... ?>`)를 쓰기 위해 사용됩니다. `StaxEventItemWriter`는 기본적으로 프래그먼트들만 연속해서 쓰기 때문에, 완전한 XML 문서를 만들려면 이 콜백들을 사용해야 할 수 있습니다.
- **동작 과정 (간략히):**
  1. `StaxEventItemWriter`는 `write(Chunk<T> items)` 메서드를 통해 아이템 묶음을 받습니다.
  2. 청크 내의 각 아이템 객체에 대해:
    - 설정된 `marshaller`에게 객체를 전달하여 XML 프래그먼트(문자열 또는 StAX 이벤트 형태)로 변환합니다.
    - 이때 `rootTagName`으로 지정된 태그로 객체의 데이터가 감싸집니다.
    - 변환된 XML 프래그먼트를 `resource`로 지정된 파일에 StAX API를 통해 씁니다.
  3. `StaxEventItemWriter`는 각 프래그먼트가 독립적인 XML 문서가 되지 않도록, `StartDocument`와 `EndDocument` 이벤트를 필터링하여 여러 프래그먼트가 하나의 XML 문서 내에 포함되도록 합니다. (단, 전체 문서를 감싸는 최상위 루트 태그는 직접 관리해주어야 할 수 있음)
- **예시 `XStreamMarshaller` 설정 (마샬링용):**
  - 언마샬링 시 사용했던 별칭 설정을 그대로 사용하거나 유사하게 설정합니다. XStream은 클래스 정보와 별칭을 통해 양방향 변환이 가능합니다.

---

### **주요 용어 해설**

- **StAX (Streaming API for XML):** XML 문서를 스트리밍 방식으로 처리하기 위한 자바 API. 이벤트 기반이면서도 개발자가 파싱 흐름을 제어할 수 있는 풀(pull) 파서 방식.
- **프래그먼트 (Fragment):** XML 문서 내에서 하나의 독립적인 데이터 단위(레코드)를 나타내는 부분. 보통 특정 루트 요소 태그로 둘러싸여 있음.
- **OXM (Object/XML Mapping):** 자바 객체와 XML 문서 간의 상호 변환을 담당하는 기술.
- **마샬링 (Marshalling):** 자바 객체 -> XML 변환.
- **언마샬링 (Unmarshalling):** XML -> 자바 객체 변환.
- **별칭 (Alias - XStream에서):** XML 태그 이름과 자바 클래스 또는 필드 이름을 연결해주는 매핑 정보.
- **루트 요소 (Root Element):**
  - **프래그먼트 루트 요소:** 각 개별 레코드(아이템)를 나타내는 XML 프래그먼트의 최상위 태그. (예: `<trade>`)
  - **문서 루트 요소:** 전체 XML 문서를 감싸는 단 하나의 최상위 태그. (예: `<records>`) `StaxEventItemWriter`의 `rootTagName`은 보통 프래그먼트 루트 요소를 의미합니다.

---

### **코드 예제 분석**

### `StaxEventItemReader` 설정 예시 (Java Configuration)

```java
@Configuration
public class XmlReaderConfig {

    // 1. Unmarshaller 빈 정의 (예: XStream 사용)
    @Bean
    public XStreamMarshaller tradeUnmarshaller() { // Marshaller/Unmarshaller 겸용 가능
        Map<String, Class<?>> aliases = new HashMap<>();
        aliases.put("trade", Trade.class);       // <trade> 태그는 Trade 객체로
        aliases.put("isin", String.class);       // <isin> 태그는 String (Trade 객체 내 필드)
        aliases.put("quantity", Long.class);     // <quantity> 태그는 Long
        aliases.put("price", BigDecimal.class);  // <price> 태그는 BigDecimal
        aliases.put("customer", String.class);   // <customer> 태그는 String

        XStreamMarshaller marshaller = new XStreamMarshaller();
        marshaller.setAliases(aliases);
        // marshaller.setSupportedClasses(Trade.class); // JAXB 등에서는 필요할 수 있음
        return marshaller;
    }

    // 2. StaxEventItemReader 빈 정의
    @Bean
    public StaxEventItemReader<Trade> xmlFileItemReader(
            @Value("classpath:input/trades.xml") Resource inputResource, // 주입받는 것이 좋음
            Unmarshaller tradeUnmarshaller) { // 위에서 정의한 Unmarshaller 빈 주입

        return new StaxEventItemReaderBuilder<Trade>()
                .name("xmlFileItemReader")         // Reader 이름
                .resource(inputResource)           // 읽을 XML 파일
                .addFragmentRootElements("trade")  // "trade" 태그를 하나의 아이템 단위로 인식
                .unmarshaller(tradeUnmarshaller)   // 사용할 Unmarshaller
                .build();
    }

    // Trade.java (도메인 객체 - 필드명과 별칭이 일치하거나, 추가 어노테이션/설정 필요)
    // public class Trade {
    //    private String isin;
    //    private Long quantity;
    //    private BigDecimal price;
    //    private String customer;
    //    // Getters and Setters
    // }
}

```

- `tradeUnmarshaller()`: XStream을 사용하여 XML 태그와 `Trade` 객체 및 그 필드들을 어떻게 매핑할지 별칭으로 정의합니다.
- `xmlFileItemReader()`:
  - `inputResource`: 읽어올 XML 파일(`trades.xml`)을 지정합니다.
  - `addFragmentRootElements("trade")`: XML 내에서 `<trade>...</trade>` 부분을 하나의 데이터 조각(프래그먼트)으로 인식하라고 알려줍니다.
  - `unmarshaller(tradeUnmarshaller)`: 위에서 정의한 `tradeUnmarshaller`를 사용하여 각 "trade" 프래그먼트를 `Trade` 객체로 변환하도록 설정합니다.

### `StaxEventItemWriter` 설정 예시 (Java Configuration)

```java
@Configuration
public class XmlWriterConfig {

    // 1. Marshaller 빈 정의 (Reader에서 사용한 Unmarshaller와 동일하거나 유사한 설정 사용 가능)
    @Bean
    public XStreamMarshaller tradeMarshaller() {
        Map<String, Class<?>> aliases = new HashMap<>();
        aliases.put("trade", Trade.class);
        aliases.put("isin", String.class);
        aliases.put("quantity", Long.class);
        aliases.put("price", BigDecimal.class);
        aliases.put("customer", String.class);

        XStreamMarshaller marshaller = new XStreamMarshaller();
        marshaller.setAliases(aliases);
        return marshaller;
    }

    // 2. StaxEventItemWriter 빈 정의
    @Bean
    public StaxEventItemWriter<Trade> xmlFileItemWriter(
            @Value("file:output/output_trades.xml") Resource outputResource, // 주입
            Marshaller tradeMarshaller) { // 위에서 정의한 Marshaller 빈 주입

        return new StaxEventItemWriterBuilder<Trade>()
                .name("xmlFileItemWriter")        // Writer 이름
                .resource(outputResource)          // 출력할 XML 파일
                .marshaller(tradeMarshaller)      // 사용할 Marshaller
                .rootTagName("trade")             // 각 Trade 객체를 <trade>...</trade> 태그로 감싸서 출력
                .overwriteOutput(true)            // 파일이 이미 있으면 덮어쓰기
                // .headerCallback(writer -> writer.writeStartDocument()) // XML 선언부 등 필요시
                // .footerCallback(writer -> writer.writeEndDocument())   // 전체 루트 요소 닫기 등 필요시
                .build();
    }
}

```

- `tradeMarshaller()`: 객체를 XML로 변환할 때 사용할 마샬러입니다. 읽기 시의 언마샬러와 동일한 별칭 설정을 사용할 수 있습니다.
- `xmlFileItemWriter()`:
  - `outputResource`: 결과 XML을 쓸 파일을 지정합니다.
  - `marshaller(tradeMarshaller)`: `Trade` 객체를 XML로 변환할 때 `tradeMarshaller`를 사용하도록 설정합니다.
  - `rootTagName("trade")`: 각 `Trade` 객체는 `<trade>...</trade>` 프래그먼트 형태로 쓰여집니다.
  - **주의:** `rootTagName`은 각 아이템을 감싸는 태그이지, XML 문서 전체의 루트 태그(예: `<trades>`)가 아닙니다. 전체 루트 태그를 만들려면 보통 `headerCallback` (시작 태그 쓰기)과 `footerCallback` (종료 태그 쓰기)을 추가로 설정해야 합니다.

---

### **"왜?" 라는 질문에 대한 답변**

- **왜 플랫 파일처럼 `LineTokenizer`나 `FieldSetMapper` 같은 것을 직접 사용하지 않고, OXM 라이브러리(Unmarshaller/Marshaller)에 의존할까?**
  - XML은 단순한 텍스트 라인 기반의 플랫 파일보다 훨씬 복잡한 계층 구조와 다양한 표현 방식(속성, 네임스페이스 등)을 가집니다. 이를 직접 파싱하고 매핑하는 것은 매우 어렵고 오류가 발생하기 쉽습니다.
  - JAXB, XStream 같은 성숙한 OXM 라이브러리들은 이러한 복잡한 XML 구조를 효과적으로 다루고, 자바 객체와의 매핑을 자동화하거나 설정을 통해 쉽게 할 수 있도록 강력한 기능을 제공합니다.
  - Spring OXM은 이러한 다양한 라이브러리들을 일관된 인터페이스(`Unmarshaller`, `Marshaller`)로 추상화해주므로, Spring Batch는 이 인터페이스에만 의존하여 특정 기술에 대한 종속성을 줄이고 유연성을 확보할 수 있습니다. 즉, "잘 만들어진 바퀴를 다시 발명하지 않는" 것입니다.
- **`StaxEventItemWriter`의 `rootTagName`은 왜 문서 전체의 루트 태그가 아닐까?**
  - `StaxEventItemWriter`는 기본적으로 아이템 스트림을 처리하여 각 아이템을 XML 프래그먼트로 출력하는 데 초점을 맞춥니다. 만약 `rootTagName`이 문서 전체의 루트 태그를 의미한다면, 각 아이템을 쓸 때마다 문서 전체를 새로 시작하고 닫아야 하는 비효율적인 상황이 발생할 수 있습니다.
  - 대신, 각 아이템에 해당하는 프래그먼트의 루트 태그(`rootTagName`)를 지정하고, 문서 전체를 감싸는 구조(XML 선언, 최상위 루트 요소 여닫기)는 `ItemStream`의 `open()`과 `close()` 시점에 (또는 `FlatFileItemWriter`의 `headerCallback` 및 `footerCallback`과 유사한 콜백을 통해) 한 번만 처리하는 것이 더 효율적이고 유연합니다. `StaxEventItemWriter` 자체도 `ItemStream`을 구현하므로, `open()`에서 헤더를, `close()`에서 푸터를 쓰는 로직을 추가할 수 있습니다. (또는 빌더의 콜백 사용)

---

### **주의사항 및 Best Practice**

1. **OXM 라이브러리 선택 및 의존성 추가:** 사용할 OXM 라이브러리(JAXB, XStream 등)를 결정하고, 해당 라이브러리와 Spring OXM 관련 의존성을 프로젝트에 추가해야 합니다.
2. **별칭/매핑 설정의 정확성:** XML 태그 이름과 자바 객체의 클래스/필드 이름 간의 매핑 설정(예: XStream의 별칭, JAXB의 어노테이션)이 정확해야 올바른 변환이 이루어집니다.
3. **네임스페이스(Namespace) 처리:** XML 문서가 네임스페이스를 사용하는 경우, `Unmarshaller` 또는 `Marshaller` 설정 시 네임스페이스 관련 처리를 고려해야 합니다. (OXM 라이브러리마다 처리 방식이 다를 수 있음)
4. **대용량 XML 처리 시 성능:** StAX는 스트리밍 방식이라 메모리 효율적이지만, 매우 복잡하거나 깊은 구조의 XML을 처리할 때는 여전히 성능에 주의해야 합니다. 프래그먼트의 크기와 구조가 성능에 영향을 줄 수 있습니다.
5. **`StaxEventItemWriter`와 완전한 XML 문서 생성:** `StaxEventItemWriter`는 기본적으로 프래그먼트의 연속을 출력합니다. 완전한 XML 문서(XML 선언, 단일 루트 요소 등)를 만들려면, `StaxEventItemWriterBuilder`의 `headerCallback`과 `footerCallback`을 사용하여 XML 문서의 시작과 끝부분을 직접 써주어야 합니다.

    ```java
    // 예시: Writer에 헤더/푸터 콜백 추가
    .headerCallback(writer -> writer.writeStartDocument("UTF-8", "1.0")) // XML 선언부
    // .headerCallback(writer -> writer.writeStartElement("records")) // 전체 루트 요소 시작 (필요 시)
    .footerCallback(writer -> writer.writeEndDocument()) // 문서 종료 (전체 루트 요소 닫기는 별도 콜백이나 open/close에서)
    
    ```

6. **`ItemStream` 구현:** `StaxEventItemReader`와 `StaxEventItemWriter` 모두 `ItemStream`을 구현합니다. 따라서 Spring Batch가 스텝 생명주기에 맞춰 `open()`, `update()`, `close()`를 호출하여 리소스 관리 및 상태 저장을 지원합니다.

---

### **이전 학습 내용과의 연관성**

- **`ItemReader` / `ItemWriter` 인터페이스:** `StaxEventItemReader`와 `StaxEventItemWriter`는 이들 인터페이스의 XML 특화 구현체입니다.
- **`ItemStream` 인터페이스:** XML 파일 리소스와 StAX 파서/라이터 상태 관리를 위해 `ItemStream`을 구현합니다.
- **`Resource` (스프링 코어):** XML 파일 위치를 지정하는 데 사용됩니다.
- **`FlatFileItemReader/Writer`와의 비교:**
  - 플랫 파일: 라인 기반, `LineTokenizer/FieldSetMapper` 또는 `FieldExtractor/LineAggregator` 사용.
  - XML: 프래그먼트 기반, `Unmarshaller/Marshaller` (OXM) 사용.
  - 둘 다 스트리밍 처리를 지향하지만, 데이터 구조와 파싱/매핑 메커니즘이 다릅니다.

---
