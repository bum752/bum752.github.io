---
title: "Spring 로그를 슬랙으로"
description: "logback-slack-appender를 이용해 스프링 어플리케이션에서 생성되는 로그를 바로 슬랙으로 발송해봅니다."
excerpt: "logback-slack-appender를 이용해 스프링 어플리케이션에서 생성되는 로그를 바로 슬랙으로 발송해봅니다."
time: 2020-04-28 13:47:42
categories: ["java", "spring", "spring boot", "logback", "slack"]
header:
  teaser: "https://user-images.githubusercontent.com/20104232/80461630-87729f80-8970-11ea-828d-097c0523f09e.png"
---

어플리케이션을 운영하다보면 실행 시간에 수많은 예외를 경험하게 됩니다.  
런타임 예외를 빠르게 잡아내고 고쳐야 사용자 경험의 저해를 막아줄 수 있습니다.

어떻게 하면 실시간으로 이 예외를 파악할 수 있을까요?

~~(답은 정해져 있어. 그냥 하면 돼!)~~

1. `tail -f /var/logs/spring-application`  
   여러 대의 서버를 이용해 요청을 처리하는 경우 모든 서버의 로그를 띄워놓고 확인해야 합니다.  
   많은 로그가 발생해 내용이 밀릴 경우 놓치는 경우가 생기거나 다시 확인하기 어렵습니다.
2. ELK 등 모니터링 도구 사용  
   위의 단점들을 개선할 수 있습니다.  
   하지만, 모니터링 시스템을 구축하기 위한 별도 리소스가 필요합니다.
3. 메신저를 이용한 알림  
   가장 쉬운 해결 방법이 아닐까 생각합니다.  
   실시간으로 개발자에게 푸시가 가니 내용을 놓칠 일도 없고,  
   복잡한 과정 없이도 좋은 효율의 모니터링을 수행할 수 있습니다.

![image](https://user-images.githubusercontent.com/20104232/80461630-87729f80-8970-11ea-828d-097c0523f09e.png)

이 글에서는 [logback-slack-appender](https://github.com/maricn/logback-slack-appender)을 이용해 메신저 중 슬랙에 logback으로 생산된 로그 메세지를 보내는 것을 해보려고 합니다.

### logback 설정

우선 스프링 어플리케이션에 의존성을 추가합니다.

```groovy
...

ext {
    set("logbackSlackAppenderVersion", "1.4.0")
}

dependencies {
    implementation "org.springframework.boot:spring-boot-starter-web"
    implementation "com.github.maricn:logback-slack-appender:${logbackSlackAppenderVersion}"
    ...
}

...
```

`src/main/resources` 하위에 `logback-spring.xml` 파일을 만들어 주고 [README.md](https://github.com/maricn/logback-slack-appender/blob/master/README.md)에 있는 내용을 적용하면 됩니다.

저는 어플리케이션 패키지의 로그만 메세지로 받기 위해 `<root>...</root>` 부분을 아래와 같이 수정해주었습니다.

```xml
<logger name="io.github.bum752.logbackslackappenderexample" level="ALL">
    <appender-ref ref="ASYNC_SLACK"/>
</logger>
```

그리고 모든 로그가 아닌 WARN, ERROR 로그 확인을 위해 발송될 로그 레벨도 지정했습니다.

```xml
<appender name="ASYNC_SLACK" class="ch.qos.logback.classic.AsyncAppender">
    <appender-ref ref="SLACK"/>
    <filter class="ch.qos.logback.classic.filter.ThresholdFilter">
        <level>WARN</level>
    </filter>
</appender>
```

💡 `webhookUri` 필드에 환경변수를 활용할 수 있고, `layout`을 이용해 로그 패턴을 지정할 수 있습니다.

```xml
<appender name="SLACK" class="com.github.maricn.logback.SlackAppender">
    <webhookUri>${SLACK_WEBHOOK_URL}</webhookUri>
    <layout class="ch.qos.logback.classic.PatternLayout">
        <pattern>%d{yyyy-MM-dd HH:mm:ss.SSS} %-5level --- [%thread] %logger{35} : %msg %n</pattern>
    </layout>
</appender>
```

### 로그 테스트

테스트를 위해 로그를 남기는 컨트롤러를 만들었습니다.

```java
  private static final Logger logger = LoggerFactory.getLogger(LogbackSlackAppenderExampleApplication.class);

  @GetMapping("/")
  @ResponseStatus(HttpStatus.INTERNAL_SERVER_ERROR)
  public String fail() {
    final String message = "failed request";

    logger.error(message);
    logger.warn(message);
    logger.info(message);
    logger.debug(message);
    logger.trace(message);
    return message;
  }
```

요청받으면 500 코드와 함께 'failed request'를 응답하며 여러 레벨의 로그를 남깁니다.

실제로 요청하면 원하던 대로 WARN, ERROR 레벨의 로그가 슬랙에 남는 것을 확인할 수 있습니다.

![image](https://user-images.githubusercontent.com/20104232/80459853-f8fd1e80-896d-11ea-87a1-56638db87b9b.png)

![image](https://user-images.githubusercontent.com/20104232/80460573-0070f780-896f-11ea-850c-8118bb5fc4b2.png)

---

위의 내용은 [GitHub](https://github.com/bum752/logback-slack-appender-example)에 있으니 참고해주세요!
