# 5주차 - HTTP / HTTPS

## 공통 키워드
- HTTP
  : HyperText Transfer Protocol 의 약자로 world wide web 상에서 데이터를 주고받기 위한 프로토콜  
  HTTP는 클라이언트와 서버 간의 요청과 응답을 통해 작동.  
  즉, 다양한 종류의 데이터를 전송할 수 있도록 설계된 프로토콜이다.
  TCP/IP 통신 프로토콜을 기반으로 동작한다.  
  

  
- Stateless
  무상태 라는 뜻으로, 서버가 클라이언트의 상태를 기억하지 않고 매 요청마다 필요한 모든 정보를 클라이언트가 보내는 방식이다.
  왜? 굳이 무상태로 만들어 둘까?(정보를 기억해두면 편하잖아)
  이유는 보안 때문.. 서버가 클라이언트의 정보를 계속해서 들고 있게 되면 보안과 확장성에 있어 제한을 받게 됨.
  1. 확장성
     트래픽이 몰리는 상황이라면 서버에 데이터가 많을수록 분리하다. (즉, 트래픽이 몰릴 때 서버를 여러 대 늘리기 쉽다.)
  2. 
  
- Method
  : 객체지향 프로그래밍 등에서 클래스 내부에 정의되어있는 클래스의 인스턴스. (그 외 관련된 동작 정의)   
  예를 들자면... 아래에서 add가 메서드라고할 수 있다.   
  class Calculator {  
    int add(int num1, int num2) {  
      return num1 + num2;  
     }  
  }  
  
- Status Code  
  흔히 아는 100 ~ 504 까지의 HTTP STATUS CODES.  
  - 1XX -> 요청이 수신되어 처리중인 과정  
      100 -> Continue: 처리가 되었으니 다음으로 진행  
      101 -> Switching Protocol: 서버가 프로토콜을 전환중  
      102 -> Processing: 서버가 요청을 아직 처리중이라 제대로된 응답을 알려줄 수 없음.  
      103 -> Early Hints: 리소스를 사전에 로드하여 로딩을 빠르게 한다.
    
  - 2XX -> 요청 정상 처리  
      201 -> Created: 클라이언트의 요청을 서버가 정상적으로 처리했고 새로운 리소스가 생김.  
      202 -> Accepted: 서버가 아직 처리를 완료하지 못 했지만 일단 ok.  
      203 -> Non-Authoritative Information: 웹사이트가 프록시 서버를 사용할 때 반환되는 상태 코드   
      204 -> No Content: 클라이언트의 요청은 정상적이지만 제공 컨텐츠 x.  
      205 -> Reset Content: 브라우저 새로고침해라.  
      206 -> Partial Content: 리소스 범위의 일부 부분만 반환  
      207 -> Multi-Status: 응답 바디가 여러개 혼합되어 응답됨.  
      208 -> Already Reported: 이미 앞에서 열거됨.  
      218 -> This is fine: 오류가 발생했지만 여긴(apache 서버) 안전하다는 뜻.  
      226 -> IM Used: 서버가 GET 요청에 대한 응답 의무를 다했다는 의미.  
  
  - 3XX -> 요청을 완료하려면 추가 행동이 필요  
      301 -> Moved Permanently: 영구적으로 이동  
      302 -> Found: 다른 URL에서 리소스를 찾음  
      303 -> See Other: 다른 URL에서 리소스를 찾음  
      304 -> Not Modified: 리소스 복사본 상태가 수정되지 않아 최신 상태이므로 캐시 이용.  
      305 -> User Proxy: 리소스가 프록시를 통해서만 액세스될 수 있음 표현. (보안문제로 사용 x)  
      306 -> Switch Proxy / Undefined: 클라이언트가 대체 프록시 사용하도록 리다이렉션 시킴. (보안문제 22)  
      307 -> Temporary Redirect: 임시로 이동  
      308 -> Permanent Redirect: 영구 이동  

  - 4XX -> 클라이언트 오류, 잘못된 문법등으로 서버가 요청을 수행할 수 없음  
     400 -> Bad Request: 클라이언트가 잘못된 요청을 보냄.  
     401 -> Unauthorized: 요청자는 인증되지 않아 수행할 수 없음을 표현.  
     402 -> Payment Required: 나중에 사용될 것을 대비해 예약된 비표준 응답 코드  
     403 -> Forbidden: 요청자는 승인되지 않아 작업을 진행할 수 없음(권한 부족)  
     404 -> Not Found: 클라이언트가 요청한 자원이 존재하지 않음.  
     405 -> Method Not Allowed: 요청이 허용되지 않은 메소드임을 의미.  
     406 -> Not Acceptable: 콘텐츠 협상에 일치하는 것이 없음.  
     407 -> Proxy Authentication Required: 프록시 인증을 요구 (401과 같으나 프록시 버전)  
     408 -> Request Timeout: 요청이 너무 커 타임아웃 됨.  
     409 -> Conflict: 클라이언트의 요청이 서버의 상태와 충돌 발생.  
     410 -> Gone: 리소스가 영구히 삭제됨.  
     411 -> Length Required: 요청 메시지에 Content-Length 헤더 있을 것을 요구.  
     412 -> Precondition Failed: 클라이언트의 조건부 요청 실패.  
     413 -> Payload Too Large: 요청 본문이 서버에서 정의한 한계보다 너무 커 처리 불가.  
    ... 이외 더있으나 생략.  


  - 5XX -> 서버 오류, 서버가 정상 요청을 처리하지 못 함.
     500 ->  Internal Server Error: 서버 내부 문제 발생
     501 -> Not Implemented: 요청에 대해 구현되지 않아 수행하지 아니함
     502 -> Bad Gateway: 게이트웨이가 잘못되어, 서버가 잘못된 응답을 수신함을 의미.
     503 -> Service Unavailable: 서비스 이용 불가 (일시적)
     504 -> Gateway Timeout: 게이트웨이 시간 초과로, 서버에서 요청을 처리하지 않고 닫음.
     505 -> HTTP Version Not Supported: 서버에서 지원되지 않는 http 버전.

- HTTP/1.1·2·3
  


- HTTPS
- TLS

## Backend 심화 (선택)
- Keep-Alive
- REST API
- Connection/Timeout

## Frontend 심화 (선택)
- HTTP Cache
- Fetch/Axios
- Resource Loading
