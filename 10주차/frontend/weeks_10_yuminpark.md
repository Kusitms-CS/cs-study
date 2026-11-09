# [Week 10] 10주차 Browser / Rendering 정리

## 목차

1. [DOM](#1-dom)
2. [CSSOM](#2-cssom)
3. [Render Tree](#3-render-tree)
4. [Critical Rendering Path](#4-critical-rendering-path)
5. [Layout](#5-layout)
6. [Paint](#6-paint)
7. [Composite](#7-composite)
8. [정리](#8-정리)

---

## 1. DOM

### 개념

- **DOM(Document Object Model)**: 브라우저가 HTML을 파싱해서 만든 **객체 트리**. 문서의 구조와 내용을 노드로 표현함
- JS가 DOM API로 문서의 구조, 내용, 스타일에 접근하고 변경할 수 있음
- 우리가 작성한 HTML 파일(텍스트)과 브라우저가 실제로 다루는 DOM(객체)은 **다른 것**

### HTML이 DOM이 되는 과정

```
Bytes ──► Characters ──► Tokens ──► Nodes ──► DOM Tree
(네트워크)   (인코딩 해석)   (태그 단위로 분해)  (객체로 변환)  (트리로 연결)
```

| 단계 | 설명 |
|---|---|
| Bytes → Characters | 네트워크로 받은 바이트를 문자 인코딩(UTF-8 등)에 따라 문자로 변환 |
| Characters → Tokens | `<html>`, `<body>`, 텍스트 같은 의미 있는 단위(토큰)로 분해 |
| Tokens → Nodes | 각 토큰을 속성과 규칙을 가진 객체로 변환 |
| Nodes → DOM | 태그의 포함 관계에 따라 부모-자식 트리로 연결 |

```html
<html>
  <body>
    <h1>Title</h1>
    <p>Hello <span>world</span></p>
  </body>
</html>
```

```
document
└─ html
   ├─ head
   └─ body
      ├─ h1
      │   └─ "Title"
      └─ p
          ├─ "Hello "
          └─ span
              └─ "world"
```

- 태그뿐 아니라 **텍스트도 노드이고**, 코드에는 없던 `<head>`도 파서가 자동으로 만들어줌

### 노드의 종류

| 종류 | 설명 | 예시 |
|---|---|---|
| Document | 트리의 최상위 | `document` |
| Element | HTML 태그 | `<div>`, `<p>` |
| Text | 태그 안의 텍스트 (공백/줄바꿈 포함) | `"Hello "` |
| Comment | 주석 | `<!-- ... -->` |

### 파싱의 특징

- **점진적(Incremental) 파싱**: 파일 전체를 다 받을 때까지 기다리지 않고, **바이트가 도착하는 대로** 파싱해서 DOM을 만들어감
- **오류에 관대함**: 닫는 태그가 빠지거나 잘못 중첩되어도 에러를 내지 않고 규칙에 따라 **보정해서 항상 DOM을 만들어냄**
- **`<script>`를 만나면 파싱이 멈춤**: 스크립트가 DOM을 바꿀 수 있어서, 동기 스크립트는 다운로드/실행이 끝나야 파싱을 계속함 (Parser-blocking, 5주차). `defer`/`async`는 이를 피함
- **Preload Scanner**: 파서가 멈춰 있어도 별도로 HTML을 훑으며 이후에 필요한 리소스를 미리 요청함 (5주차)

### 주요 이벤트

| 이벤트 | 시점 |
|---|---|
| `DOMContentLoaded` | HTML 파싱이 끝나고 DOM이 완성된 시점 (`defer` 스크립트 실행 후). 이미지, 스타일시트 로딩을 기다리지 않음 |
| `load` | 이미지, 스타일시트 등 **모든 리소스 로딩까지 끝난** 시점 |

```js
document.addEventListener('DOMContentLoaded', () => {
  // DOM에 접근해도 안전한 시점
});

window.addEventListener('load', () => {
  // 이미지 크기 등 리소스가 필요한 작업
});
```

### DOM API

```js
const el = document.querySelector('.item');            // CSS 선택자로 조회
const div = document.createElement('div');             // 노드 생성
div.textContent = 'hello';                             // 텍스트 설정 (HTML로 해석하지 않음)
parent.appendChild(div);                               // 트리에 추가
div.remove();                                          // 트리에서 제거
```

| 조회 메서드 | 반환 | 특징 |
|---|---|---|
| `getElementById` | 요소 하나 | 가장 빠름 |
| `querySelector` / `querySelectorAll` | 요소 / `NodeList` | CSS 선택자 사용, `querySelectorAll`은 **정적(Static)** 목록 |
| `getElementsByClassName` | `HTMLCollection` | **살아있는(Live)** 목록. DOM이 바뀌면 함께 갱신됨 |

### DOM 조작은 비싸다

- DOM을 바꾸면 이후의 **스타일 계산, 레이아웃, 페인트가** 다시 일어날 수 있음 (5~7장)
- 따라서 DOM 변경은 **한 번에 모아서** 수행하는 것이 유리함

```js
// 나쁨: 반복마다 DOM에 직접 추가
for (const item of items) {
  list.appendChild(createItem(item));
}

// 좋음: 메모리상의 DocumentFragment에 모았다가 한 번에 추가
const fragment = document.createDocumentFragment();
for (const item of items) {
  fragment.appendChild(createItem(item));
}
list.appendChild(fragment);
```

- React 같은 라이브러리의 Virtual DOM도 **변경을 모아서 실제 DOM에는 최소한만 반영하려는** 목적 (5주차 Virtual DOM diffing)
- `innerHTML`로 사용자 입력을 넣으면 XSS 위험이 있음 (7주차)

### 개발자 도구의 Elements vs 소스 보기

- **소스 보기**: 서버가 보낸 **원본 HTML**
- **Elements 패널**: JS로 변경되고 파서가 보정한 **현재 DOM**
- 두 내용이 다를 수 있음 (SPA는 소스에는 빈 `<div id="root">`뿐이고, DOM에는 JS가 만든 내용이 채워짐)

---

## 2. CSSOM

### 개념

- **CSSOM(CSS Object Model)**: 브라우저가 CSS를 파싱해서 만든 **스타일 규칙의 트리 구조**
- 각 노드에 최종적으로 어떤 스타일이 적용되는지 계산하는 데 사용됨
- DOM과 마찬가지로 `Bytes → Characters → Tokens → Nodes → CSSOM` 과정을 거침

```css
body { font-size: 16px; }
p    { color: gray; }
span { color: red; }
```

```
body { font-size: 16px }
 └─ p { color: gray }                      (font-size: 16px 상속)
     └─ span { color: red }                (font-size: 16px 상속)
```

- 스타일은 **부모에서 자식으로 상속되는** 속성(`font-size`, `color` 등)이 있어서 트리 구조로 표현됨

### Cascade (스타일 결정 규칙)

- 같은 요소에 여러 규칙이 겹칠 때 어느 것을 적용할지 정하는 규칙 (CSS의 C가 Cascading)
- 우선순위: **출처/중요도 → 명시도(Specificity) → 선언 순서**

| 선택자 | 명시도 |
|---|---|
| 인라인 스타일 (`style="..."`) | 가장 높음 |
| ID 선택자 (`#id`) | 높음 |
| 클래스, 속성, 가상 클래스 (`.class`, `[type]`, `:hover`) | 중간 |
| 요소, 가상 요소 (`div`, `::before`) | 낮음 |

- `!important`는 명시도를 무시하고 우선함
- 명시도가 같으면 **나중에 선언된 규칙이** 이김

### Computed Style

- 상속, 캐스케이드, 상대 단위 변환을 거쳐 **각 요소에 최종 계산된 스타일 값**
- 예: `font-size: 2em` → 부모의 `16px`를 기준으로 `32px`로 계산

```js
const style = getComputedStyle(document.querySelector('p'));
console.log(style.fontSize); // '16px' (계산된 최종 값)
```

### CSS는 렌더링을 차단한다 (Render-blocking)

- 브라우저는 **CSSOM이 완성되기 전에는 화면을 그리지 않음**
- 이유: CSS 규칙은 뒤쪽에서 앞의 규칙을 덮어쓸 수 있어서, **모든 CSS를 읽어야 최종 스타일이 확정됨**. 스타일 없는 화면이 먼저 보였다가 바뀌는 현상(FOUC)을 막기 위함

```
HTML 파싱 ──────────────► DOM 완성
CSS 다운로드/파싱 ──────► CSSOM 완성
                              │
                              ▼  둘 다 완성되어야 렌더링 시작
```

- DOM과 CSSOM은 **병렬로 만들어지지만**, 렌더링은 둘 다 필요함

### CSS와 JS의 관계

- JS는 `getComputedStyle`처럼 **스타일을 읽을 수 있기 때문에**, 앞에 있는 CSS가 아직 로딩 중이면 **스크립트 실행이 CSS 로딩이 끝날 때까지 대기함**
- 그래서 `<link rel="stylesheet">`를 `<script>`보다 **앞에** 두는 것이 일반적

```html
<head>
  <link rel="stylesheet" href="main.css">   <!-- 먼저 -->
  <script src="app.js" defer></script>       <!-- 나중에 -->
</head>
```

### 렌더링 차단 줄이기: media 속성

```html
<link rel="stylesheet" href="base.css">                         <!-- 항상 렌더링 차단 -->
<link rel="stylesheet" href="print.css" media="print">          <!-- 화면 렌더링은 차단하지 않음 -->
<link rel="stylesheet" href="wide.css" media="(min-width: 1200px)"> <!-- 조건에 맞지 않으면 차단하지 않음 -->
```

- 조건에 해당하지 않는 CSS는 렌더링을 막지 않음 (다운로드는 낮은 우선순위로 진행)

### 스타일 재계산 (Recalculate Style)

- JS로 클래스, 인라인 스타일, `<style>`을 바꾸면 브라우저는 **영향받는 요소의 Computed Style을 다시 계산함**
- 이 단계 이후에 Layout, Paint가 이어질 수 있음 (4장)
- 선택자 매칭은 **오른쪽에서 왼쪽으로** 진행됨 (`.nav a`는 모든 `a`를 찾고 조상에 `.nav`가 있는지 확인). 극단적으로 복잡한 선택자만 피하면 대부분 문제되지 않으며, **한 번에 스타일이 바뀌는 요소의 수가** 더 큰 영향을 줌

---

## 3. Render Tree

### 개념

- **Render Tree**: DOM과 CSSOM을 합쳐서 만든, **화면에 실제로 그려질 요소만** 담은 트리
- 각 노드는 **DOM 노드 + 계산된 스타일을** 가짐
- 이후 Layout, Paint 단계의 입력이 됨 (Chrome에서는 이 트리를 Layout Tree라고도 부름)

```
DOM                                  Render Tree
html
├─ head ─────────────────────────✕   (화면에 안 그려지는 요소라 제외)
└─ body ─────────────────────────►   body
   ├─ h1 ────────────────────────►     ├─ h1
   └─ p ─────────────────────────►     └─ p
      └─ span (display: none) ───✕         (display: none이라 제외)
```

### 생성 과정

1. DOM 루트부터 **보이는 노드를** 순회
2. 각 노드에 맞는 CSSOM 규칙을 찾아서 **스타일을 결합**
3. 화면에 그려질 노드만 Render Tree에 포함

### 포함되는 것과 제외되는 것

| 구분 | 요소 |
|---|---|
| 제외 | `<head>`, `<script>`, `<meta>` 등 화면에 그려지지 않는 요소 |
| 제외 | `display: none`인 요소 (자손 포함) |
| 포함 | `visibility: hidden`, `opacity: 0` 요소 (그려지지 않을 뿐 공간은 차지) |
| 포함 | `::before`, `::after` 같은 가상 요소 (DOM에는 없지만 Render Tree에는 있음) |

### 숨기는 방법별 차이

| 속성 | Render Tree | 공간 차지 | 클릭 등 이벤트 |
|---|---|---|---|
| `display: none` | 제외 | 차지하지 않음 | 불가 |
| `visibility: hidden` | 포함 | 차지함 | 불가 |
| `opacity: 0` | 포함 | 차지함 | **가능** |

- `display: none`은 Render Tree에서 빠지므로 **Layout 대상에서도 제외됨** (그래서 `none`과 `block` 전환은 레이아웃을 다시 계산함)
- `visibility: hidden`, `opacity: 0`은 공간을 유지하므로 다른 요소의 위치는 바뀌지 않음

### DOM과 Render Tree의 차이

| | DOM | Render Tree |
|---|---|---|
| 목적 | 문서의 구조와 내용 표현, JS의 조작 대상 | 화면에 그릴 대상과 그 스타일 |
| 포함 범위 | 모든 노드 (head, 주석, 숨김 요소 포함) | 보이는 요소만 |
| 스타일 정보 | 없음 | 있음 |
| 가상 요소 | 없음 | 있음 |

### 이후 단계와의 관계

```
Render Tree ──► Layout (위치/크기 계산) ──► Paint (그리기) ──► Composite (합성)
```

- DOM이나 CSSOM이 바뀌면 Render Tree도 갱신되고, **무엇이 바뀌었는지에 따라 이후 단계가 다시 실행됨** (5~7장)

---

## 4. Critical Rendering Path

### 개념

- **Critical Rendering Path(CRP)**: 브라우저가 HTML, CSS, JS를 받아서 **화면에 첫 픽셀을 그리기까지 거치는 일련의 단계**
- "Critical"은 **첫 화면 렌더링에 반드시 필요한** 리소스와 단계를 의미함
- 이 경로를 짧게 만드는 것이 **초기 로딩 성능 최적화의** 핵심

### 단계

```
HTML ──► DOM ──────┐
                   ├──► Render Tree ──► Layout ──► Paint ──► Composite ──► 화면
CSS ───► CSSOM ────┘
              ▲
JS ───────────┘  (실행 중 DOM/CSSOM을 변경할 수 있음)
```

| 단계 | 하는 일 | 다루는 장 |
|---|---|---|
| 1. DOM 구성 | HTML을 파싱해서 DOM 생성 | 1장 |
| 2. CSSOM 구성 | CSS를 파싱해서 CSSOM 생성 | 2장 |
| 3. Render Tree 구성 | DOM + CSSOM 결합 | 3장 |
| 4. Layout | 각 요소의 위치와 크기 계산 | 5장 |
| 5. Paint | 요소를 그리기 위한 명령 생성 및 래스터화 | 6장 |
| 6. Composite | 레이어를 합성해서 화면에 표시 | 7장 |

- Chrome의 내부 용어로는 **Parse → Style → Layout → Paint → Composite로** 나누고, Render Tree 생성은 Style/Layout 단계에 걸쳐 일어남. 자료마다 단계 이름이 조금씩 다르지만 흐름은 같음

### 렌더링을 막는 요소

| 요소 | 영향 | 줄이는 방법 |
|---|---|---|
| CSS | CSSOM이 완성되기 전엔 렌더링 불가 (Render-blocking) | 필요한 CSS만 로드, `media` 속성으로 분리, 첫 화면 CSS 인라인 |
| 동기 JS (`<script>`) | HTML 파싱 중단 (Parser-blocking) | `defer`/`async`, 파일 하단 배치, 코드 분할 |
| 웹 폰트 | 폰트가 오기 전까지 글자가 안 보일 수 있음 | `font-display: swap`, 폰트 preload (5주차) |

### 측정 요소

| 용어 | 의미 |
|---|---|
| Critical Resources | 첫 렌더링을 막을 수 있는 리소스의 수 |
| Critical Path Length | 첫 렌더링까지 필요한 **네트워크 왕복(Round Trip) 횟수** |
| Critical Bytes | 첫 렌더링에 필요한 **총 데이터 크기** |

- 세 값이 작을수록 첫 화면이 빨리 나타남

### 렌더링 관련 성능 지표

| 지표 | 의미 | 관련 단계 |
|---|---|---|
| FCP (First Contentful Paint) | 텍스트/이미지 등 **첫 콘텐츠가 그려진** 시점 | CRP 전체 |
| LCP (Largest Contentful Paint) | 화면에서 **가장 큰 콘텐츠가 그려진** 시점 | 리소스 로딩, 렌더링 차단 |
| CLS (Cumulative Layout Shift) | 로딩 중 **레이아웃이 예기치 않게 밀린 정도** | Layout (5장) |
| INP (Interaction to Next Paint) | 사용자 입력 후 **다음 화면 갱신까지의 지연** | 메인 스레드 점유, 렌더링 (9주차) |

```js
// Paint 시점을 코드로 확인
new PerformanceObserver((list) => {
  for (const entry of list.getEntries()) {
    console.log(entry.name, Math.round(entry.startTime)); // 'first-paint', 'first-contentful-paint'
  }
}).observe({ type: 'paint', buffered: true });
```

### 최적화 전략

| 전략 | 방법 | 줄이는 대상 |
|---|---|---|
| 첫 화면 CSS 인라인 | Above-the-fold에 필요한 CSS만 `<style>`로 직접 포함하고, 나머지는 비동기 로드 | Critical Path Length |
| JS 지연 | `defer`/`async`, 코드 분할(`import()`) | Critical Resources |
| 리소스 힌트 | `preconnect`, `preload`로 발견/연결 시점 앞당김 (5주차) | Critical Path Length |
| 용량 줄이기 | 압축(gzip/brotli), minify, 불필요한 CSS/JS 제거 | Critical Bytes |
| 캐싱 | 정적 파일 장기 캐싱 (5주차 HTTP Cache) | 재방문 시 전체 |
| 이미지 | 첫 화면 밖은 `loading="lazy"`, `width`/`height` 지정 | Critical Bytes, CLS |

```html
<head>
  <!-- 첫 화면에 필요한 최소 CSS는 인라인 -->
  <style>/* critical css */</style>

  <!-- 나머지 CSS는 렌더링을 막지 않게 로드 -->
  <link rel="preload" href="/main.css" as="style" onload="this.rel='stylesheet'">

  <!-- JS는 파싱을 막지 않게 -->
  <script src="/app.js" defer></script>
</head>
```

### 한 프레임 안에서의 렌더링

- 화면이 갱신될 때(보통 초당 60회, 약 16.7ms 간격) 브라우저는 한 프레임 동안 아래 순서로 작업함 (9주차의 이벤트 루프 렌더링 단계)

```
JS ──► Style ──► Layout ──► Paint ──► Composite
(이벤트,   (스타일      (위치/크기    (그리기)   (레이어 합성)
 rAF)     재계산)       계산)
```

- 이 모든 과정이 16.7ms 안에 끝나지 못하면 **프레임이 건너뛰어져 화면이 끊겨 보임(Jank)**
- 그래서 **뒤쪽 단계(Layout, Paint)를 건너뛸수록** 빠름 → 5~7장에서 어떤 속성이 어느 단계부터 다시 실행시키는지 확인

### 확인 도구

- **Chrome DevTools Performance 패널**: 프레임별로 Scripting, Rendering(Style, Layout), Painting, Compositing 시간을 확인
- **Lighthouse**: FCP, LCP, CLS 측정과 렌더링 차단 리소스 진단
- **Network 패널의 Waterfall**: 어떤 리소스가 렌더링을 막고 있는지 확인

---

## 5. Layout

### 개념

- **Layout(레이아웃)**: Render Tree의 각 노드가 **화면의 어디에, 얼마나 큰 크기로** 놓일지를 계산하는 단계. 과거에는 **Reflow라고** 부름
- 입력: Render Tree + 뷰포트(화면) 크기
- 출력: 각 요소의 **정확한 좌표와 크기(박스 정보)**
- `%`, `em`, `vw` 같은 상대 단위가 이 단계에서 **실제 픽셀 값으로 확정됨**

### 박스 모델

- 모든 요소는 사각형 박스로 계산됨

```
┌────────────────── margin ───────────────────┐
│ ┌──────────────── border ─────────────────┐ │
│ │ ┌────────────── padding ──────────────┐ │ │
│ │ │                                     │ │ │
│ │ │              content                │ │ │
│ │ │                                     │ │ │
│ │ └─────────────────────────────────────┘ │ │
│ └─────────────────────────────────────────┘ │
└─────────────────────────────────────────────┘
```

| 영역 | 설명 |
|---|---|
| content | 실제 내용 (`width`, `height`) |
| padding | 내용과 테두리 사이의 안쪽 여백 |
| border | 테두리 |
| margin | 테두리 바깥의 다른 요소와의 여백 |

### 뷰포트와 레이아웃

```html
<meta name="viewport" content="width=device-width, initial-scale=1">
```

- 모바일에서 이 설정이 없으면 브라우저가 가상의 넓은 화면(약 980px)을 기준으로 레이아웃한 뒤 축소해서 보여줌
- **뷰포트 너비를 기준으로 레이아웃이 계산되므로**, 반응형 디자인(미디어 쿼리)의 기반이 됨

### 계산 방식

- 일반적으로 **부모의 너비를 기준으로 자식의 너비가** 정해지고, **자식들의 높이가 쌓여서 부모의 높이가** 정해짐
- 그래서 한 요소의 크기가 바뀌면 **자식, 형제, 부모에게 연쇄적으로 영향을** 줄 수 있음
- 첫 레이아웃은 전체에 대해 수행하고, 이후 변경으로 다시 계산하는 것을 **Reflow라고** 함 (영향받은 부분만 다시 계산하려고 하지만, 영향 범위가 넓으면 비용이 커짐)

### Reflow가 발생하는 경우

| 원인 | 예시 |
|---|---|
| 크기/위치 속성 변경 | `width`, `height`, `margin`, `padding`, `border`, `top`, `left`, `font-size` |
| DOM 추가/삭제/이동 | `appendChild`, `remove` |
| 내용 변경 | 텍스트가 바뀌어 줄 바꿈/크기가 달라짐 |
| 뷰포트 변경 | 창 크기 조절, 기기 회전 |
| 폰트 로딩 완료 | 웹 폰트 교체로 글자 크기가 달라짐 |
| 레이아웃 값 읽기 | `offsetWidth`, `getBoundingClientRect()` 등 (아래 참고) |

### Forced Synchronous Layout (강제 동기 레이아웃)

- 브라우저는 스타일 변경을 바로 처리하지 않고 **모아뒀다가 프레임에서 한 번에** 레이아웃함
- 그런데 JS가 `offsetWidth` 같은 **레이아웃 결과를 읽으면**, 최신 값을 알려주기 위해 **즉시 레이아웃을 실행해야** 함

| 레이아웃을 강제하는 값 읽기 |
|---|
| `offsetTop`, `offsetLeft`, `offsetWidth`, `offsetHeight` |
| `clientTop`, `clientWidth`, `clientHeight` |
| `scrollTop`, `scrollHeight` |
| `getBoundingClientRect()` |
| `getComputedStyle(el).width` 등 레이아웃에 의존하는 값 |

### Layout Thrashing

- **읽기와 쓰기를 번갈아 반복하면** 매 반복마다 강제 레이아웃이 일어나서 성능이 크게 떨어짐

```js
// 나쁨: 쓰기(style 변경) → 읽기(offsetWidth) 반복 → 반복마다 레이아웃 재계산
for (const el of items) {
  el.style.width = box.offsetWidth + 'px';
}

// 좋음: 읽기를 먼저 한꺼번에 하고, 쓰기를 나중에 한꺼번에
const width = box.offsetWidth;                    // 읽기 1번
for (const el of items) {
  el.style.width = width + 'px';                  // 쓰기만 반복 (레이아웃은 프레임에서 1번)
}
```

- 원칙: **읽기와 쓰기를 분리하고**, 쓰기는 `requestAnimationFrame` 안에서 모아서 처리

### Layout Shift (CLS)

- 이미 보이던 콘텐츠가 **뒤늦게 로드된 요소 때문에 밀리는** 현상. 의도치 않은 클릭을 유발해서 사용성이 나빠짐

| 원인 | 해결 |
|---|---|
| 크기를 모르는 이미지/영상 | `width`/`height` 속성 또는 `aspect-ratio`로 **공간을 미리 확보** |
| 나중에 삽입되는 광고, 배너 | 영역의 크기를 미리 예약 |
| 웹 폰트 교체로 글자 크기 변화 | `font-display`, 대체 폰트 크기 조정 |
| 위쪽에 동적으로 콘텐츠 삽입 | 기존 콘텐츠 아래/화면 밖에 삽입 |

```css
img { aspect-ratio: 16 / 9; width: 100%; height: auto; }  /* 로드 전에도 공간이 확보됨 */
```

### 레이아웃 비용 줄이기

- **DOM 크기를 줄이고** 깊은 중첩을 피함 (노드가 많을수록 레이아웃이 오래 걸림)
- **애니메이션에 레이아웃 속성(`top`, `left`, `width`)을 쓰지 않고** `transform`을 사용 (7장)
- 변경 영향을 격리: `position: absolute/fixed`로 흐름에서 분리하거나, CSS `contain`으로 레이아웃 범위를 제한
- 화면 밖 콘텐츠는 `content-visibility: auto`로 렌더링을 미룸 (브라우저 지원 확인 필요)

```css
.card-list > li {
  content-visibility: auto;              /* 화면 밖이면 레이아웃/페인트를 건너뜀 */
  contain-intrinsic-size: 0 200px;       /* 건너뛴 동안의 예상 높이 */
}
```

---

## 6. Paint

### 개념

- **Paint(페인트)**: Layout으로 정해진 위치/크기를 바탕으로 요소를 **실제 픽셀로 그리는** 단계. 과거에는 **Repaint라고** 부름
- 색상, 배경, 테두리, 그림자, 텍스트, 이미지 같은 **시각적 속성을** 그림
- 두 부분으로 나뉨

| 부분 | 하는 일 |
|---|---|
| Paint | 무엇을 어떤 순서로 그릴지 **그리기 명령 목록(Display List)을** 생성 |
| Raster | 그리기 명령을 실제 **픽셀(비트맵)로** 변환 (보통 타일 단위로 GPU가 처리) |

### 그리는 순서

- 요소는 **겹치는 순서(Stacking Order)에** 따라 뒤에서 앞으로 그려짐
- 대략의 순서: 배경 → 테두리 → 자식 요소 → 텍스트 → 아웃라인
- `z-index`와 **Stacking Context가** 서로 다른 요소 간의 앞뒤 순서를 결정함

| Stacking Context를 만드는 조건 |
|---|
| `position`이 지정된 요소 + `z-index` 값 |
| `opacity`가 1보다 작음 |
| `transform`, `filter`가 지정됨 |
| `will-change`가 지정됨 |

### Repaint가 발생하는 경우

- **레이아웃에 영향을 주지 않고 모양만 바꾸는** 속성이 변경될 때

| 속성 | 예시 |
|---|---|
| 색상 | `color`, `background-color`, `border-color` |
| 장식 | `box-shadow`, `border-radius`, `outline` |
| 배경/이미지 | `background-image`, `background-position` |
| 가시성 | `visibility` |

- **Layout이 일어나면 Paint도 항상 뒤따름** (위치와 크기가 바뀌면 다시 그려야 하므로)
- 반대로 Paint만 일어나는 변경은 Layout을 건너뛰므로 **더 가벼움**

### Paint 비용

- 다시 그려야 하는 **면적이 넓을수록**, **효과가 복잡할수록** 비용이 커짐

| 비용이 큰 경우 | 이유 |
|---|---|
| `box-shadow`, `filter: blur()`, 큰 `border-radius` | 픽셀 단위 연산이 많음 |
| 큰 배경 이미지, 그라디언트 | 래스터화할 면적이 넓음 |
| 화면 전체에 영향을 주는 변경 | 다시 그릴 영역이 큼 |

### 레이어 (Layer)

- 브라우저는 페이지를 **하나의 그림이 아니라 여러 레이어로 나눠서** 그릴 수 있음
- 레이어별로 따로 그려두면, **변경된 레이어만 다시 그리고** 나머지는 재사용할 수 있음 → 7장 Composite

```
 Layer 3   ┌─────────┐   ← 모달
 Layer 2   │ 카드 목록 │   ← 스크롤 영역
 Layer 1   │ 배경/헤더 │   ← 기본 레이어
           └─────────┘
```

### Paint 줄이기

- **변경되는 영역을 작게**: 전체 컨테이너가 아니라 필요한 요소만 변경
- 복잡한 효과(그림자, 블러)를 **자주 변하는 요소에 사용하지 않음**
- 자주 움직이는 요소를 **별도 레이어로 분리해서** 주변을 다시 그리지 않게 함 (7장)
- 숨길 때 `display: none` 대신 `visibility`/`opacity`를 쓰면 레이아웃은 피할 수 있음 (3장)

### 개발자 도구로 확인

- **Rendering → Paint flashing**: 다시 그려지는 영역이 **초록색으로 깜빡임**. 스크롤이나 호버 시 불필요하게 넓은 영역이 깜빡이면 문제
- **Rendering → Layout Shift Regions**: 레이아웃이 밀린 영역을 파란색으로 표시
- **Performance 패널**: Paint에 걸린 시간 확인

---

## 7. Composite

### 개념

- **Composite(합성)**: 따로 그려진 **여러 레이어를 올바른 순서로 겹쳐서 최종 화면을** 만드는 단계
- 이 단계는 **GPU가** 담당하고, **메인 스레드가 아닌 Compositor 스레드에서** 처리됨 (2주차 브라우저 Multi-Process 구조의 GPU Process)

```
Main Thread        JS ─► Style ─► Layout ─► Paint ─► (레이어 정보를 전달)
                                                           │
Compositor Thread                                          ▼
                                       레이어 합성 ─► GPU (Raster) ─► 화면
```

### 왜 레이어로 나누는가

- 레이어 안의 내용이 바뀌지 않고 **위치, 크기, 투명도만 바뀌는 경우**, 다시 그리지 않고 **합성만 다시 하면 됨**
- Layout과 Paint는 메인 스레드에서 일어나지만, **합성은 메인 스레드와 독립적이라** JS가 바빠도 영향을 덜 받음

| | 메인 스레드 (Layout/Paint) | Compositor (Composite) |
|---|---|---|
| 처리 위치 | CPU, 메인 스레드 | GPU, Compositor 스레드 |
| JS가 바쁠 때 | 함께 지연됨 | 계속 동작 가능 |
| 비용 | 상대적으로 큼 | 작음 |

### Composite만 일어나는 속성

- **`transform`과 `opacity`** 변경은 (레이어로 분리된 요소라면) **Layout도 Paint도 건너뛰고 Composite만** 일어남

```css
/* 나쁨: left를 바꾸면 매 프레임 Layout → Paint → Composite */
@keyframes move-bad {
  to { left: 300px; }
}

/* 좋음: transform은 Composite만 */
@keyframes move-good {
  to { transform: translateX(300px); }
}
```

```js
// JS 애니메이션도 transform으로
function animate() {
  x += 2;
  box.style.transform = `translateX(${x}px)`;   // 레이아웃 없이 합성만
  requestAnimationFrame(animate);
}
requestAnimationFrame(animate);
```

### 레이어가 만들어지는 경우

| 조건 | 설명 |
|---|---|
| `transform`/`opacity` 애니메이션 진행 중 | 브라우저가 자동으로 레이어로 승격 |
| 3D transform (`translateZ`, `translate3d`) | 레이어 생성 |
| `will-change: transform` / `opacity` | **미리 레이어로 분리하라는 힌트** |
| `<video>`, `<canvas>`(WebGL) | 별도 레이어 |
| `position: fixed` 일부, 스크롤 영역 | 브라우저가 필요하다고 판단할 때 |

```css
.modal { will-change: transform; }   /* 곧 움직일 요소를 미리 레이어로 준비 */
```

### 레이어의 비용

- 레이어마다 **GPU 메모리를** 차지하고, 레이어가 많을수록 합성 비용도 늘어남
- 레이어를 무분별하게 만들면 오히려 성능이 나빠짐 (**Layer Explosion**)

| 주의 | 설명 |
|---|---|
| `will-change` 남용 | 필요한 요소에만, 애니메이션이 끝나면 제거 |
| 화면보다 큰 레이어 | 메모리 낭비 |
| 겹침으로 인한 암묵적 레이어 생성 | 레이어 위에 겹치는 요소가 의도치 않게 레이어가 됨 |

### 스크롤과 Composite

- 스크롤도 **Compositor가 처리할** 수 있어서, 메인 스레드가 바쁘더라도 스크롤 자체는 부드럽게 유지될 수 있음
- 단, `touchmove`, `wheel` 같은 이벤트에 **리스너가 있으면, 그 리스너가 `preventDefault()`로 스크롤을 막을 수 있어서** 브라우저가 핸들러 실행을 기다려야 함 → 스크롤이 지연됨
- 스크롤을 막지 않는다고 알려주면(`passive`) 기다릴 필요가 없음 (9주차 Event Loop와 연결)

```js
window.addEventListener('touchmove', onTouchMove, { passive: true });
```

### 개발자 도구로 확인

- **Layers 패널**: 레이어의 개수, 크기, 생성된 이유(Compositing reasons) 확인
- **Rendering → Layer borders**: 레이어 경계를 주황색/초록색으로 표시
- **Performance 패널**: Composite Layers 시간 확인

### 속성별로 다시 실행되는 단계

| 변경하는 속성 | Layout | Paint | Composite | 비용 |
|---|---|---|---|---|
| `width`, `height`, `margin`, `padding`, `top`, `left`, `font-size`, `display` | O | O | O | 큼 |
| `color`, `background-color`, `box-shadow`, `border-color`, `visibility` | X | O | O | 중간 |
| `transform`, `opacity` (레이어로 분리된 경우) | X | X | O | 작음 |

- 위 표는 Chrome 기준의 일반적인 경향이고, 브라우저와 상황에 따라 다를 수 있음
- 애니메이션과 자주 바뀌는 UI는 **가능한 한 `transform`과 `opacity`만** 사용하도록 설계

---

## 8. 정리

| 개념 | 핵심 |
|---|---|
| DOM | HTML을 파싱해서 만든 객체 트리. 점진적 파싱, `<script>`를 만나면 파싱 중단 |
| CSSOM | CSS를 파싱해서 만든 스타일 트리. 완성되기 전에는 렌더링하지 않음(Render-blocking) |
| Render Tree | DOM + CSSOM에서 **보이는 요소만** 추려낸 트리 (`display: none` 제외, `visibility: hidden`은 포함) |
| Critical Rendering Path | HTML/CSS/JS에서 첫 화면이 그려지기까지의 단계. 렌더링 차단 요소를 줄이는 것이 최적화의 핵심 |
| Layout | 각 요소의 위치와 크기 계산 (Reflow). 읽기/쓰기 반복은 Layout Thrashing |
| Paint | 색상, 그림자 등 시각 속성을 픽셀로 그림 (Repaint). 면적과 효과가 복잡할수록 비쌈 |
| Composite | 레이어를 GPU에서 합성. `transform`/`opacity`는 이 단계만 실행 |

### 변경 종류에 따라 다시 실행되는 단계

```
Layout 속성 변경   : Style ─► Layout ─► Paint ─► Composite    (가장 비쌈)
Paint 속성 변경    : Style ──────────► Paint ─► Composite
Composite 속성 변경: Style ──────────────────► Composite       (가장 가벼움)
```

### 렌더링 최적화 체크리스트

| 구분 | 점검 항목 |
|---|---|
| 초기 로딩 | 렌더링을 막는 CSS/JS를 줄였는가 (`defer`, `media`, 첫 화면 CSS 인라인) |
| DOM | DOM 크기가 과하지 않은가, 변경을 모아서 처리하는가 (`DocumentFragment`) |
| Layout | 읽기/쓰기를 분리했는가, 이미지에 `width`/`height`를 지정했는가 |
| Paint | 자주 바뀌는 요소에 무거운 효과(그림자, 블러)를 쓰지 않았는가 |
| 애니메이션 | `top`/`left` 대신 `transform`/`opacity`를 쓰는가, `will-change`를 남용하지 않는가 |
| 측정 | Performance 패널, Lighthouse로 FCP/LCP/CLS를 확인했는가 |