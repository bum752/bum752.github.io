---
title: 'Spring Boot 멀티 모듈 .ebextensions와 함께 배포하기'
time: 2019-05-25 10:39:00
categories: [aws, code pipeline, code build, elastic beanstalk, eb, ebextensions, spring boot]
---

현재 진행중인 프로젝트는 Spring Boot + Gradle + JPA를 사용하고  
공통적으로 사용되는 모듈을 작성해 여러 프로젝트에서 사용하는 방식의 멀티 모듈 프로젝트입니다.

Git 레포지토리는 AWS CodeCommit을, CI/CD는 AWS Pipeline(AWS CodeBuild, AWS Elastic Beanstalk)을 사용해 모든 것을 AWS 상에서 해결합니다.

![image](https://user-images.githubusercontent.com/20104232/58363374-29c31100-7ede-11e9-9b6e-4b0a74111e80.png)

각 모듈이 Elastic Beanstalk을 이용해 배포가 이루어질 때 NginX 설정을 수정해야할 일이 생겼습니다.  
[AWS 문서](https://docs.aws.amazon.com/ko_kr/elasticbeanstalk/latest/dg/java-se-nginx.html)에 설명된 것 처럼 `.ebextensions`를 이용하는 방법이 있는데 jar 파일이 배포될 때 Elastic Beanstalk에서 `.ebextensions`를 감지하지 못해 배포는 정상적으로 이루어지지만 NginX 설정이 반영되지 않은 문제를 겪고 해결하는 방법을 공유합니다.

## 문제 발생

우선 NginX 설정이 필요한 모듈 아래에 `.ebextensions` 폴더를 만들어 필요한 설정파일을 추가합니다.

```
# 프로젝트 구성

.
├── (생략)
├── COMMON_MODULE
│   ├── build.gradle
│   └── src
│       ├── main
│       └── test
├── SERVICE_MODULE_01
│   ├── .ebextensions
│   │   └── nginx
│   │       └── nginx.conf
│   ├── build.gradle
│   └── src
│       └── (생략)
└── SERVICE_MODULE_02
    ├── build.gradle
    └── src
        └── (생략)
```

빌드 후 jar를 생성할 때 `.ebextensions`를 포함하기 위해 프로젝트의 `build.gradle`에 아래와 같은 설정을 추가했습니다.

```
# build.gradle

subprojects {

    bootJar {
        from('./.ebextensions') {
            into '.ebextensions'
        }
    }

}
```

기존에 사용하던 AWS CodeBuild Buildspec 입니다.

```
# Buildspec.yml

(생략)
artifacts:
  secondary-artifacts:
    SERVICE_01:
      files:
        - SERVICE_MODULE_01/build/lib/*.jar
      discard-paths: yes
    SERVICE_02:
      files:
        - SERVICE_MODULE_01/build/lib/*.jar
      discard-paths: yes
(생략)
```

jar 파일에는 `.ebextensions`가 포함되어 있지만 SERVICE_MODULE_01 모듈이 배포될 때 `.ebextensions`를 감지하지 않아 NginX 설정이 반영되지 않는 문제가 발생했습니다.

## 원인

jar 파일에 분명 `.ebextensions`가 잘 포함되어 있습니다.

```
# jar unzip 후 내용 확인

$ ls -al
total 0
drwxr-xr-x   6 jbshin  staff  192  5 25 10:56 .
drwx------+ 11 jbshin  staff  352  5 25 10:56 ..
drwxr-xr-x@  3 jbshin  staff   96  5 24 02:54 .ebextensions
drwxr-xr-x@  4 jbshin  staff  128  5 24 02:54 BOOT-INF
drwxr-xr-x@  3 jbshin  staff   96  5 24 02:54 META-INF
drwxr-xr-x@  3 jbshin  staff   96  4  4 02:23 org
```

Elastic Beanstalk의 애플리케이션 버전 페이지에서 업로드 된 파일이 zip으로 압축 된 파일이라는 것을 확인했습니다.  
CodeBuild가 출력 Artifacts를 생성할 때 jar 파일을 압축하는데 `.ebextensions`가 jar와 함께 압축되지 않아 발생하는 문제였습니다. :sweat_drops:

## 해결

jar 파일에 .ebextensions를 포함하는 것은 무의미하다고 생각해 `build.gradle`의 설정을 제거하고 CodeBuild의 Buildspec에서 빌드 후 jar와 .ebextensions을 출력 Artifacts가 될 수 있도록 수정하기로 했습니다.

```
# Buildspec.yml

(생략)
phases:
  (생략)
  post_build:
    commands:
      # 빌드 후 jar 파일이 생성되는 디렉터리에 .ebextensions를 복사해줍니다.
      - cp -r SERVICE_MODULE_01/.ebextensions SERVICE_MODULE_01/build/libs/
artifacts:
  secondary-artifacts:
    SERVICE_01:
      # base-directory 아래 모두를 출력 artifacts로 지정합니다.
      files:
        - '**/*'
      base-directory: SERVICE_MODULE_01/build/libs
    SERVICE_02:
      files:
        - '*.jar'
      base-directory: SERVICE_MODULE_02/build/libs
(생략)
```

수정된 Buildspec으로 빌드 후 Elastic Beanstalk를 통해 배포된 어플리케이션은 NginX 설정이 정상적으로 이루어졌습니다. :clap:

```
-------------------------------------
/var/log/eb-activity.log
-------------------------------------
Nginx configuration detected in the '.ebextensions/nginx' directory. AWS Elastic Beanstalk will no longer manage the Nginx configuration for this environment.
```

![image](https://user-images.githubusercontent.com/20104232/58363397-858d9a00-7ede-11e9-8311-610613ae82fb.png)