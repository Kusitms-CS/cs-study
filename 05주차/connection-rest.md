# 커넥션과 API 설계 (Backend 심화)

## Keep-Alive

### 동작 방식

- HTTP는 원래 비연결성이라 요청 하나마다 TCP 연결을 새로 맺고 끊었음 (HTTP/1.0)
  - 3way handshake가 요청마다 반복되니 느리고, 서버도 연결 생성/해제에 자원을 계속 씀
- 이걸 해결하려고 한 번 맺은 TCP 연결을 끊지 않고 여러 요청/응답에 재사용하는 것이 Keep-Alive (Persistent Connection)
  - HTTP/1.1부터는 기본값임. 끊고 싶으면 `Connection: close` 헤더를 명시해야 한다.
  - HTTP/1.0에서는 `Connection: keep-alive` 헤더를 직접 보내야 유지됨
- 서버는 `Keep-Alive: timeout=5, max=100` 같은 식으로 조건을 알려줄 수 있음
  - timeout : 이 시간 동안 다음 요청이 없으면 연결을 끊는다 (idle 기준)
  - max : 이 연결로 처리할 최대 요청 수
- 정리하면..

1. 클라가 TCP 연결 + 첫 요청
2. 서버가 응답하고 연결을 끊지 않고 대기
3. 클라가 같은 연결로 다음 요청을 보냄 (handshake 생략)
4. timeout 동안 요청이 없거나 max에 도달하면 연결 종료

- 흠...
  - 연결을 유지하는 동안 서버는 그 커넥션에 스레드/소켓 자원을 묶어두고 있음
  - 그래서 timeout을 무한정 길게 잡으면 놀고 있는 커넥션이 쌓여서 오히려 자원이 고갈될 수 있다
  - Keep-Alive는 연결을 재사용하는 거고, 요청을 동시에 여러 개 보내는 건 아님. 한 연결 안에서는 여전히 요청 - 응답 - 요청 - 응답 순서로 처리됨 (이 한계는 HTTP/2 멀티플렉싱에서 다룸)

### Keep-Alive timeout과 로드밸런서

- 실무에서는 클라 - 로드밸런서(LB) - 서버 구조가 대부분이고, 이때 Keep-Alive timeout이 두 구간에 각각 존재함
  - 클라 ↔ LB 구간의 idle timeout
  - LB ↔ 서버 구간의 idle timeout
- 문제는 서버의 Keep-Alive timeout이 LB의 idle timeout보다 짧을 때 발생함

1. LB가 서버와 연결을 맺고 요청을 보냄
2. 한동안 요청이 없어서 서버가 자기 timeout(예: 5초)에 걸려 연결을 끊음
3. LB는 자기 timeout(예: 60초)이 아직 안 지났으니 그 연결이 살아있다고 생각함
4. 클라 요청이 들어오면 LB가 이미 죽은 연결로 요청을 보냄
5. 서버 쪽에서 RST가 날아오고 LB는 클라에게 502 Bad Gateway를 돌려줌

- 그래서 서버의 Keep-Alive timeout은 LB의 idle timeout보다 항상 길게 잡아야 한다
  - 연결을 먼저 끊는 쪽이 항상 클라에 가까운 쪽(LB)이 되도록 맞추는 게 원칙
  - AWS ALB 기본 idle timeout은 60초. Tomcat, Nginx 등 서버 쪽 keepAliveTimeout이 이보다 짧으면 간헐적 502가 난다
- 간헐적으로만 나는 이유는, 정확히 "서버가 끊은 직후 ~ LB가 끊기 전" 구간에 요청이 들어와야 재현되기 때문



## Connection/Timeout

### 타임아웃 종류

[https://junhyunny.github.io/information/kind-of-request-timeout/](https://junhyunny.github.io/information/kind-of-request-timeout/)

- 타임아웃은 하나가 아니고, 요청의 어느 단계에서 기다리는지에 따라 종류가 나뉨
- Connection Timeout
  - TCP 연결(3way handshake)이 성립되기까지 기다리는 시간
  - 서버가 아예 안 떠있거나 네트워크가 막혀있을 때 걸림
- Socket Timeout (Read Timeout)
  - 연결은 됐는데 서버로부터 데이터가 오기까지 기다리는 시간
  - 정확히는 패킷과 패킷 사이의 간격 기준임. 응답 전체 시간이 아니라 "다음 바이트가 올 때까지" 기다리는 시간
  - 서버가 느리거나 처리 중에 멈춰있을 때 걸림
- Request Timeout (전체 타임아웃)
  - 요청 시작부터 응답 완료까지의 전체 시간 제한
  - 클라 라이브러리마다 지원 여부가 다름
- 흠...
  - Connection Timeout은 짧게(1~3초), Read Timeout은 서버 처리 시간을 고려해서 그보다 길게 잡는 게 보통
  - 타임아웃을 안 걸면 장애 서버를 무한정 기다리면서 호출한 쪽 스레드까지 전부 묶여버림. 장애가 전파되는 대표적인 경로

### 커넥션 풀

- 연결을 매번 만들지 않고 미리 만들어둔 커넥션을 빌려 쓰고 반납하는 방식
  - HTTP 클라(RestTemplate, WebClient 등)도, DB(HikariCP)도 같은 개념임
  - Keep-Alive로 유지된 연결을 재사용하는 걸 클라 쪽에서 체계적으로 관리하는 것
- 풀 크기는 크면 좋을 것 같지만 그렇지 않음

[https://github.com/brettwooldridge/HikariCP/wiki/About-Pool-Sizing](https://github.com/brettwooldridge/HikariCP/wiki/About-Pool-Sizing)

- HikariCP 쪽 설명을 정리하면..
  - 실제 동시에 일할 수 있는 건 CPU 코어 수만큼임. 커넥션이 많아도 결국 코어를 나눠 쓰면서 컨텍스트 스위칭만 늘어남
  - 그래서 커넥션 풀은 작을수록 오히려 빠른 경우가 많다
  - 권장 공식 : `connections = (core_count * 2) + effective_spindle_count`
- 풀이 꽉 차면 다음 요청은 커넥션을 얻기 위해 대기함. 이 대기 시간에도 타임아웃(connection-timeout, pool timeout)이 따로 있음
- 풀에 있는 커넥션이 서버 쪽에서 이미 끊긴 상태일 수 있으므로, 꺼내기 전에 유효성 검사를 하거나 max-lifetime을 서버 timeout보다 짧게 잡는다
  - 위의 LB 502 문제와 똑같은 구조임. 먼저 끊는 쪽을 클라(풀)로 맞추는 것

### 재시도와 멱등성

[https://aws.amazon.com/builders-library/timeouts-retries-and-backoff-with-jitter/](https://aws.amazon.com/builders-library/timeouts-retries-and-backoff-with-jitter/)

- 타임아웃이 났을 때 그냥 다시 보내면 될까?
  - 서버가 과부하라서 느린 상황이면, 재시도는 과부하 서버에 요청을 더 얹는 거라 장애를 증폭시킴
  - 모든 클라가 동시에 재시도하면 요청이 한 순간에 몰림
- 그래서 재시도에는 두 가지가 같이 붙음
  - 지수 백오프 (Exponential Backoff) : 재시도 간격을 1초, 2초, 4초.. 식으로 늘림
  - 지터 (Jitter) : 간격에 랜덤 값을 섞어서 클라들의 재시도 시점을 흩어놓음
  - 재시도 횟수 제한은 당연히 필요
- 그리고 재시도해도 괜찮은 요청인지를 먼저 봐야 함. 여기서 멱등성이 나온다
  - GET, PUT, DELETE는 멱등이라 재시도해도 결과가 같음
  - POST는 멱등이 아니라서, 타임아웃이 났는데 실제로는 서버가 처리를 끝낸 상태였다면 재시도 시 결제가 두 번 되는 식의 문제가 생김
- POST를 안전하게 재시도하려면 멱등성 키 (Idempotency Key)를 쓴다

[https://brandur.org/idempotency-keys](https://brandur.org/idempotency-keys)

1. 클라가 요청마다 고유한 키를 생성해서 헤더에 담아 보냄 (`Idempotency-Key: uuid`)
2. 서버는 키를 저장해두고, 처리 결과도 키와 함께 저장
3. 같은 키로 요청이 다시 오면 처리하지 않고 저장된 결과를 그대로 응답
4. 클라는 타임아웃 시 같은 키로 안심하고 재시도

- 흠...
  - 결국 "서버에서 멱등하지 않은 동작을 멱등하게 만들어주는 장치"임. 토스, Stripe 같은 결제 API가 대표적으로 쓰고 있음
  - 키의 저장 기간, 같은 키인데 바디가 다른 경우의 처리 등은 서버가 정책으로 정해야 함



&nbsp;

## REST API

### REST 제약 조건

- REST, Representational State Transfer. Roy Fielding이 2000년 박사 논문에서 정리한 아키텍처 스타일임
  - 프로토콜이나 표준이 아니고 "이런 제약을 지키면 웹처럼 잘 확장되는 시스템이 된다"는 조건들의 묶음
- 제약 조건 6개
  - Client - Server : 역할 분리
  - Stateless : 서버가 클라 상태를 저장하지 않음. 요청에 필요한 정보가 모두 담겨있어야 함
  - Cache : 응답은 캐시 가능 여부를 명시해야 함
  - Layered System : 클라는 중간에 프록시, LB가 있는지 몰라도 됨
  - Code on Demand (선택) : 서버가 클라에 실행 가능한 코드를 보낼 수 있음 (JS 같은)
  - Uniform Interface : 아래 4개로 다시 나뉨
- Uniform Interface
  - 리소스 식별 : URI로 리소스를 식별
  - 표현을 통한 리소스 조작 : JSON, XML 같은 표현으로 리소스를 다룸
  - Self-descriptive message : 메시지만 보고 해석이 가능해야 함
  - HATEOAS : 응답에 다음에 할 수 있는 행동(링크)이 들어있어야 함. 하이퍼링크로 상태가 전이된다는 의미

[https://slides.com/eungjun/rest](https://slides.com/eungjun/rest)

- 흠...
  - 앞의 조건들은 HTTP를 쓰면 거의 자연스럽게 지켜지는데, 마지막 두 개(Self-descriptive, HATEOAS)는 대부분의 API가 안 지킴
  - 그래서 Fielding은 "하이퍼텍스트 기반이 아니면 REST API라고 부르지 말라"고 했음
  - 우리가 평소에 REST API라고 부르는 건 엄밀히는 대부분 HTTP API임 (아래에서 정리)

### REST API 설계 규칙

- 실무에서 "RESTful하다"고 할 때 보통 이야기하는 관례들
- URI는 리소스(명사)를 표현하고, 행위는 HTTP 메서드로 표현한다
  - `GET /members/1` O / `GET /getMember?id=1` X
  - 메서드 - CRUD 매핑 : POST 생성 / GET 조회 / PUT 전체 수정 / PATCH 일부 수정 / DELETE 삭제
- 컬렉션은 복수형 명사, 개별 리소스는 그 하위에 식별자
  - `/members` → 목록, `/members/1` → 1번 회원
  - 관계가 있으면 계층으로 : `/members/1/orders`
- URI 규칙
  - 소문자, 단어 구분은 하이픈(-), 마지막에 슬래시 붙이지 않음
  - 파일 확장자 쓰지 않음. 포맷은 `Accept` 헤더로 협상
- 상태 코드를 의미에 맞게 쓴다
  - 생성 성공은 201 + `Location` 헤더, 삭제 성공은 204, 잘못된 요청은 400, 없는 리소스는 404
  - 전부 200으로 주고 바디에 `success: false` 넣는 식은 HTTP의 의미를 버리는 것임
- 필터링, 정렬, 페이징은 쿼리 파라미터로
  - `/members?status=active&sort=name&page=2`
- 버전 관리는 `/v1/members`처럼 URI에 넣거나 헤더로 처리함. 정답은 없고 일관성이 중요

### REST API와 HTTP API

- 위에서 봤듯이 REST의 제약을 전부 만족하는 API는 드물고, 대부분은 "HTTP와 JSON을 잘 활용한 API"임
  - 이걸 HTTP API 혹은 Web API라고 부르는 게 정확함
  - 그래도 업계에서는 그냥 REST API라고 부르고 있고, 이미 용어가 굳어져서 굳이 고쳐 부르진 않는 분위기
- 그러면 HATEOAS까지 지켜야 하나?
  - 장점 : 클라가 URI를 하드코딩하지 않고 응답의 링크를 따라가므로, 서버가 URI를 바꿔도 클라가 깨지지 않음 (웹 브라우저가 그렇게 동작함)
  - 현실 : 클라가 보통 우리가 직접 만드는 프론트/앱이라서, API 변경 시 클라도 같이 배포하면 됨. 링크를 따라가는 범용 클라가 없으니 효용이 작음
  - 그래서 공개 API, 여러 클라가 붙는 API에서는 고려할 가치가 있고, 내부 API에서는 보통 안 함
- 정리하면..
  - REST API : Fielding의 제약 조건을 모두 만족. Self-descriptive + HATEOAS 포함
  - HTTP API : HTTP 메서드, 상태 코드, URI 규칙을 의미에 맞게 쓰는 API. 우리가 보통 만드는 것
  - 중요한 건 이름보다, HTTP의 의미(메서드/상태 코드/헤더)를 버리지 않고 일관되게 설계하는 것
