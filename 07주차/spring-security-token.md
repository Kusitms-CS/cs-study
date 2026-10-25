# Spring Security와 토큰 처리 (Backend 심화)

## Spring Security

Spring Security는 Spring 애플리케이션의 인증, 인가, 공격 방어(CSRF, 세션 고정 등)를 담당하는 프레임워크다. Spring Boot 3 / Spring Security 6 기준으로 정리했다.

### Filter Chain 구조

Spring Security는 **서블릿 필터** 기반으로 동작한다. 요청이 `DispatcherServlet`(컨트롤러)에 도달하기 전에 필터들이 인증/인가를 처리한다.

```text
Client → Servlet Filter들 → DelegatingFilterProxy → FilterChainProxy → SecurityFilterChain(필터 N개) → DispatcherServlet → Controller
```

- **DelegatingFilterProxy**: 서블릿 컨테이너의 필터와 Spring 컨테이너의 빈을 연결하는 다리. 서블릿 필터는 Spring 빈을 모르기 때문에, 이 필터가 요청을 Spring 빈인 `FilterChainProxy`에 위임한다.
- **FilterChainProxy**: 요청 URL에 맞는 `SecurityFilterChain`을 골라 실행한다.
- **SecurityFilterChain**: 실제 보안 필터들의 목록. 여러 개를 정의해 URL별로 다른 보안 정책을 적용할 수 있다.

주요 필터 (실행 순서대로 일부)

| 필터 | 역할 |
| --- | --- |
| `CorsFilter` | CORS 처리 (Preflight 응답) |
| `CsrfFilter` | CSRF 토큰 검증 |
| `LogoutFilter` | 로그아웃 처리 |
| `UsernamePasswordAuthenticationFilter` | 폼 로그인 처리 |
| `BearerTokenAuthenticationFilter` | `Authorization: Bearer` 토큰 처리 (Resource Server) |
| `ExceptionTranslationFilter` | 인증/인가 예외를 401/403 응답으로 변환 |
| `AuthorizationFilter` | 최종 인가 판단 (URL별 권한 확인) |

설정은 `SecurityFilterChain` 빈을 등록하는 방식으로 한다. (예전의 `WebSecurityConfigurerAdapter` 상속 방식은 6.0에서 제거됨)

```java
@Configuration
@EnableWebSecurity
public class SecurityConfig {

    @Bean
    SecurityFilterChain filterChain(HttpSecurity http) throws Exception {
        http
            .cors(Customizer.withDefaults())
            .csrf(csrf -> csrf.disable()) // 쿠키를 쓰지 않는 토큰 기반 API일 때
            .sessionManagement(s -> s.sessionCreationPolicy(SessionCreationPolicy.STATELESS))
            .authorizeHttpRequests(auth -> auth
                .requestMatchers("/api/auth/**").permitAll()
                .requestMatchers("/api/admin/**").hasRole("ADMIN")
                .anyRequest().authenticated())
            .addFilterBefore(jwtAuthenticationFilter, UsernamePasswordAuthenticationFilter.class);
        return http.build();
    }
}
```

- `csrf.disable()`은 **인증 정보를 쿠키로 보내지 않을 때만** 안전하다. Refresh Token을 쿠키로 보낸다면 해당 엔드포인트는 CSRF를 고려해야 한다.

### 인증 처리 흐름 (SecurityContext / AuthenticationManager)

```text
SecurityContextHolder
  └─ SecurityContext
       └─ Authentication
            ├─ Principal   (사용자 정보, 보통 UserDetails)
            ├─ Credentials (비밀번호 등, 인증 후 보통 지움)
            └─ Authorities (권한 목록, ROLE_USER 등)
```

- **SecurityContextHolder**: 현재 인증된 사용자 정보를 보관한다. 기본적으로 **ThreadLocal**을 사용하므로 같은 요청을 처리하는 스레드 어디서든 꺼낼 수 있다.
- **Authentication**: 인증 전에는 사용자가 입력한 정보를, 인증 후에는 인증된 사용자 정보와 권한을 담는다.

폼 로그인 기준 흐름

1. `UsernamePasswordAuthenticationFilter`가 아이디/비밀번호로 미인증 상태의 `Authentication`을 만든다.
2. `AuthenticationManager`(구현체 `ProviderManager`)에 인증을 요청한다.
3. `ProviderManager`는 처리 가능한 `AuthenticationProvider`(예: `DaoAuthenticationProvider`)에 위임한다.
4. `DaoAuthenticationProvider`는 `UserDetailsService`로 DB에서 사용자를 조회하고, `PasswordEncoder`로 비밀번호를 비교한다.
5. 성공하면 인증된 `Authentication`을 `SecurityContextHolder`에 저장한다.

컨트롤러에서는 다음처럼 꺼내 쓴다.

```java
@GetMapping("/api/me")
public UserResponse me(@AuthenticationPrincipal UserDetails user) {
    return userService.findByUsername(user.getUsername());
}
```

## JWT 검증

### 서명과 클레임 검증

JWT를 받으면 다음 순서로 검증해야 한다.

1. **형식 확인**: `header.payload.signature` 구조인지
2. **알고리즘 고정**: 서버가 기대하는 알고리즘만 허용한다. Header의 `alg` 값을 그대로 믿으면 `alg: none`(서명 없음) 공격이나 RS256 ↔ HS256 혼동 공격에 당할 수 있다.
3. **서명 검증**: 비밀키(HS256) 또는 공개키(RS256)로 서명이 맞는지 확인
4. **클레임 검증**
   - `exp`: 만료되지 않았는지 (서버 간 시간 차이를 고려해 약간의 여유(clock skew)를 둠)
   - `nbf`: 아직 사용 시작 전이 아닌지
   - `iss`: 우리가 신뢰하는 발급자인지
   - `aud`: 이 서버를 대상으로 발급된 토큰인지
5. (선택) **블랙리스트 확인**: 로그아웃된 토큰인지 Redis 등에서 확인

> 디코딩(decode)과 검증(verify)은 다르다. Payload는 서명 검증 없이도 읽을 수 있지만, 검증하지 않은 값은 절대 신뢰하면 안 된다.

### JWT 검증 필터 구현

직접 구현한다면 `OncePerRequestFilter`를 상속해 요청당 한 번 실행되는 필터를 만든다. (jjwt 0.12 기준)

```java
@Component
@RequiredArgsConstructor
public class JwtAuthenticationFilter extends OncePerRequestFilter {

    private final JwtProvider jwtProvider;

    @Override
    protected void doFilterInternal(HttpServletRequest request, HttpServletResponse response,
                                    FilterChain chain) throws ServletException, IOException {
        String header = request.getHeader("Authorization");

        if (header != null && header.startsWith("Bearer ")) {
            String token = header.substring(7);
            try {
                Authentication auth = jwtProvider.getAuthentication(token);
                SecurityContextHolder.getContext().setAuthentication(auth);
            } catch (JwtException e) {
                // 인증 정보를 세팅하지 않고 넘기면, 보호된 URL에서 ExceptionTranslationFilter가 401 처리
                SecurityContextHolder.clearContext();
            }
        }
        chain.doFilter(request, response);
    }
}
```

```java
public Authentication getAuthentication(String token) {
    Claims claims = Jwts.parser()
            .verifyWith(secretKey)          // 서명 검증 키 (SecretKey)
            .requireIssuer("my-service")    // iss 검증
            .build()
            .parseSignedClaims(token)       // 서명 + exp 검증, 실패 시 JwtException
            .getPayload();

    List<GrantedAuthority> authorities = List.of(new SimpleGrantedAuthority(claims.get("role", String.class)));
    return new UsernamePasswordAuthenticationToken(claims.getSubject(), null, authorities);
}
```

- 토큰이 없거나 잘못됐을 때 필터에서 바로 응답을 쓰기보다, 인증 없이 다음 필터로 넘기고 `AuthenticationEntryPoint`에서 401 응답 형식을 통일하는 것이 깔끔하다.
- 직접 구현하지 않고 Spring Security의 **OAuth2 Resource Server** 기능(`oauth2ResourceServer().jwt()`)을 써도 같은 검증을 할 수 있다. ([Resource Server](#resource-server) 참고)

## Refresh Token

### Access Token과 Refresh Token

JWT는 서버가 강제로 무효화하기 어렵기 때문에, **수명이 짧은 Access Token**과 **수명이 긴 Refresh Token**을 나눠 쓴다.

| | Access Token | Refresh Token |
| --- | --- | --- |
| 용도 | API 요청 인증 | Access Token 재발급 |
| 수명 | 짧음 (15분~1시간) | 김 (1~2주) |
| 전송 빈도 | 매 요청 | 재발급할 때만 |
| 서버 저장 | 보통 안 함 (Stateless) | 저장함 (DB/Redis) |

재발급 흐름

1. 로그인 시 Access Token과 Refresh Token을 함께 발급하고, Refresh Token은 서버(Redis 등)에도 저장한다.
2. API 요청 중 Access Token이 만료되면 서버가 401을 응답한다.
3. 클라이언트가 Refresh Token으로 `/api/auth/reissue`를 호출한다.
4. 서버는 Refresh Token의 서명과 만료를 검증하고, **저장된 값과 일치하는지** 확인한 뒤 새 Access Token을 발급한다.

- Access Token이 탈취되어도 피해 기간이 짧다.
- Refresh Token은 서버에 저장되어 있으므로, 로그아웃이나 계정 정지 시 **서버에서 삭제해 재발급을 막을 수 있다.**

### Refresh Token 저장과 Rotation

**클라이언트 저장 위치**

- Refresh Token은 `HttpOnly; Secure; SameSite` 쿠키에 담는 경우가 많다. JS에서 읽을 수 없어 XSS로 탈취하기 어렵다.
- 이때 재발급 엔드포인트는 쿠키로 인증되므로 CSRF를 고려해야 한다. `SameSite=Strict`, `Path=/api/auth/reissue`로 전송 범위를 좁히는 방법이 있다.

**서버 저장 (Redis 예시)**

```text
key:   refresh:{userId}
value: {refreshToken}
TTL:   14일 (Refresh Token 만료 시간과 동일)
```

- TTL을 걸어두면 만료된 토큰이 자동으로 정리된다.
- 기기별 로그인을 지원하려면 `refresh:{userId}:{deviceId}`처럼 키를 나눈다.

**Refresh Token Rotation**

Refresh Token을 **한 번 쓰면 폐기하고 새 Refresh Token을 함께 발급**하는 방식이다.

1. 클라이언트가 Refresh Token `R1`으로 재발급 요청
2. 서버는 `R1`을 폐기하고 새 Access Token과 Refresh Token `R2`를 발급
3. 이후 누군가 이미 폐기된 `R1`으로 요청하면, **탈취된 것으로 판단**하고 해당 사용자의 Refresh Token을 전부 무효화

정상 사용자와 공격자 중 누가 먼저 쓰든, 다른 쪽이 옛 토큰을 쓰는 순간 재사용이 감지된다. [RFC 9700](https://www.rfc-editor.org/rfc/rfc9700)은 Public Client의 Refresh Token에 Rotation 또는 Sender-constrained 토큰 적용을 요구한다.

주의할 점: 클라이언트가 동시에 여러 요청을 보내 재발급이 겹치면 정상 사용자도 재사용으로 오탐될 수 있다. 프론트에서 재발급 요청을 하나로 모으거나, 서버에서 짧은 유예 시간을 두는 식으로 처리한다.

## OAuth 서버 처리

Spring Security는 OAuth 관련 역할 중 **Client**와 **Resource Server**를 지원한다. (Authorization Server는 별도 프로젝트인 Spring Authorization Server)

### OAuth2 Login (Client)

"구글로 로그인"처럼 외부 Provider로 로그인하는 경우다. `spring-boot-starter-oauth2-client`를 추가하고 설정만 하면 Authorization Code 흐름을 Spring Security가 처리해준다.

```yaml
spring:
  security:
    oauth2:
      client:
        registration:
          google:
            client-id: ${GOOGLE_CLIENT_ID}
            client-secret: ${GOOGLE_CLIENT_SECRET}
            scope: openid, profile, email
```

```java
http.oauth2Login(oauth -> oauth
        .userInfoEndpoint(u -> u.userService(customOAuth2UserService))
        .successHandler(oAuth2SuccessHandler));
```

동작 흐름

1. 사용자가 `/oauth2/authorization/google`로 접근하면 `OAuth2AuthorizationRequestRedirectFilter`가 `state`(OIDC면 `nonce`도)를 만들어 구글 인가 페이지로 리다이렉트한다.
2. 구글이 `/login/oauth2/code/google?code=...&state=...`로 돌려보낸다.
3. `OAuth2LoginAuthenticationFilter`가 `state`를 검증하고 code로 토큰을 교환한다.
4. `OAuth2UserService`(OIDC면 `OidcUserService`)가 사용자 정보를 가져온다.
5. 인증 성공 시 `successHandler`가 실행된다.

실무에서 직접 구현하는 부분

- **`CustomOAuth2UserService`**: Provider에서 받은 사용자 정보로 우리 DB의 회원을 조회하거나 새로 가입시킨다. Provider마다 응답 형식이 달라(Google은 `sub`, Kakao는 `id` 아래 중첩 구조) 이를 공통 형태로 변환하는 작업이 필요하다.
- **`OAuth2SuccessHandler`**: 세션 대신 JWT를 쓰는 경우, 여기서 우리 서비스의 Access/Refresh Token을 발급해 프론트로 리다이렉트한다.
  - 토큰을 리다이렉트 URL 쿼리에 그대로 붙이면 히스토리, 로그에 남을 수 있다. Refresh Token은 `HttpOnly` 쿠키로 내려주고, Access Token은 짧은 일회용 코드를 넘겨 다시 교환하게 하는 식으로 노출을 줄인다.

### Resource Server

다른 서버(인증 서버, Keycloak, Cognito 등)가 발급한 JWT를 받아 **검증만 하는** API 서버 역할이다. `spring-boot-starter-oauth2-resource-server`를 추가한다.

```yaml
spring:
  security:
    oauth2:
      resourceserver:
        jwt:
          issuer-uri: https://auth.example.com
```

```java
http.oauth2ResourceServer(oauth -> oauth.jwt(Customizer.withDefaults()));
```

- `BearerTokenAuthenticationFilter`가 `Authorization: Bearer` 헤더에서 토큰을 꺼낸다.
- `JwtDecoder`가 `issuer-uri`의 Discovery 문서로 공개키(JWK Set)를 가져와 **서명, `exp`, `iss`를 자동으로 검증**한다. 공개키는 캐시하고, 키가 교체(rotation)되면 다시 가져온다.
- `JwtAuthenticationConverter`가 `scope` 클레임을 `SCOPE_` 접두사가 붙은 권한으로 변환한다. 다른 클레임(예: `roles`)을 권한으로 쓰려면 이 컨버터를 커스텀한다.

직접 구현한 [JWT 검증 필터](#jwt-검증-필터-구현)와 비교하면, 알고리즘 고정, 키 관리, 클레임 검증을 프레임워크가 처리해주므로 실수할 여지가 적다. 자체 발급 HS256 토큰도 `NimbusJwtDecoder.withSecretKey(key)`로 Resource Server 방식에 연결할 수 있다.
