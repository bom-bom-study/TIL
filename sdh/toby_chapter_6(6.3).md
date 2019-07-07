# AOP

### 6.3 다이내믹 프록시와 팩토리 빈

### 프록시와 프록시 패턴, 데코레이터 패턴
단순히 확장성을 고려해서 한 가지 기능을 분리한다면 전형적인 전략 패턴을 사용하면 된다.
* 트랜잭션 기능에는 추상화 작업을 통해 전략 패턴이 적용되어 있다.
* 전략 패턴으로는 트랜잭션 기능의 구현 내용을 분리해냈을 뿐이다.
* 트랜잭션을 적용한다는 사실은 코드에 그대로 남아있다.

트랜잭션이라는 기능은 사용자 관리 비즈니스 로직과는 성격이 다르기 때문에 아예 그 적용 사실 자체를 밖으로 분리할 수 있다.
* 이 방법을 이용해 UserServiceTx를 만들었다.
* UserServiceImpl에는 트랜잭션 관련 코드가 하나도 남지 않게 되었다.

분리된 부가기능을 담은 클래스는 중요한 특징이 있다.
* 부가기능 외의 나머지 모든 기능은 원래 핵심기능을 가진 클래스로 위임해줘야 된다.
* 핵심기능은 부가기능을 가진 클래스의 존재 자체를 모른다.
* 부가기능이 핵심기능을 사용하는 구조가 된다.

부가기능 코드에서는 핵심기능으로 요청을 위임해주는 과정에서 자신이 가진 부가적인 기능을 적용해 줄 수 있다.

프록시
* 클라이언트가 사용하려고 하는 실제 대상인 것처럼 위장해서 클라이언트의 요청을 받아주는 **대리자**, **대리인**과 같은 역할을 한다.

타깃(target) 또는 실체(real subject)
* 프록시를 통해 최종적으로 요청을 위임받아 처리하는 실제 오브젝트

프록시의 특징은 타깃과 같은 인터페이스를 구현했다는 것과 프록시가 타깃을 제어할 수 있는 위치에 있다는 것

프록시의 사용 목적
* 클라이언트가 타깃에 접근하는 방법을 제어하기 위해서
* 타깃에 부가적인 기능을 부여해주기 위해서
* 목적에 따라서 디자인 패턴에서는 다른 패턴으로 구분한다.
    * 데코레이터 패턴
    * 프록시 패턴

### 데코레이터 패턴
타깃에 **부가적인 기능**을 런타임 시 다이내믹하게 부여해주기 위해 프록시를 사용하는 패턴
* 컴파일 시점, 즉 코드상에서는 어떤 방법과 순서로 프록시와 타깃이 연결되어 사용되는지 정해져 있지 않다는 뜻
* 제품이나 케익 등을 여러 겹으로 포장하고 그 위에 장식을 해도 내용물은 동일하다는 뜻
* 프록시가 꼭 한개로 제한되지 않는다(프록시가 여러 개인 만큼 순서를 정해서 단계적으로 위임하는 구조로 만든다).

소스코드를 출력하는 기능(핵심 기능)
* 소스코드에 라인넘버를 붙여준다.
* 문법에 따라 색을 변경한다.
* 특정 폭으로 소스를 잘라 준다.
* 페이지를 표시한다.

데코레이터 패턴이 사용된 대표적인 예
* InputStream과 OutputStream 구현 클래스
    * InputStream을 구현한 타깃인 FileInputStream에 버퍼 읽기 기능을 제공해주는 BufferedInputStream이라는 데코레이터
    * InputStream is = new BufferedInputStream(new FileInputStream("a.txt"));

인터페이스를 통한 데코레이터 정의와 런타임 시의 다이내믹한 구성 방법은 스프링의 DI를 이용하면 아주 편리하다.

데코레이터 패턴은 인터페이스를 통해 위임하는 방식이기 때문에 어느 데코레이터에서 타깃으로 연결될지 코드 레벨에선 미리 알 수 없다.

데코레이터 패턴은 타깃의 코드를 손대지 않고, 클라이언트가 호출하는 방법도 변경하지 않은 채로 새로운 기능을 추가할 때 유용한 방법이다.

### 프록시 패턴
일반적으로 사용하는 프록시라는 용어와 디자인 패턴에서 말하는 프록시 패턴은 구분할 필요가 있다.
* 전자는 클라이언트와 사용 대상 사이에 대리 역할을 맡은 오브젝트를 두는 방법
* 후자는 프록시를 사용하는 방법 중에서 타깃에 대한 **접근 방법**을 제어하려는 목적을 가진 경우

프록시 패턴의 프록시는 타깃의 기능을 확장하거나 추가하지 않는다.
* 클라이언트가 타깃에 접근하는 방식을 변경해준다.
* 타깃 오브젝트를 생성하기 복잡하거나 당장 필요하지 않을 경우에는 꼭 필요한 시점까지 오브젝트를 생성하지 않는 편이 좋다.
* 클라에 타깃 레퍼런스를 넘겨야 하는데, 실제 타깃 대신 프록시를 넘겨준다. 그리고 프록시의 메소드를 통해 타깃을 사용하려고 시도하면, 그 때 프록시가 타깃 오브젝트를 생성하고 요청을 위임해주는 식이다.
* 레퍼런스는 갖고 있지만 끝까지 사용하지 않거나, 많은 작업이 진행된 후에 사용된다면, 프록시를 통해 최대한 생성을 늦춤으로써 얻는 장점이 많다.

원격 오브젝트를 이용하는 경우에도 프록시를 사용하면 편리하다.
* RMI, EJB, 각종 리모팅 기술을 이용해 다른 서버에 존재하는 오브젝트를 사용할 때, 원격 오브젝트에 대한 프록시를 만들어 두고, 클라는 마치 로컬에 존재하는 오브젝트를 쓰는 것처럼 프록시를 사용하게 할 수 있다.
* 프록시는 클라의 요청을 받으면 네트워크를 통해 원격의 오브젝트를 실행하고 결과를 받아 클라에 돌려준다.
* 클라로 하여금 원격 오브젝트에 대한 접근 방법을 제공해주는 프록시 패턴의 예라고 볼 수 있다.

특별한 상황에서 타깃에 대한 접근권한을 제어하기 위해 프록시 패턴을 사용할 수 있다.
* 수정 가능한 오브젝트가 있을 때, 특정 레이어로 넘어가서는 읽기 전용으로만 동작하도록 강제해야 할 때
* 프록시를 만들어 특정 메소드를 사용하려고 하면 접근 불가능하다고 예외를 발생시키면 된다.
* Collections의 unmodifiableCollection()

프록시와 데코레이터는 유사하다.
* 프록시는 코드에서 자신이 만들거나 접근할 타깃 클래스 정보를 알고 있는 경우가 많다.
* 생성을 지연하는 프록시라면 구체적인 생성 방법을 알아야 하기 떄문이다.
* 사용 할 때마다 사용의 목적이 부가인지, 접근 제어인지를 구분해보면 어떤 목적으로 프록시가 사용됐고, 어떤 패턴이 적용됐는지 알 수 있다.

### 다이내믹 프록시
많은 개발자는 타깃 코드를 직접 고치고 말지 번거롭게 프록시를 만들지 않겠다고 생각한다.
* 단위 테스트를 위해 목이나 스텁을 일일이 정의하는 것과 마찬가지다.

목 오브젝트를 만드는 불편함을 목 프레임워크를 사용해 편리하게 바꿨던 것처럼 프록시를 편리하게 만들어서 사용할 방법은 java.lang.reflect 패키지 안의 클래스들이다.
* 일일이 프록시 클래스를 정의하지 않고도 몇 가지 API를 이용해 프록시처럼 동작하는 오브젝트를 다이내믹하게 생성

### 프록시의 구성과 프록시 작성의 문제점
프록시는 다음의 두 가지 기능으로 구성된다.
* 타깃과 같은 메소드를 구현하고 있다가 메소드가 호출되면 타깃 오브젝트로 위임한다.
    * add(User user);
* 지정된 요청에 대해서는 부가기능을 수행한다.
    * upgradeLevels();

프록시를 만들기가 번거로운 이유
* 타깃의 인터페이스를 구현하고 위임하는 코드를 작성하기 번거롭다.
* 타깃 인터페이스의 메소드가 추가되거나 변경될 때마다 함께 수정해야 한다.
* 부가기능 코드가 중복될 가능성이 많다.

### 리플렉션
다이내믹 프록시는 리플렉션 기능을 이용해서 프록시를 만들어준다.
* 자바의 코드 자체를 추상화해서 접근하도록 만든 것

자바의 모든 클래스는 그 클래스 자체의 구성정보를 담은 Class 타입의 오브젝트를 하나씩 갖고 있다.
* '클래스이름.class'라고 하거나 오브젝트의 getClass() 메소드를 호출하면 클래스 정보를 담은 Class 타입의 오브젝트를 가져올 수 있다.
* 클래스 코드에 대한 메타정보를 가져오거나 오브젝트를 조회할 수 있다.
    * 클래스 이름, 어떤 클래스를 상속, 어떤 인터페이스를 구현, 어떤 타입의 필드를 갖고 있는지, 메소드는 어떤 것을 정의하고 어떤 파라미터와 리턴 값인지 알아낼 수 있다.

리플렉션 API 중에서 메소드에 대한 정의를 담은 Method라는 인터페이스가 있다.
Method lengthMethod = String.class.getMethod("length");
String이 가진 메소드 중에서 "length"라는 이름을 갖고 있고, 파라미터는 없는 메소드의 정보를 가져오는 것이다.

Method 인터페이스에 정의된 invoke()메소드를 통해 실행시킬 수 있다.
* 메소드를 실행시킬 대상 오브젝트와 파라미터 목록을 받아서 메소드를 호출하고 Object 타입으로 돌려준다.
public Object invoke(Object obj, Object... args);

int legnth = lengthMethod.invoke(name); // int length = name.legnth();

### 프록시 클래스

```java
public class HelloUppercase implements Hello {
    Hello hello; // 위임할 타깃 오브젝트 (다른 프록시를 추가할 수도 있으므로 인터페이스로 접근)

    public HelloUppercase(Hello hello) {
        this.hello = hello;
    }

    public String sayHello(String hello) {
        return hello.sayHello(name).toUpperCase(); // 위임과 부가기능 적용
    }

    public String sayHi(String name) {
        return hello.sayHi(name).toUpperCase();
    }

    ...
}
```

위임과 기능 부가라는 두 가지 프록시의 기능을 모두 처리하는 전형적인 프록시 클래스다.

이 프록시는 프록시 적용의 일반적인 문제점 두 가지를 모두 갖고 있다.
* 인터페이스의 모든 메소드를 구현해 위임하도록 코드를 만들어야 한다.
* 부가기능인 리턴 값을 대문자로 바꾸는 기능이 모든 메소드에 중복돼서 나타난다.

### 다이내믹 프록시 적용
다이내믹 프록시는 프록시 팩토리에 의해 런타임 시 다이내믹하게 만들어지는 오브젝트다.
* 다이내믹 프록시 오브젝트는 타깃의 인터페이스와 같은 타입으로 만들어진다.
* 클라는 다이내믹 프록시 오브젝트를 타깃 인터페이스를 통해 사용할 수 있다.
* 프록시를 만들 때 인터페이스를 모두 구현해가면서 클래스를 정의하는 수고를 덜 수 있다.
* 프록시 팩토리에게 인터페이스 정보만 제공해주면 해당 인터페이스를 구현한 클래스의 오브젝트를 자동으로 만들어준다.

다이내믹 프록시가 인터페이스 구현 클래스의 오브젝트는 만들어주지만, 프록시로서 필요한 부가기능 제공 코드는 직접 작성해야 한다.
* 부가기능은 프록시 오브젝트와 독립적으로 InvocationHandler를 구현한 오브젝트에 담는다.
* InvocationHandler 인터페이스는 한 개의 메소드를 가진 인터페이스이다.
public Object invoke(Object proxy, Method method, Object[] args)

다이내믹 프록시 오브젝트는 클라의 모든 요청을 리플렉션 정보로 변환해서 InvocationHandler 구현 오브젝트의 invoke() 메소드로 넘기는 것이다.
* 중복되는 기능을 효과적으로 제공할 수 있다. 

인터페이스를 제공하면서 프록시 팩토리에게 다이내믹 프록시를 만들어달라고 요청하면 인터페이스의 모든 메소드를 구현한 오브젝트를 생성해준다.

InvocationHandler 인터페이스를 구현한 오브젝트를 제공해주면 다이내믹 프록시가 받는 모든 요청을 InvocationHandler의 invoke() 메소드로 보내준다. 인터페이스의 메소드가 아무리 많더라도 invoke() 메소드 하나로 처리할 수 있다.

``` java
public class UppercaseHandler implements InvocationHandler {
    Hello target;

    public UppercaseHandler(Hello target) {
        this.target = target; // 다이내믹 프록시로부터 전달받은 요청을 다시 타깃 오브젝트에 위임하기 위해 주입 받는다.
    }

    public Object invoke(Object proxy, Method method, Object[] args) {
        String ret = (String)method.invoke(target, args); // 타깃으로 위임, 인터페이스의 메소드 호출에 모두 적용
        return ret.toUpperCase(); // 부가기능 제공
    }
}
```

이 InvocationHandler를 사용하고 Hello 인터페이스를 구현하는 프록시를 만들려면 Proxy 클래스의 newProxyInstance() 스태틱 팩토리 메소드를 이용하면 된다.

```java
Hello proxiedHello = (Hello)Proxy.newProxyInstance(
    getClass().getClassLoader(), // 동적으로 생성되는 다이내믹 프록시 클래스의 로딩에 사용할 클래스 로더
    new Class[] { Hello.class }, // 구현할 클래스
    new UppercaseHandler(new HelloTarget())); // 부가기능과 위임 코드를 담은 InvocationHandler
``` 

* 첫 번째 파라미터는 다이내믹 프록시가 정의되는 클래스 로더를 지정하는 것이다.
* 두 번째 파라미터는 다이내믹 프록시가 구현해야 할 인터페이스
    * 한 번에 하나 이상의 인터페이스를 구현할 수도 있기 때문에 배열로 사용
* 마지막 파라미터는 부가기능과 위임 관련 코드를 담고 있는 InvocationHandler 구현 오브젝트

### 다이내믹 프록시의 확장
다이내믹 프록시 방식이 직접 정의해서 만든 프록시보다 훨씬 유연하고 많은 장점이 있다.
* 인터페이스의 메소드가 3개가 아니라 30개로 늘어나도 UppercaseHandler와 다이내믹 프록시를 생성해서 사용하는 코드는 전혀 손댈게 없다.
* 지금은 모든 리턴 타입이 String으로 가정하지만, 다른 타입의 리턴 타입이 추가되게 된다면 리턴 타입을 확인해서 해당 타입에만 프록시를 적용하도록 할 수 있다.
* 타깃의 종류에 상관없이도 적용이 가능하다. 리플렉션의 Method 인터페이스를 이용해 타깃의 메소드를 호출하는 것이니 Hello 타입의 타깃으로 제한할 필요도 없다.
``` java
public class UppercaseHandler implements InvocationHandler {
    Object target;
    private UppercaseHandler(Object target) {
        this.target = target; // 어떤 종류의 인터페이스를 구현한 타깃에도 적용 가능하도록 Object 타입으로 수정
    }

    public Object invoke(Object proxy, Method metohd, Objectd[] args) {
        Object ret = method.invoke(target, args);
        if (ret instanceof String) {    // 리턴 타입이 String인 경우에만 적용
            return ((String)ret).toUpperCase();
        }
        else {
            return ret;
        }
    }
}
```

InvocationHandler는 단일 메소드에서 모든 요청을 처리하기 때문에 어떤 메소드에, 어떤 기능을 적용할지를 선택하는 과정이 필요할 수도 있다.
* 호출하는 메소드의 이름
* 파라미터의 개수와 타입
* 리턴 타입

메소드의 이름으로 조건을 걸면 다음과 같이 적용하면 된다.
``` java
if (ret instanceof String && method.getName().startsWith("say"))
```

### 다이내믹 프록시를 이용한 트랜잭션 부가기능
UserServiceTx는 서비스 인터페이스의 메소드를 모두 구현해야 하고 트랜잭션이 필요한 메소드마다 트랜잭션 처리코드가 중복돼서 나타나는 비효율적인 방법으로 만들어져 있다.
* 트랜잭션이 필요한 클래스와 메소드가 증가하면 프록시 클래스를 일일이 구현하는 것은 큰 부담이다.
* 따라서 트랜잭션 부가기능을 제공하는 다이내믹 프록시를 만들어 적용하는 방법이 효과적이다.

### 트랜잭션 InvocationHandler
트랜잭션 부가기능을 가진 핸들러의 코드는 다음과 같이 정의할 수 있다.
``` java
public class TransactionHandler implements InvocationHandler {
    private Object target;
    private PlatformTransactionManager transactionManager;
    private String pattern;

    // set...();

    public Object invoke(Object proxy, Method method, Object[] args) {
        if (method.getName().startsWith(pattern)) {
            return invokeInTransaction(method, args); // 적용 대상을 선별해서 기능을 부여
        } else {
            return method.invoke(target, args);
        }
    }

    private Object invokeInTransaction(Method method, Object[] args) {
        TransactionStatus status = this.transactionManager.getTransaction(new DefaultTransactionDefinition());
        try {
            Object ret = method.invoke(target, args); // 트랜잭션을 시작하고, 타깃 오브젝트의 메소드를 호출, 예외가 발생하지 않으면 커밋
            this.transactionManager.commit(status);
            return ret;
        } catch (InvocationTargetException e) {
            this.transactionManager. rollback(status); // 예외가 발생하면 롤백
            throw e.getTargetException();
        }
    }
}
```

 UserServiceTx보다 코드는 복잡하지 않으면서도 UserService뿐 아니라 모든 트랜잭션이 필요한 오브젝트에 적용 가능한 트랜잭션 프록시 핸들러가 만들어졌다.

### TransactionHandler와 다이내믹 프록시를 이용한 테스트
TransactionHandler가 UserServiceTx를 대신할 수 있는지 테스트를 확인해봐야 한다.

``` java
@Test
public void upgradeAllOrNothing() {
    ...

    TransactionHandelr txHandler = new TransactionHandler();
    txHandler.setTarget(testUserService);
    txHandler.setTransactionManager(transactionManager);
    txHandler.setPattern("upgradeLevels");
    UserService txUserService = (UserService)Proxy.newProxyInsatance(
                getClass().getClassLoader(), new Class[] { UserService.class}, txHandler);
```

만들어진 TransactionHandler를 이용해 UserService 타입의 다이내믹 프록시를 생성하고 테스트를 진행하면 된다.

### 다이내믹 프록시를 위한 팩토리 빈
TransactionHandler와 다이내믹 프록시를 스프링의 DI를 통해 사용할 수 있도록 만들어야 하는데, 다이내믹 프록시 오브젝트는 일반적인 스프링의 빈으로 등록할 방법이 없다.

스프링의 빈은 기본적으로 클래스 이름과 프로퍼티로 정의된다.
* 지정된 클래스 이름을 가지고 리플렉션을 이용해 해당 클래스의 오브젝트를 만든다.
* Class의 newInstance() 메소드는 해당 클래스의 파라미터가 없는 생성자를 호출하고 그 결과 생성되는 오브젝트를 돌려주는 리플렉션 API다.
* Date now = (Date) Class.forName("java.util.Date").newInstance();

다이내믹 프록시 오브젝트는 이런식으로 프록시 오브젝트가 생성되지 않는다. 사실 다이내믹 프록시 오브젝트의 클래스가 어떤 것인지 알 수도 없다.
* 사전에 프록시 오브젝트의 클래스 정보를 미리 알아내서 빈에 정의할 방법이 없다.
* Proxy 클래스의 newProxyInstance()라는 스태틱 팩토리 메소드를 통해서만 만들 수 있다.

### 팩토리 빈
스프링은 디폴트 생성자를 통해 오브젝트를 만드는 방법 외에도 빈을 만들 수 있는 여러 가지 방법을 제공한다.
* 대표적으로 팩토리 빈 생성 방법
    * 스프링을 대신해서 오브젝트의 생성로직을 담당하도록 만들어진 특별한 빈
    * 가장 간단한 방법은 스프링의 FactoryBean이라는 인터페이스를 구현하는 것
    * 구현된 인터페이스를 스프링의 빈으로 등록하면 팩토리 빈으로 동작함

```
<bean id="m" class="springbook.learningtest.spring.factorybean.Message">
```

* private 생성자를 가진 클래스의 직접 사용 금지
* 리플렉션을 통해 만들어주지만 접근 규약을 위반할 수 있기 때문에 강제로 생성하면 위험하다.

``` java
public class MessageFactoryBean implements FactoryBean<Message> {
    String text;

    public void setText(String text) {
        this.text = text;    // 오브젝트를 생성할 때 필요한 정보를 대신 DI 받게 한다.
    }

    @Override
    public Message getObject() {
        return Message.newMessage(this.text);    // 실제 빈으로 사용될 오브젝트를 직접 생성, 복잡한 방식의 오브젝트 생성과 초기화 작업도 가능
    }

    @Override
    public Class<? extends Message> getObjectType() {
        return Message.class;
    }

    @Override
    public boolean isSingleton() {
        return false;    // getObject()의 오브젝트가 싱글톤인지 알려준다. 
                         // 이 팩토리 빈은 매번 요청할 때마다 새로운 오브젝트를 만든다.
                         // 팩토리 빈의 동작방식에 관한 설정이고 만들어진 빈 오브젝트는 싱글톤으로 스프링이 관리해준다.
    }
}
```

팩토리 빈은 전형적인 팩토리 메소드를 가진 오브젝트이다.

### 팩토리 빈의 설정 방법
```
<bean id="message" class="springbook.learningtest.spring.factorybean.MessageFactoryBean">
    <property name="text" value="Factory Bean" />
</bean>
```

드물지만 팩토리 빈이 만들어주는 빈 오브젝트가 아니라 팩토리 빈 자체를 가져오고 싶을 경우는 '&'를 빈 이름 앞에 붙여주면 팩토리 자체를 돌려준다.
``` java
Object factory = context.getBean("&message");
```

### 다이내믹 프록시를 만들어주는 팩토리 빈
Proxy의 newProxyInstance() 메소드를 통해서만 생성이 가능한 다이내믹 프록시 오브젝트는 일반적인 방법으로는 스프링의 빈으로 등록할 수 없다.
* 대신 팩토리 빈을 사용하면 다이내믹 프록시 오브젝트를 스프링의 빈으로 만들어 줄 수가 있다.
* 팩토리 빈의 getObject() 메소드에 다이내믹 프록시 오브젝트를 만들어주는 코드를 넣으면 되기 때문이다.

<img src="img\factoryBean_proxy.png">

팩토리 빈, UserServiceImpl은 스프링 빈으로 등록하고, 다이내믹 프록시와 함께 생성할 TransactionHandler에게 전달해준다. 그 외에도 다이내믹 프록시나 TransactionHandler를 만들 때 필요한 정보는 팩토리 빈의 프로퍼티로 설정해뒀다가 다이내믹 프록시를 만들면서 전달해 줘야 한다.

### 트랜잭션 프록시 팩토리 빈
``` java
public class TxProxyFactoryBean implements FactoryBean<Object> {
    Object target;
    PlatformTransactionManager transactionManater;
    String pattern;
    Class<?> serviceInterface;

    // setter

    // FactoryBean 인터페이스 구현 메소드
    public Object getObject() {
        TransactionHandler txHandler = new TransactionHandler();
        txHandler.setTarget(target);
        // set...
        return Proxy.newProxyInstance(getClass().getClassLoader(), new Class[] { serviceInterface }, txHandler);
    }

    public Class<?> getObjectType() {
        return serviceInterface; // 팩토리 빈이 생성하는 오브젝트의 타입은 DI 받은 인터페이스 타입에 따라 달라진다. 따라서 다양한 타입의 프록시 오브젝트 생성에 재사용할 수 있다.
    }
    
    public boolean isSingleton() {
        return false; // 싱글톤 빈이 아니라는 뜻이 아니라 getObject()가 매번 같은 오브젝트를 리턴하지 않는다는 의미
    }
}
```

팩토리 빈이 만드는 다이내믹 프록시는 구현 인터페이스나, 타깃의 종류에 제한이 없다. 따라서 UserService 외에도 트랜잭션 부가기능이 필요한 오브젝트를 위한 프록시를 만들 때 얼마든지 재사용이 가능하다.
* 설정이 다른 여러 개의 TxProxyFactoryBean 빈을 등록하면 된다.

### 트랜잭션 프록시 팩토리 빈 테스트
트랜잭션 기능을 테스트 하기 위해 만든 upgradeAllOrNothing()은 예외 발생 시 트랜잭션이 롤백됨을 확인하기 위해 비즈니스 로직 코드를 수정한 TestUserService 오브젝트를 타깃 오브젝트로 대신 사용해야 한다.
* TransactionHandler와 다이내믹 오브젝트를 직접 만들어서 테스트 할 때는 바꾸기가 쉬웠다.
* 지금은 TxProxyFactoryBean 내부에서 만들어져 사용된다.
* 테스트용 설정을 별도로 만든다거나 프록시 팩토리 빈을 확장하는 방법도 있지만 빈으로 등록된 팩토리 빈을 가져와서 TestUserService 오브젝트를 주입하여 프록시를 만들면 된다.
* 빈을 가져와서 수정하는 학습 테스트이기 때문에 @DirtiesContext를 등록해주면 된다. 

``` java
public class UserServiceTest {
    @Autowired
    ApplicationContext context; 

    @Test
    @DirtiesContext
    public void upgradeAllOrNothing() {
        // TestUserService

        TxProxyFactoryBean txProxyFactoryBean = context.getBean("&userService", TxProxyFactoryBean.class); // 팩토리 빈 자체를 가져오기위해 &를 넣었고, 테스트용 타깃 주입

        txProxyFactoryBean.setTarget(testUserService);
        userService txUserService = (UserService) txProxyFactoryBean.getObject(); // 변경된 타깃 설정을 이용해서 트랜잭션 다이내믹 프록시 오브젝트를 다시 생성

    // ...
}
```

### 프록시 팩토리 빈 방식의 장점과 한계
다이내믹 프록시를 생성해주는 팩토리 빈을 사용하는 방법은 여러 가지 장점이 있다.
* 한번 부가기능을 가진 프록시를 생성하는 팩토리 빈을 만들어두면 타깃의 타입에 상관없이 재사용할 수 있다.

### 프록시 팩토리 빈의 재사용
TransactionHandler를 이용하는 다이내믹 프록시를 생성해주는 TxProxyFactoryBean은 코드의 수정 없이 타깃 오브젝트에 맞는 프로퍼티 정보를 설정해서 빈으로 등록해주기만 하면 코드의 수정 없이 다양한 클래스에 적용할 수 있다.
* 하나 이상의 TxProxyFactoryBean을 동시에 빈으로 등록해도 상관없다.
* 팩토리 빈이기 때문에 각 빈의 타입은 타깃 인터페이스와 일치한다.

CoreService 인터페이스에 트랜잭션 경계설정 기능을 부여해줄 필요가 있고, 인터페이스에 정의된 수십여개의 메소드에 트랜잭션을 모두 적용해야 한다고 했을 때 TxProxyFactoryBean을 그대로 적용해주면 된다.
```
<bean id="coreServiceTarget"  class="complex.module.CoreServiceImpl">
    <property name="coreDao" ref="coreDao" />
</bean>


<bean id="coreService" class="springbook.service.TxProxyFactoryBean">
    <property name="target" ref="coreServiceTarget" />
    <property name="transactionManager" ref="transactionManager" />
    <property name="pattern" ref="" />
    <property name="serviceInterface" ref="complex.module.CoreService" />
</bean>
```

### 프록시 팩토리 빈 방식의 장점
데코레이터 패턴이 적용된 프록시를 사용할 시 많은 장점이 있지만 적극적으로 활용되지 못하는 문제점
* 프록시를 적용할 대상이 구현하고 있는 인터페이스를 구현하는 프록시 클래스를 일일이 만들어야하는 번거로움
* 부가적인 기능이 여러 메소드에 반복적으로 코드 중복

프록시 팩토리 빈은 위 두가지 문제를 해결해준다.
* 타깃 인터페이스를 구현하는 클래스를 일일이 만드는 번거로움을 제거
* 하나의 핸들러 메소드를 구현하여 부가기능을 부여해줄 수 있어 코드 중복 문제 제거

다이내믹 프록시에 팩토리 빈을 이용한 DI까지 더해주면 번거로운 다이내믹 프록시 생성 코드도 제거할 수 있다.

### 프록시 팩토리 빈의 한계
더 욕심을 내서 중복 없는 최적화된 코드와 설정만을 이용해 이런 기능을 적용하려고 한다면 한계에 부딪힐 것이다.

프록시를 통해 타깃에 부가기능을 제공하는 것은 메소드 단위로 일어나는 일이다. 하나의 클래스 안에 존재하는 여러 개의 메소드에 부가기능을 한 번에 제공하는 건 어렵지 않지만, 한 번에 여러 개의 클래스에 공통적인 부가기능을 제공하는 일은 지금까지의 방법으로 불가능하다.
* 하나의 타깃 오브젝트에만 부여되는 부가기능은 상관없다.
* 트랜잭션과 같은 로직을 담은 많은 클래스의 메소드에 적용할 필요가 있다면 팩토리 빈의 설정이 중복되는 것을 막을 수 없다. 
* 하나의 타깃에 여러 개의 부가기능을 적용하려고 할 때도 문제다.

또 다른 문제점은 TransactionHandler 오브젝트가 프록시 팩토리 빈 개수만큼 만들어진다.
* TransactionHandler는 타깃 오브젝트를 프로퍼티로 갖고 있다.
* 트랜잭션 부가기능을 제공하는 동일한 코드임에도 불구하고 타깃 오브젝트가 달라지면 새로운 TransactionHandler 오브젝트를 만들어야 한다.
* TransactionHandler는 다이내믹 프록시처럼 굳이 팩토리 빈에서 만들지 않고, 스스로 빈으로 등록될 수도 있다.
* 하지만 타깃 오브젝트가 다르기 때문에 타깃 오브젝트 개수만큼 다른 빈으로 등록해야 하고 그만큼 많은 오브젝트가 생긴다.
* 타깃 오브젝트 외의 설정이 필요하다면 같은 설정이 중복돼서 많은 빈에 나타난다.

TransactionHandler의 중복을 없애고 모든 타깃에 적용 가능한 싱글톤 빈으로 적용할 수 없을지는 스프링의 DI를 생각하면 도전해볼 만하다.