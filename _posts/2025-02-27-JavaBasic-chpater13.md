---
title: 자바의 정석 Chapter 13
description: 모르는 내용 위주 학습 기록
author: laze
date: 2025-02-27 00:06:00 +0900
categories: [Dev, Java]
tags: [Java]
---
# Chapter 13

- process
    - data + memory + thread 로 구성
- thread
    - process의 자원을 이용해 실제로 작업을 수행
    - 개별적인 call stack을 필요로 함

- multi threading
    - 장점
        - cpu 사용률 향상
        - 자원을 효율적으로 사용
        - 작업이 분리되어 코드가 간결해짐
    - 단점
        - synchronization, deadlock 같은 문제 고려해야 함

- thread 구현 및 실행
    - Thread 클래스 상속
        
        ```java
        class MyThread extends Thread {
        	public void run(){}
        }
        ```
        
    - Runnable 인터페이스 구현
        
        ```java
        class MyThread implements Runnable {
        	public void run(){}
        }
        ```
        
    - instance 생성 방법 차이
        
        ```java
        // Thread 상속시 -> Thread의 자손 클래스의 인스턴스를 생성
        ThreadEx1 t1 = new ThreadEx1();
        
        // Runnalbe 구현시 -> Runnable을 구현한 클래스의 인스턴스를 생성
        Runnable r = new ThreadEx2();
        Thread t2 = new Thread(r); // Thread(Runnalbe target) 생성자 사용
        
        Thread t2 = new Thread(new ThreadEx2()); // 한 줄로 사용 시
        ```
        
- Thread 클래스 메서드
    - `static Thread currentThread()` 현재 실행중인 쓰레드의 참조 반환
    - `String getName()` 쓰레드의 이름 반환
    - Thread 이름 설정
        - Thread(Runnable target, String name)
        - Thread(String name)
        - void setName(String name)
    - `start()`
        - start() 호출시 실행 대기 상태 → 자신의 차례가 되면 실행
        - 한 번 실행 종료된 쓰레드는 다시 실행할 수 없음
    - `run()`
        - main 메서드에서 run()을 호출하는 것은 생성된 쓰레드를 실행시키는 것이 아니라 단순히 메서드를 호출 하는 것
        - start()는 새로운 쓰레드가 작업을 실행하는데 필요한 call stack을 생성한다음 run()을 호출해 생성된 call stack에 run()이 올라가게 함

- 실행중인 사용자 thread가 하나도 없을 때 프로그램은 종료 됨

- Single thread ↔ Multi thread
    - context switching 에 걸리는 시간 때문에 single thread의 속도가 더 빠를 수 있음
    - 쓰레드가 각기 다른 자원을 사용하는 경우에는 multi thread가 더 효율적임

- Thread priority
    - 특정 thread가 더 많은 작업 시간을 갖도록 할 수 있음
    - `void setPriority(int newPriority)`
    - `int getPriority()`
    - main 메서드는 기본값 5
    - 단 OS에 종속적이기때문에 thread에 우선순위 부여보다 PriorityQueue에 저장해서 우선순위 높은 작업 먼저 처리하는 방식이 효율적

- Thread Group
    - 서로 관련된 thread를 그룹으로 다루기 위함
    - 보안상 이유로 도입
    - 자신이 속한 쓰레드 그룹이나 하위 쓰레드 그룹은 변경 가능하나 다른 쓰레드 그룹은 변경 불가
    - `ThreadGroup(String name)`
    - `Thread(ThreadGroup threadGroup, Runnable target, String name, long stackSize)`
    - JVM은 main과 system 이라는 쓰레드 그룹을 만듦
    - 생성한 쓰레드 그룹은 자동으로 main 쓰레드 그룹 하위로 들어감
    - `ThreadGroup getThreadGroup()`
    - `void uncaughtException(Thread t, Throwable e)` 쓰레드 그룹의 쓰레드가 처리되지 않은 예외에 의해 실행 종료되었을 때 호출

- Daemon Thread
    - 다른 일반 쓰레드의 작업을 돕는 보조적인 역할을 수행하는 쓰레드
    - 일반 쓰레드가 모두 종료되면 강제적으로 자동 종료됨
    - 가비지 컬렉터, 자동 저장, 자동 갱신 등이 있음
    - `boolean isDaemon()`
    - `void setDaemon(boolean on)` - 쓰레드 실행 전에 호출하면 데몬 쓰레드가 생성됨

- 실행 제어
    - Thread scheduling 관련 메서드
        
        
        | 메서드 | 설명 |
        | --- | --- |
        | `static void sleep(long millis, int nanos)` | 지정된 시간동안 쓰레드를 일시정지, 이후 자동적으로 다시 실행대기 상태 |
        | `void join(long millis, int nanos)` | 지정된 시간동안 쓰레드가 실행되도록 함. 지정된 시간이 지나거나 작업 종료시 join을 호출한 쓰레드로 돌아와 실행을 계속한다 |
        | `void interrupt()` | sleep(), join()에 의해 일시정지 상태인 쓰레드를 깨워서 실행대기 상태로 만듦 |
        | `void stop()` | 쓰레드 즉시 종료 |
        | `void suspend()` | 쓰레드 일시 정지 |
        | `void resume()` | 일시정지 상태의 쓰레드를 실행 대기상태로 변경 |
        | `static void yield()` | 실행 중 자신에게 주어진 실행시간을 다른 쓰레드에 양보하고 자신은 실행대기 상태로 변경 |
    - Thread 상태
        
        
        | 상태 | 설명 |
        | --- | --- |
        | NEW | 생성이후 start() 호출 전 |
        | RUNNABLE | 실행 중 혹은 실행 가능 상태 |
        | BLOCKED | 동기화블럭에 의해 일시정지된 상태(lock이 풀릴때까지 대기) |
        | WAITING, TIMED_WAITING | 쓰레드의 작업이 종료되지는 않았지만 실행가능하지 않은 일시정지 상태, 
        시간이 지정된경우 TIMED_WAITING |
        | TERMINATED | 쓰레드의 작업이 종료된 상태 |
    - sleep()
        - th1.sleep(2000) 을 호출해도 sleep은 항상 현재 실행중인 쓰레드에 작동하기때문에 참조변수를 이용해서 sleep하는 것이 아니라 Thread.sleep(2000); 과 같이 선언해야함
    - interrupt() & interrupted()
        - 쓰레드의 작업을 취소
        - 작업을 멈추라고 요청을 할 뿐 강제 종료시키지는 못함
        - intterupted()는 쓰레드에 대해 interrupt()가 호출 되었는지를 알려 줌
        - run() 중간에 Thread.sleep() 을 호출하면 InterruptedException 발생하고 다시 isInterrupted = false 로 변경됨
    - suspend(), resume(), stop() → deprecated
    - yeild()
        - 다른 쓰레드에게 자신에게 주어진 실행시간을 양보함
    - join()
        - 작업을 잠시 멈추고 다른 쓰레드가 지정된 시간동안 작업을 수행하도록 함
        
        ```java
        try {
        	th1.join(); // 현재 작업을 진행중인 쓰레드가 th1의 작업이 끝날때까지 기다림
        }catch(InterruptedException e) {}
        ```
        

- Thread의 동기화
    - 같은 프로세스 내의 여러 개의 쓰레드가 자원을 공유해서 작업하기 때문에 다른 쓰레드에 의해 방해받지 않도록 방지하는 것이 필요해짐
    - 임계 영역(critical section) 과 Lock이 도입됨
    
    ```java
    // 메서드 전체를 임계 영역으로 지정
    // 쓰레드가 해당 메서드를 호출한 시점부터 메서드 안의 객체의 lock을 얻음
    public synchronized void calcSum() { ... }
    
    // 특정한 영역을 임계 영역으로 지정
    // 해당 블럭 영역안에 들어가면 lock을 얻고 벗어날 때 반납
    synchronized(객체의 참조변수) { ... }
    ```
    
    - wait() & notify()
        - 동기화된 임계영역의 코드를 수행하다가 작업을 더 이상 진행할 상황이 아닐 때 wait()를 호출하여 lock을 반납하고 기다리게 하고 나중에 작업을 진행할 수 있는 상황에서 notify()를 호출하여 다시 lock을 얻고 작업을 진행할 수 있도록 함
        - 대기중인 모든 쓰레드들 중 임의의 쓰레드만 lock을 얻기 때문에 작업이 계속 지연될 수 있음
    - Lock & Condition
        - Lock Class
            - ReentrantLock - 재진입이 가능한 lock, 일반적인 배타 lock
                - ReentrantLock() / ReentrantLock(boolean fair)
                    - fair 값을 true로 주면 가장 오래 기다린 쓰레드부터 처리하는데 어떤 쓰레드가 가장 오래 기다렸는지 확인 하는 과정이 성능에 영향을 줌
                    - `void lock()`
                    - `void unlock()` → 일반적으로 try - finally 를 사용해서 처리
                    - `boolean isLocked()`
                    - `boolean tryLock()` → 다른 쓰레드에 의해 lock이 걸려 있으면 lock을 얻으려고 기다리지 않음
                        - `boolean tryLock(long timeout, TimeUnit unit) throws InterruptException` → 지정된 시간이 지나면 InterruptException 발생시키기
            - ReentrantReadWriteLock - 읽기에는 공유적이고, 쓰기에는 배타적인 lock
                - 읽기를 위한 lock과 쓰기를 위한 lock 제공
                - 읽기 lock은 중복해서 걸 수 있음
                - 읽기 상태에서 쓰기 lock 얻을 수 없음
            - StampedLock - ReentrantReadWriteLock에 낙관적인 lock 기능 추가
                - 쓰기 lock과 읽기 lock이 충돌할 때만 쓰기가 끝난 후에 읽기 lock을 얻음
                
                ```java
                int getBalance() {
                	long stamp = lock.tryOptimisticRead(); // 낙관적 읽기 lock 가져옴
                	
                	int curBalance = this.balance; // 공유 자원 balance
                	
                	if(!lock.validate(stamp)) { // 쓰기 lock에 의해 읽기 lock이 풀렸는지 체크
                		stamp = lock.readLock(); // lock이 풀렸으면 읽기 lock을 얻기위해 기다림
                		
                		try {
                			curBalance = this.balance;
                		} finally {
                			lock.unlockRead(stamp);
                		}
                		
                		return curBalance; // 읽기 lock이 풀리지 않았으면 곧바로 읽어온 값 반환
                	}
                ```
                
            
            - Condition
                
                ```java
                private ReentrantLock lock = new ReentrantLock();
                
                private Condition forCook = lock.newCondition();
                private Condition forCust = lock.newCondition();
                
                public void add(String dish) {
                	lock.lock();
                	
                	try {
                		while (dishes.size() >= MAX_FOOD) {
                			String name = Thread.currentThread().getName();
                			System.out.println(name + " is waiting.");
                			try {
                				forCook.await();
                			} catch(InterruptException e) {}
                			
                			dishes.add(dish);
                			forCust.signal();
                			System.out.println("Dishes: " + dishes.toString());
                		} finally {
                			lock.unlock();
                		}
                	}
                ```
                
                | Condition | Object |
                | --- | --- |
                | void await() | void wait() |
                | viod signal() | void notify() |
                | void signalAll() | void notifyAll() |
                - 쓰레드를 구분할 수 있게됨
                - 핵심은 어떤 조건일때 해당되는 쓰레드를 Condition 객체에 추가하고 추가된 쓰레드를 깨울 것인지
                
                | **구분** | **기아 현상 (Starvation)** | **경쟁 상태 (Race Condition)** |
                | --- | --- | --- |
                | **정의** | 특정 스레드가 CPU 자원을 오랫동안 할당받지 못하여 실행되지 못하는 상태 | 여러 스레드가 공유 자원에 동시에 접근하려고 시도할 때, 접근 순서에 따라 프로그램의 실행 결과가 달라지는 상황 |
                | **발생 원인** | 우선순위 불균형, 불공정한 스케줄링, 자원 독점 등 | 공유 자원 동시 접근, 동기화 부족 |
                | **영향** | 스레드 실행 지연 또는 중단 | 데이터 불일치, 프로그램 오류 등 |
                | **해결 방법** | 우선순위 조정, 공정한 스케줄링, 자원 할당 제한, Thread.yield() 활용 | 동기화, 원자적 연산 활용, 불변 객체 활용 |

- volatile
    - 멀티 코어 프로세서에서는 코어마다 별도의 캐시를 가지고 있음
    - 코어는 메모리에서 읽어온 값을 캐시에 저장하고 캐시에서 값을 읽어서 작업함
    - 같은 값을 읽을 때는 캐시에 있는지 확인하고 없을때만 메모리에서 읽어옴
    - volatile을 붙이면 코어가 변수의 값을 읽어올 때 캐시가 아닌 메모리에서 읽어오기때문에 메모리간의 값 불일치가 해결됨
    - synchronized 블럭을 사용해도 동일한 효과 → 쓰레드가 해당 블럭에 들어갈 때와 나올 때 캐시와 메모리간의 동기화가 이루어지기 떄문
    - volatile은 해당 변수에 대한 읽기 쓰기를 원자화 함(하나의 명령어로 다른 것이 끼어들지 못함 synchronized 는 개념상 블럭 자체를 원자화했다고 보면 됨) ≠ 동기화 한다는 뜻은 아님

- fork & join Framework
    - 하나의 작업을 작은 단위로 나눠서 여러 쓰레드가 동시에 처리하는 것을 쉽게 만들어 줌
    - RecursiveAction - 반환값이 없는 작업을 구현할 때 사용
    - RecursiveTask - 반환값이 있는 작업을 구현할 때 사용
    - compute() 추상메서드를 상속해서 구현
    
    ```java
    class ForkJoinEx {
    	static final ForkJoinPool pool = new ForkJoinPool(); // 쓰레드풀 생성
    	
    	public static void main(String[] args) {
    		long from = 1L;
    		long to = 100_000_000L;
    		
    		SumTask task = new SumTask(from, to);
    		
    		long start = System.currentTimeMillis();
    		Long result = pool.invoke(task); // 수행할 작업 시작 메서드
    		System.out.println("Elapsed time : " + (System.currentTimeMillis() - start));
    		System.out.println("result : " + result);
    		
    	}
    }
    
    class SumTask extends RecursiveTask<Long> {
    	long from, to;
    	
    	SumTask(long from, long to) {
    		this.from = from;
    		this.to = to;
    	}
    	
    	public Long compute() {
    		long size = to - from + 1; // from <= i <= to
    		
    		if(size <= 5)
    			return sum();
    			
    		long half = (from + to)/2;
    		
    		SumTask leftSum = new SumTask(from, half);
    		SumTask rightSum = new SumTask(half+1, to);
    		
    		leftSum.fork();
    		
    		return rightSum.compute() + leftSum.join();
    	}
    	
    	long sum() {
    		long tmp = 0L;
    		
    		for(long i = from; i <= to; i++)
    			tmp += i;
    		
    		return tmp;
    	}
    }
    ```
    
    - ForkJoinPool - 지정된 수의 쓰레드를 생성해서 미리 만들어 놓고 반복해서 재사용
        - 쓰레드풀은 쓰레드가 수행해야하는 작업이 담긴 큐를 제공, 각 쓰레드는 자신의 작업 큐에 담긴 작업을 순서대로 처리
    - fork() - 해당 작업을 쓰레드 풀의 작업 큐에 넣음 (비동기 메서드)
        - 자신의 작업 큐가 비어있는 쓰레드는 다른 쓰레드의 작업 큐에서 작업을 가져와 수행함 (work stealing) - 쓰레드풀에 의해 자동적으로 이루어짐
    - join() - 해당 작업의 수행이 끝날 때 까지 기다렸다가 수행이 끝나면 그 결과를 반환 (동기 메서드)
- fork & join 관련 궁금증 정리
    
    ```java
    SumTask(1, 100).compute()
    |   long result = rightSum.compute() + leftSum.join();
    |   return result;  <-- 최종 결과 반환
    |
    +--SumTask(51, 100).compute()
        |   long result = rightSum.compute() + leftSum.join();
        |   return result;  <-- 51~100 범위의 합 반환
        |
        +--SumTask(76, 100).compute()
            |   long result = rightSum.compute() + leftSum.join();
            |   return result;  <-- 76~100 범위의 합 반환
            |
            +--SumTask(89, 100).compute()
                |   long result = rightSum.compute() + leftSum.join();
                |   return result;  <-- 89~100 범위의 합 반환
                |
                +--SumTask(95, 100).compute()
                    |   long result = rightSum.compute() + leftSum.join();
                    |   return result;  <-- 95~100 범위의 합 반환
                    |
                    +--SumTask(98, 100).compute()
                        |   return sum();  <-- 98+99+100 = 297 반환 (기본 단계)
    ```
    
    재귀 호출과 스레드 동작의 핵심인 스택 프레임에 대해 자세히 설명해 드리겠습니다. 스택 프레임을 이해하면 재귀 호출, 메서드 호출, 스레드 동작 등을 훨씬 명확하게 파악할 수 있습니다.
    
    **스택 프레임(Stack Frame)이란?**
    
    스택 프레임은 **메서드가 호출될 때마다 생성되는 메모리 공간**입니다. 각 스레드는 자신만의 호출 스택(call stack)을 가지며, 메서드가 호출될 때마다 해당 메서드의 스택 프레임이 스택에 푸시(push)됩니다. 메서드 실행이 완료되면 해당 스택 프레임은 스택에서 팝(pop)됩니다.
    
    **스택 프레임의 구성 요소:**
    
    스택 프레임은 일반적으로 다음과 같은 구성 요소를 포함합니다.
    
    1. **지역 변수 (Local Variables):** 메서드 내에서 선언된 변수들의 값
    2. **매개변수 (Parameters):** 메서드에 전달된 인자들의 값
    3. **반환 주소 (Return Address):** 메서드 실행이 완료된 후 돌아갈 코드의 위치 (호출한 메서드의 다음 실행 지점)
    4. **피연산자 스택 (Operand Stack):** 중간 계산 결과, 메서드 호출 시 인자 전달 등에 사용되는 임시 저장 공간
    5. **프레임 데이터 (Frame Data):** 예외 처리 정보, 디버깅 정보 등
    
    **스택 프레임의 역할:**
    
    - **메서드 실행:** 메서드의 코드를 실행하는 데 필요한 모든 정보를 저장하고 관리합니다.
    - **지역 변수 관리:** 메서드 내에서 선언된 변수들의 값을 저장하고, 메서드 실행 동안 해당 변수들에 접근할 수 있도록 합니다.
    - **매개변수 전달:** 메서드 호출 시 전달된 인자들의 값을 저장하고, 메서드 내에서 해당 인자들을 사용할 수 있도록 합니다.
    - **반환 값 저장:** 메서드 실행이 완료된 후 반환할 값을 저장하고, 호출한 메서드에게 전달합니다.
    - **호출 스택 관리:** 메서드 호출 순서를 기록하고, 메서드 실행이 완료된 후 어디로 돌아가야 하는지 알려줍니다.
    
    **예시 (Factorial 함수):**
    
    ```java
    public class Factorial {
        public static int factorial(int n) {
            if (n <= 1) {
                return 1;
            } else {
                return n * factorial(n - 1);
            }
        }
    
        public static void main(String[] args) {
            int num = 3;
            int result = factorial(num);
            System.out.println(num + "! = " + result);
        }
    }
    
    ```
    
    `factorial(3)`을 호출했을 때, 스택 프레임이 어떻게 생성되고 관리되는지 살펴보겠습니다.
    
    1. **factorial(3) 호출:**
        - 스택 프레임: Frame 1
        - `n = 3`
        - `return 3 * factorial(2);`
    2. **factorial(2) 호출:**
        - 스택 프레임: Frame 2 (Frame 1 위에 쌓임)
        - `n = 2`
        - `return 2 * factorial(1);`
    3. **factorial(1) 호출:**
        - 스택 프레임: Frame 3 (Frame 2 위에 쌓임)
        - `n = 1`
        - `return 1;` // 기저 조건 도달
    4. **factorial(2)로 돌아감:**
        - Frame 2: `factorial(1)`이 `1`을 반환했으므로, `2 * 1 = 2`를 계산합니다.
        - `return 2;`
    5. **factorial(3)로 돌아감:**
        - Frame 1: `factorial(2)`가 `2`를 반환했으므로, `3 * 2 = 6`을 계산합니다.
        - `return 6;`
    
    **각 스택 프레임의 내용:**
    
    - **Frame 1 (factorial(3)):**
        - `n = 3`
        - 반환 주소: `main` 메서드의 특정 코드 위치
        - 지역 변수: `result` (미정)
    - **Frame 2 (factorial(2)):**
        - `n = 2`
        - 반환 주소: `factorial(3)`의 특정 코드 위치
    - **Frame 3 (factorial(1)):**
        - `n = 1`
        - 반환 주소: `factorial(2)`의 특정 코드 위치
    
    **스택 프레임의 중요성:**
    
    - **재귀 호출 지원:** 각 재귀 호출은 자신만의 스택 프레임을 가지므로, 서로 다른 호출 간에 변수 충돌 없이 독립적으로 작업을 수행할 수 있습니다.
    - **메서드 호출 추적:** 스택 프레임은 메서드 호출 순서를 기록하므로, 예외 발생 시 스택 트레이스(stack trace)를 통해 어떤 메서드에서 예외가 발생했는지 추적할 수 있습니다.
    - **스레드 컨텍스트 관리:** 각 스레드는 자신만의 스택을 가지므로, 스레드 간에 변수 공유 없이 독립적으로 실행될 수 있습니다.
    
    **비유 (서랍장):**
    
    스택 프레임을 서랍장에 비유할 수 있습니다.
    
    - 각 서랍은 메서드 호출에 해당합니다.
    - 각 서랍 안에는 해당 메서드에서 사용하는 물건들 (지역 변수, 매개변수 등)이 들어 있습니다.
    - 서랍은 차곡차곡 쌓여 있으며, 가장 위에 있는 서랍이 현재 실행 중인 메서드를 나타냅니다.
    - 메서드 실행이 끝나면 해당 서랍을 치우고, 그 아래에 있는 서랍을 꺼내서 실행합니다.
    
    **핵심**:
    
    스택 프레임은 메서드가 호출될 때마다 생성되는 메모리 공간으로, 메서드 실행에 필요한 모든 정보를 저장하고 관리합니다. 스택 프레임을 이해하면 재귀 호출, 메서드 호출, 스레드 동작 등을 명확하게 파악할 수 있습니다. 스택 프레임은 각 메서드 호출을 위한 독립적인 작업 공간을 제공하고, 메서드 호출 순서를 추적하며, 스레드 간의 독립적인 실행을 보장하는 데 중요한 역할을 합니다.
    
    각 스택 프레임은 다음 두 가지 정보를 **반드시** 가지고 있습니다.
    
    1. 자신이 계산해야 할 값 (n 값)
    2. 이전 단계의 계산 결과 (하위 프레임의 반환 값)
    
    이 두 가지 정보를 이용하여 자신의 결과를 계산하고, 그 결과를 **자신의 반환 값**으로 설정하여 이전 프레임으로 전달합니다.