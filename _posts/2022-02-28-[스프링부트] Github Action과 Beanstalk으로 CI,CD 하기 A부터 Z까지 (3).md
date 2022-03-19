---
layout: post
title:  "[스프링부트] Github Action과 Beanstalk으로 CI/CD 하기 A부터 Z까지 (3)"
categories: [ 빌드와 배포 ]
image: <img width="751" alt="Frame 3" src="https://user-images.githubusercontent.com/59492312/159111002-5cf463d0-775a-4437-9e48-7252fe82180a.png">
---

* 빈스톡 환경 생성
* IAM 발급
* 깃헙에 IAM 설정

> 모든 코드는 [깃헙](https://github.com/sooolog/dev-spring-springboot)에 작성되어 있습니다.
* * *



<br>

### 1.Benastalk 사용을 위한 환경 생성하기

<p align="center">
<img src="https://user-images.githubusercontent.com/59492312/151652653-19bdcb04-9460-44d8-a580-4c82b28c4dab.png">
</p>

빈스톡 사용을 위한 환경생성을 진행해 본다. aws 검색창에 beanstalk를 치면 Elastic Beanstalk가 나온다.

<br>

<p align="center">
<img src="https://user-images.githubusercontent.com/59492312/151652649-d30182dd-8d28-411e-bb01-146cd2bd741e.png">
</p>

그 후 환경생성을 누르고 나면, 바로 환경 티어 선택이라는 창이 나온다.      
웹 서버 환경은 기존 EC2 환경을 그대로 사용하는것으로 일반적인 웹 서버를 사용하려 할 때 선택해주면 된다. 즉, 우리도
웹 서버 환경을 선택해서 진행해주면 된다.  

그렇다면, 작업자 환경은 무엇일까?   
웹 서버 환경에 메세지 큐 (SQS) 수신이 가능한 리스너 데몬이 별도 구축된 환경으로
보통의 HTTP API 처리가 아닌 프로세스들(배치/메시지워커 등)을 위한 환경이다. 

<br>

> 작업자 환경은 후에 SQS를 구축하여 서버를 사용하고자 할 때 다시보면 된다.

<br>

<p align="center">
<img src="https://user-images.githubusercontent.com/59492312/151652647-09aa657b-819d-4a6b-b7aa-b55243c3cadb.png">
</p>

애플리케이션의 이름은 필자는 보통 프로젝트의 이름과 같게한다. 여기서는 연습용 환경구축이니 필자는
real-test라 했지만, 이 글을 읽는 독자분들은 프로젝트 이름으로 써주거나 원하는 이름으로 어플리케이션 이름을
적어주면 된다.

<br>

<p align="center">
<img src="https://user-images.githubusercontent.com/59492312/151652640-306072d1-c9e1-476a-96a9-82cd6d8a57f8.png">
</p>

1개의 환경은 1개의 빈스톡이라고 보면된다. 규모가 큰 프로젝트의 경우 하나의 어플리케이션에 여러 환경을 구축하여 사용하기에
보통은 환경과 어플리케이션을 구분지어 놓는다. 그렇기에, Realtest-env나 Realtest-env1로 써주도록 하자. 도메인은 후에 빈스톡을
사용하여 ec2에 배포하는 경우, 배포 내용을 볼 수 있는 임시 도메인으로 봐도 된다.

도메인은 아무것도 채워넣지않아도 자동으로 채워지니 비워두고 그 다음 진행하도록 하겠다.

<br>

<p align="center">
<img src="https://user-images.githubusercontent.com/59492312/151652638-da74cfa1-514b-47a1-adb7-fec6d9bcf6bf.png">
</p>

Corretto는 AWS에서 지원하는 무료로 사용할 수 있는 Open Java Development Kit(OpenJDK)의 멀티플랫폼 배포판이다.
즉, Corretto 8은 자바 8기반의 멀티플랫폼 배포판인것이다. 일반 자바 8 SDK도 선택해서 사용할 수 있지만, deprecated된다고 되어있으니
Corretto 8을 이용하도록 하겠다. 

<br>

> 필자는 자바8로 프로젝트를 진행하지만 자바11이 익숙한 독자는 Corretto 11이나 자바11로 진행해주면 된다.

<br>

<p align="center">
<img src="https://user-images.githubusercontent.com/59492312/151652657-a8011dd1-fe08-4382-bfd2-922866e35c8c.png">
</p>

샘플 어플리케이션을 선택하여 진행하도록 하겠다. 그 아래 코드 업로드는 직접 프로젝트 파일을 jar파일이나 zip파일로 build된것을 직접
서버에 업로드하여 배포하겠다는 의미이다. 우리는 샘플 어플리케이션을 선택하겠다. 

<br>

> 다음은, 추가 옵션 구성을 눌러준다. 환경 생성을 눌러주면 안된다.

<br>

<p align="center">
<img src="https://user-images.githubusercontent.com/59492312/151652656-6388ba27-7a14-4ed5-8c2e-85aab42353c9.png">
</p>

추가 옵션 구성을 클릭하면 맨위에 이런 화면이 뜬다.
단일 인스턴스를 사용하게 되면, 로드벨런서를 사용할 수 없게된다. 그러니 사용자 지정 구성을 클릭해주자.
그 외에 고가용성에서 가용성이란 의미는 사용가능 시간을 의미한다. 즉, 간단하게 말하면 서비스가 끊이지 않고 얼마나
연속해서 이어지는 서비스를 제공하냐를 의미한다. 현재 단계에서는 보지않아도 되니, 사용자 지정 구성을 클릭해 준다.

단, 스팟 인스턴스, 온디맨드 인스턴스, 예약 인스턴스는 꼭 알아야 할 기본 개념이니 한번 살펴 보고가도록 하겠다.
(이미 개념을 안다면 그 다음 사진으로 넘어가도 된다.)

**스팟 인스턴스** : 스팟 인스턴스를 이해하려면, 스페어 EC2용량에 대해 먼저 알아야 한다. 남는 EC2에 대해, 수요와 공급에 의해 스페어 EC2의 가격이 정해지는데,
내가 인스턴스 입찰 가격의 범위를 지정해놓고, 해당하는 범위 내에 스페어 인스턴스 가격이 내려오면 입찰이 되어 이용가능해져 인스턴스가 시작된다. 다시 가격이 올라가면
이용을 할 수 없게 되어 인스턴스가 종료되는 방식이 바로 스팟 인스턴스이다. 그러다 보니 언제 종료될지 알 수 없게되는 단점이있다. 그 외에도 온디맨드 방식에 비해서는 90%까지 저렴한 가격을 제공하기에 장단점이
각각 존재한다. 주로 Batch작업에 많이 사용하며, 이 스팟 인스턴스에도 종류가 4가지나 있으니, 더 자세히 알고싶다면 아래 참조링크를 참고하도록 하자.

<br>

> batch 작업이란, 배치작업은 데이터를 실시간으로 처리하는게 아니라, 일괄적으로 모아서 한번에
> 처리하는 작업을 의미한다. 가령, 은행의 정산작업의 경우 배치 작업을 통해 일괄처리를 수행하게 되며 
> 사용자에게 빠른 응답이 필요하지 않은 서비스에 적용한다.

<br>

> [스팟 인스턴스의 개념과 4가지 종류](https://blog.leedoing.com/178)    
> [스팟 인스턴스의 개념](https://wooono.tistory.com/86)    
> [Batch란 무엇인가 ?](https://gold-jae.tistory.com/46)    

<br>

**온디맨드 인스턴스** : 이용한 만큼만 지불하는 방식으로, 1시간단위이며 단1분만 써도 1시간 단위로 계산된다.
선결제가 불가능하고, 다른 인스턴스들에 비해 비싸지만 제일 대중적으로 사용되는 인스턴스이다. 사용하지 않을때는 비용이
지불되지 않는다.

<br>

> Lamda 혹은 aws instance scheduler를 통하여 서버의 시작 및 종료지정하여 온디맨드 방식의 인스턴스의 사용량을 줄여 비용을 절감하기도 한다.

<br>

> [온디맨드 인스턴스란 (1)](https://ssue95.tistory.com/4)    
> [온디맨드 인스턴스란 (2)](https://codingmoondoll.tistory.com/entry/EC2-%EB%B9%84%EC%9A%A9-%EC%9D%B4%ED%95%B4%ED%95%98%EA%B8%B0)

<br>

**예약 인스턴스** : 일반적인 AWS EC2의 요금은 한 달 기준으로 인스턴스를 사용한 시간에 따라 요금이 책정되는 방식인데(온디맨드 인스턴스), 
예약 인스턴스를 이용하면 최소 1년 이상 인스턴스를 사용해야 하는 대신 시간당 요금을 할인받을 수 있다.
한 번에 많은 돈을 지불해야 하지만 전체적으로 봤을 때 훨씬 싸게 이용 가능한 헬스장 회원권과 비슷한 개념이다.
별다른 특이사항 없이 장기간 EC2 인스턴스를 사용해야 한다면 예약 인스턴스를 이용하는 게 비용적인 측면에서 훨씬 이득이다.

<br>

> 이렇게 보면, 당연히 온디맨드 방식이 아닌 예약 인스턴스를 사용해야할것같지만, 최소 계약 단위가 1년이고 Scale up을 할 수 없기에
> 사용하지 않게되더라도 환불할 수 없으니 잘 생각해보고 이용해야 한다.

<br>

> [예약 인스턴스의 개념 (1)](https://m.blog.naver.com/PostView.naver?isHttpsRedirect=true&blogId=rnjsrldnd123&logNo=221492606516)    
> [예약 인스턴스의 개념 (2)](https://ssue95.tistory.com/4)    

<br>

<p align="center">
<img src="https://user-images.githubusercontent.com/59492312/151652654-930b5456-1cad-4618-8a91-d8b1299c3cbe.png">
</p>

<p align="center">
<img src="https://user-images.githubusercontent.com/59492312/151652858-5c340932-670f-4101-bb06-0fbaf32c2db0.png">
</p>

인스턴스의 편집을 클릭하면 위와같이 인스턴스 보안 그룹을 선택하는 란이 보인다.
필자는 미리 만들어둔 보안그룹이 있기에, 해당 보안 그룹 test-service-ec2을 선택했다.

<br>

> 인스턴스의 보안그룹 생성에 관한것은 필자가 적어둔 [파일업로드 적용하기 A부터 Z까지 (1)](https://github.com/sooolog/dev-spring-springboot-knowledge/blob/master/dev-spring-springboot-knowledge/SpringBoot%20file%20upload%20with%20S3/%EC%8A%A4%ED%94%84%EB%A7%81%EB%B6%80%ED%8A%B8%20%ED%8C%8C%EC%9D%BC%EC%97%85%EB%A1%9C%EB%93%9C%20%EC%A0%81%EC%9A%A9%ED%95%98%EA%B8%B0%20A%EB%B6%80%ED%84%B0%20Z%EA%B9%8C%EC%A7%80%20(1)%20.md)글에
> 적어 놓았다. 또한, 보안 그룹만 따로 생성할 수도 있으니 참고하도록 하자.

<br>

여기서, 추가로 짚고 넘어가야 할 부분이 있다. 뒤에 나올 nginx에서 자세하게 다룰 내용이지만 미리 알고가면 더
이해가 잘 되기에 nginx와 로드벨런서, ec2 그리고 보안그룹의 간단한 구조에 대해 알아보고 왜 포트80을 열어주어야 하는지에 대해서도 알고 넘어가도록 하겠다. 
아래 사진들을 보자.

<br>

<p align="center">
<img src="https://user-images.githubusercontent.com/59492312/152101959-50f08e62-e2dd-4e77-987f-a70c0b25092f.png">
</p>

Internet Gateway에서 로드벨런서가 먼저 요청을 받고 이를 Nginx에 80포트로 요청을
전달 하고, 그 뒤에 어플리케이션으로 요청을 전달하게 된다. 또한, Nginx는 EC2내부에 있다.

<br>

> 여기에 포트 5000이라 적혀져있는것은 뒤에 설명할것이니 지금은 신경쓰지 않도록 하자.

<br>

<p align="center">
<img src="https://user-images.githubusercontent.com/59492312/152101954-625e2162-0f72-4e82-8f57-cd1a8c1c4f3d.png">
</p>
<p align="center">
<img src="https://user-images.githubusercontent.com/59492312/152101953-d1a59d88-e1da-4604-b0cd-28acdc3da974.png">
</p>

두 개의 그림 모두, 처음 말했던 구조를 갖고 있다. 이 그림을 보여준 이유는 Nginx는 EC2 내부에 있고, EC2 내부에 있다면,
로드벨런서로부터 80포트로 요청을 받을때 ec2 보안그룹을 통과하여야 한다. 즉, EC2의 보안그룹에도(빈스톡의 EC2 보안그룹) 80포트에 대해 
허용을 해주어야지 Nginx가 요청을 받을 수 있는것이다.

<br>

> 나머지 5000포트와 8080포트 그 외에 관한 내용은 뒤에서 자세히 다룰 것이다.

<br>

<p align="center">
<img src="https://user-images.githubusercontent.com/59492312/152107605-84ecfb27-031e-44dc-a171-44c5c95f4e9f.png">
</p>

그렇다면, 빈스톡 환경생성을 할 때 보안그룹을 어떻게 구성해야 하는지 보도록 하겠다. 위 사진은 위에서 선택한 보안 그룹 test-service-ec2이다. 
ssh에 대해서는 보안상 내ip만 접속하도록 해놓았고, 443과 8080포트에 대해서는 전부 열어놓았다. 그렇다면, 의문이 든다. 80번 포트에 대해서도 보안그룹 인바운드 규칙을 추가해주어야 웹 브라우저에서
접근할 수 있을텐데 이게 어떻게 가능할까 ? 아래 그림을 보도록 하겠다.

<br>

<p align="center">
<img src="https://user-images.githubusercontent.com/59492312/152101949-d3d9fcdb-9372-4ec3-a89a-25f9e6b17715.png">
</p>

빈스톡 환경을 생성해주면 위와같은 디폴트 보안그룹이 2개가 자동으로 생성이 된다. 각각 어떠한
인바운드 규칙이 있는지 보도록 하겠다.

<br>

> 디폴트 보안그룹 2개는 빈스톡 환경 생성을 할때, 보안그룹을 선택(기존에 내가 만들어 놓은 보안그룹) 하던 선택하지 않던
> 디폴트로 생기는 보안그룹이다.

<br>

<p align="center">
<img src="https://user-images.githubusercontent.com/59492312/152101950-e91a36aa-49ec-4012-9ead-8e999e1728d2.png">
</p>

첫번째 디폴트 보안그룹은 이렇게 80포트로 IPv4에 대해서 전부 열려있다.

<br>

<p align="center">
<img src="https://user-images.githubusercontent.com/59492312/152101957-c453f35c-f580-4306-add8-2c41ff197338.png">
</p>

다른 두번째 디폴트 보안그룹은, ssh에 대하여 포트가 전부 열려있고, 방금 전 보았던 첫번째 보안그룹의 보안그룹ID에 대해 80포트로 받아들이고 있다.
필자의 [파일업로드 적용하기 A부터 Z까지 (1)](https://github.com/sooolog/dev-spring-springboot-knowledge/blob/master/dev-spring-springboot-knowledge/SpringBoot%20file%20upload%20with%20S3/%EC%8A%A4%ED%94%84%EB%A7%81%EB%B6%80%ED%8A%B8%20%ED%8C%8C%EC%9D%BC%EC%97%85%EB%A1%9C%EB%93%9C%20%EC%A0%81%EC%9A%A9%ED%95%98%EA%B8%B0%20A%EB%B6%80%ED%84%B0%20Z%EA%B9%8C%EC%A7%80%20(1)%20.md)
의 글을 참고해보면, 보안그룹을 추가할때 다른 보안그룹ID로 추가하는 경우, 추가한 보안그룹ID에 해당하는 보안그룹이 적용된 서버로 들어온 요청이 다시 보안그룹ID를 인바운드에 추가한 서버로의 요청에 대해
허용 한다는 말이다. 아래 사진을 보면서 다시 정리하겠다.

<br>

<p align="center">
<img src="https://user-images.githubusercontent.com/59492312/152114275-145df6f5-f986-45a9-909d-c07901253b70.png">
</p>

이 사진은 빈스톡 환경 구성을 끝내고 환경을 생성한 후 다시 인스턴스 카테고리를 편집하여 인스턴스 보안 그룹 설정을 보여주는것이다.
필자가 위에서 생성한 보안그룹만 체크하고 넘어갔는데, 자동으로 디폴트 보안그룹중 하나가 체크된 모습이다. 이 체크된 디폴트 보안그룹은 두번째 디폴트 보안그룹이다.
그렇기에, 기존 필자가 만든 보안그룹에서 80번 포트 HTTP 설정을 해주지 않아도, 정상적으로 작동이 가능하게 되는것이다.

<br>

> 실제로, 두번째 디폴트 보안그룹의 80번포트를 허용하는 첫번째 보안그룹ID를 지우니 정상적으로 배포되지않고, 배포 후에
> 보안그룹ID를 두번째 디폴트 보안그룹 목록에서 삭제하면, 브라우저에서 정상적으로 열리지 않았었다.
> 추가로, 굳이 디폴트로 등록된 보안그룹ID로 80번 포트에 대해서 요청을 허용하도록 적지않고, HTTP port 80, 0.0.0.0/0로만 보안그룹 목록을
> 추가하면, 이 또한 정상적으로 작동한다.

<br>

<p align="center">
<img src="https://user-images.githubusercontent.com/59492312/152131390-4dde48c0-2709-487b-9fb7-5f3aaa5751ca.png">
</p>

그 다음, 수정해 주어야 할 부분이있는데 우리가 ec2 단일 인스턴스를 만들 때 ssh 접근해 대해서도 보안그룹으로 설정을 하여 사용한다.
ssh접근에 대해서는 aws에서 pem키를 발급하여 pem키가 있어야만 인스턴스에 ssh접속을 할 수 있는데, 그 외에도 보안그룹에서 22번 포트에 대해
접근 가능 ip를 제한해 둔다. 기존 단일 EC2인스턴스를 생성할 때도 디폴트 보안그룹에 ssh가 전체열림(0.0.0.0/0)으로 되어있기에, 이를 내IP만
접속할 수 있도록 바꿔놓는다. 빈스톡에서도 디폴트 보안그룹에서는 전체열림으로 되어있으니 우리는 이를 보안강화를 위해 내ip로만 접속할 수 있도록 
바꾸어 줄 필요가 있다. 필자가 위의 작성하여 보안그룹에는 이미 ssh에 대하여 내 ip로만 접속이 가능하다고 적어놓았으니, 두번째 디폴트 보안그룹에 
적혀있는 인바운드 규칙의 ssh에 대해서는 보안상 삭제하는게 좋다.

<br>

> 위 사진은 두번째 디폴트 보안그룹으로 ssh 목록을 삭제한 뒤의 자료이다.

<br>

<p align="center">
<img src="https://user-images.githubusercontent.com/59492312/152134332-76a73032-38d3-4403-bb6f-7da8ec8c733c.png">
</p>

마지막으로, 첫번째 디폴트 보안그룹에 80번 포트로의 접근에서 ::/0도 설정해주어, 혹시 모를 IPv6에 대한
접근도 가능하게 해주자.

<br>

> (EC2에 Nginx 설치)[https://dev-jwblog.tistory.com/42]      
> (EC2 내부에 Nginx (1))[https://jojoldu.tistory.com/549]      
> (EC2 내부에 Nginx (2))[https://browndwarf.tistory.com/66]       
> (EC2 내부에 Nginx (3))[https://docs.aws.amazon.com/ko_kr/elasticbeanstalk/latest/dg/concepts-webserver.html]     
> (EC2 내부에 Nginx (4))[https://tipsfordev.com/502-bad-gateway-elastic-beanstalk-spring-boot]     

<br>

<p align="center">
<img src="https://user-images.githubusercontent.com/59492312/151653188-2dca1ecb-e073-4116-a3a1-7267c6980143.png">
</p>

<p align="center">
<img src="https://user-images.githubusercontent.com/59492312/151652651-f4faaf49-e88d-4579-8618-4adddc127e70.png">
</p>

용량에서 편집을 눌러준다.   
그러면, Auto Scaling 그룹이 보이는데, 필자는 최대 인스턴스 수를 1로 적어주었다.
이것의 의미는, 아무리 트래픽이 많이 생기고 CPU 가동률이 올라가도 최대 1개의 인스턴스 외에는
Auto Scaling을 적용하지 않겠다는 의미이다.

<br>

> 당연히, 실제 배포되는 서비스나 규모가 어느정도 생긴다면 인스턴스의 최소 최대를 유연하게 설정해주어야 하는게
> 맞으나 우리는 프리티어로 무료로 사용해야하니 최대 1개로 진행하는 것이다.

<br>

<p align="center">
<img src="https://user-images.githubusercontent.com/59492312/151652648-f148a032-45c6-4d2c-a967-cbebdc2074c2.png">
</p>

그 아래로 스크롤을 내리면, 인스턴스 유형이 보이는데 기본적으로 t2.micro와 t2.small 두가지가
설정이 되어있다. 상황에 따라 가용성을 높이기 위한 설정이지만, 우리는 현재 프리티어로 무료로 사용하려하니 위의 사진처럼
t2.small은 지워주고 t2.micro로만 진행하도록 하겠다.

<br>

> 플릿을 비롯한 빈스톡 환경구성에서 그냥 지나쳤던 부분들에 대해서는 다음 글에서 자세하게 설명할 예정이다.
> 여기서는 기본적인 CI/CD환경 구축에만 집중하도록 한다.

<br>

<p align="center">
<img src="https://user-images.githubusercontent.com/59492312/151653123-e9ef2992-9a68-4438-bcf9-37be3d1ee574.png">
</p>
<p align="center">
<img src="https://user-images.githubusercontent.com/59492312/151652644-1038fd6b-2571-4091-b63c-d58ae2bba0e3.png">
</p>

다음은 롤링 업데이트와 배포를 보겠다. 똑같이 편집을 눌러주자. 그러면 맨위에 배포 방식이 나와있다.
여기서 배포 방식을 눌러보면, 한 번에 모두부터 트래픽 분할까지 여러 종류의 배포 방식이 나와있다. 각각의
기본 개념들을 훑고 어떠한 선택을 해야하는지를 설명하도록 하겠다.

**한 번에 모두(All at once)** - 모든 인스턴스에 동시에 새 버전을 배포한다. 배포시간은 가장 짧지만 모든 인스턴스가 업데이트 되기 때문에 다운타임이 발생한다.

**롤링(Rolling)** - 기존 인스턴스 중 일부를 배치 단위로 선정하여 새 버전을 배포한다. 업데이트 중인 배치 인스턴스에 대해서는 서비스가 작동하지 않을 수 있다. 즉 서비스 다운타임을 방지하기 위해 일부 인스턴스만 먼저 업데이트하고 그 후 나머지 인스턴스를 업데이트 하는 방식이다.

**추가 배치를 사용한 롤링(Rolling with additional batch)** - Rolling Update 방식을 그대로 따르나, Rolling Update 방식처럼 기존의 인스턴스외에 추가 인스턴스를 구성하여 배포하는 방식이다. 추가 인스턴스를 구성하기 때문에 Rolling Update에 비해 배포시간이 더 많이 걸리지만 성공적으로 배포만 된다면, 서비스 중단없이 배포할 수 있다.

**변경 불가능(Immutable)** - 기존 인스턴스를 업데이트하는 대신 새 애플리케이션 버전이 항상 새 인스턴스에 배포되도록 하는 더 느린 배포 방법이다. 또한 배포가 실패할 경우 빠르고 안전하게 롤백할 수 있다는 추가 이점이 있다. 이 방법을 사용하면 Elastic Beanstalk에서 변경 불가능한 업데이트를 수행하여 애플리케이션을 배포한다. 변경이 불가능한 업데이트에서 두 번째 Auto Scaling 그룹이 사용자 환경에서 시작되고 새 인스턴스가 상태 확인을 통과할 때까지 새 버전과 기존 버전이 함께 트래픽을 처리한다.

**트래픽 분할(Traffic splitting)** - canary 테스트 배포 방법이다. 이전 애플리케이션 버전을 통해 나머지 트래픽을 계속 처리하면서 수신 트래픽의 일부를 사용하여 새 애플리케이션 버전의 상태를 테스트하려는 경우에 적합하다.

즉, 정리하자면 한 번에 모두에서 변경 불가능까지의 옵션은 사용자의 가용성을 중요시하는 순서로도 볼 수 있다.
그렇기에 보통 개발할 때는 한 번에 모두를 선택하고 실제 서비스 배포 환경에서는 변경 불가능을 선택한다. 하지만, 우리는
현재 빈스톡을 사용한 CI/CD 환경구축을 하려고 하니 '추가 배치를 사용한 롤링'을 선택해주어서 무중단 배포 환경을 만들것이다.
만약 변경 불가능에 대한 배포방식에 대해 자세히 알고싶다면, 필자가 다음 글에 정리해놓았으니 이를 참고하도록 하자.

<br>

> [배포 정책 종류에 관해 (1)](https://docs.aws.amazon.com/ko_kr/elasticbeanstalk/latest/dg/using-features.deploy-existing-version.html)     
> [배포 정책 종류에 관해 (2)](https://velog.io/@hwal92/Elastic-Beanstalk-%EC%97%90-%EB%8C%80%ED%95%9C-%EC%A0%95%EB%A6%AC)     

<br>

<p align="center">
<img src="https://user-images.githubusercontent.com/59492312/151653127-cad86afc-717e-48e7-9f6d-62658823591c.png">
</p>
<p align="center">
<img src="https://user-images.githubusercontent.com/59492312/151652857-16a1dbe5-99a7-4077-8646-5476c536d4f0.png">
</p>

그 다음, 보안을 편집해보도록 하겠다. 여기서는 키 페어 즉, pem키를 설정하는 곳이다.
pem키는 기존에 EC2인스턴스를 생성해본적이 없다면 독자들의 aws에는 아직 pem키가 발급이 안되있을 수 있다.
참조링크 [스프링부트 파일업로드](https://github.com/sooolog/dev-spring-springboot-knowledge/blob/master/dev-spring-springboot-knowledge/SpringBoot%20file%20upload%20with%20S3/%EC%8A%A4%ED%94%84%EB%A7%81%EB%B6%80%ED%8A%B8%20%ED%8C%8C%EC%9D%BC%EC%97%85%EB%A1%9C%EB%93%9C%20%EC%A0%81%EC%9A%A9%ED%95%98%EA%B8%B0%20A%EB%B6%80%ED%84%B0%20Z%EA%B9%8C%EC%A7%80%20(1)%20.md)
를 들어가면 pem키 발급에 대해 정리해 놓았다.

여기까지 하면, 이제 환경 생성을 눌러주어 추가 옵션 구성을 적용한 환경을 생성해 보도록 하겠다.

<br>

<p align="center">
<img src="https://user-images.githubusercontent.com/59492312/152470883-ec2f15c3-a2b7-4cb7-bf48-4c53c61effd8.png">
</p>

환경 생성이 완료가 되면 위와같은 화면을 볼 수 있다. 정상적으로 환경이 생성된 것이다.

> 보면, 필자가 처음 입력한 어플리케이션 이름과 환경이름이 다른걸 알 수 있다. 필자는 처음 real-test라고 만들었던 환경을 삭제하고
> 어플리케이션 이름을 dev-real-test그리고 환경이름을 Devrealtest-env로 다시 만들어 주었기 때문에 그렇다.

#### 🪁 Reference
* 참조링크 : [빈스톡 환경 구성하기 (1)](https://jojoldu.tistory.com/549)

<br>



### 2.빈스톡 환경을 생성해주었다면, 이제는 IAM 설정을 해주도록 하자.

<p align="center">
<img src="https://user-images.githubusercontent.com/59492312/152279576-84633a5d-22b8-4dc6-9ca6-adf37c1550a8.png">
</p>

IAM이 무엇이기에 받아야 하는걸까 ? IAM은 aws 밖에서 aws 클라우드에 접근가능한 권한종류를 지정해주고
이를 인증키 (accessKey, secretKey)를 발급해주어 인증키를 아는 사용자만 사용할 수 있게 해주는것이다. 
우리는 Github Action으로 빌드하고 이를 빈스톡으로 배포해주어야 하기에, 우리가 배포하려는 깃헙 레포지토리에
IAM에서 발급해준 인증키를 등록을 하여 정상적으로 배포가 가능하게 해주려 한다.

<br>

> 그 외에도, IAM사용자라 해서 AWS 콘솔에 접근하되 권한을 제한해놓는 등 IAM의 기능이 많지만, 
> 우선 배포환경에 대한 위주로 보고 필요시 나중에 더 자세히 알도록 하겠다.

<br> 

<p align="center">
<img src="https://user-images.githubusercontent.com/59492312/152279340-7791d194-f020-446e-b29f-ae2b5004a28f.png">
</p>

AWS 콘솔창에서 IAM을 검색해주고 들어가서 사용자 추가버튼을 눌러준다.

<br>

<p align="center">
<img src="https://user-images.githubusercontent.com/59492312/152279486-04d3f6f6-3c65-4735-bef4-dbda864600ef.png">
</p>

사용자 이름을 지정해주어야 하는데, 필자는 프로젝트이름-Githubaction-beanstalk으로 지었지만,
이 IAM이 깃헙액션과 빈스톡을 이용한 CI/CD환경에 사용되는 IAM이라는것만 잘 식별되게 지으면 어떻게 지으든
크게 상관은 없다.

추가로, 아래에 보면 엑세스 유형선택이 보인다.    
1.액세스 키 - 프로그래밍 방식 엑세스
2.암호 - AWS 관리 콘솔 엑세스     

여기서 1번은 설명에 적혀있듯이, API, CLI, SDK등 기타 개발도구에 대해 인증키를 발급해주어
할당된 권한을 이용할 수 있게 해주는것이다. 두번째인 암호 유형은 IAM사용자를 위한것으로 위에서 잠깐 말했듯이
IAM을 이용하여 AWS 콘솔에 접근가능한 계정을 여러개 만든 IAM 사용자를 위한것으로 지금 당장 우리가 필요한것은
아니니, 엑세스 키만 체크하고 넘어가도록 하자.

<br>

> 우리는 깃헙의 레포지토리에 엑세스키를 등록하여 빈스톡에 접근하기 위함이니 프로그래밍 방식 엑세스를 선택해주어야 한다.

<br>

<p align="center">
<img src="https://user-images.githubusercontent.com/59492312/152279355-5648f305-250f-4b2c-a292-8217123d0e47.png">
</p>

그룹에 사용자 추가는 방금말한 IAM사용자를 위한 것으로 우리는 기존 정책 직접연결을 선택해 주자.

<br>

<p align="center">
<img src="https://user-images.githubusercontent.com/59492312/152279354-685de683-e735-4505-a6a5-39838e6c145d.png">
</p>

그 후, beanstalk을 입력해주면 AdministratorAccess-AWSElasticBeanstalk를 선택해주면 된다.
그러면 빈스톡에 대한 관리자 권한을 설정하는것이다. 원래는 이 권한도 세부적으로 나누어 놓아야 하지만, 우리는 관리자
권한 하나만으로 설정하도록 하겠다.

<br>

> 작성한 시간 기준의 모습이며, 시간이 지나면 정책 이름의 명칭이 바뀔 수 있다. 하지만 우리는 Beanstalk의 전 권한을 이용하려고 하니 정책이름이 바뀌더라도
> 전 권한에 해당하는 Beanstalk 정책명을 선택해주면 된다.

<br>

<p align="center">
<img src="https://user-images.githubusercontent.com/59492312/152279350-ba6f1fea-e7c0-4dc8-bae0-6c9b81359c10.png">
</p>

다음은, 태그 추가이다. 아무것도 적지않고 넘어가도 된다. 사용자 정보와 직책을 적어준것이다.

<br>

<p align="center">
<img src="https://user-images.githubusercontent.com/59492312/152280786-31a7c755-383b-4793-a8b8-5e9038ba77c5.png"">
</p>

이 후 다음으로 넘어가면 사용자 만들기가 있다. 사용자 만들기까지 클릭해주면, 위와같은 화면이 나온다.
엑세스 키 ID와 비밀 엑세스 키는 이 화면을 지나면 다시 볼 수 없으니 꼭 복사해서 따로 적어두거나 아니면,
.csv파일을 다운받아 보관해 놓도록 하자.

> 엑세스 키 ID와 비밀 엑세스 키는 노출되서는 안된다. 필자는 학습용으로 발급시킨것이다.
> 실제 사용을 할 때는 노출시켜서는 안된다. 노출되는 순간 누구라도 내가 만든 빈스톡에 배포를 시도할 수 있다.

#### 🪁 Reference
* 참조링크 : [IAM 생성](https://jojoldu.tistory.com/549)

<br>



### 3.이제 내가 배포용으로 사용할 Github 레포지토리로 이동해서 IAM을 설정하겠다.

<p align="center">
<img src="https://user-images.githubusercontent.com/59492312/152284188-a5c127f9-94f7-4856-b30c-9b52170ed5f8.png">
</p>

먼저 내가 빈스톡으로 배포하려는 깃헙 레포지토리에 들어간다. 그 이후 위의 화면처럼 settings를 눌러주고
왼쪽의 Secrets의 Actions를 눌러준다. 그 뒤에 New repository secret을 클릭해주자.

<br>

<p align="center">
<img src="https://user-images.githubusercontent.com/59492312/152284195-0e3155d8-6cc4-4144-a0e9-bbffea55b410.png">
</p>

그러면 이렇게 Name과 Value를 입력하라는 화면이 나오는데, 우리가 방금 전 발급받은 IAM의 엑세스키를 입력해주면 된다.
여기서는 엑세스 키 ID만을 입력해주자. 그리고 Add secret를 클릭해주자.

<br>

> Name도  AWS_ACCESS_KEY_ID라고 적어주어야 한다. 이는 뒤에 프로젝트 코드에서 그 이유를 알아보도록 할 것이다.

<br>

<p align="center">
<img src="https://user-images.githubusercontent.com/59492312/152284196-fe3f4c5d-6361-418a-86fe-765204b2a388.png">
</p>

그 다음 비밀 엑세스 키도 등록을 해주어야 하니, New repository secret을 한번 더 클릭해주어서 등록해주자.
위의 화면처럼 NAME은 AWS_SECRET_ACCESS_KEY로 해주고, Value에는 IAM으로 발급받은 비밀 엑세스 키 값을 넣어주자.

<br>

<p align="center">
<img src="https://user-images.githubusercontent.com/59492312/152284197-47f81d65-2214-4a6d-9830-d61f66249d16.png">
</p>

이렇게 두개 다 정상적으로 등록이 되어야 한다. 이 Repository secrets를 설정해 주는것은 아까 말했다시피,
깃헙 레포지토리에서 빈스톡으로 접근가능하게 하기위한 키를 등록하는것이다.

#### 🪁 Reference
* 참조링크 : [발급받은 IAM 깃헙에 설정](https://jojoldu.tistory.com/549)

<br>



태그 : #온디맨드 인스턴스, #예약 인스턴스, #스팟 인스턴스, #Nginx 80포트, #ssh 보안그룹, #80포트 IPv6, #IAM, #깃헙 IAM설정, #배포옵션, #immutable
