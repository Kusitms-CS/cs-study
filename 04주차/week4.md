# 4주차 - Network 개념 정리

## 목차
- [키워드 연관관계](#키워드-연관관계)
- [OSI 7 Layer/TCP-IP](#osi-7-layertcp-ip)
- [IP](#ip)
- [TCP/UDP](#tcpudp)
- [3-way/4-way Handshake](#3-way4-way-handshake)
- [DNS](#dns)
- [Port](#port)
- [Backend 심화 (선택)](#backend-심화-선택)
  - [Socket](#socket)
  - [Server Connection 관리](#server-connection-관리)
  - [Connection Timeout](#connection-timeout)

---

## 키워드 연관관계


```
OSI 7 Layer/TCP-IP
├─ IP → DNS
├─ IP + Port → TCP/UDP → 3-way/4-way Handshake
└─ TCP/UDP + Port → Socket → Server Connection 관리 → Connection Timeout
```

| 연결 | 관계 설명 |
|---|---|
| OSI 7 Layer/TCP-IP → IP | 네트워크 계층(3계층)에 해당하는 실제 프로토콜이 IP |
| IP → DNS | 도메인 이름을 실제 통신에 쓸 IP 주소로 변환해야 연결 가능 |
| OSI 7 Layer/TCP-IP → TCP/UDP | 전송 계층(4계층)에 해당하는 실제 프로토콜이 TCP/UDP |
| IP + Port → TCP/UDP | TCP/UDP는 IP로 상대 컴퓨터를, Port로 그 안의 프로그램을 특정해서 통신 |
| TCP/UDP → 3-way/4-way Handshake | TCP가 연결을 맺고 끊을 때 거치는 절차 |
| TCP/UDP + Port → Socket | IP+Port 조합을 프로그램에서 다룰 수 있게 만든 실제 통신 창구 |
| Socket → Server Connection 관리 | 서버는 여러 클라이언트의 소켓(연결)을 동시에 관리해야 함 |
| Server Connection 관리 → Connection Timeout | 연결/응답이 안 올 때 무한정 기다리지 않도록 제한 시간을 둠 |

---

## OSI 7 Layer/TCP-IP


데이터가 위(7층, 사용자와 가까움)에서 아래(1층, 실제 전선/신호)로 내려가며 각 층을 하나씩 거친다.

```
7  응용        (Application)     사용자 프로그램 (HTTP, DNS 등)    
─ TCP/IP: 응용 계층
- - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - -
6  표현        (Presentation)    데이터 형식 변환/암호화            
─ TCP/IP: 응용 계층
- - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - -
5  세션        (Session)         연결 세션 관리                      
─ TCP/IP: 응용 계층
- - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - -
4  전송        (Transport)       TCP/UDP – 신뢰성 있는 전달          
─ TCP/IP: 전송 계층
- - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - -
3  네트워크    (Network)         IP – 주소 지정, 경로 설정          
─ TCP/IP: 인터넷 계층
- - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - -
2  데이터 링크 (Data Link)       MAC 주소, 프레임 전송              
  ─ TCP/IP: 네트워크 접근 계층
- - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - -
1  물리        (Physical)        전기 신호, 케이블               
 ─ TCP/IP: 네트워크 접근 계층
```

- 활용 예: 웹 요청 하나가 Application(HTTP) → Transport(TCP) → Network(IP) → Data Link/Physical 순으로 캡슐화되어 내려갔다가, 상대방에서 반대로 해석(역캡슐화)된다

---

## IP


네트워크 상에서 각 장치를 식별하는 **논리적 주소** 체계.

- IPv4(32비트, 예: `192.168.0.1`) vs IPv6(128비트, 주소 고갈 문제 해결)
- 예시: 사설 IP(내부망에서만 유효, 예: `192.168.x.x`) vs 공인 IP(인터넷에서 직접 식별 가능)

---

## TCP/UDP

> EX) TCP는 등기우편 — 받았는지 확인하고, 순서대로 배달되며, 유실되면 다시 보낸다. UDP는 그냥 우편함에 넣고 끝나는 일반 우편 — 도착을 보장하진 않지만 훨씬 빠르고 간단하다.

| 구분 | TCP | UDP |
|---|---|---|
| 연결 방식 | 연결 지향(Connection-oriented) | 비연결형(Connectionless) |
| 신뢰성 | 높음 (순서 보장, 재전송) | 낮음 (보장 없음) |
| 속도 | 상대적으로 느림 | 빠름 |
| 예시 | 웹(HTTP), 이메일, 파일 전송 | 실시간 스트리밍, 온라인 게임, DNS 조회 |

---

## 3-way/4-way Handshake


- **3-way Handshake (연결 수립)**: `SYN → SYN-ACK → ACK`, 총 3번 주고받아 TCP 연결을 맺음
- **4-way Handshake (연결 종료)**: `FIN → ACK → FIN → ACK`, 양쪽이 각자 종료 의사를 확인하며 연결을 끊음
- 활용 예: HTTPS 요청을 보내기 전에도 TCP 3-way handshake가 먼저 일어나고, 그다음 TLS handshake가 별도로 한 번 더 일어난다

---

## DNS


**도메인 이름**(예: `www.google.com`)을 **IP 주소**로 변환해주는 시스템.

- 계층 구조: Root DNS → TLD(`.com`, `.kr` 등) DNS → 권한 있는(Authoritative) DNS 서버 순으로 조회
- 활용 예: 브라우저에 주소를 치면 로컬 캐시 → OS 캐시 → ISP DNS 서버 → 필요 시 Root부터 순차 조회해서 IP를 찾아낸 뒤에야 실제 TCP 연결이 시작된다

---

## Port


하나의 IP(컴퓨터) 안에서 여러 프로그램(프로세스)을 구분하기 위한 번호(0~65535).

- Well-known Port: 0~1023 (예: HTTP `80`, HTTPS `443`, DNS `53`, SSH `22`)
- 활용 예: 같은 서버(IP)에서 웹 서버는 80번 포트, DB는 3306번(MySQL) 포트로 각각 다른 서비스가 동시에 돌아갈 수 있다

---

## Backend 심화 (선택)

### Socket
IP(주소) + Port(내선번호)를 조합 "실제로 통신할 수 있는 창구"다. 프로그램은 이 소켓을 통해 데이터를 주고받는다.

 > 네트워크 통신의 **종단점(Endpoint)**.

```java
Socket socket = new Socket("localhost", 8080); // IP + Port로 연결
```

- 운영체제가 제공하는 소켓 API를 통해 프로그램이 TCP/UDP 통신을 수행한다
- 활용 예: 웹 서버가 요청을 받는 것도 결국 "특정 포트에서 소켓을 열어두고(`ServerSocket`) 클라이언트의 연결을 기다리는 것"

---

### Server Connection 관리

서버가 여러 클라이언트의 연결(소켓)을 동시에 관리하는 방식.

- **Thread-per-connection**: 연결마다 스레드 하나씩 배정 (구현은 간단하지만 연결이 많아지면 비용 급증)
- **I/O Multiplexing** (`select`/`epoll`, 자바의 NIO `Selector`): 스레드 하나가 여러 연결을 감시하다 이벤트 발생 시 처리 → 적은 자원으로 대량 연결 처리 가능
- 활용 예: 채팅 서버나 실시간 서비스처럼 대규모 트래픽을 받는 서버는 Thread-per-connection 대신 Event Loop/NIO 기반으로 연결을 관리하는 경우가 많다

---

### Connection Timeout


연결을 시도하거나 응답을 기다릴 때, 정해진 시간 안에 완료되지 않으면 강제로 끊고 예외를 발생시키는 설정.

- **Connection Timeout**: 연결 자체를 맺는 데 걸리는 시간 제한
- **Read Timeout**: 연결된 후 응답 데이터를 받는 데 걸리는 시간 제한

