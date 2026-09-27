# 4주차 - Network

## 1. OSI 7 Layer / TCP-IP

네트워크에서는 데이터를 한 번에 처리하는 것이 아니라 **여러 계층으로 역할을 나누어 처리**한다.

계층을 나누면 각 계층이 자신의 역할에 집중할 수 있고, 특정 계층의 기술이 변경되어도 다른 계층에 미치는 영향을 줄일 수 있다.

대표적인 네트워크 계층 모델로 **OSI 7 Layer**와 **TCP/IP 모델**이 있다.

### OSI 7 Layer

> 네트워크 통신 과정을 7개의 계층으로 나눈 모델이다.

| 계층 | 이름 | 주요 역할 | 대표 예시 |
|---|---|---|---|
| 7 | Application | 사용자와 가까운 네트워크 서비스 | HTTP, DNS |
| 6 | Presentation | 데이터 형식 변환, 암호화, 압축 | 인코딩, 암호화 |
| 5 | Session | 통신 세션 관리 | 세션 연결/유지 |
| 4 | Transport | 프로세스 간 데이터 전송 | TCP, UDP |
| 3 | Network | 목적지까지 패킷 전달 | IP |
| 2 | Data Link | 같은 네트워크 내 데이터 전달 | Ethernet, MAC |
| 1 | Physical | 실제 신호 전송 | 케이블, 전기/광 신호 |

예를 들어 HTTP 요청을 보낸다고 생각하면 각 계층에서 필요한 정보를 추가하면서 데이터가 전달된다.

```text
Application
HTTP 요청
    ↓
Transport
TCP Header 추가
    ↓
Network
IP Header 추가
    ↓
Data Link
Ethernet Header 추가
    ↓
Physical
Bit로 전송
```

수신 측에서는 반대로 각 계층의 정보를 제거하면서 원래 데이터를 확인한다.

```text
Physical
    ↓
Data Link
    ↓
Network
    ↓
Transport
    ↓
Application
HTTP 요청 확인
```

이처럼 데이터를 보내면서 각 계층의 정보를 추가하는 과정을 **Encapsulation(캡슐화)**, 수신하면서 제거하는 과정을 **Decapsulation(역캡슐화)**이라고 한다.

### TCP/IP 모델

실제 인터넷에서는 OSI 7 Layer보다 **TCP/IP 모델**을 중심으로 설명하는 경우가 많다.

TCP/IP 모델은 보통 다음과 같이 4개의 계층으로 구분한다.

| TCP/IP | OSI | 대표 프로토콜 |
|---|---|---|
| Application | Application + Presentation + Session | HTTP, HTTPS, DNS |
| Transport | Transport | TCP, UDP |
| Internet | Network | IP |
| Network Access | Data Link + Physical | Ethernet |

예를 들어 브라우저에서 서버로 HTTP 요청을 보낸다면 다음과 같이 연결해서 볼 수 있다.

```text
HTTP
Application Layer
    ↓
TCP
Transport Layer
    ↓
IP
Internet Layer
    ↓
Ethernet
Network Access Layer
```

### 계층을 왜 나눌까?

각 계층은 자신에게 필요한 역할을 담당한다.

예를 들어 HTTP는 실제 데이터가 어떤 경로로 목적지까지 전달되는지 직접 처리하지 않는다.

```text
HTTP
→ 어떤 요청과 응답을 주고받을지

TCP
→ 데이터를 신뢰성 있게 어떻게 전달할지

IP
→ 데이터를 어느 목적지로 전달할지

Ethernet
→ 현재 네트워크에서 데이터를 어떻게 전달할지
```

이처럼 역할을 분리하면 각 계층의 구현이 달라지더라도 전체 네트워크 구조를 비교적 독립적으로 발전시킬 수 있다.

## 2. IP

> IP(Internet Protocol)는 네트워크에서 데이터를 목적지까지 전달하기 위한 주소 지정과 라우팅을 담당하는 프로토콜이다.

인터넷에 연결된 장치들은 통신을 위해 IP 주소를 사용한다.

예를 들어 다음과 같은 IP 주소가 있다고 가정하자.

```text
Client
192.168.0.10

      ↓

Server
203.0.113.10
```

Client가 Server로 데이터를 보내려면 패킷에 출발지와 목적지 IP 주소가 포함된다.

```text
IP Packet

┌───────────────────────┐
│ Source IP             │
│ 192.168.0.10          │
├───────────────────────┤
│ Destination IP        │
│ 203.0.113.10          │
├───────────────────────┤
│ Data                  │
└───────────────────────┘
```

네트워크의 Router들은 목적지 IP를 확인하면서 패킷을 다음 네트워크로 전달한다.

```text
Client
  ↓
Router
  ↓
Router
  ↓
Router
  ↓
Server
```

이 과정을 **Routing**이라고 한다.

### IPv4

IPv4 주소는 32bit로 구성된다.

```text
192.168.0.10

192  → 8bit
168  → 8bit
0    → 8bit
10   → 8bit

총 32bit
```

따라서 IPv4가 표현할 수 있는 주소는 약 43억 개이다.

인터넷에 연결되는 장치가 증가하면서 IPv4 주소가 부족해졌고, 이를 해결하기 위한 여러 방법이 사용되고 있다.

대표적으로 **Private IP + NAT**, 그리고 더 큰 주소 공간을 제공하는 **IPv6**가 있다.

### Public IP / Private IP

인터넷에서 직접 식별되는 주소를 **Public IP(공인 IP)**라고 한다.

반면 내부 네트워크에서 사용하는 주소를 **Private IP(사설 IP)**라고 한다.

대표적인 IPv4 사설 주소 범위는 다음과 같다.

```text
10.0.0.0      ~ 10.255.255.255
172.16.0.0    ~ 172.31.255.255
192.168.0.0   ~ 192.168.255.255
```

집에서 여러 기기가 같은 공유기에 연결된 상황을 생각해볼 수 있다.

```text
Laptop  192.168.0.2 ─┐
Phone   192.168.0.3 ─┼→ Router → Public IP → Internet
Tablet  192.168.0.4 ─┘
```

내부에서는 각 장치가 서로 다른 Private IP를 사용하지만, 외부 인터넷과 통신할 때는 공유기의 Public IP를 통해 통신할 수 있다.

### NAT

> NAT(Network Address Translation)는 IP 주소를 다른 IP 주소로 변환하는 기술이다.

대표적으로 여러 Private IP를 하나의 Public IP를 통해 인터넷과 통신할 수 있도록 할 때 사용된다.

```text
Private Network

192.168.0.2 ─┐
192.168.0.3 ─┼→ NAT → Public IP → Internet
192.168.0.4 ─┘
```

실제로 여러 내부 연결을 구분하기 위해 Port까지 함께 변환하는 **NAPT(PAT)** 방식도 널리 사용된다.

여기서는 NAT를 **내부의 Private IP와 외부의 Public IP 사이의 주소 변환을 담당할 수 있는 기술** 정도로 이해하면 충분하다.

### IPv6

IPv4 주소 부족 문제를 해결하기 위해 등장한 것이 IPv6이다.

IPv4가 32bit 주소를 사용한다면 IPv6는 **128bit 주소**를 사용한다.

```text
IPv4
192.168.0.10

IPv6
2001:db8:85a3::8a2e:370:7334
```

훨씬 큰 주소 공간을 제공하기 때문에 IPv4보다 훨씬 많은 장치에 주소를 할당할 수 있다.

### IP만 있으면 통신을 신뢰할 수 있을까?

IP는 패킷을 목적지까지 전달하기 위해 사용되지만 **패킷의 도착 자체를 보장하지 않는다.**

예를 들어 네트워크 상황에 따라 다음과 같은 일이 발생할 수 있다.

```text
Packet 1 → 도착
Packet 2 → 유실
Packet 3 → 도착

또는

보낸 순서
1 → 2 → 3

도착 순서
2 → 1 → 3
```

IP 자체는 패킷이 유실되었을 때 다시 보내거나, 순서가 바뀌었을 때 원래 순서로 복구하는 기능을 제공하지 않는 **Best-Effort 방식**이다.

그렇다면 웹 서비스처럼 데이터가 정확하게 전달되어야 하는 경우에는 어떻게 해야 할까?

```text
IP
→ 목적지까지 Packet 전달

하지만
→ 패킷 유실 가능
→ 순서 변경 가능
→ 신뢰성 보장 X

        ↓

Transport Layer
        ↓
TCP / UDP
```

이러한 전송 특성을 다루는 대표적인 Transport Layer 프로토콜이 다음에 볼 **TCP와 UDP**이다.

## 3. TCP / UDP

Transport Layer의 대표적인 프로토콜로 **TCP와 UDP**가 있다.

둘 다 데이터를 애플리케이션 사이에 전달하기 위해 사용하지만, 데이터를 전달하는 방식에 차이가 있다.

### TCP

> TCP(Transmission Control Protocol)는 연결을 설정한 뒤 데이터를 신뢰성 있게 전달하는 연결 지향형 프로토콜이다.

TCP는 데이터를 보내기 전에 먼저 상대방과 연결을 설정한다.

```text
Client                     Server
  │                           │
  │      연결 설정            │
  │ ←───────────────────────→ │
  │                           │
  │       데이터 전송         │
  │ ────────────────────────→ │
  │                           │
```

그리고 데이터가 제대로 전달되었는지 확인하면서 통신한다.

### TCP가 신뢰성을 보장하는 방법

TCP는 대표적으로 다음과 같은 기능을 제공한다.

- **순서 보장**: 데이터의 순서가 바뀌어 도착해도 원래 순서로 처리
- **재전송**: 데이터가 유실되면 다시 전송
- **오류 검출**: 전송 과정에서 데이터 오류 확인
- **흐름 제어**: 수신자가 처리할 수 있는 속도를 고려해 전송
- **혼잡 제어**: 네트워크의 혼잡 상태를 고려해 전송량 조절

예를 들어 데이터를 다음 순서로 전송했다고 가정하자.

```text
전송
1 → 2 → 3

2번 데이터 유실

수신
1 → 3

      ↓

2번 데이터 재전송

      ↓

1 → 2 → 3
```

이러한 기능 덕분에 TCP는 **데이터를 정확하게 전달해야 하는 통신**에 적합하다.

HTTP/1.1과 HTTP/2 등 일반적인 웹 통신에서도 TCP가 사용된다.

### UDP

> UDP(User Datagram Protocol)는 연결을 설정하지 않고 데이터를 전달하는 비연결형 프로토콜이다.

TCP처럼 데이터를 보내기 전에 별도의 연결을 설정하지 않는다.

```text
Client                     Server
  │                           │
  │       데이터 전송         │
  │ ────────────────────────→ │
  │                           │
  │       데이터 전송         │
  │ ────────────────────────→ │
```

UDP 자체에서는 데이터가 도착했는지 확인하거나 유실된 데이터를 재전송하지 않는다.

따라서 TCP보다 제공하는 기능이 단순하고 Header도 작다.

### UDP는 언제 사용할까?

모든 데이터가 반드시 도착하는 것보다 **빠르게 데이터를 전달하는 것이 중요한 경우**에 사용할 수 있다.

대표적으로 실시간 스트리밍, 음성/영상 통신, 온라인 게임 등의 통신에서 활용될 수 있다.

DNS도 일반적인 질의에서는 주로 UDP를 사용한다.

다만 `UDP = 무조건 빠른 통신`, `TCP = 무조건 느린 통신`이라고 단순하게 이해하기보다는 **TCP가 신뢰성을 위한 여러 기능을 제공하고 UDP는 이를 애플리케이션에 맡기는 차이**가 있다고 이해하는 것이 좋다.

### TCP vs UDP

| 구분 | TCP | UDP |
|---|---|---|
| 연결 방식 | 연결 지향 | 비연결 |
| 신뢰성 | 높음 | 자체적으로 보장하지 않음 |
| 순서 보장 | O | X |
| 재전송 | O | X |
| 흐름/혼잡 제어 | O | X |
| Header | 상대적으로 큼 | 작음 |
| 대표 사용 | HTTP/1.1, HTTP/2 | DNS, 실시간 통신 등 |

### 그렇다면 HTTP/3는?

HTTP/1.1과 HTTP/2는 TCP를 기반으로 동작하지만, **HTTP/3는 UDP 위에서 동작하는 QUIC**을 사용한다.

```text
HTTP/1.1, HTTP/2
        ↓
       TCP
        ↓
        IP


HTTP/3
        ↓
       QUIC
        ↓
       UDP
        ↓
        IP
```

그렇다고 HTTP/3가 신뢰성을 포기한 것은 아니다.

QUIC이 UDP 위에서 연결 관리, 신뢰성 있는 전송, 혼잡 제어 등의 기능을 제공한다.

여기서는 **UDP를 사용한다고 반드시 신뢰성 없는 애플리케이션이 되는 것은 아니다** 정도로 이해하면 충분하다.

## 4. 3-way / 4-way Handshake

TCP는 데이터를 전송하기 전에 연결을 설정하고, 통신이 끝나면 연결을 종료한다.

대표적으로 연결 설정 과정이 **3-way Handshake**, 연결 종료 과정이 **4-way Handshake**이다.

### 3-way Handshake

> TCP 통신을 시작하기 전에 Client와 Server가 연결을 설정하는 과정이다.

```text
Client                              Server
  │                                   │
  │ -------- SYN -------------------> │
  │                                   │
  │ <------ SYN + ACK --------------- │
  │                                   │
  │ -------- ACK -------------------> │
  │                                   │
  │         Connection Established    │
```

각 단계를 보면 다음과 같다.

```text
1. Client → Server : SYN
   연결을 시작하고 싶다는 요청

2. Server → Client : SYN + ACK
   요청을 확인했고 Server도 연결 준비가 되었음을 전달

3. Client → Server : ACK
   Server의 응답을 확인

→ TCP Connection 생성
```

`SYN`은 Synchronize, `ACK`는 Acknowledgment를 의미한다.

TCP는 이 과정에서 서로 통신 가능한 상태인지 확인하고 **초기 Sequence Number를 교환하여 데이터 전송을 준비한다.**

### 왜 3번이나 주고받을까?

TCP는 양방향 통신을 하기 때문에 **Client와 Server 모두 상대방에게 데이터를 보내고 받을 수 있는 상태인지 확인할 필요가 있다.**

```text
Client → Server 가능?
Server → Client 가능?

양쪽의 통신 가능 여부 확인
        ↓
TCP Connection 설정
```

단순히 Client가 SYN을 보내고 Server가 응답하는 것만으로는 Server의 응답을 Client가 정상적으로 받았다는 사실을 Server가 확인할 수 없다.

마지막 ACK를 통해 Server도 자신의 응답이 Client에게 전달되었다는 것을 확인할 수 있다.

### 4-way Handshake

TCP 연결을 종료할 때는 일반적으로 **4-way Handshake** 과정을 거친다.

```text
Client                              Server
  │                                   │
  │ -------- FIN -------------------> │
  │                                   │
  │ <------- ACK -------------------- │
  │                                   │
  │ <------- FIN -------------------- │
  │                                   │
  │ -------- ACK -------------------> │
  │                                   │
  │          Connection 종료          │
```

과정을 보면 다음과 같다.

```text
1. Client → Server : FIN
   더 이상 보낼 데이터가 없으므로 연결 종료 요청

2. Server → Client : ACK
   종료 요청을 확인

3. Server → Client : FIN
   Server도 데이터를 모두 보낸 뒤 연결 종료 요청

4. Client → Server : ACK
   Server의 종료 요청을 확인
```

### 왜 종료는 4-way일까?

TCP 연결은 **양방향으로 독립적으로 데이터를 전송할 수 있는 Full-Duplex 통신**이다.

Client가 더 이상 데이터를 보내지 않겠다고 해도 Server에는 아직 보낼 데이터가 남아 있을 수 있다.

```text
Client
"나는 전송 끝"
   │
   │ FIN
   ↓
Server
"확인했지만 나는 아직 보낼 데이터가 있음"
   │
   │ 데이터 처리 완료
   ↓
"나도 전송 끝"
   │
   │ FIN
   ↓
Client
```

따라서 한쪽의 종료 요청을 확인하는 ACK와 반대 방향의 종료를 의미하는 FIN이 별도로 전송될 수 있어 일반적으로 4단계가 된다.

반면 연결 설정에서는 Server가 `SYN`과 `ACK`를 하나의 Segment에 함께 보낼 수 있어 3-way Handshake가 된다.

### TIME_WAIT

TCP 연결 종료 과정에서 먼저 연결 종료를 요청한 쪽은 마지막 ACK를 보낸 뒤 바로 연결 정보를 없애지 않고 일정 시간 **TIME_WAIT** 상태로 남을 수 있다.

```text
Client                     Server

FIN ──────────────────────→
    ←───────────────── ACK
    ←───────────────── FIN
ACK ──────────────────────→

Client
  ↓
TIME_WAIT
  ↓
일정 시간 후
  ↓
CLOSED
```

대표적인 이유는 **마지막 ACK가 유실되는 경우에 대비**하고, 이전 연결에서 지연된 패킷이 이후의 새로운 연결에 영향을 주는 것을 방지하기 위해서다.

서버에서 짧은 TCP Connection이 매우 많이 생성되고 종료되는 상황에서는 TIME_WAIT 상태의 Connection이 많이 보일 수도 있다.

### 전체 흐름

```text
Client가 Server와 TCP 통신 시작
        ↓
3-way Handshake
        ↓
TCP Connection 설정
        ↓
데이터 송수신
        ↓
통신 종료
        ↓
4-way Handshake
        ↓
TCP Connection 종료
```

TCP가 데이터를 신뢰성 있게 전달하는 프로토콜이라면, **3-way Handshake는 TCP 연결을 시작하기 위한 과정이고 4-way Handshake는 연결을 종료하기 위한 과정**이라고 연결해서 이해하면 된다.

## 5. DNS

> DNS(Domain Name System)는 사람이 사용하는 Domain Name을 통신에 필요한 IP 주소로 변환해주는 시스템이다.

사용자는 웹사이트에 접속할 때 일반적으로 IP 주소를 직접 입력하지 않고 Domain Name을 사용한다.

```text
www.example.com
       ↓
      DNS
       ↓
203.0.113.10
```

네트워크에서는 목적지를 찾기 위해 IP 주소가 필요하기 때문에 Domain Name에 대응하는 IP 주소를 알아내는 과정이 필요하다.

### DNS 조회 과정

브라우저에서 `www.example.com`에 접속한다고 가정하자.

개념적으로 다음과 같은 과정을 거쳐 IP 주소를 찾는다.

```text
사용자
  ↓
www.example.com 입력
  ↓
DNS Cache 확인
  ↓
DNS Resolver
  ↓
Root DNS
  ↓
TLD DNS (.com)
  ↓
Authoritative DNS
  ↓
IP 주소 확인
  ↓
해당 Server로 통신
```

실제로는 매번 모든 DNS 서버를 순서대로 조회하는 것은 아니다.

이미 조회한 결과가 Cache에 존재한다면 이를 재사용하여 DNS 조회 시간을 줄일 수 있다.

### DNS 서버의 역할

DNS는 하나의 서버가 인터넷의 모든 Domain 정보를 가지고 있는 구조가 아니라 계층적으로 구성되어 있다.

```text
Root DNS
   ↓
TLD DNS
   ↓
Authoritative DNS
```

**Root DNS**

`.com`, `.net`, `.kr` 등을 담당하는 TLD DNS Server의 위치를 안내한다.

**TLD(Top-Level Domain) DNS**

`.com`과 같은 특정 최상위 Domain을 관리하며 해당 Domain의 Authoritative DNS Server를 찾을 수 있도록 안내한다.

**Authoritative DNS**

실제 Domain에 대한 DNS Record를 가지고 있으며 최종적으로 필요한 정보를 제공한다.

### DNS Record

DNS에는 IP 주소뿐 아니라 다양한 정보를 저장할 수 있다.

대표적인 Record는 다음과 같다.

| Record | 역할 |
|---|---|
| A | Domain → IPv4 주소 |
| AAAA | Domain → IPv6 주소 |
| CNAME | 다른 Domain Name을 가리킴 |
| MX | Mail Server 정보 |
| NS | 해당 Domain의 DNS Server 정보 |

예를 들어 A Record는 다음과 같은 관계를 나타낸다.

```text
example.com
    ↓ A Record
203.0.113.10
```

### DNS Cache

DNS 조회 결과는 일정 시간 동안 Cache에 저장될 수 있다.

```text
첫 번째 요청

Domain
  ↓
DNS 조회
  ↓
IP 확인
  ↓
Cache 저장


다음 요청

Domain
  ↓
Cache 확인
  ↓
IP 바로 사용
```

DNS Record에는 **TTL(Time To Live)**이 설정될 수 있으며, 해당 정보가 얼마나 오래 Cache될 수 있는지를 나타낸다.

따라서 서버의 IP를 변경하더라도 기존 DNS Cache가 남아 있다면 모든 사용자가 즉시 새로운 IP를 사용하는 것은 아닐 수 있다.

## 6. Port

> Port는 하나의 호스트에서 어떤 프로세스 또는 네트워크 서비스를 대상으로 통신할지 구분하기 위해 사용하는 번호이다.

IP 주소만으로는 데이터를 어떤 서버 프로그램에 전달해야 하는지 알 수 없다.

예를 들어 하나의 서버에서 여러 서비스가 실행될 수 있다.

```text
Server
IP: 203.0.113.10

├─ Web Server
├─ SSH Server
└─ Database
```

이때 Port를 이용해 각각의 서비스를 구분할 수 있다.

```text
203.0.113.10:80    → HTTP
203.0.113.10:443   → HTTPS
203.0.113.10:22    → SSH
203.0.113.10:3306  → MySQL
```

즉, 간단하게 생각하면 다음과 같다.

```text
IP
→ 어떤 Host인가?

Port
→ 그 Host의 어떤 서비스인가?
```

### Port 번호

Port 번호는 16bit이므로 **0 ~ 65535** 범위를 가진다.

대표적인 Port는 다음과 같다.

| Port | 대표 서비스 |
|---|---|
| 22 | SSH |
| 53 | DNS |
| 80 | HTTP |
| 443 | HTTPS |
| 3306 | MySQL |
| 5432 | PostgreSQL |

Port 번호는 크게 Well-Known Port, Registered Port, Dynamic/Private Port 등의 범위로 나눌 수 있다.

여기서는 **서버의 특정 서비스를 구분하기 위해 Port를 사용한다**는 개념이 가장 중요하다.

### Client도 Port를 사용할까?

Server만 Port를 사용하는 것은 아니다.

Client가 Server와 TCP 통신을 시작하면 Client 측에서도 임시 Port가 사용된다.

```text
Client
192.168.0.10:53001

        ↓

Server
203.0.113.10:443
```

Client의 `53001`과 같은 Port는 OS가 사용 가능한 임시 Port 중 하나를 할당할 수 있다.

이를 **Ephemeral Port**라고 한다.

따라서 네트워크 연결은 단순히 Server의 IP와 Port만으로 구성되는 것이 아니다.

TCP 연결은 일반적으로 다음 네 가지 정보의 조합으로 구분할 수 있다.

```text
Source IP
Source Port
Destination IP
Destination Port
```

이를 흔히 **4-Tuple**이라고 한다.

```text
192.168.0.10 : 53001
        ↓
203.0.113.10 : 443
```

Server의 Port가 모두 `443`이더라도 Client의 IP와 Port 등이 다르기 때문에 여러 TCP Connection을 구분할 수 있다.

## 7. Socket

> Socket은 네트워크를 통해 프로세스가 데이터를 주고받기 위해 사용하는 통신 Endpoint이다.

애플리케이션은 TCP/IP를 직접 하나하나 제어하기보다 OS가 제공하는 Socket 인터페이스를 통해 네트워크 통신을 수행할 수 있다.

```text
Application
     ↓
   Socket
     ↓
TCP / UDP
     ↓
     IP
     ↓
 Network
```

### TCP Server의 Socket 통신

TCP Server의 기본적인 흐름은 다음과 같다.

```text
Server

socket()
   ↓
bind()
   ↓
listen()
   ↓
accept()
   ↓
데이터 송수신
   ↓
close()
```

**bind()**

Socket에 사용할 IP와 Port를 지정한다.

```text
Server Socket
     ↓
0.0.0.0:8080
```

**listen()**

Client의 연결 요청을 받을 수 있는 상태로 만든다.

**accept()**

Client의 연결 요청을 받아들이고, 해당 Client와 통신하기 위한 연결된 Socket을 얻는다.

### Server Socket과 연결된 Socket

Server는 하나의 Port에서 여러 Client의 연결을 처리할 수 있다.

```text
                Server
              Port 8080
                  │
            Listening Socket
                  │
        ┌─────────┼─────────┐
        ↓         ↓         ↓
     Client A  Client B  Client C
```

실제로 TCP Server에서는 연결 요청을 받는 **Listening Socket**과 각 Client와 실제 데이터를 주고받는 **연결된 Socket**을 구분해서 이해하는 것이 좋다.

```text
Listening Socket
       ↓ accept()
Connected Socket A ↔ Client A

       ↓ accept()
Connected Socket B ↔ Client B

       ↓ accept()
Connected Socket C ↔ Client C
```

따라서 Server가 하나의 `8080` Port를 사용한다고 해서 Client 한 명만 연결할 수 있는 것은 아니다.

각 TCP Connection은 앞에서 본 4-Tuple을 통해 서로 구분할 수 있다.

### 지금까지의 흐름

브라우저에서 서버에 접속하는 과정을 지금까지 배운 내용으로 연결하면 다음과 같다.

```text
www.example.com 입력
        ↓
DNS
        ↓
Server IP 확인
        ↓
IP + Port로 목적지 결정
        ↓
TCP 3-way Handshake
        ↓
TCP Connection 생성
        ↓
Socket을 통해 데이터 송수신
        ↓
HTTP Request / Response
```

즉,

```text
DNS
→ Domain을 IP로 변환

IP
→ 통신할 Host를 식별

Port
→ Host에서 통신할 서비스를 식별

TCP
→ 신뢰성 있는 연결과 데이터 전송

Socket
→ Application이 네트워크 통신을 사용하기 위한 Endpoint
```

로 연결해서 이해하면 된다.

## 8. Server Connection 관리

서버에는 여러 Client가 동시에 연결될 수 있다.

```text
Client A ─┐
Client B ─┼─→ Server : 8080
Client C ─┘
```

서버는 하나의 Port에서 요청을 받지만, 각각의 Connection은 서로 다른 Socket으로 관리할 수 있다.

```text
Listening Socket : 8080
        ↓
    accept()
        ↓
┌──────────────────────┐
│ Client A ↔ Socket A  │
│ Client B ↔ Socket B  │
│ Client C ↔ Socket C  │
└──────────────────────┘
```

따라서 하나의 Port를 사용하더라도 여러 Client와 동시에 통신할 수 있다.

### Connection마다 Thread를 하나씩 만들면?

가장 단순한 서버 구조는 Client가 연결될 때마다 Thread를 생성하는 방식이다.

```text
Connection A → Thread A
Connection B → Thread B
Connection C → Thread C
```

Connection이 적을 때는 단순하지만 연결이 많아지면 문제가 발생할 수 있다.

```text
Connection 증가
      ↓
Thread 증가
      ↓
메모리 사용 증가
Context Switching 증가
      ↓
성능 저하 가능
```

2주차에서 배운 것처럼 Thread는 생성 비용과 Stack 메모리가 필요하고, 지나치게 많아지면 Context Switching 비용도 증가한다.

그래서 실제 서버에서는 **Thread Pool**이나 **I/O Multiplexing** 등의 방식을 이용해 많은 Connection을 효율적으로 처리한다.

### Thread Pool

미리 일정한 수의 Worker Thread를 생성하고 들어오는 작업을 처리하는 방식이다.

```text
Connection / Request
        ↓
     작업 대기
        ↓
┌─────────────────┐
│   Thread Pool   │
│                 │
│   Worker 1      │
│   Worker 2      │
│   Worker 3      │
└─────────────────┘
```

요청마다 새로운 Thread를 생성하지 않고 기존 Worker Thread를 재사용할 수 있다.

Spring Boot에서 일반적으로 사용하는 Tomcat 역시 이러한 방식으로 요청 처리에 사용할 Worker Thread를 관리한다.

### Blocking I/O

Blocking 방식에서는 Thread가 I/O 작업의 결과를 기다리는 동안 해당 Thread도 대기한다.

```text
Thread
  ↓
DB 요청
  ↓
응답 대기...
  ↓
DB 응답
  ↓
다음 작업
```

Network I/O도 마찬가지다.

```text
Thread A → Client A의 데이터 대기
Thread B → Client B의 데이터 대기
Thread C → Client C의 데이터 대기
```

Connection이 많아지면 데이터를 기다리는 Thread도 많아질 수 있다.

### I/O Multiplexing

많은 Connection을 효율적으로 처리하기 위한 방법 중 하나가 **I/O Multiplexing**이다.

하나의 Thread가 여러 Socket의 상태를 확인하고, 실제로 읽거나 쓸 준비가 된 Socket을 처리할 수 있다.

```text
Socket A ─┐
Socket B ─┤
Socket C ─┼→ I/O Multiplexing → Thread
Socket D ─┤
Socket E ─┘
```

대표적으로 Linux의 `select`, `poll`, `epoll` 등이 관련된 기술이다.

여기서는 세부 구현보다 **Connection마다 Thread 하나를 계속 점유시키지 않고 여러 Connection의 I/O 상태를 효율적으로 관리할 수 있다**는 개념 정도로 이해하면 충분하다.

### HTTP Keep-Alive

HTTP 요청 하나가 끝날 때마다 TCP Connection을 종료하고 다시 연결하면 매번 TCP 연결 과정이 필요하다.

```text
Request
↓
TCP 연결
↓
HTTP 요청/응답
↓
TCP 종료

다음 Request
↓
다시 TCP 연결
...
```

**Keep-Alive**를 사용하면 하나의 TCP Connection을 여러 HTTP 요청과 응답에 재사용할 수 있다.

```text
TCP Connection 생성
        ↓
Request 1 ↔ Response 1
        ↓
Request 2 ↔ Response 2
        ↓
Request 3 ↔ Response 3
        ↓
Connection 종료
```

이를 통해 매 요청마다 새로운 TCP Connection을 만드는 비용을 줄일 수 있다.

하지만 Connection을 무한정 유지할 수는 없기 때문에 일정 시간이 지나면 종료하는 등의 관리가 필요하다.

이때 중요한 개념 중 하나가 **Timeout**이다.

## 9. Connection Timeout

> Timeout은 특정 작업이 일정 시간 안에 완료되지 않을 경우 더 이상 기다리지 않고 작업을 중단하기 위한 시간 제한이다.

네트워크에서는 상대방의 응답이 언제 올지 항상 보장할 수 없다.

```text
Server A
   ↓
Server B 요청
   ↓
응답 없음...
   ↓
계속 기다림
```

Timeout이 없다면 응답하지 않는 Connection이나 외부 시스템 때문에 Thread, Socket 등의 자원이 오랫동안 점유될 수 있다.

```text
응답 지연
   ↓
Thread 계속 대기
   ↓
대기 요청 증가
   ↓
사용 가능한 Thread 감소
   ↓
서버 전체 성능에 영향
```

따라서 네트워크 통신에서는 적절한 Timeout을 설정하는 것이 중요하다.

### Connect Timeout

> 상대 서버와 Connection을 설정하기까지 기다리는 최대 시간이다.

TCP 연결을 시도했지만 일정 시간 안에 연결되지 않는 경우를 생각할 수 있다.

```text
Client
  ↓
TCP 연결 시도
  ↓
Server 응답 대기
  ↓
일정 시간 초과
  ↓
Connect Timeout
```

즉, **연결 자체를 만드는 데 얼마나 기다릴 것인가**에 대한 설정이다.

### Read Timeout

> Connection이 만들어진 후 상대방의 데이터를 기다리는 최대 시간과 관련된 Timeout이다.

```text
Connection 성공
      ↓
Request 전송
      ↓
Response 기다림...
      ↓
일정 시간 동안 데이터 없음
      ↓
Read Timeout
```

서버와 연결되었다고 해서 응답이 반드시 빠르게 오는 것은 아니다.

외부 API나 DB 등이 느려지면 응답을 오랫동안 기다릴 수 있기 때문에 Read Timeout도 중요하다.

### Connection Timeout과 Read Timeout

| 구분 | 의미 |
|---|---|
| Connect Timeout | 연결을 설정할 때 얼마나 기다릴지 |
| Read Timeout | 연결 후 데이터를 기다릴 때 얼마나 기다릴지 |

예를 들어 외부 API를 호출한다고 생각하면 다음과 같이 구분할 수 있다.

```text
우리 Server
    ↓
외부 API에 연결 시도
    ↓
[Connect Timeout]
    ↓
Connection 성공
    ↓
Request 전송
    ↓
Response 대기
    ↓
[Read Timeout]
```

### Timeout은 왜 중요한가?

백엔드 서버는 하나의 요청만 처리하지 않는다.

여러 요청이 동시에 들어오는 상황에서 특정 외부 시스템이 느려졌다고 가정해보자.

```text
Request A → 외부 API 대기
Request B → 외부 API 대기
Request C → 외부 API 대기
Request D → 외부 API 대기
                ↓
          Thread 계속 점유
                ↓
        Thread Pool 고갈 가능
```

적절한 Timeout을 설정하면 문제가 발생한 외부 시스템을 무한정 기다리지 않고 실패를 처리할 수 있다.

```text
외부 API 지연
      ↓
Timeout 발생
      ↓
대기 중단
      ↓
예외 처리 / 재시도 / 실패 응답
      ↓
자원 반환
```

따라서 Timeout은 단순히 사용자가 오래 기다리지 않도록 하는 설정이 아니라 **서버의 Thread, Connection 등의 자원이 무한정 점유되는 것을 막기 위한 중요한 보호 장치**이기도 하다.

### 전체 흐름

```text
사용자가 Domain 입력
        ↓
DNS
        ↓
Server IP 확인
        ↓
IP + Port
        ↓
TCP 3-way Handshake
        ↓
TCP Connection 생성
        ↓
Socket을 통해 통신
        ↓
Server가 Connection / Request 처리
        ↓
Thread Pool / I/O 처리
        ↓
DB 또는 외부 API 호출
        ↓
응답 지연 가능
        ↓
Timeout으로 대기 시간 제한
        ↓
응답 완료
        ↓
Connection 재사용 또는 종료
```

이번 주의 개념은 각각 따로 외우기보다 **DNS로 서버의 IP를 찾고, IP와 Port를 통해 목적지를 정한 뒤 TCP Connection을 만들고, Socket을 통해 통신하며, 서버는 여러 Connection과 Timeout을 관리한다**는 하나의 흐름으로 연결해서 이해하는 것이 중요하다.
