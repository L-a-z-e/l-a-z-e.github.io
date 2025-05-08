---
title: Getting Spring Security
description: 
author: laze
date: 2025-05-08 00:00:01 +0900
categories: [Dev, SpringSecurity]
tags: [SpringSecurity]
---
## Spring Security 가져오기

이 섹션에서는 Spring Security 바이너리를 얻는 방법을 설명합니다. 소스 코드를 얻는 방법에 대해서는 소스 코드를 참조하세요.

### 릴리스 번호 체계

Spring Security 버전은 **MAJOR.MINOR.PATCH** 형식으로 지정되며, 각 의미는 다음과 같습니다:

- **MAJOR** 버전은 호환성이 깨지는 변경(breaking changes)을 포함할 수 있습니다. 일반적으로 이는 현대 보안 관행에 맞춰 보안을 개선하기 위해 수행됩니다.
- **MINOR** 버전은 개선 사항을 포함하지만, 호환성을 유지하는 업데이트(passive updates)로 간주됩니다.
- **PATCH** 레벨은 버그를 수정하는 변경을 제외하고는 이전 및 이후 버전과 완벽하게 호환되어야 합니다.

### 사용법

대부분의 오픈 소스 프로젝트와 마찬가지로, Spring Security는 의존성을 Maven 아티팩트로 배포하여 Maven과 Gradle 모두와 호환됩니다.

다음 섹션에서는 Spring Boot 및 독립 실행 환경에서의 예시와 함께 이러한 빌드 도구에 Spring Security를 통합하는 방법을 보여줍니다.

### Spring Boot

Spring Boot는 Spring Security 관련 의존성들을 모아놓은 `spring-boot-starter-security` 스타터를 제공합니다.

이 스타터를 사용하는 가장 간단하고 권장되는 방법은 IDE 통합(Eclipse, IntelliJ, NetBeans)을 이용하거나 start.spring.io를 통해 Spring Initializr를 사용하는 것입니다. 또는 다음 예시와 같이 수동으로 스타터를 추가할 수도 있습니다:

**Maven**

```xml
<dependencies>
	<!-- ... 다른 의존성 요소들 ... -->
	<dependency>
		<groupId>org.springframework.boot</groupId>
		<artifactId>spring-boot-starter-security</artifactId>
	</dependency>
</dependencies>
```

**Gradle**

```
dependencies {
	// ... 다른 의존성 요소들 ...
	implementation 'org.springframework.boot:spring-boot-starter-security'
}
```

Spring Boot는 의존성 버전을 관리하기 위해 Maven BOM (Bill of Materials)을 제공하므로 버전을 명시할 필요가 없습니다.

만약 Spring Security 버전을 재정의하고 싶다면, 아래와 같이 빌드 속성을 사용하여 재정의할 수 있습니다:

**Maven**

```xml
<properties>
	<!-- ... -->
	<spring-security.version>6.4.5</spring-security.version>
</properties>
```

**Gradle**

```
// build.gradle
ext['spring-security.version'] = '6.4.5' // 또는 gradle.properties 파일에 spring-security.version=6.4.5
```

Spring Security는 주요 릴리스에서만 호환성이 깨지는 변경을 하므로, Spring Boot와 함께 최신 버전의 Spring Security를 안전하게 사용할 수 있습니다.

그러나 때로는 Spring Framework 버전도 업데이트해야 할 수 있습니다. 다음과 같이 빌드 속성을 추가하여 업데이트할 수 있습니다:

**Maven**

```xml
<properties>
	<!-- ... -->
	<spring.version>6.2.6</spring.version>
</properties>
```

**Gradle**

```
// build.gradle
ext['spring.version'] = '6.2.6' // 또는 gradle.properties 파일에 spring.version=6.2.6
```

만약 추가 기능(LDAP, OAuth 2 등)을 사용한다면, 적절한 프로젝트 모듈 및 의존성도 포함해야 합니다.

### 독립 실행 환경 사용 (Spring Boot 없이)

Spring Boot 없이 Spring Security를 사용할 때 권장되는 방법은 Spring Security의 BOM을 사용하여 프로젝트 전체에서 일관된 버전의 Spring Security가 사용되도록 하는 것입니다.

**Maven**

```xml
<dependencyManagement>
	<dependencies>
		<!-- ... 다른 의존성 요소들 ... -->
		<dependency>
			<groupId>org.springframework.security</groupId>
			<artifactId>spring-security-bom</artifactId>
			<version>{spring-security-version}</version>
			<type>pom</type>
			<scope>import</scope>
		</dependency>
	</dependencies>
</dependencyManagement>
```

(Gradle의 경우 Spring은 Dependency Management Plugin을 제공합니다)

최소한의 Spring Security Maven 의존성 세트는 일반적으로 다음 예시와 같습니다:

**Maven**

```xml
<dependencies>
	<!-- ... 다른 의존성 요소들 ... -->
	<dependency>
		<groupId>org.springframework.security</groupId>
		<artifactId>spring-security-web</artifactId>
	</dependency>
	<dependency>
		<groupId>org.springframework.security</groupId>
		<artifactId>spring-security-config</artifactId>
	</dependency>
</dependencies>
```

**Gradle**

```
dependencies {
	// ... 다른 의존성 요소들 ...
	implementation 'org.springframework.security:spring-security-web'
	implementation 'org.springframework.security:spring-security-config'
}
```

만약 추가 기능(LDAP, OAuth 2 등)을 사용한다면, 적절한 프로젝트 모듈 및 의존성도 포함해야 합니다.

Spring Security는 Spring Framework 6.2.6 버전을 기준으로 빌드되지만, 일반적으로 Spring Framework 5.x의 최신 버전과도 작동합니다.

많은 사용자들이 Spring Security의 전이적 의존성이 Spring Framework 6.2.6을 가져오면서 이상한 클래스패스 문제를 겪을 수 있습니다.

이를 해결하는 가장 쉬운 방법은 `pom.xml`의 `<dependencyManagement>` 섹션이나 `build.gradle`의 `dependencyManagement` 섹션에 `spring-framework-bom`을 사용하는 것입니다.

**Maven**

```xml
<dependencyManagement>
	<dependencies>
		<!-- ... 다른 의존성 요소들 ... -->
		<dependency>
			<groupId>org.springframework</groupId>
			<artifactId>spring-framework-bom</artifactId>
			<version>6.2.6</version>
			<type>pom</type>
			<scope>import</scope>
		</dependency>
	</dependencies>
</dependencyManagement>
```

(Gradle의 경우 Spring은 Dependency Management Plugin을 제공합니다)

위 예시는 Spring Security의 모든 전이적 의존성이 Spring 6.2.6 모듈을 사용하도록 보장합니다.

이 접근 방식은 Maven의 "BOM (Bill of Materials)" 개념을 사용하며 Maven 2.0.9 이상에서만 사용할 수 있습니다. 의존성이 어떻게 해결되는지에 대한 자세한 내용은 Maven의 의존성 메커니즘 소개 문서를 참조하세요.

### Maven 저장소

모든 GA(정식) 릴리스는 Maven Central에 배포되므로 빌드 구성에 추가 Maven 저장소를 선언할 필요가 없습니다.

Gradle의 경우 GA 릴리스에는 `mavenCentral()` 저장소를 사용하는 것으로 충분합니다.

**build.gradle**

```
repositories {
	mavenCentral()
}
```

만약 SNAPSHOT 버전을 사용한다면, Spring Snapshot 저장소가 정의되어 있는지 확인해야 합니다:

**Maven**

```xml
<repositories>
	<!-- ... 다른 저장소 요소들 ... -->
	<repository>
		<id>spring-snapshot</id>
		<name>Spring Snapshot Repository</name>
		<url><https://repo.spring.io/snapshot></url>
	</repository>
</repositories>
```

**Gradle**

```
repositories {
	// ... 다른 저장소 요소들 ...
	maven { url '<https://repo.spring.io/snapshot>' }
}
```

만약 마일스톤 또는 릴리스 후보 버전을 사용한다면, 다음 예시와 같이 Spring Milestone 저장소가 정의되어 있는지 확인해야 합니다:

**Maven**

```xml
<repositories>
	<!-- ... 다른 저장소 요소들 ... -->
	<repository>
		<id>spring-milestone</id>
		<name>Spring Milestone Repository</name>
		<url><https://repo.spring.io/milestone></url>
	</repository>
</repositories>
```

**Gradle**

```
repositories {
	// ... 다른 저장소 요소들 ...
	maven { url '<https://repo.spring.io/milestone>' }
}
```

---

### Spring Security 시작하기: 준비물 챙기기 🚀

우리가 어떤 프로그램을 만들 때, "보안"은 정말 중요한 부분이에요.

예를 들어, 웹사이트에 로그인 기능을 만들거나, 특정 사용자만 접근할 수 있는 페이지를 만들고 싶을 때 Spring Security가 큰 도움을 준답니다.

이번 챕터에서는 Spring Security를 우리 프로젝트에 "어떻게 가져와서 사용할 수 있는지" 그 방법을 배우는 시간이라고 생각하시면 됩니다.

마치 요리를 시작하기 전에 필요한 재료를 준비하는 것과 같아요!

### 1. 버전 번호, 그거 뭔가요? 🤔

Spring Security도 계속 발전하면서 새로운 버전이 나와요.

버전은 보통 `MAJOR.MINOR.PATCH` (예: `6.1.5`) 이런 식으로 표시되는데요, 각 숫자는 의미가 있답니다.

- **`MAJOR` (예: `6`.1.5)**: 이 숫자가 바뀌면 **아주 큰 변화**가 있을 수 있어요. 기존에 잘 되던 기능이 안되거나 사용법이 바뀔 수도 있어서, MAJOR 버전이 올라갈 때는 문서를 꼼꼼히 봐야 해요. 보안 강화를 위해 큰 구조 변경이 있을 때 주로 바뀝니다.
- **`MINOR` (예: 6.`1`.5)**: 이 숫자는 새로운 기능이 추가되거나 기존 기능이 개선될 때 바뀌어요. 대부분 기존 코드와 호환되지만, 새로운 기능을 사용해 볼 수 있는 좋은 기회죠!
- **`PATCH` (예: 6.1.`5`)**: 작은 버그를 수정했을 때 이 숫자가 올라가요. 이전 버전과 거의 똑같이 사용할 수 있으니 안심하고 업데이트해도 좋습니다.

> 예시: 만약 여러분이 5.8.0 버전을 쓰고 있다가 6.0.0으로 업데이트한다면, "어? MAJOR 번호가 바뀌었네! 뭔가 크게 바뀌었을 수 있으니 조심해야겠다!" 하고 생각해야 해요. 반면 6.1.5에서 6.1.6으로 업데이트하는 건 비교적 안심하고 진행할 수 있죠.
>

### 2. Spring Security, 어떻게 내 프로젝트에 추가하나요? 🧩

Spring Security를 우리 프로젝트에 가져오는 방법은 크게 두 가지가 있어요.

**방법 1: Spring Boot와 함께 사용하기 (가장 쉽고 추천하는 방법! 👍)**

Spring Boot는 Spring 기반 애플리케이션을 정말 쉽고 빠르게 만들 수 있도록 도와주는 친구예요.

Spring Security도 Spring Boot와 함께 사용하면 아주 간단하게 설정할 수 있습니다.

- **핵심은 `spring-boot-starter-security`**: 이 친구는 Spring Security를 사용하는 데 필요한 여러 부품들(라이브러리)을 한 번에 가져다주는 "종합 선물 세트" 같은 거예요.
  - **비유:** 우리가 레고로 집을 만드는데, "보안 시스템 세트"(`spring-boot-starter-security`)를 하나 가져오면, 거기에 필요한 CCTV 블록, 경보음 블록, 문 잠금 블록 등이 한 번에 다 들어있는 것과 같아요!
- **어떻게 추가하나요?**
  - **Spring Initializr ([start.spring.io](http://start.spring.io/)):** 프로젝트를 처음 만들 때 "Security" 의존성을 선택하면 알아서 추가해줘요. (가장 편한 방법!)
  - **수동으로 추가:**
    - **Maven (`pom.xml` 파일):**

        ```xml
        <dependencies>
            <dependency>
                <groupId>org.springframework.boot</groupId>
                <artifactId>spring-boot-starter-security</artifactId>
            </dependency>
        </dependencies>
        
        ```

    - **Gradle (`build.gradle` 파일):**
      이렇게 한 줄만 추가해주면 끝! 버전 번호는 보통 적지 않아도 돼요. 왜냐하면 Spring Boot가 "이 Spring Boot 버전에는 이 Spring Security 버전이 가장 잘 맞아!" 하고 알아서 관리해주거든요. 이걸 **BOM (Bill of Materials)** 이라고 불러요. (잠시 후에 자세히 설명할게요!)

        ```
        dependencies {
            implementation 'org.springframework.boot:spring-boot-starter-security'
        }
        
        ```

- **버전 바꾸고 싶으면?**
  만약 특정 버전의 Spring Security를 쓰고 싶다면, `pom.xml`이나 `build.gradle`에 버전을 직접 명시할 수 있어요.
  - **Maven (`pom.xml`):**

      ```xml
      <properties>
          <spring-security.version>원하는.버전.번호</spring-security.version>
      </properties>
      
      ```


**방법 2: Spring Boot 없이 사용하기 (독립 실행 환경)**

Spring Boot를 사용하지 않는 경우에도 Spring Security를 쓸 수 있어요. 이때는 필요한 부품들을 우리가 직접 좀 더 챙겨줘야 합니다.

- **`spring-security-bom` 사용하기:** 여기서도 **BOM (Bill of Materials)** 이 등장해요! `spring-security-bom`은 Spring Security 관련 라이브러리들의 버전 호환성을 맞춰주는 "버전 관리 목록" 같은 거예요. 이걸 사용하면 버전 충돌 문제를 줄일 수 있어요.
  - **Maven (`pom.xml`):** `<dependencyManagement>` 안에 `spring-security-bom`을 추가해요.

      ```xml
      <dependencyManagement>
          <dependencies>
              <dependency>
                  <groupId>org.springframework.security</groupId>
                  <artifactId>spring-security-bom</artifactId>
                  <version>{spring-security-version}</version>
                  <type>pom</type>
                  <scope>import</scope>
              </dependency>
          </dependencies>
      </dependencyManagement>
      ```

- **필수 의존성 추가:** 최소한 웹 보안 기능과 설정 기능을 사용하려면 다음 두 가지를 추가해야 해요.
  - `spring-security-web`: 웹 환경에서 보안 기능을 제공해요 (URL 필터링 등).
  - `spring-security-config`: Java나 XML로 보안 설정을 할 수 있게 해줘요.
  - **Maven (`pom.xml`):**

      ```xml
      <dependencies>
          <dependency>
              <groupId>org.springframework.security</groupId>
              <artifactId>spring-security-web</artifactId>
          </dependency>
          <dependency>
              <groupId>org.springframework.security</groupId>
              <artifactId>spring-security-config</artifactId>
          </dependency>
      </dependencies>
      ```

- **Spring Framework 버전도 신경 써주세요!**
  Spring Security는 Spring Framework 위에서 동작해요. 그래서 서로 버전이 잘 맞아야 문제가 안 생겨요. 이때도 `spring-framework-bom`을 사용하면 Spring Framework 관련 라이브러리 버전을 일관성 있게 관리할 수 있어요.

### 3. BOM (Bill of Materials)이 뭐길래 자꾸 나올까요? 📜

BOM은 "자재 명세서" 또는 "부품 목록"이라는 뜻이에요. 소프트웨어에서는 **여러 라이브러리들의 검증된 버전 조합을 정의해놓은 목록**이라고 생각하면 쉬워요.

- **비유:** 우리가 맛있는 스파게티를 만들고 싶어요.
  - **BOM이 없다면:** "스파게티 면은 A사 제품, 토마토소스는 B사, 올리브 오일은 C사..." 이렇게 각각 골라야 하는데, 실수로 서로 안 어울리는 조합을 고를 수도 있겠죠? (버전 충돌!)
  - **BOM이 있다면:** "최고의 스파게티 만들기 세트" 목록(`BOM`)을 보고 그대로 사면, 면, 소스, 오일이 서로 환상의 궁합을 이루는 것들로 구성되어 있어서 실패할 확률이 줄어드는 거예요!
- **Spring Boot의 `spring-boot-starter-parent`나 `spring-security-bom`, `spring-framework-bom` 등이 이런 BOM 역할을 해서 의존성 관리를 아주 편하게 해줍니다.**

### 4. 라이브러리는 어디서 가져오나요? (Maven 저장소) 🏦

우리가 `pom.xml`이나 `build.gradle`에 의존성을 적으면, Maven이나 Gradle은 그 라이브러리들을 어딘가에서 다운로드 받아와야 해요. 그 "어딘가"가 바로 **Maven 저장소**입니다.

- **Maven Central:** 가장 기본적이고 큰 저장소예요. 대부분의 정식 버전(GA, General Availability) 라이브러리들은 여기에 있어요. Spring Security 정식 버전도 여기에 있어서, 별도 설정 없이 바로 다운로드 가능합니다.
- **특별한 저장소들:**
  - **SNAPSHOT 버전:** 아직 개발 중이고, 언제든 바뀔 수 있는 최신 개발 버전을 의미해요. 이런 버전들은 `Spring Snapshot Repository`라는 특별한 저장소에 있어요.
  - **Milestone/RC 버전:** 정식 출시 전에 미리 테스트해보는 버전(Milestone, Release Candidate)이에요. 이런 것들은 `Spring Milestone Repository`에 있습니다.
  - 만약 이런 SNAPSHOT이나 Milestone 버전을 사용하고 싶다면, `pom.xml`이나 `build.gradle`에 해당 저장소 주소를 추가해줘야 해요.

> 정리:
>
> - **Spring Boot 사용자:** `spring-boot-starter-security` 하나면 대부분 끝! 버전 관리는 Spring Boot가 BOM으로 알아서 잘 해줌.
> - **BOM:** 라이브러리 버전 조합을 관리해주는 편리한 목록.
> - **저장소:** 라이브러리를 다운로드 받는 곳. 정식 버전은 `Maven Central`에 다 있음.
