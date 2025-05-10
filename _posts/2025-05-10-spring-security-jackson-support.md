---
title: Integrations - Jackson Support
description: 
author: laze
date: 2025-05-10 00:00:04 +0900
categories: [Dev, SpringSecurity]
tags: [SpringSecurity]
---
## Jackson 지원 (Jackson Support)

Spring Security는 Spring Security 관련 클래스를 영속화하기 위한 Jackson 지원을 제공합니다.

이는 분산 세션(예: 세션 복제, Spring Session 등) 작업 시 Spring Security 관련 클래스의 직렬화 성능을 향상시킬 수 있습니다.

이를 사용하려면 `SecurityJackson2Modules.getModules(ClassLoader)`를 `ObjectMapper` (jackson-databind)에 등록하십시오:

**Java**

```java
ObjectMapper mapper = new ObjectMapper();
ClassLoader loader = getClass().getClassLoader();
List<Module> modules = SecurityJackson2Modules.getModules(loader);
mapper.registerModules(modules);

// ... ObjectMapper를 평소처럼 사용 ...
SecurityContext context = new SecurityContextImpl();
// ...
String json = mapper.writeValueAsString(context);
```

**Kotlin**

```kotlin
val mapper = ObjectMapper()
val loader = javaClass.classLoader
val modules = SecurityJackson2Modules.getModules(loader)
mapper.registerModules(modules)

// ... ObjectMapper를 평소처럼 사용 ...
val context: SecurityContext = SecurityContextImpl()
// ...
val json = mapper.writeValueAsString(context)
```

다음 Spring Security 모듈들이 Jackson 지원을 제공합니다:

- `spring-security-core` (CoreJackson2Module)
- `spring-security-web` (WebJackson2Module, WebServletJackson2Module, WebServerJackson2Module)
- `spring-security-oauth2-client` (OAuth2ClientJackson2Module)
- `spring-security-cas` (CasJackson2Module)

---

### 📦 Spring Security 객체도 JSON으로 변신! (Jackson 지원 이해하기)

우리가 웹사이트를 사용할 때, 로그인 정보 같은 것들은 보통 "세션(Session)"이라는 곳에 저장돼요.

그런데 웹사이트 방문자가 아주 많아져서 서버를 여러 대로 늘리면(분산 환경), 이 세션 정보도 여러 서버가 함께 공유해야 하는 문제가 생깁니다.

이때 세션 정보를 한 서버에서 다른 서버로 옮기거나, Redis 같은 외부 저장소에 저장했다가 다시 가져올 때가 있는데, 이런 경우 세션 정보를 **JSON**이라는 텍스트 기반 데이터 형식으로 바꾸는 경우가 많아요.

**Jackson은 뭐길래?**
Jackson은 Java 객체를 JSON으로 바꾸거나, JSON을 다시 Java 객체로 바꾸는 작업을 아주 잘 해주는 유명한 라이브러리입니다. (마치 번역기처럼 객체 <-> JSON 변환을 담당해요.)

**문제 발생! 🚨**
Spring Security가 사용하는 `SecurityContext`, `Authentication`, `UserDetails` 같은 객체들은 내부 구조가 좀 복잡할 수 있어요.

그래서 일반적인 Jackson 설정만으로는 이 객체들을 JSON으로 제대로 바꾸거나, JSON에서 다시 원래 객체로 완벽하게 복원하기 어려울 수 있습니다. (마치 복잡한 기계를 분해했다가 다시 조립하는데, 설명서가 없으면 제대로 못 하는 것과 같아요.)

**Spring Security의 해결책! 전용 Jackson 모듈! 🧩**
Spring Security는 이런 문제를 해결하기 위해 **자신이 사용하는 객체들을 Jackson이 잘 이해하고 처리할 수 있도록 특별한 "설명서(모듈)"들을 제공**합니다.

이 모듈들을 Jackson의 `ObjectMapper`에 등록해주면, Spring Security 관련 객체들도 문제없이 JSON으로 직렬화/역직렬화할 수 있게 됩니다.

### 1. 왜 필요할까요? (분산 세션 환경의 중요성)

- **분산 세션 (Distributed Sessions):**
  - **세션 복제 (Session Replication):** 여러 웹 서버가 서로 세션 정보를 복제하여 공유하는 방식.
  - **Spring Session:** 세션 정보를 Redis, Hazelcast, JDBC 등 외부 저장소에 중앙 집중식으로 저장하고 여러 서버가 공유하는 방식. (요즘 많이 사용!)
- 이런 환경에서는 `SecurityContext` 같은 보안 정보가 포함된 세션 객체가 네트워크를 통해 다른 서버로 전달되거나 외부 저장소에 저장될 때, 효율적이고 안정적인 형태로 변환(직렬화)되어야 합니다. JSON은 이런 용도로 널리 사용되는 형식입니다.
- Spring Security 전용 Jackson 모듈을 사용하면, 이 직렬화/역직렬화 과정에서 **성능을 향상**시키고, Spring Security 객체들이 **정확하게 복원**되도록 보장할 수 있습니다.

### 2. 설정 방법: ObjectMapper에게 "Spring Security 설명서" 전달하기! 📜

Jackson 라이브러리의 핵심 클래스는 `ObjectMapper`예요. 이 친구에게 "Spring Security 객체들은 이렇게 다뤄줘!" 하고 알려줘야 합니다.

```java
import com.fasterxml.jackson.databind.ObjectMapper;
import com.fasterxml.jackson.databind.Module;
import org.springframework.security.jackson2.SecurityJackson2Modules;
import org.springframework.security.core.context.SecurityContext;
import org.springframework.security.core.context.SecurityContextImpl;
import java.util.List;

// ...

// 1. ObjectMapper 인스턴스 생성
ObjectMapper mapper = new ObjectMapper();

// 2. 현재 클래스 로더 가져오기 (보통 이렇게 하면 됨)
ClassLoader loader = getClass().getClassLoader();

// 3. Spring Security가 제공하는 Jackson 모듈 목록 가져오기
List<Module> modules = SecurityJackson2Modules.getModules(loader);

// 4. ObjectMapper에 가져온 모듈들 등록! (이게 핵심!)
mapper.registerModules(modules);

// 이제부터 이 ObjectMapper(mapper)는 Spring Security 객체들을 잘 처리할 수 있어요!
// 예시: SecurityContext 객체를 JSON 문자열로 변환
SecurityContext context = new SecurityContextImpl();
// ... (context에 Authentication 객체 등을 설정하는 코드) ...
String json = mapper.writeValueAsString(context);
System.out.println("SecurityContext JSON: " + json);

// 예시: JSON 문자열을 다시 SecurityContext 객체로 변환
// SecurityContext restoredContext = mapper.readValue(json, SecurityContextImpl.class);
```

**코드 설명:**

1. `ObjectMapper`를 새로 만들거나, 이미 Spring 설정(@Bean)으로 관리되고 있는 `ObjectMapper`가 있다면 그걸 가져와서 사용합니다.
2. `SecurityJackson2Modules.getModules(loader)`: Spring Security가 제공하는 모든 Jackson 관련 모듈들(설명서들)을 한 번에 가져옵니다. `ClassLoader`는 이 모듈들을 찾는 데 필요해요.
3. `mapper.registerModules(modules)`: `ObjectMapper`에게 "이 설명서들 참고해서 작업해!" 하고 알려주는 과정입니다.

이 설정을 해두면, 예를 들어 Spring Session이 세션 정보를 Redis에 JSON 형태로 저장할 때, `SecurityContext` 같은 Spring Security 객체들도 올바르게 JSON으로 변환되고, 나중에 다시 읽어올 때도 문제없이 객체로 복원됩니다.

### 3. 어떤 Spring Security 모듈들이 Jackson 지원을 제공할까요? 📦

Spring Security는 여러 하위 모듈로 구성되어 있는데, 각 모듈에서 사용하는 특정 객체들을 위한 Jackson 지원 모듈들이 있어요. `SecurityJackson2Modules.getModules()`를 사용하면 이런 것들이 한 번에 다 등록됩니다.

- **`spring-security-core` (CoreJackson2Module):** `UserDetails`, `GrantedAuthority`, `Authentication`, `SecurityContext` 등 핵심 객체들을 위한 지원.
- **`spring-security-web` (WebJackson2Module, WebServletJackson2Module, WebServerJackson2Module):** 웹 환경과 관련된 객체들 (예: `SavedRequest`)을 위한 지원.
- **`spring-security-oauth2-client` (OAuth2ClientJackson2Module):** OAuth 2.0 클라이언트 관련 객체들을 위한 지원.
- **`spring-security-cas` (CasJackson2Module):** CAS 프로토콜 관련 객체들을 위한 지원.

보통은 `SecurityJackson2Modules.getModules()` 하나만 호출하면 필요한 것들이 대부분 처리됩니다.

---

**오늘의 핵심 요약:**

1. **분산 세션 환경(예: Spring Session 사용 시)에서는 세션 정보(Spring Security 객체 포함)를 JSON으로 직렬화/역직렬화할 필요가 생김!**
2. **Jackson은 Java 객체 <-> JSON 변환 라이브러리!**
3. **Spring Security 객체는 구조가 복잡해서, 일반 Jackson 설정만으로는 제대로 처리하기 어려울 수 있음!**
4. **해결책: `SecurityJackson2Modules.getModules(loader)`를 `ObjectMapper`에 등록해주면 끝!** (Spring Security 전용 설명서 제공)
5. **이를 통해 Spring Security 객체의 직렬화 성능 향상 및 정확한 복원 보장!**
