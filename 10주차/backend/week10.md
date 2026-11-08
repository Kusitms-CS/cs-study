# JPA / 영속성 컨텍스트: 객체를 바꾸면 언제 SQL이 실행될까?

JPA는 자바 객체와 관계형 DB를 연결하는 ORM 표준이고, Hibernate는 그 구현체 중 하나다. 이번 주는 엔티티를 조회한 뒤 필드만 바꿨는데 UPDATE가 실행되는 이유와, 연관관계를 읽다가 쿼리가 늘어나는 이유를 영속성 컨텍스트에서부터 따라가 본다.

필수 키워드: **Entity Lifecycle, 1차 Cache, Dirty Checking, Flush, Lazy/Eager, N+1**.

예제는 `jakarta.persistence` API를 사용하는 코드 조각이다. `Member`에는 `id`, `name`, `team`이, `Team`에는 `id`, `name`, `members`가 매핑되어 있다고 가정한다. 별도 설명이 없다면 활성 트랜잭션과 해당 트랜잭션에 참여하는 EntityManager가 있으며, 조회할 데이터가 존재하는 상황이다. SQL 실행 횟수 예시는 2차 캐시·쿼리 캐시·배치 조회 등을 사용하지 않는 조건이다.

## 1. 영속성 컨텍스트: 엔티티를 관리하는 범위

영속성 컨텍스트(Persistence Context)는 엔티티의 상태와 동일성을 관리하는 공간이다. EntityManager는 이 컨텍스트에 접근해 엔티티를 저장·조회·삭제하는 API다. 같은 컨텍스트 안에서는 **같은 엔티티 타입과 식별자에 대해 하나의 관리 인스턴스**를 유지한다.

```text
애플리케이션 → EntityManager → 영속성 컨텍스트 → DB
                               ├ 엔티티 동일성 관리
                               ├ 변경 감지
                               └ 변경 내용 동기화
```

영속성 컨텍스트의 수명이 항상 트랜잭션 하나와 같은 것은 아니다. Spring에서 트랜잭션에 맞춰 EntityManager를 사용하는 경우가 흔하지만, 직접 생성한 EntityManager나 확장 컨텍스트는 여러 트랜잭션에 걸쳐 유지될 수 있다. OSIV(Open EntityManager in View)를 사용하면 웹 요청의 뷰 처리까지 열어두기도 한다.

EntityManagerFactory는 공유할 수 있지만 실제 EntityManager는 여러 스레드가 동시에 사용하도록 설계되지 않았다. Spring의 `@PersistenceContext`로 주입받는 기본 프록시는 현재 트랜잭션 등에 맞는 EntityManager로 호출을 전달한다. 프록시를 공유하는 것과 실제 컨텍스트를 여러 스레드에서 공유하는 것은 구분해야 한다.

참고: [Jakarta Persistence — Entity Operations](https://jakarta.ee/specifications/persistence/3.2/jakarta-persistence-spec-3.2.html#a1060), [Spring Framework — JPA](https://docs.spring.io/spring-framework/reference/data-access/orm/jpa.html), [Spring Framework — OpenEntityManagerInViewFilter](https://docs.spring.io/spring-framework/docs/current/javadoc-api/org/springframework/orm/jpa/support/OpenEntityManagerInViewFilter.html).

---

## 2. Entity Lifecycle: DB에 있는가보다 관리 중인가가 중요하다

| 상태 | 의미 | 대표적인 진입 경로 |
| --- | --- | --- |
| 비영속(new/transient) | 아직 영속성 컨텍스트가 관리하지 않는 새 객체 | `new Member(...)` |
| 영속(managed) | 현재 영속성 컨텍스트가 관리하는 객체 | `persist()`, 엔티티를 반환하는 `find()`·JPQL 조회 |
| 준영속(detached) | 관리되었다가 컨텍스트에서 분리된 객체 | `detach()`, `clear()`, EntityManager 종료 |
| 삭제(removed) | 관리 대상에서 삭제하도록 표시된 객체 | 영속 엔티티에 `remove()` |

```java
Member member = new Member("재현"); // 비영속
em.persist(member);                // 영속
member.setName("변경한 이름");       // 변경 감지 대상
em.flush();                        // DB와 동기화, 커밋은 아님

em.detach(member);                 // 이 객체만 준영속으로 전환
member.setName("분리 후 변경");       // 이 변경은 자동 반영되지 않음
```

`persist()`는 엔티티를 관리 대상으로 만드는 연산이다. INSERT 실행 시점과 항상 같지는 않다. 특히 Hibernate의 일반적인 활성 트랜잭션에서 `IDENTITY` 전략을 사용하면 생성된 식별자를 얻기 위해 일찍 INSERT할 수 있다.

삭제하려면 영속 객체를 넘긴다. 준영속 객체에 바로 `em.remove(member)`를 호출하는 것은 올바른 삭제 흐름이 아니며, JPA에서는 예외 대상이다.

```java
Member managed = em.find(Member.class, memberId);
em.remove(managed); // 삭제 상태 → flush 과정에서 DELETE 반영
```

`merge()`도 원본 객체를 그대로 다시 관리하는 연산으로 생각하면 헷갈린다. 준영속 객체의 상태를 관리 인스턴스에 복사하고 **그 관리 인스턴스를 반환**한다. 원본은 준영속 상태로 남는다.

```java
Member managed = em.merge(detachedMember);
managed.setName("관리 인스턴스 변경"); // 이후 변경은 반환된 객체에 적용
```

Spring Data JPA의 `save()`는 엔티티의 신규 여부 판단에 따라 `persist()` 또는 `merge()`를 호출한다. 따라서 `save()`가 항상 INSERT를 하거나, 호출 즉시 SQL을 전송한다고 볼 수는 없다.

참고: [Jakarta Persistence — EntityManager](https://jakarta.ee/specifications/persistence/3.2/apidocs/jakarta.persistence/jakarta/persistence/EntityManager.html), [Spring Data JPA — Persisting Entities](https://docs.spring.io/spring-data/jpa/reference/jpa/entity-persistence.html).

---

## 3. 1차 Cache: 같은 객체를 다시 쓰는 기준

1차 캐시는 영속성 컨텍스트 내부에서 관리 엔티티를 보관하는 기능이다. 단순히 ID 숫자만이 아니라 엔티티 타입과 식별자를 함께 기준으로 삼는다.

```java
// 해당 Member가 컨텍스트에 아직 없는 상태
Member first = em.find(Member.class, 1L);  // DB 조회
Member second = em.find(Member.class, 1L); // 관리 중인 객체 반환

System.out.println(first == second); // true
```

이 동일성 보장은 현재 컨텍스트 안의 이야기다. `clear()` 후 다시 조회하거나 다른 컨텍스트에서 조회한 객체까지 `==`가 같다는 뜻은 아니다. 또한 외부 트랜잭션이 DB를 변경해도 이미 관리 중인 객체가 매번 자동으로 갱신되는 것은 아니다.

### JPQL 조회 결과도 전부 캐시에서 꺼낼까?

```java
em.find(Member.class, 1L);

List<Member> members = em.createQuery(
    "select m from Member m where m.name = :name", Member.class)
    .setParameter("name", "재현")
    .getResultList();
```

위 JPQL은 조건에 맞는 행을 찾기 위해 DB를 조회한다. 결과를 엔티티로 구성할 때 이미 관리 중인 같은 식별자의 객체가 있다면 그 인스턴스를 사용한다. **엔티티를 보관하는 1차 캐시와 쿼리 결과를 저장하는 캐시는 다르다.**

참고: [Hibernate — Persistence Contexts](https://docs.hibernate.org/orm/7.1/introduction/html_single/#persistence-contexts).

---

## 4. Dirty Checking: 관리 중인 객체의 변경을 추적한다

일반적인 Hibernate의 변경 감지는 엔티티의 로딩 상태를 스냅샷으로 보관하고, flush 때 현재 상태와 비교해 변경이 있으면 UPDATE에 반영하는 방식이다. 바이트코드 향상을 통한 변경 추적 등 구현 방식은 달라질 수 있다.

```java
Member member = em.find(Member.class, 1L);
member.setName("새 이름");
// 별도의 update() 호출 없이 flush 시 변경 반영
```

대상은 현재 컨텍스트가 관리하는 엔티티다. 새 객체나 준영속 객체의 필드만 바꿔서는 자동 UPDATE가 되지 않는다. 실제 DB 쓰기에는 트랜잭션도 필요하다.

### Spring 서비스에서는 어떻게 연결될까?

```java
import jakarta.persistence.EntityManager;
import jakarta.persistence.PersistenceContext;
import org.springframework.stereotype.Service;
import org.springframework.transaction.annotation.Transactional;

@Service
public class MemberService {
    @PersistenceContext
    private EntityManager em;

    @Transactional
    public void rename(Long memberId, String newName) {
        Member member = em.find(Member.class, memberId);
        if (member == null) {
            throw new IllegalArgumentException("회원을 찾을 수 없습니다.");
        }
        member.setName(newName);
    }
}
```

트랜잭션 프록시를 거치는 외부 호출을 전제로, 조회한 엔티티를 바꾸고 트랜잭션을 정상 커밋하면 변경이 반영된다. 관리 엔티티의 필드 변경을 DB에 반영하는 데 JPA 차원의 추가 `save()` 호출이 필요한 것은 아니다. 지난주에 정리한 것처럼 같은 객체 안의 자기 호출로는 `@Transactional` 경계가 새로 적용되지 않는다.

`@Transactional(readOnly = true)`는 쓰기를 확실히 차단하는 장치로 사용하면 안 된다. Spring과 Hibernate의 연동에서는 flush·변경 감지 최적화가 적용될 수 있으므로, 수정 작업은 쓰기 트랜잭션에서 수행한다.

참고: [Hibernate — Interacting with the Database](https://docs.hibernate.org/orm/7.1/introduction/html_single/#interacting), [Spring Data JPA — Transactionality](https://docs.spring.io/spring-data/jpa/reference/jpa/transactions.html).

---

## 5. Flush: SQL 실행과 커밋을 구분하기

flush는 관리 중인 엔티티의 변경을 DB와 동기화하는 과정이다. 일반적인 Hibernate 처리에서는 변경을 감지하고 필요한 INSERT·UPDATE·DELETE를 실행한다. 쓰기 지연은 실행을 늦추는 것이고, JDBC 배치는 여러 실행을 묶는 별도 최적화다.

| 연산 | 역할 | 주의할 점 |
| --- | --- | --- |
| `flush()` | 변경을 DB에 동기화 | 커밋하거나 컨텍스트를 비우지 않음 |
| `clear()` | 관리 중인 엔티티를 모두 분리 | 아직 동기화하지 않은 변경은 자동 반영되지 않음 |
| commit | 트랜잭션을 확정 | 일반적인 쓰기 트랜잭션에서는 확정 전에 flush 수행 |

### 언제 자동으로 동기화할까?

기본 `FlushModeType.AUTO`에서는 커밋 시점뿐 아니라 **쿼리 결과에 영향을 주는 변경이 있을 때 쿼리 실행 전에도** 동기화할 수 있다. Hibernate는 JPQL/HQL이 접근하는 테이블과 대기 중인 변경이 겹치는지 등을 판단한다. 모든 JPQL 앞에서 반드시 전체 flush를 한다는 뜻은 아니다.

`COMMIT` 모드에서도 커밋 시 동기화하지만, 그 전에 실행한 쿼리 결과에 컨텍스트의 변경이 어떻게 반영될지는 JPA에서 보장하지 않는다. 애플리케이션이 필요하면 `em.flush()`를 명시할 수 있다.

```java
Member member = em.find(Member.class, 1L);
member.setName("새 이름");

em.flush(); // UPDATE 실행 가능, 아직 트랜잭션은 진행 중
em.clear(); // 이제 member는 준영속
```

이후 트랜잭션이 롤백되면 일반적인 트랜잭션 테이블의 UPDATE도 롤백된다. 다만 롤백이 자바 객체의 필드까지 이전 값으로 돌려주는 것은 아니다.

참고: [Jakarta Persistence — FlushModeType](https://jakarta.ee/specifications/persistence/3.2/apidocs/jakarta.persistence/jakarta/persistence/FlushModeType.html), [Hibernate — Flushing](https://docs.hibernate.org/orm/7.1/userguide/html_single/#flushing).

---

## 6. Lazy / Eager: 로딩 시점과 SQL 모양은 다르다

| 전략 | 의미 | 반드시 보장하지는 않는 것 |
| --- | --- | --- |
| LAZY | 연관 데이터의 사용 시점까지 로딩을 미루도록 요청 | 실제 사용 전에는 절대로 조회하지 않는다는 보장 |
| EAGER | 엔티티를 로딩할 때 연관 데이터도 로딩하도록 요구 | SQL JOIN 한 번으로 가져온다는 보장 |

JPA에서 LAZY는 구현체에 대한 힌트다. Hibernate는 프록시나 영속 컬렉션 등으로 지연 로딩을 구현하며, 이미 로딩된 데이터라면 다시 SQL이 나가지 않을 수도 있다.

```java
// Member 엔티티의 연관관계 매핑 일부
@ManyToOne(fetch = FetchType.LAZY)
@JoinColumn(name = "team_id")
private Team team;
```

```java
Member member = em.find(Member.class, 1L);
Team team = member.getTeam();
String teamName = team.getName(); // 미초기화 프록시라면 여기서 추가 조회 가능
```

일반적인 연관관계 기본값은 `@ManyToOne`, `@OneToOne`이 EAGER, `@OneToMany`, `@ManyToMany`가 LAZY다. 필요한 조회 범위를 명시하기 위해 연관관계를 LAZY로 두고, 화면·API가 필요한 데이터를 쿼리별로 가져오는 방식을 우선 검토할 수 있다.

Hibernate에서 미초기화 연관관계를 세션과 연결되지 않은 상태에서 읽으면 `LazyInitializationException`이 발생할 수 있다. 준영속 객체라도 이미 초기화한 값은 읽을 수 있다. 예외를 피하려고 전체 매핑을 EAGER로 바꾸기보다는, 서비스 안에서 필요한 연관관계를 조회하고 응답 DTO로 변환하는 식으로 조회 범위를 정한다.

참고: [Jakarta Persistence — FetchType](https://jakarta.ee/specifications/persistence/3.2/apidocs/jakarta.persistence/jakarta/persistence/FetchType.html), [ManyToOne](https://jakarta.ee/specifications/persistence/3.2/apidocs/jakarta.persistence/jakarta/persistence/ManyToOne.html), [OneToMany](https://jakarta.ee/specifications/persistence/3.2/apidocs/jakarta.persistence/jakarta/persistence/OneToMany.html), [Hibernate — LazyInitializationException](https://docs.hibernate.org/orm/7.1/javadocs/org/hibernate/LazyInitializationException.html).

---

## 7. N+1: 목록 조회 뒤에 추가 조회가 반복된다

팀 N개를 가져온 뒤 각 팀의 회원 목록을 사용한다고 하자. `Team.members`는 LAZY이고, 아직 초기화되지 않은 상태다.

```java
List<Team> teams = em.createQuery(
    "select t from Team t", Team.class).getResultList(); // 목록 조회 1회

for (Team team : teams) {
    System.out.println(team.getMembers().size()); // 각 팀의 회원 조회 N회
}
```

이 조건에서는 총 1 + N회의 쿼리가 실행된다. 실제 횟수는 캐시, 배치 설정, 이미 로딩된 연관관계 등에 따라 달라진다. EAGER도 별도 SELECT로 연관관계를 채울 수 있으므로 N+1의 해결책이 되지는 않는다.

### 조회 목적에 맞춰 데이터를 가져오기

```java
List<Team> teams = em.createQuery(
    "select distinct t from Team t left join fetch t.members", Team.class)
    .getResultList();
```

fetch join은 연관 데이터도 함께 로딩하도록 명시한다. 위 예제는 회원이 없는 팀도 포함하기 위해 LEFT JOIN을 썼다. 컬렉션 조인에서는 팀 하나가 여러 SQL 행으로 펼쳐질 수 있다. `distinct`는 루트 엔티티 결과의 중복을 제거하려는 표현이며, Hibernate 6 이상은 fetch join의 루트 엔티티 중복을 자동 제거하므로 이 목적만으로 필수인 것은 아니다.

| 방법 | 사용 목적 | 확인할 점 |
| --- | --- | --- |
| fetch join | 해당 쿼리에 필요한 연관관계 함께 로딩 | 컬렉션 조인 시 행 증가·페이징 |
| EntityGraph | 로딩할 연관관계를 그래프로 지정 | SQL JOIN 방식이나 쿼리 1회를 항상 보장하지 않음 |
| Hibernate 배치 조회 | 미초기화 연관관계를 여러 ID의 IN 조회로 묶음 | 추가 쿼리를 줄이는 방식이며 배치 크기·접근 패턴에 영향받음 |
| DTO projection | 응답에 필요한 값만 직접 조회 | DTO를 만든 뒤 별도 연관 조회를 반복하지 않는지 확인 |

배치 조회는 Hibernate의 `@BatchSize` 또는 `hibernate.default_batch_fetch_size`로 설정할 수 있다. 여기서 말하는 배치는 연관 데이터 조회를 묶는 기능이며, INSERT·UPDATE의 JDBC 배치와 다르다.

### 컬렉션 fetch join과 페이징은 왜 조심해야 할까?

팀 하나에 회원이 여러 명이면 SQL 행 수와 팀 수가 달라진다. 이 상태에서 SQL 행에 LIMIT을 걸면 팀의 회원 컬렉션이 잘릴 수 있다. Hibernate는 컬렉션 fetch join과 페이징을 함께 사용하면 메모리에서 페이징하거나, 관련 설정에 따라 예외를 낼 수 있다.

대안은 **팀 목록을 DB에서 먼저 페이징하고 회원 컬렉션은 배치 조회**하거나, **팀 ID를 먼저 페이징한 뒤 그 ID들에 한해서 fetch join**하는 것이다. 두 단계 조회라면 최종 정렬 순서와 쿼리 사이의 데이터 변경도 고려한다.

검증할 때는 조회 메서드만 실행하지 않고 DTO 변환이나 응답 직렬화처럼 연관관계를 실제 사용하는 단계까지 SQL을 확인해야 한다. 작은 테스트 데이터와 이미 채워진 컨텍스트만으로는 N+1을 놓칠 수 있다.

참고: [Hibernate — Fetching](https://docs.hibernate.org/orm/7.1/userguide/html_single/#fetching), [Hibernate — Association Fetching](https://docs.hibernate.org/orm/7.1/querylanguage/html_single/#explicit-fetch-join), [Hibernate — BatchSize](https://docs.hibernate.org/orm/7.1/javadocs/org/hibernate/annotations/BatchSize.html), [Jakarta Persistence — Entity Graphs](https://jakarta.ee/specifications/persistence/3.2/jakarta-persistence-spec-3.2.html#a2397).

---

## 설명해 볼 질문

1. **객체의 필드만 바꿨는데 UPDATE가 나가는 이유는?** 관리 엔티티의 변경을 감지하고 flush 때 DB와 동기화하기 때문이다. 준영속 객체의 변경은 자동 반영되지 않는다.
2. **flush와 commit은 무엇이 다른가?** flush는 변경 동기화, commit은 트랜잭션 확정이다. flush 이후에도 트랜잭션은 롤백할 수 있다.
3. **1차 캐시가 있는데 왜 JPQL은 DB를 조회하는가?** 조건에 맞는 결과를 찾는 쿼리 캐시가 아니라, 관리 엔티티의 동일성을 유지하는 기능이기 때문이다.
4. **LAZY로 설정하면 N+1이 사라지는가?** 조회를 미룰 뿐이다. 목록을 순회하며 연관관계를 사용하면 추가 조회가 반복될 수 있다.
5. **fetch join을 쓰면 항상 해결되는가?** 필요한 연관관계를 함께 조회할 수 있지만, 컬렉션의 행 증가와 페이징 문제를 함께 고려해야 한다.
