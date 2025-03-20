---
title: 자바의 정석 Chapter 14
description: 모르는 내용 위주 학습 기록
author: laze
date: 2025-02-27 00:00:14 +0900
categories: [Dev, Java]
tags: [Java]
---
# Chapter 14

람다식

- 메서드를 하나의 식으로 표현한 것
- 메서드를 변수처럼 다루는 것이 가능해짐
- 람다식 작성
    - ~~반환타입 메서드이름~~ (매개변수) → { 표현식 }

함수형 인터페이스

- 하나의 추상 메서드만 정의
- 그외 default, static 메서드는 제약 없음
- @FunctionalInterface 사용 → 컴파일러에서 체크 해줌
- 람다식을 참조변수 형태로 사용할 수 있게함 → 변수처럼 메서드를 주고 받는 것이 가능해짐
- 람다식 내에서 참조하는 지역변수는 final이 붙어있지 않아도 상수로 간주됨 → 값변경 불가

java.util.Function 패키지

- Runnable - void run()
- Supplier<T> - T get()
- Consumer<T> - void accept(T t)
- Function<T, R> - R apply(T t)
- Predicate<T> - boolean test(T t)
- UnaryOperator<T> - T apply(T t)

기본형 사용 함수형 인터페이스

- DoubleToIntFunction - int applyAsInt(doulbe d)
- ToIntFunction<T> - int applyAsInt(T value)
- IntFunction<R> - R apply(int i)
- ObjIntConsumer<T> - void accpet(T t, int i)

Function 합성

```java
Function
default <V> Function<T, V> andThen(Function<? super R, ? extends V> after)
default <V> Function<V, R> compose(Function<? super V, ? extends T> before)
static <T> Function<T, T> identity()
```

- Function 추가 궁금점
    
    알겠습니다. `andThen`과 `compose` 메서드의 제네릭 타입 정의 자체를 이해하고 싶다는 말씀이시군요. `V`, `T`, `R` 그리고 `super`와 `extends`를 사용하여 제네릭 타입을 제한하는 이유를 자세히 설명해 드리겠습니다.
    
    **다시 한번 메서드 시그니처:**
    
    ```java
    default <V> Function<T, V> andThen(Function<? super R, ? extends V> after)
    default <V> Function<V, R> compose(Function<? super V, ? extends T> before)
    
    ```
    
    **각 타입 파라미터의 의미:**
    
    - **`T`**: 현재 `Function` 인터페이스의 입력 타입입니다. 즉, `Function<T, R>`에서 `T`는 `apply` 메서드의 인수의 타입입니다.
    - **`R`**: 현재 `Function` 인터페이스의 반환 타입입니다. 즉, `Function<T, R>`에서 `R`은 `apply` 메서드의 반환값의 타입입니다.
    - **`V`**: `andThen`과 `compose` 메서드가 반환하는 새로운 `Function`의 최종 반환 타입입니다. 이 타입은 메서드 수준에서 새롭게 정의되는 제네릭 타입 파라미터입니다.
    - **`? super R` (andThen):** `after` 함수의 입력 타입으로, `R` 또는 `R`의 상위 타입이 올 수 있습니다.
    - **`? extends V` (andThen):** `after` 함수의 반환 타입으로, `V` 또는 `V`의 하위 타입이 올 수 있습니다.
    - **`? super V` (compose):** `before` 함수의 입력 타입으로, `V` 또는 `V`의 상위 타입이 올 수 있습니다.
    - **`? extends T` (compose):** `before` 함수의 반환 타입으로, `T` 또는 `T`의 하위 타입이 올 수 있습니다.
    
    **`super`와 `extends`를 사용하는 이유 (공변성/반공변성):**
    
    `? super`와 `? extends`는 제네릭 타입의 유연성을 높이기 위해 사용됩니다. 이는 **공변성(Covariance)**과 **반공변성(Contravariance)**의 개념과 관련이 있습니다.
    
    - **`? super R` (반공변성 - Contravariance):**
        - `after` 함수는 `R` 타입의 값을 "소비"합니다. 즉, `R` 타입의 인수를 받아서 어떤 작업을 수행합니다.
        - `? super R`은 "R 또는 R의 상위 타입"을 의미합니다.
        - **왜 사용할까요?** `after` 함수가 `R` 타입뿐만 아니라 `R`의 상위 타입도 처리할 수 있다면, 더 많은 종류의 함수를 `andThen`에 사용할 수 있습니다. 예를 들어, `R`이 `String`이라면 `after` 함수는 `String`을 처리하는 함수일 수도 있고, `Object`를 처리하는 함수일 수도 있습니다. `Object`는 `String`의 상위 타입이므로, `String`을 입력으로 받는 함수는 당연히 `Object`를 입력으로 받을 수 있습니다.
        - **예시:** `List<? super Integer>`는 `List<Integer>`, `List<Number>`, `List<Object>` 등이 될 수 있습니다. 즉, Integer를 안전하게 추가할 수 있는 List를 의미합니다.
    - **`? extends V` (공변성 - Covariance):**
        - `after` 함수는 `V` 타입의 값을 "생산"합니다. 즉, `V` 타입의 결과를 반환합니다.
        - `? extends V`는 "V 또는 V의 하위 타입"을 의미합니다.
        - **왜 사용할까요?** `after` 함수가 `V` 타입뿐만 아니라 `V`의 하위 타입을 반환하더라도, `andThen` 메서드는 여전히 `V` 타입의 값을 반환할 수 있습니다. 예를 들어, `V`가 `Number`라면 `after` 함수는 `Integer`를 반환할 수도 있고, `Double`을 반환할 수도 있습니다. `Integer`와 `Double`은 모두 `Number`의 하위 타입이므로, `andThen` 메서드는 안전하게 `Number` 타입의 값을 반환할 수 있습니다.
        - **예시:** `List<? extends Number>`는 `List<Number>`, `List<Integer>`, `List<Double>` 등이 될 수 있습니다. 즉, Number 타입으로 안전하게 읽을 수 있는 List를 의미합니다.
    - **`compose` 메서드에서의 `? super V`와 `? extends T`:** `andThen`과 동일한 원리가 적용됩니다. `before` 함수가 `V` 또는 `V`의 상위 타입을 입력으로 받을 수 있도록 하고, `T` 또는 `T`의 하위 타입을 반환할 수 있도록 하여 유연성을 높입니다.
    
    **제네릭 타입 정의의 목적:**
    
    - **타입 안전성:** 컴파일 시점에 타입 에러를 잡아낼 수 있도록 합니다.
    - **유연성:** 다양한 타입의 함수를 조합할 수 있도록 합니다.
    
    **예시를 통한 설명:**
    
    ```java
    Function<Number, String> numberToString = (Number n) -> "Number: " + n;
    Function<Integer, Number> intToNumber = (Integer i) -> i.doubleValue();
    
    // 1. andThen
    Function<Integer, String> combined1 = intToNumber.andThen(numberToString); // 가능
    // intToNumber: Integer -> Number
    // numberToString: Number -> String
    // andThen 결과: Integer -> String
    
    // 2. compose
    Function<Integer, String> combined2 = numberToString.compose(intToNumber); // 에러! (타입 불일치)
    // numberToString: Number -> String
    // intToNumber: Integer -> Number
    // compose 결과: Integer -> String 이 되어야 하지만, numberToString 은 Integer 를 받을 수 없음
    
    Function<Integer, String> combined3 = numberToString.compose((Function<Integer, ? extends Number>) intToNumber); // 가능 (명시적 타입 캐스팅)
    // numberToString: Number -> String
    // intToNumber: Integer -> Number (Number 의 하위 타입인 Integer 를 반환하므로 안전)
    // compose 결과: Integer -> String
    
    ```
    
    **결론:**
    
    `andThen`과 `compose` 메서드의 복잡한 제네릭 타입 정의는 타입 안전성을 보장하면서도 최대한의 유연성을 제공하기 위한 설계입니다. `super`와 `extends`를 사용하여 제네릭 타입을 제한함으로써, 다양한 타입의 함수를 안전하게 조합할 수 있도록 합니다. 이러한 제네릭 타입 정의를 이해하는 것은 함수형 프로그래밍을 더욱 효과적으로 활용하는 데 필수적입니다.
    
    이 설명이 제네릭 타입 정의 자체를 이해하는 데 도움이 되었기를 바랍니다. 혹시 더 궁금한 점이 있다면 언제든지 질문해주세요.
    
- 공변성 반공변성
    
    공변성과 반공변성은 객체 지향 프로그래밍과 타입 이론에서 중요한 개념으로, 상속 관계가 있는 타입 간의 관계를 어떻게 다룰 것인지에 대한 규칙을 정의합니다. 특히 제네릭 타입과 함께 사용될 때 그 중요성이 더욱 부각됩니다.
    
    **1. 공변성 (Covariance):**
    
    - **정의:** 하위 타입 관계가 유지되는 것을 의미합니다. 즉, `Sub`가 `Super`의 하위 타입이라면, `Generic<Sub>`는 `Generic<Super>`의 하위 타입이 됩니다. (엄밀히 말하면, Java의 제네릭은 기본적으로 무공변입니다. 공변성을 지원하려면 `? extends`를 사용해야 합니다.)
    - **예시:**
        - `Integer`는 `Number`의 하위 타입입니다.
        - `List<Integer>`는 `List<? extends Number>`의 하위 타입입니다. (Java에서 `List<Integer>`는 `List<Number>`의 하위 타입이 **아닙니다**. `List<? extends Number>`를 사용해야 공변성을 표현할 수 있습니다.)
    - **동작 방식:**
        - 공변성은 주로 "읽기 (read)" 작업에서 유용합니다. `List<? extends Number>`에서 `Number` 타입의 값을 안전하게 읽을 수 있습니다. 왜냐하면 리스트에 저장된 모든 요소는 `Number` 또는 `Number`의 하위 타입이기 때문입니다.
        - 하지만 "쓰기 (write)" 작업은 일반적으로 안전하지 않습니다. `List<? extends Number>`에 어떤 값을 써야 할지 컴파일러는 알 수 없습니다. `Integer`만 들어있는 리스트일 수도 있고, `Double`만 들어있는 리스트일 수도 있습니다. 따라서 컴파일러는 타입 안전성을 위해 쓰기 작업을 허용하지 않습니다.
    - **`? extends` (와일드카드):** Java 제네릭에서 공변성을 표현하기 위해 사용됩니다. `Generic<? extends Type>`은 "Type 또는 Type의 하위 타입"을 의미합니다.
    
    **2. 반공변성 (Contravariance):**
    
    - **정의:** 하위 타입 관계가 반대로 되는 것을 의미합니다. 즉, `Sub`가 `Super`의 하위 타입이라면, `Generic<Super>`는 `Generic<Sub>`의 하위 타입이 됩니다. (Java에서 반공변성을 지원하려면 `? super`를 사용해야 합니다.)
    - **예시:**
        - `Integer`는 `Number`의 하위 타입입니다.
        - `Consumer<Number>`는 `Consumer<? super Integer>`의 하위 타입입니다. (Java에서 `Consumer<Number>`는 `Consumer<Integer>`의 하위 타입이 **아닙니다**. `Consumer<? super Integer>`를 사용해야 반공변성을 표현할 수 있습니다.)
    - **동작 방식:**
        - 반공변성은 주로 "쓰기 (write)" 작업에서 유용합니다. `Consumer<? super Integer>`는 `Integer` 또는 `Integer`의 상위 타입의 값을 안전하게 "소비"할 수 있습니다. 즉, `accept` 메서드에 `Integer` 타입의 값을 전달할 수 있습니다. 왜냐하면 `Consumer`는 `Integer`를 처리할 수 있거나, `Integer`의 상위 타입 (예: `Number`, `Object`)을 처리할 수 있기 때문입니다.
        - 하지만 "읽기 (read)" 작업은 일반적으로 안전하지 않습니다. `Consumer<? super Integer>`로부터 어떤 값을 읽어야 할지 컴파일러는 알 수 없습니다.
    - **`? super` (와일드카드):** Java 제네릭에서 반공변성을 표현하기 위해 사용됩니다. `Generic<? super Type>`은 "Type 또는 Type의 상위 타입"을 의미합니다.
    
    **3. 무공변성 (Invariance):**
    
    - **정의:** 타입 파라미터가 정확히 일치해야만 타입 관계가 성립하는 것을 의미합니다. Java 제네릭은 기본적으로 무공변입니다.
    - **예시:**
        - `Integer`는 `Number`의 하위 타입입니다.
        - `List<Integer>`는 `List<Number>`와 아무런 상속 관계가 없습니다. (`List<Integer>`는 `List<Number>`의 하위 타입도 아니고, 상위 타입도 아닙니다.)
    
    **요약:**
    
    | 특징 | 공변성 (`? extends`) | 반공변성 (`? super`) | 무공변성 (기본 제네릭) |
    | --- | --- | --- | --- |
    | 타입 관계 | 하위 타입 관계 유지 (Sub -> Super, Generic<Sub> -> Generic<Super>) | 하위 타입 관계 반전 (Sub -> Super, Generic<Super> -> Generic<Sub>) | 타입 파라미터가 정확히 일치해야 함 |
    | 주요 작업 | 읽기 (read) | 쓰기 (write) | 정확한 타입 일치 |
    | 사용 예시 | `List<? extends Number>` (Number 또는 Number의 하위 타입 리스트) | `Consumer<? super Integer>` (Integer 또는 Integer의 상위 타입 소비) | `List<Integer>` (정확히 Integer 타입만 허용) |
    
    **`andThen`과 `compose` 메서드에서의 공변성/반공변성 활용:**
    
    - **`andThen(Function<? super R, ? extends V> after)`:**
        - `? super R`: `after` 함수가 `R` 또는 `R`의 상위 타입을 입력으로 받을 수 있도록 반공변성을 활용하여 유연성을 높입니다.
        - `? extends V`: `after` 함수가 `V` 또는 `V`의 하위 타입을 반환할 수 있도록 공변성을 활용하여 유연성을 높입니다.
    - **`compose(Function<? super V, ? extends T> before)`:**
        - `? super V`: `before` 함수가 `V` 또는 `V`의 상위 타입을 입력으로 받을 수 있도록 반공변성을 활용하여 유연성을 높입니다.
        - `? extends T`: `before` 함수가 `T` 또는 `T`의 하위 타입을 반환할 수 있도록 공변성을 활용하여 유연성을 높입니다.
    
    **핵심:**
    
    공변성과 반공변성은 제네릭 타입을 사용할 때 타입 안전성을 유지하면서도 코드의 유연성을 높이는 데 사용되는 중요한 개념입니다. `? extends`와 `? super` 와일드카드를 통해 제네릭 타입의 상속 관계를 명시적으로 지정하여 컴파일러가 타입 에러를 잡아낼 수 있도록 하고, 동시에 다양한 타입의 객체를 처리할 수 있도록 합니다.
    

Predicate 결합

```java
Predicate
default Predicate<T> and(Predicate<? super T> other)
default Predicate<T> or(Predicate<? super T> other)
default Predicate<T> negate()
static <T> predicate<T> isEqual(Object targetRef)
```

메서드 참조

| 종류 | 람다 | 메서드참조 |
| --- | --- | --- |
| static메서드 | (x) → ClassName.method(x) | ClassName::method |
| 인스턴스메서드 | (obj, x) → obj.method(x) | ClassName::method |
| 특정 객체 인스턴스메서드 | (x) → obj.method(x) | obj::method |

생성자의 메서드 참조

- 샘플 코드
    
    ```java
    import java.util.function.Supplier;
    import java.util.function.Function;
    
    class MyObject {
        private String name;
    
        // 기본 생성자 (no-arg constructor)
        public MyObject() {
            this.name = "Default Name";
            System.out.println("MyObject() called");
        }
    
        // 인수를 받는 생성자 (one-arg constructor)
        public MyObject(String name) {
            this.name = name;
            System.out.println("MyObject(String name) called: " + name);
        }
    
        public String getName() {
            return name;
        }
    
        @Override
        public String toString() {
            return "MyObject{name='" + name + "'}";
        }
    }
    
    public class Main {
        public static void main(String[] args) {
            // 1. 기본 생성자 참조 (no-arg constructor)
            Supplier<MyObject> myObjectSupplier = MyObject::new;
            MyObject obj1 = myObjectSupplier.get(); // MyObject() called
            System.out.println(obj1); // MyObject{name='Default Name'}
    
            // 2. 인수를 받는 생성자 참조 (one-arg constructor)
            Function<String, MyObject> myObjectFunction = MyObject::new;
            MyObject obj2 = myObjectFunction.apply("Custom Name"); // MyObject(String name) called: Custom Name
            System.out.println(obj2); // MyObject{name='Custom Name'}
        }
    }
    
    ```
    
    **설명:**
    
    1. **기본 생성자 참조 (`MyObject::new`):**
        - `Supplier<MyObject> myObjectSupplier = MyObject::new;`는 `MyObject`의 기본 생성자를 참조하는 `Supplier` 인터페이스를 구현합니다.
        - `myObjectSupplier.get()`을 호출하면 `MyObject`의 기본 생성자가 실행되어 새로운 `MyObject` 객체가 생성됩니다.
    2. **인수를 받는 생성자 참조 (`MyObject::new`):**
        - `Function<String, MyObject> myObjectFunction = MyObject::new;`는 `String` 타입의 인수를 받는 `MyObject` 생성자를 참조하는 `Function` 인터페이스를 구현합니다.
        - `myObjectFunction.apply("Custom Name")`을 호출하면 `String` 인수를 받는 `MyObject` 생성자가 실행되어 새로운 `MyObject` 객체가 생성됩니다.
    
    **더 복잡한 예시 (Collection API 활용):**
    
    ```java
    import java.util.Arrays;
    import java.util.List;
    import java.util.ArrayList;
    import java.util.stream.Collectors;
    
    class Person {
        private String name;
    
        public Person(String name) {
            this.name = name;
        }
    
        public String getName() {
            return name;
        }
    
        @Override
        public String toString() {
            return "Person{name='" + name + "'}";
        }
    }
    
    public class Main {
        public static void main(String[] args) {
            List<String> names = Arrays.asList("Alice", "Bob", "Charlie");
    
            // 생성자 참조를 사용하여 String 리스트를 Person 리스트로 변환
            List<Person> people = names.stream()
                                       .map(Person::new) // Person(String name) 생성자 참조
                                       .collect(Collectors.toList());
    
            System.out.println(people);
            // 출력: [Person{name='Alice'}, Person{name='Bob'}, Person{name='Charlie'}]
        }
    }
    
    ```
    
    **설명:**
    
    - `names.stream().map(Person::new)`는 `names` 리스트의 각 `String` 요소에 대해 `Person` 클래스의 `String` 인수를 받는 생성자를 호출하여 새로운 `Person` 객체를 생성합니다.
    - `collect(Collectors.toList())`는 생성된 `Person` 객체들을 모아서 새로운 `List<Person>`을 생성합니다.
    
    **이해가 안 되는 부분 해소:**
    
    생성자 참조는 `ClassName::new` 형태로 사용하며, 컴파일러는 문맥(context)에 따라 어떤 생성자를 호출해야 하는지 판단합니다. `Supplier` 인터페이스를 사용하는 경우 기본 생성자가 호출되고, `Function` 인터페이스를 사용하는 경우 인수를 받는 생성자가 호출됩니다. `map` 메서드와 함께 사용될 때는 스트림의 요소가 생성자의 인자로 전달됩니다.
    
    **핵심:**
    
    - `ClassName::new`는 생성자 메서드 참조의 기본 형태입니다.
    - 컴파일러는 함수형 인터페이스의 타입과 메서드의 인자 타입에 따라 어떤 생성자를 호출할지 결정합니다.
    - 생성자 참조는 컬렉션을 변환하거나 객체를 동적으로 생성해야 할 때 유용하게 사용될 수 있습니다.
    
    이 예시들이 생성자 메서드 참조를 이해하는 데 도움이 되었기를 바랍니다. 더 궁금한 점이 있다면 언제든지 질문해주세요.
    

스트림

- 데이터소스를 추상화하고, 데이터를 다루는데 자주 사용되는 메서드들을 정의
- 데이터 소스를 변경하지 않음
- 일회용
- 내부 반복으로 작업을 처리함
- 중간 연산 → 연산 결과가 스트림인 연산
- 최종 연산 → 연산 결과가 스트림이 아닌 연산 → 스트림의 요소를 소모하므로 한 번만 가능
- 지연된 연산 → 최종 연산이 수행되기 전까지는 중간 연산이 수행되지 않는다
- 병렬 스트림 → 내부적으로 fork&join 프레임웍을 이용해 자동적으로 연산을 병렬로 수행함

| 중간연산 | 설명 |
| --- | --- |
| Stream<T> distinct() | 중복 제거 |
| Stream<T> filter(Predicate<T> predicate> | 조건에 안맞는 요소 제거 |
| Stream<T> limit(long maxSize) | 스트림의 일부를 잘라냄 |
| Stream<T> skip(long n) | 스트림의 일부를 건너뜀 |
| Stream<T> peek(Consumer<T> action) | 스트림의 요소에 작업 수행 |
| Stream<T> sorted()
Stream<T> sorted(Comparator<T> comparator) | 스트림의 요소 정렬 |
| Stream<R> map (Function<T,R> mapper) | 스트림의 요소 변환 |
| DoubleStream mapToDouble(ToDoubleFunction<T> mapper) | 스트림의 요소 변환 |
| IntStream mapToInt(ToIntFunction<T> mapper) | 스트림의 요소 변환 |
| LongStream mapToLong(ToLongFunction<T> mapper) | 스트림의 요소 변환 |
| Stream<R> faltMap(Function<T, Stream<R>> mapper) | 스트림의 요소 변환 |
| DoubleStream flatMapToDouble(Function<T, DoubleStream> m) | 스트림의 요소 변환 |
| IntStrean flatMapToInt(Function<T, IntStream> m) | 스트림의 요소 변환 |
| LongStream flatMapToLong(Function<T, LongStream> m) | 스트림의 요소 변환 |

최종연산

| 최종연산 | 설명 |
| --- | --- |
| void forEach(Consumer<? super T> action)
void forEachOrdered(Consumer<? super T> action) | 각 요소에 지정된 작업 수행 |
| long count() | 스트림의 요소의 개수 반환 |
| Optional<T> max(Comparator<? super T> comparator)
Optional<T> min(Comparator<? super T> comparator) | 스트림의 최대/최소값 반환 |
| Optional<T> findAny() | 스트림의 요소 반환 (아무거나) |
| Optional<T> findFirst() | 스트리므이 요소 반환(첫 번째) |
| boolean allMatch(Predicate<T> p) | 주어진 조건을 모든 요소가 만족 시키는지 |
| boolean anyMatch(Predicate<T> p) | 주어진 조건을 하나라도 만족하는 요소가 있는지 |
| boolean noneMatch(Predicate<T> p) | 주어진 조건을 모든 요소가 만족시키지 않는지 |
| Object[] toArray() | 스트림의 모든 요소를 배열로 반환 |
| A[] toArray(IntFunction<A[]> generator) | 스트림의 모든 요소를 배열로 반환 |
| Optional<T> reduce(BinaryOperator<T> accumulator) | 스트림의 요소를 하나씩 줄여가며 계산 |
| T reduce(T identity, BinaryOperator<T> accumulator) | 스트림의 요소를 하나씩 줄여가며 계산 |
| U reduce(U identity, BiFunction<U,T,U> accumulator, BinaryOperator<U> combiner) | 스트림의 요소를 하나씩 줄여가며 계산 |
| R collect(Collector<T,A,R> collector) | 스트림의 요소를 수집 |
| R collect(Supplier<R> supplier, BiConsumer<R, T> accumulator, BiConsumer<R, R> combiner) | 스트림의 요소를 수집 |
- Stream<T> Stream<R>
    
    `Stream<T>`와 `Stream<R>`의 차이는 스트림이 다루는 **요소의 타입**에 있습니다.
    
    - `Stream<T>`: 스트림이 `T` 타입의 요소를 다룬다는 의미입니다. 여기서 `T`는 특정 클래스, 인터페이스, 또는 기본형의 래퍼 클래스(예: `Integer`, `Double`)가 될 수 있습니다.
    - `Stream<R>`: 스트림이 `R` 타입의 요소를 다룬다는 의미입니다. 여기서 `R`은 `T`와 다른 타입일 수 있습니다.
    
    **언제 `Stream<T>`를 쓰고, 언제 `Stream<R>`를 쓸까요?**
    
    - **초기 스트림 생성:** 스트림을 처음 생성할 때, 스트림이 다룰 요소의 타입에 따라 `Stream<T>`를 사용합니다. 예를 들어, `List<String>`에서 스트림을 생성한다면 `Stream<String>`이 됩니다.
    - **`map` 연산:** `map` 연산은 스트림의 요소를 변환합니다. 만약 `Stream<T>`의 각 요소를 `R` 타입으로 변환한다면, `map` 연산의 결과는 `Stream<R>`이 됩니다.
    
    **예시:**
    
    ```java
    import java.util.Arrays;
    import java.util.List;
    import java.util.stream.Stream;
    
    public class Main {
        public static void main(String[] args) {
            List<String> names = Arrays.asList("Alice", "Bob", "Charlie");
    
            // 1. 초기 스트림: Stream<String>
            Stream<String> nameStream = names.stream();
    
            // 2. map 연산: Stream<String> -> Stream<Integer> (String의 길이를 Integer로 변환)
            Stream<Integer> nameLengthStream = nameStream.map(String::length);
    
            nameLengthStream.forEach(System.out::println); // 5 3 7
        }
    }
    
    ```
    
    **설명:**
    
    1. `names.stream()`: `List<String>`에서 스트림을 생성하므로, 초기 스트림은 `Stream<String>`입니다. (`T`는 `String`)
    2. `nameStream.map(String::length)`: `map` 연산을 사용하여 `String` 요소를 `Integer` (문자열 길이)로 변환합니다. 따라서 `map` 연산의 결과는 `Stream<Integer>`입니다. (`R`은 `Integer`)
    
    **정리:**
    
    - `Stream<T>`는 스트림의 초기 요소 타입을 나타냅니다.
    - `Stream<R>`은 `map`과 같은 중간 연산을 통해 요소의 타입이 변경되었을 때 사용됩니다.
    - `T`와 `R`은 같은 타입일 수도 있고, 다른 타입일 수도 있습니다.
    
    핵심은 스트림의 각 단계에서 다루는 요소의 타입을 정확히 파악하는 것입니다.
    
- Consumer<? super T>
    
    `forEach(Consumer<? super T> action)`와 `forEachOrdered(Consumer<? super T> action)` 메서드에서 `super`가 사용된 이유는 **반공변성(contravariance)**을 활용하기 위함입니다. 이를 통해 메서드의 유연성을 높이고, 더 다양한 타입의 `Consumer`를 사용할 수 있도록 합니다.
    
    **`Consumer<? super T> action`의 의미:**
    
    - `Consumer<T>`: `T` 타입의 인수를 받아서 소비(사용)하는 함수형 인터페이스입니다.
    - `Consumer<? super T>`: `T` 또는 `T`의 상위 타입의 인수를 받아서 소비하는 함수형 인터페이스입니다.
    
    **`super`가 필요한 이유 (반공변성):**
    
    `forEach` 메서드는 스트림의 각 요소에 대해 주어진 `action`을 수행합니다. 여기서 `action`은 `Consumer` 인터페이스를 구현한 객체입니다. `super` 키워드를 사용하면, `action`이 `T` 타입의 객체뿐만 아니라 `T`의 상위 타입의 객체도 처리할 수 있게 됩니다.
    
    **예시:**
    
    ```java
    import java.util.Arrays;
    import java.util.List;
    import java.util.function.Consumer;
    
    class Animal {
        public void eat() {
            System.out.println("Animal is eating");
        }
    }
    
    class Dog extends Animal {
        @Override
        public void eat() {
            System.out.println("Dog is eating");
        }
    
        public void bark() {
            System.out.println("Dog is barking");
        }
    }
    
    public class Main {
        public static void main(String[] args) {
            List<Dog> dogs = Arrays.asList(new Dog(), new Dog());
    
            // 1. Consumer<Animal>을 사용할 수 있음 (반공변성)
            Consumer<Animal> animalConsumer = Animal::eat;
            dogs.forEach(animalConsumer); // Dog is eating (Dog 클래스에서 오버라이딩된 eat() 메서드 실행)
    
            // 2. Consumer<Dog>을 사용할 수도 있음
            Consumer<Dog> dogConsumer = Dog::bark;
            dogs.forEach(dogConsumer); // Dog is barking
    
            // 3. 만약 <? super T>가 없다면, Consumer<Animal>은 Dog 리스트에 사용할 수 없음
            // dogs.forEach((Consumer<Dog>) animalConsumer); // 컴파일 에러 (타입 불일치)
        }
    }
    
    ```
    
    **설명:**
    
    1. `Animal` 클래스와 `Dog` 클래스가 있습니다. `Dog`는 `Animal`을 상속받습니다.
    2. `List<Dog>` 리스트를 생성합니다.
    3. `Consumer<Animal>` 타입의 `animalConsumer`는 `Animal` 객체를 받아서 `eat()` 메서드를 호출합니다.
    4. `forEach(animalConsumer)`를 호출하면, `animalConsumer`는 `Dog` 객체를 `Animal` 타입으로 간주하고 `eat()` 메서드를 호출합니다. `Dog` 클래스에서 `eat()` 메서드를 오버라이딩했기 때문에, "Dog is eating"이 출력됩니다.
    5. `Consumer<Dog>` 타입의 `dogConsumer`는 `Dog` 객체를 받아서 `bark()` 메서드를 호출합니다.
    
    **`super` 없이 `Consumer<T>`만 사용한다면:**
    
    `forEach` 메서드는 정확히 `T` 타입의 `Consumer`만 받을 수 있습니다. 따라서 `Consumer<Animal>`은 `List<Dog>`에 사용할 수 없습니다. 이는 유연성을 떨어뜨리고, 코드 재사용성을 저해합니다.
    
    **`forEachOrdered`와의 차이점:**
    
    `forEach`는 스트림의 요소 처리 순서를 보장하지 않습니다 (병렬 스트림의 경우). 반면 `forEachOrdered`는 스트림의 요소 처리 순서를 보장합니다. 하지만 `Consumer<? super T> action`에서 `super`가 사용되는 이유는 동일합니다.
    
    **요약:**
    
    `forEach(Consumer<? super T> action)`에서 `super`가 사용된 이유는 반공변성을 활용하여 메서드의 유연성을 높이기 위함입니다. 이를 통해 `forEach` 메서드는 `T` 타입뿐만 아니라 `T`의 상위 타입의 객체도 처리할 수 있게 되어 코드 재사용성을 높일 수 있습니다.
    
- A[] toArray(IntFunction<A[]> generator)
    
    `toArray(IntFunction<A[]> generator)` 메서드는 스트림의 모든 요소를 배열로 변환하여 반환하는 최종 연산입니다. 이 메서드는 일반적인 `toArray()` 메서드와 달리, **생성자 참조**를 인자로 받아서 배열의 타입을 지정하고, 배열의 크기를 동적으로 결정할 수 있도록 합니다.
    
    **인자:**
    
    - `IntFunction<A[]> generator`: 정수 값을 받아서 `A[]` 타입의 배열을 생성하는 함수형 인터페이스입니다. 여기서 `A`는 배열의 요소 타입을 나타냅니다.
    
    **동작 방식:**
    
    1. 스트림의 요소 개수를 파악합니다.
    2. `generator` 함수를 사용하여 스트림의 요소 개수와 동일한 크기의 `A[]` 타입 배열을 생성합니다.
    3. 스트림의 모든 요소를 생성된 배열에 순서대로 복사합니다.
    4. 배열을 반환합니다.
    
    **핵심:**
    
    - `generator` 함수를 사용하여 **배열의 타입을 명시적으로 지정**할 수 있습니다.
    - 스트림의 요소 개수에 맞춰 **동적으로 배열의 크기를 결정**합니다.
    
    **언제 사용해야 할까요?**
    
    - 스트림의 요소를 특정 타입의 배열로 변환해야 할 때 (예: `String[]`, `Integer[]`, `MyClass[]`)
    - 배열의 크기를 미리 알 수 없을 때 (스트림의 요소 개수가 동적으로 변할 수 있을 때)
    
    **샘플 코드:**
    
    ```java
    import java.util.Arrays;
    import java.util.List;
    
    class Person {
        private String name;
    
        public Person(String name) {
            this.name = name;
        }
    
        public String getName() {
            return name;
        }
    
        @Override
        public String toString() {
            return "Person{name='" + name + "'}";
        }
    }
    
    public class Main {
        public static void main(String[] args) {
            List<String> names = Arrays.asList("Alice", "Bob", "Charlie");
    
            // 1. String[] 배열로 변환
            String[] nameArray = names.stream()
                                      .toArray(String[]::new);
    
            System.out.println(Arrays.toString(nameArray)); // [Alice, Bob, Charlie]
    
            // 2. Person[] 배열로 변환 (생성자 참조 활용)
            List<Person> people = names.stream()
                                       .map(Person::new) // String -> Person 변환
                                       .toList();
    
            Person[] personArray = people.stream()
                                         .toArray(Person[]::new);
    
            System.out.println(Arrays.toString(personArray));
            // [Person{name='Alice'}, Person{name='Bob'}, Person{name='Charlie'}]
        }
    }
    
    ```
    
    **설명:**
    
    1. **`String[] nameArray = names.stream().toArray(String[]::new);`**
        - `String[]::new`는 `IntFunction<String[]>`을 구현하는 생성자 참조입니다. 이 생성자 참조는 정수 값 (배열 크기)을 받아서 해당 크기의 `String[]` 배열을 생성합니다.
        - `toArray()` 메서드는 스트림의 요소 개수를 파악한 후, `String[]::new`를 사용하여 해당 크기의 `String[]` 배열을 생성하고, 스트림의 모든 요소를 배열에 복사합니다.
    2. **`Person[] personArray = people.stream().toArray(Person[]::new);`**
        - `Person[]::new`는 `IntFunction<Person[]>`을 구현하는 생성자 참조입니다. 이 생성자 참조는 정수 값을 받아서 해당 크기의 `Person[]` 배열을 생성합니다.
        - `toArray()` 메서드는 스트림의 요소 개수를 파악한 후, `Person[]::new`를 사용하여 해당 크기의 `Person[]` 배열을 생성하고, 스트림의 모든 요소를 배열에 복사합니다.
    
    **`toArray()` vs `toArray(IntFunction<A[]> generator)`:**
    
    - `toArray()`: `Object[]` 타입의 배열을 반환합니다. 스트림의 요소 타입을 유지하지 못합니다.
    - `toArray(IntFunction<A[]> generator)`: 지정된 타입 (`A[]`)의 배열을 반환합니다. 스트림의 요소 타입을 유지할 수 있습니다.
    
    **장점:**
    
    - **타입 안전성:** 컴파일 시점에 배열의 타입을 확인할 수 있습니다.
    - **유연성:** 다양한 타입의 배열을 생성할 수 있습니다.
    
    **핵심:**
    
    `toArray(IntFunction<A[]> generator)` 메서드는 스트림의 요소를 특정 타입의 배열로 변환해야 할 때 유용하게 사용될 수 있습니다. 특히, 생성자 참조를 사용하여 배열의 타입을 명시적으로 지정하고, 배열의 크기를 동적으로 결정할 수 있다는 장점이 있습니다.
    
- Optional<T> reduce(BinaryOperator<T> accumulator)
    
    `Optional<T> reduce(BinaryOperator<T> accumulator)` 메서드는 스트림의 요소들을 결합하여 **단 하나의 결과**를 만들어내는 최종 연산입니다. 이전 `reduce` 메서드와 유사하지만, 스트림이 비어있는 경우를 처리하기 위해 `Optional<T>`를 반환합니다.
    
    **인자:**
    
    - `BinaryOperator<T> accumulator`: 두 개의 `T` 타입 인수를 받아서 `T` 타입의 결과를 반환하는 함수형 인터페이스입니다. 이 함수는 스트림의 요소들을 누적하여 최종 결과를 만들어내는 역할을 합니다.
    
    **동작 방식:**
    
    1. 스트림이 비어있는지 확인합니다.
    2. 만약 스트림이 비어있다면, `Optional.empty()`를 반환합니다.
    3. 만약 스트림에 요소가 있다면, 스트림의 첫 번째 요소와 두 번째 요소를 `accumulator`에 적용하여 결과를 얻습니다.
    4. 그런 다음, 이전 단계의 결과와 세 번째 요소를 `accumulator`에 적용하여 결과를 얻습니다.
    5. 이 과정을 스트림의 모든 요소에 대해 반복하여 최종 결과를 얻습니다.
    6. 최종 결과를 `Optional.of(result)`로 감싸서 반환합니다.
    
    **핵심:**
    
    - 스트림이 비어있는 경우 `Optional.empty()`를 반환하여 `NullPointerException`을 방지합니다.
    - 스트림에 요소가 있는 경우, 요소들을 누적하여 단 하나의 결과를 만들어냅니다.
    
    **언제 사용해야 할까요?**
    
    - 스트림이 비어있을 가능성이 있는 경우
    - 스트림의 요소들을 결합하여 단 하나의 결과를 얻어야 하는 경우 (예: 합, 곱, 최대/최소값)
    
    **샘플 코드:**
    
    ```java
    import java.util.Arrays;
    import java.util.List;
    import java.util.Optional;
    
    public class Main {
        public static void main(String[] args) {
            List<Integer> numbers = Arrays.asList(1, 2, 3, 4, 5);
            List<Integer> emptyList = Arrays.asList();
    
            // 1. 요소가 있는 경우
            Optional<Integer> sum = numbers.stream()
                                         .reduce((a, b) -> a + b);
    
            if (sum.isPresent()) {
                System.out.println("Sum: " + sum.get()); // Sum: 15
            }
    
            // 2. 요소가 없는 경우
            Optional<Integer> emptySum = emptyList.stream()
                                                .reduce((a, b) -> a + b);
    
            if (emptySum.isPresent()) {
                System.out.println("Sum: " + emptySum.get());
            } else {
                System.out.println("Sum is empty"); // Sum is empty
            }
        }
    }
    
    ```
    
    **설명:**
    
    1. **요소가 있는 경우:** `numbers` 리스트의 모든 요소의 합을 계산합니다. `reduce` 메서드는 `Optional<Integer>`를 반환하며, `isPresent()` 메서드를 사용하여 값이 있는지 확인한 후 `get()` 메서드를 사용하여 값을 가져옵니다.
    2. **요소가 없는 경우:** `emptyList` 리스트는 비어있으므로, `reduce` 메서드는 `Optional.empty()`를 반환합니다. `isPresent()` 메서드는 `false`를 반환하므로, "Sum is empty"가 출력됩니다.
    
    **`reduce(T identity, BinaryOperator<T> accumulator)`와의 차이점:**
    
    - `reduce(BinaryOperator<T> accumulator)`: 스트림이 비어있으면 `Optional.empty()`를 반환합니다.
    - `reduce(T identity, BinaryOperator<T> accumulator)`: 스트림이 비어있으면 `identity` (초기값)를 반환합니다. `Optional`을 사용하지 않습니다.
    
    **핵심:**
    
    `Optional<T> reduce(BinaryOperator<T> accumulator)` 메서드는 스트림이 비어있을 가능성이 있는 경우 `NullPointerException`을 방지하고, 안전하게 결과를 처리할 수 있도록 해줍니다.
    
- U reduce(U identity, BiFunction<U,T,U> accumulator, BinaryOperator<U> combiner)
    
    `U reduce(U identity, BiFunction<U,T,U> accumulator, BinaryOperator<U> combiner)` 메서드는 스트림의 요소들을 결합하여 **하나의 결과**를 만들어내는 최종 연산입니다. 이 메서드는 특히 **병렬 스트림**에서 효과적으로 동작하도록 설계되었으며, 초기값, 누적 함수, 그리고 결합 함수를 인자로 받습니다.
    
    **인자:**
    
    - `U identity`: 초기값 (initial value). 스트림이 비어있을 경우 반환되는 값입니다. 또한, 누적 연산의 초기값으로 사용됩니다.
    - `BiFunction<U, T, U> accumulator`: 누적 함수. `U` 타입의 부분 결과와 `T` 타입의 스트림 요소를 받아서 `U` 타입의 새로운 부분 결과를 반환하는 함수형 인터페이스입니다.
    - `BinaryOperator<U> combiner`: 결합 함수. 두 개의 `U` 타입 부분 결과를 받아서 `U` 타입의 최종 결과를 반환하는 함수형 인터페이스입니다. 이 함수는 병렬 처리 과정에서 각 스레드의 부분 결과를 결합하는 데 사용됩니다.
    
    **동작 방식:**
    
    1. 스트림을 여러 개의 하위 스트림으로 분할합니다 (병렬 스트림인 경우).
    2. 각 하위 스트림에 대해 `accumulator` 함수를 적용하여 부분 결과를 계산합니다. `accumulator` 함수는 `identity` 값을 초기값으로 사용하여 스트림의 요소들을 누적합니다.
    3. 각 하위 스트림의 부분 결과를 `combiner` 함수를 사용하여 결합하여 최종 결과를 생성합니다.
    4. 직렬 스트림의 경우, `combiner`는 호출되지 않습니다.
    
    **핵심:**
    
    - `identity`: 스트림이 비어있을 경우 반환되는 값이며, 누적 연산의 초기값으로 사용됩니다.
    - `accumulator`: 각 스레드에서 부분 결과를 계산하는 데 사용됩니다.
    - `combiner`: 병렬 스트림에서 각 스레드의 부분 결과를 결합하여 최종 결과를 생성하는 데 사용됩니다.
    - 병렬 스트림에서 성능 향상을 제공합니다.
    
    **언제 사용해야 할까요?**
    
    - 병렬 스트림에서 요소들을 결합하여 단 하나의 결과를 얻어야 하는 경우
    - 초기값이 필요한 경우
    - 부분 결과를 결합하는 로직이 필요한 경우
    
    **샘플 코드:**
    
    ```java
    import java.util.Arrays;
    import java.util.List;
    
    public class Main {
        public static void main(String[] args) {
            List<Integer> numbers = Arrays.asList(1, 2, 3, 4, 5);
    
            // 병렬 스트림에서 모든 요소의 합 계산
            Integer sum = numbers.parallelStream()
                                 .reduce(0,  // identity: 초기값
                                         (a, b) -> a + b,  // accumulator: 부분 합 계산
                                         (a, b) -> a + b);  // combiner: 부분 합들을 결합
    
            System.out.println("Sum: " + sum); // Sum: 15
        }
    }
    
    ```
    
    **설명:**
    
    1. `numbers.parallelStream()`: 병렬 스트림을 생성합니다.
    2. `reduce(0, (a, b) -> a + b, (a, b) -> a + b)`:
        - `0`: 초기값 (스트림이 비어있을 경우 반환되는 값)
        - `(a, b) -> a + b`: `accumulator` 함수. 부분 합 `a`에 스트림 요소 `b`를 더하여 새로운 부분 합을 계산합니다.
        - `(a, b) -> a + b`: `combiner` 함수. 두 개의 부분 합 `a`와 `b`를 더하여 최종 합계를 계산합니다.
    
    **직렬 스트림 vs 병렬 스트림:**
    
    - 직렬 스트림: 요소들이 순차적으로 처리됩니다. `combiner`는 호출되지 않습니다.
    - 병렬 스트림: 요소들이 여러 스레드에서 병렬적으로 처리됩니다. `combiner`는 각 스레드의 부분 결과를 결합하는 데 사용됩니다.
    
    **주의 사항:**
    
    - `identity` 값은 `accumulator` 함수의 입력 타입과 반환 타입, 그리고 `combiner` 함수의 입력 타입과 반환 타입과 일치해야 합니다.
    - `accumulator` 함수는 결합 법칙을 만족해야 합니다 (예: `(a + b) + c == a + (b + c)`).
    - `combiner` 함수는 결합 법칙을 만족해야 합니다.
    
    **핵심:**
    
    `U reduce(U identity, BiFunction<U,T,U> accumulator, BinaryOperator<U> combiner)` 메서드는 병렬 스트림에서 성능 향상을 제공하며, 초기값과 부분 결과 결합 로직을 명시적으로 지정할 수 있도록 해줍니다.
    
    좋습니다. `accumulator`와 `combiner`가 서로 다른 역할을 하는 `reduce` 예제를 보여드리겠습니다. 이 예제는 문자열 리스트에서 각 문자열의 길이를 계산하고, 모든 길이의 합을 구하는 동시에 문자열들을 쉼표로 연결된 문자열로 만드는 예제입니다.
    
    ```java
    import java.util.Arrays;
    import java.util.List;
    
    public class Main {
        public static void main(String[] args) {
            List<String> words = Arrays.asList("hello", "world", "java");
    
            // 1. 문자열 길이의 합과 쉼표로 연결된 문자열을 동시에 계산
            LengthAndConcat result = words.parallelStream()
                    .reduce(new LengthAndConcat(0, ""),  // identity: 초기값 (길이: 0, 연결된 문자열: "")
                            (acc, word) -> {  // accumulator: 부분 길이 합산 및 문자열 연결
                                System.out.println("Accumulator called. acc: " + acc + ", word: " + word);
                                return new LengthAndConcat(acc.length + word.length(), acc.combined + (acc.combined.isEmpty() ? "" : ",") + word);
                            },
                            (lc1, lc2) -> {  // combiner: 부분 결과 결합
                                System.out.println("Combiner called. lc1: " + lc1 + ", lc2: " + lc2);
                                return new LengthAndConcat(lc1.length + lc2.length(), lc1.combined + (lc1.combined.isEmpty() || lc2.combined.isEmpty() ? "" : ",") + lc2.combined);
                            });
    
            System.out.println("Final Result: Length = " + result.length + ", Combined String = " + result.combined);
        }
    
        // 2. 부분 결과 저장을 위한 클래스
        static class LengthAndConcat {
            int length;
            String combined;
    
            public LengthAndConcat(int length, String combined) {
                this.length = length;
                this.combined = combined;
            }
    
            @Override
            public String toString() {
                return "LengthAndConcat{" +
                        "length=" + length +
                        ", combined='" + combined + '\\'' +
                        '}';
            }
        }
    }
    
    ```
    
    **설명:**
    
    1. **`LengthAndConcat` 클래스:**
        - `length`: 문자열 길이의 합을 저장합니다.
        - `combined`: 쉼표로 연결된 문자열을 저장합니다.
    2. **`reduce(new LengthAndConcat(0, ""), (acc, word) -> { ... }, (lc1, lc2) -> { ... })`:**
        - `new LengthAndConcat(0, "")`: 초기값. 길이 합은 0, 연결된 문자열은 빈 문자열로 시작합니다.
        - `accumulator`:
            - 현재까지의 길이 합 (`acc.length`)에 현재 단어의 길이 (`word.length()`)를 더합니다.
            - 현재까지의 연결된 문자열 (`acc.combined`)에 쉼표와 현재 단어 (`word`)를 추가합니다.
        - `combiner`:
            - 두 개의 `LengthAndConcat` 객체 (`lc1`, `lc2`)를 받아서 길이 합을 더하고, 문자열을 연결합니다.
            - `lc1.combined.isEmpty() || lc2.combined.isEmpty() ? "" : ","`: 둘 중 하나라도 비어있으면 쉼표를 추가하지 않습니다.
    
    **`accumulator`와 `combiner`의 차이점:**
    
    - `accumulator`는 각 단어를 처리하면서 부분 길이 합과 연결된 문자열을 누적합니다. 즉, **입력 요소(words)를 직접 사용**하여 계산합니다.
    - `combiner`는 병렬 스트림에서 각 스레드가 계산한 부분 결과를 합치는 역할을 합니다. 즉, **입력 요소(words)를 직접 사용하지 않고, `accumulator`의 결과를 결합**합니다.
    
    **출력 결과 (병렬 스트림):**
    
    ```
    Accumulator called. acc: LengthAndConcat{length=0, combined=''}, word: java
    Accumulator called. acc: LengthAndConcat{length=0, combined=''}, word: world
    Accumulator called. acc: LengthAndConcat{length=0, combined=''}, word: hello
    Combiner called. lc1: LengthAndConcat{length=5, combined='hello'}, lc2: LengthAndConcat{length=5, combined='world'}
    Combiner called. lc1: LengthAndConcat{length=10, combined='hello,world'}, lc2: LengthAndConcat{length=4, combined='java'}
    Final Result: Length = 14, Combined String = hello,world,java
    
    ```
    
    **핵심:**
    
    - `accumulator`는 스트림의 각 요소를 사용하여 부분 결과를 계산합니다.
    - `combiner`는 각 부분 결과를 결합하여 최종 결과를 생성합니다.
    - 이 예제에서는 `accumulator`와 `combiner`가 서로 다른 로직을 수행하여, 길이 합과 연결된 문자열을 동시에 계산합니다.
    - 병렬 스트림을 사용하면 여러 스레드에서 `accumulator`가 동시에 실행되고, `combiner`가 이 결과를 결합합니다.
    
    이 예제가 `accumulator`와 `combiner`의 역할을 이해하는 데 도움이 되었기를 바랍니다.
    
- R collect(Supplier<R> supplier, BiConsumer<R, T> accumulator, BiConsumer<R, R> combiner)
    
    `R collect(Supplier<R> supplier, BiConsumer<R, T> accumulator, BiConsumer<R, R> combiner)` 메서드는 스트림의 요소들을 수집하여 최종 결과로 변환하는 최종 연산입니다. 이 메서드는 `Collector` 인터페이스를 사용하는 `collect` 메서드와 달리, 수집 과정을 더 세밀하게 제어할 수 있도록 해줍니다.
    
    **인자:**
    
    - `Supplier<R> supplier`: 새로운 결과 컨테이너(result container)를 생성하는 함수형 인터페이스입니다. 이 컨테이너는 수집 과정에서 중간 결과를 저장하는 데 사용됩니다. 즉, **결과를 담을 객체를 생성**하는 역할을 합니다.
    - `BiConsumer<R, T> accumulator`: 스트림의 요소를 결과 컨테이너에 누적하는 함수형 인터페이스입니다. 이 함수는 결과 컨테이너와 스트림의 요소를 받아서 컨테이너의 상태를 변경합니다. 즉, **스트림의 각 요소를 결과 컨테이너에 추가**하는 역할을 합니다.
    - `BiConsumer<R, R> combiner`: 두 개의 결과 컨테이너를 결합하는 함수형 인터페이스입니다. 이 함수는 병렬 스트림 처리 과정에서 각 스레드의 부분 결과를 결합하여 최종 결과를 생성하는 데 사용됩니다. 즉, **병렬 처리된 부분 결과를 합치는** 역할을 합니다.
    
    **동작 방식:**
    
    1. `supplier` 함수를 사용하여 초기 결과 컨테이너를 생성합니다.
    2. 스트림의 각 요소에 대해 `accumulator` 함수를 적용하여 결과 컨테이너를 업데이트합니다.
    3. 병렬 스트림인 경우, 스트림이 여러 개의 하위 스트림으로 분할되고, 각 하위 스트림에 대해 1, 2번 과정을 수행합니다.
    4. `combiner` 함수를 사용하여 각 하위 스트림의 결과 컨테이너를 결합하여 최종 결과 컨테이너를 생성합니다.
    5. 최종 결과 컨테이너를 반환합니다.
    
    **핵심:**
    
    - `supplier`: 결과를 담을 객체를 생성합니다.
    - `accumulator`: 스트림의 각 요소를 결과 컨테이너에 추가합니다.
    - `combiner`: 병렬 처리된 부분 결과를 합칩니다.
    
    **언제 사용해야 할까요?**
    
    - `Collector` 인터페이스에서 제공하는 수집 방법 외에 **더욱 세밀한 제어**가 필요한 경우
    - **특정 자료구조**에 스트림의 요소들을 수집해야 하는 경우
    - **병렬 스트림**에서 안전하게 동작하는 사용자 정의 수집 로직을 구현해야 하는 경우
    
    **샘플 코드:**
    
    ```java
    import java.util.ArrayList;
    import java.util.Arrays;
    import java.util.List;
    import java.util.stream.Collectors;
    
    public class Main {
        public static void main(String[] args) {
            List<String> names = Arrays.asList("Alice", "Bob", "Charlie");
    
            // 1. ArrayList에 수집 (직렬 스트림)
            ArrayList<String> nameList = names.stream()
                    .collect(
                            ArrayList::new,          // supplier: 새로운 ArrayList 생성
                            ArrayList::add,          // accumulator: ArrayList에 요소 추가
                            ArrayList::addAll        // combiner: 두 ArrayList 합치기 (병렬 스트림에서만 사용)
                    );
    
            System.out.println("Name List: " + nameList); // Name List: [Alice, Bob, Charlie]
    
            // 2. StringBuilder에 수집 (병렬 스트림)
            StringBuilder nameString = names.parallelStream()
                    .collect(
                            StringBuilder::new,      // supplier: 새로운 StringBuilder 생성
                            StringBuilder::append,   // accumulator: StringBuilder에 요소 추가
                            StringBuilder::append    // combiner: 두 StringBuilder 합치기
                    );
    
            System.out.println("Name String: " + nameString); // Name String: AliceBobCharlie
        }
    }
    
    ```
    
    **설명:**
    
    1. **`ArrayList`에 수집:**
        - `ArrayList::new`: 새로운 `ArrayList` 객체를 생성합니다.
        - `ArrayList::add`: 스트림의 각 요소를 `ArrayList`에 추가합니다.
        - `ArrayList::addAll`: 두 개의 `ArrayList`를 합칩니다 (병렬 스트림에서만 사용).
    2. **`StringBuilder`에 수집:**
        - `StringBuilder::new`: 새로운 `StringBuilder` 객체를 생성합니다.
        - `StringBuilder::append`: 스트림의 각 요소를 `StringBuilder`에 추가합니다.
        - `StringBuilder::append`: 두 개의 `StringBuilder`를 합칩니다.
    
    **`Collector` 인터페이스 vs `collect(Supplier, BiConsumer, BiConsumer)`:**
    
    - `Collector` 인터페이스: 미리 정의된 수집 로직을 사용하거나, 복잡한 사용자 정의 수집 로직을 캡슐화할 수 있습니다.
    - `collect(Supplier, BiConsumer, BiConsumer)`: 수집 과정을 더 세밀하게 제어할 수 있지만, 코드량이 증가하고 가독성이 떨어질 수 있습니다.
    
    **핵심:**
    
    `R collect(Supplier<R> supplier, BiConsumer<R, T> accumulator, BiConsumer<R, R> combiner)` 메서드는 스트림의 요소들을 수집하는 과정을 세밀하게 제어할 수 있도록 해줍니다. 특히, 특정 자료구조에 스트림의 요소들을 수집해야 하거나, 병렬 스트림에서 안전하게 동작하는 사용자 정의 수집 로직을 구현해야 하는 경우 유용하게 사용될 수 있습니다.
    
- 지연된 연산
    
    "지연된 연산(Lazy Evaluation)" 또는 "지연 연산(Deferred Execution)"은 최종 연산이 수행되기 전까지는 중간 연산이 실제로 실행되지 않는다는 개념입니다. 스트림 API의 핵심적인 특징 중 하나이며, 효율적인 데이터 처리를 가능하게 합니다.
    
    **지연된 연산의 의미:**
    
    - **중간 연산:** `filter`, `map`, `sorted`, `peek` 등과 같이 스트림을 반환하는 연산입니다. 이 연산들은 스트림 파이프라인을 구성하는 단계이며, **실제로 데이터를 처리하지 않습니다.** 단지 어떤 연산을 수행해야 하는지 정의할 뿐입니다.
    - **최종 연산:** `forEach`, `collect`, `reduce`, `count` 등과 같이 스트림을 소비하고 결과를 반환하는 연산입니다. **이 연산이 호출되는 시점에 비로소 중간 연산들이 실행됩니다.**
    
    **동작 방식:**
    
    1. **스트림 파이프라인 구성:** 중간 연산들을 연결하여 스트림 파이프라인을 구성합니다.
    2. **최종 연산 호출:** 최종 연산을 호출하면, 스트림 API는 파이프라인을 따라 데이터를 처리하기 시작합니다.
    3. **데이터 처리:** 스트림 API는 데이터를 필요한 만큼만 처리합니다. 예를 들어, `limit(10)` 중간 연산이 있다면, 스트림 API는 10개의 요소만 처리하고 더 이상 데이터를 처리하지 않습니다.
    4. **결과 반환:** 최종 연산은 처리된 데이터를 기반으로 결과를 반환합니다.
    
    **지연된 연산이 중요한 이유:**
    
    - **성능 향상:** 불필요한 연산을 줄여 성능을 향상시킵니다. 스트림의 모든 요소를 처리하는 대신, 필요한 요소만 처리하므로 처리 시간을 단축할 수 있습니다.
    - **메모리 효율성:** 모든 데이터를 메모리에 로드하지 않고, 필요한 데이터만 처리하므로 메모리 사용량을 줄일 수 있습니다.
    - **코드 가독성 향상:** 복잡한 데이터 처리 로직을 스트림 파이프라인으로 표현하여 코드 가독성을 높일 수 있습니다.
    
    **예시:**
    
    ```java
    import java.util.Arrays;
    import java.util.List;
    
    public class Main {
        public static void main(String[] args) {
            List<String> names = Arrays.asList("Alice", "Bob", "Charlie", "David", "Eve");
    
            // 스트림 파이프라인 구성 (중간 연산)
            names.stream()
                 .filter(name -> {
                     System.out.println("Filtering: " + name); // 필터링 과정 출력
                     return name.startsWith("A");
                 })
                 .map(name -> {
                     System.out.println("Mapping: " + name); // 매핑 과정 출력
                     return name.toUpperCase();
                 })
                 .limit(1); // 최대 1개의 요소만 처리
    
            System.out.println("최종 연산 전");
        }
    }
    
    ```
    
    이 코드를 실행하면 "최종 연산 전"만 출력됩니다. `filter`와 `map`은 중간 연산이므로, 최종 연산이 호출되기 전까지는 실행되지 않습니다.
    
    이제 최종 연산을 추가해 보겠습니다.
    
    ```java
    import java.util.Arrays;
    import java.util.List;
    
    public class Main {
        public static void main(String[] args) {
            List<String> names = Arrays.asList("Alice", "Bob", "Charlie", "David", "Eve");
    
            // 스트림 파이프라인 구성 (중간 연산) 및 최종 연산
            names.stream()
                 .filter(name -> {
                     System.out.println("Filtering: " + name); // 필터링 과정 출력
                     return name.startsWith("A");
                 })
                 .map(name -> {
                     System.out.println("Mapping: " + name); // 매핑 과정 출력
                     return name.toUpperCase();
                 })
                 .limit(1) // 최대 1개의 요소만 처리
                 .forEach(name -> System.out.println("Final: " + name)); // 최종 연산: 출력
        }
    }
    
    ```
    
    이제 코드를 실행하면 다음과 같은 결과가 출력됩니다.
    
    ```
    Filtering: Alice
    Mapping: Alice
    Final: ALICE
    
    ```
    
    **설명:**
    
    1. `stream()`: 스트림 생성
    2. `filter()`: "Alice"만 필터링됩니다. "Bob", "Charlie", "David", "Eve"는 필터링 조건을 만족하지 않으므로 더 이상 처리되지 않습니다.
    3. `map()`: "Alice"가 "ALICE"로 매핑됩니다.
    4. `limit(1)`: 최대 1개의 요소만 처리하므로, "ALICE" 이후의 요소는 처리되지 않습니다.
    5. `forEach()`: "ALICE"를 출력합니다.
    
    **핵심:**
    
    - 중간 연산은 최종 연산이 호출되기 전까지는 실행되지 않습니다.
    - 스트림 API는 데이터를 필요한 만큼만 처리하여 성능과 메모리 효율성을 높입니다.
    - 지연된 연산은 스트림 API의 핵심적인 특징입니다.
    
    이 예시를 통해 지연된 연산의 개념을 이해하는 데 도움이 되었기를 바랍니다.
    

특정 범위의 정수

```java
IntStream range(int begin, int end)
IntStream rangeClosed(int begin, int end) // end 포함여부
```

임의의 수

```java
IntStream ints()

IntStream intStream = new Random().ints(); // 무한스트림
intStream.limit(5).forEach(System.out::println);

ints(int begin, int end) // 지정된 범위의 난수 발생
int(long streamSize, int begin, int end)
```

- ints()? Random()? method참조
    
    **1. `IntStream ints()` 메서드:**
    
    - **`ints()` 메서드는 `java.util.Random` 클래스의 인스턴스 메서드입니다.** 이 메서드는 **무한 스트림(infinite stream)**의 `IntStream`을 반환합니다.
    - **각 요소는 `Random` 객체를 사용하여 생성된 의사 난수입니다.**
    - 따라서, `Random().ints()`를 호출해야 `Random` 클래스의 인스턴스를 사용하여 난수를 생성하고, 그 난수들을 요소로 가지는 `IntStream`을 얻을 수 있습니다.
    - `IntStream intStream = new Random().ints();`는 `intStream` 변수에 무한한 난수 스트림을 할당합니다.
    
    **2. `intStream.limit(5).forEach(System.out::println);`:**
    
    - **`limit(5)`:** `intStream`에서 처음 5개의 요소만 선택하여 새로운 스트림을 생성합니다. 이렇게 하지 않으면 무한 스트림이기 때문에 `forEach`가 종료되지 않습니다.
    - **`forEach(System.out::println)`:** 스트림의 각 요소에 대해 `System.out.println()` 메서드를 호출합니다. 여기서 `System.out::println`은 메서드 참조입니다.
    
    **3. `System.out::println` 메서드 참조:**
    
    - **`System.out`은 `java.lang.System` 클래스의 `public static final PrintStream out` 필드입니다.** 즉, `System`은 클래스이고, `out`은 해당 클래스의 **static 멤버 변수**입니다. `out`은 `PrintStream` 클래스의 **인스턴스**입니다.
    - **`System.out::println`은 `PrintStream` 클래스의 `println` 인스턴스 메서드를 참조하는 메서드 참조입니다.** 여기서 `System.out`은 특정 객체 (PrintStream 인스턴스)를 나타냅니다.
    - 따라서, `System.out::println`은 "System.out이라는 PrintStream 객체의 println 메서드를 호출하라"는 의미입니다.
    
    **정리:**
    
    - `ints()` 메서드는 `Random` 클래스의 인스턴스 메서드이며, 의사 난수를 요소로 가지는 무한 스트림을 반환합니다.
    - `Random().ints()`를 호출해야 `Random` 객체를 사용하여 난수를 생성할 수 있습니다.
    - `System.out`은 `System` 클래스의 static 멤버 변수이며, `PrintStream` 클래스의 인스턴스입니다.
    - `System.out::println`은 `PrintStream` 클래스의 `println` 인스턴스 메서드를 참조하는 메서드 참조입니다.
    
    이 설명이 궁금증을 해결하는 데 도움이 되었기를 바랍니다!
    

iterate(), generate() → 람다식을 매개변수로 받아서 무한 스트림 생성

```java
static <T> stream<T> iterate(T seed, UnaryOperator<T> f) // seed부터 f에 의해 계산된 결과를 다시 seed값으로 계산 반복
static <T> stream<T> generate(Supplier<T> s) // 이전 결과를 이용해서 다음 요소를 계산하지 않음
```

파일

```java
Stream<Path> Files.list(Path dir) // 지정된 dir에 있는 파일의 목록을 소스로 하는 스트림 생성해서 반환
```

두 스트림 연결

```java
Stream<String> strs1 = Stream.of(str1);
Stream<String> strs2 = Stream.of(str2);
Stream<String> strs3 = Stream.concat(strs1, strs2);
```

- of
    
    `Stream.of()` 메서드는 **특정 요소들을 포함하는 스트림을 생성하는 static 팩토리 메서드**입니다. 즉, 배열이나 개별 요소들을 스트림으로 쉽게 변환할 수 있도록 해줍니다.
    
    **`Stream.of()` 메서드의 다양한 형태:**
    
    1. **`Stream.of(T t)`**: 단일 요소 `t`를 포함하는 스트림을 생성합니다.
        
        ```java
        Stream<String> strStream = Stream.of("Hello"); // "Hello" 요소를 가진 스트림 생성
        
        ```
        
    2. **`Stream.of(T... values)`**: 가변 인자 목록 `values`를 포함하는 스트림을 생성합니다.
        
        ```java
        Stream<String> strStream = Stream.of("Hello", "World", "Java"); // "Hello", "World", "Java" 요소를 가진 스트림 생성
        
        ```
        
    3. **`Stream.of(T[] array)` (Java 9 이상)**: 배열 `array`의 요소들을 포함하는 스트림을 생성합니다.
        
        ```java
        String[] strArray = {"Hello", "World", "Java"};
        Stream<String> strStream = Arrays.stream(strArray); // "Hello", "World", "Java" 요소를 가진 스트림 생성
        
        ```
        
    
    **제공된 코드 예시 분석:**
    
    ```java
    Stream<String> strs1 = Stream.of(str1);
    Stream<String> strs2 = Stream.of(str2);
    Stream<String> strs3 = Stream.concat(strs1, strs2);
    
    ```
    
    1. **`Stream<String> strs1 = Stream.of(str1);`**:
        - `Stream.of(str1)`: `str1`이라는 `String` 변수가 가지고 있는 **단일 요소**를 포함하는 `Stream<String>`을 생성합니다. 즉, `str1`이 "Hello"라면, `strs1`은 "Hello"라는 요소를 가진 스트림이 됩니다.
    2. **`Stream<String> strs2 = Stream.of(str2);`**:
        - `Stream.of(str2)`: `str2`라는 `String` 변수가 가지고 있는 **단일 요소**를 포함하는 `Stream<String>`을 생성합니다. 즉, `str2`가 "World"라면, `strs2`는 "World"라는 요소를 가진 스트림이 됩니다.
    3. **`Stream<String> strs3 = Stream.concat(strs1, strs2);`**:
        - `Stream.concat(strs1, strs2)`: `strs1` 스트림과 `strs2` 스트림을 **연결**하여 새로운 스트림 `strs3`를 생성합니다. 즉, `strs1`이 "Hello"를 가지고, `strs2`가 "World"를 가진다면, `strs3`는 "Hello", "World" 두 개의 요소를 순서대로 가지는 스트림이 됩니다.
    
    **`Stream.of()`를 사용하는 이유:**
    
    - **간편한 스트림 생성:** 컬렉션이나 배열 없이도 간단하게 스트림을 생성할 수 있습니다.
    - **명확성:** 코드 의도를 명확하게 드러낼 수 있습니다. 예를 들어, 특정 요소들을 스트림으로 처리하겠다는 것을 명시적으로 나타낼 수 있습니다.
    - **유연성:** 다양한 타입의 요소를 스트림으로 처리할 수 있습니다.
    
    **`Stream.of()` vs `Arrays.stream()` (배열에서 스트림 생성):**
    
    - `Stream.of(array)`: 배열 자체를 요소로 가지는 스트림을 생성합니다 (Java 8에서는 배열을 래핑). Java 9부터는 배열의 요소들을 스트림으로 생성합니다.
    - `Arrays.stream(array)`: 배열의 요소들을 포함하는 스트림을 생성합니다.
    
    **핵심:**
    
    `Stream.of()` 메서드는 개별 요소 또는 배열을 사용하여 스트림을 간편하게 생성할 수 있도록 해주는 static 팩토리 메서드입니다. `Stream.concat()`과 함께 사용하면 여러 스트림을 연결하여 새로운 스트림을 만들 수 있습니다.
    

sorted()

```java
Stream<T> sorted()
Stream<T> sorted(Comparator<? super T> comparator)
```

- 지정된 Comparator로 스트림을 정렬 (미지정시 스트림 요소의 기본 정렬 기준으로 정렬 → 요소가 Comparable 구현해야함)
- Stream(String> strStream = Stream.of(”dd”, “aaa”, “CC”, “cc”, “b”); 정렬 예시

| strStream.sorted()
strStream.sorted(Comparator.naturalOrder())
strStream.sorted((s1, s2) → s1.compareTo(s2))
strStream.sorted(String::compareTo) | CCaaabccdd |
| --- | --- |
| strStream.sorted(Comparator.reverseOrder())
strStream.sorted(Comparator.<String>naturalOrder().reversed()) | ddccbaaaCC |
| strStream.sorted(String.CASE_INSENSITIVE_ORDER) | aaabCCccdd |
| strStream.sorted(Comparator.comparing(String::length))
strStream.sorted(Comparator.compringInt(String::length)) // 박싱비용x | bddCCccaaa |
- Comparable, Comparator
    
    **1. `Comparable` 인터페이스**
    
    - **소속:** `java.lang` 패키지
    - **역할:** 클래스 자체가 **자연적인(natural) 정렬 순서를 가지도록** 정의합니다.
    - **클래스:** `Comparable` 인터페이스는 **클래스**가 구현합니다. (예: `String`, `Integer`, `Date` 등)
    - **메서드:**
        - `int compareTo(T o)`:
            - **소속:** `Comparable` 인터페이스에 정의된 **인스턴스 메서드**
            - **역할:** 현재 객체와 다른 객체 `o`를 비교하여 정수 값을 반환합니다.
            - **사용법:** `obj1.compareTo(obj2)` 형태로 사용합니다. (`obj1`은 `Comparable`을 구현한 클래스의 인스턴스)
            - **반환값:**
                - 음수: 현재 객체가 `o`보다 작음
                - 0: 현재 객체가 `o`와 같음
                - 양수: 현재 객체가 `o`보다 큼
    - **핵심:** `Comparable` 인터페이스를 구현한 클래스는 `compareTo` 메서드를 통해 자신의 객체와 다른 객체를 비교하는 방법을 **직접 정의**합니다.
    
    **2. `Comparator` 인터페이스**
    
    - **소속:** `java.util` 패키지
    - **역할:** **클래스의 객체들을 비교하는 방법을 정의**합니다. 클래스 자체를 수정하지 않고도 다양한 정렬 기준을 제공할 수 있습니다.
    - **클래스:** `Comparator` 인터페이스는 별도의 클래스(또는 람다 표현식, 메서드 참조)로 **구현**됩니다. 즉, 정렬 대상 클래스가 아닌 **별도의 비교기(Comparator)**를 만드는 것입니다.
    - **메서드:**
        - `int compare(T o1, T o2)`:
            - **소속:** `Comparator` 인터페이스에 정의된 **인스턴스 메서드**
            - **역할:** 두 객체 `o1`과 `o2`를 비교하여 정수 값을 반환합니다.
            - **사용법:** `comparator.compare(obj1, obj2)` 형태로 사용합니다. (`comparator`는 `Comparator` 인터페이스를 구현한 객체)
            - **반환값:**
                - 음수: `o1`이 `o2`보다 작음
                - 0: `o1`이 `o2`와 같음
                - 양수: `o1`이 `o2`보다 큼
        - `Comparator<T> reversed()`:
            - **소속:** `Comparator` 인터페이스에 정의된 **디폴트 메서드**
            - **역할:** 현재 `Comparator`의 비교 순서를 뒤집은 새로운 `Comparator`를 반환합니다.
            - **사용법:** `comparator.reversed()` 형태로 사용합니다.
        - `Comparator<T> thenComparing(Comparator<? super T> other)`:
            - **소속:** `Comparator` 인터페이스에 정의된 **디폴트 메서드**
            - **역할:** 현재 `Comparator`로 비교한 결과가 같을 경우, `other` `Comparator`를 사용하여 추가적인 비교를 수행하는 새로운 `Comparator`를 반환합니다.
            - **사용법:** `comparator1.thenComparing(comparator2)` 형태로 사용합니다.
        - `comparing(...)`, `comparingInt(...)`, `comparingLong(...)`, `comparingDouble(...)`: (아래 `comparing` 메서드 참고)
    - **핵심:** `Comparator` 인터페이스를 구현한 클래스는 `compare` 메서드를 통해 두 객체를 비교하는 방법을 **별도로 정의**합니다. 이를 통해 클래스 자체를 수정하지 않고도 다양한 정렬 기준을 적용할 수 있습니다.
    
    **3. `comparing` 메서드**
    
    - **소속:** `Comparator` 인터페이스에 정의된 **static 메서드**
    - **역할:** 특정 속성(키)을 기준으로 객체를 비교하는 `Comparator`를 **쉽게 생성**할 수 있도록 해줍니다.
    - **메서드 종류:**
        - `static <T, U extends Comparable<? super U>> Comparator<T> comparing(Function<? super T, ? extends U> keyExtractor)`:
            - 객체에서 비교할 값을 추출하는 `Function`을 인자로 받습니다.
            - 추출된 값은 `Comparable` 인터페이스를 구현해야 합니다.
        - `static <T> Comparator<T> comparingInt(ToIntFunction<? super T> keyExtractor)`:
            - 객체에서 int 값을 추출하는 `ToIntFunction`을 인자로 받습니다.
            - `Integer` 객체 대신 `int` 기본형을 사용하여 성능을 향상시킵니다.
        - `static <T> Comparator<T> comparingLong(ToLongFunction<? super T> keyExtractor)`:
            - 객체에서 long 값을 추출하는 `ToLongFunction`을 인자로 받습니다.
            - `Long` 객체 대신 `long` 기본형을 사용하여 성능을 향상시킵니다.
        - `static <T> Comparator<T> comparingDouble(ToDoubleFunction<? super T> keyExtractor)`:
            - 객체에서 double 값을 추출하는 `ToDoubleFunction`을 인자로 받습니다.
            - `Double` 객체 대신 `double` 기본형을 사용하여 성능을 향상시킵니다.
    - **사용법:** `Comparator.comparing(클래스::getter)` 형태로 사용합니다.
    
    **요약:**
    
    | 구분 | `Comparable` | `Comparator` | `comparing` (static 메서드) |
    | --- | --- | --- | --- |
    | 소속 | `java.lang` 패키지 | `java.util` 패키지 | `java.util.Comparator` 인터페이스 |
    | 역할 | 클래스 자체의 자연스러운 정렬 순서 정의 | 객체 비교 방법 정의 (다양한 정렬 기준 제공) | 특정 속성을 기준으로 객체를 비교하는 `Comparator`를 쉽게 생성 |
    | 클래스 | 클래스 | 별도의 클래스 (또는 람다 표현식, 메서드 참조) | (메서드이므로 클래스에 직접 소속되지 않음) |
    | 메서드 | `compareTo(T o)` (인스턴스 메서드) | `compare(T o1, T o2)` (인스턴스 메서드), `reversed()` (디폴트 메서드), `thenComparing(Comparator<? super T> other)` (디폴트 메서드) | `comparing(Function keyExtractor)` (static 메서드), `comparingInt(ToIntFunction keyExtractor)` (static 메서드) 등 |
    | 사용 시점 | 클래스의 객체를 기본 정렬 방식으로 정렬할 때 | 다양한 정렬 기준을 적용해야 할 때, 클래스 수정 없이 정렬 방식을 변경하고 싶을 때 | 특정 속성을 기준으로 정렬하는 `Comparator`를 간결하게 생성하고 싶을 때 |
    | 핵심 | 클래스가 자신의 객체를 비교하는 방법을 직접 정의 | 별도의 비교기를 통해 객체 비교 방법을 정의 | 특정 속성을 추출하여 비교하는 `Comparator`를 쉽게 생성 |

```java
students.sorted(Comparator.comparing(Student::getClassNumber)
	.thenComparing(Comparator.naturalOrder())
	.forEach(System.out::println);
```

map()

- 스트림 요소에서 원하는 필드 혹은 원하는 형태로 변환할 때 사용

```java
Stream<R> map(Function<? super T, ? extends R> mapper)
```

peek()

- 연산과 연산사이의 결과를 알고싶을 떄 사용

```java
fileStream.map(File::getName)
	.filter(s -> s.indexOf('.') != -1)
	.peek(s -> System.out.printf("filename = %s%n",s))
	.forEach(System.out::println);
```

IntStream 등 기본형 스트림에서는

sum(), average(), max(), min() 을 제공함 → intStream.sum();

```java
IntSummaryStatistics stat = scoreStream.summaryStatistics();
long totalCount = stat.getCount();
long totalScore = stat.getSum();
double avgScore = stat.getAverage();
int minScore = stat.getMin();
int maxScore = stat.getMax();
```

flatMap()

- Stream<T[]> 를 Stream<T>로 변환

```java
Stream<String[]> strArrStream = Stream.of(
	new String[] {"abc", "def", "ghi" },
	new String[] {"ABC", "DEF", "GHI" }
	);
	
Stream<Stream<String>> mapStrStream = strArrStream.map(Arrays::stream);
Stream<String> flatMapStream = strArrStream.flatMap(Arrays::stream);
```

Optional<T>

- null 체크를 매번 if 문으로 하는 과정을 건너 뛰어도 NPE 발생하지 않는 안전한 코드 작성 가능
- Optional 객체 생성
    
    ```java
    String str = "abc";
    Optional<String> optVal = Optional.of(str);
    Optional<String> optVal = Optional.of("abc");
    Optional<String> optVal = Optional.of(new String("abc"));
    
    // 참조변수 null 가능성 있는 경우
    Optional<String> optVal = Optional.ofNullable(str);
    ```
    
- Optional 값 가져오기
    
    ```java
    Optional<String> optVal = Optional.of("abc");
    String str1 = optVal.get(); // null 일때는 예외 발생
    String str2 = optVal.orElse(T other); // null 일때는 other 값을 반환
    String str3 = optVal.orElseGet(Supplier<? extends T> supplier); // 값이 없으면 supplier가 제공하는 값 반환
    String str4 = optVal.orElseThrow(Supplier<? extends X> exceptionSupplier); // 값이 없으면 예외 발생시킴
    
    ```
    
    isPresent() → Optional<T> 값이 null이면 false, 아니면 true
    
    ifPresent(Consumer<T> block) → 값이 있으면 주어진 람다식 실행, 없으면 아무일도 하지 않음
    
    IntStream → OptionalInt 기본형 Optional 존재 int getAsInt()
    
    OptionalInt 클래스에는 boolean isPresent, int value 변수가 정의되어있음, 기본값은 0이지만 null 인지 0인지 isPresent로 구별 가능
    

최종연산

- forEach()
    
    ```java
    void forEach(Consumer<? super T> action) // 스트림의 요소를 소모하는 최종 연산
    ```
    
- allMatch(), anyMatch(), noneMatch()
    - 매개변수로 Predicate 요구, 연산결과를 boolean으로 반환
    - `boolean allMatch(Predicate<? super T> predicate)`
- findFirst(), findAny()
    - 병렬 Stream의 경우 findFirst() 대신 findAny()를 사용해야 함 (성능 이슈)
    - Optional<T> 반환, 없는경우 Optional.empty() 반환
- count(), sum(), average(), max(), min() - 통계
- reduce()
    - Optional<T> reduce(BinaryOperator<T> accumulator)
        - 초기값 없으므로 Optional<T>
    - T reduce(T identity, BinaryOperator<T> accumulator)
        - 초기값이 있으므로 T
    - U reduce(U identity, BiFunction<U, T, U> accumulator, BinaryOperator<U> combiner)
        - combiner는 병렬처리시 처리된 결과를 합칠때 사용하기 위한 것
- collect()
    - collect() - 스트림의 최종 연산, 매개변수로 컬렉터를 필요로 함
        - `Object collect(Collector collector);`
        - `Object collect(Supplier supplier, BiConsumer accumulator, BiConsumer combiner);`
    - Collector - 인터페이스, 컬렉터는 이 인터페이스를 구현해야함
    - Collectors - 클래스, static메서드로 미리 작성된 컬렉터를 제공함
        - toList(), toSet(), toMap, toCollection(), toArray()
        - toCollection, toArray
            
            `Student[] stuNames = studentStream.toArray(Student[]::new);` 코드에서 `Student[]::new`는 **배열 생성자 참조**입니다. 이 코드는 스트림의 요소들을 `Student` 타입의 배열로 변환하는 데 사용됩니다.
            
            **`toArray(IntFunction<A[]> generator)` 메서드:**
            
            `toArray()` 메서드는 스트림의 모든 요소를 배열로 변환하여 반환하는 최종 연산입니다. 이 메서드는 다음과 같은 형태를 가집니다.
            
            ```java
            <A> A[] toArray(IntFunction<A[]> generator)
            ```
            
            - `IntFunction<A[]> generator`: 정수 값을 받아서 `A[]` 타입의 배열을 생성하는 함수형 인터페이스입니다. 여기서 `A`는 배열의 요소 타입을 나타냅니다.
            
            **`Student[]::new`의 의미:**
            
            `Student[]::new`는 `IntFunction<Student[]>` 인터페이스를 구현하는 **생성자 참조**입니다. 이 생성자 참조는 정수 값 (배열 크기)을 입력으로 받아서 해당 크기의 `Student[]` 배열을 생성합니다.
            
            **동작 방식:**
            
            1. `studentStream.toArray(Student[]::new)`를 호출하면, `toArray()` 메서드는 먼저 스트림의 요소 개수를 파악합니다.
            2. 그런 다음, `Student[]::new` 생성자 참조를 사용하여 스트림의 요소 개수와 동일한 크기의 `Student[]` 배열을 생성합니다.
            3. 스트림의 모든 요소를 생성된 배열에 순서대로 복사합니다.
            4. 배열을 반환합니다.
            
            **왜 `Student[]::new`가 필요할까요?**
            
            - **타입 정보 유지:** `toArray()` 메서드는 스트림의 요소 타입을 유지하면서 배열로 변환하기 위해 `IntFunction<A[]> generator` 인자를 필요로 합니다. 이 인자를 통해 어떤 타입의 배열을 생성해야 하는지 명시적으로 지정할 수 있습니다.
            - **배열 크기 결정:** `toArray()` 메서드는 스트림의 요소 개수에 맞춰 배열의 크기를 동적으로 결정해야 합니다. `IntFunction`은 배열 크기를 입력으로 받아서 해당 크기의 배열을 생성하는 역할을 합니다.
            
            **만약 `Student[]::new`를 생략한다면:**
            
            `toArray()` 메서드를 인자 없이 호출하면 `Object[]` 타입의 배열이 반환됩니다. 이 경우, 배열의 요소 타입을 `Student`로 캐스팅해야 하며, 타입 안전성을 보장할 수 없습니다.
            
            **예시:**
            
            ```java
            import java.util.Arrays;
            import java.util.stream.Stream;
            
            class Student {
                private String name;
            
                public Student(String name) {
                    this.name = name;
                }
            
                public String getName() {
                    return name;
                }
            
                @Override
                public String toString() {
                    return "Student{name='" + name + "'}";
                }
            }
            
            public class Main {
                public static void main(String[] args) {
                    Stream<Student> studentStream = Stream.of(
                            new Student("Alice"),
                            new Student("Bob"),
                            new Student("Charlie")
                    );
            
                    // 1. Student[]::new 사용
                    Student[] stuNames = studentStream.toArray(Student[]::new);
                    System.out.println("Student[]: " + Arrays.toString(stuNames));
                    // 출력: Student[]: [Student{name='Alice'}, Student{name='Bob'}, Student{name='Charlie'}]
            
                    // 2. toArray()만 사용 (Object[] 반환)
                    Object[] objArray = Stream.of(
                            new Student("Alice"),
                            new Student("Bob"),
                            new Student("Charlie")
                    ).toArray();
            
                    System.out.println("Object[]: " + Arrays.toString(objArray));
                    // 출력: Object[]: [Student{name='Alice'}, Student{name='Bob'}, Student{name='Charlie'}]
                }
            }
            
            ```
            
            **요약:**
            
            `Student[]::new`는 `Student` 타입의 배열을 생성하는 생성자 참조이며, `toArray()` 메서드를 사용하여 스트림의 요소를 `Student` 타입의 배열로 변환할 때 필수적인 인자입니다. 이를 통해 타입 안전성을 보장하고, 스트림의 요소 타입을 유지할 수 있습니다.
            
        - toArray의 요소의 숫자를 계산해서 IntFunction 에 개수를 전달하는 과정에 대한 궁금증
            
            ## `Stream.toArray()` 메서드 시그니처 및 동작 방식 상세 설명
            
            `Stream.toArray()` 메서드의 동작 방식에 대한 의문점을 해소하기 위해, 먼저 메서드 시그니처를 정확히 확인하고, 스트림 요소 개수를 계산하여 넘겨주는 과정이 어떻게 가능한지 자세히 설명해 드리겠습니다.
            
            **1. `Stream.toArray()` 메서드 시그니처:**
            
            `Stream` 인터페이스에는 다음과 같은 두 가지 형태의 `toArray()` 메서드가 존재합니다.
            
            - **`Object[] toArray()`:**
                - 인자가 없는 `toArray()` 메서드는 스트림의 모든 요소를 `Object` 타입의 배열로 반환합니다.
                - 이 메서드는 스트림의 요소 타입을 알 수 없으므로, 가장 일반적인 타입인 `Object` 배열을 생성합니다.
                - 반환된 배열의 각 요소는 `Object` 타입으로 캐스팅해야 사용할 수 있습니다.
            - **`<A> A[] toArray(IntFunction<A[]> generator)`:**
                - 제네릭 타입 `A`를 사용하여 요소 타입을 지정할 수 있습니다.
                - `IntFunction<A[]> generator` 인자를 통해 배열의 타입과 크기를 결정하는 방법을 제공합니다.
                - 이 메서드는 스트림의 요소 타입을 유지하면서 배열로 변환할 수 있도록 해줍니다.
            
            **2. 핵심: `toArray(IntFunction<A[]> generator)` 메서드의 동작 방식**
            
            제공해주신 질문의 핵심은 바로 이 `toArray(IntFunction<A[]> generator)` 메서드의 동작 방식입니다. 이 메서드가 어떻게 스트림의 요소 개수를 계산하고, `IntFunction`을 사용하여 배열을 생성하고, 요소를 복사하는지 자세히 알아보겠습니다.
            
            **2.1. 스트림 요소 개수 계산:**
            
            - `toArray(IntFunction<A[]> generator)` 메서드는 스트림의 요소를 배열에 담기 전에 **먼저 스트림의 크기를 알아야 합니다.** 그렇지 않으면 정확한 크기의 배열을 생성할 수 없기 때문입니다.
            - 스트림의 크기를 알아내는 방법은 스트림의 종류에 따라 다릅니다.
                - **크기를 미리 알 수 있는 스트림 (Sized Stream):** `Collection.stream()`으로 생성된 스트림과 같이, 원래 컬렉션의 크기를 알고 있는 경우에는 스트림의 크기를 즉시 알 수 있습니다.
                - **크기를 미리 알 수 없는 스트림 (UnSized Stream):** `generate()`, `iterate()`, `Files.lines()` 등으로 생성된 스트림과 같이, 크기를 미리 알 수 없는 경우에는 스트림의 모든 요소를 순회하면서 크기를 계산해야 합니다. 이 경우, 내부적으로 임시 저장소(예: `ArrayList`)를 사용하여 요소를 저장하고, 저장된 요소의 개수를 파악합니다.
            
            **2.2. 배열 생성 및 요소 복사:**
            
            - 스트림의 크기를 파악한 후, `toArray(IntFunction<A[]> generator)` 메서드는 다음과 같은 단계를 거칩니다.
                1. `generator.apply(size)` 호출: `IntFunction<A[]> generator`의 `apply(int size)` 메서드를 호출하여 크기가 `size`인 `A[]` 타입의 배열을 생성합니다. 여기서 `size`는 앞서 계산한 스트림의 요소 개수입니다.
                2. 요소 복사: 스트림의 요소들을 생성된 배열에 순서대로 복사합니다.
                3. 배열 반환: 요소들이 복사된 배열을 반환합니다.
            
            **3. `AbstractPipeline.copyInto()` 메서드 (내부 구현):**
            
            `toArray()` 메서드의 실제 구현은 `AbstractPipeline` 클래스의 `copyInto()` 메서드에서 이루어집니다. 이 메서드는 스트림의 요소를 배열에 복사하는 역할을 합니다.
            
            ```java
            // AbstractPipeline.java (OpenJDK)
            @Override
            final <P_IN, A> A[] evaluateToArray(IntFunction<A[]> generator) {
                if (sourceStage.parallelism() == Spliterator.SIZED) {
                    long size = sourceStage.count();
                    if (size >= MAX_ARRAY_SIZE)
                        throw new IllegalArgumentException(Streams.MAX_ARRAY_SIZE_MSG);
                    A[] result = generator.apply((int) size);  // 배열 생성
                    new ToArrayTask<>(this, result).invoke(); // 요소 복사
                    return result;
                }
                else {
                    // 크기를 미리 알 수 없는 경우
                    ArrayList<E> list = new ArrayList<>();
                    forEach(list::add); // 임시 저장소에 요소 저장
                    return list.toArray(generator.apply(list.size())); // 배열 생성 및 요소 복사
                }
            }
            
            ```
            
            **설명:**
            
            - `sourceStage.parallelism() == Spliterator.SIZED`: 스트림이 크기를 미리 알 수 있는지 확인합니다.
            - `generator.apply((int) size)`: `IntFunction`을 사용하여 배열을 생성합니다.
            - `new ToArrayTask<>(this, result).invoke()`: 스트림의 요소를 배열에 복사합니다.
            
            **4. 의문점 해소:**
            
            - **`toArray()`가 어떻게 스트림 요소 개수를 계산하는가?** 스트림의 종류에 따라 다르지만, 크기를 미리 알 수 없는 스트림의 경우 내부적으로 스트림을 순회하면서 요소 개수를 계산합니다.
            - **`toArray()`가 어떻게 배열을 생성하는가?** `IntFunction<A[]> generator`를 통해 배열의 타입과 크기를 결정하는 방법을 제공받습니다. `toArray()`는 계산된 스트림 크기를 `generator`에 전달하여 배열을 생성합니다.
            - **`toArray()`가 어떻게 요소를 복사하는가?** `AbstractPipeline.copyInto()` 메서드를 통해 스트림의 요소를 생성된 배열에 복사합니다.
            
            **5. 정리:**
            
            `toArray(IntFunction<A[]> generator)` 메서드는 스트림의 요소 개수를 계산하고, `IntFunction`을 사용하여 배열을 생성하고, 스트림의 요소를 배열에 복사하는 과정을 **내부적으로 수행**합니다. 따라서 사용자는 배열 생성 및 요소 복사 과정을 직접 구현할 필요 없이, 간단하게 스트림을 배열로 변환할 수 있습니다.
            
            이 설명을 통해 `toArray()` 메서드의 동작 방식에 대한 의문점이 해소되었기를 바랍니다!
            
- 통계
    - counting(), summingInt(), averagingInt(), maxBy(), minBy(), summarizingInt()
    - 샘플 코드
        
        ## `Collectors` 메서드 샘플 코드: `counting()`, `summingInt()`, `averagingInt()`, `maxBy()`, `minBy()`, `summarizingInt()`
        
        다음은 Java Stream API의 `Collectors` 클래스에서 제공하는 통계 관련 메서드들의 사용법을 보여주는 샘플 코드입니다. 각 메서드의 역할과 사용 예시를 자세하게 설명했습니다.
        
        **1. `counting()`:** 스트림 요소의 개수를 반환합니다.
        
        ```java
        import java.util.Arrays;
        import java.util.List;
        import java.util.stream.Collectors;
        
        public class Main {
            public static void main(String[] args) {
                List<String> names = Arrays.asList("Alice", "Bob", "Charlie", "David");
        
                // 스트림 요소 개수 계산
                long count = names.stream()
                                   .collect(Collectors.counting());
        
                System.out.println("Number of names: " + count); // Number of names: 4
            }
        }
        
        ```
        
        **설명:** `names.stream().collect(Collectors.counting())`은 `names` 리스트의 스트림 요소 개수를 계산하여 `long` 타입으로 반환합니다.
        
        **2. `summingInt()`:** 스트림 요소의 정수 값의 합계를 반환합니다.
        
        ```java
        import java.util.Arrays;
        import java.util.List;
        import java.util.stream.Collectors;
        
        class Person {
            private String name;
            private int age;
        
            public Person(String name, int age) {
                this.name = name;
                this.age = age;
            }
        
            public int getAge() {
                return age;
            }
        }
        
        public class Main {
            public static void main(String[] args) {
                List<Person> people = Arrays.asList(
                        new Person("Alice", 25),
                        new Person("Bob", 30),
                        new Person("Charlie", 20)
                );
        
                // 사람들의 나이 합계 계산
                int totalAge = people.stream()
                                     .collect(Collectors.summingInt(Person::getAge));
        
                System.out.println("Total age: " + totalAge); // Total age: 75
            }
        }
        
        ```
        
        **설명:** `people.stream().collect(Collectors.summingInt(Person::getAge))`는 `people` 리스트의 각 `Person` 객체에서 `getAge()` 메서드를 호출하여 나이를 추출하고, 모든 나이의 합계를 `int` 타입으로 반환합니다.
        
        **3. `averagingInt()`:** 스트림 요소의 정수 값의 평균을 반환합니다.
        
        ```java
        import java.util.Arrays;
        import java.util.List;
        import java.util.stream.Collectors;
        
        class Person {
            private String name;
            private int age;
        
            public Person(String name, int age) {
                this.name = name;
                this.age = age;
            }
        
            public int getAge() {
                return age;
            }
        }
        
        public class Main {
            public static void main(String[] args) {
                List<Person> people = Arrays.asList(
                        new Person("Alice", 25),
                        new Person("Bob", 30),
                        new Person("Charlie", 20)
                );
        
                // 사람들의 평균 나이 계산
                double averageAge = people.stream()
                                         .collect(Collectors.averagingInt(Person::getAge));
        
                System.out.println("Average age: " + averageAge); // Average age: 25.0
            }
        }
        
        ```
        
        **설명:** `people.stream().collect(Collectors.averagingInt(Person::getAge))`는 `people` 리스트의 각 `Person` 객체에서 `getAge()` 메서드를 호출하여 나이를 추출하고, 모든 나이의 평균을 `double` 타입으로 반환합니다.
        
        **4. `maxBy()`:** 스트림 요소 중 지정된 `Comparator`를 기준으로 최대값을 갖는 요소를 `Optional`로 반환합니다.
        
        ```java
        import java.util.Arrays;
        import java.util.List;
        import java.util.Comparator;
        import java.util.Optional;
        import java.util.stream.Collectors;
        
        class Person {
            private String name;
            private int age;
        
            public Person(String name, int age) {
                this.name = name;
                this.age = age;
            }
        
            public String getName() {
                return name;
            }
        
            public int getAge() {
                return age;
            }
        
            @Override
            public String toString() {
                return "Person{name='" + name + "', age=" + age + '}';
            }
        }
        
        public class Main {
            public static void main(String[] args) {
                List<Person> people = Arrays.asList(
                        new Person("Alice", 25),
                        new Person("Bob", 30),
                        new Person("Charlie", 20)
                );
        
                // 가장 나이가 많은 사람 찾기
                Optional<Person> oldestPerson = people.stream()
                                                     .collect(Collectors.maxBy(Comparator.comparingInt(Person::getAge)));
        
                if (oldestPerson.isPresent()) {
                    System.out.println("Oldest person: " + oldestPerson.get()); // Oldest person: Person{name='Bob', age=30}
                } else {
                    System.out.println("People list is empty");
                }
            }
        }
        
        ```
        
        **설명:** `people.stream().collect(Collectors.maxBy(Comparator.comparingInt(Person::getAge)))`는 `people` 리스트의 각 `Person` 객체에서 `getAge()` 메서드를 호출하여 나이를 추출하고, 나이를 기준으로 가장 큰 값을 갖는 `Person` 객체를 `Optional<Person>`으로 반환합니다.
        
        **5. `minBy()`:** 스트림 요소 중 지정된 `Comparator`를 기준으로 최소값을 갖는 요소를 `Optional`로 반환합니다.
        
        ```java
        import java.util.Arrays;
        import java.util.List;
        import java.util.Comparator;
        import java.util.Optional;
        import java.util.stream.Collectors;
        
        class Person {
            private String name;
            private int age;
        
            public Person(String name, int age) {
                this.name = name;
                this.age = age;
            }
        
            public String getName() {
                return name;
            }
        
            public int getAge() {
                return age;
            }
        
            @Override
            public String toString() {
                return "Person{name='" + name + "', age=" + age + '}';
            }
        }
        
        public class Main {
            public static void main(String[] args) {
                List<Person> people = Arrays.asList(
                        new Person("Alice", 25),
                        new Person("Bob", 30),
                        new Person("Charlie", 20)
                );
        
                // 가장 나이가 어린 사람 찾기
                Optional<Person> youngestPerson = people.stream()
                                                       .collect(Collectors.minBy(Comparator.comparingInt(Person::getAge)));
        
                if (youngestPerson.isPresent()) {
                    System.out.println("Youngest person: " + youngestPerson.get()); // Youngest person: Person{name='Charlie', age=20}
                } else {
                    System.out.println("People list is empty");
                }
            }
        }
        
        ```
        
        **설명:** `people.stream().collect(Collectors.minBy(Comparator.comparingInt(Person::getAge)))`는 `people` 리스트의 각 `Person` 객체에서 `getAge()` 메서드를 호출하여 나이를 추출하고, 나이를 기준으로 가장 작은 값을 갖는 `Person` 객체를 `Optional<Person>`으로 반환합니다.
        
        **6. `summarizingInt()`:** 스트림 요소의 정수 값에 대한 다양한 통계 정보(개수, 합계, 평균, 최소값, 최대값)를 `IntSummaryStatistics` 객체로 반환합니다.
        
        ```java
        import java.util.Arrays;
        import java.util.List;
        import java.util.IntSummaryStatistics;
        import java.util.stream.Collectors;
        
        class Person {
            private String name;
            private int age;
        
            public Person(String name, int age) {
                this.name = name;
                this.age = age;
            }
        
            public int getAge() {
                return age;
            }
        
            @Override
            public String toString() {
                return "Person{name='" + name + "', age=" + age + '}';
            }
        }
        
        public class Main {
            public static void main(String[] args) {
                List<Person> people = Arrays.asList(
                        new Person("Alice", 25),
                        new Person("Bob", 30),
                        new Person("Charlie", 20)
                );
        
                // 사람들의 나이에 대한 통계 정보 계산
                IntSummaryStatistics ageStatistics = people.stream()
                                                          .collect(Collectors.summarizingInt(Person::getAge));
        
                System.out.println("Age statistics: " + ageStatistics); // Age statistics: IntSummaryStatistics{count=3, sum=75, min=20, average=25.000000, max=30}
                System.out.println("Average age: " + ageStatistics.getAverage()); // Average age: 25.0
                System.out.println("Max age: " + ageStatistics.getMax()); // Max age: 30
            }
        }
        
        ```
        
        **설명:** `people.stream().collect(Collectors.summarizingInt(Person::getAge))`는 `people` 리스트의 각 `Person` 객체에서 `getAge()` 메서드를 호출하여 나이를 추출하고, 모든 나이에 대한 통계 정보를 `IntSummaryStatistics` 객체로 반환합니다. `IntSummaryStatistics` 객체는 개수, 합계, 평균, 최소값, 최대값 등의 정보를 제공합니다.
        
        이 샘플 코드들이 `Collectors` 메서드들을 이해하고 사용하는 데 도움이 되었기를 바랍니다.
        
- reducing()
    - `Collector reducing(BinaryOperator<T> operator)`
    - `Collector reducing(T identity, BinaryOperator<T> operator)`
    - `Collector reducing(U identity, Function<T, U> mapper, BinaryOperator<U> operator)`
- joining()
    - 스트림의 모든 요소를 하나의 문자열로 연결해서 반환
    - collect(joining(delimeter, prefix, suffix) - 구분자, 최종결과 prefix, suffix
- groupingBy()
    - Function 으로 Stream 요소를 그룹화
    - 기본적으로 toList() 로 그룹화됨 (생략시)
    - mapFactory
        
        `groupingBy(Function classifier, Supplier mapFactory, Collector downstream)` 메서드에서 `mapFactory`는 **결과 맵(Map)의 구현체를 지정하는 역할**을 합니다. 즉, 어떤 종류의 `Map`을 사용하여 그룹화된 결과를 저장할지 결정하는 것이죠.
        
        **`mapFactory`의 타입:**
        
        - `Supplier<M>`: `Supplier`는 인자를 받지 않고 `M` 타입의 객체를 생성하여 반환하는 함수형 인터페이스입니다. 여기서 `M`은 `Map` 인터페이스를 구현하는 클래스입니다.
        
        **`mapFactory`를 사용하는 이유:**
        
        - **다양한 맵 구현체 활용:** 기본적으로 `groupingBy()` 메서드는 `HashMap`을 사용하여 결과를 저장합니다. 하지만 `mapFactory`를 사용하면 `TreeMap`, `LinkedHashMap` 등 다른 `Map` 구현체를 사용할 수 있습니다.
        - **특정 요구사항 충족:** 특정 맵 구현체는 특정 요구사항(예: 키 정렬, 삽입 순서 유지)을 충족하는 데 유용합니다.
        - **코드 유연성:** `mapFactory`를 사용하면 코드 변경 없이 맵 구현체를 쉽게 변경할 수 있습니다.
        
        **`mapFactory`로 사용할 수 있는 맵 구현체:**
        
        - **`HashMap`**: (기본값) 순서가 보장되지 않으며, 가장 빠른 성능을 제공합니다.
        - **`TreeMap`**: 키를 기준으로 정렬된 맵을 제공합니다. 키의 자연스러운 순서(natural ordering) 또는 `Comparator`를 사용하여 정렬할 수 있습니다.
        - **`LinkedHashMap`**: 삽입 순서를 유지하는 맵을 제공합니다.
        - **`ConcurrentHashMap`**: 스레드 안전한 맵을 제공합니다. 병렬 환경에서 안전하게 사용할 수 있습니다.
        
        **샘플 코드:**
        
        ```java
        import java.util.Arrays;
        import java.util.List;
        import java.util.Map;
        import java.util.TreeMap;
        import java.util.LinkedHashMap;
        import java.util.stream.Collectors;
        
        class Person {
            private String name;
            private int age;
        
            public Person(String name, int age) {
                this.name = name;
                this.age = age;
            }
        
            public String getName() {
                return name;
            }
        
            public int getAge() {
                return age;
            }
        
            @Override
            public String toString() {
                return "Person{name='" + name + "', age=" + age + '}';
            }
        }
        
        public class Main {
            public static void main(String[] args) {
                List<Person> people = Arrays.asList(
                        new Person("Alice", 25),
                        new Person("Bob", 30),
                        new Person("Charlie", 25),
                        new Person("David", 30),
                        new Person("Eve", 20)
                );
        
                // 1. HashMap 사용 (기본값)
                Map<Integer, List<Person>> peopleByAgeHashMap = people.stream()
                        .collect(Collectors.groupingBy(Person::getAge));
        
                System.out.println("1. HashMap: " + peopleByAgeHashMap);
                // 1. HashMap: {20=[Person{name='Eve', age=20}], 25=[Person{name='Alice', age=25}, Person{name='Charlie', age=25}], 30=[Person{name='Bob', age=30}, Person{name='David', age=30}]}
        
                // 2. TreeMap 사용 (키를 기준으로 정렬)
                TreeMap<Integer, List<Person>> peopleByAgeTreeMap = people.stream()
                        .collect(Collectors.groupingBy(
                                Person::getAge, // classifier
                                TreeMap::new, // mapFactory
                                Collectors.toList() // downstream
                        ));
        
                System.out.println("2. TreeMap: " + peopleByAgeTreeMap);
                // 2. TreeMap: {20=[Person{name='Eve', age=20}], 25=[Person{name='Alice', age=25}, Person{name='Charlie', age=25}], 30=[Person{name='Bob', age=30}, Person{name='David', age=30}]}
        
                // 3. LinkedHashMap 사용 (삽입 순서 유지)
                LinkedHashMap<Integer, List<Person>> peopleByAgeLinkedHashMap = people.stream()
                        .collect(Collectors.groupingBy(
                                Person::getAge, // classifier
                                LinkedHashMap::new, // mapFactory
                                Collectors.toList() // downstream
                        ));
        
                System.out.println("3. LinkedHashMap: " + peopleByAgeLinkedHashMap);
                // 3. LinkedHashMap: {Alice=25, Bob=30, Charlie=25, David=30, Eve=20}
            }
        }
        
        ```
        
        **설명:**
        
        1. **`HashMap` 사용:** `groupingBy()` 메서드를 인자 하나로 호출하면 기본적으로 `HashMap`이 사용됩니다.
        2. **`TreeMap` 사용:** `mapFactory`로 `TreeMap::new`를 지정하면 결과 맵이 `TreeMap`이 됩니다. `TreeMap`은 키를 기준으로 정렬되므로, 결과 맵의 키들이 정렬된 순서로 출력됩니다.
        3. **`LinkedHashMap` 사용:** `mapFactory`로 `LinkedHashMap::new`를 지정하면 결과 맵이 `LinkedHashMap`이 됩니다. `LinkedHashMap`은 요소들이 삽입된 순서를 유지하므로, 결과 맵의 키들이 삽입된 순서대로 출력됩니다.
        
        **`mapFactory` 사용 시점:**
        
        - 결과 맵의 순서가 중요한 경우 (예: 키를 기준으로 정렬하거나, 삽입 순서를 유지해야 하는 경우)
        - 특정 맵 구현체의 기능을 활용하고 싶은 경우 (예: 스레드 안전한 맵을 사용해야 하는 경우)
        
        **핵심:**
        
        - `mapFactory`는 결과 맵의 구현체를 지정하는 `Supplier`입니다.
        - `mapFactory`를 사용하면 다양한 맵 구현체(`HashMap`, `TreeMap`, `LinkedHashMap` 등)를 활용하여 `groupingBy()` 메서드의 동작을 세밀하게 제어할 수 있습니다.
        - 결과 맵의 순서가 중요하거나, 특정 맵 구현체의 기능을 활용하고 싶은 경우에 `mapFactory`를 사용합니다.
        
        이 설명을 통해 `mapFactory`에 대한 이해가 더욱 명확해졌기를 바랍니다!
        
    - collectingAndThen()
        
        `collect`와 `collectingAndThen`은 모두 `Collector`를 인자로 받아서 스트림의 요소들을 수집하는 최종 연산이지만, 반환 타입에는 차이가 있습니다.
        
        **1. `collect(Collector collector)`**
        
        - **반환 타입:** `R` (Collector 인터페이스의 세 번째 제네릭 타입)
        - **역할:** 스트림의 요소들을 수집하여 `Collector`가 지정하는 방식으로 결과를 생성하고 반환합니다.
        - **`Optional<T>` 반환 여부:** `collect` 메서드 자체가 `Optional<T>`를 직접 반환하는 경우는 **없습니다.** `Collector`의 구현에 따라 `Optional<T>`를 결과 타입으로 가질 수는 있습니다. (예: `Collectors.maxBy()`, `Collectors.minBy()`)
        
        **2. `collectingAndThen(Collector downstream, Function finisher)`**
        
        - **반환 타입:** `RR` (finisher 함수의 반환 타입)
        - **역할:**
            1. 스트림의 요소들을 `downstream` `Collector`를 사용하여 수집합니다.
            2. `downstream` `Collector`의 결과를 `finisher` 함수에 적용하여 최종 결과를 생성하고 반환합니다.
        - **`Optional<T>` 반환 여부:** `collectingAndThen` 메서드 자체가 `Optional<T>`를 직접 반환하는 경우는 **없습니다.** 하지만, `downstream` `Collector`가 `Optional<T>`를 반환하는 경우, `finisher` 함수를 사용하여 `Optional<T>`를 처리하고 다른 타입으로 변환할 수 있습니다.
        
        **정리:**
        
        | 메서드 | 반환 타입 | 역할 | `Optional<T>` 반환 여부 |
        | --- | --- | --- | --- |
        | `collect(Collector collector)` | `R` | 스트림의 요소들을 `Collector`가 지정하는 방식으로 수집하여 결과를 생성하고 반환합니다. | `Collector` 구현에 따라 `Optional<T>`를 결과 타입으로 가질 수 있습니다. |
        | `collectingAndThen(Collector downstream, Function finisher)` | `RR` | 1. 스트림의 요소들을 `downstream` `Collector`를 사용하여 수집합니다. 2. `downstream` `Collector`의 결과를 `finisher` 함수에 적용하여 최종 결과를 생성하고 반환합니다. | `downstream` `Collector`가 `Optional<T>`를 반환하는 경우, `finisher` 함수를 사용하여 `Optional<T>`를 처리하고 다른 타입으로 변환할 수 있습니다. |
        
        **핵심:**
        
        - `collect`는 `Collector`의 결과 타입을 그대로 반환합니다.
        - `collectingAndThen`은 `Collector`의 결과를 `finisher` 함수를 통해 변환한 후 반환합니다.
        - `collect`와 `collectingAndThen` 모두 `Optional<T>`를 직접 반환하는 경우는 없지만, `Collector` 또는 `finisher` 함수를 통해 `Optional<T>`를 처리할 수 있습니다.
        
        **샘플 코드:**
        
        ```java
        import java.util.Arrays;
        import java.util.List;
        import java.util.Optional;
        import java.util.stream.Collectors;
        
        class Person {
            private String name;
            private int age;
        
            public Person(String name, int age) {
                this.name = name;
                this.age = age;
            }
        
            public String getName() {
                return name;
            }
        
            public int getAge() {
                return age;
            }
        
            @Override
            public String toString() {
                return "Person{name='" + name + "', age=" + age + '}';
            }
        }
        
        public class Main {
            public static void main(String[] args) {
                List<Person> people = Arrays.asList(
                        new Person("Alice", 25),
                        new Person("Bob", 30),
                        new Person("Charlie", 20)
                );
        
                // 1. collect() 사용 (Optional<Person> 반환)
                Optional<Person> oldestPerson = people.stream()
                        .collect(Collectors.maxBy((p1, p2) -> p1.getAge() - p2.getAge()));
        
                System.out.println("Oldest person (Optional): " + oldestPerson);
                // Oldest person (Optional): Optional[Person{name='Bob', age=30}]
        
                // 2. collectingAndThen() 사용 (Person 반환)
                Person oldestPerson2 = people.stream()
                        .collect(Collectors.collectingAndThen(
                                Collectors.maxBy((p1, p2) -> p1.getAge() - p2.getAge()),
                                optionalPerson -> optionalPerson.orElse(null) // 값이 없으면 null 반환
                        ));
        
                System.out.println("Oldest person: " + oldestPerson2);
                // Oldest person: Person{name='Bob', age=30}
        
                // 3. collectingAndThen() 사용 (이름 반환)
                String oldestPersonName = people.stream()
                        .collect(Collectors.collectingAndThen(
                                Collectors.maxBy((p1, p2) -> p1.getAge() - p2.getAge()),
                                optionalPerson -> optionalPerson.map(Person::getName).orElse("No one") // 이름이 없으면 "No one" 반환
                        ));
        
                System.out.println("Oldest person name: " + oldestPersonName);
                // Oldest person name: Bob
            }
        }
        
        ```
        
        **설명:**
        
        1. `collect(Collectors.maxBy(...))`: `Collectors.maxBy()`는 `Optional<Person>`을 반환합니다.
        2. `collectingAndThen(Collectors.maxBy(...), optionalPerson -> optionalPerson.orElse(null))`: `Collectors.maxBy()`의 결과인 `Optional<Person>`을 `orElse(null)`을 사용하여 `Person` 타입으로 변환합니다.
        3. `collectingAndThen(Collectors.maxBy(...), optionalPerson -> optionalPerson.map(Person::getName).orElse("No one"))`: `Collectors.maxBy()`의 결과인 `Optional<Person>`을 `map(Person::getName)`을 사용하여 `Optional<String>`으로 변환하고, `orElse("No one")`을 사용하여 값이 없으면 "No one"을 반환합니다.
        
        이 설명을 통해 `collect`와 `collectingAndThen`의 차이점을 명확하게 이해하는 데 도움이 되었기를 바랍니다.
        
    - 
    
    ```java
    Collector groupingBy(Function classifier)
    Collector groupingBy(Function classifier, Collector downstream)
    Collector groupingBy(Function classifier, Supplier mapFactory, Collector downstream)
    
    ```
    
- partitioningBy()
    - Predicate로 Stream 요소를 분류
    - 기본형으로는 return 이 Map형식임 true / false 키로 구분
    - downstream은 각 그룹의 요소를 컬렉터를 사용해 수집
    
    ```java
    Collector partitioningBy(Predicate predicate)
    Collector partitioningBy(Predicate predicate, Collector downstream)
    ```
    

- Collector 구현
    
    ```java
    public interface Collector<T, A, R> {
    	Supplier<A> supplier(); // 작업 결과를 저장할 공간 제공
    	BiConsumer<A, T> accumulator(); // 스트림의 요소를 수집할 방법 제공
    	BinaryOperator<A> combiner(); // 두 저장곤간을 병합할 방법 제공 (병렬 스트림)
    	Function<A, R> finisher(); // 결과를 최종적으로 변환할 방법 제공
    	
    	Set<Characteristics> characteristic(); // 컬렉터 특성이 담긴 Set 반환
    	Characteristic.CONCURRENT // 병렬로 처리할 수 있는 작업
    	Characteristic.UNORDERED // 스트림의 요소의 순서가 유지될 필요가 없는 작업
    	Characteristic.IDENTITY_FINISH // finisher() 가 항등 함수인 작업 Function.identity() 사용
    ```
    
    Characteristics set
    
    ```java
    @Override
    Set<Characteristics> characteristics() {
    	return Collections.unmodifiableSet(EnumSet.of(
    						Collector.Characterlistics.CONCURRENT,
    						Collector.Characterlistics.UNORDERED
    					));
    	}
    	
    	// 속성지정 안하려면
    	@Override
    	Set<Characteristics> characteristics() {
    		return Collections.emptySet();
    	}
    ```