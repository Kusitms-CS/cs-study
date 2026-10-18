# 7주차 - Web Security / 인증

## 1. SOP (Same-Origin Policy)

> SOP는 브라우저에서 서로 다른 출처(Origin) 간의 리소스 접근을 제한하는 보안 정책이다.

웹사이트가 다른 출처의 데이터를 무단으로 읽는 것을 방지하기 위해 브라우저에서 적용한다.

### Origin의 구성

Origin은 **Protocol + Host + Port**로 결정된다.

```text
https://example.com:443
  │       │        │
Protocol  Host    Port
```

| URL | 동일 출처 여부 |
|---|---|
| `https://example.com/page` | O |
| `https://example.com:443/api` | O |
| `http://example.com` | X (Protocol 다름) |
| `https://api.example.com` | X (Host 다름) |
| `https://example.com:8080` | X (Port 다름) |

경로(Path)는 Origin 판단 기준에 포함되지 않는다.

### SOP가 필요한 이유

사용자가 은행 사이트에 로그인한 상태에서 악성 사이트에 접속한 경우이다.

```text
사용자
  │
  ├─ bank.com 로그인
  │
  └─ evil.com 접속
          ↓
     bank.com에 요청
          ↓
     응답 데이터 읽기 시도
          ↓
       SOP 차단
```

SOP가 없다면 악성 사이트의 JavaScript가 다른 출처의 민감한 응답 데이터를 읽을 수 있다.

### SOP의 적용 범위

SOP는 모든 Cross-Origin 요청을 차단하는 정책이 아니다.

- 다른 출처로 HTTP 요청을 보내는 것 자체는 가능한 경우가 많음
- `fetch`, `XMLHttpRequest` 등으로 받은 응답을 JavaScript에서 읽는 행위는 제한
- `<img>`, `<script>` 등의 리소스 로드는 별도 규칙에 따라 허용될 수 있음

즉, SOP의 핵심은 **다른 출처의 응답 데이터에 대한 접근 제한**이다.

---

## 2. CORS (Cross-Origin Resource Sharing)

> CORS는 서버가 허용한 출처에 한해 브라우저의 Cross-Origin 응답 접근을 허용하는 HTTP 기반 메커니즘이다.

SOP가 기본적으로 다른 출처의 응답 접근을 제한한다면, CORS는 이를 안전하게 허용하기 위한 방식이다.

### CORS 동작 과정

프론트엔드와 백엔드의 Origin이 다른 경우이다.

```text
Frontend
http://localhost:3000
        │
        │ API 요청
        ↓
Backend
http://localhost:8080
        │
        │ CORS 응답 헤더
        ↓
Browser
        │
        └─ 허용 여부 확인
```

서버는 다음과 같은 응답 헤더를 설정할 수 있다.

```http
Access-Control-Allow-Origin: http://localhost:3000
```

브라우저는 해당 헤더를 확인하고 응답을 JavaScript에 제공할지 결정한다.

**CORS 정책은 브라우저가 적용한다.** 따라서 Postman이나 서버 간 HTTP 통신에서는 일반적으로 브라우저와 같은 CORS 제한을 받지 않는다.

### Simple Request와 Preflight Request

CORS 요청은 조건에 따라 Preflight 요청이 먼저 발생할 수 있다.

**Simple Request**

정해진 Method와 Header 등의 조건을 만족하면 별도의 Preflight 없이 실제 요청을 전송한다.

**Preflight Request**

브라우저가 실제 요청 전에 `OPTIONS` 요청을 보내 서버의 허용 여부를 확인한다.

```text
Browser                     Server
   │                           │
   │──── OPTIONS ─────────────>│
   │<─── CORS 허용 응답 ────────│
   │                           │
   │──── POST ────────────────>│
   │<─── 실제 응답 ─────────────│
```

예를 들어 `Content-Type: application/json`을 사용하는 Cross-Origin POST 요청은 일반적으로 Preflight 대상이다.

### 주요 CORS 헤더

| 헤더 | 역할 |
|---|---|
| `Access-Control-Allow-Origin` | 허용할 Origin |
| `Access-Control-Allow-Methods` | 허용할 HTTP Method |
| `Access-Control-Allow-Headers` | 허용할 요청 Header |
| `Access-Control-Allow-Credentials` | Cookie 등 Credentials 포함 허용 |
| `Access-Control-Max-Age` | Preflight 결과 캐싱 시간 |

Cookie 등을 포함하는 Credentialed Request에서는 `Access-Control-Allow-Origin: *`을 사용할 수 없으며, 허용할 Origin을 명시해야 한다.

### Spring Boot CORS 설정

```java
@Configuration
public class CorsConfig {

    @Bean
    public CorsConfigurationSource corsConfigurationSource() {
        CorsConfiguration config = new CorsConfiguration();

        config.setAllowedOrigins(
            List.of("https://example.com")
        );
        config.setAllowedMethods(
            List.of("GET", "POST", "PUT", "DELETE")
        );
        config.setAllowedHeaders(List.of("*"));
        config.setAllowCredentials(true);

        UrlBasedCorsConfigurationSource source =
            new UrlBasedCorsConfigurationSource();

        source.registerCorsConfiguration("/**", config);

        return source;
    }
}
```

Spring Security를 사용한다면 Security Filter Chain에서도 CORS 처리를 활성화해야 한다.

```java
http.cors(Customizer.withDefaults());
```

CORS는 **브라우저의 응답 접근을 제어하는 정책**이지, API 자체에 대한 인증이나 접근 권한을 보장하는 보안 장치는 아니다.

---

## 3. XSS (Cross-Site Scripting)

> XSS는 공격자가 삽입한 악성 스크립트가 다른 사용자의 브라우저에서 실행되는 취약점이다.

사용자가 입력한 데이터를 적절히 처리하지 않고 HTML에 출력할 때 발생할 수 있다.

### XSS 공격 과정

게시판에 사용자가 작성한 내용이 그대로 HTML에 반영되는 경우이다.

```html
<script>
  alert("XSS");
</script>
```

```text
공격자
  │
  │ 악성 스크립트 게시
  ↓
Server
  │
  │ 게시글 반환
  ↓
다른 사용자 Browser
  │
  └─ 악성 스크립트 실행
```

악성 스크립트는 페이지 변조, 사용자 대신 요청 전송, 접근 가능한 민감 정보 탈취 등에 악용될 수 있다.

### XSS의 종류

| 종류 | 특징 |
|---|---|
| Stored XSS | 악성 스크립트가 DB 등에 저장된 후 실행 |
| Reflected XSS | 요청에 포함된 악성 스크립트가 응답에 반영되어 실행 |
| DOM-based XSS | 클라이언트 JavaScript가 DOM을 잘못 처리하여 발생 |

**Stored XSS**

게시글이나 댓글에 삽입한 스크립트가 저장되고, 다른 사용자가 해당 페이지를 조회할 때 실행된다.

**Reflected XSS**

URL 파라미터 등의 입력이 HTML 응답에 그대로 반영되면서 스크립트가 실행된다.

**DOM-based XSS**

서버 응답과 관계없이 클라이언트 JavaScript가 입력값을 위험한 DOM API에 전달하면서 발생할 수 있다.

### XSS 방어 방법

**1. 출력 시 인코딩(Output Encoding)**

사용자 입력을 HTML에 출력할 때 특수문자를 적절히 인코딩한다.

```text
입력:
<script>alert("XSS")</script>

HTML 인코딩:
&lt;script&gt;alert("XSS")&lt;/script&gt;
```

브라우저가 이를 HTML 태그가 아닌 일반 텍스트로 처리하도록 한다.

단, HTML 본문, 속성, JavaScript, URL 등 **출력 위치에 맞는 인코딩 방식**을 사용해야 한다.

**2. 위험한 DOM API 사용 주의**

```javascript
// 위험할 수 있는 방식
element.innerHTML = userInput;

// 일반 텍스트로 처리
element.textContent = userInput;
```

사용자 입력을 HTML로 해석할 필요가 없다면 `textContent`를 사용하는 것이 안전하다.

**3. Content Security Policy(CSP)**

브라우저에서 실행할 수 있는 스크립트의 출처 등을 제한한다.

```http
Content-Security-Policy: script-src 'self'
```

CSP는 XSS 피해를 줄이기 위한 추가적인 방어 수단이며, 출력 인코딩을 대체하지는 않는다.

**4. HttpOnly Cookie**

```http
Set-Cookie: sessionId=abc123; HttpOnly; Secure
```

HttpOnly가 설정된 Cookie는 JavaScript의 `document.cookie`로 읽을 수 없다.

따라서 XSS를 통한 Cookie 탈취 위험을 줄일 수 있지만, **XSS 자체를 방지하거나 악성 스크립트의 요청 전송까지 차단하지는 않는다.**

### SOP / CORS / XSS 비교

| 구분 | 목적 또는 공격 방식 |
|---|---|
| SOP | 서로 다른 Origin 간 응답 접근 제한 |
| CORS | 서버가 허용한 Cross-Origin 응답 접근 허용 |
| XSS | 악성 스크립트가 사용자 브라우저에서 실행되는 취약점 |

SOP와 CORS는 **출처 간 접근 제어**에 관한 개념이고, XSS는 **신뢰할 수 없는 스크립트의 실행**에 관한 보안 취약점이다.

## 4. CSRF (Cross-Site Request Forgery)

> CSRF는 사용자가 인증된 상태를 악용하여, 사용자의 의도와 관계없이 특정 웹사이트에 요청을 보내도록 유도하는 공격이다.

브라우저가 요청 대상 사이트의 Cookie를 자동으로 전송할 수 있다는 특성을 이용한다.

### CSRF 공격 과정

사용자가 은행 사이트에 로그인한 상태에서 악성 사이트에 접속한 경우이다.

```text
1. 사용자가 bank.com에 로그인
   ↓
2. 브라우저에 세션 Cookie 저장
   ↓
3. 사용자가 evil.com에 접속
   ↓
4. 악성 페이지가 bank.com에 요청 전송
   ↓
5. 브라우저가 조건에 따라 세션 Cookie 자동 첨부
   ↓
6. 서버가 요청을 정상 사용자 요청으로 오인
```

서버가 Cookie만으로 요청을 신뢰하면 사용자가 의도하지 않은 상태 변경이 발생할 수 있다.

단, 실제 공격 가능 여부는 Cookie의 SameSite 설정, 요청 방식, 서버의 CSRF 방어 여부 등에 따라 달라진다.

### CSRF 방어 방법

**1. CSRF Token**

서버가 예측하기 어려운 Token을 발급하고, 상태 변경 요청에 포함하도록 한다.

```text
Client                      Server
  │                            │
  │──── CSRF Token 요청 ───────>│
  │<─── Token 발급 ─────────────│
  │                            │
  │──── 요청 + Token ──────────>│
  │                            │
  │                      Token 검증
  │                            │
  │<─── 처리 결과 ──────────────│
```

공격자는 일반적으로 다른 출처의 페이지에서 정상적인 CSRF Token을 읽을 수 없으므로 요청 위조가 어려워진다.

**2. SameSite Cookie**

Cookie가 Cross-Site 요청에 포함되는 범위를 제한한다.

| 설정 | 특징 |
|---|---|
| `Strict` | Cross-Site 요청에 Cookie 전송 제한 |
| `Lax` | 일부 최상위 페이지 이동 등에서 Cookie 허용 |
| `None` | Cross-Site 요청에도 Cookie 허용. `Secure` 필요 |

**3. Origin / Referer 검증**

요청이 허용된 출처에서 발생했는지 확인하는 보조 방어 수단이다.

### XSS와 CSRF의 차이

| 구분 | XSS | CSRF |
|---|---|---|
| 공격 방식 | 악성 JavaScript 실행 | 인증된 사용자의 요청 위조 |
| 악용 대상 | 웹페이지의 스크립트 실행 환경 | 사용자의 인증 상태 |
| 주요 방어 | 출력 인코딩, CSP | CSRF Token, SameSite |
| 핵심 위험 | 정보 탈취, 페이지 변조 등 | 의도하지 않은 상태 변경 |

**CORS는 CSRF의 직접적인 방어 수단이 아니다.** CORS가 응답 읽기를 제한하더라도 일부 Cross-Origin 요청은 서버에 전송될 수 있기 때문이다.

---

## 5. 인증 / 인가

### 인증(Authentication)

> 인증은 요청을 보낸 사용자가 누구인지 확인하는 과정이다.

대표적인 인증 방식은 다음과 같다.

- ID / Password
- Session
- JWT
- OAuth 기반 로그인
- 생체인증 및 다중 인증(MFA)

```text
사용자 로그인 요청
        ↓
ID / Password 검증
        ↓
사용자 신원 확인
        ↓
인증 성공
```

### 인가(Authorization)

> 인가는 인증된 사용자가 특정 리소스에 접근하거나 작업을 수행할 권한이 있는지 확인하는 과정이다.

예를 들어 일반 사용자는 자신의 주문을 조회할 수 있지만, 다른 사용자의 주문은 조회할 수 없어야 한다.

```text
GET /orders/100
        ↓
사용자 인증 확인
        ↓
주문 접근 권한 확인
        ↓
허용 또는 거부
```

### 인증과 인가의 차이

| 구분 | 인증 | 인가 |
|---|---|---|
| 목적 | 사용자 신원 확인 | 접근 권한 확인 |
| 핵심 질문 | 누구인가? | 무엇을 할 수 있는가? |
| 예시 | 로그인 | 관리자 API 접근 |
| HTTP 상태 코드 | 401 Unauthorized | 403 Forbidden |

HTTP에서 `401 Unauthorized`는 이름과 달리 일반적으로 인증 정보가 없거나 유효하지 않을 때 사용한다.

`403 Forbidden`은 서버가 요청을 이해했지만 접근을 허용하지 않는 경우에 사용한다.

### RBAC (Role-Based Access Control)

> RBAC는 사용자에게 부여된 역할(Role)을 기준으로 접근 권한을 관리하는 방식이다.

```text
User
 ├─ ROLE_USER
 │    └─ 일반 기능 접근
 │
 └─ ROLE_ADMIN
      └─ 관리자 기능 접근
```

Spring Security에서는 역할에 따라 API 접근을 제한할 수 있다.

```java
.requestMatchers("/admin/**").hasRole("ADMIN")
.requestMatchers("/users/**").authenticated()
```

단, 역할 확인만으로 모든 인가 문제가 해결되는 것은 아니다.

예를 들어 일반 사용자가 다른 사용자의 주문 ID를 입력하는 경우에는 **해당 리소스의 소유자인지** 추가로 검증해야 한다.

```java
if (!order.getUserId().equals(currentUserId)) {
    throw new AccessDeniedException("접근 권한이 없습니다.");
}
```

---

## 6. Cookie / Session

### Cookie

> Cookie는 서버가 브라우저에 저장하도록 전달하는 작은 데이터이며, 브라우저는 조건에 맞는 HTTP 요청에 Cookie를 첨부한다.

HTTP는 Stateless 프로토콜이므로 서버가 이전 요청의 상태를 자동으로 기억하지 않는다.

Cookie를 이용하면 요청 간 사용자 식별 정보를 전달할 수 있다.

### Cookie 동작 과정

```text
Client                      Server
  │                            │
  │──── 로그인 요청 ───────────>│
  │<─── Set-Cookie ─────────────│
  │                            │
  │──── Cookie 포함 요청 ──────>│
  │<─── 응답 ───────────────────│
```

```http
Set-Cookie: sessionId=abc123; HttpOnly; Secure; SameSite=Lax
```

| 속성 | 역할 |
|---|---|
| `HttpOnly` | JavaScript에서 Cookie 접근 제한 |
| `Secure` | HTTPS 연결에서만 Cookie 전송 |
| `SameSite` | Cross-Site 요청 시 Cookie 전송 제한 |
| `Max-Age` | Cookie 유지 시간 |
| `Domain` | Cookie가 전송될 도메인 범위 |
| `Path` | Cookie가 전송될 경로 범위 |

Cookie 자체가 인증 방식은 아니며, Session ID나 Token 등을 전달하는 수단으로 사용된다.

### Session

> Session은 서버가 사용자별 상태를 저장하고 관리하는 방식이다.

일반적인 Session 인증에서는 브라우저에 Session ID를 Cookie로 저장하고, 실제 인증 정보는 서버에서 관리한다.

```text
Client                      Server
  │                            │
  │──── ID / Password ────────>│
  │                            │
  │                      로그인 검증
  │                      Session 생성
  │                            │
  │<─── Session ID Cookie ──────│
  │                            │
  │──── Session ID ────────────>│
  │                            │
  │                      Session 조회
  │                      사용자 식별
  │                            │
  │<─── 응답 ───────────────────│
```

서버의 Session 저장소에는 다음과 같은 정보가 저장될 수 있다.

```text
Session ID: abc123
    ↓
User ID: 10
Role: USER
Expires At: ...
```

### Session 인증의 장단점

**장점**

- 인증 정보를 서버에서 관리할 수 있음
- 로그아웃 시 Session을 삭제하여 인증 상태를 무효화할 수 있음
- 사용자 권한 변경 등을 서버에서 반영하기 용이함

**단점**

- 서버에서 Session 저장 공간이 필요함
- 서버가 여러 대라면 Session 공유 방식을 고려해야 함
- Session 저장소의 가용성과 성능을 관리해야 함

### 분산 환경에서의 Session 관리

서버가 여러 대인 경우 Session을 각 서버의 메모리에만 저장하면 문제가 발생할 수 있다.

```text
             Load Balancer
              /         \
             ↓           ↓
         Server A     Server B
         Session O    Session X
```

Server A에서 로그인한 사용자의 다음 요청이 Server B로 전달되면 Session을 찾지 못할 수 있다.

이를 해결하는 대표적인 방식은 다음과 같다.

| 방식 | 특징 |
|---|---|
| Sticky Session | 동일 사용자의 요청을 같은 서버로 전달 |
| Session Replication | 서버 간 Session 데이터 복제 |
| Centralized Session Store | Redis 등 외부 저장소에 Session 공유 |

Spring 환경에서는 Spring Session과 Redis를 이용해 여러 서버가 동일한 Session 저장소를 사용할 수 있다.

```text
             Load Balancer
              /         \
             ↓           ↓
         Server A     Server B
              \         /
               ↓       ↓
                  Redis
              Session Store
```

### Cookie와 Session의 차이

| 구분 | Cookie | Session |
|---|---|---|
| 개념 | 브라우저에 데이터를 저장하고 전달하는 메커니즘 | 서버에서 사용자 상태를 관리하는 방식 |
| 저장 위치 | 브라우저 | 서버 또는 외부 Session 저장소 |
| 저장 정보 | Session ID, 설정값 등 | 사용자 인증 상태 등 |
| 보안 고려 | 탈취, XSS, CSRF | Session 탈취, 만료, 저장소 관리 |

**Cookie와 Session은 대체 관계가 아니다.** Session 인증에서는 Cookie를 이용해 Session ID를 전달하는 경우가 일반적이다.

또한 JWT도 Cookie에 저장할 수 있으므로, **Cookie/Session과 Session/JWT는 서로 다른 기준의 개념**이다.

## 7. JWT (JSON Web Token)

> JWT는 사용자 정보와 권한 등의 Claim을 JSON 형태로 표현하고, 서명 또는 암호화를 통해 전달할 수 있도록 정의된 Token 형식이다.

일반적인 로그인 인증에서는 **서명된 JWT(JWS)**를 사용한다. 서버는 Token의 서명과 유효성을 검증하여 요청을 인증할 수 있다.

### JWT의 구조

JWT는 일반적으로 세 부분으로 구성된다.

```text
Header.Payload.Signature
```

| 구성 요소 | 역할 |
|---|---|
| Header | Token 유형과 서명 알고리즘 |
| Payload | 사용자 정보, 만료 시간 등의 Claim |
| Signature | Header와 Payload의 무결성 및 서명 검증 |

**Header**

```json
{
  "alg": "HS256",
  "typ": "JWT"
}
```

**Payload**

```json
{
  "sub": "user123",
  "role": "USER",
  "iat": 1760000000,
  "exp": 1760003600
}
```

주요 Claim은 다음과 같다.

| Claim | 의미 |
|---|---|
| `sub` | Token의 주체(Subject) |
| `iss` | 발급자(Issuer) |
| `aud` | 수신 대상(Audience) |
| `iat` | 발급 시간 |
| `exp` | 만료 시간 |
| `nbf` | 사용 가능 시작 시간 |

### JWT 인증 과정

```text
Client                     Server
  │                           │
  │──── 로그인 요청 ──────────>│
  │                      사용자 인증
  │                      JWT 발급
  │<─── JWT 반환 ──────────────│
  │                           │
  │──── JWT 포함 요청 ────────>│
  │                      JWT 검증
  │                      사용자 식별
  │<─── API 응답 ──────────────│
```

일반적으로 HTTP Authorization Header에 Token을 전달한다.

```http
Authorization: Bearer <access_token>
```

서명된 JWT는 서버가 별도의 Session 저장소를 조회하지 않고도 서명과 Claim을 검증할 수 있다는 특징이 있다.

다만 JWT를 사용한다고 해서 반드시 서버가 상태를 저장하지 않는 것은 아니다. 로그아웃, Token 폐기, Refresh Token 관리 등을 위해 별도 저장소를 사용할 수 있다.

### JWT의 서명과 암호화

일반적인 서명 JWT의 Header와 Payload는 **Base64URL로 인코딩**된다.

Base64URL은 암호화 방식이 아니므로 Payload의 내용을 확인할 수 있다.

```text
Header + Payload
       ↓
서명 알고리즘
       ↓
Signature 생성
```

서명은 Token이 변조되지 않았는지 검증하기 위한 것이다.

따라서 비밀번호나 주민등록번호 등 민감한 정보를 JWT Payload에 저장해서는 안 된다.

### JWT의 장단점

| 장점 | 단점 |
|---|---|
| 서버에서 Session 조회 없이 검증 가능 | 발급된 Token을 즉시 무효화하기 어려움 |
| 분산 환경에서 인증 처리에 활용 가능 | Token 탈취 시 만료 전까지 악용 가능 |
| Claim을 통한 정보 전달 | Payload 변경 시 Token 재발급 필요 |

JWT는 인증 정보를 전달하는 방식 중 하나이며, Session보다 항상 안전하거나 성능이 좋은 것은 아니다.

---

## 8. OAuth 2.0

> OAuth 2.0은 사용자가 자신의 비밀번호를 제3자 애플리케이션에 제공하지 않고도, 특정 리소스에 대한 접근 권한을 위임할 수 있도록 하는 인가 프레임워크이다.

대표적으로 Google, Kakao 등의 계정을 이용한 서비스 연동에 활용된다.

다만 **OAuth 2.0 자체는 인증 프로토콜이 아니다.** 사용자 신원 확인을 표준화하려면 OAuth 2.0을 기반으로 하는 OpenID Connect(OIDC)를 사용할 수 있다.

### OAuth 2.0의 주요 구성 요소

| 구성 요소 | 역할 |
|---|---|
| Resource Owner | 리소스 소유자, 일반적으로 사용자 |
| Client | 리소스 접근 권한을 요청하는 애플리케이션 |
| Authorization Server | 사용자 동의 및 Token 발급 |
| Resource Server | 보호된 리소스를 제공하는 서버 |

예를 들어 Google 계정 연동에서는 다음과 같이 구분할 수 있다.

```text
Resource Owner
→ 사용자

Client
→ 우리 서비스

Authorization Server
→ Google 인증 및 인가 서버

Resource Server
→ Google 사용자 정보 API
```

### Authorization Code Flow

OAuth 2.0에서 널리 사용하는 방식으로, Authorization Code를 발급받은 뒤 Access Token으로 교환한다.

```text
사용자          Client          Authorization Server
  │               │                       │
  │─ 로그인 요청 ─>│                       │
  │               │                       │
  │<── 인가 페이지로 이동 ──────────────────│
  │                                       │
  │──────── 로그인 및 동의 ────────────────>│
  │                                       │
  │<──── Authorization Code ──────────────│
  │               │                       │
  │               │── Code + 검증 정보 ───>│
  │               │<──── Access Token ────│
  │               │                       │
```

발급받은 Access Token을 이용해 Resource Server에 접근할 수 있다.

```http
GET /userinfo
Authorization: Bearer <access_token>
```

Authorization Code Flow에서는 **PKCE(Proof Key for Code Exchange)**를 함께 사용하는 것이 권장된다.

PKCE는 Authorization Code가 탈취되더라도 공격자가 이를 Token으로 교환하기 어렵게 한다.

### Access Token과 Refresh Token

**Access Token**

보호된 리소스에 접근하기 위한 Token이다.

**Refresh Token**

Access Token이 만료되었을 때 새로운 Access Token을 발급받는 데 사용하는 Token이다.

OAuth 2.0에서 Refresh Token은 선택적으로 발급되며, 제공 여부와 정책은 Authorization Server에 따라 달라진다.

### OAuth 2.0과 JWT의 차이

| 구분 | OAuth 2.0 | JWT |
|---|---|---|
| 개념 | 접근 권한 위임을 위한 프레임워크 | 정보를 전달하는 Token 형식 |
| 목적 | 리소스 접근 권한 부여 | Claim 전달 및 무결성 검증 |
| 주요 요소 | Authorization Code, Access Token 등 | Header, Payload, Signature |
| 관계 | JWT를 Token 형식으로 사용할 수 있음 | OAuth 2.0 없이도 사용 가능 |

OAuth 2.0의 Access Token이 반드시 JWT인 것은 아니다. 서버가 내부적으로 관리하는 불투명한 문자열(Opaque Token)일 수도 있다.

---

## 9. Spring Security

> Spring Security는 Spring 애플리케이션에서 인증, 인가 및 주요 웹 보안 기능을 제공하는 프레임워크이다.

Servlet 기반 Spring Boot 애플리케이션에서는 **Filter Chain**을 중심으로 보안 처리가 이루어진다.

### Spring Security 동작 구조

```text
Client
  ↓
HTTP Request
  ↓
Servlet Filter Chain
  ↓
Spring Security Filter Chain
  ├─ 인증 처리
  ├─ SecurityContext 관리
  └─ 인가 처리
  ↓
DispatcherServlet
  ↓
Controller
```

Spring Security는 여러 Security Filter를 연결하여 요청을 처리한다.

JWT 인증을 사용하는 경우에는 일반적으로 Custom Filter 또는 OAuth2 Resource Server 기능을 통해 Token을 검증한다.

### Authentication과 SecurityContext

**Authentication**

현재 사용자의 인증 정보와 권한을 표현하는 객체이다.

주요 구성 요소는 다음과 같다.

- `principal`: 사용자 식별 정보
- `credentials`: 비밀번호 등 인증에 사용되는 정보
- `authorities`: 사용자에게 부여된 권한
- `authenticated`: 인증 여부

**SecurityContext**

현재 요청에서 사용하는 Authentication 정보를 보관한다.

```text
SecurityContext
      ↓
Authentication
  ├─ Principal
  ├─ Credentials
  └─ Authorities
```

일반적인 Servlet 환경에서는 `SecurityContextHolder`를 통해 현재 인증 정보에 접근할 수 있다.

기본 전략은 ThreadLocal 기반이므로 비동기 작업이나 다른 Thread로 인증 정보를 전달할 때 주의해야 한다.

### SecurityFilterChain 설정

Spring Security 6 기준의 기본적인 설정 예시이다.

```java
@Configuration
@EnableWebSecurity
public class SecurityConfig {

    @Bean
    public SecurityFilterChain securityFilterChain(
            HttpSecurity http
    ) throws Exception {

        return http
            .authorizeHttpRequests(auth -> auth
                .requestMatchers("/auth/**").permitAll()
                .requestMatchers("/admin/**").hasRole("ADMIN")
                .anyRequest().authenticated()
            )
            .build();
    }
}
```

| 설정 | 의미 |
|---|---|
| `permitAll()` | 인증 없이 접근 허용 |
| `authenticated()` | 인증된 사용자만 접근 |
| `hasRole("ADMIN")` | ADMIN 역할 필요 |
| `hasAuthority("...")` | 특정 권한 필요 |

`hasRole("ADMIN")`은 기본적으로 `ROLE_ADMIN` 권한을 검사한다.

### JWT 기반 인증 처리

JWT 인증을 적용하면 일반적으로 다음과 같은 흐름으로 동작한다.

```text
HTTP Request
     ↓
Authorization Header 확인
     ↓
JWT 서명 및 Claim 검증
     ↓
Authentication 생성
     ↓
SecurityContext에 저장
     ↓
인가 처리
     ↓
Controller 실행
```

JWT 검증에 실패하면 인증되지 않은 요청으로 처리하며, 보호된 리소스에 접근할 경우 일반적으로 401 응답을 반환한다.

인증은 되었지만 필요한 권한이 없다면 일반적으로 403 응답을 반환한다.

### Session 기반 vs JWT 기반 설정

**Session 기반 인증**

서버에서 Session을 관리하고 요청마다 Session을 통해 인증 상태를 확인한다.

**JWT 기반 인증**

요청에 포함된 Token을 검증하여 인증 정보를 구성한다.

JWT 기반 Stateless API에서는 다음과 같이 Session 생성 정책을 설정할 수 있다.

```java
.sessionManagement(session -> session
    .sessionCreationPolicy(SessionCreationPolicy.STATELESS)
)
```

`STATELESS`는 Spring Security가 인증 상태 유지를 위해 HTTP Session을 생성하거나 사용하지 않도록 하는 설정이다.

단, 이 설정만으로 애플리케이션의 모든 Session 사용이 금지되는 것은 아니다.

### Spring Security와 CSRF

Spring Security는 기본적으로 CSRF 방어 기능을 제공한다.

Cookie 기반 Session 인증에서는 CSRF 방어를 유지하는 것이 일반적이다.

반면 Authorization Header에 Bearer Token을 명시적으로 전달하는 Stateless API에서는 CSRF 방어가 불필요할 수 있다.

```java
.csrf(csrf -> csrf.disable())
```

그러나 JWT를 Cookie에 저장하고 브라우저가 자동 전송하도록 구성했다면 CSRF 위험이 존재할 수 있다.

따라서 **Session인지 JWT인지가 아니라, 브라우저가 인증 정보를 자동으로 전송하는 구조인지**를 기준으로 CSRF 방어 필요성을 판단해야 한다.

## 10. JWT 검증

> JWT 검증은 클라이언트가 전달한 Token의 서명과 Claim을 확인하여 신뢰할 수 있는 인증 정보인지 판단하는 과정이다.

JWT는 서버가 발급했다는 이유만으로 신뢰할 수 없다. 요청마다 Token의 유효성을 검증해야 한다.

### JWT 검증 과정

```text
Client
  │
  │ Authorization: Bearer <JWT>
  ↓
Server
  │
  ├─ 1. Token 형식 확인
  ├─ 2. 서명 검증
  ├─ 3. 만료 시간 검증
  ├─ 4. 발급자 및 수신 대상 검증
  └─ 5. 사용자 권한 확인
  ↓
Authentication 생성
  ↓
API 접근 허용
```

**1. 서명 검증**

JWT의 Header와 Payload가 발급 이후 변조되지 않았는지 확인한다.

```text
Header.Payload
       +
   서명 검증 Key
       ↓
Signature 검증
```

대표적인 서명 알고리즘은 다음과 같다.

| 알고리즘 | 특징 |
|---|---|
| HS256 | 동일한 Secret Key로 서명 및 검증 |
| RS256 | Private Key로 서명, Public Key로 검증 |
| ES256 | 타원곡선 기반 비대칭키 서명 |

HS256에서는 Secret Key를 공유해야 하지만, RS256에서는 검증 서버에 Public Key만 전달할 수 있다.

서버는 **허용할 서명 알고리즘을 명시적으로 제한**해야 한다. Token의 Header에 적힌 알고리즘을 무조건 신뢰해서는 안 된다.

**2. Claim 검증**

서명이 유효하더라도 Token의 사용 조건을 확인해야 한다.

| Claim | 검증 내용 |
|---|---|
| `exp` | 만료되지 않았는지 |
| `nbf` | 사용 가능한 시간이 되었는지 |
| `iss` | 신뢰하는 발급자인지 |
| `aud` | 해당 서비스에 발급된 Token인지 |
| `sub` | 사용자 식별 정보가 유효한지 |

서비스 정책에 따라 Token 유형, 권한, 사용자 상태 등도 추가로 확인할 수 있다.

### JWT 검증 시 주의사항

**1. Payload는 암호화된 데이터가 아니다.**

JWT의 Payload는 디코딩할 수 있으므로 비밀번호 등 민감한 정보를 저장하지 않아야 한다.

**2. 서명 검증만으로 충분하지 않다.**

서명이 유효해도 만료된 Token이거나 다른 서비스를 대상으로 발급된 Token일 수 있다.

**3. 로그아웃한 Token도 유효할 수 있다.**

JWT는 기본적으로 만료 전까지 서명 검증에 성공할 수 있다. 즉시 무효화가 필요하다면 별도의 폐기 목록이나 사용자 상태 검증 등이 필요하다.

**4. 인증과 인가는 별도로 처리해야 한다.**

JWT가 유효하더라도 해당 사용자가 특정 리소스에 접근할 권한이 있는지는 별도로 확인해야 한다.

---

## 11. Refresh Token

> Refresh Token은 Access Token이 만료되었을 때 새로운 Access Token을 발급받기 위해 사용하는 Token이다.

Access Token의 유효기간을 짧게 설정하면서 사용자의 로그인 상태를 유지하기 위해 사용한다.

### Access Token과 Refresh Token

| 구분 | Access Token | Refresh Token |
|---|---|---|
| 목적 | API 접근 | Access Token 재발급 |
| 유효기간 | 상대적으로 짧음 | 상대적으로 김 |
| 사용 위치 | 보호된 API 요청 | Token 재발급 요청 |
| 탈취 위험 | API 접근 가능 | 새로운 Access Token 발급 가능 |

Refresh Token은 반드시 JWT일 필요는 없다. 서버가 생성한 무작위 문자열을 사용할 수도 있다.

### Token 재발급 과정

```text
Client                       Server
  │                             │
  │──── Access Token ──────────>│
  │<─── 401 (Token 만료) ────────│
  │                             │
  │──── Refresh Token ─────────>│
  │                        Token 검증
  │                             │
  │<─── 새 Access Token ─────────│
  │                             │
  │──── 새 Access Token ────────>│
  │<─── API 응답 ────────────────│
```

Refresh Token이 만료되었거나 유효하지 않다면 일반적으로 다시 로그인해야 한다.

### Refresh Token 저장 방식

**1. 서버 저장소에 관리**

Redis 또는 DB에 Refresh Token이나 그 해시값을 저장한다.

```text
User ID: 10
Token Hash: ...
Expires At: ...
```

장점은 로그아웃이나 보안 사고 발생 시 서버에서 Token을 무효화할 수 있다는 것이다.

단, 저장소 조회와 관리 비용이 발생한다.

**2. 서버 저장소 없이 검증**

서명된 JWT를 Refresh Token으로 사용하고 서버에서 서명과 만료 시간만 검증할 수도 있다.

별도 저장소가 필요하지 않지만, 개별 Token의 즉시 무효화나 재사용 탐지가 어려워질 수 있다.

### Refresh Token Rotation

> Refresh Token Rotation은 Refresh Token을 사용할 때마다 새로운 Refresh Token을 발급하고 기존 Token을 무효화하는 방식이다.

```text
Refresh Token A
       ↓
재발급 요청
       ↓
Access Token B
Refresh Token B 발급
       ↓
Refresh Token A 무효화
```

이미 사용한 Refresh Token이 다시 제출되면 Token 탈취 및 재사용 가능성을 의심할 수 있다.

이때 서버는 관련 Token을 무효화하고 재인증을 요구하는 정책을 적용할 수 있다.

동시 재발급 요청으로 인한 충돌도 고려해야 하므로 Token 교체 과정의 원자성이 중요하다.

### 로그아웃 처리

Access Token과 Refresh Token의 무효화 방식은 다를 수 있다.

```text
로그아웃 요청
    ↓
Refresh Token 삭제 또는 무효화
    ↓
Access Token 폐기 처리 (필요 시)
    ↓
클라이언트 인증 정보 제거
```

Refresh Token만 삭제하면 새로운 Access Token 발급은 막을 수 있지만, 이미 발급된 Access Token은 만료 전까지 유효할 수 있다.

즉시 로그아웃을 보장하려면 Access Token 폐기 목록 등의 추가적인 설계가 필요하다.

---

## 12. OAuth 서버 처리

> OAuth 로그인에서 백엔드 서버는 Authorization Code를 전달받아 OAuth 제공자의 Token으로 교환하고, 사용자 정보를 확인한 뒤 서비스 자체의 인증 상태를 생성한다.

OAuth 제공자의 인증과 우리 서비스의 인증은 구분해야 한다.

### OAuth 로그인 전체 흐름

```text
Client          Backend          OAuth Provider
  │                │                   │
  │─ 로그인 요청 ──>│                   │
  │                │                   │
  │<── 인가 URL ────│                   │
  │                                    │
  │──────── 사용자 로그인 및 동의 ───────>│
  │                                    │
  │<──────── Authorization Code ───────│
  │                │                   │
  │── Code 전달 ──>│                   │
  │                │── Token 요청 ────>│
  │                │<── Access Token ──│
  │                │                   │
  │                │── 사용자 정보 요청 >│
  │                │<── 사용자 정보 ───│
  │                │                   │
  │                │ 사용자 조회/가입   │
  │                │ 서비스 Token 발급  │
  │                │                   │
  │<── 서비스 JWT ──│                   │
```

Authorization Code가 백엔드의 Redirect URI로 직접 전달되는 구조도 가능하다.

### 백엔드의 주요 처리 단계

**1. Authorization Code 수신**

OAuth 제공자가 인증과 사용자 동의를 완료하면 등록된 Redirect URI로 Authorization Code를 전달한다.

이때 요청을 시작한 사용자와 Callback을 연결하기 위해 `state` 값을 검증해야 한다.

**2. Access Token 요청**

백엔드는 Authorization Code를 OAuth 제공자의 Token Endpoint에 전달한다.

```http
POST /token
Content-Type: application/x-www-form-urlencoded

grant_type=authorization_code
&code=...
&redirect_uri=...
&client_id=...
&code_verifier=...
```

Client 인증 방식과 PKCE 적용 여부에 따라 추가 파라미터가 필요할 수 있다.

**3. 사용자 정보 조회**

발급받은 Access Token으로 사용자 정보 API를 호출한다.

```http
GET /userinfo
Authorization: Bearer <provider_access_token>
```

OAuth 제공자마다 사용자 정보 API와 응답 구조가 다를 수 있다.

OIDC를 사용하는 경우에는 ID Token의 서명과 `iss`, `aud`, `exp`, `nonce` 등 필요한 Claim을 검증하여 사용자 인증 정보를 확인할 수 있다.

**4. 서비스 사용자 조회 또는 생성**

OAuth 제공자의 사용자 식별자를 기준으로 기존 회원을 조회한다.

```text
Provider: GOOGLE
Provider User ID: 123456
          ↓
서비스 회원 조회
          ↓
기존 회원 → 로그인
신규 회원 → 회원가입 처리
```

이메일은 변경되거나 검증되지 않았을 수 있으므로, 일반적으로 **Provider와 Provider User ID를 조합하여 외부 계정을 식별**한다.

**5. 서비스 인증 정보 발급**

사용자 확인이 완료되면 우리 서비스의 Session 또는 JWT를 발급한다.

```text
Google Access Token
→ Google API 접근에 사용

Service Access Token
→ 우리 서비스 API 접근에 사용
```

두 Token은 발급 주체와 사용 목적이 다르므로 구분해서 관리해야 한다.

### Spring Security OAuth2 Login

Spring Security는 OAuth2 Login 기능을 제공한다.

```java
http.oauth2Login(oauth2 -> oauth2
    .userInfoEndpoint(userInfo -> userInfo
        .userService(customOAuth2UserService)
    )
);
```

`OAuth2UserService`에서는 OAuth 제공자로부터 받은 사용자 정보를 서비스의 사용자 모델에 맞게 변환하거나 회원 조회 및 가입 처리를 수행할 수 있다.

OAuth 로그인 성공 후 별도의 서비스 JWT를 발급하려면 성공 핸들러 등에서 후속 처리를 구성할 수 있다.

### OAuth 서버 처리 시 주의사항

| 항목 | 주의사항 |
|---|---|
| `state` | 로그인 요청과 Callback 연결 및 CSRF 방어 |
| PKCE | Authorization Code 탈취 위험 완화 |
| Redirect URI | 사전에 등록된 URI 사용 |
| Client Secret | 서버에서 안전하게 관리 |
| 사용자 식별 | Provider + Provider User ID 사용 |
| Token 저장 | 필요한 Token만 안전하게 저장 |
| Token 구분 | OAuth 제공자 Token과 서비스 Token 구분 |

OAuth 로그인에서 중요한 것은 **외부 제공자가 인증한 사용자를 우리 서비스의 사용자와 안전하게 연결하고, 이후 접근 권한은 서비스 정책에 따라 별도로 관리하는 것**이다.
