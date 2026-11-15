# 11주차 - Transaction / 동시성

## 1. ACID

> Transaction은 데이터베이스에서 하나의 논리적 작업 단위로 처리되는 연산들의 집합이다.

예를 들어 계좌 이체에서는 출금과 입금이 하나의 Transaction으로 처리되어야 한다.

```text
A 계좌에서 10,000원 출금
          ↓
B 계좌에 10,000원 입금
          ↓
       Commit
```

중간에 오류가 발생하면 전체 작업을 취소하여 데이터의 일관성을 유지해야 한다.

### ACID의 4가지 특성

| 특성 | 의미 |
|---|---|
| Atomicity (원자성) | 모든 작업이 성공하거나 모두 취소되어야 함 |
| Consistency (일관성) | 트랜잭션 전후 데이터 무결성 제약 조건 유지 |
| Isolation (격리성) | 동시에 실행되는 트랜잭션 간 간섭 제어 |
| Durability (지속성) | Commit된 결과는 장애 이후에도 보존 |

**1. Atomicity (원자성)**

트랜잭션에 포함된 작업은 전부 성공하거나 전부 실패해야 한다.

```sql
START TRANSACTION;

UPDATE Account
SET balance = balance - 10000
WHERE id = 1;

UPDATE Account
SET balance = balance + 10000
WHERE id = 2;

COMMIT;
```

두 작업 중 하나라도 실패하면 Rollback을 통해 변경 사항을 취소할 수 있다.

**2. Consistency (일관성)**

트랜잭션은 데이터베이스에 정의된 무결성 제약 조건을 유지해야 한다.

예를 들어 잔액이 음수가 될 수 없다는 제약이 있다면 트랜잭션 이후에도 이를 만족해야 한다.

단, 비즈니스 규칙은 애플리케이션에서도 적절하게 검증해야 한다.

**3. Isolation (격리성)**

여러 트랜잭션이 동시에 실행되더라도 서로의 작업이 잘못된 결과를 만들지 않도록 제어한다.

격리 수준에 따라 다른 트랜잭션의 변경 사항이 보이는 범위가 달라진다.

**4. Durability (지속성)**

Commit된 데이터는 시스템 장애가 발생하더라도 보존되어야 한다.

MySQL InnoDB에서는 Redo Log 등을 활용해 장애 발생 후 데이터를 복구한다.

---

## 2. Isolation Level (트랜잭션 격리 수준)

> Isolation Level은 여러 트랜잭션이 동시에 실행될 때 서로의 데이터 변경을 어느 수준까지 허용할지 결정하는 설정이다.

격리 수준이 높을수록 동시 실행에 따른 이상 현상을 줄일 수 있지만, 동시 처리 성능이나 대기 시간에 영향을 줄 수 있다.

### 동시성 이상 현상

**1. Dirty Read**

다른 트랜잭션에서 아직 Commit하지 않은 데이터를 읽는 현상이다.

```text
Transaction A          Transaction B
     │                      │
balance = 5000               │
     │                      │
     │                 balance 조회
     │                    5000
     │                      │
  ROLLBACK                  │
```

Transaction B가 실제로 확정되지 않은 데이터를 읽게 된다.

**2. Non-Repeatable Read**

하나의 트랜잭션에서 같은 Row를 두 번 조회했을 때 결과가 달라지는 현상이다.

```text
Transaction A          Transaction B
     │                      │
balance 조회                 │
  10000                      │
     │                  balance 수정
     │                    COMMIT
     │                      │
balance 재조회               │
  20000                      │
```

**3. Phantom Read**

같은 조건으로 조회했을 때 다른 트랜잭션의 INSERT 또는 DELETE 등으로 결과 Row 집합이 달라지는 현상이다.

```text
Transaction A          Transaction B
     │                      │
주문 조회: 5건               │
     │                   주문 INSERT
     │                     COMMIT
     │                      │
주문 재조회: 6건             │
```

### Isolation Level의 종류

| 격리 수준 | Dirty Read | Non-Repeatable Read | Phantom Read |
|---|---|---|---|
| READ UNCOMMITTED | 가능 | 가능 | 가능 |
| READ COMMITTED | 방지 | 가능 | 가능 |
| REPEATABLE READ | 방지 | 방지 | 표준상 가능 |
| SERIALIZABLE | 방지 | 방지 | 방지 |

위 표는 SQL 표준을 기준으로 한 일반적인 구분이며, 실제 동작은 DBMS 구현에 따라 달라질 수 있다.

### READ UNCOMMITTED

Commit되지 않은 데이터도 읽을 수 있는 가장 낮은 격리 수준이다.

Dirty Read가 발생할 수 있어 일반적인 업무 시스템에서는 거의 사용하지 않는다.

### READ COMMITTED

다른 트랜잭션에서 Commit한 데이터만 읽을 수 있다.

동일한 트랜잭션에서 다시 조회하면 다른 트랜잭션이 Commit한 변경 사항이 보일 수 있다.

Oracle과 PostgreSQL의 기본 격리 수준이다.

### REPEATABLE READ

동일한 트랜잭션에서 반복적으로 조회할 때 일관된 결과를 제공하는 격리 수준이다.

MySQL InnoDB의 기본 격리 수준이다.

InnoDB는 MVCC 기반의 Snapshot Read를 사용하여 일반적인 SELECT에서 일관된 조회 결과를 제공한다.

```text
Transaction 시작
       ↓
Snapshot 기준 조회
       ↓
다른 Transaction Commit
       ↓
기존 Snapshot 기준 재조회
```

InnoDB의 일반적인 Snapshot Read에서는 Phantom Read도 방지할 수 있다.

다만 `SELECT ... FOR UPDATE`와 같은 Locking Read는 최신 데이터를 기준으로 동작하므로 일반 SELECT와 결과가 달라질 수 있다.

### SERIALIZABLE

트랜잭션이 직렬적으로 실행된 것과 동등한 결과를 보장하는 가장 높은 격리 수준이다.

동시 실행으로 인한 이상 현상을 방지하지만 Lock 경합이나 대기 시간이 증가할 수 있다.

### MVCC (Multi-Version Concurrency Control)

> MVCC는 데이터의 여러 버전을 관리하여 읽기와 쓰기 사이의 충돌을 줄이는 동시성 제어 방식이다.

MySQL InnoDB는 Undo Log 등을 이용해 이전 버전의 데이터를 조회할 수 있도록 한다.

```text
Transaction A              Transaction B
     │                          │
기존 데이터 조회                │
     │                      데이터 수정
     │                        COMMIT
     │                          │
이전 버전 조회 가능             │
```

MVCC를 이용하면 일반적인 SELECT가 다른 트랜잭션의 UPDATE를 기다리지 않고 일관된 데이터를 읽을 수 있다.

단, MVCC가 모든 동시성 문제를 해결하는 것은 아니다. 여러 트랜잭션이 같은 데이터를 수정하는 경우에는 별도의 동시성 제어가 필요할 수 있다.

---

## 3. Lock

> Lock은 여러 트랜잭션이 동일한 데이터에 동시에 접근할 때 충돌을 제어하기 위한 동기화 메커니즘이다.

### Shared Lock / Exclusive Lock

**Shared Lock (S-Lock)**

데이터를 읽기 위한 Lock이다.

여러 트랜잭션이 동시에 Shared Lock을 획득할 수 있지만, Exclusive Lock과는 충돌한다.

**Exclusive Lock (X-Lock)**

데이터를 수정하기 위한 Lock이다.

다른 트랜잭션의 Shared Lock 및 Exclusive Lock과 충돌한다.

| 현재 Lock | S-Lock 요청 | X-Lock 요청 |
|---|---|---|
| S-Lock | 허용 | 대기 |
| X-Lock | 대기 | 대기 |

### Row Lock

특정 Row에 대한 접근을 제어한다.

```sql
SELECT *
FROM Product
WHERE id = 1
FOR UPDATE;
```

`FOR UPDATE`는 조회 대상에 Exclusive Lock을 요청한다.

```text
Transaction A
    ↓
Product 1 Lock 획득
    ↓
재고 수정
    ↓
Commit
    ↓
Lock 해제
```

다른 트랜잭션이 충돌하는 Lock을 요청하면 대기해야 한다.

### Gap Lock / Next-Key Lock

MySQL InnoDB는 Row뿐 아니라 Index의 범위에 대해서도 Lock을 사용할 수 있다.

**Gap Lock**

Index Record 사이의 간격에 대한 삽입을 제한하는 Lock이다.

**Next-Key Lock**

Record Lock과 Gap Lock을 결합한 형태이다.

```text
Index

10       20       30
 │        │        │
 └─ Gap ──┘
```

이러한 Lock은 특정 범위에 새로운 Row가 삽입되어 조회 결과가 달라지는 문제를 방지하는 데 활용된다.

InnoDB의 Lock 범위는 격리 수준, Index, 검색 조건 등에 따라 달라진다.

특히 적절한 Index가 없으면 예상보다 넓은 범위에 Lock이 설정되어 동시 처리 성능이 저하될 수 있다.

---

## 4. Deadlock (교착 상태)

> Deadlock은 두 개 이상의 트랜잭션이 서로 상대방이 보유한 Lock의 해제를 기다리면서 진행하지 못하는 상태이다.

### Deadlock 발생 과정

```text
Transaction A              Transaction B
     │                          │
Row 1 Lock 획득          Row 2 Lock 획득
     │                          │
Row 2 Lock 요청          Row 1 Lock 요청
     │                          │
     └──── 서로 대기 ────────────┘
```

각 트랜잭션이 상대방의 Lock을 기다리므로 작업을 완료할 수 없다.

### Deadlock의 4가지 필요조건

| 조건 | 의미 |
|---|---|
| Mutual Exclusion | 자원을 동시에 사용할 수 없음 |
| Hold and Wait | 자원을 보유한 상태에서 다른 자원 대기 |
| No Preemption | 자원을 강제로 회수할 수 없음 |
| Circular Wait | 서로의 자원을 순환하며 대기 |

네 조건이 모두 만족되면 Deadlock이 발생할 수 있다.

### Deadlock 해결 방법

**1. Lock 획득 순서 통일**

여러 Row를 수정할 때 항상 동일한 순서로 Lock을 획득한다.

```text
Transaction A: Row 1 → Row 2
Transaction B: Row 1 → Row 2
```

서로 반대 순서로 Lock을 요청하는 상황을 줄일 수 있다.

**2. 트랜잭션 범위 최소화**

트랜잭션 내부에서 외부 API 호출이나 불필요한 연산을 수행하면 Lock 유지 시간이 길어질 수 있다.

가능한 한 트랜잭션을 짧게 유지하는 것이 유리하다.

**3. 적절한 Index 사용**

불필요하게 많은 Row를 탐색하거나 Lock을 획득하지 않도록 조회 조건에 맞는 Index를 구성한다.

**4. 재시도 처리**

MySQL InnoDB는 일반적으로 Deadlock을 감지하면 트랜잭션 하나를 Rollback하여 교착 상태를 해소한다.

애플리케이션에서는 Deadlock으로 실패한 트랜잭션을 재시도할 수 있도록 설계할 수 있다.

단, 재시도 시에는 트랜잭션 전체를 다시 실행해야 하며 멱등성과 재시도 횟수를 고려해야 한다.

---

## 5. 낙관적 Lock / 비관적 Lock

> 낙관적 Lock과 비관적 Lock은 여러 트랜잭션이 동일한 데이터를 수정할 때 충돌을 제어하는 대표적인 방식이다.

### Lost Update

두 트랜잭션이 동일한 데이터를 조회한 뒤 각각 수정하면서 한쪽의 변경 사항이 사라지는 문제이다.

```text
초기 재고: 10

Transaction A          Transaction B
     │                      │
재고 조회: 10           재고 조회: 10
     │                      │
재고 -1                 재고 -1
     │                      │
재고 저장: 9            재고 저장: 9
```

정상적인 최종 재고는 8이어야 하지만 9가 될 수 있다.

이를 Lost Update라고 한다.

### 낙관적 Lock (Optimistic Lock)

> 충돌이 자주 발생하지 않는다고 가정하고, 데이터 수정 시 Version을 비교하여 충돌을 감지하는 방식이다.

JPA에서는 `@Version`을 이용할 수 있다.

```java
@Entity
public class Product {

    @Id
    private Long id;

    private int stock;

    @Version
    private Long version;
}
```

데이터를 수정할 때 Version을 조건으로 사용한다.

```sql
UPDATE Product
SET stock = 9,
    version = 2
WHERE id = 1
  AND version = 1;
```

다른 트랜잭션이 먼저 수정하여 Version이 변경되었다면 UPDATE 대상이 없어 충돌을 감지할 수 있다.

```text
Transaction A          Transaction B
     │                      │
Version = 1            Version = 1
     │                      │
UPDATE 성공            UPDATE 실패
Version = 2            Version 불일치
```

JPA에서는 이러한 충돌이 발생하면 OptimisticLockException 등이 발생할 수 있다.

**특징**

- 조회 시 DB Lock을 미리 획득하지 않음
- 충돌이 적은 환경에 유리
- 충돌 발생 시 재시도 또는 실패 처리 필요

### 비관적 Lock (Pessimistic Lock)

> 데이터 충돌이 발생할 수 있다고 가정하고, DB Lock을 획득하여 다른 트랜잭션의 충돌하는 접근을 제한하는 방식이다.

JPA에서는 `PESSIMISTIC_WRITE` 등을 사용할 수 있다.

```java
@Lock(LockModeType.PESSIMISTIC_WRITE)
@Query("SELECT p FROM Product p WHERE p.id = :id")
Optional<Product> findByIdForUpdate(Long id);
```

MySQL InnoDB에서는 일반적으로 다음과 같은 SQL이 사용된다.

```sql
SELECT *
FROM Product
WHERE id = 1
FOR UPDATE;
```

```text
Transaction A          Transaction B
     │                      │
Lock 획득               Lock 요청
     │                      │
재고 수정                   대기
     │                      │
Commit                    │
     │                      │
Lock 해제               Lock 획득
                            │
                        재고 수정
```

**특징**

- 데이터 수정 충돌을 사전에 제어
- 충돌이 빈번한 환경에 적합할 수 있음
- Lock 대기 및 Deadlock 발생 가능
- 트랜잭션 유지 시간이 길어지면 성능 저하 가능

### 낙관적 Lock vs 비관적 Lock

| 구분 | 낙관적 Lock | 비관적 Lock |
|---|---|---|
| 기본 접근 | 충돌 발생 후 감지 | 충돌 발생 전 제어 |
| 구현 | Version 비교 | DB Lock |
| 충돌 시 | 예외 및 재시도 | Lock 대기 |
| 장점 | Lock 경합 감소 | 충돌 상황에서 순차 처리 |
| 단점 | 재시도 비용 | 대기 및 Deadlock |
| 적합한 상황 | 충돌이 적은 환경 | 충돌이 빈번한 환경 |

### 조건부 UPDATE를 이용한 동시성 제어

Lock을 명시적으로 획득하지 않고 UPDATE 조건을 이용해 정합성을 보장할 수도 있다.

```sql
UPDATE Product
SET stock = stock - 1
WHERE id = 1
  AND stock > 0;
```

데이터베이스는 UPDATE를 원자적으로 처리하므로 재고가 0 이하로 감소하는 것을 방지할 수 있다.

애플리케이션에서는 영향을 받은 Row 수를 확인하여 성공 여부를 판단한다.

```text
Affected Rows = 1
→ 재고 차감 성공

Affected Rows = 0
→ 조건 불충족
```

단순한 수량 변경에서는 조건부 UPDATE가 별도의 조회 후 수정 방식보다 효율적일 수 있다.

---

## 6. Transaction Propagation (트랜잭션 전파)

> Transaction Propagation은 트랜잭션이 존재하는 상황에서 다른 트랜잭션 메서드를 호출했을 때 기존 트랜잭션을 사용할지, 새로운 트랜잭션을 생성할지 결정하는 설정이다.

Spring에서는 `@Transactional`의 `propagation` 속성으로 설정한다.

### 주요 전파 옵션

| 옵션 | 기존 트랜잭션이 있는 경우 | 없는 경우 |
|---|---|---|
| REQUIRED | 기존 트랜잭션 참여 | 새 트랜잭션 생성 |
| REQUIRES_NEW | 기존 트랜잭션 일시 중단 후 새 트랜잭션 생성 | 새 트랜잭션 생성 |
| SUPPORTS | 기존 트랜잭션 참여 | 트랜잭션 없이 실행 |
| NOT_SUPPORTED | 기존 트랜잭션 일시 중단 | 트랜잭션 없이 실행 |
| MANDATORY | 기존 트랜잭션 참여 | 예외 발생 |
| NEVER | 예외 발생 | 트랜잭션 없이 실행 |
| NESTED | 중첩 트랜잭션 생성 | 새 트랜잭션 생성 |

### REQUIRED

Spring의 기본 전파 옵션이다.

```java
@Transactional
public void order() {
    paymentService.pay();
}

@Transactional
public void pay() {
    // 결제 처리
}
```

두 메서드는 일반적으로 하나의 물리적 트랜잭션에 참여한다.

```text
OrderService
     │
     │ Transaction A
     ↓
PaymentService
     │
     │ Transaction A 참여
     ↓
Commit
```

내부 작업에서 Rollback이 필요하다고 표시되면 전체 트랜잭션이 Rollback될 수 있다.

### REQUIRES_NEW

기존 트랜잭션과 별도의 새로운 트랜잭션을 생성한다.

```java
@Transactional
public void order() {
    orderRepository.save(order);
    auditService.saveLog();
}
```

```java
@Transactional(
    propagation = Propagation.REQUIRES_NEW
)
public void saveLog() {
    auditRepository.save(log);
}
```

```text
Order Transaction A
       │
       │ 일시 중단
       ↓
Log Transaction B
       │
       └─ Commit
       ↓
Transaction A 재개
```

Transaction B가 Commit된 이후 Transaction A가 Rollback되더라도 B의 결과는 유지될 수 있다.

따라서 감사 로그처럼 독립적인 트랜잭션 처리가 필요한 경우 활용할 수 있다.

단, 별도의 DB Connection이 필요할 수 있으므로 Connection Pool 크기와 동시 실행 수를 고려해야 한다.

### NESTED

기존 트랜잭션 내부에 Savepoint 기반의 중첩 트랜잭션을 생성하는 방식이다.

```text
Transaction A
     │
     ├─ 작업 1
     │
     ├─ Savepoint
     │     │
     │     └─ 작업 2
     │
     └─ Commit
```

중첩 작업이 실패하면 Savepoint까지 Rollback할 수 있다.

다만 외부 트랜잭션이 Rollback되면 중첩 작업도 함께 Rollback된다.

`NESTED` 지원 여부는 사용하는 Transaction Manager와 DBMS에 따라 달라진다.

### @Transactional의 주의사항

**1. Self-Invocation 문제**

Spring의 기본 Proxy 기반 트랜잭션 처리에서는 같은 클래스 내부에서 메서드를 직접 호출하면 `@Transactional`이 적용되지 않을 수 있다.

```java
public void order() {
    saveOrder();
}

@Transactional
public void saveOrder() {
    // 트랜잭션 적용을 기대한 작업
}
```

```text
외부 호출
   ↓
Spring Proxy
   ↓
Target Method
```

내부 메서드 호출은 Proxy를 거치지 않으므로 트랜잭션 AOP가 적용되지 않는다.

**2. Rollback 조건**

Spring의 기본 설정에서는 일반적으로 다음과 같이 동작한다.

| 예외 | 기본 Rollback |
|---|---|
| RuntimeException | O |
| Error | O |
| Checked Exception | X |

Checked Exception에서도 Rollback이 필요하다면 다음과 같이 설정할 수 있다.

```java
@Transactional(rollbackFor = Exception.class)
```

**3. 트랜잭션 범위**

트랜잭션 내부에서 외부 API 호출이나 장시간 작업을 수행하면 DB Connection과 Lock을 오래 점유할 수 있다.

따라서 트랜잭션은 데이터 정합성을 보장하는 데 필요한 범위로 설정하는 것이 중요하다.

**4. 예외를 처리하더라도 Rollback될 수 있음**

기존 트랜잭션에 참여한 내부 메서드에서 트랜잭션이 Rollback-Only로 표시되었다면, 외부 메서드가 예외를 처리하더라도 최종 Commit 시 `UnexpectedRollbackException`이 발생할 수 있다.

트랜잭션 전파 설정에서는 **논리적 트랜잭션의 경계와 실제 DB 트랜잭션의 경계가 동일한지**를 구분해야 한다.
