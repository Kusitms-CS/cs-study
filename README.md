# 💻 CS 스터디 운영 가이드

## 🎯 스터디 목표

취업 준비 과정에서 자주 접하게 되는 CS 주제를 매주 하나씩 학습하고, 학습한 내용을 꾸준히 기록하는 것을 목표로 합니다.

단순히 내용을 정리하는 것에서 끝나지 않고, 다른 스터디원의 글을 읽고 질문하거나 서로의 생각을 공유하면서 함께 학습합니다.

스터디 전반부에는 **Frontend와 Backend가 함께 공통 CS를 학습**하고, 후반부에는 **각 직군에서 중요하게 다루는 주제를 중심으로 심화 학습**을 진행합니다.

---

# 📚 진행 방식

## 1. 주제 학습 및 PR 작성

매주 스터디장이 해당 회차의 **주제와 필수 학습 키워드**를 공유합니다.

스터디원은 해당 키워드를 중심으로 개별 학습을 진행한 뒤, 학습한 내용을 정리하여 **GitHub Organization에 PR을 작성**합니다.

### 1~7주차

Frontend와 Backend가 동일한 **공통 CS 주제**를 학습합니다.

각 주차에는 공통 학습 키워드와 함께 `Backend / Frontend 선택 심화 키워드`가 제공됩니다.

* **공통 키워드:** 모두 학습
* **직군별 심화 키워드:** 본인의 직군 또는 관심 분야에 따라 자유롭게 선택하여 추가 학습

모든 키워드를 백과사전처럼 정리하기보다는, 기본 개념을 이해한 뒤 **본인이 중요하다고 생각하거나 관심 있는 내용을 조금 더 깊게 학습하는 것을 권장합니다.**

### 8~13주차

Backend와 Frontend가 각 직군에서 중요하게 다루는 주제를 중심으로 **직군별 심화 학습**을 진행합니다.

### 14주차

그동안 학습한 공통 CS와 직군별 심화 내용을 바탕으로 **종합 CS 모의면접**을 진행합니다.

---

## 2. PR 공유

작성한 PR 링크를 해당 회차의 **Notion 페이지에 공유**합니다.

```text
이름 | 직군 | PR 링크 | 리뷰 완료 여부
```

Notion 페이지를 통해 매주 학습 및 리뷰 진행 상황을 함께 확인합니다.

---

## 3. 상호 리뷰 및 질의응답

다른 스터디원이 작성한 PR을 읽어보고 서로 질문과 의견을 나눕니다.

단순히 내용을 확인하는 것에서 끝나지 않도록, **매주 최소 1명의 PR을 리뷰**합니다.

리뷰에서는 아래와 같은 내용을 자유롭게 작성할 수 있습니다.

* 읽으면서 궁금했던 내용
* 이해하기 어려웠던 부분
* 추가로 찾아본 내용
* 다른 관점이나 의견
* 면접에서 나올 것 같은 질문 또는 꼬리질문

공통 CS를 학습하는 **1~7주차에는 Frontend와 Backend가 서로의 PR을 리뷰할 수 있도록 섞어서 진행**하고, 직군별 심화 학습이 시작되는 **8~13주차에는 같은 직군을 중심으로 리뷰**합니다.

---


# 🗓️ 14주 커리큘럼

| 주차      | 구분    | 공통 / 메인 주제                                                                                                      | Backend 심화                                                                                                          | Frontend 심화                                                                                                              |
| ------- | ----- | --------------------------------------------------------------------------------------------------------------- | ------------------------------------------------------------------------------------------------------------------- | ------------------------------------------------------------------------------------------------------------------------ |
| **1주**  | 공통    | **자료구조 / 시간복잡도**<br>Big-O, Array, LinkedList, Stack, Queue, Hash Table, Tree, Heap                              | Java Collection Framework, HashMap 내부 구조, ConcurrentHashMap                                                         | JS Array/Object/Map/Set, 배열 연산의 시간복잡도, 불변성                                                                               |
| **2주**  | 공통    | **OS - Process / Thread / 동시성**<br>Process, Thread, Context Switching, 동시성/병렬성, Race Condition, Mutex/Semaphore | Java Thread, Thread Pool, synchronized, Atomic, WAS Thread                                                        | JS Single Thread, Web Worker, 브라우저 Multi-Process 구조                                                                      |
| **3주**  | 공통    | **OS - Memory / Virtual Memory**<br>Memory 구조, Stack/Heap, Virtual Memory, Paging, Page Fault                   | JVM Runtime Data Area, Heap/Stack, GC                                                                               | JS Memory Heap, Call Stack, Garbage Collection, Memory Leak                                                              |
| **4주**  | 공통    | **Network**<br>OSI 7 Layer/TCP-IP, IP, TCP/UDP, 3-way/4-way Handshake, DNS, Port                                | Socket, Server Connection 관리, Connection Timeout                                                                    | 브라우저 Network 요청, WebSocket, SSE                                                                                          |
| **5주**  | 공통    | **HTTP / HTTPS**<br>HTTP, Stateless, Method, Status Code, HTTP/1.1·2·3, HTTPS, TLS                              | Keep-Alive, REST API, Connection/Timeout                                                                            | HTTP Cache, Fetch/Axios, Resource Loading                                                                                |
| **6주**  | 공통    | **Database**<br>RDB/NoSQL, PK/FK, 정규화, JOIN, Index, B+Tree                                                      | Composite/Covering Index, Query Plan, EXPLAIN, Query 최적화                                                            | Server State, API 데이터 관리, Client Cache와 서버 데이터 정합성                                                                       |
| **7주**  | 공통    | **Web Security / 인증**<br>SOP, CORS, XSS, CSRF, 인증/인가, Cookie/Session, JWT, OAuth                                | Spring Security, JWT 검증, Refresh Token, OAuth 서버 처리                                                                 | Preflight, Cookie 옵션, Token 저장 방식, OAuth Redirect                                                                        |
| **8주**  | 직군 심화 | -                                                                                                               | **Java / JVM**<br>JVM/JDK/JRE, Generic, Collection, Exception, JVM Architecture, Class Loader, GC                   | **JavaScript Core**<br>Scope, Hoisting, Closure, `this`, Prototype, Execution Context                                    |
| **9주**  | 직군 심화 | -                                                                                                               | **Spring Core**<br>IoC/DI, Bean, BeanFactory, ApplicationContext, BeanDefinition, Bean Lifecycle, Proxy             | **비동기 / Event Loop**<br>Call Stack, Web API, Event Loop, Promise, async/await, Microtask/Macrotask                       |
| **10주** | 직군 심화 | -                                                                                                               | **JPA / 영속성 컨텍스트**<br>Entity Lifecycle, 1차 Cache, Dirty Checking, Flush, Lazy/Eager, N+1                            | **Browser / Rendering**<br>DOM, CSSOM, Render Tree, Critical Rendering Path, Layout, Paint, Composite                    |
| **11주** | 직군 심화 | -                                                                                                               | **Transaction / 동시성**<br>ACID, Isolation Level, Lock, Deadlock, 낙관적/비관적 Lock, Transaction Propagation               | **DOM / Rendering 최적화**<br>Reflow, Repaint, Virtual DOM, Reconciliation, React Rendering                                 |
| **12주** | 직군 심화 | -                                                                                                               | **Redis / Cache**<br>Local/Distributed Cache, Redis, TTL, Cache Aside, Cache Invalidation, Distributed Lock         | **Rendering Architecture**<br>SPA/MPA, CSR/SSR/SSG, Hydration, Server/Client Component                                   |
| **13주** | 직군 심화 | -                                                                                                               | **Scale-out / Backend Architecture**<br>Scale-up/out, Load Balancing, Stateless Server, Replication, MQ, Sync/Async | **Web Performance / Cache**<br>Browser Cache, Cache-Control, Lazy Loading, Code Splitting, Web Vitals, Debounce/Throttle |
| **14주** | 종합    | **CS + 직군별 모의면접**                                                                                               | Java / Spring / JPA / DB / Redis / Architecture                                                                     | JavaScript / Browser / React / Rendering / Web Performance                                                               |

---

# 🔍 학습 방향

## 1~7주차: 공통 CS + 선택 심화

Frontend와 Backend가 개발자로서 공통적으로 알아야 하는 CS 기반을 함께 학습합니다.

```text
공통 CS 학습
      ↓
기본 개념 이해
      ↓
직군별 선택 심화
      ↓
PR 작성 및 상호 리뷰
```

직군별 심화 내용은 **필수가 아닌 선택 사항**이며, 본인의 관심 분야에 따라 자유롭게 추가 학습할 수 있습니다.

---

## 8~13주차: 직군별 심화

공통 CS에서 학습한 내용을 각자의 기술 스택과 연결하여 조금 더 깊게 학습합니다.

### Backend

```text
Java / JVM
    ↓
Spring Core
    ↓
JPA
    ↓
Transaction / 동시성
    ↓
Redis / Cache
    ↓
Backend Architecture
```

### Frontend

```text
JavaScript Core
    ↓
Event Loop / 비동기
    ↓
Browser / Rendering
    ↓
Rendering 최적화
    ↓
Rendering Architecture
    ↓
Web Performance / Cache
```

---

## 14주차: 종합 CS 모의면접

마지막 주차에는 새로운 내용을 학습하기보다 그동안 공부한 내용을 바탕으로 **CS + 직군별 기술 모의면접**을 진행합니다.

공통 CS 질문과 직군별 질문을 랜덤으로 선정하고, 실제 기술 면접처럼 답변에 따라 꼬리질문을 이어갑니다.

```text
Q. HTTP가 Stateless하다는 것은 무엇인가요?

→ 그렇다면 로그인 상태는 어떻게 유지할 수 있나요?

→ Session과 JWT의 차이는 무엇인가요?

→ JWT를 사용하면 서버가 완전히 Stateless해질까요?
```

정답을 바로 맞히는 것보다는 **자신이 공부한 내용을 직접 설명하고, 부족한 부분을 확인하는 것**을 목표로 합니다.

---

# 📌 스터디 규칙

스터디는 약 14주 동안 진행되는 만큼 꾸준한 참여를 가장 중요하게 생각합니다.

다만 학업, 인턴, 취업 준비 등 개인 일정이 있을 수 있기 때문에 **전체 기간 동안 최대 3회까지 쉬어갈 수 있습니다.**

휴식하는 주차에는 PR 작성 및 리뷰에 참여하지 않아도 됩니다.

그 외 주차에는 아래 활동에 최대한 꾸준히 참여해주세요.

* 해당 주차 CS 주제 학습
* 학습 내용 정리 및 PR 작성
* 다른 스터디원의 PR 최소 1개 리뷰
* 주간 온라인 스터디 참여

서로의 학습을 강제하기보다는, **14주 동안 함께 꾸준히 공부하고 기록을 남기는 것**을 가장 중요한 목표로 합니다.

> **혼자 하면 미루게 되는 CS 공부를 함께 꾸준히 해봅시다! 🚀**
