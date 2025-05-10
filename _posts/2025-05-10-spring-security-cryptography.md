---
title: Integrations - Cryptography
description: 
author: laze
date: 2025-05-10 00:00:01 +0900
categories: [Dev, SpringSecurity]
tags: [SpringSecurity]
---
## Spring Security Crypto 모듈

Spring Security Crypto 모듈은 대칭 암호화, 키 생성 및 비밀번호 인코딩을 지원합니다.

이 코드는 핵심 모듈의 일부로 배포되지만 다른 Spring Security (또는 Spring) 코드에 대한 의존성은 없습니다.

### 암호화기 (Encryptors)

`Encryptors` 클래스는 대칭 암호화기를 구성하기 위한 팩토리 메소드를 제공합니다.

이 클래스를 사용하면 원시 `byte[]` 형식의 데이터를 암호화하기 위한 `BytesEncryptor` 인스턴스를 만들 수 있습니다.

또한 텍스트 문자열을 암호화하기 위한 `TextEncryptor` 인스턴스를 구성할 수도 있습니다.

`Encryptors`는 스레드 안전합니다.

`BytesEncryptor`와 `TextEncryptor`는 모두 인터페이스입니다.

`BytesEncryptor`에는 여러 구현체가 있습니다.

### BytesEncryptor

`Encryptors.stronger` 팩토리 메소드를 사용하여 `BytesEncryptor`를 구성할 수 있습니다:

**BytesEncryptor**

```java
// Java
Encryptors.stronger("password", "salt");
```

```kotlin
// Kotlin
Encryptors.stronger("password", "salt")
```

`stronger` 암호화 메소드는 Galois Counter Mode(GCM)를 사용하는 256비트 AES 암호화를 사용하여 암호화기를 만듭니다.

이는 PKCS #5의 PBKDF2 (Password-Based Key Derivation Function #2)를 사용하여 비밀 키를 파생합니다.

이 메소드에는 Java 6이 필요합니다.

`SecretKey`를 생성하는 데 사용되는 비밀번호는 안전한 장소에 보관해야 하며 공유해서는 안 됩니다.

솔트(salt)는 암호화된 데이터가 손상된 경우 키에 대한 사전 공격(dictionary attacks)을 방지하는 데 사용됩니다.

각 암호화된 메시지가 고유하도록 16바이트 임의 초기화 벡터도 적용됩니다.

제공된 솔트는 16진수 인코딩된 문자열 형식이어야 하며, 임의적이어야 하고, 최소 8바이트 길이어야 합니다.

`KeyGenerator`를 사용하여 이러한 솔트를 생성할 수 있습니다:

**키 생성하기**

```java
// Java
String salt = KeyGenerators.string().generateKey(); // 임의의 8바이트 솔트를 생성한 다음 16진수로 인코딩합니다.
```

```kotlin
// Kotlin
val salt = KeyGenerators.string().generateKey() // 임의의 8바이트 솔트를 생성한 다음 16진수로 인코딩합니다.
```

Cipher Block Chaining (CBC) 모드의 256비트 AES인 표준 암호화 방법을 사용할 수도 있습니다.

이 모드는 인증되지 않으며 데이터의 신뢰성에 대한 보장을 제공하지 않습니다.

더 안전한 대안으로는 `Encryptors.stronger`를 사용하세요.

### TextEncryptor

`Encryptors.text` 팩토리 메소드를 사용하여 표준 `TextEncryptor`를 구성할 수 있습니다:

**TextEncryptor**

```java
// Java
Encryptors.text("password", "salt");
```

```kotlin
// Kotlin
Encryptors.text("password", "salt")
```

`TextEncryptor`는 표준 `BytesEncryptor`를 사용하여 텍스트 데이터를 암호화합니다.

암호화된 결과는 파일 시스템이나 데이터베이스에 쉽게 저장할 수 있도록 16진수 인코딩된 문자열로 반환됩니다.

### 키 생성기 (Key Generators)

`KeyGenerators` 클래스는 다양한 유형의 키 생성기를 구성하기 위한 여러 편의 팩토리 메소드를 제공합니다.

이 클래스를 사용하면 `byte[]` 키를 생성하기 위한 `BytesKeyGenerator`를 만들 수 있습니다.

또한 문자열 키를 생성하기 위한 `StringKeyGenerator`를 구성할 수도 있습니다.

`KeyGenerators`는 스레드 안전한 클래스입니다.

### BytesKeyGenerator

`KeyGenerators.secureRandom` 팩토리 메소드를 사용하여 `SecureRandom` 인스턴스로 지원되는 `BytesKeyGenerator`를 생성할 수 있습니다:

**BytesKeyGenerator**

```java
// Java
BytesKeyGenerator generator = KeyGenerators.secureRandom();
byte[] key = generator.generateKey();
```

```kotlin
// Kotlin
val generator = KeyGenerators.secureRandom()
val key = generator.generateKey()
```

기본 키 길이는 8바이트입니다. `KeyGenerators.secureRandom` 변형은 키 길이를 제어할 수 있도록 합니다:

**KeyGenerators.secureRandom**

```java
// Java
KeyGenerators.secureRandom(16);
```

```kotlin
// Kotlin
KeyGenerators.secureRandom(16)
```

`KeyGenerators.shared` 팩토리 메소드를 사용하여 호출할 때마다 항상 동일한 키를 반환하는 `BytesKeyGenerator`를 구성합니다:

**KeyGenerators.shared**

```java
// Java
KeyGenerators.shared(16);
```

```kotlin
// Kotlin
KeyGenerators.shared(16)
```

### StringKeyGenerator

`KeyGenerators.string` 팩토리 메소드를 사용하여 각 키를 문자열로 16진수 인코딩하는 8바이트 `SecureRandom` `KeyGenerator`를 구성할 수 있습니다:

**StringKeyGenerator**

```java
// Java
KeyGenerators.string();
```

```kotlin
// Kotlin
KeyGenerators.string()
```

### 비밀번호 인코딩 (Password Encoding)

`spring-security-crypto` 모듈의 `password` 패키지는 비밀번호 인코딩을 지원합니다.

`PasswordEncoder`는 핵심 서비스 인터페이스이며 다음 시그니처를 가집니다:

```java
public interface PasswordEncoder {
	String encode(CharSequence rawPassword);

	boolean matches(CharSequence rawPassword, String encodedPassword);

	default boolean upgradeEncoding(String encodedPassword) {
		return false;
	}
}
```

`matches` 메소드는 `rawPassword`가 인코딩된 후 `encodedPassword`와 같으면 `true`를 반환합니다.

이 메소드는 비밀번호 기반 인증 체계를 지원하도록 설계되었습니다.

`BCryptPasswordEncoder` 구현은 널리 지원되는 "bcrypt" 알고리즘을 사용하여 비밀번호를 해시합니다.

Bcrypt는 임의의 16바이트 솔트 값을 사용하며 비밀번호 크래커를 방해하기 위해 의도적으로 느린 알고리즘입니다.

4에서 31 사이의 값을 사용하는 `strength` 매개변수를 사용하여 수행하는 작업의 양을 조정할 수 있습니다.

값이 높을수록 해시를 계산하는 데 더 많은 작업이 필요합니다.

기본값은 10입니다.

이 값은 인코딩된 해시에도 저장되므로 기존 비밀번호에 영향을 주지 않고 배포된 시스템에서 이 값을 변경할 수 있습니다.

다음 예는 `BCryptPasswordEncoder`를 사용합니다:

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

`Pbkdf2PasswordEncoder` 구현은 PBKDF2 알고리즘을 사용하여 비밀번호를 해시합니다.

비밀번호 크래킹을 막기 위해 PBKDF2는 의도적으로 느린 알고리즘이며 시스템에서 비밀번호를 확인하는 데 약 0.5초가 걸리도록 조정해야 합니다.

다음 시스템은 `Pbkdf2PasswordEncoder`를 사용합니다:

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

---

### 🔐 데이터도 비밀번호처럼 안전하게! (Spring Security Crypto 모듈 파헤치기)

Spring Security는 단순히 로그인 처리만 하는 것이 아니라, 우리 애플리케이션에서 다루는 다양한 데이터를 안전하게 암호화하고, 비밀번호를 효과적으로 인코딩하며, 안전한 암호화 키를 만들 수 있도록 도와주는 **Crypto 모듈**을 제공해요.

이 모듈은 다른 Spring Security 기능이나 Spring 프레임워크 자체에 크게 의존하지 않아서, 필요하다면 독립적으로도 사용할 수 있는 편리한 도구 상자랍니다.

### 1. 대칭키 암호화: 나만 아는 비밀 열쇠로 잠그고 풀기! (Encryptors 🔑)

"대칭키 암호화"는 **하나의 비밀 열쇠(비밀 키)로 데이터를 암호화하고, 똑같은 비밀 열쇠로 다시 복호화(원래 데이터로 되돌리기)하는 방식**이에요. 마치 똑같은 열쇠로 잠그고 여는 자물쇠와 같죠.

Spring Security는 `Encryptors`라는 클래스를 통해 이런 대칭키 암호화 기능을 쉽게 사용할 수 있게 해줘요.

- **`BytesEncryptor`:** 원시 데이터(바이트 배열 `byte[]`)를 암호화하고 복호화할 때 사용해요.
  - **가장 강력하고 권장되는 방법:** `Encryptors.stronger("나만의비밀번호", "랜덤솔트값")`
    - **원리:** 256비트 AES-GCM이라는 매우 안전한 암호화 방식을 사용해요.
    - **비밀 키 생성:** 우리가 제공한 "나만의비밀번호"와 "랜덤솔트값"을 PBKDF2라는 함수로 조합해서 안전한 비밀 키를 만들어요. (이 비밀번호와 솔트는 아주 안전하게 보관해야 해요!)
    - **솔트(Salt)의 역할:** 여기서 솔트는 비밀번호 해싱 때와 비슷하게, 만약 암호화된 데이터가 유출되더라도 원래의 "나만의비밀번호"를 추측하기 어렵게 만들어줘요.
    - **IV (Initialization Vector):** 암호화할 때마다 임의의 값을 추가해서, 똑같은 데이터를 암호화해도 매번 다른 암호문이 나오도록 해요 (더 안전!).
    - **솔트 생성 예시:**

        ```java
        String salt = KeyGenerators.string().generateKey(); // 16진수로 인코딩된 랜덤 8바이트 솔트 생성
        ```

  - **표준 방법 (덜 권장):** 256비트 AES-CBC 방식을 사용해요. `stronger` 방식보다 보안성이 약간 떨어져요. (데이터의 진위성을 보장하지 않음)
- **`TextEncryptor`:** 일반 텍스트(문자열)를 암호화하고 복호화할 때 사용해요.
  - **사용법:** `Encryptors.text("나만의비밀번호", "랜덤솔트값")`
  - **내부 동작:** `BytesEncryptor`를 사용해서 텍스트를 암호화하고, 그 결과를 16진수 문자열(hex-encoded string)로 바꿔서 돌려줘요. (파일이나 DB에 저장하기 편하게)

**언제 사용할까요?**
DB에 저장하는 API 키, 외부 서비스 접속 정보, 개인 설정 중 민감한 내용 등 비밀번호는 아니지만 안전하게 보호해야 하는 데이터를 암호화할 때 유용해요.

### 2. 안전한 암호화 키 만들기! (Key Generators 🗝️)

암호화에는 "키(key)"가 필수죠. Spring Security는 안전하고 예측하기 어려운 키를 만들 수 있도록 `KeyGenerators`라는 도우미 클래스를 제공해요.

- **`BytesKeyGenerator`:** 바이트 배열(`byte[]`) 형태의 키를 만들어요.
  - **가장 일반적인 방법 (랜덤 키):** `KeyGenerators.secureRandom()` 또는 `KeyGenerators.secureRandom(원하는키길이_바이트단위)`
    - `SecureRandom`이라는 암호학적으로 안전한 난수 생성기를 사용해서 키를 만들어요.
    - 기본 길이는 8바이트(64비트)입니다.

      ```java
      BytesKeyGenerator generator = KeyGenerators.secureRandom(16); // 16바이트 (128비트) 키 생성기
      byte[] key = generator.generateKey(); // 실제 키 생성
      ```

  - **항상 같은 키 반환:** `KeyGenerators.shared(원하는키길이_바이트단위)`
    - 여러 곳에서 동일한 공유 키를 사용해야 할 때 쓸 수 있지만, 키가 고정되므로 보안에 더 신경 써야 해요.
- **`StringKeyGenerator`:** 문자열 형태의 키를 만들어요.
  - **사용법:** `KeyGenerators.string()`
  - **내부 동작:** 8바이트 길이의 랜덤 바이트 키를 만들고, 그걸 16진수 문자열로 바꿔서 돌려줘요. (주로 솔트값 생성 등에 편리하게 사용됨)

**왜 필요할까요?**
암호화에 사용할 키나 솔트값을 직접 아무렇게나 만들면 예측 가능하거나 취약할 수 있어요. `KeyGenerators`를 사용하면 이런 위험을 줄이고 안전한 값을 쉽게 얻을 수 있습니다.

### 3. 비밀번호 인코딩: (Password Encoding 🛡️)

이 부분은 앞에서 "Password Storage" 챕터에서 자세히 다뤘던 내용과 거의 같아요! Spring Security Crypto 모듈 안에 `PasswordEncoder` 인터페이스와 그 구현체들(`BCryptPasswordEncoder`, `Pbkdf2PasswordEncoder` 등)이 포함되어 있다는 것을 다시 한번 알려주는 거예요.

- **`PasswordEncoder` 인터페이스:**

    ```java
    public interface PasswordEncoder {
        String encode(CharSequence rawPassword); // 평문 비밀번호를 인코딩(해시)
        boolean matches(CharSequence rawPassword, String encodedPassword); // 평문과 인코딩된 비밀번호 비교
        default boolean upgradeEncoding(String encodedPassword) { // 인코딩 방식 업그레이드 필요 여부 (잘 안 쓰임)
            return false;
        }
    }
    ```

- **주요 구현체:**
  - **`BCryptPasswordEncoder`:** bcrypt 알고리즘 사용, 랜덤 솔트, 의도적으로 느림, `strength` 값으로 작업량 조절 가능 (기본 10). **가장 널리 권장되는 방식 중 하나!**

      ```java
      BCryptPasswordEncoder encoder = new BCryptPasswordEncoder(12); // strength 12로 설정
      String hashedPassword = encoder.encode("내비밀번호");
      boolean isMatch = encoder.matches("내비밀번호", hashedPassword);
      ```

  - **`Pbkdf2PasswordEncoder`:** PBKDF2 알고리즘 사용, 의도적으로 느림, 비밀번호 검증에 약 0.5초 정도 걸리도록 튜닝 권장. (FIPS 인증 필요 시 좋은 선택)

**기억하세요!**`DelegatingPasswordEncoder`는 이러한 다양한 `PasswordEncoder` 구현체들을 상황에 맞게 선택해서 사용할 수 있도록 해주는 "교통정리 담당"이었죠? Crypto 모듈은 그 개별 "선수"들을 제공하는 셈입니다.

---

**오늘의 핵심 요약:**

1. **Spring Security Crypto 모듈은 독립적으로 사용할 수 있는 암호화 도구 상자!**
2. **`Encryptors`:** 대칭키 암호화/복호화 (민감 데이터 보호용). `Encryptors.stronger()`가 안전!
3. **`KeyGenerators`:** 안전한 암호화 키 및 솔트 생성. `KeyGenerators.secureRandom()`으로 랜덤 값 생성!
4. **`PasswordEncoder`:** 안전한 비밀번호 해싱 (bcrypt, PBKDF2 등). 비밀번호 저장의 필수품!
