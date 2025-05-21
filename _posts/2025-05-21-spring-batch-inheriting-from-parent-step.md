---
title: Spring Batch - Configuring a Step (Inheriting from a Parent Step)
description: 
author: laze
date: 2025-05-21 00:00:02 +0900
categories: [Dev, SpringBatch]
tags: [SpringBatch]
---
**부모 Step으로부터 상속 (Inheriting from a Parent Step)**

여러 `Step` 그룹이 유사한 구성을 공유하는 경우, 구체적인 `Step`들이 속성을 상속받을 수 있는 "부모" `Step`을 정의하는 것이 도움이 될 수 있습니다.

Java의 클래스 상속과 유사하게, "자식" `Step`은 부모의 요소 및 속성과 자신의 요소 및 속성을 결합합니다.

또한 자식은 부모의 모든 `Step` 속성을 재정의(override)합니다.

다음 예에서 `Step`(concreteStep1)은 `parentStep`으로부터 상속받습니다.

이 `Step`은 `itemReader`, `itemProcessor`, `itemWriter`, `startLimit=5`, 그리고 `allowStartIfComplete=true`로 인스턴스화됩니다.

추가적으로, `commitInterval`은 `concreteStep1` `Step`에 의해 재정의되므로 5가 됩니다.

```xml
<step id="parentStep">
    <tasklet allow-start-if-complete="true"> <!-- 부모 속성: 작업 완료 후에도 시작 허용 -->
        <chunk reader="itemReader" writer="itemWriter" commit-interval="10"/> <!-- 부모 청크 설정 -->
    </tasklet>
</step>

<step id="concreteStep1" parent="parentStep"> <!-- parentStep으로부터 상속 -->
    <tasklet start-limit="5"> <!-- 자식 속성: 최대 시작 횟수 5 -->
        <!-- itemReader와 itemWriter는 부모로부터 상속 -->
        <!-- allow-start-if-complete도 부모로부터 상속 -->
        <chunk processor="itemProcessor" commit-interval="5"/> <!-- 자식 청크 설정 (processor 추가, commit-interval 재정의) -->
    </tasklet>
</step>

```

`job` 요소 내의 `step`에는 여전히 `id` 속성이 필요합니다. 여기에는 두 가지 이유가 있습니다.

1. `StepExecution`을 영속화할 때 `id`가 스텝 이름으로 사용됩니다. 동일한 독립형 스텝(standalone step)이 Job 내 여러 스텝에서 참조되면 오류가 발생합니다.
2. 이 장의 뒷부분에서 설명하는 것처럼 Job 흐름을 만들 때, `next` 속성은 독립형 스텝이 아닌 흐름 내의 스텝을 참조해야 합니다.

**추상 Step (Abstract Step)**

때로는 완전한 `Step` 구성이 아닌 부모 `Step`을 정의해야 할 수도 있습니다.

예를 들어, `Step` 구성에서 `reader`, `writer`, `tasklet` 속성이 누락되면 초기화에 실패합니다.

이러한 속성 중 하나 이상 없이 부모를 정의해야 하는 경우, `abstract` 속성을 사용해야 합니다.

추상 `Step`은 오직 확장(상속)될 뿐, 직접 인스턴스화되지 않습니다.

다음 예에서 `Step`(abstractParentStep)은 `abstract`로 선언되지 않았다면 인스턴스화되지 않았을 것입니다.

`Step`(concreteStep2)은 `itemReader`, `itemWriter`, 그리고 `commit-interval=10`을 가집니다.

```xml
<step id="abstractParentStep" abstract="true"> <!-- 추상 스텝으로 선언 (직접 사용 불가) -->
    <tasklet>
        <chunk commit-interval="10"/> <!-- commit-interval만 정의 -->
    </tasklet>
</step>

<step id="concreteStep2" parent="abstractParentStep"> <!-- abstractParentStep으로부터 상속 -->
    <tasklet>
        <!-- commit-interval은 부모로부터 상속 (10) -->
        <chunk reader="itemReader" writer="itemWriter"/> <!-- reader와 writer는 자식에서 정의 -->
    </tasklet>
</step>

```

**리스트 병합 (Merging Lists)**

`Step`에서 구성 가능한 요소 중 일부는 `<listeners/>` 요소와 같이 리스트 형태입니다.

부모와 자식 `Step` 모두 `<listeners/>` 요소를 선언하면, 자식의 리스트가 부모의 리스트를 재정의합니다.

자식이 부모가 정의한 리스트에 추가적인 리스너를 추가할 수 있도록 하려면, 모든 리스트 요소에 `merge` 속성이 있습니다.

요소가 `merge="true"`를 지정하면, 자식의 리스트는 부모의 리스트를 재정의하는 대신 부모의 리스트와 결합됩니다.

다음 예에서 `Step`("concreteStep3")은 `listenerOne`과 `listenerTwo`라는 두 개의 리스너로 생성됩니다.

```xml
<step id="listenersParentStep" abstract="true">
    <listeners>
        <listener ref="listenerOne"/> <!-- 부모 리스너 -->
    </listeners>
</step>

<step id="concreteStep3" parent="listenersParentStep"> <!-- listenersParentStep으로부터 상속 -->
    <tasklet>
        <chunk reader="itemReader" writer="itemWriter" commit-interval="5"/>
    </tasklet>
    <listeners merge="true"> <!-- 부모 리스너와 자식 리스너를 병합 -->
        <listener ref="listenerTwo"/> <!-- 자식 리스너 -->
    </listeners>
</step>
```

---

## Step 설정도 재활용! - 부모 Step에게 물려받기 (XML 설정) 붕어빵 Step 만들기! 🐟🍞

우리가 `Job`을 만들 때 여러 개의 `Step`을 사용하죠? 그런데 가끔씩 이 `Step`들이 서로 아주 비슷하게 생겼을 때가 있어요. 예를 들어,

- 데이터를 읽어오는 `ItemReader`는 똑같은데, 처리하는 `ItemProcessor`만 다르거나,
- 대부분의 설정은 같은데, `commit-interval` 값만 다른 `Step`들이 여러 개 필요할 수 있죠.

이럴 때마다 매번 모든 설정을 복사해서 붙여넣기 하는 건 너무 번거롭고, 나중에 하나를 수정하려면 모든 `Step`을 다 고쳐야 하는 불편함이 있어요. 😥

이런 문제를 해결하기 위해 Spring Batch XML 설정에서는 **"부모 Step"**을 만들어두고, **"자식 Step"**들이 그 설정을 **상속**받아서 사용할 수 있는 기능을 제공해요!

---

### 1. 기본 상속: 부모님 설정 그대로! 👨‍👩‍👧

자식 `Step`은 `parent="부모Step_ID"` 속성을 사용해서 부모 `Step`을 지정할 수 있어요.

- **자식 `Step`은 부모 `Step`의 모든 설정을 물려받아요.**
- 만약 자식 `Step`에서 부모와 동일한 속성을 다시 정의하면, **자식의 설정이 부모의 설정을 덮어써요(override).** (마치 "우리 집 규칙은 이거지만, 내 방에서는 내 규칙을 따를 거야!" 하는 것과 같아요.)

**예시:**

```xml
<!-- 🎅 부모 Step: 기본 설정을 가진 붕어빵 틀 -->
<step id="parentStep">
    <tasklet allow-start-if-complete="true"> <!-- 속성1: 작업 완료 후에도 재시작 허용 -->
        <chunk reader="commonReader"   writer="commonWriter"   commit-interval="10"/> <!-- 속성2,3,4 -->
    </tasklet>
</step>

<!-- 🧑 자식 Step: 팥 붕어빵 (부모 틀을 사용 + 약간의 변화) -->
<step id="concreteStep1" parent="parentStep"> <!-- "parentStep"을 부모로 지정! -->
    <tasklet start-limit="5">                 <!-- 속성5: 최대 시작 횟수 5 (자식만 가짐) -->
        <!-- commonReader, commonWriter, allow-start-if-complete는 부모로부터 상속! -->
        <chunk processor="팥_ItemProcessor"  commit-interval="5"/> <!-- 속성6: processor 추가, 속성4: commit-interval은 10 대신 5로 덮어쓰기! -->
    </tasklet>
</step>
```

위 예시에서 `concreteStep1`은 다음과 같은 최종 설정을 갖게 돼요:

- `reader`: `commonReader` (부모로부터 상속)
- `writer`: `commonWriter` (부모로부터 상속)
- `processor`: `팥_ItemProcessor` (자식에서 새로 정의)
- `commit-interval`: `5` (자식이 부모의 `10`을 덮어씀)
- `allow-start-if-complete`: `true` (부모로부터 상속)
- `start-limit`: `5` (자식에서 새로 정의)

**`Job` 안에서 `Step`을 사용할 때 `id`는 왜 필요할까요?**

`Job` 설정 안에서 실제로 `Step`을 배치할 때는 (즉, `<job><step id="여기ID" .../></job>` 처럼 사용할 때) 부모-자식 관계와 별개로 고유한 `id`가 필요해요.

1. **`StepExecution` 저장 시 이름으로 사용:** `JobRepository`에 `Step` 실행 정보를 저장할 때, 이 `id`가 `Step`의 이름으로 사용돼요. 만약 `Job` 내에서 여러 `Step`이 동일한 `id`를 사용하면 어떤 `Step`의 실행 정보인지 구분할 수 없어서 오류가 나요. (독립적으로 정의된 부모 `Step`의 `id`와는 다른 개념이에요.)
2. **`Step` 흐름 제어 시 참조 대상:** `Job` 내에서 "A 스텝 다음에 B 스텝 실행해" (`<step id="A" next="B">`) 와 같이 흐름을 제어할 때, `next` 속성은 `Job` 내에 정의된 `Step`의 `id`를 가리켜야 해요.

---

### 2. 추상 Step: "나를 직접 쓰지 말고, 상속해서 써!" (템플릿 전용) 📜

때로는 부모 `Step`을 만들 때, 그 자체로는 완전한 `Step`이 아니도록 하고 싶을 때가 있어요. 예를 들어, `ItemReader`나 `ItemWriter` 없이 `commit-interval` 같은 공통 속성만 정의해두고, 자식 `Step`에서 `Reader`와 `Writer`를 채워 넣도록 하는 거죠.

이런 경우, 부모 `Step`에 `abstract="true"` 속성을 붙여줘요.

- **추상 `Step`은 직접 실행(인스턴스화)될 수 없어요.** 오직 다른 `Step`에게 상속되기 위한 **템플릿 역할**만 해요.
- 만약 `reader`, `writer`, `tasklet` 같은 필수 요소 없이 `Step`을 정의하고 `abstract="true"`를 안 붙이면, Spring Batch가 `Step`을 만들려고 할 때 "어? 필수 부품이 없잖아!" 하고 오류를 내요.

**예시:**

```xml
<!-- 🎨 추상 부모 Step: commit-interval만 있는 미완성 붕어빵 틀 -->
<step id="abstractParentStep" abstract="true"> <!-- "나는 템플릿이야, 직접 만들지 마!" -->
    <tasklet>
        <chunk commit-interval="10"/> <!-- 공통 속성: commit-interval은 10 -->
    </tasklet>
</step>

<!-- 🧑 자식 Step: 슈크림 붕어빵 (미완성 틀에 내용물 채우기) -->
<step id="concreteStep2" parent="abstractParentStep"> <!-- "abstractParentStep" 템플릿 사용! -->
    <tasklet>
        <!-- commit-interval은 부모로부터 상속 (10) -->
        <chunk reader="슈크림_ItemReader" writer="슈크림_ItemWriter"/> <!-- Reader와 Writer는 자식에서 정의! -->
    </tasklet>
</step>
```

위 예시에서 `concreteStep2`는 `commit-interval`은 `10`으로 (부모로부터), `reader`와 `writer`는 자신이 정의한 것으로 설정돼요. `abstractParentStep` 자체는 `Job`에서 직접 사용할 수 없어요.

---

### 3. 리스트 설정 합치기: 부모님 것도 내 것도 다 좋아! (Merging Lists) ➕

`Step` 설정 중에는 리스너(`listeners`)처럼 여러 개의 값을 가질 수 있는 리스트 형태의 설정들이 있어요.

- 기본적으로 자식 `Step`에서 리스트 형태의 설정을 다시 정의하면, **부모의 리스트는 무시되고 자식의 리스트만 적용돼요.** (덮어쓰기)
- 하지만 "부모 `Step`의 리스너 목록도 그대로 쓰고, 거기에 내가 정의한 리스너도 추가하고 싶어!" 할 때가 있겠죠?
- 이럴 때는 해당 리스트 설정 태그(예: `<listeners>`)에 `merge="true"` 속성을 추가해주면 돼요.

**예시:**

```xml
<!-- 👂 추상 부모 Step: 기본 리스너가 있는 템플릿 -->
<step id="listenersParentStep" abstract="true">
    <listeners>
        <listener ref="listenerOne"/> <!-- 부모 리스너 1 -->
    </listeners>
</step>

<!-- 🧑 자식 Step: 부모 리스너 + 내 리스너 모두 사용! -->
<step id="concreteStep3" parent="listenersParentStep">
    <tasklet>
        <chunk reader="itemReader" writer="itemWriter" commit-interval="5"/>
    </tasklet>
    <listeners merge="true"> <!-- "부모 리스너 목록이랑 내 리스너 목록 합쳐줘!" -->
        <listener ref="listenerTwo"/> <!-- 자식 리스너 2 -->
    </listeners>
</step>
```

위 예시에서 `concreteStep3`은 `listenerOne` (부모로부터)과 `listenerTwo` (자신이 정의) **두 개의 리스너를 모두** 가지게 돼요. 만약 `merge="true"`가 없었다면 `listenerTwo`만 적용되었을 거예요.

---

**부모 Step 상속, 왜 사용할까요?**

- **코드 중복 감소:** 비슷한 `Step` 설정을 여러 번 반복해서 작성할 필요가 없어져요.
- **유지보수 용이:** 공통 설정을 변경해야 할 때, 부모 `Step` 하나만 수정하면 모든 자식 `Step`에 자동으로 반영돼요.
- **설정의 표준화:** 자주 사용되는 `Step` 설정 패턴을 부모 `Step`으로 만들어두면, 일관성 있는 `Step`을 쉽게 만들 수 있어요.

**핵심:** (XML 설정에서) 부모 `Step` 상속 기능을 잘 활용하면, **`Step` 설정을 더 깔끔하고 효율적으로 관리**할 수 있어요. 마치 객체지향 프로그래밍의 상속처럼요!

---

## 리스너(Listener): 배치 작업의 특별한 순간을 포착하라! 🎣🔔

배치 작업은 여러 단계를 거쳐 실행되죠? 예를 들면,

- `Job`이 시작되기 직전/직후
- `Step`이 시작되기 직전/직후
- 청크(Chunk) 단위 처리가 시작되기 직전/직후/오류 발생 시
- 아이템(Item) 하나하나를 읽거나(Read), 처리하거나(Process), 쓸 때(Write) 오류가 발생했을 때

이런 **"특별한 순간들"**에 "나 지금 이 일 좀 하고 싶은데!" 하고 우리가 만든 코드를 실행시키고 싶을 때가 있어요. 바로 이럴 때 사용하는 것이 **리스너**입니다!

**리스너는 마치...**

- **스포츠 경기의 심판:** 중요한 순간에 개입해서 규칙을 적용하거나 상황을 알리죠.
- **이벤트 알리미:** "띵동! Job이 시작되었습니다!" 또는 "삐빅! Step 처리 중 오류 발생!" 처럼 특정 상황을 알려주고, 그에 맞는 행동을 할 수 있게 해줘요.
- **감시 카메라:** 작업이 잘 돌아가는지 지켜보다가 특정 상황이 되면 기록을 남기거나 알람을 울리죠.

---

### Spring Batch에는 어떤 종류의 리스너들이 있을까요?

Spring Batch는 다양한 시점에 개입할 수 있도록 여러 종류의 리스너 인터페이스를 제공해요. 대표적인 것들만 몇 가지 살펴볼게요.

1. **`JobExecutionListener`:** (앞에서 `Job` 설정할 때 잠깐 봤었죠?)
  - **`beforeJob(JobExecution jobExecution)`:** `Job`이 시작되기 직전에 호출돼요.
    - 예시: 작업 시작 로그 남기기, 필요한 외부 리소스 초기화하기.
  - **`afterJob(JobExecution jobExecution)`:** `Job`이 (성공이든 실패든) 완료된 직후에 호출돼요.
    - 예시: 작업 완료 로그 남기기, 결과 요약해서 알림 보내기, 사용한 리소스 정리하기.
2. **`StepExecutionListener`:**
  - **`beforeStep(StepExecution stepExecution)`:** 각 `Step`이 시작되기 직전에 호출돼요.
    - 예시: 해당 `Step`에서 사용할 데이터 유효성 검사, `ExecutionContext`에 초기값 설정.
  - **`afterStep(StepExecution stepExecution)`:** 각 `Step`이 (성공이든 실패든) 완료된 직후에 호출돼요.
    - 예시: `Step` 처리 결과 (읽은 건수, 쓴 건수 등) 로그 남기기, `ExecutionContext`에서 값 가져와서 다음 `Step`에 전달할 준비. `ExitStatus`를 커스터마이징해서 `Step` 흐름 제어.
3. **`ChunkListener`:** (청크 지향 `Step`에서 사용)
  - **`beforeChunk(ChunkContext chunkContext)`:** 청크 처리가 시작되기 직전에 (즉, `ItemReader`가 아이템들을 읽어오기 시작하기 전, 또는 트랜잭션 시작 직후) 호출돼요.
  - **`afterChunk(ChunkContext chunkContext)`:** 청크 처리가 성공적으로 완료된 직후에 (즉, `ItemWriter`가 쓰기를 마치고 트랜잭션이 커밋된 직후) 호출돼요.
  - **`afterChunkError(ChunkContext chunkContext)`:** 청크 처리 중 오류가 발생해서 롤백된 직후에 호출돼요.
    - 예시: 청크 처리 시작/종료 로그, 특정 청크에서 오류 발생 시 알림.
4. **`ItemReadListener`:** (청크 지향 `Step`에서 `ItemReader`와 함께 사용)
  - **`beforeRead()`:** `ItemReader`가 아이템을 읽기 직전에 호출돼요.
  - **`afterRead(T item)`:** `ItemReader`가 아이템을 성공적으로 읽은 직후에 호출돼요 (읽은 아이템을 파라미터로 받음).
  - **`onReadError(Exception ex)`:** `ItemReader`가 아이템을 읽다가 오류가 발생했을 때 호출돼요.
    - 예시: 특정 아이템 읽기 시도 로그, 읽기 오류 발생 시 대체 값 제공 또는 해당 오류 기록.
5. **`ItemProcessListener`:** (청크 지향 `Step`에서 `ItemProcessor`와 함께 사용)
  - **`beforeProcess(S item)`:** `ItemProcessor`가 아이템을 처리하기 직전에 호출돼요 (처리 전 아이템을 파라미터로 받음).
  - **`afterProcess(S item, T result)`:** `ItemProcessor`가 아이템을 성공적으로 처리한 직후에 호출돼요 (처리 전 아이템과 처리 후 결과 아이템을 파라미터로 받음).
  - **`onProcessError(S item, Exception ex)`:** `ItemProcessor`가 아이템을 처리하다가 오류가 발생했을 때 호출돼요.
    - 예시: 특정 아이템 처리 시간 측정, 처리 오류 발생 시 해당 아이템 정보와 오류 내용 기록.
6. **`ItemWriteListener`:** (청크 지향 `Step`에서 `ItemWriter`와 함께 사용)
  - **`beforeWrite(Chunk<? extends T> items)`:** `ItemWriter`가 아이템 청크를 쓰기 직전에 호출돼요 (쓸 아이템 청크를 파라미터로 받음).
  - **`afterWrite(Chunk<? extends T> items)`:** `ItemWriter`가 아이템 청크를 성공적으로 쓴 직후에 호출돼요.
  - **`onWriteError(Exception ex, Chunk<? extends T> items)`:** `ItemWriter`가 아이템 청크를 쓰다가 오류가 발생했을 때 호출돼요.
    - 예시: 쓰기 작업 전 데이터 검증, 쓰기 오류 발생 시 해당 청크 정보와 오류 내용 기록.
7. **`SkipListener`:** (Skip 기능과 함께 사용)
  - 아이템 처리 중 오류가 발생하여 해당 아이템을 건너뛸(skip) 때, 각 단계(read, process, write)에서 skip이 발생했음을 알려줘요.
  - **`onSkipInRead(Throwable t)`**, **`onSkipInProcess(S item, Throwable t)`**, **`onSkipInWrite(T item, Throwable t)`**
    - 예시: 어떤 아이템이 왜 skip 되었는지 상세 로그 기록, skip된 아이템들을 별도로 모아두기.

---

### 리스너는 어떻게 등록하고 사용할까요?

1. **리스너 인터페이스 구현:** 위에서 언급된 리스너 인터페이스 중 필요한 것을 선택하여 클래스로 구현해요.

    ```java
    // 예시: 간단한 JobExecutionListener 구현
    public class MyJobListener implements JobExecutionListener {
        @Override
        public void beforeJob(JobExecution jobExecution) {
            System.out.println(jobExecution.getJobInstance().getJobName() + " is starting!");
        }
    
        @Override
        public void afterJob(JobExecution jobExecution) {
            System.out.println(jobExecution.getJobInstance().getJobName() + " finished with status: " + jobExecution.getStatus());
        }
    }
    
    ```

2. **빈(Bean)으로 등록:** 구현한 리스너 클래스를 Spring의 빈으로 등록해요.

    ```java
    @Bean
    public MyJobListener myJobListener() {
        return new MyJobListener();
    }
    
    ```

3. **`Job` 또는 `Step`에 등록:**
  - **Java 설정:** `JobBuilder`나 `StepBuilder`의 `.listener()` 메서드를 사용해요.

      ```java
      @Bean
      public Job sampleJob(JobRepository jobRepository, Step sampleStep, MyJobListener myJobListener) {
          return new JobBuilder("sampleJob", jobRepository)
                  .listener(myJobListener) // Job에 리스너 등록
                  .start(sampleStep)
                  .build();
      }
      
      @Bean
      public Step sampleStep(JobRepository jobRepository, PlatformTransactionManager transactionManager,
                             ItemReader<String> reader, ItemWriter<String> writer, MyStepListener myStepListener) {
          return new StepBuilder("sampleStep", jobRepository)
                  .<String, String>chunk(10, transactionManager)
                  .reader(reader)
                  .writer(writer)
                  .listener(myStepListener) // Step에 리스너 등록 (StepExecutionListener, ChunkListener 등)
                  .build();
      }
      
      ```

  - **XML 설정:** `<listeners><listener ref="myJobListenerBean"/></listeners>` 와 같이 `<listeners>` 태그를 사용해요.

**어노테이션 기반 리스너:**
몇몇 리스너는 인터페이스를 직접 구현하는 대신, 일반 클래스의 메서드에 `@BeforeJob`, `@AfterStep`, `@OnReadError` 같은 어노테이션을 붙여서 더 간편하게 리스너를 만들 수도 있어요.

---

### 리스너, 왜 사용할까요? (활용 사례)

- **로깅 및 모니터링:** 작업 진행 상황, 오류 발생 지점, 처리 건수 등을 상세하게 로그로 남겨서 문제 추적 및 성능 분석에 활용해요.
- **알림:** 작업 성공/실패 시 관리자에게 이메일이나 슬랙 메시지로 알림을 보내요.
- **자원 관리:** 작업 시작 전에 필요한 파일을 열거나 DB 연결을 준비하고, 작업 완료 후에는 자원을 안전하게 닫아요.
- **통계 정보 수집:** 특정 조건에 맞는 데이터가 몇 건 처리되었는지, 오류는 총 몇 건 발생했는지 등의 통계 정보를 `ExecutionContext`에 저장하거나 외부 시스템에 전송해요.
- **오류 처리 및 복구 로직:** 특정 오류 발생 시 단순 로그 기록을 넘어, 데이터를 백업하거나, 대체 값을 사용하거나, 관리자에게 개입을 요청하는 등의 복잡한 오류 처리 로직을 수행할 수 있어요.
- **실행 흐름 커스터마이징:** `StepExecutionListener`의 `afterStep`에서 `ExitStatus`를 조작하여 다음 `Step`으로의 흐름을 동적으로 변경할 수도 있어요.

**핵심:** 리스너는 Spring Batch의 기본 흐름을 변경하지 않으면서, **부가적인 관심사(횡단 관심사, cross-cutting concerns)**를 깔끔하게 분리하여 처리할 수 있게 해주는 강력한 AOP(관점 지향 프로그래밍) 스타일의 기능입니다!

---

## Java 코드로 Step 설정 재사용하기: 스마트하게 Step 만들기! ☕🛠️

XML의 `parent`와 `abstract` `Step` 개념을 Java 코드로 어떻게 구현할 수 있을까요? 핵심은 **공통된 부분을 분리하고, 필요한 곳에서 가져다 쓰는 것**입니다.

### 1. 공통 설정 메서드 또는 `@Bean` 메서드 활용

가장 기본적인 방법은 여러 `Step`에서 공통적으로 사용될 설정 로직이나 컴포넌트(Reader, Writer, Listener 등)를 별도의 `@Bean` 메서드로 정의하고, 각 `Step`을 정의할 때 이 빈들을 주입받아 사용하는 것입니다.

**예시:** `ItemReader`와 `ItemWriter`는 공통으로 사용하고, `ItemProcessor`와 `commit-interval`만 다른 두 개의 `Step`을 만든다고 가정해 봅시다.

```java
@Configuration
public class CommonStepComponents { // 공통 컴포넌트를 정의하는 설정 클래스

    @Bean
    public ItemReader<String> commonItemReader() {
        // 예시: 간단한 리스트 기반 ItemReader
        return new ListItemReader<>(Arrays.asList("item1", "item2", "item3", "item4", "item5"));
    }

    @Bean
    public ItemWriter<String> commonItemWriter() {
        // 예시: 콘솔에 아이템을 출력하는 ItemWriter
        return chunk -> {
            System.out.println("Writing chunk: " + chunk.getItems());
        };
    }

    @Bean
    public StepExecutionListener commonStepListener() {
        return new StepExecutionListener() {
            @Override
            public void beforeStep(StepExecution stepExecution) {
                System.out.println("Common listener: Before step " + stepExecution.getStepName());
            }

            @Override
            public ExitStatus afterStep(StepExecution stepExecution) {
                System.out.println("Common listener: After step " + stepExecution.getStepName());
                return null;
            }
        };
    }
}

@Configuration
@Import(CommonStepComponents.class) // 공통 컴포넌트 설정 클래스를 가져옴
public class MyStepsConfig {

    @Autowired // 공통 ItemReader 주입
    private ItemReader<String> commonReader;

    @Autowired // 공통 ItemWriter 주입
    private ItemWriter<String> commonWriter;

    @Autowired // 공통 StepExecutionListener 주입
    private StepExecutionListener commonListener;

    // Step 1: Processor A와 commit-interval 5 사용
    @Bean
    public Step concreteStep1(JobRepository jobRepository, PlatformTransactionManager transactionManager,
                              ItemProcessor<String, String> processorA) {
        return new StepBuilder("concreteStep1", jobRepository)
                .<String, String>chunk(5, transactionManager) // commit-interval 5
                .reader(commonReader)       // 공통 Reader 사용
                .processor(processorA)      // 이 Step만의 Processor 사용
                .writer(commonWriter)       // 공통 Writer 사용
                .listener(commonListener)   // 공통 Listener 사용
                .build();
    }

    @Bean
    public ItemProcessor<String, String> processorA() {
        return item -> "ProcessedA-" + item.toUpperCase();
    }

    // Step 2: Processor B와 commit-interval 10 사용
    @Bean
    public Step concreteStep2(JobRepository jobRepository, PlatformTransactionManager transactionManager,
                              ItemProcessor<String, String> processorB) {
        return new StepBuilder("concreteStep2", jobRepository)
                .<String, String>chunk(10, transactionManager) // commit-interval 10
                .reader(commonReader)        // 공통 Reader 사용
                .processor(processorB)       // 이 Step만의 Processor 사용
                .writer(commonWriter)        // 공통 Writer 사용
                .listener(commonListener)    // 공통 Listener 사용
                .build();
    }

    @Bean
    public ItemProcessor<String, String> processorB() {
        return item -> "ProcessedB-" + item.toLowerCase();
    }

    // 만약 리스너를 병합하고 싶다면?
    // Step 3: 공통 리스너 + 이 Step만의 리스너
    @Bean
    public Step concreteStep3(JobRepository jobRepository, PlatformTransactionManager transactionManager,
                              StepExecutionListener specificListenerForStep3) {
        return new StepBuilder("concreteStep3", jobRepository)
                .<String, String>chunk(3, transactionManager)
                .reader(commonReader)
                .writer(commonWriter)
                .listener(commonListener)         // 공통 리스너 등록
                .listener(specificListenerForStep3) // 이 Step만의 리스너 추가 등록
                .build();
    }

    @Bean
    public StepExecutionListener specificListenerForStep3() {
        return new StepExecutionListener() {
            @Override
            public void beforeStep(StepExecution stepExecution) {
                System.out.println("Specific listener for Step3: Before step " + stepExecution.getStepName());
            }
            // afterStep 구현 생략
        };
    }
}
```

**설명:**

- `CommonStepComponents` 클래스에 여러 `Step`에서 공통으로 사용할 `commonItemReader`, `commonItemWriter`, `commonStepListener`를 `@Bean`으로 정의했습니다.
- `MyStepsConfig` 클래스에서 `@Import(CommonStepComponents.class)`를 통해 공통 컴포넌트들을 가져오고, `@Autowired`로 주입받아 사용합니다.
- `concreteStep1`과 `concreteStep2`는 각각 다른 `ItemProcessor`와 `commit-interval`을 가지지만, `reader`, `writer`, `listener`는 공통된 것을 재사용합니다.
- `concreteStep3`에서는 XML의 `merge="true"`와 유사하게, 여러 개의 `.listener()`를 호출하여 여러 리스너를 등록할 수 있습니다. Spring Batch는 등록된 모든 리스너를 순서대로 호출해줍니다.

---

### 2. 추상 클래스 또는 팩토리 메서드를 사용한 "템플릿 Step" 구현

XML의 `abstract="true"`와 유사한 효과를 내기 위해, `Step`을 생성하는 팩토리 메서드나 추상 클래스를 활용할 수 있습니다.

**예시: `Step` 생성을 위한 팩토리 클래스**

```java
@Component // 이 클래스 자체를 빈으로 등록하여 다른 곳에서 주입받아 사용할 수 있도록 함
public class StepTemplateFactory {

    @Autowired
    private JobRepository jobRepository;

    @Autowired
    private PlatformTransactionManager transactionManager;

    // 공통 설정을 일부 포함하는 기본 Step 빌더를 반환하는 메서드
    // 이 메서드 자체가 "추상 Step"의 역할을 한다고 볼 수 있음
    protected <I, O> SimpleStepBuilder<I, O> createBaseChunkStepBuilder(String stepName, int commitInterval) {
        return new StepBuilder(stepName, jobRepository)
                .<I, O>chunk(commitInterval, transactionManager);
    }

    // 구체적인 Step을 생성하는 메서드들
    public Step createCustomStep(String stepName, int commitInterval,
                                 ItemReader<String> reader, ItemProcessor<String, String> processor, ItemWriter<String> writer) {
        return this.createBaseChunkStepBuilder(stepName, commitInterval)
                .reader(reader)
                .processor(processor)
                .writer(writer)
                .build();
    }

    public Step createAnotherCustomStep(String stepName, int commitInterval,
                                        ItemReader<Integer> reader, ItemWriter<Integer> writer) {
        return this.createBaseChunkStepBuilder(stepName, commitInterval)
                .reader(reader)
                .writer(writer) // Processor 없이 바로 Writer로
                .build();
    }
}

// 실제 Job 설정 클래스에서 사용
@Configuration
public class AnotherJobConfig {

    @Autowired
    private StepTemplateFactory stepFactory; // 위에서 만든 팩토리 주입

    @Autowired // 각 Step에 필요한 Reader, Processor, Writer 등은 별도로 빈으로 정의하거나 직접 생성
    private ItemReader<String> myReaderForStepA;
    @Autowired
    private ItemProcessor<String, String> myProcessorForStepA;
    @Autowired
    private ItemWriter<String> myWriterForStepA;

    @Bean
    public Job myJob() {
        return new JobBuilder("myJob", stepFactory.jobRepository) // JobRepository는 팩토리에서 가져오거나 직접 주입
                .start(stepA())
                // .next(stepB())
                .build();
    }

    @Bean
    public Step stepA() {
        return stepFactory.createCustomStep("stepA", 10,
                myReaderForStepA, myProcessorForStepA, myWriterForStepA);
    }

    // stepB에 대한 설정도 유사하게 가능
}
```

**설명:**

- `StepTemplateFactory` 클래스는 `Step`을 만드는 데 필요한 공통 의존성(`JobRepository`, `transactionManager`)을 주입받습니다.
- `createBaseChunkStepBuilder()` 메서드는 `StepBuilder`의 초기 설정(이름, `JobRepository`, 청크 크기, 트랜잭션 매니저)을 담당하는 일종의 "템플릿" 또는 "기반 설정" 역할을 합니다. 이 메서드 자체는 완전한 `Step`을 반환하지 않으므로 XML의 `abstract="true"`와 유사한 개념으로 볼 수 있습니다.
- `createCustomStep()`, `createAnotherCustomStep()` 같은 메서드들이 이 기반 설정을 사용하여 구체적인 `Reader`, `Processor`, `Writer`를 추가로 설정하여 완전한 `Step` 객체를 생성하여 반환합니다.
- 실제 `Job`을 설정하는 `AnotherJobConfig`에서는 이 `StepTemplateFactory`를 주입받아 필요한 `Step`들을 생성합니다.

---

### Java 설정 방식의 장점

- **타입 안전성:** XML의 문자열 기반 참조 대신 실제 Java 객체를 사용하므로 컴파일 시점에 오류를 발견하기 쉽습니다.
- **리팩토링 용이:** IDE의 지원을 받아 설정을 변경하거나 구조를 개선하기 편리합니다.
- **유연성:** Java 코드를 사용하므로 단순한 설정 상속을 넘어 더 복잡한 조건부 설정이나 동적 설정도 가능합니다.
- **가독성:** 잘 작성된 Java 코드는 그 자체로 문서가 될 수 있으며, XML보다 로직의 흐름을 파악하기 쉬울 수 있습니다.

**결론적으로,** Java 설정에서는 XML과 같은 명시적인 `parent` 나 `abstract` 키워드는 없지만, **객체지향 원칙(상속, 컴포지션), 팩토리 패턴, 설정 클래스 분리 및 `@Import`** 등을 통해 XML 상속이 제공하는 코드 재사용성과 유지보수성의 이점을 충분히 누릴 수 있습니다. 어떤 방법을 선택할지는 프로젝트의 복잡성과 팀의 선호도에 따라 달라질 수 있습니다.

가장 중요한 것은 **공통된 부분을 식별하고, 이를 재사용 가능한 형태로 분리하여 관리**하는 것입니다! 😊

---
