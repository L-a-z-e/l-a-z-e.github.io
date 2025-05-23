---
title: Spring Framework Architecture
description: 
author: laze
date: 2025-04-30 00:00:02 +0900
categories: [Dev, SpringBoot]
tags: [SpringBoot]
---
# Spring Design Philosophy

- **모든 레벨에서 선택권 제공:**  스프링은 설계 결정을 가능한 한 늦출 수 있게 합니다. 예를 들어, 코드 변경 없이 설정을 통해 영속성 제공자(persistence provider)를 바꿀 수 있습니다. 다른 많은 인프라 문제나 서드파티 API 통합에서도 마찬가지입니다.
- **다양한 관점 수용:**  스프링은 유연성을 중시하며, 특정 방식만을 고집하지 않습니다(not opinionated). 다양한 관점을 가진 광범위한 애플리케이션 요구를 지원합니다.
- **강력한 하위 호환성 유지:**  스프링의 발전은 버전 간 호환성 문제가 거의 발생하지 않도록 신중하게 관리되어 왔습니다. 스프링에 의존하는 애플리케이션과 라이브러리의 유지보수를 용이하게 하기 위해 엄선된 범위의 JDK 버전과 서드파티 라이브러리를 지원합니다.
- **API 디자인 중시:**  스프링 팀은 직관적이고 여러 버전과 오랜 세월에 걸쳐 유지될 수 있는 API를 만들기 위해 많은 생각과 시간을 투자합니다.
- **높은 코드 품질 기준 설정:**  스프링 프레임워크는 의미 있고, 최신이며, 정확한 javadoc을 강력히 강조합니다. 패키지 간 순환 의존성이 없는 깨끗한 코드 구조를 가진 몇 안 되는 프로젝트 중 하나입니다.
