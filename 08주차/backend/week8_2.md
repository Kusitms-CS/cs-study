# 8주차 - Monitoring / 성능 병목 진단 

## 목차
- [의문점](#의문점)
- [키워드 분류](#키워드-분류)
- [키워드 연관관계](#키워드-연관관계)
- [Actuator](#actuator)
- [Micrometer](#micrometer)
- [MDC / Trace ID](#mdc--trace-id)
- [Tomcat Thread Pool](#tomcat-thread-pool)
- [HikariCP](#hikaricp)
- [Transaction과 외부 호출](#transaction과-외부-호출)
- [Timeout](#timeout)
- [GC (Stop-the-World)](#gc-stop-the-world)

---

## 의문점

> **대규모 트래픽을 받는 웹 애플리케이션은 무엇을 보고 이상을 알아챌까?**

- 많은 이용자 → **많은 요청과 로그**가 발생하므로 사람이 직접 볼 수 없고, 수집·집계된 수치가 필요
- 요청은 `WEB → WAS → DB`로 흐르는데 **애플리케이션 내부 상태는 밖에서 보이지 않음** → 내부를 드러내는 장치가 필요
- 사용자가 많을 때는 CPU보다 **애플리케이션 내부 자원(스레드, 커넥션 등)이 먼저 한계**에 도달

---

## 키워드 분류

| 관점 | 핵심 질문 | 관련 키워드 |
|---|---|---|
| 많은 이용자 | 요청과 로그가 많아도 추적할 수 있는가 | Tomcat Thread Pool, HikariCP, Micrometer, MDC·Trace ID |
| 웹 애플리케이션 | 안쪽(WAS ~ DB)을 어떻게 볼 것인가 | Actuator, Tomcat Thread Pool, Transaction과 외부 호출 |
| 모니터링 | 수집 → 감지 → 원인 추적 | Actuator, Micrometer, MDC·Trace ID |
| 성능 | 느려졌을 때 어디서 막혔는가 | Tomcat Thread Pool, HikariCP, Transaction과 외부 호출, GC (Stop-the-World) |

---

## 키워드 연관관계

```
모니터링 (수집 → 감지 → 원인 추적)
├─ Actuator ──노출──> 애플리케이션 상태/지표 엔드포인트
├─ Micrometer ──수집──> 지표(Metric)를 모니터링 시스템으로 전달
└─ MDC·Trace ID ──추적──> 요청 단위로 로그를 연결

성능 (병목 지점)
요청 ──> Tomcat Thread Pool ──> 비즈니스 로직 ──> HikariCP ──> DB
              │                      │
              │                      └─ Transaction 안의 외부 호출 ──> 커넥션 점유 시간 증가
              └─ GC Stop-the-World ──> 모든 스레드 일시 정지

Timeout : 위 대기 구간에 상한을 두어 장애 전파 차단
```

| 연결 | 관계 설명 |
|---|---|
| Actuator → Micrometer | Actuator의 `/metrics`, `/prometheus` 엔드포인트가 Micrometer가 수집한 지표를 노출 |
| Micrometer → Tomcat Thread Pool, HikariCP, GC | Micrometer가 스레드 수, 커넥션 수, GC 시간 등의 지표를 자동으로 수집하는 대상 |
| Tomcat Thread Pool → HikariCP | 요청 스레드가 DB 작업 시 커넥션을 얻어야 하므로, 커넥션이 부족하면 스레드가 대기하고 스레드 풀까지 고갈될 수 있음 |
| Transaction과 외부 호출 → HikariCP | 트랜잭션 안에서 외부 API를 호출하면 응답이 올 때까지 DB 커넥션을 점유해 커넥션 풀이 고갈될 수 있음 |
| Timeout → Transaction과 외부 호출 | 외부 호출에 Timeout이 없으면 응답 지연이 무한 대기로 이어져 스레드와 커넥션을 계속 점유 |
| GC → Tomcat Thread Pool | Stop-the-World 동안 모든 요청 스레드가 멈춰 응답 지연이 발생하고 요청이 큐에 쌓임 |
| MDC·Trace ID → 원인 추적 | 동시에 많은 요청이 섞인 로그에서 하나의 요청 흐름을 식별할 수 있게 하여, 지표로 감지한 이상의 원인을 로그로 좁혀감 |

---

## Actuator

- **Spring Boot Actuator**: 실행 중인 애플리케이션의 상태와 지표를 **HTTP 엔드포인트로 노출**하는 모듈
- 주요 엔드포인트

| 엔드포인트 | 내용 |
|---|---|
| `/actuator/health` | 애플리케이션 및 DB, 디스크 등 상태(UP/DOWN) |
| `/actuator/metrics` | 수집된 지표 목록 및 값 |
| `/actuator/info` | 빌드/버전 정보 |
| `/actuator/prometheus` | Prometheus 형식의 지표 |

- 기본은 `health` 정도만 외부에 노출되며, 나머지는 `management.endpoints.web.exposure.include`로 설정
- 민감 정보가 포함되므로 **외부 접근 제한(인증, 별도 포트)** 필요

---

## Micrometer

- JVM 애플리케이션의 **지표(Metric) 수집 API를 추상화한 라이브러리** (Spring Boot 기본 포함)
- 벤더 중립적: 한 번 계측하면 Prometheus, Datadog 등 **여러 모니터링 시스템으로 전송** 가능 (로깅의 SLF4J와 유사한 위치)
- 기본 지표 유형

| 유형 | 설명 | 예 |
|---|---|---|
| Counter | 증가만 하는 누적 값 | 요청 수, 에러 수 |
| Gauge | 증감하는 현재 값 | 활성 스레드 수, 커넥션 수 |
| Timer | 소요 시간과 호출 횟수 | API 응답 시간 |

- 흐름: 애플리케이션 → Micrometer 수집 → Prometheus 저장 → Grafana 시각화/알림

---

## MDC / Trace ID

- **MDC(Mapped Diagnostic Context)**: 스레드 단위로 키-값 데이터를 저장해 **모든 로그에 자동으로 포함**시키는 기능 (SLF4J/Logback)
- **Trace ID**: 하나의 요청(또는 트랜잭션)을 식별하는 고유 ID. 요청 진입 시 생성해 MDC에 넣으면 해당 요청의 로그가 모두 같은 ID를 가짐
- 필요성: 동시에 수많은 요청 로그가 뒤섞이므로, Trace ID로 **한 요청의 흐름을 필터링**
- 분산 환경(MSA): 서비스 간 호출에 Trace ID를 헤더로 전달해 전체 호출 경로를 추적 (분산 추적: Micrometer Tracing, Zipkin 등)
- 주의: 스레드 풀에서 스레드가 재사용되므로 요청 종료 시 `MDC.clear()` 필요, 비동기 처리 시 MDC가 전파되지 않음

---

## Tomcat Thread Pool

- Tomcat(WAS)이 요청을 처리하기 위해 **미리 만들어 두고 재사용하는 스레드 집합** (Thread-per-Request 모델)
- 주요 설정

| 설정 | 기본값 | 설명 |
|---|---|---|
| `server.tomcat.threads.max` | 200 | 최대 스레드 수 |
| `server.tomcat.threads.min-spare` | 10 | 최소 유지 스레드 수 |
| `server.tomcat.accept-count` | 100 | 모든 스레드가 사용 중일 때 대기 큐 크기 |
| `server.tomcat.max-connections` | 8192 | 동시에 유지하는 최대 커넥션 수 |

- 스레드가 모두 사용 중이면 요청은 큐에서 대기, 큐도 가득 차면 연결 거부
- 고갈 징후: 활성 스레드 수가 최대치에 근접, 응답 시간 증가 (`tomcat.threads.busy` 지표)
- 스레드가 DB/외부 호출에서 블로킹되면 CPU는 여유로워도 새 요청을 처리하지 못함

---

## HikariCP

- Spring Boot의 기본 **JDBC 커넥션 풀**. DB 커넥션을 미리 만들어 두고 재사용
- 필요성: 커넥션 생성은 비용이 크므로 매 요청마다 생성하지 않고 풀에서 **빌려 쓰고 반납**
- 주요 설정

| 설정 | 설명 |
|---|---|
| `maximum-pool-size` | 최대 커넥션 수 (기본 10) |
| `connection-timeout` | 커넥션을 얻기 위해 대기하는 최대 시간 (기본 30초), 초과 시 예외 |
| `max-lifetime` | 커넥션의 최대 수명 |

- 풀의 크기는 클수록 좋은 게 아님: DB가 감당할 수 있는 동시 커넥션 수가 한계이므로 DB 사양에 맞게 설정
- 모니터링 지표: `hikaricp.connections.active`(사용 중), `pending`(대기 중 스레드), `timeout`
- `pending`이 증가하면 커넥션 풀이 병목임을 의미

---

## Transaction과 외부 호출

- `@Transactional` 메서드는 **트랜잭션 시작 시점부터 종료(커밋/롤백)까지 DB 커넥션을 점유**
- 트랜잭션 안에서 **외부 API 호출, 파일 I/O 등 느린 작업**을 하면 그 시간 동안 커넥션을 반환하지 못함

```
@Transactional 시작 (커넥션 획득)
  ├─ DB 조회/수정
  ├─ 외부 API 호출  ← 응답 대기 동안 커넥션 점유
  └─ 커밋 (커넥션 반납)
```

- 결과: 외부 API 지연 → 커넥션 점유 시간 증가 → HikariCP 풀 고갈 → 스레드 대기 → 전체 장애로 전파
- 해결: 외부 호출은 **트랜잭션 밖으로 분리**하고, 트랜잭션 범위를 최소화, 외부 호출에 Timeout 설정

---

## Timeout

- 작업이 **일정 시간 내에 끝나지 않으면 포기**하도록 설정하는 대기 시간의 상한
- Timeout이 없으면 느린 의존 대상 하나가 스레드/커넥션을 무한정 붙잡아 **장애가 전체로 전파**됨

| 종류 | 설명 |
|---|---|
| Connection Timeout | 연결을 맺기까지 기다리는 시간 |
| Read(Response) Timeout | 연결 후 응답을 기다리는 시간 |
| HikariCP `connection-timeout` | 풀에서 커넥션을 얻기까지 기다리는 시간 |
| Transaction Timeout | 트랜잭션의 최대 수행 시간 |

- 설정 원칙: 호출 체인에서 **바깥(상위) Timeout이 안쪽(하위) Timeout보다 길어야** 함
- 관련 패턴: Retry, Circuit Breaker (장애 전파 차단)

---

## GC (Stop-the-World)

- **Stop-the-World(STW)**: GC 수행 중 JVM이 **모든 애플리케이션 스레드를 일시 정지**하는 현상
- STW 동안 요청 처리가 멈추므로 응답 지연(Latency Spike)이 발생하고, 요청이 쌓이면 스레드 풀 고갈로 이어짐
- 징후: 응답 시간이 주기적으로 튐, 힙 사용량이 톱니 모양, Full GC 빈번, 심하면 OOM
- 모니터링 지표: `jvm.gc.pause`(GC 일시 정지 시간), `jvm.memory.used`(힙 사용량)
- 대응: 메모리 누수 제거, 힙 크기/GC 알고리즘 조정(G1, ZGC 등으로 STW 시간 감소)

---


