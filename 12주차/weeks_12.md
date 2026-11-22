# 12주차 - Redis / Cache (Backend)

같은 데이터를 매번 DB에서 읽지 않도록 **어디에, 얼마나 오래, 어떻게 맞춰서** 저장해 둘지 정리했다. 그리고 서버가 여러 대일 때 Redis로 Lock을 거는 방법까지 다룬다. 예시는 Spring Boot 3 + Spring Data Redis(Lettuce) + MySQL 기준이다.

## 1. Cache 기본

캐시는 **느린 저장소의 결과를 빠른 저장소에 복사해 두고 재사용**하는 것이다. 3주차의 CPU 캐시(L1/L2/L3), 5주차의 HTTP 캐시와 원리가 같다.

| 저장소 | 대략적인 접근 시간 |
| --- | --- |
| JVM Heap (Local Cache) | 수십~수백 ns |
| Redis (같은 데이터센터 네트워크) | 수백 µs ~ 1ms |
| MySQL (인덱스 타는 단건 조회) | 1ms ~ 수 ms |
| MySQL (복잡한 JOIN·집계) | 수십 ms ~ 수 초 |
| 외부 API | 수십 ms ~ 수 초 |

### 캐시에 넣기 좋은 데이터

| 조건 | 예 |
| --- | --- |
| 읽기가 쓰기보다 훨씬 많다 | 상품 상세, 카테고리 목록, 공지사항 |
| 계산 비용이 크다 | 랭킹, 통계, 추천 결과 |
| 조금 오래된 값이어도 괜찮다 | 조회수, 좋아요 수, 인기 검색어 |
| 같은 요청이 반복된다 | 환율, 설정값, 외부 API 응답 |

반대로 **잔액, 재고, 결제 상태**처럼 항상 정확해야 하는 값은 캐시를 "읽기 보조"로만 쓰고, 판단은 DB에서 한다.

### Cache Hit Ratio

```
Hit Ratio = Hit / (Hit + Miss)
평균 응답 시간 ≈ Hit Ratio × 캐시 응답 시간 + (1 - Hit Ratio) × (캐시 조회 + DB 응답 시간)
```

Hit Ratio가 낮으면 캐시 조회 시간만큼 오히려 느려진다. 캐시를 넣었으면 **Hit/Miss 지표를 꼭 본다** (Redis `INFO stats`의 `keyspace_hits`, `keyspace_misses`, Caffeine의 `recordStats()`).

## 2. Local Cache / Distributed Cache

### 비교

| 구분 | Local Cache | Distributed Cache (Global Cache) |
| --- | --- | --- |
| 위치 | 각 서버의 JVM Heap | 별도 캐시 서버 (Redis, Memcached) |
| 대표 구현 | Caffeine, Ehcache, `ConcurrentHashMap` | Redis, Memcached |
| 속도 | 가장 빠름 (네트워크·직렬화 없음) | 네트워크 왕복 + 직렬화 비용 |
| 서버 간 일관성 | 서버마다 값이 다를 수 있음 | 모든 서버가 같은 값을 봄 |
| 용량 | Heap 크기에 묶임, 크면 GC 부담 | 캐시 서버 메모리만큼, 확장 가능 |
| 서버 재시작 | 사라짐 (Cold Start) | 유지됨 |
| 장애 영향 | 해당 서버만 | 캐시 서버가 죽으면 모든 서버가 DB로 몰림 |

```
[Local Cache]                         [Distributed Cache]
 서버 A ─ Heap{product:1 = v1}          서버 A ─┐
 서버 B ─ Heap{product:1 = v2}  ← 불일치  서버 B ─┼──▶ Redis{product:1 = v2}
 서버 C ─ Heap{ 없음 }                    서버 C ─┘
```

### Local Cache: Caffeine

```java
@Configuration
@EnableCaching
public class LocalCacheConfig {

    @Bean
    public CacheManager caffeineCacheManager() {
        CaffeineCacheManager manager = new CaffeineCacheManager("categories", "configs");
        manager.setCaffeine(Caffeine.newBuilder()
                .maximumSize(10_000)                    // 크기 제한 필수, 없으면 OOM 위험
                .expireAfterWrite(Duration.ofMinutes(10))
                .recordStats());
        return manager;
    }
}
```

Caffeine은 **W-TinyLFU** 알고리즘으로 "최근에 + 자주" 쓰인 항목을 남겨서 단순 LRU보다 Hit Ratio가 높다. `maximumSize` 없이 쓰면 1주차의 `HashMap`을 캐시로 쓰는 것과 같아서 메모리가 계속 늘어난다.

### 서버 간 Local Cache 맞추기

Local Cache를 여러 서버에서 쓰면, 한 서버에서 값을 바꿔도 다른 서버의 캐시는 그대로다.

| 방법 | 설명 |
| --- | --- |
| 짧은 TTL | 불일치를 허용하되 최대 N초로 제한. 가장 단순 |
| Redis Pub/Sub 무효화 | 변경한 서버가 `cache-evict` 채널에 키를 발행 → 모든 서버가 자기 Local Cache에서 삭제 |
| 바뀌지 않는 데이터만 | 코드성 데이터, 설정값처럼 배포 때만 바뀌는 값 |

### 2-Level Cache

```
요청 → L1 (Caffeine, TTL 짧게) ─miss→ L2 (Redis, TTL 길게) ─miss→ DB
```

트래픽이 매우 큰 **Hot Key**(메인 배너, 실시간 인기 상품)는 Redis 한 노드에 요청이 몰리기 때문에, 앞단에 짧은 TTL의 Local Cache를 두면 Redis 부하가 크게 줄어든다.

## 3. Redis

**Re**mote **Di**ctionary **S**erver. 메모리에 데이터를 저장하는 Key-Value 저장소로, 단순 문자열뿐 아니라 여러 자료구조를 지원한다.

### 왜 빠른가

| 이유 | 설명 |
| --- | --- |
| In-Memory | 디스크 I/O 없이 메모리에서 읽고 씀 |
| 단일 스레드 명령 실행 | Lock·Context Switching 없음 (2주차) |
| I/O Multiplexing | `epoll`/`kqueue`로 한 스레드가 수많은 소켓을 처리 (4주차) |
| 단순한 자료구조 | 대부분 명령이 O(1) 또는 O(log N) |

Redis 6.0부터 **네트워크 읽기/쓰기는 I/O 스레드**로 나눌 수 있지만(`io-threads`), **명령 실행은 여전히 단일 스레드**다. 그래서 한 명령은 원자적으로 실행되고, 동시에 다른 명령이 끼어들지 않는다.

반대로 말하면 **느린 명령 하나가 전체를 멈춘다.**

| 피해야 할 명령 | 이유 | 대안 |
| --- | --- | --- |
| `KEYS *` | 전체 키를 O(N)으로 훑으며 블로킹 | `SCAN` (커서 기반, 나눠서 조회) |
| 큰 컬렉션 `DEL` | 수백만 원소 해제 동안 블로킹 | `UNLINK` (백그라운드 해제) |
| `HGETALL`, `SMEMBERS`, `LRANGE 0 -1` (큰 키) | 결과 전체를 한 번에 응답 | `HSCAN`, `SSCAN`, 범위 나눠 조회 |
| `FLUSHALL` | 전체 삭제 | 운영에서는 `rename-command`로 막아 둠 |

### 자료구조와 쓰임

| 타입 | 특징 | 백엔드 활용 | 주요 명령 |
| --- | --- | --- | --- |
| String | 최대 512MB, 숫자 증감 가능 | 캐시(JSON), 카운터, 분산 Lock | `GET` `SET` `INCR` `SETNX` |
| Hash | 필드-값 쌍 | 객체 필드 단위 저장, 장바구니 | `HSET` `HGET` `HINCRBY` |
| List | 양방향 연결 리스트 | 최근 본 상품, 간단한 큐 | `LPUSH` `RPOP` `LTRIM` |
| Set | 중복 없는 집합 | 좋아요한 사용자, 중복 방지 | `SADD` `SISMEMBER` `SCARD` |
| Sorted Set | score로 정렬된 집합 (Skip List) | 랭킹, 대기열, 시간순 피드 | `ZADD` `ZRANGE` `ZRANK` |
| Bitmap | 비트 단위 | 일별 출석 체크 | `SETBIT` `BITCOUNT` |
| HyperLogLog | 오차 0.81%, 12KB 고정 | 순 방문자(UV) 추정 | `PFADD` `PFCOUNT` |
| Stream | 로그형, Consumer Group | 이벤트 처리, 메시지 큐 | `XADD` `XREADGROUP` |

```bash
# 실시간 랭킹 (Sorted Set)
ZINCRBY ranking:2026-11-22 1 "product:42"
ZREVRANGE ranking:2026-11-22 0 9 WITHSCORES   # Top 10

# 조회수 (원자적 증가 — 11주차의 Lost Update가 생기지 않음)
INCR view:product:42
```

### 영속성 (Persistence)

메모리 데이터라 재시작하면 사라지므로, 디스크에 남기는 방법이 두 가지 있다.

| 방식 | 동작 | 장점 | 단점 |
| --- | --- | --- | --- |
| RDB (Snapshot) | 일정 주기로 메모리 전체를 파일로 덤프. `fork()` 후 자식 프로세스가 기록 (Copy-on-Write, 3주차) | 파일이 작고 복구가 빠름 | 마지막 스냅샷 이후 데이터 유실 |
| AOF (Append Only File) | 쓰기 명령을 로그로 계속 추가. `appendfsync everysec`(기본) | 유실이 최대 약 1초 | 파일이 크고 복구가 느림 (Rewrite로 압축) |

캐시 용도면 영속성을 꺼도 된다. 다만 재시작 직후 모든 요청이 DB로 가는 **Cold Start**를 감안해야 한다. 세션·Lock·랭킹처럼 **원본이 Redis에만 있는 데이터**라면 RDB + AOF를 함께 쓴다. 11주차의 Redo Log(WAL)와 같은 발상이다.

### 고가용성 구성

| 구성 | 설명 |
| --- | --- |
| Replication | Master 1 + Replica N. 쓰기는 Master, 읽기는 Replica로 분산 가능. **복제는 비동기** |
| Sentinel | Master를 감시하다가 장애 시 Replica를 Master로 승격 (Failover) |
| Cluster | 데이터를 **16384개 Hash Slot**으로 나눠 여러 Master에 분산 (`CRC16(key) % 16384`). 각 Master에 Replica |

Cluster에서는 여러 키를 다루는 명령(`MGET`, Lua 스크립트, 트랜잭션)이 **같은 Slot**에 있어야 한다. `{user:1}:cart`, `{user:1}:profile`처럼 **Hash Tag** `{}`를 쓰면 괄호 안 문자열로만 Slot을 계산해서 같은 노드에 모을 수 있다.

### Spring Boot 설정

```yaml
spring:
  data:
    redis:
      host: localhost
      port: 6379
      timeout: 1s            # 명령 타임아웃. Redis가 느릴 때 요청 스레드가 묶이지 않게
      lettuce:
        pool:                # commons-pool2 의존성 필요. 블로킹 명령·트랜잭션 없으면 풀 없이도 충분
          max-active: 16
```

Spring Boot의 기본 클라이언트는 **Lettuce**다. Netty 기반 비동기 클라이언트라 커넥션 하나를 여러 스레드가 공유할 수 있다 (Jedis는 스레드마다 커넥션 필요).

```java
@Configuration
public class RedisConfig {

    @Bean
    public RedisTemplate<String, Object> redisTemplate(RedisConnectionFactory factory) {
        RedisTemplate<String, Object> template = new RedisTemplate<>();
        template.setConnectionFactory(factory);
        template.setKeySerializer(new StringRedisSerializer());
        template.setValueSerializer(new GenericJackson2JsonRedisSerializer()); // 기본값은 JDK 직렬화
        template.setHashKeySerializer(new StringRedisSerializer());
        template.setHashValueSerializer(new GenericJackson2JsonRedisSerializer());
        return template;
    }
}
```

기본 `RedisTemplate`은 **JDK 직렬화**를 써서 `redis-cli`로 보면 `\xac\xed\x00\x05...`처럼 깨져 보이고, 클래스 패키지·필드가 바뀌면 역직렬화가 실패한다. JSON 직렬화로 바꾸는 것이 일반적이다. 단, `GenericJackson2JsonRedisSerializer`는 `@class` 타입 정보를 함께 저장하므로 **DTO 패키지를 옮기면 기존 캐시를 읽지 못한다** → 배포 시 캐시 키에 버전을 붙이거나(`product:v2:42`) 캐시를 비운다.

## 4. TTL (Time To Live)

키에 만료 시간을 걸어 두면 그 시간이 지난 뒤 자동으로 사라진다. **캐시의 모든 키에는 TTL을 거는 것이 원칙**이다. 무효화를 놓쳐도 TTL이 지나면 결국 DB 값으로 돌아오는 안전장치가 된다.

```bash
SET product:42 '{"name":"키보드"}' EX 600   # 600초
EXPIRE product:42 300                        # 기존 키에 TTL 설정
TTL product:42                               # 남은 시간 (-1: TTL 없음, -2: 키 없음)
PERSIST product:42                           # TTL 제거
```

`SET`으로 값을 덮어쓰면 **기존 TTL이 사라진다** (`KEEPTTL` 옵션으로 유지 가능). `INCR`, `HSET`처럼 값을 수정하는 명령은 TTL을 유지한다.

### 만료된 키는 언제 지워지나

| 방식 | 동작 |
| --- | --- |
| Passive (Lazy) | 키에 접근할 때 만료됐으면 그때 삭제하고 없다고 응답 |
| Active | 주기적으로(기본 `hz 10`, 초당 10번) TTL 있는 키 일부를 샘플링해서 만료된 것을 삭제. 만료 비율이 높으면 반복 |

그래서 TTL이 지난 키는 **조회 결과로는 즉시 사라진 것처럼 보이지만, 메모리에서는 조금 늦게 빠질 수 있다.**

### 메모리가 가득 차면: Eviction

`maxmemory`를 넘으면 `maxmemory-policy`에 따라 키를 내쫓는다.

| 정책 | 대상 | 동작 |
| --- | --- | --- |
| `noeviction` (기본값) | - | 쓰기 명령에 에러 반환 |
| `allkeys-lru` | 모든 키 | 가장 오래 안 쓴 키 삭제 |
| `allkeys-lfu` | 모든 키 | 가장 적게 쓴 키 삭제 (Redis 4.0+) |
| `volatile-lru` / `volatile-lfu` | TTL 있는 키 | LRU / LFU |
| `volatile-ttl` | TTL 있는 키 | 만료가 가장 임박한 키 |
| `allkeys-random` / `volatile-random` | - | 무작위 |

순수 캐시 서버라면 `allkeys-lru`나 `allkeys-lfu`가 적절하다. 기본값 `noeviction` 그대로 두면 메모리가 차는 순간 **캐시 쓰기가 전부 실패**한다. Lock·세션처럼 지워지면 안 되는 데이터와 캐시를 한 Redis에 섞으면 Eviction이 그것까지 지울 수 있으므로 **용도별로 Redis를 분리**하는 편이 안전하다.

Redis의 LRU는 정확한 LRU가 아니라 **샘플링 기반 근사 LRU**다 (`maxmemory-samples`, 기본 5개 중 가장 오래된 것). 모든 키에 연결 리스트를 유지하는 메모리 비용을 피하기 위해서다.

### TTL을 얼마로 잡을까

| 데이터 | TTL 예 | 근거 |
| --- | --- | --- |
| 설정값, 카테고리 | 1시간 ~ 1일 | 거의 안 바뀌고, 바뀌면 명시적으로 삭제 |
| 상품 상세 | 5 ~ 30분 | 수정 시 삭제 + TTL로 안전장치 |
| 랭킹, 통계 | 집계 주기와 동일 | 1분 주기 배치면 1~2분 |
| 외부 API 응답 | 상대 API의 갱신 주기 | 환율이면 수 분 |
| 존재하지 않는 값(null) | 30초 ~ 수 분 | Cache Penetration 방어 (6장) |

같은 시각에 대량으로 캐시를 채우면 같은 시각에 대량으로 만료된다. TTL에 **무작위 값(Jitter)** 을 더해 만료 시점을 흩어 놓는다.

```java
Duration ttl = Duration.ofMinutes(10).plusSeconds(ThreadLocalRandom.current().nextInt(0, 60));
redisTemplate.opsForValue().set(key, value, ttl);
```

## 5. Cache 전략

### 읽기 전략

| 전략 | 흐름 | 특징 |
| --- | --- | --- |
| **Cache Aside** (Lazy Loading) | 앱이 캐시 조회 → miss면 앱이 DB 조회 → 앱이 캐시에 저장 | 가장 흔함. 캐시가 죽어도 DB로 동작. 첫 요청은 항상 miss |
| Read Through | 앱은 캐시만 조회, miss면 **캐시(라이브러리)가** DB를 읽어 채움 | 앱 코드 단순. Caffeine `LoadingCache`, Spring `@Cacheable`이 이 형태에 가까움 |

### 쓰기 전략

| 전략 | 흐름 | 장점 | 단점 |
| --- | --- | --- | --- |
| Write Around | DB에만 쓰고 캐시는 삭제 (Cache Aside와 짝) | 단순, 안 읽히는 데이터는 캐시에 안 올라감 | 쓴 직후 읽으면 miss |
| Write Through | 캐시와 DB에 **동시에(동기)** 씀 | 캐시가 항상 최신 | 쓰기 지연 증가, 안 읽힐 데이터도 캐시에 쌓임 |
| Write Behind (Write Back) | 캐시에만 쓰고 DB에는 **나중에 모아서** 씀 | 쓰기 매우 빠름, DB 부하 감소 | 캐시 장애 시 **데이터 유실** |

Write Behind는 조회수·좋아요처럼 **조금 잃어도 되는 카운터**에 쓴다. Redis에서 `INCR`로 모으다가 스케줄러가 주기적으로 DB에 반영하는 방식이다.

### Cache Aside 직접 구현

```java
@Service
@RequiredArgsConstructor
public class ProductQueryService {

    private static final Duration TTL = Duration.ofMinutes(10);
    private final RedisTemplate<String, Object> redisTemplate;
    private final ProductRepository productRepository;

    public ProductResponse getProduct(Long id) {
        String key = "product:" + id;

        try {
            ProductResponse cached = (ProductResponse) redisTemplate.opsForValue().get(key);
            if (cached != null) return cached;                    // Hit
        } catch (RedisConnectionFailureException | QueryTimeoutException e) {
            log.warn("Redis 조회 실패, DB로 대체", e);              // 캐시 장애가 서비스 장애가 되지 않게
        }

        ProductResponse response = ProductResponse.from(          // Miss → DB
                productRepository.findById(id).orElseThrow());

        try {
            redisTemplate.opsForValue().set(key, response, TTL);
        } catch (DataAccessException e) {
            log.warn("Redis 저장 실패", e);
        }
        return response;
    }
}
```

캐시에는 **JPA Entity가 아니라 DTO**를 넣는다. Entity를 직렬화하면 Lazy 로딩 프록시(10주차) 때문에 `LazyInitializationException`이 나거나, 연관 엔티티가 줄줄이 직렬화된다.

### Spring Cache 추상화

```java
@Configuration
@EnableCaching
public class RedisCacheConfig {

    @Bean
    public RedisCacheManager redisCacheManager(RedisConnectionFactory factory) {
        RedisCacheConfiguration defaults = RedisCacheConfiguration.defaultCacheConfig()
                .entryTtl(Duration.ofMinutes(10))
                .disableCachingNullValues()
                .serializeKeysWith(RedisSerializationContext.SerializationPair
                        .fromSerializer(new StringRedisSerializer()))
                .serializeValuesWith(RedisSerializationContext.SerializationPair
                        .fromSerializer(new GenericJackson2JsonRedisSerializer()));

        return RedisCacheManager.builder(factory)
                .cacheDefaults(defaults)
                .withCacheConfiguration("ranking", defaults.entryTtl(Duration.ofMinutes(1)))
                .build();
    }
}
```

```java
@Service
public class ProductService {

    @Cacheable(cacheNames = "product", key = "#id")        // Redis 키: "product::42"
    @Transactional(readOnly = true)
    public ProductResponse getProduct(Long id) { ... }

    @CacheEvict(cacheNames = "product", key = "#id")       // 수정 시 삭제
    @Transactional
    public void updateProduct(Long id, ProductUpdateRequest request) { ... }

    @CachePut(cacheNames = "product", key = "#result.id")  // 항상 실행하고 결과로 캐시 갱신
    @Transactional
    public ProductResponse createProduct(ProductCreateRequest request) { ... }
}
```

| 애노테이션 | 동작 |
| --- | --- |
| `@Cacheable` | 캐시에 있으면 **메서드를 실행하지 않고** 반환, 없으면 실행 후 저장 |
| `@CachePut` | 항상 메서드를 실행하고 결과를 캐시에 저장 |
| `@CacheEvict` | 캐시 삭제. `allEntries = true`면 해당 캐시 전체, `beforeInvocation = true`면 실행 전 삭제 |
| `@Caching` | 여러 개를 묶음 |

`@Cacheable`도 `@Transactional`처럼 **AOP Proxy**(9주차)로 동작한다. 그래서 11주차와 똑같은 함정이 있다.

- 같은 클래스 안에서 `this.getProduct()`로 부르면 캐시가 적용되지 않는다.
- `private` 메서드에는 적용되지 않는다.
- 키를 생략하면 모든 파라미터로 `SimpleKey`를 만든다. 파라미터 객체에 `equals/hashCode`, 직렬화 형태가 안정적인지 확인한다.

## 6. Cache Invalidation

> "There are only two hard things in Computer Science: cache invalidation and naming things." — Phil Karlton

캐시는 원본의 **복사본**이라 원본이 바뀌면 복사본도 맞춰야 한다. 핵심 질문은 두 가지다.

1. 캐시를 **갱신**할까, **삭제**할까?
2. DB와 캐시 중 **무엇을 먼저** 바꿀까?

### 갱신보다 삭제

| 방식 | 문제 |
| --- | --- |
| 캐시 갱신 (`SET`) | 두 요청이 동시에 수정하면 DB 반영 순서와 캐시 반영 순서가 엇갈려 **오래된 값이 캐시에 남는다** |
| 캐시 삭제 (`DEL`) | 다음 조회가 DB에서 최신값을 읽어 채움. 순서가 엇갈려도 "없음"이라 안전 |

```
[갱신 방식의 경합]
요청 A: DB price=100 ──────────────────────── 캐시 SET 100
요청 B:            DB price=200 ── 캐시 SET 200
결과: DB=200, 캐시=100  ← TTL 만료까지 틀린 값
```

### 순서: DB 먼저 반영, 그다음 캐시 삭제

| 순서 | 문제 상황 | 발생 가능성 |
| --- | --- | --- |
| 캐시 삭제 → DB 수정 | 삭제와 DB 수정 **사이에** 들어온 조회가 옛 값을 DB에서 읽어 다시 캐시에 넣음 | 높음 (DB 수정이 느리므로 틈이 김) |
| DB 수정 → 캐시 삭제 | 캐시 miss인 조회가 옛 값을 읽은 뒤, 수정·삭제가 모두 끝나고 나서야 캐시에 저장 | 낮음 (조회가 쓰기보다 느려야 함) |

```
[DB 수정 → 캐시 삭제에서도 남는 경합]
조회 C: 캐시 miss → DB 읽기(옛 값) ─────────────────────── 캐시 SET 옛 값
수정 D:                  DB 수정 → 커밋 → 캐시 DEL
결과: 캐시에 옛 값  ← TTL이 마지막 안전장치
```

완전히 없애려면 **지연 이중 삭제**(삭제 후 수백 ms 뒤 한 번 더 삭제)나, DB 변경 로그(CDC, Debezium)를 구독해 삭제하는 방법이 있다. 대부분은 "DB 수정 → 삭제 + 짧은 TTL"로 충분하다.

### 트랜잭션 커밋 후에 삭제하기

`@Transactional` 메서드 안에서 캐시를 지우면 **커밋 전에** 삭제된다. 그 사이 다른 요청이 커밋 전 옛 값을 DB에서 읽어 캐시에 다시 넣을 수 있고, 트랜잭션이 롤백되면 괜히 지운 셈이 된다.

```java
@Service
@RequiredArgsConstructor
public class ProductCommandService {

    private final ProductRepository productRepository;
    private final ApplicationEventPublisher eventPublisher;

    @Transactional
    public void updatePrice(Long id, int price) {
        productRepository.findById(id).orElseThrow().changePrice(price);
        eventPublisher.publishEvent(new ProductChangedEvent(id));
    }
}

@Component
@RequiredArgsConstructor
class ProductCacheEvictor {

    private final RedisTemplate<String, Object> redisTemplate;

    @TransactionalEventListener(phase = TransactionPhase.AFTER_COMMIT)  // 11주차와 같은 패턴
    public void evict(ProductChangedEvent event) {
        redisTemplate.delete("product:" + event.productId());
    }
}
```

`@CacheEvict` + `@Transactional`을 같은 메서드에 붙이면 프록시 순서에 따라 삭제 시점이 달라진다. `RedisCacheManager.builder(factory).transactionAware()`를 켜면 캐시 쓰기/삭제를 **커밋 이후로 미룬다.**

### 캐시 장애 패턴

| 이름 | 상황 | 대응 |
| --- | --- | --- |
| **Cache Stampede** (Thundering Herd) | 인기 키가 만료되는 순간 수많은 요청이 동시에 miss → 모두 DB 조회 | Lock으로 한 요청만 DB 조회, 만료 전 미리 갱신(Refresh Ahead), 확률적 조기 갱신(PER) |
| **Cache Penetration** | **존재하지 않는** 데이터를 계속 조회 → 항상 miss → 항상 DB | null도 짧은 TTL로 캐시, Bloom Filter로 존재 여부 선검사, 입력값 검증 |
| **Cache Avalanche** | 대량의 키가 동시에 만료되거나 Redis가 죽음 → DB로 폭주 | TTL Jitter, Redis HA 구성, Local Cache 2단, Circuit Breaker로 DB 보호 |
| **Hot Key** | 키 하나에 요청 집중 → Cluster에서도 노드 하나만 과부하 | Local Cache 앞단, 키 복제(`product:42:{0..9}` 중 랜덤 조회) |
| **Big Key** | 값 하나가 수 MB, 컬렉션 원소가 수백만 | 쪼개서 저장, `UNLINK`, `redis-cli --bigkeys`로 점검 |

### Cache Stampede 방어 예시

```java
public ProductResponse getProduct(Long id) {
    String key = "product:" + id;
    ProductResponse cached = (ProductResponse) redisTemplate.opsForValue().get(key);
    if (cached != null) return cached;

    RLock lock = redissonClient.getLock("lock:" + key);
    try {
        if (lock.tryLock(1, 3, TimeUnit.SECONDS)) {          // 한 요청만 DB 조회
            try {
                cached = (ProductResponse) redisTemplate.opsForValue().get(key); // Double Check
                if (cached != null) return cached;

                ProductResponse response = ProductResponse.from(productRepository.findById(id).orElseThrow());
                redisTemplate.opsForValue().set(key, response, TTL);
                return response;
            } finally {
                lock.unlock();
            }
        }
    } catch (InterruptedException e) {
        Thread.currentThread().interrupt();
    }
    return ProductResponse.from(productRepository.findById(id).orElseThrow()); // Lock 못 얻으면 DB 직접 (또는 잠시 후 캐시 재조회)
}
```

`@Cacheable(sync = true)`도 같은 키의 동시 miss를 한 번만 계산하게 해 주지만, **JVM 하나 안에서만** 막는다. 서버가 10대면 최대 10번 DB를 조회한다. 대부분은 그 정도로 충분하다.

## 7. Distributed Lock

11주차에서 본 `synchronized`, `ReentrantLock`은 **한 JVM 안에서만** 동작한다. 서버가 여러 대면 서버 A와 B가 동시에 임계 영역에 들어간다.

| 방법 | 특징 |
| --- | --- |
| DB 비관적 Lock (`SELECT ... FOR UPDATE`) | 이미 DB를 쓰면 추가 인프라 없음. Lock 대상 행이 있어야 함 |
| MySQL Named Lock (`GET_LOCK`) | 행 없이 이름으로 Lock. 커넥션에 묶여 있어 커넥션 관리 주의 |
| **Redis Lock** | 빠르고 TTL로 자동 해제. Redis 장애·Failover 시 안전성 한계 |
| ZooKeeper / etcd | 합의 알고리즘 기반으로 가장 안전. 운영 부담 큼 |

### 직접 구현: `SET NX PX`

```bash
SET lock:coupon:1 "uuid-서버A-스레드3" NX PX 3000
# NX: 키가 없을 때만 설정 → 성공하면 OK, 이미 있으면 nil
# PX: 3000ms 뒤 자동 만료 → 서버가 죽어도 Lock이 영원히 남지 않음
```

`SETNX` 후 `EXPIRE`를 따로 보내면 두 명령 사이에 서버가 죽었을 때 **TTL 없는 Lock**이 남는다. 반드시 한 명령(`SET ... NX PX`)으로 건다.

해제할 때는 **내가 건 Lock인지 확인하고** 지워야 한다.

```
1. A가 Lock 획득 (TTL 3초)
2. A가 GC Pause 등으로 4초 멈춤 → Lock 만료
3. B가 Lock 획득
4. A가 깨어나 DEL → B의 Lock을 지워 버림
5. C도 Lock 획득 → B와 C가 동시에 실행
```

"GET으로 확인 → DEL"을 두 명령으로 보내면 그 사이에도 같은 일이 생기므로 **Lua 스크립트로 원자적으로** 처리한다.

```lua
if redis.call("GET", KEYS[1]) == ARGV[1] then
    return redis.call("DEL", KEYS[1])
else
    return 0
end
```

```java
@Component
@RequiredArgsConstructor
public class SimpleRedisLock {

    private static final RedisScript<Long> UNLOCK = new DefaultRedisScript<>(
            "if redis.call('GET', KEYS[1]) == ARGV[1] then return redis.call('DEL', KEYS[1]) else return 0 end",
            Long.class);
    private final StringRedisTemplate redisTemplate;

    public Optional<String> tryLock(String key, Duration ttl) {
        String token = UUID.randomUUID().toString();
        Boolean ok = redisTemplate.opsForValue().setIfAbsent(key, token, ttl); // SET NX PX
        return Boolean.TRUE.equals(ok) ? Optional.of(token) : Optional.empty();
    }

    public void unlock(String key, String token) {
        redisTemplate.execute(UNLOCK, List.of(key), token);
    }
}
```

이 방식은 Lock을 못 얻으면 직접 재시도(Spin Lock)해야 해서 Redis에 부하가 생기고, 작업이 TTL보다 길어지면 Lock이 먼저 풀린다.

### Redisson

```java
@Component
@RequiredArgsConstructor
public class CouponFacade {

    private final RedissonClient redissonClient;
    private final CouponService couponService;

    public void issue(Long couponId, Long userId) {
        RLock lock = redissonClient.getLock("lock:coupon:" + couponId);
        try {
            boolean acquired = lock.tryLock(3, TimeUnit.SECONDS); // 최대 3초 대기, leaseTime 생략 → Watchdog
            if (!acquired) throw new IllegalStateException("요청이 많습니다. 잠시 후 다시 시도해 주세요.");
            couponService.issue(couponId, userId);                // @Transactional은 이 안에서 시작·커밋
        } catch (InterruptedException e) {
            Thread.currentThread().interrupt();
            throw new IllegalStateException(e);
        } finally {
            if (lock.isHeldByCurrentThread()) lock.unlock();
        }
    }
}
```

| 기능 | 설명 |
| --- | --- |
| Pub/Sub 대기 | Lock 해제 시 채널로 알림을 받아 대기 중인 클라이언트가 깨어남 → Spin Lock보다 Redis 부하가 적음 |
| Watchdog | `leaseTime`을 지정하지 않으면 기본 30초 TTL로 잡고, 작업이 끝날 때까지 **주기적으로 TTL을 연장** (10초마다). 서버가 죽으면 연장이 멈춰 30초 뒤 해제 |
| `tryLock(wait, lease, unit)` | `leaseTime`을 지정하면 Watchdog이 **동작하지 않고** 그 시간 뒤 무조건 해제 |
| 재진입 | 같은 스레드가 다시 획득 가능 (Hash에 스레드 ID와 횟수 저장) |
| `FairLock`, `ReadWriteLock`, `Semaphore` | 다양한 동기화 도구 제공 |

### Lock과 트랜잭션의 순서

```
[잘못된 순서]  @Transactional 안에서 Lock
트랜잭션 시작 → Lock 획득 → 재고 감소 → Lock 해제 → (아직 커밋 전!)
                                         ↑ 다른 요청이 Lock 얻고 커밋 전 재고를 읽음 → Lost Update

[올바른 순서]  Lock이 트랜잭션을 감싼다
Lock 획득 → 트랜잭션 시작 → 재고 감소 → 커밋 → Lock 해제
```

11주차에서 `synchronized` + `@Transactional`이 실패한 이유와 같다. 그래서 위 예시처럼 **Facade에서 Lock을 걸고, 안쪽 Service에서 트랜잭션을 연다.**

### Redis Lock의 한계

| 문제 | 상황 |
| --- | --- |
| Failover 시 Lock 유실 | Master에 Lock 기록 → **비동기 복제** 전에 Master 장애 → 승격된 Replica에는 Lock이 없음 → 다른 클라이언트가 획득 |
| 긴 Pause | GC Pause, 네트워크 지연으로 TTL이 지난 뒤에도 클라이언트는 자기가 Lock을 가졌다고 믿음 |
| 시계 의존 | TTL은 Redis 서버 시계 기준. 시계가 튀면 예상보다 빨리 만료 |

**RedLock**은 독립된 Redis 5대 중 과반(3대 이상)에서 Lock을 얻어야 성공으로 보는 알고리즘이다. Failover 문제는 줄지만, Martin Kleppmann은 "GC Pause·시계 문제는 여전히 해결하지 못한다"고 비판했고 Redis 개발자 antirez가 반론하는 논쟁이 있었다.

결론적으로 Redis Lock은 **효율성을 위한 Lock**(중복 작업을 대부분 막으면 되는 경우)에 적합하다. **정확성이 필수**인 경우에는 Lock만 믿지 않고 마지막 방어선을 둔다.

- DB **UNIQUE 제약조건** (쿠폰 중복 발급)
- **원자적 UPDATE**: `UPDATE stock SET qty = qty - 1 WHERE id = ? AND qty > 0`
- **Fencing Token**: Lock을 얻을 때마다 증가하는 번호를 받아 저장소에 함께 보내고, 저장소가 더 작은 번호의 쓰기를 거부 (11주차 `@Version`과 같은 원리)

### Lock 없이 Redis만으로 처리하기

선착순 쿠폰처럼 "개수만 세면 되는" 경우는 Lock 없이 Redis의 원자적 명령으로 끝낼 수 있다.

```java
public boolean tryIssue(Long couponId, Long userId) {
    Long added = redisTemplate.opsForSet().add("coupon:" + couponId + ":users", userId.toString());
    if (added == null || added == 0) return false;                       // 이미 발급받은 사용자

    Long count = redisTemplate.opsForValue().increment("coupon:" + couponId + ":count");
    if (count != null && count > 100) {
        redisTemplate.opsForSet().remove("coupon:" + couponId + ":users", userId.toString());
        return false;                                                    // 수량 초과
    }
    eventPublisher.publish(new CouponIssueRequested(couponId, userId));  // 실제 DB 저장은 비동기로
    return true;
}
```

여러 명령을 하나로 묶어야 한다면 Lua 스크립트(`EVAL`)로 한 번에 실행한다. Redis가 단일 스레드라 스크립트 실행 중에는 다른 명령이 끼어들지 않는다. 대신 스크립트가 길면 그동안 Redis 전체가 멈추므로 짧게 유지한다.

## 8. Backend에서 확인할 점

- 캐시를 넣기 전에 **무엇이 느린지 먼저 측정**한다. 쿼리 하나가 느린 거라면 6주차의 인덱스·EXPLAIN이 먼저다. 캐시를 넣었다면 Hit Ratio를 본다.
- 캐시의 모든 키에 **TTL**을 건다. 무효화 로직에 버그가 있어도 TTL이 지나면 복구된다. 대량으로 채우는 키에는 Jitter를 더한다.
- 캐시 서버용 Redis는 `maxmemory`와 `allkeys-lru`/`allkeys-lfu`를 설정한다. 기본값 `noeviction`은 메모리가 차면 쓰기가 실패한다. Lock·세션용 Redis와는 분리한다.
- 데이터 변경 시 캐시는 **갱신보다 삭제**, **DB 커밋 후 삭제**(`@TransactionalEventListener(AFTER_COMMIT)` 또는 `transactionAware()`).
- 캐시에는 Entity가 아니라 **DTO**를 JSON으로 넣는다. DTO 구조가 바뀌는 배포에서는 키 버전을 올린다.
- Redis 장애가 서비스 장애가 되지 않게 **타임아웃**을 짧게 잡고, 조회 실패 시 DB로 대체한다. 동시에 DB가 그 트래픽을 감당할 수 있는지(Avalanche) 확인한다.
- 운영 Redis에서 `KEYS`, 큰 키 `DEL`, `HGETALL` 같은 O(N) 명령을 쓰지 않는다. `SCAN`, `UNLINK`를 쓰고 `SLOWLOG GET`으로 느린 명령을 점검한다.
- `@Cacheable`은 AOP Proxy라 내부 호출·private 메서드에서 동작하지 않는다. `sync = true`는 JVM 하나 안에서만 Stampede를 막는다.
- 분산 Lock은 `SET NX PX` + **토큰 확인 후 Lua로 해제**가 기본이다. 실무에서는 Redisson을 쓰고, `leaseTime`을 지정하면 Watchdog이 꺼진다는 점을 기억한다.
- Lock은 **트랜잭션 바깥**에서 건다. 그리고 Redis Lock은 Failover·GC Pause에서 깨질 수 있으므로 UNIQUE 제약조건·원자적 UPDATE 같은 DB 레벨 방어선을 함께 둔다.
- Redis 동작은 Testcontainers로 실제 Redis를 띄워 테스트한다.

```java
@SpringBootTest
@Testcontainers
class CouponFacadeTest {

    @Container
    @ServiceConnection(name = "redis")   // GenericContainer는 이름을 지정해야 Redis 연결 정보로 인식
    static GenericContainer<?> redis = new GenericContainer<>("redis:7-alpine").withExposedPorts(6379);

    @Autowired CouponFacade couponFacade;
    @Autowired CouponIssueRepository couponIssueRepository;

    @Test
    void 서버_여러_대처럼_동시에_요청해도_100장만_발급된다() throws InterruptedException {
        int requests = 1000;
        ExecutorService executor = Executors.newFixedThreadPool(32);
        CountDownLatch latch = new CountDownLatch(requests);

        for (long userId = 1; userId <= requests; userId++) {
            long uid = userId;
            executor.submit(() -> {
                try { couponFacade.issue(1L, uid); }
                catch (Exception ignored) { }          // 수량 초과·Lock 대기 초과는 실패로 처리
                finally { latch.countDown(); }
            });
        }
        latch.await();

        assertThat(couponIssueRepository.countByCouponId(1L)).isEqualTo(100); // 쿠폰 수량 100
    }
}
```
