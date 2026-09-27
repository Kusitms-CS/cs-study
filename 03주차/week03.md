# Memory

## 1. Memory 구조

프로그램이 실행되면 운영체제로부터 메모리 공간을 할당받음

일반적으로 프로세스의 메모리 영역은 다음과 같이 구분됨

- Code 영역
- Data 영역
- Heap 영역
- Stack 영역

```text
높은 주소
┌─────────────────┐
│      Stack      │
│        ↓        │
│                 │
│        ↑        │
│      Heap       │
├─────────────────┤
│      Data       │
├─────────────────┤
│      Code       │
└─────────────────┘
낮은 주소
```

### 1.1 Code 영역

실행할 프로그램의 명령어가 저장되는 영역임

CPU가 Code 영역에 저장된 명령어를 하나씩 가져와 실행함

일반적으로 프로그램 실행 중 내용이 변경되지 않는 읽기 전용 영역으로 관리됨

### 1.2 Data 영역

전역 변수와 정적 변수가 저장되는 영역임

프로그램 시작 시 할당되고 프로그램 종료 시 해제됨

초기화된 데이터와 초기화되지 않은 데이터를 별도의 영역으로 구분하기도 함

### 1.3 Heap 영역

프로그램 실행 중 동적으로 할당되는 메모리 영역임

필요한 시점에 메모리를 할당하고 더 이상 필요하지 않을 경우 해제함

C/C++에서는 개발자가 직접 메모리를 관리해야 하며 Java에서는 GC가 사용하지 않는 객체를 정리함

일반적으로 낮은 주소에서 높은 주소 방향으로 확장됨

### 1.4 Stack 영역

함수 호출과 관련된 데이터가 저장되는 영역임

지역 변수, 매개변수, 반환 주소 등의 정보를 저장함

함수가 호출될 때 Stack Frame이 생성되고 함수가 종료되면 제거됨

LIFO 구조로 관리됨

일반적으로 높은 주소에서 낮은 주소 방향으로 확장됨

---

## 2. Stack / Heap

### 2.1 Stack

함수 호출 단위로 필요한 데이터를 저장하는 메모리 영역임

함수가 호출될 때마다 Stack Frame이 생성됨

Stack Frame에는 지역 변수, 매개변수, 반환 주소 등의 정보가 포함됨

함수 종료 시 해당 Stack Frame이 자동으로 제거됨

메모리 할당과 해제가 빠르지만 사용할 수 있는 크기가 제한적임

재귀 호출이 지나치게 깊어질 경우 Stack Overflow가 발생할 수 있음

### 2.2 Heap

프로그램 실행 중 동적으로 생성되는 데이터를 저장하는 영역임

Stack보다 상대적으로 큰 메모리 공간을 사용할 수 있음

객체나 동적으로 생성되는 데이터가 주로 저장됨

Java에서는 `new`를 통해 생성한 객체가 일반적으로 Heap에 저장됨

Heap에 생성된 객체를 더 이상 참조하지 않을 경우 GC의 정리 대상이 될 수 있음

### 2.3 Stack과 Heap 비교

| 구분 | Stack | Heap |
| --- | --- | --- |
| 용도 | 함수 호출 및 지역 데이터 | 동적으로 생성되는 데이터 |
| 관리 | 자동 | 개발자 또는 GC |
| 속도 | 상대적으로 빠름 | 상대적으로 느림 |
| 크기 | 비교적 작음 | 비교적 큼 |
| 구조 | LIFO | 동적 메모리 공간 |
| 대표 문제 | Stack Overflow | 메모리 부족, Memory Leak |

---

# Virtual Memory

## 3. Virtual Memory

### 3.1 가상 메모리란

프로세스가 실제 물리 메모리보다 큰 메모리 공간을 사용하는 것처럼 보이게 하는 메모리 관리 기법임

각 프로세스에게 독립적인 가상 주소 공간을 제공함

프로세스는 실제 RAM의 물리 주소를 직접 사용하는 것이 아니라 가상 주소를 사용함

운영체제와 하드웨어가 가상 주소를 실제 물리 주소로 변환함

필요한 데이터만 물리 메모리에 올려 사용할 수 있어 RAM을 효율적으로 활용할 수 있음

### 3.2 가상 주소와 물리 주소

프로세스가 사용하는 주소를 가상 주소라고 함

실제 RAM에서 사용하는 주소를 물리 주소라고 함

```text
Process
   │
   │ Virtual Address
   ▼
┌──────────────┐
│     MMU      │
└──────────────┘
   │
   │ Physical Address
   ▼
┌──────────────┐
│     RAM      │
└──────────────┘
```

프로세스마다 독립적인 가상 주소 공간을 가지므로 다른 프로세스의 메모리에 직접 접근하기 어려움

이를 통해 프로세스 간 메모리 보호가 가능함

### 3.3 MMU

MMU는 Memory Management Unit의 약자임

CPU가 생성한 가상 주소를 물리 주소로 변환하는 하드웨어 장치임

주소 변환 과정에서 Page Table 등의 정보를 활용함

---

# Paging

## 4. Paging

### 4.1 Paging이란

가상 메모리를 일정한 크기의 블록으로 나누어 관리하는 방식임

가상 메모리의 블록을 Page라고 함

물리 메모리의 블록을 Frame이라고 함

Page와 Frame의 크기는 동일함

```text
Virtual Memory            Physical Memory

┌─────────┐               ┌─────────┐
│ Page 0  │ ────────────→ │ Frame 3 │
├─────────┤               ├─────────┤
│ Page 1  │ ──────┐       │ Frame 1 │
├─────────┤       │       ├─────────┤
│ Page 2  │ ──┐   └─────→ │ Frame 4 │
└─────────┘   │           ├─────────┤
              └─────────→ │ Frame 0 │
                          └─────────┘
```

Page를 연속된 Frame에 저장할 필요가 없기 때문에 물리 메모리를 효율적으로 사용할 수 있음

### 4.2 Page와 Frame

Page는 가상 메모리를 일정한 크기로 나눈 단위임

Frame은 물리 메모리를 Page와 동일한 크기로 나눈 단위임

하나의 Page를 하나의 Frame에 매핑하여 사용함

### 4.3 Page Table

가상 Page가 어떤 물리 Frame에 위치하는지 저장하는 자료구조임

프로세스마다 자신의 Page Table을 가짐

가상 주소는 일반적으로 다음과 같이 구성됨

```text
Virtual Address

┌────────────────┬───────────────┐
│  Page Number   │    Offset     │
└────────────────┴───────────────┘
```

Page Number를 이용하여 Page Table에서 Frame Number를 찾음

Offset은 해당 Page 내부에서 실제 데이터가 위치한 위치를 나타냄

```text
Page Number
     │
     ▼
┌─────────────┐
│ Page Table  │
└─────────────┘
     │
     ▼
Frame Number + Offset
     │
     ▼
Physical Address
```

### 4.4 TLB

TLB는 Translation Lookaside Buffer의 약자임

최근 사용한 Page Table의 주소 변환 정보를 저장하는 고속 캐시임

매번 메모리에 있는 Page Table을 조회하면 주소 변환 비용이 커지기 때문에 사용함

TLB에 필요한 정보가 존재하면 TLB Hit임

TLB에 정보가 없으면 TLB Miss가 발생하고 Page Table을 조회함

```text
Virtual Address
      │
      ▼
     TLB
   ┌──┴──┐
 Hit    Miss
  │       │
  │       ▼
  │   Page Table
  │       │
  └───┬───┘
      ▼
Physical Address
```

---

# Page Fault

## 5. Page Fault

### 5.1 Page Fault란

프로세스가 접근하려는 Page가 현재 물리 메모리에 존재하지 않을 때 발생하는 예외임

가상 메모리를 사용하면 모든 Page를 처음부터 RAM에 올려놓을 필요가 없음

필요한 Page만 RAM에 올려 사용하는 Demand Paging 방식을 사용할 수 있음

### 5.2 Page Fault 처리 과정

Page Fault가 발생하면 일반적으로 다음 과정으로 처리함

1. CPU가 특정 Page에 접근함
2. Page Table을 확인함
3. 해당 Page가 RAM에 존재하지 않는 것을 확인함
4. Page Fault가 발생함
5. 운영체제가 디스크에서 필요한 Page를 찾음
6. 빈 Frame을 찾거나 기존 Page를 교체함
7. 필요한 Page를 RAM에 적재함
8. Page Table을 수정함
9. 중단되었던 명령을 다시 실행함

```text
CPU
 │
 ▼
Page 접근
 │
 ▼
RAM에 존재?
 │
 ├── YES → 정상 실행
 │
 └── NO
      │
      ▼
  Page Fault
      │
      ▼
Disk에서 Page 탐색
      │
      ▼
RAM에 적재
      │
      ▼
Page Table 갱신
      │
      ▼
명령 재실행
```

디스크 접근은 RAM 접근보다 매우 느리기 때문에 Page Fault가 자주 발생하면 성능이 크게 저하될 수 있음

### 5.3 페이지 교체

RAM에 빈 Frame이 없는 상태에서 새로운 Page를 적재해야 하는 경우 기존 Page 중 하나를 제거해야 함

이를 Page Replacement라고 함

대표적인 페이지 교체 알고리즘은 다음과 같음

- FIFO
- LRU
- Optimal

FIFO는 가장 먼저 들어온 Page를 제거하는 방식임

LRU는 가장 오랫동안 사용되지 않은 Page를 제거하는 방식임

Optimal은 앞으로 가장 오랫동안 사용되지 않을 Page를 제거하는 방식임

Optimal은 미래의 메모리 접근을 알아야 하기 때문에 실제 구현보다는 다른 알고리즘의 성능 비교 기준으로 사용됨

Page Fault가 지나치게 자주 발생하여 CPU보다 Page 교체 작업에 대부분의 시간을 사용하는 현상을 Thrashing이라고 함

---

# Backend 심화

## 6. JVM Runtime Data Area

Java 프로그램은 JVM 위에서 실행됨

JVM은 프로그램 실행을 위해 여러 메모리 영역을 사용함

```text
JVM Runtime Data Area

┌───────────────────────────────┐
│            Heap               │
│         모든 Thread 공유       │
├───────────────────────────────┤
│         Method Area           │
│         모든 Thread 공유       │
├───────────────────────────────┤
│ JVM Stack │ JVM Stack │ ...   │
│ Thread 1  │ Thread 2  │       │
├───────────────────────────────┤
│ PC Register / Native Stack    │
│         Thread별 생성          │
└───────────────────────────────┘
```

### 6.1 Method Area

클래스와 관련된 메타데이터를 저장하는 영역임

클래스 정보, 메서드 정보, 런타임 상수 풀 등의 정보를 관리함

모든 Thread가 공유하는 영역임

HotSpot JVM에서는 Java 8부터 기존 Permanent Generation이 제거되고 클래스 메타데이터가 Native Memory의 Metaspace에서 관리됨

### 6.2 Heap

Java 객체와 배열이 주로 저장되는 영역임

모든 Thread가 공유하는 메모리 영역임

GC의 주요 관리 대상임

Heap 공간이 부족하고 더 이상 확보할 수 없는 경우 `OutOfMemoryError`가 발생할 수 있음

### 6.3 JVM Stack

각 Thread마다 독립적으로 생성되는 영역임

메서드가 호출될 때마다 Stack Frame이 생성됨

Stack Frame에는 지역 변수, 메서드 실행에 필요한 중간 연산 정보 등이 저장됨

메서드가 종료되면 해당 Stack Frame이 제거됨

호출이 지나치게 깊어져 Stack 공간을 초과하면 `StackOverflowError`가 발생할 수 있음

### 6.4 PC Register

각 Thread가 현재 실행하고 있는 JVM 명령의 위치를 관리하기 위한 영역임

Thread마다 독립적으로 생성됨

Thread가 여러 개 존재하면 각각 자신의 실행 위치를 관리해야 하기 때문에 Thread별로 PC Register가 필요함

### 6.5 Native Method Stack

Java가 아닌 C/C++ 등의 Native Method를 실행하기 위한 Stack 영역임

JNI 등을 통해 Native Method를 호출할 때 사용됨

Thread마다 독립적으로 생성됨

---

# JVM Heap / Stack

## 7. JVM Heap / Stack

### 7.1 객체 생성 과정

다음과 같은 코드가 있다고 가정함

```java
Member member = new Member();
```

`new Member()`를 통해 생성된 객체는 일반적으로 Heap에 저장됨

지역 변수 `member`는 현재 메서드의 Stack Frame에 위치하며 Heap 객체를 가리키는 참조값을 가짐

```text
JVM Stack                         Heap

┌──────────────────┐           ┌─────────────────┐
│ member           │ ────────→ │ Member Object   │
│ 참조값            │           │                 │
└──────────────────┘           └─────────────────┘
```

Stack에는 객체 자체가 아니라 객체를 가리키는 참조값이 저장되는 형태로 이해할 수 있음

### 7.2 Heap과 Stack의 관계

Stack은 Thread마다 독립적으로 존재함

Heap은 여러 Thread가 공유함

```text
Thread A Stack ───┐
                  │
                  ▼
              ┌────────┐
              │  Heap  │
              └────────┘
                  ▲
                  │
Thread B Stack ───┘
```

여러 Thread가 동일한 Heap 객체에 접근할 수 있기 때문에 동시성 문제가 발생할 수 있음

Race Condition을 방지하기 위해 `synchronized`, Lock, Atomic 등의 동기화 방법을 사용할 수 있음

### 7.3 Thread와 Stack

Java에서 Thread가 생성되면 해당 Thread만 사용하는 JVM Stack도 생성됨

각 Thread는 자신의 지역 변수와 메서드 호출 정보를 독립적으로 관리함

따라서 일반적인 지역 변수는 다른 Thread와 직접 공유되지 않음

반면 Heap의 객체는 여러 Thread에서 참조할 수 있으므로 공유 데이터에 대한 동기화가 중요함

---

# Garbage Collection

## 8. Garbage Collection

### 8.1 GC란

Garbage Collection은 Heap에서 더 이상 사용되지 않는 객체를 자동으로 제거하는 메모리 관리 방식임

Java 개발자가 직접 객체의 메모리를 해제하지 않아도 JVM의 GC가 이를 관리함

```text
Stack
 │
 │ Reference
 ▼
Object A ─────→ Object B

Object C
   ↑
어디에서도 참조되지 않음
   ↓
GC 대상
```

### 8.2 GC 대상 판단

Java GC는 단순히 참조 횟수만으로 객체의 생존 여부를 판단하지 않음

GC Root에서 객체에 도달할 수 있는지를 기준으로 객체의 생존 여부를 판단함

이를 Reachability라고 함

GC Root에서 도달할 수 있는 객체는 Reachable 객체임

GC Root에서 도달할 수 없는 객체는 Unreachable 객체이며 GC 대상이 될 수 있음

대표적인 GC Root에는 실행 중인 Thread의 Stack에서 참조하는 객체, Static 영역에서 참조하는 객체 등이 있음

### 8.3 Young / Old Generation

Heap은 GC 효율을 위해 객체의 생존 기간 등을 기준으로 여러 영역으로 나누어 관리할 수 있음

전통적인 세대별 GC에서는 크게 Young Generation과 Old Generation으로 구분함

```text
Heap

┌───────────────────────────────┐
│       Young Generation        │
│                               │
│   Eden   │ Survivor │ Survivor│
├───────────────────────────────┤
│        Old Generation         │
└───────────────────────────────┘
```

Young Generation은 새롭게 생성된 객체가 주로 위치하는 영역임

대부분의 객체는 생성된 후 오래 살아남지 않는다는 Weak Generational Hypothesis를 기반으로 함

Eden에서 생성된 객체 중 GC 이후에도 살아남은 객체가 Survivor 영역 등을 거치며 관리됨

여러 번 GC에서 살아남은 객체는 Old Generation으로 이동할 수 있으며 이를 Promotion이라고 함

### 8.4 Minor GC / Major GC

Young Generation을 중심으로 수행되는 GC를 일반적으로 Minor GC라고 부름

Young 영역은 객체 생성과 소멸이 빈번하기 때문에 비교적 자주 GC가 발생함

Old Generation을 대상으로 하는 GC를 흔히 Major GC라고 표현함

전체 Heap을 대상으로 수행되는 GC를 Full GC라고 표현하는 경우가 많음

다만 Minor, Major, Full GC의 정확한 의미와 동작 방식은 사용하는 GC 알고리즘과 JVM 구현에 따라 차이가 있을 수 있음

### 8.5 Stop-The-World

GC 수행 과정에서 애플리케이션 Thread의 실행이 일시적으로 중단되는 현상임

GC가 객체의 참조 관계를 안전하게 확인하거나 메모리를 정리하기 위해 발생할 수 있음

이를 Stop-The-World 또는 STW라고 함

```text
Application Thread

실행 ────────┐
             │
         Stop-The-World
             │
             ▼
           GC 수행
             │
             ▼
실행 ◀───────┘
```

STW 시간이 길어지면 서버의 응답 지연으로 이어질 수 있음

백엔드 서버에서는 GC 횟수뿐만 아니라 GC Pause Time도 중요한 성능 지표가 될 수 있음

### 8.6 GC 종류

HotSpot JVM에서 대표적으로 사용되는 GC는 다음과 같음

- Serial GC
- Parallel GC
- G1 GC
- ZGC

Serial GC는 하나의 GC Thread를 중심으로 처리하는 단순한 방식임

Parallel GC는 여러 Thread를 사용하여 GC 처리량을 높이는 방식임

G1 GC는 Heap을 여러 Region으로 나누어 관리하며 긴 GC Pause를 줄이기 위한 방식임

ZGC는 매우 짧은 Pause Time을 목표로 대부분의 GC 작업을 애플리케이션과 동시에 수행하도록 설계된 GC임

GC는 각각 처리량, 지연 시간, CPU 사용량 등의 특성이 다르므로 애플리케이션 환경에 맞는 선택과 튜닝이 필요함
