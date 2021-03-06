# 11장 클라이언트 식별과 쿠키

- 웹 서버는 서로 다른 수천개의 클라이언트들과 통신
- 서버가 통신하는 대상을 식별하는 데 사용하는 기술을 알아보자

## 11.1 개별접촉
- HTTP는 익명으로 사용하며 상태가 없고 요청과 응답으로 통신
- 웹서버는 요청을 보낸 사용자를 식별하거나 방문자가 보낸 연속적인 요청을 추적하기 위해 약간의 정보를 이용할 수 있음
- 현대의 웹 사이트들은 개인화된 서비스를 제공하고 싶어 함

- 개별인사
	- 온라인 쇼핑이 맞춤형으로 느껴지도록 특화된 환영메세지 만들기
- 사용자 맞춤 추천
	- 고객의 흥미를 학습하고 제품을 추천
- 저장된 사용자 정보
	- 주소와 신용카드 정보를 데이터베이스에 저장
- 세션 추적
	- HTTP 트랜잭션은 상태가 없고 사용자가 사이트와 상호작용할 수 있게 사용자의 상태를 남김

## 11.2 HTTP 헤더
- From 헤더
	- 사용자의 이메일 주소를 포함
	- 악의적인 서버가 스팸 메일을 발송하며 악용할 수 있어서 From 헤더를 보내는 브라우저는 많지않음
- User-Agent 헤더
	- 사용자가 쓰고 있는 브라우저의 이름과 버전 정보를 알려줌
	- 특정 브라우저에서 제대로 동작하도록 속성에 맞추어 콘텐츠를 최적화하는 데 유용할 수 있지만, 특정 사용자를 식별하는데는 큰 도움이 되지는 않음
- Referer 헤더
	- 사용자가 현재 페이지로 유입하게 한 웹페이지의 URL을 가리킴
	- 사용자의 웹 사용 행태나 취향을 파악할 수 있음

## 11.3 클라이언트 IP 주소
- 클라이언트의 IP 주소는 보통 HTTP 헤더에 없지만 웹 서버는 HTTP 요청을 보내는 반대쪽 TCP 커넥션의 IP 주소를 알아낼 수 있음
- 아래와 같은 약점을 지님
	- 클라이언트 IP 주소는 사용자가 아닌 사용하는 컴퓨터를 가리킴, 여러 사용자를 식별하기 어려움
	- 인터넷 서비스 제공자는 사용자가 로그인하면 동적으로 IP 주소를 할당, 식별하기 어려움
	- 보안을 강화하고 부족한 주소들을 관리하려고 많은 사용자가 네트워크 주소 변환 방화벽을 통해 인터넷 사용, IP 주소를 방화벽 뒤로 숨기므로 식별 어려움
	- HTTP 프락시와 게이트웨이는 원서버에 새로운 TCP 연결을 함, 웹 서버는 클라이언트의 IP 주소 대신 프락시 서버의 IP 주소를 보게됨
- 이 방식은 잘 사용되지 않음

## 11.4 사용자 로그인
- 사용자 이름과 비밀번호로 인증할 것을 요구해서 사용자에게 식별 요청
- HTTP는 WWW-Authenticate와 Authorization 헤더를 이용해 웹사이트에 사용자 이름을 전달하는 자체적인 체제를 가지고 있음
- 한번 로그인하면 브라우저는 사이트로 보내는 모든 요청에 이 로그인 정보를 함께 보내므로 웹 서버는 그 로그인 정보는 항상 확인할 수 있음
- 서버에서 사용자가 사이트에 접근하기 전에 로그인을 시키고자 한다면 HTTP 401 Login Required 응답 코드를 브라우저에 보낼 수 있음
- 브라우저는 로그인 대화상자를 보여주고, 다음 요청부터 Authorization 헤더에 그 정보를 기술하여 보냄

## 11.5 뚱뚱한 URL
- 사용자의 URL마다 버전을 기술하여 사용자를 식별하고 추적
- 보통 URL은 URL 경로의 처음이나 끝에 어떤 상태 정보를 추가해 확장
- 사용자가 그 사이트를 돌아다닌다면, 웹 서버는 URL에 있는 상태 정보를 유지하는 하이퍼링크를 동적으로 생성
- 사용자의 상태 정보를 포함하고 있는 URL을 뚱뚱한 URL이라고 함
- 사용자가 웹 사이트에 처음 방문하면 유일한 ID가 생성되고, 그 값은 서버가 인식할 수 있는 방식으로 URL에 추가되며, 서버는 클라이언트를 이 뚱뚱한 URL로 리다이렉트 시킴
- 단점
	- 못생긴URL
	- 공유하지 못하는 URL
	- 캐시를 사용할 수 없음
	- 서버 부하 가중
	- 이탈
		- 사용자가 의도치 않게 다른 사이트로 이동하거나 이탈할 수 있음
	- 세션간 지속성의 부재
		- 그 url을 북마킹하지 않는 이상 로그인하면 모든 정보를 잃음

## 11.6 쿠키
### 11.6.1 쿠키의 타입
- 쿠키는 세션쿠키와 지속 쿠키로 나눌 수 있음
- 세션쿠키 : 사용자가 사이트를 탐색할 때, 관련한 설정과 선호 사항들을 저장하는 임시 쿠키
	- 사용자가 브라우저를 닫으면 삭제됨
- 지속쿠키는 삭제되지 않고 유지, 디스크에 저장되어 브라우저를 닫거나 컴퓨터를 재시작하더라도 유지
- 사용자가 주기적으로 방문하는 사이트에 대한 설정 정보나 로그인 이름 유지에 사용
- 둘의 차이는 파기되는 시점

### 11.6.2 쿠키는 어떻게 동작하는가
1. 처음에 사용자가 웹 사이트 방문하면, 웹 서버는 사용자에 대해 아무것도 모름
2. 웹 서버는 사용자가 다시 돌아 왔을 때에 사용자를 식별하기 위한 유일한 값을 쿠키에 할당
	- 임의의 이름=값 ( key=value 형태)의 리스트를 가지고, `Set-Cookie` 같은 HTTP 응답 헤더에 포함
	- 쿠키는 어떤 정보든 포함할 수 있지만, 서버가 사용자 추적용으로 생성한 유일한 단순 식별 번호만 포함하기도 함
3. 브라우저는 서버로 온 `Set-Cookie` 헤더의 쿠키 콘텐츠를 브라우저 쿠키 데이터베이스에 저장
4. 사용자가 미래에 같은 사이트를 방문하면 서버가 사용자에게 할당했던 쿠키를 Cookie 요청헤더에 기술해 전송

### 11.6.3 쿠키 상자: 클라이언트 측 상태
- 쿠키의 기본 발상은 브라우저가 서버 관련 정보를 저장하고 사용자가 해당 서버에 접근할 때마다 그 정보를 함께 전송하게 하는 것
- 브라우저는 쿠키 정보를 저장할 책임이 있고, 이를 '클라이언트 측 상태' 라고 함
- 쿠키 명세에서 이것의 공식적인 이름은 'HTTP 상태 관리 체계(HTTP State Management Mechanism)'임

- 구글 크롬 쿠키
	- 각 브라우저는 각기 다른 방식으로 쿠키 저장
	- Cookies라는 SQLite 파일에 쿠키 저장
	- creation_utc: 쿠키가 생성된 시점
	- host_key: 쿠키의 도메인
	- name: 쿠키의 이름
	- value: 쿠키의 값
	- path: 쿠키와 관련된 도메인에 있는 경로
	- expires_utc: 쿠키의 파기 시점
	- is_secure: SSL 커넥션인 경우에만 보낼지 가리킴

- 마이크로소프트 인터넷 익스플로러 쿠키
	- 캐시 디렉터리에 각각의 개별 파일로 쿠키를 저장
	- 파일에 있는 각 쿠키의 첫번째 줄은 쿠키의 이름
	- 다음줄은 쿠키의 값
	- 세번째 줄에는 도메인과 경로

### 11.6.4 사이트마다 각기 다른 쿠키들
- 브라우저는 보통 각 사이트에 두개 혹은 세개의 쿠키만을 보냄
	- 쿠키를 모두 전달하면 성능이 크게 저하됨
	- 이 쿠키들 대부분은 서버에 특화된 이름/값 쌍을 포함하고 있어서 대부분 사이트에서는 인식하지 않는 무의미한 값
	- 모든 사이트에 쿠키 전체를 전달하는 것은, 특정 사이트에서 제공한 정보를 신뢰하지 않는 사이트에서 가져갈 수 있어서 잠재적인 개인정보 문제를 일으킬 것
- 브라우저는 보통 쿠키를 생성한 서버에게만 쿠키에 담긴 정보를 전달
- 많은 웹 사이트는 광고 협력업체와 계약, 광고들이 웹 사이트의 일부인 거처럼 제작되고 지속쿠키를 만들어 냄
- 같은 광고사에서 제공하는 서로 다른 웹 사이트에 사용자가 방문하면, 브라우저는 앞서 만든 지속 쿠키를 다시 광고사로 서버 전송
- 광고사는 이 기술에 Referer 헤더를 접목하여, 사용자의 프로필과 웹 사이트 사용 습관 데이터를 구축
- 최신 브라우저는 개인정보 설정 기능을 통해 쿠키 사용 방식에 제약을 가할 수 있음

- 쿠키 Domain 속성
	- 서버는 쿠키를 생성할 때 Set-Cookie 응답 헤더에 Domain 속성을 기술해서 어떤 사이트가 그 쿠키를 읽을 수 있는지 제어할 수 있음
- 쿠키 Path 속성
	- 웹 사이트 일부에만 쿠키를 적용할 수도 있음
	- URL 경로의 앞부분을 가리키는 Path 속성을 기술해 해당경로에만 쿠키 전달 가능

###  11.6.5 쿠키 구성 요소
- 현재 사용되는 쿠키 명세에는 version 0 쿠키와 version 1 쿠키가 있음

### 11.6.6 version 0 쿠키
- 최초의 쿠키 명세, set-cookie 응답 헤더와 cookie 요청 헤더와 쿠키를 조작하는 데 필요한 필드들을 정의

### 11.6.7 version 1  쿠키
- version 0 시스템과도 호환되는 확장된 버전

### 11.6.8 쿠키와 세션 추적
- 웹 사이트에 수차례 트랜잭션을 만들어내는 사용자를 추적하는데 사용
1. 브라우저가 루트 페이지를 처음 요청
2. 서버는 클라이언트를 전자상거래 소프트웨어 URL로 리다이렉트
3. 클라이언트는 리다이렉트 URL로 요청 보냄
4. 서버는 응답에 두 개의 세션 쿠키를 기술하고, 사용자를 다른 URL로 리다이렉트, 클라이언트는 다시 이 쿠키들을 첨부하여 요청보냄
5. 클라이언트는 새로운 URL 요청을 앞서 받은 두 개의 쿠키와 함께 전송
6. 서버는 home.html 페이지로 리다이렉트 시키고, 쿠키 두개 더 첨부
7. 클라이언트는 home.html 페이지를 가져오고 총 네개의 쿠키 전달
8. 서버는 콘텐츠를 보냄

### 11.6.9 쿠키와 캐싱
- 쿠키 트랜잭션과 관련된 문서를 캐싱하는 것은 주의
- 사용자의 쿠키가 다른 사용자에게 할당돼버리거나 노출될수도 있는 것
- 지켜야할 기본 원칙
	- 캐시되지 말아야 할 문서가 있다면 표시하기
	- 혹은 해도되는 문서에도 표시해주면 대역폭 절약
	- Set-cookie 헤더를 캐시하는 것에 유의
		- 같은 set-cookie 헤더를 여러 사용자에게 보내면 사용자 추적에 실패할 것이기 때문
		- 어떤 캐시는 응답을 받기전에 위 헤더를 삭제해서 문제 발생할수도
		- 캐시가 모든 요청마다 원 서버와 재검사 시켜 클라이언트로 가는 응답에 Set-coockie 헤더를 기술해서 이 문제를 개선할 수 있음
- Cookie 헤더를 가지고 있는 요청을 주의
	- 요청이 Cookie 헤더와 함께 오면 결과 콘텐츠가 개인정보를 담고있을수도 있다는 힌트
	- 캐시 이미지에 파기시간이 0인 cookie헤더를 설정해서 매번 재검사를 하도록 강제

### 11.6.10 쿠키, 보안, 그리고 개인정보
- 원격 데이터베이스에 개인정보를 저장하고 해당 데이터의 키 값을 쿠키에 저장하는 방식을 표준으로 사용하면, 클라이언트와 서버 사이에 예민한 데이터가 오가는 것을 줄일 수 있음
