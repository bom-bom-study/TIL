# AOP

### 6.4 스프링의 프록시 팩토리 빈

### ProxyFactoryBean
자바에는 JDK에서 제공하는 다이내믹 프록시 외에도 편리하게 프록시를 만들 수 있도록 지원해주는 다양한 기술이 존재한다.

스프링의 ProxyFactoryBean은 프록시를 생성해서 빈 오브젝트로 등록하게 해주는 팩토리 빈이다.
* 기존에 만들었던 TxProxyFactoryBean과 달리, ProxyFactoryBean은 순수하게 프록시를 생성하는 작업만을 담당하고 프록시를 통해 제공해줄 부가기능은 별도의 빈에 둘 수 있다.
* 부가기능은 MethodInterceptor 인터페이스를 구현해서 만든다.
    * InvocationHandler의 invoke() 메소드는 타깃 오브젝트에 대한 정보를 제공하지 않아서 타깃을 직접 알고 있어야 한다.
    * MethodInterceptoer의 invoke() 메소드는 ProxyFactoryBean으로부터 타깃 오브젝트에 대한 정보도 함께 받는다.
    * 그래서 타깃 오브젝트에 상관없이 독립적으로 만들어질 수 있고, 타깃이 다른 여러 프록시에서 함께 사용가능하다.
    * 싱글톤 빈으로 등록 가능하다.

### 어드바이스: 타깃이 필요 없는 순수한 부가기능
MethodInterceptor로는 메소드 정보와 함께 타깃 오브젝트가 담긴 MethodInvocation 오브젝트가 전달된다.
* 타깃 오브젝트의 메소드를 실행할 수 있는 기능이 있기 때문에 부가기능을 제공하는 데만 집중할 수 있다.
* MethodInvocation은 일종의 콜백 오브젝트로, proceed() 메소드를 실행하면 타깃 오브젝트의 메소드를 내부적으로 실행해주는 기능이 있다.
* ProxyFactoryBean은 작은 단위의 템플릿/콜백 구조를 응용해서 적용했기 때문에 템플릿 역할을 하는 MethodInvocation을 싱글톤으로 두고 공유할 수 있다.

MethodInterceptor 오브젝트를 추가하는 메소드 이름은 addMethodInterceptor가 아니라 addAdvice다.
* Advice 인터페이스를 상속하고 있는 서브 인터페이스이기 때문이다.
* 스프링은 단순히 메소드 실행을 가로채는 방식 외에도 부가기능을 추가하는 여러 가지 다양한 방법을 제공하낟.
* 타깃 오브젝트에 적용하는 부가기능을 담은 오브젝트를 스프링에서는 어드바이스(advice)라고 부른다.

> 어드바이스는 타깃 오브젝트에 종속되지 않는 순수한 부가기능을 담은 오브젝트다.

### 포인트컷: 부가기능 적용 대상 메소드 선정 방법
기존의 InvocationHandler를 직접 구현 했을 때는 부가 기능 적용 외에도 한 가지 작업이 더 있었다.
* pattern이라는 메소드 이름 비교용 스트링 값을 DI받아서 부가기능 적용 대상 메소드를 선정하는 것

ProxyFactoryBean과 MethodInterceptor를 사용하는 방식에서도 메소드 선정 기능을 넣을 수 있을까?
* MethodInterceptor 오브젝트는 여러 프록시가 공유해서 사용할 수 있다. 
* 그러기 위해서 MethodInterceptor 오브젝트는 타깃 정보를 갖고 있지 않도록 만들었다.
* 그 덕분에 싱글톤 빈으로 등록할 수 있었다.
* 트랜잭션 적용 메소드 패턴은 프록시마다 다를 수 있기 때문에 MethodInterceptor에 특정 프록시만 적용되는 패턴을 넣으면 문제가 된다.
* MethodInterceptor를 프록시마다 따로 등록하고 독립적이게 만들지 말고 함께 두기 곤란한 성격이 다르고 변경 이유와 시점이 다르고, 생성 방식과 의존관계가 다른 코드가 있다면 분리해주면 된다.

프록시에 부가기능 적용 메소드를 선택하는 기능을 전략 패턴을 이용해 적용하면 된다.

기존 TxProxyFactoryBean의 InvocationHandler도 다이내믹 프록시와 부가기능을 분리할 수 있고, 부가기능 적용 대상 메소드를 선정할 수 있게 되어 있다.
* 문제는 InvocationHandler가 타깃과 메소드 선정 알고리즘 코드에 의존하고 있다는 점이다.
* 타깃, 메소드 선정 방식이 다르다면 InvocationHandler 오브젝트를 여러 프록시가 공유할 수 없다.
* 타깃, 메소드 선정 알고리즘은 DI를 통해 분리할 수 있지만 한번 빈으로 등록된 오브젝트는, 오브젝트 차원에서 특정 타깃을 위한 프록시에 제한된다는 뜻이다.
* 그래서 InvocationHandler는 굳이 따로 빈으로 등록하는 대신 TxProxyrFactoryBean 내부에서 매번 생성하도록 만들었던 것이다.
* 결국 OCP 원칙을 깔끔하게 잘 지키지 못하는 어정쩡한 구조라고 볼 수 있다.

반면 스프링의 ProxyFactoryBean 방식은 두 가지 확장 기능인 부가기능(Advice)와 메소드 선정 알고리즘(Pointcut)을 활용하는 유연한 구조를 제공한다.

<img src="img\Spring_ProxyFactoryBean.png">

> 어드바이스 : 부가기능을 제공하는 오브젝트
> 포인트컷 : 메소드 선정 알고리즘을 담은 오브젝트

어드바이스와 포인트컷 모두 프록시에 DI로 주입돼서 사용된다.
* 두 가지 모두 여러 프록시에서 공유가 가능하도록 만들어지기 때문에 스프링의 싱글톤 빈으로 등록이 가능하다.

* 프록시는 클라이언트로부터 요청을 받으면 먼저 포인트컷에게 부가기능을 부여할 메소드인지를 확인해달라고 요청한다.
    * 포인트컷은 Pointcut 인터페이스를 구현해서 만들면 된다.
* 프록시는 포인트컷으로부터 부가기능을 적용할 대상 메소드인지 확인받으면, MethodInterceptor 타입의 어드바이스를 호출한다.
* JDK 다이내믹 프록시의 InvocationHandler와 달리 직접 타깃을 호출하지 않는다. 자신이 공유돼야하므로 타깃 정보라는 상태를 가질 수 없다.
* 따라서 직접 의존하지 않도록 일종의 템플릿 구조로 설계되어 있다.
* 어드바이스가 부가기능을 부여하는 중에 타깃 메소드의 호출이 필요하면 프록시로부터 전달받은 
MethodInvocation 타입 콜백 오브젝트의 proceed() 메소드를 호출해주기만 하면 된다.

실제 위임 대상인 타깃 오브젝트의 레퍼런스를 갖고 있고, 이를 이용해 타깃 메소드를 직접 호출하는 것은 프록시가 메소드 호출에 따라 만드는 Invocation 콜백의 역할이다.
* 어드바이스가 일종의 템플릿이 되고 타깃을 호출하는 기능을 갖고 있는 MethodInvocation 오브젝트가 콜백이 되는 것이다.
* 어드바이스도 도긻적인 싱글톤 빈으로 등록하고 DI를 주입해서 여러 프록시가 사용하도록 만들 수 있다.

> 재사용 가능한 기능은 만들어두고 바뀌는 부분(콜백 오브젝트와 메소드 호출정보)만 외부에서 주입해서 이를 작업 흐름(부가기능 부여) 중에 사용하도록 하는 전형적인 템플릿/콜백 구조다.

프록시로부터 어드바이스와 포인트컷을 독립시키고 DI를 사용하게 한 것은 전형적인 전략 패턴 구조다.

```java
@Test
public void pointcutAdvisor() {
    ProxyFactoryBean pfBean = new ProxyFactoryBean();
    pfBean.setTarget(new HelloTarget());

    NameMatchMethodPointcut pointcut = new NameMatchMethodPointcut(); // 메소드 이름을 비교해서 대상을 선정하는 알고리즘을 제공하는 포인트컷 생성
    pointcut.setMappedName("sayH*");

    pfBean.addAdvisor(new DefaultPointcutAdvisor(pointcut, new UppercaseAdvice())); // 포인트컷, 어드바이스를 Advisor로 묶어서 한번에 추가

    Hello proxiedHello = (Hello) pfBean.getObject();

    // assetThat
}
```

포인트컷이 필요 없을 때는 addAdvice() 메소드를 호출해서 어드바이스만 등록하면 된다.

포인트컷을 함께 등록할 때는 어드바이스와 포인트 컷을 Advisor 타입으로 묶어서 addAdvisor() 메소드를 호출해야 한다.
* 같이 묶어서 등록하는 이유는 ProxyFactoryBean에는 여러 개의 어드바이스와 포인트컷이 추가될 수 있기 때문이다.
* 따로 등록하면 어드바이스(부가기능)에 대해 어떤 포인트컷(메소드 선정)을 적용할지 애매해지기 때문이다.
* 이렇게 어드바이스와 포인트컷을 묶은 오브젝트를 인터페이스 이름을 따서 어드바이저라고 부른다.
* 어드바이저 = 포인트컷(메소드 선정 알고리즘) + 어드바이스(부가기능)

### ProxyFactoryBean 적용
JDK 다이내믹 프록시 방식으로 만든 부분을 제거하고 MethodInterceptor라는 Advice 서브인터페이스를 구현해서 만들면 된다.

``` java
public class TransactionAdvice implements MethodInterceptor { // 스프링의 어드바이스 인터페이스 구현
    PlatfoormTransactionManager transactionManager;

    // set 

    public Object invoke(MethodInvocation invocation) throw Throwable {
        TransactionStatus status = this.transactionManager.getTransaction(new DefaultTransactionDefinition());
        try {
            Object ret = invocation.proceed();
            this.transactionManager.commit(status);
            return ret;
        } catch (RuntimeException e) {
            this.transactionManager.rollback(status);
            throw e;
        }
    }
}
```

public Object invoke(Method invocation)
* 타깃을 호출하는 기능을 가진 콜백 오브젝트를 프록시로부터 받는다. 덕분에 어드바이스는 특정 타깃에 의존하지 않고 재사용 가능하다.

Object ret = invocation.proceed();
* 콜백을 호출해서 타깃의 메소드를 실행시킨다.
* 타깃 메소드 호출 전후로 필요한 부가기능을 넣을 수 있다.
* 경우에 따라서 타깃이 아예 호출되지 않게 하거나 재시도를 위한 반복적인 호출도 가능하다.