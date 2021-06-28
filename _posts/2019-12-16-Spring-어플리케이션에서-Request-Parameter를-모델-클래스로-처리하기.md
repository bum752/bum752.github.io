---
title: "Spring 어플리케이션에서 Request Parameter를 모델 클래스로 처리하기"
description: "스프링 애플리케이션에서 Request Parameter를 @RequestParam 어노테이션을 사용하지 않고 모델 클래스를 이용해 처리하는 방법을 공유합니다."
excerpt: "스프링 애플리케이션에서 Request Parameter를 @RequestParam 어노테이션을 사용하지 않고 모델 클래스를 이용해 처리하는 방법을 공유합니다."
time: 2019-12-16 15:17:34
categories: [java, spring, spring boot]
header:
  teaser: "https://user-images.githubusercontent.com/20104232/70888902-b53ff300-2024-11ea-8ac1-f8329bab9607.png"
---

스프링 애플리케이션에서 Request Parameter를 어떻게 다룰 수 있을까요?

```java
@GetMapping("/users")
public Page<User> getUsers(
    @RequestParam(name = "name", required = false) String userName,
    @RequestParam(name = "phone", required = false) String userPhone,
    // ...
) {
    // ...
}
```

`@RequestParam` 어노테이션을 사용하면 매우 간단합니다. 하지만 파라미터의 수가 많아진다면 코드의 가독성을 떨어뜨립니다.

이를 해결할 방법으로 모델 클래스를 사용할 수 있습니다.

```java
@Data // lombok
public class GetUserModel {

    private String userName;
    private String userPhone;
    // ...
}

@GetMapping("/users")
public Page<User> getUsers(GetUserModel getUserModel) {
    // ...
}
```

모델 클래스를 사용해서 Request Parameter 매핑을 했지만 `@RequestParam`이 제공하는 `name`의 기능은 사용할 수 없어 클라이언트는 모델의 멤버변수명과 동일하게 요청해야 합니다.

이 경우에 해결할 수 있는 방법을 공유합니다.  
(아래에 나오는 테스트에서는 모델을 그대로 응답하는 컨트롤러 메소드를 사용했습니다.)

## 방법 1. Jackson을 이용한 Map 매핑

이 방법은 모델 클래스를 사용하지 않습니다.

```java
@Getter // lombok
public class GetUserModel {

    @JsonAlias("name")
    private String userName;

    @JsonAlias("phone")
    private String userPhone;
    // ...
}
```

```java
import com.fasterxml.jackson.databind.ObjectMapper;

@GetMapping("/users")
public Page<User> getUsers(@RequestParam Map<String, String> parameterMap) {
    ObjectMapper objectMapper = new ObjectMapper();
    GetUserModel getUserModel = objectMapper.convertValue(parameterMap, GetUserModel.class);

    // ...
}
```

### 테스트

요청할 때 파라미터에 `name`과 `phone`을 입력하고 컨트롤러에서 모델의 멤버변수명으로 제대로 매핑되는지 확인합니다.

- `WebMvcTest`를 사용합니다.
- 위에서 말씀드렸던 것처럼 테스트에서는 **요청한 파라미터 그대로 모델을 응답**합니다.
- `MockMvcResultMatchers.jsonPath()`를 이용해 응답 json의 값을 가져와 요청과 동일한지 확인합니다.

```java
@ExtendWith(SpringExtension.class)
@WebMvcTest(UserController.class)
class UserControllerTest {

	@Autowired
	MockMvc mockMvc;

	@Test
	public void jackson_이용_테스트() throws Exception {
		String name = "Eddy";
		String phone = "010xxxxXXXX";

		MultiValueMap<String, String> requestParam = new LinkedMultiValueMap<>();
		requestParam.set("name", name);
		requestParam.set("phone", phone);

		mockMvc
				.perform(
						MockMvcRequestBuilders.get("/users/jackson")
								.params(requestParam)
				)
				.andDo(MockMvcResultHandlers.print())
				.andExpect(MockMvcResultMatchers.status().isOk())
				.andExpect(MockMvcResultMatchers.jsonPath("$.userName").value(name))
				.andExpect(MockMvcResultMatchers.jsonPath("$.userPhone").value(phone));
	}
}
```

![image](https://user-images.githubusercontent.com/20104232/70885856-65116280-201d-11ea-9cd9-75c17b719d98.png)

## 방법 2. set 메소드 사용

모델 클래스에 원하는 이름의 set 메소드를 생성해줍니다.

```java
@Getter
public class GetUserModelUsingSetter {

    private String userName;
    private String userPhone;
    // ...

    public void setName(String name) {
        this.userName = name;
    }

    public void setPhone(String phone) {
        this.userPhone = phone;
    }

    // ... setter
}
```

### 테스트

```java
@ExtendWith(SpringExtension.class)
@WebMvcTest(UserSetterController.class)
class UserSetterControllerTest {

	@Autowired
	MockMvc mockMvc;

	@Test
	public void setter_이용_테스트() throws Exception {
		String name = "Eddy";
		String phone = "010xxxxXXXX";

		MultiValueMap<String, String> requestParam = new LinkedMultiValueMap<>();
		requestParam.set("name", name);
		requestParam.set("phone", phone);

		mockMvc
				.perform(
						MockMvcRequestBuilders.get("/users/setter")
								.params(requestParam)
				)
				.andDo(MockMvcResultHandlers.print())
				.andExpect(MockMvcResultMatchers.status().isOk())
				.andExpect(MockMvcResultMatchers.jsonPath("$.userName").value(name))
				.andExpect(MockMvcResultMatchers.jsonPath("$.userPhone").value(phone));
	}
}
```

![image](https://user-images.githubusercontent.com/20104232/70887237-b96a1180-2020-11ea-9d11-a1c4598915ab.png)

## 방법 3. 생성자 사용

생성자의 파라미터 변수명을 요청받을 이름과 같이 해줍니다.

```java
@Getter
public class GetUserModelUsingConstructor {

    private String userName;
    private String userPhone;
    // ...

    public GetUserModelUsingConstructor(String name, String phone, /* ... */) {
        this.userName = name;
        this.userPhone = phone;
        // ...
    }
}
```

### 테스트

```java
@ExtendWith(SpringExtension.class)
@WebMvcTest(UserConstructorController.class)
class UserConstructorControllerTest {

	@Autowired
	MockMvc mockMvc;

	@Test
	public void 생성자_이용_테스트() throws Exception {
		String name = "Eddy";
		String phone = "010xxxxXXXX";

		MultiValueMap<String, String> requestParam = new LinkedMultiValueMap<>();
		requestParam.set("name", name);
		requestParam.set("phone", phone);

		mockMvc
				.perform(
						MockMvcRequestBuilders.get("/users/constructor")
								.params(requestParam)
				)
				.andDo(MockMvcResultHandlers.print())
				.andExpect(MockMvcResultMatchers.status().isOk())
				.andExpect(MockMvcResultMatchers.jsonPath("$.userName").value(name))
				.andExpect(MockMvcResultMatchers.jsonPath("$.userPhone").value(phone));
	}
}
```

![image](https://user-images.githubusercontent.com/20104232/70887647-c9cebc00-2021-11ea-941b-bf3099a74013.png)

## Tip!

setter와 생성자를 이용할 때 모델의 멤버 변수에 유효성 검사 어노테이션을 사용하고 컨트롤러의 파라미터에 `@Valid` 어노테이션을 사용하면 유효성 검사를 할 수 있습니다.

```java
@Getter
public class GetUserModelUsingSetter {

    @NotEmpty // 유효성 검사
    private String userName;
    private String userPhone;
    // ...

    public void setName(String name) {
        this.userName = name;
    }

    public void setPhone(String phone) {
        this.userPhone = phone;
    }

    // ... setter
}
```

```java
@GetMapping("/users")
public Page<User> getUsers(@Valid GetUserModelUsingSetter getUserModel) {
    // ...
}
```