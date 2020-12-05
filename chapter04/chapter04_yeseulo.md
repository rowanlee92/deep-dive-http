# 04. 커넥션 관리

이번 장에서 배울 내용은
* HTTP는 어떻게 TCP 커넥션을 사용하는가
* TCP 커넥션의 지연, 병목, 막힘
* 병렬 커넥션, keep-alive 커넥션, 커넥션 파이프라인을 활용한 HTTP의 최적화
* 커넥션 관리를 위해 따라야 할 규칙들

## 4.1 TCP 커넥션

* 전 세계 모든 HTTP 통신은, 지구상의 컴퓨터와 네트워크 장비에서 널리 쓰이고 있는 패킷 교환 네트워크 프로토콜들의 계층화된 집합인 TCP/IP를 통해 이루어진다.
* 커넥션이 맺어지면 클라이언트와 서버 컴퓨터 간 주고받는 메시지들은 손실/손상되거나 순서가 바뀌지 않고 안전하게 전달된다.
* 웹 브라우저가 TCP 커넥션을 통해 웹 서버에 요청을 보내는 과정
  ```
  1. 브라우저가 yeseulo.kr이라는 호스트명을 추출한다.
  2. 브라우저가 이 호스트 명에 대한 IP주소를 찾는다.
  3. 브라우저가 포트 번호(80)를 얻는다.
  4. 브라우저가 202.43.78.3의 80포트로 TCP 커넥션을 생성한다.
  5. 브라우저가 서버로 HTTP GET 요청 메시지를 보낸다.
  6. 브라우저가 서버에서 온 HTTP 응답 메시지를 읽는다.
  7. 브라우저가 커넥션을 끊는다.
  ```

### 4.1.1 신뢰할 수 있는 데이터 전송 통로인 TCP

* HTTP 커넥션은 몇몇 사용 규칙을 제외하고 TCP 커넥션에 불과하다.
* TCP 커넥션은 인터넷을 안정적으로 연결해주며, TCP는 HTTP에게 신뢰할 만한 통신 방식을 제공한다.
* TCP 커넥션의 한쪽에 있는 바이트들은 반대쪽으로 순서에 맞게 정확히 전달된다.

### 4.1.2 TCP 스트림은 세그먼트로 나뉘어 IP 패킷을 통해 전송된다

* HTTP는 'IP, TCP, HTTP'로 구성된 '프로토콜 스택'에서 최상위 계층이다.
* HTTPS는 HTTP에 보안 기능을 더한 것으로, TLS 혹은 SSL이라 블리기도 하며, HTTP와 TCP 사이에 있는 암호화(cryptographic encryption) 계층이다.
* HTTP 네트워크 프로토콜 스택
  ```
    HTTP              --- 애플리케이션 계층
    TCP               --- 전송 계층
    IP                --- 네트워크 계층
    Network Interface --- 데이터 링크 계층
  ```
* HTTPS 네트워크 프로토콜 스택
  ```
    HTTP              --- 애플리케이션 계층
    TLS or SSL        --- 보안 계층
    TCP               --- 전송 계층
    IP                --- 네트워크 계층
    Network Interface --- 데이터 링크 계층
  ```
* TCP는 세그먼트라는 단위로 데이터 스트림을 잘게 나누고, 세그먼트를 IP 패킷(혹은 IP 데이터그램)이라 불리는 봉투에 담아 인터넷을 통해 데이터를 전달한다.
* 이 모든 것은 TCP/IP 소프트웨어에 의해 처리되며 그 과정은 보이지 않는다.
* 각 TCP 세그먼트는 하나의 IP 주소에서 다른 IP 주소로 IP 패킷에 담겨 전달되며, IP 패킷은 다음을 포함한다.
  * IP 패킷 헤더(보통 20byte): IP 헤더는 발신자와 목적지 IP 주소, 크기, 기타 플래그를 가진다.
  * TCP 세그먼트 헤더(보통 20byte): TCP 포트번호, TCP 제어 플래그, 데이터의 순서와 무결성을 검사하기 위해 사용되는 숫자 값을 포함한다.
  * TCP 데이터 조각(0 혹은 그 이상의 byte)

### 4.1.3 TCP 커넥션 유지하기

* 컴퓨터는 항상 TCP 커넥션을 여러 개 가지고 있으며, TCP는 포트 번호를 통해 이런 여러 개의 커넥션을 항상 유지한다.
* TCP 커넥션은 네 가지 값으로 식별한다. `<발신지 IP 주소, 발신지 포트, 수신지 IP 주소, 수신지 포트>`
  * 이 네 가지 값으로 유일한 커넥션을 생성한다. 서로 다른 두 개의 TCP 커넥션은 네 가지 주소 구성요소 값이 모두 같을 수 없다. (주소 구성요소 일부가 같을 수는 있다.)

### 4.1.4 TCP 소켓 프로그래밍

* 운영체제는 TCP 커넥션의 생성과 관련된 여러 기능을 제공한다.
* TCP 커넥션 프로그래밍을 위한 공통 소켓 인터페이스 함수들
  * `s = socket(<parameters>)` : 연결이 되지 않는 익명의 새로운 소켓 생성
  * `bind(s, <local IP:port>)` : 소켓에 로컬 포트 번호와 인터페이스 할당
  * `connect(s, <remote IP:port>)` : 로컬의 소켓과 원격의 호스트 및 포트 사이에서 TCP 커넥션 생성
  * `listen(s,...)` : 커넥션을 받아들이기 위해 로컬 소켓에 허용함을 표시
  * `s2= = accept(s)` : 누군가 로컬 포트에 커넥션 맺기를 기다림
  * `n = read(s, buffer, n)` : 소켓으로부터 버퍼에 n바이트 읽기 시도
  * `n = write(s, bufffer, n)` : 소켓으로부터 버퍼에 n바이트 쓰기 시도
  * `close(s)` : TCP 커넥션을 완전히 끊음
  * `shutdown(s, <side>)` : TCP 커넥션의 입출력만 닫음
  * `getsockopt(s, ...)` : 내부 소켓 설정 옵션값을 읽음
  * `setsockopt(s, ...)` : 내부 소켓 설정 옵션값을 변경
* 소켓 API를 사용하면 TCP 종단(endpoint) 데이터 구조를 생성하고, 원격 서버의 TCP 종단에 그 종단 데이터 구조를 연결하여 데이터 스트림을 읽고 쓸 수 있다.
* TCP API는 기본적인 네트워크 프로토콜의 핸드셰이킹, TCP 데이터 스트림과 IP 패킷 간의 분할 및 재조립에 대한 모든 세부사항을 외부로부터 숨긴다.
* 클라이언트와 서버가 TCP 소켓 인터페이스를 사용하여 상호작용하는 방법
  1. 웹 서버는 커넥션을 기다리기 시작한다.
  2. 클라이언트는 URL에서 IP 주소와 포트 번호를 알아내고 서버에 TCP 커넥션을 생성하기 시작한다.
  3. 커넥션 생성은 서버와의 거리, 서버의 부하, 인터넷 혼잡도에 따라서 시간이 걸린다.
  4. 일단 커넥션이 맺어지면 클라이언트는 HTTP 요청을 보내고 서버는 그것을 읽는다.
  5. 서버가 요청 메시지를 다 받으면 그 요청을 분석하여 클라이언트가 원하는 동작을 수행하고 클라이언트에게 데이터를 보낸다.
  6. 클라이언트는 그것을 받아 응답 데이터를 처리한다.

## 4.2 TCP의 성능에 대한 고려

* HTTP는 TCP 바로 위 계층이기 때문에 HTTP 트랜잭션의 성능은 그 아래 계층인 TCP 성능의 영향을 받는다.

### 4.2.1 HTTP 트랜잭션 지연

* 트랜잭션을 처리하는 시간은 TCP 커넥션을 설정하고, 요청을 전송하고, 응답 메시지를 보내는 거셍 비하면 상당히 짧다.
* 클라이언트나 서버가 너무 많은 데이터를 내려받거나 복잡하고 동적인 자원들을 실행하지 않는 한, 대부분의 HTTP 지연은 TCP 네트워크 지연 때문에 발생한다.
* HTTP 트랜잭션을 지연시키는 원인
  1. 클라이언트는 URI에서 웹 서버의 IP 주소와 포트번호를 알아내야 한다.
      * 만약 URI에 기술된 호스트에 방문한 적이 최근에 없으면, DNS 이름 분석(DNS resolution) 인프라를 사용하여 URI에 있는 호스트 명을 IP 주소로 변화하는데 수십 초의 시간이 걸릴 것이다.
  2. 클라이언트는 TCP 커넥션 요청을 서버에게 보내고 서버가 커넥션 허가 응답을 회신하기를 기다린다.
      * 커넥션 설정 시간은 새로운 TCP 커넥션에서 항상 발생한다. 이는 보통 1-2초 시간이 소요되지만, 수백 개의 HTTP 트랜잭션이 만들어지면 소요시간은 크게 증가할 것이다.
  3. 커넥션이 맺어지면 클라이언트는 HTTP 요청을 새로 생성된 TCP 파이프를 통해 전송한다.
      * 웹 서버는 데이터가 도착하는 대로 TCP 커넥션에서 요청 메시지를 읽고 처리한다.
      * 요청 메시지가 인터넷을 통해 전달되고 서버에 의해서 처리되는 데 까지는 시간이 걸린다.
  4. 웹 서버가 HTTP 응답을 보내는 것 역시 시간이 걸린다.
* 이런 TCP 네트워크 지연은 하드웨어의 성능, 네트워크와 서버의 전송 속도, 요청과 응답 메시지의 크기, 클라이언트와 서버 간의 거리에 따라 크게 달라진다. 또한 TCP 프로토콜의 기술적 복잡성도 큰 영향을 끼친다.

### 4.2.2 성능 관련 중요 요소

* TCP 커넥션의 해드셰이크 설정
* 인터넷의 혼잡을 제어하기 위한 TCP의 느린 시작(slow-start)
* 데이터를 한데 모아 한 번에 전송하기 위한 네이글(nagle) 알고리즘
* TCP의 편승(piggyback) 확인응답(acknowledgement)을 위한 확인응답 지연 알고리즘
* TIME_WAIT 지연과 포트 고갈

### 4.2.3 TCP 커넥션 핸드셰이크 지연

* 어떤 데이터를 전송하든, 새로운 TCP 커넥션을 열 때면, TCP 소프트웨어는 커넥션을 맺기 위한 조건을 맞추기 위해 연속으로 IP 패킷을 교환한다.
* 작은 크기의 데이터 전송에 커넥션이 사용된다면, 이런 패킷 교환은 HTTP 성능을 크게 저하시킬 수 있다.
* TCP 커넥션이 핸드셰이크를 하는 순서
  1. 클라이언트는 새로운 TCP 커넥션을 생성하기 위해 작은 TCP 패킷(보통 40~60byte)을 서버에 보낸다. 그 패킷은 `SYN`라는 특별한 플래그를 가지는데, 커넥션 생성 요청이란 뜻이다.
  2. 서버가 그 커넥션을 받으면 몇 가지 커넥션 매개변수를 산출하고, 커넥션 요청이 받아들여졌음을 의미하는 `SYN`과 `ACK` 플래그를 포함한 TCP 패킷을 클라이언트에게 보낸다.
  3. 마지막으로 클라이언트는 커넥션이 잘 맺어졌음을 알리기 위해 서버에게 다시 확인응답 신호를 보낸다. 오늘날 TCP는 클라이언트가 이 확인응답 패킷과 함께 데이터를 보낼 수 있다.
* HTTP 프로그래머는 패킷들을 보지 못한다(TCP 소프트웨어가 패킷들을 보이지 않게 관리한다). 프로그래머가 보는 것은 새로운 TCP 커넥션이 생성될 때 발생하는 지연이 전부이다.
* HTTP 트랜잭션이 아주 큰 데이터를 주고받지 않는 평범한 경우, `SYN/SYN+ACK` 핸드셰이크가 눈에 띄는 지연을 발생시킨다.
  * TCP의 ACK 패킷은 HTTP 요청 메시지 전체를 전달할 수 있을 만큼 큰 경우가 많고, 많은 HTTP 서버 응답 메시지는 하나의 IP 패킷에도 담길 수 있다.
* 결국 크기가 작은 HTTP 트랜잭션은 50%이상의 시간을 TCP를 구성하는 데 쓴다.

### 4.2.4 확인응답 지연

* 인터넷 자체가 패킷 전송을 완벽보장하지 않기 때문에(인터넷 라우터는 과부하가 걸렸을 때 패킷을 마음대로 파기할 수 있다), TCP는 성공적인 데이터 전송을 보장하기 위해 자체 확인 체계를 가진다.
* 각 TCP 세그먼트는 순번과 데이터 무결성 체크섬을 가진다.
  * 각 세그먼트 수신자는 세그먼트를 온전히 받으면, 작은 확인응답 패킷을 송신자에게 반환한다.
  * 송신자가 특정 시간 안에 확인응답 메시지를 받지 못하면 패킷이 파기되었거나 오류가 있는 것으로 판단하고 데이터를 다시 전송한다.
* 확인응답은 크기가 작기 때문에, TCP는 같은 방향으로 송출되는 데이터 패킷에 확인응답을 '편승(piggyback)' 시킨다.
  * TCP는 송출 데이터 패킷과 확인응답을 하나로 묶어 네트워크를 좀 더 효율적으로 사용한다.
* 확인응답이 같은 방향으로 나가는 데이터 패킷에 편승되는 경우를 늘리기 위해 많은 TCP 스택은 '확인 응답 지연' 알고리즘을 구현한다.
  * 확인응답 지연은 송출할 확인응답을 특정 시간 동안(보통 0.1~0.2초) 버퍼에 저장해 두고, 확인응답을 편승시키기 위한 송출 데이터 패킷을 찾는다. 일정 시간 패킷을 찾지 못하면 확인응답은 별도 패킷을 만들어 전송된다.
* HTTP 동작 방식은 요청과 응답 두 가지 형식으로만 이루어지므로, 확인 응답이 송출 데이터 패킷에 편승할 기회를 감소시킨다.
* 막상 편승할 패킷을 찾으려고 하면, 해당 방향으로 송출될 패킷이 많지 않기 때문에 확인 응답 지연 알고리즘으로 인한 지연이 자주 발생한다.
  * 운영체제마다 다르나, 지연 원이이 되는 확인응답 지연 관련 기능을 수정하거나 비활성화 할 수 있다.

### 4.2.5 TCP 느린 시작(slow start)

* TCP 데이터 전송속도는 TCP 커넥션이 만들어진 지 얼마나 지났는지에 따라 달라질 수 있다.
* TCP 커낵션은 시간이 지나면서 자체적으로 '튜닝'되어서, 처음에는 커넥션의 최대 속도를 제한하고 데이터가 성공적으로 전송됨에 따라 속도 제한을 높여나간다. 이 과정을 TCP 느린 시작이라고 하며, 급작스런 부하와 혼잡을 방지한다.
* TCP 느린 시작은 TCP가 한 번에 전송할 수 있는 패킷 수를 제한한다.
  * 패킷이 성공적으로 전달되는 각 시점에 송신자는 추가로 2개 패킷을 더 전송할 수 있는 권한을 얻는다. 전송 데이터 양이 많으면 모든 것을 한 번에 전송할 수 없으므로, 한 개 패킷만 전송하고 확인응답을 기다려야 한다. 확인 응답을 받으면 2개 패킷을 보낼 수 있고, 그 패킷 각각의 확인응답을 받으면 총 4개 패킷을 보낼 수 있다. : '혼잡 윈도를 얻다' , '혼잡 제어 기능'
* 이 기능 때문에 새 커넥션은 이미 어느 정도 데이터를 주고받은 튜닝된 커넥션보다 느려서 더 빠르게 사용하기 위해 이미 존재하는 커넥션을 재사용하는 기능이 있다.

### 4.2.6 네이글(Nagle) 알고리즘과 TCP_NODELAY

* TCP가 작은 데이터 크기를 포함한 많은 수의 패킷을 전송한다면 네트워크 성능은 크게 떨어진다.
* 네이글 알고리즘: 네트워크 효울을 위해서 패킷을 전송하기 전에 많은 양의 TCP 데이터를 한 개 덩어리로 합친다.
  * 세그먼트가 최대 크기(패킷의 최대 크기는 LAN 상에서 1,500바이트, 인터넷상에서는 수백 바이트 정도)가 되지 않으면 전송을 하지 않는다. 다만 다른 모든 패킷이 확인응답 받았을 경우 최대 크기보다 작은 패킷의 전송을 허락한다.
  * 다른 패킷들이 아직 전송중이면 데이터는 버퍼에 저장된다.
  * 전송 후 확인응답을 기다리던 패킷이 확인응답을 받았거나 전송하기 충분할 만큼 패킷이 쌓였을 때 버퍼에 저장되어 있던 데이터가 전송된다.
* 네이글 알고리즘은 HTTP 성능 관련 여러 문제를 발생시킨다.
  * 크기가 작은 HTTP 메시지는 패킷을 채우지 못해서 앞으로 생길지 모르는 추가적 데이터를 기다리며 지연된다.
  * 확인응답 지연과 함께 쓰일 때 형편없이 동작한다.
      * 네이글은 확인 응답 도착까지 데이터 전송을 멈추고 있는 반면, 확인응답 지연 알고리즘은 확인응답을 100~200밀리초 지연시킨다.
* HTTP 애플리케이션은 성능 향상을 위해 HTTP 스택에 TCP_NODELAY 파라미터 값을 설정하여 네이글 알고리즘을 비활성화하기도 한다.
  * 이 경우 작은 크기의 패킷이 너무 많이 생기지 않도록 큰 크기의 데이터 덩어리를 만들어야 한다.


### 4.2.7 TIME_WAIT의 누적과 포트 고갈

* TIME_WAIT 포트 고갈은 성능 측정 시 심각한 성능 저하를 발생시키지만, 보통 실제 상황에서는 문제 발생시키지지 않는다. 하지만 결국 이 문제를 마주하게 되고, 생각치 못한 성능상 문제로 오해할 수 있으니 주의해야 한다.
* TCP 커넥션 종단에서 TCP 커넥션을 끊으면, 종단에서는 커넥션의 IP주소와 포트 번호를 메모리의 작은 제어영역 control block에 기록해둔다.
  * 이것은 같은 주소와 포트 번호를 사용한 새로운 TCP 커넥션이 일정 시간 동안에는 생성되지 않기 위한 것으로, 보통 세그먼트의 최대 생명주기 두 배 정도(`2MSL`이라 불리며 보통 2분)의 시간 동안만 유지된다.
  * 이는 이전 커넥션과 관련된 패킷이 그 커넥션과 같은 주소, 포트번호를 가지는 새 커넥션에 삽입되는 문제를 방지한다.
* 실제로 특정 커넥션이 생성되고 닫힌 다음 그와 같은 IP 주소, 포트번호를 가지는 커넥션이 2분 이내 또 생성되는 것을 막아준다.

## 4.3 HTTP 커넥션 관리

### 4.3.1 흔히 잘못 이해하는 Connection 헤더

* HTTP는 클라이언트와 서버 사이에 프락시 서버, 캐시 서버와 같은 중개 서버가 놓이는 것을 허용한다.
* 두 개의 인접한 HTTP 애플리케이션이 현재 맺고 있는 커넥션에만 적용될 옵션을 지정해야할 때, HTTP Connection 헤더 필드는 커넥션 토큰을 쉼표로 구분하여 가지고 있으며 그 값들은 다른 커넥션에 전달되지 않는다.
  * 예를 들어 다음 메시지를 보낸 다음 끊어져야 할 커넥션은 `Connection: close`라고 명시할 수 있다.
* Connection 헤더는 다음 세 종류의 토큰이 전달될 수 있다.
  * HTTP 헤더 필드 명: 이 커넥션에만 해당되는 헤더들을 나열한다. 다음 커넥션에 전달하면 안 된다.
  * 임시적인 토큰 값: 커넥션에 대한 비표준 옵션을 의미한다.
  * close 값은: 커넥션 작업이 완료되면 종료되어야 함을 의미한다.
* Connection 헤더의 모든 헤더 필드는 메시지를 다른 곳으로 전달하는 시점에 삭제되어야 한다.
* 헤더 보호하기: Connection 헤더에는 홉별(hop-by-hop) 헤더 명을 기술한다.
```
HTTP/1.1 200 OK
Cache-control: max-age=3600
Connection: meter, close, bill-my-credit-card
Meter: max-uses=3, max-refuses=6, dont-report
```
-> Connection 헤더는 Meter 헤더를 다른 커넥션으로 전달하면 안 되고, `bill-my-credit-card` 옵션을 적용할 것이며, 이 트랜잭션이 끝나면 커넥션이 끊길 것이라고 말한다.

### 4.3.2 순차적인 트랜잭션 처리에 의한 지연

* 이미지 3개가 있는 웹페이지의 경우 4개의 HTTP 트랜잭션을 만들어야 한다.
  * 1개는 HTML 받기, 3개는 첨부된 이미지 받기
  * 각 트랜잭션이 새로운 커넥션을 필요로 한다면 커넥션 맺는데 발생하는 지연과 함께 느린 시작 지연이 발생할 것이다.
* 특정 브라우저의 경우 객체를 화면에 배치하려면 객체 크기를 알아야 하기 때문에, 객체를 모두 내려받기 전까지 텅 빈 화면을 보여주기도 한다. 이 경우 객체를 하나씩 연속해서 내려받는 것이 효율적일 것이다.
* HTTP 커넥션의 성능을 향상시킬 수 있는 최신 기술에는 병렬 커넥션, 지속 커넥션, 파이프라인 커넥션, 다중 커넥션이 있다.

## 4.4 병렬 커넥션

* 여러 개의 TCP 커넥션을 통한 동시 HTTP 요청이다.

### 4.4.1 병렬 커넥션은 페이지를 더 빠르게 내려받는다

* 각 커넥션의 지연시간을 겹치게 하면 총 지연 시간을 줄일 수 있고, 클라이언트의 인터넷 대역폭을 한 개의 커넥션이 다 써버리는 것이 아니라면 나머지 객체를 내려받는 데에 남은 대역폭을 사용할 수 있다.
* HTML 페이지를 먼저 내려받고 남은 세 개의 트랜잭션이 각각 별도 커넥션에서 동시에 처리된다. 이미지들은 병렬로 내려받아 커넥션 지연이 겹쳐져 총 지연시간은 줄어든다.

### 4.4.2 병렬 커넥션이 항상 더 빠르지는 않다

* 클라이언트이 네트워크 대역폭이 좁을 경우, 제한 된 대역폭 내에서 각 객체를 전송받는 것은 느리기 때문에 성능상 장점이 거의 없다.
* 다수의 커넥션은 메모리를 많이 소모하고 자체 성능 저하를 발생시킨다. 클라이언트가 수백개 커넥션을 열 수도 있지만, 서버는 다른 사용자 요청도 함께 처리해야하 하므로 수백 개의 커넥션을 허용하는 경우는 드물다. 이는 서버, 프락시 성능도 크게 떨어뜨릴 수 있다.
* 브라우저는 실제로 병렬 커넥션을 사용하지만, 적은 수(대부분 4개)의 병렬 커넥션만을 허용한다. 서버는 과도한 수의 커넥션이 맺어졌을 때 임의로 끊어버릴 수 있다.

### 4.4.3 병렬 커넥션은 더 빠르게 '느껴질 수' 있다

* 병렬 커넥션이 실제로 페이지를 더 빠르게 내려받는 것은 아니지만, 화면에 여러 객체가 동시에 보이면서 내려받고 있는 상황을 볼 수 있기 떄문에 사용자는 더 빠른 것 처럼 느낄 수 있다.

## 4.5 지속 커넥션

* 사이트 지역성(site locally): 보통 웹 클라이언트는 같은 사이트에 여러 개의 커넥션을 맺는다. 서버에 HTTP 요청을 하기 시작한 애플리케이션은 페이지 내 이미지, 링크 등을 가져오기 위해 그 서버에 또 요청을 하게될 것이다.
* HTTP/1.1을 지원하는 기기는 처리 완료 후에도 TCP 커넥션을 유지하여 앞으로의 HTTP 요청에 재사용할 수 있다. 처리가 완료된 후에도 계속 연결된 상태로 있는 TCP 커넥션을 '지속 커넥션'이라고 한다.
  * 지속 커넥션은 클라이언트나 서버가 커넥션을 끊기 전까지는 트랜잭션 간에도 커넥션을 유지한다.
  * 이미 맺어진 지속 커넥션을 재사용하여 커낵션 맺기 위해 준비하는 시간을 절약할 수 있고, 이미 맺어진 커넥션은 TCP의 느린 시작으로 인한 지연을 피하여 더 빨리 데이터를 전송할 수 있다.

### 4.5.1 지속 커넥션 vs 병렬 커넥션

* 병렬 커넥션은 여러 객체가 있는 페이지를 더 빠르게 전송하지만 몇 가지 단점이 있다.
  * 각 트랜잭션마다 새로운 커넥션을 맺고 끊기 떄문에 시간과 대역폭이 소요된다.
  * 각각 새로운 커넥션은 TCP 느린 시작 때문에 성능이 떨어진다.
  * 실제로 연결할 수 있는 병렬 커넥션의 수에는 제한이 있다.
* 지속 커넥션의 장점
  * 커넥션을 맺기 위한 사전 작업과 지연을 줄여준다.
  * 튜닝된 커넥션을 유지하며, 커넥션 수를 줄여준다.
* 지속 커넥션을 잘못 관리하면 계속 연결된 상태로 있는 수많은 커낵션이 쌓이게 될 것이다. 이는 로컬 리소스, 원격의 클라이언트와 서버 리소스에도 불필요한 소모를 발생시킨다.
* 지속 커넥션은 병렬 커넥션과 함께 사용할 때 가장 효과적이다.
* 두 가지 지속 커넥션 타입: HTTP/1.0+에는 `keep-alive` 커넥션이 있고, HTTP/1.1에는 `지속` 커넥션이 있다.

### 4.5.2 HTTP/1.0+의 Keep-Alive 커넥션

* 초기 지속 커넥션은 상호 운용과 관련된 설계에 문제가 있었지만, 아직 많은 클라이언트와 서버는 이 초기 keep-alive 커넥션을 사용하고 있다.
* keep-alive 커넥션의 성능상 장점은, 커넥션을 맺고 끊는 데 필요한 작업이 없어서 시간이 단축되었다.

### 4.5.3 Keep-Alive 동작

* keep-alive를 사용하지 않기로 결정되어 HTTP/1.1 명세에는 빠졌으나, 아직도 브라우저와 서버 간 keep-alive 핸드셰이크가 널리 사용되고 있어서 HTTP 애플리케이션은 그를 처리할 수 있게 개발해야 한다.
* HTTP/1.0 keep-alive 커넥션을 구현한 클라이언트는 커넥션 유지를 위해 요청에 `Connection:Keep-Alive` 헤더를 포함시킨다.
* 요청 받은 서버가 그 다음 요청도 이 커넥션을 통해 받고자 한다면, 응답 메시지에 같은 헤더를 포함시켜 답한다. 응답에 이 헤더가 없으면 클라이언트는 서버가 keep-alive를 지원하지 않으며 응답 메시지 전송 후 서버 커넥션을 끊을 것이라 추정한다.

### 4.5.4 Keep-Alive 옵션

* Keep-Alive 헤더는 커넥션을 유지하기를 바라는 요청일 뿐이다.
* 클라이언트나 서버가 keep-alive 요청을 받고 무조건 그를 따를 필요는 없다. 언제든 현재 keep-alive 커넥션을 끊을 수 있고, keep-alive 커넥션에 처리되는 트랜잭션의 수를 제한할 수도 있다.
* 쉼표로 구분된 옵션들
  * timeout: Keep-Alive 응답 헤더를 통해 보낸다. 커넥션이 얼마간 유지될 것인지를 의미하나, 이대로 동작한다는 보장은 없다.
  * max: Keep-Alive 응답 헤더를 통해 보낸다. 커넥션이 몇 개의 HTTP 트랜잭션을 처리할 때까지 유지될 것인지를 의미한다. 이대로 동작한다는 보장은 없다.
  * Keep-Alive 헤더: 진단, 디버깅을 주목적으로 하는, 처리되지 않는 임의 속성들을 지원하기도 한다. 이름[=값] 같은 식. 헤더 사용은 선택사항이지만 `Connection:Keep-Alive` 헤더가 있을 때만 사용할 수 있다.

### 4.5.5 Keep-Alive 커넥션 제한과 규칙

* HTTP/1.0에서 기본으로 사용되지 않는다. `Connection:Keep-Alive` 요청헤더를 보내야 사용할 수 있다.
* 커넥션을 계속 유지하려면 모든 메시지에 `Connection:Keep-Alive` 헤더를 포함해 보내야 한다.
* 클라이언트는 `Connection:Keep-Alive` 응답 헤더가 없는 것을 보고 서버가 응답 후에 커넥션을 끊을 것임을 알 수 있다.
* 커넥션이 끊기기 전, 엔터티 본문의 길이를 알아야 커넥션을 유지할 수 있다. 정확한 Content-Length 값과 함께 멀티파트 미디어형식을 가지거나 청크 전송 인코딩으로 인코드 되어야 한다.
* 프락시와 게이트웨이는 메시지를 전달하거나 캐시에 넣기 전에 Connection 헤더에 명시된 모든 헤더 필드와 Connection 헤더를 제거해야 한다.
* Connection 헤더를 인식하지 못하는 프락시 서버와는 맺어지면 안된다.
* 기술적으로 HTTP/1.0을 따르는 기기로 부터 받는 모든 Connection 헤더 필드는 Keep-Alive를 포함하여 무시해야 한다.
* 클라이언트는 응답 전체를 모두 받기 전에 커넥션이 끊어졌을 경우, 별다른 문제가 없으면 요청을 다시 보낼 수 있게 준비되어있어야 한다.

### 4.5.6 Keep-Alive와 멍청한(dumb) 프락시

#### Connection 헤더의 무조건 전달

* 프락시는 Connection 헤더를 이해하지 못해 해당 헤더를 삭제하지 않고 요청 그대로 다음 프락시에 전달한다.
* Connection 헤더는 홉별 헤더라 다음 서버로 전달되어서는 안 된다. 전달된 HTTP 요청이 서버에 도착하여 웹서버가 프락시로부터 `Connection:Keep-Alive` 헤더를 받으면, 서버는 프락시가 커넥션을 유지하고 요청하는 것으로 잘못 판단한다.
* 프락시와 커넥션 유지하는 것에 동의하여 `Connection:Keep-Alive` 헤더를 포함한 응답을 보낸다. 웹 서버는 프락시와 keep-alive 커넥션이 맺어진 상태로 keep-alive 규칙에 맞게 통신하는 것으로 판단하지만, 프락시는 keep-alive를 이해하지 못한다.
* 프락시는 서버에게 받은 응답 메시지를 클라이언트에게 전달하고, 클라이언트는 헤더를 통해 프락시가 커넥션을 유지하는 것에 동의한 것으로 추정한다.
* 프락시는 데이터를 클라이언트에게 전달 후 서버가 커넥션을 끊기를 기다리지만, 서버는 프락시가 자신에게 커넥션 유지를 요청한 것으로 알고 있어서 커넥션을 끊지 않는다.
* 클라이언트는 응답을 받은 후 다음 요청을 보내기 시작하는데, 커넥션이 유지되고 있는 프락시에 요청을 보낸다. 프락시는 커넥션 상에서 다른 요청이 오는 경우를 예상치 못했기 떄문에 그 요청은 무시되고, 브라우저는 아무 응답 없이 로드중 표시만 나온다.
* 이런 잘못된 통신으로 브라우저는 자신이나 서버가 타임아웃이 나서 커넥션이 끊길 때까지 기다린다.

#### 프락시와 홉별 헤더

* 이런 잘못된 통신을 피하려면 프락시는 Connection 헤더와 Connection 헤더에 명시된 헤더들은 절대 전달하면 안된다.
* Connection 헤더 값으로 명시되지 않는 홉별 헤더들 역시 전달하거나 캐시하면 안 된다.

### 4.5.7 Proxy-Connection 살펴보기

* 클라이언트 요청이 중개서버를 통해 이어지는 경우 모든 헤더를 무조건 전달하는 문제 해결을 위해 넷스케이프는 Connection 헤더 대신 비표준인 Proxy-Connection 확장 헤더를 사용한다.(모든 상황에서 동작하지는 않는다)
* 프락시를 별도로 설정할 수 있는 현대 브라우저에서 지원하며, 많은 프락시들도 인식한다.
* 프락시가 Proxy-Connection을 무조건 전달해도 웹 서버에서 무시한다.
* 영리한 프락시는 의미없는 Proxy-Connection 헤더가 keep-alive 요청임을 인식하여, keep-alive 커넥션을 맺기 위해 자체적으로 `Connection:Keep-Alive` 헤더를 웹서버에 전송한다.
  * 이것은 클라이언트와 서버 사이에 한 개의 프락시만 있는 경우에만 한정된다.

### 4.5.8 HTTP/1.1의 지속 커넥션

* HTTP/1.1에서는 keep-alive 커넥션을 지원하지 않는 대신 설계가 더 개선된 지속 커넥션을 지원한다.
* HTTP/1.1에서는 별도 설정을 하지 않는 한, 모든 커넥션을 지속커넥션으로 취급한다.

### 4.5.9 지속 커넥션의 제한과 규칙

* 클라이언트가 요청에 `Connection: close` 헤더를 포함해 보냈으면, 클라이언트는 그 커넥션으로 추가 요청을 보낼 수 없다.
* 커넥션에 있는 모든 메시지가 자신의 길이를 정확히 가지고 있을 때에만 커넥션을 지속시킬 수 있다.
* HTTP/1.1 프락시는 클라이언트와 서버 각각에 대해 별도 지속 커넥션을 맺고 관리해야 한다.
* HTTP/1.1 프락시는 클라이언트의 커넥션 관련 기능에 대한 지원 범위를 알고 있지 않은 한 지속 커넥션을 맺으면 안 된다.
* HTTP/1.1 기기는 Connection 헤더 값과 상관없이 언제든 커넥션을 끊을 수 있다.
* 클라이언트는 전체 응답을 받기 전에 커넥션이 끊어지면, 요청을 반복해서 보내도 문제 없는 경우에는 요청을 다시 보낼 준비가 되어야한다.
* N명의 사용자가 서버로 접근하려면, 프락시는 서버나 상위 프락시에 넉넉잡아 2N개의 커넥션을 유지해야한다.

## 4.6 파이프라인 커넥션

* HTTP/1.1은 지속 커넥션을 통해서 요청을 파이프라이닝할 수 있다.
* 파이프라인의 제약사항
  * HTTP 클라이언트는 커넥션이 지속 커넥션인지 확인하기 전까지는 파이프라인을 이어서 안 된다.
  * HTTP 응답은 요청 순서와 같게 와야 한다.
  * HTTP 클라이언트는 커넥션이 언제 끊어지더라도, 완료되지 않은 요청이 파이프라인에 있으면 언제든지 다시 요청을 보낼 준비가 되어있어야 한다.
  * HTTP 클라이언트는 POST 요청같이 반복해서 보낼 경우 문제가 생기는 요청은 파이프라인을 통해 보내면 안 된다.


## 4.7 커넥션 끊기에 대한 미스터리

### 4.7.1 '마음대로' 커넥션 끊기

* HTTP 클라이언트, 서버, 프락시 등 언제든지 TCP 전송 커넥션을 끊을 수 있다.
* 보통 커넥션은 메시지 다 보낸 다음 끊지만, 에러 상황에서는 헤더 중간이나 다른 엉뚱한 곳에서 끊길 수 있다.

### 4.7.2 Content-Length와 Truncation

* 각 HTTP 응답은 본문의 정확한 크기 값을 가지는 Orientel-Lengt 헤더를 가지고 있어야 한다.
* 클라이언트나 프락시가 커넥션이 끊어진 HTTP 응답을 받은 후, 실제 전달된 엔터티의 길이와 Content-Length의 값이 일치하지 않거나 Content-Length 자체가 존재하지 않으면 수신자는 데이터의 정확한 길이를 서버에게 물어봐야 한다.
* 프락시가 캐시 프락시일 경우 응답을 캐시해서는 안 된다.

### 4.7.3 커넥션 끊기의 허용, 재시도, 멱등성

* 커넥션은 에러가 없어도 언제든 끊을 수 있다.
* 어떤 요청 데이터가 전송되었지만, 응답이 오기 전에 커넥션이 끊기면 클라이언트는 실제로 서버에서 얼마만큼 요청이 처리되었는지 알 수 없다.
* 한 번 혹은 여러 번 실행됐는지에 상관없이 같은 결과를 반환한다면, 그 트랜잭션은 멱등(idempotent)하다고 한다. GET, HEADE, PUT, DELETE, TRACE, OPTIONS 메서드들은 멱등하다고 이해하면 된다.
* 클라이언트는 POST와 같이 멱등이 아닌 요청은 파이프라인을 통해 요청하면 안 된다. 전송 커넥션이 예상치 못하게 끊어져 버리면 알 수 없는 결과를 초래할 수도 있다.

### 4.7.4 우아한 커넥션 끊기

* TCP 커넥션은 양방향이다. TCP 커넥션의 양쪽에는 데이터를 읽고 쓰기 위한 입력 큐와 출력 큐가 있다.
* 출력 큐에 있는 데이터는 다른 쪽 입력 큐에 보내질 것이다.

#### 전체 끊기와 절반 끊기

* 애플리케이션은 TCP 입력 채널과 출력 채널 중 한 개만 끊거나 둘 다 끊을 수 있다.
* 전체끊기 `close()` : TCP 커넥션의 입력/출력 채널의 커넥션을 모두 끊는다.
* 절반끊기 `shutdown()` : 입력/출력 채널 중 하나를 개별적으로 끊는다.

#### TCP 끊기와 리셋 에러

* 단순한 HTTP 애플리케이션은 전체 끊기만 사용할 수 있다. 그러나 예상치 못한 쓰기 에러를 발생하는 것을 예방하기 위해 절반 끊기를 사용해야 한다.
* 보통 커넥션의 출력 채널을 끊는 것이 안전하다.
* 클라이언트에서 이미 끊긴 입력 채널에 데이터를 전송하면 서버의 운영체제는 `TCP 'connection reset by peer'` 메시지 응답을 클라이언트에 보낼 것이다.
* 데이터를 읽으려 하면 connection reset by peer 에러를 받게 되고, 응답 데이터가 내 기기에 잘 도착했어도, 아직 읽히지 않은 버퍼에 있는 응답 데이터는 사라지게 된다.

#### 우아하게 커넥션 끊기

* 일반적으로 우아하게 커넥션 끊기란, 애플리케이션 자신의 출력 채널을 먼저 끊고 다른 쪽 기기의 출력 채널이 끊기는 것을 기다리는 것이다.
* 그러나 상대방이 절반 끊기를 구현했다는 보장도 없고, 절반 끊기를 했는지 검사해준다는 보장도 없다. 따라서, 절반 끊기를 하고 난 후에도 데이터나 스트림의 끝을 식별하기 위해 입력 채널에 대해 상태 검사를 주기적으로 해야한다.
* 입력채널이 특정 타임아웃 시간 내에 끊어지지 않으면 애플리케이션은 리소스를 보호하기 위해 커넥션을 강제로 끊을 수도 있다.

## 4.8 추가 정보

### 4.8.1 HTTP 커넥션 관련 참고자료
### 4.8.2 HTTP 성능 이슈 관련 참고자료
### 4.8.3 TCP/IP 관련 참고자료