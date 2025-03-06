# 인수인계

## API Server 와 github branch 동기화 규칙

### 개요

API Server와 Git Branch의  동기화 규칙을 준수하여 각 서버에 적용된 소스코드 관련 실수가 발생하지 않게 했습니다.

1. 배포 버전

Live Server (idc 143 server) - 모든 repo의 main branch

2. 개발 버전

Dev Server (es01 23 server) - 모든 repo의 dev branch

3. 로컬 버전

나의 개발 PC - 모든 repo의 기능 branch (feature, fix, refac, hotfix 등등)


### 상황

개발자가 소수일 때는 큰 문제가 없었습니다. 프론트 개발자가 합류하고 협업 수준이 깊어지고 mypage 코드를 새 인증서버 연결을 위해 dev branch 에서 작업하고 기능 연결을 위해 feature branch 에서 동시에 개발이 작업이 진행됐습니다. 완료 후 개발 서버에 코드를 반영하면서 코드 충돌이 개발 서버에서 발생하고 충돌을 해결하기 위해 mypage 서버를 구성한지는 얼마 되지 않았지만 충돌작업을 직접 개발 server 에서 하고 있는게 부적절해 보였습니다. 

### 개선

개발 server 에서 직접 코드 수정을 최대한 줄이고자 "API Server 와 github branch 동기화 규칙"을 만들어 지키기로 했고 이 후 개발서버에서 잘못된 브랜치 연결 때문에 다른 코드를 실행하고 test 한다던지, dev 서버에서 오랬동안 코드 작업을 한다던지 하는 위험 요소들을 줄일 수 있었습니다.
