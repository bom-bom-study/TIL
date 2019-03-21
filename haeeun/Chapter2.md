## 2장 테스트
### 2.1 UserDaoTest 다시보기
---
#### 2.1.1 테스트의 유용성
테스트 : 예상하고 의도했던 대로 코드가 정확히 동작하는지 확인하고, 코드의 결함을 제거해가는 디버깅을 거치는 작업.

#### 2.1.2 UserDaoTest의 특징
```java
public class UserDaoTest {
    public static void main(String[] args) throws SQLException {
        ApplicationContext context =new GenericXmlApplicationContext("ApplicationContext.xml");

        UserDao dao = context.getBean("userDao", UserDao.class);

        User user =new User(); 
        user.setld("user"); 
        user.setName("백기선"); 
        user.setPassword("married");
        
        dao.add(user);

        System.out.println(user.getId() + " 등록 성공");
        
        User user2 = dao.get(user.getId()); 
        System.out.println(user2.getName()); 
        System.out.println(user2.getPassword());

        System.out.println(user2.getId() + " 조회 성공");
    }
}
```
* 내용
    - main() 이용
    - 테스트 대상인 UserDao의 오브젝트를 가져와 메소드 호출
    - 체스테 사용할 입력 값(User 오브젝트)을 직접 코드에서 만들어서 넣어줌
    - 테스트 결과를 콘솔에 출력
    - 에러가 없는 경우 콘솔에 성공 메시지 출력

1. 웹을 통한 DAO 테스트 방법의 문제점  
: DAO 뿐만 아니라 서비스 클래스, 컨트롤러, 뷰 등 모든 레이어의 기능을 다 만들고 나서야 테스트가 가능하다. 또한 테스트 해야 하는 DAO의 문제가 아니라 다른 코드 때문에 에러가 나거나 테스트가 실패할 수 있다.

2. 작은 단위의 테스트 (Unit test)  
: 테스트하고자 하는 대상이 명확하다면 그 대상에만 집중해서 테스트해야 한다. 한꺼번에 많은 것을 테스트하면 수행 과정도 복잡해지고, 오류 발생 시 정확한 원인을 찾기 힘들다. 테스트의 관심이 다르다면 테스트할 대상을 분리하고 집중해서 접근해야 하다. 개발자가 설계하고 만든 코드가 원래 의도한 대로 동작하는지 빨리 확인하기 위해서 수행한다. 확인의 대상과 조건이 간단하고 명확할수록 좋다.

3. 자동수행 테스트 코드  
: 테스트는 자동으로 수행되도록 코드로 만들어지는 것이 중요하다. 자주 반복할 수 있고, 빠르게 테스트를 실행할 수 있기 때문에 필요하다. 

4. 지속적인 개선과 점진적인 개발을 위한 테스트  
: 새로운 기능에 대한 동작을 확인할 수 있고, 기존에 만들어뒀던 기능들이 새로운 기능이 추가되며 수정한 코드에 영향을 받지 않고 계속해서 잘 동작하는지 확인 가능


#### 2.1.3 UserDaoTest의 문제점
1. 수동 확인 작업의 번거로움
2. 실행작업의 번거로움


### 2.2 UserDaoTest의 개선
---
#### 2.2.1 테스트 검증의 자동화
#### 2.2.2 테스트의 효율적인 수행과 결과 관리
1. JUnit 테스트로 전환. 
JUnit : 자바 테스팅 프레임워크. 프레임워크이므로 개발자가 만든 클래스의 오브젝트를 생성하고 실행하는 일이 프레임워크에 의해 진행되므로 main() 메소드가 필요없고 오브젝트를 만들어서 실행시키는 코드를 만들 필요도 없다. 

2. 테스트 메소드 전환  
테스트가 main() 메소드로 만들어졌다는 건 제어권을 직접 갖는다는 의미이므로 main()에 있던 테스트 코드를 일반 메소드로 옮겨야 한다. JUnit 프레임워크가 요구하는 조건은 메소드가 public으로 선언되어야 하고, @Test 어노테이션이 필요하다.

3. 검증 코드 전환  
테스트 결과를 검증할 때 if/else 대신 assertThat이라는 static method로 사용하면 된다.
``` java
if(!user.getName().equals(user2.getName())) {...}

assertThat(user2.getName(), is(user.getName()));
```
assertThat()은 첫번째 파라미터 값을 뒤에 나오는 matcher라는 조건으로 비교해서 일치하면 다음으로 넘어가고, 아니면 실패하도록 한다. is()는 matcher의 일종으로 equals()로 비교해주는 기능을 함.

* JUnit을 적용한 UserDaoTest
```java
import static org.hamcrest.CoreMatchers.is; 
import static org.junit.Assert.assertThat;

public class UserDaoTest {
    @Test
    public void addAndGet() throws SQLException {
        ApplicationContext context =new GenericXmlApplicationContext("applicationContext.xml");
        UserDao dao = context.getBean("userDao", UserDao.class); User user = new User();
        user.setId("gyumee");
        user.setName("박성철"); 
        user.setPassword("springnol");
        
        dao.add(user);

        User user2 =dao.get(user.getld());
        assertThat(user2.getName(), is(user.getName())); 
        assertThat(user2.getPassword(), is(user.getPassword()));
    }
} 
```

4. JUnit 테스트 실행
main() 메소드를 추가하고, 그 안에 JUnitCore 클래스의 main 메소드를 호출하는 간단한 코드를 넣어주면 됨. 메소드 파라미터에는 @Test 테스트 메소드를 가진 클래스 이름을 넣어준다.
``` java
import org.junit.runner.1UnitCore;

public static void main(String[] args) { 
    JUnitCore.main("springbook.user.dao.UserDaoTest");
}
```


### 2.3 개발자를 위한 테스팅 프레임워크 JUnit
#### 2.3.1 JUnit 테스트 실행 방법
1. IDE
2. 빌드 툴

#### 2.3.2 테스트 결과의 일관성
코드에 변경사항이 없다면 테스트는 항상 동일한 결과를 내야 한다.  
이전 테스트로 인해 DB에 등록된 중복데이터가 있는 경우 테스트에 실패할 수 있다. 가장 좋은 해결책은 테스트를 마치고 나서 테스트가 등록한 정보를 삭제해서 테스트를 수행하기 이전 상태로 만들어주어야 한다. 

1. deleteAll()의 getCount() 추가
    - deleteAll 
        : UserDao에 deleteAll() 추가. 테이블의 모든 레코드를 삭제해주는 기능.
    - getCount
        : 테이블의 레코드 개수 반환

2. deleteAll()과 getCount() 테스트
* deleteAll()과 getCount()가 추가된 addAndGet()
```java
import static org.hamcrest.CoreMatchers.is; 
import static org.junit.Assert.assertThat;

public class UserDaoTest {
    @Test
    public void addAndGet() throws SQLException {
        ApplicationContext context =new GenericXmlApplicationContext("applicationContext.xml");
        UserDao dao = context.getBean("userDao", UserDao.class); 
        
        dao.deleteAll();
        assertThat(dao.getCount(), is(0));

        User user = new User();
        user.setId("gyumee");
        user.setName("박성철"); 
        user.setPassword("springnol");
        
        dao.add(user);
        assertThat(dao.getCount(), is(1));

        User user2 =dao.get(user.getld());
        assertThat(user2.getName(), is(user.getName())); 
        assertThat(user2.getPassword(), is(user.getPassword()));
    }
} 
```

3. 동일한 결과를 보장하는 테스트
단위 테스트를 할 경우 다른 메소드에서도 DB를 사용할 수 있으므로 이전에 어떤 작업을 하다가 실행할 지 알 수 없다. 따라서 테스트 후에 테이블을 지우는 것보다 테스트 전에 테스트 실행에 문제가 되지 않는 상태를 만들어 주는 것이 낫다. 


#### 2.3.3 포괄적인 테스트
1. getCount() 테스트  
JUnit은 하나의 클래스 안에 여러 개의 테스트 메소드가 들어있을 수 있다. @Test가 있고, public이 있으며 리턴값이 void이고 파라미터가 없다는 조건을 지켜야 한다.   
*테스트 시나리오 : 테이블의 데이터를 모두 지우고 getCount()로 레코드 개수가 0임을 확인하고 3개의 사용자 정보를 하나씩 추가하면서 getCount()의 결과가 하나씩 증가하는지 확인. 테스트 전에, 모델 클래스에 모든 정보를 넣을 수 있도록 리스트를 초기화 할 수 있는 생성자 추가.  

2. addAndGet() 테스트 보완  
조건으로 검색하는 경우에 대한 테스트 기능 보완이 필요. 

3. get() 예외조건에 대한 테스트
get()에 전달된 id 값에 해당하는 사용자 정보가 없는 경우, null을 리턴하거나 예외를 던지는 방법이 있다. 스프링이 정의한 데이터 액세스 예외 클래스인 EmptyResultDataAccessException 예외 이용.  
이 경우 예외가 던져져야 테스트가 성공하는데, 이런 경우 assertThat()으로 검증 불가.  
JUnit에서 제공하는 예외 조건 테스트 방법을 이용하면 된다.

* get()의 예외상황에 대한 테스트
``` java
@Test(expected=EmptyResultDataAccessException.class){
    public void getUserFailure() throws SQLException {
         ApplicationContext context =new GenericXmlApplicationContext ("applicationContext .xml");

        UserDao dao = context.getBean("userDao", UserDao.class); 
        dao.deleteAll();
        assertThat(dao.getCount(), is(0));

        dao.get( "unknown_id");
}
```
expected 엘리먼트 : expected에는 테스트 메소드 실행 중 발생할 것으로 기대하는 예외 클래스를 넣어주면 됨. ecpected를 추가해놓으면 정상적으로 테스트 메소드가 끝나면 테스트가 실패하고, expected에 지정한 예외가 던져지면 테스트 성공.

4. 테스트를 성공시키기 위한 코드의 수정
get() 메소드 코드 수정이 필요함. 결과가 null일 경우 예외를 던지는 코드 추가.

5. 포괄적인 테스트
테스트를 작성할 때 부정적인 케이스를 먼저 만드는 것이 좋다. 이를 확인할 수 있는 테스트를 먼저 만들어야 예외 상황을 빠뜨리지 않는 개발이 가능하다. 

#### 2.3.4 테스트가 이끄는 개발
1. 기능 설계를 위한 테스트

2. 테스트 주도 개발 (TDD, Test Driven Development)  
만들고자 하는 기능의 내용을 담고 있으면서 코드 검증도 가능한 테스트 코드를 먼저 만들고 테스트를 성공하게 하는 코드를 작성하는 방식의 개발. 테스트를 먼저 만들고 테스트가 성공하도록 하는 코드만 만드는 식으로 진행하기 때문에 테스트를 빼먹지 않고 만들수 있고, 테스트 작성 시간고 코드 작성 시간의 간격이 짧아진다. 또한 자연스럽게 단위 테스트를 만들 수 있다. 

#### 2.3.5 테스트 코드 개선
테스트 코드 리팩토링. 중복된 코드를 별도의 메소드로 뽑아내는 방법.
JUnit 프레임워크는 테스트 메소드를 실행할 때 부가적인 작업을 하는데, 그 중 테스트를 실행할 때마다 반복되는 준비 작업을 별도의 메소드에 넣게 하고, 이를 테스트 메소드 실행 전에 먼저 실행시켜주는 기능이 있다. 

1. @Before
setUp()을 만들고 중복 코드를 넣는다. 테스트 메소드에서 필요한 dao 변수가 setUp() 로컬 변수로 되어 있으므로 dao를 테스트 메소드에서 접근할 수 있도록 인스턴스 변수로 
```java
import org.junit.Before;
...
public class UserDaoTest {
    private UserDao dao;

    @Before
    public void setUp(){
        ApplicationContext context = new GenericXmlApplicationContext("applicationContext.xml"); 
        this.dao = context.getBean("userDao", UserDao.class);
    }
    ...
    @Test
    public void addAndGet() throws SQLException { ... }
    
    @Test
    public void count() throws SQLException { ... }

    @Test(expected=EmptyResultDataAccessException.class)
    public void getUserFailure() throws SQLException { ... }
}
```

* JUnit 프레임워크가 테스트 메소드를 실행하는 과정
    1. 테스트 클래스에서 @Test가 붙은 public이고 void이며 파라미터가 없는 테스트 메소드를 모두 찾는다.
    2. 테스트 클래스의 오브젝트를 하나 만든다.
    3. @Before가 붙은 메소드가 있으면 실행한다.
    4. @Test가 붙은 메소드를 하나 호출하고 테스트 결과를 저장해둔다.
    5. @After가 붙은 메소드가 있으면 실행한다.
    6. 나머지 테스트 메소드에 대해 2~5번 반복한다.
    7. 모든 테스트의 결과를 종합해 돌려준다.

@Before나 @After가 붙은 메소드를 자동 실행하므로 공통적인 테스트 준비와 정리 작업을 자동으로 실행해준다. 대신, @Before나 @After 메소드를 테스트 메소드에서 직접 호출하지 않기 때문에 서로 주고받을 정보나 오브젝트가 있으면 인스턴스 변수를 이용해야 함.  
테스트 메소드를 실행할 때마다 테스트 클래스의 오브젝트를 새로 만든다. 한 번 만들어진 테스트 클래스의 오브젝트는 하나의 테스트 메소드를 사용하고 나면 버려진다.  
각 테스트가 서로 영향을 주지 않고 독립적으로 실행됨을 보장하기 위해 새로운 오브젝트를 생성한다. 따라서 인스턴스 변수가 있어도 다음 테스트 메소드가 실행될 때는 새로운 오브젝트가 만들어져 초기화되므로 사용 가능하다. 

2. 픽스처  
* fixture : 테스트를 수행하는 데 필요한 정보나 오브젝트. 일반적으로 여러 테스트에서 반복적으로 사용되므로 @Before 메소드로 생성해 두는 것이 좋다. 


### 2.4 스프링 테스트 적용
---
@Before 메소드가 테스트 메소드 개수만큼 반복되기 때문에 어플리케이션 컨텍스트도 그만큼 만들어진다. 어플리케이션 컨텍스트가 만들어질 때는 모든 싱글톤 빈 오브젝트를 초기화한다. 어떤 빈은 오브젝트가 생성될 때 자체적인 초기화 작업을 진행하기 때문에 빈이 많아지고 복잡해지면 어플리케이션 컨텍스트 생성에 시간이 오래 걸릴 수 있다.  
또한 어플리케이션 컨텍스트가 초기화될 대 어떤 빈은 독자적으로 많은 리소스를 할당하거나 독립적인 스레드를 띄우기도 하는데 테스트를 마칠 때마다 할당한 리소스 등을 정리하지 않으면 다음 테스트에서 새로운 어플리케이션 컨텍스트가 만들어지며 무제가 발생할 수도 있음.  
빈은 싱글톤이기 때문에 상태를 갖지 않기 때문에 어플리케이션 컨텍스트는 초기화되고 나면 내부 상태가 거의 바뀌지 않아 한 번만 만들고 공유해서 사용해도 되지만 JUnit이 매번 테스트 클래스의 오브젝트를 새로 만들기 때문에 참조할 어플리케이션 컨텍스트를 오브젝트 레벨에 저장해두면 안된다. 따라서 테스트 클래스 전체에 걸쳐 한 번만 실행되는 @BeforeClass static 메소드에서 어플리케이션 컨텍스트를 만들어 static 변수에 저장해두고 테스트 메소드에서 사용하게 하면 된다.

#### 2.4.1 테스트를 위한 애플리케이션 컨텍스트 관리
1. 스프링 테스트 컨텍스트 프레임워크 적용  
    @Before 메소드에서 어플리케이션 컨텍스트를 생성하는 코드를 제거하고 ApplicationContext 타입의 인스턴스 변수를 선언한 뒤 @Autowired 어노테이션을 붙인다. 클래스 레벨에 @RunWith와 @ContextConfiguration 어노테이션을 리스트에 추가한다.
    ```java
    @RunWith(SpringJUnit4ClassRunner.class)
    @ContextConfiguation(locations="/applicationContext.xml")
    public class UserDaoTest {
        @Autowired
        private ApplicationContext context;
        ...

        @Before
        pubilc void setUp() {
            this.dao = this.context.getBean("userDao", UserDao.class);
            ...
        }
    }
    ```
    인스턴스 변수인 context를 초기화하지 않아도 오류가 발생하지 않는 이유는 context 변수에 어플리케이션 컨텍스트가 들어있기 때문. 

2. 테스트 메소드의 컨텍스트 공유
    context 변수에 어플리케이션 컨텍스트가 들어있을 수 있는 이유는 JUnit 확장기능이 테스트가 실행되기 전에 한 번만 어플리케이션 컨텍스트를 만들어두고, 테스트 오브젝트가 만들어질 때마다 어플리케이션 컨텍스트 자신을 오브젝트의 특정 필드에 주입해줌.(일종의 DI)

3. 테스트 클래스의 컨텍스트 공유
    여러 개의 테스트 클래스에서 모두 같은 설정파일을 가진 어플리케이션 컨텍스트를 사용한다면 스프링은 테스트 클래스 사이에서도 어플리케이션 컨텍스트를 공유하게 해준다. 

4. @Autowired
    스프링의 DI에서 사용되는 특별한 어노테이션.  
    @Autowired가 붙은 인스턴스 변수가 있으면, 테스트 컨텍스트 프레임워크는 변수 타입과 일치하는 컨텍스트 내의 빈을 찾는다. 타입이 일치하는 빈이 있으면 인스턴스 변수에 주입한다. 
    스프링 어플리케이션 컨텍스트는 초기화할 때 자기 자신도 빈으로 등록하기 때문에 어플리케이션 컨텍스트에는 Applicationcontext 타입의 빈이 존재하는 것이므로 DI가 가능하다.  
    @Autowired로 어플리케이션 컨텍스트가 가지고 있는 빈을 DI 받을 수 있다면 UserDao 빈을 직접 DI 받을 수 있기 때문에 ApplicationContext 타입의 인스턴스 변수를 없애도 된다.  
    @Autowired는 변수에 할당 가능한 타입을 가진 빈을 자동으로 찾는다. @Autowired는 같은 타입의 빈이 두 개 이상 있는 경우에는 타입으로 가져올 빈 하나를 선택할 수 없기 때문에 변수의 이름과 같은 이름의 빈이 있는지 확인하고, 변수 이름으로도 빈을 가져올 수 없으면 예외 발생.
  

#### 2.4.2 DI와 테스트
cf) 인터페이스를 두고 DI를 적용해야 하는 이유
```
- 변경이 필요한 상황에서 수정 용이
- 다른 차원의 서비스 기능 도입 가능 (ex. DB 커넥션 카운팅 기능 등) => AOP
- 효율적인 테스트 쉽게 가능
```

1. 테스트 코드에 의한 DI
테스트 코드 내에서 setter를 호출해서 사용. 

2. 테스트를 위한 별도의 DI 설정  
테스트에서 사용될 DataSource 클래스가 빈으로 정의된 테스트 전용 설정 파일을 따로 만들어두고 테스트에서는 테스트 전용 설정파일만 사용하면 됨.

3. 컨테이너 없는 DI 테스트  
스프링 컨테이너에 의존하지 않고 테스트 코드에서 직접 오브젝트를 만들도 DI에서 사용
@Before 메소드에서 직접 UserDao 오브젝트를 생성하고 테스트용 DataSource 오브젝트를 만들어서 직접 DI

cf) 침투적 기술과 비침투적 기술
```
- 침투적 기술 (invasive)
    : 기술을 적용했을 때 어플리케이션 코드에 기술 관련 API가 등장하거나, 특정 인터페이스나 클래스를 사용하도록 강제하는 기술.  
    어플리케이션 코드가 해당 기술에 종속됨.
- 비침투적 기술 (noninvasive)
    : 어플리케이션 로직을 담은 코드에 아무런 영향을 주지 않고 적용 가능. 기술에 종속적이지 않은 순수한 코드 유지 가능.  
    스프링은 비침투적 기술
```
4. DI를 이용한 테스트 방법 선택  
항상 스프링 컨테이너 없이 테스트할 수 있는 방법을 우선적으로 고려하는 것이 좋다. 속도가 가장 빠르고 간결하다.  
여러 오브젝트와 복잡한 의존관계를 갖고 있는 오브젝트의 경우 스프링 설정을 이용한 DI 방식의 테스트 이용이 편리하다. 테스트 전용 설정 파일을 사용하는 편이 좋은데, 예외적인 의존관계를 간제로 구성해서 테스트 해야 할 경우 수동으로 DI해서 테스트 하면 됨.


### 2.5 학습 테스트로 배우는 스프링
---
학습 테스트(learning test) : 자신이 만들지 않은 프레임워크나 다른 곳에서 만들어서 제공한 라이브러리 등에 대해 작성한 테스트. 사용할 API나 프레임워크의 기능을 테스트로 보면서 사용 방법을 익히는 목적. 

#### 2.5.1 학습 테스트의 장점
* 다양한 조건에 따른 기능을 손쉽게 확인 가능
* 학습 테스트 코드를 개발 중에 참고 가능
* 프레임워크나 제품 업그레이드 시 호환성 검증 가능
* 테스트 작성에 대한 좋은 훈련
* 새로운 기술 공부에 도움

#### 2.5.2 학습 테스트 예제
1. JUnit 테스트 오브젝트  
    * 테스트 메소드를 수행할 때마다 새로운 오브젝트를 만드는지 확인  
        : 새로운 테스트 클래스를 만들고 n개의 테스트 메소드를 추가한다. 테스트 클래스 타입으로 static 변수를 선언하고 매 테스트 매소드에서 현재 static 변수에 담긴 오브젝트와 자신을 비교해서 같지 않다는 사실을 확인한다. 그리고 현제 오브젝트를 그 static 변수에 저장한다.   
        테스트를 해보면 테스트 메소드가 실행될 때마다 static 변수에 저장해둔 오브젝트와 다른 새로운 테스트 오브젝트가 만들어진 것을 확인할 수 있다.

    * n 개의 테스트 오브젝트 중 어떤 것도 중복되지 않는지 확인
        : static 변수로 테스트 오브젝트를 저장할 수 있는 컬렉션을 만들고 테스트마다 현재 테스트 오브젝트가 컬렉션에 이미 등록되어 있는지 확인하고 없으면 자기 자신을 추가하는 과정을 반복한다. 이 방법은 테스트가 어떤 순서로 실행되는지에 상관없이 오브젝트 중복 여부 확인 가능.

2. 스프링 테스트 컨텍스트 테스트
    * 스프링 테스트용 어플리케이션 컨텍스트가 한 개만 생성되는지 확인
        : 테스트 메소드에서 매번 동일한 어플리케이션 컨텍스트가 context 변수에 주입됐는지 확인해야 한다. context를 저장해둘 static 변수가 null인지 확인하고 null이라면 통과하고 static 변수에 현재 context를 저장한다. 다음부터는 저장된 static 변수와 현재의 context와 같은지 비교 가능.  
        - assertThat() 이용
        - 조건문을 받아서 true/false 확인

#### 2.5.3 버그 테스트
버그 테스트 : 코드에 오류가 있을 때 그 오류를 가장 잘 드러내줄 수 있는 테스트. 일단 실패하도록 만들고, 버그 테스트가 성공하면 버그 해결.

* 필요성과 장점
    - 테스트의 완성도를 높여줌
    - 버그의 내용을 명확하게 분석하게 해줌
    - 기술적인 문제를 해결하는데 도움

cf)
```
- 동등분할(equivalence partitioning)
    : 같은 결과를 내는 값의 범위를 구분해서 각 대표 값으로 테스트 하는 방법

- 경계값 분석(boundary value analysis)
    : 에러는 동등분할 범위의 경계에서 주로 많이 발생한다는 특징을 이용해 경계 근처에 있는 값을 이용해 테스트 하는 방법.
```
