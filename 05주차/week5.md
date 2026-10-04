# HTTP / HTTPS: 연결 위에서 요청을 주고받는 규칙

지난주에는 DNS로 주소를 찾고, IP와 Port로 목적지를 구분하고, TCP로 연결을 만드는 과정을 살펴봤다. 그런데 연결이 만들어졌다고 해서 서버가 클라이언트의 의도를 알아듣는 것은 아니다. **무엇을 요청하는지, 처리 결과를 어떻게 알리는지, 그 내용을 어떻게 보호하는지**에 대한 규칙이 필요하다.

이번에는 HTTP 요청과 응답을 출발점으로 Stateless, Method, Status Code를 정리했다. 이어서 HTTP 버전별 전송 방식과 HTTPS/TLS를 살펴보고, 백엔드에서 연결 재사용과 API 설계, 타임아웃을 어떻게 다뤄야 하는지 연결해 봤다.

---

## 1. HTTP: 연결이 만들어진 다음에는 무엇을 주고받을까?

HTTP(Hypertext Transfer Protocol)는 클라이언트와 서버가 **요청과 응답을 주고받는 응용 계층 프로토콜**이다. 이름에는 Hypertext가 들어가지만 HTML뿐 아니라 JSON, 이미지 등도 전달한다.

```text
클라이언트 → GET /users/42 → 서버
클라이언트 ← 200 OK + 사용자 정보 ← 서버
```

HTTP/1.1 메시지를 단순화하면 다음과 같다. 줄바꿈 등 실제 바이트 표현은 생략한 개념 예시다.

```http
GET /users/42 HTTP/1.1
Host: api.example.com
Accept: application/json

```

```http
HTTP/1.1 200 OK
Content-Type: application/json
Content-Length: 9

{"id":42}
```

| 구성 | 요청 | 응답 |
| --- | --- | --- |
| 시작 줄 | Method, 요청 대상, 버전 | 버전, Status Code 등 |
| 헤더 | 원하는 표현 형식, 인증 정보 등 | 본문 형식, 캐시 정책 등 |
| 본문 | 전달할 데이터가 있는 경우 사용 | 처리 결과나 리소스 표현 등 |

헤더와 본문 사이에는 빈 줄이 있다. 본문이 모든 메시지에 존재하는 것은 아니다. HTTP/2와 HTTP/3은 위 텍스트를 그대로 전송하지 않고 프레임으로 표현하지만, Method와 Status Code 같은 의미는 유지한다.

HTTP는 애플리케이션의 의도를 표현하고, 전송 계층은 그 메시지를 전달한다. 이 구분이 있어야 TCP 연결 성공과 HTTP 요청 성공을 따로 판단할 수 있다. [HTTP 메시지와 공통 의미](https://www.rfc-editor.org/rfc/rfc9110.html)를 기준으로 정리했다.

---

## 2. Stateless: 로그인했는데 왜 매번 인증 정보가 필요할까?

HTTP가 Stateless하다는 것은 **프로토콜 자체가 이전 요청의 대화 상태를 자동으로 기억해 다음 요청에 연결해 주지 않는다**는 뜻이다.

```text
첫 요청: 로그인 정보 전달 → 서버에서 사용자 확인
다음 요청: GET /orders → 이 요청은 누구의 주문 조회일까?
```

같은 TCP 연결을 사용한다고 해서 다음 요청이 앞서 로그인한 사용자의 요청이라는 의미가 자동으로 생기지는 않는다. 애플리케이션은 쿠키나 인증 헤더 등으로 각 요청을 사용자와 연결한다.

예를 들어 세션 방식에서는 다음 흐름을 사용할 수 있다.

```text
로그인 성공
    ↓
서버에 세션 저장, 클라이언트에 세션 식별자 쿠키 전달
    ↓
이후 요청에 쿠키 전송
    ↓
서버가 식별자로 세션 조회
```

따라서 **Stateless = 서버에 어떤 데이터도 저장하면 안 됨**은 아니다. DB의 주문 정보나 서버의 로그인 세션은 애플리케이션이 관리하는 상태다. HTTP의 특성과 서비스의 상태 관리 방식을 구분해야 한다. [MDN의 Stateless와 세션 설명](https://developer.mozilla.org/en-US/docs/Web/HTTP/Guides/Overview#http_is_stateless_but_not_sessionless)도 이 차이를 설명한다.

서버가 여러 대일 때 로그인 세션을 한 서버의 메모리에만 저장하면 다른 서버는 그 세션을 모를 수 있다. 세션 공유 저장소나 요청 라우팅 정책을 고려해야 하는 이유다. 토큰 방식도 폐기 목록이나 사용자 상태 조회 등이 필요할 수 있으므로, 토큰을 쓴다는 이유만으로 모든 상태 관리가 사라지지는 않는다.

---

## 3. Method: 서버에 어떤 동작을 요청할까?

Method는 요청 대상에 대해 어떤 동작을 원하는지 표현한다.

| Method | 의미 | 안전한가? | 멱등한가? |
| --- | --- | --- | --- |
| GET | 현재 표현 조회 | O | O |
| HEAD | GET과 같은 의미지만 응답 본문 없이 메타데이터 조회 | O | O |
| POST | 대상 리소스가 정의한 방식으로 데이터 처리 | X | 보장하지 않음 |
| PUT | 대상 리소스의 상태를 요청 표현으로 생성하거나 대체 | X | O |
| PATCH | 부분 수정 적용 | X | 보장하지 않음 |
| DELETE | 대상 리소스와 현재 기능의 연결 제거 | X | O |
| OPTIONS | 대상의 통신 옵션 조회 | O | O |

안전성은 클라이언트가 서버의 상태 변경을 요청하지 않는다는 뜻이다. GET을 처리하면서 접근 로그를 남기는 것까지 금지하는 의미는 아니다. 멱등성은 **같은 요청을 여러 번 보냈을 때 서버에 의도한 효과가 한 번 보낸 경우와 같다**는 뜻이다. [Method의 의미](https://developer.mozilla.org/en-US/docs/Web/HTTP/Reference/Methods)를 구분해 외우는 편이 재시도 판단에 도움이 된다.

```text
PUT /users/42 + 이름을 "Kim"으로 설정
→ 반복해도 의도한 최종 상태는 이름이 "Kim"

POST /orders + 주문 생성
→ 별도 중복 방지 장치가 없다면 반복 호출로 주문이 여러 개 생길 수 있음
```

멱등하다고 응답까지 같아야 하는 것은 아니다. DELETE의 첫 응답은 성공이고 다음 응답은 404일 수 있지만, 대상의 연결이 제거된 상태라는 의도한 효과는 같다. PATCH도 값을 설정하는 방식인지 누적 증가하는 방식인지에 따라 멱등성이 달라진다.

GET으로 상태를 바꾸면 캐시나 자동 재시도 등이 예상하지 못한 변경을 일으킬 수 있다. 안전성과 멱등성은 이름만으로 구현에 부여되는 성질이 아니라, 서버가 지켜야 할 의미다.

---

## 4. Status Code: 결과를 어떻게 알릴까?

Status Code는 요청 처리 결과를 세 자리 숫자로 표현한다.

| 범위 | 의미 | 대표 예시 |
| --- | --- | --- |
| 1xx | 중간 정보 | 100 Continue |
| 2xx | 성공 | 200 OK, 201 Created, 204 No Content |
| 3xx | 리다이렉션 등 추가 처리 | 301, 302, 304, 307, 308 |
| 4xx | 클라이언트 오류 범주 | 400, 401, 403, 404, 409, 429 |
| 5xx | 서버 오류 범주 | 500, 502, 503, 504 |

API를 구현하며 특히 혼동하기 쉬운 코드는 다음과 같다.

- `201`: 새 리소스가 생성됨. `Location`으로 생성된 리소스를 식별할 수 있다.
- `204`: 요청은 성공했지만 응답 본문이 없음.
- `401`: 유효한 인증 정보가 부족함. 이름은 Unauthorized지만 인증과 관련된다.
- `403`: 서버가 요청을 이해했지만 수행을 거부함. 권한 부족 등이 이유가 될 수 있다.
- `409`: 요청이 리소스의 현재 상태와 충돌함.
- `502`: 게이트웨이·프록시가 상위 서버에서 유효하지 않은 응답을 받음.
- `504`: 게이트웨이·프록시가 필요한 상위 서버 응답을 제때 받지 못함.

`304 Not Modified`는 다른 주소로 이동하라는 뜻이 아니라, 조건부 조회에서 저장된 표현을 재사용할 수 있다는 응답이다. 리다이렉션에서는 `307/308`이 Method를 유지하고, `301/302`는 POST가 GET으로 바뀌는 동작을 허용한다. [상태 코드별 의미](https://developer.mozilla.org/en-US/docs/Web/HTTP/Reference/Status)를 확인해야 숫자 범위만으로 잘못 해석하지 않는다.

5xx를 받았다고 원래 요청이 아무 효과도 없었다고 단정할 수도 없다. 주문 생성은 완료됐지만 중간 프록시에서 응답 전달이 실패했을 수 있으므로 재시도는 별도 판단이 필요하다.

---

## 5. HTTP/1.1·2·3: 의미는 같고 전송 방식이 달라진다

### HTTP/1.1에서는 무엇이 기다림을 만들까?

HTTP/1.1은 텍스트 기반 메시지를 사용하며, 기본적으로 연결을 재사용한다. Pipelining으로 응답을 기다리지 않고 여러 요청을 보내더라도 응답은 요청 순서대로 보내야 하므로, 앞선 응답이 늦으면 뒤 응답도 기다리는 문제가 있다. 브라우저는 일반적으로 Pipelining 대신 여러 연결을 활용해 왔다. [HTTP/1.1 연결 관리](https://www.rfc-editor.org/rfc/rfc9112.html#section-9)를 기준으로 이해했다.

### HTTP/2는 여러 요청을 어떻게 나눌까?

HTTP/2는 메시지를 바이너리 프레임으로 나누고, **하나의 연결에서 여러 스트림을 다중화**한다. 각 스트림의 프레임을 섞어 전달할 수 있어 HTTP/1.1의 응답 순서에 따른 기다림을 줄인다. 헤더에는 HPACK 압축을 사용한다.

```text
하나의 TCP 연결
  ├─ Stream 1: HTML 응답
  ├─ Stream 3: CSS 응답
  └─ Stream 5: 이미지 응답
       각 스트림의 프레임을 섞어서 전송
```

하지만 TCP는 연결 전체에 순서 있는 바이트 스트림을 제공한다. 패킷 손실로 앞선 바이트가 비면 뒤에 도착한 바이트도 기다릴 수 있어 여러 HTTP/2 스트림이 함께 지연된다. **HTTP 계층의 기다림을 줄여도 TCP 계층의 Head-of-Line Blocking은 남는다.** [HTTP/2 명세](https://www.rfc-editor.org/rfc/rfc9113.html)를 통해 두 계층을 구분했다.

### HTTP/3는 왜 UDP 위에서 동작할까?

HTTP/3은 TCP 대신 UDP 기반 QUIC을 사용한다. QUIC은 신뢰성, 혼잡 제어, 스트림 관리와 TLS 1.3 기반 보안을 제공한다. UDP만으로 HTTP의 전달 요구를 충족하는 것이 아니라, **그 위의 QUIC이 필요한 기능을 구현**하는 것이다.

QUIC은 스트림별로 순서를 관리하므로 한 스트림의 손실 때문에 다른 스트림이 같은 순서 복원을 기다릴 필요가 없다. 다만 혼잡 제어 등 공유 자원의 영향은 남으며, 모든 지연이 없어지는 것은 아니다. HTTP/3의 헤더 압축은 QPACK을 사용한다. [HTTP/3 명세](https://www.rfc-editor.org/rfc/rfc9114.html)를 참고했다.

| 구분 | HTTP/1.1 | HTTP/2 | HTTP/3 |
| --- | --- | --- | --- |
| 전송 기반 | TCP | TCP | QUIC / UDP |
| 메시지 표현 | 텍스트 기반 | 바이너리 프레임 | 바이너리 프레임 |
| 다중화 | 같은 연결의 응답 순서에 제약 | 여러 스트림 다중화 | 여러 QUIC 스트림 활용 |
| 헤더 압축 | 전용 헤더 압축 없음 | HPACK | QPACK |
| 손실 시 순서 대기 | TCP 연결 단위 | TCP 연결 단위 | QUIC 스트림 단위 |

버전이 높다고 모든 환경에서 항상 더 빠르지는 않다. 실제 성능은 지연, 손실, 서버 지원, 네트워크 환경에 영향을 받는다.

---

## 6. HTTPS와 TLS: 요청 내용을 어떻게 보호할까?

HTTPS는 HTTP 통신을 TLS로 보호하는 방식이다. TLS는 전송 내용의 **기밀성, 무결성, 상대 인증**을 제공한다. 일반적인 웹 접속에서는 서버를 인증하고, 설정에 따라 클라이언트 인증도 사용할 수 있다. [TLS의 역할](https://developer.mozilla.org/en-US/docs/Web/Security/Defenses/Transport_Layer_Security)을 HTTP의 역할과 나눠 봤다.

### 인증서가 있으면 무엇을 확인할 수 있을까?

클라이언트는 인증서의 신뢰 체인, 접속한 이름과의 일치, 유효 기간 등을 검증한다. TLS 핸드셰이크에서는 상대가 인증서에 대응하는 개인 키를 가지고 있는지도 확인한다.

그 결과는 해당 이름의 서버와 보호된 채널을 만들었다는 의미다. 서비스의 내용이 믿을 만한지나 서버 내부에 취약점이 없는지까지 보증하는 것은 아니다. CDN이나 로드밸런서에서 TLS를 종료한다면 그 지점 이후 원본 서버까지의 통신 보호도 별도로 확인해야 한다.

### TLS 1.3에서는 키를 어떻게 정할까?

인증서를 사용하는 일반적인 최초 연결을 단순화하면 다음과 같다. 세션 재개와 클라이언트 인증 등은 생략했다.

```text
클라이언트                              서버
    | -- ClientHello: 지원 설정·키 공유 --> |
    | <-- ServerHello: 선택 설정·키 공유 -- |
    | <-- 암호화된 인증서·서명·Finished --- |
    |        인증서와 서명 등 검증          |
    | -- Finished ----------------------> |
    | <==== 보호된 HTTP 데이터 교환 ====> |
```

양쪽은 키 교환 결과로 통신에 사용할 대칭 키를 도출한다. 서버가 인증서의 공개 키로 모든 HTTP 데이터를 암호화하거나, 대칭 키를 평문으로 보내는 방식이 아니다. 이후 데이터는 대칭 키를 이용한 인증된 암호화로 보호한다. [TLS 1.3 명세](https://www.rfc-editor.org/rfc/rfc8446.html#section-2)를 기준으로 정리했다.

TLS 1.3의 일반적인 최초 핸드셰이크는 TLS 단계에서 1-RTT이며, TCP 연결 수립 시간과는 구분한다. 세션 재개 시 가능한 0-RTT 데이터는 재전송 공격 위험이 있어 상태를 변경하는 요청에 무조건 사용해서는 안 된다. HTTP/3은 QUIC이 TLS 1.3 핸드셰이크를 통합하므로 별도의 TCP 연결을 만들지 않는다.

---

## Backend 심화

## 7. Keep-Alive: 요청마다 연결을 만들 필요가 있을까?

새 TCP 연결과 TLS 연결을 매번 만들면 지연과 처리 비용이 발생한다. HTTP의 지속 연결은 **이미 열린 연결로 여러 요청과 응답을 처리**하게 한다.

```text
TCP + TLS 연결 수립
    ↓
요청 A / 응답 A
    ↓ 같은 연결 사용
요청 B / 응답 B
    ↓
정책에 따라 유휴 상태 유지 또는 종료
```

HTTP/1.1은 기본적으로 지속 연결을 사용하고, `Connection: close`로 종료 의도를 표현할 수 있다. 매번 `Connection: keep-alive`를 넣어야 재사용되는 것은 아니다. HTTP/2와 HTTP/3은 `Connection` 같은 연결 전용 헤더를 사용하지 않는다. [HTTP/1.1 지속 연결 규칙](https://www.rfc-editor.org/rfc/rfc9112.html#section-9.3)이 기준이다.

연결을 무한히 유지하면 소켓과 버퍼 등의 자원을 차지한다. 서버는 유휴 시간, 최대 연결 수 등을 관리하고, 클라이언트의 연결 풀도 재사용할 연결 수와 대기 시간을 제한해야 한다. 상대가 먼저 종료한 유휴 연결을 재사용하면 실패할 수 있다는 점도 고려한다.

지난주 살펴본 TCP Keepalive는 유휴 연결의 생존 여부를 확인하는 기능이다. HTTP 연결 재사용이나 요청 응답 기한과는 목적이 다르다. 연결이 살아 있어도 요청 처리가 제때 끝난다는 보장은 없다.

---

## 8. REST API: URL과 Method만 맞추면 REST일까?

REST(Representational State Transfer)는 HTTP API 작성 규칙 몇 개가 아니라 **분산 시스템의 아키텍처 스타일**이다. Client-Server, Stateless, Cache, Uniform Interface, Layered System, 선택적인 Code-on-Demand 제약으로 구성된다. [Fielding의 REST 정의](https://ics.uci.edu/~fielding/pubs/dissertation/rest_arch_style.htm)를 참고했다.

Uniform Interface에는 리소스 식별, 표현을 통한 조작, 자기 서술적인 메시지, 하이퍼미디어를 통한 애플리케이션 상태 전이가 포함된다. 따라서 JSON을 사용하고 URL에 명사를 넣는 것만으로 모든 REST 제약을 충족하는 것은 아니다.

실무 API 설계에서는 우선 리소스와 HTTP 의미를 일관되게 표현하는 것이 유용하다.

```text
GET    /orders       → 주문 목록 조회
POST   /orders       → 새 주문 생성
GET    /orders/42    → 특정 주문 조회
PATCH  /orders/42    → 주문 일부 수정
DELETE /orders/42    → 주문 리소스 제거
```

위 예시는 리소스와 Method를 연결한 설계 예시다. 주문 취소가 이력 보존과 상태 전이를 요구한다면 단순 삭제보다 별도 취소 리소스나 상태 변경으로 모델링할 수 있다. URL 형식보다 실제 도메인의 의미가 먼저다.

REST의 Stateless 제약에서는 각 요청에 처리에 필요한 정보를 담고, 서버가 클라이언트의 대화 상태를 기억해야만 다음 요청을 해석할 수 있는 구조를 피한다. 앞에서 설명한 **HTTP 자체의 Stateless와 REST 아키텍처의 제약은 관련되지만 동일한 설명은 아니다.** 서버 세션을 사용하는 HTTP 서비스가 자동으로 REST의 Stateless 제약을 충족하는 것은 아니다.

---

## 9. Connection / Timeout: 응답이 늦으면 어디를 확인할까?

외부 API 호출이 느리다는 결과만으로는 원인을 알 수 없다. 먼저 시간이 소비되는 단계를 나눠야 한다.

```text
DNS 조회
    ↓
연결 풀에서 사용 가능한 연결 대기
    ↓ 새 연결이 필요한 경우
TCP 연결 + TLS 핸드셰이크
    ↓
요청 전송 → 서버 처리 → 응답 수신
```

| 제한 | 확인할 기다림 |
| --- | --- |
| 연결 풀 대기 제한 | 사용할 연결을 얻는 데 걸리는 시간 |
| Connect timeout | 새 연결 수립에 걸리는 시간 |
| TLS handshake timeout | TLS 협상에 걸리는 시간 |
| Read/response timeout | 응답을 받거나 다음 데이터를 기다리는 시간 |
| Idle timeout | 연결에 활동이 없는 시간 |
| 전체 요청 기한 | 여러 단계를 포함한 작업의 총 시간 |

이 표는 설정을 확인할 때 사용할 구분이다. 실제 이름과 범위는 라이브러리마다 다르다. 특히 Read timeout이 읽기 사이의 유휴 시간 제한이면 데이터가 조금씩 오는 동안 전체 작업은 오래 계속될 수 있다. DNS, TLS, 연결 풀 대기까지 어느 제한에 포함되는지 문서를 확인해야 한다.

타임아웃은 클라이언트가 기다리기를 끝냈다는 뜻이지, 서버 작업이 취소되거나 롤백됐다는 뜻이 아니다. 주문 생성처럼 중복 실행이 문제가 되는 요청에는 요청 식별자 등을 이용한 중복 처리 방지와 결과 조회 방법을 설계해야 한다.

재시도는 멱등성, 오류 원인, 남은 전체 기한을 보고 제한적으로 수행한다. 여러 계층이 각각 재시도하면 요청 수가 급증할 수 있어 횟수와 간격도 함께 관리한다.

---

## 전체 흐름 정리

새 TCP 연결을 사용하는 HTTPS 요청을 기준으로 지난주와 이번 주 내용을 연결하면 다음과 같다.

```text
DNS로 IP 조회
    ↓
TCP 연결 수립
    ↓
TLS로 서버 인증 및 보호된 채널 수립
    ↓
Method + 요청 대상 + 헤더 + 필요한 본문 전송
    ↓
서버의 인증·권한 확인과 비즈니스 처리
    ↓
Status Code + 헤더 + 필요한 본문 수신
    ↓
연결 재사용 또는 종료
```

HTTP/3은 TCP 대신 QUIC을 사용하고, 기존 연결이나 캐시를 활용하면 일부 단계가 생략될 수 있다.

이번 주제를 정리하면서 연결 상태, HTTP의 상태 관리, 업무 처리 결과를 따로 봐야 한다는 점이 가장 기억에 남았다. TCP가 연결됐다고 로그인한 것은 아니고, HTTPS를 쓴다고 권한 검사가 해결되는 것도 아니다. 응답을 받지 못했다고 주문 생성이 실패한 것도 아니다. 각 계층이 보장하는 범위를 구분해야 API의 응답 코드와 재시도 정책도 제대로 정할 수 있다.

## 참고 자료

- [RFC 9110 — HTTP Semantics](https://www.rfc-editor.org/rfc/rfc9110.html): HTTP 버전 공통 의미
- [MDN — Overview of HTTP](https://developer.mozilla.org/en-US/docs/Web/HTTP/Guides/Overview): 요청·응답과 Stateless
- [MDN — HTTP request methods](https://developer.mozilla.org/en-US/docs/Web/HTTP/Reference/Methods): Method, 안전성, 멱등성
- [MDN — HTTP response status codes](https://developer.mozilla.org/en-US/docs/Web/HTTP/Reference/Status): 상태 코드
- [RFC 9112 — HTTP/1.1](https://www.rfc-editor.org/rfc/rfc9112.html): 메시지 형식과 지속 연결
- [RFC 9113 — HTTP/2](https://www.rfc-editor.org/rfc/rfc9113.html): 프레임과 스트림 다중화
- [RFC 9114 — HTTP/3](https://www.rfc-editor.org/rfc/rfc9114.html): QUIC 기반 HTTP
- [MDN — Transport Layer Security](https://developer.mozilla.org/en-US/docs/Web/Security/Defenses/Transport_Layer_Security): TLS의 역할
- [RFC 8446 — TLS 1.3](https://www.rfc-editor.org/rfc/rfc8446.html): 키 교환과 핸드셰이크
- [Fielding — REST](https://ics.uci.edu/~fielding/pubs/dissertation/rest_arch_style.htm): REST 아키텍처 제약
