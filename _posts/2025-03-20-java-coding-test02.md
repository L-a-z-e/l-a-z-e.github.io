---
title: 코딩 테스트 효율적으로 준비하기
description: Coding Test Java
author: laze
date: 2025-03-20 00:00:04 +0900
categories: [Dev, CodingTest]
tags: [CodingTest, Java]
---
# 코딩 테스트 효율적으로 준비하기

언어 선택하기

- 알아야 하는 수준
    - 변수 선언하기
    - 함수 정의하기
    - 컬렉션 자료형 다루기
    - 조건문, 반복문 사용하기

문제 분석 연습하기

- 문제를 쪼개서 분석
- 제약 사항을 파악하고 테스트 케이스를 추가
    - 어떤 알고리즘을 사용할 지 고민할 때 유용하고, 코드 구현 단계에서 예외를 거를 때 도움 됨
- 입력값 분석
    - 시간 복잡도는 입력값이 결정하는 경우가 많음
    - 불가능한 알고리즘을 걸러내는 데 도움 됨
- 그리디하게 접근할 때는 근거를 명확하게 할 것
    - 그리디 - 현재 상황에서 가장 유리해 보이는 선택을 하는 것
    - 가장 많이 하는 실수는 그렇게 접근하지 말아야 할 문제를 그리디하게 접근하는 것
- 데이터 흐름이나 구성 파악
    - 삽입과 삭제가 빈번하게 일어나는 경우 힙 자료구조 고려

의사 코드로 설계하는 연습

- 프로그램의 논리를 설명하고 알고리즘을 표현하기 위해 작성한 일종의 지침
- 원칙
    1. 프로그래밍 언어로 작성하면 안됨
    2. 일반인도 이해할 수 있는 자연어로 작성해야 함
    3. 일정한 형식이 없음
    - 세부 구현이 아닌 동작 중심으로 작성
        - ex) 국어, 영어, 수학 점수를 입력 받는다 ↔ 크기가 256 바이트인 문자배열 3 개를 선언해서 표준 입력으로 국어, 영어, 수학 점수를 입력받는다.
        - 프로그래밍 요소는 의사 코드에 추가하면 안됨
    - 문제 해결 순서로 작성
    - 충분히 테스트 할 것