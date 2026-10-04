# 1주차 - 자료구조 / 시간복잡도

## 1. Big-O

### 시간복잡도

입력 크기 n 증가에 따른 연산 횟수의 증가 추세를 확인한다.

- 실제 실행 시간보다 입력 크기에 따른 연산량 변화에 초점
- 데이터가 커졌을 때 알고리즘의 성능 변화 비교
- 상수 및 낮은 차수는 제외하고 가장 영향이 큰 항을 기준으로 표현

```js
// O(1)
const first = arr[0];

// O(n)
for (const item of arr) {
  console.log(item);
}
```

### 주요 시간복잡도

| Big-O | 의미 | 예시 |
| --- | --- | --- |
| O(1) | 입력 크기와 관계없이 일정 | 배열 인덱스 접근 |
| O(log n) | 입력 증가 대비 연산량 증가가 작음 | 이진 탐색 |
| O(n) | 입력 크기에 비례 | 배열 전체 탐색 |
| O(n log n) | 선형 탐색 + 로그 연산 | 효율적인 정렬 |
| O(n²) | 입력 크기의 제곱에 비례 | 이중 반복문 |
| O(2ⁿ) | 입력 증가에 따라 연산량 급증 | 일부 완전 탐색 |

```text
O(2n)      → O(n)
O(n + 100) → O(n)
O(n² + n)  → O(n²)
```

### 확인할 내용

- O(1), O(n), O(log n), O(n²) 증가 차이
- 반복문 중첩에 따른 시간복잡도 변화
- 연속된 연산과 중첩된 연산의 차이
- 최선 / 평균 / 최악 시간복잡도

---

## 2. Array

### 구조

데이터를 순서와 인덱스를 기준으로 관리하는 자료구조

```text
index    0        1         2
       ┌───────┬────────┬────────┐
       │ apple │ banana │ orange │
       └───────┴────────┴────────┘
```

### 특징

- 인덱스를 이용한 직접 접근
- 특정 위치 접근 O(1)
- 값 검색 시 순차 탐색 O(n)
- 중간/앞쪽 삽입·삭제 시 기존 요소 이동 발생

### 시간복잡도

| 연산 | 시간복잡도 |
| --- | --- |
| 인덱스 접근 | O(1) |
| 검색 | O(n) |
| 끝 삽입 | 보통 O(1) |
| 끝 삭제 | O(1) |
| 앞 삽입 | O(n) |
| 앞 삭제 | O(n) |

### 확인할 내용

- 인덱스 접근이 O(1)인 이유
- 앞쪽 삽입/삭제가 O(n)인 이유
- Array와 Linked List의 차이

---

## 3. Linked List

### 구조

각 노드가 다음 노드의 위치를 가지고 연결되는 구조

```text
┌───┬──────┐    ┌───┬──────┐    ┌───┬──────┐
│ 1 │ next │ →  │ 2 │ next │ →  │ 3 │ null │
└───┴──────┘    └───┴──────┘    └───┴──────┘
```

```js
const node = {
  value: 1,
  next: null,
};
```

### 특징

- 인덱스를 통한 직접 접근 불가능
- 특정 위치 접근 시 앞에서부터 순차 탐색
- 연결할 노드를 알고 있는 경우 삽입/삭제 시 링크만 변경
- Singly / Doubly Linked List 구분

### Array 비교

| 연산 | Array | Linked List |
| --- | --- | --- |
| 접근 | O(1) | O(n) |
| 검색 | O(n) | O(n) |
| 앞 삽입/삭제 | O(n) | O(1) |
| 특정 노드 뒤 삽입 | 요소 이동 필요 | 연결 변경 |

---

## 4. Stack

### 구조

LIFO (Last In First Out) — 마지막으로 들어온 데이터부터 제거

```text
┌───┐
│ 3 │ ← pop
├───┤
│ 2 │
├───┤
│ 1 │
└───┘
```

### 주요 연산

- push : 데이터 추가
- pop : 최상단 데이터 제거
- peek/top : 최상단 데이터 확인

```js
const stack = [];

stack.push(1);
stack.push(2);
stack.push(3);

stack.pop(); // 3
```

push, pop → 보통 O(1)

### 활용

- 함수 호출 스택
- Undo / Redo
- 괄호 검사
- DFS

---

## 5. Queue

### 구조

FIFO (First In First Out) — 먼저 들어온 데이터부터 제거

```text
enqueue → [3] [2] [1] → dequeue
```

### 주요 연산

- enqueue : 데이터 추가
- dequeue : 가장 먼저 들어온 데이터 제거

### JavaScript Array 사용 시 주의

```js
const queue = [];

queue.push(1);
queue.push(2);

queue.shift();
```

`shift()` → O(n). 첫 번째 요소 제거 후 나머지 요소의 인덱스 이동이 발생한다.

```js
const queue = [];
let head = 0;

queue.push(1);
queue.push(2);
queue.push(3);

queue[head++]; // 1
queue[head++]; // 2
```

인덱스를 이용한 Queue 처리 → dequeue를 O(1) 수준으로 처리 가능

### 활용

- BFS
- 작업 대기열
- 순차적인 이벤트 처리

---

## 6. Hash Table

### 구조

Key를 Hash Function에 전달해 데이터를 저장할 위치를 결정한다.

```text
Key
 ↓
Hash Function
 ↓
Index
 ↓
Value
```

### 평균 시간복잡도

| 연산 | 평균 |
| --- | --- |
| 삽입 | O(1) |
| 조회 | O(1) |
| 삭제 | O(1) |

### Hash Collision

서로 다른 Key가 동일한 위치로 계산되는 상황

```text
"Kim" → 3
"Lee" → 3
```

### 충돌 처리

- Chaining
- Open Addressing

### Frontend 연결

- Object
- Map
- Set

---

## 7. Tree

### 구조

부모-자식 관계를 이용한 계층형 자료구조

```text
        A
       / \
      B   C
     / \
    D   E
```

### 기본 용어

- Root
- Parent
- Child
- Leaf
- Depth
- Height

### Binary Tree

하나의 노드가 최대 2개의 자식 노드를 가진다.

### Binary Search Tree

왼쪽 < 부모 < 오른쪽

- 균형이 잡힌 경우 검색 → 평균 O(log n)
- 한쪽으로 치우친 경우 → 최악 O(n)

### Frontend 연결

DOM 구조

```text
body
 └── div
      ├── h1
      └── p
```

컴포넌트 트리, DOM 탐색 구조와 연결해서 이해

---

## 8. Heap

### 구조

최댓값 또는 최솟값을 빠르게 찾기 위한 완전 이진 트리 기반 자료구조

### Max Heap

부모 노드 ≥ 자식 노드

```text
        10
       /  \
      7    8
     / \
    3   5
```

루트 → 최댓값

### Min Heap

부모 노드 ≤ 자식 노드

```text
        1
       / \
      3   2
     / \
    7   5
```

루트 → 최솟값

### 시간복잡도

| 연산 | 시간복잡도 |
| --- | --- |
| 최댓값/최솟값 확인 | O(1) |
| 삽입 | O(log n) |
| 삭제 | O(log n) |

### 확인할 내용

- Heap과 BST 차이
- Min Heap / Max Heap
- Priority Queue와의 관계

---

# Frontend 심화

## 9. JS Array

JavaScript에서 가장 자주 사용하는 순차 데이터 구조

```js
const users = [
  { id: 1, name: "Kim" },
  { id: 2, name: "Lee" },
];
```

### 주요 연산 시간복잡도

| 연산 | 시간복잡도 |
| --- | --- |
| arr[i] | O(1) |
| push() | 보통 O(1) |
| pop() | O(1) |
| shift() | O(n) |
| unshift() | O(n) |
| find() | O(n) |
| findIndex() | O(n) |
| includes() | O(n) |
| indexOf() | O(n) |
| map() | O(n) |
| filter() | O(n) |
| reduce() | O(n) |
| slice() | O(n) |
| splice() | O(n) |
| sort() | 보통 O(n log n) |

### 중점

단일 메서드 암기보다 메서드 조합에 따른 시간복잡도 계산이 중요하다.

```js
users.map(user => {
  return selectedIds.includes(user.id);
});
```

`map`의 각 요소마다 `includes`가 실행된다.

```text
map       → O(n)
includes  → O(m)

전체      → O(n × m)
```

두 배열 크기가 비슷한 경우 사실상 O(n²) 수준.

---

## 10. Object

문자열 또는 Symbol 기반 Key-Value 데이터 관리

```js
const user = {
  id: 1,
  name: "Kim",
};

user.name;
user["name"];
```

### 확인할 내용

- Property 접근
- Property 추가/삭제
- Object.keys(), Object.values(), Object.entries()
- Prototype
- Object와 Hash Table의 관계

---

## 11. Map

Key-Value 데이터 관리에 특화된 자료구조

```js
const users = new Map();

users.set(1, "Kim");
users.set(2, "Lee");

users.get(1);
users.has(2);
users.delete(1);
```

### Object와 비교

| | Object | Map |
| --- | --- | --- |
| Key | String / Symbol | 모든 값 |
| 조회 | obj[key] | map.get() |
| 추가 | obj[key] = value | map.set() |
| 확인 | Object.hasOwn() | map.has() |
| 삭제 | delete | map.delete() |
| 크기 | 별도 계산 | map.size |
| 반복 | 별도 API 활용 | iterable |

### 중점

- Object와 Map 선택 기준
- 동적 Key-Value 데이터 관리
- get, set, has, delete

---

## 12. Set

중복되지 않는 값 관리

```js
const ids = new Set();

ids.add(1);
ids.add(2);
ids.add(2);

// Set { 1, 2 }
```

### 배열 중복 제거

```js
const arr = [1, 1, 2, 2, 3];

const unique = [...new Set(arr)];
```

### includes()와 has()

```js
arr.includes(value); // O(n)
set.has(value);      // 평균 O(1)
```

반복적인 존재 여부 확인이 필요한 경우 Set을 활용할 수 있다.

```js
const selectedIds = new Set([1, 5, 10, 20]);

users.filter(user => selectedIds.has(user.id));
```

---

## 13. 불변성

기존 데이터를 직접 수정하는 대신 새로운 데이터를 생성한다.

### Object

```js
// 직접 변경
user.age = 21;

// 새로운 객체 생성
const updatedUser = {
  ...user,
  age: 21,
};
```

### Array 추가

```js
const newUsers = [...users, newUser];
```

### Array 삭제

```js
const newUsers = users.filter(
  user => user.id !== targetId
);
```

### Array 수정

```js
const newUsers = users.map(user =>
  user.id === targetId
    ? { ...user, name: "Lee" }
    : user
);
```

### React와 불변성

상태 변경 시 참조값 변경을 통해 변경을 감지한다.

```js
const a = { count: 1 };
const b = a;

a === b; // true
```

새로운 객체 생성:

```js
const b = {
  ...a,
  count: 2,
};

a === b; // false
```

### Shallow Copy

Spread 사용 → 얕은 복사

```js
const user = {
  name: "Kim",
  profile: {
    age: 20,
  },
};

const copied = { ...user };

user === copied;                 // false
user.profile === copied.profile; // true
```

중첩 객체 변경:

```js
const updated = {
  ...user,
  profile: {
    ...user.profile,
    age: 21,
  },
};
```

### 중점

- Mutable / Immutable 차이
- 참조 동일성
- Shallow Copy / Deep Copy 차이
- React 상태 업데이트와 불변성
- Spread 사용 시 중첩 객체 처리
