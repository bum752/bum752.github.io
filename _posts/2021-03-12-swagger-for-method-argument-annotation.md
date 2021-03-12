---
title: "컨트롤러 메소드의 파라미터에 사용된 커스텀 어노테이션의 Swagger 문서화"
description: "Swagger 문서에서 커스텀 어노테이션을 다룰 수 있는 방법을 소개합니다."
excerpt: "Swagger 문서에서 커스텀 어노테이션을 다룰 수 있는 방법을 소개합니다."
time: 2021-03-12 18:26:19
categories: [java, spring boot, annotation, swagger, ParameterBuilderPlugin]
header:
  teaser: "https://user-images.githubusercontent.com/20104232/110879282-0cb03a00-8320-11eb-813d-52ec48d11177.png"
---

> - [HandlerMethodArgumentResolver을 이용해 파라미터 어노테이션 만들기](../MethodArgumentResolver)
> - 컨트롤러 메소드의 파라미터에 사용된 커스텀 어노테이션의 Swagger 문서화 (현재 글)

---

[HandlerMethodArgumentResolver을 이용해 파라미터 어노테이션 만들기](../MethodArgumentResolver)를 이용해  
컨트롤러 메소드에 전달되는 파라미터의 값을 가공할 수 있는 커스텀 어노테이션을 만들어봤습니다.

여러 API에서 필요한 중복되는 기능을 간단한 방식으로 해결할 수 있었지만  
Swagger를 이용해 자동으로 API 문서를 생산하는 환경에서는 아쉬운 점이 있었습니다.

문제가 무엇인지 확인하고 어떻게 해결할지 이 글을 통해 공유합니다.

### 문제점

- `@RequestHeader` 를 커스텀 어노테이션으로 변경했는데 Swagger에서 헤더가 아닌 파라미터로 입력된다.
- `String` 형 헤더값에서 `Integer` 형 유저 ID로 파라미터의 타입을 바꿨는데 Swagger에서 Integer를 입력해야 한다.
- `@RequestHeader` 의 옵션 중 `required` 를 이용해 필수값 여부를 지정할 수 있었지만 커스텀 어노테이션을 사용하니 Swagger 에서 선택항목이다.

![image](https://user-images.githubusercontent.com/20104232/110870822-90fac100-8310-11eb-902b-488d13977a10.png)

### 해결 방법

Swagger를 사용중이시거나 아시는 분이라면 `@ApiImplicitParam`, `@ApiIgnore` 등과 같은 Swagger가 제공하는 어노테이션을 이용해 해결 할 수 있습니다.  
이 방법의 단점은 같은 코드가 커스텀 어노테이션이 있는 곳이라면 중복해서 사용되는 것입니다.

다른 방법은 어떤게 있을까요?

Swagger는 `ParameterBuilderPlugin` 이라고 하는 인터페이스를 제공합니다.  
이 인터페이스를 구현하면 파라미터에 대한 속성을 변경해줄 수 있습니다.

![image](https://user-images.githubusercontent.com/20104232/110742564-f64fa300-8279-11eb-8540-47b133e16384.png)

```java
@Component
@Order(SwaggerPluginSupport.SWAGGER_PLUGIN_ORDER) // (1)
public class SessionIdToUserIdSwaggerPlugin implements ParameterBuilderPlugin { // (2)

  @Override
  public void apply(ParameterContext parameterContext) { // (3)
    ResolvedMethodParameter parameter = parameterContext.resolvedMethodParameter();
    Optional<SessionIdToUserId> annotation = parameter.findAnnotation(SessionIdToUserId.class);

    if (annotation.isPresent()) {
      parameterContext
        .requestParameterBuilder()
        .in(ParameterType.HEADER) // (4)
        .name("session_id") // (5)
        .description("Session ID") // (6)
        .required(Boolean.TRUE) // (7)
        .query(q -> q.model(m -> m.scalarModel(ScalarType.STRING))) // (8)
      ;
    }
  }

  @Override
  public boolean supports(DocumentationType delimiter) { // (9)
    return true;
  }
}
```

- (1) Swagger 플러그인이 적용될 `Order` 를 지정합니다.
- (2) `ParameterBuilderPlugin` 을 구현해 파라미터 속성을 수정합니다.
- (3) `ParameterBuilderPlugin` 인터페이스에 정의된 파라미터 속성을 적용할 메소드입니다.
- (4) 파라미터의 타입 (`QUERY`, `HEADER`, `PATH`, `COOKIE`, `FORM`, `FORMDATA`, `BODY`)
- (5) 파라미터명
- (6) 파라미터의 설명
- (7) 파라미터의 필수여부 (`true`, `false` (default))
- (8) 파라미터의 형
- (9) `Plugin` 인터페이스에 정의된 적용여부를 반환하는 메소드입니다.

위의 설정이 적용된 Swagger 문서입니다.
![image](https://user-images.githubusercontent.com/20104232/110876242-71689600-831a-11eb-8f3d-8df805cb2efd.png)

`@ApiImplicitParam`, `@ApiIgnore` 같이 문서를 수정하기 위한 중복되는 코드 없이  
간단한 설정으로 커스텀 어노테이션의 Swagger 문서화를 전역적으로 설정할 수 있었습니다.

이 글에서는 주로 사용할만한 옵션들을 이용해 Swagger의 문서를 수정해보았습니다.  
추가적으로 수정해야할 옵션이 있다면 [Swagger Plugin](https://springfox.github.io/springfox/docs/current/#steps-to-create-a-plugin)을 참고하시면 도움이 될 것 같습니다.

---

> - [HandlerMethodArgumentResolver을 이용해 파라미터 어노테이션 만들기](../MethodArgumentResolver)
> - 컨트롤러 메소드의 파라미터에 사용된 커스텀 어노테이션의 Swagger 문서화 (현재 글)
