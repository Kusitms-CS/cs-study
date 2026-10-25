# 7주차 - Web Security / 인증

## 공통 키워드

### SOP (Same-Origin Policy)
- 브라우저가 **다른 출처(origin)** 의 리소스 접근을 제한하는 기본 보안 정책
- origin = 프로토콜 + 도메인 + 포트 (셋 중 하나라도 다르면 다른 출처)
- 악성 사이트가 다른 사이트의 데이터를 몰래 읽는 걸 막기 위함

### CORS (Cross-Origin Resource Sharing)
- SOP 때문에 막히는 요청을 서버가 **허용해주는 방법**
- 서버가 `Access-Control-Allow-Origin` 같은 헤더로 허용할 출처를 알려줌
- Preflight: 복잡한 요청(PUT, DELETE, 커스텀 헤더 등) 전에 브라우저가 `OPTIONS`로 미리 확인
- 프론트(localhost:3000) ↔ 백엔드(localhost:8080) 연동할 때 자주 마주침

### XSS (Cross-Site Scripting)
- 공격자가 웹페이지에 **악성 스크립트를 심어** 다른 사용자 브라우저에서 실행시키는 공격
- 쿠키/토큰 탈취, 화면 조작 등이 가능
- 방어: 출력 시 이스케이프, 입력값 검증, CSP 적용, 쿠키에 `HttpOnly`

### CSRF (Cross-Site Request Forgery)
- 로그인된 사용자의 **권한을 이용해** 사용자가 원하지 않는 요청을 보내게 만드는 공격
- 브라우저가 쿠키를 자동으로 붙여 보내는 특성을 악용
- 방어: CSRF 토큰, `SameSite` 쿠키, Origin/Referer 검증

> XSS는 "사이트가 사용자를 믿는 것"을, CSRF는 "사이트가 브라우저를 믿는 것"을 악용한다고 보면 편함

### 인증 / 인가
- **인증(Authentication)**: 너 누구야? (로그인)
- **인가(Authorization)**: 너 이거 해도 돼? (권한 확인)
- 순서: 인증 → 인가

### Cookie / Session
- **Cookie**: 브라우저에 저장되는 작은 데이터, 요청마다 자동 전송
- **Session**: 서버가 로그인 상태를 저장하고, 클라이언트는 session ID만 쿠키로 들고 있음
- 장점: 서버가 상태를 통제 가능 (강제 로그아웃 쉬움)
- 단점: 서버 확장 시 세션 공유 필요 (Redis 등)
- 쿠키 옵션: `HttpOnly`, `Secure`, `SameSite`

### JWT (JSON Web Token)
- 사용자 정보를 담아 **서버가 서명한 토큰** (Header.Payload.Signature)
- 서버가 상태를 저장하지 않아도 됨 (Stateless)
- Payload는 암호화가 아니라 인코딩이라 **민감 정보 넣으면 안 됨**
- 단점: 발급 후 강제 만료가 어려움 → 만료시간을 짧게 + Refresh Token 사용

### OAuth
- 비밀번호 공유 없이 **다른 서비스(구글, 카카오 등)에 권한을 위임**하는 표준
- 주요 역할: Resource Owner(사용자), Client(우리 앱), Authorization Server, Resource Server
- 대표 흐름: Authorization Code Grant
  1. 사용자가 소셜 로그인 클릭
  2. 인증 서버에서 로그인/동의
  3. 우리 앱이 authorization code 수신
  4. code로 access token 교환
  5. access token으로 사용자 정보 조회

## Backend 심화 (선택)

### Spring Security
- 스프링 기반 인증/인가를 처리하는 보안 프레임워크
- 핵심은 **Filter Chain**: 요청이 컨트롤러에 도달하기 전에 여러 필터를 거침
- `SecurityFilterChain`으로 URL별 접근 권한 설정
- `SecurityContext`에 인증된 사용자 정보 저장
- 비밀번호는 `PasswordEncoder`(BCrypt)로 해시 저장

### JWT 검증
- 커스텀 필터에서 `Authorization: Bearer <token>` 헤더를 꺼내 검증
- 검증 항목: 서명 유효성, 만료시간(exp), 발급자(iss) 등
- 검증 성공 시 `SecurityContext`에 인증 정보 등록
- 실패 시 에러코드 401 응답

### Refresh Token
- Access Token은 짧게(예: 15 - 30분), Refresh Token은 길게(예: 2주) 운영
- Access Token 만료 시 Refresh Token으로 재발급
- Refresh Token은 서버(DB/Redis)에 저장해서 폐기 가능하게 관리
- 보안 강화: Refresh Token Rotation (재발급 때마다 새 Refresh Token 발급)

### OAuth 서버 처리
- 프론트가 받은 code를 백엔드로 넘기거나, 백엔드가 직접 redirect를 처리
- 백엔드가 code → access token 교환 → 사용자 정보 조회
- 우리 서비스 DB에 사용자 있으면 로그인, 없으면 회원가입 처리
- 마지막에 **우리 서비스의 JWT**를 발급해서 이후 인증은 JWT로 진행ㄴ