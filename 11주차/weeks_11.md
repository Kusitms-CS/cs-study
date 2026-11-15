# 11주차 - Transaction / 동시성 (Backend)

여러 요청이 같은 데이터를 동시에 바꿀 때 무엇이 깨지고, DB와 Spring이 이를 어떻게 막는지 정리했다. 예시는 MySQL(InnoDB) + Spring Data JPA 기준이다.

## 1. ACID

트랜잭션은 **더 이상 쪼갤 수 없는 작업 단위**다. "계좌 A에서 출금 + 계좌 B에 입금"은 둘 다 되거나 둘 다 안 되어야 한다.

| 속성 | 의미 | InnoDB에서 보장하는 방법 |
| --- | --- | --- |
| 원자성 (Atomicity) | 전부 반영되거나 전부 취소된다 | **Undo Log**에 변경 전 값을 남겨 롤백 시 되돌림 |
| 일관성 (Consistency) | 트랜잭션 전후로 데이터가 규칙(제약조건)을 만족한다 | PK·FK·UNIQUE·CHECK 제약조건 + 애플리케이션 로직 |
| 격리성 (Isolation) | 동시에 실행되는 트랜잭션이 서로 영향을 주지 않는 것처럼 보인다 | Lock + **MVCC** (격리 수준에 따라 정도가 다름) |
| 지속성 (Durability) | 커밋된 내용은 장애가 나도 사라지지 않는다 | **Redo Log**(WAL)를 커밋 시점에 디스크에 기록 |

### WAL (Write-Ahead Logging)

데이터 페이지를 매번 디스크에 쓰면 느리다. 그래서 변경 내용을 먼저 **순차 쓰기**인 Redo Log에 기록하고, 실제 데이터 페이지는 메모리(Buffer Pool)에서 바꾼 뒤 나중에 디스크로 내린다.

```
UPDATE 요청
  → Buffer Pool의 페이지 수정 (Dirty Page)
  → Undo Log 기록 (롤백·MVCC용)
  → Redo Log 기록
COMMIT
  → Redo Log를 디스크에 fsync  ← 이 시점부터 지속성 보장
  → Dirty Page는 나중에 Checkpoint 때 디스크로
장애 후 재시작
  → Redo Log로 커밋된 변경을 다시 적용, Undo Log로 미완료 트랜잭션 롤백
```

| `innodb_flush_log_at_trx_commit` | 동작 | 장애 시 |
| --- | --- | --- |
| `1` (기본값) | 커밋마다 Redo Log를 디스크에 fsync | 커밋된 데이터 유실 없음 |
| `2` | 커밋마다 OS 캐시에 쓰고, 1초마다 fsync | OS가 죽으면 최대 1초 유실 |
| `0` | 1초마다 쓰기 + fsync | MySQL만 죽어도 최대 1초 유실 |

### 일관성은 DB만으로 보장되지 않는다

"재고는 0 미만이 될 수 없다"를 애플리케이션 코드에서만 검사하면 동시 요청에서 깨질 수 있다. 가능하면 **DB 제약조건**으로 마지막 방어선을 둔다.

```sql
ALTER TABLE product ADD CONSTRAINT chk_stock CHECK (stock >= 0);  -- MySQL 8.0.16+
ALTER TABLE coupon_issue ADD CONSTRAINT uk_user_coupon UNIQUE (user_id, coupon_id); -- 중복 발급 방지
```

## 2. Isolation Level

격리성을 완벽하게 지키면(모든 트랜잭션을 줄 세워 실행) 동시성이 떨어진다. 그래서 **어느 정도의 이상 현상을 허용할지** 단계로 나눈 것이 격리 수준이다.

### 동시성 이상 현상

| 현상 | 설명 | 예 |
| --- | --- | --- |
| Dirty Read | 커밋되지 않은 다른 트랜잭션의 변경을 읽음 | B가 잔액을 0으로 바꾸고 롤백했는데 A는 0을 읽고 처리 |
| Non-Repeatable Read | 같은 행을 두 번 읽었는데 값이 다름 | A가 읽은 뒤 B가 수정·커밋, A가 다시 읽으니 값이 바뀜 |
| Phantom Read | 같은 조건으로 두 번 조회했는데 **행의 개수**가 다름 | A가 `age > 20` 조회 후 B가 새 행 INSERT·커밋 |
| Lost Update | 두 트랜잭션이 같은 값을 읽고 각자 수정해서 한쪽 변경이 사라짐 | 재고 10을 둘 다 읽고 각자 9로 저장 → 실제로는 8이어야 함 |
| Write Skew | 각자 다른 행을 수정했지만 합쳐 보면 규칙이 깨짐 | "당직 의사는 최소 1명" 규칙에서 2명이 동시에 빠짐 |

### 표준 격리 수준

| 격리 수준 | Dirty Read | Non-Repeatable Read | Phantom Read |
| --- | --- | --- | --- |
| READ UNCOMMITTED | 발생 | 발생 | 발생 |
| READ COMMITTED | 방지 | 발생 | 발생 |
| REPEATABLE READ | 방지 | 방지 | 발생 (InnoDB는 대부분 방지) |
| SERIALIZABLE | 방지 | 방지 | 방지 |

| DB | 기본 격리 수준 |
| --- | --- |
| MySQL (InnoDB) | REPEATABLE READ |
| PostgreSQL, Oracle, SQL Server | READ COMMITTED |

### MVCC (Multi-Version Concurrency Control)

InnoDB는 읽기 때문에 쓰기가 막히지 않도록, 행의 **이전 버전을 Undo Log에 보관**하고 트랜잭션마다 볼 수 있는 버전을 골라 보여준다. 일반 `SELECT`는 Lock 없이 이 스냅샷을 읽는다(Consistent Read).

```
row: stock = 8 (trx 200이 커밋)
       └─ undo: stock = 9 (trx 150)
              └─ undo: stock = 10 (trx 100)

trx 180이 SELECT → 자신의 Read View보다 나중에 커밋된 trx 200의 값은 안 보임
                 → undo를 따라가 stock = 9를 읽음
```

| 격리 수준 | Read View(스냅샷)를 만드는 시점 |
| --- | --- |
| READ COMMITTED | **매 SELECT마다** 새로 만듦 → 다른 트랜잭션의 커밋이 바로 보임 |
| REPEATABLE READ | 트랜잭션의 **첫 SELECT 때 한 번** 만듦 → 끝날 때까지 같은 데이터가 보임 |

트랜잭션이 오래 열려 있으면 그 스냅샷 때문에 Undo Log를 지울 수 없어 Undo 영역이 커지고 성능이 떨어진다. **긴 트랜잭션을 피해야 하는 이유** 중 하나다.

### Consistent Read와 Locking Read

InnoDB의 REPEATABLE READ에서 헷갈리는 부분은, **읽는 방식에 따라 보이는 데이터가 다르다**는 점이다.

| 구분 | 쿼리 | 읽는 데이터 |
| --- | --- | --- |
| Consistent Read (Non-locking) | 일반 `SELECT` | 트랜잭션 시작 시점의 스냅샷 |
| Locking Read (Current Read) | `SELECT ... FOR UPDATE`, `FOR SHARE`, `UPDATE`, `DELETE` | **최신 커밋된 데이터** + Lock 획득 |

```sql
-- 세션 A (REPEATABLE READ)
START TRANSACTION;
SELECT stock FROM product WHERE id = 1;            -- 10

-- 세션 B
UPDATE product SET stock = 5 WHERE id = 1; COMMIT;

-- 세션 A
SELECT stock FROM product WHERE id = 1;            -- 여전히 10 (스냅샷)
SELECT stock FROM product WHERE id = 1 FOR UPDATE; -- 5 (최신 값)
UPDATE product SET stock = stock - 1 WHERE id = 1; -- 5 기준으로 4가 됨
```

그래서 MySQL REPEATABLE READ에서도 **Lost Update는 막아주지 않는다.** 애플리케이션에서 읽고(10) → 계산하고(9) → 저장(`SET stock = 9`)하면 다른 트랜잭션의 변경을 덮어쓴다. PostgreSQL의 REPEATABLE READ는 이 상황에서 직렬화 오류를 내고 트랜잭션을 실패시킨다는 점이 다르다.

```java
// 위험: JPA 변경 감지는 "SET stock = 9"처럼 계산된 값을 저장한다
@Transactional
public void decrease(Long id) {
    Product p = productRepository.findById(id).orElseThrow(); // 스냅샷 10
    p.decrease(1);                                            // 9
}                                                             // UPDATE ... SET stock = 9

// 안전: DB가 현재 값 기준으로 계산 (원자적 UPDATE)
@Modifying
@Query("update Product p set p.stock = p.stock - :qty where p.id = :id and p.stock >= :qty")
int decreaseStock(@Param("id") Long id, @Param("qty") int qty); // 0이면 재고 부족
```

### Phantom Read와 InnoDB

- 일반 `SELECT`는 스냅샷을 읽으므로 새로 INSERT된 행이 보이지 않는다.
- `SELECT ... FOR UPDATE` 같은 Locking Read는 **Next-Key Lock**으로 범위 사이(Gap)에 INSERT 자체를 막는다.
- 단, 스냅샷 읽기와 Locking Read를 섞으면 결과가 달라질 수 있으므로 "InnoDB RR은 Phantom이 완전히 없다"고 외우기보다 **읽는 방식에 따라 다르다**고 이해하는 것이 정확하다.

### Spring에서 격리 수준 지정

```java
@Transactional(isolation = Isolation.READ_COMMITTED)
public void settle() { ... }
```

기본값 `Isolation.DEFAULT`는 DB 설정을 따른다. 격리 수준을 올리는 것보다 **필요한 곳에만 Lock이나 원자적 쿼리를 쓰는 것**이 보통 더 낫다. SERIALIZABLE은 InnoDB에서 모든 SELECT를 `FOR SHARE`로 바꾸므로 Lock 대기와 Deadlock이 크게 늘어난다.

## 3. Lock

### 공유 Lock과 배타 Lock

| Lock | 의미 | 획득하는 쿼리 |
| --- | --- | --- |
| 공유 Lock (S, Shared) | 읽는 중이니 **수정하지 마라** (다른 S Lock은 허용) | `SELECT ... FOR SHARE` (`LOCK IN SHARE MODE`) |
| 배타 Lock (X, Exclusive) | 내가 수정할 거니 **다른 Lock 모두 불가** | `SELECT ... FOR UPDATE`, `UPDATE`, `DELETE`, `INSERT` |

| 요청 \ 보유 | S | X |
| --- | --- | --- |
| S | 호환 | 대기 |
| X | 대기 | 대기 |

MVCC 덕분에 **일반 SELECT는 X Lock이 걸린 행도 대기 없이 읽는다.** Lock은 Locking Read·쓰기끼리만 충돌한다.

Lock은 트랜잭션이 **커밋 또는 롤백할 때 해제**된다(2PL, Two-Phase Locking). 트랜잭션이 길면 Lock도 그만큼 오래 잡힌다.

### Lock의 범위

| 단위 | 설명 |
| --- | --- |
| 테이블 Lock | `LOCK TABLES`, DDL 시 메타데이터 Lock(MDL) |
| Intention Lock (IS/IX) | 행 Lock을 걸기 전에 테이블에 "이 테이블의 어떤 행에 Lock을 걸 것"이라고 표시. 테이블 Lock과 행 Lock의 충돌을 빠르게 확인하는 용도 |
| Record Lock | 인덱스 레코드 하나에 거는 Lock |
| Gap Lock | 인덱스 레코드 **사이의 빈 공간**에 거는 Lock. 그 사이에 INSERT를 막음 |
| Next-Key Lock | Record Lock + 그 앞의 Gap Lock. REPEATABLE READ의 기본 행 Lock |
| Insert Intention Lock | INSERT 직전에 거는 Gap Lock의 일종. 같은 Gap에 다른 위치로 INSERT하는 것끼리는 충돌하지 않음 |

```
인덱스 age: 10, 20, 30

SELECT * FROM member WHERE age BETWEEN 15 AND 25 FOR UPDATE;
→ (10, 20], (20, 30] 구간에 Next-Key Lock
→ age = 17, 22, 28 INSERT는 모두 대기
```

READ COMMITTED에서는 Gap Lock을 거의 쓰지 않는다(FK·중복 검사 제외). 그래서 Phantom은 생길 수 있지만 Lock 대기와 Deadlock은 줄어든다.

### Lock은 인덱스에 걸린다

InnoDB는 행 자체가 아니라 **조회에 사용한 인덱스 레코드**에 Lock을 건다. 조건 컬럼에 인덱스가 없으면 풀 스캔하면서 **훑은 모든 행에 Lock**이 걸린다.

```sql
-- status에 인덱스가 없으면 사실상 테이블 전체가 잠김
UPDATE orders SET status = 'CANCELED' WHERE status = 'PENDING' AND created_at < '2026-11-01';
```

대량 UPDATE·DELETE는 조건 컬럼의 인덱스를 확인하고, PK 범위로 나눠서 짧게 여러 번 실행한다.

### Lock 대기 시간

| 설정 | 기본값 | 의미 |
| --- | --- | --- |
| `innodb_lock_wait_timeout` | 50초 | 행 Lock을 기다리는 최대 시간. 초과 시 해당 **문장만** 실패 (트랜잭션은 유지) |
| `lock_wait_timeout` | 1년 | 메타데이터 Lock 대기 시간 |

웹 요청에서 50초 대기는 너무 길다. 비관적 Lock을 쓸 때는 쿼리 단위로 타임아웃을 짧게 주거나 `NOWAIT`/`SKIP LOCKED`를 쓴다.

```sql
SELECT * FROM seat WHERE id = 1 FOR UPDATE NOWAIT;  -- 잠겨 있으면 바로 오류
SELECT * FROM job WHERE status = 'READY' LIMIT 10 FOR UPDATE SKIP LOCKED; -- 잠긴 행은 건너뜀 (작업 큐)
```

### 메타데이터 Lock 주의

운영 중 `ALTER TABLE`은 테이블의 MDL을 기다리고, **그 뒤에 들어오는 일반 SELECT까지 모두 대기**하게 만든다. 오래 열린 트랜잭션 하나가 있으면 DDL이 막히고 서비스 전체가 멈출 수 있다. DDL 전에 `information_schema.innodb_trx`로 긴 트랜잭션을 확인한다.

## 4. Deadlock

두 트랜잭션이 **서로 상대가 가진 Lock을 기다리며** 영원히 진행하지 못하는 상태다.

### 발생 조건 (4가지 모두 만족해야 함)

| 조건 | 의미 |
| --- | --- |
| 상호 배제 | 자원을 한 번에 하나만 사용 |
| 점유와 대기 | 자원을 가진 채 다른 자원을 기다림 |
| 비선점 | 다른 트랜잭션의 자원을 강제로 뺏을 수 없음 |
| 순환 대기 | A → B → A 형태로 대기가 원을 이룸 |

DB에서는 앞의 세 조건은 Lock의 성질상 피할 수 없으므로, **순환 대기를 끊는 것**이 핵심이다.

### 예시 1: Lock 획득 순서가 다름

```
T1: 계좌 1 → 2 송금           T2: 계좌 2 → 1 송금
UPDATE account ... id = 1     
                              UPDATE account ... id = 2
UPDATE account ... id = 2     
  (T2가 가진 2를 기다림)      UPDATE account ... id = 1
                                (T1이 가진 1을 기다림)  → Deadlock
```

해결: **항상 같은 순서로 Lock을 건다.**

```java
@Transactional
public void transfer(Long fromId, Long toId, long amount) {
    Long first = Math.min(fromId, toId);
    Long second = Math.max(fromId, toId);
    Account a = accountRepository.findByIdForUpdate(first);
    Account b = accountRepository.findByIdForUpdate(second);

    Account from = a.getId().equals(fromId) ? a : b;
    Account to = (from == a) ? b : a;
    from.withdraw(amount);
    to.deposit(amount);
}
```

### 예시 2: Gap Lock + INSERT

"없으면 INSERT" 로직에서 자주 발생한다.

```
T1: SELECT ... WHERE code = 'X' FOR UPDATE;  -- 없음 → Gap Lock
T2: SELECT ... WHERE code = 'X' FOR UPDATE;  -- 없음 → Gap Lock (Gap Lock끼리는 호환!)
T1: INSERT code = 'X'  → T2의 Gap Lock 때문에 대기
T2: INSERT code = 'X'  → T1의 Gap Lock 때문에 대기 → Deadlock
```

해결: `SELECT FOR UPDATE` 후 INSERT 대신 **UNIQUE 제약조건 + INSERT 후 중복 예외 처리**, 또는 `INSERT ... ON DUPLICATE KEY UPDATE`를 쓴다.

```java
try {
    couponIssueRepository.saveAndFlush(new CouponIssue(userId, couponId));
} catch (DataIntegrityViolationException e) {
    throw new AlreadyIssuedException();
}
```

### InnoDB의 Deadlock 처리

- `innodb_deadlock_detect = ON`(기본)이면 대기 그래프에서 순환을 즉시 찾아 **변경한 행이 적은 쪽 트랜잭션을 롤백**한다.
- 롤백된 쪽은 `ERROR 1213: Deadlock found when trying to get lock`을 받는다. Spring에서는 `CannotAcquireLockException`(`PessimisticLockingFailureException` 하위)로 변환된다.
- 마지막 Deadlock 정보는 `SHOW ENGINE INNODB STATUS`의 `LATEST DETECTED DEADLOCK`에서 볼 수 있고, 모두 기록하려면 `innodb_print_all_deadlocks = ON`.

### 줄이는 방법

| 방법 | 설명 |
| --- | --- |
| Lock 순서 통일 | ID 정렬 후 Lock, 테이블 접근 순서 통일 |
| 트랜잭션 짧게 | 외부 API 호출, 파일 처리 등을 트랜잭션 밖으로 |
| 인덱스 설계 | 인덱스가 없으면 Lock 범위가 넓어져 충돌 가능성 증가 |
| 격리 수준 | Gap Lock이 문제면 READ COMMITTED 검토 |
| 재시도 | Deadlock은 완전히 없앨 수 없으므로 **트랜잭션 전체를 재시도**할 수 있게 설계 |

## 5. 낙관적 Lock / 비관적 Lock

### 비교

| 구분 | 낙관적 Lock (Optimistic) | 비관적 Lock (Pessimistic) |
| --- | --- | --- |
| 가정 | 충돌이 **드물다** | 충돌이 **잦다** |
| 방식 | Lock 없이 진행하고, 커밋 시 버전으로 충돌 **검사** | 읽을 때 DB Lock을 걸어 충돌을 **예방** |
| 구현 | `version` 컬럼 (애플리케이션 레벨) | `SELECT ... FOR UPDATE` (DB 레벨) |
| 충돌 시 | 예외 발생 → 재시도 또는 사용자에게 알림 | 다른 트랜잭션은 대기 |
| 장점 | Lock 대기 없음, DB 부하 적음 | 데이터 정합성 확실, 재시도 로직 불필요 |
| 단점 | 충돌이 많으면 재시도 비용이 큼 | 대기·Deadlock 가능, 처리량 감소 |
| 적합 | 게시글 수정, 회원 정보 수정, 관리자 설정 | 재고 차감, 좌석 예약, 포인트 차감 |

### 낙관적 Lock (JPA `@Version`)

```java
@Entity
public class Product {
    @Id @GeneratedValue
    private Long id;
    private int stock;

    @Version
    private Long version;
}
```

```sql
-- JPA가 실행하는 UPDATE
UPDATE product SET stock = 9, version = 6 WHERE id = 1 AND version = 5;
-- 영향받은 행이 0이면 다른 트랜잭션이 먼저 수정한 것 → OptimisticLockException
```

Spring에서는 `ObjectOptimisticLockingFailureException`으로 받는다. 재시도는 **새 트랜잭션에서** 다시 읽어야 의미가 있으므로, 재시도 로직은 `@Transactional` 메서드 **바깥**에 둔다.

```java
@Component
@RequiredArgsConstructor
public class StockFacade {
    private final StockService stockService; // @Transactional decrease()

    public void decrease(Long id, int qty) {
        for (int attempt = 1; ; attempt++) {
            try {
                stockService.decrease(id, qty);
                return;
            } catch (ObjectOptimisticLockingFailureException e) {
                if (attempt == 3) throw e;
            }
        }
    }
}
```

HTTP API에서는 버전을 클라이언트에 내려주고 수정 요청에 함께 받으면, "다른 사람이 먼저 수정했습니다"를 `409 Conflict`로 알려줄 수 있다(ETag / `If-Match`와 같은 개념).

### 비관적 Lock (JPA `@Lock`)

```java
public interface ProductRepository extends JpaRepository<Product, Long> {

    @Lock(LockModeType.PESSIMISTIC_WRITE)                 // SELECT ... FOR UPDATE
    @QueryHints(@QueryHint(name = "jakarta.persistence.lock.timeout", value = "3000"))
    @Query("select p from Product p where p.id = :id")
    Optional<Product> findByIdForUpdate(@Param("id") Long id);
}
```

| LockModeType | SQL | 용도 |
| --- | --- | --- |
| `PESSIMISTIC_READ` | `FOR SHARE` | 읽는 동안 수정만 막음 |
| `PESSIMISTIC_WRITE` | `FOR UPDATE` | 읽기(Locking Read)·수정 모두 막음 |
| `PESSIMISTIC_FORCE_INCREMENT` | `FOR UPDATE` + version 증가 | 비관적 Lock과 버전을 함께 사용 |
| `OPTIMISTIC_FORCE_INCREMENT` | 커밋 시 version 증가 | 연관 엔티티만 바뀌어도 부모 버전을 올리고 싶을 때 |

비관적 Lock은 **트랜잭션 안에서만** 의미가 있다. 트랜잭션 없이 호출하면 쿼리 직후 Lock이 풀린다(JPA는 아예 `TransactionRequiredException`).

### 그 밖의 선택지

| 방법 | 설명 | 주의 |
| --- | --- | --- |
| 원자적 UPDATE | `SET stock = stock - 1 WHERE stock >= 1` | 가장 단순하고 빠름. 조건이 복잡하면 어려움 |
| UNIQUE 제약조건 | 중복 생성 방지 (쿠폰 1인 1매, 멱등키) | 예외를 비즈니스 오류로 변환 |
| `synchronized` | JVM 내 스레드 동기화 | 서버가 2대 이상이면 무의미. `@Transactional`과 함께 쓰면 **커밋 전에 Lock이 풀려** 실패 |
| MySQL Named Lock | `GET_LOCK('key', timeout)` | 별도 커넥션 관리 필요 |
| Redis 분산 Lock | Redisson `RLock` 등 | DB 트랜잭션 커밋 후에 Lock을 풀어야 함. Lock 만료 시간 설정 |

```java
// 잘못된 예: synchronized가 끝난 뒤 프록시가 커밋한다
@Transactional
public synchronized void decrease(Long id) { ... }
// 스레드 A가 메서드를 빠져나오는 순간 B가 들어오지만, A는 아직 커밋 전
```

### 선택 기준

1. 원자적 UPDATE나 UNIQUE 제약조건으로 해결되면 그것을 쓴다.
2. 충돌이 드물고 실패 시 사용자가 다시 해도 되면 **낙관적 Lock**.
3. 충돌이 잦고 반드시 순서대로 처리해야 하면 **비관적 Lock** (타임아웃 필수).
4. 여러 DB·외부 시스템에 걸친 자원이면 분산 Lock이나 큐로 직렬화를 검토한다.

## 6. Transaction Propagation

`@Transactional` 메서드가 이미 트랜잭션이 진행 중인 상태에서 호출될 때 **기존 트랜잭션에 참여할지, 새로 만들지**를 정하는 규칙이다.

### Spring 트랜잭션 동작 원리

```
Client → [Proxy] → OrderService (실제 객체)
           │
           ├─ TransactionManager.getTransaction()   (커넥션 획득, autocommit=false)
           ├─ target.placeOrder() 호출
           ├─ 정상 종료 → commit
           └─ RuntimeException → rollback
```

같은 스레드 안에서는 `TransactionSynchronizationManager`가 ThreadLocal에 커넥션을 보관해, 여러 Repository가 **같은 커넥션(같은 트랜잭션)** 을 쓰게 된다.

### 전파 옵션

| 옵션 | 기존 트랜잭션 있음 | 기존 트랜잭션 없음 |
| --- | --- | --- |
| `REQUIRED` (기본값) | 참여 | 새로 생성 |
| `REQUIRES_NEW` | 기존을 **보류**하고 새로 생성 (별도 커넥션) | 새로 생성 |
| `SUPPORTS` | 참여 | 트랜잭션 없이 실행 |
| `NOT_SUPPORTED` | 기존을 보류하고 트랜잭션 없이 실행 | 트랜잭션 없이 실행 |
| `MANDATORY` | 참여 | 예외 |
| `NEVER` | 예외 | 트랜잭션 없이 실행 |
| `NESTED` | Savepoint를 만들고 중첩 실행 (부분 롤백 가능) | 새로 생성 |

### 물리 트랜잭션과 논리 트랜잭션

`REQUIRED`로 여러 메서드가 참여하면 **실제 DB 트랜잭션(물리)은 하나**이고, 각 메서드는 논리 트랜잭션이다.

- 모든 논리 트랜잭션이 커밋되어야 물리 트랜잭션이 커밋된다.
- 하나라도 롤백되면 물리 트랜잭션은 롤백된다.
- 실제 commit/rollback은 **처음 트랜잭션을 시작한 쪽**(신규 트랜잭션)만 수행한다.

### 자주 만나는 함정: `UnexpectedRollbackException`

```java
@Service
public class OrderService {
    @Transactional
    public void placeOrder(OrderRequest req) {
        orderRepository.save(...);
        try {
            pointService.earn(req.userId(), 100);   // REQUIRED로 참여
        } catch (RuntimeException e) {
            log.warn("포인트 적립 실패", e);         // 예외를 잡았으니 주문은 커밋될까?
        }
    }
}

@Service
public class PointService {
    @Transactional
    public void earn(Long userId, int amount) {
        throw new IllegalStateException();
    }
}
```

`earn()`의 프록시가 예외를 보고 **물리 트랜잭션에 rollback-only 표시**를 한다. `placeOrder()`가 예외를 잡고 정상 종료해도 커밋 시점에 표시를 확인하고 롤백한 뒤 `UnexpectedRollbackException`을 던진다. 주문까지 사라진다.

포인트 실패와 무관하게 주문을 남기려면 다음 중 하나를 쓴다.

```java
// 1) 포인트를 별도 트랜잭션으로 분리
@Transactional(propagation = Propagation.REQUIRES_NEW)
public void earn(Long userId, int amount) { ... }

// 2) 주문 커밋 후에 처리 (이벤트)
@TransactionalEventListener(phase = TransactionPhase.AFTER_COMMIT)
public void onOrderPlaced(OrderPlacedEvent event) { pointService.earn(...); }
```

### `REQUIRES_NEW` 사용 시 주의

- 외부 트랜잭션의 커넥션을 쥔 채로 **커넥션을 하나 더** 가져온다. 동시 요청이 많으면 커넥션 풀이 바닥나 서로 기다리는 상태(풀 Deadlock)가 될 수 있다. 커넥션 풀 크기를 고려한다.
- 내부 트랜잭션은 먼저 커밋되므로, 외부가 나중에 롤백되어도 내부 변경은 남는다. 실패 로그·감사 기록처럼 **반드시 남아야 하는 데이터**에 적합하다.
- 내부에서 외부 트랜잭션이 Lock을 건 행을 수정하면, 외부는 내부가 끝나길 기다리고 내부는 외부의 Lock을 기다려 **자기 자신과 Deadlock**이 된다.

### `NESTED`

Savepoint를 이용해 내부만 롤백할 수 있다. 내부가 롤백되어도 외부는 계속 진행할 수 있고, 외부가 롤백되면 내부도 함께 롤백된다. JDBC Savepoint를 지원하는 `DataSourceTransactionManager`에서 동작하며, **JPA(Hibernate) 환경의 `JpaTransactionManager`에서는 기본적으로 지원되지 않는다.**

### `@Transactional`이 적용되지 않는 경우

| 상황 | 이유 | 해결 |
| --- | --- | --- |
| 같은 클래스 내부 호출 (`this.inner()`) | 프록시를 거치지 않음 | 별도 빈으로 분리 |
| `private` 메서드 | 프록시가 오버라이드할 수 없음 | public으로, 또는 구조 변경 |
| Checked Exception 발생 | 기본적으로 `RuntimeException`·`Error`만 롤백 | `rollbackFor = Exception.class` |
| 예외를 메서드 안에서 잡고 끝냄 | 프록시가 예외를 못 봄 | 다시 던지거나 `setRollbackOnly()` |
| 새 스레드·`@Async`에서 실행 | 트랜잭션은 ThreadLocal에 묶임 | 해당 메서드에 별도 `@Transactional` |

### readOnly

```java
@Transactional(readOnly = true)
public OrderResponse getOrder(Long id) { ... }
```

- Hibernate가 플러시와 변경 감지용 스냅샷을 생략해 메모리·CPU를 아낀다.
- MySQL 드라이버에 `SET SESSION TRANSACTION READ ONLY`를 보내 실수로 쓰는 것을 막는다.
- `LazyConnectionDataSourceProxy` + `AbstractRoutingDataSource`와 함께 쓰면 **Read Replica로 라우팅**하는 기준이 된다. 단 복제 지연이 있으므로 방금 쓴 데이터를 바로 읽어야 하는 경우는 주의한다.

### 트랜잭션 범위 설계

```java
// 나쁜 예: 외부 API 응답을 기다리는 동안 커넥션과 Lock을 쥐고 있음
@Transactional
public void pay(Long orderId) {
    Order order = orderRepository.findByIdForUpdate(orderId);
    PgResult result = pgClient.approve(order);   // 수 초 걸릴 수 있음
    order.complete(result);
}

// 나은 예: 트랜잭션은 DB 작업만 짧게
public void pay(Long orderId) {
    Order order = orderService.startPayment(orderId);    // @Transactional: 상태 PAYING
    PgResult result = pgClient.approve(order);           // 트랜잭션 밖
    orderService.completePayment(orderId, result);       // @Transactional: 상태 PAID
}
```

외부 호출이 성공했는데 DB 반영이 실패하는 경우를 대비해, 결제 상태를 중간 상태(`PAYING`)로 남기고 **멱등키 + 재조회/보상 처리**로 맞춘다. 하나의 DB 트랜잭션으로 외부 시스템까지 묶을 수는 없다.

## 7. Backend에서 확인할 점

- "읽고 → 계산하고 → 저장"하는 코드는 동시 요청에서 Lost Update가 생긴다. MySQL REPEATABLE READ가 이를 막아주지 않는다는 점을 기억하고, 원자적 UPDATE·`@Version`·`FOR UPDATE` 중 하나를 고른다.
- 중복 생성(쿠폰, 결제, 회원가입)은 애플리케이션 검사만 믿지 말고 **UNIQUE 제약조건**을 둔다. "조회 후 없으면 INSERT"는 Gap Lock Deadlock의 단골이다.
- `FOR UPDATE`의 조건 컬럼에 인덱스가 있는지 확인한다. 없으면 훨씬 많은 행이 잠긴다.
- 비관적 Lock에는 타임아웃을 건다. 기본 50초 대기는 웹 요청에 맞지 않는다.
- 여러 행을 잠글 때는 **항상 같은 순서**로 잠그고, Deadlock·낙관적 Lock 충돌은 트랜잭션 바깥에서 재시도할 수 있게 만든다.
- `synchronized`나 로컬 Lock은 서버가 여러 대면 의미가 없고, `@Transactional`과 함께 쓰면 커밋 전에 풀린다.
- 트랜잭션 안에서 외부 API 호출, 메시지 발행, 파일 업로드를 하지 않는다. 커밋 후 처리는 `@TransactionalEventListener(AFTER_COMMIT)`이나 Outbox 패턴을 쓴다.
- 내부 `REQUIRED` 메서드에서 난 예외를 잡아도 전체가 롤백된다(`UnexpectedRollbackException`). 독립적으로 남겨야 하면 `REQUIRES_NEW`를 쓰되 커넥션 풀 크기를 함께 본다.
- 같은 클래스 내부 호출, private 메서드, Checked Exception에서는 `@Transactional`이 기대대로 동작하지 않는다.
- 동시성 로직은 `ExecutorService` + `CountDownLatch`로 동시 요청 테스트를 작성해 실제로 재고·잔액이 맞는지 확인한다.

```java
@Test
void 동시에_100명이_주문해도_재고가_정확하다() throws InterruptedException {
    int threads = 100;
    ExecutorService executor = Executors.newFixedThreadPool(32);
    CountDownLatch latch = new CountDownLatch(threads);

    for (int i = 0; i < threads; i++) {
        executor.submit(() -> {
            try { stockService.decreaseWithPessimisticLock(productId, 1); } // 낙관적 Lock이면 재시도 횟수가 부족해 실패할 수 있음
            finally { latch.countDown(); }
        });
    }
    latch.await();

    assertThat(productRepository.findById(productId).orElseThrow().getStock()).isZero(); // 초기 재고 100
}
```
