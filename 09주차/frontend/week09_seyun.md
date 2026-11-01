# 9주차 - 비동기 / Event Loop

## 1. Call Stack

- **콜 스택(Call Stack):** 함수 호출 시 생성되는 **실행 컨텍스트를 쌓아두고 순서대로 실행하는 LIFO(Last In First Out) 자료구조**
- 함수를 호출하면 **push**, 실행이 끝나면 **pop** 되며, 가장 위에 있는 컨텍스트가 **현재 실행 중인 코드**
- JS 엔진은 **콜 스택이 하나뿐인 싱글 스레드**라서 **한 번에 하나의 작업만** 실행할 수 있음

### 📍 콜 스택 동작 과정

```js
function multiply(a, b) {
  return a * b;
}
function square(n) {
  return multiply(n, n);
}
function printSquare(n) {
  console.log(square(n));
}
printSquare(4);
```

```
                                   ┌──────────────┐
                                   │ multiply(4,4)│
                  ┌──────────────┐ ├──────────────┤ ┌──────────────┐ ┌──────────────┐
                  │  square(4)   │ │  square(4)   │ │  square(4)   │ │ console.log  │
┌──────────────┐  ├──────────────┤ ├──────────────┤ ├──────────────┤ ├──────────────┤
│printSquare(4)│  │printSquare(4)│ │printSquare(4)│ │printSquare(4)│ │printSquare(4)│
├──────────────┤  ├──────────────┤ ├──────────────┤ ├──────────────┤ ├──────────────┤
│    main()    │  │    main()    │ │    main()    │ │    main()    │ │    main()    │
└──────────────┘  └──────────────┘ └──────────────┘ └──────────────┘ └──────────────┘
```

- 에러가 발생하면 콘솔에 찍히는 **Stack Trace**가 바로 그 시점의 콜 스택 상태

### 📍 Stack Overflow

- 콜 스택의 크기는 제한되어 있어, **종료 조건이 없는 재귀**처럼 컨텍스트가 계속 쌓이면 에러 발생

```js
function recurse() {
  recurse();
}
recurse(); // RangeError: Maximum call stack size exceeded
```

### 📍 Blocking (콜 스택 점유 문제)

- 콜 스택이 비어야 **이벤트 처리, 비동기 콜백, 화면 렌더링**이 가능함
- 오래 걸리는 동기 작업이 콜 스택을 점유하면 그동안 **클릭·스크롤에 반응하지 않고 화면이 멈춤**

```js
button.addEventListener("click", () => {
  const start = Date.now();
  while (Date.now() - start < 3000) {} // 3초 동안 콜 스택 점유 → 화면 전체가 멈춤
});
```

| 해결 방법               | 설명                                                                              |
| ----------------------- | --------------------------------------------------------------------------------- |
| 비동기 API 사용         | 네트워크·타이머처럼 기다리는 작업은 **Web API에 위임**하고 콜백으로 결과를 받음   |
| 작업 분할 (Chunking)    | 큰 작업을 잘게 나눠 `setTimeout` 등으로 쪼개 실행해 **중간중간 렌더링 기회를 줌** |
| Web Worker              | CPU를 많이 쓰는 계산은 **별도 스레드**에서 실행해 메인 스레드를 비워둠            |

### 💡 JS는 싱글 스레드인데 어떻게 비동기 처리가 가능한가요?

**⇒ 싱글 스레드인 것은 JS 엔진(콜 스택)이고, JS가 실행되는 브라우저나 Node.js 런타임은 멀티 스레드로 동작하기 때문. 타이머, 네트워크 요청, 이벤트 감지 같은 작업은 엔진이 아니라 런타임의 Web API(Node는 libuv)가 별도로 처리하고, 작업이 끝나면 콜백을 큐에 넣어둠. 이벤트 루프가 콜 스택이 비었을 때 큐의 콜백을 콜 스택으로 옮겨 실행해주므로, JS 코드는 싱글 스레드로 실행되면서도 기다리는 작업을 동시에 처리할 수 있음**

---

## 2. Web API

- **브라우저가 제공하는 API**로, JS 엔진(V8 등) 자체에는 포함되어 있지 않음
- JS 코드는 Web API를 **호출만 하고**, 실제 작업(타이머 대기, 네트워크 통신, 이벤트 감지)은 **브라우저의 다른 스레드가 처리**한 뒤 완료되면 **콜백을 태스크 큐에 넣음**

### 📍 JS 엔진 vs 런타임

| 구분      | JS 엔진 (V8, SpiderMonkey)                    | 런타임 (브라우저, Node.js)                                     |
| --------- | --------------------------------------------- | -------------------------------------------------------------- |
| 역할      | JS 코드를 **파싱·컴파일·실행**                | 엔진을 포함하고, 엔진 밖의 기능과 **이벤트 루프**를 제공      |
| 구성      | **Call Stack**, **Memory Heap**               | Web API(Node는 libuv, 내장 모듈), Task Queue, Event Loop       |
| 예시      | 변수 선언, 함수 호출, 연산                    | `setTimeout`, `fetch`, DOM, `fs.readFile`                      |

- `setTimeout`은 JS 언어 스펙(ECMAScript)이 아니라 **브라우저/Node가 제공하는 API**임

### 📍 주요 Web API

| 분류          | API                                                                 | 완료 시 콜백이 들어가는 곳 |
| ------------- | ------------------------------------------------------------------- | -------------------------- |
| 타이머        | `setTimeout`, `setInterval`                                         | Task Queue                 |
| 네트워크      | `fetch`, `XMLHttpRequest`, `WebSocket`                              | `fetch`는 Promise → Microtask Queue, XHR 이벤트는 Task Queue |
| DOM 이벤트    | `addEventListener` (click, scroll, input 등)                        | Task Queue                 |
| 렌더링 관련   | `requestAnimationFrame`                                             | 렌더링 직전에 실행 (Animation Frame) |
| Observer      | `IntersectionObserver`, `ResizeObserver`, `MutationObserver`        | `MutationObserver`는 Microtask, 나머지는 렌더링 단계에서 실행 |
| 기타          | `localStorage`, `Geolocation`, `Web Worker`                         | -                          |

### 📍 setTimeout의 동작

```js
console.log("A");
setTimeout(() => console.log("B"), 0);
console.log("C");
// A → C → B
```

1. `setTimeout` 호출 → 콜 스택에서 **타이머 등록을 Web API에 위임**하고 바로 pop
2. 브라우저가 **지정한 시간만큼 대기** 후 콜백을 **Task Queue에 추가**
3. 이벤트 루프가 **콜 스택이 비었을 때** 콜백을 콜 스택으로 옮겨 실행

- `setTimeout(fn, 0)`도 **즉시 실행되지 않고**, 현재 실행 중인 동기 코드가 모두 끝난 뒤 실행됨
- 지정한 delay는 **"최소 대기 시간"** 일 뿐 정확한 실행 시점을 보장하지 않음 (콜 스택이 바쁘면 더 늦어짐)
- 중첩된 타이머가 5단계 이상 깊어지면 최소 **4ms**로 제한되고, 비활성 탭에서는 **1초 이상**으로 늦춰지기도 함

### 💡 `setTimeout(fn, 1000)`은 정확히 1초 뒤에 실행되나요?

**⇒ 보장되지 않음. 1000ms는 Web API가 콜백을 Task Queue에 넣기까지의 최소 대기 시간일 뿐이고, 큐에 들어간 뒤에도 콜 스택이 비어 있고 앞선 태스크와 마이크로태스크가 모두 처리되어야 실행되기 때문. 그래서 무거운 동기 작업이 실행 중이면 그만큼 지연되고, 브라우저가 비활성 탭의 타이머를 늦추기도 함. 정확한 간격이 필요한 애니메이션은 `requestAnimationFrame`을, 시간 측정은 `performance.now()`와 실제 경과 시간을 비교하는 방식을 사용함**

---

## 3. Event Loop

- **이벤트 루프(Event Loop):** **콜 스택이 비어 있는지 계속 확인**하면서, 비어 있으면 **큐에 대기 중인 콜백을 콜 스택으로 옮겨 실행**시키는 런타임의 동작 방식
- JS 엔진이 아니라 **브라우저(또는 Node.js)가 제공**하는 메커니즘

### 📍 전체 구조

```
 ┌────────────────────────────┐        ┌────────────────────────────┐
 │         JS Engine          │        │          Web APIs          │
 │  ┌────────┐  ┌──────────┐  │ call   │  setTimeout, fetch,        │
 │  │  Heap  │  │Call Stack│──┼───────▶│  DOM events, ...           │
 │  └────────┘  └──────────┘  │        └─────────────┬──────────────┘
 └──────────────────▲─────────┘                      │ callback
                    │                                ▼
                    │        ┌───────────────────────────────────────┐
                    │        │ Microtask Queue  [then] [await] ...   │
           Event Loop        ├───────────────────────────────────────┤
           (when stack       │ Task Queue       [timeout] [click] ...│
            is empty)        └───────────────────┬───────────────────┘
                    │                            │
                    └────────────────────────────┘
```

### 📍 이벤트 루프의 한 사이클 (브라우저)

1. **Task Queue에서 가장 오래된 태스크 1개**를 꺼내 실행 (처음에는 스크립트 전체 실행이 첫 태스크)
2. 콜 스택이 비면 **Microtask Queue가 빌 때까지 모두 실행** (실행 중 새로 추가된 마이크로태스크도 이어서 실행)
3. 렌더링이 필요하면 **렌더링 단계** 수행 → `requestAnimationFrame` 콜백 → Style → Layout → Paint
4. 다시 1번으로 반복

- 렌더링은 보통 **약 16.7ms(60fps)마다** 기회가 오며, 화면 변경이 없으면 생략됨
- 즉 **태스크 하나 → 마이크로태스크 전부 → (렌더링)** 순서가 핵심

### 📍 Node.js의 이벤트 루프

- Node.js는 **libuv** 기반으로, 여러 단계(Phase)를 순서대로 돌며 각 단계의 큐를 처리

| Phase            | 처리하는 콜백                                      |
| ---------------- | -------------------------------------------------- |
| timers           | `setTimeout`, `setInterval`                        |
| pending callbacks | 이전 루프에서 지연된 일부 I/O 콜백                |
| poll             | 새로운 **I/O 이벤트** 대기 및 처리 (`fs`, 네트워크) |
| check            | `setImmediate`                                     |
| close callbacks  | `socket.on("close")` 등                            |

- 각 콜백 실행 사이마다 **`process.nextTick` 큐 → Promise 마이크로태스크 큐** 순으로 먼저 비움

### 💡 이벤트 루프가 무엇인지 설명해주세요

**⇒ JS는 콜 스택이 하나인 싱글 스레드라 비동기 작업을 직접 기다릴 수 없어서, 타이머나 네트워크 같은 작업은 브라우저의 Web API에 맡기고 완료된 콜백은 큐에 쌓아둠. 이벤트 루프는 콜 스택이 비었는지 계속 확인하다가 비면 Task Queue에서 태스크 하나를 꺼내 실행하고, 이어서 Microtask Queue를 모두 비운 뒤, 필요하면 화면을 렌더링하는 과정을 반복함. 이 덕분에 싱글 스레드인 JS가 블로킹 없이 비동기 작업을 처리할 수 있음**

### 💡 무거운 작업을 하면 왜 화면이 멈추나요?

**⇒ 브라우저의 렌더링도 이벤트 루프의 한 단계라서, 콜 스택이 비고 현재 태스크와 마이크로태스크가 모두 끝나야만 렌더링 기회가 오기 때문. 50ms 이상 메인 스레드를 점유하는 Long Task가 있으면 그동안 클릭 이벤트 처리와 화면 갱신이 모두 밀려 버벅임이 생기고, INP 같은 Web Vitals 지표도 나빠짐. 그래서 무거운 작업은 Web Worker로 옮기거나, 작업을 잘게 쪼개 태스크 사이에 렌더링이 끼어들 수 있게 해야 함**

---

## 4. Promise

- **비동기 작업의 최종 완료(또는 실패)와 그 결과 값을 나타내는 객체**
- 콜백 방식의 **콜백 지옥(Callback Hell)** 과 에러 처리의 어려움을 해결하기 위해 ES6에서 도입

### 📍 콜백 지옥 (Callback Hell)

```js
getUser(id, (user) => {
  getPosts(user.id, (posts) => {
    getComments(posts[0].id, (comments) => {
      // 들여쓰기가 계속 깊어지고, 에러 처리를 단계마다 따로 해야 함
    });
  });
});
```

```js
// Promise 체이닝으로 평평하게
getUser(id)
  .then((user) => getPosts(user.id))
  .then((posts) => getComments(posts[0].id))
  .then((comments) => console.log(comments))
  .catch((err) => console.error(err)); // 어느 단계에서 실패해도 한 곳에서 처리
```

### 📍 Promise의 상태

| 상태                  | 설명                                       | 전환                        |
| --------------------- | ------------------------------------------ | --------------------------- |
| Pending (대기)        | 비동기 작업이 **아직 완료되지 않은** 상태  | 초기 상태                   |
| Fulfilled (이행)      | 작업이 **성공**해 결과 값을 가진 상태      | `resolve(value)` 호출 시    |
| Rejected (거부)       | 작업이 **실패**해 에러를 가진 상태         | `reject(error)` 호출 또는 예외 발생 시 |

- Fulfilled 또는 Rejected가 된 상태를 **Settled**라고 하며, 한 번 Settled되면 **상태가 다시 바뀌지 않음**

```js
const promise = new Promise((resolve, reject) => {
  console.log("executor"); // executor는 생성 즉시 "동기적으로" 실행됨
  setTimeout(() => resolve("done"), 1000);
});

promise
  .then((value) => console.log(value)) // "done"
  .catch((err) => console.error(err))
  .finally(() => console.log("성공/실패 관계없이 실행"));
```

### 📍 then / catch / finally

- `then`, `catch`, `finally`는 **항상 새로운 Promise를 반환**하므로 체이닝 가능
- `then` 콜백에서 **값을 반환하면** 다음 `then`으로 전달되고, **Promise를 반환하면** 그 Promise가 완료될 때까지 기다림
- `then` 콜백에서 **에러를 던지면** 가장 가까운 `catch`로 이동
- `then`, `catch`의 콜백은 Promise가 Settled되었을 때 **Microtask Queue**에 들어가 실행됨

### 📍 Promise 정적 메서드

| 메서드                | 동작                                                                       | 실패 처리                              |
| --------------------- | -------------------------------------------------------------------------- | -------------------------------------- |
| `Promise.all`         | **모두 성공**하면 결과 배열 반환                                          | **하나라도 실패하면 즉시 reject**      |
| `Promise.allSettled`  | 성공/실패 관계없이 **모두 끝나면** `{ status, value/reason }` 배열 반환   | reject되지 않음                        |
| `Promise.race`        | **가장 먼저 Settled된** Promise의 결과 (성공이든 실패든)                  | 가장 먼저 끝난 것이 실패면 reject      |
| `Promise.any`         | **가장 먼저 성공한** Promise의 결과                                       | 모두 실패하면 `AggregateError`         |

```js
// 타임아웃 구현
const timeout = (ms) =>
  new Promise((_, reject) => setTimeout(() => reject(new Error("timeout")), ms));

await Promise.race([fetch("/api/data"), timeout(5000)]);
```

### 💡 `Promise.all`과 `Promise.allSettled`는 언제 각각 사용하나요?

**⇒ `Promise.all`은 하나라도 실패하면 즉시 reject되므로, 결제 정보와 주문 정보처럼 모든 요청이 성공해야 의미가 있는 경우에 사용함. `Promise.allSettled`는 일부가 실패해도 모든 결과를 받을 수 있어서, 대시보드의 여러 위젯 데이터처럼 각각 독립적이라 실패한 것만 에러로 보여주면 되는 경우에 적합함. 참고로 `Promise.all`이 reject되어도 나머지 요청은 취소되지 않고 계속 진행되므로, 취소가 필요하면 `AbortController`를 함께 사용해야 함**

### 💡 Promise에서 `catch`를 빠뜨리면 어떻게 되나요?

**⇒ reject된 Promise를 아무도 처리하지 않으면 `try/catch`로 잡히지 않고, 브라우저에서는 `unhandledrejection` 이벤트와 함께 콘솔에 "Uncaught (in promise)" 에러가 출력되며, Node.js(v15+)에서는 기본적으로 프로세스가 종료됨. 비동기 에러는 이미 콜 스택을 벗어난 뒤에 발생하므로 바깥의 동기 `try/catch`로는 잡을 수 없어서, 체인의 마지막에 `catch`를 붙이거나 `await`과 `try/catch`를 함께 사용해야 함**

---

## 5. async / await

- **Promise를 동기 코드처럼 읽기 쉽게 작성할 수 있게 해주는 문법** (ES2017)
- `async` 함수는 **항상 Promise를 반환**하고, `await`은 **Promise가 Settled될 때까지 해당 async 함수의 실행을 일시 중지**함

```js
async function loadComments(id) {
  try {
    const user = await getUser(id);
    const posts = await getPosts(user.id);
    return await getComments(posts[0].id);
  } catch (err) {
    console.error(err); // 동기 코드처럼 try/catch로 에러 처리
  }
}
```

### 📍 async / await의 동작

- `async` 함수에서 값을 `return`하면 **`Promise.resolve(값)`** 으로 감싸져 반환, 에러를 던지면 **rejected Promise** 반환
- `await`을 만나면 **async 함수만 일시 중지되고 콜 스택에서 빠져나감** → 메인 스레드는 막히지 않고 다른 코드를 계속 실행
- 기다리던 Promise가 완료되면 **나머지 코드(await 이후)가 Microtask Queue에 들어가** 이어서 실행됨

```js
async function foo() {
  console.log(2);
  await null; // 이후 코드는 마이크로태스크로 예약됨
  console.log(4);
}

console.log(1);
foo();
console.log(3);
// 1 → 2 → 3 → 4
```

### 📍 순차 실행 vs 병렬 실행

```js
// 순차 실행: 서로 의존성이 없는데도 1초 + 1초 = 약 2초
const user = await fetchUser();
const products = await fetchProducts();

// 병렬 실행: 동시에 시작해서 약 1초
const [user2, products2] = await Promise.all([fetchUser(), fetchProducts()]);
```

- 이전 결과가 다음 요청에 필요할 때만 순차로, **독립적인 요청은 `Promise.all`로 병렬** 처리

### 📍 반복문에서의 await

```js
// forEach는 콜백의 Promise를 기다리지 않음 → 순서 보장 X, 완료를 기다리지 않고 다음 줄 실행
ids.forEach(async (id) => {
  await deleteItem(id);
});
console.log("done"); // 삭제가 끝나기 전에 출력

// 순차 처리가 필요하면 for...of
for (const id of ids) {
  await deleteItem(id);
}

// 병렬 처리가 필요하면 map + Promise.all
await Promise.all(ids.map((id) => deleteItem(id)));
```

### 📍 Promise vs async/await

| 구분        | Promise (`then` 체이닝)              | async / await                                   |
| ----------- | ------------------------------------ | ----------------------------------------------- |
| 가독성      | 체인이 길어지면 흐름 파악이 어려움   | **동기 코드처럼 위에서 아래로** 읽힘           |
| 에러 처리   | `.catch()`                           | `try / catch`                                   |
| 중간 값 사용 | 여러 단계의 값을 쓰려면 중첩/변수 필요 | 변수에 담아 **자유롭게 사용**                 |
| 디버깅      | 콜백 단위로 끊겨 추적이 어려움       | 브레이크포인트, 스택 트레이스 추적이 쉬움       |
| 관계        | 기반 기술                            | **Promise 위에서 동작하는 문법** (대체가 아님) |

### 💡 `await`을 쓰면 메인 스레드가 멈추나요?

**⇒ 멈추지 않음. `await`은 해당 async 함수의 실행만 일시 중지하고 함수를 콜 스택에서 빼내서, 그동안 메인 스레드는 다른 코드나 이벤트, 렌더링을 계속 처리함. 기다리던 Promise가 완료되면 `await` 이후의 코드가 마이크로태스크로 예약되어 이어서 실행됨. 즉 "함수 입장에서는 기다리지만, 스레드 입장에서는 기다리지 않는" 구조이며, 동기적으로 무거운 연산을 `await`으로 감싸도 그 연산 자체는 여전히 메인 스레드를 막음**

### 💡 여러 API를 `await`으로 연달아 호출할 때 주의할 점

**⇒ 서로 의존성이 없는 요청을 `await`으로 하나씩 호출하면 앞 요청이 끝나야 다음 요청이 시작되어 전체 시간이 각 요청 시간의 합이 되는 워터폴 문제가 생김. 독립적인 요청은 `Promise.all`로 동시에 시작해 가장 오래 걸리는 요청 시간만큼만 기다리도록 해야 함. 또한 `forEach`는 async 콜백의 완료를 기다리지 않으므로, 순차 처리가 필요하면 `for...of`, 병렬 처리가 필요하면 `map` + `Promise.all`을 사용해야 함**

---

## 6. Microtask / Macrotask

- 비동기 콜백이 대기하는 큐는 하나가 아니라 **우선순위가 다른 두 종류**로 나뉨
- **Macrotask(Task):** 이벤트 루프 한 사이클에 **하나씩** 실행
- **Microtask:** 현재 태스크가 끝날 때마다 **큐가 빌 때까지 전부** 실행 → **Macrotask보다 우선순위가 높음**

### 📍 종류

| 구분                    | 해당 작업                                                                                   |
| ----------------------- | ------------------------------------------------------------------------------------------- |
| Macrotask (Task Queue)  | 스크립트 실행, `setTimeout`, `setInterval`, DOM 이벤트(click 등), `MessageChannel`, I/O, `setImmediate`(Node) |
| Microtask Queue         | Promise의 `then/catch/finally`, `await` 이후 코드, `queueMicrotask`, `MutationObserver`, `process.nextTick`(Node, 가장 우선) |
| 렌더링 단계             | `requestAnimationFrame` (렌더링 직전에 실행, 태스크/마이크로태스크와는 별개)                |

### 📍 실행 순서 예제

```js
console.log("1");

setTimeout(() => console.log("2"), 0);

Promise.resolve()
  .then(() => console.log("3"))
  .then(() => console.log("4"));

queueMicrotask(() => console.log("5"));

(async () => {
  console.log("6");
  await null;
  console.log("7");
})();

console.log("8");

// 출력: 1 → 6 → 8 → 3 → 5 → 7 → 4 → 2
```

1. **동기 코드 실행:** `1`, async 함수 내부의 `await` 전까지인 `6`, `8` 출력
   - 이 과정에서 Microtask Queue: `[3, 5, 7]`, Task Queue: `[2]`
2. **Microtask Queue를 모두 비움:** `3` 실행 → 다음 `then`인 `4`가 큐 뒤에 추가됨 → `5` → `7` → `4`
3. **Task Queue에서 하나 실행:** `2`

- `4`가 `5`, `7`보다 늦은 이유는 `then` 체인의 다음 콜백이 **이전 `then`이 실행된 뒤에야 큐에 추가**되기 때문

### 📍 Microtask Starvation (기아 상태)

- 마이크로태스크는 **큐가 빌 때까지 계속 실행**되므로, 마이크로태스크가 계속 새 마이크로태스크를 추가하면 **Task Queue와 렌더링이 영원히 실행되지 못함**

```js
function loop() {
  Promise.resolve().then(loop); // 화면이 멈춤 (렌더링, 클릭 이벤트 처리 불가)
}
loop();

function loop2() {
  setTimeout(loop2, 0); // 화면은 멈추지 않음 (태스크 사이에 렌더링 기회가 있음)
}
loop2();
```

### 💡 Microtask Queue와 Task Queue를 분리한 이유

**⇒ Promise의 후속 처리처럼 현재 작업과 이어지는 작업은 다른 이벤트나 렌더링이 끼어들기 전에 바로 처리되어야 상태의 일관성이 보장되기 때문. 예를 들어 데이터를 받아 상태를 갱신하는 `then` 체인 사이에 렌더링이 끼어들면 중간 상태의 화면이 보일 수 있음. 그래서 마이크로태스크는 현재 태스크가 끝나자마자 렌더링 전에 모두 처리하고, 타이머나 사용자 이벤트 같은 독립적인 작업은 태스크로 분리해 사이사이에 렌더링 기회를 줌**

### 💡 `setTimeout(fn, 0)`과 `Promise.resolve().then(fn)` 중 무엇이 먼저 실행되나요?

**⇒ `Promise.resolve().then(fn)`이 먼저 실행됨. `then` 콜백은 Microtask Queue에, `setTimeout` 콜백은 Task Queue에 들어가는데, 이벤트 루프는 현재 실행 중인 코드가 끝나면 Microtask Queue를 먼저 모두 비운 뒤에 Task Queue에서 다음 태스크를 꺼내기 때문. 같은 이유로 `await` 이후의 코드도 `setTimeout(fn, 0)`보다 먼저 실행됨**
