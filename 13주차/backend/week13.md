## 키워드
- MQ
- Sync/Async
- MQ 심화: 전달 보장
  - Queue vs Log
  - At-most-once / At-least-once / Exactly-once
  - 멱등성 (Idempotency)
  - ACK와 커밋 타이밍
  - Kafka Idempotent Producer / Kafka 트랜잭션


---

## 5. Sync / Async

> **한 줄 정의:** 작업 결과를 **기다렸다가 다음으로 넘어가면 동기**, **기다리지 않고 넘어가면 비동기**

**해결하는 문제:** 하나의 요청 안에서 느린 작업(외부 API, 메일 발송 등) 때문에 전체 응답이 느려지는 문제

**비교**

| | Sync | Async |
|---|---|---|
| 흐름 | A가 끝나야 B 시작 | A를 맡겨 두고 B 진행 |
| 장점 | 흐름이 단순, 디버깅 쉬움, 결과를 즉시 앎 | 빠른 응답, 장애 격리, 처리량 증가 |
| 단점 | 느린 작업 하나가 전체를 막음, 장애 전파 | 복잡도 증가, 결과 추적과 디버깅이 어려움, 최종 일관성 |

**⚠️ 자주 헷갈리는 개념: Sync/Async vs Blocking/Non-blocking**
- **Sync / Async:** 결과를 **누가, 언제 신경 쓰는가**
  - 호출한 쪽이 결과를 직접 챙기면 Sync, 콜백이나 이벤트로 받으면 Async
- **Blocking / Non-blocking:** 호출한 쪽이 **제어권을 바로 돌려받는가**
  - 대기하면 Blocking, 바로 반환되면 Non-blocking

**무엇을 비동기로 뺄지 판단하는 기준**
- 사용자가 결과를 **즉시** 알아야 하는가? → 동기 (결제 승인, 재고 차감)
- 조금 늦어도 괜찮은가? → 비동기 (알림, 메일, 포인트 적립, 통계)

**구현 방법**
- `@Async`, `CompletableFuture`, 이벤트 리스너
- 단, 서버가 죽으면 작업이 유실될 수 있으므로 **신뢰성이 필요하면 MQ**를 쓴다.

**면접 포인트**
- Q. 모든 걸 Async로 바꾸면 안 되는 이유는?
  - 결과를 즉시 보장해야 하는 작업이 있고, 실패 처리와 순서 보장, 디버깅 비용이 커지기 때문이다.

---

## 6. MQ (Message Queue)

> **한 줄 정의:** 서비스 사이에서 **메시지를 잠시 보관했다가 전달**해 주는 중간 저장소. 비동기 통신을 안정적으로 구현하는 도구

**해결하는 문제**
1. **비동기 처리:** 결제 후 알림·메일을 큐에 넣고 즉시 응답
2. **서비스 간 결합도 감소:** 주문 서비스는 알림 서비스가 살아 있는지 몰라도 된다.
3. **트래픽 완충(버퍼):** 순간 폭주를 큐에 쌓아 두고 처리 가능한 속도로 소비 (티켓팅 대기열)
4. **장애 격리:** 소비자가 죽어도 메시지는 큐에 남아 있다.

**구조**
```
Producer(발행) → Queue / Topic → Consumer(소비)
```

**대표 도구 비교**

| | RabbitMQ | Kafka |
|---|---|---|
| 모델 | 메시지 브로커 (Push) | 분산 로그 (Pull) |
| 메시지 보관 | 소비하면 삭제 | 설정한 기간 동안 보관, **재처리 가능** |
| 강점 | 복잡한 라우팅, 작업 큐 | 대용량 처리, 이벤트 스트리밍 |
| 적합한 용도 | 메일 발송, 작업 분배 | 로그 수집, 이벤트 기반 MSA |

> 이 차이가 왜 생기는지는  Queue vs Log 참고

**트레이드오프 / 새로 생기는 문제**
- **메시지 중복:** 대부분 "최소 한 번(at-least-once)" 전달이라 같은 메시지가 두 번 올 수 있다 → **멱등성** 처리 (주문 ID로 중복 체크) → *7-2, 7-3*
- **메시지 유실:** 브로커 장애나 소비 실패 → ACK, 영속화, 복제 설정 → *7-4*
- **처리 실패:** 재시도 후 실패하면 **DLQ(Dead Letter Queue)**로 격리
- **순서 보장:** Kafka는 같은 파티션 안에서만 순서가 보장된다 → 같은 주문은 같은 키로 발행
- **DB 저장과 메시지 발행의 원자성:** DB 커밋은 됐는데 발행에 실패하는 문제 → **Transactional Outbox 패턴**

**면접 포인트**
- Q. 메시지가 두 번 처리되면?
  - 멱등성 키로 중복을 체크한다. "정확히 한 번"은 매우 어렵기 때문에 "최소 한 번 + 멱등성"이 실무의 정석이다.

---

## 7. MQ 심화: 전달 보장


```
7-1. 메시지를 저장하는 방식이 다르다 (Queue vs Log)
 → 7-2. 그래서 메시지는 어디서, 어떻게 유실·중복되는가 (전달 보장 수준)
 → 7-3. 중복은 피할 수 없으니 무해하게 만든다 (멱등성)
 → 7-4. 유실과 중복을 가르는 건 "처리"와 "ACK"의 순서다 (ACK와 커밋 타이밍)
 → 7-5. 발행 구간의 중복은 브로커가 막아 준다 (Idempotent Producer)
 → 7-6. Kafka 안에서는 여러 작업을 원자적으로 묶을 수 있다 (Kafka 트랜잭션)
```

---

### 7-1. Queue vs Log

**RabbitMQ: Queue 모델**
```
Producer → [Exchange] → Queue [m3][m2][m1] → Consumer
                                       ↑ ACK를 받으면 삭제
```
- 메시지는 **한 컨슈머에게만** 전달되고, 컨슈머가 ACK하면 큐에서 **삭제**된다.
- 같은 큐에 컨슈머를 여러 개 붙이면 메시지를 **나눠 가진다** (경쟁 소비, Competing Consumers).
- 여러 서비스가 같은 메시지를 받아야 하면 → Fanout Exchange로 **큐를 서비스마다 따로** 만들어서 복사본을 보낸다.
- 지나간 메시지를 다시 읽을 수 없다.

**Kafka: Log 모델**
```
Partition 0: [0][1][2][3][4][5][6] ← 계속 뒤에 추가만 됨 (append-only)
                   ↑           ↑
         정산 그룹 offset=2   알림 그룹 offset=6
```
- 메시지를 읽어도 **삭제되지 않는다.** 보관 기간(retention, 기본 7일)이나 용량이 넘으면 삭제된다.
- 각 **컨슈머 그룹이 자기 위치(offset)를 따로 관리**한다 → 여러 서비스가 같은 데이터를 각자 독립적으로 읽는다.
- offset을 뒤로 돌리면 **과거 메시지를 재처리(replay)**할 수 있다. (버그 수정 후 다시 처리, 새 서비스가 과거 데이터부터 읽기)

**비교**

| | RabbitMQ (Queue) | Kafka (Log) |
|---|---|---|
| 소비 후 | 삭제 | 보관 (retention까지) |
| 여러 서비스가 구독 | 큐를 서비스마다 생성 | 컨슈머 그룹만 추가 |
| 재처리 | 불가 | offset 이동으로 가능 |
| "어디까지 처리했나" 관리 | 브로커가 메시지별 ACK로 관리 | 컨슈머가 offset을 커밋 |
| 적합한 용도 | 작업 분배 (메일 발송, 이미지 변환) | 이벤트 스트림, 로그, 여러 서비스로 팬아웃 |

> ⚠️ RabbitMQ도 3.9부터 **Streams**라는 Log 방식 기능을 제공한다. 위 비교는 전통적인 Queue 기준이다.

**→ 다음으로:** 저장 방식은 달라도, 둘 다 "어디까지 처리했는지"를 **ACK(RabbitMQ)** 또는 **offset 커밋(Kafka)**으로 기록한다. 이 기록이 어긋날 때 유실과 중복이 생긴다.

---

### 7-2. 전달 보장 (Delivery Semantics)

**메시지는 어디서 유실되거나 중복될까?**
```
Producer ──①──> Broker ──②──> Consumer ──③──> 처리 (DB 저장 등)
```
- **① 발행 구간:** 브로커가 받았다는 응답이 네트워크에서 유실됨 → Producer가 재전송하면 **중복**, 안 하면 **유실** 가능 → *7-5에서 해결*
- **②③ 소비 구간:** "처리"와 "처리했다고 알리기(ACK/커밋)" 중 **무엇을 먼저 하느냐**에 따라 결정 → *7-4에서 해결*

**세 가지 보장 수준**

| | 의미 | 결과 | 만드는 방법 (소비 쪽) |
|---|---|---|---|
| **At-most-once** | 최대 한 번 | 유실 O, 중복 X | **커밋 먼저** → 처리 |
| **At-least-once** | 최소 한 번 | 유실 X, 중복 O | **처리 먼저** → 커밋 |
| **Exactly-once** | 정확히 한 번 | 유실 X, 중복 X | At-least-once + **중복 제거** |

**핵심 포인트**
- 네트워크와 서버는 언제든 죽을 수 있으므로 **"유실 없음"과 "중복 없음"을 전송만으로 동시에 보장하기는 매우 어렵다.**
- 그래서 실무의 정석은 **At-least-once로 받고, 멱등성으로 중복을 무해하게 만드는 것**이다.
- 엄밀히 말하면 Exactly-once는 "정확히 한 번 **전달**"이 아니라 "**처리 결과가** 한 번 처리한 것과 같다(effectively-once)"는 뜻이다.

---

### 7-3. 멱등성 (Idempotency)

> **한 줄 정의:** **같은 요청을 여러 번 실행해도 결과가 한 번 실행한 것과 같은** 성질

**해결하는 문제:** At-least-once에서 피할 수 없는 **중복 수신**을 무해하게 만드는 것

**자연스럽게 멱등한 연산 vs 그렇지 않은 연산**

| 멱등함 ✅ | 멱등하지 않음 ❌ |
|---|---|
| `UPDATE orders SET status = 'PAID' WHERE id = 1` | `UPDATE users SET point = point + 100 WHERE id = 1` |
| `DELETE FROM cart WHERE id = 1` | `INSERT INTO payments (...)` (PK가 자동 증가일 때) |
| 값 덮어쓰기 | 증감, 누적, 추가, 외부 발송(메일, 알림톡) |

두 번 실행했을 때 `status`는 여전히 `PAID`지만, 포인트는 200이 쌓인다.

**멱등하지 않은 연산을 멱등하게 만드는 방법**

**① Idempotent Consumer (처리 이력 테이블)**
```sql
CREATE TABLE processed_messages (
    message_id   VARCHAR(64) PRIMARY KEY,   -- UNIQUE 제약이 핵심
    processed_at TIMESTAMP
);
```
```java
@Transactional  // 비즈니스 로직과 이력 저장을 반드시 같은 트랜잭션으로
public void handle(PointEvent event) {
    if (processedRepo.existsById(event.getEventId())) return; // 이미 처리함
    processedRepo.save(new Processed(event.getEventId()));    // 동시에 들어오면 PK 충돌로 하나는 실패
    pointService.add(event.getUserId(), 100);
}
```
- **같은 트랜잭션**이어야 하는 이유: 포인트는 적립됐는데 이력이 저장되지 않으면 다시 중복이 된다.
- **UNIQUE 제약**이 필요한 이유: 두 컨슈머가 동시에 `exists` 검사를 통과하는 경쟁 상황을 DB가 막아 준다.
- ⚠️ **message_id는 Producer가 만든 이벤트 ID**(UUID나 `주문ID + 이벤트 타입`)를 써야 한다. Kafka offset은 Producer가 재전송하면 **새 offset**이 붙기 때문에 중복을 잡지 못한다.

**② 조건부 업데이트 (상태 기반)**
```sql
UPDATE orders SET status = 'PAID' WHERE id = 1 AND status = 'READY';
-- 영향받은 행이 0이면 이미 처리된 것
```

**③ 버전(낙관적 락), 외부 API의 Idempotency-Key**
- 결제 API(Toss, Stripe 등)는 요청 헤더에 **Idempotency-Key**를 넣으면 같은 키의 요청을 한 번만 처리해 준다.

---

### 7-4. ACK와 커밋 타이밍

ACK
Kafka에서는 "커밋"
RabbitMQ는 메시지마다 ACK를 보냄
Kafka는 메시지마다 ACK하지 않고, "여기까지 처리했어요"라고 offset(책갈피 위치)을 저장// 이걸 offset 커밋이라고 하고, 역할은 ACK와 동일 

**핵심 원리 (Kafka와 RabbitMQ 공통)**

| 순서 | 중간에 죽으면 | 결과 |
|---|---|---|
| **처리 → ACK/커밋** | 처리는 했는데 ACK 전에 죽음 → 재시작 후 **다시 받음** | **중복** (At-least-once) |
| **ACK/커밋 → 처리** | ACK했는데 처리 전에 죽음 → 이미 처리됐다고 기록됨 | **유실** (At-most-once) |

→ 유실보다는 중복이 낫고, 중복은 멱등성(7-3)으로 막을 수 있으니 **"처리 후 ACK/커밋"이 기본**이다.

**Kafka: offset 커밋**
- **커밋 = "이 파티션은 여기까지 처리했어"라고 저장하는 것.** 재시작하면 마지막으로 커밋한 offset부터 다시 읽는다.

*auto commit (`enable.auto.commit=true`)*
- 일정 주기(`auto.commit.interval.ms`, 기본 5초)마다 **`poll()`을 호출할 때** 이전에 받아 온 offset을 자동으로 커밋한다.
- ⚠️ 흔히 "auto commit을 쓰면 유실된다"고만 알려져 있지만 정확히는 이렇다.
  - **poll → 처리를 동기로 반복하는 루프**라면 다음 poll 때 커밋되므로 → 대체로 **중복** 쪽 문제 (마지막 커밋 이후 처리분을 재처리)
  - **처리를 다른 스레드로 넘기고 바로 다음 poll을 호출**하면 처리가 끝나기 전에 커밋될 수 있으므로 → **유실**
  - 결국 문제는 **커밋 시점이 "처리 완료"와 무관하게 정해진다**는 것이다.

*수동 커밋*
```java
// Spring Kafka
@KafkaListener(topics = "order-completed")
public void listen(OrderEvent event, Acknowledgment ack) {
    service.process(event);  // 1. 처리
    ack.acknowledge();       // 2. 처리 끝나면 커밋
}
// 설정: spring.kafka.listener.ack-mode=manual
```
- 참고: Spring Kafka는 기본값이 이미 `enable.auto.commit=false`이고, **컨테이너가 리스너 처리가 끝난 뒤 커밋**(AckMode `BATCH`)한다.

**RabbitMQ: ACK**

*autoAck = true (RabbitMQ 클라이언트 옵션)*
- 메시지를 **보내는 즉시** 전달 완료로 보고 삭제한다 → 컨슈머가 처리 중에 죽으면 **유실** (At-most-once)

*manual ack*
- 처리가 끝난 뒤 `basicAck`를 호출한다.
- ACK하지 않은 채 연결이 끊기면 → 브로커가 메시지를 **다시 큐에 넣고** 다른 컨슈머에게 재전달한다 (`redelivered=true`) → 중복 가능
- `basicNack(requeue=false)` → 메시지를 버리거나, 설정된 **DLX(Dead Letter Exchange)**로 보낸다. (6장의 DLQ)

> ⚠️ **헷갈리기 쉬운 부분:** Spring AMQP의 기본값 `AcknowledgeMode.AUTO`는 RabbitMQ의 autoAck와 **다르다.** Spring의 AUTO는 "**리스너가 예외 없이 끝나면** 컨테이너가 ACK, 예외가 나면 NACK"라는 뜻이라 실제로는 처리 후 ACK에 가깝다.

*prefetch (`basicQos`)*
- **ACK하지 않은 메시지를 컨슈머 한 명이 최대 몇 개까지 받아 둘 수 있는지** 정하는 값이다.
- 너무 크면 한 컨슈머가 메시지를 몰아서 가져가 다른 컨슈머는 놀고, 그 컨슈머가 죽으면 대량으로 재전달된다.
- 너무 작으면 매번 ACK를 기다려야 해서 처리량이 떨어진다.
- Spring AMQP 기본값은 250이다. 처리가 무거운 작업이면 낮추는 편이 좋다.

---

### 7-5. Kafka Idempotent Producer

> **한 줄 정의:** 브로커가 **Producer의 재시도로 생긴 중복 메시지를 알아서 걸러 주는** 기능

**해결하는 문제:** 7-2의 **① 발행 구간 중복.** Producer가 보냈는데 브로커의 응답이 유실돼서 **재시도하면 같은 메시지가 두 번 저장되는** 문제

**동작 방식**
```
Producer (PID=7)
  send(seq=0) → Broker 저장 ✅ → 응답 유실 ❌
  재시도 send(seq=0) → Broker: "PID 7의 seq 0은 이미 있음" → 저장하지 않고 성공 응답
```
- 브로커가 Producer에게 **PID(Producer ID)**를 부여한다.
- Producer는 파티션마다 메시지에 **시퀀스 번호**를 붙인다.
- 브로커는 `PID + 파티션 + 시퀀스`가 이미 저장된 것이면 **중복으로 보고 무시**한다. 순서가 꼬인 시퀀스는 거부하므로 **순서도 보장**된다.

**설정**
```properties
enable.idempotence=true   # Kafka 3.0부터 기본값 true
acks=all                  # 필수
retries > 0
max.in.flight.requests.per.connection <= 5
```

**⚠️ 한계 (중요)**
- **Producer 세션 안에서만** 보장된다. Producer가 재시작되면 PID가 새로 바뀐다.
- **브로커의 재시도 중복만** 막는다. 애플리케이션 코드가 `send()`를 두 번 호출하면 서로 다른 메시지로 저장된다.
- **컨슈머 쪽 중복은 전혀 막지 못한다.** 컨슈머 쪽은 여전히 멱등성(7-3)이 필요하다.

---

### 7-6. Kafka 트랜잭션

> **한 줄 정의:** **여러 파티션에 메시지 쓰기 + 컨슈머 offset 커밋**을 하나로 묶어 **전부 성공하거나 전부 취소**되게 만드는 기능

**해결하는 문제:** Kafka에서 읽고 → 가공해서 → Kafka에 다시 쓰는 **Read-Process-Write** 흐름의 중복

**트랜잭션이 없으면 생기는 문제**
```
1. order 토픽에서 메시지 읽음 (offset 10)
2. 가공해서 settlement 토픽에 발행 ✅
3. offset 10 커밋하기 전에 서버 죽음 ❌
4. 재시작 → offset 10부터 다시 읽음 → settlement 토픽에 같은 메시지 또 발행 (중복)
```
→ "발행"과 "offset 커밋"이 **따로따로** 일어나기 때문이다.

**트랜잭션을 쓰면**
```java
producer.initTransactions();

while (true) {
    ConsumerRecords<String, String> records = consumer.poll(Duration.ofMillis(100));
    producer.beginTransaction();
    try {
        for (var record : records) {
            producer.send(new ProducerRecord<>("settlement", record.key(), transform(record.value())));
        }
        // offset 커밋도 트랜잭션 안에 포함 (컨슈머가 직접 커밋하지 않음)
        producer.sendOffsetsToTransaction(currentOffsets(records), consumer.groupMetadata());
        producer.commitTransaction();   // 발행 + offset 커밋이 한 번에 확정
    } catch (ProducerFencedException e) {
        producer.close();               // 같은 transactional.id의 새 인스턴스가 떠서 밀려남
        break;
    } catch (KafkaException e) {
        producer.abortTransaction();    // 전부 취소 → 처음부터 다시
    }
}
```
- Spring에서는 `spring.kafka.producer.transaction-id-prefix`를 설정하면 리스너가 트랜잭션 안에서 실행되도록 컨테이너가 처리해 준다.

**꼭 알아야 할 설정과 개념**
- **`transactional.id`:** Producer를 식별하는 고정 ID. 재시작해도 같은 ID를 쓰면 이전 트랜잭션을 이어서 정리할 수 있다.
  (Idempotent Producer의 PID는 재시작하면 바뀌지만, 트랜잭션은 이 ID 덕분에 **재시작을 넘어서** 보장된다.)
- **좀비 차단(Fencing):** 같은 `transactional.id`로 새 Producer가 뜨면 epoch가 올라가고, 예전 인스턴스(좀비)의 쓰기는 거부된다.
- **`isolation.level=read_committed` (컨슈머 쪽):** 커밋된 트랜잭션의 메시지만 읽는다.
  - ⚠️ 기본값은 `read_uncommitted`라서, 설정하지 않으면 **취소(abort)된 메시지까지 읽는다.** Producer만 트랜잭션을 써서는 의미가 없다.
- Kafka Streams에서는 `processing.guarantee=exactly_once_v2` 한 줄로 이 모든 걸 적용할 수 있다.

**⚠️ 한계 (가장 중요)**
- 보장 범위는 **Kafka 안에서만**이다. **DB 저장, 외부 API 호출, 메일 발송은 트랜잭션에 포함되지 않는다.**
  - Kafka에서 읽고 DB에 저장하는 흐름 → 여전히 **멱등 컨슈머(7-3)**가 필요하다.
  - DB에 저장하고 Kafka에 발행하는 흐름 → 여전히 **Transactional Outbox 패턴**이 필요하다.
- 트랜잭션 커밋 비용이 있어서 처리량이 다소 떨어진다.

**Idempotent Producer vs 트랜잭션**

| | Idempotent Producer (7-5) | 트랜잭션 (7-6) |
|---|---|---|
| 막아 주는 것 | 재시도로 인한 발행 중복 | 발행 + offset 커밋의 불일치 |
| 범위 | 단일 파티션, 단일 세션 | 여러 파티션 + offset, 재시작 이후까지 |
| 설정 | 기본값으로 켜져 있음 | `transactional.id` + 코드 + `read_committed` |
| Kafka 밖(DB, API) | ❌ | ❌ |

---

### 7장 정리: 구간별로 무엇을 쓰나

| 구간 | 위험 | 해결책 |
|---|---|---|
| Producer → Broker | 재시도로 인한 중복 | Idempotent Producer + `acks=all` |
| Broker 내부 | 브로커 장애로 인한 유실 | 복제 (Replication Factor, `min.insync.replicas`) |
| Broker → Consumer | 처리·커밋 순서로 인한 유실·중복 | **처리 후 커밋/ACK** |
| Consumer → DB/외부 | 중복 처리 | **멱등 컨슈머** (처리 이력 테이블, 조건부 업데이트) |
| Kafka → Kafka | 발행·커밋 불일치 | Kafka 트랜잭션 |
| DB → Kafka | DB 커밋과 발행의 불일치 | Transactional Outbox |

**면접 포인트**
- Q. Kafka는 Exactly-once를 지원한다는데, 그럼 멱등성 처리는 필요 없나?
  - 필요하다. Kafka의 Exactly-once는 **Kafka 안에서 읽고 쓰는 범위**에서만 보장된다. DB 저장이나 외부 API 호출은 포함되지 않으므로 컨슈머 쪽 멱등성은 여전히 필요하다.
- Q. auto commit을 쓰면 왜 문제가 되나?
  - 커밋 시점이 처리 완료와 무관하게 주기적으로 정해지기 때문이다. 동기 루프에서는 주로 중복이, 처리를 비동기로 넘기면 유실이 생길 수 있다.
- Q. 멱등 컨슈머의 중복 체크 키로 Kafka offset을 쓰면 안 되는 이유는?
  - Producer가 재전송하면 같은 내용이 새 offset으로 저장되기 때문이다. Producer가 만든 이벤트 ID나 비즈니스 키를 써야 한다.

---

## 한눈에 보는 연결 관계

```
트래픽 증가
 → Scale-up (한계, SPOF)
 → Scale-out
    → Load Balancing (요청 분산)
       → Stateless (세션 문제 해결)
 → DB 병목
    → Replication (읽기 분산) → 복제 지연 주의
 → 응답 지연
    → Sync를 Async로
       → MQ (신뢰성 있는 비동기) → 중복·유실·순서 주의
          → 저장 방식 이해: Queue vs Log
          → 유실 방지: 처리 후 ACK/커밋 (At-least-once)
             → 그러면 중복 발생 → 멱등 컨슈머
          → 발행 중복 방지: Idempotent Producer
          → Kafka→Kafka 원자성: Kafka 트랜잭션
          → DB→Kafka 원자성: Transactional Outbox (다음 공부 주제)
```