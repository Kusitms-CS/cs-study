# 7주차 - Web Security / 인증 개념 정리

## 목차
- [키워드 연관관계](#키워드-연관관계)
- [SOP](#sop)
- [CORS](#cors)
- [XSS](#xss)
- [CSRF](#csrf)
- [인증/인가](#인증인가)
- [Cookie/Session](#cookiesession)
- [JWT](#jwt)
- [OAuth](#oauth)
- [Backend](#backend)
  - [Spring Security](#spring-security)
  - [JWT 검증](#jwt-검증)
  - [Refresh Token](#refresh-token)
  - [OAuth 서버 처리](#oauth-서버-처리)

---

## 키워드 연관관계

```
브라우저 보안
├─ SOP ──(완화)──> CORS
├─ XSS   (SOP로 막을 수 없는 동일 출처 내 스크립트 실행)
└─ CSRF  (쿠키 자동 첨부를 이용한 요청 위조)

인증/인가
├─ 인증 ──> Cookie/Session (Stateful)
│       └─> JWT (Stateless) ──> Access Token + Refresh Token
└─ 인가 ──> 권한(Role/Scope) 검사

OAuth : 인가 위임 프레임워크 ──> Access Token 발급 ──> (자체 JWT/세션 발급과 연계)
```

| 연결 | 관계 설명 |
|---|---|
| SOP → CORS | SOP는 브라우저가 다른 Origin 응답을 스크립트에 노출하지 않도록 제한하는 정책이고, CORS는 서버가 `Access-Control-Allow-*` 헤더로 허용 Origin을 명시해 그 제한을 선택적으로 해제하는 HTTP 헤더 기반 메커니즘 |
| SOP ↔ XSS | SOP는 Origin 간 접근을 제한하지만, XSS로 삽입된 스크립트는 대상 사이트와 동일한 Origin에서 실행되므로 SOP의 제한을 받지 않음 |
| XSS ↔ 토큰 저장 | XSS로 실행된 스크립트는 동일 Origin의 JS 접근 가능 저장소(localStorage, 일반 쿠키)를 읽을 수 있으므로 토큰이 탈취될 수 있음. `HttpOnly` 쿠키는 JS 접근이 차단되어 탈취를 막음 |
| Cookie ↔ CSRF | 브라우저는 요청 출처와 무관하게 대상 도메인의 쿠키를 자동 첨부하므로, 외부 사이트가 유발한 요청도 인증된 요청으로 처리될 수 있음. `SameSite` 쿠키와 CSRF 토큰으로 이를 차단 |
| 인증 → Cookie/Session, JWT | HTTP는 Stateless이므로 인증 결과를 후속 요청에 연결하는 수단이 필요함. Session은 서버가 상태를 저장하고 식별자(Session ID)만 전달하며, JWT는 서명된 토큰에 상태를 담아 서버 저장 없이 검증 |
| 인증 → 인가 | 인증으로 확인된 주체(Principal)의 권한(Role, Scope)을 기준으로 자원 접근 가능 여부를 판단하는 후속 단계 |
| JWT → Refresh Token | 발급 후 서버가 무효화하기 어려운 JWT의 한계를 보완하기 위해, 수명이 짧은 Access Token과 서버에서 폐기 가능한 수명이 긴 Refresh Token으로 역할을 분리 |
| OAuth → JWT | OAuth는 Access Token 발급까지의 인가 절차를 정의할 뿐 토큰 형식은 규정하지 않음. 로그인 연동 시 서비스는 OAuth 결과로 얻은 사용자 정보를 바탕으로 자체 JWT 또는 세션을 별도로 발급하는 구성이 일반적 |
| 인증/인가 → Spring Security | 서블릿 Filter Chain에서 요청의 인증 정보 추출·검증과 인가 판단을 수행하는 Spring 보안 프레임워크 |

---

## SOP

- **SOP(Same-Origin Policy, 동일 출처 정책)**: 브라우저가 **다른 Origin의 리소스를 JS로 읽는 것**을 제한하는 보안 정책
- **Origin** = `프로토콜 + 호스트(도메인) + 포트` 세 가지가 모두 같아야 같은 출처

| URL (기준 `https://a.com`) | 같은 출처? | 이유 |
|---|---|---|
| `https://a.com/page` | O | 경로만 다름 |
| `http://a.com` | X | 프로토콜 다름 |
| `https://b.com` | X | 도메인 다름 |
| `https://a.com:8080` | X | 포트 다름 |

- 요청 자체는 서버로 **나가지만**, 브라우저가 **응답을 JS에 넘겨주지 않음** 


---

## CORS

- **CORS(Cross-Origin Resource Sharing)**: 서버가 응답 헤더로 **허용할 Origin을 알려주어 SOP를 예외적으로 허용**하는 방식
- 핵심 헤더
  - `Access-Control-Allow-Origin`: 허용할 출처
  - `Access-Control-Allow-Methods` / `Allow-Headers`: 허용할 메서드/헤더
  - `Access-Control-Allow-Credentials`: 쿠키 등 인증 정보 포함 허용 (이때 Origin에 `*` 사용 불가)
- **Preflight 요청**: 실제 요청 전에 브라우저가 `OPTIONS` 메서드로 "이 요청 보내도 되나요?" 먼저 확인
  - 단순 요청(Simple Request: GET/POST + 기본 헤더)이 아니면 발생 (예: `Authorization` 헤더, JSON Content-Type, PUT/DELETE)
- CORS 에러는 **서버 설정 문제**이며 브라우저에서만 발생 (Postman은 에러 없음)

---

## XSS


- **XSS(Cross-Site Scripting)**: 공격자가 웹사이트에 **악성 스크립트를 삽입**해 다른 사용자의 브라우저에서 실행시키는 공격
- 피해: 쿠키/토큰 탈취, 사용자 대신 요청 전송, 피싱 화면 노출


| 종류 | 설명 |
|---|---|
| Stored XSS | 악성 스크립트가 DB에 저장되어 글을 보는 모든 사용자에게 실행 (게시글, 댓글) |
| Reflected XSS | URL 파라미터 등이 응답에 그대로 반영되어 실행 (악성 링크 클릭 유도) |
| DOM-based XSS | 서버 거치지 않고 클라이언트 JS가 DOM을 조작하는 과정에서 발생 |

- 방어
  - 출력 시 **이스케이프(Escape)**: `<` → `&lt;` 로 변환
  - 입력값 검증 / Sanitize
  - **HttpOnly 쿠키**: JS에서 쿠키 접근 차단
  - **CSP(Content-Security-Policy)**: 허용된 스크립트 출처만 실행

---

## CSRF

- **CSRF(Cross-Site Request Forgery)**: 로그인된 사용자의 **브라우저가 쿠키를 자동 첨부**하는 점을 이용해, 사용자가 원치 않는 요청을 보내게 하는 공격
- 발생 조건: ① 쿠키(세션) 기반 인증 ② 공격자가 요청 형태를 예측 가능 ③ 사용자가 이미 로그인 상태
- XSS와의 차이

| | XSS | CSRF |
|---|---|---|
| 악용하는 것 | 사용자가 사이트를 **신뢰**하는 점 | 사이트가 사용자 브라우저를 **신뢰**하는 점 |
| 공격 위치 | 대상 사이트 안에서 스크립트 실행 | 외부 사이트에서 대상 사이트로 요청 위조 |
| 공격자가 응답을 읽나 | 가능 | 불가능 (요청만 보냄) |

- 방어
  - **SameSite 쿠키** (`Lax`/`Strict`): 다른 사이트에서 온 요청에 쿠키 미첨부
  - **CSRF 토큰**: 서버가 발급한 임의 토큰을 요청에 함께 보내 검증
  - 중요 요청은 Referer/Origin 검증, 재인증
  - Authorization 헤더로 토큰을 직접 넣는 방식은 자동 첨부가 없어 CSRF에 상대적으로 안전

---

## 인증/인가

- **인증(Authentication)**: "**너 누구야?**" 사용자의 신원을 확인 (로그인)
- **인가(Authorization)**: "**너 이거 해도 돼?**" 인증된 사용자의 권한 확인 (관리자만 접근 등)
- 순서: 인증 → 인가
- HTTP 상태 코드

| 코드 | 의미 |
|---|---|
| 401 Unauthorized | 인증 안 됨 (로그인 필요 / 토큰 없음·만료) |
| 403 Forbidden | 인증은 됐지만 권한 없음 |

- HTTP는 **Stateless**라서 요청마다 "누구인지"를 알려줄 방법이 필요 → Cookie/Session, JWT

---

## Cookie/Session


- **Cookie**: 서버가 브라우저에 저장시키는 작은 데이터. 이후 요청마다 **브라우저가 자동으로 첨부**
- **Session**: 서버가 사용자 상태를 **서버 메모리/DB/Redis**에 저장하고, 클라이언트에는 **Session ID**만 쿠키로 전달

**세션 인증 흐름**
1. 로그인 성공 → 서버가 Session 생성 후 Session ID 발급
2. `Set-Cookie: JSESSIONID=abc123` 로 브라우저에 저장
3. 이후 요청마다 쿠키로 Session ID 전송 → 서버가 저장소에서 조회해 사용자 식별



**장단점**
- 장점: 서버가 세션을 직접 제어 
- 단점: 서버가 상태를 저장 → 서버 여러 대일 때 세션 공유 필요, CSRF에 취약

---

## JWT


- **JWT(JSON Web Token)**: 사용자 정보(Claims)를 담고 **서명된** 토큰. 서버가 상태를 저장하지 않는 **Stateless** 인증
- 구조: `Header.Payload.Signature` (각각 Base64URL 인코딩, `.`으로 구분)

| 구성 | 내용 |
|---|---|
| Header | 서명 알고리즘(`alg`), 토큰 타입 |
| Payload | Claims: `sub`(사용자), `iat`(발급 시각), `exp`(만료 시각), `role` 등 |
| Signature | Header+Payload를 비밀키로 서명한 값 → **위변조 검증** |


- 장점: 서버 저장소 불필요, 확장성 좋음 
- 단점: **발급한 토큰을 서버가 임의로 무효화하기 어려움**, 토큰이 길어 전송량 증가, 탈취되면 만료 전까지 악용 가능

| | Session | JWT |
|---|---|---|
| 상태 저장 | 서버 | 클라이언트  |
| 확장성 | 세션 공유 필요 | 유리 |
| 강제 만료 | 쉬움 | 어려움  |
| 주 전송 방식 | Cookie | Authorization 헤더  |

---

## OAuth

- **OAuth 2.0**: 사용자의 비밀번호를 공유하지 않고, 제3자 앱이 **제한된 권한(Scope)** 으로 자원에 접근하도록 **위임**하는 **인가 프레임워크**
- "구글로 로그인" 같은 소셜 로그인에 쓰임 (로그인 자체 표준은 OIDC, OAuth 위에 인증 계층을 얹은 것)

**등장 역할**

| 역할 | 설명 | 예시 |
|---|---|---|
| Resource Owner | 자원의 주인 | 사용자 |
| Client | 자원에 접근하려는 앱 | 우리 서비스 |
| Authorization Server | 인증 후 토큰을 발급 | 구글 인증 서버 |
| Resource Server | 보호된 자원을 가진 서버 | 구글 API 서버 |

**Authorization Code Grant 흐름 (가장 많이 사용)**
1. 사용자가 "구글 로그인" 클릭 → Client가 Authorization Server로 **리다이렉트**
2. 사용자가 로그인 및 권한 동의
3. Authorization Server가 `redirect_uri`로 **Authorization Code** 전달
4. Client(백엔드)가 Code + Client Secret으로 **Access Token 요청**
5. Access Token으로 Resource Server에서 사용자 정보/자원 조회

- Code를 한 번 거치는 이유: Access Token이 브라우저 URL에 노출되지 않게 하고, 백엔드가 Client Secret으로 안전하게 교환하려고
- **state** 파라미터: CSRF 방지용 임의 값. 요청 때 보낸 값과 돌아온 값이 같은지 검증

---

## Backend

### Spring Security

- 스프링 기반 **인증/인가를 담당하는 보안 프레임워크**. 서블릿 **Filter Chain** 형태로 요청이 Controller에 닿기 전에 처리
- 핵심 구성

| 구성 | 역할 |
|---|---|
| SecurityFilterChain | 어떤 URL에 어떤 보안 규칙(인증 필요, 권한 등)을 적용할지 정의 |
| AuthenticationManager / Provider | 인증 로직 수행 (ID/PW 검증 등) |
| UserDetailsService | 사용자 정보를 DB에서 조회 |
| SecurityContext / SecurityContextHolder | 인증된 사용자(`Authentication`)를 현재 요청(스레드)에 보관 |
| PasswordEncoder | 비밀번호 단방향 해시 (BCrypt) |


### JWT 검증

서버가 토큰을 받으면 확인하는 순서
1. 형식 확인: `Bearer` 접두어, `.` 3분할
2. **서명(Signature) 검증**: 비밀키(또는 공개키)로 위변조 여부 확인
3. **만료(`exp`) 검증**: 만료된 토큰 거부 → 401
4. 필요 시 `iss`(발급자), `aud`(대상), 블랙리스트 여부 확인
5. Claims에서 사용자/권한을 꺼내 `SecurityContext`에 저장

- 서명 알고리즘: **HS256**(대칭키, 같은 비밀키로 서명·검증) vs **RS256**(비대칭키, 개인키로 서명·공개키로 검증)
- `alg: none` 공격: 알고리즘을 none으로 바꿔 서명 검증을 우회하려는 공격 → 서버가 허용 알고리즘을 고정해야 함

### Refresh Token

- **Access Token**: API 호출용, **수명 짧게**(예: 15~30분) → 탈취돼도 피해 최소화
- **Refresh Token**: Access Token 재발급용, **수명 길게**(예: 7~14일)
- 흐름: Access 만료(401) → Refresh Token으로 재발급 요청 → 새 Access 발급 → 재시도
- Refresh Token은 **서버(DB/Redis)에 저장**해 검증·폐기 가능하게 함 (로그아웃 시 삭제)
- 보안 강화
  - **Refresh Token Rotation**: 재발급할 때마다 Refresh Token도 새로 교체, 이전 것은 폐기 → 재사용 감지 시 탈취로 판단
  - HttpOnly + Secure + SameSite 쿠키에 저장

### OAuth 서버 처리

- 백엔드가 하는 일
  1. 프론트에서 받은 **Authorization Code**로 Authorization Server에 Access Token 요청 (Client Secret 사용, 서버에서만 보관)
  2. Access Token으로 사용자 정보(이메일, 고유 ID) 조회
  3. 우리 DB에서 해당 사용자를 조회, 없으면 **회원가입 처리**
  4. 우리 서비스 전용 **JWT(또는 세션)를 자체 발급**해 응답

