# 10주차 - JPA / 영속성 컨텍스트

## 1. JPA (Java Persistence API)

> JPA는 Java 객체와 관계형 데이터베이스 사이의 데이터를 매핑하고 관리하기 위한 ORM 표준 명세이다.

ORM(Object-Relational Mapping)은 객체와 관계형 데이터베이스의 테이블을 연결하는 기술이다.

JPA는 인터페이스와 동작 방식을 정의한 명세이며, 대표적인 구현체로 **Hibernate**가 있다.

### JPA를 사용하는 이유

JDBC를 직접 사용하면 SQL 작성, 실행, 결과 매핑 등을 개발자가 처리해야 한다.

JPA는 이러한 작업의 상당 부분을 자동화한다.

```java
@Entity
public class Member {

    @Id
    @GeneratedValue(strategy = GenerationType.IDENTITY)
    private Long id;

    private String name;
}
```

```java
Member member = new Member("Kim");

entityManager.persist(member);
```

JPA는 Entity의 매핑 정보를 바탕으로 INSERT SQL을 생성하고 실행한다.

### JPA의 특징

- **객체 중심 개발**: SQL보다 Entity를 중심으로 데이터 관리
- **영속성 컨텍스트**: Entity의 상태와 변경 사항 관리
- **Dirty Checking**: Entity 변경 사항 자동 감지
- **Lazy Loading**: 연관 Entity를 필요한 시점에 조회
- **JPQL**: Entity를 대상으로 Query 작성

다만 JPA를 사용하더라도 실제 SQL의 실행 방식과 데이터베이스 성능을 이해해야 한다.

---

## 2. 영속성 컨텍스트 (Persistence Context)

> 영속성 컨텍스트는 Entity를 관리하는 논리적인 저장 공간이다.

JPA는 영속성 컨텍스트를 통해 Entity의 상태를 추적하고 데이터베이스와 동기화한다.

`EntityManager`는 영속성 컨텍스트에 접근하여 Entity를 관리하는 인터페이스이다.

### 영속성 컨텍스트의 구조

```text
Application
     ↓
EntityManager
     ↓
Persistence Context
  ├─ 1차 Cache
  ├─ Entity 상태 관리
  ├─ 변경 감지
  └─ 쓰기 지연 작업 관리
     ↓
Database
```

영속성 컨텍스트는 다음과 같은 기능을 제공한다.

| 기능 | 설명 |
|---|---|
| 1차 Cache | Entity를 식별자 기준으로 관리 |
| 동일성 보장 | 같은 영속성 컨텍스트에서 동일 Entity 인스턴스 반환 |
| Dirty Checking | Entity의 변경 사항 감지 |
| Write-Behind | SQL 실행을 Flush 시점까지 지연 |
| Lazy Loading | 연관 Entity를 필요한 시점에 조회 |

Spring Data JPA에서는 일반적으로 트랜잭션 범위에서 영속성 컨텍스트를 사용한다.

---

## 3. Entity Lifecycle

> Entity Lifecycle은 Entity가 영속성 컨텍스트와 어떤 관계에 있는지를 나타내는 상태이다.

JPA Entity는 크게 네 가지 상태로 구분된다.

### Entity의 4가지 상태

| 상태 | 설명 |
|---|---|
| 비영속(Transient) | 영속성 컨텍스트에서 관리하지 않는 상태 |
| 영속(Managed) | 영속성 컨텍스트에서 관리하는 상태 |
| 준영속(Detached) | 관리되다가 영속성 컨텍스트에서 분리된 상태 |
| 삭제(Removed) | 삭제 대상으로 등록된 상태 |

### 상태 전이

```text
비영속(Transient)
       │
       │ persist()
       ↓
영속(Managed)
   │       │
   │       └─ remove() → 삭제(Removed)
   │
   └─ detach() / clear()
              ↓
        준영속(Detached)
              │
              │ merge()
              ↓
          영속(Managed)
```

**1. 비영속**

```java
Member member = new Member("Kim");
```

객체만 생성한 상태이며 영속성 컨텍스트에서 관리하지 않는다.

**2. 영속**

```java
entityManager.persist(member);
```

Entity가 영속성 컨텍스트의 관리 대상이 된다.

단, `persist()`를 호출했다고 해서 반드시 즉시 INSERT SQL이 실행되는 것은 아니다.

**3. 준영속**

```java
entityManager.detach(member);
```

Entity가 영속성 컨텍스트에서 분리된다.

준영속 상태에서는 변경 감지가 동작하지 않는다.

**4. 삭제**

```java
entityManager.remove(member);
```

Entity를 삭제 대상으로 등록한다. 실제 DELETE SQL은 일반적으로 Flush 시점에 실행된다.

### merge()의 특징

```java
Member managed = entityManager.merge(detachedMember);
```

`merge()`는 준영속 Entity 자체를 다시 영속 상태로 만드는 것이 아니라, 해당 상태를 영속 Entity에 복사하고 **영속 Entity를 반환**한다.

따라서 반환된 객체를 사용하는 것이 중요하다.

---

## 4. 1차 Cache

> 1차 Cache는 영속성 컨텍스트 내부에서 Entity를 식별자 기준으로 저장하고 관리하는 공간이다.

Entity를 조회할 때 먼저 1차 Cache를 확인한다.

### 조회 과정

```java
Member member1 = entityManager.find(Member.class, 1L);
Member member2 = entityManager.find(Member.class, 1L);
```

```text
첫 번째 조회
    ↓
1차 Cache 확인
    ↓
데이터 없음
    ↓
SELECT 실행
    ↓
Entity 저장


두 번째 조회
    ↓
1차 Cache 확인
    ↓
기존 Entity 반환
```

동일한 영속성 컨텍스트에서 같은 식별자의 Entity를 조회하면 동일한 객체를 반환한다.

```java
member1 == member2; // true
```

이를 **영속 Entity의 동일성 보장**이라고 한다.

### 1차 Cache의 특징

- 영속성 컨텍스트 단위로 관리된다.
- Entity 타입과 식별자를 기준으로 Entity를 구분한다.
- 동일한 Entity의 중복 조회를 줄일 수 있다.
- 서로 다른 영속성 컨텍스트 사이에서는 기본적으로 공유되지 않는다.

단, JPQL 조회는 일반적으로 SQL을 실행한다. 조회 결과를 Entity로 구성할 때 영속성 컨텍스트에 이미 존재하는 Entity가 있다면 해당 인스턴스를 활용한다.

따라서 **1차 Cache가 모든 SELECT SQL을 생략하는 것은 아니다.**

---

## 5. Dirty Checking (변경 감지)

> Dirty Checking은 영속 상태의 Entity에서 발생한 변경 사항을 감지하여 UPDATE SQL을 실행하는 기능이다.

개발자가 직접 UPDATE Query를 작성하지 않아도 Entity의 변경 사항을 데이터베이스에 반영할 수 있다.

### 동작 과정

```java
@Transactional
public void updateMember(Long memberId) {
    Member member = memberRepository.findById(memberId)
        .orElseThrow();

    member.changeName("Lee");
}
```

```text
Entity 조회
    ↓
영속성 컨텍스트에서 관리
    ↓
Entity 필드 변경
    ↓
Flush 시점
    ↓
변경 사항 감지
    ↓
UPDATE SQL 실행
```

JPA 구현체는 영속 Entity의 기존 상태와 현재 상태를 비교하여 변경 여부를 판단한다.

Hibernate는 일반적으로 조회 시점의 상태를 Snapshot으로 관리하며, Bytecode Enhancement를 이용한 변경 추적 방식도 지원한다.

### Dirty Checking의 조건

Dirty Checking이 정상적으로 동작하려면 다음 조건을 고려해야 한다.

- Entity가 영속 상태여야 한다.
- 변경 사항이 Flush 시점에 감지되어야 한다.
- 데이터베이스에 반영하려면 적절한 트랜잭션 처리가 필요하다.

준영속 Entity를 변경해도 자동으로 UPDATE되지 않는다.

또한 `@Transactional(readOnly = true)` 환경에서는 Hibernate의 Flush 설정 등에 의해 변경 감지가 제한될 수 있으므로 조회 전용 트랜잭션에서 Entity를 수정하는 방식은 피해야 한다.

---

## 6. Flush

> Flush는 영속성 컨텍스트에서 관리하는 변경 사항을 데이터베이스에 반영하기 위해 SQL을 실행하는 과정이다.

Flush는 영속성 컨텍스트를 비우는 작업이 아니다.

### Flush와 Commit의 차이

```text
Entity 변경
    ↓
Flush
    ↓
SQL 실행
    ↓
Commit
    ↓
트랜잭션 확정
```

| 구분 | Flush | Commit |
|---|---|---|
| 역할 | 변경 사항을 DB에 동기화 | 트랜잭션 확정 |
| SQL 실행 | 발생할 수 있음 | 필요 시 Flush 후 Commit |
| 변경 확정 | X | O |
| Rollback | 가능 | Commit 이후에는 해당 트랜잭션 Rollback 불가 |

Flush로 SQL이 실행되더라도 트랜잭션이 Commit되지 않았다면 Rollback할 수 있다.

### Flush 발생 시점

기본 Flush Mode인 `AUTO`에서는 대표적으로 다음 상황에서 Flush가 발생할 수 있다.

1. 트랜잭션 Commit 직전
2. 변경 사항과 관련된 JPQL 실행 전
3. `entityManager.flush()` 명시적 호출

```java
entityManager.persist(member);
entityManager.flush();
```

단, `GenerationType.IDENTITY`처럼 INSERT를 통해 식별자를 확보해야 하는 전략에서는 `persist()` 시점에 INSERT가 실행될 수 있다.

### 쓰기 지연 (Write-Behind)

JPA는 변경 작업을 모아 두었다가 Flush 시점에 SQL을 실행할 수 있다.

```java
entityManager.persist(member1);
entityManager.persist(member2);
```

```text
persist(member1)
persist(member2)
       ↓
쓰기 지연 작업 등록
       ↓
Flush
       ↓
INSERT SQL 실행
```

쓰기 지연은 SQL 실행 시점을 조절하는 기능이며, 여러 INSERT를 하나의 SQL로 자동 합친다는 의미는 아니다.

Hibernate JDBC Batch 설정 등을 이용하면 여러 SQL을 묶어 데이터베이스에 전달하는 최적화가 가능하다.

---

## 7. Lazy / Eager Loading

> Fetch 전략은 연관된 Entity를 언제 조회할지 결정하는 방식이다.

JPA에서는 Lazy Loading과 Eager Loading을 제공한다.

### Lazy Loading (지연 로딩)

연관 Entity가 실제로 필요한 시점에 조회한다.

```java
@ManyToOne(fetch = FetchType.LAZY)
private Member member;
```

```java
Order order = orderRepository.findById(1L)
    .orElseThrow();

// 연관 Entity 접근
String name = order.getMember().getName();
```

```text
Order 조회
    ↓
Order SELECT
    ↓
Member 접근
    ↓
Member SELECT
```

Hibernate에서는 일반적으로 Proxy 등을 이용해 연관 Entity의 조회를 지연시킨다.

### Eager Loading (즉시 로딩)

연관 Entity를 함께 로딩하는 방식이다.

```java
@ManyToOne(fetch = FetchType.EAGER)
private Member member;
```

즉시 로딩이라고 해서 반드시 JOIN 한 번으로 조회하는 것은 아니다.

Query 방식과 구현체에 따라 추가 SELECT가 발생할 수 있다.

### Lazy vs Eager

| 구분 | Lazy Loading | Eager Loading |
|---|---|---|
| 조회 시점 | 연관 데이터 접근 시 | 초기 조회 과정 |
| 장점 | 불필요한 조회 감소 | 연관 데이터 즉시 사용 가능 |
| 단점 | 추가 Query 발생 가능 | 불필요한 데이터 조회 가능 |
| 주의점 | N+1, LazyInitializationException | N+1, 과도한 조회 |

JPA의 기본 Fetch 전략은 다음과 같다.

| 연관관계 | 기본 Fetch |
|---|---|
| `@ManyToOne` | EAGER |
| `@OneToOne` | EAGER |
| `@OneToMany` | LAZY |
| `@ManyToMany` | LAZY |

실무에서는 불필요한 연관 데이터 조회를 줄이기 위해 **연관관계를 기본적으로 LAZY로 설정하고, 필요한 조회에서 Fetch Join 등을 사용하는 방식**이 널리 활용된다.

### LazyInitializationException

Lazy Loading이 필요한 시점에 영속성 컨텍스트가 이미 종료되어 연관 Entity를 조회할 수 없으면 발생할 수 있다.

```text
Transaction 종료
      ↓
영속성 컨텍스트 종료
      ↓
지연 로딩 대상 접근
      ↓
LazyInitializationException
```

이를 방지하기 위해 트랜잭션 내에서 필요한 데이터를 조회하거나 DTO Projection, Fetch Join 등을 활용할 수 있다.

---

## 8. N+1 문제

> N+1 문제는 최초 Query 1번으로 N개의 Entity를 조회한 뒤, 각 Entity의 연관 데이터를 조회하기 위해 추가 Query가 N번 발생하는 문제이다.

### N+1 발생 예시

사용자 100명을 조회한 뒤 각 사용자의 주문 목록에 접근하는 경우이다.

```java
List<Member> members = memberRepository.findAll();

for (Member member : members) {
    member.getOrders().size();
}
```

```text
Member 전체 조회
       ↓
SELECT 1번
       ↓
Member 100명
       ↓
각 Member의 Orders 조회
       ↓
SELECT 최대 100번

총 101번 Query
```

Lazy Loading은 연관 데이터 조회를 지연시킬 뿐, 여러 Entity의 연관 데이터를 자동으로 한 번에 조회하지 않는다.

또한 Eager Loading에서도 JPQL 실행 방식에 따라 N+1 문제가 발생할 수 있다.

### N+1 해결 방법

**1. Fetch Join**

연관 Entity를 JOIN하여 함께 조회한다.

```java
@Query("""
    SELECT DISTINCT m
    FROM Member m
    LEFT JOIN FETCH m.orders
""")
List<Member> findAllWithOrders();
```

연관 데이터를 한 번의 SQL로 조회할 수 있다.

다만 1:N 관계에서는 JOIN 결과의 Row 수가 증가하며, 컬렉션 Fetch Join과 Pagination을 함께 사용할 때 주의해야 한다.

**2. EntityGraph**

조회할 연관관계를 선언적으로 지정할 수 있다.

```java
@EntityGraph(attributePaths = {"orders"})
List<Member> findAll();
```

필요한 연관 Entity를 함께 로딩하도록 지정하는 방식이다.

**3. Batch Fetching**

연관 Entity를 개별적으로 조회하는 대신 여러 식별자를 묶어 조회한다.

```java
@BatchSize(size = 100)
@OneToMany(mappedBy = "member")
private List<Order> orders;
```

```sql
SELECT *
FROM Orders
WHERE member_id IN (1, 2, 3, ...);
```

Hibernate의 `hibernate.default_batch_fetch_size` 설정을 이용해 전역적으로 Batch Fetching을 적용할 수도 있다.

### 해결 방법 비교

| 방식 | 특징 | 주의점 |
|---|---|---|
| Fetch Join | 연관 데이터를 JOIN으로 조회 | 1:N에서 Row 증가, Pagination |
| EntityGraph | 연관 데이터 로딩 범위 지정 | 조회 대상이 많으면 비용 증가 |
| Batch Fetching | 여러 연관 데이터를 IN Query로 조회 | 추가 Query는 발생할 수 있음 |
| DTO Projection | 필요한 데이터만 직접 조회 | Entity 관리 기능과 구분 필요 |

N+1 문제는 단순히 Query 횟수만 줄이는 것이 아니라 **조회 데이터 크기, JOIN 결과, Pagination, 메모리 사용량**을 함께 고려해야 한다.

특히 대량의 1:N 데이터를 조회할 때는 Fetch Join 한 번보다 Batch Fetching이 효율적일 수 있다.
