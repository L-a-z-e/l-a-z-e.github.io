---
title: 자바의 정석 Chapter 10
description: 모르는 내용 위주 학습 기록
author: laze
date: 2025-02-27 00:00:10 +0900
categories: [Dev, Java]
tags: [Java]
---
# Chapter 10

Date - jdk 1.0 부터 제공

Calendar - jdk 1.1 부터 제공

`Calendar cal = Calender.getInstance();` - 추상클래스라서 new Calender() x

GregorianCalendar(공용) / BuddhistCalendar(태국)

Date ↔ Calender 변환

`Date d = new Date(cal.getTimeInMillis())`

`Calender cal = Calender.getInstance;`

`cal.setTime(d);`

`Calendar.DAY_OF_WEEK`

1: 일요일, 2: 월요일, …, 7:토요일

`Calender.MONTH`

0: 1월, 1: 2월, … 11: 12월

형식화 클래스

DecimalFormat

- 0, #, ., -, , ;(패턴구분자 ex- #,###.##+;#,###.##-)

`DecimalFormat df = new DecimalFormat(”#.#E0”);`

`String result = df.format(number);`

Number 클래스는 Integer, Double 같은 래퍼클래스의 조상

`Number num = df.format(number);`

`double d = num.doubleValue();` 형식으로 사용 가능

SimpleDateFormat → 날짜 데이터 원하는 형태로 출력

`SimpleDateFormat simpleDateFormat = new SimpleDateFormat(”yyyy-MM-dd HH:mm:ss.SSS);`

ChoiceFormat → 특정 범위에 속하는 값을 문자열로 변환

```java
double[] limits = {60, 70, 80, 90}; // 경계값 오름차순 정렬
String[] grades = {"D", "C", "B", "A"}; // limits 개수와 일치해야함

int[] score = {100, 95, 88, 70, 52, 60, 70};
ChoiceFormat choiceFormat = new ChoiceFormat(limits, grades);

for (int i = 0; i < scores.length; i++) {
	System.out.println(scores[i] + " : " + choiceFormat.format(scores[i]));
}
```

MessageFormat → 데이터를 정해진 양식에 맞게 출력

```java
String msg = "Name: {0} \nTel: {1} \nAge: {2} \nBirthday: {3}";

Object[] arguments = {"laze","010-0000-0000","9510"};

String result = MessageFormat.format(msg, arguments);
System.out.println(result);
```

java.time 패키지

LocalTime - 시간

LocalDate - 날짜

LocalDateTime - 시간&날짜

ZonedDateTime - LocalDateTime + 시간대

Period = 날짜 - 날짜

Duration = 시간 - 시간

now(), of() 를 사용해서 객체 생성

Temporal, TemporalAccessor, TemporalAdjuster 구현한 클래스

- LocalDate, LocalTime, LocalDateTime, ZonedDateTime, Instant

TemporalAmount 구현한 클래스

- Period, Duration

LocalDate birthDate = LocalDate.ofYearDay(1999,365); // 1999년 12월 31일

LocalDate birthDate = LocalDate.parse(”1999-12-31”);

그외 필요한 내용은 필요할 때 찾는게 도움될 듯