# Web Security / 인증: 요청을 어디까지 신뢰할 수 있을까?

## 1. SOP: 다른 사이트에서 내 정보를 읽을 수 있을까?

SOP(Same-Origin Policy, 동일 출처 정책)는 브라우저가 서로 다른 출처의 문서나 스크립트 사이 상호작용을 제한하는 정책이다. Origin은 **scheme, host, port의 조합**으로 구분한다.

`https://shop.example.com`을 기준으로 비교하면 다음과 같다.

| URL | 같은 출처인가? | 이유 |
| --- | --- | --- |
| `https://shop.example.com/orders` | O | 경로만 다름 |
| `http://shop.example.com` | X | scheme이 다름 |
| `https://api.example.com` | X | host가 다름 |
| `https://shop.example.com:8443` | X | port가 다름 |

사용자가 은행 사이트에 로그인한 상태에서 악성 사이트를 열었다고 하자. 악성 사이트의 스크립트가 은행의 계좌 정보를 마음대로 읽을 수 있다면, 은행의 로그인 검사가 있어도 정보가 유출된다. SOP는 이런 출처 사이의 접근을 제한한다.

다만 **다른 출처로 보내는 모든 요청을 막는 정책은 아니다.** 링크 이동, 폼 제출, 이미지 삽입처럼 허용되는 동작이 있다. 요청 전송과 응답 내용을 스크립트로 읽는 것은 구분해야 한다. [MDN의 SOP 설명](https://developer.mozilla.org/en-US/docs/Web/Security/Defenses/Same-origin_policy)을 참고했다.

---

## 2. CORS: 필요한 출처에는 어떻게 응답을 공유할까?

CORS(Cross-Origin Resource Sharing)는 서버가 허용한 출처에 브라우저가 응답을 공유하도록 하는 HTTP 헤더 기반 체계다. 프런트엔드와 API의 출처가 다를 때 자주 만나게 된다.

```http
Origin: https://shop.example.com
```

서버는 요청 출처를 확인하고 허용된 출처라면 응답에 다음과 같이 표시할 수 있다.

```http
Access-Control-Allow-Origin: https://shop.example.com
Vary: Origin
```

허용 목록에 따라 헤더를 바꾼다면 `Vary: Origin`으로 캐시가 출처별 응답을 구분하도록 한다. 전달받은 Origin을 검증 없이 그대로 반영해서는 안 된다.

### Preflight는 언제 발생할까?

예를 들어 다른 출처로 `Content-Type: application/json`인 POST를 보내거나 `Authorization` 헤더를 사용하면, 브라우저는 일반적으로 OPTIONS 사전 요청으로 허용 여부를 먼저 확인한다. 캐시된 사전 허가가 있으면 새 사전 요청은 생략될 수 있다.

```http
OPTIONS /orders HTTP/1.1
Host: api.example.com
Origin: https://shop.example.com
Access-Control-Request-Method: POST
Access-Control-Request-Headers: content-type
```

```http
HTTP/1.1 204 No Content
Access-Control-Allow-Origin: https://shop.example.com
Access-Control-Allow-Methods: POST
Access-Control-Allow-Headers: Content-Type
Vary: Origin
```

실제 POST 응답에도 필요한 CORS 헤더가 있어야 한다. 사전 요청 없이 전송되는 요청도 있으므로 CORS 오류가 보인다고 서버의 상태 변경까지 없었다고 단정할 수 없다.

쿠키를 포함하는 cross-origin fetch는 클라이언트의 `credentials: 'include'`와 서버의 `Access-Control-Allow-Credentials: true` 등이 필요하다. 이때 허용 출처에 `*`를 사용할 수 없으며, 쿠키의 SameSite와 브라우저의 제3자 쿠키 정책도 별도로 적용된다.

CORS는 브라우저의 응답 공유 정책이다. 서버 간 호출을 차단하거나 사용자의 권한을 증명하지 않는다. [MDN의 CORS 설명](https://developer.mozilla.org/en-US/docs/Web/HTTP/Guides/CORS)을 참고했다.

---

## 3. XSS: 사용자 입력이 코드로 실행된다면?

XSS(Cross-Site Scripting)는 신뢰할 수 없는 입력이 페이지에서 실행 가능한 콘텐츠로 해석되는 취약점이다. 공격자 입력이 서비스의 출처에서 실행되면 사용자가 그 서비스에서 할 수 있는 동작을 악용할 수 있다.

| 구분 | 예시 |
| --- | --- |
| Stored XSS | 게시글에 저장한 악성 입력이 다른 사용자의 화면에서 실행됨 |
| Reflected XSS | 요청의 검색어 등이 응답에 삽입되어 실행됨 |
| DOM-based XSS | 브라우저 코드가 URL 등의 값을 위험한 DOM API에 넣음 |

Stored/Reflected는 입력이 전달되는 방식이고 DOM-based는 클라이언트의 처리에 초점을 둔 구분이라 서로 겹칠 수 있다.

```javascript
// 문자열을 HTML로 해석하는 위치에 넣으면 위험할 수 있다.
messageElement.innerHTML = userInput;

// 텍스트를 표시하려는 목적이라면 텍스트로 처리한다.
messageElement.textContent = userInput;
```

HTML 본문, 속성, URL, JavaScript는 해석 규칙이 달라 **출력 위치에 맞는 인코딩과 안전한 API**를 사용해야 한다. HTML 자체를 허용해야 한다면 검증된 sanitizer로 허용 범위를 제한한다. 프레임워크의 자동 이스케이프도 HTML 직접 삽입 기능까지 보호하는 것은 아니다.

CSP(Content Security Policy)는 추가 방어 수단이며 입력 처리의 대체재는 아니다. HttpOnly 쿠키도 스크립트의 쿠키 읽기를 제한할 뿐, XSS 코드가 사용자 권한으로 요청을 보내는 것까지 막지는 못한다. [OWASP의 XSS 예방 지침](https://cheatsheetseries.owasp.org/cheatsheets/Cross_Site_Scripting_Prevention_Cheat_Sheet.html)을 참고했다.

---

## 4. CSRF: 로그인한 사용자의 브라우저를 대신 움직인다면?

CSRF(Cross-Site Request Forgery)는 공격자가 사용자의 브라우저를 통해 원하지 않는 요청을 보내게 하는 공격이다. 서버가 브라우저에서 자동 첨부되는 쿠키만 보고 사용자의 의도까지 신뢰하면 문제가 된다.

```text
사용자가 서비스에 로그인 → 브라우저에 인증 쿠키 저장
    ↓
악성 페이지 방문 → 서비스로 상태 변경 요청 유도
    ↓
쿠키 전송 조건이 맞으면 브라우저가 쿠키 첨부
    ↓
서버가 요청 의도 검증 없이 처리하면 피해 발생
```

SOP 때문에 공격자가 응답을 읽지 못하더라도 요청 자체는 성공할 수 있다. 따라서 CORS를 설정했다는 이유만으로 CSRF 방어가 끝나지는 않는다.

상태를 변경하는 요청에는 세션에 연결된 CSRF 토큰 검증, Origin 검증 등의 방어를 적용한다. 쿠키 기반의 double-submit 방식을 쓴다면 단순 값 비교에 그치지 않고 세션에 연결한 서명 등으로 쿠키 주입 위험을 고려해야 한다. SameSite도 방어에 도움이 되지만 단독으로 의존하지 않는다. GET이 상태를 바꾸게 만드는 설계도 피해야 한다.

| 공격 | 악용하는 신뢰 | 주요 대응 |
| --- | --- | --- |
| XSS | 서비스 화면의 콘텐츠를 실행해도 된다는 신뢰 | 문맥별 인코딩, 안전한 DOM API, HTML 정제 |
| CSRF | 인증 쿠키가 붙은 요청은 사용자가 의도했다는 신뢰 | CSRF 토큰, 출처 검증, SameSite |

JWT도 쿠키에 넣어 자동 전송하면 CSRF 검토가 필요하다. 토큰 형식보다 **인증 정보가 요청에 붙는 방식**이 중요하다. [OWASP의 CSRF 예방 지침](https://cheatsheetseries.owasp.org/cheatsheets/Cross-Site_Request_Forgery_Prevention_Cheat_Sheet.html)을 참고했다.

---

## 5. 인증 / 인가: 누구인지 확인하면 모든 요청이 가능할까?

인증(Authentication)은 요청 주체가 누구인지 확인하는 과정이다. 인가(Authorization)는 그 주체가 특정 작업을 수행할 권한이 있는지 판단하는 과정이다.

```text
GET /orders/101
    ↓
인증: 유효한 세션이나 토큰인가? 누구의 요청인가?
    ↓
인가: 이 사용자가 주문 101을 조회할 수 있는가?
    ↓
조회 결과 반환
```

로그인한 사용자라는 사실만으로 다른 사람의 주문을 조회할 수 있어서는 안 된다. URL의 ID는 사용자가 바꿀 수 있으므로 서버가 대상 데이터의 소유권이나 허용된 역할을 확인해야 한다.

화면에서 관리자 버튼을 숨기는 것은 사용자 경험을 위한 처리다. 최종 권한 검사는 서버에서 수행한다. 인증 정보가 부족한 경우에는 일반적으로 401, 인증된 요청을 권한 등의 이유로 거부할 때는 403을 사용한다. 리소스 존재를 숨기기 위해 404를 사용하는 정책도 있을 수 있다. [HTTP 상태 코드 명세](https://www.rfc-editor.org/rfc/rfc9110.html#name-status-codes)를 참고했다.

---

## 6. Cookie / Session: 로그인 상태는 어디에 있을까?

쿠키는 브라우저에 저장하고 조건에 맞는 요청에 전송하는 데이터다. 세션은 여러 요청을 같은 사용자 상태로 연결하는 방식이다. 흔한 서버 세션 방식에서는 쿠키에 세션 식별자를 넣고, 실제 인증 상태는 서버 저장소에서 관리한다.

```text
로그인 정보 확인
    ↓
서버: 새 세션 식별자 발급, 인증 상태 저장
    ↓ Set-Cookie
브라우저: 세션 식별자 보관
    ↓ Cookie
서버: 세션 조회 → 인증 및 권한 검사
```

```http
Set-Cookie: __Host-session=opaque-session-id; Path=/; Secure; HttpOnly; SameSite=Lax
```

위 값은 설명용 식별자다. 실제 세션 식별자는 충분히 예측하기 어렵게 생성한다.

| 옵션 | 역할 |
| --- | --- |
| Secure | HTTPS 연결에서 전송하도록 제한 |
| HttpOnly | JavaScript의 쿠키 직접 접근 제한 |
| SameSite | cross-site 요청에서의 전송 조건 제어 |
| Domain 생략 | 발급한 host에 한정하는 host-only 쿠키 |
| Path | 전송할 경로 범위. 보안 격리 경계로 사용하지 않음 |
| Max-Age / Expires | 브라우저 측 쿠키 수명 제어 |

`SameSite=Lax`는 일부 cross-site 최상위 안전한 탐색에서도 전송된다. `SameSite=None`에는 Secure가 필요하다. Site는 Origin과 다르므로 HTTPS의 `shop.example.com`과 `api.example.com`은 다른 출처이면서 같은 site일 수 있다. [쿠키 옵션](https://developer.mozilla.org/en-US/docs/Web/HTTP/Reference/Headers/Set-Cookie)을 참고했다.

로그인이나 권한 상승 후에는 세션 ID를 갱신해 세션 고정 공격을 막는다. 로그아웃은 쿠키 삭제뿐 아니라 서버 세션 무효화도 포함해야 한다. 유휴·절대 만료는 서버에서 강제하고, 서버가 여러 대면 세션 공유나 라우팅 정책을 정한다. [OWASP의 세션 관리 지침](https://cheatsheetseries.owasp.org/cheatsheets/Session_Management_Cheat_Sheet.html)을 참고했다.

---

## 7. JWT: 서명이 있으면 무엇을 믿을 수 있을까?

JWT(JSON Web Token)는 클레임을 전달하는 형식이다. 흔히 쓰는 서명된 JWS compact 표현은 세 부분으로 구성된다.

```text
Base64url(Header).Base64url(Payload).Signature
```

Payload에는 `sub`(주체), `iss`(발급자), `aud`(대상), `exp`(만료 시각) 등의 클레임이 들어갈 수 있다. 일반적인 서명 JWT의 내용은 디코딩해서 볼 수 있다. **서명은 암호화가 아니므로 비밀번호 같은 비밀을 넣어서는 안 된다.** 암호화된 JWE는 별도의 표현과 처리를 사용한다. [JWT 명세](https://www.rfc-editor.org/rfc/rfc7519.html)를 참고했다.

서명 검증은 허용된 키와 알고리즘으로 무결성과 발급 주체를 확인하는 과정이다. 만료·대상·용도 검증까지 해야 해당 API가 받아들일 토큰인지 판단할 수 있다. 단순 디코딩은 검증이 아니다.

| 방식 | 인증 상태 확인 | 운영에서 살필 점 |
| --- | --- | --- |
| 서버 세션 | 식별자로 서버 상태 조회 | 공유 저장소, 만료·무효화 |
| 자체 검증 JWT | 서명과 클레임 검증 | 키 교체, 만료 전 폐기, 권한 변경 반영 |

JWT로 인증해도 즉시 로그아웃이나 계정 정지를 위해 폐기 상태를 조회할 수 있다. JWT를 사용한다는 사실만으로 모든 요청 처리가 무상태가 되는 것은 아니다.

---

## 8. OAuth: 비밀번호를 넘기지 않고 권한을 맡기는 방법

OAuth 2.0은 다른 애플리케이션이 사용자의 자원에 제한된 접근 권한을 얻도록 하는 권한 위임 체계다.

| 역할 | 의미 |
| --- | --- |
| Resource Owner | 접근을 허용하는 사용자 등 |
| Client | 자원 접근 권한을 요청하는 애플리케이션 |
| Authorization Server | 사용자를 확인하고 동의·정책에 따라 토큰 발급 |
| Resource Server | Access Token을 확인하고 자원 제공 |

OAuth의 Access Token은 자원 접근용이다. 로그인 표준으로 사용하려면 인증 계층인 OIDC(OpenID Connect)를 구분해야 한다. OIDC의 ID Token은 클라이언트에 인증 결과를 전달하며 API 접근용 Access Token과 역할이 다르다. [OpenID Connect Core](https://openid.net/specs/openid-connect-core-1_0.html)를 참고했다.

```text
클라이언트 → 인가 서버: 로그인·동의 화면으로 이동
사용자 → 인가 서버: 인증 및 필요한 동의
인가 서버 → 등록된 콜백: Authorization Code 전달
클라이언트 → 토큰 엔드포인트: Code + PKCE verifier로 교환
인가 서버 → 클라이언트: Access Token 등 발급
클라이언트 → Resource Server: Access Token으로 요청
```

PKCE는 인가 요청 때 보낸 challenge와 코드 교환 시의 verifier를 연결해 코드 탈취 악용을 줄인다. verifier는 요청마다 안전하게 생성하고 `S256` 방식을 사용한다. 서버 클라이언트의 인증을 대체하는 기능은 아니다. [PKCE 명세](https://www.rfc-editor.org/rfc/rfc7636.html)를 참고했다.

---

## Backend 심화

## 9. Spring Security: 컨트롤러에 도착하기 전에 무엇을 할까?

Servlet 애플리케이션에서 Spring Security는 필터 체인을 통해 요청을 처리한다.

```text
HTTP 요청
    ↓
DelegatingFilterProxy
    ↓
FilterChainProxy → 요청에 맞는 SecurityFilterChain 선택
    ↓
보안 필터: 컨텍스트 처리, 인증, CSRF, 인가 등
    ↓
DispatcherServlet → Controller
```

여러 SecurityFilterChain이 있으면 **처음 매칭된 체인**을 사용한다. API용 체인과 웹 로그인용 체인을 나눌 때 순서와 매칭 범위를 확인해야 한다.

인증 결과는 SecurityContext의 Authentication으로 표현하고, 인가는 권한과 요청 대상에 따라 판단한다. URL 규칙만으로 주문 소유권 같은 업무 권한까지 해결되지 않으므로 서비스·메서드 수준 검사도 필요하다. [Spring Security 아키텍처](https://docs.spring.io/spring-security/reference/servlet/architecture.html)를 참고했다.

CORS 사전 요청에는 일반적으로 세션 쿠키가 없으므로 인증 실패로 먼저 차단하지 않게 처리 순서를 맞춘다. [Spring Security의 CORS 통합](https://docs.spring.io/spring-security/reference/servlet/integrations/cors.html)을 참고했다. `permitAll`은 인가 규칙이며 모든 보안 필터를 건너뛴다는 뜻은 아니다. 세션을 쓰지 않아도 인증 쿠키 등을 자동 전송한다면 CSRF를 검토해야 한다. JWT 사용 여부만으로 CSRF 설정을 끄지 않는다.

---

## 10. JWT 검증: 디코더에 맡길 것과 정책으로 정할 것

Resource Server는 토큰을 파싱한 뒤 신뢰 정책에 따라 검증해야 한다.

1. 서버가 허용한 알고리즘과 신뢰한 발급자의 키로 서명을 검증한다.
2. 기대한 `iss`, 이 API를 위한 `aud`인지 확인한다.
3. 서비스가 요구하는 `exp`, `nbf` 등 시간 조건을 검증한다.
4. 토큰의 종류와 용도를 구분하고 필요한 권한·소유권을 검사한다.

토큰 헤더의 알고리즘이나 키 URL을 그대로 신뢰해서는 안 된다. `kid`는 신뢰한 키 집합에서 키를 선택하는 단서이며, 그 자체가 신뢰 증명은 아니다. 허용 알고리즘을 고정하고 서로 다른 용도의 JWT가 혼용되지 않도록 한다. [JWT 보안 권고](https://www.rfc-editor.org/rfc/rfc8725.html)를 참고했다.

Spring Security의 Resource Server 지원은 JwtDecoder를 통해 서명과 클레임을 검증한다. issuer 기반 구성은 발급자와 시간 검증을 지원하지만, audience와 서비스 고유 규칙은 사용 버전의 설정이나 validator로 명시해야 한다. 라이브러리를 도입했다는 이유로 모든 업무 정책까지 자동 검증되는 것은 아니다. [Spring Security의 JWT 처리](https://docs.spring.io/spring-security/reference/servlet/oauth2/resource-server/jwt.html)를 참고했다.

키 교체 시에는 새 키 배포와 기존 토큰의 유효 기간, 키 캐시 갱신을 함께 고려한다. 서명 검증 성공은 현재 계정 상태나 해당 주문에 대한 권한까지 보증하지 않는다.

---

## 11. Refresh Token: 오래 로그인하면서 탈취 위험을 줄이려면?

Access Token의 수명을 짧게 두면 유출 시 사용 가능한 시간을 줄일 수 있지만 재인증이 잦아진다. Refresh Token은 클라이언트가 인가 서버에서 새 Access Token을 받는 데 사용한다. 일반 API에 보내는 토큰이 아니다.

```text
Access Token 만료
    ↓
클라이언트 → 인가 서버: Refresh Token으로 갱신
    ↓
인가 서버: 만료·폐기·클라이언트 연결 검증
    ↓
새 Access Token 발급, 정책에 따라 Refresh Token 회전
```

회전 방식은 갱신마다 새 Refresh Token을 발급하고 이전 값을 무효화한다. 이미 사용한 값이 다시 들어오면 탈취 가능성을 감지하고 관련 토큰 계열을 폐기할 수 있다. 정상 클라이언트의 동시 갱신도 재사용처럼 보일 수 있으므로 클라이언트의 갱신 요청을 조정하고 서버에서 원자적으로 상태를 변경해야 한다.

OAuth 보안 권고는 public client의 Refresh Token에 회전 또는 sender-constraining으로 재전송을 탐지하도록 요구한다. 로그아웃·비밀번호 변경·의심 이벤트에 대한 폐기 정책과 최대 수명도 필요하다. [Refresh Token 보호 권고](https://www.rfc-editor.org/rfc/rfc9700.html#section-4.14)를 참고했다.

Refresh Token을 폐기해도 이미 발급한 Access Token이 자동으로 사라지지는 않는다. 즉시 차단이 필요하면 자원 서버에서도 세션·폐기 상태 확인 등의 정책을 적용해야 한다.

---

## 12. OAuth 서버 처리: 콜백을 받았다고 로그인이 끝난 것은 아니다

백엔드가 OAuth 클라이언트 역할을 한다면 콜백 이후에도 검증이 남는다.

- 시작한 인증 흐름과 콜백을 연결한다. `state`를 사용한다면 예측하기 어렵게 생성하고 브라우저 세션에 연결해 일회성으로 검증한다.
- Redirect URI는 등록된 값과 엄격히 일치시키고 임의의 외부 URL로 연결하지 않는다.
- Authorization Code를 토큰 엔드포인트에서 교환하며 PKCE verifier와 필요한 클라이언트 인증을 전달한다.
- OIDC에서는 ID Token의 서명, 발급자, 대상, 만료와 사용한 `nonce`를 검증한다.
- 외부 사용자 식별 후 서비스 자체의 세션이나 토큰을 발급한다. OIDC 계정 연결은 검증된 `iss`와 `sub`를 기준으로 판단하고 이메일만으로 자동 병합하지 않는다.

`state`는 인증 흐름의 연결과 CSRF 방어에, OIDC의 `nonce`는 ID Token을 시작한 요청에 연결하고 재전송 위험을 줄이는 데 사용한다. PKCE도 조건을 충족하면 콜백 CSRF 방어에 기여하지만 각 값의 목적을 구분해 구현해야 한다. [OAuth 보안 권고](https://www.rfc-editor.org/rfc/rfc9700.html)와 [OIDC 검증 규칙](https://openid.net/specs/openid-connect-core-1_0.html)을 참고했다.

서비스 백엔드가 외부 인가 서버를 이용한다면 OAuth Client이고, 자체 API에서 토큰을 받아 자원을 제공하면 Resource Server다. 직접 인가·토큰 엔드포인트를 운영할 때 Authorization Server 역할까지 맡는다. 한 서비스가 여러 역할을 할 수 있지만 필요한 검증은 역할마다 다르다.

## 참고 자료

- [MDN — CORS](https://developer.mozilla.org/en-US/docs/Web/HTTP/Guides/CORS)
- [OWASP — CSRF Prevention](https://cheatsheetseries.owasp.org/cheatsheets/Cross-Site_Request_Forgery_Prevention_Cheat_Sheet.html)
- [RFC 8725 — JWT Best Current Practices](https://www.rfc-editor.org/rfc/rfc8725.html)
- [RFC 9700 — OAuth 2.0 Security Best Current Practice](https://www.rfc-editor.org/rfc/rfc9700.html)
- [Spring Security — OAuth 2.0 Resource Server JWT](https://docs.spring.io/spring-security/reference/servlet/oauth2/resource-server/jwt.html)
