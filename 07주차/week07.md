# 7주차 - Web Security / 인증

## 인증과 인가

### 인증

- 사용자가 누구인지 확인하는 과정
- 예: 아이디·비밀번호, 인증 코드, 생체 인증, 소셜 로그인

```text
아이디·비밀번호 입력
        ↓
서버가 사용자 확인
        ↓
로그인 성공
```

### 인가

- 인증된 사용자가 특정 작업을 할 권한이 있는지 확인하는 과정

```text
로그인한 사용자
      ↓
관리자 권한 확인
      ↓
관리자 페이지 접근 허용 또는 거부
```

| 구분 | 인증 | 인가 |
| --- | --- | --- |
| 확인 대상 | 사용자의 신원 | 사용자의 권한 |
| 질문 | 누구인가? | 무엇을 할 수 있는가? |
| 실패 예시 | 로그인 실패 | 관리자 페이지 접근 거부 |
| 상태 코드 | `401` | `403` |

### 핵심 정리

- 인증: 누구인가?
- 인가: 무엇을 할 수 있는가?
- 일반적으로 인증 후 인가 확인

## Cookie와 Session

- HTTP의 Stateless 특성을 보완해 사용자를 계속 식별

### Cookie

- 브라우저에 저장되는 작은 데이터
- 서버가 응답으로 설정하면 브라우저가 저장
- 조건에 맞는 다음 요청에 자동 포함
- 사용자 설정과 로그인 식별값 등에 사용
- 민감한 정보를 그대로 저장하면 안 됨

```http
Set-Cookie: sessionId=abc123
```

### Session

- 사용자 상태를 서버에 저장하는 방식

```text
1. 로그인 성공
2. 서버가 Session 생성
3. 브라우저에 Session ID를 Cookie로 전달
4. 브라우저가 요청마다 Session ID 전송
5. 서버가 Session ID로 사용자 확인
```

```text
브라우저                         서버
sessionId=abc123  ─────────→  abc123: 사용자 태경
```

| 구분 | Cookie | Session |
| --- | --- | --- |
| 저장 위치 | 브라우저 | 서버 |
| 저장 내용 | 작은 문자열 데이터 | 사용자 상태 |
| 요청 전송 | 조건에 맞으면 자동 전송 | Session ID로 조회 |
| 서버 부담 | 비교적 적음 | 사용자 수에 따라 증가 |

### 핵심 정리

- 브라우저에는 Session ID, 실제 로그인 상태는 서버에 저장
- Session ID가 탈취되면 공격자가 사용자 행세 가능

## Cookie 보안 옵션

```http
Set-Cookie: sessionId=abc123; HttpOnly; Secure; SameSite=Lax
```

### HttpOnly

- 자바스크립트의 `document.cookie`로 접근 불가
- XSS 공격으로 Session ID가 직접 탈취될 위험 감소
- 브라우저의 HTTP 요청에는 자동 포함

### Secure

- HTTPS 요청에서만 Cookie 전송
- 네트워크에서 Cookie가 노출될 위험 감소
- 로컬 개발 환경에는 브라우저별 예외가 있을 수 있음

### SameSite

- 다른 사이트에서 시작된 요청에 Cookie를 보낼지 결정
- `Strict`: 같은 사이트의 요청에만 전송
- `Lax`: 같은 사이트 요청과 일부 안전한 최상위 페이지 이동에 전송
- `None`: 다른 사이트 요청에도 전송하며 `Secure` 필수
- `Strict`와 `Lax`는 CSRF 위험 감소에 도움

### 그 외 옵션

- `Domain`: Cookie를 전송할 도메인 범위
- `Path`: Cookie를 전송할 경로 범위
- `Max-Age`: Cookie 유지 시간
- `Expires`: Cookie 만료 시각

### 핵심 정리

- `HttpOnly`: 자바스크립트 접근 제한
- `Secure`: HTTPS에서만 전송
- `SameSite`: 다른 사이트에서 시작된 요청의 전송 제한

## JWT와 토큰 저장 방식

- 사용자에 관한 정보인 Claim을 전달하는 토큰 형식
- 흔히 사용하는 서명된 JWT는 세 부분으로 구성

```text
Header.Payload.Signature
```

- Header: 토큰 종류와 서명 알고리즘
- Payload: 사용자 ID, 권한, 만료 시간 등의 Claim
- Signature: 토큰이 변조되지 않았는지 확인

### 주의

- 일반적인 Payload는 암호화가 아닌 Base64URL 인코딩
- 누구나 Payload 내용을 확인할 수 있음
- 비밀번호 같은 민감한 정보를 넣으면 안 됨
- 서명은 기밀성이 아니라 무결성과 발급자 확인을 제공

### 인증 흐름

```text
로그인 성공
    ↓
서버가 JWT 발급
    ↓
클라이언트가 요청에 JWT 포함
    ↓
서버가 서명과 만료 시간 검증
```

- 서버가 세션 상태를 저장하지 않고 검증 가능
- 발급한 토큰을 만료 전에 즉시 무효화하기 어려움

### 토큰 저장 방식

| 저장 위치 | 특징 | 주요 위험 |
| --- | --- | --- |
| 메모리 | 새로고침하면 사라짐 | 사용성 저하 |
| Local Storage | 오래 유지되고 사용이 간단 | XSS로 탈취 가능 |
| HttpOnly Cookie | 자바스크립트로 읽을 수 없음 | 자동 전송되므로 CSRF 대비 필요 |

- 저장 방식은 서비스 구조와 공격 위험을 고려해 선택
- Access Token은 짧게 사용
- 장기 인증은 더 엄격하게 보호한 Refresh Token으로 갱신 가능

### 핵심 정리

- JWT 서명은 Payload를 숨기지 않음
- Payload에 민감한 정보 저장 금지
- 토큰 저장 위치에 따라 XSS 또는 CSRF 위험 고려

## OAuth와 Redirect

- 비밀번호를 다른 서비스에 제공하지 않고 자원 접근 권한을 위임하는 방식
- OAuth는 인가를 위한 기술
- 소셜 로그인처럼 사용자 신원 확인이 필요하면 OpenID Connect를 함께 사용

### Authorization Code 흐름

```text
1. 사용자가 소셜 로그인 클릭
2. 인증 서버로 Redirect
3. 로그인과 권한 동의
4. 등록된 Redirect URI로 Authorization Code 전달
5. Code를 Access Token으로 교환
6. Access Token으로 자원 요청
```

- 토큰 대신 수명이 짧은 Code를 브라우저를 통해 전달

### PKCE

- 브라우저 앱은 비밀 키를 안전하게 숨길 수 없어 PKCE 사용
- 요청 시작 시 `code_verifier` 생성
- 변환한 `code_challenge`를 인증 서버에 전달
- Code 교환 시 원본 `code_verifier` 제출
- Code가 탈취돼도 공격자가 토큰으로 교환하기 어렵게 함
- 브라우저 앱은 Authorization Code와 PKCE 조합 사용 권장

### Redirect 보안

- `redirect_uri`: 미리 등록한 주소와 정확히 일치해야 함
- `state`: 요청을 시작할 때 만든 값과 응답 값을 비교해 요청 위조 방지
- 임의의 Redirect URI를 허용하면 Authorization Code가 공격자 주소로 전달될 수 있음

```text
정상: 인증 서버 → 우리 서비스 주소 → Code 전달
공격: 인증 서버 → 공격자 주소 → Code 탈취
```

### 핵심 정리

- OAuth: 권한 위임
- OpenID Connect: OAuth 기반 사용자 인증
- Redirect URI 검증과 PKCE로 Authorization Code 보호

## SOP

- Same-Origin Policy
- 한 Origin의 문서나 스크립트가 다른 Origin의 데이터에 접근하는 것을 제한하는 브라우저 보안 정책

### Origin 기준

- 프로토콜, 호스트, 포트가 모두 같아야 같은 Origin

```text
Origin = 프로토콜 + 호스트 + 포트
```

기준 주소: `https://example.com:443`

| 주소 | 같은 Origin? | 이유 |
| --- | --- | --- |
| `https://example.com/users` | O | 경로만 다름 |
| `http://example.com` | X | 프로토콜 다름 |
| `https://api.example.com` | X | 호스트 다름 |
| `https://example.com:8443` | X | 포트 다름 |

### 제한하는 이유

- 악성 사이트가 사용자가 로그인한 다른 서비스의 데이터를 읽는 것을 방지
- 다른 Origin의 응답 읽기 제한
- 다른 Origin 문서의 DOM 접근 제한
- 다른 Origin의 Local Storage 접근 제한

### 주의

- SOP가 모든 교차 출처 요청의 전송을 막는 것은 아님
- 링크 이동과 Form 전송은 가능한 경우가 있음
- `<img>`, `<script>` 등의 교차 출처 삽입도 가능한 경우가 있음
- 응답 내용을 자바스크립트가 읽는 것은 제한
- 서버가 다른 Origin의 응답 읽기를 허용하려면 CORS 사용

### 핵심 정리

- `localhost:3000`과 `localhost:8080`은 포트가 달라 다른 Origin
- SOP는 다른 Origin의 데이터를 함부로 읽지 못하게 하는 브라우저 정책

## CORS와 Preflight

- CORS는 서버가 응답 헤더로 다른 Origin의 브라우저 코드에 응답 접근을 허용하는 방식
- CORS 허용 여부는 프론트엔드가 아니라 서버에서 설정

```http
Access-Control-Allow-Origin: http://localhost:3000
```

### Preflight

- 실제 요청 전에 서버가 해당 요청을 허용하는지 확인하는 사전 요청
- 브라우저가 자동으로 `OPTIONS` Method 사용

```text
브라우저 ── OPTIONS ──→ 서버
브라우저 ←── 허용 정보 ── 서버
브라우저 ── 실제 요청 ─→ 서버
```

주요 발생 조건:

- `PUT`, `PATCH`, `DELETE` 등의 Method 사용
- `Authorization` 같은 안전 목록 밖의 Header 사용
- `Content-Type: application/json` 사용

### 주요 응답 Header

```http
Access-Control-Allow-Origin: http://localhost:3000
Access-Control-Allow-Methods: GET, POST, DELETE
Access-Control-Allow-Headers: Content-Type, Authorization
Access-Control-Allow-Credentials: true
```

- `Allow-Origin`: 허용할 Origin
- `Allow-Methods`: 허용할 Method
- `Allow-Headers`: 허용할 Header
- `Allow-Credentials`: Cookie 등 인증 정보 허용
- 인증 정보를 포함하면 `Allow-Origin: *`를 사용할 수 없음

### 핵심 정리

- CORS는 SOP의 제한을 서버가 선택적으로 완화하는 방식
- Preflight는 `OPTIONS` 요청으로 실제 요청의 허용 여부 확인
- CORS는 브라우저 정책이며 서버의 인증과 인가를 대신하지 않음

## XSS

- 공격자가 입력한 악성 스크립트가 정상 사이트의 코드처럼 사용자의 브라우저에서 실행되는 공격

공격에 성공하면:

- 페이지 내용 조작
- Local Storage 데이터 탈취
- 사용자 권한으로 요청 전송
- 입력 정보 가로채기

### 종류

- Stored XSS: 악성 입력이 DB에 저장된 뒤 여러 사용자에게 실행
- Reflected XSS: URL이나 요청의 입력이 응답에 바로 포함되어 실행
- DOM-based XSS: 프론트엔드 코드가 안전하지 않은 입력을 DOM에 넣어 실행

### 위험한 코드

```js
element.innerHTML = userInput;
```

- `innerHTML`은 문자열을 HTML로 해석
- 사용자 입력이 코드로 실행될 수 있음

```js
element.textContent = userInput;
```

- 단순 문자열은 `textContent`처럼 텍스트로 처리

### 방어 방법

- 출력 위치에 맞게 입력 이스케이프
- HTML이 꼭 필요하면 검증된 도구로 Sanitization
- `innerHTML`, `dangerouslySetInnerHTML` 사용 최소화
- 프레임워크의 기본 이스케이프 기능 우회 금지
- CSP로 실행 가능한 스크립트 제한
- Session Cookie에 `HttpOnly` 적용

### 핵심 정리

- 사용자 입력을 코드가 아닌 데이터로 처리
- `HttpOnly`와 CSP는 추가 방어이며 안전한 출력 처리를 대신하지 않음

## CSRF

- 사용자가 로그인한 상태를 악용해 공격자가 의도한 요청을 사용자의 브라우저에서 보내게 하는 공격

```text
1. 사용자가 은행 사이트에 로그인
2. Session Cookie가 브라우저에 저장
3. 사용자가 공격자 사이트 방문
4. 공격자 사이트가 송금 요청 유도
5. 브라우저가 은행 Cookie를 자동으로 포함
6. 서버가 정상 사용자의 요청으로 착각
```

- 공격자는 응답을 읽지 못해도 요청을 실행시키는 것만으로 목적 달성 가능

### SOP로 막지 못하는 이유

- SOP는 다른 Origin의 응답 읽기를 제한
- Form 전송 같은 일부 교차 출처 요청 자체는 허용
- 조건에 맞으면 브라우저가 Cookie를 자동으로 포함

### 방어 방법

- Cookie에 `SameSite=Lax` 또는 `Strict` 적용
- 상태 변경 요청에 예측 불가능한 CSRF Token 포함
- 서버에서 CSRF Token 검증
- `Origin`, `Referer`, Fetch Metadata Header 검증
- 상태를 변경하는 작업에 `GET` 사용 금지
- 중요한 작업은 비밀번호 재확인 등의 추가 인증 사용
- XSS가 있으면 CSRF 방어도 우회될 수 있으므로 함께 방어

### XSS와 비교

| XSS | CSRF |
| --- | --- |
| 악성 코드가 정상 사이트에서 실행 | 외부 사이트가 인증된 요청을 유도 |
| 사용자 입력의 안전하지 않은 출력 악용 | Cookie의 자동 전송 악용 |
| 이스케이프·Sanitization으로 방어 | SameSite·CSRF Token으로 방어 |

### 핵심 정리

- XSS: 악성 코드를 정상 사이트에서 실행
- CSRF: 사용자의 인증 정보를 이용해 요청 위조
- SOP는 요청 전송을 모두 차단하지 않으므로 CSRF를 완전히 방지하지 못함
