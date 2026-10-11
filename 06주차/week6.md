# 6주차 - Database

## 1. RDB / NoSQL

데이터베이스는 데이터를 저장하고 관리하기 위한 시스템으로, 대표적으로 RDB와 NoSQL로 구분할 수 있다.

### RDB(Relational Database)

> 데이터를 행(Row)과 열(Column)로 구성된 테이블에 저장하고, 테이블 간 관계를 통해 데이터를 관리하는 관계형 데이터베이스이다.

대표적으로 MySQL, PostgreSQL, Oracle 등이 있다.

- **Schema**: 컬럼, 데이터 타입, 제약 조건 등을 정의
- **Relationship**: PK와 FK를 통해 테이블 간 관계 표현
- **SQL**: 데이터 조회 및 변경에 사용하는 언어
- **Transaction**: 여러 작업을 하나의 논리적 단위로 처리

RDB는 데이터 간 관계와 정합성이 중요한 시스템에서 주로 사용된다.

### NoSQL(Not Only SQL)

> 관계형 테이블 구조에 한정되지 않고 다양한 데이터 모델을 사용하는 데이터베이스이다.

| 종류 | 데이터 구조 | 대표 예시 |
|---|---|---|
| Key-Value | Key와 Value | Redis |
| Document | JSON과 유사한 문서 | MongoDB |
| Wide-Column | Column Family | Cassandra |
| Graph | Node와 Edge | Neo4j |

NoSQL은 유연한 데이터 구조나 분산 처리에 적합한 제품이 많다. 다만 모든 NoSQL이 Schema가 없거나 Transaction을 지원하지 않는 것은 아니다.

### RDB vs NoSQL

| 구분 | RDB | NoSQL |
|---|---|---|
| 데이터 모델 | 관계형 테이블 | Key-Value, Document 등 |
| Schema | 명시적인 구조 중심 | 제품에 따라 유연한 구조 지원 |
| 관계 표현 | FK, JOIN | 데이터 모델에 따라 다름 |
| 확장 방식 | 수직/수평 확장 | 수평 확장에 특화된 제품이 많음 |
| Transaction | 일반적으로 강력한 지원 | 제품에 따라 지원 범위가 다름 |

실제 서비스에서는 데이터의 특성에 따라 두 방식을 함께 사용하기도 한다.

```text
MySQL
→ 사용자, 주문, 결제 데이터

Redis
→ Cache, Session, 임시 데이터
```

데이터베이스 선택에서는 단순히 성능뿐 아니라 **데이터 구조, 정합성 요구사항, 조회 패턴, 확장 방식** 등을 고려해야 한다.

## 2. PK / FK

### Primary Key(PK)

> 테이블에서 각 Row를 유일하게 식별하기 위한 키이다.

PK는 다음 조건을 만족해야 한다.

- **Unique**: 중복된 값을 허용하지 않음
- **NOT NULL**: NULL 값을 허용하지 않음

하나의 컬럼으로 구성하거나 여러 컬럼을 조합한 **Composite Primary Key(복합 기본키)**로 구성할 수 있다.

### Foreign Key(FK)

> 다른 테이블의 키를 참조하여 테이블 간 관계를 정의하는 키이다.

**Users**

| id (PK) | name |
|---|---|
| 1 | Kim |
| 2 | Lee |

**Orders**

| id (PK) | user_id (FK) | product |
|---|---|---|
| 101 | 1 | Keyboard |
| 102 | 1 | Mouse |
| 103 | 2 | Monitor |

`Orders.user_id`가 `Users.id`를 참조하므로 사용자와 주문은 1:N 관계를 가진다.

```text
Users
  │
  │ 1:N
  ↓
Orders
```

### 참조 무결성(Referential Integrity)

FK의 주요 역할은 **존재하지 않는 데이터를 참조하지 않도록 제약하는 것**이다.

Users 테이블에 `id = 999`인 사용자가 없다면, FK 제약 조건이 설정된 Orders 테이블에는 `user_id = 999`인 주문을 삽입할 수 없다.

참조 중인 데이터를 삭제할 때는 다음과 같은 정책을 설정할 수 있다.

| 옵션 | 동작 |
|---|---|
| RESTRICT / NO ACTION | 참조 중인 Row가 있으면 삭제 제한 |
| CASCADE | 참조하는 Row도 함께 삭제 |
| SET NULL | 참조하는 FK 값을 NULL로 변경 |

주문이나 결제처럼 이력을 보존해야 하는 데이터에서는 CASCADE 사용에 주의해야 한다.

### PK와 FK의 차이

| 구분 | PK | FK |
|---|---|---|
| 목적 | Row 식별 | 테이블 간 관계 표현 |
| 중복 | 불가능 | 가능 |
| NULL | 불가능 | 설정에 따라 가능 |
| 개수 | 테이블당 하나의 PK 제약 조건 | 여러 FK 설정 가능 |

MySQL InnoDB에서는 PK를 기반으로 Clustered Index가 구성된다. FK 역시 참조 무결성 검사를 위한 Index가 필요하며, 적절한 Index가 없으면 자동으로 생성될 수 있다.

## 3. 정규화

> 정규화(Normalization)는 데이터 중복을 줄이고 데이터의 일관성을 유지하기 위해 테이블을 체계적으로 분리하는 과정이다.

### 정규화가 필요한 이유

**Orders**

| order_id | user_id | user_name | user_email | product |
|---|---|---|---|---|
| 101 | 1 | Kim | kim@example.com | Keyboard |
| 102 | 1 | Kim | kim@example.com | Mouse |
| 103 | 2 | Lee | lee@example.com | Monitor |

사용자 정보가 주문마다 반복 저장되면서 다음과 같은 **이상 현상(Anomaly)**이 발생할 수 있다.

- **갱신 이상(Update Anomaly)**: 이메일 변경 시 일부 Row만 수정하면 데이터 불일치 발생
- **삽입 이상(Insert Anomaly)**: 주문이 없는 사용자의 정보를 저장하기 어려움
- **삭제 이상(Delete Anomaly)**: 마지막 주문을 삭제하면 사용자 정보까지 사라질 수 있음

이를 해결하기 위해 사용자 정보와 주문 정보를 별도 테이블로 분리한다.

```text
정규화 이전

Orders
├─ 주문 정보
└─ 사용자 정보


정규화 이후

Users
  │
  │ 1:N
  ↓
Orders
```

### 제1정규형(1NF)

> 각 컬럼의 값이 원자적인 값으로 구성되도록 하는 정규형이다.

하나의 컬럼에 여러 값을 목록 형태로 저장하지 않고 개별 값으로 분리한다.

**1NF 위반**

| order_id | products |
|---|---|
| 101 | Keyboard, Mouse |

**1NF 적용**

| order_id | product |
|---|---|
| 101 | Keyboard |
| 101 | Mouse |

### 제2정규형(2NF)

> 1NF를 만족하면서, 키가 아닌 모든 속성이 후보키 전체에 완전 함수 종속되도록 하는 정규형이다.

핵심은 **복합키의 일부에만 의존하는 속성을 분리하는 것**이다.

| order_id | product_id | product_name | quantity |
|---|---|---|---|
| 101 | 1 | Keyboard | 2 |
| 101 | 2 | Mouse | 1 |
| 102 | 1 | Keyboard | 1 |

`(order_id, product_id)`가 복합 PK인 경우 다음과 같은 종속 관계가 존재한다.

```text
(order_id, product_id) → quantity

product_id → product_name
```

`product_name`은 복합키 전체가 아닌 `product_id`에만 의존한다. 이를 **부분 함수 종속(Partial Functional Dependency)**이라고 한다.

따라서 상품 정보를 Products 테이블로 분리하여 중복을 줄일 수 있다.

### 제3정규형(3NF)

> 2NF를 만족하면서, 키가 아닌 속성 사이의 불필요한 이행적 함수 종속을 제거하는 정규형이다.

| employee_id | employee_name | department_id | department_name |
|---|---|---|---|
| 1 | Kim | 10 | Development |
| 2 | Lee | 10 | Development |
| 3 | Park | 20 | Design |

`employee_id`가 PK이고, 부서 ID가 부서 이름을 결정한다면 다음과 같은 관계가 존재한다.

```text
employee_id
     ↓
department_id
     ↓
department_name
```

`department_name`이 `employee_id`에 간접적으로 의존하는 것을 **이행적 함수 종속(Transitive Functional Dependency)**이라고 한다.

따라서 Employees와 Departments 테이블로 분리하여 부서 정보의 중복을 줄일 수 있다.

### 정규형 비교

| 정규형 | 핵심 |
|---|---|
| 1NF | 컬럼에 원자적인 값 저장 |
| 2NF | 복합키 일부에만 의존하는 속성 제거 |
| 3NF | 키가 아닌 속성 간 이행적 종속 제거 |

### 반정규화(Denormalization)

> 조회 성능이나 데이터 접근 편의성을 높이기 위해 의도적으로 데이터 중복을 허용하는 설계 방식이다.

정규화로 테이블을 분리하면 데이터 중복은 줄어들지만, 여러 테이블의 정보를 함께 조회하기 위해 JOIN이 필요할 수 있다.

반정규화는 일부 데이터를 중복 저장하여 JOIN을 줄이거나 조회 성능을 개선하는 방법이다.

| 구분 | 정규화 | 반정규화 |
|---|---|---|
| 목적 | 중복 감소, 데이터 일관성 | 조회 성능 및 편의성 개선 |
| 데이터 중복 | 줄어듦 | 증가할 수 있음 |
| JOIN | 필요해질 수 있음 | 줄어들 수 있음 |
| 주의점 | 조회 구조가 복잡해질 수 있음 | 데이터 불일치 가능성 |

**정규화를 기본으로 설계하되, 실제 조회 패턴과 성능을 분석하여 필요한 경우 반정규화를 적용**하는 것이 일반적이다.

## 4. JOIN

> JOIN은 두 개 이상의 테이블을 특정 조건에 따라 결합하여 데이터를 조회하는 SQL 연산이다.

관계형 데이터베이스에서는 정규화 등을 통해 데이터를 여러 테이블로 분리한다. JOIN은 이렇게 분리된 테이블의 데이터를 함께 조회할 때 사용한다.

### JOIN의 종류

**Users**

| id | name |
|---|---|
| 1 | Kim |
| 2 | Lee |
| 3 | Park |

**Orders**

| id | user_id | product |
|---|---|---|
| 101 | 1 | Keyboard |
| 102 | 1 | Mouse |
| 103 | 2 | Monitor |

**1. INNER JOIN**

두 테이블에서 JOIN 조건을 만족하는 Row만 반환한다.

```sql
SELECT u.name, o.product
FROM Users u
INNER JOIN Orders o
    ON u.id = o.user_id;
```

| name | product |
|---|---|
| Kim | Keyboard |
| Kim | Mouse |
| Lee | Monitor |

주문이 없는 Park는 결과에서 제외된다.

**2. LEFT JOIN**

왼쪽 테이블의 모든 Row를 반환하며, 오른쪽 테이블에 매칭되는 데이터가 없으면 NULL로 표시한다.

```sql
SELECT u.name, o.product
FROM Users u
LEFT JOIN Orders o
    ON u.id = o.user_id;
```

| name | product |
|---|---|
| Kim | Keyboard |
| Kim | Mouse |
| Lee | Monitor |
| Park | NULL |

**3. RIGHT JOIN**

오른쪽 테이블의 모든 Row를 반환하며, 왼쪽 테이블에 매칭되는 데이터가 없으면 NULL로 표시한다.

LEFT JOIN과 테이블의 방향만 반대이므로 테이블 순서를 변경하면 LEFT JOIN으로도 표현할 수 있다.

**4. CROSS JOIN**

두 테이블의 모든 Row 조합을 반환한다.

```sql
SELECT *
FROM Users
CROSS JOIN Orders;
```

Users가 3건, Orders가 3건이라면 총 9건의 결과가 생성된다.

### JOIN 종류 비교

| 종류 | 특징 |
|---|---|
| INNER JOIN | 양쪽 테이블에서 조건을 만족하는 Row만 반환 |
| LEFT JOIN | 왼쪽 테이블의 모든 Row 유지 |
| RIGHT JOIN | 오른쪽 테이블의 모든 Row 유지 |
| CROSS JOIN | 두 테이블의 모든 Row 조합 반환 |

### ON과 WHERE의 차이

`ON`은 JOIN 조건을 정의하고, `WHERE`는 조회 결과를 필터링한다.

특히 **LEFT JOIN에서는 조건의 위치에 따라 결과가 달라질 수 있다.**

**ON에 조건을 작성한 경우**

```sql
SELECT u.name, o.product
FROM Users u
LEFT JOIN Orders o
    ON u.id = o.user_id
    AND o.product = 'Keyboard';
```

| name | product |
|---|---|
| Kim | Keyboard |
| Lee | NULL |
| Park | NULL |

LEFT JOIN이므로 Keyboard 주문이 없는 사용자도 결과에 포함된다.

**WHERE에 조건을 작성한 경우**

```sql
SELECT u.name, o.product
FROM Users u
LEFT JOIN Orders o
    ON u.id = o.user_id
WHERE o.product = 'Keyboard';
```

| name | product |
|---|---|
| Kim | Keyboard |

WHERE 조건을 만족하지 않는 NULL Row가 제외되므로 INNER JOIN과 같은 결과가 나타난다.

### JOIN 알고리즘

SQL에서 JOIN을 작성하더라도 실제 데이터를 결합하는 방식은 DBMS의 **Query Optimizer**가 결정한다.

대표적인 JOIN 알고리즘은 다음과 같다.

**1. Nested Loop Join**

한쪽 테이블의 Row를 기준으로 다른 테이블에서 매칭되는 Row를 반복적으로 탐색한다.

```text
Users
  │
  ├─ User 1 → Orders 탐색
  ├─ User 2 → Orders 탐색
  └─ User 3 → Orders 탐색
```

내부 테이블의 JOIN 컬럼에 Index가 있다면 전체 테이블을 반복 탐색하지 않고 Index를 활용할 수 있다.

이를 **Index Nested Loop Join**이라고 한다.

**2. Hash Join**

한쪽 테이블의 JOIN Key로 Hash Table을 생성한 뒤, 다른 테이블의 Key를 이용해 매칭되는 데이터를 찾는다.

```text
Users
  ↓
Hash Table 생성 (Build)
  ↓
Orders.user_id로 탐색 (Probe)
  ↓
JOIN 결과 생성
```

적절한 Index가 없거나 많은 데이터를 결합하는 상황에서 유리할 수 있지만, Hash Table을 위한 메모리가 필요하다.

**3. Merge Join**

JOIN Key를 기준으로 정렬된 두 입력을 순차적으로 비교하며 결합한다.

```text
Users.id        Orders.user_id
   1                 1
   2                 1
   3                 2
                     3
        ↓
정렬된 Key를 비교하며 JOIN
```

이미 정렬된 데이터를 결합할 때 효율적이지만, 별도의 정렬이 필요하면 추가 비용이 발생한다.

MySQL 8.0은 Nested Loop Join과 Hash Join을 지원하지만, 일반적인 Sort-Merge Join은 지원하지 않는다.

| 알고리즘 | 동작 방식 | 특징 |
|---|---|---|
| Nested Loop | Row별 반복 탐색 | Index가 있으면 효율적 |
| Hash Join | Hash Table 생성 및 탐색 | 대량 데이터 결합에 유리할 수 있음 |
| Merge Join | 정렬된 Key 순차 비교 | 정렬된 입력 활용 |

### JOIN과 Index

JOIN 성능은 JOIN 대상 데이터의 크기와 Index 유무에 영향을 받는다.

```sql
SELECT u.name, o.product
FROM Users u
JOIN Orders o
    ON u.id = o.user_id;
```

`Orders.user_id`에 Index가 있다면 DBMS는 이를 이용해 특정 사용자의 주문을 효율적으로 찾을 수 있다.

```sql
CREATE INDEX idx_orders_user_id
ON Orders(user_id);
```

```text
Index가 없는 경우
→ Orders 전체 탐색 가능성

Index가 있는 경우
→ user_id Index를 이용한 탐색 가능
```

다만 Index가 존재하더라도 Optimizer가 Full Table Scan이 더 효율적이라고 판단하면 Index를 사용하지 않을 수 있다.

### JOIN과 N+1 문제

N+1 문제는 연관된 데이터를 조회할 때 최초 Query 1번 이후 각 데이터에 대해 추가 Query가 N번 실행되는 문제이다.

```text
Users 목록 조회       → 1번
User 1의 Orders 조회  → 1번
User 2의 Orders 조회  → 1번
User 3의 Orders 조회  → 1번

총 N+1번 Query 실행
```

이를 개선하는 대표적인 방법은 다음과 같다.

**1. JOIN을 이용한 일괄 조회**

```sql
SELECT u.id, u.name, o.product
FROM Users u
LEFT JOIN Orders o
    ON u.id = o.user_id;
```

하나의 Query로 사용자와 주문을 함께 조회한다.

다만 1:N 관계에서는 부모 데이터가 자식 수만큼 중복 반환될 수 있다.

**2. IN Query를 이용한 일괄 조회**

```sql
SELECT *
FROM Orders
WHERE user_id IN (1, 2, 3);
```

사용자 목록을 먼저 조회한 뒤 관련 주문을 한 번에 조회하고, 애플리케이션에서 `user_id`를 기준으로 그룹핑한다.

```text
Users 조회
    ↓
User ID 목록 추출
    ↓
Orders 일괄 조회
    ↓
user_id 기준 그룹핑
```

N+1 문제는 단순히 JOIN을 추가하는 것보다 **Query 실행 횟수, 반환 데이터 크기, 연관관계, Pagination 여부** 등을 고려해 적절한 조회 방식을 선택하는 것이 중요하다.

## 5. Index

> Index는 데이터베이스에서 특정 데이터를 빠르게 찾기 위해 사용하는 자료구조이다.

Index가 없다면 DBMS는 조건에 맞는 데이터를 찾기 위해 테이블 전체를 탐색해야 할 수 있다. 이를 **Full Table Scan**이라고 한다.

반면 Index를 사용하면 전체 데이터를 확인하지 않고도 원하는 데이터를 효율적으로 찾을 수 있다.

### Index의 동작 원리

100만 건의 데이터가 저장된 Users 테이블에서 특정 이메일을 조회하는 경우이다.

```sql
SELECT *
FROM Users
WHERE email = 'joon@example.com';
```

```text
Index가 없는 경우

Users Table
  ↓
Row를 순차적으로 확인
  ↓
조건에 맞는 데이터 발견


Index가 있는 경우

Email Index
  ↓
joon@example.com 탐색
  ↓
해당 Row 조회
```

MySQL InnoDB의 일반적인 Index는 **B+Tree**를 기반으로 구성된다.

B+Tree는 Key를 정렬된 상태로 관리하므로 정확한 값 검색뿐 아니라 범위 검색에도 활용할 수 있다.

### Index 생성

```sql
CREATE INDEX idx_users_email
ON Users(email);
```

`email` 컬럼에 Index가 생성된다.

다만 Index가 존재하더라도 DBMS가 반드시 사용하는 것은 아니다. **Query Optimizer**가 데이터 크기와 분포, 조회 조건 등을 고려하여 실행 계획을 결정한다.

### Clustered Index

> Clustered Index는 실제 데이터 Row를 Index의 Leaf Node에 저장하는 구조이다.

MySQL InnoDB에서는 Primary Key를 기준으로 Clustered Index가 구성된다.

```text
Clustered Index (PK)
        ↓
B+Tree 탐색
        ↓
Leaf Node
        ↓
실제 Row 데이터
```

따라서 PK를 이용한 조회는 Clustered Index를 탐색하여 데이터를 찾을 수 있다.

InnoDB에서는 테이블의 데이터가 Clustered Index 구조로 관리되므로 테이블당 하나의 Clustered Index만 존재한다.

### Secondary Index

> Secondary Index는 Clustered Index와 별도로 생성되는 Index이다.

InnoDB의 Secondary Index는 Leaf Node에 Index Key와 해당 Row의 PK 값을 저장한다.

```text
Secondary Index (email)

email                  PK
──────────────────────────
a@example.com          3
joon@example.com       10
z@example.com          25
```

Secondary Index에서 검색한 뒤 실제 Row의 다른 컬럼이 필요하다면 PK를 이용해 Clustered Index를 다시 탐색한다.

```text
email 검색
    ↓
Secondary Index
    ↓
PK 값 확인
    ↓
Clustered Index 탐색
    ↓
실제 Row 조회
```

반면 Secondary Index에 저장된 정보만으로 Query를 처리할 수 있다면 추가적인 Clustered Index 탐색이 필요하지 않다. 이를 **Covering Index**와 연결해서 이해할 수 있다.

### Index의 장단점

| 장점 | 단점 |
|---|---|
| 검색 성능 개선 | 추가 저장 공간 필요 |
| 범위 검색에 유리 | INSERT/UPDATE/DELETE 시 Index 갱신 비용 |
| 정렬 비용 감소 가능 | Index가 많을수록 유지 비용 증가 |

Index는 조회 성능을 개선하지만 데이터가 변경될 때마다 관련 Index도 갱신해야 한다.

따라서 모든 컬럼에 Index를 생성하는 것이 아니라 **조회 빈도, 검색 조건, 데이터 분포, 쓰기 비용** 등을 고려해야 한다.

### Cardinality와 Selectivity

**Cardinality**는 컬럼에 존재하는 서로 다른 값의 개수이다.

```text
Users.email
→ 서로 다른 값 100만 개

Users.status
→ 서로 다른 값 3개
```

일반적으로 Cardinality가 높은 컬럼은 특정 값을 조회할 때 검색 범위를 좁히기 유리하다.

**Selectivity**는 검색 조건이 얼마나 많은 Row를 걸러낼 수 있는지를 나타낸다.

예를 들어 전체 100만 건 중 10건만 조회된다면 검색 범위를 크게 줄일 수 있다.

```text
email = 'joon@example.com'
→ 1건 조회
→ Index 활용에 유리

status = 'ACTIVE'
→ 전체의 95% 조회
→ Full Table Scan이 유리할 수 있음
```

Selectivity는 문헌에 따라 정의하는 비율의 방향이 다를 수 있다. 핵심은 **조건에 해당하는 Row가 적을수록 Index를 통한 탐색이 유리한 경우가 많다**는 점이다.

---

## 6. B+Tree

> B+Tree는 데이터를 정렬된 상태로 관리하는 균형 탐색 트리로, 데이터베이스의 Index에서 널리 사용하는 자료구조이다.

MySQL InnoDB의 일반적인 Index는 B+Tree를 기반으로 구성된다.

### B+Tree의 구조

B+Tree는 Root Node, Internal Node, Leaf Node로 구성된다.

```text
                  [30 | 60]
                 /    |    \
                /     |     \
          [10, 20] [30, 40, 50] [60, 70, 80]
             Leaf      Leaf       Leaf
```

- **Root Node**: 탐색이 시작되는 최상위 Node
- **Internal Node**: 검색 범위를 결정하는 중간 Node
- **Leaf Node**: 실제 Index Entry가 저장되는 Node

B+Tree는 모든 Leaf Node가 같은 깊이에 위치하는 균형 트리이다.

또한 Leaf Node가 Key 순서대로 연결되어 있어 범위 검색에 유리하다.

### B+Tree의 탐색 과정

`45`를 검색하는 경우이다.

```text
                 [30 | 60]
                /    |    \
               /     |     \
        [10, 20] [30, 40, 45, 50] [60, 70]
```

```text
45 검색
   ↓
Root Node [30 | 60]
   ↓
30 <= 45 < 60
   ↓
가운데 Leaf Node
   ↓
45 발견
```

전체 데이터를 순차적으로 확인하지 않고 Key를 비교하며 탐색 범위를 줄인다.

B+Tree의 검색 시간복잡도는 일반적으로 **O(log N)**이다.

### B+Tree가 빠른 이유

B+Tree는 하나의 Node에 여러 Key와 Child Pointer를 저장할 수 있다.

하나의 Node가 가질 수 있는 자식 Node의 수를 **Fan-out**이라고 한다.

```text
Binary Tree

       50
      /  \
     30   70
    / \   / \
   10 40 60 90


B+Tree

       [30 | 60]
      /    |    \
     /     |     \
  [10,20] [30,40,50] [60,70]
```

B+Tree는 Fan-out이 크기 때문에 데이터가 많아져도 Tree의 높이를 낮게 유지할 수 있다.

데이터베이스에서는 디스크 접근 비용이 크므로 **탐색에 필요한 Page 수를 줄이는 것**이 중요하다.

```text
Root Page
    ↓
Internal Page
    ↓
Leaf Page
    ↓
데이터 조회
```

MySQL InnoDB에서는 기본적으로 16KB 크기의 Page를 사용하며, B+Tree의 Node도 Page 단위로 관리된다.

### 범위 검색에 유리한 이유

B+Tree의 Leaf Node는 Key 순서대로 연결되어 있다.

```text
[10, 20] → [30, 40] → [50, 60] → [70, 80]
```

따라서 시작 Key를 찾은 뒤 인접한 Leaf Node를 순차적으로 탐색할 수 있다.

```sql
SELECT *
FROM Users
WHERE id BETWEEN 30 AND 60;
```

```text
30 탐색
   ↓
Leaf Node 도착
   ↓
30 → 40 → 50 → 60
```

이러한 구조는 정확한 값 검색뿐 아니라 범위 검색과 정렬에도 활용할 수 있다.

### B-Tree와 B+Tree의 차이

| 구분 | B-Tree | B+Tree |
|---|---|---|
| 데이터 저장 | Internal / Leaf Node | Leaf Node |
| 검색 종료 | Internal Node에서도 가능 | Leaf Node에서 종료 |
| 범위 검색 | 상대적으로 불리 | Leaf Node 연결 활용 |
| 특징 | Key와 데이터 함께 관리 가능 | 높은 Fan-out과 순차 탐색에 유리 |

B+Tree는 Internal Node에 실제 Row 대신 검색을 위한 Key와 Pointer를 저장하므로 하나의 Page에서 더 많은 분기를 관리할 수 있다.

### Hash Index와의 차이

Hash Index는 Hash Function을 이용해 특정 Key에 접근하는 구조이다.

| 구분 | B+Tree Index | Hash Index |
|---|---|---|
| 검색 방식 | Tree 탐색 | Hash Function |
| 정확한 값 검색 | O(log N) | 평균 O(1) |
| 범위 검색 | 가능 | 일반적으로 부적합 |
| 정렬 활용 | 가능 | 어려움 |

Hash Index는 정확한 값 검색에 유리하지만 Key의 정렬 순서를 유지하지 않는다.

반면 B+Tree는 정렬된 구조를 이용해 `=`, `>`, `<`, `BETWEEN` 등의 조건을 효율적으로 처리할 수 있다.

### B+Tree의 INSERT

B+Tree는 데이터 삽입 시 Key의 정렬 순서를 유지해야 한다.

Leaf Node에 공간이 부족하면 **Node Split**이 발생할 수 있다.

```text
기존 Leaf Node
[10, 20, 30, 40]

50 INSERT
    ↓
공간 부족
    ↓
Node Split
    ↓
[10, 20] → [30, 40, 50]
```

Node Split이 발생하면 새로운 Page를 할당하고 상위 Node의 Key와 Pointer를 갱신해야 할 수 있다.

이러한 작업은 INSERT 성능에 영향을 준다.

특히 InnoDB에서는 PK가 Clustered Index의 기준이 되므로 PK 값의 삽입 순서도 중요하다.

```text
순차 증가 PK
1 → 2 → 3 → 4 → 5

무작위 PK
100 → 20 → 500 → 75 → 300
```

순차 증가 PK는 주로 Index의 오른쪽 끝에 삽입되는 반면, 무작위 PK는 여러 위치에 삽입되어 Page Split이나 분산된 Page 접근이 증가할 수 있다.

따라서 무작위 UUID를 PK로 사용하는 경우 순차 증가 PK보다 Index 관리 비용이 커질 수 있다. 다만 실제 성능은 데이터 크기와 캐시, UUID 형식 등에 따라 달라진다.

## 7. Composite / Covering Index

### Composite Index(복합 인덱스)

> Composite Index는 두 개 이상의 컬럼을 조합하여 생성하는 Index이다.

여러 컬럼을 함께 검색하는 Query에서 활용할 수 있다.

```sql
CREATE INDEX idx_orders_user_status
ON Orders(user_id, status);
```

복합 인덱스는 컬럼을 지정한 순서대로 정렬된다.

```text
Index(user_id, status)

user_id | status
--------|--------
1       | CANCELLED
1       | COMPLETED
1       | PENDING
2       | COMPLETED
2       | PENDING
3       | COMPLETED
```

먼저 `user_id`를 기준으로 정렬하고, 같은 `user_id` 안에서 `status`를 기준으로 정렬하는 구조이다.

### Leftmost Prefix Rule(최좌측 접두사 규칙)

> B+Tree 기반 복합 인덱스는 일반적으로 가장 왼쪽 컬럼부터 연속적으로 사용하는 조건에서 효율적으로 탐색할 수 있다.

`INDEX(user_id, status)`가 생성된 경우이다.

**1. 첫 번째 컬럼으로 조회**

```sql
SELECT *
FROM Orders
WHERE user_id = 1;
```

`user_id`가 인덱스의 첫 번째 컬럼이므로 Index를 활용할 수 있다.

**2. 두 컬럼으로 조회**

```sql
SELECT *
FROM Orders
WHERE user_id = 1
  AND status = 'COMPLETED';
```

두 컬럼을 모두 활용하여 검색 범위를 좁힐 수 있다.

**3. 두 번째 컬럼만 조회**

```sql
SELECT *
FROM Orders
WHERE status = 'COMPLETED';
```

`status`는 첫 번째 컬럼인 `user_id`에 종속된 정렬 순서를 가지므로 일반적인 Index 탐색에서는 효율적으로 활용하기 어렵다.

```text
INDEX(user_id, status)

WHERE user_id = ?
→ 효율적인 탐색 가능

WHERE user_id = ? AND status = ?
→ 두 컬럼을 이용한 탐색 가능

WHERE status = ?
→ 일반적인 탐색에는 불리
```

단, MySQL 8.0에서는 조건과 데이터 분포에 따라 **Index Skip Scan**을 이용해 선행 컬럼 없이도 복합 인덱스를 활용할 수 있다.

따라서 최좌측 접두사 규칙은 기본적인 설계 원칙이며, 실제 Index 사용 여부는 실행 계획으로 확인해야 한다.

### 복합 인덱스의 컬럼 순서

복합 인덱스는 컬럼 순서에 따라 활용할 수 있는 검색 조건이 달라진다.

```sql
CREATE INDEX idx_orders_user_date
ON Orders(user_id, created_at);
```

다음 Query는 두 컬럼을 이용해 검색 범위를 좁힐 수 있다.

```sql
SELECT *
FROM Orders
WHERE user_id = 1
  AND created_at >= '2026-01-01';
```

반면 컬럼 순서를 반대로 구성하면 다른 조회 패턴에 유리해진다.

```sql
CREATE INDEX idx_orders_date_user
ON Orders(created_at, user_id);
```

이 Index는 날짜를 기준으로 조회하는 Query에 적합할 수 있다.

```sql
SELECT *
FROM Orders
WHERE created_at >= '2026-01-01';
```

복합 인덱스의 컬럼 순서는 **실제 Query의 WHERE 조건과 데이터 분포**를 기준으로 결정해야 한다.

### Equality와 Range 조건

복합 인덱스에서는 일반적으로 동등 조건(`=`)을 먼저 활용하고, 이후 범위 조건을 사용하는 구성이 효율적이다.

```sql
CREATE INDEX idx_orders_user_date
ON Orders(user_id, created_at);
```

```sql
SELECT *
FROM Orders
WHERE user_id = 1
  AND created_at >= '2026-01-01';
```

```text
user_id = 1
    ↓
해당 사용자의 데이터로 범위 축소
    ↓
created_at 범위 탐색
```

반면 선행 컬럼에 범위 조건이 적용되면 후속 컬럼을 Index 탐색 범위를 좁히는 데 활용하기 어려울 수 있다.

```sql
CREATE INDEX idx_orders_date_user
ON Orders(created_at, user_id);
```

```sql
SELECT *
FROM Orders
WHERE created_at >= '2026-01-01'
  AND user_id = 1;
```

이 경우 `created_at` 범위를 탐색한 뒤 `user_id` 조건을 추가로 검사할 수 있다.

후속 컬럼이 Index Condition Pushdown 등에 활용될 수는 있지만, 반드시 두 컬럼 모두 탐색 범위를 줄이는 데 사용되는 것은 아니다.

### Covering Index(커버링 인덱스)

> Covering Index는 Query에 필요한 모든 컬럼이 Index에 포함되어 있어 실제 테이블 Row를 추가로 조회하지 않아도 되는 경우를 의미한다.

Covering Index는 별도의 Index 종류가 아니라 **특정 Query가 Index만으로 처리될 수 있는 상태**를 의미한다.

```sql
CREATE INDEX idx_users_email_name
ON Users(email, name);
```

다음 Query는 필요한 컬럼이 모두 Index에 포함되어 있다.

```sql
SELECT email, name
FROM Users
WHERE email = 'joon@example.com';
```

```text
Secondary Index
      ↓
email 검색
      ↓
name 확인
      ↓
조회 완료
```

반면 Index에 없는 컬럼까지 조회하면 실제 Row에 접근해야 할 수 있다.

```sql
SELECT email, name, age
FROM Users
WHERE email = 'joon@example.com';
```

```text
Secondary Index
      ↓
email 검색
      ↓
PK 확인
      ↓
Clustered Index 탐색
      ↓
age 조회
```

InnoDB의 Secondary Index에는 PK 값도 포함되므로, Index 컬럼과 PK만 조회하는 경우에도 Covering Index가 성립할 수 있다.

### Composite Index vs Covering Index

| 구분 | Composite Index | Covering Index |
|---|---|---|
| 의미 | 여러 컬럼으로 구성된 Index | Query를 Index만으로 처리 |
| 목적 | 검색 범위 축소 | 추가적인 Row 접근 감소 |
| 특징 | 컬럼 순서가 중요 | 조회 컬럼에 따라 달라짐 |

복합 인덱스를 생성했다고 해서 반드시 Covering Index가 되는 것은 아니다.

또한 Covering Index를 위해 불필요하게 많은 컬럼을 추가하면 Index 크기와 쓰기 비용이 증가할 수 있다.

---

## 8. Query Plan

> Query Plan(실행 계획)은 DBMS가 SQL을 어떤 순서와 방법으로 실행할지 결정한 계획이다.

같은 결과를 반환하는 SQL이라도 데이터를 조회하는 방법에 따라 실행 비용이 달라질 수 있다.

### Query Optimizer

> Query Optimizer는 여러 실행 방법의 예상 비용을 비교하여 적절한 실행 계획을 선택하는 DBMS의 구성 요소이다.

```text
SQL
 ↓
Query Optimizer
 ↓
실행 방법 비교
 ├─ Full Table Scan
 ├─ Index Scan
 └─ JOIN 순서 및 방식
 ↓
Execution Plan
 ↓
Query 실행
```

Optimizer는 테이블 크기, Index, 데이터 분포, 통계 정보 등을 바탕으로 실행 비용을 추정한다.

따라서 Index가 존재하더라도 Full Table Scan이 더 효율적이라고 판단하면 Index를 사용하지 않을 수 있다.

### Full Table Scan vs Index Scan

**Full Table Scan**

테이블의 데이터를 전체적으로 탐색하는 방식이다.

```sql
SELECT *
FROM Users
WHERE status = 'ACTIVE';
```

전체 사용자의 대부분이 ACTIVE라면 Full Table Scan이 효율적일 수 있다.

**Index Scan**

Index를 이용해 조건에 맞는 데이터를 탐색하는 방식이다.

```sql
SELECT *
FROM Users
WHERE email = 'joon@example.com';
```

```text
Index 탐색
    ↓
조건에 맞는 Row 확인
    ↓
필요한 데이터 조회
```

검색 결과가 적다면 전체 테이블을 탐색하는 것보다 유리할 수 있다.

### 실행 계획에 영향을 주는 요소

| 요소 | 설명 |
|---|---|
| Index | 사용 가능한 Index의 종류와 구성 |
| Cardinality | 컬럼의 서로 다른 값의 개수 |
| 데이터 분포 | 조건에 해당하는 Row의 비율 |
| 테이블 크기 | 전체 Row 수와 데이터 크기 |
| JOIN | 테이블 결합 순서와 알고리즘 |
| 통계 정보 | Optimizer가 비용을 추정하는 근거 |

실행 계획은 항상 최적이라고 보장할 수 없다.

통계 정보가 실제 데이터 분포와 다르거나 비용 추정이 부정확하면 비효율적인 실행 계획이 선택될 수 있다.

따라서 Query 성능을 개선할 때는 **SQL 자체뿐 아니라 DBMS가 실제로 선택한 실행 계획을 확인하는 과정**이 필요하다.

## 9. EXPLAIN

> EXPLAIN은 DBMS가 SQL을 어떤 방식으로 실행할지 확인하는 명령어이다.

MySQL에서는 EXPLAIN을 통해 테이블 접근 방식, 사용 Index, 예상 조회 Row 수, JOIN 순서 등을 확인할 수 있다.

### EXPLAIN 사용법

```sql
EXPLAIN
SELECT *
FROM Orders
WHERE user_id = 10;
```

MySQL의 주요 출력 항목은 다음과 같다.

| 항목 | 의미 |
|---|---|
| `id` | SELECT 구문의 식별자 |
| `select_type` | SELECT의 유형 |
| `table` | 접근하는 테이블 |
| `type` | 테이블 접근 방식 |
| `possible_keys` | 사용 가능한 Index 후보 |
| `key` | 실제 선택된 Index |
| `rows` | 탐색할 것으로 예상되는 Row 수 |
| `filtered` | 조건을 통과할 것으로 예상되는 비율(%) |
| `Extra` | 추가적인 실행 정보 |

특히 **`type`, `key`, `rows`, `Extra`**를 중심으로 확인한다.

### type: 테이블 접근 방식

`type`은 DBMS가 테이블의 데이터에 접근하는 방법을 나타낸다.

| type | 의미 |
|---|---|
| `const` | PK 또는 Unique Index로 최대 1건 조회 |
| `eq_ref` | JOIN에서 PK 또는 Unique Index를 이용해 최대 1건 조회 |
| `ref` | Non-Unique Index를 이용한 동등 조건 조회 |
| `range` | Index를 이용한 범위 조회 |
| `index` | Index 전체를 스캔 |
| `ALL` | 테이블 전체를 스캔 |

일반적으로 `ALL`보다 `ref`, `range` 등의 접근 방식이 효율적인 경우가 많다.

다만 `ALL`이 항상 나쁜 것은 아니다. 테이블이 작거나 대부분의 Row를 조회한다면 Full Table Scan이 더 효율적일 수 있다.

또한 **`type = index`는 Index를 사용한다는 의미이지만, Index 전체를 스캔할 수 있으므로 반드시 효율적인 것은 아니다.**

### key와 rows

**key**

실제로 선택된 Index를 나타낸다.

```text
possible_keys: idx_orders_user_id
key:           idx_orders_user_id
```

`key = NULL`이면 해당 테이블 접근에 Index가 선택되지 않았다는 의미이다.

**rows**

Optimizer가 탐색할 것으로 예상하는 Row 수이다.

```text
rows: 100000
```

이는 실제로 읽은 Row 수가 아니라 통계 정보를 바탕으로 계산한 **추정치**이다.

따라서 예상 Row 수와 실제 처리 Row 수에는 차이가 발생할 수 있다.

### Extra: 추가 실행 정보

| Extra | 의미 |
|---|---|
| `Using where` | 조건을 이용해 Row를 필터링 |
| `Using index` | Index에 포함된 정보만으로 필요한 데이터를 조회 |
| `Using index condition` | Index Condition Pushdown을 이용해 조건 평가 |
| `Using temporary` | 임시 테이블 사용 |
| `Using filesort` | Index 순서만으로 정렬하지 못해 별도 정렬 수행 |

`Using index`는 Covering Index가 적용된 경우에 나타날 수 있다.

```sql
CREATE INDEX idx_orders_user_status
ON Orders(user_id, status);

SELECT user_id, status
FROM Orders
WHERE user_id = 10;
```

반면 `Using filesort`가 나타났다고 해서 반드시 디스크에서 정렬한다는 의미는 아니다. MySQL이 별도의 정렬 작업을 수행한다는 의미이며, 메모리에서 처리될 수도 있다.

### EXPLAIN ANALYZE

일반적인 EXPLAIN은 실제 Query를 실행하지 않고 예상 실행 계획을 보여준다.

반면 **EXPLAIN ANALYZE는 Query를 실제로 실행하여 측정한 정보**를 제공한다.

```sql
EXPLAIN ANALYZE
SELECT *
FROM Orders
WHERE user_id = 10;
```

주요 확인 항목은 다음과 같다.

- 예상 Row 수와 실제 Row 수
- 실제 실행 시간
- 반복 실행 횟수(`loops`)
- 실행 단계별 처리 방식

```text
EXPLAIN
→ 예상 실행 계획 확인

EXPLAIN ANALYZE
→ 실제 실행 결과를 바탕으로 성능 분석
```

EXPLAIN ANALYZE는 Query를 실제로 실행하므로 운영 환경에서 사용 시 주의해야 한다.

---

## 10. Query 최적화

> Query 최적화는 SQL의 실행 비용을 줄여 데이터 조회 및 처리 성능을 개선하는 과정이다.

단순히 SQL을 짧게 작성하는 것이 아니라 **불필요한 데이터 접근과 연산을 줄이는 것**이 핵심이다.

### 1. 적절한 Index 사용

조회 조건에 맞는 Index를 생성하면 탐색 범위를 줄일 수 있다.

```sql
SELECT *
FROM Orders
WHERE user_id = 10
  AND status = 'COMPLETED';
```

```sql
CREATE INDEX idx_orders_user_status
ON Orders(user_id, status);
```

복합 인덱스는 컬럼 순서에 따라 활용도가 달라지므로 실제 조회 패턴을 기준으로 구성해야 한다.

Index를 과도하게 생성하면 INSERT, UPDATE, DELETE 성능과 저장 공간에 부담을 줄 수 있다.

### 2. 필요한 컬럼만 조회

```sql
-- 불필요한 컬럼까지 조회
SELECT *
FROM Users
WHERE id = 10;
```

```sql
-- 필요한 컬럼만 조회
SELECT id, name
FROM Users
WHERE id = 10;
```

필요한 컬럼만 조회하면 데이터 전송량을 줄일 수 있다.

또한 조회 컬럼이 모두 Index에 포함되어 있다면 Covering Index를 활용할 가능성도 높아진다.

### 3. Index 컬럼에 함수 사용 주의

Index가 생성된 컬럼에 함수를 적용하면 일반적인 Index Range Scan을 활용하기 어려울 수 있다.

```sql
-- Index 활용에 불리할 수 있음
SELECT *
FROM Orders
WHERE YEAR(created_at) = 2026;
```

```sql
-- 범위 조건으로 변경
SELECT *
FROM Orders
WHERE created_at >= '2026-01-01'
  AND created_at < '2027-01-01';
```

두 번째 Query는 `created_at` Index의 정렬 순서를 이용해 범위를 탐색할 수 있다.

이처럼 Index를 효율적으로 활용할 수 있는 검색 조건을 **SARGable(Search Argument Able)** 조건이라고 한다.

단, MySQL의 Functional Index처럼 함수 표현식 자체에 Index를 생성하는 방법도 있다.

### 4. LIKE 검색 조건 확인

```sql
-- 접두사 검색
SELECT *
FROM Users
WHERE name LIKE 'Kim%';
```

B+Tree Index는 정렬된 Key를 활용하므로 접두사 검색에 유리할 수 있다.

반면 다음과 같은 검색은 일반적인 B+Tree Index로 탐색 범위를 좁히기 어렵다.

```sql
-- 앞에 와일드카드 사용
SELECT *
FROM Users
WHERE name LIKE '%Kim';
```

문자열 중간 검색이 빈번하다면 Full-Text Index나 별도의 검색 시스템을 검토할 수 있다.

### 5. N+1 문제 해결

연관된 데이터를 반복적으로 조회하면 Query 실행 횟수가 증가한다.

```text
Users 조회         → 1번
각 User의 Orders 조회 → N번

총 N+1번
```

이를 개선하기 위해 JOIN 또는 IN Query를 활용할 수 있다.

```sql
SELECT *
FROM Orders
WHERE user_id IN (1, 2, 3);
```

다만 JOIN은 1:N 관계에서 부모 Row가 중복될 수 있으므로 반환 데이터 크기도 함께 고려해야 한다.

### 6. Pagination 최적화

**OFFSET 기반 Pagination**

```sql
SELECT *
FROM Orders
ORDER BY id
LIMIT 20 OFFSET 100000;
```

OFFSET이 커지면 앞선 데이터를 건너뛰는 과정에서 많은 Row를 읽어야 할 수 있다.

**Keyset Pagination**

```sql
SELECT *
FROM Orders
WHERE id > 100000
ORDER BY id
LIMIT 20;
```

마지막으로 조회한 `id`를 기준으로 다음 데이터를 조회한다.

```text
OFFSET Pagination
→ 앞선 데이터 건너뛰기

Keyset Pagination
→ 마지막 조회 위치 이후부터 탐색
```

Keyset Pagination은 큰 OFFSET을 피할 수 있어 대량 데이터 조회에 유리하다.

단, 임의 페이지로 바로 이동하기 어렵고 정렬 기준에 맞는 조건과 Index가 필요하다.

### 7. 실행 계획 기반 최적화

Query 최적화는 실제 실행 계획과 성능 측정을 기반으로 진행해야 한다.

```text
느린 Query 발견
      ↓
EXPLAIN 확인
      ↓
비효율적인 접근 방식 분석
      ↓
Index 또는 SQL 수정
      ↓
EXPLAIN ANALYZE 및 실행 시간 측정
      ↓
개선 효과 검증
```

대표적인 확인 사항은 다음과 같다.

| 확인 항목 | 점검 내용 |
|---|---|
| `type` | 불필요한 Full Table Scan 발생 여부 |
| `key` | 의도한 Index 사용 여부 |
| `rows` | 예상 탐색 Row 수가 과도한지 |
| `Using filesort` | 별도 정렬 비용 발생 여부 |
| `Using temporary` | 임시 테이블 사용 여부 |
| 실제 실행 시간 | 변경 전후 성능 차이 |

Query 최적화에서 중요한 것은 **Index 사용 자체가 아니라 실제 처리 비용과 응답 시간을 줄이는 것**이다.

실행 계획은 데이터 크기와 분포에 따라 달라질 수 있으므로, 운영 환경과 유사한 데이터로 개선 효과를 검증해야 한다.
