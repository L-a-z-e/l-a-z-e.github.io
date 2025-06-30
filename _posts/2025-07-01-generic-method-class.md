---
title: Java Generic
description: 
author: laze
date: 2025-07-01 00:00:01 +0900
categories: [Dev, Java]
tags: [Java]
---
## ✅ 제네릭을 선언하는 3가지 주요 위치

| 선언 위치 | 예시 | 설명 |
| --- | --- | --- |
| **클래스/인터페이스 레벨** | `class Box<T> { ... }` | 클래스나 인터페이스 전체에서 T를 타입 파라미터로 사용 |
| **메서드 레벨** | `<T> T methodName(...)` | 해당 메서드 내에서만 T를 타입 파라미터로 사용 |
| **제네릭 메서드 내 매개변수/반환타입** | `public <T> void print(T value)` | 메서드가 받거나 반환하는 타입에 T를 사용 |

---

## 🔹 1. 클래스/인터페이스 레벨에서의 제네릭

```java
public interface ItemReader<I> {
    I read();
}
```

- `ItemReader<I>`: 여기서 `I`는 **타입 파라미터**, 즉 타입을 나중에 지정할 수 있게 한 **템플릿**입니다.
- `I read();` 메서드 안에서 `I`를 타입으로 사용합니다.
- 예: `ItemReader<String>`으로 구현하면 `String read();`가 되는 거죠.

### 장점

- 한 번 타입을 지정하면, 인터페이스 전체에서 일관되게 사용할 수 있음
- 다양한 타입에 재사용 가능

---

## 🔹 2. 메서드 레벨에서의 제네릭

```java
public class MyUtil {
    public static <T> T identity(T input) {
        return input;
    }
}
```

- `<T>`는 **이 메서드 안에서만 유효한 제네릭 타입**입니다.
- `T identity(T input)`는 호출 시 전달한 인자의 타입을 자동 추론해서 반환합니다.

### 사용 예:

```java
String result = MyUtil.identity("hello"); // T는 String
Integer result = MyUtil.identity(123);     // T는 Integer
```

---

## 🔹 3. 차이점 요약

| 구분 | 선언 위치 | 의미 | 특징 |
| --- | --- | --- | --- |
| `class Box<T>` | 클래스/인터페이스 | Box 전체에서 T 사용 | T는 필드, 메서드에 모두 사용 가능 |
| `<T> T method()` | 메서드 내부 | 이 메서드에서만 T 사용 | 독립적으로 타입 추론 가능 |
| `class Box<T> { <U> U method(U u); }` | 둘 다 사용 | 클래스는 T, 메서드는 U | 둘은 서로 독립된 타입 파라미터 |

---

## 🔹 응용 예제

```java
public interface Processor<I, O> {
    O process(I input);
}
```

- 입력 타입과 출력 타입을 각각 제네릭으로 지정할 수 있음
- 구현 시 `Processor<String, Integer>`처럼 명시 가능

```java
public class StringLengthProcessor implements Processor<String, Integer> {
    public Integer process(String input) {
        return input.length();
    }
}
```

---

## 🔸 한 단계 심화: 와일드카드(`?`)와 바운디드 타입

- `<?>`: 어떤 타입이든 허용
- `<? extends Number>`: Number의 하위 타입만 가능
- `<T extends Comparable<T>>`: 특정 타입만 가능하게 제한

---

## 🔚 정리 (비유적으로)

| 용도 | 비유 | 예시 |
| --- | --- | --- |
| `class Box<T>` | 상자에 들어갈 물건 종류를 나중에 정함 | `Box<Apple>` 또는 `Box<Banana>` |
| `<T> T method()` | 메서드가 일시적으로 사용할 변수 타입을 나중에 정함 | `T`는 그때그때 타입 추론됨 |

---

## ✅ 예제 코드 (복잡한 제네릭 예시)

```java
public class Transformer<T, U extends Number, V> {

    private final T input;

    public Transformer(T input) {
        this.input = input;
    }

    public T getInput() {
        return input;
    }

    // 메서드 자체 제네릭: 이 메서드 내에서만 W 사용 가능
    public <W> void printCombined(W extra, V value) {
        System.out.println("T: " + input);
        System.out.println("V: " + value);
        System.out.println("W: " + extra);
    }

    // static 제네릭 메서드: 클래스의 제네릭과 독립적
    public static <X, Y extends Comparable<Y>> X merge(X first, Y second) {
        System.out.println("Merging: " + first + " with " + second);
        return first;
    }
}
```

---

## 🧠 각 위치별 제네릭 설명

| 위치 | 제네릭 | 설명 |
| --- | --- | --- |
| 클래스 레벨 | `<T, U extends Number, V>` | 클래스 전반에서 사용할 타입 파라미터. `U`는 `Number`의 하위 타입만 가능 |
| 인스턴스 메서드 | `<W> void printCombined(W extra, V value)` | 이 메서드 내에서만 `W`를 타입 파라미터로 사용 |
| static 메서드 | `<X, Y extends Comparable<Y>> X merge(...)` | 클래스와 독립적으로 타입을 추론하는 메서드 전용 제네릭 |

---

## 📦 사용 예시

```java
Transformer<String, Integer, Double> transformer = new Transformer<>("hello");

transformer.printCombined(true, 3.14); // T=String, V=Double, W=Boolean

String result = Transformer.merge("data", 42); // X=String, Y=Integer
```

---

## 🎯 동작 흐름 (설명)

1. `Transformer<String, Integer, Double>`
  - `T = String` → `input`의 타입
  - `U = Integer` → `Number`의 하위 클래스만 가능
  - `V = Double` → `printCombined()` 같은 데서 사용됨
2. `transformer.printCombined(true, 3.14)`
  - `W = Boolean` (메서드에서 자체 정의한 타입)
  - `V = Double` (클래스에서 지정한 타입)
  - `T = String` (클래스 필드에서 사용됨)
3. `Transformer.merge("data", 42)`
  - `X = String`
  - `Y = Integer` → `Comparable`을 구현하므로 OK
  - 클래스와는 독립적으로 타입이 추론됨

---

## 🔍 핵심 포인트 정리

| 개념 | 설명 |
| --- | --- |
| **클래스 제네릭** | 객체 생성 시 타입 결정됨. 필드, 일반 메서드, 생성자에 사용 가능 |
| **메서드 제네릭** | 메서드 호출 시 타입이 결정됨. 클래스 제네릭과 독립적일 수 있음 |
| **제약 조건 (`extends`)** | 타입에 제한을 걸 수 있음. (`U extends Number`, `Y extends Comparable<Y>`) |
| **타입 추론** | 대부분의 경우 컴파일러가 파라미터를 보고 타입을 추론해줌 |

---

## 🔄 확장: 와일드카드 버전도 예시로 가능

```java
public void acceptList(List<? extends Number> list) {
    for (Number n : list) {
        System.out.println(n);
    }
}
```

---

## 📚 요약

- **클래스에 쓰는 제네릭**은 상태(state)와 관련 (필드나 생성자)
- **메서드에 쓰는 제네릭**은 동작(behavior)과 관련 (한 번성)
- **static 메서드**는 클래스 제네릭과 무관하므로 반드시 **자체 제네릭 선언** 필요
- **extends** 제약을 통해 **허용 타입 범위 제어** 가능
- **타입 추론**으로 메서드 호출 시 자동 타입 결정 가능

---

## 🔍 질문

- 클래스 제네릭(T, U, V)은 **왜** 필요한가?
- 어차피 메서드에서 `<W>`, `<X, Y>` 이런 새로운 제네릭을 **다시 선언**할 수 있는데, **왜 클래스에서 제네릭을 선언해?**
- 클래스 제네릭과 메서드 제네릭은 **어떻게 다르고**, **언제 써야 하는가?**

---

## ✅ 답변 요지

### 🔸 `W`는 **메서드에서만 임시로 쓰는 타입**이에요.

```java
public <W> void printCombined(W extra, V value)
```

- `<W>` ← 이건 **이 메서드를 호출할 때만 타입이 결정되는** 임시 타입이에요.
- `W`는 **클래스의 상태나 다른 메서드에는 절대 영향을 못 줍니다.**

---

### 🔸 그에 반해, 클래스 제네릭은 클래스 전체의 **"성격"** 을 결정해요.

```java
public class Transformer<T, U extends Number, V>

```

- `T`, `U`, `V`는 이 **클래스를 사용하는 동안 고정**됩니다.
- 클래스의 **필드**, **생성자**, **여러 메서드**에 걸쳐서 **공통된 타입**을 사용하게 해줍니다.

```java
Transformer<String, Integer, Double> transformer = new Transformer<>("hello");

```

- 여기서 `T = String`, `U = Integer`, `V = Double`로 **고정**됩니다.
- `input`은 String이고, 어떤 메서드든 `T`, `V`, `U` 쓰면 전부 이 타입들로 적용됩니다.

---

### 📌 다시 정리하자면...

| 구분 | 제네릭 선언 위치 | 타입 결정 시점 | 적용 범위 | 예시 |
| --- | --- | --- | --- | --- |
| 클래스 제네릭 `<T, U, V>` | 클래스/인터페이스 선언부 | 객체 생성 시 | 필드, 생성자, 인스턴스 메서드 | `Transformer<String, Integer, Double>` |
| 메서드 제네릭 `<W>` | 메서드 선언부 | 메서드 호출 시 | 해당 메서드 내부 | `printCombined(true, 3.14)` → `W = Boolean` |
| static 메서드 제네릭 `<X, Y>` | static 메서드 선언부 | 메서드 호출 시 | 해당 static 메서드 내부 | `merge("data", 42)` → `X = String`, `Y = Integer` |

---

## 🔧 클래스 제네릭이 없으면?

클래스 제네릭 없이 `Object`로 만들면 이렇게 됩니다:

```java
public class Transformer {
    private final Object input;

    public Transformer(Object input) {
        this.input = input;
    }

    public Object getInput() {
        return input;
    }
}

```

- `String`, `Integer`, `Double` 아무거나 넣을 수 있어요.
- 하지만 꺼낼 때마다 **캐스팅 해야 하고**, **타입 안정성도 깨집니다**.

### 제네릭으로 바꾸면?

```java
Transformer<String, Integer, Double> t = new Transformer<>("abc");
String s = t.getInput(); // 캐스팅 없음!

```

✅ 타입 안정성 확보

✅ IDE의 자동완성, 리팩토링, 타입 추론 가능

✅ 코드가 훨씬 명확해짐

---

## 💡 비유로 설명

- 클래스 제네릭 `<T, U, V>`는 **생산 설계도**처럼 미리 정해놓는 규칙이에요.
- 메서드 제네릭 `<W>`는 **잠깐 빌려 쓰는 재료**예요.
- 클래스 설계도를 만들 때 어떤 재료로 만들지를 정해야 전체 구조가 튼튼해져요.

---

## 🔚 결론

- 클래스 제네릭은 **클래스를 쓸 때 타입을 고정시켜 전체 동작의 기반을 정하는 것**
- 메서드 제네릭은 **그때그때 필요한 임시 타입을 추가로 쓸 수 있게 해주는 유연함**
- 둘은 역할이 다르며, **중복되거나 불필요한 것이 아니라, 목적이 다른 도구**입니다.

---

## ✅ 전제 상황 복습

```java
public class Transformer<T, U extends Number, V> {
    public <X> void printCombined(X extra, V value) {
        System.out.println("X: " + extra);
        System.out.println("V: " + value);
    }
}
```

```java
Transformer<String, Integer, Double> transformer = new Transformer<>("hi");
```

---

## ❓이 상황에서 `transformer.printCombined(...)`를 호출하면?

### 📌 타입 해석:

- `T = String`
- `U = Integer` (Number의 하위 클래스)
- `V = Double`

  👉 따라서 `printCombined` 메서드의 **`V value`는 Double로 고정**!


### ✅ `X`는 메서드 호출 시 **임시로 추론되는 타입**이기 때문에 **아무거나 가능**:

```java
transformer.printCombined(123, 3.14);       // X = Integer, V = Double ✅
transformer.printCombined("hello", 2.71);   // X = String,  V = Double ✅
transformer.printCombined(true, 1.618);     // X = Boolean, V = Double ✅
```

---

## 🚫 이렇게는 불가능

```java
transformer.printCombined("text", "not a double"); // ❌ V가 Double이 아닌 String이므로 컴파일 오류!
```

- `V`는 클래스 생성 시 `Double`로 **이미 고정**됐기 때문에

  `printCombined` 메서드에서 그 **자리에 다른 타입(String 등)** 은 넣을 수 없습니다.


---

## 🎯 정리

| 항목 | 의미 |
| --- | --- |
| `T, U, V` | 클래스 생성 시 타입을 고정함. 이후 전역적으로 사용됨 (필드, 메서드 등) |
| `<X>` | `printCombined()` 메서드 호출 시에만 임시로 타입이 정해짐 |
| `V` | 클래스 생성 시 고정된 타입이라 메서드 호출 시에도 고정되어 있음 |

---

## 🔧 다시 비유하자면

- `Transformer<String, Integer, Double>`이라는 클래스를 만들면:
  - 이 클래스는 "입력은 String이고, 어떤 숫자 처리 타입은 Integer, 그리고 출력이나 조작 결과는 Double로 처리한다"는 약속이에요.
- `printCombined<X>`는 **"이 메서드는 당신이 어떤 타입이든 간단한 보조 정보를 넣어도 되게 열어둘게요"** 같은 유연성입니다.

---

## 🔍 질문 핵심 요약

> static 메서드에서 <X, Y extends Comparable<Y>> 선언
>
>
> 메서드 호출 시 타입이 정해진다
>
> T, U, V와는 상관없이 사용된다
>
> **→ 이 설명에서 틀린 점은?**
>

---

## ✅ 정답: **틀린 점은 거의 없고**, 다만 아래 두 가지를 **명확히 구분**하면 더 좋습니다.

---

### ❶ **static 메서드는 클래스 제네릭(T, U, V)** 와 **절대 연관되지 않는다**

✅ **완전히 독립적입니다.**

```java
public class Transformer<T, U, V> {

    // static 메서드에서는 클래스의 T, U, V 사용 불가!
    public static <X, Y extends Comparable<Y>> X merge(X x, Y y) {
        return x;
    }
}
```

- 이 `merge` 메서드는 `Transformer<T, U, V>`와는 **무관하게** 작동합니다.
- 그래서 반드시 `<X, Y>` 같이 **자체 제네릭 선언**이 필요합니다.
- `T`, `U`, `V`는 인스턴스가 있어야 접근 가능한데, `static`은 클래스 레벨에서 존재하기 때문입니다.

❗ 만약 `static`에서 `T`를 쓰면 컴파일 오류 납니다:

```java
public static T failMethod(T input) { // ❌ 컴파일 오류
    return input;
}
```

---

### ❷ 메서드 호출 시 **X, Y는 호출된 값의 타입에 따라 컴파일러가 추론**함

```java
Transformer.merge("abc", 123); // X = String, Y = Integer
```

- `123`은 `Integer`, 그리고 `Integer`는 `Comparable<Integer>` 구현하므로 OK
- `merge()`는 `String`을 반환

---

## ✅ 완벽 정리

| 개념 | 설명 |
| --- | --- |
| `static <X, Y extends Comparable<Y>> X merge(X x, Y y)` | 클래스와 무관한 제네릭 메서드 |
| `X`, `Y`는 메서드 호출 시 결정 | 클래스에서 어떤 `T`, `U`, `V`를 쓰고 있든 상관 없음 |
| `Y extends Comparable<Y>` | `Y` 타입은 `Comparable`을 반드시 구현해야 함 |
| 반환형 `X` | 첫 번째 인자 `x`의 타입이 그대로 리턴됨 |

---

## 🔚 결론 (틀린 부분 정리)

> "X, Y는 메서드 호출할 때 지정되고, T, U, V랑 관계없는 타입을 파라미터로 사용할 수 있다"
>
>
> ✅ 맞습니다. 다만 더 정확히 말하면:
>
- **"T, U, V와 관계없는 이유는 static 메서드는 클래스 제네릭을 사용할 수 없기 때문"**
- **"그래서 static 메서드는 반드시 자기 제네릭을 `<X, Y>`처럼 선언해야 한다"**

---

## ✅ `static <T> T fromJson(String json, Class<T> clazz)`

---

### 💡 이 메서드는 어떤 상황에서 쓰이냐면?

- JSON 문자열을 어떤 객체로 바꾸고 싶을 때 사용됩니다.
- 라이브러리로는 대표적으로 **Jackson**, **Gson**, **Moshi** 같은 JSON 파서에서 쓰입니다.

---

## ✅ 실전 예제: JSON을 타입에 맞게 객체로 파싱

```java
public class JsonUtil {
    private static final ObjectMapper objectMapper = new ObjectMapper();

    // 제네릭 static 메서드
    public static <T> T fromJson(String json, Class<T> clazz) {
        try {
            return objectMapper.readValue(json, clazz);
        } catch (IOException e) {
            throw new RuntimeException(e);
        }
    }
}
```

---

## 🔍 이 메서드의 핵심 구조

```java
public static <T> T fromJson(String json, Class<T> clazz)
```

| 위치 | 의미 |
| --- | --- |
| `<T>` | 이 static 메서드 안에서만 쓸 **타입 파라미터 T**를 선언 |
| `T fromJson(...)` | 이 메서드는 T 타입 객체를 반환함 |
| `Class<T> clazz` | Java는 타입 정보를 런타임에 유지하지 않기 때문에, **직접 타입 정보를 넘겨야 함** (type erasure 문제 때문에) |

---

### 📌 사용 예

```java
String json = "{\"name\": \"Mun\", \"age\": 29}";

Person person = JsonUtil.fromJson(json, Person.class);
```

- `T = Person` 으로 추론됨
- 반환값은 `Person` 타입 객체
- `Class<Person>` 이 전달되므로, 타입 정보를 유지함

---

## ❓왜 이렇게 복잡하게 `Class<T> clazz`를 넘기나?

Java는 제네릭을 컴파일 시점에만 알고, **런타임에는 지워버리는(type erasure)** 언어이기 때문입니다.

그래서 다음은 안 됩니다:

```java
public static <T> T fromJson(String json) {
    // objectMapper.readValue(json, ???); ← 타입 정보를 알 수 없음 ❌
}
```

그래서 `Class<T> clazz`를 넘겨서 컴파일러에게 이렇게 말해주는 겁니다:

> "이 T가 뭔지 네가 잊었더라도, 내가 직접 알려줄게!"
>

---

## ✅ 한 줄 요약

> static <T> T fromJson(String json, Class<T> clazz) 는
>
>
> **"어떤 타입이든 JSON으로부터 객체로 변환할 수 있게 만들어주는 제네릭 정적 팩토리 메서드"**입니다.
>

---

## 📦 이런 패턴은 어디에 많이 쓰일까?

| 상황 | 메서드 이름 | 설명 |
| --- | --- | --- |
| 객체 생성 | `<T> T of(...)` | 정적 팩토리 메서드 (`Optional.of`, `List.of`) |
| 변환 | `<T> T parse(...)` | JSON/XML/String → 객체 |
| 유틸 함수 | `<T> T copyOf(...)` | 복사, deep clone |

---

## 🔚 지금까지 배운 static 제네릭 핵심 요약

| 특징 | 설명 |
| --- | --- |
| `static`이라 클래스 제네릭 못 씀 | 그래서 `<T>`를 메서드 선언부에서 **새로 선언해야** 함 |
| 타입 추론 가능 | 메서드 파라미터를 보면 컴파일러가 알아서 타입을 추론함 |
| 필요 시 타입 직접 전달 | `Class<T>`를 같이 넘겨야 Java의 타입 소거 문제를 해결 가능 |
| 매우 많이 쓰임 | `Optional.of`, `Collectors.toMap`, `JsonUtil.fromJson`, `Function.identity()` 등 |

---

## ✅ 먼저: 타입 소거(type erasure)란?

Java는 제네릭을 컴파일 타임까지만 사용하고, **런타임에서는 제네릭 타입 정보를 지워버립니다**.

```java
List<String> list1 = new ArrayList<>();
List<Integer> list2 = new ArrayList<>();
```

위의 두 변수는 컴파일 타임에서는 다르게 보이지만, **런타임에서는 둘 다 그냥 `List`**로 취급됩니다.

```java
System.out.println(list1.getClass() == list2.getClass()); // true
```

→ 즉, JVM 입장에서는 `List<String>`도 `List<Integer>`도 그냥 `List`일 뿐이에요.

---

## ❓그럼 왜 타입 정보가 필요한가?

타입 정보를 유지해야 하는 대표적인 이유는 **런타임에서 올바른 타입 객체를 생성하거나 변환해야 할 때**입니다.

### 대표 상황: JSON → 객체 변환

```java
String json = "{\"name\":\"Mun\", \"age\":29}";

// 이걸 Person 객체로 만들고 싶다
```

이런 걸 하려면, 파서(JSON 라이브러리)는 `"Person"`이라는 **타입 정보를 알아야** 내부적으로 `new Person()`을 만들고 필드를 채우겠죠?

하지만:

```java
public static <T> T fromJson(String json) {
    // objectMapper.readValue(json, ???); // 타입 정보 없음 ❌
}
```

여기선 타입 `T`가 런타임에 존재하지 않기 때문에 **ObjectMapper는 T가 뭔지 알 수 없습니다.**

### 해결 방법: 타입을 외부에서 받아서 "강제로" 알려줌

```java
public static <T> T fromJson(String json, Class<T> clazz) {
    return objectMapper.readValue(json, clazz); // 여기서 clazz가 T의 실체!
}
```

---

## 🎯 한 줄 요약

> 제네릭은 런타임에 타입 정보가 사라지기 때문에, 객체 생성/변환과 같은 런타임 동작을 위해선 타입 정보를 "직접 유지하거나 넘겨줘야" 한다.
>

---

## 💡 실제로 타입 정보가 꼭 필요한 경우들

| 상황 | 이유 |
| --- | --- |
| JSON/XML → 객체 | 어떤 객체를 만들어야 하는지 알아야 함 |
| 리플렉션으로 타입 확인 | 클래스의 필드나 메서드의 파라미터 타입 추적 |
| List의 내부 타입 추적 | `List<String>` vs `List<User>` 구분 |
| 제네릭 DAO / Repository 구현 | 어떤 타입의 테이블/엔티티를 다루는지 알아야 함 |

---

## 🔍 조금 더 고급 예시

다음은 이 타입 정보를 유지하지 않으면 아예 불가능한 경우입니다:

```java
<T> List<T> parseList(String json, Class<T> clazz)
```

- 여기서 T의 리스트를 만들고 싶은데, `T`가 뭔지 몰라선 불가능해요.
- 그래서 `clazz`를 써야 `List<T>`를 올바르게 구성할 수 있어요.

---

## ✨ 덤: Kotlin, Scala는 이 문제를 어떻게 해결할까?

- Java는 타입 소거 때문에 불편함이 많지만
- **Kotlin은 `reified` 키워드로 타입 정보를 유지할 수 있게 해줍니다** (인라인 함수로)
- **Scala는 타입 태그(TypeTag)** 등을 이용해서 타입 정보를 런타임에 유지합니다

즉, Java의 타입 소거는 언어 설계상 오래된 결정이고, 후속 언어들은 이 문제를 더 우아하게 해결하고 있어요.

---

## ✅ 결론

| 요약 포인트 |
| --- |
| 제네릭 타입은 컴파일 이후 지워짐 (type erasure) |
| 그래서 런타임에 타입이 필요한 경우, 직접 넘겨줘야 함 |
| `Class<T>` 혹은 `TypeReference<T>` 등을 전달 |
| 이를 통해 안전하고 정확한 객체 생성, 변환 등이 가능해짐 |

---
