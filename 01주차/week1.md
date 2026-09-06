# 1주차 - 자료구조 / 시간복잡도

## 1. Big-O

> 입력 크기가 커질수록 알고리즘의 실행 비용이 얼마나 빠르게 증가하는지 나타내는 표기법이다.

아래 코드는 `N`개의 데이터를 한 번씩 확인한다.

```java
for (int i = 0; i < n; i++) {
    System.out.println(i);
}
```

따라서 실행 횟수는 입력 크기 `N`에 비례하며 시간복잡도는 O(N)이다.

### 왜 필요한가?

실제 실행 시간만 비교하면 컴퓨터 성능이나 JVM 상태에 따라 결과가 달라질 수 있다.

```
알고리즘 A → 1ms
알고리즘 B → 2ms
```

작은 입력에서는 큰 차이가 없어 보이지만 데이터가 커지면 전혀 다른 결과가 나올 수 있다.

```
N = 1,000,000

O(1)       → 1
O(log N)   → 약 20
O(N)       → 1,000,000
O(N log N) → 약 20,000,000
O(N²)      → 1,000,000,000,000
```

따라서 Big-O에서는 현재 몇 초 걸리는가보다 N이 증가했을 때 연산량이 어떻게 증가하는가를 판단한다.

### 대표 시간복잡도

| 복잡도        | 의미                 | 대표 사례      |
| ---------- | ------------------ | ---------- |
| O(1)       | 입력 크기와 관계없는 일정한 연산 | 배열 인덱스 접근  |
| O(log N)   | 탐색 범위를 반복해서 줄임     | 이진 탐색      |
| O(N)       | 전체 데이터를 한 번 확인     | 배열 순회      |
| O(N log N) | 데이터를 나누면서 전체를 처리   | Merge Sort |
| O(N²)      | 각 데이터마다 전체를 다시 확인  | 이중 반복문     |
| O(2ⁿ)      | 경우의 수가 지수적으로 증가    | 부분집합 탐색    |
| O(N!)      | 모든 순서를 탐색          | 순열 완전탐색    |

### Big-O는 항상 worst case라는 뜻인가?

엄밀하게 말하면 그렇지 않다.

Big-O는 증가율의 상한을 표현하는 표기법이다.

다만 알고리즘 설명에서는 보통 안정적인 성능 보장을 위해 최악 시간복잡도를 Big-O로 설명하는 경우가 많다.

선형 탐색은 아래처럼 볼 수 있다.

```
Best   → O(1)
Average → O(N)
Worst  → O(N)
```

### 면접 연결 포인트

O(1)이면 항상 빠른 알고리즘인가?

- 그렇지 않다.
- Big-O에서는 상수를 제거하기 때문에
    ```
    알고리즘 A → 1,000번 연산 = O(1)
    알고리즘 B → N번 연산 = O(N)
    ```
    이라도 작은 N에서는 B가 더 빠를 수 있다.
- Big-O는 입력이 충분히 커졌을 때의 증가 추세를 비교하기 위한 개념이다.

## 2. Array

> 같은 타입의 데이터를 연속된 공간에 저장하고 인덱스로 접근하는 자료구조이다.

```java
int[] arr = {10, 20, 30, 40, 50};
```

```
index    0    1    2    3    4
       ┌────┬────┬────┬────┬────┐
       │ 10 │ 20 │ 30 │ 40 │ 50 │
       └────┴────┴────┴────┴────┘
```

### 왜 index 접근은 O(1)일까?

배열의 시작 위치와 데이터 하나의 크기를 알고 있기 때문이다.

개념적으로는 아래와 같이 바로 계산할 수 있다.

```
원하는 위치
= 배열 시작 위치 + index × 요소 크기
```

따라서 `arr[100]`을 실행한다고 해서 0번부터 99번까지 확인하지 않고, 바로 인덱스 100의 위치를 계산해서 접근하기 때문에 **O(1)**이다.

### 삽입과 삭제는 왜 O(N)일까?

```
[A][B][C][D]
```

여기서 B와 C 사이에 X를 넣으려면 C, D를 한 칸씩 뒤로 이동해야 한다.

```
[A][B][X][C][D]
```

따라서 최악의 경우 거의 모든 데이터를 이동해야 하기 때문에 O(N)이다.

삭제도 삽입과 동일하다.

### 시간복잡도

| 연산     |             시간복잡도 |
| ------ | ----------------: |
| 인덱스 접근 |              O(1) |
| 값 검색   |              O(N) |
| 중간 삽입  |              O(N) |
| 중간 삭제  |              O(N) |
| 끝 삽입   | 정적 배열에서는 직접 확장 불가 |

### Array와 ArrayList

자바에서 배열은 크기가 고정된다.

```java
int[] arr = new int[10];
```

반면 ArrayList는 내부적으로 배열을 사용하지만 공간이 부족하면 더 큰 배열을 만들고 기존 데이터를 복사한다.

```java
List<Integer> list = new ArrayList<>();
```

따라서 ArrayList.add()는 대부분 O(1)이지만 배열 확장이 발생할 때는 O(N)이 된다.

전체 연산을 평균내면 Amortized O(1)이라고 한다.

### Cache Locality

배열은 데이터가 인접해서 저장되기 때문에 CPU 캐시 관점에서도 유리하다.

CPU는 데이터를 하나만 가져오기보다 주변 데이터까지 함께 캐시 라인에 가져온다.

```
Memory

[A][B][C][D][E]
 ↑
A를 읽으면서 주변 데이터도 Cache에 올라올 가능성이 높음
```

따라서 순차 접근에서는 Array가 LinkedList보다 실제 성능이 좋은 경우가 많다.

### 면접 연결 포인트

삽입과 삭제가 많으면 LinkedList가 항상 Array보다 좋은가?

- 그렇지 않다.
- LinkedList도 삽입 위치를 먼저 찾아야 한다면 O(N)이 필요하고, 배열은 캐시 지역성이 좋다.
- 따라서 단순히 삽입/삭제가 많다고 연결 리스트라고 결정하면 안 되며, 실제 접근 패턴과 데이터 크기를 함께 봐야 한다.

## 3. LinkedList

> 각 노드가 다른 노드를 참조하면서 연결되는 자료구조이다.

```
[A | next] → [B | next] → [C | next] → null
```

배열처럼 데이터가 반드시 연속된 공간에 있을 필요가 없다.

### 종류

- Singly Linked List
    
    - 다음 노드만 알고 있다.
        ```
        A → B → C → null
        ```

- Doubly Linked List

    - 앞뒤 노드를 모두 알고 있다.
        ```
        null ← A ⇄ B ⇄ C → null
        ```
    - 자바의 LinkedList는 Doubly Linked List 구조이다.

### 왜 index 접근은 O(N)인가?

LinkedList는 특정 노드의 주소를 바로 계산할 수 없다.

list.get(100)을 수행하기 위해서는 연결을 따라가야 한다.

```
head
 ↓
A → B → C → D → ... → 100번째
```

따라서 O(N)이다.

### 삽입/삭제는 O(1)인가?

```
A ⇄ B ⇄ C
```

B를 제거한다면,

```
A ⇄ C
```

참조만 변경하면 되기 때문에 O(1)이다.

하지만 B를 먼저 찾아야 한다면 아래와 같이 전체는 O(N)이 된다.

```
노드 탐색 O(N)
+
연결 변경 O(1)
```

따라서 정확한 표현으로 정리하면 삽입/삭제할 노드를 이미 알고 있을 때 O(1)이다.

### Array와의 비교

|                | Array | LinkedList |
| -------------- | ----: | ---------: |
| index 접근       |  O(1) |       O(N) |
| 값 검색           |  O(N) |       O(N) |
| 위치를 아는 삽입/삭제   |  O(N) |       O(1) |
| Cache Locality |    좋음 |   상대적으로 나쁨 |
| 추가 메모리         |    적음 |   참조 저장 필요 |

## 4. Stack

> 가장 마지막에 들어온 데이터가 가장 먼저 나오는 LIFO 구조이다.

```
push A

A

push B

B
A

push C

C
B
A

pop()
→ C
```

### 주요 연산

| 연산   | 의미     | 시간복잡도 |
| ---- | ------ | ----: |
| push | 삽입     |  O(1) |
| pop  | 제거     |  O(1) |
| peek | 최상단 조회 |  O(1) |

### 함수 호출

스택은 함수 호출에서 대표적으로 사용된다.

```
main()
 ↓
foo()
 ↓
bar()
```

호출은 main → foo → bar 순서이지만, 반환은 bar → foo → main 순서인 스택 구조이다.

따라서 재귀 호출이 너무 깊어지면 StackOverflowError가 발생할 수 있다.

### DFS와 스택

DFS는 한 방향으로 최대한 깊게 탐색하기 때문에 스택 구조와 잘 맞는다.

```
Deque<Node> stack = new ArrayDeque<>();
```

위와 같이 사용할 수도 있고, 재귀 호출을 이용해 JVM Call Stack을 사용할 수도 있다.

### 자바에서는 왜 스택을 잘 안 쓸까?

```java
Deque<Integer> stack = new ArrayDeque<>();
```

Stack보다 Deque를 사용하는 것이 일반적으로 권장된다.

스택은 오래된 벡터 기반 클래스이기 때문에 불필요한 동기화 비용과 레거시 API 설계 문제가 있기 때문이다.

## 5. Queue

> 먼저 들어온 데이터가 먼저 나오는 FIFO 구조이다.

```
A → B → C

poll()

B → C
```

### 주요 연산

| 연산    | 의미       |
| ----- | -------- |
| offer | 뒤에 삽입    |
| poll  | 앞에서 제거   |
| peek  | 앞 데이터 조회 |

일반적으로 O(1)이다.

### BFS와 Queue

BFS는 현재 노드와 가까운 노드부터 탐색한다.

```java
Queue<Node> queue = new ArrayDeque<>();

A
↓
B C
↓
D E F
```

먼저 발견된 노드를 먼저 탐색해야 하기 때문에 큐를 사용한다.

### 배열로 큐를 만들면?

단순히 배열의 첫 번째 값을 삭제한다면, 삭제 후 나머지 요소를 이동해야 하기 때문에 O(N)이 된다.

그래서 배열 기반 큐는 일반적으로 Circular Queue 방식으로 구현한다.

앞과 뒤 위치를 가리키는 인덱스만 이동시킨다.

## 6. Hash Table

> Key를 해시 함수에 넣어 저장할 위치를 계산하고, 해당 위치에 데이터를 저장하는 자료구조이다.

```
Key
 ↓
Hash Function
 ↓
Hash Value
 ↓
Bucket
 ↓
Data
```

키를 기반으로 저장할 위치를 결정하기 때문에 데이터를 처음부터 순회하지 않고 원하는 위치에 빠르게 접근할 수 있다.

### 왜 평균 O(1)일까?

배열에서 특정 값을 찾으려면 일반적으로 처음부터 순회해야 하기 때문에 검색은 O(N)이다.

반면 해시 테이블은 키를 이용해 저장 위치를 계산한다.

따라서 데이터가 적절히 분산되어 있다면 원하는 버킷으로 바로 접근할 수 있다.

| 연산 | 평균 시간복잡도 |
| -- | -------: |
| 조회 |     O(1) |
| 삽입 |     O(1) |
| 삭제 |     O(1) |

다만 중요한 점은 항상 O(1)이 아니라 평균 O(1)이라는 것이다.

### 해시 함수와 버킷

해시 함수는 키를 입력받아 해시 값을 만드는 함수이다.

```
Key → Hash Function → Hash Value
```

해시 값을 이용해 실제 데이터를 저장할 버킷을 결정한다.

단순화하면 다음과 같이 생각할 수 있다.

```
bucket index = hash % bucketSize
```

예를 들어 아래와 같다면 버킷 3에 데이터를 저장한다.

```
hash = 27
bucketSize = 8

27 % 8 = 3
```

```
0
1
2
3 → ("kim", User)
4
5
6
7
```

### 해시 충돌

서로 다른 키가 같은 버킷으로 계산될 수 있다.

```
"A" → bucket 3
"B" → bucket 3
```

이를 해시 충돌이라고 한다.

버킷의 개수는 제한되어 있기 때문에 충돌 자체를 완전히 없앨 수는 없다.

따라서 해시 테이블에서는 충돌을 어떻게 처리하는가가 중요하다.

- Separate Chaining

    - 같은 버킷에 들어온 데이터를 연결해서 저장한다.
        ```
        bucket 3
        ↓
        [A] → [B] → [C]
        ```
    - 자바의 HashMap이 사용하는 방식도 이 계열이다.

- Open Addressing

    - 충돌이 발생하면 다른 빈 버킷을 찾아 저장한다.

        ```
        A → bucket 3

        B → bucket 3
            이미 사용 중
                ↓
            bucket 4 저장
        ```

### 충돌이 많으면 왜 느려질까?

해시 테이블의 빠른 성능은 데이터가 여러 버킷에 잘 분산된다는 전제가 있다.

```
0 → A
1 → B
2 → C
3 → D
4 → E
```

하지만 모든 데이터가 하나의 버킷에 몰린다면 원하는 값을 찾기 위해 버킷 내부를 계속 탐색해야 한다.

```
bucket 3
 ↓
A → B → C → D → E → ...
```

따라서 일반적인 해시 테이블은 아래와 같다.

```
평균 → O(1)
최악 → O(N)
```

그래서 좋은 해시 함수는 키들을 가능한 고르게 분산시키는 것이 중요하다.

### 해시 테이블에서 중요한 조건

좋은 해시 함수는 다음 특성이 필요하다.

- 같은 키에 대해서는 항상 같은 해시 값을 만들어야 한다.
- 서로 다른 키가 특정 버킷에 지나치게 몰리지 않도록 고르게 분산해야 한다.
- 해시 계산 자체가 너무 무거우면 안 된다.

```
좋은 Hash Function
→ 충돌 감소
→ Bucket 내부 탐색 감소
→ 평균 O(1)에 가까운 성능
```

### 자바에서의 사용

```java
HashMap<K, V>
HashSet<E>
ConcurrentHashMap<K, V>
```

```java
Map<String, User> users = new HashMap<>();

users.put("kim", user);
User result = users.get("kim");
```

### 면접 연결 포인트

해시 테이블은 왜 평균 O(1)인가?

- 키의 해시 값을 이용해 데이터가 위치한 버킷을 바로 계산할 수 있기 때문이다. 다만 충돌이 발생하면 버킷 내부를 추가로 탐색해야 하기 때문에 항상 O(1)은 아니다.

서로 다른 키가 같은 해시 값을 가지면 어떻게 되는가?

- 해시 충돌이 발생하며, Chaining이나 Open Addressing 등의 방법으로 충돌을 처리한다.

해시 충돌은 완전히 없앨 수 있는가?

- 일반적으로 어렵다. 저장 가능한 버킷 수는 제한되어 있기 때문에 중요한 것은 충돌을 제거하는 것이 아니라 가능한 균등하게 분산하고 충돌이 발생했을 때 효율적으로 처리하는 것이다.

## 7. Tree

> 노드들이 부모-자식 관계로 연결된 계층형 자료구조이다.

배열이나 연결 리스트처럼 데이터가 일렬로 연결되는 것이 아니라, 하나의 노드에서 여러 방향으로 구조가 뻗어나간다.

```
        A
      /   \
     B     C
    / \   / \
   D   E F   G
```

파일 시스템, 조직도, DOM 구조처럼 계층 관계를 표현해야 하는 경우에 많이 사용한다.

### 트리의 기본 용어

```
        A
      /   \
     B     C
    / \   / \
   D   E F   G
```

| 용어      | 의미                            | 예시              |
| ------- | ----------------------------- | --------------- |
| Node    | Tree를 구성하는 각각의 데이터            | A, B, C         |
| Edge    | Node 사이의 연결                   | A-B             |
| Root    | 가장 위에 있는 Node                 | A               |
| Parent  | 바로 위에 연결된 Node                | B의 Parent = A   |
| Child   | 바로 아래 연결된 Node                | A의 Child = B, C |
| Sibling | 같은 Parent를 가진 Node            | B와 C            |
| Leaf    | Child가 없는 Node                | D, E, F, G      |
| Subtree | 특정 Node를 Root로 하는 Tree        | B-D-E           |
| Depth   | Root에서 특정 Node까지의 Edge 수      | D의 Depth = 2    |
| Height  | 특정 Node에서 가장 먼 Leaf까지의 Edge 수 | A의 Height = 2   |

### 이진 트리

이진 트리는 각 노드가 최대 2개의 자녀를 가지는 트리이다.

```
        A
       / \
      B   C
     / \
    D   E
```

중요한 점은 이진 트리라고 해서 데이터가 정렬되어 있는 것은 아니라는 것이다.

### BST

이진 탐색 트리는 이진 트리에 정렬 규칙을 추가한 자료구조이다.

기본적인 규칙은 아래와 같다.

```
왼쪽 Subtree < 현재 Node < 오른쪽 Subtree
```

전체 데이터를 하나씩 확인하지 않고, 비교 결과에 따라 한쪽 subtree를 탐색 대상에서 제외할 수 있다.

### BST는 왜 O(log N)일까?

트리가 균형 잡혀 있다면 한 번 비교할 때마다 탐색해야 할 범위가 크게 줄어든다.

```
            8
        /       \
       4         12
     /   \      /  \
    2     6   10    14
```

대략적으로 아래 형태로 탐색 범위가 줄어든다.

```
N
↓
N / 2
↓
N / 4
↓
N / 8
...
```

따라서 트리 높이가 약 log N이 되고 연산은 아래와 같다.

| 연산 | 균형 잡힌 BST |
| -- | --------: |
| 검색 |  O(log N) |
| 삽입 |  O(log N) |
| 삭제 |  O(log N) |

### BST는 항상 O(log N)일까?

그렇지 않다.

BST에 다음 순서대로 값을 삽입한다고 가정하자.

```
1, 2, 3, 4, 5
```

그러면 아래와 같이 한쪽으로 치우친 Skewed Tree가 만들어질 수 있다.

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

여기서 5를 찾으려면 모든 노드를 확인해야 한다.

결국 연결 리스트와 비슷한 구조가 되어 O(N)까지 성능이 떨어진다.

따라서 BST의 시간복잡도는 정확히는 아래와 같다.

```
평균적인 경우 → O(log N)
최악의 경우   → O(N)
```

### Balanced Tree가 필요한 이유

BST가 한쪽으로 치우치는 문제를 해결하기 위해 Self-Balancing Tree가 등장했다.

대표적으로 AVL 트리, Red-Black 트리가 있다.

데이터가 삽입되거나 삭제될 때 트리의 균형이 지나치게 무너지지 않도록 구조를 조정한다.

```
1
 \
  2
   \
    3

    ↓ 균형 조정

    2
   / \
  1   3
```

이를 통해 트리의 높이를 O(log N) 수준으로 유지하여 검색, 삽입, 삭제 성능이 O(log N)에서 크게 벗어나지 않도록 한다.

AVL 트리는 균형을 더 엄격하게 유지하며, Red-Black 트리는 AVL보다 균형 조건은 느슨하지만 삽입/삭제 시 균형 조정 부담이 상대적으로 적다.

자바에서는 TreeMap, TreeSet 등에 Red-Black 트리가 사용된다.

### 트리 순회

트리는 배열처럼 단순히 왼쪽부터 오른쪽으로 읽는 구조가 아니기 때문에 어떤 순서로 노드를 방문해야 할 것인지 정해야 한다.

대표적인 순회 방법이 다음 세 가지이다.

```
        A
       / \
      B   C
     / \
    D   E
```

**Preorder**

- Root → Left → Right

    ```
    A → B → D → E → C
    ```

- 현재 Node를 먼저 처리한다.

**Inorder**

- Left → Root → Right

    ```
    D → B → E → A → C
    ```

- 특히 BST를 Inorder 방식으로 순회하면 정렬된 순서로 데이터를 얻을 수 있다.

**Postorder**

- Left → Right → Root

    ```
    D → E → B → C → A
    ```

- Child를 먼저 처리한 뒤 Parent를 처리해야 하는 경우 사용할 수 있다.

### 트리의 시간 복잡도는 무조건 O(log N)인가?

그렇지 않다.

트리라는 자료구조 자체의 시간복잡도를 O(log N)이라고 하면 안 된다.

트리의 모양과 종류에 따라 달라진다.

```
일반 Tree
→ 특정 값 탐색 O(N) 가능

균형 잡힌 BST
→ 탐색 O(log N)

편향된 BST
→ 탐색 O(N)

Balanced BST
→ 탐색 O(log N)
```

> Tree라서 O(log N)이 아니라, 탐색 범위를 줄일 수 있는 구조와 적절한 높이가 유지되기 때문에 O(log N)인 것이다.

### 면접 연결 포인트

이진 트리와 이진 탐색 트리의 차이는 무엇인가?

- 이진 트리는 각 노드가 최대 2개의 자녀를 가진다는 구조적 조건만 존재한다. 이진 탐색 트리는 여기에 왼쪽에는 작은 값, 오른쪽에는 큰 값을 저장하는 정렬 규칙이 추가된다.

BST의 탐색은 항상 O(log N)인가?

- 그렇지 않다. 균형 잡힌 경우에는 높이가 O(log N)이므로 탐색도 O(log N)이지만, 한쪽으로 치우치면 높이가 O(N)이 되어 탐색 역시 O(N)이 될 수 있다.

Balanced Tree를 사용하는 이유는 무엇인가?

- BST가 한쪽으로 편향되면서 O(N)으로 성능이 저하되는 것을 방지하기 위해 트리의 높이를 O(log N) 수준으로 유지하기 위해 사용한다.

BST를 Inorder로 순회하면 어떻게 되는가?

- BST의 왼쪽 < Root < 오른쪽 규칙 때문에 데이터를 오름차순으로 얻을 수 있다.

## 8. Heap

> 최댓값 또는 최솟값을 빠르게 찾기 위해 사용하는 완전 이진 트리 기반의 자료구조이다.

```
Max Heap
→ 부모 ≥ 자식
→ Root = 최댓값

Min Heap
→ 부모 ≤ 자식
→ Root = 최솟값
```

Min Heap은 아래와 같으며, 모든 부모가 자신의 자식보다 작거나 같다.

따라서 최솟값은 항상 Root에 존재한다.

```
        1
      /   \
     3     2
    / \   /
   7   5  8
```

### 힙의 두 가지 조건

힙은 아무 이진 트리나 사용할 수 있는 것이 아니다.

**① 완전 이진 트리**

마지막 Level을 제외한 모든 Level이 채워져 있고, 마지막 Level은 왼쪽부터 차례대로 채워진다.

```
        1
      /   \
     3     2
    / \   /
   7   5  8
```

**② Heap Property**

Min Heap이라면 부모 ≤ 자식,
Max Heap이라면 부모 ≥ 자식이라는 규칙을 만족해야 한다.

### 힙은 정렬된 트리일까?

Min Heap이 아래와 같을 수 있다.

```
        1
      /   \
     3     2
    / \   / \
   7   8  5   4
```

힙이 보장하는 것은 부모 ≤ 자식뿐이다. 따라서 형제끼리는 어떤 순서여도 상관없다.

> 즉, Min Heap은 Root가 최솟값이라는 것은 보장하지만 전체 데이터가 정렬되어 있는 것은 아니다.

따라서 최솟값 조회는 O(1)이지만 특정 값을 검색하는 것은 O(N)이다.

### 힙은 왜 배열로 구현할까?

힙은 트리이지만 일반적으로 배열을 사용해서 구현한다.

완전 이진 트리는 노드가 왼쪽부터 빈틈없이 채워지기 때문에 배열에 그대로 넣을 수 있기 때문이다.

```
        10
       /  \
      8    7
     / \
    3   5
```

배열로 표현하면 아래와 같다.

```
index   0   1   2   3   4
      ┌───┬───┬───┬───┬───┐
      │10 │ 8 │ 7 │ 3 │ 5 │
      └───┴───┴───┴───┴───┘
```

별도의 left, right 참조가 없어도 index 계산으로 부모와 자식을 찾을 수 있다.

0-based index 기준으로 아래와 같다.

```
현재 Node = i

부모      = (i - 1) / 2
왼쪽 자식  = 2i + 1
오른쪽 자식 = 2i + 2
```

예를 들어 index 1의 자식은 아래와 같이 계산한다.

```
Left  = 2 × 1 + 1 = 3
Right = 2 × 1 + 2 = 4
```

### 삽입/삭제

Heap에 데이터를 삽입할 때는 완전 이진 트리를 유지하기 위해 가장 마지막 위치에 삽입한 뒤 부모와 비교하며 위로 이동한다.

이를 Sift Up이라고 한다.

```
        1
       / \
      4   3
     /
    2

       ↓

        1
       / \
      2   3
     /
    4
```

Tree의 높이가 log N이므로 삽입은 O(log N) 이다.

삭제할 때는 Root를 제거한 뒤 마지막 Node를 Root로 옮기고 자식과 비교하며 아래로 이동한다.

이를 Sift Down이라고 한다.

```
Root 제거
   ↓
마지막 Node를 Root로 이동
   ↓
자식과 비교하며 내려감
```

따라서 삭제 역시 O(log N) 이다.

| 연산         |    시간복잡도 |
| ---------- | -------: |
| 최솟값/최댓값 조회 |     O(1) |
| 삽입         | O(log N) |
| Root 삭제    | O(log N) |
| 특정 값 검색    |     O(N) |

특정 값 검색이 O(N)인 이유는 Heap이 전체 정렬을 보장하지 않기 때문이다.

Min Heap도 부모 ≤ 자식만 보장한다.

```
        1
       / \
      4   2
     / \
    7   5
```

따라서 5를 찾을 때 BST처럼 왼쪽이나 오른쪽 중 한쪽을 버릴 수 없다.

### 자바에서는 PriorityQueue를 사용한다

Java에서는 Heap을 직접 구현하기보다 주로 PriorityQueue를 사용한다.

기본적으로 최소값이 가장 높은 우선순위를 가진다.

```java
PriorityQueue<Integer> pq = new PriorityQueue<>();

pq.offer(5);
pq.offer(1);
pq.offer(3);

pq.peek(); // 1
pq.poll(); // 1
```

Max Heap처럼 사용하려면 Comparator를 지정한다.

```java
PriorityQueue<Integer> pq =
        new PriorityQueue<>(Comparator.reverseOrder());
```

## 9. Java Collection Framework

> 여러 데이터를 저장하고 처리하기 위한 자료구조를 표준화하여 제공하는 Java의 인터페이스와 구현체 집합이다.

```java
List<String> list = new ArrayList<>();
Set<String> set = new HashSet<>();
Queue<String> queue = new ArrayDeque<>();
Map<String, Integer> map = new HashMap<>();
```

### 전체 구조

```
                    Iterable
                       │
                  Collection
              ┌────────┼────────┐
             List      Set     Queue
              │         │        │
         ArrayList   HashSet    Deque
         LinkedList  TreeSet      │
                              ArrayDeque


Map
├─ HashMap
├─ LinkedHashMap
├─ TreeMap
└─ ConcurrentHashMap
```

여기서 중요한 점은 Map은 Collection의 하위 인터페이스가 아니라는 것이다.

Collection은 하나의 Element들을 관리하지만 Map은 Key-Value 쌍을 관리하기 때문에 별도의 인터페이스로 존재한다.

### List

List는 순서가 있고 중복을 허용한다.

대표적인 구현체는 ArrayList, LinkedList다.

```java
List<String> list = new ArrayList<>();

list.add("A");
list.add("B");
list.add("A");
```

**ArrayList**

내부적으로 배열을 사용한다.

- 인덱스 접근: O(1)
- 검색: O(N)
- 중간 삽입/삭제: O(N)
- 끝에 삽입: Amortized O(1)

용량이 부족하면 더 큰 배열을 만들고 기존 데이터를 복사하기 때문에 가끔 O(N)이 발생하지만, 평균적으로 add()는 Amortized O(1) 이다.

**LinkedList**

Java의 LinkedList는 Doubly Linked List 기반이다.

- 인덱스 접근: O(N)
- 검색: O(N)
- 위치를 이미 알고 있는 삽입/삭제: O(1)

일반적인 List가 필요하다면 보통 ArrayList를 우선 고려한다.

### Set

Set은 중복을 허용하지 않는 자료구조다.

```java
Set<String> set = new HashSet<>();

set.add("A");
set.add("B");
set.add("A");
```

| 구현체           | 특징                     |
| ------------- | ---------------------- |
| HashSet       | 일반적인 중복 제거, 평균 O(1) 조회 |
| LinkedHashSet | 삽입 순서 유지               |
| TreeSet       | 정렬된 순서 유지              |

HashSet은 내부적으로 HashMap을 사용한다.

개념적으로는 아래처럼 Set의 값을 HashMap의 Key로 사용하는 구조다.

```java
HashSet.add("A")
        ↓
HashMap.put("A", dummyValue)
```

### Queue/Deque

Queue는 FIFO 구조다.

```java
Queue<Integer> queue = new ArrayDeque<>();

queue.offer(1);
queue.offer(2);

queue.poll(); // 1
```

Deque는 Double Ended Queue로 양쪽에서 삽입과 삭제가 가능하다. 따라서 Queue뿐만 아니라 Stack으로도 사용할 수 있다.

```java
Deque<Integer> stack = new ArrayDeque<>();

stack.push(1);
stack.push(2);
stack.pop();
```

Java에서는 일반적으로 오래된 Stack 클래스보다 ArrayDeque를 Stack 용도로 사용하는 것을 권장한다.

### Map

Map은 Key-Value 형태로 데이터를 저장한다.

```java
Map<String, User> users = new HashMap<>();

users.put("kim", user1);
users.put("lee", user2);
```

Key는 중복될 수 없고 Value는 중복될 수 있다.

| 구현체               | 특징                |
| ----------------- | ----------------- |
| HashMap           | 일반적인 Key-Value 저장 |
| LinkedHashMap     | 순서 유지             |
| TreeMap           | Key 기준 정렬         |
| ConcurrentHashMap | 멀티스레드 환경          |

### 어떤 컬렉션을 선택할까?

```
순서가 있는 목록
→ ArrayList

중복 제거
→ HashSet

정렬된 중복 없는 데이터
→ TreeSet

Queue / Stack
→ ArrayDeque

Key-Value
→ HashMap

Key 정렬 필요
→ TreeMap

멀티스레드에서 공유하는 Map
→ ConcurrentHashMap
```

자료구조를 선택할 때는 단순히 시간복잡도만 보는 것이 아니라 순서, 중복, 정렬, 접근 방식, 동시성 여부를 함께 고려해야 한다.

## 10. HashMap 내부 구조

> HashMap은 Key의 Hash 값을 이용해 저장할 Bucket을 찾고, 해당 Bucket에서 Key를 비교해 Value를 저장하거나 조회하는 자료구조다.

```
Key
 ↓
hashCode()
 ↓
Hash 계산
 ↓
Bucket 결정
 ↓
Key 비교
 ↓
Value
```

평균적으로 put(), get()은 O(1) 이다.

### 내부 구조

HashMap 내부에는 Bucket 배열이 존재한다.

```
index
  0 → null
  1 → [A]
  2 → null
  3 → [B] → [C]
  4 → [D]
```

put(key, value)을 호출하면 다음과 같이 동작한다.

```
1. key.hashCode() 계산
2. Hash 값을 가공
3. Bucket 위치 결정
4. Bucket이 비어 있으면 저장
5. 데이터가 있다면 Key 비교
   → 같은 Key: Value 변경
   → 다른 Key: 충돌 처리
```

get() 역시 Hash를 이용해 Bucket을 찾은 뒤 Key를 비교해 Value를 찾는다.

### hashCode()와 equals()

```
hashCode()
→ 어느 Bucket을 확인할지 결정

equals()
→ 실제로 같은 Key인지 확인
```

객체를 HashMap의 Key로 사용할 때는 equals()가 같은 객체는 같은 hashCode()를 가져야 한다.

그래서 equals()를 재정의한다면 일반적으로 hashCode()도 함께 재정의해야 한다.

### Hash Collision

서로 다른 Key가 같은 Bucket에 들어가는 것을 Hash Collision이라고 한다.

충돌이 많아지면 Bucket 내부에서 여러 Key를 확인해야 하므로 성능이 떨어질 수 있다.

Java 8 이후 HashMap은 한 Bucket에 데이터가 과도하게 몰리면 일정 조건에서 Linked List를 Red-Black Tree로 변환(Treeify) 하여 탐색 성능 저하를 완화한다.

```
Linked List → O(N)
Red-Black Tree → O(log N)
```

### Load Factor와 Resize

데이터가 많아질수록 Hash Collision 가능성이 높아지므로 HashMap은 일정 수준 이상 차면 Bucket 배열의 크기를 늘린다.

기본 Load Factor는 0.75다.

```
Capacity × Load Factor
→ Resize 기준

16 × 0.75 = 12
```

임계치를 넘으면 더 큰 배열로 Resize한다.

Resize 자체는 비용이 크기 때문에 put()이 항상 O(1)인 것은 아니지만, 일반적으로 평균 O(1)로 본다.

## 11. ConcurrentHashMap

> ConcurrentHashMap은 여러 Thread가 동시에 접근하고 수정해도 안전하게 사용할 수 있도록 설계된 Map이다.

```java
Map<String, Integer> map = new ConcurrentHashMap<>();
```

일반적인 HashMap은 Thread-Safe하지 않기 때문에 여러 Thread가 동시에 공유하고 수정하면 데이터 정합성 문제가 발생할 수 있다.

### HashMap과 무엇이 다를까?

```
HashMap
Thread A ─┐
          ├─→ 같은 데이터에 동시 접근 → 동시성 문제 가능
Thread B ─┘

ConcurrentHashMap
Thread A ─┐
          ├─→ 동시 접근을 안전하게 처리
Thread B ─┘
```

단순히 Map 전체를 하나의 Lock으로 막으면 안전하게 만들 수는 있다. 하지만 아래와 같이 서로 다른 데이터를 처리하는 경우에도 기다려야 해서 동시성이 떨어진다.

```
Thread A → Key A 수정 중
Thread B → Key B 수정하려고 함
```

ConcurrentHashMap은 Map 전체를 하나의 Lock으로 막지 않고 필요한 부분에만 동기화를 적용해 여러 Thread의 동시 접근 성능을 높인다.

### 어떻게 동시성을 보장할까?

Java 8 이후 ConcurrentHashMap은 주로 CAS와 synchronized를 조합해 데이터를 안전하게 수정한다.

개념적으로는 아래와 같이 동작한다.

```
비어 있는 Bucket에 삽입
→ CAS를 이용한 원자적 처리

이미 데이터가 있는 Bucket 수정
→ 해당 Bucket 단위로 synchronized
```

따라서 Map 전체에 Lock을 거는 방식보다 여러 Thread가 서로 다른 Bucket을 수정할 때 동시에 작업하기 유리하다.

### 원자적 연산에 주의하기

ConcurrentHashMap을 사용한다고 해서 여러 연산을 조합한 코드까지 자동으로 Thread-Safe해지는 것은 아니다.

```java
if (!map.containsKey(key)) {
    map.put(key, value);
}
```

containsKey()와 put() 각각은 안전하지만 두 연산 사이에 다른 Thread가 끼어들 수 있다.

이럴 때는 하나의 원자적 연산을 사용해야 한다.

```java
map.putIfAbsent(key, value);

map.computeIfAbsent(key, k -> createValue());
```

Concurrent Collection을 사용하더라도 여러 연산을 묶으면 원자성이 자동으로 보장되는 것은 아니다.

### HashMap vs ConcurrentHashMap

|              | HashMap  | ConcurrentHashMap   |
| ------------ | -------- | ------------------- |
| Thread-Safe  | ❌        | ✅                   |
| 동시 수정        | 안전하지 않음  | 가능                  |
| `null` Key   | 허용       | 허용하지 않음             |
| `null` Value | 허용       | 허용하지 않음             |
| 주요 사용        | 일반적인 Map | 여러 Thread가 공유하는 Map |
