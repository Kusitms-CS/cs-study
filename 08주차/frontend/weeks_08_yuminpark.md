# [Week 8] 8주차 JavaScript Core 정리

## 목차

1. [Execution Context](#1-execution-context)
2. [Scope](#2-scope)
3. [Hoisting](#3-hoisting)
4. [Closure](#4-closure)
5. [this](#5-this)
6. [Prototype](#6-prototype)
7. [정리](#7-정리)

> 키워드 목록과 달리 **Execution Context를 맨 앞에** 두었음. Scope, Hoisting, Closure, `this`가 모두 Execution Context가 만들어지는 과정에서 결정되기 때문에, 이것을 먼저 이해하면 나머지가 하나로 이어짐

---

## 1. Execution Context

### 개념

- **Execution Context(실행 컨텍스트)**: JS 코드가 평가되고 실행되는 **환경**. 코드를 실행하는 데 필요한 정보(변수, 함수, 스코프, `this`)를 담고 있음
- JS 엔진은 코드를 실행할 때 먼저 실행 컨텍스트를 만들고, 그 안에서 코드를 한 줄씩 실행함
- 3주차의 **Call Stack에 쌓이는 단위가** 바로 실행 컨텍스트

### 종류

| 종류 | 생성 시점 | 개수 |
|---|---|---|
| Global Execution Context | 스크립트가 처음 실행될 때 | 하나 |
| Function Execution Context | **함수가 호출될 때마다** | 호출 횟수만큼 |
| Eval Execution Context | `eval()` 실행 시 | 거의 쓰이지 않음 |

- 함수는 **정의될 때가 아니라 호출될 때** 실행 컨텍스트가 만들어짐
- 같은 함수를 여러 번 호출하면 호출마다 독립된 실행 컨텍스트가 생기고, 재귀 호출도 단계마다 별도의 컨텍스트를 가짐

### 구성 요소

```
Execution Context
├─ Variable Environment   var 선언, 함수 선언
├─ Lexical Environment    let/const, 블록 스코프 + outer 참조 (스코프 체인)
└─ this Binding           이 컨텍스트에서의 this 값
```

| 구성 요소 | 역할 |
|---|---|
| Variable Environment | `var` 변수와 함수 선언을 저장 |
| Lexical Environment | `let`/`const`/`class` 선언을 저장하고, **바깥(outer) 환경에 대한 참조를** 가짐 |
| this Binding | `this`가 무엇을 가리키는지 |

- 과거 자료에서는 같은 개념을 **Variable Object, Scope Chain, this로** 설명하기도 함. 용어만 다르고 "선언 저장소 + 바깥 참조 + this"라는 구조는 같음
- **Lexical Environment의 outer 참조가 연결된 것이 스코프 체인이고**, 2장과 4장의 핵심이 됨

### 생성 단계와 실행 단계

| 단계 | 하는 일 |
|---|---|
| 생성(Creation) 단계 | 선언을 먼저 등록 (변수, 함수 → **호이스팅**), outer 환경 결정, `this` 결정 |
| 실행(Execution) 단계 | 코드를 위에서부터 한 줄씩 실행, 변수에 값을 할당, 함수 호출 |

```js
console.log(a); // undefined
var a = 1;
console.log(a); // 1
```

- 생성 단계에서 `a`가 `undefined`로 먼저 등록되고, 실행 단계에서 `1`이 할당됨

### Call Stack과 함께 보기

```js
const x = 'global';

function outer() {
  const y = 'outer';
  inner();
}

function inner() {
  const z = 'inner';
  console.log(z);
}

outer();
```

```
Call Stack (아래 → 위)

┌──────────────┐
│ inner() EC   │  ← 현재 실행 중 (맨 위)
├──────────────┤
│ outer() EC   │
├──────────────┤
│ Global EC    │
└──────────────┘
```

- 함수가 호출되면 새 컨텍스트가 **push**, 함수가 끝나면 **pop**
- 맨 위에 있는 컨텍스트가 현재 실행 중인 코드
- 스택이 비어야 비동기 콜백(이벤트 루프)이 실행될 수 있음

### 실행 컨텍스트로 결과 예측하기

```js
var a = 1;

function foo() {
  console.log(a); // ?
  var a = 2;
}

foo();
```

- `foo()` 호출 → `foo`의 컨텍스트 생성 단계에서 **함수 내부의 `var a`가 `undefined`로 먼저 등록됨**
- `console.log(a)`는 바깥의 `a = 1`이 아니라 **자기 컨텍스트의 `a`(undefined)를** 참조 → 출력은 `undefined`
- 같은 코드가 `let a = 2`였다면 TDZ 때문에 `ReferenceError`가 발생함 (3장)

### 종료된 컨텍스트의 변수는?

- 함수가 끝나면 컨텍스트는 Call Stack에서 제거되지만, **그 환경(Lexical Environment)을 다른 함수가 참조하고 있다면 메모리에 남음**
- 이것이 **클로저의** 원리 (4장)

---

## 2. Scope

### 개념

- **Scope(스코프)**: 변수와 함수를 **참조할 수 있는 유효 범위**
- JS 엔진이 식별자(변수 이름)를 만났을 때 **어디서부터 어떤 순서로 찾을지를** 정하는 규칙

### 스코프의 종류

| 종류 | 범위 | 만드는 선언 |
|---|---|---|
| Global Scope | 코드 전체 | 최상위에서 선언한 변수/함수 |
| Function Scope | 함수 내부 | `var`, 함수 선언 (함수 단위로 생김) |
| Block Scope | `{ }` 블록 내부 | `let`, `const`, `class` |
| Module Scope | 모듈(파일) 내부 | ES Module의 최상위 선언 |

### var vs let/const

| | var | let | const |
|---|---|---|---|
| 스코프 | 함수 스코프 | 블록 스코프 | 블록 스코프 |
| 재선언 | 가능 | 불가 | 불가 |
| 재할당 | 가능 | 가능 | 불가 |
| 호이스팅 시 초기화 | `undefined`로 초기화 | 초기화 안 됨 (TDZ) | 초기화 안 됨 (TDZ) |
| 전역 선언 시 | `window`의 프로퍼티가 됨 | 프로퍼티가 되지 않음 | 프로퍼티가 되지 않음 |

```js
if (true) {
  var a = 1;
  let b = 2;
}

console.log(a); // 1  → var는 블록을 무시하고 함수/전역 스코프에 속함
console.log(b); // ReferenceError → let은 블록 안에서만 유효
```

- `var`는 블록 스코프가 없어서 `if`, `for` 안에서 선언해도 바깥으로 새어 나옴
- 현재는 **`const`를 기본으로, 재할당이 필요할 때만 `let`**, `var`는 사용하지 않는 것이 일반적

### const는 불변이 아니다

```js
const user = { name: 'kim' };
user.name = 'lee';      // 가능: 객체의 내용 변경
user = { name: 'park' }; // TypeError: 재할당 불가
```

- `const`는 **변수에 다른 값을 다시 할당하는 것을** 막을 뿐, 객체 내부의 변경은 막지 않음 (1주차 불변성과 연결)

### Lexical Scope (정적 스코프)

- 스코프는 함수를 **어디서 호출했느냐가 아니라, 어디에 정의(작성)했느냐로** 결정됨
- 코드를 실행하기 전에, 작성된 구조만 보고 정해지기 때문에 "렉시컬(정적)" 스코프라고 부름

```js
const x = 'global';

function foo() {
  console.log(x);
}

function bar() {
  const x = 'bar';
  foo();          // foo는 bar 안에서 호출되었지만
}

bar(); // 'global'  → foo가 "정의된" 위치(전역)의 x를 참조
```

- 호출 위치 기준으로 결정되는 방식(동적 스코프)이었다면 `'bar'`가 출력됐을 것
- 이 "정의된 위치의 환경을 기억한다"는 성질이 **클로저의** 바탕이 됨

### 스코프 체인 (Scope Chain)

- 식별자를 현재 스코프에서 찾지 못하면 **바깥 스코프로 한 단계씩 올라가며** 찾음
- 전역까지 올라가도 없으면 `ReferenceError`

```js
const a = 'global';

function outer() {
  const b = 'outer';

  function inner() {
    const c = 'inner';
    console.log(a, b, c); // inner → outer → global 순으로 탐색
  }
  inner();
}

outer();
```

```
inner 스코프 { c }  ──►  outer 스코프 { b }  ──►  global 스코프 { a }  ──►  없으면 ReferenceError
   (현재)                 (outer 참조)              (outer 참조)
```

- 탐색은 **안에서 바깥 방향으로만** 가능. 바깥에서 안쪽 변수는 볼 수 없음
- 1장에서 본 Lexical Environment의 **outer 참조가** 이 체인을 만듦

### 변수 가림 (Shadowing)

```js
const name = 'global';

function greet() {
  const name = 'local';   // 바깥의 name을 가림
  console.log(name);      // 'local'
}

greet();
console.log(name);        // 'global'
```

- 안쪽 스코프에 같은 이름이 있으면 **가장 가까운 것이 우선이고**, 바깥 변수는 가려짐

### 전역 변수의 문제

```js
var count = 0;           // 브라우저: window.count 가 됨
let total = 0;           // 전역 스코프이지만 window.total 은 아님

function leak() {
  oops = 'implicit global'; // 선언 없이 할당 → 전역 변수가 됨 (sloppy mode)
}
```

- 전역 변수는 어디서든 수정할 수 있어서 **이름 충돌과 예측하기 어려운 버그의** 원인이 됨
- `'use strict'` 또는 ES Module(자동으로 strict)에서는 선언 없는 할당이 `ReferenceError`가 됨 (3주차 Memory Leak의 "의도치 않은 전역 변수")
- ES Module은 파일마다 **모듈 스코프를** 가지므로 최상위 `var`도 전역으로 새지 않음

### 블록 스코프 활용

```js
for (let i = 0; i < 3; i++) {
  const doubled = i * 2;   // 반복마다 새로 생성되는 블록 스코프 변수
}
console.log(typeof i);       // 'undefined' → 반복문 밖에서는 접근 불가
```

- 변수의 **생존 범위를 최소화할수록** 의도치 않은 수정과 충돌을 줄일 수 있음

---

## 3. Hoisting

### 개념

- **Hoisting(호이스팅)**: 변수와 함수의 **선언이 코드 실행 전에 먼저 처리되어**, 선언문보다 앞에서도 해당 식별자를 참조할 수 있는 것처럼 보이는 현상
- 코드가 실제로 위로 끌어올려지는 것이 아니라, **실행 컨텍스트의 생성 단계에서 선언이 먼저 등록되기** 때문에 생기는 결과 (1장)

### 선언 방식별 동작

| 선언 방식 | 호이스팅 시 | 선언 전에 사용하면 |
|---|---|---|
| `var` | 선언 + `undefined`로 초기화 | `undefined` |
| 함수 선언문 `function foo() {}` | 선언 + **함수 본체까지** 초기화 | 정상 호출 가능 |
| 함수 표현식 `var foo = function() {}` | `var`의 규칙을 따름 (`undefined`) | `TypeError: foo is not a function` |
| `let`, `const` | 선언만 등록, 초기화는 안 됨 | `ReferenceError` (TDZ) |
| `class` | `let`/`const`와 같음 | `ReferenceError` (TDZ) |

### var 호이스팅

```js
console.log(a); // undefined (에러가 아님)
var a = 10;
console.log(a); // 10
```

```js
// 엔진이 이해하는 순서 (개념적으로)
var a;            // 생성 단계: 선언 + undefined 초기화
console.log(a);   // undefined
a = 10;           // 실행 단계: 할당
console.log(a);   // 10
```

- **선언만** 끌어올려지고 **할당은 원래 위치에서** 일어남

### 함수 호이스팅

```js
hello();                  // 'hello' (선언문은 어디서든 호출 가능)
function hello() {
  console.log('hello');
}

bye();                    // TypeError: bye is not a function
var bye = function () {   // 표현식은 변수만 호이스팅 (값은 undefined)
  console.log('bye');
};
```

- 함수 선언문은 **함수 전체가** 호이스팅되지만, 함수 표현식은 **변수 선언만** 호이스팅됨

### 같은 이름이 있을 때

```js
console.log(typeof foo); // 'function'
var foo = 1;
function foo() {}
console.log(typeof foo); // 'number'
```

- 생성 단계에서 `var foo`(undefined)보다 **함수 선언이 우선해서** `foo`가 함수로 초기화됨
- 이후 실행 단계에서 `foo = 1` 할당이 일어나 `'number'`가 됨

### TDZ (Temporal Dead Zone)

- `let`/`const`/`class`는 호이스팅은 되지만, **선언문이 실행되기 전까지는 접근할 수 없는 구간이** 존재함
- 이 구간(블록의 시작 ~ 선언문 직전)을 **TDZ(일시적 사각지대)라고** 부름

```js
{
  // ---- TDZ 시작 ----
  console.log(x); // ReferenceError: Cannot access 'x' before initialization
  // ---- TDZ 끝 ----
  let x = 1;
  console.log(x); // 1
}
```

- 에러 메시지가 "x is not defined"가 아니라 **"before initialization"인** 점이 호이스팅이 일어났다는 증거

### let은 호이스팅되지 않는다는 오해

```js
let x = 'outer';

{
  console.log(x); // ReferenceError (undefined나 'outer'가 아님)
  let x = 'inner';
}
```

- `let`이 호이스팅되지 않는다면 `'outer'`가 출력돼야 함
- 블록 안의 `let x`가 **호이스팅되어 블록 전체를 자신의 영역으로 만들었기 때문에** 바깥의 `x`가 가려지고, TDZ에 걸려 에러가 발생함

### typeof와 TDZ

```js
console.log(typeof undeclared); // 'undefined' (선언 자체가 없으면 안전)

console.log(typeof tdz);        // ReferenceError
let tdz = 1;
```

- 예전에 안전하다고 알려진 `typeof`도 TDZ 구간의 변수에는 에러를 던짐

### 실무 가이드

- `var`를 쓰지 않고 `let`/`const`를 사용해서 **선언 전 접근을 에러로 드러내기**
- 변수는 사용하는 곳 **가까이에서 선언하고**, 함수 표현식/화살표 함수는 선언 이후에 호출
- ESLint의 `no-use-before-define`, `no-var` 규칙으로 방지

---

## 4. Closure

### 개념

- **Closure(클로저)**: 함수가 **자신이 정의된 환경(Lexical Environment)을 기억하고**, 그 함수가 **원래 스코프 밖에서 실행되더라도** 그 환경의 변수에 접근할 수 있는 현상
- 함수와 그 함수가 선언된 렉시컬 환경의 조합

```js
function createCounter() {
  let count = 0;                 // createCounter의 지역 변수

  return function () {
    count += 1;                  // 바깥 함수의 변수를 계속 사용
    return count;
  };
}

const counter = createCounter(); // createCounter()는 이미 종료됨
console.log(counter()); // 1
console.log(counter()); // 2
console.log(counter()); // 3
```

- `createCounter()`가 끝나서 실행 컨텍스트가 Call Stack에서 사라졌는데도 `count`가 유지됨

### 동작 원리

- 함수 객체가 만들어질 때, **그 시점의 Lexical Environment에 대한 참조를** 내부 슬롯(`[[Environment]]`)에 저장함
- 함수가 나중에 실행될 때, 이 참조를 **outer 환경으로** 삼아 스코프 체인을 구성함
- 바깥 함수의 컨텍스트가 사라져도, 그 환경을 **내부 함수가 참조하고 있으므로 GC 대상이 되지 않음** (3주차 Reachability)

```
createCounter() 호출
  └─ Lexical Environment { count: 0 }
              ▲
              │ [[Environment]] 참조
          counter (반환된 함수)

createCounter() 의 실행 컨텍스트는 Call Stack 에서 제거되지만,
{ count: 0 } 환경은 counter 가 참조하는 한 메모리에 남아 있음
```

### 호출마다 독립된 환경

```js
const c1 = createCounter();
const c2 = createCounter();

c1(); // 1
c1(); // 2
c2(); // 1  → c2는 별도의 환경(count)을 가짐
```

- `createCounter()`를 호출할 때마다 **새로운 실행 컨텍스트와 환경이** 만들어지므로, 각 클로저는 서로 독립된 상태를 가짐

### 활용 1: 데이터 은닉 (private 상태)

```js
function createAccount(initial) {
  let balance = initial;          // 외부에서 직접 접근할 수 없음

  return {
    deposit(amount) { balance += amount; },
    getBalance() { return balance; },
  };
}

const account = createAccount(1000);
account.deposit(500);
console.log(account.getBalance()); // 1500
console.log(account.balance);      // undefined (직접 접근 불가)
```

- 변수를 바깥에 노출하지 않고, **정해진 함수를 통해서만 접근하게** 만듦 (모듈 패턴)

### 활용 2: 함수 팩토리, 부분 적용

```js
const multiply = (a) => (b) => a * b;

const double = multiply(2);   // a = 2 를 기억하는 함수
const triple = multiply(3);

double(5); // 10
triple(5); // 15
```

### 활용 3: debounce / memoize

```js
function debounce(fn, delay) {
  let timer;                          // 호출 사이에 유지되어야 하는 상태
  return (...args) => {
    clearTimeout(timer);
    timer = setTimeout(() => fn(...args), delay);
  };
}

function memoize(fn) {
  const cache = new Map();            // 클로저 안에 숨겨진 캐시
  return (n) => {
    if (cache.has(n)) return cache.get(n);
    const result = fn(n);
    cache.set(n, result);
    return result;
  };
}
```

- 이벤트 핸들러, 타이머 콜백, 비동기 콜백이 **자신이 만들어진 시점의 값을** 기억하는 것도 모두 클로저

### 반복문과 클로저 (var vs let)

```js
for (var i = 0; i < 3; i++) {
  setTimeout(() => console.log(i), 0);
}
// 3 3 3
```

- `var i`는 **하나의 함수 스코프 변수라서** 세 콜백이 모두 같은 `i`를 참조함
- 콜백이 실행되는 시점에는 반복이 끝나서 `i`가 3

```js
for (let i = 0; i < 3; i++) {
  setTimeout(() => console.log(i), 0);
}
// 0 1 2
```

- `let`은 **반복마다 새로운 블록 스코프 바인딩을** 만들어서 각 콜백이 자기 반복의 `i`를 기억함

```js
// let이 없던 시절의 해결: 즉시 실행 함수(IIFE)로 매 반복마다 새 스코프 생성
for (var i = 0; i < 3; i++) {
  ((j) => setTimeout(() => console.log(j), 0))(i);
}
```

### 클로저와 메모리

```js
function load() {
  const bigData = new Array(1_000_000).fill('*');

  return function use() {
    console.log(bigData.length);   // bigData를 계속 참조
  };
}

const fn = load();   // bigData가 GC되지 않고 유지됨
```

- 클로저가 참조하는 변수는 **클로저가 살아있는 동안 해제되지 않음** (3주차 Memory Leak)
- 더 이상 필요 없으면 함수 참조를 해제(`fn = null`)하고, 큰 데이터를 불필요하게 붙잡지 않도록 설계
- 클로저 자체가 문제인 것은 아니고, **오래 사는 클로저가 큰 데이터를 참조할 때** 문제가 됨

### Stale Closure (React)

- 컴포넌트가 렌더링될 때마다 함수가 새로 만들어지고, 각 함수는 **그 렌더링 시점의 값을** 캡처함

```jsx
function Timer() {
  const [count, setCount] = useState(0);

  useEffect(() => {
    const id = setInterval(() => {
      console.log(count);       // 항상 0 (첫 렌더링의 count를 캡처한 클로저)
      setCount(count + 1);      // 항상 0 + 1 = 1 → 1에서 멈춤
    }, 1000);
    return () => clearInterval(id);
  }, []);                       // 의존성 배열이 비어 있어서 콜백이 갱신되지 않음
}
```

```jsx
// 해결 1: 최신 값을 인자로 받는 함수형 업데이트
setCount((prev) => prev + 1);

// 해결 2: 의존성 배열에 count를 포함 (effect가 count마다 다시 실행됨)
useEffect(() => { /* ... */ }, [count]);
```

### 정리: 클로저를 볼 때의 체크 포인트

| 질문 | 확인할 것 |
|---|---|
| 이 함수는 어디서 **정의되었나** | 그 위치의 변수들을 기억함 |
| 함수가 몇 번 만들어졌나 | 만들어질 때마다 독립된 환경을 가짐 |
| 참조하는 변수가 **언제 바뀌나** | 클로저는 값이 아니라 **변수(환경)를 참조하므로** 이후의 변경도 반영됨 |
| 오래 살아남는가 | 오래 살면 참조하는 데이터도 계속 메모리에 남음 |

---

## 5. this

### 개념

- **`this`**: 함수가 실행될 때 **"누가 이 함수를 호출했는가"를 가리키는 값**
- 스코프(정의된 위치로 결정)와 달리, **`this`는 호출되는 방식에 따라 결정됨** (화살표 함수는 예외)
- 같은 함수라도 어떻게 호출하느냐에 따라 `this`가 달라짐

```js
function who() {
  console.log(this);
}

who();               // 일반 호출
obj.who = who;
obj.who();           // 메서드 호출 → this는 obj
```

### 호출 방식별 this

| 호출 방식 | this | 예시 |
|---|---|---|
| 일반 함수 호출 | strict: `undefined` / sloppy: `globalThis` | `fn()` |
| 메서드 호출 | 점(`.`) 앞의 객체 | `obj.fn()` |
| 생성자 호출 | 새로 생성되는 객체 | `new Fn()` |
| `call` / `apply` / `bind` | 인자로 지정한 객체 | `fn.call(obj)` |
| 화살표 함수 | **자신의 this가 없음**, 정의된 위치의 외부 this | `() => {}` |
| DOM 이벤트 핸들러 (일반 함수) | 이벤트가 걸린 요소 (`currentTarget`) | `el.addEventListener('click', function () {})` |

### 1) 일반 함수 호출

```js
function show() {
  console.log(this);
}

show(); // sloppy mode: window (브라우저) / strict mode: undefined
```

- 엔진은 ES Module, `class` 내부, `'use strict'`에서 **자동으로 strict mode를** 적용함 → `undefined`

### 2) 메서드 호출

```js
const user = {
  name: 'kim',
  hello() {
    console.log(this.name);
  },
};

user.hello(); // 'kim' → 점 앞의 객체 user가 this
```

### 메서드를 꺼내서 호출하면 this를 잃는다

```js
const fn = user.hello;
fn();                    // undefined → 이제 "일반 호출"이므로 this가 user가 아님

setTimeout(user.hello, 0);                     // 마찬가지로 this를 잃음
button.addEventListener('click', user.hello);  // this는 button이 됨
```

- `this`는 **함수가 어디에 정의되었는지가 아니라, 호출하는 모양(`.` 앞에 객체가 있는지)으로** 결정되기 때문
- 콜백으로 메서드를 넘길 때 가장 많이 발생하는 문제

```js
// 해결 1: bind로 this를 고정
setTimeout(user.hello.bind(user), 0);

// 해결 2: 화살표 함수로 감싸서 메서드 호출 형태를 유지
setTimeout(() => user.hello(), 0);
```

### 3) 생성자 호출 (new)

```js
function Person(name) {
  this.name = name;      // this는 new가 만든 새 객체
}

const p = new Person('kim');
console.log(p.name);     // 'kim'
```

- `new`로 호출하면 **새 객체를 만들어 `this`에 바인딩하고**, 이 객체가 기본으로 반환됨 (6장)

### 4) call / apply / bind

| | 동작 | 인자 전달 |
|---|---|---|
| `call` | `this`를 지정하고 **즉시 호출** | 개별 인자: `fn.call(obj, a, b)` |
| `apply` | `this`를 지정하고 **즉시 호출** | 배열: `fn.apply(obj, [a, b])` |
| `bind` | `this`가 고정된 **새 함수를 반환** (호출은 안 함) | 개별 인자 (일부를 미리 고정 가능) |

```js
function greet(greeting, punctuation) {
  return `${greeting}, ${this.name}${punctuation}`;
}

const person = { name: 'kim' };

greet.call(person, 'hi', '!');      // 'hi, kim!'
greet.apply(person, ['hi', '!']);   // 'hi, kim!'

const boundGreet = greet.bind(person, 'hi'); // this와 첫 인자를 고정한 새 함수
boundGreet('!');                              // 'hi, kim!'
```

- `bind`로 고정한 `this`는 이후 `call`/`apply`로 **다시 바꿀 수 없음**

### 5) 화살표 함수

- 화살표 함수는 **자신만의 `this`가 없고**, 정의된 위치의 **외부(렉시컬) 스코프의 `this`를 그대로 사용함**
- 스코프 체인으로 `this`를 찾는 것처럼 동작하기 때문에, **호출 방식과 무관하게 정의될 때 이미 결정됨**

```js
const obj = {
  name: 'kim',

  later() {
    setTimeout(function () {
      console.log(this.name);   // undefined → 일반 함수 호출이라 this가 obj가 아님
    }, 0);

    setTimeout(() => {
      console.log(this.name);   // 'kim' → 바깥 later()의 this(obj)를 그대로 사용
    }, 0);
  },
};

obj.later();
```

- 화살표 함수는 `call`/`apply`/`bind`로도 `this`를 바꿀 수 없고, 생성자(`new`)로 사용할 수 없으며 `arguments`도 없음

**화살표 함수를 쓰면 안 되는 경우**

```js
const user = {
  name: 'kim',
  hello: () => console.log(this.name),  // this는 user가 아니라 바깥(전역/모듈)의 this
};

user.hello(); // undefined
```

- 객체 리터럴의 **메서드를 화살표 함수로 정의하면** 그 객체를 가리키지 못함 → 메서드는 일반 함수/축약 메서드로 정의
- 반대로 **콜백 안에서 바깥 `this`를 유지하고** 싶을 때는 화살표 함수가 적합

### 6) DOM 이벤트 핸들러

```js
button.addEventListener('click', function () {
  console.log(this === button);   // true → 이벤트가 걸린 요소
});

button.addEventListener('click', () => {
  console.log(this === button);   // false → 바깥 스코프의 this
});
```

- 요소가 필요하면 `this` 대신 **`event.currentTarget`을 사용하면** 함수 종류와 무관하게 안정적

### 7) class 메서드

```js
class Counter {
  count = 0;

  increment() {          // prototype에 정의된 일반 메서드
    this.count++;
  }

  incrementArrow = () => {   // 인스턴스마다 만들어지는 화살표 함수 (class field)
    this.count++;
  };
}

const c = new Counter();
const { increment, incrementArrow } = c;

increment();        // TypeError (this가 undefined: class 내부는 strict mode)
incrementArrow();   // 정상 (정의 시점에 this가 인스턴스로 고정됨)
```

- 이벤트 핸들러로 클래스 메서드를 넘길 때(`onClick={this.handleClick}`) `this`를 잃는 문제의 원인
- 해결: 화살표 함수 class field, 또는 생성자에서 `bind`

### this 결정 규칙 정리

```
① 화살표 함수인가?              → 정의된 위치의 외부 this (여기서 결정 끝, 이후 규칙 무시)
② new 로 호출했는가?            → 새로 생성된 객체
③ call / apply / bind 로 지정?  → 지정한 객체
④ obj.method() 형태인가?        → obj (점 앞의 객체)
⑤ 그 외 (일반 호출)             → strict: undefined / sloppy: globalThis
```

| 우선순위 | 규칙 |
|---|---|
| 1 | `new` 바인딩 |
| 2 | 명시적 바인딩 (`call`/`apply`/`bind`) |
| 3 | 암시적 바인딩 (메서드 호출) |
| 4 | 기본 바인딩 (일반 호출) |
| 별도 | 화살표 함수는 위 규칙과 무관하게 **외부 `this`를 렉시컬하게 캡처** |

### Scope와 this 혼동하지 않기

| | Scope (변수 탐색) | this |
|---|---|---|
| 결정 시점 | 코드를 **작성한 위치** (렉시컬) | 함수를 **호출한 방식** (화살표 함수는 제외) |
| 탐색 방식 | 스코프 체인을 따라 바깥으로 | 호출 시점에 한 번 결정 |
| 화살표 함수 | 동일하게 렉시컬 | `this`도 렉시컬하게 결정됨 |

---

## 6. Prototype

### 개념

- JS는 클래스 기반이 아니라 **프로토타입 기반으로** 상속을 구현하는 언어
- 모든 객체는 다른 객체를 가리키는 숨겨진 링크 **`[[Prototype]]`을** 가지고 있음
- 객체에서 프로퍼티를 찾지 못하면 이 링크를 따라 올라가며 탐색하는데, 이 연결을 **프로토타입 체인(Prototype Chain)이라고** 함

### 용어 정리

| 용어 | 설명 |
|---|---|
| `[[Prototype]]` | **모든 객체가** 가지는 내부 슬롯. 상속할 대상 객체를 가리킴 |
| `Object.getPrototypeOf(obj)` | `[[Prototype]]`을 읽는 표준 방법 (`obj.__proto__`로도 접근되지만 사용을 권장하지 않음) |
| `prototype` 프로퍼티 | **함수(생성자)만** 가지는 프로퍼티. `new`로 만든 인스턴스의 `[[Prototype]]`이 가리킬 객체 |
| `constructor` | `prototype` 객체에 기본으로 들어있는, 생성자 함수를 가리키는 프로퍼티 |

- `[[Prototype]]`(객체의 상속 링크)과 `prototype`(함수가 가진 프로퍼티)은 **다른 것이고**, 이 둘을 혼동하는 경우가 많음

### 프로토타입 체인

```js
function Animal(name) { this.name = name; }
Animal.prototype.eat = function () { return `${this.name} eats`; };

function Dog(name) { Animal.call(this, name); }
Dog.prototype = Object.create(Animal.prototype);
Dog.prototype.constructor = Dog;
Dog.prototype.bark = function () { return 'woof'; };

const d = new Dog('rex');
```

```
d (instance)         { name: 'rex' }
   │ [[Prototype]]
   ▼
Dog.prototype        { bark() }
   │ [[Prototype]]
   ▼
Animal.prototype     { eat() }
   │ [[Prototype]]
   ▼
Object.prototype     { toString(), hasOwnProperty() ... }
   │ [[Prototype]]
   ▼
null                 (체인의 끝)
```

- `d.bark()` → d에 없음 → `Dog.prototype`에서 발견
- `d.eat()` → d, `Dog.prototype`에 없음 → `Animal.prototype`에서 발견
- `d.toString()` → 체인 끝의 `Object.prototype`에서 발견
- 끝까지 없으면 `undefined` (메서드 호출이면 `TypeError`)

### 읽기와 쓰기의 차이

```js
const parent = { greet: 'hello' };
const child = Object.create(parent);   // child의 [[Prototype]]이 parent

child.greet;            // 'hello'  → child에 없어서 parent에서 찾음 (읽기: 체인 탐색)
child.greet = 'hi';     // child에 새 프로퍼티가 생김 (쓰기: 항상 자기 자신에게)
child.greet;            // 'hi'     → 자신의 프로퍼티가 우선 (가려짐)
parent.greet;           // 'hello'  → parent는 바뀌지 않음
```

- **읽을 때만** 체인을 탐색하고, **쓸 때는 항상 자기 객체에** 프로퍼티를 생성함 (프로토타입을 수정하지 않음)

### 생성자 함수와 prototype

```js
function Person(name) {
  this.name = name;                 // 인스턴스마다 따로 가지는 데이터
}

Person.prototype.hello = function () {   // 모든 인스턴스가 공유하는 메서드
  return `hi, ${this.name}`;
};

const a = new Person('kim');
const b = new Person('lee');

a.hello === b.hello;                          // true → 같은 함수를 공유
Object.getPrototypeOf(a) === Person.prototype; // true
```

- 메서드를 생성자 안에서 `this.hello = function...`으로 만들면 **인스턴스마다 함수 객체가 새로 생겨 메모리 낭비**
- `prototype`에 두면 **모든 인스턴스가 하나의 함수를 공유함**

### new 연산자가 하는 일

1. 빈 객체를 생성
2. 그 객체의 `[[Prototype]]`을 **생성자의 `prototype`으로** 연결
3. `this`를 그 객체에 바인딩하고 생성자 함수를 실행
4. 생성자가 객체를 반환하지 않으면 **생성한 객체를 반환**

```js
function myNew(Ctor, ...args) {
  const obj = Object.create(Ctor.prototype);      // 1, 2
  const result = Ctor.apply(obj, args);           // 3
  const isObject = result !== null && (typeof result === 'object' || typeof result === 'function');
  return isObject ? result : obj;                 // 4
}

const p = myNew(Person, 'kim');   // new Person('kim') 과 같은 결과
```

### class 문법

- ES2015의 `class`는 **프로토타입 상속을 더 읽기 쉽게 쓰는 문법(Syntactic Sugar)이고**, 내부 동작은 프로토타입 기반 그대로

```js
class Animal {
  constructor(name) { this.name = name; }
  eat() { return `${this.name} eats`; }       // Animal.prototype에 정의됨
}

class Dog extends Animal {
  bark() { return 'woof'; }                   // Dog.prototype에 정의됨
}

const d = new Dog('rex');

typeof Animal;                                          // 'function'
Object.getPrototypeOf(Dog.prototype) === Animal.prototype; // true → extends가 체인을 연결
d instanceof Animal;                                    // true
```

| | 생성자 함수 | class |
|---|---|---|
| 호출 방식 | `new` 없이 호출해도 동작 (this 문제 발생) | `new` 없이 호출하면 `TypeError` |
| 호이스팅 | 함수 선언은 호이스팅 | TDZ (`let`처럼 동작) |
| 모드 | 기본은 sloppy | 항상 strict |
| 메서드 열거 | `for...in`에 열거됨 | 열거되지 않음 |
| 상속 | `Object.create` + 수동 연결 | `extends`, `super` |

### 프로퍼티 확인 방법

| 방법 | 동작 |
|---|---|
| `'x' in obj` | 자신 + **프로토타입 체인에** 있으면 `true` |
| `Object.hasOwn(obj, 'x')` | **자신의** 프로퍼티일 때만 `true` (`hasOwnProperty`의 후속) |
| `Object.keys(obj)` | 자신의 열거 가능한 프로퍼티만 |
| `for...in` | 자신 + **체인의 열거 가능한 프로퍼티까지** 순회 |

```js
const base = { inherited: 1 };
const obj = Object.create(base);
obj.own = 2;

'inherited' in obj;                 // true
Object.hasOwn(obj, 'inherited');    // false
Object.keys(obj);                   // ['own']

for (const k in obj) console.log(k);  // 'own', 'inherited'
```

- `for...in`으로 순회할 때는 상속된 프로퍼티가 섞이므로 `Object.hasOwn`으로 걸러내거나 `Object.keys`/`entries`를 사용

### instanceof

- `obj instanceof Ctor`는 **`Ctor.prototype`이 `obj`의 프로토타입 체인 어딘가에 있는지** 확인함

```js
d instanceof Dog;      // true
d instanceof Animal;   // true  (체인에 Animal.prototype이 있음)
d instanceof Object;   // true  (모든 객체는 결국 Object.prototype으로 이어짐)
```

### 내장 객체도 프로토타입 체인

```js
const arr = [1, 2, 3];
arr.map((n) => n * 2);   // map은 arr에 없고 Array.prototype에 있음

// arr → Array.prototype → Object.prototype → null
Object.getPrototypeOf(arr) === Array.prototype; // true
```

- `map`, `filter`, `toString` 같은 내장 메서드는 모두 **프로토타입 체인에서 찾아서** 호출되는 것
- `Array.prototype`에 임의로 메서드를 추가하면 **모든 배열에** 영향을 주므로 내장 객체의 prototype을 수정하는 것은 피해야 함

### 프로토타입이 없는 객체

```js
const dict = Object.create(null);   // [[Prototype]]이 null
dict.key = 'value';

dict.toString;                       // undefined (상속받는 메서드가 없음)
'toString' in dict;                  // false
```

- `{}`로 만든 객체에 `'toString' in obj`가 `true`가 되는 문제를 피할 수 있어서, 순수한 **키-값 저장소로** 쓰기 좋음 (순서/크기가 필요하면 `Map`)

### Prototype Pollution (보안 연결)

- 공격자가 `__proto__` 같은 키를 이용해서 **`Object.prototype`에 프로퍼티를 주입하면**, 이후 만들어지는 **모든 객체가 그 값을 상속받게** 되는 취약점 (7주차 보안과 연결)

```js
// 외부 입력을 검증 없이 깊게 병합하는 코드
function merge(target, source) {
  for (const key in source) {
    if (typeof source[key] === 'object') {
      target[key] = merge(target[key] ?? {}, source[key]);
    } else {
      target[key] = source[key];
    }
  }
  return target;
}

merge({}, JSON.parse('{"__proto__": {"isAdmin": true}}'));

const user = {};
console.log(user.isAdmin);  // true → 모든 객체가 오염됨
```

- 방어: `__proto__`, `constructor`, `prototype` 키를 **차단하거나**, 병합 대상을 `Object.create(null)`/`Map`으로 만들고, 검증된 라이브러리를 사용

### 프로토타입 관련 권장 사항

- 상속은 **`class`와 `extends`를 사용하고**, `Object.getPrototypeOf`로 읽기만 함
- `Object.setPrototypeOf`나 `__proto__` 직접 변경은 **엔진 최적화를 깨뜨려 성능이 크게 떨어질 수 있어** 피함
- 상속 대신 **조합(Composition)으로** 기능을 합치는 설계가 더 유연한 경우가 많음

---

## 7. 정리

| 개념 | 핵심 |
|---|---|
| Execution Context | 코드가 실행되는 환경. 생성 단계(선언 등록, this 결정)와 실행 단계로 나뉘고, Call Stack에 쌓임 |
| Scope | 식별자의 유효 범위. **정의된 위치로** 결정(Lexical)되고, 스코프 체인을 따라 바깥으로 탐색 |
| Hoisting | 선언이 생성 단계에서 먼저 등록됨. `var`는 `undefined`, 함수 선언문은 전체, `let`/`const`는 TDZ |
| Closure | 함수가 **정의된 환경을 기억**. 외부 함수가 끝나도 환경이 GC되지 않고 유지됨 |
| this | **호출 방식으로** 결정. new > call/apply/bind > 메서드 호출 > 일반 호출, 화살표 함수는 렉시컬 |
| Prototype | 프로퍼티를 못 찾으면 `[[Prototype]]` 체인을 따라 탐색. `class`는 프로토타입 상속의 문법 |

| 구분 | 결정되는 시점 | 기준 |
|---|---|---|
| 스코프 (변수 탐색) | 코드 작성 시 (정적) | 정의된 위치 |
| 클로저 | 함수 생성 시 | 함수가 만들어진 시점의 환경 |
| this | 함수 호출 시 (동적) | 호출하는 방식 |
| 프로토타입 | 객체 생성 시 | `new` / `Object.create`로 연결된 대상 |