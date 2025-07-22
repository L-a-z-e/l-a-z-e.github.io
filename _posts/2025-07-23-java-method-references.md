---
title: 메서드 참조, 파라미터는 어디서 오는 걸까? (동작 원리 완벽 해부)
description: 
author: laze
date: 2025-07-23 00:00:01 +0900
categories: [Dev, Java]
tags: [Lambda, Java, Method References, Functional Interface]
---
# 메서드 참조, 파라미터는 어디서 오는 걸까? (동작 원리 완벽 해부)

## 1. 서론: 마법처럼 보이는 메서드 참조

Java 8에서 람다와 함께 도입된 메서드 참조(`::`)는 코드를 놀랍도록 간결하게 만들어 줍니다. 하지만 처음 마주하면 그 동작 방식이 마법처럼 느껴져 혼란스러울 때가 많습니다.

```java
List<String> names = List.of("Alice", "Bob", "Charlie");

// 람다 표현식
names.forEach(name -> System.out.println(name));

// 메서드 참조
names.forEach(System.out::println);
```

두 코드는 완벽하게 동일하게 동작합니다. 하지만 `System.out::println`에는 `name`이라는 파라미터가 보이지 않습니다. 대체 이 파라미터는 어디로 사라졌다가, 어떻게 `println` 메서드에 전달되는 것일까요?

이 글에서는 메서드 참조가 마법이 아니라, **'함수형 인터페이스'**와 **'컨텍스트(Context)'**를 기반으로 동작하는 매우 정교한 문법임을 심층적으로 분석해 보겠습니다.

## 2. 모든 것의 시작: 함수형 인터페이스

메서드 참조를 이해하기 위한 첫 번째 열쇠는 바로 **함수형 인터페이스(Functional Interface)**입니다. 함수형 인터페이스란, **단 하나의 추상 메서드(abstract method)만을 가진 인터페이스**를 말합니다.

Java는 `java.util.function` 패키지에 다양한 표준 함수형 인터페이스를 제공합니다.

- `Consumer<T>`: `void accept(T t)` - T 타입의 값을 하나 받아 소비하고, 반환값은 없다.
- `Supplier<T>`: `T get()` - 아무것도 받지 않고, T 타입의 값을 반환한다.
- `Function<T, R>`: `R apply(T t)` - T 타입의 값을 받아 R 타입의 값으로 변환하여 반환한다.
- `Predicate<T>`: `boolean test(T t)` - T 타입의 값을 받아 `boolean`을 반환한다.

Java의 람다와 메서드 참조는 바로 이 "단 하나의 추상 메서드"를 구현하는 익명 클래스를 매우 간결하게 표현하는 방법입니다.

## 3. 파라미터의 비밀: 컨텍스트가 모든 것을 결정한다

"파라미터는 어디에 있는가?" 라는 질문에 대한 답은 **"메서드 참조가 사용되는 바로 그 자리(Context)가 알고 있다"** 입니다.

`System.out::println`이라는 표현 자체는 아무런 동작도 하지 않습니다. 이것은 그저 "이런 모양의 메서드가 저기 있다"는 **'설계도' 또는 '메서드의 주소값'**에 불과합니다. 이 설계도가 실제 '구현체'가 되려면, 특정 함수형 인터페이스가 필요한 자리에 놓여야 합니다.

`names.forEach(System.out::println);` 코드를 컴파일러의 시점에서 따라가 보겠습니다.

1. 컴파일러는 `names.forEach()` 메서드를 분석합니다. `names`는 `List<String>` 타입입니다.
2. `List`의 `forEach` 메서드 시그니처를 확인합니다: `void forEach(Consumer<? super String> action)`
3. "아하! `forEach`는 `Consumer<String>` 타입의 객체를 파라미터로 받는구나."
4. `Consumer<String>` 인터페이스의 유일한 추상 메서드를 확인합니다: `void accept(String t)`
5. "그렇다면, 이 자리에 들어올 람다나 메서드 참조는 **'String 하나를 받아서 아무것도 반환하지 않는(void)'** 모양이어야겠군."
6. 이제 `System.out::println`을 분석합니다.
7. `System.out` 객체에는 `println(String s)` 라는 메서드가 존재합니다. 이 메서드는 String 하나를 받고 아무것도 반환하지 않습니다.
8. "놀랍군! `println(String s)`는 `accept(String t)`와 모양(시그니처)이 완벽하게 일치한다!"
9. 컴파일러는 `System.out::println`을 `(String s) -> System.out.println(s)` 라는 람다 표현식으로 안전하게 변환합니다. `forEach`가 리스트를 순회할 때마다 각 요소(`"Alice"`, `"Bob"`, ...)가 이 람다의 `s` 파라미터로 전달되어 실행됩니다.

결론적으로, **메서드 참조의 생략된 파라미터는, 그 메서드 참조가 할당되는 함수형 인터페이스의 추상 메서드 시그니처로부터 온다**고 할 수 있습니다.

## 4. 메서드 참조의 4가지 유형 완전 정복

메서드 참조는 크게 4가지 유형으로 나뉩니다. 이 유형을 이해하면 어떤 상황에서든 자유자재로 사용할 수 있습니다.

### 유형 1: 정적 메서드 참조 (`ClassName::staticMethod`)

람다의 모든 파라미터를 정적 메서드에 그대로 전달합니다.

```java
// 람다 표현식
Function<String, Integer> lambda = s -> Integer.parseInt(s);

// 메서드 참조
Function<String, Integer> reference = Integer::parseInt;

Integer result = reference.apply("123"); // 123

```

- `Function<String, Integer>`의 `apply(String t)`는 `Integer.parseInt(String s)`와 시그니처가 일치합니다.

### 유형 2: 특정 객체의 인스턴스 메서드 참조 (`instance::instanceMethod`)

람다의 모든 파라미터를 **이미 생성된 특정 객체(`instance`)**의 메서드에 전달합니다. `System.out::println`이 바로 이 경우입니다. (`System.out`은 `PrintStream` 타입의 특정 인스턴스입니다.)

```java
String prefix = "Hello, ";

// 람다 표현식
Function<String, String> lambda = name -> prefix.concat(name);

// 메서드 참조
Function<String, String> reference = prefix::concat;

String greeting = reference.apply("Java"); // "Hello, Java"

```

### 유형 3: 임의 객체의 인스턴스 메서드 참조 (`ClassName::instanceMethod`)

가장 헷갈리지만 매우 강력한 유형입니다. 람다의 **첫 번째 파라미터가 메서드를 호출할 주체(객체)**가 되고, 나머지 파라미터가 그 메서드의 인자로 전달됩니다.

```java
// 람다 표현식: 두 문자열을 받아 첫 번째 문자열이 비어있는지 확인 (두 번째는 무시)
Predicate<String> lambda = s -> s.isEmpty();

// 메서드 참조
Predicate<String> reference = String::isEmpty;

reference.test(""); // true

// 파라미터가 2개인 경우
// 람다 표현식: str1이 str2로 시작하는지 확인
BiPredicate<String, String> lambda2 = (str1, str2) -> str1.startsWith(str2);

// 메서드 참조
BiPredicate<String, String> reference2 = String::startsWith;

reference2.test("Java", "J"); // true

```

`BiPredicate<String, String>`의 `test(String t, String u)`에 `String::startsWith`가 오면, 컴파일러는 이를 `t.startsWith(u)`로 해석합니다.

### 유형 4: 생성자 참조 (`ClassName::new`)

람다의 파라미터를 받아 새 객체를 생성합니다.

```java
// 파라미터가 없는 생성자
// 람다 표현식
Supplier<List<String>> lambda = () -> new ArrayList<>();

// 메서드 참조
Supplier<List<String>> reference = ArrayList::new;

List<String> list = reference.get();

// 파라미터가 있는 생성자
// 람다 표현식
Function<String, User> lambda2 = name -> new User(name);

// 메서드 참조
Function<String, User> reference2 = User::new;

User user = reference2.apply("Alice");

```

## 5. 실전 예제: 람다를 메서드 참조로 리팩토링하기

Stream API를 사용할 때, 메서드 참조는 코드를 극도로 간결하고 가독성 높게 만들어 줍니다.

```java
List<String> stringNumbers = List.of("1", "2", "3", "", "5");

// Before: 람다 표현식 사용
List<Integer> numbers = stringNumbers.stream()
        .filter(s -> !s.isEmpty())
        .map(s -> Integer.parseInt(s))
        .collect(Collectors.toList());

// After: 메서드 참조 사용
List<Integer> numbersRef = stringNumbers.stream()
        .filter(Predicate.not(String::isEmpty)) // `String::isEmpty`의 결과를 반전
        .map(Integer::parseInt)
        .collect(Collectors.toList());
```

## 6. 결론

메서드 참조는 독립적으로 동작하는 코드가 아닙니다. 그 자체로는 단순한 '메서드 포인터'일 뿐입니다.
**메서드 참조의 생략된 파라미터는, 그것이 할당되는 함수형 인터페이스의 추상 메서드 시그니처로부터 추론된다**는 것이 핵심 원리입니다.

처음에는 어색할 수 있지만, 이 원리를 이해하고 나면 람다 표현식을 메서드 참조로 바꾸는 것이 자연스러워집니다. 적절한 상황에 메서드 참조를 사용하면, 코드를 훨씬 더 간결하고 의도가 명확하게 만들 수 있는 강력한 무기가 될 것입니다.

---
