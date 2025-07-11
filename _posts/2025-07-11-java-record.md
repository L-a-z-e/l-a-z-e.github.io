---
title: Record의 본질과 5가지 실전 활용 패턴
description: 
author: laze
date: 2025-07-11 00:00:03 +0900
categories: [Dev, Java]
tags: [Java, Record]
---
# [Java] `Record`, DTO에만 쓰시나요? (Record의 본질과 5가지 실전 활용 패턴)

## 1. 서론: DTO 작성을 위한 고통의 나날들

Java 개발자라면 누구나 계층 간 데이터 전달을 위한 DTO(Data Transfer Object)를 만들어 본 경험이 있을 것이다. 단순히 데이터를 '담아서 전달'하는 목적의 객체를 만들기 위해, 우리는 다음과 같은 수많은 보일러플레이트(boilerplate) 코드를 반복적으로 작성해야 했다.

### Before: 전통적인 DTO 클래스

```java
// UserData.java
import java.util.Objects;

// 불변 객체를 만들기 위한 수많은 장치들...
public final class UserData {
    private final String name;
    private final int age;

    public UserData(String name, int age) {
        this.name = name;
        this.age = age;
    }

    public String getName() { return name; }
    public int getAge() { return age; }

    @Override
    public boolean equals(Object o) {
        if (this == o) return true;
        if (o == null || getClass() != o.getClass()) return false;
        UserData userData = (UserData) o;
        return age == userData.age && Objects.equals(name, userData.name);
    }

    @Override
    public int hashCode() {
        return Objects.hash(name, age);
    }

    @Override
    public String toString() {
        return "UserData[name='" + name + "', age=" + age + ']';
    }
}
```

Lombok 라이브러리가 이 고통을 일부 덜어주었지만, Java 14부터는 언어 자체에서 이 문제를 해결하는 우아한 방법을 제공한다. 바로 `Record`다.

### After: `Record`를 사용한 DTO

```java
public record UserData(String name, int age) {}
```

이 단 한 줄의 코드는 위에서 작성한 모든 보일러플레이트 코드를 컴파일 시점에 완벽하게 자동으로 생성해준다.

이 글에서는 `Record`가 단순히 코드를 줄여주는 문법 설탕을 넘어, 어떻게 우리의 코드를 더 안전하고 의도가 명확하게 만드는지 5가지 실전 패턴을 통해 알아보겠다.

## 2. `Record`의 본질: "나는 불변 데이터 운반자입니다"

`Record`는 컴파일러에게 **"이 클래스의 유일한 목적은 여러 필드의 데이터를 불변(immutable)하게 캡슐화하여 전달하는 것입니다"** 라고 알려주는 명시적인 선언이다. 이 선언 하나로 컴파일러는 다음 요소들을 자동으로 생성한다.

- 모든 필드에 대한 `private final` 선언.
- 모든 필드를 초기화하는 `public` 생성자 (Canonical Constructor).
- 각 필드에 대한 public `getter` (메서드 이름이 `getX()`가 아닌 필드 이름과 동일한 `name()`, `age()` 형식).
- 모든 필드의 값을 비교하는 `equals()` 구현.
- 모든 필드의 값을 기반으로 생성하는 `hashCode()` 구현.
- 모든 필드의 값을 보기 좋게 출력하는 `toString()` 구현.

`Record`의 가장 중요한 철학은 **불변성(Immutability)**이다. 한번 생성된 `Record` 객체의 상태는 절대 변하지 않는다.

이 불변성 덕분에 `Record` 객체는 멀티스레드 환경에서 동기화 없이도 안전하게 공유할 수 있으며(Thread-safe), 예측 가능한 동작을 보장한다.

## 3. 언제 `Record`를 써야 할까? 5가지 실전 활용 패턴

`Record`는 '데이터 운반'이라는 역할이 명확한 모든 곳에 이상적이다.

### 패턴 1: DTO (Data Transfer Object)

가장 기본적이고 널리 알려진 용도다. Controller, Service, Repository 등 계층 간 데이터를 전달할 때 `Record`를 사용하면 코드가 매우 간결해진다.

- **상황:** 클라이언트에게 API 응답을 보낼 때, 응답 상태와 데이터를 함께 담아 전달하고 싶다.

```java
// 제네릭을 활용하여 재사용 가능한 API 응답 DTO를 만들 수 있다.
public record ApiResponse<T>(
    boolean success,
    T data,
    String message
) {
    // 정적 팩토리 메서드를 추가하여 편의성을 높일 수 있다.
    public static <T> ApiResponse<T> ok(T data) {
        return new ApiResponse<>(true, data, null);
    }

    public static <T> ApiResponse<T> error(String message) {
        return new ApiResponse<>(false, null, message);
    }
}

// Controller에서 사용 예시
@GetMapping("/users/{id}")
public ApiResponse<UserDto> getUser(@PathVariable Long id) {
    UserDto user = userService.findById(id);
    return ApiResponse.ok(user);
}
```

### 패턴 2: 복합 `Map` 키 (Composite Map Key)

`HashMap`의 키는 불변 객체여야 한다. 두 개 이상의 필드를 조합하여 `Map`의 키로 사용해야 할 때, `Record`는 완벽한 해결책이다. `equals()`와 `hashCode()`가 자동으로 구현되기 때문이다.

- **상황:** (사용자 ID, 상품 ID)의 조합으로 구매 횟수를 카운팅해야 한다.

```java
public record UserProductKey(Long userId, Long productId) {}

public class PurchaseService {
    private final Map<UserProductKey, Integer> purchaseCountMap = new HashMap<>();

    public void recordPurchase(Long userId, Long productId) {
        UserProductKey key = new UserProductKey(userId, productId);
        purchaseCountMap.merge(key, 1, Integer::sum);
    }
}
```

`UserProductKey` `Record` 덕분에, 우리는 복잡한 `equals/hashCode` 구현 없이도 두 개의 ID 조합을 `Map`의 키로 안전하게 사용할 수 있다.

### 패턴 3: 메서드 다중 값 반환 (Multiple Return Values)

메서드에서 여러 값을 반환해야 할 때, 길이가 2인 배열이나 `Map`을 사용하는 것은 타입 안전성을 해치고 코드의 의도를 모호하게 만든다. `Record`를 사용하면 반환 값들의 의미를 명확히 할 수 있다.

- **상황:** 숫자 리스트에서 최소값과 최대값을 동시에 찾아 반환해야 한다.

```java
public record MinMaxResult(int min, int max) {}

public class StatUtils {
    public MinMaxResult findMinMax(List<Integer> numbers) {
        if (numbers == null || numbers.isEmpty()) {
            throw new IllegalArgumentException("List cannot be empty");
        }
        int min = Collections.min(numbers);
        int max = Collections.max(numbers);
        return new MinMaxResult(min, max);
    }
}

// 사용처
MinMaxResult result = StatUtils.findMinMax(List.of(5, 1, 9, 3, 8));
System.out.println("Min: " + result.min() + ", Max: " + result.max());
```

### 패턴 4: 간단한 이벤트/메시지 객체 (Event/Message Object)

메시징 큐(Kafka, RabbitMQ)나 Spring의 이벤트 시스템에서 사용되는, 상태가 변하지 않는 데이터 객체를 정의할 때 `Record`는 매우 유용하다.

- **상황:** 주문이 완료되었을 때, `OrderPlacedEvent`라는 이벤트를 발행해야 한다.

```java
import java.time.Instant;
import java.util.List;

public record OrderPlacedEvent(
    Long orderId,
    Long userId,
    List<Long> productIds,
    Instant eventTimestamp
) {}

// 이벤트 발행기
@Component
public class OrderEventPublisher {
    @Autowired
    private ApplicationEventPublisher publisher;

    public void publishOrderPlaced(Order order) {
        OrderPlacedEvent event = new OrderPlacedEvent(
            order.getId(),
            order.getUserId(),
            order.getProductIds(),
            Instant.now()
        );
        publisher.publishEvent(event);
    }
}
```

### 패턴 5: Stream API와의 환상적인 궁합

Stream API의 중간 연산 과정에서 여러 데이터를 임시로 묶어 처리해야 할 때, 메서드 내부에 `local record`를 선언하면 매우 깔끔한 코드를 작성할 수 있다.

- **상황:** 사용자 목록에서, 각 도시별 평균 연령을 계산해야 한다.

```java
public Map<String, Double> calculateAverageAgeByCity(List<User> users) {
    // 메서드 내부에만 사용될 임시 데이터 묶음을 위한 local record 선언
    record UserCityAndAge(String city, int age) {}

    return users.stream()
            .map(user -> new UserCityAndAge(user.getAddress().getCity(), user.getAge()))
            .collect(Collectors.groupingBy(
                    UserCityAndAge::city,
                    Collectors.averagingInt(UserCityAndAge::age)
            ));
}
```

`UserCityAndAge`라는 `local record` 덕분에, `groupingBy` 연산이 훨씬 더 명확하고 타입 안전해졌다.

## 4. `Record` 심화 학습: 알아두면 좋은 점들

- **Compact Constructor:** 생성자에서 유효성 검사를 추가하고 싶을 때, 아래와 같이 간결한 생성자를 사용할 수 있다.

    ```java
    public record UserData(String name, int age) {
        public UserData { // Compact Constructor
            if (age < 0) {
                throw new IllegalArgumentException("Age cannot be negative");
            }
        }
    }
    ```

- **인터페이스 구현:** `Record`도 클래스이므로 인터페이스를 구현할 수 있다. `implements Serializable`이 대표적인 예다.
- **언제 쓰면 안되나?:** `Record`는 불변 객체이므로, 상태 변경이 필요한 객체에는 적합하지 않다. 대표적으로 JPA의 `@Entity` 어노테이션이 붙는 클래스는 `Record`로 만들 수 없다. 엔티티는 프록시 객체 생성 등 내부적인 메커니즘을 위해 `final`이 아니어야 하고, 기본 생성자가 필요하기 때문이다.

## 5. 결론: `Record`로 코드의 '의도'를 명확히 하라

`Record`는 단순히 코드를 줄여주는 편리한 기능을 넘어, **"이 객체는 불변 데이터를 담는 역할만 수행한다"**는 개발자의 의도를 코드 자체로 표현하는 강력한 도구다.

계층 간 데이터 전달, 복합 키, 다중 값 반환 등 역할이 명확한 곳에 `Record`를 적극적으로 사용해 보자. 당신의 코드는 더 안전하고, 더 간결하며, 더 가독성 높은 코드로 거듭날 것이다.
