# 토비의 스프링 3.1 Vol.1
## 6장 : AOP 

### 6.1. 트랜잭션 코드의 분리
- 트랜잭션에서 비즈니스 로직을 담당하는 코드를 메소드로 추출
=> 트랜잭션을 담당하는 기술적인 코드가 UserService내에 있는 문제
=> 트랜잭션 코드를 클래스 밖으로 뽑아내자

- DI 적용을 이용한 트랜잭션 분리
    - DI : 실제 사용할 오브젝트의 클래스 정체는 감춘 채, 인터페이스를 통해 간접적으로 접근
    => UserService를 인터페이스로 만들고, 이를 구현하는 UserServiceImpl을 만든다
    => UserService를 구현하는 UserServiceTx 를 만들어 트랜잭션의 경계설정을 담당시킨다.
    => TransactionManager 라는 이름의 빈으로 등록된 트랜잭션 매니저를 DI로 받아뒀다가, 트랜잭션 안에서 동작하도록 만들어줘야하는 메소드 호출의 전과 후에 필요한 트랜잭션 경계설정 API를 사용해주면 된다.

### 6.2. 고립된 단위 테스트
- 가장 편하고 좋은 테스트 방법은 가능한 한 작은 단위로 쪼개서 테스트하는 것
- 그 원인을 찾기가 쉽다!

#### 6.2.1. 복잡한 의존관계 속의 테스트
- 여러개를 의존하는 경우, 배보다 배꼽이 더 큰경우 발생
#### 6.2.2. 테스트 대상 오브젝트 고립시키기
- 테스트를 위한 대역을 사용하자!
- UserServiceImpl 을 고립시키자
- UserServiceImpl을 테스를 위해 만들어진 MockUserDao, MockMailSender에만 의존하게 만든다



- 테스트 스텁 : 테스트 대상의 코드가 정상적으로 수행되도록 도와줌
- 목 오브젝트 : 뿐만 아니라, 부가적인 검증기능까지 가능

#### 6.2.3. 단위테스트와 통합테스트
- 단위테스트 : 테스트 대상 클래스를 목 오브젝트 등의 테스트 대역을 이용해 의존 오브젝트나 외부의 리소스를 사용하지 않도록 고립시켜서 테스트하는 것
- 통합테스트 : 두 개 이상의, 성격이나 계층이 다른 오브젝트가 연동하도록 만들어 테스트 하거나, 외부의 DB파일, 서비스 등의 리소스가 참여하는 테스트

- 가이드 라인
    - 단위테스트를 먼저 고려
    - 외부와의 의존관계를 차단하고, 목 오브젝트나 스텁의 테스트 대역을 이용
    - 외부 리소스가 반드시 필요한 테스트는 통합테스트로

#### 6.2.4. 목 프레임워크
- 대부분 의존 오브젝트를 필요로 하는 코드를 테스트하기 때문에 목 오브젝트가 필수적
- Mockito 프레임워크 : 목 오브젝트를 편리하게 작성하도록 도와주는 목 오브젝트 지원 프레임워크
    - 목 클래스를 일일히 준비해둘 필요 없이, 간단한 호출로 특정 인터페이스를 구현한 테스트용 목 오브젝트를 만들 수 있다.
    
    ``` java
    UserDao mockUserDao = mock(UserDao.class);
    when(mockUserDao.getAll()).thenRetrun(this.users);
    UserServiceImpl.setUserDao(mockUserDao); //테스트 대상에 DI
    verify(mockUserDao, times(2)).update(any(User.class)); //any는 파라미터의 내용은 무시하고, 호출 횟수만 확인.
    
    ```
    - mock()은 org.mockito.Matchers에 정의된 스태틱 메소드
    - Mockito의 static 메소드를 한 번 호출해주면 목 오브젝트가 만들어진다.
    - Mockito 오브젝트는 인터페이스를 이용해 목 오브젝트를 만든다.
    - getAll()이 호출될 때, users의 목록을 반환
    - User타입의 오브젝트를 파라미터로 받으면, update()메소드가 2번 호출됐는지 확인하라


### 6.3. 다이내믹 프록시와 팩토리 빈
#### 6.3.1. 프록시와 프록시 패턴, 데코레이터 패턴
- 프록시 : 마치 자신이 클라이언트가 사용하려고 하는 실제 대상인 것처럼 위장해서 클라이언트의 요청을 받아주는 것
- 타킷, 실체 : 프록시를 통해 최종적으로 요청을 받아 처리하는 실제 오브젝트
- 프록시의 목적 두 가지 
    - 클라이언트가 타깃에 접근하는 방법 제어
    - 타깃에 부가적인 기능을 부여
- 프록시 패턴 : 타깃에 대한 기능 자체에는 관여하지 않으면서 **접근방법**을 **제어**하려는 목적을 가진 경우. 타깃의 기능을 확장하거나 추가하지 않고 클라이언트가 타깃에 **접근하는 방식**을 변경해준다.
- ex) 클라이언트에게 타깃에 대한 레퍼런스를 넘겨야 할 때, 타깃 보으젝트를 만드는 대신 프록시를 넘겨준다. 그리고 프록시의 메소드를 통해 타깃을 사용하려고 시도하면, 그때 프록시가 타깃 오브젝트를 생성하고 요청을 위임해주는 식이다. 레퍼런스는 갖고있지만 끝까지 사용하지 않는 경우면 프록시를 통해 생성을 최대한 늦춤으로써 얻는 장점이 많음.
- ex) 특별한 상황에서 타깃에 대한 접근권한을 제어. 수정가능한 오브젝트를 특정 레이어에서는 읽기전용으로 동작하게 해야한다면, 오브젝트의 프록시를 만들어서 프록시의 특정 메소드를 사용하려고 하면 접근이 불가능하다고 예외를 발생시키기
- 프록시는 데코레이터와 유사하지만 코드에서 자신이 만들거나 접근할 타깃 클래스 정보를 알고있는 경우가 많음. why? 생성을 지연하는 프록시라면 구체적인 생성방법을 알아야 하기 때문에, 타깃 클래스에 대한 직접적인 정보를 알아야한다

- 데코레이터 패턴 : 타깃에 부가적인 기능을 런타임 시 다이내믹하게(컴파일 시점에서는 어떤 방법과 순서로 프록시와 타깃이 연결되어 사용되는지 정해져있지 않음) 부여해주기 위해 프록시를 사용하는 패턴. => 실제 내용물은 동일하지만 부가적인 효과를 줄 수 있다. **인터페이스**로 선언하여, **생성자**나 **수정자** 메소드를 통해 위임 대상을 외부에서 런타임 시에 주입받을 수 있도록 만든다.
    - ex) InputStream, OutputStream, 스프링의 빈 주입
     - InputStream
         ``` 
         InputStream is = new BufferedInputStream(new FileInputStream("a.txt"));
         ```
         InputSteram이라는 인터페이스를 구현한 FileInputStream에 버퍼 읽기 기능을 제공해주는 BufferedInputStream이라는 데코레이터 적용

    -![image](https://user-images.githubusercontent.com/32324250/60146200-985bfd00-9803-11e9-9e50-17e6f856001d.png)
    - 인터페이스를 통해 위임하는 방식이기에, 어느 데코레이터에서 타깃으로 연결될지 코드레벨에선 알 수 없음
    - 타깃의 코드를 손대지 않고 클라이언트가 호출하는 방법도 변경하지 않은채로 새로운 기능 추가

![image](https://user-images.githubusercontent.com/32324250/60146530-eb827f80-9804-11e9-84c4-ad28b8e422a8.png)

#### 6.3.2. (JDK의) 다이내믹 프록시
- 프록시 팩토리에 의해 런타임 시 다이내믹하게 만들어지는 오브젝트
- 프록시는 기존 코드에 영향을 주지 않으면서, 타깃의 기능을 확장하거나 접근방법을 제어할 수 있는 유용한 방법
- java.lang.reflect 에 프록시를 쉽게 만들 수 있도록 지원해주는 클래스가 있음
- 기존에는 타깃의 인터페이스를 구현하고 위임하는 코드를 작성하기가 번거롭고, 부가기능 코드가 중복될 가능성이 많아 귀찮았음
- 리플렉션 : 자바의 코드 자체를 추상화해서 접근하도록 만든 것. 다이내믹 프록시는 리플렉션 기능을 이용해서 프록시를 만들어준다. 
    - 자바의 클래스는 "클래스이름.class" 또는 getClass()를 통해 클래스의 정보를 담은 class타입의 오브젝트를 가져올 수 있다
    - getMothod() : 메소드에 대한 정의를 담음
    - invoke() : 메소드를 실행시킨 후 그 결과를 Object타입으로 돌려줌
    ``` java
    Method lengthMethod = String.class.getMethod("length");
    int length = lengthMethod.invoke(name); // int length = name.length();
    ```
- ![image](https://user-images.githubusercontent.com/32324250/60147232-a0b63700-9807-11e9-99ff-97165df9ce22.png)
- 프록시 팩토리에게 인터페이스 정보만 제공해주면, 해당 인터페이스를 구현한 클래스의 오브젝트를 자동으로 만들어준다 -> 프록시를 만들 때 인터페이스를 모두 구현해가면서 클래스를 정의하는 수고를 덜 수 있다.
- 오브젝트는 다이내믹 프록시가 만들어주지만, 프록시로서 필요한 부가기능 제공 코드는 직접 작성해야 한다. -> 부가기능은 InvocationHandler를 구현한 오브젝트에 담는다
    - ``` java
        public Object invoke(Object proxy, Method method, Object[] args)
        ```
    - 리플렉션의 Method 인터페이스를 파라미터로 받고, 메소드를 호출할 때 전달되는 파라미터도 args로 받는다. 
    - **다이내믹 프록시 오브젝트는 클라이언트의 모든 요청을 리플렉션 정보로 변환해서 invoke() 메소드로 넘긴다**
- 다이내믹 프록시는 타깃의 인터페이스와 같은 타입으로 만들어진다
- 클라이언트는 다이내믹 프록시를 타깃 인터페이스를 통해 사용할 수 있다
- 다이내믹 프록시의 생성은 Proxy클래스의 newProxyInstance()
    - 클래스 로더를 제공
    - 다이내믹 프록시가 구현할 인터페이스
    - 부가기능과 위임관련 코드를 담은 InvocationHandler 구현 오브젝트를 제공
- 다이내믹 프록시는 구현한 인터페이스의 메소드의 개수가 많을 때 좋다 -> 다이내믹 프록시가 만들어질 때 추가된 메소드가 자동으로 포함될 것이고, 부가기능은 invoke()에서 처리되므로

#### 6.3.3. 다이내믹 프록시를 이용한 트랜잭션 부가기능
- invoke() 메소드 구현 법 : 적용할 대상을 선별하는 대상을 작업을 먼저 진행

#### 6.3.4. 다이내믹 프록시를 위한 팩토리 빈
- TransactionHandler와 다이내믹 프록시를 스프링의 DI를 통해 사용할 수 있도록 만들어야함
- 스프링의 빈 생성방법
    - 1. 내부적으로 리플렉션의 newInstance()메소드를 통해 해당 클래스의 파라미터가 없는 생성자를 호출하고, 그 클래스 이름을 가지고 빈 오브젝트를 생성
        - 그러나 다이내믹 프록시 오브젝트는 이런식으로 프록시 오브젝트가 생성되지 않음. 클래스가 어떤것인지 알 수 없음
        - -> 다이내믹 프록시는 Proxy클래스의 newProxyInstance()라는 스태틱 팩토리 메소드를 통해서 만듦
    - 2. 팩토리 빈을 이용한 빈 생성
        - 팩토리 빈 : 스프링을 대신하여 오브젝트의 생성로직을 담당하도록 만들어진 특별한 빈
        - 스프링의 FactoryBean 인터페이스를 구현하여 만듦
        - ![image](https://user-images.githubusercontent.com/32324250/60148241-40c18f80-980b-11e9-9503-75579dbc53cd.png)
        - 팩토리 빈 클래스의 getObject()를 이용해 오브젝트를 가져오고, 이를 빈 오브젝트로 사용


#### 6.3.5. 프록시 팩토리 빈 방식의 장점과 한계
- 장점 - 프록시 팩토리 빈의 재사용. 기존 코드에 부가적인 기능을 추가해줄 수 있다.
    - 타깃 인터페이스를 구현하는 클래스를 일일이 만드는 번거로움을 제거

- 단점 - 한 번에 여러 개의 클래스에 공통적인 부가기능을 제공하는 일이 불가능
    - 하나의 타깃에 여러개의 부가기능을 적용하려고 할 때에도, 프록시 팩토리 빈 설정이 부가기능의 개수만큼 붙어야하므로 코드의 양이 늘어난다.
    - TransactionHandler가 프록시 팩토리 빈 개수만큼 만들어진다.


### 6.4. 스프링의 프록시 팩토리 빈
#### 6.4.1. ProxyFactoryBean
- 스프링은 서비스 추상화를 프록시 기술에도 동일하게 적용
- 스프링은 일관된 방법으로 프록시를 만들 수 있게 도와주는 추상 레이어를 제공
- 프록시는 스프링의 빈으로 등록돼야 함
- ProxyFactoryBean : 프록시를 생성해서 빈 오브젝트로 등록하게 해주는 팩토리 빈. 순수하게 프록시를 생성하는 작업만을 담당. 
- MethodInterceptor : 프록시를 통해 제공하는 부가기능을 담는 빈
    - invoke()메소드는 ProxyFactoryBean으로부터 타깃 오브젝트에 관한 정보도 함께 제공받음
    - 때문에 타깃 오브젝트에 상관없이 독립적으로 만들어질 수 있다
    - 타깃이 다른 여러 프록시에서 함께 사용할 수 있고, 싱글톤 빈으로 등록 가능
- 프록시를 생성하는 작업만을 담당, 프록시를 통해 제공해줄 부가기능은 별도의 빈에 둘 수 있다.

- 어드바이스 : 타깃 오브젝트에 적용하는 부가기능을 담은 오브젝트. 타깃이 필요없는 순수한 부가기능 
    - MethodInterceptor를 구현하면 타깃 오브젝트가 등장하지 않음   
    - ProxyFactoryBean에 있는 인터페이스 자동검출 기능을 사용해 타깃 오브젝트가 구현하고 있는 인터페이스 정보를 알아냄. 그 후 알아낸 인터페이스를 모두 구현하는 프록시를 만들어줌
    - ProxyFactoryBean은 JDK가 제공하는 다이내믹 프록시를 만들어줌
    - MethodInvocation은 타깃 오브젝트의 메소드를 실행할 수 있는 기능이 있기 때문에 부가기능을 제공하는 데에만 집중할 수 있다. 부가기능을 제공하는 오브젝트
    - MethodInterceptor는 콜백 오브젝트
    - proceed()를 실행하면 타깃 오브젝트의 메소드를 내부적으로 실행
    -> MethodInterceptor 구현 클래스는 일종의 공유가능한 템플릿처럼 동작
    - **ProxyFactoryBean은 작은 단위의 빔플릿/콜백 구조를 응용해서 적용했기 때문에 템플릿 역할을 하는 Methodlnvocation을 싱글톤으로 두고 공유할 수 있다**
    - addAdvice() : ProxyFactoryBean에 MethodInterceptor를 설정할 때 사용
        - Methodlnterceptor는 Advice 인터페이스를 상속하고 있는 서브인터페이스
        - 하나의 ProxyFactoryBean에 여러개의 부가기능을 제공해주는 MethodInterceptor를 추가 가능
        
- 포인트 컷 : 부가기능 적용 대상 메소드 선정 방법. 
    - InvocationHandler를 사용했을 때에는 메소드 이름과 패턴으로 구분
    - 그러나 Methodlnterceptor오브젝트는 여러 프록시가 공유해서 사용. 때문에 적용할 메소드의 이름패턴을 넣어주는것은 곤란..
    - MethodInterceptor에는 재사용 가능한 순수 부가기능 제공 코드만 남긴다
    - 프록시에 부가기능 적용 메소드를 선택하는 기능을 넣는 것. 메소드 선정 알고리즘을 담은 오브젝트

- ProxyFactoryBean 방식을 통해 Advice와 Pointcut을 활용하는 구조
![image](https://user-images.githubusercontent.com/32324250/60151122-846dc680-9816-11e9-86f8-17a484064ef9.png)
- 포인트컷 : 메소드 선정 알고리즘을 담은 오브젝트 (Pointcut인터페이스를 구현)
- 어드바이스 : 부가기능을 제공하는 오브젝트 (MethodInterceptor)
- 어드바이스와 포인트컷 모두 프록시에 DI로 주입돼서 사용됨 => 여러 프록시에서 공유 가능. 스프링의 싱글톤 빈으로 등록이 가능
- proceed() : 어드바이스가 부가기능을 부여하는 중, 타깃메소드의 호출이 필요할 때 실행하는 콜백

- 포인트컷과 어드바이스를 Advisor타입의 오브젝트에 담아서 **조합**을 만들어 등록
- 어드바이저 : 포인트컷(메소드 선정 알고리즘) + 어드바이스(부가기능)
    - 부가기능이 타깃 오브젝트마다 새로 만들어지는 문제를 해결!
- addAdvice() : 어드바이스만 등록
- addAdvisor() : 포인트컷과 어드바이스를 함께 등록

#### 6.4.2. ProxyFactoryBean 적용
- MethodInterceptor라는 Advice 서브인터페이스를 구현해서 만든다.
- 어드바이스와 포인트컷의 재사용. ProxyFactoryBean은 스프링의 DI, 템플릿콜백, 서비스 추상화가 모두 적용된 것. 독립적이며 여러 프록시가 공유할 수 있는 어드바이스와 포인트컷으로 확장 기능을 분리할 수 있다.

![image](https://user-images.githubusercontent.com/32324250/60151684-a0726780-9818-11e9-94b9-90098775907b.png)
- TransactionAdvice : 트랜잭션 부가기능을 담음. 하나만 만들어서 싱글톤 빈으로 등록하면 DI를 통해 모든 서비스에 적용 가능
- Pointcut : 메소드 선정 방식이 달라지는 경우에만 따로 등록
- Advisor로 조합해서 적용해주면 된다
- ProxyFactoryBean : 프록시를 생성해서 빈 오브젝트로 등록하게 해주는 팩토리 빈. 순수하게 프록시를 생성하는 작업만을 담당. 


### 6.5. 스프링 AOP
#### 6.5.1. 자동 프록시 생성
- 문제점 : 부가기능의 적용이 필요한 타깃 오브젝트마다 거의 비슷한 내용의 ProxyFactoryBean 빈 설정정보를 추가해준다.
- Target 프로퍼티를 제외하면 빈 클래스의 종류, 어드바이스, 포인트컷의 설정이 동일

- 빈 후처리기를 이용한 자동 프록시 생성기
    - 빈 후처리기 : BeanPostProcessor인터페이스를 구현하여 만듦. 스프링 빈 오브젝트로 만들어진 후, 빈 오브젝트를 다시 가공할 수 있게 해줌
    - DefaultAdvisorAutoProxyCreator : 어드바이저를 이용한 자동 프록시 생성기. 빈 후처리기. 
    - 빈 후처리기를 빈으로 등록하면, 스프링은 빈 오브젝트가 생성될 때마다 빈 후처리기에 보내서 후처리 작업을 요청
    - 빈 후처리기는 빈 오브젝트의 프로퍼티를 강제로 수정 또는 별도의 초기화 작업을 수행 가능
    - 이를 이용해 스프링이 생성하는 빈 오브젝트의 일부를 프록시로 포장하고, 프록시를 빈으로 대신 등록할 수 있다

- DefaultAdvisorAutoProxyCreator
    - ![image](https://user-images.githubusercontent.com/32324250/60152201-8cc80080-981a-11e9-8e68-aa947c34e7b2.png)
    - 스프링은 빈 오브젝트를 만들 때마다 DefaultAdvisorAutoProxyCreator에 빈을 보낸다
    - DefaultAdvisorAutoProxyCreator은 빈으로 등록된 모든 어드바이저 내의 포인트컷을 이용해, 전달받은 빈이 프록시 적용 대상인지 확인한다
    - 프록시 적용 대상이라면, 내장된 프록시 생성기에게 현재 빈에 대한 프록시를 만들게 하고, 만들어진 프록시에 어드바이저를 연결해준다
    - DefaultAdvisorAutoProxyCreator는 프록시가 생성되면 원래 컨테이너가 전달해준 빈 오브젝트대신 프록시 오브젝트를 컨테이너에게 돌려준다. 
    - 컨테이너는 최종적으로 DefaultAdvisorAutoProxyCreator가 돌려준 프록시를 빈으로 등록하고 사용한다

이처럼, 적용할 빈을 선정하는 로직이 추가된 포인트컷이 담긴 어드바이저를 등록하고 빈 후처리기를 시용하면 일일이 ProxyFactoryBean 빈을 등록하지 않아도 타깃 오브젝트에 자동으로 프록시가 적용되게 할 수 있다.

``` java
public interface Pointcut {
    ClassFilter getClassFilter(); // 프록시를 적용할 클래스인지 확인
    MethodMatcher getMethodMatcher(); // 어드바이스를 적용할 메소드인지 확인
}
```

#### 6.5.2. DefaultAdvisorAutoProxyCreator의 적용
- 클래스 필터를 적용한 포인트컷 작성
    - ClassFilter
    - 메소드 이름만 비교하던 포인트컷인 NameMatchMethodPointcut을 상속
    - 프로떠티로 주어진 이름 패턴을 가지고 클래스 이름을 비교
- 어드바이저를 이용하는 자동 프록시 생성기 등록
    - DefaultAdvisorAutoProxyCreator는 등록된 빈중에서 Advisor인터페이스를 구현한 것을 모두 찾음. 그 후 생성되는 모든 빈에대해 프록시 적용대상을 선정
    - DefaultAdvisorAutoProxyCreator 등록
        - ``` java
            <bean class="org.springframework.aoP.framework.autoproxy.Defa띠 tAdvisorAutoProxyCreator" />
     ```
        - 다른 빈에서 참조되는 빈이면 아이디를 등록하지 않아도 된다
- 포인트컷 등록
    - ``` java
        <bean id="transactionPointcut" class="springbook .service.NameMatchCl assMethodPointcut">
        <property name="mappedCl assName" value="*Servicelmpl" />// 클래스 이름 패턴
        <property name="mappedName" value= "upgrade*" /> // 메소드 이름 때턴 
        </bean>

    ```
- 어드바이스와 어드바이저
    - 어드바이스인 transactionAdvice의 빈은 수정할게 없다
    - 어드바이저로서 사용되는 방법 : DefaultAdvisorAutoProxyCreator에 의해 자동수집되고, 자동생성된 프록시에 다이내믹하게 DI되어 동작하는 어드바이저가 된다

- 스프링 빈 설정
    - ``` java
        <bean id= "testUserService" class= "springbook.user.service .UserServiceTest$TestUserServiceImpl" parent="userService" />
    ```
    - $ : 클래스 이름에 $ 기호를 사용하면, 스태틱 멤버 클래스를 지정하는 것
    - parent : <bean> 태그에 parent 애트리뷰트를 사용하면 다른 빈 설정의 내용을 상속받을 수 있다. 

- 자동생성 프록시 확인
    - 1. 트랜잭션이 필요한 빈에 트랜잭션 부가기능이 적용됐는가 
        -> 예외상황에서 트랜잭션이 롤백되게 함으로써 트랜잭션 적용 여부를 태스트
    - 2. 아무 빈에나 트랜잭션이 부가기능이 적용된 것은 아닌지 확인
        -> 포인트컷의 클래스 필터를 이용해서 정 확히 원하는 빈에만 프록시를 생성했는지 확인

#### 6.5.3. 포인트컷 표현식을 이용한 포인트컷
- 리플렉션 API는 코드를 작성하기가 제법 번거롭다
- 리플렉션 API를 이용해 메타정보를 비교하는 방법은 조건이 달라질 때마다 포 인트컷 구현 코드를 수정해야 한다
- 포인트컷 표현식 : 스프링이 제공하는, 포인트컷의 클래스와 메소드를 선정하 는 알고리즘을 작성할 수 있는 방법

##### 포인트컷 표현식
- AspectJExpressionPointcut 클래스를 사용
    - 클래스와 메소드의 선정 알고리즘을 포인트 컷 표현식을 이용해 한 번에 지정
    - AspectJ 포인트컷 표현식이라고도 부름
- 스프링의 포인트컷 : 1. 클래스 필터 2. 메소드 매처 (NameMatchClassMethodPointcut)
- 1. 포인트컷표현식문법 : 시그니처 비교
    - execution()
        - 메소드 실행에 대한 포인트컷이라는 의미 
        - 시그니처를 ()안에 넣어준다. 
        - ![image](https://user-images.githubusercontent.com/32324250/60154216-8f7a2400-9821-11e9-8efc-61a9be5a9cd5.png)
        - [] 괄호 : 생략가능
        - | : OR
        - 메소드의 풀 시그니처를 문자열로 비교하는 개념
    - ``` java
        System.out.println(Target.class.getMethod("minus", int.class, int.class));

        public int springbook.learningtest.spring.pointcut.Target.minus(int,int) throws java.lang.RuntimeException
    ```
        - public : 접근제한자. 생략가능(조건부여 않겠다)
        - int : 리턴값의 타입. 생략불가
        - springbook.learningtest.spring.pointcut.Target : 패키지와 타입 이름을 포함한 클래스의 타입패턴.생략가능(모두허용). 패키지 이름과 클래스 또는 인터페이스 이름에 *를 사용 가능. '..'를 사용하여 한 번에 여러개의 패키지 선택 가능
        - minus : 메소드 이름 패턴. 생략불가. *는 모든 메소드
        - (int, int) : 메소드 파라미터의 타입 패턴. '.'로 구분. 생략 불가. 순서대로 적는다. 파라미터가 없는 메소드면 ()로 표기. 모두 다 허용은 '..' 파라미터 조건 생략은'...'

  - ``` java
         int minus(int,int)
    ```
        - 어떤 접근제한자이든, 어떤 클래스에 정의됐든, 어떤 예외를 던지든 다 통과
  - ``` java
         * minus(int,int)
    ```  
        - 리턴타입 상관 없음
   - ``` java
         * minus(..)
    ```         
        - 파라미터 종류, 개수 상관 없음
  - ``` java
         * *(..)
    ``` 
        - 리턴 타입. 따라미터， 메쓰 이름에 상관없이 모든 메알 조건을 다 허용핸 포인트킷 표현식

- 2. bean() : 클래스와 메소드라는 기준을 넘어서 포인트컷을 비교
- 3. 애노테이션 비교 : 특정 애노테이션이 타입 메소드 파라미터에 적용되어 있는 것을 보고 메소드를 선정하게 하는 포인트컷
    - ``` java
        @annotation(org .springframework .transaction .annotation.Transactional)

    ```
    - @Transactional이라는 애노테이션이 적용된 메소드를 선정
   - ``` java
         <property name="mappedCl assName" value="ServiceImpl"/> 
         <property name="mappedName" value="upgrade*" />
         
    
        execution(* *..*Servicelmpl.upgrade*(..)
    ```
    - 클래스 이름은 Servicelmpl로 끝나고 메소드 이름은 upgrade로 시작하는 모든 클래스에 적용되도록 하는 표현식

- 단점 : 문자열로 된 표현식이므 로 런타임 시점까지 문법의 검증이나 기능 확인이 되지 않는다
- 유의 : 포인트컷 표현식의 클래스 이름에 적용되는 패턴은 클래스 이름 패턴이 아니라 타입 패턴
    - 때문에, 클래스의 이름을 바꾸어도, 포인트컷 표현식이 정의된다

#### 6.5.4. AOP란 무엇인가?
- 트랜잭션 서비스 추상화
    - 문제 : 특정 트랜잭션 기술에 종속되는 코드
    - 해결 : 트랜잭션 적용이라는 추상적인 작업 내용은 유치한 채로 구체적인 구현 방법을자유롭게 비꿀 수 있도록 서비스 추상화 기법을 적용
- 프록시와 데코레이터 패턴
    - 문제 : 트랜잭션은 거의 대부분의 비즈니스 로직을 담은 메소드에 필요. 단순한 추상화와 메소드 추출 방법으로는 더이상 제거할 방법이 없었다.
    - 해결 : 데코레이터 패턴을 적용. 투명한 부가기능 부여를 가능하게 하는 데코레이터 패턴의 적용
        - 데코레이터 패턴 : 타깃에 부가적인 기능을 런타임 시 다이내믹하게(컴파일 시점에서는 어떤 방법과 순서로 프록시와 타깃이 연결되어 사용되는지 정해져있지 않음) 부여해주기 위해 프록시를 사용하는 패턴.
        - 클라이언트가 일종의 대리자인 프록시 역할을 히는 트랜잭션 데코레이터를 거쳐서 타깃에 접근할 수 있게 됐다.

- 다이내믹 프록시와 프록시 팩토리 번
    - 문제 : 비즈니스 로직 인터페이스의 모든 메소드마다 트랜잭션 기능을 부여하는 코드를 넣어 프록시 클래스를 만드는 작업이 오히려 큰 짐. 인터페이스를 일일이 다 구현을 해줘야
    - 해결 :프록시 오브젝트를 런타임 시에 만들어주는 JDK 다이내믹 프록시 기술을 적용
       - 스프링의 프록시 팩토리 빈을 이용해서 다이내믹 프록시 생성 방법에 DI를 도입
       - 부가기능을 담은 어드바이스와 부가기능 선정 알고리즘을 담은 포인트컷은 프록시에서 분리될 수 있었고 여러 프록시에서 공 유해서 사용

- 자동 프록시 생성 방법과 포인트컷
    - 문제 : 적용 대상이 되는 빈마다 일일이 프록시 팩토리 빈을 설정해줘야 한다는 부담
    - 해결 : 스프링 컨테이너의 빈 생성 후처리 기법을 활용해 컨테이너 초기화 시점에서 지동으로 프록시를 만들어주는 방법을 도입
        - 트랜잭션 부가기능을 어디에 적용 하는지에 대한 정보를 포인트컷이라는 독립적인 정보로 완전히 분리

- 부가기능의 모듈화
    - 트랜잭션은 부가기능이기 때문에 스스로는 독립적인 방식으로 존재해서는 적용되기 어렵다. 
    - TransactionAdvice : 트랜잭션 경계설정 기능의 모듈화

- AOP: 애스펙트 지향 프로그래밍
    - 애스팩트 : 독립적인 모듈화가 불가능한 부가기능 모듈. 
        - 애플리케이션의 핵심기능은 아니지만, 애플리케이션을 구성하는 중요한 한가지 요소
        - 애스팩트 == 어드바이저
            - 애스팩트 = 어드바이스 + 포인트컷
    - 애스팩트 지향 프로그래밍(AOP) : 애플리케이션의 핵심적인 기능에서 부가적인 기능을 분리해서 애스펙트라는 독특한 모율로 만들어서 설계하고 개발하는 방법

#### 6.5.5. AOP 적용 기술
- 프록시를 이용한 AOP
    - 스프링의 AOP
    - 프록시로 만들어서 DI로 연결된 빈 사이에 적용해 타깃의 메소드 호출 과정에 참여해서 부가기능을 제공
    - 특별한 환경이나 JVM 설정 등을 요구하지 않음
    - 독립적으로 개발한 부가기능 모률을 다양한 타깃 오브젝트의 메소드에 다이내믹하게 적용해주기 위해 가장 중요한 역할을 맡고 있는 게 바로 프록시
    - 프록시를 통해 타깃의 메소드를 호출하는 전후에 다양한 부가기능을 제공 가능
- 바이트코드 샘성과 조작을 통한 AOP
    - AspectJ
    - 프록시 방식이 아닌 AOP
    - 타깃 오브젝트를 묻어고쳐서 부가 기능을 직접 넣어주는 직접적인 방법을 사용
    - 컴파일된 타깃의 클래스 파일 자체를 수정
    - 클래스가 JVM에 로딩되는 시점을 가로채서 바이트코드를 조작하는 복잡한 방법을 사용
    - 스프링과 같은 DI 컨테이너가 사용되지 않는 환경에서도 손쉽게 AOP의 적용이 가능
    - 프록시 방식보다 훨씬 강력하고 유연한 AOP가 가능
    - JVM의 실행 옵션을 변경, 별도의 바이트코드 컴파일러를 사용, 특별한 클래스 로더를 사용

#### 6.5.6. AOP의 용어
- 타깃 : 부가기능을 부여할 대상
- 어드바이스 : 타깃에게 제공할 부가기능을 담은 모듈
- 조인포인트 : 어드바이스가 적용될 수 있는 위치
    - 스프링에서 조인포인트는 메소드의 실행 단계 뿐
- 포인트컷 : 어드바이스를 적용할 조인 포인트를 선별하는 작업 또는 그 기능을 정 의한 모률
- 프록시 : 클라이언트와 타깃 사이에 투명하게 존재하면서 부가기능을 제공히는 오브젝트
- 어드바이저 : 포인트컷 + 어드바이스
    - 어떤 부가기능어드바이스)을 어디에(포인트컷) 전달할 것인가
    - 스프링은 자동 프록시 생성기가 어드바이저를 활용
    - 스프링 AOP에서만 사용되는 용어
- 애스펙트 : 한 개 또는 그 이상의 포인트컷과 어드바이스의 조합
    - 보통 싱글톤 형태의 오브젝트로 존재

#### 6.5.7. AOP 네임스페이스
- 스프링의 프록시 방식 AOP를 적용하기 위해 등록해야하는 4가지의 빈
    - 1. 자동프록시생성기
        - DefaultAdvisorAutoProxyCreator 클래스를 빈으로 등록
            - 빈 후처리기
            - 빈으로 등록된 어드바이저를 이용해서 프록시를 자동으로 생성하는 기능을 담당
    - 2. 어드바이스
        - TransactionAdvice
        - 유일하게 직접 구현한 클래스를 사용
        - 부가기능을 구현한 클래스를 빈으로 등록
    - 3. 포인트컷
        - AspectJExpressionPointcut을 빈으로 등록
        - 포인트컷 표현식을 넣어주면 된다
    - 4. 어드바이저
        - DefaultPointcutAdvisor 클래스를 빈으로 등록
        - 어드바이스와 포인트컷을 프로퍼티로 참조
        - 자동 프록시 생성기 에 의해 자동 검색되어 사용된다

- AOP 네임스페이스
    - 스프링은 AOP와 관련된 태그를 정의해둔 aop 스키마를 제공
    - 디폴트 네임스페이스의 <bean> 태그와 구분해서 사용
    
    ```     java    
    <beans xmlns="http://www.springframework.org/schema/beans"
        xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
        xmlns:aop="http://www.springframework.org/schema/aop" 

     
    ```

#### 6.6. 트랜잭션 속성
- DefaultTransactionDefinition : 트랜잭션 매니저에서 트랜잭션을 가져올 때 시용

##### 6.6.1. 트랜잭션 정의
- 트랜잭션 : 더 이상 쪼갤 수 없는 최소 단위의 작업 
- TransactionDefinition : DefaultTransactionDefinition이 구현. 트랜잭션의 동작방식에 영향을 줄 수 있는 네 가지 속성을 정의
    - 1. 트랜잭션전파 : 트랜잭션의 경계에서 이미 진행 중인 트랜잭션이 있을 때 또는 없을 때 어떻게 동작할 것인가를 결정하는 방식
        - getTransaction() : 트랜잭션 전파속성을 가져옴
        - PROPAGATION REQUIRED : 가장 많이 사용되는 트랜잭션 전파 속성
            - 진행 중인 트랜잭션이 없으면 새로 시 작하고， 이미 시작된 트랜잭션이 있으면 이에 참여
        - PROPAGATION_REQUIRES_NEW : 항상 새로운 트랜잭션을 시작한다. 즉 앞에서 시작된 트랜잭션이 있든 없든 상관없이 새로운 트랜잭션을 만들어서 독자적으로 동작
        - PROPAGATION_NOT_SUPPORTED : 트랜잭션 없이 동작. 진행 중인 트랜잭션이 있어도 무시
            - 그럼 왜 사용?? 모든 메소드에 트랜잭션 AOP가 적용되게 하고 특정 메소드의 트랜잭션 전파 속성만 PROPAGATION_NOT_ SUPPORTED로 설정해서 트랜잭션 없이 동작하게 만들 때 사용.
    - 2. 격리수준 : 적절하게 격리수준을 조정해서 가능한 한 많은 트랜잭션을 동시에 진행시키면서도 문제가 발생하지 않게 하는 제어가 펼요
        - ISOLATION_DEFAULT : DefaultTransactionDefinition 에 설정된 격리수준. DataSource에 설정되어 있는 디폴트
    - 3. 제한시간 : 기본 설정은 제한시간이 없는 것
    - 4. 읽기전용 : 트랜잭션 내에서 데이터를 조작하는 시도를 막아줄 수 있다

- 트랜잭션 정의 수정 :  DefaultTransactionDefinition을 사용하는 대신 외부에서 정의된 TransactionDefinition 오브젝트를 DI 받아서 사용하도록 만들면 된다.
    - 문제점 :  TransactionAdvice를 사용하는 모든 트랜잭션의 속성이 한꺼번에 바뀐다


#### 원히는 메소드만 선 택해서 독자적인 트랜잭션 정의를 적용할수 있는방법
#### 6.6.2. 트랜잭션 인터셉터와 트랜잭션 속성 
- 어드바이스의 기능을 확장 : 메소드 이름 패턴에 따라 다른 트랜잭션 정의가 적용되도록 만드는 것
- Transactionlnterceptor : 스프링이 제공
    - 트랜잭션 정의를 메소드 이름 패턴을 이용해서 다르게 지정할 수 있는 방법을 추가로 제공
    - 프로퍼티
        - PlatformTransactionManager
        - Properties(transactionAttributes) : 트랜잭션 속성을 정의한 프로퍼티
            - TransactionDefinition의 네 가지 기본 항목에 rollbackOn()이라는 메소드를 하나 더 갖고 있음
            - TransactionAdvice는 RuntimeException이 발생하는 경우에만 트랜잭션을 롤백시킨다.
            - rollbackOn() : 기본 원칙과 다른 예외처리가 가능하게 해준다. 즉 체크 예외를 처리할 수 있다. 
            - ex) 특정 체크예외의 경우는 트랜잭션을 롤백시키고， 특정 런타임 예외에 대해서는 트랜잭션을 커밋
            - 메소드 이름 패턴을 이용한 트랜잭션 속성 지정
            - ![image](https://user-images.githubusercontent.com/32324250/60164090-ea1e7a80-9837-11e9-9e9a-980280450dc6.png)
            - 트랜잭션 전파 항목만 필수
            ``` java
            <beans xmlns="http://www.springframework.org/schema/beans"
            xmlns:tx="http://www.springframework.org/schema/tx">
            ```
    - tx 네임스페이스를 이용한 설정 방법

#### 6.6.3 포인트컷과 트랜잭션 속성의 적용 전략 
- 포인트컷 표현식과 트랜잭션 속성을 정의할 때 따르면 좋은 몇 가지 전략
    - 1. 트랜잭션 포인트컷 표현식은 타입 패턴이나 번 이름을 이용한다
        - 트랜잭션용 포인트컷 표현식은 그 클래스들이 모여 있는 패커지를 통째로 선택하거나 클래스 이름에서 일정한 패턴을 찾아서 만든다.
        - 클래스나 인터페이스 이름에 일정한 규칙을 만들기가 어려운 경우 빈 이름을 이용하는 bean() 표현식을 시용. bean(*Service) 
    - 2. 공통된 메소드 이름 규칙을 통해 최소한의 트랜잭션 어드바이스와 속성을 정의한다
        - 모든 메소드에 대해 디폴트 속성을 지정
        - <tx:method name="*" />
        - 읽기전용 속성 추가
        - <tx:method name="get*" read-only=‘true" />
        - 일반화하기에는 적당하지 않은 특별한 트랜잭션 속성이 필요한 타깃 오브젝트에 대해서는 별도의 어드바이스와 포인트컷 표현식을 사용
    - 3. 프록시 방식 AOP는 같은 타깃 오브젝트 내의 메소드를 호출할 때는 적용 되지않는다
        - 프록시를 통한 부가 기능의 적용은 클라이언트로부터 호출이 일어날 때만 가능
        - 클라이언트 : 인터페이스를 통해 타깃 오브젝트를 사용하는 다른 모든 오브젝트
        - ![image](https://user-images.githubusercontent.com/32324250/60164951-72514f80-9839-11e9-8e6a-c7e7fe073ad1.png)

#### 6.6.4 트랜잭션 속성 적용
- 비즈니스 로직을 담고 있는 서비스 계층 오브젝트의 메소드가 트랜잭션 경계를 부여하기에 가장 적절
- 가능하면 다른 모률의 DAO에 접근할 때는 서비스 계층을 거치도록 하는 게 바람직
- 기존 포인트컷 표현식을 모든 비즈니스 로직의 서비스 빈에 적용되도록 수정
- ``` java
    <aop:config>
    <aop:advisor advice-ref="transactionAdvice" pointcut="bean(*Service)" />
    </aop :config>
    ``` 

#### 6.7. 애노테이션 트랜잭션 속성과 포인트컷
- 포인트컷 표현식과 트랜잭션 속성을 이용해 트랜잭션을 일괄적으로 적용하는 방식이 적용되기 힘든 복잡한 트랜잭션 속성을 설정하는 경우
- => 직접 타깃에 트랜잭션 속성정보를 가진 애노테이션을 지정

##### 6.7.1 트랜잭션 애노테이션
- @Transactional 
    - @Target ({ElementType .METHOD, ElementType .TYPE} ) : 애노테이션을 사용힐 대상올 지정. 한 개 이상의 대상을 지정할 수 있음
    - @Retention(RetentionPolicY .RUNTIME) : 애노테이선 정보가 언제까지 유지되는지를 지정. 런타임 때도 애노테이션 정보를 리율렉션을 통해 얻을 수 있다
    - @Inherited : 상속을 통해서도 애노테이션 정보를 얻올 수 있게 한다
    - 메소드， 클래스， 인터페이스에 시용할 수 있다.
    - 스프링은 @Transactional이 부여된 모든 오브젝트를 자동으로 타깃 오브젝트로 인식
    - TransactionAttributeSourcePointcut이 사용됨
        - 부여 된 빈 오브젝트를 모두 찾아서 포인트컷의 선정 결과로 돌려준다
        - 포인트컷의 자동 등록

- @Transactional 애노태이션을 시용했을 때 어드바이저의 동작방식
- ![image](https://user-images.githubusercontent.com/32324250/60165932-315a3a80-983b-11e9-8728-d2acb006ff64.png)
- @Transactional 애노테이션의 엘리먼트에서 트랜잭션 속성을 가져 오는 AnnotationTransactionAttributeSource를 시용
- 포인트컷도 @Transactional을 통한 트랜잭션 속성정보를 참조하도록 만든다
- 트랜잭션 부가기능 적용 단위는 메소드 -> 지저분해짐.. 동일한 속성 정보를 가진 애노테이션을 반복적으로 메소드마다 부여..

-  @Transactional을 적용할 때 4단계의 대체(fallback) 정책
    - 메소드의 속성을 확인할 때 
        - 1. 타깃 메소드
        - 2. 타깃 클래스
        - 3. 선언 메소드
        - 4. 선언 타입 (클래스, 인터메이스)
    - 의 순서에 따라서 @Transactional이 적용됐는지 차례로 확인하고， 가장 먼저 발견되는 속성정보를 사용
    - 끝까지 발견되지 않으면 해당 메소드는 트랜잭션 적용 대상이 아니라고 판단
    - @Transactional을 타깃 클래스보다는 인터페이스에 두는 게 바람직
    - why?? @Transactional 적용 대상은 클라이언트가 사용하는 인터페이스가 정의한 메소드. 구현 클래스가 바뀌더라도 트랜잭션 속성을 유지할 수 있음
    - 하지만 인터페이스를 사용히는 프록시 방식의 AOP가 아닌 방식으로 트랜잭션을 적용하면 인터페이스에 정의한 @Transactional은 무시되기 때문에 안전하게 타깃 클래스에 @Transactional을 두는 방법을 권장

- 트랜잭션 애노테이션 사용을 위한 설정 : <tx:annotation-driven />

#### 6.7.2 트랜잭션 애노테이션 적용
- JDBC를 직접 사용하는 기술의 경우는 트랜잭션이 없어도 DAO가 동작
- 장점 
    - IDE의 자동완성 기능을 활용할 수 있음
    - 속성을 잘못 지정한 경우 컴파일 에러가 발생해서 손쉽게 확인할 수 있음

#### 6.8. 트랜잭션 지원 테스트
#### 6.8.1 선언적 트랜잭션과 트랜잭션 전파 속성
 -  REQUIRED 전파 속성 : 앞에서 진행 중인 트랜잭션이 있으면 참여하고 없으면 자동으로 새로운 트랜잭션을 시작. 불필요한 코드 중복이 일어나지 않는다.
 - 선언적 트랜잭션 : AOP를 이용해 코드 외부에서 트랜잭션의 기능을 부여해주고 속성을 지정 할 수 있게 하는 방법 (바람직)
 - 프로그램에 의한 트랜잭션 : TransactionTemplate이나 개별 데이터 기술의 트랜잭션 API를 사용해 직접 코드 안에 서 사용히는 방법

#### 6.8.2 트랜잭션 동기화와 테스트 
- 트랜잭션 매니저(PlatformTransactionManager 인터페이스를 구현)를 통해 구체적인 트랜잭션 기술의 종류에 상관없이 일관된 트랜잭션 제어가 가능
- 트랜잭션 동기화 : 진행 중인 트랜잭션이 있는지 확인하고 트랜잭션 전파 속성에 따라서 이에 참여할 수 있도록 만들어주는 것
- 코드에서 트랜잭션 매니저를 이용해 트랜잭션을 제어하는 것도 가능 => 테스트
- ``` java 
    @Autowired
    PlatformTransactionManager transactionManager;
    ```
- 트랜잭션 매니저를 이용한 테스트용 트랜잭션 제어
    - 트랜잭션의 전파는 트랜잭션 매니저를 통해 트랜잭션 동기화 방식이 적용되기 때문에 가능
    - 트랜잭션 매니저를 이용해 트랜잭션을 시작시키고 이를 동기화해주면 트랜잭션이 한 번만 일어나게 할 수 있음
    - ``` java
        @Test
        public void transactionSync() (
            DefaultTransactionDefinition txDefinition = new DefaultTransactionDefinition(); 
            TransactionStatus txStatus = transactionManager.getTransaction(txDefinition); // 트랜잭션 매니저에게 트랜잭션율 요청

            userService .deleteAll();
            userService.add(users.get(0)); 
            userService.add(users.get(1));
        ```
    - 위의 코드에서 트랜잭션은 한 번만 일어난다.
    - ``` java
        public void transactionSync() (
        DefaultTransactionDefinition txDefinition =new DefaultTransactionDefinition();
        txDefinition.setReadOnly(true);
        ```
    - 위의 코드와 같이, 강제적으로 읽기전용 트랜잭션으로 설정할 수 있다
    - 읽기전용과 제한시간 등은 처음 트랜잭션이 시작할 때만 적용되고 그 이후에 참여하는 메소드의 속성은 무시된다

    - 롤백테스트 :  테스트 내의 모든 DB 작업을 하나의 트랜잭션 안에서 동작하게 하고 테스트가 끝나면 무조건 롤백해버리는 테스트
    - ``` java
        @Test
        public void transactionSync () {
        DefaultTransactionDefinition txDefinition =new DefaultTransactionDefinition();

            try {
            userService.deleteAll();
            userService.add(users.get(0)); 
            userService.add(users.get(1));
            }
            finally (
                transactionManager .rollback(txStatus);
            }
        }
    ```
    - 장점
        - 롤백 태스트는 DB 작업이 포함된 태스트가 수행돼도 DB에 영향을 주지 않음
        - MySQL에서는 통일한 작업을 수행한 뒤에 롤백하는 게 커맛하는 것보다 더 빠름

#### 6.8.3 테스트를 위한 트랜잭션 애노테이션
- @Transactional : 테스트에도 적용할 수 있음. 태스트용 트랜잭션은 테스트가 끝나면 자동으로 롤백됨. 테스트 클래스에 넣어서 모든 테스트 메소드에 일괄 적용가능
- @Rollback : 테스트에서 진행한 작업을 그대로 DB에 반영하고 싶을 경우 사용. 원치 않을 때는 @Rollback(false). 메소드 레벨에만 적용 가능
- @TransactionConfiguration : 테스트 클래스의 모든 메소드에 트랜잭션을 적용하면서 모든 트랜잭션이 롤백되지 않고 커밋되게 할 때 사용 
- Transactional(propagation=Propagation.NEVER) : 클래스 레벨의 @Transactional 설정을 무시하고 트랜잭션을 시작하지 않은 채로 테스트를 진행. 태스트 메소드에 사용
