# 5주차 - HTTP / HTTPS

## 1. HTTP

### 1.1 HTTP란

- HTTP(HyperText Transfer Protocol)는 클라이언트와 서버 간 데이터를 주고받기 위한 애플리케이션 계층 프로토콜임
- 클라이언트가 Request를 보내면 서버가 Response를 반환하는 요청-응답 구조임
- 웹 브라우저와 웹 서버 간 통신뿐만 아니라 REST API 등 다양한 네트워크 통신에 사용됨
- HTTP 자체는 데이터를 암호화하지 않기 때문에 보안이 필요한 환경에서는 HTTPS를 사용함

### 1.2 HTTP Request

HTTP 요청은 크게 다음과 같이 구성됨

- Request Line: HTTP Method, 요청 URI, HTTP Version 등의 정보가 포함됨
- Header: 요청에 대한 부가적인 정보가 포함됨
- Body: 서버에 전달할 실제 데이터가 포함됨

```http
POST /users HTTP/1.1
Host: example.com
Content-Type: application/json

{
  "name": "user"
}
```

### 1.3 HTTP Response

HTTP 응답은 크게 다음과 같이 구성됨

- Status Line: HTTP Version과 Status Code 등의 정보가 포함됨
- Header: 응답 데이터의 형식, 길이, 캐시 등의 정보가 포함됨
- Body: 클라이언트에게 전달할 실제 데이터가 포함됨

---

## 2. Stateless

### 2.1 Stateless란

- HTTP는 기본적으로 Stateless한 프로토콜임
- 서버가 이전 요청의 상태를 기본적으로 기억하지 않는 방식임
- 각각의 요청은 독립적으로 처리됨

예를 들어 다음과 같은 요청이 존재함

```text
요청 1 → 로그인
요청 2 → 내 정보 조회
```

HTTP 자체만 사용하면 서버는 요청 2가 요청 1에서 로그인한 사용자의 요청인지 알 수 없음

따라서 실제 웹 서비스에서는 상태를 유지하기 위해 다음과 같은 방법을 사용함

- Cookie
- Session
- JWT 등의 Access Token

### 2.2 Stateless의 장점

- 각각의 요청을 독립적으로 처리할 수 있음
- 특정 서버에 클라이언트 상태가 종속되지 않도록 구성하기 쉬움
- 서버 확장과 로드 밸런싱에 유리함

---

## 3. HTTP Method

HTTP Method는 클라이언트가 서버에게 요청하는 작업의 종류를 나타냄

### GET

- 리소스를 조회할 때 사용함
- 일반적으로 Request Body를 사용하지 않음

```http
GET /users/1
```

### POST

- 새로운 리소스를 생성하거나 데이터를 처리하도록 요청할 때 사용함

```http
POST /users
```

### PUT

- 특정 리소스를 전체적으로 수정하거나 교체할 때 주로 사용함

```http
PUT /users/1
```

### PATCH

- 특정 리소스의 일부 데이터를 수정할 때 주로 사용함

```http
PATCH /users/1
```

### DELETE

- 특정 리소스를 삭제할 때 사용함

```http
DELETE /users/1
```

### 주요 Method 특징

| Method | 주요 용도 | 멱등성 |
| --- | --- | --- |
| GET | 조회 | O |
| POST | 생성 및 처리 | X |
| PUT | 전체 수정 및 교체 | O |
| PATCH | 부분 수정 | 구현에 따라 다름 |
| DELETE | 삭제 | O |

- 멱등성이란 동일한 요청을 여러 번 수행해도 서버의 최종 상태가 동일한 성질을 의미함

---

## 4. HTTP Status Code

HTTP Status Code는 서버가 요청을 처리한 결과를 나타내는 3자리 숫자임

### 1xx - Informational

- 요청이 수신되어 처리 중임을 의미함

### 2xx - Success

- `200 OK`: 요청 성공
- `201 Created`: 새로운 리소스 생성 성공
- `204 No Content`: 요청은 성공했지만 응답 Body가 없음

### 3xx - Redirection

- `301 Moved Permanently`: 리소스가 영구적으로 이동함
- `302 Found`: 리소스가 일시적으로 다른 위치에 존재함
- `304 Not Modified`: 캐시된 리소스를 그대로 사용할 수 있음

### 4xx - Client Error

- `400 Bad Request`: 잘못된 요청
- `401 Unauthorized`: 인증이 필요하거나 인증에 실패함
- `403 Forbidden`: 해당 리소스에 접근할 권한이 없음
- `404 Not Found`: 요청한 리소스를 찾을 수 없음
- `409 Conflict`: 현재 리소스 상태와 요청이 충돌함

### 5xx - Server Error

- `500 Internal Server Error`: 서버 내부 오류
- `502 Bad Gateway`: Gateway 또는 Proxy가 잘못된 응답을 받음
- `503 Service Unavailable`: 서버가 일시적으로 요청을 처리할 수 없음
- `504 Gateway Timeout`: Gateway 또는 Proxy가 상위 서버의 응답을 제시간에 받지 못함

---

## 5. HTTP/1.1 · HTTP/2 · HTTP/3

### 5.1 HTTP/1.1

- 하나의 TCP 연결을 여러 HTTP 요청에서 재사용할 수 있음
- Keep-Alive를 통해 매 요청마다 TCP 연결을 새로 생성하는 비용을 줄일 수 있음
- 여러 요청을 동시에 효율적으로 처리하는 데 구조적인 한계가 존재함

### 5.2 HTTP/2

- 하나의 TCP 연결에서 여러 요청과 응답을 동시에 처리하는 Multiplexing을 지원함
- HTTP 메시지를 Binary Frame 단위로 처리함
- HPACK 기반 Header Compression을 지원함
- HTTP/1.1보다 하나의 연결을 효율적으로 사용할 수 있음
- TCP를 사용하기 때문에 패킷 손실 발생 시 TCP 수준의 Head-of-Line Blocking이 발생할 수 있음

### 5.3 HTTP/3

- TCP 대신 UDP 기반의 QUIC 프로토콜을 사용함
- QUIC 내부에서 신뢰성 있는 데이터 전송과 연결 관리를 제공함
- 스트림별로 데이터를 독립적으로 처리하여 HTTP/2의 TCP 수준 Head-of-Line Blocking 문제를 완화함
- 연결 설정 비용을 줄이고 네트워크 환경이 변경되는 모바일 환경에서도 효율적으로 통신할 수 있음

```text
HTTP/1.1 → TCP
HTTP/2   → TCP + Multiplexing
HTTP/3   → QUIC 기반
```

---

## 6. HTTPS

### 6.1 HTTPS란

- HTTPS는 HTTP 통신에 TLS를 적용하여 보안을 강화한 방식임
- HTTP 메시지를 암호화하여 클라이언트와 서버 사이에서 안전하게 데이터를 전달함
- 일반적으로 HTTP는 80번 포트, HTTPS는 443번 포트를 사용함

```text
HTTP + TLS = HTTPS
```

### 6.2 HTTPS가 제공하는 보안

#### 기밀성

- 데이터를 암호화하여 제3자가 통신 내용을 확인하기 어렵게 함

#### 무결성

- 데이터가 전송 과정에서 변조되었는지 확인할 수 있음

#### 인증

- 인증서를 이용하여 클라이언트가 접속한 서버가 신뢰할 수 있는 서버인지 확인함

---

## 7. TLS

### 7.1 TLS란

- TLS(Transport Layer Security)는 네트워크 통신을 암호화하기 위한 프로토콜임
- 과거 SSL이 사용되었으며 현재는 SSL의 후속 기술인 TLS가 사용됨
- HTTPS는 HTTP 데이터를 TLS를 통해 안전하게 전달하는 구조임

### 7.2 TLS 통신 방식

TLS는 대칭키 암호화와 공개키 기반 암호 기술을 함께 활용함

#### 대칭키 암호화

- 암호화와 복호화에 관련된 동일한 비밀키를 사용하는 방식임
- 연산 속도가 빠르기 때문에 실제 데이터 통신에 적합함

#### 공개키 기반 암호 기술

- 공개키와 개인키를 사용하는 비대칭 암호 기술을 활용함
- TLS 연결 과정에서 서버 인증과 안전한 키 합의 등에 사용됨

### 7.3 TLS 연결 과정

```text
Client
   ↓ 연결 요청
Server
   ↓ 인증서 및 TLS 연결에 필요한 정보 전달
Client
   ↓ 인증서 검증
Client ↔ Server
   ↓ 안전하게 세션 키 합의
암호화된 HTTP 통신
```

- 클라이언트와 서버가 지원 가능한 TLS 버전과 암호화 방식을 협상함
- 서버가 인증서를 제공함
- 클라이언트가 인증서를 검증함
- 키 교환 및 키 합의를 통해 세션 키를 생성함
- 이후 세션 키를 이용해 실제 데이터를 암호화하여 통신함

---

# Backend 심화

## 8. Keep-Alive

### 8.1 Keep-Alive란

- 하나의 네트워크 연결을 여러 HTTP 요청과 응답에서 재사용하는 방식임
- 매 요청마다 새로운 TCP 연결을 생성하고 종료하는 비용을 줄일 수 있음

```text
연결 생성
↓
Request 1 → Response 1
Request 2 → Response 2
Request 3 → Response 3
↓
연결 종료
```

### 8.2 Keep-Alive의 장점

- TCP 연결 생성 비용을 줄일 수 있음
- 네트워크 지연 시간을 감소시킬 수 있음
- 서버와 클라이언트의 연결 효율을 높일 수 있음
- 너무 많은 연결을 장시간 유지하면 서버 자원을 점유할 수 있으므로 적절한 Timeout 설정이 필요함

---

## 9. REST API

### 9.1 REST란

- REST(Representational State Transfer)는 웹의 특성을 활용하여 리소스를 중심으로 API를 설계하는 아키텍처 스타일임
- URI는 리소스를 표현하고 HTTP Method를 통해 해당 리소스에 수행할 동작을 표현하는 방식이 일반적임

```text
GET    /users
GET    /users/1
POST   /users
PATCH  /users/1
DELETE /users/1
```

### 9.2 REST API 설계

- URI에는 동사보다 명사를 사용하는 것이 일반적임
- HTTP Method를 통해 리소스에 수행할 행위를 표현함
- HTTP Status Code를 활용하여 요청 처리 결과를 명확하게 표현함
- Stateless한 통신을 지향함
- 클라이언트와 서버의 역할을 분리하여 독립적으로 개발할 수 있도록 구성함

```text
GET /getUsers  X
GET /users     O
```

---

## 10. Connection / Timeout

### 10.1 Connection

- 클라이언트와 서버가 데이터를 주고받기 위해 생성하는 네트워크 연결임
- HTTP/1.1과 HTTP/2는 기본적으로 TCP 연결을 기반으로 통신함
- 서버가 처리할 수 있는 연결 수는 한정되어 있기 때문에 Connection 관리가 중요함
- 연결이 불필요하게 오래 유지되면 서버의 Socket, Thread, Memory 등의 자원을 점유할 수 있음

### 10.2 Timeout

- 특정 작업이 일정 시간 이상 완료되지 않을 경우 해당 작업을 중단하기 위한 설정임
- 서버 자원이 비정상적인 요청에 장시간 점유되는 것을 방지하는 역할을 함

#### Connection Timeout

- 서버와 연결을 생성하기 위해 기다리는 최대 시간임
- 해당 시간 내 연결이 생성되지 않으면 실패 처리함

#### Read Timeout

- 연결 이후 서버로부터 데이터를 읽기 위해 기다리는 최대 시간임
- 서버의 응답이 지나치게 늦을 경우 연결을 계속 기다리지 않도록 제한함

#### Write Timeout

- 데이터를 상대방에게 전송하는 과정에서 허용하는 최대 시간임

#### Keep-Alive Timeout

- 추가 요청이 없는 상태에서 연결을 유지할 최대 시간임
- 설정된 시간이 지나면 유휴 연결을 종료함

### 10.3 Backend에서 Timeout이 중요한 이유

- 외부 API가 응답하지 않을 경우 요청이 무한정 대기하는 것을 방지함
- 느린 연결이 서버 자원을 계속 점유하는 문제를 줄일 수 있음
- 장애가 다른 서버나 서비스로 연쇄적으로 전파되는 것을 완화할 수 있음
- 안정적인 서버 운영을 위해 적절한 Connection Pool과 Timeout 설정이 필요함

---

## 핵심 정리

- HTTP는 클라이언트와 서버가 Request와 Response를 통해 통신하는 Stateless 프로토콜임
- HTTP Method와 Status Code를 통해 요청의 목적과 처리 결과를 표현함
- HTTP/1.1은 TCP 연결 재사용, HTTP/2는 Multiplexing, HTTP/3는 QUIC 기반 통신이 주요 특징임
- HTTPS는 HTTP에 TLS를 적용하여 기밀성, 무결성, 인증을 제공함
- TLS는 서버 인증과 안전한 키 합의를 수행한 후 세션 키를 이용해 실제 데이터를 암호화함
- Backend에서는 Keep-Alive와 Connection을 적절하게 관리하여 네트워크 비용과 서버 자원 사용을 줄이는 것이 중요함
- Connection Timeout, Read Timeout 등의 설정을 통해 느리거나 비정상적인 요청이 서버 자원을 장시간 점유하지 않도록 관리해야 함
