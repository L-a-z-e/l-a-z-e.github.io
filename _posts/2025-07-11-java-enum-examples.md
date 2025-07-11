---
title: Enum, 아직도 상수로만 쓰세요? (Enum을 객체처럼 활용하는 5가지 실전 패턴)
description: 
author: laze
date: 2025-07-11 00:00:02 +0900
categories: [Dev, Java]
tags: [Enum, Java]
---
# [Java] Enum, 아직도 상수로만 쓰세요? (Enum을 객체처럼 활용하는 5가지 실전 패턴)

## 1. 서론: `Enum`에 대한 흔한 오해

많은 Java 개발자들이 `Enum`(열거형)을 처음 배울 때, 그저 "관련 있는 상수들의 묶음" 정도로 이해하고 넘어간다. `public static final int`로 상수를 정의하던 시절의 문제점을 해결해주는, 타입 안전한 상수 집합이라는 것이다.

```java
// 이런 식으로만 사용하고 있다면, Enum의 잠재력을 30%도 활용하지 못하는 것이다.
public enum Status {
    ACTIVE, DORMANT, DELETED;
}
```

틀린 말은 아니지만, 이것은 `Enum`의 거대한 잠재력 중 극히 일부에 불과하다. `Enum`의 본질은 단순한 상수가 아니라, **'상태와 행위를 갖는 타입 안전한 객체'** 이다.

이 글에서는 `Enum`을 단순한 상수 집합을 넘어, 하나의 잘 설계된 '작은 클래스'처럼 활용하여 코드를 얼마나 더 깔끔하고, 안전하며, 객체지향적으로 만들 수 있는지 5가지 실전 패턴을 통해 알아보겠다.

## 패턴 1: 데이터와의 결합 - 속성을 가진 똑똑한 상수

가장 기본적이면서도 강력한 활용법이다. 각 `Enum` 상수가 자신만의 고유한 데이터를 갖게 만드는 것이다.

- **상황:** 애플리케이션의 사용자 등급(BRONZE, SILVER, GOLD)을 관리해야 하며, 각 등급은 고유한 이름과 최소 구매 금액 조건을 가진다.

### Before (데이터와 상수가 분리된 경우)

```java
// 상수는 여기에 있고...
public static final int BRONZE = 1;
public static final int SILVER = 2;
public static final int GOLD = 3;

// 데이터는 다른 곳에서 관리된다.
public String getTierName(int tierCode) {
    if (tierCode == 1) return "브론즈";
    // ...
}
```

데이터와 상수가 분리되어 있어 코드가 지저분하고, 새로운 등급이 추가될 때 여러 곳을 수정해야 한다.

### After (Enum 활용)

`Enum` 내부에 `private final` 필드와 생성자를 선언하여, 각 상수가 자신의 데이터를 직접 소유하게 한다.

```java
public enum UserTier {
    // 각 상수는 생성자를 통해 자신의 데이터를 초기화한다.
    GOLD("골드", 300_000),
    SILVER("실버", 100_000),
    BRONZE("브론즈", 0);

    private final String koreanName;
    private final int minPurchaseAmount;

    // 생성자는 반드시 private 이어야 한다.
    UserTier(String koreanName, int minPurchaseAmount) {
        this.koreanName = koreanName;
        this.minPurchaseAmount = minPurchaseAmount;
    }

    // 외부에서 데이터를 사용할 수 있도록 Getter 제공
    public String getKoreanName() {
        return koreanName;
    }

    public boolean isAchievable(int purchaseAmount) {
        return purchaseAmount >= this.minPurchaseAmount;
    }
}

// 사용처
UserTier myTier = UserTier.GOLD;
System.out.println(myTier.getKoreanName()); // "골드"
System.out.println(myTier.isAchievable(500000)); // true
```

- **효과:** 사용자 등급과 관련된 모든 정보(이름, 조건)와 관련 로직(`isAchievable`)이 `UserTier`라는 하나의 `Enum` 안에 캡슐화되었다. 응집도가 높아지고 유지보수가 매우 용이해졌다.

## 패턴 2: 행위의 캡슐화 - `if/switch`를 제거하는 마법

`Enum` 활용의 정수. 각 `Enum` 상수가 자신만의 고유한 행위(메서드 구현)를 갖게 하여, 외부의 `if/switch` 분기문을 완전히 제거하는 패턴이다. **전략 패턴(Strategy Pattern)**의 완벽한 구현체이기도 하다.

- **상황:** 결제 수단(CARD, CASH, POINT)에 따라 각기 다른 계산 로직을 적용해야 한다.

### Before (지저분한 `switch` 분기문)

```java
public class PaymentService {
    public long calculateFee(PaymentMethod method, long amount) {
        switch (method) {
            case CARD:
                return (long) (amount * 0.015); // 카드 수수료
            case CASH:
                return 0; // 현금은 수수료 없음
            case POINT:
                return (long) (amount * 0.005); // 포인트 사용 수수료
            default:
                throw new IllegalArgumentException("Unsupported payment method");
        }
    }
}

```

새로운 결제 수단이 추가될 때마다 이 `switch` 문을 찾아 수정해야 한다. 이는 OCP(개방-폐쇄 원칙)를 위반한다.

### After (Enum에 책임 위임)

`Enum`에 추상 메서드를 선언하고, 각 상수가 자신만의 방식으로 이를 구현하도록 한다.

```java
public enum PaymentMethod {
    CARD("신용카드") {
        @Override
        public long calculateFee(long amount) {
            return (long) (amount * 0.015);
        }
    },
    CASH("현금") {
        @Override
        public long calculateFee(long amount) {
            return 0;
        }
    },
    POINT("포인트") {
        @Override
        public long calculateFee(long amount) {
            return (long) (amount * 0.005);
        }
    };

    private final String title;
    PaymentMethod(String title) { this.title = title; }
    public String getTitle() { return title; }

    // 각 결제수단이 반드시 구현해야 할 '행위'를 추상 메서드로 선언
    public abstract long calculateFee(long amount);
}

// 사용처 (코드가 놀랍도록 깔끔해진다)
public class PaymentService {
    public long calculateFee(PaymentMethod method, long amount) {
        // 어떤 결제수단이 오든, 그냥 calculateFee()를 호출하면 끝.
        // switch 분기가 완전히 사라졌다!
        return method.calculateFee(amount);
    }
}
```

- **효과:** `PaymentService`는 더 이상 각 결제 수단의 수수료 계산 방법을 알 필요가 없다. 그 책임은 `PaymentMethod` `Enum` 자신에게로 위임되었다. 새로운 결제 수단이 추가되어도 `PaymentService`는 전혀 수정할 필요가 없다. 이것이 바로 OCP를 준수하는 객체지향적인 설계다.

## 패턴 3: 유틸리티 메서드 제공 - 가독성과 편의성 향상

`Enum`과 관련된 편의 기능을 `Enum` 내부에 `static` 메서드로 제공하여, 외부에서 지저분한 코드를 작성하지 않도록 돕는다.

- **상황:** DB에 저장된 숫자 코드 값(`10`, `20`, `30`)을 보고, 해당하는 `UserTier` Enum 상수를 찾아야 한다.

### Before (외부에서 `for` 루프 돌리기)

```java
public UserTier findTierByCode(int tierCode) {
    for (UserTier tier : UserTier.values()) {
        if (tier.getCode() == tierCode) {
            return tier;
        }
    }
    throw new IllegalArgumentException("Invalid tier code");
}
```

이런 코드가 여러 곳에 중복해서 나타날 수 있다.

### After (Enum 내부에 `static` 팩토리 메서드 제공)

```java
public enum UserTier {
    GOLD("골드", 10),
    SILVER("실버", 20),
    BRONZE("브론즈", 30);

    // ... 필드, 생성자 ...
    private final int code;

    // 캐싱을 통해 성능 최적화
    private static final Map<Integer, UserTier> CODE_MAP =
            Collections.unmodifiableMap(Stream.of(values())
                    .collect(Collectors.toMap(UserTier::getCode, Function.identity())));

    // 외부에 제공할 public static 메서드
    public static UserTier fromCode(int code) {
        UserTier result = CODE_MAP.get(code);
        if (result == null) {
            throw new IllegalArgumentException("Invalid tier code: " + code);
        }
        return result;
    }

    public int getCode() { return this.code; }
    // ...
}

// 사용처
UserTier tier = UserTier.fromCode(20); // UserTier.SILVER
```

- **효과:** 코드 변환 로직이 `UserTier` `Enum` 내부로 캡슐화되었다. 외부에서는 `UserTier.fromCode()`만 호출하면 되므로 코드가 깔끔해지고, `static` 초기화 블록에서 `Map`을 만들어 캐싱하면 성능까지 향상시킬 수 있다.

## 패턴 4: 인터페이스 구현 - 다형성을 통한 확장

`Enum`도 일반 클래스처럼 인터페이스를 구현할 수 있다. 이를 통해 `Enum` 그룹 간에 공통된 행위를 정의하고 다형성을 활용할 수 있다.

- **상황:** 정액 할인(`FIXED`)과 정률 할인(`PERCENT`) 정책이 있다. 두 정책 모두 할인 금액을 계산하는 기능이 필요하다.

```java
// 공통 행위를 정의할 인터페이스
public interface DiscountPolicy {
    long applyDiscount(long originalPrice);
}

// 정액 할인 Enum
public enum FixedDiscount implements DiscountPolicy {
    WON_1000(1000),
    WON_5000(5000);

    private final long amount;
    FixedDiscount(long amount) { this.amount = amount; }

    @Override
    public long applyDiscount(long originalPrice) {
        return Math.max(0, originalPrice - amount);
    }
}

// 정률 할인 Enum
public enum PercentDiscount implements DiscountPolicy {
    PERCENT_10(10),
    PERCENT_20(20);

    private final int rate;
    PercentDiscount(int rate) { this.rate = rate; }

    @Override
    public long applyDiscount(long originalPrice) {
        return originalPrice - (originalPrice * rate / 100);
    }
}

// 사용처
public class DiscountService {
    public void applySomeDiscount(DiscountPolicy policy, long price) {
        long finalPrice = policy.applyDiscount(price);
        System.out.println("할인 적용가: " + finalPrice);
    }

    public void test() {
        applySomeDiscount(FixedDiscount.WON_1000, 20000); // 정액 할인 적용
        applySomeDiscount(PercentDiscount.PERCENT_10, 20000); // 정률 할인 적용
    }
}
```

- **효과:** `DiscountService`는 넘어온 `policy`가 `FixedDiscount`인지 `PercentDiscount`인지 전혀 신경 쓸 필요 없이, `DiscountPolicy` 인터페이스의 `applyDiscount` 메서드만 호출하면 된다. 다형성을 통해 코드가 매우 유연하고 확장 가능해진다.

## 패턴 5 (심화): 가장 안전한 싱글턴(Singleton) 만들기

"이펙티브 자바"에서 조슈아 블로크가 추천하는, 싱글턴을 구현하는 가장 좋은 방법이다.

- **상황:** 시스템 전체에서 유일해야 하는 설정 관리자 객체가 필요하다.

```java
public enum SettingsManager {
    INSTANCE; // 원소가 하나뿐인 Enum

    private final String settingValue;

    // 생성자는 JVM에 의해 최초 한 번만 호출됨이 보장된다.
    SettingsManager() {
        System.out.println("SettingsManager 초기화 중...");
        // 파일에서 설정을 읽어오는 등 복잡한 초기화 로직
        this.settingValue = "some-default-value";
    }

    public String getSettingValue() {
        return settingValue;
    }

    public static SettingsManager getInstance() {
        return INSTANCE;
    }
}

// 사용처
SettingsManager.getInstance().getSettingValue();

```

- **효과:** `Enum`은 JVM 레벨에서 직렬화/역직렬화, 리플렉션을 통한 강제 인스턴스화 등 싱글턴을 깨뜨리려는 모든 시도를 원천적으로 방어해준다. 복잡한 `double-checked locking`이나 `static holder` 패턴 없이, 단 몇 줄의 코드로 가장 완벽하고 안전한 싱글턴을 구현할 수 있다.

## 7. 결론: `Enum`은 '작은 클래스'다

`Enum`은 더 이상 단순한 상수 묶음이 아니다. **상태(데이터)와 행위(메서드)를 함께 캡슐화하여, 타입 안정성과 객체지향적 설계를 모두 만족시키는 강력한 도구**다.

코드에서 `if/switch` 분기문이 보이거나, 특정 타입과 관련된 데이터와 로직이 여러 곳에 흩어져 있다면, `Enum`을 '작은 클래스'로 바라보고 리팩토링하는 것을 고려해보자. 당신의 코드는 훨씬 더 안전하고, 유연하며, 가독성 높은 코드로 거듭날 것이다.
