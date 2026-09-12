# 2주차 - OS - Process / Thread / 동시성

## 1. Process

> 실행 중인 프로그램이다.

프로그램 자체는 디스크에 저장된 실행 파일이고, 이를 실행하면 OS가 메모리와 필요한 자원을 할당하면서 프로세스가 된다.

### 프로세스의 메모리 구조

프로세스는 실행되면서 자신만의 메모리 공간을 가진다.

```
┌─────────────────┐
│      Stack      │ → 함수 호출, 지역 변수
├─────────────────┤
│        ↓        │
│                 │
│        ↑        │
├─────────────────┤
│      Heap       │ → 동적으로 생성된 객체
├─────────────────┤
│      Data       │ → 전역/정적 변수
├─────────────────┤
│      Code       │ → 실행할 코드
└─────────────────┘
```

- **Code**: 실행할 프로그램 코드
- **Data**: 전역 변수, 정적 변수 등
- **Heap**: 동적으로 생성되는 데이터
- **Stack**: 함수 호출 정보, 지역 변수 등

자바에서는 객체가 주로 힙에 생성되고, 메서드 호출과 지역 변수 등은 각 스레드의 스택과 관련된다.

### 프로세스는 서로 독립적이다

각 프로세스는 기본적으로 독립된 메모리 공간을 가진다.

```
Process A                Process B

┌──────────┐            ┌──────────┐
│ Memory A │            │ Memory B │
└──────────┘            └──────────┘
```

따라서 프로세스 A가 프로세스 B의 메모리에 직접 접근할 수 없다.

덕분에 한 프로세스에 문제가 발생해도 다른 프로세스에 미치는 영향을 줄일 수 있지만, 서로 데이터를 주고받으려면 별도의 방법이 필요하다.

이를 IPC(Inter-Process Communication)라고 한다.

대표적으로 Pipe, Socket, Shared Memory 등이 있다.

## 2. Thread

> 프로세스 내부에서 실제 작업을 실행하는 실행 단위이다.

하나의 프로세스는 하나 이상의 스레드를 가질 수 있다.

```
Process
│
├─ Thread 1
├─ Thread 2
└─ Thread 3
```

프로세스가 자원을 할당받는 단위라면, 스레드는 그 자원을 이용해 실제로 작업을 수행하는 단위이다.

### 스레드는 무엇을 공유할까?

같은 프로세스의 스레드들은 프로세스의 자원을 공유한다.

```
Process
│
├─ Code  ────────── 공유
├─ Data  ────────── 공유
├─ Heap  ────────── 공유
│
├─ Thread A
│    └─ Stack A
│
└─ Thread B
     └─ Stack B
```

스레드는 힙을 공유한다.

여러 스레드가 같은 객체에 접근할 수 있기 때문에 `race condition`과 `동기화 문제`가 발생한다.

### 스택은 왜 스레드마다 따로 가질까?

각 스레드는 서로 다른 메서드를 실행할 수 있다.

```
Thread A
→ methodA()
→ methodB()

Thread B
→ methodC()
```

함수 호출 순서와 지역 변수 등이 서로 다르기 때문에 각 스레드가 자신의 스택을 가져야 한다.

```
Thread A Stack        Thread B Stack

methodB()             methodC()
methodA()
```

반대로 객체 등이 저장되는 힙은 공유하기 때문에 여러 스레드가 같은 객체를 사용할 수 있다.

### 프로세스 vs 스레드

| 구분 | Process | Thread |
|---|---|---|
| 의미 | 실행 중인 프로그램 | Process 내부의 실행 단위 |
| 메모리 | Process마다 독립 | 같은 Process의 자원 공유 |
| Code / Data / Heap | Process별로 독립 | 같은 Process 내에서 공유 |
| Stack | 내부 Thread마다 별도 | Thread마다 독립 |
| 자원 공유 | IPC 필요 | 같은 Process 내에서 쉽게 공유 |
| 생성/전환 비용 | 상대적으로 큼 | 상대적으로 작음 |
| 장애 영향 | 다른 Process와 격리 | 같은 Process에 영향을 줄 수 있음 |

스레드는 자원을 쉽게 공유할 수 있다는 장점이 있지만, 공유하기 때문에 동시성 문제도 발생할 수 있다.

```
공유가 쉬움
→ 성능/효율 측면의 장점

하지만
→ 동시에 같은 데이터 수정
→ Race Condition 가능
```

## 3. 컨텍스트 스위칭

> CPU가 현재 실행 중인 프로세스 또는 스레드를 멈추고 다른 프로세스나 스레드를 실행하기 위해 실행 상태를 저장하고 복원하는 과정이다.

예를 들어 하나의 CPU 코어에서 아래와 같은 전환이 발생할 수 있다.

```
Thread A 실행
     ↓
A의 상태 저장
     ↓
B의 상태 복원
     ↓
Thread B 실행
```

이 과정을 컨텍스트 스위칭이라고 한다.

### 컨텍스트에는 무엇이 들어갈까?

스레드가 나중에 다시 실행되려면 어디까지 실행했는지 등의 상태를 기억해야 한다.

```
Thread A 실행

↓ Context 저장

Thread B 실행

↓ Context 저장

Thread A Context 복원

↓
Thread A가 이전 위치부터 다시 실행
```

### 컨텍스트 스위칭은 왜 비용이 발생할까?

컨텍스트 스위칭 중에는 실제 애플리케이션 작업 대신 실행 상태를 저장하고 다른 실행 상태를 복원하는 작업이 필요하다.

또, CPU 캐시 관점에서도 추가적인 비용이 발생할 수 있다.

따라서 스레드를 많이 생성한다고 해서 무조건 성능이 좋아지는 것은 아니다.

```
Thread 증가
→ 동시에 처리할 수 있는 작업 증가 가능

하지만 너무 많으면
→ Context Switching 증가
→ 메모리 사용 증가
→ 오히려 성능 저하 가능
```

### 프로세스와 스레드의 컨텍스트 스위칭

일반적으로 프로세스 간 컨텍스트 스위칭이 스레드 간 컨텍스트 스위칭보다 더 많은 비용이 발생할 수 있다.

프로세스는 서로 독립된 주소 공간을 가지지만, 같은 프로세스의 스레드들은 코드, 데이터, 힙 등의 자원을 공유하기 때문이다.

다만 스레드 전환도 컨텍스트 스위칭 비용이 없는 것은 아니다.

## 4. 동시성/병렬성

### 동시성

> 여러 작업이 같은 시간 동안 번갈아 실행되면서 동시에 진행되는 것처럼 보이는 것이다.

싱글 코어에서도 가능하다.

```
Task A 실행
    ↓
Task B 실행
    ↓
Task A 실행
    ↓
Task B 실행
```

CPU가 빠르게 실행 대상을 전환하면서 여러 작업을 조금씩 진행한다.

```
시간 ─────────────────────→

CPU  A A | B B | A A | B B
```

### 병렬성

> 여러 작업이 실제로 같은 시점에 실행되는 것이다.

이를 위해서는 실제로 여러 작업을 동시에 실행할 수 있는 여러 CPU 코어가 필요하다.

```
시간 ─────────────────────→

Core 1    A A A A A
Core 2    B B B B B
```

### 동시성 vs 병렬성

| 구분 | 동시성(Concurrency) | 병렬성(Parallelism) |
|---|---|---|
| 핵심 | 여러 작업을 번갈아 처리 | 여러 작업을 실제 동시에 처리 |
| Single Core | 가능 | 불가능 |
| Multi Core | 가능 | 가능 |
| 목적 | 여러 작업을 효율적으로 다룸 | 작업 처리 속도 향상 |

동시성과 병렬성이 서로 반대되는 개념은 아니다.

멀티 코어 환경에서는 여러 작업을 동시성 있게 관리하면서 그중 일부가 실제로 병렬 실행될 수도 있다.

## 5. Race Condition

> 여러 스레드가 공유 자원에 동시에 접근하면서 실행 순서에 따라 결과가 달라지는 문제이다.

예를 들어 두 스레드가 같은 count를 증가시킨다고 가정하자.

```java
count++;

1. count 값을 읽음
2. count + 1 계산
3. 결과를 count에 저장
```

겉으로는 단순한 한 줄이지만 개념적으로는 위의 단계처럼 여러 단계의 연산으로 이루어진다.

### count++는 왜 안전하지 않을까?

처음 count = 10이라고 가정하자.

정상적으로 두 번 증가했다면 결과는 12여야 한다.

그러나 아래와 같이 최종 결과가 11이 될 수 있다.

```
Thread A → count 읽음 → 10
Thread B → count 읽음 → 10

Thread A → 10 + 1 → 11
Thread B → 10 + 1 → 11

Thread A → 11 저장
Thread B → 11 저장

최종 결과 → 11
```

두 스레드가 동시에 같은 값을 읽고 수정하면서 한 번의 증가가 사라졌다.

이처럼 실행 순서에 따라 결과가 달라질 수 있는 상황이 Race condition이다.

### 임계 영역

여러 스레드가 동시에 접근했을 때 문제가 발생할 수 있는 코드 영역을 임계 영역이라고 한다.

공유 자원을 안전하게 변경하려면 임계 영역에 여러 스레드가 동시에 들어가지 못하도록 제어해야 한다.

이때 사용할 수 있는 대표적인 방법이 Mutex, Semaphore, synchronized, Atomic 등이다.

```
공유 자원
   ↓
Race Condition 발생 가능
   ↓
Critical Section 보호 필요
   ↓
동기화(Synchronization)
```

## 6. Mutex/Semaphore

### 뮤텍스

> 하나의 공유 자원에 하나의 스레드만 접근하도록 제한하는 방식이다.

```
Thread A ─→ 🔒 Critical Section
Thread B ─→ 대기
Thread C ─→ 대기

Thread A 작업 완료
       ↓
       🔓
       ↓
다른 Thread 접근
```

락을 획득한 스레드가 임계 영역에 들어가고, 작업이 끝나면 락을 반환한다.

### 세마포어

> 동시에 접근할 수 있는 스레드의 개수를 제한한다.

예를 들어 세마포어의 permit이 3개일 때를 가정해보자.

```
Semaphore = 3

Thread A → 접근 O
Thread B → 접근 O
Thread C → 접근 O
Thread D → 대기
Thread E → 대기
```

누군가 작업을 끝내고 permit을 반환하면 다른 스레드가 접근할 수 있다.

### 뮤텍스 vs 세마포어

| 구분 | Mutex | Semaphore |
|---|---|---|
| 동시 접근 | 하나만 허용 | 정해진 개수만큼 허용 |
| 핵심 목적 | 상호 배제 | 동시 접근 개수 제한 |
| 상태 | Lock / Unlock | Permit 개수 관리 |
| 사용 예 | 하나의 공유 데이터 보호 | 제한된 자원 사용량 제어 |

예를 들어 하나의 공유 데이터를 한 스레드만 수정하도록 보호하고 싶다면 뮤텍스 개념이 적합하다.

반대로 사용할 수 있는 자원이 10개이고, 동시에 최대 10개 작업만 허용하고 싶다면 세마포어가 적합하다.

### 자바에서의 사용

자바에서도 세마포어를 직접 제공한다.

```java
Semaphore semaphore = new Semaphore(3);

semaphore.acquire();

try {
    // 동시에 최대 3개의 Thread만 실행
} finally {
    semaphore.release();
}
```

뮤텍스에 해당하는 상호 배제는 synchronized와 Lock 등을 통해 구현할 수 있다.

## 7. Java Thread

자바에서는 하나의 JVM 프로세스 안에서 여러 스레드를 실행할 수 있다.

자바 프로그램을 실행하면 가장 먼저 Main 스레드가 실행된다.

```java
public static void main(String[] args) {
    System.out.println(Thread.currentThread().getName());
}
```

여기서 새로운 스레드를 생성하면 여러 작업을 별도의 스레드에서 수행할 수 있다.

```
JVM Process
│
├─ Main Thread
├─ Thread A
└─ Thread B

Code / Heap 등은 공유
Stack은 Thread마다 독립
```

### 스레드 생성과 실행

간단하게는 스레드 객체를 만들어 실행할 수 있다.

```java
Thread thread = new Thread(() -> {
    System.out.println("작업 실행");
});

thread.start();
```

Runnable은 스레드가 수행할 작업을 표현한다.

```java
Runnable task = () -> {
    System.out.println("작업 실행");
};

Thread thread = new Thread(task);
thread.start();
```

### start() vs run()

thread.start()를 호출하면 새로운 스레드가 생성되어 해당 스레드에서 run()을 실행한다.

thread.run()을 직접 호출하면 단순한 메서드 호출이므로 현재 스레드에서 실행된다.

따라서 새로운 스레드에서 작업을 실행하기 위해서는 start()를 호출해야 한다.

### 스레드 상태

자바 스레드는 실행 과정에서 여러 상태를 가진다.

```
NEW
 ↓ start()
RUNNABLE
 ↓
WAITING / TIMED_WAITING / BLOCKED
 ↓
RUNNABLE
 ↓
TERMINATED

- NEW: 생성되었지만 아직 시작하지 않음
- RUNNABLE: 실행 가능하거나 실행 중
- BLOCKED: Monitor Lock을 얻기 위해 대기
- WAITING / TIMED_WAITING: 다른 Thread 또는 일정 시간을 기다림
- TERMINATED: 실행 종료
```

### 스레드를 계속 생성하면 안 될까?

요청이 들어올 때마다 새로운 스레드를 만든다고 가정하자.

```
요청 1 → Thread 생성
요청 2 → Thread 생성
요청 3 → Thread 생성
요청 4 → Thread 생성
...
```

스레드 생성과 제거에는 비용이 발생하며, 스레드마다 스택 등의 메모리도 필요하다.

스레드가 지나치게 많아지면

```
Thread 증가
→ 메모리 사용 증가
→ Context Switching 증가
→ 성능 저하 가능
```

그래서 실제 서버에서는 필요한 작업마다 스레드를 계속 새로 만들기보다 스레드를 미리 만들어 재사용하는 스레드 풀을 사용한다.

## 8. Thread Pool

> 미리 일정한 수의 스레드를 만들어두고, 들어오는 작업을 스레드에게 할당하여 재사용하는 방식이다.

```
         Task Queue
        [A][B][C][D]
              ↓
      ┌───────────────┐
      │  Thread Pool  │
      │               │
      │  Worker 1     │
      │  Worker 2     │
      │  Worker 3     │
      └───────────────┘
```

Worker Thread가 작업을 하나 처리한 뒤 사라지는 것이 아니라 다음 작업을 다시 처리한다.

### 왜 스레드 풀을 사용할까?

**스레드 생성 비용 감소**

매번 아래와 같은 작업을 하지 않고 기존 스레드를 재사용한다.

```
Thread 생성
→ 작업
→ Thread 제거
```

**스레드 개수 제한**

무제한으로 스레드가 만들어지는 것도 막을 수 있다.

```
요청 1,000개

Thread 1,000개 생성 ❌

일정한 Worker Thread
       +
나머지 작업은 Queue에서 대기
```

즉, 스레드 풀은 단순히 스레드를 재사용하는 것뿐만 아니라 동시에 실행되는 스레드 수를 제어하여 시스템 자원을 관리하는 역할도 한다.

### 자바의 ExecutorService

자바에서는 직접 스레드를 관리하기보다 ExecutorService를 이용할 수 있다.

```java
ExecutorService executor =
        Executors.newFixedThreadPool(4);

executor.submit(() -> {
    System.out.println("작업 실행");
});

executor.shutdown();
```

여기서는 스레드를 직접 생성하는 것이 아니라 실행할 작업을 스레드 풀에 제출한다.

아래와 같이 작업과 스레드 관리를 분리할 수 있다.

```
Task 제출
   ↓
Thread Pool
   ↓
사용 가능한 Worker Thread
   ↓
Task 실행
```

### 스레드 풀의 크기는 클수록 좋을까?

그렇지 않다.

스레드가 너무 적으면 아래와 같다.

```
작업 많음
→ 사용할 Thread 부족
→ Queue 대기 증가
```

반대로 너무 많으면 아래와 같다.

```
Thread 과다
→ 메모리 사용 증가
→ Context Switching 증가
→ 오히려 성능 저하
```

따라서 작업의 특성과 시스템 자원을 고려해 적절한 스레드 수를 설정하는 것이 중요하다.

CPU 연산이 많은 작업과 DB/API 호출처럼 대기 시간이 많은 작업은 적절한 Thread Pool 크기도 달라질 수 있다.

## 9. synchronized

> 여러 스레드가 동시에 공유 자원에 접근하지 못하도록 임계 영역을 보호하는 자바의 동기화 방법이다.

여러 스레드가 동시에 실행할 경우 값이 유실될 수 있는 문제를 synchronized로 보호할 수 있다.

```java
public synchronized void increment() {
    count++;
}
```

한 스레드가 실행 중이라면 같은 Lock이 필요한 다른 스레드는 기다린다.

```
Thread A → Lock 획득 → count++ → Lock 반환
Thread B → 대기 ────────────────→ Lock 획득
```

### Monitor Lock

자바의 모든 객체는 모니터 락과 연결될 수 있다.

synchronized 영역에 들어가기 위해서는 해당 객체의 모니터 락을 획득해야 한다.

```java
synchronized (lock) {
    count++;
}
```

```
Thread A
→ lock 획득
→ Critical Section 실행

Thread B
→ 같은 lock 필요
→ A가 반환할 때까지 대기
```

중요한 것은 synchronized라는 키워드 자체보다 어떤 객체를 Lock으로 사용하느냐다.

서로 다른 Lock을 사용한다면 동시에 실행될 수 있다.

### synchronized 메서드 vs 블록

메서드 전체를 동기화할 수도 있다.

```java
public synchronized void increment() {
    count++;
}
```

특정 부분만 동기화할 수도 있다.

```java
public void increment() {

    // 동기화가 필요 없는 작업

    synchronized (this) {
        count++;
    }
}
```

임계 영역이 너무 크면 다른 스레드가 기다리는 시간도 길어진다.

따라서 공유 자원을 보호하는 데 필요한 범위만 동기화하는 것이 중요하다.

### synchronized의 단점

synchronized는 Race Condition을 방지할 수 있지만 동시 접근을 제한한다.

```
Thread A → Lock 획득
Thread B → 대기
Thread C → 대기
```

락 경쟁이 심하면 스레드의 대기 시간이 증가하면서 성능이 저하될 수 있다.

또한 여러 락을 잘못 사용하는 경우 데드락 같은 문제도 발생할 수 있다.

따라서 모든 코드에 무조건 synchronized를 적용하는 것이 아니라 실제로 보호해야 하는 공유 자원의 범위를 판단해야 한다.

## 10. Atomic

Java는 간단한 연산을 원자적(Atomic)으로 처리할 수 있는 클래스를 제공한다.

```java
AtomicInteger
AtomicLong
AtomicBoolean
AtomicReference

AtomicInteger count = new AtomicInteger(0);

count.incrementAndGet();
```

incrementAndGet()은 증가 연산을 원자적으로 수행하기 때문에 여러 스레드가 동시에 접근해도 증가 결과가 유실되지 않는다.

### Atomic Operation이란?

Atomic은 더 이상 중간 단계로 나뉘어 다른 스레드가 끼어들 수 없는 하나의 연산처럼 처리되는 것을 의미한다.

### CAS

자바의 Atomic 클래스는 주로 CAS(Compare-And-Set)를 이용한다.

```
현재 값 == 내가 예상한 값?

YES
→ 새로운 값으로 변경

NO
→ 다른 Thread가 먼저 변경
→ 다시 시도
```

예를 들어 아래와 같으면 성공이다.

```
현재 값 = 10
예상 값 = 10

→ 11로 변경 성공
```

그런데 다른 스레드가 먼저 값을 바꿨다면 아래와 같이 진행된다.

```
현재 값 = 11
예상 값 = 10

→ 변경 실패
→ 새로운 값을 확인하고 재시도
```

락을 획득하고 다른 스레드를 대기시키는 방식과는 차이가 있다.

### synchronized vs Atomic

| 구분 | synchronized | Atomic |
|---|---|---|
| 방식 | Lock 기반 | 주로 CAS 기반 |
| 다른 Thread | Lock 획득까지 대기 가능 | 실패하면 재시도 |
| 적합한 경우 | 여러 연산을 묶어 보호 | 단순한 하나의 값 변경 |
| 예시 | 여러 공유 데이터의 일관된 변경 | 카운터 증가 |

아래와 같이 여러 연산을 하나의 임계 영역으로 묶어야 한다면 Atomic 변수 하나만으로 해결할 수 없다.

```
잔액 확인
→ 잔액 차감
→ 주문 상태 변경
```

즉, Atomic을 사용한다고 모든 동시성 문제가 해결되는 것은 아니다.

## 11. WAS Thread

### WAS에서 Thread는 어떤 역할을 할까?

Spring Boot의 기본적인 Servlet 기반 웹 애플리케이션에서는 HTTP 요청이 들어오면 WAS의 Worker Thread가 요청을 할당받아 처리한다.

Spring Boot에서 흔히 사용하는 Tomcat을 예로 들면 아래와 같다.

```
Client
  ↓
HTTP Request
  ↓
Tomcat
  ↓
Thread Pool
  ↓
Worker Thread
  ↓
Controller
  ↓
Service
  ↓
Repository
  ↓
HTTP Response
```

```java
@GetMapping("/users")
public List<User> getUsers() {
    return userService.findAll();
}
```

위와 같은 코드도 일반적으로 Spring MVC 환경에서는 Tomcat의 Worker Thread가 실행한다.

### 요청마다 새로운 스레드를 만들까?

그렇지 않다. WAS도 Thread Pool을 사용한다.

```
HTTP Request
     ↓
┌───────────────────┐
│    Thread Pool    │
│                   │
│ Thread 1 → 요청 A  │
│ Thread 2 → 요청 B  │
│ Thread 3 → 요청 C  │
└───────────────────┘
```

요청 처리가 끝난 스레드는 사라지는 것이 아니라 Pool로 돌아가 다음 요청을 처리하는 데 재사용된다.

### 요청 처리 중 DB가 느리다면?

예를 들어 요청을 처리하다 DB Query를 실행했다고 가정하자.

```
Thread 1
   ↓
Controller
   ↓
Service
   ↓
DB Query
   ↓
DB 응답 대기...
```

일반적인 Blocking 방식에서는 DB 응답이 올 때까지 해당 요청을 담당하는 스레드가 점유된 상태로 대기한다.

외부 API 호출도 마찬가지이다.

```
요청 증가
   ↓
DB / 외부 API 응답 지연
   ↓
많은 Thread가 대기
   ↓
사용 가능한 Thread 감소
```


### Thread Pool 고갈

처리 시간이 긴 요청이 계속 쌓이면 모든 Worker Thread가 사용 중인 상황이 발생할 수 있다.

```
Thread Pool

Thread 1 → DB 대기
Thread 2 → DB 대기
Thread 3 → 외부 API 대기
Thread 4 → 작업 중

새로운 요청
    ↓
사용 가능한 Thread 없음
    ↓
대기
```

이를 Thread Pool Exhaustion(고갈)이라고 한다.

스레드가 모두 점유되면 새로운 요청을 즉시 처리할 수 없어 응답 지연이나 처리량 저하로 이어질 수 있다.

그래서 단순히 Thread Pool 크기를 크게 만드는 것뿐 아니라 아래와 같이 스레드를 오래 점유하게 만드는 원인도 함께 확인해야 한다.

```
느린 DB Query
외부 API 지연
과도한 Blocking
적절하지 않은 Timeout
```

### WAS Thread Pool과 DB Connection Pool

둘은 서로 다른 Pool이지만 실제 서버에서는 밀접하게 연결된다.

```
HTTP Request
     ↓
WAS Thread Pool
     ↓
Worker Thread
     ↓
DB Connection 필요
     ↓
DB Connection Pool
     ↓
Database
```

```
WAS Worker Thread = 많음

DB Connection = 적음
```

위와 같은 상황에서 DB 요청이 몰리면 일부 스레드는 DB Connection을 얻기 위해 대기할 수 있다.

반대로 Connection 수를 무조건 크게 늘리는 것도 DB가 감당해야 할 동시 작업량을 증가시킬 수 있다.

따라서 서버 성능을 볼 때는 WAS Thread Pool만 독립적으로 보기보다 DB Connection Pool과 실제 DB 처리 능력까지 함께 고려해야 한다.
