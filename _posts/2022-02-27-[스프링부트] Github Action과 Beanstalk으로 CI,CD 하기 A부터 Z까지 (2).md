---
layout: post
title:  "[스프링부트] Github Action과 Beanstalk으로 CI/CD 하기 A부터 Z까지 (2)"
categories: [ 빌드와 배포 ]
image: https://user-images.githubusercontent.com/59492312/151473981-f0504fde-808a-4c1f-9cea-a887f2bb9ace.png
---

<p align="center">
.<br>   
.<br>
.<br>
</p>

<p align="center">
<img src="https://user-images.githubusercontent.com/59492312/151309696-7ab7e274-101f-4467-ba73-74c65cc872bf.png">
</p>

# 📖 [스프링부트] Github Action과 Beanstalk으로 CI/CD 하기 A부터 Z까지 (2)

* Github Action을 위한 deploy.yml작성과 이해
* bash, shell, git bash, vim, cli, 터미널의 기본개념 훑고가기
* 깃헙에 workflow사용을 위한 push에 Personal Access tokens로 권한 부여하기

> 모든 코드는 [깃헙](https://github.com/sooolog/dev-spring-springboot)에 작성되어 있습니다.
* * *

<br>
    
### 1.스프링부트에서 CI/CD는 Github Action과 Beanstalk으로 진행하도록 한다.

<p align="center">
<img src="https://user-images.githubusercontent.com/59492312/151473981-f0504fde-808a-4c1f-9cea-a887f2bb9ace.png">
</p>

기존에는 CI/CD 툴로 Travic Ci, Jenkins, AWS Code Deploy를 많이 사용했었다. 또는,      
Travis CI + AWS Code Deploy     
Travis CI + AWS Beanstalk   
와 같은 환경조합으로 CI/CD 파이프라인을 구축했다.

하지만, 대세가 Travis CI에서 Github Action으로 넘어가고있으며, Auto Scaling과 Load Balancer, 클라우드 워치 등을 한번에
관리할 수 있는 Beanstalk 또한 많이 사용되고 있기에 우리는 Github Action + Beanstalk으로 CI/CD 파이프라인을 구축해보도록 하겠다.

<p align="center">
<img src="https://user-images.githubusercontent.com/59492312/151483421-031667e6-0d78-4a8e-94a6-12b933cbff47.png">
</p>

Github Action과 Beanstalk를 사용했을 경우의 기본 구조는 이러하다.

#### 🪁 References
* 참조링크 : [기존 CI/CD 툴 (1)](https://artist-developer.tistory.com/24)
* 참조링크 : [기존 CI/CD 툴 (2)](https://jojoldu.tistory.com/543)

<br>



### 2.Github Action으로 빌드하기 위한, deploy.yml 작성

<p align="center">
<img src="https://user-images.githubusercontent.com/59492312/151495796-3c4e532a-bbb3-433e-939d-6aa87ccfddee.png">
</p>

Github repository의 action에서 템플릿을 만들어서 해당 코드들을 적용해도 되지만, 우리는 프로젝트내에서 deploy.yml을 작성하여 
진행해 보도록 하겠다.

위의 그림처럼 루트 디렉토리에서 .github/workflows/deploy.yml을 만들어 준다. 저처럼 .github 폴더에 아이콘이 뜨길
원한다면, [extra-icons 플러그인](https://plugins.jetbrains.com/plugin/11058-extra-icons)을 적용해주면 된다.
이 외에도 인텔리제이 아이콘 플러그인 검색해서 원하는것을 설치하고 적용해 주면 된다. 

<br>

> 필자는 CI/CD 관련 코드들을 프로젝트 내에서 작성하여 한번에 관리하는것을 선호한다. Github에서 작성하여 적용하는
> 방법도 있으니, 궁금하신분들은 찾아보셔서 한번쯤 공부해보는것도 좋다.

<br>

```yml
name: dev-spring-springboot

on:
  push:
    branches:
      - master # (1).브랜치 이름
  workflow_dispatch: (2).수동 실행

jobs:
  build:
    runs-on: ubuntu-latest # (3).OS환경

    steps:
      - name: Checkout
        uses: actions/checkout@v2 # (4).코드 check out

      - name: Set up JDK 1.8
        uses: actions/setup-java@v1.4.3
        with:
          java-version: 1.8 # (5).자바 설치

      - name: Grant execute permission for gradlew
        run: chmod +x ./gradlew
        shell: bash # (6).권한 부여

      - name: Build with Gradle
        run: ./gradlew clean build
        shell: bash # (7).build 시작

      - name: Get current time
        uses: 1466587594/get-current-time@v2
        id: current-time
        with:
          format: YYYY-MM-DDTHH-mm-ss 
          utcOffset: "+09:00"작 # (8).build 시점의 시간확보

      - name: Show Current Time
        run: echo "CurrentTime=${{steps.current-time.outputs.formattedTime}}" 
        shell: bash # (9).확보한 시간 보여주기
```

Github Action으로 빌드를 하기 위해서는 위의 코드들을 deploy.yml에 작성해주면 된다. 그럼 각각의 문단의 코드들이
어떠한 의미가 담긴것인지 보도록 하겠다.

<p align="center">
<img src="https://user-images.githubusercontent.com/59492312/151495800-119b0984-2334-454d-8c84-0996c28d9495.png">
</p>

name은 위의 사진처럼 프로젝트의 해당 코드를 push하면 해당하는 깃헙의 repository에서 메뉴 목록에 Actions를 클릭하게 되면 나오는
화면의 글자를 의미한다.

조금 더 쉽게 말하자면, Commit했을때 설명 + deploy.yml의 name + 내가 여태 Push한 workflow갯수 로 표시되는거다.
필자는 commit할때 github action build라고 commit message를 적었기에 이와같이 나온거다.

<br>

> 즉, 이 name은 Repo Action 탭에 나타나는 이름으로 실제 workflow 진행창에는 나오지 않는다. 그렇기에
> build나 배포에 영향을 주는것은 아니고 통상 필자처럼 프로젝트명으로 입력하거나 원하는 nmae을 입력하면 된다.  
> [name에 대한 내용](https://stalker5217.netlify.app/devops/github-action-aws-ci-cd-1/)

<br>

이제 나머지 #(1) 부터 #(9) 까지의 개념들에 대해 설명하도록 하겠다.

##### (1).브랜치 이름

```yml
on:
  push:
    branches:
      - master # (1).브랜치 이름
```

이 'master'가 들어가는 자리가 branch의 이름인데, 해당하는 branch 이름으로 push가 진행된다면
Github Action이 시작된다는 의미이다. 즉, Github Action 트리거 브랜치인것이다. master로 적어놓았지만,
다른 브랜치에서 Github Action을 실행시키고 싶다면 해당하는 branch 명을 바꾸어도 된다.

<br>

> 당연히, 지정한 브랜치외에 다른 명을 갖은 브랜치에서 push를 하게되고, 레포지토리에 pull request되어
> merge가 진행이 되었다고 해도, Github Action은 실행되지 않는다. 추가로 push외에 pull request나
> 다른 이벤트에 대해서도 트리거를 작동시키게 할 수 있다.      
> [push외에 다른 이벤트에 대한 Github Action 작동](https://hwasurr.io/git-github/github-actions/)
> [pull request에 대한 트리거 설정 코드](https://stalker5217.netlify.app/devops/github-action-aws-ci-cd-1/)

<br>

##### (2).수동 실행
```yml
on:
  workflow_dispatch: # (2).수동 실행
```

위의 push이벤트외에도 깃헙액션을 수동으로 실행하는것도 가능하게 하는 옵션이다.


##### (3).OS 환경

```yml
jobs:
  build:
    runs-on: ubuntu-latest # (3).OS환경
```

해당 Github Action 스크립트(여기서는 jobs의 steps들을 의미한다. 조금 더 쉽게 말하자면, (3)~(9)까지의
단계들을 말하며 해당 workflow를 말한다. job,step,workflow에 대해 잘 모른다면 만 아래의 '추가로'를 참고하자.)
가 작동될 OS 환경 지정하는것이다.

<br>

> Github Action의 workflow는 Runner라고 하는 Github Action Runner 어플리케이션이 설치된 인스턴스 서버에서
> 실행이 된다. 그 서버의 OS 환경을 지정하는것이다. 

<br>

일반적으로, 웹 서비스의 OS는 우분투보다는 센토를 많이 쓰는데, Github Action에서는 공식지원하는 OS목록에는
Centos가 없기에 우분투를 대신 사용하도록 한다.

<br>

> 우분투를 선택한다고 해도, 우리가 하려는 CI/CD에는 영향을 주지않으니 걱정하지 않아도 된다.

<br>

[OS목록](https://docs.github.com/en/actions/using-workflows/workflow-syntax-for-github-actions#jobsjob_idruns-on)을 본다면
우분투의 버전들이 나와있다. 현재 Ubuntu-latest는 Ubuntu 20.04와 같다고 나와있다.(필자가 글을 작성할 당시의 버전이니 조금 달라져 있을수도 있다.)
ubuntu-latest로 지정하거나 ubuntu-20.04로 적어주면 된다.

##### (4).코드 check out

```yml
jobs:
  build:
    steps:
      - name: Checkout
        uses: actions/checkout@v2 # (4).코드 check out
```

말 그대로, 코드 check out인데, 이 check out이라는 말은 개발용어에서 자주 등장하며 여러 의미로 쓰이니
간단하게 짚고 넘어가도록 하겠다.   

1. 형상관리 툴인 깃같은곳에서 소스코드를 내려받을 때 쓰는 check out : 이 경우에는 말 그대로 해당 소스코드를
내려받는것을 의미한다. 여기서는 스프링부트 프로젝트 소스코드를 내려받는것이다.

2. git checkout <branch name>처럼 terminal과 같은 Cli 사용하는 경우 : 특정 브랜치 여기서는 branch name에
해당하는 브랜치를 사용하겠다는것을 명시적으로 보여주는 것이다.

> [check out의 두가지 의미](https://backlog.com/git-tutorial/kr/stepup/stepup2_3.html)

##### (5).자바 설치

```yml
jobs:
  build:
    steps:
      - name: Set up JDK 1.8
        uses: actions/setup-java@v1.4.3
        with:
          java-version: 1.8 # (5).자바 설치
```

Github Action이 실행될 OS 즉, Runner에 Java를 설치하는 것이다. with: java-version: 1.8은 내가 다운받을
자바의 버전을 의미하는데, 자바11을 원한다면 자바11에 맞는 코드를 적어주면 된다. 필자는 자바8로 설치를 할것이다. 실제로도 가장
많이 쓰이고, 상용화가 잘 되있는 자바8을 많이 사용한다.

##### (6).권한 부여
```yml
jobs:
  build:
    steps:
      - name: Grant execute permission for gradlew
        run: chmod +x ./gradlew
        shell: bash # (6).권한 부여
```

다음 step인 gradle wrapper를 실행하여 build할 수 있도록 실행 권한 (+x)을 부여해주는 것이다.
실제로 터미널에서 ./gradlew build 명령어로 빌드를 실행하는데, Permission Denied가 되기 때문에 그 전에
chmod +x gradlew  를 터미널에 입력 하여 권한을 미리 부여해주는 것이다.

<br>

> run은 Runner에서 명령어를 실행하라는 의미이다.    
> [./gradlew build를 위한 권한 부여](https://javalism.tistory.com/101)    

<br>

**bash가 나왔으니 bash와 그 외에 반드시 알아야 할 기본용어에 대해 간단하게 한번 훑고 지나가도록 하겠다.** 그 다음 build에 대한 내용을 바로 보고싶다면 다음 문단으로
넘어가면 된다.

CLI, 터미널, 커맨드 프롬프트, 쉘, Bash, Git Bash, shell script, vim에 대해 알아보겠다.

(1). CLI는 Command Line Interface로 말 그대로 명령 줄 인터페이스로 터미널이나 커맨드 프롬프트처럼 한 줄의 명령어로
사용자와 컴퓨터가 상호 작용하는 '방식'을 의미한다.

(2). 터미널과 커맨드 프롬프트는 CLI를 가능하게 하는것으로 맥에서는 티미널이 쓰이고, 윈도우에서는 커맨드 프롬프트가 쓰이는것이다.

(3). 쉘은 터미널을 사용하기 위한 소프트웨어 환경을 뜻한다. 즉, 운영체제 상에서 사용자가 입력하는 명령을 읽고 해석하여
실행해주는 것으로, 사용자와 컴퓨터 사이에 소통하기위한 언어로 볼 수 있다.

(4). bash는 쉘의 한 종류이다. 쉘은 언어라 했는데, 언어에도 종류가 있듯이 bash가 linux에서 사용하는 쉘의 한 종류이며,
이 bash는 linux에서 가장 많이 사용되며 기본 쉘이다. 조금 더 이해하기 쉽게 예를 들자면, cd, ls, rm 등과같은것이 모두 bash인것이다.

(5). Git bash란, 윈도우에서는 커맨드 프롬프트라는 CLI를 사용하는데, bash를 사용하려면 git bash가 있어야 하는것이다.
윈도우에서 git bash를 다운받으면 깃과 bash 둘다 쓸 수 있다.
Bash는 CLI(Command Line Interface)이다. 리눅스와 맥에서는 기본적으로 Bash가 쓰인다.

(6). 쉘 스크립트는 쉘에서 사용할 수 있는 명령어들의 조합을 모아놓은 파일이다. 한줄씩 순차적으로 읽으면서 명령어들을 실행시켜주는 인터프리터 방식의 프로그램
이다.

(7). 마지막으로 vim은 텍스트 편집기이다. cd, ls와 같은 bash 명령어 외에, vim hi.text처럼 텍스트를 만들거나 수정 등 텍스트
편집기용 명령어이다.

<br>

> [git bash의 개념](https://2srin.tistory.com/117)    
> [CLI의 개념](https://ko.wikipedia.org/wiki/%EB%AA%85%EB%A0%B9_%EC%A4%84_%EC%9D%B8%ED%84%B0%ED%8E%98%EC%9D%B4%EC%8A%A4)    
> [쉘과 bash의 개념 (1)](https://dinfree.com/lecture/core/101_basic_3.html)   
> [쉘과 bash의 개념 (2)](https://minkwon4.tistory.com/159)      
> [shell script의 개념](https://minkwon4.tistory.com/159)    
> [vim의 개념](https://ko.wikipedia.org/wiki/Vim)    

<br>

##### (7).build 시작
```yml
jobs:
  build:
    steps:
      - name: Build with Gradle
        run: ./gradlew clean build
        shell: bash # (7).build 시작    
```

마지막 build의 시작이다. ./gradlew clean build 명령어를 실행하여 빌드를 시작한다. 보이는 바와 같이 shell: bash는 bash를 사용하여서
build를 시작하라는 의미이다. github action으로 빌드하기는 여기까지이다.

##### (8).build 시점의 시간 확보

(7)까지만 해도 build는 되는데, 왜 (8),(9) 코드들이 있는걸까?
용도는 두가지이다.    
첫번째 : 빌드되는 시점의 시간을 보기위함    
두번째 : 후에 배포시킬때 version label이란것에 빌드된 시간을 넣어서 어플리케이션 버전으로 사용하기 위함이다.
무슨말인지 모르겠다면, 아래 차근차근 설명하니 따라보면 충분히 이해가 될 것이다.

```yml
jobs:
  build:
    steps:
      - name: Get current time
        uses: 1466587594/get-current-time@v2
        id: current-time
        with:
          format: YYYY-MM-DDTHH-mm-ss # (1)
          utcOffset: "+09:00"작 # (8).build 시점의 시간확보
```

1466587594/get-current-time이라는 것을 사용하여(runner에 있는) 빌드시점의 시간을 확보해 놓는것이다.
format은 YYYY-MM-DDTHH-mm-ss로 해놓아서 확보해놓을 시간의 형태를 정해놓고, utcOffset까지 설정해놓아서
한국 시간으로 맞춰서 확보해놓는것이다.(UTC 기준이기 때문에 한국시간에 맞추려면 +09:00을 해주어야 한다.)

그렇다면, 이렇게 확보해놓은 빌드시점의 시간은 어떻게 쓰일까 ?

##### (9).확보한 시간 보여주기
```yml
jobs:
  build:
    steps:
      - name: Show Current Time
        run: echo "CurrentTime=${{steps.current-time.outputs.formattedTime}}" # (2)
        shell: bash # (9).확보한 시간 보여주기
```

다음 (9)를 보자. echo ~ }}"라고 쓰여진 코드 부분은 일종의 화면에 출력하는것이다. 즉, 내가 프로젝트를 push를 해서
github action으로 build가 완료가 됬을때 확보한 시간을 github action 빌드 상태창에 시간이 어떻게 되는지 보여주겠다는
의미이다.

조금 이해가 안갈 수 있다. 아래 사진을 보도록 하자.

<p align="center">
<img src="https://user-images.githubusercontent.com/59492312/151648743-ae6c4a39-f124-4be6-9df6-0f649e9c15c8.png">
</p>

잠시 후에 Push를 하고 깃헙 레포지토리에 Action탭을 클릭하면 실시간으로 push한 프로젝트가 build가 잘 되고 있는지 확인할 수 있는 창이다.
이곳에 Show Current time에 CurrentTime=2022-01-28T14-47-53가 잘 출력된것을 볼 수 있다.
(직접 해보면 더 빠르게 이해가 될 것이다. 곧 직접해보도록 하겠다.)

<br>

> 그렇다면, 앞서 말한 Version Label에 사용되어서 배포한 어플리케이션의 버전 식별용으로 사용된다는 것은 언제쓰는건가요 ?
> 라고 물어볼 수 있다. 첫 (8)에서 확보한 시간을 바탕으로 다음 장에 배포를 진행할때, 그대로 Version Label에 적용하여 식별용으로
> 어떻게 사용되는지 보여줄 것이다. 다음 글 (3) 배포하기로 넘어가서 설명하도록 하겠다.

<br>

##### 👨‍💻 추가적인 얘기

위에서 name은 단순히 github action의 workflow에서 단계별 명칭을 보여주는 역활만 하기 때문에
본인이 알아보기에 맞게 새롭게 작성해도 된다.

아래는 그 예시이다.    
필자는 위의 name에서 추가로 의미없는 숫자를 덧붙여 보았다.
<p align="center">
<img src="https://user-images.githubusercontent.com/59492312/151544083-af4a11b6-7388-4b97-b135-6dbba56250f4.png">
</p>

<br>

> 깃헙 레포지토리에서 직접 Github Action을 실행시켜보고 확인해보면 더 정확하게 이해가 될 것이다.

<br>

#### 🪁 References
* 참조링크 : [deploy.yml에 관한 전반적인 내용](https://jojoldu.tistory.com/543)

<br>



### 3.직접 PUsh를 해서 build를 진행해보도록 하겠다.

<p align="center">
<img src="https://user-images.githubusercontent.com/59492312/151648904-65022ea0-b707-4377-a6ca-e98c08a872d8.png">
</p>

로컬 저장소에서(IntelliJ 등 그 외) 원격 저장소인 깃헙 레포지토리로 PUSH를 하게되면, github action이 실행된다.
위와같이 빌드가 잘되면 모두 초록색으로 체크가 된다.

<br>

> 해당 화면을 보려면 내가 push한 레포지토리로 들어가서 위와같이 Actions탭을 누르면 된다.

<br>

#### 🪁 References
* 참조링크 : [deploy.yml에 관한 전반적인 내용](https://jojoldu.tistory.com/543)

<br>




### 4.(push가 실패하는 분들만 보면 된다.)

<p align="center">
<img src="https://user-images.githubusercontent.com/59492312/151649056-26134ea1-734c-4aff-ad41-c28eeac75d54.png">
</p>

기본적으로 로컬에서 원격 깃헙으로 push를 하거나 아니면 workflows를 작동시키려면 깃헙 계정의 settings로 들어가서(레포지토리의 setting이 아니다.)
Personal access tokens에서 토큰을 발급받아서 권한을 부여해주어야 가능하다.

<p align="center">
<img src="https://user-images.githubusercontent.com/59492312/151649059-44daf835-0c0b-4a9e-b194-eaffa5729b5b.png">
</p>

위 화면처럼, 보통 우리가 commit하고 push할때와는 다르게 workflow를 선택해주고 토큰을 발급받아서 사용해야 권한이 부여되고 정상적으로 push가 되는것이다.
만약, access token에 대해 아직 잘 모르겠다면, 깃헙 access token으로 원격 저장소 push하기 글들을 찾아본다면 확실하게 지금의 내용이 이해가 될것이다.

<br>



### 🚀 추가로

깃헙액션의 코어 개념에 대해 간단하게 정리하도록 하겠다.    

**Workflow** : 자동화된 전체 프로세스로(즉, 위에서는 jobs에 있는것들로, (3)부터 (9)까지가 하나의 workflow인것이다.). 하나 이상의 Job으로 구성되고, Event에 의해 예약되거나 트리거될 수 있는 자동화된 절차를 말한다. Workflow 파일은 YAML으로 작성되고, Github Repository의 .github/workflows 폴더 아래에 저장된다. Github에게 YAML 파일로 정의한 자동화 동작을 전달하면, Github Actions는 해당 파일을 기반으로 그대로 실행시킨다.    

**Event** : Workflow를 트리거(실행)하는 특정 활동이나 규칙. 예를 들어, 누군가가 커밋을 리포지토리에 푸시하거나 풀 요청이 생성 될 때 GitHub에서 활동이 시작될 수 있다.   

**Job** : Job은 여러 Step으로 구성되고, 단일 가상 환경에서 실행된다. 

**Step** : Job 안에서 순차적으로 실행되는 프로세스 단위.   

**Runner** : Gitbub Action Runner 어플리케이션이 설치된 일종의 서로, Workflow가 실행될 인스턴스이다.      

#### 🪁 References
* 참조링크 : [깃헙액션의 코어개념](https://velog.io/@ggong/Github-Action%EC%97%90-%EB%8C%80%ED%95%9C-%EC%86%8C%EA%B0%9C%EC%99%80-%EC%82%AC%EC%9A%A9%EB%B2%95)

<br>



태그 : #Github Action, #Benastalk, #깃헙액션 코어개념, #extra-icons 플러그인, #deploy.yml, #깃헙액션 코어개념, #workflow, #runner, #job, #event, #step, #check out, #check out의 두가지 의미, #bash, #shell, #shell script, #cli, #커맨드 프롬프트, #git bash, #vim
