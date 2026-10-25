# SOP / CORS

## SOP

SOP(Same-Origin Policy, 동일 출처 정책)는 **한 출처에서 불러온 문서나 스크립트가 다른 출처의 리소스와 상호작용하는 것을 제한**하는 브라우저의 보안 정책이다.

### Origin의 정의

Origin(출처)은 **스킴(프로토콜) + 호스트 + 포트** 세 가지로 결정된다. 셋 중 하나라도 다르면 다른 출처다.

`https://example.com` 기준으로 비교하면 다음과 같다.

| URL | 동일 출처? | 이유 |
| --- | --- | --- |
| `https://example.com/user/1` | O | 경로만 다름 |
| `https://example.com:443/` | O | https 기본 포트가 443 |
| `http://example.com` | X | 스킴이 다름 |
| `https://api.example.com` | X | 호스트가 다름 (서브도메인도 다른 출처) |
| `https://example.com:8080` | X | 포트가 다름 |

- 로컬 개발에서 프론트(`localhost:3000`)와 백엔드(`localhost:8080`)는 **포트가 달라 다른 출처**이므로 CORS 에러가 자주 난다.

### SOP가 필요한 이유

브라우저는 요청을 보낼 때 해당 도메인의 쿠키를 자동으로 실어 보낸다. SOP가 없다면 다음과 같은 일이 가능해진다.

1. 사용자가 은행 사이트에 로그인한 상태로 악성 사이트에 접속한다.
2. 악성 사이트의 스크립트가 `fetch("https://bank.com/account")`를 호출한다.
3. 브라우저가 은행 쿠키를 실어 보내므로 요청은 로그인된 사용자로 처리된다.
4. 스크립트가 **응답(계좌 정보)을 읽어서** 공격자 서버로 전송한다.

SOP는 이 중 4번, 즉 **다른 출처 응답을 스크립트가 읽는 것**을 막는다.

- SOP가 모든 교차 출처 요청을 막는 것은 아니다. `<img>`, `<script>`, `<link>`로 다른 출처 리소스를 불러오는 것이나 `<form>` 제출은 허용된다.
- 이 때문에 "요청은 보내지지만 응답을 못 읽는" 상황이 생기고, 요청만으로 피해를 주는 공격이 [CSRF](./xss-csrf.md#csrf)다.

## CORS

CORS(Cross-Origin Resource Sharing)는 SOP의 예외를 허용하는 방법이다. **서버가 HTTP 응답 헤더로 "이 출처는 내 리소스를 읽어도 된다"고 브라우저에게 알려주는 방식**이다.

### CORS 동작 방식

1. 브라우저는 교차 출처 요청에 `Origin` 헤더를 붙여 보낸다.
2. 서버는 응답에 `Access-Control-Allow-Origin` 헤더를 담아 허용할 출처를 알려준다.
3. 브라우저는 요청의 `Origin`과 응답의 `Access-Control-Allow-Origin`을 비교해, 허용되지 않았다면 **응답을 버리고 CORS 에러를 낸다**.

```http
GET /api/posts HTTP/1.1
Host: api.example.com
Origin: https://example.com
```

```http
HTTP/1.1 200 OK
Access-Control-Allow-Origin: https://example.com
Content-Type: application/json
```

중요한 점은 **CORS 검사는 브라우저가 한다**는 것이다.

- 서버는 요청을 정상 처리하고 응답도 보낸다. 브라우저가 그 응답을 JS에 넘겨주지 않을 뿐이다.
- 그래서 Postman, curl, 서버 간 통신에서는 CORS 에러가 발생하지 않는다.
- CORS 에러는 서버 쪽 헤더 설정으로 해결해야 한다. 프론트에서 할 수 있는 건 개발 서버 프록시로 같은 출처처럼 보이게 하는 정도다.

### Simple Request와 Preflight Request

**Simple Request**: 아래 조건을 모두 만족하면 사전 확인 없이 바로 본 요청을 보낸다.

- 메서드가 `GET`, `HEAD`, `POST` 중 하나
- 브라우저가 자동으로 붙이는 헤더 외에는 `Accept`, `Accept-Language`, `Content-Language`, `Content-Type` 등 CORS-safelisted 헤더만 사용
- `Content-Type`이 `application/x-www-form-urlencoded`, `multipart/form-data`, `text/plain` 중 하나

이 조건은 "HTML `<form>`으로도 원래 보낼 수 있던 요청"이다. 이미 SOP 이전부터 가능했던 요청이라 추가 확인을 하지 않는다.

**Preflight Request**: 조건을 벗어나면 브라우저가 본 요청 전에 `OPTIONS` 메서드로 "이런 요청 보내도 돼?"를 먼저 물어본다.

실무에서는 `Content-Type: application/json`이나 `Authorization` 헤더를 쓰는 순간 Preflight 대상이 되므로, 대부분의 API 요청은 Preflight를 거친다.

```http
OPTIONS /api/posts HTTP/1.1
Origin: https://example.com
Access-Control-Request-Method: PUT
Access-Control-Request-Headers: Content-Type, Authorization
```

```http
HTTP/1.1 204 No Content
Access-Control-Allow-Origin: https://example.com
Access-Control-Allow-Methods: GET, POST, PUT, DELETE
Access-Control-Allow-Headers: Content-Type, Authorization
Access-Control-Max-Age: 3600
```

- `Access-Control-Max-Age`: Preflight 결과를 캐시하는 시간(초). 이 시간 동안은 같은 요청에 Preflight를 다시 보내지 않는다.
- Preflight가 통과해야 본 요청이 전송된다. 실패하면 **본 요청은 아예 서버에 가지 않는다.**
- 흔한 실수: 서버의 인증 필터가 `OPTIONS` 요청까지 막아서 401/403이 나는 경우. Preflight에는 `Authorization` 헤더가 실리지 않으므로 `OPTIONS`는 인증 없이 통과시켜야 한다.

### Credentialed Request

기본적으로 브라우저는 교차 출처 요청에 **쿠키를 보내지 않는다**. 쿠키나 HTTP 인증 정보를 함께 보내려면 클라이언트와 서버 양쪽에서 설정해야 한다.

```js
// fetch
fetch("https://api.example.com/me", { credentials: "include" });

// axios
axios.get("https://api.example.com/me", { withCredentials: true });
```

```http
HTTP/1.1 200 OK
Access-Control-Allow-Origin: https://example.com
Access-Control-Allow-Credentials: true
```

- 서버는 `Access-Control-Allow-Credentials: true`를 응답해야 한다.
- 이때 `Access-Control-Allow-Origin`에 **와일드카드 `*`를 쓸 수 없고**, 정확한 출처를 명시해야 한다.
- 쿠키 자체도 `SameSite=None; Secure`여야 교차 사이트 요청에 실린다. ([쿠키 옵션](./auth-session-jwt.md#쿠키-보안-옵션-httponly--secure--samesite) 참고)

Spring에서는 보통 다음과 같이 설정한다.

```java
@Bean
CorsConfigurationSource corsConfigurationSource() {
    CorsConfiguration config = new CorsConfiguration();
    config.setAllowedOrigins(List.of("https://example.com"));
    config.setAllowedMethods(List.of("GET", "POST", "PUT", "DELETE", "OPTIONS"));
    config.setAllowedHeaders(List.of("*"));
    config.setAllowCredentials(true);
    config.setMaxAge(3600L);

    UrlBasedCorsConfigurationSource source = new UrlBasedCorsConfigurationSource();
    source.registerCorsConfiguration("/**", config);
    return source;
}
```

Spring Security를 쓴다면 `http.cors(Customizer.withDefaults())`로 이 설정을 Security 필터 체인에 연결해야 한다. 그렇지 않으면 Preflight가 Security 필터에서 먼저 막힐 수 있다.
