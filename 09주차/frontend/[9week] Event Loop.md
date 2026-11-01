# 9주차 — 비동기 / Event Loop

> **키워드** Call Stack · Web API · Event Loop · Promise · async/await · Microtask/Macrotask

---

## 0. 개요

JS는 싱글 스레드인데 `fetch` 응답을 기다리는 동안에도 클릭이 되고 화면이 그려짐. JS 엔진 혼자가 아니라 **엔진 + 브라우저(호스트)**가 일을 나눠 맡기 때문임.

| 묶음 | 키워드 | 한 줄 요약 |
|---|---|---|
| 실행 | Call Stack | JS 코드가 실제로 실행되는 곳. 하나뿐이라 한 번에 한 작업만 함 |
| 위임 | Web API | 기다리는 일(타이머, 네트워크, 이벤트 감지)은 브라우저가 대신 처리함 |
| 대기·순서 | Microtask/Macrotask, Event Loop | 끝난 일의 콜백은 큐에서 대기하고, 이벤트 루프가 스택이 비었을 때 하나씩 올림 |
| 문법 | Promise, async/await | 콜백 지옥 없이 비동기 결과를 다루는 방법 |

**전체 구조**
```text
 +------------------ JS Engine (V8) ------------------+
 |  Memory Heap            Call Stack                 |
 |  (objects)              +------------------+       |
 |                         | handleClick()    |       |
 |                         | (global)         |       |
 |                         +------------------+       |
 +--------|-------------------------------^-----------+
          |                               |
          | (1) call Web API              | (4) push next callback
          v                               |     (only when stack is empty)
 +-- Web APIs (Browser) --+     +------- Event Loop --------+
 | timer, network, events |     | (3) stack empty?          |
 +-----------|------------+     |     microtasks: ALL first |
             |                  |     tasks: ONE at a time  |
             | (2) done         +---------------------^-----+
             v                                        |
 +------------------------- Queues -------------------|-----+
 |  Microtask Queue : [ then cb, await resume ]  -----+     |
 |  Task Queue      : [ timeout cb, click cb ]  ------+     |
 +----------------------------------------------------------+
```
1. 콜 스택에서 `setTimeout`, `fetch` 같은 Web API를 호출하면, JS 엔진은 요청만 넘기고 바로 다음 줄로 진행함
2. 브라우저가 기다리는 일(타이머 카운트, 응답 대기)을 끝내면 콜백을 큐에 넣음
3. 이벤트 루프가 콜 스택이 비었는지 확인함. 비었으면 마이크로태스크를 **전부**, 그다음 태스크를 **하나** 꺼냄
4. 꺼낸 콜백을 콜 스택에 올려 실행함

연결 고리
- 콜 스택이 하나 → 오래 걸리는 동기 코드는 전부를 멈춤 → 그래서 기다리는 일은 **Web API에 위임**
- 위임한 일의 결과(콜백)는 **큐**에서 순서를 기다림 → 큐가 두 종류라 **실행 순서 규칙**이 생김
- 콜백을 직접 넘기는 방식은 중첩이 깊어짐 → **Promise**로 결과를 객체로 받고, **async/await**로 동기 코드처럼 씀
- Promise의 `then`, `await` 이후 코드는 **마이크로태스크** → `setTimeout`보다 먼저 실행됨

---

## 1. Call Stack

**정의**: 지금 실행 중인 함수(실행 컨텍스트)를 쌓아두는 LIFO 스택. 함수를 호출하면 push, return하면 pop함

- JS 엔진의 콜 스택은 **하나** → 한 번에 한 함수만 실행함. "JS는 싱글 스레드"의 의미
- 스택에 쌓이는 단위가 8주차의 **실행 컨텍스트**임

```js
function multiply(a, b) { return a * b; }
function square(n) { return multiply(n, n); }
function printSquare(n) { console.log(square(n)); }

printSquare(4);
```
```text
      (1)              (2)              (3)              (4)              (5)
                                  +-------------+
                 +-------------+  | multiply    |  +-------------+
+-------------+  | square      |  | square      |  | console.log |
| printSquare |  | printSquare |  | printSquare |  | printSquare |
| (global)    |  | (global)    |  | (global)    |  | (global)    |    (empty)
+-------------+  +-------------+  +-------------+  +-------------+
```
1. `printSquare(4)` 호출 → push
2. 안에서 `square(4)` 호출 → push
3. 안에서 `multiply(4, 4)` 호출 → push. 가장 위에 있는 함수만 실행 중임
4. `multiply`가 16을 반환하고 pop → `square`도 16을 반환하고 pop → `console.log(16)` push
5. 전부 반환하고 pop → 스택이 빔. 이때부터 이벤트 루프가 큐의 콜백을 올릴 수 있음

**스택을 오래 점유하면: Blocking**
```js
button.addEventListener('click', () => {
  const start = Date.now();
  while (Date.now() - start < 3000) {} // 3초 동안 스택을 비우지 않음
});
```
- 3초 동안 다른 클릭·스크롤·화면 갱신이 전부 멈춤 → 렌더링도 같은 메인 스레드에서 돌기 때문
- 재귀가 끝나지 않으면 스택이 넘침 → `RangeError: Maximum call stack size exceeded` (Stack Overflow)

**헷갈리는 포인트: "JS는 싱글 스레드"의 정확한 의미**
- JS 코드를 실행하는 **메인 스레드(콜 스택)가 하나**라는 뜻
- 브라우저 전체는 멀티 스레드·멀티 프로세스임 (네트워크, 타이머, 렌더러, GPU 등 — 2주차 브라우저 Multi-Process 연결)
- 그래서 "기다리는 일"은 다른 스레드에 맡기고, 메인 스레드는 결과(콜백)만 받아서 실행함
- Web Worker를 쓰면 JS도 별도 스레드에서 실행 가능. 단, DOM 접근 불가이고 메시지로만 통신함

---

## 2. Web API

**정의**: 브라우저가 JS에 제공하는 기능. ECMAScript(언어) 스펙이 아니라 **브라우저(호스트)**가 구현함

| Web API | 브라우저가 대신 하는 일 | 끝나면 콜백이 가는 곳 |
|---|---|---|
| `setTimeout` / `setInterval` | 타이머 카운트 | Task Queue |
| `addEventListener` | 클릭·키 입력 감지 | Task Queue |
| `fetch` | 네트워크 요청·응답 대기 | 응답 도착 → Promise resolve → `then` 콜백은 Microtask Queue |
| `MutationObserver` | DOM 변경 감지 | Microtask Queue |
| `requestAnimationFrame` | 다음 화면 갱신까지 대기 | 렌더링 직전에 실행 (큐와 별도) |

- `setTimeout`, `fetch`, `console`도 JS 문법이 아님 → Node.js에서는 같은 이름의 API를 Node(libuv)가 따로 구현함
- JS 엔진은 "요청"만 하고 즉시 다음 줄로 넘어감 → 이게 **Non-blocking**

**헷갈리는 포인트: `setTimeout(fn, 0)`은 "0ms 뒤 실행"이 아님**
- "최소 0ms 뒤에 **Task Queue에 넣음**"이라는 뜻
- 실제 실행은 ① 현재 동기 코드가 끝나 스택이 비고 ② 마이크로태스크가 전부 처리되고 ③ 앞에 줄 선 태스크가 끝난 뒤
- 그래서 delay는 "보장 시간"이 아니라 "최소 대기 시간"
- 추가 지연: 5단계 넘게 중첩된 타이머는 최소 4ms로 조정되고, 백그라운드 탭에선 1초 이상으로 늦춰지기도 함 (브라우저 정책)

---

## 3. Event Loop

**정의**: 콜 스택이 비었는지 확인하고, 큐에서 다음 할 일을 꺼내 스택에 올리는 무한 반복 루프

- V8 같은 JS 엔진이 아니라 **호스트**가 구현함 (브라우저는 HTML 스펙, Node.js는 libuv)

**루프 1회에 하는 일 (브라우저)**
```text
 while (true) {
   (1) Task Queue      -> take ONE task, run to completion
   (2) Microtask Queue -> run ALL (including ones added during this step)
   (3) render?         -> only when a new frame is due (~16.7ms at 60Hz)
                          rAF callbacks -> Style -> Layout -> Paint
 }
```
1. 태스크 큐에서 **하나만** 꺼내 끝까지 실행함 (맨 처음 태스크는 `<script>` 전체 실행)
2. 마이크로태스크 큐는 **빌 때까지 전부** 실행함. 실행 도중 새로 추가된 마이크로태스크도 이번에 같이 처리함
3. 화면을 갱신할 시점이면 rAF 콜백 → 스타일 계산 → 레이아웃 → 페인트 (10주차 Rendering 연결)
4. 다시 1번으로

**Run-to-completion**: 한 번 시작한 작업은 끝날 때까지 다른 콜백이 끼어들지 못함
- 장점: 실행 중에 변수가 갑자기 바뀌는 일이 없음 → 멀티 스레드처럼 락이 필요 없음
- 단점: 작업 하나가 길면 그동안 전부 멈춤

**렌더링과의 관계**
```text
 main thread
 |-task A-|-micro-|-render-|------------- task B (120ms) -------------|-micro-|-render-|
 0ms               16.7ms   (frames due at 33ms, 50ms ... are skipped)
```
- 렌더링도 같은 메인 스레드에서 실행됨 → 태스크 하나가 길면 그동안 프레임을 못 그림 (버벅임)
- 50ms 넘는 태스크를 **Long Task**라고 함. 클릭해도 반응이 늦는 원인 (Web Vitals의 INP 지표 저하 — 13주차 연결)

---

## 4. Microtask / Macrotask

| 구분 | 예시 | 처리 시점 | 한 번에 |
|---|---|---|---|
| **Macrotask (Task)** | `<script>` 실행, `setTimeout`/`setInterval`, 클릭 등 UI 이벤트, `MessageChannel` | 루프 1회마다 | **1개** |
| **Microtask** | `Promise.then/catch/finally`, `await` 이후 코드, `queueMicrotask`, `MutationObserver` | 현재 작업이 끝나 스택이 빌 때마다 | 큐가 빌 때까지 **전부** |
| (참고) rAF | `requestAnimationFrame` | 렌더링 직전 | 그 프레임에 등록된 것 전부 |

**우선순위**
```text
  sync code      >   Microtask Queue   >   (render if due)   >   Task Queue
  (Call Stack)       drain completely                            one per loop
```

**출력 순서 추적**
```js
console.log('1');
setTimeout(() => console.log('2'), 0);
Promise.resolve().then(() => console.log('3'));
queueMicrotask(() => console.log('4'));
console.log('5');
// 1 5 3 4 2
```
| 단계 | Call Stack | Microtask Q | Task Q | 출력 |
|---|---|---|---|---|
| 1. `log('1')` | global | | | 1 |
| 2. `setTimeout` 등록 → 0ms 뒤 Task Q로 | global | | cb2 | |
| 3. `then` 등록 (이미 resolve 상태라 바로 Microtask Q로) | global | cb3 | cb2 | |
| 4. `queueMicrotask` | global | cb3, cb4 | cb2 | |
| 5. `log('5')` | global | cb3, cb4 | cb2 | 5 |
| 6. 스크립트 종료 → 스택이 빔 → 마이크로태스크 전부 | cb3 → cb4 | | cb2 | 3, 4 |
| 7. 다음 루프: 태스크 하나 | cb2 | | | 2 |

**헷갈리는 포인트 1: 마이크로태스크가 마이크로태스크를 계속 만들면**
```js
function loopA() { queueMicrotask(loopA); } // 화면이 멈춤
function loopB() { setTimeout(loopB, 0); }  // 화면은 안 멈춤
```
- 마이크로태스크는 "큐가 빌 때까지" 처리함 → 계속 추가하면 영원히 안 비어서 렌더링·클릭 처리가 불가능해짐 (Starvation)
- `setTimeout`은 루프마다 하나씩 처리함 → 사이사이 렌더링 기회가 있음

**헷갈리는 포인트 2: "Macrotask"는 스펙 용어가 아님**
- HTML 스펙에선 그냥 **task**라고 부름. Microtask와 구분하려고 관습적으로 macrotask라고 부르는 것
- 태스크 큐는 실제로 여러 개(타이머, 사용자 입력, 네트워크 등)이고, 어느 큐를 먼저 볼지는 브라우저가 정함 (사용자 입력을 우선하는 식) → "`setTimeout` 콜백 vs 클릭 콜백" 순서는 보장되지 않음
- Node.js는 구조가 다름: 단계(phase)별 큐(timers → poll → check …)가 있고, `process.nextTick`이 Promise 마이크로태스크보다도 먼저 실행됨

**헷갈리는 포인트 3: 마이크로태스크는 "태스크가 끝날 때"가 아니라 "스택이 빌 때마다" 처리됨**
- 버튼에 리스너 2개가 있고, 각 리스너가 `then`으로 로그를 찍는 경우
  - 사용자가 직접 클릭: 리스너1 → (스택 빔) → 리스너1의 마이크로태스크 → 리스너2 → 리스너2의 마이크로태스크
  - 코드에서 `button.click()` 호출: `click()`을 부른 스크립트가 스택에 남아 있음 → 리스너1 → 리스너2 → 마이크로태스크 2개
- 테스트 코드에서 `click()`으로 재현한 순서가 실제 사용자 클릭과 다를 수 있는 이유

---

## 5. Promise

**정의**: 비동기 작업의 **미래 결과**를 담는 객체. 콜백을 인자로 넘기는 대신, 결과 객체를 받아 나중에 `then`으로 이어 붙임

**왜 필요한가: 콜백 지옥**
```js
// 콜백 방식: 중첩이 깊어지고 에러 처리를 단계마다 따로 해야 함
getUser(id, (user) => {
  getPosts(user.id, (posts) => {
    getComments(posts[0].id, (comments) => {
      render(comments);
    }, onError);
  }, onError);
}, onError);

// Promise 체이닝: 평평해지고 에러는 catch 하나로 처리
getUser(id)
  .then((user) => getPosts(user.id))
  .then((posts) => getComments(posts[0].id))
  .then((comments) => render(comments))
  .catch(onError);
```
- 콜백 방식의 또 다른 문제는 **제어의 역전**: 콜백을 남의 함수에 넘기면 몇 번 호출될지, 에러를 어떻게 넘길지가 그쪽 구현에 달림. Promise는 결과가 **딱 한 번**만 정해지는 걸 보장함

**3가지 상태**
```text
                  resolve(value)
             +----------------------> fulfilled ---> .then(onFulfilled)
             |
  pending ---+
             |    reject(err) / throw
             +----------------------> rejected  ---> .catch(onRejected)

  fulfilled or rejected = settled (once settled, it never changes)
```
- `pending`: 아직 결과 없음 / `fulfilled`: 성공, 값 있음 / `rejected`: 실패, 에러 있음
- 한 번 settled 되면 다시 바뀌지 않음 → `resolve`를 두 번 불러도 첫 번째만 유효함

**헷갈리는 포인트: Promise 안의 코드는 동기로 실행됨**
```js
console.log('A');
new Promise((resolve) => {
  console.log('B');                 // executor: 즉시, 동기로 실행
  resolve();
}).then(() => console.log('D'));    // then 콜백만 마이크로태스크
console.log('C');
// A B C D
```
- Promise가 코드를 비동기로 "만들어 주는" 게 아님. 비동기인 건 **`then` 콜백의 실행 시점**뿐
- 그래서 executor 안에 무거운 계산을 넣으면 그대로 메인 스레드를 막음

**체이닝 규칙**: `then`은 항상 **새 Promise**를 반환함

| `then` 콜백에서 | 다음 `then`이 받는 것 |
|---|---|
| 값을 return | 그 값 |
| Promise를 return | 그 Promise가 settled 된 결과 (끝날 때까지 기다림) |
| throw | 건너뛰고 다음 `catch`로 이동 |
| return 안 함 | `undefined` — 흔한 실수: `then(() => { fetchX(); })`는 `fetchX`를 기다리지 않음 |

**조합 메서드**

| 메서드 | 완료 조건 | 하나가 실패하면 | 쓰는 곳 |
|---|---|---|---|
| `Promise.all` | 전부 fulfilled | 즉시 reject (나머지 결과는 버림) | 화면에 전부 필요한 데이터를 동시에 요청 |
| `Promise.allSettled` | 전부 settled | 계속 기다림. `{ status, value / reason }` 배열 반환 | 일부가 실패해도 되는 위젯 여러 개 |
| `Promise.race` | 처음 settled 된 것 | 그게 실패면 reject | 타임아웃 구현 |
| `Promise.any` | 처음 fulfilled 된 것 | 전부 실패해야 `AggregateError` | 여러 서버 중 가장 빠른 응답 사용 |

- `Promise.all`에서 하나가 실패해도 나머지 요청이 **취소되지는 않음**. 결과만 무시될 뿐이고, 진짜 취소는 `AbortController`가 필요함

---

## 6. async/await

**정의**: Promise를 동기 코드처럼 위에서 아래로 읽히게 쓰는 문법 (ES2017). 새로운 기능이 아니라 Promise 위에 얹은 문법임

- `async` 함수는 **항상 Promise를 반환**함 (return 값은 resolve, throw는 reject)
- `await`는 Promise가 settled 될 때까지 **그 async 함수만** 멈추고, 콜 스택에서 빠져나와 호출한 쪽에 제어권을 돌려줌
- 멈춘 지점 이후의 코드는 나중에 **마이크로태스크로 재개**됨

```js
async function f() {
  console.log('B');
  await null;          // 여기서 f는 스택에서 빠짐
  console.log('D');    // 마이크로태스크로 재개
}
console.log('A');
f();
console.log('C');
// A B C D
```
```text
 Call Stack   global  ->  f  ->  global  ->  (empty)  ->  f (resumed)
 output       A           B      C                        D
 Microtask                   [resume f] ----------------> run
```
1. `A` 출력
2. `f()` 호출. `await` 전까지는 **동기**로 실행 → `B` 출력
3. `await`를 만남 → 나머지 코드("D 출력")를 마이크로태스크로 예약하고 `f`는 스택에서 빠짐
4. 호출한 쪽으로 돌아와 `C` 출력
5. 스크립트가 끝나 스택이 빔 → 마이크로태스크로 `f` 재개 → `D` 출력

**await는 then의 문법 설탕**
```js
// 두 코드는 같은 동작
async function load() {
  const user = await getUser();
  console.log(user.name);
}

function load() {
  return getUser().then((user) => {
    console.log(user.name);
  });
}
```

**에러 처리**: `try/catch`로 동기 코드처럼 처리함
```js
async function save(body) {
  try {
    const res = await fetch('/api/posts', { method: 'POST', body });
    if (!res.ok) throw new Error(`HTTP ${res.status}`);
    return await res.json();
  } catch (e) {
    showToast('저장 실패');
  }
}
```
- `fetch`는 **네트워크 실패만** reject함. 404·500은 정상 응답으로 resolve → `res.ok`를 직접 확인해야 함 (axios는 2xx 외엔 reject — 5주차 Fetch/Axios 연결)
- `await`를 안 붙인 Promise의 reject는 `try/catch`로 못 잡음 → 처리 안 된 reject는 `unhandledrejection` 이벤트로 감

**헷갈리는 포인트 1: 순차 vs 병렬**
```js
// 순차: 서로 의존하지 않는데 하나씩 기다림
const user = await getUser();
const posts = await getPosts();

// 병렬: 동시에 시작하고 둘 다 기다림
const [user, posts] = await Promise.all([getUser(), getPosts()]);
```
```text
 sequential   |---- getUser 1s ----|---- getPosts 1s ----|   total 2s
 parallel     |---- getUser 1s ----|
              |---- getPosts 1s ---|                         total 1s
```
- 앞 결과가 다음 요청에 필요할 때만 순차로 씀

**헷갈리는 포인트 2: `forEach` 안의 `await`는 기다리지 않음**
```js
// X: forEach는 콜백이 반환한 Promise를 무시함 -> '완료'가 먼저 찍힘
ids.forEach(async (id) => { await deletePost(id); });
console.log('완료');

// O: 순차로 하나씩
for (const id of ids) { await deletePost(id); }

// O: 병렬로 한 번에
await Promise.all(ids.map((id) => deletePost(id)));
```

**헷갈리는 포인트 3: `await`는 스레드를 막지 않음**
- 멈추는 건 **그 async 함수 하나**뿐이고, 메인 스레드는 그동안 다른 태스크·렌더링을 처리함
- 반면 `while`로 기다리는 건 진짜 블로킹 → 화면 전체가 멈춤

---

## 7. 실무 연결 (FE)

### 7-1. useEffect에서 비동기 처리 + Race Condition

```js
// X: useEffect 콜백은 cleanup 함수(또는 undefined)를 반환해야 하는데, async 함수는 Promise를 반환함
useEffect(async () => { /* ... */ }, []);

// O: 안에서 async 함수를 만들어 호출하고, cleanup에서 이전 요청 취소
useEffect(() => {
  const controller = new AbortController();

  (async () => {
    try {
      const res = await fetch(`/api/search?q=${query}`, { signal: controller.signal });
      setResults(await res.json());
    } catch (e) {
      if (e.name !== 'AbortError') setError(e);
    }
  })();

  return () => controller.abort(); // query가 바뀌거나 언마운트되면 이전 요청 취소
}, [query]);
```

**Race Condition**: 검색어를 `a` → `ab`로 빠르게 바꾸면 요청 2개가 동시에 나가고, 응답 순서는 보장되지 않음
```text
 time -->
 req "a"   |--------------------- slow ---------------------| setResults(a)   <- overwrites!
 req "ab"     |--------- fast ---------| setResults(ab)
```
- 늦게 도착한 `a`의 결과가 최신인 `ab`의 결과를 덮어씀
- cleanup에서 `abort()`하면 `a` 요청이 취소돼서 덮어쓰지 않음
- TanStack Query는 queryKey별로 이 문제를 대신 처리해 줌 (6주차 Server State 연결)

### 7-2. Long Task 쪼개기

```js
// X: 1만 개를 한 번에 처리 -> 수백 ms 동안 클릭·렌더링이 멈춤
items.forEach(processItem);

// O: 일정 단위로 나누고, 사이사이 메인 스레드를 양보
async function processInChunks(items, size = 500) {
  for (let i = 0; i < items.length; i += size) {
    items.slice(i, i + size).forEach(processItem);
    await new Promise((r) => setTimeout(r, 0)); // 태스크로 양보 -> 렌더링·입력 처리 기회
  }
}
```
- 양보를 `await Promise.resolve()`(마이크로태스크)로 하면 **효과 없음** → 마이크로태스크는 렌더링 전에 전부 처리되므로 결국 한 덩어리로 실행됨
- 반드시 태스크(`setTimeout`, `MessageChannel`)나 `scheduler.yield()`(지원 브라우저 한정)로 양보해야 함
- 계산 자체가 무거우면 Web Worker로 메인 스레드 밖에서 처리 (2주차 Web Worker 연결)
- 매 프레임 갱신하는 애니메이션은 `setTimeout` 대신 `requestAnimationFrame` → 렌더링 직전에 실행돼서 프레임과 맞음

---

## 예상 면접 질문

<details><summary>Q1. JS는 싱글 스레드인데 어떻게 비동기 처리가 가능한가?</summary>

JS 코드를 실행하는 콜 스택은 하나지만, 타이머·네트워크처럼 기다리는 일은 브라우저(Web API)가 다른 스레드에서 처리함. 끝나면 콜백을 큐에 넣고, 이벤트 루프가 콜 스택이 비었을 때 꺼내서 실행함. 그래서 메인 스레드는 기다리지 않고 다음 코드를 계속 실행할 수 있음.
</details>

<details><summary>Q2. setTimeout(fn, 0)은 바로 실행되나?</summary>

안 됨. 0ms 뒤 Task Queue에 들어갈 뿐이고, 현재 동기 코드와 마이크로태스크가 전부 끝난 뒤 다음 루프에서 실행됨. delay는 최소 대기 시간이지 보장 시간이 아님.
</details>

<details><summary>Q3. Microtask와 Macrotask의 차이는?</summary>

처리 시점과 개수가 다름. 태스크는 루프 1회에 하나씩, 마이크로태스크는 스택이 빌 때마다 큐가 빌 때까지 전부 처리함. 그래서 `Promise.then`이 `setTimeout`보다 먼저 실행됨. 마이크로태스크가 계속 쌓이면 렌더링이 막힐 수 있음.
</details>

<details><summary>Q4. 다음 코드의 출력 순서는?</summary>

```js
async function a1() {
  console.log('a1 start');
  await a2();
  console.log('a1 end');
}
async function a2() {
  console.log('a2');
}

console.log('script start');
setTimeout(() => console.log('timeout'), 0);
a1();
new Promise((r) => { console.log('p1'); r(); })
  .then(() => console.log('then'));
console.log('script end');
```
정답: `script start` → `a1 start` → `a2` → `p1` → `script end` → `a1 end` → `then` → `timeout`

1. 동기 코드: `script start`, `a1 start`, `a2`(await 전까지 동기), `p1`(executor는 동기), `script end`
2. 마이크로태스크는 등록된 순서대로: `a1`의 재개(`a1 end`)가 먼저 등록됐고 그다음 `then`
3. 태스크: `timeout`

참고: 2019년 스펙 변경 전의 엔진(예: Node 8)에선 `await`가 마이크로태스크를 3번 거쳐서 `then`이 `a1 end`보다 먼저 찍혔음. 지금 브라우저·Node 기준으로는 위 순서가 맞음
</details>

<details><summary>Q5. Promise.all과 Promise.allSettled의 차이는?</summary>

`all`은 하나라도 실패하면 즉시 reject하고, `allSettled`는 전부 끝날 때까지 기다린 뒤 각 결과의 성공·실패를 배열로 돌려줌. 전부 있어야 의미 있는 데이터면 `all`, 일부 실패해도 나머지를 보여줘야 하면 `allSettled`를 씀. 둘 다 실패한 요청 외의 나머지 요청을 취소하지는 않음.
</details>

<details><summary>Q6. async/await와 Promise는 어떤 관계이고, await는 메인 스레드를 막는가?</summary>

async/await는 Promise 위에 얹은 문법임. async 함수는 항상 Promise를 반환하고, `await` 이후 코드는 `then` 콜백처럼 마이크로태스크로 실행됨. `await`는 그 함수만 멈추고 콜 스택을 비워 주기 때문에 메인 스레드는 막지 않음.
</details>

<details><summary>Q7. 검색 입력마다 API를 호출할 때 생길 수 있는 문제는?</summary>

응답 순서가 보장되지 않아서, 먼저 보낸 요청의 응답이 늦게 오면 최신 결과를 덮어쓰는 Race Condition이 생김. `useEffect` cleanup에서 `AbortController`로 이전 요청을 취소하거나, 최신 요청인지 확인하는 플래그를 둬서 해결함. 요청 횟수 자체는 debounce로 줄임 (13주차 연결).
</details>

---

## 참고 자료

- [MDN — Using microtasks in JavaScript with queueMicrotask()](https://developer.mozilla.org/en-US/docs/Web/API/HTML_DOM_API/Microtask_guide)
- [HTML Living Standard — Event loops](https://html.spec.whatwg.org/multipage/webappapis.html#event-loops)
- [Jake Archibald — Tasks, microtasks, queues and schedules](https://jakearchibald.com/2015/tasks-microtasks-queues-and-schedules/)
- [V8 Blog — Faster async functions and promises](https://v8.dev/blog/fast-async)
- [web.dev — Optimize long tasks](https://web.dev/articles/optimize-long-tasks)
- [Philip Roberts — What the heck is the event loop anyway? (JSConf EU)](https://www.youtube.com/watch?v=8aGhZQkoFbQ)
