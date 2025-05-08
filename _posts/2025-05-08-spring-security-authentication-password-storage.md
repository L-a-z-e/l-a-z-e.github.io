---
title: Authentication Password Storage
description: 
author: laze
date: 2025-05-08 00:00:02 +0900
categories: [Dev, SpringSecurity]
tags: [SpringSecurity]
---
# Authentication

## 인증 (Authentication)

Spring Security는 인증에 대한 포괄적인 지원을 제공합니다.

인증은 특정 리소스에 접근하려는 사람의 신원을 확인하는 방법입니다.

사용자를 인증하는 일반적인 방법은 사용자에게 사용자 이름과 비밀번호를 입력하도록 요구하는 것입니다.

인증이 수행되면 우리는 신원을 알 수 있고 권한 부여(authorization)를 수행할 수 있습니다.

Spring Security는 사용자 인증을 위한 내장 지원을 제공합니다.

이 섹션은 서블릿(Servlet) 환경과 웹플럭스(WebFlux) 환경 모두에 적용되는 일반적인 인증 지원에 대해 설명합니다.

각 스택에서 지원되는 내용에 대한 자세한 내용은 서블릿 및 웹플럭스 인증 섹션을 참조하세요.

## Password Storage
## 비밀번호 저장 (Password Storage)

Spring Security의 `PasswordEncoder` 인터페이스는 비밀번호를 안전하게 저장할 수 있도록 단방향 변환을 수행하는 데 사용됩니다.

`PasswordEncoder`는 단방향 변환이므로, 비밀번호 변환이 양방향이어야 하는 경우(예: 데이터베이스 인증에 사용되는 자격 증명 저장)에는 유용하지 않습니다.

일반적으로 `PasswordEncoder`는 인증 시 사용자가 제공한 비밀번호와 비교해야 하는 비밀번호를 저장하는 데 사용됩니다.

### 비밀번호 저장의 역사

수년에 걸쳐 비밀번호를 저장하는 표준 메커니즘은 발전해 왔습니다.

초기에는 비밀번호가 평문(plaintext)으로 저장되었습니다.

비밀번호가 저장된 데이터 저장소에 접근하려면 자격 증명이 필요했기 때문에 비밀번호는 안전하다고 가정했습니다.

그러나 악의적인 사용자는 SQL Injection과 같은 공격을 사용하여 사용자 이름과 비밀번호의 대량 "데이터 덤프"를 얻는 방법을 찾아냈습니다.

점점 더 많은 사용자 자격 증명이 공개되면서 보안 전문가들은 사용자 비밀번호를 보호하기 위해 더 많은 노력이 필요하다는 것을 깨달았습니다.

그 후 개발자들은 SHA-256과 같은 단방향 해시를 실행한 후 비밀번호를 저장하도록 권장받았습니다.

사용자가 인증을 시도하면 해시된 비밀번호가 입력한 비밀번호의 해시와 비교되었습니다.

이는 시스템이 비밀번호의 단방향 해시만 저장하면 된다는 것을 의미했습니다.

침해가 발생하면 비밀번호의 단방향 해시만 노출되었습니다.

해시는 단방향이고 해시를 가지고 비밀번호를 추측하는 것은 계산적으로 어렵기 때문에 시스템의 각 비밀번호를 알아내는 것은 노력할 가치가 없었습니다.

이 새로운 시스템을 무력화하기 위해 악의적인 사용자들은 레인보우 테이블(Rainbow Tables)로 알려진 조회 테이블을 만들기로 결정했습니다.

매번 각 비밀번호를 추측하는 작업을 하는 대신, 비밀번호를 한 번 계산하여 조회 테이블에 저장했습니다.

레인보우 테이블의 효과를 완화하기 위해 개발자들은 솔트된(salted) 비밀번호를 사용하도록 권장받았습니다.

해시 함수의 입력으로 비밀번호만 사용하는 대신, 모든 사용자 비밀번호에 대해 임의의 바이트(솔트라고 함)가 생성되었습니다.

솔트와 사용자 비밀번호는 해시 함수를 통해 실행되어 고유한 해시를 생성했습니다.

솔트는 사용자 비밀번호와 함께 일반 텍스트로 저장되었습니다.

그런 다음 사용자가 인증을 시도하면 해시된 비밀번호가 저장된 솔트와 입력한 비밀번호의 해시와 비교되었습니다.

고유한 솔트는 모든 솔트와 비밀번호 조합에 대해 해시가 다르기 때문에 레인보우 테이블이 더 이상 효과적이지 않다는 것을 의미했습니다.

현대에는 SHA-256과 같은 암호화 해시가 더 이상 안전하지 않다는 것을 알고 있습니다.

그 이유는 현대 하드웨어를 사용하면 초당 수십억 개의 해시 계산을 수행할 수 있기 때문입니다.

이는 각 비밀번호를 개별적으로 쉽게 해독할 수 있음을 의미합니다.

이제 개발자들은 비밀번호를 저장하기 위해 적응형 단방향 함수(adaptive one-way functions)를 활용하도록 권장받고 있습니다.

적응형 단방향 함수를 사용한 비밀번호 검증은 의도적으로 리소스 집약적입니다(의도적으로 많은 CPU, 메모리 또는 기타 리소스를 사용함).

적응형 단방향 함수는 하드웨어가 향상됨에 따라 증가할 수 있는 "작업 계수(work factor)"를 구성할 수 있도록 합니다.

시스템에서 비밀번호를 확인하는 데 약 1초가 걸리도록 "작업 계수"를 조정하는 것이 좋습니다.

이 절충안은 공격자가 비밀번호를 해독하기 어렵게 만들지만, 자체 시스템에 과도한 부담을 주거나 사용자를 짜증나게 할 정도로 비용이 많이 들지 않도록 하기 위한 것입니다.

Spring Security는 "작업 계수"에 대한 좋은 시작점을 제공하려고 시도했지만, 성능이 시스템마다 크게 다르므로 사용자가 자체 시스템에 맞게 "작업 계수"를 사용자 정의하도록 권장합니다.

사용해야 하는 적응형 단방향 함수의 예로는 bcrypt, PBKDF2, scrypt, argon2가 있습니다.

적응형 단방향 함수는 의도적으로 리소스 집약적이므로 모든 요청에 대해 사용자 이름과 비밀번호를 확인하면 애플리케이션의 성능이 크게 저하될 수 있습니다.

보안은 검증을 리소스 집약적으로 만들어 얻어지기 때문에 Spring Security(또는 다른 라이브러리)가 비밀번호 검증 속도를 높이기 위해 할 수 있는 일은 없습니다.

사용자는 장기 자격 증명(즉, 사용자 이름 및 비밀번호)을 단기 자격 증명(예: 세션, OAuth 토큰 등)으로 교환하도록 권장됩니다.

단기 자격 증명은 보안 손실 없이 신속하게 확인할 수 있습니다.

### DelegatingPasswordEncoder

Spring Security 5.0 이전에는 기본 `PasswordEncoder`가 평문 비밀번호를 요구하는 `NoOpPasswordEncoder`였습니다.

비밀번호 저장 내역 섹션을 기반으로 하면 이제 기본 `PasswordEncoder`가 `BCryptPasswordEncoder`와 같은 것이라고 예상할 수 있습니다.

그러나 이는 세 가지 실제 문제를 무시합니다:

1. 많은 애플리케이션이 쉽게 마이그레이션할 수 없는 오래된 암호 인코딩을 사용합니다.
2. 비밀번호 저장에 대한 모범 사례는 다시 변경될 것입니다.
3. 프레임워크로서 Spring Security는 호환성이 깨지는 변경을 자주 할 수 없습니다.

대신 Spring Security는 다음을 통해 모든 문제를 해결하는 `DelegatingPasswordEncoder`를 도입합니다:

- 현재 비밀번호 저장 권장 사항을 사용하여 비밀번호가 인코딩되도록 보장합니다.
- 최신 및 레거시 형식의 비밀번호 검증을 허용합니다.
- 향후 인코딩 업그레이드를 허용합니다.

`PasswordEncoderFactories`를 사용하여 `DelegatingPasswordEncoder`의 인스턴스를 쉽게 구성할 수 있습니다:

**기본 DelegatingPasswordEncoder 생성**

```java
// Java
PasswordEncoder passwordEncoder =
    PasswordEncoderFactories.createDelegatingPasswordEncoder();
```

```kotlin
// Kotlin
val passwordEncoder = PasswordEncoderFactories.createDelegatingPasswordEncoder()
```

또는 자신만의 사용자 정의 인스턴스를 만들 수 있습니다:

**사용자 정의 DelegatingPasswordEncoder 생성**

```java
// Java
String idForEncode = "bcrypt";
Map<String, PasswordEncoder> encoders = new HashMap<>();
encoders.put(idForEncode, new BCryptPasswordEncoder());
encoders.put("noop", NoOpPasswordEncoder.getInstance());
encoders.put("pbkdf2", Pbkdf2PasswordEncoder.defaultsForSpringSecurity_v5_5());
encoders.put("pbkdf2@SpringSecurity_v5_8", Pbkdf2PasswordEncoder.defaultsForSpringSecurity_v5_8());
encoders.put("scrypt", SCryptPasswordEncoder.defaultsForSpringSecurity_v4_1());
encoders.put("scrypt@SpringSecurity_v5_8", SCryptPasswordEncoder.defaultsForSpringSecurity_v5_8());
encoders.put("argon2", Argon2PasswordEncoder.defaultsForSpringSecurity_v5_2());
encoders.put("argon2@SpringSecurity_v5_8", Argon2PasswordEncoder.defaultsForSpringSecurity_v5_8());
encoders.put("sha256", new StandardPasswordEncoder()); // 예전에는 StandardPasswordEncoder, 현재는 new DigestPasswordEncodingUtils("SHA-256", false) 와 유사한 방식으로 내부 구현됨.

PasswordEncoder passwordEncoder =
    new DelegatingPasswordEncoder(idForEncode, encoders);
```

```kotlin
// Kotlin
val idForEncode = "bcrypt"
val encoders = mutableMapOf<String, PasswordEncoder>()
encoders[idForEncode] = BCryptPasswordEncoder()
encoders["noop"] = NoOpPasswordEncoder.getInstance()
encoders["pbkdf2"] = Pbkdf2PasswordEncoder.defaultsForSpringSecurity_v5_5()
encoders["pbkdf2@SpringSecurity_v5_8"] = Pbkdf2PasswordEncoder.defaultsForSpringSecurity_v5_8()
encoders["scrypt"] = SCryptPasswordEncoder.defaultsForSpringSecurity_v4_1()
encoders["scrypt@SpringSecurity_v5_8"] = SCryptPasswordEncoder.defaultsForSpringSecurity_v5_8()
encoders["argon2"] = Argon2PasswordEncoder.defaultsForSpringSecurity_v5_2()
encoders["argon2@SpringSecurity_v5_8"] = Argon2PasswordEncoder.defaultsForSpringSecurity_v5_8()
encoders["sha256"] = StandardPasswordEncoder() // 예전에는 StandardPasswordEncoder, 현재는 new DigestPasswordEncodingUtils("SHA-256", false) 와 유사한 방식으로 내부 구현됨.

val passwordEncoder = DelegatingPasswordEncoder(idForEncode, encoders)
```

### 비밀번호 저장 형식

비밀번호의 일반적인 형식은 다음과 같습니다:

**DelegatingPasswordEncoder 저장 형식**

```
{id}encodedPassword
```

`id`는 어떤 `PasswordEncoder`를 사용해야 하는지 조회하는 데 사용되는 식별자이고 `encodedPassword`는 선택된 `PasswordEncoder`에 대한 원래 인코딩된 비밀번호입니다.

`id`는 비밀번호 시작 부분에 있어야 하며, `{`로 시작하고 `}`로 끝나야 합니다. `id`를 찾을 수 없으면 `id`는 null로 설정됩니다.

예를 들어, 다음은 서로 다른 `id` 값을 사용하여 인코딩된 비밀번호 목록일 수 있습니다.

모든 원래 비밀번호는 "password"입니다.

**DelegatingPasswordEncoder 인코딩된 비밀번호 예시**

```
{bcrypt}$2a$10$dXJ3SW6G7P50lGmMkkmwe.20cQQubK3.HZWzG3YB1tlRy.fqvM/BG
{noop}password
{pbkdf2}5d923b44a6d129f3ddf3e3c8d29412723dcbde72445e8ef6bf3b508fbf17fa4ed4d6b99ca763d8dc
{scrypt}$e0801$8bWJaSu2IKSn9Z9kM+TPXfOc/9bdYSrN1oD9qfVThWEwdRTnO7re7Ei+fUZRJ68k9lTyuTeUp4of4g24hHnazw==$OAOec05+bXxvuu/1qZ6NUR+xQYvYv7BeL1QxwRpY5Pc=
{sha256}97cde38028ad898ebc02e690819fa220e88c62e0699403e94fff291cfffaf8410849f27605abcbc0
```

- 첫 번째 비밀번호는 `PasswordEncoder` id가 `bcrypt`이고 `encodedPassword` 값은 `$2a$10$dXJ3SW6G7P50lGmMkkmwe.20cQQubK3.HZWzG3YB1tlRy.fqvM/BG`입니다. 일치시킬 때 `BCryptPasswordEncoder`에 위임됩니다.
- 두 번째 비밀번호는 `PasswordEncoder` id가 `noop`이고 `encodedPassword` 값은 `password`입니다. 일치시킬 때 `NoOpPasswordEncoder`에 위임됩니다.
- 세 번째 비밀번호는 `PasswordEncoder` id가 `pbkdf2`이고 `encodedPassword` 값은 `5d923b44a6d129f3ddf3e3c8d29412723dcbde72445e8ef6bf3b508fbf17fa4ed4d6b99ca763d8dc`입니다. 일치시킬 때 `Pbkdf2PasswordEncoder`에 위임됩니다.
- 네 번째 비밀번호는 `PasswordEncoder` id가 `scrypt`이고 `encodedPassword` 값은 `$e0801$8bWJaSu2IKSn9Z9kM+TPXfOc/9bdYSrN1oD9qfVThWEwdRTnO7re7Ei+fUZRJ68k9lTyuTeUp4of4g24hHnazw==$OAOec05+bXxvuu/1qZ6NUR+xQYvYv7BeL1QxwRpY5Pc=`입니다. 일치시킬 때 `SCryptPasswordEncoder`에 위임됩니다.
- 마지막 비밀번호는 `PasswordEncoder` id가 `sha256`이고 `encodedPassword` 값은 `97cde38028ad898ebc02e690819fa220e88c62e0699403e94fff291cfffaf8410849f27605abcbc0`입니다. 일치시킬 때 `StandardPasswordEncoder`에 위임됩니다. (역주: 현재는 `StandardPasswordEncoder`가 deprecated 되었으며, `sha256`은 내부적으로 다른 방식으로 처리될 수 있습니다.)

일부 사용자는 저장 형식이 잠재적인 해커에게 제공되는 것에 대해 우려할 수 있습니다.

이는 비밀번호 저장이 알고리즘이 비밀이라는 것에 의존하지 않기 때문에 우려 사항이 아닙니다.

또한 대부분의 형식은 공격자가 접두사 없이도 쉽게 알아낼 수 있습니다.

예를 들어, BCrypt 비밀번호는 종종 `$2a`로 시작합니다.

### 비밀번호 인코딩

생성자에 전달된 `idForEncode`는 비밀번호 인코딩에 사용될 `PasswordEncoder`를 결정합니다.

이전에 구성한 `DelegatingPasswordEncoder`에서 이는 `password`를 인코딩한 결과가 `BCryptPasswordEncoder`에 위임되고 `{bcrypt}` 접두사가 붙는다는 것을 의미합니다.

최종 결과는 다음 예와 같습니다:

**DelegatingPasswordEncoder 인코딩 예시**

```
{bcrypt}$2a$10$dXJ3SW6G7P50lGmMkkmwe.20cQQubK3.HZWzG3YB1tlRy.fqvM/BG
```

### 비밀번호 일치

일치는 `{id}`와 생성자에 제공된 `PasswordEncoder`에 대한 `id` 매핑을 기반으로 합니다.

비밀번호 저장 형식의 예시는 이것이 어떻게 수행되는지에 대한 작동 예시를 제공합니다.

기본적으로 매핑되지 않은 `id`(null `id` 포함)를 가진 비밀번호와 `matches(CharSequence, String)`를 호출한 결과는 `IllegalArgumentException`이 발생합니다.

이 동작은 `DelegatingPasswordEncoder.setDefaultPasswordEncoderForMatches(PasswordEncoder)`를 사용하여 사용자 정의할 수 있습니다.

`id`를 사용하면 모든 비밀번호 인코딩에서 일치시킬 수 있지만 가장 최신의 비밀번호 인코딩을 사용하여 비밀번호를 인코딩할 수 있습니다.

이는 암호화와 달리 비밀번호 해시는 평문을 복구할 간단한 방법이 없도록 설계되었기 때문에 중요합니다.

평문을 복구할 방법이 없으므로 비밀번호를 마이그레이션하기가 어렵습니다.

사용자가 `NoOpPasswordEncoder`를 마이그레이션하는 것은 간단하지만, 시작 환경을 간단하게 만들기 위해 기본적으로 포함하도록 선택했습니다.

### 시작 환경

데모나 샘플을 만들고 있다면 사용자의 비밀번호를 해시하는 데 시간을 할애하는 것이 다소 번거롭습니다.

이를 더 쉽게 만들기 위한 편의 메커니즘이 있지만, 이는 여전히 프로덕션 환경을 위한 것이 아닙니다.

**withDefaultPasswordEncoder 예시**

```java
// Java
UserDetails user = User.withDefaultPasswordEncoder()
  .username("user")
  .password("password")
  .roles("user")
  .build();
System.out.println(user.getPassword());
// {bcrypt}$2a$10$dXJ3SW6G7P50lGmMkkmwe.20cQQubK3.HZWzG3YB1tlRy.fqvM/BG
```

```kotlin
// Kotlin
val user = User.withDefaultPasswordEncoder()
  .username("user")
  .password("password")
  .roles("user")
  .build()
println(user.password)
// {bcrypt}$2a$10$dXJ3SW6G7P50lGmMkkmwe.20cQQubK3.HZWzG3YB1tlRy.fqvM/BG
```

여러 사용자를 만드는 경우 빌더를 재사용할 수도 있습니다:

**withDefaultPasswordEncoder 빌더 재사용**

```java
// Java
User.UserBuilder users = User.withDefaultPasswordEncoder();
UserDetails user = users
  .username("user")
  .password("password")
  .roles("USER")
  .build();
UserDetails admin = users
  .username("admin")
  .password("password")
  .roles("USER","ADMIN")
  .build();
```

```kotlin
// Kotlin
val users = User.withDefaultPasswordEncoder()
val user = users
  .username("user")
  .password("password")
  .roles("USER")
  .build()
val admin = users
  .username("admin")
  .password("password")
  .roles("USER","ADMIN")
  .build()
```

이렇게 하면 저장된 비밀번호가 해시되지만, 비밀번호는 여전히 메모리와 컴파일된 소스 코드에 노출됩니다.

따라서 프로덕션 환경에서는 여전히 안전하다고 간주되지 않습니다.

프로덕션 환경에서는 외부에서 비밀번호를 해시해야 합니다.

### Spring Boot CLI로 인코딩

비밀번호를 올바르게 인코딩하는 가장 쉬운 방법은 Spring Boot CLI를 사용하는 것입니다.

예를 들어, 다음 예는 `DelegatingPasswordEncoder`와 함께 사용할 "password"라는 비밀번호를 인코딩합니다:

**Spring Boot CLI encodepassword 예시**

```bash
spring encodepassword password
{bcrypt}$2a$10$X5wFBtLrL/kHcmrOGGTrGufsBX8CJ0WpQpF3pgeuxBB/H73BK1DW6
```

### 문제 해결

다음 오류는 비밀번호 저장 형식에서 설명한 대로 저장된 비밀번호 중 하나에 `id`가 없을 때 발생합니다.

```
java.lang.IllegalArgumentException: There is no PasswordEncoder mapped for the id "null"
	at org.springframework.security.crypto.password.DelegatingPasswordEncoder$UnmappedIdPasswordEncoder.matches(DelegatingPasswordEncoder.java:233)
	at org.springframework.security.crypto.password.DelegatingPasswordEncoder.matches(DelegatingPasswordEncoder.java:196)
```

이를 해결하는 가장 쉬운 방법은 현재 비밀번호가 어떻게 저장되고 있는지 파악하고 올바른 `PasswordEncoder`를 명시적으로 제공하는 것입니다.

Spring Security 4.2.x에서 마이그레이션하는 경우 `NoOpPasswordEncoder` 빈을 노출하여 이전 동작으로 되돌릴 수 있습니다.

또는 모든 비밀번호에 올바른 `id` 접두사를 붙이고 `DelegatingPasswordEncoder`를 계속 사용할 수 있습니다. 예를 들어, BCrypt를 사용하는 경우 다음과 같이 비밀번호를 마이그레이션합니다:

```
$2a$10$dXJ3SW6G7P50lGmMkkmwe.20cQQubK3.HZWzG3YB1tlRy.fqvM/BG
```

에서

```
{bcrypt}$2a$10$dXJ3SW6G7P50lGmMkkmwe.20cQQubK3.HZWzG3YB1tlRy.fqvM/BG
```

매핑의 전체 목록은 `PasswordEncoderFactories`의 Javadoc을 참조하세요.

### BCryptPasswordEncoder

`BCryptPasswordEncoder` 구현은 널리 지원되는 bcrypt 알고리즘을 사용하여 비밀번호를 해시합니다.

비밀번호 해독에 더 강하도록 bcrypt는 의도적으로 느립니다.

다른 적응형 단방향 함수와 마찬가지로 시스템에서 비밀번호를 확인하는 데 약 1초가 걸리도록 조정해야 합니다.

`BCryptPasswordEncoder`의 기본 구현은 `BCryptPasswordEncoder`의 Javadoc에 언급된 대로 강도 10을 사용합니다.

시스템에서 비밀번호를 확인하는 데 약 1초가 걸리도록 강도 매개변수를 조정하고 테스트하는 것이 좋습니다.

**BCryptPasswordEncoder**

```java
// Java
// 강도 16으로 인코더 생성
BCryptPasswordEncoder encoder = new BCryptPasswordEncoder(16);
String result = encoder.encode("myPassword");
assertTrue(encoder.matches("myPassword", result));
```

```kotlin
// Kotlin
// 강도 16으로 인코더 생성
val encoder = BCryptPasswordEncoder(16)
val result = encoder.encode("myPassword")
assertTrue(encoder.matches("myPassword", result))
```

### Argon2PasswordEncoder

`Argon2PasswordEncoder` 구현은 Argon2 알고리즘을 사용하여 비밀번호를 해시합니다.

Argon2는 암호 해싱 경쟁(Password Hashing Competition)의 우승자입니다.

사용자 지정 하드웨어에서의 비밀번호 해독을 막기 위해 Argon2는 대량의 메모리를 필요로 하는 의도적으로 느린 알고리즘입니다.

다른 적응형 단방향 함수와 마찬가지로 시스템에서 비밀번호를 확인하는 데 약 1초가 걸리도록 조정해야 합니다.

`Argon2PasswordEncoder`의 현재 구현에는 BouncyCastle이 필요합니다.

**Argon2PasswordEncoder**

```java
// Java
// 모든 기본값으로 인코더 생성
Argon2PasswordEncoder encoder = Argon2PasswordEncoder.defaultsForSpringSecurity_v5_8();
String result = encoder.encode("myPassword");
assertTrue(encoder.matches("myPassword", result));
```

```kotlin
// Kotlin
// 모든 기본값으로 인코더 생성
val encoder = Argon2PasswordEncoder.defaultsForSpringSecurity_v5_8()
val result = encoder.encode("myPassword")
assertTrue(encoder.matches("myPassword", result))
```

### Pbkdf2PasswordEncoder

`Pbkdf2PasswordEncoder` 구현은 PBKDF2 알고리즘을 사용하여 비밀번호를 해시합니다.

비밀번호 해독을 막기 위해 PBKDF2는 의도적으로 느린 알고리즘입니다.

다른 적응형 단방향 함수와 마찬가지로 시스템에서 비밀번호를 확인하는 데 약 1초가 걸리도록 조정해야 합니다.

이 알고리즘은 FIPS 인증이 필요한 경우 좋은 선택입니다.

**Pbkdf2PasswordEncoder**

```java
// Java
// 모든 기본값으로 인코더 생성
Pbkdf2PasswordEncoder encoder = Pbkdf2PasswordEncoder.defaultsForSpringSecurity_v5_8();
String result = encoder.encode("myPassword");
assertTrue(encoder.matches("myPassword", result));
```

```kotlin
// Kotlin
// 모든 기본값으로 인코더 생성
val encoder = Pbkdf2PasswordEncoder.defaultsForSpringSecurity_v5_8()
val result = encoder.encode("myPassword")
assertTrue(encoder.matches("myPassword", result))
```

### SCryptPasswordEncoder

`SCryptPasswordEncoder` 구현은 scrypt 알고리즘을 사용하여 비밀번호를 해시합니다.

사용자 지정 하드웨어에서의 비밀번호 해독을 막기 위해 scrypt는 대량의 메모리를 필요로 하는 의도적으로 느린 알고리즘입니다.

다른 적응형 단방향 함수와 마찬가지로 시스템에서 비밀번호를 확인하는 데 약 1초가 걸리도록 조정해야 합니다.

**SCryptPasswordEncoder**

```java
// Java
// 모든 기본값으로 인코더 생성
SCryptPasswordEncoder encoder = SCryptPasswordEncoder.defaultsForSpringSecurity_v5_8();
String result = encoder.encode("myPassword");
assertTrue(encoder.matches("myPassword", result));
```

```kotlin
// Kotlin
// 모든 기본값으로 인코더 생성
val encoder = SCryptPasswordEncoder.defaultsForSpringSecurity_v5_8()
val result = encoder.encode("myPassword")
assertTrue(encoder.matches("myPassword", result))
```

### 기타 PasswordEncoders

하위 호환성을 위해서만 존재하는 다른 많은 `PasswordEncoder` 구현이 있습니다.

이들은 더 이상 안전하다고 간주되지 않음을 나타내기 위해 모두 deprecated(사용 자제)되었습니다.

그러나 기존 레거시 시스템을 마이그레이션하기 어렵기 때문에 제거할 계획은 없습니다.

### 비밀번호 저장소 구성

Spring Security는 기본적으로 `DelegatingPasswordEncoder`를 사용합니다. 그러나 `PasswordEncoder`를 Spring 빈으로 노출하여 이를 사용자 정의할 수 있습니다.

Spring Security 4.2.x에서 마이그레이션하는 경우 `NoOpPasswordEncoder` 빈을 노출하여 이전 동작으로 되돌릴 수 있습니다.

`NoOpPasswordEncoder`로 되돌리는 것은 안전한 것으로 간주되지 않습니다.

대신 안전한 비밀번호 인코딩을 지원하기 위해 `DelegatingPasswordEncoder`를 사용하도록 마이그레이션해야 합니다.

**NoOpPasswordEncoder**

```java
// Java
@Bean
public static NoOpPasswordEncoder passwordEncoder() {
    return NoOpPasswordEncoder.getInstance();
}
```

```xml
<!-- XML -->
<bean id="passwordEncoder" class="org.springframework.security.crypto.password.NoOpPasswordEncoder" factory-method="getInstance"/>
```

```kotlin
// Kotlin
@Bean
fun passwordEncoder(): NoOpPasswordEncoder = NoOpPasswordEncoder.getInstance()
```

XML 구성에서는 `NoOpPasswordEncoder` 빈 이름이 `passwordEncoder`여야 합니다.

### 비밀번호 변경 구성

사용자가 비밀번호를 지정할 수 있도록 허용하는 대부분의 애플리케이션에는 해당 비밀번호를 업데이트하는 기능도 필요합니다.

잘 알려진 비밀번호 변경 URL은 비밀번호 관리자가 특정 애플리케이션의 비밀번호 업데이트 엔드포인트를 검색할 수 있는 메커니즘을 나타냅니다.

이 검색 엔드포인트를 제공하도록 Spring Security를 구성할 수 있습니다.

예를 들어 애플리케이션의 비밀번호 변경 엔드포인트가 `/change-password`인 경우 다음과 같이 Spring Security를 구성할 수 있습니다:

**기본 비밀번호 변경 엔드포인트**

```java
// Java
http
    .passwordManagement(Customizer.withDefaults());
```

```xml
<!-- XML (HttpSecurity 설정 내) -->
<password-management/>
```

```kotlin
// Kotlin
http {
    passwordManagement { }
}
```

그러면 비밀번호 관리자가 `/.well-known/change-password`로 이동하면 Spring Security가 엔드포인트인 `/change-password`로 리디렉션합니다.

또는 엔드포인트가 `/change-password` 이외의 다른 것이라면 다음과 같이 지정할 수도 있습니다:

**비밀번호 변경 엔드포인트 변경**

```java
// Java
http
    .passwordManagement((management) -> management
        .changePasswordPage("/update-password")
    );
```

```xml
<!-- XML (HttpSecurity 설정 내) -->
<password-management change-password-page="/update-password"/>
```

```kotlin
// Kotlin
http {
    passwordManagement {
        changePasswordPage = "/update-password"
    }
}
```

위 구성에서는 비밀번호 관리자가 `/.well-known/change-password`로 이동하면 Spring Security가 `/update-password`로 리디렉션합니다.

### 유출된 비밀번호 확인

민감한 데이터를 다루는 애플리케이션을 만드는 경우와 같이 비밀번호가 유출되었는지 확인해야 하는 몇 가지 시나리오가 있습니다.

사용자의 비밀번호 신뢰성을 확인하기 위해 일부 검사를 수행해야 하는 경우가 많습니다.

이러한 검사 중 하나는 일반적으로 데이터 유출에서 발견되었기 때문에 비밀번호가 유출되었는지 여부일 수 있습니다.

이를 용이하게 하기 위해 Spring Security는 `CompromisedPasswordChecker` 인터페이스의 `HaveIBeenPwnedRestApiPasswordChecker` 구현을 통해 Have I Been Pwned API와의 통합을 제공합니다.

`CompromisedPasswordChecker` API를 직접 사용하거나 Spring Security 인증 메커니즘을 통해 `DaoAuthenticationProvider`를 사용하는 경우 `CompromisedPasswordChecker` 빈을 제공하면 Spring Security 구성에서 자동으로 선택됩니다.

이렇게 하면 폼 로그인을 통해 약한 비밀번호(예: 123456)로 인증을 시도할 때 401을 받거나 `/login?error` 페이지로 리디렉션됩니다(사용자 에이전트에 따라 다름).

그러나 이 경우 401 또는 리디렉션만으로는 그다지 유용하지 않으며, 사용자가 올바른 비밀번호를 제공했지만 여전히 로그인할 수 없었기 때문에 혼란을 야기할 수 있습니다.

이러한 경우 `AuthenticationFailureHandler`를 통해 `CompromisedPasswordException`을 처리하여 원하는 로직(예: 사용자 에이전트를 `/reset-password`로 리디렉션)을 수행할 수 있습니다:

**CompromisedPasswordChecker 사용**

```java
// Java
@Bean
public SecurityFilterChain filterChain(HttpSecurity http) throws Exception {
    http
        .authorizeHttpRequests(authorize -> authorize
            .anyRequest().authenticated()
        )
        .formLogin((login) -> login
            .failureHandler(new CompromisedPasswordAuthenticationFailureHandler())
        );
    return http.build();
}

@Bean
public CompromisedPasswordChecker compromisedPasswordChecker() {
    return new HaveIBeenPwnedRestApiPasswordChecker();
}

static class CompromisedPasswordAuthenticationFailureHandler implements AuthenticationFailureHandler {

    private final SimpleUrlAuthenticationFailureHandler defaultFailureHandler = new SimpleUrlAuthenticationFailureHandler(
            "/login?error");

    private final RedirectStrategy redirectStrategy = new DefaultRedirectStrategy();

    @Override
    public void onAuthenticationFailure(HttpServletRequest request, HttpServletResponse response,
            AuthenticationException exception) throws IOException, ServletException {
        if (exception instanceof CompromisedPasswordException) {
            this.redirectStrategy.sendRedirect(request, response, "/reset-password");
            return;
        }
        this.defaultFailureHandler.onAuthenticationFailure(request, response, exception);
    }
}
```

```kotlin
// Kotlin
@Bean
fun filterChain(http: HttpSecurity): SecurityFilterChain {
    http {
        authorizeHttpRequests {
            authorize(anyRequest, authenticated)
        }
        formLogin {
            failureHandler = CompromisedPasswordAuthenticationFailureHandler()
        }
    }
    return http.build()
}

@Bean
fun compromisedPasswordChecker(): CompromisedPasswordChecker {
    return HaveIBeenPwnedRestApiPasswordChecker()
}

class CompromisedPasswordAuthenticationFailureHandler : AuthenticationFailureHandler {

    private val defaultFailureHandler = SimpleUrlAuthenticationFailureHandler("/login?error")
    private val redirectStrategy: RedirectStrategy = DefaultRedirectStrategy()

    override fun onAuthenticationFailure(
        request: HttpServletRequest,
        response: HttpServletResponse,
        exception: AuthenticationException
    ) {
        if (exception is CompromisedPasswordException) {
            redirectStrategy.sendRedirect(request, response, "/reset-password")
            return
        }
        defaultFailureHandler.onAuthenticationFailure(request, response, exception)
    }
}
```

---

### 🔒 비밀번호, 그냥 저장하면 큰일나요! (Password Storage 완벽 이해)

우리가 웹사이트에 회원가입할 때 입력하는 비밀번호, 그게 서버에 어떻게 저장될까요? "그냥 글자 그대로 저장하면 되지 않나?" 싶겠지만, 절대! 절대! 안 됩니다.

만약 해커가 서버를 공격해서 비밀번호를 그대로 가져간다면... 상상만 해도 끔찍하죠? 😱

그래서 Spring Security는 `PasswordEncoder`라는 아주 중요한 도구를 제공해요.

이 친구는 우리가 입력한 비밀번호를 **"아무도 못 알아보게 꽁꽁 숨겨주는 마법사"** 라고 생각하면 돼요.

### 1. 비밀번호 저장, 왜 이렇게 복잡해졌을까요? (비밀번호 저장의 역사 📜)

옛날 옛날에는...

- **1단계: 평문 저장 (아주 위험! ❌)**
  - `내비밀번호123` -> 그대로 `내비밀번호123` 저장.
  - **문제점:** DB 털리면 모든 비밀번호가 그대로 노출!
- **2단계: 해시 사용 (SHA-256 등) (조금 나아졌지만...)**
  - `내비밀번호123` -> `암호화된문자열` (단방향 변환: 원래 비밀번호로 되돌릴 수 없음)
  - 로그인 시: 사용자가 입력한 비밀번호도 똑같이 해시해서 DB에 저장된 해시값과 비교.
  - **문제점:** "레인보우 테이블"이라는 해시값 미리 계산해놓은 사전을 이용하면 특정 해시값의 원본 비밀번호를 쉽게 찾을 수 있음. (마치 정답지를 보고 푸는 것과 같아요!)
- **3단계: 솔트(Salt) + 해시 (꽤 좋아졌지만!)**
  - **솔트(Salt):** 사용자마다 다른 임의의 문자열을 비밀번호에 추가.
  - `내비밀번호123` + `랜덤솔트값` -> `더복잡한암호화된문자열`
  - DB에는 이 `더복잡한암호화된문자열`과 사용된 `랜덤솔트값`을 함께 저장.
  - **장점:** 사용자마다 솔트가 다르니 레인보우 테이블이 무용지물이 됨! (같은 `내비밀번호123`이라도 솔트가 다르면 해시값도 달라지니까요.)
  - **문제점:** 요즘 컴퓨터가 너무 빨라져서, 하나의 해시값에 대해 엄청나게 많은 비밀번호를 빠르게 대입해보는 "무차별 대입 공격(Brute-force attack)"에 취약해짐. SHA-256 같은 일반 해시는 너무 빨리 계산돼요.
- **4단계: 적응형 단방향 함수 (현재의 추천 방식! 👍)**
  - **핵심:** **일부러 느리게, 일부러 많은 자원(CPU, 메모리)을 사용하도록** 설계된 해시 함수.
  - **종류:** `bcrypt`, `PBKDF2`, `scrypt`, `argon2`
  - **"작업 계수(work factor)" 조절:** 컴퓨터 성능이 좋아지면 이 "작업 계수"를 높여서 해시 계산 시간을 계속 1초 정도로 유지할 수 있게 함. (마치 암호문의 난이도를 계속 올리는 것과 같아요!)
  - **장점:** 해커가 비밀번호 하나를 추측하는 데 시간이 오래 걸리게 만들어 공격을 매우 어렵게 만듦.
  - **주의점:** 매번 로그인 요청마다 이 느린 함수를 실행하면 서버 성능에 부담이 될 수 있음. 그래서 보통 로그인 성공 후에는 세션이나 토큰 같은 단기 자격 증명을 발급해서 사용합니다.

### 2. Spring Security 5.0 이후, 비밀번호는 `DelegatingPasswordEncoder`에게 맡겨주세요! 🦸

예전에는 `NoOpPasswordEncoder`(비밀번호 암호화 안 함, 평문 저장 😱)가 기본이었던 시절도 있었지만, 이젠 아니에요!

Spring Security는 **`DelegatingPasswordEncoder`** 라는 아주 똑똑한 친구를 기본으로 사용합니다. 이 친구가 왜 좋냐면요:

- **최신 암호화 방식 사용:** 현재 가장 안전하다고 권장되는 방식(기본적으로 `bcrypt`)으로 새 비밀번호를 암호화해요.
- **과거 방식도 이해해요:** 만약 예전에 다른 방식(예: `SHA-256`)으로 저장된 비밀번호가 있더라도, `DelegatingPasswordEncoder`는 "아, 이건 예전 방식이네?" 하고 알아서 해당 방식으로 비교해줘요. (마치 여러 나라 말을 할 줄 아는 통역사 같아요!)
- **미래에도 대비해요:** 나중에 더 좋은 암호화 방식이 나오면, 쉽게 그 방식으로 업그레이드할 수 있도록 유연하게 설계되었어요.

**어떻게 생겼을까요? 저장된 비밀번호 형식:**

`DelegatingPasswordEncoder`는 비밀번호를 저장할 때 앞에 어떤 암호화 방식을 썼는지 표시를 해둬요.

```
{id}암호화된비밀번호
```

- `{id}`: 어떤 암호화 방식을 썼는지 알려주는 이름표 (예: `{bcrypt}`, `{noop}`, `{pbkdf2}`)
- `암호화된비밀번호`: 실제 암호화된 비밀번호 문자열

**예시:**

- `{bcrypt}$2a$10$dXJ3SW6G7P50lGmMkkmwe.20cQQubK3.HZWzG3YB1tlRy.fqvM/BG` (bcrypt로 암호화됨)
- `{noop}password` (암호화 안 됨 - 테스트용으로만 사용해야 해요!)

**`DelegatingPasswordEncoder` 만들기:**

- **가장 쉬운 방법 (추천):**

    ```java
    PasswordEncoder passwordEncoder = PasswordEncoderFactories.createDelegatingPasswordEncoder();
    ```

  이렇게 하면 Spring Security가 알아서 기본 설정(`bcrypt` 사용)으로 만들어줘요.

- **내가 직접 설정하고 싶을 때:**

    ```java
    String idForEncode = "bcrypt"; // 새 비밀번호는 bcrypt로 암호화할 거야!
    Map<String, PasswordEncoder> encoders = new HashMap<>();
    encoders.put(idForEncode, new BCryptPasswordEncoder());
    encoders.put("pbkdf2", Pbkdf2PasswordEncoder.defaultsForSpringSecurity_v5_8()); // pbkdf2로 저장된 것도 비교할 수 있게
    // ... 다른 인코더들도 추가 ...
    
    PasswordEncoder passwordEncoder = new DelegatingPasswordEncoder(idForEncode, encoders);
    ```


**비밀번호 인코딩 (새 비밀번호 저장 시):**`DelegatingPasswordEncoder`는 `idForEncode`로 지정된 방식(기본은 `bcrypt`)으로 비밀번호를 암호화하고, 앞에 `{id}`를 붙여서 저장해요.

**비밀번호 매칭 (로그인 시):**
사용자가 입력한 비밀번호와 DB에 저장된 `{id}암호화된비밀번호`를 비교할 때,

1. `{id}`를 보고 어떤 `PasswordEncoder`를 써야 할지 결정해요.
2. 해당 `PasswordEncoder`에게 "이 비밀번호 맞는지 확인해줘!" 하고 요청해요.

> 핵심: DelegatingPasswordEncoder 덕분에 과거, 현재, 미래의 다양한 암호화 방식을 유연하게 지원하면서도, 새로 저장되는 비밀번호는 항상 최신 보안 기준을 따를 수 있어요!
>

### 3. 다양한 비밀번호 암호화 방식들 🛡️

Spring Security는 여러 가지 `PasswordEncoder` 구현체를 제공해요. 그중 대표적인 것들은 "적응형 단방향 함수"들이죠.

- **`BCryptPasswordEncoder`:**
  - 가장 널리 쓰이고, Spring Security의 기본 권장 방식.
  - "strength" (강도) 값을 조절해서 계산 속도를 늦출 수 있어요 (기본값 10). 보통 1초 정도 걸리도록 튜닝합니다.
  - **예시:** `new BCryptPasswordEncoder(16)` (강도를 16으로 설정)
- **`Argon2PasswordEncoder`:**
  - 비밀번호 해싱 대회(Password Hashing Competition) 우승자! 🏆
  - CPU뿐만 아니라 메모리도 많이 사용해서 최신 하드웨어로도 뚫기 어렵게 만들었어요.
  - 사용하려면 BouncyCastle 라이브러리가 추가로 필요할 수 있어요.
- **`Pbkdf2PasswordEncoder`:**
  - PBKDF2 알고리즘 사용. 이것도 일부러 느리게 만들었어요.
  - FIPS (미국 연방 정보 처리 표준) 인증이 필요한 경우 좋은 선택.
- **`SCryptPasswordEncoder`:**
  - scrypt 알고리즘 사용. 이것도 메모리를 많이 사용해서 공격을 어렵게 해요.
- **기타 (`NoOpPasswordEncoder`, 예전 해시 방식 등):**
  - `NoOpPasswordEncoder`: **절대! 프로덕션에서는 사용 금지!** 그냥 비밀번호를 그대로 저장해요. 테스트나 아주 간단한 예제에서만 잠깐 쓸 수 있어요.
  - 오래된 해시 방식(SHA-1, MD5 등)을 위한 `PasswordEncoder`도 있지만, 이제는 안전하지 않아서 "deprecated" (사용 자제) 딱지가 붙어있어요. 기존 시스템 호환성을 위해 남아있는 것뿐이에요.

### 4. 실제 프로젝트에서 `PasswordEncoder` 설정하기 🛠️

Spring Security는 기본적으로 `DelegatingPasswordEncoder`를 사용하지만, 우리가 직접 `PasswordEncoder` 빈(Bean)을 등록하면 그걸 사용해요.

```java
import org.springframework.context.annotation.Bean;
import org.springframework.context.annotation.Configuration;
import org.springframework.security.crypto.bcrypt.BCryptPasswordEncoder;
import org.springframework.security.crypto.password.PasswordEncoder;

@Configuration
public class SecurityConfig {

    @Bean
    public PasswordEncoder passwordEncoder() {
        // DelegatingPasswordEncoder를 직접 만들거나, 특정 인코더를 반환할 수 있음
        // return PasswordEncoderFactories.createDelegatingPasswordEncoder(); // 이게 기본!
        return new BCryptPasswordEncoder(); // BCrypt만 쓰고 싶다면 이렇게 해도 되지만, Delegating이 더 유연해요.
    }
}
```

**가장 좋은 방법은 `PasswordEncoderFactories.createDelegatingPasswordEncoder()`를 빈으로 등록하는 것입니다.**

그러면 Spring Security가 알아서 최신 권장 사항으로 설정해줍니다.

**만약 옛날 시스템에서 마이그레이션 중이고, 아직 `{id}` 접두사가 없다면?**`java.lang.IllegalArgumentException: There is no PasswordEncoder mapped for the id "null"` 이런 에러를 만날 수 있어요.
이럴 때는 기존 비밀번호들이 어떤 방식으로 암호화되었는지 확인하고, DB의 모든 비밀번호 앞에 `{bcrypt}`나 `{sha256}` 같은 적절한 `{id}`를 붙여주는 작업을 해야 합니다.

### 5. 잠깐! 테스트할 때 비밀번호 일일이 해시하기 귀찮다면? 🤫 (개발 편의 기능)

데모나 간단한 샘플 만들 때, 사용자 정보를 코드에 직접 넣기도 하죠. 이때 `User.withDefaultPasswordEncoder()`를 사용하면 편리해요.

```java
UserDetails user = User.withDefaultPasswordEncoder()
  .username("user")
  .password("password") // "password"라고 적으면 알아서 bcrypt로 해시해서 저장해줘요!
  .roles("USER")
  .build();

// 저장된 비밀번호 출력해보면: {bcrypt}$2a$10$...(해시값) 이렇게 나와요.
System.out.println(user.getPassword());

```

**주의!** 이건 정말 **테스트나 데모용**이에요.

실제 서비스(프로덕션)에서는 이렇게 쓰면 안 돼요! 비밀번호가 코드에 그대로 노출되니까요.

**프로덕션에서는? Spring Boot CLI 사용!**
터미널에서 Spring Boot CLI를 이용해 안전하게 비밀번호를 해시할 수 있어요.

```bash
spring encodepassword 내비밀번호
# 결과: {bcrypt}$2a$10$...... (이 값을 DB에 저장하거나 설정 파일에 사용)
```

### 6. "이 비밀번호, 혹시 유출된 거 아니야?" (유출된 비밀번호 확인 기능 🕵️)

Spring Security는 "Have I Been Pwned?" (HIBP) 라는 유명한 서비스와 연동해서, 사용자가 입력한 비밀번호가 이미 유출된 적 있는지 확인해주는 기능도 제공해요 (`HaveIBeenPwnedRestApiPasswordChecker`).

만약 유출된 비밀번호로 로그인을 시도하면, 그냥 "로그인 실패!" 라고만 하는 것보다 "이 비밀번호는 유출된 적 있어서 위험해요! 다른 비밀번호로 바꿔주세요." 라고 안내해주는 게 더 좋겠죠?

이런 처리를 `AuthenticationFailureHandler`를 통해 구현할 수 있습니다.

---

휴! 비밀번호 저장에 대한 내용이 정말 많았죠? 하지만 그만큼 중요하기 때문이에요.

**오늘의 핵심 요약:**

1. 비밀번호는 **절대 평문으로 저장하면 안 된다!**
2. 현대에는 **적응형 단방향 함수 (`bcrypt`, `argon2` 등)**를 사용해야 안전하다. (일부러 느리게!)
3. Spring Security 5.0부터는 **`DelegatingPasswordEncoder`** 가 기본! 여러 암호화 방식을 유연하게 지원하고, 새 비밀번호는 최신 방식으로 암호화한다. 저장 형식은 `{id}암호화된비밀번호`.
4. `PasswordEncoderFactories.createDelegatingPasswordEncoder()`를 빈으로 등록하는 것이 가장 권장되는 방법.
5. 테스트 시에는 `User.withDefaultPasswordEncoder()`가 편리하지만, 프로덕션에서는 절대 사용 금지!

### 1. 단방향 함수 (One-way Function) ➡️

**단방향 함수**는 말 그대로 **한쪽 방향으로만 작동하는 함수**를 의미해요.

- **입력 값 (예: 비밀번호) ➡️ [단방향 함수] ➡️ 출력 값 (예: 해시값)**

**특징:**

- **역추적 불가능 (Irreversible):** 가장 중요한 특징이에요! 출력 값(해시값)을 가지고 원래의 입력 값(비밀번호)을 알아내는 것이 **거의 불가능**해야 합니다. 마치 계란을 깨서 스크램블 에그를 만들면, 그 스크램블 에그를 보고 원래 계란 모양으로 되돌릴 수 없는 것과 같아요.
- **동일 입력, 동일 출력 (Deterministic):** 같은 입력 값을 넣으면 항상 같은 출력 값이 나와야 해요. `password123`을 해시하면 항상 `abcdef123456` (예시)이 나와야, 나중에 사용자가 로그인할 때 입력한 `password123`을 다시 해시해서 저장된 값과 비교할 수 있겠죠.
- **눈사태 효과 (Avalanche Effect):** 입력 값이 아주 조금만 달라도 출력 값은 완전히 다르게 나와야 해요. 예를 들어 `password123`과 `Password123` (P 대문자)의 해시값은 전혀 연관성이 없어 보여야 합니다. 이는 해시값을 보고 원래 비밀번호를 유추하기 어렵게 만듭니다.

**예시:**
일반적인 해시 함수들 (SHA-256, SHA-512, MD5 등)이 단방향 함수의 예시입니다. 이들은 비밀번호를 저장할 때, 원래 비밀번호 대신 이 해시값을 저장하는 데 사용되었어요.

**왜 단방향이 필요할까요?**
만약 비밀번호가 그대로 저장되어 있다면, DB가 유출되었을 때 모든 사용자의 비밀번호가 그대로 노출됩니다. 하지만 단방향 함수로 변환된 해시값만 저장하면, 설령 DB가 유출되더라도 해커가 원래 비밀번호를 직접적으로 알 수 없게 됩니다. (물론, 앞에서 배운 레인보우 테이블이나 무차별 대입 공격 같은 방법으로 뚫릴 위험은 여전히 존재합니다.)

### 2. 적응형 단방향 함수 (Adaptive One-way Function) 🐢💪

**적응형 단방향 함수**는 단방향 함수의 기본 특징을 가지면서, **보안성을 더욱 강화하기 위한 추가적인 속성**을 가진 함수예요. 여기서 "적응형"이라는 말은 **상황에 맞춰 조절할 수 있다**는 의미를 담고 있습니다.

**특징 (단방향 함수의 특징 + 추가 특징):**

- **의도적인 느린 속도 (Computationally Intensive):** 가장 큰 특징! 이 함수들은 **일부러 계산을 복잡하고 느리게** 만들어요. 즉, 해시값을 하나 계산하는 데 CPU 시간이나 메모리 같은 자원을 많이 소모하도록 설계되었습니다.
  - **왜 느리게?** 해커가 무차별 대입 공격(하나씩 다 넣어보는 공격)을 시도할 때, 비밀번호 하나를 검증하는 데 시간이 오래 걸리면 전체 공격 시간이 기하급수적으로 늘어나 공격을 매우 비효율적으로 만듭니다. 예를 들어, 일반 해시가 0.001초 걸린다면, 적응형 해시는 1초가 걸리도록 설정할 수 있습니다.
- **작업 계수 (Work Factor) 또는 비용 매개변수 (Cost Parameter) 조절 가능:** 이 "느린 정도"를 개발자가 조절할 수 있는 파라미터(예: bcrypt의 `strength`, scrypt/argon2의 반복 횟수, 메모리 사용량 등)를 제공합니다.
  - **왜 조절 가능?** 컴퓨터 하드웨어 성능은 계속 발전합니다. 5년 전에는 1초 걸리던 계산이 지금은 0.1초 만에 끝날 수도 있어요. "작업 계수"를 높여서 해시 계산 시간을 다시 1초 정도로 "적응"시킬 수 있습니다. 이렇게 함으로써 미래의 더 강력한 하드웨어 공격에도 대비할 수 있습니다.
- **메모리 사용량 조절 (Memory-Hard) (일부 함수):** `scrypt`나 `argon2` 같은 함수는 CPU뿐만 아니라 **메모리 사용량도 많이 요구**하도록 설계되었습니다. 이는 GPU나 ASIC 같은 특수 하드웨어를 사용한 병렬 공격을 더 어렵게 만듭니다. 일반 PC는 메모리가 충분하지만, 병렬 처리를 위한 특수 하드웨어는 코어당 메모리가 제한적인 경우가 많기 때문입니다.

**예시:**`bcrypt`, `PBKDF2`, `scrypt`, `Argon2`가 대표적인 적응형 단방향 함수입니다. Spring Security에서 권장하는 방식들이죠.

**왜 적응형이 필요할까요?**
단순한 단방향 해시 함수(SHA-256 등)는 오늘날의 빠른 컴퓨팅 파워 앞에서는 너무 빨리 계산됩니다. 해커는 초당 수십억 개의 해시를 계산하여 비밀번호를 추측할 수 있습니다. 적응형 단방향 함수는 이러한 현대적인 공격에 대응하기 위해, 의도적으로 계산 비용을 높이고 이 비용을 미래의 기술 발전에 맞춰 "적응"시킬 수 있도록 설계된 것입니다.

**요약:**

| 특징 | 단방향 함수 (예: SHA-256) | 적응형 단방향 함수 (예: bcrypt, Argon2) |
| --- | --- | --- |
| **역추적 불가능** | O | O |
| **동일 입력, 동일 출력** | O | O |
| **계산 속도** | 매우 빠름 | 의도적으로 느림 |
| **작업 계수 조절** | X | O (미래 하드웨어에 적응 가능) |
| **리소스 사용** | 주로 CPU (낮음) | CPU, 메모리 (높음, 조절 가능) |
| **주요 목적** | 데이터 무결성, 간단한 해싱 | **비밀번호 저장 (보안 강화)** |
