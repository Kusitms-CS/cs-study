# 10주차 - JPA / 영속성 컨텍스트

> **영속성 컨텍스트** = JPA가 엔티티를 보관·관리하는 "작업 공간".
> 엔티티를 이 안에서 관리해주기 때문에 1차 캐시, 변경 감지, 쓰기 지연 같은 기능이 가능하다.

---

## 1. Entity Lifecycle (엔티티 생명주기)

| 상태 | 설명 | 진입 방법 |
|---|---|---|
| 비영속 (new) | `new`로 만든 객체. JPA와 무관| `new Member()` |
| 영속 (managed) | 영속성 컨텍스트가 관리 중 | `persist()`, `find()` |
| 준영속 (detached) | 관리하다가 분리됨 | `detach()`, `clear()`, `close()` |
| 삭제 (removed) | 삭제 예정 | `remove()` |

```java
Member member = new Member("경수");   // 비영속
em.persist(member);                   // 영속 (아직 INSERT 안 나감)
em.detach(member);                    // 준영속
em.remove(member);                    // 삭제
```

**핵심:** 영속 상태일 때만 JPA의 기능(변경 감지 등)이 적용된다.

---

## 2. 1차 Cache

- 영속성 컨텍스트 안의 **Map 캐시** (key: `@Id`, value: 엔티티)
- `find()` 흐름: **1차 캐시 확인 → 없으면 DB 조회 → 캐시에 저장**

```java
Member a = em.find(Member.class, 1L);  // DB 조회 (SELECT 1번)
Member b = em.find(Member.class, 1L);  // 캐시에서 반환 (SELECT 없음)
a == b;                                // true (동일성 보장)
```

- 범위는 **트랜잭션 단위**라서 성능 이점은 크지 않다. 진짜 장점은 **같은 트랜잭션 안에서 동일성 보장**.

---

## 3. Dirty Checking (변경 감지)

- 엔티티를 조회할 때 **스냅샷(최초 상태)** 을 같이 저장
- flush 시점에 **현재 값 vs 스냅샷** 비교 → 다르면 UPDATE 자동 생성

```java
Member member = em.find(Member.class, 1L);
member.setName("바뀐 이름");   // update() 호출 안 해도 됨!
// 트랜잭션 커밋 시 → UPDATE 쿼리 자동 실행
```

- 영속 상태 엔티티만 대상. **준영속이면 변경 감지 안 됨.**

---

## 4. Flush

- 영속성 컨텍스트의 변경 내용을 **DB에 동기화** (SQL 전송)
- ⚠️ 헷갈리지 말 것: 컨텍스트를 비우는 것 X, 커밋 X

**발생 시점**
1. 트랜잭션 커밋 직전
2. JPQL 쿼리 실행 직전 (조회 결과가 최신이어야 하니까)
3. `em.flush()` 직접 호출

**쓰기 지연:** `persist()` 하면 SQL을 바로 안 보내고 모아뒀다가 flush 때 한 번에 전송.
(단, `IDENTITY` 전략은 PK를 알아야 해서 `persist()` 시점에 바로 INSERT)

---

## 5. Lazy / Eager (지연 로딩 / 즉시 로딩)

| | LAZY | EAGER |
|---|---|---|
| 조회 시점 | 실제로 사용할 때 | 엔티티 조회할 때 같이 |
| 동작 | 프록시 객체로 대체했다가 나중에 쿼리 | 처음부터 JOIN 또는 추가 쿼리 |

```java
Member m = em.find(Member.class, 1L);
Team t = m.getTeam();       // LAZY: 아직 프록시, 쿼리 없음
t.getName();                // 이 순간 SELECT 실행
```

- 기본값: `@ManyToOne`, `@OneToOne` → EAGER / `@OneToMany`, `@ManyToMany` → LAZY
- **실무 원칙: 전부 LAZY로 설정**하고 필요할 때만 fetch join
- 주의: 준영속 상태에서 LAZY 접근 시 `LazyInitializationException`

---

## 6. N+1 문제

**"쿼리 1번으로 끝날 줄 알았는데 N번이 추가로 나가는 문제"**

```java
List<Team> teams = em.createQuery("select t from Team t", Team.class)
                     .getResultList();   // ① 쿼리 1번 (팀 N개 조회)

for (Team t : teams) {
    t.getMembers().size();               // ② 팀마다 쿼리 → N번
}
// 총 1 + N번
```

**왜 EAGER로 해결이 안 되나?**
JPQL은 쿼리를 그대로 실행하고, 그 후에 EAGER 연관관계를 채우려고 추가 쿼리를 날린다. 결국 똑같이 N+1.

**해결 방법**
| 방법 | 설명 |
|---|---|
| **fetch join** | `select t from Team t join fetch t.members` → 한 방에 조회 |
| `@EntityGraph` | 메서드에 붙여서 fetch join 효과 |
| `@BatchSize` / `default_batch_fetch_size` | N번 대신 IN 쿼리로 묶어서 조회 |

⚠️ 컬렉션 fetch join + 페이징 = 메모리에서 페이징(위험). 이럴 땐 `@BatchSize` 사용.

---

## 전체 흐름 한눈에 보기

```
persist/find  →  [영속성 컨텍스트]  →  flush  →  DB
                  ├ 1차 캐시 (조회 최적화)
                  ├ 스냅샷 (변경 감지)
                  └ 쓰기 지연 저장소 (SQL 모아두기)

연관관계는 LAZY로 → 필요할 때 fetch join → N+1 방지
```