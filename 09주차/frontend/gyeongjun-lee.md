# 9주차 - 비동기 / Event Loop

## 1. Call Stack

### 구조

```js
function c() { console.log("c"); }
function b() { c(); }
function a() { b(); }
a();
```

```text
         ┌─────┐
         │ c() │  ← 맨 위만 실행
   ┌─────┼─────┤
   │     │ b() │
   │     ├─────┤
   │     │ a() │
   │     ├─────┤
   │     │ 전역 │
   └─────┴─────┘
c 끝 → b 끝 → a 끝 순서로 빠짐 (LIFO)
```

### 특징

- 함수 호출 시 실행 컨텍스트가 쌓이고, 실행이 끝나면 빠지는 스택 구조(8주차 Execution Context와 연결)
- JavaScript 엔진은 Call Stack이 하나라서 한 번에 하나의 작업만 실행함(2주차 Single Thread)
- 오래 걸리는 동기 작업이 Call Stack을 차지하고 있으면 클릭 처리, 렌더링 같은 다른 작업이 모두 멈춤
- 재귀가 끝나지 않으면 스택이 넘쳐 `RangeError: Maximum call stack size exceeded`(Stack Overflow) 발생

### 정리

- Call Stack은 "지금 무엇을 실행 중인지"를 기록하는 곳이고, 하나뿐이라서 동기 코드가 길어지면 화면 전체가 멈춤
- 그래서 오래 기다려야 하는 작업은 Call Stack 밖으로 보내야 함 → Web API와 Event Loop가 필요한 이유

---

## 2. Web API

### 구조

```text
JavaScript 엔진 (V8)            브라우저가 제공하는 기능 (Web API)
┌───────────┐                   ┌─────────────────────────┐
│ Call Stack│ ── setTimeout ──> │ 타이머                   │
│ Heap      │ ── fetch ───────> │ 네트워크 요청             │
└───────────┘ ── addEventListener│ DOM 이벤트               │
                                └───────────┬─────────────┘
                                            │ 완료되면 콜백을
                                            ▼ 큐에 넣음
                                     Task Queue / Microtask Queue
```

### 특징

- `setTimeout`, `fetch`, DOM 이벤트 등은 JavaScript 언어 자체가 아니라 브라우저가 제공하는 기능
- JavaScript는 작업을 Web API에 맡기고 바로 다음 코드를 실행함. 실제 대기(타이머, 네트워크)는 브라우저의 다른 스레드가 처리
- 작업이 끝나면 등록해둔 콜백이 큐에 들어가고, 실행은 Event Loop가 Call Stack이 비었을 때 넣어줌
- Node.js에서는 Web API 대신 libuv가 비슷한 역할을 함

### 정리

- JavaScript는 싱글 스레드지만, 기다리는 일은 브라우저에 맡기기 때문에 여러 작업을 동시에 하는 것처럼 동작함
- Web API는 "기다리는 일을 대신 해주고, 끝나면 콜백을 큐에 넣어주는 곳"

---

## 3. Event Loop

### 구조

```text
┌────────────┐
│ Call Stack │ ← 비었는지 계속 확인
└─────▲──────┘
      │
 Event Loop
      │ 1. Task Queue에서 작업 1개 실행
      │ 2. Microtask Queue를 전부 비움
      │ 3. (필요하면) 화면 렌더링
      │ 4. 다시 1번으로
┌─────┴──────────────┐   ┌──────────────────┐
│ Microtask Queue    │   │ Task Queue        │
│ Promise.then, await│   │ setTimeout, 이벤트 │
└────────────────────┘   └──────────────────┘
```

### 특징

- Call Stack이 비면 큐에 있던 콜백을 꺼내 Call Stack으로 옮겨 실행하는 반복 구조
- 콜백은 Call Stack이 완전히 비어야 실행됨. 그래서 `setTimeout(fn, 0)`도 즉시 실행이 아니라 지금 실행 중인 코드가 모두 끝난 뒤 실행됨
- `setTimeout`의 시간은 "그 시간 후에 실행"이 아니라 "최소 그 시간 후에 큐에 들어감"이라는 뜻. 앞의 작업이 길면 더 늦어짐
- 브라우저는 보통 Task 사이에 렌더링 기회를 가짐. 하나의 작업이 오래 걸리면 렌더링도 그만큼 밀려 화면이 멈춘 것처럼 보임(10주차 Rendering과 연결)

### 정리

- Event Loop는 "Call Stack이 비었을 때 다음에 실행할 콜백을 큐에서 꺼내주는 관리자"
- 비동기 콜백은 언제 실행될지 정확히 보장되지 않고, 지금 실행 중인 동기 코드가 다 끝나야 차례가 옴

---

## 4. Microtask / Macrotask

### 구조

```js
console.log(1);
setTimeout(() => console.log(2));             // Macrotask (Task)
Promise.resolve().then(() => console.log(3)); // Microtask
console.log(4);

// 출력: 1 → 4 → 3 → 2
```

```text
1. 동기 코드 실행: 1, 4
2. Call Stack이 빔 → Microtask Queue 전부 실행: 3
3. 다음 Task 실행: 2
```

| 구분 | 예시 | 실행 시점 |
| --- | --- | --- |
| Microtask | `Promise.then/catch/finally`, `await` 이후 코드, `queueMicrotask`, `MutationObserver` | 현재 작업이 끝난 직후, 큐가 빌 때까지 전부 |
| Macrotask (Task) | `setTimeout`, `setInterval`, 이벤트 콜백, `MessageChannel` | Event Loop 한 바퀴에 하나씩 |

### 특징

- Microtask가 Macrotask보다 우선. Task 하나가 끝날 때마다 Microtask Queue를 완전히 비운 뒤 다음으로 넘어감
- Microtask 실행 중에 새 Microtask가 추가되면 그것까지 이어서 실행함. 그래서 Microtask를 계속 만들면 Task와 렌더링이 영원히 밀려 화면이 멈출 수 있음

```js
function loop() {
  Promise.resolve().then(loop); // 렌더링 기회가 오지 않아 화면이 멈춤
}
```

### 정리

- 실행 순서는 "동기 코드 → Microtask 전부 → Macrotask 하나 → Microtask 전부 → …"로 기억
- Promise 콜백이 `setTimeout`보다 먼저 실행되는 이유는 Microtask Queue가 먼저 비워지기 때문

---

## 5. Promise

### 구조

```text
            resolve(value)
pending ───────────────────> fulfilled → .then 실행
   │
   └──────── reject(error) ─> rejected  → .catch 실행

한 번 fulfilled/rejected가 되면(settled) 상태가 다시 바뀌지 않음
```

```js
fetch("/api/user")
  .then((res) => res.json())    // then은 새 Promise를 반환 → 체이닝 가능
  .then((user) => console.log(user))
  .catch((err) => console.error(err)) // 앞 단계 어디서 실패해도 여기로 옴
  .finally(() => setLoading(false));
```

### 특징

- 비동기 작업의 미래 결과를 나타내는 객체. 콜백을 중첩하던 콜백 지옥을 체이닝으로 평평하게 만듦
- `new Promise((resolve) => { ... })`의 executor 함수는 **동기적으로 즉시 실행**됨. 비동기로 실행되는 것은 `then`에 넘긴 콜백
- `then` 콜백은 이미 fulfilled된 Promise라도 바로 실행되지 않고 Microtask로 실행됨
- 여러 Promise를 다루는 정적 메서드

| 메서드 | 동작 |
| --- | --- |
| `Promise.all` | 모두 성공하면 결과 배열, 하나라도 실패하면 즉시 실패 |
| `Promise.allSettled` | 성공·실패 상관없이 모두 끝날 때까지 기다려 각각의 결과 반환 |
| `Promise.race` | 가장 먼저 끝난(성공 또는 실패) 결과 하나 |
| `Promise.any` | 가장 먼저 성공한 결과 하나, 모두 실패하면 실패 |

### 정리

- Promise는 "나중에 성공 또는 실패로 한 번만 결정되는 값"을 담는 상자
- 체이닝 끝에 `catch`를 두지 않으면 실패가 처리되지 않고 `unhandledrejection`으로 남음

---

## 6. async / await

### 구조

```js
async function load() {
  console.log("A");
  await null;            // 여기서 함수 실행을 멈추고 나머지를 Microtask로 넘김
  console.log("B");
}

console.log("start");
load();
console.log("end");

// 출력: start → A → end → B
```

### 특징

- Promise를 동기 코드처럼 위에서 아래로 읽히게 쓰는 문법. `async` 함수는 항상 Promise를 반환함
- `await`를 만나면 함수가 일시 정지하고 Call Stack에서 빠짐. 스레드를 막는 것이 아니라 다른 코드가 실행될 수 있음
- `await` 이후 코드는 Promise가 settled된 뒤 Microtask로 이어서 실행됨
- 에러 처리는 `try/catch`로 함

```js
// 순차 실행: 서로 관계없는 요청인데 앞 요청이 끝날 때까지 기다림 (느림)
const user = await fetchUser();
const posts = await fetchPosts();

// 병렬 실행: 동시에 보내고 둘 다 끝날 때까지 기다림
const [user2, posts2] = await Promise.all([fetchUser(), fetchPosts()]);
```

- `forEach`는 콜백이 반환한 Promise를 기다리지 않으므로, 순서대로 기다려야 하면 `for...of` + `await`, 병렬이면 `Promise.all` + `map`을 사용

### 정리

- `async/await`는 Promise를 대체하는 것이 아니라 Promise를 더 읽기 쉽게 쓰는 문법
- 서로 의존하지 않는 요청을 `await`로 하나씩 기다리면 불필요하게 느려지므로 `Promise.all`로 묶음

---

## 7. 딥다이브: 비동기 요청의 경쟁 상태 (Race Condition)

### 왜 검색 결과가 꼬일까?

검색창에 `r` → `re` → `rea`를 빠르게 입력하면 요청이 세 번 나간다. 응답은 보낸 순서대로 오지 않을 수 있어서, `rea` 결과가 먼저 오고 `r` 결과가 늦게 오면 마지막에 도착한 `r` 결과가 화면을 덮어쓴다. 2주차에서 본 Race Condition이 싱글 스레드인 JavaScript에서도 비동기 작업 사이에서 생기는 것이다.

```jsx
function Search({ query }) {
  const [results, setResults] = useState([]);

  useEffect(() => {
    fetch(`/api/search?q=${encodeURIComponent(query)}`)
      .then((res) => res.json())
      .then(setResults); // 늦게 도착한 예전 응답이 최신 결과를 덮어쓸 수 있음
  }, [query]);
}
```

```text
요청: q=r ─────────────────────────────> 응답(r)   ← 마지막에 도착해서 화면에 남음
요청: q=rea ──────────> 응답(rea)
                        화면: rea → r  (입력은 rea인데 r 결과가 보임)
```

### 해결 1: 이전 요청 결과 무시하기

Effect가 다시 실행되기 전에 cleanup이 실행되는 점을 이용한다. 8주차의 클로저 덕분에 각 요청의 `ignore` 변수는 자기 렌더링의 것을 따로 기억한다.

```jsx
useEffect(() => {
  let ignore = false;

  fetch(`/api/search?q=${encodeURIComponent(query)}`)
    .then((res) => res.json())
    .then((data) => {
      if (!ignore) setResults(data); // 최신 요청일 때만 반영
    });

  return () => {
    ignore = true; // query가 바뀌면 이전 요청은 무시 표시
  };
}, [query]);
```

### 해결 2: AbortController로 요청 자체를 취소하기

```jsx
useEffect(() => {
  const controller = new AbortController();

  fetch(`/api/search?q=${encodeURIComponent(query)}`, { signal: controller.signal })
    .then((res) => res.json())
    .then(setResults)
    .catch((err) => {
      if (err.name !== "AbortError") setError(err); // 취소는 에러로 처리하지 않음
    });

  return () => controller.abort(); // 이전 요청 취소
}, [query]);
```

- `ignore` 방식은 응답을 받은 뒤 버리는 것이고, `AbortController`는 브라우저가 요청 자체를 중단해서 불필요한 네트워크 사용도 줄인다.
- 취소된 요청은 `AbortError`로 reject되므로 일반 에러와 구분해야 한다.
- 입력마다 요청을 보내지 않도록 debounce를 함께 쓰면 요청 수 자체를 줄일 수 있다(13주차 Debounce/Throttle과 연결).
- React Query 같은 서버 상태 라이브러리는 Query Key별로 결과를 관리하고 취소 signal도 제공해서 이런 처리를 대신 해준다(6주차 Server State와 연결).

### 정리

- 싱글 스레드라도 비동기 응답의 도착 순서는 보장되지 않기 때문에 Race Condition이 생길 수 있다.
- "마지막으로 보낸 요청의 결과만 화면에 반영한다"는 원칙을 cleanup(`ignore` 플래그 또는 `AbortController`)으로 지킨다.

---

## 8. 딥다이브: Long Task와 메인 스레드 양보

### 왜 클릭했는데 반응이 늦을까?

브라우저의 메인 스레드는 JavaScript 실행과 렌더링, 사용자 입력 처리를 모두 맡는다. Event Loop는 Task 하나를 끝까지 실행한 뒤에야 렌더링이나 다음 입력을 처리하므로, 한 Task가 오래 걸리면 그동안 클릭해도 화면이 반응하지 않는다. 50ms 이상 걸리는 Task를 **Long Task**라고 부르고, 이런 지연은 사용자 입력 반응성을 측정하는 Web Vitals 지표 INP(Interaction to Next Paint)를 나쁘게 만든다(13주차 Web Vitals와 연결).

```text
Long Task 하나 (300ms)
메인 스레드: [■■■■■■■■■■■■■■■■■■■■■■■■■■■■■] [입력 처리] [렌더링]
사용자 클릭:        ↑ 여기서 클릭해도 300ms 뒤에야 반응

작업을 나눠서 중간에 양보
메인 스레드: [■■■■] [입력 처리] [렌더링] [■■■■] [■■■■] ...
사용자 클릭:   ↑ 다음 조각 사이에서 바로 반응
```

### 작업을 잘게 나누고 중간에 양보하기

```js
function yieldToMain() {
  if (globalThis.scheduler?.yield) {
    return scheduler.yield(); // 지원 브라우저: 양보 후 이어서 실행할 작업을 우선 처리
  }
  return new Promise((resolve) => setTimeout(resolve, 0)); // 미지원 브라우저: 새 Task로 넘김
}

async function processItems(items) {
  let lastYield = performance.now();

  for (const item of items) {
    process(item);

    if (performance.now() - lastYield > 50) { // 50ms마다 한 번씩 양보
      await yieldToMain();
      lastYield = performance.now();
    }
  }
}
```

- `await yieldToMain()`에서 함수가 멈추고 남은 작업이 새 Task로 넘어가므로, 그 사이에 브라우저가 입력 처리와 렌더링을 할 수 있다.
- `await Promise.resolve()`로는 양보가 되지 않는다. Microtask는 현재 Task가 끝나기 전에 전부 실행되기 때문에 렌더링 기회가 생기지 않는다(4번 Microtask/Macrotask와 연결).
- `setTimeout`으로 양보하면 남은 작업이 큐의 맨 뒤로 가서 다른 Task보다 늦어질 수 있다. `scheduler.yield()`는 이어서 실행할 작업을 우선 처리해 주지만, 아직 모든 브라우저가 지원하지는 않아 위처럼 대체 코드를 둔다.

### 상황별로 쓰는 API

| API | 실행 시점 | 쓰는 곳 |
| --- | --- | --- |
| `setTimeout(fn, 0)` | 다음 Task | 긴 작업을 나눌 때 (대체 수단) |
| `scheduler.yield()` | 양보 후 우선적으로 이어서 실행 | 긴 작업을 나눌 때 |
| `requestAnimationFrame(fn)` | 다음 화면 그리기 직전 | 애니메이션, DOM 위치·크기 변경 |
| `requestIdleCallback(fn)` | 브라우저가 한가할 때 | 로그 전송, 미리 불러오기처럼 급하지 않은 작업 |
| Web Worker | 메인 스레드가 아닌 별도 스레드 | 이미지 처리, 대용량 계산처럼 DOM이 필요 없는 무거운 작업 |

- 애니메이션을 `setTimeout`으로 돌리면 화면 갱신 주기와 맞지 않아 끊겨 보일 수 있어서 `requestAnimationFrame`을 쓴다.
- 나눠도 계산 자체가 무거우면 메인 스레드에서 빼서 Web Worker로 옮기는 것이 근본적인 해결이다(2주차 Web Worker와 연결).

### 정리

- 싱글 스레드에서 반응성을 지키는 핵심은 "한 번에 오래 붙잡지 않는 것"이다. 긴 작업은 나누고, 조각 사이에 Task 경계를 만들어 브라우저에 차례를 넘긴다.
- Microtask로는 양보가 되지 않는다는 점이 Event Loop 실행 순서를 이해했는지 확인하는 좋은 꼬리질문이다.
