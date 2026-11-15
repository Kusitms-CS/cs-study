# 10주차 - Browser / Rendering

## 0. 브라우저 렌더링 파이프라인 한눈에 보기

- 브라우저가 서버로부터 받은 **HTML, CSS, JS를 화면의 픽셀로 바꾸는 과정**을 렌더링 파이프라인이라고 함
- 이번 주 키워드는 모두 이 파이프라인의 단계에 해당함

```
 HTML ──parse──▶ DOM ───┐
                        ├──▶ Render Tree ──▶ Layout ──▶ Paint ──▶ Composite ──▶ Screen
 CSS  ──parse──▶ CSSOM ─┘     (Style)        (Reflow)   (Raster)   (Layers)
```

| 단계         | 하는 일                                                         | 결과물                   |
| ------------ | --------------------------------------------------------------- | ------------------------ |
| Parsing      | HTML → **DOM**, CSS → **CSSOM**                                  | DOM, CSSOM               |
| Style        | DOM과 CSSOM을 결합해 **화면에 보이는 요소와 스타일 계산**       | Render Tree              |
| Layout       | 각 요소의 **위치와 크기(기하학 정보)** 계산                     | Layout 정보 (박스 좌표)  |
| Paint        | 요소를 **어떤 순서로 어떻게 그릴지** 기록하고 픽셀로 변환       | 레이어별 비트맵          |
| Composite    | 여러 레이어를 **올바른 순서로 합쳐** 최종 화면 생성             | 화면에 표시되는 프레임   |

### 📍 렌더러 프로세스의 스레드

- Chrome은 탭마다 **렌더러 프로세스**를 두고, 그 안에서 여러 스레드가 파이프라인을 나눠 처리함 (2주차 브라우저 Multi-Process 구조 참고)

| 스레드              | 담당                                                                 |
| ------------------- | -------------------------------------------------------------------- |
| Main Thread         | HTML/CSS 파싱, **JS 실행**, Style, Layout, Paint 기록                |
| Compositor Thread   | 레이어를 타일로 나누고 **합성(Composite)**, 스크롤 처리              |
| Raster Thread       | 타일 단위로 **래스터화(픽셀로 변환)**                                |
| GPU Process         | 합성된 결과를 **실제 화면에 출력**                                   |

- JS 실행과 Layout/Paint가 **같은 메인 스레드**에서 일어나므로, JS가 오래 실행되면 렌더링도 함께 밀림 (9주차 Event Loop의 렌더링 단계)

---

## 1. DOM (Document Object Model)

- **HTML 문서를 브라우저와 JS가 이해하고 조작할 수 있도록 트리 구조의 객체로 표현한 것**
- HTML의 각 태그, 텍스트, 주석이 **노드(Node)** 가 되고, 부모-자식 관계로 연결된 **트리 구조**를 이룸
- DOM은 JS 언어의 일부가 아니라 **브라우저가 제공하는 Web API** (`document.querySelector` 등)

### 📍 HTML → DOM 변환 과정

```
 Bytes ──▶ Characters ──▶ Tokens ──▶ Nodes ──▶ DOM Tree
 3C 62..    <body>...      StartTag   Element   html
                           EndTag     Text       └ body
                           Text                    └ p
```

| 단계            | 설명                                                                           |
| --------------- | ------------------------------------------------------------------------------ |
| 바이트 → 문자   | 서버에서 받은 바이트를 **인코딩(UTF-8 등)** 에 따라 문자로 변환                |
| 토큰화          | 문자열을 `<html>`, `<body>` 같은 **시작 태그, 종료 태그, 텍스트 토큰**으로 분리 |
| 노드 생성       | 토큰을 속성과 규칙을 가진 **노드 객체**로 변환                                 |
| DOM 트리 구성   | 노드들의 **부모-자식 관계**를 연결해 트리 완성                                 |

- HTML은 전체를 다 받기 전에도 **받은 만큼 점진적으로(incremental) 파싱**함
- HTML 파서는 관대해서 닫는 태그 누락 같은 **문법 오류를 자동으로 보정**함 (`<table>`에 `<tbody>` 자동 추가 등)

### 📍 DOM 노드의 종류

| 노드 타입     | 설명                                  | 예시                       |
| ------------- | ------------------------------------- | -------------------------- |
| Document      | DOM 트리의 **최상위 루트 노드**       | `document`                 |
| Element       | HTML **태그**에 해당하는 노드         | `<div>`, `<p>`             |
| Text          | 태그 안의 **텍스트**                  | `"Hello"`                  |
| Attribute     | 요소의 **속성** (트리의 자식은 아님)  | `class="box"`              |
| Comment       | **주석**                              | `<!-- comment -->`         |

### 📍 파싱을 멈추는 `<script>`

- 파서가 `<script>`를 만나면 JS가 DOM을 변경할 수 있으므로 **DOM 생성을 멈추고 JS를 다운로드·실행**한 뒤 파싱을 재개함 (Parser Blocking)
- 그래서 `<script>`는 `<body>` 끝에 두거나 **`defer` / `async`** 를 사용함 (5주차 Resource Loading 참고)
- 브라우저는 파싱이 멈춘 동안에도 **Preload Scanner**가 뒤쪽 HTML을 미리 훑어 CSS, JS, 이미지 같은 리소스 요청을 먼저 시작함

### 💡 HTML 소스와 DOM은 같은 건가요?

**⇒ 같지 않음. HTML은 서버에서 받은 텍스트 문서이고, DOM은 브라우저가 그 HTML을 파싱해서 메모리에 만든 객체 트리임. 브라우저가 HTML 오류를 보정하면서 `<tbody>` 같은 노드를 추가하기도 하고, JS로 노드를 추가·삭제하면 HTML 소스는 그대로지만 DOM은 바뀜. 그래서 "페이지 소스 보기"는 원본 HTML을, 개발자 도구의 Elements 탭은 현재 DOM 상태를 보여줌**

### 💡 `DOMContentLoaded`와 `load` 이벤트의 차이

**⇒ `DOMContentLoaded`는 HTML 파싱이 끝나 DOM 트리가 완성되었을 때 발생하고(`defer` 스크립트 실행 후), 이미지·스타일시트·iframe 같은 외부 리소스 로딩은 기다리지 않음. `load`는 이런 외부 리소스까지 모두 로드된 뒤에 발생함. 그래서 DOM 조작만 필요하면 `DOMContentLoaded`에서 처리하는 것이 더 빠르고, 이미지 크기처럼 리소스가 로드되어야 알 수 있는 정보가 필요할 때 `load`를 사용함**

---

## 2. CSSOM (CSS Object Model)

- **CSS를 파싱해서 만든 트리 구조의 객체 모델**로, 각 요소에 어떤 스타일이 적용되는지를 표현
- DOM과 마찬가지로 **바이트 → 문자 → 토큰 → 노드 → CSSOM** 과정을 거쳐 생성됨
- 외부 CSS(`<link>`), `<style>` 태그, 인라인 `style` 속성, **브라우저 기본 스타일(User Agent Stylesheet)** 이 모두 반영됨

### 📍 CSSOM이 트리 구조인 이유

- CSS는 부모의 스타일이 자식에게 **상속(Inheritance)** 되고, 여러 규칙이 겹치면 **캐스케이딩(Cascading)** 규칙에 따라 최종 값이 결정되므로 트리로 표현함

```css
body { font-size: 16px; }
p { font-weight: bold; }
span { color: red; }
```

```
 body (font-size: 16px)
  └ p (font-size: 16px [inherited], font-weight: bold)
     └ span (font-size: 16px [inherited], font-weight: bold [inherited], color: red)
```

### 📍 스타일 우선순위 결정 (Cascade)

| 우선순위 기준            | 설명                                                                                     |
| ------------------------ | ---------------------------------------------------------------------------------------- |
| 출처와 `!important`      | `!important` > 작성자 스타일 > 브라우저 기본 스타일                                      |
| 명시도 (Specificity)     | **인라인 스타일 > ID > 클래스·속성·가상 클래스 > 태그·가상 요소**                        |
| 선언 순서                | 명시도가 같으면 **나중에 선언된 규칙**이 적용                                            |

```
 style="..."   #id    .class / [attr] / :hover    div / ::before
   1,0,0,0    0,1,0,0          0,0,1,0                0,0,0,1
```

- 최종적으로 각 요소에 적용될 값을 계산한 것이 **Computed Style** (`getComputedStyle(el)`로 확인 가능, `em`, `%` 등을 계산한 값)

### 📍 CSS는 Render Blocking 리소스

- 렌더 트리를 만들려면 완성된 CSSOM이 필요하므로, **CSS를 모두 받아 CSSOM을 만들기 전까지 렌더링을 하지 않음**
- CSSOM은 DOM과 달리 **점진적으로 사용할 수 없음** → 뒤에 나오는 규칙이 앞의 규칙을 덮어쓸 수 있어서 전체를 다 파싱해야 최종 스타일을 알 수 있기 때문
- CSS는 HTML 파싱 자체는 막지 않지만, **CSSOM이 완성될 때까지 그 뒤의 JS 실행을 막음** (JS가 `getComputedStyle` 등으로 스타일을 읽을 수 있기 때문)
- `media` 속성이 현재 환경과 맞지 않는 CSS는 **렌더링을 막지 않음** (낮은 우선순위로 다운로드)

```html
<link rel="stylesheet" href="style.css" />                     <!-- 렌더링 차단 -->
<link rel="stylesheet" href="print.css" media="print" />       <!-- 인쇄 시에만 적용 → 차단 X -->
<link rel="stylesheet" href="mobile.css" media="(max-width: 600px)" /> <!-- 조건이 맞을 때만 차단 -->
```

### 💡 CSS가 렌더링을 차단하는 이유

**⇒ CSS가 준비되기 전에 화면을 그리면 스타일이 적용되지 않은 HTML이 먼저 보였다가 CSS가 로드된 뒤 갑자기 바뀌는 FOUC(Flash of Unstyled Content)가 생기고, 레이아웃을 두 번 계산해야 해서 비효율적이기 때문. 또한 CSS는 뒤의 규칙이 앞의 규칙을 덮어쓸 수 있어 전체를 파싱해야 최종 스타일을 확정할 수 있음. 그래서 브라우저는 CSSOM이 완성될 때까지 렌더링을 미루고, 개발자는 CSS를 `<head>`에 두어 최대한 빨리 받게 하고 크기를 줄여야 함**

### 💡 CSS 선택자는 왜 오른쪽에서 왼쪽으로 매칭하나요?

**⇒ `.nav li a` 같은 선택자를 왼쪽부터 매칭하면 `.nav`의 모든 자손을 탐색하며 조건을 확인해야 해서 비효율적임. 브라우저는 오른쪽 끝의 `a`를 가진 요소만 먼저 고른 뒤, 부모 방향으로 올라가며 `li`와 `.nav`가 있는지 확인해서 맞지 않으면 바로 탈락시키므로 탐색 범위가 훨씬 좁아짐. 그래서 가장 오른쪽 선택자(Key Selector)가 구체적일수록 매칭이 빠르지만, 최신 브라우저에서는 최적화가 잘 되어 있어 실무에서 체감되는 차이는 크지 않음**

---

## 3. Render Tree

- **DOM과 CSSOM을 결합해서, 실제 화면에 보이는 요소와 그 스타일만 담은 트리**
- DOM 트리의 루트부터 **보이는 노드를 순회하며 CSSOM에서 매칭되는 스타일을 찾아 적용**함
- 최신 Chrome에서는 이 단계를 **Style**(스타일 계산), 그 결과로 만든 트리를 **Layout Tree**라고 부르기도 함

### 📍 렌더 트리에 포함되는 것 / 제외되는 것

| 구분 | 예시                                                                                                 |
| ---- | ---------------------------------------------------------------------------------------------------- |
| 제외 | `<head>`, `<script>`, `<meta>`, `<link>` 등 **화면에 표시되지 않는 태그**                            |
| 제외 | CSS로 **`display: none`** 이 적용된 요소와 **그 자손 전체**                                          |
| 포함 | **`visibility: hidden`**, **`opacity: 0`** 인 요소 (보이지는 않지만 **공간을 차지**함)              |
| 포함 | DOM에는 없는 **`::before`, `::after` 가상 요소** (CSS로 생성되어 화면에 보임)                         |

```
     DOM                    CSSOM                         Render Tree
 html                   p { color: red }             html
  ├ head                span { display: none }        └ body
  │  └ title                                             └ p (color: red)
  └ body                                                    └ "Hello"
     └ p
        ├ "Hello"
        └ span
```

### 📍 숨김 처리 방식 비교

| 속성                   | 렌더 트리 | 공간 차지 | 이벤트 반응 | 변경 시 발생하는 작업                |
| ---------------------- | --------- | --------- | ----------- | ------------------------------------ |
| `display: none`        | **제외**  | X         | X           | **Layout + Paint** (레이아웃 재계산) |
| `visibility: hidden`   | 포함      | **O**     | X           | **Paint**                            |
| `opacity: 0`           | 포함      | **O**     | **O** (클릭됨) | **Composite** (레이어 분리 시)     |

### 💡 `display: none`과 `visibility: hidden`의 차이

**⇒ `display: none`은 요소를 렌더 트리에서 아예 제외해서 공간도 차지하지 않으므로, 토글할 때마다 주변 요소들의 위치가 바뀌어 레이아웃을 다시 계산함. `visibility: hidden`은 렌더 트리에 남아 공간은 그대로 차지한 채 보이지만 않게 하므로 레이아웃은 그대로이고 다시 그리기(Paint)만 발생함. 단, 둘 다 스크린 리더가 내용을 읽지 않으므로, 화면에서는 숨기되 스크린 리더에는 읽히게 하려면 `sr-only` 같은 별도 기법을 사용해야 함**

---

## 4. Critical Rendering Path (CRP)

- **브라우저가 HTML, CSS, JS를 받아 첫 화면의 픽셀을 그리기까지 거치는 필수 단계들의 경로**
- **DOM 생성 → CSSOM 생성 → Render Tree → Layout → Paint → Composite**
- CRP를 최적화한다는 것은 **첫 화면이 그려지는 시간(FCP, LCP)을 줄이는 것**

### 📍 CRP 흐름과 JS가 끼어드는 지점

```
 main thread ------------------------------------------------------------------------------------------------->

 [ parse HTML ] [ wait CSSOM ] [ run JS ] [ parse HTML ] [ Style > Layout > Paint > Composite ] --> first frame
        ^              ^            ^
        |              |            + -- JS runs once CSSOM is ready
        |              + -- CSS still loading -> JS must wait
        + -- <script> found -> HTML parsing paused
```

1. HTML을 파싱해 DOM을 만들다가 `<script>`를 만나면 **파싱을 멈춤**
2. 이때 앞에서 요청한 CSS가 아직 로드 중이라면, **CSSOM이 완성될 때까지 JS 실행도 대기**
3. JS 실행이 끝나면 HTML 파싱을 재개해 DOM 완성
4. DOM과 CSSOM이 모두 준비되면 Render Tree → Layout → Paint → Composite로 **첫 화면 출력**

- 즉 **CSS는 JS를 막고, JS는 HTML 파싱을 막는** 의존 관계 때문에 첫 렌더링이 늦어짐

### 📍 CRP를 측정하는 3가지 지표

| 지표                             | 설명                                                    |
| -------------------------------- | ------------------------------------------------------- |
| Critical Resources (주요 리소스 수) | 첫 렌더링을 **차단할 수 있는 리소스의 개수**         |
| Critical Bytes (주요 바이트)     | 첫 렌더링에 필요한 리소스의 **총 크기**                 |
| Critical Path Length (경로 길이) | 주요 리소스를 받기 위해 필요한 **네트워크 왕복(RTT) 수** |

### 📍 CRP 최적화 방법

| 목표                        | 방법                                                                                                      |
| --------------------------- | --------------------------------------------------------------------------------------------------------- |
| 주요 리소스 수 줄이기       | JS에 **`defer` / `async`** 적용, 조건부 CSS에 **`media` 속성** 지정, 첫 화면에 필요 없는 리소스는 지연 로드 |
| 주요 바이트 줄이기          | CSS/JS **압축(minify)**, **gzip/brotli** 전송, 사용하지 않는 CSS 제거, **코드 스플리팅**                   |
| 경로 길이 줄이기            | 첫 화면에 필요한 **Critical CSS를 `<head>`에 인라인**, `preload`/`preconnect`로 리소스를 빨리 요청        |
| 렌더링 시작 앞당기기        | **SSR/SSG**로 완성된 HTML을 내려주기 (12주차 Rendering Architecture)                                       |

```html
<head>
  <style>
    /* 첫 화면(Above the fold)에 필요한 최소한의 Critical CSS 인라인 */
    header { height: 64px; background: #fff; }
  </style>
  <link rel="preload" href="/main.css" as="style" onload="this.rel='stylesheet'" />
  <script src="/app.js" defer></script>
</head>
```

### 💡 브라우저가 URL을 입력받고 화면을 그리기까지의 과정을 설명해주세요

**⇒ 먼저 DNS 조회로 IP를 얻고 TCP·TLS 연결 후 HTTP 요청을 보내 HTML을 받음(4·5주차). 브라우저는 HTML을 받는 대로 파싱해 DOM을 만들고, `<link>`로 CSS를 요청해 CSSOM을 만들며, 동기 `<script>`를 만나면 파싱을 멈추고 JS를 실행함. DOM과 CSSOM이 준비되면 이를 결합해 화면에 보이는 요소만 담은 렌더 트리를 만들고, Layout 단계에서 각 요소의 위치와 크기를 계산한 뒤, Paint 단계에서 레이어별로 픽셀을 채우고, Composite 단계에서 레이어들을 합쳐 GPU를 통해 화면에 출력함**

### 💡 첫 화면 로딩 속도를 개선하려면 어떻게 해야 하나요?

**⇒ CRP 관점에서 첫 렌더링을 막는 리소스의 수, 크기, 왕복 횟수를 줄여야 함. JS는 `defer`나 `async`로 파싱을 막지 않게 하고, 첫 화면에 필요한 Critical CSS는 인라인하고 나머지는 비동기로 로드하며, 리소스를 압축하고 코드 스플리팅으로 초기 번들 크기를 줄임. 또한 `preconnect`, `preload`로 중요한 리소스를 일찍 요청하고, LCP 이미지는 lazy loading하지 않으며, 필요하면 SSR로 완성된 HTML을 내려줘 JS 실행 전에도 화면이 보이도록 함**

---

## 5. Layout

- **렌더 트리의 각 노드가 화면의 어디에, 어떤 크기로 위치할지(기하학 정보)를 계산하는 단계**
- 처음 계산하는 것을 **Layout**, 이후 변경으로 인해 다시 계산하는 것을 **Reflow**라고 함
- `%`, `em`, `vw` 같은 상대 단위를 뷰포트 기준의 **절대 픽셀 값**으로 변환함

### 📍 Layout에서 계산하는 것

- 박스 모델(**content, padding, border, margin**)에 따른 각 요소의 **너비, 높이**
- 각 요소의 **x, y 좌표** (`position`, `float`, `flex`, `grid` 규칙 적용)
- 텍스트의 **줄바꿈 위치**, 인라인 요소의 배치

```html
<div style="width: 50%">
  <div style="width: 50%">Hello</div>
</div>
<!-- 뷰포트 너비가 1000px이면 → 바깥 div 500px, 안쪽 div 250px로 계산 -->
```

- 부모의 크기가 자식에, 자식의 내용이 부모의 높이에 영향을 주므로 **하나의 변경이 트리 전체로 전파**될 수 있음 → 비용이 큼

### 📍 Reflow(Layout)를 발생시키는 작업

| 분류               | 예시                                                                                  |
| ------------------ | ------------------------------------------------------------------------------------- |
| 크기·위치 변경     | `width`, `height`, `padding`, `margin`, `border`, `top/left`, `font-size`             |
| DOM 변경           | 요소 **추가·삭제**, `display: none` 토글, 텍스트 내용 변경                            |
| 환경 변경          | **브라우저 창 크기 변경**, 폰트 로드 완료                                             |
| 레이아웃 값 읽기   | `offsetWidth`, `offsetHeight`, `clientWidth`, `scrollTop`, `getBoundingClientRect()` |

### 📍 강제 동기 레이아웃 (Forced Synchronous Layout)

- 브라우저는 스타일 변경을 모아두었다가 **한 번에 Layout**하지만, 변경 직후 레이아웃 값을 **읽으면** 정확한 값을 주기 위해 **그 자리에서 즉시 Layout을 수행**함
- 이를 반복문 안에서 읽기·쓰기를 번갈아 하면 매번 Layout이 일어나는 **Layout Thrashing** 발생

```js
// Bad: 읽기와 쓰기를 번갈아 → 반복마다 강제 Layout
boxes.forEach((box) => {
  box.style.width = container.offsetWidth / 2 + "px"; // 읽기(offsetWidth) → 쓰기
});

// Good: 읽기를 먼저 모아서 한 번에, 쓰기는 그 다음에
const half = container.offsetWidth / 2;
boxes.forEach((box) => {
  box.style.width = half + "px";
});
```

- Reflow/Repaint 최적화는 **11주차 DOM / Rendering 최적화**에서 자세히 다룸

### 💡 Layout(Reflow)이 비싼 작업인 이유

**⇒ 한 요소의 크기나 위치가 바뀌면 그 요소의 자식, 형제, 부모까지 연쇄적으로 위치와 크기를 다시 계산해야 할 수 있고, Layout이 다시 일어나면 그 뒤의 Paint와 Composite도 모두 다시 수행해야 하기 때문. 게다가 이 작업은 JS와 같은 메인 스레드에서 실행되므로 Reflow가 잦으면 JS 실행과 사용자 입력 처리까지 지연됨. 그래서 애니메이션은 `top/left` 대신 Layout을 일으키지 않는 `transform`으로 처리하고, 레이아웃 값 읽기와 스타일 쓰기를 분리해 강제 동기 레이아웃을 피해야 함**

---

## 6. Paint

- **Layout으로 계산된 위치에 실제로 색, 텍스트, 이미지, 테두리, 그림자 등을 그리는 단계**
- 이후 스타일 변경으로 다시 그리는 것을 **Repaint**라고 함

### 📍 Paint의 두 단계

| 단계                       | 설명                                                                                         |
| -------------------------- | -------------------------------------------------------------------------------------------- |
| Paint (Paint Records 생성) | 메인 스레드가 "배경 → 테두리 → 텍스트" 처럼 **무엇을 어떤 순서로 그릴지 명령 목록**을 기록   |
| Rasterization (래스터화)   | 그리기 명령을 실행해 **실제 픽셀(비트맵)로 변환**. 화면을 **타일**로 나눠 Raster Thread/GPU가 처리 |

- 요소들은 **쌓임 맥락(Stacking Context)** 과 `z-index`에 따라 정해진 순서로 그려짐 → 순서가 틀리면 가려져야 할 요소가 위에 보이는 문제가 생김

```
 Paint Records (draw order)
 1. draw background of <body>
 2. draw border of <div>
 3. draw text "Hello"
 4. draw box-shadow of .card
```

### 📍 Repaint만 발생시키는 속성

- Layout(기하학 정보)에는 영향을 주지 않고 **겉모습만 바꾸는 속성**은 Layout을 건너뛰고 Paint부터 다시 수행

| 구분                     | 속성 예시                                                                      |
| ------------------------ | ------------------------------------------------------------------------------ |
| Repaint만 발생           | `color`, `background-color`, `background-image`, `border-color`, `box-shadow`, `outline`, `visibility` |
| Reflow + Repaint 발생    | `width`, `height`, `margin`, `padding`, `font-size`, `display`, `position`     |

- 같은 Paint라도 `box-shadow`, `filter: blur()`, 큰 그라데이션처럼 **그리는 비용이 큰 속성**은 성능에 영향을 줌

### 💡 Reflow와 Repaint의 차이

**⇒ Reflow는 요소의 크기나 위치 같은 기하학 정보가 바뀌어 Layout을 다시 계산하는 것이고, Repaint는 위치와 크기는 그대로인 채 색상이나 배경처럼 겉모습만 바뀌어 다시 그리는 것임. Reflow가 일어나면 바뀐 위치에 맞게 다시 그려야 하므로 Repaint도 반드시 함께 발생하지만, Repaint는 Reflow 없이 단독으로 발생할 수 있음. 따라서 비용은 Reflow가 더 크며, 가능하면 Layout을 건드리지 않는 속성으로 변경하는 것이 좋음**

---

## 7. Composite

- **페이지를 여러 레이어(Layer)로 나눠 각각 그린 뒤, 올바른 순서로 합쳐서 최종 화면을 만드는 단계**
- 메인 스레드가 아닌 **Compositor Thread와 GPU**가 처리하므로, 메인 스레드가 바쁘더라도 합성만 필요한 변경은 부드럽게 동작함

### 📍 레이어를 나누는 이유

- 페이지 전체를 하나의 이미지로 그리면, 일부만 움직여도 **전체를 다시 그려야 함**
- 움직이는 요소를 **별도 레이어**로 분리하면, 레이어 자체는 다시 그리지 않고 **위치나 투명도만 바꿔서 다시 합성**하면 됨

```
 ┌─────────────────────┐  Layer 3: modal (opacity animation)
 └─────────────────────┘
 ┌─────────────────────┐  Layer 2: sticky header (scroll)
 └─────────────────────┘
 ┌─────────────────────┐  Layer 1: page content
 └─────────────────────┘
            │  Compositor Thread combines layers in order
            ▼
 ┌─────────────────────┐
 │        Screen       │
 └─────────────────────┘
```

### 📍 별도의 레이어(Compositing Layer)가 생기는 경우

- `will-change: transform` / `will-change: opacity`
- 3D transform (`transform: translateZ(0)`, `translate3d`)
- `transform`, `opacity`에 **애니메이션/트랜지션**이 적용된 요소
- `<video>`, `<canvas>`, `<iframe>`
- `position: fixed` 요소 (브라우저에 따라 다름)
- 이미 레이어가 된 요소 **위에 겹치는 요소** (암시적 레이어 생성)

### 📍 속성 변경에 따라 다시 실행되는 단계

| 변경한 속성                                | Layout | Paint | Composite | 비용     |
| ------------------------------------------ | ------ | ----- | --------- | -------- |
| `width`, `height`, `top`, `left`, `margin` | O      | O     | O         | **높음** |
| `color`, `background`, `box-shadow`        | X      | O     | O         | 중간     |
| `transform`, `opacity`                     | X      | X     | O         | **낮음** |

```css
/* Bad: left를 바꾸면 매 프레임 Layout → Paint → Composite */
.box { transition: left 0.3s; }
.box.move { left: 100px; }

/* Good: transform은 Composite만 발생 → Compositor Thread에서 처리 */
.box { transition: transform 0.3s; }
.box.move { transform: translateX(100px); }
```

### 📍 레이어 남용 주의 (Layer Explosion)

- 레이어마다 **GPU 메모리에 별도의 비트맵**을 저장하므로, 레이어가 너무 많으면 **메모리 사용량이 늘고 합성 비용도 커짐** (특히 모바일)
- `will-change`는 **애니메이션 직전에 필요한 요소에만** 적용하고, 끝나면 제거하는 것이 좋음
- 개발자 도구의 **Layers 패널**, **Rendering → Paint flashing / Layer borders**로 레이어와 Repaint 영역을 확인할 수 있음

### 💡 애니메이션에 `transform`과 `opacity`를 권장하는 이유

**⇒ `top`, `left`, `width` 같은 속성을 애니메이션하면 매 프레임마다 메인 스레드에서 Layout과 Paint를 다시 수행해야 해서, 16.7ms(60fps) 안에 처리하지 못하면 프레임이 끊기는 버벅임(Jank)이 생김. 반면 `transform`과 `opacity`는 요소의 기하학 정보나 픽셀을 바꾸지 않고 이미 그려진 레이어를 이동·투명도 조절만 하면 되므로 Layout과 Paint를 건너뛰고 Composite 단계만 실행됨. 이 작업은 메인 스레드가 아닌 Compositor Thread와 GPU가 처리하므로, JS 실행으로 메인 스레드가 바빠도 애니메이션이 부드럽게 유지됨**

### 💡 `will-change`를 모든 요소에 걸면 성능이 좋아지나요?

**⇒ 오히려 나빠질 수 있음. `will-change`는 브라우저에게 해당 요소를 미리 별도 레이어로 분리하라는 힌트인데, 레이어마다 GPU 메모리를 차지하고 레이어 수가 많아질수록 합성해야 할 대상도 늘어나기 때문. 특히 메모리가 적은 모바일 기기에서는 레이어가 과도하게 생기면 오히려 렌더링이 느려지거나 깨질 수 있음. 따라서 실제로 애니메이션되는 요소에만, 성능 문제가 측정되었을 때 제한적으로 사용하는 것이 좋음**
