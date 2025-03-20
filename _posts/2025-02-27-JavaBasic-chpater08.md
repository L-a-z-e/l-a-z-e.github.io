---
title: 자바의 정석 Chapter 8
description: 모르는 내용 위주 학습 기록
author: laze
date: 2025-02-27 00:00:08 +0900
categories: [Dev, Java]
tags: [Java]
---
# Chapter 08

에러 → 프로그램 코드에 의해서 수습될 수 없는 심각한 오류

예외 → 프로그램 코드에 의해서 수습될 수 있는 오류

printStackTrace() → 예외 발생 당시 Call Stack에 있던 메서드 정보와 예외 메시지를 출력

getMessage() → 발생한 예외 클래스의 인스턴스에 저장된 메시지 출력

try-with-resources

try문의 괄호 안에 객체를 생성하는 문장을 넣으면 이 객체는 따로 close()를 호출하지 않아도 try 블럭을 벗어나는 순간 close()가 호출됨

chained exception

`Throwable initCause(Throwable cause)`

`Throwable getCause()`

예외 A가 예외 B를 발생시켰을때

```java
try{
	startInstall();
	copyFiles(); // SpaceException 발생
}
catch (SpaceException e) {
	InstallException ie = new InstallException("설치중 예외발생");
	ie.initCause(e); // InstallException의 원인 예외를 SpaceException 으로 지정
	throw ie;
}
```