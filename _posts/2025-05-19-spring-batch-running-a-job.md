---
title: Spring Batch - Running a Job
description: 
author: laze
date: 2025-05-19 00:00:05 +0900
categories: [Dev, SpringBatch]
tags: [SpringBatch]
---
**Job 실행하기**

최소한 배치 Job을 시작하려면 두 가지, 즉 시작할 `Job`과 `JobLauncher`가 필요합니다.

이 둘은 동일한 컨텍스트 또는 다른 컨텍스트에 포함될 수 있습니다.

예를 들어, 커맨드 라인에서 Job을 시작하는 경우 각 `Job`에 대해 새로운 JVM이 인스턴스화됩니다.

따라서 모든 Job은 자체 `JobLauncher`를 가집니다.

그러나 `HttpRequest` 범위 내에 있는 웹 컨테이너 내에서 실행하는 경우, 일반적으로 여러 요청이 해당 Job을 시작하기 위해 호출하는 (비동기 Job 시작을 위해 구성된) 하나의 `JobLauncher`가 있습니다.

**커맨드 라인에서 Job 실행하기**

엔터프라이즈 스케줄러에서 Job을 실행하려면 커맨드 라인이 주요 인터페이스입니다.

이는 대부분의 스케줄러가 (NativeJob을 사용하는 경우가 아니면 Quartz 제외) 주로 셸 스크립트로 시작되는 운영 체제 프로세스와 직접 작동하기 때문입니다.

셸 스크립트 외에도 Perl, Ruby 또는 Ant나 Maven과 같은 빌드 도구 등 Java 프로세스를 시작하는 여러 가지 방법이 있습니다.

그러나 대부분의 사람들이 셸 스크립트에 익숙하기 때문에 이 예제는 셸 스크립트에 중점을 둡니다.

**CommandLineJobRunner**

Job을 시작하는 스크립트는 Java 가상 머신(JVM)을 시작해야 하므로, 기본 진입점 역할을 하는 `main` 메서드가 있는 클래스가 필요합니다.

Spring Batch는 이러한 목적을 수행하는 구현인 `CommandLineJobRunner`를 제공합니다.

이는 애플리케이션을 부트스트랩하는 한 가지 방법일 뿐이라는 점에 유의하십시오.

Java 프로세스를 시작하는 방법은 여러 가지가 있으며, 이 클래스가 최종적인 방법으로 간주되어서는 안 됩니다.

`CommandLineJobRunner`는 다음 네 가지 작업을 수행합니다.

1. 적절한 `ApplicationContext` 로드.
2. 커맨드 라인 인수를 `JobParameters`로 파싱.
3. 인수를 기반으로 적절한 Job 찾기.
4. 애플리케이션 컨텍스트에 제공된 `JobLauncher`를 사용하여 Job 시작.

이 모든 작업은 전달된 인수만으로 수행됩니다. 다음 표는 필수 인수를 설명합니다.

[표 1. CommandLineJobRunner 인수]

| 인수명 | 설명 |
| --- | --- |
| jobPath | `ApplicationContext`를 만드는 데 사용되는 XML 파일의 위치. 이 파일에는 전체 `Job`을 실행하는 데 필요한 모든 것이 포함되어야 합니다. |
| jobName | 실행할 Job의 이름. |

이러한 인수는 경로가 먼저, 이름이 두 번째로 전달되어야 합니다.

이들 이후의 모든 인수는 Job 매개변수로 간주되어 `JobParameters` 객체로 변환되며, `name=value` 형식이어야 합니다.

다음 예는 Java로 정의된 Job에 Job 매개변수로 날짜를 전달하는 것을 보여줍니다.
*(XML 파일 경로 대신 Java 설정 클래스 이름을 사용할 수 있음을 보여줍니다.)*

```bash
<bash$ java CommandLineJobRunner io.spring.EndOfDayJobConfiguration endOfDay schedule.date=2007-05-05,java.time.LocalDate
```

기본적으로 `CommandLineJobRunner`는 키/값 쌍을 암묵적으로 식별용 Job 매개변수로 변환하는 `DefaultJobParametersConverter`를 사용합니다.

그러나 각각 `true` 또는 `false`를 접미사로 붙여 어떤 Job 매개변수가 식별용인지 아닌지를 명시적으로 지정할 수 있습니다.

다음 예에서 `schedule.date`는 식별용 Job 매개변수이고, `vendor.id`는 그렇지 않습니다.

```bash
<bash$ java CommandLineJobRunner endOfDayJob.xml endOfDay \\
                                 schedule.date=2007-05-05,java.time.LocalDate,true \\
                                 vendor.id=123,java.lang.Long,false
<bash$ java CommandLineJobRunner io.spring.EndOfDayJobConfiguration endOfDay \\
                                 schedule.date=2007-05-05,java.time.LocalDate,true \\
                                 vendor.id=123,java.lang.Long,false
```

사용자 정의 `JobParametersConverter`를 사용하여 이 동작을 재정의할 수 있습니다.

대부분의 경우 jar 파일에 메인 클래스를 선언하기 위해 매니페스트를 사용하지만, 단순성을 위해 클래스를 직접 사용했습니다.

이 예는 배치의 도메인 언어 섹션의 `EndOfDay` 예제를 사용합니다.

첫 번째 인수는 `Job`을 포함하는 구성 클래스의 정규화된 클래스 이름인 `io.spring.EndOfDayJobConfiguration`입니다.

두 번째 인수인 `endOfDay`는 Job 이름을 나타냅니다.

마지막 인수인 `schedule.date=2007-05-05,java.time.LocalDate`는 `java.time.LocalDate` 타입의 `JobParameter` 객체로 변환됩니다.

다음 예는 Java에서 `endOfDay`에 대한 샘플 구성을 보여줍니다.

```java
@Configuration
@EnableBatchProcessing
public class EndOfDayJobConfiguration {

    @Bean
    public Job endOfDay(JobRepository jobRepository, Step step1) {
        return new JobBuilder("endOfDay", jobRepository)
    				.start(step1)
    				.build();
    }

    @Bean
    public Step step1(JobRepository jobRepository, PlatformTransactionManager transactionManager) {
        return new StepBuilder("step1", jobRepository)
    				.tasklet((contribution, chunkContext) -> null, transactionManager) // 간단한 Tasklet 예시
    				.build();
    }
}
```

일반적으로 Spring Batch에서 배치 Job을 실행하는 데에는 더 많은 요구 사항이 있으므로 위 예는 지나치게 단순화되었지만, `CommandLineJobRunner`의 두 가지 주요 요구 사항인 `Job`과 `JobLauncher`를 보여주는 역할을 합니다.

**종료 코드 (Exit Codes)**

커맨드 라인에서 배치 Job을 시작할 때 엔터프라이즈 스케줄러가 종종 사용됩니다.

대부분의 스케줄러는 상당히 단순하며 프로세스 수준에서만 작동합니다.

즉, 자신이 호출하는 일부 운영 체제 프로세스(예: 셸 스크립트)에 대해서만 알고 있습니다.

이 시나리오에서 Job의 성공 또는 실패에 대해 스케줄러에 다시 통신하는 유일한 방법은 반환 코드(return codes)를 통하는 것입니다.

반환 코드는 실행 결과를 나타내기 위해 프로세스가 스케줄러에 반환하는 숫자입니다.

가장 간단한 경우 0은 성공이고 1은 실패입니다.

그러나 "Job A가 4를 반환하면 Job B를 시작하고, 5를 반환하면 Job C를 시작하라"와 같이 더 복잡한 시나리오가 있을 수 있습니다.

이러한 유형의 동작은 스케줄러 수준에서 구성되지만, Spring Batch와 같은 처리 프레임워크가 특정 배치 Job에 대한 종료 코드의 숫자 표현을 반환하는 방법을 제공하는 것이 중요합니다.

Spring Batch에서 이는 `ExitStatus` 내에 캡슐화됩니다.

종료 코드에 대해 논의하는 목적상 알아야 할 유일한 중요한 점은 `ExitStatus`에는 프레임워크(또는 개발자)가 설정한 종료 코드 속성이 있으며, 이는 `JobLauncher`에서 반환된 `JobExecution`의 일부로 반환된다는 것입니다. `CommandLineJobRunner`는 `ExitCodeMapper` 인터페이스를 사용하여 이 문자열 값을 숫자로 변환합니다.

```java
public interface ExitCodeMapper {

    public int intValue(String exitCode);

}
```

`ExitCodeMapper`의 핵심 계약은 문자열 종료 코드가 주어지면 숫자 표현이 반환된다는 것입니다.

Job 러너에서 사용되는 기본 구현은 `SimpleJvmExitCodeMapper`이며, 완료 시 0, 일반 오류 시 1, 제공된 컨텍스트에서 Job을 찾을 수 없는 것과 같은 Job 러너 오류 시 2를 반환합니다.

위 세 가지 값보다 더 복잡한 것이 필요한 경우, `ExitCodeMapper` 인터페이스의 사용자 정의 구현을 제공해야 합니다.

`CommandLineJobRunner`는 `ApplicationContext`를 생성하는 클래스이므로 '연결(wired together)'될 수 없으므로, 덮어써야 하는 모든 값은 자동 주입(autowired)되어야 합니다.

즉, `ExitCodeMapper`의 구현이 `BeanFactory` 내에서 발견되면 컨텍스트가 생성된 후 러너에 주입됩니다.

자체 `ExitCodeMapper`를 제공하기 위해 해야 할 일은 구현을 루트 수준 빈으로 선언하고 러너가 로드하는 `ApplicationContext`의 일부인지 확인하는 것뿐입니다.

**웹 컨테이너 내에서 Job 실행하기**

역사적으로 오프라인 처리(예: 배치 Job)는 앞서 설명한 것처럼 커맨드 라인에서 시작되었습니다.

그러나 `HttpRequest`에서 시작하는 것이 더 나은 옵션인 경우가 많습니다.

이러한 사용 사례에는 보고, 임시(ad-hoc) Job 실행 및 웹 애플리케이션 지원 등이 포함됩니다.

배치 Job은 (정의상) 오래 실행되므로 가장 중요한 관심사는 Job을 비동기적으로 시작하는 것입니다.

이 경우 컨트롤러는 Spring MVC 컨트롤러입니다.

컨트롤러는 비동기적으로 시작하도록 구성된 `JobLauncher`를 사용하여 `Job`을 시작하며, 이는 즉시 `JobExecution`을 반환합니다.

`Job`은 여전히 실행 중일 가능성이 높습니다.

그러나 이러한 비차단(nonblocking) 동작을 통해 컨트롤러가 즉시 반환할 수 있으며, 이는 `HttpRequest`를 처리할 때 필요합니다. 다음 목록은 예제를 보여줍니다.

```java
@Controller
public class JobLauncherController {

    @Autowired
    JobLauncher jobLauncher; // 비동기로 설정된 JobLauncher 주입

    @Autowired
    Job job; // 실행할 Job 주입

    @RequestMapping("/jobLauncher.html")
    public void handle() throws Exception{
        jobLauncher.run(job, new JobParameters()); // JobParameters는 필요에 따라 설정
    }
}
```

---

## Spring Batch Job, 어떻게 시동을 걸까? 🚦

Spring Batch Job을 실행하려면 기본적으로 두 가지가 필요해요.

1. **실행할 `Job`:** 우리가 만들고 설정한 배치 작업 그 자체죠.
2. **`JobLauncher`:** 이 `Job`에 "시작!" 명령을 내리는 실행기예요.

이 `Job`과 `JobLauncher`는 같은 설정 파일(ApplicationContext) 안에 있을 수도 있고, 서로 다른 곳에 있을 수도 있어요. 상황에 따라 실행 방식이 달라지거든요.

---

### 1. 전통적인 방식: 커맨드 라인에서 Job 실행하기 💻

가장 기본적인 방법은 터미널이나 명령 프롬프트 같은 **커맨드 라인(Command Line)**에서 Job을 실행하는 거예요. 특히 회사에서 사용하는 **엔터프라이즈 스케줄러** (예: Control-M, Autosys 등)들은 대부분 셸 스크립트(Shell Script)를 통해 운영체제의 프로그램을 실행시키기 때문에, 커맨드 라인 방식이 아주 중요해요.

**`CommandLineJobRunner`: 커맨드 라인의 해결사!**

우리가 Java 프로그램을 커맨드 라인에서 실행하려면, `public static void main(String[] args)` 메서드를 가진 시작 클래스가 필요하죠? Spring Batch는 이런 역할을 해주는 **`CommandLineJobRunner`**라는 편리한 클래스를 제공해요.

`CommandLineJobRunner`가 하는 일은 크게 네 가지예요.

1. **설정 파일 로드:** Job 실행에 필요한 모든 설정(Job, Step, JobLauncher, JobRepository 등)이 담긴 Spring 설정 파일(XML 또는 Java Configuration 클래스)을 읽어들여서 `ApplicationContext`를 만들어요.
2. **파라미터 변환:** 커맨드 라인으로 전달된 인자들을 `JobParameters` 객체로 바꿔줘요.
3. **Job 찾기:** 실행할 `Job`의 이름을 바탕으로 `ApplicationContext`에서 해당 `Job` 빈(Bean)을 찾아요.
4. **Job 실행:** `ApplicationContext` 안에 있는 `JobLauncher`를 사용해서 찾아낸 `Job`을 실행시켜요.

**`CommandLineJobRunner` 사용법:**

커맨드 라인에서 `java CommandLineJobRunner [설정파일경로/클래스명] [Job이름] [파라미터1=값1] [파라미터2=값2] ...` 형식으로 실행해요.

- **첫 번째 인수 (필수):** Job 설정이 담긴 XML 파일의 경로 또는 Java 설정 클래스의 전체 이름 (패키지명 포함).
- **두 번째 인수 (필수):** 실행할 `Job`의 이름 (Spring 설정 파일에 정의된 Job 빈의 ID 또는 이름).
- **그 이후 인수들 (선택):** `JobParameters`로 전달될 값들이에요. `이름=값` 형식으로 전달하고, 콤마(`,`)로 구분해서 데이터 타입이나 식별 여부를 추가로 지정할 수 있어요.
  - `이름=값,타입` (예: `schedule.date=2023-01-01,java.time.LocalDate`)
  - `이름=값,타입,식별여부` (예: `schedule.date=2023-01-01,java.time.LocalDate,true` -> 식별용 파라미터)
    - `true`: 이 파라미터는 `JobInstance`를 구분하는 데 사용됨 (기본값).
    - `false`: 이 파라미터는 `JobInstance` 구분에는 사용되지 않음.

**예시:**

```bash
# Java 설정 클래스(io.spring.EndOfDayJobConfiguration)를 사용하고,
# "endOfDay" Job을 실행하며,
# schedule.date 파라미터를 LocalDate 타입의 2007-05-05로 전달 (식별용),
# vendor.id 파라미터를 Long 타입의 123으로 전달 (식별용 아님)
java org.springframework.batch.core.launch.support.CommandLineJobRunner \\
     io.spring.EndOfDayJobConfiguration \\
     endOfDay \\
     schedule.date=2007-05-05,java.time.LocalDate,true \\
     vendor.id=123,java.lang.Long,false
```

*(실제 실행 시에는 `CommandLineJobRunner`의 전체 패키지 경로를 명시하거나, jar 파일의 Manifest에 Main-Class로 지정해서 더 짧게 실행할 수 있어요.)*

**`CommandLineJobRunner`를 위한 Java 설정 예시:**

```java
@Configuration
@EnableBatchProcessing // JobRepository, JobLauncher 등을 자동으로 설정
public class EndOfDayJobConfiguration {

    @Bean // "endOfDay" 라는 이름의 Job
    public Job endOfDay(JobRepository jobRepository, Step step1) {
        return new JobBuilder("endOfDay", jobRepository)
                    .start(step1)
                    .build();
    }

    @Bean // "step1" 이라는 이름의 Step (간단한 Tasklet 예시)
    public Step step1(JobRepository jobRepository, PlatformTransactionManager transactionManager) {
        return new StepBuilder("step1", jobRepository)
                    .tasklet((contribution, chunkContext) -> {
                        System.out.println("Step 1이 실행되었습니다!");
                        return RepeatStatus.FINISHED; // 한 번 실행하고 종료
                    }, transactionManager)
                    .build();
    }
}
```

이 설정에는 `Job`("endOfDay")과 `@EnableBatchProcessing`을 통해 자동으로 생성될 `JobLauncher`가 포함되어 있어서 `CommandLineJobRunner`가 잘 찾아 쓸 수 있어요.

**핵심:** `CommandLineJobRunner`는 **커맨드 라인에서 Spring Batch Job을 쉽게 실행할 수 있도록 도와주는 도구**예요. 스케줄러 연동 시 매우 유용하답니다!

---

### 2. 스케줄러와의 약속: 종료 코드 (Exit Codes) 📜

엔터프라이즈 스케줄러는 보통 커맨드 라인으로 프로그램을 실행시키고, 그 프로그램이 끝날 때 **"종료 코드(Exit Code)"**라는 숫자를 받아서 작업의 성공/실패 여부를 판단해요.

- 일반적으로 **0은 성공**, 0이 아닌 다른 숫자(주로 1)는 실패를 의미해요.
- 더 복잡하게는, "종료 코드가 4면 B 작업 실행, 5면 C 작업 실행"처럼 스케줄러에서 다음 행동을 결정하는 데 사용되기도 해요.

Spring Batch는 `Job`이 실행된 후 최종 상태를 **`ExitStatus`**라는 객체에 담아서 `JobExecution`의 일부로 반환해요. 이 `ExitStatus` 안에는 `exitCode`라는 문자열 값이 들어있죠.

`CommandLineJobRunner`는 이 문자열 `exitCode`를 스케줄러가 이해할 수 있는 **숫자 값으로 변환**해야 하는데, 이때 **`ExitCodeMapper`** 인터페이스를 사용해요.

```java
public interface ExitCodeMapper {
    int intValue(String exitCode); // 문자열 exitCode를 받아서 숫자 int로 변환
}
```

- **기본 매퍼:** `SimpleJvmExitCodeMapper`가 기본으로 사용돼요.
  - `ExitStatus.COMPLETED.getExitCode()` (기본적으로 "COMPLETED") -> **0**
  - 일반적인 오류 (예: `ExitStatus.FAILED.getExitCode()` - "FAILED") -> **1**
  - `CommandLineJobRunner` 자체 오류 (예: Job을 못 찾음) -> **2**
- **커스텀 매퍼:** 만약 "WARN" 상태일 때는 3, "SKIPPED" 상태일 때는 4처럼 더 복잡한 종료 코드가 필요하다면, `ExitCodeMapper` 인터페이스를 구현한 클래스를 만들고 Spring 빈으로 등록해주면 돼요. `CommandLineJobRunner`가 알아서 그 빈을 찾아서 사용한답니다.

**핵심:** `ExitCodeMapper`를 통해 Spring Batch Job의 최종 상태를 **스케줄러가 이해할 수 있는 숫자 종료 코드로 변환**하여 전달할 수 있어요.

---

### 3. 요즘 대세: 웹에서 Job 실행하기 (비동기는 필수!) 🌐💨

예전에는 배치 Job은 무조건 밤에 커맨드 라인으로만 돌렸지만, 요즘에는 웹 화면에서 사용자가 버튼을 눌러 특정 Job을 실행시키는 경우도 많아요. (예: 특정 조건으로 데이터 추출하는 리포팅 Job, 사용자 요청에 따른 임시 데이터 처리 Job 등)

**웹에서 Job 실행 시 가장 중요한 것: 비동기 실행!**

- 배치 Job은 오래 걸리는 작업이라고 했죠?
- 만약 웹 요청을 처리하는 스레드가 Job이 끝날 때까지 동기적으로 기다린다면, 그동안 웹 브라우저는 계속 "로딩 중..." 상태로 멈춰있을 거예요. 이건 사용자에게 매우 안 좋은 경험을 줍니다.
- 따라서 웹에서 Job을 실행할 때는 **반드시 `JobLauncher`를 비동기로 설정**해서, Job 실행을 요청하면 `JobLauncher`는 **즉시 응답을 반환**하고, 실제 Job은 백그라운드에서 돌아가도록 해야 해요. (앞에서 `JobLauncher` 설정 때 배운 내용이죠!)

**웹 컨트롤러 예시 (Spring MVC):**

```java
@Controller
public class JobLauncherController {

    @Autowired
    JobLauncher jobLauncher; // 비동기로 설정된 JobLauncher를 주입받아야 함!

    @Autowired
    Job myJob; // 실행시킬 Job도 주입

    @RequestMapping("/launchMyJob") // 이 URL로 요청이 오면
    public String handleJobLaunch(Model model) throws Exception {
        try {
            // JobParameters는 필요에 따라 생성 (예: 현재 시간, 사용자 입력값 등)
            JobParameters jobParameters = new JobParametersBuilder()
                                                .addDate("run.date", new Date())
                                                .toJobParameters();
            JobExecution jobExecution = jobLauncher.run(myJob, jobParameters); // 비동기 실행! 바로 JobExecution 반환

            model.addAttribute("message", "Job이 성공적으로 시작되었습니다! (ID: " + jobExecution.getId() + ")");
            model.addAttribute("jobId", jobExecution.getJobId()); // JobExecution에서 Job ID를 가져올 수 있음
        } catch (Exception e) {
            model.addAttribute("message", "Job 시작에 실패했습니다: " + e.getMessage());
        }
        return "jobStatusPage"; // Job 상태를 보여줄 페이지로 이동 (또는 JSON 응답)
    }
}
```

이 컨트롤러는 `/launchMyJob` 요청을 받으면, (비동기로 설정된) `jobLauncher`를 사용해 `myJob`을 실행시키고, Job이 실제로 끝났는지와 상관없이 사용자에게 "Job이 시작되었습니다"라는 메시지를 즉시 보여줄 수 있어요.

실제 Job의 진행 상태는 나중에 별도의 화면에서 확인하도록 만들 수 있겠죠.

**핵심:** 웹 환경에서 Job을 실행할 때는 **비동기 `JobLauncher`를 사용**하여 사용자에게 빠른 응답을 제공하는 것이 매우 중요해요!

---
