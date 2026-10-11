# 쿼리 최적화 (Backend 심화)

## Composite / Covering Index

### 복합 인덱스와 컬럼 순서

[https://dev.mysql.com/doc/refman/8.0/en/multiple-column-indexes.html](https://dev.mysql.com/doc/refman/8.0/en/multiple-column-indexes.html)

- 컬럼 여러 개를 묶어서 하나의 인덱스로 만든 것. `(a, b, c)` 인덱스는 a로 정렬하고, a가 같으면 b로, b도 같으면 c로 정렬된 B+Tree임
  - 전화번호부가 성 → 이름 순으로 정렬된 것과 같음. 성을 모르면 이름만으로는 못 찾음
- 왼쪽 접두사 규칙 (Leftmost Prefix)
  - `(a, b, c)` 인덱스로 쓸 수 있는 조건 : `a` / `a, b` / `a, b, c`
  - 쓸 수 없는 조건 : `b` 단독 / `c` 단독 / `b, c`
  - `a, c` 는 a까지만 인덱스로 좁히고 c는 걸러낸 행에서 확인함
  - 그래서 `(a, b)` 인덱스가 있으면 `(a)` 인덱스는 따로 안 만들어도 됨. 반대로 `(b)` 가 필요하면 따로 만들어야 함
  - WHERE 절에 적는 순서는 상관없음. 옵티마이저가 인덱스 순서에 맞춰 재배열함
- 등치 조건은 앞에, 범위 조건은 뒤에
  - 범위 조건(`>`, `BETWEEN`, `LIKE 'x%'`, `IN` 은 경우에 따라)을 만난 컬럼 **뒤**의 컬럼은 인덱스 정렬을 못 씀
  - `WHERE a = 1 AND b > 10 AND c = 5` 에 `(a, b, c)` 인덱스 → a, b까지만 인덱스로 범위를 좁히고 c는 걸러낸 행을 하나씩 확인함 (filtered)
  - `(a, c, b)` 순서로 만들면 a, c 등치로 좁힌 뒤 b 범위까지 전부 인덱스로 처리됨
  - EXPLAIN의 `key_len` 으로 몇 번째 컬럼까지 썼는지 알 수 있음. INT 3개짜리 인덱스인데 key_len이 8이면 두 컬럼만 쓴 것
- 정렬도 인덱스 순서를 따라야 함
  - `WHERE a = 1 ORDER BY b` 에 `(a, b)` 인덱스 → a로 좁힌 범위가 이미 b 순으로 정렬되어 있으므로 정렬 작업이 필요 없음
  - `WHERE a > 1 ORDER BY b` → a가 범위라서 그 안의 b는 정렬 상태가 아님 → 별도 정렬(filesort) 발생
  - `WHERE a = 1 ORDER BY c` 에 `(a, b, c)` → b를 건너뛰어서 정렬에 못 씀. `WHERE a = 1 AND b = 2 ORDER BY c` 는 가능
- 컬럼 순서 정하는 법
  - "카디널리티 높은 걸 앞에"라는 말이 많은데, 그보다 먼저 볼 건 "어떤 쿼리가 들어오는가"임
  - 등치로 자주 쓰는 컬럼을 앞에, 범위/정렬 컬럼을 뒤에 두는 게 우선이고, 등치 컬럼끼리의 순서를 정할 때 카디널리티를 고려함
  - 인덱스는 쿼리 하나하나에 맞추는 게 아니라, 여러 쿼리가 왼쪽 접두사를 공유하도록 설계해야 인덱스 수를 줄일 수 있음. `(status, created_at)` 하나로 "status 조회"와 "status + 기간 조회"를 다 받는 식

### 커버링 인덱스

[https://jojoldu.tistory.com/476](https://jojoldu.tistory.com/476)

- 쿼리에 필요한 컬럼이 전부 인덱스 안에 있어서, 테이블(클러스터링 인덱스)을 읽지 않고 인덱스만으로 결과를 만드는 것
  - [Index / B+Tree](./index-btree.md)에서 봤듯이 세컨더리 인덱스 조회는 "인덱스 탐색 → PK로 테이블 재탐색" 두 단계인데, 두 번째 단계를 통째로 생략함
  - 랜덤 I/O가 사라지므로 조회 행 수가 많을수록 효과가 큼
  - EXPLAIN의 Extra에 `Using index` 가 나오면 커버링 인덱스로 처리된 것
- 조건
  - SELECT 절, WHERE 절, ORDER BY / GROUP BY 절에 나오는 컬럼이 모두 인덱스에 포함되어야 함
  - InnoDB 세컨더리 인덱스 리프에는 PK가 들어있으므로 PK 컬럼은 인덱스에 명시하지 않아도 커버됨
  - `SELECT *` 는 당연히 커버링이 안 됨. `COUNT(*)` 는 행 존재만 확인하면 되므로 조건 컬럼만 인덱스에 있으면 커버링 가능

```sql
-- (status, created_at) 인덱스가 있을 때
SELECT id, created_at FROM orders WHERE status = 'PAID';   -- Using index (id는 PK라 포함)
SELECT id, amount     FROM orders WHERE status = 'PAID';   -- amount 때문에 테이블 접근 발생
```

- 페이징에 응용

[https://jojoldu.tistory.com/529](https://jojoldu.tistory.com/529)

  - `SELECT * FROM posts WHERE category = ? ORDER BY id DESC LIMIT 20 OFFSET 100000` 같은 쿼리는 10만 행을 테이블에서 읽고 버림
  - 커버링 인덱스로 PK만 먼저 뽑으면 인덱스만 훑으므로 버리는 비용이 훨씬 작음

```sql
SELECT p.*
FROM posts p
JOIN (
    SELECT id
    FROM posts
    WHERE category = ?
    ORDER BY id DESC
    LIMIT 20 OFFSET 100000
) AS t ON p.id = t.id;
```

  - 서브쿼리는 `(category, id)` 인덱스만으로 처리되고, 테이블 접근은 최종 20건에 대해서만 일어남
  - 더 나아가면 OFFSET 자체를 없애는 No Offset(커서) 방식 : `WHERE category = ? AND id < {마지막으로 본 id} ORDER BY id DESC LIMIT 20`. 몇 페이지든 읽는 양이 20건으로 일정함
- 주의할 점
  - 커버링을 노리고 인덱스에 컬럼을 계속 추가하면 인덱스가 커져서 쓰기 비용과 메모리(버퍼 풀) 사용량이 늘어남. 자주 실행되고 느린 쿼리에만 선택적으로 적용하는 것
  - TEXT, BLOB 같은 큰 컬럼은 인덱스에 넣을 수 없거나 접두사만 들어가므로 커버링 대상이 아님



&nbsp;

## Query Plan / EXPLAIN

### 옵티마이저와 실행 계획

- SQL은 "무엇을" 가져올지만 적으므로, "어떻게" 가져올지는 옵티마이저가 정함. 그 결과물이 실행 계획 (Query Plan)
  - 어느 테이블부터 읽을지 (드라이빙 테이블)
  - 어떤 인덱스를 쓸지, 아니면 풀스캔할지
  - 조인은 어떤 알고리즘으로 할지
  - 정렬/그룹핑을 인덱스로 처리할지 따로 할지
- 비용 기반 옵티마이저 (Cost-Based Optimizer)
  - 가능한 계획들의 예상 비용(읽어야 할 페이지 수, 행 수, CPU 연산 등)을 계산해서 가장 싼 걸 고름
  - 비용 계산의 근거는 통계 정보. 테이블 행 수, 인덱스의 카디널리티, 값 분포(히스토그램, 8.0+)
  - 통계가 실제와 다르면 엉뚱한 계획이 나옴. 대량 INSERT/DELETE 후 갑자기 쿼리가 느려졌다면 `ANALYZE TABLE` 로 통계를 갱신해보는 게 첫 번째 시도
  - 옛날 방식인 규칙 기반 옵티마이저(RBO)는 "인덱스가 있으면 무조건 쓴다" 같은 고정 규칙이라 데이터 분포를 반영 못 함. 요즘 DBMS는 전부 CBO
- 같은 쿼리인데 파라미터 값에 따라 계획이 달라질 수 있음
  - `status = 'DELETED'` 가 전체의 1%면 인덱스를 타고, `status = 'ACTIVE'` 가 90%면 풀스캔을 고르는 식
  - 개발 DB에서 빠르던 쿼리가 운영에서 느린 이유 중 하나. 데이터 양과 분포가 다르면 계획도 다름. 성능 테스트는 운영과 비슷한 분포의 데이터로 해야 의미가 있음
- 옵티마이저 힌트
  - `USE INDEX`, `FORCE INDEX`, `STRAIGHT_JOIN`, 8.0의 `/*+ INDEX(t idx) */` 스타일 힌트로 계획을 강제할 수 있음
  - 데이터가 바뀌면 강제한 계획이 오히려 나빠질 수 있으므로 최후의 수단. 힌트가 필요하다는 건 보통 통계나 인덱스 설계에 문제가 있다는 신호

### EXPLAIN 읽는 법

[https://dev.mysql.com/doc/refman/8.0/en/explain-output.html](https://dev.mysql.com/doc/refman/8.0/en/explain-output.html)

- 쿼리 앞에 `EXPLAIN` 을 붙이면 실행하지 않고 실행 계획만 보여줌

```sql
EXPLAIN
SELECT o.id, o.created_at, m.name
FROM orders o
JOIN members m ON m.id = o.member_id
WHERE o.status = 'PAID'
ORDER BY o.created_at DESC
LIMIT 20;
```

```
+----+-------------+-------+--------+---------------------------+---------------------------+---------+-------------+--------+----------+-------------+
| id | select_type | table | type   | possible_keys             | key                       | key_len | ref         | rows   | filtered | Extra       |
+----+-------------+-------+--------+---------------------------+---------------------------+---------+-------------+--------+----------+-------------+
|  1 | SIMPLE      | o     | ref    | idx_orders_status_created | idx_orders_status_created | 82      | const       | 120000 |   100.00 | Using index |
|  1 | SIMPLE      | m     | eq_ref | PRIMARY                   | PRIMARY                   | 8       | o.member_id |      1 |   100.00 | NULL        |
+----+-------------+-------+--------+---------------------------+---------------------------+---------+-------------+--------+----------+-------------+
```

  - orders를 `(status, created_at)` 인덱스로 status 등치 조회(ref)하고, 그 인덱스가 created_at 순이라 정렬 없이 LIMIT 20까지 읽은 뒤, 각 행의 member_id로 members를 PK로 1건씩(eq_ref) 가져오는 계획
  - 첫 줄이 드라이빙 테이블. rows 120000은 status='PAID' 전체 추정치이고 LIMIT 때문에 실제로는 20건 읽고 멈춤
- 주요 컬럼
  - `id` : SELECT 단위 번호. 같으면 조인, 다르면 서브쿼리. 큰 번호가 먼저 실행됨
  - `select_type` : SIMPLE(단순), PRIMARY/SUBQUERY, DERIVED(FROM절 서브쿼리), DEPENDENT SUBQUERY(바깥 행마다 실행되는 서브쿼리라 주의) 등
  - `table` : 접근하는 테이블. 위에서 아래 순서가 실제 접근 순서 (첫 줄이 드라이빙 테이블)
  - `type` : 테이블 접근 방식. **제일 먼저 볼 것**
  - `possible_keys` / `key` : 후보 인덱스 / 실제 선택된 인덱스. key가 NULL이면 인덱스를 안 쓴 것
  - `key_len` : 사용된 인덱스의 바이트 길이. 복합 인덱스에서 몇 번째 컬럼까지 썼는지 여기서 알 수 있음
  - `ref` : 인덱스와 비교되는 값. const(상수), 다른 테이블의 컬럼
  - `rows` : 읽을 것으로 예상되는 행 수 (추정치)
  - `filtered` : 읽은 행 중 WHERE 조건을 통과할 것으로 예상되는 비율(%). rows × filtered / 100 이 다음 테이블로 넘어가는 행 수
  - `Extra` : 추가 정보. **두 번째로 볼 것**
- `type` 좋은 순서

| type | 의미 |
| --- | --- |
| const / system | PK나 UNIQUE로 1건 조회 |
| eq_ref | 조인에서 PK/UNIQUE로 1건씩 매칭 |
| ref | 일반 인덱스로 등치 조회 (여러 건 가능) |
| ref_or_null | ref + IS NULL 조건 |
| range | 인덱스 범위 스캔 (`>`, `BETWEEN`, `IN`, `LIKE 'x%'`) |
| index_merge | 인덱스 여러 개를 각각 쓰고 결과를 합침 (OR 조건) |
| index | 인덱스 풀스캔. 인덱스 전체를 처음부터 끝까지 읽음 |
| ALL | 테이블 풀스캔 |

  - 보통 range 이상이면 괜찮고, index나 ALL이 큰 테이블에서 나오면 손봐야 함
  - index는 이름 때문에 좋아 보이지만 풀스캔임. 커버링이라 테이블보다는 싸다는 것뿐
- `Extra` 자주 나오는 값
  - `Using index` : 커버링 인덱스. 좋음
  - `Using index condition` : 인덱스 컨디션 푸시다운(ICP). 인덱스 컬럼 조건을 스토리지 엔진 단에서 먼저 걸러서 테이블 접근을 줄임. 좋음
  - `Using where` : 읽어온 행을 WHERE로 다시 걸러냄. 흔하지만 rows 대비 filtered가 낮으면 인덱스가 조건을 충분히 못 걸러준다는 뜻
  - `Using filesort` : 인덱스로 정렬을 못 해서 별도 정렬 수행. 이름과 달리 메모리(sort_buffer_size)에서 먼저 시도하고 넘치면 디스크. ORDER BY 컬럼을 인덱스에 맞추면 사라짐
  - `Using temporary` : GROUP BY, DISTINCT, UNION 처리를 위해 임시 테이블 생성. filesort와 같이 나오면 대부분 느림
  - `Using join buffer (hash join)` : 드리븐 테이블에 쓸 인덱스가 없어 해시 조인으로 처리. 조인 컬럼 인덱스를 검토할 신호
  - `Using index for group-by` : Loose Index Scan. 좋음
- EXPLAIN ANALYZE (8.0.18+)
  - 실제로 쿼리를 실행하고 각 단계의 실제 소요 시간과 실제 행 수를 트리 형태로 보여줌

```
-> Limit: 20 row(s)  (cost=... rows=20) (actual time=0.05..0.21 rows=20 loops=1)
    -> Nested loop inner join  (actual time=0.05..0.20 rows=20 loops=1)
        -> Index lookup on o using idx_orders_status_created (status='PAID')  (actual time=0.03..0.08 rows=20 loops=1)
        -> Single-row index lookup on m using PRIMARY (id=o.member_id)  (actual time=0.005..0.005 rows=1 loops=20)
```

  - 추정(rows)과 실제(actual rows)가 크게 다르면 통계가 틀렸거나 옵티마이저가 잘못 판단한 것
  - `loops` 가 드리븐 테이블 접근 횟수. NLJ에서 바깥 행 수와 같음
  - 실제로 실행되므로 운영 DB에서 UPDATE/DELETE에 쓰면 안 됨. `EXPLAIN FORMAT=JSON` 은 실행 없이 비용 세부 내역을 보여줌
- 정리하면..

1. `type` 이 ALL / index 인 큰 테이블이 있는지 본다
2. `key` 가 NULL이거나 의도한 인덱스가 아니면 왜 못 탔는지 본다 (컬럼 가공, 형변환, 접두사 규칙)
3. `rows` 가 결과 건수에 비해 너무 크면 인덱스가 조건을 못 좁히는 것. `key_len` 으로 복합 인덱스를 끝까지 썼는지 확인
4. `Extra` 에 filesort / temporary 가 있으면 정렬·그룹핑을 인덱스로 대체할 수 있는지 본다
5. 고친 뒤 다시 EXPLAIN 해서 비교. 추정치만 믿지 말고 EXPLAIN ANALYZE나 실제 실행 시간으로 확인



&nbsp;

## Query 최적화

### 자주 보는 패턴

- 인덱스를 못 타게 만드는 조건 고치기
  - 컬럼 가공 → 상수 쪽을 가공 : `DATE(created_at) = '2026-10-11'` → `created_at >= '2026-10-11' AND created_at < '2026-10-12'`
  - 묵시적 형변환 → 타입 맞추기. 특히 ORM에서 파라미터 타입이 컬럼과 다른 경우
  - `OR` 로 서로 다른 컬럼을 묶으면 인덱스를 못 쓰는 경우가 많음 → `UNION ALL` 로 분리하거나 각각 인덱스가 있어야 index merge 가능
  - 앞 와일드카드 `LIKE '%키워드%'` → FULLTEXT 인덱스나 검색 엔진으로
- `SELECT *` 대신 필요한 컬럼만
  - 커버링 인덱스 가능성이 생기고, 네트워크 전송량과 ORM의 매핑 비용도 줄어듦
  - 나중에 컬럼이 추가돼도 쿼리 결과가 바뀌지 않아 안전하기도 함
- 페이징
  - 큰 OFFSET은 앞의 행을 다 읽고 버리는 구조 → 커버링 인덱스 서브쿼리 또는 No Offset
  - 무한 스크롤처럼 "다음 페이지"만 필요하면 No Offset이 맞고, "37페이지로 바로 이동"이 필요하면 OFFSET을 써야 함. 요구사항에 따라 결정
  - 커서 기준 컬럼은 유일해야 함. created_at만으로 커서를 잡으면 같은 시각 데이터가 빠지거나 중복되므로 `(created_at, id)` 처럼 PK를 붙임
- COUNT
  - `COUNT(*)` 는 InnoDB에서 행을 세어야 해서 큰 테이블에선 느림. 페이징 UI의 전체 건수 표시가 병목이 되는 경우가 많음
  - 정확하지 않아도 되면 통계 테이블이나 캐시로 대체, "다음 페이지 있음" 정도만 필요하면 `LIMIT n+1` 로 대체
  - `COUNT(컬럼)` 은 NULL을 제외하고 세므로 `COUNT(*)` 와 결과가 다를 수 있음. 성능 차이보다 의미 차이로 고르는 것
- 서브쿼리와 조인
  - WHERE 절의 `IN (서브쿼리)` 는 8.0 옵티마이저가 대부분 세미 조인으로 잘 바꿔주지만, 계획을 확인하는 습관이 필요함
  - `EXISTS` 는 매칭되는 행을 하나 찾으면 멈추므로 "존재 여부"만 필요할 때 적합
  - SELECT 절의 스칼라 서브쿼리는 행마다 실행됨. 결과 행이 많으면 조인으로 바꾸는 게 낫다
  - FROM 절 서브쿼리(DERIVED)는 임시 테이블을 만들 수 있고 거기엔 인덱스가 없음. 8.0은 가능하면 바깥 쿼리로 병합해주지만 GROUP BY, LIMIT이 있으면 병합이 안 됨
- 조인 순서와 조건
  - 드리븐 테이블의 조인 컬럼에 인덱스가 있는지 확인. 없으면 NLJ가 테이블 행 수의 곱만큼 읽음
  - 조인 전에 WHERE로 드라이빙 테이블 행 수를 최대한 줄이기
  - 조인 컬럼의 타입과 문자셋/콜레이션이 양쪽 테이블에서 같아야 함. 다르면 변환이 들어가서 인덱스를 못 씀
- 정렬과 그룹핑
  - `ORDER BY` 컬럼이 인덱스 순서와 같아야 filesort가 사라짐. `WHERE a = ? ORDER BY b` 면 `(a, b)`
  - 여러 컬럼 정렬에서 방향이 섞이면(`a ASC, b DESC`) 8.0부터 내림차순 인덱스 `(a ASC, b DESC)` 로 대응 가능
  - `GROUP BY` 도 인덱스 순서와 맞으면 임시 테이블 없이 처리됨
- 쓰기 쿼리
  - 대량 UPDATE/DELETE는 한 번에 하지 말고 PK 범위로 나눠서 (배치 사이즈 1,000~10,000) 처리. 한 트랜잭션이 너무 오래 락을 잡으면 다른 요청이 전부 대기함
  - INSERT는 여러 건을 한 문장으로 (`INSERT ... VALUES (...), (...), (...)`) 묶는 게 건별보다 훨씬 빠름. JPA라면 batch insert 설정

### 최적화 순서

- "느린 쿼리를 찾고 → EXPLAIN으로 원인을 보고 → 인덱스나 쿼리를 고치고 → 다시 EXPLAIN과 실행 시간으로 확인" 임. 감으로 인덱스부터 추가하는 건 쓰기 비용만 늘릴 수 있음
  - 느린 쿼리는 slow query log(`long_query_time`), APM, `performance_schema` 의 statement 통계로 찾음
  - 한 번에 1초 걸리는 쿼리보다 10ms인데 초당 1,000번 실행되는 쿼리가 전체 부하에는 더 클 수 있음. 실행 횟수 × 시간으로 봐야 함
- 인덱스는 추가하는 것만큼 지우는 것도 중요함
  - 안 쓰는 인덱스는 쓰기마다 비용만 발생시킴. `sys.schema_unused_indexes` 로 확인 가능
  - 중복 인덱스도 흔함. `(a)` 와 `(a, b)` 가 같이 있으면 `(a)` 는 불필요. `sys.schema_redundant_indexes`
- 쿼리 튜닝으로 안 되는 규모면 다음 단계는 캐시(12주차), 읽기 복제본 분리, 샤딩(13주차)으로 넘어감. 그전에 인덱스와 쿼리로 해결되는 경우가 생각보다 많음
- ORM(JPA)을 쓸 때도 결국 나가는 SQL을 보고 EXPLAIN 해야 함. 로그에 SQL을 찍어두고 N+1, 불필요한 조인, 전체 컬럼 조회가 있는지 보는 게 시작 (10주차에서 자세히)

### 면접 질문으로 정리하면..

- Q. 복합 인덱스 컬럼 순서는 어떻게 정하나요? → 왼쪽 접두사 규칙, 등치 앞 범위 뒤, 쿼리들이 접두사를 공유하도록
- Q. 커버링 인덱스가 뭔가요? → 인덱스만으로 쿼리를 처리해 테이블 접근을 생략. InnoDB 2단계 탐색 구조와 연결해서 설명
- Q. 쿼리가 느릴 때 어떻게 접근하나요? → slow log → EXPLAIN (type, key, rows, Extra) → 인덱스/쿼리 수정 → 재측정. 실제 경험이 있으면 그 사례로
- Q. 인덱스가 있는데 EXPLAIN에서 ALL이 나와요 → 조건에서 컬럼 가공/형변환, 선택도가 낮아 풀스캔이 더 싸다고 판단, 통계 불일치. ANALYZE TABLE과 힌트는 그 다음
