![우당타타탙](https://user-images.githubusercontent.com/82383294/197936303-8d05268f-621e-4160-bc42-8f83149d8ebf.png)

<a href = "https://velog.io/@noyohanx/Ops-CICD-%EB%9E%80">저번 글</a>에 이어서 AWS의 코드 시리즈로 CI/CD를 어떻게 하는지 실습을 진행해보려고 합니다.

(이 글은 AWS의 기본적인 지식을 가졌다는 전제하에 글이 진행되니 읽기 전 참고해주세요)
### 목차
- 준비물
- 애플리케이션 세팅
- EC2
- Code Series
    - Code commit
    - Code build
    - Code pipeline

## 준비물
- AWS 계정 : EC2,CodeSeries 등을 사용할 예정
- 끈기와 노력
- CI/CD를 진행할 애플리케이션

## EC2
![](https://velog.velcdn.com/images/noyohanx/post/99976228-71cc-4c3c-b2a0-1b8865334328/image.png)
그냥 별다른 설정 없이 EC2를 시작해줍니다. 용량은 t2.micro로 프리티어가 제공되는 용량을 사용할 거예요.
![](https://velog.velcdn.com/images/noyohanx/post/414a4eda-667c-43d2-a6a8-dfd3eab67810/image.png)

보안그룹은 22번(ssh 접속) 포트와 3000번(노드 애플리케이션) 포트를 열어줍니다.
이후 아래의 명령어를 터미널에 입력해준다면 서버에 성공적으로 접속할 수 있을 거예요.
```
$ chmod 400 'pemkey.pem'
$ ssh -i "pemkey.pem" ec2-user@ec2-'public-ip'.ap-northeast-2.compute.amazonaws.com
```

![](https://velog.velcdn.com/images/noyohanx/post/da105611-de91-43e6-a385-40dd43ef21ba/image.png)

## 애플리케이션 세팅
최근에 진행한 프로젝트에서 사용한 스택인 NestJs를 사용해 간단한 서버 애플리케이션을 구축해보도록 하겠습니다.
NestJs 설치 및 세팅에 관한 내용은 <a href = "https://wikidocs.net/148194">여기</a>를 참고해주시면 좋을 것 같아요.
위의 과정을 진행하기 귀찮다면 필자의 깃허브에서 클론을 받아오시면 됩니다.

```
$ yum install git
$ https://github.com/NohGaSeong/CICD-Practice.git
// 아래의 과정을 통해 nodejs 14를 설치해줍니다.
$ curl -o- https://raw.githubusercontent.com/nvm-sh/nvm/v0.38.0/install.sh | bash
$ nvm install 14 
$ npm -v // npm의 버전을 체크합니다.
$ npm install -g yarn //yarn 을 설치합니다.
$ cd CICD-Practice 
$ yarn  // 필요한 패키지를 설치합니다.
$ yarn add pm2 -g //pm2 를 설치해줍니다.
$ yarn build // 프로젝트를 빌드합니다.
$ yarn pm2 start dist/main.js //빌드된 프로젝트를 실행합니다.
```
여기서 사용한 pm2라는 라이브러리를 잘 모르겠다면 <a href = "https://hellominchan.tistory.com/11	">여기</a>를 보고 오시면 좋아요.
![](https://velog.velcdn.com/images/noyohanx/post/e0a6e2ef-1d79-4080-9dc6-785b8c8a4dd0/image.png)
이런 식으로 서버에 잘 접속되면 애플리케이션 세팅이 끝났습니다. (스크린샷은 로컬호스트지만 이걸 ec2 서버 내에서 하면 됩니다!)

## CodeSeries
본격적으로 code series를 이용해서 ci/cd를 하기 전에 우선 이 서비스들이 어느 역할을 수행하는지 먼저 알아보도록 하겠습니다.

**Code Commit**
- 비유를 하자면 aws의 github라고 볼 수 있다.

**Code Build**
- 소스코드를 컴파일해주고 빌드해주는 서비스이다.

**Code Deploy**
- 배포하고자 하는 대상을 deployment group으로 배포시켜준다. 배포 가능 대상은 EC2, Lambda, ECS 등 다양하다.

**CodePipeline **
- source- build - deploy 등의 과정을 관리해주느 서비스이다. code series 말고도 github, jenkins 등 지원하는 다른 툴들도 혼용이 가능하다.

### 아키텍쳐
![](https://velog.velcdn.com/images/noyohanx/post/5ec45c03-5735-44b9-bce5-21e9f6941ccd/image.png)
구성하고자하는 아키텍처의 흐름은 다음과 같아요.
1. 유저가 깃허브에 코드를 푸쉬
2. codebuild를 통해 CI 진행
3. codeDeploy를 통해 CD 진행
4. 이후 변경사항 확인

이제 실제로 CI/CD를 구성하러 가봅시다.
### EC2 IAM Setting
CodeSeries를 사용해보기 앞서 미리 준비를 해야합니다.
ec2 우클릭 -> 보안 -> IAM 역할 수정을 눌러줍니다.
![](https://velog.velcdn.com/images/noyohanx/post/aace7900-e7c1-4b51-99c6-9bec41a0543a/image.png)
새 IAM 역할 생성버튼을 눌러주고,
![](https://velog.velcdn.com/images/noyohanx/post/aec99cdc-daf3-4f3f-b853-7de66f36d1c9/image.png)
아래의 스크린샷과 같은 것을 선택해주고 다음버튼을 누른 뒤
![](https://velog.velcdn.com/images/noyohanx/post/e864fce8-1420-4ee0-880f-04658e62ffb7/image.png)
`AWSCodeDeployFullAccess`, `AmazonS3FullAccess` 권한 정책을 넣어주고 생성.(codedeploy에서 ec2 접근과 s3를 사용하기 떄문) 이후 ec2 와 연결해줍니다.
![](https://velog.velcdn.com/images/noyohanx/post/9b4a1f84-6b2d-4acc-8987-bb4d4651c25b/image.png)

그럼 EC2 IAM의 세팅은 끝났습니다.
### Codebuild
1. 빌드 프로젝트 생성
- 이름, 설명은 알잘딱 만들어줍니다.
- 소스 같은 경우 깃허브를 사용할건데 깃허브 -> setting -> token으로 가서 엑세스 토큰 발급 이후 연결해주고 리포지토리를 선택해주면됩니다.
- 소스 버전 같은 경우 리포지토리 커밋 옆에 보이는 `7b76666` 같은 영어 숫자가 섞인 텍스트를 클릭해주면 full commit id가 보인다. 그것을 복사, 붙여넣기.
![](https://velog.velcdn.com/images/noyohanx/post/ceb240e0-3b93-4298-a68c-cf254155e7eb/image.png)

2. 환경
아래의 스크린샷과 같이 해주면됩니다. 깊게 들어갈 것 없이 눈에 보이는대로라 딱히 설명할 부분은 없는 것 같습니다.
![](https://velog.velcdn.com/images/noyohanx/post/bfb1975e-74ab-41ab-a753-266f5ce3c79d/image.png)

3. BuildSpec
- 빌드를 할 때 이걸 참고해서 빌드를 해주세요 ~ 라는 뜻이다. 우리가 구성할 buildspec은 아래와 같습니다.
```
version: 0.2

phases:
  install:
    runtime-versions:
      nodejs: 16
    commands:
      - node -v
      - npm -v
  pre_build:
    commands:
      - echo Entered the install phase...
      - sudo yarn
  build:
    commands:
      - sudo yarn add pm2 -g
      - sudo yarn build
  post_build:
    commands:
      - sudo yarn pm2 kill
      - sudo yarn pm2 start dist/main.js --watch -f
 ```
- version : 말그대로 버전이다. 0.2 를 권장한다고합니다.
- phases.install.runtime-versions : 런타임버전. 14는 지원을 안하는 것 같아서 16을 사용했습니다.
- phases.install.commands : 빌드 전 설치할게 있으면 수행하는 명령어. 없기때문에 버전 확인만 해줬습니다.
- phases.pre_build.commands : 빌드 전 수행되는 명령어. yarn 으로 패키지를 다운 받아줬습니다.
- phases.build.commands: 빌드시 수행되는 명령어.
- phases.post_build.commands : 빌드 후 수행되는 명령어.

이후 생성버튼을 클릭합니다.

### CodeDeploy
#### 애플리케이션
손 쉽게 생성해줍시다.
![](https://velog.velcdn.com/images/noyohanx/post/fea63166-cdca-4943-b2da-14c11374e17a/image.png)
#### 배포 그룹 생성
배포 그룹의 이름을 생성, 서비스 역할은 알아서 생성해주는 역할을 사용하면됩니다.
![](https://velog.velcdn.com/images/noyohanx/post/50e2bf0c-974b-40bf-8d91-01a21cfd0d25/image.png)
배포 방법은 현재위치, 그리고 우리가 생성했던 ec2를 선택해줍니다.
![](https://velog.velcdn.com/images/noyohanx/post/30e78878-3ca9-477d-951a-8897e5199cb8/image.png)
배포 설정은 `CodedeployDefault.AllatOne`으로. (배포 구성을 만들어보고 싶으면 다른 곳들을 참고해서 만들어봐도 좋습니다.)  로드 밸런서는 지금 우리가 사용하고 있지 않기때문에 체크하지 않고 배포그룹을 생성해줍니다.
![](https://velog.velcdn.com/images/noyohanx/post/e89897ec-8224-4dbe-bb2d-70f673e9394d/image.png)

#### CodeDeployAgent 설치
codedeploy가 CD를 진행하기 위해서 ec2에 codedeployAgent를 설치해줘야합니다. 설치 방법은 아래와같습니다.

```
$ yum -y update
$ yum install -y ruby //루비 설치
$ wget https://aws-codedeploy-ap-northeast-2.s3.ap-northeast-2.amazonaws.com/latest/install //리전에 서울이라면 이렇게 다운.
$ chmod +x ./install
$ sudo ./install auto
$ sudo service codedeploy-agent status //상태확인
$sudo service codedeploy-agent start //서비스시작
```
`service codedeploy-agent status` 를 입력했을때 `The AWS CodeDeploy agent is running as PID 14890` 가 뜨면 성공적으로 실행된 것입니다.

#### appspec.yml
필자의 깃허브를 클론 받았으면 있을텐데 cd 를 어떤식으로 진행해주세요 ~ 를 적어두는 파일입니다. 중복되는 파일로 인한 오류를 방지하기위해 overwrite 옵션을 넣어놨습니다.
```
version: 0.0
os: linux
files:
  - source:  /
    destination: /home/ec2-user/CICD-Practice/
    overwrite: yes
file_exists_behavior: OVERWRITE
```

### Codepipeline
이제 모든 준비가 끝났습니다.파이프라인 생성을 누른 뒤 이름과 새 서비스 역할을 선택해줍니다.
![](https://velog.velcdn.com/images/noyohanx/post/93ff316b-1351-442b-b6df-7a41e7736080/image.png)
이후 Github와 연동을 하여 리포지토리 이름과 브랜치 이름을 설정해줍니다. 설정해준 리포지토리, 브랜치에서 변동 사항이 일어나면 이 파이프라인이 자동으로 시작될겁니다.
![](https://velog.velcdn.com/images/noyohanx/post/51cdc8da-6f94-46bc-bca9-830c0c393816/image.png)
빌드 단계를 추가해주는 단계입니다. codebuild로 빌드를 할 것이기에 codebuild를 선택해주고 만들었던 프로젝트를 선택해주면 됩니다.
![](https://velog.velcdn.com/images/noyohanx/post/1e6b3b99-bb64-4e06-85ae-a6af37d71bc3/image.png)
배포 단계를 추가해주는 단계입니다.. codedeploy로 배포를 할 것이기에 codedeploy를 선택해주고 만들었던 프로젝트, 배포그룹을 선택해줍니다.
![](https://velog.velcdn.com/images/noyohanx/post/a96d461e-c54e-405f-9646-22b9bee557c0/image.png)
그럼 이제 CI/CD를 위한 과정들이 모두 끝났습니다. 그럼 이제 CI/CD가 잘 되는지 테스트를 해봅시다.
### 코드 변경하고 변경되는지 테스트하기
간단한 예제이기에 웹사이트에서 진행했씁니다. 기존 return 값인 `hello world` 를 `cicd/test!!` 라는 문자열로 바꿔주고 커밋을 하였습니다.
![](https://velog.velcdn.com/images/noyohanx/post/854bfadb-0a2c-4c82-adeb-8f80e347ebd0/image.png)
커밋을 하고 파이프라인에 들어가보면 이렇게 source-build-deploy 과정이 진행되고 있을 것입니다.이제 에러없이 이 모든 과정이 잘 수행되기를 기도하면됩니다.
![](https://velog.velcdn.com/images/noyohanx/post/b73a3819-e341-44cc-9b97-322cabae54f0/image.png)
셋 다 성공. 이제 서버에 들어가 return 값이 잘 바뀌었는지 확인을 해주면 됩니다. (스샷의 파이프라인 실행 ID가 다른 이유는 필자가 이 글을 작성하면서 삽질을 하느라 시도를 여러번해서 그렇습니다..ㅎㅎ)
![](https://velog.velcdn.com/images/noyohanx/post/9a251f04-75e9-4a33-8339-a92f3425b66c/image.png)
짠! 그럼 이렇게 변경 사항이 잘 반영된 것을 볼 수 있습니다.
![](https://velog.velcdn.com/images/noyohanx/post/9b51e5f7-696d-4b94-9ac1-c0d964da24c7/image.png)

![](https://velog.velcdn.com/images/noyohanx/post/5b84980c-1ea0-4171-bf90-dca86d017c8e/image.png)


## 글을 마치며
이 코드 series를 처음 써봤을때 굉장히 많은 삽질을 하였는데요..(지금도 원트에 안됨) 또 다시 같은 삽질을 반복하지 않기위해 이렇게라도 사용해봤던 서비스들을 정리하려고해요.

많이 써보지 않은 서비스이고, 백엔드 부분도 깊게 알진 못해서 설명도 난잡하고 부족한 글인데도 끝까지 봐주셔서 감사합니다.

궁금한 부분이나 안되는 부분은 댓글로 질문해주시면 열심히 답변해보겠습니다! 다시 한번 끝까지 읽어주셔서 감사합니다!
