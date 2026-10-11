# 5주차 - HTTP / HTTPS (Backend)

## 1. Keep-Alive

HTTP는 TCP 위에서 동작하므로, 연결을 요청마다 새로 맺으면 매번 3-way handshake(HTTPS라면 TLS handshake까지) 비용이 든다. Keep-Alive(Persistent Connection)는 하나의 TCP 연결로 여러 요청/응답을 주고받아 이 비용을 줄인다.

| 버전 | 연결 방식 |
| --- | --- |
| HTTP/1.0 | 기본은 요청마다 연결 종료. `Connection: keep-alive`를 보내야 재사용 |
| HTTP/1.1 | 기본이 연결 재사용. 끊으려면 `Connection: close` |
| HTTP/2 | 연결 하나에서 여러 요청을 동시에 처리(Multiplexing). `Connection` 헤더를 사용하지 않음 |

```http
HTTP/1.1 200 OK
Connection: keep-alive
Keep-Alive: timeout=5, max=100
```

`timeout`은 유휴 연결을 유지할 시간(초), `max`는 이 연결로 처리할 최대 요청 수다.

HTTP/1.1은 연결을 재사용하더라도 한 연결에서 **요청을 순서대로** 처리한다. 앞 요청의 응답이 늦으면 뒤 요청도 기다리는 Head-of-Line Blocking이 생기고, 그래서 브라우저·HTTP Client는 호스트당 여러 연결을 동시에 연다. HTTP/2는 Stream 단위 Multiplexing으로 이 문제를 줄였다.

HTTP Keep-Alive와 TCP Keepalive(`SO_KEEPALIVE`)는 다른 개념이다. 전자는 **연결 재사용**이고, 후자는 유휴 연결에 probe 패킷을 보내 **상대가 살아 있는지 확인**하는 기능이다.

### 서버 설정

| Tomcat 설정 (Spring Boot) | 의미 |
| --- | --- |
| `server.tomcat.keep-alive-timeout` | 다음 요청을 기다리며 유휴 연결을 유지하는 시간 (미설정 시 `connection-timeout` 값) |
| `server.tomcat.max-keep-alive-requests` | 연결 하나로 처리할 최대 요청 수 (기본 100) |

### Timeout 불일치 문제

연결은 양쪽이 각자 유휴 Timeout을 갖고 있어서, **요청을 받는 쪽이 먼저 끊으면** 보내는 쪽은 이미 닫힌 연결로 요청을 보내게 된다.

- **LB → 서버**: 서버의 keep-alive timeout이 LB의 idle timeout보다 짧으면, LB가 재사용하려던 연결이 이미 닫혀 있어 간헐적으로 `502 Bad Gateway`가 발생한다. 서버 쪽 Timeout을 LB보다 길게 잡는다.
- **서버 → 외부 API**: HTTP Client Pool의 유휴 연결 유지 시간이 상대 서버의 keep-alive timeout보다 길면 `Connection reset` 계열 오류가 난다. Client 쪽 유휴 시간을 더 짧게 잡는다.

정리하면 **요청을 보내는 쪽의 유휴 Timeout < 받는 쪽의 유휴 Timeout**이 되어야 한다.

## 2. REST API

REST는 자원(Resource)을 URI로 식별하고, 자원에 대한 행위를 HTTP Method로 표현하는 아키텍처 스타일이다.

| 제약 조건 | 의미 |
| --- | --- |
| Client-Server | 클라이언트와 서버의 역할 분리 |
| Stateless | 서버가 요청 간 상태를 저장하지 않음. 요청에 필요한 정보가 모두 담겨야 함 |
| Cacheable | 응답이 캐시 가능 여부를 명시 |
| Uniform Interface | URI로 자원 식별, 표현을 통한 조작, 자기 서술적 메시지, HATEOAS |
| Layered System | 클라이언트는 중간 계층(LB, Proxy)의 존재를 몰라도 됨 |
| Code on Demand (선택) | 서버가 실행 가능한 코드를 내려줄 수 있음 |

Stateless하기 때문에 어떤 서버가 요청을 받아도 처리할 수 있고, 서버를 수평 확장하기 쉽다.

### Method

| Method | 용도 | 안전(Safe) | 멱등(Idempotent) |
| --- | --- | --- | --- |
| GET | 조회 | O | O |
| POST | 생성, 그 외 처리 | X | X |
| PUT | 전체 교체 (없으면 생성) | X | O |
| PATCH | 부분 수정 | X | X (보장되지 않음) |
| DELETE | 삭제 | X | O |

- **안전**: 호출해도 자원 상태가 바뀌지 않는다.
- **멱등**: 같은 요청을 여러 번 보내도 **서버 상태**가 한 번 보낸 것과 같다. 응답 코드가 같아야 한다는 뜻은 아니다(DELETE를 두 번 보내면 두 번째는 `404`일 수 있다).

멱등성은 **재시도 가능 여부**를 판단하는 기준이 된다.

### URI 설계

- 자원은 명사·복수형으로, 행위는 Method로 표현한다. `POST /users` (O), `POST /createUser` (X)
- 계층 관계는 경로로 표현한다. `GET /users/1/orders`
- 필터·정렬·페이지는 Query String으로 표현한다. `GET /orders?status=PAID&page=0&size=20`

### Status Code

| 코드 | 의미 | 사용 예 |
| --- | --- | --- |
| 200 OK | 성공 | 조회, 수정 |
| 201 Created | 생성됨 | POST 성공. `Location` 헤더에 새 자원의 URI |
| 204 No Content | 성공, 본문 없음 | DELETE 성공 |
| 400 Bad Request | 요청 형식·값 오류 | 검증 실패 |
| 401 Unauthorized | 인증되지 않음 | 토큰 없음·만료 |
| 403 Forbidden | 인증됐지만 권한 없음 | 타인의 자원 접근 |
| 404 Not Found | 자원 없음 | 존재하지 않는 ID |
| 409 Conflict | 현재 자원 상태와 충돌 | 중복 가입 |
| 500 Internal Server Error | 서버 오류 | 처리하지 못한 예외 |

```java
@RestController
@RequestMapping("/users")
public class UserController {

    @PostMapping
    public ResponseEntity<UserResponse> create(@RequestBody @Valid UserCreateRequest request) {
        UserResponse user = userService.create(request);
        return ResponseEntity.created(URI.create("/users/" + user.id())).body(user); // 201 + Location
    }

    @DeleteMapping("/{id}")
    public ResponseEntity<Void> delete(@PathVariable Long id) {
        userService.delete(id);
        return ResponseEntity.noContent().build(); // 204
    }
}
```

## 3. Connection / Timeout

4주차에서 TCP 수준의 Connection/Read Timeout을 정리했다. 여기서는 HTTP 요청이 지나가는 구간별로 Timeout이 어떻게 드러나는지를 본다.

```
Client → LB/Gateway → Server → DB / 외부 API
```

| 코드 | 의미 | 주로 의심할 곳 |
| --- | --- | --- |
| 408 Request Timeout | 서버가 클라이언트의 요청 전송을 기다리다 시간 초과 | 클라이언트·네트워크 |
| 502 Bad Gateway | Gateway가 뒤 서버에서 잘못된 응답을 받음 (연결 끊김 포함) | 서버 다운, keep-alive timeout 불일치 |
| 503 Service Unavailable | 서버가 일시적으로 요청을 처리할 수 없음 | 과부하, 배포 중 |
| 504 Gateway Timeout | Gateway가 뒤 서버의 응답을 시간 안에 받지 못함 | 서버의 느린 처리, DB·외부 API 지연 |

### 서버가 요청을 받을 때

```yaml
server:
  tomcat:
    connection-timeout: 20s      # 연결 수락 후 요청 데이터를 기다리는 시간
    keep-alive-timeout: 65s      # LB의 idle timeout(예: 60s)보다 길게
    max-keep-alive-requests: 100
```

### 서버가 외부 API를 호출할 때

```java
HttpClient client = HttpClient.newBuilder()
        .connectTimeout(Duration.ofSeconds(3))   // 연결 맺기까지
        .build();

HttpRequest request = HttpRequest.newBuilder(URI.create("https://api.example.com/orders"))
        .timeout(Duration.ofSeconds(5))          // 응답을 받기까지
        .build();
```

HTTPS는 TCP 연결 뒤에 TLS handshake가 추가로 필요해 새 연결 비용이 더 크다. 그래서 Connection Pool로 연결을 재사용하는 효과도 더 크다.

### Timeout과 재시도

Timeout이 났다고 해서 **요청이 처리되지 않은 것은 아니다.** 서버는 처리를 끝냈는데 응답만 늦었을 수 있다.

- GET, PUT, DELETE처럼 멱등한 요청은 재시도해도 안전하다.
- POST(결제, 주문 등)는 재시도하면 중복 처리될 수 있다. `Idempotency-Key` 같은 요청 식별자를 헤더로 받아, 서버가 같은 키의 요청을 한 번만 처리하도록 한다.
- 재시도는 횟수와 간격(backoff)을 제한한다. 장애 중인 서버에 재시도가 몰리면 복구가 더 늦어진다.

## 4. Backend에서 확인할 점

- 간헐적인 `502`는 LB와 서버의 keep-alive timeout 관계를 먼저 확인한다.
- `504`는 Gateway의 Timeout 값보다 서버가 느린 원인(DB 쿼리, 외부 API 호출)을 먼저 찾는다.
- 외부 API를 호출하는 HTTP Client는 Timeout과 Connection Pool 설정(최대 연결 수, 유휴 연결 유지 시간)을 명시한다.
- 상태를 바꾸는 API는 재시도되어도 안전한지(멱등성) 설계 단계에서 정한다.
- Status Code는 클라이언트가 재시도·오류 처리를 판단하는 근거이므로, 모든 오류를 `200`이나 `500`으로 뭉개지 않는다.
