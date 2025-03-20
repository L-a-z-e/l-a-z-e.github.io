---
title: 자바의 정석 Chapter 6
description: 모르는 내용 위주 학습 기록
author: laze
date: 2025-02-27 00:00:05 +0900
categories: [Dev, Java]
tags: [Java]
---
# Chapter 06

| 변수 종류 | 선언 위치 | 생성시기 |
| --- | --- | --- |
| class variable | 클래스영역 | 클래스가 메모리에 올라갈 때 |
| instance variable | 클래스영역 | 인스턴스가 생성될 때 |
| local variable | 클래스영역 이외 | 변수 선언문이 수행되었을 때 |

```java
class Variables {
	int iv; // 인스턴스 변수
	static int cv; // 클래스 변수
	//// 클래스 영역
	
	void method() {
		int lv = 0; // 지역변수
	}
	//// 메서드 영역
```

JVM 메모리 구조

- Method Area
    - *.class 파일을 읽어 Class Data를 이곳에 저장함
        - 이 시점에서 class variable도 이 영역에 생성됨
- Heap
    - 인스턴스가 생성되는 공간
    - 인스턴스 변수도 이 영역에 생성된다
- Call Stack
    - 메서드 작업에 필요한 메모리 공간 제공
    - 메서드가 수행되는 동안 지역변수 + 연산의 중간 결과등을 저장하는데 사용
    - 작업을 마치면 할당되었던 메모리공간 반환

메서드 호출시

기본형 매개변수 → 값복사 → read only

참조형 매개변수 → 주소복사 → read & write

static method ↔ instance method

static method 에는 인스턴스 변수를 사용할 수 없음

메서드 내에서 인스턴스 변수를 사용하지 않는다면 static을 붙이는게

호출 시간이 짧아지므로 성능 향상됨

(인스턴스 메서드는 실행시 호출되어야할 메서드를 찾는 과정이 추가적으로 필요함)

가변인자 (varargs)

`타입… 변수명` 으로 선언 (Object… args)

매개변수 중 제일 마지막에 선언해야 함

내부적으로 배열을 이용하는 비효율성이 있으므로 필요한 경우에만 사용

delim, delimeter → 구분자

생성자 → 인스턴스가 생성될 때 호출되는 인스턴스 초기화 메서드

- 클래스 이름과 같아야한다
- 생성자는 리턴 값이 없다
- 오버로딩 가능 ( 매개변수 없는 생성자, 있는 생성자 )

new → 수행 순서

`Card c = new Card();`

1. new에 의해 메모리(heap)에 Card 클래스의 인스턴스가 생성됨
2. 생성자 Card()가 호출되어 수행됨
3. 연산자 new의 결과로 생성된 Card 인스턴스의 주소가 반환되어 참조변수 c에 저장

기본 생성자는 컴파일러가 클래스에 생성자가 하나도 정의되지 않은 경우 자동으로 추가해줌

`클래스이름() {}`

 

생성자에서 다른 생성자 호출

- 생성자의 이름으로 클래스이름 대신 this를 사용
- 한 생성자에서 다른 생성자를 호출할 때는 반드시 첫 줄에서만 호출 가능

```java
class Car {
	String color;
	String gearType;
	int door;
	
	Car() {
		this("white", "auto", 4);
	}
	
	Car(String color) {
		this(color, "auto", 4);
	}
	
	Car(String color, String gearType, int door) {
		this.color = color;
		this.gearType = gearType;
		this.door = door;
	}
}
```

this - 인스턴스 자신을 가리키는 참조변수 (인스턴스의 주소 저장), 모든 인스턴스 메서드에 지역변수로 숨겨진채 존재

this(), this(매개변수) - 생성자, 같은 클래스의 다른 생성자 호출 시 사용

변수의 초기화

- 멤버 변수(클래스, 인스턴스 변수)와 배열의 초기화는 선택적
- 지역 변수의 초기화는 필수적
- 참조형 변수의 기본값은 null임

멤버 변수 초기화 방법

- 명시적 초기화
- 생성자
- 초기화 블럭