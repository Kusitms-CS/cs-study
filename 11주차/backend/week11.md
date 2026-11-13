# 11주차 - Transaction / 동시성 개념 정리

## 목차
- [키워드 연관관계](#키워드-연관관계)
- [Transaction](#transaction)
- [ACID](#acid)
- [Isolation Level](#isolation-level)
- [Lock](#lock)
- [Deadlock](#deadlock)
- [낙관적/비관적 Lock](#낙관적비관적-lock)
- [Transaction Propagation](#transaction-propagation)

---

## 키워드 연관관계

```
Transaction (여러 작업을 하나의 논리적 단위로 묶음)
├─ ACID : 트랜잭션이 보장해야 하는 4가지 성질
│   └─ Isolation(격리성) ──구현 수준──> Isolation Level
│                          └─ 동시성 문제(Dirty/Non-Repeatable/Phantom Read) 방지
│                          └─ 내부 구현 수단 ──> Lock (또는 MVCC)
│                                                 ├─ Deadlock : Lock 대기가 순환
│                                                 └─ 낙관적/비관적 Lock : Lock 적용 전략
└─ Transaction Propagation : 트랜잭션이 중첩 호출될 때 경계를 결정 (Spring)
```
---


## Transaction

- DB에서 **하나의 논리적 작업 단위**로 처리되어야 하는 연산들의 묶음. 전부 성공(**Commit**)하거나 전부 취소(**Rollback**)
- 예: 계좌 이체 = 출금 + 입금이 함께 성공하거나 함께 실패
- 흐름: `BEGIN` → 작업 수행 → `COMMIT` / `ROLLBACK`

---

## ACID

트랜잭션이 보장해야 하는 4가지 성질(필기 시험 자주 나옴!! 외우기1!!)

| 성질 | 정의 | 구현 수단 |
|---|---|---|
| **Atomicity(원자성)** | 트랜잭션의 모든 연산이 전부 반영되거나 전부 반영되지 않음 (All or Nothing) | Undo Log, Rollback |
| **Consistency(일관성)** | 트랜잭션 전후로 데이터가 정의된 규칙(제약조건, 무결성)을 만족하는 상태를 유지 | 제약조건(PK/FK/UNIQUE), 애플리케이션 로직 |
| **Isolation(격리성)** | 동시에 실행되는 트랜잭션이 서로의 중간 상태에 영향을 주지 않음 | Lock, MVCC, Isolation Level |
| **Durability(지속성)** | 커밋된 결과는 장애가 발생해도 영구적으로 보존 | Redo Log(WAL), 디스크 기록 |

---

## Isolation Level

- 동시에 실행되는 트랜잭션 간 **격리 정도를 단계로 정한 설정**. 높을수록 데이터 정합성은 높아지고 동시 처리 성능은 낮아짐

**동시성 문제**

| 현상 | 설명 |
|---|---|
| Dirty Read | 다른 트랜잭션이 **커밋하지 않은** 변경 데이터를 읽음 |
| Non-Repeatable Read | 같은 트랜잭션 내에서 같은 행을 두 번 읽었는데 **다른 값**이 읽힘 (다른 트랜잭션이 UPDATE 후 커밋) |
| Phantom Read | 같은 조건으로 조회했는데 **행의 개수가 달라짐** (다른 트랜잭션이 INSERT/DELETE 후 커밋) |

**격리 수준별 허용 여부 (표준 기준)****(필기 시험 자주 나옴 알아두작!!!!!)**

| 격리 수준 | Dirty Read | Non-Repeatable Read | Phantom Read |
|---|---|---|---|
| READ UNCOMMITTED | 발생 | 발생 | 발생 |
| READ COMMITTED | 방지 | 발생 | 발생 |
| REPEATABLE READ | 방지 | 방지 | 발생 (MySQL InnoDB는 대부분 방지) |
| SERIALIZABLE | 방지 | 방지 | 방지 |

- 기본값: **MySQL(InnoDB) = REPEATABLE READ**, **PostgreSQL/Oracle = READ COMMITTED**
- **MVCC(Multi-Version Concurrency Control)**: 데이터의 여러 버전(스냅숏)을 유지해 읽기가 쓰기를 막지 않도록 하는 방식. READ COMMITTED/REPEATABLE READ에서 일반 SELECT는 Lock 없이 스냅숏을 읽음

---



## Lock

- 여러 트랜잭션이 같은 데이터에 동시에 접근할 때 **정합성을 위해 접근을 제한**하는 메커니즘

**종류**

| 종류 | 설명 | 호환성 |
|---|---|---|
| **Shared Lock (S, 공유/읽기)** | 읽기 시 획득 | 다른 S Lock과 공존 가능, X Lock과는 불가 |
| **Exclusive Lock (X, 배타/쓰기)** | 쓰기(UPDATE/DELETE) 시 획득 | 다른 S/X Lock 모두와 공존 불가 |

**사례 연결: 쇼핑몰 쿠폰 이중 차감 ([week11_3.md](week11_3.md#4-운영))**
Pod 20개에서 스케줄러가 동시에 실행되어 같은 쿠폰 행을 UPDATE하는 상황




| 단계 | Lock 개념과의 연결 |
|---|---|
| 같은 행 동시 UPDATE | 먼저 UPDATE한 트랜잭션이 해당 행의 **X Lock**을 획득, 나머지는 커밋/롤백까지 **Lock 대기** |
| 대기 중 커넥션 점유 | 대기 중인 트랜잭션도 DB 커넥션을 반납하지 못해 Pool이 고갈 → **커넥션 대기 타임아웃** (HikariCP, 8주차 참고) |
| 대기 시간 초과 | `innodb_lock_wait_timeout`(기본 50초)을 넘으면 쿼리가 에러로 종료 |
| 서로 다른 순서로 Lock | Pod A는 쿠폰1 → 쿠폰2, Pod B는 쿠폰2 → 쿠폰1 순으로 잠그면 **Deadlock** 발생 → DB가 한쪽을 강제 Rollback |
| 락이 있는데 이중 차감 | Lock은 **"동시에"** 수정하는 것만 막고, **"차례대로 여러 번"** 실행되는 것은 막지 못함. 20번 중복 실행은 순차 실행되어 20번 모두 차감 |

- 해결: 한 번만 실행되게 하는 **분산 락 / CronJob**과, 중복 실행되어도 결과가 같은 **멱등성**(조건부 UPDATE)을 함께 사용
- 정리: DB Lock은 **같은 시각의 데이터 정합성**을 지키는 수단이고, 중복 실행 방지(분산 락)와 결과 보장(멱등성)은 **애플리케이션 레벨**에서 별도로 필요

---

## Deadlock

- 둘 이상의 트랜잭션이 **서로가 보유한 Lock을 기다리며 영원히 대기**하는 상태 (순환 대기)

- 발생 조건(Coffman): 상호 배제, 점유 대기, 비선점, **순환 대기** — 하나라도 깨면 방지 가능
- DB의 처리: **Deadlock Detection** 후 한 트랜잭션을 **희생자(Victim)**로 선택해 강제 Rollback (MySQL은 `Deadlock found` 에러)
- 예방
  - 모든 트랜잭션이 **Lock 획득 순서를 일관**되게 유지
  - 트랜잭션을 **짧게** 유지하고, 트랜잭션 안에서 외부 호출 등 느린 작업 제거
  - 인덱스를 적절히 사용해 Lock 범위 최소화
  - Lock Timeout 설정, 애플리케이션에서 재시도(Retry)




### 낙관적/비관적 Lock

| | 비관적 Lock (Pessimistic) | 낙관적 Lock (Optimistic) |
|---|---|---|
| 가정 | 충돌이 **자주** 발생한다 | 충돌이 **드물게** 발생한다 |
| 방식 | 데이터를 읽을 때부터 **DB Lock**을 걸어 선점 | Lock 없이 진행하고 **커밋 시점에 충돌 검출** |
| 구현 | `SELECT ... FOR UPDATE`, JPA `@Lock(PESSIMISTIC_WRITE)` | **Version 컬럼**, JPA `@Version` |
| 충돌 시 | 다른 트랜잭션이 Lock 해제까지 **대기** | 갱신 실패 → 예외(`OptimisticLockException`) 발생 후 **재시도 로직 필요** |
| 장점 | 충돌 방지 확실, 데이터 정합성 높음 | 대기 없음 → 읽기 많은 환경에서 성능 좋음, Deadlock 위험 낮음 |
| 단점 | 대기·Deadlock 가능, 동시성 저하 | 충돌 잦으면 재시도 비용 증가 |
| 적합한 경우 | 재고 차감, 포인트 등 충돌이 잦은 쓰기 | 게시글 수정 등 충돌이 드문 경우 |

**낙관적 Lock 동작 (Version)**
```sql
UPDATE product SET stock = 9, version = 2
WHERE id = 1 AND version = 1;
-- 영향받은 행이 0이면 다른 트랜잭션이 먼저 수정한 것 → 충돌
```


## Transaction Propagation

- **이미 진행 중인 트랜잭션이 있을 때, 트랜잭션이 필요한 메서드를 호출하면 어떻게 동작할지 정하는 규칙** (Spring `@Transactional(propagation = ...)`)
- 기본값은 `REQUIRED`

| 속성 | 진행 중인 트랜잭션이 있을 때 | 없을 때 |
|---|---|---|
| **REQUIRED** (기본) | 기존 트랜잭션에 **참여** | 새로 생성 |
| **REQUIRES_NEW** | 기존 트랜잭션을 **보류**하고 **새 트랜잭션** 생성 (독립 커밋/롤백) | 새로 생성 |
| **SUPPORTS** | 참여 | 트랜잭션 없이 실행 |
| **NOT_SUPPORTED** | 기존 트랜잭션을 보류하고 트랜잭션 없이 실행 | 트랜잭션 없이 실행 |
| **MANDATORY** | 참여 | 예외 발생 |
| **NEVER** | 예외 발생 | 트랜잭션 없이 실행 |
| **NESTED** | **중첩(Savepoint)** 트랜잭션 생성. 내부 롤백이 외부에 영향 없음 | 새로 생성 |


- 주의: 같은 클래스 내부 메서드 호출(self-invocation)은 프록시를 거치지 않아 `@Transactional`/전파 속성이 **적용되지 않음**

