---
title: 자바의 정석 Chapter 7
description: 모르는 내용 위주 학습 기록
author: laze
date: 2025-02-27 00:06:00 +0900
categories: [Dev, Java]
tags: [Java]
---
# Chapter 07

상속에서 생성자와 초기화 블럭은 상속되지 않고 멤버만 상속됨

Composite(포함관계) 를 맺어서도 재사용이 가능함

is-a → 상속관계

has-a → 포함관계

상속에서 오버라이딩

- 조상 클래스의 메서드보다 좁은 범위로 변경할 수 없음
- 조상 클래스의 메서드보다 많은 수의 예외를 선언할 수 없음

super → 자손 클래스에서 조상 클래스로부터 상속받은 멤버를 참조하는데 사용되는 참조변수

super() → 조상 클래스의 생성자

컴파일러에서 자동적으로 super() 를 생성자의 첫 줄에 삽입함 (명시한 적이 없는 경우)

package

클래스의 실제 full name 은 패키지명을 포함한 것임

String → java.lang.String

이렇게 구분하는이유는 동일한 클래스명도 다른 패키지에 존재할 수 있기때문

지정하지 않는 경우 unnamed package 에 속함 (자바에서 기본 제공)

import문에서

import java.util.*;

import java.text.*;

를 import java.*; 로 통합할 수는 없음

→ 하위 패키지의 클래스까지 포함한다는 의미가 아니기때문에

*을 사용한다고해서 성능상의 차이는 없음

static import → static 멤버를 호출할 때 클래스 이름을 생략할 수 있게 해줌

import static java.lang.Math.random; → Math.random() 을 random() 으로 사용가능

final → 변경될 수 없는

인터페이스는 인터페이스로부터만 상속가능 → extends

클래스에서는 implements

인터페이스를 이용한 다형성

다른것과 마찬가지로 인터페이스 타입의 참조변수이므로 인터페이스의 메서드만 사용가능

instance of 로 확인해서 명시적 형변환을 다시한 뒤 사용하는 방법도 존재

`Fightable f = new Boxer();` → Fightable 인터페이스 타입의 참조변수

리턴 타입이 인터페이스라는 것은 해당 인터페이스를 구현한 클래스의 인스턴스를 반환한다는 의미

인터페이스의 이해

클래스를 사용하는 쪽(User) - A

클래스를 제공하는 쪽(Provider) - B

B의 methodB 선언부가 변경되면 사용하는 A도 변경되어야함

A - B 관계

```java
class A {
	public void methodA(B b) {
		b.methodB();
	}
}

class B {
	public void methodB() {
		System.out.prinln("methodB()");
	}
}

class InterfaceTest {
	public static void main(String args[]) {
		A a = new A();
		a.methodA(new B());
	}
}
```

A - B 직접적인 관계 → A - I - B 간접적인 관계로 변경됨 실제로 사용하는 B라는 클래스를 몰라도 I를 통해 사용가능하며 I의 변경에 따른 영향만 받게됨

```java
interface I {
	public abstract void methodB();
}

class B implements I {
	public void methodB() {
		System.out.println("methodB in B class");
	}
}

class A {
	public void methodA(I i) {
		i.methodB();
	}
}
```

디폴트메서드

여러 인터페이스의 디폴트 메서드간 충돌 시

→ 인터페이스를 구현한 클래스에서 디폴트 메서드를 오버라이딩해야 함

디폴트 메서드와 조상 클래스 메서드 간의 호출

→ 조상 클래스의 메서드가 상속되고 디폴트 메서드는 무시됨

⇒ 필요한 쪽의 메서드를 오버라이딩하면 해결됨

inner class

- 내부 클래스에서 외부 클래스의 멤버들을 쉽게 접근할 수 있다
- 코드의 복잡성을 줄일 수 있다.