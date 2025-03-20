---
title: 자바의 정석 Chapter 9
description: 모르는 내용 위주 학습 기록
author: laze
date: 2025-02-27 00:00:09 +0900
categories: [Dev, Java]
tags: [Java]
---
# Chapter 09

Object 클래스

| method | description |
| --- | --- |
| protected Object clone() | 객체 자신의 복사본 반환 |
| public boolean equals(Object obj) | 객체 자신과 객체 obj가 같은 객체인지 여부 |
| protected void finalize() | 객체 소멸시 수행될 코드 |
| public Class getClass() | Class 정보를 담고있는 인스턴스 반환 |
| public int hashCode() | 객체 자신의 해시코드 반환 |
| public String toString() | 객체 자신의 정보를 문자열로 반환 |
| public void notify() | 객체 자신을 사용하려 기다리는 쓰레드 하나를 꺠운다 |
| public void notifyAll() | 객체 자신을 사용하려 기다리는 모든 쓰레드를 깨운다 |
| public void wait() | 다른 쓰레드가 notify(), notifyAll()을 호출할때까지 현재 쓰레드를 무한히 혹은 지정된 시간 동안 기다리게 함 |
| public void wait(long timeout) |  |
| public void wait(long timeout, int nanos) |  |

equlas(Object obj) → 객체의 참조변수를 받아서 비교하므로 다른 객체면 false 반환됨

주로 주소가 아닌 저장된 내용을 비교하는 방식으로 오버라이딩해서 사용함

hashCode()

- 객체의 주소값으로 해시코드를 만들어 반환
- 서로 다른 두 객체의 해시코드는 일반적으로는 다르지만 64bit JVM에서는 8byte 주소값으로 4byte 해시코드를 만들기 때문에 중복 가능성이 존재함
- 서로 같은 객체라면 equals() 뿐아니라 hashCode()도 같아야함
- String(”abc”) 를 두번 선언하고 hashCode()를 비교하면 동일함
- identityHashCode()를 사용하면 동일하지 않음
    - String 클래스에서는 문자열이 동일하면 동일한 해시코드를 반환하도록 hashCode()메서드가 오버라이딩 되어있음
    - Systme.identityHashCode(Object obj)는 객체의 주소값으로 해시코드 생성하기때문에 다른 값 반환됨

toString()

```java
public String toString() {
	return getClass().getName()+"@"+Integer.toHexString(hashCode());
}
```

clone()

단순히 인스턴스 변수의 값만 복제하기때문에 참조타입의 인스턴스 변수가 있는 경우

완전한 복제를 기대하기 어려움

이럴때는 clone() 메서드를 오버라이딩하여 사용해야 함

```java
class Point implements Cloneable { // Cloneable 인터페이스를 구현한 클래스만 clone() 호출 가능
	int x,y;
	
	Point(int x, int y) {
		this.x = x;
		this.y = y;
	}
	
	public Object clone() {
		Object obj = null;
		try {
			obj = super.clone();
		} catch (CloneNotSupportedException e) { }
		
		return obj;
	}
	
	public Point clone() { // 공변 반환타입(convariant return type) -> 조상메서드 반환타입을 자식 클래스 타입으로 변경
		Object obj = null;
		try {
			obj = super.clone();
		} catch (CloneNotSupportedException e) { }
		
		return (Point)obj;
	}
}
```

getClass()

ClassLoader에 의해 메모리에 올라갈 때 자동으로 생성됨

classpath에 지정된 경로를 따라 클래스 파일을 찾고 없으면 ClassNotFoundException

Class 객체 얻는 법

Class cObj 

- new Card().getClass() - 생성된 객체로부터 얻는법
- Card.class - 클래스 리터럴 *.class 로 부터 얻는 법
- Class.forName(”Card”) - 클래스 이름으로부터 얻는 법
- Class.forName 에서 Class의 Fullname → 정규화된 이름으로 사용해야함 패키지명.클래스명
    
    네, `Class.forName()` 메서드가 클래스 파일을 찾는 과정에 대한 아주 좋은 질문입니다. 핵심은 `Class.forName()`에 전달하는 **클래스 이름은 반드시 패키지 이름을 포함한 정규화된 이름(Fully Qualified Name)**이어야 한다는 점입니다.
    
    **`Class.forName()`의 동작 방식**
    
    1. **정규화된 클래스 이름 요구:** `Class.forName()` 메서드는 인자로 전달받는 문자열을 클래스의 정규화된 이름으로 간주합니다. 정규화된 이름은 **패키지 이름 + 클래스 이름** 형태로 구성됩니다.
        - 예시: `com.example.cards.Card`, `org.util.Date`, `java.lang.String`
        - 단순한 클래스 이름(`Card`)만으로는 어떤 패키지에 속한 클래스인지 알 수 없기 때문에 `Class.forName()`은 오류를 발생시킵니다.
    2. **클래스 로더 사용:** `Class.forName()` 메서드는 JVM의 클래스 로더를 사용하여 해당 클래스를 찾고 메모리에 로드합니다. 클래스 로더는 클래스패스에 지정된 경로를 따라 클래스 파일을 탐색합니다.
    3. **클래스 로딩:** 클래스 파일이 발견되면 클래스 로더는 해당 클래스 파일을 읽어들여 JVM 내부에 클래스 정보를 생성합니다. 이렇게 생성된 클래스 정보가 `Class` 객체로 표현됩니다.
    
    **`Class.forName("Card")`가 오류를 발생시키는 이유**
    
    - `Class.forName("Card")`와 같이 단순히 클래스 이름만 전달하면, JVM은 어떤 패키지에 속한 `Card` 클래스를 로드해야 할지 알 수 없습니다.
    - 따라서 `ClassNotFoundException` 예외가 발생합니다.
    
    **올바른 사용 방법**
    
    `Class.forName()`을 사용하여 클래스를 로드하려면, 반드시 해당 클래스의 정규화된 이름을 인자로 전달해야 합니다.
    
    ```java
    // 예시
    Class<?> cardClass1 = Class.forName("package1.Card"); // package1.Card 클래스 로드
    Class<?> cardClass2 = Class.forName("package2.Card"); // package2.Card 클래스 로드
    
    ```
    
    **클래스 경로(Classpath)와 `Class.forName()`**
    
    - **클래스 경로:** 클래스 경로는 JVM이 클래스 파일을 찾을 때 사용하는 파일 시스템 경로 목록입니다. 일반적으로 JAR 파일이나 디렉터리를 클래스 경로에 추가합니다.
    - **탐색 순서:** JVM은 클래스 경로에 지정된 경로들을 순서대로 탐색하여 해당 클래스 파일을 찾습니다. 만약 여러 곳에 동일한 정규화된 이름을 가진 클래스 파일이 존재하면, 먼저 발견된 파일을 사용합니다.
    - **패키지 구조:** 패키지 구조는 클래스 경로 내에서 디렉터리 구조로 표현됩니다. 예를 들어, `com.example.cards` 패키지에 속한 `Card` 클래스는 클래스 경로에서 `com/example/cards/Card.class`와 같은 경로에 존재합니다.
    
    **정리**
    
    - `Class.forName()` 메서드는 반드시 클래스의 정규화된 이름을 인자로 요구합니다.
    - 클래스 이름은 패키지 이름 + 클래스 이름 형태로 구성됩니다.
    - `Class.forName("Card")`와 같이 단순한 클래스 이름만으로는 JVM이 어떤 클래스를 로드해야 할지 알 수 없습니다.
    - 클래스 경로는 JVM이 클래스 파일을 찾을 때 사용하는 파일 시스템 경로 목록입니다.
    
    질문하신 것처럼, `Class.forName()`은 클래스의 정규화된 이름을 기반으로 클래스를 찾아 로드합니다. 따라서 패키지명이 중요한 역할을 합니다.
    
    더 궁금한 점이 있으시면 언제든지 질문해주세요!
    

String

immutable 클래스

“+” 연산시 인스턴스 내 문자열이 바뀌는게 아니라 그때마다 새 String 인스턴스가 생기는 것

예를들어 AAA라는 문자열을 String str1, str2, str3 에서 String s1 = “AAA”; 형식으로 할당하면

“AAA”를 담고있는 str1,2,3은 모두 같은 참조변수를 갖게됨 “AAA”는 JVM 내의 constant pool에 저장됨

join() ↔ split()

`String[] split(String regex)` str.split(”,”)→ “dog,cat,bear” → {”dog”, “cat”, “bear”}

`String join(String[] arr)` arr.join(”-”)→ “dog-cat-bear”

StringJoiner

```java
StringJoiner sj = new StringJoiner(",", "[", "]"); // ,구분자 / [ = 처음 / ] = 끝
String[] arr = {"aaa", "bbb", "ccc"};

for (string s : strArr) {
	sj.add(s);
}

System.out.println(sj.toString()); // [aaa,bbb,ccc]
```

getBytes(String charsetName) → 문자열의 문자 인코딩을 다른 인코딩으로 변경

byte[] utf8_str = “가”.getBytes(”UTF-8”);

String str = new String(utf8_str, “UTF-8”);

String.format() -형식화된 문자열을 만들어내는 방법, printf()와 사용법 동일

```java
String str = String.format(”%d 더하기 %d”, 3, 5);
System.out.println(str); // 3 더하기 5
```

기본형 ↔ 문자열

`String.valueOf(int, boolean …)`

`Integer.parseInt(String s)` 

StringBuffer → 내부적으로 문자열 편집을 위한 버퍼를 가지고 있음

equals의 경우 String은 오버라이딩되어서 문자열비교를하지만 StringBuffer는 오버라이딩되어있지 않음

굳이 비교하려면 toString으로 String객체에 담아서 비교

thread safe

StringBuilder → StringBuffer에서 thread safe 제거된 버전, 멀티쓰레드가 아닌 경우 더 높은 성능 제공

java.util.Objects

- 모든 메서드가 static
- `static boolean isNull(Object obj)`
- `static boolean nonNull(Object obj)`
- `static <T> T requireNonNull(T obj, String message)` → null이면 NullPointerException 발생시킴
- `static int compare(Object a, Object b, Comparator c)` → 같으면 0 크면 양수 작으면 음수 반환
- `static boolean equals(Object a, Object b)` → null 검사를 하지 않아도 equals 내부에서 null 검사를 해줌
- `static boolean deepEquals(Object a, Object b)` → 2차원 배열같이 객체를 재귀적으로 비교해줌

java.util.Random

- System.currentTimeMillis() 를 종자값으로함
- Random(long seed) 로 시드값 설정 가능
- boolean, int 등등 new Random().nextBoolean() 형태로 랜덤값 반환 가능
    - nextInt(int n) → 0부터 n 사이의 정수를 반환함
- 같은 종자값을 갖는 Random 인스턴스는 시스템이나 실행시간에 관계없이  항상 같은 값을 같은 순서로 반환할 것을 보장함

정규식

1. 정규식을 매개변수로 Pattern 클래스의 static 메서드인 Pattern compile(String regex)를 호출하여 Pattern 인스턴스를 얻는다
    1. Pattern p = Pattern.compile(”c[a-z]*”);
2. 정규식으로 비교할 대상을 매개변수로 Pattern 클래스의 Matcher matcher(CharSequence input)을 호출해서 Matcher 인스턴스를 얻는다.
    1. Matcher m = p.matcher(data[i]);
3. Matcher 인스턴스에 boolean matches()를 호출해서 정규식에 부합하는지 확인한다
    1. if(m.matches())

정규식 그룹화

```java
String source = "HP:011-1111-1111, HOME:02-999-9999";
String pattern = “(0\\d{1,2})-(\\d{3,4})-(\\d{4})”;

Pattern p = Pattern.compile(pattern);
Matcher m = p.matcher(source);

int i = 0;
while(m.find()) {
	System.out.println( ++i + ": " + m.group() + " - > " + m.group(1) + ", "
	+ m.group(2) + ", " + m.group(3));
}
// 1: 011-1111-1111 -> 011, 1111, 1111
// 2: 02-999-9999 -> 02, 999, 9999
// group(), group(n) 으로 전체 혹은 n번째 그룹을 반환할 수 있음
// find()는 패턴과 일치하는 부분 찾으면 true, 없으면 false 반환
```

StringTokenizer → 구분자가 한 개고 정규식 사용안할때 사용됨 그외 String의 split 등을 사용

```java
String source = "100,200,300,400";
StringTokenizer st = new StringTokenizer(source, ",");

while(st.hasMoreTokens()){
	System.out.println(st.nextToken());
}
```

java.math.BigInteger

- 내부적으로 int 배열을 사용해서 큰 값을 다룰 수 있으나 성능은 떨어짐
- BigInteger는 불변이므로 String 처럼 새로운 인스턴스르 반환함
- 비트단위 연산 메서드 존재함

java.math.BigDecimal

- 정수 x 10 ^ -scale ( 0 ≤ scale ≤ Integer.MAX_VALUE)
- 정수를 저장하는데 BigInteger 사용함
- double 타입의 값을 매개변수로 갖는 생성자를 사용하면 오차 발생할 수 있음
    - new BigDecimal(0.1) → 0.100000000000000000000555111…

123.45 = 12345 * 10 ^ -2 ⇒ 정수 12345, scale 2, precision 5 (정수의 전체 자리수 의미)