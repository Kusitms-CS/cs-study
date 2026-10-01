# 5주차 참고 자료

## 입문

- [HTTP 개요 (MDN)](https://developer.mozilla.org/ko/docs/Web/HTTP/Guides/Overview)
  - HTTP의 기본 특징(무상태, 확장성, 연결 기반)과 요청/응답 메시지 구조
- [HTTP 요청 메서드 (MDN)](https://developer.mozilla.org/ko/docs/Web/HTTP/Reference/Methods)
  - GET, POST, PUT, PATCH, DELETE 등 메서드별 용도와 안전/멱등/캐시 가능 여부
- [HTTP 상태 코드 (MDN)](https://developer.mozilla.org/ko/docs/Web/HTTP/Reference/Status)
  - 1xx~5xx 상태 코드 분류와 자주 쓰이는 코드별 의미
- [쿠키(Cookie)와 세션(Session)](https://junhyunny.github.io/information/cookie-and-session/)
  - 무상태 프로토콜인 HTTP에서 쿠키와 세션으로 사용자 상태를 유지하는 방법
- [HTTP1.1, HTTP2 비교 (BESPIN Tech Blog)](https://blog.bespinglobal.com/post/http1-1-http2/)
  - 바이너리 프레이밍, 멀티플렉싱, 헤더 압축 등 HTTP/2에서 달라진 점
- [HTTPS와 SSL 인증서 (생활코딩)](https://opentutorials.org/course/228/4894)
  - 대칭키/공개키 암호화, SSL 핸드셰이크, 인증서의 역할 (인증서 발급 실습 부분은 오래된 내용)
- [HTTP Keep-Alive란? Connection을 유지하면 뭐가 좋을까?](https://velog.io/@sin_0/%EB%84%A4%ED%8A%B8%EC%9B%8C%ED%81%AC-HTTP-Keep-Alive%EB%9E%80-Connection%EC%9D%84-%EC%9C%A0%EC%A7%80%ED%95%98%EB%A9%B4-%EB%AD%90%EA%B0%80-%EC%A2%8B%EC%9D%84%EA%B9%8C)
  - 지속 연결의 개념과 장점, Tomcat/Nginx 설정 예시와 timeout 설정 시 주의점
- [클라이언트 요청 타임아웃(request timeout)](https://junhyunny.github.io/information/kind-of-request-timeout/)
  - Connection/Socket/Read Timeout의 차이를 Spring RestTemplate 테스트 코드로 확인
- [REST API가 뭔가요? (얄코)](https://www.yalco.kr/23_rest_api/)
  - REST API의 개념과 HTTP 메서드로 CRUD를 표현하는 URL 설계 예시

## 기본기

- [High Performance Browser Networking](https://hpbn.co/)
  - TCP, TLS, HTTP/1.x, HTTP/2를 지연 시간 관점에서 설명하는 무료 공개 도서
- [RFC 9110 - HTTP Semantics](https://www.rfc-editor.org/rfc/rfc9110)
  - Method의 safe/idempotent 정의와 Status Code의 의미를 규정한 표준 문서

## HTTP/1.1 · 2 · 3

- [HTTP/3는 왜 UDP를 선택한 것일까?](https://evan-moon.github.io/2019/10/08/what-is-http3/)
  - TCP의 3-way handshake 비용과 HOL Blocking 한계, QUIC이 UDP 위에서 이를 해결하는 방식
- [HTTP/3 explained (한국어판)](https://http3-explained.haxx.se/ko)
  - curl 개발자가 쓴 HTTP/3와 QUIC의 동작 원리 해설
- [Comparing HTTP/3 vs. HTTP/2 Performance](https://blog.cloudflare.com/http-3-vs-http-2/)
  - 실측 결과 HTTP/3는 TTFB가 개선되지만 혼잡 제어 차이로 평균 1~4% 느리기도 함
- [LINT: HTTP/2와 TLS를 통한 네트워크 현대화](https://engineering.linecorp.com/ko/blog/LINT-newtork-modernization-http2-tls)
  - LINE이 SPDY를 HTTP/2로 전환하고 TLS 1.3과 세션 재개를 도입한 사례

## HTTPS / TLS

- [A Detailed Look at RFC 8446 (a.k.a. TLS 1.3)](https://blog.cloudflare.com/rfc-8446-aka-tls-1-3/)
  - TLS 1.3이 핸드셰이크를 1-RTT로 줄인 방법과 제거된 구식 암호 기능
- [토스페이먼츠의 Open API 생태계](https://toss.tech/article/42679)
  - 전사 TLS 1.3 적용, JWE 기반 추가 암호화와 재전송 공격 방지 등 API 설계 원칙

## Keep-Alive / Connection / Timeout

- [AWS ALB로부터 반환되는 502 Bad Gateway 에러 트러블슈팅](https://devocean.sk.com/blog/techBoardDetail.do?ID=165428&boardType=techBlog)
  - 서버 keep-alive timeout이 LB idle timeout보다 짧아 발생한 간헐적 502를 해결한 사례
- [HTTP keep-alive, pipelining, multiplexing & connection pooling](https://www.haproxy.com/blog/http-keep-alive-pipelining-multiplexing-and-connection-pooling)
  - 커넥션 재사용과 관련된 네 개념을 프록시 관점에서 비교
- [Timeouts, retries, and backoff with jitter](https://aws.amazon.com/builders-library/timeouts-retries-and-backoff-with-jitter/)
  - 타임아웃 값 설정 기준과 재시도가 장애를 증폭시키는 문제, 지수 백오프와 지터
- [About Pool Sizing (HikariCP)](https://github.com/brettwooldridge/HikariCP/wiki/About-Pool-Sizing)
  - 커넥션 풀은 작을수록 오히려 성능이 좋다는 근거와 풀 크기 산정 공식

## REST API

- [그런 REST API로 괜찮은가 (DEVIEW 2017)](https://slides.com/eungjun/rest)
  - 대부분의 REST API가 Self-descriptive message와 HATEOAS를 만족하지 못한다는 문제 제기
- [REST APIs must be hypertext-driven](https://roy.gbiv.com/untangled/2008/rest-apis-must-be-hypertext-driven)
  - REST 창시자 Roy Fielding이 하이퍼텍스트 기반이 아닌 API는 REST가 아니라고 정리한 원문
- [Implementing Stripe-like Idempotency Keys in Postgres](https://brandur.org/idempotency-keys)
  - 멱등성 키로 POST 요청을 안전하게 재시도하는 API 설계와 구현 방법
