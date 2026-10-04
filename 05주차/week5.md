# 5주차 - HTTP / HTTPS 개념 정리

## 목차
- [키워드 연관관계](#키워드-연관관계)
- [HTTP](#http)
- [Stateless](#stateless)
- [Method](#method)
- [Status Code](#status-code)
- [HTTP/1.1·2·3](#http112middot3)
- [HTTPS](#https)
- [TLS](#tls)
- [Backend 심화 (선택)](#backend-심화-선택)
  - [Keep-Alive](#keep-alive)
  - [REST API](#rest-api)
  - [Connection/Timeout](#connectiontimeout)

---

## 키워드 연관관계

각 개념이 독립된 게 아니라, 아래처럼 HTTP라는 하나의 프로토콜에서 뻗어나간 특성/구성요소다.

```
HTTP
├─ Stateless
├─ Method → Status Code
├─ HTTP/1.1·2·3 → Keep-Alive → Connection/Timeout
├─ HTTPS → TLS
└─ Method → REST API
```

| 연결 | 관계 설명 |
|---|---|
| HTTP → Stateless | HTTP 프로토콜 자체가 이전 요청 상태를 기억하지 않는 특성을 가짐 |
| HTTP → Method → Status Code | 클라이언트가 Method로 요청하면 서버가 Status Code로 처리 결과를 응답 |
| HTTP → HTTP/1.1·2·3 | HTTP는 버전을 거듭하며 발전해왔음 |
| HTTP/1.1·2·3 → Keep-Alive | HTTP/1.1부터 하나의 연결을 재사용(Keep-Alive)하는 기능이 기본화 |
| Keep-Alive → Connection/Timeout | 연결을 계속 유지하는 대신, 언제까지 유지할지 제한 시간을 정해야 함 |
| HTTP → HTTPS → TLS | HTTPS는 HTTP에 TLS(암호화 계층)를 추가한 것 |
| HTTP → Method → REST API | REST API는 HTTP Method와 URL을 이용해 자원을 다루는 설계 스타일 |

---

## HTTP


클라이언트(브라우저 등)와 서버가 데이터를 주고받기 위한 **약속(프로토콜)**. 요청(Request)을 보내면 응답(Response)이 오는 구조.

- 활용 예: 브라우저에 주소를 입력하면 HTTP `GET` 요청이 서버로 전송되고, 서버는 HTML을 HTTP 응답으로 돌려준다

---

## Stateless

서버가 **이전 요청의 상태(정보)를 기억하지 않는** 특성. 각 요청은 서로 독립적으로 처리된다.

- 장점: 서버가 상태를 관리할 필요가 없어 서버를 여러 대로 늘리기(Scale-out) 쉬움
- 단점: 로그인 유지처럼 "상태가 필요한 기능"은 Cookie, Session, JWT 같은 별도 수단으로 보완해야 함


---

## Method

| Method | 용도 | 예시 |
|---|---|---|
| GET | 조회 | 게시글 목록 조회 |
| POST | 생성 | 새 게시글 작성 |
| PUT | 전체 수정 | 게시글 전체 내용 교체 |
| PATCH | 일부 수정 | 게시글 제목만 수정 |
| DELETE | 삭제 | 게시글 삭제 |

---

## Status Code

| 코드대 | 의미 | 예시 |
|---|---|---|
| 1xx | 정보성(처리 중) | 100 Continue |
| 2xx | 성공 | 200 OK, 201 Created |
| 3xx | 리다이렉션 | 301 Moved Permanently, 304 Not Modified |
| 4xx | 클라이언트 오류 | 400 Bad Request, 401 Unauthorized, 404 Not Found |
| 5xx | 서버 오류 | 500 Internal Server Error, 503 Service Unavailable |

---

## HTTP/1.1·2·3



- **HTTP/1.1**: 연결 하나당 기본적으로 순차 처리 (Keep-Alive로 연결은 재사용하지만, 앞 요청이 끝나야 다음 요청 처리 → HOL Blocking 문제)
- **HTTP/2**: 하나의 연결에서 여러 요청/응답을 동시에 처리(멀티플렉싱), 헤더 압축(HPACK), 서버 푸시 지원
- **HTTP/3**: 전송 프로토콜을 TCP 대신 UDP 기반의 **QUIC**으로 교체 → 연결 설정이 빠르고, 패킷 손실이 다른 스트림에 영향을 주지 않음
- 활용 예: 최신 브라우저와 주요 웹사이트는 이미 HTTP/2나 HTTP/3을 기본으로 사용 중

---

## HTTPS



HTTP에 **TLS(암호화 계층)** 를 결합한 프로토콜. 데이터 암호화, 서버 신원 확인(인증서), 데이터 무결성을 보장한다.

- 활용 예: 로그인 폼, 결제 정보처럼 민감한 데이터를 주고받을 땐 반드시 HTTPS를 사용해야 한다. 브라우저 주소창의 자물쇠 아이콘이 HTTPS 연결임을 나타낸다

---

## TLS

통신 내용을 암호화하는 프로토콜(이전 이름: SSL). **TLS Handshake** 과정에서 서버 인증서를 확인하고, 암호화에 쓸 키를 교환한다.

- 과정(간단히): `ClientHello → ServerHello(인증서 제공) → 키 교환 → 암호화된 통신 시작`

---

## Backend 심화 (선택)

### Keep-Alive


하나의 TCP 연결을 여러 HTTP 요청/응답에 **재사용**하는 기법. HTTP/1.1부터 기본으로 활성화되어 있다.

- 활용 예: Keep-Alive가 없으면 이미지 10개가 있는 웹페이지를 불러올 때 TCP 연결을 10번 새로 맺어야 하지만, Keep-Alive를 쓰면 연결 하나로 10개를 이어서(또는 HTTP/2라면 동시에) 받아올 수 있다

---

### REST API


HTTP Method와 URL(자원 경로)을 이용해 **자원(Resource)** 을 다루는 API 설계 스타일.

| 동작 | Method | URL 예시 |
|---|---|---|
| 게시글 목록 조회 | GET | `/posts` |
| 게시글 작성 | POST | `/posts` |
| 특정 게시글 조회 | GET | `/posts/1` |
| 게시글 수정 | PUT/PATCH | `/posts/1` |
| 게시글 삭제 | DELETE | `/posts/1` |

- 특징: 요청마다 필요한 정보를 다 담아 보냄(Stateless), 자원은 명사(URL)로, 행위는 Method로 표현

---

### Connection/Timeout



HTTP 요청을 보낼 때, 연결이나 응답을 얼마나 기다릴지 정하는 설정.

- 너무 길게 잡으면 느린 서버에 자원이 오래 묶이고, 너무 짧게 잡으면 정상 요청도 실패 처리될 수 있음
- 활용 예: 자바에서 `RestTemplate`이나 `WebClient`에 `connectTimeout`, `readTimeout`을 설정해, 외부 API 호출이 무한정 대기하지 않도록 방지한다
