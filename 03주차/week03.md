# 3주차 - Memory / Virtual Memory

## 프로세스의 메모리 구조

운영체제는 프로세스마다 독립적인 가상 주소 공간을 제공한다. 일반적인 구조는 다음과 같으며, 실제 배치와 증가 방향은 운영체제와 시스템에 따라 달라질 수 있다.

```text
높은 주소
┌──────────────────┐
│      Stack       │ 함수 호출 정보, 지역 변수
│        ↓         │
├──────────────────┤
│                  │
│     빈 공간        │
│                  │
├──────────────────┤
│        ↑         │
│       Heap       │ 동적 할당 메모리
├──────────────────┤
│   Data / BSS     │ 전역 변수, 정적 변수
├──────────────────┤
│       Code       │ 실행할 기계어 명령
└──────────────────┘
낮은 주소
```

### Code 영역

- 프로그램의 기계어 명령 저장
- Text 영역이라고도 함
- 일반적으로 읽기 전용으로 보호
- 실행 중 크기가 거의 변하지 않음

### Data와 BSS 영역

- 전역 변수와 정적 변수 저장
- Data: 초기값이 명시된 변수
- BSS: 초기화하지 않았거나 0으로 초기화되는 변수
- 프로세스 시작 시 생성되고 종료 시 제거

```c
int initialized = 10; // Data
int uninitialized;    // BSS
```

### Heap 영역

- 실행 중 동적으로 할당한 메모리
- 개발자가 직접 해제하거나 Garbage Collector가 관리
- 잘못 관리하면 메모리 누수 발생 가능

### Stack 영역

- 함수 호출 정보
- 매개변수와 지역 변수
- 반환 주소
- 함수 호출 시 생성되고 종료 시 자동 제거

```c
int globalValue = 10;

void example() {
    int localValue = 20;
    int* dynamicValue = malloc(sizeof(int));
}
```

- `globalValue`: Data 영역
- `localValue`: Stack 영역
- `dynamicValue` 포인터 변수: Stack 영역
- `malloc()`으로 할당한 실제 메모리: Heap 영역

### 핵심 정리

- 프로세스마다 독립적인 가상 주소 공간 사용
- Code: 실행 명령
- Data·BSS: 전역 변수와 정적 변수
- Heap: 동적 할당 데이터
- Stack: 함수 호출과 지역 데이터

## Stack과 Heap

### Stack

- 함수 호출과 관련된 데이터 저장
- 매개변수, 지역 변수, 반환 주소 등으로 Stack Frame 구성
- 각 스레드가 독립적인 Stack 보유
- LIFO 방식으로 관리
- 함수 호출과 종료에 따라 자동으로 생성·제거
- Stack Pointer 이동으로 할당해 일반적으로 빠름
- 크기가 제한되어 깊은 재귀 호출 시 Stack Overflow 가능

### Heap

- 크기와 수명을 동적으로 결정하는 데이터 저장
- 같은 프로세스의 스레드들이 공유 가능
- 함수가 종료돼도 데이터 유지 가능
- 직접 해제하거나 Garbage Collector가 관리
- 할당과 해제를 위한 관리 비용 발생
- 잘못 관리하면 메모리 누수와 단편화 발생 가능

### 비교

| Stack | Heap |
| --- | --- |
| 함수 호출과 지역 데이터 | 동적으로 생성한 데이터 |
| 스레드마다 독립적 | 같은 프로세스에서 공유 가능 |
| 자동으로 생성·제거 | 직접 해제하거나 GC가 관리 |
| 크기가 비교적 작음 | 비교적 큰 데이터 저장 가능 |
| 할당과 해제가 빠름 | 관리 비용이 비교적 큼 |
| Stack Overflow 가능 | 메모리 누수와 단편화 가능 |

### 데이터의 수명

```c
void example() {
    int local = 10;
    int* dynamic = malloc(sizeof(int));
}
```

함수 종료 후:

- `local`: Stack Frame과 함께 제거
- `dynamic` 포인터 변수: Stack에서 제거
- `malloc()`으로 만든 Heap 공간: 자동으로 제거되지 않음
- Heap 공간을 해제하지 않고 참조를 잃으면 메모리 누수 발생

### 핵심 정리

- 종료 조건 없는 재귀 호출: Stack Overflow
- 해제하지 않은 동적 메모리: Memory Leak
- Stack은 실행 흐름 중심, Heap은 동적 데이터 수명 중심

## Virtual Memory

### 개념

- 프로세스마다 독립적인 주소 공간이 있는 것처럼 제공하는 메모리 관리 방식
- 프로세스가 사용하는 가상 주소와 RAM의 물리 주소를 분리
- CPU가 만든 가상 주소를 MMU와 Page Table을 이용해 물리 주소로 변환

```text
가상 주소
   ↓
Page Table 조회
   ↓
물리 주소
```

### 프로세스 격리

```text
프로세스 A의 가상 주소 1000 → 물리 주소 5000
프로세스 B의 가상 주소 1000 → 물리 주소 9000
```

- 같은 가상 주소도 프로세스별로 다른 물리 주소에 연결 가능
- 한 프로세스가 다른 프로세스의 메모리에 직접 접근하는 것을 제한
- 공유 메모리는 여러 프로세스가 같은 물리 메모리를 사용하도록 의도적으로 연결

### 장점

- 프로그램에 연속된 주소 공간처럼 제공
- 실제 RAM의 빈 위치를 프로그램이 직접 관리할 필요 없음
- 현재 필요한 부분만 RAM에 적재 가능
- 제한된 물리 메모리를 여러 프로세스가 효율적으로 사용
- 물리 메모리보다 큰 가상 주소 공간 사용 가능

### TLB

- 최근의 가상 주소와 물리 주소 변환 결과를 저장하는 캐시
- Page Table을 매번 조회하는 비용 감소
- TLB에 변환 정보가 있으면 빠르게 물리 주소 확인

### 핵심 정리

- 프로세스가 보는 주소: 가상 주소
- RAM에서 사용하는 실제 주소: 물리 주소
- MMU와 Page Table이 주소 변환
- 프로세스별 Page Table을 통한 메모리 격리
- 필요한 데이터만 물리 메모리에 적재 가능

## Paging

### 개념

- 가상 메모리를 고정 크기의 Page로 분할
- 물리 메모리를 같은 크기의 Frame으로 분할
- 가상 메모리의 연속된 Page를 떨어진 Frame에 배치 가능

```text
가상 메모리                 물리 메모리
Page 0 ─────────────────→ Frame 3
Page 1 ─────────────────→ Frame 7
Page 2 ─────────────────→ Frame 1
```

### Page Table

- Page 번호와 Frame 번호의 연결 정보 저장
- 현재 RAM에 존재하는지 표시
- 읽기·쓰기·실행 권한 관리
- 내용 변경 여부와 접근 여부 등의 상태 저장 가능

### 주소 변환

```text
가상 주소 = Page 번호 + Page 내부 Offset
물리 주소 = Frame 번호 + 같은 Offset
```

Page 크기가 4KB이고 가상 주소가 `9000`인 경우:

```text
4KB = 4096바이트
9000 = 2 × 4096 + 808

Page 번호: 2
Offset: 808
```

- Page 번호: 주소를 Page 크기로 나눈 몫
- Offset: 주소를 Page 크기로 나눈 나머지
- Page Table에서 Frame 번호를 찾고 Offset은 그대로 사용

### 장점

- 연속된 가상 주소 공간 제공
- 물리 Frame을 연속해서 배치할 필요 없음
- 필요한 Page만 RAM에 적재 가능
- 고정 크기 블록으로 외부 단편화 감소

### 단점

- Page Table 저장 공간 필요
- 주소 변환 비용 발생
- 마지막 Page의 남는 공간으로 내부 단편화 가능
- 작은 Page는 Page Table 크기 증가
- 큰 Page는 내부 단편화와 불필요한 데이터 적재 증가 가능

### 핵심 정리

- Page와 Frame의 크기는 같음
- Page Table이 가상 Page와 물리 Frame을 연결
- Offset은 주소 변환 전후 동일
- Paging은 외부 단편화를 줄이지만 내부 단편화가 발생할 수 있음

## Page Fault

### 개념

- 프로세스가 접근하려는 Page가 현재 RAM에 없을 때 발생
- 필요한 Page만 RAM에 올리는 Demand Paging에서는 정상적인 처리 과정
- 항상 프로그램 오류나 Segmentation Fault를 의미하지 않음

### 처리 과정

1. CPU가 가상 주소에 접근
2. Page Table에서 Page가 RAM에 없음을 확인
3. 운영체제로 제어권 전달
4. 유효한 메모리 접근인지 검사
5. 디스크에서 필요한 Page 위치 확인
6. 비어 있는 Frame을 찾거나 기존 Page 교체
7. 필요한 Page를 RAM으로 적재
8. Page Table과 TLB 갱신
9. 중단됐던 명령 다시 실행

### 유효하지 않은 접근

- 유효한 Page가 RAM에 없음: Page 적재 후 실행 재개
- 할당되지 않은 주소: 프로세스에 오류 전달
- 읽기 전용 Page에 쓰기 시도: 권한 오류
- Unix 계열에서는 Segmentation Fault로 이어질 수 있음

### Page Replacement

RAM에 비어 있는 Frame이 없을 때 기존 Page를 선택해 교체한다.

- FIFO: 가장 먼저 들어온 Page 교체
- LRU: 가장 오래 사용하지 않은 Page 교체
- Clock: 참조 비트를 이용해 교체 대상을 근사
- Dirty Page는 교체 전에 디스크에 기록해야 하므로 추가 비용 발생

### Thrashing

- Page Fault가 지나치게 자주 발생하는 상태
- 실제 작업보다 Page 교체에 많은 시간 사용
- 디스크 접근은 RAM보다 느려 시스템 성능이 크게 저하

### 핵심 정리

- Page Fault는 Page 부재를 운영체제에 알리는 예외
- 유효한 접근이면 Page 적재 후 명령 재실행
- 잘못된 주소나 권한 위반이면 오류로 처리
- 잦은 Page Fault는 Thrashing으로 이어질 수 있음

## JavaScript Memory Heap과 Call Stack

### Call Stack

- 현재 실행 중인 함수의 실행 정보 관리
- 함수 호출 시 Stack Frame 추가
- 함수 종료 시 Stack Frame 제거
- LIFO 방식으로 동작
- 지나치게 깊은 재귀 호출은 Stack Overflow 유발

Stack Frame에 포함되는 개념적 정보:

- 함수의 매개변수
- 지역 변수
- 현재 실행 위치
- 함수 종료 후 돌아갈 위치

### Memory Heap

- JavaScript 객체처럼 동적으로 생성되는 데이터 관리
- V8의 Garbage Collector가 JavaScript Heap 관리
- 객체가 더 이상 접근 가능하지 않으면 GC 회수 대상

### 함께 동작하는 방식

```javascript
function createUser() {
  const user = { name: "태경" };
  return user;
}

const result = createUser();
```

- `createUser()` 실행 정보: Call Stack
- `{ name: "태경" }` 객체: Memory Heap
- 함수 종료 후 `createUser()`의 Stack Frame 제거
- `result`가 객체를 계속 참조하므로 Heap 객체는 유지
- 참조가 사라지면 즉시 삭제되는 것이 아니라 GC의 회수 대상이 됨

### 주의할 점

- JavaScript 명세는 값의 실제 저장 위치를 Stack과 Heap으로 규정하지 않음
- 엔진은 레지스터, Stack, Heap 등을 상황에 따라 최적화 가능
- 원시값은 무조건 Stack, 객체는 무조건 Heap이라는 설명은 지나친 단순화

학습을 위한 개념적 구분:

- Call Stack: 함수 실행 순서와 실행 정보
- Memory Heap: 동적으로 생성된 객체의 메모리

### 핵심 정리

- 함수 호출은 Call Stack의 Stack Frame으로 관리
- 동적 객체는 JavaScript Heap에서 관리
- 객체의 생존 여부는 함수 종료가 아닌 접근 가능성에 영향받음
- 실제 메모리 배치는 JavaScript 엔진 구현과 최적화에 따라 달라질 수 있음

### 참고 자료

- [Chrome DevTools Memory Terminology](https://developer.chrome.com/docs/devtools/memory-problems/get-started)
- [V8 Garbage Collection](https://v8.dev/blog/trash-talk)

## Garbage Collection

### 개념

- 더 이상 사용할 수 없는 JavaScript 객체의 메모리를 자동으로 회수
- 객체의 도달 가능성을 기준으로 생존 여부 판단
- 개발자가 객체 메모리를 직접 해제하지 않음

대표적인 GC Root:

- 전역 객체
- 현재 Call Stack의 지역 변수와 매개변수
- 실행 환경이 유지하는 내부 참조

### 도달 가능성

```javascript
let user = { name: "태경" };
let admin = user;

user = null;
```

- `admin`이 객체를 계속 참조하므로 GC 대상이 아님
- `admin`의 참조도 제거돼 Root에서 도달할 수 없으면 GC 대상이 될 수 있음
- 참조를 제거해도 메모리가 즉시 회수되는 것은 아님

### Mark and Sweep

1. GC Root에서 시작
2. 참조를 따라가며 도달 가능한 객체 표시
3. 표시되지 않은 객체의 메모리 회수

- 객체끼리 서로 참조해도 Root에서 도달할 수 없다면 회수 가능
- 순환 참조 자체가 항상 메모리 누수를 의미하지는 않음

### V8의 세대별 GC

- Young Generation: 새로 생성된 객체
- Old Generation: 여러 번의 GC에서도 살아남은 객체
- Minor GC: Young Generation을 자주 정리
- Major GC: Old Generation을 포함한 Heap 정리
- 오래 살아남은 객체는 Old Generation으로 이동

대부분의 객체가 짧게 사용된 뒤 사라진다는 세대 가설을 이용한다. V8은 중단 시간을 줄이기 위해 병렬, 점진적, 동시 처리 방식도 활용한다.

### 주의할 점

- GC 실행 시점은 JavaScript 코드가 정확히 통제할 수 없음
- Root에서 도달 가능한 객체는 사용하지 않아도 회수되지 않음
- GC도 CPU 시간을 사용하고 일부 과정은 JavaScript 실행을 잠시 멈출 수 있음

### 핵심 정리

- 객체의 생존 기준은 사용 여부가 아니라 도달 가능성
- 순환 참조도 Root에서 끊어지면 회수 가능
- 참조 제거는 즉시 삭제가 아니라 GC 대상이 되는 것

### 참고 자료

- [V8 Garbage Collection](https://v8.dev/blog/trash-talk)
- [Chrome DevTools Memory Terminology](https://developer.chrome.com/docs/devtools/memory-problems/get-started)

## Memory Leak

### 개념

- 더 이상 사용하지 않는 데이터가 참조로 남아 GC가 회수하지 못하는 현상
- Root에서 도달 가능한 상태가 유지되면 사용 여부와 관계없이 메모리에 남음
- 메모리 사용량 증가, 응답 속도 저하, 페이지 종료로 이어질 수 있음

### 제거하지 않은 Event Listener

```javascript
useEffect(() => {
  function handleResize() {
    console.log(window.innerWidth);
  }

  window.addEventListener("resize", handleResize);

  return () => {
    window.removeEventListener("resize", handleResize);
  };
}, []);
```

- `window`가 핸들러를 계속 참조할 수 있음
- 핸들러의 Closure가 컴포넌트 상태를 유지할 수 있음
- 정리하지 않으면 재마운트마다 리스너가 중복 등록될 수 있음

### 종료하지 않은 Timer

- `setInterval()` 작업이 불필요해지면 `clearInterval()` 호출
- Timer 콜백이 참조하는 객체도 함께 유지될 수 있음

### 무제한 Cache

- `Map`에 데이터를 계속 추가하고 제거하지 않으면 메모리 사용량 증가
- 최대 크기, 만료 시간, 오래된 데이터 제거 정책 적용
- 객체 키 메타데이터에는 상황에 따라 `WeakMap` 검토

### Closure

- Closure가 외부의 큰 객체를 참조하면 함수가 존재하는 동안 객체도 유지
- 필요하지 않은 참조는 제거하거나 함수의 수명 축소

### 분리된 DOM 노드

- DOM에서 요소를 제거해도 JavaScript 참조가 남으면 메모리에 유지 가능
- 사용이 끝난 DOM 참조와 관련 Listener 정리

### 확인 방법

Chrome DevTools Memory 패널:

- Heap Snapshot
- Allocation instrumentation on timeline
- Allocation sampling
- Detached elements

객체 수와 Retained Size가 계속 증가하는지 확인하고, 객체를 유지하는 참조 경로를 추적한다.

### 핵심 정리

- Memory Leak은 메모리 할당 자체보다 불필요한 참조 유지가 핵심
- Event Listener, Timer, Cache, Closure, DOM 참조의 수명 관리
- React의 `useEffect`에서 등록한 외부 작업은 정리 함수에서 해제

### 참고 자료

- [Chrome DevTools Memory Panel](https://developer.chrome.com/docs/devtools/memory)
- [Chrome DevTools Memory Terminology](https://developer.chrome.com/docs/devtools/memory-problems/get-started)
