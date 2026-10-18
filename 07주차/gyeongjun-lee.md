# 7주차 - Web Security / 인증

## 1. SOP (Same-Origin Policy)

### 구조

```text
Origin = scheme + host + port

https://app.example.com:443  ── 같은 origin ──> https://app.example.com/api
https://app.example.com:443  ── 다른 origin ──> https://api.example.com
http://app.example.com:80    ── 다른 origin ──> https://app.example.com
```

### 특징

- 브라우저가 한 출처에서 실행된 문서나 스크립트가 다른 출처의 응답을 함부로 읽지 못하게 하는 기본 보안 정책
- 출처는 프로토콜, 호스트, 포트가 모두 같아야 같다고 본다. 경로(`/api`, `/users`)는 출처 비교에 포함되지 않는다.
- 다른 출처로 요청을 보내는 것 자체를 전부 막는 규칙은 아니다. 예를 들어 이미지 태그나 form 요청은 가능할 수 있지만, JavaScript가 응답 내용을 읽으려면 CORS 허용이 필요하다.

### 정리

- SOP는 악성 사이트가 로그인된 사용자의 다른 사이트 데이터를 JavaScript로 읽는 일을 막는 브라우저의 기본 경계다.
- `api.example.com`과 `app.example.com`은 이름이 비슷해도 host가 달라 서로 다른 출처다.

---

## 2. CORS (Cross-Origin Resource Sharing)

### 구조

```text
Browser                 API Server
   │  Origin: https://app.example.com
   │ ─────── GET /me ───────> │
   │ <── Access-Control-Allow-Origin: https://app.example.com
   │
   └─ 응답을 JavaScript에 전달
```

### 특징

- SOP로 막히는 교차 출처 응답 읽기를 서버가 `Access-Control-Allow-Origin` 등의 응답 헤더로 허용하는 방식이다.
- 브라우저가 적용하는 정책이므로 서버 간 통신이나 curl에는 CORS 제한이 없다.
- Cookie처럼 인증 정보를 함께 보내려면 클라이언트에 `credentials: "include"`, 서버에 `Access-Control-Allow-Credentials: true`가 필요하다. 이 경우 `Access-Control-Allow-Origin: *`는 쓸 수 없다.

### 정리

- CORS는 프론트엔드에서 우회하는 오류가 아니라 API 서버가 어떤 출처에 응답 공개를 허용할지 정하는 정책이다.
- 인증이 필요한 API는 허용할 origin을 정확히 지정해야 한다.

---

## 3. XSS (Cross-Site Scripting)

### 구조

```text
게시글 입력: <img src=x onerror="fetch('/api/me').then(...)">
                    ↓
검증 없이 HTML로 렌더링
                    ↓
다른 사용자의 브라우저에서 악성 스크립트 실행
```

### 특징

- 공격자가 입력한 스크립트가 신뢰받는 사이트의 문맥에서 실행되는 취약점이다.
- 저장형 XSS(게시글·댓글 등에 저장), 반사형 XSS(URL 파라미터를 응답에 반영), DOM 기반 XSS(클라이언트 코드가 위험하게 DOM을 조작)로 나뉜다.
- React처럼 기본적으로 텍스트를 escape하는 프레임워크를 써도 `dangerouslySetInnerHTML` 같은 HTML 직접 삽입 지점은 별도로 sanitize해야 한다.
- `HttpOnly` Cookie는 JavaScript에서 Cookie 값을 읽지 못하게 해 피해를 줄이지만, XSS 자체를 막지는 못한다.

### 정리

- 사용자 입력은 HTML이 아니라 텍스트로 렌더링하고, 꼭 HTML을 허용해야 하면 검증된 sanitizer를 거쳐야 한다.
- CSP(Content Security Policy)는 허용하지 않은 스크립트 실행을 제한하는 추가 방어선이다.

---

## 4. CSRF (Cross-Site Request Forgery)

### 구조

```text
사용자는 bank.example.com에 로그인 상태 (Cookie 보유)

evil.example.com
   │  POST https://bank.example.com/transfer
   │  브라우저가 bank Cookie를 자동으로 포함
   ▼
bank.example.com이 요청을 정상 사용자 요청으로 오해할 수 있음
```

### 특징

- 공격자가 사용자의 로그인 상태를 이용해 사용자가 의도하지 않은 상태 변경 요청을 보내게 만드는 공격이다.
- Cookie 기반 인증에서 특히 문제가 된다. 브라우저가 대상 도메인의 Cookie를 요청에 자동으로 붙이기 때문이다.
- `SameSite=Lax` 또는 `Strict`, CSRF 토큰, Origin/Referer 검증을 함께 사용한다. 중요한 변경 요청은 재인증도 고려한다.
- Authorization 헤더에 토큰을 직접 넣는 방식은 브라우저가 외부 사이트의 form 요청에 그 헤더를 자동으로 붙이지 않으므로 전형적인 CSRF 위험은 낮지만, XSS 위험이 사라지는 것은 아니다.

### 정리

- CSRF는 "로그인한 사용자의 브라우저를 대신 조종"하는 공격이고, XSS는 "사이트 안에서 스크립트를 실행"하는 공격이다.
- 상태를 바꾸는 요청은 SameSite 설정만 믿지 말고 서버에서 요청 출처 또는 CSRF 토큰도 확인해야 한다.

---

## 5. 인증 / 인가

### 구조

```text
인증(Authentication): "누구인가?"      로그인 → 사용자 식별
인가(Authorization):  "해도 되는가?"   관리자 권한 확인 → 삭제 허용 여부
```

### 특징

- 인증은 사용자의 신원을 확인하는 과정이고, 인가는 인증된 사용자가 특정 자원과 행동에 권한이 있는지 판단하는 과정이다.
- 로그인에 성공했다고 모든 API를 호출할 수 있는 것은 아니다. 서버는 각 요청에서 소유자·역할·권한을 확인해야 한다.
- 프론트엔드에서 버튼을 숨기는 것은 UX 처리일 뿐 보안이 아니다. 실제 인가는 반드시 서버가 강제한다.

### 정리

- "로그인했는가"와 "이 글을 삭제할 권한이 있는가"는 다른 질문이다.
- 권한 검사를 클라이언트에만 두면 요청을 직접 만들어 우회할 수 있다.

---

## 6. Cookie / Session

### 구조

```text
로그인
Client ── ID/PW ──> Server
Client <─ Set-Cookie: sessionId=abc; HttpOnly; Secure ─ Server

이후 요청
Client ── Cookie: sessionId=abc ──> Server ──> 세션 저장소에서 사용자 조회
```

### 특징

- Cookie는 브라우저가 도메인별로 저장하고 요청에 자동으로 붙일 수 있는 작은 데이터다. Session은 보통 서버가 사용자 상태를 저장하는 방식이다.
- Cookie에 세션 식별자만 저장하고 실제 사용자 정보는 서버의 세션 저장소에 둔다.
- `HttpOnly`는 JavaScript 접근 제한, `Secure`는 HTTPS에서만 전송, `SameSite`는 교차 사이트 전송 범위를 제어한다.
- 서버가 여러 대라면 세션을 공유 저장소에 두거나 sticky session을 써야 한다. 로그아웃·강제 만료 처리가 비교적 직접적이다.

### 정리

- Cookie와 Session은 경쟁 개념이 아니다. 일반적으로 Cookie는 세션 ID를 전달하고, Session은 서버에 로그인 상태를 보관한다.
- 세션 ID는 비밀번호처럼 보호해야 하며 HTTPS와 안전한 Cookie 옵션이 기본이다.

---

## 7. JWT (JSON Web Token)

### 구조

```text
header.payload.signature
eyJ... . eyJzdWIiOiIxMjMiLCJleHAiOj... . SflKx...

Server: 서명 검증 → 만료 시간과 issuer/audience 등 claims 검증 → 요청 처리
```

### 특징

- JSON 형태의 claims에 서명을 붙인 토큰으로, 서버는 서명을 검증해 위변조 여부를 확인한다.
- payload는 암호화되지 않은 경우 누구나 decode할 수 있으므로 비밀번호나 민감한 개인정보를 넣으면 안 된다.
- 토큰만 검증하면 되는 구조라 서버가 세션을 반드시 저장할 필요는 없지만, 탈취 토큰의 즉시 폐기나 권한 변경 반영은 별도 설계가 필요하다.
- 서버는 서명뿐 아니라 만료 시간(`exp`), 발급자(`iss`), 대상(`aud`), 허용한 알고리즘까지 검증해야 한다.

### 정리

- JWT는 "로그인 정보가 담긴 암호문"이 아니라 "서명으로 위변조를 막은 전달용 claims"다.
- 토큰 저장 위치는 XSS와 CSRF 위험을 함께 보고 결정해야 한다.

---

## 8. OAuth 2.0

### 구조

```text
사용자 → 우리 서비스 → 인증 제공자(Google 등)
                         │ 사용자 로그인·동의
우리 서비스 ← authorization code ───────┘
우리 서비스 ── code 교환 ──> 인증 제공자 → access token
```

### 특징

- OAuth 2.0은 사용자의 비밀번호를 서비스에 알려주지 않고, 다른 서비스의 자원 접근 권한을 위임받는 프로토콜이다.
- 로그인에 쓰이는 경우가 많지만 OAuth 자체는 인증 프로토콜이 아니다. 신원 확인에는 일반적으로 OpenID Connect의 ID Token을 함께 사용한다.
- 웹·SPA에서는 Authorization Code Flow와 PKCE를 사용한다. redirect URI, `state`, PKCE 검증값을 정확히 검증해야 한다.

### 정리

- "Google 로그인"은 보통 OAuth 2.0에 OpenID Connect를 조합한 흐름이다.
- redirect URI를 느슨하게 허용하면 인증 코드나 토큰이 공격자에게 전달될 수 있다.

---

# Frontend 심화

## 9. Preflight

### 구조

```text
Browser ── OPTIONS /orders ──> Server
        <─ Access-Control-Allow-Methods: POST
        <─ Access-Control-Allow-Headers: Authorization, Content-Type

Browser ── POST /orders ─────> Server
```

### 특징

- `POST` + JSON, `Authorization` 헤더처럼 단순 요청 조건을 벗어나면 브라우저가 실제 요청 전에 OPTIONS 요청으로 허용 여부를 묻는다.
- 서버는 허용할 method, header, origin을 응답해야 하며, 허용되지 않으면 브라우저는 실제 요청을 보내지 않는다.
- `Access-Control-Max-Age`로 preflight 결과를 잠시 캐시할 수 있지만, 보안을 위해 지나치게 길게 잡지 않는다.

### 정리

- Preflight는 프론트엔드 버그가 아니라 브라우저가 교차 출처의 위험한 요청을 보내기 전 서버 정책을 확인하는 과정이다.
- API에서 Authorization 헤더를 사용하면 OPTIONS 요청 처리도 함께 준비해야 한다.

---

## 10. Cookie 옵션과 Token 저장 방식

### 구조

```text
HttpOnly Cookie                 JavaScript 메모리
브라우저가 자동 전송            Authorization: Bearer <token>으로 직접 전송
XSS로 값 읽기 어려움             새로고침하면 사라짐
CSRF 방어 필요                  XSS로 탈취될 수 있음
```

### 특징

- 토큰을 `localStorage`에 두면 구현은 간단하지만 XSS가 발생했을 때 JavaScript로 바로 읽힐 수 있다.
- HttpOnly Cookie는 토큰 값을 JavaScript에서 읽지 못하게 하지만, 자동 전송되므로 CSRF 방어가 필요하다.
- 민감한 access token은 가능한 짧게 유지하고, 장기 자격 증명은 HttpOnly·Secure Cookie와 회전 정책 등을 포함해 설계한다.
- 어떤 저장 방식도 단독으로 안전하지 않다. XSS 방어, HTTPS, 짧은 만료, 서버 검증이 함께 필요하다.

### 정리

- 저장 위치 선택은 "Cookie냐 localStorage냐"의 단순 문제가 아니라 XSS와 CSRF 중 어떤 위험을 어떻게 줄일지의 설계다.
- 장기 토큰을 localStorage에 두는 방식은 피하고, 서버의 보안 정책과 함께 결정한다.

---

## 11. OAuth Redirect 처리

### 구조

```text
1. 앱이 state와 PKCE verifier를 생성·보관
2. 인증 제공자로 이동
3. /callback?code=...&state=... 으로 복귀
4. state 비교 성공 → code를 token으로 교환
5. URL에서 code, state 제거
```

### 특징

- callback에서 받은 `state`는 처음 생성한 값과 비교해 요청 위조를 막는다.
- SPA에서는 PKCE verifier를 안전하게 보관하고, authorization code를 access token으로 교환할 때 함께 보낸다.
- 콜백 URL의 query string에 code나 state가 남아 있으면 브라우저 기록·로그·Referer에 남을 여지가 있어 처리 후 `history.replaceState` 등으로 URL을 정리한다.
- 토큰 교환이나 refresh token 보관을 백엔드에서 처리하면 브라우저에 민감한 토큰을 덜 노출할 수 있다.

### 정리

- OAuth callback은 단순히 `code`를 받아 로그인 완료하는 화면이 아니라, state와 PKCE를 검증하는 보안 경계다.
- redirect URI는 제공자 콘솔과 애플리케이션 모두에서 정확한 주소로 제한한다.

---

## 12. 딥다이브: Cookie 기반 인증에서 XSS·CSRF 함께 다루기

### 왜 Cookie만 HttpOnly로 설정하면 끝나지 않을까?

`HttpOnly`는 JavaScript가 Cookie 값을 읽는 일을 막는다. 그래서 XSS가 생겨도 공격자가 세션 ID나 refresh token을 바로 훔치기는 어렵다. 하지만 XSS로 실행된 스크립트는 같은 출처의 API를 호출할 수 있고, 브라우저는 요청에 Cookie를 자동으로 붙인다. 또 다른 사이트에서 보낸 요청도 Cookie 설정에 따라 포함될 수 있으므로 CSRF 방어가 필요하다.

```text
XSS 발생
공격 스크립트 ── fetch("/api/withdraw", { method: "POST" }) ──> 서버
                    └─ Cookie 값은 못 읽어도 브라우저가 자동 첨부

CSRF 공격
evil.example ── POST https://app.example/api/withdraw ──> 서버
                 └─ Cookie가 자동 첨부될 수 있음
```

### Cookie 옵션의 역할 분담

| 옵션 | 하는 일 | 한계 |
| --- | --- | --- |
| `HttpOnly` | JavaScript의 Cookie 값 읽기 차단 | XSS가 API 요청을 보내는 것은 막지 못함 |
| `Secure` | HTTPS 연결에서만 Cookie 전송 | HTTPS 자체를 대체하지 않음 |
| `SameSite=Lax` | 대부분의 교차 사이트 POST 요청에서 Cookie 미전송 | 같은 사이트 내 공격·일부 로그인 흐름은 별도 검토 필요 |
| `SameSite=Strict` | 교차 사이트 요청에서 더 강하게 Cookie 제한 | 외부 링크로 들어온 첫 요청에도 Cookie가 빠져 UX가 깨질 수 있음 |
| `SameSite=None; Secure` | 다른 사이트에서도 Cookie 전송 허용 | CSRF 토큰·Origin 검증이 반드시 필요 |

`SameSite`의 site는 origin보다 넓은 개념이다. 예를 들어 `app.example.com`과 `api.example.com`은 서로 다른 origin이지만 같은 site이므로, 서브도메인으로 프론트와 API를 나눈 경우 CORS와 SameSite를 따로 생각해야 한다.

### CSRF 토큰을 함께 보내는 흐름

서버가 로그인 응답에서 세션 Cookie와 CSRF 토큰을 발급하고, 프론트엔드는 상태 변경 요청에 토큰을 별도 헤더로 보낸다. 공격 사이트는 SOP 때문에 정상 사이트의 CSRF 토큰을 읽어 커스텀 헤더에 넣을 수 없으므로 서버가 위조 요청을 구분할 수 있다.

```text
1. Server → Set-Cookie: session=...; HttpOnly; Secure; SameSite=Lax
2. Server → CSRF token을 응답 본문 또는 읽을 수 있는 별도 Cookie로 전달
3. Client → X-CSRF-Token: <token> 헤더와 함께 POST/PUT/PATCH/DELETE 요청
4. Server → 세션의 토큰(또는 서명된 토큰)과 헤더 값을 비교한 뒤 처리
```

```js
async function request(url, options = {}) {
  const csrfToken = document.cookie
    .split("; ")
    .find((cookie) => cookie.startsWith("csrfToken="))
    ?.split("=")[1];

  return fetch(url, {
    ...options,
    credentials: "include",
    headers: {
      ...options.headers,
      "X-CSRF-Token": csrfToken,
    },
  });
}
```

- 이 예시는 double-submit Cookie 방식의 흐름을 보여준다. 서버는 Cookie에만 토큰이 있는지 확인해서는 안 되고, 요청 헤더 값과도 비교해야 한다.
- 모든 GET 요청에 CSRF 토큰을 붙일 필요는 없지만, 서버는 GET이 상태를 변경하지 않도록 설계해야 한다.
- Cookie를 쓰는 교차 출처 API라면 `credentials: "include"`와 함께 서버의 정확한 CORS origin 허용, `Access-Control-Allow-Credentials: true`가 필요하다.

### Access Token 갱신을 한 번만 실행하기

access token을 메모리에만 두고 refresh token을 HttpOnly Cookie에 둔다면, access token 만료 시 여러 API 요청이 동시에 401을 받을 수 있다. 요청마다 refresh를 보내면 refresh token 회전 시 충돌할 수 있으므로, 진행 중인 갱신 Promise를 공유해 한 번만 갱신한다.

```js
let accessToken;
let refreshPromise;

async function refreshAccessToken() {
  if (!refreshPromise) {
    refreshPromise = fetch("/api/auth/refresh", {
      method: "POST",
      credentials: "include",
    })
      .then((response) => {
        if (!response.ok) throw new Error("로그인이 만료되었습니다.");
        return response.json();
      })
      .then(({ accessToken: nextToken }) => {
        accessToken = nextToken;
        return nextToken;
      })
      .finally(() => {
        refreshPromise = undefined;
      });
  }

  return refreshPromise;
}
```

- refresh 요청도 Cookie 인증을 사용한다면 CSRF 보호 대상이다. `SameSite` 설정과 CSRF 토큰 또는 Origin 검증을 똑같이 적용한다.
- refresh가 실패하면 저장된 클라이언트 상태를 지우고 로그인 화면으로 보낸다. 만료된 access token으로 무한 재시도하면 안 된다.
- 이 방식은 토큰을 새로고침 뒤에 다시 발급받아야 하므로, 앱 초기 진입 시 인증 상태를 복구하는 로딩 상태를 별도로 다뤄야 한다.

### 정리

- Cookie 인증은 `HttpOnly` 하나만으로 끝나지 않는다. XSS는 입력·출력 처리와 CSP로 줄이고, CSRF는 SameSite·CSRF 토큰·Origin 검증으로 방어한다.
- access token을 메모리에, refresh token을 HttpOnly Cookie에 두는 방식은 XSS로 장기 토큰이 노출되는 위험을 줄일 수 있지만, 갱신·초기 인증 복구·실패 처리를 프론트엔드가 책임져야 한다.
- 어떤 저장 방식을 택해도 서버의 인증·인가 검증이 최종 보안 경계다.
