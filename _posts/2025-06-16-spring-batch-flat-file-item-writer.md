---
title: Spring Batch - Flat Files(FlatFileItemWriter)
description: 
author: laze
date: 2025-06-16 00:00:02 +0900
categories: [Dev, SpringBatch]
tags: [SpringBatch]
---
### 플랫 파일 아이템 라이터 (FlatFileItemWriter)

플랫 파일에 쓰는 작업은 파일을 읽어 들일 때 극복해야 하는 것과 동일한 문제와 이슈를 가지고 있습니다.

스텝(step)은 구분되거나 고정 길이 형식으로 트랜잭션 방식으로 쓸 수 있어야 합니다.

### 라인 애그리게이터 (LineAggregator)

아이템을 가져와 `String`으로 바꾸기 위해 `LineTokenizer` 인터페이스가 필요한 것처럼, 파일 쓰기는 여러 필드를 파일에 쓰기 위한 단일 문자열로 집계(aggregate)하는 방법이 있어야 합니다.

Spring Batch에서 이것은 다음 인터페이스 정의에 표시된 `LineAggregator`입니다:

```java
public interface LineAggregator<T> {

    public String aggregate(T item);

}
```

`LineAggregator`는 `LineTokenizer`의 논리적 반대입니다.

`LineTokenizer`는 `String`을 받아 `FieldSet`을 반환하는 반면, `LineAggregator`는 아이템을 받아 `String`을 반환합니다.

### PassThroughLineAggregator

`LineAggregator` 인터페이스의 가장 기본적인 구현은 `PassThroughLineAggregator`이며, 이는 객체가 이미 문자열이거나 그 문자열 표현이 다음 코드와 같이 쓰기에 적합하다고 가정합니다:

```java
public class PassThroughLineAggregator<T> implements LineAggregator<T> {

    public String aggregate(T item) {
        return item.toString();
    }
}
```

위의 구현은 문자열 생성에 대한 직접적인 제어가 필요하지만 `FlatFileItemWriter`의 장점(예: 트랜잭션 및 재시작 지원)이 필요한 경우 유용합니다.

### 간단한 파일 쓰기 예제 (Simplified File Writing Example)

이제 `LineAggregator` 인터페이스와 그 가장 기본적인 구현인 `PassThroughLineAggregator`가 정의되었으므로, 쓰기의 기본 흐름을 설명할 수 있습니다:

1. 쓸 객체가 `String`을 얻기 위해 `LineAggregator`에 전달됩니다.
2. 반환된 `String`이 구성된 파일에 쓰여집니다.

```java
// FlatFileItemWriter 내부의 (간략화된) 단일 아이템 처리 로직 예시
// 실제 ItemWriter 인터페이스의 write 메서드는 Chunk를 받음
public void writeSingleItem(T item) throws Exception { // 메서드명은 이해를 돕기 위해 변경
    // lineAggregator.aggregate(item)으로 객체를 문자열로 변환
    // LINE_SEPARATOR는 줄바꿈 문자
    outputBufferedWriter.write(lineAggregator.aggregate(item) + LINE_SEPARATOR);
}
```

**Java 설정 (Java Configuration)**

```java
@Bean
public FlatFileItemWriter<Foo> itemWriter() { // Foo 타입 아이템을 쓴다고 가정
	return  new FlatFileItemWriterBuilder<Foo>()
           			.name("itemWriter") // ItemWriter 이름 설정
           			.resource(new FileSystemResource("target/test-outputs/output.txt")) // 출력 파일 리소스
           			.lineAggregator(new PassThroughLineAggregator<>()) // 가장 간단한 LineAggregator 사용
           			.build();
}

```

### 필드 추출기 (FieldExtractor)

앞의 예제는 파일에 쓰는 가장 기본적인 용도에는 유용할 수 있습니다. 그러나 `FlatFileItemWriter`의 대부분 사용자는 기록해야 할 도메인 객체를 가지고 있으므로, 이는 라인으로 변환되어야 합니다. 파일 읽기에서는 다음이 필요했습니다:

1. 파일에서 한 줄을 읽습니다.
2. `FieldSet`을 검색하기 위해 라인을 `LineTokenizer#tokenize()` 메서드에 전달합니다.
3. 토큰화에서 반환된 `FieldSet`을 `FieldSetMapper`에 전달하고, 그 결과를 `ItemReader#read()` 메서드에서 반환합니다.

파일 쓰기에는 유사하지만 반대되는 단계가 있습니다:

1. 쓸 아이템을 라이터에 전달합니다.
2. 아이템의 필드를 배열로 변환합니다.
3. 결과 배열을 라인으로 집계합니다.

프레임워크가 객체에서 어떤 필드를 기록해야 하는지 알 수 있는 방법이 없기 때문에, 아이템을 배열로 바꾸는 작업을 수행하기 위해 다음 인터페이스 정의에 표시된 `FieldExtractor`를 작성해야 합니다:

```java
public interface FieldExtractor<T> {

    Object[] extract(T item);

}
```

`FieldExtractor` 인터페이스의 구현은 제공된 객체의 필드로부터 배열을 생성해야 하며, 이 배열은 요소 사이에 구분 기호를 사용하여 또는 고정 너비 라인의 일부로 기록될 수 있습니다.

### PassThroughFieldExtractor

배열, `Collection` 또는 `FieldSet`과 같은 컬렉션이 기록되어야 하는 경우가 많습니다.

이러한 컬렉션 유형 중 하나에서 배열을 "추출"하는 것은 매우 간단합니다.

그렇게 하려면 컬렉션을 배열로 변환하십시오.

따라서 이 시나리오에서는 `PassThroughFieldExtractor`를 사용해야 합니다.

전달된 객체가 컬렉션 유형이 아닌 경우, `PassThroughFieldExtractor`는 추출할 아이템만 포함하는 배열을 반환한다는 점에 유의해야 합니다.

### BeanWrapperFieldExtractor

파일 읽기 섹션에서 설명한 `BeanWrapperFieldSetMapper`와 마찬가지로, 직접 변환을 작성하는 것보다 도메인 객체를 객체 배열로 변환하는 방법을 구성하는 것이 종종 선호됩니다.

`BeanWrapperFieldExtractor`는 다음 예제와 같이 이 기능을 제공합니다:

```java
BeanWrapperFieldExtractor<Name> extractor = new BeanWrapperFieldExtractor<>();
extractor.setNames(new String[] { "first", "last", "born" }); // 추출할 필드 이름 지정

String first = "Alan";
String last = "Turing";
int born = 1912;

Name n = new Name(first, last, born); // Name 객체 생성 (가정)
Object[] values = extractor.extract(n); // 객체에서 필드 값 추출

assertEquals(first, values[0]);
assertEquals(last, values[1]);
assertEquals(born, values[2]);
```

이 추출기 구현에는 단 하나의 필수 속성만 있습니다: 매핑할 필드의 이름입니다. `BeanWrapperFieldSetMapper`가 `FieldSet`의 필드를 제공된 객체의 세터에 매핑하기 위해 필드 이름이 필요한 것처럼, `BeanWrapperFieldExtractor`는 객체 배열을 만들기 위해 게터(getter)에 매핑할 이름이 필요합니다. 이름의 순서가 배열 내 필드의 순서를 결정한다는 점에 유의할 가치가 있습니다.

### 구분 파일 쓰기 예제 (Delimited File Writing Example)

가장 기본적인 플랫 파일 형식은 모든 필드가 구분 기호로 분리된 것입니다. 이것은 `DelimitedLineAggregator`를 사용하여 수행할 수 있습니다. 다음 예제는 고객 계정에 대한 크레딧을 나타내는 간단한 도메인 객체를 기록합니다:

```java
public class CustomerCredit {

    private int id;
    private String name;
    private BigDecimal credit;

    //getters and setters removed for clarity
}
```

도메인 객체가 사용되고 있으므로, 사용할 구분 기호와 함께 `FieldExtractor` 인터페이스의 구현이 제공되어야 합니다.

다음 예제는 Java에서 구분 기호와 함께 `FieldExtractor`를 사용하는 방법을 보여줍니다:

**Java 설정 (Java Configuration)**

```java
@Bean
public FlatFileItemWriter<CustomerCredit> itemWriter(Resource outputResource) throws Exception { // outputResource 주입
	BeanWrapperFieldExtractor<CustomerCredit> fieldExtractor = new BeanWrapperFieldExtractor<>();
	fieldExtractor.setNames(new String[] {"name", "credit"}); // CustomerCredit 객체에서 "name"과 "credit" 필드 추출
	fieldExtractor.afterPropertiesSet(); // (선택적) 초기화 완료 후 호출

	DelimitedLineAggregator<CustomerCredit> lineAggregator = new DelimitedLineAggregator<>();
	lineAggregator.setDelimiter(","); // 필드 구분자로 쉼표 사용
	lineAggregator.setFieldExtractor(fieldExtractor); // 필드 추출기 설정

	return new FlatFileItemWriterBuilder<CustomerCredit>()
				.name("customerCreditWriter")
				.resource(outputResource) // 출력 파일 리소스 설정
				.lineAggregator(lineAggregator) // 생성한 LineAggregator 설정
				.build();
}
```

이전 예제에서는 이 챕터 앞부분에서 설명한 `BeanWrapperFieldExtractor`를 사용하여 `CustomerCredit` 내의 `name` 및 `credit` 필드를 객체 배열로 변환한 다음, 각 필드 사이에 쉼표를 사용하여 기록합니다.

다음 예제와 같이 `FlatFileItemWriterBuilder.DelimitedBuilder`를 사용하여 `BeanWrapperFieldExtractor`와 `DelimitedLineAggregator`를 자동으로 생성할 수도 있습니다:

**Java 설정 (Java Configuration)**

```java
@Bean
public FlatFileItemWriter<CustomerCredit> itemWriter(Resource outputResource) { // throws Exception 제거 가능
	return new FlatFileItemWriterBuilder<CustomerCredit>()
				.name("customerCreditWriter")
				.resource(outputResource)
				.delimited() // DelimitedBuilder 사용 시작
				.delimiter("|") // 구분자로 파이프 사용
				.names(new String[] {"name", "credit"}) // 추출할 필드 이름 (내부적으로 BeanWrapperFieldExtractor 생성 및 설정)
				.build();
}
```

### 고정 너비 파일 쓰기 예제 (Fixed Width File Writing Example)

구분된 것은 플랫 파일 형식의 유일한 유형이 아닙니다.

많은 사람들은 필드를 구분하기 위해 각 열에 대해 설정된 너비를 사용하는 것을 선호하며, 이는 일반적으로 '고정 너비'라고 합니다.

Spring Batch는 `FormatterLineAggregator`를 사용하여 파일 쓰기에서 이를 지원합니다.

위에서 설명한 동일한 `CustomerCredit` 도메인 객체를 사용하여 Java에서는 다음과 같이 구성할 수 있습니다:

**Java 설정 (Java Configuration)**

```java
@Bean
public FlatFileItemWriter<CustomerCredit> itemWriter(Resource outputResource) throws Exception {
	BeanWrapperFieldExtractor<CustomerCredit> fieldExtractor = new BeanWrapperFieldExtractor<>();
	fieldExtractor.setNames(new String[] {"name", "credit"});
	fieldExtractor.afterPropertiesSet();

	FormatterLineAggregator<CustomerCredit> lineAggregator = new FormatterLineAggregator<>();
	lineAggregator.setFormat("%-9s%-2.0f"); // 포맷 문자열 설정
	lineAggregator.setFieldExtractor(fieldExtractor);

	return new FlatFileItemWriterBuilder<CustomerCredit>()
				.name("customerCreditWriter")
				.resource(outputResource)
				.lineAggregator(lineAggregator)
				.build();
}
```

앞의 예제 대부분은 익숙해 보일 것입니다. 그러나 `format` 속성의 값은 새로운 것입니다.

다음 예제는 Java의 `format` 속성을 보여줍니다:

```java
// ...
FormatterLineAggregator<CustomerCredit> lineAggregator = new FormatterLineAggregator<>();
lineAggregator.setFormat("%-9s%-2.0f"); // 첫 번째 필드는 9자리 문자열(왼쪽 정렬), 두 번째 필드는 소수점 없는 2자리 실수
// ...
```

기본 구현은 Java 5의 일부로 추가된 동일한 `Formatter`를 사용하여 구축됩니다.

Java `Formatter`는 C 프로그래밍 언어의 `printf` 기능을 기반으로 합니다. 포맷터를 구성하는 방법에 대한 대부분의 세부 정보는 `Formatter`의 Javadoc에서 찾을 수 있습니다.

다음 예제와 같이 `FlatFileItemWriterBuilder.FormattedBuilder`를 사용하여 `BeanWrapperFieldExtractor`와 `FormatterLineAggregator`를 자동으로 생성할 수도 있습니다:

**Java 설정 (Java Configuration)**

```java
@Bean
public FlatFileItemWriter<CustomerCredit> itemWriter(Resource outputResource) {
	return new FlatFileItemWriterBuilder<CustomerCredit>()
				.name("customerCreditWriter")
				.resource(outputResource)
				.formatted() // FormattedBuilder 사용 시작
				.format("%-9s%-2.0f") // 포맷 문자열
				.names(new String[] {"name", "credit"}) // 추출할 필드 이름 (내부적으로 BeanWrapperFieldExtractor 생성 및 설정)
				.build();
}
```

### 파일 생성 처리 (Handling File Creation)

`FlatFileItemReader`는 파일 리소스와 매우 간단한 관계를 가집니다.

리더가 초기화될 때 파일을 열고(존재하는 경우), 존재하지 않으면 예외를 발생시킵니다.

파일 쓰기는 그렇게 간단하지 않습니다.

언뜻 보기에는 `FlatFileItemWriter`에 대해서도 유사한 간단한 계약이 존재해야 할 것 같습니다: 파일이 이미 존재하면 예외를 발생시키고, 존재하지 않으면 생성하고 쓰기를 시작합니다.

그러나 잡(Job)을 잠재적으로 재시작하면 문제가 발생할 수 있습니다.

일반적인 재시작 시나리오에서 계약은 반대입니다: 파일이 존재하면 마지막으로 알려진 양호한 위치에서 쓰기를 시작하고, 존재하지 않으면 예외를 발생시킵니다.

그러나 이 잡의 파일 이름이 항상 같다면 어떻게 될까요? 이 경우 재시작이 아니라면 파일이 존재할 경우 파일을 삭제하고 싶을 것입니다.

이러한 가능성 때문에 `FlatFileItemWriter`에는 `shouldDeleteIfExists` 속성이 포함되어 있습니다.

이 속성을 `true`로 설정하면 라이터가 열릴 때 동일한 이름의 기존 파일이 삭제됩니다.

---

### **학습 목표 제시**

이번 "FlatFileItemWriter" 챕터를 통해 학생분은 다음을 이해하고 배울 수 있습니다:

1. **`FlatFileItemWriter`의 주요 구성 요소 및 속성 이해:** `Resource`, `LineAggregator` 등 `FlatFileItemWriter`를 설정하는 데 필요한 핵심 의존성과 `shouldDeleteIfExists`와 같은 파일 처리 관련 주요 속성들의 역할을 파악합니다.
2. **`LineAggregator`와 `FieldExtractor`의 역할과 상호 관계 파악:** 도메인 객체의 필드들이 어떻게 `FieldExtractor`에 의해 값의 배열로 추출되고, 이 배열이 다시 `LineAggregator`에 의해 최종 한 줄의 문자열로 조합되어 파일에 쓰여지는지 그 과정을 이해합니다.
3. **다양한 플랫 파일 형식으로 데이터 쓰기 방법 습득:** 구분자 기반 파일, 고정 너비 파일 형식으로 데이터를 쓰는 방법을 익히고, `PassThroughLineAggregator`, `DelimitedLineAggregator`, `FormatterLineAggregator`와 `PassThroughFieldExtractor`, `BeanWrapperFieldExtractor` 등 관련 구현체들의 사용법을 학습합니다.
4. **`FlatFileItemWriterBuilder`를 활용한 간결한 설정 방법 이해:** 빌더 패턴을 사용하여 `FlatFileItemWriter`와 그 내부 컴포넌트들을 보다 쉽고 간결하게 설정하는 방법을 익힙니다.

---

### **핵심 개념 설명**

이 챕터의 핵심은 **"처리된 자바 객체(아이템)를 어떻게 원하는 형식의 플랫 파일(CSV, 고정 길이 텍스트 등)로 변환하여 기록할 것인가?"** 입니다. 이를 위해 `FlatFileItemWriter`와 그 구성 요소들이 사용됩니다.

### 1. `FlatFileItemWriter` 개요

- **역할:** 처리된 아이템(자바 객체)들을 받아, 각 아이템을 한 줄의 문자열로 변환한 후 지정된 파일에 순차적으로 기록하는 `ItemWriter`의 구현체입니다.
- **주요 의존성:**
  1. **`Resource`**: 데이터를 기록할 파일의 위치를 나타냅니다. (예: `new FileSystemResource("output/result.csv")`)
  2. **`LineAggregator`**: 단일 아이템 객체(`T`)를 파일에 쓰여질 **한 줄의 문자열(String)로 변환(집계)**하는 핵심 인터페이스입니다.
