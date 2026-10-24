# 8주차 - JavaScript Core

## 1. Scope

- **스코프(Scope):** 변수와 함수 같은 **식별자에 접근할 수 있는 유효 범위**
- JS는 **함수를 어디서 호출했는지가 아니라 어디에 정의했는지**로 상위 스코프가 결정되는 **렉시컬 스코프(Lexical Scope, 정적 스코프)** 를 따름

### 📍 스코프의 종류

| 스코프         | 설명                                                                     | 생성 기준                         |
| -------------- | ------------------------------------------------------------------------ | --------------------------------- |
| Global Scope   | 코드 어디서든 접근 가능한 최상위 스코프                                  | 파일(스크립트) 최상단             |
| Module Scope   | ES Module 파일마다 생기는 스코프, `export`하지 않으면 외부에서 접근 불가 | `<script type="module">`, ESM 파일 |
| Function Scope | 함수 내부에서만 접근 가능                                                | `function` 블록                   |
| Block Scope    | `{}` 블록 내부에서만 접근 가능 (`let`, `const`, `class`)                 | `if`, `for`, `while`, `{}`        |

### 📍 var vs let vs const

| 구분        | `var`                              | `let`                   | `const`                                 |
| ----------- | ---------------------------------- | ----------------------- | --------------------------------------- |
| 스코프      | **함수 스코프**                    | **블록 스코프**         | **블록 스코프**                         |
| 재선언      | 가능                               | 불가                    | 불가                                    |
| 재할당      | 가능                               | 가능                    | **불가** (객체 내부 값 변경은 가능)     |
| 호이스팅    | 선언 + `undefined`로 초기화        | 선언만 (TDZ 존재)       | 선언만 (TDZ 존재)                       |
| 전역 선언 시 | `window`(전역 객체)의 프로퍼티가 됨 | 전역 객체에 추가되지 않음 | 전역 객체에 추가되지 않음             |

```js
if (true) {
  var a = 1;
  let b = 2;
}
console.log(a); // 1 → var는 블록을 무시하고 함수/전역 스코프에 붙음
console.log(b); // ReferenceError: b is not defined
```

### 📍 스코프 체인 (Scope Chain)

- 식별자를 찾을 때 **현재 스코프 → 바깥 스코프 → ... → 전역 스코프** 순으로 탐색하는 연결 구조
- 끝까지 찾지 못하면 `ReferenceError`
- 안쪽 스코프에 같은 이름이 있으면 바깥 변수가 가려지는 **섀도잉(Shadowing)** 발생

```js
const x = "global";

function outer() {
  const x = "outer";
  function inner() {
    console.log(x); // "outer" → 가장 가까운 스코프부터 탐색
  }
  inner();
}
outer();
```

### 📍 렉시컬 스코프 (Lexical Scope)

```js
const value = "global";

function printValue() {
  console.log(value);
}

function run() {
  const value = "run";
  printValue(); // "global" → 호출 위치(run)가 아니라 정의 위치(전역) 기준
}
run();
```

### 💡 `var` 대신 `let`, `const`를 써야 하는 이유

**⇒ `var`는 함수 스코프라서 `if`나 `for` 블록 안에서 선언해도 블록 밖에서 접근할 수 있고, 같은 이름으로 재선언해도 에러가 나지 않아 의도치 않게 값을 덮어쓰기 쉬움. 또한 선언 전에 접근해도 `undefined`가 나와 버그를 늦게 발견하게 됨. `let`/`const`는 블록 스코프와 TDZ로 이런 실수를 막아주므로, 기본은 `const`로 선언하고 재할당이 필요할 때만 `let`을 사용하는 것이 좋음**

### 💡 렉시컬 스코프와 동적 스코프의 차이

**⇒ 렉시컬 스코프는 함수를 정의한 위치에 따라 상위 스코프가 결정되고, 동적 스코프는 함수를 호출한 위치에 따라 결정됨. JS는 렉시컬 스코프를 따르므로 함수가 어디서 호출되든 정의될 때의 바깥 변수를 참조하며, 이 성질 덕분에 클로저가 가능함. 단, JS에서 `this`는 예외적으로 호출 방식에 따라 동적으로 결정됨**

---

## 2. Hoisting

- **호이스팅(Hoisting):** 코드 실행 전에 **변수·함수 선언이 스코프의 최상단으로 끌어올려진 것처럼 동작**하는 현상
- 실제로 코드가 이동하는 것이 아니라, **실행 컨텍스트의 생성 단계에서 선언을 먼저 등록**하기 때문에 발생

### 📍 선언 방식별 호이스팅

| 선언 방식                            | 호이스팅 | 선언 전에 접근하면                                |
| ------------------------------------ | -------- | ------------------------------------------------- |
| `var`                                | O        | **`undefined`** (선언과 동시에 `undefined`로 초기화) |
| `let` / `const`                      | O        | **`ReferenceError`** (TDZ)                        |
| 함수 선언문 `function foo() {}`      | O        | **정상 호출 가능** (함수 전체가 등록됨)           |
| 함수 표현식 `var foo = function(){}` | 변수만 O | `TypeError: foo is not a function` (`undefined` 호출) |
| `class`                              | O        | **`ReferenceError`** (TDZ)                        |

```js
console.log(a); // undefined
var a = 1;

console.log(b); // ReferenceError: Cannot access 'b' before initialization
let b = 2;

hello(); // "hello" → 함수 선언문은 전체가 호이스팅
function hello() {
  console.log("hello");
}

bye(); // TypeError: bye is not a function
var bye = function () {
  console.log("bye");
};
```

### 📍 TDZ (Temporal Dead Zone, 일시적 사각지대)

- `let`, `const`, `class`도 호이스팅되지만, **선언문에 도달해 초기화되기 전까지는 접근할 수 없는 구간**
- 이 구간에서 접근하면 `ReferenceError` 발생

```js
let x = "outer";
{
  console.log(x); // ReferenceError → 바깥 x가 아니라, TDZ에 있는 안쪽 x를 참조
  let x = "inner";
}
```

- 위 예시는 `let`이 **호이스팅되지 않았다면** 바깥의 `"outer"`가 출력되었어야 함 → 호이스팅은 되지만 초기화되지 않았다는 증거

### 📍 변수 생성 3단계

| 단계            | 설명                                     | `var`               | `let` / `const`        |
| --------------- | ---------------------------------------- | ------------------- | ---------------------- |
| 선언 (Declaration) | 스코프에 식별자 등록                  | 생성 단계           | 생성 단계              |
| 초기화 (Initialization) | 메모리를 확보하고 `undefined`로 초기화 | **생성 단계 (선언과 동시)** | **선언문에 도달했을 때** |
| 할당 (Assignment) | 실제 값 할당                           | 할당문 실행 시      | 할당문 실행 시         |

- **선언과 초기화 사이의 구간**이 바로 TDZ

### 💡 `let`, `const`는 호이스팅이 안 되나요?

**⇒ `let`, `const`도 호이스팅됨. 실행 컨텍스트 생성 단계에서 식별자가 스코프에 등록되지만 `var`와 달리 `undefined`로 초기화되지 않아, 선언문에 도달하기 전까지 TDZ에 놓여 접근하면 `ReferenceError`가 발생하는 것. 블록 안에서 같은 이름의 `let`을 선언하면 선언문 이전에도 바깥 변수가 아니라 안쪽 변수를 참조해 에러가 나는 것이 호이스팅되었다는 근거임**

### 💡 함수 선언문과 함수 표현식의 차이

**⇒ 함수 선언문은 실행 컨텍스트 생성 단계에서 함수 객체까지 통째로 만들어지므로 선언 전에 호출할 수 있지만, 함수 표현식은 변수에 함수를 할당하는 것이라 변수 선언만 호이스팅되고 할당은 런타임에 일어남. 그래서 `var`로 선언한 함수 표현식을 먼저 호출하면 `undefined`를 호출하게 되어 `TypeError`가 나고, `const`로 선언했다면 TDZ로 인해 `ReferenceError`가 남. 함수 표현식을 쓰면 "선언 후 사용" 순서가 강제되어 코드 흐름을 예측하기 쉬움**

---

## 3. Closure

- **클로저(Closure):** 함수가 **자신이 선언될 때의 렉시컬 환경(바깥 스코프)을 기억**해서, 바깥 함수의 실행이 끝난 뒤에도 **그 변수에 접근할 수 있는 현상** (함수 + 선언 당시의 렉시컬 환경)
- JS의 함수는 생성될 때 내부 슬롯 `[[Environment]]`에 **상위 스코프의 참조**를 저장하므로, 상위 함수가 종료되어도 참조가 남아있는 한 그 환경은 **가비지 컬렉션되지 않음**

```js
function createCounter() {
  let count = 0; // 외부에서 직접 접근 불가

  return {
    increase: () => ++count,
    decrease: () => --count,
    getCount: () => count,
  };
}

const counter = createCounter(); // createCounter는 종료되었지만
counter.increase(); // 1
counter.increase(); // 2 → count를 계속 기억
console.log(counter.count); // undefined → 외부에서 직접 수정 불가
```

### 📍 클로저의 활용

| 활용                    | 설명                                                                 |
| ----------------------- | -------------------------------------------------------------------- |
| 정보 은닉 (캡슐화)      | 외부에서 직접 접근할 수 없는 **private 변수**를 만들고 메서드로만 조작 |
| 상태 유지               | 함수 호출 사이에 **상태를 기억** (카운터, 캐시, 메모이제이션)        |
| 함수 팩토리 / 커링      | 일부 인자를 기억한 **새 함수를 생성** (`add(1)(2)`)                   |
| 콜백 / 이벤트 핸들러    | 비동기 콜백이 나중에 실행될 때 **선언 당시의 변수를 참조**            |
| debounce / throttle     | 타이머 ID를 클로저로 기억해 호출을 제어                              |
| React Hooks             | `useState`, `useEffect` 등이 렌더링 시점의 값을 클로저로 기억        |

```js
function debounce(fn, delay) {
  let timer; // 클로저로 유지되는 상태
  return (...args) => {
    clearTimeout(timer);
    timer = setTimeout(() => fn(...args), delay);
  };
}
```

### 📍 반복문과 클로저 (var 문제)

```js
for (var i = 0; i < 3; i++) {
  setTimeout(() => console.log(i), 100);
}
// 3 3 3 → var는 함수 스코프라 모든 콜백이 같은 i를 공유

for (let i = 0; i < 3; i++) {
  setTimeout(() => console.log(i), 100);
}
// 0 1 2 → let은 반복마다 새로운 블록 스코프(새 i)를 생성
```

### 📍 React의 Stale Closure

- 컴포넌트 함수는 렌더링마다 새로 실행되고, 각 렌더링의 함수는 **그 시점의 state를 클로저로 기억**함
- 의존성 배열을 빠뜨리면 **오래된(stale) 값을 계속 참조**하는 버그 발생

```jsx
function Timer() {
  const [count, setCount] = useState(0);

  useEffect(() => {
    const id = setInterval(() => {
      setCount(count + 1); // 항상 첫 렌더링의 count(0)를 참조 → 1에서 멈춤
      // setCount((prev) => prev + 1); // 함수형 업데이트로 해결
    }, 1000);
    return () => clearInterval(id);
  }, []);

  return <div>{count}</div>;
}
```

### 💡 클로저는 메모리 누수를 일으키나요?

**⇒ 클로저는 바깥 함수의 렉시컬 환경을 참조하므로, 클로저 함수가 살아있는 동안 그 환경의 변수들은 가비지 컬렉션되지 않음. 이는 의도된 동작이지만, 큰 데이터를 참조하는 클로저를 전역 변수나 해제하지 않은 이벤트 리스너·타이머에 등록해 두면 필요 없어진 뒤에도 메모리가 계속 남아 누수가 됨. 따라서 사용이 끝난 리스너와 타이머는 해제하고(React에서는 `useEffect`의 cleanup), 필요 없는 참조는 끊어주는 것이 좋음. 참고로 최신 JS 엔진은 클로저가 실제로 사용하는 변수만 유지하도록 최적화함**

### 💡 클로저로 private 변수를 만드는 것과 class의 `#private` 필드의 차이

**⇒ 둘 다 외부에서 직접 접근할 수 없는 상태를 만들지만, 클로저는 인스턴스마다 메서드 함수가 새로 생성되어 메모리를 더 쓰고, `#private` 필드는 메서드를 프로토타입에 공유하면서 언어 차원에서 접근을 막음. ES2022 이전에는 class에 진짜 private이 없어 클로저(모듈 패턴)로 은닉을 구현했지만, 지금은 객체 지향 구조라면 `#private`을, 함수형 스타일이나 간단한 상태 캡슐화에는 클로저를 사용하는 식으로 선택함**

---

## 4. this

- **`this`:** 함수가 **호출될 때** 결정되는, 그 함수를 **호출한 객체(실행 주체)를 가리키는 값**
- 스코프는 정의 위치로 결정되지만(렉시컬), **`this`는 호출 방식에 따라 동적으로 결정**됨 (화살표 함수 제외)

### 📍 this 바인딩 규칙

| 호출 방식                 | 예시                              | `this`                                                  |
| ------------------------- | --------------------------------- | ------------------------------------------------------- |
| 기본 바인딩 (일반 함수 호출) | `foo()`                         | 전역 객체(`window`) / **strict mode에서는 `undefined`** |
| 암시적 바인딩 (메서드 호출) | `obj.foo()`                      | **`.` 앞의 객체** (`obj`)                               |
| 명시적 바인딩             | `foo.call(obj)`, `foo.apply(obj)`, `foo.bind(obj)` | **지정한 객체** (`obj`)                |
| `new` 바인딩 (생성자 호출) | `new Foo()`                      | **새로 생성된 인스턴스**                                |
| 화살표 함수               | `() => this`                      | 자신의 `this`가 없고 **선언된 위치의 상위 스코프 `this`** 사용 |
| 이벤트 핸들러             | `el.addEventListener("click", function () {})` | **이벤트가 바인딩된 요소** (`e.currentTarget`) |

- 우선순위: **`new` > 명시적 바인딩 > 암시적 바인딩 > 기본 바인딩**

```js
const user = {
  name: "seyun",
  greet() {
    console.log(this.name);
  },
};

user.greet(); // "seyun" → 암시적 바인딩

const greet = user.greet;
greet(); // undefined (strict mode면 TypeError) → 객체와의 연결이 끊겨 기본 바인딩

const bound = user.greet.bind({ name: "bound" });
bound(); // "bound" → 명시적 바인딩
```

### 📍 call / apply / bind

| 메서드  | 실행 시점              | 인자 전달 방식           |
| ------- | ---------------------- | ------------------------ |
| `call`  | **즉시 실행**          | 인자를 하나씩 `(this, a, b)` |
| `apply` | **즉시 실행**          | 인자를 배열로 `(this, [a, b])` |
| `bind`  | 실행하지 않고 **`this`가 고정된 새 함수 반환** | 인자를 하나씩 (부분 적용 가능) |

### 📍 화살표 함수와 this

```js
const timer = {
  seconds: 0,
  startNormal() {
    setInterval(function () {
      this.seconds++; // this = 전역 객체(또는 undefined) → 의도와 다름
    }, 1000);
  },
  startArrow() {
    setInterval(() => {
      this.seconds++; // this = timer → 상위 스코프(startArrow)의 this를 그대로 사용
    }, 1000);
  },
};
```

- 화살표 함수는 `this`를 **바인딩하지 않으므로** `call/apply/bind`로도 바꿀 수 없고, `new`로 호출할 수 없음
- 그래서 **객체의 메서드를 화살표 함수로 정의하면** `this`가 객체가 아니라 바깥(전역)을 가리키게 됨

```js
const obj = {
  name: "obj",
  arrow: () => console.log(this.name), // undefined → 객체 리터럴은 스코프를 만들지 않음
};
```

### 💡 콜백으로 메서드를 넘기면 `this`가 사라지는 이유

**⇒ `this`는 함수가 정의될 때가 아니라 호출될 때 `.` 앞에 어떤 객체가 있는지로 결정되기 때문. `button.addEventListener("click", user.greet)`나 `setTimeout(user.greet)`처럼 메서드를 콜백으로 넘기면 함수 참조만 전달되고, 나중에 다른 곳에서 `greet()` 형태로 호출되어 `user`와의 연결이 끊김. 이를 해결하려면 `user.greet.bind(user)`로 `this`를 고정하거나, `() => user.greet()`처럼 화살표 함수로 감싸서 메서드 호출 형태를 유지해야 함**

### 💡 화살표 함수를 쓰면 안 되는 경우

**⇒ 화살표 함수는 자신의 `this`, `arguments`, `prototype`이 없으므로 `this`가 호출 객체를 가리켜야 하는 경우에는 쓰면 안 됨. 대표적으로 객체 리터럴의 메서드, 프로토타입 메서드, 생성자 함수(`new` 불가), 그리고 `this`로 이벤트 대상 요소를 참조해야 하는 이벤트 핸들러가 있음. 반대로 메서드 안의 콜백(`setTimeout`, `map`)처럼 바깥의 `this`를 그대로 쓰고 싶을 때는 화살표 함수가 적합함**

---

## 5. Prototype

- JS는 클래스 기반이 아니라 **프로토타입 기반 언어**로, 객체가 **다른 객체(프로토타입)를 참조해 그 프로퍼티와 메서드를 상속**받음
- 모든 객체는 내부 슬롯 **`[[Prototype]]`** 에 자신의 프로토타입 객체에 대한 참조를 가짐

### 📍 prototype vs `[[Prototype]]` (`__proto__`)

| 구분                         | 소유자                        | 의미                                                              |
| ---------------------------- | ----------------------------- | ----------------------------------------------------------------- |
| `prototype` 프로퍼티         | **생성자 함수** (`function`, `class`) | 이 함수로 `new` 해서 만들어질 **인스턴스의 프로토타입이 될 객체** |
| `[[Prototype]]` (`__proto__`) | **모든 객체**                | 이 객체가 **실제로 상속받고 있는 프로토타입 객체**에 대한 참조     |
| `constructor`                | 프로토타입 객체               | 자신을 `prototype`으로 가지는 **생성자 함수를 다시 가리킴**        |

```js
function Person(name) {
  this.name = name;
}
Person.prototype.greet = function () {
  return `hi, ${this.name}`;
};

const p = new Person("seyun");

Object.getPrototypeOf(p) === Person.prototype; // true (p.__proto__ === Person.prototype)
Person.prototype.constructor === Person; // true
p.greet(); // "hi, seyun" → p에는 greet이 없지만 프로토타입에서 찾음
```

- `__proto__`는 레거시 접근자이므로 **`Object.getPrototypeOf()` / `Object.setPrototypeOf()`** 사용을 권장

### 📍 프로토타입 체인 (Prototype Chain)

- 객체에서 프로퍼티를 찾을 때 **자신 → 프로토타입 → 프로토타입의 프로토타입 → ... → `Object.prototype` → `null`** 순으로 탐색
- 끝까지 없으면 `undefined`

```
p ──[[Prototype]]──▶ Person.prototype ──[[Prototype]]──▶ Object.prototype ──▶ null
{ name }             { greet, constructor }               { toString, hasOwnProperty, ... }
```

```js
p.toString(); // Person.prototype에 없음 → Object.prototype.toString 사용
p.hasOwnProperty("name"); // true  → 인스턴스 자신의 프로퍼티
p.hasOwnProperty("greet"); // false → 프로토타입에서 상속받은 프로퍼티
```

- 스코프 체인은 **식별자(변수)** 를 찾는 경로, 프로토타입 체인은 **객체의 프로퍼티**를 찾는 경로

### 📍 class와 프로토타입

- ES6 `class`는 **프로토타입 기반 상속을 더 쉽게 쓰도록 만든 문법**으로, 내부적으로는 프로토타입으로 동작함

```js
class Animal {
  constructor(name) {
    this.name = name; // 인스턴스 프로퍼티
  }
  speak() {
    return `${this.name} makes a sound`; // Animal.prototype에 저장
  }
}

class Dog extends Animal {
  speak() {
    return `${this.name} barks`;
  }
}

const d = new Dog("coco");
Object.getPrototypeOf(Dog.prototype) === Animal.prototype; // true
```

- 단, class는 단순한 문법 설탕만은 아니고 **`new` 없이 호출 시 에러**, **TDZ 적용**, **strict mode 강제**, `#private` 필드 등 차이가 있음

### 📍 메서드를 프로토타입에 두는 이유

```js
function User(name) {
  this.name = name;
  this.sayHi = function () {}; // 인스턴스마다 새 함수 생성 → 메모리 낭비
}
User.prototype.sayHi = function () {}; // 모든 인스턴스가 하나의 함수를 공유
```

### 💡 프로토타입 체인의 동작 원리를 설명해주세요

**⇒ 객체의 프로퍼티에 접근하면 JS 엔진은 먼저 객체 자신에서 찾고, 없으면 `[[Prototype]]`이 가리키는 프로토타입 객체에서, 그래도 없으면 그 프로토타입의 프로토타입으로 올라가며 `Object.prototype`을 거쳐 `null`에 도달할 때까지 탐색함. 이 덕분에 배열이 `map`, `filter`를 직접 갖고 있지 않아도 `Array.prototype`의 메서드를 사용할 수 있음. 반대로 프로퍼티를 할당할 때는 체인을 따라가지 않고 객체 자신에 새로 만들기 때문에, 같은 이름이면 프로토타입의 프로퍼티를 가리는(섀도잉) 효과가 생김**

### 💡 JS의 class는 다른 언어의 class와 같은가요?

**⇒ 문법은 Java 같은 클래스 기반 언어와 비슷하지만, 내부적으로는 클래스가 설계도로 복사되는 것이 아니라 인스턴스가 `prototype` 객체를 참조하는 프로토타입 체인으로 동작함. 그래서 런타임에 `Dog.prototype`에 메서드를 추가하면 이미 만들어진 인스턴스에도 바로 반영되고, `extends`도 결국 `Dog.prototype`의 `[[Prototype]]`을 `Animal.prototype`으로 연결하는 것임. 다만 `new` 강제, TDZ, `#private` 필드처럼 생성자 함수에는 없는 제약과 기능이 추가되어 있음**

---

## 6. Execution Context

- **실행 컨텍스트(Execution Context):** JS 코드가 실행되기 위해 필요한 **환경 정보(변수, 스코프, `this`)를 담고 있는 객체**
- 지금까지 본 **스코프, 호이스팅, 클로저, `this`가 모두 실행 컨텍스트의 동작으로 설명됨**

### 📍 실행 컨텍스트의 종류

| 종류                         | 생성 시점                        |
| ---------------------------- | -------------------------------- |
| Global Execution Context     | 스크립트가 **처음 실행될 때 1개** 생성 |
| Function Execution Context   | **함수가 호출될 때마다** 생성    |
| Eval Execution Context       | `eval()` 실행 시 생성 (사용 지양) |

### 📍 Call Stack (실행 컨텍스트 스택)

- 실행 컨텍스트는 **콜 스택에 쌓이고(push) 실행이 끝나면 제거(pop)** 됨 → 가장 위에 있는 컨텍스트가 **현재 실행 중인 코드**
- JS는 싱글 스레드라 **콜 스택이 하나**이고, 재귀가 너무 깊으면 `RangeError: Maximum call stack size exceeded` (Stack Overflow)

```js
function first() {
  second();
}
function second() {
  console.log("hi");
}
first();
```

```
           ┌────────────┐
           │  second()  │
┌────────┐ ├────────────┤ ┌────────────┐
│ first()│ │  first()   │ │  first()   │
├────────┤ ├────────────┤ ├────────────┤ ┌────────────┐
│ Global │ │  Global    │ │  Global    │ │  Global    │
└────────┘ └────────────┘ └────────────┘ └────────────┘
 first 호출  second 호출    second 종료    first 종료
```

### 📍 실행 컨텍스트의 구성 요소

| 구성 요소                | 설명                                                                                                    |
| ------------------------ | ------------------------------------------------------------------------------------------------------- |
| LexicalEnvironment       | 식별자와 값을 관리하는 환경. **`let`/`const`/함수 선언** 등을 저장                                      |
| ┗ Environment Record     | 현재 스코프에 선언된 **식별자와 값을 기록**하는 저장소 → **호이스팅**의 원인                             |
| ┗ Outer Environment Reference | **상위(바깥) 렉시컬 환경을 가리키는 참조** → **스코프 체인**과 **클로저**의 원리                   |
| VariableEnvironment      | 생성 시점의 환경 스냅샷. **`var` 선언**을 저장                                                          |
| This Binding             | 이 컨텍스트에서 사용할 **`this` 값** → 함수 호출 방식에 따라 결정                                       |

### 📍 실행 컨텍스트의 생성과 실행 과정

1. **생성 단계 (Creation Phase)**
   - 코드를 훑으며 선언을 Environment Record에 등록 → `var`는 `undefined`로 초기화, `let/const`는 초기화 X(TDZ), **함수 선언문은 함수 객체까지 생성** → **호이스팅**
   - Outer Environment Reference에 **함수가 정의된 위치의 렉시컬 환경** 연결 → **렉시컬 스코프**
   - **`this` 바인딩** 결정
2. **실행 단계 (Execution Phase)**
   - 코드를 한 줄씩 실행하면서 **변수에 값을 할당**하고, 함수 호출을 만나면 **새 실행 컨텍스트를 생성해 콜 스택에 push**
3. **종료**
   - 함수 실행이 끝나면 실행 컨텍스트가 **콜 스택에서 pop**
   - 단, 내부 함수가 이 렉시컬 환경을 참조하고 있으면 **환경은 메모리에 남음** → **클로저**

```js
const x = 1;

function outer() {
  const y = 2;
  return function inner() {
    return x + y; // inner의 Outer Reference → outer의 환경 → 전역 환경
  };
}

const fn = outer(); // outer의 실행 컨텍스트는 pop 되지만, 렉시컬 환경은 inner가 참조 중이라 유지
fn(); // 3
```

### 💡 실행 컨텍스트로 호이스팅과 클로저를 설명해주세요

**⇒ 함수가 호출되면 실행 컨텍스트가 생성되는데, 코드를 실행하기 전 생성 단계에서 해당 스코프의 선언들을 Environment Record에 미리 등록하기 때문에 선언 이전에도 식별자가 존재하는 호이스팅이 일어남. 또한 함수 객체는 생성될 때 자신이 정의된 위치의 렉시컬 환경을 `[[Environment]]`에 저장하고, 호출되면 이를 Outer Environment Reference로 연결함. 바깥 함수의 실행 컨텍스트가 콜 스택에서 제거되어도 내부 함수가 그 렉시컬 환경을 참조하고 있으면 가비지 컬렉션되지 않아 계속 접근할 수 있는데, 이것이 클로저임**

### 💡 콜 스택과 이벤트 루프는 어떤 관계인가요?

**⇒ JS 엔진은 콜 스택 하나로 실행 컨텍스트를 순서대로 처리하는 싱글 스레드 구조라 한 번에 하나의 작업만 실행함. `setTimeout`, `fetch` 같은 비동기 작업은 브라우저의 Web API가 처리하고, 완료된 콜백은 태스크 큐(마이크로태스크 큐)에 대기함. 이벤트 루프는 콜 스택이 비었을 때 큐의 콜백을 꺼내 콜 스택에 올려 실행시키므로, 콜 스택을 오래 점유하는 동기 작업이 있으면 비동기 콜백과 렌더링이 모두 지연됨 (9주차 Event Loop에서 자세히 다룸)**
