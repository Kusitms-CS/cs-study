# DNS

## DNS 란

- **DNS (Domain Name System)** : 호스트의 도메인 네임(`www.example.com`)을 네트워크 주소(`192.168.1.0`)로 변환하거나, 그 반대의 역할을 수행하는 시스템
- naver.com, google.com 같은 도메인 네임(DN)은 사실 **문자열의 탈을 쓴 IP** 임
  - 예 : daum.net → 203.133.167.81, naver.com → 223.130.200.104, google.com → 142.250.207.14
- 원래는 IP 주소를 브라우저에 주면 해당 서버가 홈페이지를 제공하는 식으로 동작함
- 복잡한 숫자 덩어리를 외우기 힘드니, **별명을 지어 전화번호부에 정리**하고 접근하기 쉽게 만든 시스템
- 큰 그림
  1. 브라우저에 도메인 주소 입력 → 도메인 주소를 가진 네임서버(DNS 서버)에 접속
  2. 네임서버가 도메인과 연결된 IP 를 확인해 사용자 PC 에 전달
  3. 사용자 PC 가 전달받은 IP 로 접속
  4. 서버의 내용(홈페이지)을 브라우저에 출력
- 실제로는 전 세계 도메인 수가 너무 많아 **DNS 서버를 계층화해서 단계적으로 처리**함

## 도메인 계층 구조

- 도메인은 루트를 시작점으로 하는 **트리 구조**이며, 뒤쪽 문자열일수록 상위 계층임
- 예 : `blog.example.com.`
  - **Root** : 인터넷 도메인 체계의 시작점. 모든 컴퓨터는 Root DNS 서버의 IP 를 알고 있음
  - **TLD (Top-Level Domain, 1단계 도메인)** : 루트 바로 아래. `.com`, `.net`, `.kr` 등
    - 국가최상위도메인(ccTLD, `.kr` / `.jp`)과 일반최상위도메인(gTLD, `.com` / `.net`)으로 구분
    - `.com` 영리 기업·단체, `.net` 네트워크 관리 기관, `.org` 비영리 기관, `.edu` 미국 4년제 이상 교육기관, `.biz` 사업, `.info` 정보, `.name` 개인 등
    - 도메인을 구입할 때는 1단계 도메인 중 하나를 고르고 원하는 이름을 붙여 등록함
    - 관리 체계 : ICANN 아래에 REGISTRY(gTLD, new gTLD)와 NIC(국가 단위, ccTLD)가 있음
    - 예 : velog.io, github.io 의 `.io` 는 영국령 인도양 지역의 국가 코드 도메인. com/net 이 점유한 이름을 벗어나 새 도메인을 확보할 수 있어 많이 씀
  - **Second-level Domain (2차 도메인)** : `example`, `naver`, `google` 처럼 조직·서비스 이름
  - **Sub Domain (최하위)** : `www.`, `dev.`, `mail.`, `cafe.` 등 한 조직 안의 여러 서비스를 구분함
    - 예 : 네이버 홈 / 메일 / 블로그 / 카페
- 각 계층의 DNS 서버는 **바로 아래 계층을 담당하는 서버 목록과 IP** 를 알고 있음
  - Root → TLD 서버 목록, TLD → 2차 도메인 서버 목록, 2차 도메인 → 서브 도메인 서버 목록
  - 결국 `blog.example.com.` 의 IP 는 서브 도메인을 전담하는 DNS 서버가 알고 있음

### DNS 서버 종류

- **Local DNS (기지국 DNS 서버)**
  - 인터넷을 쓰려면 IP 를 할당해 주는 통신사(KT, SK, LG 등)에 등록하게 되는데, 이때 그 통신사의 DNS 서버가 자동으로 세팅됨 (KT 집이면 KT DNS)
  - 도메인을 입력했을 때 **가장 먼저 물어보는 DNS 서버**
  - 예전에 접속한 적이 있으면 캐싱된 IP 를 바로 돌려줌
- **Root DNS 서버 (루트 네임서버)**
  - 인터넷 도메인 네임 시스템의 루트 존. **ICANN 이 직접 관리**함
  - TLD DNS 서버들의 IP 를 저장해 두고 안내하는 역할. 전 세계에 961개가 운영 중
  - 모든 DNS 서버는 Root DNS 의 주소를 기본으로 갖고 있어서, 모르는 도메인이 오면 가장 먼저 Root 에 물어봄
  - Root : "나한텐 없다. 대신 `.com` 을 관리하는 서버 주소를 알고 있으니 거기에 물어봐라"
- **TLD DNS 서버 (최상위 도메인 서버)**
  - 도메인 등록 기관(Registry)이 관리. `.com`, `co.kr` 같은 도메인을 관리하고 부여함
  - Authoritative DNS 서버 주소를 저장해 두고 안내함
- **Authoritative DNS 서버**
  - **실제 개인 도메인과 IP 주소의 관계가 기록/저장/변경되는 서버**. 그래서 '권한(Authoritative)'이 붙음
  - 일반적으로 도메인/호스팅 업체의 '네임서버'를 말하지만, 개인이나 회사가 직접 DNS 서버를 구축한 경우도 해당함
  - 2차 도메인 서버가 자체적으로 서브 도메인 서버로 요청을 넘기기도 함

## DNS 동작 과정

`www.naver.com` 을 입력했고, Local DNS 에 캐시가 없다고 가정

1. 브라우저에 `www.naver.com` 입력 → PC 에 설정된 **Local DNS(기지국 DNS)** 에 IP 주소 요청
   - 캐시가 있으면 여기서 바로 IP 를 받고 끝남 (1 → 8)
2. Local DNS 가 IP 를 찾기 위해 다른 DNS 서버들과 통신(DNS 쿼리) 시작. 먼저 **Root DNS** 에 요청
3. Root DNS : "찾을 수 없다. 다른 DNS 서버에 물어봐" + `.com` TLD 서버 안내
4. Local DNS 가 `.com` 을 관리하는 **TLD DNS** 에 다시 요청
5. TLD DNS : "찾을 수 없다. 다른 DNS 서버에 물어봐" + `naver.com` Authoritative 서버 안내
6. Local DNS 가 `naver.com` **Authoritative DNS** 에 요청
7. Authoritative DNS 에는 `www.naver.com` 의 IP 가 있음 → "222.122.195.6" 응답
8. Local DNS 는 이 IP 를 **캐싱**해 두고, 단말(PC)에 전달

- 이렇게 Local DNS 가 Root → TLD → Authoritative 순으로 차례대로 요청해 답을 찾아오는 과정을 **재귀적 쿼리(Recursive Query)** 라고 함

## DNS 캐시 / TTL

- 몇 분 후 같은 도메인에 다시 접속할 때마다 위 과정을 반복하면 비효율적임
- 그래서 PC 는 **DNS Cache** 에 자주 쓰는 도메인의 IP 를 저장해 두고, 찾는 과정 없이 바로 IP 를 얻음
  - Windows 에서 확인 : `ipconfig /displaydns`
- **TTL (Time To Live)** : DNS 서버나 PC 캐시(메모리)에 남아 있는 유효 시간
  - 3600초에서 시작해 매초 감소하다가 0이 되면 메모리에서 사라짐
- 주의할 점
  - 캐시로 응답 속도는 빨라지지만, 바이러스·네트워크 오류 등으로 **캐시 정보가 변조**될 수 있음
  - 특정 도메인을 입력했을 때 원래 IP 가 아닌 해킹 사이트 IP 로 연결될 수 있으므로, 주기적으로 캐시를 정리해 줄 필요가 있음

## DNS 레코드 종류

참고 자료 : [DNS 개념 & 동작 완벽 이해 (Inpa Dev)](https://inpa.tistory.com/entry/WEB-%F0%9F%8C%90-DNS-%EA%B0%9C%EB%85%90-%EB%8F%99%EC%9E%91-%EC%99%84%EB%B2%BD-%EC%9D%B4%ED%95%B4-%E2%98%85-%EC%95%8C%EA%B8%B0-%EC%89%BD%EA%B2%8C-%EC%A0%95%EB%A6%AC)
