---
title: Spring Framework @PostConstruct & @PreDestory
description: 
author: laze
date: 2025-05-04 00:00:02 +0900
categories: [Dev, SpringBoot]
tags: [SpringBoot]
---
**Using @PostConstruct and @PreDestroy**

`CommonAnnotationBeanPostProcessor`는 `@Resource` 어노테이션뿐만 아니라 JSR-250 생명주기 어노테이션인 `jakarta.annotation.PostConstruct`와 `jakarta.annotation.PreDestroy`도 인식합니다.

스프링 2.5에서 도입된 이 어노테이션들에 대한 지원은 초기화 콜백(initialization callbacks) 및 소멸 콜백(destruction callbacks)에서 설명된 생명주기 콜백 메커니즘의 대안을 제공합니다.

`CommonAnnotationBeanPostProcessor`가 스프링 `ApplicationContext` 내에 등록되어 있다면,

이러한 어노테이션 중 하나를 가진 메소드는 해당 스프링 생명주기 인터페이스 메소드 또는 명시적으로 선언된 콜백 메소드와 동일한 생명주기 시점에서 호출됩니다.

다음 예제에서는 캐시가 초기화 시 미리 채워지고(pre-populated) 소멸 시 비워집니다(cleared):

```java
// Java
public class CachingMovieLister {

	@PostConstruct
	public void populateMovieCache() {
		// 초기화 시 영화 캐시 채우기...
	}

	@PreDestroy
	public void clearMovieCache() {
		// 소멸 시 영화 캐시 비우기...
	}
}
```

```kotlin
// Kotlin
class CachingMovieLister {

    @PostConstruct
    fun populateMovieCache() {
        // 초기화 시 영화 캐시 채우기...
    }

    @PreDestroy
    fun clearMovieCache() {
        // 소멸 시 영화 캐시 비우기...
    }
}
```

`@Resource`와 마찬가지로, `@PostConstruct` 및 `@PreDestroy` 어노테이션 타입은 JDK 6부터 8까지 표준 자바 라이브러리의 일부였습니다.

그러나 전체 `javax.annotation` 패키지는 JDK 9에서 핵심 자바 모듈과 분리되었고 결국 JDK 11에서 제거되었습니다.

Jakarta EE 9부터 해당 패키지는 현재 `jakarta.annotation`에 존재합니다.

필요한 경우, `jakarta.annotation-api` 아티팩트는 이제 Maven Central을 통해 얻어야 하며, 다른 라이브러리처럼 단순히 애플리케이션의 클래스패스에 추가하면 됩니다.

---

**전체 주제: `@PostConstruct`와 `@PreDestroy` 사용하기**

이 부분은 스프링 빈의 **초기화**와 **소멸** 시점에 특정 로직을 실행하기 위해, 자바 표준(JSR-250)에서 정의된 `@PostConstruct`와 `@PreDestroy` 어노테이션을 사용하는 방법에 대한 내용입니다.

**핵심 아이디어:** 스프링 전용 인터페이스나 설정 대신, 표준 어노테이션을 사용하여 빈의 생명주기 콜백 메소드를 지정하자!

---

**1. 동작 원리 (`CommonAnnotationBeanPostProcessor`):**

- `@Autowired`나 `@Resource` 같은 어노테이션을 처리하는 `BeanPostProcessor`가 있었습니다.
  마찬가지로, `@PostConstruct`와 `@PreDestroy` 어노테이션을 **찾아서 처리**하는 역할을 하는 것이 바로 **`CommonAnnotationBeanPostProcessor`** 입니다.
- 이 `BeanPostProcessor`는 스프링 컨테이너에 등록되어 있으면 (보통 `<context:annotation-config/>`나 `<context:component-scan>`을 사용하면 자동으로 등록됨), 빈 생성 및 소멸 과정에서 해당 어노테이션이 붙은 메소드를 발견하고 적절한 시점에 호출해줍니다.

---

**2. `@PostConstruct` 사용법 및 역할:**

- **역할:** 빈의 **초기화 콜백**을 지정합니다.
- **호출 시점:** 스프링 컨테이너가 빈 객체를 생성하고 모든 의존성 주입(`@Autowired`, `@Resource` 등)을 완료한 직후, 그리고 빈이 실제로 사용되기 **전에** 호출됩니다.
  - 이는 `InitializingBean`의 `afterPropertiesSet()` 메소드나 `init-method`로 지정된 메소드와 **동일한 시점**에 해당합니다.
- **사용법:** 초기화 로직을 담고 있는 메소드 위에 `@PostConstruct` 어노테이션을 붙입니다. 메소드는 **인자(argument)가 없어야** 하며, 일반적으로 `void`를 반환하지만 반환 타입은 크게 중요하지 않습니다. 접근 제한자(public, protected, private)도 상관없습니다.

    ```java
    import javax.annotation.PostConstruct; // 또는 jakarta.annotation.PostConstruct
    
    public class CachingMovieLister {
        @PostConstruct // ★ 초기화 메소드임을 표시 ★
        public void populateMovieCache() {
            // 예: 데이터베이스에서 영화 정보를 읽어와 캐시에 미리 저장하는 로직
            System.out.println("캐시 초기화 시작...");
            // ... 캐시 채우는 작업 ...
            System.out.println("캐시 초기화 완료.");
        }
        // ...
    }
    ```


---

**3. `@PreDestroy` 사용법 및 역할:**

- **역할:** 빈의 **소멸(정리) 콜백**을 지정합니다.
- **호출 시점:** 빈이 컨테이너에서 제거되기 **직전**에 호출됩니다. 컨테이너가 종료되거나 해당 빈의 스코프가 끝날 때입니다.
  - 이는 `DisposableBean`의 `destroy()` 메소드나 `destroy-method`로 지정된 메소드와 **동일한 시점**에 해당합니다.
- **사용법:** 소멸(정리) 로직을 담고 있는 메소드 위에 `@PreDestroy` 어노테이션을 붙입니다. 메소드 시그니처 규칙은 `@PostConstruct`와 동일합니다 (인자 없음, 접근 제한자 무관).

    ```java
    import javax.annotation.PreDestroy; // 또는 jakarta.annotation.PreDestroy
    
    public class CachingMovieLister {
        // ... (@PostConstruct 메소드 등)
    
        @PreDestroy // ★ 소멸 메소드임을 표시 ★
        public void clearMovieCache() {
            // 예: 사용하던 캐시를 비우거나 관련 리소스를 해제하는 로직
            System.out.println("캐시 정리 시작...");
            // ... 캐시 비우는 작업 ...
            System.out.println("캐시 정리 완료.");
        }
    }
    ```


---

**4. 장점 및 권장 이유:**

- **표준:** `@PostConstruct`와 `@PreDestroy`는 **자바 표준(JSR-250, 현 Jakarta Annotations)** 이므로, 스프링뿐만 아니라 다른 프레임워크나 환경에서도 인식될 가능성이 있습니다. 코드의 이식성이 높아집니다.
- **낮은 결합도:** 빈 코드가 스프링 특정 인터페이스(`InitializingBean`, `DisposableBean`)에 의존하지 않게 됩니다. POJO(Plain Old Java Object) 상태를 유지하는 데 도움이 됩니다.
- **가독성:** 코드 내에서 해당 메소드가 어떤 역할을 하는지 (초기화용인지, 소멸용인지) 명확하게 드러납니다. 설정 파일을 뒤져볼 필요가 없습니다.
- **가장 권장되는 방식:** 현대 스프링 개발에서는 생명주기 콜백을 위해 이 어노테이션들을 사용하는 것이 **일반적으로 가장 권장**됩니다.

---

**5. 여러 생명주기 메커니즘 결합 시 순서:**

- 앞서 설명했듯이, 한 빈에 여러 콜백 방식이 혼합되어 사용될 경우 정해진 순서대로 실행됩니다.
  - 초기화: `@PostConstruct` -> `afterPropertiesSet()` -> `init-method`
  - 소멸: `@PreDestroy` -> `destroy()` -> `destroy-method`
- 이 어노테이션들은 각 단계에서 **가장 먼저** 실행됩니다.

---

**6. 중요: JDK 버전과 의존성 추가 (매우 실용적인 내용!)**

- **과거:** `@PostConstruct`, `@PreDestroy` 어노테이션은 **자바 6부터 자바 8까지는 JDK에 기본적으로 포함**되어 있었습니다 (`javax.annotation` 패키지).
- **현재:** **자바 9부터 모듈 시스템이 도입되면서 핵심 자바 모듈에서 제외되었고, 자바 11에서는 완전히 제거되었습니다.**
- **해결책:** 만약 **자바 9 이상 (특히 자바 11 이상)** 환경에서 스프링 프로젝트를 개발한다면, 이 어노테이션들을 사용하기 위해 **별도의 의존성을 추가**해주어야 합니다.
  - 이 어노테이션들은 이제 **Jakarta EE 스펙**의 일부이며, `jakarta.annotation-api` 라는 아티팩트(라이브러리)로 제공됩니다.
  - **Maven 사용 시:** `pom.xml` 파일에 다음과 같은 의존성을 추가해야 합니다.

      ```xml
      <dependency>
          <groupId>jakarta.annotation</groupId>
          <artifactId>jakarta.annotation-api</artifactId>
          <version>...</version> <!-- 적절한 버전 명시 -->
      </dependency>
      ```

  - **Gradle 사용 시:** `build.gradle` 파일에 다음과 같이 추가합니다.

      ```
      dependencies {
          implementation 'jakarta.annotation:jakarta.annotation-api:...' // 적절한 버전 명시
      }
      ```

- 이 의존성을 추가하지 않으면 `@PostConstruct`, `@PreDestroy` 어노테이션을 코드에서 `import` 할 수 없거나, 스프링이 제대로 인식하지 못할 수 있습니다.

**요약:**

`@PostConstruct`와 `@PreDestroy`는 각각 빈의 초기화 및 소멸 시점에 호출될 메소드를 지정하는 자바 표준 어노테이션입니다. 스프링은 `CommonAnnotationBeanPostProcessor`를 통해 이들을 인식하고 처리하며, 스프링 특정 인터페이스나 XML 설정보다 권장되는 방식입니다. 자바 9 이상 환경에서는 `jakarta.annotation-api` 의존성을 프로젝트에 추가해야 사용할 수 있습니다.
