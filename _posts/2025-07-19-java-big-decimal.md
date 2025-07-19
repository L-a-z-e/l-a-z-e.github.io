---
title: BigDecimal - 정확한 계산을 위한 필수 가이드 (Overflow Exception)
description: 
author: laze
date: 2025-07-19 00:00:01 +0900
categories: [Dev, Java]
tags: [Java, BigDecimal]
---
# [Java] `BigDecimal`, 정확한 계산을 위한 필수 가이드 (Overflow Exception)

## 1. 서론: "0.1 + 0.2는 0.3이 아니다"

Java를 처음 배울 때, 많은 개발자들이 아래 코드의 결과를 보고 의아해합니다.

```java
System.out.println(0.1 + 0.2);
// 결과: 0.30000000000000004
```

우리의 상식과 다른 결과가 나오는 이유는, 컴퓨터가 `double`이나 `float` 같은 부동소수점 타입으로 10진수 소수를 2진수로 변환하면서 발생하는 미세한 오차 때문입니다.

일반적인 계산에서는 문제가 되지 않지만, 단 1원의 오차도 용납할 수 없는 금융, 결제, 정산 시스템에서는 이 작은 차이가 치명적인 버그를 유발합니다.

이 문제를 해결하기 위해 Java는 `BigDecimal`이라는 강력한 클래스를 제공합니다.

`BigDecimal`은 숫자를 10진수 형태로 정확하게 표현하여 오차 없는 계산을 보장합니다.

하지만 그 강력함만큼이나 내부 동작 원리(`precision`, `scale`, `MathContext`)를 제대로 이해하지 못하면, 예상치 못한 예외를 마주하게 됩니다.

이 글에서는 `BigDecimal`의 핵심 개념부터, 실무에서 흔히 겪는 `Overflow Exception` 해결 과정, 그리고 반드시 지켜야 할 Best Practice까지 심층적으로 알아보겠습니다.

## 2. `BigDecimal`의 내부 구조: `precision`과 `scale`의 이해

`BigDecimal`은 숫자를 세 가지 핵심 요소로 표현합니다.

- `unscaledValue`: 소수점을 제거한 순수한 정수 (`BigInteger`).
- `scale`: 소수점의 위치. 0 또는 양의 정수 (소수점 이하 자릿수).
- `precision`: 숫자의 전체 유효 자릿수 (`unscaledValue`의 길이).

이 세 가지 개념을 이해하는 것이 `BigDecimal`을 정복하는 첫걸음입니다.

**예시: `new BigDecimal("123.45")`**

- `unscaledValue`: `12345`
- `scale`: `2` (소수점 이하 자릿수가 2개)
- `precision`: `5` (숫자 '12345'의 총 길이)

데이터베이스의 숫자 타입, 예를 들어 `Oracle NUMBER(21, 2)`는 `precision <= 21`, `scale <= 2` 라는 제약 조건을 의미합니다. 즉, 전체 길이는 21자리를 넘을 수 없고, 소수점 이하는 2자리를 넘을 수 없다는 뜻입니다.

## 3. 연산의 함정과 `precision` 폭증의 비밀

`BigDecimal`의 사칙연산은 우리가 아는 상식과 조금 다르게 동작하며, 이것이 바로 `precision`이 예측 불가능하게 증가하는 원인입니다.

- **`add()` / `subtract()`:** 두 숫자의 `scale` 중 더 큰 쪽으로 결과의 `scale`이 맞춰집니다. `precision`은 일반적으로 크게 변하지 않습니다.
- **`multiply(BigDecimal multiplicand)`:**
  - `result.scale() = this.scale() + multiplicand.scale()`
  - `precision`은 예측이 매우 어려우며, 최대 `this.precision() + multiplicand.precision()`까지 폭발적으로 증가할 수 있습니다. 이것이 바로 오버플로우의 주범입니다.
- **`divide(BigDecimal divisor)`:**
  - `10.divide(3)`처럼 나누어 떨어지지 않는 경우, 무한 소수가 발생하여 `ArithmeticException`이 발생합니다.
  - 따라서 `divide` 연산은 **반드시 `scale`과 반올림 정책(`RoundingMode`)을 함께 지정**해야 합니다.

      ```java
      BigDecimal a = new BigDecimal("10");
      BigDecimal b = new BigDecimal("3");
      // a.divide(b); // ArithmeticException 발생!
      
      // 반드시 scale과 RoundingMode를 지정해야 한다.
      BigDecimal result = a.divide(b, 10, RoundingMode.HALF_UP); // 소수점 10자리까지 반올림
      ```


## 4. `precision`을 제어하는 유일한 도구: `MathContext`

"계산 중간 결과물의 `precision`이 얼마나 커질지 모르겠다"는 통제 불능의 상황을 해결하기 위해 `MathContext`가 존재합니다.

`MathContext`는 **"이 연산의 결과물은 최대 몇 자리(precision)를 넘지 않도록 하고, 만약 넘는다면 이런 반올림 정책(RoundingMode)을 적용해라"** 라고 지시하는 '계산 규칙' 객체입니다.

```java
// 최대 정밀도 50, 반올림 정책은 HALF_UP으로 설정
private static final MathContext MC_50 = new MathContext(50, RoundingMode.HALF_UP);

BigDecimal veryLargeNumber = new BigDecimal("12345678901234567890"); // precision 20
BigDecimal anotherLargeNumber = new BigDecimal("98765432109876543210"); // precision 20

// MathContext 없이 곱하면 precision이 40에 가깝게 증가한다.
BigDecimal result = veryLargeNumber.multiply(anotherLargeNumber);

// MathContext를 적용하면 결과물의 precision이 50으로 제어된다.
BigDecimal controlledResult = veryLargeNumber.multiply(anotherLargeNumber, MC_50);
```

## 5. 실전 Best Practice: `Overflow Exception` 해결 여정

대용량 금융 데이터를 처리하다 보면, `multiply`나 `pow` 연산 과정에서 `precision`이 DB 컬럼의 한계를 초과하여 `Overflow Exception`이 발생하는 경우가 많습니다.

### 1단계 (잘못된 접근): `.setScale()`만으로 해결하려는 시도

많은 개발자들이 오버플로우의 원인을 `scale`의 문제로 착각하고, 계산 마지막에 `.setScale(2, ...)`를 호출하여 문제를 해결하려 합니다. 하지만 이것은 실패합니다.

`.setScale()`은 모든 계산이 끝난 후의 **'후처리 포맷팅'** 도구일 뿐, **계산 과정에서 이미 `precision`이 폭증하는 것을 막을 수는 없기 때문**입니다.

### 2단계 (올바른 접근): "계산은 `MathContext`로, 저장은 `.setScale()`으로"

안전한 금융 데이터 처리의 핵심 전략은 두 단계로 나뉩니다.

1. **안전한 계산 (using `MathContext`)**:
  - `multiply`, `divide`, `pow` 같이 `precision`에 영향을 주는 모든 핵심 연산을 수행할 때, **`MathContext`** 객체를 함께 사용하여 결과물의 최대 `precision`을 명시적으로 제어합니다. 이를 통해 계산 중간값이 절대 예측 범위를 벗어나지 않도록 보장합니다.
2. **정확한 포맷팅 (using `.setScale()`)**:
  - 모든 계산이 안전하게 끝난 후, 최종적으로 DB에 저장하거나 DTO 필드에 값을 설정하기 **바로 직전**에, `.setScale()`을 호출하여 결과물의 소수점 자릿수를 DB 컬럼의 요구사항(`scale=2` 등)에 정확하게 맞춰줍니다.

### 최종 해결 코드 예시

```java
public class FinancialCalculator {

    // 계산 시 사용할 공통 MathContext (DB precision보다 넉넉하게 설정)
    private static final MathContext CALCULATION_CONTEXT = new MathContext(30, RoundingMode.HALF_UP);

    public BigDecimal calculateInterest(BigDecimal principal, BigDecimal rate, int years) {
        // 1. 안전한 계산: pow, multiply 연산에 MathContext 적용
        BigDecimal rateFactor = BigDecimal.ONE.add(rate);
        BigDecimal powerOfRate = rateFactor.pow(years, CALCULATION_CONTEXT);
        BigDecimal calculatedValue = principal.multiply(powerOfRate, CALCULATION_CONTEXT);

        // 2. 정확한 포맷팅: 최종 결과를 DB 컬럼의 scale(예: 2)에 맞춰줌
        BigDecimal finalFormattedValue = calculatedValue.setScale(2, RoundingMode.HALF_UP);

        return finalFormattedValue;
    }
}
```

## 6. 생성자의 함정: `new BigDecimal(double)` vs. `BigDecimal.valueOf(double)`

`double` 타입의 값을 `BigDecimal`로 변환할 때, 어떤 생성자를 사용하느냐에 따라 결과가 완전히 달라집니다.

- **`new BigDecimal(0.1)`:** `double` 타입 `0.1`은 이미 메모리에 `0.1000...00555...` 와 같은 근사치로 저장되어 있습니다. 이 생성자는 이 **부정확한 값을 그대로** `BigDecimal`로 만듭니다. **절대 사용하면 안 됩니다.**
- **`BigDecimal.valueOf(0.1)`:** 이 정적 팩토리 메서드는 내부적으로 `double` 값을 문자열(`"0.1"`)로 변환한 뒤, `new BigDecimal("0.1")`을 호출합니다. 10진수 표현을 정확하게 변환해주므로 안전합니다.

**결론: `double`이나 `float`을 `BigDecimal`로 변환할 때는, 예측 불가능한 오차를 피하기 위해 반드시 `BigDecimal.valueOf()`나 `new BigDecimal("문자열")`을 사용해야 합니다.**

## 7. 결론: `BigDecimal` 사용을 위한 체크리스트

`BigDecimal`은 정확하지만, 올바르게 사용하지 않으면 오히려 더 큰 문제를 일으킬 수 있습니다. 금융 계산 코드를 작성할 때, 아래 체크리스트를 항상 기억합시다.

- [ ]  **생성:** `String` 생성자나 `valueOf()` 팩토리 메서드를 사용했는가? (`new BigDecimal(double)`은 피했는가?)
- [ ]  **나눗셈:** `divide()` 연산에 `scale`과 `RoundingMode`를 명시적으로 지정했는가?
- [ ]  **곱셈/거듭제곱:** `multiply()`, `pow()` 등 `precision`이 증가할 수 있는 연산에 `MathContext`를 사용하여 결과물의 정밀도를 제어했는가?
- [ ]  **최종 저장:** DB에 저장하거나 외부에 노출하기 직전, `.setScale()`을 사용하여 최종 포맷을 맞췄는가?

이 원칙들을 지킨다면, `BigDecimal`을 통해 안전하고 신뢰성 높은 금융 애플리케이션을 구축할 수 있을 것입니다.
