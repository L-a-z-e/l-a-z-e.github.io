---
title: Spring Batch - Flat Files(FlatFileItemReader)
description: 
author: laze
date: 2025-06-15 00:00:02 +0900
categories: [Dev, SpringBatch]
tags: [SpringBatch]
---
### 플랫 파일 아이템 리더 (FlatFileItemReader)

플랫 파일은 최대 2차원(표 형식) 데이터를 포함하는 모든 유형의 파일입니다.

Spring Batch 프레임워크에서 플랫 파일을 읽는 것은 `FlatFileItemReader`라는 클래스에 의해 용이해지며, 이 클래스는 플랫 파일을 읽고 파싱하기 위한 기본 기능을 제공합니다.

`FlatFileItemReader`의 가장 중요한 두 가지 필수 의존성은 `Resource`와 `LineMapper`입니다.

`LineMapper` 인터페이스는 다음 섹션에서 더 자세히 살펴봅니다.

`resource` 속성은 Spring Core의 `Resource`를 나타냅니다.

이 유형의 빈을 생성하는 방법에 대한 문서는 Spring 프레임워크, 5장. 리소스에서 찾을 수 있습니다.

따라서 이 가이드는 다음의 간단한 예제를 보여주는 것 외에는 `Resource` 객체 생성에 대한 자세한 내용을 다루지 않습니다:

```java
Resource resource = new FileSystemResource("resources/trades.csv");
```

복잡한 배치 환경에서는 디렉토리 구조가 종종 EAI(Enterprise Application Integration) 인프라에 의해 관리되며,

여기서 외부 인터페이스를 위한 드롭 존(drop zones)이 FTP 위치에서 배치 처리 위치로 파일을 이동시키거나 그 반대로 이동시키기 위해 설정됩니다.

파일 이동 유틸리티는 Spring Batch 아키텍처의 범위를 벗어나지만, 배치 잡 스트림이 잡 스트림의 단계로서 파일 이동 유틸리티를 포함하는 것은 드문 일이 아닙니다.

배치 아키텍처는 처리할 파일을 찾는 방법만 알면 됩니다. Spring Batch는 이 시작점에서 파이프로 데이터를 공급하는 프로세스를 시작합니다.

그러나 Spring Integration은 이러한 유형의 많은 서비스를 제공합니다.

`FlatFileItemReader`의 다른 속성들은 다음 표에 설명된 대로 데이터가 해석되는 방식을 추가로 지정할 수 있게 해줍니다:

**표 1. FlatFileItemReader 속성**

| 속성 | 타입 | 설명 |
| --- | --- | --- |
| `comments` | `String[]` | 주석 행을 나타내는 라인 접두사를 지정합니다. |
| `encoding` | `String` | 사용할 텍스트 인코딩을 지정합니다. 기본값은 UTF-8입니다. |
| `lineMapper` | `LineMapper` | `String`을 아이템을 나타내는 `Object`로 변환합니다. |
| `linesToSkip` | `int` | 파일 상단에서 무시할 라인 수입니다. |
| `recordSeparatorPolicy` | `RecordSeparatorPolicy` | 라인 끝이 어디인지 결정하고, 예를 들어 따옴표로 묶인 문자열 내부에 있는 경우 라인 끝을 넘어 계속 진행하는 등의 작업을 수행하는 데 사용됩니다. |
| `resource` | `Resource` | 읽어올 리소스입니다. |
| `skippedLinesCallback` | `LineCallbackHandler` | 파일에서 건너뛸 라인의 원시 라인 내용을 전달하는 인터페이스입니다. `linesToSkip`이 2로 설정되면 이 인터페이스는 두 번 호출됩니다. |
| `strict` | `boolean` | 엄격 모드(strict mode)에서는 입력 리소스가 존재하지 않으면 리더가 `ExecutionContext`에 예외를 발생시킵니다. 그렇지 않으면 문제를 기록하고 계속 진행합니다. |

### 라인 매퍼 (LineMapper)

`ResultSet`과 같은 저수준 구조를 받아 `Object`를 반환하는 `RowMapper`와 마찬가지로, 플랫 파일 처리에는 다음 인터페이스 정의와 같이 `String` 라인을 `Object`로 변환하기 위한 동일한 구조가 필요합니다:

```java
public interface LineMapper<T> {

    T mapLine(String line, int lineNumber) throws Exception;

}
```

기본 계약은, 현재 라인과 그것이 연관된 라인 번호가 주어지면, 매퍼가 결과 도메인 객체를 반환해야 한다는 것입니다.

이것은 `ResultSet`의 각 행이 행 번호에 연결되는 것처럼 각 라인이 라인 번호와 연관된다는 점에서 `RowMapper`와 유사합니다.

이를 통해 라인 번호를 ID 비교 또는 더 유익한 로깅을 위해 결과 도메인 객체에 연결할 수 있습니다.

그러나 `RowMapper`와 달리, `LineMapper`는 위에서 논의한 바와 같이 절반만 해결해 주는 원시 라인을 제공받습니다.

라인은 `FieldSet`으로 토큰화되어야 하며, 이 `FieldSet`은 이 문서의 뒷부분에서 설명하는 것처럼 객체로 매핑될 수 있습니다.

### 라인 토큰화기 (LineTokenizer)

입력 라인을 `FieldSet`으로 바꾸는 추상화가 필요한 이유는 `FieldSet`으로 변환해야 하는 플랫 파일 데이터의 형식이 많을 수 있기 때문입니다. Spring Batch에서 이 인터페이스는 `LineTokenizer`입니다:

```java
public interface LineTokenizer {

    FieldSet tokenize(String line);

}
```

`LineTokenizer`의 계약은, 입력 라인이 주어지면 (이론적으로 `String`은 한 줄 이상을 포함할 수 있음), 해당 라인을 나타내는 `FieldSet`이 반환된다는 것입니다.

이 `FieldSet`은 그런 다음 `FieldSetMapper`에 전달될 수 있습니다. Spring Batch에는 다음과 같은 `LineTokenizer` 구현이 포함되어 있습니다:

- `DelimitedLineTokenizer`: 레코드의 필드가 구분 기호로 분리된 파일에 사용됩니다. 가장 일반적인 구분 기호는 쉼표이지만, 파이프나 세미콜론도 자주 사용됩니다.
- `FixedLengthTokenizer`: 레코드의 각 필드가 "고정 너비"인 파일에 사용됩니다. 각 필드의 너비는 각 레코드 유형에 대해 정의되어야 합니다.
- `PatternMatchingCompositeLineTokenizer`: 패턴과 대조하여 특정 라인에 대해 토큰화기 목록 중 어떤 `LineTokenizer`를 사용해야 하는지 결정합니다.

### 필드셋 매퍼 (FieldSetMapper)

`FieldSetMapper` 인터페이스는 `FieldSet` 객체를 받아 그 내용을 객체로 매핑하는 단일 메서드 `mapFieldSet`을 정의합니다.

이 객체는 잡(job)의 필요에 따라 사용자 정의 DTO, 도메인 객체 또는 배열일 수 있습니다.

`FieldSetMapper`는 리소스의 데이터 라인을 원하는 유형의 객체로 변환하기 위해 `LineTokenizer`와 함께 사용되며, 다음 인터페이스 정의와 같습니다:

```java
public interface FieldSetMapper<T> {

    T mapFieldSet(FieldSet fieldSet) throws BindException;

}
```

사용되는 패턴은 `JdbcTemplate`에서 사용되는 `RowMapper`와 동일합니다.

### 기본 라인 매퍼 (DefaultLineMapper)

이제 플랫 파일을 읽어들이기 위한 기본 인터페이스가 정의되었으므로, 세 가지 기본 단계가 필요하다는 것이 분명해집니다:

1. 파일에서 한 줄을 읽습니다.
2. `String` 라인을 `LineTokenizer#tokenize()` 메서드에 전달하여 `FieldSet`을 검색합니다.
3. 토큰화에서 반환된 `FieldSet`을 `FieldSetMapper`에 전달하고, 그 결과를 `ItemReader#read()` 메서드에서 반환합니다.

위에서 설명한 두 인터페이스는 두 가지 개별 작업을 나타냅니다: 라인을 `FieldSet`으로 변환하는 것과 `FieldSet`을 도메인 객체로 매핑하는 것입니다.

`LineTokenizer`의 입력이 `LineMapper`의 입력(라인)과 일치하고, `FieldSetMapper`의 출력이 `LineMapper`의 출력과 일치하므로, `LineTokenizer`와 `FieldSetMapper`를 모두 사용하는 기본 구현이 제공됩니다.

다음 클래스 정의에 표시된 `DefaultLineMapper`는 대부분의 사용자에게 필요한 동작을 나타냅니다:

```java
public class DefaultLineMapper<T> implements LineMapper<T>, InitializingBean { // LineMapper<T>로 수정

    private LineTokenizer tokenizer;

    private FieldSetMapper<T> fieldSetMapper;

    public T mapLine(String line, int lineNumber) throws Exception {
        return fieldSetMapper.mapFieldSet(tokenizer.tokenize(line));
    }

    public void setLineTokenizer(LineTokenizer tokenizer) {
        this.tokenizer = tokenizer;
    }

    public void setFieldSetMapper(FieldSetMapper<T> fieldSetMapper) {
        this.fieldSetMapper = fieldSetMapper;
    }
}
```

위의 기능은 프레임워크의 이전 버전에서 수행되었던 것처럼 리더 자체에 내장되는 대신 기본 구현으로 제공되어, 특히 원시 라인에 대한 접근이 필요한 경우 사용자가 파싱 프로세스를 제어하는 데 더 큰 유연성을 제공합니다.

### 간단한 구분 파일 읽기 예제 (Simple Delimited File Reading Example)

다음 예제는 실제 도메인 시나리오로 플랫 파일을 읽는 방법을 보여줍니다. 이 특정 배치 잡은 다음 파일에서 축구 선수들을 읽어들입니다:

```
ID,lastName,firstName,position,birthYear,debutYear
"AbduKa00,Abdul-Jabbar,Karim,rb,1974,1996",  // 원문에는 각 줄이 따옴표로 묶여있으나, 일반적인 CSV 형식을 고려하여 제거함. 실제 데이터 형식에 따라 다를 수 있음.
AbduRa00,Abdullah,Rabih,rb,1975,1999,
AberWa00,Abercrombie,Walter,rb,1959,1982,
AbraDa00,Abramowicz,Danny,wr,1945,1967,
AdamBo00,Adams,Bob,te,1946,1969,
AdamCh00,Adams,Charlie,wr,1979,2003
```

이 파일의 내용은 다음 `Player` 도메인 객체로 매핑됩니다:

```java
public class Player implements Serializable {

    private String ID;
    private String lastName;
    private String firstName;
    private String position;
    private int birthYear;
    private int debutYear;

    public String toString() {
        return "PLAYER:ID=" + ID + ",Last Name=" + lastName +
            ",First Name=" + firstName + ",Position=" + position +
            ",Birth Year=" + birthYear + ",DebutYear=" +
            debutYear;
    }

    // setters and getters...
}
```

`FieldSet`을 `Player` 객체로 매핑하려면, 플레이어를 반환하는 `FieldSetMapper`를 다음 예제와 같이 정의해야 합니다:

```java
protected static class PlayerFieldSetMapper implements FieldSetMapper<Player> {
    public Player mapFieldSet(FieldSet fieldSet) {
        Player player = new Player();

        player.setID(fieldSet.readString(0));
        player.setLastName(fieldSet.readString(1));
        player.setFirstName(fieldSet.readString(2));
        player.setPosition(fieldSet.readString(3));
        player.setBirthYear(fieldSet.readInt(4));
        player.setDebutYear(fieldSet.readInt(5));

        return player;
    }
}
```

그런 다음 `FlatFileItemReader`를 올바르게 구성하고 `read`를 호출하여 파일을 읽을 수 있습니다. 다음 예제와 같습니다:

```java
FlatFileItemReader<Player> itemReader = new FlatFileItemReader<>();
itemReader.setResource(new FileSystemResource("resources/players.csv"));
DefaultLineMapper<Player> lineMapper = new DefaultLineMapper<>();
//DelimitedLineTokenizer는 기본적으로 쉼표를 구분 기호로 사용합니다.
lineMapper.setLineTokenizer(new DelimitedLineTokenizer());
lineMapper.setFieldSetMapper(new PlayerFieldSetMapper());
itemReader.setLineMapper(lineMapper);
itemReader.open(new ExecutionContext()); // ItemStream의 open 호출 (필수)
Player player = itemReader.read();
```

`read`를 호출할 때마다 파일의 각 라인에서 새로운 `Player` 객체가 반환됩니다. 파일의 끝에 도달하면 `null`이 반환됩니다.

### 이름으로 필드 매핑하기 (Mapping Fields by Name)

`DelimitedLineTokenizer`와 `FixedLengthTokenizer` 모두에서 허용되며 JDBC `ResultSet`의 기능과 유사한 추가 기능이 하나 있습니다.

매핑 함수의 가독성을 높이기 위해 필드 이름을 이러한 `LineTokenizer` 구현 중 하나에 주입할 수 있습니다. 먼저, 플랫 파일의 모든 필드에 대한 열 이름을 다음 예제와 같이 토큰화기에 주입합니다:

```java
tokenizer.setNames(new String[] {"ID", "lastName", "firstName", "position", "birthYear", "debutYear"});
```

`FieldSetMapper`는 이 정보를 다음과 같이 사용할 수 있습니다:

```java
public class PlayerMapper implements FieldSetMapper<Player> {
    public Player mapFieldSet(FieldSet fs) {

       if (fs == null) { // FieldSet이 null인 경우 방어 코드 (이론적으로 LineMapper가 호출 시 null이 아닌 FieldSet을 전달)
           return null;
       }

       Player player = new Player();
       player.setID(fs.readString("ID"));
       player.setLastName(fs.readString("lastName"));
       player.setFirstName(fs.readString("firstName"));
       player.setPosition(fs.readString("position"));
       player.setDebutYear(fs.readInt("debutYear"));
       player.setBirthYear(fs.readInt("birthYear")); // 순서가 원본과 다름을 주의 (이름 기반이므로 순서 무관)

       return player;
   }
}
```

### FieldSet을 도메인 객체로 자동 매핑하기 (Automapping FieldSets to Domain Objects)

많은 사람들에게 특정 `FieldSetMapper`를 작성하는 것은 `JdbcTemplate`에 대한 특정 `RowMapper`를 작성하는 것만큼 번거롭습니다.

Spring Batch는 JavaBean 사양을 사용하여 필드 이름과 객체의 세터(setter)를 일치시켜 필드를 자동으로 매핑하는 `FieldSetMapper`

다시 축구 예제를 사용하면, `BeanWrapperFieldSetMapper` 구성은 Java에서 다음 스니펫과 같습니다:

**Java 설정 (Java Configuration)**

```java
@Bean
public FieldSetMapper<Player> fieldSetMapper() { // 반환 타입을 Player로 명시
	BeanWrapperFieldSetMapper<Player> fieldSetMapper = new BeanWrapperFieldSetMapper<>(); // 제네릭 타입 명시

	// fieldSetMapper.setPrototypeBeanName("player"); // 이 방식 대신 targetType 사용 권장
    fieldSetMapper.setTargetType(Player.class); // 대상 타입을 직접 지정하는 것이 더 일반적이고 타입 안전함

	return fieldSetMapper;
}

// @Bean  // BeanWrapperFieldSetMapper가 내부적으로 new Player()를 하므로,
// @Scope("prototype") // 이 player 빈은 BeanWrapperFieldSetMapper에 의해 직접 사용되지 않음.
// public Player player() { // setPrototypeBeanName을 사용한다면 필요.
//	return new Player();
// }
```

`FieldSet`의 각 항목에 대해, 매퍼는 Spring 컨테이너가 속성 이름과 일치하는 세터를 찾는 것과 동일한 방식으로 `Player` 객체의 새 인스턴스(이러한 이유로 프로토타입 스코프가 필요함 - `setPrototypeBeanName` 사용 시)에서 해당 세터를 찾습니다. `FieldSet`에서 사용 가능한 각 필드가 매핑되고, 결과 `Player` 객체가 코드가 필요 없이 반환됩니다.

### 고정 길이 파일 형식 (Fixed Length File Formats)

지금까지는 구분 파일에 대해서만 자세히 설명했습니다. 그러나 이는 파일 읽기 그림의 절반만을 나타냅니다. 플랫 파일을 사용하는 많은 조직에서는 고정 길이 형식을 사용합니다. 다음은 고정 길이 파일 예제입니다:

```
UK21341EAH4121131.11customer1
UK21341EAH4221232.11customer2
UK21341EAH4321333.11customer3
UK21341EAH4421434.11customer4
UK21341EAH4521535.11customer5
```

이것은 하나의 큰 필드처럼 보이지만, 실제로는 4개의 고유한 필드를 나타냅니다:

- ISIN: 주문되는 아이템의 고유 식별자 - 12자 길이.
- Quantity: 주문되는 아이템의 수량 - 3자 길이.
- Price: 아이템의 가격 - 5자 길이.
- Customer: 아이템을 주문하는 고객의 ID - 9자 길이.

`FixedLengthLineTokenizer`를 구성할 때, 이러한 각 길이는 범위 형태로 제공되어야 합니다.

다음 예제는 Java에서 `FixedLengthLineTokenizer`에 대한 범위를 정의하는 방법을 보여줍니다:

```java
@Bean
public FixedLengthTokenizer fixedLengthTokenizer() {
	FixedLengthTokenizer tokenizer = new FixedLengthTokenizer();

	tokenizer.setNames("ISIN", "Quantity", "Price", "Customer");
	tokenizer.setColumns(new Range(1, 12),   // 1부터 12까지 (12자)
						new Range(13, 15),  // 13부터 15까지 (3자)
						new Range(16, 20),  // 16부터 20까지 (5자)
						new Range(21, 29)); // 21부터 29까지 (9자)

	return tokenizer;
}
```

`FixedLengthLineTokenizer`는 위에서 설명한 것과 동일한 `LineTokenizer` 인터페이스를 사용하므로, 구분 기호가 사용된 것처럼 동일한 `FieldSet`을 반환합니다.

이를 통해 `BeanWrapperFieldSetMapper`를 사용하는 것과 같이 출력을 처리하는 데 동일한 접근 방식을 사용할 수 있습니다.

### 단일 파일 내 여러 레코드 유형 (Multiple Record Types within a Single File)

지금까지의 모든 파일 읽기 예제는 단순화를 위해 한 가지 핵심 가정을 했습니다:

파일의 모든 레코드가 동일한 형식을 갖는다는 것입니다.

그러나 항상 그런 것은 아닙니다.

파일에 다르게 토큰화되고 다른 객체로 매핑되어야 하는 다른 형식의 레코드가 있을 수 있는 경우가 매우 흔합니다.

```
USER;Smith;Peter;;T;20014539;F
LINEA;1044391041ABC037.49G201XX1383.12H
LINEB;2134776319DEF422.99M005LI
```

이 파일에는 "USER", "LINEA", "LINEB"의 세 가지 유형의 레코드가 있습니다.

"USER" 라인은 `User` 객체에 해당합니다.

"LINEA"와 "LINEB"는 모두 `Line` 객체에 해당하지만, "LINEA"는 "LINEB"보다 더 많은 정보를 가지고 있습니다.

`ItemReader`는 각 라인을 개별적으로 읽지만, `ItemWriter`가 올바른 아이템을 받도록 다른 `LineTokenizer`와 `FieldSetMapper` 객체를 지정해야 합니다.

`PatternMatchingCompositeLineMapper`는 패턴과 `LineTokenizer`의 맵, 패턴과 `FieldSetMapper`의 맵을 구성할 수 있게 하여 이를 쉽게 만듭니다.

```java
@Bean
public PatternMatchingCompositeLineMapper<Object> orderFileLineMapper( // 반환 타입을 Object 또는 공통 상위 타입으로
    LineTokenizer userTokenizer, LineTokenizer lineATokenizer, LineTokenizer lineBTokenizer,
    FieldSetMapper<User> userFieldSetMapper, FieldSetMapper<Line> lineFieldSetMapper) { // 각 토큰화기/매퍼 빈 주입
	PatternMatchingCompositeLineMapper<Object> lineMapper =
		new PatternMatchingCompositeLineMapper<>();

	Map<String, LineTokenizer> tokenizers = new HashMap<>(3);
	tokenizers.put("USER*", userTokenizer); // userTokenizer() 대신 주입받은 빈 사용
	tokenizers.put("LINEA*", lineATokenizer);
	tokenizers.put("LINEB*", lineBTokenizer);

	lineMapper.setTokenizers(tokenizers);

	Map<String, FieldSetMapper<?>> mappers = new HashMap<>(2); // 와일드카드 사용
	mappers.put("USER*", userFieldSetMapper);
	mappers.put("LINE*", lineFieldSetMapper); // LINEA와 LINEB 모두 이 매퍼 사용

	lineMapper.setFieldSetMappers(mappers);

	return lineMapper;
}

// 각 토큰화기 및 필드셋 매퍼에 대한 빈 정의 필요
// 예시:
// @Bean public LineTokenizer userTokenizer() { /* ... */ }
// @Bean public FieldSetMapper<User> userFieldSetMapper() { /* ... */ }
// ...
```

이 예제에서 "LINEA"와 "LINEB"는 별도의 `LineTokenizer` 인스턴스를 갖지만, 둘 다 동일한 `FieldSetMapper`를 사용합니다.

`PatternMatchingCompositeLineMapper`는 각 라인에 대해 올바른 델리게이트를 선택하기 위해 `PatternMatcher#match` 메서드를 사용합니다.

`PatternMatcher`는 두 가지 와일드카드 문자를 특별한 의미로 허용합니다: 물음표("?")는 정확히 하나의 문자와 일치하고, 별표("*")는 0개 이상의 문자와 일치합니다.*

*위 구성에서 모든 패턴은 별표로 끝나므로 사실상 라인의 접두사 역할을 합니다.*

`*PatternMatcher`는 구성 순서에 관계없이 항상 가능한 가장 구체적인 패턴과 일치합니다.*

*따라서 "LINE\*"과 "LINEA\*"가 모두 패턴으로 나열된 경우, "LINEA"는 패턴 "LINEA\*"와 일치하고, "LINEB"는 패턴 "LINE\*"와 일치합니다. 또한, 단일 별표("\*")는 다른 어떤 패턴과도 일치하지 않는 모든 라인과 일치하여 기본값 역할을 할 수 있습니다.

다음 예제는 Java에서 다른 어떤 패턴과도 일치하지 않는 라인을 일치시키는 방법을 보여줍니다:

```java
// ...
tokenizers.put("*", defaultLineTokenizer()); // defaultLineTokenizer 빈을 주입받아 사용
// ...
```

토큰화에만 사용할 수 있는 `PatternMatchingCompositeLineTokenizer`도 있습니다.

플랫 파일에 각 레코드가 여러 줄에 걸쳐 있는 레코드가 포함되는 것도 일반적입니다. 이 상황을 처리하려면 더 복잡한 전략이 필요합니다.

이 일반적인 패턴의 데모는 `multiLineRecords` 샘플에서 찾을 수 있습니다.

### 플랫 파일의 예외 처리 (Exception Handling in Flat Files)

라인을 토큰화할 때 예외가 발생할 수 있는 시나리오는 많습니다.

많은 플랫 파일은 불완전하며 잘못된 형식의 레코드를 포함합니다.

많은 사용자는 이러한 오류가 있는 라인을 건너뛰면서 문제, 원본 라인 및 라인 번호를 기록하도록 선택합니다.

이러한 로그는 나중에 수동으로 또는 다른 배치 잡에 의해 검사될 수 있습니다.

이러한 이유로 Spring Batch는 파스 예외를 처리하기 위한 예외 계층 구조를 제공합니다: `FlatFileParseException` 및 `FlatFileFormatException`.

`FlatFileParseException`은 파일을 읽으려고 할 때 오류가 발생하면 `FlatFileItemReader`에 의해 발생합니다.

`FlatFileFormatException`은 `LineTokenizer` 인터페이스의 구현에 의해 발생하며 토큰화 중에 발생한 더 구체적인 오류를 나타냅니다.

### IncorrectTokenCountException

`DelimitedLineTokenizer`와 `FixedLengthLineTokenizer` 모두 `FieldSet`을 만드는 데 사용할 수 있는 열 이름을 지정하는 기능이 있습니다.

그러나 열 이름 수가 라인을 토큰화하는 동안 발견된 열 수와 일치하지 않으면 `FieldSet`을 만들 수 없으며, 다음 예제와 같이 발생한 토큰 수와 예상 수를 포함하는 `IncorrectTokenCountException`이 발생합니다:

```java
tokenizer.setNames(new String[] {"A", "B", "C", "D"});

try {
    tokenizer.tokenize("a,b,c");
}
catch (IncorrectTokenCountException e) {
    assertEquals(4, e.getExpectedCount());
    assertEquals(3, e.getActualCount());
}
```

토큰화기가 4개의 열 이름으로 구성되었지만 파일에서 3개의 토큰만 발견되었으므로 `IncorrectTokenCountException`이 발생했습니다.

### IncorrectLineLengthException

고정 길이 형식으로 포맷된 파일은 파싱할 때 추가 요구 사항이 있습니다.

왜냐하면 구분된 형식과 달리 각 열은 미리 정의된 너비를 엄격하게 준수해야 하기 때문입니다.

총 라인 길이가 이 열의 가장 넓은 값과 같지 않으면 다음 예제와 같이 예외가 발생합니다:

```java
tokenizer.setColumns(new Range[] { new Range(1, 5), // 실제로는 Range 객체 배열이어야 함
                                   new Range(6, 10),
                                   new Range(11, 15) });
try {
    tokenizer.tokenize("12345");
    fail("Expected IncorrectLineLengthException");
}
catch (IncorrectLineLengthException ex) {
    assertEquals(15, ex.getExpectedLength());
    assertEquals(5, ex.getActualLength());
}
```

위 토큰화기에 대해 구성된 범위는 1-5, 6-10, 11-15입니다.

결과적으로 라인의 총 길이는 15입니다.

그러나 위 예제에서는 길이 5의 라인이 전달되어 `IncorrectLineLengthException`이 발생했습니다.

여기서 예외를 발생시키면 `FieldSetMapper`에서 열 2를 읽으려고 할 때 실패하는 것보다 더 일찍, 더 많은 정보와 함께 라인 처리가 실패할 수 있습니다.

그러나 라인 길이가 항상 일정하지 않은 시나리오도 있습니다.

이러한 이유로 다음 예제와 같이 'strict' 속성을 통해 라인 길이 유효성 검사를 끌 수 있습니다:

```java
tokenizer.setColumns(new Range[] { new Range(1, 5), new Range(6, 10) }); // Range 배열
tokenizer.setStrict(false);
FieldSet tokens = tokenizer.tokenize("12345");
assertEquals("12345", tokens.readString(0));
assertEquals("", tokens.readString(1)); // 두 번째 필드는 빈 문자열로 채워짐
```

위 예제는 `tokenizer.setStrict(false)`가 호출되었다는 점을 제외하고는 이전 예제와 거의 동일합니다.

이 설정은 토큰화기가 라인을 토큰화할 때 라인 길이를 강제하지 않도록 지시합니다.

이제 `FieldSet`이 올바르게 생성되고 반환됩니다. 그러나 나머지 값에 대해서는 빈 토큰만 포함합니다.

---

### **학습 목표 제시**

이번 "FlatFileItemReader" 챕터를 통해 학생분은 다음을 이해하고 배울 수 있습니다:

1. **`FlatFileItemReader`의 주요 구성 요소 및 속성 이해:** `Resource`, `LineMapper` 등 `FlatFileItemReader`를 설정하는 데 필요한 핵심 의존성과 `comments`, `linesToSkip`, `encoding` 등 주요 설정 속성들의 역할을 파악합니다.
2. **`LineMapper`, `LineTokenizer`, `FieldSetMapper`의 역할과 상호 관계 파악:** 플랫 파일의 한 줄이 어떻게 `LineTokenizer`에 의해 `FieldSet`으로 변환되고, 다시 `FieldSetMapper`에 의해 최종 도메인 객체로 매핑되는지 그 과정을 이해하며, `DefaultLineMapper`가 이 둘을 어떻게 조합하는지 학습합니다.
3. **다양한 플랫 파일 형식 처리 방법 습득 및 예외 처리 이해:** 구분자 기반 파일, 고정 길이 파일, 단일 파일 내 여러 레코드 유형 처리 방법을 익히고, 플랫 파일 처리 시 발생할 수 있는 주요 예외(`FlatFileParseException`, `IncorrectTokenCountException`, `IncorrectLineLengthException`)와 대처 방법을 이해합니다.

---

### **핵심 개념 설명**

이 챕터의 핵심은 **"다양한 형태의 플랫 파일을 어떻게 효과적으로 읽고, 그 내용을 우리가 원하는 자바 객체로 변환할 것인가?"** 입니다. 그 중심에는 `FlatFileItemReader`와 이를 구성하는 여러 헬퍼 인터페이스 및 클래스들이 있습니다.

### 1. `FlatFileItemReader` 개요

- **역할:** 플랫 파일(CSV, 고정 길이 텍스트 파일 등)로부터 데이터를 읽어와 아이템(보통 자바 객체) 형태로 제공하는 `ItemReader`의 구현체입니다.
- **주요 의존성:**
  1. **`Resource`**: 읽어올 파일의 위치를 나타냅니다. (예: `new FileSystemResource("data/input.csv")`) 스프링 프레임워크의 `Resource` 추상화를 사용합니다.
  2. **`LineMapper`**: 파일에서 읽은 **한 줄(String)을 최종 아이템 객체(T)로 변환**하는 핵심 인터페이스입니다.
- **비유:** 똑똑한 문서 처리 로봇.
  - 로봇(`FlatFileItemReader`)에게 "이 문서(`Resource`)를 가져와서, 각 줄을 이런 규칙(`LineMapper`)에 따라 해석해서, 특정 정보(`T` 타입 객체)로 만들어줘!" 라고 명령하는 것과 같습니다.

### 2. `FlatFileItemReader`의 주요 속성 (설정값들)

문서의 표에 나온 주요 속성들을 다시 한번 살펴보겠습니다.

- **`comments` (String[])**: 특정 문자열로 시작하는 줄을 주석으로 간주하고 건너뛸 때 사용합니다. (예: `comments = new String[]{"#", "//"}`)
- **`encoding` (String)**: 파일의 인코딩을 지정합니다. (기본값: "UTF-8") 한글 파일 등을 읽을 때 중요합니다.
- **`lineMapper` (LineMapper)**: 가장 중요한 설정! 파일의 한 줄을 어떻게 객체로 변환할지 정의합니다. (아래에서 자세히 설명)
- **`linesToSkip` (int)**: 파일의 맨 위에서부터 지정된 수만큼의 줄을 건너뜁니다. (예: 헤더 라인 무시)
- **`recordSeparatorPolicy` (RecordSeparatorPolicy)**: 한 레코드(줄)가 어떻게 끝나는지를 결정하는 정책입니다. 기본적으로는 일반적인 줄바꿈 문자(`\\n`, `\\r\\n`)를 사용하지만, 따옴표 안의 줄바꿈을 무시하고 하나의 레코드로 이어 읽는 등의 고급 처리가 필요할 때 커스텀 정책을 사용할 수 있습니다.
- **`resource` (Resource)**: 읽어올 파일 리소스. (필수 설정)
- **`skippedLinesCallback` (LineCallbackHandler)**: `linesToSkip`으로 인해 건너뛰는 각 줄에 대해 어떤 작업을 수행하고 싶을 때 사용합니다. (예: 건너뛰는 줄 로깅)
- **`strict` (boolean)**: `true` (기본값)이면 `resource`로 지정된 파일이 없을 때 예외를 발생시키고 잡을 실패시킵니다. `false`이면 경고 로그만 남기고 (아이템 없이) 스텝을 계속 진행합니다.

### 3. `LineMapper`의 작동 원리: 3단계 프로세스

`FlatFileItemReader`가 파일에서 한 줄을 읽으면, 이 줄은 `LineMapper`에게 전달되어 최종 객체로 변환됩니다. `LineMapper`는 보통 내부적으로 두 가지 주요 작업을 수행합니다.

**`LineMapper<T>` 인터페이스:**

```java
public interface LineMapper<T> {
    T mapLine(String line, int lineNumber) throws Exception;
}
```

- `line`: 파일에서 읽은 한 줄의 문자열.
- `lineNumber`: 해당 라인의 번호 (1부터 시작). 로깅이나 오류 추적에 유용.
- 반환값 `T`: 최종적으로 변환된 객체.

이 `mapLine` 메서드 내부에서는 보통 다음과 같은 일들이 일어납니다 (특히 `DefaultLineMapper` 사용 시).

**1단계: 라인 토큰화 (Line Tokenization) - `LineTokenizer`**

- **역할:** 전달받은 한 줄의 문자열(`line`)을 의미 있는 조각들(토큰)로 나누어 `FieldSet` 객체로 만드는 역할.
- **`LineTokenizer` 인터페이스:**

    ```java
    public interface LineTokenizer {
        FieldSet tokenize(String line);
    }
    ```

- **주요 구현체:**
  - **`DelimitedLineTokenizer`**: 쉼표(,), 세미콜론(;), 파이프(|) 등 **구분자**로 필드가 나뉘는 파일에 사용.
    - `setDelimiter(String delimiter)`: 구분자 지정 (기본값: 쉼표).
    - `setNames(String[] names)`: 각 필드의 이름을 지정할 수 있음 (선택 사항).
    - `setQuoteCharacter(char quoteCharacter)`: 필드를 감싸는 따옴표 문자 지정 (기본값: `"`).
  - **`FixedLengthTokenizer`**: 각 필드가 **고정된 길이**를 가지는 파일에 사용.
    - `setColumns(Range[] ranges)`: 각 필드의 시작과 끝 위치를 `Range` 객체의 배열로 지정. (예: `new Range(1,5)`, `new Range(6,10)`)
    - `setNames(String[] names)`: 필드 이름 지정 가능.
    - `setStrict(boolean strict)`: (기본값 `true`) 라인의 총 길이가 설정된 범위의 합과 정확히 일치해야 하는지 여부. `false`로 하면 짧은 라인도 허용하고 부족한 부분은 빈 값으로 처리. (예외 처리 섹션에서 자세히)

**2단계: `FieldSet` 생성**

- `LineTokenizer`가 `tokenize()` 메서드를 통해 반환하는 것이 바로 이전 챕터에서 배운 `FieldSet` 객체입니다.
- 이 `FieldSet`은 해당 라인의 데이터를 타입 변환 가능한 형태로 가지고 있습니다.

**3단계: `FieldSet`을 객체로 매핑 (Object Mapping) - `FieldSetMapper`**

- **역할:** `LineTokenizer`가 만들어준 `FieldSet` 객체를 받아서, 최종적으로 원하는 자바 도메인 객체(`T` 타입)로 변환하는 역할.
- **`FieldSetMapper<T>` 인터페이스:**

    ```java
    public interface FieldSetMapper<T> {
        T mapFieldSet(FieldSet fieldSet) throws BindException;
    }
    ```

- **주요 구현체/방식:**
  - **직접 구현:** 개발자가 `FieldSetMapper` 인터페이스를 직접 구현하여, `mapFieldSet` 메서드 내에서 `fieldSet.readString("name")`, `fieldSet.readInt("age")` 등으로 값을 읽어와 객체를 생성하고 필드를 채웁니다. (예제의 `PlayerFieldSetMapper`)
  - **`BeanWrapperFieldSetMapper<T>`**: 리플렉션을 사용하여 `FieldSet`의 필드 이름과 대상 객체(`T`)의 프로퍼티(setter 메서드) 이름을 자동으로 매칭하여 값을 채워줍니다. 코드를 훨씬 줄일 수 있습니다.
    - `setTargetType(Class<T> targetType)`: 매핑할 대상 객체의 클래스 타입을 지정합니다. (예: `Player.class`)
    - `setPrototypeBeanName(String name)` (덜 권장): 스프링 컨텍셔너에 프로토타입 스코프로 등록된 빈 이름을 지정하여 매번 새로운 객체를 가져와 사용합니다. `targetType`을 사용하는 것이 더 간단하고 명확합니다.

**`DefaultLineMapper<T>`: 위의 1, 3단계를 합친 편리한 구현체**

Spring Batch는 `LineTokenizer`와 `FieldSetMapper`를 조합하여 사용하는 일반적인 패턴을 위해 `DefaultLineMapper`를 제공합니다.

```java
public class DefaultLineMapper<T> implements LineMapper<T>, InitializingBean {
    private LineTokenizer tokenizer;
    private FieldSetMapper<T> fieldSetMapper;

    public T mapLine(String line, int lineNumber) throws Exception {
        // 1. tokenizer로 라인을 FieldSet으로 변환
        FieldSet fieldSet = tokenizer.tokenize(line);
        // 2. fieldSetMapper로 FieldSet을 객체 T로 변환
        return fieldSetMapper.mapFieldSet(fieldSet);
    }
    // ... setters for tokenizer and fieldSetMapper ...
}
```

개발자는 `FlatFileItemReader`에 `DefaultLineMapper`를 설정하고, 이 `DefaultLineMapper`에는 적절한 `LineTokenizer`와 `FieldSetMapper`를 설정해주면 됩니다.

### 4. 다양한 파일 형식 처리

- **구분자 기반 파일 (Delimited Files):** `DelimitedLineTokenizer` 사용. (예제: `players.csv`)
  - 필드 이름을 `tokenizer.setNames(...)`로 설정하면 `FieldSetMapper`에서 이름으로 필드 접근 가능.
- **고정 길이 파일 (Fixed Length Files):** `FixedLengthTokenizer` 사용.
  - `tokenizer.setColumns(new Range(1,12), ...)` 형태로 각 필드의 범위를 지정.
  - 마찬가지로 필드 이름 설정 가능.
- **한 파일 내에 여러 레코드 타입이 섞여 있는 경우:** `PatternMatchingCompositeLineMapper` 사용.
  - **`PatternMatchingCompositeLineMapper`**: 라인의 내용(패턴)에 따라 서로 다른 `LineTokenizer`와 `FieldSetMapper`를 선택적으로 적용할 수 있게 해줍니다.
  - `setTokenizers(Map<String, LineTokenizer> tokenizers)`: 패턴(예: "USER\*", "LINEA\*")과 해당 패턴일 때 사용할 `LineTokenizer`를 매핑.
  - `setFieldSetMappers(Map<String, FieldSetMapper> mappers)`: 패턴과 사용할 `FieldSetMapper`를 매핑.
  - 패턴 매칭 규칙: 와일드카드 `?` (한 글자),  (0개 이상 문자). 가장 구체적인 패턴이 우선. 는 기본값(default)으로 사용 가능.
  - 이를 통해 "USER"로 시작하는 라인은 `User` 객체로, "LINE"으로 시작하는 라인은 `Line` 객체로 변환하는 등의 처리가 가능.

### 5. 예외 처리

플랫 파일은 형식이 깨지기 쉽습니다. Spring Batch는 이를 위한 예외 계층 구조를 제공합니다.

- **`FlatFileParseException`**: `FlatFileItemReader`가 파일을 읽는 중 발생하는 일반적인 파싱 관련 최상위 예외.
- **`FlatFileFormatException`**: `LineTokenizer` 구현체들이 토큰화 중 더 구체적인 형식 오류를 만났을 때 발생.
  - **`IncorrectTokenCountException`**: `DelimitedLineTokenizer`나 `FixedLengthTokenizer`에서 `setNames()`로 설정한 필드 이름의 개수와 실제 토큰화된 필드의 개수가 다를 때 발생. (예: 이름은 4개인데 토큰은 3개)
  - **`IncorrectLineLengthException`**: `FixedLengthTokenizer`에서 `strict=true` (기본값)일 때, 라인의 실제 길이가 `setColumns()`로 설정된 전체 길이와 다를 때 발생.
    - `setStrict(false)`로 설정하면 이 예외를 발생시키지 않고, 부족한 필드는 빈 값으로 채우거나 긴 라인은 잘라낼 수 있습니다 (구현에 따라 다름, 보통 빈 값).

이러한 예외들은 Spring Batch의 skip/retry 정책과 연동되어, 오류가 발생한 라인을 건너뛰거나, 로깅하거나, 잡을 실패시키는 등의 처리를 할 수 있습니다.

---

### **주요 용어 해설**

- **드롭 존 (Drop Zone):** 외부 시스템과 파일을 주고받기 위해 마련된 특정 디렉토리 위치. EAI 시스템에서 자주 사용.
- **EAI (Enterprise Application Integration):** 기업 내 다양한 애플리케이션과 시스템들을 통합하는 기술 및 프로세스.
- **토큰화 (Tokenization):** 문자열을 의미 있는 단위(토큰)로 분리하는 과정.
- **바인딩 예외 (BindException):** `FieldSetMapper`가 `FieldSet`의 데이터를 객체의 필드에 바인딩(매핑)하려 할 때 발생하는 예외 (예: 타입 불일치, 필수 값 누락 등).
- **프로토타입 스코프 (Prototype Scope):** 스프링에서 빈을 요청할 때마다 항상 새로운 인스턴스를 생성하는 빈 스코프. `BeanWrapperFieldSetMapper`에서 `setPrototypeBeanName` 사용 시 필요.
- **와일드카드 (Wildcard):** 패턴 매칭에서 여러 문자를 대표하는 특수 문자 (예: , `?`).

---

### **코드 예제 분석 (간단한 구분 파일 읽기)**

```java
// 1. ItemReader 생성 및 리소스 설정
FlatFileItemReader<Player> itemReader = new FlatFileItemReader<>();
itemReader.setResource(new FileSystemResource("resources/players.csv")); // 어떤 파일을 읽을 것인가

// 2. LineMapper 설정 (DefaultLineMapper 사용)
DefaultLineMapper<Player> lineMapper = new DefaultLineMapper<>();

// 2-1. LineTokenizer 설정 (쉼표로 구분된 파일이므로 DelimitedLineTokenizer)
DelimitedLineTokenizer lineTokenizer = new DelimitedLineTokenizer();
// lineTokenizer.setNames(new String[]{"ID", "lastName", ...}); // (선택) 필드 이름 설정하면 PlayerFieldSetMapper에서 이름으로 접근 가능
lineMapper.setLineTokenizer(lineTokenizer); // DefaultLineMapper에 토큰화기 설정

// 2-2. FieldSetMapper 설정 (직접 구현한 PlayerFieldSetMapper)
PlayerFieldSetMapper fieldSetMapper = new PlayerFieldSetMapper();
lineMapper.setFieldSetMapper(fieldSetMapper); // DefaultLineMapper에 매퍼 설정

// 3. 최종적으로 ItemReader에 LineMapper 설정
itemReader.setLineMapper(lineMapper);

// (스텝 생명주기 내에서는 프레임워크가 호출)
// itemReader.open(new ExecutionContext()); // 사용 전 스트림 열기 (ItemStream 구현 시)

// 4. 데이터 읽기
// Player player1 = itemReader.read();
// Player player2 = itemReader.read();
// ... 파일 끝까지 읽으면 null 반환
```

이 예제는 `FlatFileItemReader`를 구성하는 가장 기본적인 단계를 보여줍니다.

1. 어떤 파일을 읽을지 (`setResource`).
2. 읽은 각 라인을 어떻게 객체로 변환할지 (`setLineMapper`).
  - `LineMapper`는 다시, 라인을 어떻게 토큰으로 나눌지 (`setLineTokenizer`).
  - 그리고 그 토큰들(`FieldSet`)을 어떻게 최종 객체로 매핑할지 (`setFieldSetMapper`)를 설정합니다.

이 구조를 이해하는 것이 `FlatFileItemReader` 사용의 핵심입니다!

---

### **"왜?" 라는 질문에 대한 답변**

- **왜 `LineMapper`는 `LineTokenizer`와 `FieldSetMapper`로 분리되었을까?**
  - **유연성 및 재사용성:** 라인을 토큰화하는 방식(구분자, 고정 길이 등)과 `FieldSet`을 객체로 매핑하는 방식은 서로 독립적으로 변경되거나 재사용될 수 있습니다. 예를 들어, 동일한 `FieldSetMapper`를 사용하면서 `DelimitedLineTokenizer` 대신 `FixedLengthLineTokenizer`를 사용하고 싶을 수 있습니다. 또는, 같은 `LineTokenizer`로 `FieldSet`을 만들되, 서로 다른 정보를 추출하는 여러 `FieldSetMapper`를 사용할 수도 있습니다. 이렇게 역할을 분리하면 각 컴포넌트를 독립적으로 개발하고 테스트하며 조합하기 용이합니다.
  - **관심사 분리:** 토큰화는 "구문 분석"에 가깝고, `FieldSet` 매핑은 "데이터 바인딩 및 객체 생성"에 가깝습니다. 서로 다른 관심사를 분리하여 각 컴포넌트가 하나의 책임에만 집중하도록 합니다.
- **`BeanWrapperFieldSetMapper`는 왜 프로토타입 스코프 빈을 (과거에) 요구했거나, `targetType`을 사용할까?**
  - `BeanWrapperFieldSetMapper`는 매핑할 때마다 **새로운** 대상 객체 인스턴스가 필요합니다. 만약 싱글톤 빈을 사용한다면, 모든 라인이 동일한 객체 인스턴스에 계속 덮어쓰여 마지막 라인의 데이터만 남게 될 것입니다.
  - `setPrototypeBeanName("playerBean")` 방식은 스프링 컨테이너에게 "playerBean"이라는 이름의 프로토타입 빈을 매번 새로 요청하여 사용합니다.
  - `setTargetType(Player.class)` 방식은 `BeanWrapperFieldSetMapper`가 내부적으로 `Player.class.newInstance()` (또는 이와 유사한 리플렉션 방식)를 호출하여 매번 새로운 `Player` 객체를 직접 생성합니다. 이 방식이 스프링 컨테이너에 대한 의존성을 줄이고 더 명확하기 때문에 최근에 더 선호됩니다.

---

### **주의사항 및 Best Practice**

1. **리소스 경로 및 인코딩 확인:** `setResource()`로 파일 경로를 정확히 지정하고, 파일 인코딩에 맞춰 `setEncoding()`을 (필요하다면) 설정해야 합니다.
2. **`linesToSkip`과 헤더:** CSV 파일의 첫 줄이 헤더 정보라면 `linesToSkip = 1`로 설정하고, `DelimitedLineTokenizer`의 `setNames()`를 사용하여 실제 필드 이름을 제공하는 것이 좋습니다. (또는, 첫 줄을 읽어 동적으로 이름을 설정하는 방법도 있습니다.)
3. **`BeanWrapperFieldSetMapper` 사용 시 필드명/세터명 일치:** `BeanWrapperFieldSetMapper`를 사용할 때는 `FieldSet`의 필드 이름(보통 `LineTokenizer`의 `setNames()`로 설정)과 대상 객체의 자바빈 프로퍼티 이름(세터 메서드 이름에서 `set`을 제외하고 첫 글자를 소문자로 바꾼 것)이 일치해야 자동으로 매핑됩니다.
4. **예외 처리 전략 고려:** `FlatFileParseException` 등이 발생했을 때, 해당 라인을 스킵할지, 잡을 실패시킬지 등을 스텝의 skip/retry 정책을 통해 설정할 수 있습니다.
5. **성능:** 매우 큰 파일을 처리할 때는 `FlatFileItemReader`의 내부 버퍼링이나 기타 설정이 성능에 영향을 미칠 수 있습니다. (일반적으로는 충분히 효율적입니다.)
6. **`ItemStream` 구현:** `FlatFileItemReader`는 `ItemStream` 인터페이스를 구현합니다. 따라서 `open()`, `update()`, `close()` 메서드가 있으며, 이는 파일 리소스를 열고 닫거나, 재시작을 위한 상태(예: 현재까지 읽은 라인 수)를 `ExecutionContext`에 저장/복원하는 데 사용됩니다. 개발자가 직접 이 메서드들을 호출하는 경우는 드물고, 주로 Spring Batch 프레임워크가 스텝 생명주기에 맞춰 호출합니다.

---

### **이전 학습 내용과의 연관성**

- **`FieldSet`:** 이 챕터의 모든 것은 `FieldSet`을 중심으로 돌아갑니다. `LineTokenizer`의 결과물이 `FieldSet`이고, `FieldSetMapper`의 입력이 `FieldSet`입니다.
- **`ItemReader` 인터페이스:** `FlatFileItemReader`는 `ItemReader`의 구체적인 구현체입니다.
- **`ItemStream` 인터페이스:** `FlatFileItemReader`는 `ItemStream`을 구현하여 파일 리소스 관리와 상태 저장/복원을 지원합니다.
- **`Resource` (스프링 코어):** 파일 위치를 지정하는 데 사용됩니다.
- **`ExecutionContext`:** `FlatFileItemReader`가 재시작 시 상태를 복원하고 저장하는 데 사용됩니다.

---
