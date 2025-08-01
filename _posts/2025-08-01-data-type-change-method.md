---
title: `valueOf` vs `parseInt` vs `toString` - 헷갈리는 자료형 변환 메서드 완벽 정리
description: 
author: laze
date: 2025-08-01 00:00:01 +0900
categories: [Dev, Java]
tags: [Java, Type Casting, parseInt, valueOf, toString]
---
# [Java] `valueOf` vs `parseInt` vs `toString`: 헷갈리는 자료형 변환 메서드 완벽 정리

## 1. 서론: 매번 헷갈리는 자료형 변환

"문자열을 숫자로 바꿀 때 `parseInt`를 써야 하나, `valueOf`를 써야 하나?", "객체를 문자열로 바꿀 때 `toString`과 `String.valueOf` 중 뭐가 더 안전할까?"

Java 개발자라면 누구나 한 번쯤 해봤을 고민입니다. 이 혼란의 근본적인 원인은 Java의 **'원시 타입(Primitive Type)'**과 **'래퍼 객체(Wrapper Object)'**라는 두 가지 데이터 표현 방식의 미묘한 차이를 제대로 이해하지 못했기 때문인 경우가 많습니다.

이 글에서는 자료형 변환의 '방향'과 '결과물'을 기준으로, 각 메서드의 역할을 명확히 구분하고 어떤 상황에서 무엇을 사용해야 하는지에 대한 명확한 선택 기준을 제시합니다.

## 2. 기초 다지기: 원시 타입(Primitive) vs 래퍼 객체(Wrapper)

자료형 변환을 논하기 전, 먼저 Java의 두 가지 숫자 표현 방식을 명확히 구분해야 합니다.

| 구분 | **원시 타입 (Primitive Type)** | **래퍼 객체 (Wrapper Object)** |
| --- | --- | --- |
| **예시** | `int`, `double`, `long`, `boolean` | `Integer`, `Double`, `Long`, `Boolean` |
| **본질** | 순수한 '값' 자체 | 값을 감싸고 있는 '객체' |
| **`null` 저장** | **불가능** (e.g., `int i = null;` 컴파일 에러) | **가능** (e.g., `Integer i = null;`) |
| **메서드 소유** | 없음 (e.g., `i.toString()` 불가) | 있음 (e.g., `i.intValue()`, `i.compareTo()`) |
| **메모리** | 스택(Stack) 영역에 주로 저장 (가벼움) | 힙(Heap) 영역에 저장 (상대적으로 무거움) |

Java 5부터 도입된 **오토박싱(Autoboxing: `int` → `Integer`)**과 **언박싱(Unboxing: `Integer` → `int`)** 기능 때문에 이 둘의 경계가 모호하게 느껴질 수 있지만, 우리가 다룰 변환 메서드들은 바로 이 둘 사이를 명시적으로 오가는 역할을 합니다.

## 3. Part 1: 문자열 → 숫자 변환 (`parseInt` vs `valueOf`)

문자열을 숫자로 바꿀 때 가장 먼저 마주치는 두 메서드입니다. 선택의 기준은 **"내가 원하는 결과물이 순수한 값인가, 아니면 객체인가?"** 입니다.

### `Integer.parseInt(String s)`

- **역할:** "문자열을 **파싱(Parse)**해서 **원시 타입 `int`*를 반환한다."
- **반환 타입:** `int` (순수한 값)
- **언제 쓰는가?:** 대부분의 경우. 단순히 숫자 값이 필요하고, 그 값으로 산술 연산을 해야 할 때 가장 빠르고 직접적인 방법입니다. 객체를 생성하는 추가적인 오버헤드가 없습니다.
- **`Double.parseDouble`, `Long.parseLong`** 등 다른 숫자 타입도 동일한 규칙이 적용됩니다.

```java
String strNumber = "123";

// "123" 문자열을 파싱하여 원시 타입 int 123을 얻는다.
int primitiveInt = Integer.parseInt(strNumber);

System.out.println(primitiveInt + 7); // 130 (산술 연산 가능)
```

### `Integer.valueOf(String s)`

- **역할:** "문자열에 해당하는 **값(Value)을 가진 `Integer` 래퍼 객체**를 반환한다."
- **반환 타입:** `Integer` (래퍼 객체)
- **언제 쓰는가?:** `Integer` 객체 자체가 필요할 때. 예를 들어 컬렉션(`List<Integer>`)에 넣거나, 제네릭 타입으로 사용하거나, 메서드 반환 값으로 `null`을 표현해야 할 때 사용합니다.
- **특징:** `128`에서 `127` 사이의 값에 대해서는 새로운 객체를 만들지 않고, 미리 만들어 둔 **캐시된 객체**를 반환하여 효율을 높입니다.

```java
String strNumber = "123";

// "123" 문자열에 해당하는 값을 가진 Integer 객체를 얻는다.
Integer wrapperInt = Integer.valueOf(strNumber);

List<Integer> numberList = new ArrayList<>();
numberList.add(wrapperInt); // 컬렉션에는 객체만 담을 수 있다.
```

## 4. Part 2: 객체/원시 타입 → 문자열 변환 (`toString` vs `String.valueOf`)

어떤 값을 문자열로 바꿀 때, 선택의 기준은 **"Null을 어떻게 다룰 것인가?"** 입니다.

### `object.toString()`

- **역할:** "객체를 사람이 읽을 수 있는 문자열로 표현한다."
- **특징:** `Object` 클래스에 정의된 메서드이므로, 모든 Java 객체는 이 메서드를 가지고 있습니다. 각 클래스는 자신의 상태를 잘 나타내도록 이 메서드를 **오버라이드(Override)**하는 것이 권장됩니다.
- **치명적인 함정:** **`null` 참조에 대해 호출하면 `NullPointerException`이 발생합니다.**

```java
User user = new User("laze");
System.out.println(user.toString()); // User 클래스에 오버라이드된 내용 출력

User nullUser = null;
// System.out.println(nullUser.toString()); // CRASH! -> NullPointerException
```

### `String.valueOf(Object obj)`

- **역할:** "어떤 타입의 값이든 **안전하게** 문자열로 변환한다."
- **핵심 장점:** **Null-Safe.** 파라미터로 `null`이 들어오면 `NullPointerException`을 던지는 대신, **문자열 `"null"`을 반환**합니다.
- **내부 동작:** `String.valueOf(obj)`는 내부적으로 `obj == null ? "null" : obj.toString()` 와 유사하게 동작합니다.

```java
User user = new User("laze");
System.out.println(String.valueOf(user)); // user.toString() 호출 결과와 동일

User nullUser = null;
System.out.println(String.valueOf(nullUser)); // "null" 문자열 출력 (안전!)

int primitiveInt = 100;
System.out.println(String.valueOf(primitiveInt)); // "100"
```

**Best Practice:** `null`일 가능성이 있는 모든 변수를 문자열로 변환할 때는 `String.valueOf()`를 사용하는 것이 가장 안전하고 바람직합니다.

## 5. Part 3: 래퍼 객체 → 원시 타입 변환 (`...Value()`)

`Integer`와 같은 래퍼 객체에서 순수한 원시 타입 값을 꺼내고 싶을 때 사용합니다.

- **역할:** "래퍼 객체가 감싸고 있는(wrapped) **원시 타입 값(Value)**을 추출한다."

```java
Integer integerObject = Integer.valueOf("200");

// 명시적 언박싱 (Unboxing)
int primitiveInt = integerObject.intValue();
double primitiveDouble = integerObject.doubleValue(); // 다른 숫자 타입으로도 변환 가능

System.out.println(primitiveInt); // 200
```

현대 Java에서는 언박싱이 자동으로 이루어지므로(`int i = integerObject;`), `intValue()`를 명시적으로 호출할 일은 적습니다. 하지만 `null`인 `Integer` 객체를 `int` 변수에 대입하면 `NullPointerException`이 발생할 수 있으므로, 그 내부 동작을 이해하는 것은 중요합니다.

## 6. 한눈에 보는 변환 가이드 (치트 시트)

| 변환 방향 | 목적 | 추천 메서드 | 예시 코드 | 반환 타입 |
| --- | --- | --- | --- | --- |
| **String → 숫자** | 원시 타입 값 필요 | `Integer.parseInt()` | `int i = Integer.parseInt("10");` | `int` |
|  | 래퍼 객체 필요 | `Integer.valueOf()` | `Integer i = Integer.valueOf("10");` | `Integer` |
| **모든 타입 → String** | **Null-Safe 변환** | **`String.valueOf()`** | `String s = String.valueOf(obj);` | `String` |
|  | 객체 상태 표현 | `obj.toString()` | `String s = user.toString();` | `String` |
| **래퍼 객체 → 원시 타입** | 값 추출 (언박싱) | `obj.intValue()` | `int i = integerObj.intValue();` | `int` |

## 7. 결론

자료형 변환 메서드를 선택할 때, 더 이상 망설일 필요가 없습니다. 다음 두 가지 질문만 스스로에게 던져보면 됩니다.

1. **"내가 원하는 최종 결과물이 원시 타입(값)인가, 래퍼 객체인가?"** (`parseInt` vs `valueOf`)
2. **"변환하려는 대상이 `null`일 수 있는가?"** (`toString` vs `String.valueOf`)

이 두 가지 기준만 명확히 세우면, 어떤 상황에서든 정확하고 안전한 코드를 작성할 수 있을 것입니다.
