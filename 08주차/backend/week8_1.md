# 8주차 - Java / JVM 개념 정리

## 목차
- [키워드 연관관계](#키워드-연관관계)
- [JVM/JDK/JRE](#jvmjdkjre)
- [Generic](#generic)
- [Collection](#collection)
- [Exception](#exception)
- [JVM Architecture](#jvm-architecture)
- [Class Loader](#class-loader)
- [GC](#gc)

---

## 키워드 연관관계

```
JDK ⊃ JRE ⊃ JVM
         │
         ├─ 소스(.java) --javac--> 바이트코드(.class)
         │
         └─ JVM Architecture
             ├─ Class Loader : .class를 Runtime Data Area에 로딩
             ├─ Runtime Data Area : Method / Heap / Stack / PC / Native Stack
             │                       └─ Heap ──관리──> GC
             └─ Execution Engine : 인터프리터, JIT, GC

언어/라이브러리 레벨: Generic ↔ Collection (타입 안정성), Exception (오류 처리)
```

| 연결 | 관계 설명 |
|---|---|
| JDK ⊃ JRE ⊃ JVM | JDK는 개발 도구(javac 등) + JRE, JRE는 실행 환경(표준 라이브러리) + JVM을 포함하는 포함 관계 |
| JDK(javac) → JVM | javac가 `.java`를 JVM이 실행하는 바이트코드(`.class`)로 컴파일 → 플랫폼 독립성 |
| JVM → JVM Architecture | JVM의 내부 구성(Class Loader, Runtime Data Area, Execution Engine)을 설명하는 것이 JVM Architecture |
| Class Loader → Runtime Data Area | Class Loader가 `.class`를 읽어 클래스 메타데이터를 Method Area에 적재하고, 이후 객체는 Heap에 생성 |
| Heap → GC | GC는 Heap에서 더 이상 참조되지 않는 객체를 찾아 메모리를 회수 |
| Generic ↔ Collection | Collection은 `List<String>`처럼 Generic으로 요소 타입을 컴파일 시점에 지정해 타입 안정성 확보 |
| Generic → JVM | Generic은 컴파일 후 타입 정보가 제거(Type Erasure)되어 JVM은 제네릭 타입을 알지 못함 |
| Collection → Heap/GC | Collection의 객체들은 Heap에 저장되며, 참조가 끊기면 GC 대상 |
| Exception ↔ JVM | 오류 발생 시 JVM이 예외 객체를 생성해 Call Stack을 거슬러 올라가며 처리 코드를 탐색 |

---

## JVM/JDK/JRE

- **JVM(Java Virtual Machine)**: 바이트코드를 OS에 맞게 실행하는 가상 머신. OS에 독립적인 실행(Write Once, Run Anywhere)을 가능하게 함
- **JRE(Java Runtime Environment)**: JVM + 표준 클래스 라이브러리. 자바 프로그램을 **실행**하는 환경
- **JDK(Java Development Kit)**: JRE + 컴파일러(`javac`), 디버거 등 **개발 도구**

---

## Generic

- 클래스/메서드에서 사용할 **타입을 파라미터로 받아 컴파일 시점에 지정**하는 기능
- 효과: 형 변환 제거, 잘못된 타입을 컴파일 시점에 검출
- **Type Erasure**: 컴파일 후 제네릭 타입 정보가 제거되고 런타임에는 `Object` 등으로 동작

---

## Collection

- 데이터 그룹을 저장·관리하는 자료구조 프레임워크 (`java.util`)

| 인터페이스 | 특징 | 구현체 |
|---|---|---|
| List | 순서 유지, 중복 허용 | ArrayList, LinkedList |
| Set | 중복 불허 | HashSet, TreeSet |
| Map | Key-Value 쌍 (Collection 인터페이스는 아님) | HashMap, TreeMap |
| Queue | FIFO | LinkedList, PriorityQueue |

---

## Exception

- 프로그램 실행 중 발생하는 비정상 상황을 객체로 표현하고 처리하는 메커니즘 (`try-catch-finally`, `throw`, `throws`)

## JVM Architecture

- **Class Loader**: 클래스 로딩
- **Runtime Data Area**: 실행 중 사용하는 메모리 영역
  - Method Area(클래스 메타데이터, static), Heap(객체, 모든 스레드 공유), Stack(스레드별 메서드 호출 프레임/지역변수), PC Register, Native Method Stack
- **Execution Engine**: 바이트코드를 실행. Interpreter + **JIT 컴파일러**(자주 쓰는 코드를 기계어로 컴파일해 캐싱) + GC

---

## Class Loader

- 클래스를 **런타임에 동적으로 로딩**하는 JVM 구성요소
- 단계: **Loading**(`.class` 읽기) → **Linking**(Verify 검증 / Prepare 메모리 할당 및 기본값 / Resolve 심볼릭 참조 해석) → **Initialization**(static 초기화)
---

## GC

- Heap에서 **더 이상 참조되지 않는 객체를 자동으로 회수**하는 메모리 관리 기능

- **Stop-The-World**: GC 수행 중 애플리케이션 스레드가 일시 정지되는 현상. GC 튜닝의 핵심 대상

