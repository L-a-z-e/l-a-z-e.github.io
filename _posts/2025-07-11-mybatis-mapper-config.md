---
title: MyBatis - Mapper가 동작하지 않을 때 확인해야 할 모든 것 (`mapper-locations`와 `@MapperScan` 가이드)
description: 
author: laze
date: 2025-07-11 00:00:01 +0900
categories: [Dev, SpringBoot]
tags: [MyBatis, mapper-locations, MapperScan]
---
# [MyBatis] Mapper가 동작하지 않을 때 확인해야 할 모든 것 (`mapper-locations`와 `@MapperScan` 가이드)

## 1. 서론: "분명히 설정했는데 왜 SQL을 못 찾을까?"

Spring Boot 환경에서 MyBatis를 연동할 때, 많은 개발자들이 `org.apache.ibatis.binding.BindingException: Invalid bound statement (not found)` 라는 예외를 마주한다.

설정 파일을 몇 번이고 확인하고, 경로도 올바른 것 같은데 대체 왜 MyBatis는 내가 작성한 SQL을 찾지 못하는 것일까?

이 문제의 원인은 대부분 MyBatis의 두 가지 핵심 설정, **`@MapperScan`**과 **`mybatis.mapper-locations`**의 연결고리가 어딘가에서 끊어졌기 때문이다.

이 글에서는 이 두 설정의 내부 동작 원리를 파헤치고, 다양한 프로젝트 구조에 맞는 설정 방법, 그리고 가장 중요한 **오류 발생 시 원인을 추적하는 디버깅 방법**까지 심층적으로 다룬다.

## 2. MyBatis의 두 가지 핵심 연결고리: 원리 이해하기

MyBatis가 Spring과 함께 동작하기 위해서는 다음 두 단계가 순서대로 성공해야 한다.

1. **`@MapperScan`**: Mapper 인터페이스를 찾아 Spring Bean으로 등록한다.
  - Spring은 `@MapperScan`에 지정된 `basePackages`를 탐색하여 Mapper 인터페이스들을 찾는다.
  - 찾아낸 각 인터페이스에 대해, 실제 SQL을 실행할 수 있는 **프록시(Proxy) 객체**를 생성하여 Spring 컨테이너에 Bean으로 등록한다.
  - 이 덕분에 우리는 서비스 레이어에서 `@Autowired`로 Mapper 인터페이스를 주입받을 수 있다.
2. **`mybatis.mapper-locations`**: Mapper XML 파일을 로딩하여 SQL을 준비한다.
  - MyBatis는 이 설정에 명시된 경로 패턴을 사용하여 프로젝트 내의 모든 Mapper XML 파일을 찾아서 파싱한다.
  - 파싱된 XML 내부의 SQL 구문들은 `<mapper namespace="...">`에 지정된 값과 `<select id="...">` 같은 id의 조합을 키(Key)로 하여 메모리(MyBatis `Configuration` 객체)에 저장된다.

**[최종 연결]**
서비스에서 Mapper 인터페이스의 메서드(예: `userMapper.findById(1L)`)를 호출하면, **1번**에서 생성된 프록시 Bean이 동작한다. 이 프록시는 호출된 메서드 정보(인터페이스 전체 경로 + 메서드 이름)를 이용해, **2번**에서 메모리에 로딩된 SQL 맵에서 일치하는 키(`com.example.UserMapper.findById`)를 찾아 실제 SQL을 실행한다.

`Invalid bound statement` 오류는 바로 이 **[최종 연결]** 단계에서, 프록시가 실행할 SQL을 메모리에서 찾지 못했을 때 발생한다.

## 3. Case Study 1: `mybatis.mapper-locations` 경로 패턴 정복하기

가장 흔한 오류의 원인이다. `mapper-locations`의 경로는 **항상 `classpath:`의 루트(보통 `src/main/resources` 또는 빌드 후 `target/classes`)에서 시작**한다는 점을 기억해야 한다.

### 프로젝트 구조 예시

```
src/main/java/
└── com/example/
    ├── user/
    │   └── mapper/
    │       └── UserMapper.java
    └── order/
        └── mapper/
            └── OrderMapper.java
src/main/resources/
├── mappers/
│   ├── user/
│   │   └── user-mapper.xml
│   └── order/
│       └── order-mapper.xml
└── application.yml
```

### 설정 방법: `application.yml` vs. Java Config

**1) `application.yml` (권장)**
가장 간편하고 일반적인 방법이다.

```yaml
mybatis:
  # type-aliases-package: DTO/VO 클래스의 패키지를 지정하면 XML에서 짧은 별칭으로 사용 가능.
  type-aliases-package: com.example.user.dto, com.example.order.dto
  # mapper-locations: classpath* 와 ** 패턴을 활용하여 모든 XML을 찾는다.
  mapper-locations: classpath:mappers/**/*.xml
```

- **`classpath:mappers/**/*.xml`** 의미:
  - `classpath:`: `src/main/resources` 디렉토리에서 찾기 시작.
  - `mappers/`: `mappers` 폴더 안으로 들어감.
  - `**/`: `mappers` 폴더 아래의 모든 하위 디렉토리를 포함.
  - `.xml`: 확장자가 `.xml`인 모든 파일.
- **`classpath*:` vs `classpath:`**: 만약 프로젝트가 여러 모듈(JAR)로 구성되어 있고, 다른 모듈에 있는 Mapper XML까지 찾아야 한다면 `classpath:` 대신 `classpath*:`를 사용해야 한다. 일반적으로 `classpath*:`를 쓰는 것이 더 안전하다.

**2) Java Config**`application.yml` 대신 Java 코드로 직접 `SqlSessionFactory`를 설정할 수도 있다.

```java
@Configuration
public class MybatisConfig {
    @Bean
    public SqlSessionFactory sqlSessionFactory(DataSource dataSource) throws Exception {
        SqlSessionFactoryBean sessionFactoryBean = new SqlSessionFactoryBean();
        sessionFactoryBean.setDataSource(dataSource);

        Resource[] res = new PathMatchingResourcePatternResolver().getResources("classpath:mappers/**/*.xml");
        sessionFactoryBean.setMapperLocations(res);

        return sessionFactoryBean.getObject();
    }
}
```

## 4. Case Study 2: `@MapperScan` 설정 전략

`@MapperScan`은 Mapper 인터페이스를 Spring Bean으로 등록하는 역할을 한다.

### 전략 1: 단일 `@MapperScan`으로 중앙 관리 (Best Practice)

`@MapperScan`은 애플리케이션의 메인 클래스(`*Application.java`)에 **단 한 번만 선언**하고, 모든 Mapper 인터페이스를 포함하는 **가장 상위의 공통 패키지**를 `basePackages`로 지정하는 것이 가장 이상적이다.

```java
// 모든 Mapper는 com.example 패키지 하위에 위치
// 예: com.example.user.mapper, com.example.order.mapper

import org.mybatis.spring.annotation.MapperScan;
import org.springframework.boot.SpringApplication;
import org.springframework.boot.autoconfigure.SpringBootApplication;

@SpringBootApplication
@MapperScan(basePackages = "com.example")
public class MyApplication {
    public static void main(String[] args) {
        SpringApplication.run(MyApplication.class, args);
    }
}
```

- **장점:** 설정이 한 곳에 있어 관리가 용이하고, 새로운 도메인/모듈이 추가되어도 `com.example` 하위에 Mapper를 만들기만 하면 자동으로 인식된다.

### 전략 2: 다중 `@MapperScan` 사용 (특수한 경우)

프로젝트 구조가 복잡하여 공통 상위 패키지가 없거나, 모듈별로 설정을 분리해야 하는 불가피한 경우에 사용할 수 있다.

```java
// 각 도메인별 설정 클래스에 @MapperScan을 분리
@Configuration
@MapperScan(basePackages = "com.mycompany.user.mapper")
public class UserModuleConfig { /* ... */ }

@Configuration
@MapperScan(basePackages = "com.mycompany.payment.mapper")
public class PaymentModuleConfig { /* ... */ }

```

- **단점:**
  - 설정이 여러 곳에 흩어져 있어 추적이 어렵다.
  - 새로운 모듈 추가 시 `@MapperScan` 설정을 누락할 위험이 있다.
  - 꼭 필요한 경우가 아니라면 지양해야 할 구조다.

## 5. Troubleshooting: `Invalid bound statement` 오류 디버깅하기

이 오류가 발생했을 때, 무작정 코드를 수정하기보다 **오류의 원인이 '연결고리 1번'인지 '2번'인지**를 먼저 파악해야 한다.

### Step 1: Mapper Bean이 정상적으로 등록되었는가? (연결고리 1번 확인)

먼저 `@MapperScan`이 제대로 동작하여 Mapper 인터페이스가 Spring Bean으로 등록되었는지 확인한다.
가장 간단한 방법은, 해당 Mapper를 사용하는 서비스 클래스에서 `@Autowired`가 실패하는지 보는 것이다.

```java
@Service
public class UserService {
    private final UserMapper userMapper;

    // 만약 UserMapper Bean이 없다면, 애플리케이션 구동 시점에서
    // "No qualifying bean of type 'com.example.user.mapper.UserMapper' available"
    // 같은 에러가 발생한다.
    @Autowired
    public UserService(UserMapper userMapper) {
        this.userMapper = userMapper;
    }
}
```

만약 위와 같은 Bean 주입 실패 에러가 발생한다면, 원인은 `@MapperScan` 설정 오류다. `basePackages` 경로에 오타가 있거나, 범위가 잘못되었을 가능성이 99%다.

### Step 2: SQL이 메모리에 로딩되었는가? (연결고리 2번 확인 및 디버깅)

Bean 주입은 성공했는데 `Invalid bound statement`가 발생한다면, 문제는 `mapper-locations` 설정 또는 XML 파일 자체에 있다. 즉, **Bean은 있는데 실행할 SQL을 못 찾는 상황**이다.

이때는 MyBatis의 내부 동작을 들여다보는 디버깅이 매우 효과적이다.

1. **디버그 포인트 설정:** `org.apache.ibatis.binding.MapperMethod` 클래스의 `execute` 메서드에 브레이크포인트를 건다. 이곳이 모든 Mapper 메서드 호출이 최종적으로 처리되는 지점이다.
2. **애플리케이션 디버그 모드로 실행:** 오류가 발생하는 API를 호출하거나 테스트 코드를 실행한다.
3. **`configuration` 객체 확인:** 브레이크포인트에 멈추면, `this.command` 필드를 통해 `org.apache.ibatis.session.Configuration` 객체에 접근할 수 있다. 이 객체는 MyBatis의 모든 설정을 담고 있는 핵심 객체다.
4. **`mappedStatements` 맵 탐색:** `configuration` 객체 내부의 `mappedStatements` 라는 `Map`을 확인한다.
  - 이 맵의 **Key**는 `"인터페이스_전체경로.메서드이름"` (e.g., `"com.example.user.mapper.UserMapper.findById"`) 형태이다.
  - 이 맵의 **Value**는 해당 SQL의 모든 정보(SQL 쿼리, 파라미터 타입, 결과 타입 등)를 담고 있는 `MappedStatement` 객체다.
5. **원인 분석:**
  - **Case A: `mappedStatements` 맵에 내가 찾으려는 Key가 아예 존재하지 않는 경우**
    - **원인:** `mybatis.mapper-locations` 경로 설정이 잘못되어 해당 XML 파일을 아예 읽어오지 못했거나, 빌드 시 XML 파일이 누락된 것이다. `classpath:mappers/**/*.xml` 경로가 올바른지, 빌드 결과물(`target/classes`)에 XML이 복사되었는지 확인한다.
  - **Case B: Key는 존재하는데, 내 코드의 네임스페이스/ID와 다른 경우**
    - **원인:** `namespace`나 메서드 `id`에 오타가 있는 것이다. `mappedStatements`에 등록된 키 목록을 쭉 살펴보면, 내가 의도한 이름과 미묘하게 다른 키(e.g., `UserMapper.findbyid` - 대소문자 오류)를 발견할 수 있다.

이 디버깅 방법을 통해 "왜 못 찾는지"를 추측이 아닌 사실에 기반하여 정확하게 진단할 수 있다.

## 6. 결론: 명확한 규칙이 유지보수성을 높인다

MyBatis 설정 오류는 대부분 사소한 오타나 경로 불일치에서 비롯되지만, 그 원인을 찾는 과정은 매우 고통스러울 수 있다. 프로젝트 시작 단계부터 다음과 같은 명확한 규칙을 정립하면 이러한 문제를 크게 줄일 수 있다.

1. **`@MapperScan`은 최상위 설정 클래스에 단 하나만 사용한다.** (`basePackages`는 가장 넓은 범위로)
2. **`mapper-locations`는 `classpath*:`와 `*` 패턴으로 유연하게 설정한다.**
3. **Mapper 인터페이스의 FQCN과 XML의 `namespace`는 반드시 1:1로 정확히 일치시킨다.**

이 세 가지 원칙을 지키고, 문제가 발생했을 때 위에서 제시한 디버깅 방법을 활용한다면 더 이상 MyBatis 설정 때문에 시간을 낭비하는 일은 없을 것이다.

---
