# 4주차 - Network (Backend)

## 1. Socket

Socket은 프로세스가 네트워크로 데이터를 주고받기 위해 OS가 제공하는 통신 끝점(endpoint)이다. 애플리케이션은 Socket에 읽고 쓰기만 하고, 실제 TCP/UDP 처리는 OS 커널이 담당한다. TCP 연결 하나는 `(출발지 IP, 출발지 Port, 목적지 IP, 목적지 Port, 프로토콜)` 5-tuple로 구분된다.

### TCP 서버/클라이언트 흐름

| 단계 | 서버 | 클라이언트 |
| --- | --- | --- |
| 1 | `socket()` 생성 | `socket()` 생성 |
| 2 | `bind()`로 IP·Port 지정 | - |
| 3 | `listen()`으로 연결 대기 | - |
| 4 | `accept()`로 연결 수락 | `connect()`로 연결 요청 (3-way handshake) |
| 5 | `read()` / `write()` | `read()` / `write()` |
| 6 | `close()` | `close()` (4-way handshake) |

`accept()`는 연결마다 **새로운 Socket**을 반환한다. 처음 만든 listening socket은 계속 연결 요청만 받고, 실제 데이터 송수신은 `accept()`가 돌려준 socket으로 한다. 그래서 서버는 같은 Port(예: 8080)로 여러 클라이언트와 동시에 통신할 수 있다.

```java
try (ServerSocket serverSocket = new ServerSocket(8080)) {
    while (true) {
        Socket client = serverSocket.accept(); // 연결마다 새 Socket
        executor.submit(() -> handle(client));
    }
}
```

### Backlog 큐

커널은 연결 요청을 두 단계 큐로 관리한다.

- **SYN 큐**: SYN을 받고 handshake가 진행 중인 연결
- **Accept 큐**: handshake가 끝나 `accept()`를 기다리는 연결

애플리케이션이 `accept()`를 늦게 호출하면 Accept 큐가 가득 차고, 새 연결 요청이 버려지거나 거절된다. `ServerSocket(port, backlog)`의 backlog와 OS 설정(`net.core.somaxconn`)이 이 큐 크기에 영향을 준다.

## 2. Server Connection 관리

### Blocking I/O와 Non-blocking I/O

| 방식 | 구조 | 특징 |
| --- | --- | --- |
| BIO (Thread per Connection) | 연결 하나당 스레드 하나 | 구현이 단순하지만 연결 수만큼 스레드가 필요 |
| NIO (Selector) | 소수 스레드가 여러 채널의 이벤트를 감시 | 대량 연결에 유리, 구현이 복잡 |

Tomcat은 NIO Connector를 기본으로 사용한다. Poller 스레드가 Selector로 여러 연결의 I/O 이벤트를 감시하다가, 읽을 데이터가 준비된 요청만 Worker 스레드 풀에 넘긴다. 따라서 **연결 수 ≠ 요청 처리 스레드 수**이다.

| Tomcat 설정 (Spring Boot) | 의미 |
| --- | --- |
| `server.tomcat.threads.max` | 요청을 처리하는 Worker 스레드 최대 수 (기본 200) |
| `server.tomcat.max-connections` | 동시에 유지할 수 있는 최대 연결 수 (기본 8192) |
| `server.tomcat.accept-count` | max-connections 초과 시 OS 대기 큐 크기 (기본 100) |

### Keep-Alive

HTTP/1.1은 기본적으로 연결을 재사용한다(Keep-Alive). 요청마다 TCP handshake를 반복하지 않아 지연이 줄지만, 유휴 연결도 서버 자원(파일 디스크립터, 메모리)을 점유한다. 그래서 서버는 `keep-alive-timeout`, 최대 요청 수 등으로 유휴 연결을 정리한다.

### Connection Pool

서버가 **클라이언트 입장**이 되는 경우(DB, 외부 API 호출)에는 연결을 매번 새로 만들지 않고 Pool에서 재사용한다.

- **DB**: HikariCP가 Spring Boot 기본 Connection Pool이다. `maximum-pool-size`보다 동시 요청이 많으면 스레드는 연결이 반납될 때까지 대기한다.
- **HTTP Client**: RestTemplate/WebClient도 내부 Connection Pool 설정이 가능하며, 기본 설정은 호스트별 연결 수가 작을 수 있어 확인이 필요하다.

Pool 크기는 무조건 크게 잡는다고 좋은 것이 아니다. DB는 동시에 처리 가능한 연결 수에 한계가 있으므로, Worker 스레드 수·DB 연결 수·실제 쿼리 시간을 함께 보고 정한다.

### TIME_WAIT

TCP 연결을 **먼저 끊는 쪽**은 4-way handshake 이후 `TIME_WAIT` 상태로 일정 시간(보통 2MSL) 머문다. 늦게 도착한 패킷이 새 연결에 섞이지 않게 하기 위함이다. 서버가 짧은 연결을 대량으로 맺고 끊으면 `TIME_WAIT` 소켓이 쌓여 로컬 Port가 고갈될 수 있다. 연결 재사용(Keep-Alive, Connection Pool)이 가장 기본적인 해결책이다.

## 3. Connection Timeout

Timeout이 없으면 상대가 응답하지 않을 때 스레드가 무한히 대기하고, 이런 스레드가 쌓이면 스레드 풀·Connection Pool이 고갈되어 서비스 전체가 멈춘다. 그래서 외부 호출에는 반드시 Timeout을 설정한다.

| 종류 | 의미 | 초과 시 (Java) |
| --- | --- | --- |
| Connection Timeout | TCP 연결(3-way handshake)이 맺어질 때까지의 최대 대기 시간 | `ConnectException` / `SocketTimeoutException` |
| Read (Socket) Timeout | 연결 후 데이터 패킷 사이의 최대 대기 시간 | `SocketTimeoutException: Read timed out` |
| Connection Request Timeout | Pool에서 연결을 빌려오기까지의 최대 대기 시간 | Pool 구현별 예외 (HikariCP: `SQLTransientConnectionException`) |

Read Timeout은 **전체 응답 시간 제한이 아니라** 패킷 사이 간격의 제한이다. 데이터가 조금씩 계속 오면 전체 응답이 오래 걸려도 Timeout이 나지 않을 수 있으므로, 전체 요청 시간 제한이 필요하면 별도로 설정한다.

```java
Socket socket = new Socket();
socket.connect(new InetSocketAddress("api.example.com", 443), 3000); // Connection Timeout 3초
socket.setSoTimeout(5000);                                          // Read Timeout 5초
```

```yaml
spring:
  datasource:
    hikari:
      connection-timeout: 3000   # Pool에서 연결을 얻기까지 대기 (ms)
      maximum-pool-size: 10
```

### Timeout 값 정하기

- 연결은 보통 빠르게 맺어지므로 Connection Timeout은 짧게(수백 ms~수 초) 잡는다.
- Read Timeout은 상대 API의 실제 응답 시간(p99 등)을 측정해 그보다 약간 여유 있게 잡는다.
- 호출 체인이 있다면 하위 호출의 Timeout 합이 상위 요청의 Timeout보다 작아야 한다.
- Timeout 후 재시도할 때는 멱등성을 확인하고, 재시도 횟수와 간격(backoff)을 제한한다.

## 4. Backend에서 확인할 점

- 외부 API·DB 호출에는 Connection/Read Timeout을 명시적으로 설정한다. 라이브러리 기본값이 무한대인 경우가 있다.
- 응답 지연 장애 시 Worker 스레드, Connection Pool 대기 수, `TIME_WAIT`/`CLOSE_WAIT` 소켓 수를 함께 확인한다.
- `CLOSE_WAIT`이 계속 쌓이면 애플리케이션이 상대가 끊은 연결을 `close()`하지 않고 있다는 신호다.
- 스레드 풀·Connection Pool 크기는 측정 결과를 바탕으로 조정한다.
