---
title: "Saga 보상 체계에서 발견한 8가지 결함과 해결"
description: "Shopping 서비스 분해 후 Saga 오케스트레이터의 보상 로직에서 발견한 8가지 정합성 문제와 각각의 해결 과정 기록"
author: laze
date: 2026-03-12 12:00:00 +0900
categories: [Dev, MSA]
tags: [Saga, DistributedTransaction, Compensation]
---

## 배경

Shopping 모놀리스를 3개 서비스로 분해한 후, 주문 프로세스는 Saga 패턴으로 관리하고 있습니다. 주문 생성 → 재고 예약 → 결제 → 재고 차감 → 배송 생성 → 주문 확정까지 5단계를 거치고, 중간에 실패하면 완료된 단계를 역순으로 보상합니다.

분해 직후 정합성 검증을 하면서 8가지 결함을 발견했고, 하나씩 수정했습니다.

## D1. 서비스 간 내부 API 인증 부재

분해 전에는 메서드 호출이었으므로 인증이 필요 없었지만, 분해 후 Feign으로 호출하면서 `/internal/*` API가 인증 없이 노출되었습니다.

**해결**: `X-Internal-Token` 헤더 기반 인증을 추가했습니다. common-library에 `InternalTokenAuthFilter`를 만들어, `/internal/*` 경로로 들어오는 요청에서 토큰을 검증하고 `ROLE_INTERNAL` 권한을 부여합니다.

```java
@Bean
public RequestInterceptor requestInterceptor(
        @Value("${app.internal.token}") String internalToken) {
    return requestTemplate -> {
        requestTemplate.header("X-Internal-Token", internalToken);
    };
}
```

Feign 요청 인터셉터가 모든 내부 호출에 토큰을 자동 첨부하므로, 개별 호출마다 토큰을 넣을 필요가 없습니다.

## D2. sellerId 전파 누락

주문 생성 시 상품의 판매자 정보가 필요한데, 분해 후 상품 정보는 seller-service에 있습니다. 매 주문마다 seller-service를 조회하면 지연과 장애 전파 위험이 있습니다.

**해결**: 장바구니에 상품을 담는 시점에 `sellerId`를 스냅샷으로 저장합니다. `CartItem`과 `OrderItem` 테이블에 `seller_id` 컬럼을 추가하여, 주문 처리 중에는 seller-service를 호출하지 않도록 했습니다.

**트레이드오프**: 스냅샷 방식이므로 장바구니에 담긴 후 판매자가 바뀌면 stale `sellerId`가 주문에 들어갈 수 있습니다. 다만 `sellerId`는 상품 등록 시 결정되고 변경 빈도가 극히 낮으므로, 실시간 조회 비용과 장애 전파 위험 대비 허용 가능한 수준으로 판단했습니다.

## D3. Saga 보상 로직 불완전

기존 보상은 `releaseStock()`만 있었습니다. 결제까지 진행된 후 실패하면 재고 차감을 되돌려야 하는데, 이 경로가 없었습니다.

**해결**: 재고 상태에 따라 두 가지 보상 연산을 구분했습니다.

```java
// DEDUCT 완료 상태에서의 보상 — 차감 역연산
public void restore(int quantity) {
    this.availableQuantity += quantity;
    this.totalQuantity += quantity;
}

// RESERVE만 완료된 상태에서의 보상 — 예약 해제
public void release(int quantity) {
    this.reservedQuantity -= quantity;
    this.availableQuantity += quantity;
}
```

보상 서비스에서 `isStepCompleted(DEDUCT_INVENTORY)` 여부에 따라 `restore`와 `release`를 분기합니다.

```java
public void compensateSagaSteps(Order order, SagaState sagaState) {
    boolean deductCompleted = sagaState.isStepCompleted(SagaStep.DEDUCT_INVENTORY);

    // 역순 보상
    if (sagaState.isStepCompleted(SagaStep.CREATE_DELIVERY))
        deliveryService.cancelDelivery(order.getId());

    if (deductCompleted)
        sellerInventoryClient.restoreStock(stockRequest);   // 차감 역연산

    if (sagaState.isStepCompleted(SagaStep.PROCESS_PAYMENT))
        paymentClient.refundForCompensation(orderNumber);    // Silent 실패

    if (!deductCompleted && sagaState.isStepCompleted(SagaStep.RESERVE_INVENTORY))
        sellerInventoryClient.releaseStock(stockRequest);    // 예약 해제
}
```

결제 환불이 실패해도 보상 전체를 중단하지 않습니다(Silent 실패). 환불 실패 건은 `DeadSagaRecoveryScheduler`가 주기적으로 재시도하고, 최대 재시도 횟수를 초과하면 운영 알림을 발송하여 수동으로 처리합니다.

## D4. Dead Saga 미처리

네트워크 타임아웃이나 프로세스 crash로 Saga가 중간에 멈추면, 재고가 예약된 채로 영구히 잠기게 됩니다.

**해결**: `DeadSagaRecoveryScheduler`가 5분마다 30분 이상 멈춘 Saga를 탐지하여 보상을 실행합니다.

```java
@Scheduled(fixedDelay = 300_000, initialDelay = 60_000)
@Transactional
public void recoverDeadSagas() {
    Instant cutoff = Instant.now().minus(30, ChronoUnit.MINUTES);
    List<SagaState> deadSagas = sagaStateRepository.findDeadSagas(
        List.of(SagaStatus.STARTED, SagaStatus.COMPENSATING), cutoff);

    for (SagaState saga : deadSagas) {
        sagaCompensationService.compensate(saga, "Dead saga recovery (timeout)");
    }
}
```

다중 인스턴스에서 동일 Dead Saga를 동시에 처리하는 것을 방지하기 위해 `PESSIMISTIC_WRITE + SKIP_LOCKED`를 사용합니다.

```java
@Lock(LockModeType.PESSIMISTIC_WRITE)
@QueryHints(@QueryHint(name = "jakarta.persistence.lock.timeout", value = "-2")) // SKIP_LOCKED
@Query("SELECT s FROM SagaState s WHERE s.status IN :statuses AND s.startedAt < :cutoff")
List<SagaState> findDeadSagas(@Param("statuses") List<SagaStatus> statuses,
                               @Param("cutoff") Instant cutoff);
```

`SKIP_LOCKED`(`-2`)는 다른 트랜잭션이 이미 잠근 row를 건너뛰므로, 인스턴스 간 경합 없이 Dead Saga를 분산 처리할 수 있습니다.

## D5. completedSteps 타입 안전성

Saga의 완료 단계를 `Set<String>`으로 저장하고 있었습니다. "RESERVE_INVENTORY"라는 문자열로 비교하다 보니, 오타 하나로 보상 분기가 잘못될 위험이 있었습니다.

**해결**: JPA `AttributeConverter`로 DB의 CSV 문자열과 `Set<SagaStep>` enum 사이를 자동 변환합니다.

```java
@Converter
public class SagaStepSetConverter implements AttributeConverter<Set<SagaStep>, String> {

    @Override
    public String convertToDatabaseColumn(Set<SagaStep> steps) {
        return steps.stream().map(SagaStep::name).collect(Collectors.joining(","));
    }

    @Override
    public Set<SagaStep> convertToEntityAttribute(String dbData) {
        return Arrays.stream(dbData.split(","))
            .map(SagaStep::valueOf)
            .collect(Collectors.toCollection(() -> EnumSet.noneOf(SagaStep.class)));
    }
}
```

DB에는 `"RESERVE_INVENTORY,PROCESS_PAYMENT"` 형태로 저장되고, Java 코드에서는 `Set<SagaStep>`으로 다루므로 컴파일 타임에 오류를 잡을 수 있습니다.

## D6. 결제 취소 → 주문 취소 이벤트 미연결

결제가 취소되면 주문도 취소되어야 하는데, 이 경로가 없었습니다. 결제 취소는 비동기로 처리해도 사용자 경험에 영향이 없고, 결제 모듈과 주문 모듈 사이의 결합도를 낮추기 위해 Kafka 이벤트 기반으로 구현했습니다.

```java
@KafkaListener(topics = ShoppingTopics.PAYMENT_CANCELLED, groupId = "shopping-service")
public void onPaymentCancelled(PaymentCancelledEvent event) {
    orderService.cancelOrder(event.getUserId().toString(),
        event.getOrderNumber().toString(),
        new CancelOrderRequest("Payment cancelled: " + event.getCancelReason()));
}
```

## D7. 보상 트랜잭션 self-invocation

`OrderSagaOrchestrator`에서 `compensate()` 메서드에 `@Transactional(propagation = REQUIRES_NEW)`를 걸었는데, 같은 클래스에서 호출하면 Spring AOP 프록시를 거치지 않아 새 트랜잭션이 생성되지 않았습니다.

**해결**: `SagaCompensationService`를 별도 Bean으로 분리하여 외부 호출이 되도록 했습니다.

```java
@Service
public class SagaCompensationService {

    @Transactional(propagation = Propagation.REQUIRES_NEW)
    public void compensate(SagaState sagaState, String errorMessage) {
        // 보상 로직 — 별도 트랜잭션에서 실행
    }
}
```

## D8. 동일 주문 Saga 중복 생성

동시에 같은 주문에 대해 Saga가 두 개 생성되면, 재고가 이중으로 예약됩니다.

**해결**: `saga_states` 테이블에 `order_number` unique constraint를 추가했습니다.

```sql
ALTER TABLE saga_states
    ADD CONSTRAINT uq_saga_order_number UNIQUE (order_number);
```

## 정리

| # | 결함 | 해결 |
|---|------|------|
| D1 | 내부 API 인증 부재 | X-Internal-Token + InternalTokenAuthFilter |
| D2 | sellerId 전파 누락 | CartItem/OrderItem에 스냅샷 저장 |
| D3 | 보상 로직 불완전 | restore(차감 역연산) + release(예약 해제) 분리 |
| D4 | Dead Saga 미처리 | 5분 주기 스케줄러 + PESSIMISTIC_WRITE + SKIP_LOCKED |
| D5 | completedSteps 타입 불안전 | SagaStepSetConverter (CSV ↔ Set&lt;SagaStep&gt;) |
| D6 | 결제 취소 → 주문 취소 미연결 | PaymentCancelledEvent Kafka consumer |
| D7 | 보상 트랜잭션 self-invocation | SagaCompensationService 별도 Bean 분리 |
| D8 | 동일 주문 Saga 중복 | order_number unique constraint |

## 마치며

대부분 서비스 분해 전에는 문제가 되지 않던 것들입니다. 모놀리스에서는 같은 프로세스 안에서 메서드를 호출하므로 인증, 데이터 전파, 트랜잭션 경계 같은 것을 의식하지 않아도 됩니다. 서비스를 분해하면 이런 암묵적 보장들이 사라지고, 명시적으로 처리해야 합니다.
