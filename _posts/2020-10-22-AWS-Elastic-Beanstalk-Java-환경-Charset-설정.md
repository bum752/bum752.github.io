---
title: "AWS Elastic Beanstalk Java 환경 Charset 설정 (feat. JVM 옵션 설정)"
time: 2020-10-22 16:28:46
categories:
  [
    aws,
    elastic beanstalk,
    eb,
    java,
    jar,
    jvm,
    charset,
    UTF-8,
    utf8,
    spring boot,
  ]
header:
  teaser: "https://user-images.githubusercontent.com/20104232/64586886-8e946f80-d3d8-11e9-859e-4beb8ab11652.png"
---

이 글은 [AWS Elastic Beanstalk Procfile을 이용한 Java 환경 Charset 설정](../AWS-Elastic-Beanstalk-Procfile을-이용한-Java-환경-Charset-설정)에 이어  
Elastic Beanstalk 환경 변수를 이용해 Java 환경 애플리케이션의 Charset 설정할 수 있는 방법을 공유합니다.

---

앞선 글의 마지막 내용 중 아래와 같은 내용이 있습니다.

> Procfile을 이용하면 여러 개의 애플리케이션을 실행시킬 수 있고 위처럼 옵션을 이용해 애플리케이션 실행을 직접 제어할 수 있고, CD 환경을 사용하고 있다면 앱을 배포할 때 jar와 함께 Procfile을 함께 배포해 편리하게 사용할 수 있습니다.

이런 장점이 있기는 하지만 Procfile을 애플리케이션과 함께 배포하지 않고도 설정할 수 있는 방법이 있습니다.

## Elastic Beanstalk 의 환경 변수 설정

자바 애플리케이션이 실행될 때 [`JAVA_TOOL_OPTIONS`](https://docs.oracle.com/javase/8/docs/technotes/guides/troubleshoot/envvars002.html) 환경 변수를 읽어와 사용합니다.  
이 환경 변수를 이용하면 원하던 Charset 설정을 빈스톡 구성 변경으로 간단하게 해결할 수 있습니다.

<u>(Elastic Beanstalk 환경 - 구성 - 소프트웨어 메뉴에서 설정할 수 있습니다.)</u>

- 키: `JAVA_TOOL_OPTIONS`
- 값: `-Dfile.encoding=UTF-8`

환경 변수를 저장하고 애플리케이션이 재실행 후 기대했던 대로 사용이 가능합니다.

### 다른 옵션을 추가로 사용하고 싶어요!

만약 애플리케이션이 사용하고 있는 Timezone을 변경하고 싶다면 어떻게 해야할까요?
애플리케이션에서 코드를 이용해 설정할 수 있는 방법도 있지만 환경 변수를 사용할 수 있습니다.
이 때 동일하게 `JAVA_TOOL_OPTIONS` 를 사용해 여러개의 옵션을 지정할 수 있습니다.

예를 들어 Timezone, Charset을 동시에 세팅하고자 한다면

- 키: `JAVA_TOOL_OPTIONS`
- 값: `-Dfile.encoding=UTF-8 -Duser.timezone=Asia/Seoul`

과 같이 사용할 수 있습니다.

---

필요에 따라 _Elastic Beanstalk 의 환경변수_ 또는 _Procfile_ 을 선택해 사용하면 좋을 것 같습니다!
