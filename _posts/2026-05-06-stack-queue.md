---
title: "Stack, Queue, synchronized, JVM"
description: "Stack, Queue, synchronized, JVM"
author: laze
date: 2026-05-06 16:02:00 +0900
categories: [Dev, DataStructure]
tags: [DataStructure, Stack, Queue, Synchronized, JVM]
---
## Java에서 Stack 클래스를 지양해야 하는 이유와 대체재

Stack, Queue, Deque의 기본 구조와 활용 사례를 정리하고, Java 환경에서 이를 구현할 때 주의해야 할 점을 짚어본다.

### 기본 자료구조와 활용 사례

각 자료구조는 데이터의 입출력 방향과 순서에 따라 명확한 사용 목적을 가진다.

| 자료구조 | 핵심 구조 | 대표적인 활용 사례 |
| :--- | :--- | :--- |
| **Stack** | LIFO (후입선출) | 브라우저 뒤로가기, 함수 콜스택, 괄호 유효성 검사, DFS 알고리즘 |
| **Queue** | FIFO (선입선출) | 스레드풀 작업 대기열, Kafka 메시지 처리, BFS 알고리즘 |
| **Deque** | 양방향 삽입/삭제 | 슬라이딩 윈도우 최솟값/최댓값 문제 (윈도우 이동 시 앞뒤 동시 처리) |

### Stack 클래스의 치명적인 문제점

Java에서 이러한 자료구조를 구현할 때 가장 주의해야 할 점은 `java.util.Stack` 클래스의 사용을 피해야 한다는 것이다. 이름은 Stack이지만, 내부 구현을 살펴보면 크게 두 가지 구조적 결함이 존재한다.

**첫째, Vector 상속으로 인한 동기화 오버헤드다.**

```java
// java.util.Stack의 내부 선언
public class Stack<E> extends Vector<E> {
    // ...
}

// 부모 클래스인 Vector의 내부 메서드 예시
public synchronized void addElement(E obj) {
    // ...
}
```

Stack 클래스는 Java 1.0 시절의 레거시 자료구조인 `Vector`를 상속받는다. Vector의 가장 큰 특징은 모든 주요 연산에 `synchronized` 키워드가 붙어 있다는 점이다. 이로 인해 단일 스레드 환경에서 안전하게 사용할 때조차 불필요하게 락(Lock)을 획득하고 반납하는 동기화 비용을 계속해서 지불해야 하며, 이는 곧장 성능 저하로 이어진다.

**둘째, LIFO(후입선출) 원칙의 붕괴다.**

Vector를 상속받았기 때문에 Stack의 본질에 어긋나는 Vector의 메서드들이 외부에 그대로 노출된다. Stack은 한쪽 끝에서만 데이터를 넣고 빼야 하지만, `get(int index)`나 `add(int index, E e)` 같은 메서드를 사용하면 중간 원소에 무작위로 접근하거나 삽입할 수 있게 된다. 결과적으로 자료구조의 목적과 캡슐화가 완전히 깨지게 된다.

### 해결방안

이러한 문제들로 인해 `Stack` 클래스 대신 목적과 환경에 맞는 다른 구현체를 선택해야 한다.

*   **단일 스레드 환경:** 동기화 오버헤드가 없는 `ArrayDeque`를 사용한다. Stack의 기능이 필요할 때는 `push()`와 `pop()`을, Queue의 기능이 필요할 때는 `offer()`와 `poll()`을 사용하여 인터페이스의 의도를 명확히 할 수 있다.
*   **멀티스레드 환경:** 스레드 안전성(Thread-safety)이 보장되어야 한다면, 무거운 락을 쓰는 `Stack` 대신 `ConcurrentLinkedDeque`나 `BlockingQueue` 등 동시성 처리에 최적화된 클래스를 용도에 맞게 선택한다.

`Stack` 클래스의 문제를 파악하는 과정에서 동기화 키워드인 `synchronized`가 객체에 어떻게 락을 걸고 성능에 영향을 미치는지 확인했다.
`synchronized`가 내부적으로 JVM과 상호작용하며 락을 통제하는 원리, 그리고 이를 대체할 수 있는 더 나은 동시성 제어 기법에 대해 다뤄본다.

## 동시성: synchronized 키워드와 JVM Monitor 구조

앞서 `Stack` 클래스의 구조적 결함을 살펴보며, `synchronized` 키워드가 불필요한 동기화 오버헤드를 발생시키는 원인임을 확인했다. 
`synchronized`가 내부적으로 어떻게 상호 배제(Mutual Exclusion)를 강제하는지 JVM 레벨의 동작 원리를 알아본다.

### 동시성 문제의 근원과 Mutex Lock

JVM 환경에서 여러 스레드는 힙(Heap) 메모리를 공유한다. 이는 여러 스레드가 같은 객체의 상태를 동시에 변경할 수 있음을 의미한다.

```java
class Counter {
    int count = 0; 
    
    void increment() {
        count++; 
    }
}
```

위 코드의 `count++`는 단일 명령어처럼 보이지만, 실제로는 **읽기(LOAD) → 더하기(ADD) → 쓰기(STORE)** 의 3단계로 동작한다. 두 스레드가 동시에 이 메서드를 실행하면 갱신되기 전의 같은 값을 읽어 연산하므로 결과가 누락되는 문제가 발생한다. 즉, 연산의 원자성(Atomicity)이 보장되지 않는다.

이를 해결하기 위한 기법이 **뮤텍스 락(Mutex Lock)** 이다. CPU 레벨에서 지원하는 CAS(Compare-And-Swap) 명령어를 활용하여, 한 번에 하나의 스레드만 임계 구역에 진입할 수 있도록 락 상태를 원자적으로 읽고 쓴다.

### JVM의 Monitor와 객체 헤더(Mark Word)

Java는 뮤텍스 락을 기반으로 **Monitor**라는 동기화 메커니즘을 제공한다. Java의 모든 객체는 생성될 때 힙 메모리에 '객체 헤더(Object Header)'를 가지며, 이 안의 **Mark Word** 영역이 락 상태와 스레드 ID를 기록하는 Monitor 역할을 수행한다.

```text
[힙 메모리의 객체 구조]
┌─────────────────────────────┐
│ Object Header               │
│   Mark Word: [락 상태 정보] │  ← 동기화 시 진입 스레드 ID 기록
│   Klass pointer             │
├─────────────────────────────┤
│ 인스턴스 필드 변수들        │
└─────────────────────────────┘
```

개발자가 `synchronized` 키워드를 사용하면, 컴파일러는 바이트코드 레벨에서 임계 구역의 시작과 끝에 `monitorenter`와 `monitorexit` 명령어를 자동으로 삽입한다.

*   **`monitorenter`:** 객체의 Mark Word를 확인한다. 비어있으면 CAS 연산을 통해 현재 스레드 ID를 기록하고 진입(락 획득)한다.
*   **`monitorexit`:** 연산이 끝나면 Mark Word를 비워 락을 반납하고 대기 중인 다른 스레드를 깨운다.

### 대기 공간의 분리: Entry Set과 Wait Set

단순히 락을 획득하고 반납하는 것만으로는 복잡한 비즈니스 로직을 처리할 수 없다. 예를 들어, 큐(Queue)가 비어있을 때 소비 스레드가 락을 쥔 채로 데이터가 들어올 때까지 무한 루프(Busy Waiting)를 돌면 심각한 자원 낭비와 교착 상태(Deadlock)가 발생한다.

이를 방지하기 위해 Monitor는 대기 공간을 두 가지로 분리하여 관리한다.

| 구분 | Entry Set | Wait Set |
| :--- | :--- | :--- |
| **진입 원인** | 락을 획득하지 못해 대기 | 락은 얻었으나, 특정 조건이 불만족하여 대기 |
| **스레드 상태** | `BLOCKED` | `WAITING` |
| **사용 메서드** | `synchronized` 진입 시 자동 | `wait()` 호출 시 |
| **탈출 조건** | 락 보유 스레드가 종료 시 경쟁 | 다른 스레드가 `notify()` / `notifyAll()` 호출 |

`wait()`와 `notifyAll()`을 활용하면 조건에 따른 안전한 스레드 제어가 가능하다.

```java
public class BoundedQueue {
    private final Object lock = new Object();
    private final Queue<Integer> queue = new LinkedList<>();
    private final int MAX = 5;

    public void put(int item) throws InterruptedException {
        synchronized (lock) {
            // 조건문은 반드시 if가 아닌 while을 사용해야 함
            while (queue.size() == MAX) {  
                lock.wait(); // 락 반납 후 Wait Set에서 대기
            }
            queue.add(item);
            lock.notifyAll(); // Wait Set의 스레드들을 Entry Set으로 이동
        }
    }
}
```

위 코드에서 조건을 `if`가 아닌 `while`로 검사하는 이유는 필수적이다. `notifyAll()`로 인해 깨어난 스레드가 다시 락을 획득하는 그 짧은 찰나에, 다른 스레드의 개입으로 조건이 다시 불만족 상태로 변경될 수 있기 때문이다. `while`을 사용해야만 락 획득 직후 조건을 재검증하여 버그를 차단할 수 있다.

### 한계와 다음 단계

`synchronized` 키워드는 JVM이 락 획득과 반납(`monitorexit`)을 자동으로 처리해주어 예외 발생 시에도 안전하다는 장점이 있다.

하지만 치명적인 단점도 존재한다.
1. 특정 시간 동안만 락을 기다리는 **타임아웃(Timeout) 설정이 불가능**하다.
2. 대기 중인 스레드의 순서를 보장하는 **공정성(Fairness) 제어가 안 된다**.
3. 락 대기 중에 외부에서 **작업을 취소(Interrupt)할 수 없다**.

실무에서는 이러한 단순 상호배제의 한계를 극복하고 더 세밀한 제어를 하기 위해 `java.util.concurrent` 패키지의 도구들을 사용한다. 

다음은 은행 창구 시뮬레이션 코드를 바탕으로 `ReentrantLock`, `Semaphore`, 그리고 `volatile`의 실제 활용법을 다룬다.

## 코드로 검증하는 멀티스레딩: 은행 창구 시뮬레이션과 한계 극복

앞선 과정에서 `synchronized`와 JVM Monitor 구조를 통해 락(Lock)의 기본 원리를 살펴보았다. 하지만 실무에서는 단순히 하나의 임계 구역을 잠그는 것을 넘어, 자원의 개수를 제한하거나 대기 순서를 보장해야 하는 등 더 세밀한 제어가 필요하다. 이러한 요구사항을 반영해 3개의 창구가 있는 은행 시뮬레이션 코드를 작성하며 `java.util.concurrent` 패키지의 도구들을 검증해 보았다.

### 자원 제한과 순서 보장: Semaphore와 ReentrantLock

은행에는 3개의 창구가 있고, 고객은 번호표 순서대로 진입해야 한다. `synchronized`는 동시 진입을 무조건 1개로 제한하며, 대기 중인 스레드(Entry Set) 중 누가 먼저 락을 얻을지 순서를 보장하지 않는다는 문제가 있다.

이 문제는 `Semaphore`와 `ReentrantLock`을 결합하여 해결할 수 있다.

```java
public class BankShared {
    // 1. 창구 슬롯 3개 제한 (소유권 없는 카운터)
    private static final Semaphore windowSlots = new Semaphore(3);
    
    // 2. 대기열 순서 보장 (공정 모드 락)
    private static final ReentrantLock fairLock = new ReentrantLock(true); 
    
    // 3. 직원 업무 준비 상태 (가시성 보장)
    public static volatile boolean staffReady = false;
    public static final Object staffMonitor = new Object();
}
```

*   **`Semaphore(3)`:** 락의 소유권 개념 없이 허용 개수(카운터)만 관리한다. `acquire()` 호출 시 카운터가 차감되고 0이 되면 대기하며, 처리가 끝난 후 `release()`를 호출해 대기 중인 다른 스레드를 통과시킨다.
*   **`ReentrantLock(true)`:** 생성자의 `true` 옵션은 대기열을 FIFO(선입선출) 순서로 관리하는 공정 모드(Fairness)를 켠다는 의미다. 먼저 도착한 스레드가 먼저 락을 획득하도록 보장한다.

이러한 공유 락 객체들은 반드시 `private static final`로 선언해야 한다. `static`으로 모든 스레드가 힙 영역의 같은 락을 공유하도록 하고, `final`로 참조값 변경을 막아야 한다. 특히 `private` 접근 제어자가 누락될 경우, 외부의 악의적이거나 잘못된 코드가 락 객체에 임의로 접근해 시스템 전체를 교착 상태로 만들 수 있다.

### 2. 스레드 가시성(Visibility)과 volatile

은행의 영업 상태나 직원의 준비 상태를 나타내는 `boolean` 플래그는 여러 스레드가 동시에 참조한다. 이 플래그를 일반 변수로 선언하면 CPU 코어별 캐시(Cache) 메모리에 값이 저장되어, 한 스레드가 값을 변경해도 다른 스레드가 과거의 값을 읽는 가시성(Visibility) 문제가 발생한다.

이때 `volatile` 키워드를 사용하면 변수를 읽고 쓸 때 CPU 캐시를 무시하고 메인 메모리(RAM)에 직접 접근하도록 강제할 수 있다.

| 제어 기법 | 가시성 (메모리 갱신 확인) | 원자성 (복합 연산 동시성) | 적합한 용도 |
| :--- | :--- | :--- | :--- |
| **`volatile`** | 보장함 | 보장 안 함 | 한 스레드가 쓰고 나머지는 읽기만 하는 플래그 (`bankOpen` 등) |
| **`synchronized`** | 보장함 | 보장함 | 여러 스레드가 복합적인 상태를 읽고 수정할 때 |
| **`AtomicInteger`** | 보장함 | 보장함 (CAS 연산) | 동시성 환경에서의 단순 숫자 증감 (`count++` 등) |

`volatile`은 변경된 최신 상태를 보여줄 뿐, 여러 단계의 연산(읽고-더하고-쓰기)을 하나로 묶어주지는 않는다. 따라서 여러 스레드가 값을 동시에 갱신해야 한다면 CAS(Compare-And-Swap) 기반의 `Atomic` 클래스를 사용하는 것이 적합하다.

### 안전한 스레드 중단과 종료: interrupt와 join

은행 영업이 종료되면 대기 중인 고객 스레드들을 안전하게 취소하고 돌려보내야 한다. 스레드는 강제로 종료할 수 없으며, 대신 `interrupt()` 메서드를 통해 중단 신호(플래그)를 보내야 한다.

```java
// 고객 스레드 내부
try {
    // 대기 중 인터럽트 신호 수신 가능
    fairLock.lockInterruptibly(); 
    
    // ... 업무 로직 ...
    
} catch (InterruptedException e) {
    // 1. 신호를 받으면 플래그가 자동 초기화되며 예외 발생
    // 2. 현재 스레드의 인터럽트 상태를 다시 복원 (상위 코드 인지를 위해)
    Thread.currentThread().interrupt(); 
    return; // 3. run() 메서드 종료 시 스레드 소멸
} finally {
    fairLock.unlock(); // 락 반납 보장
}
```

`lock()` 대신 `lockInterruptibly()`를 사용하면 락을 기다리는 중에도 인터럽트 신호를 받아 예외를 발생시키고 대기를 취소할 수 있다. 예외가 발생하는 순간 내부의 `interrupted` 플래그가 `false`로 초기화되므로, 호출 스택 상단의 다른 로직이 인터럽트 여부를 알 수 있도록 `Thread.currentThread().interrupt()`를 호출해 플래그를 복원하는 습관을 들이는 것이 좋다. 이후 `return`을 통해 `run()` 메서드를 빠져나가면 스레드는 자연스럽게 종료되며, 메인 스레드는 `join()` 메서드를 사용해 작업 스레드들이 모두 종료될 때까지 대기할 수 있다.

### 로컬 환경의 한계와 다중 인스턴스 확장

지금까지 살펴본 `synchronized`, `ReentrantLock`, `volatile`은 모두 단일 JVM의 힙 메모리를 공유한다는 전제하에 동작한다.

결과적으로 물리적으로 분리된 다중 서버(인스턴스) 환경에서는 이러한 Java 키워드 수준의 제어가 무의미해진다. 메모리 공간이 나뉘어 있어 서로의 락 상태를 인지할 수 없기 때문이다. 사이드 프로젝트인 Portal Universe를 개발하며 분산 환경의 동시성 제어를 위해 Redis Lua Script를 활용했던 근본적인 이유가 바로 여기에 있었다. 단일 Redis를 여러 서버의 중재자로 두어야만 다중 인스턴스 간의 상호 배제가 가능했던 것이다.

자료구조의 동작 원리에 대한 단순한 호기심으로 시작된 학습이었지만, 모니터 구조부터 분산 락의 필요성까지 이어지며 동시성 제어의 전체적인 윤곽을 체계적으로 짚어볼 수 있었다.
