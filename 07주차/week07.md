# 7주차 - Web Security / 인증

## 1. 공통 키워드

### 1.1 SOP (Same-Origin Policy)

- 동일 출처 정책으로 브라우저에서 다른 출처의 리소스에 접근하는 것을 제한하는 보안 정책임
- 출처(Origin)는 프로토콜(Protocol), 호스트(Host), 포트(Port)의 조합으로 결정됨
- 세 가지 요소가 모두 동일해야 같은 출처로 판단함
- 악성 웹사이트가 사용자의 인증 정보를 이용해 다른 사이트의 데이터를 읽는 행위를 방지하기 위한 정책임
- SOP는 브라우저에서 적용되며 서버 간 HTTP 통신에는 적용되지 않음
- 다른 출처로 요청을 보내는 것 자체를 모두 차단하는 것은 아니며 주로 응답에 대한 접근을 제한함

**동일 출처 판단 예시**

| URL | 동일 출처 여부 | 이유 |
|---|---|---|
| https://example.com/api | O | 동일한 프로토콜, 호스트, 포트 |
| http://example.com/api | X | 프로토콜 다름 |
| https://api.example.com | X | 호스트 다름 |
| https://example.com:8080 | X | 포트 다름 |

### 1.2 CORS (Cross-Origin Resource Sharing)

- 교차 출처 리소스 공유 정책으로 SOP에 의해 제한된 다른 출처의 리소스 접근을 서버가 선택적으로 허용하는 메커니즘임
- 서버가 HTTP 응답 헤더를 통해 허용할 출처와 메서드 등을 지정함
- 브라우저가 서버의 CORS 응답 헤더를 확인하고 JavaScript의 응답 접근 허용 여부를 결정함
- CORS는 브라우저에서 적용되는 정책이며 Postman이나 서버 간 통신에서는 일반적으로 적용되지 않음

**주요 CORS 헤더**

| 헤더 | 설명 |
|---|---|
| Access-Control-Allow-Origin | 요청을 허용할 출처 |
| Access-Control-Allow-Methods | 허용할 HTTP 메서드 |
| Access-Control-Allow-Headers | 허용할 요청 헤더 |
| Access-Control-Allow-Credentials | 쿠키 등 인증 정보 포함 허용 여부 |
| Access-Control-Max-Age | Preflight 응답 캐싱 시간 |

**Preflight Request**

- 실제 요청 전에 브라우저가 OPTIONS 메서드로 서버의 허용 여부를 확인하는 사전 요청임
- 모든 CORS 요청에서 발생하는 것은 아니며 특정 메서드나 헤더 등을 사용하는 경우 발생함
- 서버가 허용 가능한 메서드와 헤더를 응답하면 브라우저가 실제 요청을 전송함

**Spring Boot CORS 설정 예시**

```java
@Configuration
public class CorsConfig {

    @Bean
    public WebMvcConfigurer corsConfigurer() {
        return new WebMvcConfigurer() {
            @Override
            public void addCorsMappings(CorsRegistry registry) {
                registry.addMapping("/api/**")
                        .allowedOrigins("https://example.com")
                        .allowedMethods("GET", "POST", "PUT", "DELETE")
                        .allowedHeaders("*")
                        .allowCredentials(true);
            }
        };
    }
}
```

- Spring Security를 사용하는 경우 SecurityFilterChain에서도 CORS 처리를 활성화해야 함

### 1.3 XSS (Cross-Site Scripting)

- 공격자가 웹페이지에 악성 JavaScript 코드를 삽입해 다른 사용자의 브라우저에서 실행되도록 하는 공격임
- 사용자 입력값을 검증하거나 적절히 인코딩하지 않고 HTML에 출력할 때 발생할 수 있음
- 세션 탈취, 사용자 정보 유출, 악성 요청 실행 등의 피해가 발생할 수 있음

**XSS 공격 유형**

| 유형 | 설명 |
|---|---|
| Stored XSS | 악성 스크립트를 DB 등에 저장한 후 다른 사용자가 페이지를 조회할 때 실행 |
| Reflected XSS | URL 파라미터 등의 입력값이 응답에 반영되면서 실행 |
| DOM-based XSS | 클라이언트 JavaScript가 DOM을 안전하지 않게 조작하면서 발생 |

**방어 방법**

- 사용자 입력값을 HTML에 출력할 때 문맥에 맞는 인코딩 적용
- HTML 삽입이 필요한 경우 신뢰할 수 있는 Sanitizer 사용
- CSP(Content Security Policy)를 설정해 스크립트 실행 제한
- 쿠키에 HttpOnly 속성을 설정해 JavaScript를 통한 쿠키 접근 차단
- 사용자 입력값을 innerHTML 등에 직접 삽입하지 않도록 주의

### 1.4 CSRF (Cross-Site Request Forgery)

- 사용자가 인증된 상태를 악용해 공격자가 의도하지 않은 요청을 보내도록 유도하는 공격임
- 브라우저가 쿠키를 자동으로 전송하는 특성을 이용할 수 있음
- 사용자의 권한으로 계좌 이체, 비밀번호 변경, 정보 수정 등의 요청이 실행될 수 있음
- XSS가 악성 스크립트 실행을 유도하는 공격이라면 CSRF는 인증된 사용자의 권한을 악용해 요청을 실행하는 공격임

**CSRF 공격 과정**

1. 사용자가 은행 사이트에 로그인해 세션 쿠키를 보유함
2. 공격자가 만든 악성 웹페이지에 사용자가 접근함
3. 악성 페이지에서 은행 서버로 송금 요청을 유도함
4. 브라우저가 조건에 따라 은행 세션 쿠키를 자동으로 첨부함
5. 서버가 요청의 정당성을 검증하지 않으면 사용자의 권한으로 송금이 실행될 수 있음

**방어 방법**

- CSRF Token을 발급하고 요청마다 검증
- 쿠키에 SameSite 속성을 설정해 교차 사이트 요청에서 쿠키 전송 제한
- Origin 또는 Referer 헤더 검증
- 중요 작업에 추가 인증 적용
- 상태를 변경하는 요청에 GET 메서드 사용 금지

### 1.5 인증 / 인가 (Authentication / Authorization)

**인증 (Authentication)**

- 사용자가 누구인지 신원을 확인하는 과정임
- 아이디와 비밀번호, OTP, 생체인증, 소셜 로그인 등이 대표적인 인증 방식임
- 인증에 성공하면 서버는 세션 또는 토큰 등을 통해 인증 상태를 관리함

**인가 (Authorization)**

- 인증된 사용자가 특정 리소스에 접근하거나 작업을 수행할 권한이 있는지 확인하는 과정임
- 사용자 역할(Role)이나 권한(Permission)을 기반으로 접근을 제어함
- 일반 사용자와 관리자의 접근 가능한 API를 구분하는 것이 대표적인 예시임

**인증과 인가 차이**

| 구분 | 인증 | 인가 |
|---|---|---|
| 목적 | 사용자 신원 확인 | 접근 권한 확인 |
| 질문 | 누구인가 | 무엇을 할 수 있는가 |
| 예시 | 로그인 | 관리자 페이지 접근 |
| HTTP 상태 코드 | 401 Unauthorized | 403 Forbidden |

- 401은 유효한 인증 정보가 없거나 인증에 실패한 경우 사용함
- 403은 서버가 요청을 이해했지만 접근을 허용하지 않는 경우 사용함

### 1.6 Cookie / Session

**Cookie**

- 브라우저에 저장되는 작은 데이터로 서버와 클라이언트 사이에서 상태를 유지하는 데 활용함
- HTTP의 Stateless 특성을 보완하기 위한 수단 중 하나임
- 서버가 Set-Cookie 응답 헤더를 통해 쿠키를 설정함
- 브라우저는 쿠키의 도메인, 경로, 보안 속성 등에 따라 요청에 쿠키를 자동으로 포함함

**주요 쿠키 속성**

| 속성 | 설명 |
|---|---|
| HttpOnly | JavaScript에서 쿠키 접근 차단 |
| Secure | HTTPS 요청에서만 쿠키 전송 |
| SameSite | 교차 사이트 요청에서 쿠키 전송 범위 제한 |
| Max-Age | 쿠키 유지 시간 |
| Expires | 쿠키 만료 시각 |
| Domain | 쿠키를 전송할 도메인 범위 |
| Path | 쿠키를 전송할 URL 경로 범위 |

**Session**

- 서버에서 사용자의 로그인 상태 등 필요한 정보를 관리하는 방식임
- 클라이언트에는 일반적으로 세션 ID만 쿠키로 저장함
- 서버는 전달받은 세션 ID를 통해 세션 저장소에서 사용자 정보를 조회함
- 서버 메모리, Redis, DB 등을 세션 저장소로 활용할 수 있음

**Session 인증 과정**

1. 사용자가 아이디와 비밀번호로 로그인 요청
2. 서버가 사용자 인증 후 세션 생성
3. 서버가 세션 ID를 쿠키로 전달
4. 브라우저가 이후 요청에 세션 쿠키 포함
5. 서버가 세션 ID를 조회해 사용자 인증 상태 확인

**Cookie와 Session 차이**

| 구분 | Cookie | Session |
|---|---|---|
| 저장 위치 | 클라이언트 | 서버 |
| 저장 데이터 | 쿠키 값 | 사용자 상태 정보 |
| 보안 | 탈취 및 변조 위험 존재 | 세션 ID 탈취 위험 존재 |
| 확장성 | 서버 저장소 불필요 | 공유 세션 저장소 등이 필요할 수 있음 |
| 만료 | 쿠키 속성에 따라 결정 | 서버의 세션 만료 정책에 따라 결정 |

### 1.7 JWT (JSON Web Token)

- JSON 기반의 정보를 안전하게 전달하기 위한 토큰 형식임
- 서명된 JWT는 토큰의 무결성과 발급 주체를 검증할 수 있음
- 서버가 세션 상태를 저장하지 않고도 인증 정보를 검증하는 방식에 활용할 수 있음
- JWT 자체가 데이터를 암호화하는 것은 아니므로 Payload의 내용은 확인 가능함

**JWT 구조**

```text
Header.Payload.Signature
```

**Header**

- 토큰 유형과 서명 알고리즘 등의 정보를 포함함
- 일반적으로 typ, alg 등의 필드를 사용함

**Payload**

- 사용자 식별자, 권한, 만료 시간 등의 Claim을 포함함
- Base64URL로 인코딩되며 암호화되지 않으므로 민감한 정보 저장을 피해야 함

**Signature**

- Header와 Payload를 기반으로 생성한 서명값임
- 토큰이 변조되지 않았는지 확인하는 데 사용함
- HMAC 방식은 비밀키를 사용하며 RSA 등의 비대칭키 방식은 개인키와 공개키를 활용함

**JWT 인증 과정**

1. 사용자가 로그인 요청
2. 서버가 사용자 인증 후 JWT 발급
3. 클라이언트가 JWT 저장
4. 클라이언트가 API 요청 시 JWT 전달
5. 서버가 서명과 만료 시간 등을 검증
6. 검증 성공 시 사용자 정보와 권한을 기반으로 요청 처리

**장점**

- 서버에 세션 정보를 저장하지 않는 Stateless 인증 구현 가능
- 여러 서버에서 동일한 검증 키를 활용해 토큰 검증 가능
- MSA 환경에서 인증 정보 전달에 활용 가능

**단점**

- 발급된 토큰을 만료 전에 즉시 무효화하기 어려움
- 토큰 탈취 시 만료 전까지 악용될 가능성 존재
- Payload에 민감한 정보를 저장하면 정보 노출 위험 존재
- 토큰 크기에 따라 요청 데이터가 증가할 수 있음

### 1.8 OAuth (Open Authorization)

- 사용자의 비밀번호를 외부 서비스에 직접 제공하지 않고 특정 리소스에 대한 접근 권한을 위임하는 표준 프레임워크임
- Google, Kakao 등의 소셜 로그인 구현에 활용됨
- OAuth 2.0 자체는 인가 프레임워크이며 사용자 인증은 일반적으로 OpenID Connect(OIDC) 또는 제공자의 사용자 정보 API를 활용함

**OAuth 2.0 주요 구성 요소**

| 구성 요소 | 설명 |
|---|---|
| Resource Owner | 리소스 소유자 |
| Client | 리소스 접근 권한을 요청하는 애플리케이션 |
| Authorization Server | 사용자 동의 및 토큰 발급 담당 서버 |
| Resource Server | 보호된 리소스를 제공하는 서버 |

**Authorization Code 방식**

1. 사용자가 소셜 로그인 버튼 클릭
2. 클라이언트가 Authorization Server로 인증 및 동의 요청
3. 사용자가 로그인하고 권한 제공에 동의
4. Authorization Server가 Authorization Code 발급
5. 클라이언트가 Authorization Code를 이용해 Access Token 요청
6. Authorization Server가 코드를 검증한 후 토큰 발급
7. 클라이언트가 Access Token으로 사용자 정보 요청
8. 서비스 서버가 사용자 식별 및 자체 로그인 처리

- Authorization Code 방식은 OAuth 2.0에서 널리 사용되는 방식임
- PKCE를 적용해 Authorization Code 탈취 및 악용 위험을 줄일 수 있음
- OAuth Access Token과 서비스 자체 JWT는 목적과 발급 주체가 다를 수 있음

---

## 2. Backend 심화

### 2.1 Spring Security

- Spring 기반 애플리케이션에서 인증과 인가를 처리하는 보안 프레임워크임
- 인증, 인가, CSRF 방어, 세션 관리, 보안 헤더 등의 기능을 제공함
- Servlet Filter 기반으로 HTTP 요청을 가로채 보안 처리를 수행함
- SecurityFilterChain을 통해 요청별 보안 정책을 설정할 수 있음

**주요 구성 요소**

| 구성 요소 | 역할 |
|---|---|
| SecurityFilterChain | 보안 필터 체인 구성 |
| Authentication | 인증된 사용자 정보 및 권한 표현 |
| AuthenticationManager | 인증 처리 진입점 |
| AuthenticationProvider | 실제 인증 로직 수행 |
| UserDetailsService | 사용자 정보 조회 |
| PasswordEncoder | 비밀번호 해싱 및 검증 |
| SecurityContextHolder | 현재 실행 흐름의 인증 정보 관리 |

**Spring Security 인증 흐름**

1. 클라이언트가 인증 요청 전송
2. SecurityFilterChain의 필터가 요청 처리
3. 인증 필터가 인증 정보를 추출
4. AuthenticationManager가 인증 처리 위임
5. AuthenticationProvider가 사용자 정보 및 자격 증명 검증
6. 인증 성공 시 Authentication 객체 생성
7. SecurityContextHolder에 인증 정보 저장
8. 인가 처리 후 요청한 리소스에 접근

**Spring Security 설정 예시**

```java
@Configuration
@EnableWebSecurity
public class SecurityConfig {

    @Bean
    public SecurityFilterChain filterChain(HttpSecurity http)
            throws Exception {

        http
            .csrf(csrf -> csrf.disable())
            .sessionManagement(session ->
                session.sessionCreationPolicy(
                    SessionCreationPolicy.STATELESS
                )
            )
            .authorizeHttpRequests(auth -> auth
                .requestMatchers("/api/auth/**").permitAll()
                .requestMatchers("/api/admin/**").hasRole("ADMIN")
                .anyRequest().authenticated()
            );

        return http.build();
    }
}
```

- 위 설정은 Stateless API에서 사용하는 기본적인 보안 구성 예시임
- CSRF 비활성화는 인증 정보 전달 방식과 CSRF 공격 가능성을 검토한 후 결정해야 함
- 실제 JWT 인증을 적용하려면 JWT 검증 필터 또는 OAuth2 Resource Server 등의 추가 구성이 필요함

### 2.2 JWT 검증

- 클라이언트가 전달한 JWT의 유효성을 확인하는 과정임
- 단순히 토큰을 디코딩하는 것과 서명을 검증하는 것은 다른 작업임
- 서버는 JWT의 서명, 만료 시간, 발급자, 대상 등을 검증해야 함

**JWT 검증 과정**

1. Authorization 헤더에서 Bearer Token 추출
2. JWT 구조 및 허용된 서명 알고리즘 확인
3. 서버의 검증 키로 Signature 검증
4. exp Claim을 통해 만료 여부 확인
5. iss, aud 등 필요한 Claim 검증
6. 사용자 식별자와 권한 정보 추출
7. Authentication 객체 생성 및 SecurityContext 등록
8. 인가 정책에 따라 API 접근 허용 여부 결정

**주요 JWT Claim**

| Claim | 설명 |
|---|---|
| sub | 토큰의 주체 |
| iss | 토큰 발급자 |
| aud | 토큰 사용 대상 |
| exp | 토큰 만료 시간 |
| iat | 토큰 발급 시간 |
| nbf | 토큰 활성화 시작 시간 |
| jti | 토큰 고유 식별자 |

**보안 고려 사항**

- 서명 검증 없이 Payload를 신뢰하면 안 됨
- 허용할 서명 알고리즘을 서버에서 명시적으로 제한해야 함
- 비밀키는 환경 변수나 Secret Manager 등을 통해 안전하게 관리해야 함
- 만료 시간이 짧은 Access Token 사용 권장
- 필요에 따라 토큰 폐기 목록 또는 사용자 상태 검증 적용

### 2.3 Refresh Token

- Access Token이 만료되었을 때 새로운 Access Token을 발급받기 위한 토큰임
- Access Token의 유효 기간을 짧게 유지하면서 사용자 로그인 상태를 지속하기 위해 사용함
- 일반적으로 Access Token보다 긴 만료 시간을 설정함
- Refresh Token은 JWT 또는 불투명한 랜덤 문자열 형태로 구현할 수 있음

**Access Token과 Refresh Token 차이**

| 구분 | Access Token | Refresh Token |
|---|---|---|
| 목적 | API 접근 인증 | Access Token 재발급 |
| 만료 시간 | 상대적으로 짧음 | 상대적으로 김 |
| 사용 위치 | 일반 API 요청 | 토큰 재발급 요청 |
| 탈취 위험 | API 접근 악용 | 장기간 토큰 재발급 악용 |
| 관리 | Stateless 검증 가능 | 서버 저장 및 폐기 관리 권장 |

**Refresh Token 동작 과정**

1. 사용자가 로그인에 성공
2. 서버가 Access Token과 Refresh Token 발급
3. 클라이언트가 Access Token으로 API 요청
4. Access Token 만료 시 Refresh Token으로 재발급 요청
5. 서버가 Refresh Token의 유효성 및 폐기 여부 검증
6. 검증 성공 시 새로운 Access Token 발급
7. 필요에 따라 Refresh Token도 새로 발급

**Refresh Token Rotation**

- Refresh Token을 사용할 때마다 새로운 Refresh Token을 발급하는 방식임
- 기존 Refresh Token은 재사용할 수 없도록 무효화함
- 이미 사용한 Refresh Token이 다시 제출되면 탈취 가능성을 탐지할 수 있음
- Redis 또는 DB를 활용해 토큰 상태와 만료 정보를 관리할 수 있음

**보안 고려 사항**

- Refresh Token은 안전한 저장소에 보관해야 함
- 브라우저 환경에서는 HttpOnly, Secure 쿠키 사용을 고려할 수 있음
- 로그아웃 시 서버에 저장된 Refresh Token을 폐기하는 방식 활용 가능
- Refresh Token 재사용 탐지 및 토큰 계열 폐기 정책 적용 가능
- Refresh Token을 쿠키로 전달하는 경우 CSRF 방어도 고려해야 함

### 2.4 OAuth 서버 처리

- 백엔드 서버에서 OAuth 제공자와 통신해 인가 코드를 토큰으로 교환하고 사용자 정보를 조회하는 과정임
- 소셜 로그인 제공자의 Access Token과 서비스 자체 인증 토큰을 구분해 관리해야 함
- Spring Security OAuth2 Client를 활용하면 OAuth 로그인 과정을 구현할 수 있음

**백엔드 OAuth 로그인 처리 과정**

1. 사용자가 Google 또는 Kakao 로그인 요청
2. 서비스가 OAuth 제공자의 인증 페이지로 리다이렉트
3. 사용자가 로그인 및 권한 제공 동의
4. OAuth 제공자가 Redirect URI로 Authorization Code 전달
5. 백엔드가 Authorization Code를 이용해 토큰 요청
6. OAuth 제공자가 Access Token 발급
7. 백엔드가 Access Token으로 사용자 정보 조회
8. 사용자 고유 식별자를 기준으로 회원 조회 또는 신규 가입 처리
9. 서비스 자체 Access Token과 Refresh Token 발급
10. 클라이언트가 자체 토큰으로 서비스 API 이용

**OAuth 구현 시 주요 고려 사항**

- state 파라미터를 활용해 요청과 콜백의 연관성을 검증하고 CSRF 위험 방지
- Authorization Code Flow에서 PKCE 적용 권장
- Redirect URI를 사전에 등록하고 엄격하게 검증
- Client Secret은 서버에서 안전하게 관리
- 사용자 식별 시 이메일보다 제공자의 고유 사용자 ID 활용 고려
- OAuth 제공자의 Access Token을 서비스 자체 JWT와 혼동하지 않도록 구분
- 외부 API 호출 실패 및 토큰 만료에 대한 예외 처리 필요

**OAuth와 JWT의 차이**

| 구분 | OAuth 2.0 | JWT |
|---|---|---|
| 개념 | 권한 위임 프레임워크 | 토큰 형식 |
| 목적 | 리소스 접근 권한 위임 | 정보 전달 및 무결성 검증 |
| 역할 | 토큰 발급 및 사용 절차 정의 | 토큰 데이터 구조 정의 |
| 사용 예시 | Google, Kakao 로그인 | 서비스 API 인증 |
| 관계 | OAuth에서 JWT 사용 가능 | OAuth 없이도 JWT 사용 가능 |

- OAuth와 JWT는 서로 대체하는 기술이 아니라 함께 사용할 수 있는 기술임
- OAuth 2.0의 Access Token은 반드시 JWT 형식일 필요는 없음
- OAuth 로그인 성공 후 서비스 자체 JWT를 발급하는 구조가 가능함

