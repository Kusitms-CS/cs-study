# 6주차 참고 자료

## 입문

- [NoSQL이란? (AWS)](https://aws.amazon.com/ko/nosql/)
  - NoSQL의 종류(Key-Value, Document, Wide-Column, Graph)와 관계형 DB와의 차이를 용도 중심으로 정리
- [What Is NoSQL? NoSQL Databases Explained (MongoDB)](https://www.mongodb.com/resources/basics/databases/nosql-explained)
  - NoSQL이 나온 배경, 네 가지 유형, RDBMS와의 비교와 흔한 오해
- [DBMS 정규화: 1NF, 2NF, 3NF 데이터베이스 예 (guru99)](https://www.guru99.com/ko/database-normalization.html)
  - 영화 대여 예시 테이블 하나를 1NF → 2NF → 3NF로 차례로 분해하는 과정
- [[DataBase] 인덱스와 B-Tree (MySQL)](https://velog.io/@yjl8628/DataBase-인덱스와-B-Tree-MySQL)
  - 인덱스가 왜 B-Tree/B+Tree를 쓰는지, 클러스터링 인덱스와 세컨더리 인덱스의 리프 노드 차이
- [Real MySQL 5장 인덱스 정리](https://velog.io/@dlsghl92/인덱스)
  - Real MySQL 8.0의 인덱스 장을 요약한 노트. B-Tree 구조, 인덱스 스캔 방식, 다중 컬럼 인덱스
- [2. MySQL 실행 계획](https://velog.io/@kimkevin90/2.-MySQL-실행-계획)
  - EXPLAIN 출력의 각 컬럼(type, key, rows, Extra)이 무슨 의미인지 예시와 함께 정리

## 기본기

- Real MySQL 8.0 (백은빈, 이성욱 저)
  - 8장 인덱스, 9장 옵티마이저와 힌트, 10장 실행 계획. 이 주차 Backend 심화 키워드가 전부 들어있는 책
- [MySQL 8.0 Reference Manual - EXPLAIN Output Format](https://dev.mysql.com/doc/refman/8.0/en/explain-output.html)
  - EXPLAIN 각 컬럼과 type / Extra 값의 공식 설명
- [MySQL 8.0 Reference Manual - Clustered and Secondary Indexes](https://dev.mysql.com/doc/refman/8.0/en/innodb-index-types.html)
  - InnoDB에서 PK가 클러스터링 인덱스가 되는 규칙과 세컨더리 인덱스가 PK를 담는 구조
- [MySQL 8.0 Reference Manual - Multiple-Column Indexes](https://dev.mysql.com/doc/refman/8.0/en/multiple-column-indexes.html)
  - 복합 인덱스의 왼쪽 접두사 규칙 공식 설명
- [Use The Index, Luke](https://use-the-index-luke.com/)
  - 개발자 관점의 SQL 인덱싱 무료 공개 도서. B-Tree 구조부터 WHERE / JOIN / ORDER BY별 인덱스 설계까지
- [CMU 15-445 Database Systems](https://15445.courses.cs.cmu.edu/)
  - 카네기멜런 DB 강의. 저장 구조, B+Tree, 조인 알고리즘, 쿼리 최적화를 내부 구현 관점에서 다룸

## 정규화

- [A Simple Guide to Five Normal Forms in Relational Database Theory (William Kent, 1983)](https://www.bkent.net/Doc/simple5.htm)
  - 관계 이론 없이 1NF~5NF를 설명한 고전. "키에, 전체 키에, 오직 키에만"의 출처격 문서

## JOIN

- [MySQL 8.0 Reference Manual - Nested-Loop Join Algorithms](https://dev.mysql.com/doc/refman/8.0/en/nested-loop-joins.html)
  - NLJ와 Block NLJ 동작 방식
- [MySQL 8.0 Reference Manual - Hash Join Optimization](https://dev.mysql.com/doc/refman/8.0/en/hash-joins.html)
  - 8.0.18부터 도입된 해시 조인의 적용 조건과 8.0.20 이후 Block NLJ를 대체한 내용
- [A Beginner's Guide to the True Order of SQL Operations (jOOQ)](https://blog.jooq.org/a-beginners-guide-to-the-true-order-of-sql-operations/)
  - FROM → WHERE → GROUP BY → HAVING → SELECT → ORDER BY 논리적 실행 순서와 그로 인한 제약

## Index / B+Tree

- [B-trees and database indexes (PlanetScale)](https://planetscale.com/blog/btrees-and-database-indexes)
  - 인터랙티브 그림으로 B-Tree / B+Tree 삽입 과정을 보여주고, 순차 PK와 랜덤(UUID) PK가 페이지 분할에 미치는 차이를 설명

## Composite / Covering Index / Query 최적화

- [1. 커버링 인덱스 (기본 지식 / WHERE / GROUP BY)](https://jojoldu.tistory.com/476)
  - 커버링 인덱스의 개념과 WHERE, GROUP BY에서 인덱스가 적용되는 조건을 실측으로 비교
- [2. 커버링 인덱스 (WHERE + ORDER BY / GROUP BY + ORDER BY)](https://jojoldu.tistory.com/481)
  - ORDER BY가 인덱스를 타려면 앞 컬럼이 등치 조건이어야 하는 이유와 복합 인덱스 순서 실험
- [2. 페이징 성능 개선하기 - 커버링 인덱스 사용하기](https://jojoldu.tistory.com/529)
  - 큰 OFFSET 페이징을 커버링 인덱스 서브쿼리로 바꿔 26초 → 0.3초로 줄인 사례. Querydsl / JdbcTemplate 구현 포함
- [PostgreSQL - Using EXPLAIN](https://www.postgresql.org/docs/current/using-explain.html)
  - MySQL과 비교용. PostgreSQL 실행 계획 트리 읽는 법과 EXPLAIN ANALYZE
