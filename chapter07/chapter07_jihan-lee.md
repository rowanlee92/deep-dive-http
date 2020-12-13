# HTTP 완벽가이드 정리
## 7장 캐시 
### 7.1 불필요한 데이터 전송

복수의 클라이언트가 자주 쓰이는 원 서버 페이지에 접근할 때, 서버는 클라이언트들에게 각각 한 번씩
전송하게 된다. 같은 내용의 데이터 전송이 반복적으로 이루어진다.

이 불필요한 데이터 전송은 네트워크 대역폭을 잡아먹고, 전송을 느리게 하며, 웹 서버에 부하를 준다.
캐시를 사용하면, 첫 번째 서버 응답을 캐시에 보관하여 캐시된 사본이 뒤이은 요청들에 대한 응답으로 사용된다.

---

### 7.2 대역폭 병목

캐시는 네트워크 병목을 줄여준다.
클라이언트들이 서버에 접근하는 속도는, 그 경로에 있는 가장 느린 네트워크의 속도와 같다.
만약, 클라이언트가 빠른 LAN에 있는 캐시로 부터 사본을 가져온다면, 캐싱은 성능을 대폭 개선할 수 있다.

---

### 7.3 갑작스런 요청 쇄도(Flash Crowds)

캐싱은 갑작스런 요청 쇄도에 대처하기 위해 특히 중요하다.

갑작스런 사건(뉴스, 속보, 스팸, 유명인사 관련)으로 인해 많은 사람이 동시에 웹 문서에 접근할 때 이런 일이 발생한다.
이로인한 트래픽 급증은 네트워크와 웹 서버의 심각한 장애를 발생시킨다.

---

### 7.4 거리로 인한 지연

대역폭이 문제가 되지 않더라도 순수한 거리 자체가 문제가 될 수 있다.
빛의 속도 그 자체가 유의미한 지연을 유발한다.

---

### 7.5 적중과 부적중

 캐시에  요청이 도착했을 때, 만약 그에 대응하는 사본이 있다면 그를 이용해 요청이 처리될 수 있다.
 이것을 ```캐시 적중```이라고 부른다. 대응되는 사본이 없다면 원 서버로 전달되는데 이것을 ```캐시 부적중```이라고 한다.

 #### 7.5.1 재검사(Revalidation)

 캐시는 사본이 여전히 최신인지 서버를 통해 때때로 점검해야 한다. 이러한 검사를 ```HTTP 재검사```라 부른다.
 그러나 대부분의 캐시는 그 사본이 검사를 할 필요가 있을 정도로 충분히 오래된 경우에만 재검사를 한다.

HTTP 캐시는 객체를 재확인 하기위해 ```If-Modified-Since```라는 헤더를 가장 많이 사용한다.
캐시된 시간 이후에 변경된 경우에만 사본을 보내달라는 의미가 된다.


* 재검사 적중(느린적중)
  캐시는 사본의 재검사가 필요할 때 재검사 요청을 보낸다. 서버는 304 Not Modified 응답을 보낸다.
  캐시 적중 보다는 느리지만, 캐시 부적중 보다는 빠르다.

* 재검사 부적중
  서버 객체가 캐시된 사본과 다르다면, 서버는 콘텐츠 전체와 함께 평범한 HTTP 200 OK 응답을 클라이언트에게 보낸다.

* 객체 삭제
  서버 객체가 삭제되었다면, 서버는 404 Not Found 응답을 돌려보내며, 캐시 사본을 삭제한다.

#### 7.5.2 적중률

캐시가 요청을 처리하는 비율을 ```캐시 적중률 혹은 문서 적중률``` 이라고 부르기도 한다.

* 0% -> 모든 요청이 캐시 부적중(네트워크 너머로)
* 100% -> 모든 요청이 캐시 적중(캐시에서 사본)

#### 7.5.3 바이트 적중률

바이트 단위 적중률은 캐시를 통해 제공된 모든 바이트의 비율을 표현한다.
바이트 단위 적중률 100%는 모든 바이트가 캐시에서 왔으며, 어떤 트래픽도 인터넷으로 나가지 않았음을 의미한다.

문서 적중률과 바이트 단위 적중률은 둘 다 캐시 성능에 대한 유용한 지표다.

문서 적중률을 개선하면 전체 대기시간(지연)이 줄어든다.
바이트 단위 적중률의 개선은 대역폭 절약을 최적화한다.

#### 7.5.4 적중과 부적중의 구별

응답의 생성일이 더 오래되었다면 클라이언트는 응답이 캐시된 것임을 알아낼 수 있다.
또는, Age 헤더를 확인한다.

---

### 7.6 캐시 토폴로지

* 전용 캐사(private cache)
  * 한 명에게만 할당된 캐시
  * 한 명의 사용자가 자주 찾는 페이지를 담는다.

* 공용캐시(public cache)
  * 사용자 집단에게 자주 쓰이는 페이지를 담는다.

#### 7.6.1 개인 전용 캐시

웹 브라우저는 개인 전용 캐시를 내장하고 있다. 자주 쓰이는 문서를 개인용 컴퓨터의 디스크와
메모리에 캐시해놓고, 사용자가 캐시 사이즈와 설정을 수정할 수 있도록 허용한다.

#### 7.6.2 공용 프락시 캐시

캐시 프락시 서버 혹은 ```프락시 캐시```라고 불리는 특별한 종류의 ```공유된 프락시 서버```다.
프락시 캐시는 로컬 캐시에서 문서를 제공하거나, 혹은 사용자의 입장에서 서버에 접근한다.
공용 캐시에는 여러 사용자가 접근하기 때문에, 불필요한 트래픽을 줄일 수 있다.

#### 7.6.3 프락시 캐시 계층들

작은 캐시에서 캐시 부적중이 발생했을 때 더 큰 부모 캐시가 걸러 남겨진 트래픽을 처리하도록 하는 계층을
만드는 방식이 합리적인 경우가 많다.

캐시 계층이 깊다면 요청은 캐시의 긴 연쇄를 따라가게 된다. 프락시 연쇄가 길어질수록 각 중간 프락시는
현저한 성능 저하가 발생할 것이다.

#### 7.6.4 캐시망, 콘텐츠 라우팅, 피어링

몇몇 네트워크 아키텍처는 단순한 캐시 계층 대신 복잡한 캐시망을 만든다.

캐시망 안에서의 콘텐츠 라우팅을 위해 설계된 캐시들은 아래의 일들을 모두 할 수 있다.

* URL에 근거하여, 부모 캐시 / 원 서버 중 하나를 동적으로 선택
* URL에 근거하여 특정 부모 캐시를 도적으로 선택
* 부모 캐시에 가기전에, 캐시된 사본을 로컬에서 찾기
* 다른 캐시들이 그들의 캐시된 콘텐츠에 부분적으로 접근할 수 있도록 허용하되, 그들의 캐시를 통한
  트랜짓은 허용하지 않음

복잡한 캐시 사이의 관계는, 서로 다른 조직들이 상호 이득을 위해 그들의 캐시를 연결하여 서로를
찾아볼 수 있도록 해준다. ```선택적인 피어링을 지원하는 캐시는 형제 캐시```라고 불린다.

HTTP는 형제 캐시를 지원하지 않기 때문에 인터넷 캐시 프로토콜(ICP)나 하이퍼텍스트 캐시 프로토콜(HTCP)를
이용해 HTTP를 확장한다.

---

### 7.7 캐시 처리 단계

#### 7.7.1 단계 1: 요청 받기

캐시는 네트워크 커넥션에서의 활동을 감지하고, 들어오는 데이터를 읽어들인다.

#### 7.7.2 단계 2: 파싱

요청 메시지를 여러 부분으로 파싱하여 헤더 부분을 조작하기 쉬운 자료 구조에 담는다.

#### 7.7.3 단계 3: 검색

캐시는 URL을 알아내고 그에 해당하는 로컬 사본이 있는지 검색한다.
캐시된 객체는 서버 응답 본문과 원 서버 응답 헤더를 포함하고 있으므로, 캐시 적중 동안 올바른 서버 헤더가 반환될 수 있다.

또한, 얼마나 오랫동안 캐시 되었는지, 얼마나 자주 사용되었는지 등에 대한 몇몇 메타 데이터를 포함한다.

#### 7.7.4 단계 4: 신선도 검사

HTTP는 캐시가 일정 기간 동안 서버 문서의 사본을 보유할 수 있도록 해준다.
신선도 한계를 넘을 정도로 오래 갖고 있었다면, 그 객체는 신선하지 않은 것으로 간주되며
변경이 있었는지 검사하기 위해 서버와 재검사를 해야한다.

#### 7.7.5 단계 5: 응답 생성

캐시는 캐시된 서버 응답 헤더를 토대로 응답 헤더를 생성한다.
또한, 캐시는 신선도 정보를 삽입하며(Cache-Control, Age, Expires 헤더), 또 요청이 프락시
캐시를 거쳐갔음을 알려주기 위해 종종 Via 헤더를 포함시킨다.

```캐시가 Date 헤더를 조정해서는 안 된다는 것에 주의해야함!```

#### 7.7.6 단계 6: 전송

응답 헤더가 준비되면, 캐시는 응답을 클라이언트에게 돌려준다.

#### 7.7.7 단계 7: 로깅

대부분의 캐시는 로그 파일과 캐시 사용에 대한 통계를 유지한다.
가장 많이 쓰이는 캐시 로그 포맷은 **스퀴드 로그 포맷**과 **넷스케이프 확장 공용 로그 포맷**이지만,
많은 캐시 제품이 커스텀 로그 파일을 허용한다.

---

### 7.8 사본을 신선하게 유지하기




  

                                                                                                                                                                                                                                                                                                                                                                                   