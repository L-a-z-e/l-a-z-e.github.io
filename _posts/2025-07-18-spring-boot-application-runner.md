---
title: ApplicationRunner 제대로 사용하는 법
description: 
author: laze
date: 2025-07-18 00:00:01 +0900
categories: [Dev, SpringBoot]
tags: [SpringBoot, ApplicationRunner, CommandLineRunner]
---
# [Spring Boot] `main`에 코드를 넣지 마세요: `ApplicationRunner` 제대로 사용하는 법

## 1. 서론: "애플리케이션 시작할 때 딱 한 번 실행하고 싶은 코드가 있는데..."

Spring Boot로 애플리케이션을 개발하다 보면, "애플리케이션이 시작될 때, 딱 한 번만 실행하고 싶은 초기화 작업"이 필요한 경우가 많습니다.

- 개발 환경에서 사용할 초기 데이터 DB에 삽입하기
- 애플리케이션 구동 시 캐시 미리 데워놓기(Cache-warming)
- 필요한 설정 값이나 외부 리소스를 미리 로딩하기

이런 요구사항에 직면했을 때, 많은 주니어 개발자들은 가장 익숙한 `main` 메서드에 해당 로직을 넣으려는 시도를 합니다.

```java
@SpringBootApplication
public class MyApplication {

    // 이런 시도는 실패한다!
    // @Autowired
    // private UserRepository userRepository;

    public static void main(String[] args) {
        // userRepository.save(...); // userRepository는 null이므로 NullPointerException 발생!
        SpringApplication.run(MyApplication.class, args);
    }
}
```

하지만 위 코드는 어김없이 `NullPointerException`을 발생시킵니다.

대체 왜일까요? 이 글에서는 왜 `main` 메서드에 코드를 넣으면 안 되는지, 그리고 Spring Boot가 제공하는 우아한 해결책인 `ApplicationRunner`를 어떻게 제대로 사용하는지 알아보겠습니다.

## 2. `main` 메서드의 한계: Spring 세상 밖의 코드

`main` 메서드에서 `userRepository`가 `null`인 이유는 간단합니다. `main` 메서드가 실행되는 시점은 **Spring 컨테이너가 만들어지기 전**이기 때문입니다.

Spring Boot 애플리케이션의 실행 순서를 보면 명확히 알 수 있습니다.

```java
public static void main(String[] args) {
    // [시점 1] Spring 세상의 "밖"
    // 이 시점은 순수한 Java의 영역입니다.
    // Spring 컨테이너(ApplicationContext)가 아직 존재하지 않으므로,
    // @Component, @Service, @Repository 등으로 등록된 Bean들도 당연히 존재하지 않습니다.
    // 따라서 @Autowired를 통한 의존성 주입은 불가능합니다.

    SpringApplication.run(MyApplication.class, args);

    // [시점 2] 애플리케이션 종료 후
    // 웹 애플리케이션의 경우, 서버가 종료되기 전까지 이 코드는 실행되지 않습니다.
}
```

`main` 메서드는 오직 `SpringApplication.run()`을 호출하여 Spring 세상을 깨우는 '점화 장치' 역할만 할 뿐, Spring의 제어(IoC)를 받는 공간이 아닙니다.

## 3. Spring의 해답: `ApplicationRunner`

그렇다면, Spring의 모든 혜택(DI, AOP 등)을 누리면서 초기화 코드를 실행하려면 어떻게 해야 할까요?

바로 이럴 때 사용하라고 Spring Boot가 제공하는 인터페이스가 `ApplicationRunner`와 `CommandLineRunner`입니다.

`ApplicationRunner` 인터페이스를 구현한 클래스를 Bean으로 등록해두면, Spring Boot는 **"모든 Bean의 생성과 의존성 주입이 완료된 직후, 애플리케이션이 본격적으로 요청을 받기 바로 전"** 이라는 완벽한 타이밍에 `run` 메서드를 호출해 줍니다.

```java
import org.springframework.boot.ApplicationArguments;
import org.springframework.boot.ApplicationRunner;
import org.springframework.stereotype.Component;

@Component // 반드시 Bean으로 등록해야 Spring이 인식할 수 있다.
public class MyInitializer implements ApplicationRunner {

    // 생성자 주입을 통해 모든 Bean을 안전하게 사용할 수 있다.
    private final UserRepository userRepository;

    public MyInitializer(UserRepository userRepository) {
        this.userRepository = userRepository;
    }

    @Override
    public void run(ApplicationArguments args) throws Exception {
        // 이 시점에서는 userRepository가 완벽하게 주입되어 있다.
        System.out.println(">> ApplicationRunner가 실행되었습니다!");
        userRepository.save(new User("admin", "admin@test.com"));
    }
}
```

이제 `main` 메서드는 깨끗하게 유지하고, 모든 초기화 로직은 `ApplicationRunner`에게 위임할 수 있습니다.

## 4. 실전 활용 패턴: 언제 `ApplicationRunner`를 사용하는가?

`ApplicationRunner`는 "모든 준비가 끝난 후, 딱 한 번 실행하고 싶은 초기화 작업"에 매우 유용합니다.

### 패턴 1: 초기 데이터 적재 (Initial Data Loading)

개발 환경에서 애플리케이션을 실행할 때마다 테스트용 데이터를 DB에 자동으로 생성하는 경우입니다.

```java
@Profile("dev") // "dev" 프로파일일 때만 이 Runner가 실행되도록 설정
@Component
@RequiredArgsConstructor
public class DevDataInitializer implements ApplicationRunner {

    private final UserRepository userRepository;
    private final PostRepository postRepository;

    @Override
    public void run(ApplicationArguments args) throws Exception {
        if (userRepository.count() == 0) {
            User user1 = userRepository.save(User.builder().name("laze1").email("laze1@test.com").build());
            User user2 = userRepository.save(User.builder().name("laze2").email("laze2@test.com").build());

            postRepository.save(Post.builder().title("테스트 게시글 1").author(user1).build());
            postRepository.save(Post.builder().title("테스트 게시글 2").author(user2).build());
        }
    }
}
```

### 패턴 2: 캐시 예열 (Cache Warming)

애플리케이션 구동 후 첫 번째 사용자 요청이 느려지는 것을 방지하기 위해, 자주 사용되는 데이터를 미리 조회하여 캐시에 저장해 두는 경우입니다.

```java
@Component
@RequiredArgsConstructor
public class CacheWarmerRunner implements ApplicationRunner {

    private final ProductService productService;
    private final CacheManager cacheManager;

    @Override
    public void run(ApplicationArguments args) throws Exception {
        System.out.println("캐시 예열을 시작합니다...");
        // "products" 캐시에 인기 상품 목록을 미리 로드
        productService.findPopularProducts();
        System.out.println("캐시 예열 완료.");
    }
}
```

`productService.findPopularProducts()` 메서드에는 `@Cacheable("products")`가 적용되어 있다고 가정합니다.

### 패턴 3: 배치 작업 실행

웹 애플리케이션이 구동되면서, 특정 배치 잡(Batch Job)을 한 번 실행시켜야 할 때도 유용합니다.

```java
@Component
@RequiredArgsConstructor
public class BatchJobRunner implements ApplicationRunner {

    private final JobLauncher jobLauncher;
    private final Job dailySummaryJob;

    @Override
    public void run(ApplicationArguments args) throws Exception {
        JobParameters jobParameters = new JobParametersBuilder()
                .addDate("runDate", new Date())
                .toJobParameters();
        jobLauncher.run(dailySummaryJob, jobParameters);
    }
}
```

## 5. 쌍둥이 형제: `ApplicationRunner` vs `CommandLineRunner`

Spring Boot에는 `ApplicationRunner`와 거의 동일한 역할을 하는 `CommandLineRunner`도 있습니다. 둘의 유일한 차이는 `run` 메서드가 받는 파라미터 타입입니다.

- **`ApplicationRunner`:** `void run(ApplicationArguments args)`
  - `ApplicationArguments` 객체를 통해 `-name=value` 형식의 옵션 인자와 그 외의 일반 인자를 구분하여 다룰 수 있습니다. 더 구조화된 방식입니다.
- **`CommandLineRunner`:** `void run(String... args)`
  - 커맨드 라인 인자를 단순한 `String` 배열로 받습니다. 개발자가 직접 파싱해야 합니다.

간단한 실행만 필요하다면 둘 중 어느 것을 사용해도 무방하지만, 복잡한 커맨드 라인 인자를 처리해야 한다면 `ApplicationRunner`가 더 편리합니다.

## 6. 실행 순서 제어하기: `@Order` 어노테이션

하나의 애플리케이션에 여러 개의 Runner가 있다면, 실행 순서를 보장해야 할 때가 있습니다. 예를 들어, 데이터가 먼저 로딩된 후에 캐시 예열을 해야 합니다. 이때 `@Order` 어노테이션을 사용하면 됩니다. 숫자가 낮을수록 먼저 실행됩니다.

```java
@Component
@Order(1) // 가장 먼저 실행
public class DataLoaderRunner implements ApplicationRunner { /* ... */ }

@Component
@Order(2) // DataLoaderRunner 다음에 실행
public class CacheWarmerRunner implements ApplicationRunner { /* ... */ }
```

## 7. 결론

`ApplicationRunner`는 **의존성 주입(DI)의 모든 혜택을 누리며, 애플리케이션 시작 시점에 안전하게 초기화 코드를 실행하기 위한** Spring Boot의 표준적이고 우아한 방법입니다.

더 이상 `main` 메서드에 비즈니스 로직을 넣으려 시도하지 마세요. `main` 메서드는 오직 `SpringApplication.run()`을 호출하여 Spring 세상을 깨우는 역할에만 집중시키고, 모든 애플리케이션 레벨의 초기화 로직은 `ApplicationRunner`에게 위임하는 것이 올바른 설계입니다.
