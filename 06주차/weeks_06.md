# 6주차 - Database (Backend)

예시는 MySQL 8.0(InnoDB) 기준이며, 아래 테이블을 사용한다.

```sql
CREATE TABLE orders (
    id         BIGINT PRIMARY KEY AUTO_INCREMENT,
    user_id    BIGINT      NOT NULL,
    status     VARCHAR(20) NOT NULL,
    amount     INT         NOT NULL,
    created_at DATETIME    NOT NULL
);
```

## 1. Composite / Covering Index

### InnoDB의 인덱스 구조

InnoDB는 데이터를 16KB 크기의 **페이지** 단위로 저장하고, 인덱스는 B+Tree로 구성한다. B+Tree는 leaf 노드에만 실제 데이터가 있고 leaf끼리 연결 리스트로 이어져 있어서, 시작 지점만 찾으면 옆 leaf로 이동하며 범위를 읽을 수 있다.

| 구분 | Clustered Index | Secondary Index |
| --- | --- | --- |
| 개수 | 테이블당 1개 (PK) | 여러 개 |
| leaf 노드에 저장되는 것 | **행 전체** | **인덱스 컬럼 값 + PK 값** |
| 정렬 기준 | PK | 인덱스 컬럼 (같으면 PK) |

테이블 데이터 자체가 PK 순서로 정렬된 Clustered Index다. 그래서 Secondary Index로 조회하면 보통 두 번 탐색한다.

1. Secondary Index에서 조건에 맞는 PK를 찾는다.
2. 그 PK로 Clustered Index를 다시 탐색해 실제 행을 읽는다. (테이블 접근)

1번은 정렬된 leaf를 순서대로 읽지만, 2번은 PK가 여기저기 흩어져 있어 행마다 다른 페이지를 읽는 **Random I/O**가 된다. 읽어야 할 행이 많을수록 2번 비용이 커진다.

이 구조에서 나오는 결론은 다음과 같다.

- PK가 길면 **모든 Secondary Index가 커진다.** PK 값이 leaf마다 들어가기 때문이다.
- `AUTO_INCREMENT`처럼 증가하는 PK는 항상 마지막 페이지에 삽입된다. UUID처럼 무작위인 PK는 중간 페이지에 삽입되어 페이지 분할(Page Split)이 자주 일어난다.
- 2번 단계를 줄이거나 없애는 것이 조회 최적화의 핵심이다. (Covering Index)

### Composite Index

여러 컬럼을 묶어 만든 인덱스다.

```sql
CREATE INDEX idx_user_status_created ON orders (user_id, status, created_at);
```

인덱스의 leaf는 **첫 번째 컬럼으로 정렬하고, 같으면 두 번째, 또 같으면 세 번째** 순으로 정렬되어 있다.

| user_id | status | created_at | (PK) id |
| --- | --- | --- | --- |
| 1 | CANCELED | 2026-03-02 | 17 |
| 1 | PAID | 2026-01-10 | 3 |
| 1 | PAID | 2026-04-21 | 42 |
| 2 | PAID | 2026-02-01 | 9 |
| 2 | SHIPPED | 2026-01-05 | 2 |

`status`만 보면 정렬되어 있지 않고(`CANCELED, PAID, PAID, PAID, SHIPPED`가 `user_id` 경계를 넘어 섞임), `user_id`가 정해진 범위 안에서만 정렬되어 있다. 그래서 **왼쪽 컬럼부터 조건에 있어야** 인덱스를 탐색에 쓸 수 있다(Leftmost Prefix).

| WHERE 조건 | 인덱스 사용 | key_len |
| --- | --- | --- |
| `user_id = 1` | O | 8 |
| `user_id = 1 AND status = 'PAID'` | O | 90 |
| `user_id = 1 AND status = 'PAID' AND created_at >= '2026-01-01'` | O (세 컬럼 모두) | 95 |
| `status = 'PAID'` | X (선두 컬럼 없음) | - |
| `user_id = 1 AND created_at >= '2026-01-01'` | △ (`user_id`까지만 범위를 좁힘) | 8 |

`key_len`은 `EXPLAIN`이 보여주는 "사용한 인덱스의 바이트 길이"다. `BIGINT` 8바이트, `VARCHAR(20)`(utf8mb4)은 20×4+2=82바이트, `DATETIME` 5바이트이므로 8 / 90 / 95로 몇 번째 컬럼까지 썼는지 알 수 있다. (NULL 허용 컬럼은 1바이트가 더 붙는다.)

#### 범위 조건 뒤의 컬럼

**범위 조건(`>`, `<`, `BETWEEN`, `LIKE 'a%'`) 뒤의 컬럼**은 탐색 범위를 좁히는 데 쓰이지 못한다. 범위에 걸린 값들 전체로 보면 그다음 컬럼이 정렬되어 있지 않기 때문이다.

```sql
-- 인덱스 (user_id, created_at, status)라면
-- created_at이 범위 조건이므로 status는 범위를 좁히지 못한다.
WHERE user_id = 1 AND created_at >= '2026-01-01' AND status = 'PAID'
```

다만 `status`가 인덱스 안에 들어 있기는 하므로, 테이블에 접근하기 전에 인덱스에서 먼저 걸러낼 수는 있다. 이를 **Index Condition Pushdown(ICP)**이라 하고 `Extra`에 `Using index condition`으로 표시된다. 범위를 좁히는 것보다는 못하지만 테이블 접근 횟수를 줄여 준다.

`IN`은 범위가 아니라 **동등 조건 여러 개**로 처리된다. `user_id = 1 AND status IN ('PAID', 'SHIPPED') AND created_at >= ...`는 세 컬럼을 모두 사용할 수 있다.

#### 정렬과 인덱스

인덱스는 이미 정렬되어 있으므로, `ORDER BY`가 인덱스 순서와 맞으면 별도의 정렬(filesort)을 생략한다. `LIMIT`이 있으면 필요한 건수만 읽고 멈출 수 있어 효과가 크다.

```sql
-- (user_id, status, created_at) 순서 그대로 → filesort 없음, 20건만 읽고 종료
SELECT * FROM orders
WHERE user_id = 1 AND status = 'PAID'
ORDER BY created_at DESC
LIMIT 20;
```

| 쿼리 | 정렬에 인덱스 사용 |
| --- | --- |
| `WHERE user_id = 1 AND status = 'PAID' ORDER BY created_at` | O |
| `WHERE user_id = 1 ORDER BY status, created_at` | O |
| `WHERE user_id = 1 ORDER BY created_at` | X (`status`를 건너뜀) |
| `WHERE user_id = 1 AND status IN ('PAID', 'SHIPPED') ORDER BY created_at` | X (`status` 값별로 따로 정렬되어 있음) |
| `WHERE user_id = 1 ORDER BY status ASC, created_at DESC` | X (정렬 방향이 섞임. 내림차순 인덱스 필요) |

역순 정렬(`DESC`)은 인덱스를 뒤에서부터 읽으면 되므로 가능하다(`Backward index scan`). 방향이 섞인 정렬은 `(status ASC, created_at DESC)`처럼 방향을 지정한 인덱스가 있어야 한다.

#### 컬럼 순서를 정하는 기준

1. **동등 조건(`=`, `IN`) 컬럼을 앞에** 둔다.
2. 그 뒤에 **정렬 컬럼**을 둔다.
3. **범위 조건 컬럼은 마지막**에 둔다.
4. 여러 쿼리가 공통으로 가진 조건을 선두에 두어 인덱스 하나로 여러 쿼리를 처리한다.

"카디널리티가 높은 컬럼을 앞에"라는 기준도 흔히 쓰지만, 동등 조건끼리는 순서가 바뀌어도 좁혀지는 범위가 같다. 동등/범위 구분과 쿼리 패턴이 더 중요한 기준이다.

### Covering Index

쿼리에 필요한 컬럼이 **모두 인덱스에 들어 있어** 테이블(Clustered Index)에 접근하지 않고 인덱스만으로 결과를 만들 수 있는 경우다. 앞에서 본 2번 단계(Random I/O)가 사라진다.

```sql
-- 필요한 컬럼(user_id, status, created_at, id)이 모두 인덱스에 있음 → Covering
SELECT id, status, created_at FROM orders WHERE user_id = 1;

-- amount는 인덱스에 없음 → 행마다 테이블 접근 필요
SELECT id, status, amount FROM orders WHERE user_id = 1;
```

- Secondary Index는 PK를 항상 포함하므로 `id`는 인덱스에 따로 넣지 않아도 된다.
- SELECT 절뿐 아니라 WHERE, ORDER BY, GROUP BY, JOIN 조건에 쓰인 컬럼도 모두 인덱스에 있어야 한다.
- Covering Index로 처리되면 `EXPLAIN`의 `Extra`에 `Using index`가 표시된다.
- `SELECT *`는 Covering Index를 쓸 수 없게 만드는 대표적인 원인이다.

`COUNT(*)`나 존재 여부 확인(`EXISTS`)처럼 컬럼 값이 필요 없는 쿼리는 Covering Index로 처리되기 쉽다.

#### Covering Index를 활용한 페이징 (Deferred Join)

뒤쪽 페이지를 `OFFSET`으로 조회하면 버려질 행까지 모두 테이블에서 읽는다. 먼저 **인덱스만으로 PK를 추려낸 뒤**, 그 PK에 해당하는 행만 테이블에서 읽으면 테이블 접근이 20번으로 줄어든다.

```sql
SELECT o.*
FROM orders o
JOIN (
    SELECT id FROM orders          -- (user_id, status, created_at) + PK → Covering
    WHERE user_id = 1 AND status = 'PAID'
    ORDER BY created_at DESC
    LIMIT 20 OFFSET 10000
) t ON o.id = t.id
ORDER BY o.created_at DESC;
```

### 인덱스의 비용

| 비용 | 내용 |
| --- | --- |
| 쓰기 성능 | `INSERT`/`UPDATE`/`DELETE` 때마다 관련 인덱스를 모두 수정 |
| 저장 공간 | 인덱스마다 별도의 B+Tree |
| 메모리 | Buffer Pool을 인덱스 페이지가 차지해 데이터 페이지가 밀려남 |
| 옵티마이저 | 후보 인덱스가 많을수록 실행 계획 선택이 어려워지고 잘못 고를 가능성이 생김 |

- `(a)`와 `(a, b)`가 있으면 `(a)`는 대부분 중복이다.
- Covering을 위해 컬럼을 계속 추가하면 인덱스가 테이블만큼 커진다. 호출 빈도가 높은 핵심 쿼리에만 적용한다.
- 인덱스를 지우기 전에 `ALTER TABLE orders ALTER INDEX idx_name INVISIBLE;`로 옵티마이저에게만 숨겨 영향을 확인할 수 있다.

## 2. Query Plan과 EXPLAIN

### 옵티마이저와 실행 계획

SQL은 **무엇을** 가져올지만 적고, **어떻게** 가져올지는 옵티마이저가 정한다. 같은 결과를 내는 방법이 여러 가지(어떤 인덱스를 쓸지, Full Scan을 할지, JOIN 순서를 어떻게 할지)이기 때문이다.

```
SQL → Parser(문법 검사) → Optimizer(통계 기반 비용 계산) → 실행 계획 → Storage Engine(InnoDB)
```

MySQL 옵티마이저는 **비용 기반**이다. 각 방법이 읽어야 할 페이지·행 수를 추정해 비용을 계산하고 가장 싼 계획을 고른다. 이 추정은 **통계 정보**에 의존한다.

- InnoDB는 인덱스 페이지 일부를 표본으로 뽑아 테이블의 행 수와 인덱스별 카디널리티(서로 다른 값의 수)를 추정한다.
- 통계는 데이터가 일정 비율 이상 바뀌면 자동으로 갱신되고, `ANALYZE TABLE orders;`로 직접 갱신할 수 있다.
- 표본 기반이므로 실제와 다를 수 있다. 대량 적재·삭제 직후나 값이 한쪽으로 쏠린 컬럼에서 실행 계획이 잘못 잡히는 원인이 된다.

### EXPLAIN 읽기

`EXPLAIN`은 쿼리를 실행하지 않고 실행 계획만 보여준다.

```sql
EXPLAIN
SELECT id, status, created_at FROM orders
WHERE user_id = 1 AND status = 'PAID';
```

```
id | select_type | table  | type | possible_keys           | key                     | key_len | ref         | rows | filtered | Extra
 1 | SIMPLE      | orders | ref  | idx_user_status_created | idx_user_status_created | 90      | const,const |   35 |   100.00 | Using index
```

읽는 법: `idx_user_status_created` 인덱스를 두 번째 컬럼까지(`key_len` 90) 동등 조건으로 탐색하고(`ref`), 35행 정도를 읽을 것으로 예상하며, 테이블 접근 없이 인덱스만으로 처리한다(`Using index`).

| 컬럼 | 의미 |
| --- | --- |
| `id` | SELECT의 순번. 같은 `id`의 여러 행은 JOIN이며 **위에 나온 테이블부터** 읽는다 |
| `select_type` | SELECT의 종류 |
| `table` | 접근하는 테이블 |
| `type` | 테이블 접근 방식 |
| `possible_keys` | 사용할 수 있는 후보 인덱스 |
| `key` | 실제로 선택한 인덱스. `NULL`이면 인덱스를 쓰지 않음 |
| `key_len` | 사용한 인덱스의 바이트 길이 |
| `ref` | 인덱스와 비교하는 대상 (상수 `const`, 또는 JOIN 상대 컬럼) |
| `rows` | 읽을 것으로 **예상**하는 행 수 |
| `filtered` | 읽은 행 중 나머지 조건을 통과할 것으로 예상하는 비율(%) |
| `Extra` | 추가 정보 |

`rows × filtered / 100`이 다음 단계(JOIN이라면 다음 테이블)로 넘어가는 예상 행 수다. `rows`는 큰데 `filtered`가 낮다면 많이 읽고 대부분 버린다는 뜻이므로 인덱스가 조건을 제대로 받쳐 주지 못하는 것이다.

#### select_type

| select_type | 의미 |
| --- | --- |
| `SIMPLE` | 서브쿼리·UNION이 없는 단순 SELECT |
| `PRIMARY` | 가장 바깥쪽 SELECT |
| `SUBQUERY` | 바깥 쿼리와 무관하게 한 번만 실행되는 서브쿼리 |
| `DEPENDENT SUBQUERY` | 바깥 쿼리의 행마다 다시 실행되는 서브쿼리 (주의 대상) |
| `DERIVED` | FROM 절의 서브쿼리. 임시 테이블로 만들어짐 |
| `UNION` | UNION의 두 번째 이후 SELECT |

#### type

위에서 아래로 갈수록 느린 접근 방식이다.

| type | 의미 |
| --- | --- |
| `const` | PK 또는 Unique Index로 한 건 조회 |
| `eq_ref` | JOIN에서 앞 테이블의 행마다 PK/Unique Index로 한 건씩 매칭 |
| `ref` | Unique가 아닌 인덱스로 동등 조건 조회 |
| `range` | 인덱스 범위 스캔 (`>`, `<`, `BETWEEN`, `IN`, `LIKE 'a%'`) |
| `index_merge` | 인덱스 여러 개를 각각 탐색한 뒤 결과를 합침 |
| `index` | 인덱스 전체 스캔 (Index Full Scan) |
| `ALL` | 테이블 전체 스캔 (Full Table Scan) |

- `index`는 이름과 달리 인덱스를 처음부터 끝까지 읽는다는 뜻이므로 좋은 신호가 아니다.
- `ALL`이 항상 나쁜 것은 아니다. 작은 테이블이거나 대부분의 행을 읽어야 한다면 Full Scan이 맞는 선택이다.
- `index_merge`는 보통 적절한 Composite Index가 없다는 신호다.

#### Extra

| Extra | 의미 |
| --- | --- |
| `Using index` | Covering Index로 처리. 테이블 접근 없음 |
| `Using index condition` | 인덱스에 있는 컬럼의 조건을 스토리지 엔진에서 먼저 걸러냄 (ICP) |
| `Using where` | 읽어온 행을 MySQL 엔진이 추가로 필터링 |
| `Using filesort` | 인덱스 순서를 쓰지 못해 별도 정렬 수행 |
| `Using temporary` | 임시 테이블 사용 (`GROUP BY`, `DISTINCT`, `UNION` 등) |
| `Using join buffer (hash join)` | JOIN 컬럼에 쓸 인덱스가 없어 Hash Join 수행 |
| `Backward index scan` | 인덱스를 역순으로 읽음 |

`Using filesort`는 이름과 달리 파일을 쓴다는 뜻이 아니라 "정렬 작업을 한다"는 뜻이다. 정렬 대상이 작으면 메모리에서 끝나지만, 크면 디스크를 사용한다. `Using filesort`·`Using temporary`가 큰 결과 집합에서 함께 나타나면 우선 확인 대상이다.

### JOIN의 실행 계획

MySQL의 기본 JOIN 방식은 **Nested Loop Join**이다. 먼저 읽는 테이블(Driving Table)의 행마다 다음 테이블(Driven Table)을 조회한다.

```sql
EXPLAIN
SELECT u.name, o.amount
FROM users u JOIN orders o ON o.user_id = u.id
WHERE u.grade = 'VIP';
```

```
id | table | type | key                     | ref  | rows
 1 | u     | ref  | idx_grade               | const|  100
 1 | o     | ref  | idx_user_status_created | u.id |   35
```

`users`에서 100행을 읽고, 행마다 `orders`를 인덱스로 35행씩 조회한다(약 100 × 35행).

- **Driven Table의 JOIN 컬럼에 인덱스가 있어야 한다.** 없으면 앞 테이블의 행마다 Full Scan을 하거나 Hash Join으로 넘어간다.
- 옵티마이저는 보통 조건으로 **행을 더 많이 줄일 수 있는 테이블**을 Driving Table로 고른다. SQL에 쓴 순서와 다를 수 있다.
- JOIN 컬럼의 타입이나 문자셋·콜레이션이 서로 다르면 형 변환이 일어나 인덱스를 쓰지 못한다.

### EXPLAIN ANALYZE

`EXPLAIN`의 `rows`는 추정값이다. `EXPLAIN ANALYZE`(MySQL 8.0.18+)는 쿼리를 **실제로 실행**해 단계별 실제 행 수와 소요 시간을 트리 형태로 보여준다.

```
-> Limit: 20 row(s)  (cost=12.5 rows=20) (actual time=0.05..0.12 rows=20 loops=1)
    -> Index lookup on orders using idx_user_status_created (user_id=1, status='PAID')
       (cost=12.5 rows=35) (actual time=0.04..0.11 rows=20 loops=1)
```

| 항목 | 의미 |
| --- | --- |
| `cost`, 앞쪽 `rows` | 옵티마이저의 추정 비용과 추정 행 수 |
| `actual time=A..B` | 첫 행을 반환하기까지 A ms, 모든 행을 반환하기까지 B ms |
| 뒤쪽 `rows` | 실제로 반환한 행 수 |
| `loops` | 이 단계가 반복 실행된 횟수 (JOIN의 안쪽 테이블은 바깥 행 수만큼 반복) |

- 들여쓰기가 깊은 쪽부터 실행된다.
- 추정 `rows`와 실제 `rows`의 차이가 크면 통계가 틀린 것이므로 `ANALYZE TABLE`로 갱신한다.
- 실제로 실행되므로 운영 DB에서 무거운 쿼리에 쓸 때는 주의한다. `UPDATE`/`DELETE`에 쓰면 데이터가 실제로 바뀐다.

## 3. Query 최적화

### 느린 쿼리 찾기

추측으로 인덱스를 만들지 않고, 실제로 느린 쿼리를 먼저 찾는다.

```ini
# my.cnf
slow_query_log = ON
long_query_time = 1      # 1초 이상 걸린 쿼리를 기록 (기본값 10초)
```

Slow Query Log에는 실행 시간과 함께 `Rows_examined`(읽은 행 수)와 `Rows_sent`(반환한 행 수)가 남는다. **읽은 행 수가 반환한 행 수보다 훨씬 많다면** 인덱스가 조건을 제대로 좁히지 못하고 있다는 뜻이다.

한 번 실행에 0.05초여도 초당 수백 번 호출되는 쿼리가 가장 큰 부하일 수 있다. 실행 시간뿐 아니라 **호출 빈도 × 실행 시간**으로 본다.

### 인덱스를 못 쓰게 만드는 조건

| 패턴 | 문제 | 개선 |
| --- | --- | --- |
| `WHERE DATE(created_at) = '2026-10-01'` | 컬럼을 가공하면 인덱스의 정렬 순서를 쓸 수 없음 | `created_at >= '2026-10-01' AND created_at < '2026-10-02'` |
| `WHERE amount * 2 > 1000` | 위와 동일 | `amount > 500` |
| `WHERE phone = 01012345678` (phone이 VARCHAR) | 묵시적 형 변환으로 컬럼 쪽이 숫자로 변환됨 | `phone = '01012345678'` |
| `WHERE name LIKE '%kim'` | 앞부분이 정해지지 않아 범위 탐색 불가 | `LIKE 'kim%'` 또는 Full-Text 검색 |
| `WHERE user_id = 1 OR amount > 10000` | 서로 다른 컬럼의 OR은 인덱스 하나로 처리 불가 | 각각 인덱스를 타는 쿼리로 나눠 `UNION ALL` |
| `WHERE status != 'CANCELED'` | 부정 조건은 대부분의 행이 해당되어 범위가 넓음 | 필요한 값을 `IN`으로 나열 |

원칙은 **인덱스 컬럼은 가공하지 않고, 비교하는 값 쪽을 가공**하는 것이다.

형 변환은 방향이 중요하다. 문자열 컬럼을 숫자와 비교하면 컬럼이 변환되어 인덱스를 못 쓰지만, 숫자 컬럼을 문자열 상수와 비교하면(`user_id = '1'`) 상수 쪽이 변환되므로 인덱스를 쓸 수 있다.

### 인덱스가 있어도 쓰지 않는 경우

조건에 맞는 행이 테이블의 큰 비율을 차지하면, 인덱스를 거쳐 행마다 테이블에 접근(Random I/O)하는 것보다 Full Scan으로 순서대로 읽는 것이 빠르다. 옵티마이저는 이 비용을 비교해 Full Scan을 선택한다.

`status`처럼 값의 종류가 적은(카디널리티가 낮은) 컬럼의 단독 인덱스가 잘 쓰이지 않는 이유다. 이런 컬럼은 다른 컬럼과 Composite Index로 묶어서 쓴다. 다만 값이 한쪽으로 쏠려 있어서 `status = 'FAILED'`가 전체의 0.1%뿐이라면 그 조건에는 인덱스가 유효하다.

옵티마이저가 명백히 잘못된 인덱스를 고를 때는 `FORCE INDEX` 같은 힌트로 강제할 수 있지만, 데이터 분포가 바뀌면 오히려 독이 된다. 통계 갱신과 인덱스 재설계를 먼저 검토한다.

### 필요한 만큼만 읽기

- **필요한 컬럼만 조회한다.** `SELECT *`는 Covering Index를 막고, 네트워크 전송량과 애플리케이션 메모리 사용량을 늘린다. JPA에서는 조회 전용 화면에 DTO Projection을 쓴다.
- **필요한 행만 조회한다.** 전부 가져와 애플리케이션에서 필터링·집계하지 않고 WHERE, `LIMIT`, `GROUP BY`를 DB에서 처리한다.
- **존재 여부만 필요하면 `COUNT(*)` 대신 `EXISTS`를 쓴다.** `COUNT`는 조건에 맞는 행을 끝까지 세지만 `EXISTS`는 한 건을 찾으면 멈춘다.
- InnoDB는 테이블의 행 수를 따로 저장하지 않으므로 `COUNT(*)`는 인덱스를 실제로 스캔한다. 큰 테이블의 목록 화면에서 매번 전체 건수를 세는 쿼리가 본 쿼리보다 느린 경우가 많다.

### Pagination

```sql
-- OFFSET 방식: 앞의 100000건을 읽고 버림. 뒤 페이지일수록 느려짐
SELECT * FROM orders ORDER BY id DESC LIMIT 20 OFFSET 100000;

-- Cursor(Keyset) 방식: 마지막으로 본 id 이후부터 인덱스로 바로 탐색
SELECT * FROM orders WHERE id < 900000 ORDER BY id DESC LIMIT 20;
```

| 구분 | OFFSET | Cursor (Keyset) |
| --- | --- | --- |
| 속도 | 뒤 페이지일수록 느려짐 | 일정 |
| 특정 페이지로 이동 | 가능 | 불가능 (다음/이전만) |
| 조회 중 데이터 추가·삭제 | 행이 중복되거나 누락될 수 있음 | 영향 없음 |
| 적합한 화면 | 페이지 번호가 있는 관리자 화면 | 무한 스크롤, 피드 |

정렬 기준이 유일하지 않으면(예: `created_at`) 같은 값을 가진 행의 순서가 정해지지 않으므로, PK를 함께 사용해 순서를 고정한다.

```sql
-- 인덱스 (created_at, id) 사용
SELECT * FROM orders
WHERE created_at < :lastCreatedAt
   OR (created_at = :lastCreatedAt AND id < :lastId)
ORDER BY created_at DESC, id DESC
LIMIT 20;
```

페이지 번호가 꼭 필요하다면 앞에서 본 Deferred Join으로 OFFSET의 비용을 줄인다.

### N+1 문제

목록을 조회한 뒤 각 행의 연관 데이터를 한 건씩 다시 조회하면 쿼리가 1 + N번 실행된다. 쿼리 하나하나는 인덱스를 타서 빠르기 때문에 Slow Query Log에는 잡히지 않고, DB 왕복 횟수가 문제가 된다. JPA에서 지연 로딩된 연관 객체를 반복문에서 접근할 때 자주 발생한다.

```java
// 주문 N건 조회 후, 주문마다 회원 조회 쿼리가 1번씩 추가로 나감
List<Order> orders = orderRepository.findAll();
orders.forEach(order -> order.getMember().getName());

// Fetch Join으로 한 번에 조회
@Query("select o from Order o join fetch o.member")
List<Order> findAllWithMember();
```

| 해결 방법 | 내용 | 주의점 |
| --- | --- | --- |
| Fetch Join / `@EntityGraph` | JOIN으로 연관 데이터를 한 번에 조회 | 컬렉션(일대다)을 Fetch Join하면서 페이징하면 전체를 읽어 메모리에서 페이징함 |
| Batch Size | 연관 데이터를 `IN` 쿼리로 묶어 조회 (1 + 1번) | `IN` 목록이 너무 크지 않게 100~1000 정도로 설정 |
| DTO Projection | 필요한 컬럼만 JOIN으로 직접 조회 | 엔티티가 아니므로 변경 감지 대상이 아님 |

```yaml
spring:
  jpa:
    properties:
      hibernate:
        default_batch_fetch_size: 100
```

즉시 로딩(`EAGER`)으로 바꾸는 것은 해결책이 아니다. JPQL 조회에서는 여전히 N+1이 발생하고, 필요 없는 경우에도 항상 연관 데이터를 읽게 된다.

### 최적화 순서

1. Slow Query Log나 APM으로 **실제로 느린(또는 많이 호출되는) 쿼리**를 찾는다.
2. `EXPLAIN`으로 `type`, `key`, `key_len`, `rows`, `Extra`를 확인한다.
3. 쿼리를 고친다(조건 가공 제거, 필요한 컬럼만 조회, 페이징 방식 변경, N+1 제거).
4. 그래도 부족하면 인덱스를 추가·수정한다.
5. 다시 `EXPLAIN`과 실제 실행 시간으로 개선을 확인한다.

인덱스 추가를 뒤에 두는 이유는, 인덱스는 쓰기 비용을 영구히 늘리고 큰 테이블에서는 추가하는 작업 자체가 부담이기 때문이다.

## 4. Backend에서 확인할 점

- 인덱스는 테이블이 아니라 **쿼리**를 보고 설계한다. WHERE, ORDER BY, JOIN 조건에 쓰이는 컬럼과 순서를 먼저 확인한다.
- ORM이 만드는 SQL을 로그로 직접 확인한다. 코드에서는 한 줄이어도 쿼리는 여러 번 나갈 수 있다.
- 개발 DB는 데이터가 적어 실행 계획이 운영과 다르다. 운영과 비슷한 데이터 양·분포에서 `EXPLAIN`을 확인한다.
- `EXPLAIN`에서 `type`이 `ALL`/`index`인지, `key_len`이 기대한 컬럼 수만큼 나오는지, `Extra`에 `Using filesort`·`Using temporary`가 있는지 본다.
- 인덱스를 추가하면 쓰기 성능과 저장 공간 비용이 따른다. 쓰지 않는 인덱스와 중복 인덱스는 정리한다.
- 큰 테이블에 인덱스를 추가하는 작업은 시간이 오래 걸리고 부하를 주므로 트래픽이 적은 시간에 수행한다.
- 느린 쿼리는 Connection을 오래 점유해 Connection Pool 고갈로 이어진다(4주차). 쿼리 성능은 서버 전체 처리량의 문제다.
