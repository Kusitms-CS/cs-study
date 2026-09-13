# 2주차 - OS - Process / Thread / 동시성

## 1. Process

### 구조

운영체제로부터 자원(메모리, CPU, 파일 등)을 할당받아 실행되는 프로그램의 단위

```text
┌─────────────────────────┐
│         Process         │
│  ┌─────┐ ┌─────┐ ┌─────┐│
│  │Code │ │Data │ │Heap ││
│  └─────┘ └─────┘ └─────┘│
│  ┌───────────────────┐  │
│  │       Stack       │  │
│  └───────────────────┘  │
└─────────────────────────┘
```

### 특징

- 프로그램이 메모리에 적재되어 실행 중인 상태
- 각 프로세스는 독립된 주소 공간(Code, Data, Heap, Stack)을 가짐
- 프로세스끼리는 기본적으로 메모리를 공유하지 않음
- 하나 이상의 Thread를 포함

### 정리

- 생성 → 준비 → 실행 → 대기 → 종료 순서로 상태가 바뀌는 구조
- 준비 상태의 여러 프로세스 중 하나를 골라 실행 상태로 바꾸고, 당장 처리할 수 없는 작업이 생기면 대기 상태로 뺐다가 끝나면 다시 준비 상태로 돌림
- 프로세스끼리는 저장 공간을 따로 써서 서로 직접 들여다볼 수 없음 → 데이터를 주고받으려면 별도의 통신 방법이 필요함
- 하나가 죽어도 다른 프로세스에 영향이 적어 안정적이지만, 새로 만들고 전환하는 비용이 크고 통신을 위한 비용도 추가로 듦

---

## 2. Thread

### 구조

하나의 프로세스 내에서 실제로 실행되는 흐름의 단위

```text
┌───────────────────────────────────┐
│              Process              │
│  ┌─────┐ ┌─────┐ ┌───────────────┐│
│  │Code │ │Data │ │     Heap      ││
│  └─────┘ └─────┘ └───────────────┘│
│  ┌─────────┐ ┌─────────┐          │
│  │Thread 1  │ │Thread 2  │  ...   │
│  │ Stack    │ │ Stack    │        │
│  └─────────┘ └─────────┘          │
└───────────────────────────────────┘
```

### 특징

- 같은 프로세스 내의 Thread끼리는 Code, Data, Heap을 공유
- Stack은 Thread마다 독립적으로 가짐
- Thread 간 자원 공유가 가능한 만큼 데이터 동기화 문제가 발생할 수 있음

### Process vs Thread

| | Process | Thread |
| --- | --- | --- |
| 메모리 | 독립된 주소 공간 | Code/Data/Heap 공유, Stack만 독립 |
| 통신 비용 | IPC 필요, 상대적으로 비쌈 | 메모리 공유로 상대적으로 저렴 |
| 생성/전환 비용 | 상대적으로 비쌈 | 상대적으로 저렴 |
| 안정성 | 하나가 죽어도 다른 프로세스에 영향 적음 | 하나가 죽으면 프로세스 전체에 영향 |

### 정리

- 같은 프로세스 안의 자원을 함께 쓰기 때문에 새로 만들고 전환하는 비용이 적음 → 여러 Thread를 쓰는 이유
- 자원을 함께 쓰는 만큼 여러 Thread가 같은 데이터를 동시에 건드리면 값이 꼬이는 문제가 생길 수 있음
- 한 Thread에서 처리되지 못한 오류가 같은 프로세스 안의 다른 Thread까지 함께 멈추게 할 수 있음

---

## 3. Context Switching

### 구조

CPU가 다른 프로세스/스레드를 실행하기 위해 현재 실행 상태(Context)를 저장하고, 다음에 실행할 대상의 상태를 불러오는 과정

```text
Process A 실행 중
      ↓
A의 상태(PC, Register 등) 저장 (PCB)
      ↓
Process B의 상태 복원
      ↓
Process B 실행
```

### 특징

- 여러 프로세스/스레드가 하나의 CPU를 번갈아 사용할 수 있게 해주는 핵심 메커니즘
- Context는 PCB(Process Control Block)에 저장
- 전환 자체는 오버헤드이므로 잦은 Context Switching은 성능 저하로 이어질 수 있음

### Process 전환 vs Thread 전환

- Process 간 전환: 주소 공간까지 전환 필요 → 상대적으로 비용이 큼
- Thread 간 전환(같은 프로세스): 주소 공간은 공유 → 상대적으로 비용이 적음

### 정리

- 정해진 실행 시간이 끝나거나, 더 급한 작업이 들어오거나, 입출력을 기다려야 할 때 전환이 일어남
- 지금까지의 실행 상태를 저장해두고 다음 작업의 상태를 다시 불러오는 과정 자체에 시간이 걸림
- Process 전환은 저장 공간까지 통째로 바뀌는 거라 Thread 전환보다 비용이 더 큼
- Thread를 너무 많이 만들면 전환만 자주 일어나서, 실제 작업 시간보다 전환에 쓰는 시간이 늘어나 오히려 느려질 수 있음

---

## 4. 동시성 / 병렬성

### 개념 구분

동시성(Concurrency)과 병렬성(Parallelism)은 여러 작업을 다루는 방식이 다름

```text
동시성 (Concurrency) - Single Core
Task A ─┐
Task B ─┼─→ CPU가 빠르게 번갈아 실행 (Context Switching)
Task C ─┘

병렬성 (Parallelism) - Multi Core
Task A ─→ Core 1
Task B ─→ Core 2   (동시에 실제로 실행)
Task C ─→ Core 3
```

### 특징

- 동시성: 여러 작업을 번갈아 처리하여 동시에 처리되는 것처럼 보이게 함 (논리적 동시 실행)
- 병렬성: 여러 작업을 물리적으로 동시에 실행 (Multi Core 필요)
- 병렬성은 동시성의 부분집합으로 볼 수 있음 (병렬 실행되는 작업들은 동시성도 가짐)

### 정리

- Core가 하나여도 아주 짧은 단위로 작업을 번갈아 실행하기 때문에 여러 작업이 동시에 처리되는 것처럼 보이는 것 (실제로는 한 번에 하나씩 처리)
- 여러 Core가 실제로 동시에 작업을 나눠 처리하는 것과는 구분됨
- Multi Thread/Process는 실행 흐름 자체를 여러 개 만들어서, JS의 Event Loop는 하나의 실행 흐름 안에서 순서를 조정해서 동시성을 만든다는 차이가 있음

---

## 5. Race Condition

### 개념

여러 Thread/Process가 공유 자원에 동시에 접근할 때, 접근 순서에 따라 결과가 달라지는 상황

```text
공유 변수: count = 0

Thread A: count 읽기(0) → +1 계산 → count 쓰기(1)
Thread B: count 읽기(0) → +1 계산 → count 쓰기(1)

기대값: count = 2
실제값: count = 1  (Race Condition 발생)
```

### 발생 조건 (Critical Section)

- 여러 실행 흐름이 공유 자원에 동시에 접근
- 적어도 하나의 흐름이 자원을 변경
- 접근 순서에 대한 동기화 장치가 없음

### 정리

- 여러 실행 흐름이 동시에 건드리면 문제가 생길 수 있는 공유 자원 영역 (Critical Section)
- count++ 같은 연산도 읽기·계산·쓰기로 나뉘어 있어서, 그 사이에 다른 실행 흐름이 끼어들면 값이 꼬일 수 있음 (앞의 예시 참고)
- 순서를 맞춰주는 장치 없이 공유 자원을 건드리면, 실행할 때마다 결과가 달라지는 재현하기 어려운 문제로 이어짐

---

## 6. Mutex / Semaphore

### 구조

Critical Section에 대한 동시 접근을 제어하기 위한 동기화 도구

### Mutex (Mutual Exclusion)

- 한 번에 하나의 Thread만 Critical Section에 진입 가능 (Lock/Unlock)
- Lock을 획득한 Thread만 Unlock 가능

```text
Thread A: lock() → Critical Section → unlock()
Thread B: lock() 시도 → A가 unlock할 때까지 대기
```

### Semaphore

- 지정된 개수(N)만큼의 Thread가 동시에 접근 가능
- 카운터 기반으로 동작 (P 연산: 감소/대기, V 연산: 증가/신호)

```text
Semaphore(N=2)

Thread A: acquire() → count 2→1 → 진입
Thread B: acquire() → count 1→0 → 진입
Thread C: acquire() → count 0    → 대기
```

### Mutex vs Semaphore

| | Mutex | Semaphore |
| --- | --- | --- |
| 동시 접근 허용 개수 | 1개 | N개 (설정 가능) |
| 소유 개념 | Lock을 건 주체만 해제 가능 | 소유 개념 없음 |
| 용도 | 상호 배제 | 자원 개수 제한, 신호 전달 |

### 정리

- 한 번에 하나만 접근해야 하는 자원엔 Mutex, 정해진 개수까지는 동시 접근을 허용해도 되는 자원엔 Semaphore를 선택
- 서로 상대방이 가진 자원을 기다리며 멈춰버린 상태가 Deadlock
- 자원을 가져오는 순서를 항상 똑같이 맞추거나, 기다리는 시간에 제한을 두는 방식으로 예방 가능

---

# Frontend 심화

## 7. JS Single Thread

### 구조

JavaScript는 하나의 Call Stack만 가지는 Single Thread 언어

```text
Call Stack (1개)
┌────────────┐
│  function C │
├────────────┤
│  function B │
├────────────┤
│  function A │
└────────────┘
```

### 특징

- 한 번에 하나의 코드만 실행 가능 (Race Condition 걱정이 상대적으로 적음)
- 무거운 동기 작업이 있으면 Call Stack이 비워질 때까지 다른 작업이 막힘(Blocking)
- 비동기 처리(Web API, Event Loop)를 통해 Single Thread의 한계를 보완

### 정리

- 실행 흐름은 하나뿐이지만, 오래 걸리는 작업은 브라우저 쪽에 맡겨두고 끝난 뒤 순서대로 처리하는 방식으로 비동기 처리를 함
- 무거운 연산이 실행 흐름을 오래 붙잡고 있으면 그동안 다른 처리(화면 갱신, 클릭 반응)가 밀려서 화면이 멈춘 것처럼 보이는 것

---

## 8. Web Worker

### 구조

브라우저에서 JS를 메인 Thread와 별도의 Thread에서 실행할 수 있게 해주는 API

```text
Main Thread                Worker Thread
    │                            │
    │  postMessage(data)  →      │
    │                            │  무거운 연산 수행
    │      ← postMessage(result) │
```

```js
// main.js
const worker = new Worker("worker.js");

worker.postMessage(10000000);
worker.onmessage = (e) => {
  console.log("결과:", e.data);
};

// worker.js
self.onmessage = (e) => {
  let sum = 0;
  for (let i = 0; i < e.data; i++) sum += i;
  self.postMessage(sum);
};
```

### 특징

- Main Thread(UI Thread)와 별도의 Thread에서 실행되어 UI Blocking 방지
- DOM에 직접 접근 불가능, `postMessage`를 통한 메시지 전달로만 통신
- 무거운 연산(이미지 처리, 대량 데이터 계산 등)을 분리할 때 유용

### 정리

- 화면 요소는 여러 실행 흐름이 동시에 건드리면 꼬일 수 있어서, 별도 실행 흐름에서는 접근하지 못하게 막아둠
- 이미지·영상 처리나 대량 계산처럼 오래 걸리는 작업엔 적합하지만, 화면 요소를 직접 다뤄야 하거나 간단한 작업엔 오히려 비효율적

---

## 9. 브라우저 Multi-Process 구조

### 구조

현대 브라우저(Chrome 등)는 여러 Process로 역할을 분리하여 동작

```text
┌────────────────┐
│ Browser Process │  (주소창, 북마크, 네트워크 요청 등 총괄)
└────────────────┘
        │
   ┌────┴─────┬─────────────┬───────────────┐
┌────────┐ ┌────────┐  ┌──────────┐  ┌────────────┐
│Renderer│ │Renderer│  │GPU Process│  │Network Proc│
│(Tab A) │ │(Tab B) │  └──────────┘  └────────────┘
└────────┘ └────────┘
```

### 특징

- 탭(사이트)마다 별도의 Renderer Process를 사용하는 경우가 많음 (Site Isolation)
- 하나의 탭이 크래시되어도 다른 탭에 영향을 최소화
- Renderer Process 내부에는 Main Thread(JS 실행, DOM/CSSOM, 렌더링), Compositor Thread 등 여러 Thread가 존재

### 정리

- 탭 하나에 문제가 생겨도 다른 탭이나 브라우저 전체에 영향이 가지 않도록, 사이트별로 공간을 나눠서 관리
- 각 탭 안에서 화면 처리와 코드 실행을 담당하는 실행 흐름은 하나뿐 → 여러 탭 구조 위에, 각 탭 내부는 Single Thread로 동작
