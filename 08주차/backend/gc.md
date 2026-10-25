# GC

## GC란

GC(Garbage Collection)는 **Heap에서 더 이상 사용되지 않는 객체를 찾아 메모리를 자동으로 회수**하는 JVM의 기능이다.

- C/C++는 `malloc`/`free`로 개발자가 직접 메모리를 해제해야 해서, 해제를 잊으면 메모리 누수, 두 번 해제하면 오류가 난다.
- 자바는 GC가 대신 처리하므로 이런 실수가 줄지만, **언제 GC가 실행될지 개발자가 제어할 수 없고** 실행 중에 애플리케이션이 멈추는 비용이 생긴다.
- `System.gc()`는 GC를 "요청"할 뿐 실행을 보장하지 않으며, 전체 GC를 유발할 수 있어 쓰지 않는다.

## Reachability와 GC Root

GC는 **"참조할 수 있는가(Reachability)"** 로 살아있는 객체를 판단한다.

- **GC Root**에서 출발해 참조를 따라갈 수 있는 객체는 살아있는(reachable) 객체다.
- 어떤 GC Root에서도 닿지 않는 객체는 쓰레기(unreachable)로 보고 회수한다.

GC Root가 되는 것

- JVM Stack의 지역 변수, 매개변수
- Method Area의 static 변수
- JNI로 참조되는 객체
- 활성 상태의 스레드

```java
void method() {
    User a = new User();   // a → User1 (reachable)
    a = new User();        // a → User2, User1은 아무도 참조하지 않음 → GC 대상
}                          // 메서드 종료 → 스택 프레임 제거 → User2도 GC 대상
```

- 참조 횟수를 세는 방식(Reference Counting)이 아니라 Root부터 추적하므로, **서로만 참조하는 순환 참조 객체도 회수**된다.
- 그래도 메모리 누수는 생길 수 있다. 다 쓴 객체를 static 컬렉션, 캐시, 리스너 목록 등에서 계속 참조하고 있으면 GC가 회수하지 못한다.

참조 강도를 조절하는 `java.lang.ref` 클래스도 있다.

| 참조 | GC 동작 | 용도 |
| --- | --- | --- |
| Strong | 참조가 있는 한 회수 안 함 | 일반적인 참조 |
| Soft | 메모리가 부족할 때 회수 | 메모리 민감한 캐시 |
| Weak | 다음 GC 때 회수 | `WeakHashMap`, 메타데이터 매핑 |
| Phantom | 회수 직전 알림 | 자원 정리 (`Cleaner`) |

## Stop-The-World

GC를 실행하려면 객체 참조 관계가 바뀌지 않도록 **GC 스레드를 제외한 모든 애플리케이션 스레드를 멈춘다.** 이를 Stop-The-World(STW)라고 한다.

- STW 동안 요청을 처리하지 못하므로 응답 지연(latency)이 생긴다.
- **GC 튜닝은 대부분 이 STW 시간을 줄이는 것**이다.
- 최신 GC(G1, ZGC)는 대부분의 작업을 애플리케이션 스레드와 **동시에(concurrent)** 수행해 STW를 줄인다.

## Generational GC

**Weak Generational Hypothesis** 라는 관찰을 바탕으로 Heap을 세대로 나눈다.

1. 대부분의 객체는 금방 쓰레기가 된다. (요청 처리 중 만든 DTO, 임시 객체 등)
2. 오래 살아남은 객체가 젊은 객체를 참조하는 경우는 드물다.

```text
Heap
 ├─ Young Generation
 │   ├─ Eden
 │   ├─ Survivor 0
 │   └─ Survivor 1
 └─ Old Generation
```

**Minor GC (Young 영역)**

1. 새 객체는 **Eden**에 할당된다.
2. Eden이 가득 차면 Minor GC가 일어나, 살아남은 객체를 비어있는 **Survivor** 영역으로 옮긴다.
3. 이후 Minor GC마다 Eden과 사용 중인 Survivor의 생존 객체를 다른 Survivor로 옮긴다. 두 Survivor 중 하나는 항상 비어 있다.
4. 살아남을 때마다 객체의 age가 늘고, 일정 횟수(age threshold)를 넘기면 **Old 영역으로 승격(promotion)** 된다.

- Young 영역은 작고 대부분 객체가 죽어 있으므로 Minor GC는 **빠르다.**

**Major GC / Full GC (Old 영역)**

- Old 영역이 가득 차면 발생한다.
- Old 영역은 크고 살아있는 객체가 많아서 **훨씬 오래 걸리고 STW도 길다.**
- 운영 중 Full GC가 자주 일어난다면 메모리 누수나 Heap 크기 부족을 의심한다.

> Old 영역 객체가 Young 객체를 참조하는 경우를 위해 **Card Table**에 그 정보를 기록해 둔다. Minor GC 때 Old 영역 전체를 훑지 않고 Card Table만 확인하면 된다.

## GC 알고리즘

**Mark and Sweep (and Compact)**

1. **Mark**: GC Root부터 참조를 따라가며 살아있는 객체에 표시한다.
2. **Sweep**: 표시되지 않은 객체를 제거한다.
3. **Compact**: 살아남은 객체를 한쪽으로 모아 **메모리 단편화**를 없앤다. 그래야 큰 객체를 연속된 공간에 할당할 수 있다.

Young 영역의 Survivor 복사 방식은 살아남은 객체만 복사하므로 자연스럽게 압축 효과가 있다(Copying).

## GC 종류

| GC | 특징 | 적합한 환경 |
| --- | --- | --- |
| Serial GC | GC를 **단일 스레드**로 수행 | 작은 Heap, CPU 코어 1개 |
| Parallel GC | Young/Old GC를 **여러 스레드**로 수행. 처리량(throughput) 중시 | 배치 작업 (Java 8 기본) |
| G1 GC | Heap을 같은 크기의 **Region**으로 나눠 관리. 쓰레기가 많은 Region부터 회수. 목표 정지 시간 설정 가능 | 대부분의 서버 (**Java 9+ 기본**) |
| ZGC | 대부분 작업을 동시에 처리해 **STW를 1ms 수준**으로 유지. TB 단위 Heap도 지원 | 지연 시간이 중요한 대용량 서비스 |

- CMS GC는 STW를 줄이려고 나왔지만 단편화 문제 등으로 Java 9에서 deprecated, Java 14에서 제거되었다.
- G1은 Region마다 Eden/Survivor/Old 역할을 동적으로 부여한다. 세대 구분은 유지하지만 물리적으로 연속될 필요가 없다.
- ZGC는 Java 15에서 정식 기능이 되었고, Java 21에서 세대 구분 방식(Generational ZGC)이 추가되어 Java 23부터 기본 모드가 되었다.

```bash
java -XX:+UseG1GC -Xms2g -Xmx2g -XX:MaxGCPauseMillis=200 -jar app.jar
java -XX:+UseZGC -jar app.jar
```

## GC 모니터링과 튜닝

**튜닝 전에 먼저 할 것**

- GC 튜닝은 **최후의 수단**이다. 대부분은 불필요한 객체 생성 줄이기, 메모리 누수 수정, 적절한 Heap 크기 설정으로 해결된다.
- 컨테이너 환경에서는 `-Xmx` 대신 `-XX:MaxRAMPercentage=75.0`처럼 컨테이너 메모리 대비 비율로 지정하는 경우가 많다.
- 서버에서는 `-Xms`와 `-Xmx`를 같게 설정해 Heap 크기 조정 비용을 없애기도 한다.

**모니터링 도구**

```bash
java -Xlog:gc*:file=gc.log -jar app.jar   # GC 로그 남기기 (Java 9+)
jstat -gcutil <pid> 1000                  # 1초마다 영역별 사용률, GC 횟수/시간
jcmd <pid> GC.heap_info                   # Heap 정보
jcmd <pid> GC.heap_dump heap.hprof        # Heap Dump (메모리 누수 분석)
```

- `-XX:+HeapDumpOnOutOfMemoryError`를 켜두면 OOM 발생 시 자동으로 Heap Dump를 남겨 원인을 분석할 수 있다. (Eclipse MAT, VisualVM 등으로 분석)
- Spring Boot Actuator + Micrometer로 `jvm.gc.pause`, `jvm.memory.used` 같은 지표를 Prometheus/Grafana에서 볼 수 있다.

볼 지표

- GC 빈도와 STW 시간 (특히 Full GC)
- GC 후 Old 영역 사용량. GC를 해도 계속 우상향하면 **메모리 누수**를 의심한다.
