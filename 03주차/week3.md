# 3주차 - OS - Memory / Virtual Memory 개념 정리

## 목차
- [키워드 연관관계](#키워드-연관관계)
- [Memory 구조](#memory-구조)
- [Stack/Heap](#stackheap)
- [Virtual Memory](#virtual-memory)
- [Paging](#paging)
- [Page Fault](#page-fault)
- [Backend 심화 (선택)](#backend-심화-선택)
  - [JVM Runtime Data Area](#jvm-runtime-data-area)
  - [Heap/Stack (JVM)](#heapstack-jvm)
  - [GC](#gc)

---

## 키워드 연관관계

각 개념이 독립된 게 아니라, 아래처럼 큰 개념 → 세부 메커니즘 → 자바 구현체로 이어져 있다.

```
Virtual Memory
├─ Memory 구조
│    ├─ Stack/Heap (OS 일반 관점)
│    └─ JVM Runtime Data Area (자바 관점) → Heap/Stack (JVM) → GC
└─ Paging → Page Fault

```

| 연결 | 관계 설명 |
|---|---|
| Virtual Memory → Memory 구조 | 가상 주소 공간 안에 Code/Data/Heap/Stack 레이아웃이 구성됨 |
| Memory 구조 → Stack/Heap | 그중 실행 중 계속 변하는 두 영역 |
| Memory 구조 → JVM Runtime Data Area | OS 레벨 메모리 구조를 JVM이 자바 프로그램용으로 다시 나눠 관리 |
| JVM Runtime Data Area → Heap/Stack (JVM) | JVM도 똑같이 Heap(객체)과 Stack(호출 정보)으로 나뉨 |
| Heap (JVM) → GC | Heap에 쌓인 객체 중 더 이상 안 쓰는 것을 자동으로 회수 |
| Virtual Memory → Paging → Page Fault | 가상 주소를 페이지 단위로 매핑하고, 해당 페이지가 물리 메모리에 없으면 Page Fault 발생 |

---

## Memory 구조



| 영역 | 저장 내용 |
|---|---|
| Code | 실행할 명령어 |
| Data | 전역 변수, 정적(static) 변수 |
| Heap | 동적으로 할당한 데이터  |
| Stack | 함수 호출 정보, 지역 변수, 매개변수 |



---

## Stack/Heap


| 구분 | Stack | Heap |
|---|---|---|
| 저장 대상 | 지역 변수, 매개변수, 리턴 주소 | 동적으로 할당된 객체 |
| 할당/해제 | 함수 호출/종료 시 자동 (LIFO) | 개발자가 할당 |
| 속도 | 빠름 | 상대적으로 느림 |



- 예시(자바): `int x = 5;`의 `x`는 Stack에, `new Person()`으로 만든 객체 실체는 Heap에 저장되고 그 객체를 가리키는 참조 변수는 Stack에 저장된다.

---

## Virtual Memory


프로세스마다 실제 물리 메모리 크기와 무관하게, 독립적이고 연속된 **가상의 주소 공간**을 제공하는 기법.


---

## Paging

> EX) 이사할 때 짐을 고정 크기 박스(페이지) 단위로 나눠서 창고(물리 메모리)의 빈 칸 아무 데나 넣어두고, "몇 번 박스가 어디 있는지" 목록(페이지 테이블)으로 관리하는 것.

가상 메모리를 고정 크기의 **Page** 단위로 나누고, 물리 메모리도 같은 크기의 **Frame**으로 나눠서 매핑하는 방식.

- 가상 주소 → 물리 주소 변환은 페이지 테이블(Page Table)이 담당하며, `MMU(Memory Management Unit)`가 이를 처리


---

## Page Fault

 Page Fault가 너무 잦으면 디스크 I/O가 급증해 성능이 크게 떨어지는 **스래싱(Thrashing)** 현상이 발생할 수 있다

---

## Backend 심화 (선택)

### JVM Runtime Data Area

> OS가 정해준 프로세스 메모리 구조를, JVM이 자바 프로그램 실행에 맞게 자기만의 방식으로 세분화해서 쓰는 것.

JVM이 자바 프로그램 실행을 위해 관리하는 메모리 영역.

| 영역 | 설명 | 공유 여부 |
|---|---|---|
| Method Area | 클래스 정보, static 변수, 상수 풀 | 스레드 공유 |
| Heap | `new`로 생성된 객체, 배열 | 스레드 공유 |
| Stack | 메서드 호출 시 생성되는 지역 변수, 호출 정보 | 스레드별 독립 |
| PC Register | 현재 실행 중인 명령어 주소 | 스레드별 독립 |
| Native Method Stack | 네이티브(C/C++) 메서드 호출 정보 | 스레드별 독립 |


---

### Heap/Stack (JVM)

>  OS의 Stack/Heap을 JVM 안에서 그대로 재현한 것. 다만 Heap은 모든 스레드가 같이 쓰는 창고이고, Stack은 스레드마다 하나씩 따로 갖는 개인 메모지다.

- **Heap**: `new`로 생성된 모든 객체와 배열이 저장되는 공간. 모든 스레드가 공유하며, GC의 관리 대상
- **Stack**: 스레드마다 하나씩 생성되며, 메서드를 호출할 때마다 Stack Frame이 쌓이고 리턴되면 제거됨(LIFO)

-  여러 스레드가 동시에 실행돼도 각자의 지역 변수는 자기 Stack에만 있어 안전하지만, Heap의 객체는 공유되므로 동시성 문제(Race Condition)에 주의

---

### GC

> EX) 창고지기가 정기적으로 돌아다니며, 더 이상 아무도 찾지 않는 짐(참조되지 않는 객체)을 알아서 치워주는 것.

JVM Heap에 쌓인 객체 중 더 이상 어디서도 참조되지 않는 객체를 자동으로 탐지해서 메모리를 회수하는 기능. 개발자가 직접 메모리를 해제하지 않아도 된다.

- 방식: 참조를 추적해 안 쓰는 객체를 표시하고 제거하는 Mark & Sweep, 객체 생존 기간에 따라 영역을 나눠 관리하는 Generational GC 등

