## 5장 서비스 추상화
### 5.1 사용자 레벨 관리 기능 추가
---
인터넷 서비스의 사용자 관리 기능에서 구현해야 할 비즈니스 로직  
* 사용자의 레벨은 BASIC, SILVER, GOLD 세가지 중 하나다.  
* 사용자가 처음 가입하면 BASIC 레벨, 이후 활동에 따라 업그레이드 될 수 있다.
* 가입 후 50회 이상 로그인 하면 BASIC에서 SILVER 레벨이 된다.
* SILVER 레벨이면서 30번 이상 추천을 받으면 GOLD 레벨이 된다.
* 사용자 레벨의 벼녕 작업은 일정한 주기를 가지고 일괄적으로 진행된다. 변경 작업 전에는 조건을 충족하더라도 레벨의 변경이 일어나지 않는다.
#### 5.1.1 필드 추가
##### 1. Level 이늄  
User 클래스에 사용자의 레벨을 저장할 필드 추가  
상수 값을 정해놓고 int 타입으로 레벨을 사용  
하지만 이러한 경우 level의 타입이 int이기 때문에 다른 종류의 정보를 넣어도 컴파일러가 체크해주지 못하고, 범위를 벗어나는 값을 넣을 위험도 있다. 따라서 숫자 타입을 직접 사용하는 것보다 **enum**을 사용하는 것이 안전하고 편리하다.
enum은 내부에는 DB에 저장할 int 타입의 값을 갖고 있지만, 겉으로는 Level 타입의 오브젝트이기 때문에 안전하게 사용할 수 있다.
```java
...
public enum Level {
    BASIC(1), SILVER(2), GOLD(3);   // 세 개의 enum 오브젝트 정의

    private final int value;

    Level(int value) { this.value = value; }
    public int intValue() { return value; }

    public static Level valueOf(int value) {
        switch(value) {
            case 1: return BASIC;
            case 2: return SILVER;
            case 3: return GOLD;
            default: throw new AssertionError("Unknown value: "+ value);
        }
    }
}
```

##### 2. User 필드 추가
Level 타입의 변수를 User 클래스에 추가  
로그인 횟수와 추천수, 각가의 접근자와 수정자 메소드도 추가

##### 3. UserDaoTest 테스트 수정
UserDaoJdbc와 테스트에도 필드 추가  
먼저 테스트 픽스처로 만든 user1, user2, user3에 새로 추가된 세 필드의 값을 넣는다.  
이에 맞게 User 클래스 생성자의 파라미터도 추가  
UserDaoTest 테스트에서 두 개의 User 오브젝트 필드 값이 모두 같은지 비교하는 checkSameUser()메소드 수정  
새로운 필드를 비교하는 코드 추가  

##### 4. UserDaoJdbc 수정  
등록을 위한 INSERT 문장이 있는 add() 메소드의 SQL과 각종 조회 작업에 사용되는 User 오브젝트 매핑용 콜백인 userMapper에 추가된 필드를 넣는다.  
Level enum은 오브젝트이므로 DB에 저장될 수 있는 SQL 타입이 아니기 때문에 정수형 값으로 값을 변환해줘야 한다. Level에 미리 만들어둔 intValue() 메소드를 이용해야 한다.  
조회를 할 경우 ResultSet에서는 DB 타입인 int로 level 정보를 가져오므로 Level의 스태틱 메소드인 valueOf()를 이용해 int 타입의 값을 Level 타입의 enum 오브젝트로 만들어서 setLevel()에 넣어주어야 한다.

#### 5.1.2 사용자 수정 기능 추가
##### 수정 기능 테스트 추가
픽스처 오브젝트를 등록하고 id를 제외한 필드의 내용을 바꾼 뒤 update()를 호출한다,  
다시 id로 조회해서 가져온 User 오브젝트와 수정한 픽스처 오브젝트를 비교한다.
##### UserDao와 UserDaoJdbc 수정
UserDao 인터페이스에 update() 추가  
UserDaoJdbc에서 메소드 구현
##### 수정 테스트 보완
update()의 경우 수정할 로우의 내용이 바뀐 것만 확인할 뿐, 수정하지 않아야 할 로우의 내용이 그대로 남아 있는지는 확인해주지 못한다.  
**해결 방법**
1. JdbcTemplate의 update()가 돌려주는 리턴 값을 확인한다.  
JdbcTemplate의 update()는 테이블 내용에 영향을 주는 SQL을 실행하면 영향받은 로우의 개수를 돌려준다.  
따라서 이 값이 1인지 확인하는 코드를 추가해주면 된다. 

2. 테스트를 보강해서 원하는 사용자 외의 정보는 변경되지 않았음을 지접 확인한다.  
사용자를 두 명 등록해 놓고, 그 중 하나만 수정한 뒤 수정된 사용자와 수정하지 않은 사용자의 정보를 모두 확인한다.

#### 5.1.3 UserService.upgradeLevels()
UserDao의 getAll() 메소드로 사용자를 다 가져와서 사용자별로 레벨 업그레이드 작업을 진행하면서 UserDao의 update()를 호출해 DB에 결과를 넣어주면 된다.  
사용자 관리 로직은 따로 만들어야 한다. (UserService)
UserService는 UserDao 인터페이스 타입으로 userDao 빈을 DI 받아 사용하게 만든다.  
UserService는 UserDao의 구현 클래스가 바뀌어도 영향받지 않도록 해야 한다. 따라서 DAO 인터페이스를 사용하고 DI를 적용해야 한다. 
##### UserService 클래스와 빈 등록
```java
public class UserService{
    ...
    UserDao userDao;

    public void setUserDao(UserDao userDao) {
        this.userDao = userDao;
    }
}
```
##### UserServiceTest 테스트 클래스
UserServiceTest 클래스를 추가하고 테스트 대상인 UserService 빈을 제공받을 수 있도록 @Autowired가 붙은 인스턴스 변수로 선언해준다. UserService는 컨테이너가 관리하는 스프링 빈이므로 스프링 테스트 컨텍스트를 통해 주입받을 수 있다. 또 userService 빈이 생성돼서 userService 변수에 주입되는지만 확인하는 테스트 메소드도 필요하다.
``` java
@RunWith(SpringJUnit4ClassRunner.class)
@ContextConfiguration(locations="/test-applicationContext.xml")
puvlic class UserServiceTest {
    @Autowired
    UserSercice userService;
}
```
```java
// userService 빈의 주입을 확인하는 테스트
@Test
public void bean() {
    assertThat(this.userService, is(notNullValue()));
}
```
##### upgradeLevels() 메소드 
모든 사용자 정보를 DAO에서 가져온 후 한 명 씩 레벨 변경 작업을 수행한다. 현재 사용자의 레벨이 변경됐는지 확인할 수 있는 플래그를 선언하고, 플래그를 확인해서 레벨 변경이 있는 경우에만 update()를 이용해 수정 내용을 DB에 반영한다.
```java
public void upgradeLevels() {
    List<User> users = userDao.getAll();
    for(User user : users) {
        Boolean changed = null;
        if(user.getLevel() == Level.BASIC && user.getLogin() >= 50) {
            user.setLevel(Level.SILVER);
            changed = true;
        } else if (user.getLevel() == Level.SILVER && user.getRecommand() > = 30) {
            user.setLevel(Level.GOLD;
            changed = true;
        } else if (user.getLevel() == Level.GOLD) { changed = false; }
        else { chnged = false; }
        if(changed) { userDao.update(user); }   // 레벨 변경이 있는 경우에만 update() 호출
    }
}
```

##### upgradeLevels() 테스트
가능한 모든 조건을 하나씩은 확인해봐야 한다. GOLD를 제외한 나머지 두 가지는 업그레이득가 되는 경우와 아닌 경우가 있을 수 있으므로 최소한 다섯 가지 경우를 확인해야 한다.  
```java
//리스트로 만든 테스트 픽스처
class UserServiceTest {
    ...
    List<User> users;

    @Before
    public void setUp() {
        users = Arrays.asList(
            new User("bumjin", "박범진", "p1", Level.BASIC, 49, 0),
            new User("joytouch" , "강명성", "p2", Level.BASIC , 50 , 0),
            new User("erwins", "신승한", "p3", Level.SILVER, 60, 29),
            new User("madnite1", "이상호", "p4", Level.SILVER, 60, 30), 
            new User("green", "오민규", "p5", Level.GOLD, 100, 100)
        );
    }
}
```
```java
@Test
public void upgradeLevles() {
    userDao.deleteAll();
    for(User user : users) userDao.add(user);

    userService.upgradeLevles();

    checkLevel(users.get(0), Level.BASIC);
    checkLevel(users.get(1), Level.SILVER);
    checkLevel(users.get(2), Level.SILVER);
    checkLevel(users.get(3), Level.GOLD); 
    checkLevel(users.get(4), Level.GOLD);

    private void checkLevel(User user, Level expectedLevel) {
        User userUpdate = userDao.get(user.getId());
        assertThat(userUpdate.getLevel(), is(expectedLevel));
    }
}
```

#### 5.1.4 UserSrvice.add()
처음 가입하는 사용자는 BASIC 레벨이어야 한다는 부분을 구현해야 한다.  
사용자 관리에 대한 비즈니스 로직을 담고 있는 UserService에 이 로직을 추가한다. UserDao의 add()는 사용자 정보를 담은 User 오브젝트를 받아서 DB에 넣어주는 데 충실한 역할을 하면, UserService에도 add()를 만들어 두고 사용자가 등록될 때 적용할 만한 비즈니스 로직을 담당하게 하면 된다.  

먼저, 테스트 케이스의 경우 레벨이 미리 정해진 경우와 레벨이 비어 있는 두 가지 경우에 각각 add()를 호출하고 결과를 확인하도록 만들면된다. 검증하기 위해서는 UserService까 UserDao를 통해 DB에 사용자 정보를 저장하기 때문에 이를 확인해보는 게 가장 확실한 방법이다. 

```java
@Test
public void add() {
    userDao.deleteAll();

    User userWithLevel = users.get(4);  //GOLD 레벨
    //레벨이 비어있는 사용자. 로직에 따라 등록 중에 BASIC 레벨로 설정되어야 함
    User userWithoutLevel = users.get(0);   
    userWithoutLevel.setLevel(null);

    userService.add(userWithLevel);
    userService.add(userWithoutLevel);

    //DB에 저장된 결과 가져와서 확인
    User userWithLevelRead = userDao.get(userWithLevel.getId());
    User userWithoutLevelRead = userDao.get(userWithoutLevel.getId());

    assertThat(userWithLevelRead.getLevel(), is(userWithLevel.getLevel()));
    assertThat(userWithoutLevelRead.getLevel(), is(Level.BASIC));
}
```

``` java
// 사용자 신규 등록 로직을 담은 add()
public void add(User user) {
    if(user.getLevel() == null) user.setLevel(Level.BASIC);
    userDao.add(user);
}
```

#### 5.1.5 코드 개선
##### upgradeLevels() 메소드 코드의 문제점
레벨 개숙가 많아지면 if / elseif / else 블록들이 많아져 코드가 지저분해진다.  
또한 레벨과 업그레이드 조건을 동시에 비교하는 부분도 문제가 될 수 있다.  
조건을 두 단계에 걸쳐서 비교해야 한다. 첫 단계에서는 레벨을 확인하고 각 레벨별로 다시 조건을 판단하는 조건식을 넣어야 한다. 

##### upgradeLevels() 리팩토링
- 레벨을 업그레이드 하는 작업의 기본 흐름만 가지는 upgradeLevel()
```java
public void upgradeLevels() {
    List<User> users = userDao.getAll();
    for(User user : users) {
        if(canUpgradeLevel(user)) {
            upgradeLevel(user);
        }
    }
}
```
- 업그레이드가 가능한지 알려주는 메소드 canUpgradeLevel()
```java
private boolean canUpgradeLevel(User user) {
    Level currentLevel = user.getLevel();
    //switch 문으로 레벨을 구분하고 각 레벨에 대한 업그레이드 조건 확인
    switch(currentLevel) {
        case BASIC: return (user.getLogin() >= 50);
        case SILVER: return (user.getRecommand() >= 30);
        case GOLD: return false;    // 항상 업그레이드 불가
        // 로직에서 처리할 수 없는 레벨인 경우 예외 던짐
        default: throw new IllegalArgumentException("Unknown Level: " + currentLevel);
    }
}
```
- 업그레이드 조건을 만족했을 경우 구체적으로 무엇을 할 것인지에 대한 upgradeLevel()
```java
private void upgradeLevel(User user){
    // 사용자의 오브젝트의 레벨정보를 다음 단계로 변경
    if(user.getLevel() == Level.BASIC) user.setLevel(Level.SILVER;
    else if(user.getLevel() == LEVEL.SILVER) user.setLevel(Level.GOLD);
    // 변경된 오브젝트를 DB에 업데이트
    userDao.update(user);
}
```
하지만 다음 단계를 확인하는 로직과 변경하는 로직이 함께 있고 예외 상황에 대한 처리가 없다. 만약 업그레이드 조건을 GOLD 레벨인 사용자를 업그레이드하려고 하면 아무것도 처리하지 않고 그냥 DAO의 업데이트 메소드만 실행된다. 레벨이 늘어나면 코드도 점점 길어질 것이다.  
이를 수정하기 위해서는 레벨의 순서와 다음 단계 레벨이 무엇인지 결정하는 일은 Level이 하도록 해야 한다.
```java
public enum Level {
    // enum 선언에 DB에 저장할 값과 다음 단계의 레벨 정보도 추가
    GOLD(3, null), SILVER(2, GOLD), BASIC(1, SILVER);
    private final int value;
    private final Level next;

    Level(int value, Level next) {
        this.value = value;
        this.next = next;
    }

    public int intValue() { return value; }

    // 다음 레벨이 무엇인지 알고 싶을 때 호출
    public Level nextLevel() {
        return this.next;
    }

    public static Level valueOf(int value) {
        switch(value) {
            case 1: return BASIC;
            case 2: return SILVER;
            case 3: return GOLD;
            default: throw new AssertionError("Unknown value: "+ value);
        }
    }
}
```
- 사용자 정보가 바뀌는 부분을 UserService 메소드에서 User로 이동
```java
public void upgradeLevel() {
    Level nextLevel = this.level.nextLevel();
    if(nextLevel == null){
        throw new IllegalStateException(this.level + "은 업그레이드 불가");
    } else {
        this.level = nextLevel;
    }
}
```
- UserService는 User 오브젝트에 업그레이드에 필요한 작업을 수행하라고 요청만 하면 됨
``` java
private void upgradeLevel(User user) {
    user.upgradeLevel();
    userDao.update(user);
}
```

> 객체지향적인 코드는 다른 오브젝트의 데이터를 가져와서 작업하는 대신 데이터를 갖고 있는 다른 오브젝트에게 작업을 해달라고 요청한다. 오브젝트에게 데이터를 요구하지 말고 작업을 요청하라는 것이 객체지향 프로그래밍의 가장 기본이되는 원리이기도 하다.

<br>

### 5.2 트랜잭션 서비스 추상화
---
#### 5.2.1 모 아니면 도
##### 테스트용 UserService 대역
작업 중간에 예외를 강제로 만들기 위해 테스트 용으로 만든 UserSevice 대역을 사용하는 방법이 좋다.  
UserService를 상속해서 테스트에 필요한 기능을 추가하도록 일부 메소드를 오버라이딩 하는 방법을 사용하면 된다.  
```java
static class TestUserService extends UserService {
    private String id;

    // 예외를 발생시킬 User 오브젝트의 id를 지정할 수 있게 만든다.
    private TestUserSerice(String id) {
        this.id = id;
    }

    // 상속을 통해 오버라이딩이 가능하도록 protected로 선언
    protected void upgradeLevel(User user) {
        // 지정된 id의 User 오브젝트가 발견되면 예외를 던져서 작업을 강제로 중단시킨다.
        if(user.getId().equals(this.id)) throw new TestUserServiceException();
        super.upgradeLevel(user);
    }
}
```
다른 예외가 발생했을 경우와 구분하기 위해 테스트 목적을 띤 예외를 따로 정의해둔다.
```java
static class TestUserServiceException extends RuntimeException{}
```
##### 강제 예외 발생을 통한 테스트
테스트의 목적은 사용자 레벨 업그레이드를 시도하다가 중간에 예외가 발생했을 경우, 그 전에 업그레이드 했던 사용자도 다시 원래 상태로 돌아갔는지 확인하는 것.  
테스트용으로 만든 upgradeLevels()에서는 DB에서 5개의 User를 가져와 차례로 업그레이드를 하다가 지정해둔 4번째 사용자 오브젝트 차례가 되면 TestUserServiceException을 발생시킬 것이다.   TestUserServiceException을 잡은 후에는 checkLevelUpgrae()를 이용해 두 번째 사용자 레벨이 변경됐는지 확인해보면, 이미 레벨을 수정했던 두 번째 사용자도 원래 상태로 돌아가야 하지만 그대로 유지되고 있는 것을 확인할 수 있다.  

##### 테스트 실패의 원인
**트랜잭션** 문제. 모든 사용자의 레벨을 업그레이드하는 작업인 upgradeLevels()가 하나의 트랜잭션 안에서 동작하지 않았기 때문이다.  
> _**트랜잭션**이란 더 이상 나눌 수 없는 단위 작업 (원자성)_

#### 5.2.2 트랜잭션 경계설정
DB는 그 자체로 완벽한 트랜잭션을 지원한다. SQL을 이용해 다중 로우의 수정이나 삭제를 위한 요청을 했을 때 일부 로우만 삭제되고 나머지는 안되거나, 일부 필드는 수정했는데 나머지 필드는 수정이 안 되고 실패로 끝나는 경우는 없다. **하나의 SQL 명령을 처리하는 경우는 DB가 트랜잭션을 보장해준다고 믿을 수 있다.**  
하지만 여러 개의 SQL이 사용되는 작업을 트랜잭션으로 취급해야 하는 경우, 뒤에 실행되는 SQL이 성공적으로 DB에서 수행되기 전에 문제가 발생할 경우는 앞에서 처리한 SQL 작업도 취소시켜야 한다. (**트랜잭션 롤백**)  
반대로 여러 개의 SQL을 하나의 트랜잭션으로 처리하는 경우, 모든 SQL 수행 작업이 성공적으로 마무리 됐다고 DB에 알려서 작업을 확정시켜야 한다.(**트랜잭션 커밋**)

##### JDBC 트랜잭션의 트랜잭션 경계 설정
**트랜잭션 경계** : 어플리케이션 내에서 트랜잭션이 시작되고 끝나는 위치  
JDBC의 트랜잭션은 하나의 Connection을 가져와 사용하다가 닫는 사이에서 일어난다.  
JDBC에서 트랜잭션을 시작하려면 자동 커밋 옵션을 false로 만들어주면 된다.  
트랜잭션이 한 번 시작되면 commit()이나 rollback()이 호출될 때까지의 작업이 하나의 트랜잭션으로 묶인다. commit()이나 rollback()이 호출되면 그에 따라 작업 결과가 DB에 반영되거나 취소되고 트랜잭션이 종료된다. 일반적으로 작업 중에 예외가 발생하면 롤백한다.   

**트랜잭션 경계 설정** : setAutoCommit(false)로 트랜잭션 시작을 선언하고 commit() 또는 rollback()으로 트랜잭션을 종료하는 작업  

**로컬 트랜잭션** : 하나의 DB커넥션 안에서 만들어지는 트랜잭션

##### UserService와 UserDao의 트랜잭션 문제
JdbcTemplate의 작업 흐름은 하나의 템플릿 메소드 안에서 DataSource의 getConnection()을 호출해서 Connection 오브젝트를 가져오고, 작업을 마치면 Connection을 확실하게 닫아주고 템플릿 메소드를 빠져나온다. 일반적으로 트랜잭션은 커넥션보다 존재 범위가 짧다. 따라서 템플릿 메소드가 호출될 때마다 트랜잭션이 새로 만들어지고 메소드를 빠져나오기 전에 종료된다. 즉, JdbcTemplate 메소드를 사용하는 UserDao는 각 메소드마다 하나씩의 독립적인 트랜잭션으로 실행될 수 밖에 없다.  

데이터 액세스 코드를 DAO로 만들어서 분리해 놓았을 경우 DAO 메소드를 호출할 때마다 하나의 새로운 트랜잭션이 만들어지는 구조가 된다. 결국 DAO를 사용하면 비즈니스 로직을 담고 있는 UserService 내에서 진행되는 여러 작업을 하나의 트랜잭션으로 묶는 일이 불가능해진다.  

##### 비즈니스 로직 내의 트랜잭션 경계 설정
UserService(비즈니스 로직)과 UserDao(데이터 로직)를 그대로 두고 트랜잭션을 적용하려면 트랜잭션의 경계 설정 작업을 UserService쪽으로 가져와야 한다. 프로그램의 흐름을 볼 때 upgradeLevels()의 시작과 함께 트랜잭션이 시작하고 메소드를 빠져나올 때 트랜잭션이 종료돼야 하기 때문이다. 
```java
// upgrageLevels의 트랜잭션 경계설정 구조
public void upgradeLevels() throws Exception{
    (1) DB Conncection 생성
    (2) 트랜잭션 시작
    try {
        (3) DAO 메소드 호출
        (4) 트랜잭션 커밋
    } catch (Exception e) {
        (5) 트랜잭션 롤백
        throw e;
    } fianlly {
        (6) DB Connection 종료
    }
}
```
같은 트랜잭션 안에서 동작하기 위해서 UserDao의 update()는 반드시 upgradeLevels()에서 만든 Connection을 사용해야 한다.  
따라서 DAO 메소드를 호출할 때마다 Connection 오브젝트를 파라미터로 전달해줘야 한다.
```java
public interface UserDao{
    public void add(Connection c, User user);
    public User get(Connection c, String id);
    ...
    public void update(Connection c, User user1);
}
```
UserService의 upgradeLevels()는 UserDao의 update()를 직접 호출하지 않는다. UserDao를 사용하는 것은 사용자별로 업그레이드 작업을 진행하는 upgradeLevel()이다. 따라서 트랜잭션을 담고 있는 Connection을 공유하려면 UserService의 메소드 사이에도 같은 Connection 오브젝트를 사용하도록 파라미터로 전달해줘야 한다. 
 
##### UserService 트랜잭션 경계설정의 문제점
1. JdbcTemplate 활용 불가.
2. DAO의 메소드와 비즈니스 로직을 담고 있는 UserService의 메소드에 Connection 파라미터가 추가돼야 한다.
3. Connection 파라미터가 UserDao 인터페이스 메소드에 추가되면 UserDao는 더 이상 데이터 액세스 기술에 독맂벅일 수 없다.
4. 테스트 코드에 영향

#### 5.2.3 트랜잭션 동기화
##### Connection 파라미터 제거
**트랜잭션 동기화** : 트랜젝션을 시작하기 위해 만든 Connection 오브젝트를 특별한 저장소에 보관해두고, 이후세 호출되는 DAO의 메소드에서는 저장된 Connection을 가져다가 사용하게 하는 것. DAO가 사용하는 JdbcTemplate이 트랜잭션 동기화 방식을 이용하도록 하는 것. 그리고 트랜잭션이 모두 종료도면 동기화를 마치면 된다.
1. UserService는 Connection 생성
2. 트랜잭션 동기화 저장소에 저장해두고 Connection의 setAutoCommit(false)를 호출해 트랜잭션 시작시킨 후 DAO 기능 사용
3. 첫 번째 update() 호출
4. update() 내부에서 이용하는 JdbcTemplate 메소드에서는 트랜잭션 동기화 저장소에 현재 시작된 트랜잭션을 가진 Connection 오브젝트가 존재하는지 확인하고, 저장해둔 Connection을 발견하고 이를 가져옴
5. Connection을 이용해 PreparedStatement를 만들어 수정 SQL 실행하고 트랜잭션 동기화 저장소에서 DB 커넥션을 가져왔을 경우 JdbcTemplate은 Connection을 닫지 않은 채로 작업 종료
6. 두 번째 update() 호출
7. 트랜잭션 동기화 저장소에서 Connection을 가져옴
8. 가져온 Connection 사용
9. 마지막 update()
10. 같은 트랜잭션을 가진 Connection을 가져옴
11. 가져온 Connection 사용  
12. 트랜잭션 내의 모든 작업이 정상적으로 끝났으면 UserService는 Connection의 commit()을 호출해 트랜잭션 완료시킴
13. 트랜잭션 저장소가 더 이상 Connection 오브젝트를 저장해두지 않도록 이를 제거.  

어느 작업 중에라도 예외상황이 발생하면 UserSercive는 즉시 Connection의 rollback()을 호출해 트랜잭션을 종료할 수 있다. 이 때에도 저장소에 저장된 동기화된 Connection 오브젝트는 제거해줘야 한다.  
트랜잭션 동기화 저장소는 작업 스레드마다 독립적으로 Connection 오브젝트를 저장하고 관리하기 때문에 다중 사용자를 처리하는 서버의 멀티스레드 환경에서도 충돌날 염려는 없다.  

##### 트랜잭션 동기화 적용
트랜잭션 동기화 방식을 적용한 UserService
```java
// Connection 생성 시 사용항 DataSource를 DI 받도록 한다.
private DataSource dataSource;

public void setDataSource(DataSource dataSource) {
    this.dataSource = dataSource;
}

public void upgradeLevels() throws Exception {
    // 트랜잭션 동기화 관리자를 이용해 동기화 작업 초기화
    TransactionSynchronizationManager.initSynchronization();
    // DB 커넥션을 생성하고 트랜잭션 시작. 이후의 DAO 작업은 모두 여기서 시작한 트랜잭션 안에서 진행됨
    Connection c= DataSourceUtils.getConnection(dataSource);
    c.setAutoCommit(false);

    try{
        List<User> users = userDao.getAll();
        for(User user : users) {
            if(canUpgradeLevel(user)){
                upgradeLevel(user);
            }
        }
        c.commit();
    } catch (Exception e) {
        c.rollback();
        thrwo e;
    } finally {
        // 스프링 유틸리티 메소드를 이용해 DB 커넥션을 닫음
        DataSourceUtils.releaseConnection(c, dataSource);
        // 동기화 작업 종료 및 정리
        TransactionSynchronizationManager.unbindResource(this.dataSource);
        TransactionSynchronizationManager.clearSynchronization();
    }
}
```

##### 트랜잭션 테스트 보완
테스트용으로 확장해서 만든 TestUserService는 UserService의 서브클래스이므로 UserService와 마찬가지로 트랜잭션 동기화에 필요한 DataSource를 DI 해줘야 한다.   

##### JdbcTemplate과 트랜잭션 동기화
JdbcTemplate은 미리 생성돼서 트랜잭션 동기화 저장소에 등록된 DB커넥션이나 트랜잭션이 없는 경우, JdbcTemplate이 직접 DB 커넥션을 만들고 트랜잭션을 시작해서 JDBC 작업을 진행한다.  
반면 트랜잭션 동기화를 시작해 놓았다면 그 때부터 실행되는 JdbcTemplate의 메소드에서는 직접 DB 커넥션을 만드는 대신 트랜잭션 동기화 저장소에 들어 있는 DB 커넥션을 가져와서 사용한다. 이를 통해 이미 시작된 트랜잭션에 참여하게 된다.  
따라서 DAO를 사용할 때 트랜잭션이 굳이 필요 없다면 바로 호출해서 사용해도 되고, DAO 외부에서 트랜잭션을 만들고 이를 관리할 필요가 있다면 미리 DB 커넥션을 생성한 다음 트랜잭션 동기화를 해주고 사용하면 된다.  

#### 5.2.4 트랜잭션 서비스 추상화
##### 기술과 환경에 종속되는 트랜잭션 경계설정 코드
하나의 트랜잭션 안에서 여러 개의 DB에 데이터를 넣는 작업을 해야 하는 경우 JDBC의 Connection을 이용한 트랜잭션 방식인 로컬 트랜잭션으로는 불가능하다. 로컬 트랜잭션은 하나의 DB Connection에 종속되기 때문이다.  
따라서 각 DB와 독립적으로 만들어지는 Connection을 통해서가 아니라, 별도의 트랜잭션 관리자를 통해 트랜잭션을 관리하는 **글로벌 트랜잭션** 방식을 사용해야 한다.  
글로벌 트랜잭션을 적용해야 트랜잭션 매니저를 통해 여러 개의 DB가 참여하는 작업을 하나의 트랜잭션으로 만들 수 있다.    

**JTA** : 글로벌 트랜잭션을 지원하는 트랜잭션 매니저를 지원하기 위한 API  
어플리케이션에서는 기존 방법대로 DB는 JDBC, 메시징 서버라면 JMS 같은 API를 사용해서 필요한 작업을 수행한다. 단, 트랜잭션은 JTA를 통해 트랜잭션 매니저가 관리하도록 위임한다.  
트랜잭션 매니저는 DB와 메시징 서버를 제어하고 관리하는 각각의 리소스 매니저와 XA 프로토콜을 통해 연결된다. 트랜잭션 매니저를 활용하면 여러 개의 DB나 메시징 서버에 대한 작업을 하나의 트랜잭션으로 통합하는 분산 트랜잭션 또는 글로벌 트랜잭션이 가능해진다.  

```java
InitialContext ctx = new InitialContext();
UserTransaction tx = (UserTransaction)ctx.lookup(USER_TX_JNDI_NAME);

tx.begin();
Connection c = dataSource.getConnection();
try{
    // 데이터 액세스 코드
    tx.commit();
} catch (Exception e) {
    tx.rollback();
    throw e;
} finally {
    c.close();
}
```
하지만 JDBC 로컬 트랜잭션을 JTA를 이용하는 글로벌 트랜잭션으로 바꾸려면 UserService 코드를 수정해야 한다는 문제가 있다.  
또한 하이버네이트를 이용한 트랜잭션 관리코드는 JDBC나 JTA와 또 다르다. 하이버네이트는 Connection을 직접 하용하지 않고 Session이라는 것을 사용하고, 독자적인 트랜잭션 관리 API를 사용한다. 

##### 트랜잭션 API의 의존관계 문제와 해결책  
원래 UserService는 UserDao 인터페이스에만 의존하는 구조였기 때문에 DAO 클래스의 구현 기술이 JDBC에서 다른 기술로 바뀌어도 UserService 코드는 영향을 받지 않았다.  
하지만 JDBC에 종속적인 Connection을 이용한 트랜잭션 코드가 나오면서 UserService는 UserDaoJdbc에 간접적으로 의존하는 코드가 되었다.  

UserService의 코드가 특정 트랜잭션 방법에 의존적이지 않고 독립적이려면, 트랜잭션의 경계 설정을 담당하는 코드의 일정한 패턴을 찾아 추상화해야 한다.  
> _**추상화**란 하위 시스템의 공통점을 뽑아내서 분리시키는 것_

JDBC, JTA, HIBERNATE, JPA, JDO, JMS도 모두 트랜잭션 개념을 가지고 있으므로 트랜잭션 경계 설정 방법에서 공통점이 있을 것이다. 이 공통점을 모아서 추상화된 트랜잭션 관리 계층을 만들수 있고, 어플리케이션 코드에서는 트랜잭션 추상 계층이 제공하는 API를 이용해 트랜잭션을 이용하게 만든다면, 특정 기술에 종속되지 않는 트랜잭션 경계설정 코드를 만들 수 있을 것이다. 

##### 스프링의 트랜잭션 서비스 추상화
스프링은 트랜잭션 기술의 공통점을 담은 트랜잭션 추상화 기술을 제공하고 있다. 이를 이용하면 어플리케이션에서 직접 각 기술의 트랜잭션 API를 이용하지 않고도 일관된 방식으로 트랜잭션을 제어하는 트랜잭션 경계설정 작업이 가능해진다.  
```java
public void upgradeLevels() {
    // JDBC 트랜잭션 추상 오브젝트 생성
    PlatformTransactionManager transactionManager = new DataSourceTransactionManager(dataSource);

    // 트랜잭션 시작
    TransactionStatus status = transactionManager.getTransaction(new DefaultTransactionDefinition());

    try{
        List<User> users = userDao.getAll();
        for(User user : users) {
            if(canUpgradeLevel(user)){
                upgradeLevel(user);
            }
        }
        c.commit();
    } catch (Exception e) {
        c.rollback();
        thrwo e;
    } finally {
        // 스프링 유틸리티 메소드를 이용해 DB 커넥션을 닫음
        DataSourceUtils.releaseConnection(c, dataSource);
        // 동기화 작업 종료 및 정리
        TransactionSynchronizationManager.unbindResource(this.dataSource);
        TransactionSynchronizationManager.clearSynchronization();
    }
}
```

##### 트랜잭션 기술 설정의 분리 
트랜잭션 추상화 API를 적용한 UserService 코드를 JTA를 이용하는 글로벌 트랜잭션으로 변경하려면 PlatfromTransactionManager 구현 클래스를 DataSourceTransactionManager에서 JTATransactionManager로 바꿔주면 된다.  
```java
PlatformTransactionManager txManager = new JTATrasactionManager();
``` 
하지만 어떤 트랜잭션 매니저 구현 클래스를 사용할지 UserService 코드가 알고 있는 것은 DI 원칙에 위배되므로 자신이 사용할 구체적인 클래스를 컨테이너를 통해 외부에서 제공받도록 DI 방식으로 구현해야 한다.  
이를 위해서는 DataSourceTransactionManager는 스프링 빈으로 등록하고 UserService가 DI 방식으로 사용하게 해야 한다. 클래스를 스프링 빈으로 등록하기 위해서는 싱긅톤으로 만들어져 여러 스레드에서 동시에 사용해도 안전해야 한다. 스프링이 제공하는 모든 PlatformTransactionManager의 구현 클래스는 싱글톤으로 사용이 가능하므로 스프링의 싱글톤 빈으로 등록해도 된다.  

<br>

### 5.3 서비스 추상화와 단일 책임 원칙
---
##### 수직, 수평 계층구조와 의존관계
UserDao와 UserService는 각각 담당하는 코드의 기능적인 관심에 따라 분리되고, 서로 불필요한 영향을 주지 않으면서 독자적으로 확장이 가능하도록 만들어졌다. 같은 어플리케이션 로직을 담은 코드이지만 내용에 따라 분리되었으므로, 같은 계층에서 수평적인 분리라고 볼 수 있다.  
트랜잭션 추상화는 어플리케이션의 비즈니스 로직과 하위에서 동작하는 로우레벨의 트랜잭션 기술이라는 아예 다른 게층의 특성을 갖는 코드를 분리한 것이다.   

어플리케이션 로직의 종류에 따른 수평적인 구분이든, 로직과 기술이라는 수직적인 구분이든 모두 결합도가 낮으며, 서로 영향을 주지 않고 자유롭게 확장될 수 있는 구조를 만들 수 있는 데는 스프링의 DI가 중요한 역할을 하고 있다. 

##### 단일 책임 원칙
> **단일 책임 원칙** : 하나의 모듈은 한 가지 책임을 가져야 한다. (= 하나의 모듈이 바뀌는 이유는 한 가지여야 한다.)

##### 단일 책임 원칙의 장점
단일 책임 원칙을 잘 지키면 어떤 변경이 필요할 때 수정 대상이 명확해진다. 기술이 바뀌면 기술 계층과의 연동을 담당하는 기술 추상화 계층의 설정만 바꿔주면 된다.  

객체지향 설계와 프로그래밍의 원칙은 서로 긴밀하게 관련되어 있다. 단일 책임 원칙을 잘 지키는 코드를 만들려면 인터페이스를 도입하고 이를 DI로 연결해야 하며, 그 결과로 단일 책임 원칙뿐 아니라 개방 폐쇄 원칙도 지켜지고, 모듈 간에 결합도가 낮아지며, 변경일 단일 책임에 집중되는 응집도가 높은 코드가 된다. 또한 이러한 과정에서 전략 패턴, 어댑터 패턴, 브리지 패턴, 미디에이터 패턴 등 많은 디자인 패턴이 적용되기도 한다.   

스프링의 의존관계 주입 기술인 DI는 모든 스프링 기술의 기반이 되는 핵심 엔진이자 원리이며 스프링이 지지하고 지원하는 과정에서 사용되는 가장 중요한 도구이다. 

<br>

### 5.4 메일 서비스 추상화
---
#### 5.4.1 JavaMail을 이용한 메일 발송 기능
사용자 정보에 이메일 추가
##### JavaMail 메일 발송
자바에서 메일을 발송할 때는 표준 기술인 JavaMail을 사용하면 된다. javax.mail 패키지에서 제공하는 자바의 이메일 클래스를 사용한다.  
```java
private void sendUpgradeEMail(User user) {
    Properties props = new Properties();
    props.put("mail.smtp.host", "mail.ksug.org");
    Session s = Sessionl.getInstance(props, null);

    MimeMessage message = new MimeMessage(s);
    try{
        message.setFrom(new InternetAddress("useradmin@kug.org"));
        message.addRecipient(Message.RecipientType.TO, new InternetAddress(user.getEmail()));
        message.setSubject("Upgrade 안내");
        message.setText("사용자님의 등급이 " + user.getLevel().name() + "로 업그레이드 되었습니다");

        Transport.send(message);
    } catch (AddressException e) {
        throw new RuntimeException(e);
    } catch (MessagingException e) {
        throw new RuntimeException(e);
    } catch (UnsupportEncodingException e){
        throw new RuntimeException(e);
    }
}
```

#### 5.4.2 JavaMail이 포함됨 코드의 테스트
메일 빌송은 부하가 크고, 실제로 발송되면 안되므로 테스트용 사용자 정보의 메일 주소는 존재하지 않는 주소로 하거나, 테스트용으로 미리 준비한 특별한 주소로만 사용하게 해야 한다.  
그래도 테스트 할 때마다 메일 서버에 부담을 주게 되므로 테스트 때는 메일 서버 설정을 다르게 해서 테스트용으로 따로 준비된 메일 서버를 이용하는 것이 좋다.  
하지만 메일 발송 테스트라고 해도 잘 도착했는지 확인할 수 없기 때문에 메일 발송용 서버에 별 문제없이 전달됐을을 확인할 뿐이다. 하지만 메일 서버는 충분히 테스트된 시스템이므로 믿고 사용해도 된다.  
따라서 메일 테스트를 할 때에는 테스트 가능한 메일 서버까지만 잘 전송되는지 확인하면 되고 테스트용 메일 서버는 외부로 메일을 발송하지 않고 JavaMail과 연동해서 메일 전송 요청을 받는 것까지만 담당한다.  

JavaMail은 자바의 표준 기술이고 이미 많은 시스템에 사용돼서 검증된 안정적인 모듈이므로 JavaMail API를 통해 요청이 들어간다는 보장만 있으면 테스트 할 때마다 JavaMail을 직접 구동시킬 필요가 없다. 따라서 개발 중이거나 테스트 수행시에는 JavaMail을 대신할 수 있는, JavaMail을 사용할 때와 동일한 인터페이스를 갖는 코드가 동작하도록 만들어도 된다.

#### 5.4.3 테스트를 위한 서비스 추상화
##### JavaMail을 이용한 테스트의 문제점
JavaMail의 핵심 API는 DataSource처럼 인터페이스로 만들어져 구현을 바꿀 수 있는 것이 없다.  
JavaMail에서는 Session 오브젝트를 만들어야만 메일 메시지를 생성할 수 있고, 메일을 전송할 수 있다. 하지만 Session은 인터페이스가 아니고 클래스이다. 그리고 생성자가 모두 private으로 되어 있어 직접 생성도 불가능하며 Session 클래스는 상속이 불가능한 final 클래스이다.  

이러한 JavaMail처럼 테스트하기 힘든 구조인 API를 테스트하기 좋게 만드는 방법은 **서비스 추상화**를 적용하면 된다.

##### 메일 발송 기능 추상화
JavaMail의 서비스 추상화 인터페이스
```java
...

public interface MailSender {
    void send(SimpleMailMessage simpleMessage) throws MailException;
    void send(SimpleMailMessage[] simpleMessages) throws MailException;
}
```
위 인터페이스는 SimpleMailMessage라는 인터페이스를 구현한 클래스에 담긴 메일 메시지를 전송하는 메소드로만 구현되어 있다.  
기본적으로 JavaMail을 사용해 메일 발송 기능을 제공하는 JavaMailSenderImpl 클래스를 이용하면 된다. 
```java
private void sendUpgradeEMail(User user) {
    JavaMailSenderImpl mailSender = new JavaMailSenderImpl();
    mailSender.setHost("mail.server.com");

    SimpleMailMessage mailMessage = new SimpleMailMessage();
    mailMessage.setTo(user.getEmail());
    mailMessage.setFrom("useradmin@ksug.org"); mailMessage.setSubject("Upgrade 안내 "); mailMessage.setText("사용자님의 등급이 " + user.getLevel().name());

    mailSender.send(mailMessage);
}
```
스프링의 예외 처리 원칙에 따라 JavaMail을 처리하는 중에 발생한 각종 예외를 MailException이라는 런타임 예외로 포장해서 던져주기 때문에 try/catch가 필요 없다.  
MailMessage 타입의 SimpleMailMessage 오브젝트를 만들어서 메시지를 넣은 뒤 JavaMailSender 타입 오브젝트의 send() 메소드에 전달해주면 된다. 메시지는 간단한 텍스트가 전부이므로 MailMessage 인터페이스를 구현한 JavaMailSenderImpl의 오브젝트를 만들어 사용한다. JavaMailSenderImpl은 내부적으로 JavaMail API를 이용해 메일을 전송해준다.  

하지만 아직 JavaMail API를 이용하는 JavaMailSenderImpl 클래스의 오브젝트를 직접사용하기 때문에 스프링의 DI를 적용할 필요가 있다.  
sendUpgradeMail()에는 JavaMailSenderImpl 클래스가 구현한 MailSender 인터페이스만 남기고, 구체적인 메일 전송 구현을 담은 클래스 정보는 코드에서 모두 제거한다. UserService에 MailSender 인터페이스 타입의 변수를 만들고 수정자 메소드를 추가해 DI가 가능하도록 만든다.  

##### 테스트용 메일 발송 오브젝트
테스트용 메일 전송 클래스
MailSender 인터페이스를 구현해야 한다.

##### 테스트와 서비스 추상화

#### 5.4.4 테스트 대역