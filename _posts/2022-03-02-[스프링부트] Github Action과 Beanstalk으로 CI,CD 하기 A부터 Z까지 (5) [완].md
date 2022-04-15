---
layout: post
title:  "[스프링부트] Github Action과 Beanstalk으로 CI,CD 하기 A부터 Z까지 (5) [완]"
categories: [ 빌드와 배포 ]
image: https://user-images.githubusercontent.com/59492312/163088078-f5b45b97-edb2-4b24-88ec-7b70c813ff2a.png
---

* 리버스 프록시로써 작동하는 Nginx
* 빈스톡 환경 배포의 마지막 설정파일 nginx.conf 설정

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

> 여기서는 Nginx로 무중단 배포를 사용하지 않겠다.(참고로, Nginx로 무중단 배포를 사용해도 리버스 프록시로써 역활도 동시에한다.)

<br>

Nginx.conf내에 쓰는 코드들을 설명하기에 앞서, 빈스톡 환경에서 리버스 프록시로써의 역활에 대해
필수적으로 알아야 할 기본개념들을 보고 가도록 하겠다. 꼭 참고하기를 바란다.

<br>

> 빈스톡에서는 로드벨런서(Application Load Balancer)가 무중단 배포를 대신해서 수행한다.

<br>

> 리버스 프록시란, 내부망의 서버 앞단에서 요청을 처리해주는 서버를 의미한다. 조금 더
> 쉽게 설명하자면, EC2 인스턴스 내부에는 리버스 프록시 역활을 하는 Nginx가 있고, 우리가 배포한
> 스프링부트 프로젝트인 WAS(웹 어플리케이션 서버)가 존재한다. EC2 인스턴스로 오는 요청을 WAS가 직접받지않고
> 리버스프록시인 Nginx가 대신받은 후에 다시 이를 WAS로 보내는것이다.

> [nginx와 리버스 프록시란](https://juneyr.dev/nginx-basics)

<br>

<p align="center">
<img src="https://user-images.githubusercontent.com/59492312/152486577-aa3176b8-8786-4923-b70f-3dd1e98f9b91.png">
</p>

보는바와 같이 EC2내부에 Nginx가 먼저 요청을 받게된다. 그렇다면, 궁금증이 생긴다. 왜 바로 WAS에서 받지않고
번거롭게 리버스 프록시를 거쳐서 요청을 받을까 ?    

그 이유는 바로 보안때문이다. WAS는 대부분 DB서버와 연결 되어있기에, WAS가 최전방에 있으면 보안이 취약해진다. 그 때문에
리버스 프록시인 nginx를 앞 단에 두고 사용하는 것이다.

<br>

> [Nginx를 리버스 프록시로 사용하는 이유](https://juneyr.dev/nginx-basics)    
> [Nginx 그림 참조](https://stackoverflow.com/questions/54612962/502-bad-gateway-elastic-beanstalk-spring-boot)

<br>

그런데, 위 사진을 보면 뭔가 이질적인게 느껴진다. 로드벨런서로 부터 클라이언트의 요청을 Nginx가
받는것 까지는 이해가 되는데, 그 요청을 그대로 WAS에 보내는게 아니라 5000포트로 허공에 보내는걸 알 수 있다.
왜 이러는 걸까 ? 

Nginx는 빈스톡에서 사용될 때 별다른 설정을 해주지 않으면 받은 요청을 다시 5000포트로 보내게 설정되어있다.
즉, 정상적으로 작동하게 하려면 우리는 이를 8080포트(WAS로)로 보내주거나, 아니면 스프링부트 어플리케이션을 5000포트로
실행하여야 한다.

<br>

> 5000포트로 스프링부트를 실행하여 진행하는 방법에 대해서도 글 맨 마지막 추가부분에 잠깐 설명하겠지만, 필자는 8080포트로 진행하도록 하겠다.

<br>

스프링부트 어플리케이션은 8080포트에서 실행시키고, Nginx는 받아온 요청을 다시 8080포트인 WAS로 보내는
설정을 바로 .platform/nginx/nginx.conf 에 있는 nginx.conf에서 설정하여 사용한다.

<br>



### 2.빈스톡 환경 배포의 마지막 설정파일 nginx.conf 설정

```conf
user                    nginx;
error_log               /var/log/nginx/error.log warn;
pid                     /var/run/nginx.pid;
worker_processes        auto;
worker_rlimit_nofile    33282;

events {
    use epoll;
    worker_connections  1024;
    multi_accept on;
}

http {
  include       /etc/nginx/mime.types;
  default_type  application/octet-stream;

  log_format  main  '$remote_addr - $remote_user [$time_local] "$request" '
                    '$status $body_bytes_sent "$http_referer" '
                    '"$http_user_agent" "$http_x_forwarded_for"';

  include       conf.d/*.conf;

  map $http_upgrade $connection_upgrade {
      default     "upgrade";
  }

  upstream springboot {
    server 127.0.0.1:8080;
    keepalive 1024;
  }

  server {
      listen        80 default_server;
      listen        [::]:80 default_server;

      location / {
          proxy_pass          http://springboot;
          proxy_http_version  1.1;
          proxy_set_header    Connection          $connection_upgrade;
          proxy_set_header    Upgrade             $http_upgrade;

          proxy_set_header    Host                $host;
          proxy_set_header    X-Real-IP           $remote_addr;
          proxy_set_header    X-Forwarded-For     $proxy_add_x_forwarded_for;
      }

      access_log    /var/log/nginx/access.log main;

      client_header_timeout 60;
      client_body_timeout   60;
      keepalive_timeout     60;
      gzip                  off;
      gzip_comp_level       4;

      # Include the Elastic Beanstalk generated locations
      include conf.d/elasticbeanstalk/healthd.conf;
  }
}
```

nginx.conf에 작성된 코드이다. 이제부터 각 문단이나 중요키워드를 중심으로
설명해 나가도록 할텐데, 그전에 부수적으로 알아야 할 개념들에 대해서도 설명하니, 천천히 따라오면
된다.

1. 먼저 nginx 설정파일의 문법에 대해서 보도록 하겠다.

* **Directives(지시어)** : nginx.conf 처럼 설정파일에서는 디렉티브라는것으로 옵션설정을 한다. 즉,
위의 코드중에 user, error_log, pid, http, server모두 디렉티브이다. 또한, 디렉티브는 블록(혹은 컨텍스트)디렉티브와
심플 디렉티브로 나뉜다.
 
* **심플디렉티브** : 세미콜론(;)으로 끝나는 디렉티브이다. 위의 디렉티브중 user,error_log,pid가 심플디렉티브이다.
 
* **블록 디렉티브** : 세미콜론 대신에 중괄호({})로 끝난다. 또한, 블록 디렉티브는 중괄호 안에 다른 디렉티브를 가질 수 있다.
위의 디렉티브중 http, server가 블록 디렉티브이다..

<br>

> 블록은 컨텍스트라고도 불린다. 그렇기에 컨텍스트안에는 다른 디렉티브들이 있을 수 있는데, user, error_log, pid처럼
> 다른 컨텍스트에 속해있지않은 디렉티브들을 메인 컨텍스트안에 있는것으로 보며, server디렉티브는 http 컨텍스트안에
> 있는것으로 보면 된다.

<br>

> [심플 디렉티브와 블록(컨텍스트) 디렉티브 (1)](https://kscory.com/dev/nginx/install)      
> [심플 디렉티브와 블록(컨텍스트) 디렉티브 (2)](https://architectophile.tistory.com/12)

<br>

2. 다음으로, nginx.conf의 파일이 어디있는지 그리고 디렉터리 구조는 어떠한지 보도록 하겠다.
(모두 필요한 내용이니 가볍게 읽어보면 도움이 된다.)

nginx.conf파일은 기본적으로 /etc/nginx/ 폴더안에 위치하게 된다.

즉, 조금 더 풀어서 말하자면, 우리가 EC2인스턴스에 ssh로 접속하게 되면 제일 먼저 홈디렉토리에서 시작하게 된다.
그곳에서 명령어 cd /etc/nginx로 들어가보면 각종 nginx 설정파일들과 폴더들이 있고 거기에 nginx.conf파일도 있는것이다.
그 중에 nginx.conf파일이 가장 주요한 설정파일이다.

<br>

> 루트 디렉토리란, 최상위 디렉토리로 모든 디렉토리들의 시작점을 의미한다. 루트 디렉토리를 '/'로 표시한다.
> 홈 디렉토리란, '\~' 로 물결모양의 디렉토리명으로 표시된다. 통상 EC2에 ssh로 접속하면 시작하는 위치이다. 이
> 디렉토리는 최상위 디렉토리 '/' 하위의 home디렉토리 안에 사용자 디렉토리를 의미하며, ec2인스턴스는 단일 인스턴스이건
> 빈스톡을 이용한 인스턴스이건 ec2-user로 디렉토리명이 지정되어있다. 실제로 해당 홈 디렉토리를 들어가보면 디렉토리가
> '\~' 로 표시된것을 알 수 있다.    
> [루트 디렉토리 '/'와 홈 디렉토리'~'](https://dana-study-log.tistory.com/entry/Linux-%EB%A6%AC%EB%88%85%EC%8A%A4-%ED%8C%8C%EC%9D%BC-%EC%8B%9C%EC%8A%A4%ED%85%9C-%EA%B5%AC%EC%A1%B0-%EB%A3%A8%ED%8A%B8-%EB%94%94%EB%A0%89%ED%86%A0%EB%A6%AC-%ED%99%88-%EB%94%94%EB%A0%89%ED%86%A0%EB%A6%AC)

<br>

> [nginx.conf 파일의 위치 (1)](https://kscory.com/dev/nginx/install)      
> [nginx.conf 파일의 위치 (2)](https://architectophile.tistory.com/12)

<br>

```conf
user                    nginx;
error_log               /var/log/nginx/error.log warn;
pid                     /var/run/nginx.pid;
worker_processes        auto;
worker_rlimit_nofile    33282;
```

(1).**user** : 프로세스가 실행되는 권한을 설정하는거다. root혹은 nginx와 같은 값을 지정할 수 있다.

(2). **error_log** : nginx에서 일어나는 에러 로그파일이 존재하는 곳이다. warn은 로그 레벨을 의미하며, 해당 로그레벨 이상만 기록한다. error로 변경할 수도 있다.

<br>

> 우리가 빈스톡으로 배포한 ec2인스턴스에 ssh접속하여 vim /var/log/nginx/error.log를 하면 nginx의 에러로그를 볼 수 있다.
> 그러나, 조금 더 편한 방법으로는 빈스톤 콘솔에서 로그를 클릭하고, 전체 로그를 요청해서 다운받은다음에, nginx폴더의 error텍스트를
> 열어보면 더 편하게 볼 수 있다.(같은 로그다.)

<br>

(3).**worker_processes** : nginx에서 몇 개의 워커 프로세스를 생성할 것인지를 지정하는 지시어이다. 1이면 모든 요청을 하나의
프로세스로 실행하겠다는 의미이다. CPU 멀티코어 시스템에서 1이면 하나의 코어만으로 요청을 처리하겠다는 의미이다. 보통 명시적으로 서버에
장착되어 있는 코어 수 만큼 할당하는 것이 보통이며, 그렇기에 auto로 주로 설정한다.

<br>

> EC2인스턴스는 인스턴스 종류마다 코어의 개수가 다르다. 그렇기에 auto로 설정하는 편이 좋다.
> auto는 사용가능한 CPU 코어를 자동탐지하여 적용해준다.

<br>

(4).**worker_rlimit_nofile (number)** : 위의 워커 프로세스에서
    사용하는 오픈 파일의 최대 개수의 한계를 변경하려할 때 사용한다.

(5).**worker_rlimit_core (size)** : 위에서는 나와있지 않지만, 추가로 함께 알아두면 좋다. 이는, 워커 프로세스에서
    사용하는 코어 파일의 가장 큰 사이즈의 한계를 변경하려할 때 사용한다.

<br>

추가로,    
1. nginx 프로세스는 마스터(master)와 워커(worker) 프로세스로 나뉜다. 우리가 설정한 프로세스는
워커 프로세스이며, user에서 지시한 값은 해당 워커 프로세스의 권한 지정이다. 만약 user에서 권한을 루트사용자(최고사용자)로
지정해놓았는데, 악의적인 사용자가 제어권을 갖게되면 최고 사용자의 권한으로 원격제어당하는것이기 때문에 루트사용자로 지정하지않고
nginx로 사용하는것이다.

2. nginx성능 튜닝에 관해서이다. 
통상 worker_processes는 auto로 설정하지만, 운영체제를 위해서 CPU core수의 10\~20%는 남겨두는경우가 있다.
예를 들면, 24core를 갖은 CPU라면 Nginx에서 사용할 코어수를 1\~20개 정도 할당하고 2\~4개는 OS용으로 남겨두는것이다.
지금 당장은 적용할 필요는 없지만, 참고하도록하자.

<br>

> [user nginx의 의미 (1)](https://whatisthenext.tistory.com/123)    
> [user nginx의 의미 (2)](https://narup.tistory.com/209)    
> [nginx 에러 로그](https://prohannah.tistory.com/136)    
> [worker_processes의 개념 (1)](https://whatisthenext.tistory.com/123)    
> [worker_processes의 개념 (2)](https://narup.tistory.com/209)    
> [nginx에서 worker processes의 auto 의미](https://nginx.org/en/docs/ngx_core_module.html#worker_processes)    
> [nginx 성능튜닝에 관한 이야기](https://couplewith.tistory.com/entry/%EA%BF%80%ED%8C%81-%EA%B3%A0%EC%84%B1%EB%8A%A5-Nginx%EB%A5%BC%EC%9C%84%ED%95%9C-%ED%8A%9C%EB%8B%9D-2-%ED%94%84%EB%A1%9C%EC%84%B8%EC%8A%A4-%EC%B2%98%EB%A6%AC%EB%9F%89-%EB%8A%98%EB%A6%AC%EA%B8%B0)

<br>

이후, pid는 필요시 nginx.org 공식홈에서 설명해주고 있으니,
궁금하다면 읽어보도록 하자.

<br>

```conf
events {
    use epoll;
    worker_connections  1024;
    multi_accept on;
}
```

(1).**use epoll** : Linux커널 2.6이상인경우에 쓰이는 효율적인 이벤트 처리 방식이다.

<br>

> FreeBSD 4.1+, OpenBSD 2.9+, NetBSD 2.0 및 MacOS에서 순차적인 처리를 위한 방식으로 kqueue가 사용되고 있으나
> Linux 2.6+ 이상에서 사용하는 효율적인 이벤트 처리 방식으로 epoll가 사용되고 있다. 여기서 말하는 순차적인 처리와 효율적인 이벤트
> 처리 방식을 multi_accept에서 다시 설명하도록 하겠다.

<br>

(2).**worker_connections** : 하나의 프로세스가 처리할 수 있는 동시 접속 최대 수를 의미한다. 
그렇기에, 한번에 받을 수 있는 요청은 worker_processess(프로세스 수) X worker_connections수로
정해진다. 통상 512, 1024를 기준으로 적어진다.

<br>

> 여기서 고려해야 할 사항은, 이 커넥션의 의미는 클라이언트와의 커넥션 뿐만이 아니라, 프록시 서버들간의 연결이나 아니면 다른 연결들에대해서도
> 포함한 총 커넥션의 수라는것이다. 또한, 실제 동시 연결 커넥션 수는 오픈 되는 파일의 최대값을 넘을 수 없다는 거다. 이 값은
> 위에서 설명한 worker_rlimit_nofile을 의미한다. 지금 당장은 고려하지 않아도 되지만, 서비스의 규모가 커지면 고려해야 할 부분이다.    
> [worker_connections의 추가내용](https://nginx.org/en/docs/ngx_core_module.html#worker_connections)

<br>

(3).**multi_accept** : 순차적으로 요청(커넥션)을 받지 않고 동시에 요청을 접수하는 방식이다. 디폴트값은 off이다.

<br>

> 위에 user가 epoll로 설정되어야만 사용할 수 있다. 만약 kqueue로 설정되어 있으면 해당 디렉티브(multi_accept on)는 무시되는데
> 그 이유는 kqueue방식은 애초에 커넥션들을 순차적으로 받아들여기 위해 사용되는 방식이기 때문이다.

<br>

> [use epoll, kqueue (1)](https://couplewith.tistory.com/entry/%EA%BF%80%ED%8C%81-%EA%B3%A0%EC%84%B1%EB%8A%A5-Nginx%EB%A5%BC%EC%9C%84%ED%95%9C-%ED%8A%9C%EB%8B%9D-2-%ED%94%84%EB%A1%9C%EC%84%B8%EC%8A%A4-%EC%B2%98%EB%A6%AC%EB%9F%89-%EB%8A%98%EB%A6%AC%EA%B8%B0)       
> [use epoll, kqueue (2)](https://nomaddream.tistory.com/19)   
> [worker_connections에 관하여 (1)](https://couplewith.tistory.com/entry/%EA%BF%80%ED%8C%81-%EA%B3%A0%EC%84%B1%EB%8A%A5-Nginx%EB%A5%BC%EC%9C%84%ED%95%9C-%ED%8A%9C%EB%8B%9D-2-%ED%94%84%EB%A1%9C%EC%84%B8%EC%8A%A4-%EC%B2%98%EB%A6%AC%EB%9F%89-%EB%8A%98%EB%A6%AC%EA%B8%B0)   
> [worker_connections에 관하여 (2)](https://nomaddream.tistory.com/19)   
> [worker_connections에 관하여 (3)](https://whatisthenext.tistory.com/123)   
> [multi_accept 개념 (1)](https://nginx.org/en/docs/ngx_core_module.html#worker_connections)    
> [multi_accept 개념 (2)](https://couplewith.tistory.com/entry/%EA%BF%80%ED%8C%81-%EA%B3%A0%EC%84%B1%EB%8A%A5-Nginx%EB%A5%BC%EC%9C%84%ED%95%9C-%ED%8A%9C%EB%8B%9D-2-%ED%94%84%EB%A1%9C%EC%84%B8%EC%8A%A4-%EC%B2%98%EB%A6%AC%EB%9F%89-%EB%8A%98%EB%A6%AC%EA%B8%B0)

<br>

```conf
http {
  include       /etc/nginx/mime.types;
  default_type  application/octet-stream;

  log_format  main  '$remote_addr - $remote_user [$time_local] "$request" '
                    '$status $body_bytes_sent "$http_referer" '
                    '"$http_user_agent" "$http_x_forwarded_for"';

  include       conf.d/*.conf;

  map $http_upgrade $connection_upgrade {
      default     "upgrade";
  }
}
```

http 블록은 웹 서버에 대한 동작을 설정하는 영역으로, server, location 블록을 포함한다.
또한, 여기서 선언된 값은 하위블록에 상속된다.

<br>

> http 블록을 여러개 생성하여 관리할 수 있지만, 권장사항은 아니다. http블록을 하나만 생성하여 이용하는것이
> 권장사항이다.     
> [http블록 갯수 권장사항](https://taewooblog.tistory.com/74)

<br>

> [http 블록에 관하여 (1)](https://prohannah.tistory.com/136)
> [http 블록에 관하여 (2)](https://taewooblog.tistory.com/74)
> [선언된 값에 대한 하위 블록의 상속](https://juneyr.dev/nginx-basics)

<br>

그 다음, include 디렉티브는 추가적인 설정파일(conf)을 포함시켜주거나 MIME 타입 목록을
지정하는데 사용된다.

  include       /etc/nginx/mime.types;    
  include       conf.d/*.conf;

이곳에서도, mime.types 파일을 읽어들이거나 *.conf로 설정파일을 불러오고있다.

<br>

> 추가로 위의 include의 경로에 대해 알려주자면, include /etc~처럼 /로 시작하는 경우에는
> 루트 디렉토리를 기반으로 시작한 경로의 파일을 의미하고 그냥 include conf.d/~로 사용하는 경우네는
> 해당 코드(include conf.d/~)가 적혀진 파일의 위치를 기준 즉, 상대 경로로 해당 파일을 include 하겠다는
> 의미이다.

<br>

> [include 지시어에 관하여 (1)](https://narup.tistory.com/209)    
> [include 지시어에 관하여 (2)](https://aimaster.tistory.com/11)    
> [mime.types에 관하여](https://kscory.com/dev/nginx/install)

<br>

default_type의 default_type  application/octet-stream;는 옥텟 스트림 기반의 http를 사용한다는 의미이며,
MIME 타입 설정이다.

<br>

> [default_type에 관하여 (1)](https://narup.tistory.com/209)    
> [default_type에 관하여 (2)](https://kscory.com/dev/nginx/install)

<br>

log_format은 nginx의 access 로그의 형식을 지정해준다.

<br>

> 뒤에 나오는 access_log의 로그 형식을 지정한다. 또한, 앞서 언급한 error로그의 형식에는
> 영향을 주지 않는다. 

<br>

```conf
  upstream springboot {
    server 127.0.0.1:8080;
    keepalive 1024;
  }
```

이제는 upstream 블록 디렉티브를 보겠다.   
@@@@@@@@@@@@@@@@
 
upstream은 origin은 WAS 즉, 웹 어플리케이션 서버를 의미한다. nginx와 연결한 웹 어플리케이션 서버를 지정하는데
사용된다. 하위에 있는 server 지시어는 연결할 웹 어플리케이션 서버의 'IP주소(호스트주소):포트'로 지정해준다.

upstream은 여러개를 만들 수 있으며,a

<br>

> nginx서버를 가리키는것은 downstream에 해당한다.

<br>

> upstream 지시어 안에, server 지시어를 여러개 적어서 로드벨런싱으로써 작용하게 할 수도 있다.    
> upstream test{    
>         server 호스트주소:포트    
>         server 호스트주소:포트    
> }    
> 와 같이 써주며, 순서대로 요청을 돌아가며 처리해주는 라운드로빈 방식과 서버마다 가중치를 주고 가중ㅊ치가 높은곳 부터 부하를 보내는 가중치
> 라운드 로빈 방식이 있는데, 다른 설정이 없다면 라운드 로빈 방식으로 작동한다.       
> [엔진엑스로 로드벨런싱 이용하기](https://cantcoding.tistory.com/77)

<br>

> [upstream 지시어 (1)](https://narup.tistory.com/209)
> 


/var/log/nginx/access.log이거 형식 있었는데 main;
이거 서버마다 로그 이름 다르게 하는거 있었다.

@@@@@@@@@@@@@@@@


<br>

```conf
  server {
      listen        80 default_server;
      listen        [::]:80 default_server;

      access_log    /var/log/nginx/access.log main;

      client_header_timeout 60;
      client_body_timeout   60;
      keepalive_timeout     60;
      gzip                  off;
      gzip_comp_level       4;

      # Include the Elastic Beanstalk generated locations
      include conf.d/elasticbeanstalk/healthd.conf;
  }
```

다음은, server 블록 지시어다.    
server 블록의 역활을 간단히 말하면, 하나의 웹사이트를 선언하는 데 사용되며, 가상 호스팅(Virtual Host)의 개념이다.
아래 listen 지시어와 server_name지시어의 역활을 알아가면서 더 쉽게 이해해보도록 하겠다.

<br>

```conf
  server {
      listen         80 default_server;
      listen         [::]:80 default_server;
  }
```

(1).**listen지시어** : listen지시어 다음에는 포트번호가 온다. 예를 들어, "listen 80처럼 되어있으면
EC2에 들어온 요청중에 포트가 80번에 해당하는것을 받겠다." 라는 의미이다.(Nginx는 EC2 인스턴스 내부에 있다.)

바로 다음에는 default_server 인자가 나오는걸 볼 수 있다.
이 default_server는 프로토콜(http, https, ftp등) 별로 즉, 포트별로 단 하나의 
server 블록에만 존재해야한다. 즉, listen의 값이 80인 server블록이 여러개 있다면
단 하나의 server블록내의 listen 80 지시어에 대해서만 default_server로 지시할 수 있다는것이다.

이 default_server의 기능은, 들어온 특정 포트(여기선 80으로 보자)에 대한 모든 요청이
real-test.com:80, real-test.net:80, www.real-test.com:80와 같은 형태로 요청이 들어온다 할때,
server_name이 매칭되는 도메인이 없는경우 해당 요청의 포트 즉, 여기서는 80번 포트에 대해 모든요청이
listen지시어 값이 80 default_server로 지정된 server블록에서 처리 하게 되는 것이다.    
(바로 아래 server_name 지시어에 대한 설명이 나와있다.)

<br>

> listen HostName 혹은 IP주소 / 포트(port)로 적게 되어있는데, 'HostName 혹은 IP주소'에 대해서는
> 언급하지 않고 가도록 하겠다.

<br>

> 만약, default_server가 따로 지정되어있지 않은경우 기본적으로 가장 먼저 정의된
> 특정 포트에 대한 server블록이 default_server로 지정이 된다.     
> [default_server의 자동 지정](http://i5on9i.blogspot.com/2016/01/nginx-server.html)

<br>

> listen [::]:80 와 같이 사용되면, 이는 IPv6형식의 요청을
> 처리한다는 의미이다.    
> [listen \[::\]:80에 관하여 (1)](https://architectophile.tistory.com/12)     
> [listen \[::\]:80에 관하여 (2)](https://swiftcoding.org/nginx-routing)    

<br>

```conf
  server {
      listen         80 default_server;
  }
```

또한, default_server가 명시된 server 블록의 경우, server_name 지시어를
써주던 써주지 않던 그대로 정상적으로 기능을 한다. 예를 들면, 위처럼 listen 80 default_server;로
지정해놓고, server_name지시어가 없거나 아니면 server_name 지시어 값이 real-test.com인 경우 여부에 상관없이
www.real-test.com:80, m.real-test.com:80, real-test.net:80, real-test.co.kr:80와 같은 요청이
오면 모두 listen 80 default_server; 가 있는 server에서 요청을 모두 받아간다.(이 경우에 server블록이 하나밖에
설정을 안한경우였다.)

<br>

> 위의 도메인들이 Route53으로 DNS 도메인설정을 했을때 얘기이다. 빈스톡 환경에서 
> Route53을 이용한 도메인 연결은 다음 글에서 설명할것이다.

<br>

> [listen의 개념](https://architectophile.tistory.com/12)
> [default_server의 개념 (1)](https://swiftcoding.org/nginx-routing)
> [default_server의 개념 (2)](https://architectophile.tistory.com/12)

<br>

```conf
  server {
      listen         80 default_server;
      listen         [::]:80 default_server;
      server_name    real-test.com   #추가된 지시어
  }
```

(2).**server_name지시어** : server_name지시어는 클라이언트가 특정 포트로 요청을 하되, 어느 도메인으로
요청을 했는지에 따라 매칭해주는 지시어이다. 예를 들면, listen 80; 이지만, server_name지시어의 값을 
real-test.com으로 했으면 클라이언트가 브라우저 주소창에 real-test.com으로 입력해야지 해당 server 블록
지시어로 요청이 매칭이 된다는 의미이다. 만약 www.real-test.com이나 혹은 real-test.net처럼 요청이 들어오면
server_name 지시어값과 달라서 해당 server 블록에는 매칭되지 않는다.

이를 조금 더 기능적으로 기술하자면,    
클라이언트의 요청(request)의 header에 명시된 도메인값이 server_name값과 일치하는 경우 server블록에 분기해준다는 의미이다.

<br>

> 여기서 server_name에 들어갈 수 있는 값은 하위도메인(서브도메인, ex)www.도메인.com, m.도메인.com)이나 최상위 도메인이
> 다른 도메인 ( ex)real-test.net, real-test.co.kr)이 들어갈 수 있다.

<br>

> server_name에 대한 개념을 보기위해 지시어를 넣어서 적어주었으나, 우리는 server_name지시어를 적지않고
> 배포하도록 하겠다. 뒤에 하위도메인에 대한 처리나 리다이렉팅을 위해서 필요한 개념이니 반드시 알고가자.

<br>

> [server_name의 개념 (1)](https://swiftcoding.org/nginx-routing)    
> [server_name의 개념 (2)](https://narup.tistory.com/209)      

<br>

```conf
  server {
      listen        80 default_server;
      listen        [::]:80 default_server;

      access_log    /var/log/nginx/access.log main;
  }
```

여기서는 access_log 지시어에 대해 보도록 하겠다.
위 access_log는 nginx로 들어오는 요청이 해당 server에서 받아서 처리할 경우
그에 대한 로그을 담을 파일의 저장위치를 지정해 주는것이다.

access_log 지시어 값의 맨뒤 main은 이 전 log_format  main
에서 log출력형식을 지정하는 지시어에서 main 이름(alias)으로 설정된 format
을 사용하겠다는 의미이다.

<br>

> 만약 server블록 내에서 access_log를 쓰지 않고, http블록 바로 하위나 아니면 루트 컨텍스트에서
> access_log 지시어를 쓰면 모든 server블록에 대한 로그가 한 파일에 담기게 된다. 그렇게 되면, 나중에 로그분석을
> 할 경우 어려움이 많아지니 server블록 단위로 access_log 지시어를 사용하여 각기 다른 파일에 저장해 주는게 좋다.

<br>

> [access_log 지시어에 관하여 (1)](https://kscory.com/dev/nginx/install)
> [access_log 지시어에 관하여 (2)](https://youngwonhan-family.tistory.com/93)

<br>

```conf
  server {
      listen        80 default_server;
      listen        [::]:80 default_server;

      access_log    /var/log/nginx/access.log main;

      client_header_timeout 60;
      client_body_timeout   60;
      keepalive_timeout     60;
      gzip                  off;
      gzip_comp_level       4;

      # Include the Elastic Beanstalk generated locations
      include conf.d/elasticbeanstalk/healthd.conf;
  }
```

나머지 코드들을 정리해 보도록 하겠다.

* client_header_timeout : 클라이언트와 서버가 연결된 후 지정된 시간안에 클라이언트가 온전한 헤더 전체를
  전송하지 않으면 해당 요청은 제거되고, 408(Request Time-out)로 끝난다. 디폴트 값은 60초다.

* client_body_timeout : 클라이언트가 서버로 데이터를 보냈을때 즉, 요청(Request) body를 보냈을때, 

* keepalive_timeout : 

* gzip : HTTP 응답(Response)에서 데이터를 클라이언트에게 보낼 때 압축해서 보낼지 아니면 그냥
  보낼지 설정할 수 있다.

* gzip_comp_level : gzip의 압축기능의 압축률을 지정하는 설정이다. 

<br>

> 위의 gzip과 gzip_comp_level은 HTTP 응답(Response)의 데이터 전송속도에 영향을 미치기 때문에 조금 더
> 자세히 알아보기 위해서 따로 [부록](https://sooolog.dev/Nginx-gzip-%EC%84%A4%EC%A0%95%EC%9D%84-%ED%86%B5%ED%95%9C-%ED%85%8D%EC%8A%A4%ED%8A%B8-%EB%8D%B0%EC%9D%B4%ED%84%B0(HTML,-CSS,-Javascript,-JSON/)-%EC%86%8D%EB%8F%84-%ED%85%8C%EC%8A%A4%ED%8A%B8/)으로 글을 정리해두었다.
> 꼭 참고하길 바란다.

<br>

> []()    
> []()    
> []()

<br>

정리하자면,      
그렇기에, server블록은 하나의 웺사이트를 선언하는데 사용되며, server 블록이 여러개이면, 한대의 머신(호스트)에 여러 웹사이트를 서빙할 수
있게되는것이다.(server_name으로 여러 도메인을 지정할 수 있기때문) 여기서 호스트란 EC2로 보아도 되고, Nginx 웹서버로 볼 수도 있다.

이렇게, 실제로 호스트는 한대지만, 여러 웹사이트를 서빙하기에 마치 가상으로 호스트가 여러개 존재하는 것처럼
동작하게 되기에 이런 개념을 가상 호스트라고 한다. server 블록 자체가 가상 호스팅을 가능하게 하는것이다. 

<br>

> [server 블록이란 (1)](https://prohannah.tistory.com/136)    
> [server 블록이란 (2)](https://juneyr.dev/nginx-basics)

<br>aaaaaaaaaaa

다음, access log가 어디 저장될지 보이는건데, 이는 이 서버에 해당하는 거에 대해서만
로그를 남긴다. 이것도 상속의 개념aaa

<br>

server 블록 또한 여러개 만들 수 있는데, 그렇게되면 한대의 머신(=호스트)에
여러 웹사이트를 서빙할 수 있게 되어 실제로 호스트는 한대지만, 가상으로 마치 호스트가 여러개 존재하는 것처럼 동작하게 할 수 있기에 가상 호스트라고 한다.

조금 더 쉽게 말하자면, http 컨텐스트 내에 아래와 같이    

그리고 http 블록도 server, location이런거 끝내고 개념 한번 더 정리

```conf
server {
    server_name test123.com
}

server {
    server_name test456.com
}
```

<br>

> 위에는 적혀져 있지 않지만, server_name이라는 지시어를 server 컨텐스트내에 사용할 수 있다.
> 

<br>

```conf
      location / {
          proxy_pass          http://springboot;
          proxy_http_version  1.1;
          proxy_set_header    Connection          $connection_upgrade;
          proxy_set_header    Upgrade             $http_upgrade;

          proxy_set_header    Host                $host;
          proxy_set_header    X-Real-IP           $remote_addr;
          proxy_set_header    X-Forwarded-For     $proxy_add_x_forwarded_for;
      }
```

마지막으로, location 블록 지시어를 보겠다.
a

<br>

아니 근데, 이거하고 디렉티브니 이런 기본 개념도 정리해야해, 블럭이니

(3). http 블록 : 웹서버에 대한 동작을 설정하는 영역으로 server블록과 location블록 그리고 upstream블록의 루트 블록이다. 여기서 선언된
값은 하위블록에 상속되어, 서버의 기본값이 된다.

(4). server블록과 location블록 : server블록은 하나의 웹사이트를 선언하는데 사용되며, 가상 호스팅의 개념이다. location블록은 server블록 내에서
특정 URL을 처리하는 방법을 정의한다. 

(5). upstream블록 : origin서버라고도 하며, 여기서는 WAS를 의미한다. nginx는 downstream에 해당한다.


<br>

> 위의 nginx.conf설정 그대로 빈스톡 환경 구성의 소프트웨어 환경 속성에서 PORT로 8080 혹은 
> 5000으로 설정하고 해도 정상적으로 작동한다.(특히, PORT 5000은 빈스톡 환경을 생성하자마자 자동으로 소프트웨어
> 환경 속성에 적혀져있는데, 위의 nginx.conf설정이 담긴 프로젝트를 배포하면 해당 환경 속성 PORT 5000이 없어진다.)      
> [PORT 8080 적용](https://stackoverflow.com/questions/54612962/502-bad-gateway-elastic-beanstalk-spring-boot)

<br>

```conf
user                    nginx;
error_log               /var/log/nginx/error.log warn;
pid                     /var/run/nginx.pid;
worker_processes        auto;
worker_rlimit_nofile    33282;

events {
    use epoll;
    worker_connections  1024;
    multi_accept on;
}

http {
  include       /etc/nginx/mime.types;
  default_type  application/octet-stream;

  log_format  main  '$remote_addr - $remote_user [$time_local] "$request" '
                    '$status $body_bytes_sent "$http_referer" '
                    '"$http_user_agent" "$http_x_forwarded_for"';

  include       conf.d/*.conf;

  map $http_upgrade $connection_upgrade {
      default     "upgrade";
  }

  upstream springboot {
    server 127.0.0.1:8080;
    keepalive 1024;
  }
  server {
      listen 80;
      server_name celebmine.net www.celebmine.net;
      return 301 http://celebmine.com$request_uri;
  }

  server {
      listen 80;
      server_name celebmine.co.kr www.celebmine.co.kr;
      return 301 http://celebmine.com$request_uri;
  }

  server {
      listen        80 default_server;

      if ($host = www.celebmine.com) {
          return 301 http://celebmine.com$request_uri;
      }

      location / {
          proxy_pass          http://springboot;
          proxy_http_version  1.1;
          proxy_set_header    Connection          $connection_upgrade;
          proxy_set_header    Upgrade             $http_upgrade;

          proxy_set_header    Host                $host;
          proxy_set_header    X-Real-IP           $remote_addr;
          proxy_set_header    X-Forwarded-For     $proxy_add_x_forwarded_for;
      }

      access_log    /var/log/nginx/access.log main;

      client_header_timeout 60;
      client_body_timeout   60;
      keepalive_timeout     60;
      gzip                  off;
      gzip_comp_level       4;

      # Include the Elastic Beanstalk generated locations
      include conf.d/elasticbeanstalk/healthd.conf;
  }
}
```

마지막으로 정리된 nginx.conf파일을 보자면 위와같이 적어주면 된다.

<br>

#### 🪁 Reference
* 참조링크 : []()
* 참조링크 : []()

<br>



### 3. server_tokens off 로 버전정보 노출안되게해서 보안 up


1.아래, port 5000이랑, server_sport 5000이랑은 달랐었다. 이거 체크하자.
2.그리고 nginx가 리버스 프록시이면서 웹서버이다 ? 라는것도 정리하




### 🚀 추가로

<p align="center">
<img src="https://user-images.githubusercontent.com/59492312/152488115-f6e4b0d7-2953-4d84-ac7e-e122c588f229.png">
</p>

위에 보이는것처럼 Nginx가 80번포트로 받은 외부의 요청들을 5000포트로 그대로 요청을 보내며, 스프링부트 어플리케이션을 5000포트로 실행시키는 방법에 대해서도 보고 가도록
하겠다.

첫번째는, 스프링부트 설정파일인 application.properties 또는 application.yml에      
```properties
server.port:5000   
```
(properties설정파일)
```yml
server:   
  port: 5000
```
(yml설정파일)    
와 같이 적어주면, 스프링부트 프로젝트가 실행될때 해당 5000포트에서 실행이 된다.

<br>

<p align="center">
<img src="https://user-images.githubusercontent.com/59492312/152491989-a3c18aa9-7581-4be7-8cfc-2b1fe0b3f23e.png">
</p>
<p align="center">
<img src="https://user-images.githubusercontent.com/59492312/152515745-47e37589-6117-41b5-9e72-b54ad47e83ec.png">
</p>

두번째 방법은, 바로 빈스톡 환경 생성을 할 때 추가 옵션 구성에서 소프트웨어 편집을 누르고
그 안의 환경 속성에서 SERVER_PORT와 5000을 적어주고 적용하면, 이 또한 스프링부트 어플리케이션이 5000포트에서
실행되게된다.

<br>

> 그런데, properties나 yml설정파일을 사용하는것이나, 빈스톡 환경 구성에서 직접 SERVER_PORT를 5000으로 잡아주는 방식은
> 어디까지나, 위에 작성한 nginx.conf파일처럼 포트를 직접 설정해주지 않았을 때 이야기다. nginx.conf 파일도 그대로 사용하면서 바로 위의
> 설정들도 함께 사용하지 말길 바란다.

<br>

#### 🪁 Reference

* 참조링크 : [Nginx 포트 5000 그대로 사용하는 방법 (1)](https://browndwarf.tistory.com/66)    
* 참조링크 : [Nginx 포트 5000 그대로 사용하는 방법 (2)](https://stackoverflow.com/questions/54612962/502-bad-gateway-elastic-beanstalk-spring-boot)    
* 참조링크 : [Nginx 포트 5000 그대로 사용하는 방법 (3)](https://wky.kr/6)

<br>
