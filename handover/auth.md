# 인증서버 인수인계 (tvcf_sso_fastapi_be)

## 개요

tvcf의 인증 기능을 담당하는 서버이다. 미션 시작 후 초기에 구축된 서버로 cookie token 인증을 사용하고, user 정보를 반환하는 API를 많이 사용했지만, MSA 구조가 자리를 잡아가며 Frontend 보다 Backend 에서 요청이 많아지고 있어 Header에 담긴 jwt를 인증하는 빈도가 증가하고 있다. cookie token 인증 (v1) 과 header token 인증 (v2) 의 두 가지 인증 방식을 URL 로 구분하려고 했지만 API URL 관리 포인트의 증가로 v2는 거의 호출하지 않고 있다.
기존 API Server에서 사용하는 비지니스 로직은 최대한 동일하게 옮겨 구현했다. 로그인 페이지를 구현한 Frontend 개발자와 기존 로직을 기존 페이지와 동일하게 구현하려고 노력했다.

## 성과

### Ad Consumer Report(ACR) 로그인

현재 ACR에서 이미 local storage 에 session 정보를 담아 Login 기능이 제공되고 있다. 잦은 페이지 전환과 로그인이 풀리는 오류가 있어 불만이 쌓이고 있다. 

새로 구축한 SSO Server를 사용하여 최대한 페이지 전환을 줄이고 로그인 오류를 줄여 구현했다. ACR의 Dotnet 서버에서 tvcf.co.kr에 접근 후 cookie token을 얻어 ACR의 dotnet 서버 내부의 인증 로직을 거치게 했다. 로그인 정상 동작을 확인 후 기능을 숨겨두었다.

### Mypage(React)

기존 sso 서버와 통신하고 구성되는 Mypage는 이미 React로 구현되어있다. react project 빌드 후 실행 코드를 www1.tvcf.co.kr/mypage/ 에 배포하고 있다. 

기존 인증 로직을 걷어내고 신규 인증서버에서 Cookie token을 활용하여 인증 로직을 구현했다.(FE 김영재) dotnet 서버에서 구현된 mypage를 위한 API들을 분석하고 그래도 비지니스 로직을 수행하도록 구현했다. 기존 React Page에서 신규 인증 서버와 달라진 인증 방식으로 임무가 수행된다는 것을 확인 했다. Mypage를 위한 API 서버구축이 완료되면 인증 서버에 임시로 구현한 기능을 삭제해야 한다. 

신규 인증이 적용된 mypage 접근 URL: https://react.tvcf.co.kr/

### SMS 인증

SMS 발송 부분을 구현 하면서 SMS 발송 서비스를 Naver api 서비스로 교체 했다. Test코드와 문서가 현대화 되어있어 사용하기 쉬웠다.

### 동시접근 제한

신규 인증서버를 구축하면서 동시접근 제한 기능을 추가했다. desktop과 mobile 두가지 종류의 로그인에해서 각 1개씩 계정당 두개의 동시 로그인만 허용하게 했다. Redis를 사용하여 동시접근을 제한하고 있다.

## 문제

### 코드 구조

초창기에 구현된 API Server로 코드구조가 좋지 않다.(refac 작업중인 branch가 있다.) 역할이 많은 만큼 코드 양이 많아 시간이 오래걸려 리팩토링을 완성하지 못했다.

### router 이름

sign, register, users 성격과 규칙에 맞는 URL 을 정의해서 사용했어야 했는데 타이밍을 놓쳤다. 이제는 많은 곳에서 사용하고 있어 변경하기 부담스럽다.

### Naver 인증

Naver의 인증 정책의 Redirect 주소 갯수제한(5개)으로 인해 개인 Key를 사용한 인증은 성공 확인 했지만 TVCF 기업 계정으로는 제대로 Test 하지 못했다.

### Email 발송

자체 ASP Email 발송 서비스를 사용하는 것으로 보인다. IDC(143 ip) live Server에서는 정상동작 하지만 개발 Server(es01), Local Server에서는 이메일이 전송되지 않는다. IP 또는 도메인 제한이 있는 것으로 보인다. 개발 환경에서 Test시 주의 필요.

### TVCF DB, TVCF_TMP DB

API Server 중 드물게 MSSQL Server안에 TVCF 와 TVCF_TMP DB에 모두 접근이 필요하다. 구현 내용에는 큰 어려움이 없지만 같은 도메인 내부에서 DB를 다르게 연결하고 제어하기 때문에 코드가 복잡해 지는 문제가 있다. MSSQL을 벗어나기 전엔 개선이 어렵다.

### Google 인증 Key

TVCF의 Google 공식 계정이 없어 Google 인증 API 키를 개인 계정으로 사용했다. TVCF 공식 계정으로 Google 인증키를 교체해야 인수인계나 이슈관리가 원활해질거라 생각한다.

### Redis

docker에서 1개 Redis를 뛰워 인증서버, 알림서버가 함께 사용하고 있다.

### docker compose 

```yaml
services:
  sso:
    build: .
    container_name: tvcf_sso_fastapi
    ports:
      - "50304:8004"
    volumes:
      - .:/app
    networks:
      - default
      - redis_default
    restart: always

  redis:
    container_name: tvcf_sso_redis
    image: redis:latest
    ports:
      - "6379:6379"
    volumes:
      - ./redis/redis-data:/data
      - ./redis/redis-logs:/var/log/redis
      - ./redis/redis.conf:/usr/local/etc/redis/redis.conf
    command: redis-server /usr/local/etc/redis/redis.conf
    networks:
      - redis_default
    restart: always

networks:
  default:
    name: tvcf_sso_fastapi_be
  redis_default:
    external: true
    name: sso_redis_default
```