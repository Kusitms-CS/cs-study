# 12주차 - Rendering Architecture

## 1. SPA / MPA

### 구조

```text
MPA (Multi Page Application)
/home  클릭 → 서버에 /about 요청 → 새 HTML 전체 수신 → 페이지 전체 새로 로드
/about 클릭 → 서버에 /posts 요청 → 새 HTML 전체 수신 → 페이지 전체 새로 로드

SPA (Single Page Application)
처음 한 번: index.html + JS 번들 수신
/home → /about 클릭: JS가 URL만 바꾸고(History API) 필요한 데이터만 받아 화면 일부를 교체
```

### 특징

- MPA는 페이지를 이동할 때마다 서버가 새 HTML을 보내고 브라우저가 페이지 전체를 다시 그림. 전통적인 서버 렌더링 방식
- SPA는 HTML 하나를 받은 뒤 JavaScript가 라우팅과 화면 전환을 직접 처리함. 페이지 전환이 앱처럼 부드럽고 이동할 때 필요한 데이터만 받음
- SPA는 첫 진입 때 큰 JS 번들을 받아 실행해야 화면이 보여서 초기 로딩이 느릴 수 있고, JS 실행 전 HTML이 비어 있어 SEO에 불리할 수 있음
- SPA는 새로고침 없이 URL을 바꾸기 때문에 서버가 `/about` 같은 경로 요청에도 `index.html`을 돌려주도록 설정해야 함

### 정리

- MPA는 "페이지마다 서버가 새로 만들어 주는 방식", SPA는 "한 페이지 안에서 JS가 화면을 바꿔 끼우는 방식"
- 요즘 Next.js 같은 프레임워크는 첫 페이지는 서버에서 HTML로 보내고, 이후 이동은 SPA처럼 처리해서 둘의 장점을 함께 씀

---

## 2. CSR (Client-Side Rendering)

### 구조

```text
Browser                         Server
  │ ─── GET / ─────────────────> │
  │ <── <div id="root"></div> ── │   ← 거의 빈 HTML
  │ ─── GET /bundle.js ────────> │
  │ <── JS 번들 ──────────────── │
  │ JS 실행 → API 요청 → 데이터로 화면 생성
  ▼
화면 표시 (이때부터 상호작용 가능)
```

### 특징

- 브라우저가 JavaScript를 실행해서 화면을 만드는 방식. Vite + React로 만든 기본 SPA가 여기에 해당함
- 서버는 정적 파일만 보내면 되어 구조가 단순하고, 한 번 로딩된 뒤에는 화면 전환이 빠름
- JS 번들 다운로드 → 실행 → 데이터 요청이 끝나야 내용이 보여서 첫 화면(FCP, LCP)이 늦어지기 쉬움(10주차 Critical Rendering Path와 연결)
- 검색 엔진 크롤러나 링크 미리보기(Open Graph)가 JS를 실행하지 않으면 빈 페이지로 인식할 수 있음

### 정리

- CSR은 "빈 HTML을 받고 브라우저가 화면을 다 만드는 방식"
- 로그인 후에만 쓰는 대시보드나 관리자 페이지처럼 SEO가 필요 없고 상호작용이 많은 서비스에 잘 맞음

---

## 3. SSR (Server-Side Rendering)

### 구조

```text
Browser                         Server
  │ ─── GET /posts/1 ──────────> │  요청마다 데이터 조회 → HTML 생성
  │ <── 내용이 채워진 HTML ───── │
  │ 화면 바로 표시 (아직 클릭 등은 동작 안 함)
  │ ─── GET /bundle.js ────────> │
  │ JS 실행 → Hydration
  ▼
상호작용 가능
```

### 특징

- 요청이 올 때마다 서버가 데이터를 조회해서 완성된 HTML을 만들어 보내는 방식
- 내용이 들어 있는 HTML이 바로 오기 때문에 첫 화면이 빨리 보이고 SEO에 유리함
- 요청마다 서버가 렌더링해야 해서 서버 부하가 커지고, 데이터 조회가 느리면 HTML 응답 자체가 늦어짐(TTFB 증가)
- 화면은 보이지만 JS가 실행되어 Hydration이 끝나기 전까지는 버튼을 눌러도 반응하지 않는 구간이 있음
- React 18부터는 Streaming SSR로 준비된 부분의 HTML부터 먼저 보내고, 느린 부분은 `<Suspense>`의 fallback을 보여줬다가 나중에 채울 수 있음

### 정리

- SSR은 "요청마다 서버가 완성된 HTML을 만들어 주는 방식"
- 사용자마다 다르거나 자주 바뀌면서 SEO도 필요한 페이지(상품 상세, 검색 결과)에 잘 맞음

---

## 4. SSG (Static Site Generation) / ISR

### 구조

```text
SSG
빌드 시점: 데이터 조회 → 모든 페이지 HTML 미리 생성 → CDN에 배포
요청 시점: CDN이 만들어 둔 HTML을 그대로 응답 (서버 렌더링 없음)

ISR (Incremental Static Regeneration)
요청 시점: 만들어 둔 HTML 응답
          + 정해진 시간(revalidate)이 지났으면 뒤에서 새로 생성해서 교체
```

### 특징

- 빌드할 때 HTML을 미리 만들어 두는 방식. CDN에서 바로 응답하므로 가장 빠르고 서버 부하도 거의 없음
- 빌드 이후 데이터가 바뀌면 다시 빌드하기 전까지 반영되지 않음. 페이지가 많으면 빌드 시간도 길어짐
- ISR은 SSG에 주기적 갱신을 더한 방식. 처음엔 이전 HTML을 보여주고 뒤에서 새로 만들어 교체함(6주차 stale-while-revalidate와 같은 아이디어)

| 방식 | HTML 생성 시점 | 장점 | 단점 | 어울리는 페이지 |
| --- | --- | --- | --- | --- |
| CSR | 브라우저에서 실행할 때 | 서버 단순, 전환 빠름 | 첫 화면 느림, SEO 불리 | 대시보드, 관리자 |
| SSR | 요청마다 서버에서 | 최신 데이터 + SEO | 서버 부하, TTFB 증가 | 상품 상세, 검색 결과 |
| SSG | 빌드할 때 | 가장 빠름, 서버 부하 없음 | 데이터 갱신 어려움 | 블로그, 문서, 랜딩 |
| ISR | 빌드 후 주기적으로 | 빠름 + 주기적 갱신 | 잠깐 이전 데이터가 보임 | 뉴스, 상품 목록 |

### 정리

- SSG는 "미리 만들어 둔 HTML을 나눠 주는 방식", ISR은 "미리 만들어 두되 가끔 새로 고쳐 놓는 방식"
- 한 서비스 안에서도 페이지마다 다른 방식을 고르는 것이 일반적임

---

## 5. Hydration

### 구조

```text
서버에서 받은 HTML (보이지만 동작하지 않음)
<button>좋아요 0</button>
        │
        ▼  JS 번들 로드 후 React가 같은 컴포넌트를 다시 실행해서
           만든 결과와 기존 HTML을 맞춰 보고, 이벤트 핸들러를 연결
<button onClick={like}>좋아요 0</button>   ← 이제 클릭 가능
```

```jsx
// CSR: 빈 root에 처음부터 DOM 생성
createRoot(document.getElementById("root")).render(<App />);

// SSR/SSG: 서버가 만든 DOM을 재사용하면서 이벤트만 연결
hydrateRoot(document.getElementById("root"), <App />);
```

### 특징

- 서버가 만든 정적인 HTML에 JavaScript를 연결해서 상호작용이 가능한 앱으로 만드는 과정. DOM을 새로 만들지 않고 이미 있는 DOM을 재사용함
- 클라이언트에서 처음 렌더링한 결과가 서버 HTML과 같아야 함. 다르면 Hydration Mismatch가 발생함(딥다이브와 연결)
- 페이지 전체 컴포넌트 코드를 내려받아 다시 실행해야 하므로, 화면은 빨리 보여도 JS가 크면 상호작용 가능 시점이 늦어짐
- React 18부터는 `<Suspense>`로 나눈 영역별로 따로 Hydration하고, 사용자가 클릭한 영역을 먼저 처리함(Selective Hydration)

### 정리

- Hydration은 "보이기만 하는 HTML에 물을 부어 움직이게 만드는 것"
- SSR이 빨라 보이는 건 화면이 먼저 보이기 때문이고, 실제로 쓸 수 있는 시점은 Hydration이 끝난 뒤라는 점이 핵심

---

## 6. Server Component / Client Component

### 구조

```text
app/posts/[id]/page.tsx  (Server Component, 기본값)
 ├─ DB에서 게시글 조회 → 서버에서 렌더링, 코드는 번들에 포함되지 않음
 ├─ <PostBody />          (Server Component)
 └─ <LikeButton />        ("use client" → Client Component)
                            서버에서 HTML로 미리 렌더링 + 브라우저에서 Hydration
```

```tsx
// app/posts/[id]/page.tsx — Server Component
export default async function PostPage({ params }) {
  const { id } = await params;
  const post = await db.post.findUnique({ where: { id } }); // 서버에서 바로 DB 접근
  return (
    <article>
      <h1>{post.title}</h1>
      <LikeButton postId={post.id} initialCount={post.likes} />
    </article>
  );
}

// LikeButton.tsx — Client Component
"use client";
export function LikeButton({ postId, initialCount }) {
  const [count, setCount] = useState(initialCount);
  return <button onClick={() => setCount((c) => c + 1)}>좋아요 {count}</button>;
}
```

### 특징

- React Server Component는 서버에서만 실행되는 컴포넌트. Next.js App Router에서는 기본값
  - `async`로 만들어 DB나 파일을 직접 조회할 수 있고, API 키 같은 비밀 값을 안전하게 쓸 수 있음
  - 컴포넌트 코드가 브라우저 JS 번들에 포함되지 않아 번들 크기가 줄고 Hydration할 양도 줄어듦
  - `useState`, `useEffect`, 이벤트 핸들러, `window` 같은 브라우저 API는 쓸 수 없음
- Client Component는 파일 맨 위에 `"use client"`를 붙인 컴포넌트. state, 이벤트, 브라우저 API를 쓸 수 있음
- **Client Component도 서버에서 HTML로 먼저 렌더링(SSR)된 뒤 브라우저에서 Hydration됨.** "Client Component = CSR"이 아님
- `"use client"`는 경계를 표시하는 것이라, 그 파일이 import하는 컴포넌트도 모두 Client Component가 됨. 그래서 경계는 상호작용이 필요한 가장 작은 단위(버튼, 폼)에 두는 것이 좋음
- Server Component에서 Client Component로 넘기는 props는 직렬화할 수 있어야 함(문자열, 숫자, 객체 등. 일반 함수는 불가)
- Client Component는 Server Component를 import할 수 없지만, `children`으로 받아서 렌더링할 수는 있음(11주차 children 패턴과 연결)
- SSR과 Server Component는 다른 개념. SSR은 "HTML을 서버에서 만드는 것", Server Component는 "어떤 컴포넌트를 서버에서만 실행할지 나누는 것"

### 정리

- 데이터 조회와 정적인 화면은 Server Component, 상호작용이 필요한 작은 부분만 Client Component로 나누는 것이 기본 전략
- Server Component의 핵심 이점은 "브라우저로 보내는 JS와 Hydration할 양을 줄이는 것"

---

## 7. 딥다이브: Hydration Mismatch

### 왜 생길까?

Hydration은 서버가 만든 HTML과 브라우저에서 처음 렌더링한 결과가 같다고 가정하고 DOM을 재사용한다. 그런데 서버와 브라우저에서 다른 값을 만드는 코드가 있으면 둘이 어긋나고, React는 경고를 띄운 뒤 해당 부분을 클라이언트에서 다시 렌더링한다. 화면이 깜빡이거나 SSR의 이점이 사라지고, 경우에 따라 잘못된 내용이 남을 수 있다.

```text
서버 (UTC, 서버 시각)              브라우저 (KST, 사용자 시각)
<p>2026. 11. 22. 오전 3:00:00</p>  vs  <p>2026. 11. 22. 오후 12:00:00</p>
                     → 텍스트가 달라 Hydration Mismatch
```

### 자주 나오는 원인

| 원인 | 예시 |
| --- | --- |
| 실행할 때마다 달라지는 값 | `Date.now()`, `new Date().toLocaleString()`, `Math.random()` |
| 브라우저에만 있는 값으로 분기 | `typeof window !== "undefined"`, `localStorage`, `window.innerWidth` |
| 서버와 브라우저의 환경 차이 | 시간대, 언어 설정에 따라 다른 날짜·숫자 포맷 |
| 잘못된 HTML 중첩 | `<p>` 안에 `<div>`, `<a>` 안에 `<a>` → 브라우저가 HTML을 고쳐서 파싱하면서 구조가 달라짐 |
| 브라우저 확장 프로그램 | 번역·다크모드 확장이 Hydration 전에 DOM을 바꿈 |

### 해결 방법

```tsx
// 1. 브라우저에서만 알 수 있는 값은 Hydration 이후에 반영
"use client";
function LocalTime({ iso }: { iso: string }) {
  const [text, setText] = useState(""); // 서버와 첫 렌더링 결과를 같게 맞춤

  useEffect(() => {
    setText(new Date(iso).toLocaleString()); // Effect는 브라우저에서 Hydration 후에만 실행됨
  }, [iso]);

  return <time dateTime={iso}>{text}</time>;
}
```

```tsx
// 2. 서버에서 값을 확정해서 props로 넘기기
//    시간대·포맷을 서버에서 정해 문자열로 넘기면 서버와 브라우저가 같은 값을 씀
<PostDate label={formatInKST(post.createdAt)} />

// 3. 브라우저 전용 라이브러리(지도, 에디터)는 서버 렌더링에서 제외 (Next.js, Client Component 안에서)
const Map = dynamic(() => import("./Map"), { ssr: false });

// 4. 피할 수 없는 텍스트 차이만 경고 숨기기 (해당 요소 한 단계에만 적용, 남용 금지)
<time suppressHydrationWarning>{new Date().toLocaleTimeString()}</time>;
```

- 1번처럼 `useEffect`로 미루면 첫 화면에는 빈 값이 잠깐 보이므로, 자리를 미리 잡아 두거나 스켈레톤을 보여주는 것이 좋다.
- `typeof window`로 렌더링 결과 자체를 바꾸는 코드는 서버에서는 `false`, 브라우저에서는 `true`가 되어 Mismatch의 대표 원인이다. 브라우저 전용 처리는 렌더링 중이 아니라 `useEffect` 안에서 한다.
- `suppressHydrationWarning`은 경고만 숨기는 것이라 근본적인 해결이 아니다. 타임스탬프처럼 달라도 괜찮은 텍스트에만 쓴다.

### 정리

- Hydration Mismatch는 "서버와 브라우저의 첫 렌더링 결과가 달라서" 생긴다. 렌더링 중에는 양쪽에서 같은 값을 만들어야 한다.
- 브라우저에서만 알 수 있는 값은 Hydration 이후(`useEffect`)에 반영하고, 시간대처럼 환경에 따라 다른 값은 서버에서 확정해서 넘긴다.
