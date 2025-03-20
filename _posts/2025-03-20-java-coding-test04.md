---
title: 코딩 테스트 필수 문법
description: Coding Test Java
author: laze
date: 2025-03-20 00:00:06 +0900
categories: [Dev, CodingTest]
tags: [CodingTest, Java]
---
# 코딩 테스트 필수 문법

**primitive type / reference type**

**부동소수형**

10.0 % 3.2 ⇒ 0.3999999999947 ≠ 0.4

부동소수형 데이터를 이진법으로 표현하기때문에 오차가 발생하며 이를 epsilon 엡실론이라고함

collection framework

List, Stack, Queue, ArrayDeque, Hashmap

**배열**

```java
import java.util.Arrays;

public class Solution {
	public static void main(String[] args) {
		int[] array = { 1, 2, 3, 4, 5 };
		int[] array2 = new int[] { 1, 3, 5, 7, 9 };
		int[] array3 = new int[5];
		
		array3[0] = 0;
		array3[1] = 2;
		array3[2] = 4;
		array3[3] = 6;
		array3[4] = 8;
		
		System.out.println(Arrays.toString(array));
		System.out.println(Arrays.toString(array2));
		System.out.println(Arrays.toString(array3));
```

인덱스를 이용한 배열의 접근, 변경 시간복잡도 O(1)

**리스트**

일반적으로 ArrayList 의미

새 데이터 맨 뒤에 삽입시 O(1) , 기존 데이터 중간에 삽입 또는 삭제시 O(N)

```java
ArrayList<Integer> list = new ArrayList<>();

list.add(1);
list.add(2);

System.out.println(list.get(1)); // 2
System.out.println(list); // [1,2]
```

**해시맵**

key, value 쌍을 저장하는 해시 테이블로 구현됨

```java
HashMap<String, Integer> map = new HashMap<>();

map.put("apple",1);
map.put("banana",2);
map.put("orange",3);

System.println(map); // {banana=2, orange=3, apple=1}

// 해시맵 데이터 검색
String key = "apple";

if(map.containsKey(key)) {
	int value = map.get(key);
	System.out.println(key + ": " + value);
}
else {
	System.out.println(key + "는 해시맵에 없습니다.);
}

// 해시맵 수정
map.put("banana", 4);
System.out.println(map); // {banana=4, orange=3, apple=1}

// 해시맵 삭제
map.remove("orange");
System.out.prinln(map); // {banana=4, apple=1}
```

**문자열**

문자들을 배열의 형태로 구성한 immutable 객체

```java
// 문자열 초기화
String string = "Hello, World!";

// 문자열 추가, 삭제 -> 기존 객체를 수정하는 것이 아니라 새로운 객체를 반환하는 것임
String string = "He";
string += "llo";
System.out.println(string); // "Hello"

// 문자열 수정
String string = "Hello";
string = string.replace("l","");
System.out.println(string); // "Heo"
```

```java
String s = "abc";
System.out.println(System.identityHashCode(s)); // 1808253012
s += "def";
System.out.println(System.identityHashCode(s)); // 589431969
System.out.println(s); // abcdef

// 새로운 String s 객체를 생성 -> s가 가진 "abc" 값을 하나씩 복사 -> 뒤에 "def" 저장
// 시간 복잡도 O(N)
```

이 과정을 String을 사용하게되면 생성 → 복사 + 저장 무한 반복이 되므로

Mutable한 객체인 StringBuilder / StringBuffer 를 사용함

아래와 같은 코드를 String 과 StringBuilder를 사용한 경우 6.5 ↔ 0.005초 로 많은 차이가 발생하게됨

```java
long start = System.currentTimeMillis();

String s = "";

for (int i = 1; i <= 100000; i++) {
	s += i;
}

long end = System.currentTimeMillis();

System.out.println(((end - start) / 1000.0) + "초");
```

```java
long start = System.currentTimeMillis();

StringBuilder s = new StringBuilder();

for (int i = 1; i <= 100000; i++) {
	s.append(i);
}

long end = System.currentTimeMillis();

System.out.println(((end - start) / 1000.0) + "초");
```

멀티 스레드 환경에서 Thread-Safe 여부로 나뉘는데 StringBuilder가 Thread-Safe가 없어 미세하게 빠름

StringBuilder 클래스 활용법

```java
StringBuilder sb = new StringBuilder();

sb.append(10);
sb.append("ABC);

System.out.println(sb); // 10ABC
sb.deleteCharAt(3); // 3번째 인덱스 문자 삭제
System.out.println(sb); // 10AC
sb.insert(1,2); // 1번째 인덱스에 2라는 문자 추가
System.out.println(sb); // 120AC
```

**메서드**

클래스 내부에 정의한 함수

```java
public int function_name(int param1, int param2) {
	// 메서드 실행코드
	
	return result; // 반환값
}
```

람다식

- 자바 1.8 버전에서 추가
- 익명 함수 (annonymous function)
- 간결하게 표현가능하고 가독성이 좋아짐

```java
private static class Node {
	int dest, cost;
	
	public Node(int dest, int cost) {
		this.dest = dest;
		this.cost = cost;
	}
}

public static void main(String[] args) {
	Node[] nodes = new Node[5];
	nodes[0] = new Node(1, 10);
	nodes[1] = new Node(2, 20);
	nodes[2] = new Node(3, 15);
	nodes[3] = new Node(4, 5);
	nodes[4] = new Node(1, 25);
	
	Arrays.sort(nodes, (o1, o2) -> Integer.compare(o1.cost, o2.cost));
	// 밑의 Arrays.sort 와 동일한 기능 수행
	// Integer.compare(int x, int y)
	// x < y -> -1 반환
	// x > y -> 1 반환
	// x = y -> 0 반환
	
	Arrays.sort(nodes, new Comparator<Node>() {
		@Override
		public int compare(Node o1, Node o2) {
			return Integer.compare(o1.cost, o2.cost);
		}
	});
}
```

**구현 노하우**

조기 반환 (early return)

코드 실행 과정이 함수 끝까지 도달하기 전에 반환하는 기법

```java
public static void main(String[] args) {
	System.out.println(totalPrice(4, 50));
}

static int totalPrice(int quantity, int price) {
	int total = quantity * price;
	if (total > 100)
		return (int)(total * 0.9);
	return total;
}
```

보호 구문 (guard clauses)

본격적인 로직을 진행하기 전에 예외 처리 코드를 추가하는 기법

```java
import java.util.List;

static double calculateAverage(List<Integer> numbers) {
	if (numbers == null)
		return 0;
	
	if (numbers.isEmpty())
		return 0;
		
	int total = numbers.stream().mapToInt(i -> i).sum();
	return (double) total / numbers.size();
}
```

제네릭 (generic)

- 빌드 레벨에서 타입을 체크하여 타입 안전성 제공
- 타입 체크와 형변환을 생략할 수 있게 해주어 코드를 간결하게 만듦
- 여러 타입의 데이터를 하나의 컬렉션에 넣어야하는 경우는 거의 없으므로 일반적으로 제네릭으로 타입을 강제하여 실수 방지하는 편이 좋음

```java
List list = new ArrayList();
list.add(0);
list.add("abc");

int sum1 = (int)list.get(0) + (int)list.get(1); // 런타임 오류 발생

List<Integer> genericList = new ArrayList<>();
genericList.add(10);
genericList.add("abc"); // 빌드레벨에서 오류 발생

int sum2 = genericList.get(0) + genericList.get(1);
```