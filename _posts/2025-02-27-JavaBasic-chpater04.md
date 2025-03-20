---
title: 자바의 정석 Chapter 4
description: 모르는 내용 위주 학습 기록
author: laze
date: 2025-02-27 00:00:03 +0900
categories: [Dev, Java]
tags: [Java]
---
# Chapter04

switch 문의 제약조건

switch 문의 조건식 결과는 정수 또는 문자열

case문의 값은 정수 상수만 가능 중복X

enhanced for statment

for ( 타입 변수명 : 배열 또는 컬렉션 )

break문 자신이 포함된 가장 가까운 반복문을 벗어남

continue 반복문의 끝으로 이동하여 다음 반복으로 넘어감

이름 붙은 반복문

→ 여러 개의 반복문이 중첩될 경우 중첩 반복문을 한번에 벗어날때 사용

```java
Loop1 : for (int i = 2; i <= 9; i++) {
	for(int j = 1; j <= 9; j++) {
		if(j==5)
			break Loop1;
			break;
			continue Loop1;
			continue;
			System.out.println(i + " * " + j + " = " + i*j);
		}
		System.out.println();
	}
```