# Memory 구조와 Virtual Memory, 그리고 JVM Memory

지난 스터디에서는 Process와 Thread를 공부하면서 Process마다 독립적인 메모리 공간을 가지고, 같은 Process에 속한 Thread들은 Heap을 공유하지만 각자의 Stack을 가진다는 것을 알아봤다.

이번에는 여기서 조금 더 들어가서 **프로그램이 실행될 때 메모리는 어떤 구조로 사용되는지, 실제 RAM보다 큰 메모리를 어떻게 사용하는 것처럼 보이게 만드는지, 그리고 이러한 개념들이 Java와 JVM에서는 어떻게 연결되는지**를 중심으로 공부했다.

---

## 1. Memory 구조

### 프로그램을 실행하면 메모리에는 어떻게 올라갈까?

프로그램은 실행되기 전에는 SSD나 HDD 같은 저장장치에 존재하는 파일일 뿐이다.

프로그램을 실행하면 운영체제가 해당 프로그램을 Process로 만들고 실행에 필요한 내용을 메모리에 올린다.

이때 Process의 메모리 공간은 개념적으로 다음과 같이 나눠서 볼 수 있다.

```
높은 주소
┌──────────────────┐
│      Stack       │
│        ↓         │
│                  │
│        ↑         │
│       Heap       │
├──────────────────┤
│       Data       │
├──────────────────┤
│       Code       │
└──────────────────┘
낮은 주소
```

각 영역은 서로 다른 목적을 가지고 있다.

| 영역 | 역할 |
| --- | --- |
| Code | 실행할 프로그램의 명령어 |
| Data | 전역 변수, static 변수 등 |
| Heap | 동적으로 생성되는 데이터 |
| Stack | 함수 호출 정보, 지역 변수 등 |

그렇다면 굳이 하나의 메모리 공간을 이렇게 여러 영역으로 나누는 이유는 무엇일까?

프로그램에서 사용하는 데이터는 각각 **생명주기와 사용 목적이 다르기 때문**이다.

예를 들어 프로그램의 실행 코드는 실행 중 계속 필요한 반면 함수의 지역 변수는 해당 함수가 실행되는 동안에만 필요하다.

```
프로그램 코드
→ 프로그램 실행 동안 유지

전역 데이터
→ 비교적 오랫동안 유지

Heap 객체
→ 필요한 시점에 생성되고 필요 없어지면 제거

지역 변수
→ 함수 호출 시 생성되고 종료되면 제거
```

따라서 서로 다른 특징을 가진 데이터를 적절하게 관리하기 위해 메모리 영역을 구분해서 사용한다고 이해할 수 있다.

---

## 2. Stack과 Heap

Memory 구조에서 특히 자주 등장하는 것이 `Stack`과 `Heap`이다.

### Stack은 왜 필요할까?

프로그램에서는 함수가 계속 호출된다.

```java
public void controller() {
    service();
}

public void service() {
    repository();
}
```

실행 흐름을 보면

```
controller()
    ↓
service()
    ↓
repository()
```

순서로 호출되지만 종료될 때는 반대다.

```
repository() 종료
    ↓
service()로 복귀
    ↓
service() 종료
    ↓
controller()로 복귀
```

따라서 프로그램은 함수가 호출될 때마다

```
어디로 돌아가야 하는가?
지역 변수는 무엇인가?
현재 함수의 실행 상태는 무엇인가?
```

같은 정보를 저장해둘 필요가 있다.

이런 함수 호출 정보를 관리하는 데 Stack 구조가 잘 맞는다.

```
| repository() |
| service()    |
| controller() |
----------------
```

가장 마지막에 호출된 함수가 가장 먼저 종료되는 `LIFO` 구조이기 때문이다.

함수가 종료되면 해당 Stack Frame도 제거된다.

```
repository() 종료

| service()    |
| controller() |
----------------
```

이 때문에 Stack에 저장되는 데이터는 **생명주기가 비교적 명확하고 관리가 빠르다.**

---

### 그렇다면 Heap은 왜 필요할까?

모든 데이터를 Stack으로 관리할 수는 없다.

예를 들어 어떤 객체가 하나의 함수가 끝난 이후에도 계속 사용되어야 한다고 생각해볼 수 있다.

```
User user = new User();
```

`User` 객체가 현재 함수가 종료됐다고 바로 사라져버리면 다른 곳에서 이 객체를 사용할 수 없다.

그래서 함수 호출과 관계없이 더 유연한 생명주기를 가지는 데이터를 저장할 공간이 필요하고 이러한 데이터를 관리하는 영역이 Heap이다.

Java를 기준으로 단순화하면 다음과 같이 이해할 수 있다.

```
User user = new User();
```

```
Stack                     Heap

user ─────────────────→ User 객체
```

여기서 지역 변수 `user`는 객체 자체라기보다는 Heap에 생성된 객체를 참조하고 있다.

또한 지난주에 공부했던 Thread와도 연결된다.

```
Process

Heap
┌─────────────────────┐
│       User 객체      │ ← Thread들이 공유 가능
└─────────────────────┘

Thread 1             Thread 2
┌──────────┐         ┌──────────┐
│ Stack    │         │ Stack    │
└──────────┘         └──────────┘
```

같은 Process 내부의 Thread들은 Heap 영역을 공유할 수 있지만 각각 별도의 Stack을 가진다.

그래서 지난주에 배웠던 Race Condition도 결국 **여러 Thread가 Heap에 존재하는 공유 가변 데이터에 동시에 접근할 수 있기 때문에 발생할 수 있는 문제**라고 연결해서 이해할 수 있었다.

---

## 3. Virtual Memory

### RAM이 부족하면 프로그램을 실행할 수 없을까?

처음에는 프로그램이 사용하는 메모리가 전부 RAM에 그대로 올라간다고 생각했다.

하지만 실제 시스템에서는 여러 프로그램이 동시에 실행된다.

```
Chrome
IntelliJ
Spring Boot
MySQL
Docker
...
```

각 Process가 사용하는 메모리를 모두 물리 RAM에 그대로 올려야 한다면 메모리 관리가 상당히 어려울 것이다.

또 다른 문제도 있다.

Process A가 사용하는 주소와 Process B가 사용하는 주소를 프로그램이 직접 물리 메모리 기준으로 관리해야 한다면 프로그램마다 현재 RAM의 어느 위치가 비어 있는지 알아야 한다.

이를 해결하기 위해 운영체제는 Process에게 실제 물리 메모리 주소를 직접 사용하게 하는 대신 **Virtual Address Space**, 즉 가상 주소 공간을 제공한다.

```
Process A
Virtual Address
0x1000
0x2000
0x3000
     ↓
   Mapping
     ↓
Physical Memory

Process B
Virtual Address
0x1000
0x2000
0x3000
     ↓
   Mapping
     ↓
Physical Memory
```

Process A와 Process B가 같은 `0x1000`이라는 주소를 사용한다고 해도 실제 RAM에서는 서로 다른 위치를 가리킬 수 있다.

즉 각 Process 입장에서는

```
"내가 메모리를 독립적으로 사용하고 있다."
```

라고 생각할 수 있다.

이것이 Virtual Memory의 중요한 목적 중 하나이다.

Virtual Memory를 이용하면

```
Process별 독립적인 주소 공간 제공
            +
물리 메모리보다 큰 주소 공간 사용 가능
            +
메모리 보호와 관리가 쉬워짐
```

같은 장점을 얻을 수 있다.

다만 **Virtual Memory = 단순히 RAM이 부족할 때 Disk를 RAM처럼 사용하는 기술**이라고만 이해하면 부족하다.

Disk를 활용할 수 있다는 것도 Virtual Memory의 한 부분이지만 더 중요한 핵심은 **Process가 사용하는 가상 주소와 실제 물리 주소를 분리한다는 것**이라고 이해했다.

---

## 4. Paging

### 가상 주소를 실제 RAM과 어떻게 연결할까?

Virtual Memory를 사용한다면 다음 문제가 생긴다.

```
Process가 사용하는 Virtual Address
              ↓
실제 RAM의 어느 위치인가?
```

이 매핑을 관리해야 한다.

이를 효율적으로 관리하기 위해 메모리를 일정한 크기의 단위로 나누는데 이를 `Paging`이라고 한다.

가상 메모리를 나눈 단위를 `Page`, 물리 메모리를 나눈 단위를 `Frame`이라고 한다.

```
Virtual Memory

Page 0
Page 1
Page 2
Page 3
Page 4

       ↓ Page Table

Physical Memory

Frame 0
Frame 1  ← Page 3
Frame 2  ← Page 0
Frame 3
Frame 4  ← Page 2
```

중요한 점은 Virtual Memory에서 연속되어 있다고 해서 Physical Memory에서도 연속될 필요가 없다는 것이다.

```
Virtual

Page 0
Page 1
Page 2

Physical

Page 0 → Frame 7
Page 1 → Frame 2
Page 2 → Frame 10
```

운영체제는 `Page Table`을 이용해 어떤 Page가 어떤 Frame에 위치하는지 관리한다.

CPU가 특정 가상 주소에 접근하면 이를 기반으로 실제 물리 주소를 찾아간다.

개념적으로는

```
Virtual Address
      ↓
Page Number 확인
      ↓
Page Table 조회
      ↓
Frame Number 확인
      ↓
Physical Address
```

와 같은 과정이라고 볼 수 있다.

이 구조 덕분에 Process는 실제 물리 메모리가 어떻게 배치되어 있는지 알 필요가 없다.

---

## 5. Page Fault

### 필요한 Page가 RAM에 없다면?

Paging을 사용한다고 해서 Process의 모든 Page가 항상 RAM에 올라가 있어야 하는 것은 아니다.

예를 들어 Process가 다음과 같은 Page를 가지고 있다고 생각해보자.

```
Process

Page 0 → RAM
Page 1 → RAM
Page 2 → Disk
Page 3 → Disk
```

현재 실행에 필요한 Page만 RAM에 있고 당장 필요하지 않은 Page는 Disk에 있을 수 있다.

그런데 CPU가 `Page 2`에 접근하려고 한다면 문제가 발생한다.

```
CPU
 ↓
Page 2 접근
 ↓
Page Table 확인
 ↓
RAM에 없음
 ↓
Page Fault
```

이처럼 **Process가 접근하려는 Page가 현재 Physical Memory에 존재하지 않을 때 발생하는 상황**을 Page Fault라고 한다.

Page Fault가 발생하면 운영체제가 개입한다.

```
1. CPU가 특정 Page 접근

2. Page Table 확인

3. RAM에 Page가 없음
        ↓
   Page Fault 발생

4. OS가 Disk에서 해당 Page를 찾음

5. RAM의 Frame에 Page를 로드

6. Page Table 갱신

7. 중단됐던 명령 다시 실행
```

만약 RAM에 빈 Frame이 없다면 기존 Page 중 하나를 내보내야 할 수도 있다.

```
RAM 가득 참
   ↓
어떤 Page를 내보낼지 결정
   ↓
Page Replacement
   ↓
필요한 Page 로드
```

이때 FIFO, LRU 같은 Page Replacement Algorithm을 사용할 수 있다.

---

### Page Fault가 왜 성능에 영향을 줄까?

RAM과 Disk의 접근 속도에는 큰 차이가 있다.

Page가 이미 RAM에 있다면 바로 메모리에 접근할 수 있지만 Page Fault가 발생하면 저장장치에서 데이터를 가져오는 과정이 추가된다.

```
RAM 접근
→ 빠름

Page Fault
→ OS 개입
→ Disk I/O 가능
→ Page 로드
→ Page Table 변경
→ 다시 실행
```

따라서 Page Fault가 지나치게 많이 발생하면 프로그램 성능이 크게 떨어질 수 있다.

특히 계속 필요한 Page를 RAM에 올렸다가 다시 내보내는 상황이 반복된다면

```
Page A 로드
↓
Page B 필요
↓
Page A 제거
↓
Page B 로드
↓
다시 Page A 필요
↓
Page B 제거
...
```

실제 프로그램 실행보다 Page를 교체하는 작업에 더 많은 시간을 사용할 수 있는데 이런 상황을 `Thrashing`이라고 한다.

결국 Virtual Memory가 있다고 해서 RAM 크기를 전혀 신경 쓰지 않아도 되는 것은 아니다.

---

# Backend 심화

## 6. JVM Runtime Data Area

### 지금까지 배운 Memory 구조가 Java에서는 어떻게 보일까?

Java 프로그램은 JVM 위에서 실행된다.

```
Java Application
       ↓
      JVM
       ↓
Operating System
       ↓
Hardware
```

따라서 Java 개발자는 운영체제의 메모리를 직접 다루기보다는 JVM이 관리하는 메모리 구조를 주로 보게 된다.

JVM에는 실행 중 데이터를 관리하기 위한 여러 Runtime Data Area가 존재한다.

개념적으로 정리하면 다음과 같다.

```
JVM

├── Heap                 ← Thread 공유
│
├── Method Area          ← Thread 공유
│
├── JVM Stack            ← Thread마다 생성
│
├── PC Register          ← Thread마다 생성
│
└── Native Method Stack  ← Thread마다 생성
```

### Heap

객체와 배열 등이 생성되는 공간이다.

```
User user = new User();
```

```
JVM Stack               Heap

user ───────────────→ User Object
```

JVM의 모든 Thread가 Heap을 공유하며 Heap의 객체 중 더 이상 사용되지 않는 객체는 Garbage Collector의 대상이 된다.

---

### JVM Stack

각 Thread마다 하나씩 존재한다.

메서드가 호출될 때마다 `Stack Frame`이 생성되고 메서드 실행이 끝나면 제거된다.

```
Thread

JVM Stack

┌─────────────────┐
│ repository()    │
├─────────────────┤
│ service()       │
├─────────────────┤
│ controller()    │
└─────────────────┘
```

Stack Frame에는 지역 변수, 연산에 필요한 값, 메서드 실행과 관련된 정보 등이 저장된다.

재귀 호출을 지나치게 많이 하면 Stack Frame이 계속 쌓인다.

```java
public void recursive() {
    recursive();
}
```

```
recursive()
recursive()
recursive()
recursive()
recursive()
...
```

결국 Stack 공간을 초과하면 Java에서는 `StackOverflowError`가 발생할 수 있다.

---

### PC Register

각 Thread가 **현재 어떤 JVM 명령어를 실행하고 있는지**를 나타내기 위한 영역이다.

Thread마다 실행 위치가 다르기 때문에 PC Register 역시 Thread마다 별도로 존재한다.

```
Thread A → instruction 100
Thread B → instruction 240
Thread C → instruction 52
```

Context Switching이 발생하더라도 각각의 Thread가 자신이 실행하던 흐름을 이어갈 수 있어야 하기 때문에 이러한 실행 위치 정보가 필요하다.

---

### Method Area

클래스와 관련된 정보를 저장하는 영역이다.

예를 들어

```
클래스 정보
메서드 정보
필드 정보
Runtime Constant Pool
```

등이 저장된다.

HotSpot JVM에서는 Java 8 이후 Method Area를 구현하는 방식으로 `Metaspace`가 사용된다.

여기서 주의할 점은

```
Method Area = JVM Specification에서 정의한 논리적인 영역

Metaspace = HotSpot JVM에서 이를 구현하는 방식
```

이라는 것이다.

둘을 완전히 같은 개념으로 보는 것보다는 명세와 구현의 관계로 이해하는 것이 좋다.

---

### Native Method Stack

Java 코드가 아닌 C/C++ 등으로 구현된 Native Method를 실행할 때 사용되는 Stack이다.

Java에서는 JNI(Java Native Interface)를 통해 Native Code를 호출할 수 있다.

```
Java Code
    ↓
   JNI
    ↓
Native Code
```

일반적인 Spring 백엔드 개발에서 직접 다룰 일은 많지 않지만 JVM도 결국 운영체제 위에서 동작하는 프로그램이기 때문에 Java Heap만 사용하는 것은 아니라는 점을 이해할 수 있었다.

---

## 7. JVM Heap과 Stack

### Java 코드 하나는 실제로 어디에 저장될까?

다음 코드를 생각해봤다.

```java
public void createUser() {

    int age = 25;

    User user = new User();
    user.setName("Kim");
}
```

단순화해서 보면 다음처럼 이해할 수 있다.

```
Thread Stack

createUser() Stack Frame

age = 25
user = reference
       │
       │
       ↓
Heap

User Object
┌───────────────┐
│ name = "Kim"  │
└───────────────┘
```

메서드가 종료되면 Stack Frame은 제거된다.

```
createUser() 종료
       ↓
Stack Frame 제거
       ↓
user reference 제거
```

그런데 Heap의 `User Object`는 Stack처럼 메서드가 끝났다고 즉시 제거되는 구조가 아니다.

다른 곳에서도 해당 객체를 참조하고 있을 수 있기 때문이다.

```
Reference A ─┐
             ├──→ User Object
Reference B ─┘
```

따라서 JVM은 객체가 실제로 더 이상 사용되지 않는지를 판단하고 필요 없는 객체를 정리해야 한다.

여기서 Garbage Collection이 등장한다.

---

## 8. Garbage Collection

### Java에서는 왜 free()를 하지 않을까?

C 같은 언어에서는 개발자가 직접 메모리를 할당하고 해제할 수 있다.

```
malloc(...)
free(...)
```

하지만 메모리를 직접 관리하면 실수할 가능성도 생긴다.

```
메모리를 할당함
       ↓
사용 완료
       ↓
해제를 깜빡함
       ↓
Memory Leak
```

반대로 아직 사용 중인 메모리를 잘못 해제하는 문제도 발생할 수 있다.

Java에서는 이러한 메모리 관리를 JVM의 Garbage Collector가 담당한다.

```
Object 생성
    ↓
Heap 저장
    ↓
사용
    ↓
더 이상 접근할 수 없음
    ↓
Garbage
    ↓
GC가 메모리 회수
```

중요한 것은 단순히

```
변수 = null
→ 바로 삭제
```

되는 것은 아니라는 것이다.

GC의 중요한 판단 기준은 해당 객체에 더 이상 도달할 수 있는가(Reachability)이다.

---

### Reachable과 Unreachable

예를 들어

```java
User user = new User();
```

현재 `user`를 통해 객체에 접근할 수 있다.

```
GC Root
   ↓
 user
   ↓
User Object
```

따라서 이 객체는 `Reachable`하다.

그런데 참조가 사라진다면

```java
user = null;
```

```
GC Root

       User Object
```

더 이상 GC Root에서 해당 객체에 도달할 수 없게 될 수 있다.

이런 객체가 `Unreachable Object`가 되고 GC의 수거 대상이 될 수 있다.

---

## 9. Generational GC

### 모든 객체를 매번 검사하면 비효율적이지 않을까?

프로그램에서는 객체가 굉장히 많이 생성된다.

예를 들어 웹 요청 하나를 처리하면서도 DTO, Entity, Collection 등 다양한 객체가 만들어질 수 있다.

그런데 GC가 매번 Heap 전체를 검사한다면 비용이 상당히 클 것이다.

여기서 대부분의 객체가 오래 살아남지 않는다는 특성을 이용할 수 있다.

```
객체 생성
 ↓
잠깐 사용
 ↓
금방 필요 없어짐
```

이러한 특성을 `Weak Generational Hypothesis`와 연결해서 볼 수 있다.

그래서 전통적인 Generational GC에서는 Heap을 세대별로 관리하는 방식을 사용한다.

단순화하면

```
Heap

Young Generation
├── Eden
├── Survivor 0
└── Survivor 1

Old Generation
```

새로운 객체는 주로 Young 영역에서 시작한다.

```
new User()
     ↓
    Eden
```

GC 이후에도 계속 살아남는 객체는 Survivor 영역 등을 거치면서 살아남은 횟수가 증가하고 조건을 만족하면 Old Generation으로 이동할 수 있다.

```
Eden
 ↓
Survivor
 ↓
Survivor
 ↓
...
 ↓
Old Generation
```

따라서 수명이 짧은 객체는 Young 영역에서 빠르게 정리하고 오래 살아남는 객체는 별도로 관리할 수 있다.

다만 실제 Heap 구조와 GC 동작 방식은 사용하는 Garbage Collector에 따라 달라질 수 있기 때문에 모든 GC가 무조건 정확히 같은 구조로 동작한다고 이해하면 안 된다.

---

## 10. GC가 백엔드 성능과 무슨 관계가 있을까?

Spring Boot 서버에서도 요청을 처리할 때 계속 객체가 생성된다.

```
HTTP Request
   ↓
Request DTO
   ↓
Service Object 사용
   ↓
Entity
   ↓
Response DTO
   ↓
HTTP Response
```

요청이 많아질수록 많은 객체가 생성되고 사라질 수 있다.

이때 Heap이 부족하거나 GC가 너무 자주 발생한다면 애플리케이션 성능에도 영향을 줄 수 있다.

특히 일부 GC 과정에서는 애플리케이션 Thread의 실행이 잠시 멈추는 `Stop-The-World(STW)`가 발생할 수 있다.

```
Application Threads

Thread 1 ────────┐
Thread 2 ────────┼── STOP
Thread 3 ────────┘
                    ↓
                   GC
                    ↓
Thread 1 ─────────────→
Thread 2 ─────────────→
Thread 3 ─────────────→
```

따라서 백엔드에서 메모리 문제를 볼 때 단순히

```
"RAM이 몇 GB인가?"
```

만 확인하는 것이 아니라

```
Heap이 얼마나 사용되고 있는가?

객체가 계속 증가하고 있지는 않은가?

GC가 얼마나 자주 발생하는가?

GC Pause가 얼마나 긴가?

Old Generation이 계속 증가하는가?
```

같은 부분도 함께 볼 수 있다.

예를 들어 객체가 계속 참조된 상태로 남아 있다면 GC 입장에서는 실제 애플리케이션에서 필요하지 않은 객체라도 제거할 수 없다.

```
Cache / Collection
       ↓
Object
       ↓
Object
       ↓
Object
```

참조가 계속 유지된다면 GC는 해당 객체를 `Reachable`하다고 판단한다.

따라서 Java에서도 GC가 있다고 해서 Memory Leak이 절대 발생하지 않는 것은 아니다.

---

# 전체 흐름 정리

이번에 공부한 내용을 하나의 흐름으로 연결하면 다음과 같이 정리할 수 있었다.

```
프로그램 실행
      ↓
Process 생성
      ↓
독립적인 Virtual Address Space 제공
      ↓
Code / Data / Heap / Stack 등의 영역 사용
      ↓
Virtual Memory를 Page 단위로 관리
      ↓
Page Table을 통해 Physical Memory와 연결
      ↓
필요한 Page가 RAM에 없다면
      ↓
Page Fault
      ↓
OS가 Page를 RAM에 로드
```

그리고 Java에서는 그 위에서 JVM이 다시 메모리를 관리한다.

```
Operating System
        ↓
Virtual Memory
        ↓
JVM Process
        ↓
JVM Runtime Data Area

├── Heap
├── Method Area
├── JVM Stack
├── PC Register
└── Native Method Stack
```

여기서 Java 객체는 주로 Heap에 생성되고

```
Object 생성
   ↓
Heap
   ↓
Reference를 통해 사용
   ↓
더 이상 Reachable하지 않음
   ↓
Garbage Collection 대상
   ↓
메모리 회수
```

라는 과정으로 관리된다.

# 스터디하면서 느낀 점

지난주 Process와 Thread를 공부할 때는 단순히 **Thread는 Heap을 공유하고 Stack은 따로 가진다** 정도로 이해했는데 이번에 메모리를 공부하면서 왜 이런 구조가 필요한지 조금 더 이해할 수 있었다.

특히 처음에는 `Virtual Memory`를 단순히 **RAM이 부족할 때 Disk를 추가 RAM처럼 사용하는 기술**이라고 생각했는데, 실제로는 각 Process에게 독립적인 가상 주소 공간을 제공하고 이를 Physical Memory와 매핑하는 것이 중요한 개념이라는 것을 알게 되었다.

또 OS에서 배운 내용과 JVM의 메모리 구조가 별개의 내용이 아니라는 것도 흥미로웠다.

```
OS
Virtual Memory / Paging / Page Fault
          ↓
JVM Process
          ↓
JVM Heap / Stack
          ↓
Java Object
          ↓
Garbage Collection
```

결국 Spring Boot 애플리케이션에서 `new User()` 하나를 만드는 것도 가장 아래에서는 운영체제가 제공하는 Virtual Memory 위에서 JVM이 확보한 메모리를 다시 관리하면서 이루어지는 작업이라고 볼 수 있다.

특히 백엔드에서는 서버가 오랜 시간 실행되면서 수많은 요청을 처리하기 때문에 메모리가 계속 증가하거나 GC가 지나치게 자주 발생하는 문제를 실제 장애나 성능 문제로 만날 수 있다.

그래서 단순히 **Stack에는 지역 변수, Heap에는 객체가 저장된다**를 외우는 것보다 OS가 메모리를 어떻게 관리하고 그 위에서 JVM이 다시 메모리를 어떻게 관리하는지를 연결해서 이해하는 것이 더 중요하다고 느꼈다.
