# 11주차 - Transaction / 동시성

## 1. ACID

| 속성 | 의미 | 구현 |
|---|---|---|
| Atomicity (원자성) | 전부 성공 or 전부 롤백 | Undo Log |
| Consistency (일관성) | 트랜잭션 전후로 제약조건(PK, FK, 도메인 규칙) 유지 | 제약조건 + 애플리케이션 로직 |
| Isolation (격리성) | 동시 실행되는 트랜잭션끼리 간섭 X | Lock, MVCC |
| Durability (지속성) | 커밋된 데이터는 장애가 나도 유지 | Redo Log (WAL) |

## 2. Isolation Level

격리 수준이 높을수록 정합성은 좋아지지만 동시성(성능)은 떨어짐.

### 발생 가능한 이상 현상
- **Dirty Read**: 커밋 안 된 데이터를 읽음
- **Non-Repeatable Read**: 같은 row를 두 번 읽었는데 값이 다름 (UPDATE)
- **Phantom Read**: 같은 조건으로 조회했는데 row 개수가 달라짐 (INSERT/DELETE)

### 수준별 비교

| Isolation Level | Dirty Read | Non-Repeatable Read | Phantom Read |
|---|---|---|---|
| READ UNCOMMITTED | O | O | O |
| READ COMMITTED | X | O | O |
| REPEATABLE READ | X | X | O (MySQL InnoDB는 대부분 방지) |
| SERIALIZABLE | X | X | X |

- MySQL(InnoDB) 기본값: `REPEATABLE READ`
- PostgreSQL / Oracle 기본값: `READ COMMITTED`
- **MVCC**: 읽기 시 Lock 대신 Undo Log의 스냅샷 버전을 읽어서 읽기/쓰기가 서로 블로킹되지 않게 함

## 3. Lock

- **Shared Lock (S, 공유 락)**: 읽기용. S끼리는 공존 가능, X와는 불가
- **Exclusive Lock (X, 배타 락)**: 쓰기용. 다른 S/X 모두 대기
- `SELECT ... FOR SHARE` → S Lock
- `SELECT ... FOR UPDATE` → X Lock
- InnoDB는 인덱스 기반으로 Record Lock / Gap Lock / Next-Key Lock을 사용
  - 인덱스를 타지 않는 조건으로 락을 걸면 **테이블 전체 락**처럼 동작할 수 있음 → 주의

## 4. Deadlock

두 트랜잭션이 서로 상대가 가진 Lock을 기다리며 영원히 대기하는 상태.

```text
TX1: lock(A) → lock(B) 대기
TX2: lock(B) → lock(A) 대기
```

### 해결/예방
- 여러 자원을 잠글 때 **항상 같은 순서**로 Lock 획득 (예: id 오름차순)
- 트랜잭션을 **짧게** 유지, 외부 API 호출은 트랜잭션 밖으로 분리
- Lock timeout 설정 (`innodb_lock_wait_timeout`)
- DB가 Deadlock 감지 시 한쪽을 victim으로 롤백 → 애플리케이션에서 **재시도 로직** 필요

## 5. 낙관적 Lock vs 비관적 Lock

| | 낙관적 Lock | 비관적 Lock |
|---|---|---|
| 가정 | 충돌이 드물다 | 충돌이 자주 난다 |
| 방식 | Lock 없이 진행, 커밋 시 version 비교 | 조회 시점에 DB Lock 획득 |
| 충돌 시 | 예외 발생 → 재시도 | 먼저 잡은 쪽이 끝날 때까지 대기 |
| 장점 | 동시성 높음, Deadlock 없음 | 정합성 확실 |
| 단점 | 충돌 많으면 재시도 비용 큼 | 대기/Deadlock 위험, 처리량 감소 |

### JPA 예시

```java
// 낙관적 Lock: @Version
@Entity
public class Product {
    @Id @GeneratedValue
    private Long id;
    private int stock;

    @Version
    private Long version; // UPDATE ... WHERE id=? AND version=?
}
// 충돌 시 ObjectOptimisticLockingFailureException 발생
```

```java
// 비관적 Lock: SELECT ... FOR UPDATE
public interface ProductRepository extends JpaRepository<Product, Long> {

    @Lock(LockModeType.PESSIMISTIC_WRITE)
    @Query("select p from Product p where p.id = :id")
    Optional<Product> findByIdForUpdate(@Param("id") Long id);
}
```

> 재고 차감처럼 충돌이 잦은 경우 → 비관적 Lock 또는 Redis 분산 락 고려

## 6. Transaction Propagation (전파)

이미 진행 중인 트랜잭션이 있을 때, `@Transactional` 메서드를 호출하면 어떻게 동작할지 정하는 옵션.

| 옵션 | 동작 |
|---|---|
| `REQUIRED` (기본) | 기존 TX 있으면 참여, 없으면 새로 생성 |
| `REQUIRES_NEW` | 항상 새 TX 생성 (기존 TX는 잠시 중단) |
| `SUPPORTS` | TX 있으면 참여, 없으면 TX 없이 실행 |
| `MANDATORY` | TX 없으면 예외 |
| `NOT_SUPPORTED` | TX 없이 실행 (기존 TX는 중단) |
| `NEVER` | TX 있으면 예외 |
| `NESTED` | 중첩 TX (Savepoint 사용) |

### 주의할 점
- `REQUIRED`에서 내부 메서드가 예외로 롤백되면 **전체 TX가 rollback-only**로 마킹됨
  → 외부에서 예외를 catch해도 커밋 시 `UnexpectedRollbackException` 발생
- 로그/이력 저장처럼 **본 로직이 롤백돼도 남겨야 하는 작업**은 `REQUIRES_NEW` 사용
- **같은 클래스 내부 호출(self-invocation)** 은 프록시를 거치지 않아 `@Transactional`이 적용되지 않음

```java
@Service
public class OrderService {

    @Transactional
    public void order() {
        // 주문 처리
        auditService.saveLog(); // REQUIRES_NEW → 별도 TX
    }
}

@Service
public class AuditService {

    @Transactional(propagation = Propagation.REQUIRES_NEW)
    public void saveLog() {
        // order()가 롤백돼도 이 로그는 커밋됨
    }
}
```

