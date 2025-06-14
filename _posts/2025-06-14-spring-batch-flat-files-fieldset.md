---
title: Spring Batch - Flat Files(FiledSet)
description: 
author: laze
date: 2025-06-14 00:00:02 +0900
categories: [Dev, SpringBatch]
tags: [SpringBatch]
---
### 필드셋 (The FieldSet)

Spring Batch에서 플랫 파일을 다룰 때, 입력이든 출력이든 관계없이 가장 중요한 클래스 중 하나는 `FieldSet`입니다.

많은 아키텍처와 라이브러리에는 파일에서 읽어오는 데 도움이 되는 추상화가 포함되어 있지만,

대개 `String` 또는 `String` 객체의 배열을 반환합니다.

이것은 실제로 절반만 해결해 줄 뿐입니다.

`FieldSet`은 파일 리소스의 필드 바인딩을 가능하게 하는 Spring Batch의 추상화입니다.

이를 통해 개발자는 데이터베이스 입력을 다루는 것과 거의 동일한 방식으로 파일 입력을 다룰 수 있습니다.

`FieldSet`은 개념적으로 JDBC `ResultSet`과 유사합니다.

`FieldSet`은 토큰(token)의 `String` 배열이라는 단 하나의 인수만 필요로 합니다.

선택적으로, 다음 예제와 같이 `ResultSet`을 본떠 필드 인덱스 또는 이름으로 필드에 접근할 수 있도록 필드 이름을 구성할 수도 있습니다:

```java
String[] tokens = new String[]{"foo", "1", "true"};
FieldSet fs = new DefaultFieldSet(tokens);
String name = fs.readString(0);
int value = fs.readInt(1);
boolean booleanValue = fs.readBoolean(2);
```

`FieldSet` 인터페이스에는 `Date`, `long`, `BigDecimal` 등과 같은 훨씬 더 많은 옵션이 있습니다.

`FieldSet`의 가장 큰 장점은 플랫 파일 입력의 일관된 파싱을 제공한다는 것입니다.

각 배치 잡(job)이 잠재적으로 예상치 못한 방식으로 다르게 파싱하는 대신, 형식 예외로 인한 오류를 처리할 때나 간단한 데이터 변환을 수행할 때 모두 일관성을 유지할 수 있습니다.

---

### **학습 목표 제시**

이번 "The FieldSet" 챕터를 통해 학생분은 다음을 이해하고 배울 수 있습니다:

1. **`FieldSet`의 개념과 필요성 이해:** 플랫 파일에서 읽어온 문자열 데이터를 단순히 배열로 받는 것의 한계를 인지하고, `FieldSet`이 어떻게 타입 변환과 일관된 데이터 접근을 제공하여 이 문제를 해결하는지 이해합니다.
2. **`FieldSet`의 기본 사용법 숙지:** `FieldSet` 객체를 생성하고, 인덱스나 필드 이름을 사용하여 특정 타입(String, int, boolean, Date 등)으로 데이터를 읽어오는 방법을 익힙니다.
3. **`FieldSet` 사용의 이점 파악:** `FieldSet`을 사용함으로써 얻을 수 있는 일관된 파싱, 오류 처리, 데이터 변환의 이점을 이해합니다.

---

### **핵심 개념 설명**

이 챕터의 핵심은 **"플랫 파일에서 읽어온 원시(raw) 문자열 데이터를 어떻게 하면 사용하기 쉽고 타입-안전(type-safe)한 방식으로 다룰 수 있을까?"** 이며, `FieldSet`이 그 해답을 제공합니다.

### 1. `FieldSet`이란 무엇인가?

- **개념:** 플랫 파일의 한 줄(레코드)에서 추출된 필드(데이터 조각)들의 묶음을 나타내는 객체입니다. 이 객체를 통해 각 필드에 인덱스나 이름으로 접근하고, 원하는 데이터 타입(문자열, 숫자, 날짜 등)으로 편리하게 변환하여 가져올 수 있습니다.
- **비유:**
  - **잘 정리된 명함첩의 명함 한 장:** 명함(한 줄의 데이터)에 이름, 전화번호, 이메일(각 필드) 정보가 적혀있고, "이름 주세요", "전화번호 알려주세요"라고 요청하면 해당 정보를 꺼내주는 것과 같습니다. 이때 단순히 글자 뭉치가 아니라, 전화번호는 숫자 형식으로, 이메일은 이메일 형식으로 이해하고 줄 수 있습니다.
  - **데이터베이스 `ResultSet`의 한 행(Row):** JDBC에서 SQL 쿼리 결과를 `ResultSet`으로 받아 `rs.getString("columnName")`, `rs.getInt("columnIndex")` 등으로 특정 컬럼의 값을 가져오는 것과 매우 유사합니다. `FieldSet`은 파일 데이터를 위한 `ResultSet`이라고 생각할 수 있습니다.
- **필요성:**
  - **단순 문자열 배열의 한계:** 플랫 파일에서 한 줄을 읽으면 보통 쉼표(,) 등으로 구분된 문자열 배열(`String[]`)을 얻게 됩니다. 하지만 이 배열만으로는 다음과 같은 불편함이 있습니다.
    - 모든 데이터가 문자열 형태이므로, 숫자나 날짜로 사용하려면 매번 수동으로 변환해야 합니다. (예: `Integer.parseInt(tokens[1])`)
    - 변환 과정에서 오류(예: 숫자가 와야 할 자리에 문자열이 옴)가 발생하기 쉽고, 이에 대한 예외 처리도 번거롭습니다.
    - 데이터에 접근할 때 항상 인덱스를 사용해야 하므로 가독성이 떨어지고, 파일 형식이 변경되면 인덱스를 수정해야 하는 유지보수의 어려움이 있습니다.
  - **`FieldSet`의 해결책:**
    - **타입 변환 제공:** `readInt()`, `readDate()`, `readBoolean()` 등 다양한 타입으로 데이터를 직접 읽을 수 있는 메서드를 제공하여 수동 변환의 번거로움을 줄여줍니다.
    - **일관된 파싱 및 오류 처리:** 내부적으로 표준화된 파싱 로직을 사용하여 데이터 변환 오류를 일관되게 처리하고, 필요한 경우 예외를 발생시킵니다.
    - **이름 기반 접근 (선택 사항):** 필드에 이름을 부여하면 인덱스 대신 이름으로 데이터에 접근할 수 있어 코드의 가독성과 유지보수성을 높일 수 있습니다. (예: `fs.readString("lastName")`)
- **"왜 필요한가?"**:
  - 플랫 파일 데이터를 마치 구조화된 데이터(데이터베이스 레코드처럼)처럼 다룰 수 있게 하여 개발 편의성을 크게 향상시킵니다.
  - 데이터 파싱 및 변환 로직을 `FieldSet`에 위임함으로써, 개발자는 비즈니스 로직에 더 집중할 수 있습니다.
  - 여러 배치 잡에서 파일 데이터를 일관된 방식으로 처리하도록 표준을 제공합니다.

### 2. `FieldSet`의 기본 사용법

문서의 예제를 통해 살펴보겠습니다.

```java
// 가정: 플랫 파일의 한 줄에서 "foo,1,true" 라는 데이터를 읽어서
// 쉼표(,)를 기준으로 나눈 결과가 tokens 배열에 담겼다고 가정합니다.
String[] tokens = new String[]{"foo", "1", "true"};

// 1. FieldSet 객체 생성 (보통 DefaultFieldSet 사용)
//    - 생성자의 첫 번째 인자로 String 배열 (토큰들)을 전달합니다.
FieldSet fs = new DefaultFieldSet(tokens);

// 2. 인덱스를 사용하여 데이터 읽기 (0부터 시작)
String name = fs.readString(0);       // tokens[0] ("foo")을 문자열로 읽음
int value = fs.readInt(1);          // tokens[1] ("1")을 정수(int)로 변환하여 읽음
boolean booleanValue = fs.readBoolean(2); // tokens[2] ("true")를 불리언(boolean)으로 변환하여 읽음

System.out.println("Name: " + name);             // 출력: Name: foo
System.out.println("Value: " + value);           // 출력: Value: 1
System.out.println("Boolean Value: " + booleanValue); // 출력: Boolean Value: true

```

- **`DefaultFieldSet`**: `FieldSet` 인터페이스의 기본 구현체입니다.
- **생성자**:
  - `new DefaultFieldSet(String[] tokens)`: 토큰 배열만 전달하여 생성. 인덱스로만 접근 가능.
  - `new DefaultFieldSet(String[] tokens, String[] names)`: 토큰 배열과 함께 각 토큰에 해당하는 필드 이름 배열을 전달하여 생성. 인덱스 또는 이름으로 접근 가능.

**필드 이름으로 접근하는 예시 (생성 시 필드 이름 지정):**

```java
String[] tokens = new String[]{"bar", "42", "false", "2023-01-15"};
String[] names = new String[]{"itemName", "quantity", "isUrgent", "orderDate"}; // 필드 이름 정의

FieldSet fsWithNames = new DefaultFieldSet(tokens, names);

// 이름으로 데이터 읽기
String itemName = fsWithNames.readString("itemName");        // "bar"
int quantity = fsWithNames.readInt("quantity");           // 42
boolean isUrgent = fsWithNames.readBoolean("isUrgent");     // false
java.util.Date orderDate = fsWithNames.readDate("orderDate", "yyyy-MM-dd"); // "2023-01-15", 날짜 형식 지정

System.out.println("Item Name: " + itemName);
System.out.println("Order Date: " + orderDate);

```

- 필드 이름을 사용하면 코드의 가독성이 훨씬 좋아지고, 파일의 필드 순서가 변경되더라도 이름만 같다면 코드를 수정할 필요가 없어 유지보수에 유리합니다.
- `readDate(String name, String pattern)`처럼 날짜 형식을 지정하여 파싱할 수도 있습니다.

### 3. `FieldSet` 인터페이스의 다양한 `readXXX()` 메서드

`FieldSet`은 다양한 데이터 타입으로 값을 읽을 수 있도록 여러 `readXXX()` 메서드를 제공합니다.

- `readString(int index)` / `readString(String name)`
- `readChar(int index)` / `readChar(String name)`
- `readBoolean(int index)` / `readBoolean(String name)`
  - `readBoolean(int index, String trueValue)` / `readBoolean(String name, String trueValue)`: 특정 문자열(예: "Y", "T")을 `true`로 인식하도록 지정 가능.
- `readByte(int index)` / `readByte(String name)`
- `readShort(int index)` / `readShort(String name)`
- `readInt(int index)` / `readInt(String name)`
  - `readInt(int index, int defaultValue)` / `readInt(String name, int defaultValue)`: 파싱 실패 시 사용할 기본값 지정 가능.
- `readLong(int index)` / `readLong(String name)`
- `readFloat(int index)` / `readFloat(String name)`
- `readDouble(int index)` / `readDouble(String name)`
- `readBigDecimal(int index)` / `readBigDecimal(String name)`
- `readDate(int index)` / `readDate(String name)` (기본 날짜 형식 사용)
- `readDate(int index, String pattern)` / `readDate(String name, String pattern)` (사용자 정의 날짜 형식 사용)
- `getProperties()`: `FieldSet`의 모든 필드와 값을 `java.util.Properties` 객체로 반환.
- `getFieldCount()`: 필드의 총 개수 반환.
- `hasNames()`: 필드 이름이 설정되어 있는지 여부 반환.
- `getNames()`: 설정된 필드 이름 배열 반환.
- `getValues()`: 원본 토큰(문자열) 배열 반환.

### 4. `FieldSet` 사용의 이점

- **일관된 파싱 (Consistent Parsing):**
  - 가장 큰 장점입니다. `FieldSet`은 숫자, 날짜, 불리언 등의 타입을 파싱하는 표준화된 방법을 제공합니다.
  - 만약 `FieldSet` 없이 각 배치 잡마다 개발자가 직접 `Integer.parseInt()`, `SimpleDateFormat.parse()` 등을 사용한다면, 파싱 로직이 중복되고, 각기 다른 방식으로 오류를 처리하거나 미묘한 파싱 규칙 차이로 인해 예기치 않은 결과가 발생할 수 있습니다.
  - `FieldSet`을 사용하면 이러한 파싱 로직이 한 곳(Spring Batch 프레임워크 내)에 집중되어 일관성을 보장합니다.
- **데이터 변환 용이성:** 다양한 `readXXX()` 메서드를 통해 원하는 타입으로 쉽게 변환할 수 있습니다.
- **오류 처리:** 데이터 형식 변환 중 오류가 발생하면 `FieldSet`은 적절한 예외(예: `NumberFormatException`을 내부적으로 처리하고, 더 구체적인 배치 관련 예외로 래핑하거나 정보를 제공할 수 있음)를 던져 일관된 오류 처리를 돕습니다.
- **가독성 및 유지보수성 향상 (이름 사용 시):** 필드 이름으로 데이터에 접근하면 코드가 훨씬 이해하기 쉬워지고, 파일 구조 변경에도 유연하게 대처할 수 있습니다.

---

### **주요 용어 해설**

- **토큰 (Token):** 플랫 파일의 한 줄에서 구분자(delimiter)에 의해 분리된 개별 문자열 조각. 예를 들어 "apple,100,red"라는 줄에서 쉼표로 구분하면 "apple", "100", "red"가 각각 토큰이 됩니다.
- **필드 (Field):** 토큰이 나타내는 의미있는 데이터 단위. `FieldSet`은 이 토큰들을 필드로 취급하고 접근할 수 있게 합니다.
- **파싱 (Parsing):** 특정 형식의 문자열 데이터를 분석하여 의미있는 구조로 변환하거나, 다른 데이터 타입으로 바꾸는 과정. (예: 문자열 "123"을 숫자 123으로 파싱)
- **바인딩 (Binding):** 파일에서 읽어온 데이터(필드)를 프로그램 내의 변수나 객체의 속성에 연결(할당)하는 과정. `FieldSet`은 이 바인딩 과정을 용이하게 합니다.

---

### **코드 예제 분석 (문서 예제 다시 보기)**

```java
String[] tokens = new String[]{"foo", "1", "true"}; // 원시 데이터 (문자열 배열)
FieldSet fs = new DefaultFieldSet(tokens);         // FieldSet으로 래핑

// FieldSet을 통해 타입에 맞게 데이터를 읽어옴
String name = fs.readString(0); // "foo" (String)
int value = fs.readInt(1);      // 1 (int)
boolean booleanValue = fs.readBoolean(2); // true (boolean)

```

이 간단한 예제가 `FieldSet`의 핵심 가치를 잘 보여줍니다. `tokens[1]`은 문자열 "1"이지만, `fs.readInt(1)`을 통해 우리는 별도의 변환 코드 없이 바로 `int` 타입의 값 1을 얻을 수 있습니다. 마찬가지로 `tokens[2]` ("true")도 `fs.readBoolean(2)`를 통해 `boolean` 값 `true`로 변환됩니다.

만약 `tokens[1]`이 "abc"처럼 숫자로 변환될 수 없는 문자열이었다면, `fs.readInt(1)` 호출 시 내부적으로 `NumberFormatException`과 같은 예외가 발생하고, 이는 `FieldSet`이나 이를 사용하는 `ItemReader`에 의해 적절히 처리될 것입니다 (예: 스킵 정책에 따라 해당 레코드 건너뛰기, 또는 잡 실패).

---

### **"왜?" 라는 질문에 대한 답변**

- **왜 단순 `String[]` 대신 `FieldSet`을 사용할까?**
  - **타입 안전성:** `String[]`은 모든 것을 문자열로만 다루지만, `FieldSet`은 각 필드를 의도한 데이터 타입(정수, 날짜, 불리언 등)으로 변환해주어 타입 오류를 줄이고 코드의 안정성을 높입니다.
  - **편의성:** `Integer.parseInt()`, `SimpleDateFormat.parse()` 같은 변환 코드를 반복적으로 작성할 필요가 없습니다. `FieldSet`이 알아서 해줍니다.
  - **일관성:** 여러 개발자가 각자 다른 방식으로 데이터를 파싱하고 변환하는 것을 방지하고, Spring Batch가 제공하는 표준화된 방식을 따르도록 유도하여 전체 시스템의 일관성을 유지합니다.
  - **가독성 (이름 사용 시):** `tokens[5]`보다는 `fs.getString("emailAddress")`가 훨씬 이해하기 쉽습니다.
- **`FieldSet`은 JDBC `ResultSet`과 어떻게 유사한가?**
  - **데이터 접근 방식:** 둘 다 특정 "행(row)"에 해당하는 데이터 묶음에서 개별 "열(column)" 또는 "필드(field)"의 값을 인덱스나 이름으로 가져올 수 있는 메서드(`getString`, `getInt` 등)를 제공합니다.
  - **타입 변환:** 둘 다 원시 데이터(DB의 경우 다양한 SQL 타입, 파일의 경우 문자열)를 자바의 특정 데이터 타입으로 변환해주는 기능을 내장하고 있습니다.
  - **추상화 수준:** `ResultSet`이 데이터베이스 접근의 복잡성을 숨기고 일관된 데이터 접근 인터페이스를 제공하듯이, `FieldSet`은 플랫 파일 데이터 접근의 복잡성(파싱, 변환 등)을 숨기고 유사한 경험을 제공합니다.

---

### **주의사항 및 Best Practice**

1. **필드 이름 사용 권장:** 가능하다면 `FieldSet` 생성 시 필드 이름을 함께 제공하여 이름 기반으로 데이터에 접근하는 것이 좋습니다. 이는 코드의 가독성과 유지보수성을 크게 향상시킵니다.
2. **날짜 형식 주의:** `readDate()` 사용 시, 입력 데이터의 날짜 형식과 `FieldSet`에 지정한 날짜 패턴(또는 기본 패턴)이 일치하는지 주의해야 합니다. 불일치 시 `ParseException`이 발생할 수 있습니다.
3. **기본값(Default Value) 활용:** 숫자 등을 읽을 때, 해당 필드가 비어있거나 잘못된 형식일 경우를 대비하여 `readInt(name, defaultValue)`처럼 기본값을 제공하는 메서드를 활용하면 예외 발생을 줄이고 보다 안정적으로 데이터를 처리할 수 있습니다. (물론, 엄격한 유효성 검사가 필요하다면 예외를 발생시키는 것이 맞습니다.)
4. **`FieldSet`은 어디서 만들어질까?:** 보통 `FlatFileItemReader`와 같은 파일 기반 `ItemReader` 내부에서, 파일의 각 줄을 읽고 파싱(토큰화)한 후에 `FieldSet` 객체를 생성하여 반환합니다. 개발자가 직접 `FieldSet`을 생성하는 경우는 테스트 코드나 특정 유틸리티를 만들 때 외에는 흔치 않습니다.

---

### **이전 학습 내용과의 연관성**

- **`ItemReader` (특히 `FlatFileItemReader`):** `FieldSet`은 주로 `FlatFileItemReader`와 함께 사용됩니다. `FlatFileItemReader`는 파일에서 한 줄씩 읽어, 그 줄을 토큰으로 분리한 후 `FieldSet`으로 만들어 `read()` 메서드의 반환값으로 사용하거나, 이 `FieldSet`을 `FieldSetMapper`에게 전달하여 도메인 객체로 변환합니다.
- **데이터 변환 및 파싱:** `FieldSet`은 Spring Batch 내에서 데이터 변환과 파싱의 일관성을 담당하는 중요한 역할을 합니다.

---
