---
title: 자바의 정석 Chapter 11
description: 모르는 내용 위주 학습 기록
author: laze
date: 2025-02-27 00:06:00 +0900
categories: [Dev]
tags: [Java]
---
# Chapter 11

List 인터페이스

- 중복을 허용
- 저장 순서가 유지됨

Set 인터페이스

- 중복을 허용하지 않음
- 저장순서 유지되지 않음

Map 인터페이스

- 키, 값 을 하나의 쌍으로 묶어서 저장하는 클래스
- 키는 중복허용하지 않음

Map.Entry  인터페이스

- Map에 저장되는 key-value 쌍을 다루기 위해 내부적으로 Entry 인터페이스를 정의함

ArrayList

- 배열에서 중간에 있는 데이터를 삭제하면 삭제할 객체 바로 아래부터 복사해서 배열을 복사하기 때문에 가장 마지막 데이터를 삭제하는게 아닌 이상 이런 연산이 일어나 수행시간이 증가할 수 있음

LinkedList

```java
class Node {
	Node next; // 다음 요소의 주소 저장
	Object obj; // 데이터를 저장
}
```

- 불연속적으로 존재하는 데이터를 서로 연결한 형태로 구성되어있음
- 단방향이라 이전 요소에 대한 접근이 어려움 → DoubleLinkedList = LinkedList + 이전요소의 주소
- 첫값과 끝값을 이으면 doubly circular linked list 가 됨
- 순차적으로 추가 및 삭제는 ArrayList가 빠르며 중간에서 추가 삭제가 일어나는 경우는 LinkedList가 빠름
- 읽는 속도는 ArrayList가 빠르고 LinkedList가 느림 다음번 요소의 주소를 찾아서 원하는 주소까지 찾아야 하므로
    - 처음 데이터 작업시에는 ArrayList로 사용하다가 중간에 삽입 삭제 연산이 일어날때 LinkedList로 옮겨서 작업하는 방식으로 프로세스 고려 가능함

Stack

- LIFO

Queue

- FIFO
- PriorityQueue → 저장한 순서에 관계 없이 우선순위가 높은 것부터 꺼낼 때 사용
- Deque(Double-Ended Queue) → 양쪽끝에서 추가 삭제 가능함
    - 구현체로는 ArrayDequeue, LinkedList 가 있음

Iterator, ListIterator, Enumeration

- 컬렉션에 저장된 요소를 접근하는데 사용됨
- Iterator
    - hasNext, next, remove 인터페이스가 존재함
- ListIterator → Iterator + 이전방향으로 이동 가능

Arrays

- 배열을 다루는데 유용한 메서드들이 정의되어있음
- copyOf(), copyOfRange() - 배열 전체 / 일부를 복사해서 새로운 배열을 만들어 반환
    - Arrays.copyOf(arr, arr.length);
- fill() → 배열의 모든 요소를 지정된 값으로 채움
    - Arrays.fill(arr,9);
- setAll() → 배열을 채우는데 사용할 함수형 인터페이스를 매개변수로 받음
    - Arrays.setAll(arr, () → (int)(Math.random() * 5) + 1);
- sort()
    - 배열 정렬
- binarySearch()
    - 배열 검색
    - int idx = Arrays.binarySearch(arr, 2);
- asList(Object… a)
    - List list = Arrays.asList(new Integer[]{1,2,3,4,5});
    - List list = Arrays.asList(1,2,3,4,5);
    - 크기 변경 불가 내용 변경 가능
        - 크기 변경이 필요한 경우 List lit = new ArrayList(Arrays.asList(1,2,3,4,5)); 처럼 사용해야함

Comparator & Comparable

- 정렬시 사용

| **특징** | **Comparable** | **Comparator** |
| --- | --- | --- |
| 위치 | java.lang 패키지 | java.util 패키지 |
| 목적 | 객체 자신의 기본 정렬 기준 설정 | 객체 비교를 위한 별도의 비교자 제공 |
| 메서드 | compareTo(T o) | compare(T o1, T o2) |
| 구현 위치 | 정렬 대상 클래스 내 | 별도 클래스 또는 람다식 |
| 사용 시점 | 기본 정렬 기준이 필요한 경우 | 다양한 정렬 기준이 필요하거나, 기본 정렬 기준이 없을 때 |
| 변경 가능성 | 객체 자체를 수정해야 함 | 별도 비교자를 생성하여 유연하게 변경 가능 |
- **객체 자체에 자연스러운 정렬 순서가 있다면:** Comparable을 구현하여 기본 정렬 기준을 설정
    - public class Person implements Comparable<Person>
    - Override → compareTo
- **객체를 다양한 기준으로 정렬해야 하거나, 객체 자체의 기본 정렬 기준을 변경하고 싶지 않다면:** Comparator를 사용하여 다양한 비교자를 생성
    - Comparator<Person> ageReverseComparator = (p1, p2) -> p2.getAge() - p1.getAge();
    - Collections.sort(people, ageReverseComparator);
- 그외 Comparator 상수로 제공하는 값들 활용가능
    - Arrays.sort(strArr, String.CASE_INSENSITIVE_ORDER); // 대소문자 구분 없이 정렬

HashSet

- 중복된 요소를 저장하려하면 false 반환 및 중복 저장 안됨
- 저장한 순서를 유지하고싶으면  LinkedHashSet 사용
- hashCode() 와 equals() 를 오버라이딩해야함 그게 아니면 객체는 주소값 그대로 비교해서 중복이라 판단안함

TreeSet

- 중복 저장 허용하지 않음
- 정렬된 위치에 저장하기때문에 저장 순서도 유지하지 않음

```java
class TreeNode {
	TreeNode left; // 왼쪽 자식노드
	Object element; // 객체를 저장하기위한 참조 변수
	TreeNode right; // 오른쪽 자식노드
}
```

- TreeSet에 저장되는 객체는 Comparable을 구현하거나 Comparator를 제공해야함
- 범위 검색과 정렬에 유리함 (subSet(from, to), headSet, tailSet)
- 추가 삭제에는 시간이 오래걸림

HashMap

- Key 와 Value 를 묶어서 하나의 entry로 저장
- Entry라는 내부 클래스를 정의하고 다시 Entry 타입의 배열을 선언해 사용함
- getOrDefault(Object key, Object defaultValue);
- 해싱 - 해시함수를 이용해 데이터를 해시테이블에 저장하고 검색하는 기법
    - 검색하고자하는 값의 키로 해시함수 호출
    - 해시코드로 해당 값이 저장되어있는 링크드 리스트를 찾음
    - 링크드리스트에서 검색한 키와 일치하는 데이터를 찾음
- Object 클래스에 정의된 hashCode()는 객체의 주소를 이용하는 알고리즘이라 모든 객체에 대해 hashCode()를 호출한 결과가 유일함
- String은 문자열의 내용으로 해시코드를 만들어냄

TreeMap

- 이진 검색트리의 형태로 키와 값의 쌍으로 이루어진 데이터를 저장함
- 검색과 정렬에 적합한 컬렉션 클래스
- TreeMap(Comparator c) - Comparator 기준으로 정렬하는 TreeMap 객체 생성
- Map.Entry ceilingEntry(Object key) - 지정된 키와 일치하거나 큰 것중 제일 작은 것의 Map.Entry 반환
    - floorEntry → 일치하거나 작은 것중 제일 큰 Map.Entry 반환

Properties

- Hashtable을 상속받아 구현한 것으로 HashMap<Object,Object> ↔ Properties<String,String>
- 주로 애플리케이션의 환경설정과 관련된 속성을 저장하는데 사용

```java
Properties prop = new Properties();

prop.setPrpoerty(key, value);

prop.list(System.out); // prop에 저장된 요소들을 출력
```

- 그 외에도 XML로 저장, 읽기 등이 가능함
    - loadFromXML, storeToXML

Collections

- 멀티쓰레드 환경에서 동기화 처리
    - synchronizedCollection(Collection c)
- 변경 불가 컬렉션 ( 읽기 전용)
    - unmodifiableCollection(Collection c)
- 싱글톤 컬렉션
    - singletonList(Object o) / singleton(Object o) (set) / singletonMap(Object key, Object value)
- 한 종류의 객체만 저장하는 컬렉션
    - checkedCollection(Collection c, Class type)