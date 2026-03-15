---
title: "CQRS 도입기: 재고 읽기 모델 분리와 Payment 서비스 독립"
description: "shopping-service 분해 이후 CQRS로 재고 조회를 분리하고, Payment Intent 패턴으로 결제 서비스를 독립시키면서 적용한 Outbox 패턴과 이벤트 소싱 원장까지의 기록"
author: laze
date: 2026-03-13 10:00:00 +0900
categories: [Dev, MSA]
tags: [CQRS, EventDriven, PaymentIntent, OutboxPattern]
---

## 배경

[이전 글]({% raw %}{% post_url 2026-03-12-shopping-monolith-to-microservices %}{% endraw %})에서 shopping-service를 Buyer, Seller, Settlement 3개 서비스로 분해했습니다. 분해 직후 두 가지 문제가 드러났습니다.

**재고 조회 병목.** 구매자가 상품 목록을 볼 때마다 shopping-service가 seller-service에 Feign으로 재고를 요청합니다. 상품 10개를 한 페이지에 보여주면 10번의 동기 호출이 발생하고, seller-service에 장애가 나면 상품 목록 전체가 먹통이 됩니다.

**결제 로직 결합.** 결제는 PG 연동, 감사 추적, 보안 요건이 주문과 전혀 다른 변경 주기를 가집니다. 주문 흐름을 수정할 때 결제 코드를 건드려야 하고, 그 반대도 마찬가지였습니다. 결제만 독립적으로 배포하거나 스케일링할 수 없는 구조였습니다.

## 재고 CQRS: Command와 Query 분리

해결책은 읽기 모델을 분리하는 것이었습니다. seller-service가 재고를 변경하면(Command), shopping-service가 자체 읽기 전용 테이블에 그 상태를 복제합니다(Query). 구매자의 재고 조회는 로컬 DB에서 처리되므로 seller-service 호출이 사라집니다.

### Command Side — seller-service

재고 변경은 네 가지 연산으로 구성됩니다.

```java
public class Inventory extends BaseEntity {
    private Integer availableQuantity;
    private Integer reservedQuantity;
    private Integer totalQuantity;

    @Version
    private Long version;

    public void reserve(int quantity) {
        if (this.availableQuantity < quantity)
            throw new CustomBusinessException(SellerErrorCode.INSUFFICIENT_STOCK);
        this.availableQuantity -= quantity;
        this.reservedQuantity += quantity;
    }

    public void deduct(int quantity) {
        this.reservedQuantity -= quantity;
        this.totalQuantity -= quantity;
    }

    public void release(int quantity) {
        this.reservedQuantity -= quantity;
        this.availableQuantity += quantity;
    }

    public void restore(int quantity) {
        // Saga 보상: deduct의 역연산
        this.availableQuantity += quantity;
        this.totalQuantity += quantity;
    }
}
```

`InventoryServiceImpl`은 각 연산 수행 후 `ApplicationEventPublisher`로 스프링 이벤트를 발행합니다. 이 이벤트를 `InventoryEventPublisher`가 받아 Kafka로 전달합니다.

```java
@TransactionalEventListener(phase = TransactionPhase.AFTER_COMMIT)
public void handleInventoryChanged(InventoryChangedEvent event) {
    String key = String.valueOf(event.getProductId());
    avroKafkaTemplate.send(SellerTopics.INVENTORY_CHANGED, key, event)
            .whenComplete((result, ex) -> {
                if (ex != null) {
                    log.error("Failed to publish InventoryChangedEvent: productId={}",
                            event.getProductId(), ex);
                }
            });
}
```

`AFTER_COMMIT` 시점에 발행하는 이유가 있습니다. DB에 재고 변경이 커밋되기 전에 이벤트가 나가면, 컨슈머가 아직 존재하지 않는 상태를 읽게 됩니다. 트랜잭션 커밋 이후에만 발행해서 읽기 모델이 항상 커밋된 상태만 반영하도록 보장합니다.

### Query Side — shopping-service

shopping-service에는 읽기 전용 `Inventory` 테이블이 있습니다. `InventoryEventConsumer`가 Kafka 이벤트를 수신해서 이 테이블을 갱신합니다.

```java
@KafkaListener(topics = SellerTopics.INVENTORY_CHANGED,
               groupId = "shopping-service",
               containerFactory = "avroKafkaListenerContainerFactory")
@Transactional
public void onInventoryChanged(InventoryChangedEvent event) {
    Inventory inventory = inventoryRepository.findByProductId(event.getProductId())
            .orElse(null);

    if (inventory == null) {
        inventory = Inventory.builder()
                .productId(event.getProductId())
                .initialQuantity(event.getAvailableQuantity())
                .build();
    }

    inventory.adjust(event.getAvailableQuantity(), event.getReservedQuantity());
    inventoryRepository.save(inventory);
}
```

핵심은 **스냅샷 기반 동기화**입니다. 이벤트가 `available=8, reserved=2`라고 보내면, 읽기 모델은 그 값으로 덮어씁니다. 델타(`-2`)가 아니라 전체 상태를 매번 보내므로 이벤트 순서가 뒤바뀌거나 중복 수신되어도 최종 상태가 맞습니다. 멱등성이 구조적으로 보장됩니다.

**트레이드오프**: 이벤트 발행과 소비 사이에 지연이 있으므로 읽기 모델은 최종 일관성(eventual consistency)입니다. 구매자가 보는 재고와 실제 재고 사이에 짧은 차이가 존재할 수 있습니다. 다만 주문 시 실제 재고 차감은 seller-service의 Command Side에서 처리하므로, 읽기 모델의 지연이 과매도로 이어지지는 않습니다.

### 실시간 스트림 — Redis Pub/Sub + SSE

읽기 모델의 지연을 줄이기 위해 실시간 채널도 구성했습니다. `InventoryEventConsumer`가 Kafka 이벤트를 수신하면 Redis Pub/Sub로도 발행하고, 프론트엔드는 SSE로 구독합니다.

```java
@GetMapping(value = "/stream", produces = MediaType.TEXT_EVENT_STREAM_VALUE)
public Flux<ServerSentEvent<SseEnvelope<?>>> streamInventory(
        @RequestParam List<Long> productIds) {

    Flux<ServerSentEvent<SseEnvelope<?>>> heartbeat = Flux.interval(Duration.ofSeconds(30))
            .map(i -> ServerSentEvent.<SseEnvelope<?>>builder()
                    .comment("heartbeat")
                    .data(SseEnvelope.heartbeat())
                    .build());

    Flux<ServerSentEvent<SseEnvelope<?>>> updates = streamService.subscribe(productIds)
            .map(update -> ServerSentEvent.<SseEnvelope<?>>builder()
                    .id(String.valueOf(update.getProductId()))
                    .event("inventory-update")
                    .data(SseEnvelope.of("inventory-update", update))
                    .build());

    return Flux.merge(heartbeat, updates);
}
```

`InventoryStreamService`는 `ConcurrentHashMap`으로 상품별 `Sinks.Many`를 관리합니다. 구독자가 0이 되면 Redis 리스너를 해제해서 불필요한 리소스를 정리합니다.

## Avro 이벤트 계약

서비스 간 이벤트는 Avro 스키마로 정의합니다. `event-contracts` 모듈에 `.avsc` 파일을 두고, Gradle 플러그인이 Java 클래스를 자동 생성합니다.

```json
{
  "type": "record",
  "name": "InventoryChangedEvent",
  "namespace": "com.portal.universe.event.seller",
  "fields": [
    { "name": "productId", "type": "long" },
    { "name": "availableQuantity", "type": "int" },
    { "name": "reservedQuantity", "type": "int" },
    { "name": "totalQuantity", "type": "int" },
    { "name": "changeType", "type": "InventoryChangeType" },
    { "name": "quantity", "type": "int" },
    { "name": "timestamp", "type": { "type": "long", "logicalType": "timestamp-millis" } }
  ]
}
```

파티션 키는 `productId`입니다. 같은 상품에 대한 이벤트는 반드시 같은 파티션에 들어가므로, 한 컨슈머가 순서대로 처리합니다. 스냅샷 기반이라 순서가 뒤바뀌어도 문제없지만, 파티션 키를 맞추면 불필요한 갱신 횟수를 줄일 수 있습니다.

Avro를 선택한 이유는 스키마 진화(schema evolution)입니다. 필드를 추가할 때 default 값을 지정하면 기존 컨슈머가 깨지지 않습니다. JSON은 이런 호환성 보장이 없고, Protobuf는 이 프로젝트의 NestJS 서비스에서 생태계가 불편했습니다.

## Payment 서비스 분리

결제를 shopping-service에서 분리해 payment-service(:8090)로 독립시켰습니다.

### Payment Intent 패턴

결제에서 가장 위험한 시나리오는 클라이언트가 금액을 변조하는 것입니다. 프론트엔드에서 `amount=100`을 보내면 서버가 그대로 PG에 전달하는 구조는 변조에 취약합니다.

Payment Intent 패턴은 이 문제를 구조적으로 차단합니다. 서버가 주문 정보를 기반으로 금액을 확정한 Intent를 먼저 생성하고, 클라이언트는 그 Intent ID만으로 결제를 확인합니다.

```java
@Transactional
public IntentResponse createIntent(CreateIntentRequest request) {
    // 동일 주문에 대한 기존 Intent 확인 (멱등성)
    intentRepository.findByOrderNumber(request.orderNumber())
            .ifPresent(existing -> {
                if (existing.getStatus().isPayable()) {
                    throw new CustomBusinessException(
                        PaymentErrorCode.INTENT_ALREADY_EXISTS);
                }
            });

    PaymentIntent intent = PaymentIntent.builder()
            .orderNumber(request.orderNumber())
            .userId(request.userId())
            .amount(request.amount())     // 서버가 계산한 금액
            .metadata(request.metadata())
            .expiryMinutes(expiryMinutes) // 기본 30분
            .build();

    return IntentResponse.from(intentRepository.save(intent));
}
```

Intent의 상태 머신은 다음과 같습니다.

```
REQUIRES_PAYMENT → PROCESSING → SUCCEEDED
                              → FAILED → (재시도 가능)
                 → EXPIRED (30분 초과)
                 → CANCELLED
```

### 결제 확인 흐름

`confirmPayment`는 Intent를 검증한 뒤 PG 호출을 수행합니다.

```java
@Transactional
public PaymentResponse confirmPayment(String userId, String intentId,
                                      ConfirmPaymentRequest request) {
    PaymentIntent intent = intentRepository.findByIntentId(intentId)
            .orElseThrow(() -> new CustomBusinessException(
                PaymentErrorCode.INTENT_NOT_FOUND));

    intent.validateOwnership(userId);
    intent.validatePayable();

    intent.startProcessing();
    intentRepository.save(intent);

    Payment payment = Payment.builder()
            .intentId(intent.getIntentId())
            .orderNumber(intent.getOrderNumber())
            .userId(userId)
            .amount(intent.getAmount())  // Intent에서 금액을 가져옴
            .paymentMethod(request.paymentMethod())
            .build();
    payment = paymentRepository.save(payment);

    PgResponse pgResponse = mockPGClient.processPayment(
            payment.getPaymentNumber(), payment.getAmount(),
            payment.getPaymentMethod().name(), request.cardNumber());

    if (pgResponse.success()) {
        payment.complete(pgResponse.transactionId(), pgResponse.rawResponse());
        intent.succeed();
        eventPublisher.publishPaymentCompleted(/* Outbox에 저장 */);
    } else {
        payment.fail(pgResponse.errorCode() + ": " + pgResponse.message(),
                    pgResponse.rawResponse());
        intent.fail();
        eventPublisher.publishPaymentFailed(/* Outbox에 저장 */);
    }

    return PaymentResponse.from(payment);
}
```

클라이언트는 `amount`를 보내지 않습니다. `intent.getAmount()`로 서버가 확정한 금액을 PG에 전달하므로, 금액 변조가 구조적으로 불가능합니다.

`MockPGClient`는 90% 성공률로 결제를 시뮬레이션합니다. 실제 PG사 연동 시에는 이 클래스만 교체하면 됩니다.

### Saga 통합

shopping-service의 Saga Orchestrator가 주문 취소 시 환불을 요청합니다. payment-service는 내부 API를 제공합니다.

```java
@RestController
@RequestMapping("/internal")
public class InternalPaymentController {

    @PostMapping("/intents")
    public ApiResponse<IntentResponse> createIntent(
            @RequestHeader("X-Internal-Token") String token,
            @RequestBody CreateIntentRequest request) { /* ... */ }

    @PostMapping("/refund/{orderNumber}")
    public ApiResponse<Void> refund(
            @RequestHeader("X-Internal-Token") String token,
            @PathVariable String orderNumber) { /* ... */ }
}
```

`X-Internal-Token` 헤더로 서비스 간 인증을 처리합니다. 외부에서는 API Gateway가 이 경로를 차단합니다.

## Outbox 패턴: 이벤트 발행의 원자성

payment-service가 결제를 완료하면 Kafka로 `PaymentCompletedEvent`를 발행해야 합니다. 문제는 DB 커밋과 Kafka 발행이 하나의 트랜잭션이 아니라는 것입니다. DB 커밋은 성공했는데 Kafka 발행이 실패하면 shopping-service는 결제 완료를 영원히 모릅니다.

Outbox 패턴으로 해결했습니다. 이벤트를 Kafka에 직접 보내는 대신, 같은 DB 트랜잭션 내에서 `outbox_events` 테이블에 저장합니다.

```java
@Component
public class PaymentEventPublisher {

    private final OutboxEventRepository outboxEventRepository;

    public void publishPaymentCompleted(PaymentCompletedEvent event) {
        outboxEventRepository.save(
            OutboxEvent.create(ShoppingTopics.PAYMENT_COMPLETED,
                              event.getPaymentNumber().toString(), event));
    }
}
```

Payment 엔티티 저장과 OutboxEvent 저장이 같은 `@Transactional` 안에서 실행되므로, 둘 다 커밋되거나 둘 다 롤백됩니다.

별도 스케줄러가 3초마다 `outbox_events`를 폴링해서 Kafka로 발행합니다.

```java
@Scheduled(fixedDelay = 3000)
@Transactional
public void pollAndPublish() {
    List<OutboxEvent> pending = outboxEventRepository.findPendingForUpdate(BATCH_SIZE);
    if (pending.isEmpty()) return;

    for (OutboxEvent event : pending) {
        try {
            SpecificRecord record = deserialize(event.getEventType(), event.getPayload());
            avroKafkaTemplate.send(event.getTopic(), event.getEventKey(), record).get();
            event.markPublished();
        } catch (Exception ex) {
            event.incrementRetry();
            if (event.getRetryCount() >= MAX_RETRIES) {
                event.markFailed();
            }
        }
    }
}
```

폴링 쿼리의 `FOR UPDATE SKIP LOCKED`가 핵심입니다.

```sql
SELECT * FROM outbox_events
WHERE status = 'PENDING'
ORDER BY created_at
LIMIT :batchSize
FOR UPDATE SKIP LOCKED
```

다중 인스턴스에서 같은 이벤트를 중복 처리하지 않도록, 이미 다른 인스턴스가 잡은 행은 건너뜁니다. `SKIP LOCKED` 덕분에 잠금 경합 없이 수평 확장이 가능합니다.

이 패턴은 shopping-service, seller-service, payment-service 3개 서비스에 동일하게 적용했습니다. shopping-service의 Outbox는 `OrderCreatedEvent`를 AWS EventBridge로도 추가 발행하는 차이가 있지만, 기본 구조는 같습니다.

**트레이드오프**: Outbox 패턴은 최대 3초의 발행 지연이 생깁니다. 폴링 주기를 줄이면 지연은 줄지만 DB 부하가 늘어납니다. 또한 at-least-once 보장이므로 컨슈머 측에서 멱등성 처리가 필수입니다. 그럼에도 메시지 손실을 구조적으로 방지하는 것이 3초 지연보다 중요한 선택이었습니다.

## Settlement Ledger: 이벤트 소싱 원장

settlement-service는 결제와 취소 이벤트를 원장(ledger)에 기록합니다. 기존 레코드를 수정하지 않고, 항상 새 행을 추가하는 append-only 방식입니다.

```java
@KafkaListener(topics = ShoppingTopics.ORDER_SETTLEMENT_CREATED,
               groupId = "shopping-settlement-service",
               containerFactory = "avroKafkaListenerContainerFactory")
public void onOrderSettlementCreated(OrderSettlementCreatedEvent event) {
    // 판매자별로 금액 합산
    Map<Long, BigDecimal> sellerAmounts = event.getItems().stream()
            .collect(Collectors.groupingBy(
                    OrderItemInfo::getSellerId,
                    Collectors.reducing(BigDecimal.ZERO,
                            item -> item.getPrice()
                                .multiply(BigDecimal.valueOf(item.getQuantity())),
                            BigDecimal::add)));

    sellerAmounts.forEach((sellerId, amount) ->
            saveLedgerIdempotent(event.getOrderNumber(), sellerId,
                    "PAYMENT_COMPLETED", amount, event.getSettledAt()));
}
```

주문 취소 시에는 역분개(negate)로 처리합니다. 기존 `PAYMENT_COMPLETED` 행을 삭제하는 대신, 같은 금액의 음수 행을 추가합니다.

```java
@KafkaListener(topics = ShoppingTopics.ORDER_CANCELLED,
               groupId = "shopping-settlement-service",
               containerFactory = "avroKafkaListenerContainerFactory")
public void onOrderCancelled(OrderCancelledEvent event) {
    List<SettlementLedger> paymentLedgers = ledgerRepository
            .findByOrderNumberAndEventType(event.getOrderNumber(),
                    "PAYMENT_COMPLETED");

    for (SettlementLedger paymentLedger : paymentLedgers) {
        saveLedgerIdempotent(
                event.getOrderNumber(),
                paymentLedger.getSellerId(),
                "ORDER_CANCELLED",
                paymentLedger.getAmount().negate(),
                event.getCancelledAt());
    }
}
```

멱등성은 `UNIQUE(order_number, seller_id, event_type)` 제약으로 보장합니다. 같은 이벤트가 중복 수신되면 `DataIntegrityViolationException`이 발생하고, catch해서 무시합니다.

```java
private void saveLedgerIdempotent(String orderNumber, Long sellerId,
                                  String eventType, BigDecimal amount, Instant eventAt) {
    try {
        ledgerRepository.save(SettlementLedger.builder()
                .orderNumber(orderNumber).sellerId(sellerId)
                .eventType(eventType).amount(amount).eventAt(eventAt)
                .build());
    } catch (DataIntegrityViolationException e) {
        log.warn("Duplicate event ignored: order={}, seller={}, type={}",
                orderNumber, sellerId, eventType);
    }
}
```

Spring Batch가 매일 02:00에 `processed=false`인 원장 레코드를 집계해서 판매자별 Settlement을 생성합니다. 원장에 모든 이력이 남아 있으므로 특정 시점의 정산 상태를 언제든 재구성할 수 있습니다.

## 교훈

| 패턴 | 해결한 문제 | 비용 |
|------|-----------|------|
| CQRS (재고) | seller-service 장애 전파, 조회 지연 | 최종 일관성, 데이터 중복 |
| Payment Intent | 클라이언트 금액 변조 | Intent 생성/만료 관리 |
| Outbox | DB 커밋 ↔ Kafka 발행 불일치 | 최대 3초 지연, DB 폴링 부하 |
| Settlement Ledger | 정산 이력 추적, 취소 처리 | 저장 공간, 집계 배치 필요 |

읽기와 쓰기를 분리하면 각각을 독립적으로 최적화할 수 있습니다. 재고 조회에 캐시를 추가하거나 인덱스를 변경해도 seller-service의 쓰기 성능에 영향이 없습니다.

Payment Intent로 금액 확정을 서버에서 하면, 검증 로직을 아무리 촘촘히 짜는 것보다 확실합니다. 클라이언트가 보내는 값을 신뢰하지 않는 구조 자체가 방어입니다.

Outbox 패턴은 구현이 단순한 대신, 폴링 주기만큼의 지연과 DB I/O가 발생합니다. CDC(Change Data Capture) 기반이면 지연을 거의 없앨 수 있지만, Debezium 같은 인프라를 운영해야 하는 부담이 있습니다. 현 규모에서는 3초 폴링이 충분하다고 판단했습니다.
