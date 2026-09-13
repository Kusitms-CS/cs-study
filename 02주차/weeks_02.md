# 2주차 - OS - Process / Thread / 동시성

## 1. Process

Process(프로세스)는 **실행 중인 프로그램**이다. 디스크에 있는 프로그램 파일이 메모리에 올라가 CPU에서 실행되면 프로세스가 된다.

각 프로세스는 다른 프로세스와 분리된 가상 주소 공간을 가진다.

```
프로세스 A                         프로세스 B
Code / Data / Heap / Stack         Code / Data / Heap / Stack
        독립된 메모리 공간                  독립된 메모리 공간
```

### 프로세스의 메모리 구조

| 영역 | 역할 |
|---|---|
| Code | 실행할 기계어 코드 |
| Data | 전역 변수, static 변수 |
| Heap | `new` 등으로 동적 할당한 객체 |
| Stack | 함수 호출 정보, 지역 변수, 매개변수 |

프로세스끼리는 기본적으로 메모리를 공유하지 않는다. 그래서 한 프로세스의 오류가 다른 프로세스에 미치는 영향을 줄일 수 있지만, 데이터를 주고받으려면 IPC(파이프, 소켓, 공유 메모리 등)가 필요하다.

### 프로세스 상태

```
new → ready → running → waiting → ready → ... → terminated
```

- **ready**: CPU를 배정받기를 기다리는 상태
- **running**: CPU에서 실행 중인 상태
- **waiting (blocked)**: I/O 완료, 락 획득 같은 이벤트를 기다리는 상태

`waiting` 상태는 CPU를 기다리는 것이 아니라 이벤트를 기다리므로, 이벤트가 끝나기 전에는 실행될 수 없다.

## 2. Thread

Thread(스레드)는 프로세스 안에서 실제로 실행되는 작업 단위다. 하나의 프로세스는 최소 하나의 스레드를 가지며 여러 스레드를 가질 수 있다.

같은 프로세스의 스레드는 Code, Data, Heap을 공유하지만 각자 Stack과 레지스터 값(Program Counter 포함)은 따로 가진다.

```
하나의 프로세스
 ├─ 공유: Code, Data, Heap, 파일/소켓 같은 자원
 ├─ Thread 1: Stack 1, PC 1, Registers 1
 └─ Thread 2: Stack 2, PC 2, Registers 2
```

| 구분 | Process | Thread |
|---|---|---|
| 메모리 | 다른 프로세스와 분리 | 같은 프로세스 내에서 대부분 공유 |
| 생성/전환 비용 | 상대적으로 큼 | 상대적으로 작음 |
| 통신 | IPC 필요 | 공유 메모리로 직접 접근 가능 |
| 장애 영향 | 대체로 프로세스 단위로 격리 | 잘못된 공유 메모리 접근이 전체 프로세스에 영향 |

스레드는 공유 메모리 덕분에 빠르게 협업할 수 있지만, 동시에 같은 데이터를 다루면 동기화 문제가 생긴다.

## 3. Context Switching

CPU 코어는 한 순간에 하나의 스레드만 실행한다. 운영체제는 실행할 스레드를 바꿀 때 현재 실행 상태를 저장하고 다음 스레드의 상태를 복원하는데, 이를 **Context Switching(문맥 교환)** 이라고 한다.

```
Thread A 실행
  → A의 PC, 레지스터 등 저장
  → 스케줄러가 다음 대상 선택
  → Thread B의 PC, 레지스터 등 복원
Thread B 실행
```

문맥에는 다음에 실행할 명령어 위치(PC), CPU 레지스터 값, 스택 포인터, 스레드 상태 등이 포함된다.

문맥 교환 자체는 사용자 기능을 수행하지 않는 비용이다. 스레드 수를 과도하게 늘리면 전환·스케줄링 비용이 커지고 캐시 효율도 나빠질 수 있다. 프로세스 전환은 주소 공간까지 바뀔 수 있어 일반적으로 스레드 전환보다 무겁다.

## 4. 동시성(Concurrency)과 병렬성(Parallelism)

### 동시성

동시성은 여러 작업이 **겹치는 시간 동안 진행되는 것처럼** 다루는 능력이다. CPU 코어가 하나여도 작업을 짧게 번갈아 실행하면 동시성을 만들 수 있다.

```
코어 1개: A → B → A → B → A
```

예: 웹 서버가 DB 응답을 기다리는 요청은 잠시 멈추고 그 사이 다른 요청을 처리한다.

### 병렬성

병렬성은 여러 작업을 **실제로 같은 시각에** 실행하는 것이다. 보통 여러 CPU 코어가 필요하다.

```
코어 1: AAAAA
코어 2: BBBBB
```

| 구분 | 동시성 | 병렬성 |
|---|---|---|
| 핵심 | 여러 일을 잘 전환하며 처리 | 여러 일을 동시에 실행 |
| 코어 수 | 1개로도 가능 | 보통 2개 이상 필요 |
| 주된 목적 | 응답성, I/O 대기 활용 | 처리량, 계산 속도 향상 |

동시성과 병렬성은 함께 사용될 수 있지만 같은 의미는 아니다.

## 5. Race Condition

Race Condition(경쟁 상태)은 여러 스레드가 공유 데이터를 동시에 읽고 수정할 때, 실행 순서에 따라 결과가 달라지는 문제다.

`count++`는 하나의 원자적 연산이 아니다. 값을 읽고, 1을 더하고, 다시 저장하는 여러 단계로 수행된다.

```java
count++; // read → add → write
```

두 스레드가 모두 `count == 0`을 읽은 뒤 각각 1을 저장하면, 두 번 증가했어야 하는 값이 1이 된다.

```
Thread A: count 읽기 (0)
Thread B: count 읽기 (0)
Thread A: 1 저장
Thread B: 1 저장
결과: 1
```

### 해결 원칙

- 공유 가변 상태를 줄인다.
- 꼭 공유해야 하면 임계 구역(critical section)을 한 번에 한 스레드만 실행하게 한다.
- 단순한 카운터나 상태 변경에는 원자 타입을 사용한다.
- 여러 자료를 함께 변경해야 하면 락으로 변경 전체를 보호한다.

## 6. Mutex와 Semaphore

### Mutex

Mutex(Mutual Exclusion)는 임계 구역에 **한 스레드만** 들어가게 하는 락이다.

```
lock 획득 → 공유 데이터 변경 → lock 해제
```

락을 얻지 못한 스레드는 락이 풀릴 때까지 기다린다. 락을 얻은 코드에서 예외가 나도 락이 해제되도록 해야 한다.

### Semaphore

Semaphore는 허용 가능한 동시 접근 수를 카운터로 관리한다.

- `acquire`: 카운터를 하나 감소시킨다. 0이면 기다린다.
- `release`: 카운터를 하나 증가시키고 기다리는 작업을 깨울 수 있다.

초기값이 1인 Semaphore는 Mutex처럼 쓸 수 있다. 초기값이 N이면 DB 커넥션 N개처럼 제한된 자원을 N개까지 동시에 사용하게 할 수 있다.

| 구분 | Mutex | Semaphore |
|---|---|---|
| 허용 진입 수 | 1개 | 0개 이상 N개 |
| 용도 | 공유 데이터 보호 | 제한된 자원 개수 관리 |

### 주의할 문제

- **Deadlock**: 서로 가진 락이 풀리기를 영원히 기다림
- **Starvation**: 특정 스레드가 계속 기회를 얻지 못함
- **Lock contention**: 많은 스레드가 하나의 락을 두고 경쟁해 성능 저하

락을 여러 개 잡아야 하면 항상 같은 순서로 획득하고, 임계 구역을 짧게 유지하는 것이 기본 원칙이다.

## 7. Java Thread

Java에서는 `Thread`를 직접 만들 수 있지만, 작업과 실행 정책을 분리하기 위해 보통 `Runnable` 또는 `ExecutorService`를 사용한다.

```java
Runnable task = () -> System.out.println("작업 실행");
Thread thread = new Thread(task);
thread.start(); // run()을 직접 호출하면 새 스레드가 아니다.
```

`start()`는 새 실행 흐름을 만들고, `run()`은 그 메서드를 현재 스레드에서 일반 호출하는 것뿐이다.

## 8. Thread Pool

Thread Pool은 미리 만든 일정 수의 스레드가 작업 큐에서 일을 꺼내 처리하는 방식이다.

```
작업 제출 → 작업 큐 → Worker Thread 1 / 2 / 3
```

매 요청마다 스레드를 새로 만들지 않아 생성 비용과 무제한 스레드 증가를 피할 수 있다.

```java
ExecutorService executor = Executors.newFixedThreadPool(4);
executor.submit(() -> doWork());
executor.shutdown();
```

풀 크기가 너무 작으면 작업이 오래 대기하고, 너무 크면 문맥 교환과 메모리 사용량이 증가한다. CPU 연산 중심인지, I/O 대기 중심인지에 따라 적절한 크기가 다르다.

## 9. synchronized와 Atomic

### synchronized

`synchronized`는 객체 모니터 락을 사용해 임계 구역을 한 스레드만 실행하도록 한다. 또한 락 해제 전의 변경이 이후 락 획득 스레드에게 보이도록 하는 메모리 가시성도 제공한다.

```java
private int count;

public synchronized void increment() {
    count++;
}
```

### Atomic

`AtomicInteger` 같은 원자 타입은 단일 값의 읽기-수정-쓰기를 락 없이 원자적으로 처리할 수 있다.

```java
private final AtomicInteger count = new AtomicInteger();

public void increment() {
    count.incrementAndGet();
}
```

단, 잔액과 거래 내역처럼 **여러 값의 일관성을 함께 지켜야 하는 작업**은 Atomic 하나로 해결되지 않는다. 이때는 하나의 락으로 전체 변경을 보호하거나 트랜잭션 같은 더 적절한 경계를 사용해야 한다.

## 10. WAS Thread

WAS(Web Application Server)는 요청을 처리할 때 보통 스레드 풀의 워커 스레드를 사용한다.

```
HTTP 요청 → WAS의 요청 스레드 → Controller → Service → DB/외부 API → 응답
```

요청 처리 스레드에서 DB나 외부 API를 오래 기다리면 해당 스레드는 다른 요청을 처리하지 못한다. 모든 요청이 같은 풀을 사용한다면, 느린 의존성 하나가 풀을 고갈시켜 서버 전체 응답 지연으로 이어질 수 있다.

- 요청 스레드에서 긴 블로킹 작업을 피한다.
- DB 커넥션 풀과 WAS 스레드 풀 크기를 함께 고려한다.
- 타임아웃을 설정해 무한 대기를 막는다.
- 공유 싱글턴 객체에 요청별 상태를 보관하지 않는다.

## 11. JavaScript Single Thread와 Web Worker

브라우저의 JavaScript 실행은 일반적으로 하나의 메인 스레드에서 이벤트 루프를 통해 이루어진다. 긴 계산을 메인 스레드에서 수행하면 렌더링과 사용자 입력 처리도 함께 멈춘다.

```javascript
// 긴 반복문이 메인 스레드를 점유하면 화면이 멈출 수 있다.
for (let i = 0; i < 1_000_000_000; i++) {
  // expensive work
}
```

비동기 API가 있다고 해서 JavaScript 코드가 여러 줄에서 동시에 실행되는 것은 아니다. 네트워크나 타이머 작업은 브라우저가 처리하고, 완료 콜백은 이벤트 루프를 통해 메인 스레드에서 실행된다.

### Web Worker

Web Worker는 별도 스레드에서 JavaScript를 실행해 무거운 계산이 UI를 막지 않게 한다.

```javascript
const worker = new Worker("worker.js");
worker.postMessage({ numbers });
worker.onmessage = (event) => console.log(event.data);
```

Worker는 DOM에 직접 접근할 수 없다. 메인 스레드와는 `postMessage`로 데이터를 주고받으며, 전달 방식에 따라 데이터 복사 비용이 생길 수 있다.

## 12. 브라우저 Multi-Process 구조

현대 브라우저는 안정성과 보안을 위해 여러 프로세스를 사용한다. 구현은 브라우저마다 다르지만 보통 다음 역할이 분리된다.

| 프로세스/구성 요소 | 역할 |
|---|---|
| Browser Process | 탭, 주소창, 권한, 전체 브라우저 관리 |
| Renderer Process | HTML/CSS/JS 처리, 렌더링 |
| GPU Process | 그래픽 작업 |
| Network Process | 네트워크 요청 처리 |

탭 또는 사이트별로 렌더러 프로세스를 분리하면 한 페이지의 충돌이 다른 페이지에 미치는 영향을 줄이고, 사이트 간 데이터를 분리하는 보안 경계로도 활용할 수 있다. 대신 프로세스마다 메모리를 사용하므로 탭이 많을수록 자원 사용량도 늘어난다.

## 정리

1. 프로세스는 독립된 실행 환경이고, 스레드는 그 안의 실행 단위다.
2. 스레드는 메모리를 공유하므로 빠르지만 Race Condition을 조심해야 한다.
3. 동시성은 일을 번갈아 진행하는 방식이고, 병렬성은 실제로 동시에 실행하는 방식이다.
4. 공유 상태는 최소화하고, 필요한 경우 Mutex, `synchronized`, Atomic 같은 도구로 보호한다.
5. 서버와 브라우저 모두 제한된 스레드·프로세스 자원을 다루므로, 긴 블로킹 작업과 무제한 생성은 피해야 한다.
