# [Week 13] 13주차 Web Performance / Cache 정리

## 목차

1. [Browser Cache](#1-browser-cache)
2. [Cache-Control](#2-cache-control)
3. [Lazy Loading](#3-lazy-loading)
4. [Code Splitting](#4-code-splitting)
5. [Web Vitals](#5-web-vitals)
6. [Debounce/Throttle](#6-debouncethrottle)
7. [정리](#7-정리)

> 5주차에서 HTTP 캐시의 **동작 원리를**, 10주차에서 **렌더링 단계와 성능 지표를** 다뤘다면, 이번 주는 그 둘을 바탕으로 **실제로 로딩과 반응 속도를 줄이는 방법과** 측정 기준을 정리함

---

## 1. Browser Cache

### 개념

- **Browser Cache**: 한 번 받은 리소스(HTML, JS, CSS, 이미지, API 응답)를 브라우저가 저장해 두었다가 **다시 요청할 때 네트워크 없이 재사용하는** 것
- 효과
  - 네트워크 왕복(RTT)을 없애거나 줄여서 **로딩 속도를 개선**
  - 서버 부하와 데이터 사용량 감소
- 캐시는 속도를 얻는 대신 **오래된 데이터를 보여줄 위험이** 생기므로, 무엇을 얼마나 오래 캐시할지 정하는 것이 핵심

### 캐시가 존재하는 위치

```
브라우저 ──→ Memory Cache ──→ Service Worker Cache ──→ HTTP Cache(Disk) ──→ CDN ──→ Origin
(빠름, 탭 단위)              (JS로 직접 제어)         (헤더로 제어)       (중간 서버)  (원본 서버)
```

| 계층 | 특징 |
|---|---|
| Memory Cache | 메모리에 보관. 가장 빠르지만 탭을 닫으면 사라짐. 같은 페이지 안에서 같은 리소스를 반복 사용할 때 적중 |
| Service Worker Cache | JS(Cache API)로 **직접 저장/응답을 제어**. 오프라인 지원 가능 |
| HTTP Cache (Disk) | 응답 헤더(`Cache-Control` 등)에 따라 동작. 브라우저를 껐다 켜도 유지 |
| CDN / Proxy Cache | 브라우저 바깥의 중간 캐시. 여러 사용자가 공유 |

- DevTools Network 탭의 Size 열에서 `(memory cache)`, `(disk cache)`, `(ServiceWorker)`로 어느 계층에서 응답했는지 확인 가능

### 캐시 동작 흐름

```
리소스 요청
  │
  ├─ 캐시에 없음 ─────────────────────────→ 네트워크 요청 → 응답 저장
  │
  └─ 캐시에 있음
       ├─ 아직 신선함(fresh) ─────────────→ 네트워크 없이 캐시 사용 (200, from cache)
       └─ 만료됨(stale)
            └─ 서버에 재검증 요청 (조건부 요청)
                 ├─ 안 바뀜 → 304 Not Modified, 본문 없이 캐시 재사용
                 └─ 바뀜    → 200 + 새 본문, 캐시 갱신
```

- **Fresh**: 유효 기간 안이라 서버에 묻지 않고 바로 사용
- **Stale**: 유효 기간이 지나서 **재검증이** 필요한 상태. 캐시에서 지워진 것이 아니라 사용 전에 확인이 필요하다는 뜻

### 재검증 (Validation)

- 만료된 캐시가 아직 유효한지 서버에 물어보고, 안 바뀌었으면 **본문 전송 없이 304로** 응답받음
- **ETag / If-None-Match**: 리소스의 버전 식별자(해시 등)를 비교
- **Last-Modified / If-Modified-Since**: 마지막 수정 시각을 비교

```
첫 응답:    200 OK
            ETag: "a1b2c3"
            Cache-Control: max-age=0

재요청:     GET /app.js
            If-None-Match: "a1b2c3"

서버 응답:  304 Not Modified   (본문 없음, 캐시 재사용)
```

- 304는 본문은 없지만 **왕복 자체는 발생하므로**, 가장 빠른 것은 재검증도 필요 없는 fresh 캐시 적중

### 새로고침 종류별 동작

| 동작 | 캐시 처리 |
|---|---|
| 링크 이동, 주소 입력 | 캐시를 최대한 활용 (fresh면 요청 없음) |
| 일반 새로고침 (F5) | 메인 리소스를 **재검증하며** 요청 |
| 강력 새로고침 (Ctrl+Shift+R) | 캐시를 **무시하고** 전부 새로 요청 |
| DevTools의 Disable cache | 개발 중에만 캐시를 꺼서 확인 |

- "캐시 때문에 수정이 안 보인다"는 상황을 디버깅할 때 이 차이를 구분해야 함

### 브라우저 캐시 vs 서버 상태 캐시

- 이 장의 Browser Cache는 **리소스(파일/응답) 단위로 HTTP 헤더에 따라** 브라우저가 처리하는 캐시
- 6주차에서 다룬 React Query의 캐시는 **JS 메모리 안에서 데이터 단위로** 앱이 직접 관리하는 캐시
- 둘은 다른 계층이라 같이 쓰임. API 응답이 HTTP 캐시와 React Query 캐시에 모두 저장될 수 있음

---

## 2. Cache-Control

### 개념

- **Cache-Control**: 응답(또는 요청)에 붙여서 **캐시 방식을 지시하는** HTTP 헤더
- 하나의 헤더에 여러 지시어를 쉼표로 나열 (`Cache-Control: public, max-age=3600`)
- 5주차의 내용을 바탕으로, 이번에는 **지시어를 실무 기준으로 묶어서** 정리

### 주요 지시어

| 지시어 | 의미 |
|---|---|
| `max-age=N` | 응답을 받은 시점부터 **N초 동안 fresh**. 이 기간에는 서버에 묻지 않음 |
| `s-maxage=N` | **공유 캐시(CDN)에만** 적용되는 유효 기간. 브라우저에는 `max-age`가 적용됨 |
| `no-cache` | 캐시에 **저장은 하되**, 사용 전에 **항상 재검증해야** 함 |
| `no-store` | **저장 자체를 금지**. 민감한 데이터에 사용 |
| `public` | CDN 같은 공유 캐시에도 저장 가능 |
| `private` | **브라우저에만** 저장 가능. 사용자별 응답에 사용 |
| `immutable` | fresh 기간 동안 **절대 바뀌지 않음**. 새로고침해도 재검증하지 않음 |
| `must-revalidate` | 만료된 뒤에는 **반드시 재검증하고**, 서버에 연결할 수 없으면 오래된 캐시를 쓰지 않음 |
| `stale-while-revalidate=N` | 만료 후 N초 동안은 **오래된 캐시를 먼저 응답하고** 뒤에서 갱신 |

### 자주 헷갈리는 두 지시어

- `no-cache`는 이름과 달리 **캐시를 쓰지 않는다는 뜻이 아님**
  - 저장은 하고, 쓸 때마다 서버에 확인한 뒤 304면 캐시를 사용
- `no-store`가 진짜로 **저장하지 않는** 지시어
- 로그인 정보, 개인 금융 정보 같은 민감한 응답에는 `no-store`

### 캐시 전략: 리소스 종류별로 다르게

- 핵심 원칙은 **"안 바뀌는 것은 길게, 바뀌는 것은 항상 확인"**

```
HTML          →  Cache-Control: no-cache
JS / CSS      →  Cache-Control: public, max-age=31536000, immutable   (파일명에 해시)
이미지, 폰트   →  Cache-Control: public, max-age=31536000, immutable   (파일명에 해시)
API (공개)     →  Cache-Control: public, max-age=60, stale-while-revalidate=300
API (사용자별) →  Cache-Control: private, no-cache  또는  no-store
```

### Cache Busting: 파일명에 해시

- 정적 파일을 1년씩 캐시하면, 배포 후에 **사용자가 새 버전을 받지 못하는** 문제가 생김
- 해결: **파일 내용의 해시를 파일명에 포함**
  - 내용이 바뀌면 파일명이 달라져서 **새 URL이 되므로 캐시와 충돌하지 않음**
  - 내용이 같으면 URL이 같아서 계속 캐시를 재사용

```
배포 1:  index.html  →  <script src="/assets/app.3f9a1c.js">
배포 2:  index.html  →  <script src="/assets/app.b27e04.js">   ← 새 URL이므로 새로 받음
         (코드가 안 바뀐 vendor.9d1f52.js 는 URL이 그대로라서 캐시 재사용)
```

- 그래서 **HTML은 `no-cache`로 항상 최신을 확인하고**, 그 안의 해시 URL이 바뀌면서 나머지 파일이 자연스럽게 교체되는 구조
- 번들러(Webpack, Vite)가 빌드할 때 `[contenthash]`로 자동 처리
- 이 구조 때문에 Code Splitting(4장)에서 **자주 안 바뀌는 코드를 따로 분리하면** 캐시 적중률이 올라감

### 배포 직후 생길 수 있는 문제

- 사용자가 예전 HTML을 열어둔 채로 새 배포가 되면, 그 HTML이 가리키는 **예전 해시 파일이 서버에서 삭제되어** 404가 날 수 있음
- 대응
  - 이전 버전의 정적 파일을 일정 기간 남겨둠
  - 동적 import 실패 시 에러를 잡아서 **새로고침을 유도** (4장)

### Stale-While-Revalidate

- 만료된 캐시를 **즉시 응답해서 빠르게 보여주고**, 백그라운드에서 갱신해서 **다음 요청부터 새 데이터를** 쓰는 방식
- 사용자는 항상 빠른 응답을 받고, 약간의 오래된 데이터를 허용할 수 있는 곳에 적합 (뉴스 목록, 추천 등)
- 6주차 React Query의 staleTime / 백그라운드 refetch, 12주차의 ISR도 같은 아이디어

```
Cache-Control: max-age=60, stale-while-revalidate=300

0~60초      fresh        캐시 즉시 사용, 서버 요청 없음
60~360초    stale        캐시 즉시 응답 + 백그라운드에서 재검증
360초 이후  만료          서버 응답을 기다림
```

### Vary

- 같은 URL이라도 **요청 헤더에 따라 응답이 달라지는** 경우 캐시를 구분하기 위한 헤더
- 예: `Vary: Accept-Encoding`이면 gzip/br 압축 여부별로 따로 캐시
- `Vary: Cookie`처럼 값의 종류가 많은 헤더를 지정하면 **캐시 적중률이 크게 떨어짐**
- 사용자별 응답은 `Vary`로 공유 캐시를 구분하기보다 `private`으로 공유 캐시를 막는 것이 안전

### 서버 설정 예시

```js
// Express: 경로별로 다른 캐시 정책
app.use('/assets', express.static('dist/assets', {
  maxAge: '1y',
  immutable: true,          // 해시가 붙은 정적 파일
}));

app.get('/', (req, res) => {
  res.set('Cache-Control', 'no-cache');   // HTML은 항상 재검증
  res.sendFile('dist/index.html');
});

app.get('/api/me', (req, res) => {
  res.set('Cache-Control', 'private, no-store');   // 사용자별, 민감 데이터
  res.json(getUser(req));
});
```

### Service Worker 캐시

- Service Worker는 브라우저와 네트워크 사이에서 **요청을 가로채서** 직접 응답을 결정할 수 있음
- HTTP 캐시와 별개로 `Cache API`에 리소스를 저장하고 꺼내 쓰므로, **오프라인 지원과 세밀한 전략이** 가능

| 전략 | 동작 | 적합한 리소스 |
|---|---|---|
| Cache First | 캐시에 있으면 사용, 없으면 네트워크 | 해시된 정적 파일, 폰트, 이미지 |
| Network First | 네트워크 우선, 실패하면 캐시 | 자주 바뀌는 API, HTML |
| Stale While Revalidate | 캐시를 먼저 응답하고 백그라운드 갱신 | 빠른 응답과 최신성의 균형이 필요한 것 |

```js
// sw.js: Cache First 예시
self.addEventListener('fetch', (event) => {
  event.respondWith(
    caches.match(event.request).then((cached) => {
      if (cached) return cached;                          // 캐시 적중
      return fetch(event.request).then((res) => {
        const copy = res.clone();                         // 응답 본문은 한 번만 읽을 수 있으므로 복제
        caches.open('static-v1').then((c) => c.put(event.request, copy));
        return res;
      });
    })
  );
});
```

- Service Worker는 **HTTPS에서만** 동작 (7주차)
- 캐시 이름에 버전을 붙이고, 새 SW가 활성화될 때 **이전 캐시를 삭제해야** 함 (안 그러면 오래된 파일이 계속 남음)
- 잘못 구성하면 사용자가 **계속 옛 버전을 보게 되는** 문제가 생기므로 업데이트 전략을 함께 설계해야 함

---

## 3. Lazy Loading

### 개념

- **Lazy Loading**: 지금 당장 필요하지 않은 리소스를 **처음부터 받지 않고, 필요한 시점에 불러오는** 기법
- 효과
  - 초기에 받는 데이터가 줄어서 **초기 로딩이 빨라짐**
  - 사용자가 보지 않을 부분의 네트워크와 메모리 낭비 방지
- 반대 개념은 **Eager Loading** (미리 전부 로드)
- 핵심은 **"무엇을 늦출 것인가"를** 구분하는 것
  - 첫 화면에 보이는 것(above the fold)은 늦추면 안 됨
  - 스크롤해야 보이는 것, 클릭해야 열리는 것은 늦춰도 됨

### 이미지와 iframe

- 브라우저 기본 기능으로 `loading="lazy"`를 지원

```html
<!-- 화면 밖 이미지: 뷰포트 근처에 오면 로드 -->
<img src="photo.jpg" loading="lazy" width="600" height="400" alt="..." />

<!-- iframe(지도, 동영상 등)도 동일 -->
<iframe src="https://example.com/embed" loading="lazy"></iframe>
```

- **`width`, `height`를 반드시 지정**: 로드되기 전에 공간을 확보해야 이미지가 나타날 때 레이아웃이 밀리지 않음 (CLS, 5장)
- 브라우저가 알아서 **뷰포트 근처에 오기 전에 미리 로드를** 시작하므로 사용자는 대부분 끊김을 느끼지 못함

### LCP 이미지에는 적용하지 않음

- 첫 화면의 가장 큰 이미지(LCP 요소)에 `loading="lazy"`를 쓰면 **오히려 LCP가 느려짐**
  - lazy 이미지는 레이아웃 계산 후에야 로드 여부가 결정되어 요청이 늦어짐
- LCP 이미지는 **빠르게 받도록 우선순위를 높임**

```html
<!-- 첫 화면의 대표 이미지: lazy 금지, 우선순위 상향 -->
<img src="hero.jpg" fetchpriority="high" width="1200" height="600" alt="..." />

<!-- 나머지: lazy -->
<img src="card-1.jpg" loading="lazy" width="300" height="200" alt="..." />
```

### Intersection Observer로 직접 구현

- `loading="lazy"`로 처리되지 않는 경우(배경 이미지, 커스텀 컴포넌트, 무한 스크롤)에 사용 (11주차)

```js
const io = new IntersectionObserver((entries) => {
  for (const entry of entries) {
    if (!entry.isIntersecting) continue;
    const img = entry.target;
    img.src = img.dataset.src;       // data-src에 두었던 실제 주소를 설정해서 로드 시작
    io.unobserve(img);               // 한 번 로드한 뒤에는 관찰 중단
  }
}, { rootMargin: '200px' });         // 화면에 들어오기 200px 전에 미리 로드

document.querySelectorAll('img[data-src]').forEach((img) => io.observe(img));
```

- `rootMargin`으로 **여유 거리를** 두면 사용자가 스크롤할 때 이미 로드가 끝나 있음

### 컴포넌트 / 모듈 단위 Lazy Loading

- 이미지뿐 아니라 **JS 코드 자체도 필요한 시점에 로드할** 수 있음 (4장 Code Splitting과 연결)
- 사용자가 열기 전까지 필요 없는 **모달, 차트, 에디터, 관리자 화면** 등이 대상

```jsx
import { lazy, Suspense, useState } from 'react';

const Chart = lazy(() => import('./Chart'));      // 사용될 때 비로소 코드를 받음

function Report() {
  const [show, setShow] = useState(false);
  return (
    <>
      <button onClick={() => setShow(true)}>차트 보기</button>
      {show && (
        <Suspense fallback={<p>불러오는 중...</p>}>
          <Chart />
        </Suspense>
      )}
    </>
  );
}
```

### 로딩 시점을 조절하는 Resource Hints

| 힌트 | 용도 |
|---|---|
| `preload` | **지금 현재 페이지에서 곧 필요한** 리소스를 높은 우선순위로 미리 요청 (LCP 이미지, 핵심 폰트) |
| `prefetch` | **다음에 필요할 가능성이 높은** 리소스를 한가한 시간에 낮은 우선순위로 미리 받아 둠 (다음 페이지) |
| `preconnect` | 다른 도메인에 **연결(DNS + TCP + TLS)을 미리** 맺어 둠 (4주차) |
| `dns-prefetch` | DNS 조회만 미리 수행 |

```html
<link rel="preload" href="/fonts/main.woff2" as="font" type="font/woff2" crossorigin />
<link rel="preconnect" href="https://cdn.example.com" />
<link rel="prefetch" href="/next-page.js" />
```

- lazy는 **늦추기**, preload/prefetch는 **당겨오기라서** 반대로 보이지만, 모두 **"필요한 때에 맞춰 로드한다"는** 같은 목적
- `preload`를 남발하면 정작 중요한 리소스와 **대역폭을 경쟁하므로** 꼭 필요한 것에만 사용

### 주의할 점

- **늦게 로드할수록 사용자는 로딩을 기다릴 수 있음**: 클릭 후에 로드하면 지연이 느껴지므로, 호버나 유휴 시간에 `prefetch`로 보완
- **fallback UI 필요**: 로드 중에 스켈레톤이나 스피너를 보여주고, 크기를 맞춰 CLS 방지
- **SEO**: 중요한 콘텐츠를 스크롤/이벤트 후에만 로드하면 크롤러가 못 볼 수 있음
- **JS가 없는 환경에서는** `data-src` 방식이 동작하지 않음

---

## 4. Code Splitting

### 개념

- **Code Splitting**: 하나의 큰 JS 번들을 **여러 개의 작은 청크(chunk)로 나눠서**, 필요한 청크만 필요한 시점에 받게 하는 기법
- 왜 필요한가
  - 번들이 크면 **다운로드, 파싱, 실행** 시간이 모두 길어짐 (10주차)
  - JS는 같은 용량의 이미지보다 **처리 비용이 비쌈** (실행까지 해야 하므로)
  - 사용자가 지금 보는 화면에 필요 없는 코드까지 받는 것은 낭비

```
분할 전:   [ main.js (2MB) ] ─────────────────────────→ 전부 받고 실행해야 첫 화면
분할 후:   [ main.js (200KB) ] → 첫 화면 즉시
           [ dashboard.js ] ← /dashboard 진입 시에만
           [ editor.js    ] ← 에디터를 열 때만
           [ vendor.js    ] ← 라이브러리, 캐시 적중률 높음
```

### Dynamic import

- 정적 `import`는 번들 시점에 한꺼번에 포함되지만, **`import()`는 실행 시점에 비동기로 모듈을 로드하는** Promise를 반환 (9주차)
- 번들러가 `import()`를 만나면 **그 모듈을 별도 청크로 분리함**

```js
// 정적 import: main 번들에 포함됨
import { heavyFn } from './heavy';

// 동적 import: 별도 청크로 분리, 호출 시점에 로드
button.addEventListener('click', async () => {
  const { heavyFn } = await import('./heavy');   // 이때 네트워크 요청
  heavyFn();
});
```

### 분할 기준

| 기준 | 설명 |
|---|---|
| Entry Point 분리 | 여러 진입점을 각각 번들링 |
| Route 단위 | **페이지(라우트)마다 청크를 분리**. 가장 효과적이고 흔한 방식 |
| Component 단위 | 모달, 차트, 에디터처럼 무겁고 조건부로 쓰이는 컴포넌트 |
| Vendor 분리 | 자주 안 바뀌는 **라이브러리 코드를 앱 코드와 분리해서** 캐시 활용 |
| 공통 청크 | 여러 청크가 함께 쓰는 모듈을 따로 빼서 중복 제거 |

### Route 단위 분할 (React Router)

```jsx
import { lazy, Suspense } from 'react';
import { Routes, Route } from 'react-router-dom';

const Home = lazy(() => import('./pages/Home'));
const Dashboard = lazy(() => import('./pages/Dashboard'));
const Settings = lazy(() => import('./pages/Settings'));

export default function App() {
  return (
    <Suspense fallback={<PageSkeleton />}>
      <Routes>
        <Route path="/" element={<Home />} />
        <Route path="/dashboard" element={<Dashboard />} />
        <Route path="/settings" element={<Settings />} />
      </Routes>
    </Suspense>
  );
}
```

- 방문한 라우트의 코드만 받으므로 **초기 번들이 크게 줄어듦**
- Next.js는 **페이지(라우트) 단위로 자동 분할하고**, 컴포넌트는 `next/dynamic`으로 분할

```jsx
import dynamic from 'next/dynamic';

const Editor = dynamic(() => import('./Editor'), {
  ssr: false,                       // 서버에서 렌더하지 않음 (브라우저 전용 라이브러리일 때)
  loading: () => <p>로딩 중...</p>,
});
```

### 번들러 설정 (Vendor 분리)

```js
// webpack: 라이브러리를 별도 청크로 분리
optimization: {
  splitChunks: {
    chunks: 'all',
    cacheGroups: {
      vendor: {
        test: /[\\/]node_modules[\\/]/,
        name: 'vendor',
      },
    },
  },
},
```

- vendor 청크는 앱 코드가 바뀌어도 **해시가 그대로라서**, 사용자가 다시 받지 않고 캐시를 재사용 (2장 Cache Busting)
- 반대로 너무 잘게 쪼개면 **요청 수가 늘고 워터폴이 생겨서** 오히려 느려질 수 있음 (HTTP/2에서도 한계가 있음)

### 번들 줄이기: 분할 외의 방법

- **Tree Shaking**: 사용하지 않는 export를 번들에서 제거 (ES Module의 정적 구조 덕분에 가능)
- **라이브러리 선택**: 전체를 가져오지 않고 필요한 함수만 import

```js
// 전체 라이브러리가 번들에 포함될 수 있음
import _ from 'lodash';
_.debounce(fn, 300);

// 필요한 함수만 포함
import debounce from 'lodash/debounce';
```

- **번들 분석**: `webpack-bundle-analyzer`, `vite-bundle-visualizer`로 **무엇이 큰지 먼저 확인한** 뒤에 최적화
- **압축**: gzip / Brotli 전송 압축, minify (4주차 / 5주차)

### 분할한 청크를 미리 받아 두기

- 클릭한 뒤에 청크를 받으면 지연이 보이므로, **사용자가 이동할 가능성이 높을 때 미리 받아 둠**

```jsx
const loadSettings = () => import('./pages/Settings');
const Settings = lazy(loadSettings);

// 링크에 마우스를 올리면 미리 로드
<Link to="/settings" onMouseEnter={loadSettings}>설정</Link>
```

- 번들러 힌트로 `prefetch`를 지정할 수도 있음 (webpack의 `/* webpackPrefetch: true */` 매직 코멘트)

### 청크 로드 실패 처리

- 네트워크 오류나 **배포로 인한 파일 교체로** `import()`가 실패할 수 있음 (2장)
- 실패하면 앱이 하얀 화면이 될 수 있으므로, **에러 바운더리와 재시도/새로고침 유도가** 필요

```jsx
async function loadWithRetry(importer, retries = 1) {
  try {
    return await importer();
  } catch (err) {
    if (retries > 0) return loadWithRetry(importer, retries - 1);
    window.location.reload();     // 새 배포로 청크가 사라졌을 수 있으므로 최신 HTML을 다시 받음
  }
}

const Settings = lazy(() => loadWithRetry(() => import('./pages/Settings')));
```

### Lazy Loading과 Code Splitting의 관계

- **Code Splitting**: 코드를 **나누는** 작업 (빌드 시점)
- **Lazy Loading**: 나눠 둔 것을 **필요할 때 가져오는** 작업 (실행 시점)
- 둘은 짝으로 쓰임. 나누기만 하고 처음에 전부 받으면 효과가 없음

---

## 5. Web Vitals

### 개념

- **Web Vitals**: Google이 정의한 **사용자 경험 중심의 성능 지표** 모음
- 그중 검색 순위에도 반영되는 핵심 3개를 **Core Web Vitals라고** 함
- "로딩이 빠른가 / 누르면 반응하는가 / 화면이 안 흔들리는가"를 각각 측정

| 지표 | 측정하는 것 | 좋음 | 나쁨 |
|---|---|---|---|
| **LCP** (Largest Contentful Paint) | **로딩**: 가장 큰 콘텐츠가 표시되는 시점 | 2.5초 이하 | 4초 초과 |
| **INP** (Interaction to Next Paint) | **반응성**: 사용자 입력 후 화면이 갱신되기까지의 지연 | 200ms 이하 | 500ms 초과 |
| **CLS** (Cumulative Layout Shift) | **시각적 안정성**: 예상치 못한 레이아웃 이동의 누적 점수 | 0.1 이하 | 0.25 초과 |

- 기준 구간은 "좋음 / 개선 필요 / 나쁨" 세 단계이며, 사용자의 **75번째 백분위(p75)** 값으로 평가
- 임계값은 변경될 수 있으므로 정확한 최신 기준은 공식 문서 확인

### LCP (Largest Contentful Paint)

- 뷰포트 안에서 **가장 큰 이미지나 텍스트 블록이 그려지는 시점**
- 대표 요소: 히어로 이미지, 큰 제목 텍스트, 비디오 포스터

```
LCP까지의 구성 시간
TTFB  →  리소스 로드 지연  →  리소스 다운로드  →  렌더링 지연
(서버)   (발견이 늦음)        (파일 크기)         (렌더 차단 JS/CSS)
```

- 개선 방법
  - **TTFB 줄이기**: CDN, 서버 캐시, SSG/ISR (12주차)
  - **LCP 이미지 우선순위 상향**: `fetchpriority="high"`, `preload`, lazy 금지 (3장)
  - **이미지 최적화**: WebP/AVIF 포맷, 적절한 크기(`srcset`), 압축
  - **렌더 차단 리소스 줄이기**: 중요하지 않은 JS는 `defer`/`async`, CSS 최소화 (10주차 CRP)
  - **CSR 의존 줄이기**: 빈 HTML 후 JS로 그리면 LCP가 늦어지므로 SSR/SSG 고려
  - 웹폰트 때문에 텍스트가 늦게 보이지 않도록 `font-display: swap`

### INP (Interaction to Next Paint)

- 페이지에서 일어난 **모든 클릭, 탭, 키 입력의 지연을 관찰해서**, 그중 **가장 느린 것에 가까운 값을** 보고함
- 한 번의 상호작용 지연 = **입력 지연 + 이벤트 처리 시간 + 다음 화면이 그려지기까지의 시간**
- 기존 FID(첫 입력만 측정)를 대체. **페이지를 쓰는 내내의 반응성을** 보는 지표

```
클릭 ──→ [입력 지연] ──→ [핸들러 실행] ──→ [렌더링/페인트] ──→ 화면 갱신
          메인 스레드가     JS 처리 시간      Layout, Paint
          바빠서 대기                        (11주차)
```

- 메인 스레드가 **Long Task(50ms 이상)로 막혀 있으면** 입력이 대기하므로 INP가 나빠짐 (9주차)
- 개선 방법
  - **긴 작업을 쪼개기**: 청크 단위로 처리하고 사이에 메인 스레드를 양보
  - **무거운 계산을 Web Worker로** 이동 (2주차)
  - **Code Splitting과 Hydration 비용 줄이기로** 초기 JS 실행량 감소 (4장, 12주차)
  - **불필요한 리렌더링 줄이기** (11주차 React Rendering)
  - **이벤트 핸들러를 가볍게**: 스크롤/입력은 Debounce/Throttle (6장)

```js
// 긴 작업을 쪼개서 중간에 브라우저가 입력을 처리하게 함
async function processAll(items) {
  for (let i = 0; i < items.length; i++) {
    process(items[i]);
    if (i % 100 === 0) {
      await new Promise((r) => setTimeout(r, 0));   // 메인 스레드를 양보 (9주차 Macrotask)
    }
  }
}
```

### CLS (Cumulative Layout Shift)

- 화면에 보이던 요소가 **사용자 의도 없이 밀리는 정도를** 점수로 누적
- 대표 상황: 읽던 글이 광고가 로드되면서 밀려 **엉뚱한 버튼을 누르는** 경우
- 점수는 **얼마나 큰 영역이 얼마나 멀리 이동했는지의** 곱으로 계산 (11주차 Reflow와 연결)

| 원인 | 해결 |
|---|---|
| 크기 없는 이미지/iframe | `width`, `height` 또는 `aspect-ratio` 지정 |
| 나중에 삽입되는 광고, 배너 | 미리 **공간(최소 높이)을 확보** |
| 웹폰트 교체로 글자 크기 변화 | `font-display`와 대체 폰트 크기 보정 |
| 기존 콘텐츠 위에 동적 삽입 | 위가 아니라 **아래쪽에 삽입하거나** 공간 확보 |
| 레이아웃 속성으로 애니메이션 | `transform` 사용 (10주차 Composite) |

```css
img, iframe { aspect-ratio: 16 / 9; width: 100%; height: auto; }   /* 로드 전에 공간 확보 */
.ad-slot { min-height: 250px; }                                      /* 광고가 들어올 자리 */
```

- 사용자 입력 직후(500ms 이내)에 일어나는 이동은 CLS에 포함되지 않음

### 그 외 지표

| 지표 | 설명 |
|---|---|
| FCP (First Contentful Paint) | 첫 콘텐츠(텍스트/이미지)가 처음 그려지는 시점 |
| TTFB (Time to First Byte) | 요청 후 **첫 바이트를 받기까지의 시간**. 서버/네트워크/캐시 영향 |
| TBT (Total Blocking Time) | FCP~TTI 사이 Long Task의 **차단 시간 합계**. 실험실 도구에서 INP와 연관된 지표로 사용 |

### 측정 방법: Lab vs Field

| | Lab (실험실) | Field (실사용자) |
|---|---|---|
| 방식 | 정해진 환경에서 시뮬레이션 | **실제 사용자 브라우저에서** 수집 |
| 도구 | Lighthouse, DevTools Performance | CrUX, `web-vitals` 라이브러리, RUM 서비스 |
| 장점 | 재현 가능, 배포 전 확인, 원인 분석 | **진짜 사용자 경험** 반영 |
| 한계 | 실제 기기/네트워크 다양성을 반영 못함 | 원인 추적이 어렵고 데이터가 쌓여야 함 |

- INP는 사용자 상호작용이 있어야 측정되므로 **Field 데이터가 중요**
- Lab은 **문제를 찾고 고치는 용도**, Field는 **실제 결과를 확인하는 용도로** 함께 사용

```js
// web-vitals 라이브러리로 실사용자 데이터 수집
import { onLCP, onINP, onCLS } from 'web-vitals';

function send(metric) {
  navigator.sendBeacon('/analytics', JSON.stringify({
    name: metric.name,
    value: metric.value,
    id: metric.id,
  }));   // 페이지가 닫혀도 전송되도록 sendBeacon 사용
}

onLCP(send);
onINP(send);
onCLS(send);
```

### PerformanceObserver

- 브라우저가 제공하는 **성능 이벤트를 구독하는** API. 지표 라이브러리의 기반

```js
new PerformanceObserver((list) => {
  for (const entry of list.getEntries()) {
    console.log('LCP 후보:', entry.startTime, entry.element);
  }
}).observe({ type: 'largest-contentful-paint', buffered: true });
```

### 지표와 이번 주 내용의 연결

| 문제 | 관련 지표 | 해결 수단 |
|---|---|---|
| 첫 화면이 늦음 | LCP, TTFB | 캐시/CDN (1~2장), LCP 이미지 우선순위 (3장), SSR/SSG (12주차) |
| 번들이 커서 느림 | LCP, TBT | Code Splitting (4장), Lazy Loading (3장) |
| 누르면 반응이 느림 | INP | Long Task 분할, Debounce/Throttle (6장), 리렌더링 감소 |
| 화면이 흔들림 | CLS | 이미지 크기 지정, 공간 확보 |

---

## 6. Debounce/Throttle

### 개념

- 스크롤, 리사이즈, 입력 같은 이벤트는 **짧은 시간에 매우 많이 발생하는데**, 그때마다 무거운 작업(API 호출, 레이아웃 계산)을 하면 성능이 나빠짐
- **Debounce와** **Throttle은** **함수의 실행 빈도를 제한해서** 이를 줄이는 기법 (INP 개선에도 도움)

| | Debounce | Throttle |
|---|---|---|
| 동작 | 이벤트가 **멈추고 일정 시간이 지난 뒤** 한 번 실행 | **일정 간격마다 최대 한 번** 실행 |
| 이벤트가 계속 오면 | 계속 미뤄짐 (실행 안 됨) | 간격마다 꾸준히 실행됨 |
| 기준 | "그만 입력했을 때" | "진행되는 동안 주기적으로" |
| 용도 | 검색어 자동완성, 입력값 검증, 리사이즈 종료 후 계산 | 스크롤 위치 추적, 무한 스크롤, 마우스 이동 |

### 시간 흐름으로 비교

```
이벤트   : x x x x x x . . . . x x x . . . . . .
                              (300ms 간격)

Debounce : . . . . . . . . . X . . . . . . . . X     (이벤트가 멈춘 뒤 300ms 후 1회)

Throttle : X . . . X . . . X . . . X . . . X . .     (300ms 간격마다 1회)
```

### Debounce 구현

- 이벤트가 올 때마다 **기존 타이머를 취소하고 새로 설정**
- 클로저(8주차)로 타이머 ID를 유지

```js
function debounce(fn, delay) {
  let timer;                                  // 클로저로 유지되는 타이머 ID
  return function (...args) {
    clearTimeout(timer);                      // 이전 예약을 취소
    timer = setTimeout(() => {
      fn.apply(this, args);                   // 마지막 호출 후 delay가 지나면 실행
    }, delay);
  };
}

const onSearch = debounce((keyword) => fetchResults(keyword), 300);
input.addEventListener('input', (e) => onSearch(e.target.value));
```

### Throttle 구현

- 마지막 실행 이후 **일정 시간이 지났을 때만** 실행

```js
function throttle(fn, interval) {
  let lastTime = 0;
  return function (...args) {
    const now = Date.now();
    if (now - lastTime >= interval) {         // 간격이 지났을 때만 실행
      lastTime = now;
      fn.apply(this, args);
    }
  };
}

window.addEventListener('scroll', throttle(() => {
  updateScrollProgress();
}, 100));
```

### Leading / Trailing

- 구간의 **처음(leading)에 실행할지, 끝(trailing)에 실행할지를** 정하는 옵션
- 위 구현은 Debounce는 **trailing**, Throttle은 **leading** 방식
- 필요에 따라 선택
  - 버튼 연타 방지: **leading** debounce (첫 클릭만 즉시 처리하고 나머지 무시)
  - 검색 자동완성: **trailing** debounce (입력이 끝난 뒤 조회)
  - 스크롤 throttle: 마지막 위치도 반영되도록 **trailing 실행을 함께** 사용
- 직접 구현보다 `lodash`의 `debounce`, `throttle`이 이런 옵션(`leading`, `trailing`, `cancel`, `flush`)을 제공

### requestAnimationFrame으로 Throttle

- 스크롤 중 **화면 갱신 작업은** 시간 간격 대신 **프레임에 맞춰 한 번만** 실행하는 것이 자연스러움 (10주차)

```js
let ticking = false;

window.addEventListener('scroll', () => {
  if (ticking) return;                        // 이미 예약되어 있으면 무시
  ticking = true;
  requestAnimationFrame(() => {
    updateScrollUI();                         // 프레임당 한 번만 실행
    ticking = false;
  });
});
```

- 브라우저가 화면을 그리는 주기(보통 16ms)에 맞춰지므로 **불필요한 중복 계산이 없음**

### React에서의 주의점

- 컴포넌트가 리렌더링될 때마다 `debounce(...)`를 호출하면 **새 함수와 새 타이머가 만들어져서** debounce가 동작하지 않음 (11주차 React Rendering)
- 함수를 **렌더링 사이에 유지해야** 함

```jsx
import { useMemo, useEffect, useState } from 'react';
import debounce from 'lodash/debounce';

function Search() {
  const [keyword, setKeyword] = useState('');

  // 나쁨: 렌더링마다 새 debounce 함수가 생김
  // const onChange = debounce((v) => fetchResults(v), 300);

  // 좋음: 한 번 만든 debounce 함수를 유지
  const debouncedFetch = useMemo(() => debounce((v) => fetchResults(v), 300), []);

  useEffect(() => () => debouncedFetch.cancel(), [debouncedFetch]);   // 언마운트 시 예약된 호출 취소

  return (
    <input
      value={keyword}
      onChange={(e) => {
        setKeyword(e.target.value);           // 입력창 값은 즉시 반영
        debouncedFetch(e.target.value);       // 요청만 debounce
      }}
    />
  );
}
```

- **입력창에 보이는 값은 즉시 갱신하고**, **무거운 작업(요청)만 debounce해야** 입력이 끊기지 않음
- 값 자체를 지연시키는 방식으로 **debounced value 훅을** 만들어 `useEffect`의 의존성으로 쓰는 패턴도 흔함

```jsx
function useDebouncedValue(value, delay) {
  const [debounced, setDebounced] = useState(value);
  useEffect(() => {
    const id = setTimeout(() => setDebounced(value), delay);
    return () => clearTimeout(id);            // 값이 다시 바뀌면 이전 타이머 제거
  }, [value, delay]);
  return debounced;
}

// 사용: debounced 값이 바뀔 때만 요청
const debouncedKeyword = useDebouncedValue(keyword, 300);
const { data } = useQuery({ queryKey: ['search', debouncedKeyword], queryFn: () => fetchResults(debouncedKeyword) });
```

- React 18의 `useDeferredValue`는 지연 시간 대신 **렌더링 우선순위로 지연시키는** 방식. 네트워크 요청을 줄이는 것과는 목적이 다름

### 요청 취소와 함께 쓰기

- Debounce로 호출 횟수를 줄여도, **이전 요청의 응답이 나중에 도착해서 최신 결과를 덮어쓰는** 경쟁 상태(race condition)가 생길 수 있음 (9주차)
- 새 요청을 보낼 때 **이전 요청을 취소**

```js
let controller;

async function search(keyword) {
  controller?.abort();                        // 이전 요청 취소
  controller = new AbortController();
  try {
    const res = await fetch(`/api/search?q=${keyword}`, { signal: controller.signal });
    render(await res.json());
  } catch (e) {
    if (e.name !== 'AbortError') throw e;     // 취소로 인한 에러는 무시
  }
}
```

### 선택 기준

- **"끝났을 때 결과만 필요"하면** Debounce (검색어, 자동 저장, 리사이즈 후 레이아웃 재계산)
- **"진행 중에도 주기적으로 필요"하면** Throttle (스크롤 위치, 드래그, 무한 스크롤 감지)
- **화면 갱신이** 목적이면 `requestAnimationFrame`
- 스크롤 노출 감지처럼 **위치 계산이 목적이면** Throttle 대신 `IntersectionObserver`가 더 적합 (11주차)
- 지연 시간이 길면 반응이 둔해지고, 짧으면 효과가 작으므로 **용도에 맞게 조정** (검색은 보통 200~500ms, 스크롤은 100ms 안팎)

---

## 7. 정리

| 개념 | 핵심 |
|---|---|
| Browser Cache | 받은 리소스를 재사용해서 네트워크를 줄임. Memory, Service Worker, HTTP Cache 계층이 있음. fresh면 요청 없음, stale이면 재검증(304) |
| Cache-Control | `max-age`, `no-cache`(재검증 후 사용), `no-store`(저장 금지), `immutable`, `stale-while-revalidate`. 리소스 종류별로 다르게 지정 |
| Cache Busting | 파일명에 해시를 넣어 정적 파일은 길게 캐시하고, HTML은 `no-cache`로 최신 해시 URL을 가리키게 함 |
| Lazy Loading | 필요할 때 로드. `loading="lazy"`, IntersectionObserver, `React.lazy`. LCP 이미지에는 적용하지 않음 |
| Code Splitting | 번들을 청크로 나눔. 라우트 단위가 가장 효과적. `import()`로 로드 시점 제어, vendor 분리로 캐시 활용 |
| Web Vitals | LCP(로딩), INP(반응성), CLS(안정성). Lab으로 원인을 찾고 Field로 실제 결과를 확인 |
| Debounce/Throttle | 이벤트 실행 빈도 제한. Debounce는 멈춘 뒤 한 번, Throttle은 간격마다 한 번 |

### 한 장으로 보는 성능 개선 흐름

```
측정 (Lighthouse, web-vitals)
  │
  ├─ 첫 화면이 느림 (LCP)     → 캐시/CDN, LCP 이미지 우선순위, 이미지 최적화, SSR/SSG
  ├─ 번들이 큼 (TBT, LCP)     → 번들 분석 → Code Splitting, Lazy Loading, Tree Shaking
  ├─ 반응이 느림 (INP)        → Long Task 분할, 리렌더링 감소, Debounce/Throttle, Web Worker
  └─ 화면이 흔들림 (CLS)      → 이미지 크기/공간 확보, 폰트 설정
  │
다시 측정해서 개선 확인
```

### 시나리오별 선택

| 상황 | 방법 |
|---|---|
| 배포 후에도 사용자가 옛 JS를 계속 씀 | HTML은 `no-cache`, 정적 파일은 해시 파일명 + 장기 캐시 |
| 첫 화면 로딩이 느리고 번들이 2MB | 번들 분석 후 라우트 단위 Code Splitting |
| 히어로 이미지가 늦게 표시됨 | lazy 제거, `fetchpriority="high"`, `preload`, 포맷/크기 최적화 |
| 긴 목록의 이미지가 많음 | `loading="lazy"` + 크기 지정 |
| 검색창에 입력할 때마다 API 호출 | Debounce + 이전 요청 취소 |
| 스크롤할 때마다 무거운 계산 | `requestAnimationFrame` 또는 Throttle, 노출 감지는 IntersectionObserver |
| 광고가 로드되면서 화면이 밀림 | 광고 영역에 최소 높이 확보 |
| 로그인한 사용자의 개인 정보 API | `private, no-store` |

### 체크리스트

- HTML에는 `no-cache`, 해시된 정적 파일에는 `max-age=31536000, immutable`이 설정되어 있는가
- 사용자별 응답이나 민감 데이터에 `no-store` 또는 `private`이 적용되어 있는가
- 첫 화면의 LCP 이미지에 `loading="lazy"`를 쓰지 않았는가
- 이미지와 iframe에 크기(`width`/`height` 또는 `aspect-ratio`)가 지정되어 있는가
- 번들 분석으로 큰 의존성을 확인했고, 라우트 단위로 분할했는가
- 동적 import 실패(배포로 인한 404, 네트워크 오류)를 처리하는가
- Lab 도구뿐 아니라 실사용자 데이터(Field)로 LCP, INP, CLS를 확인하는가
- 입력/스크롤 핸들러에 Debounce/Throttle을 적용했고, React에서 함수가 렌더링마다 새로 만들어지지 않는가