# JVM Architecture / Class Loader

## JVM 구성 요소

```text
            .class 파일
                │
        ┌───────▼────────┐
        │  Class Loader  │  클래스를 찾아 메모리에 로드
        └───────┬────────┘
┌───────────────▼─────────────────────────────────────────┐
│                    Runtime Data Area                    │
│  [공유]                     [스레드마다 생성]            │
│  - Method Area (Metaspace)  - JVM Stack                 │
│  - Heap                     - PC Register               │
│                             - Native Method Stack       │
└───────────────┬─────────────────────────────────────────┘
        ┌───────▼──────────┐       ┌──────────────┐
        │ Execution Engine │◀─────▶│ JNI / Native │
        │ Interpreter, JIT,│       │   Library    │
        │ GC               │       └──────────────┘
        └──────────────────┘
```

- **Class Loader**: `.class` 파일을 찾아 JVM 메모리에 올린다.
- **Runtime Data Area**: 프로그램 실행 중 사용하는 메모리 영역
- **Execution Engine**: 바이트코드를 실행한다. 인터프리터, JIT 컴파일러, GC가 여기 속한다.
- **JNI(Java Native Interface)**: C/C++ 같은 네이티브 코드를 호출하는 인터페이스

## Runtime Data Area

[3주차](../../03주차/memory-structure.md)에서 정리한 프로세스 메모리 구조(code, data, heap, stack)를 JVM이 자기 방식으로 나눈 것이라고 볼 수 있다.

**모든 스레드가 공유하는 영역**

| 영역 | 저장하는 것 |
| --- | --- |
| Method Area | 클래스 메타데이터(클래스 이름, 필드/메서드 정보, 바이트코드), static 변수, Runtime Constant Pool |
| Heap | `new`로 생성한 **객체와 배열**. GC의 대상 |

- Method Area는 JVM 명세상의 개념이고, HotSpot JVM에서는 Java 8부터 **Metaspace**로 구현한다.
  - Java 7까지는 Heap 안의 **PermGen**에 있어 크기가 고정되어 `OutOfMemoryError: PermGen space`가 자주 났다.
  - Java 8부터는 **네이티브 메모리**를 쓰는 Metaspace로 바뀌어 필요에 따라 늘어난다. (`-XX:MaxMetaspaceSize`로 제한 가능)
  - 이때 static 변수와 String Constant Pool은 Heap으로 옮겨졌다. (String Pool은 Java 7에서 먼저 이동)

**스레드마다 생성되는 영역**

| 영역 | 저장하는 것 |
| --- | --- |
| JVM Stack | 메서드 호출마다 **스택 프레임** 생성 (지역 변수, 매개변수, 연산 중간값, 반환 주소) |
| PC Register | 현재 실행 중인 바이트코드 명령어 주소 |
| Native Method Stack | 네이티브 메서드 실행용 스택 |

```java
void method() {
    int count = 10;              // 기본 타입 값 → Stack
    User user = new User();      // 참조 변수 user → Stack, User 객체 → Heap
}
```

- 메서드가 끝나면 스택 프레임이 사라지고, Heap의 `User` 객체는 참조가 없어지면 GC 대상이 된다.
- 재귀 호출이 너무 깊으면 스택 프레임이 쌓여 `StackOverflowError`, Heap이 부족하면 `OutOfMemoryError: Java heap space`가 난다.
- Stack은 스레드마다 따로 있으므로 **지역 변수는 스레드 안전**하고, Heap은 공유되므로 여러 스레드가 같은 객체에 접근하면 동기화가 필요하다.

## Execution Engine과 JIT

- **인터프리터**: 바이트코드를 한 명령어씩 해석해 실행한다. 시작은 빠르지만 반복 실행 시 느리다.
- **JIT(Just-In-Time) 컴파일러**: 자주 호출되는 메서드나 반복문(Hot Spot)을 감지해 **기계어로 컴파일하고 캐시**한다. 이후에는 기계어를 바로 실행한다.
  - HotSpot JVM은 **C1**(빠르게 가볍게 최적화)과 **C2**(오래 걸리지만 강하게 최적화) 컴파일러를 단계적으로 사용한다. (Tiered Compilation)
  - 인라이닝, 탈출 분석(escape analysis) 등의 최적화를 런타임 정보를 바탕으로 수행한다.
- 서버를 막 띄웠을 때는 JIT 컴파일이 안 된 상태라 느리다. 그래서 배포 직후 트래픽을 받기 전에 **웜업(warm-up)** 요청을 보내기도 한다.

## Class Loader

### 클래스 로딩 과정

클래스는 JVM 시작 시 한꺼번에 올라가지 않고, **처음 사용될 때 동적으로(lazy) 로딩**된다.

```text
Loading → Linking (Verification → Preparation → Resolution) → Initialization
```

1. **Loading**: `.class` 파일을 찾아 바이너리를 읽고 Method Area에 클래스 정보를 저장한다. Heap에 해당 클래스의 `Class` 객체를 만든다.
2. **Linking**
   - **Verification**: 바이트코드가 JVM 명세에 맞고 안전한지 검증한다.
   - **Preparation**: static 변수를 위한 메모리를 할당하고 **기본값**(0, null, false)으로 초기화한다.
   - **Resolution**: 심볼릭 참조(클래스, 메서드 이름)를 실제 메모리 주소 참조로 바꾼다.
3. **Initialization**: static 변수에 **코드에 작성한 값을 대입**하고 static 블록을 실행한다.

```java
class Config {
    static int timeout = 30;          // Preparation: 0 → Initialization: 30
    static { System.out.println("init"); }
}
```

초기화는 다음처럼 클래스를 **실제로 사용하는 시점**에 일어난다.

- 인스턴스 생성(`new`), static 메서드 호출, static 필드 접근(상수 제외), 리플렉션, 하위 클래스 초기화

이 특성을 이용한 것이 **Holder 패턴 싱글톤**이다. 내부 클래스는 `getInstance()`가 처음 호출될 때 로딩되고, JVM이 클래스 초기화를 스레드 안전하게 보장하므로 `synchronized` 없이 지연 초기화가 된다.

```java
public class Singleton {
    private Singleton() {}
    private static class Holder {
        private static final Singleton INSTANCE = new Singleton();
    }
    public static Singleton getInstance() { return Holder.INSTANCE; }
}
```

### 클래스 로더 계층과 위임 모델

| 클래스 로더 (Java 9+) | 로딩 대상 |
| --- | --- |
| Bootstrap ClassLoader | JVM 핵심 클래스 (`java.lang.*` 등 `java.base` 모듈). 네이티브 코드로 구현 |
| Platform ClassLoader | Java SE 플랫폼 모듈 (Java 8까지는 Extension ClassLoader) |
| Application (System) ClassLoader | 애플리케이션 classpath의 클래스 (우리가 작성한 코드, 라이브러리) |

**위임 모델 (Parent Delegation)**

클래스 로딩 요청이 오면 **먼저 부모에게 위임**하고, 부모가 못 찾으면 자신이 찾는다.

```text
Application → Platform → Bootstrap 순으로 위임
              (못 찾으면) ← ← 순으로 내려오며 직접 탐색
```

- **보안**: 누군가 `java.lang.String`이라는 악성 클래스를 만들어도, Bootstrap이 먼저 진짜 `String`을 로딩하므로 덮어쓸 수 없다.
- **중복 방지**: 부모가 이미 로딩한 클래스는 다시 로딩하지 않는다.
- **가시성**: 자식 로더는 부모가 로딩한 클래스를 볼 수 있지만, 부모는 자식이 로딩한 클래스를 볼 수 없다.
- 클래스는 **(클래스 로더 + 클래스 이름)** 으로 식별된다. 같은 이름이라도 로더가 다르면 다른 클래스로 취급되어, 서로 캐스팅하면 `ClassCastException`이 난다.

Tomcat 같은 WAS는 웹 애플리케이션마다 별도의 클래스 로더를 두고, 각 애플리케이션 내부 클래스를 먼저 찾도록 위임 순서를 일부 바꿔서 애플리케이션 간 라이브러리 충돌을 막는다.

자주 보는 에러

- `ClassNotFoundException`: `Class.forName()` 등으로 **런타임에 동적으로** 로딩하려는 클래스를 classpath에서 찾지 못함 (Checked Exception)
- `NoClassDefFoundError`: **컴파일 때는 있었는데** 런타임에 클래스가 없거나, static 초기화에 실패함 (Error). 의존성 버전 충돌, 빌드 누락 시 자주 발생한다.
