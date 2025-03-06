# admin 인수인계 (admin_fastapi_mono)

## 개요 

admin page를 만든 시작은 컨텐츠 팀의 노가다 업무를 부분 자동화 해주기 위한 page로 시작했다. nav 요소에 content의 페이지들이 성제님과 희준님 은아님의 반복되는 복붙과 실수요소들을 줄여주기위한 기능 page들이다. 그래서 처음에 project 이름 앞에 TVCF를 붙이지 않았다. 이후 award관리, collection관리, i18n 다국어 관리 도메인이 정식으로 추가되었다.
jinja template를 사용한 SSR을 하는 API Server가 Monolithic 구조를 사용한다고 생각해 뒤에 BE가 아닌 MONO를 붙였다.

idc 143 sever에 https://www.tvcf.co.kr/api/admin/v1/ 으로 접속하는 admin 서버가 배포되어 사용하고 있다. 해당 서버는 live DB에 접근하여 최신 data를 제어하고 있고 컨텐츠 팀과 컨슈머팀에서 업무활용을 위해 사용 중이다.

## 문제

### 코드 구조 문제

다른 API Server들과 다르게 초기부터 후기까지 지속적으로 코드가 추가되고 가장 많이 수정되고 있는 Web Server다. 각 도메인마다 API Dataflow가 구조가 다르게 설계 되어있다. 도메인마다 코드 구조가 조금씩 다르다. 초창기 코드 구조로 시작 해서, 현대적으로 개선된 코드 구조까지 혼재되어 코드 분석에 큰 걸림돌이 되고 있다. 구조가 다르게 설계 되었다는 말은 "코드가 일관적이지 않아 코드 분석이 어렵다" 기본적인 문제 외에 추가적인 것들이 문제가 된다. 오류 처리방법, jinja template 코드들도 함께 개선되며 코드 흐름이 각각 달라 유의해야 할 부분이 많다.

domain으로 분류한 개발 순서는

> content -> collection -> award -> notification -> makerpart


### Collection

jinja template / SSR의 한계를 숙지하지 못한 BE개발자의 무지로 jinja template를 사용자 친화적인 UI를 구성하고 난이도를 높이 끌어올렸다. 과정에서 선호님이 큰 고생을 하고 높은 난이도로 BE 개발 속도도 더뎌 온진히 끝내지 못한 상태로 BE 개발자가 퇴사 했다. 동작은 하고 있지만 다시는 따라하면 안되는 code라고 생각한다.

### jinja template

admin server의 역할이 많아지며 html css 코드도 비례하여 많아졌다. template 코드가 많아지며 생기는 불편함이 있고 main page의 nav 요소에 하위 page들이 다 보이지 않는 일이 생겨 i18n 도메인 분리 필요성을 느끼게 되었다. html jinja template를 최대한 재사용해서 페이지의 일관성도 함께 얻는 개선을 해야 한다.

## 실수/교훈

결국엔 속도 보다는 방향이다. 더 좋은 코드구조와 API 사용 방법이 정해졌을 때 그때그때 제대로 코드들을 함께 업데이트하고 코드들의 일관성을 위해 시간을 할애했어야 했다. 동작은 하고 있지만 엄청난 코드 양과 질 낮은 코드들이 쌓여 코드 수정의 속도를 늦추는 원인이 되었다.

