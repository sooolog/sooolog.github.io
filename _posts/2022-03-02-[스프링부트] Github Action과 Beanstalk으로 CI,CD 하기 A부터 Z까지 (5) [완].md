---
layout: post
title:  "[스프링부트] Github Action과 Beanstalk으로 CI,CD 하기 A부터 Z까지 (5) [완]"
categories: [ 빌드와 배포 ]
image: https://user-images.githubusercontent.com/59492312/162915206-638c2517-c122-4344-abf5-e8a984dd3aa8.png
---

* 리버스 프록시로써 작동하는 Nginx
* 빈스톡 환경 배포의 마지막 설정파일 nginx.conf 설정
* 

> 모든 코드는 [깃헙](https://github.com/sooolog/dev-spring-springboot)에 작성되어 있습니다.

* * *

<br>



### 1.리버스 프록시로써 작동하는 Nginx

<p align="center">
<img src="https://user-images.githubusercontent.com/59492312/152483197-fe279aef-46bc-4a51-8b4e-f5d4bd19c091.png">
</p>

Nginx는 무중단 배포로도 사용이 되지만, 여기서는 Nginx가 받은요청을 스프링부트 임베디드 톰캣(WAS)로
보내는 역활만 하는 리버스 프록시로 보도록 하겠다.

<br>
