---
layout: post
title:  "Beanstalk환경과 nginx기반에서 멀티도메인, 서브도메인, HTTPS 리다이렉팅"
categories: [ 도메인연결 ]
image: https://user-images.githubusercontent.com/59492312/166934711-797918c8-e3e8-4bc9-a44d-e717cd2a073c.png
---

* 하위도메인(www.도메인)연결과 리다이렉팅
* 도메인.net, 도메인.co.kr을 도메인.com으로 리다이렉팅
* 하위도메인(m.도메인)설계와 리다이렉팅
* http:// -> https:// 리다이렉팅

> 모든 코드는 [깃헙](https://github.com/sooolog/dev-spring-springboot)에 작성되어 있습니다.

* * *

<br>



### 1.하위도메인(www.도메인)연결과 리다이렉팅

<p align="center">
<img src="https://user-images.githubusercontent.com/59492312/153786539-3a375a17-591d-419b-836f-3ef32edcdcd6.png">
</p>
