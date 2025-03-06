# 인수인계

## 개요

유저 계정이 tvcf에서 일어나는 이슈들을 알림 기록 통해 쉽게 볼 수 있게 해 사용자에게 긍정적인 경험을 제공하기 위한 API Server다. Google firebase token 을 활용한 browser 알림 popup기능을 포함하고 있다. Firebase token을 등록하지 않거나 브라우저에서 push 알림 거부를 하더라도 browser popup 기능을 사용하지 않는 것이지 유저의 알림 메세지가 등록되어 볼 수 있다. (프로필 버튼 옆에 '종'모양 아이콘)
frontend 측에서 notification api 사용은 firebase token 등록, 초대, 개인 메세지. 나머지 API는 backend 측에서 사용하게 된다.

## 성과

### 프로젝트 구조 도입

API Server구축에 hexagonal 구조를 도입했다. 각 계층의 Layer와 API의 Data 흐름이 잘 정돈되어있다. 

### 토큰 사용 구분

frontend 와 backend 에서 동시에 사용하는 성격을 가지고 있어 Cookie token 과 Header token 의 사용 흐름을 고민하게 되었고 token 사용을 구분하는 규칙을 적용하였다.

| 구분 | header token | cookie token |
|-------------|-------------|-------------|
| 호출 | backend 에서 호출 | frontend 에서 호출 |
| url | [Post] /api/notification/v1/ | [Post] /api/notification/v1/users/ |
| 설명1 | url에 users 가 없다  | url에 users 가 있다 |
| 설명2 | 알림서버 관리를 위한 API (admin 에서 사용중) | 유저가 사용하는 API(front에서 사용중) |

### Firebase 알림 token 기능

tvcf-next-fe 프로젝트에서 firebase token을 가져와 Server에 등록하고있다.

### psql 도입

tvcf에 처음 생기는 도메인 API Server로 기존의 여러 기술적 문제를 가지는 MSSQL을 벗어나 psql 도입을 결정했다. psql이 docker 안에서 실행 되고 있어서 psql의 내부 data를 보는게 mssql에 비해 어려워졌다.

## 문제

Hexagonal + Clean architecture 를 적용하면서 코드가 많아지고 규칙이 많아 이해하기 어려워졌다. 하지만 API의 Data flow는 명확하고 보기 쉬워졌다. Docs/ 폴더에 문서를 작성하여 코드 이해도를 높이고 프로젝트 구현하려고 했다. 코드 양이 많아지는 것은 수정할 코드도 많아지지만 GPT에 코드를 던지고 답변 받는 것에도 불리했다.

## 실수/교훈

### 구축 난이도

대표님이 코드를 쉽게 써달라고 하셨지만 제대로 된 아키텍처를 도입해 API Server를 구축하고 싶은 욕심으로 시간을 많이 사용했다. 이후 봄 님에게 인수인계 과정에서 아키텍처의 러닝 커브 이슈로 협업, 내용 전달, 코드 설명에 문제가 생겼고 인수인계를 포기하고 admin, i18n api server의 다른 이슈로 진행하게 했다.

### 협업 문제

API Server 구축과정에서 Test와 더 나은 구축을 위해 영재님과 많은 대화를 나눴다. 협업 과정에서 나의 이전 개발경험, 노하우들과 고정관념들이 현대화된 기술로 극복 가능하다는걸 나중에 알았다. 이후 GPT를 활용한 모든 기술검토에서 "일반적인가? 보편적인가? 상용서비스 레벨에서 사용하는 방법인가?"를 추가 질문을 하게 됐다. 결과로 Frontend 개발자들과 마찰이 줄어들었다.