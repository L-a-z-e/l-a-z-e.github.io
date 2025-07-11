---
title: JobRunner 충돌 (incrementer(), JobInstanceAlreadyCompleteException)
description: 
author: laze
date: 2025-07-10 00:00:01 +0900
categories: [Dev, SpringBatch]
tags: [SpringBatch, JobRunner, incrementer]
---
# [Spring Batch 심층 분석] 내가 만든 JobRunner가 실패하는 이유: `JobLauncherApplicationRunner`와의 충돌 해결하기

## 1. 문제 상황: "내 코드는 완벽한데, `JobInstanceAlreadyCompleteException`이 발생한다"

Spring Batch로 커맨드 라인 기반의 배치 애플리케이션을 개발하다 보면, 매우 당황스러운 예외를 만날 때가 있다. 분명히 처음 실행하는 것인데도, 마치 동일한 파라미터로 재실행한 것처럼 `JobInstanceAlreadyCompleteException`이 발생하는 경우다.

```java
// Job 실행을 위해 직접 만든 CommandLineRunner
@Component
public class MyCustomJobRunner implements CommandLineRunner {
    @Override
    public void run(String... args) throws Exception {
        // 커맨드 라인 인자(args)를 파싱하고
        // new JobParametersBuilder().addDate("run.timestamp", new Date()) ...
        // 항상 새로운 타임스탬프를 추가하여 JobParameters를 만드는데도 불구하고...
        jobLauncher.run(myJob, jobParameters);
    }
}
```

위와 같이 매번 고유한 파라미터를 추가하는 로직을 넣었음에도 불구하고, 아래와 같은 예외가 발생하며 애플리케이션이 실패한다.

```
Caused by: org.springframework.batch.core.repository.JobInstanceAlreadyCompleteException:
A job instance already exists and is complete for parameters={your_param=some_value}.
If you want to run this job again, use a different set of parameters.
```

로그를 자세히 보면 더 이상한 점이 발견된다. 예외 메시지에 찍힌 `parameters`에 내가 추가한 고유 파라미터(`run.timestamp` 등)가 보이지 않는다. 대체 무슨 일이 벌어지고 있는 것일까?

## 2. 범인 추적: 보이지 않는 경쟁자, `JobLauncherApplicationRunner`

이 미스터리의 범인은 바로 Spring Boot의 '자동 설정(Auto-Configuration)' 기능 속에 숨어있다. 우리가 만든 `MyCustomJobRunner` 외에, 또 다른 Job 실행기가 조용히, 그리고 먼저 동작하고 있었던 것이다.

**범인의 정체: `org.springframework.boot.autoconfigure.batch.JobLauncherApplicationRunner`**

- **역할:** Spring Boot가 기본적으로 제공하는 `ApplicationRunner` 구현체.
- **임무:** 애플리케이션 구동 시 `application.properties`나 커맨드 라인 인자를 분석하여, **조건에 맞는 배치 잡(Job)을 자동으로 실행**한다.

즉, 우리의 애플리케이션에는 최소 두 개의 Job 실행기가 존재했다.

1. **`MyCustomJobRunner`**: 우리가 직접 만든, 명시적으로 제어하려는 실행기.
2. **`JobLauncherApplicationRunner`**: Spring Boot가 친절하게(?) 제공하는, 자동화된 실행기.

이 두 Runner는 모두 `(Command|Application)Runner` 인터페이스를 구현했기 때문에, Spring Boot는 애플리케이션 구동이 완료되면 두 Runner를 모두 실행하려고 시도한다.

## 3. 사건 재구성: 꼬여버린 실행 타임라인 분석

이 문제가 어떻게 `JobInstanceAlreadyCompleteException`으로 이어지는지, 애플리케이션 실행 타임라인을 따라가며 사건을 재구성해 보자.

**[T=0] 실행 명령**

```bash
java -jar my-batch-app.jar myJob targetDate=2024-01-01
```

**[T=1] Spring Boot 구동**
스프링 컨테이너가 로딩되며, `MyCustomJobRunner`와 `JobLauncherApplicationRunner`가 모두 Bean으로 등록되고 실행 대기열에 오른다.

**[T=2] 경쟁자의 선제공격: `JobLauncherApplicationRunner`의 자동 실행**
특별한 순서가 지정되지 않았다면, Spring Boot 내장 Runner가 먼저 실행될 가능성이 높다.

1. `JobLauncherApplicationRunner`가 먼저 동작한다.
2. 이 Runner는 **자체적인 규칙**으로 커맨드 라인 인자를 파싱한다. 이 규칙은 매우 단순해서, 로 시작하지 않는 인자(`targetDate=2024-01-01`)를 Job의 파라미터로 인식한다.
3. **핵심:** 이 자동화된 규칙에는 우리가 `MyCustomJobRunner`에 넣었던 `run.timestamp` 같은 고유 파라미터를 추가하는 로직이 **전혀 없다.**
4. 결과적으로 `{'targetDate': '2024-01-01'}`라는 단순한 `JobParameters`로 `myJob`을 실행한다.
5. DB의 배치 메타데이터 테이블에는 해당 파라미터로 실행된 기록이 없으므로, 첫 실행은 **성공(COMPLETED)** 처리된다.

**[T=3] 우리의 주자 등판: `MyCustomJobRunner`의 실행 시도**
이제 우리가 만든 `MyCustomJobRunner`가 실행될 차례다.

1. `run` 메서드가 호출되고, 우리의 로직에 따라 `{'targetDate': '2024-01-01', 'run.timestamp': 1704067200000L}` 같은 고유 파라미터가 포함된 `JobParameters`가 생성된다.
2. `jobLauncher.run()`을 호출하여 잡 실행을 시도한다.

**[T=4] `JobRepository`에서의 충돌**

1. `JobRepository`는 새로운 잡 실행 요청을 받으면, 이 잡이 과거에 동일한 **'식별 파라미터(Identifying Parameters)'**로 성공한 적이 있는지 확인한다.
2. `JobParameter`는 '식별(identifying)' 여부를 나타내는 플래그를 가지는데, `RunIdIncrementer` 등이 추가하는 `run.id` 같은 일부 파라미터를 제외하면 대부분의 파라미터는 기본적으로 '식별' 파라미터다.
3. `JobRepository`는 `targetDate=2024-01-01`라는 식별 파라미터를 기준으로 DB를 확인한다.
4. **발견!** 이미 `[T=2]` 단계에서 `JobLauncherApplicationRunner`가 실행하고 성공시킨 `JobInstance`가 DB에 존재한다.
5. `JobRepository`는 "이 식별 파라미터 조합으로는 이미 성공한 잡이 존재합니다!"라며 `JobInstanceAlreadyCompleteException`을 던진다.
6. 이 예외는 `MyCustomJobRunner`에서 처리되지 않았으므로 위로 전파되고, 결국 애플리케이션은 비정상적으로 종료된다.

## 4. 해결책: 경쟁자를 잠재우고 실행의 주도권 되찾기

문제의 원인은 우리 코드의 로직 오류가 아니라, **의도치 않은 자동 실행 기능과의 충돌**이었다. 따라서 해결책은 이 자동 실행 기능을 비활성화하여, 우리의 `MyCustomJobRunner`가 유일한 실행 주체가 되도록 만드는 것이다.

`src/main/resources/application.properties` (또는 `.yml`) 파일에 다음 설정을 추가한다.

```
# ==============================
# Spring Batch 설정
# ==============================
# Spring Boot가 애플리케이션 시작 시 자동으로 Job을 실행하는 기능을 비활성화합니다.
# 이렇게 해야 우리가 만든 커스텀 Runner만 동작하게 됩니다.
spring.batch.job.enabled=false
```

이 `spring.batch.job.enabled=false` 설정 하나가 `JobLauncherApplicationRunner`의 동작을 멈추게 하는 스위치 역할을 한다. 이제 애플리케이션이 시작되어도 이 '보이지 않는 경쟁자'는 아무 일도 하지 않고, 오직 `MyCustomJobRunner`만이 잡을 실행시키는 유일한 주체가 되어 우리가 의도한 대로 완벽하게 동작하게 된다.

## 5. 심화 학습: Spring Boot 자동 설정(Auto-Configuration)을 대하는 자세

이번 사례는 Spring Boot의 강력한 자동 설정 기능이 어떻게 동작하는지 엿볼 수 있는 좋은 기회다. Spring Boot는 `spring-boot-autoconfigure.jar` 내부에 수많은 `@Configuration` 클래스들을 가지고 있다. 이 클래스들은 `@ConditionalOn...` 어노테이션을 통해 "특정 조건이 만족될 때만 이 Bean을 등록하라"는 식으로 동작한다.

`JobLauncherApplicationRunner` 역시 `@ConditionalOnProperty(prefix = "spring.batch.job", name = "enabled", havingValue = "true", matchIfMissing = true)`와 같은 조건에 의해 활성화된다. 즉, `spring.batch.job.enabled` 속성이 `false`가 아닐 경우 (기본값은 `true`) 자동으로 활성화되는 것이다.

우리가 `false`로 설정한 것은, 이 조건을 깨뜨려 `JobLauncherApplicationRunner`가 스프링 컨테이너에 등록되지 않도록 명시적으로 제어한 것이다.

## 6. 결론

"내 코드는 완벽한데 이유를 알 수 없는 예외가 발생한다"면, 혹시 Spring Boot의 자동 설정 기능이 보이지 않는 곳에서 예상치 못한 동작을 하고 있지는 않은지 의심해볼 필요가 있다.

이번 `JobInstanceAlreadyCompleteException` 문제의 원인은 **두 개의 Runner가 잡 실행을 두고 경쟁**한 것이었다. 그리고 `spring.batch.job.enabled=false` 설정은 이 경쟁에서 **자동화된 Runner를 제외시키고, 개발자가 만든 Runner에게 실행의 주도권을 온전히 넘겨주는** 핵심적인 해결책이다.

Spring Boot의 자동화는 매우 강력하지만, 그 내부 동작을 이해하고 필요할 때 명시적으로 제어할 수 있어야 진정한 전문가로 거듭날 수 있다.

---
