# OAuth

## OAuth 2.0

OAuth 2.0은 **사용자가 자신의 비밀번호를 넘겨주지 않고도, 제3자 애플리케이션에게 자기 리소스에 대한 제한된 접근 권한을 위임**하는 인가(Authorization) 프레임워크다. 표준은 [RFC 6749](https://www.rfc-editor.org/rfc/rfc6749)다.

### 등장 배경

어떤 서비스가 사용자의 Google 캘린더에 일정을 등록하고 싶다고 하자. OAuth 이전에는 사용자의 Google 아이디/비밀번호를 직접 받아야 했다.

- 서비스가 비밀번호를 저장해야 해서 유출 위험이 크다.
- 캘린더만 필요한데 메일, 드라이브 등 **모든 권한**을 얻게 된다.
- 권한을 회수하려면 사용자가 비밀번호를 바꾸는 수밖에 없다.

OAuth는 비밀번호 대신 **범위(scope)와 기한이 정해진 Access Token**을 발급해 이 문제를 해결한다. 사용자는 언제든 해당 앱의 권한을 철회할 수 있다.

### 역할 (Resource Owner / Client / Authorization Server / Resource Server)

| 역할 | 설명 | 예시 (우리 서비스에서 구글 로그인) |
| --- | --- | --- |
| Resource Owner | 리소스의 주인, 권한을 위임하는 사람 | 사용자 |
| Client | 리소스에 접근하려는 애플리케이션 | 우리 서비스 |
| Authorization Server | 사용자를 인증하고 토큰을 발급 | Google 인증 서버 |
| Resource Server | 보호된 리소스를 가진 서버, 토큰을 검증 | Google API 서버 (프로필, 캘린더 등) |

사전에 Client는 Authorization Server에 앱을 등록하고 다음 값을 받는다.

- `client_id`: 앱 식별자 (공개 가능)
- `client_secret`: 앱 비밀키 (서버에만 보관)
- `redirect_uri`: 인증 후 돌아올 주소 (등록된 주소로만 리다이렉트 허용)

### Authorization Code Grant 흐름

OAuth 2.0에는 여러 Grant Type이 있지만, 웹 서비스에서 가장 많이 쓰고 권장되는 방식은 **Authorization Code Grant**다.

```text
사용자        Client(우리 서비스)        Authorization Server        Resource Server
  │ 1. 구글 로그인 클릭 │                          │                          │
  │──────────────────▶│                          │                          │
  │ 2. 인가 페이지로 리다이렉트                     │                          │
  │◀──────────────────│                          │                          │
  │ 3. 구글 로그인 + 권한 동의                     │                          │
  │─────────────────────────────────────────────▶│                          │
  │ 4. redirect_uri?code=xxx 로 리다이렉트         │                          │
  │◀─────────────────────────────────────────────│                          │
  │──────────────────▶│ 5. code + client_secret으로 토큰 요청                │
  │                   │─────────────────────────▶│                          │
  │                   │ 6. Access Token 발급       │                          │
  │                   │◀─────────────────────────│                          │
  │                   │ 7. Access Token으로 리소스 요청                       │
  │                   │────────────────────────────────────────────────────▶│
```

1~2. 사용자를 Authorization Server로 보낸다.

```text
https://accounts.google.com/o/oauth2/v2/auth
  ?response_type=code
  &client_id=CLIENT_ID
  &redirect_uri=https://myapp.com/login/oauth2/code/google
  &scope=openid profile email
  &state=랜덤값
```

3~4. 사용자가 로그인하고 동의하면, `redirect_uri`로 **Authorization Code**를 붙여 돌려보낸다.

5~6. Client 서버가 이 code와 `client_secret`으로 **서버 간 통신(Back Channel)** 으로 토큰을 받는다.

7. 받은 Access Token으로 Resource Server에 사용자 정보를 요청한다.

왜 토큰을 바로 주지 않고 code를 거칠까?

- 4번의 리다이렉트는 브라우저 주소창을 거치므로(Front Channel) 히스토리, 로그, Referer 등으로 노출될 수 있다.
- code는 **한 번만 쓸 수 있고 수명이 매우 짧으며**, `client_secret` 없이는 토큰으로 바꿀 수 없다.
- 실제 토큰은 브라우저를 거치지 않는 서버 간 통신으로만 전달된다.

`state` 파라미터는 Client가 만든 랜덤값으로, 돌아올 때 같은 값인지 확인해 **OAuth 흐름에 대한 CSRF 공격**(공격자의 code로 피해자를 로그인시키는 것)을 막는다.

> 과거에 쓰던 Implicit Grant(토큰을 리다이렉트 URL로 바로 받는 방식)는 토큰이 노출되기 쉬워 현재는 권장되지 않는다. ([RFC 9700](https://www.rfc-editor.org/rfc/rfc9700))

### PKCE

PKCE(Proof Key for Code Exchange, "픽시")는 **Authorization Code를 탈취당해도 토큰으로 바꾸지 못하게** 하는 확장이다. ([RFC 7636](https://www.rfc-editor.org/rfc/rfc7636))

SPA나 모바일 앱처럼 `client_secret`을 안전하게 숨길 수 없는 클라이언트(Public Client)를 위해 나왔지만, 현재는 모든 클라이언트에 사용이 권장된다.

1. Client가 랜덤 문자열 `code_verifier`를 만든다.
2. `code_challenge = BASE64URL(SHA256(code_verifier))`를 계산해 인가 요청에 함께 보낸다. (`code_challenge_method=S256`)
3. Authorization Server는 `code_challenge`를 기억해 둔다.
4. 토큰 요청 시 Client가 원본 `code_verifier`를 보낸다.
5. 서버가 해시해서 처음 받은 `code_challenge`와 같은지 확인한다.

code를 가로챈 공격자는 원본 `code_verifier`를 모르기 때문에 토큰을 받을 수 없다. 해시는 역산이 불가능하므로 `code_challenge`가 노출돼도 괜찮다.

## OAuth와 OpenID Connect

OAuth 2.0은 **인가**를 위한 프레임워크다. Access Token은 "이 토큰을 가진 사람은 캘린더를 읽을 수 있다"는 권한을 의미할 뿐, "이 사람이 누구인지"를 표준화된 방식으로 알려주지 않는다.

그런데 실무에서는 "구글로 로그인" 같은 **인증** 용도로 OAuth를 많이 쓴다. 초기에는 Access Token으로 프로필 API를 호출해 사용자를 식별했는데, 서비스마다 방식이 제각각이었다.

**OpenID Connect(OIDC)** 는 OAuth 2.0 위에 인증 계층을 표준으로 얹은 것이다.

- scope에 `openid`를 포함하면 Access Token과 함께 **ID Token**을 발급한다.
- ID Token은 JWT 형식이며 `iss`, `sub`(사용자 고유 ID), `aud`(client_id), `exp`, `nonce` 등 사용자 인증 정보를 담는다.
- 사용자 정보 조회를 위한 `UserInfo` 엔드포인트와 설정 정보 조회용 Discovery 문서(`/.well-known/openid-configuration`)도 표준화되어 있다.

| | OAuth 2.0 | OpenID Connect |
| --- | --- | --- |
| 목적 | 인가 (리소스 접근 권한 위임) | 인증 (사용자 신원 확인) |
| 결과물 | Access Token | ID Token (+ Access Token) |
| 토큰을 보는 대상 | Resource Server | Client |

Google, Kakao 등은 OIDC를 지원하고, 지원하지 않는 Provider는 Access Token으로 사용자 정보 API를 호출하는 방식으로 로그인을 구현한다.
