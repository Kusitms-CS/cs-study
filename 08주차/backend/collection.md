# Collection

Collection Framework의 계층 구조, HashMap 내부 구조, ConcurrentHashMap은 [1주차 자바 심화](../../01주차/java-collection.md)에 정리했다. 여기서는 구현체 선택과 사용할 때 주의할 점을 중심으로 정리한다.

## List 구현체 비교

| 연산 | ArrayList | LinkedList |
| --- | --- | --- |
| 내부 구조 | 동적 배열 | 이중 연결 리스트 |
| 인덱스 조회 `get(i)` | O(1) | O(n) |
| 끝에 추가 `add(e)` | 평균 O(1) (가끔 배열 확장) | O(1) |
| 중간 삽입/삭제 | O(n) (요소 이동) | 위치를 찾는 데 O(n), 연결 변경은 O(1) |
| 메모리 | 연속된 배열, 캐시 친화적 | 노드마다 앞뒤 포인터 추가 |

- `ArrayList`는 용량이 차면 **기존의 1.5배** 크기로 새 배열을 만들어 복사한다. 크기를 미리 알면 `new ArrayList<>(size)`로 초기 용량을 지정해 복사 비용을 줄인다.
- 이론상 중간 삽입은 `LinkedList`가 유리해 보이지만, 위치를 찾는 순회 비용과 캐시 미스 때문에 **실무에서는 대부분 `ArrayList`가 더 빠르다.**
- 큐/스택이 필요하면 `LinkedList`나 `Stack`보다 **`ArrayDeque`** 를 쓴다. (`Stack`은 `Vector` 기반이라 불필요한 동기화가 있다)

## Set / Map 구현체 비교

| | HashSet / HashMap | LinkedHashSet / LinkedHashMap | TreeSet / TreeMap |
| --- | --- | --- | --- |
| 내부 구조 | 해시 테이블 | 해시 테이블 + 연결 리스트 | Red-Black Tree |
| 순서 | 보장 안 함 | 입력 순서 유지 | 키 정렬 순서 |
| 조회/삽입 | 평균 O(1) | 평균 O(1) | O(log n) |
| null 키 | 허용 | 허용 | 불가 (비교 불가) |

- `HashSet`은 내부적으로 `HashMap`의 키만 사용하는 구조다.
- `LinkedHashMap`은 생성자에 `accessOrder = true`를 주고 `removeEldestEntry`를 오버라이드하면 간단한 **LRU 캐시**를 만들 수 있다.
- `TreeMap`은 `floorKey`, `ceilingKey`, `subMap`처럼 **범위 탐색**이 필요할 때 유용하다.

## equals와 hashCode

`HashMap`, `HashSet`은 키를 찾을 때 `hashCode()`로 버킷을 찾고, `equals()`로 같은 객체인지 비교한다.

규약

- `equals()`가 `true`인 두 객체는 **반드시 같은 `hashCode()`** 를 반환해야 한다.
- `hashCode()`가 같다고 `equals()`가 `true`일 필요는 없다. (해시 충돌)

```java
class User {
    private final String email;
    User(String email) { this.email = email; }

    @Override
    public boolean equals(Object o) {
        if (this == o) return true;
        if (!(o instanceof User user)) return false;
        return email.equals(user.email);
    }
    // hashCode를 재정의하지 않으면?
}

Set<User> set = new HashSet<>();
set.add(new User("a@a.com"));
set.contains(new User("a@a.com"));   // false! 서로 다른 hashCode → 다른 버킷을 찾음
```

- `equals()`를 재정의하면 `hashCode()`도 반드시 함께 재정의한다. (`Objects.hash(email)`)
- **HashMap의 키로 쓰는 객체는 불변으로** 만드는 것이 좋다. 넣은 뒤 필드가 바뀌면 `hashCode`가 바뀌어 다시 찾을 수 없다.
- `record`는 `equals`/`hashCode`/`toString`을 자동으로 만들어준다.

## Comparable과 Comparator

| | Comparable | Comparator |
| --- | --- | --- |
| 패키지 | `java.lang` | `java.util` |
| 메서드 | `compareTo(T o)` | `compare(T a, T b)` |
| 위치 | 클래스 내부에 구현 | 외부에서 별도로 정의 |
| 용도 | 객체의 **기본(자연) 정렬 기준** | 상황에 따른 **다른 정렬 기준** |

```java
class Member implements Comparable<Member> {
    String name; int age;
    @Override
    public int compareTo(Member o) { return Integer.compare(age, o.age); }   // 기본: 나이순
}

members.sort(Comparator.comparing(Member::getName)
                       .thenComparing(Member::getAge, Comparator.reverseOrder()));
```

- 비교할 때 `a.age - b.age`처럼 빼기를 쓰면 **오버플로**가 날 수 있어 `Integer.compare`를 쓴다.
- `TreeSet`/`TreeMap`은 `equals`가 아니라 `compareTo`/`compare` 결과가 0이면 같은 원소로 본다. 정렬 기준이 `equals`와 다르면 원소가 누락될 수 있다.

## 순회 중 수정과 불변 컬렉션

**fail-fast Iterator**

```java
for (String s : list) {
    if (s.isEmpty()) list.remove(s);   // ConcurrentModificationException
}
```

- 컬렉션은 내부적으로 수정 횟수(`modCount`)를 기록하고, Iterator는 순회 중 이 값이 바뀌면 예외를 던진다.
- 순회 중 삭제는 `iterator.remove()` 또는 `list.removeIf(String::isEmpty)`를 쓴다.
- 반면 `ConcurrentHashMap`, `CopyOnWriteArrayList`의 Iterator는 **fail-safe**(weakly consistent)라서 순회 중 수정해도 예외가 나지 않는다.

**불변 컬렉션**

| 방식 | 원본 변경 시 | 수정 시도 | null |
| --- | --- | --- | --- |
| `Collections.unmodifiableList(list)` | **반영됨** (원본을 감싼 뷰) | `UnsupportedOperationException` | 허용 |
| `List.of(...)` (Java 9+) | 원본 없음 | `UnsupportedOperationException` | 불가 |
| `List.copyOf(list)` (Java 10+) | 반영 안 됨 (복사본) | `UnsupportedOperationException` | 불가 |
| `stream.toList()` (Java 16+) | 반영 안 됨 | `UnsupportedOperationException` | 허용 |

- `Arrays.asList(arr)`는 크기 고정 리스트다. `set`은 되지만 `add`/`remove`는 예외가 나고, 원본 배열과 연결되어 있다.
- 엔티티나 DTO에서 내부 리스트를 그대로 반환하면 외부에서 수정할 수 있으므로, getter에서 불변 뷰나 복사본을 반환하는 것이 안전하다.
