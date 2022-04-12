---
layout: post
title:  "Beanstalk기반 환경에 Route53을 이용하여 도메인 연결하기"
categories: [ 도메인연결 ]
image: https://user-images.githubusercontent.com/59492312/162870779-50740894-d604-4365-b517-6687dad87c07.png
---

# 📖 Beanstalk기반 환경에 Route53을 이용하여 도메인 연결하기

* route53 호스팅 영역 생성
* 도메인 네임서버 변경
* 호스팅 영역 A레코드 생성

> 모든 코드는 [깃헙](https://github.com/sooolog/dev-spring-springboot)에 작성되어 있습니다.

* * *

<br>



### 1.호스팅영역인 route53 생성하기

<p align="center">
<img src="https://user-images.githubusercontent.com/59492312/153199300-6e5cd361-0d96-4d1b-a0ec-2ca8df85d3e2.png">
</p>

aws 검색란에 route53을 검색하면 위와같은 화면이 나타난다. 우리는 목록에서 호스팅 영역을 클릭해주고, 호스팅 영역을 생성해준다.

<br>

> 여기서 알고가야 할 용어가 있는데, 우리는 지금 DNS 시스템으로 route53을 사용하는거다. DNS는 Domain Name System으 요청받은 도메인 이름을
> 네트워크 주소로 바꾸어주거나, 그 반대의 변환을 시행해주는 시스템이다. 또한, 호스팅영역은 레코드들의 컨테이너이며 레코드에는 특정 도메인과 그 하위 도메인의 트래픽을
> 라우팅하는 방식에 대한 정보가 담겨있다. 뒤에 자세히 설명해놓았으니, 순서대로 읽으면 더 잘 이해가 될것이다.

<br>

> [route53과 DNS](https://wwlee94.github.io/category/blog/aws-eb-https/)    
> [호스팅영역이란](https://docs.aws.amazon.com/ko_kr/Route53/latest/DeveloperGuide/hosted-zones-working-with.html)     

<br>

<p align="center">
<img src="https://user-images.githubusercontent.com/59492312/153199317-ab8976b0-7aef-4d70-b52e-c248b5967f5d.png">
</p>

도메인 이름은 내가 구매한 도메인이나 무료로 발급받은 도메인명을 적어주면 된다.
필자는 test.com이라고 적어놓았지만, 독자분들은 갖고있는 도메인명을 적어주면 된다.

<br>

> 아래 설명 부분은 이름이 같은 호스팅영역을 구분하기 위한것으로 지금 당장은 필요하지않다.
> 또한, 프라이빗 호스팅 영역은 Amazon VPC영역 내에서 트래픽을 라우팅 하는 방식을 의미하기에
> 우리는 퍼블릭 호스팅 영역을 선택해준다.

<br>

<p align="center">
<img src="https://user-images.githubusercontent.com/59492312/153199323-adea2639-a93f-4604-94a1-1b75245751d7.png">
</p>

호스팅 영역 생성을 클릭해주자.

<br>

<p align="center">
<img src="https://user-images.githubusercontent.com/59492312/153206006-94323c3e-79de-4dba-9ffb-378f5bb3ff28.png">
</p>

생성한 호스팅 영역을 들어가보면 위와같은 화면이 나온다.
레코드타입을 보면 NS라고 나오는데, Name Server을 의미하며, DNS(Domain Name Server)와 의미가 같다.
타입이 NS인 부분에 총 4개의 값이 등록되어 있어서 라우팅 대상이 4개나 되는데, 그 이유는 라우팅 대상의 네임서버가
한개밖에 안되고 그 서버가 죽으면 클라이언트들이 우리의 서비스에 들어올 수 없기에 4개까지 지정해준것이다. 이
네임서버를 통해 사용자가 요청을 했을 때 ip정보를 반환해준다.

다음 설명들을 보면 조금 더 쉽게 이해할 수 있다.

<br>

> 위에서는 DNS가 Domain name System이라 했는데, 기본적인 의미는 같으며 S가 server인 DNS는 말 그대로 
> 도메인 이름을 네트워크 주소로 바꿔주거나 그 반대의 역활을 해주는 서버자체를 의미하며, S가 System인 DNS는 
> 그 시스템을 의미한다.

<br>

> 도메인이 이름이 real-test.com인 이유는 test.com은 만들수가 없어서 real-test.com으로
> 도메인명을 입력하여 호스팅 영역을 생성해줬다.

<br>

> [NS타입의 역활과 개념](https://insight-bgh.tistory.com/261)

<br>

<p align="center">
<img src="https://user-images.githubusercontent.com/59492312/153209229-b84834fe-ba4b-4159-8c9b-24a8409f5a54.png">
</p>

위 화면은 필자가 구메한 도메인에서 네임서버를 설정하는 부분이다. 네임서버를 설정하는 곳 에서 우리가 방금 위에서
봤던 aws의 4개의 네임서버값을 호스트명에 입력해야, 클라이언트들이 브라우저에서 도메인명을 입력하였을때 aws의 네임서버가
정상적으로 요청을 받을 수 있는것이다.

<br>

> 필자는 가비아를 사용했지만, 도메인을 구입하건 아니면 무료로 발급을 받아도 네임서버를 설정하는 부분이 있다. 

<br>

> 호스트명이란, IP주소를 대신해서 사용할 수 있는 이름이다.    
> [호스트명, hostname이란](https://www.google.com/search?q=%ED%98%B8%EC%8A%A4%ED%8A%B8%EB%AA%85&rlz=1C5CHFA_enKR982KR982&oq=%ED%98%B8%EC%8A%A4%ED%8A%B8%EB%AA%85&aqs=chrome..69i57j35i39j69i59j46i131i433i512j0i3j69i60j69i61l2.1952j0j7&sourceid=chrome&ie=UTF-8)

<br>

> 자세히 보면 1차,2차,3차는 호스트명은 마지막에 '.'이 찍혀있지 않지만, 4차는 찍혀져있다. 호스팅영역의 NS 값을
> 그대로 복붙해오면 '.'도 같이 복사되니 이 점을 지워주어야 한다.

<br>

<p align="center">
<img src="https://user-images.githubusercontent.com/59492312/153206006-94323c3e-79de-4dba-9ffb-378f5bb3ff28.png">
</p>

레코드타입 SOA는 도메인과 관련된 타이머 설정부분이며 SOA 레코드가 등록되지않을 경우
다른 레코드들을 등록 할 수 없다.

SOA레코드와 NS레코드는 호스팅영역을 생성해주면 AWS에서 자동으로 생성해주는 값들이다.

<br>

> [SOA 레코드에 관하여 (1)](https://server-talk.tistory.com/176)    
> [SOA 레코드에 관하여 (2)](https://insight-bgh.tistory.com/261)

<br>

<p align="center">
<img src="https://user-images.githubusercontent.com/59492312/153537042-fa1c85bd-156d-4456-97fb-55fe8e8d62c6.png">
</p>

다음은, 레코드 생성 버튼을 눌러주면 레코드를 등록할 수 있는 화면이 나온다.
레코드이름은 어느 하위도메인이나, 도메인에 대해 라우팅을 해줄것인지 적는거다. 우리는 우선 도메인에 대한
A레코드 타입으로 진행하도록 하니 레코드 이름에는 아무것도 적어주지 않는다.

<br>

> 도메인은 real-test.com을 의미하며, 하위도메인은 www.real-test.com, m.real-test.com을 의미한다.

<br>

> A레코드 타입이란, 지정된 레코드이름으로 즉, 지정된 도메인명으로 요청이 들어온다면, 지정한 IP 주소로 
> 연결되게 하는 방식을 의미한다. 우리는 빈스톡 환경에서 만든 로드벨런서로 라우팅 값을 지정할것이다.

<br>

트래픽 라우팅 대상에는 내가 라우팅해줄 ip나 그 외에 값들을 입력해주어야 하는데 우리는 A레코드 타입이며,
빈스톡 배포에 사용한 로드벨런서로 값을 지정해줄것이다.

<br>

> Elastic Beanstalk환경에 대한 별칭을 선택하지 않고, application/Classic Load balancer을 선택해준 이유는
> 뒤에 ACM SSL 인증서(https)를 로드벨런서에 연결할것인데, 여기서 A레코드값을 로드벨런서로 지정해야 정상적으로 작동하기 때문이다. 
> 다음 빈스톡과 ssl설정 작성글에서 더 자세하게 다룰것이다.    
> [ACM SSL인증서를 로드벨런서에 연결](https://aws.amazon.com/ko/premiumsupport/knowledge-center/associate-acm-certificate-alb-nlb/)

<br>

<p align="center">
<img src="https://user-images.githubusercontent.com/59492312/153540770-fd3e359e-7aa9-4768-ac81-86d494dd0408.png">
</p>

마지막으로 볼 항목은 라우팅 정책이다.   
각각의 라우팅 정책에 보고가겠다. 실제 사용을 위해선 더 자세히 다루어야 하지만, 우리는 뒤에 나올 개념들을 위한
기본기를 다지는 정도로만 하겠다. (우리는 이번 프로젝트에서는 단순라우팅 정책을 사용할것이다.)

* **단순 라우팅** : 다른 라우팅처럼 가중치나 지연시간을 고려하지 않고 우리가 지정한 리소스(로드벨런서)
로 트래픽을 라우팅하는 방법을 말한다. 일반적으로 가장 많이 사용하는 라우팅 정책이다.

* **가중치 기반 라우팅** : 다수의 리소스(EC2, 로드벨런서)를 단일 도메인이나 하위 도메인이름과 연결하여
각 리소스로 라우팅되는 트래픽 비율을 선택해서 보낼 수 있게된다.

<br>

> 이 방식을 사용하면, 하나의 로드벨런서가 아니라 여러 로드벨런서로 다중 분산을 시킬 수 있다.

<br>

* **지리적 위치 라우팅** : 사용자 즉, 클라이언트의 위치(DNS 쿼리가 발생하는)를 기반으로 라우팅할
리소스(로드벨런서)를 지정할 수 있다. 클라이언트의 각 위치마다 서비스를 제공할 수 있어 글로벌 서비스에 특화되어 있는 
정책이다. 예를 들면, 유럽에서 발생하는 모든 쿼리를 프랑크푸르트 리전에 위치한 ELB 로드벨런서로 라우팅할 수 있다. 지리적 라우팅을 사용하는 경우
콘텐츠를 지역화하고 웹사이트의 일부 또는 전체를 사용자의 언어로 제공할 수 있게된다. 

<br>

> 하나의 도메인 google.com이나 Netflix.com에서 같은 도메인이지만 지역별로 다른 언어와 서비스들을 보여줄 수 
> 있게된다. 위치별로 어느 리전의 aws 리소스로 라우팅할지 정할 수 있기 때문이다.

<br>

* **지연시간 라우팅** : 여러 리전의 리소스를 사용하는 경우 클라이언트의 DNS 쿼리 요청이 들어오면 지연 시간이 가장 낮은 AWS리전의
리소스로 요청을 라우팅하는것을 말한다. 이 경우는 다중 리전을 사용하는 사용자에게 제공하는 서비스이다.

* **장애 조치 라우팅** : 첫번째, 두번째 리소스를 정해두고 첫번째 리소스가 비정상일 경우 두 번째 리소스로 라우팅한다. 이 정책은
첫번째 리소스가 비정상일 경우에만 두번째 리소스로 라우팅을 하기 때문에, 가중치 기반 라우팅이랑은 차이가 있다.

* **다중값 응답 라우팅** : DNS 쿼리에 대해 다수의 값(서버의 IP주소)을 반환하도록 구성할 수 있다. 하지만
레코드 타입을 지정할 때 CNAME, NS를 지원하지 않아서 ALB(Application LoadBalancer)등의
값을 줄 수 없고 오로지 IP주소나 그 외의 값으로만 지정이 가능하다.

<br>

> 직접 사용하는 단순 라우팅외에는 실제 사용하는경우에 자세히 보도록 하겠다. 더 자세한 내용은 아래 참조링크들을
> 참고하면 된다.

<br>

> [Route53 라우팅 정책 (1)](https://docs.aws.amazon.com/ko_kr/Route53/latest/DeveloperGuide/routing-policy.html)    
> [Route53 라우팅 정책 (2)](https://velog.io/@pgh8019/AWS-Route-53-Routing-Policy%EB%A5%BC-%EC%95%8C%EC%95%84%EB%B3%B4%EC%9E%90)    
> [Route53 라우팅 정책 (3)](https://gooners0304.tistory.com/entry/Route-53-DNS-%EB%A0%88%EC%BD%94%EB%93%9C-%EB%93%B1%EB%A1%9D%ED%95%98%EA%B8%B0)

<br>

추가로, 라우팅 정책 옆에 대상 상태평가라는 란이 보인다.
우리가 사용하는 로드벨런서로 얘기를 해보자면, 로드벨런서의 여러 인스턴스 중에
하나라도 정상 인스턴스가 있으면, 상태평가에서 해당 로드벨런서를 정상이라 표시되고
로드벨런서 내에 모든 인스턴스가 비정상이면 해당 로드벨런서로 비정상으로 표시된다. 또한,
로드벨런서내에 인스턴스가 없는 경우도 비정상으로 표시된다.

<br>

> [Route53 대상 상태 평가](https://aws.amazon.com/ko/premiumsupport/knowledge-center/load-balancer-marked-unhealthy/)

<br>

<p align="center">
<img src="https://user-images.githubusercontent.com/59492312/153740934-43bc20a9-f158-47c0-9bb8-2444949a771d.png">
</p>

route53으로 도메인 설정이 완료됬으니, real-test.com으로 브라우저에서 접속해보면 정상적으로
화면이 뜨는걸 볼 수 있다.(필자는 반환되는 String값으로 haha nwe라고 적어놓은것이다.)

<br>



### 2.Route53은 리전별 서비스가 아니다.

<p align="center">
<img src="https://user-images.githubusercontent.com/59492312/153741283-fea2ab1c-c46e-4950-b7a2-ff71a7b00b52.png">
</p>

aws route53은 위의 그림과같이 리전이 글로벌로 뜬다. 즉, 리전한정 리소스가
아니란것을 염두해 둘 필요가 있다. 이 부분에 더 궁금하다면 route53과 레코드가 어떻게
작동하는지 알아본다면 더 자세히 알 수 있다.

<br>



### 🚀 추가로,

<p align="center">
<img src="https://user-images.githubusercontent.com/59492312/153552296-134c0c66-59a6-4a26-a9d7-0623c6ed82f6.png">
</p>

위에서는 별칭값을 선택했기 때문에 안나왔지만, TTL이라는 속성이 하나 더 있다.
TTL이란, DNS resolver가 이 레코드에 관한 정보를 캐싱할 시간(초)이다.
조금 더 쉽게 이해가 되기 위해 아래 그림을 보겠다.

<br>

> [TTL이란](https://docs.aws.amazon.com/ko_kr/Route53/latest/DeveloperGuide/resource-record-sets-values-basic.html#rrsets-values-basic-ttl)

<br>

<p align="center">
<img src="https://user-images.githubusercontent.com/59492312/153543841-c6377ecb-88be-4037-9fb8-b38a80b8a8fa.png">
</p>

즉, TTL을 설정해 놓으면 위의 그림과같이 DNS resolver에서 해당 레코드에 대한 정보를
캐싱하기 때문에, 캐싱된 레코드는 요청이들어오면 지연 시간을 줄일 수 있고 Route 53 서비스 비용 또한 줄이는 효과가 있다.
(TTL을 이용한 캐싱을 할경우 DNS resolver에서 같은 도메인에 대한 IP주소를 기억해서 캐싱된 도메인에 대한
DNS 쿼리 요청을 받으면 바로 IP주소를 반환하게 되는것이다.)

<br>

> 하지만, 만약 특정 레코드에 대한 라우팅 값이 바뀌면 TTL시간을 줄여주어서 새로운 라우팅값이
> 반영되도록 해야하는것도 알아두자.

> 추가로, 별칭을 지정해주면, TTL을 입력하는 부분이 사라진다.
> 그 이유는, 별칭 레코드가 AWS 리소스(라우팅 값)를 가리키는 경우 
> 유지 시간(TTL)을 설정할 수 없고. 라우팅 값으로 입력한
> aws 리소스내에서 유지 시간(TTL)을 사용하여 설정한다. 그래서 로드벨런서로 라우팅값을
> 지정해줄때 TTL 속성란이 보이지 않았던거다.    
> [레코드 설정시 별칭은 TTL 속성이 없는이유](https://docs.aws.amazon.com/ko_kr/Route53/latest/DeveloperGuide/resource-record-sets-choosing-alias-non-alias.html)

<br>

> [Route53과 DNS resolver 그리고 도메인 관계](https://docs.aws.amazon.com/ko_kr/Route53/latest/DeveloperGuide/welcome-dns-service.html#welcome-dns-service-how-route-53-routes-traffic)

<br>

<p align="center">
<img src="https://user-images.githubusercontent.com/59492312/153557545-e4ef8dc2-2a25-4f75-a575-f192a15b4a29.png">
</p>

마지막으로, DNS 쿼리의 개념에 대해서도 보고가도록 하자.
바로 전 위의 그림 에서 보다시피, DNS 쿼리는 특정 도메인의 IP를 조회하기 위한 
DNS resolver로 보내는 요청이다. 

<br>

> [DNS쿼리 개념과 요금, 호스팅영역 요금](https://jw910911.tistory.com/92)

<br>



태그 : #호스팅영역, #Domain Name System, #Domain Name Server, #NS, #호스트명, #하위도메인, #라우팅정책, #대상상태평가, #TTL, #DNS resolver, #DNS 쿼리
