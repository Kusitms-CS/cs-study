# 네트워크

## 1. OSI 7 Layer / TCP-IP

### 1.1 OSI 7 Layer

네트워크 통신 과정을 7개의 계층으로 나눈 표준 모델임

각 계층은 독립적인 역할을 수행하며 상위 계층은 하위 계층의 기능을 이용함

| 계층 | 이름 | 주요 역할 | 대표 프로토콜 |
|---|---|---|---|
| 7 | Application | 사용자와 직접 상호작용하는 네트워크 서비스 제공 | HTTP, HTTPS, DNS, FTP |
| 6 | Presentation | 데이터 형식 변환, 암호화, 압축 | SSL/TLS |
| 5 | Session | 통신 세션 생성 및 유지, 종료 | RPC |
| 4 | Transport | 프로세스 간 데이터 전송 및 신뢰성 관리 | TCP, UDP |
| 3 | Network | 목적지까지의 경로 결정 및 패킷 전달 | IP, ICMP |
| 2 | Data Link | 같은 네트워크 내 장치 간 데이터 전달 | Ethernet, MAC |
| 1 | Physical | 전기적, 물리적 신호를 통한 데이터 전송 | 케이블, 전파 |

### 1.2 TCP/IP 모델

실제 인터넷 통신에서 주로 사용하는 네트워크 모델임

OSI 7 Layer를 실용적으로 단순화한 형태로 볼 수 있음

| TCP/IP 계층 | OSI 계층 | 대표 프로토콜 |
|---|---|---|
| Application | 5~7 계층 | HTTP, HTTPS, DNS |
| Transport | 4 계층 | TCP, UDP |
| Internet | 3 계층 | IP, ICMP |
| Network Access | 1~2 계층 | Ethernet, Wi-Fi |

### 1.3 데이터 전달 과정

HTTP 요청을 보낼 경우 각 계층을 내려가며 필요한 정보가 추가됨

    Application
    HTTP 데이터
        ↓
    Transport
    TCP Header + HTTP 데이터
        ↓
    Internet
    IP Header + TCP Header + HTTP 데이터
        ↓
    Network Access
    Frame + IP Header + TCP Header + HTTP 데이터

송신 측에서 헤더를 추가하는 과정을 캡슐화라고 함

수신 측에서 헤더를 제거하며 원래 데이터를 확인하는 과정을 역캡슐화라고 함


## 2. IP

IP는 Internet Protocol의 약자로 네트워크에서 장치를 식별하고 목적지까지 패킷을 전달하기 위한 프로토콜임

### 2.1 IP 주소

네트워크에 연결된 장치를 식별하기 위한 주소임

대표적으로 IPv4와 IPv6가 존재함

- IPv4는 32bit 주소 체계를 사용함
- IPv6는 128bit 주소 체계를 사용함

예시는 다음과 같음

    IPv4
    192.168.0.10

    IPv6
    2001:0db8:85a3::8a2e:0370:7334

### 2.2 공인 IP와 사설 IP

공인 IP는 인터넷에서 장치를 식별하기 위해 사용하는 IP임

사설 IP는 내부 네트워크에서 사용하는 IP임

가정이나 회사에서는 여러 장치가 사설 IP를 사용하고 공유기가 NAT를 통해 하나의 공인 IP를 공유하는 구조가 일반적임

    PC 192.168.0.2
          ↓
        공유기
          ↓
    사설 IP → 공인 IP 변환
          ↓
       Internet

### 2.3 IP의 특징

IP는 목적지까지 패킷을 전달하는 역할을 담당함

패킷이 정상적으로 도착했는지 보장하지 않음

패킷의 순서 역시 보장하지 않음

이러한 신뢰성 문제를 상위 계층인 TCP 등에서 처리할 수 있음


## 3. TCP / UDP

TCP와 UDP는 Transport Layer에서 사용되는 대표적인 프로토콜임

IP가 컴퓨터까지 데이터를 전달한다면 TCP와 UDP는 Port를 이용하여 컴퓨터 내부의 특정 프로세스까지 데이터를 전달함

### 3.1 TCP

TCP는 연결 지향형 프로토콜임

통신 전에 연결을 설정하고 데이터의 신뢰성 있는 전달을 보장하는 것이 특징임

주요 특징은 다음과 같음

- 데이터 전달 보장
- 데이터 순서 보장
- 오류 검출 및 재전송
- 흐름 제어
- 혼잡 제어
- 연결 설정 과정 필요

HTTP/1.1, HTTP/2 등 신뢰성이 중요한 통신에서 사용됨

### 3.2 UDP

UDP는 비연결형 프로토콜임

연결 설정 없이 데이터를 바로 전송하기 때문에 TCP보다 구조가 단순하고 오버헤드가 적음

주요 특징은 다음과 같음

- 연결 과정 없음
- 데이터 전달 여부를 보장하지 않음
- 데이터 순서를 보장하지 않음
- TCP보다 오버헤드가 적음
- 실시간성이 중요한 환경에서 활용됨

DNS 조회, 실시간 스트리밍, 게임 등에서 활용됨

HTTP/3의 기반인 QUIC 역시 UDP 위에서 동작함

### 3.3 TCP와 UDP 비교

| 구분 | TCP | UDP |
|---|---|---|
| 연결 방식 | 연결 지향 | 비연결 |
| 신뢰성 | 높음 | 자체적으로 보장하지 않음 |
| 순서 보장 | 보장 | 보장하지 않음 |
| 재전송 | 지원 | 기본적으로 지원하지 않음 |
| 오버헤드 | 상대적으로 큼 | 상대적으로 작음 |
| 사용 예시 | HTTP/1.1, HTTP/2, SSH | DNS, 게임, 스트리밍, QUIC |


## 4. 3-way / 4-way Handshake

TCP는 데이터를 주고받기 전에 연결을 설정하고 통신이 끝나면 연결을 종료함

### 4.1 3-way Handshake

TCP 연결을 설정하는 과정임

    Client                         Server

       -------- SYN -------->

       <----- SYN + ACK -----

       -------- ACK -------->

           Connection
            Established

#### 1단계 SYN

클라이언트가 서버에게 연결을 요청함

#### 2단계 SYN + ACK

서버가 요청을 받았음을 알리고 자신도 연결을 요청함

#### 3단계 ACK

클라이언트가 서버의 응답을 확인했음을 전달함

이 과정이 완료되면 TCP 연결이 성립됨


### 4.2 4-way Handshake

TCP 연결을 정상적으로 종료하는 과정임

    Client                         Server

       -------- FIN -------->

       <------- ACK ---------

       <------- FIN ---------

       -------- ACK -------->

           Connection
              Closed

#### 1단계 FIN

한쪽에서 데이터 전송을 마쳤다는 의미로 연결 종료를 요청함

#### 2단계 ACK

상대방이 FIN을 받았음을 응답함

#### 3단계 FIN

상대방도 데이터 전송을 완료한 후 연결 종료를 요청함

#### 4단계 ACK

마지막 FIN을 확인했음을 응답함

TCP는 양방향 통신이므로 각각의 방향을 독립적으로 종료하기 때문에 일반적으로 4단계 과정이 필요함


## 5. DNS

DNS는 Domain Name System의 약자로 사람이 사용하는 도메인 이름을 IP 주소로 변환하는 시스템임

    www.example.com
           ↓
          DNS
           ↓
    93.184.216.34

사용자는 IP 주소를 직접 기억하지 않고 도메인 이름을 이용하여 서버에 접근할 수 있음

### 5.1 DNS 조회 과정

일반적인 흐름은 다음과 같음

    사용자
      ↓
    Browser / OS Cache
      ↓
    DNS Resolver
      ↓
    Root DNS
      ↓
    TLD DNS
      ↓
    Authoritative DNS
      ↓
    IP 주소 반환

Root DNS는 `.com`, `.kr` 등을 담당하는 TLD DNS의 위치를 알려주는 역할을 함

TLD DNS는 해당 도메인의 정보를 관리하는 Authoritative DNS의 위치를 알려주는 역할을 함

Authoritative DNS는 실제 도메인에 대응되는 IP 주소 등의 DNS 레코드를 가지고 있음

실제 환경에서는 캐시가 존재하므로 항상 모든 DNS 서버를 순서대로 조회하는 것은 아님


## 6. Port

Port는 하나의 컴퓨터에서 실행 중인 여러 네트워크 프로세스를 구분하기 위한 번호임

IP가 특정 컴퓨터를 찾는 역할이라면 Port는 해당 컴퓨터 내부의 특정 프로세스를 찾는 역할임

    192.168.0.10:8080
          IP      Port
           ↓        ↓
         Server   Spring Boot

Port 번호는 0~65535 범위를 사용함

대표적인 Port는 다음과 같음

| 서비스 | 기본 Port |
|---|---|
| HTTP | 80 |
| HTTPS | 443 |
| SSH | 22 |
| DNS | 53 |
| MySQL | 3306 |
| PostgreSQL | 5432 |

Spring Boot는 기본적으로 8080 Port를 사용함

하나의 IP에서도 Port가 다르면 서로 다른 서버 프로그램이나 서비스를 제공할 수 있음


# Backend 심화

## 7. Socket

Socket은 네트워크에서 프로세스 간 통신을 위한 소프트웨어 인터페이스임

애플리케이션은 Socket을 이용하여 TCP 또는 UDP 통신을 수행함

TCP 서버의 기본적인 흐름은 다음과 같음

    Server

    socket()
       ↓
    bind()
       ↓
    listen()
       ↓
    accept()
       ↓
    read() / write()
       ↓
    close()

`socket()`은 소켓을 생성함

`bind()`는 소켓에 IP와 Port를 할당함

`listen()`은 클라이언트의 연결 요청을 받을 수 있는 상태로 변경함

`accept()`는 클라이언트의 연결 요청을 수락함

연결 이후 데이터를 읽고 쓰며 통신함

백엔드 개발자가 일반적인 Spring Boot 개발에서 Socket API를 직접 다루는 경우는 많지 않음

Tomcat과 같은 WAS가 하위 네트워크 통신을 처리하고 개발자는 HTTP 요청과 응답을 중심으로 처리하는 경우가 일반적임


## 8. Server Connection 관리

서버는 동시에 여러 클라이언트의 연결을 처리해야 함

각 요청마다 무제한으로 Thread나 Connection을 생성하면 서버 자원이 고갈될 수 있음

따라서 서버에서는 Connection과 Thread 등의 자원을 적절하게 관리해야 함

### 8.1 Thread Pool

미리 일정 수의 Thread를 생성하고 요청이 들어오면 사용 가능한 Thread가 요청을 처리하는 방식임

    Client Requests
           ↓
       Web Server
           ↓
    ┌─────────────────┐
    │   Thread Pool   │
    │ Thread 1        │
    │ Thread 2        │
    │ Thread 3        │
    │ Thread 4        │
    └─────────────────┘
           ↓
      Application

Thread를 계속 생성하고 제거하는 비용을 줄일 수 있음

동시에 처리할 수 있는 요청 수를 제한하여 서버 자원을 관리할 수 있음


### 8.2 Connection Pool

DB와 같은 외부 시스템과의 Connection을 미리 생성하여 재사용하는 방식임

    Spring Boot
         ↓
    Connection Pool
     ├─ Connection 1
     ├─ Connection 2
     ├─ Connection 3
     └─ Connection 4
         ↓
      Database

DB Connection 생성에는 네트워크 연결과 인증 등의 비용이 발생함

매 요청마다 Connection을 새로 생성하지 않고 Pool에서 빌려 사용한 후 반환하는 방식이 효율적임

Spring Boot에서는 HikariCP가 기본 JDBC Connection Pool로 사용됨


### 8.3 Keep-Alive

하나의 네트워크 연결을 여러 요청에서 재사용하는 방식임

매 요청마다 TCP 연결을 새롭게 생성하면 Handshake 등의 추가 비용이 발생함

연결을 일정 시간 유지하여 여러 요청에서 재사용하면 이러한 비용을 줄일 수 있음

사용하지 않는 연결을 지나치게 오래 유지하면 서버 자원을 차지할 수 있으므로 적절한 관리가 필요함


## 9. Connection Timeout

Connection Timeout은 서버나 외부 시스템과의 연결을 시도할 때 연결이 완료되기를 기다리는 최대 시간임

    Application
         ↓
    외부 API 연결 시도
         ↓
      응답 없음
         ↓
    Connection Timeout
         ↓
     연결 시도 중단

Timeout을 설정하지 않거나 지나치게 길게 설정하면 장애가 발생한 외부 시스템 때문에 서버의 Thread와 Connection이 장시간 점유될 수 있음

### 9.1 Connection Timeout

상대 서버와 연결이 성립될 때까지 기다리는 최대 시간임

TCP 연결 자체를 만드는 단계와 관련됨

### 9.2 Read Timeout

연결이 성립된 이후 상대방으로부터 데이터를 기다리는 최대 시간임

외부 API 서버의 응답이 지나치게 느린 경우 등에 발생할 수 있음

### 9.3 Connection Pool Timeout

Connection Pool에서 사용 가능한 Connection을 얻기 위해 기다리는 최대 시간임

모든 Connection이 사용 중일 경우 일정 시간 동안 기다린 후 실패하도록 설정할 수 있음


## 10. 백엔드 요청의 전체 흐름

사용자가 Spring Boot 서버의 API를 호출한다고 가정한 흐름임

    사용자
      ↓
    https://example.com/api/users
      ↓
    DNS
    도메인 → IP 주소 확인
      ↓
    IP
    목적지 서버 확인
      ↓
    TCP
    3-way Handshake를 통한 연결
      ↓
    Port 443
    서버의 HTTPS 서비스 접근
      ↓
    TLS
    암호화된 통신 연결 구성
      ↓
    HTTP Request
    GET /api/users
      ↓
    Web Server / WAS
      ↓
    Thread Pool
    요청 처리 Thread 할당
      ↓
    Spring Boot
    Controller → Service → Repository
      ↓
    Connection Pool
    DB Connection 획득
      ↓
    Database
      ↓
    HTTP Response
      ↓
    Client

결국 백엔드 네트워크에서는 `IP → Port → TCP → Socket → HTTP → WAS → Application`의 연결 관계를 이해하는 것이 중요함
