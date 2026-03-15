---
title: "Spring @Cacheable의 self-invocation 함정과 캐시 컴포넌트 분리"
description: "같은 클래스 내에서 @Cacheable 메서드를 호출하면 캐시가 동작하지 않는 문제의 원인과, 캐시 전담 컴포넌트 분리로 해결한 과정에 대한 기록"
author: laze
date: 2026-03-12 10:00:00 +0900
categories: [Dev, SpringBoot]
tags: [SpringBoot, Cache, AOP]
---

## 문제 상황

auth-service의 RBAC(Role-Based Access Control) 시스템에서 역할 계층(Role Hierarchy)을 Redis에 캐시하려고 했습니다. 역할 계층 그래프는 자주 변경되지 않지만 매 요청마다 DB를 조회하면 비효율적이므로, `@Cacheable`로 캐시하는 것이 자연스러운 선택이었습니다.

처음 시도한 구조는 `RoleHierarchyService` 안에서 `@Cacheable` 메서드를 정의하고, 같은 클래스의 다른 메서드에서 호출하는 방식이었습니다.

```java
@Service
@RequiredArgsConstructor
public class RoleHierarchyService {

    private final RoleIncludeRepository roleIncludeRepository;

    @Cacheable(value = "roleHierarchy", key = "'graph'")
    public Map<Long, Set<Long>> getHierarchyGraph() {
        // DB에서 역할 계층 조회
        return roleIncludeRepository.findAll().stream()
            .collect(Collectors.groupingBy(...));
    }

    public Set<Long> resolveEffectiveRoles(Set<Long> directRoleIds) {
        Map<Long, Set<Long>> graph = getHierarchyGraph(); // self-invocation
        // BFS로 모든 유효 역할 계산
        // ...
    }
}
```

캐시가 동작하지 않았습니다. `getHierarchyGraph()`를 호출할 때마다 DB 쿼리가 실행되었습니다.

## 근본 원인: Spring AOP Proxy의 한계

Spring의 `@Cacheable`은 AOP 프록시 기반으로 동작합니다. 외부에서 Bean의 메서드를 호출하면 프록시를 경유하면서 캐시 인터셉터가 개입하지만, 같은 클래스 내부에서 `this.getHierarchyGraph()`를 호출하면 프록시를 거치지 않고 실제 객체의 메서드가 직접 호출됩니다.

```
외부 호출: caller → Proxy(캐시 확인) → 실제 메서드    ✅ 캐시 동작
내부 호출: this.getHierarchyGraph()  → 실제 메서드    ❌ 캐시 무시
```

이것이 Spring AOP의 self-invocation 문제입니다. `@Transactional`, `@Async` 등 모든 AOP 기반 어노테이션에 동일하게 적용됩니다.

## 검토한 우회 방법

### 1. @Lazy self-injection

자기 자신을 `@Lazy`로 주입받아 프록시를 경유하도록 하는 방법입니다.

```java
@Service
public class RoleHierarchyService {

    @Lazy
    @Autowired
    private RoleHierarchyService self;

    @Cacheable(value = "roleHierarchy", key = "'graph'")
    public Map<Long, Set<Long>> getHierarchyGraph() { ... }

    public Set<Long> resolveEffectiveRoles(Set<Long> directRoleIds) {
        Map<Long, Set<Long>> graph = self.getHierarchyGraph(); // 프록시 경유
    }
}
```

기술적으로는 동작하지만, 두 가지 문제가 있었습니다.

- **Lombok `@RequiredArgsConstructor`와 비호환**: `@Autowired` 필드 주입을 사용해야 하므로 생성자 주입 패턴이 깨짐
- **순환 참조의 냄새**: 자기 자신을 주입받는 것은 설계상 어색하고, 코드를 처음 보는 사람이 의도를 파악하기 어려움

### 2. ApplicationContext에서 직접 Bean 조회

```java
applicationContext.getBean(RoleHierarchyService.class).getHierarchyGraph();
```

Option 1보다 문제가 더 많은 방법입니다.

- **서비스 로케이터 안티패턴**: `ApplicationContext`를 직접 사용하면 의존성이 코드 내부에 숨겨져 생성자 시그니처만으로 의존 관계를 파악할 수 없게 됩니다
- **테스트 복잡도 증가**: 단위 테스트에서 Mock 주입 대신 `ApplicationContext` 자체를 stubbing해야 하므로 테스트 설정이 번거로워집니다
- **타입 안전성 약화**: Bean 조회가 런타임에 이루어지므로 의존 대상이 존재하지 않거나 타입이 다를 때 컴파일 시점에 잡히지 않습니다
- **DI 컨테이너의 이점 포기**: Spring이 제공하는 의존성 주입, 순환 참조 감지, 라이프사이클 관리를 수동으로 우회하는 셈입니다

### 3. 캐시 전담 컴포넌트 분리

캐시 로직을 별도 Bean으로 분리하면, 호출이 항상 외부 호출이 되므로 프록시가 정상적으로 개입합니다. 이 방법을 선택했습니다.

## 해결: 캐시 컴포넌트 분리

`RoleHierarchyCacheComponent`를 별도 Bean으로 만들어 `@Cacheable`과 `@CacheEvict`를 전담하게 했습니다.

```java
@Component
@RequiredArgsConstructor
public class RoleHierarchyCacheComponent {

    private final RoleIncludeRepository roleIncludeRepository;

    /**
     * 역할 계층 DAG를 Redis에 캐시합니다.
     */
    @Cacheable(value = "roleHierarchy", key = "'graph'")
    public Map<Long, Set<Long>> getHierarchyGraph() {
        List<RoleInclude> includes = roleIncludeRepository.findAll();
        return includes.stream()
            .collect(Collectors.groupingBy(
                ri -> ri.getRole().getRoleId(),
                Collectors.mapping(
                    ri -> ri.getIncludedRole().getRoleId(),
                    Collectors.toSet()
                )
            ));
    }

    /**
     * 역할 구조 변경 시 캐시를 무효화합니다.
     */
    @CacheEvict(value = "roleHierarchy", allEntries = true)
    public void evictHierarchyCache() {
        // 캐시 무효화만 수행
    }
}
```

`RoleHierarchyService`는 비즈니스 로직만 담당하고, 캐시 컴포넌트를 주입받아 사용합니다.

```java
@Service
@RequiredArgsConstructor
public class RoleHierarchyService {

    private final RoleHierarchyCacheComponent cacheComponent;

    /**
     * BFS로 직접 역할에서 도달 가능한 모든 역할을 계산합니다.
     */
    public Set<Long> resolveEffectiveRoles(Set<Long> directRoleIds) {
        Map<Long, Set<Long>> graph = cacheComponent.getHierarchyGraph(); // 외부 호출 → 캐시 동작

        Set<Long> effective = new LinkedHashSet<>(directRoleIds);
        Queue<Long> queue = new LinkedList<>(directRoleIds);

        while (!queue.isEmpty()) {
            Long current = queue.poll();
            Set<Long> children = graph.getOrDefault(current, Collections.emptySet());
            for (Long child : children) {
                if (effective.add(child)) {
                    queue.add(child);
                }
            }
        }
        return effective;
    }

    public boolean wouldCreateCycle(Long parentRoleId, Long childRoleId) {
        Map<Long, Set<Long>> graph = cacheComponent.getHierarchyGraph();
        // cycle 감지 로직 (DAG 검증)
        // ...
    }

    // 기존 호출자가 Service를 통해 접근하므로 public API를 유지하기 위한 pass-through (Facade)
    public Map<Long, Set<Long>> getHierarchyGraph() {
        return cacheComponent.getHierarchyGraph();
    }

    public void evictHierarchyCache() {
        cacheComponent.evictHierarchyCache();
    }
}
```

## 같은 패턴의 반복: Saga 보상 트랜잭션

동일한 self-invocation 문제가 shopping-service의 Saga 보상 로직에서도 발생했습니다. `OrderSagaOrchestrator`에서 주문 실행과 보상을 모두 처리하고 있었는데, 보상 메서드에 `@Transactional(propagation = Propagation.REQUIRES_NEW)`를 걸어도 같은 클래스에서 호출하면 새 트랜잭션이 생성되지 않았습니다.

```java
@Service
@RequiredArgsConstructor
public class OrderSagaOrchestrator {

    @Transactional
    public OrderResult executeSaga(OrderCommand command) {
        try {
            // 1. 재고 차감 (Feign)
            // 2. 결제 처리
            // 3. 주문 확정
        } catch (Exception e) {
            compensate(sagaState, e.getMessage()); // self-invocation → REQUIRES_NEW 무시
            throw e;
        }
    }

    @Transactional(propagation = Propagation.REQUIRES_NEW)
    public void compensate(SagaState sagaState, String errorMessage) {
        // 보상 로직 — 별도 트랜잭션이어야 하지만 실제로는 기존 트랜잭션에 참여
    }
}
```

`executeSaga()`가 실패하면 트랜잭션 전체가 롤백되면서 보상 기록까지 함께 롤백되는 것이 문제였습니다. 해결 방법은 동일합니다. `SagaCompensationService`를 별도 Bean으로 분리했습니다.

```java
@Service
@RequiredArgsConstructor
public class SagaCompensationService {

    @Transactional(propagation = Propagation.REQUIRES_NEW)
    public void compensate(SagaState sagaState, String errorMessage) {
        // 보상 로직 — 별도 트랜잭션에서 실행
    }
}
```

## 정리

| 문제 | 원인 | 해결 |
|------|------|------|
| `@Cacheable` 미동작 | self-invocation으로 프록시 우회 | 캐시 전담 컴포넌트 분리 |
| `REQUIRES_NEW` 미동작 | self-invocation으로 프록시 우회 | 보상 로직 별도 서비스 분리 |

핵심 원칙은 단순합니다. **AOP 어노테이션이 붙은 메서드는 반드시 외부 Bean에서 호출되어야 합니다.** 같은 클래스 내부 호출은 프록시를 거치지 않으므로 어노테이션이 무시됩니다. 이 제약을 알면, 캐시든 트랜잭션이든 처음부터 컴포넌트를 분리하는 습관을 들이게 됩니다.
