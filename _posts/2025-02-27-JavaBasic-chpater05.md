---
title: 자바의 정석 Chapter 5
description: 모르는 내용 위주 학습 기록
author: laze
date: 2025-02-27 00:06:00 +0900
categories: [Dev, Java]
tags: [Java]
---
# Chapter05

배열의 복사

System.arraycopy(num, 0, newNum, 0, num.length);

원본배열, 시작인덱스, 복사할배열, 시작인덱스, 복사할길이

String class 주요 메서드

char chartAt(int index) → 해당 index에 있는 문자 반환

String substring(int from, int to) - from~to 에 있는 문자열 반환 (to는 포함되지 않음)

char[] toCahrArray() → 문자열을 문자 배열로 변환해서 반환