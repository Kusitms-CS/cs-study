# 8주차 - JavaScript Core

## 1. Execution Context

### 구조

```text
Call Stack
┌──────────────────────────┐
│ inner() 실행 컨텍스트     │  ← 현재 실행 중
├──────────────────────────┤
│ outer() 실행 컨텍스트     │
├──────────────────────────┤
│ 전역 실행 컨텍스트        │
└──────────────────────────┘

실행 컨텍스트 = {
  LexicalEnvironment : 변수·함수 선언 기록 + 바깥 환경 참조(outer)
  this 바인딩
}
```

### 특징

- 코드가 실행되는 데 필요한 정보(변수, 함수, `this`, 바깥 스코프 참조)를 담은 환경. 전역 코드가 시작될 때와 함수가 호출될 때마다 만들어져 Call Stack에 쌓임
- 생성 단계에서 선언을 먼저 기록하고, 실행 단계에서 코드를 한 줄씩 실행하며 값을 할당함. 이 두 단계 때문에 Hoisting이 생김
- 함수 실행이 끝나면 컨텍스트는 Call Stack에서 빠지지만, 그 환경을 참조하는 함수가 남아 있으면 환경은 메모리에 남음(Closure)
- 3주차의 Call Stack, 9주차의 Event Loop와 이어지는 개념

### 정리

- 실행 컨텍스트는 "지금 실행 중인 코드가 무엇을 볼 수 있고 `this`가 무엇인지"를 담은 상자
- Scope, Hoisting, Closure, `this`는 모두 실행 컨텍스트가 만들어지는 방식에서 나온 결과

---

## 2. Scope

### 구조

```js
const a = "global";

function outer() {
  const b = "outer";

  function inner() {
    const c = "inner";
    console.log(a, b, c); // inner → outer → global 순서로 찾음 (Scope Chain)
  }

  inner();
}
```

```text
inner 스코프 ──outer 참조──> outer 스코프 ──outer 참조──> 전역 스코프 ──> null
```

### 특징

- 변수에 접근할 수 있는 유효 범위. 전역 스코프, 함수 스코프, 블록 스코프가 있음
- `var`는 함수 스코프, `let`/`const`는 블록(`{}`) 스코프
- 스코프는 함수를 어디서 호출했는지가 아니라 어디에 작성했는지로 정해짐(렉시컬 스코프)
- 안쪽에서 변수를 찾지 못하면 바깥 스코프로 차례대로 올라가며 찾음(Scope Chain). 끝까지 없으면 `ReferenceError`

### 정리

- 스코프는 코드를 작성한 위치로 정해지고, 안쪽에서 바깥은 보이지만 바깥에서 안쪽은 보이지 않음
- 변수 범위를 좁게 두려면 `var` 대신 블록 스코프인 `let`/`const`를 사용

---

## 3. Hoisting

### 구조

```js
console.log(x);   // undefined
var x = 1;

console.log(y);   // ReferenceError (TDZ)
let y = 2;

hello();          // "hello" — 함수 선언문은 함수 전체가 올라감
function hello() { console.log("hello"); }

bye();            // TypeError: bye is not a function
var bye = function () {};
```

```text
생성 단계
  var x       → 등록 + undefined로 초기화
  let y       → 등록만, 초기화 전 (TDZ)
  function    → 등록 + 함수 객체로 초기화
실행 단계
  코드를 위에서부터 실행하며 값 할당
```

### 특징

- 실행 컨텍스트 생성 단계에서 선언이 먼저 기록되어, 선언이 코드 위로 끌어올려진 것처럼 보이는 현상
- `var`는 `undefined`로 초기화되어 선언 전에 접근해도 에러가 나지 않음
- `let`/`const`도 호이스팅되지만 선언문이 실행되기 전까지 초기화되지 않아 접근하면 에러. 이 구간을 TDZ(Temporal Dead Zone)라고 함
- 함수 선언문은 함수 전체가 호이스팅되지만, 함수 표현식은 변수 규칙을 따름

### 정리

- 코드가 실제로 위로 옮겨지는 것이 아니라, 실행 전에 선언을 먼저 기록하기 때문에 생기는 현상
- "`let`/`const`는 호이스팅되지 않는다"가 아니라 "호이스팅되지만 TDZ 때문에 선언 전에 접근할 수 없다"가 정확한 설명

---

## 4. Closure

### 구조

```js
function createCounter() {
  let count = 0;            // 외부에서 직접 접근 불가

  return function () {
    count += 1;             // 바깥 함수의 count를 계속 참조
    return count;
  };
}

const counter = createCounter();  // createCounter는 실행이 끝남
counter(); // 1
counter(); // 2                   // 그래도 count는 살아 있음
```

### 특징

- 함수가 자신이 만들어진 위치의 렉시컬 환경을 기억해서, 바깥 함수가 끝난 뒤에도 그 변수에 접근할 수 있는 것
- 외부에서 직접 바꿀 수 없는 상태를 만들 수 있음(정보 은닉). 커스텀 Hook, debounce, 이벤트 핸들러 등에서 자주 쓰임
- 클로저가 참조하는 변수는 GC가 회수하지 못하므로, 큰 객체를 붙잡은 핸들러나 타이머를 정리하지 않으면 메모리 누수가 될 수 있음(3주차 Memory Leak과 연결)

```js
for (var i = 0; i < 3; i++) {
  setTimeout(() => console.log(i)); // 3, 3, 3 — 모든 콜백이 같은 i 하나를 공유
}
for (let i = 0; i < 3; i++) {
  setTimeout(() => console.log(i)); // 0, 1, 2 — 반복마다 새로운 i가 만들어짐
}
```

### 정리

- 클로저는 "함수 + 그 함수가 만들어질 때의 환경"
- 값을 복사해서 기억하는 것이 아니라 변수 자체를 참조하기 때문에, 나중에 값이 바뀌면 바뀐 값이 보임

---

## 5. this

### 구조

| 호출 방식 | `this` |
| --- | --- |
| 일반 함수 호출 `fn()` | strict mode: `undefined` / 아니면 전역 객체 |
| 메서드 호출 `obj.fn()` | 점 앞의 객체 `obj` |
| 생성자 호출 `new Fn()` | 새로 만들어진 인스턴스 |
| `call` / `apply` / `bind` | 직접 지정한 객체 |
| 화살표 함수 | 자신의 `this`가 없고 바깥 스코프의 `this`를 사용 |

### 특징

- 일반 함수의 `this`는 작성 위치가 아니라 호출 방식에 따라 정해짐(동적 바인딩). Scope가 작성 위치로 정해지는 것과 반대
- 메서드를 변수에 담거나 콜백으로 넘기면 점 앞의 객체가 사라져 `this`를 잃어버림

```js
const user = {
  name: "Kim",
  hello() { console.log(this.name); },
};

user.hello();                     // "Kim"
const fn = user.hello;
fn();                             // undefined 또는 TypeError — 일반 함수 호출이 됨
setTimeout(user.hello);           // 마찬가지로 this를 잃음
setTimeout(() => user.hello());   // "Kim" — 메서드 호출 형태를 유지
setTimeout(user.hello.bind(user)); // "Kim" — this를 고정
```

- 화살표 함수는 `this`를 바깥에서 가져오므로 콜백 안에서 바깥 `this`를 유지할 때 유용하지만, 객체의 메서드로 정의하면 그 객체를 가리키지 않음

### 정리

- `this`는 "이 함수를 누가 호출했는가"로 정해지고, 화살표 함수만 예외적으로 "어디에 작성했는가"를 따름
- 콜백으로 메서드를 넘길 때 `this`가 사라지는 문제는 화살표 함수나 `bind`로 해결

---

## 6. Prototype

### 구조

```js
function User(name) {
  this.name = name;
}
User.prototype.hello = function () {
  return `hi, ${this.name}`;
};

const kim = new User("Kim");
kim.hello(); // kim에는 hello가 없음 → User.prototype에서 찾음
```

```text
kim ──[[Prototype]]──> User.prototype ──[[Prototype]]──> Object.prototype ──> null
{ name }               { hello }                         { toString, ... }
```

### 특징

- JavaScript의 모든 객체는 다른 객체를 가리키는 내부 링크 `[[Prototype]]`을 가지고 있고, 자기에게 없는 속성은 이 링크를 따라 올라가며 찾음(Prototype Chain)
- 메서드를 prototype에 한 번만 두고 모든 인스턴스가 공유하므로 인스턴스마다 함수를 복사하지 않아 메모리를 아낌
- `class` 문법도 내부적으로는 prototype 기반으로 동작함
- `Object.getPrototypeOf(obj)`로 prototype을 확인하고, `Object.create(proto)`로 원하는 prototype을 가진 객체를 만들 수 있음

### 정리

- Prototype은 객체끼리 연결해서 속성을 물려받는 JavaScript의 상속 방식
- Scope Chain이 변수를 찾는 길이라면, Prototype Chain은 객체의 속성을 찾는 길

---

## 7. 딥다이브: React에서 만나는 Closure 문제 (Stale Closure)

### 왜 값이 갱신되지 않을까?

React 함수 컴포넌트는 렌더링마다 함수가 다시 실행되고, 그때마다 새로운 `count` 변수와 새로운 핸들러 함수가 만들어진다. 그런데 Effect가 처음 렌더링 때 한 번만 실행되면, 그 안의 콜백은 첫 렌더링의 `count`(0)만 계속 참조한다.

```jsx
function Timer() {
  const [count, setCount] = useState(0);

  useEffect(() => {
    const id = setInterval(() => {
      setCount(count + 1); // 항상 0 + 1 — 첫 렌더링의 count를 붙잡고 있음
    }, 1000);
    return () => clearInterval(id);
  }, []);

  return <p>{count}</p>; // 1에서 멈춤
}
```

```text
1번째 렌더링: count = 0  ← interval 콜백이 이 환경을 기억
2번째 렌더링: count = 1  ← 새 환경이지만 콜백은 모름
3번째 렌더링: count = 1  ...
```

클로저가 값을 복사해 두는 것은 아니다. 클로저는 변수 자체를 참조하지만, 렌더링마다 `count`라는 변수가 새로 만들어지기 때문에 오래된 콜백이 참조하는 첫 렌더링의 `count`는 계속 0으로 남아 있다. 즉 "예전 값을 기억한다"기보다 "예전 렌더링의 환경(변수)을 참조하고 있다"가 정확한 설명이다.

### 해결 방법

```jsx
// 1. 함수형 업데이트: 현재 값을 React가 넘겨주므로 클로저의 count에 의존하지 않음
setCount((prev) => prev + 1);

// 2. 의존성 배열에 count 추가: count가 바뀔 때마다 Effect를 다시 만들어 새 값을 참조
useEffect(() => {
  const id = setInterval(() => setCount(count + 1), 1000);
  return () => clearInterval(id);
}, [count]);

// 3. useRef: 렌더링과 관계없이 같은 객체를 유지하므로 항상 최신 값을 읽을 수 있음
const countRef = useRef(count);
countRef.current = count;
```

- 상태를 이전 값 기준으로 바꾸는 경우라면 함수형 업데이트가 가장 단순하다.
- 의존성 배열을 비워두기 위해 사용하는 값을 일부러 빼면 stale closure가 생긴다. `eslint-plugin-react-hooks`의 `exhaustive-deps` 규칙이 이런 누락을 알려준다.
- cleanup에서 `clearInterval`을 하지 않으면 이전 콜백이 남아 계속 실행되고, 그 콜백이 참조하는 환경도 회수되지 않는다.

### 정리

- Stale closure는 React의 버그가 아니라 "클로저는 만들어진 시점의 환경을 기억한다"는 JavaScript 규칙 그대로의 결과다.
- 렌더링마다 새로운 환경이 만들어진다는 점을 기억하면, 의존성 배열과 함수형 업데이트가 왜 필요한지 자연스럽게 이해된다.
