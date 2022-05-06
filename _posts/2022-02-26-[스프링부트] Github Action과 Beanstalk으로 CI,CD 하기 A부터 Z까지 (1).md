---
layout: post
title:  "[스프링부트] Github Action과 Beanstalk으로 CI/CD 하기 A부터 Z까지 (1)"
categories: [ 빌드와 배포 ]
image: https://user-images.githubusercontent.com/59492312/159110114-45fd9f9c-9bf1-4c7f-a354-329b34cdebbc.png
---

* CI/CD의 개념
* CI/CD의 구성요소와 그에 관한 개념
* CI/CD 순서에 대한 내용과 Pipeline 개념

* * *

<br>

### 1.우선은 CI/CD의 개념과 추가로 알아야 할 용어들에 대해 정리하고 가도록 하겠다.(바로 CI/CD 실무로 넘어가고 싶다면 글(2)로 넘어가면 된다.)

<p align="center">
<img src="https://user-images.githubusercontent.com/59492312/150467453-6427d3f0-5933-4adb-b99d-0710a458bf77.png">
</p>

우리는 CI(Continuous Integration)/CD(Continuous Deployment)의 각 과정들과 개념 대해 알아보기에 앞서,
이를 이루고 있는 일련의 과정들에 대해 먼저 보도록 하겠다.   

그림에서 보듯이, CI(지속적통합)는 Build,Test,Merge로 이루어져있다.   
CD(지속적배포)는 Continuouse Delivery(지속적인 서비스 제공) + Continuous Deployment(지속적인 배포)가 합쳐진
용어이다.

#### 🪁 References
* 참조링크 : [CI/CD 개념 그리고 흐름 (1)](https://abbo.tistory.com/225)
* 참조링크 : [CI/CD 개념 그리고 흐름 (2)](https://artist-developer.tistory.com/24)

<br>



### 2.CI의 Build, Test, Merge는 무엇이고 그렇다면 컴파일은 어떻게 연동되는걸까 ?

<p align="center">
<img src="https://user-images.githubusercontent.com/59492312/151288599-0dc84d6a-1e09-4a94-97ba-2a6e0b9f7cbb.png">
</p>

#### (1).Build
우선, Build는 소스 코드 파일을 컴퓨터에서 실행할 수 있는 독립적인 형태로 변환하는 과정과 그 결과를 말한다.
쉽게 말하자면, Java프로젝트를 진행한다면 개발자가 작성한 A.java와 여러가지 정적 파일등에 해당하는 resource가 존재한다.
빌드를 한다면 소스코드(A.java)를 컴파일해서 .class로 변환하고 resource를 .class에서 참조할 수 있는 적절한 위치로 옮기고 
META-INF와 MANIFEST.MF 들을 하나로 압축하는 과정을 의미한다. 즉, 컴파일을 빌드의 부분집합이라고 생각하면 된다.
또한, 빌드 과정을 도와주는 도구를 Build Tool이라고 한다. 즉, 컴파일 된 코드를 실제 실행할 수 있는 상태로 만드는 
일을 Build 라는 개념으로 생각하면 된다.

> 스프링부트 프로젝트에서 쓰는 Gradle도 Build Tool의 일종이다.

<br>

<p align="center">
<img src="https://user-images.githubusercontent.com/59492312/151288671-4483d871-64ac-4f4f-be8d-cb9d06189b8d.png">
</p>

#### (2).Test,Merge
그 다음은 Test와 Merge인데, test는 말 그대로 단위테스트나 종합테스트 그 외에 해당 코드가 잘되는지
테스트하는것을 말한다. 또한 Merge는 코드병합으로 깃과 깃헙을 이용하 서로 다른 branch에서 commit 한것을 pull request를
받아들임으로써 코드를 병합하는것이다.(merge에 대해 헷갈린다면, 깃과 깃헙을 공부하고 오자.)

<br>

##### 👨‍💻 추가적인 얘기
그럼 CI에 대해 조금 더 풀어 말하자면, 어플리케이션의 새로운 코드 변경 사항이 정기적으로 빌드되고
테스트 되어 공유 레포지토리에 병합되는것을 얘기 한다. 즉, 이를 수동이 아닌 자동으로 지속적으로 쉽게 해주는 환경(CI)을 말한다.
(CI의 단계와 순서에 대해서는 글 후반부에 추가적으로 얘기하겠다.)

> 여기서 우리가 사용하는 Git은 형상관리(=구성관리)툴로, 버전이나 변경사항을 체계적으로 추적하고 통제하는것을 가능하게 해준다.

<br>

#### 🪁 References
* 참조링크 : [빌드의 개념](https://choseongho93.tistory.com/296)
* 참조링크 : [CI의 전반적인 개념](https://artist-developer.tistory.com/24)
* 참조링크 : [형상관리,구성관리 개념](https://ko.wikipedia.org/wiki/%EA%B5%AC%EC%84%B1_%EA%B4%80%EB%A6%AC)

<br>



### 3.그렇다면, 이번에는 CD의 Continuous Delivery와 Continuous Deployment에 대해 보도록 하겠다.

<p align="center">
<img src="https://user-images.githubusercontent.com/59492312/151291909-4045e805-9d64-41b1-b809-f74ba7c8e145.png">
</p>

CD는 Continuous Delivery 혹은 Continuous Depolyment 두 용어 모두의 축약어이다.   
Continuous Delivery는 지속적 제공으로, CI를 통해서 새로운 소스코드의 빌드와 테스트 병합까지 
성공적으로 진행되었다면, 빌드와 테스트를 거쳐 github과 같은 저장소에 업로드하는 것을 의미한다.   
Continuous Deployment는 지속적 배포로, 이렇게 지속적 제공으로 인해 성공적으로 병합된 내역을 저장소뿐만이 아니라 클라이언트가 사용할 수 있는 환경까지 배포하는것을 의미한다.
예를들면, EC2같은곳을 말한다.       
  
#### 🪁 References
* 참조링크 : [CD의 두 개념 (1)](https://abbo.tistory.com/225)   
* 참조링크 : [CD의 두 개념 (2)](https://ggn0.tistory.com/118)   

<br>



### 정리 및 추가내용

<p align="center">
<img src="https://user-images.githubusercontent.com/59492312/151293092-275d1aa6-cf5f-4289-99f9-c3216fe0f200.png">
</p>

#### 1.CI와 CD의 순서에 대해서 정리

위에서 통상적인 CI와 CD의 순서는 CI의 build, test, merge가 끝나면 공유 레포지터리로 코드가 공유되고(Continuous Delivery) 
그 이후에 실제 서비스 서버에 배포(Continuous Deployment)하는 것을 CI/CD의 통상적인 순서라고 보았다.

첫번째 CI/CD순서    
A팀에 개발자가 5명 존재한다. 이 팀은 각각, 코드 작업을 완료하여 github에 push한다.
그리고 모인 코드는 병합을 거치게 된다. 그런데 병합 과정에서 동일한 코드를 수정한 경우 충돌이 발생하게 된다.
발생한 충돌을 해결하기 위해 수동으로 작업을 진행한다. 그 다음 배포를 해야하는데, 배포 하기에 앞서 코드에 문제가 없는 지 테스트를 진행한다.
그런데 코드에 치명적인 버그가 발생한다. 이 버그를 해결하기 위해 이 코드와 관련된 팀원이 일을 진행한다.

이처럼 A팀은 코드 푸시 -> 병합 -> 충돌 발생시 해결 -> 기능 테스트 작업을 매번 거친다.

두번째 CI/CD순서    
개발자 구현한 코드를 기존 코드와 병합한다.(master branch가 아닌 다른 branch에서 했다고 봐도되고, 아니면 원래부터
master branch로 push하여 병합하는거로 볼 수도 있다.) 병합된 코드가 올바르게 동작하고 빌드되는지 검증한다.
테스트 결과 문제가 있다면, 수정하고 다시 다시 코드를 작성하여 병합하고 그게 아니라면 배포를 진행 한다.

세번째 CI/CD순서    
CI를 진행할때, build를 하면서 중간에 test를 하는 경우도 있다. 또한, 실제 github action에서 
build할때 test를 진행하면서 build를 하게된다.(Action칸에 보면, build 프로세스를 눌러오면 알 수 있다.)

정리하자면, 
CI/CD 파이프라인은 개발 플로우마다 조금씩 다르거나 추가될 수 있는것이다. 깃헙액션과 빈스톡(다음장에서
본격적으로 진행할 CI/CD는 깃헙액션과 빈스톡으로 사용하여 진행할 것이다.)을 사용하는 경우에 혼자 개발하는 경우는
원격 레포지토리에 push를하고 merge한다음에 build하는 과정에서 test를 진행한다. build가 무사히 진행된다면
그 이후에 실제 서버 환경에 배포를 진행하게 된다.(원격 레포지토리에 push를 하기전에 단위테스트 등을 진행할 수도 있다.)
그게 아니면 여러 개발자가 branch를 만들고, 로컬 저장소에서 코드작성과 테스트까지 진행한 후에 원격 레포지토리에 push를
하게하고(master 브랜치가 push 했을때만 build되게 설) 이를 merge하여, master 브랜치에서 최종 test를 하고 다시 정
원격 레포지토리에 push하여 test를 하며 build를 진행할 수도 있다. 즉, 이 CI/CD의 과정은 개발 플로우에 따라 얼마든지 달라질 수 있다는것을 알고가자.

> [첫번째 CI/CD순서](https://abbo.tistory.com/225)   
> [두번째 CI/CD순서](https://ggn0.tistory.com/118)    
> 세번째 CI/CD순서 - aws 혼자구현하기 책 298pg    

<br>

#### 2.Pipeline(파이프라인)의 개념

<p align="center">
<img src="https://user-images.githubusercontent.com/59492312/151294755-efebeccc-27c4-407a-a846-8ad1d89c7515.png">
</p>

여기서 파이프라인의 개념은 코드를 컴파일(Compile), 빌드(Build) 그리고 배포(Deploy) 하게 해주는 자동화된 프로세스들의 묶음(set)을
이야기 한다. CI만 있는 프로세스들의 묶음을 CI파이프라인이라고도 하며, CD들만 있는 프로세스들의 묶음을 CD파이프라인이라고도 한다.
통상, CI/CD의 전 단계를 CI/CD파이프라인이라 부른다. AWS에는 CodePipeline이라고 파이프라인을 자동화하여 완전관리형 지속전달 서비스도 있다고 한다.

<br>

> [Pipeline의 개념](https://linux.systemv.pe.kr/%EC%86%8C%ED%94%84%ED%8A%B8%EC%9B%A8%EC%96%B4-%EC%97%94%EC%A7%80%EB%8B%88%EC%96%B4%EB%A7%81%EC%97%90%EC%84%9C-%ED%8C%8C%EC%9D%B4%ED%94%84%EB%9D%BC%EC%9D%B8pipeline%EC%9D%80-%EB%AC%B4%EC%97%87/)   
> [CI와 CD파이프라인](https://ichi.pro/ko/circleci-dae-gitlab-olbaleun-ci-cd-dogu-seontaeg-273919299873289)   
> [CI와 CD파이프라인](https://www.redhat.com/ko/topics/devops/what-cicd-pipeline)   
> [AWS CodePipeline](https://aws.amazon.com/ko/codepipeline/)    

<br>



### 🚀 추가로
본격적인 실습은 (2)글부터 진행하도록 하겠다.

<br>

태그 : #형상관리 #구상관리 #CI/CD #build #test #merge #Pipeline
