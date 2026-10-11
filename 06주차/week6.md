# Database: 데이터를 나누어 저장하고 필요한 만큼 읽는 방법

## 1. RDB / NoSQL: 데이터의 관계를 어디에서 관리할까?

RDB(Relational Database)는 데이터를 테이블로 표현하고 키로 관계를 연결한다. 주문 서비스라면 고객, 주문, 상품, 주문 항목을 각각 저장하고 필요한 시점에 함께 조회할 수 있다.

```text
customers ──< orders ──< order_items >── products
  고객         주문        주문 항목        상품
```

NoSQL은 하나의 데이터 모델이 아니라 문서, 키-값, 와이드 컬럼, 그래프 등 여러 모델을 묶어 부르는 이름이다. 예를 들어 문서형 DB에서는 함께 읽는 주문 항목을 주문 문서 안에 포함하는 설계를 할 수 있다.

```json
{
  "orderId": 101,
  "customerId": 42,
  "items": [
    { "productId": 7, "quantity": 2, "unitPrice": 12000 }
  ]
}
```

| 판단 기준 | 관계형 모델 | 문서형 모델의 예 |
| --- | --- | --- |
| 관계 표현 | 별도 테이블과 키로 연결 | 문서 내부 포함 또는 다른 문서 참조 |
| 함께 읽는 데이터 | JOIN 등으로 결합 | 한 문서에 포함해 함께 조회 가능 |
| 중복 데이터 변경 | 분리된 원본을 갱신 | 복제된 값의 갱신 정책이 필요할 수 있음 |
| 설계에서 살필 점 | 관계, 제약 조건, 조회 패턴 | 문서 경계, 크기, 조회·갱신 패턴 |

**RDB는 항상 느리고 NoSQL은 항상 빠르다**고 비교할 수는 없다. 같은 데이터라도 관계를 자주 조합하는지, 한 단위로 읽고 쓰는지에 따라 비용이 달라진다. MongoDB도 스키마 검증과 다중 문서 트랜잭션을 지원하므로, NoSQL을 스키마나 트랜잭션이 없는 DB로 이해하면 안 된다. [문서 포함 모델](https://www.mongodb.com/docs/manual/data-modeling/embedding/)과 [MongoDB 트랜잭션](https://www.mongodb.com/docs/manual/core/transactions/)을 참고했다.

---

## 2. PK / FK: 행을 식별하는 것과 관계를 지키는 것

PK(Primary Key)는 행을 유일하게 식별하는 키다. 중복과 NULL을 허용하지 않으며 여러 컬럼의 조합으로 만들 수도 있다. FK(Foreign Key)는 참조하는 값이 유효한 대상과 연결되도록 하는 제약이다.

```sql
CREATE TABLE customers (
    id BIGINT PRIMARY KEY,
    name TEXT NOT NULL
);

CREATE TABLE orders (
    id BIGINT PRIMARY KEY,
    customer_id BIGINT NOT NULL REFERENCES customers(id),
    status TEXT NOT NULL,
    created_at TIMESTAMP NOT NULL,
    total_amount NUMERIC(12, 2) NOT NULL CHECK (total_amount >= 0)
);
```

`orders.id`는 주문의 식별자이고, `orders.customer_id`는 고객과의 관계다. 없는 고객의 ID로 주문을 만들면 FK 제약에 어긋난다. 관계가 필수라면 위 예제처럼 `NOT NULL`도 필요하다. FK만 선언하면 NULL을 허용할 수 있다.

FK는 PK뿐 아니라 적절한 유일 키를 참조할 수도 있다. 참조 대상 삭제 시 연결된 행도 삭제할지, 삭제를 거부할지 등은 `ON DELETE` 정책과 업무 요구에 따라 결정한다.

키와 인덱스는 목적이 다르다. PK/FK는 데이터의 규칙이고, 인덱스는 접근을 돕는 구조다. PostgreSQL은 PK에 유일 B-tree 인덱스를 자동 생성하지만 **FK를 선언해도 참조하는 쪽 컬럼의 인덱스는 자동 생성하지 않는다.** 고객별 주문 조회나 참조 대상 삭제 검사 비용을 보고 별도 인덱스를 판단해야 한다. [PostgreSQL의 제약 조건](https://www.postgresql.org/docs/17/ddl-constraints.html)을 기준으로 정리했다.

---

## 3. 정규화: 같은 사실을 여러 행에 적으면 무엇이 어려울까?

다음은 한 주문에서 같은 상품을 한 번만 담는다고 가정한 테이블이다. 후보 키는 `(order_id, product_id)`이고, 고객 이름과 상품의 현재 이름을 함께 저장한다.

| order_id | product_id | customer_id | customer_name | product_name | quantity |
| --- | --- | --- | --- | --- | --- |
| 101 | 7 | 42 | Kim | 키보드 | 2 |
| 101 | 8 | 42 | Kim | 마우스 | 1 |
| 102 | 7 | 43 | Lee | 키보드 | 1 |

고객 이름을 바꾸려면 여러 주문 항목을 수정해야 한다. 일부만 바꾸면 같은 고객의 이름이 달라진다. 주문이 없어도 상품 정보를 등록하고 싶은데 주문 컬럼이 발목을 잡을 수 있고, 마지막 주문 항목을 삭제하면서 상품 정보까지 사라질 수도 있다. 각각 갱신·삽입·삭제 이상이다.

정규화는 **함수 종속성을 기준으로 중복과 변경 이상을 줄이는 과정**이다. `X → Y`는 X가 같으면 Y도 하나로 결정된다는 뜻이다. 표에 우연히 같은 값이 보이는지가 아니라, 업무에서 성립하는 규칙으로 판단한다.

```text
order_id → customer_id
customer_id → customer_name
product_id → product_name
(order_id, product_id) → quantity
```

| 단계 | 핵심 조건 | 예제에서의 적용 |
| --- | --- | --- |
| 1NF | 각 속성에 도메인 기준의 단일 값 저장 | 상품 목록을 한 셀에 반복 나열하지 않고 항목별 행으로 표현 |
| 2NF | 1NF + 비주요 속성이 후보 키 일부에만 종속되지 않음 | 주문 정보와 상품 정보를 주문 항목에서 분리 |
| 3NF | 2NF + 비주요 속성의 이행 종속 제거 | 주문에서 고객 이름을 분리하고 고객 테이블에서 관리 |

여기서 비주요 속성은 어느 후보 키에도 포함되지 않는 속성이다. 3NF의 엄밀한 조건은 비자명한 함수 종속 `X → A`마다 X가 슈퍼키이거나 A가 주요 속성이어야 한다는 것이다. 위 표의 이행 종속 설명은 예제처럼 키 구조가 단순한 경우를 이해하기 위한 설명이다.

```text
customers(id, name)
orders(id, customer_id, status, created_at, total_amount)
products(id, name)
order_items(order_id, product_id, quantity, unit_price)
```

이 구조에서는 고객 이름을 한 곳에서 관리하고, 조회할 때 관계를 연결한다. [정규화의 단계와 변경 이상](https://learn.microsoft.com/en-us/previous-versions/troubleshoot/microsoft-365/microsoft-365-apps/access/database-normalization-description)을 참고했다.

다만 모든 반복 값이 제거 대상은 아니다. `order_items.unit_price`를 **주문 당시 판매 단가**로 정의하면 현재 상품 가격과는 다른 사실이다. 상품 가격이 바뀌어도 과거 주문 금액은 유지해야 한다. 주문 당시 배송지나 상품명도 같은 이유로 스냅샷을 보관할 수 있다. 반정규화는 이런 의미를 구분한 뒤, 조회 비용과 변경 정합성 관리 비용을 측정해서 선택해야 한다.

---

## 4. JOIN: 나눈 데이터를 어떤 기준으로 다시 연결할까?

JOIN은 여러 테이블의 행을 조건에 따라 결합한다.

| 종류 | 결과 |
| --- | --- |
| INNER JOIN | 양쪽에서 조건이 맞는 조합 |
| LEFT JOIN | 왼쪽 행을 유지하고, 대응하는 오른쪽 행이 없으면 NULL로 채움 |
| RIGHT JOIN | 오른쪽 행을 유지하는 외부 조인 |
| FULL OUTER JOIN | 양쪽의 대응하지 않는 행까지 유지 |
| CROSS JOIN | 양쪽 행의 모든 조합 |

고객별 주문을 조회하려면 다음처럼 연결한다.

```sql
SELECT c.id AS customer_id, c.name, o.id AS order_id, o.status
FROM customers AS c
INNER JOIN orders AS o ON o.customer_id = c.id;
```

주문이 없는 고객까지 포함하되 완료 주문만 연결하려면 `LEFT JOIN`의 조건을 조정한다.

```sql
SELECT c.id AS customer_id, c.name, o.id AS order_id
FROM customers AS c
LEFT JOIN orders AS o
    ON o.customer_id = c.id AND o.status = 'COMPLETED';
```

여기서 `o.status = 'COMPLETED'`를 `WHERE`로 옮기면 대응 주문이 없어 NULL로 채워진 행은 제거된다. **ON은 어떤 행을 연결할지 정하고, WHERE는 결합된 결과를 걸러낸다.** 외부 조인에서 두 위치는 결과가 달라질 수 있다. [JOIN과 테이블 표현식](https://www.postgresql.org/docs/17/queries-table-expressions.html)을 참고했다.

또한 고객 한 명의 주문이 세 개면 고객은 세 행에 나타난다. JOIN이 고객을 중복 저장한 것이 아니라 관계의 개수만큼 결과가 생긴 것이다. 주문에 주문 항목까지 붙이면 주문 수도 행 수와 달라질 수 있으므로 집계와 페이지네이션의 기준을 먼저 정해야 한다.

---

## 5. Index: 테이블을 전부 읽지 않으려면?

인덱스는 조건에 맞는 데이터에 접근하도록 돕는 별도 구조다. 고객 42의 주문을 찾을 때 인덱스가 없다면 많은 행을 확인해야 할 수 있다.

```sql
CREATE INDEX idx_orders_customer ON orders(customer_id);

SELECT id, status
FROM orders
WHERE customer_id = 42;
```

인덱스에 고객별로 정리된 키가 있으면 필요한 위치를 찾고 대응하는 행에 접근할 수 있다. B-tree 계열은 정렬된 키를 이용해 동등 비교뿐 아니라 범위 조회와 일부 정렬 처리도 돕는다. 다만 모든 인덱스가 같은 구조는 아니다. PostgreSQL에는 Hash, GIN, GiST, BRIN 등도 있으며 조건과 데이터에 따라 선택이 달라진다. [인덱스 유형](https://www.postgresql.org/docs/17/indexes-types.html)을 참고했다.

인덱스에는 저장 공간과 쓰기 비용이 든다. 행을 삽입·삭제하거나 인덱스 대상 값을 수정하면 인덱스도 관리해야 한다. 많은 행을 반환하는 조회에서는 인덱스를 거쳐 여기저기 접근하는 것보다 테이블을 순차적으로 읽는 편이 유리할 수 있다.

따라서 **인덱스가 있으면 항상 사용하고, 사용하면 항상 빠르다**는 보장은 없다. 자주 실행하는 조건, 반환 비율, 정렬, 테이블 크기와 쓰기 빈도를 함께 봐야 한다.

---

## 6. B+Tree: 왜 DB 인덱스에는 가지가 많은 트리를 쓸까?

메모리에서 비교 횟수만 줄이는 것과 저장 장치에서 페이지를 적게 읽는 것은 다른 문제다. B+Tree는 한 노드에 여러 키와 자식 참조를 담아 높이를 낮춘 균형 트리다. 내부 노드는 탐색 경로를 안내하고, 리프에는 키와 데이터 또는 데이터 위치가 있다. 리프끼리 연결되어 범위 조회를 이어갈 수 있다.

```text
                  [20 | 40]
                 /    |    \
                /     |     \
      [5, 10, 15] ↔ [20, 25, 30] ↔ [40, 45, 50]
                     리프 계층
```

위 그림은 구분 키와 리프 연결을 나타낸 개념도다. `20 이상 45 이하`를 찾는다면 시작 리프를 탐색하고 연결된 리프를 따라 필요한 범위를 읽는다. 분기 수가 크면 트리 높이가 낮아져 탐색 중 거치는 페이지 수를 줄일 수 있다.

균형 트리의 시작점 탐색은 보통 `O(log N)`으로 설명하지만, 전체 범위 조회에는 반환할 항목을 읽는 비용도 더해진다. 인덱스로 찾은 뒤 실제 행을 읽는 비용과 캐시 상태도 있어, 시간복잡도만으로 쿼리 실행 시간을 예측할 수는 없다.

일반적인 B-tree 설명에서는 내부 노드에도 데이터 항목을 둘 수 있지만, B+Tree는 리프에 데이터 항목을 모은다. DB 문서는 리프 중심의 구현도 B-tree라고 부를 수 있어 이름만으로 저장 구조를 단정하지 않는다. [PostgreSQL B-tree 구현](https://www.postgresql.org/docs/17/btree.html#BTREE-STRUCTURE)은 내부 페이지의 하위 페이지 참조와 리프의 테이블 행 참조를 설명한다.

리프에 담기는 내용도 엔진마다 다르다. **MySQL InnoDB의 클러스터 인덱스 리프에는 행 데이터가 있고, 보조 인덱스에는 기본 키 값이 있다.** 반면 PostgreSQL의 인덱스는 테이블의 heap과 별도로 저장된다. [InnoDB의 클러스터·보조 인덱스](https://dev.mysql.com/doc/refman/8.4/en/innodb-index-types.html)를 PostgreSQL의 구조와 구분해야 한다.

---

## Backend 심화

## 7. Composite Index: 컬럼 순서가 왜 중요할까?

고객별 최신 주문 20개를 조회한다고 하자.

```sql
SELECT id, created_at, total_amount
FROM orders
WHERE customer_id = 42
ORDER BY created_at DESC, id DESC
LIMIT 20;

CREATE INDEX idx_orders_customer_recent
    ON orders(customer_id, created_at DESC, id DESC);
```

복합 인덱스는 여러 키를 순서대로 정렬한다. 이 예제는 고객으로 범위를 좁힌 뒤 그 안에서 생성 시각과 ID 순으로 읽는 조회와 맞는다. 생성 시각이 같아도 ID로 순서를 정하므로 결과 순서가 명확하다.

```text
customer_id → 같은 고객 안의 created_at DESC → 같은 시각의 id DESC
```

`customer_id` 조건 없이 날짜만 조회하면 같은 인덱스가 똑같이 효율적이지 않을 수 있다. 그렇다고 선두 컬럼이 없으면 절대 사용할 수 없다고 외우는 것도 부정확하다. 사용할 수 있는 경로와 효율은 DBMS, 버전, 분포에 따라 달라진다.

PostgreSQL 17의 다중 컬럼 B-tree에서는 선두 컬럼의 동등 조건과 그 뒤 첫 범위 조건이 스캔 구간을 제한한다. 더 뒤의 조건은 인덱스에서 걸러낼 수 있어도 같은 방식으로 구간을 줄이지는 못한다. [복합 인덱스의 규칙](https://www.postgresql.org/docs/17/indexes-multicolumn.html)을 참고했다.

인덱스 순서는 단순히 값의 종류가 많은 컬럼부터 정하는 문제가 아니다. 실제 `WHERE`, `ORDER BY`, 범위 조건과 다른 조회에서의 재사용을 함께 봐야 한다. 위 고객 단일 인덱스와 복합 인덱스도 모두 유지할 필요가 있는지 비교 대상이다.

---

## 8. Covering Index: 인덱스에서 조회를 끝낼 수 있을까?

커버링 인덱스는 **특정 쿼리에 필요한 컬럼을 모두 포함하는 인덱스**다. 독립적인 트리 종류가 아니라 쿼리와 인덱스의 관계다. 앞선 최신 주문 조회를 위해 PostgreSQL에서는 다음 후보를 만들 수 있다.

```sql
CREATE INDEX idx_orders_customer_recent_cover
    ON orders(customer_id, created_at DESC, id DESC)
    INCLUDE (total_amount);
```

`total_amount`는 반환을 위한 값이며 `INCLUDE`에 넣었다고 탐색·정렬 키가 되는 것은 아니다. 이 인덱스는 앞의 복합 인덱스와 비교할 대안이다. 비슷한 인덱스를 모두 추가하면 저장 공간과 쓰기 비용도 늘어난다.

PostgreSQL에서는 컬럼이 모두 있어도 MVCC의 행 가시성을 확인해야 한다. Visibility Map으로 확인할 수 없는 페이지는 heap을 방문하므로 **Index Only Scan이 표시되어도 Heap Fetches가 0이라는 보장은 없다.** [Index-Only Scan과 커버링 인덱스](https://www.postgresql.org/docs/17/indexes-index-only-scans.html)를 참고했다.

많은 컬럼, 특히 큰 값을 무작정 포함하면 인덱스가 커진다. 테이블 방문을 줄이는 효과와 인덱스 유지 비용을 비교해야 한다.

---

## 9. Query Plan / EXPLAIN: DB는 SQL을 어떤 순서로 처리할까?

SQL은 원하는 결과를 표현하고, 옵티마이저는 통계와 비용 추정을 바탕으로 실행 방법을 고른다. 같은 SQL도 데이터와 통계가 달라지면 계획이 바뀔 수 있다.

```sql
EXPLAIN
SELECT id, created_at, total_amount
FROM orders
WHERE customer_id = 42
ORDER BY created_at DESC, id DESC
LIMIT 20;

EXPLAIN (ANALYZE, BUFFERS)
SELECT id, created_at, total_amount
FROM orders
WHERE customer_id = 42
ORDER BY created_at DESC, id DESC
LIMIT 20;
```

일반 `EXPLAIN`은 추정 계획을 보여주고, `EXPLAIN ANALYZE`는 실제 쿼리를 실행해 측정값도 보여준다. UPDATE나 DELETE에 적용하면 데이터 변경도 실행하므로 학습용 데이터에서 확인해야 한다.

| 볼 항목 | 해석 |
| --- | --- |
| Seq Scan / Index Scan | 테이블을 순차적으로 읽는지, 인덱스로 행에 접근하는지 |
| Sort / 조인 노드 | 정렬과 결합 방식, 처리하는 행 수 |
| cost / rows | 비용과 결과 행 수의 추정치. cost는 밀리초가 아님 |
| actual time / rows / loops | 실제 시간, 행 수, 반복 횟수. 반복 노드의 시간·행 수는 회당 평균 |
| Buffers | 버퍼 접근·읽기 정보. read만으로 물리 디스크 접근을 단정하지 않음 |
| Heap Fetches | Index Only Scan에서 발생한 heap 접근 |

노드 시간에는 하위 노드의 작업이 포함될 수 있어 단순 합산하지 않는다. 추정 행 수와 실제 행 수가 크게 다른지, 반복 횟수가 많은지, 정렬이 디스크로 넘어가는지부터 살펴본다. [EXPLAIN 사용법](https://www.postgresql.org/docs/17/using-explain.html)을 기준으로 정리했다.

---

## 10. Query 최적화: 먼저 결과를 지키고, 그다음 읽는 양을 줄이기

인덱스를 추가하기 전에 쿼리가 어떤 결과를 내야 하는지 확인해야 한다. 빠른 쿼리라도 고객이나 주문이 빠지거나 중복 집계되면 잘못된 결과다.

최적화는 다음 순서로 진행할 수 있다.

1. 느린 요청의 SQL, 호출 횟수, 응답 시간과 반환 크기를 확인한다.
2. 대표적인 데이터 분포에서 실행 계획과 실제 처리량을 확인한다.
3. 필요 없는 컬럼·행, 반복 조회, 정렬 비용을 줄인다.
4. 조건과 정렬에 맞는 인덱스를 후보로 비교한다.
5. 같은 조건에서 결과와 성능을 다시 확인하고 쓰기 비용도 살핀다.

예를 들어 하루의 주문을 찾을 때 컬럼을 함수로 변환하는 대신 범위로 표현하면 일반 `created_at` 인덱스를 활용할 여지가 생긴다. 여기서는 시간대 없는 TIMESTAMP 컬럼을 사용한다.

```sql
-- 같은 날짜를 고르지만 컬럼을 변환하는 표현
SELECT id FROM orders
WHERE CAST(created_at AS DATE) = DATE '2026-10-11';

-- 시작 포함, 다음 날 시작 제외
SELECT id FROM orders
WHERE created_at >= TIMESTAMP '2026-10-11 00:00:00'
  AND created_at < TIMESTAMP '2026-10-12 00:00:00';
```

위 고객별 복합 인덱스가 날짜만 조회하는 이 쿼리에도 최적이라는 뜻은 아니다. 날짜 중심 조회가 많다면 별도의 인덱스 후보를 비교한다. 시간대가 있는 컬럼에서는 업무 날짜의 시간대를 반영한 경계를 만들어야 한다. 표현식 인덱스도 대안이므로 함수 사용 자체가 항상 잘못된 것은 아니다.

큰 `OFFSET`은 반환하지 않을 앞선 행도 처리하게 할 수 있다. 고객별 최신 주문은 마지막으로 받은 정렬 키를 기준으로 다음 페이지를 찾는 방식도 고려할 수 있다.

```sql
SELECT id, created_at, total_amount
FROM orders
WHERE customer_id = 42
  AND (created_at, id) < (TIMESTAMP '2026-10-11 10:00:00', 101)
ORDER BY created_at DESC, id DESC
LIMIT 20;
```

이는 `created_at`, `id`가 NULL이 아니고 내림차순으로 정렬된 앞선 예제에 맞춘 커서 방식이다. 임의 페이지 이동에는 제약이 있으며, 페이지 사이에 데이터가 바뀌면 하나의 고정된 스냅샷과 같은 결과를 보장하지 않는다.

애플리케이션에서 주문 20개를 조회한 뒤 주문별 추가 조회를 20번 보내는 문제는 쿼리 하나의 인덱스만으로 해결되지 않는다. 필요한 관계를 JOIN이나 일괄 조회로 가져올 수 있는지 확인하되, JOIN으로 늘어나는 행과 전송량도 함께 살펴야 한다.

---

## 전체 흐름 정리

```text
HTTP 주문 요청
    ↓
데이터의 의미와 관계에 맞춰 저장 모델 선택
    ↓
PK / FK 등으로 식별과 관계의 규칙 적용
    ↓
정규화로 같은 사실의 중복 관리 줄이기
    ↓
JOIN으로 필요한 관계 결합
    ↓
조회 조건과 정렬에 맞는 인덱스 검토
    ↓
EXPLAIN으로 실행 경로, 실제 처리량 확인
    ↓
결과 정합성과 읽기·쓰기 비용 비교
```

이번 주제에서는 데이터의 의미와 접근 방법을 함께 봐야 한다는 점이 기억에 남았다. 주문 당시 가격과 현재 상품 가격은 다른 사실이고, FK가 있다고 조회 인덱스까지 준비되는 것은 아니다. 인덱스를 추가했다는 사실보다 어떤 조회에서 얼마나 덜 읽게 됐는지를 확인해야 설계와 최적화의 이유를 설명할 수 있다.

## 참고 자료

- [MongoDB — Embedded Data](https://www.mongodb.com/docs/manual/data-modeling/embedding/): 문서 포함과 조회 패턴
- [MongoDB — Transactions](https://www.mongodb.com/docs/manual/core/transactions/): 단일 문서 원자성과 다중 문서 트랜잭션
- [Microsoft Learn — Database normalization description](https://learn.microsoft.com/en-us/previous-versions/troubleshoot/microsoft-365/microsoft-365-apps/access/database-normalization-description): 정규화와 변경 이상
- [PostgreSQL 17 — Constraints](https://www.postgresql.org/docs/17/ddl-constraints.html): PK, FK와 인덱스 생성 규칙
- [PostgreSQL 17 — Table Expressions](https://www.postgresql.org/docs/17/queries-table-expressions.html): JOIN, ON과 WHERE
- [PostgreSQL 17 — Index Types](https://www.postgresql.org/docs/17/indexes-types.html): 인덱스 종류와 지원 조건
- [PostgreSQL 17 — B-Tree Indexes](https://www.postgresql.org/docs/17/btree.html): 내부·리프 페이지 구조
- [MySQL 8.4 — Clustered and Secondary Indexes](https://dev.mysql.com/doc/refman/8.4/en/innodb-index-types.html): InnoDB 인덱스의 리프 데이터
- [PostgreSQL 17 — Multicolumn Indexes](https://www.postgresql.org/docs/17/indexes-multicolumn.html): 복합 인덱스의 컬럼 순서
- [PostgreSQL 17 — Index-Only Scans and Covering Indexes](https://www.postgresql.org/docs/17/indexes-index-only-scans.html): INCLUDE, Visibility Map, heap 접근
- [PostgreSQL 17 — Using EXPLAIN](https://www.postgresql.org/docs/17/using-explain.html): 추정 계획과 실제 실행 측정
