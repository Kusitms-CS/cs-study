# 10주차 - Browser / Rendering

## 1. DOM (Document Object Model)

### 구조

```text
바이트 → 문자 → 토큰 → 노드 → DOM 트리

<html>                         html
  <body>                        └─ body
    <h1>Title</h1>                  ├─ h1 ── "Title"
    <p>Hello <span>JS</span></p>    └─ p ─── "Hello "
  </body>                                └─ span ── "JS"
</html>
```

### 특징

- 브라우저가 HTML을 파싱해서 만든 트리 구조의 객체 모델. JavaScript는 이 DOM을 통해 문서를 읽고 바꿈
- HTML을 끝까지 받기 전에도 받은 만큼 점진적으로 파싱해서 DOM을 만들어 감
- 파싱 중에 일반 `<script>`를 만나면 HTML 파싱을 멈추고 스크립트를 다운로드·실행한 뒤 이어서 파싱함. 스크립트가 `document.write`나 DOM 조작으로 문서를 바꿀 수 있기 때문(딥다이브와 연결)
- HTML 파싱이 끝나면 `DOMContentLoaded` 이벤트가 발생함

### 정리

- DOM은 HTML 문서를 JavaScript로 다룰 수 있게 만든 트리
- 화면에 보이는 것과 DOM은 다름. DOM에는 `display: none`인 요소나 `<head>` 안의 내용도 모두 들어 있음

---

## 2. CSSOM (CSS Object Model)

### 구조

```css
body { font-size: 16px; }
p { color: gray; }
p span { display: none; }
```

```text
body (font-size: 16px)
 └─ p (font-size: 16px ← 상속, color: gray)
     └─ span (font-size: 16px, color: gray, display: none)
```

### 특징

- CSS를 파싱해서 만든 트리. 각 요소에 최종적으로 어떤 스타일이 적용되는지 계산하는 데 쓰임
- CSS는 상속과 캐스케이드(우선순위) 때문에 일부만 보고 스타일을 확정할 수 없어서, 브라우저는 CSSOM이 준비될 때까지 화면을 그리지 않음(**렌더링 차단 리소스**)
- 스크립트가 스타일 값을 읽을 수 있기 때문에, 앞에 받는 중인 CSS가 있으면 스크립트 실행도 CSS가 끝날 때까지 기다림

### 정리

- CSSOM은 "어떤 요소에 어떤 스타일이 최종 적용되는가"를 담은 트리
- CSS가 늦게 오면 그만큼 첫 화면도 늦게 그려지므로, 첫 화면에 필요한 CSS는 작고 빠르게 받아야 함

---

## 3. Render Tree

### 구조

```text
DOM                       CSSOM
html                      p { color: gray }
 ├─ head                  span { display: none }
 └─ body                  .hidden { visibility: hidden }
     ├─ p
     │   └─ span
     └─ div.hidden
             │
             ▼  DOM + CSSOM 결합, 화면에 그릴 노드만 남김
Render Tree
 body
  ├─ p (color: gray)        ← span은 display: none이라 제외
  └─ div.hidden             ← visibility: hidden은 공간을 차지하므로 포함
```

### 특징

- DOM과 CSSOM을 합쳐 실제로 화면에 그려야 할 노드와 그 스타일만 모은 트리
- `<head>`, `<script>`, `display: none` 요소는 제외됨
- `visibility: hidden`은 보이지 않지만 자리를 차지하므로 포함되고, `::before` 같은 의사 요소는 DOM에 없지만 포함됨

### 정리

- Render Tree는 "화면에 무엇을 어떤 스타일로 그릴지"의 목록
- `display: none`은 트리에서 빠져 자리도 없어지고, `visibility: hidden`은 자리는 남는다는 차이로 기억

---

## 4. Critical Rendering Path

### 구조

```text
HTML ──> DOM ──┐
               ├──> Render Tree ──> Layout ──> Paint ──> Composite ──> 화면
CSS  ──> CSSOM ┘
          ▲
JavaScript (DOM·CSSOM을 읽고 바꿀 수 있어 파싱을 막을 수 있음)
```

### 특징

- 브라우저가 HTML, CSS, JavaScript를 받아 첫 화면을 그리기까지 거치는 단계
- 이 경로가 짧을수록 첫 화면이 빨리 뜸. 관련 지표는 FCP(첫 콘텐츠 표시), LCP(가장 큰 콘텐츠 표시)(13주차 Web Vitals와 연결)
- 최적화 방향은 세 가지
  - 렌더링을 막는 리소스 수 줄이기 (`async`/`defer`, 첫 화면에 필요 없는 CSS 분리)
  - 리소스 크기 줄이기 (압축, 사용하지 않는 코드 제거)
  - 왕복 횟수 줄이기 (중요한 리소스를 `preload`로 미리 요청, 5주차 HTTP/2와 연결)

### 정리

- CRP는 "HTML을 받고 첫 픽셀을 그리기까지의 경로"
- CSS는 렌더링을, 일반 스크립트는 파싱을 막기 때문에 둘을 어떻게 불러오는지가 첫 화면 속도를 좌우함

---

## 5. Layout

### 구조

```text
Render Tree                       Layout 결과
body                              body  : x=0,  y=0,  w=1200, h=800
 ├─ div (width: 50%)       ──>     div   : x=0,  y=0,  w=600,  h=200
 └─ p                              p     : x=0,  y=200, w=1200, h=40
```

### 특징

- Render Tree의 각 노드가 화면의 어디에, 어떤 크기로 놓일지 계산하는 단계. Reflow라고도 부름
- `%`, `vw`, `em` 같은 상대 단위를 실제 픽셀 값으로 바꿈. 뷰포트 크기가 바뀌면 다시 계산해야 함
- 한 요소의 크기가 바뀌면 형제나 부모, 자식의 위치도 바뀔 수 있어 계산 범위가 넓어지기 쉬움
- `width`, `height`, `margin`, `top`, 글자 크기를 바꾸거나 `offsetWidth`, `getBoundingClientRect()`로 크기를 읽으면 Layout이 다시 일어날 수 있음(11주차 Reflow와 연결)

### 정리

- Layout은 "각 요소의 위치와 크기를 픽셀 단위로 정하는 계산"
- 렌더링 단계 중 비용이 큰 편이라, 애니메이션에서 Layout을 다시 일으키는 속성은 피하는 게 좋음

---

## 6. Paint

### 구조

```text
Layout 결과 (위치·크기)
        │
        ▼
그리기 명령 목록 만들기
  1. 배경색 칠하기
  2. 테두리 그리기
  3. 텍스트 그리기
  4. 그림자 그리기
        │
        ▼
(다음 단계) 레이어별로 실제 픽셀로 변환 (Rasterize)
```

### 특징

- 위치와 크기가 정해진 요소들을 어떤 순서로 어떻게 그릴지(배경, 테두리, 텍스트, 그림자 등) 결정하는 단계. 이 단계에서 만드는 것은 픽셀이 아니라 그리기 명령 목록이고, 실제 픽셀로 바꾸는 작업은 Raster 단계에서 따로 함(9번 딥다이브와 연결)
- 화면을 여러 레이어로 나눠 레이어마다 따로 그림. 바뀐 레이어만 다시 그리면 됨
- `color`, `background`, `box-shadow`처럼 위치는 그대로이고 모양만 바뀌면 Layout 없이 Paint부터 다시 실행됨(Repaint)
- 넓은 영역의 `box-shadow`, `filter: blur()`처럼 그리기 비용이 큰 스타일도 있음

### 정리

- Paint는 "정해진 자리에 무엇을 어떻게 칠할지 적은 작업 지시서를 만드는 작업"
- 레이아웃은 그대로 두고 색만 바꾸면 Layout은 건너뛰지만 Paint는 다시 일어남

---

## 7. Composite

### 구조

```text
레이어 1: 배경
레이어 2: 본문
레이어 3: 고정 헤더 (position: fixed)
레이어 4: 애니메이션 중인 모달 (transform)
        │
        ▼  순서와 위치에 맞게 겹쳐서 최종 화면 생성 (Compositor 스레드, GPU)
     화면
```

### 특징

- 따로 그려진 레이어들을 올바른 순서로 합쳐 최종 화면을 만드는 단계
- 메인 스레드가 아니라 Compositor 스레드에서 처리할 수 있어서, 메인 스레드가 JavaScript로 바빠도 스크롤이나 일부 애니메이션은 부드럽게 유지될 수 있음(9주차 Long Task와 연결)
- `transform`과 `opacity`는 별도 레이어에서 Layout·Paint 없이 Composite 단계만으로 바꿀 수 있어 애니메이션에 가장 저렴함
- `will-change: transform`으로 레이어를 미리 만들 수 있지만, 레이어마다 메모리를 쓰므로 많이 만들면 오히려 느려짐

### 정리

- Composite는 "여러 장의 투명 필름을 순서대로 겹쳐 한 장의 화면으로 만드는 작업"
- 애니메이션은 `top`/`left` 대신 `transform`, 깜빡임은 `visibility` 대신 `opacity`처럼 Composite만 일으키는 속성을 쓰는 것이 좋음

### 속성별 렌더링 비용

| 바꾼 속성 | 다시 실행되는 단계 | 비용 |
| --- | --- | --- |
| `width`, `height`, `margin`, `top`, `font-size` | Layout → Paint → Composite | 큼 |
| `color`, `background-color`, `box-shadow` | Paint → Composite | 중간 |
| `transform`, `opacity` | Composite | 작음 |

---

## 8. 딥다이브: 첫 화면을 막는 리소스 다루기 (script async / defer, CSS)

### 스크립트가 파싱을 막는 이유

일반 `<script>`는 실행 중에 DOM을 바꿀 수 있어서, 브라우저는 HTML 파싱을 멈추고 스크립트를 다운로드·실행한 뒤에야 파싱을 이어간다. 스크립트가 `<head>`에 있고 크기가 크면 그동안 화면에는 아무것도 그려지지 않는다.

```text
<script src="app.js">
HTML 파싱  ■■■■■□□□□□□□□□□□■■■■■■■
다운로드        ▓▓▓▓▓▓▓
실행                   ████
                ↑ 파싱 멈춤 구간

<script async src="analytics.js">
HTML 파싱  ■■■■■■■■■□□□□■■■■■■■■
다운로드        ▓▓▓▓
실행               ████        ← 다운로드가 끝나는 즉시 실행 (파싱을 잠깐 멈춤)

<script defer src="app.js">
HTML 파싱  ■■■■■■■■■■■■■■■■■■■■■
다운로드        ▓▓▓▓
실행                            ████  ← 파싱이 끝난 뒤, DOMContentLoaded 전에 실행
```

| 방식 | 다운로드 | 실행 시점 | 실행 순서 | 쓰는 곳 |
| --- | --- | --- | --- | --- |
| 일반 `<script>` | 파싱을 멈추고 다운로드 | 다운로드 직후 | 문서 순서 | 거의 쓰지 않음 |
| `async` | 파싱과 동시에 | 다운로드가 끝나는 즉시 | 보장되지 않음 | 다른 코드와 상관없는 분석·광고 스크립트 |
| `defer` | 파싱과 동시에 | HTML 파싱이 끝난 뒤 | 문서 순서 보장 | DOM을 다루는 앱 코드 |
| `type="module"` | 파싱과 동시에 | `defer`와 같음 (기본) | 문서 순서 보장 | ES Module 번들 |

- `defer` 스크립트는 DOM이 다 만들어진 뒤 실행되므로 `DOMContentLoaded`를 기다리는 코드 없이 바로 DOM에 접근할 수 있다.
- `async` 스크립트끼리는 먼저 다운로드된 것이 먼저 실행되므로, 서로 의존하는 스크립트에는 쓰면 안 된다.
- Vite 같은 번들러가 만드는 앱은 보통 `type="module"`로 불러오기 때문에 기본적으로 `defer`처럼 동작한다.

### CSS는 렌더링을 막는다

CSS는 HTML 파싱은 막지 않지만, CSSOM이 완성될 때까지 화면을 그리지 않는다. CSS 없이 먼저 그리면 스타일이 없는 화면이 잠깐 보였다가 바뀌는 FOUC(Flash of Unstyled Content)가 생기기 때문이다.

```html
<!-- 첫 화면에 필요한 핵심 CSS는 바로 불러옴 -->
<link rel="stylesheet" href="main.css" />

<!-- 조건에 맞지 않는 미디어용 CSS는 렌더링을 막지 않음 -->
<link rel="stylesheet" href="print.css" media="print" />

<!-- 나중에 필요한 중요 리소스는 미리 다운로드만 시작 -->
<link rel="preload" href="/fonts/Pretendard.woff2" as="font" type="font/woff2" crossorigin />
```

- `media` 조건이 지금 화면에 맞지 않는 CSS는 다운로드는 하지만 렌더링을 막지 않는다.
- 첫 화면에 필요한 CSS만 HTML에 인라인으로 넣고 나머지는 나중에 불러오는 방법(Critical CSS)도 있다.
- `preload`는 브라우저가 늦게 발견하는 리소스(CSS 안에서 참조하는 폰트 등)를 미리 요청해서 왕복 시간을 줄인다. 폰트는 `crossorigin`을 빼면 두 번 다운로드될 수 있다.

### 정리

- 첫 화면 속도는 "HTML 파싱을 막는 스크립트"와 "렌더링을 막는 CSS"를 얼마나 줄이느냐에 달려 있다.
- 앱 코드는 `defer`(또는 `type="module"`), 독립적인 외부 스크립트는 `async`, CSS는 첫 화면에 필요한 것만 먼저 불러오는 것이 기본 전략이다.

---

## 9. 딥다이브: Rasterize와 Compositor 스레드

### Paint와 Raster는 다른 단계다

Paint 단계에서 만드는 것은 "(0, 0)부터 600×200 크기로 흰 배경 칠하기, (16, 24)에 'Title' 글자 쓰기" 같은 **그리기 명령 목록**이다. 이 명령을 실제 화면에 표시할 수 있는 픽셀(비트맵)로 바꾸는 작업을 **Rasterize(래스터화)**라고 한다. Chrome 기준으로 보면 렌더링 작업은 두 스레드에 나뉘어 있다.

```text
메인 스레드                         Compositor 스레드              GPU
┌──────────────────────────┐       ┌──────────────────────┐      ┌──────────┐
│ JavaScript               │       │ 레이어를 타일로 나눔    │      │ 타일을    │
│ Style → Layout → Paint   │ ───>  │ 화면 근처 타일부터     │ ───> │ 픽셀로    │
│ (그리기 명령 목록 생성)    │ Commit│ Raster 요청           │      │ 변환      │
└──────────────────────────┘       │ 타일을 합쳐 프레임 생성 │ <─── │ (Raster) │
                                   └──────────┬───────────┘      └──────────┘
                                              ▼
                                            화면
```

- 메인 스레드는 Paint까지 끝내고 결과(레이어와 그리기 명령)를 Compositor 스레드에 넘긴다(Commit).
- Rasterize는 메인 스레드 밖에서 처리된다. 요즘 브라우저는 대부분 GPU로 래스터화한다(GPU Rasterization).
- 그래서 메인 스레드가 JavaScript로 바빠도, 이미 래스터화된 결과를 다시 합치기만 하면 되는 작업(스크롤, `transform` 애니메이션)은 계속 부드럽게 움직일 수 있다(9주차 Long Task와 연결).

### 타일 단위로 래스터화한다

긴 페이지 전체를 한 번에 픽셀로 바꾸면 시간과 메모리가 많이 든다. 그래서 Compositor 스레드는 레이어를 작은 **타일**로 나누고, 지금 화면(뷰포트)에 보이거나 곧 보일 타일부터 래스터화한다.

```text
페이지 레이어
┌────┬────┬────┐
│ ✓  │ ✓  │ ✓  │  ← 화면에 보이는 영역: 먼저 래스터화
├────┼────┼────┤
│ ✓  │ ✓  │ ✓  │
├────┼────┼────┤
│ …  │ …  │ …  │  ← 곧 보일 영역: 그다음
├────┼────┼────┤
│    │    │    │  ← 먼 영역: 아직 안 함
└────┴────┴────┘
```

- 아주 빠르게 스크롤하면 래스터화가 따라오지 못해 빈 칸이나 흰 영역이 잠깐 보일 수 있다(Checkerboarding).
- 한 번 래스터화한 타일은 재사용되므로, 스크롤처럼 위치만 바뀌는 경우에는 다시 그리지 않고 합치기만 한다.

### 프론트엔드에서 신경 쓸 점

| 상황 | 일어나는 일 | 대응 |
| --- | --- | --- |
| `transform: translate`, `opacity` 애니메이션 | 이미 래스터화된 레이어를 그대로 옮기거나 투명도만 바꿈 | 애니메이션은 이 두 속성으로 구현 |
| `transform: scale`로 확대 | 작은 크기로 래스터화된 결과를 늘려서 보여주다 보니 글자가 잠깐 흐릿해질 수 있음 | 애니메이션이 끝난 뒤 다시 래스터화됨. 계속 흐리면 `will-change`를 애니메이션 중에만 적용 |
| `will-change`를 남발 | 레이어마다 래스터화된 비트맵을 GPU 메모리에 들고 있음 | 실제로 움직이는 요소에만, 필요한 동안만 적용 |
| 큰 이미지 | 픽셀로 쓰기 전에 디코딩이 필요하고, 표시 크기보다 크면 메모리 낭비 | 표시 크기에 맞는 이미지, `srcset` 사용, `decoding="async"`로 디코딩 지연 줄이기 |
| `filter: blur()`, 큰 `box-shadow` | 래스터화할 때 계산이 많아 프레임이 밀릴 수 있음 | 애니메이션 대상에는 피하거나 범위를 줄이기 |

### 정리

- 렌더링 마지막 단계는 "Paint로 그리기 명령 만들기 → Raster로 픽셀 만들기 → Composite로 합치기"로 나눠서 이해한다.
- Raster와 Composite는 메인 스레드 밖에서 처리되기 때문에, 이 단계만 일으키는 `transform`·`opacity` 애니메이션은 JavaScript가 바빠도 부드럽다.
- 레이어와 래스터화된 타일은 모두 메모리를 쓰므로, 레이어를 무조건 많이 만드는 것이 최적화는 아니다.
