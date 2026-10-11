# RDB / NoSQL과 키

## RDB

### 관계형 모델

- Relational Database. 데이터를 행(row)과 열(column)로 이루어진 테이블(릴레이션)에 저장하고, 테이블 사이의 관계를 키로 표현함
  - 1970년 E.F. Codd의 관계형 모델이 기반. 수학의 집합/관계 개념을 가져온 것이라 "릴레이션"임
  - 테이블 구조(스키마)를 미리 정의하고, 데이터는 그 구조를 따라야 함
- 용어 대응
  - 릴레이션 = 테이블, 튜플 = 행, 속성(attribute) = 열, 도메인 = 컬럼이 가질 수 있는 값의 범위
  - 차수(degree) = 컬럼 수, 카디널리티(cardinality) = 행 수. 인덱스 얘기할 때의 카디널리티(서로 다른 값의 수)와는 쓰임이 조금 다름
- SQL로 조회/조작. 선언형이라 "무엇을" 가져올지만 적으면 "어떻게" 가져올지는 DBMS(옵티마이저)가 결정함
  - DDL (CREATE / ALTER / DROP) : 구조 정의
  - DML (SELECT / INSERT / UPDATE / DELETE) : 데이터 조작
  - DCL (GRANT / REVOKE), TCL (COMMIT / ROLLBACK)
- 트랜잭션으로 ACID를 보장함 (11주차에서 자세히)
  - 원자성, 일관성, 격리성, 지속성. "돈을 보내는 쪽에서 빠졌으면 받는 쪽에 반드시 들어가야 한다"를 DB가 보장
- 대표 : MySQL, PostgreSQL, Oracle, MariaDB

### 스키마와 제약 조건

- 스키마가 고정되어 있다는 건 DB가 데이터의 모양을 검사해준다는 의미임
  - NOT NULL, UNIQUE, DEFAULT, CHECK, PK, FK
  - 잘못된 데이터가 들어오면 애플리케이션이 아니라 DB가 막아줌

```sql
CREATE TABLE members (
    id         BIGINT       NOT NULL AUTO_INCREMENT,
    email      VARCHAR(255) NOT NULL,
    age        INT          CHECK (age >= 0),
    status     VARCHAR(20)  NOT NULL DEFAULT 'ACTIVE',
    created_at DATETIME     NOT NULL DEFAULT CURRENT_TIMESTAMP,
    PRIMARY KEY (id),
    UNIQUE KEY uk_members_email (email)
);
```

- 짚고 넘어갈 것
  - 스키마가 엄격한 게 단점처럼 들리지만, 여러 서비스나 배치가 같은 DB를 바라볼 때는 DB가 최후의 방어선이 됨. 애플리케이션 코드는 경로가 여러 개라 하나쯤 검증이 빠질 수 있지만 DB 제약은 어디서 들어오든 똑같이 막음
  - 대신 컬럼 하나 추가하려면 ALTER TABLE이 필요하고, 수억 건 테이블에서는 이게 꽤 부담임. MySQL 8.0의 Instant ADD COLUMN, pt-online-schema-change, gh-ost 같은 온라인 스키마 변경 도구가 나온 이유



## NoSQL

### 종류

- Not Only SQL. 관계형 모델이 아닌 저장소를 통칭하는 말이라 종류마다 성격이 많이 다름

| 유형 | 구조 | 잘 맞는 용도 | 대표 |
| --- | --- | --- | --- |
| Key-Value | 키 하나에 값 하나 | 캐시, 세션, 랭킹, 분산 락 | Redis, DynamoDB |
| Document | JSON 같은 문서. 문서마다 필드가 달라도 됨 | 상품 카탈로그, 콘텐츠, 사용자 프로필 | MongoDB, CouchDB |
| Wide-Column | 행마다 다른 컬럼을 가질 수 있는 컬럼 패밀리 | 시계열, 로그, 메시지처럼 쓰기가 폭주하는 데이터 | Cassandra, HBase |
| Graph | 노드와 엣지로 관계 자체를 저장 | 소셜 그래프, 추천, 사기 탐지 | Neo4j |

- 검색 엔진(Elasticsearch)도 넓게 보면 Document 계열 NoSQL임. 역색인으로 전문 검색에 특화

### 왜 나왔나

- 2000년대 후반 웹 서비스 규모가 커지면서 RDB 한 대로 트래픽을 못 받게 됨
  - RDB는 scale-up(장비 키우기)은 쉬운데 scale-out(장비 늘리기)이 어려움. 조인과 트랜잭션이 여러 장비에 걸치면 복잡해지기 때문
  - NoSQL은 처음부터 데이터를 여러 노드에 나눠 저장(샤딩)하고 복제하는 걸 전제로 설계됨
- 그 대가로 포기한 것
  - 조인 : 대부분 지원하지 않거나 제한적. 필요한 데이터를 쓰기 시점에 미리 한 문서에 모아둠(비정규화)
  - 강한 일관성 : 여러 노드에 복제하다 보니 잠깐은 노드마다 값이 다를 수 있음. 결국엔 같아진다는 결과적 일관성(Eventual Consistency)을 택하는 경우가 많음
- CAP와 BASE
  - CAP 정리 : 분산 시스템은 일관성(Consistency), 가용성(Availability), 분할 내성(Partition tolerance) 셋을 동시에 만족할 수 없다. 네트워크 분할은 피할 수 없으니 실질적으로는 C와 A 중 하나를 고르는 문제
  - RDB는 C를, 많은 NoSQL은 A를 우선함. 그래서 NoSQL 쪽 성질을 ACID에 빗대어 BASE라고 부름 (Basically Available, Soft state, Eventually consistent)
  - 다만 Redis 단일 노드처럼 분산이 아니면 CAP 얘기가 의미 없고, MongoDB처럼 설정에 따라 C 쪽으로 기울일 수 있는 것도 많아서 "NoSQL = AP"로 단정하면 안 됨
- 정리하면..
  - RDB : 정규화해서 저장하고, 읽을 때 조인으로 조립
  - NoSQL : 읽는 모양대로 미리 조립해서 저장하고, 읽을 때는 그냥 꺼냄
- 설계 순서가 반대라는 점이 핵심임
  - NoSQL 모델링은 "어떤 쿼리가 들어올지"를 먼저 정하고 거기에 맞춰 데이터 모양을 잡음. RDB가 데이터 구조를 먼저 잡고 쿼리는 나중에 생각하는 것과 순서가 반대
  - 읽기 패턴이 바뀌면 데이터를 다시 쌓아야 할 수도 있음. 그래서 요구사항이 확정되지 않은 초기 서비스에는 RDB가 안전한 선택인 경우가 많음

### RDB와 NoSQL 선택

- RDB가 맞는 경우
  - 데이터 사이의 관계가 많고, 여러 테이블을 묶어서 조회하는 경우가 잦을 때
  - 돈, 재고, 예약처럼 정합성이 깨지면 안 되는 데이터
  - 쿼리 패턴을 미리 예측하기 어려울 때 (SQL은 어떤 조합으로도 질의할 수 있음)
- NoSQL이 맞는 경우
  - 읽기/쓰기 패턴이 단순하고 명확한데 양이 매우 많을 때 (로그, 세션, 캐시, 채팅 메시지, 알림)
  - 스키마가 자주 바뀌거나 항목마다 필드가 다를 때 (상품 옵션, 설문 응답 같은)
  - 한 대의 DB로 감당이 안 돼서 수평 확장이 필요할 때
- 실제로는 둘 중 하나를 고르기보다 섞어 씀 (Polyglot Persistence)
  - 주문/결제/회원은 MySQL, 세션과 캐시는 Redis, 상품 검색은 Elasticsearch, 알림 로그는 MongoDB 식으로 용도별로 나눔
  - 저장소가 늘어날수록 데이터 동기화와 운영 부담도 같이 늘어나므로, 필요가 증명된 뒤에 추가하는 게 보통
- 경계가 흐려지고 있음
  - MySQL/PostgreSQL은 JSON 컬럼과 JSON 함수를 지원해서 "일부 필드만 유연하게"가 가능함
  - MongoDB는 4.0부터 다중 문서 트랜잭션을, 5.0부터 시계열 컬렉션을 지원함
  - "NoSQL이 더 빠르다"는 말은 조건부임. 조인 없이 키로 단건 조회하는 패턴에서 빠른 거고, 그 패턴이면 RDB도 인덱스 하나로 충분히 빠름. 선택 기준은 속도보다 데이터 모양과 확장 방식임

### 면접 질문으로 정리하면..

- Q. RDB와 NoSQL의 차이는? → 스키마 고정 여부, 조인/트랜잭션 지원, 확장 방식(scale-up vs scale-out), 일관성 모델
- Q. 그럼 언제 NoSQL을 쓰나요? → 읽기 패턴이 명확하고 양이 많거나, 스키마가 유동적이거나, 수평 확장이 필요할 때. 예시 하나 들 수 있어야 함
- Q. NoSQL은 트랜잭션이 안 되나요? → 종류와 버전에 따라 다름. MongoDB는 4.0부터 지원. 다만 분산 트랜잭션은 비용이 커서 설계 단계에서 한 문서 안에서 끝나게 모델링하는 게 우선



&nbsp;

## PK / FK

### Primary Key

- 테이블에서 행 하나를 유일하게 식별하는 컬럼(또는 컬럼 조합)
  - UNIQUE + NOT NULL. 테이블당 하나만 가능
  - 여러 컬럼을 묶으면 복합 키 (Composite Key). 수강(학번, 과목코드)처럼 조합이 있어야 유일해지는 경우
- 키 관련 용어
  - 슈퍼키 : 행을 유일하게 식별할 수 있는 모든 컬럼 조합. (id), (id, email), (email, name) 전부 슈퍼키일 수 있음
  - 후보키 : 슈퍼키 중 더 이상 뺄 컬럼이 없는 최소 조합. (id), (email)
  - 기본키 : 후보키 중 대표로 고른 하나. (id)
  - 대체키 : 기본키로 뽑히지 않은 나머지 후보키. (email) → 보통 UNIQUE 제약으로 남겨둠
- 자연키와 대리키
  - 자연키 (Natural Key) : 주민번호, 이메일, 사업자번호처럼 데이터 자체에 있는 값
  - 대리키 (Surrogate Key) : AUTO_INCREMENT 정수, UUID처럼 식별만을 위해 만든 값
  - 자연키는 비즈니스 규칙이 바뀌면(이메일 변경 가능해짐, 주민번호 수집 금지 등) 키가 흔들리고, 키가 바뀌면 참조하는 모든 FK도 바꿔야 함. 그래서 보통 대리키를 PK로 두고 자연키에는 UNIQUE를 건다
- 대리키 종류별 특징
  - AUTO_INCREMENT : 작고(8바이트) 순차적이라 저장 효율이 가장 좋음. 단점은 값이 노출되면 전체 건수나 다음 id를 추측할 수 있고, DB가 여러 대면 중복이 날 수 있음
  - UUID v4 : 어디서 만들어도 겹치지 않아 분산 환경과 클라이언트 측 생성에 유리. 단점은 36자 문자열(바이너리로 16바이트)로 크고, 랜덤이라 삽입 시 B+Tree 여기저기에 흩어짐
  - UUID v7 / ULID / Snowflake : 앞부분이 시간이라 순차적이면서도 분산 생성이 가능함. 둘의 장점을 합친 최근 추세
- InnoDB에서는 PK가 곧 데이터가 저장되는 순서를 결정함 (클러스터링 인덱스). 그래서 PK를 뭘로 잡느냐가 저장 구조와 성능에 직접 영향을 줌. 이건 [Index / B+Tree](./index-btree.md)에서 정리

### Foreign Key

- 다른 테이블의 PK(또는 UNIQUE)를 참조하는 컬럼. 테이블 사이의 관계를 표현함
  - 참조 무결성 : 존재하지 않는 부모를 가리키는 자식 행이 생기지 않도록 DB가 보장함
  - 예 : `orders.member_id`는 `members.id`에 실제로 있는 값이어야 함. 없는 회원의 주문을 INSERT하면 에러

```sql
CREATE TABLE orders (
    id        BIGINT NOT NULL AUTO_INCREMENT,
    member_id BIGINT NOT NULL,
    PRIMARY KEY (id),
    CONSTRAINT fk_orders_member
        FOREIGN KEY (member_id) REFERENCES members (id)
        ON DELETE RESTRICT
        ON UPDATE CASCADE
);
```

- 부모 행이 삭제/수정될 때의 동작을 지정할 수 있음
  - RESTRICT / NO ACTION : 자식이 있으면 부모를 못 지움 (기본). 회원 탈퇴 전에 주문을 먼저 정리하라는 의미
  - CASCADE : 부모를 지우면 자식도 같이 지움. 게시글 삭제 시 댓글도 같이 지우는 경우. 연쇄 삭제 범위가 예상보다 커질 수 있어 주의
  - SET NULL : 자식의 FK 컬럼을 NULL로 바꿈. 작성자가 탈퇴해도 글은 남기되 작성자만 비우는 경우
- 관계의 종류와 FK 위치
  - 1:N : N쪽에 FK. 회원 1명이 주문 여러 건 → orders에 member_id
  - 1:1 : 어느 쪽에 둬도 되지만 보통 선택적인 쪽, 나중에 생기는 쪽에 두고 UNIQUE를 건다. 회원 - 회원상세
  - N:M : 중간 테이블을 만들어 1:N 두 개로 풀어냄. 학생 - 수강 - 과목
- 실무에서 FK 제약을 걸지 않는 팀도 많음
  - 자식 행을 넣을 때마다 부모 행 존재 여부를 확인하고 부모 행에 공유 락을 잡아서, 쓰기가 많은 테이블에서는 성능과 데드락에 영향
  - 테이블을 다른 DB로 분리하거나 샤딩할 때 FK가 걸림돌이 됨. DB가 다르면 FK를 걸 수 없음
  - 데이터 마이그레이션, 배치 삭제, 테스트 데이터 적재 시 순서를 맞춰야 해서 운영이 번거로움
  - 대신 애플리케이션 코드에서 정합성을 책임져야 함. DB가 막아주던 걸 사람이 막는 거라 트레이드오프이고, 고아 레코드를 주기적으로 찾아내는 배치를 두기도 함
- 제약을 안 걸더라도 FK 컬럼에 인덱스는 걸어야 함
  - MySQL은 FK 제약을 만들면 인덱스를 자동으로 만들지만, 제약 없이 컬럼만 두면 인덱스도 없음
  - FK 컬럼은 조인 조건으로 거의 항상 쓰이므로 인덱스가 없으면 조인마다 풀스캔이 남
- JPA의 `@ManyToOne` 같은 연관관계 매핑은 FK 컬럼을 객체 참조로 다루는 것. 연관관계의 주인 = FK를 가진 쪽 (10주차에서 자세히)

### 면접 질문으로 정리하면..

- Q. PK와 UNIQUE의 차이는? → 둘 다 유일성 보장. PK는 NULL 불가 + 테이블당 하나 + InnoDB에서는 클러스터링 인덱스. UNIQUE는 NULL 허용 + 여러 개 가능
- Q. PK로 UUID를 쓰면 어떤 문제가 있나요? → 크기와 랜덤 삽입. 세컨더리 인덱스까지 커지고 페이지 분할이 잦아짐. 대안은 UUID v7, ULID
- Q. FK를 안 거는 이유는? → 쓰기 성능, 락, 샤딩/DB 분리, 운영 유연성. 대신 애플리케이션이 정합성을 책임져야 한다는 말까지
