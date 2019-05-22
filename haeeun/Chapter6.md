## Chapter6. AOP
### 6.1 트랜잭션 코드의 분리
트랜잭션의 경계는 비즈니스 로직 전후에 설정돼야 한다.
---
#### 6.1.1 메소드 분리
upgradeLevels() 에는 트랜잭션 경계설정과 비즈니스 로직이 공존한다.  
하지만 트랜잭션 경계설정 코드와 비즈니스로직 코드가 구분되어 있고 두 코드 간에 서로 주고 받는 코드가 없으므로 독립적인 코드라고 할 수 있다.  
따라서 두 개의 메소드로 분리할 수 있다. 

#### 6.1.2 DI를 이용한 클래스의 분리
UserService에 트랜잭션을 담당하는 기술적인 코드가 아직 남아있다. 이 문제를 해결하기 위해서는 트랜잭션 코드를 클래스 밖으로 뽑아내면 된다.

**DI 적용을 이용한 트랜잭션 분리**  
다른 클래스나 모듈에서 UserService를 호출해서 사용한다면 UserService 클래스를 직접 참조하게 된다. 트랜잭션 코드를 UserService 밖으로 빼버리면 UserService 클래스를 직접 사용하는 클라이언트 코드에서는 트랜잭션 기능이 빠진 UserService를 사용하게 된다. _(구현 클래스를 직접 참조하는 경우의 단점)_  

DI의 기본 아이디어
> 실제 사용할 오브젝트의 클래스 정체는 감춘 채 인터페이스를 통해 간접으로 접근하는 것
> 따라서 구현 클래스는 외부에서 변경 가능.

UserService를 인터페이스로 만들고 기존 코드는 UserService 인터페이스의 구현 클래스를 만들면 클라이언트와 결합이 약해지고 직접 구현 클래스에 의존하지 않기 때문에 확장이 가능해진다.  

한 번에 두 개의 UserService 인터페이스 구현 클래스를 동시에 이용하는 것도 가능하다. 트랜잭션의 경계설정은 UserServiceTx에서, UserServiceImpl에는 실제적인 로직 처리를 위임한다. 그 위임을 위한 호출 작업 이전과 이후에 트랜잭션 경계를 설정해주면 된다.  

**UserService 인터페이스 도입**  
UserServiceImpl은 기존 UserService 클래스의 내용을 대부분 그대로 유지하되, 트랜잭션과 관련된 코드는 제거한다.  

**분리된 트랜잭션 기능**  
UserServiceTx는 UserService를 구현하게 만든다. 그리고 같은 인터페이스를 구현한 다른 오브젝트에게 작업을 위임하게 만들면 되다.  
그 후 UserService에 transactionManager라는 이름의 빈으로 등록된 트랜잭션 매니저를 DI로 받아뒀다가 트랜잭션 안에서 동작하도록 만들어줘야 하는 메소드 호출의 전 후에 트랜잭션 경계설정 API를 사용하면 된다.  
```java
public class UserServiceTx implements UserService{
    // UserService를 구현한 다른 오브젝트를 DI 받는다.
    UserService userService;
    PlatformTransactionManager transactionManager;

    public void setTransactionManager(PlaformTransactionManager transactionManager){
        this.trasactionManager = transactionManager;
    }

    public void setUserService(UserService userService) {
        this.userService = userService;
    }

    // DI 받은 UserService 오브젝트에 모든 기능 위임 
    public void add(User user){
        userService.add(user);
    }

    public void upgradeLevels() {
        TransactionStatus status = this.transactionManager.getTransaction(new DefaultTransactionDefinition());
        try{
            userService.upgradeLevels();
            this.transactionManager.commit(status);
        } catch(RuntimeException e) {
            this.transactionManager.rollback(status);
            throw e;
        }
    }
}
```

**트랜잭션 적용을 위한 DI 설정**  
기존에 userService 빈이 의존하고 있던 transanctionManager는 UserServiceTx의 빈이, userDao와 mailSender는 UserServiceImpl 빈이 각각 의존하도록 프로퍼티 정보를 분리한다.  

클라이언트는 UserServiceTx 빈을 호출해서 사용하도록 만들어야 하고, userService 빈은 UserServiceImpl 클래스로 정의되는 userServiceImpl인 빈을 DI 하게 만든다.  

**트랜잭션 분리에 따른 테스트 수정**
기존에는 UserService 클래스 타입의 빈을 @Autowired로 가져다 사용했다.  
인터페이스도 @Autowired로 가져오는 데는 문제가 없지만, @Autowired는 타입이 일치하는 빈을 찾아주기 때문에 UserService라는 인터페이스 타입을 가진 두개의 빈이 존재하기 때문에 문제가 발생한다.  
타입으로 하나의 빈을 결정할 수 없는 경우에는 필드 이름을 이용해 빈을 찾는다.
```
@Autowired
UserService userService;
```

UserServiceTest는  UserServiceImpl 클래스로 정의된 빈도 필요하다.  
MailSender Mock 오브젝트를 이용한 테스트에서는 테스트에서 직접 MailSender를 DI 해줘야 하는데 MailSender를 DI 해줄 대상을 구체적으로 알고 있어야 하기 때문이다.  
```java
@Test
public void upgradeLevels() throws Exception {
    MockMailSender mockMailSender = new MockMailSender();
    userServicelmpl.setMailSender(mockMailSender);
} 
```

**트랜잭션 경계설정 코드 분리의 장점**  
1. 비즈니스 로직을 담당하고 있는 UserServiceImpl의 코드를 작성할 때는 트랜잭션 같은 기술적인 내용은 신경쓰지 않아도 된다.  
트랜잭션은 DI를 이용해 UserServiceTx와 같은 트랜잭션 기능을 가진 오브젝트가 먼저 실행되도록 하면 된다. 
2. 비즈니스 로직에 대한 테스트를 쉽게 만들 수 있다.

<br>

### 6.2 고립된 단위 테스트
---
#### 6.2.1 복잡한 의존관계 속의 테스트
UserService의 구현 클래스들이 동작하려면 세 가지 타입의 의존 오브젝트가 필요하다. (UserDao, MailSender, PlatfromTransactionManager)  
따라서 세가지 의존관계를 갖는 오브젝트들이 테스트가 진행되는 동안 같이 실행되기 때문에 의존관계를 따라 등장하는 오브젝트와 서비스, 환경 등이 모두 합쳐져 테스트 대상이 된다.  
#### 6.2.2 테스트 대상 오브젝트 고립시키기
**테스트를 위한 UserServiceImpl 고립**  
트랜잭션 코드를 독립시켰기 때문에 사용자 관리 로직을 담을 UserServiceImpl은 PlatformTransactionManager에 더 이상 의존하지 않는다.  

UserDao는 테스트 대상의 코드가 정상적으로 수행되도록 도와주는 스텁인 동시에 부가적인 검증 기능까지 가진 목 오브젝트이다. upgradeLevels()의 테스트 결과를 검증할 방법이 필요하기 때문이다.  
upgradeLevels()는 void 형이기 때문에 검증이 불가능하고 DB를 직접 확인할 수밖에 없다.  

그런데 UserServiceImpl은 아무리 그 기능이 수행되도 결과가 DB에도 남지 않는다. 이럴 땐 테스트 대상인  UserServiceImpl과 UserDao에 어떤 요청을 했는지 확인하는 작업이 필요하다.  

**고립된 단위 테스트 활용**  
테스트 작업을 분류해보면 UserService의 upgradeLevles() 메소드가 실행되는 동안에 사용하는 오브젝트가 테스트의 목적에 맞게 동작하도록 준비한다. (DB, MailSender 목 오브젝트 DI)  
실제 테스트 대상인 userService의 메소를 실행한 다음은 테스트 대상 코드를 실행한 후에 결과를 확인하는 작업이다. (DB 확인, 목 오브젝트를 통해 메소드 실행 중 메일 발송 요청이 나간 적이 있는지만 확인)  

**UserDao 목 오브젝트**  
DB 작업도 목 오브젝트로 만들 수 있다. 목 오브젝트는 기본적으로 스텁과 같은 방식으로 테스트 대상을 통해 사용될 때 필요한 기능을 지원해줘야 한다.   
upgradeLevels()가 실행되는 중에 UserDao와 어떤 정보를 주고받는지 알아야 한다.  
1. userDao.getAll() 
: 레벨 업그레이드 후보가 될 사용자의 목록을 받아온다. 
=> DB에서 읽어온 것처럼 미리 준비된 사용자 목록 제공

2. userDao.update(user)
: 리턴값이 없으므로 빈 메소드로 만들어도 된다.
하지만 레벨을 변경하는 부분을 검증할 수 있는 기능이므로 중요한 기능이다. 
업그레이드를 통해 레벨이 변경된 사용자는 DB에 반영되도록 userDao의 update()에 전달돼야 한다.  
따라서 getAll()에 대해서는 스텁으로, update()에 대해서는 목 오브젝트로서 동작하는 UserDao 타입의 테스트 대역이 필요하다. (MockUserDao)  

MockUserDao에는 두개의 User 타입 리스트를 정의해둔다. (생성자를 통해 전달받은 사용자 목록을 저장해뒀다가 getAll() 메소드가 호출되면 돌려주는 용도 / update() 메소드를 실행하면서 넘겨준 업그레이드 대상 User 오브젝트를 저장해뒀다가 검증을 위해 돌려주기 위한 용도)  

_MockUserDao를 사용해서 만든 고립된 테스트_
먼저 테스트하고 싶은 로직을 담은 클래스인 UserServiceImpl의 오브젝트를 직접 생성하고 MockUserDao 오브젝트를 사용하도록 수동 DI 해주면 된다.  
UserServiceImpl은 UserDao의 update()를 이용해 몇 명의 사용자 정보를 DB에 수정하려고 했는지, 사용자드링 누구인지, 어떤 레벨로 변경됐는지 확인하면 된다.  
먼저 레벨 업그레이드가 두 명에게만 일어났는지 update() 메소드를 호출해서 변경을 시도한 것만 확인하면 된다.  
MockUserDao에는 update() 가 호출될때마다 저장해 둔 사용자 목록이 있고, 목록의 수가 2이면 된다. 순서에 따라 업그레이드 된 사용자 아이디와 바뀐 레벨을 확인하면 된다.  

**테스트 수행 성능의 향상**  
UserServiceImpl과 테스트를 도와주는 두 개의 목 오브젝트 외에는 사용자 관리 로직을 검증하는 데 직접적으로 필요하지 않은 의존 오브젝트와 서비스를 모두 제거했기 때문에 빠른 테스트가 가능하다.  

#### 6.2.3 단위 테스트와 통합 테스트  
단위 테스트라는 용어를 사용할 때는 그 의미를 명확히 할 필요가 있다.  
(`단위 테스트`: 테스트 대상 글래스를 목 오브젝트 등의 테스트 대역을 이용해 의존 오브젝트나 외부의 리소스를 사용하지 않도록 고립시켜 테스트 하는 것으로 정의)  
<-> (`통합 테스트`: 두 개 이상의 성격이나 계층이 다른 오브젝트가 연동하도록 만들어 테스트하거나, 외부의 DB나 파일, 서비스 등의 리소스가 참여하는 테스트)


#### 6.2.4 목 프레임워크
**Mockito 프레임워크**  
목 오브젝트를 편리하게 작성하도록 도와주는 목 오브젝트 지원 프레임워크 중 하나.  
목 프레임워크는 목 클래스를 일일이 준비할 필요 없이 간단한 메소드 호출만드로 다이내믹하게 특정 인터페이스를 구현한 테스트 용 목 오브젝트를 만들 수 있다.  
``` java
UserDao mockuserDao = mock(UserDao.class);
```

getAll() 메소드가 불려올 때 사용자 목록을 리턴하도록 스텁 기능을 추가해야 한다.  
```java
when(mockUserDao.getAll().thenReturn(this.users));
```
mockUserDao.getAll()이 호출됐을 때, users 리스트를 리턴해주라는 선언   

update() 호출이 있는지 검증
```java
verify(mockUserDao, times(2)).update(andy(User.class));
```

* 목 오브젝트 사용하는 방법
    - 인터페이스를 이용해 목 오브젝트를 만든다.
    - 목 오브젝트가 리턴할 값이 있으면 지정해준다. 메소드가 호출되면 예외를 강제로 던지게 만들 수도 있다.
    - 테스트 대상 오브젝트에 DI해서 목 오브젝트가 테스트 중에 사용되도록 만든다.
    - 테스트 대상 오브젝트를 사용한 후에 목 오브젝트의 특정 메소드가 호출됐는지, 어떤 값을 가지고 몇 번 호출 됐는지 검증

```java
@Test
public void mockUpgradeLevels() throws Exception {
    UserServicelmpl userServicelmpl =new UserServicelmpl();
    // UserDao의 목 오브젝트를 생성하고 getAll()이 호출됐을 때의 리턴 값을 설정해준 뒤 테스트 대상에 DI
    UserDao mockUserDao = mock(UserDao.class);
    when(mockUserDao.getAll()).thenReturn(this.users); 
    userServicelmpl.setUserDao(mockUserDao);

    MailSender mockMailSender = mock(MailSender.class);
    userServiceImpl.setMailSender(mockMailSender);

    userServiceImpl.upgradeLevels();

    // 목 오브젝트가 제공하는 검증 기능을 통해 어떤 메소드가 몇 번 호출 됐는지, 파라미터가 무엇인지 확인
    // times()는 메소드 호출 횟수 검증, any()는 파라미터 내용 무시
    verify(mockUserDao, times(2)).update(any(User.class));
    verify(mockUserDao, times(2)).update(any(User.class));
    verify(mockUserDao).update(users.get(1));
    assertThat(users.get(1).getLevel(), is(Level.SILVER));
    verify(mockUserDao).update(users.get(3));
    assertThat(users.get(3).getLevel(), is(Level.GOLD));

    ArgumentCaptor<SimpleMailMessage> mailMessageArg = ArgumentCaptor.forClass(SimpleMailMessage.class);
    verify(mockMailSender, times(2)).send(mailMessageArg.capture());
    List<SimpleMailMessage> mailMessages = mailMessageArg.getAllValues();
    assertThat(mailMessages.get(0).getTo()[0], is(users.get(1).getEmail()));
    assertThat(mailMessages.get(1).getTo()[0], is(users.get(3).getEmail()));
}
```

### 6.3 다이내믹 프록시와 팩토리 빈
---
#### 6.3.1 프록시와 프록시 패턴, 데코레이터 패턴  
트랜잭션 기능은 사용자 관리 비즈니스 로직과는 성격이 다르기 때문에 아예 적용 사실 자체를 밖으로 분리할 수 있다.  
이렇게 분리된 부가기능을 담은 클래스는 부가 기능 외의 나머지 모든 기능은 원래 핵심 기능을 가진 클래스로 위임해줘야 한다. **부가기능이 핵심기능을 사용하는 구조**  

클라이언트는 인터페이스를 통해서만 핵심기능을 사용하게 하고, 부가기능 자신도 같은 인터페이스를 구현한 뒤에 자신이 그 사이에 끼어들어야 한다.  

부가기능 코드에서는 핵심기능으로 요청을 위임해주는 과정에서 자신이 가진 부가적인 기능을 적용해줄 수 있다. (비즈니스 로직 코드에 트랜잭션 기능 부여)

위와 같이 자신이 클라이언트가 사용하려고 하는 실제 대상인 것처럼 위장해서 클라이언트의 요청을 받아주는 것을 대리자, 대리인과 같은 역할을 한다고 해서 `**프록시**`라고 부른다. 프록시를 통해 최종적으로 요청을 위임받아 처리하는 실제 오브젝트는 `타깃`, 또는 `실체`라고 부른다.  

* 프록시의 특징
    > 타깃과 같은 인터페이스를 구현했다는 것과 프록시가 타깃을 제어할 수 있다는 위치에 있다.

* 사용 목적에 따른 구분
    1. 클라이언트가 타깃에 접근하는 방법 제어
    2. 타깃에 부가적인 기능 부여  


**데코레이터 패턴**
> 타깃에 부가적인 기능을 런타임 시 다이내믹하게 부여해주기 위해 프록시를 사용하는 패턴

컴파일 시점, 즉 코드 상에서는 어떤 방법과 순서로 프록시와 타깃이 연결되어 사용되는지 정해져 있기 때문에 데코레이터패턴에서는 프록시가 꼭 한 개로 제한되지 않는다.   
데코레이터 패턴에서는 같은 인터페이스를 구현한 타겟과 여러 개의 프록시를 사용할 수 있다. 프록시가 여러 개인 만큼 순서를 정해서 단계적으로 위임하는 구조로 만들면 된다.  

프록시로 동작하는 각 데코레이터는 위임하는 대상에도 인터페이스로 접근하기 때무에 자신이 최종 타깃으로 위임하는지, 다음 단게의 데코레이터 프록시로 위임하는지 알지 못한다. 따라서 데코레이터의 다음 위임 대상은 인터페이스로 선언하고 생성자나 수정자 메소드를 통해 위임 대상을 외부에서 런타임시에 주입받을 수 있도록 만들어야 한다. 

인터페이스를 통한 데코레이터 정의와 런타임 시의 다이내믹한 구성 방법은 스프링의 DI를 이용하여 데코레이터 빈의 프로퍼티로 같은 인터페이스를 구현한 다른 데코레이터 또는 타깃 빈을 설정하면 된다.  

데코레이터 패턴은 타깃의 코드를 손대지 않고, 클라이언트가 호출하는 방법도 변경하지 않은 채로 새로운 기능을 추가할 때 유용하다.  

**프록시 패턴**
> * 프록시를 사용하는 방법 중에서 타깃에 대한 접근 방법을 제어하려는 목적을 가진 경우 
> * 타깃의 기능 자체에는 관여하지 않으면서 접근하는 방법을 제어해주는 프록시를 이용

프록시 패턴의 프록시는 타깃의 기능을 확장하거나 추가하지 않는 대신 클라이언트가 타깃에 접근하는 방식을 변경해준다.  
* 프록시를 사용하는 경우
    1. 클라이언트에게 타깃에 대한 레퍼런스를 넘겨야 하는 경우  
    : 실제 타깃 오브젝트를 만드는 대신 프록시를 넘겨 준다.  
    2. 원격 오브젝트를 이용하는 경우  
    : 다른 서버에 존재하는 오브젝트를 사용해야 한다면, 원격 오브젝트에 대한 프록시를 만들어두고, 클라이언트는 로컬에 존재하는 오브젝트를 쓰는 것처럼 프록시를 사용하게 할 수 있다.  
    3. 특별한 상황에서 타깃에 대한 접근권한을 제어하기 위해 사용  
    : 예를 들어 수정 가능한 오브젝트가 있는데, 특정 레이어로 넘어가서는 읽기 전용으로만 동작하게 강제해야 할 때 오브젝트의 프록시를 만들어서 사용할 수 있다.  

프록시와 데코레이터는 유사한 구조이지만, 프록시는 코드에서 자신이 만들거나 접근할 타깃 클래스 정보를 알고 있는 경우가 많다.  

#### 6.3.2 다이내믹 프록시
**프록시의 구성과 프록시 작성의 문제점**  
* 프록시의 기능
    - 타깃과 같은 메소드를 구현하고 있다가 메소드가 호출되면 타깃 오브젝트로 위임한다.
    - 지정된 요청에 대해서는 부가기능을 수행한다. 

* 프록시의 역할
    - 위임
    - 부가작업

* 프록시를 만들기 번거로운 이유
    - 타깃의 인터페이스를 구현하고 위임하는 코드를 작성하기 번거롭다.
    - 부가기능 코드가 중복될 가능성이 많다.  

**리플렉션**  
다이내믹 프록시는 리플렉션 기능을 이용해서 프록시를 만들어준다. 리플랙션은 자바의 코드 자체를 추상화해서 접근하도록 만든 것이다.  
클래스 오브젝트를 이용하면 클래스 코드에 대한 메타정보를 가져오거나 오브젝트를 조작할 수 있다. (java.lang.reflect)

**프록시 클래스**
1. Hello 인터페이스 정의
2. 인터페이스를 구현한 타깃 클래스
3. Hello 인터페이스를 통해 HelloTarget 오브젝트를 사용하는 클라이언트(테스트)  
4. Hello 인터페이스를 구현한 프록시 : 데코레이터 패턴 적용해 부가기능 추가  
    -> HelloUppercase  
Hello 인터페이스 구현 메소드에서는 타깃 오브젝트의 메소드를 호출한 뒤 결과를 대문자로 바꿔주는 부가기능을 적용하고 리턴한다.  

이 프록시는 인터페이스의 모든 메소드를 구현해 위임하도록 코드를 만들어야 하며, 부가 기능인 리턴 값을 대문자로 바꾸는 기능이 모든 메소드에 중볻돼서 나타난다는 문제점을 갖고 있다.

**다이내믹 프록시 적용**  
다이내믹 프록시는 프록시 팩토리에 의해 런타임 시 다이내믹하게 만들어지는 오브젝트다.  
다이내믹 프록시 오브젝트는 타깃의 인터페이스와 같은 타입으로 만들어진다.  
클라이언트는 다이내믹 프록시 오브젝트를 타깃 인터페이스를 통해 사용할 수 있다.  
프록시 팩토리에게 인터페이스 정보만 제공하면 해당 인터페이스를 구현한 클래스의 오브젝트를 자동으로 만들어 준다.   

부가기능은 프록시 오브젝트와 독립적으로 InvocationHandler를 구현한 오브젝트에 담는다. invoke()는 리플렉션의 Method 인터페이스를 파라미터로 받는다.  
다이내믹 프록시 오브젝트는 클라이언트의 모든 요청을 리플렉션 정보로 변환해서 InvocationHandler 구현 오브젝트의 invoke() 메소드로 넘긴다.  
타깃 인터페이스의 모든 메소드 요청이 하나의 메소드로 집중되기 때문에 중복되는 기능을 효과적으로 제공할 수 있다.  


다이내믹 프록시로부터 요청을 전달받으려면 **InvocationHandler**를 구현해야 한다. 
* 메소드는 invoke() 하나뿐이다. 다이내믹 프록시가 클라이언트로부터 받는 모든 요청은 invoke()로 전달된다. 
* 다이내믹 프록시를 통해 요청이 전달되면 리플렉션 API를 이용해 타깃 오브젝트의 메소드를 호출한다. 
* 타깃 오브젝트는 생성자를 통해 미리 전달받아 둔다.  
* 타깃 오브젝트의 메소드 호출이 끝나면 프록시가 제공하려는 부가기능을 수행하고 리턴한다.  

InvocationHandler를 사용하고 Hello 인터페이스를 구현하는 **프록시 생성**  
* 다이내믹 프록시 생성은 Proxy 클래스의 newProxyInstance() 스태택 팩토리 메소드 이용 
* newProxyInstance()에 의해 만들어지는 다이내믹 프록시 오브젝트는 파라미터로 제공한 Hello 인터페이스를 구현한 클래스의 오브젝트이기 때문에 Hello 타입으로 캐스팅이 가능하다. 

**다이내믹 프록시의 확장**
InvocationHandler 방식의 또 한 가지 장점은 타깃의 종류에 상관없이도 적용이 가능하다. 리플렉션의 Method 인터페이스를 이용해 타깃의 메소드를 호출하므로 Hello 타입의 타깃으로 제한할 필요도 없다.  
InvocationHandler는 단일 메소드에서 모든 요청을 처리하기 때문에 어던 메소드에 어떤 기능을 적용할지 선택하는 과정이 필요할 수도 있다.   

#### 6.3.3 다이내믹 프록시를 이용한 트랜잭션 부가기능  
UserServiceTx는 트랜잭션이 필요한 메소드마다 트랜잭션 처리가 중복되는데 다이내믹 프록시와 연동하면 InvocationHandler 한 개만 정의해도 된다.  

**트랜잭션 InvocationHandler**  
요청을 위임할 타깃을 DI로 제공받는다. 타깃을 저장할 변수는 Object로 선언한다. 따라서 UserServiceImpl 외에 트랜잭션 적용이 필요한 어떤 타깃 오브젝트에도 적용할 수 있다. 트랜잭션 추상화 인터페이스를 DI 받고 트랜잭션을 적용할 메소드 이름의 패턴을 DI 받는다. (ex. get : get으로 시작하는 모든 메소드)  
InvocationHandler의 invoke() 구현은 적용할 대상을 선별해서 DI 받은 이름과 패턴으로 시작되는 이름을 가진 메소드인지 확인한다.  
차이점은 롤백을 적용하기 위해 RuntimeExcetpion이 아닌  InvocationTargetException을 잡도록 해야 한다.   

**TransactionHandler와 다이내믹 프록시를 이용하는 테스트**  
UserServiceTx 오브젝트 대신 TransactionHandler를 만들고 타깃 오브젝트와 트랜잭션 매니저, 메소드 패턴을 주입해준다.  

#### 6.3.4 다이내믹 프록시를 위한 팩토리 빈  
DI 대상이 되는 다이내믹 프록시 오브젝트는 일반적인 스프링 빈으로 등록할 수 없다.  
스프링은 지정된 클래스 이름을 가지고 리플렉션을 이용해 해당 클래스의 오브젝트를 만든다.  
스프링은 내부적으로 리플렉션 API를 이용해서 빈 정의에 나오는 클래스 이름을 가지고 빈 오브젝트를 생성한다. 하지만 다이내믹 프록시 오브젝트는 이런 식으로 프록시 오브젝트가 생성되지 않는다. _다이내믹 프록시는 Proxy 클래스의 newProxyInstance()라는 스태틱 팩토리 메소드를 통해서만 만들 수 있다._  

**팩토리 빈**  
> 스프링을 대신해서 오브젝트의 생성로직을 담당하도록 만들어진 특별한 빈 
스프링의 FactoryBean이라는 인터페이스를 구현하는 방법이 가장 간단하다.  




#### 6.3.5 프록시 팩토리 빈 방식의 장점과 한계