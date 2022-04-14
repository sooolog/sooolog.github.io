---
layout: post
title:  "Nginx gzip 설정을 통한 텍스트 데이터(HTML, CSS, Javascript, JSON..) 속도 테스트"
categories: [ 속도 테스트 ]
image: "https://user-images.githubusercontent.com/59492312/163343790-d47a36a9-2ad4-4bc4-ac2b-b8ed4dda4eef.png"
---

* gzip에 대한 개념과 텍스트 데이터
* gzip compression level(gzip_comp_level)의 개념
* Nginx의 gzip 설정으로 텍스트 데이터에 대한 속도 테스트

> 모든 코드는 [깃헙](https://github.com/sooolog/dev-spring-springboot)에 작성되어 있습니다.

* * *

<br>



### 1.gzip에 대한 개념과 텍스트 데이터

<p align="center">
<img src="https://user-images.githubusercontent.com/59492312/163304532-943b3931-988b-4ad6-8b7c-78a767b08994.png">
</p>

gzip 이란 ?

gzip은 리눅스/유닉스 시스템에서 널리쓰이는 압축 소프트웨어이다. 웹서버 통신을 할 때 데이터를 gzip 압축하여 전송하면 속도가 더 빨라진다.
조금 더 쉽게 이해하기 위해 예시를 보겠다.

* gzip을 사용한 경우 : [서버에서 HTML 데이터 전송] -> [클라이언트 브라우저가 표시]       
* gzip을 사용하지 않은 경우 : [서버에서 HTML 데이터를 압축 후 전송] -> [클라이언트 브라우저가 압축을 풀고 표시]    

이렇게 데이터를 압축하면 데이터의 크기가 줄어들고 전송속도가 빨라져 서비스의 속도개선이 이루어진다. 또한, gzip을
사용하여 압축을 진행하고 압축을 푸는 과정을 진행 하더라도 요즘 서버나 PC의 경우 충분히 고사양이기때문에 CPU(웹서버와 클라이언트의 브라우저)의 
사용량은 아주 약간 늘어나기에 무시가 가능한 수준이다.(0.1%미만)

특히, 국가간 트래픽이나 느린 인터넷 환경에서 속도가 더 빨라진 것을 크게 느낄 수 있다.

<br>

> 하지만, 아무 데이터들에 대해서나 압축을 진행한다해서 성능이 좋아지고 속도가 빨라지는것은 아니다. 너무 작은 파일은 그냥 전송하는게 더 빠르고
> 이미 충분히 압축된 파일인 바이너리 데이터(image, video, pdf, zip)들은 압축을 진행하면 데이터 크기가 더 커지거나 혹은 불필요한 압축 & 해제
> 단계가 추가되어 오히려 비효율적인 경우가 많다. 그에 반해, 텍스트 데이터는 압축효율이 좋다. 그러기에 주로 텍스트 데이터가 주로 사용되는 경우에
> gzip을 사용하는것이 속도 향상에 도움이 된다.

<br>

> [gzip에 대한 개념](https://blog.lael.be/post/6553)    

<br>

<p align="center">
<img src="https://user-images.githubusercontent.com/59492312/163304538-6c26d0f7-ec6a-48e2-9727-0fbfefa6fb4b.png">
</p>

그렇다면, 텍스트 데이터는 구체적으로 무슨 데이터를 의미하는걸까 ?

텍스트 데이터는 문자를 기반으로 하는 코드값이 저장된 파일이다. 즉, 문자라는 가공 조건이 하나 추가된 형식의
데이터인 것이다. 우리가 아는 HTML, CSS, Javascript모두 텍스트 데이터이며 JSON 또한 텍스트 데이터이다.

<br>

> 바이너리 데이터도 간단하게 짚고 넘어가자면, 바이너리 데이터는 이진 데이터로 0과 1로 표현되는 데이터이다.
> 예를 들면, jpg, png같은 그림파일이나 mp3와 같은 음악파일 exe같은 실행 파일등이 바이너리 파일에 해당한다.    
> [바이너리 데이터(이진 데이터)란](https://ko.wikipedia.org/wiki/%EC%9D%B4%EC%A7%84_%EB%8D%B0%EC%9D%B4%ED%84%B0)    

<br>

> [텍스트 파일이란 (1)](https://m.blog.naver.com/PostView.naver?isHttpsRedirect=true&blogId=tipsware&logNo=221353023593)     
> [텍스트 파일이란 (2)](https://recordsoflife.tistory.com/595)  

<br>



### 2.gzip compression level(gzip_comp_level)에 대해

<p align="center">
<img src="https://user-images.githubusercontent.com/59492312/163313399-c96e6a6b-0a13-40e8-9345-da4b5c4b9e33.png">
</p>

본격적으로 gzip을 사용하여 텍스트 데이터에 대한 속도 테스트 비교를 해보기 이전에
gzip compression level(gzip_comp_level)에 대해 보고 가보도록 하겠다.

gzip compression level이란, 압축률을 의미한다. 이는 1부터 9까지 설정할 수 있으며
아래에서 nginx.conf파일로 설정하겠지만, gzip_comp_level의 값으로 이를 조정 할 수 있다.

숫자가 높아질수록 압축률이 높아지고 파일 크기가 줄어들지만, 동시에 압축과 해제에서 더 많은
리소스(resource)가 들어갈 수 밖에 없다. 즉, 압축률이 높아질 수록 서버 혹은 브라우저 클라이언트단에서
CPU 사용량을 더 높아진다.

<br>

> 그러면, gzip_comp_level을 몇으로 해야 가장 효율이 좋을까. 어느곳은 레벨을 9로 해야 제일 효율이
> 좋다고 하며 다른 참조링크는 3에서 4사이가 좋다고 한다. 통상 압축률이 올라갈수록 CPU가 더 많이 필요하고
> 만약 서버가 CPU 사용량으로 어려움을 겪고 있는경우 압축률을 높이올리는거에 굉장히 한계점이 크고 억지로 끓어올려도
> 올리기 전과의 차이가 그렇게 크지 않아 통상 4정도의 중간 압축률을 유지하는게 좋다고 한다. 하지만, 조금 더 정확하게
> 하려면 내 실제 프로젝트의 EC2 성능과 구성 데이터 파일들을 고려하여 실제 테스트를 해보고 압축률을 지정하는것이 정확하다.      
> [gzip_comp_level 추천 9](https://blog.lael.be/post/6553)        
> [gzip_comp_level 추천 3~4](https://minholee93.tistory.com/entry/Nginx-Compressed-Response-with-gzip)      
> [gzip_comp_level 추천 4](https://www.linuxcapable.com/ko/how-to-enable-configure-gzip-compression-on-nginx/)    

<br>

> [gzip compression level에 대해 (1)](https://minholee93.tistory.com/entry/Nginx-Compressed-Response-with-gzip)     
> [gzip compression level에 대해 (2)](https://www.lesstif.com/system-admin/nginx-gzip-59343019.html)     
> [gzip compression level 1부터 9까지](https://www.pingdom.com/blog/can-gzip-compression-really-improve-web-performance/)

<br>



### 3.Nginx의 gzip, gzip_comp_level 설정으로 텍스트 데이터에 대한 성능테스트

``conf
      client_header_timeout 60;
      client_body_timeout   60;
      keepalive_timeout     10;
      gzip                  on;
      gzip_comp_level       4;
``

nginx.conf을 보면, server블록에 위와같은 코드를 작성할 수 있다. gzip의 값을 ON으로
적용해서 gzip을 사용한다 해주고, gzip_comp_level은 4로 지정하여 중간값으로 설정했다.

<br>

> gzip의 값을 off로 하면 gzip을 사용하지 않는다는 의미이다.    
> [gzip on과 off 값](https://twpower.github.io/48-set-up-gzip-in-nginx)

<br>

<p align="center">
<img src="https://user-images.githubusercontent.com/59492312/163304543-c2bfc4ac-253f-4e01-8c0a-cec39e49795e.png">
</p>

gzip을 적용하기 이전의 네트워크탭에서 본 속도이다.
다음 이미지를 봐보겠다.

<br>

<p align="center">
<img src="https://user-images.githubusercontent.com/59492312/163319734-3c64dc87-dd9e-4da7-bb46-d902f017e96f.png">
</p>

노란색으로 된 부분이 서비스 서버에서 보내는 응답이다. 다른 응답들은 서비스 서버
외의 외부로 요청을 보내온거에 대한 응답이다.

바로 위의 이미지와 비교해보면    

index.html: 15ms -> 4ms   
Creative.min.css: 18ms -> 8ms   
jquery.min.js: 17ms -> 7ms   

텍스트 데이터에 대한 전송시간이 상당히 개선된것을 볼 수 있다. 물론, bootstrap.min.css나
font-awesome.min.css의 경우는 다소 전송시간이 늘어났지만, 전반적으로 전송시간이 상당히
단축된것을 알 수 있다.

<br>

> 위는 다른 참조사이트의 실제 속도 개선 결과값을 갖고와서 보여준것이다. 실제로 내 프로젝트에 gzip과 gzip_comp_level을
> 사용하는경우, 텍스트 데이터가 많이 사용되는지 아니면 사진이나 음악 실행파일등이 많이 사용되는 서비스라 하더라도 이는
> S3같은 스토리지에서 따로 관리해주는지 등을 모두 따져보아야 한다. gzip을 사용하면 스토리지를 따로 두어서 사진파일등은 따로 관리해주는것이
> 좋다. 또한, 서버의 CPU성능에 따라서도 압축률을 고려해야하기 때문에 처음은 중간 압축률인 4로 잡고 최종 프로젝트가 완료되면 압축률을
> 조절하여 서비스별로 커스터마이즈하면 된다.

<br>

> 또한, 크롬 개발자도구의 네트워크 탭에서 속도가 얼마나 개선되었는지 보는것도 중요하지만, 그 외에 웹사이트
> 속도를 측정해주는 구글의 Paged insight등의 툴을 사용하여 전반적으로 한번 더 점검해보는게 좋다.

<br>

> [nginx의 gzip을 활용한 전송속도 개선](https://ko.linux-console.net/?p=1877)

<br>



### 🚀 추가로,

<p align="center">
<img src="https://user-images.githubusercontent.com/59492312/163314916-5217e83d-a8c5-4876-a1dd-ae2cb44ae54a.png">
</p>

다른 웹사이트가 gzip을 사용하는지 알 수 있는 방법이 있다.

위의 이미지는 네이버를 들어간 후 크롬 개발자도구의 네트워크 탭을 들어간것이다.
그리고 파일의 종류인 All이나 Doc을 클릭하여 맨 위의 www.naver.com을 클릭하면
이미지와 같은 화면이 뜬다.

여기서 Response-Headers의 content-encoding을 클릭하면 gzip이라고 적혀있는것이
보인다. 네이버 또한, 메인화면 Document(Doc)을 가져올때 gzip이 사용된걸 알 수 있다.

<br>

<p align="center">
<img src="https://user-images.githubusercontent.com/59492312/163317573-2093029b-1ae5-4131-b9fb-3207d1c2cf3c.png">
</p>

두번째로 알아두면 좋은것이,
위의 네트워크 탭에서 All 부터 시작해서 Fetch/XHR 등등 여러 파일 종류들을 선택해서 볼 수 있는란이 있다.
이 중에 Doc은 HTML이나 String형 문자열등을 볼 수 있는 유형이다. 즉, html 유형의 파일도 Type으로는 
Document에 속한다. 

<br>

<p align="center">
<img src="https://user-images.githubusercontent.com/59492312/163337825-fa792f54-1956-4fb1-8469-cd6a042a9d95.png">
</p>

마지막으로,
위의 이미지를 보면 맨 하단에     
4requests, Finish: 592ms, DOMContentLoaded: 414ms, Load: 568ms
라고 적혀져 있는걸 알 수 있다. 

* requests는 브라우저에서 보낸 모든 요청을 뜻한다. 내 서비스와
  관련이 없이 클라이언트가 구글 확장프로그램을 쓰거나 그 외의 것들도 모두 브라우저단에서 요청을 보내는것이니
  이 모든것들이 요청에 포함된다.

* Finish는 모든 요청에 대한 응답이 끝난 시간을 의미한다. 요청이라고 꼭 웹서비스에 접속했을때만이
  아닌 위의 request에서 본것처럼 시간을 두고 주기적으로 요청을 보낼 수 있기때문에 finish의 값은 얼마든지
  커질 수 있다.

* DOMContentLoaded는 DOM Tree를 그리는데 걸리는 시간을 의미한다.

* Load는 DoM Tree를 그리는것과 이미지를 화면에 로드하는것까지 포함하는 시간을 의미한다.

이 중, DOMContentLoaded와 Load는 사용자 경험을 판단하는 기준 중, 가장 기본이 되는 것으로
속도나 성능개선에 참고할 수 있다.

<br>

> DOM Tree 구조란 쉽게 말해서 우리가 아는 <html>~</html> 즉, html요소와
> 그 안의 모든 요소들을 포함한 구조라고 이해하면 된다.

<br>

> [개발자도구 네트워크탭 요소들](https://velog.io/@te-ing/%EA%B0%9C%EB%B0%9C%EC%9E%90%EB%8F%84%EA%B5%AC-Network%ED%83%AD-%EC%B4%9D%EC%A0%95%EB%A6%AC)    

<br>



태그 : #gzip, #텍스트 데이터, #바이너리 데이터, #gzip compression level, #gzip_comp_level, #content-encoding, #Doc, #DOMContentLoaded, #Load, #finish, #requests
