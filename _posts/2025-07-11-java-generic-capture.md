---
title: capture of ? 에러 완벽 해부 - 제네릭 와일드카드
description: 
author: laze
date: 2025-07-11 00:00:06 +0900
categories: [Dev, Java]
tags: [Java, Generic, Capture]
---
# [Java/Spring Batch] `capture of ?` 에러 완벽 해부: 제네릭 와일드카드

## 1. 서론: 마주치면 당황스러운 `capture of ?` 에러

Spring Batch로 `ItemWriter`를 구현하다 보면, `chunk.getItems()`를 호출하는 지점에서 컴파일러가 낯선 에러 메시지를 뱉어내는 경우가 있다.

```
Required Type: List<List<MyDto>>
Provided:    List<capture of ? extends List<MyDto>>
```

"내가 보기엔 똑같은 타입 같은데, `capture of ?`는 대체 뭐지?"

이 에러는 단순한 문법 오류가 아니다. Java의 제네릭 시스템이 **타입 안전성(Type Safety)**을 지키기 위해 얼마나 정교하게 동작하는지를 보여주는 증거이며, 와일드카드(`?`)의 본질을 이해해야만 그 원인을 파악할 수 있다.

이 글에서는 이 미스터리한 `capture of ?` 에러의 원인을 심층적으로 분석하고, 현대적인 Java 스타일로 이 문제를 해결하는 가장 깔끔한 방법을 알아본다.

## 2. 문제의 배경: 와일드카드(`?`)는 왜 필요한가?

Java 제네릭에서 `List<Dog>`는 `List<Animal>`의 하위 타입이 아니다. 즉, **제네릭은 불공변(invariant)**하다. 이로 인해 발생하는 유연성 문제를 해결하기 위해 와일드카드(`?`)가 도입되었다.

```java
// "? extends Animal": Animal 또는 Animal의 모든 자식 타입을 허용한다.
// 이를 '상한 경계 와일드카드(Upper Bounded Wildcard)'라 한다.
void printAnimalNames(List<? extends Animal> animals) {
    for (Animal animal : animals) {
        System.out.println(animal.getName());
    }
}

List<Dog> dogs = new ArrayList<>();
List<Cat> cats = new ArrayList<>();

printAnimalNames(dogs); // OK
printAnimalNames(cats); // OK
```

Spring Batch의 `ItemWriter` 인터페이스에 정의된 `write` 메서드 시그니처도 이와 같은 유연성을 위해 와일드카드를 사용한다.

```java
public interface ItemWriter<T> {
    void write(Chunk<? extends T> chunk) throws Exception;
}
```

`ItemReader`가 `List<Dog>`를 아이템으로 반환하면, `ItemWriter<List<Animal>>`는 `Chunk<List<Dog>>`를 안전하게 받아 처리할 수 있어야 하기 때문이다.

## 3. 'capture of ?'의 정체: 컴파일러의 추론

이제 문제의 핵심으로 들어가 보자. 컴파일러는 "알 수 없는 타입"을 의미하는 와일드카드(`?`)를 코드 상에서 직접 다룰 수 없다.

그래서 컴파일러는 이 와일드카드를 만날 때마다, 내부적으로 **임시적인 타입 이름(e.g., `CAP#1`, `CAP#2`)을 부여**하여 그 타입을 특정한다.

이 과정을 **'캡처 변환(Capture Conversion)'** 이라고 한다.

에러 메시지에 등장하는 `capture of ?`는 바로 컴파일러의 이 내부 동작을 그대로 우리에게 보여주는 것이다.

> 컴파일러 왈: "지금 내가 다루고 있는 이 타입은... 음... extends 뒤에 있는 타입의 자식인 건 알겠는데, 정확한 이름은 모르겠다. 내가 방금 임시로 이름을 붙인 '그 녀석(capture of ?)' 이라고 해두자."
>

## 4. 에러의 근본 원인 분석: 타입 불일치의 진실

자, 이제 우리가 겪었던 에러 상황을 컴파일러의 시점에서 다시 분석해 보자.

**우리의 코드:**

```java
// ItemWriter의 제네릭 T는 List<BIVL106002Dto> 이다.
public ItemWriter<List<BIVL106002Dto>> myWriter() {
    return chunk -> {
        // chunk의 실제 타입은 'Chunk<? extends List<BIVL106002Dto>>' 이다.
        // chunk.getItems()의 반환 타입은 'List<? extends List<BIVL106002Dto>>' 이다.

        List<List<? extends BIVL106002Dto>> lists = chunk.getItems(); // 에러 발생 지점
    };
}
```

**에러 메시지 해석:**

- **Required Type (내가 선언한 변수 타입):** `List<List<? extends BIVL106002Dto>>`
  - **컴파일러의 해석:** "리스트의 리스트인데, 그 안쪽 요소의 타입은 **'와일드카드를 포함한 `List`'라는, 바로 그 구체적인 타입**이어야 한다." (`List<? extends ...>` 라는 타입 자체를 의미)
- **Provided (컴파일러가 `chunk.getItems()`로부터 실제로 받은 타입):** `List<capture of ? extends List<BIVL106002Dto>>`
  - **컴파일러의 해석:** "리스트의 리스트인데, 그 안쪽 요소의 타입은 **'정체를 알 수 없는 어떤 타입'(`capture of ?`)**이다. 그 타입은 `List<BIVL106002Dto>`의 하위 타입이긴 하다."

미묘해 보이지만, 컴파일러에게 이 둘은 명백히 다르다. **'정체를 알 수 없는 타입'을 '구체적인 타입' 변수에 할당하는 것은 타입 안전성을 해칠 수 있는 위험한 행동**이므로, 컴파일러는 이를 거부하고 경고 또는 에러를 발생시키는 것이다.

## 5. 해결책: 컴파일러를 도와주는 두 가지 방법

이 문제를 해결하는 방법은 간단하다. 컴파일러가 혼란스러워하지 않도록 도와주면 된다.

### 해결책 1 (Best): `var` 키워드로 타입 추론 위임하기 (Java 10+)

가장 현대적이고 강력한 해결책이다. 개발자가 복잡한 제네릭 타입을 직접 작성하는 대신, 컴파일러가 가장 정확한 타입을 추론하도록 맡기는 것이다.

```java
public ItemWriter<List<BIVL106002Dto>> myWriter() {
    return chunk -> {
        // 'var'를 사용하면 컴파일러가 chunk.getItems()의 반환 타입을
        // 정확히 'List<? extends List<BIVL106002Dto>>'로 추론해준다.
        var lists = chunk.getItems();

        // 이후 로직에서 'lists' 변수를 사용하는 데 아무런 문제가 없다.
        List<BIVL106002Dto> flattenedList = lists.stream()
                .flatMap(list -> list.stream())
                .collect(Collectors.toList());
        // ...
    };
}
```

- **장점:** 개발자는 복잡한 제네릭 타입 문법을 신경 쓸 필요가 없다. 코드가 매우 간결해지고 가독성이 비약적으로 향상된다.

### 해결책 2 (Alternative): 와일드카드를 사용하여 타입 정확히 맞추기

`var`를 사용할 수 없는 Java 10 미만 환경이라면, 변수 타입을 `getItems()`가 반환하는 실제 타입과 정확하게 일치시켜야 한다.

```java
public ItemWriter<List<BIVL106002Dto>> myWriter() {
    return chunk -> {
        // 변수 타입을 getItems()의 실제 리턴 타입과 동일하게 맞춰준다.
        List<? extends List<BIVL106002Dto>> lists = chunk.getItems();

        List<BIVL106002Dto> flattenedList = lists.stream()
                .flatMap(Collection::stream) // List::stream 대신 Collection::stream 사용 가능
                .collect(Collectors.toList());
        // ...
    };
}
```

- **장점:** `var` 없이도 문제를 해결할 수 있으며, 타입이 코드에 명시적으로 드러난다.
- **단점:** 코드가 길고 복잡해 보이며, 제네릭 문법에 대한 정확한 이해가 필요하다.

## 6. 결론

`capture of ?` 에러는 Java의 제네릭 시스템이 코드의 타입 안전성을 얼마나 철저하게 지키려고 노력하는지를 보여주는 좋은 예시다. 이 에러는 버그가 아니라, "정체를 알 수 없는 타입"을 다루는 과정에서 발생하는 자연스러운 현상이다.

복잡한 제네릭 타입을 마주쳤을 때, 그 타입을 일일이 분석하고 작성하려 애쓰기보다 **`var` 키워드를 사용하여 컴파일러에게 타입 추론을 위임하는 것**이 현대적인 Java 개발에서 가장 현명하고 생산적인 접근법이다. `var`는 단순히 코드를 줄여주는 것을 넘어, 이와 같은 복잡한 제네릭 문제를 우아하게 해결해주는 강력한 도구임을 기억하자.

---
