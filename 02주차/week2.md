# 2주차 - OS - Process / Thread / 동시성 개념 정리

## 목차
- [키워드 연관관계](#키워드-연관관계)
- [Process](#process)
- [Thread](#thread)
- [Context Switching](#context-switching)
- [동시성/병렬성](#동시성병렬성)
- [Race Condition](#race-condition)
- [Mutex/Semaphore](#mutexsemaphore)
- [Backend 심화 (선택)](#backend-심화-선택)
  - [Java Thread](#java-thread)
  - [Thread Pool](#thread-pool)
  - [synchronized](#synchronized)
  - [Atomic](#atomic)
  - [WAS Thread](#was-thread)

---

## 키워드 연관관계

각 개념이 독립된 게 아니라, 아래처럼 원인 → 결과 → 해결책으로 서로 이어져 있다.

```
Process
 └─ Thread ...................... 프로세스 안에 1개 이상 포함
     ├─ Context Switching ........ 스레드가 여러 개라 CPU가 번갈아 실행
     │    ├─ 동시성 ............... 싱글 코어에서 번갈아 처리
     │    └─ 병렬성 ............... 멀티 코어에서 실제 동시 처리
     ├─ Race Condition ........... 자원을 공유하다 동시 접근 시 충돌
     │    ├─ Mutex / Semaphore ... OS 레벨 해결책
     │    ├─ synchronized ........ Java 레벨 해결책
     │    └─ Atomic .............. Java 레벨 해결책 (락 없이 처리)
     └─ Java Thread .............. Thread를 자바 코드(Thread 클래스/Runnable)로 만든 것
          └─ Thread Pool ......... 스레드를 매번 만들지 않고 묶음으로 재사용
               └─ WAS Thread .... 요청이 올 때마다 풀에서 스레드 하나씩 배정
```

| 연결 | 관계 설명 |
|---|---|
| Process → Thread | 프로세스 안에 하나 이상의 스레드가 존재 |
| Thread → Context Switching | 스레드가 여러 개일 때 CPU가 번갈아 실행하며 전환 발생 |
| Context Switching → 동시성/병렬성 | 싱글 코어면 동시성, 멀티 코어면 병렬성으로 이어짐 |
| Thread → Race Condition | 스레드끼리 Code/Data/Heap을 공유하기 때문에 동시 접근 시 충돌 발생 |
| Race Condition → Mutex/Semaphore, synchronized, Atomic | 임계 구역을 보호해서 충돌을 막는 해결책 (OS 레벨 vs Java 레벨) |
| Thread → Java Thread | 같은 Thread 개념을 자바 코드로 만들고 제어하는 것 (OS 개념의 언어 레벨 구현체) |
| Java Thread → Thread Pool → WAS Thread | 스레드를 직접 만드는 대신 풀로 재사용, WAS는 요청마다 풀에서 스레드를 꺼내 씀 |

---

## Process

> EX) 실행 중인 프로그램 하나. 

운영체제로부터 자원(CPU, 메모리)을 할당받아 실행되는 **프로그램의 실행 단위**.

- 프로세스마다 Code, Data, Heap, Stack 영역을 독립적으로 가짐
- 프로세스끼리는 메모리를 공유하지 않음 → 서로 통신하려면 IPC(프로세스 간 통신, 예: 파이프, 소켓)가 필요
- 하나의 프로세스 안에는 최소 1개 이상의 스레드가 존재

---

## Thread

> EX) 한 집(프로세스) 안에 사는 가족 구성원들. 거실, 주방(Code/Data/Heap)은 같이 쓰지만, 각자 자기 방(Stack)은 따로 있다.

프로세스 내에서 실제로 실행되는 **흐름의 단위**. 하나의 프로세스는 여러 스레드를 가질 수 있다(멀티스레드).

| 영역 | 공유 여부 |
|---|---|
| Code, Data, Heap | 같은 프로세스 내 스레드끼리 공유 |
| Stack | 스레드마다 독립적으로 가짐 |

- 자원을 공유하기 때문에 통신(데이터 주고받기)은 쉽지만, 동시에 같은 데이터를 건드리면 Race Condition 같은 동시성 문제가 생길 수 있음

---

## Context Switching


CPU가 실행 중인 프로세스/스레드를 전환할 때, 현재 상태(레지스터, 프로그램 카운터 등)를 저장하고 다음 실행할 대상의 상태를 불러오는 작업.

- 멀티태스킹을 가능하게 해주지만, 전환 자체에 비용(오버헤드)이 든다 → 너무 잦으면 오히려 성능이 떨어짐
- Thread Context Switching은 Code/Data/Heap을 공유하므로, Process Context Switching보다 교체할 정보가 적어 더 가볍다

---

## 동시성/병렬성

> EX)
> - **동시성**: 요리사 한 명이 여러 요리를 빠르게 번갈아 가며 하는 것 — 실제로는 한 순간엔 한 가지만 하지만, 동시에 하는 것처럼 보임

| 구분 | 필요 조건 | 특징 |
|---|---|---|
| 동시성(Concurrency) | 싱글 코어에서도 가능 | Context Switching으로 여러 작업을 번갈아 처리 |
| 병렬성(Parallelism) | 멀티 코어 필요 | 여러 작업을 실제로 같은 시각에 처리 |

- 멀티코어 환경에서는 동시성과 병렬성이 함께 일어날 수 있다 (여러 코어가 각각 여러 작업을 번갈아 처리)

---

## Race Condition

> EX) 잔액 100만원인 계좌에서 두 사람이 동시에 "잔액 확인 → 10만원 출금 → 잔액 저장"을 하면, 둘 다 100만원을 확인하고 각자 90만원으로 계산해서 저장한다. 실제로는 20만원이 빠져야 하는데, 최종 잔액은 90만원으로 남아버리는 문제가 생긴다.

여러 스레드/프로세스가 **같은 공유 자원**에 동시에 접근해 값을 읽고 쓸 때, 실행 순서(타이밍)에 따라 결과가 달라지는 문제.

- 원인: 공유 자원에 접근하는 코드 구간(Critical Section, 임계 구역)이 보호되지 않음
- 해결: `Mutex`/`Semaphore`, `synchronized`, `Atomic` 연산 등으로 임계 구역을 한 번에 하나의 스레드만 실행하도록 보호

---

## Mutex/Semaphore

> EX)
> - **Mutex**: 열쇠가 하나뿐인 공용 화장실. 한 사람이 들어가 문을 잠그면(lock), 그 사람이 나올 때까지(unlock) 다른 사람은 못 들어간다.
> - **Semaphore**: 주차 공간이 N개인 주차장. 정해진 자리 수만큼만 동시에 들어갈 수 있고, 다 차면 빈자리가 날 때까지 기다린다.

| 구분 | 동시 접근 허용 | 특징 |
|---|---|---|
| Mutex | 1개 | lock을 건 스레드만 unlock 가능 (소유 개념 O) |
| Semaphore | N개 (카운트 지정) | 누구나 반납(release) 가능, 자원 개수를 카운트로 관리 |

---

## Backend 심화 (선택)

### Java Thread

- 자바에서 스레드를 만드는 방법은 크게 두 가지
   1. `Thread` 클래스를 상속받기
   2. `Runnable`을 구현해서 `Thread`에 넘기는 것.

---

### Thread Pool

> EX) 일이 생길 때마다 직원을 새로 뽑고 끝나면 바로 자르는 대신, 미리 몇 명을 채용해두고(Pool) 일이 들어오면 시키고 끝나면 다시 대기시키는 것.

스레드를 매번 새로 생성/소멸하는 비용을 줄이기 위해, 미리 만들어둔 스레드 묶음을 재사용하는 방식.

- 자바에서는 `java.util.concurrent.ExecutorService`로 구현
- WAS(톰캣 등)도 내부적으로 요청을 처리할 스레드 풀을 미리 만들어두고 재사용한다

---

### synchronized

- `synchronized`가 붙은 코드를 한 스레드가 실행하는 동안, 다른 스레드는 그 코드에 못 들어오고 밖에서 기다린다.

---

### Atomic

> EX) `count++`는 사실 "읽기 → 1 더하기 → 쓰기" 세 단계로 이루어져 있어서, 여러 스레드가 동시에 실행하면 Race Condition이 생긴다. `Atomic` 클래스는 이 세 단계를 "한 번에 끝나는 하나의 동작"처럼 처리해서, 중간에 다른 스레드가 끼어들 틈을 없앤다.


- `java.util.concurrent.atomic` 패키지 (`AtomicInteger`, `AtomicLong` 등)
- 내부적으로 `CAS(Compare-And-Swap)` 연산을 사용해 락 없이(lock-free) 동시성 문제를 해결 → `synchronized`보다 성능이 좋은 경우가 많음

---

### WAS Thread

> EX) 톰캣 같은 WAS(Web Application Server)는 손님(요청)이 올 때마다 직원(스레드) 한 명을 붙여서 응대하는 식으로 동작한다(Thread-per-request 모델).

- 요청이 들어오면 Thread Pool에서 놀고 있는 스레드를 꺼내 배정하고, 스레드가 다 차면 요청은 대기 큐에서 기다린다
- 스레드 수를 너무 적게 잡으면 요청 처리가 밀리고, 너무 많이 잡으면 Context Switching 비용과 메모리 사용량이 늘어난다

