---
layout: post
title:  "[스프링부트] devtools 적용, a부터 z까지 완벽정리(+ OS별 차이)"
categories: [ DevTools ]
image: https://user-images.githubusercontent.com/59492312/161212879-0363726a-d315-403b-ad65-feb4d24a4f84.png
description: "[스프링부트] devtools 적용, a부터 z까지 완벽정리(+ OS별 차이)"
featured: true
---

* springboot devtools 사용을 위한 의존성 설정과 기타설정
* 기타 오류들 및 OS별, 프로젝트별 에러사항 정리

> 모든 코드는 [깃헙](https://github.com/sooolog/dev-spring-springboot)에 작성되어 있습니다.

* * * 

### 1.우선 빠르게 SpringBoot devtools 의존성을 추가해 보도록 하겠다.

<p align="center">
<img src="https://user-images.githubusercontent.com/59492312/144983726-50030525-fe6d-4a09-a005-cad5e405ca2c.png">
</p>

<p align="center">
<img src="https://user-images.githubusercontent.com/59492312/144983711-87ff0dce-accd-4a02-aa03-cc7d661f6437.png">
</p>

의존성을 추가해주는데,위 첫번째 코드처럼 runtimeOnly로 해주어도 되고 아니면 developmentOnly와 위에 configurations { ~ } 를 함께 적어주어서 의존성을 추가해주어도 된다.

### 2.그 다음은 인텔리제이에서 기타 설정들을 해보겠다.

<p align="center">
<img src="https://user-images.githubusercontent.com/59492312/144983736-2eb88366-c70b-4a5b-8ae0-4ad6102d1ab7.png">
</p>

인텔리제이에서 Preferences를 들어가서 Compiler란에 들어간다. 그리고 Build project automatically를 클릭 ! (난 미리 체크를 해둔 상태다.) 그리고 OK를 클릭하고 나온다.

> 맥북은 Preferences라 뜨고, 윈도우에서는 Settings로 들어가면 된다.

<p align="center">
<img src="https://user-images.githubusercontent.com/59492312/144986984-f938ca88-0018-4013-bb80-40dc4196324a.png">
</p>

그 다음은, 맥북은 command + shift + a 를 눌러서 해당 창이 뜨게되면 Registry...를 클릭한다.

> 윈도우는 Ctrl + shift + a 를 클릭하면 해당 창이 뜬다.

<p align="center">
<img src="https://user-images.githubusercontent.com/59492312/144983876-497a5d06-51a4-4f2c-a12c-486df86ae8ac.png">
</p>

그러면, 이런 창이 나오는데, 당황하지 말고 compiler.automake.allow.when.app.running이라는것을 찾아서 체크해준다. 필자는 이미 체크를 해둔상태라 맨위에 뜨게 되는것이다.

<p align="center">
<img src="https://user-images.githubusercontent.com/59492312/144983902-1c5e63c8-7144-44bc-9d44-2d2b00381a7b.png">
</p>

마지막 설정이다. 맥북과 윈도우 동일하게 Run을 클릭하고 Edit Configurations...를 클릭해보자.

<p align="center">
<img src="https://user-images.githubusercontent.com/59492312/144987265-64eaae70-8261-42bf-a28e-60c3455af3a6.png">
</p>

그러면, 새로운 창이 하나 뜨는데, 여기서 
On 'Update' actions:    
On frame deactivation:   
의 값을 모두 **Update classes and resources**로 바꾸어 주어야 한다.

이 마지막 설정이 되야 제대로 실시간 업데이트가 되며, classes는 클래스파일들에 대한 업데이트 resources는 정적파일에 대한 업데이트이다. 즉, Update resources와 같은것도 있는데 이것을 클릭하고 저장하면 class파일이 변경되어도 devtools가 정상적으로 작동하지 않는다. 주의 바란다.

> 가끔씩 해당 창에서 On 'Update' actions: 칸이 없는 경우가 있다. 이런경우는 그레이들 프로젝트에 스프링부트 프로젝트를 만들었기에 보이지 않는것이다. 처음에 Springinitializr를 통해 스프링부트 프로젝트를 만들어주어야 해당 칸이 보인다.

#### 마지막 설정. LiveReload 확장프로그램 추가

🎈 필자는 크롬에서 바로 리로드가 되는지 확인할것인데, 만약 새로고침없이 프로젝트가 리로드 되자마자 브라우저도 자동 새로고침이 되고싶게하고싶다면 크롬 확장프로그램인 Enable liveReload를 추가해주기만 하면된다.

### 3. 코드 작성

```java
@Controller
public class devToolsController {

    @GetMapping("/devtoolstest")
    public String devToolsTest(Model model) throws Exception{
        model.addAttribute("test1","테스트전1");
        model.addAttribute("test2","테스트전2");
        model.addAttribute("test3","테스트전3");
        return "devToolsTest/test";
    }
}
```
Controller하나를 이와같이 작성해준다.

~~~html
<!doctype html>
<html lang=ko">
<head>
    <meta charset="UTF-8">
    <meta name="viewport"
          content="width=device-width, user-scalable=no, initial-scale=1.0, maximum-scale=1.0, minimum-scale=1.0">
    <meta http-equiv="X-UA-Compatible" content="ie=edge">
    <title>Document</title>
</head>
<body>
<div>스프링부트 devtools test 후</div>
    <p>{{test1}}</p>
    <p>{{test2}}</p>
    <p>{{test3}}</p>
</body>
</html>
~~~

필자는 mustache를 사용하기에 머스테치 템플릿을 사용했다.

### 4. 결과 보기(브라우저)

<p align="center">
<img src="https://user-images.githubusercontent.com/59492312/144987735-060f23b8-a301-4916-ad7b-7e34d7e3c4b9.png">
</p>

코드 수정 전이며

<p align="center">
<img src="https://user-images.githubusercontent.com/59492312/144987747-91f1e181-47ee-4e6b-a753-abc6beebfa29.png">
</p>

코드를 수정하면 바로 반영되어 브라우저 화면에 이렇게 뜬다.
수정한 코드는

```java
    @GetMapping("/devtoolstest")
    public String devToolsTest(Model model) throws Exception{
        model.addAttribute("test1","테스트후1");
        model.addAttribute("test2","테스트후2");
        model.addAttribute("test3","테스트후3");
        return "devToolsTest/test";
    }
```
컨트롤러와
```html
<div>스프링부트 devtools test 후</div>
    <p>{{test1}}</p>
    <p>{{test2}}</p>
    <p>{{test3}}</p>
```
이와 같이 수정하였다.

> 맥북에서는 인텔리제이 말고 다른 창을 클릭해야만 바로 반영이 됬고, 윈도우에서는 인텔리제이 내의 다른 클래스파일이나 공간을 클릭해도 반영이 됬다. 그러나 반응속도면에서 맥북이 월등히 빨랐고, 윈도우에서는 차라리 안하는게 낫다싶을정도로 자동 리로드 반응속도가 느렸다. 참고하도록 하자.

<br>

🚀 추가로
1. 이 devtools의 원래기능은 5가지이나 우리가 본것은 Property Defaults, Automatic Restart, live Reload이다. 템플릿의 캐쉬 설정이나 live Reload등을 사용하겠다 하는 모든 설정은 devtools 의존성을 추가함으로써 자동 설정되는 부분이니 건드리지 않아도된다.
2. Devtools의 작동원리의 개념에 대한것을 알고싶다면 아래 참조링크를 보도록 하자.   
[[SpringBoot Devtools 원리]](https://iksflow.tistory.com/57))


## 🪁 Reference
* [스프링부트 Devtools 설정1](https://velog.io/@bread_dd/Spring-Boot-Devtools)      
* [스프링부트 Devtools 설정2](https://otrodevym.tistory.com/entry/spring-boot-설정하기-10-dev-tools-설정-및-테스트-소스)      
* [스프링부트 Devtools 설정3](https://otrodevym.tistory.com/entry/spring-boot-설정하기-10-dev-tools-설정-및-테스트-소스)      
* [스프링부트 시작하기 책](https://velog.io/@sooolog)
