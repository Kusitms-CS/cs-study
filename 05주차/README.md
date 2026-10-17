# 5주차 - HTTP / HTTPS

## 공통 키워드

### HTTP
- HyperText Transfer Protocol. 클라이언트-서버 간 요청/응답으로 데이터를 주고받는 응용 계층 프로토콜
- 기본 포트 80, 하위에서 TCP 사용 (HTTP/3 제외)
- 구조: Request(Start line, Header, Body) → Response(Status line, Header, Body)

### Stateless
- 서버가 클라이언트의 이전 요청 상태를 저장하지 않음. 요청마다 필요한 정보를 전부 담아서 보내야 함
- 장점: 서버 확장(scale-out)이 쉬움. 어느 서버가 받아도 동일하게 처리 가능
- 단점: 매번 같은 정보를 보내야 해서 데이터 증가
- 상태 유지가 필요하면 Cookie, Session, JWT 등으로 보완

### Method
| Method | 용도 | 멱등 | 안전 |
|---|---|---|---|
| GET | 리소스 조회 | O | O |
| POST | 리소스 생성, 데이터 처리 | X | X |
| PUT | 리소스 전체 교체(없으면 생성) | O | X |
| PATCH | 리소스 일부 수정 | X(구현에 따라) | X |
| DELETE | 리소스 삭제 | O | X |

- 멱등성: 여러 번 호출해도 결과가 같음
- 안전성: 호출해도 리소스가 변경되지 않음

### Status Code
- 1xx: 정보 (처리 중)
- 2xx: 성공 (200 OK, 201 Created, 204 No Content)
- 3xx: 리다이렉션 (301 영구 이동, 302 임시 이동, 304 Not Modified)
- 4xx: 클라이언트 오류 (400 Bad Request, 401 Unauthorized, 403 Forbidden, 404 Not Found, 429 Too Many Requests)
- 5xx: 서버 오류 (500 Internal Server Error, 502 Bad Gateway, 503 Service Unavailable, 504 Gateway Timeout)

### HTTP/1.1 · 2 · 3
| 버전 | 특징 | 한계 |
|---|---|---|
| 1.1 | Keep-Alive 기본, 파이프라이닝(실사용 X) | 요청이 순서대로 처리되는 HOL Blocking, 헤더 중복 |
| 2 | 바이너리 프레이밍, 멀티플렉싱(한 연결에서 동시 요청), 헤더 압축(HPACK), 서버 푸시 | TCP 레벨 HOL Blocking은 여전함 |
| 3 | TCP 대신 QUIC(UDP 기반) 사용, TLS 1.3 내장, 연결 수립 빠름(0-RTT/1-RTT), 스트림 단위 독립 | UDP 차단 환경 등 호환성 이슈 |

### HTTPS
- HTTP + TLS(암호화 계층) 결합. 기본 포트 443
- 제공하는 것: 기밀성(암호화), 무결성(변조 방지), 인증(서버 신원 확인)
- 서버가 CA(인증 기관)에서 발급받은 인증서를 사용

### TLS
- Transport Layer Security. SSL의 후속 표준 (SSL은 취약점으로 폐기됨)
- 핸드셰이크 흐름 (TLS 1.2 기준)
  1. ClientHello: 지원하는 TLS 버전, 암호 스위트, 랜덤값 전달
  2. ServerHello + 인증서 전달
  3. 클라이언트가 인증서 검증 (CA 서명, 도메인, 유효기간)
  4. 키 교환으로 세션 키(대칭키) 생성
  5. 이후 통신은 대칭키로 암호화
- 비대칭키는 키 교환/인증에, 대칭키는 실제 데이터 암호화에 사용 (속도 때문)
- TLS 1.3은 핸드셰이크가 1-RTT로 줄어듦

### TCP 3-way Handshake (TLS 이전 단계)
1. SYN → 2. SYN+ACK → 3. ACK
- TCP 연결을 먼저 맺고, 그 위에서 TLS 핸드셰이크 진행

## Backend 심화 (선택)

### Keep-Alive
- 한 번 맺은 TCP 연결을 재사용해서 여러 요청을 처리
- 연결 수립/종료 비용(3-way, TLS 핸드셰이크) 절감
- `Connection: keep-alive`, `Keep-Alive: timeout=5, max=100`
- HTTP/1.1부터 기본값

### REST API
- 자원(URI) + 행위(Method) + 표현(Representation)으로 API를 설계하는 아키텍처 스타일
- URI는 명사 복수형, 행위는 HTTP Method로 표현
  - `GET /users/1`, `POST /users`, `PUT /users/1`, `PATCH /users/1`, `DELETE /users/1`
- 특징: Stateless, 클라이언트-서버 분리, 캐시 가능, 계층화, 일관된 인터페이스

### Connection / Timeout
- Connection Timeout: 연결 수립까지 기다리는 최대 시간
- Read(Socket) Timeout: 연결 후 응답 데이터를 기다리는 최대 시간
- Keep-Alive Timeout: 유휴 연결을 유지하는 시간
- Connection Pool: 연결을 미리 만들어두고 재사용 (HikariCP, HttpClient 등)
- 타임아웃을 안 걸면 느린 외부 API 하나 때문에 스레드가 고갈될 수 있음