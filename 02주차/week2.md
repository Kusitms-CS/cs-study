# Process, Thread, 그리고 동시성

이번 스터디에서는 Process와 Thread의 정의만 외우기보다는 **왜 Process와 Thread가 필요한지, 여러 작업을 동시에 처리하면서 어떤 문제가 발생하는지, 그리고 이러한 개념들이 실제 백엔드 서버에서는 어떻게 사용되는지**를 중심으로 공부했다.

---

## 1. Process

### Process는 왜 필요할까?

Process를 이해하려면 먼저 프로그램과 Process의 차이를 봐야 한다. 프로그램은 디스크에 저장되어 있는 실행 파일 자체를 의미하고, 이 프로그램이 실제로 실행되면서 CPU와 메모리 같은 자원을 할당받은 상태를 Process라고 한다.

예를 들어 크롬이나 IntelliJ가 설치되어 있다고 해서 항상 실행 중인 것은 아니다. 사용자가 프로그램을 실행하면 운영체제가 해당 프로그램이 실행될 수 있도록 메모리 공간과 필요한 자원을 할당하고 이때부터 하나의 Process로 관리된다.

Process마다 기본적으로 독립적인 메모리 공간을 가진다는 것도 중요한 특징이다. 하나의 Process는 Code, Data, Heap, Stack 등의 영역을 가지고 있으며, 다른 Process가 이 메모리 영역에 직접 접근할 수 없도록 분리되어 있다.

```
Process A
├── Code
├── Data
├── Heap
└── Stack

Process B
├── Code
├── Data
├── Heap
└── Stack
```

이렇게 Process별로 메모리를 분리하면 하나의 프로그램에서 문제가 발생했을 때 다른 프로그램에 미치는 영향을 줄일 수 있다. 반대로 Process끼리는 메모리가 분리되어 있기 때문에 서로 데이터를 주고받으려면 Pipe, Socket, Shared Memory 같은 IPC(Inter-Process Communication)가 필요하다.

### 실제 개발에서는?

백엔드 환경에서도 Process는 쉽게 볼 수 있다. 예를 들어 하나의 서버에서 Spring Boot 애플리케이션과 MySQL, Redis, Nginx를 실행한다면 각각 별도의 Process로 실행될 수 있다.

그런데 여러 작업을 동시에 처리하기 위해 매번 Process를 새롭게 생성한다면 어떨까? Process는 각각 독립적인 메모리 공간과 자원을 가지기 때문에 생성하고 관리하는 비용이 비교적 크다.

그래서 **하나의 Process 안에서 자원을 공유하면서 여러 작업을 수행할 수 있는 더 작은 실행 단위**가 필요했고, 여기서 Thread가 등장한다.

---

## 2. Thread

### Process가 있는데 Thread는 왜 필요할까?

Thread는 **Process 내부에서 실제 작업을 수행하는 실행 단위**이다. 하나의 Process는 여러 개의 Thread를 가질 수 있으며, 같은 Process에 속한 Thread들은 Code, Data, Heap 영역을 공유한다. 반면 각 Thread는 자신의 실행 흐름을 관리하기 위해 별도의 Stack을 가진다.

```
Process
│
├── Code ───────── 공유
├── Data ───────── 공유
├── Heap ───────── 공유
│
├── Thread 1
│   └── Stack
│
├── Thread 2
│   └── Stack
│
└── Thread 3
    └── Stack
```

Process를 여러 개 생성하는 것보다 같은 Process 내부에서 Thread를 여러 개 사용하는 것이 상대적으로 가볍고, 메모리를 공유하기 때문에 Thread끼리 데이터를 주고받기도 쉽다.

웹 서버를 생각하면 Thread가 필요한 이유를 쉽게 이해할 수 있다. 만약 서버가 하나의 실행 흐름만 가지고 있다면 사용자 A의 요청을 처리하는 동안 사용자 B의 요청은 기다려야 한다. 하지만 여러 Thread를 이용하면 각 요청을 서로 다른 Thread가 처리하면서 여러 사용자의 요청을 동시에 처리할 수 있다.

```
사용자 A 요청 → Thread 1
사용자 B 요청 → Thread 2
사용자 C 요청 → Thread 3
```

하지만 여기서 또 하나의 문제가 생긴다. CPU가 처리할 수 있는 작업에는 한계가 있는데 Thread를 여러 개 실행하면 어떻게 여러 작업이 동시에 실행되는 것처럼 보이는 것일까?

이 부분을 이해하기 위해서는 Context Switching을 알아야 한다.

---

## 3. Context Switching

CPU Core 하나는 기본적으로 한 순간에 하나의 작업을 실행한다. 하지만 우리가 컴퓨터를 사용할 때는 음악을 들으면서 브라우저를 사용하고, 동시에 IntelliJ에서 서버를 실행하는 등 여러 작업을 동시에 하고 있는 것처럼 느껴진다.

이는 운영체제가 여러 Process와 Thread에 CPU 사용 시간을 나누어 주면서 각각의 작업을 매우 빠르게 번갈아 실행하기 때문이다.

```
Thread A 실행
     ↓
Thread B 실행
     ↓
Thread C 실행
     ↓
Thread A 실행
```

이때 CPU가 Thread A를 실행하다가 Thread B로 전환하려면 Thread A가 어디까지 실행됐는지에 대한 상태를 저장하고, Thread B가 이전에 실행되던 상태를 다시 불러와야 한다. 이렇게 **현재 실행 중인 작업의 상태를 저장하고 다른 작업의 상태를 불러오는 과정**을 Context Switching이라고 한다.

그렇다면 Thread를 많이 만들수록 여러 작업을 더 많이 처리할 수 있으니 무조건 좋은 것일까?

그렇지는 않다. Context Switching 자체에도 비용이 발생하기 때문이다. Thread가 지나치게 많아지면 실제 작업을 처리하는 시간보다 Thread를 전환하고 관리하는 데 사용하는 비용이 커질 수 있고 각 Thread의 Stack을 위한 메모리도 필요하다.

즉,

```
Thread 증가
→ 동시에 처리할 수 있는 작업 증가

하지만 지나치게 많아지면
→ Context Switching 증가
→ 메모리 사용량 증가
→ 오히려 성능 저하 가능
```

라는 Trade-Off가 존재한다.

이 때문에 실제 백엔드 서버에서는 요청이 들어올 때마다 Thread를 계속 새로 만드는 것이 아니라 **일정한 수의 Thread를 미리 만들어두고 재사용하는 Thread Pool**을 많이 사용한다.

---

## 4. 동시성과 병렬성

Thread를 공부하면서 같이 헷갈렸던 개념이 동시성과 병렬성이었다. 둘 다 여러 작업을 처리한다는 점에서는 비슷하지만 실제 의미에는 차이가 있다.

### 동시성(Concurrency)

동시성은 여러 작업이 **같은 시간 동안 함께 진행되는 것**을 의미한다. CPU Core가 하나뿐이어도 여러 작업을 빠르게 번갈아 실행하면 여러 작업이 동시에 진행되고 있는 상태를 만들 수 있다.

```
Core 1

A → B → A → C → B → A
```

한 순간만 보면 하나의 작업을 실행하고 있지만 일정 시간 범위에서 보면 A, B, C가 모두 진행되고 있다.

### 병렬성(Parallelism)

병렬성은 여러 작업을 **실제로 같은 순간에 실행하는 것**을 의미한다. 이를 위해서는 여러 CPU Core처럼 실제로 동시에 작업을 수행할 수 있는 실행 자원이 필요하다.

```
Core 1 → 작업 A
Core 2 → 작업 B
Core 3 → 작업 C
```

따라서 동시성은 **여러 작업을 어떻게 함께 진행시킬 것인가**에 가까운 개념이고, 병렬성은 **여러 작업을 실제로 동시에 실행하는 것**에 가깝다고 이해했다.

백엔드 서버에서는 수많은 사용자의 요청을 동시에 다뤄야 하기 때문에 동시성이 중요하고, 멀티코어 환경에서는 여러 Thread가 서로 다른 Core에서 실행되면서 실제 병렬 처리까지 일어날 수 있다.

---

## 5. Race Condition

### Thread가 메모리를 공유하면 어떤 문제가 생길까?

Thread의 장점 중 하나는 같은 Process 내부의 Heap과 같은 메모리 영역을 공유한다는 것이다. 하지만 이러한 특징은 동시에 단점이 될 수도 있다.

예를 들어 여러 Thread가 하나의 `count` 값을 증가시킨다고 생각해보자.

```
count++;
```

코드만 보면 단순한 한 줄이지만 실제로는 값을 읽고, 증가시키고, 다시 저장하는 여러 단계의 작업으로 이루어질 수 있다.

```
1. count 값을 읽는다.
2. count + 1을 계산한다.
3. 계산된 값을 저장한다.
```

현재 `count = 0`인 상황에서 Thread A와 Thread B가 동시에 접근한다면 다음과 같은 상황이 발생할 수 있다.

```
Thread A → count 0 읽음
Thread B → count 0 읽음

Thread A → 1 저장
Thread B → 1 저장

결과: 1
기대값: 2
```

두 Thread 모두 같은 값을 읽은 뒤 각각 1을 저장했기 때문에 두 번 증가시켰는데도 결과는 1이 된다.

이처럼 **여러 Thread가 공유 자원에 동시에 접근하면서 실행 순서에 따라 결과가 달라질 수 있는 상황**을 Race Condition이라고 한다.

실제 백엔드에서도 Singleton Bean의 가변 상태나 서버 내부의 공유 캐시, 카운터처럼 여러 요청이 동시에 접근할 수 있는 데이터를 잘못 관리하면 이러한 문제가 발생할 수 있다.

그렇다면 여러 Thread가 같은 데이터를 동시에 변경하지 못하도록 막을 방법이 필요하다. 여기서 Mutex와 Semaphore 같은 동기화 기법이 등장한다.

---

## 6. Mutex와 Semaphore

### Race Condition을 어떻게 막을까?

가장 단순하게 생각하면 한 Thread가 공유 자원을 사용하고 있을 때 다른 Thread가 접근하지 못하도록 하면 된다.

예를 들어 화장실에 한 사람만 들어갈 수 있고 문을 잠그는 것과 비슷하다.

```
Thread A
   ↓
[ LOCK ]
   ↓
공유 자원 사용
   ↓
[ UNLOCK ]
```

이처럼 공유 자원에 한 번에 하나의 Thread만 접근할 수 있도록 제어하는 방법을 Mutex라고 한다. Thread A가 Mutex를 획득하고 있다면 Thread B는 A가 Mutex를 반환할 때까지 기다려야 한다.

반면 Semaphore는 동시에 접근할 수 있는 Thread의 개수를 정할 수 있다.

```
Semaphore = 3

Thread A → 접근 가능
Thread B → 접근 가능
Thread C → 접근 가능
Thread D → 대기
```

Mutex가 보통 하나의 공유 자원에 대해 상호 배제를 만드는 데 사용된다면, Semaphore는 **제한된 개수의 자원을 여러 Thread가 사용할 수 있도록 접근 개수를 제어하는 데 사용**할 수 있다.

예를 들어 동시에 사용할 수 있는 외부 API 연결이나 특정 자원의 수가 제한되어 있다면 Semaphore를 이용해 한 번에 접근할 수 있는 Thread의 수를 제한하는 식으로 활용할 수 있다.

다만 이렇게 동기화를 적용하면 Race Condition을 막을 수 있는 대신 다른 Thread가 기다리는 시간이 생긴다. Lock의 범위를 너무 크게 잡거나 필요 이상으로 동기화를 사용하면 동시 처리 성능이 떨어질 수 있고 여러 Lock을 잘못 사용하면 Deadlock 같은 새로운 문제도 발생할 수 있다.

결국 중요한 것은 모든 곳에 Lock을 거는 것이 아니라 **실제로 공유되는 가변 데이터가 무엇인지 확인하고 필요한 범위에 적절하게 동기화를 적용하는 것**이다.

---

# Backend 심화

## 7. Java Thread

Java에서는 `Thread` 클래스를 통해 직접 Thread를 생성할 수 있다.

```java
Thread thread = new Thread(() -> {
    System.out.println("작업 실행");
});

thread.start();
```

여기서 `run()`을 직접 호출하는 것과 `start()`를 호출하는 것은 다르다. `run()`을 직접 호출하면 현재 Thread에서 일반 메서드를 실행하는 것과 같지만 `start()`를 호출하면 새로운 Thread가 생성되고 해당 Thread에서 `run()`이 실행된다.

하지만 실제 백엔드 애플리케이션에서 작업이 발생할 때마다 다음처럼 Thread를 직접 생성하는 것은 좋은 방법이 아니다.

```
new Thread(task).start();
```

요청이 10개라면 큰 문제가 없을 수 있지만 요청이 수천 개, 수만 개 들어오면 그만큼 Thread가 만들어질 수 있다. Thread 생성 자체에도 비용이 들고 Thread가 많아질수록 Context Switching과 메모리 사용량도 증가한다.

그래서 실제 서버에서는 Thread를 매번 생성하기보다는 **미리 일정한 수의 Thread를 만들어두고 재사용하는 Thread Pool**을 사용한다.

---

## 8. Thread Pool

Thread Pool은 일정 개수의 Thread를 미리 생성해두고 작업이 들어오면 사용 가능한 Thread에게 작업을 할당하는 방식이다.

```
              ┌→ Thread 1
요청 → Queue ─┼→ Thread 2
              ├→ Thread 3
              └→ Thread 4
```

Thread가 작업을 끝냈다고 사라지는 것이 아니라 다시 Pool로 돌아가 다음 작업을 처리한다. 덕분에 요청마다 Thread를 새롭게 생성하고 제거하는 비용을 줄일 수 있고, 동시에 실행되는 Thread의 수도 제한할 수 있다.

Java에서는 `ExecutorService`를 이용해서 Thread Pool을 사용할 수 있다.

```
ExecutorService executor =
        Executors.newFixedThreadPool(10);

executor.submit(() -> {
    System.out.println("작업 처리");
});
```

Thread Pool의 크기를 정하는 것도 중요하다. 무조건 Thread를 많이 만든다고 성능이 좋아지는 것이 아니기 때문이다.

CPU 계산이 많은 작업이라면 CPU Core보다 지나치게 많은 Thread를 만드는 것이 큰 도움이 되지 않을 수 있다. 반대로 DB나 외부 API 응답을 기다리는 I/O 작업이 많다면 Thread가 대기하는 시간이 있기 때문에 CPU 작업보다 더 많은 Thread를 활용할 여지가 있다.

결국 Thread Pool 크기는 **CPU 작업인지 I/O 작업인지, 서버의 Core 수, 요청량, 작업 시간 등의 조건을 고려해서 결정해야 한다.**

---

## 9. synchronized

Java에서는 공유 자원에 여러 Thread가 동시에 접근하면서 발생하는 문제를 제어하기 위해 `synchronized`를 사용할 수 있다.

예를 들어 다음 코드는 여러 Thread가 동시에 실행한다면 Race Condition이 발생할 수 있다.

```java
private int count = 0;

public void increase() {
    count++;
}
```

이를 다음처럼 변경할 수 있다.

```java
public synchronized void increase() {
    count++;
}
```

한 Thread가 해당 synchronized 영역을 실행하고 있다면 같은 Lock을 필요로 하는 다른 Thread는 Lock을 얻을 때까지 기다리게 된다.

메서드 전체가 아니라 필요한 코드 영역에만 적용하는 것도 가능하다.

```java
public void increase() {
    synchronized (this) {
        count++;
    }
}
```

처음에는 `synchronized`를 사용하면 동시성 문제가 해결되니까 공유 데이터에는 전부 적용하면 되는 것 아닌가 생각할 수 있다. 하지만 Lock을 사용하는 동안 다른 Thread가 기다려야 하기 때문에 범위를 필요 이상으로 크게 잡으면 처리량이 떨어질 수 있다.

따라서 **공유 자원 중에서도 실제로 동시에 수정될 가능성이 있는 부분을 확인하고 Critical Section을 가능한 적절한 범위로 잡는 것**이 중요하다.

---

## 10. Atomic

단순한 숫자 증가 같은 작업에서도 `synchronized`를 반드시 사용해야 할까?

Java에서는 이런 상황을 위해 `AtomicInteger`, `AtomicLong` 같은 Atomic 클래스를 제공한다.

```java
AtomicInteger count = new AtomicInteger(0);

count.incrementAndGet();
```

Atomic 클래스는 단순한 값을 읽고 변경하는 연산을 원자적으로 처리할 수 있도록 지원한다. 내부적으로는 CAS(Compare-And-Set) 같은 방식을 활용하여 값을 비교하고 조건에 맞을 때 변경한다.

개념적으로 보면 다음과 비슷하다.

```
현재 값이 내가 알고 있던 값과 같은가?
        ↓
      YES
        ↓
새로운 값으로 변경
```

중간에 다른 Thread가 값을 변경했다면 변경에 실패하고 다시 시도할 수 있다.

따라서 단순 카운터처럼 비교적 간단한 공유 상태를 다룰 때 Atomic 클래스를 활용할 수 있다. 다만 여러 값을 함께 변경해야 하거나 여러 연산을 하나의 작업으로 묶어야 한다면 Atomic 하나만으로 해결되지 않을 수 있기 때문에 상황에 따라 `synchronized`, Lock 등의 다른 동기화 방법을 고려해야 한다.

---

## 11. WAS Thread

### 지금까지 배운 Thread가 Spring 서버에서는 어떻게 사용될까?

지금까지 공부한 내용을 실제 백엔드 서버와 연결하면 WAS의 Thread 처리 구조를 이해하는 데 도움이 된다.

Spring Boot에서 일반적으로 사용하는 Spring MVC + Tomcat 환경을 예로 들면 클라이언트의 요청이 들어올 때마다 무조건 새로운 Thread를 생성하는 것이 아니라 Tomcat이 관리하는 Thread Pool의 Thread를 이용해 요청을 처리한다.

```
Client
  ↓
HTTP Request
  ↓
Tomcat
  ↓
Thread Pool
  ↓
사용 가능한 Thread 할당
  ↓
Controller
  ↓
Service
  ↓
Repository
  ↓
Response
```

예를 들어 Thread Pool에 사용할 수 있는 Thread가 있다면 요청 하나에 Thread 하나가 할당되고 해당 Thread가 Controller부터 Service, Repository 등의 코드를 실행하면서 요청을 처리한다. 요청 처리가 끝나면 Thread를 제거하는 것이 아니라 다시 Pool로 반환하여 다음 요청에서 재사용한다.

여기서 Thread Pool의 Thread가 모두 사용 중인데 새로운 요청이 계속 들어온다면 바로 처리하지 못하고 대기하게 될 수 있다. 대기할 수 있는 요청의 수도 한계가 있기 때문에 서버가 감당할 수 있는 수준보다 많은 요청이 들어오면 결국 요청이 거절되거나 Timeout이 발생할 수도 있다.

이 구조를 보면 앞에서 공부했던 내용들이 실제 백엔드와 연결된다.

```
여러 사용자의 요청을 처리해야 함
        ↓
여러 Thread 필요
        ↓
요청마다 Thread를 생성하면 비용이 큼
        ↓
Thread Pool 사용
        ↓
여러 Thread가 같은 객체에 접근할 수 있음
        ↓
공유 가변 상태가 있다면 Race Condition 가능
        ↓
synchronized / Atomic / Lock 등의 동시성 제어 고려
```

특히 Spring의 Bean은 기본적으로 Singleton Scope로 관리되는 경우가 많기 때문에 여러 요청 Thread가 동일한 Bean 객체를 사용할 수 있다. 그래서 Service나 Controller에 요청별로 변경되는 값을 필드에 저장하는 식의 코드를 작성하면 동시성 문제가 발생할 수 있다.

```java
@Service
public class UserService {

    private Long currentUserId;

    public void process(Long userId) {
        this.currentUserId = userId;
    }
}
```

따라서 Spring Bean은 가능한 **Stateless하게 설계하고 요청에 필요한 데이터는 지역 변수나 파라미터 등을 통해 관리하는 것**이 중요하다.

이번 스터디를 통해 Thread가 단순히 Java 문법에서 사용하는 기능이 아니라, **여러 사용자의 요청을 동시에 처리하는 백엔드 서버의 기본적인 동작 방식과 직접 연결되는 개념**이라는 것을 알 수 있었다.
