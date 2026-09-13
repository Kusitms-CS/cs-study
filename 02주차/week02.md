# 운영체제 - Process & Thread / 동시성

## 1. Process

### 1.1 프로세스란?

**프로세스(Process)** = 실행 중인 프로그램

- 프로그램: 디스크에 저장된 실행 가능한 정적 파일
- 프로세스: 프로그램이 실행되어 메모리에 올라간 상태
- 운영체제로부터 CPU, 메모리 등의 자원을 할당받음
- 각 프로세스는 기본적으로 독립적인 메모리 공간을 가짐
- 다른 프로세스의 메모리에 직접 접근할 수 없음

```text
Program
   ↓ 실행
Process
```

### 1.2 프로세스 메모리 구조

```text
┌─────────────────┐
│      Stack      │
├─────────────────┤
│                 │
│       Heap      │
├─────────────────┤
│       Data      │
├─────────────────┤
│       Code      │
└─────────────────┘
```

| 영역 | 내용 |
|---|---|
| Code | 실행할 프로그램의 코드 |
| Data | 전역 변수, static 변수 |
| Heap | 동적으로 할당되는 메모리 |
| Stack | 함수 호출 정보, 지역 변수, 매개변수 |

프로세스마다 독립적인 Code, Data, Heap, Stack 영역을 가짐

---

### 1.3 프로세스 상태

```text
New → Ready → Running → Terminated
        ↑        ↓
        └── Waiting
```

- **New**: 프로세스 생성 중
- **Ready**: CPU 할당 대기 중
- **Running**: CPU를 할당받아 실행 중
- **Waiting / Blocked**: I/O 등의 작업 완료 대기 중
- **Terminated**: 실행 종료

CPU는 한정되어 있으므로 운영체제의 스케줄러가 실행할 프로세스를 선택함

---

### 1.4 PCB

**PCB(Process Control Block)** = 운영체제가 프로세스를 관리하기 위해 사용하는 자료구조

주요 저장 정보

- Process ID
- Process State
- Program Counter
- CPU Register
- Scheduling 정보
- Memory 관리 정보

프로세스가 CPU 사용을 중단할 때 현재 실행 정보를 PCB에 저장함

이후 다시 CPU를 할당받으면 PCB의 정보를 복원하여 이전 실행 위치부터 이어서 실행 가능

---

# 2. Thread

## 2.1 스레드란?

**스레드(Thread)** = 프로세스 내부에서 실제 작업을 수행하는 실행 단위

```text
Process
│
├── Thread 1
├── Thread 2
└── Thread 3
```

하나의 프로세스는 여러 개의 Thread를 가질 수 있음

같은 프로세스의 Thread끼리는 일부 메모리를 공유함

```text
Process

┌─────────────────────┐
│        Code         │ ← 공유
├─────────────────────┤
│        Data         │ ← 공유
├─────────────────────┤
│        Heap         │ ← 공유
├─────────────────────┤
│   Thread 1 Stack    │ ← 독립
├─────────────────────┤
│   Thread 2 Stack    │ ← 독립
└─────────────────────┘
```

### 공유 영역

- Code
- Data
- Heap

### 독립 영역

- Stack
- Register
- Program Counter

공유 메모리를 사용하므로 Thread 간 데이터 공유가 쉬움

반면 여러 Thread가 동일한 데이터에 접근할 수 있어 동시성 문제 발생 가능

---

## 2.2 Process와 Thread 비교

| 구분 | Process | Thread |
|---|---|---|
| 의미 | 실행 중인 프로그램 | 프로세스 내부 실행 단위 |
| 메모리 | 독립적 | 일부 영역 공유 |
| Code / Data / Heap | 독립적 | 공유 |
| Stack | 독립적 | Thread마다 독립적 |
| 데이터 공유 | 상대적으로 어려움 | 상대적으로 쉬움 |
| 생성 비용 | 큼 | 상대적으로 작음 |
| Context Switching | 비용이 큼 | 상대적으로 작음 |
| 안정성 | 다른 프로세스에 영향 적음 | 공유 자원 문제 가능 |

---

# 3. Context Switching

## 3.1 Context Switching이란?

**Context Switching** = CPU가 현재 실행 중인 작업을 중단하고 다른 Process 또는 Thread로 실행 대상을 변경하는 과정

```text
Process A 실행
      ↓
A 실행 상태 저장
      ↓
B 실행 상태 복원
      ↓
Process B 실행
```

현재 작업의 실행 상태를 저장하고 다음 작업의 실행 상태를 복원해야 함

프로세스의 경우 PCB에 실행 상태를 저장함

---

## 3.2 Context Switching 비용

Context Switching은 실제 애플리케이션의 작업을 처리하는 시간이 아님

주요 비용

- 현재 실행 상태 저장
- 다음 실행 상태 복원
- CPU Cache 효율 저하
- 스케줄링 비용 발생

Thread를 무조건 많이 생성한다고 성능이 증가하는 것은 아님

Thread가 지나치게 많으면 Context Switching 증가 → 오히려 성능 저하 가능

---

## 3.3 Process와 Thread의 Context Switching

### Process Context Switching

서로 다른 메모리 공간을 사용하는 프로세스 간 전환

상대적으로 많은 상태 변경 필요

### Thread Context Switching

같은 프로세스 내부 Thread끼리는 Code, Data, Heap 등을 공유함

따라서 일반적으로 Process Context Switching보다 비용이 적음

---

# 4. 동시성과 병렬성

## 4.1 동시성

**Concurrency(동시성)** = 여러 작업을 번갈아 실행하여 동시에 진행되는 것처럼 처리하는 방식

Single Core에서도 가능

```text
시간 →

Thread A ███     ███
Thread B    ███     ███
```

실제로 한 순간에는 하나의 작업만 실행될 수 있지만 매우 빠르게 작업을 전환함

핵심

> 여러 작업을 번갈아 처리

---

## 4.2 병렬성

**Parallelism(병렬성)** = 여러 작업을 실제로 같은 순간에 실행하는 방식

여러 CPU Core 필요

```text
Core 1 : Thread A █████████

Core 2 : Thread B █████████
```

핵심

> 여러 작업을 실제로 동시에 처리

---

## 4.3 동시성과 병렬성 비교

| 구분 | 동시성 | 병렬성 |
|---|---|---|
| 영어 | Concurrency | Parallelism |
| 핵심 | 번갈아 처리 | 실제 동시 처리 |
| Single Core | 가능 | 불가능 |
| Multi Core | 가능 | 가능 |
| 목적 | 여러 작업의 효율적 관리 | 처리 속도 향상 |

Multi Thread = 무조건 병렬 처리는 아님

Single Core에서는 여러 Thread가 동시성 방식으로 실행됨

Multi Core에서는 여러 Thread가 실제 병렬로 실행될 수도 있음

---

# 5. Race Condition

## 5.1 Race Condition이란?

**Race Condition** = 여러 Thread가 공유 데이터에 동시에 접근하여 실행 순서에 따라 결과가 달라지는 문제

```java
int count = 0;

count++;
```

`count++`은 코드상 한 줄이지만 실제로는 여러 단계로 처리될 수 있음

```text
1. count 읽기
2. count + 1 계산
3. count에 결과 저장
```

두 Thread가 동시에 접근하는 경우

```text
초기 count = 0

Thread A : count 읽기 → 0
Thread B : count 읽기 → 0

Thread A : 0 + 1 → 저장
Thread B : 0 + 1 → 저장

최종 count = 1
```

기대 결과는 `2`이지만 실제 결과는 `1`이 될 수 있음

공유 데이터에 대한 동시 접근이 원인

---

## 5.2 Critical Section

**Critical Section(임계 영역)** = 여러 Thread가 동시에 접근할 경우 문제가 발생할 수 있는 코드 영역

```text
Thread A ──┐
           ↓
    Critical Section
           ↑
Thread B ──┘
```

임계 영역에 대한 접근 제어 필요

대표적인 동기화 방법

- Mutex
- Semaphore
- synchronized
- Atomic

---

# 6. Mutex

## 6.1 Mutex란?

**Mutex(Mutual Exclusion)** = 하나의 공유 자원에 한 번에 하나의 Thread만 접근하도록 제한하는 동기화 기법

```text
Thread A
   ↓
Lock 획득
   ↓
공유 자원 접근
   ↓
Lock 해제

Thread B
   ↓
Lock 대기
   ↓
Lock 획득
   ↓
공유 자원 접근
```

한 Thread가 Lock을 획득하면 다른 Thread는 Lock이 해제될 때까지 대기

```text
lock()

Critical Section

unlock()
```

공유 자원의 상호 배제가 핵심

---

# 7. Semaphore

## 7.1 Semaphore란?

**Semaphore** = 공유 자원에 동시에 접근할 수 있는 Thread의 개수를 제한하는 동기화 기법

예를 들어 Semaphore 값이 3인 경우

```text
Semaphore = 3

Thread A → 접근 가능
Thread B → 접근 가능
Thread C → 접근 가능

Thread D → 대기
Thread E → 대기
```

기존 Thread가 작업을 완료하면 대기 중인 다른 Thread가 접근 가능

제한된 개수의 자원을 관리할 때 활용 가능

---

## 7.2 Mutex와 Semaphore 비교

| 구분 | Mutex | Semaphore |
|---|---|---|
| 접근 가능 개수 | 1개 | N개 |
| 핵심 | 상호 배제 | 접근 가능한 작업 수 제한 |
| 사용 예시 | 공유 변수 보호 | 제한된 자원 관리 |

Semaphore의 값이 1이면 Mutex와 유사한 형태로 동작 가능

단, 두 개념의 목적과 동작 방식이 완전히 동일한 것은 아님

---

# Backend 심화

# 8. Java Thread

## 8.1 Thread 생성

Java에서 `Thread` 클래스를 통해 Thread 생성 가능

```java
public class MyThread extends Thread {

    @Override
    public void run() {
        System.out.println("Thread 실행");
    }
}
```

```java
MyThread thread = new MyThread();

thread.start();
```

`start()` 호출 시 새로운 Thread 생성 후 해당 Thread에서 `run()` 실행

`run()`을 직접 호출하면 새로운 Thread가 생성되지 않고 현재 Thread에서 일반 메서드처럼 실행되므로 주의

---

## 8.2 Runnable

`Runnable` 인터페이스를 이용해 실행할 작업 정의 가능

```java
Runnable task = () -> {
    System.out.println("Thread 실행");
};

Thread thread = new Thread(task);

thread.start();
```

Thread와 실제 실행할 작업을 분리할 수 있음

실제 서버에서는 요청마다 Thread를 직접 생성하기보다 Thread Pool을 사용하는 경우가 일반적

---

# 9. Thread Pool

## 9.1 Thread Pool이란?

**Thread Pool** = 미리 일정 개수의 Thread를 생성해두고 작업이 들어오면 기존 Thread를 재사용하는 방식

```text
          Task
           ↓
      ┌─────────┐
      │  Queue  │
      └─────────┘
           ↓
┌─────────────────────┐
│     Thread Pool     │
│                     │
│  T1   T2   T3   T4 │
└─────────────────────┘
```

요청마다 새로운 Thread를 생성하면 생성/제거 비용 발생

Thread Pool 사용 시 기존 Thread 재사용 가능

### 장점

- Thread 생성/제거 비용 감소
- Thread 재사용
- 최대 Thread 개수 제한 가능
- 시스템 자원 사용량 제어 가능
- 무분별한 Thread 생성 방지

Thread가 너무 많으면 Context Switching 증가 및 메모리 사용량 증가 가능

따라서 Thread Pool 크기를 적절하게 설정하는 것이 중요함

---

## 9.2 ExecutorService

Java에서는 `ExecutorService`를 통해 Thread Pool 관리 가능

```java
ExecutorService executor =
        Executors.newFixedThreadPool(4);

executor.submit(() -> {
    System.out.println("Task 실행");
});
```

4개의 Thread를 생성하고 들어오는 작업을 처리

```text
Task 1 ─┐
Task 2 ─┤
Task 3 ─┤
Task 4 ─┼→ Thread Pool → 실행
Task 5 ─┘
```

모든 Thread가 작업 중이면 추가 작업은 Queue에서 대기

---

# 10. synchronized

## 10.1 synchronized란?

Java에서 공유 자원에 대한 동시 접근을 제어하기 위한 키워드

다음 코드는 Race Condition 발생 가능

```java
public class Counter {

    private int count = 0;

    public void increase() {
        count++;
    }
}
```

여러 Thread가 동시에 `count++` 실행 가능

`synchronized` 적용

```java
public class Counter {

    private int count = 0;

    public synchronized void increase() {
        count++;
    }
}
```

한 Thread가 메서드를 실행 중이면 Lock을 획득하지 못한 다른 Thread는 대기

---

## 10.2 synchronized block

메서드 전체가 아닌 특정 영역에만 적용 가능

```java
public void increase() {

    synchronized (this) {
        count++;
    }
}
```

동기화가 필요한 부분만 Critical Section으로 설정 가능

동기화 범위가 넓을수록 다른 Thread의 대기 시간이 증가할 수 있음

따라서 필요한 범위에만 적용하는 것이 중요함

---

# 11. Atomic

## 11.1 Atomic이란?

Java에서 원자적인 연산을 지원하는 클래스

`java.util.concurrent.atomic` 패키지에서 제공

대표적으로 `AtomicInteger` 존재

```java
AtomicInteger count = new AtomicInteger(0);

count.incrementAndGet();
```

여러 Thread가 동시에 증가 연산을 수행하더라도 안전하게 처리 가능

```java
public class Counter {

    private final AtomicInteger count =
            new AtomicInteger(0);

    public void increase() {
        count.incrementAndGet();
    }
}
```

---

## 11.2 Atomic Operation

**Atomic Operation** = 연산이 중간에 끊기지 않고 하나의 단위처럼 처리되는 연산

일반적인 증가 연산

```text
Read
 ↓
Modify
 ↓
Write
```

각 단계 사이에 다른 Thread가 접근하면 Race Condition 발생 가능

Atomic 클래스는 이러한 연산을 원자적으로 처리하도록 지원

대표적으로 CAS(Compare-And-Set) 방식 활용

```text
현재 값 == 예상 값?

YES → 새로운 값으로 변경

NO → 다시 시도
```

단순한 숫자 증가, 감소 등의 연산에 유용

복잡한 여러 연산을 하나의 임계 영역으로 묶어야 하는 경우에는 `synchronized`나 Lock 등의 방식 필요

---

# 12. WAS Thread

## 12.1 WAS와 Thread

Spring Boot와 같은 웹 서버에서는 여러 HTTP 요청을 동시에 처리해야 함

Spring Boot에서 일반적으로 사용하는 Tomcat도 내부적으로 Thread Pool을 사용함

```text
Client A ── Request ──┐
Client B ── Request ──┤
Client C ── Request ──┤
                      ↓
               ┌─────────────┐
               │     WAS     │
               │ Thread Pool │
               └─────────────┘
                 ↓    ↓    ↓
                T1   T2   T3
```

요청마다 사용 가능한 Thread가 할당됨

Thread가 Controller → Service → Repository 등의 코드를 실행함

---

## 12.2 Spring MVC 요청 처리

일반적인 Spring MVC 요청 처리 구조

```text
HTTP Request
     ↓
WAS Thread 할당
     ↓
Controller
     ↓
Service
     ↓
Repository
     ↓
Database
     ↓
HTTP Response
     ↓
Thread Pool 반환
```

요청 처리가 끝나면 Thread가 제거되는 것이 아니라 Thread Pool로 반환됨

이후 다른 HTTP 요청 처리에 재사용됨

---

## 12.3 Blocking과 WAS Thread

DB 조회나 외부 API 요청처럼 결과를 기다려야 하는 작업 존재

```text
Thread
  ↓
Controller
  ↓
Service
  ↓
DB Query
  ↓
[ DB 응답 대기 ]
  ↓
결과 처리
  ↓
Response
```

DB 응답을 기다리는 동안 해당 Thread가 다른 요청을 처리하지 못하는 상태가 될 수 있음

이러한 작업이 많아지면 Thread Pool의 Thread가 모두 사용될 수 있음

```text
Thread Pool

T1 → DB 대기
T2 → API 대기
T3 → DB 대기
T4 → API 대기

새로운 Request
      ↓
사용 가능한 Thread 없음
      ↓
Queue에서 대기
```

따라서 백엔드 서버 성능에서 Thread Pool과 Blocking 작업 관리가 중요함

---

## 12.4 Spring Bean과 Thread Safety

Spring Bean은 기본적으로 Singleton Scope

즉, 하나의 객체를 여러 요청 Thread가 함께 사용할 수 있음

```java
@Service
public class UserService {

    private int count = 0;

    public void increase() {
        count++;
    }
}
```

```text
Thread 1 ─┐
          ↓
      UserService
          ↑
Thread 2 ─┘
```

여러 Thread가 동일한 `count`에 접근 → Race Condition 발생 가능

따라서 Singleton Bean 내부에 요청마다 변경되는 상태를 필드로 저장하는 것은 주의해야 함

```java
@Service
public class UserService {

    public int calculate(int value) {

        int result = value + 1;

        return result;
    }
}
```

지역 변수는 각 Thread의 Stack에 저장되므로 다른 Thread와 공유되지 않음

Spring의 Controller, Service 등은 가능하면 **상태를 저장하지 않는 Stateless 구조**로 설계하는 것이 안전함
