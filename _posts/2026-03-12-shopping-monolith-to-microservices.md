---
title: "Shopping 모놀리스를 3개 서비스로 분해한 과정"
description: "단일 shopping-service를 Buyer/Seller/Settlement 3개 마이크로서비스로 분해하면서 겪은 도메인 분리, Saga 오케스트레이션, 서비스 간 통신 설계 기록"
author: laze
date: 2026-03-12 10:30:00 +0900
categories: [Dev, MSA]
tags: [Microservices, DDD, ServiceDecomposition]
---

## 배경

shopping-service는 상품 관리, 장바구니, 주문, 결제, 배송, 쿠폰, 타임딜, 재고, 정산까지 모든 쇼핑 관련 기능을 담고 있었습니다. 단일 서비스 안에 10개 이상의 도메인이 얽혀 있었고, 하나의 도메인을 수정할 때 관련 없는 도메인에서 사이드이펙트가 발생하는 일이 잦았습니다. 배포 역시 쿠폰 로직 한 줄을 고치기 위해 주문·결제 코드까지 함께 배포해야 했고, 멀티셀러 마켓플레이스로 확장하면서 Seller 관련 기능이 급격히 늘어나 분리가 더 이상 미룰 수 없는 상황이었습니다.

## 분해 기준: Bounded Context

도메인 분석 결과, 크게 세 가지 맥락으로 나뉩니다.

| 맥락 | 도메인 | 핵심 액터 |
|------|--------|----------|
| **구매** | Cart, Order, Payment, Delivery, Search | Buyer |
| **판매** | Seller, Product, Inventory, Coupon, TimeDeal, Queue | Seller |
| **정산** | Settlement, Ledger | 시스템 (Batch) |

각 맥락은 서로 다른 변경 주기를 가집니다. 상품 CRUD는 판매자가 수시로 하지만, 정산은 주기적 배치입니다. 주문/결제 흐름은 구매자의 행동에 따라 바뀌지만, 재고 관리 로직은 판매자 정책에 따릅니다. 이 변경 주기의 차이가 서비스 분리의 가장 명확한 근거였습니다.

## 분해 결과

```
shopping-service (:8083)           → Buyer 전용 (Cart, Order, Payment, Delivery, Search)
shopping-seller-service (:8088)    → Seller 전용 (Seller, Product, Inventory, Coupon, TimeDeal)
shopping-settlement-service (:8089)→ Settlement (Spring Batch, Ledger)
```

### 데이터베이스 전략

동일 PostgreSQL 인스턴스에서 스키마를 분리했습니다.

```
shopping_db              → Cart, Order, Payment, Delivery, SagaState
shopping_seller_db       → Seller, Product, Inventory, Coupon, TimeDeal
shopping_settlement_db   → Settlement, SettlementLedger, Spring Batch Meta
```

완전한 물리적 DB 분리가 아닌 논리적 분리를 선택했습니다. 현 단계에서 서비스별 트래픽 차이가 크지 않아 독립 스케일링 필요성이 없고, 논리 분리의 단점(스키마 간 FK 유혹, 장애 격리 불가)을 감수하는 대신 운영 복잡도를 줄이는 것을 우선했습니다. 프로덕션에서 트래픽 패턴이 분화되면 물리적으로 분리할 수 있습니다.

## 서비스 간 통신

분해 후 가장 큰 과제는 서비스 간 통신입니다. 주문 시 재고를 예약해야 하는데, 재고는 이제 seller-service에 있습니다.

### Feign Client — 동기 통신

주문 Saga처럼 즉각적인 응답이 필요한 경우 Feign Client로 동기 호출합니다.

```java
@FeignClient(name = "shopping-seller-inventory",
             url = "${feign.shopping-seller-service.url}",
             path = "/internal/inventory")
public interface SellerInventoryClient {

    @PostMapping("/reserve")
    ApiResponse<Void> reserveStock(@RequestBody StockReserveRequest request);

    @PostMapping("/deduct")
    ApiResponse<Void> deductStock(@RequestBody StockReserveRequest request);

    @PostMapping("/release")
    ApiResponse<Void> releaseStock(@RequestBody StockReserveRequest request);

    @PostMapping("/restore")
    ApiResponse<Void> restoreStock(@RequestBody StockReserveRequest request);
}
```

보상 흐름은 별도 포스트(Saga 보상 체계의 8가지 결함과 해결)에서 상세히 다룹니다.

### 서비스 간 인증: X-Internal-Token

외부에서 `/internal/*` API에 접근하는 것을 막기 위해, Feign 요청에 내부 인증 토큰을 자동 첨부합니다.

```java
@Bean
public RequestInterceptor requestInterceptor(
        @Value("${app.internal.token}") String internalToken) {
    return requestTemplate -> {
        requestTemplate.header("X-Internal-Token", internalToken);

        // HTTP 요청 컨텍스트가 있으면 사용자 헤더도 전파
        ServletRequestAttributes attributes = (ServletRequestAttributes)
            RequestContextHolder.getRequestAttributes();
        if (attributes != null) {
            // X-User-Id, X-User-Role 등 전파
        }
    };
}
```

### Kafka — 비동기 통신

결제 완료, 주문 취소 같은 이벤트는 Kafka를 통해 비동기로 전파합니다.

```java
// 결제 완료 → Saga 후속 단계 진행
@KafkaListener(topics = ShoppingTopics.PAYMENT_COMPLETED, groupId = "shopping-service")
public void onPaymentCompleted(PaymentCompletedEvent event) {
    orderService.completeOrderAfterPayment(event.getOrderNumber().toString());
}

// 결제 취소 → 주문 취소 연쇄
@KafkaListener(topics = ShoppingTopics.PAYMENT_CANCELLED, groupId = "shopping-service")
public void onPaymentCancelled(PaymentCancelledEvent event) {
    orderService.cancelOrder(event.getUserId().toString(),
        event.getOrderNumber().toString(),
        new CancelOrderRequest("Payment cancelled: " + event.getCancelReason()));
}
```

### 재고 모델: 3-state 설계

seller-service의 `Inventory` 엔티티는 재고를 세 가지 상태로 관리합니다.

```java
@Entity
public class Inventory {
    private Integer availableQuantity;  // 구매 가능 수량
    private Integer reservedQuantity;   // 예약된 수량 (결제 대기)
    private Integer totalQuantity;      // 전체 수량

    @Version
    private Long version;  // 낙관적 락

    // RESERVE: available -= qty, reserved += qty
    public void reserve(int quantity) { ... }

    // DEDUCT: reserved -= qty, total -= qty (결제 확정)
    public void deduct(int quantity) { ... }

    // RELEASE: reserved -= qty, available += qty (주문 취소)
    public void release(int quantity) { ... }

    // RESTORE: available += qty, total += qty (Saga 보상)
    public void restore(int quantity) { ... }
}
```

`reserve` → `deduct`가 정상 흐름이고, `release`는 결제 전 취소, `restore`는 결제 후 Saga 보상 시 사용합니다. `@Version`으로 낙관적 락을 걸어 동시 주문 시 재고 정합성을 보장합니다.

## sellerId 전파 문제

서비스를 분리하고 나니 예상치 못한 문제가 나타났습니다. 주문 시점에 상품의 판매자 정보(`sellerId`)가 필요한데, 상품 정보는 seller-service에 있고 주문은 shopping-service에서 처리합니다.

매 주문마다 seller-service에 판매자 정보를 조회하는 것은 비효율적이고 장애 전파 위험이 있습니다. 대신 **스냅샷 방식**을 적용했습니다. 장바구니에 상품을 담는 시점에 `sellerId`를 `CartItem`에 저장하고, 주문 생성 시 `OrderItem`으로 복사합니다.

```sql
-- CartItem, OrderItem 테이블에 sellerId 컬럼 추가
ALTER TABLE cart_items ADD COLUMN seller_id BIGINT;
ALTER TABLE order_items ADD COLUMN seller_id BIGINT NOT NULL;
```

이렇게 하면 주문 처리 중 seller-service를 호출할 필요가 없고, 정산 이벤트에도 `sellerId`가 이미 포함되어 있으므로 settlement-service에서 판매자별 정산을 바로 처리할 수 있습니다.

## 정산 이벤트 흐름

settlement-service는 이벤트 소싱 방식의 원장(Ledger) 패턴을 사용합니다.

```java
@KafkaListener(topics = ShoppingTopics.ORDER_SETTLEMENT_CREATED,
               groupId = "shopping-settlement-service")
public void onOrderSettlementCreated(OrderSettlementCreatedEvent event) {
    // 판매자별 금액 합산
    Map<Long, BigDecimal> sellerAmounts = event.getItems().stream()
        .collect(Collectors.groupingBy(
            OrderItemInfo::getSellerId,
            Collectors.reducing(BigDecimal.ZERO,
                item -> item.getPrice().multiply(BigDecimal.valueOf(item.getQuantity())),
                BigDecimal::add)
        ));

    // 판매자별 개별 ledger 생성
    sellerAmounts.forEach((sellerId, amount) ->
        saveLedgerIdempotent(event.getOrderNumber(), sellerId, "PAYMENT_COMPLETED", amount));
}
```

주문 취소 시에는 기존 `PAYMENT_COMPLETED` 레코드의 금액을 역분개(`negate`)합니다. Unique constraint(`order_number + seller_id + event_type`)로 중복 이벤트를 멱등하게 처리합니다.

## 교훈

**1. 변경 주기가 다르면 분리 후보다**

같은 도메인이라도 변경 주기가 다르면 하나의 서비스에 두는 것이 오히려 복잡도를 높입니다. Buyer/Seller/Settlement는 각각 다른 속도로 변화하므로 독립 배포가 가능한 구조가 유리합니다.

**2. 데이터 전파 전략을 미리 결정하라**

서비스를 분리하면 필연적으로 데이터 참조 문제가 생깁니다. `sellerId` 스냅샷처럼 어떤 데이터를 어느 시점에 복제할지 미리 정하지 않으면, 런타임에 서비스 간 호출이 늘어나 결합도가 다시 높아집니다.

**3. 동기/비동기 통신의 경계를 명확히 하라**

재고 예약처럼 즉각 응답이 필요한 것은 Feign, 이벤트 전파는 Kafka. 이 경계가 모호하면 장애 전파 범위를 예측하기 어렵습니다.
