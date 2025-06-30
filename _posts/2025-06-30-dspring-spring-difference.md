---
title: -Dspring.profiles.active <-> --spring.profiles.active
description: 
author: laze
date: 2025-06-30 00:00:02 +0900
categories: [Dev, SpringBoot]
tags: [SpringBoot]
---
### -D와 --의 차이점
-Dspring.profiles.active <-> --spring.profiles.active

이 둘은 자바 애플리케이션에 외부 설정을 전달하는 방식에서 근본적인 차이가 있습니다.

-Dkey=value (JVM 시스템 속성 - System Property):

누구에게 전달?: 자바 가상 머신(JVM) 전체에 전달되는 설정입니다.

언제 사용?: 애플리케이션의 동작 환경 자체를 설정할 때 사용합니다. (예: -Dfile.encoding=UTF-8)

어떻게 접근?: 자바 코드에서 System.getProperty("key")로 접근합니다.

IntelliJ 위치: VM options 필드에 입력해야 합니다.

--key=value (애플리케이션 인자 - Application Argument):

누구에게 전달?: main 메소드의 String[] args 파라미터를 통해 애플리케이션 자체에 직접 전달되는 설정입니다.

언제 사용?: Spring Boot처럼 애플리케이션이 스스로 인자를 해석하여 동작을 바꾸도록 설계되었을 때 사용합니다.

IntelliJ 위치: Program arguments 필드에 입력해야 합니다.

결론: Spring Boot는 --spring.profiles.active라는 애플리케이션 인자를 받도록 설계되었기 때문에, Program arguments 필드에 -- 형식으로 입력해야만 정상적으로 프로필을 인지할 수 있습니다.
