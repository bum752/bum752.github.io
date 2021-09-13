---
title: "HandlerMethodArgumentResolver을 이용해 파라미터 어노테이션 만들기"
description: "컨트롤러 메소드에 전달 된 헤더의 값을 가공해주는 어노테이션을 만드는 예를 통해 HandlerMethodArgumentResolver를 소개합니다."
excerpt: "컨트롤러 메소드에 전달 된 헤더의 값을 가공해주는 어노테이션을 만드는 예를 통해 HandlerMethodArgumentResolver를 소개합니다."
time: 2020-04-25 15:46:33
categories: ["spring", "spring boot"]
header:
  teaser: "https://user-images.githubusercontent.com/20104232/80274274-2b134400-8714-11ea-96db-e09ba79c77e4.png"
---

> - HandlerMethodArgumentResolver을 이용해 파라미터 어노테이션 만들기 (현재 글)
> - [컨트롤러 메소드의 파라미터에 사용된 커스텀 어노테이션의 Swagger 문서화](../swagger-for-method-argument-annotation)

---

헤더에 담긴 인증정보를 처리해 사용해야 하는 경우가 있습니다.  
이 때 컨트롤러에서 `@RequestHeader` 어노테이션을 사용해 값을 전달받을 수 있습니다.

아래처럼 헤더의 `session_id`의 값을 읽어 user id를 가져와 응답하는 컨트롤러를 예로 사용해보겠습니다.

```java
@GetMapping("/")
public Integer getUserId(@RequestHeader("session_id") String sessionId) {
  return sessionRepository.getUserIdBy(sessionId);
}
```

하나의 컨트롤러에서 이렇게 `session_id`를 이용해 user id를 가져와 사용할 때는 문제가 없어보입니다.  
하지만 API에서는 인증정보를 이용하는 컨트롤러가 굉장히 많을 것입니다.

이럴 때마다 같은 로직을 반복해서 작성해야 할까요?  
어노테이션을 직접 작성해 반복된 내용을 없애줄 수 있습니다.

`@RequestHeader` 어노테이션처럼 `@SessionIdToUserId` 어노테이션을 직접 만들어보겠습니다.

```java
@Retention(RetentionPolicy.RUNTIME)
@Target(ElementType.PARAMETER)
public @interface SessionIdToUserId {

  String value() default "session_id";
}
```

이 어노테이션을 사용한 컨트롤러는 아래처럼 바뀝니다.

```java
@GetMapping("/")
public Integer getUserId(@SessionIdToUserId Integer userId) {
  return userId;
}
```

반복된 로직을 제거할 수 있는 간단한 컨트롤러 메소드로 바꼈습니다.  
하지만 이 어노테이션은 아직 아무 역할을 하지 않기 때문에 `HandlerMethodArgumentResolver`를 이용해 `session_id`를 `userId`로 변경해줄 수 있도록 해야 합니다.

`HandlerMethodArgumentResolver`을 구현하는 클래스를 만들어줍니다.

```java
@Component
public class SessionIdToUserIdResolver implements HandlerMethodArgumentResolver {

  private final SessionRepository sessionRepository;

  public SessionIdToUserIdResolver(SessionRepository sessionRepository) {
    this.sessionRepository = sessionRepository;
  }

  @Override
  public boolean supportsParameter(MethodParameter parameter) {
    return parameter.hasParameterAnnotation(SessionIdToUserId.class) 
      && Integer.class.equals(parameter.getGenericParameterType());
  }

  @Override
  public Object resolveArgument(MethodParameter parameter, ModelAndViewContainer mavContainer, NativeWebRequest webRequest, WebDataBinderFactory binderFactory) throws Exception {
    SessionIdToUserId annotation = parameter.getParameterAnnotation(SessionIdToUserId.class);

    assert annotation != null;

    String sessionIdHeaderName = annotation.value();
    String sessionId = webRequest.getHeader(sessionIdHeaderName);

    return sessionRepository.getUserIdBy(sessionId);
  }
}
```

`supportsParameter` 메소드에 파라미터가 이 resolver를 지원하는지 정의해줍니다.  
Integer의 파라미터에 이 어노테이션이 사용된 경우 `true`를 리턴합니다.

`resolveArgument` 메소드는 `supportsParameter`가 `true`인 경우 이 어노테이션이 붙은 파라미터에 값을 넣을 수 있도록 구현합니다.  
요청의 헤더에서 `@SessionIdToUserId`의 `value`에 해당하는 헤더를 읽어 `sessionId`를 가져와  
변경 전 컨트롤러의 로직처럼 `sessionRepository`를 주입받아 `userId`를 조회할 수 있도록 했습니다.

마지막으로 `WebMvcConfigurer`를 상속받은 클래스의 `addArgumentResolver` 메소드를 오버라이드해 이 resolver 클래스를 추가해줍니다.

```java
@Configuration
public class WebMvcConfigurerImpl implements WebMvcConfigurer {

  private final SessionIdToUserIdResolver sessionIdToUserIdResolver;

  public WebMvcConfigurerImpl(SessionIdToUserIdResolver sessionIdToUserIdResolver) {
    this.sessionIdToUserIdResolver = sessionIdToUserIdResolver;
  }

  @Override
  public void addArgumentResolvers(List<HandlerMethodArgumentResolver> resolvers) {
    resolvers.add(sessionIdToUserIdResolver);
  }
}
```

`@SessionIdToUserId` 어노테이션을 사용한 변경 된 컨트롤러를 테스트 합니다.

```java
@WebMvcTest(AfterController.class)
@DisplayName("@SessionIdToUserID 어노테이션")
class AfterControllerTest {

  @Autowired
  MockMvc mockMvc;

  @MockBean
  SessionRepository sessionRepository;

  @Test
  @DisplayName("MethodArgumentResolver 이용해 userId 획득")
  public void test() throws Exception {
    final String sessionId = "MTQ0NjJkZmQ5OTM2NDE1ZTZjNGZmZjI3";
    final Integer userId = 1001;

    given(sessionRepository.getUserIdBy(sessionId)).willReturn(userId);

    mockMvc
        .perform(
            MockMvcRequestBuilders
                .get("/")
                .header("session_id", sessionId)
        )
        .andExpect(MockMvcResultMatchers.content().string(String.valueOf(userId)));
  }
}
```

![image](https://user-images.githubusercontent.com/20104232/80274146-3f0a7600-8713-11ea-92b1-986398b24fc3.png)

---

> - HandlerMethodArgumentResolver을 이용해 파라미터 어노테이션 만들기 (현재 글)
> - [컨트롤러 메소드의 파라미터에 사용된 커스텀 어노테이션의 Swagger 문서화](../swagger-for-method-argument-annotation)
