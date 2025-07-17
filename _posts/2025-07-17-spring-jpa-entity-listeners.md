---
title: @EntityListeners
description: 
author: laze
date: 2025-07-17 00:00:01 +0900
categories: [Dev, SpringJpa]
tags: [SpringJpa]
---
# [Spring Data JPA] `@EntityListeners`와 Auditing: `createdAt` 자동화의 비밀

## 1. 서론: 반복되는 `setCreatedAt(LocalDateTime.now())` 코드

데이터베이스를 다루는 거의 모든 애플리케이션에서는 데이터의 생성 시각(`createdAt`)과 마지막 수정 시각(`updatedAt`)을 기록하는 것이 필수적입니다. 이 정보는 데이터의 히스토리를 추적하고 문제를 디버깅하는 데 중요한 단서가 됩니다.

하지만 이 필드들을 관리하기 위해, 우리는 종종 다음과 같은 반복적인 코드를 작성하게 됩니다.

```java
// 안 좋은 예시: 개발자가 직접 시간을 설정
@Service
public class UserService {
    public void createUser(UserDto dto) {
        User user = new User();
        // ... 비즈니스 로직 ...
        user.setCreatedAt(LocalDateTime.now()); // <--- 반복
        user.setUpdatedAt(LocalDateTime.now()); // <--- 반복
        userRepository.save(user);
    }

    public void updateUser(Long id, UserUpdateDto dto) {
        User user = userRepository.findById(id).orElseThrow();
        // ... 비즈니스 로직 ...
        user.setUpdatedAt(LocalDateTime.now()); // <--- 반복, 누락하기 쉬움
        // userRepository.save(user) -> Dirty Checking
    }
}

```

이 방식은 몇 가지 명백한 문제점을 가집니다.

- **코드 중복:** 모든 `save`, `update` 로직에 동일한 코드가 반복됩니다.
- **누락 위험:** 개발자가 깜빡하고 시간 설정 코드를 빼먹기 쉽습니다.
- **책임 분리 위배:** 생성/수정 시간 설정은 기술적인 관심사이지, 핵심 비즈니스 로직이 아닙니다.

Spring Data JPA의 Auditing 기능은 이 모든 문제를 어노테이션 몇 개로 매우 우아하게 해결해 줍니다.

## 2. JPA Auditing의 배우들: 각 어노테이션의 역할

JPA Auditing의 동작 원리를 하나의 잘 짜인 연극에 비유해 봅시다. 각 어노테이션과 클래스는 무대 위에서 각자의 역할을 수행합니다.

| 역할 | 컴포넌트 | 설명 |
| --- | --- | --- |
| **총괄 제작자** | **`@EnableJpaAuditing`** | "자, 이제부터 우리 프로젝트에서 Auditing 기능을 시작하겠습니다!" 라고 선언하여 전체 기능을 활성화하는 총책임자. |
| **무대** | **`@Entity` 클래스** | 데이터의 생성, 수정 등 모든 사건이 발생하는 공간. |
| **감독** | **`@EntityListeners(...)`** | 무대에서 벌어지는 사건(이벤트)을 특정 배우(리스너)에게 전달하라고 지시하는 역할. |
| **배우** | **`AuditingEntityListener.class`** | 감독의 지시를 받아, 실제 동작(시간 주입)을 수행하는 주체. Spring Data JPA가 기본으로 제공. |
| **대본** | **`@CreatedDate`, `@LastModifiedDate`** | 배우가 어떤 필드에, 어떤 값을 채워야 하는지 알려주는 상세한 지시서. |

이 배우들이 모두 제 역할을 해야만 Auditing이라는 연극이 성공적으로 막을 올릴 수 있습니다.

## 3. 동작 원리: `save()`가 호출된 후 벌어지는 일들

개발자가 `userRepository.save(user)`를 호출했을 때, 데이터베이스에 `INSERT` 쿼리가 날아가기까지 그 뒤에서 어떤 일이 벌어지는지 단계별로 따라가 보겠습니다.

1. **[설정]** 먼저, 메인 애플리케이션 클래스에 총괄 제작자(`@EnableJpaAuditing`)를 선언하여 Auditing 기능을 활성화합니다.

    ```java
    @EnableJpaAuditing // JPA Auditing 기능 활성화!
    @SpringBootApplication
    public class MyApplication { ... }

    ```

2. **[준비]** `User` 엔티티에 감독(`@EntityListeners`)과 대본(`@CreatedDate`, `@LastModifiedDate`)을 준비합니다.

    ```java
    @Entity
    @EntityListeners(AuditingEntityListener.class) // 감독이 배우(리스너)를 지정
    public class User {
        // ... 다른 필드들 ...
    
        @CreatedDate // 생성 시각 대본
        @Column(updatable = false) // 생성 시간은 수정되면 안 됨
        private LocalDateTime createdAt;
    
        @LastModifiedDate // 수정 시각 대본
        private LocalDateTime updatedAt;
    }
    ```

3. **[실행]** `UserService`에서 `userRepository.save(newUser)`를 호출합니다.
4. **[이벤트 발생]** JPA의 영속성 컨텍스트가 `newUser` 객체를 관리하기 시작하고, DB에 `INSERT` 쿼리를 보내기 직전에 **`@PrePersist`** 라는 생명주기 이벤트를 발생시킵니다.
5. **[이벤트 감지 및 처리]**
  - `User` 엔티티에 지정된 감독(`@EntityListeners`)이 이 이벤트를 포착합니다.
  - 감독은 지정된 배우인 `AuditingEntityListener`에게 이벤트를 전달합니다.
  - `AuditingEntityListener`는 `newUser` 객체의 필드를 스캔하여 대본(`@CreatedDate`, `@LastModifiedDate`)을 찾습니다.
  - `@CreatedDate`가 붙은 `createdAt` 필드에 현재 시각을 주입합니다.
  - `@LastModifiedDate`가 붙은 `updatedAt` 필드에도 현재 시각을 주입합니다.
6. **[DB 저장]** 이제 `createdAt`과 `updatedAt` 값이 모두 채워진 `newUser` 객체를 바탕으로 최종 `INSERT` 쿼리가 생성되어 DB로 전송됩니다.

만약 이미 존재하는 데이터를 수정하는 경우, `@PreUpdate` 이벤트가 발생하고 `AuditingEntityListener`는 `@LastModifiedDate` 필드만 새로운 시간으로 갱신합니다. (`@CreatedDate`는 `updatable = false` 속성 때문에 변경되지 않습니다.)

## 4. 실전 적용: `BaseTimeEntity`로 코드 중복 제거하기

모든 엔티티에 `createdAt`, `updatedAt` 필드와 `@EntityListeners`를 반복적으로 추가하는 것은 비효율적입니다. 실무에서는 이러한 공통 필드를 추상 클래스로 분리하여 상속받는 방식을 사용합니다.

`@MappedSuperclass` 어노테이션을 사용하면, 이 클래스를 테이블로 매핑하지는 않되, 이 클래스를 상속받는 자식 엔티티에게 필드 정보만 물려줄 수 있습니다.

### `BaseTimeEntity.java`

```java
import jakarta.persistence.EntityListeners;
import jakarta.persistence.MappedSuperclass;
import lombok.Getter;
import org.springframework.data.annotation.CreatedDate;
import org.springframework.data.annotation.LastModifiedDate;
import org.springframework.data.jpa.domain.support.AuditingEntityListener;

import java.time.LocalDateTime;

@Getter
@MappedSuperclass // 이 클래스를 상속받는 엔티티에게 필드만 물려준다.
@EntityListeners(AuditingEntityListener.class) // 리스너를 여기에 적용
public abstract class BaseTimeEntity {

    @CreatedDate
    @Column(updatable = false)
    private LocalDateTime createdAt;

    @LastModifiedDate
    private LocalDateTime updatedAt;
}

```

### `User.java`

이제 `User` 엔티티는 `BaseTimeEntity`를 상속받기만 하면 됩니다.

```java
@Entity
public class User extends BaseTimeEntity { // 상속받기만 하면 끝!

    @Id
    @GeneratedValue
    private Long id;

    private String name;

    // createdAt, updatedAt 필드는 여기에 선언할 필요가 없다.
}

```

이제 어떤 엔티티든 `BaseTimeEntity`를 상속하면, 별도의 설정 없이 생성/수정 시각이 자동으로 관리됩니다.

## 5. 심화 학습: 생성자/수정자 자동화 (`@CreatedBy`, `@LastModifiedBy`)

시간뿐만 아니라, **"누가"** 이 데이터를 생성하고 수정했는지 기록하고 싶을 때도 Auditing 기능을 사용할 수 있습니다.

1. `BaseEntity`에 필드를 추가합니다.

    ```java
    @CreatedBy
    @Column(updatable = false)
    private String createdBy;
    
    @LastModifiedBy
    private String updatedBy;
    
    ```

2. 현재 로그인한 사용자의 정보를 제공하는 `AuditorAware` 구현체를 만듭니다.
   이제 `@EnableJpaAuditing` 설정이 이 `AuditorAware` Bean을 찾아, 데이터를 생성/수정할 때마다 현재 사용자의 정보를 `createdBy`, `updatedBy` 필드에 자동으로 주입해줍니다.

    ```java
    @Configuration
    public class JpaConfig {
        @Bean
        public AuditorAware<String> auditorProvider() {
            // 실제로는 Spring Security의 SecurityContextHolder를 사용하여
            // 현재 로그인한 사용자의 ID나 이름을 반환해야 한다.
            return () -> Optional.of("adminUser"); // 여기서는 예시로 "adminUser" 하드코딩
        }
    }
    
    ```


## 6. 결론

`@EntityListeners(AuditingEntityListener.class)`와 Spring Data JPA Auditing 기능은 단순히 코드를 줄여주는 것을 넘어, **횡단 관심사(Cross-cutting Concerns)**인 '생성/수정 시간 기록'을 핵심 비즈니스 로직과 완벽하게 분리시켜주는 강력한 도구입니다.

이는 JPA의 생명주기 이벤트를 특정 로직(Listener)과 연결하여, 데이터베이스에 저장되기 직전의 엔티티를 조작할 수 있게 해주는 JPA의 강력한 기능을 Spring이 사용하기 쉽게 포장해준 것입니다. 이 기능을 적극적으로 활용하여 더 깔끔하고 유지보수하기 쉬운 코드를 작성해 보시기 바랍니다.
