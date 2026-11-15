# 11주차 - DOM / Rendering 최적화

## 1. Reflow

- **Reflow(Layout):** DOM이나 스타일 변경으로 요소의 **크기·위치(기하학 정보)가 바뀌어 Layout을 다시 계산**하는 과정
- Reflow가 일어나면 그 뒤의 **Paint와 Composite도 모두 다시 수행**되므로 렌더링 파이프라인에서 **가장 비용이 큰 작업**
- 개념과 발생 조건은 10주차 Layout에서 정리했고, 이번 주는 **어떻게 줄일지**에 집중

```
 style/DOM change ──▶ Style ──▶ Layout ──▶ Paint ──▶ Composite
                                  ▲
                                  └ Reflow starts here (most expensive)
```

### 📍 Reflow의 영향 범위

- 한 요소의 변경이 **자식, 형제, 부모**로 전파될 수 있음 (부모 크기가 자식에, 자식 내용이 부모 높이에 영향)
- `<body>`나 최상위 컨테이너의 변경, 창 크기 변경은 **문서 전체의 Reflow**를 일으킴
- DOM 트리의 **아래쪽(말단) 요소**를 변경할수록, 레이아웃에서 **분리된 요소**(`position: absolute/fixed`)일수록 영향 범위가 좁음

### 📍 Reflow를 줄이는 방법

| 방법                                | 설명                                                                                       |
| ----------------------------------- | ------------------------------------------------------------------------------------------ |
| 스타일을 한 번에 변경               | `style`을 여러 번 바꾸지 말고 **클래스 하나를 토글**하거나 `cssText`로 묶어서 변경         |
| 읽기와 쓰기 분리                    | `offsetHeight` 같은 레이아웃 값 **읽기를 먼저 모으고, 쓰기는 나중에** → 강제 동기 레이아웃 방지 |
| DOM 변경을 모아서 한 번에           | `DocumentFragment`나 문자열로 만든 뒤 **한 번만 삽입**                                     |
| 오프라인에서 수정                   | `display: none`으로 렌더 트리에서 뺀 뒤 여러 번 수정하고 다시 표시 (Reflow 2번으로 줄임)    |
| 애니메이션 요소는 레이아웃에서 분리 | `position: absolute/fixed`로 띄워 **주변 요소에 영향이 가지 않게** 함                      |
| Layout 대신 Composite 속성 사용     | `top/left/width` 대신 **`transform`**, `display` 토글 대신 **`opacity`**                   |
| 쓰기 시점을 프레임에 맞추기         | DOM 쓰기를 **`requestAnimationFrame`** 안에서 실행해 렌더링 직전에 한 번에 처리            |
| CSS Containment                     | `contain: layout` / `content-visibility: auto`로 **Reflow 범위를 요소 내부로 제한**        |

```js
// Bad: 스타일을 하나씩 바꾸고, 매 반복마다 Layout 값을 읽음
items.forEach((item) => {
  item.style.width = list.offsetWidth + "px"; // 읽기 → 쓰기 반복 (Layout Thrashing)
  item.style.margin = "4px";
});

// Good: 읽기는 한 번, 변경은 클래스로 묶어서
const width = list.offsetWidth;
items.forEach((item) => {
  item.style.width = width + "px";
  item.classList.add("spaced");
});
```

```js
// DOM 삽입을 한 번으로
const fragment = document.createDocumentFragment();
data.forEach((d) => {
  const li = document.createElement("li");
  li.textContent = d.name;
  fragment.appendChild(li); // 화면에 붙어있지 않아 Reflow 없음
});
list.appendChild(fragment); // 여기서 Reflow 1번
```

### 📍 CSS Containment

- 브라우저에게 **"이 요소 내부의 변경은 바깥에 영향을 주지 않는다"** 고 알려서 Layout/Paint 범위를 제한하는 속성

| 속성                         | 효과                                                                                      |
| ---------------------------- | ----------------------------------------------------------------------------------------- |
| `contain: layout`            | 내부 레이아웃 변경이 **바깥 요소의 Layout에 영향을 주지 않음**                            |
| `contain: paint`             | 자식이 요소 **경계 밖으로 그려지지 않음** → 화면 밖이면 Paint 생략 가능                   |
| `contain: strict / content`  | 여러 containment를 한 번에 적용                                                           |
| `content-visibility: auto`   | **화면 밖에 있는 요소의 Layout·Paint를 건너뜀** (긴 목록, 긴 문서에 효과적)              |

```css
.card {
  content-visibility: auto;
  contain-intrinsic-size: auto 300px; /* 렌더링 전 예상 높이 → 스크롤바 튐 방지 */
}
```

### 💡 Layout Thrashing이란 무엇이고 어떻게 해결하나요?

**⇒ 브라우저는 스타일 변경을 모아두었다가 다음 렌더링 시점에 한 번에 Layout하는데, 스타일을 바꾼 직후 `offsetHeight`, `getBoundingClientRect()` 같은 레이아웃 값을 읽으면 정확한 값을 위해 그 자리에서 즉시 Layout을 수행함. 반복문 안에서 이런 쓰기와 읽기를 번갈아 하면 매 반복마다 Layout이 강제로 실행되는데 이것이 Layout Thrashing임. 읽기를 먼저 모두 모아서 하고 쓰기는 그 뒤에 한 번에 하거나, 쓰기를 `requestAnimationFrame`으로 미뤄 읽기와 쓰기 단계를 분리해서 해결함**

### 💡 긴 목록을 렌더링할 때 Reflow 비용을 줄이는 방법

**⇒ 수천 개의 항목을 모두 DOM에 그리면 Layout 대상이 많아져 초기 렌더링과 스크롤 시 Reflow 비용이 커짐. 가장 효과적인 방법은 화면에 보이는 항목만 DOM에 렌더링하는 가상 스크롤(Virtualization, `react-window`, `TanStack Virtual`)이고, DOM을 유지해야 한다면 `content-visibility: auto`로 화면 밖 항목의 Layout과 Paint를 건너뛸 수 있음. 또한 항목 추가 시에는 `DocumentFragment`나 한 번의 상태 업데이트로 묶어서 삽입해 Reflow 횟수를 줄임**

---

## 2. Repaint

- **Repaint:** 요소의 **위치·크기는 그대로인 채 색상, 배경, 그림자 같은 겉모습만 바뀌어 다시 그리는** 과정
- Layout은 건너뛰지만 **Paint와 Composite는 다시 수행**됨
- Reflow가 일어나면 Repaint는 반드시 따라오지만, **Repaint는 Reflow 없이 단독으로** 일어날 수 있음

### 📍 단계별 비용 비교

| 변경 유형                   | 실행 단계                          | 대표 속성                                              | 비용     |
| --------------------------- | ---------------------------------- | ------------------------------------------------------ | -------- |
| Reflow                      | Layout → Paint → Composite         | `width`, `height`, `margin`, `font-size`, `display`    | **높음** |
| Repaint                     | Paint → Composite                  | `color`, `background`, `box-shadow`, `visibility`      | 중간     |
| Composite only              | Composite                          | `transform`, `opacity` (별도 레이어일 때)              | **낮음** |

### 📍 Repaint 비용에 영향을 주는 요소

- **다시 그리는 영역의 크기:** 작은 버튼 색 변경보다 화면 전체 배경 변경이 훨씬 비쌈
- **그리기 자체가 비싼 속성:** `box-shadow`(특히 큰 blur), `filter: blur()`, `border-radius` + 그림자, 복잡한 그라데이션
- **빈도:** 스크롤·`mousemove`·애니메이션처럼 **매 프레임** 발생하면 16.7ms 예산을 쉽게 초과

### 📍 Repaint를 줄이는 방법

| 방법                                 | 설명                                                                                          |
| ------------------------------------ | --------------------------------------------------------------------------------------------- |
| `transform` / `opacity`로 애니메이션 | 레이어를 다시 그리지 않고 **Composite만 수행** (Compositor Thread에서 처리)                   |
| 자주 바뀌는 요소를 레이어로 분리     | `will-change: transform`으로 분리하면 해당 요소의 변경이 **다른 영역의 Repaint를 일으키지 않음** |
| 비싼 효과를 미리 그려두기            | 그림자가 바뀌는 hover 효과는 **그림자가 있는 가상 요소의 `opacity`만 변경**                   |
| 이벤트 빈도 줄이기                   | `scroll`, `resize` 핸들러는 **throttle / requestAnimationFrame**으로 프레임당 1번만 처리     |
| 레이어 남용 피하기                   | 레이어가 너무 많으면 메모리와 합성 비용이 커지므로 **필요한 요소에만** 적용                   |

```css
/* Bad: hover마다 큰 box-shadow를 다시 그림 */
.card:hover {
  box-shadow: 0 16px 32px rgba(0, 0, 0, 0.3);
}

/* Good: 그림자는 미리 그려두고 opacity만 변경 → Composite만 발생 */
.card { position: relative; }
.card::after {
  content: "";
  position: absolute;
  inset: 0;
  box-shadow: 0 16px 32px rgba(0, 0, 0, 0.3);
  opacity: 0;
  transition: opacity 0.3s;
}
.card:hover::after { opacity: 1; }
```

### 📍 개발자 도구로 확인하기

| 도구                                   | 확인할 수 있는 것                                                     |
| -------------------------------------- | --------------------------------------------------------------------- |
| Performance 패널                       | 프레임별 **Layout(보라색) / Paint(초록색)** 작업 시간, Long Task      |
| Rendering → Paint flashing             | **Repaint되는 영역을 초록색으로 표시**                                |
| Rendering → Layout Shift Regions       | 레이아웃이 이동한 영역 표시 (CLS 원인 파악)                           |
| Layers 패널                            | 생성된 **컴포지팅 레이어**와 그 이유, 메모리 사용량                   |

### 💡 Reflow와 Repaint를 줄이기 위해 어떤 노력을 할 수 있나요?

**⇒ 먼저 변경할 속성을 고를 때 Layout을 일으키는 속성 대신 `transform`, `opacity`처럼 Composite만 일으키는 속성을 우선 사용함. DOM 조작은 `DocumentFragment`나 클래스 토글로 묶어서 한 번에 반영하고, 레이아웃 값 읽기와 스타일 쓰기를 분리해 강제 동기 레이아웃을 피함. 스크롤·리사이즈처럼 자주 발생하는 이벤트는 throttle이나 `requestAnimationFrame`으로 빈도를 줄이고, 긴 목록은 가상 스크롤이나 `content-visibility`로 렌더링 대상 자체를 줄임. 마지막으로 Performance 패널과 Paint flashing으로 실제 병목을 측정한 뒤 최적화함**

---

## 3. Virtual DOM

- **Virtual DOM:** 실제 DOM을 흉내 낸 **가벼운 JS 객체 트리**로, UI의 상태를 메모리에 표현한 것
- 상태가 바뀌면 **새 Virtual DOM을 만들고 이전 Virtual DOM과 비교(diff)** 해서, **바뀐 부분만 실제 DOM에 반영**함
- React, Vue 등이 사용하며, 개발자는 DOM을 직접 조작하지 않고 **"상태에 따라 UI가 어떻게 보여야 하는지"만 선언**하면 됨

### 📍 Virtual DOM의 형태

```jsx
<div className="box">
  <h1>Hello</h1>
</div>
```

```js
// JSX는 React.createElement 호출로 변환되고, 결과는 단순한 JS 객체
{
  type: "div",
  props: {
    className: "box",
    children: {
      type: "h1",
      props: { children: "Hello" },
    },
  },
}
```

- 실제 DOM 노드는 수백 개의 프로퍼티와 브라우저 내부 연결을 가진 무거운 객체지만, Virtual DOM 노드는 **필요한 정보만 담은 일반 객체**라 생성·비교 비용이 작음

### 📍 Virtual DOM의 동작 과정

```
 state change
      │
      ▼
 1. Render  : create new Virtual DOM tree
      │
      ▼
 2. Diff    : compare with previous Virtual DOM   (Reconciliation)
      │
      ▼
 3. Commit  : apply only the changes to the real DOM
      │
      ▼
 Browser    : Style ──▶ Layout ──▶ Paint ──▶ Composite
```

### 📍 Virtual DOM이 해결하는 문제

| 문제 (직접 DOM 조작)                                              | Virtual DOM의 해결 방식                                                         |
| ----------------------------------------------------------------- | ------------------------------------------------------------------------------- |
| 상태가 바뀔 때마다 **어떤 DOM을 어떻게 바꿀지 직접 관리**해야 함  | **선언형 UI:** 상태만 바꾸면 라이브러리가 필요한 DOM 변경을 계산                |
| 여러 변경이 생기면 **DOM을 여러 번 건드려 Reflow가 반복**됨       | 변경 사항을 **모아서(batch) 한 번에 커밋**                                      |
| UI가 복잡해질수록 **상태와 화면이 어긋나는 버그** 발생            | 화면은 항상 **현재 상태로부터 다시 계산**되므로 일관성 유지                     |
| 브라우저 환경에만 종속                                            | 같은 Virtual DOM을 **다른 환경으로 렌더링** 가능 (React Native, SSR 문자열 등) |

### 💡 Virtual DOM은 실제 DOM 조작보다 항상 빠른가요?

**⇒ 항상 빠르지는 않음. Virtual DOM도 결국 마지막에 실제 DOM을 조작하므로, 바뀔 부분을 정확히 알고 직접 최소한으로 조작하는 코드보다는 비교(diff) 과정만큼 오히려 느림. Virtual DOM의 진짜 가치는 개발자가 DOM 변경을 일일이 관리하지 않고 상태만 선언해도 충분히 빠른 성능을 얻게 해주는 것, 즉 생산성과 유지보수성을 해치지 않으면서 불필요한 DOM 조작을 자동으로 줄여주는 데 있음. 실제로 Svelte나 SolidJS처럼 Virtual DOM 없이 컴파일 단계나 세밀한 반응성(Signal)으로 바뀐 DOM만 직접 업데이트하는 프레임워크도 있음**

### 💡 Virtual DOM과 Shadow DOM의 차이

**⇒ 이름은 비슷하지만 목적이 다름. Virtual DOM은 React 같은 라이브러리가 효율적인 업데이트를 위해 메모리에 만드는 JS 객체 트리이고, Shadow DOM은 브라우저가 제공하는 표준 기술로 컴포넌트 내부의 DOM과 CSS를 바깥 문서와 격리(캡슐화)하기 위해 사용함. `<video>` 태그의 재생 버튼이나 Web Components가 Shadow DOM으로 내부 구조를 숨기는 대표적인 예임**

---

## 4. Reconciliation

- **재조정(Reconciliation):** 이전 Virtual DOM 트리와 새 Virtual DOM 트리를 **비교해서 실제 DOM에 반영할 최소한의 변경을 찾는 과정** (Diffing 알고리즘)
- 두 트리를 완벽하게 비교하는 일반 알고리즘은 **O(n³)** 이라 1000개 요소만 돼도 10억 번 비교가 필요함
- React는 **두 가지 가정(휴리스틱)** 으로 이를 **O(n)** 으로 줄임

### 📍 React의 비교 규칙 (Diffing 휴리스틱)

| 가정                                                | 동작                                                                                              |
| --------------------------------------------------- | ------------------------------------------------------------------------------------------------- |
| 1. **타입이 다른 요소는 다른 트리를 만든다**        | `<div>` → `<span>`, `<A />` → `<B />`처럼 타입이 바뀌면 **하위 트리 전체를 버리고 새로 생성** (state도 초기화) |
| 2. **`key`로 어떤 자식이 유지되는지 알 수 있다**    | 같은 `key`를 가진 자식은 **같은 요소로 보고 재사용**하며, 순서가 바뀌면 이동만 함                 |

- 타입이 같은 DOM 요소: **바뀐 속성(attribute, style)만 업데이트**
- 타입이 같은 컴포넌트: **인스턴스(state)를 유지**한 채 새 props로 다시 렌더링하고 그 결과를 재귀적으로 비교
- 트리는 **같은 레벨끼리만 비교**하며, 다른 레벨로 이동한 요소는 삭제 후 새로 생성함

```jsx
// 타입이 바뀌면 하위 트리 전체가 새로 생성됨 → Counter의 state 초기화
{isEditing ? <div><Counter /></div> : <section><Counter /></section>}
```

### 📍 key의 역할

```jsx
// key가 없거나 index를 key로 쓰면: 맨 앞에 추가 시 모든 항목이 "내용이 바뀐 것"으로 판단됨
// 고유한 id를 key로 쓰면: 새 항목 하나만 삽입하고 기존 항목은 재사용

{todos.map((todo) => (
  <TodoItem key={todo.id} todo={todo} />
))}
```

```
 before            after (prepend "C")
 key=a  "A"        key=c  "C"   <- new: insert only this
 key=b  "B"        key=a  "A"   <- same key: reuse
                   key=b  "B"   <- same key: reuse
```

| key 선택                   | 결과                                                                                                  |
| -------------------------- | ----------------------------------------------------------------------------------------------------- |
| 고유하고 안정적인 id       | 추가·삭제·정렬 시 **정확히 바뀐 항목만** 업데이트                                                     |
| 배열 index                 | 순서가 바뀌면 **다른 항목이 같은 key를 받아** 불필요한 업데이트, **input 값·state가 엉뚱한 항목에 남는 버그** |
| `Math.random()` 등 매번 새 값 | 매 렌더링마다 key가 바뀌어 **모든 항목을 삭제 후 재생성** (state 유실, 성능 저하)                    |

- 반대로 **`key`를 일부러 바꿔서 컴포넌트를 초기화**하는 용도로 쓰기도 함 (`<Profile key={userId} />`)

### 📍 React Fiber

- React 16부터 도입된 **새로운 재조정 엔진**으로, 렌더링 작업을 **Fiber라는 작은 작업 단위로 쪼개서** 처리함
- 기존(Stack Reconciler)은 재귀로 트리 전체를 한 번에 비교해서 **도중에 멈출 수 없었고**, 큰 트리를 비교하는 동안 메인 스레드를 점유해 입력이 버벅였음

| 단계            | 설명                                                                                             | 중단 가능 여부 |
| --------------- | ------------------------------------------------------------------------------------------------ | -------------- |
| Render Phase    | 컴포넌트를 호출해 새 트리를 만들고 **이전 트리와 비교해 변경 사항을 계산**                       | **가능** (우선순위 높은 작업이 오면 양보) |
| Commit Phase    | 계산된 변경 사항을 **실제 DOM에 한 번에 반영**하고 `useLayoutEffect`, `useEffect` 실행           | **불가능** (화면 일관성을 위해 동기 실행) |

- 작업마다 **우선순위(Lane)** 를 매겨, 사용자 입력 같은 급한 업데이트를 먼저 처리하고 덜 급한 업데이트는 뒤로 미룸 → React 18의 **Concurrent Rendering**(`useTransition`, `useDeferredValue`)의 기반

### 💡 리스트 렌더링 시 key에 index를 쓰면 안 되는 이유

**⇒ React는 key로 이전 렌더링의 어떤 요소가 이번 렌더링의 어떤 요소인지 판단하는데, index를 key로 쓰면 맨 앞에 항목을 추가하거나 정렬했을 때 같은 index(key)에 다른 데이터가 오게 됨. 그러면 React는 기존 요소의 내용이 바뀐 것으로 판단해 모든 항목을 업데이트하고, 컴포넌트 내부의 state나 비제어 input의 입력값은 key를 따라 그대로 남아서 다른 항목에 엉뚱하게 표시되는 버그가 생김. 따라서 데이터의 고유한 id를 key로 사용해야 하고, 순서가 절대 바뀌지 않는 정적인 목록일 때만 index를 써도 괜찮음**

### 💡 React Fiber가 해결한 문제는 무엇인가요?

**⇒ 이전 Stack Reconciler는 재귀로 트리 전체를 한 번에 비교했기 때문에 한 번 시작하면 중간에 멈출 수 없었고, 큰 컴포넌트 트리를 업데이트하는 동안 메인 스레드를 오래 점유해 타이핑이나 애니메이션이 끊겼음. Fiber는 렌더링 작업을 작은 단위로 쪼개고 각 작업에 우선순위를 부여해서, Render Phase를 도중에 멈추고 사용자 입력처럼 급한 작업을 먼저 처리한 뒤 다시 이어서 할 수 있게 함. 다만 실제 DOM에 반영하는 Commit Phase는 화면이 중간 상태로 보이지 않도록 한 번에 동기적으로 실행함**

---

## 5. React Rendering

- React에서 **렌더링**이란 브라우저가 화면을 그리는 것이 아니라, **React가 컴포넌트 함수를 호출해 새 Virtual DOM(React Element)을 계산하는 것**을 의미함
- 렌더링 결과 바뀐 것이 있을 때만 **Commit 단계에서 실제 DOM이 변경**되고, 그 뒤에 브라우저 렌더링(Layout, Paint)이 일어남
- 즉 **"리렌더링 = DOM 변경"이 아님** → 렌더링은 됐지만 DOM은 그대로일 수 있음

### 📍 React 렌더링 과정

```
 Trigger ──▶ Render ───────────▶ Commit ─────────▶ Browser Paint
 (setState)  (call components,   (update DOM,      (Layout, Paint)
              diff with prev)     run effects)
```

| 단계      | 설명                                                                                         |
| --------- | -------------------------------------------------------------------------------------------- |
| Trigger   | 첫 렌더링(`createRoot().render()`) 또는 **state 변경**으로 렌더링 요청                        |
| Render    | 컴포넌트 함수를 호출해 JSX를 반환받고, **이전 결과와 비교(Reconciliation)**. 순수해야 함     |
| Commit    | 변경된 부분만 **실제 DOM에 반영**, ref 연결, `useLayoutEffect` 실행                           |
| Paint 후  | 브라우저가 화면을 그린 뒤 **`useEffect` 실행**                                               |

### 📍 리렌더링이 발생하는 조건

| 조건                       | 설명                                                                                  |
| -------------------------- | ------------------------------------------------------------------------------------- |
| **state 변경**             | `setState`로 **이전 값과 다른 값**이 들어오면 (`Object.is`로 비교, 같으면 생략)       |
| **부모 컴포넌트 리렌더링** | 부모가 렌더링되면 **props가 바뀌지 않아도 자식도 기본적으로 모두 리렌더링**           |
| **Context 값 변경**        | `useContext`로 구독 중인 Context의 value가 바뀌면 해당 컴포넌트 리렌더링              |

- **props 변경 자체는 리렌더링의 원인이 아님** → props가 바뀌는 것은 부모가 리렌더링되었기 때문
- 객체·배열 state를 **직접 수정(mutate)** 하면 참조가 같아 리렌더링되지 않으므로, 항상 **새 객체로 교체**해야 함

```jsx
const [user, setUser] = useState({ name: "seyun" });

user.name = "lim";
setUser(user); // 참조가 같아서 리렌더링 X

setUser({ ...user, name: "lim" }); // 새 객체 → 리렌더링 O
```

### 📍 Batching (일괄 처리)

- 여러 `setState`를 **하나의 리렌더링으로 묶어서** 처리
- React 18부터는 이벤트 핸들러뿐 아니라 `setTimeout`, Promise, 네이티브 이벤트 안에서도 **자동으로 Batching** (Automatic Batching)

```jsx
function handleClick() {
  setCount((c) => c + 1);
  setFlag((f) => !f);
  // 리렌더링은 1번만 발생
}

setTimeout(() => {
  setCount((c) => c + 1);
  setFlag((f) => !f);
  // React 17: 2번 렌더링 / React 18+: 1번 렌더링
}, 1000);
```

### 📍 불필요한 리렌더링 줄이기

| 방법                              | 설명                                                                                              |
| --------------------------------- | ------------------------------------------------------------------------------------------------- |
| state를 필요한 곳 가까이 두기     | state를 **실제로 쓰는 하위 컴포넌트로 내리면** 리렌더링 범위가 줄어듦 (State Colocation)          |
| children으로 넘기기               | 자주 바뀌는 state를 가진 컴포넌트가 무거운 컴포넌트를 **`children`으로 받으면** 함께 리렌더링되지 않음 |
| `React.memo`                      | **props가 얕은 비교로 같으면** 자식 컴포넌트 렌더링을 건너뜀                                      |
| `useMemo`                         | 비용이 큰 **계산 결과**를 의존성이 바뀔 때만 다시 계산                                            |
| `useCallback`                     | **함수의 참조**를 유지해 `memo`된 자식에 넘길 때 props가 바뀌지 않게 함                           |
| Context 분리                      | 자주 바뀌는 값과 잘 안 바뀌는 값을 **다른 Context로 분리**, 또는 Zustand 같은 selector 기반 상태 관리 |
| `useTransition` / `useDeferredValue` | 무거운 업데이트를 **낮은 우선순위**로 처리해 입력 반응성을 유지                                |
| React Compiler                    | 빌드 시점에 컴포넌트를 분석해 **memo/useMemo/useCallback을 자동으로 적용**                        |

```jsx
// memo된 자식에게 매번 새 함수/객체를 넘기면 memo가 무의미해짐
const Child = memo(function Child({ onClick, style }) {
  return <button onClick={onClick} style={style}>click</button>;
});

function Parent() {
  const [count, setCount] = useState(0);

  const handleClick = useCallback(() => console.log("click"), []); // 참조 유지
  const style = useMemo(() => ({ color: "red" }), []); // 참조 유지

  return (
    <>
      <p>{count}</p>
      <button onClick={() => setCount((c) => c + 1)}>+</button>
      <Child onClick={handleClick} style={style} />
    </>
  );
}
```

### 📍 useEffect vs useLayoutEffect

| 구분        | `useEffect`                                     | `useLayoutEffect`                                         |
| ----------- | ----------------------------------------------- | --------------------------------------------------------- |
| 실행 시점   | 브라우저가 **화면을 그린 후** 비동기로 실행     | DOM 변경 후 **화면을 그리기 전**에 동기로 실행            |
| 화면 차단   | 차단하지 않음                                   | 실행이 끝날 때까지 **Paint를 막음**                       |
| 용도        | 데이터 요청, 구독, 로깅 등 대부분의 부수 효과   | DOM 크기를 읽어 위치를 조정하는 등 **깜빡임 방지**가 필요할 때 |

### 💡 React에서 리렌더링이 발생하는 조건은 무엇인가요?

**⇒ 컴포넌트의 state가 이전과 다른 값으로 바뀌었을 때, 부모 컴포넌트가 리렌더링되었을 때, 구독하고 있는 Context 값이 바뀌었을 때 리렌더링됨. 흔히 props가 바뀌면 리렌더링된다고 생각하지만, 실제로는 부모가 리렌더링되면 props가 그대로여도 자식은 기본적으로 다시 렌더링되고, 이를 막고 싶을 때 `React.memo`로 props를 비교하게 하는 것임. 또한 state는 `Object.is`로 비교하므로 객체를 직접 수정하면 참조가 같아 리렌더링되지 않아서 항상 새 객체로 교체해야 함**

### 💡 `useMemo`와 `useCallback`을 모든 곳에 쓰면 좋은가요?

**⇒ 그렇지 않음. 메모이제이션도 의존성 배열을 비교하고 이전 값을 메모리에 보관하는 비용이 들기 때문에, 계산이 가볍거나 자식이 `memo`되어 있지 않은 곳에 쓰면 이득 없이 코드만 복잡해짐. `useMemo`는 실제로 비용이 큰 계산이나 `memo`된 자식에 넘기는 객체·배열에, `useCallback`은 `memo`된 자식의 props나 다른 Hook의 의존성으로 쓰이는 함수에 사용하는 것이 효과적임. 먼저 state 위치 조정이나 children 패턴으로 구조를 개선하고, React DevTools Profiler로 실제 병목을 확인한 뒤 적용하는 것이 좋으며, 최근에는 React Compiler가 이를 자동으로 처리해주기도 함**

### 💡 리렌더링이 일어나면 실제 DOM도 매번 바뀌나요?

**⇒ 아님. React의 리렌더링은 컴포넌트 함수를 다시 호출해 새 Virtual DOM을 만드는 Render 단계일 뿐이고, 이전 결과와 비교해서 바뀐 부분이 있을 때만 Commit 단계에서 실제 DOM을 수정함. 바뀐 것이 없으면 DOM은 그대로라서 브라우저의 Reflow나 Repaint도 일어나지 않음. 다만 Render 단계 자체도 컴포넌트 호출과 비교 비용이 들기 때문에, 무거운 컴포넌트가 자주 리렌더링되면 메인 스레드를 점유해 성능에 영향을 줄 수 있음**
