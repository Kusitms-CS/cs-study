# 7주차 - Web Security / 인증 (Backend)

예시는 Spring Boot 3 / Spring Security 6 기준이다.

## 1. Spring Security

### 인증과 인가

| 구분 | 인증 (Authentication) | 인가 (Authorization) |
| --- | --- | --- |
| 질문 | 누구인가? | 이 요청을 해도 되는가? |
| 시점 | 먼저 수행 | 인증 이후 수행 |
| 실패 시 응답 | `401 Unauthorized` | `403 Forbidden` |
| Spring Security 예외 | `AuthenticationException` | `AccessDeniedException` |

`401`의 이름은 Unauthorized지만 의미는 "인증되지 않음"이다. 로그인은 했지만 권한이 없는 경우는 `403`이다.

### Filter 기반 구조

Spring Security는 DispatcherServlet(Controller)에 요청이 도달하기 **전에** Servlet Filter 단계에서 동작한다.

```
Client
  → Servlet Container (Tomcat)의 Filter Chain
      → DelegatingFilterProxy        // Servlet Filter와 Spring Bean을 연결
          → FilterChainProxy         // 요청 URL에 맞는 SecurityFilterChain 선택
              → SecurityFilterChain  // 보안 Filter들을 순서대로 실행
  → DispatcherServlet → Controller
```

- **DelegatingFilterProxy**: Servlet Container는 Spring Bean을 모르기 때문에, 컨테이너에 등록된 이 Filter가 요청을 Spring의 `FilterChainProxy` Bean에 넘긴다.
- **FilterChainProxy**: `SecurityFilterChain`을 여러 개 등록해 두고, 요청 URL에 맞는 체인 **하나**를 골라 실행한다. (예: `/api/**`는 JWT 체인, `/admin/**`는 세션 로그인 체인)
- **SecurityFilterChain**: 실제 보안 Filter들의 목록이다.

주요 Filter는 다음 순서로 실행된다.

| Filter | 역할 |
| --- | --- |
| `SecurityContextHolderFilter` | 저장된 `SecurityContext`(세션 등)를 불러옴 |
| `CorsFilter` | CORS 처리. Preflight(`OPTIONS`) 요청에 응답 |
| `CsrfFilter` | CSRF 토큰 검증 |
| `LogoutFilter` | 로그아웃 요청 처리 |
| `UsernamePasswordAuthenticationFilter` | 폼 로그인(아이디/비밀번호) 인증 |
| `BearerTokenAuthenticationFilter` | `Authorization: Bearer` 토큰 인증 (Resource Server 사용 시) |
| `AnonymousAuthenticationFilter` | 인증 정보가 없으면 익명 사용자로 채움 |
| `ExceptionTranslationFilter` | 뒤에서 발생한 인증/인가 예외를 `401`/`403` 응답으로 변환 |
| `AuthorizationFilter` | URL별 접근 권한 검사 |

직접 만든 JWT Filter는 보통 `UsernamePasswordAuthenticationFilter` 앞에 끼워 넣는다.

### 인증 처리 흐름

```
Filter
  → 아직 인증되지 않은 Authentication 생성 (아이디, 비밀번호)
  → AuthenticationManager (구현체: ProviderManager)
      → AuthenticationProvider 중 이 Authentication을 처리할 수 있는 것 선택
          → DaoAuthenticationProvider
              → UserDetailsService.loadUserByUsername()   // DB에서 사용자 조회
              → PasswordEncoder.matches()                 // 비밀번호 비교
  → 인증된 Authentication 반환 (principal, authorities)
  → SecurityContextHolder에 저장
```

| 구성 요소 | 역할 |
| --- | --- |
| `Authentication` | 인증 정보. 인증 전에는 입력값, 인증 후에는 사용자 정보와 권한(`authorities`)을 담음 |
| `AuthenticationManager` | 인증 요청을 받아 적절한 Provider에게 위임 |
| `AuthenticationProvider` | 실제 인증 로직. 방식(비밀번호, OAuth, JWT 등)마다 구현체가 다름 |
| `UserDetailsService` | 사용자 이름으로 사용자 정보(`UserDetails`)를 조회 |
| `PasswordEncoder` | 비밀번호 해시와 비교. 보통 `BCryptPasswordEncoder` |
| `SecurityContextHolder` | 현재 요청의 인증 정보를 보관 |

### SecurityContextHolder

`SecurityContextHolder`는 기본적으로 **ThreadLocal**에 인증 정보를 저장한다. 요청을 처리하는 스레드 안에서는 어디서든 현재 사용자를 꺼낼 수 있다.

```java
Authentication auth = SecurityContextHolder.getContext().getAuthentication();

// Controller에서는 파라미터로 받는 것이 편하다
@GetMapping("/me")
public UserResponse me(@AuthenticationPrincipal CustomUserDetails user) { ... }
```

- ThreadLocal이므로 `@Async`나 별도 스레드 풀에서는 인증 정보가 **전달되지 않는다.** `DelegatingSecurityContextExecutor` 등으로 감싸서 전달해야 한다.
- 스레드 풀은 스레드를 재사용하므로, 요청이 끝나면 반드시 비워야 한다(Spring Security가 처리한다). 직접 Filter에서 설정했다면 실패 시 `clearContext()`를 호출한다.

### 설정 예시 (JWT 기반 API 서버)

```java
@Configuration
@EnableWebSecurity
@EnableMethodSecurity
@RequiredArgsConstructor
public class SecurityConfig {

    private final JwtProvider jwtProvider;

    @Bean
    public SecurityFilterChain filterChain(HttpSecurity http) throws Exception {
        http
            .csrf(csrf -> csrf.disable())                         // Header로 토큰을 보내므로 CSRF 위험이 낮음
            .cors(Customizer.withDefaults())                      // CorsConfigurationSource Bean 사용
            .formLogin(form -> form.disable())
            .httpBasic(basic -> basic.disable())
            .sessionManagement(session ->
                session.sessionCreationPolicy(SessionCreationPolicy.STATELESS))
            .authorizeHttpRequests(auth -> auth
                .requestMatchers("/auth/**", "/oauth2/**", "/login/oauth2/**").permitAll()
                .requestMatchers(HttpMethod.GET, "/posts/**").permitAll()
                .requestMatchers("/admin/**").hasRole("ADMIN")
                .anyRequest().authenticated())
            .exceptionHandling(e -> e
                .authenticationEntryPoint(new JwtAuthenticationEntryPoint())   // 401
                .accessDeniedHandler(new JwtAccessDeniedHandler()))            // 403
            .addFilterBefore(new JwtAuthenticationFilter(jwtProvider),
                             UsernamePasswordAuthenticationFilter.class);
        return http.build();
    }

    @Bean
    public PasswordEncoder passwordEncoder() {
        return PasswordEncoderFactories.createDelegatingPasswordEncoder(); // 기본 bcrypt
    }
}
```

- `authorizeHttpRequests`는 **위에서부터 처음 일치하는 규칙**이 적용된다. 구체적인 경로를 위에, `anyRequest()`를 마지막에 둔다.
- `hasRole("ADMIN")`은 권한 이름 `ROLE_ADMIN`을 검사한다. 권한을 만들 때 `ROLE_` 접두사를 붙여야 한다(`hasAuthority`는 접두사 없이 그대로 비교).
- `STATELESS`는 Spring Security가 인증 정보를 세션에 저장하지 않는다는 뜻이다. 매 요청마다 토큰으로 다시 인증해야 한다.
- `@EnableMethodSecurity`를 켜면 메서드 단위로 `@PreAuthorize("hasRole('ADMIN')")`, `@PreAuthorize("#userId == authentication.principal.id")` 같은 검사를 할 수 있다.

### CSRF를 꺼도 되는 경우

CSRF는 브라우저가 **쿠키를 자동으로 붙여 보내는** 성질을 이용한 공격이다.

| 인증 정보 전달 방식 | CSRF 위험 | `csrf().disable()` |
| --- | --- | --- |
| `Authorization` 헤더 (JS가 직접 붙임) | 낮음. 다른 사이트가 헤더 값을 알 수 없음 | 가능 |
| 쿠키 (세션 ID, 쿠키에 담은 JWT) | 있음. 다른 사이트의 요청에도 쿠키가 자동 전송됨 | 신중히. `SameSite`, CSRF 토큰 필요 |

"JWT를 쓰니까 CSRF를 끈다"가 아니라 **토큰을 어디에 담아 보내느냐**가 기준이다. Refresh Token을 쿠키로 보낸다면 재발급 API는 CSRF를 고려해야 한다.

### 예외 처리 시 주의점

- Filter에서 발생한 예외는 DispatcherServlet 밖이므로 `@RestControllerAdvice`로 잡히지 않는다. 인증 실패 응답은 `AuthenticationEntryPoint`, 인가 실패 응답은 `AccessDeniedHandler`에서 만든다.
- `ExceptionTranslationFilter`보다 **앞에 있는 Filter**에서 던진 예외는 이 Filter가 처리하지 못한다. JWT Filter에서는 예외를 던지기보다 인증 정보를 비워 둔 채 다음 Filter로 넘기고, 이후 `AuthorizationFilter`가 미인증으로 판단하게 하는 방식이 안전하다.
- 커스텀 Filter를 `@Component`로 등록하면 Spring Boot가 Servlet Filter로도 자동 등록해 Security 체인 밖에서 한 번 더 실행될 수 있다. Security 체인에서만 쓸 Filter는 Bean으로 등록하지 않거나 `FilterRegistrationBean`으로 자동 등록을 끈다.

### CORS 설정

브라우저는 다른 출처로 요청을 보내기 전에 `OPTIONS` Preflight 요청을 보낸다. Preflight에는 `Authorization` 헤더가 없으므로, 인증 Filter보다 먼저 CORS가 처리되어야 한다. `http.cors()`를 설정하면 `CorsFilter`가 Security 체인 앞쪽에서 처리한다.

```java
@Bean
public CorsConfigurationSource corsConfigurationSource() {
    CorsConfiguration config = new CorsConfiguration();
    config.setAllowedOrigins(List.of("https://my-frontend.com"));
    config.setAllowedMethods(List.of("GET", "POST", "PUT", "PATCH", "DELETE", "OPTIONS"));
    config.setAllowedHeaders(List.of("Authorization", "Content-Type"));
    config.setAllowCredentials(true);          // 쿠키 전송 허용
    config.setMaxAge(3600L);                   // Preflight 결과 캐시 시간(초)

    UrlBasedCorsConfigurationSource source = new UrlBasedCorsConfigurationSource();
    source.registerCorsConfiguration("/**", config);
    return source;
}
```

`allowCredentials(true)`일 때는 `allowedOrigins("*")`를 쓸 수 없다. 출처를 명시해야 한다.

## 2. JWT 검증

### 구조

JWT는 `.`으로 구분된 세 부분으로 이루어진다.

```
eyJhbGciOiJIUzI1NiJ9 . eyJzdWIiOiIxIiwiZXhwIjoxNzYwMDAwMDAwfQ . 4f8x...서명
      Header                         Payload                     Signature
```

| 부분 | 내용 |
| --- | --- |
| Header | 서명 알고리즘(`alg`), 토큰 타입(`typ`), 키 식별자(`kid`) |
| Payload | Claim(토큰에 담는 정보) |
| Signature | `Header.Payload`를 비밀 키로 서명한 값 |

Header와 Payload는 **Base64URL 인코딩일 뿐 암호화가 아니다.** 누구나 디코딩해서 읽을 수 있으므로 비밀번호, 개인정보 등을 넣으면 안 된다. 서명은 내용을 숨기는 것이 아니라 **위조·변조되지 않았음**을 보장한다.

| 표준 Claim | 의미 |
| --- | --- |
| `iss` (Issuer) | 발급자 |
| `sub` (Subject) | 토큰의 주체. 보통 사용자 ID |
| `aud` (Audience) | 토큰을 사용할 대상 서버 |
| `exp` (Expiration) | 만료 시각 |
| `nbf` (Not Before) | 이 시각 이전에는 무효 |
| `iat` (Issued At) | 발급 시각 |
| `jti` (JWT ID) | 토큰 고유 ID. 폐기 목록 관리에 사용 |

### 서명 알고리즘

| 구분 | HS256 (대칭키) | RS256 / ES256 (비대칭키) |
| --- | --- | --- |
| 서명 | 비밀 키 | 개인 키 (Private Key) |
| 검증 | 같은 비밀 키 | 공개 키 (Public Key) |
| 키 공유 | 검증하는 모든 서버가 비밀 키를 가져야 함 | 검증 서버는 공개 키만 있으면 됨 |
| 적합한 경우 | 발급과 검증을 같은 서버(또는 하나의 서비스)에서 할 때 | 여러 서비스가 검증하거나, 외부(Google 등)가 발급한 토큰을 검증할 때 |

HS256의 비밀 키는 최소 256비트(32바이트) 이상의 무작위 값이어야 하며, 코드가 아니라 환경 변수나 Secret Manager로 관리한다. 비대칭키 방식에서는 발급자가 공개 키 목록을 JWKS(`/.well-known/jwks.json`) 형태로 제공하고, 검증 측은 Header의 `kid`로 맞는 공개 키를 찾는다. 키를 교체(Rotation)할 때도 `kid`로 기존 키와 새 키를 동시에 운영할 수 있다.

### 검증 단계

서명만 확인하고 끝내면 안 된다. 아래 항목을 모두 확인해야 "유효한 토큰"이다.

| 순서 | 검증 항목 | 실패하면 |
| --- | --- | --- |
| 1 | 형식: 세 부분으로 나뉘고 디코딩 가능한가 | 위조·손상된 토큰 |
| 2 | 알고리즘: **서버가 허용한 알고리즘**인가 | 알고리즘 공격 |
| 3 | 서명: 키로 검증했을 때 일치하는가 | 위조·변조 |
| 4 | 시간: `exp`가 지나지 않았고 `nbf` 이후인가 | 만료 |
| 5 | 발급자·대상: `iss`, `aud`가 기대한 값인가 | 다른 서비스용 토큰 |
| 6 | 용도: Access Token 자리에 Refresh Token을 넣지 않았는가 | 토큰 오용 |
| 7 | (선택) 상태: 폐기 목록에 없는가, 사용자가 탈퇴·정지되지 않았는가 | 폐기된 토큰 |

#### 알고리즘을 토큰에서 믿으면 안 되는 이유

Header의 `alg`는 공격자가 마음대로 바꿀 수 있는 값이다.

- **`alg: none`**: 서명이 없는 토큰이다. 라이브러리가 이를 허용하면 서명 없이 아무 Payload나 통과한다.
- **알고리즘 혼동 (RS256 → HS256)**: 서버가 RS256 공개 키로 검증하는데, 공격자가 `alg`를 HS256으로 바꾸고 **공개 키를 HMAC 비밀 키로 써서** 서명한다. 서버가 `alg`를 그대로 믿고 같은 공개 키로 HS256 검증을 하면 통과한다. 공개 키는 말 그대로 공개되어 있으므로 누구나 위조할 수 있게 된다.

따라서 검증에 사용할 알고리즘과 키는 **서버 설정으로 고정**하고, 토큰의 `alg`가 그와 다르면 거부한다. 최신 라이브러리는 키 타입으로 알고리즘을 제한하지만, 직접 검증 로직을 만들지 않는 것이 가장 안전하다.

### 구현 예시 (jjwt 0.12)

```java
@Component
public class JwtProvider {

    private final SecretKey key;
    private static final String ISSUER = "my-service";
    private static final Duration ACCESS_TTL = Duration.ofMinutes(30);

    public JwtProvider(@Value("${jwt.secret}") String secret) {
        this.key = Keys.hmacShaKeyFor(Decoders.BASE64.decode(secret)); // 256비트 미만이면 예외
    }

    public String createAccessToken(Long userId, String role) {
        Instant now = Instant.now();
        return Jwts.builder()
                .issuer(ISSUER)
                .subject(String.valueOf(userId))
                .claim("role", role)
                .claim("type", "access")
                .issuedAt(Date.from(now))
                .expiration(Date.from(now.plus(ACCESS_TTL)))
                .signWith(key)                     // 키 타입에 맞는 HS 알고리즘으로 서명
                .compact();
    }

    public Claims parse(String token) {
        return Jwts.parser()
                .verifyWith(key)                   // 이 키(HMAC)로만 검증
                .requireIssuer(ISSUER)
                .clockSkewSeconds(30)              // 서버 간 시계 오차 허용
                .build()
                .parseSignedClaims(token)          // 서명 없는 토큰(alg: none)은 예외
                .getPayload();
    }
}
```

`parseSignedClaims`는 형식, 서명, `exp`/`nbf`를 함께 검증하고 실패하면 예외를 던진다.

| 예외 | 원인 |
| --- | --- |
| `ExpiredJwtException` | 만료됨 |
| `SignatureException` | 서명 불일치 |
| `MalformedJwtException` | 형식 오류 |
| `IncorrectClaimException` / `MissingClaimException` | `require...`로 지정한 Claim 불일치·누락 |
| `JwtException` | 위 예외들의 상위 타입 |

만료와 그 외 실패는 클라이언트의 대응이 다르다. 만료면 Refresh Token으로 재발급을 시도하고, 위조·형식 오류면 다시 로그인해야 한다. 응답 본문에 오류 코드(예: `TOKEN_EXPIRED`, `INVALID_TOKEN`)를 구분해서 내려준다.

### JWT 인증 Filter

```java
@RequiredArgsConstructor
public class JwtAuthenticationFilter extends OncePerRequestFilter {

    private final JwtProvider jwtProvider;

    @Override
    protected void doFilterInternal(HttpServletRequest request, HttpServletResponse response,
                                    FilterChain chain) throws ServletException, IOException {
        String header = request.getHeader(HttpHeaders.AUTHORIZATION);

        if (header != null && header.startsWith("Bearer ")) {
            try {
                Claims claims = jwtProvider.parse(header.substring(7));
                if (!"access".equals(claims.get("type", String.class))) {
                    throw new JwtException("not an access token");
                }
                var authorities = List.of(new SimpleGrantedAuthority("ROLE_" + claims.get("role", String.class)));
                var authentication = new UsernamePasswordAuthenticationToken(
                        Long.valueOf(claims.getSubject()), null, authorities);  // 인증된 상태로 생성
                SecurityContextHolder.getContext().setAuthentication(authentication);
            } catch (ExpiredJwtException e) {
                request.setAttribute("authError", "TOKEN_EXPIRED");   // EntryPoint에서 응답에 사용
                SecurityContextHolder.clearContext();
            } catch (JwtException | IllegalArgumentException e) {
                request.setAttribute("authError", "INVALID_TOKEN");
                SecurityContextHolder.clearContext();
            }
        }
        chain.doFilter(request, response);   // 인증 여부와 무관하게 다음 Filter로
    }
}
```

- 토큰이 없거나 잘못되어도 여기서 응답을 끝내지 않는다. `permitAll()` 경로는 그대로 통과해야 하기 때문이다. 인증이 필요한 경로라면 뒤의 `AuthorizationFilter`가 막고 `AuthenticationEntryPoint`가 `401`을 응답한다.
- `UsernamePasswordAuthenticationToken`은 생성자에 따라 인증 여부가 달라진다. 권한(`authorities`)을 받는 3인자 생성자가 **인증된** 토큰을 만든다.
- 매 요청 DB에서 사용자를 조회하지 않고 토큰의 Claim만으로 인증하는 것이 JWT의 장점이다. 대신 권한이 바뀌어도 토큰이 만료될 때까지 이전 권한이 유지된다.

직접 구현 대신 Spring Security의 **Resource Server**(`spring-boot-starter-oauth2-resource-server`)를 쓰면 `BearerTokenAuthenticationFilter`와 `JwtDecoder`가 서명과 `exp`/`nbf` 검증을 처리한다. 외부 인증 서버(Keycloak, Cognito 등)가 발급한 토큰을 검증할 때는 이쪽이 표준이다.

```java
http.oauth2ResourceServer(oauth2 -> oauth2.jwt(Customizer.withDefaults()));
```

```yaml
spring:
  security:
    oauth2:
      resourceserver:
        jwt:
          issuer-uri: https://auth.example.com   # JWKS 조회 + iss 검증
```

### JWT의 한계

| 한계 | 내용 | 대응 |
| --- | --- | --- |
| 강제 만료 불가 | 서버가 상태를 저장하지 않으므로, 발급된 토큰은 `exp`까지 유효 | Access Token 수명을 짧게. 필요하면 `jti` 폐기 목록(Redis)을 `exp`까지만 보관 |
| 정보 갱신 지연 | 권한·사용자 정보가 바뀌어도 토큰에 반영되지 않음 | 짧은 수명 + 재발급 시 최신 정보 반영 |
| 크기 | 모든 요청 헤더에 실리므로 Claim이 많으면 트래픽 증가 | 식별자와 최소한의 권한만 담음 |
| 탈취 시 피해 | 토큰만 있으면 누구나 사용 가능 | HTTPS 필수, 짧은 수명, 저장 위치 주의 |

폐기 목록을 매 요청 조회하면 결국 상태를 저장하는 것이므로, "모든 요청을 즉시 차단해야 하는가"에 따라 세션 방식과 비교해서 선택한다.

## 3. Refresh Token

### 왜 필요한가

Access Token의 수명은 보안과 편의의 trade-off다.

- 길게 잡으면: 탈취되었을 때 오래 악용되고, 강제로 무효화하기 어렵다.
- 짧게 잡으면: 사용자가 자주 다시 로그인해야 한다.

그래서 토큰을 두 개로 나눈다.

| 구분 | Access Token | Refresh Token |
| --- | --- | --- |
| 용도 | API 요청 인증 | Access Token 재발급 |
| 수명 | 짧음 (예: 15~30분) | 김 (예: 2주) |
| 전송 빈도 | 매 요청 | 재발급할 때만 |
| 서버 저장 | 안 함 (Stateless) | **함** (DB/Redis) |
| 형식 | JWT | JWT 또는 무작위 문자열(Opaque Token) |

Refresh Token은 **서버에 저장하기 때문에** 강제로 폐기할 수 있다. 자주 쓰이는 Access Token은 Stateless로 빠르게 처리하고, 드물게 쓰이는 Refresh Token에서 상태를 관리해 통제권을 확보하는 구조다.

### 재발급 흐름

```
1. 로그인 성공 → Access Token(응답 본문) + Refresh Token(HttpOnly 쿠키) 발급
                 서버는 Refresh Token을 저장소에 저장
2. API 요청   → Authorization: Bearer {Access Token}
3. Access Token 만료 → 401 + TOKEN_EXPIRED
4. 클라이언트 → POST /auth/refresh (쿠키로 Refresh Token 자동 전송)
5. 서버       → Refresh Token 검증 (서명·만료 + 저장소에 존재하는지)
              → 새 Access Token + 새 Refresh Token 발급, 기존 Refresh Token 폐기
6. Refresh Token도 만료·무효 → 401 → 다시 로그인
```

### 저장 위치

**클라이언트 쪽**

| 저장 위치 | XSS로 탈취 | CSRF 위험 | 비고 |
| --- | --- | --- | --- |
| `localStorage` | 가능 (JS로 읽힘) | 없음 | Refresh Token 보관에 부적합 |
| JS 메모리 (변수) | 어려움 | 없음 | 새로고침 시 사라짐. Access Token 보관에 적합 |
| `HttpOnly` 쿠키 | 불가 (JS로 못 읽음) | 있음 | Refresh Token 보관에 적합 |

Refresh Token 쿠키는 다음 옵션을 함께 건다.

```java
ResponseCookie cookie = ResponseCookie.from("refresh_token", refreshToken)
        .httpOnly(true)               // JS 접근 차단
        .secure(true)                 // HTTPS에서만 전송
        .sameSite("Strict")           // 다른 사이트 요청에 쿠키를 붙이지 않음 (CSRF 방어)
        .path("/auth/refresh")        // 재발급 API에만 전송
        .maxAge(Duration.ofDays(14))
        .build();
response.addHeader(HttpHeaders.SET_COOKIE, cookie.toString());
```

`path`를 재발급 API로 제한하면 일반 API 요청마다 Refresh Token이 전송되지 않아 노출 범위가 줄어든다. 프론트엔드와 API의 도메인이 달라 `SameSite=None`이 필요하다면 CSRF 방어(Origin 검사, CSRF 토큰)를 따로 해야 한다.

**서버 쪽**

```
Key:   refresh:{userId}:{deviceId}
Value: Refresh Token의 해시값
TTL:   Refresh Token 만료 시간과 동일
```

- Redis를 쓰면 TTL로 만료된 토큰이 자동 삭제된다.
- 저장소가 유출되어도 바로 쓰이지 않도록 원문 대신 **해시값**을 저장한다. (비밀번호와 같은 원리)
- 기기(세션)별로 키를 나누면 "이 기기에서 로그아웃", "모든 기기에서 로그아웃"을 구현할 수 있다.

### Refresh Token Rotation

재발급할 때마다 **Refresh Token도 새로 발급하고 기존 것은 폐기**한다.

- 탈취된 Refresh Token의 유효 기간이 "다음 재발급 전까지"로 줄어든다.
- **재사용 감지**: 이미 폐기된 Refresh Token으로 재발급 요청이 오면, 누군가 토큰을 복사해서 쓰고 있다는 뜻이다. 정상 사용자와 공격자 중 누가 먼저 썼는지 알 수 없으므로, 해당 사용자의 Refresh Token을 **모두 폐기**하고 다시 로그인하게 한다.

```java
@Transactional
public TokenPair refresh(String refreshToken) {
    Claims claims = jwtProvider.parseRefresh(refreshToken);          // 서명·만료·type 검증
    Long userId = Long.valueOf(claims.getSubject());
    String deviceId = claims.get("deviceId", String.class);

    String saved = refreshTokenStore.find(userId, deviceId)
            .orElseThrow(() -> new InvalidTokenException());

    if (!saved.equals(hash(refreshToken))) {                         // 이미 교체된 토큰을 재사용
        refreshTokenStore.deleteAll(userId);                         // 전체 폐기
        throw new TokenReuseDetectedException();
    }

    TokenPair newTokens = tokenIssuer.issue(userId, deviceId);
    refreshTokenStore.save(userId, deviceId, hash(newTokens.refreshToken()));  // 기존 값 덮어쓰기
    return newTokens;
}
```

#### 동시 요청 문제

Access Token이 만료된 순간 화면에서 API 여러 개가 동시에 `401`을 받으면, 클라이언트가 재발급을 동시에 여러 번 요청한다. Rotation을 쓰면 첫 요청이 토큰을 교체한 뒤 두 번째 요청은 "폐기된 토큰 재사용"으로 판단되어 정상 사용자가 로그아웃된다.

- **클라이언트**: 재발급 요청을 하나만 보내고, 나머지 요청은 그 결과를 기다렸다가 재시도한다.
- **서버**: 직전 Refresh Token을 짧은 유예 시간(수 초) 동안은 허용한다.

### 로그아웃

1. 서버 저장소에서 Refresh Token을 삭제한다.
2. 쿠키를 `maxAge(0)`으로 덮어써서 삭제한다.
3. Access Token은 `exp`까지 유효하므로, 즉시 무효화가 필요하면 `jti`를 폐기 목록에 남은 수명만큼 저장한다.

## 4. OAuth 서버 처리

### 역할

OAuth 2.0은 사용자가 비밀번호를 우리 서버에 주지 않고, **다른 서비스(Google, Kakao)에 있는 자신의 정보에 접근할 권한을 위임**하는 프로토콜이다.

| 역할 | 예시 |
| --- | --- |
| Resource Owner | 사용자 |
| Client | **우리 백엔드 서버** (사용자 정보를 받아가는 쪽) |
| Authorization Server | Google/Kakao의 인증 서버 (로그인, 동의, 토큰 발급) |
| Resource Server | Google/Kakao의 API 서버 (사용자 정보 제공) |

OAuth 자체는 **인가(권한 위임)** 프로토콜이다. "이 사용자가 누구인지" 확인하는 인증 용도로 표준화한 것이 그 위에 얹은 **OpenID Connect(OIDC)**이며, 이때 사용자 정보를 담은 `id_token`(JWT)이 함께 발급된다.

### Authorization Code Grant

서버가 있는 웹 서비스에서 쓰는 표준 방식이다.

```
1. 사용자 → 우리 서버: "Google로 로그인" 클릭
2. 우리 서버 → 브라우저: Google 인증 페이지로 Redirect
     GET https://accounts.google.com/o/oauth2/v2/auth
         ?response_type=code
         &client_id=...
         &redirect_uri=https://api.my.com/login/oauth2/code/google
         &scope=openid profile email
         &state={무작위 값}
         &code_challenge={PKCE}&code_challenge_method=S256
3. 사용자: Google에 로그인하고 정보 제공에 동의
4. Google → 브라우저 → 우리 서버: redirect_uri로 Redirect
     GET /login/oauth2/code/google?code={Authorization Code}&state={1에서 보낸 값}
5. 우리 서버 → Google (서버 간 통신, Back Channel)
     POST /token  code + client_id + client_secret + code_verifier
     ← access_token, id_token (, refresh_token)
6. 우리 서버 → Google: access_token으로 사용자 정보 조회 (또는 id_token 검증)
7. 우리 서버: 회원 조회/가입 → 우리 서비스의 Access/Refresh Token 발급
```

브라우저를 거치는 구간(Front Channel)에서는 **Authorization Code만** 오가고, 실제 토큰은 서버 간 통신(Back Channel)으로 받는다. Code는 한 번만 쓸 수 있고 수명이 짧으며, 토큰으로 바꾸려면 `client_secret`이 필요하므로 Code가 노출되어도 토큰을 얻기 어렵다.

### 보안 파라미터

| 파라미터 | 막는 공격 | 동작 |
| --- | --- | --- |
| `state` | CSRF (공격자의 Code로 피해자를 로그인시키는 공격) | 2단계에서 무작위 값을 저장해 두고, 4단계에서 돌아온 값과 비교 |
| PKCE (`code_verifier`, `code_challenge`) | Authorization Code 탈취 | 2단계에서 무작위 `code_verifier`의 해시(`code_challenge`)를 보내고, 5단계에서 원문을 보내 같은 클라이언트임을 증명 |
| `nonce` (OIDC) | `id_token` 재사용 | 요청 시 보낸 값이 `id_token` 안에 그대로 들어 있는지 확인 |
| `redirect_uri` 정확히 일치 | Code를 공격자 서버로 빼돌림 | 인증 서버에 등록한 URI와 완전히 같아야 함 |

PKCE는 원래 `client_secret`을 안전하게 보관할 수 없는 앱(모바일, SPA)을 위해 만들어졌지만, 현재는 서버 측 클라이언트에도 사용을 권장한다(OAuth 2.1).

### id_token 검증

OIDC의 `id_token`을 사용자 정보로 쓴다면 2장의 JWT 검증을 그대로 적용한다.

- 서명: Google의 JWKS(공개 키)로 검증
- `iss`: `https://accounts.google.com`
- `aud`: **우리 서비스의 `client_id`** (다른 서비스용으로 발급된 `id_token`이 아닌지)
- `exp`, `nonce`

`aud` 검증이 빠지면 다른 앱이 받은 사용자의 `id_token`으로 우리 서비스에 로그인할 수 있다.

### Spring Security OAuth2 Login

`spring-boot-starter-oauth2-client`를 사용하면 위 흐름 중 2~6단계를 Spring Security가 처리한다.

| Filter / 구성 요소 | 처리하는 단계 |
| --- | --- |
| `OAuth2AuthorizationRequestRedirectFilter` | `/oauth2/authorization/{registrationId}` 요청 시 인증 서버로 Redirect (state, PKCE 생성) |
| `OAuth2LoginAuthenticationFilter` | `/login/oauth2/code/{registrationId}`로 돌아온 Code 처리, state 검증, 토큰 교환 |
| `OAuth2UserService` / `OidcUserService` | 사용자 정보 조회. **여기서 회원 조회·가입 로직을 넣음** |
| `AuthenticationSuccessHandler` | 로그인 성공 후 처리. **여기서 우리 서비스의 JWT 발급** |

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
          kakao:
            client-id: ${KAKAO_CLIENT_ID}
            client-secret: ${KAKAO_CLIENT_SECRET}
            client-authentication-method: client_secret_post
            authorization-grant-type: authorization_code
            redirect-uri: "{baseUrl}/login/oauth2/code/{registrationId}"
            scope: profile_nickname
        provider:
          kakao:                                   # Google은 기본 제공, Kakao는 직접 등록
            authorization-uri: https://kauth.kakao.com/oauth/authorize
            token-uri: https://kauth.kakao.com/oauth/token
            user-info-uri: https://kapi.kakao.com/v2/user/me
            user-name-attribute: id
```

```java
http.oauth2Login(oauth2 -> oauth2
        .userInfoEndpoint(userInfo -> userInfo.userService(customOAuth2UserService))
        .successHandler(oAuth2SuccessHandler)
        .failureHandler(oAuth2FailureHandler));
```

```java
@Service
@RequiredArgsConstructor
public class CustomOAuth2UserService extends DefaultOAuth2UserService {

    private final UserRepository userRepository;

    @Override
    @Transactional
    public OAuth2User loadUser(OAuth2UserRequest request) throws OAuth2AuthenticationException {
        OAuth2User oAuth2User = super.loadUser(request);              // 사용자 정보 API 호출
        String provider = request.getClientRegistration().getRegistrationId();   // "kakao"
        String providerId = oAuth2User.getName();                     // user-name-attribute 값

        User user = userRepository.findByProviderAndProviderId(provider, providerId)
                .orElseGet(() -> userRepository.save(User.fromOAuth(provider, oAuth2User)));

        return new CustomOAuth2User(user, oAuth2User.getAttributes());
    }
}
```

Google 로그인에 `openid` scope를 쓰면 OIDC로 처리되어 `OidcUserService`가 호출된다. 이 경우 `oidcUserService(...)`를 따로 지정해야 한다.

### 서버 처리 시 주의점

#### 사용자 식별은 provider + providerId로

이메일은 사용자가 바꿀 수 있고, 제공자에 따라 인증되지 않은 이메일일 수도 있다. 회원은 `(provider, providerId)`(OIDC의 `sub`)를 유니크 키로 식별한다. 같은 이메일이라는 이유만으로 기존 계정에 자동 연결하면, 공격자가 다른 제공자에 피해자의 이메일로 가입해 계정을 탈취할 수 있다.

#### 로그인 결과를 프론트엔드에 전달하는 방법

OAuth 콜백은 브라우저 Redirect로 끝나므로, 우리 서비스의 토큰을 프론트엔드에 넘기는 방법을 정해야 한다.

| 방식 | 문제 |
| --- | --- |
| Redirect URL에 토큰을 붙임 (`?accessToken=...`) | 브라우저 기록, 서버 로그, `Referer` 헤더에 토큰이 남음 |
| Refresh Token을 HttpOnly 쿠키로 설정 후 Redirect, 프론트가 `/auth/refresh`로 Access Token 받음 | 쿠키 도메인 설정이 필요하지만 토큰이 URL에 노출되지 않음 |
| 일회용 코드를 붙여 Redirect, 프론트가 코드로 토큰 교환 | 코드는 짧은 수명·1회 사용으로 관리해야 함 |

Redirect 목적지 URL을 요청 파라미터로 받는다면, 허용 목록에 있는 주소인지 반드시 검사한다. 그렇지 않으면 공격자 사이트로 토큰이나 코드를 보내는 Open Redirect 취약점이 된다.

#### 서버가 여러 대일 때

Spring Security는 2단계에서 만든 인가 요청(state, PKCE 값)을 기본적으로 **세션**에 저장하고 4단계에서 꺼내 비교한다. 서버가 여러 대이고 4단계 요청이 다른 서버로 가면 state를 찾지 못해 로그인이 실패한다. Sticky Session, 세션 공유(Spring Session + Redis), 또는 쿠키 기반 `AuthorizationRequestRepository` 구현 중 하나가 필요하다.

#### 제공자 토큰의 보관

로그인 용도로만 쓴다면 Google/Kakao의 Access Token은 사용자 정보를 조회한 뒤 버린다. 이후 제공자 API(캘린더, 친구 목록 등)를 계속 호출해야 할 때만 저장하며, 이때는 암호화해서 보관한다.

## 5. Backend에서 확인할 점

- `401`(인증 실패)과 `403`(권한 없음)을 구분해서 응답하고, 토큰 만료와 위조를 오류 코드로 구분해 클라이언트가 재발급/재로그인을 판단할 수 있게 한다.
- JWT 검증은 서명만이 아니라 알고리즘 고정, `exp`, `iss`, `aud`, 토큰 용도까지 확인한다. 검증 로직을 직접 만들지 않고 검증된 라이브러리를 쓴다.
- JWT Payload에는 민감한 정보를 넣지 않는다. 서명 키는 코드와 저장소에 두지 않는다.
- Access Token은 짧게, Refresh Token은 서버에 저장하고 Rotation과 재사용 감지를 적용한다.
- CSRF 설정은 "JWT라서"가 아니라 **인증 정보가 쿠키로 자동 전송되는지**를 보고 결정한다.
- OAuth는 `state`·PKCE·`redirect_uri` 일치를 확인하고, 회원은 이메일이 아니라 `(provider, providerId)`로 식별한다.
- 로그인·토큰 재발급 API에는 Rate Limit과 실패 로그를 남겨 무차별 대입과 토큰 재사용을 감지한다.
- 토큰, 비밀번호, Authorization 헤더가 애플리케이션 로그에 남지 않는지 확인한다.
