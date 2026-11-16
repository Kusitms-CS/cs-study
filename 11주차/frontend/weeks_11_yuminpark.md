# [Week 11] 11주차 DOM / Rendering 최적화 정리

## 목차

1. [Reflow](#1-reflow)
2. [Repaint](#2-repaint)
3. [Virtual DOM](#3-virtual-dom)
4. [Reconciliation](#4-reconciliation)
5. [React Rendering](#5-react-rendering)
6. [정리](#6-정리)

> 10주차에서 Layout/Paint/Composite의 **원리를** 다뤘다면, 이번 주는 그 비용을 **줄이는 방법과**, React가 DOM 변경을 줄이기 위해 쓰는 **Virtual DOM / Reconciliation / Rendering을** 다룸

---

## 1. Reflow

### 개념

- **Reflow**: DOM/스타일 변경으로 요소의 **위치와 크기를 다시 계산하는** 과정 (Layout, 10주차)
- 렌더링 파이프라인에서 **가장 비싼 단계이고**, 일어나면 Paint, Composite까지 연쇄적으로 따라옴

```
Layout 속성 변경 :  Style ─► Layout ─► Paint ─► Composite    (가장 비쌈)
```

### 왜 비싼가

- 한 요소의 크기가 바뀌면 **자손, 형제, 조상의** 위치와 크기까지 영향을 받을 수 있음
- 계산량이 **영향받는 DOM 노드 수에** 비례하고, 메인 스레드에서 실행되므로 그동안 JS와 이벤트 처리가 멈춤 (9주차) → 프레임이 건너뛰어져 끊김(Jank)

```
parent  ◄── 높이가 달라짐 (자식 합으로 결정)
 ├─ target (width 변경)
 │   └─ child   ◄── 크기 재계산
 └─ sibling     ◄── 위치가 밀림
```

### 브라우저가 이미 하는 최적화

- 스타일 변경을 **즉시 처리하지 않고 모아뒀다가**, 프레임이 시작될 때 한 번에 레이아웃함
- 따라서 **쓰기만 연속으로** 하면 레이아웃은 한 번만 일어남
- 이 최적화가 깨지는 순간이 **쓰기 사이에 레이아웃 값을 읽을 때** (Forced Synchronous Layout, 10주차)

### 최적화 1: 읽기와 쓰기를 분리

```js
// 나쁨: 읽기 → 쓰기 → 읽기 → 쓰기 ... (반복마다 레이아웃 재계산)
items.forEach((el) => {
  const h = el.offsetHeight;          // 읽기 (레이아웃 강제)
  el.style.height = h + 10 + 'px';    // 쓰기 (레이아웃 무효화)
});

// 좋음: 읽기 일괄 → 쓰기 일괄
const heights = items.map((el) => el.offsetHeight);   // 읽기만 모아서 1번
requestAnimationFrame(() => {
  items.forEach((el, i) => {
    el.style.height = heights[i] + 10 + 'px';         // 쓰기만 모아서 프레임에서 1번
  });
});
```

- 읽기/쓰기 스케줄링을 대신해주는 **FastDOM** 같은 라이브러리도 있음

### 최적화 2: 스타일은 클래스로 한 번에

```js
// 여러 속성을 하나씩 변경
el.style.width = '100px';
el.style.height = '50px';
el.style.margin = '10px';

// 클래스 하나로 교체: 변경이 한 곳에 모이고 CSS와 JS의 역할이 분리됨
el.classList.add('expanded');
```

- 성능만 보면 연속 쓰기는 브라우저가 이미 모아주지만, 클래스 방식은 **중간에 읽기가 끼어드는 실수를 줄이고** 유지보수도 쉬움

### 최적화 3: 레이아웃 속성으로 애니메이션하지 않기

```css
/* 나쁨: 매 프레임 Layout */
.box { transition: left 0.3s, width 0.3s; }

/* 좋음: 합성만 (2장) */
.box { transition: transform 0.3s; }
```

### 최적화 4: 레이아웃 영향 범위 제한

| 방법 | 효과 |
|---|---|
| `position: absolute / fixed` | 일반 흐름에서 빠져서 주변 요소 재배치에 영향이 없음 |
| 고정 크기 컨테이너 | 내부가 변해도 바깥 레이아웃은 그대로 |
| `contain: layout` / `content-visibility: auto` | 내부 변경이 바깥에 영향 주지 않도록 격리, 화면 밖은 건너뜀 (10주차) |
| 깊은 중첩 줄이기 | 노드 수와 연쇄 범위 감소 |
| 이미지 크기 미리 지정 | 로드 후 레이아웃 이동(CLS) 방지 |

### 최적화 5: 스크롤/리사이즈 이벤트에서 레이아웃 읽지 않기

- `scroll`, `resize` 핸들러는 **매우 자주 호출되는데**, 그 안에서 `getBoundingClientRect()`를 읽으면 강제 레이아웃이 반복됨
- 브라우저가 제공하는 **Observer를** 사용하면 메인 스레드 부담 없이 변화를 감지할 수 있음

```js
// 나쁨: 스크롤할 때마다 모든 요소의 위치를 읽음
window.addEventListener('scroll', () => {
  images.forEach((img) => {
    if (img.getBoundingClientRect().top < window.innerHeight) load(img);
  });
});

// 좋음: IntersectionObserver가 화면 진입을 알려줌
const io = new IntersectionObserver((entries) => {
  for (const entry of entries) {
    if (entry.isIntersecting) {
      load(entry.target);
      io.unobserve(entry.target);
    }
  }
});
images.forEach((img) => io.observe(img));
```

| Observer | 용도 |
|---|---|
| `IntersectionObserver` | 요소가 화면에 보이는지 (지연 로딩, 무한 스크롤, 노출 측정) |
| `ResizeObserver` | 요소 크기 변화 감지 (`resize` 이벤트 + 반복 측정 대체) |
| `MutationObserver` | DOM 변경 감지 |

- 이벤트 자체를 줄이려면 **debounce**(입력이 멈춘 뒤 한 번), **throttle**(일정 간격으로 한 번)을 사용

### 최적화 6: DOM 노드 수 줄이기 (리스트 가상화)

- 수천 개의 항목을 모두 DOM으로 만들면 **레이아웃, 페인트, 메모리 비용이 모두 커짐**
- **가상화(Windowing)**: 데이터는 전부 들고 있되, **화면에 보이는 범위(+ 여유분)만 DOM으로 렌더링**

```
전체 데이터 10,000개                 실제 DOM 노드
┌──────────────┐
│ item 1       │
│ ...          │
│ item 41      │ ◄─ viewport 시작      item 41
│ item 42      │                       item 42
│ ...          │                        ...        (보이는 약 20개 + 여유분)
│ item 60      │ ◄─ viewport 끝        item 60
│ ...          │
│ item 10000   │
└──────────────┘
```

```js
// 가상화의 핵심 계산 (고정 높이 항목 기준)
const start = Math.floor(scrollTop / ITEM_HEIGHT);
const visibleCount = Math.ceil(viewportHeight / ITEM_HEIGHT) + 5;   // 위아래 여유분
const visible = items.slice(start, start + visibleCount);

// 전체 스크롤 길이는 유지: 위쪽 여백 + 전체 높이로 스크롤바 크기를 맞춤
const offsetY = start * ITEM_HEIGHT;
```

- 실무에서는 `react-window`, `@tanstack/react-virtual` 같은 라이브러리를 사용 (가변 높이, 스크롤 복원 등 처리할 것이 많음)

---

## 2. Repaint

### 개념

- **Repaint**: 레이아웃은 그대로인 채로 **시각적 속성(색, 그림자 등)이 바뀌어 다시 그리는** 과정 (Paint, 10주차)
- Reflow가 일어나면 Repaint는 **항상** 뒤따르지만, Repaint는 Reflow 없이도 일어날 수 있음

| | Reflow | Repaint |
|---|---|---|
| 원인 | 크기, 위치, DOM 구조 변경 | 색, 배경, 그림자 등 모양 변경 |
| 다시 실행되는 단계 | Layout → Paint → Composite | Paint → Composite |
| 비용 | 큼 | 중간 |
| 예시 | `width`, `margin`, `display` | `color`, `background`, `box-shadow` |

```
Paint 속성 변경    :  Style ──────────► Paint ─► Composite    (중간)
Composite 속성 변경:  Style ────────────────────► Composite   (가장 가벼움)
```

- 최적화의 방향: **Reflow 회피 > Repaint 회피 > Composite만 일어나게**

### 최적화 1: 애니메이션은 transform/opacity로

| 하고 싶은 동작 | 피할 것 (Layout/Paint 발생) | 대신 사용 (Composite만) |
|---|---|---|
| 이동 | `top`, `left`, `margin` | `transform: translate()` |
| 크기 변경 | `width`, `height` | `transform: scale()` |
| 회전 | - | `transform: rotate()` |
| 나타남/사라짐 | `display`, `visibility` 전환 | `opacity` |

- `transform`과 `opacity`는 (레이어로 분리된 경우) **Layout과 Paint를 건너뛰고 Compositor 스레드에서 처리되어** JS가 바빠도 부드러움 (10주차)

```js
// JS 애니메이션도 transform으로, 타이밍은 requestAnimationFrame으로
let x = 0;
function frame() {
  x += 2;
  box.style.transform = `translateX(${x}px)`;
  if (x < 300) requestAnimationFrame(frame);
}
requestAnimationFrame(frame);
```

### 최적화 2: 비싼 효과를 애니메이션하지 않기

- `box-shadow`, `filter: blur()`, 큰 그라디언트는 **픽셀 연산이 많아서** 변하는 동안 계속 다시 그리면 비쌈

```css
/* 나쁨: hover마다 box-shadow 값을 애니메이션 → 매 프레임 Paint */
.card { transition: box-shadow 0.2s; }
.card:hover { box-shadow: 0 8px 24px rgba(0, 0, 0, 0.2); }

/* 좋음: 그림자는 미리 그려두고 opacity만 전환 → Composite */
.card { position: relative; }
.card::after {
  content: '';
  position: absolute;
  inset: 0;
  box-shadow: 0 8px 24px rgba(0, 0, 0, 0.2);
  opacity: 0;
  transition: opacity 0.2s;
  pointer-events: none;
}
.card:hover::after { opacity: 1; }
```

### 최적화 3: 레이어 분리와 will-change

- 자주 움직이는 요소를 **별도 레이어로 승격하면** 주변을 다시 그리지 않고 **합성만** 다시 함

```css
.modal { will-change: transform; }   /* 곧 움직일 것이라는 힌트: 미리 레이어로 준비 */
```

```js
// 필요한 시점에만 켜고, 끝나면 해제 (레이어는 GPU 메모리를 차지함)
el.addEventListener('mouseenter', () => { el.style.willChange = 'transform'; });
el.addEventListener('transitionend', () => { el.style.willChange = 'auto'; });
```

| 주의 | 이유 |
|---|---|
| `will-change` 남용 금지 | 레이어마다 GPU 메모리를 사용하고, 레이어가 많으면 합성 비용도 늘어남 (Layer Explosion) |
| 상시 적용하지 않기 | 애니메이션 직전에 켜고 끝나면 제거 |
| 효과를 먼저 측정 | 레이어 분리가 오히려 느려질 수도 있음 |

---

## 3. Virtual DOM

### 개념

- **Virtual DOM**: 실제 DOM을 **가벼운 JS 객체 트리로 표현한** 것
- UI 상태가 바뀌면 **새 Virtual DOM을 만들고 이전 것과 비교(Diff)해서, 달라진 부분만 실제 DOM에 반영하는** 방식
- 브라우저의 기능이 아니라 **React 같은 라이브러리가 만들어 쓰는 자료구조**

### React Element의 모양

```jsx
<div className="card">
  <h1>Title</h1>
</div>
```

```js
// JSX가 변환된 결과 (단순화): 일반 JS 객체
{
  type: 'div',
  props: {
    className: 'card',
    children: {
      type: 'h1',
      props: { children: 'Title' },
    },
  },
}
```

- 실제 DOM 노드는 수백 개의 프로퍼티와 메서드를 가진 무거운 객체이지만, React Element는 **`type`, `props` 정도만 가진 가벼운 객체라서** 만들고 비교하는 비용이 작음

### 동작 흐름

```
① 상태 변경 (setState)
        ▼
② 새 Virtual DOM 생성 (컴포넌트 함수를 다시 호출)
        ▼
③ 이전 Virtual DOM과 비교 (Diff = Reconciliation)
        ▼
④ 달라진 부분만 실제 DOM에 반영 (Commit)
        ▼
⑤ 브라우저: Layout → Paint → Composite
```

```
이전 Virtual DOM            새 Virtual DOM              실제 DOM에 반영
div                         div
├─ h1 "Title"               ├─ h1 "Title"              (변화 없음)
└─ p  "Count: 1"            └─ p  "Count: 2"           → p의 텍스트만 변경
```

- 전체를 다시 그리는 것이 아니라 **바뀐 텍스트 하나만** DOM에 반영되므로, 이후의 Reflow/Repaint 범위도 작아짐

### 왜 사용하는가

| 이유 | 설명 |
|---|---|
| 선언형 UI | "상태가 이럴 때 UI는 이렇다"만 작성하고, **어떻게 DOM을 바꿀지는 라이브러리가 처리** |
| 실수 방지 | 모든 상태 변화마다 DOM을 직접 동기화하다 생기는 누락/불일치를 줄임 |
| 변경 모아서 반영 | 여러 상태 변경을 모아 한 번의 DOM 업데이트로 처리 |
| 플랫폼 분리 | 같은 방식으로 DOM 외에 Native(React Native), 서버(SSR)로도 렌더링 |

### 흔한 오해

| 오해 | 실제로는 |
|---|---|
| Virtual DOM이 실제 DOM보다 빠르다 | Diff라는 **추가 작업이** 생기므로, 정확히 필요한 곳만 직접 조작하면 이론상 더 빠름. 가치는 **충분히 빠르면서 코드가 예측 가능하다는** 점 |
| Virtual DOM 덕분에 Reflow가 안 일어난다 | 실제 DOM이 바뀌면 Reflow/Repaint는 그대로 발생함. **변경을 최소화할** 뿐 |
| Virtual DOM은 브라우저에 내장되어 있다 | 라이브러리가 만든 JS 객체이고, **Shadow DOM(웹 표준)과는 다른 개념** |
| 직접 DOM을 만져도 문제없다 | `ref`로 React가 관리하는 DOM을 직접 수정하면 **React가 모르는 변경이** 되어 불일치가 생길 수 있음 |

### 한계와 대안

- 상태가 바뀔 때마다 **트리를 다시 만들고 비교하는 비용이** 있어서, 큰 트리에서는 불필요한 비교를 줄이는 최적화가 필요함 (5장)
- 그래서 VDOM 없이 변경 지점을 직접 추적하는 접근도 있음

| 방식 | 변경 반영 방법 | 대표 |
|---|---|---|
| Virtual DOM | 트리를 새로 만들어 Diff | React, Vue (컴파일러가 Diff를 최적화) |
| 컴파일 타임 | 빌드 시 "상태 변경 → DOM 업데이트" 코드를 생성 | Svelte |
| Fine-grained Reactivity | 상태를 구독한 DOM 지점만 직접 갱신 | Solid, Signals 계열 |

---

## 4. Reconciliation

### 개념

- **Reconciliation(재조정)**: 이전 트리와 새 트리를 **비교해서 실제 DOM에 무엇을 어떻게 반영할지 결정하는 알고리즘** (React의 Diffing)
- 일반적인 두 트리 비교 알고리즘은 **O(n³)이라** UI에는 너무 느림
- React는 두 가지 **휴리스틱(가정)으로** **O(n)에** 처리함

| 가정 | 내용 |
|---|---|
| 1 | **타입이 다른 요소는 완전히 다른 트리를** 만든다 |
| 2 | 리스트는 **`key`로 같은 항목인지 식별할** 수 있다 |

- 비교도 **같은 레벨끼리만** 수행해서 (트리를 가로질러 이동한 요소는 추적하지 않음) 계산량을 줄임

### 규칙 1: 타입이 다르면 통째로 교체

```jsx
// <div> → <span> 로 바뀌면: 하위 전체를 unmount(state 소실) 후 새로 mount
<div><Counter /></div>
<span><Counter /></span>
```

| 상황 | 동작 |
|---|---|
| 요소 타입이 다름 (`div` → `span`) | 이전 서브트리를 **제거(unmount)하고** 새로 **생성(mount)**. 내부 state도 사라짐 |
| 컴포넌트 타입이 다름 (`<A />` → `<B />`) | 마찬가지로 이전 컴포넌트와 자식을 모두 제거하고 새로 생성 |

### 규칙 2: 타입이 같으면 업데이트

| 대상 | 동작 |
|---|---|
| DOM 요소 | **같은 DOM 노드를 재사용하고**, 바뀐 속성만 갱신 (`className`, `style` 일부 등) |
| 컴포넌트 | **같은 인스턴스를 유지(state 보존)하고**, 새 props로 다시 렌더 → 자식을 재귀적으로 비교 |

```jsx
<div className="a" title="x" />
<div className="b" title="x" />
// 같은 div 노드를 재사용하고, className만 'a' → 'b'로 변경
```

### 규칙 3: 리스트는 key로 식별

- `key`가 없으면 React는 **순서(위치)** 기준으로 비교함

```jsx
// 이전                      // 이후: 맨 앞에 A 삽입
<li>B</li>                   <li>A</li>
<li>C</li>                   <li>B</li>
                             <li>C</li>

// key 없음: 위치 기준 비교 → B→A 변경, C→B 변경, C 새로 추가  (3번 작업)
// key 있음: key로 B, C가 그대로임을 알고 → A만 추가              (1번 작업)
```

```
key 없음 (위치로 비교)            key 있음 (key로 식별)
  [B]  ──►  [A]  (변경)             [  ]  ──►  [A]  (추가)
  [C]  ──►  [B]  (변경)             [B]   ═══  [B]  (재사용)
  [  ] ──►  [C]  (추가)             [C]   ═══  [C]  (재사용)
```

### 좋은 key의 조건

| 조건 | 설명 |
|---|---|
| 형제 사이에서 유일 | 같은 리스트 안에서 중복되면 안 됨 |
| 안정적 | 렌더링마다 바뀌지 않아야 함 |
| 데이터의 고유 ID | DB의 `id` 같은 값이 이상적 |

| 잘못된 key | 문제 |
|---|---|
| `Math.random()`, `Date.now()` | 렌더마다 key가 바뀌어 **모든 항목이 매번 unmount/mount** |
| 배열 `index` | 항목이 **추가/삭제/재정렬되면** key와 항목의 대응이 어긋남 |

### index를 key로 쓰면 생기는 버그

```jsx
{todos.map((todo, index) => (
  <li key={index}>
    <input defaultValue={todo.text} />   {/* 입력 중인 값은 DOM(내부 state)에 있음 */}
  </li>
))}
```

- 첫 번째 항목을 삭제하면, 남은 항목들의 `index` key가 한 칸씩 당겨짐
- React는 "key 0, 1은 그대로 있고 마지막 key가 사라졌다"고 이해해서 **DOM 노드와 입력 중이던 값이 엉뚱한 항목에 남음**
- 해결: `key={todo.id}`

### 컴포넌트를 렌더 안에서 정의하지 않기

```jsx
function Parent() {
  const Child = () => <input />;    // 렌더링할 때마다 새 함수 = 새로운 "타입"
  return <Child />;                 // 규칙 1에 따라 매번 unmount/mount → 입력값, 포커스 소실
}

const Child = () => <input />;      // 바깥에서 한 번만 정의
function Parent() {
  return <Child />;
}
```

### Fiber 아키텍처

- 기존(Stack Reconciler)은 트리를 **재귀로 한 번에** 처리해서, 큰 트리에서 메인 스레드를 오래 점유하고 **중간에 멈출 수 없었음** (9주차의 긴 작업 문제)
- **Fiber**(React 16~): 작업을 **컴포넌트 단위의 작은 조각으로** 나누고, 중간에 **멈추고 이어서 하거나 우선순위를 바꿀 수 있게** 재설계한 구조

```
App
 │ child
 ▼
Header ── sibling ──► Main ── sibling ──► Footer
                       │ child
                       ▼
                      Post
(각 Fiber는 return으로 부모를 가리킴)
```

- 각 컴포넌트가 하나의 **Fiber 노드이고**, `child`/`sibling`/`return` 포인터로 연결된 **연결 리스트 형태라서** 재귀 없이 순회하고 중간에 멈췄다가 이어갈 수 있음 (3주차 LinkedList)
- **두 개의 트리를** 유지: 현재 화면에 반영된 **current 트리와**, 작업 중인 **workInProgress 트리**. 작업이 끝나면 둘을 교체함 (더블 버퍼링)

### Render Phase와 Commit Phase

| | Render Phase | Commit Phase |
|---|---|---|
| 하는 일 | 컴포넌트 함수 호출, 새 트리 생성, **Diff로 변경 사항 계산** | 계산된 변경 사항을 **실제 DOM에 반영** |
| 중단 가능 여부 | **가능** (동시성 기능 사용 시 중단/재개/폐기) | **불가능** (동기적으로 한 번에 끝냄) |
| 부수 효과 | **없어야 함** (순수해야 함) | DOM 변경, ref 연결, `useLayoutEffect` 실행 |
| 이후 | | 브라우저가 Layout → Paint → Composite |

- Render Phase는 **여러 번 실행되거나 버려질 수 있으므로** 렌더 중에 API 호출이나 DOM 조작 같은 부수 효과를 하면 안 됨

### 용어 정리

| 용어 | 의미 |
|---|---|
| Render | 컴포넌트 함수를 호출해서 **새 React Element 트리를 만드는 것** (DOM 갱신이 아님) |
| Reconciliation | 이전/새 트리를 **비교해서 변경 사항을 계산하는** 과정 |
| Commit | 계산된 변경 사항을 **실제 DOM에 적용하는** 것 |

---

## 5. React Rendering

### 렌더링이란

- React에서 **Render = 컴포넌트 함수를 호출해서 UI 설명(Element)을 만드는 것이고**, **DOM을 갱신하는 것과는 다름**
- 렌더 결과가 이전과 같으면 Commit 단계에서 **실제 DOM은 바뀌지 않음**
- 그래도 렌더 자체(함수 호출, Diff)에는 비용이 들기 때문에 **불필요한 렌더를** 줄이는 것이 최적화의 핵심

### 전체 흐름

```
Trigger (setState 등)
   ▼
Render Phase   컴포넌트 함수 호출 → 새 Element 트리 → 이전과 Diff (Reconciliation)
   ▼
Commit Phase   DOM 변경 적용 → ref 연결 → useLayoutEffect 실행
   ▼
Browser        Layout → Paint → Composite  (화면 갱신, 10주차)
   ▼
useEffect      보통 화면이 그려진 뒤 비동기로 실행
```

### 리렌더링이 일어나는 경우

| 트리거 | 설명 |
|---|---|
| 초기 렌더 | `root.render(<App />)` |
| **state 변경** | `setState` 호출. 값이 이전과 같으면(`Object.is`) 건너뜀 |
| **부모의 리렌더** | 부모가 렌더되면 **자식도 기본적으로 모두 다시 렌더됨** (props가 같아도) |
| **Context 값 변경** | 그 Context를 구독하는 모든 컴포넌트 |

- "props가 바뀌어서 리렌더된다"는 표현은 정확히는 **부모가 리렌더되었기 때문임**
- 이 **"부모가 렌더되면 자식도 렌더된다"가** 불필요한 렌더의 가장 큰 원인

### 렌더링의 규칙: 순수해야 한다

```jsx
let count = 0;

function Bad() {
  count += 1;                        // 렌더 중에 바깥 변수 변경 (부수 효과)
  document.title = 'x';              // 렌더 중에 DOM 조작
  return <div>{count}</div>;
}
```

- 같은 props/state라면 **항상 같은 JSX를** 반환해야 함
- 부수 효과는 **이벤트 핸들러나 `useEffect`** 안에서 수행
- 개발 모드의 `<StrictMode>`는 순수성 위반을 찾아내려고 **컴포넌트를 두 번 호출함**

### State 업데이트와 Batching

- state는 **렌더링 시점의 스냅샷이라서**, `setState`를 호출해도 **현재 렌더의 변수는 바뀌지 않음** (8주차 클로저)
- React 18부터는 이벤트 핸들러, 타이머, Promise 안에서도 **여러 `setState`를 모아서 한 번만 렌더링함** (Automatic Batching)

```js
function handleClick() {
  setCount(count + 1);
  setCount(count + 1);
  setCount(count + 1);
  // count는 이 렌더의 스냅샷(예: 0)이라서 세 줄 모두 setCount(1) → 결과 1
}

function handleClick2() {
  setCount((c) => c + 1);
  setCount((c) => c + 1);
  setCount((c) => c + 1);
  // 이전 state를 받아 이어서 계산 → 결과 3
}
```

```js
// 객체/배열은 새 참조로 교체해야 변경이 감지됨 (1주차 불변성)
setUser({ ...user, name: 'lee' });          // 올바름
user.name = 'lee'; setUser(user);           // 같은 참조라 갱신되지 않음
```

### 불필요한 렌더링 줄이기

| 방법 | 핵심 |
|---|---|
| State 위치 조정 | 상태를 **쓰는 곳 가까이로** 내려서 영향 범위를 좁힘 |
| 컴포넌트 합성 (`children`) | 상태가 바뀌어도 `children`으로 받은 요소는 다시 만들어지지 않음 |
| `React.memo` | props가 같으면(얕은 비교) 렌더를 건너뜀 |
| `useMemo` / `useCallback` | 계산 결과와 함수의 **참조를 유지** |
| Context 분리 | 값이 자주 바뀌는 Context와 아닌 Context를 나눔 |
| 리스트 가상화 | 화면에 보이는 항목만 렌더 (1장) |
| 코드 분할 | `React.lazy` + `Suspense`로 필요할 때 로드 (5주차) |
| 동시성 기능 | `useTransition`, `useDeferredValue`로 급하지 않은 업데이트를 뒤로 미룸 |

### 1) State를 쓰는 곳 가까이로

```jsx
// 나쁨: 입력할 때마다 App 전체(무거운 ExpensiveList 포함)가 리렌더
function App() {
  const [text, setText] = useState('');
  return (
    <>
      <input value={text} onChange={(e) => setText(e.target.value)} />
      <ExpensiveList />
    </>
  );
}

// 좋음: 입력 state를 별도 컴포넌트로 분리 → 입력해도 ExpensiveList는 리렌더되지 않음
function SearchBox() {
  const [text, setText] = useState('');
  return <input value={text} onChange={(e) => setText(e.target.value)} />;
}

function App() {
  return (
    <>
      <SearchBox />
      <ExpensiveList />
    </>
  );
}
```

### 2) children으로 합성

```jsx
function MouseTracker({ children }) {
  const [pos, setPos] = useState({ x: 0, y: 0 });
  return (
    <div onMouseMove={(e) => setPos({ x: e.clientX, y: e.clientY })}>
      <Cursor pos={pos} />
      {children}        {/* 부모(App)가 만든 element라서, pos가 바뀌어도 새로 만들어지지 않음 */}
    </div>
  );
}

<MouseTracker>
  <ExpensiveList />     {/* MouseTracker의 state 변경에 영향받지 않음 */}
</MouseTracker>
```

### 3) React.memo와 참조 안정화

```jsx
const Item = React.memo(function Item({ item, onSelect }) {
  return <li onClick={() => onSelect(item.id)}>{item.name}</li>;
});
```

- `memo`는 props를 **얕은 비교하는데**, 부모가 렌더될 때마다 **새 객체/함수를 만들면 매번 "다른 props"로** 판단되어 효과가 없음

```jsx
function List({ items }) {
  const [selected, setSelected] = useState(null);

  // 렌더마다 새 함수 → memo(Item)가 무력화됨
  // const handleSelect = (id) => setSelected(id);

  // 참조를 유지해서 props가 같게 보이도록
  const handleSelect = useCallback((id) => setSelected(id), []);

  return items.map((item) => (
    <Item key={item.id} item={item} onSelect={handleSelect} />
  ));
}
```

| 훅 | 용도 |
|---|---|
| `useMemo(fn, deps)` | **비싼 계산 결과를** 캐싱, 또는 객체/배열의 참조 유지 |
| `useCallback(fn, deps)` | **함수의 참조를** 유지 |

- 메모이제이션도 **비교 비용과 메모리를 쓰므로**, 렌더가 실제로 느린 곳에서 **측정한 뒤** 적용하는 것이 원칙 (남용하면 코드만 복잡해짐)
- 최근에는 **React Compiler가** 빌드 시 자동으로 메모이제이션을 적용해주는 방향이라, 수동 `memo`/`useMemo`/`useCallback`의 필요성이 줄어듦 (프로젝트의 도입 여부와 버전 확인 필요)

### 4) Context 값 안정화

```jsx
// 나쁨: 렌더마다 새 객체 → Provider가 리렌더될 때마다 모든 consumer가 리렌더
<ThemeContext.Provider value={{ theme, toggle }}>

// 좋음: 값의 참조를 유지
const value = useMemo(() => ({ theme, toggle }), [theme, toggle]);
<ThemeContext.Provider value={value}>
```

- Context 값이 바뀌면 `React.memo`로도 막을 수 없어서, **자주 바뀌는 값은 Context를 분리하거나** selector를 지원하는 상태 라이브러리(Zustand 등)를 사용

### useEffect vs useLayoutEffect

| | useEffect | useLayoutEffect |
|---|---|---|
| 실행 시점 | 보통 **화면이 그려진 뒤** 비동기로 | **Commit 직후, 화면이 그려지기 전** 동기로 |
| 브라우저 Paint | 막지 않음 | **막음** (끝나야 Paint) |
| 용도 | 데이터 요청, 구독, 로깅 등 대부분의 부수 효과 | DOM을 **측정한** 뒤 **깜빡임 없이** 즉시 보정 |

```jsx
function Tooltip({ targetRef, children }) {
  const ref = useRef(null);
  const [pos, setPos] = useState({ top: 0, left: 0 });

  useLayoutEffect(() => {
    const rect = targetRef.current.getBoundingClientRect();   // 레이아웃 값 읽기
    setPos({ top: rect.bottom, left: rect.left });            // Paint 전에 위치 보정
  }, [targetRef]);

  return <div ref={ref} style={{ position: 'fixed', ...pos }}>{children}</div>;
}
```

- `useEffect`로 같은 일을 하면 **잘못된 위치로 한 번 그려진 뒤 움직이는 깜빡임이** 생길 수 있음
- 하지만 `useLayoutEffect`는 Paint를 막고 강제 레이아웃을 일으키므로 **꼭 필요한 경우에만** 사용

### 측정하기

- **React DevTools Profiler**: 어떤 컴포넌트가 **왜**, **얼마나 오래** 렌더됐는지 기록. 설정에서 "Record why each component rendered"를 켜면 이유까지 확인 가능
- **Highlight updates when components render**: 리렌더되는 컴포넌트에 테두리가 깜빡임
- 원칙: **측정 → 병목 확인 → 최적화 → 다시 측정**. 느리지 않은 곳을 미리 최적화하지 않음

```jsx
<Profiler id="List" onRender={(id, phase, ms) => console.log(id, phase, ms.toFixed(1))}>
  <List />
</Profiler>
```

---

## 6. 정리

| 개념 | 핵심 |
|---|---|
| Reflow | 위치/크기 재계산. 가장 비쌈. 읽기/쓰기 분리, Observer 사용, 범위 제한, 가상화로 줄임 |
| Repaint | 모양만 다시 그림. `transform`/`opacity`로 Composite만 일어나게, 비싼 효과 회피 |
| Virtual DOM | DOM을 JS 객체로 표현해 Diff 후 필요한 변경만 반영. "더 빠르다"가 아니라 "예측 가능하고 충분히 빠르다" |
| Reconciliation | 타입이 다르면 교체, 같으면 갱신, 리스트는 key로 식별. Fiber로 중단 가능한 작업 단위 |
| React Rendering | Render(함수 호출) → Commit(DOM 반영). 부모가 렌더되면 자식도 렌더, 순수해야 함 |

### 최적화의 방향

```
① 측정           Profiler, Performance 패널로 병목 확인
② 렌더 줄이기     State 위치, memo, Context 분리 (React 레벨)
③ DOM 변경 줄이기 key, 가상화, 배치 처리 (DOM 레벨)
④ 브라우저 비용   Reflow 회피 > Repaint 회피 > Composite만 (렌더링 파이프라인 레벨)
```

| 레벨 | 줄이는 것 | 대표 수단 |
|---|---|---|
| React | 불필요한 렌더 | State 내리기, `children`, `memo`, `useCallback`, Context 분리 |
| Reconciliation | 불필요한 DOM 변경/재생성 | 안정적인 `key`, 컴포넌트를 바깥에서 정의 |
| DOM | 노드 수, 변경 횟수 | 가상화, `DocumentFragment`, 한 번에 변경 |
| 렌더링 파이프라인 | Layout/Paint | 읽기/쓰기 분리, `transform`/`opacity`, `will-change`(필요할 때만), Observer |

### 확인 질문 체크리스트

| 구분 | 점검 항목 |
|---|---|
| Reflow | 쓰기 사이에 레이아웃 값을 읽고 있지 않은가, 스크롤/리사이즈 핸들러에서 측정하지 않는가 |
| Repaint | 애니메이션에 `top`/`left`/`box-shadow`를 쓰고 있지 않은가 |
| Reconciliation | `key`가 고유하고 안정적인가, 렌더 안에서 컴포넌트를 정의하지 않았는가 |
| Rendering | 상태가 너무 위에 있지 않은가, props로 매번 새 객체/함수를 넘기지 않는가 |
| 측정 | Profiler와 Performance 패널로 실제 병목을 확인했는가 |