# 1주차 - 자료구조 & 시간복잡도

# 1. 시간복잡도 (Big-O)

## 1.1 시간복잡도란?

시간복잡도(Time Complexity)는 입력 데이터의 크기 `N`이 증가할 때 알고리즘의 실행 시간이 얼마나 증가하는지를 나타내는 척도

실제 실행 시간을 초 단위로 측정하는 개념이 아니라, 입력 크기가 증가함에 따라 필요한 연산 횟수가 어떤 비율로 증가하는지를 표현

예를 들어 배열에서 특정 값을 찾는 코드

```cpp
for (int i = 0; i < n; i++) {
    if (arr[i] == target) {
        return i;
    }
}
```

최악의 경우 배열의 모든 원소 확인 필요

입력 크기가 `N`이라면 최대 `N`번의 연산 수행

시간복잡도는 `O(N)`

---

## 1.2 Big-O 표기법

Big-O는 입력 크기가 커졌을 때 알고리즘의 연산량이 증가하는 정도를 표현하는 표기법

| Big-O | 의미 | 대표적인 예 |
|---|---|---|
| `O(1)` | 입력 크기와 관계없이 일정 | 배열 인덱스 접근 |
| `O(log N)` | 입력 증가 대비 연산량 증가가 작음 | Binary Search |
| `O(N)` | 입력 크기에 비례 | 배열 전체 탐색 |
| `O(N log N)` | 효율적인 정렬에서 주로 등장 | Merge Sort |
| `O(N²)` | 입력 크기의 제곱만큼 증가 | 이중 반복문 |
| `O(2^N)` | 입력마다 경우의 수가 2배씩 증가 | 부분집합 탐색 |
| `O(N!)` | 모든 순열 탐색 | 순열 완전탐색 |

일반적인 성능 순서

```text
O(1)
<
O(log N)
<
O(N)
<
O(N log N)
<
O(N²)
<
O(2^N)
<
O(N!)
```

입력 크기가 커질수록 뒤쪽 시간복잡도의 연산량이 매우 빠르게 증가

---

## 1.3 코드로 시간복잡도 판단하기

### O(1)

입력 크기와 관계없이 일정한 횟수의 연산만 수행하는 경우

```cpp
int value = arr[5];
```

배열 크기가 10이든 1,000,000이든 특정 인덱스에 접근하는 연산 횟수는 동일

따라서 `O(1)`

---

### O(N)

입력 데이터를 한 번씩 순회하는 경우

```cpp
for (int i = 0; i < n; i++) {
    cout << arr[i];
}
```

입력 크기가 `N`이면 반복문도 `N`번 실행

따라서 `O(N)`

---

### O(N²)

반복문 안에 또 다른 반복문이 존재하는 대표적인 경우

```cpp
for (int i = 0; i < n; i++) {
    for (int j = 0; j < n; j++) {
        cout << i << j;
    }
}
```

외부 반복문 `N`번 실행

내부 반복문도 각 반복마다 `N`번 실행

총 연산 횟수 약 `N × N`

따라서 `O(N²)`

---

### O(log N)

반복할 때마다 문제의 크기가 일정 비율로 감소하는 경우

```cpp
while (n > 1) {
    n /= 2;
}
```

입력 크기의 변화

```text
N
N / 2
N / 4
N / 8
...
1
```

입력값이 절반씩 감소

약 `log₂N`번의 연산 필요

따라서 `O(log N)`

Binary Search가 대표적인 `O(log N)` 알고리즘

---

# 2. Array

## 2.1 Array란?

Array는 같은 타입의 데이터를 연속된 메모리 공간에 저장하는 자료구조

```text
Index   0    1    2    3    4

       ┌────┬────┬────┬────┬────┐
       │ 10 │ 20 │ 30 │ 40 │ 50 │
       └────┴────┴────┴────┴────┘
```

각 원소가 메모리에 연속적으로 저장

인덱스를 이용하여 원하는 원소에 접근

```cpp
arr[3];
```

배열의 시작 주소와 데이터 하나의 크기를 알고 있기 때문에 원하는 원소의 메모리 위치를 바로 계산 가능

인덱스를 통한 접근은 `O(1)`

---

## 2.2 Array의 주요 연산

### 접근

인덱스를 알고 있는 경우 원하는 데이터에 바로 접근

```cpp
arr[3];
```

시간복잡도 `O(1)`

---

### 탐색

특정 값이 어느 위치에 존재하는지 모르는 경우 처음부터 순차적으로 확인

```cpp
for (int i = 0; i < n; i++) {
    if (arr[i] == target) {
        return i;
    }
}
```

최악의 경우 모든 데이터 확인 필요

시간복잡도 `O(N)`

---

### 삽입

배열의 중간에 데이터를 삽입하는 경우 뒤에 존재하는 원소들을 한 칸씩 이동

```text
10 20 30 40

20과 30 사이에 25 삽입

10 20 __ 30 40
      ↑
      25
```

최악의 경우 거의 모든 원소 이동 필요

시간복잡도 `O(N)`

---

### 삭제

배열 중간의 데이터를 삭제하면 빈 공간을 제거하기 위해 뒤쪽 원소들을 앞으로 이동

최악의 경우 `O(N)`

---

## 2.3 Array 시간복잡도

| 연산 | 시간복잡도 |
|---|---|
| 접근 | `O(1)` |
| 탐색 | `O(N)` |
| 삽입 | `O(N)` |
| 삭제 | `O(N)` |

배열 마지막 위치에서 삽입하거나 삭제하는 경우 상황에 따라 `O(1)` 처리 가능

---

## 2.4 Array 장점

- 인덱스를 이용한 접근 속도 빠름
- 연속된 메모리 공간 사용
- 구조가 단순
- 순차적으로 데이터를 읽을 때 캐시 효율이 좋음
- 특정 위치의 데이터를 자주 조회하는 상황에 적합

---

## 2.5 Array 단점

- 중간 삽입과 삭제 비용이 큼
- 일반적인 고정 크기 배열은 크기를 미리 결정
- 연속된 메모리 공간 필요
- 크기가 고정된 배열의 경우 크기 변경 어려움

---

# 3. LinkedList

## 3.1 LinkedList란?

LinkedList는 데이터를 Node 단위로 저장하고 각 Node가 다음 Node의 위치를 참조하는 자료구조

```text
┌──────┬──────┐
│  10  │ next │
└──────┴───┬──┘
           ↓
┌──────┬──────┐
│  20  │ next │
└──────┴───┬──┘
           ↓
┌──────┬──────┐
│  30  │ null │
└──────┴──────┘
```

Array와 달리 각각의 Node가 메모리상 연속적으로 존재할 필요 없음

Node가 일반적으로 가지는 정보

```text
Data + 다음 Node의 주소
```

---

## 3.2 LinkedList 종류

### Singly Linked List

각 Node가 다음 Node만 참조하는 구조

```text
A → B → C → D
```

한 방향으로만 이동 가능

---

### Doubly Linked List

각 Node가 이전 Node와 다음 Node를 모두 참조하는 구조

```text
A ⇄ B ⇄ C ⇄ D
```

양방향 이동 가능

Node마다 이전 주소와 다음 주소를 모두 저장하므로 추가 메모리 필요

---

### Circular Linked List

마지막 Node가 다시 첫 번째 Node를 참조하는 구조

```text
A → B → C → D
↑           ↓
└───────────┘
```

마지막과 처음이 연결된 형태

---

## 3.3 LinkedList 접근과 탐색

LinkedList에는 Array와 같은 인덱스 기반 직접 접근 없음

세 번째 Node를 찾으려면 Head부터 순서대로 이동

```text
Head
 ↓
A → B → C
```

접근과 탐색의 시간복잡도 `O(N)`

---

## 3.4 LinkedList 삽입

삽입할 위치의 Node를 이미 알고 있다면 참조 관계만 변경

기존 상태

```text
A → B
```

A와 B 사이에 C 삽입

```text
A → C → B
```

Node 자체를 찾는 과정이 없다면 삽입 자체의 시간복잡도 `O(1)`

---

## 3.5 LinkedList 삭제

삭제할 Node와 필요한 인접 Node를 이미 알고 있다면 참조 관계만 변경

```text
A → B → C
```

B 삭제

```text
A → C
```

삭제 자체는 `O(1)`

단, 삭제할 Node를 먼저 탐색해야 한다면 탐색 비용 `O(N)` 추가

---

## 3.6 LinkedList 시간복잡도

| 연산 | 시간복잡도 |
|---|---|
| 접근 | `O(N)` |
| 탐색 | `O(N)` |
| 삽입 | `O(1)*` |
| 삭제 | `O(1)*` |

`*` 삽입 또는 삭제에 필요한 Node를 이미 알고 있는 경우

Node를 먼저 찾아야 하는 상황이라면 전체 연산 `O(N)`

---

## 3.7 Array vs LinkedList

| 구분 | Array | LinkedList |
|---|---|---|
| 메모리 구조 | 연속적 | 비연속적 |
| 접근 | `O(1)` | `O(N)` |
| 탐색 | `O(N)` | `O(N)` |
| 삽입/삭제 | `O(N)` | `O(1)*` |
| 추가 메모리 | 적음 | Pointer / Reference 필요 |
| Cache 효율 | 좋음 | 상대적으로 낮음 |

Array는 조회가 많은 상황에서 유리

LinkedList는 특정 위치에서 삽입과 삭제가 빈번한 구조에 적합

---

# 4. Stack

## 4.1 Stack이란?

Stack은 **LIFO(Last In First Out)** 구조를 가지는 자료구조

가장 마지막에 들어온 데이터가 가장 먼저 나오는 구조

```text
push(10)

│ 10 │
└────┘

push(20)

│ 20 │ ← Top
├────┤
│ 10 │
└────┘

push(30)

│ 30 │ ← Top
├────┤
│ 20 │
├────┤
│ 10 │
└────┘
```

이 상태에서 데이터를 제거하면 가장 마지막에 삽입된 `30`부터 제거

---

## 4.2 Stack 주요 연산

### Push

Stack의 가장 위에 새로운 데이터를 추가하는 연산

```text
push()
```

시간복잡도 `O(1)`

---

### Pop

Stack의 가장 위 데이터를 제거하는 연산

```text
pop()
```

시간복잡도 `O(1)`

---

### Top / Peek

Stack에서 가장 위에 존재하는 데이터를 확인하는 연산

데이터를 제거하지 않고 값만 확인

시간복잡도 `O(1)`

---

## 4.3 Stack 시간복잡도

| 연산 | 시간복잡도 |
|---|---|
| Push | `O(1)` |
| Pop | `O(1)` |
| Top / Peek | `O(1)` |

---

## 4.4 Stack 활용

Stack은 가장 최근에 수행한 작업을 다시 처리해야 하는 상황에 적합

대표적인 활용 사례

- DFS
- 함수 호출
- 재귀 호출
- 괄호 검사
- Undo / Redo
- 브라우저 뒤로 가기

---

## 4.5 함수 호출과 Stack

프로그램에서 함수가 호출될 때 Call Stack 사용

```text
main()
 ↓
functionA()
 ↓
functionB()
```

`functionB()` 종료 후 `functionA()`로 복귀

`functionA()` 종료 후 `main()`으로 복귀

가장 마지막에 호출된 함수가 가장 먼저 종료되므로 Stack의 LIFO 구조와 동일

---

# 5. Queue

## 5.1 Queue란?

Queue는 **FIFO(First In First Out)** 구조를 가지는 자료구조

가장 먼저 들어온 데이터가 가장 먼저 나오는 구조

```text
OUT ← [10][20][30] ← IN
        ↑
      Front
```

---

## 5.2 Queue 주요 연산

### Enqueue

Queue의 뒤쪽에 새로운 데이터를 삽입하는 연산

```text
enqueue()
```

시간복잡도 `O(1)`

---

### Dequeue

Queue의 가장 앞쪽 데이터를 제거하는 연산

```text
dequeue()
```

시간복잡도 `O(1)`

---

### Front / Peek

Queue의 가장 앞 데이터를 확인하는 연산

데이터를 제거하지 않고 값만 확인

시간복잡도 `O(1)`

---

## 5.3 Queue 시간복잡도

| 연산 | 시간복잡도 |
|---|---|
| Enqueue | `O(1)` |
| Dequeue | `O(1)` |
| Front / Peek | `O(1)` |

---

## 5.4 Queue와 BFS

BFS는 가까운 Node부터 순서대로 탐색하는 알고리즘

다음과 같은 Tree

```text
        A
       / \
      B   C
     / \
    D   E
```

BFS 탐색 순서

```text
A → B → C → D → E
```

먼저 발견한 Node를 먼저 탐색해야 하므로 FIFO 구조인 Queue 사용

---

## 5.5 Queue 활용

Queue는 먼저 들어온 작업을 먼저 처리해야 하는 상황에 적합

대표적인 활용

- BFS
- 요청 처리
- 작업 Queue
- 프린터 Queue
- 메시지 Queue
- 작업 스케줄링

---

## 5.6 Stack vs Queue

| 구분 | Stack | Queue |
|---|---|---|
| 구조 | LIFO | FIFO |
| 삽입 | Push | Enqueue |
| 삭제 | Pop | Dequeue |
| 대표 알고리즘 | DFS | BFS |
| 핵심 특징 | 마지막 데이터 우선 처리 | 먼저 들어온 데이터 우선 처리 |

---

# 6. Hash Table

## 6.1 Hash Table이란?

Hash Table은 Key를 Hash Function에 입력하여 데이터가 저장될 위치를 계산하는 자료구조

```text
Key
 ↓
Hash Function
 ↓
Hash Value
 ↓
Bucket
 ↓
Value
```

예시

```text
"apple"
   ↓
Hash Function
   ↓
3
```

`apple`에 해당하는 데이터를 3번 Bucket에 저장

배열처럼 처음부터 데이터를 하나씩 탐색하지 않고 Hash를 이용해 저장 위치 계산

평균적으로 빠른 검색 가능

---

## 6.2 Hash Function

Hash Function은 입력받은 Key를 특정 Hash 값으로 변환하는 함수

```text
Key → Hash Function → Hash Value
```

좋은 Hash Function은 Key를 Bucket 전체에 최대한 고르게 분산

특정 Bucket에 데이터가 집중되면 Hash Table 성능 저하

---

## 6.3 Bucket

Bucket은 Hash Table 내부에서 실제 Entry가 저장되는 공간

```text
Bucket 0
Bucket 1
Bucket 2
Bucket 3 → apple
Bucket 4
```

Hash 값을 이용해 어떤 Bucket에 데이터를 저장하거나 탐색할지 결정

---

## 6.4 Hash Collision

서로 다른 Key가 같은 Bucket을 가리키는 상황

```text
apple  → hash → 3
banana → hash → 3
```

Hash Function은 제한된 개수의 Bucket으로 수많은 Key를 변환하므로 Collision 완전 제거 불가능

충돌을 처리하는 별도 방법 필요

---

## 6.5 Separate Chaining

같은 Bucket에 여러 Entry를 연결하여 저장하는 방식

```text
Bucket 3

apple → banana → mango
```

하나의 Bucket 안에서 여러 데이터를 별도의 구조로 관리

Java의 `HashMap`도 기본적으로 이와 유사한 방식으로 Collision 처리

---

## 6.6 Open Addressing

Collision 발생 시 같은 Bucket에 저장하지 않고 다른 비어 있는 Bucket을 찾아 저장하는 방식

대표적인 방식

- Linear Probing
- Quadratic Probing
- Double Hashing

---

## 6.7 Hash Table 시간복잡도

| 연산 | 평균 | 최악 |
|---|---:|---:|
| 탐색 | `O(1)` | `O(N)` |
| 삽입 | `O(1)` | `O(N)` |
| 삭제 | `O(1)` | `O(N)` |

Hash가 적절하게 분산되어 있다면 원하는 Bucket에 바로 접근 가능

평균 시간복잡도 `O(1)`

모든 데이터가 하나의 Bucket에 몰리는 극단적인 상황에서는 하나씩 데이터를 비교해야 하므로 `O(N)` 가능

---

# 7. Tree

## 7.1 Tree란?

Tree는 Node들이 부모와 자식 관계를 가지는 계층형 자료구조

```text
        A
       / \
      B   C
     / \
    D   E
```

파일 시스템, 조직도, DOM 구조처럼 계층 관계를 표현하는 데 적합

---

## 7.2 Tree 기본 용어

### Root

Tree에서 가장 위에 존재하는 Node

```text
        A ← Root
       / \
      B   C
```

### Parent

특정 Node의 바로 위 Node

B와 C의 Parent는 A

### Child

특정 Node 바로 아래에 연결된 Node

A의 Child는 B와 C

### Sibling

같은 Parent를 가지는 Node

B와 C는 Sibling 관계

### Leaf

Child를 가지지 않는 Node

```text
        A
       / \
      B   C
     / \
    D   E
```

위 구조에서 C, D, E가 Leaf

### Depth

Root에서 특정 Node까지 내려가는 거리

Root의 Depth를 0으로 두면 B와 C는 Depth 1, D와 E는 Depth 2

### Height

특정 Node에서 가장 멀리 있는 Leaf까지의 거리

Tree 전체 Height는 Root를 기준으로 가장 깊은 Leaf까지의 거리

### Subtree

특정 Node를 Root로 가지는 Tree 내부의 작은 Tree

---

## 7.3 Binary Tree

Binary Tree는 하나의 Node가 최대 두 개의 Child를 가지는 Tree

```text
        10
       /  \
      5    20
```

각 Node는 최대 Left Child와 Right Child를 하나씩 가짐

---

## 7.4 Binary Search Tree

Binary Search Tree는 Binary Tree에 정렬 조건을 추가한 자료구조

기본 조건

```text
왼쪽 Subtree < 현재 Node < 오른쪽 Subtree
```

예시

```text
        8
       / \
      3   10
     / \
    1   6
```

8보다 작은 값은 왼쪽 Subtree에 저장

8보다 큰 값은 오른쪽 Subtree에 저장

---

## 7.5 BST 탐색

현재 Node와 찾으려는 값을 비교하여 탐색 방향 결정

```text
target < node → 왼쪽 탐색
target > node → 오른쪽 탐색
```

Tree가 균형 잡혀 있다면 한 번 비교할 때마다 탐색 범위 감소

평균 시간복잡도 `O(log N)`

---

## 7.6 BST가 O(N)이 되는 경우

BST가 항상 균형 잡힌 형태를 유지하는 것은 아님

다음 순서로 데이터 삽입

```text
1, 2, 3, 4
```

생성 가능한 구조

```text
1
 \
  2
   \
    3
     \
      4
```

Tree가 한쪽으로 치우치면서 사실상 LinkedList와 비슷한 구조

이 경우 최악 시간복잡도

| 연산 | 평균 | 최악 |
|---|---:|---:|
| 탐색 | `O(log N)` | `O(N)` |
| 삽입 | `O(log N)` | `O(N)` |
| 삭제 | `O(log N)` | `O(N)` |

---

## 7.7 Tree Traversal

Tree의 모든 Node를 방문하는 방법

기준 Tree

```text
        A
       / \
      B   C
     / \
    D   E
```

### Preorder

현재 Node를 먼저 방문한 뒤 왼쪽과 오른쪽 Subtree 탐색

```text
Root → Left → Right

A B D E C
```

### Inorder

왼쪽 Subtree를 먼저 방문한 후 현재 Node와 오른쪽 Subtree 탐색

```text
Left → Root → Right

D B E A C
```

BST를 Inorder 방식으로 탐색하면 데이터를 정렬된 순서로 확인 가능

### Postorder

왼쪽과 오른쪽 Subtree를 먼저 탐색한 후 현재 Node 방문

```text
Left → Right → Root

D E B C A
```

### Level Order

Tree의 같은 Depth에 존재하는 Node를 순서대로 방문

```text
A B C D E
```

Queue를 이용한 BFS 방식으로 구현

---

# 8. Heap

## 8.1 Heap이란?

Heap은 Complete Binary Tree를 기반으로 특정 우선순위 조건을 만족하도록 구성된 자료구조

대표적으로 Max Heap과 Min Heap 존재

Heap에서는 전체 데이터가 정렬되어 있는 것은 아님

부모와 자식 사이의 우선순위만 보장

---

## 8.2 Complete Binary Tree

Complete Binary Tree는 마지막 Level을 제외한 모든 Level이 채워져 있고, 마지막 Level의 Node가 왼쪽부터 순서대로 채워지는 Binary Tree

```text
        1
       / \
      2   3
     / \
    4   5
```

Heap은 이러한 구조 덕분에 Array를 이용한 효율적인 구현 가능

---

## 8.3 Max Heap

부모 Node의 값이 자식 Node보다 크거나 같은 Heap

```text
        10
       /  \
      8    7
     / \
    3   5
```

Root에는 항상 가장 큰 값 위치

```text
Parent >= Child
```

---

## 8.4 Min Heap

부모 Node의 값이 자식 Node보다 작거나 같은 Heap

```text
        1
       / \
      3   2
     / \
    7   5
```

Root에는 항상 가장 작은 값 위치

```text
Parent <= Child
```

---

## 8.5 Heap 삽입

새로운 데이터를 Complete Binary Tree의 마지막 위치에 삽입

이후 부모 Node와 비교하면서 Heap 조건을 만족할 때까지 위치 교환

```text
새로운 Node 삽입
      ↓
부모와 비교
      ↓
Heap 조건 위반
      ↓
Swap
      ↓
조건 만족까지 반복
```

Tree의 높이는 `log N` 수준

삽입 시간복잡도 `O(log N)`

---

## 8.6 Heap 삭제

Heap에서는 일반적으로 Root 제거

Max Heap에서는 최댓값 제거

Min Heap에서는 최솟값 제거

Root 삭제 후 마지막 Node를 Root로 이동

이후 자식 Node와 비교하면서 Heap 조건을 만족하도록 위치 조정

```text
Root 삭제
   ↓
마지막 Node를 Root로 이동
   ↓
Child와 비교
   ↓
Swap
   ↓
Heap 조건 복구
```

시간복잡도 `O(log N)`

---

## 8.7 Heap 시간복잡도

| 연산 | 시간복잡도 |
|---|---|
| 최댓값 / 최솟값 조회 | `O(1)` |
| 삽입 | `O(log N)` |
| Root 삭제 | `O(log N)` |
| 특정 값 탐색 | `O(N)` |

Heap은 전체 정렬을 보장하지 않으므로 특정 값을 찾는 탐색에는 부적합

---

## 8.8 Heap과 Priority Queue

Priority Queue는 삽입 순서가 아니라 우선순위가 높은 데이터부터 꺼내는 Queue

Heap은 Priority Queue를 효율적으로 구현하는 대표적인 자료구조

Java 사용 예시

```java
PriorityQueue<Integer> pq = new PriorityQueue<>();
```

Java의 기본 `PriorityQueue`는 Min Heap 기반

가장 작은 값이 가장 높은 우선순위

---

## 8.9 Heap 활용

Heap은 가장 크거나 작은 값을 반복적으로 가져와야 하는 상황에서 유용

대표적인 활용

- Priority Queue
- 작업 스케줄링
- Top K 문제
- Dijkstra 알고리즘
- 가장 작은 값 또는 큰 값을 반복적으로 추출하는 문제

---

# 9. Java Collection Framework

## 9.1 Java Collection Framework란?

Java Collection Framework는 여러 자료구조를 Java에서 쉽게 사용할 수 있도록 제공하는 표준 API

대표적인 구조

```text
Collection
├── List
│   ├── ArrayList
│   └── LinkedList
│
├── Set
│   ├── HashSet
│   └── TreeSet
│
└── Queue
    ├── LinkedList
    └── PriorityQueue

Map
├── HashMap
├── TreeMap
└── ConcurrentHashMap
```

`Map`은 Key와 Value 형태의 구조를 가지며 `Collection` 인터페이스를 직접 상속하지 않음

---

## 9.2 List

List는 데이터의 순서가 존재하며 중복 데이터를 허용하는 자료구조

```text
[10, 20, 20, 30]
```

같은 값이 여러 번 들어갈 수 있음

삽입된 순서 유지

---

## 9.3 ArrayList

ArrayList는 내부적으로 동적 배열을 사용하는 List 구현체

```java
List<Integer> list = new ArrayList<>();
```

특징

- 인덱스를 통한 접근이 빠름
- 데이터가 연속적인 형태로 저장
- 중간 삽입과 삭제 시 데이터 이동 발생
- Java에서 일반적인 List 구현체로 많이 사용
- 내부 배열의 크기가 부족하면 더 큰 배열로 확장

---

## 9.4 LinkedList

Java의 `LinkedList`는 연결 리스트 기반 List 구현체

```java
List<Integer> list = new LinkedList<>();
```

특징

- 인덱스 접근이 ArrayList보다 느림
- Node 간 연결 구조 사용
- 특정 Node 위치에서 삽입과 삭제 자체는 빠름
- `List`와 `Deque` 인터페이스 모두 구현
- Queue 또는 Deque 형태로도 사용 가능

---

## 9.5 Set

Set은 중복 데이터를 허용하지 않는 자료구조

```text
10
20
30
```

같은 값을 여러 번 추가해도 하나만 유지

---

## 9.6 HashSet

HashSet은 Hash Table을 기반으로 구현된 Set

```java
Set<Integer> set = new HashSet<>();
```

평균 시간복잡도

```text
add       O(1)
remove    O(1)
contains  O(1)
```

중복 여부를 빠르게 확인해야 하는 상황에서 많이 사용

---

## 9.7 TreeSet

TreeSet은 Tree 기반 Set 구현체

```java
Set<Integer> set = new TreeSet<>();
```

데이터를 정렬된 상태로 유지

주요 연산의 시간복잡도 `O(log N)`

```text
add       O(log N)
remove    O(log N)
contains  O(log N)
```

정렬이 필요한 경우 HashSet보다 TreeSet이 적합

---

## 9.8 Map

Map은 데이터를 `Key - Value` 형태로 저장하는 자료구조

```text
Key      Value

apple → 100
banana → 200
```

Key 중복 불가

하나의 Key에 하나의 Value 연결

---

## 9.9 HashMap

HashMap은 Hash Table 기반의 Map 구현체

```java
Map<String, Integer> map = new HashMap<>();
```

평균적인 주요 연산

```text
put       O(1)
get       O(1)
remove    O(1)
```

Java에서 가장 일반적으로 사용하는 Map 구현체 중 하나

---

## 9.10 TreeMap

TreeMap은 Tree 기반 Map 구현체

```java
Map<String, Integer> map = new TreeMap<>();
```

Key를 정렬된 상태로 관리

주요 연산의 시간복잡도 `O(log N)`

HashMap보다 탐색 속도는 느리지만 Key의 정렬이 필요한 상황에 적합

---

## 9.11 PriorityQueue

PriorityQueue는 우선순위를 기준으로 데이터를 꺼내는 Queue

```java
PriorityQueue<Integer> pq = new PriorityQueue<>();
```

Java에서는 기본적으로 Min Heap 기반

가장 작은 값이 먼저 반환

```java
pq.add(30);
pq.add(10);
pq.add(20);

pq.poll(); // 10
```

---

# 10. HashMap 내부 구조

## 10.1 HashMap이란?

`HashMap`은 Java에서 Key와 Value 형태의 데이터를 저장하는 대표적인 자료구조

```java
Map<String, Integer> map = new HashMap<>();

map.put("apple", 100);
map.put("banana", 200);
```

내부적으로 Hash Table을 기반으로 동작

---

## 10.2 HashMap의 기본 구조

HashMap은 내부적으로 Bucket 배열 보유

각 Key의 Hash 값을 계산하여 어느 Bucket에 Entry를 저장할지 결정

```text
Key
 ↓
hashCode()
 ↓
Hash 계산
 ↓
Bucket Index 계산
 ↓
Entry 저장
```

Entry에 포함되는 주요 정보

```text
Key
Value
Hash
다음 Entry에 대한 참조
```

---

## 10.3 put()

예시

```java
map.put("apple", 100);
```

개념적인 내부 과정

```text
"apple"
    ↓
hashCode()
    ↓
Hash 값 계산
    ↓
Bucket Index 결정
    ↓
해당 Bucket 확인
    ↓
같은 Key가 존재하는지 확인
    ↓
저장 또는 기존 Value 갱신
```

같은 Key가 존재하지 않으면 새로운 Entry 저장

같은 Key가 이미 존재하면 기존 Value 변경

---

## 10.4 get()

예시

```java
map.get("apple");
```

Key의 Hash를 다시 계산하여 해당 Bucket 위치 탐색

```text
Key
 ↓
Hash 계산
 ↓
Bucket 접근
 ↓
Key 비교
 ↓
Value 반환
```

모든 Entry를 처음부터 순차적으로 탐색할 필요 없음

평균적으로 `O(1)` 탐색 가능

---

## 10.5 hashCode()

`hashCode()`는 Java 객체를 정수형 Hash 값으로 표현하는 메서드

HashMap에서는 Key가 어느 Bucket에 들어갈지 결정하는 데 사용

```java
key.hashCode();
```

같은 객체 또는 논리적으로 동일한 객체라면 일관된 Hash 값을 가지는 것이 중요

---

## 10.6 equals()

`equals()`는 두 객체가 논리적으로 동일한 객체인지 비교하는 메서드

HashMap에서 같은 Bucket에 여러 Key 후보가 존재할 경우 실제 같은 Key인지 확인하는 데 사용

```text
hashCode()
   ↓
같은 Bucket 발견
   ↓
equals()
   ↓
실제로 같은 Key인지 확인
```

---

## 10.7 hashCode()와 equals() 관계

두 객체가 `equals()` 기준으로 동일하다면 동일한 `hashCode()` 반환 필요

```text
a.equals(b) == true
```

이라면

```text
a.hashCode() == b.hashCode()
```

조건 만족 필요

반대로 동일한 `hashCode()`를 가진다고 해서 반드시 두 객체가 같은 것은 아님

```text
같은 hashCode()
≠
반드시 같은 객체
```

Hash Collision 발생 가능

---

## 10.8 HashMap의 Collision 처리

서로 다른 Key가 같은 Bucket에 배정 가능

```text
apple
   ↓
Bucket 3

banana
   ↓
Bucket 3
```

Java의 HashMap은 같은 Bucket에 여러 Entry를 연결하여 충돌 처리

Java 8 이후에는 하나의 Bucket에 많은 Entry가 집중되고 일정 조건을 만족하면 Linked List 형태를 Tree 구조로 변환

이를 Treeification이라고 표현

```text
충돌이 적음

Bucket
 ↓
Entry → Entry → Entry
```

충돌이 많고 조건을 만족하는 경우

```text
Bucket
 ↓
Tree
```

Tree 구조를 사용하여 충돌이 심한 상황에서 탐색 성능 저하 완화

---

## 10.9 Load Factor

HashMap은 Bucket이 지나치게 가득 차면 Hash Collision 증가 가능

이를 조절하기 위해 Load Factor 사용

Java HashMap의 기본 Load Factor는 `0.75`

```text
Load Factor = 0.75
```

현재 Capacity가 16이라면

```text
16 × 0.75 = 12
```

Threshold는 12

Entry 수가 Threshold를 초과하면 Resize 발생

---

## 10.10 Resize

HashMap의 Entry 수가 Threshold를 초과하면 내부 Bucket 배열 크기 증가

```text
Capacity 16
    ↓
Capacity 32
```

Bucket 배열 크기가 변경되므로 기존 Entry들도 새로운 구조에 맞게 재배치

Resize는 상대적으로 비용이 큰 작업

대량의 데이터를 저장할 것이 명확한 경우 적절한 초기 Capacity 설정으로 불필요한 Resize 횟수 감소 가능

---

## 10.11 HashMap이 평균 O(1)인 이유

일반적인 List에서는 원하는 값을 찾기 위해 처음부터 탐색 필요

```text
A → B → C → D → E
```

이 경우 `O(N)` 탐색 발생 가능

HashMap은 Key의 Hash를 이용하여 데이터가 존재할 가능성이 높은 Bucket을 바로 계산

```text
Key
 ↓
Hash
 ↓
Bucket
```

Hash 값이 적절하게 분산되어 있다는 가정하에 Bucket 내부 Entry 수가 적게 유지

탐색, 삽입, 삭제의 평균 시간복잡도 `O(1)`

---

# 11. ConcurrentHashMap

## 11.1 HashMap의 동시성 문제

일반적인 `HashMap`은 Thread Safe하지 않음

```java
Map<String, Integer> map = new HashMap<>();
```

여러 Thread가 하나의 HashMap을 동시에 수정할 경우 데이터가 예상하지 못한 상태로 변경될 가능성 존재

예시

```text
Thread A ─┐
          ├─ HashMap 수정
Thread B ─┘
```

여러 Thread가 공유하는 자료구조에서는 동기화 문제 발생 가능

---

## 11.2 ConcurrentHashMap이란?

`ConcurrentHashMap`은 여러 Thread가 하나의 Map을 동시에 사용하는 환경에서 안전하게 사용할 수 있도록 설계된 Map 구현체

```java
Map<String, Integer> map = new ConcurrentHashMap<>();
```

Thread Safe한 자료구조

동시에 여러 Thread가 접근할 수 있도록 설계

---

## 11.3 Hashtable

Java에는 기존부터 Thread Safe한 Map 구현체인 `Hashtable` 존재

```java
Map<String, Integer> map = new Hashtable<>();
```

Hashtable은 주요 메서드에 `synchronized` 사용

하나의 Thread가 Map에 접근하는 동안 다른 Thread의 접근이 제한될 수 있어 동시성이 낮은 편

현재는 Legacy 성격이 강함

새로운 코드에서는 일반적으로 `ConcurrentHashMap` 사용 비중이 높음

---

## 11.4 ConcurrentHashMap의 동시성 처리

ConcurrentHashMap은 Map 전체를 항상 하나의 Lock으로 막는 방식이 아니라 필요한 범위에 대해서만 동기화 수행

Java 8 이후 구현에서는 CAS(Compare-And-Set)와 `synchronized` 등을 활용

```text
Thread A → Bucket A 수정

Thread B → Bucket B 수정
```

서로 다른 영역을 수정하는 작업이라면 동시에 진행 가능

Thread Safety를 유지하면서 Hashtable보다 높은 동시성 제공

---

## 11.5 CAS

CAS는 Compare-And-Set 또는 Compare-And-Swap을 의미

현재 값이 예상했던 값과 동일한 경우에만 새로운 값으로 변경하는 원자적 연산

개념적인 흐름

```text
현재 값 == 예상 값 ?

YES → 새로운 값으로 변경
NO  → 변경 실패
```

Lock 사용을 줄이면서 여러 Thread 사이의 경쟁을 제어할 때 사용

---

## 11.6 HashMap vs Hashtable vs ConcurrentHashMap

| 구분 | HashMap | Hashtable | ConcurrentHashMap |
|---|---|---|---|
| Thread Safe | X | O | O |
| null Key | 가능 | 불가능 | 불가능 |
| null Value | 가능 | 불가능 | 불가능 |
| 동시성 | 지원하지 않음 | 낮은 편 | 높은 편 |
| 주요 용도 | 일반적인 Map | Legacy | 멀티스레드 환경 |
| 동기화 | 없음 | 메서드 단위 동기화 | 세분화된 동기화 |

---

## 11.7 ConcurrentHashMap 사용 상황

여러 Thread가 하나의 Map을 공유하면서 동시에 데이터를 읽고 수정하는 상황에서 사용

Spring Boot와 같은 서버 애플리케이션에서는 여러 요청을 여러 Thread가 동시에 처리 가능

이 과정에서 여러 Thread가 공유하는 Map이 존재한다면 일반 `HashMap` 사용에 주의 필요

```text
Thread 1 ─┐
Thread 2 ─┼─ 공유 Map 접근
Thread 3 ─┘
```

이와 같은 상황에서 `ConcurrentHashMap` 고려 가능

단, `ConcurrentHashMap` 자체가 Thread Safe하다고 해서 여러 연산을 조합한 모든 로직까지 자동으로 Thread Safe가 되는 것은 아님

---

# 12. 자료구조 시간복잡도 총정리

| 자료구조 | 접근 | 탐색 | 삽입 | 삭제 |
|---|---:|---:|---:|---:|
| Array | `O(1)` | `O(N)` | `O(N)` | `O(N)` |
| LinkedList | `O(N)` | `O(N)` | `O(1)*` | `O(1)*` |
| Stack | - | - | `O(1)` | `O(1)` |
| Queue | - | - | `O(1)` | `O(1)` |
| Hash Table | - | `O(1)` avg | `O(1)` avg | `O(1)` avg |
| BST | - | `O(log N)` avg | `O(log N)` avg | `O(log N)` avg |
| Heap | - | `O(N)` | `O(log N)` | `O(log N)` |

`*` 필요한 Node의 위치를 이미 알고 있는 경우
