# 7주차 참고 자료

## 입문

- [동일 출처 정책 (MDN)](https://developer.mozilla.org/ko/docs/Web/Security/Same-origin_policy)
  - Origin(스킴, 호스트, 포트)의 정의와 출처가 다른 리소스 접근을 제한하는 규칙
- [교차 출처 리소스 공유 CORS (MDN)](https://developer.mozilla.org/ko/docs/Web/HTTP/Guides/CORS)
  - 단순 요청, Preflight 요청, 자격 증명을 포함한 요청과 CORS 응답 헤더
- [CORS는 왜 이렇게 우리를 힘들게 하는걸까?](https://evan-moon.github.io/2020/05/21/about-cors/)
  - SOP와 CORS의 관계, 세 가지 CORS 시나리오와 자주 겪는 에러 해결법
- [HTTP 쿠키 (MDN)](https://developer.mozilla.org/ko/docs/Web/HTTP/Guides/Cookies)
  - 쿠키 생성과 수명, Secure/HttpOnly/SameSite 등 보안 속성
- [Introduction to JSON Web Tokens (jwt.io)](https://jwt.io/introduction)
  - JWT의 Header/Payload/Signature 구조와 토큰 기반 인증 흐름
- [WEB2 - OAuth 2.0 (생활코딩)](https://opentutorials.org/course/3405)
  - OAuth 2.0의 역할 구분과 Authorization Code 방식 흐름을 단계별로 설명

## 기본기

- [RFC 7519 - JSON Web Token](https://www.rfc-editor.org/rfc/rfc7519)
  - JWT 형식과 등록된 클레임(iss, sub, aud, exp 등)을 정의한 표준 문서
- [RFC 6749 - The OAuth 2.0 Authorization Framework](https://www.rfc-editor.org/rfc/rfc6749)
  - OAuth 2.0의 역할, Grant Type, 토큰 발급 흐름을 규정한 표준 문서
- [OAuth 2.0 Simplified](https://www.oauth.com/)
  - OAuth 2.0을 서버/클라이언트 구현 관점에서 풀어 쓴 무료 공개 도서

## XSS / CSRF

- [Cross Site Scripting Prevention Cheat Sheet (OWASP)](https://cheatsheetseries.owasp.org/cheatsheets/Cross_Site_Scripting_Prevention_Cheat_Sheet.html)
  - 출력 위치(HTML, 속성, JS, URL)별 인코딩 규칙과 CSP 등 XSS 방어 기법
- [Cross-Site Request Forgery Prevention Cheat Sheet (OWASP)](https://cheatsheetseries.owasp.org/cheatsheets/Cross-Site_Request_Forgery_Prevention_Cheat_Sheet.html)
  - Synchronizer Token, Double Submit Cookie, SameSite 쿠키 등 CSRF 방어 패턴
- [Cross-site scripting (PortSwigger Web Security Academy)](https://portswigger.net/web-security/cross-site-scripting)
  - Reflected/Stored/DOM-based XSS의 원리와 직접 풀어볼 수 있는 실습 랩
- [Cross-site request forgery (PortSwigger Web Security Academy)](https://portswigger.net/web-security/csrf)
  - CSRF 공격이 성립하는 조건과 CSRF 토큰, SameSite 우회 사례
- [SameSite cookies explained (web.dev)](https://web.dev/articles/samesite-cookies-explained)
  - SameSite=Strict/Lax/None의 차이와 브라우저 기본값이 Lax로 바뀐 배경

## OAuth / Token

- [RFC 7636 - PKCE](https://www.rfc-editor.org/rfc/rfc7636)
  - Authorization Code 탈취 공격을 막기 위한 code_verifier/code_challenge 확장
- [RFC 9700 - OAuth 2.0 Security Best Current Practice](https://www.rfc-editor.org/rfc/rfc9700)
  - Implicit Grant 비권장, PKCE 의무화, Refresh Token 보호 등 최신 보안 권고
- [Refresh Token Rotation (Auth0)](https://auth0.com/docs/secure/tokens/refresh-tokens/refresh-token-rotation)
  - Refresh Token을 사용할 때마다 새로 발급하고, 재사용을 감지하면 토큰 계열 전체를 폐기하는 방식

## Spring Security

- [Spring Security Architecture (공식 문서)](https://docs.spring.io/spring-security/reference/servlet/architecture.html)
  - DelegatingFilterProxy, FilterChainProxy, SecurityFilterChain으로 이어지는 필터 구조
- [Servlet Authentication Architecture (공식 문서)](https://docs.spring.io/spring-security/reference/servlet/authentication/architecture.html)
  - SecurityContextHolder, Authentication, AuthenticationManager, ProviderManager의 역할
- [OAuth 2.0 Resource Server JWT (공식 문서)](https://docs.spring.io/spring-security/reference/servlet/oauth2/resource-server/jwt.html)
  - JwtDecoder로 서명과 클레임을 검증하고 권한을 매핑하는 Resource Server 설정
- [OAuth 2.0 Login (공식 문서)](https://docs.spring.io/spring-security/reference/servlet/oauth2/login/core.html)
  - Google, GitHub 등 외부 Provider로 로그인하는 OAuth2 Client 설정과 동작
