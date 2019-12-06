---
title: 'JPA entity listener 만들기'
description: '엔티티의 변화를 감지해 같은 테이블과 다른 테이블 모두 데이터를 조작하는 entity listener 를 만들어 적용한 경험을 간단한 예와 소개합니다.'
excerpt: '엔티티의 변화를 감지해 같은 테이블과 다른 테이블 모두 데이터를 조작하는 entity listener 를 만들어 적용한 경험을 간단한 예와 소개합니다.'
time: 2019-09-06 18:17:00
categories: [java, spring, spring boot, jpa, hibernate]
header:
    teaser: 'https://user-images.githubusercontent.com/20104232/64585171-96511580-d3d2-11e9-947d-8f1e98e46100.png'
---

![image](https://user-images.githubusercontent.com/20104232/64585171-96511580-d3d2-11e9-947d-8f1e98e46100.png)

엔티티의 변화를 감지해 같은 테이블과 다른 테이블 모두 데이터를 조작하는 entity listener 를 만들어 적용한 경험을 간단한 예와 소개합니다.

## 엔티티 리스너의 필요

특정 컬럼의 변화를 감지해 데이터를 변화시키고 로그 테이블에 저장해야 할 필요가 있었습니다.

기존에는 변화를 감지해야할 컬럼이 있을 때 직접 다른 컬럼을 업데이트하고 로그 테이블에 데이터를 추가하는 코드가 추가적으로 존재했습니다.

해당 컬럼이 수정되는 곳은 갈수록 늘어가는 상황에서 똑같은 로직 반복으로 인한 비효율과 인간 실수의 가능성이라는 엄청난 위험성이 존재했습니다.

이 상황을 개선하고자 엔티티 리스너를 사용하기로 했습니다.

## 엔티티 리스너 만들어 보기

문제 해결을 위해 적용한 엔티티 리스너와 비슷한 역할을 하는 엔티티 리스너를 만들어 보겠습니다.  
JPA, Hibernate, H2를 사용한 예입니다.

### 요구사항

고객이 비밀번호를 변경하면 비밀번호를 변경한 시간을 저장하고 변경 전의 비밀번호와 변경할 비밀번호를 저장해야 합니다.

### 테이블 설계

고객의 정보를 저장하는 테이블 `member` 테이블에는 `member_id` , `username` , `password` , `created_at` , `updated_at` , `is_deleted` , `password_updated_at` 이라는 컬럼이 존재하고, 

이전 비밀번호와 동일한 비밀번호로 변경하는 것을 방지하기 위해 비밀번호가 변경된 히스토리를 보관하기 위해 `history_id` , `member_id` , `before_password` , `after_password` , `created_at` 컬럼을 가지는 `member_password_history` 테이블이 필요합니다.

( `created_at` , `updated_at` 컬럼은 JPA의 `AuditingEntityListener` 를 이용해 저장합니다.)

### 서비스 로직

요구사항을 충족하기 위해 고객이 비밀번호를 변경하면 `member` 테이블의 `password_updated_at` 컬럼이 현재시간으로 수정되고 `member_password_history` 에 변경 전의 비밀번호와 변경할 비밀번호를 저장해야 합니다.

엔티티 리스너를 사용하지 않았다면 아래처럼 멤버 엔티티와 히스토리 엔티티의 데이터를 각각 입력하고 저장해주어야 합니다.

{% highlight java linenos %}
// 비밀번호 변경
member.setPassword("NEW_PASSWORD"); 
member.setPasswordUpdatedAt(LocalDateTime.now()); 
memberRepository.save(member); 

// 비밀번호 변경 히스토리 저장
MemberPasswordHistory memberPasswordHistory = new MemberPasswordHistory(); 
memberPasswordHistory.setMemberId(member.getId()); 
memberPasswordHistory.setBeforePassword("MY_PASSWORD"); 
memberPasswordHistory.setAfterPassword("NEW_PASSWORD"); 
memberPasswordHistoryRepository.save(memberPasswordHistory); 
{% endhighlight %}

`password` 컬럼의 변화를 감지해 위의 로직을 자동으로 수행하는 `PasswordUpdateListener` 를 만들어 보겠습니다.

* 멤버 엔티티 클래스( `Member` )에 비밀번호 컬럼이 변경될 때 이전 값을 임시로 저장할 수 있도록 변수를 추가합니다.

{% highlight java linenos %}
@Transient
private String prePassword; 
{% endhighlight %}

* 리스너 클래스( `PasswordUpdateListener` )에 영속성 컨텍스트에서 조회될 때 이전 비밀번호를 1번 과정에서 만든 `prePassword` 에 저장해둡니다.

> 엔티티 리스너의 이벤트 종류는 [Entity listeners and Callback methods](https://docs.jboss.org/hibernate/core/4.0/hem/en-US/html/listeners.html)에서 확인하실 수 있습니다.

{% highlight java linenos %}
@PostLoad
public void postLoad(Member member) {
    member.setPrePassword(member.getPassword());
}
{% endhighlight %}

* 멤버 엔티티가 저장될 때 `prePassword`를 이용해 `password` 의 변화를 확인하고 비밀번호 변경 시간을 저장합니다.
  `updatePassword` 메소드를 만들어 비밀번호 변경 시간을 수정해줍니다.

{% highlight java linenos %}
@PrePersist
public void prePersist(Member member) {
    if (member.getPrePassword() == null) {
        updatePassword();
    }
}

@PreUpdate
public void preUpdate(Member member) {
    if (!member.getPrePassword().equals(member.getPassword())) {
        updatePassword();
    }
}

void updatePassword(Member member) {
    LocalDateTime now = LocalDateTime.now();
    member.setPasswordUpdatedAt(now);
}
{% endhighlight %}

* 멤버 엔티티 클래스에서 리스너를 사용하도록 추가해줍니다.

{% highlight java linenos %}
@EntityListeners(value = {AuditingEntityListener.class, PasswordUpdateListener.class})
{% endhighlight %}

아주 쉽게 비밀번호를 변경하면 변경 시간을 저장할 수 있는 리스너를 만들었지만 다른 테이블( `member_password_history` )에 저장하기 위한 과정이 또 필요합니다.

* 리스너 클래스에서는 의존성 주입이 되지 않기 때문에 빈을 반환하는 static 메소드를 가진 `BeanUtils` 컴포넌트 클래스가 필요합니다.

{% highlight java linenos %}
@Component
public class BeanUtils implements ApplicationContextAware {

    private static ApplicationContext applicationContext;
    
    @Override
    public void setApplicationContext(ApplicationContext applicationContext) throws BeansException {
        BeanUtils.applicationContext = applicationContext;
    }
    
    public static <T> T getBean(Class<T> cls) {
        return applicationContext.getBean(cls);
    }
    
}
{% endhighlight %}

* `updatePassword` 메소드에 히스토리를 저장하는 로직을 추가합니다.
  리스너 클래스에서 `BeanUtils.getBean()` 을 통해 `MemberPasswordHistoryRepository` 빈을 사용할 수 있습니다.

{% highlight java linenos %}
void updatePassword(Member member) {
    LocalDateTime now = LocalDateTime.now();
    member.setPasswordUpdatedAt(now);
    
    MemberPasswordHistoryRepository memberPasswordHistoryRepository =
            BeanUtils.getBean(MemberPasswordHistoryRepository.class);
    
    MemberPasswordHistory memberPasswordHistory = MemberPasswordHistory.builder()
            .member(member)
            .beforePassword(member.getPrePassword())
            .afterPassword(member.getPassword())
            .build();
    memberPasswordHistoryRepository.save(memberPasswordHistory);
}
{% endhighlight %}

### 테스트

테스트 클래스를 작성해서 원하는대로 동작하는지 확인해보겠습니다.

{% highlight java linenos %}
@RunWith(value = SpringRunner.class)
@SpringBootTest
public class MemberTest {

    @Autowired
    MemberRepository memberRepository;

    @Autowired
    MemberPasswordHistoryRepository memberPasswordHistoryRepository;

    private Member member;

    @Before
    public void setUp() {
        member = memberRepository.save(
                Member.builder()
                        .username("test_user")
                        .password("p@ssw0rd")
                        .build()
        );
    }

    @Test
    public void 비밀번호_변경_테스트() {
        LocalDateTime dateTimeBeforeUpdatePassword = member.getPasswordUpdatedAt();
        member.setPassword("updateP@ssw0rd");
        member = memberRepository.save(member);

        List<MemberPasswordHistory> memberPasswordHistoryList = memberPasswordHistoryRepository.findAll();
        memberPasswordHistoryList.forEach(System.out::println);

        Assertions.assertThat(dateTimeBeforeUpdatePassword).isBefore(member.getPasswordUpdatedAt());
        Assertions.assertThat(memberPasswordHistoryList).hasSize(2);
    }

}
{% endhighlight %}

![image](https://user-images.githubusercontent.com/20104232/64472593-c498e000-d19b-11e9-8984-a0d110accd2e.png)

1. 비밀번호가 변경전과 변경 후의 시간을 비교하는 테스트와 히스토리 테이블에 저장된 기록을 확인하는 테스트가 성공했고  
2. 히스토리의 출력 내용에도 변경 전과 변경 후의 비밀번호가 잘 저장되어 있는 것을 확인할 수 있습니다.

---

같은 테이블의 데이터 조작은 물론 다른 테이블에 추가적인 행위를 하는 리스너를 소개했습니다.  
엔티티 리스너를 통해 동일한 로직이 여러 곳에 존재하는 코드를 걷어내고 사람의 실수를 방지할 수 있게 되었습니다.

이 글에서 사용한 예는 [GitHub](https://github.com/bum752/spring-boot-jpa-entity-listener)에 있으니 참고해주세요!