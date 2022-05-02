---
layout: post
title:  "[스프링부트] Github Action과 Beanstalk으로 CI,CD 하기 A부터 Z까지 (5) [완]"
categories: [ 빌드와 배포 ]
image: https://user-images.githubusercontent.com/59492312/163088078-f5b45b97-edb2-4b24-88ec-7b70c813ff2a.png
---

# 📖 [스프링부트] Github Action과 Beanstalk으로 CI/CD 하기 A부터 Z까지 (5) [완]

* 리버스 프록시로써 작동하는 Nginx
* Nginx 설정파일인 nginx.conf 작성
* 최종 배포시 발생하는 문제들

> 모든 코드는 [깃헙](https://github.com/sooolog/dev-spring-springboot)에 작성되어 있습니다.

* * *

<br>



### 1.리버스 프록시로써 작동하는 Nginx

<p align="center">
<img src="https://user-images.githubusercontent.com/59492312/152483197-fe279aef-46bc-4a51-8b4e-f5d4bd19c091.png">
</p>

Nginx는 무중단 배포로도 사용이 되지만, 여기서는 Nginx가 받은요청을 스프링부트 임베디드 톰캣(WAS)로
보내는 역활만 하는 리버스 프록시로 보도록 하겠다.

> 여기서는 Nginx로 무중단 배포를 사용하지 않겠다.(참고로, Nginx로 무중단 배포를 사용해도 리버스 프록시로써 역활도 동시에한다.)

Nginx.conf내에 쓰는 코드들을 설명하기에 앞서, 빈스톡 환경에서 리버스 프록시로써의 역활에 대해
필수적으로 알아야 할 기본개념들을 보고 가도록 하겠다. 꼭 참고하기를 바란다.

> 빈스톡에서는 로드벨런서(Application Load Balancer)가 무중단 배포를 대신해서 수행한다.

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
 
* **심플 디렉티브** : 세미콜론(;)으로 끝나는 디렉티브이다. 위의 디렉티브중 user,error_log,pid가 심플디렉티브이다.
 
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
그 중에 nginx.conf 파일이 가장 주요한 설정파일이다.

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

http 블록은 Nginx 서버에 대한 동작을 설정하는 영역으로, server, location 블록을 포함한다.
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

이제는 upstream 블록 디렉티브와 그 안의 다른 디렉티브들인 server, keepalive에 대해서 알아보겠다.   

upstream 블록 지시어는 origin 서버(server)를 가리키는데, 위에서는 WAS를 가리키고 있다. upstream 블록 지시어 안에 
심플 지시어인 server 값으로 origin 서버(server)의 호스트명(혹은 ip주소)과 포트를 적어주어서 설정하게 된다. 이렇게 설정해주어서
origin 서버(server)를 가리키는 upstream 블록 지시어를 사용하는 이유는 후에 nginx로 HTTP 요청(Request)이 들어오게 되면 
우리가 설정한 upstream의 origin 서버(server)로 요청을 다시 보내기 위함이다. 

upstream 블록 지시어까지는 origin 서버(server)를 설정하기 위함이고, Nginx으로 들어오는 HTTP 요청(Request)을
다시 origin server(여기서는 WAS)로 보내는 역활(=리버스 프록시)을 하는 지시어는 뒤에서 location 블록 디렉티브안에 있는 
proxy_pass 심플 디렉티브에서 더 자세히 보도록 하겠다.

다음으로 upstream 다음에 명칭을 적게되는데 위에서는 springboot라고 적었지만, 얼마든지 내가 해당 origin 서버(server)를
표현하고자 하는 명칭을 적어주면 된다. 이 부분이 뒤에 나올 proxy_pass에서 사용하게 된다.

<br>

> origin 서버는 하나의 컴퓨터로써 들어오는 요청에 대해 응답할 수 있게 구조화된 서버를 의미한다. 여기서는
> WAS가 origin 서버이다.     
> [origin server의 개념](https://www.cdnetworks.com/ko/knowledge-center/what-is-origin-server/)

<br>

> upstream의 한국어 의미는 상류이고, downstream의 한국어 의미는 하류이다. 물이 흘러 내려가서 받는 곳이 하류(downstream)이고
> 윗쪽에서 물을 흘러 내려보내는 곳이 상류(upstream)이다. 물을 데이터 패킷으로 비유 하자면, 네트워크에서 데이터를 보내는 쪽 즉, 흘러 보내는
> 쪽이 상류(upstream)이고, 데이터 패킷을 받는쪽은 하류(downstream)가 되는것이다. 위의 upstream 지시어가 가리키는 origin 서버(server)는
> upstream이 되고 우리는 WAS로 지정했기에 WAS가 upstream이 되는것이다.('상류'로써 들어오는 요청에 응답을 하며 데이터를 내보내는 쪽이기때문) 
> Nginx는 이 경우 WAS로 부터 데이터를 받는 쪽이니 downstream이 되는것이다.        
> [upstream 지시어의 의미](https://developer88.tistory.com/299)     

<br>

> upstream 블록 지시어 안에 있는 심플 지시어 server는 아래에서 볼 server 블록 지시어와는 다른것이다.
> 지시어의 명칭은 같지만 하나는 심플 지시어이고 다른 하나는 블록 지시어이다.

<br>

> [upstream 지시어와 server 지시어의 개념과 역활 (1)](https://developer88.tistory.com/299)      
> [upstream 지시어와 server 지시어의 개념과 역활 (2)](https://nginx.org/en/docs/http/ngx_http_upstream_module.html#upstream)      
> [upstream 지시어와 server 지시어의 개념과 역활 (3)](https://juneyr.dev/nginx-basics)      
> [upstream 지시어와 server 지시어의 개념과 역활 (4)](https://hyeo-noo.tistory.com/205)     

<br>

마지막으로 keepalive는 해당 origin 서버로 통신을 할 때 캐싱할 커넥션 수를 의미한다.
조금 더 쉽게 말하자면 keepalive기능을 사용하는데 몇개의 커넥션들에 대해 keepalive를 사용할지
개수를 설정하는 지시어다.

만약, 위에 처럼 keepalive 1024;이면 keepalive를 사용할 커넥션 수를 1024개로 설정한다는
의미이다. keepalive 값을 설정했는데 만약 요청이 계속해서 들어오고 keepalive가 적용된 커넥션 수가
설정 값에 다다르게 되면 초과되는 요청에 대해서는 LRU(Least Recently Used)에 따라서 가장 최근에
사용되지않은 keepalive 커넥션의 소켓 연결을 닫는다.

<br>

> [upstream 지시어의 keepalive 하위 지시어 (1)](https://nginx.org/en/docs/http/ngx_http_upstream_module.html#keepalive)    
> [upstream 지시어의 keepalive 하위 지시어 (2)](https://fuirosun.tistory.com/entry/nginx-keepalive-%EC%84%A4%EC%A0%95%ED%95%98%EA%B8%B0)    

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

      client_max_body_size  10m;
      client_header_timeout 60;
      client_body_timeout   60;
      keepalive_timeout     60;
      server_tokens         off;
      gzip                  on;
      gzip_comp_level       4;

      # Include the Elastic Beanstalk generated locations
      include conf.d/elasticbeanstalk/healthd.conf;
  }
```

나머지 코드들을 정리해 보도록 하겠다.

* client_max_body_size : 클라이언트가 보내는 HTTP 요청(Request)에서 body 사이즈의 크기를 제한하는 것이다.

* client_header_timeout : 클라이언트와 서버가 연결된 후 지정된 시간안에 클라이언트가 온전한 헤더 전체를
  전송하지 않으면 해당 요청은 제거되고, 408(Request Time-out) 에러로 끝난다. 디폴트 값은 60초다.

* client_body_timeout : 클라이언트가 서버로 데이터를 보냈을때 즉, 요청(Request) body를 보냈을때
  request body 전체 전송 시간에 대한 것은 제외한, 두개의 연속적인 읽기 작업 사이의 timeout 시간이다.
  디폴트 값은 60초이며, 만약 클라이언트가 이 시간안에 아무것도 전송하지 않는다면 요청(request)은 제거되고
  408 에러를 발생시킨다.

* keepalive_timeout : HTTP 요청(Request)을 처리한 후에 연결을 끊지말고 유지하라는 KeepAlive 모드가 있다.
  이 keepalive 상태 즉, 연결을 끊지않고 유지하는 시간 값을 입력하면 된다. 디폴트값은 75s(75초)이며 일부 브라우저 에서는
  60s(60초)까지만 유지하기에 60초로 설정이 된다.

* server_tokens : 에러페이지나 아니면 서버의 응답(Response) 헤더에 어떤 nginx의 버전이 쓰였는지를
  명시하거나 아니면 명시하지 않을지를 설정하는 부분이다. 디폴트값은 on이다. Nginx의 버전이 노출되면 이를 이용하여
  악의적인 공격이 들어올 수 있어 보안상 off값으로 명시해주는게 좋다.

* gzip : HTTP 응답(Response)에서 데이터를 클라이언트에게 보낼 때 압축해서 보낼지 아니면 그냥
  보낼지 설정할 수 있다.

* gzip_comp_level : gzip의 압축기능의 압축률을 지정하는 설정이다. 

이 외에도 성능과 속도와 관련된 디렉티브들이 많이 있다. 기본적인 세팅은 이대로 진행하며, 추가적인 디렉티브의 설정과 개념들에서는 다시한번 정리하도록 하겠다.

<br>

> client_max_body_size는 위에서도 보았듯이 클라이언트가 보내는 HTTP 요청(Request)의 body 사이즈 크기제한을 설정하는 디렉티브이다.
> 실제로 실 서비스에서 사용할 때는 여러 상황들을 고려하여 값을 지정해야 한다. 우선 여기서는 10m(=10mb)으로 지정하도록 하겠다. 
> 따로 디렉티브값을 명시해 주지않으면 디폴트값은 1m(1mb)이다. 만약 클라이언트에서 요청을 보낼 때 해당 디렉티브 값을 넘어서 요청을 보내면,
> 413 에러가 발생하며 브라우저에서는 이에 대한 오류를 명확하게 보여주지 않으니 미리 인지하고 설정값을 지정하여야 한다.(request의 Content-Length 헤더값은
> client_max_body_size에 설정된 값을 초과할 수 없다)     
> [Nginx의 client_max_body_size 디렉티브 (1)](https://nginx.org/en/docs/http/ngx_http_core_module.html#client_max_body_size)    
> [Nginx의 client_max_body_size 디렉티브 (2)](https://velog.io/@jongwoo328/Nginx-request-size-%EB%B3%80%EA%B2%BD%ED%95%98%EA%B8%B0)     
> [Nginx의 client_max_body_size 디렉티브 (3)](https://ohdowon064.tistory.com/293)

<br>

> 위의 gzip과 gzip_comp_level은 HTTP 응답(Response)의 데이터 전송속도에 영향을 미치기 때문에 조금 더
> 자세히 알아보기 위해서 따로 [부록](https://sooolog.dev/Nginx-gzip-%EC%84%A4%EC%A0%95%EC%9D%84-%ED%86%B5%ED%95%9C-%ED%85%8D%EC%8A%A4%ED%8A%B8-%EB%8D%B0%EC%9D%B4%ED%84%B0(HTML,-CSS,-Javascript,-JSON/)-%EC%86%8D%EB%8F%84-%ED%85%8C%EC%8A%A4%ED%8A%B8/)으로 글을 정리해두었다.
> 꼭 참고하길 바란다.

<br>

> [nginx.conf의 디렉티브들에 관하여 (1)](https://nginx.org/en/docs/http/ngx_http_core_module.html)    
> [nginx.conf의 디렉티브들에 관하여 (2)](https://yangbongsoo.tistory.com/12)    
> [nginx.conf의 디렉티브들에 관하여 (3)](https://intrepidgeeks.com/tutorial/nginx-timeout-setting)     
> [nginx.conf의 server_tokens와 보안](https://goodgid.github.io/Nginx-Option-Server-Tokens/)

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

마지막으로 location 블록 지시어와 location 블록 지시어안의 심플 지시어들을 보도록 하겠다.

location은 server 블록 지시어의 하위 블록 지시어로 특정 url을 처리하는 지시어다.
조금 더 쉽게 말하자면, server 블록 지시어는 www.도메인A.com, 도메인A.com, 도메인A.net처럼 최상위 도메인이나 아니면
하위도메인 혹은 도메인B.com과 같이 아예 도메인 이름이 다른 경우를 모두 식별하여 HTTP 요청(Request)에 맞게 연결하지만
location은 server로 받아온 도메인에서 특정 세부 url을 처리하는데 사용한다. 예를들면 도메인A.com/one, 도메인A.com/two
로 요청이 들어오면 각각 다른 origin서버나 혹은 정적파일을 제공해줄 수 있는것이다. 예를 보면 더 정확하게 알 수 있다.

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

location /one {
    root /home/deploy/main
    index index.html
  }

location /two
    return 200;
}
```

이처럼, server 블록지시어안에 location 블록 지시어를 여러개 지정할 수 있으며
만약 server 블록이 도메인A.com 요청을 받아들이면 위의 location /로 매칭이 되고 도메인A.com/one 요청이 들어와
받아들이면 location /one으로 매칭이 되는것이다.

만약 위와같이 설정하였는데 요청이 도메인A.com/one/main으로 온다면 Nginx는 그 중 가장 길게 매칭이 되는
location 블록과 연결시킨다. 도메인A.com/one/main이면 location /one과 가장 길게 매칭이되니 location /one으로
연결이 되는것이다.

<br>

> 위에서 location /은 우리가 작성한 upstream 블록지시어와 매칭시켜주어 origin 서버인 WAS로 보내주기
> 때문에 Nginx를 리버스 프록시로써 사용할 수 있다. location /one의 경우에는 요청이 들어오면 정적자료를 반환하기에
> 이 경우는 Nginx가 웹 서버로써 작동하는 거다. 마지막으로 location /two는 status code(상태코드)를 반환하는 것이다.   

<br>

> Nginx는 정적파일을 서빙하는 웹 서버로써 사용하거나 혹은 리버스 프록시로써 사용할 수 있는데, 이곳에서는 리버스 프록시로써만 Nginx를
> 사용하려 한다.        
> [웹 서버 혹은 리버스 프록시로써의 Nginx](https://juneyr.dev/nginx-basics)

<br>

> [location 지시어에 관하여 (1)](https://juneyr.dev/nginx-basics)     
> [location 지시어에 관하여 (2)](https://narup.tistory.com/209)     
> [location 지시어에 관하여 (3)](https://architectophile.tistory.com/11)     

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

우리는 리버스 프록시로써 작동하는 이 코드들에 대해 마지막으로 보도록 하겠다.

* proxy_pass : HTTP 요청(Request)이 location과 매칭되었을때 해당 요청을 어디로 보낼지 나타내는 지시어이다.
  값으로는 IP주소:포트번호로 설정하거나, 도메인명 아니면 우리가 위에서 설정한 upstream의 명칭을 가져다가 사용할 수 있다. 위처럼
  http://springboot로 적어주면 이는 127.0.0.1:8080을 의미하게 된다. 또한, http:// 혹은 https://를 같이 작성하여
  어느 규약을 사용할것인지 알려준다.(작성해주지 않으면 EC2로 배포가 되지 않는다.)

* proxy_http_version  : 디폴트 값은 1.0이다. keepalive 지시어를 사용하려면 버전 1.1을 사용하여야 한다.

* proxy_set_header Host : Nginx에서 proxy_pass로 요청이 넘어갈때 헤더값들이 이전 클라이언트가 보낸 HTTP 요청(Request)의 헤더 값들이 그대로 상속되어서 넘어가게 된다.
  하지만, proxy_set_header Host와 proxy_set_header Connection는 상속된 값이 아니라 디폴트값으로 재설정이 되어서 Nginx에서 origin 서버로 보내지는데
  그때의 값은 proxy_set_header Host  $proxy_host; 와 proxy_set_header Connection close; 이다. 하지만, proxy_set_header Host  $proxy_host;
  의 경우 클라이언트가 보낸 요청의 헤더에서 $proxy_host와 일치하는 항목이 없으면 빈값으로 보내지게 되고 그렇게 되면 오류가 발생하게 된다. 그렇기에 
  proxy_set_header Host  $host; 와 같이 작성하여 제대로 된 Host값이 헤더에 담아져서 보내지게 해야한다.
  
* proxy_set_header Connection : 해당 지시어는 upstream 지시어를 사용하기 위한것이기 때문에 
  proxy_set_header Connection  $connection_upgrade; 로 다시 명시해주어야 한다.
  
* proxy_set_header X-Real-IP : 실제 클라이언트의 IP를 헤더의 X-Real-IP에 값으로 넣어서 전송시킨다.
  조작이 불가능하고, 여러 프록시 서버를 거치더라도 실제 처음 요청을 한 클라이언트의 ip주소만을 담고있다.

* proxy_set_header X-Forwarded-For : 이 또한 X-Forwarded-For에 클라이언트 IP 주소를 넣어서 전송시킨다.
  다만, 이전에 프록시 서버를 거쳐서 온 요청이라면 처음 요청을 한 클라이언트가 아닌 바로 전 프록시 서버의 ip주소가 담아지게 된다.
  또한, 이 헤더값은 조작이 가능하기에 보통 proxy_set_header X-Real-IP와 proxy_set_header X-Forwarded-For를 같이
  작성하여 사용한다.

* proxy_set_header Upgrade : 이는 upgrade 기능을 사용함으로써 HTTP에서 웹소켓으로 커넥션을 업그레이드 시켜준다.
  이는 나중에 웹소켓을 사용하게 되는 경우에 다시 보도록 하자.

<br>

> 한가지 의문점이 들수도 있다. 왜 여기서는 굳이 proxy_pass의 값으로 ip주소:포트번호를 쓰지 않고 upstream의
> 명칭을 쓰냐는것인데, 이렇게 upstream 블록 지시어를 사용하게 되면, keepalive 심플 지시어같은 추가적인 설정을 할 수 있기
> 때문에 사용을 한다.

<br>

> Nginx를 프록시로 앞단에 두고 Tomcat을 뒷단에 두면, 별도 설정 없이는 톰캣(WAS)에서는
> 클라이언트 IP를 알 수가 없다. 그렇기에 proxy_set_header X-Real-IP와 proxy_set_header X-Forwarded-For
> 를 사용해서 클라이언트의 ip주소를 확인한다.     
> [X-Real-IP와 X-Forwarded-For를 사용하는 이유](https://sg-choi.tistory.com/540)

<br>

> [proxy 지시어에 관하여 (1)](https://nginx.org/en/docs/http/ngx_http_proxy_module.html#proxy_pass)    
> [proxy 지시어에 관하여 (2)](https://dev-jwblog.tistory.com/42)    
> [proxy 지시어에 관하여 (3)](https://nginx.org/en/docs/http/ngx_http_proxy_module.html#proxy_http_version)    
> [proxy 지시어에 관하여 (4)](https://developer88.tistory.com/299)    
> [proxy 지시어에 관하여 (5)](https://velog.io/@csk917work/Nginx-%EC%84%9C%EB%B2%84-%EC%84%A4%EC%A0%95)    
> [proxy 지시어에 관하여 (6)](https://sg-choi.tistory.com/540)       
> [proxy 지시어에 관하여 (7)](https://dev-gorany.tistory.com/330)     

<br>

정리하자면 server블록은 하나의 웹사이트(도메인)를 선언하는데 사용되며, server 블록이 여러개면
한대의 호스트에 여러 웹사이트(도메인)를 서빙할 수 있게되는것이다.(server_name으로 여러 도메인을 지정할 수 있기때문)

이렇게, 실제로 호스트는 한대지만, server 블록을 여러개 만들어 설정하게되면, 하나의 호스트에
여러 웹사이트(도메인)를 서빙할 수 있게 되어, 마치 가상으로 호스트가 여러개 존재하는 것처럼 동작하기에
이를 가상 호스트라고 한다. server 블록 자체가 가상 호스팅을 가능하게 하는것이다.

location의 경우에는 server 블록 지시어가 받아들인 도메인에 대해 더욱 세부적으로 나누어서
매칭을 해주고 Nginx를 리버스 프록시로써 작동시킬지 아니면 웹 서버로써 활용할지를 결정해준다.

<br>

> 여기서 호스트란 IP를 가지고 있는 양방향 통신이 가능한 컴퓨터를 의미한다. 조금 더 정확하게 얘기하자면
> 네트워크에 연결된 모든 종류의 장치를 노드(node)라고 하는데, 노드중에서도 네트워크 주소(IP)가 할당된 것을
> 호스트(host)라고 한다.    
> [호스트(host)란 무엇인가](https://m.blog.naver.com/jysaa5/221736421275)         

<br>

> [server 블록이란 (1)](https://prohannah.tistory.com/136)    
> [server 블록이란 (2)](https://juneyr.dev/nginx-basics)    

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

      client_max_body_size  10m;
      client_header_timeout 60;
      client_body_timeout   60;
      keepalive_timeout     60;
      server_tokens         off;
      gzip                  on;
      gzip_comp_level       4;

      # Include the Elastic Beanstalk generated locations
      include conf.d/elasticbeanstalk/healthd.conf;
  }
}
```

마지막으로 정리된 nginx.conf 파일의 코드들이다.

이제 깃헙으로 push를 진행하게 되면, 정상적으로 배포가 진행되게 된다.

<br>



### 3. 스프링부트 최종 배포시 발생하는 문제들

```java
@Controller
public class MainController {

    @GetMapping("/hi1")
    public String main1() throws Exception{
        return "/main";
    }

    @GetMapping("/hi2")
    public String main2() throws Exception{
        return "main";
    }
}
```

첫번째로, 위를 보면 @Controller 클래스의 코드를 보면 main1 메서드는
반환값이 "/main"이고 main2 메서드는 반환값이 "main"이다. 스프링부트를 로컬로
실행시키고 크롬에서 접속하면 두 메서드 모두 정상적으로 작동한다. 하지만, 실제 빈스톡과 Nginx를 사용하여
EC2로 배포시키면 크롬에서 "도메인/hi1"로 접속시 정상적으로 맵핑이 이루어지고 결과값이 나오지만
"도메인/hi2"로 접속하면 404 에러가 발생한다. 에러가 발생하는 이유는 해당 요청 도메인에 대해
컨트롤러에서 맵핑은 되지만 return 값이 정상적으로 전달되지 않기 때문이다.

<br>

```java
@RestController
public class MainRestController {
    
    @GetMapping("/hi3")
    public String main3() throws Exception{
        return "test(테스트)";
    }
}
```

두번째로는 처음 빈스톡을 사용하여 스프링부트 프로젝트를 EC2에 배포할때
한개라도 URL이든 맵핑되고 정상적으로 반환값이 존재하는 컨트롤러가 있어야 한다.
즉, 조금 더 쉽게 설명하자면 위에서 설명한대로 "도메인/hi1"에 대해서는 정상적으로
컨트롤러에서 맵핑이되고 값이 반환이 되는데 이러한것 없이 값이 정상적으로 반환되지않는
"main2" 메서드만 존재한다거나 아니면 적어도 RestController를 이용하여 문자열 반환 컨트롤러 메서드가
1개라도 존재하지 않는다면 배포를 진행할 때 Fail이 발생하게 된다. 그렇기에, 처음에 테스트를 할 때
컨트롤러의 경우 정상적으로 return값을 설정했는지 그게 아니라면 RestController를 이용하여
문자열이라도 출력하게 했는지를 체크해야 정상적으로 EC2에 배포되는것을 볼 수 있다.

<br>



### 🚀 추가로,

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
> 설정들도 함께 사용하면 안된다.

<br>

> 추가로, 스프링부트 설정파일에서 어플리케이션 실행 포트를 설정해주는 것외에 빈스톡 환경의 추가 옵션 구성에서
> 소프트웨어 편집을 열고 PORT로 8080 설정을 해주는 방법이 있다. 이는 Nginx가 가리키는 포트 지정이 아닌 스프링부트
> 어플리케이션이 실행되는 포트를 지정하는 설정이다.(PORT 5000은 빈스톡 환경을 생성하자마자 자동으로 소프트웨어 환경 속성에 
> 적혀져있기도 하다.)          
> [빈스톡 환경 구성 PORT 8080 설정 (1)](https://stackoverflow.com/questions/54612962/502-bad-gateway-elastic-beanstalk-spring-boot)     
> [빈스톡 환경 구성 PORT 8080 설정 (2)](https://stackguides.com/questions/54612962/502-bad-gateway-elastic-beanstalk-spring-boot)

<br>

#### 🪁 Reference

* 참조링크 : [Nginx 포트 5000 그대로 사용하는 방법 (1)](https://browndwarf.tistory.com/66)    
* 참조링크 : [Nginx 포트 5000 그대로 사용하는 방법 (2)](https://stackoverflow.com/questions/54612962/502-bad-gateway-elastic-beanstalk-spring-boot)    
* 참조링크 : [Nginx 포트 5000 그대로 사용하는 방법 (3)](https://wky.kr/6)

<br>



태그 : #Nginx, #Reverse Proxy, #리버스 프록시, #5000port, #simple directive, #block directive, #nginx.conf, #mapping error
