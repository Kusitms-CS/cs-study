# 5주차 - HTTP / HTTPS

## 1. HTTP

> HTTP(HyperText Transfer Protocol)는 Client와 Server가 데이터를 주고받기 위해 사용하는 Application Layer 프로토콜이다.

웹 브라우저가 서버에 데이터를 요청하면 Server가 이에 대한 응답을 보내는 **Request / Response 방식**으로 동작한다.

```text
Client                              Server
  │                                   │
  │ -------- HTTP Request ----------> │
  │                                   │
  │ <------- HTTP Response ---------- │
  │                                   │
```

HTTP는 Application Layer의 프로토콜이므로 실제 데이터 전송은 하위 계층의 프로토콜을 이용한다.

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

### HTTP Request

Client가 Server에 보내는 HTTP 메시지이다.

HTTP/1.1 메시지를 단순화하면 다음과 같은 형태이다.

```http
POST /users HTTP/1.1
Host: example.com
Content-Type: application/json

{
  "name": "Joon"
}
```

크게 다음과 같이 구성된다.

```text
Start Line
→ Method, 요청 대상, HTTP Version

Header
→ 요청에 대한 부가 정보

Body
→ Server에 전달할 데이터
```

예를 들어 위 요청에서는

```text
POST
→ HTTP Method

/users
→ 요청 대상

Content-Type: application/json
→ Body 데이터의 형식

Body
→ 실제 전달할 데이터
```

를 나타낸다.

### HTTP Response

Server가 Client의 요청을 처리한 뒤 보내는 HTTP 메시지이다.

```http
HTTP/1.1 201 Created
Content-Type: application/json

{
  "id": 1,
  "name": "Joon"
}
```

응답 역시 크게 다음과 같이 구성된다.

```text
Status Line
→ HTTP Version + Status Code

Header
→ 응답에 대한 부가 정보

Body
→ 실제 응답 데이터
```

HTTP는 이처럼 **Client가 Request를 보내고 Server가 Response를 반환하는 방식**을 기본으로 한다.

## 2. Stateless

> Stateless는 Server가 이전 요청의 상태를 기본적으로 기억하지 않고 각각의 요청을 독립적으로 처리하는 특성이다.

예를 들어 다음 요청이 들어왔다고 가정하자.

```text
Request 1
→ 상품 조회

Request 2
→ 주문 요청
```

HTTP 자체에서는 Request 2가 Request 1을 보낸 사용자와 같은 사용자의 요청인지 자동으로 기억하지 않는다.

따라서 각 요청은 처리에 필요한 정보를 스스로 포함해야 한다.

```text
Request A
→ 사용자 정보 + 요청 정보

Request B
→ 사용자 정보 + 요청 정보
```

### Stateless의 장점

Server가 Client의 이전 요청 상태에 강하게 의존하지 않으면 여러 Server가 요청을 처리하기 쉬워진다.

```text
                ┌→ Server A
Client → LB ────┼→ Server B
                └→ Server C
```

Request마다 필요한 정보가 있다면 이전 요청을 처리했던 Server가 아니더라도 요청을 처리할 수 있다.

따라서 서버 확장과 Load Balancing에 유리하다.

### 그러면 로그인 상태는 어떻게 유지할까?

실제 웹 서비스에서는 로그인처럼 여러 요청에 걸쳐 유지해야 하는 상태가 있다.

이때 Cookie, Session, Token 등의 방법을 사용할 수 있다.

예를 들어 Session 방식에서는 다음과 같이 동작할 수 있다.

```text
로그인
  ↓
Server가 Session 생성
  ↓
Session ID 전달
  ↓
Client가 Cookie 등에 Session ID 저장
  ↓
다음 Request에 Session ID 포함
  ↓
Server가 Session 조회
```

즉, HTTP가 Stateless라고 해서 **애플리케이션이 어떠한 상태도 가질 수 없다는 의미는 아니다.**

HTTP 자체는 Stateless하지만 필요한 경우 별도의 방법을 이용해 상태를 관리할 수 있다.

## 3. Method

> HTTP Method는 Client가 Server의 Resource에 대해 어떤 작업을 원하는지 나타낸다.

대표적으로 GET, POST, PUT, PATCH, DELETE가 있다.

| Method | 주요 용도 |
|---|---|
| GET | Resource 조회 |
| POST | 데이터 처리, Resource 생성 |
| PUT | Resource 전체 교체 |
| PATCH | Resource 일부 수정 |
| DELETE | Resource 삭제 |

### GET

Resource를 조회할 때 주로 사용한다.

```http
GET /users/1 HTTP/1.1
```

```text
/users/1
→ ID가 1인 User 조회
```

GET 요청은 Server의 상태를 변경하기 위한 용도로 사용하지 않는 것이 원칙이다.

### POST

Server에 데이터를 전달하여 처리를 요청할 때 사용한다.

Resource 생성에 많이 사용된다.

```http
POST /users HTTP/1.1
Content-Type: application/json

{
  "name": "Joon"
}
```

```text
POST /users
→ 새로운 User 생성
```

POST는 반드시 생성에만 사용하는 Method는 아니며, 요청 데이터를 기반으로 Server에서 특정 처리를 수행하도록 할 때도 사용할 수 있다.

### PUT

Resource를 **전체적으로 교체**할 때 주로 사용한다.

기존 데이터가 다음과 같다고 가정하자.

```json
{
  "name": "Joon",
  "age": 25
}
```

다음 PUT 요청을 보낸다면

```http
PUT /users/1

{
  "name": "Kim"
}
```

Resource 전체를 요청 내용으로 교체하는 의미이므로 기존 `age`가 유지된다고 가정해서는 안 된다.

### PATCH

Resource의 **일부분을 수정**할 때 사용한다.

```http
PATCH /users/1

{
  "name": "Kim"
}
```

기존 Resource에서 `name`만 변경하는 식으로 사용할 수 있다.

따라서 개념적으로 다음과 같이 구분할 수 있다.

```text
PUT
→ Resource 전체 교체

PATCH
→ Resource 일부 수정
```

### DELETE

Resource를 삭제할 때 사용한다.

```http
DELETE /users/1 HTTP/1.1
```

```text
/users/1
→ 해당 User 삭제
```

### Safe Method

HTTP에서 **Safe**는 요청이 Server의 상태를 변경하도록 의도되지 않았다는 의미이다.

대표적으로 GET은 Safe Method이다.

```text
GET /users/1
→ 조회

여러 번 호출해도
Resource를 변경하려는 요청이 아님
```

여기서 Safe는 보안상 안전하다는 의미가 아니다.

### Idempotent

> 같은 요청을 여러 번 수행해도 한 번 수행한 것과 Server의 최종 상태가 같은 성질이다.

예를 들어 DELETE를 생각해보자.

```text
DELETE /users/1

1번 호출
→ User 삭제

다시 호출
→ 이미 User가 없음

최종 상태
→ User가 존재하지 않음
```

응답 결과는 달라질 수 있지만 Server의 최종 상태는 동일하므로 DELETE는 멱등성을 가진다.

대표적으로 다음과 같이 구분할 수 있다.

| Method | Safe | Idempotent |
|---|---|---|
| GET | O | O |
| POST | X | X |
| PUT | X | O |
| DELETE | X | O |

PATCH는 요청의 구현 방식에 따라 멱등성이 보장될 수도 있고 그렇지 않을 수도 있다.

멱등성은 특히 **네트워크 오류로 같은 요청을 다시 보내야 하는 상황**에서 중요하다.

```text
요청 전송
   ↓
응답을 받지 못함
   ↓
요청이 처리됐는지 불확실
   ↓
재시도
```

이때 요청이 멱등하다면 같은 요청을 다시 수행하더라도 최종 Resource 상태가 달라지지 않도록 설계할 수 있다.

## 4. Status Code

> HTTP Status Code는 Server가 Client의 요청을 어떻게 처리했는지 나타내는 3자리 숫자이다.

Status Code는 첫 번째 숫자에 따라 크게 다섯 종류로 나뉜다.

| 범위 | 의미 |
|---|---|
| 1xx | Informational |
| 2xx | Success |
| 3xx | Redirection |
| 4xx | Client Error |
| 5xx | Server Error |

### 2xx - Success

요청이 정상적으로 처리되었음을 나타낸다.

**200 OK**

가장 일반적인 성공 응답이다.

```text
GET /users/1
        ↓
200 OK
```

**201 Created**

새로운 Resource가 성공적으로 생성되었을 때 사용할 수 있다.

```text
POST /users
      ↓
User 생성
      ↓
201 Created
```

**204 No Content**

요청은 성공했지만 Response Body로 전달할 내용이 없을 때 사용한다.

### 3xx - Redirection

요청을 완료하기 위해 다른 위치로 이동하는 등의 추가 동작이 필요한 경우 사용한다.

대표적으로 `301 Moved Permanently`, `302 Found`, `304 Not Modified` 등이 있다.

```text
Client
  ↓
GET /old
  ↓
301 Moved Permanently
Location: /new
  ↓
Client가 /new 요청
```

`304 Not Modified`는 HTTP Cache와 함께 자주 사용된다.

### 4xx - Client Error

Client의 요청에 문제가 있을 때 사용한다.

**400 Bad Request**

요청 형식이나 값 등이 올바르지 않아 Server가 요청을 처리할 수 없는 경우 사용할 수 있다.

**401 Unauthorized**

인증(Authentication)이 필요한 요청에서 유효한 인증 정보가 없는 경우 사용한다.

```text
Authentication
→ "누구인가?"
```

이름 때문에 헷갈리기 쉽지만 `401`은 주로 **인증 문제**와 관련된다.

**403 Forbidden**

Server가 요청을 이해했지만 해당 Resource에 접근할 권한이 없는 경우 사용한다.

```text
Authorization
→ "이 작업을 할 권한이 있는가?"
```

```text
401
→ 인증되지 않음

403
→ 접근 권한이 없음
```

**404 Not Found**

요청한 Resource를 찾을 수 없는 경우 사용한다.

### 5xx - Server Error

Server가 정상적인 요청을 처리하는 과정에서 문제가 발생했을 때 사용한다.

**500 Internal Server Error**

Server 내부에서 예상하지 못한 오류가 발생한 경우 사용한다.

**502 Bad Gateway**

Gateway나 Proxy 역할을 하는 Server가 상위 Server로부터 정상적인 응답을 받지 못한 경우 발생할 수 있다.

```text
Client
  ↓
Gateway / Proxy
  ↓
Upstream Server
       X
  ↓
502 Bad Gateway
```

**503 Service Unavailable**

Server가 일시적으로 요청을 처리할 수 없는 상태에서 사용한다.

예를 들어 서버 과부하나 일시적인 점검 등이 있을 수 있다.

### 4xx와 5xx의 차이

가장 크게는 **문제의 원인이 어디에 있는가**로 이해할 수 있다.

```text
4xx
→ Client 요청 측의 문제
→ 요청을 수정해야 할 가능성이 큼

5xx
→ Server 처리 과정의 문제
→ 같은 요청도 Server 상태에 따라 성공할 수 있음
```

다만 실제 API에서는 상황에 맞는 Status Code를 선택하는 것이 중요하다.

### 지금까지의 흐름

```text
Client
   ↓
HTTP Request
   │
   ├─ Method
   ├─ Header
   └─ Body
   ↓
Server
   ↓
요청 처리
   ↓
HTTP Response
   │
   ├─ Status Code
   ├─ Header
   └─ Body
   ↓
Client
```

HTTP는 Request와 Response를 기반으로 통신하고 기본적으로 Stateless한 특성을 가진다.

Client는 **HTTP Method로 원하는 작업을 표현**하고, Server는 요청을 처리한 결과를 **Status Code로 표현**한다고 연결해서 이해하면 된다.

## 5. HTTP/1.1 · HTTP/2 · HTTP/3

HTTP는 웹의 발전과 함께 성능을 개선하기 위해 여러 버전으로 발전해왔다.

큰 흐름은 다음과 같이 이해하면 된다.

```text
HTTP/1.1
→ TCP Connection 재사용

HTTP/2
→ 하나의 Connection에서 여러 요청을 효율적으로 처리

HTTP/3
→ TCP 대신 QUIC을 사용해 전송 계층의 한계까지 개선
```

### HTTP/1.1

HTTP/1.1의 중요한 특징 중 하나는 **Persistent Connection**이다.

HTTP 요청마다 TCP Connection을 새로 만든다고 가정해보자.

```text
TCP 연결
  ↓
Request / Response
  ↓
TCP 종료

TCP 연결
  ↓
Request / Response
  ↓
TCP 종료
```

이 방식은 요청마다 TCP 3-way Handshake가 필요하기 때문에 비효율적이다.

HTTP/1.1에서는 기본적으로 하나의 TCP Connection을 여러 HTTP 요청에 재사용하는 **Persistent Connection**을 사용한다.

```text
TCP Connection
      ↓
Request 1 / Response 1
      ↓
Request 2 / Response 2
      ↓
Request 3 / Response 3
      ↓
Connection 종료
```

이를 흔히 **HTTP Keep-Alive**라고도 한다.

하지만 하나의 Connection에서 요청과 응답을 순차적으로 처리하면 앞선 요청의 응답이 늦어질 때 뒤의 요청도 기다려야 하는 문제가 발생할 수 있다.

HTTP/1.1에는 여러 요청을 응답을 기다리지 않고 연속으로 보내는 **Pipelining**도 있지만 여러 한계 때문에 널리 활용되지는 않았다.

### HTTP/2

HTTP/2의 핵심적인 개선점은 **Multiplexing**이다.

하나의 TCP Connection 안에서 여러 요청과 응답을 여러 Stream으로 나누어 동시에 처리할 수 있다.

```text
하나의 TCP Connection

Stream 1 ─ Request A ─ Response A
Stream 2 ─ Request B ─ Response B
Stream 3 ─ Request C ─ Response C
```

HTTP/1.1에서는 여러 요청을 효율적으로 처리하기 위해 여러 TCP Connection을 사용하는 경우가 많았지만, HTTP/2에서는 하나의 Connection에서 여러 Stream을 동시에 처리할 수 있다.

HTTP/2는 HTTP 메시지를 **Binary Frame** 단위로 나누어 전송한다.

```text
HTTP Message
     ↓
Binary Frame
     ↓
Stream
     ↓
TCP Connection
```

이러한 구조를 통해 여러 요청과 응답의 Frame을 하나의 Connection에서 섞어서 전송할 수 있다.

### HTTP/2에도 문제가 있을까?

HTTP/2는 Application Layer에서는 여러 Stream을 독립적으로 처리하지만 그 아래에서는 하나의 TCP Connection을 사용한다.

```text
HTTP/2

Stream A ─┐
Stream B ─┼→ TCP Connection
Stream C ─┘
```

TCP는 데이터를 순서대로 전달해야 한다.

따라서 TCP Packet 일부가 유실되면 해당 데이터를 재전송하고 순서를 맞출 때까지 뒤의 데이터 전달도 영향을 받을 수 있다.

```text
TCP

Packet 1 → 도착
Packet 2 → 유실
Packet 3 → 도착
Packet 4 → 도착

        ↓

Packet 2 재전송 대기
        ↓

뒤의 데이터 전달에도 영향
```

이를 TCP 수준의 **Head-of-Line Blocking(HOL Blocking)** 문제라고 한다.

HTTP/2가 HTTP 수준의 요청 처리 문제를 크게 개선했지만, TCP 자체의 순서 보장 특성으로 인한 HOL Blocking은 남아 있는 것이다.

### HTTP/3

HTTP/3는 이러한 문제를 개선하기 위해 **QUIC**을 사용한다.

HTTP/1.1과 HTTP/2는 TCP 기반이지만 HTTP/3는 QUIC을 사용하며, QUIC은 UDP 위에서 동작한다.

```text
HTTP/1.1
    ↓
   TCP

HTTP/2
    ↓
   TCP

HTTP/3
    ↓
   QUIC
    ↓
   UDP
```

QUIC은 여러 Stream을 독립적으로 처리할 수 있다.

따라서 하나의 Stream에서 패킷 유실이 발생하더라도 다른 Stream까지 동일하게 막히는 TCP 기반 HTTP/2의 문제를 줄일 수 있다.

```text
QUIC

Stream A → Packet 유실 → 대기

Stream B → 계속 처리
Stream C → 계속 처리
```

또한 QUIC은 TLS 1.3을 프로토콜에 통합하여 연결 설정 과정도 효율적으로 구성한다.

### 버전 비교

| 구분 | HTTP/1.1 | HTTP/2 | HTTP/3 |
|---|---|---|---|
| 기반 전송 | TCP | TCP | QUIC / UDP |
| Persistent Connection | O | O | O |
| Multiplexing | X | O | O |
| 전송 형식 | Text 기반 메시지 | Binary Frame | Binary 기반 |
| 주요 개선 | Connection 재사용 | 하나의 Connection에서 여러 Stream 처리 | QUIC 기반 전송 |

큰 흐름은 다음과 같이 기억하면 된다.

```text
HTTP/1.1
→ TCP 연결을 재사용하자

HTTP/2
→ 하나의 연결에서 여러 요청을 동시에 처리하자

HTTP/3
→ TCP에서 발생하는 전송 계층의 한계도 개선하자
```

## 6. Keep-Alive

> HTTP Keep-Alive는 하나의 Connection을 여러 HTTP Request / Response에 재사용하는 방식이다.

4주차에서 TCP Connection 관점으로 살펴봤다면, 이번에는 HTTP 관점에서 이해하면 된다.

### Keep-Alive가 없다면

요청마다 TCP Connection을 새로 생성해야 한다.

```text
HTTP Request 1

TCP 3-way Handshake
        ↓
Request / Response
        ↓
TCP Connection 종료


HTTP Request 2

TCP 3-way Handshake
        ↓
Request / Response
        ↓
TCP Connection 종료
```

웹 페이지 하나를 불러오는 데도 HTML, JavaScript, CSS, Image 등 여러 Resource가 필요할 수 있다.

매번 Connection을 생성한다면 그만큼 연결 설정 비용이 반복된다.

### Keep-Alive를 사용하면

기존 TCP Connection을 다음 요청에서도 사용할 수 있다.

```text
TCP Connection 생성
        ↓
GET /index.html
        ↓
Response
        ↓
GET /app.js
        ↓
Response
        ↓
GET /image.png
        ↓
Response
        ↓
Connection 종료
```

따라서 TCP Connection 생성과 종료를 반복하는 비용을 줄일 수 있다.

HTTP/1.1에서는 Persistent Connection이 기본 동작이다.

### Connection을 계속 유지하면 좋은 것 아닐까?

Connection 역시 Server의 자원을 사용한다.

따라서 사용하지 않는 Connection까지 무한정 유지하는 것은 좋지 않다.

```text
Client A ─ Connection 유지
Client B ─ Connection 유지
Client C ─ Connection 유지
Client D ─ Connection 유지
              ↓
      Server 자원 사용
```

그래서 Server는 일정 시간 동안 추가 요청이 없는 Connection을 종료하는 등의 정책을 사용할 수 있다.

```text
HTTP 요청/응답
      ↓
Connection 유지
      ↓
일정 시간 동안 요청 없음
      ↓
Keep-Alive Timeout
      ↓
Connection 종료
```

즉, Keep-Alive의 핵심은 **Connection을 영원히 유지하는 것이 아니라 필요한 범위에서 재사용하여 연결 비용을 줄이는 것**이다.

### TCP Keepalive와 HTTP Keep-Alive는 다르다

이름이 비슷하지만 서로 다른 개념이다.

**HTTP Keep-Alive**

```text
HTTP Request / Response에서
하나의 TCP Connection을 재사용
```

**TCP Keepalive**

```text
오랫동안 통신이 없는 TCP Connection이
여전히 유효한지 확인하기 위한 TCP 기능
```

따라서 면접에서 `Keep-Alive`라는 표현이 나오면 HTTP Connection 재사용을 의미하는지, TCP Keepalive를 의미하는지 구분해서 보는 것이 좋다.

### HTTP 버전과 연결해서 보기

```text
HTTP/1.1
→ Persistent Connection
→ TCP Connection 재사용

HTTP/2
→ 하나의 TCP Connection 재사용
→ 여러 Stream을 Multiplexing

HTTP/3
→ QUIC Connection
→ 여러 Stream을 Multiplexing
```

HTTP가 발전하면서 단순히 Connection을 재사용하는 것에서 더 나아가 **하나의 Connection 안에서 여러 요청을 효율적으로 처리하는 방향**으로 발전했다고 이해하면 된다.

## 7. HTTPS

> HTTPS(HTTP Secure)는 HTTP 통신에 TLS를 적용하여 데이터를 안전하게 주고받을 수 있도록 만든 방식이다.

HTTP 자체는 데이터를 암호화하지 않는다.

따라서 HTTP로 통신하면 네트워크 중간에서 데이터를 확인하거나 변조할 위험이 있다.

```text
Client
  │
  │ HTTP
  │ ID: joon
  │ Password: 1234
  ↓
Server

→ 중간에서 데이터가 노출될 가능성
```

HTTPS에서는 HTTP 메시지를 **TLS를 통해 암호화한 뒤 전송**한다.

```text
HTTP
  ↓
TLS
  ↓
TCP
  ↓
IP
```

HTTP/1.1과 HTTP/2를 기준으로 보면 다음과 같은 구조이다.

```text
Client
  ↓
HTTP Request
  ↓
TLS로 보호
  ↓
TCP를 통해 전송
  ↓
Server
```

HTTP/3에서는 QUIC이 TLS 1.3의 보안 기능을 통합하여 사용한다.

### HTTPS가 제공하는 것

HTTPS를 이해할 때는 크게 세 가지를 기억하면 된다.

```text
HTTPS

├─ 기밀성
│   → 데이터를 암호화
│
├─ 무결성
│   → 데이터가 변조되지 않았는지 확인
│
└─ 인증
    → 통신 상대가 신뢰할 수 있는 서버인지 확인
```

**기밀성(Confidentiality)**

통신 내용을 암호화하여 제3자가 내용을 쉽게 확인할 수 없도록 한다.

**무결성(Integrity)**

전송 중 데이터가 변경되었는지 확인할 수 있도록 한다.

**인증(Authentication)**

인증서를 이용해 Client가 접속한 Server의 신원을 확인할 수 있도록 한다.

### HTTP와 HTTPS

| 구분 | HTTP | HTTPS |
|---|---|---|
| 암호화 | X | O |
| 무결성 보호 | X | O |
| 서버 인증 | X | O |
| 기본 Port | 80 | 443 |
| 보안 계층 | 없음 | TLS |

그렇다면 HTTPS는 실제로 데이터를 어떻게 암호화하고 Server가 진짜인지 어떻게 확인할까?

이를 담당하는 핵심 기술이 **TLS**이다.

## 8. TLS

> TLS(Transport Layer Security)는 네트워크 통신에서 암호화, 무결성, 인증을 제공하는 보안 프로토콜이다.

HTTPS에서는 TLS를 이용해 HTTP 데이터를 보호한다.

TLS를 이해하려면 먼저 **대칭키 암호화와 비대칭키 암호화**의 차이를 알아야 한다.

### 대칭키 암호화

> 암호화와 복호화에 같은 Secret Key를 사용하는 방식이다.

```text
평문
 ↓
Secret Key로 암호화
 ↓
암호문
 ↓
Secret Key로 복호화
 ↓
평문
```

Client와 Server가 같은 Secret Key를 가지고 있다면 이를 이용해 데이터를 빠르게 암호화하고 복호화할 수 있다.

하지만 문제가 있다.

```text
Client                    Server
  │                         │
  │ Secret Key 전달?        │
  │ ----------------------> │
  │                         │
```

Secret Key 자체를 안전하게 공유해야 한다.

누군가 이 Key를 가로채면 암호화된 데이터도 복호화할 수 있기 때문이다.

### 비대칭키 암호화

> 서로 다른 Public Key와 Private Key를 사용하는 방식이다.

```text
Public Key
→ 외부에 공개 가능

Private Key
→ 소유자가 안전하게 보관
```

Public Key와 Private Key는 서로 수학적으로 연결되어 있다.

비대칭키 방식은 키 교환과 인증 등에 활용할 수 있지만 대칭키 방식에 비해 연산 비용이 크다.

따라서 TLS는 **비대칭키 기반 기술만으로 모든 HTTP 데이터를 암호화하지 않는다.**

TLS Handshake를 통해 안전하게 통신에 사용할 Key를 합의한 뒤 실제 애플리케이션 데이터는 효율적인 **대칭키 암호화**를 이용한다.

```text
TLS Handshake
     ↓
비대칭키 기반 기술 등을 이용
     ↓
안전하게 Key 합의
     ↓
대칭키 기반 암호화
     ↓
HTTP 데이터 전송
```

정확한 Key 교환 방식은 TLS 버전과 Cipher Suite에 따라 달라질 수 있지만, 면접에서는 **Handshake에서 안전하게 Key를 합의하고 실제 데이터 통신에는 대칭키 암호화를 사용한다**는 흐름을 이해하는 것이 중요하다.

### 인증서는 왜 필요할까?

암호화만 한다고 해서 현재 통신하는 Server가 진짜 Server라는 보장은 없다.

공격자가 중간에서 자신의 Public Key를 Server의 것처럼 전달한다면 Client가 공격자와 암호화 통신을 시작할 수도 있다.

이를 **Man-In-The-Middle Attack(MITM)**이라고 한다.

```text
Client
   ↓
Attacker
   ↓
Server
```

그래서 HTTPS에서는 **Digital Certificate(디지털 인증서)**를 이용해 Server의 신원을 확인한다.

인증서에는 대표적으로 다음과 같은 정보가 포함될 수 있다.

```text
Certificate

├─ Domain 정보
├─ Server의 Public Key 관련 정보
├─ 유효 기간
├─ 발급자 정보
└─ CA의 Digital Signature
```

### CA

> CA(Certificate Authority)는 인증서를 발급하고 인증서의 신뢰성을 보증하는 기관이다.

브라우저나 운영체제에는 신뢰할 수 있는 CA에 대한 정보가 포함되어 있다.

Server가 인증서를 전달하면 Client는 인증서의 서명과 신뢰 체인 등을 검증한다.

```text
Server
  ↓
Certificate 전달
  ↓
Client
  ↓
CA 서명 / 신뢰 체인 확인
Domain 확인
유효 기간 확인
  ↓
Server 신뢰 여부 판단
```

따라서 공격자가 단순히 Server의 인증서를 임의로 만들어 전달한다고 해서 정상적인 HTTPS 인증을 통과할 수 있는 것은 아니다.

### TLS Handshake

TLS Handshake의 세부 과정은 TLS 버전에 따라 다르다.

면접 준비에서는 모든 메시지 이름을 외우기보다 다음 흐름을 이해하는 것이 중요하다.

```text
Client                         Server
  │                              │
  │ ---- TLS 연결 시작 --------> │
  │                              │
  │ <---- Certificate ---------- │
  │                              │
  │   인증서 검증                │
  │                              │
  │ <---- Key 합의 과정 -------> │
  │                              │
  │   Session Key 생성/합의      │
  │                              │
  │ ===== 암호화 통신 =========> │
  │ <=========================== │
```

크게 보면 다음과 같다.

```text
1. Client와 Server가 TLS 통신에 필요한 정보 협상

2. Server가 Certificate 제공

3. Client가 Certificate 검증

4. Key Exchange를 통해
   통신에 사용할 Key 합의

5. 합의된 Key를 기반으로
   암호화된 Application Data 통신
```

현대적인 TLS에서는 일반적으로 각 연결에 사용할 대칭키를 직접 네트워크로 보내는 것이 아니라 **Key Exchange 과정을 통해 양쪽이 같은 Secret을 만들어낸다.**

### 인증서는 암호화를 위한 것일까?

인증서의 핵심 역할은 단순히 데이터를 암호화하는 것이 아니라 **Public Key와 Server의 신원을 연결하고 이를 신뢰할 수 있도록 하는 것**이다.

```text
Certificate
      ↓
"이 Public Key는
example.com의 것이다"
      ↓
CA가 서명
      ↓
Client가 검증
```

이를 통해 Client는 자신이 의도한 Server와 통신하고 있는지 확인할 수 있다.

### HTTPS 전체 흐름

HTTP와 TLS를 함께 보면 다음과 같이 정리할 수 있다.

```text
Client
  ↓
Server와 연결
  ↓
TLS Handshake
  │
  ├─ Server 인증
  └─ 암호화에 사용할 Key 합의
  ↓
안전한 통신 채널 생성
  ↓
HTTP Request
  ↓
TLS로 암호화
  ↓
Server
  ↓
TLS로 복호화
  ↓
HTTP Request 처리
```

즉,

```text
HTTP
→ 어떤 Request / Response를 주고받을 것인가?

TLS
→ 그 데이터를 어떻게 안전하게 전달할 것인가?

HTTPS
→ HTTP + TLS
```

라고 연결해서 이해하면 된다.

HTTPS의 핵심은 단순히 **HTTP 데이터를 암호화한다**에서 끝나는 것이 아니라, TLS를 통해 **기밀성, 무결성, 인증**을 제공한다는 것이다.

## 9. REST API

> REST는 Resource를 중심으로 HTTP의 특성을 활용해 시스템 간 통신을 설계하는 Architectural Style이다.

REST는 특정 프로토콜이나 라이브러리가 아니다.

HTTP를 사용할 때 **Resource를 URI로 표현하고, HTTP Method를 통해 Resource에 대한 행위를 나타내는 방식**으로 많이 활용된다.

예를 들어 User라는 Resource가 있다고 생각해보자.

```text
/users
/users/1
```

URI에서는 가능하면 Resource를 표현하고, 어떤 작업을 할지는 HTTP Method로 나타낸다.

```text
GET    /users/1
→ User 조회

POST   /users
→ User 생성

PUT    /users/1
→ User 전체 수정

PATCH  /users/1
→ User 일부 수정

DELETE /users/1
→ User 삭제
```

### URI는 Resource를 표현한다

REST API에서는 URI에 동작을 직접 표현하기보다 **Resource를 중심으로 설계하는 것**이 일반적이다.

예를 들어 다음과 같은 API가 있다고 해보자.

```text
GET /getUser/1
POST /createUser
POST /deleteUser/1
```

이보다는 다음처럼 Resource를 URI로 표현하고 동작은 Method로 구분하는 방식이 REST의 관점에 더 잘 맞는다.

```text
GET    /users/1
POST   /users
DELETE /users/1
```

```text
URI
→ 무엇(Resource)을 대상으로 하는가?

Method
→ 무엇을 할 것인가?
```

따라서 URI에는 보통 동사보다 명사를 사용하는 것이 자연스럽다.

### HTTP Status Code 활용

REST API에서는 요청 처리 결과를 HTTP Status Code를 통해 표현할 수 있다.

```text
POST /users
→ 201 Created

GET /users/999
→ 404 Not Found

인증되지 않은 요청
→ 401 Unauthorized

권한이 없는 요청
→ 403 Forbidden

Server 내부 오류
→ 500 Internal Server Error
```

항상 특정 상황에 하나의 Status Code만 사용할 수 있는 것은 아니며, API의 의미와 정책에 맞게 적절한 Status Code를 선택해야 한다.

### Stateless

REST의 중요한 제약 중 하나도 **Stateless**이다.

각 요청은 Server가 요청을 처리하는 데 필요한 정보를 포함해야 하며, Server는 이전 요청의 Client Context에 의존해서 다음 요청을 처리하지 않는 것이 기본 원칙이다.

```text
Request 1
→ 필요한 정보 포함
→ Server 처리

Request 2
→ 필요한 정보 포함
→ Server 처리
```

앞에서 살펴본 HTTP의 Stateless 특성을 REST에서도 활용한다고 연결해서 이해하면 된다.

### REST API와 JSON

REST가 반드시 JSON을 사용해야 하는 것은 아니다.

REST는 API를 설계하는 Architectural Style이고 JSON은 데이터를 표현하는 형식이다.

다만 실제 웹 API에서는 JSON을 많이 사용하기 때문에 다음과 같은 형태를 자주 볼 수 있다.

```http
POST /users HTTP/1.1
Content-Type: application/json

{
  "name": "Joon",
  "age": 25
}
```

```text
REST
→ API 설계 방식

HTTP
→ 통신 프로토콜

JSON
→ 데이터 표현 형식
```

서로 관련되어 자주 사용되지만 같은 개념은 아니다.

### RESTful API

REST의 원칙을 잘 따르도록 설계한 API를 흔히 **RESTful API**라고 부른다.

실무에서는 REST의 모든 제약을 엄격하게 만족하기보다 HTTP Method, Resource 중심 URI, Status Code 등을 적절하게 활용한 HTTP API를 REST API라고 부르는 경우도 많다.

면접에서는 REST를 단순히 `URL을 예쁘게 만드는 규칙`으로 이해하기보다 다음과 같이 정리하는 것이 좋다.

```text
Resource
→ URI로 표현

행위
→ HTTP Method로 표현

처리 결과
→ HTTP Status Code로 표현

요청
→ Stateless하게 처리
```

## 10. Connection / Timeout

4주차에서는 Connect Timeout과 Read Timeout의 기본 개념을 살펴봤다.

이번에는 **HTTP Client가 다른 Server를 호출하는 상황**을 기준으로 Connection 관리와 Timeout을 연결해서 살펴보자.

예를 들어 우리 Server가 외부 결제 API를 호출한다고 가정하자.

```text
Client
  ↓
우리 Server
  ↓
외부 결제 API
```

우리 Server 역시 외부 API 입장에서는 Client가 된다.

### 매번 새로운 Connection을 만든다면?

외부 API를 호출할 때마다 새로운 TCP Connection을 만든다면 연결 설정 비용이 반복된다.

HTTPS라면 TLS Handshake 비용도 고려해야 한다.

```text
API 호출
   ↓
TCP Connection 생성
   ↓
TLS Handshake
   ↓
HTTP Request / Response
   ↓
Connection 종료

다음 API 호출
   ↓
다시 Connection 생성
...
```

요청이 많다면 이러한 연결 설정을 계속 반복하는 것은 비효율적일 수 있다.

### Connection 재사용

기존 Connection을 재사용하면 매 요청마다 새로운 연결을 생성하는 비용을 줄일 수 있다.

```text
Connection 생성
      ↓
Request 1 / Response 1
      ↓
Request 2 / Response 2
      ↓
Request 3 / Response 3
      ↓
Connection 종료
```

HTTP Keep-Alive가 이러한 Connection 재사용을 가능하게 한다.

HTTP Client에서는 여기에 더해 여러 Connection을 관리하고 재사용하기 위해 **Connection Pool**을 사용할 수 있다.

```text
HTTP Client
     ↓
┌───────────────────┐
│  Connection Pool  │
│                   │
│  Connection 1     │
│  Connection 2     │
│  Connection 3     │
└───────────────────┘
     ↓
External Server
```

요청이 들어오면 사용 가능한 Connection을 빌려 사용하고, 요청이 끝난 뒤 재사용할 수 있도록 반환하는 방식이다.

### Connection Pool을 왜 사용할까?

Connection을 재사용하면 TCP 연결과 TLS Handshake 등을 매번 반복하는 비용을 줄일 수 있다.

또한 외부 Server로 생성할 수 있는 Connection의 수를 제한하여 자원 사용량을 관리할 수도 있다.

```text
요청
 ↓
Connection Pool
 ↓
사용 가능한 Connection 획득
 ↓
HTTP 요청
 ↓
응답
 ↓
Connection 반환
```

하지만 Pool의 Connection이 모두 사용 중이라면 새로운 요청은 사용 가능한 Connection이 생길 때까지 기다려야 할 수 있다.

```text
Connection 1 → 사용 중
Connection 2 → 사용 중
Connection 3 → 사용 중

새로운 Request
      ↓
사용 가능한 Connection 대기
```

따라서 Connection 관리에서도 적절한 크기와 대기 시간 설정이 중요하다.

### Timeout

외부 API 호출에서는 여러 단계에서 지연이 발생할 수 있다.

```text
HTTP 요청

1. Connection을 얻기 위해 대기

2. 상대 Server와 Connection 설정

3. Request 전송

4. Response 대기
```

각 단계에서 무한정 기다리지 않도록 Timeout을 설정할 수 있다.

대표적으로 다음과 같이 구분해서 생각할 수 있다.

| Timeout | 의미 |
|---|---|
| Connection Pool 대기 Timeout | Pool에서 사용할 Connection을 얻기까지 기다리는 시간 |
| Connect Timeout | 상대 Server와 Connection을 설정하기까지 기다리는 시간 |
| Read Timeout | 연결 후 데이터를 읽기 위해 기다리는 시간 |

구체적인 Timeout의 이름과 동작은 사용하는 HTTP Client 라이브러리에 따라 조금씩 다를 수 있다.

### Timeout이 너무 길다면?

외부 Server에 장애가 발생했다고 가정해보자.

```text
Request A → 외부 API 응답 대기
Request B → 외부 API 응답 대기
Request C → 외부 API 응답 대기
Request D → 외부 API 응답 대기
```

Timeout이 지나치게 길면 요청을 처리하는 Thread와 Connection이 오랫동안 반환되지 않을 수 있다.

```text
외부 API 장애
      ↓
요청들이 장시간 대기
      ↓
Thread 점유
Connection 점유
      ↓
새로운 요청도 대기
      ↓
우리 Server까지 영향
```

하나의 외부 시스템 장애가 이를 호출하는 Server까지 영향을 주는 **장애 전파**가 발생할 수 있다.

### Timeout이 너무 짧다면?

반대로 Timeout을 무조건 짧게 설정하는 것도 좋은 것은 아니다.

```text
정상적인 요청
→ 처리에 1초 필요

Timeout
→ 500ms

결과
→ 정상적으로 처리될 요청도 실패
```

따라서 Timeout은 외부 시스템의 정상적인 응답 시간, 서비스 요구사항 등을 고려해서 설정해야 한다.

### 재시도와 Timeout

Timeout이 발생했다고 무조건 재시도하는 것도 주의해야 한다.

예를 들어 결제 요청을 보냈는데 Response를 받기 전에 Timeout이 발생했다고 생각해보자.

```text
우리 Server
   │
   │ 결제 요청
   ↓
결제 Server
   │
   │ 결제 처리 완료
   ↓
Response
   X
Timeout
```

우리 Server 입장에서는 요청이 실패한 것처럼 보이지만 실제 결제 Server에서는 이미 처리가 완료되었을 수 있다.

이 상황에서 단순히 같은 요청을 다시 보내면 중복 처리가 발생할 가능성이 있다.

따라서 재시도를 설계할 때는 HTTP Method의 멱등성뿐 아니라 **실제 비즈니스 작업이 중복 수행되지 않도록 Idempotency Key 등의 방법을 함께 고려**할 수 있다.

```text
Timeout
   ↓
무조건 재시도 X
   ↓
요청의 멱등성 확인
   ↓
중복 처리 방지 고려
   ↓
필요한 경우 재시도
```

### 전체 흐름

```text
Client
  ↓
HTTP Request
  │
  ├─ Method
  ├─ Header
  └─ Body
  ↓
HTTPS라면 TLS로 보호
  ↓
Server
  ↓
REST API
  │
  ├─ Resource
  ├─ Method
  └─ Status Code
  ↓
필요한 경우 외부 API 호출
  ↓
Connection Pool에서 Connection 획득
  ↓
HTTP Connection 재사용
  ↓
외부 Server 응답 대기
  ↓
Timeout으로 대기 시간 제한
  ↓
HTTP Response
```

결국 HTTP 기반 백엔드에서는 단순히 Request와 Response의 형식만 아는 것이 아니라 **Connection을 어떻게 재사용하고, 외부 시스템이 느리거나 장애가 발생했을 때 자원을 어떻게 보호할 것인지까지 함께 고려해야 한다.**
