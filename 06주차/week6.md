# 6주차 - Database 개념 정리

## 목차
- [키워드 연관관계](#키워드-연관관계)
- [RDB/NoSQL](#rdbnosql)
- [PK/FK](#pkfk)
- [정규화](#정규화)
- [JOIN](#join)
- [Index](#index)
- [B+Tree](#btree)
- [Backend 심화 (선택)](#backend-심화-선택)
  - [Composite/Covering Index](#compositecovering-index)
  - [Query Plan](#query-plan)
  - [EXPLAIN](#explain)
  - [Query 최적화](#query-최적화)

---

## 키워드 연관관계

```
RDB/NoSQL
├─ PK/FK → 정규화 → JOIN
├─ Index → B+Tree
├─ Index → Composite/Covering Index
└─ Index → Query Plan → EXPLAIN → Query 최적화
```

| 연결 | 관계 설명 |
|---|---|
| RDB/NoSQL → PK/FK | RDB는 PK/FK로 테이블 간 관계를 맺는 구조 |
| PK/FK → 정규화 → JOIN | 중복 줄이려고 테이블을 쪼개고(정규화), 쪼갠 테이블은 JOIN으로 다시 합쳐서 조회 |
| RDB → Index → B+Tree | 조회 속도를 높이는 Index는 보통 B+Tree로 구현됨 |
| Index → Composite/Covering Index | 여러 컬럼 묶기(Composite), 테이블 조회 자체를 생략(Covering)하는 Index의 변형 |
| Index → Query Plan → EXPLAIN → Query 최적화 | 인덱스를 쓰는지 Query Plan으로 확인(EXPLAIN)하고, 그 결과를 바탕으로 쿼리/인덱스를 개선 |

---

## RDB/NoSQL

> EX) RDB는 칸이 정해진 엑셀 표, NoSQL은 형식이 자유로운 메모장.

- **RDB**: 정해진 스키마(테이블/컬럼)와 관계로 데이터를 저장하고 SQL로 조회 (MySQL, PostgreSQL)
- **NoSQL**: 스키마가 유연하고 종류도 다양 (문서형 MongoDB, Key-Value Redis 등)
- 활용: 정형화된 트랜잭션 데이터는 RDB, 자주 바뀌는 로그/비정형 데이터는 NoSQL이 유리

---

## PK/FK


- **PK(Primary Key)**: 테이블의 각 행을 고유하게 식별하는 키. 중복/NULL 불가
- **FK(Foreign Key)**: 다른 테이블의 PK를 참조해 테이블 간 관계를 맺는 키
- 예시: `orders.user_id`가 `users` 테이블의 PK를 참조하는 FK

---

## 정규화


- 데이터 중복을 줄이고 일관성을 높이기 위해 테이블을 분리하는 과정
- 단점: 테이블이 늘어나 JOIN이 많아지고, 조회 성능이 떨어질 수 있음 (그래서 상황에 따라 일부러 중복을 허용하는 반정규화도 함)

---

## JOIN


| 종류 | 설명 |
|---|---|
| INNER JOIN | 두 테이블에 공통으로 존재하는 데이터만 |
| LEFT JOIN | 왼쪽 테이블은 모두, 오른쪽은 매칭되는 것만 |
| RIGHT JOIN | 오른쪽 테이블은 모두, 왼쪽은 매칭되는 것만 |

---

## Index



- 특정 컬럼의 검색 속도를 높이기 위한 자료구조. 대신 삽입/수정/삭제 시 인덱스도 같이 갱신해야 해 쓰기 성능은 떨어짐
- 예시: `WHERE email = ?` 조회가 잦다면 `email` 컬럼에 인덱스를 거는 것이 좋음

---

## B+Tree



- 대부분의 RDB Index가 쓰는 자료구조. 리프 노드끼리 연결돼 있어 범위 검색(`BETWEEN`, `ORDER BY`)에도 강함
- 탐색/삽입/삭제 모두 `O(log n)`

---

## Backend 심화 (선택)

### Composite/Covering Index


- **Composite Index**: 두 개 이상의 컬럼을 묶어서 만든 인덱스 (컬럼 순서가 중요)
- **Covering Index**: 쿼리에 필요한 컬럼이 모두 인덱스에 포함돼, 테이블을 추가 조회하지 않고 인덱스만으로 결과 반환
- 예시: `(user_id, created_at)` 복합 인덱스는 `WHERE user_id = ? ORDER BY created_at` 쿼리에 효과적

---

### Query Plan


- DB 옵티마이저가 쿼리를 실행하기 위해 세우는 실행 계획
- 같은 결과를 내는 쿼리도 계획에 따라 성능이 크게 달라질 수 있음

---

### EXPLAIN



- 쿼리를 실제로 실행하지 않고 옵티마이저가 세운 Query Plan을 확인하는 명령어
- 예시: `EXPLAIN SELECT * FROM users WHERE email = 'a@a.com';` → 인덱스를 쓰는지, 풀스캔(`type: ALL`)인지 확인 가능

---

### Query 최적화



- Query Plan/EXPLAIN 결과를 바탕으로 인덱스를 추가·수정하거나 쿼리 구조를 바꿔 성능을 개선하는 작업
- 예시: 풀스캔이 나오면 WHERE 조건 컬럼에 인덱스를 추가하거나, `SELECT *` 대신 필요한 컬럼만 조회하도록 변경
