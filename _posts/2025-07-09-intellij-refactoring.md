---
title: IntelliJ 리팩토링 완전 정복 - 단축키부터 실전 워크플로우까지 (Mac/Windows)
description: 
author: laze
date: 2025-07-09 00:00:01 +0900
categories: [Dev, IntelliJ]
tags: [IDE, IntelliJ, Refactoring]
---
# IntelliJ 리팩토링 완전 정복 - 단축키부터 실전 워크플로우까지 (Mac/Windows)

## 1. 서론: 왜 리팩토링을 '도구'와 함께해야 하는가?

리팩토링(Refactoring)이란 "소프트웨어의 겉보기 동작은 그대로 유지한 채, 코드의 내부 구조를 개선하여 더 이해하기 쉽고, 수정하기 쉽게 만드는 과정"을 의미한다. 이는 버그 수정이나 기능 추가와는 다른, 코드의 건강을 유지하는 필수적인 활동이다.

많은 개발자들이 리팩토링을 코드를 복사/붙여넣기하며 수동으로 진행한다. 하지만 이는 메서드 이름 변경을 누락하거나, 로직을 잘못 옮기는 등 새로운 버그를 유발할 위험이 크다. IntelliJ와 같은 현대적인 IDE는 이 모든 과정을 **안전하게(Safely)** 자동화하여, 개발자가 오직 '구조 개선'이라는 본질에만 집중할 수 있도록 돕는다.

## 2. "이럴 땐 이 단축키!" - Code Smell 기반 핵심 리팩토링 TOP 5

좋은 리팩토링의 시작은 'Code Smell'을 감지하는 것이다. Code Smell이란, 코드가 당장 버그는 아니지만, 잠재적으로 문제를 일으킬 수 있거나 구조적으로 좋지 않다는 신호를 의미한다.

각 Code Smell에 맞는 IntelliJ의 처방전을 단축키와 함께 알아보자.

| 단축키 (Mac) | 단축키 (Windows/Linux) | 기능 |
| --- | --- | --- |
| `⌘ + ⌥ + M` | `Ctrl + Alt + M` | **Extract Method** (메서드 추출) |
| `⌘ + ⌥ + V` | `Ctrl + Alt + V` | **Extract Variable** (변수 추출) |
| `⌘ + ⌥ + C` | `Ctrl + Alt + C` | **Extract Constant** (상수 추출) |
| `⌘ + ⌥ + F` | `Ctrl + Alt + F` | **Extract Field** (필드 추출) |
| `⌘ + ⌥ + P` | `Ctrl + Alt + P` | **Extract Parameter** (파라미터 추출) |

### 2.1. Code Smell: 메서드가 너무 길다 (Long Method)

하나의 메서드가 주석으로 구분된 여러 단계의 작업을 수행하고 있다면, 이는 명백한 'Long Method' Code Smell이다.

**처방전: `Extract Method`**

- **방법:** 논리적인 코드 블록을 선택하고 단축키를 누른다.
- **효과:** IntelliJ가 선택된 블록이 사용하는 변수와 반환 값을 분석하여 완벽한 메서드 시그니처를 자동으로 생성해준다. 코드의 가독성이 극적으로 향상되고, 각 메서드는 단일 책임 원칙(SRP)에 가까워진다.

```java
// Before: 너무 많은 일을 하는 메서드
public void processOrder(Order order) {
    // 1단계: 유효성 검사
    if (order == null || order.getItems().isEmpty()) {
        throw new IllegalArgumentException("Order is invalid");
    }
    // ... 다른 로직들 ...
}

// After: 블록 선택 후 ⌘ + ⌥ + M
public void processOrder(Order order) {
    validateOrder(order); // 훨씬 명확해짐
    // ... 다른 로직들 ...
}

private void validateOrder(Order order) {
    if (order == null || order.getItems().isEmpty()) {
        throw new IllegalArgumentException("Order is invalid");
    }
}
```

### 2.2. Code Smell: 표현식이 복잡하다 (Complex Expression)

`if`문이나 stream 연산 등이 너무 길고 복잡해서 한눈에 의도가 파악되지 않는 경우다.

**처방전: `Extract Variable`**

- **방법:** 복잡한 표현식 부분을 선택하고 단축키를 누른다.
- **효과:** 표현식의 결과가 의미 있는 이름의 변수(Explanatory Variable)로 추출된다. 코드는 '무엇을' 하는지가 아니라 '왜' 하는지를 설명하게 된다.

```java
// Before: 조건문의 의도를 파악하기 어렵다
if ((user.getRole().equals("ADMIN") || user.getPoint() > 1000) && user.isActive()) {
    // ...
}

// After: isEligibleForPromotion 이라는 변수명 자체가 설명이 된다
boolean isEligibleForPromotion = (user.getRole().equals("ADMIN") || user.getPoint() > 1000) && user.isActive();
if (isEligibleForPromotion) {
    // ...
}
```

### 2.3. Code Smell: 마법의 숫자/문자열 (Magic Number/String)

코드 곳곳에 `0.9`나 `"PENDING"`처럼 의미를 알 수 없는 값들이 하드코딩되어 있는 경우다.

**처방전: `Extract Constant`**

- **방법:** 해당 값을 선택하고 단축키를 누른다.
- **효과:** 값이 `private static final` 상수로 추출되고, 모든 사용처가 이 상수를 참조하도록 변경된다. 값의 의미가 명확해지고, 향후 변경이 필요할 때 한 곳만 수정하면 된다.

```java
// Before
BigDecimal discountedPrice = originalPrice.multiply(new BigDecimal("0.9")); // 0.9가 뭘까?

// After
private static final BigDecimal SALE_DISCOUNT_RATE = new BigDecimal("0.9");
BigDecimal discountedPrice = originalPrice.multiply(SALE_DISCOUNT_RATE);
```

### 2.4. Code Smell: 데이터 뭉치 (Data Clumps)

여러 메서드에서 동일한 값(e.g., DB 접속 정보)을 반복적으로 정의해서 사용하는 경우다.

**처방전: `Extract Field`**

- **방법:** 지역 변수 선언부를 선택하고 단축키를 누른다.
- **효과:** 지역 변수가 클래스의 멤버 변수(필드)로 승격된다. 객체의 '상태'로서 관리되며, 코드 중복이 제거된다.

```java
// Before
public class ApiClient {
    public void getUsers() {
        String baseUrl = "<https://api.example.com>";
        // ... baseUrl 사용
    }
    public void getPosts() {
        String baseUrl = "<https://api.example.com>";
        // ... baseUrl 사용
    }
}

// After
public class ApiClient {
    private final String baseUrl = "<https://api.example.com>"; // 필드로 승격

    public void getUsers() {
        // ... baseUrl 사용
    }
    public void getPosts() {
        // ... baseUrl 사용
    }
}
```

### 2.5. Code Smell: 경직된 메서드 (Inflexible Method)

메서드 내부의 특정 값이 하드코딩되어 있어, 다른 값으로 재사용하기 어려운 경우다.

**처방전: `Extract Parameter`**

- **방법:** 메서드 내부에서 파라미터로 바꾸고 싶은 값을 선택하고 단축키를 누른다.
- **효과:** 선택한 값이 메서드의 파라미터로 추출되고, 기존 호출부에 해당 값이 인자로 전달되도록 코드가 자동 수정된다. 메서드의 재사용성이 높아진다.

```java
// Before: "admin@example.com" 으로만 보낼 수 있다
public void sendNotification() {
    emailService.send("admin@example.com", "System Alert");
}

// After: 어떤 주소로든 보낼 수 있게 됨
public void sendNotification(String recipientEmail) {
    emailService.send(recipientEmail, "System Alert");
}
```

## 3. 리팩토링 실전 워크플로우: 기능 연계하기

실제 리팩토링은 하나의 기능을 사용하는 것으로 끝나지 않는다.

여러 기능을 연계하여 점진적으로 코드를 개선해 나가야 한다.

아래 `OrderService`의 `processOrder` 메서드를 리팩토링하는 과정을 따라가 보자.

### Before: 모든 로직이 뭉쳐있는 코드

```java
public class OrderService {
    public void processOrder(Order order) {
        if (order == null || order.getItems().isEmpty()) { throw new IllegalArgumentException("Order is invalid"); } // 유효성 검사

        BigDecimal total = BigDecimal.ZERO; // 총액 계산
        for (OrderItem item : order.getItems()) {
            total = total.add(item.getPrice().multiply(new BigDecimal(item.getQuantity())));
        }

        if (order.getCoupon() != null) { // 할인 적용
            total = total.multiply(new BigDecimal("0.9")); // 10% 할인
        }
        orderRepository.save(order);
    }
}
```

### 리팩토링 과정

1. **각 논리적 단위를 `Extract Method`(`⌘ + ⌥ + M`)로 분리한다.**
  - 유효성 검사 블록 선택 → `validateOrder(order)`
  - 총액 계산 블록 선택 → `calculateTotalAmount(order)`
  - 할인 적용 블록 선택 → `applyDiscount(order, total)`
2. **'마법의 숫자' `0.9`를 `Extract Constant`(`⌘ + ⌥ + C`)로 상수로 만든다.**
  - `0.9` 선택 → `SALE_DISCOUNT_RATE`
3. **메서드 이름이 마음에 안 들면 `Rename`(`⇧ + F6`)으로 즉시 변경한다.**
  - `applyDiscount` 메서드에 커서를 놓고 `⇧ + F6` → `applyCouponDiscountIfApplicable` 처럼 더 명확한 이름으로 변경. IntelliJ가 호출부까지 모두 안전하게 바꿔준다.
4. **(심화) 책임이 다른 코드를 다른 클래스로 `Move`(`F6`)한다.**
  - `calculateTotalAmount`, `applyCouponDiscountIfApplicable` 등 가격 계산 관련 메서드들이 `OrderService`보다 `PriceCalculator`라는 별도 클래스에 있는 것이 더 적합해 보인다.
  - `PriceCalculator` 클래스를 새로 만들고, 관련 메서드들을 선택한 후 `F6`을 눌러 새 클래스로 이동시킨다.

### After: 역할과 책임이 분리된 코드

```java
public class OrderService {
    private final PriceCalculator priceCalculator;
    // ... 생성자
    public void processOrder(Order order) {
        validateOrder(order);
        BigDecimal finalPrice = priceCalculator.calculateFinalPrice(order);
        order.setFinalAmount(finalPrice);
        orderRepository.save(order);
    }
    private void validateOrder(Order order) { /* ... */ }
}

public class PriceCalculator {
    private static final BigDecimal SALE_DISCOUNT_RATE = new BigDecimal("0.9");
    public BigDecimal calculateFinalPrice(Order order) {
        BigDecimal total = calculateTotalAmount(order);
        total = applyCouponDiscountIfApplicable(order, total);
        return total;
    }
    private BigDecimal calculateTotalAmount(Order order) { /* ... */ }
    private BigDecimal applyCouponDiscountIfApplicable(Order order, BigDecimal total) { /* ... */ }
}
```

## 4. 반드시 알아야 할 만능 단축키

모든 단축키를 외울 필요는 없다. 이 세 가지만 알아도 대부분의 리팩토링이 가능하다.

| 단축키 (Mac) | 단축키 (Windows/Linux) | 기능 | 설명 |
| --- | --- | --- | --- |
| `⌃ + T` | `Ctrl + Alt + Shift + T` | **Refactor This** | **만능 팝업.** 현재 커서 위치에서 가능한 모든 리팩토링 목록을 보여준다. **단축키가 기억나지 않을 때 최고의 선택.** |
| `⇧ + F6` | `Shift + F6` | **Rename** | 변수, 메서드, 클래스, 파일명 등 **모든 것의 이름을 안전하게 변경**한다. 가장 많이 사용하는 기능. |
| `⌘ + ⌥ + N` | `Ctrl + Alt + N` | **Inline** | **`Extract`의 반대.** 너무 잘게 쪼개진 메서드나 변수를 다시 코드로 합친다. 과한 리팩토링을 되돌릴 때 유용하다. |

## 5. 결론: 습관이 실력을 만든다

IntelliJ의 리팩토링 기능은 단순히 편리한 도구가 아니다. 코드의 구조를 안전하게 개선하고, 유지보수 비용을 낮추며, 동료와의 협업을 원활하게 만드는 핵심적인 기술이다.
처음에는 마우스 우클릭으로 시작하더라도, 의식적으로 단축키를 사용하려 노력해보자. 리팩토링을 '나중에 할 일'로 미루지 않고, 코드를 작성하는 모든 순간에 함께하는 '습관'으로 만들 때, 당신의 코드 품질과 개발 생산성은 한 차원 높은 수준에 도달할 것이다.
