---
title: 자바의 정석 Chapter 12
description: 모르는 내용 위주 학습 기록
author: laze
date: 2025-02-27 00:06:00 +0900
categories: [Dev, Java]
tags: [Java]
---
# Chapter 12

Generics

- 컴파일 시 타입 체크 해주는 기능
- 타입 안정성
- 타입 체크와 형변환 생략 → 코드 간결해짐

class Box<T>

- Box<T> - generic class
- T - 타입 변수
- Box - 원시 타입(raw type)

제한

- static 에 타입변수 사용 불가능
- 지네릭 배열은 new 연산자 때문에 불가능 컴파일 시점에서 T가 무엇인지 정확히 알아야 사용가능
    - Reflection API의 newInstance() 나 Object 배열을 생성하고 T[]로 형변환 하는 방법 사용 가능

타입 제한

- class FruitBox<T extends Fruit> → Fruit의 자손 타입으로 제한
- class FruitBox<T extends Eatable> → Eatable interface로도 제한 가능
- class FruitBox<T extends Fruit & Eatable> → Fruit 이면서 Eatable 구현한 클래스로 제한

와일드카드

- 밑의 예시를 <Apple> <Grape> 로 메서드를 두번 정의한다해서 지네릭 타입이 다르다고 오버로딩 되는게 아니라 중복정의 한 것으로 컴파일됨 이럴때 와일드 카드를 사용
- <? extends T> - 상한 제한, T와 자손들만 가능
- <? super T> - 하한 제한, T와 조상들만 가능
- 와일드카드에는 & 사용 불가

```java
static Juice makeJuice(FruitBox<? extends Fruit> box) {
	String tmp = "";
	for(Fruit f : box.getList()) tmp += f + " ";
	return new Juice(tmp);
}
```

지네릭 메서드

- 반환 타입의 바로 앞에 선언
- static <T> void sort(List<T> list, Comparator<? super T> c)
- 타입 매개변수와는 별개의 것으로 같은 T를 사용해도 같은 것이 아님
- 메서드에 선언된 지네릭 타입은 지역 변수를 선언한 것과 같음

```java
static <T extends Fruit> Juice makeJuice(Fruit<T> box)

FruitBox<Fruit> fruitBox = new FruitBox<Fruite>;
System.out.println(Juicer.<Fruit>makeJuice(fruitBox);

FruitBox<Apple> appleBox = new FruitBox<Apple>;
System.out.println(Juicer.<Apple>makeJuice(appleBox);
```

- 헷갈리는 부분 정리
    
    **1. `<T extends Fruit>`의 `T`와 `(FruitBox<T> box)`의 `T`는 다를 수 있다는 뜻인가?**
    
    아니요, 다를 수 있다는 뜻이 아닙니다. 둘은 **같은** `T`를 나타냅니다.  `static <T extends Fruit> Juice makeJuice(FruitBox<T> box)` 선언은 다음과 같은 의미를 가집니다.
    
    - `<T extends Fruit>`: `makeJuice` 메서드 내에서 사용할 타입 매개변수 `T`를 선언합니다. 그리고 `T`는 반드시 `Fruit`이거나 `Fruit`의 하위 타입이어야 한다는 제약 조건을 설정합니다.
    - `(FruitBox<T> box)`: `makeJuice` 메서드의 파라미터로 `FruitBox<T>` 타입의 `box`를 받습니다. 여기서 `T`는 위에서 선언한 타입 매개변수 `T`와 **동일한 타입**을 나타냅니다.
    
    즉, `makeJuice` 메서드를 호출할 때, `T`가 어떤 타입으로 결정되면, `FruitBox<T>`의 `T`도 똑같은 타입으로 결정됩니다.
    
    예를 들어, `makeJuice` 메서드를 `FruitBox<Apple>` 타입의 인자로 호출했다면, `T`는 `Apple`이 되고, 메서드 내에서 사용되는 `T`는 모두 `Apple`을 나타냅니다.  `FruitBox<Orange>` 타입의 인자로 호출했다면, `T`는 `Orange`가 됩니다.
    
    **2. `T extends Fruit`를 왜 반환값 앞에 붙이는가?**
    
    `T extends Fruit`는 반환값을 제한하는 것이 아니라, **타입 매개변수** `T`를 선언하는 부분입니다.  지네릭 메서드에서 타입 매개변수는 반드시 반환 타입 바로 앞에 선언해야 합니다.
    
    이것은 자바 문법 규칙입니다.  컴파일러는 이 위치에 있는 `<>`를 보고 "아, 이 메서드는 지네릭 메서드구나.  <> 안에 선언된 타입 매개변수를 이 메서드 내에서 사용할 수 있겠구나"라고 인식합니다.
    
    `T extends Fruit`는 `T`가 될 수 있는 타입을 `Fruit` 또는 `Fruit`의 하위 타입으로 제한하는 역할을 합니다.  이렇게 제약을 둠으로써 컴파일러는 메서드 내에서 `T` 타입의 객체에 대해 `Fruit` 클래스에 정의된 메서드를 안전하게 호출할 수 있다는 것을 보장할 수 있습니다.
    
    **정리:**
    
    - `<T extends Fruit>`는 `makeJuice` 메서드 내에서 사용할 타입 매개변수 `T`를 선언하고, `T`의 범위를 `Fruit` 또는 `Fruit`의 하위 타입으로 제한합니다.
    - `(FruitBox<T> box)`는 `makeJuice` 메서드의 파라미터로 `FruitBox<T>` 타입의 `box`를 받으며, 여기서 `T`는 위에서 선언한 타입 매개변수 `T`와 동일한 타입을 나타냅니다.
    - `<T extends Fruit>`는 반환값을 제한하는 것이 아니라, 타입 매개변수를 선언하는 문법적인 위치입니다.
    
    질문:
    
    - 반환 타입 앞에 `<T extends Fruit>`를 사용하고, 매개변수 자리에는 `FruitBox<T extends Berry>`를 사용하면 어떻게 되는가?
    - 왜 타입 매개변수 제한을 반환 타입 앞에 선언하는가? 그 이유가 무엇인가?
    
    **답변:**
    
    1. **`FruitBox<T extends Berry>`는 유효하지 않은 문법입니다.**
        
        자바에서 제네릭 타입 매개변수 제한(bounds)은 **타입 매개변수를 선언하는 위치**에서만 지정할 수 있습니다.  `FruitBox<T extends Berry>`는 `FruitBox` 클래스의 타입 매개변수를 제한하는 것이 아니라, 메서드 내에서 `T`라는 새로운 지역 타입 매개변수를 선언하려고 시도하는 것으로 해석될 수 있습니다.  하지만, 파라미터 타입 내에서는 타입 매개변수를 선언할 수 없으므로 컴파일 에러가 발생합니다.
        
        만약 `FruitBox` 클래스 자체가 `FruitBox<T extends Fruit>`처럼 선언되어 있다면, `FruitBox`에 전달되는 타입은 이미 `Fruit`의 하위 타입으로 제한되어 있습니다. 따라서 메서드 파라미터에서 추가적인 제한을 둘 필요가 없습니다.
        
    2. **타입 매개변수 제한을 반환 타입 앞에 선언하는 이유**
        
        타입 매개변수 제한은 **메서드 내에서 타입 안전성을 확보하기 위한 핵심 메커니즘**입니다.  이 제한을 통해 컴파일러는 메서드 내에서 해당 타입 매개변수로 수행할 수 있는 연산을 검증하고, 런타임에 타입 관련 오류가 발생하는 것을 방지할 수 있습니다.
        
        **왜 반환 타입 앞인가?**
        
        - **스코프(Scope) 정의:** 반환 타입 앞에 `<T extends Fruit>`를 선언함으로써, 해당 `<T>`가 **메서드 전체**에서 유효한 타입 매개변수임을 명시적으로 선언하는 것입니다. 즉, `<T>`는 이 메서드의 "스코프" 내에서만 의미를 가지는 지역적인 타입 매개변수라는 것을 알려줍니다.
        - **타입 추론의 시작점:** 컴파일러는 이 선언을 보고 메서드 호출 시점에 `T`가 어떤 타입으로 결정될지 추론합니다. 메서드의 인자 타입을 기반으로 `T`를 추론하거나, 명시적으로 타입을 지정할 수도 있습니다.
        - **제약 조건 적용:** `<T extends Fruit>`는 `T`가 `Fruit` 또는 `Fruit`의 하위 타입이어야 한다는 제약 조건을 설정합니다. 이 제약 조건은 메서드 내에서 `T` 타입의 객체에 대해 `Fruit` 클래스에 정의된 메서드를 안전하게 호출할 수 있도록 보장합니다. 만약 `T`에 대한 제약 조건이 없다면, 컴파일러는 `T`가 어떤 타입인지 알 수 없으므로, 특정 메서드를 호출하는 것이 안전한지 판단할 수 없습니다.
        
        **예시:**
        
        ```java
        static <T extends Fruit> void printFruitName(T fruit) {
            System.out.println(fruit.getName()); // Fruit 클래스에 getName() 메서드가 있다고 가정
        }
        
        ```
        
        위 코드에서 `<T extends Fruit>`가 없다면, 컴파일러는 `fruit.getName()` 호출이 안전한지 알 수 없습니다. 왜냐하면 `T`가 어떤 타입인지 알 수 없기 때문입니다. 하지만 `<T extends Fruit>`가 있으면, 컴파일러는 `fruit`가 `Fruit` 또는 `Fruit`의 하위 타입임을 알고, `Fruit` 클래스에 `getName()` 메서드가 정의되어 있다면 안전하게 메서드를 호출할 수 있습니다.
        
    
    **요약:**
    
    - 타입 매개변수 제한은 타입 안전성을 보장하고, 컴파일러가 메서드 내에서 수행할 수 있는 연산을 검증하는 데 사용됩니다.
    - 반환 타입 앞에 `<T extends Fruit>`를 선언하는 것은 해당 `<T>`가 메서드 내에서 유효한 타입 매개변수임을 선언하고, `T`에 대한 제약 조건을 설정하는 것입니다.
    - 이러한 제약 조건은 컴파일러가 메서드 내에서 `T` 타입의 객체에 대해 안전하게 메서드를 호출할 수 있도록 보장합니다.
    
    **2. 메서드 단에서 T라 하면 클래스에서 선언한 T인지 제네릭 메서드로 사용한 T인지 어떻게 구별하는가?**
    
    컴파일러는 **스코프(Scope)**를 기반으로 `T`가 어떤 `T`인지 구별합니다. 스코프는 변수 또는 식별자가 유효한 영역을 의미합니다.
    
    - **클래스 레벨의 `T`:** 클래스 선언 시 `<T>`로 선언된 `T`는 해당 클래스 전체에서 유효합니다. 이 `T`는 클래스의 멤버 변수, 메서드의 파라미터, 반환 타입 등에서 사용될 수 있습니다.
    - **메서드 레벨의 `T`:** 메서드 선언 시 `<T>`로 선언된 `T`는 해당 메서드 내에서만 유효합니다. 이 `T`는 메서드의 파라미터, 반환 타입, 지역 변수 등에서 사용될 수 있습니다.
    
    **구별 규칙:**
    
    1. **가장 가까운 스코프 우선:** 메서드 내에서 `T`를 사용하면, 컴파일러는 **가장 가까운 스코프**에서 `T`를 찾습니다. 즉, 메서드 레벨에서 `T`가 선언되었다면, 클래스 레벨의 `T`는 가려지고 메서드 레벨의 `T`가 사용됩니다.
    2. **이름 충돌 시:** 만약 클래스 레벨과 메서드 레벨에서 같은 이름의 `T`가 사용된다면, 메서드 레벨의 `T`가 클래스 레벨의 `T`를 가립니다 (shadowing). 메서드 내에서는 메서드 레벨의 `T`만 사용할 수 있습니다.
    3. **명시적인 타입 지정:** 제네릭 메서드 호출 시 `<Type>`을 사용하여 명시적으로 타입을 지정할 수 있습니다. 이렇게 하면 컴파일러는 해당 타입을 메서드 레벨의 `T`로 사용합니다.
    
    **예시:**
    
    ```java
    class GenericClass<T> { // 클래스 레벨의 T
        private T value;
    
        public <T> T genericMethod(T arg) { // 메서드 레벨의 T (클래스 레벨의 T를 가린다)
            return arg; // 메서드 레벨의 T를 반환
        }
    
        public T getClassValue() { // 클래스 레벨의 T를 반환
            return value;
        }
    }
    
    public class Main {
        public static void main(String[] args) {
            GenericClass<String> obj = new GenericClass<>();
            String result = obj.<Integer>genericMethod(123); // 메서드 레벨의 T는 Integer
            String classValue = obj.getClassValue(); // 클래스 레벨의 T는 String
        }
    }
    
    ```
    
    위 예시에서 `genericMethod` 내의 `T`는 클래스 `GenericClass`의 `T`를 가립니다. `main` 메서드에서 `obj.<Integer>genericMethod(123)`를 호출하면, 컴파일러는 `genericMethod`의 `T`를 `Integer`로 인식하고, 클래스 `GenericClass`의 `T`는 여전히 `String`으로 유지됩니다.
    
    **요약:**
    
    - 컴파일러는 스코프를 기반으로 `T`를 구별합니다.
    - 가장 가까운 스코프의 `T`가 우선적으로 사용됩니다.
    - 메서드 레벨의 `T`는 클래스 레벨의 `T`를 가릴 수 있습니다.
    - 제네릭 메서드 호출 시 명시적으로 타입을 지정하여 메서드 레벨의 `T`를 결정할 수 있습니다.
    
    이 설명이 도움이 되었기를 바랍니다. 궁금한 점이 있다면 언제든지 다시 질문해주세요!
    

지네릭 타입의 형변환

- 지네릭 타입과 원시타입의 형 변환은 가능함 (경고는 발생)

```java
Box box = null;
Box<Object> objBox = null;

box = (Box)objBox;
objBox = (Box<Object>)box;

Box<Object> objBox = null;
Box<String> strBox = null;
strBox = (Box<String>)objBox; // error Box<String> <-> Box<Object> 지네릭 타입이 다르면 형변환 불가
```

지네릭타입 제거

- 컴파일러단에서 지네릭 타입을 이용해 소스파일을 체크하고 필요한 곳에 형변환을 넣어준다.
- 이후 지네릭타입을 제거하므로 class 파일에서는 지네릭타입에 대한 정보가 없음
- 과정
    - 지네릭 타읩의 경계(bound) 제거 → <T extends Fruit> → <Fruit> 로 치환
    - 지네릭타입을 제거한 후 타입이 일치하지 않으면 형변환을 추가
    
    ```java
    class Box<T extends Fruit> {
    	void add(T t) {
    		...
    	}
    }
    
    class Box {
    	void add(Fruit t) {
    		...
    	}
    }
    
    T get(int i) {
    	return list.get(i);
    }
    
    Fruit get(int i) {
    	return (Fruit)list.get(i);
    }
    ```
    

Enums 열거형

- 서로 관련된 상수를 편리하게 선언하기 위한 것

```java
class Card {
	enum Kind { CLOVER, HEART, DIAMOND, SPADE }
	enum Value { TWO, THREE, FOUR }
	
	final Kind kind;
	final Value value;
}
```

- 타입이 달라도 값이 같으면 조건식 결과가 참인 경우를 막고 타입까지 체크
- 상수의 값이 바뀌면 해당 상수를 참조하는 모든 소스를 다시 컴파일해야하는 경우를 열거형 상수를 사용하면 기존 소스를 다시 컴파일하지 않아도 됨

```java
Direction d1 = Direction.valueOf("WEST");
Direction d2 = Direction.WEST;
Direction d3 = Enum.valueOf(Direction.class, "WEST");

// d2.ordinal 열거형 상수가 정의된 순서를 반환
```

- 열거형에 멤버 추가

```java
enum Direction {
    EAST(1, "→"), SOUTH(5, "↓"), WEST(-1, "←"), NORTH(10, "⬆");

    private final int value;
    private final String arrow; // 화살표 문자열 추가

    Direction(int value, String arrow) { // 생성자 수정
        this.value = value;
        this.arrow = arrow;
    }

    public int getValue() {
        return value;
    }

    public String getArrow() { // 화살표 문자열 getter
        return arrow;
    }
}

public class Main {
    public static void main(String[] args) {
        Direction direction = Direction.EAST;
        System.out.println("방향: " + direction.getArrow()); // 출력: 방향: →
        System.out.println(direction.getName()); // EAST (Enum 클래스에 정의)
    }
}
```

Annotation

- 프로그램의 소스 코드 안에 다른 프로그램을 위한 정보를 미리 약속된 형식으로 포함시킨 것
- 프로그래밍 언어에 영향을 미치지 않으면서 다른 프로그램에 유용한 정보를 제공할 수 있다는 장점이 있음
- @Override
    - 메서드 앞에만 붙일 수 있는 annotation → 컴파일러에게 오버라이딩했다는 정보 제공
- @Deprecated
    - 더 이상 사용되지 않음을 알림
- @FunctionalInterface
    - 컴파일러가 함수형 인터페이스를 올바르게 선언했는지 체크함
- @SuppressWarnings
    - 묵인해야하는 경고 메시지를 나타나지않게 해줌
    - `@SuppressWarnings({”deprecation”, “unchecked”, “varargs”, "rawtypes"})`
- @SafeVarargs
    - 메서드에 선언된 가변인자의 타입이 non-refiable타입 (컴파일 후에 제거되는 타입)인경우 unchecked 경고 억제

meta annotation

- annotation을 위한 annotation
- target이나 retention을 지정하는데 사용
- @target
    - 적용가능한 대상을 지정하는데 사용됨
    - `@Target({ANNOTATION_TYPE, CONSTRUCTOR, FIELD, LOCAL_VARIABLE, METHOD, PACKAGE, PARAMETER, TYPE, TYPE_PARAMETER, TYPE_USE})`
- @Retention
    - annotation이 유지되는 기간을 지정
    - SOURCE(소스 파일에만 존재), CLASS(클래스 파일에 존재, 실행시 사용불가, 기본값), RUNTIME(클래스 파일에 존재, 실행시 사용 가능)
    - `@Retention(RetentionPolicy.SOURCE)`

- @Documented
    - annotation에 대한 정보가 javadoc으로 작성한 문서에 포함되도록 함
- @Inherited
    - annotation이 자손 클래스에 상속되도록 함
    
    ```java
    @Inherited
    @interface SupperAnno {}
    
    @SuperAnno
    class Parent {}
    
    class Child extends Parent {} // Child에 annotation이 붙은 것으로 인식
    ```
    
- @Repeatable
    - 한 대상에 여러번 annotation을 적용할 때
    
    ```java
    @Repeatable(ToDos.class)
    @interface ToDo {
    	String value();
    }
    
    @Todo("string1")
    @Todo("string2")
    class MyClass { }
    
    // 일반적으로 annotation들을 하나로 묶어서 다룰 수 있는 annotation을 추가로 정의해야함
    @interface ToDos {
    	ToDo[] value();
    }
    ```
    
    ```java
    import java.lang.annotation.Annotation;
    import java.util.Arrays;
    
    public class Main {
        public static void main(String[] args) {
            Class<MyClass> clazz = MyClass.class;
    
            // 1. ToDos 컨테이너 어노테이션을 직접 가져오는 방법 (Java 8 이전 스타일, 드물게 사용)
            ToDos todosAnnotation = clazz.getAnnotation(ToDos.class);
            if (todosAnnotation != null) {
                ToDo[] todoArray = todosAnnotation.value();
                Arrays.stream(todoArray).forEach(todo -> System.out.println("ToDo (ToDos 컨테이너): " + todo.value()));
            }
    
            // 2. 반복 가능한 ToDo 어노테이션을 직접 가져오는 방법 (일반적인 사용법)
            ToDo[] todoAnnotations = clazz.getAnnotationsByType(ToDo.class);
            Arrays.stream(todoAnnotations).forEach(todo -> System.out.println("ToDo: " + todo.value()));
    
            // 3. 모든 어노테이션을 가져와서 ToDo 어노테이션만 필터링하는 방법 (유연하지만 오버헤드 발생 가능)
            Annotation[] annotations = clazz.getAnnotations();
            Arrays.stream(annotations)
                    .filter(annotation -> annotation instanceof ToDo)
                    .map(annotation -> (ToDo) annotation)
                    .forEach(todo -> System.out.println("ToDo (모든 어노테이션): " + todo.value()));
        }
    }
    
    // 출력 결과:
    // ToDo: string1
    // ToDo: string2
    // ToDo (모든 어노테이션): string1
    // ToDo (모든 어노테이션): string2
    ```
    

- @Native
    - native method에 의해 참조되는 constant field(상수 필드)에 붙이는 annotation
    - native method는 JVM이 설치된 OS의 메서드를 뜻함
    - Object 클래스의 메서드들이 대부분 native method임

Annotation 타입 정의

```java
@interface annotation_name {
	타입 요소이름();
	int count() default 1; // 기본값 null 제외하고 지정가능
	String value(); // String value만 단독으로 있는 경우 annotation 적용시 요소 이름 생략가능
}
```

- annotation의 요소 (element)
    - 반환값이 있고 매개변수는 없는 추상 메서드 형태
    - enum, 자신이 아닌 다른 annotation 등을 포함할 수 있음

Marker Annotation

- 값을 지정할 필요가 없는 경우