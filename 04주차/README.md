# 4주차 - Network

## 공통 키워드

### OSI 7 Layer/TCP-IP

네트워크 통신 과정을 **계층별로 나눠 역할을 분리**한 모델. 계층을 나누면 특정 계층에 문제가 생겼을 때 그 계층만 보면 되고, 각 계층을 독립적으로 교체·발전시킬 수 있다.

| OSI 7 Layer | 역할 | 대표 프로토콜/장비 | 데이터 단위 | TCP/IP 4계층 |
|---|---|---|---|---|
| 7. Application (응용 계층) | 사용자 응용 서비스 제공 | HTTP, FTP, SMTP, DNS | Data | Application (응용) |
| 6. Presentation (표현 계층) | 인코딩, 암호화, 압축 | TLS(일부), JPEG, ASCII | Data | Application (응용) |
| 5. Session (세션 계층) | 세션 연결 수립/유지/종료 | RPC, NetBIOS | Data | Application (응용) |
| 4. Transport (전송 계층) | 프로세스 간 신뢰성/흐름 제어 | TCP, UDP | Segment / Datagram | Transport (전송) |
| 3. Network (네트워크 계층) | 논리 주소(IP) 기반 라우팅 | IP, ICMP, 라우터 | Packet | Internet (인터넷) |
| 2. Data Link (데이터 링크 계층) | 같은 네트워크 내 노드 간 전달(MAC) | Ethernet, 스위치 | Frame | Network Access (네트워크 접근) |
| 1. Physical (물리 계층) | 전기 신호/비트 전송 | 케이블, 허브 | Bit | Network Access (네트워크 접근) |

> 💡 **암기법: 물데네전세표응** (1계층 → 7계층)
> **물**리 → **데**이터 링크 → **네**트워크 → **전**송 → **세**션 → **표**현 → **응**용
> 거꾸로(7 → 1)는 **응표세전네데물**

- **캡슐화**: 송신 측은 상위 → 하위로 내려가며 각 계층 헤더를 붙임
- **역캡슐화**: 수신 측은 하위 → 상위로 올라가며 헤더를 제거
- **OSI vs TCP/IP**: OSI는 이론적 참조 모델, TCP/IP는 실제 인터넷에서 쓰이는 실용 모델
- **스위치 vs 라우터**: 스위치는 L2에서 MAC 주소로 같은 네트워크 내 전달, 라우터는 L3에서 IP 주소로 다른 네트워크 간 경로 결정

### IP

- **IP (Internet Protocol)**: 패킷을 목적지 IP 주소까지 전달하는 L3 프로토콜
- **비연결성 / 비신뢰성**: 대상이 없어도 패킷을 보내고, 전달·순서를 보장하지 않음. 같은 IP 내 여러 애플리케이션도 구분 못 함 → TCP/UDP와 포트가 보완
- **IPv4**: 32비트, 약 43억 개 → 주소 고갈
- **IPv6**: 128비트, 사실상 무한, 헤더 단순화
- **공인 IP vs 사설 IP**: 사설 IP(10.x, 172.16~31.x, 192.168.x)는 외부에서 직접 접근 불가
- **NAT**: 사설 IP ↔ 공인 IP 변환. IPv4 주소 부족 해결 + 내부망 구조를 숨기는 보안 효과
- **서브넷 마스크 / CIDR**: IP를 네트워크 부분과 호스트 부분으로 구분 (예: `192.168.0.0/24`)
- **IP vs MAC**: IP는 논리 주소(바뀔 수 있음, 네트워크 간), MAC은 물리 주소(NIC 고유, 같은 네트워크 내). IP → MAC 변환은 **ARP**

### TCP/UDP

| 구분 | TCP | UDP |
|---|---|---|
| 연결 | 연결 지향 (handshake) | 비연결 |
| 신뢰성 | 보장 (ACK, 재전송) | 보장 X |
| 순서 | 보장 (Sequence Number) | 보장 X |
| 흐름/혼잡 제어 | O | X |
| 속도 | 상대적으로 느림 | 빠름 |
| 헤더 | 20바이트~ | 8바이트 |
| 전송 방식 | 1:1 | 1:1, 1:N, N:N |
| 사용 예 | HTTP/1.1·2, 이메일, 파일 전송 | DNS, 스트리밍, 게임, VoIP, HTTP/3(QUIC) |

- **신뢰성 보장 방법**: 순서 번호(SEQ), 확인 응답(ACK), 타임아웃 재전송, 체크섬
- **흐름 제어 (Flow Control)**: **송신자-수신자 간** 문제. 수신자 버퍼가 넘치지 않도록 송신 속도 조절 → 슬라이딩 윈도우
- **혼잡 제어 (Congestion Control)**: **네트워크 전체** 문제. 혼잡을 줄이기 위해 송신량 조절 → Slow Start, AIMD, Fast Retransmit/Recovery
- **UDP 위의 신뢰성**: 애플리케이션 레벨에서 ACK·재전송·순서 번호를 직접 구현 (예: QUIC)
- **DNS가 UDP를 쓰는 이유**: 요청/응답이 작고 빠른 응답이 중요해서. 응답이 512바이트를 넘거나 존 전송 시에는 TCP 사용

### 3-way/4-way Handshake

#### 3-way Handshake (연결 수립)
```
Client                          Server
  | ---- SYN (seq=x) ----------->  |   CLOSED → SYN_SENT
  | <--- SYN+ACK (seq=y, ack=x+1)  |   LISTEN → SYN_RCVD
  | ---- ACK (ack=y+1) --------->  |   ESTABLISHED
```
- 목적: **양쪽이 서로 송수신 가능함을 확인**하고 **초기 순서 번호(ISN)를 교환**
- 2-way로는 서버가 자신의 SYN이 도착했는지 알 수 없어서 최소 3번 필요
- ISN은 랜덤 → 이전 연결의 지연 패킷과 혼동 방지, 시퀀스 예측 공격 방지
- **SYN Flooding**: SYN만 대량으로 보내고 ACK를 안 보내 백로그 큐를 채우는 DoS 공격 → SYN Cookie로 방어

#### 4-way Handshake (연결 종료)
```
Client                          Server
  | ---- FIN ------------------->  |   FIN_WAIT_1
  | <--- ACK --------------------  |   CLOSE_WAIT  (서버는 남은 데이터 전송)
  | <--- FIN --------------------  |   LAST_ACK
  | ---- ACK ------------------->  |   TIME_WAIT → (일정 시간 후) CLOSED
```
- 4단계인 이유: 서버가 FIN을 받아도 **아직 보낼 데이터가 남아 있을 수 있어서** ACK와 FIN을 따로 보냄 (Half-Close)
- **TIME_WAIT**: 마지막 ACK 유실 시 재전송 대비 + 지연 패킷이 새 연결에 섞이는 것 방지. 보통 2MSL 동안 대기
  - 너무 많이 쌓이면 포트 고갈 → Keep-Alive, 커넥션 풀, `SO_REUSEADDR`, `tcp_tw_reuse` 등으로 완화
- **CLOSE_WAIT 누적**: 애플리케이션이 `close()`를 안 한 것 → 대부분 커넥션/스트림 반환 누락 같은 **코드 버그**

### DNS

- **DNS (Domain Name System)**: 도메인 이름 → IP 주소로 변환하는 분산 계층형 DB
- **계층 구조**: Root(`.`) → TLD(`.com`, `.kr`) → Authoritative(`example.com`)
- **조회 과정**
  1. 브라우저 캐시 → OS 캐시 → `hosts` 파일
  2. **Local DNS(Resolver, 보통 ISP)** 에 질의
  3. Local DNS가 Root → TLD → Authoritative 순서로 질의
  4. 결과를 받아 캐싱 후 클라이언트에 응답
- **재귀적(Recursive) vs 반복적(Iterative) 질의**: 클라이언트 → Local DNS는 재귀적, Local DNS → 각 서버는 반복적
- **주요 레코드**

| 레코드 | 의미 |
|---|---|
| A | 도메인 → IPv4 |
| AAAA | 도메인 → IPv6 |
| CNAME | 도메인 → 다른 도메인(별칭) |
| MX | 메일 서버 |
| NS | 해당 도메인의 네임서버 |
| TXT | 텍스트(도메인 소유 인증 등) |

- **TTL**: 캐시 유지 시간. 짧으면 변경 반영 빠름/부하 증가, 길면 반대. IP 변경 전에 TTL을 미리 줄여두면 반영이 빠름
- **DNS 로드밸런싱**: 한 도메인에 여러 A 레코드 (Round Robin DNS). 헬스체크가 없고 캐시 때문에 정교한 제어는 어려움

### Port

- **포트**: 한 IP 안에서 **어떤 프로세스(애플리케이션)** 에게 데이터를 줄지 구분하는 16비트 번호 (0 ~ 65535)
- IP = 아파트 주소, 포트 = 호수
- **범위**
  - 0 ~ 1023: Well-known (HTTP 80, HTTPS 443, SSH 22, FTP 21, DNS 53, SMTP 25)
  - 1024 ~ 49151: Registered (MySQL 3306, PostgreSQL 5432, Redis 6379, Tomcat/Spring 8080)
  - 49152 ~ 65535: Dynamic/Ephemeral (클라이언트가 연결할 때 OS가 임시 할당)
- **연결 식별 5-tuple**: `(프로토콜, 출발지 IP, 출발지 Port, 목적지 IP, 목적지 Port)`
  → 서버는 포트 하나(443)로 수많은 클라이언트 연결을 받을 수 있음
- 프로토콜이 다르면 같은 포트 번호를 동시에 사용 가능 (예: DNS 53/TCP, 53/UDP)

## Backend 심화 (선택)

### Socket

- **소켓**: 애플리케이션이 네트워크(TCP/UDP)를 쓰기 위해 OS가 제공하는 **엔드포인트 추상화**. 리눅스에선 **파일 디스크립터**로 다뤄짐
- **TCP 서버/클라이언트 흐름**
```
Server: socket() → bind() → listen() → accept() → read()/write() → close()
Client: socket() → connect() ─────────────────→ read()/write() → close()
```
- `connect()` 호출 시 3-way handshake 발생, `close()` 시 4-way handshake 시작
- `listen()`의 **backlog 큐**: handshake 완료된 연결이 `accept()` 되기 전 대기하는 큐
- **Blocking vs Non-blocking I/O**
  - Blocking: `accept()`/`read()`가 끝날 때까지 스레드 대기 → 연결당 스레드 필요 (Spring MVC + Tomcat)
  - Non-blocking + I/O Multiplexing: `select` / `poll` / **`epoll`** 로 하나의 스레드가 여러 소켓 감시 → Netty, Nginx, Node.js, Spring WebFlux
- **C10K 문제**: 동시 접속 1만 개를 연결당 스레드로 처리하면 메모리·컨텍스트 스위칭 비용이 한계 → 이벤트 루프 기반 논블로킹 모델로 해결
- **소켓 vs 웹소켓**: 소켓은 OS 레벨 통신 인터페이스, WebSocket은 HTTP에서 업그레이드되는 **L7 양방향 프로토콜**

### Server Connection 관리

- **연결 생성 비용이 비싸다**: TCP handshake(+TLS handshake) + 서버 리소스 할당 → 매 요청마다 새로 맺으면 지연↑, TIME_WAIT↑
- **HTTP Keep-Alive (Persistent Connection)**
  - HTTP/1.0: 요청마다 연결 종료가 기본 → `Connection: keep-alive` 헤더 필요
  - HTTP/1.1: 기본이 keep-alive
  - HTTP/2: 하나의 연결에서 **멀티플렉싱** (여러 요청 동시 처리)
  - HTTP/3: QUIC(UDP) 기반, TCP 레벨 HOL Blocking까지 해결
  - 단점: 요청 없이 열려만 있는 연결도 리소스를 점유 → keep-alive timeout, 최대 요청 수로 제한
- **Connection Pool**: 미리 연결을 만들어 두고 재사용 → 응답 시간 단축 + DB로 향하는 연결 수 제한
  - **DB**: HikariCP (Spring Boot 기본)
    - `maximumPoolSize`: 최대 커넥션 수
    - `minimumIdle`: 사용되지 않고 대기 중인 커넥션의 최소 개수
    - `maxLifetime`: 커넥션 최대 수명 (DB의 `wait_timeout`보다 짧게)
    - `connectionTimeout`: 풀에서 커넥션 얻기까지 대기 한도
  - **HTTP Client**: RestTemplate/WebClient + Apache HttpClient/Reactor Netty 풀 설정
- **풀 사이즈**: 크다고 좋은 게 아님. 과도하면 DB 쪽 컨텍스트 스위칭/락 경합 증가 → 부하 테스트로 결정
- **풀 고갈 원인**: 슬로우 쿼리로 점유 시간 증가, 커넥션 누수, 트랜잭션 안에서 외부 API 호출 같은 긴 작업
- **Connection Leak**: 반환 누락 → 풀 고갈 → 전체 요청 대기. HikariCP `leakDetectionThreshold`로 탐지
- **Graceful Shutdown**: 새 요청은 받지 않고 진행 중인 요청을 마무리한 뒤 종료 (`server.shutdown=graceful`)

### Connection Timeout

타임아웃이 없으면 **하나의 느린 외부 시스템이 스레드를 전부 잡아먹고 서비스 전체로 장애가 전파**된다.

| 종류 | 의미 | 발생 시점 |
|---|---|---|
| **Connection Timeout** | 연결 수립(3-way handshake)까지 기다리는 최대 시간 | 서버 다운, 방화벽 차단, 네트워크 불통 |
| **Read(Socket) Timeout** | 연결 후 응답 데이터를 기다리는 최대 시간 | 서버가 느리게 처리 중 |
| **Write Timeout** | 데이터 전송 완료까지 최대 시간 | 대용량 전송, 네트워크 느림 |
| **Pool Acquire Timeout** | 풀에서 커넥션을 얻기까지 대기 시간 | 풀 고갈 (HikariCP `connectionTimeout`) |
| **Idle Timeout** | 사용되지 않는 연결을 끊기까지 시간 | 오래 쓰이지 않은 연결 정리 |

- Read Timeout은 **전체 응답 시간이 아니라 패킷 간 간격** 기준인 경우가 많음 → 전체 제한이 필요하면 별도 설정 (예: WebClient `responseTimeout`)
- **타임아웃 값 설정**: 평소 응답 시간 분포(p99 등)에 여유를 두고, retry까지 포함해도 SLA 안에 들어오게
- **계층별 타임아웃 정합성**: LB/Nginx > 애플리케이션 > DB/외부 API 순으로 안쪽이 먼저 끊기게 설계. 반대면 클라이언트는 끊겼는데 서버는 계속 일하는 낭비 발생
- **타임아웃 = 실패가 아니라 "결과를 모름"**: 서버에선 이미 처리됐을 수 있음 → 결제 같은 요청은 **멱등성 키(Idempotency Key)** 나 상태 조회 후 재시도
- **장애 대응 패턴**
  - **Retry**: 일시적 오류 + 멱등한 요청에만, **지수 백오프 + Jitter**
  - **Circuit Breaker**: 실패율이 높으면 호출 자체를 차단 (Resilience4j)
  - **Fallback**: 캐시 데이터나 기본값 반환

## 면접 질문

**Q1. 브라우저에 URL을 입력하면 어떤 일이 일어나나요?**
> URL을 파싱하고 캐시를 확인한 뒤, DNS로 도메인의 IP를 조회합니다. 그 IP와 포트로 TCP 3-way handshake를 맺고, HTTPS라면 TLS handshake를 거쳐 HTTP 요청을 보냅니다. 서버가 응답하면 브라우저가 이를 렌더링합니다.

**Q2. TCP와 UDP의 차이는 무엇인가요?**
> TCP는 연결 지향으로 handshake를 거치고 ACK·재전송·순서 보장으로 신뢰성을 제공하지만 상대적으로 느립니다. UDP는 연결 없이 데이터를 보내 빠르지만 유실과 순서를 보장하지 않습니다. 정확성이 중요하면 TCP, 실시간성이 중요하면 UDP를 사용합니다.

**Q3. 왜 2-way가 아니라 3-way handshake인가요?**
> 2-way로는 서버가 자신의 SYN이 클라이언트에 도착했는지 확인할 수 없습니다. 양방향 모두 송수신이 가능한지, 그리고 서로의 초기 순서 번호를 확인하려면 최소 3번의 교환이 필요합니다.

**Q4. 커넥션 풀을 사용하는 이유와, 풀이 고갈됐을 때 어떻게 대응하나요?**
> 연결을 맺을 때마다 TCP 연결과 인증 비용이 들기 때문에, 미리 만든 연결을 재사용해 응답 시간을 줄이고 DB 연결 수를 제한하기 위해 사용합니다. 풀이 고갈되면 사이즈부터 늘리기보다 슬로우 쿼리, 커넥션 누수, 트랜잭션 안의 긴 외부 호출 여부를 먼저 확인합니다.