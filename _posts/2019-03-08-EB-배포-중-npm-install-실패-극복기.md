---
title: 'EB(AWS Elastic Beanstalk) 배포 중 npm install 실패 극복기'
time: 2018-03-08 18:34:00
categories: [aws, elastic beanstalk, eb, npm, node]
header:
  teaser: 'https://user-images.githubusercontent.com/20104232/64587523-a79e2000-d3da-11e9-9bea-771b5de9c539.png'
---

## EB(AWS Elastic Beanstalk) 배포 중 npm install 실패 극복기

스타트업에서 인턴을 하며 경험한 node.js 의존성 모듈 설치 실패에 대해 공유해보고자 합니다.

프로젝트에 node.js를, 배포 환경으로 eb를 사용하고 있었습니다. 서비스 중인 eb 환경에 배포하는 경우 문제가 없지만 개발 서버에 배포시에는 `npm install` 중 실패가 간헐적으로 발생했습니다.

> **개발 서버 환경**  
> t2.micro  
> node.js 7.10.1  
> npm 4.2.0

### 에러의 발생

에러의 내용은 아래처럼 `npm install` 과정 중 발생하는 것을 알 수 있지만 명확한 원인을 알려주지 않았습니다.

> Elastic Beanstalk 이벤트  
> ![image](https://user-images.githubusercontent.com/20104232/53719281-ff465680-3ea0-11e9-8cbe-4299567ed5c7.png)

> /var/log/eb-activity.log
>
> ```bash
> Running npm install: /opt/elasticbeanstalk/node-install/node-v7.10.1-linux-x64/bin/npm
> Setting npm config jobs to 1
> npm config jobs set to 1
> Running npm with --production flag
> Failed to run npm install. Snapshot logs for more details.
> UTC 2019/01/13 03:13:37 cannot find application npm debug log at /tmp/deployment/application/npm-debug.log
> 
> Traceback (most recent call last):
> File "/opt/elasticbeanstalk/containerfiles/ebnode.py", line 695, in <module>
>  main()
> File "/opt/elasticbeanstalk/containerfiles/ebnode.py", line 677, in main
>  node_version_manager.run_npm_install(options.app_path)
> File "/opt/elasticbeanstalk/containerfiles/ebnode.py", line 136, in run_npm_install
>  self.npm_install(bin_path, self.config_manager.get_container_config('app_staging_dir'))
> File "/opt/elasticbeanstalk/containerfiles/ebnode.py", line 180, in npm_install
>  raise e
> subprocess.CalledProcessError: Command '['/opt/elasticbeanstalk/node-install/node-v7.10.1-linux-x64/bin/npm', '--production', 'install']' returned non-zero exit status 1 (Executor::NonZeroExitStatus) 
> ```

### 원인

eb에 접속해 ebnode.py의 내용도 확인해보고 여러가지 키워드를 이용해 검색도 해보며 원인과 해결방법을 찾아보려 했습니다.

1. ebnode.py에서는 단지 npm install 명령어를 처리할 뿐...

2. 배포할 때 node_modules 가 포함되어서다.  
   node_modules를 포함하지 않고 배포시 `npm install`을 통해 의존성을 설치하며 그 과정에서 발생하기 때문에 이 문제는 아니라 판단했습니다.

3. 메모리 부족이다. (블로그 참고, :link:[An Insufficient Memory Deployment Failure](https://medium.com/@deanslamajr/an-insufficient-memory-deployment-failure-d9f1cb9b5c0) by Dean Slama Jr)  
   가장 그럴듯해 보이는 원인이지만 확실히 원인으로 제시하는 자료들을 찾을 수 없었습니다. (검색 능력이 부족한 제 탓을...)  
   하지만 메모리 부족을 원인으로 추측해 해결 방법으로 제시된 내용은

   - EC2 인스턴스 사이즈를 키워라!  
     가장 간단한 방법이지만 개발 서버에 비용을...
   - 개발 의존 모듈을 운영환경에서는 설치하지 말라  
     모듈은 이미 분리되어 있으며 운영환경에서는 production 옵션이 붙기 때문에 설치하지 않습니다.
   - 프론트 앱을 번들링해서 배포하라  
     배포되는 앱은 프론트 앱이 아닙니다...ㅜ
   - 도커를 사용하라  
     도커를 정말 좋게 생각하지만 역시 개발서버에 적용하기 위해 들여야하는 노력이 큰 것 같았습니다.

   추가적으로 이 원인에 대해 해결할 수 있을 것 같은 방법은

   - 메모리 스왑
   - 의존성 모듈 버전 고정(`npm shrinkwrap`)으로 설치되어 있는 모듈 재설치 방지

메모리 부족이라고 명시되는 로그가 없었기 때문에 `npm install`시 사용하는 메모리를 `top` 명령어를 이용해 측정했습니다.

> npm install 시 메모리 확인 (npm v4.2.0)
> ![image](https://user-images.githubusercontent.com/20104232/54004861-94e02f80-419a-11e9-8ef0-471a1242381a.png)

t2.micro는 1GiB의 메모리를 제공합니다. 하지만 `npm install`이 메모리의 50% 이상 사용했고 이렇게 메모리 문제라는 추측이 맞는 듯 보였습니다. 하지만 더 확신이 필요했습니다. 로그 파일들은 모조리 찾아본 것 같았지만 놓치고 있었던 로그가 있었습니다.

로그 중

1. `Snapshot logs for more details.`의 Snapshot logs는 eb에서 제공하는 로그인데 위에 언급된 로그(`/var/log/eb-activity.log`)와 같은 형식입니다. *(이 로그만으로는 원인 파악을 할 수 없었습니다.)*  

> Elastic Beanstalk 메뉴 중 로그  
> ![image](https://user-images.githubusercontent.com/20104232/53773454-e20b9980-3f2d-11e9-9d41-b78184daf656.png)

2. `cannot find application npm debug log at /tmp/deployment/application/npm-debug.log`에 있는 `npm-debug.log`는 존재하지 않는 파일이었고 로그가 저장되지 않는 것으로 오해했습니다.  
   npm v4.2.0부터는 `npm install`시 프로젝트 경로에 `npm-debug.log`파일을 저장하지 않고 캐시 경로 내 `(cache)/_logs/` 아래에 파일을 저장합니다. (~~때마침 4.2.0을 쓰고 있었네요...~~)

> :link:[npm v4.2.0 릴리즈 노트](https://github.com/npm/npm/releases/tag/v4.2.0)  
> WHERE DID THE DEBUG LOGS GO
> This is another pretty significant change: Usually, when the npm process crashed, you would get an npm-debug.log in your current working directory. This debug log would get cleared out as soon as you ran npm again. This was a bit annoying because 1) you would get a random file in your git status that you might accidentally commit, and 2) if you hit a hard-to-reproduce bug and instinctually tried again, you would no longer have access to the repro npm-debug.log.
>
> So now, any time a crash happens, **we'll save your debug logs to your cache folder, under _logs (~/.npm on *nix, by default -- use npm config get cache to see what your current value is)**. The cache will now hold a (configurable) number of npm-debug.log files, which you can access in the future. Hopefully this will help clean stuff up and reduce frustration from missed repros! In the future, this will also be used by npm report to make it super easy to put up issues about crashes you run into with npm. 💃🕺🏿👯‍♂️

캐시 디렉토리 아래 _logs 에 저장되어 있는 로그입니다.

> /tmp/.npm/_logs/...-debug.log
>
> ```bash
> $ tail /tmp/.npm/_logs/2019-03-04T07_53_12_242Z-debug.log
> 55423 error argv "/opt/elasticbeanstalk/node-install/node-v7.10.1-linux-x64/bin/node" "/opt/elasticbeanstalk/node-install/node-v7.10.1-linux-x64/bin/npm" "--production" "install"
> 55424 error node v7.10.1
> 55425 error npm  v4.2.0
> 55426 error code ENOMEM
> 55427 error errno ENOMEM
> 55428 error syscall spawn
> 55429 error spawn ENOMEM
> 55430 error If you need help, you may report this error at:
> 55430 error     <https://github.com/npm/npm/issues>
> 55431 verbose exit [ 1, true ]
> ```

추측하고 있었지만 원인이 무엇인지(ENOMEM, Out Of Memory) 확실히 알아냈습니다.

### 해결

npm 5.0.0 부터 디스크를 이용해 패키지를 다운로드한다고 합니다. (out of memory 없이)

> :link:[npm v5.0.0 릴리즈 노트](https://github.com/npm/npm/releases/tag/v5.0.0)  
> Downloads for large packages are streamed in and out of disk. npm is now able to install packages of """any""" size **without running out of memory**. Support for publishing them is pending (due to registry limitations).

더 높은 버전의 npm 을 사용할 수 있도록 eb 구성의 node.js 버전을 업그레이드 했습니다.  
노드 버전에 따라 사용하는 npm 버전은 :link:[node.js 이전 릴리스](https://nodejs.org/ko/download/releases/)에서 확인할 수 있습니다.

> node.js 7.10.1 > 8.11.3
> npm 4.2.0 > 5.6.0

높은 버전을 사용해 해결이 된 것을 확인하고자 의존성 설치 과정의 메모리 사용을 `top` 명령어를 이용해 또 한번 측정했습니다.

> npm install 시 메모리 확인 (npm v5.6.0)
> ![image](https://user-images.githubusercontent.com/20104232/54004955-f1434f00-419a-11e9-8ace-df5b461c0c43.png)

확실히 더 적은 메모리를 사용하는 것을 볼 수 있습니다.

### 마치며

인턴을 하는 동안 발생한 첫 문제였고 오래 걸렸지만 원인을 찾아 해결할 수 있었던 경험이었습니다. (~~검색해도 동일 현상에 대한 원인을 찾지못해 헤매다 해결해서 뿌듯!~~)

:link:[node.js 이전 릴리스](https://nodejs.org/ko/download/releases/) 기준 npm v5.4.2을 사용하는 <u>v8.8.0 이하의 nodejs</u>를 사용하고 t2.micro 같이 비교적 저사양의 운영환경을 가지고 있는 분들이 저와 같은 문제를 겪으시고 이 글을 통해 쉽게 해결하실 수 있었으면 해서 이 글을 공유합니다.

감사합니다.

------

참고자료  

- https://stackoverflow.com/questions/42769975/elastic-beanstalk-npm-failing
- https://medium.com/@deanslamajr/an-insufficient-memory-deployment-failure-d9f1cb9b5c0
- https://tutel.me/c/programming/questions/43969008/aws+eb+deploying+node+app+failed+to+run+npm+install