# 11주차 - DOM / Rendering 최적화

## 1. Reflow

### 구조

```text
DOM/스타일 변경 (width, font-size, 요소 추가·삭제 …)
        │
        ▼
Style 재계산 → Layout(Reflow) → Paint → Composite
                    ▲
         위치·크기를 다시 계산하는 단계부터 다시 실행
```

### 특징

- 요소의 위치나 크기가 바뀌어 Layout 단계를 다시 실행하는 것(10주차 Layout과 연결)
- 발생하는 경우
  - `width`, `height`, `padding`, `margin`, `border`, `top`, `font-size` 변경
  - 요소 추가·삭제, 텍스트 내용 변경, 클래스 변경으로 레이아웃 관련 스타일이 바뀔 때
  - 창 크기 변경, 폰트 로딩 완료
- 한 요소가 바뀌어도 부모, 형제, 자식의 위치가 함께 바뀔 수 있어 계산 범위가 넓어지기 쉬움
- 브라우저는 변경 사항을 모아 두었다가 다음 프레임에 한 번에 Layout을 실행함. 그런데 그 사이에 `offsetHeight`, `getBoundingClientRect()`, `scrollTop`, `getComputedStyle()`처럼 레이아웃 값을 읽으면, 최신 값을 돌려주기 위해 그 자리에서 바로 Layout을 실행함(**강제 동기 레이아웃**)

```js
// 나쁜 예: 쓰기와 읽기를 번갈아 하면 반복마다 Layout이 강제로 실행됨 (Layout Thrashing)
items.forEach((item) => {
  item.style.width = `${box.offsetWidth}px`; // 읽기 → 강제 Layout
});                                          // 쓰기 → 다음 읽기에서 또 Layout

// 좋은 예: 읽기를 먼저 모아서 한 번에, 쓰기는 그다음에 한 번에
const width = box.offsetWidth;               // 읽기 1번
items.forEach((item) => {
  item.style.width = `${width}px`;           // 쓰기만 반복 → Layout은 다음 프레임에 1번
});
```

### 정리

- Reflow는 "위치와 크기를 다시 계산하는 것"이고, 렌더링 단계 중 비용이 가장 큰 편
- Reflow 자체를 없애기보다 횟수와 범위를 줄이는 게 목표. 특히 레이아웃 값 읽기와 스타일 쓰기를 섞지 않는 것이 핵심

---

## 2. Repaint

### 구조

```text
color, background-color, box-shadow, visibility 변경
        │
        ▼
Style 재계산 → (Layout 건너뜀) → Paint → Composite
```

### 특징

- 위치와 크기는 그대로이고 색이나 그림자 같은 겉모습만 바뀌어 Paint 단계를 다시 실행하는 것
- Reflow가 일어나면 Repaint도 반드시 뒤따르지만, Repaint는 Reflow 없이 일어날 수 있음
- `transform`, `opacity`는 별도 레이어에서 처리되면 Reflow와 Repaint 없이 Composite만으로 바뀜(10주차 Composite와 연결)

| 변경 | Reflow | Repaint | Composite |
| --- | --- | --- | --- |
| `width`, `top`, `font-size` | O | O | O |
| `color`, `background-color` | X | O | O |
| `transform`, `opacity` | X | X | O |

### 정리

- Repaint는 "자리는 그대로 두고 다시 칠하는 것"으로 Reflow보다 가볍지만, 넓은 영역이나 `box-shadow`·`filter`처럼 그리기 비용이 큰 스타일은 여전히 부담
- 애니메이션은 Reflow·Repaint가 없는 `transform`, `opacity`로 구현하는 것이 기본

---

## 3. Virtual DOM

### 구조

```text
상태 변경
   │
   ▼
새 Virtual DOM 생성 (JavaScript 객체)
   │
   ▼  이전 Virtual DOM과 비교 (Diffing)
바뀐 부분만 계산
   │
   ▼
실제 DOM에 필요한 변경만 반영 (Commit)
```

```js
// <button class="primary">저장</button> 을 표현하는 객체 (단순화한 형태)
{
  type: "button",
  props: { className: "primary", children: "저장" },
}
```

### 특징

- 실제 DOM을 흉내 낸 가벼운 JavaScript 객체 트리. 상태가 바뀌면 새 트리를 만들고 이전 트리와 비교해 바뀐 부분만 실제 DOM에 반영함
- 개발자는 "어떻게 DOM을 바꿀지"가 아니라 "상태에 따라 화면이 어떻게 보여야 하는지"만 선언하면 됨(선언형 UI)
- 여러 변경을 모아 한 번에 DOM에 반영하므로 불필요한 Reflow를 줄이는 데 도움이 됨
- **Virtual DOM이 항상 직접 DOM 조작보다 빠른 것은 아님.** 비교 작업 자체도 비용이고, 필요한 DOM만 정확히 바꾸는 직접 조작이 이론적으로 가장 빠름. Svelte, Solid처럼 Virtual DOM 없이 바뀐 곳만 직접 업데이트하는 프레임워크도 있음

### 정리

- Virtual DOM의 핵심 가치는 "무조건 빠름"이 아니라 "선언형으로 작성해도 충분히 빠르게 DOM을 업데이트해 주는 것"
- 면접에서 "Virtual DOM은 왜 빠른가요?"라는 질문에는 "항상 빠른 건 아니고, 변경을 모아서 필요한 부분만 반영하기 때문에 개발 편의성 대비 성능이 좋다"로 답하는 것이 정확함

---

## 4. Reconciliation

### 구조

```text
이전 트리                새 트리
   div                     div
  ┌─┴─┐                  ┌─┴─┐
  h1  ul                 h1  ul        ← 같은 타입: 속성만 비교해서 업데이트
      ├ li(key=a)            ├ li(key=c)  ← 새로 추가
      └ li(key=b)            ├ li(key=a)  ← key가 같으니 재사용 (이동)
                             └ li(key=b)  ← key가 같으니 재사용 (이동)
```

### 특징

- 이전 트리와 새 트리를 비교해서 실제 DOM에 어떤 변경을 할지 결정하는 과정
- 두 트리를 완벽하게 비교하는 일반 알고리즘은 O(n³)이라 너무 느려서, React는 두 가지 가정으로 O(n)에 비교함
  1. **타입이 다르면 다른 트리로 봄.** `<div>`가 `<section>`으로 바뀌거나 컴포넌트 종류가 바뀌면 아래 트리를 통째로 버리고 새로 만듦. 이때 그 안의 state도 사라짐
  2. **리스트는 `key`로 같은 항목을 구분함.** `key`가 같으면 같은 요소로 보고 재사용함
- 배열 index를 `key`로 쓰면 항목 순서가 바뀌거나 중간에 추가될 때 다른 항목을 같은 요소로 착각해서, 입력값이나 체크 상태가 엉뚱한 항목에 남는 버그가 생길 수 있음

```jsx
// 나쁜 예: 맨 앞에 항목이 추가되면 모든 index가 밀려서 상태가 엉뚱한 줄에 붙음
todos.map((todo, index) => <TodoItem key={index} todo={todo} />);

// 좋은 예: 데이터마다 고유하고 바뀌지 않는 값 사용
todos.map((todo) => <TodoItem key={todo.id} todo={todo} />);
```

- `key`를 바꾸면 React가 다른 컴포넌트로 인식해서 state를 초기화함. 이를 이용해 프로필이 바뀔 때 폼을 초기화하는 식으로 활용할 수 있음 (`<ProfileForm key={userId} />`)

### 정리

- Reconciliation은 "무엇이 바뀌었는지 찾아내는 비교 과정"이고, React는 타입과 `key`를 기준으로 빠르게 비교함
- `key`는 성능 최적화용 값이 아니라 "이 항목이 이전의 어떤 항목인지"를 알려주는 식별자라서, 고유하고 변하지 않는 값을 써야 함

---

## 5. React Rendering

### 구조

```text
1. Trigger  : 렌더링이 필요하다고 알림 (첫 렌더링, setState)
2. Render   : 컴포넌트 함수를 호출해서 새 화면 결과(JSX)를 계산하고 이전 결과와 비교
3. Commit   : 바뀐 부분만 실제 DOM에 반영
4. 브라우저 : Reflow / Repaint / Composite 후 화면 표시
```

### 특징

- React에서 "렌더링"은 화면을 그리는 것이 아니라 **컴포넌트 함수를 다시 호출하는 것**. 렌더링이 일어나도 결과가 같으면 실제 DOM은 바뀌지 않음
- 리렌더링이 일어나는 경우
  - 자신의 state가 바뀔 때
  - **부모가 리렌더링될 때** (props가 바뀌지 않아도 자식은 기본적으로 다시 렌더링됨)
  - 구독 중인 Context 값이 바뀔 때
- `setState`에 이전과 같은 값(`Object.is`로 비교)을 넣으면 리렌더링을 건너뜀. 그래서 객체·배열 state는 직접 수정하지 말고 새 객체로 바꿔야 변경이 감지됨(1주차 불변성과 연결)
- React 18부터는 이벤트 핸들러뿐 아니라 `setTimeout`, Promise 안의 여러 `setState`도 모아서 한 번만 렌더링함(Automatic Batching)
- Render 단계는 순수 계산이라 중간에 멈추거나 다시 실행될 수 있고(Fiber, Concurrent 렌더링), Commit 단계는 한 번에 실행됨. 개발 모드의 StrictMode는 순수하지 않은 코드를 찾기 위해 렌더링을 두 번 실행함

```jsx
function Counter() {
  const [count, setCount] = useState(0);

  const handleClick = () => {
    setCount(count + 1);
    console.log(count); // 아직 이전 값 — 이번 렌더링의 count는 고정된 값 (8주차 Stale Closure와 연결)
  };
}
```

### 정리

- React 렌더링은 "함수를 다시 실행해 결과를 계산하는 것"이고, 실제 DOM 변경은 Commit 단계에서 바뀐 부분만 일어남
- 부모가 렌더링되면 자식도 렌더링된다는 점이 불필요한 리렌더링의 가장 흔한 원인

---

## 6. 딥다이브: 불필요한 리렌더링 줄이기

### 먼저 구조로 해결하기

메모이제이션을 쓰기 전에, 상태를 어디에 두는지만 바꿔도 리렌더링 범위를 줄일 수 있다.

```jsx
// 문제: input에 입력할 때마다 App 전체와 무거운 차트가 다시 렌더링됨
function App() {
  const [text, setText] = useState("");
  return (
    <>
      <input value={text} onChange={(e) => setText(e.target.value)} />
      <HeavyChart />
    </>
  );
}

// 해결 1: 상태를 쓰는 곳으로 내리기 (State Colocation)
function SearchInput() {
  const [text, setText] = useState("");
  return <input value={text} onChange={(e) => setText(e.target.value)} />;
}

function App() {
  return (
    <>
      <SearchInput />   {/* 입력해도 여기만 리렌더링 */}
      <HeavyChart />
    </>
  );
}

// 해결 2: 상태를 가진 컴포넌트가 감싸야 한다면 children으로 넘기기
function ScrollTracker({ children }) {
  const [scrollY, setScrollY] = useState(0);
  // ... 스크롤 이벤트로 scrollY 갱신
  return <div data-y={scrollY}>{children}</div>; // children은 부모(App)가 만든 것이라 다시 렌더링되지 않음
}

<ScrollTracker>
  <HeavyChart />
</ScrollTracker>;
```

- children은 `App`이 렌더링될 때 이미 만들어진 결과라서, `ScrollTracker`의 state가 바뀌어도 다시 계산되지 않는다.

### 그래도 필요하면 메모이제이션

| 도구 | 하는 일 | 쓰는 곳 |
| --- | --- | --- |
| `React.memo` | props가 얕은 비교로 같으면 컴포넌트 렌더링을 건너뜀 | 렌더링 비용이 크고 같은 props로 자주 다시 렌더링되는 컴포넌트 |
| `useMemo` | 의존성이 같으면 계산 결과를 재사용 | 무거운 계산, `memo` 컴포넌트에 넘기는 객체·배열 |
| `useCallback` | 의존성이 같으면 같은 함수를 재사용 | `memo` 컴포넌트에 넘기는 콜백, Effect 의존성에 들어가는 함수 |

```jsx
const Chart = memo(function Chart({ options, onSelect }) { /* 무거운 렌더링 */ });

function Dashboard({ data }) {
  // 매번 새 객체·새 함수를 넘기면 memo의 얕은 비교가 항상 "다름"이 되어 효과가 없음
  const options = useMemo(() => ({ data, theme: "dark" }), [data]);
  const handleSelect = useCallback((id) => console.log(id), []);

  return <Chart options={options} onSelect={handleSelect} />;
}
```

- `memo`는 props를 얕게 비교하므로, 렌더링마다 새로 만들어지는 객체·배열·함수를 넘기면 매번 다르다고 판단한다. 그래서 `memo`와 `useMemo`/`useCallback`은 함께 써야 효과가 있다.
- 메모이제이션도 비교 비용과 메모리를 쓰고 코드를 복잡하게 만든다. 모든 곳에 쓰기보다 React DevTools Profiler로 실제로 느린 컴포넌트를 찾은 뒤 적용한다.
- React Compiler를 쓰면 이런 메모이제이션을 빌드할 때 자동으로 넣어 주기 때문에, 직접 `useMemo`/`useCallback`을 쓰는 일이 줄어든다.

### 정리

- 불필요한 리렌더링은 "상태 위치 조정 → children 패턴 → 측정 후 메모이제이션" 순서로 줄인다.
- 리렌더링 자체가 문제가 아니라, 비싼 리렌더링이 자주 일어날 때가 문제다. 측정 없이 미리 최적화하지 않는다.
