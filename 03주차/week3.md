# 3주차 - OS - Memory / Virtual Memory

## 1. Memory 구조

> 프로그램이 실행되면 OS는 프로세스에 필요한 메모리 공간을 할당한다.

프로세스의 메모리는 대표적으로 코드, 데이터, 힙, 스택 영역으로 구분할 수 있다.

```text
┌─────────────────┐
│      Stack      │
│        ↓        │
│        ↑        │
│      Heap       │
├─────────────────┤
│      Data       │
├─────────────────┤
│      Code       │
└─────────────────┘
```

| 영역 | 역할 |
|---|---|
| Code | 실행할 프로그램의 명령어 |
| Data | 전역 변수, 정적 변수 등 |
| Heap | 동적으로 할당되는 데이터 |
| Stack | 함수 호출 정보, 지역 변수 등 |

### Code 영역

프로그램이 실행할 명령어가 저장되는 영역이다.

작성한 프로그램의 코드가 컴파일되어 만들어진 실행 명령들이 위치하며, 일반적으로 실행 중 임의로 변경되지 않도록 읽기 전용으로 관리된다.

### Data 영역

프로그램 실행 동안 유지되는 전역 변수, 정적 변수 등의 데이터가 저장되는 영역이다.

실제로는 초기화 여부 등에 따라 더 세부적인 영역으로 나눌 수 있지만, 여기서는 Data 영역 정도로 이해하면 충분하다.

### Heap과 Stack

프로그램 실행 중 필요한 데이터는 Heap과 Stack을 통해 관리할 수 있다.

```text
Stack
→ 함수 호출과 관련된 데이터
→ 지역 변수, 함수 호출 정보 등

Heap
→ 실행 중 동적으로 필요한 데이터
→ 필요한 시점에 메모리 할당
```

두 영역은 메모리를 사용하는 목적과 관리 방식에 차이가 있다.

## 2. Stack/Heap

### Stack

> 함수 호출과 관련된 정보를 관리하는 메모리 영역이다.

함수가 호출되면 해당 함수 실행에 필요한 **Stack Frame**이 생성된다.

예를 들어 아래와 같은 순서로 함수가 호출되었다고 가정하자.

```text
main()
  ↓
foo()
  ↓
bar()
```

Stack에는 다음과 같이 Stack Frame이 쌓인다.

```text
┌─────────────┐
│ bar() Frame │
├─────────────┤
│ foo() Frame │
├─────────────┤
│main() Frame │
└─────────────┘
```

`bar()`의 실행이 끝나면 `bar()`의 Stack Frame이 제거되고 이전 함수인 `foo()`로 돌아간다.

Stack Frame에는 함수의 지역 변수, 매개변수, 복귀에 필요한 정보 등이 포함될 수 있다.

### Stack의 특징

Stack은 함수 호출과 반환에 따라 메모리가 관리된다.

함수가 종료되면 해당 Stack Frame을 제거하면 되기 때문에 메모리 관리가 단순하고 빠르다.

하지만 Stack의 크기는 제한되어 있기 때문에 지나치게 깊은 함수 호출이 발생하면 **Stack Overflow**가 발생할 수 있다.

대표적인 예가 무한 재귀 호출이다.

```java
void recursive() {
    recursive();
}
```

```text
recursive()
   ↓
recursive()
   ↓
recursive()
   ↓
  ...
   ↓
Stack Overflow
```
### Heap

> 실행 중 필요한 데이터를 동적으로 할당하기 위한 메모리 영역이다.

Stack처럼 함수 호출과 종료에 따라 일정한 순서로 메모리가 관리되는 것이 아니라, 필요한 시점에 메모리를 할당한다.

```text
┌─────────────────┐
│      Heap       │
│                 │
│    Object A     │
│    Object B     │
│    Object C     │
└─────────────────┘
```

Heap에 할당된 메모리도 더 이상 필요하지 않으면 해제해야 한다.

C/C++에서는 개발자가 직접 메모리를 해제하는 경우가 있지만, Java에서는 **Garbage Collector(GC)**가 더 이상 사용되지 않는 객체의 메모리를 관리한다.

### Stack vs Heap

| 구분 | Stack | Heap |
|---|---|---|
| 주요 용도 | 함수 호출, 지역 변수 | 동적으로 생성되는 데이터 |
| 관리 방식 | 함수 호출/종료에 따라 관리 | 동적으로 할당/해제 |
| 속도 | 상대적으로 빠름 | 상대적으로 느림 |
| 크기 | 상대적으로 제한적 | 상대적으로 큼 |
| 문제 예시 | Stack Overflow | 메모리 부족, 메모리 누수 |

단순히 `Stack은 빠르고 Heap은 느리다`라고 외우기보다 **메모리를 관리하는 방식이 다르다**고 이해하는 것이 중요하다.

Stack은 함수 호출 구조에 따라 Stack Frame을 순서대로 생성하고 제거할 수 있다.

반면 Heap은 크기와 생명주기가 서로 다른 데이터를 동적으로 관리해야 하기 때문에 Stack보다 관리가 복잡하다.

### 실제 메모리는 한정되어 있다

Stack과 Heap을 포함해 프로세스가 사용하는 메모리는 무한하지 않다.

여러 프로세스가 동시에 실행되더라도 실제 컴퓨터의 Physical Memory(RAM)는 한정되어 있다.

```text
Process A ─┐
Process B ─┼─→ Physical Memory(RAM)
Process C ─┘        한정됨
```

또한 각 프로세스는 다른 프로세스의 메모리에 직접 접근하지 못하도록 서로 독립된 주소 공간을 사용하는 것이 필요하다.

그렇다면 다음과 같은 문제가 생긴다.

```text
한정된 Physical Memory
        +
여러 Process의 메모리 사용
        +
Process 간 메모리 보호
        ↓
메모리를 어떻게 효율적으로 관리할까?
```

이를 해결하기 위해 사용하는 핵심 개념이 **Virtual Memory(가상 메모리)**이다.

## 3. Virtual Memory

> 프로세스가 실제 Physical Memory(RAM)를 직접 사용하는 대신, 자신만의 가상 주소 공간을 사용하는 메모리 관리 기법이다.

프로그램이 사용하는 메모리 주소와 실제 RAM의 주소를 분리하고, OS가 둘 사이의 관계를 관리한다.

```text
Process
   ↓
Virtual Address
   ↓
주소 변환
   ↓
Physical Address
   ↓
Physical Memory(RAM)
```

프로그램 입장에서는 자신만의 연속된 메모리 공간을 사용하는 것처럼 보이지만, 실제 데이터가 RAM에서 연속된 위치에 존재할 필요는 없다.

### 왜 Virtual Memory를 사용할까?

여러 프로세스가 Physical Memory를 직접 사용한다고 생각해보자.

```text
Physical Memory

┌─────────────────┐
│   Process A     │
├─────────────────┤
│   Process B     │
├─────────────────┤
│   Process C     │
└─────────────────┘
```

프로세스마다 필요한 메모리 크기가 다르고, 실행과 종료가 반복되기 때문에 실제 메모리를 직접 관리하는 것은 복잡하다.

Virtual Memory를 사용하면 각 프로세스는 자신만의 독립적인 가상 주소 공간을 가질 수 있다.

```text
Process A                Process B
Virtual Memory           Virtual Memory

┌──────────────┐         ┌──────────────┐
│ 0x0000       │         │ 0x0000       │
│     ...      │         │     ...      │
│ 0xFFFF       │         │ 0xFFFF       │
└──────┬───────┘         └──────┬───────┘
       │                        │
       └────────┐      ┌────────┘
                ↓      ↓
           Physical Memory
```

같은 가상 주소를 사용하더라도 실제로는 서로 다른 Physical Memory 영역에 매핑될 수 있다.

이를 통해 프로세스마다 독립적인 주소 공간을 제공하고, 다른 프로세스의 메모리에 함부로 접근하지 못하도록 보호할 수 있다.

### 실제 RAM보다 큰 메모리를 사용할 수 있는 이유

Virtual Memory의 모든 내용을 항상 RAM에 올려둘 필요는 없다.

현재 필요한 부분은 RAM에 두고, 당장 필요하지 않은 부분은 디스크의 공간을 활용할 수 있다.

```text
Virtual Memory
┌───────────────┐
│ 현재 사용 중  │ ───→ RAM
├───────────────┤
│ 현재 미사용   │ ───→ Disk
├───────────────┤
│ 현재 사용 중  │ ───→ RAM
└───────────────┘
```

따라서 프로세스는 실제 RAM 크기에 직접 제한된 하나의 연속된 공간만 사용하는 것이 아니라 더 큰 가상 주소 공간을 사용할 수 있다.

이러한 Virtual Memory를 구현하는 대표적인 방법이 **Paging**이다.

## 4. Paging

> 가상 메모리와 물리 메모리를 일정한 크기의 블록으로 나누어 관리하는 방식이다.

가상 메모리를 나눈 단위를 **Page**, Physical Memory를 같은 크기로 나눈 단위를 **Frame**이라고 한다.

```text
Virtual Memory             Physical Memory

┌──────────┐               ┌──────────┐
│ Page 0   │ ────────────→ │ Frame 2  │
├──────────┤               ├──────────┤
│ Page 1   │ ──────┐       │ Frame 0  │
├──────────┤       │       ├──────────┤
│ Page 2   │ ──┐   └─────→ │ Frame 3  │
└──────────┘   │           ├──────────┤
               └─────────→ │ Frame 1  │
                           └──────────┘
```

Page와 Frame의 크기는 동일하기 때문에 각 Page를 사용 가능한 Frame에 배치할 수 있다.

따라서 프로세스의 메모리가 Physical Memory에서 연속적으로 배치될 필요가 없다.

### Page Table

그렇다면 CPU가 사용하는 가상 주소가 실제 RAM의 어디에 있는지 알아야 한다.

이를 위해 **Page Table**을 사용한다.

Page Table은 각 Page가 어떤 Frame에 매핑되어 있는지 관리한다.

```text
Virtual Page       Page Table       Physical Frame

Page 0 ─────────→ Frame 2 ───────→ Frame 2
Page 1 ─────────→ Frame 3 ───────→ Frame 3
Page 2 ─────────→ Frame 1 ───────→ Frame 1
```

CPU가 가상 주소에 접근하면 해당 주소가 속한 Page를 확인하고 Page Table을 이용해 실제 Frame을 찾는다.

```text
Virtual Address
      ↓
Page 번호 확인
      ↓
Page Table 조회
      ↓
Frame 번호 확인
      ↓
Physical Address
```

### Page Table을 매번 조회하면 느리지 않을까?

가상 주소를 실제 주소로 변환할 때마다 Page Table을 메모리에서 조회하면 추가적인 메모리 접근이 필요하다.

이를 줄이기 위해 CPU는 **TLB(Translation Lookaside Buffer)**라는 빠른 캐시를 사용한다.

```text
Virtual Address
      ↓
     TLB
   ↙     ↘
Hit       Miss
 ↓          ↓
Frame     Page Table 조회
```

최근에 사용한 주소 변환 정보를 TLB에 저장해두고, 필요한 정보가 존재하면 Page Table을 다시 조회하지 않고 빠르게 주소를 변환할 수 있다.

여기서는 **TLB는 Page Table 조회 비용을 줄이기 위한 주소 변환 캐시** 정도로 이해하면 충분하다.

### Paging과 단편화

Paging에서는 Page와 Frame의 크기가 동일하기 때문에 메모리에 Page를 연속적으로 배치할 필요가 없다.

따라서 연속된 큰 공간을 찾지 못해서 메모리를 할당하지 못하는 **외부 단편화 문제를 줄일 수 있다.**

하지만 Page가 고정된 크기를 사용하기 때문에 마지막 Page의 일부 공간이 사용되지 않을 수 있다.

```text
Page 크기 = 4KB

필요한 데이터 = 10KB

Page 1 → 4KB 사용
Page 2 → 4KB 사용
Page 3 → 2KB 사용 + 2KB 낭비
```

이처럼 할당된 Page 내부에 사용되지 않는 공간이 발생하는 것을 **내부 단편화**라고 한다.

## 5. Page Fault

> 프로세스가 접근하려는 Page가 현재 Physical Memory에 존재하지 않을 때 발생한다.

Virtual Memory의 모든 Page가 항상 RAM에 존재하는 것은 아니다.

따라서 CPU가 특정 Page에 접근했는데 해당 Page가 RAM에 없다면 Page Fault가 발생할 수 있다.

```text
CPU
 ↓
Page 3 접근
 ↓
Page Table 확인
 ↓
Page 3이 RAM에 없음
 ↓
Page Fault
```

### Page Fault가 발생하면 어떻게 될까?

Page Fault가 발생하면 OS가 필요한 Page를 Physical Memory로 가져와야 한다.

```text
1. CPU가 Page에 접근
        ↓
2. Page가 RAM에 없음
        ↓
3. Page Fault 발생
        ↓
4. OS가 필요한 Page를 Disk에서 찾음
        ↓
5. 빈 Frame에 Page를 적재
        ↓
6. Page Table 갱신
        ↓
7. 중단되었던 작업 다시 실행
```

만약 사용할 수 있는 Frame이 없다면 기존 Page 중 하나를 선택해 내보내고 필요한 Page를 가져와야 할 수도 있다.

이때 어떤 Page를 내보낼지 결정하는 방법을 **Page Replacement Algorithm**이라고 한다.

대표적으로 FIFO, LRU 등의 방식이 있다.

이번에는 각각의 알고리즘보다 **메모리가 부족하면 기존 Page를 교체해야 할 수 있다**는 개념 정도를 이해하면 충분하다.

### Page Fault는 왜 느릴까?

Page Fault 자체는 가상 메모리를 사용하는 과정에서 정상적으로 발생할 수 있다.

문제는 필요한 Page를 Disk에서 가져와야 하는 경우이다.

```text
RAM 접근
→ 매우 빠름

Disk 접근
→ 상대적으로 매우 느림
```

따라서 Page Fault가 자주 발생하면 프로그램의 성능이 크게 떨어질 수 있다.

### Thrashing

Page Fault가 지나치게 자주 발생하면서 CPU가 실제 작업보다 Page를 가져오고 교체하는 데 많은 시간을 사용하는 상태를 **Thrashing**이라고 한다.

```text
메모리 부족
   ↓
Page Fault 증가
   ↓
Page 교체 증가
   ↓
다시 필요한 Page가 없음
   ↓
또 Page Fault
   ↓
실제 작업보다 Page 교체에 많은 시간 사용
```

즉, Virtual Memory를 사용한다고 해서 무조건 성능이 좋아지는 것은 아니며, Physical Memory가 지나치게 부족하면 오히려 성능이 크게 저하될 수 있다.

### Virtual Memory → Paging → Page Fault 흐름

```text
Physical Memory는 한정됨
        ↓
Virtual Memory 사용
        ↓
프로세스마다 가상 주소 공간 제공
        ↓
메모리를 Page 단위로 관리
        ↓
Paging
        ↓
Page Table을 이용해
Page → Frame 매핑
        ↓
필요한 Page가 RAM에 없다면
        ↓
Page Fault
        ↓
Disk에서 필요한 Page를 가져옴
```

Virtual Memory는 **프로세스에게 독립적인 가상 주소 공간을 제공하는 개념**이고, Paging은 이를 구현하기 위해 메모리를 Page와 Frame 단위로 관리하는 대표적인 방법이다.

그리고 필요한 Page가 현재 Physical Memory에 존재하지 않을 때 발생하는 것이 Page Fault이다.

## 6. JVM Runtime Data Area

> JVM이 Java 프로그램을 실행하면서 사용하는 메모리 영역이다.

Java 프로그램이 실행되면 JVM은 필요한 메모리를 여러 영역으로 나누어 관리한다.

대표적으로 Heap, Method Area, JVM Stack, PC Register, Native Method Stack이 있다.

```text
JVM Runtime Data Area

┌───────────────────────────────┐
│            Heap               │ ← 공유
├───────────────────────────────┤
│         Method Area           │ ← 공유
├───────────────┬───────────────┤
│   Thread A    │   Thread B    │
│               │               │
│ JVM Stack     │ JVM Stack     │
│ PC Register   │ PC Register   │
│ Native Stack  │ Native Stack  │
└───────────────┴───────────────┘
```

크게 보면 Heap과 Method Area는 여러 스레드가 공유하고, JVM Stack과 PC Register, Native Method Stack은 스레드마다 각각 생성된다.

| 영역 | 공유 여부 | 역할 |
|---|---|---|
| Heap | 공유 | 객체와 배열 저장 |
| Method Area | 공유 | 클래스 관련 정보 저장 |
| JVM Stack | 스레드별 | 메서드 호출과 지역 변수 관리 |
| PC Register | 스레드별 | 현재 실행 중인 명령의 위치 관리 |
| Native Method Stack | 스레드별 | Native Method 실행 지원 |

### Heap

객체와 배열 등이 생성되는 영역이다.

```java
User user = new User();
```

위 코드에서 `new User()`를 통해 생성된 객체는 일반적으로 Heap에 저장된다.

Heap은 여러 스레드가 공유하기 때문에 여러 스레드가 같은 객체에 접근할 수도 있다.

또한 Heap은 **Garbage Collector(GC)의 주요 관리 대상**이다.

### Method Area

클래스와 관련된 정보를 저장하는 영역이다.

대표적으로 클래스의 구조, 메서드 정보, 런타임 상수 풀 등의 클래스 관련 데이터가 저장된다.

```text
Class Loading
     ↓
Method Area
     ↓
클래스 관련 정보 저장
```

Java 8 이후 HotSpot JVM에서는 Method Area를 구현하는 영역으로 **Metaspace**를 사용한다.

여기서는 `Method Area = 클래스 관련 정보를 관리하는 JVM의 논리적 영역` 정도로 이해하면 충분하다.

### JVM Stack

각 스레드마다 독립적으로 생성된다.

메서드가 호출될 때마다 **Stack Frame**이 생성되고, 메서드 실행이 끝나면 제거된다.

```text
Thread A

┌─────────────────┐
│ methodC() Frame │
├─────────────────┤
│ methodB() Frame │
├─────────────────┤
│ methodA() Frame │
└─────────────────┘
```

Stack Frame에는 지역 변수, 연산을 위한 정보, 메서드 실행 후 돌아갈 정보 등이 포함된다.

따라서 같은 프로세스 안에서 여러 스레드가 실행되더라도 각 스레드는 자신의 JVM Stack을 가진다.

### PC Register

각 스레드가 **현재 어떤 JVM 명령을 실행하고 있는지**를 나타내기 위한 영역이다.

여러 스레드가 번갈아 실행되더라도 각 스레드가 자신의 실행 위치를 관리할 수 있어야 하므로 스레드마다 독립적으로 존재한다.

### Native Method Stack

Java가 아닌 C/C++ 등으로 작성된 Native Method를 실행할 때 사용되는 영역이다.

Java에서는 JNI(Java Native Interface)를 통해 Native Code를 호출할 수 있다.

이번에는 **Native Method 실행을 지원하는 스레드별 영역** 정도로 이해하면 충분하다.

## 7. JVM Heap / Stack

앞에서 OS 관점의 Heap과 Stack을 살펴봤다면, 이번에는 Java 코드에서 객체와 변수가 JVM의 메모리에 어떻게 위치하는지 살펴본다.

예를 들어 다음 코드가 있다고 가정하자.

```java
public void createUser() {
    User user = new User();
}
```

개념적으로 보면 다음과 같다.

```text
JVM Stack                    Heap

┌──────────────┐          ┌──────────────┐
│ user         │─────────→│ User 객체    │
└──────────────┘          └──────────────┘
```

지역 변수 `user`는 현재 메서드의 Stack Frame에 존재하고, `new User()`를 통해 생성된 객체는 Heap에 존재한다.

`user`에는 객체 자체가 들어가는 것이 아니라 **Heap에 있는 객체를 가리키는 참조 값**이 저장된다.

### 메서드가 종료되면?

`createUser()`의 실행이 끝나면 해당 Stack Frame은 제거된다.

```text
실행 중

Stack                    Heap

user ──────────────────→ User 객체


메서드 종료

Stack                    Heap

                         User 객체
                         ↑
                    참조가 없다면?
```

Stack에서 `user`가 사라졌다고 해서 Heap의 객체가 즉시 제거되는 것은 아니다.

더 이상 해당 객체를 참조하는 곳이 없다면 객체는 GC에 의해 회수될 수 있는 대상이 된다.

이 부분이 다음 GC와 연결된다.

### 여러 스레드가 실행된다면?

각 스레드는 자신의 Stack을 가지지만 Heap은 공유한다.

```text
Thread A Stack           Heap           Thread B Stack

referenceA ──────┐   ┌──────────┐   ┌────── referenceB
                 └──→│ Object A │←──┘
                     └──────────┘
```

따라서 여러 스레드가 같은 Heap 객체를 참조할 수 있다.

이 때문에 2주차에서 배운 Race Condition과 동기화 문제가 발생할 수 있다.

```text
각 Thread의 Stack
→ 독립

Heap
→ Thread 간 공유

같은 객체를 동시에 수정
→ Race Condition 가능
```

## 8. GC

> Garbage Collection은 더 이상 사용되지 않는 객체를 찾아 Heap 메모리에서 회수하는 과정이다.

Java에서는 개발자가 일반적으로 객체의 메모리를 직접 해제하지 않는다.

```java
User user = new User();

user = null;
```

객체를 더 이상 사용하지 않는다고 해서 즉시 메모리에서 삭제되는 것은 아니다.

GC가 해당 객체가 더 이상 사용되지 않는다고 판단하면 나중에 메모리를 회수한다.

### 어떤 객체가 필요 없는지 어떻게 판단할까?

GC는 단순히 참조 횟수가 0인지 확인하는 방식이 아니라, **GC Root에서 객체에 도달할 수 있는지(Reachability)**를 기준으로 판단한다.

```text
GC Root
   │
   ├──→ Object A
   │       ↓
   │    Object B
   │
   └──→ Object C

       Object D
```

Object A, B, C는 GC Root에서 도달할 수 있기 때문에 **Reachable Object**이다.

반면 Object D처럼 GC Root에서 도달할 수 없는 객체는 **Unreachable Object**가 되어 GC의 대상이 될 수 있다.

GC Root에는 실행 중인 스레드의 Stack에서 참조하는 객체 등 여러 종류가 존재한다.

여기서는 **GC Root에서 도달할 수 없으면 GC 대상이 될 수 있다**는 개념이 중요하다.

### Heap은 왜 영역을 나눌까?

많은 객체는 생성된 후 비교적 빠르게 사용되지 않게 된다는 특성을 이용해 Heap을 세대별로 관리할 수 있다.

대표적으로 Young Generation과 Old Generation으로 구분해서 이해할 수 있다.

```text
Heap

┌─────────────────────────────┐
│      Young Generation       │
│   새로 생성된 객체 중심     │
├─────────────────────────────┤
│       Old Generation        │
│   오래 살아남은 객체 중심   │
└─────────────────────────────┘
```

새로운 객체는 주로 Young Generation에서 시작하고, GC 이후에도 계속 살아남은 객체는 일정 과정을 거쳐 Old Generation으로 이동할 수 있다.

이를 통해 수명이 짧은 객체와 오래 살아남는 객체를 구분하여 효율적으로 메모리를 관리할 수 있다.

### Minor GC와 Major GC

Young Generation을 대상으로 수행되는 GC를 일반적으로 **Minor GC**라고 한다.

```text
Young Generation
      ↓
Minor GC
      ↓
사용하지 않는 객체 제거
```

Old Generation을 대상으로 하는 GC는 흔히 Major GC라고 부른다.

다만 실제 GC의 동작과 용어는 사용하는 Garbage Collector에 따라 차이가 있을 수 있으므로, `Minor GC = Young 영역 중심` 정도의 기본 개념을 이해하는 것이 중요하다.

### Stop-The-World

GC가 수행되는 과정에서는 애플리케이션의 스레드 실행을 일시적으로 멈춰야 하는 구간이 발생할 수 있다.

이를 Stop-The-World(STW)라고 한다.

```text
Application 실행
       ↓
       ↓
  Stop-The-World
       ↓
   GC 작업 수행
       ↓
Application 실행 재개
```

STW 시간이 길어지면 애플리케이션의 응답 시간이 증가할 수 있기 때문에 서버 성능을 볼 때 GC도 중요한 요소가 된다.

### GC가 있으면 메모리 문제는 없을까?

그렇지 않다.

객체가 실제로는 더 이상 필요하지 않더라도 프로그램 어딘가에서 계속 참조하고 있다면 GC는 해당 객체를 제거할 수 없다.

```text
GC Root
   ↓
Collection
   ↓
더 이상 필요하지 않은 Object

하지만 여전히 참조 중
→ Reachable
→ GC가 회수하지 못함
```

이처럼 필요하지 않은 객체의 참조가 계속 유지되면 Java에서도 **Memory Leak**이 발생할 수 있다.

결국 GC가 있다고 해서 메모리 관리 문제를 완전히 신경 쓰지 않아도 되는 것은 아니다.

### JVM Memory와 GC 흐름

```text
Java 프로그램 실행
        ↓
JVM Runtime Data Area
        ↓
┌─────────────────────────┐
│                         │
JVM Stack               Heap
│                         │
메서드 호출             객체 생성
지역 변수                   │
│                         ↓
Thread별 독립          여러 Thread가 공유
                          ↓
                    사용하지 않는 객체 발생
                          ↓
                         GC
                          ↓
                    Heap 메모리 회수
```

JVM의 Stack은 메서드 호출과 종료에 따라 Stack Frame이 생성되고 제거된다.

반면 Heap에 생성된 객체는 Stack Frame이 사라진다고 바로 제거되는 것이 아니라, 더 이상 Reachable하지 않을 때 GC의 대상이 될 수 있다.
