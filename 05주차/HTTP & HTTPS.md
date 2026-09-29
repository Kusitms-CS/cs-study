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
  
- Status Code
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
