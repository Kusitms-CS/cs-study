# 8주차 - Java / JVM

## 키워드
- JVM/JDK/JRE
- Generic
- Collection
- Exception
- JVM Architecture
- Class Loader
- GC

---

## 1. JVM / JDK / JRE

| 구분 | 설명 |
|---|---|
| **JVM** | 바이트코드를 실행하는 가상 머신. OS와 상관없이 같은 코드를 돌릴 수 있게 해줌 |
| **JRE** | JVM + 표준 라이브러리. 자바 프로그램을 **실행**만 할 때 필요 |
| **JDK** | JRE + 컴파일러(`javac`) 등 개발 도구. 자바 프로그램을 **개발**할 때 필요 |

관계: `JDK ⊃ JRE ⊃ JVM`

**실행 흐름**

```
.java → (javac 컴파일) → .class (바이트코드) → JVM → 기계어 실행
```

- 바이트코드는 OS에 종속되지 않아서 JVM만 있으면 어디서든 실행됨 (Write Once, Run Anywhere)

---

## 2. Generic

타입을 변수처럼 파라미터로 받는 문법. `List<String>`처럼 씀.

```java
List<String> list = new ArrayList<>();
list.add("hello");
String s = list.get(0); // 형변환 필요 없음
```

**장점**
- 컴파일 시점에 타입 체크 → 런타임 에러 방지
- 형변환(캐스팅) 불필요

**특징**
- **타입 소거(Type Erasure)**: 컴파일이 끝나면 제네릭 타입 정보가 사라짐
  - 그래서 런타임에는 `List<String>`과 `List<Integer>`를 구분 못 함
  - `new T()`, `T.class` 같은 건 못 씀
- **와일드카드**
  - `? extends T`: T 또는 T의 하위 타입 (주로 **읽기**)
  - `? super T`: T 또는 T의 상위 타입 (주로 **쓰기**)

---

## 3. Collection

데이터를 담는 자료구조 모음. 크게 List, Set, Map(+Queue)이 있음.
(Map은 Collection 인터페이스를 상속하지 않지만 컬렉션 프레임워크에 포함됨)

| 종류 | 특징 | 대표 구현체 |
|---|---|---|
| **List** | 순서 O, 중복 O | `ArrayList`, `LinkedList` |
| **Set** | 순서 X, 중복 X | `HashSet`, `LinkedHashSet`, `TreeSet` |
| **Map** | key-value, key 중복 X | `HashMap`, `LinkedHashMap`, `TreeMap` |
| **Queue** | FIFO | `LinkedList`, `PriorityQueue` |

**비교 대상 모음**
- `ArrayList` vs `LinkedList`
  - ArrayList: 배열 기반, **조회 빠름**, 중간 삽입/삭제 느림
  - LinkedList: 노드 연결 기반, **삽입/삭제 빠름**, 조회 느림
- `HashSet` / `LinkedHashSet` / `TreeSet`
  - 순서 없음 / 입력 순서 유지 / 정렬 상태 유지
- `HashMap`
  - 해시 기반이라 평균 O(1)
  - key로 쓰는 객체는 `equals()`와 `hashCode()`를 같이 재정의해야 함

---

## 4. Exception

```
Throwable
 ├─ Error        (복구 불가: OutOfMemoryError, StackOverflowError)
 └─ Exception
     ├─ Checked Exception    (RuntimeException 아닌 것)
     └─ Unchecked Exception  (RuntimeException 하위)
```

| 구분 | 특징 | 예시 |
|---|---|---|
| **Error** | JVM 레벨의 심각한 문제. 보통 잡지 않음 | `OutOfMemoryError` |
| **Checked** | 컴파일러가 처리(try-catch / throws)를 **강제** | `IOException`, `SQLException` |
| **Unchecked** | 처리 강제 X. 주로 프로그래밍 실수 | `NullPointerException`, `IllegalArgumentException` |

**처리 방법**
```java
try {
    // 예외 발생 가능 코드
} catch (IOException e) {
    // 예외 처리
} finally {
    // 무조건 실행 (정리 작업)
}
```

- `try-with-resources`: `AutoCloseable` 구현체를 자동으로 close 해줌
- `throw`: 예외를 직접 발생시킴 / `throws`: 메서드가 예외를 던질 수 있다고 선언

---

## 5. JVM Architecture

JVM은 크게 **3가지**로 구성됨.

```
[ Class Loader ] → [ Runtime Data Area ] ← [ Execution Engine ]
```

### Runtime Data Area (메모리 영역)

| 영역 | 설명 | 공유 여부 |
|---|---|---|
| **Method Area** | 클래스 정보, static 변수, 상수 | 모든 스레드 공유 |
| **Heap** | `new`로 만든 객체. **GC 대상** | 모든 스레드 공유 |
| **Stack** | 메서드 호출 프레임, 지역변수 | 스레드별 |
| **PC Register** | 현재 실행 중인 명령 위치 | 스레드별 |
| **Native Method Stack** | 네이티브(C/C++) 코드 호출용 | 스레드별 |

### Execution Engine
- **Interpreter**: 바이트코드를 한 줄씩 해석해서 실행
- **JIT Compiler**: 자주 실행되는 코드를 기계어로 컴파일해서 캐싱 → 속도 향상
- **GC**: 안 쓰는 객체 정리

---

## 6. Class Loader

클래스를 **필요할 때** JVM 메모리에 동적으로 로딩해주는 모듈.

**로딩 과정**
1. **Loading**: `.class` 파일을 읽어서 메모리에 올림
2. **Linking**
   - Verify: 바이트코드가 올바른지 검증
   - Prepare: static 변수 메모리 할당, 기본값으로 초기화
   - Resolve: 심볼릭 참조를 실제 참조로 변경
3. **Initialization**: static 변수에 실제 값 할당, static 블록 실행

**계층 구조**
```
Bootstrap  →  Platform  →  Application
(java.lang 등)  (확장 클래스)   (내가 만든 클래스)
```
- 클래스를 로딩할 때 먼저 **부모에게 위임**하고, 부모가 못 찾으면 자기가 로딩
- 덕분에 `java.lang.String` 같은 핵심 클래스를 누가 바꿔치기하는 걸 막을 수 있음

---

## 7. GC (Garbage Collection)

더 이상 **참조되지 않는 객체**를 자동으로 메모리에서 제거하는 기능.
개발자가 직접 `free` 안 해도 됨.

**Heap 구조 (Generational)**
```
Heap
 ├─ Young Generation: Eden + Survivor(S0, S1)
 └─ Old Generation
```
- 대부분의 객체는 금방 죽는다는 가정(weak generational hypothesis)에서 출발

**동작 흐름**
1. 새 객체는 **Eden**에 생성
2. Eden이 차면 **Minor GC** → 살아남은 객체는 Survivor로 이동
3. Survivor를 오가며 나이(age)가 쌓이면 **Old**로 승격
4. Old가 차면 **Major(Full) GC** → 오래 걸림

**특징**
- **Stop-The-World**: GC 실행 중 애플리케이션 스레드가 멈추는 현상. GC 튜닝의 핵심은 이 시간을 줄이는 것
- 기본 알고리즘: **Mark → Sweep → Compact**
  - Mark: 살아있는 객체 표시
  - Sweep: 표시 안 된 객체 제거
  - Compact: 흩어진 메모리 정리

