# 3주차 - OS Memory / Virtual Memory (Backend)

## 1. JVM Runtime Data Area

JVM이 Java 프로그램을 실행할 때 사용하는 메모리 영역이다. 스레드마다 생성되는 영역과 모든 스레드가 공유하는 영역으로 나뉜다.

| 영역 | 공유 여부 | 역할 |
| --- | --- | --- |
| PC Register | 스레드별 | 현재 실행 중인 JVM 명령의 위치 |
| JVM Stack | 스레드별 | 메서드 호출 프레임, 지역 변수, 피연산자 스택 |
| Native Method Stack | 스레드별 | JNI 등 네이티브 메서드 실행 정보 |
| Heap | 공유 | 객체와 배열 저장 |
| Method Area (Metaspace) | 공유 | 클래스 메타데이터, static 필드, 런타임 상수 풀 |

`Metaspace`는 Java 8부터 기존 PermGen을 대체했다. JVM 힙이 아니라 네이티브 메모리를 사용하며, 클래스 로더가 더 이상 참조되지 않으면 관련 메타데이터도 회수될 수 있다.

## 2. JVM Stack과 Heap

메서드가 호출되면 해당 스레드의 JVM Stack에 stack frame이 쌓인다. 프레임에는 지역 변수, 매개변수, 중간 연산값, 반환 정보가 담긴다. 메서드가 끝나면 프레임도 함께 제거된다.

```java
public void createUser() {
    User user = new User("pooreum");
}
```

위 코드에서 `user`라는 참조 변수는 `createUser`의 stack frame에 있고, `new User(...)`로 생성한 객체는 Heap에 있다. 메서드가 끝나면 참조 변수는 사라지고 다른 참조도 없다면 객체는 GC 대상이 된다.

| 구분 | JVM Stack | Heap |
| --- | --- | --- |
| 생명주기 | 메서드 호출부터 반환까지 | 참조가 남아 있는 동안 |
| 관리 방식 | 프레임 단위 자동 정리 | GC가 도달 불가능한 객체 회수 |
| 공유 여부 | 스레드별 | 모든 스레드가 공유 |
| 대표 오류 | `StackOverflowError` | `OutOfMemoryError: Java heap space` |

깊은 재귀 호출은 스택 프레임을 계속 쌓아 `StackOverflowError`를 유발할 수 있다. 힙에 객체가 너무 많이 남거나 힙 크기가 부족하면 `OutOfMemoryError`가 발생한다.

## 3. Garbage Collection

GC는 GC Root에서 도달할 수 없는 객체를 회수한다. GC Root에는 실행 중인 스레드의 스택 참조, static 필드, JNI 참조 등이 포함된다. 단순히 지역 변수가 스코프를 벗어났다는 사실만으로 즉시 수거되는 것은 아니며, 실제 GC 시점은 JVM이 결정한다.

### 세대별 관리

대부분의 객체는 빨리 사라진다는 가설을 바탕으로 힙을 세대별로 나눈다.

| 영역 | 특징 | 주로 발생하는 GC |
| --- | --- | --- |
| Young Generation | 새 객체가 생성되는 Eden과 Survivor 영역 | Minor GC |
| Old Generation | 여러 Minor GC를 살아남은 객체가 이동 | Major/Full GC |

Minor GC는 보통 짧지만 객체 생성량이 과도하면 자주 발생할 수 있다. Old Generation까지 영향을 주는 Full GC는 애플리케이션 중지 시간이 길어질 수 있으므로, 장애 분석 시 GC 로그와 정지 시간을 함께 확인한다.

### Stop-The-World

GC의 일부 단계에서는 애플리케이션 스레드가 멈춘다. 이를 Stop-The-World라고 한다. GC 종류와 JVM 버전에 따라 정지 시간의 특성은 달라지므로, 단순히 GC 알고리즘을 바꾸기보다 실제 요청 지연과 GC 로그를 먼저 측정해야 한다.

## 4. Backend에서 확인할 점

- 큰 컬렉션이나 응답 객체를 static 필드·캐시에 무제한 보관하지 않는다.
- 대량 데이터를 처리할 때는 한 번에 모두 적재하기보다 페이지네이션 또는 스트리밍을 검토한다.
- `OutOfMemoryError` 발생 시 힙 덤프로 어떤 객체가 메모리를 점유하는지 먼저 확인한다.
- GC 옵션과 힙 크기 조정은 측정 결과가 있을 때만 수행한다.
