<p align="center">
<img src="https://user-images.githubusercontent.com/59492312/166184112-67d8e456-bad2-46c0-8df5-01528b404275.png">
</p>

# 📖 Beanstalk기반 환경에 도메인 SSL 설정하기

* HTTPS 작동원리와 SSL/TLS 인증서의 개념
* ACM에서 SSL 인증서 발급받기
* 빈스톡(Beanstalk) 환경 구성에 SSL/TLS 인증서 등록하기

> 모든 코드는 [깃헙](https://github.com/sooolog/dev-spring-springboot)에 작성되어 있습니다.

* * *

<br>



### 1.HTTPS 작동원리와 SSL/TLS 인증서의 개념

<p align="center">
<img src="https://user-images.githubusercontent.com/59492312/166401664-c65fc2ba-6ce0-4490-a021-3134a550e923.png">
</p>

HTTPS란 무엇이고 SSL 인증서란 무엇일까 ?     
이 글을 읽고나면 빈스톡기반 서비스의 도메인에 https가 적용되는것을 볼 수 있을것이다.
다만, 기본적인 원리와 개념들은 알고있어야 후에 이를 응용하거나 아니면 리다이렉트(http -> https)에 대한
개념을 볼 때 더 잘 이해할 수 있으니 꼭 읽어보고 가도록 하자.

* HTTPS : 말 그대로 기존 HTTP에서 보안이 더 강화된 버전이다. 조금 더 쉽고 자세하게 얘기하자면, 인증된 서버에 한해서
  데이터를 전송할 때 암호화를 적용하여 진행한다. 아래 예시를 보면 완전히 이해할 수 있다.

* SSL/TLS 인증서 : 클라이언트와 서버 간의 통신의 안정을 보증해주는 문서이다. 조금 더 쉽게 설명하자면 클라이언트(브라우저)에서
  서버에 데이터를 보내려하는데 해당 서버가 신뢰할 수 있는 서버인지 확인하려 할 때 이 SSL/TLS 인증서를 확인하게 된다. 이 또한 아래
  구체적인 예시를 보면서 다시 한번 설명하겠다.

> 우리가 HTTPS 연결을 위해 가장 많이 들어본 인증서는 SSL(Secure Socket Layer) 인증서이다. 그렇다면, TLS(Transport Layer Security) 인증서는 
> 무엇일까? TLS는 SSL의 업데이트된 버전이며 명칭만 바뀐것이다. 현재 우리가 사용하는 인증서는 이 둘을 함께 사용하고 있다. 우리가 앞으로 받을 인증서도 SSL/TLS 인증서이다.
> 하지만, SSL/TLS 인증서나 TLS인증서를 모두 그냥 SSL 인증서라고 혼용해서 부르기도 한다.(하지만, SSl과 TLS는 기능적인 면에서 차이점이 존재한다. 자세한 내용을 알고싶다면 
> SSL과 TLS의 차이점에 대해 찾아보길바란다.)    
> [SSL과 TLS에 관하여 (1)](https://kanoos-stu.tistory.com/46)     
> [SSL과 TLS에 관하여 (2)](https://ssungkang.tistory.com/entry/Web-SSL-%EC%9D%B8%EC%A6%9D%EC%84%9C%EC%97%90-%EB%8C%80%ED%95%9C-%EC%9D%B4%ED%95%B4%EC%82%AC%EC%A0%84-%EC%A7%80%EC%8B%9D-%EC%A0%95%EC%9D%98-%EB%8F%99%EC%9E%91%EC%9B%90%EB%A6%AC-%EC%9D%B8%EC%A6%9D%EC%84%9C-%EB%B9%84%EA%B5%90)    

<br>

<p align="center">
<img src="https://user-images.githubusercontent.com/59492312/166401675-33281ada-1054-432a-a193-176003d196b2.png">
</p>

브라우저, 인증기관 그리고 서버와의 관계에 대해 알아보기전에 다른 예시를 보고 가겠다.

성인이되면 동사무소에서 주민등록증을 발급한다. 술집에서는 발급된 주민등록증을 확인하고 술을 주는데
이 주민등록증이 과연 진짜 주민등록증인지 확인을 해야한다. 술집은 주민등록증을 발급해주는 동사무소에 연락을
하여(실제로는 해당 주민등록증의 지문을 단말기로 대조 한다고 한다.) 진짜 주민등록증인지 확인을 하고 손님에게
최종적으로 술을 건내주게 된다.

HTTPS보안과 SSL/TLS인증서 서버와 브라우저사이의 관계도 이와 같다.

<br>

<p align="center">
<img src="https://user-images.githubusercontent.com/59492312/166401683-9abcb994-bc80-4dd7-9281-48046b159e6f.png">
</p>

서버는 자신의 서버가 신뢰할만한 사이트인지를 알려주기위해 SSL/TLS 인증서를 발급받는다.
해당 SSL/TLS 인증서는 인증기관(CA)에서 발급받아 사용하게된다. 그리고나서 브라우저(클라이언트)가 해당 사이트의 서버에
접속하려 할 때 이 서버가 안전하고 신뢰할만한지 알기 위해서는 해당 서버가 발급받은 SSL/TLS인증서를 확인하고 또한 이 인증서를
발급해주는 인증기관(CA)에 이 인증서가 진짜인지 확인 과정을 거치게 된다. 최종적으로 신뢰할만한 인증서라는 확인을 거치게 되면
브라우저는 이제 서버에 보낼 데이터들에 대해 어떻게 암호화를 할것인지 서버와 함께 협상을 진행 하게 된다.

정리하자면, 클라이언트(브라우저)가 HTTPS를 사용하게 되면 신뢰할만한 서버에만 데이터를 보내게 되고, 데이터를 보낼때도
암호화를 설정하여 데이터를 보내게 된다. SSL/TLS 인증서는 서버가 자신이 신뢰할만하고 안전한 서버인지를 보여주는것이다.

기본적인 개념과 원리에 대해 알았으니 이제는 실제로 빈스톡 환경 기반의 도메인에 SSL/TLS 인증서를 발급받고
HTTPS가 정상적으로 작동하도록 설정해 보겠다.

> [HTTPS와 SSL 그리고 브라우저(클라이언트)와 서버의 작동원리](https://aws-hyoh.tistory.com/entry/HTTPS-%ED%86%B5%EC%8B%A0%EA%B3%BC%EC%A0%95-%EC%89%BD%EA%B2%8C-%EC%9D%B4%ED%95%B4%ED%95%98%EA%B8%B0%EC%9A%B0%EB%A6%AC%EB%8A%94-%EA%B5%AC%EA%B8%80%EC%97%90-%EC%96%B4%EB%96%BB%EA%B2%8C-%EB%93%A4%EC%96%B4%EA%B0%80%EB%8A%94%EA%B0%80)

<br>



### 2.ACM에서 SSL/TLS 인증서 발급받기

<p align="center">
<img src="https://user-images.githubusercontent.com/59492312/166397052-138ae7c6-df2a-4216-8ce9-2ccbe0f6c833.png">
</p>

AWS의 ACM(AWS Certificate Manager)을 접속해보면 위와같은 화면이 나온다. 
이곳에서 우리는 SSL/TLS 인증서를 요청하고 발급받을 것이다. 요청을 클릭해보자.

> ACM(AWS Certificate Manager)에 SSL/TLS 인증서를 요청하면, Amazon의 인증기관(CA)인 
> Amazon Trust Services (ATS)에서 발급이 된다.    
> [SSL/TLS 인증서와 Amazon 인증기관(CA)](https://aws.amazon.com/ko/blogs/korea/new-aws-certificate-manager-deploy-ssltls-based-apps-on-aws/)

<br>

<p align="center">
<img src="https://user-images.githubusercontent.com/59492312/166397059-6fbb60b0-7218-46ba-98ca-1e875424a81e.png">
</p>

퍼블릭 인증서 요청을 클릭해준다.

> 프라이빗 인증서를 사용하려면 프라이빗 CA(Certificate Manager)를 생성해야 한다. 프라이빗 CA는
> AWS 클라우드 내부에 PKI (퍼블릭 키 인프라) 를 구축하는 기업 고객을 대상으로 하며 조직 내에서 비공개로 
> 사용할 수 있도록 고안된 서비스이다. 사설 CA를 생성하면 외부 CA에서 유효성 검사를 받지 않고 직접 인증서를 발급하고 
> 조직의 내부 요구 사항에 맞게 사용자 지정하여 발급하고 사용할 수 있다. 간단하게 말하면, 기업 내부에서 사용하기 위한 목적이며
> 나중에 실제 사용시 더 자세하게 봐도 무방하다.     
> [ACM의 사설 CA와 프라이빗 인증서](https://docs.aws.amazon.com/ko_kr/acm-pca/latest/userguide/PcaWelcome.html)

<br>

<p align="center">
<img src="https://user-images.githubusercontent.com/59492312/166397062-a6b3fdc5-1e59-4989-a321-4d3a990be7f5.png">
</p>

도메인 이름에는 인증서에 포함시키고 싶은 도메인명을 적어주면 된다.

도메인.com으로 기본 도메인 형식으로 추가해줄 수 있고, *.도메인.com으로 해서 해당 도메인의 
모든 서브 도메인을 인증서에 포함시킬수도 있다. 혹은 www.도메인.com으로 해서 www.도메인.com 
서브 도메인만 인증서에 포함시킬수도 있다.

그 외에도, 꼭 A도메인.com을 추가했다고 해서 A도메인의 서브도메인만 추가가 가능한것은 아니다.
A도메인.net이나, B도메인.com도 인증서에 한번에 추가가 가능하다.

> 완전히 정규화된 도메인 이름(FQDN)이란 간단하게 얘기하면 .com 또는 .org와 같은 최상위 도메인 확장명이 
> 마지막에 위치한 도메인을 의미한다.(더 자세히 알고싶다면 FQDN에 관해 검색해보자.)    
> [FQDN이란](https://ko.eyewated.com/fqdn%EC%9D%80-%EB%AC%B4%EC%97%87%EC%9D%84-%EC%9D%98%EB%AF%B8%ED%95%A9%EB%8B%88%EA%B9%8C/)

> [도메인 이름에 도메인명 추가 (1)](https://wwlee94.github.io/category/blog/aws-eb-https/)    
> [도메인 이름에 도메인명 추가 (2)](https://comocode.tistory.com/28)    
> [도메인 이름에 도메인명 추가 (3)](https://aws.amazon.com/ko/premiumsupport/knowledge-center/acm-add-domain-certificates-elb/)

<br>

<p align="center">
<img src="https://user-images.githubusercontent.com/59492312/166397063-4a7ee18e-682e-4312-8cc9-33fe2d67cbd6.png">
</p>

검증 방법 선택은 DNS검증과 이메일 검증이 있다.

DNS 검증방법은 Route53 호스팅영역에서 추가한 도메인에 대해 CNAME레코드를 생성하여 검증을 진행한다.
검증을 한다는것은 해당 도메인의 소유 여부와 이 도메인과 연결된 DNS(Domain Name Server)를 제어할 수 있는지를 체크한다.
DNS 검증을 시작하게 되면, 본인이 입력한 도메인과 매칭되는 Route53의 호스팅 영역에서 CNAME이 추가되어서 검증을 진행하게 된다.
CNAME이 추가되면, 해당 CNAME 레코드 이름으로 요청이 들어오게 되고 값/트래픽 라우팅 대상으로 정상적으로 요청이 전달되면 해당 도메인의 소유권과
연동된 DNS에 대한 제어권이 검증된다.

이메일 검증방법은 해당 도메인을 구매할 당시에 입력한 이메일로 검증 메일이 보내지게 된다. 
이 방법으로 도메인에 대한 소유권을 검증하게 된다.

우리는 DNS검증 방법으로 진행하도록 하겠다. DNS 검증을 클릭하고 요청을 눌러주자.

> 이메일 검증 인증서는 최초 검증일로부터 최대 825일까지만 갱신할 수 있다. 825일이 지나면 도메인 소유자가 새 인증서를 다시 요청해야 한다. 
> 그러나, DNS 검증 인증서는 자동으로 갱신할 수 있다. 즉, 한번만 등록하면 문제가 없는한 AWS에서 자동으로 갱신을 해준다.    
> [이메일 검증과 DNS 검증의 차이](https://skyksit.tistory.com/entry/AWS-%EC%9D%B8%EC%A6%9D%EC%84%9C-%EB%B0%9C%EA%B8%89%EB%B0%9B%EA%B8%B0-2-%EB%A9%80%ED%8B%B0-%EA%B3%84%EC%A0%95%EC%97%90%EC%84%9C-%EC%97%AC%EB%9F%AC%EA%B0%9C%EC%9D%98-%EC%84%9C%EB%B8%8C%EB%8F%84%EB%A9%94%EC%9D%B8-%EC%82%AC%EC%9A%A9%ED%95%98%EA%B8%B0)

> Amazon ACM에 요청하여 발급해주는 SSL/TLS 인증서는 무료이다. 다만, DNS검증을 통하여 인증서를 발급받으려면 AWS DNS(Domain Name Server)인 Route53을 사용해야 한다. 
> 즉, 소유하고 있는 도메인의 DNS로 Route53을 이용하여 미리 연결된 상태여야 인증서에 등록하려는 도메인에 대한 검증이 가능하다. 만약, 아직 연결해놓지 않았다면 필자가 적어놓은 [Beanstalk기반 환경에 Route53을 이용하여 도메인 연결](https://sooolog.dev/Beanstalk%EA%B8%B0%EB%B0%98-%ED%99%98%EA%B2%BD%EC%97%90-Route53%EC%9D%84-%EC%9D%B4%EC%9A%A9%ED%95%98%EC%97%AC-%EB%8F%84%EB%A9%94%EC%9D%B8-%EC%97%B0%EA%B2%B0%ED%95%98%EA%B8%B0/)
> 를 참고하도록 하자.     
> [ACM에서 발급해주는 SSL/TLS 인증서](https://wwlee94.github.io/category/blog/aws-eb-https/)    

> [이메일 검증과 DNS 검증 (1)](https://comocode.tistory.com/28)    
> [이메일 검증과 DNS 검증 (2)](https://jojoldu.tistory.com/434)    

<br>

<p align="center">
<img src="https://user-images.githubusercontent.com/59492312/166873864-09228e09-7b21-4eb6-920e-4e0228003dc7.png">
</p>

요청을 클릭하고 나면 인증서 목록에서 내가 방금 요청한 인증서를 클릭해준다.
그러면 위 이미지처럼 검증 대기중 상태로 표시가 되고, 아래에 도메인(6)과 "Route 53에서 레코드 생성"
클릭 아이콘도 보인다.

이 화면의 의미는 아직 도메인 혹은 DNS에 대한 검증(우리는 DNS 검증을 선택했다.)이 완료되지 않은 상태이며 
도메인(6)의 의미는 우리가 방금전 SSL/TLS 인증서에 등록하려한 도메인의 개수를 의미한다.(필자는 위에서 6개 도메인에 대한
인증서 등록을 하려했으니 6개로 표시되는 것이다.)

여기서 "Route 53에서 레코드 생성" 버튼을 눌러주자. 

<br>

<p align="center">
<img src="https://user-images.githubusercontent.com/59492312/166397067-80d1c598-ae7a-40a0-9c3b-3777afdf07bb.png">
</p>

그러면, 도메인별로 유형이 CNAME인 목록들이 나열된다.

이 목록들에 대한 레코드 생성의 의미는 앞서말했던, DNS 검증의 방법으로 SSL/TLS 인증서에 등록하려는
도메인과 연동된 Route53 호스팅영역에 CNAME 유형의 레코드를 생성하는 방식이다. 이렇게 레코드를 생성한 후,
해당 CNAME 이름으로 요청이 보내지고, CNAME 값으로 요청이 정상적으로 전달이 되면 검증이 완료된다.

이렇게 함으로써 도메인 소유권에 대한 검증과 DNS 제어권에 대한 인증이 검증된다.
해당 도메인들에 대한 각각의 검증이 완료되야 SSL/TLS 인증서가 발급이 된다. 

우측 하단의 "레코드 생성" 버튼을 눌러준다.

> 도메인마다 검증하기 위한 CNAME 레코드 목록들은 직접 Route53의 호스팅 영역에 추가해주어도 된다. 하지만, 위에 처럼
> "Route53에서 레코드 생성" 버튼을 누르고 레코드를 생성해주면 직접 해당 CNAME 레코드들을 추가해줄 필요없이 자동으로
> 추가해준다.    
> [Route53에 검증용 CNAME 자동추가](https://wwlee94.github.io/category/blog/aws-eb-https/)

> 추가로, 보면 알겠지만 www.도메인A.com, 도메인A.com에 대한 검증은 서로 다른 CNAME명과 CNAME값이 적용되어서 호스팅영역에 레코드가
> 생성되게 된다. 하지만, 도메인B.com과 *.도메인B.com은 서로 같은 CNAME명과 CNAME값을 갖은걸 알 수 있다. 또한 알면 재밌는점이 필자는 
> m.도메인C.com, *.도메인C.com, 도메인C.com 도메인들에 대해 SSL/TLS 인증서 발급을 신청했고, 호스팅 영역은 도메인C.com과 m.도메인C.com
> 두개를 만들었는데 정작 CNAME 레코드는 도메인C.com에만 생성되었다는것이다.(m.도메인C.com 호스팅영역에는 아무 레코드도 생성되지 않았다.)

<br>

<p align="center">
<img src="https://user-images.githubusercontent.com/59492312/166397069-7d05d580-cd76-421a-89e2-d43bd7acceb2.png">
</p>

조금만 시간이 지난 후, 다시 해당 인증서를 들어가보면 이렇게 신청한 도메인들에 대해
성공이라고 뜬것을 볼 수 있다. 그럼 우리는 신청한 도메인에 대한 SSL/TLS 인증서 발급을
완료한것이다.

> 헷갈리면 안되는것이 ACM에서는 AWS의 CA에서 인증서를 발급받기 위한 요청만 할 뿐이다. 또한, 도메인을 입력하였던것도
> 발급받으려는 SSL/TLS 인증서에 해당 도메인을 추가하기 위함이다. 즉, 인증서만을 발급받기 위한 과정이지 해당 도메인들에 대해
> 직접적으로 어떠한 설정이 이루어진것은 아니다.

> [ACM인증서 발급 (1)](https://wwlee94.github.io/category/blog/aws-eb-https/)    
> [ACM인증서 발급 (2)](https://comocode.tistory.com/28)      

<br>



### 3.빈스톡(Beanstalk) 환경 구성에 SSL/TLS 인증서 등록하기

<p align="center">
<img src="https://user-images.githubusercontent.com/59492312/166936544-3def4f69-1ef9-49e5-a2dc-c6070510bfcc.png">
</p>

이제는 빈스톡의 로드밸런서에 SSL/TLS 인증서를 등록할 차례이다.

빈스톡 환경으로 이동해주면 왼쪽에 "구성"을 클릭해준다.

<br>

<p align="center">
<img src="https://user-images.githubusercontent.com/59492312/166936558-e5f5a0ce-f39a-4830-9902-532d0b31742a.png">
</p>

이후 구성 요소들중에 "로드 밸런서"의 "편집" 아이콘을 클릭해준다.

<br>

<p align="center">
<img src="https://user-images.githubusercontent.com/59492312/166936562-51e8a66b-f162-438a-a812-f0d5c847dd34.png">
</p>

이곳에서 "리스너 추가"버튼을 눌러주자.

> 로드 밸런서는 클라이언트 요청을 리소스(EC2 등등)로 라우팅 하는 역활을 하는데, 이 때 리스너(listener)는 지정된 포트로 들어오는 요청만을
> 받기 위해 포트 번호를 설정하거나 아니면 받은 요청에 대해 라우팅할 때 프로토콜도 설정할 수 있다. 기본적으로 로드 밸런서는 포트 80를 기본 리스너로 지정한다. 의
> 이곳에서 443 포트에 대해 리스너 추가를 해주어야 로드 밸런서로 HTTPS 요청이 들어왔을때 정상적으로 EC2로 라우팅할 수 있게 해준다.

<br>

<p align="center">
<img src="https://user-images.githubusercontent.com/59492312/167127086-b4aaf3af-0e32-472b-b4f9-91f6d1aef560.png">
</p>

Application Load Balancer 리스너를 설정하여 추가할 수 있는 화면이 나온다.

포트는 443으로 설정해준다. 443 포트를 설정함으로써 HTTPS 요청도 받을 수 있도록 한다.
프로토콜은 로드 밸런서가 받은 요청을 다시 라우팅해서 보내줄 때 사용되는 규약(프로토콜)을
의미하는것이므로, HTTPS로 설정해주자.(우리는 EC2를 사용하니, EC2로 라우팅될 때의 프로토콜을 설정하는 것이다.)

SSL인증서는 우리가 방금전 ACM에서 요청하여 발급받은 SSL/TLS 인증서를 선택해주면 된다.
이는 우리가 발급받은 SSL/TLS 인증서를 로드 밸런서에 등록하는것이다. 추후에 브라우저(클라이언트단)가
HTTPS 통신으로 서버의 인증서를 확인하려 할 때 이 로드밸런서의 인증서를 확인하게 된다.

SSL 정책은 간단히 말하면, 클라이언트와 로드 밸런서간의 HTTPS 통신에서 어떠한 암호화를
협상하여 정할지 기준이 되는 정책이다. 여러 보안 정책들이 있지만, 호환성을 위해 ELBSecurityPolicy-2016-08 정책을 
사용하는 것을 AWS에서는 추천한다.(ELBSecurityPolicy-2016-08가 default이나 ELBSecurityPolicy-TLS-1-1-2017-01과 같은
보안 정책을 선택해도 HTTPS가 정상적으로 작동하긴한다.)

> 필자는 이전에 [Beanstalk 환경 기반에서 Route53을 이용한 도메인 연결]()을 작성한적이 있다. 그곳에서 별칭으로 "트래픽 라우팅 대상"을
> Elastic Beanstalk환경을 선택하지 않고 로드밸런서를 선택하여 설정을 해준적이 있다. 이러한 이유가 바로 로드 밸런서에 SSL/TLS 인증서를 등록해주기
> 때문에 라우팅 대상 값이 로드 밸런서이여야 정상적으로 SSL/TLS 인증서를 인식하고 HTTPS 통신을 할 수 있기 때문이다.

> [SSL 정책에 관하여 (1)](https://docs.aws.amazon.com/ko_kr/elasticloadbalancing/latest/application/create-https-listener.html#describe-ssl-policies)    
> [SSL 정책에 관하여 (2)](https://comocode.tistory.com/28)    

<br>

<p align="center">
<img src="https://user-images.githubusercontent.com/59492312/166937097-3e9eb21a-201f-4eea-a366-57425b73982c.png">
</p>

끝으로, 리스너를 추가했으면 아래 "적용" 아이콘을 클릭해주자.

잠시 후, 환경 재설정 시간이 지나고 인증서에 등록한 도메인으로 HTTPS 접속을 시도해보면
정상적으로 적용이 되는것을 볼 수 있다.

<br>



### 🚀 추가로,

<p align="center">
<img src="https://user-images.githubusercontent.com/59492312/166397062-a6b3fdc5-1e59-4989-a321-4d3a990be7f5.png">
</p>

ACM에서 SSL/TLS 인증서에 등록하기 위한 도메인을 입력하는 화면이다.
이곳에서 입력을 안하고 그냥 지나가면 나중에 SSL/TLS 인증서가 발급되었을 때
도메인 추가하거나 혹은 삭제할 수 없다. 만약에 추가하거나 삭제하려면 다시 ACM에서
SSL/TLS 인증서 발급을 요청해야하니 이 점에 주의하도록 하자.

<br>



태그 : #HTTPS, #SSL/TLS 인증서, #인증기관(CA), #ACM, #FQDN, #Listener, #리스너, #보안정책
