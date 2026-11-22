# 12주차 - Rendering Architecture

## 1. SPA / MPA

- 웹 애플리케이션이 **페이지를 어떻게 구성하고 전환하는지**에 따른 구조 구분
- **MPA(Multi Page Application):** 페이지를 이동할 때마다 **서버에서 새 HTML 문서를 받아** 화면 전체를 다시 로드
- **SPA(Single Page Application):** **최초에 HTML 한 장과 JS 번들을 받은 뒤**, 페이지 이동 시 JS가 필요한 데이터만 받아 **화면의 일부만 교체**

### 📍 동작 방식 비교

```
 MPA
 /home  ──request──▶ Server ──▶ full HTML ──▶ full page reload
 /about ──request──▶ Server ──▶ full HTML ──▶ full page reload

 SPA
 first visit ──▶ Server ──▶ index.html + bundle.js
 /about      ──▶ JS router changes URL (History API)
             ──▶ fetch JSON from API ──▶ update only the changed part of DOM
```

### 📍 SPA vs MPA

| 구분             | MPA                                                  | SPA                                                          |
| ---------------- | ---------------------------------------------------- | ------------------------------------------------------------ |
| 페이지 전환      | 매번 **새 HTML 요청**, 화면 전체 새로고침 (깜빡임)   | JS가 **화면 일부만 교체**, 앱처럼 부드러운 전환              |
| 초기 로딩        | 해당 페이지에 필요한 것만 받아 **빠름**              | 앱 전체 JS 번들을 받아야 해서 **느릴 수 있음**               |
| 이후 이동        | 매번 서버 왕복이 필요해 상대적으로 느림              | 필요한 데이터만 받아 **빠름**                                |
| SEO              | 완성된 HTML을 받으므로 **유리**                      | 초기 HTML이 비어 있어 **불리** (SSR/SSG로 보완)              |
| 상태 유지        | 페이지 이동 시 JS 상태 **초기화**                    | 페이지 이동해도 **상태 유지**                                |
| 서버 역할        | HTML 렌더링까지 담당                                 | 주로 **API(JSON) 제공**, 프론트/백 분리가 쉬움               |
| 예시             | 전통적인 JSP, PHP, Django 템플릿 사이트              | React, Vue, Angular로 만든 앱 (CRA, Vite)                    |

### 📍 SPA 라우팅의 원리

- 실제로 서버에 새 페이지를 요청하지 않고, **History API(`pushState`, `popstate`)** 로 URL만 바꾼 뒤 JS가 해당 경로의 컴포넌트를 렌더링함

```js
// 링크 클릭 시 새로고침 없이 URL만 변경
history.pushState({}, "", "/about");
render("/about");

// 뒤로가기/앞으로가기 처리
window.addEventListener("popstate", () => render(location.pathname));
```

- 사용자가 `/about`에서 **새로고침하면 서버에 `/about`을 직접 요청**하므로, 서버(Nginx, S3 등)가 **모든 경로에 `index.html`을 응답**하도록 설정해야 함 (안 하면 404)

```nginx
location / {
  try_files $uri $uri/ /index.html;
}
```

### 💡 SPA의 단점과 보완 방법

**⇒ SPA는 처음에 앱 전체 JS를 받아 실행해야 화면이 보여서 초기 로딩이 느리고, 초기 HTML이 거의 비어 있어 JS를 실행하지 않는 크롤러나 SNS 링크 미리보기에서 내용이 보이지 않아 SEO에 불리함. 또한 페이지 이동이 JS로 처리되므로 스크롤 위치 복원, 포커스 관리 같은 브라우저 기본 기능을 직접 신경 써야 함. 이를 보완하기 위해 코드 스플리팅으로 초기 번들을 줄이고, SSR·SSG로 완성된 HTML을 먼저 내려준 뒤 Hydration으로 SPA처럼 동작하게 하는 방식(Next.js, Remix 등)을 사용함**

---

## 2. CSR / SSR / SSG

- **HTML을 언제, 어디서 만드느냐**에 따른 렌더링 방식 구분
- **CSR:** 브라우저(클라이언트)에서 / **SSR:** 요청이 올 때마다 서버에서 / **SSG:** 빌드할 때 미리

### 📍 CSR (Client Side Rendering)

- 서버는 **거의 빈 HTML과 JS 번들**만 보내고, **브라우저가 JS를 실행해 화면을 그림**

```html
<!-- 서버가 보내는 HTML -->
<body>
  <div id="root"></div>
  <script src="/bundle.js"></script>
</body>
```

```
 Browser ──▶ empty HTML ──▶ download JS ──▶ run JS ──▶ fetch data ──▶ render
             (blank page)                                             (FCP, interactive)
```

### 📍 SSR (Server Side Rendering)

- **요청이 올 때마다 서버가 데이터를 조회해 완성된 HTML을 만들어** 응답하고, 브라우저는 이후 JS를 받아 **Hydration**

```
 Browser ──▶ Server fetches data + renders HTML ──▶ full HTML ──▶ download JS ──▶ hydrate
                                                    (FCP)                         (interactive)
```

### 📍 SSG (Static Site Generation)

- **빌드 시점에 HTML을 미리 생성**해두고, 요청이 오면 **CDN에서 정적 파일을 그대로 응답**

```
 Build time : fetch data ──▶ render HTML ──▶ deploy to CDN
 Request    : Browser ──▶ CDN ──▶ prebuilt HTML (very fast) ──▶ hydrate
```

- 데이터가 바뀌면 **다시 빌드**해야 하는 한계가 있어, 이를 보완한 **ISR(Incremental Static Regeneration)** 이 있음 → 정해진 시간이 지나면 다음 요청 때 **백그라운드에서 해당 페이지만 다시 생성**

### 📍 렌더링 방식 비교

| 구분             | CSR                               | SSR                                         | SSG                                   | ISR                                  |
| ---------------- | --------------------------------- | ------------------------------------------- | ------------------------------------- | ------------------------------------ |
| HTML 생성 시점   | 브라우저에서 JS 실행 후           | **요청마다** 서버에서                       | **빌드 시**                           | 빌드 시 + **주기적으로 재생성**      |
| 초기 로딩 (FCP)  | 느림 (빈 화면 → JS 실행)          | 빠름                                        | **매우 빠름** (CDN)                   | 매우 빠름                            |
| TTFB             | 빠름 (빈 HTML)                    | **느릴 수 있음** (서버에서 렌더링 대기)     | 매우 빠름                             | 매우 빠름                            |
| SEO              | 불리                              | **유리**                                    | **유리**                              | 유리                                 |
| 데이터 최신성    | 항상 최신 (클라이언트 요청)       | **항상 최신**                               | 빌드 시점 데이터                      | 재생성 주기만큼 지연                 |
| 서버 부하        | 낮음 (정적 파일)                  | **높음** (요청마다 렌더링)                  | 낮음                                  | 낮음                                 |
| 적합한 페이지    | 로그인 후 대시보드, 관리자 페이지 | 개인화된 페이지, 실시간 데이터, 검색 결과   | 블로그, 문서, 랜딩·마케팅 페이지      | 상품 목록, 뉴스 등 가끔 바뀌는 콘텐츠 |

### 📍 Next.js에서의 선택 (App Router 기준)

```tsx
// SSG: 기본적으로 빌드 시 정적으로 생성
export default async function Page() {
  const posts = await fetch("https://api.example.com/posts", { cache: "force-cache" }).then((r) => r.json());
  return <PostList posts={posts} />;
}

// ISR: 60초마다 재생성
export const revalidate = 60;

// SSR: 요청마다 새로 렌더링
const data = await fetch("https://api.example.com/me", { cache: "no-store" });
```

- 실무에서는 하나만 고르는 것이 아니라 **페이지(또는 컴포넌트)마다 특성에 맞게 섞어서** 사용함

### 💡 CSR과 SSR의 차이와 각각 언제 사용하나요?

**⇒ CSR은 빈 HTML을 받은 뒤 브라우저가 JS를 실행해 화면을 만들기 때문에 초기 로딩이 느리고 SEO에 불리하지만, 서버 부하가 적고 한 번 로드된 뒤에는 페이지 전환이 빠름. SSR은 서버가 요청마다 완성된 HTML을 만들어 보내므로 첫 화면이 빠르게 보이고 검색 엔진이 내용을 바로 읽을 수 있지만, 요청마다 서버에서 렌더링해야 해서 서버 부하와 TTFB가 늘어남. 그래서 검색 노출이 중요한 상품·콘텐츠 페이지는 SSR이나 SSG를, 로그인 후 사용하는 대시보드나 관리자 페이지처럼 SEO가 필요 없고 상호작용이 많은 화면은 CSR을 사용함**

### 💡 SSR을 쓰면 무조건 빠른가요?

**⇒ 첫 화면이 보이는 시점(FCP)은 빨라지지만, 서버가 데이터를 모두 가져와 HTML을 만들 때까지 응답을 시작하지 못해 TTFB가 느려질 수 있고, 화면이 보여도 JS를 받아 Hydration이 끝나기 전까지는 버튼이 동작하지 않는 구간이 생김. 또한 트래픽이 많으면 서버 렌더링 비용이 커짐. 이를 보완하기 위해 바뀌지 않는 페이지는 SSG/ISR로 CDN 캐싱을 하고, Streaming SSR로 준비된 부분부터 HTML을 먼저 보내며, Server Component로 클라이언트에 보내는 JS 자체를 줄이는 방식을 함께 사용함**

---

## 3. Hydration

- **서버에서 만든 정적인 HTML에, 클라이언트의 JS가 이벤트 핸들러와 상태를 연결해서 상호작용 가능한 앱으로 만드는 과정**
- "마른 HTML에 물(JS)을 부어 살아 움직이게 한다"는 의미
- SSR/SSG로 받은 HTML은 **보이기만 하고 클릭해도 반응하지 않는 상태**이고, Hydration이 끝나야 비로소 동작함

### 📍 Hydration 과정

```
 1. Server : render components ──▶ HTML string
 2. Browser: show HTML immediately          (visible, but not interactive)
 3. Browser: download + run JS bundle
 4. React  : render components again in memory
             compare with existing DOM, attach event handlers   (Hydration)
 5. Done   : page is interactive
```

```jsx
// CSR: 빈 div에 DOM을 새로 만들어 렌더링
createRoot(document.getElementById("root")).render(<App />);

// SSR: 이미 있는 HTML을 재사용하며 이벤트만 연결
hydrateRoot(document.getElementById("root"), <App />);
```

- Hydration은 DOM을 새로 만드는 것이 아니라 **기존 DOM을 재사용**하고, 서버와 클라이언트의 렌더링 결과가 **같다고 가정**함

### 📍 Hydration Mismatch

- 서버에서 렌더링한 HTML과 클라이언트에서 처음 렌더링한 결과가 **다르면** 에러(경고)가 발생하고, React는 해당 부분을 **클라이언트에서 다시 렌더링**함 → 성능 저하, 화면 깜빡임

| 원인                                      | 예시                                                        |
| ----------------------------------------- | ----------------------------------------------------------- |
| 실행할 때마다 값이 달라지는 코드          | `Date.now()`, `new Date().toLocaleString()`, `Math.random()` |
| 브라우저에만 있는 API를 렌더링에 사용     | `window.innerWidth`, `localStorage`, `navigator`            |
| 잘못된 HTML 중첩                          | `<p>` 안에 `<div>`, `<a>` 안에 `<a>` → 브라우저가 구조를 보정 |
| 브라우저 확장 프로그램                    | 확장 프로그램이 DOM에 요소를 삽입                           |

```jsx
// Bad: 서버와 클라이언트의 시간이 달라 mismatch
function Clock() {
  return <span>{new Date().toLocaleTimeString()}</span>;
}

// Good: 브라우저 전용 값은 Hydration 이후(useEffect)에 반영
function Clock() {
  const [time, setTime] = useState(null);
  useEffect(() => setTime(new Date().toLocaleTimeString()), []);
  return <span>{time ?? "--:--"}</span>;
}
```

### 📍 Hydration의 비용과 개선 방법

- 화면은 보이지만 **Hydration이 끝날 때까지 클릭해도 반응하지 않는 구간**이 생김 (Uncanny Valley)
- 정적인 부분까지 포함해 **페이지 전체 컴포넌트의 JS를 받아서 다시 실행**해야 하므로, 페이지가 클수록 비용이 큼

| 방법                              | 설명                                                                                              |
| --------------------------------- | ------------------------------------------------------------------------------------------------- |
| Selective Hydration (React 18)    | `<Suspense>`로 나눈 영역별로 **준비된 곳부터 Hydration**, 사용자가 클릭한 영역을 **우선** 처리    |
| Progressive Hydration             | 화면에 보이거나 상호작용이 필요해지는 시점에 **점진적으로** Hydration                             |
| Islands Architecture (Astro)      | 페이지는 정적 HTML로 두고, **상호작용이 필요한 컴포넌트(섬)만** JS를 보내 Hydration              |
| Server Component (React)          | 상호작용이 없는 컴포넌트는 **JS를 아예 보내지 않아** Hydration 대상에서 제외                     |
| Resumability (Qwik)               | Hydration으로 다시 실행하지 않고, 서버의 실행 상태를 HTML에 담아 **필요할 때 이어서 실행**       |

### 💡 Hydration이란 무엇이고 왜 필요한가요?

**⇒ SSR이나 SSG로 받은 HTML은 서버에서 문자열로 만들어진 것이라 화면에 보이기만 할 뿐 이벤트 핸들러나 state가 연결되어 있지 않음. Hydration은 브라우저가 JS를 받아 같은 컴포넌트를 메모리에서 다시 렌더링하고, 이미 있는 DOM과 맞춰보면서 이벤트 핸들러와 상태를 연결해 상호작용 가능한 React 앱으로 만드는 과정임. 덕분에 사용자는 서버 렌더링으로 빠르게 첫 화면을 보면서도, 이후에는 SPA처럼 동작하는 앱을 사용할 수 있음**

### 💡 Hydration Mismatch는 왜 발생하고 어떻게 해결하나요?

**⇒ Hydration은 서버가 만든 HTML과 클라이언트의 첫 렌더링 결과가 같다고 가정하고 DOM을 재사용하는데, 현재 시간·랜덤 값처럼 실행할 때마다 달라지는 값이나 `window`, `localStorage`처럼 브라우저에만 있는 값을 렌더링에 사용하면 결과가 달라져 mismatch가 발생함. 해결하려면 브라우저 전용 값은 `useEffect`에서 Hydration 이후에 반영하거나, 해당 컴포넌트를 클라이언트에서만 렌더링하도록 분리(`next/dynamic`의 `ssr: false`)함. 날짜처럼 의도적으로 달라질 수밖에 없는 작은 텍스트는 `suppressHydrationWarning`으로 경고를 끄기도 함**

---

## 4. Server / Client Component

- **React Server Component(RSC):** **서버에서만 실행**되고, 렌더링 결과만 클라이언트로 보내며 **컴포넌트의 JS 코드는 번들에 포함되지 않는** 컴포넌트
- **Client Component:** 기존 React 컴포넌트처럼 **서버에서 HTML로 미리 렌더링(SSR)된 뒤 브라우저에서 Hydration되어** 상호작용을 담당
- Next.js App Router에서는 **기본이 Server Component**이고, 파일 맨 위에 **`"use client"`** 를 적으면 Client Component가 됨

### 📍 Server Component vs Client Component

| 구분                     | Server Component                                         | Client Component (`"use client"`)                          |
| ------------------------ | -------------------------------------------------------- | ---------------------------------------------------------- |
| 실행 위치                | **서버에서만**                                           | 서버(SSR) + **브라우저**                                   |
| JS 번들                  | **포함되지 않음** (번들 크기 0)                          | 포함됨                                                     |
| Hydration                | **필요 없음**                                            | 필요                                                       |
| 데이터 접근              | **DB, 파일, 비밀 키에 직접 접근** 가능, `async/await` 컴포넌트 | API를 통해 접근                                      |
| state / Effect           | `useState`, `useEffect` **사용 불가**                    | 사용 가능                                                  |
| 이벤트 핸들러            | `onClick` 등 **사용 불가**                               | 사용 가능                                                  |
| 브라우저 API             | 사용 불가                                                | `window`, `localStorage` 사용 가능                         |
| 용도                     | 데이터 조회, 정적인 UI, 무거운 라이브러리 사용 (마크다운 파싱 등) | 버튼, 폼, 모달 등 **상호작용이 필요한 UI**         |

```tsx
// app/posts/page.tsx - Server Component (기본값)
import { db } from "@/lib/db";
import LikeButton from "./LikeButton";

export default async function PostsPage() {
  const posts = await db.post.findMany(); // 서버에서 DB 직접 조회, 코드는 클라이언트로 가지 않음
  return (
    <ul>
      {posts.map((post) => (
        <li key={post.id}>
          {post.title}
          <LikeButton postId={post.id} /> {/* 상호작용이 필요한 부분만 Client Component */}
        </li>
      ))}
    </ul>
  );
}
```

```tsx
// app/posts/LikeButton.tsx - Client Component
"use client";
import { useState } from "react";

export default function LikeButton({ postId }: { postId: number }) {
  const [liked, setLiked] = useState(false);
  return <button onClick={() => setLiked((v) => !v)}>{liked ? "♥" : "♡"}</button>;
}
```

### 📍 동작 방식 (RSC Payload)

```
 Server
   Server Components run ──▶ RSC Payload (serialized UI tree)
                             + references to Client Component JS
   SSR                   ──▶ initial HTML

 Browser
   show HTML ──▶ read RSC Payload ──▶ hydrate only Client Components
```

- Server Component의 결과는 HTML이 아니라 **RSC Payload라는 직렬화된 UI 트리**로 전달되며, 이 안에 Client Component가 들어갈 **자리와 JS 파일 참조**가 표시됨
- 페이지 이동 시에도 HTML 전체가 아니라 RSC Payload만 받아 **클라이언트 state를 유지한 채** 화면을 갱신할 수 있음
- **Server Component ≠ SSR:** SSR은 "HTML을 서버에서 미리 만드는 것"이고(Client Component도 SSR됨), Server Component는 "**클라이언트에서는 아예 실행되지 않는 컴포넌트**"

### 📍 컴포넌트 경계 규칙

| 규칙                                                       | 설명                                                                                            |
| ---------------------------------------------------------- | ----------------------------------------------------------------------------------------------- |
| `"use client"`는 **경계**를 만듦                           | 해당 파일과 그 파일이 **import하는 모든 컴포넌트가 클라이언트 번들**에 포함됨                  |
| Client가 Server Component를 **import할 수 없음**           | 대신 **`children`이나 props로 Server Component를 전달**하면 가능                               |
| Server → Client로 넘기는 props는 **직렬화 가능**해야 함    | 문자열, 숫자, 배열, 일반 객체, Promise 등은 가능 / **함수, 클래스 인스턴스는 불가**             |
| `"use client"`는 **최대한 말단(leaf)에**                   | 페이지 전체가 아니라 **상호작용이 필요한 작은 컴포넌트에만** 붙여야 JS 번들이 작아짐           |

```tsx
// Client Component 안에 Server Component를 넣는 방법: children으로 전달
// layout.tsx (Server)
<Modal>
  <ServerContent /> {/* 서버에서 렌더링된 결과가 children으로 전달됨 */}
</Modal>

// Modal.tsx
"use client";
export default function Modal({ children }: { children: React.ReactNode }) {
  const [open, setOpen] = useState(false);
  return open ? <div className="modal">{children}</div> : <button onClick={() => setOpen(true)}>open</button>;
}
```

### 📍 Streaming과 Suspense

- Server Component는 `<Suspense>`와 함께 쓰면, **데이터가 준비된 부분부터 HTML을 나눠서 스트리밍**할 수 있음
- 느린 데이터 하나 때문에 페이지 전체 응답이 늦어지는 SSR의 문제를 해결

```tsx
export default function Page() {
  return (
    <>
      <Header /> {/* 바로 전송 */}
      <Suspense fallback={<Skeleton />}>
        <Recommendations /> {/* 데이터 준비되면 이어서 스트리밍 */}
      </Suspense>
    </>
  );
}
```

### 💡 Server Component와 SSR의 차이는 무엇인가요?

**⇒ SSR은 컴포넌트를 서버에서 미리 HTML로 렌더링해 첫 화면을 빠르게 보여주는 기술이고, 그 컴포넌트의 JS는 여전히 클라이언트로 전송되어 Hydration됨. Server Component는 서버에서만 실행되고 결과(RSC Payload)만 전송되어 컴포넌트 코드가 클라이언트 번들에 아예 포함되지 않으며 Hydration도 필요 없음. 즉 SSR은 "언제 HTML을 만드느냐"의 문제이고, Server Component는 "어떤 컴포넌트를 클라이언트로 보내지 않느냐"의 문제라서 둘은 함께 사용됨. Next.js에서는 Client Component도 첫 요청 시 SSR되고, Server Component는 SSR과 별개로 서버에서만 실행됨**

### 💡 Server Component를 사용하면 어떤 장점이 있나요?

**⇒ 서버에서만 실행되므로 DB나 내부 API에 직접 접근할 수 있고 API 키 같은 비밀 값이 클라이언트에 노출되지 않음. 또한 컴포넌트 코드와 그 안에서 쓰는 마크다운 파서·날짜 라이브러리 같은 의존성이 클라이언트 번들에 포함되지 않아 JS 크기가 줄고, Hydration할 대상도 줄어 초기 로딩과 상호작용 가능 시점이 빨라짐. 데이터를 서버에서 가져오므로 클라이언트의 요청 워터폴도 줄어듦. 대신 state, 이벤트 핸들러, 브라우저 API를 쓸 수 없으므로 상호작용이 필요한 부분만 Client Component로 분리해 말단에 두는 설계가 중요함**

### 💡 `"use client"`를 페이지 최상단에 붙이면 어떤 문제가 생기나요?

**⇒ `"use client"`는 해당 파일뿐 아니라 그 파일이 import하는 모든 하위 컴포넌트를 클라이언트 번들에 포함시키는 경계이기 때문에, 페이지 최상단에 붙이면 페이지 전체가 Client Component가 되어 Server Component의 장점(번들 크기 감소, 서버 데이터 직접 조회, Hydration 감소)을 모두 잃게 됨. 따라서 페이지와 레이아웃은 Server Component로 두고, 버튼·폼·모달처럼 상태와 이벤트가 필요한 작은 단위에만 `"use client"`를 붙이고, Client Component 안에 서버에서 렌더링한 내용이 필요하면 `children`으로 전달하는 것이 좋음**
