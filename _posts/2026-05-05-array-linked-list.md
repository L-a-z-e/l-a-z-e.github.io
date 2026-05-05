---
title: "Array vs LinkedList"
description: "Array vs LinkedList"
author: laze
date: 2026-05-05 16:44:00 +0900
categories: [Dev, DataStructure]
tags: [DataStructure, Array, LinkedList, Cache]
---

***

# [Data Structure] Array vs LinkedList

자료구조를 학습할 때 가장 먼저 마주하는 주제는 아마도 Array와 LinkedList의 비교일 것이다. 면접 단골 질문이기도 한 이 주제는 대개 시간 복잡도(Big-O)를 비교하는 선에서 마무리되곤 한다.

하지만 **"이론적인 O(1)이 현실의 하드웨어에서도 동일한 성능을 보장할까?"**, 그리고 **"실무에서 LinkedList는 도대체 언제 쓰이는 걸까?"**라는 의문이 들었다. 이번 글에서는 단순한 시간 복잡도 비교를 넘어, 메모리 구조(캐시 지역성)와 실제 서비스 아키텍처(Redis, Spring) 레벨까지 이 둘의 차이를 파고들어 본다.

---

## 1. 이론적 기초 점검: 흔한 착각 바로잡기

먼저 교과서적인 시간 복잡도를 짚고 넘어가 보자.

| 자료구조 | 탐색 (Search) | 끝 삽입/삭제 | 중간 삽입/삭제 |
| :--- | :---: | :---: | :---: |
| **Array** | O(1) | O(1) | O(N) |
| **LinkedList** | O(N) | O(1) | **O(1) (참조 보유 시)** |

흔히 "삽입과 삭제가 잦으면 LinkedList를 써야 한다"고 기계적으로 외우곤 한다. 하지만 여기에는 중요한 전제조건이 빠져있다. **LinkedList의 중간 삽입/삭제가 O(1)인 것은 '해당 노드의 참조(주소)를 이미 알고 있을 때'에 한정된다.**

만약 특정 값을 찾아서 지우거나 그 뒤에 삽입해야 한다면, 탐색 비용 O(N)이 선행되므로 결국 O(N)의 시간이 걸린다.

---

## 2. 이론을 뒤집는 현실: CPU 캐시 지역성 (Cache Locality)

그렇다면 탐색 없이 맨 앞/뒤에만 요소를 추가/삭제하는 경우라면 무조건 LinkedList가 빠를까? 벤치마크 테스트를 돌려보면 예상과 달리 **ArrayList가 더 빠른 경우가 많다.** 그 이유는 하드웨어, 즉 **CPU 캐시 지역성(Cache Locality)** 때문이다.

> **캐시 지역성이란?**
> CPU는 메인 메모리에서 데이터를 읽을 때, 요청한 주소 하나만 가져오지 않는다. 효율을 위해 캐시 라인 단위(보통 64바이트)로 인접한 메모리를 통째로 L1/L2 캐시에 올려버린다.

- **Array (연속 메모리):** 메모리 공간에 데이터가 연속으로 배치되어 있다. `arr[0]`을 읽을 때 `arr[1]`, `arr[2]` 등이 이미 CPU 캐시에 올라오므로(Cache Hit), 다음 데이터 접근 속도가 비약적으로 빠르다.
- **LinkedList (산재된 메모리):** 각 노드가 Heap 메모리 공간 곳곳에 흩어져 있다. (예: Node A는 `0x100`, Node B는 `0x4F0`). 노드를 순회할 때마다 높은 확률로 캐시 미스(Cache Miss)가 발생하여 메인 메모리까지 다녀와야 한다.

결국 LinkedList는 각 노드가 `prev/next` 포인터를 가져야 하는 메모리 오버헤드에 더해, 캐시 미스 문제까지 겹치며 실무에서 단독으로 쓰이는 경우가 극히 드물어지게 된다.

---

## 3. LinkedList의 진가: LRU 캐시 알고리즘

그렇다면 Java 생태계에서 `LinkedList`는 무용지물일까? 그렇지 않다. LinkedList가 진정한 O(1)의 마법을 부리는 완벽한 유스케이스가 있다. 바로 **LRU(Least Recently Used) 캐시 구현**이다.

앞서 LinkedList의 삽입/삭제가 O(1)이 되려면 '외부에서 노드 참조를 직접 쥐고 있어야 한다'고 했다. 이를 해결하기 위해 **HashMap**과 **Doubly LinkedList**를 결합한다.

1. **HashMap**은 Key와 함께 `LinkedList의 Node 참조`를 Value로 저장한다.
2. 데이터를 찾을 때 HashMap을 통해 O(1)로 Node의 메모리 주소를 바로 찾아낸다.
3. Node를 찾았으니 탐색 과정 없이 포인터(prev, next)만 조작하여 해당 Node를 LinkedList의 맨 앞(최근 사용)으로 이동시킨다. **(완벽한 O(1))**

만약 이를 Array로 구현했다면, 최근에 사용된 데이터를 맨 앞으로 옮기기 위해 나머지 모든 요소들을 한 칸씩 뒤로 밀어야 하므로 꼼짝없이 O(N)이 발생했을 것이다.

### 💡CPU 캐시와 LRU 캐시
- **CPU 캐시 (하드웨어 레벨):** 메모리 접근 횟수를 줄이는 것이 목적 (Array가 유리)
- **LRU 캐시 (애플리케이션 레벨):** DB 조회나 API 호출 횟수를 줄이는 것이 목적 (LinkedList 기반 결합이 유리)

LinkedList 기반의 LRU는 힙에 데이터가 산재해 있어 *CPU 캐시 관점에서는 불리*하지만, *DB/API 접근이라는 훨씬 큰 비용을 줄여주기 때문에* 트레이드오프(Trade-off) 관점에서 이를 감수하고 사용하는 것이다.

---

## 4. 실무 환경에서의 캐시 아키텍처

이러한 원리들은 실제 백엔드 생태계에서 어떻게 구현되어 있을까?

### Java의 `LinkedHashMap`
Java에서는 `LinkedHashMap`이 앞서 설명한 HashMap + Doubly LinkedList 구조를 이미 구현해 두었다. `accessOrder=true` 옵션을 주고 `removeEldestEntry()`를 오버라이드하면 손쉽게 LRU 캐시를 만들 수 있다.

```java
// 최대 100개의 엔트리만 유지하는 LRU 캐시 구현
int capacity = 100;
LinkedHashMap<String, Object> lruCache = new LinkedHashMap<>(capacity, 0.75f, true) {
    @Override
    protected boolean removeEldestEntry(Map.Entry<String, Object> eldest) {
        return size() > capacity; // 용량을 초과하면 가장 오래된 데이터 자동 삭제
    }
};
```

### Redis의 근사 LRU (Approximated LRU)
메모리 기반 NoSQL인 Redis는 정통적인 LinkedList 기반 LRU를 사용하지 않는다. 수천만 개의 Key를 다뤄야 하는 환경에서 LinkedList의 포인터 메모리 오버헤드와 CPU 캐시 미스는 치명적이기 때문이다. 대신, Key마다 마지막 접근 시간을 기록하고 한계에 다다르면 랜덤으로 N개의 샘플을 뽑아 그중 가장 오래된 것을 지우는 **근사치 알고리즘**을 사용하여 성능과 효율을 모두 챙겼다.

### Spring 프레임워크와 Caffeine
Spring에서 `@Cacheable`과 함께 현대에 가장 많이 쓰이는 로컬 캐시 라이브러리는 **Caffeine**이다. Caffeine은 단순한 LRU의 한계를 극복하기 위해 `W-TinyLFU`라는 훨씬 발전된 알고리즘을 사용하여 히트율(Hit Rate)을 극대화한다.

---

## 5. 마무리

단순히 "Array는 탐색, LinkedList는 삽입/삭제에 좋다"라는 1차원적인 지식에서 출발했지만, 하드웨어의 동작 방식(캐시 지역성)을 이해하고 나니 이론과 현실의 괴리를 알 수 있었다.

나아가 이 한계를 극복하기 위해 자료구조를 결합(HashMap+LinkedList)하고, 실무의 거대한 트래픽 앞에서는 또 다른 아키텍처적 타협(Redis의 근사치 알고리즘, Caffeine의 LFU)을 이뤄낸다는 사실은 엔지니어링에서 '절대적인 Silver Bullet은 없다'는 것을 다시 한번 깨닫게 해 준다.
