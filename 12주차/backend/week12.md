# Redis / Cache: 빠르게 읽으면서 오래된 값은 어떻게 다룰까?

같은 상품 정보를 매번 DB에서 조회한다면, 자주 읽는 결과를 메모리에 보관해 응답 시간과 DB 부하를 줄일 수 있다. 다만 DB의 값이 바뀌거나 캐시가 사라지는 순간에는 두 저장소를 함께 고려해야 한다. 이번 주는 상품 조회를 예로 캐시의 위치, 읽기·쓰기 전략, 무효화와 분산 락을 연결해 본다.

필수 키워드: **Local/Distributed Cache, Redis, TTL, Cache Aside, Cache Invalidation, Distributed Lock**.

예제에서는 관계형 DB가 원본이고 Redis는 다시 만들 수 있는 조회 결과를 보관한다. Redis 명령은 별도 설명이 없다면 단일 primary에 실행하며, 의사 코드는 캐시 흐름을 설명하기 위한 것으로 직렬화와 예외 처리 등을 생략했다. 상품 설명은 잠깐 오래된 값을 허용할 수 있지만, 주문 시 재고 검증은 DB에서 처리한다고 가정한다.

## 1. Local / Distributed Cache: 누가 같은 값을 공유할까?

| 구분 | Local Cache | Distributed Cache |
| --- | --- | --- |
| 위치 | 애플리케이션 프로세스 메모리 | 애플리케이션 외부의 공유 저장소 |
| 예시 | JVM 내부의 Caffeine | 별도 서버의 Redis |
| 접근 비용 | 프로세스 안에서 접근 | 네트워크·직렬화 비용 추가 |
| 여러 서버의 값 | 서버마다 별도 사본 | 같은 키 공간에 접근하도록 구성 가능 |
| 주요 고민 | 서버별 메모리 중복, 사본 무효화 | 네트워크 지연, 저장소 장애, 메모리 한도 |

```text
Local Cache                         Distributed Cache
서버 A → A의 캐시 → DB               서버 A ─┐
서버 B → B의 캐시 → DB                       ├→ Redis → miss이면 DB
                                    서버 B ─┘
```

Redis를 사용한다고 여러 서버의 조회 결과가 항상 최신인 것은 아니다. DB와 Redis가 다를 수 있고, replica에서 읽으면 복제 지연도 고려해야 한다. Redis의 기본 복제는 비동기다.

로컬 캐시와 Redis를 함께 쓰면 네트워크 접근을 더 줄일 수 있지만, Redis의 키를 지운 뒤에도 로컬 사본이 남을 수 있다. 두 계층의 TTL과 무효화 전달 방식을 함께 설계해야 한다.

참고: [Microsoft — Cache-Aside의 로컬 캐시 고려 사항](https://learn.microsoft.com/en-us/azure/architecture/patterns/cache-aside), [Redis — Replication](https://redis.io/docs/latest/operate/oss_and_stack/management/replication/).

## 2. Redis: 문자열 외에도 자료구조를 저장한다

Redis는 메모리를 중심으로 데이터를 다루는 저장소다. 캐시뿐 아니라 카운터, 순위, 이벤트 처리 등에도 사용한다.

| 자료형 | 명령 예 | 사용 예 |
| --- | --- | --- |
| String | `SET`, `GET`, `INCR` | 직렬화한 상품 DTO, 조회 수 |
| Hash | `HSET`, `HGET` | 객체의 필드별 값 |
| List | `LPUSH`, `RPOP` | 순서가 있는 목록 |
| Set | `SADD`, `SISMEMBER` | 중복 없는 ID 집합 |
| Sorted Set | `ZADD`, `ZRANGE` | 점수에 따른 순위 |

단일 `INCR` 같은 연산의 원자성과, 여러 명령을 조합한 작업의 원자성은 구분해야 한다. `GET`으로 숫자를 읽고 애플리케이션에서 더한 뒤 `SET`하면 다른 요청의 갱신을 덮어쓸 수 있다. 여러 Redis 연산을 묶어야 한다면 짧은 Lua 스크립트 등을 검토할 수 있지만, Redis에서의 원자성이 외부 DB 변경까지 묶어 주지는 않는다.

메모리 기반이라고 영속화 기능이 없는 것은 아니다. RDB 스냅샷과 AOF 로그를 지원하며, 설정에 따라 복구 범위가 달라진다. 이 글처럼 재생성 가능한 캐시로 쓸 때와, 데이터의 유일한 저장소로 쓸 때는 유실의 의미부터 다르다.

참고: [Redis — Data types](https://redis.io/docs/latest/develop/data-types/), [Redis — Scripting with Lua](https://redis.io/docs/latest/develop/programmability/eval-intro/), [Redis — Persistence](https://redis.io/docs/latest/operate/oss_and_stack/management/persistence/).

## 3. Cache Aside: 없을 때 DB에서 가져와 채운다

Cache Aside는 애플리케이션이 캐시와 DB를 직접 다루는 방식이다. 캐시에 값이 있으면 hit, 없으면 miss이며, miss일 때 DB에서 조회한 값을 캐시에 채운다.

```text
1. GET product:v1:42
2. hit  → 캐시 값 반환
3. miss → DB에서 상품 42 조회
4. 조회 결과를 Redis에 TTL과 함께 저장
5. 조회 결과 반환
```

```text
findProduct(id):
    key = "product:v1:" + id
    cached = cache.get(key)
    if cached exists:
        return cached

    product = database.findProduct(id)
    if product exists:
        cache.set(key, product, ttl = 300 seconds)
    return product
```

예제의 `v1`은 캐시 값의 구조를 바꿀 때 이전 형식과 구분하기 위한 키 버전이다. 실제 키에는 결과를 바꾸는 조건도 포함해야 한다. 같은 상품이라도 언어·테넌트·권한에 따라 결과가 다르면 상품 ID만으로 공유해서는 안 된다.

### 읽기와 쓰기의 책임은 어떻게 다를까?

| 방식 | 흐름 | 확인할 점 |
| --- | --- | --- |
| Cache Aside | 애플리케이션이 miss 시 DB 조회 후 캐시 저장 | 첫 조회 비용, DB 변경 후 무효화 |
| Read Through | 캐시 계층이 miss 시 원본 로딩을 담당 | 로더 구성과 실패 처리 |
| Write Through | 쓰기 경로에서 원본과 캐시를 동기적으로 갱신 | 쓰기 지연, 부분 실패 처리 |
| Write Behind | 캐시에 쓴 뒤 원본 반영을 비동기로 처리 | 반영 전 유실, 순서·재시도 |

이들은 읽기·쓰기 경로의 서로 다른 선택이므로 조합할 수 있다. Redis에 `SET`한다고 자동으로 DB에도 쓰이는 것은 아니다. Write Through 역시 두 저장소의 변경을 자동으로 하나의 트랜잭션으로 만드는 기능은 아니다.

참고: [Microsoft — Cache-Aside](https://learn.microsoft.com/en-us/azure/architecture/patterns/cache-aside), [AWS — Caching challenges and strategies](https://aws.amazon.com/builders-library/caching-challenges-and-strategies/).

## 4. TTL: 유효 기간과 메모리 확보를 구분하기

TTL(Time To Live)은 키의 남은 유효 시간이다. 아래는 `redis-cli`에 입력할 수 있는 명령이다.

```redis
SET product:v1:42 '{"id":42,"name":"키보드"}' EX 300
GET product:v1:42
TTL product:v1:42
```

`EX`는 초, `PX`는 밀리초 단위다. `TTL`은 초 단위의 남은 시간을 반환하며, Redis 2.8 이상에서 `-1`은 만료 설정이 없음, `-2`는 키가 없음을 뜻한다. 일반적인 `GET`은 TTL을 연장하지 않는다.

```redis
SET product:v1:42 '{"id":42,"name":"새 키보드"}'
TTL product:v1:42
```

일반 `SET`으로 값을 덮어쓰면 기존 TTL이 사라져 위 결과는 `-1`이 된다. 만료를 계속 적용하려면 다시 `EX`를 지정하거나, 기존 TTL을 보존하려는 경우 지원 버전에서 `KEEPTTL`을 사용한다. 캐시 저장 때 `SET`과 `EXPIRE`를 별도로 실행하면 그 사이 실패로 만료 없는 키가 남을 수 있으므로 값과 만료를 함께 설정한다.

만료 시각마다 별도 타이머가 즉시 메모리를 회수하는 구조로 이해하면 안 된다. Redis는 접근 시 만료를 확인하고, 주기적으로 만료된 키도 찾아 삭제한다.

### Expiration과 Eviction

| 구분 | 기준 | 예 |
| --- | --- | --- |
| Expiration | 키의 유효 기간 경과 | 300초 TTL이 지난 상품 캐시 |
| Eviction | `maxmemory`에 따른 메모리 압박 | LRU/LFU 정책으로 키 제거 |

`allkeys-lru`는 전체 키를 후보로 최근 덜 사용한 키를, `allkeys-lfu`는 덜 자주 사용한 키를 제거한다. `volatile-*`는 만료 설정이 있는 키만 후보로 삼는다. `noeviction`은 메모리 확보를 위해 키를 쫓아내지 않고, 메모리를 늘리는 쓰기 명령 등에 오류를 반환한다. TTL에 따른 만료는 계속 동작한다.

따라서 **TTL이 남아 있어도 키가 제거될 수 있다.** 캐시는 언제든 miss가 날 수 있다는 전제로 작성한다. TTL은 오래된 사본의 수명을 제한하는 수단이지만, DB 변경 직후의 최신성까지 보장하지 않는다.

참고: [Redis — SET](https://redis.io/docs/latest/commands/set/), [TTL](https://redis.io/docs/latest/commands/ttl/), [EXPIRE](https://redis.io/docs/latest/commands/expire/), [Key eviction](https://redis.io/docs/latest/develop/reference/eviction/).

## 5. Cache Invalidation: DB를 바꾸고 캐시를 지우면 끝일까?

상품 수정의 기본 흐름은 **DB 트랜잭션 커밋 → 해당 캐시 삭제**로 잡을 수 있다. 먼저 캐시를 삭제하면 DB 커밋 전에 다른 요청이 예전 값을 다시 채울 수 있다.

하지만 아래는 커밋 후 삭제해도 오래된 값이 남는 경우다. DB의 초기 값은 `old`, 캐시는 비어 있다고 하자.

```text
순서   읽기 요청 A                         수정 요청 B
1      캐시 miss
2      DB에서 old 조회
3                                         DB를 new로 변경하고 커밋
4                                         캐시 삭제
5      앞서 읽은 old를 캐시에 저장
6      이후 요청은 캐시에서 old를 읽음
```

이 시간표는 읽기와 쓰기가 겹칠 때의 예시다. 캐시 삭제만으로 이미 진행 중인 로더를 취소하거나, 그 로더가 읽은 DB 버전을 바꿀 수는 없다.

대응은 허용할 수 있는 오래된 값의 범위에 따라 달라진다. 짧은 TTL로 사본의 수명을 제한하거나, 읽기 채움과 쓰기 무효화가 같은 동기화 규칙을 따르게 할 수 있다. 버전을 비교하는 방식도 비교 기준과 캐시 저장 판단을 원자적으로 연결해야 한다. DB와 무관하게 Redis에 버전 필드만 추가한다고 경쟁 조건이 사라지지는 않는다.

커밋 직후 프로세스가 종료되거나 Redis 삭제가 실패하는 경우도 별도 문제다. 무효화 이벤트의 재시도가 필요하면 DB 변경과 이벤트 기록을 같은 트랜잭션에 남기는 outbox나 CDC를 검토할 수 있다. 이런 방식도 이벤트 전달 지연과 위의 재삽입 경쟁을 각각 다뤄야 한다.

### Spring에서는 메서드 종료와 커밋을 구분한다

`@Cacheable`은 결과를 캐시하고, `@CacheEvict`는 캐시를 제거한다. 기본 `@CacheEvict`의 실행 기준은 메서드의 성공적인 종료다. 트랜잭션·캐시 인터셉터의 순서와 CacheManager 설정에 따라 실제 삭제 시점이 달라질 수 있으므로, `@Transactional`과 함께 붙였다는 사실만으로 커밋 후 삭제라고 단정하면 안 된다.

커밋 이후 처리를 명시하는 방법 중 하나가 트랜잭션 이벤트다. 아래는 수정 서비스가 활성 트랜잭션 안에서 `ProductChanged` 이벤트를 발행하고, 별도 Spring 빈이 받는 예제다. `cache`는 캐시 삭제를 제공하는 객체라고 가정한다.

```java
@TransactionalEventListener(phase = TransactionPhase.AFTER_COMMIT)
public void invalidate(ProductChanged event) {
    cache.delete("product:v1:" + event.productId());
}
```

기본 설정에서는 트랜잭션 없이 발행한 이벤트에 이 리스너가 실행되지 않는다. 또한 이 예제의 일반적인 프로세스 내부 이벤트는 영속 메시지 큐가 아니므로 프로세스 종료나 삭제 실패를 복구하는 재시도 장치가 따로 필요하다. 커밋 후 삭제 시점을 맞추는 것과 무효화 성공을 보장하는 것은 다른 문제다.

참고: [Spring Framework — Cache annotations](https://docs.spring.io/spring-framework/reference/integration/cache/annotations.html), [Transaction-bound Events](https://docs.spring.io/spring-framework/reference/data-access/transaction/event.html). 위 시간표와 실패 시나리오는 Cache Aside 흐름을 바탕으로 구성했다.

## 6. miss가 몰리면 DB는 괜찮을까?

인기 상품의 캐시 하나가 만료되면 여러 요청이 동시에 DB에서 같은 값을 조회할 수 있다. 이를 Cache Stampede라고 한다. 평소 DB 요청 대부분을 캐시가 흡수했다면, 캐시 장애 때 DB로 우회하는 것만으로는 부하를 버티지 못할 수 있다.

| 상황 | 대응 예 | 남는 문제 |
| --- | --- | --- |
| 한 키에 동시 miss | 같은 키의 로딩 요청 합치기, 재생성 락 | 대기 시간 제한, 로더 실패 |
| 여러 키가 함께 만료 | TTL에 무작위 편차(jitter) 추가 | 한 인기 키의 동시 miss는 별도 대응 |
| 존재하지 않는 ID 반복 조회 | 짧은 TTL로 없음 결과 캐시 | 실제 데이터 생성 시 무효화 |
| Redis 장애 | 짧은 타임아웃, 회로 차단, DB 유입량 제한 | DB 용량에 따라 요청 거절·기능 축소 필요 |

재생성 락을 얻은 요청은 캐시를 다시 확인한 뒤 DB를 조회한다. 기다리는 동안 다른 요청이 이미 채웠을 수 있기 때문이다. 락을 얻지 못한 요청은 무한 대기 대신 제한된 재조회나 허용 가능한 이전 값 반환 등을 선택한다. 이전 값 반환은 상품 설명에는 가능해도 결제·재고 판정에는 맞지 않을 수 있다.

Spring의 `@Cacheable(sync = true)`는 캐시 제공자에 동기화된 로딩을 요청한다. 실제 지원 방식과 범위를 확인해야 하며, 이 옵션만 보고 여러 서버 전체에서 DB 조회가 한 번이라고 판단해서는 안 된다.

참고: [AWS — Caching challenges and strategies](https://aws.amazon.com/builders-library/caching-challenges-and-strategies/), [Spring Framework — Synchronized caching](https://docs.spring.io/spring-framework/reference/integration/cache/annotations.html#cache-annotations-cacheable-synchronized).

## 7. Distributed Lock: 락이 만료되어도 작업은 계속될 수 있다

한 JVM 안의 `synchronized`는 다른 서버의 요청을 막지 못한다. 여러 프로세스의 작업을 조율할 때 공유 저장소의 락을 사용할 수 있다. 단일 Redis에서의 기본 획득 예는 다음과 같다.

```redis
SET lock:product:42 unique-token-A NX PX 10000
```

`NX`는 키가 없을 때만 설정하고, `PX 10000`은 10초 후 만료시킨다. 값에는 **락 획득 시도마다 새로 만든 충돌 가능성이 충분히 낮은 토큰**을 넣는다. 토큰은 소유자 확인용이며, 락 유효 기간 안에서 작업하는 것을 전제로 한다.

락 키도 eviction 대상이다. `allkeys-*`뿐 아니라 `volatile-*`도 TTL이 있는 락을 제거할 수 있어, 10초가 지나기 전에 다른 요청이 락을 얻을 수 있다. 락 저장소는 캐시와 분리한 `noeviction` 인스턴스 등으로 조기 제거를 막고, 메모리 부족으로 획득 명령이 실패하는 경우도 처리한다. 이는 만료·재시작·failover에 대한 보호와 별개다.

획득과 만료 설정을 별도 명령으로 나누면 중간 실패로 락이 남을 수 있다. 획득 성공을 확인한 요청만 작업하며, 통신 타임아웃은 명확한 획득 실패와 구분한다. 서버에는 락이 생겼지만 응답만 받지 못했을 수 있기 때문이다.

### 해제 시에는 토큰 비교와 삭제를 묶는다

```lua
-- KEYS[1]: 락 키, ARGV[1]: 획득 시 사용한 토큰
if redis.call("get", KEYS[1]) == ARGV[1] then
    return redis.call("del", KEYS[1])
end
return 0
```

이 Lua 스크립트는 Redis 안에서 비교와 삭제를 원자적으로 실행한다. 애플리케이션이 `GET` 후 따로 `DEL`하면 그 사이 락이 만료되고 다른 요청이 획득할 수 있다. Redis 8.4 이상에서는 `DELEX key IFEQ token`으로 조건부 삭제도 가능하며, 위 스크립트는 이전 버전에서도 사용할 수 있는 방식이다.

### 안전한 해제가 작업의 상호 배제까지 해결할까?

```text
1. A가 10초짜리 락 획득
2. A의 처리가 긴 GC 정지 등으로 멈춤
3. 락 만료 후 B가 새 락을 획득하고 작업
4. A가 재개하여 자신도 작업을 계속함
```

토큰을 비교하면 A가 B의 락을 삭제하는 것은 막을 수 있지만, A가 DB에 쓰는 행위까지 막지는 못한다. 자동 연장도 지연·연장 실패 상황을 고려해야 한다. 강한 보호가 필요하면 저장소가 오래된 작업의 쓰기를 거절하도록 해야 한다.

예를 들어 fencing token은 획득 순서에 따라 증가하는 번호를 발급하고, 보호 대상 저장소가 이미 수락한 번호보다 작은 요청을 거절하는 방식이다. 앞의 임의 소유자 토큰과 역할이 다르다. 번호의 발급·비교·저장이 장애 상황에서도 안전해야 하며, UUID나 초기화될 수 있는 Redis 카운터만 추가해서 같은 보장을 얻지는 못한다.

또한 비동기 복제 중 primary가 죽고 락 기록이 없는 replica가 승격되면 다른 요청이 락을 얻을 수 있다. Sentinel이나 replica 추가만으로 상호 배제가 보장되지는 않는다. Redis 공식 문서는 여러 독립 primary를 사용하는 Redlock도 설명하지만, 시간·장애 가정과 fencing에 대한 주의 사항을 함께 제시한다.

재고 차감처럼 정합성이 핵심이라면 캐시 재생성 락과 구분해 DB 조건부 UPDATE, 낙관적 버전 검사, DB 락 등으로 최종 불변식을 지키는 방법을 검토한다. 재생성 중복을 줄이는 목적과 업무 데이터의 정확성을 지키는 목적은 요구하는 보장이 다르다.

참고: [Redis — Distributed locks](https://redis.io/docs/latest/develop/clients/patterns/distributed-locks/), [Redis — Scripting with Lua](https://redis.io/docs/latest/develop/programmability/eval-intro/), [Hazelcast — FencedLock의 fencing 설명](https://docs.hazelcast.com/hazelcast/latest/data-structures/fencedlock), [Redis — Replication](https://redis.io/docs/latest/operate/oss_and_stack/management/replication/).

## 적용 전에 확인할 것

캐시를 넣기 전에는 DB 쿼리와 인덱스로 해결할 수 있는 문제인지, 실제로 같은 값이 반복 조회되는지 확인한다. 적용 후에는 적중률뿐 아니라 응답 지연과 DB 유입량을 함께 본다.

- **정확성:** 데이터별로 오래된 값을 얼마나 허용하는가? 수정 시 상세·목록·로컬 사본 중 무엇을 무효화하는가?
- **용량:** 키 수와 값 크기, TTL, `maxmemory` 및 eviction 정책이 맞는가?
- **성능:** hit/miss별 지연, DB QPS, 타임아웃, 재생성 대기 시간이 개선되었는가?
- **장애:** 캐시 전체가 비었거나 Redis가 느릴 때 DB 유입량을 제한할 수 있는가?

전체 hit 비율이 높더라도 한 개의 느린 쿼리나 특정 인기 키가 병목일 수 있다. 캐시를 비운 상태와 동시 요청 상황도 확인해야 한다.

## 설명해 볼 질문

1. **Local Cache와 Redis를 함께 쓰면 무엇이 어려워지는가?** 각 서버의 로컬 사본까지 무효화해야 하며, 계층별 TTL과 변경 전달 실패를 고려해야 한다.
2. **Cache Aside에서 DB 커밋 후 캐시를 삭제하면 항상 최신인가?** 아니다. 먼저 예전 값을 읽은 요청이 삭제 이후 그 값을 다시 채울 수 있다.
3. **TTL이 남았는데 왜 키가 없어질 수 있는가?** 메모리 압박에 따른 eviction은 TTL 만료와 별개다.
4. **여러 키의 TTL에 무작위 편차를 주면 stampede가 모두 해결되는가?** 여러 키의 동시 만료는 분산할 수 있지만, 한 인기 키의 동시 miss는 별도 대응이 필요하다.
5. **분산 락을 그냥 DEL로 해제하면 왜 위험한가?** 내 락이 만료된 뒤 다른 요청이 얻은 락을 지울 수 있다. 소유자 토큰 비교와 삭제를 원자적으로 실행해야 한다.
6. **소유자 토큰을 비교하면 재고 차감도 안전한가?** 해제의 안전성과 DB 쓰기의 안전성은 다르다. 만료 후에도 계속되는 작업을 저장소가 제어할 수 있어야 한다.
