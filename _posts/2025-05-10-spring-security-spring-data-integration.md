---
title: Integrations - Spring Data
description: 
author: laze
date: 2025-05-10 00:00:02 +0900
categories: [Dev, SpringSecurity]
tags: [SpringSecurity]
---
## Spring Data 통합 (Spring Data Integration)

Spring Security는 쿼리 내에서 현재 사용자를 참조할 수 있도록 하는 Spring Data 통합을 제공합니다.

결과를 나중에 필터링하는 것은 확장되지 않으므로 페이징된 결과를 지원하기 위해 쿼리에 사용자를 포함하는 것은 유용할 뿐만 아니라 필요합니다.

### Spring Data 및 Spring Security 구성

이 지원을 사용하려면 `org.springframework.security:spring-security-data` 의존성을 추가하고 `SecurityEvaluationContextExtension` 타입의 빈을 제공해야 합니다. Java 구성에서는 다음과 같습니다:

**Java**

```java
@Bean
public SecurityEvaluationContextExtension securityEvaluationContextExtension() {
	return new SecurityEvaluationContextExtension();
}
```

**Kotlin**

```kotlin
@Bean
fun securityEvaluationContextExtension(): SecurityEvaluationContextExtension {
    return SecurityEvaluationContextExtension()
}
```

XML 구성에서는 다음과 같습니다:

```xml
<bean class="org.springframework.security.data.repository.query.SecurityEvaluationContextExtension"/>
```

### @Query 내 보안 표현식 (Security Expressions within @Query)

이제 쿼리 내에서 Spring Security를 사용할 수 있습니다. 예를 들어:

**Java**

```java
@Repository
public interface MessageRepository extends PagingAndSortingRepository<Message,Long> {
	@Query("select m from Message m where m.to.id = ?#{ principal?.id }")
	Page<Message> findInbox(Pageable pageable);
}
```

**Kotlin**

```kotlin
@Repository
interface MessageRepository : PagingAndSortingRepository<Message,Long> {
    @Query("select m from Message m where m.to.id = ?#{ principal?.id }")
    fun findInbox(pageable: Pageable): Page<Message>
}
```

이는 `Authentication.getPrincipal().getId()`가 `Message`의 수신자와 같은지 확인합니다.

이 예는 principal을 `id` 속성을 가진 객체로 사용자 정의했다고 가정합니다.

`SecurityEvaluationContextExtension` 빈을 노출함으로써 모든 공통 보안 표현식(Common Security Expressions)을 쿼리 내에서 사용할 수 있습니다.

---

### 🔍 내 데이터만 쏙! (Spring Data와 Spring Security의 똑똑한 만남)

우리가 웹 애플리케이션을 만들다 보면, "현재 로그인한 사용자가 작성한 게시글만 보여주기", "현재 로그인한 사용자에게 온 메시지만 보여주기" 처럼 **"현재 사용자"와 관련된 데이터만** 가져와야 하는 경우가 아주 많아요.

이때 Spring Data (JPA, MongoDB 등 데이터를 다루는 기술들의 모음)와 Spring Security를 함께 사용하면, 데이터베이스 쿼리문 자체에 "현재 로그인한 사용자의 정보"를 쉽게 넣을 수 있는 방법이 있습니다!

### 1. 왜 필요할까요? (그냥 다 가져와서 나중에 거르면 안 되나요? 🤔)

"그냥 DB에서 모든 메시지를 다 가져온 다음에, 프로그램 코드에서 현재 사용자에게 온 것만 골라내면 되지 않나?" 라고 생각할 수도 있어요. 하지만 이런 방식은 데이터가 많아지면 심각한 문제가 생깁니다:

- **성능 저하:** 수백만 건의 메시지를 DB에서 전부 가져와서 메모리에서 일일이 확인하는 건 매우 느리고 비효율적이에요.
- **페이징 처리의 어려움:** "1페이지에 10개씩 보여줘" 같은 페이징 처리를 할 때, 전체 데이터를 가져와서 필터링한 후 다시 페이징하는 것은 거의 불가능에 가깝습니다. (예: 100만 건 중 현재 사용자 메시지가 100건인데, 첫 페이지 10건을 보려고 100만 건을 다 가져올 순 없죠!)

그래서 **애초에 DB에서 데이터를 가져올 때부터 "현재 사용자" 조건을 포함해서 필요한 만큼만 가져오는 것이 훨씬 효율적이고 올바른 방법**입니다.

### 2. 준비물 챙기기 🛠️ (설정 방법)

이 편리한 기능을 사용하려면 두 가지 준비가 필요해요.

1. **의존성 추가:** `pom.xml` (Maven) 또는 `build.gradle` (Gradle) 파일에 다음 의존성을 추가해야 합니다.
  - `org.springframework.security:spring-security-data`
  - 이 친구가 Spring Security와 Spring Data를 연결해주는 다리 역할을 해요.
2. **`SecurityEvaluationContextExtension` 빈(Bean) 등록:** Spring 설정 파일(보통 Java Configuration 클래스)에 다음과 같이 빈을 하나 만들어줘야 합니다.

    ```java
    // Java Configuration
    import org.springframework.context.annotation.Bean;
    import org.springframework.context.annotation.Configuration;
    import org.springframework.security.data.repository.query.SecurityEvaluationContextExtension;
    
    @Configuration
    public class SecurityConfig { // 또는 다른 설정 클래스
    
        @Bean
        public SecurityEvaluationContextExtension securityEvaluationContextExtension() {
            return new SecurityEvaluationContextExtension();
        }
    }
    ```

  - 이 빈은 Spring Data 쿼리에서 Spring Security의 특별한 표현식(변수 같은 것들)을 사용할 수 있도록 도와주는 마법사 같은 역할을 해요.

### 3. `@Query` 안에서 현재 사용자 정보 사용하기! ✨ (예시)

자, 이제 준비가 끝났으니 실제로 어떻게 사용하는지 봅시다.

Spring Data JPA에서 Repository 인터페이스에 `@Query` 어노테이션으로 직접 JPQL(Java Persistence Query Language) 쿼리를 작성할 때 이 기능을 쓸 수 있어요.

**예시: 현재 로그인한 사용자에게 온 메시지 목록 가져오기**

```java
import org.springframework.data.domain.Page;
import org.springframework.data.domain.Pageable;
import org.springframework.data.jpa.repository.Query;
import org.springframework.data.repository.PagingAndSortingRepository;
import org.springframework.stereotype.Repository;

@Repository
public interface MessageRepository extends PagingAndSortingRepository<Message, Long> {

    // 여기가 핵심! ?#{ principal?.id }
    @Query("select m from Message m where m.to.id = ?#{ principal?.id }")
    Page<Message> findInbox(Pageable pageable);

}
```

위 코드에서 가장 중요한 부분은 `?#{ principal?.id }` 입니다.

- **`?#{ ... }`**: Spring Expression Language (SpEL)을 사용하는 특별한 문법이에요.
- **`principal`**: Spring Security가 관리하는 **현재 로그인한 사용자 정보 객체(Principal)**를 의미해요. (보통 `UserDetails` 인터페이스를 구현한 객체나, 우리가 커스텀한 사용자 정보 객체가 들어있어요.)
- **`?.id`**: `principal` 객체가 null이 아닐 경우에만 `id` 속성에 접근하겠다는 의미예요 (안전한 호출). 만약 `principal`이 `User` 클래스의 인스턴스이고 `User` 클래스에 `getId()` 메소드가 있다면, 이 부분은 `principal.getId()`와 같이 동작해서 현재 로그인한 사용자의 ID 값을 가져옵니다.

**결국, 위 쿼리는 다음과 같은 의미가 됩니다:**
"Message(m) 중에서, 그 메시지의 받는 사람([m.to](http://m.to/))의 ID가 **현재 로그인한 사용자의 ID와 같은** 메시지들만 골라서, 페이징 처리해서(Pageable) 가져와줘!"

**중요한 가정:**
이 예시는 `principal` 객체에 `id`라는 속성(또는 `getId()` 메소드)이 있다고 가정하고 있어요. 만약 여러분이 Spring Security의 사용자 정보를 커스터마이징해서 `User` 클래스 대신 `CustomUser` 클래스를 사용하고, 그 안에 `userId`라는 필드가 있다면 `?#{ principal?.userId }` 처럼 사용해야겠죠.

**다른 보안 표현식도 사용 가능!**`SecurityEvaluationContextExtension` 빈을 등록하면, `principal` 외에도 `authentication`, `hasRole('ROLE_USER')` 등 Spring Security의 다양한 [공통 보안 표현식(Common Security Expressions)](https://docs.spring.io/spring-security/reference/ Ausdruck-based-access-control.html#el-common-built-in)들을 `@Query` 안에서 사용할 수 있게 됩니다. (하지만 주로 `principal`을 가장 많이 사용해요.)

---

**오늘의 핵심 요약:**

1. **Spring Data 쿼리에 현재 사용자 정보를 직접 넣어 DB에서 효율적으로 데이터 필터링 가능!**
2. **`spring-security-data` 의존성 추가 & `SecurityEvaluationContextExtension` 빈 등록 필수!**
3. **`@Query` 안에서 `?#{ principal?.속성명 }` 형태로 현재 사용자 정보 접근!** (예: `?#{ principal?.id }`, `?#{ principal?.username }`)
4. **페이징 처리 시 매우 유용!** (DB에서부터 필요한 만큼만 가져오니까)
