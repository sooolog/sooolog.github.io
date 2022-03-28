---
layout: post
title:  "[스프링부트] Github Action과 Beanstalk으로 CI/CD 하기 A부터 Z까지 (4)"
categories: [ 빌드와 배포 ]
image: https://user-images.githubusercontent.com/59492312/159111005-d86c6063-f0fd-4435-bed6-1078dac356c7.png
---

* deploy.yml 작성 
* 00-makeFiles.config
* Procfile 작성

> 모든 코드는 [깃헙](https://github.com/sooolog/dev-spring-springboot)에 작성되어 있습니다.

* * *

<br>



### 1.build할 때 사용했던 deploy.yml을 빈스톡까지 배포하게 설정해 보자.

```yml
      - name: Generate deployment package 
        run: |
          mkdir -p deploy
          cp build/libs/*.jar deploy/application.jar
          cp Procfile deploy/Procfile
          cp -r .ebextensions deploy/.ebextensions
          cp -r .platform deploy/.platform
          cd deploy && zip -r deploy.zip .  # (1)

      - name: Beanstalk Deploy
        uses: einaregilsson/beanstalk-deploy@v20
        with:
          aws_access_key: ${{ secrets.AWS_ACCESS_KEY_ID }}
          aws_secret_key: ${{ secrets.AWS_SECRET_ACCESS_KEY }}
          application_name: dev-real-test
          environment_name: Devrealtest-env
          version_label: github-action-${{steps.current-time.outputs.formattedTime}}
          region: ap-northeast-2
          deployment_package: deploy/deploy.zip  # (2)
```

빈스톡으로 배포할 수 있는 방법은 많지만, 우리는 Github Action으로 빌드하고 배포까지 간편하게 하나의 스크립트에서
진행하기 위해서, 기존에 작성해 두었던 deploy.yml 뒤에 추가로 작성해준 코드이다. 이해를 돕기위해 deploy.yml의 전체코드를
첨부하도록 하겠다.

<br>

```yml
name: dev-spring-springboot

on:
  push:
    branches:
      - master
  workflow_dispatch:

jobs:
  build:
    runs-on: ubuntu-latest
    steps:
      - name: Checkout
        uses: actions/checkout@v2

      - name: Set up JDK 1.8
        uses: actions/setup-java@v1.4.3
        with:
          java-version: 1.8

      - name: Grant execute permission for gradlew
        run: chmod +x ./gradlew
        shell: bash

      - name: Build with Gradle
        run: ./gradlew clean build
        shell: bash

      - name: Get current time
        uses: 1466587594/get-current-time@v2
        id: current-time
        with:
          format: YYYY-MM-DDTHH-mm-ss
          utcOffset: "+09:00"

      - name: Show Current Time
        run: echo "CurrentTime=${{steps.current-time.outputs.formattedTime}}"
        shell: bash

      - name: Generate deployment package
        run: |
          mkdir -p deploy
          cp build/libs/*.jar deploy/application.jar
          cp Procfile deploy/Procfile
          cp -r .ebextensions deploy/.ebextensions
          cp -r .platform deploy/.platform
          cd deploy && zip -r deploy.zip .

      - name: Beanstalk Deploy
        uses: einaregilsson/beanstalk-deploy@v20
        with:
          aws_access_key: ${{ secrets.AWS_ACCESS_KEY_ID }}
          aws_secret_key: ${{ secrets.AWS_SECRET_ACCESS_KEY }}
          application_name: dev-real-test
          environment_name: Devrealtest-env
          version_label: github-action-${{steps.current-time.outputs.formattedTime}}
          region: ap-northeast-2
          deployment_package: deploy/deploy.zip
```

그렇다면, 이제 이번에 새로배울 코드들의 각 줄의 의미에 대해 알아 보도록 하겠다.

<br>

```yml
      - name: Generate deployment package 
        run: |
          mkdir -p deploy
          cp build/libs/*.jar deploy/application.jar
          cp Procfile deploy/Procfile
          cp -r .ebextensions deploy/.ebextensions
          cp -r .platform deploy/.platform
          cd deploy && zip -r deploy.zip .  # (1)
```

우선, # (1)에 해당하는 코드들을 보겠다.     
이전에 우리가 Build를 진행하여 만들어진 Jar파일을 Beanstalk으로 배포하기 위해서는 zip파로 만들어주어야 한다.      
(1). mkdir -p deploy는 deploy 디렉토리를 만들어준다.    

<br>

> mkdir뒤에 나오는 -p는 부모디렉토리까지 만들어주는 옵션이다. 즉, mkdir -p test/deploy라고 한다면
> 만약 test 디렉토리가 없다면, test 디렉토리와 deploy디렉토리를 한번에 생성시킨다는 의미이다. 그렇기에 위의
> 코드에서 mkdir deploy만 적어주어도 된다. 추가로, 그냥 mkdir test/deploy를 하면 오류가 난다고 한다.               
> [mkdir options](https://phoenixnap.com/kb/create-directory-linux-mkdir-command)

<br>

(2). cp build/libs/*.jar deploy/application.jar는 내가 build해준 jar파일을 (1)에서 만들어준 디렉토리로 application.jar이라는 파일명으로 바꿔서 복사해주는것이다. 
이렇게 해주는 이유는 빌드하고 난뒤의 jar파일명은 버전과 타임스탬프로 파일명이 변경되기 때문에, 이를 application.jar로 통일 시키고 뒤에 나올 00-makeFiles.config내에서
application.jar로 지정한 jar파일명으로 어플리케이션을 실행시키기 위함이다.

<br>

> 뒤에 00-makeFiles.config 파일에 대해서도 보겠지만, application.jar파일 대신에 application123.jar을 하고
> 위의 cp build/libs/*.jar deploy/application123.jar을 해도 정상적으로 작동한다. 그러나, cp build/libs/*.jar deploy/application123.jar으로
> 바꿔주고 00-makeFile.config는 그대로 application.jar이라 적으면 정상적으로 작동하지않는다.(=어플리케이션이 실행되지않아 배포가 되지 않는다.)

<br>

(3). cp Procfile deploy/Procfile도 디렉토리 deploy안에 복사해서 넣어주는것이다.    
(4). 나머지 cp -r .ebextensions~ 와 cp -r .platform~은 각각의 하위 디렉토리와 파일들 모두를 deploy/.ebextensions 그리고
deploy/.platform에 그대로 복사하라는 의미이다.

<br>

> cp -r은 cp의 옵션으로 해당하는 디렉토리의 하위디렉토리와 모든파일들까지 한번에 복사해서 옮겨놓는다는 의미이다.       
> [cp -r에 관하여](https://jframework.tistory.com/6)      

<br>

(5). 마지막으로 cd deploy && zip -r deploy.zip은 deploy디렉토리로 들어가서 해당하는 모든 디렉토리와 파일들을 압축하라는 의미이다.

<br>

> zip -r은 해당 위치에 있는 파일들은 물론이고 디렉터리까지 압축하라는 의미이다.    
> [zip -r에 관하여](https://www.lesstif.com/lpt/linux-zip-unzip-80248839.html)   

<br>

정리하자면, Procfile, 00-makeFiles.config, nginx.conf를 jar파일과 함께 압축해주어서 하나의 zip파일안에
담고자 하는것이다. 이렇게 하나의 zip파일에 jar파일과 다 함께 압축하는 이유는 우리는 스프링부트의 embedded tomcat 을 사용하면서 
.jar로 배포하는 경우에는 jar에 .ebextenions 폴더를 포함시킬 수 없기 때문에 따로 배포전에 jar 파일과 함께 .ebextenions 폴더를 
하나의 zip 파일로 묶어서 배포해야 제대로 설정파일이 적용되기 때문이다.

<br>

> [jar파일과 .ebextensions를 하나의 zip파일안에 두는 이유](https://techblog.woowahan.com/2539/)

<br>

> 추가로, deploy.yml은 github action runner에서만 사용되기에 같이 압축파일에 들어가지 않는것이다.

<br>

```yml
      - name: Beanstalk Deploy
        uses: einaregilsson/beanstalk-deploy@v20
        with:
          aws_access_key: ${{ secrets.AWS_ACCESS_KEY_ID }}
          aws_secret_key: ${{ secrets.AWS_SECRET_ACCESS_KEY }}
          application_name: dev-real-test
          environment_name: Devrealtest-env
          version_label: github-action-${{steps.current-time.outputs.formattedTime}}
          region: ap-northeast-2
          deployment_package: deploy/deploy.zip # (2)
```

그 다음은, 압축한 파일을 빈스톡으로 배포하는것이다.     
여기서 자세히 보면, einaregilsson내에 있는 beanstalk-deploy의 버전20에 해당하는것을 사용한다고 나와있다.
이는 [Github Action 플러그인](https://github.com/marketplace/actions/beanstalk-deploy)이다. 
AWS CLI로 배포를 진행할 수도 있지만, 이렇게 스크립트안에 알집 압축과 배포까지 한번에 진행시키기 위해 사용한다. 링크를
클릭하고, Use latest Version을 클릭해보면      

<br>

```yml
- name: Beanstalk Deploy
  uses: einaregilsson/beanstalk-deploy@v20
```

와 같은 코드를 얻을 수 있다. 현재(글 작성일 기준으로) 최신버전은 v20이다.

<br>

> 위에 적어놓은 코드 그대로만 적용해주면 플러그인 사용이 가능한것이다. 따로 설치하거나 할 필요가 없다.

<br>

그 뒤로 나와있는 코드를 보겠다.

<br>

```yml
aws_access_key: ${{ secrets.AWS_ACCESS_KEY_ID }}
aws_secret_key: ${{ secrets.AWS_SECRET_ACCESS_KEY }}
```

우리가 기존에 받아놓았던, IAM 발급키이다. 여기서는 위의 코드처럼 적어주면 된다.
그러면, Github Action에서는 우리가 깃헙 레포지토리의 secrets로 등록한 secrets.AWS_ACCESS_KEY_ID 키와
secrets.AWS_SECRET_ACCESS_KEY 키를 불러와 각각 적용하게 된다.

<br>

> 앞에서 secrets.AWS_ACCESS_KEY_ID와 secrets.AWS_SECRET_ACCESS_KEY를 정확하게 입력해야 했던 이유가 여기서 적은 키워드와
> 동일해야지 Github Action에서 값을 매치시켜주기 때문이다.

<br>

```yml
application_name: dev-real-test
environment_name: Devrealtest-env
```

그 다음 코드는 갖고있는 IAM키로 접근할 빈스톡 환경을 지정해주는것이다. 보면, application name과
environment name을 적어주어서 어느 빈스톡 환경으로 연동할지를 설정하는것이다. 

<br>

<p align="center">
<img src="https://user-images.githubusercontent.com/59492312/152470883-ec2f15c3-a2b7-4cb7-bf48-4c53c61effd8.png">
</p>

우리가 이전 글에서 만들어놓은 환경을 들어가 보면, 어플리케이션 이름이 dev-real-test이고, 환경이름이 Devrealtest-env라는걸 알 수 있다.
이 명칭들을 적어서 넣어주면 된다.

<br>

```yml
version_label: github-action-${{steps.current-time.outputs.formattedTime}}
```

그 다음 볼 코드는 version_label이다. version label은 말 그대로 어떠한 버전인지 라벨을 붙였다고 생각하면 된다. 우리가
이전 Github Action에서 빌드하는 글을 보면, Get Current Time에 대해 설명한적이 있다. 위의 Deploy.yml 풀코드를 보아도 알 수 있는데
빌드를 완료하고 난 후 시점을 확보하고 그 시점을 배포하려는 어플리케이션의 버전 라벨로 사용한다는 것이다. 즉, github-action-${{steps.current-time.outputs.formattedTime}}
코드는 github-action-빌드한시간 으로 버전 라벨로 사용한다는 의미이다.

<br>

> deploy.yml에서 Show Current Time에서도 이 ${{steps.current-time.outputs.formattedTime}} 코드를 사용하여 빌드된 시점을 보여줬었다.

<br>

```yml
region: ap-northeast-2
deployment_package: deploy/deploy.zip # (2)
```

나머지 코드들을 한번에 정리하도록 하겠다.     
region: ap-northeast-2부터 보면, ap-northeast-2는 서울 리전을 의미한다. 그렇다면 왜 리전을 적어주어야 하는걸까?
바로 빈스톡 환경이름 즉, 환경name은 각 리전별로 고유한 이름이여야 한다. 기본적으로 어플리케이션 내에 여러환경이 있을 수 있기에 어플리케이션은 중복해서
빈스톡 환경을 생성할 수는 있지만, 빈스톡 환경의 이름은 각 환경마다 고유한 이름을 갖어야 된다는것이다. 고유한 이름의 범위는 바로 region내로 한정짓기 때문에
접근하려고 하는 빈스톡이 어느 리전에 있는지 구체화해서 적어주어야 하는것이다.

<br>

> region이 다르면 중복된 빈스톡 환경이름으로 생성할 수 있다.

<br>

deployment_package는 우리가 이전에 빈스톡으로 서버에 배포하기위해 Jar파일을 압축하여 zip파일로 만들었다고 했다.
이 zip파일을 배포하겠다는 의미이다.

이로써, deploy.yml작성을 마무리하고 다른 설정파일들을 보도록 하겠다.

#### 🪁 Reference
* 참조링크 : [deploy.yml 작성하기](https://jojoldu.tistory.com/549)

<br>



### 2. 00-makeFiles.config, Procfile을 작성해보고 개념에 대해 알아보겠다.

<p align="center">
<img src="https://user-images.githubusercontent.com/59492312/152473761-9c06c854-0f78-4014-9092-97e47dc72dc3.png">
</p>

00-makeFiles.config, Procfile를 작성하기전에 파일이 위치한 구조부터 보고가겠다.
보이는 바와같이, 00-makeFiles.config는 루트디렉토리에서 .ebextensions디렉토리를 만들어서 작성을 해주면 되고,
Procfile은 루트디렉토리에 바로 만들어 주면 된다.

<br>

```config
files:
    "/sbin/appstart" :
        mode: "000755"
        owner: webapp
        group: webapp
        content: |
            #!/usr/bin/env bash
            JAR_PATH=/var/app/current/application.jar

            # run app
            killall java
            java -Dfile.encoding=UTF-8 -jar $JAR_PATH
```

00-makeFiles.config파일부터 보도록 하겠다. 처음 드는 궁금증은 왜 .ebextensions라는 디렉토리안에 이 파일을 생성하느냐는 것이다.
그 이유는 바로 빈스톡은 EC2를 대신해서 생성해주기 때문에, 우리가 직접 EC2를 직접 생성할때처럼은 사용할 수가 없다. 
그래서 직접생성해서 사용했을때처럼 커스터마이즈를 하고싶을 때 이 .ebextensions라는 디렉토리아래에 설정파일을 두어서 적용시키는 것이다.

00-makeFile.config라는 .config 확장명을 갖은 파일안에  YAML이나 JSON형식으로 코드를 작성하면, 빈스톡에 배포시
해당 코드들이 사용이 되는것이다. 그럼, 이 .config라는 파일 확장자는 사실 크게 의미는 없지만 설정파일이라는 의미를 주기위해 .config로
적어주는것이고 그 안의 코드들은 YAML이나 JSON으로 적어주어 적용하게 되는것이다. 이 00-makeFiles.config 파일에서 00-xxx.config형태로
쓰는이유는, 파일의 나열이 alphabetical order 순서이기에, 적용 순서가 중요한대로 00,01로 붙여서 사용을 하는것이다.

<br>

>[.ebextension과 00-xxx.config파일명의 이해](https://techblog.woowahan.com/2539/)

<br>

이제 00-makeFiles.config안의 코드들에 대해 보도록 하겠다. 우선,
files로 시작을 하는데, 이는 파일을 만들거나 다운로드를 받게하는 코드이다. 여기서는
만드는 기능으로 적용이 된다. 

<br>

> [00-makeFiles.config 문법](https://techblog.woowahan.com/2539/)

<br>

우리가 배포한 zip파일이 압축이 풀리고, 어느 파일을 실행할지를 설정하는 어플리케이션 실행
스크립트를 생성하는것이다.

<br>

```config
    "/sbin/appstart" :
        mode: "000755"
        owner: webapp
        group: webapp
        content: |
            #!/usr/bin/env bash
            JAR_PATH=/var/app/current/application.jar

            # run app
            killall java
            java -Dfile.encoding=UTF-8 -jar $JAR_PATH
```

/sbin 아래에 스크립트 파일을 두면, 이는 전역에서 사용이 가능하기에 /sbin/appstart 라고 작성하여 appsatrt 스크립트 파일을 생성한다.
권한은 755를 갖으며, owner는 webapp이고, content를 갖은 스크립트 파일이 생성된다.

content안에 있는 JAR_PATH는 압축이 해제된 zip파일내에 있던 jar파일의 경로를 의미한다.

<br>

> 우리가 이전 글에서 build한 후 jar파일들과 여러 설정파일들을 압축해주는 과정에서 
> cp build/libs/*.jar deploy/application.jar 로 생성된 jar파일명을 application.jar로
> 통일해준것을 알 것이다. 이렇게 통일해 준 이유는 위의 JAR_PATH에서 jar파일명을 그대로 명시해주어야 하기 때문이다. 만약,
> build할 때 jar파일명을 application.jar로 통일해주지 않으면 JAR_PATH의 jar파일명도
> 매번 그에 맞게 수정해주어야 하기때문에 번거로워 진다. 그래서 앞선 글에서 JAR파일을 통일시켜준것이다.

<br>

그 아래, killall java는 기존에 실행중이던 어플리케이션을 종료하라는 의미이다. 하지만, 우리는
추가 배치를 이용한 롤링으로 새로운 인스턴스를 생성하여 배포해주고 기존의 인스턴스를 대체하는것이기 때문에
필요한 코드는 아니지만, 적어주어도 상관은 없다.

java -Dfile.encoding=UTF-8 -jar $JAR_PATH는 해당되는 Jar파일을 실행시킨다는
의미이며, -Dfile.encoding=UTF-8을 함으로써 파일 인코딩을 UTF-8로 실행하라는 의미이다. 

설정파일 00-makeFile.config로 인해 생성된 실행 스크립트의 content를 모두 보았다면,
이제 이렇게 생성한 appstart 스크립트 파일을 실행시키는 일만 남았다.
스크립트 파일의 실행은 Procfile에서 담당한다.

<br>

```text
web: appstart
```

Procfile에 작성한 내용이다.     
위에서 만든 어플리케이션 실행 스크립트를 실행시키는 일은 Procfile에서 하고
또, Procfile을 실행시켜야 Procfile이 작동하는 것이니 결국에는 배포된 어플리케이션의 실행은
Procfile을 실행시킨다는 의미와 같다.

#### 🪁 Reference
* 참조링크 : [00-mkaeFile.config, Procfile 작성](https://jojoldu.tistory.com/549)

<br>



### 🚀 추가로, nginx에 관한 설정파일인 nginx.conf에 대해서는 알아야 하는 양이 방대하기에, 다음 글에서 다음 글에서 정리하도록 하겠다.

<br>



태그 : #deploy.yml, #.ebextensions, #00-makeFiles.config, #Procfile, #하위블록으로 상속, #mkdir -p, #cp -r, #zip -r, #IAM
