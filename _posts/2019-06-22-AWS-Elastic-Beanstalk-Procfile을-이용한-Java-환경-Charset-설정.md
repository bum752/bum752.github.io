---
title: "AWS Elastic Beanstalk Procfile을 이용한 Java 환경 Charset 설정 (feat. JVM 옵션 설정)"
time: 2019-06-22 17:00:00
categories:
  [
    aws,
    elastic beanstalk,
    eb,
    java,
    jar,
    jvm,
    procfile,
    charset,
    UTF-8,
    utf8,
    spring boot,
  ]
header:
  teaser: "https://user-images.githubusercontent.com/20104232/64586886-8e946f80-d3d8-11e9-859e-4beb8ab11652.png"
---

> 💡  
> Procfile이 아닌 Elastic Beanstalk의 환경 변수를 이용하는 방법도 있으니 참고해주세요!  
> [AWS Elastic Beanstalk Java 환경 Charset 설정](../AWS-Elastic-Beanstalk-Java-환경-Charset-설정)

---

Elastic Beanstalk에 배포된 자바 애플리케이션에서 한글이 정상적으로 처리되지 않은 문제를 겪고 해결한 방법을 공유합니다.

### 원인

Java는 Default Charset 설정을 System Property의 `file.encoding` 을 읽어와 사용합니다.

Elastic Beanstalk의 Java 환경은 jar 파일을 실행할 때 별도의 옵션없이 아래처럼 실행시킵니다. ("[AWS Elastic Beanstalk Java SE 플랫폼 사용](https://docs.aws.amazon.com/ko_kr/elasticbeanstalk/latest/dg/java-se-platform.html)" 참고)

```
java -jar application_name.jar
```

위의 명령어를 통해 실행되는 애플리케이션의 설정을 확인하기 위해 코드를 작성해 jar 를 만들어 배포해보았습니다.

eb 로그를 확인해보니 `file.encoding`이 `ANSI_X3.4-1968`이었습니다.

```
2019-06-23 03:33:16.854  INFO 4041 --- [           main] com.example.demo.DemoApplication         : ##### file.encoding: ANSI_X3.4-1968
```

`file.encoding`이 `UTF-8`이 아니어서 발생한 문제인 것을 확인했습니다.

### 해결

jar를 실행할 때 java 명령어의 옵션을 이용해 jvm의 설정을 바꿀 수 있습니다.

```
$ java --help

(생략)
    -D<name>=<value>
                  시스템 속성을 설정합니다.
(생략)
```

`-D` 옵션으로 `file.encoding`을 변경하는 방법은 아래와 같습니다.

```
$ java -Dfile.encoding=UTF-8 -jar application_name.jar
```

애플리케이션을 직접 실행시키는 환경이라면 위처럼 실행시키면 되지만 Elastic Beanstalk는 애플리케이션을 수동으로 실행시키지 않습니다.

이 문제 해결을 위해 `Procfile`을 이용할 수 있습니다. ("[Procfile을 사용하여 애플리케이션 프로세스 구성](https://docs.aws.amazon.com/ko_kr/elasticbeanstalk/latest/dg/java-se-procfile.html)" 참고)

jar와 함께 배포 될 `Procfile`을 생성해 애플리케이션 실행 명령을 입력합니다.

```
# Procfile
web: java -Dfile.encoding=UTF-8 -jar application_name.jar
```

`web`이라는 애플리케이션을 `java -Dfile.encoding=UTF-8 -jar application_name.jar` 명령으로 실행시킨다는 의미입니다.

Procfile을 `file.encoding`을 `UTF-8`로 변경해 한글을 정상적으로 처리하지 못하는 문제를 해결할 수 있었습니다.

---

Procfile을 이용하면 여러 개의 애플리케이션을 실행시킬 수 있고 위처럼 옵션을 이용해 애플리케이션 실행을 직접 제어할 수 있고, CD 환경을 사용하고 있다면 앱을 배포할 때 jar와 함께 Procfile을 함께 배포해 편리하게 사용할 수 있습니다.
