# 6주차 - Database

## 공통 키워드

### RDB / NoSQL

| 구분 | RDB | NoSQL |
|---|---|---|
| 데이터 모델 | 테이블(행/열), 고정 스키마 | Key-Value, Document, Column-Family, Graph 등 유연한 스키마 |
| 관계 | FK + JOIN | 보통 비정규화/임베딩으로 처리 |
| 트랜잭션 | ACID 지원 | 제품별 상이 (최종적 일관성, BASE 지향이 많음) |
| 대표 | MySQL, PostgreSQL, Oracle | Redis, MongoDB, Cassandra, Neo4j |
| 적합한 경우 | 정합성이 중요, 관계가 복잡 (결제, 주문) | 대용량/고속 쓰기, 스키마 변동 잦음 (로그, 캐시, 피드) |



### PK / FK

- **PK (Primary Key)**: 행을 유일하게 식별. `UNIQUE` + `NOT NULL`, 테이블당 1개. InnoDB는 PK 기준으로 데이터가 정렬 저장(클러스터드 인덱스).
- **FK (Foreign Key)**: 다른 테이블의 PK(또는 UNIQUE 키)를 참조. 참조 무결성 보장.
- 대리키(Surrogate, AUTO_INCREMENT/UUID) vs 자연키(Natural, 이메일·주민번호 등)
  - 실무는 대리키 선호 (변경 가능성, 크기, 인덱스 효율)
  - UUID는 랜덤이라 삽입 시 페이지 분할 증가 → UUIDv7/ULID 같은 시간순 정렬 가능한 값 고려
- FK 옵션: `ON DELETE CASCADE / SET NULL / RESTRICT`
- 실무 논쟁: FK 제약을 DB에 걸 것인가? (무결성 vs 대량 쓰기·샤딩·마이그레이션 유연성)

```sql
CREATE TABLE member (
  id BIGINT AUTO_INCREMENT PRIMARY KEY,
  email VARCHAR(100) NOT NULL UNIQUE
);

CREATE TABLE orders (
  id BIGINT AUTO_INCREMENT PRIMARY KEY,
  member_id BIGINT NOT NULL,
  ordered_at DATETIME NOT NULL,
  FOREIGN KEY (member_id) REFERENCES member(id) ON DELETE RESTRICT
);
```

### 정규화

중복을 제거해 **이상(Anomaly)** 을 방지하는 설계 과정.

- **이상 현상**: 삽입 이상 / 갱신 이상 / 삭제 이상
- **1NF**: 모든 컬럼이 원자값 (배열·쉼표 구분 값 금지)
- **2NF**: 1NF + 부분 함수 종속 제거 (복합키의 일부에만 종속된 컬럼 분리)
- **3NF**: 2NF + 이행 함수 종속 제거 (A→B→C 형태 분리)
- **BCNF**: 모든 결정자가 후보키
- **반정규화(Denormalization)**: 조회 성능을 위해 의도적으로 중복 허용 (집계 컬럼, 중복 컬럼 등). 정합성 관리 비용이 생기므로 근거 있을 때만.

```
[비정규] 주문(주문ID, 고객명, 고객주소, 상품명1, 상품명2 ...)
   ↓ 정규화
고객(고객ID, 이름, 주소) / 주문(주문ID, 고객ID) / 주문상품(주문ID, 상품ID, 수량) / 상품(상품ID, 이름)
```

### JOIN

두 테이블 이상을 연결해 조회.

| 종류 | 설명 |
|---|---|
| INNER JOIN | 양쪽에 매칭되는 행만 |
| LEFT (OUTER) JOIN | 왼쪽 전부 + 매칭되는 오른쪽 (없으면 NULL) |
| RIGHT (OUTER) JOIN | LEFT의 반대 |
| FULL OUTER JOIN | 양쪽 전부 (MySQL 미지원, UNION으로 대체) |
| CROSS JOIN | 카티전 곱 |
| SELF JOIN | 자기 자신과 조인 (계층 구조 등) |

```sql
SELECT m.email, o.id, o.ordered_at
FROM member m
INNER JOIN orders o ON o.member_id = m.id
WHERE o.ordered_at >= '2026-01-01';
```

- **조인 알고리즘**: Nested Loop Join (드라이빙 테이블 행마다 드리븐 테이블 탐색), Hash Join (MySQL 8.0.18+), Sort-Merge Join
- 조인 컬럼에는 **인덱스 필수** (특히 드리븐 테이블)
- 작은 결과 집합을 드라이빙 테이블로 두는 것이 유리
- **N+1 문제**: ORM에서 연관 엔티티를 건건이 조회 → fetch join / `@EntityGraph` / batch size로 해결

### Index

테이블 전체를 읽지 않고(Full Scan) 원하는 행을 빠르게 찾기 위한 자료구조.

- 장점: 조회(WHERE, JOIN, ORDER BY, GROUP BY) 속도 향상
- 단점: 추가 저장 공간, INSERT/UPDATE/DELETE 시 인덱스 갱신 비용 → **읽기 많은 컬럼에만**
- **클러스터드 인덱스**: 리프 노드 = 실제 데이터 (InnoDB의 PK). 테이블당 1개.
- 인덱스 설계 기준: **카디널리티(고유값 다양성)가 높은** 컬럼, 자주 조회되는 컬럼
- 인덱스를 못 타는 경우
  - 컬럼에 함수/연산 적용: `WHERE YEAR(created_at) = 2026`
  - 암묵적 형변환: 문자열 컬럼에 숫자 비교
  - 앞부분 와일드카드: `LIKE '%kim'`
  - 부정 조건(`!=`, `NOT IN`)이나 옵티마이저가 Full Scan이 더 싸다고 판단할 때

```sql
CREATE INDEX idx_orders_member ON orders(member_id);
```

### B+Tree

대부분의 RDB 인덱스가 사용하는 균형 트리.

- **균형 트리**: 모든 리프까지의 깊이가 동일 → 탐색 `O(log N)` 보장
- **B-Tree와 차이**
  - B-Tree: 모든 노드(내부 포함)가 데이터를 가짐
  - B+Tree: **데이터는 리프에만**, 내부 노드는 키(라우팅 용도)만 → 한 노드에 더 많은 키 → 트리 높이 낮아짐 (디스크 I/O 감소)
  - **리프 노드끼리 Linked List로 연결** → 범위 검색(`BETWEEN`, `>`), 정렬 순회에 유리
- 디스크 페이지(InnoDB 기본 16KB) 단위로 노드 구성, 삽입/삭제 시 노드 분할·병합으로 균형 유지
- Hash Index와 비교: 동등 비교(`=`)는 빠르지만 범위·정렬 불가

```
          [ 30 | 60 ]              ← 내부 노드 (키만)
         /     |     \
  [10|20] → [30|40|50] → [60|70|80]   ← 리프 노드 (데이터/포인터, 서로 연결)
```

---

## Backend 심화 (선택)

### Composite Index (복합 인덱스)

여러 컬럼을 묶어 하나의 인덱스로 구성.

- **컬럼 순서가 핵심**: 왼쪽 컬럼부터 차례로 정렬됨 (**Leftmost Prefix Rule**)
  - `INDEX(a, b, c)` → `a`, `a+b`, `a+b+c` 조건은 활용 가능 / `b`만, `c`만, `b+c`는 활용 불가
- 순서 결정 기준
  1. 동등(`=`) 조건 컬럼을 앞에
  2. 범위(`>`, `BETWEEN`) 조건 컬럼은 뒤에 (범위 이후 컬럼은 인덱스 필터링에 사용 불가)
  3. 카디널리티가 높은 컬럼을 앞에 (단, 쿼리 패턴이 우선)
- `ORDER BY`도 인덱스 순서와 맞으면 별도 정렬(filesort) 생략 가능

```sql
CREATE INDEX idx_orders_member_date ON orders(member_id, ordered_at);

-- 활용 O
SELECT * FROM orders WHERE member_id = 1 AND ordered_at >= '2026-01-01';
SELECT * FROM orders WHERE member_id = 1 ORDER BY ordered_at;
-- 활용 X (선두 컬럼 없음)
SELECT * FROM orders WHERE ordered_at >= '2026-01-01';
```

### Covering Index (커버링 인덱스)

쿼리에 필요한 **모든 컬럼이 인덱스에 포함**되어, 테이블(데이터 페이지) 접근 없이 인덱스만으로 결과를 반환.

- 세컨더리 인덱스의 PK Lookup(랜덤 I/O) 제거 → 큰 성능 이점
- `EXPLAIN`의 Extra에 **`Using index`** 표시
- SELECT 절 컬럼까지 고려해 설계, 단 인덱스가 커지면 쓰기 비용·공간 증가 (트레이드오프)
- InnoDB 세컨더리 인덱스는 PK를 자동 포함 → PK 컬럼 조회는 커버링 가능
- 페이징 최적화에 활용: 커버링 인덱스로 PK만 먼저 뽑고 → 그 PK로 본 테이블 조인

```sql
-- INDEX(member_id, ordered_at) 가 있을 때
SELECT member_id, ordered_at FROM orders WHERE member_id = 1;  -- Using index

-- 페이징: 커버링 인덱스로 id만 먼저 조회 후 조인
SELECT o.*
FROM orders o
JOIN (
  SELECT id FROM orders
  WHERE member_id = 1
  ORDER BY ordered_at DESC
  LIMIT 100000, 20
) t ON o.id = t.id;
```

### Query Plan (실행 계획)

옵티마이저가 쿼리를 **어떤 방식으로 실행할지 결정한 계획**.

- **옵티마이저**: 비용 기반(CBO). 통계 정보(카디널리티, 데이터 분포)로 접근 경로·조인 순서·조인 방식을 선택
- 통계가 오래되면 계획이 잘못될 수 있음 → `ANALYZE TABLE`
- 결정 항목: 인덱스 사용 여부, 테이블 접근 순서(조인 순서), 조인 알고리즘, 정렬/그룹 방식
- 인덱스 힌트(`USE INDEX`, `FORCE INDEX`)로 개입 가능하나 최후의 수단

### EXPLAIN

실행 계획을 확인하는 명령어. (`EXPLAIN ANALYZE`는 MySQL 8.0.18+, 실제 실행 후 실측 시간까지 출력)

```sql
EXPLAIN SELECT * FROM orders WHERE member_id = 1;
```

| 컬럼 | 의미 / 확인 포인트 |
|---|---|
| `type` | 접근 방식. 좋음 → 나쁨: `system` > `const` > `eq_ref` > `ref` > `range` > `index` > `ALL`(Full Scan) |
| `possible_keys` / `key` | 후보 인덱스 / 실제 사용된 인덱스 |
| `key_len` | 사용된 인덱스 길이 (복합 인덱스에서 몇 컬럼까지 썼는지 추정) |
| `rows` | 예상 탐색 행 수 (적을수록 좋음) |
| `filtered` | 조건으로 걸러지고 남는 비율(%) |
| `Extra` | `Using index`(커버링), `Using where`, `Using filesort`(별도 정렬, 개선 대상), `Using temporary`(임시 테이블, 개선 대상) |

- 체크 순서: `type`이 `ALL`인가? → `key`가 의도한 인덱스인가? → `rows`가 과도한가? → `Extra`에 filesort/temporary 있나?

### Query 최적화

**진단 → 개선 → 재측정** 순서로 접근. (슬로우 쿼리 로그 → EXPLAIN → 수정 → 재확인)

1. **인덱스 설계**
   - WHERE / JOIN / ORDER BY 컬럼 기준으로 복합·커버링 인덱스 적용
   - 사용하지 않는 중복 인덱스 제거
2. **SELECT 정리**
   - `SELECT *` 지양, 필요한 컬럼만 (커버링 가능성 증가, 네트워크 절감)
3. **조건절 작성**
   - 컬럼 가공 금지 (`WHERE created_at >= '2026-01-01' AND created_at < '2027-01-01'`)
   - 형변환 일치, 선행 와일드카드 지양
4. **JOIN 최적화**
   - 조인 컬럼 인덱스, 불필요한 조인 제거, 작은 집합 먼저 필터링
   - 서브쿼리 → JOIN 또는 `EXISTS`로 변경 검토
5. **페이징**
   - 큰 `OFFSET`은 앞의 행을 전부 읽고 버림 → 커서(No-offset) 방식: `WHERE id < :lastId ORDER BY id DESC LIMIT 20`
6. **N+1 / 불필요한 쿼리 제거** (fetch join, batch size, 캐싱)
7. **구조 차원**: 반정규화, 캐시(Redis), 읽기 분산(Read Replica), 파티셔닝
8. **측정 기반**: 추측 말고 `EXPLAIN ANALYZE`와 실측으로 검증. 데이터 양이 달라지면 계획도 달라짐 (운영과 유사한 데이터로 테스트)



