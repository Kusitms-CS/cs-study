# 1주차 - 자료구조 / 시간복잡도

## 1. Big-O

### Big-O란?

입력 크기 N이 커질 때 알고리즘의 시간 또는 공간 사용량이 얼마나 증가하는지 표현하는 방법.

중요한 점은 실제 실행 시간 자체가 아니라, 입력 크기가 증가함에 따라 실행량이 **어떤 비율로 증가하는가**를 보는 것.

예를 들어 반복문이 n번 실행되므로: **O(N)**.

```java
for (int i = 0; i < n; i++) {
    System.out.println(i);
}
```



### 대표적인 시간복잡도

| 복잡도 | 의미 | 대표 사례 |
|---|---|---|
| O(1) | 입력 크기와 무관 | 배열 인덱스 접근, HashMap 평균 조회 |
| O(log N) | 탐색 범위를 계속 절반으로 줄임 | 이진 탐색 |
| O(N) | 데이터 전체 탐색 | 배열 순회 |
| O(N log N) | 효율적인 정렬 | Merge Sort, Heap Sort |
| O(N²) | 이중 반복 | 단순 비교 정렬 |
| O(2^N) | 모든 부분집합 탐색 | 백트래킹 |
| O(N!) | 모든 순열 탐색 | 순열 완전탐색 |

성능 차이는 N이 커질수록 극단적으로 벌어짐 / 예를 들어 N = 1,000,000이면 대략

```
O(1)       → 1
O(log N)   → 약 20
O(N)       → 1,000,000
O(N log N) → 약 20,000,000
O(N²)      → 1,000,000,000,000
```

### 상수 제거

전체 연산은 `2N`이지만 Big-O에서는 **O(N)**.

또한 `N² + N + 100` 이라면 가장 빠르게 증가하는 N²만 남겨 **O(N²)**.

```java
for (...) {} // N
for (...) {} // N
```


### 평균 / 최악 / 최선

예를 들어 배열에서 값을 찾는 경우 

- 첫 번째 위치에 있으면: O(1)
- 마지막에 있거나 없으면: O(N)
- 일반적으로 알고리즘에서는 **최악의 시간복잡도**를 많이 이야기함.

```java
for (int i = 0; i < arr.length; i++) {
    if (arr[i] == target) {
        return i;
    }
}
```

## 2. Array

Array는 데이터를 연속된 메모리 공간에 저장하는 자료구조

```java
int[] arr = new int[5];
```

개념

```
index   0   1   2   3   4
       ┌───┬───┬───┬───┬───┐
       │10 │20 │30 │40 │50 │
       └───┴───┴───┴───┴───┘
```

### 인덱스 접근이 O(1)인 이유

배열 시작 주소를 알고 있는 경우 

```
주소 = 시작주소 + index × 데이터크기로 원하는 위치를 바로 계산 가능
그래서 arr[3]은 전체 배열을 순회할 필요 없이 바로 접근 가능 → **O(1)**
```




### 삽입 / 삭제

중간에 값을 삽입하는 경우

기존
```
[A][B][C][D]
```

B와 C 사이에 X 삽입
```
[A][B][X][C][D]
```

뒤의 값들을 모두 한 칸씩 이동해야 함

- 삽입 O(N)
- 삭제 O(N)

### Array 복잡도

| 연산 | 복잡도 |
|---|---|
| index 조회 | O(1) |
| 값 검색 | O(N) |
| 마지막 삽입 | 경우에 따라 O(1) |
| 중간 삽입 | O(N) |
| 중간 삭제 | O(N) |

## 3. LinkedList

LinkedList는 Array처럼 연속된 메모리 공간을 사용하지 않음 /  각 Node가 다음 Node를 가리키는 형태

```
[A | next] → [B | next] → [C | next] → null
```

Java의 LinkedList는 정확히는 **Doubly Linked List**

```
null ← A ⇄ B ⇄ C → null
```

각 Node가 아래처럼 이전 노드와 다음 노드를 모두 알고 있음

```java
class Node {
    Node prev;
    Object item;
    Node next;
}
```



### 조회

Array라면 `arr[100];` 으로 바로 접근 가능

하지만 LinkedList의 경우 아래와 깉이 하나씩 따라가야 함 / 따라서 **O(N)**.

```
head
 ↓
A → B → C → D → ...
```



### 삽입 / 삭제

노드를 이미 알고 있다고 가정하면

```
A → B → C
```

여기에서 B를 제거할 때

```
A → C
```

연결만 변경하면 됨 → **O(1)**

다만 삭제하려는 노드를 찾는 과정은 O(N)일 수 있음.

그래서 흔히 "LinkedList의 삽입/삭제는 무조건 O(1)"이라고 외우기  X

정확하게는: **삽입/삭제 위치를 이미 알고 있다면 O(1), 위치 탐색까지 필요하면 O(N)**

### Array vs LinkedList

| | Array | LinkedList |
|---|---|---|
| 메모리 | 연속 | 비연속 |
| index 조회 | O(1) | O(N) |
| 검색 | O(N) | O(N) |
| 중간 삽입/삭제 | O(N) | 노드 확보 시 O(1) |
| Cache locality | 좋음 | 나쁨 |
| 추가 포인터 | 없음 | 필요 |


## 4. Stack

Stack은 **LIFO - Last In First Out** 구조. 마지막에 들어간 값이 가장 먼저 나옴

```
push A
[A]

push B
[B]
[A]

push C
[C]
[B]
[A]

pop → C
```

### 주요 연산

| 연산 | 설명 |
|---|---|
| push | 데이터 삽입 |
| pop | 데이터 제거 |
| peek | 최상단 조회 |

모두 일반적으로 **O(1)**

### 대표 사용 사례

**함수 호출 Stack**

```
main()
 ↓
foo()
 ↓
bar()
```

호출 순서는 `main → foo → bar`, 반환 순서는 `bar → foo → main` — 즉 LIFO

**DFS**

```java
Stack<Node> stack;
```

또는 재귀 함수가 내부적으로 Call Stack을 사용

**Undo**

```
작업1
작업2
작업3

Undo:
작업3 취소
작업2 취소
작업1 취소
```



### Java에서는 Stack보다 Deque

옛날

```java
Stack<Integer> stack = new Stack<>();
```

요즘은 deque 권장 => Stack은 오래된 Vector 기반 클래스이며 불필요한 동기화 등의 문제가 있기 때문
```java
Deque<Integer> stack = new ArrayDeque<>();

stack.push(1);
stack.pop();
stack.peek();
```



## 5. Queue

Queue는 **FIFO — First In First Out** / 먼저 들어온 데이터가 먼저 나감

```
A → B → C

poll()
A 제거

B → C
```

### 주요 연산

| 연산 | 설명 |
|---|---|
| offer() | 삽입 |
| poll() | 제거 |
| peek() | 조회 |

일반적인 구현에서 **O(1)**

### 대표 사용 사례

**BFS**

```java
Queue<Node> queue = new ArrayDeque<>();
```

BFS는 가까운 노드부터 탐색하기 때문에 Queue를 사용

**작업 처리**

백엔드 시스템에서도 굉장히 자주 등장

```
Client
   ↓
Queue
   ↓
Worker
```

예: 주문 요청, 메일 발송, Push 알림, 이미지 처리, 로그 처리

AWS SQS, RabbitMQ, Kafka 등의 메시징 시스템도 넓게 보면 Queue와 관련된 아이디어를 가지고 있음.

## 6. Hash Table

Java의 `HashMap`, `HashSet`, `ConcurrentHashMap` 등이 모두 관련되어 있음

핵심 아이디어는 **Key를 Hash Function에 넣어 저장 위치를 계산**하는 것

```
Key
 ↓
hash()
 ↓
Hash Value
 ↓
Bucket
```

예시

```
"pooreum"
   ↓
 hash()
   ↓
9348293
   ↓
bucket index = 5
```

Bucket

```
0
1
2
3
4
5 → pooreum
6
7
```

따라서 데이터를 처음부터 탐색하지 않아도 됨.
- 조회 O(1)
- 삽입 O(1)
- 삭제 O(1)

## 7. Hash Collision

Hash 값이 다르더라도 같은 Bucket에 들어갈 수 있음

```
hash(A) → bucket 3
hash(B) → bucket 3
```

이것을 **Hash Collision** 이라고 함

충돌을 처리하는 대표 방식으로 **Separate Chaining**

```
Bucket 3
   ↓
[A] → [B] → [C]
```

한 Bucket에 여러 Entry를 연결하여 Java HashMap이 사용하는 방식

## 8. Tree

Tree는 계층형 자료구조

```
        A
      /   \
     B     C
    / \   / \
   D   E F   G
```

주요 용어

| 용어 | 설명 |
|---|---|
| Root | A |
| Parent | B의 Parent = A |
| Child | A의 Child = B, C |
| Leaf | D, E, F, G |
| Depth | Root에서 떨어진 거리 |
| Height | 가장 깊은 Leaf까지 거리 |

## 9. Binary Tree

각 Node가 최대 2개의 Child를 가지는 Tree / **Binary Tree라고 해서 자동으로 정렬되어 있는 것은 아님**

```
       A
      / \
     B   C
```



## 10. Binary Search Tree

Binary Search Tree(BST)는 추가 규칙을 가지고 있음

```
왼쪽 < 부모 < 오른쪽
```

예시

```
        8
       / \
      4   12
     / \  / \
    2  6 10 14
```

10을 찾는 경우

```
10 > 8
→ 오른쪽

10 < 12
→ 왼쪽

10 발견
```

균형 잡힌 경우: **O(log N)**

### BST 최악의 경우

데이터가 `1 2 3 4 5` 순서대로 들어가면:

```
1
 \
  2
   \
    3
     \
      4
       \
        5
```

사실상 LinkedList가 됨. 따라서 탐색이 **O(N)** 까지 떨어짐

이를 해결하기 위해 **AVL Tree**, **Red-Black Tree** 같은 Self-Balancing Tree가 사용됨

## 11. Tree 순회

- **Preorder**: Root → Left → Right
- **Inorder**: Left → Root → Right — BST를 Inorder 순회하면 정렬된 결과를 얻을 수 있음
- **Postorder**: Left → Right → Root

## 12. Heap

Heap은 우선순위를 빠르게 처리하기 위한 완전 이진 트리. 대표적으로 **Max Heap**, **Min Heap** 이 있음

### Max Heap

부모가 자식보다 큼

```
        10
       /  \
      8    7
     / \
    3   5
```

따라서 Root에는 항상 최대값이 있음 → 최대값 조회 O(1)

### Min Heap

부모가 자식보다 작음

```
        1
       / \
      3   2
     / \
    7   5
```

Root에는 최소값이 있음

### 삽입

새 노드를 마지막에 넣고 부모와 비교하며 올라감 (**Bubble Up / Heapify Up**)

Tree 높이가 log N 이므로 **O(log N)**

### 삭제

Root를 제거한 뒤 마지막 노드를 Root로 가져옴. 그 다음 자식과 비교하며 내려감 (**Bubble Down / Heapify Down**)

역시 **O(log N)**

### Heap 복잡도

| 연산 | 시간복잡도 |
|---|---|
| 최소/최대 조회 | O(1) |
| 삽입 | O(log N) |
| 삭제 | O(log N) |
| 특정 값 검색 | O(N) |

### Java PriorityQueue

Java에서는 Heap을 직접 구현하지않고 PriorityQueue를 사용. 기본적으로 **Min Heap**

```java
PriorityQueue<Integer> pq = new PriorityQueue<>();
```



```java
pq.offer(5);
pq.offer(1);
pq.offer(3);

pq.poll(); // 1
```

Max Heap은 아래 코드처럼 사용 가능
```java
PriorityQueue<Integer> pq =
        new PriorityQueue<>(Comparator.reverseOrder());
```



---

# Backend 심화

## 13. Java Collection Framework

Java Collection Framework는 여러 자료구조를 표준화하여 제공하는 API

```
                 Iterable
                    │
                Collection
             ┌──────┼──────┐
            List    Set    Queue
             │       │       │
         ArrayList HashSet   Deque
         LinkedList TreeSet   │
                         ArrayDeque


Map
 │
 ├─ HashMap
 ├─ LinkedHashMap
 ├─ TreeMap
 └─ ConcurrentHashMap
 Map은 Collection의 하위 인터페이스가 아님
```

 

## 14. List

순서가 있고 중복을 허용

```java
List<String> list = new ArrayList<>();
```

대표 구현체: `ArrayList`, `LinkedList`

### ArrayList

내부적으로 `Object[]` 배열 사용.  따라서 index 조회가 **O(1)**

```java
transient Object[] elementData;
```



### ArrayList 크기가 부족한 경우

예를 들어 배열 용량이 10인데 11번째 데이터를 넣으면 기존 배열을 계속 사용할 수 없음. 그래서 더 큰 배열을 만든 후 데이터를 복사

이 작업 자체는 **O(N)** / 그런데 매번 발생하는 것은 아니므로 평균적으로 마지막 삽입은 **Amortized O(1)** 이라고 표현

```
[기존 배열]
   ↓
더 큰 배열 생성
   ↓
기존 값 복사
```



## 15. Set

중복을 허용하지 않는 자료구조. 대표적으로 `HashSet`, `TreeSet`, `LinkedHashSet`

### HashSet

```java
Set<String> set = new HashSet<>();
```

평균: 삽입 O(1), 조회 O(1), 삭제 O(1)

 **HashSet 내부적으로 HashMap을 사용**



```java
private transient HashMap<E, Object> map;
```

Set의 값은 HashMap의 Key로 저장

```
HashSet.add("A")
≈
HashMap.put("A", dummyValue)
```

## 16. Map

Key-Value 자료구조 / Key는 중복될 수 없지만 Value는 중복 가능

```java
Map<String, User> users = new HashMap<>();
```

```
"kim" → User
"lee" → User
"park" → User
```



## 17. HashMap 내부 구조

```
HashMap
  │
  └── Node<K,V>[] table
             │
             ├── bucket 0
             ├── bucket 1
             ├── bucket 2 → Node → Node
             └── ...
```

Entry 개념

```java
class Node<K,V> {
    int hash;
    K key;
    V value;
    Node<K,V> next;
}
```

## 18. HashMap put()

예시

```java
map.put("pooreum", user);
```


```
1. key.hashCode()
        ↓
2. Hash 보정
        ↓
3. Bucket index 계산
        ↓
4. 해당 Bucket 탐색
        ↓
5. Key가 없으면 추가
   Key가 있으면 Value 교체
```

## 19. hashCode()와 equals()

Key를 비교할 때 기본적인 흐름

```
hashCode 비교
      ↓
equals 비교
```

예시

```java
class User {
    private Long id;
}
```

두 객체가 논리적으로 동일한 Key여야 한다면 `equals()`와 `hashCode()` 계약을 올바르게 구현해야 함

핵심 규칙: **`equals()`가 true인 두 객체는 반드시 같은 `hashCode()`를 반환해야 함**

반대는 성립하지 않음

```
hashCode가 같음
≠
equals가 반드시 true => Hash Collision이 존재할 수 있음
```



## 20. HashMap Bucket 계산

단순하게 생각하면 `index = hash % bucketSize` 라고 생각할 수 있음

하지만 Java HashMap의 table 크기는 2의 거듭제곱으로 유지되기 때문에 bit 연산을 사용

```java
index = (n - 1) & hash;
```

예를 들어 `n = 16` 이면 `n - 1 = 15 = 1111₂`

이 비트 연산으로 0 ~ 15 범위의 Bucket을 결정 가능.

## 21. HashMap Collision

다음처럼 충돌하는 경우

```
bucket 5

Node A
   ↓
Node B
   ↓
Node C
```

초기에는 LinkedList 형태로 저장 /  그렇다면 최악에는 **O(N)** 이 될 수 있음

## 22. Java 8 이후 Treeification

Java 8부터 한 Bucket에 너무 많은 Entry가 몰리면 LinkedList를 Red-Black Tree로 변환

```
LinkedList
A → B → C → D → ...
↓
Red-Black Tree
```

이렇게 되면 탐색 복잡도가 **O(N) → O(log N)** 으로 개선

대표적인 Treeify 기준은 bucket의 노드 수가 일정 수준 이상일 때이며, table 자체도 충분히 커야 함

## 23. HashMap Load Factor

HashMap은 Bucket이 너무 많이 차면 성능이 떨어짐. 그래서 `capacity`, `load factor`, `threshold` 개념을 사용

기본 Load Factor는 **0.75**

예를 들어 capacity가 16이면 `16 × 0.75 = 12`, 약 12개를 넘어가면서 resize가 발생할 수 있음

## 24. HashMap Resize

Resize가 발생하면 더 큰 Table을 만듦

```
16 → 32 → 64 → 128
```

그리고 기존 Entry를 새로운 Bucket 구조에 맞게 재배치. 따라서 Resize는 상대적으로 비싼 작업

그래서 데이터 크기를 미리 아는 경우 =>  아래처럼 적절한 초기 크기를 고려할 수도 있음

```java
new HashMap<>(expectedSize);
```



## 25. HashMap 시간복잡도

평균적인 경우

```
put    O(1)
get    O(1)
remove O(1)
```

충돌이 심한 최악의 상황에서는 Tree 구조 등의 영향을 받아 달라질 수 있음

포인트 =>  **HashMap O(1)은 절대적인 것이 아니라 평균 시간복잡도**

## 26. HashMap은 Thread-Safe한가?

`HashMap`은 Thread-Safe하지 않음.

예시

```
Thread A ─┐
          ├→ HashMap
Thread B ─┘
```

두 Thread가 동시에 변경하면 데이터 일관성 문제가 발생할 수 있음

그래서 멀티스레드 환경에서는 상황에 따라 `ConcurrentHashMap`을 사용

## 27. ConcurrentHashMap

ConcurrentHashMap은 **여러 Thread가 동시에 Map에 접근할 수 있도록 설계된 Thread-Safe Map**.

```java
ConcurrentHashMap<String, User> map =
        new ConcurrentHashMap<>();
```

## 28. Hashtable과 차이

과거에는 `Hashtable`을 사용했음. Hashtable은 주요 메서드에 동기화가 걸려 있어서 개념적으로는 아래처럼 병목이 생길 수 있음

```
Thread A
 ↓
Map 전체 Lock
 ↓
작업
 ↓
Unlock
```


ConcurrentHashMap은 더 세밀하게 동시성을 제어. 그래서 여러 Thread가 서로 다른 영역을 수정 가능

```
Thread A → bucket A
Thread B → bucket B

동시 작업 가능
```

## 29. Java 7 ConcurrentHashMap

Java 7에서는 Segment Lock 구조 사용 / Map 전체를 Lock하지 않고 Segment 단위로 Lock

```
ConcurrentHashMap

Segment 1
 ├ bucket
 ├ bucket

Segment 2
 ├ bucket
 ├ bucket

Segment 3
 ...
```



## 30. Java 8 이후 ConcurrentHashMap

Java 8부터 Segment 구조가 제거됨. 주요 아이디어는 **CAS + synchronized**

충돌이 없는 경우에는 CAS 같은 기법을 이용하고, 특정 Bucket에서 충돌하면서 수정해야 하는 경우에는 해당 영역에 대해 동기화를 수행

즉, **Map 전체 Lock X, 필요한 Bucket 수준 동기화**에 가까움

## 31. CAS란?

CAS는 **Compare And Swap**

```
현재 값이 내가 예상한 값과 같은가?

YES
→ 새로운 값으로 변경

NO
→ 실패 후 재시도
```

예시

```
Expected = 10
Current  = 10
→ 11로 변경 성공
```

예외

```
Expected = 10
Current  = 11
→ 다른 Thread가 이미 변경함
→ 변경 실패
```

Lock을 무조건 잡는 대신 CPU의 원자적 연산을 이용해 동시성 처리 가능

## 32. ConcurrentHashMap이 null을 허용하지 않는 이유


HashMap은 `map.put(null, "value");` 같은 것이 가능

하지만 ConcurrentHashMap에서는 `map.put(null, "value");` 가 허용되지 않음

왜냐하면 멀티스레드 환경에서는 `map.get(key) == null` 일 때 이것이

1. Key가 존재하지 않는 것인지
2. Key는 있는데 Value가 null인 것인지 구별하기 어렵기 때문

동시에 다른 Thread가 데이터를 변경할 수도 있기 때문에 애매함을 제거하고자 null을 금지

## 33. HashMap vs ConcurrentHashMap

| | HashMap | ConcurrentHashMap |
|---|---|---|
| Thread-Safe | X | O |
| null Key | 가능 | 불가능 |
| null Value | 가능 | 불가능 |
| 동시 수정 | 위험 | 지원 |
| 일반 단일 Thread | 적합 | 필요 이상일 수 있음 |
| 멀티 Thread 공유 Map | 부적합 | 적합 |

## 34. 자료구조 전체 시간복잡도 정리

| 자료구조 | 접근 | 검색 | 삽입 | 삭제 |
|---|---|---|---|---|
| Array | O(1) | O(N) | O(N) | O(N) |
| LinkedList | O(N) | O(N) | O(1)* | O(1)* |
| Stack | O(N) | O(N) | O(1) | O(1) |
| Queue | O(N) | O(N) | O(1) | O(1) |
| Hash Table | - | 평균 O(1) | 평균 O(1) | 평균 O(1) |
| BST | 평균 O(log N) | 평균 O(log N) | 평균 O(log N) | 평균 O(log N) |
| Balanced BST | O(log N) | O(log N) | O(log N) | O(log N) |
| Heap | - | O(N) | O(log N) | O(log N) |



## 35. Java Collection 선택 기준

```
순서가 필요한가?
│
├─ YES
│   │
│   ├─ index 접근이 중요한가?
│   │      └─ ArrayList
│   │
│   └─ Queue / Stack인가?
│          └─ ArrayDeque
│
└─ NO
    │
    ├─ Key-Value인가?
    │      │
    │      ├─ 일반 → HashMap
    │      ├─ 정렬 → TreeMap
    │      └─ 멀티스레드 → ConcurrentHashMap
    │
    └─ 중복 제거가 필요한가?
           │
           ├─ 일반 → HashSet
           └─ 정렬 → TreeSet
```

## 36. 백엔드 개발자 관점



**ArrayList → 내부 배열 → Resize → Amortized O(1)**

```
add()
  ↓
공간 있음 → O(1)

공간 없음
  ↓
배열 확장
  ↓
복사 O(N)

하지만 평균적으로 Amortized O(1)
```

**HashMap → Hash → Bucket → Collision → LinkedList → Red-Black Tree**

```
key
 ↓
hashCode()
 ↓
hash
 ↓
bucket
 ↓
Node
 ↓ collision
LinkedList
 ↓ collision 증가
Red-Black Tree
```

**equals/hashCode → HashMap/HashSet 동작과 연결**

```
equals가 같다면 hashCode도 같아야 함
```

**HashMap → Thread Safe X → ConcurrentHashMap**

```
HashMap
→ 단일 Thread

ConcurrentHashMap
→ 공유 데이터 + Multi Thread
```

**Heap → PriorityQueue**

```
우선순위 기반 작업
→ Heap
→ PriorityQueue
```

**Queue → 비동기 백엔드 처리**

```
API 요청
 ↓
Message Queue
 ↓
Worker
 ↓
DB / 외부 API
```

## 37. 면접 예상 질문??
1. Array와 LinkedList의 차이는?
2. Array의 index 조회가 왜 O(1)인가?
3. LinkedList의 삽입이 정말 항상 O(1)인가?
4. Stack과 Queue의 차이는?
5. DFS와 BFS는 각각 어떤 자료구조를 사용하나?
6. Hash Table이 평균 O(1)인 이유는?
7. Hash Collision이 발생하면 어떻게 처리하나?
8. hashCode()와 equals()의 관계는?
9. HashMap에서 hashCode()만 비교하지 않고 equals()도 사용하는 이유는?
10. HashMap 내부 구조 설명.
11. Java 8에서 HashMap의 충돌 처리 방식이 어떻게 개선됐나?
12. Load Factor란?
13. HashMap Resize는 언제 발생하나?
14. HashMap과 ConcurrentHashMap의 차이는?
15. ConcurrentHashMap은 어떻게 Thread Safety를 보장하나?
16. ConcurrentHashMap이 null을 허용하지 않는 이유는?
17. Binary Tree와 BST는 어떻게 다른가?
18. BST의 탐색이 항상 O(log N)인가?
19. Red-Black Tree를 사용하는 이유는?
20. Heap과 BST의 차이는?
21. PriorityQueue는 내부적으로 어떤 자료구조를 사용하나?
22. ArrayList의 add()가 왜 Amortized O(1)인가?
23. HashSet은 내부적으로 어떻게 구현되어 있나?
24. Map은 왜 Collection을 상속하지 않나?

