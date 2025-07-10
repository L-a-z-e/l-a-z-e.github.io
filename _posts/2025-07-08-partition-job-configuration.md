---
title: Spring Batch 파티셔닝 불연속 데이터와 그룹핑 처리 문제 해결 가이드
description: 
author: laze
date: 2025-07-08 00:00:01 +0900
categories: [Dev, SpringBatch]
tags: [SpringBatch, Partition, ItemReader]
---
# Spring Batch 파티셔닝 심층 분석: 불연속 데이터와 그룹핑 처리 문제 해결 가이드

Spring Batch의 파티셔닝(Partitioning)은 대용량 데이터를 병렬 처리하여 배치 성능을 극대화하는 강력한 기능이다. 하지만 실무에 적용하다 보면, 교과서적인 예제에서는 다루지 않는 복잡하고 미묘한 문제들에 부딪히게 된다.

이 글에서는 파티셔닝 적용 시 마주하는 두 가지 레벨의 문제를 정의하고, 특히 비즈니스 로직과 데이터 분할 단위가 충돌하는 고차원적인 문제를 해결하기 위한 구체적인 전략을 심층적으로 분석한다.

## 1. 문제 제기: 파티셔닝, 왜 실무에서는 어려운가?

### Level 1 문제: 불연속적인 ID와 데이터 왜곡

가장 흔하게 마주하는 문제는 **PK(Primary Key)를 기준으로 데이터를 분할할 때 발생**한다. 예를 들어, `id`의 `min`, `max` 값을 구해 `gridSize` 만큼 나누는 방식을 생각해보자.

```java
// 일반적인 min-max 기반 Partitioner 로직
long minId = ...;
long maxId = ...;
long targetSize = (maxId - minId) / gridSize + 1;

for (int i = 0; i < gridSize; i++) {
    long start = minId + (i * targetSize);
    long end = start + targetSize - 1;
    // ... ExecutionContext에 start, end 값 저장
}
```

만약 `id`가 `1, 2, 5, 10, 100, 1000`처럼 불연속적이라면, 어떤 파티션은 수많은 데이터를 처리하는 반면, 다른 파티션은 데이터가 거의 없거나 아예 없는 '데이터 쏠림' 현상이 발생한다. 이는 병렬 처리의 효율을 심각하게 저하시킨다.

이 문제의 해결책은 비교적 잘 알려져 있다.

- **`ROW_NUMBER()` 사용:** 데이터베이스의 윈도우 함수를 이용해 물리적인 행 번호를 기준으로 분할한다.
- **ID 목록 사전 조회:** 처리할 ID 전체를 미리 조회하여 메모리상에서 균등하게 분할한다. (메모리 부담 주의)

하지만 진짜 문제는 여기서부터 시작된다.

### Level 2 문제 (진짜 문제): '데이터 묶음'과 '페이지(Chunk)' 단위의 충돌

Spring Batch의 Chunk 지향 처리는 `pageSize` 만큼 데이터를 읽고(Read), 처리하고(Process), 쓰는(Write) 모델이다. 이 모델은 각 데이터 아이템이 독립적(independent)이라는 암묵적인 가정을 전제로 한다.

하지만 실무 로직은 **'특정 고객의 모든 주문 내역'**, **'하나의 거래에 포함된 모든 상품 목록'** 처럼, **여러 아이템의 묶음(Group)이 하나의 의미 있는 처리 단위(Unit of Work)** 인 경우가 많다.

예를 들어, 10개의 상품 목록이 있어야만 최종 금액 계산이 가능한데, `pageSize=8`로 설정했다면 어떻게 될까?

- **첫 번째 Chunk:** 8개의 상품 데이터만 읽어온다. → 계산 불가.
- **두 번째 Chunk:** 나머지 2개의 상품 데이터가 읽어온다.

첫 번째 Chunk에서 처리하지 못한 8개의 데이터를 두 번째 Chunk로 넘겨서 함께 처리하는 것은 상태를 관리해야 하므로 매우 복잡하며, Batch의 철학에도 부합하지 않는다.

**결국 우리의 목표는 다음과 같다:**

> 어떻게 하면 각 파티션(또는 Step)이 비즈니스 로직이 요구하는 **'온전한 데이터 묶음'**을 독립적으로 처리하도록 만들 수 있을까?
>

## 2. 핵심 원칙: 파티션 경계와 논리적 데이터 경계의 일치

성공적인 파티셔닝의 목표는 단순히 물리적인 데이터 양을 균등하게 나누는 것을 넘어, **'독립적으로 완결될 수 있는 일(Unit of Work)' 단위로 나누는 것**이다.

이를 위한 두 가지 핵심 전략을 소개한다.

## 3. 해결 전략 1: The "Smart" ItemReader (현실적인 접근)

이 전략은 `ItemReader`가 단순히 아이템 하나가 아닌, **'완성된 그룹' 객체 하나를 아이템으로 반환하도록 책임을 확장**하는 방식이다.

- **Partitioner:** 기존처럼 `ROW_NUMBER()` 등을 이용해 대략적인 데이터 처리 범위를 나눈다.
- **Custom ItemReader:** 페이지 단위로 데이터를 읽어 내부 버퍼(Buffer)에 쌓고, 비즈니스 규칙에 따라 '완성된 그룹'을 식별하여 그 그룹 전체를 하나의 아이템으로 반환한다.

### Example Code

```java
@Component
@StepScope // 상태를 가지므로 StepScope로 생성
public class GroupingItemReader implements ItemReader<ItemGroup> {

    // 실제 DB에서 페이지 단위로 데이터를 읽어오는 Reader
    private final JpaPagingItemReader<OrderItem> delegateReader;
    private final List<OrderItem> buffer = new ArrayList<>();
    private Long currentOrderId = null;

    public GroupingItemReader(EntityManagerFactory emf, @Value("#{stepExecutionContext['partitionRange']}") PartitionRange range) {
        // ... JpaPagingItemReader 초기화 로직 (파티션 범위 설정 등) ...
        this.delegateReader = new JpaPagingItemReaderBuilder<OrderItem>()
                .name("delegateReader")
                // ...
                .build();
    }

    @Override
    public ItemGroup read() throws Exception {
        // 루프를 돌며 데이터를 버퍼에 채운다
        while (true) {
            OrderItem nextItem = delegateReader.read();

            // 더 이상 읽을 데이터가 없는 경우
            if (nextItem == null) {
                // 버퍼에 남아있는 데이터가 있다면 마지막 그룹으로 처리
                return buffer.isEmpty() ? null : createGroupAndClearBuffer();
            }

            if (isNewGroup(nextItem)) {
                // 새로운 그룹이 시작되면, 기존 버퍼에 쌓인 내용을 그룹으로 만들어 반환
                ItemGroup completedGroup = createGroupAndClearBuffer();
                buffer.add(nextItem); // 새 그룹의 첫 아이템을 버퍼에 추가
                this.currentOrderId = nextItem.getOrderId();
                return completedGroup;
            } else {
                // 같은 그룹이면 버퍼에 계속 추가
                buffer.add(nextItem);
            }
        }
    }

    private boolean isNewGroup(OrderItem item) {
        // 첫 아이템이거나, 이전 아이템과 orderId가 다르면 새로운 그룹으로 판단
        return this.currentOrderId == null || !this.currentOrderId.equals(item.getOrderId());
    }

    private ItemGroup createGroupAndClearBuffer() {
        if (buffer.isEmpty()) return null;
        ItemGroup group = new ItemGroup(new ArrayList<>(buffer));
        buffer.clear();
        return group;
    }
}
```

- **장점:** 파티셔닝이나 다른 컴포넌트의 변경을 최소화하고, 그룹핑 로직을 `ItemReader`에 집중시켜 문제를 해결할 수 있다. 직관적이고 현실적인 방안이다.
- **단점:** `ItemReader`의 구현이 복잡해진다. 상태(버퍼)를 가지게 되므로 반드시 `@StepScope`로 선언하여 각 스레드(파티션)별로 독립적인 인스턴스를 갖도록 해야 한다.

## 4. 해결 전략 2: Group-Based Partitioning (이상적인 접근)

이는 파티셔닝의 기준 자체를 'ID'나 'ROWNUM'이 아닌 **'그룹 ID'**로 바꾸는, 더 근본적이고 구조적인 접근법이다.

### 실행 흐름

1. **[준비 Step/Tasklet] 그룹 정보 생성:** 본 파티셔닝 스텝 이전에, 전체 데이터를 스캔하여 그룹의 경계를 식별하고, 그 정보를 별도의 테이블(e.g., `GROUP_INFO_TABLE`)에 저장한다.

    ```sql
    -- GROUP_INFO_TABLE
    -- group_id | start_pk | end_pk | item_count
    -- 1        | 1001     | 1050   | 50
    -- 2        | 1051     | 1120   | 70
    ```

2. **[본 Step] 그룹 ID 기반 파티셔닝:** `Partitioner`는 `GROUP_INFO_TABLE`을 읽어, 각 파티션에 `group_id`의 범위를 할당한다. (`group_id` 1~100번은 파티션1, 101~200번은 파티션2...)
3. **[Slave Step] 그룹 단위 처리:** 각 `slave` 스텝의 `ItemReader`는 할당받은 `group_id` 범위를 파라미터로 받아, 해당 그룹들에 속한 데이터를 읽어온다. 이 경우, `ItemReader`는 그룹핑 로직 없이 단순 조회만 수행하면 되므로 구현이 간단해진다.
- **장점:** 각 파티션이 논리적으로 완벽하게 분리된 작업을 수행하여 구조가 매우 깔끔하고 안정적이다.
- **단점:** 전체 데이터를 미리 스캔하는 준비 단계가 추가되어 전체 배치 실행 시간이 길어질 수 있다. 구현 복잡도가 높고, 추가적인 테이블 관리가 필요하다.

## 5. 의사결정 가이드: 어떤 전략을 선택할 것인가?

| 기준 | "Smart" ItemReader | Group-Based Partitioning |
| --- | --- | --- |
| **구현 복잡도** | 중간 (ItemReader에 집중) | 높음 (Tasklet, Partitioner, DB 테이블 등) |
| **구조적 안정성** | 중간 (상태 관리에 주의 필요) | 높음 (명확한 작업 분리) |
| **실행 시간** | 상대적으로 빠름 (사전 스캔 없음) | 상대적으로 느림 (사전 스캔 시간 추가) |
| **적합한 상황** | 그룹핑 규칙이 복잡하지 않고, 빠른 개발이 필요할 때 | 데이터 정합성이 매우 중요하고, 사전 스캔 비용을 감당할 수 있을 때 |

가장 핵심적인 질문은 **"전체 데이터에 대한 사전 스캔이 비용/시간적으로 허용 가능한가?"** 이다.

- **Yes:** `Group-Based Partitioning` 전략을 고려한다. 더 견고하고 장기적으로 유지보수에 유리하다.
- **No:** `Smart ItemReader` 전략을 사용한다. 대부분의 경우 이 방법으로 충분히 문제를 해결할 수 있다.

## 6. 결론

Spring Batch 파티셔닝은 단순히 기술을 적용하는 것을 넘어, 처리할 **데이터의 특성**과 **비즈니스 로직의 요구사항**을 깊이 이해해야 하는 엔지니어링 영역이다.

데이터의 물리적 분포와 논리적 경계가 다를 때, 우리는 `ItemReader`, `Partitioner` 등 각 컴포넌트의 책임을 유연하게 재분배하여 문제를 해결해야 한다.

단순한 `min-max` 분할에서 시작하여, 데이터 그룹핑의 문제에 부딪혔을 때 본문에서 제시한 두 가지 전략은 강력한 해결의 실마리를 제공할 것이다.

---
