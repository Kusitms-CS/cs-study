# 10주차 - Browser / Rendering

## 브라우저 렌더링 전체 흐름

```text
HTML 파싱 → DOM 생성
CSS 파싱  → CSSOM 생성
              ↓
       Render Tree 생성
              ↓
           Layout
              ↓
            Paint
              ↓
          Composite
```

### 1. DOM 생성

- HTML을 분석해 요소의 구조를 트리로 표현

### 2. CSSOM 생성

- CSS를 분석해 요소에 적용할 스타일 규칙을 트리로 표현

### 3. Render Tree 생성

- DOM과 CSSOM을 결합
- 실제로 화면에 표시할 요소와 스타일 결정
- `display: none`인 요소는 제외

### 4. Layout

- 각 요소의 크기와 화면상 위치 계산

### 5. Paint

- 글자, 배경, 테두리, 그림자 등을 그릴 작업 생성

### 6. Composite

- 여러 레이어를 적절한 순서로 합쳐 최종 화면 표시

### 핵심 정리

- DOM: 어떤 요소가 있는지 표현
- CSSOM: 요소를 어떻게 보여줄지 표현
- DOM과 CSSOM을 결합해야 화면에 그릴 요소와 스타일을 결정할 수 있음

## DOM

- HTML 문서를 자바스크립트가 다룰 수 있도록 객체와 트리 구조로 표현

```text
Document
└─ html
   └─ body
      ├─ h1
      │  └─ "안녕하세요"
      └─ button
         └─ "변경"
```

### Node

- DOM을 구성하는 각각의 항목
- Document Node
- Element Node
- Text Node
- Comment Node

### DOM 조작

```js
const title = document.querySelector("#title");

title.textContent = "반갑습니다";
title.classList.add("active");
```

- 원본 HTML 파일이 아니라 브라우저 메모리의 현재 문서 구조 변경
- 변경 결과에 따라 Render Tree와 화면이 갱신될 수 있음

### DOM 생성

- 브라우저가 HTML을 위에서 아래로 읽으며 생성
- 일반적인 외부 JavaScript를 만나면 다운로드와 실행 중 HTML 파싱이 중단될 수 있음
- DOM 생성 뒤 실행해도 되는 스크립트에는 `defer` 사용 가능

```html
<script defer src="app.js"></script>
```

### 핵심 정리

- HTML: 문서의 원본 문자열
- DOM: 브라우저가 생성한 문서 객체 구조
- JavaScript의 DOM 변경은 서버의 원본 HTML을 변경하지 않음

## CSSOM

- CSS를 브라우저가 계산할 수 있도록 객체와 트리 구조로 표현
- 각 요소에 적용되는 최종 스타일 계산

```css
body {
  color: black;
}

.title {
  color: blue;
  font-size: 24px;
}
```

```text
body: color black
└─ .title
   ├─ color blue
   └─ font-size 24px
```

### 최종 스타일 결정

- 상속
- 선택자 우선순위
- CSS 작성 순서
- 브라우저 기본 스타일

```css
p {
  color: black;
}

.content p {
  color: blue;
}
```

- `.content p`의 선택자 우선순위가 높아 파란색 적용

### 렌더링 차단

- CSSOM 완성 전 화면을 그리면 스타일 변경 후 다시 그려야 할 수 있음
- 브라우저는 일반적으로 CSSOM이 완성될 때까지 Render Tree 생성과 화면 표시를 기다림
- 외부 CSS 다운로드가 느리면 첫 화면 표시도 지연될 수 있음

### 핵심 정리

- CSSOM: CSS 규칙과 최종 스타일을 표현하는 객체 구조
- DOM과 CSSOM을 결합해 Render Tree 생성

## Render Tree

- DOM과 CSSOM을 결합해 실제로 화면에 그릴 요소와 스타일을 표현

```text
DOM + CSSOM → Render Tree
```

### DOM과 차이

- DOM의 모든 요소가 포함되는 것은 아님
- `<head>`, `<meta>`처럼 화면에 표시하지 않는 요소 제외
- `display: none`이 적용된 요소 제외

```html
<p>보이는 글자</p>
<p class="hidden">숨긴 글자</p>
```

```css
.hidden {
  display: none;
}
```

### 숨김 속성 비교

| 속성 | Render Tree | 공간 차지 |
| --- | --- | --- |
| `display: none` | 제외 | X |
| `visibility: hidden` | 포함 | O |
| `opacity: 0` | 포함 | O |

- `visibility: hidden`: 보이지 않지만 Layout 공간 유지
- `opacity: 0`: 투명할 뿐 요소가 존재하며 이벤트 대상이 될 수 있음

### 핵심 정리

- Render Tree는 화면에 필요한 요소만 포함
- `display: none`과 `visibility: hidden`은 렌더링 동작이 다름

## Critical Rendering Path

- 브라우저가 리소스를 받아 첫 화면을 그리기까지 거치는 핵심 과정

```text
HTML → DOM
CSS  → CSSOM
        ↓
   Render Tree
        ↓
      Layout
        ↓
       Paint
        ↓
    Composite
```

- 과정이 짧아질수록 사용자가 화면을 더 빨리 볼 수 있음

### 렌더링을 늦추는 요소

- 큰 HTML과 복잡한 DOM
- 큰 CSS 파일과 사용하지 않는 CSS
- HTML 파싱을 막는 JavaScript
- 너무 많은 외부 리소스 요청
- 늦게 불러오는 글꼴과 핵심 이미지

### JavaScript의 영향

```html
<script defer src="app.js"></script>
```

- HTML 파싱과 동시에 스크립트 다운로드
- DOM 생성 후 스크립트 실행
- HTML 파싱 중단과 첫 화면 표시 지연 감소

### 최적화 방법

- 초기 화면에 필요한 CSS 우선 제공
- 사용하지 않는 CSS와 JavaScript 제거
- 비필수 JavaScript에 `defer` 또는 `async` 사용
- 핵심 리소스만 `preload`
- 이미지 크기 최적화와 지연 로딩
- 지나치게 복잡한 DOM 구조 피하기

### 핵심 정리

- Critical Rendering Path 최적화는 첫 화면 표시 속도 개선
- 렌더링을 막는 리소스를 줄이고 핵심 리소스를 먼저 처리

## Layout

- Render Tree의 각 요소가 화면에서 차지할 크기와 위치를 계산하는 단계

계산하는 정보:

- 너비와 높이
- 화면상의 위치
- Margin과 Padding
- 부모·자식 요소의 배치
- 화면 크기에 따른 반응형 배치

```text
요소 A: x=0, y=0, width=200, height=100
요소 B: x=0, y=100, width=200, height=50
```

### 핵심 정리

- Layout: 요소의 크기와 위치 계산
- 한 요소의 크기가 바뀌면 부모와 주변 요소의 배치도 달라질 수 있음

## Paint

- Layout에서 계산한 위치와 크기를 바탕으로 요소의 시각적인 모양을 그리는 단계

그리는 정보:

- 글자
- 배경색과 이미지
- 테두리
- 그림자
- 색상

```text
Layout: 어디에, 얼마나 크게
Paint: 어떤 색과 모양으로
```

### 성능 영향

- Paint 영역이 크면 그리는 비용 증가
- 복잡한 그림자와 필터가 많으면 비용 증가 가능
- 변경되는 영역과 복잡한 시각 효과를 줄이는 것이 좋음

### 핵심 정리

- Paint: 요소의 글자, 색상, 배경, 테두리 등을 그리는 단계

## Composite

- 여러 Paint 결과를 레이어 단위로 합쳐 최종 화면을 만드는 단계

```text
배경 레이어
버튼 레이어
모달 레이어
   ↓
순서와 위치를 계산해 합성
   ↓
최종 화면
```

- 브라우저는 필요에 따라 요소를 별도의 레이어로 분리
- 각 레이어를 그린 뒤 올바른 순서와 위치로 합성

### Composite만 발생할 수 있는 속성

- `transform`
- `opacity`

```css
.box {
  transform: translateX(100px);
}
```

- 환경과 요소 상태에 따라 Layout과 Paint를 생략하고 Composite만 다시 수행할 수 있음
- 애니메이션에 유리

### 속성별 과정

```text
width 변경
→ Layout → Paint → Composite

background-color 변경
→ Paint → Composite

transform 변경
→ Composite만 수행 가능
```

### 주의

```css
.box {
  will-change: transform;
}
```

- `will-change`로 변경을 미리 알릴 수 있음
- 너무 많은 레이어는 메모리 사용량과 관리 비용 증가
- 실제로 자주 움직이는 요소에만 제한적으로 사용

### 핵심 정리

- Composite: 여러 레이어를 합쳐 최종 화면 생성
- 이동 애니메이션은 `left`보다 `transform`을 사용하면 Layout을 피할 가능성이 높음
