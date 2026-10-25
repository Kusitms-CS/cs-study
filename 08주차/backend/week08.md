
# 8주차 - Java / JVM

## 1. JVM / JDK / JRE

### 1.1 JVM (Java Virtual Machine)
- Java 바이트코드를 실행하는 가상 머신
- 운영체제에 관계없이 Java 프로그램을 실행할 수 있도록 지원하는 환경
- 메모리 관리, 가비지 컬렉션, 바이트코드 실행 등을 담당
- JVM 자체는 운영체제에 종속적이지만 바이트코드는 플랫폼 독립적임

### 1.2 JRE (Java Runtime Environment)
- Java 프로그램 실행에 필요한 환경
- JVM과 Java 표준 라이브러리 등으로 구성
- Java 프로그램 실행을 위한 구성 요소이며 개발 도구는 포함하지 않음

### 1.3 JDK (Java Development Kit)
- Java 프로그램 개발에 필요한 도구 모음
- 컴파일러(javac), 디버거 등 개발 도구와 실행 환경을 포함
- Java 소스코드를 바이트코드로 컴파일하는 기능 제공

### 1.4 Java 실행 과정
1. 개발자가 Java 소스코드(.java) 작성
2. Java 컴파일러(javac)가 바이트코드(.class) 생성
3. Class Loader가 바이트코드를 JVM 메모리에 로드
4. Execution Engine이 바이트코드를 실행
5. 실행 과정에서 JVM이 메모리 관리 및 GC 수행

---

## 2. Generic

### 2.1 Generic 개념
- 클래스, 인터페이스, 메서드에서 사용할 데이터 타입을 매개변수화하는 기능
- 컴파일 시점에 타입 안정성을 보장하기 위한 기능
- 불필요한 형변환을 줄이고 코드 재사용성을 높이는 역할
- 잘못된 타입 사용을 컴파일 단계에서 검출 가능

### 2.2 Generic 예시

```java
List<String> names = new ArrayList<>();

names.add("Java");
names.add("Spring");

// 컴파일 오류
// names.add(100);
```

- String 타입만 저장하도록 제한
- 다른 타입의 데이터 추가 시 컴파일 오류 발생

### 2.3 Generic 주요 특징
- 타입 매개변수로 T, E, K, V 등을 주로 사용
  - T : Type
  - E : Element
  - K : Key
  - V : Value
- 와일드카드(?)를 통해 타입 범위를 유연하게 지정 가능
- `<? extends T>` : T 또는 T의 하위 타입을 허용
- `<? super T>` : T 또는 T의 상위 타입을 허용
- Type Erasure : 컴파일 과정에서 제네릭 타입 정보가 제거되거나 경계 타입으로 대체되는 방식

---

## 3. Collection

### 3.1 Collection Framework
- 여러 데이터를 저장하고 관리하기 위한 Java 자료구조 프레임워크
- List, Set, Queue 등의 인터페이스와 구현체 제공
- Map은 Collection 인터페이스를 상속하지 않는 별도의 자료구조

### 3.2 List
- 데이터의 저장 순서를 유지하는 자료구조
- 중복 데이터 저장 허용
- 인덱스를 통한 데이터 접근 가능

**ArrayList**
- 내부적으로 동적 배열을 사용하는 List 구현체
- 인덱스 기반 조회 시간복잡도 O(1)
- 중간 삽입 및 삭제 시 요소 이동으로 O(n) 발생 가능

**LinkedList**
- 노드 간 연결 구조를 사용하는 List 구현체
- 특정 위치 탐색 시간복잡도 O(n)
- 노드 위치를 알고 있는 경우 연결 변경을 통한 삽입 및 삭제 O(1)

### 3.3 Set
- 중복 데이터를 허용하지 않는 자료구조

**HashSet**
- 해시 테이블 기반으로 데이터 저장
- 저장 순서를 보장하지 않음
- 평균 삽입, 삭제, 탐색 시간복잡도 O(1)

**TreeSet**
- 정렬된 상태로 데이터를 관리하는 Set
- Red-Black Tree 기반으로 구현
- 삽입, 삭제, 탐색 시간복잡도 O(log n)

### 3.4 Map
- Key-Value 형태로 데이터를 저장하는 자료구조
- Key 중복 불가, Value 중복 허용

**HashMap**
- 해시 테이블 기반으로 데이터 저장
- 평균 조회, 삽입, 삭제 시간복잡도 O(1)
- 동기화를 기본적으로 지원하지 않음

**ConcurrentHashMap**
- 멀티스레드 환경에서 안전하게 사용할 수 있는 Map
- 동시 접근 시 데이터 일관성을 유지하도록 설계
- 전체 Map을 잠그지 않고 높은 동시성을 지원

### 3.5 Queue
- 일반적으로 FIFO(First In First Out) 방식의 자료구조
- 먼저 들어온 데이터를 먼저 처리하는 구조

**PriorityQueue**
- 우선순위에 따라 데이터를 처리하는 Queue
- 내부적으로 Heap 자료구조 사용
- 삽입 및 삭제 시간복잡도 O(log n)

---

## 4. Exception

### 4.1 Exception 개념
- 프로그램 실행 중 발생하는 비정상적인 상황을 나타내는 객체
- 예외 처리를 통해 프로그램의 비정상적인 종료를 방지하거나 오류에 대응 가능
- Java에서는 Throwable을 최상위 클래스로 사용

### 4.2 Checked Exception
- 컴파일 시점에 예외 처리 여부를 검사하는 예외
- 예외 처리 또는 throws 선언이 필요함
- IOException, SQLException 등이 대표적

### 4.3 Unchecked Exception
- 컴파일 시점에 예외 처리를 강제하지 않는 예외
- RuntimeException과 그 하위 클래스가 해당
- NullPointerException, IllegalArgumentException 등이 대표적

### 4.4 Error
- JVM 또는 시스템 수준에서 발생하는 심각한 문제
- 일반적인 애플리케이션 예외 처리로 복구하기 어려운 상황
- OutOfMemoryError, StackOverflowError 등이 대표적

### 4.5 예외 처리 키워드
- try : 예외 발생 가능성이 있는 코드 작성
- catch : 발생한 예외를 처리하는 블록
- finally : 예외 발생 여부와 관계없이 실행되는 블록
- throw : 예외 객체를 명시적으로 발생시키는 키워드
- throws : 메서드에서 발생할 수 있는 예외를 호출자에게 전달하도록 선언

### 4.6 예외 처리 예시

```java
try {
    int result = 10 / 0;
} catch (ArithmeticException e) {
    System.out.println("0으로 나눌 수 없음");
} finally {
    System.out.println("작업 종료");
}
```

- ArithmeticException 발생 시 catch 블록 실행
- finally 블록은 일반적인 예외 처리 흐름에서 항상 실행

---

## 5. JVM Architecture

### 5.1 JVM 주요 구성 요소
- Class Loader : 클래스 파일을 JVM 메모리에 로드
- Runtime Data Area : 프로그램 실행에 필요한 메모리 영역
- Execution Engine : 바이트코드를 실행하는 엔진
- JNI : Java 코드와 네이티브 코드 간 상호작용 지원
- Native Method Libraries : 네이티브 메서드 실행에 필요한 라이브러리

### 5.2 Runtime Data Area

**Method Area**
- 클래스 메타데이터, 메서드 정보, 런타임 상수 풀 등을 저장
- 모든 스레드가 공유하는 메모리 영역
- HotSpot JVM에서는 클래스 메타데이터를 주로 Metaspace에 저장

**Heap**
- 객체와 배열이 생성되는 메모리 영역
- 모든 스레드가 공유하는 영역
- GC의 주요 관리 대상

**Stack**
- 스레드마다 독립적으로 생성되는 메모리 영역
- 메서드 호출 시 Stack Frame 생성
- 지역변수, 매개변수, 중간 연산 결과 등을 저장
- 메서드 종료 시 해당 Stack Frame 제거

**PC Register**
- 각 스레드가 현재 실행 중인 JVM 명령어의 위치를 관리
- 스레드마다 독립적으로 존재

**Native Method Stack**
- JNI 등을 통해 호출되는 네이티브 메서드 실행을 지원하는 영역

### 5.3 Execution Engine
- JVM 바이트코드를 실제로 실행하는 구성 요소

**Interpreter**
- 바이트코드를 명령어 단위로 해석하여 실행
- 초기 실행이 빠르지만 반복 실행 시 성능 저하 가능

**JIT Compiler**
- 자주 실행되는 코드를 네이티브 머신 코드로 컴파일
- 반복적인 바이트코드 해석 비용을 줄여 실행 성능 향상
- 컴파일 과정에서 추가적인 CPU 및 메모리 자원 사용

---

## 6. Class Loader

### 6.1 Class Loader 개념
- 컴파일된 클래스 파일(.class)을 JVM 메모리에 동적으로 로드하는 구성 요소
- 클래스가 필요한 시점에 로딩하는 방식 지원

### 6.2 클래스 로딩 과정

**Loading**
- 클래스 파일의 바이트코드를 읽어 JVM 메모리에 로드
- 클래스 정보를 기반으로 Class 객체 생성

**Linking**
- Verification : 바이트코드의 유효성 및 안전성 검증
- Preparation : static 필드에 기본값을 할당하는 단계
- Resolution : 심볼릭 참조를 실제 참조로 변환하는 단계

**Initialization**
- static 필드의 명시적 초기화 및 static 블록 실행
- 클래스 초기화 메서드인 `<clinit>` 실행

### 6.3 Class Loader 종류
- Bootstrap Class Loader : Java 핵심 클래스 로드
- Platform Class Loader : 플랫폼 관련 클래스 로드
- Application Class Loader : 애플리케이션 클래스패스의 클래스 로드
- Custom Class Loader : 개발자가 직접 정의한 클래스 로더

### 6.4 Parent Delegation Model
- 클래스 로딩 요청을 상위 Class Loader에 먼저 위임하는 방식
- 상위 로더가 클래스를 찾지 못하면 하위 로더가 로딩 시도
- 핵심 Java 클래스의 중복 로딩 및 임의 대체를 방지하는 데 기여

---

## 7. GC (Garbage Collection)

### 7.1 GC 개념
- Heap 메모리에서 더 이상 사용되지 않는 객체를 자동으로 회수하는 기능
- 개발자가 직접 메모리를 해제하지 않아도 메모리 관리 가능
- GC 실행 과정에서 애플리케이션 성능에 영향을 줄 수 있음

### 7.2 GC 동작 원리
- GC Root에서 도달 가능한 객체를 살아 있는 객체로 판단
- GC Root에서 도달할 수 없는 객체를 회수 대상으로 판단
- GC Root에는 스택의 참조, static 참조 등이 포함

### 7.3 GC 주요 알고리즘

**Mark and Sweep**
- Mark : GC Root에서 도달 가능한 객체를 표시
- Sweep : 표시되지 않은 객체의 메모리를 회수
- 메모리 단편화 발생 가능

**Mark and Compact**
- 살아 있는 객체를 표시한 후 한쪽으로 이동
- 메모리 단편화를 줄이는 방식

**Copying**
- 살아 있는 객체를 다른 메모리 영역으로 복사
- 메모리 단편화 감소에 유리

### 7.4 Generational GC
- 객체의 생존 기간에 따라 메모리를 세대별로 관리하는 방식
- 대부분의 객체가 생성된 후 빠르게 소멸한다는 가설을 활용

**Young Generation**
- 새롭게 생성된 객체가 주로 저장되는 영역
- Eden, Survivor 영역으로 구성되는 전통적인 세대별 구조
- Minor GC가 주로 발생하는 영역

**Old Generation**
- Young Generation에서 오래 살아남은 객체가 이동하는 영역
- 장기간 생존하는 객체를 주로 관리

### 7.5 GC 종류
- Serial GC : 단일 GC 스레드를 사용하여 메모리 회수
- Parallel GC : 여러 GC 스레드를 사용하여 처리량 향상
- G1 GC : Heap을 여러 Region으로 나누어 관리하며 일시 정지 시간을 제어하는 데 중점
- ZGC : 매우 짧은 일시 정지 시간을 목표로 하는 저지연 GC

### 7.6 Stop The World (STW)
- GC 작업을 위해 애플리케이션 스레드의 실행이 일시 중단되는 현상
- STW 시간이 길어질 경우 응답 지연 발생 가능
- GC 종류와 설정에 따라 STW 발생 시점과 지속 시간이 달라짐

### 7.7 GC 모니터링
- GC 로그를 통해 수집 횟수, 소요 시간, 메모리 사용량 분석 가능
- JVisualVM, JConsole, JFR 등의 도구 활용 가능
- GC 빈도가 지나치게 높거나 일시 정지 시간이 길면 성능 저하 가능
- 메모리 누수 및 불필요한 객체 생성 여부 확인 필요
