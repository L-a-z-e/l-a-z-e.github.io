---
title: Jakarta, Springframework Import
description: 
author: laze
date: 2025-06-28 00:00:03 +0900
categories: [Dev, SpringBoot]
tags: [SpringBoot]
---
네, 이 질문은 현대 자바와 스프링 생태계의 중요한 역사적 배경을 이해하는 데 있어 매우 핵심적인 질문입니다. 이 둘의 차이를 명확히 알면, 앞으로 어떤 어노테이션을 선택해야 할지 절대 헷갈리지 않게 될 것입니다.

> springframework / jakarta 중 무엇을 import 해야하는지
>

---

### **결론부터: 무조건 `jakarta.persistence`를 선택해야 합니다.**

우리가 사용하는 **Spring Boot 3.x 버전 이상**에서는, JPA와 관련된 모든 어노테이션(`@Entity`, `@Id`, `@Table` 등)은 반드시 **`jakarta.persistence.*`** 패키지에 있는 것을 사용해야 합니다.

### **왜 그렇게 바뀌었을까? "Java EE의 이사"**

이 이야기를 이해하려면, 자바의 역사에 대한 짧은 배경지식이 필요합니다.

**1. 과거: `javax.persistence`의 시대**

- 과거에 자바의 기업용(Enterprise) 기술 표준은 **Java EE(Java Enterprise Edition)** 라는 이름으로 불렸고, 이 기술들은 모두 **Oracle**이 관리했습니다.
- 이때, JPA(Java Persistence API)의 표준 명세도 Java EE에 포함되어 있었고, 모든 관련 클래스와 어노테이션들은 **`javax.persistence.*`** 라는 패키지 이름 안에 있었습니다. (`x`는 e**x**tension을 의미)
- Spring Boot 2.x 버전대까지는 모두 이 `javax` 패키지를 사용했습니다.

**2. 변화의 시작: Eclipse Foundation으로의 이전**

- Oracle은 Java EE 기술 스택 전체를 **Eclipse 재단**이라는 오픈소스 재단에 기부하기로 결정했습니다. 더 개방적인 방식으로 커뮤니티와 함께 발전시키기 위함이었죠.
- **문제 발생:** 하지만 Oracle은 'Java'라는 상표권(Trademark)은 넘겨주지 않았습니다. 따라서 Eclipse 재단은 더 이상 'Java EE'라는 이름을 쓸 수 없게 되었습니다.

**3. 현재: `jakarta.persistence`의 시대**

- Eclipse 재단은 이 프로젝트의 새로운 이름으로 **"Jakarta EE"**를 선택했습니다.
- 상표권 문제 때문에, 기존의 `javax.*` 라는 패키지 이름도 더 이상 사용할 수 없게 되었습니다. 그래서 모든 패키지 이름을 **`jakarta.*`*로 변경하기로 결정했습니다.
- **`javax.persistence`** -> **`jakarta.persistence`**
- **`javax.servlet`** -> **`jakarta.servlet`**
- 이 큰 변화는 **Spring Framework 6**와 **Spring Boot 3**부터 전면적으로 적용되었습니다.

**요약:**

- **`javax.*`**: 과거의 이름. Spring Boot 2.x / Java EE 시절의 유산.
- **`jakarta.*`**: 현재의 이름. Spring Boot 3.x / Jakarta EE 시대의 표준.

---

### **그렇다면, `org.springframework.data.annotation.Id`는 무엇인가?**

당신이 올린 스크린샷에는 세 번째 선택지가 있었습니다. 바로 **`org.springframework.data.annotation.Id`** 입니다.

- **역할:** 이것은 **JPA와는 관련이 없는**, Spring Data 프로젝트 자체에서 사용하는 `@Id` 어노테이션입니다.
- **주요 사용처:**
  - **JPA가 아닌 다른 데이터 저장소**와 연동할 때 사용됩니다.
  - 예를 들어, **MongoDB (`@Document`)**, **Redis**, 또는 우리가 사용 중인 **Elasticsearch (`@Document`)**의 ID를 지정할 때 바로 이 `@Id`를 사용합니다.
  - 우리가 `ProductDocument.java`를 만들 때, `id` 필드 위에 붙였던 `@Id`가 바로 `org.springframework.data.annotation.Id` 입니다.

### **최종 정리: 언제 무엇을 선택해야 하는가?**

이 세 가지를 구별하는 명확한 규칙은 다음과 같습니다.

1. **`@Entity`가 붙은 클래스 (JPA 엔티티) 안에서 ID를 지정할 때:**
  - **무조건 `jakarta.persistence.Id`를 선택합니다.**
2. **`@Document`가 붙은 클래스 (Elasticsearch, MongoDB 등) 안에서 ID를 지정할 때:**
  - **무조건 `org.springframework.data.annotation.Id`를 선택합니다.**
3. **`javax.persistence.Id`는 언제 사용하는가?**
  - **절대 사용하지 않습니다.** (당신이 Spring Boot 2.x 버전의 아주 오래된 프로젝트를 유지보수하는 경우가 아니라면)

### Transactional ⇒ springframework ? jakarta ?

### **결론부터: `@Transactional`은 `springframework`의 것을 사용하는 것이 맞습니다.**

왜 그럴까요? 그 이유는 `@Transactional`의 **역할과 주체**가 다르기 때문입니다.

### **1. 두 `@Transactional`의 정체**

- **`jakarta.transaction.Transactional`:**
  - 이것은 **Jakarta EE 표준(과거 Java EE)**에 정의된, 트랜잭션 처리에 대한 **'명세(Specification)' 또는 '약속'**입니다.
  - "자바 애플리케이션에서 트랜잭션을 사용하려면, 이런 어노테이션을 이런 방식으로 사용해야 한다"는 **규칙**만 정의해 놓은 것입니다.
  - 이 어노테이션 자체에는 트랜잭션을 실제로 시작하고, 커밋하고, 롤백하는 **코드가 들어있지 않습니다.** 그냥 '이름표'일 뿐입니다.
- **`org.springframework.transaction.annotation.Transactional`:**
  - 이것은 **스프링 프레임워크**가 위 표준을 포함하여, **자체적으로 제공하는 훨씬 더 강력하고 다양한 기능을 담아 만든 '구현체'**입니다.
  - 이 어노테이션을 사용하면, 스프링의 **AOP(관점 지향 프로그래밍)** 기능이 동작하여, 메소드 시작 전에 트랜잭션을 열고, 메소드 종료 후에 커밋하거나 예외 발생 시 롤백하는 **실제 동작 코드**가 실행됩니다.

### **2. 왜 `springframework`의 것을 써야 하는가?**

**"스프링의 모든 생태계는 `springframework.transaction.annotation.Transactional`을 중심으로 설계되었기 때문입니다."**

스프링의 `@Transactional` 어노테이션은 `jakarta`의 표준 어노테이션이 제공하지 못하는, 매우 유용하고 강력한 추가 속성들을 제공합니다.

- **`readOnly`**: 읽기 전용 트랜잭션을 설정하여 성능을 최적화합니다. (우리가 매우 유용하게 사용 중이죠)
- **`propagation`**: 여러 트랜잭션 메소드가 서로 호출될 때, 트랜잭션을 어떻게 전파하고 관리할지 그 동작 방식을 세밀하게 제어할 수 있습니다. (e.g., `REQUIRED`, `REQUIRES_NEW`)
- **`isolation`**: 트랜잭션의 격리 수준을 설정할 수 있습니다.
- **`rollbackFor` / `noRollbackFor`**: 어떤 특정 예외가 발생했을 때만 롤백하거나, 롤백하지 않도록 예외를 지정할 수 있습니다.
- **스프링 테스트와의 연동:** `@Transactional`을 테스트 코드에 붙이면 테스트가 끝난 후 자동으로 롤백되는 기능 등, 스프링의 다른 기능들과 매우 긴밀하게 통합되어 있습니다.

### **3. `@Id`와의 근본적인 차이점**

- **`@Id` (`jakarta.persistence.Id`)**:
  - 이것은 **JPA 표준**의 일부입니다. JPA 구현체(Hibernate)가 "아, 이 필드가 데이터베이스의 기본 키(PK)구나" 라고 **'인지'**하기 위해 필요한, 순수한 **'메타데이터'**입니다.
  - 실제 동작(PK 생성 등)은 JPA 구현체와 데이터베이스가 담당합니다.
- **`@Transactional` (`org.springframework.transaction.annotation.Transactional`)**:
  - 이것은 **스프링 프레임워크**가 제공하는 **'기능'**입니다. 스프링의 AOP 프록시가 이 어노테이션을 보고, 트랜잭션을 관리하는 **'실제 로직'**을 실행합니다.

**요약:**`@Id`는 JPA에게 "이것은 ID야" 라고 알려주는 **'정보성 이름표'**의 역할이 강하고, `@Transactional`은 스프링에게 "이 메소드는 트랜잭션으로 감싸서 실행해 줘" 라고 명령하는 **'동작 지시서'**의 역할이 강합니다. 우리는 스프링 프레임워크 위에서 개발하고 있으므로, 당연히 스프링이 제공하는 강력하고 모든 기능과 통합된 **'동작 지시서'**를 사용하는 것이 맞습니다.

이제 `@Id`는 왜 `jakarta`의 것을, `@Transactional`은 왜 `springframework`의 것을 사용해야 하는지에 대한 차이가 명확하게 이해되셨을 겁니다.
