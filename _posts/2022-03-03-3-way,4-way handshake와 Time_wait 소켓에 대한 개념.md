---
layout: post
title:  "3-way,4-way handshake와 Time_wait 소켓에 대한 개념"
categories: [ 네트워크 기본 ]
image: assets/images/11.jpg
description: "3-way,4-way handshake와 Time_wait 소켓에 대한 개념"
featured: true
---

<p align="center">
<img src="https://user-images.githubusercontent.com/59492312/154784990-1d04c43b-662c-4b21-9b0e-f58d3881d8f9.png">
</p>

<p align="center">
.   
.   
.   
</p>
  
<p align="center">
<img src="https://user-images.githubusercontent.com/59492312/156134675-b9190eec-3abc-46b7-b9b1-6acd22a4fb50.png">
</p>

# 👨‍💻 3-way,4-way handshake와 Time_wait 소켓에 대한 개념

* 3-way handshake와 4-way handshake에 대하여
* Linux kernel 버전확인과 레드핫 계열 리눅스 버전확인
* Time_wait 소켓이 문제가 되는 이유

> 모든 코드는 [깃헙](https://github.com/sooolog/dev-spring-springboot)에 작성되어 있습니다.
* * *

<br>



### 1.네트워크 통신 3 way handshake, 4 way handshake에 대해 알아보자.

<p align="center">
<img src="https://user-images.githubusercontent.com/59492312/154784990-1d04c43b-662c-4b21-9b0e-f58d3881d8f9.png">
</p>

우리가 개발할 때 사용하는 네트워크 통신에 대해 들어가기게 앞서 알아두어야 할 기본 지식들이 있다.

TCP는 한번씩 들어봤을거다.TCP(Transmission Control Protocol)는 쉽게 말하자면 인터넷상에서 데이터를 
메세지의 형태로 보내기 위해 IP와 함께 사용하는 규약이다. 브라우저에서 클라이언트가 우리가 배포한 서버로 접근하는것도,
리버스 프록시인 엔진엑스에서 WAS로 보내는 네트워크 통신도 TCP 통신의 일종이다.
               
<br>

> **HTTP통신과 TCP통신은 다른게 아닌가요?**         
> Http통신도 TCP통신을 기반으로 만들어져 있다. 그렇기에 아래에 설명할 3-way handshake, 4-way handshake 과정을
> TCP통신 뿐만이 아니라, Http통신도 똑같이 적용이된다. 아래에 3-way handshake와 4-way handshake를 보고나면 이해가 더 
> 잘 될것이다.    
> [HTTP통신과 TCP통신](https://hwan-shell.tistory.com/271)
                                                          
<br>

> [TCP란 무엇인가](https://mangkyu.tistory.com/15)

<br>

<p align="center">
<img src="https://user-images.githubusercontent.com/59492312/154784783-bb6a0a2f-adfc-435d-a479-627d43f795f7.png">
</p>

그럼 TCP 통신에 대해 알았으니, 그 과정에 대해 보도록 하겠다. 클라이언트와 서버단에서 서로 TCP 통신을
하기 위해서는 여러 과정을 거친다. 우선 클라이언트단에서 서버단에 최초의 연결을 맺으려 할 때 **3-way handshake**라는
과정을 거치게 된다.

**3-way handshake**란,    
1.클라이언트는 서버로 통신을 시작하겠다는 SYN을 보내고,       
2.서버는 그에 대한 응답으로 SYN+ACK를 보낸다.    
3.마지막으로 클라이언트는 서버로부터 받은 패킷에 대한 응답으로 ACK를 보낸다.   
이렇게 3-way-handshake를 정상적으로 마친 다음 클라이언트는 서버에 데이터를 요청한다.

3-Way Handshake를 하는 이유는 클라이언트, 서버 모두 데이터를 전송할 준비가 
되었다는 것을 보장하기 위한 것이다.

<br>

> [3-way handshake에 관하여 (1)](https://ohcodingdiary.tistory.com/7)     
> [3-way handshake에 관하여 (2)](https://puzzle-puzzle.tistory.com/entry/TIMEWAIT-%EC%86%8C%EC%BC%93%EC%9D%B4-%EC%84%9C%EB%B9%84%EC%8A%A4%EC%97%90-%EB%AF%B8%EC%B9%98%EB%8A%94-%EC%98%81%ED%96%A5)

<br>

<p align="center">
<img src="https://user-images.githubusercontent.com/59492312/154784775-0da1bd23-d1c3-4e51-9b2c-b982ea204789.png">
</p>

그 다음, 데이터를 주고 받은 뒤에 해당 연결을 끊어야 하는데 이 또한 4가지 과정을 거쳐야 한다.
이를 4-way handshake라 한다. 아래 순서를 보겠다.

**4-way handshake란**    
1.클라이언트는 응답을 주고 연결을 끊기 위해 FIN패킷을 보낸다.     
2.서버는 클라이언트에서 보낸 패킷에 대한 응답으로 ACK 패킷을 보낸다.     
3.서버는 자신이 사용한 소켓을 정리하며 통신을 완전히 끝내도 된다는 의미로 FIN 패킷을 보낸다.    
4.클라이언트는 서버 패킷에 대한 응답으로 ACK패킷을 보낸다.    

4-way handshake과정을 거치고 나면 클라이언트와 서버단의 네트워크는 종료가 된다.

<br>

> 연결을 맺을 때는 연결을 맺고자 하는 쪽에서 먼저 SYN를 보내며, 
> 연결을 끊을 때는 연결을 끊으려는 쪽에서 먼저 FIN을 보낸다.

<br>

> [4-way handshake에 관하여](https://puzzle-puzzle.tistory.com/entry/TIMEWAIT-%EC%86%8C%EC%BC%93%EC%9D%B4-%EC%84%9C%EB%B9%84%EC%8A%A4%EC%97%90-%EB%AF%B8%EC%B9%98%EB%8A%94-%EC%98%81%ED%96%A5)

<br>

<p align="center">
<img src="https://user-images.githubusercontent.com/59492312/154785710-fbedc126-d668-4b2c-b898-fbd442514f43.png">
</p>

위에서 본 TCP 통신은 모두 TCP Socket 통신이다. TCP Socket 통신은 소켓(socket)을 이용하여
클라이언트단과 서버단이 각각 소켓에 명시된 자신의 포트를 통해 통신해야만 한다. 즉, 클라이언트과 서버가 연결을 할 때도 
데이터를 교환할 때도 해당 소켓의 포트를 이용해야한다. 

즉, 3-way handshake, 4-way handshake 모두 클라이언트단과 서버단에 소켓(socket)이
생성되어야만 이 소켓을 연결부로 사용하여 통신을 할 수 있게 되는것이다.

<br>

> 소켓(Socket)은 프로토콜, IP주소, 포트로 정의되며, 네트워크상에서 동작하는 프로그램 간 통신의 종착점(Endpoint)이다.
> (정확히는 엔드포인트의 연결부이다.) 즉, 데이터를 통신할 수 있도록 해주는 연결부이기 때문에 통신할 두 프로그램(Client, Server) 
> 모두에 소켓이 생성되야 한다.         
> [소켓(socket)이란 (1)](https://medium.com/@su_bak/term-socket%EC%9D%B4%EB%9E%80-7ca7963617ff)   
> [소켓(socket)이란 (2)](https://helloworld-88.tistory.com/215)   

<br>

> Endpoint : IP Address와 port 번호의 조합을 뜻하며 최종 목적지를 나타낸다. 
> 예시로 최종목적지는 사용자의 디바이스(PC, 스마트폰 등) 또는 Server가 될 수 있다.       
> [엔드포인트(Endpoint)란](https://medium.com/@su_bak/term-socket%EC%9D%B4%EB%9E%80-7ca7963617ff)

<br>

<p align="center">
<img src="https://user-images.githubusercontent.com/59492312/154796824-93793128-9ae0-414b-8529-c8f6bd8c48e7.png">
</p>

그렇다면, 4-way handshake에 대해 더 자세히보고 4-way handshake를 사용하는 이유를 보도록 하겠다. 
위의 과정에서 왼쪽이 A, 오른쪽이 B라하면 A에서 먼저 연결을 끊는것으로 볼 수 있다.
(A가 FIN을 B로 보내고 있으니) 그 이후 마지막으로 A가 ACK를 B로 보내고
time_wait상태로 빠진다. B는 A에게서 마지막 ACK를 받고 소켓을 종료시킨다.
하지만, A는 time_wait로 인해 일정시간이 지나고 나서야 소켓이 닫힌다.

여기서, A가 time_wait시간을 갖는 이유는, 만약 B가 마지막 ACK를 받지못하면 
위에서 보이는 바와같이 B가 A에게 다시한번 FIN(위의 그림에서 위에서 세번째 화살표)을
보내게된다. 그런데, 만약 A가 time_wait없이 종료된 상태면 B가 원활한 종료를 하지 못하는것이다.
그렇기에, 마지막 A가 B에게 ACK를 보낼때 A는 time_wait상태에 빠지고, B는 해당 ACK를 받으면
정상적으로 소켓이 닫히고, A 소켓은 정해진 time 후에 자동으로 닫힌다.

<br>

> 위의 그림에서는 계속해서 클라이언트단이 연결을 먼저 끊어서 FIN을 서버단에 보낸다고 나와있는데,
> 클라이언트뿐만 아니라 서버단에서도 먼저 끊을 수 있다. 즉, 클라이언트라해서 꼭 연결을 먼저 끊는것은 아니며
> 서버단이라고 연결이 끊어질 때 FIN을 먼저 받기만하지 않는다. 반대로도 일어날 수 있는거다. 서버단에서 먼저 연결을 끊으면
> 서버단에서 먼저 FIN을 클라이언트단으로 보내고 위와같은 과정이 똑같이 일어나지만 서버단과 클라이언트단의 역활만 바뀔 뿐이다.
> 그렇기에, 연결을 먼저 끊는 쪽을 active closer라하고 그 반대를 passive closer라 한다. 클라이언트,서버 둘다 active closer
> passive closer가 될 수 있는거다.     
> [active closer와 passive closer에 대하여](https://puzzle-puzzle.tistory.com/entry/TIMEWAIT-%EC%86%8C%EC%BC%93%EC%9D%B4-%EC%84%9C%EB%B9%84%EC%8A%A4%EC%97%90-%EB%AF%B8%EC%B9%98%EB%8A%94-%EC%98%81%ED%96%A5)

<br>

> [active closer와 passive closer의 소켓 반환 시기 (1)](https://m.blog.naver.com/PostView.naver?isHttpsRedirect=true&blogId=cache798&logNo=130039539454)   
> [active closer와 passive closer의 소켓 반환 시기 (2)](https://velog.io/@sparkbosing/Network-%EB%84%A4%ED%8A%B8%EC%9B%8C%ED%81%AC%EC%9D%98-%EC%9B%90%EB%A6%AC7)   
> [4-way handshake와 time_wait 소켓 (1)](https://m.blog.naver.com/PostView.naver?isHttpsRedirect=true&blogId=cache798&logNo=130039539454)   
> [4-way handshake와 time_wait 소켓 (2)](https://tech.kakao.com/2016/04/21/closewait-timewait/)   

<br>



### 2.내가 사용하는 서버의 time_wait는 몇분일까?

<p align="center">
<img src="https://user-images.githubusercontent.com/59492312/156115401-59694091-b744-4285-92d8-37b6f1108386.png">
</p>

RFC 793(국제 인터넷 표준화 기구에서 기술한 핵심 프로토콜)에 따르면, time_wait의 시간을 2MSL(Maximum Segment Lifetime)로 설정해 놓았다.
하지만, 이 2MSL은 운영체제나 혹은 운영체제 버전(linus의 경우 Centos의 버전)에 따라서도 다르게 지정될 수 있다.(통상 MSL하나당 30초 ~ 2분) 하지만, 보통의 경우
OS에서는 최적화를 위하여 2MSL을 1분으로 설정하여 사용하도록 구현되어 있다. 실제 Centos6의 경우는 60초로 설정되어있다.

<br>

> [RFC 793 이란?](https://ko.wikipedia.org/wiki/%EC%A0%84%EC%86%A1_%EC%A0%9C%EC%96%B4_%ED%94%84%EB%A1%9C%ED%86%A0%EC%BD%9C)    
> [MSL은 운영체제 혹은 운영체제의 버전에 따라서 다르게 지정 가능 (1)](https://meetup.toast.com/posts/55)    
> [MSL은 운영체제 혹은 운영체제의 버전에 따라서 다르게 지정 가능 (2)](https://lazymankook.tistory.com/6)    
> [CentOS 6버전에서의 TIme_wait 시간](https://tech.kakao.com/2016/04/21/closewait-timewait/)      
    
<br>

<p align="center">
<img src="https://user-images.githubusercontent.com/59492312/156116286-9e4d9860-7b62-41fa-9934-3869d2e55d46.png">
</p>

그렇다면, 우리가 AWS EC2 인스턴스를 생성하는 경우(이 글에서는 aws 리소스를 예시로 설명하겠다.) 우리가
사용하는 서버의 time_wait를 어떻게 알 수 있을까?

이걸 알아가기에 앞서 우리는 리눅스 커널의 개념, 센토스와 우분투는 무엇인지 에 대해 보도록 하고,
리눅스 버전마다 time_wait이 다르니 EC2에서 리눅스버전(CentOS 버전) 그리고 리눅스커널 버전을
보는 방법에 대해서도 알도록 하겠다. 

<br>

<p align="center">
<img src="https://user-images.githubusercontent.com/59492312/156119891-19d94d7a-888c-4d12-a835-d3fc7c27e9d1.png">
</p>

우선 센토스와 우분투에 대해서 보도록 하겠다.    
리눅스는 공개  운영체제여서 오픈소스로 누구나 자유롭게 수정이 가능하다. 그렇기에 무수히 많은 버전의 리눅스가 있는데, 보통
레드햇계열과 데비안계열의 리눅스를 많이 사용한다. 레드햇이 센토스OS이고 데비안이 우분투OS이다.

**센토스**    
리눅스에서 서버용 운영체제로써 제일 인기가 많으며, 실제로도 서버용으로 리눅스를 운영할 목적이라면 센토스OS를 대부분
추천한다.

**우분투**    
센토스에 비해서 성능이 딸리거나 부족하지는 않으나 서버용으로는 점유율로 볼때 센토스에 많이 밀린다. 
하지만, 개인용 데스크톱 운영체제에서 많이 사용되고 있다.

이렇기에, 개인이 익숙한 리눅스 계열을 사용해도되고, 처음이라면 센토스로 시작해도 나쁘지않다.
(우리가 aws에서 사용하는 EC2의 경우는 센토스 계열이다.)

<br>

추가로,    
**리눅스 커널**    
리눅스 커널은 리눅스 운영체제의 주요 구성 요소이며, 컴퓨터 하드웨어와 프로세스를 잇는 핵심 인터페이스이다.
즉, 우분투와 센토스는 리눅스 그 자체라면 리눅스 커널은 이를 구성하는 요소로 보면된다.

<br>

> [리눅스의 센토스와 우분투에 관하여](https://coding-factory.tistory.com/318)         
> [리눅스 커널이란](https://redhat.com/ko/topics/linux/what-is-the-linux-kernel)      

<br>

<p align="center">
<img src="https://user-images.githubusercontent.com/59492312/156133324-d88d6d45-1278-4513-883b-92fe476ea7b7.png">
</p>

그렇다면, EC2의 리눅스 커널 버전과 센토스(aws ec2는 센토스를 사용한다.) 버전을 확인하는 방법에
대해 보도록 하겠다.

<br>

> EC2 인스턴스를 이미 생성하고 SSH접속까지 완료됬다는 가정하에 진행한다.

<br>

EC2 인스턴스에 ssh접속을 하였으면,      
cat /etc/*release* 명령어를 입력해준다.   

그러면,
```
[ec2-user@ip-172-31-1-230 ~]$ cat /etc/*release*
cat: /etc/lsb-release.d: Is a directory
NAME="Amazon Linux"
VERSION="2"
ID="amzn"
ID_LIKE="centos rhel fedora"
VERSION_ID="2"
PRETTY_NAME="Amazon Linux 2"
ANSI_COLOR="0;33"
CPE_NAME="cpe:2.3:o:amazon:amazon_linux:2"
HOME_URL="https://amazonlinux.com/"
Amazon Linux release 2 (Karoo)
cpe:2.3:o:amazon:amazon_linux:2
```

이와같이 보여지는데, 기존같으면 센토스 몇인지 나와야 하는데, 나오지가 않는다.
그 이유는 현재 필자가 글을 쓰는 시점에는 CentOS8의 경우 2021년 12월 31일부로 종료가 되었고, 
RHEL로 전환이 되었는데, 이러한 것을 반영한것일수도 있고, 내부적으로 CentOS 7을 사용하는데
보여지지 않는것일수도 있다. 우선은 레드헷 계열의 리눅스를 사용한다는걸 알 수 있다.

<br>

> RHEL(Red Hat Enterprise Linux)를 설명하자면, 기존의 CentoS가 이 RHEL의 소스를
> 그대로 갖고와서 Red Hat의 브랜드와 로고만 제거하고 만든 배포본이다. 즉, RHEL의 소스를 거의 수정없이
> 사용하였기에 센토스의 원본이라고 보아도 된다.   
> [Red Hat이란?](https://dejavuhyo.github.io/posts/linux-centos/)   

<br>

> [센토스 버전 확인방법 (1)](https://jesc1249.tistory.com/296)   
> [센토스 버전 확인방법 (2)](https://manualfactory.net/10698)   
> [CentOS 8 종료와 RHEL로 전환](http://www.opennaru.com/linux/centos-종료/)   

<br>

<p align="center">
<img src="https://user-images.githubusercontent.com/59492312/156129128-7753a5b0-3c09-4437-bbb7-18c04a44eca5.png">
</p>

리눅스 커널의 버전을 보는방법은 두가지다.    
하나는, AWS 인스턴스 생성시에 AMI(Amazon Machine Image)를 고르는 과정에서 볼 수 있는 방법이
있고, 다른하나는 ssh 접속하여 명령어로 알아보는 방법이다. 첫번째는 쉬우니 두번째로 알아보도록 하겠다.

아까와 같이 ssh접속을 한 상태에서, uname -r 을 입력해준다.

```
4.14.256-197.484.amzn2.x86_64
```

라고 뜨는게 보인다. 해당 버전은 커널 4.14버전임을 알 수 있다.
실제로, aws의 AMI 선택란에도 보면 Linux Jernel 4.14버전임을 알 수 있다.
(필자가 작성한 날짜 기준이다. 시간이 지나면 버전은 바뀔 수 있다.)

결론적으로, 내가 사용하는 EC2 인스턴스의 리눅스 커널 버전을 알고, redhat계열의
리눅스라는건 알았지만 자세한 버전은 알 수 없었다. 하지만, 기존 OS에서는 time_wait 소켓의
시간이 1분이라는 점과 CentOS 6의 time_wait도 1분이라는 점을 볼때 현재 사용하는 EC2의
time_wait도 1분으로 생각하고 가도 충분할것이다.

<br>

> 후에 정확한 time_wait을 알아야 하는 경우, aws문서나 혹은 ssh접속으로
> 명령어를 통해 알아보도록 하자.

<br>

> [EC2에서 센토스 버전 확인 (1)](https://carfediem-is.tistory.com/4)    
> [EC2에서 센토스 버전 확인 (2)](https://jesc1249.tistory.com/296)    

<br>



### 3.time-wait 소켓이 발생한다 했을때 이게 왜 문제가 될까?

<p align="center">
<img src="https://user-images.githubusercontent.com/59492312/154788409-5bb4fd57-b393-4b59-b8d0-37ad4783ab36.png">
</p>

사실 TCP 통신에서 TIME_WAIT 상태의 소켓이 발생하는것은 자연스러운 현상이다.
하지만, 이게 문제가 되는것은 너무많은 time_wait 상태의 소켓이 생성되어있을 때이다.

어느쪽이든 Client 입장(요청하는 쪽)에선 요청을 보내기 위해서는 커널에서 할당 받은 
로컬 포트로 소켓 생성을 요청한다. 

<br>

> 위의 3-way handshake나 4-way handshake에서는 클라이언트와 서버단이 마치 유저와 서비스 서버로 나누어 놓았지만, 이해상
> 이렇게 구분되어지게 설명한것이다. 실제로는 EC2인스턴스가 클라이언트단이 되고 유저들이 서버단이 될 수도 있다. 어떠한 단이 되는지에 대한
> 기준은 요청을 먼저 하는쪽이 클라이언트단, 받는쪽이 서버단이 되는것이다. 즉, EC2서버에서 먼저 요청을 보내겠다고 하면 EC2서버가 클라이언트단이 되는것이다. 
> 그 예로,      
> (1).톰캣서버(WAS)와 DB 연동     
> (2).톰캣서버(WAS)와 외부 API 연동        
> (3).Nginx와 톰캣서버(WAS) 연동    
> 의 경우 (1),(2)는 톰캣서버가 클라이언트단이 되고, (3)에서는 Nginx가 클라이언트단이 되는것이다.   
> [클라이언트단과 서버단의 개념](https://jojoldu.tistory.com/319)

<br>

그런데, 위 그림과 같이 통신할 때 사용할 포트를 커널에서 할당받을 때 이 포트번호는 고유한
번호(로컬 포트)를 할당받게 된다. 그리고 이 포트번호로 다시 커널에 소켓생성을 요청하여 생성된 소켓을 사용하여
TCP통신이 진행되게 된다.

문제는 여기서부터인데, 통신이 완전히 종료되고(여기서는 time_wait이 종료되는것으로 본다.) 소켓이
커널로 돌아가기전까지는 해당 소켓에 할당된 고유 포트번호를 사용할 수가 없다는 것이다. 만약 time_wait 상태의 소켓이 많아지게 되면
사용가능한 로컬포트 또한 줄어들게 되고, 끝으로는 모든 로컬포트가 time_wait소켓에 묶여있어 더이상 할당할 수 있는
로컬포트가 없게되는경우에 새로운 통신을 위한 소켓을 생성할 수 없게된다. 그러면, 더 이상 클라이언트단과 서버단은 서로 통신을
할 수 없게되는 것이다.

<br>

> 또 한가지 알아야 할것은 이 time_wait 소켓은 클라이언트단이나 서버단이나 연결을 먼저 끊으려고 하는쪽에서
> 발생한다. 하지만, 통상 클라이언트단에서 먼저 연결을 끊으려고 FIN을 서버단에 보내기에 대부분 클라이언트단에서
> time_wait 소켓이 발생한다고 볼 수 있다.      
> [time_wait 소켓의 발생 (1)](https://puzzle-puzzle.tistory.com/entry/TIMEWAIT-%EC%86%8C%EC%BC%93%EC%9D%B4-%EC%84%9C%EB%B9%84%EC%8A%A4%EC%97%90-%EB%AF%B8%EC%B9%98%EB%8A%94-%EC%98%81%ED%96%A5)    
> [time_wait 소켓의 발생 (2)](https://jojoldu.tistory.com/319)

<br>

> [time_wait 소켓이 문제가 되는 이유](https://jojoldu.tistory.com/319)

<br>

<p align="center">
<img src="https://user-images.githubusercontent.com/59492312/156504161-6c477067-119e-43c2-9127-764c9fb963da.png">
</p>

이해를 돕기위한 실제 예를 보여주도록 하겠다. 필자는 빈스톡과 Nginx를 사용하여 스프링부트 톰캣(내장)을
연결하여 사용한다.

위의 자료는 실제 Nginx가 클라이언트단이 되고, 서버단은 톰캣(WAS)이 되어서 통신하는 과정에서
time_wait 소켓이 발생한 목록이다. 자세히 보면, 왼쪽은 127.0.0.1:고유포트번호이고
오른쪽은 127.0.0.1:8080이다. 왼쪽이 클라이언트단인 Nginx의 소켓 고유 포트번호와 ip주소이고 오른쪽
127.0.0.1:8080이 스프링부트 어플리케이션이 실행중인 포트번호이다.

해당 time_wait 소켓(클라이언트단의 소켓)이 사용하는 포트번호가 모두 다른것을 볼 수 있다.

<br>

> 양쪽 다 ip주소가 127.0.0.1인 이유는 같은 EC2 인스턴스 내에서 이루어지는 통신이기 때문이다.

<br>

> [Nginx와 내장톰캣의 통신으로 본 ip주소와 포트번호 예시들](https://jojoldu.tistory.com/319)

<br>

여기까지가 네트워크 통신의 기본과 time_wait 소켓의 발생과정 그리고 time_wait 소켓이 왜 오류를 일으키는지에 대해
알아보았다. 그렇다면, 실제 서비스에서는 이 time_wait를 어떻게 해결할것인지 그 해결방안에 대해서는 실제 필자가 사용하고 있는
빈스톡과 nginx 그리고 스프링부트를 예로 들어서 '성능 튜닝' 관련 글에 정리해보도록 한다.

<br>



### 🚀 추가로,

<p align="center">
<img src="https://user-images.githubusercontent.com/59492312/154786799-3d5a6320-20db-4062-be1f-4571df1a84a8.png">
</p>

앞서,HTTP통신이 TCP통신기반이라 얘기했었다. 하지만, HTTP통신과 TCP통신의 차이점도 분명히 존재한다.

**Http통신**  
1. 클라이언트가 요청을 보내는 경우에만 Server가 응답하는 단방향 통신이다.
2. Server로 부터 응답을 받은 후에는 연결이 바로 종료된다.
3. 요청을 보내 server의 응답을 기다리는 어플리케이션의 개발에 주로 사용된다.

**TCP 소켓통신**
1. Server와 Client가 계속 연결을 유지하는 양방향 통신이다.
2. Server와 Client가 실시간으로 데이터를 주고받는 상황이 필요한 경우에 사용된다.
3. 실시간 동영상 Streaming이나 온라인 게임등에 자주 사용된다.

<br>

> 만약, 웹서비스에서 리버스 프록시인 Nginx와 로드벨런서를 사용하는 경우, 브라우저 클라이언트단에서
> 요청을 보낼때는 Http통신을 하고, Nginx에서 이를 받아 WAS로 보낼때는 TCP 소켓(socket)통신을
> 하게 된다.(또한, WAS와 외부api통신, WAS와 RDS통신 모두 소켓통신이다.)

<br>

#### 🪁 References
* 참조링크 : [http통신과 TCP 소켓통신 (1)](https://hwan-shell.tistory.com/271)    
* 참조링크 : [http통신과 TCP 소켓통신 (2)](https://helloworld-88.tistory.com/215)

<br>



태그 : #포트, #소켓, #3-way handshake, #4-way handshake, #time_wait, #HTTP통신, #TCP통신, #TCP 소켓통신, #리눅스커널, #우분투, #센토스, #RHEL, #RFC 793, #MSL
