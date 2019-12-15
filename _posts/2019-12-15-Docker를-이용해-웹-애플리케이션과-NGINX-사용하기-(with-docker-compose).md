---
title: "Docker를 이용해 웹 애플리케이션과 NGINX 사용하기 (with docker-compose)"
description: "스프링 애플리케이션을 컨테이너 이미지로 만들어 nginx 로 프록시하는 방법을 소개합니다."
excerpt: "스프링 애플리케이션을 컨테이너 이미지로 만들어 nginx 로 프록시하는 방법을 소개합니다."
time: 2019-12-15 17:00:00
categories: [java, spring, spring boot, nginx, docker, docker-compose]
header:
  teaser: "https://user-images.githubusercontent.com/20104232/70860777-62553580-1f69-11ea-8ac7-f4ed34e17421.png"
---

스프링 애플리케이션을 컨테이너 이미지로 만들어 nginx 로 프록시하는 방법을 소개합니다.

_스프링을 이용한 애플리케이션 개발은 다루지 이 포스트에서 다루지 않습니다._

## 1. 도커 설치

1. https://www.docker.com/get-started 에서 도커를 설치합니다.
1. 컨테이너 이미지를 받아오기 위해 로그인이 필요하니 가입해주세요!

## 2. 스프링 어플리케이션 개발

우선 스프링을 이용해 `Hello, world`를 응답하는 애플리케이션을 만들어줍니다.

```bash
$ curl http://localhost:8080/
Hello, world
```

## 3. 스프링 어플리케이션 빌드

스프링 프로젝트에 `Dockerfile`을 생성합니다.

```
# Dockerfile

# https://docs.docker.com/engine/reference/builder/#from
# java 8 사용
FROM java:8

# https://docs.docker.com/engine/reference/builder/#expose
# 애플리케이션 포트
EXPOSE 8080

# https://docs.docker.com/engine/reference/builder/#add
# 애플리케이션 파일 추가
ADD ./build/libs/kotlinspring-0.0.1-SNAPSHOT.jar application.jar

# https://docs.docker.com/engine/reference/builder/#entrypoint
# 실행
ENTRYPOINT ["java", "-jar", "application.jar"]
```

Dockerfile 에 대한 내용은 [여기](https://docs.docker.com/engine/reference/builder/)에서 자세히 확인할 수 있습니다.

이제 이미지를 빌드합니다.

```bash
$ docker build --tag spring-app:0.1 . # --tag 옵션으로 이미지의 태그를 지정합니다.
```

`docker images` 명령어로 확인하면 빌드한 이미지가 생성된 것을 확인할 수 있습니다.

![image](https://user-images.githubusercontent.com/20104232/70859975-e1913c00-1f5e-11ea-90fb-44c2ca85f9de.png)

## 4. 스프링 어플리케이션 실행

`docker run -p 18080:8080 spring-app:0.1` 을 이용해 이미지를 실행합니다.  
`-p` 옵션으로 컨테이너 이미지가 사용하는 포트(8080)와 호스트가 사용할 포트(18080)를 지정할 수 있습니다.

![image](https://user-images.githubusercontent.com/20104232/70860070-1d78d100-1f60-11ea-805f-a0885ce273b2.png)

```bash
$ curl localhost:18080
Hello, world
```

## 5. docker-compose 를 이용해 spring-app 프록싱

docker-compose는 docker를 명령형으로 사용하던 것을 선언적으로 사용해 여러 이미지를 좀 더 쉽게 관리할 수 있도록 해줍니다.

먼저 nginx 설정을 위한 `nginx/nginx.conf`를 생성합니다.

```
# nginx/nginx.conf

user nginx;
worker_processes 1;

pid         /var/run/nginx.pid;

events {
    worker_connections  1024;
}

http {
    access_log  /var/log/nginx/access.log;
    error_log   /var/log/nginx/error.log;

    include       /etc/nginx/mime.types;
    default_type  application/octet-stream;

    server {
        listen 80;
        server_name localhost;
        location / {
            proxy_pass         http://spring-app:8080;
            proxy_redirect     off;
            proxy_set_header   Host $host;
            proxy_set_header   X-Real-IP $remote_addr;
            proxy_set_header   X-Forwarded-For $proxy_add_x_forwarded_for;
        }
    }

    sendfile        on;
    keepalive_timeout  65;
    include /etc/nginx/conf.d/*.conf;
}
```

nginx 설정의 예는 [여기](https://www.nginx.com/resources/wiki/start/topics/examples/full/)를 참고해주세요!

다음으로 `docker-compose.yml`을 생성합니다.

```yaml
# docker-compose.yml

version: "3"

services:
  spring-app: # 서비스명
    image: spring-app:0.1 # 이미지, 태그
  nginx:
    image: nginx
    ports:
      - "80:80"
    volumes:
      - ./nginx/nginx.conf:/etc/nginx/nginx.conf
```

- 스프링 어플리케이션은 외부로 공개될 필요가 없기 때문에 별도로 포트를 지정하지 않았습니다.
- 호스트 디바이스의 저장소와 컨테이너를 연결하기 위해 `volumes`를 사용했습니다.

다른 docker-compose.yml 설정은 [여기](https://docs.docker.com/compose/gettingstarted/)를 참고해주세요!

## 6. docker-compose 실행

`docker-compose up` 명령어를 이용해 실행합니다.

![image](https://user-images.githubusercontent.com/20104232/70860558-882d0b00-1f66-11ea-87e5-558cf03dea3b.png)

nginx와 스프링 애플리케이션이 실행되는 것을 확인할 수 있습니다.

```bash
$ curl localhost
Hello, world
```

80포트를 이용해 요청하면 스프링 어플리케이션의 응답을 받을 수 있는 것을 확인할 수 있습니다.
