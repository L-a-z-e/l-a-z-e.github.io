---
title: Spring Batch - Configuring a Step (Chunk-oriented Processing)
description: 
author: laze
date: 2025-05-20 00:00:04 +0900
categories: [Dev, SpringBatch]
tags: [SpringBatch]
---
---

**청크 지향 처리 (Chunk-oriented Processing)**

Spring Batch는 가장 일반적인 구현에서 "청크 지향(chunk-oriented)" 처리 스타일을 사용합니다.

청크 지향 처리는 데이터를 한 번에 하나씩 읽고 트랜잭션 경계 내에서 기록되는 '청크(chunks)'를 만드는 것을 의미합니다.

읽은 항목 수가 커밋 간격(commit interval)과 같아지면 전체 청크가 `ItemWriter`에 의해 기록된 다음 트랜잭션이 커밋됩니다.

다음 이미지는 해당 프로세스를 보여줍니다.

다음 의사 코드(pseudo code)는 동일한 개념을 단순화된 형태로 보여줍니다.

```java
List items = new ArrayList();
for(int i = 0; i < commitInterval; i++){ // 커밋 간격만큼 반복
    Object item = itemReader.read();    // ItemReader가 아이템 하나 읽기
    if (item != null) {                 // 아이템이 있으면
        items.add(item);                // 리스트에 추가
    }
}
itemWriter.write(items);               // 모인 아이템들을 ItemWriter로 한 번에 쓰기
```

`ItemWriter`에 항목을 전달하기 전에 항목을 처리하는 선택적 `ItemProcessor`를 사용하여 청크 지향 단계를 구성할 수도 있습니다.

다음 의사 코드는 이것이 단순화된 형태로 어떻게 구현되는지 보여줍니다.

```java
List items = new ArrayList();
for(int i = 0; i < commitInterval; i++){ // 커밋 간격만큼 반복
    Object item = itemReader.read();    // ItemReader가 아이템 하나 읽기
    if (item != null) {                 // 아이템이 있으면
        items.add(item);                // 리스트에 추가
    }
}

List processedItems = new ArrayList();
for(Object item: items){                     // 읽어온 아이템들 각각에 대해
    Object processedItem = itemProcessor.process(item); // ItemProcessor로 가공
    if (processedItem != null) {            // 가공된 아이템이 있으면 (null은 필터링됨)
        processedItems.add(processedItem);  // 가공된 리스트에 추가
    }
}

itemWriter.write(processedItems);          // 가공된 아이템들을 ItemWriter로 한 번에 쓰기
```

Item Processor 및 해당 사용 사례에 대한 자세한 내용은 Item 처리 섹션을 참조하십시오.

---

## Spring Batch의 일꾼, 청크 지향 Step 파헤치기! 🧱👷‍♂️

`Step`은 `Job` 안에서 실제 일을 하는 단위라고 했죠? 그중에서도 가장 흔하게 사용되는 방식이 바로 **청크 지향 처리**예요. 마치 공장에서 물건을 생산하는 라인과 비슷하다고 생각하면 쉬워요.

**"청크(Chunk)"가 뭐길래?**

"청크"는 우리말로 하면 **"덩어리", "묶음"**이라는 뜻이에요. Spring Batch에서 청크 지향 처리는 데이터를 **일정한 크기의 덩어리(청크)로 묶어서 한꺼번에 처리**하는 방식을 말해요.

**청크 지향 처리, 왜 쓰는 걸까요?**

만약 수백만 건의 데이터를 처리해야 한다고 생각해 보세요.

- **한 건씩 읽고, 한 건씩 쓰고, 매번 트랜잭션을 커밋한다면?** 🐢
  - 데이터베이스에 너무 많은 연결과 커밋 요청이 발생해서 성능이 매우 느려질 거예요.
  - 중간에 실패하면 어디서부터 다시 해야 할지 관리하기도 복잡하고요.
- **모든 데이터를 한꺼번에 메모리에 올리고, 마지막에 한 번만 쓴다면?** 💣
  - 데이터가 너무 많으면 메모리가 부족해서 `OutOfMemoryError`가 발생할 수 있어요.
  - 중간에 실패하면 처음부터 다시 해야 할 수도 있고요.

**청크 지향 처리는 이 두 가지 방식의 단점을 보완한 아주 똑똑한 방법이에요!**

---

### 청크 지향 처리의 기본 흐름

청크 지향 `Step`은 기본적으로 우리가 앞에서 잠깐 만났던 **데이터 처리 삼총사**가 활약하는 무대예요.

1. **읽기 (Read) - `ItemReader`**
  - `ItemReader`는 데이터 소스(파일, 데이터베이스 등)에서 데이터를 **하나씩** 끈기 있게 읽어와요.
  - "다음 데이터 주세요!" 하면 하나씩 가져다주는 역할을 하죠.
2. **(선택) 처리 (Process) - `ItemProcessor`**
  - `ItemReader`가 읽어온 데이터를 **하나씩** 받아서, 우리가 원하는 형태로 **가공하거나 변환**하는 역할을 해요.
  - 예를 들어, 고객 이름을 대문자로 바꾸거나, 특정 조건에 따라 고객 등급을 부여하는 등의 작업을 할 수 있어요.
  - **만약 이 단계에서 `null`을 반환하면, 해당 아이템은 "필터링"되어서 다음 단계(쓰기)로 넘어가지 않아요!** "이건 불량품이니 버려!" 하는 거죠.
  - `ItemProcessor`는 필수는 아니에요. 읽어온 데이터를 바로 써야 한다면 생략할 수 있어요.
3. **쓰기 (Write) - `ItemWriter`**
  - `ItemReader`가 읽어오거나 (또는 `ItemProcessor`가 가공한) 아이템들이 차곡차곡 쌓여요.
  - 이렇게 쌓인 아이템의 개수가 미리 정해둔 **"커밋 간격(commit interval)" 또는 "청크 크기(chunk size)"**에 도달하면, `ItemWriter`가 이 **아이템 덩어리(청크)를 한꺼번에** 대상(데이터베이스, 파일 등)에 써요.
  - 그리고 이 "쓰기" 작업 전체가 **하나의 트랜잭션** 안에서 이루어져요. 즉, 청크 단위로 데이터베이스에 커밋(commit)이 발생해요.

**의사 코드(Pseudo Code)로 보는 청크 지향 처리 (ItemProcessor 없을 때):**

```java
// 설정: 청크 크기(commitInterval)는 10이라고 가정
List<MyItem> 아이템_바구니 = new ArrayList<>(); // 아이템을 담을 바구니 (청크)

for (int i = 0; i < 10; i++) { // 청크 크기만큼 반복
    MyItem 하나_읽은_아이템 = itemReader.read(); // 1. ItemReader가 데이터 하나 읽기

    if (하나_읽은_아이템 == null) { // 더 이상 읽을 데이터가 없으면
        break; // 반복 중단
    }
    아이템_바구니.add(하나_읽은_아이템); // 2. 바구니에 담기
}

// 3. 바구니(청크)가 다 찼거나 더 이상 읽을 데이터가 없으면
if (!아이템_바구니.isEmpty()) {
    // 하나의 트랜잭션 시작!
    itemWriter.write(아이템_바구니); // 4. ItemWriter가 바구니에 담긴 아이템들을 한꺼번에 쓰기
    // 트랜잭션 커밋!
}

// 이 과정을 데이터가 없을 때까지 반복!
```

**의사 코드(Pseudo Code)로 보는 청크 지향 처리 (ItemProcessor 있을 때):**

```java
// 설정: 청크 크기(commitInterval)는 10이라고 가정
List<MyProcessedItem> 가공된_아이템_바구니 = new ArrayList<>(); // 가공된 아이템을 담을 바구니

for (int i = 0; i < 10; i++) { // 청크 크기만큼 반복
    MyItem 하나_읽은_아이템 = itemReader.read(); // 1. ItemReader가 데이터 하나 읽기

    if (하나_읽은_아이템 == null) {
        break;
    }

    MyProcessedItem 가공된_아이템 = itemProcessor.process(하나_읽은_아이템); // 2. ItemProcessor가 데이터 하나 가공

    if (가공된_아이템 != null) { // 가공 결과가 null이 아니면 (필터링되지 않았으면)
        가공된_아이템_바구니.add(가공된_아이템); // 3. 가공된 바구니에 담기
    }
}

if (!가공된_아이템_바구니.isEmpty()) {
    // 하나의 트랜잭션 시작!
    itemWriter.write(가공된_아이템_바구니); // 4. ItemWriter가 가공된 바구니의 아이템들을 한꺼번에 쓰기
    // 트랜잭션 커밋!
}
```

---

### 청크 지향 처리의 핵심 장점!

- **성능 향상:** 여러 건의 데이터를 묶어서 한 번의 쓰기 작업과 한 번의 트랜잭션 커밋으로 처리하므로, I/O 및 트랜잭션 오버헤드가 크게 줄어들어요.
- **메모리 효율성:** 전체 데이터를 한 번에 메모리에 올리지 않고, 청크 크기만큼만 유지하므로 대용량 데이터 처리 시 메모리 부족 문제를 방지할 수 있어요.
- **트랜잭션 관리:** 청크 단위로 트랜잭션이 관리되므로, 만약 특정 청크 처리 중 오류가 발생하면 해당 청크만 롤백(rollback)돼요. 이전에 성공적으로 커밋된 청크들은 안전하게 유지되죠.
- **재시작 용이성:** `JobRepository`에 어디까지 처리했는지 (몇 번째 아이템, 어떤 청크) 기록되기 때문에, 실패 시 중단된 지점부터 쉽게 재시작할 수 있어요 (물론 `ItemReader`가 상태를 기억하도록 잘 만들어야 해요!).
- **유연성 및 재사용성:** `ItemReader`, `ItemProcessor`, `ItemWriter`는 각각 독립적인 컴포넌트이므로, 필요에 따라 쉽게 교체하거나 다른 `Step`에서 재사용하기 좋아요.

---

**"커밋 간격(commit interval)" 또는 "청크 크기(chunk size)"는 어떻게 정할까요?**

이건 정답이 없어요. 처리하는 데이터의 특성, 시스템 환경, 메모리 크기 등을 고려해서 적절한 값을 찾아야 해요.

- 너무 작으면: 트랜잭션 커밋이 너무 자주 발생해서 성능이 저하될 수 있어요.
- 너무 크면: 하나의 청크를 처리하는 데 메모리가 많이 필요하고, 실패 시 롤백해야 할 데이터 양이 많아질 수 있어요.

보통 10, 100, 1000 같은 단위로 시작해서 테스트를 통해 최적의 값을 찾아나가는 것이 일반적이에요.
