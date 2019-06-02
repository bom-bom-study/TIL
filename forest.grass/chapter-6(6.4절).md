# toby-spring

## 05월 15일

### keyword

#### 다이내믹 프록시와 팩토리 빈

#### 프록시와 프록시 패턴, 데코레이터 패턴
- 트랜잭션 경계설정 코드를 비즈니스 로직 코드에서 분리해낼 때 적용했던 기법을 다시 검토해보자
- 단순히 확장성을 고려해서 한 가지 기능을 분리한다면 전형적인 전략 패턴을 사용하면 된다.

![AOP-2](/forest.grass/img/AOP-2.png)
- 전략 패턴으로는 트랜잭션 기능의 구현 내용을 분리해닜을 뿐이다. 트랜잭션을 적용한다는 사실은 코드에 그대로 남아 있다.
- 위 그림은 트랜잭션과 같은 부가적인 기능을 위임을 통해 외부로 분리했을 때의 결과를 보여준다. 
- 구체적인 구현 코드는 제거 했을지라도 위임을 통해 기능을 상용하는 코드는 핵심 코드와 함계 남아 있다.
- 트랜잭션이라는 기능은 사용자 관리 비즈니스 로직과는 성격이 다르기 때문에 아예 그적용 사실 자체를 밖으로 분리할 수 있다.

![AOP-3](/forest.grass/img/AOP-3.png)
- 위 그림과 같이 부가기능 전부를 핵심 코드기 딤긴 클래스에서 독립시킬 수 있다.
- 이렇게 분리된 부가기능을 담은 클래스는 중요한 특징이 있다. 부가기능 외의 나머지 모든 기능은 원래 핵심기능을 가진 클래스로 위임해줘야한다.
- 핵심 기능은 부가기능을 가진 클래스의 존재 자체를 모른다. 따라서 부가기능이 핵심기능을 사용하는 구조가되는 것이다.

![AOP-4](/forest.grass/img/AOP-4.png)
- 부가기능은 마치 자신이 핵심 기능을 가진 클래스인 것처럼 꾸며서, 클라이언트가 자신을 거쳐서 핵심기능을 사용하도록 만들어야 한다.
- 그러기 위해서는 클라이언트는 인터페이스를 통해서만 핵심기능을 사용하게 하고, 부가기능 자신도 같은 인터페이스를 구현한 뒤에 자신이 그사이에 끼어들어야 한다.
- 부가기능 코드에서는 핵심기능으로 요청을 위임해주는 과정에서 자신이 가진 부가적인 기능을 적용해줄 수 있다.
- 비즈니스 로직 코드에 트랜잭션 기능을 부여해주는 것이 바로 그런 대표적인 경우다.

![AOP-5](/forest.grass/img/AOP-5.png)
- 마치 자신이 클라이언트가 사용하려고 하는 실제 대상인 것처럼 위장해서 클라이언트의 요청을 받아주는 것을 대리자, 대리인과 같은 역할을 한다고 해서 프록시라고 부른다.
- 그리고 프록시를 통해 최종적으로 요청을 위임받아 처리하는 실제 오브젝트를 타킷 또는 실체라고 부른다.
- 프록시의 특징은 타깃과 같은 인터페이스를 구현했다는 것과 프록시가 타깃을 제어할 수 있는 위치에 있다는 것이다.
- 프록시의 사용 목적 두가지
  - 1.첫째는 클라이언트가 타깃에 접근하는 방법을 제어하기 위해서다
  - 2.두 번째는 타깃에 부가적인 기능을 부여해주기 위해서다
- 두가지 모두 대리 오브젝트라는 개념의 프록시를 두고 사용한다는 점은 동일 하지만, 목적에 따라서 디자인 패턴에서는 다른 패턴으로 구분한다.

#### 데코레이터 패턴
- 데코레이터 패턴은 타깃에 부가적인 기능을 런타임 시 다이내믹하게 부여해주기 위해 프록시를 사용하는 패턴을 말한다.
- 다이내믹하게 기능을 부가한다는 의미는 컴파일 시점, 즉 코드상에서는 어떤 방법과 순서로 프록시와 타깃이 연결되어 사용되는지 정해져 있지 않다는 뜻이다.
- 데코레이터 패턴에서는 프록시가 꼭 한 개로 제한되지 않는다.
- 프록시가 직접 타깃을 사용하도록 고정시킬 필요도 없다.
- 데코레이터 패턴에서는 같은 인터페이스를 구현한 타켓과 여러 개의 프록시를 사용할 수 있다.
- 프록시가 여러 개인 만큼 순서를 정해서 단계적으로 위임하는 구조로 만들면 된다.
![AOP-6](/forest.grass/img/AOP-6.png)
- 프록시로서 동작하는 각 데코레이터는 위임하는 대상에도 인터페이스로 접근하기 때문에 자신이 최종 타킷으로 위임하는지, 아니면 다음 단계의 데코레이터 프록시로 위임하는지 알지 못한다.
- 그래서 데코레이터의 다음 위임 대상은 인터페이스로 선언하고 생성자나 수정자 메소드를 통해 위임 대상을 외부에서 런타임 시에 주입받을 수 있도록 만들어야 한다.
- ex)자바 IO 패키지의 InputStream과 OutputStream 구현 클래스는 데코레이터 패턴이 사용된 대표적인 예이다.
- UserService 인터페이스를 구현한 타깃인 UserServiceImpl에 트랜잭션 부가기능을 제공해주는 UserServiceTx를 추가한 것도 데코레이터 패턴을 적용한 것이라고 볼 수 있다.
- 데코레이터 패턴은 인터페이스를 통해 위임하는 방식이기 때문에 어느 데코레이터에서 타깃으로 연결될지 코드 레벨에서 미리 알 수 없다.
- 구성하기에 따라서 여러 개의 데코레이터를 적용할 수도 있다.
- 데코레이터 패턴은 타깃의 코드를 손대지 않고, 클라이언트가 호출하는 방법도 변경하지 않은 채로 새로운 기능을 추가할 때 유용한 방법이다.

#### 프록시 패턴
- 일반적으로 사용하는 프록시라는 용어와 디자인 패턴에서 말하는 프록시 패턴은 구분할 필요가 있다.
- 프록시
  - 클라이언트와 사용 대상 사이에 대리 역할을 맡은 오브젝트를 두는 방법을 총칭
- 프록시 패턴
  - 프록시를 사용하는 방법 중에서 타깃에 대한 접근 방법을 제어하려는 목적을 가진 경우
- 프록시 패턴의 프록시는 타깃의 기능을 확장하거나 추가하지 않는다. 대신 클라이언트가 타깃에 접근하는 방식을 변경해준다.
- 타깃 오브젝트를 생성하기가 복잡하거나 당장 필요하지 않은 경우에는 꼭 필요한 시점까지 오브젝트를 생성하지 않는 편이 좋다.
- 그런데 타깃 오브젝트에 대한 레퍼런스가 미리 필요할 수 있다. 이럴 때 프록시 패턴을 적용하면 된다.
- 클라이언트에게 타깃에 대한 레퍼런스를 넘겨야 하는데, 실제 타깃 오브젝트는 만드는 대신 프록시를 넘겨주는 것이다. 그리고 프록시의 메소드를 통해 타깃을 사용하려고 시도하면, 그때 프록시가 타깃 오브젝트를 생성하고 요청을 위임해주는 식이다.
- 프록시를 통해 생성을 최대한 늦춤으로써 얻는 장점이 많다.
  - 레퍼런스는 갖고 있지만 끝까지 사용하지 않거나
  - 많은 작업이 진행된 후에 사용되는 경우
  - 원격 오브젝트를 이용하는 경우에도 프록시를 사용하면 편리하다.(RMI, EJB, 각종 리모팅 기술을 이용해서 다른 서버에 존재하는 오브젝트를 사용할 경우)
- 특별한 상황에서 타깃에 대한 접근권한을 제어하기 위해 프록시 패턴을 사용할 수 있다.
  - 수정 가능한 오브젝트가 있는데, 특정 레이어로 넘어가서는 읽기전용으로만 동작하게 강제해야 한다고 하자
  - 이럴 때는 오브젝트의 프록시를 만들어서 사용할 수 있다.
  - 프록시의 특정 메소드를 사용하려고 하면 접근이 불가능하다고 예외를 발생시키면 된다.
- 프록시 패턴은 타깃은 기능자체에는 관여하지 않으면서 접근하는 방법을 제어해주는 프록시를 이용하는 것이다.
- 구조적으로 보자면 프록시와 데코레이터는 유사하다. 다만 프록시는 코드에서 자신이 만들거나 접근할 타깃 클래스 정보를 알고 있는 경우가 많다.
- 생성을 지연하는 프록시라면 구체적인 생성방법을 알아야 하기 때문에 타깃 클래스에 대한 직접적인 정보를 알아야 한다.
- 프록시 패턴이라고 하더라도 인터페이스를 통해 위임하도록 만들수도 있다.
![AOP-7](/forest.grass/img/AOP-7.png)
- 토비에서의 프록시
  - 타깃과 동일한 인터페이스를 구현하고 클라이언트와 타깃 사이에 존재하면서 기능의 부가 또는 접근 제어를 담당하는 오브젝트를 모두 프록시라고 부르겠다.
  - 사용의 목적이 기능의 부가인지, 접근 제어인지를 구분해보면 각각 어떤 목적으로 프록시가 사용됐는지, 그에 따라 어떤 패턴이 적용됐는지 알 수 있을 것이다.

#### 다이내믹 프록시
- 프록시는 기존 코드에 영향을 주지 않으면서 타깃의 기능을 확장하거나 접근 방법을 제어할 수 있는 유용한 방법이다.
- 프록시를 만드는 일이 상당히 번거롭다
- 매번 새로운 클래스를 정의해야 하고, 인터페이스의 구현해야 할 메소드는 많으면 모든 메소드를 일일히 구현해서 위함하는 코드를 넣어야 하기 때문이다.
- 자바에는 java.lang.reflect 패키지 안에 프록시를 손쉽게 만들 수 있도록 지원해주는 클래스들이 있다.

#### 프록시 구성과 프록시 작성의 문제점
- 프록시의 두가지 기능
  - 타깃과 같은 메소드를 구현하고 있다가 메소드가 호출되면 타깃 오브젝트로 위임한다.
  - 지정된 요청에 대해서는 부가기능을 수행한다.
- 프록시를 만들기가 번거로운 이유 두가지
  - 1.타깃의 인터페이스를 구현하고 윔하는 코드를 작성하기가 번거롭다는 점이다. 부가기능이 필요 없는 메소드도 구현해서 타깃으로 윔하는 코드를 일일이 만들어줘야 한다.
  - 2.부가기능 코드가 중복될 가능성이 많다는 점이다. 트랜잭션은 DB를 사용하는 대부분의 로직에 적용될 필요가 있다. 트랜잭션을 적용하는 부가기능 코드가 중복되어 많아 질 수가 있다.

#### 리플렉션
- 다이내믹 프록시는 리플렉션 기능을 이용해서 프록시를 만들어준다. 리플렉션은 자바의 코드 자체를 추상해서 접근하도록 만든 것이다.
- 클래스 오브젝트를 이용하면 클래스 코드에 대한 메타정보를 가져오거나 오브젝트를 조작할 수 있다.
- 예를 들어 클래스의 이름이 무엇이고, 어떤 클래스를 상속하고, 어떤 인터페이스를 구현했는지, 어떤 필드를 갖고 있고, 각각의 타입은 무엇인지, 메소드는 어떤 것을 정의했고, 메소드의 파라미터와 리턴 타입은 무엇인지를 알아낼 수 있다.
- 더 나아가서 오브젝트 필드의 값을 일고 수정할 수도 있고, 원하는 파라미터 값을 이용해 메소드를 호출할 수도 있다.
```{.java}
@Test
	public void invokeMethod() throws Exception {
		String name = "Spring";

		// length
		assertThat(name.length(), is(6));
		
		Method lengthMethod = String.class.getMethod("length");
		assertThat((Integer)lengthMethod.invoke(name), is(6));
		
		// charAt()
		assertThat(name.charAt(0), is('S'));
		
		Method charAtMethod = String.class.getMethod("charAt", int.class);
		assertThat((Character)charAtMethod.invoke(name, 0), is('S'));
	}
```

#### 프록시 클래스
```{.java}
  interface Hello {
		String sayHello(String name);
		String sayHi(String name);
		String sayThankYou(String name);
	}

  //타깃
  class HelloTarget implements Hello {
		public String sayHello(String name) {
			return "Hello " + name;
		}

		public String sayHi(String name) {
			return "Hi " + name;
		}

		public String sayThankYou(String name) {
			return "Thank You " + name;
		}
	}

  //프록시
  class HelloUppercase implements Hello {
		Hello hello;
		
		public HelloUppercase(Hello hello) {
			this.hello = hello;
		}

		public String sayHello(String name) {
			return hello.sayHello(name).toUpperCase();
		}

		public String sayHi(String name) {
			return hello.sayHi(name).toUpperCase();
		}

		public String sayThankYou(String name) {
			return hello.sayThankYou(name).toUpperCase();
		}
		
	}

@Test
	public void simpleProxy() {
	  Hello proxiedHello = new HelloUppercase(new HelloTarget());
		assertThat(proxiedHello.sayHello("Toby"), is("HELLO TOBY"));
		assertThat(proxiedHello.sayHi("Toby"), is("HI TOBY"));
		assertThat(proxiedHello.sayThankYou("Toby"), is("THANK YOU TOBY"));
	}

```
- 두가지의 문제점이 발생한다.
- 인터페이스의 모든 메소드를 구현해 위임하도록 코드를 만들어야 한다.
- 부가기능인 리턴 값을 대문자로 바꾸는 기능이 모든 메소드에 중복돼서 나타난다.

#### 다이내믹 프록시 적용
![AOP-8](/forest.grass/img/AOP-8.png)
- 다이내믹 프록시는 프록시 팩토리에 의해 런타임 시 다이내믹하게 만들어지는 오브젝트다.
- 다이내믹 프록시 오브젝트는 타깃의 인터페이스와 같은 타입으로 만들어진다.
- 클라이언트는 다이내믹 프록시 오브젝트를 타깃 인터페이스를 통해 사용할 수 있다.
- 이 덕분에 프록시를 만들 때 인터페이스를 모두 구현해가면서 클래스를 정의하는 수고를 덜 수 있다.
- 다이내믹 프록시가 인터페이스 구현 클래스의 오브젝트는 만들어주지만, 프록시로서 필요한 부가기능 제공 코드는 직접 작성해야 한다.
- 부가기능은 프록시 오브젝트와 독립적으로 InvocationHandler를 구현한 오브젝트에 담는다.
```{.java}
public Object invoke(Object proxy, Method method, Object[] args)
```
- 다이내믹 프록시 오브젝트는 클라이언트의 모든 요청을 리플렉션 정보로 변환해서 InvocationHandler 구현 오브젝트의 invoke() 메소드로 넘기는 것이다.
- 타깃 인터페이스의 모든 메소드 요청이 하나의 메소드로 집중되기 때문에 중복되는 기능을 효과적으로 제공할 수 있다.
- 리플렉션으로 메소드와 파라미터 정보를 모두 갖고 잇으므로 타깃 오브젝트의 메소드를 호출하게 할 수도 있다.
- InvocationHandeler 구현 오브젝트가 타깃 오브젝트 레퍼런스를 갖고 있다면 리플렉션을 이용해 간단히 위임 코드를 만들어 낼 수 있다.
![AOP-9](/forest.grass/img/AOP-9.png)
- Hello 인터페이스를 제공하면서 프록시 팩토리에게 다이내믹 프록시를 만들어 달라고 요청하면 Hello 인터페이스의 모든 메소드를 구현한 오브젝트를 생성해준다.
- InvocationHandler 인터페이스를 구현한 오브젝트를 제공해주면 다이내믹 프록시가 받은 모든 요청을 InovationHandler의 invoke() 메소드로 보내준다.
- Hello 인터페이스의 메소드가 아무리 많더라도 invoke() 메소드 하나로 처리할 수 있다.

#### 다이내믹 프록시를 만들어 보자
```{.java}
  public class UppercaseHandler implements InvocationHandler {
		Hello taget

		private UppercaseHandler(Hello target) {
			this.target = target;
		}

		public Object invoke(Object proxy, Method method, Object[] args)
			throws Throwable {
			String ret = (String)method.invoke(target, args);
			return ret.toUpperCase();
		}
	}

	Hello proxiedHello = (Hello)Proxy.newProxyInstance(
			getClass().getClassLoader(), 
			new Class[] { Hello.class},
			new UppercaseHandler(new HelloTarget()));
```
- 다이내믹 프록시부터 요청을 전달 받으려면 InvocationHandler를 구현해야 한다.
- 메소드는 invoke() 하나뿐이다. 다이내믹 프록시가 클라이언트로부터 받은 모든 요청은 invoke() 메소드로 전달된다.
- 다이내믹 프록시를 통해 요청이 전달되면 리플렉션API를 이용해 타깃 오브젝트의 메소드를 호출한다.
- 타깃 오브젝트는 생성자를 통해 미리 전달 받아 둔다.
- 다이내믹 프록시의 생성은 Proxy 클래스의 newProxyInstance() 스태틱 팩토리 메소드를 이용하면 된다.
- 사용하는 방법
  - 1. 첫번째 파라미터는 클래스 로더를 제공해야 한다. 다이내믹 프록시가 정의되는 클래스 로더를 지정하는 것이다.
  - 2. 두번째 파라미터는 다이내믹 프록시가 구현해야 할 인터페이스다. 다이내믹 프록시는 한 번에 하나 이상의 인터페이스를 구현 할 수도 있다. 따라서 인터페이스의 배열을 사용한다.
  - 3. 마지막 파리미터는 부가기능과 위임 관련 코드를 담고 있는 InvocationHandler 구현 오브젝트를 제공 해야한다. 타깃 오브젝트와, 부가기능이 있는 오브젝트를 넘겨준다.

#### 다이내믹 프록시 확장
- 다이내믹 프록시 방식이 직접 정의해서 만든 프록시보다 훨씬 유연하고 많은 장점이 있다.
- Hello 인터페이스의 메소드 3개가 아니라 30개로 늘어나면 어떻게 될까?
- 인터페이스가 바뀐다면 HelloUppercase처럼 클래스로 직접 구현한 프록시는 매번 코드를 추가해야한다.
- 다이내믹 프록시를 생성해서 사용하는 코드는 전혀 손댈 게 없다.
- 다이내믹 프록시가 만들어질 때 추가된 메소드가 자동으로 포함 될 것이고, 부가기능은 invoke() 메소드에서 처리되기 때문이다.
- InvocationHandler 방식의 또 한 가지 장점은 타깃의 종류에 상관없이도 적용이 가능하다 점이다.
- 리플렉션의 Method 인터페이스를 이용해 타깃의 메소드를 호출하는 것
- InvocationHandler는 단일 메소드에서 모든 요청을 처리하기 때ㅜㅁㄴ에 어떤 메소드에 어떤 기능을 적용할지를 선택하는 과정이 필요할 수도 있다.
- 호출하는 메소드의 이름, 파라미터의 개수와 타입, 리턴 타입 등의 정보를 가지고 부가적인 기능을 적용할 메소드를 선택할 수 있다.
- 리턴 타입뿐 아니라 메소드의 이름도 조건을 걸 수 있다.
```{.java}
	public class UppercaseHandler implements InvocationHandler {
		Object target;

		private UppercaseHandler(Object target) {
			this.target = target;
		}

		public Object invoke(Object proxy, Method method, Object[] args)
				throws Throwable {
			Object ret = method.invoke(target, args);
			if (ret instanceof String && method.getName().startsWith("say")) {
				return ((String)ret).toUpperCase();
			}
			else {
				return ret;
			}
		}
	}
```

#### 다이내믹 프록시를 이용한 트랜잭션 부가기능
- UserServiceTx를 다이내믹 프록시 방식으로 변경해보자.
- 기존의 방식은 서비스 인터페이스를 모두 구현해야 했고, 트랜잭션이 필요한 메소드마다 트랜잭션 코드를 중복적으로 작성해줘야 했다.
- 그러므로 트랜잭션 부가기능을 다이내믹 프록시를 만들어 적용하는 방법이 효율적이다.
- 트랜잭션 InvocationHandler 코드
```{.java}
	public class TransactionHandler implements InvocationHandler {
		private Object target;
		private PlatformTransactionMananger transactionManager;
		private String pattern;

		public void setTarget(Object target) {
			this.target = target;
		}

		public void setTransactional(PlatformTransacationManager transactionManager) {
			this.trasactionManager = transactionManager;
		}

		public void setPattern(String pattern) {
			this.pattern = pattern;
		}

		public Object invoke(Object proxy, Method method, Object[] args)
			throws Throwable {
			if (method.getName().startsWith(pattern)) {
				return invokeInTransaction(method, args);
			} else {
				return method.invoke(target, args);
			}
		}

		private Object invokeInTransaction(Method method, Object[] args)
			throws Throwable {
				TransactionStatus status = this.transactionManager
				.getTransaction(new DefaultTransactionDefinition());
			try {
				Object ret = method.invoke(target, args);
				this.transactionManager.commit(status);
				return ret;
			} catch (InvocationTargetException e) {
				this.transactionManager.rollback(status);
				throw e.getTargetException();
			}
		}
	}
```
- 요청을 위임할 타깃을 DI로 제공받도록 한다. 타깃을 저장할 변수는 Object로 선언했다.
- 따라서 UserServiceImpl 외에 트랜잭션 적용이 푤요한 어떤 타깃 오브젝트에도 적용할 수 있다.
- 롤백을 적용하기 위한 예외는 RuntimeException 대신에 InvocationTargetException을 잡도록 해야 한다.
- 예외가 InvoactionTargetException으로 포장되어서 되돌아기 때문이다.

#### 다이내믹 프록시를 위한 팩토리 빈
- 다이내믹 프록시 오브젝트는 일반적인 스프링의 빈으로 등록할 방법이 없다.
- 스프링의 빈은 기본적으로 클래스 이름과 프로퍼티로 정의된다.
- 스프링은 지정된 클래스 일므을 가지고 리플렉션을 이용해서 해당 클래스의 오브젝트를 만든다.
- 다이내믹 프록시는 Proxy 클래스의 newProxyInstance()라는 스태틱 팩토리 메소드를 통해서만 만들 수 있따.

#### 팩토리 빈
- 스프링은 클래스 정보를 가지고 디폴트 생성자를 통해 오브젝트를 만드는 방법 외에도 빈을 만들 수 있는 여러 가지 방법을 제공한다.
- 팩토리 빈을 이용한 빈 생성방법으로 빈을 만들수 있다.
- 팩토리 빈이란 스프링을 대신해서 오브젝트의 생성로직을 담당하도록 만들어진 특별한 빈을 말한다.
- 가장 간단한 방법은 스프링의 FactoryBean이라는 인터페이스를 구현하는 것이다.
```{.java}
public interface FactoryBean<T> {
	T getObject() throws Exception;
	Class<? exctends T> getObjectType();
	boolean isSingleton();
}
```
- 생성자가 private로 막혀있고 스태틱 팩토리 메소드로 선언되어 있는 경우 클래스를 직접 스프링 빈으로 등록해서 사용할 수 없다.
- private 생성자를 가진 클래스도 빈으로 등록해주면 리플렉션을 이용해 오브젝트를 만들어준다.
- 하지만 생성자를 private으로 만들었따는 것은 스태틱 메소드를 통해 오브젝트가 만들어져야 하는 중요한 이유가 있기 때문이므로 이를 무시하고 오브젝트를 강제로 생성하면 위험하다.
- 팩토리 빈 코드
```{.java}
//스태틱 팩토리 메소드를 가진 클래스
public class Message {
	String text;
	
	private Message(String text) {
		this.text = text;
	}
	
	public String getText() {
		return text;
	}

	public static Message newMessage(String text) {
		return new Message(text);
	}
}

//bean factory
public class MessageFactoryBean implements FactoryBean<Message> {
	String text;
	
	public void setText(String text) {
		this.text = text;
	}

	public Message getObject() throws Exception {
		return Message.newMessage(this.text);
	}

	public Class<? extends Message> getObjectType() {
		return Message.class;
	}

	public boolean isSingleton() {
		return true;
	}
}

//xml bean 설정
<bean id="message" class="springbook.learningtest.spring.factorybean.MessageFactoryBean">
		<property name="text" value="Factory Bean" />
</bean>

//message bean
@Test
public void getMessageFromFactoryBean() {
	Object message = context.getBean("message");
	assertThat(message, is(Message.class));
	assertThat(((Message)message).getText(), is("Factory Bean"));
}
	
//bean factory bean
@Test
public void getFactoryBean() throws Exception {
	Object factory = context.getBean("&message");
	assertThat(factory, is(MessageFactoryBean.class));
}
```

#### 다이내믹 프록시를 만들어주는 팩토리 빈
- Proxy의 newProxyInstrance() 메서드를 통해서만 생성이 가능한 다이내믹 프록시 오브젝트는 일방적인 방법으로는 스프링의 빈으로 등록할 수없다.
- 대신 팩토리 빈을 사용하면 다이내믹 프록시 오브젝트를 스프링의 빈으로 만들어줄 수가 있다.
- 스프링 빈에는 팩토리 빈과 UserServiceImpl만 빈으로 등록한다. 
- 팩토리 빈은 다이내믹 프록시가 위임할 타깃 오브젝트인 UserServiceImpl에 대한 레퍼런스를 프로퍼티를 통해 DI 받아둬야 한다.
- 다이내믹 프록시와 함께 생성할 TransactionHandler에게 타깃 오브젝트를 전달해줘야 하기 때문이다.
- 그외에도 다이내믹 프록시나 TransactionHandler를 만들 때 필요한 정보는 팩토리 빈의 프로퍼티로 설정해뒀다가 다이내믹 프록시를 만들면서 전달해줘야 한다.

#### 트랜잭션 프록시 팩토리 빈
```{.java}
public class TxProxyFactoryBean implements FactoryBean<Object> {
	Object target;
	PlatformTransactionManager transactionManager;
	String pattern;
	Class<?> serviceInterface;
	
	public void setTarget(Object target) {
		this.target = target;
	}

	public void setTransactionManager(PlatformTransactionManager transactionManager) {
		this.transactionManager = transactionManager;
	}

	public void setPattern(String pattern) {
		this.pattern = pattern;
	}

	public void setServiceInterface(Class<?> serviceInterface) {
		this.serviceInterface = serviceInterface;
	}

	// FactoryBean 인터페이스 구현 메소드
	public Object getObject() throws Exception {
		TransactionHandler txHandler = new TransactionHandler();
		txHandler.setTarget(target);
		txHandler.setTransactionManager(transactionManager);
		txHandler.setPattern(pattern);
		return Proxy.newProxyInstance(
			getClass().getClassLoader(),new Class[] { serviceInterface }, txHandler);
	}

	public Class<?> getObjectType() {
		return serviceInterface;
	}

	public boolean isSingleton() {
		return false;
	}
}

	<bean id="userService" class="springbook.user.service.TxProxyFactoryBean">
		<property name="target" ref="userServiceImpl" />
		<property name="transactionManager" ref="transactionManager" />
		<property name="pattern" value="upgradeLevels" />
		<property name="serviceInterface" value="springbook.user.service.UserService" />
	</bean>
```

#### 프록시 팩토리 빈 방식의 장점
- 장점
	- 프록시 팩토리 빈의 재사용
		- TransactionHandler를 이용하는 다이내믹 프록시를 생성해주는 TxPrxoyFactoryBean은 코드의 수정 없이도 다양한 클래스에 적용할 수 있다.
		- 타깃 오브젝트에 맞는 프로퍼티 정보를 설정해서 빈으로 등록해주기만 하면 된다.
		- 하나 이상의 TxProxyFactoryBean을 동시에 빈으로 등록해도 상관없다.
		- 팩토리 빈이기 때문에 각 빈의 타입은 타깃 인터페이스와 일치 한다.
		- 타깃의 모든 메소드에 트랜잭션 기능을 적용하려면 pattern 값을 빈 문자열을 사용해주면 된다.
		- 코드 한 줄 만들지 않고 기존 코드에 부가적인 기능을 추가해줄 수 있다는 건 정 말 매력적인 방법이 아닐 수 없다.
	- 데코레이터 패턴의 두가지 문제점을 해결 해준다.
		- 첫째 프록시를 적용할 대상이 구현하고 있는 인터페이스를 구현하는 프록시 클래스를 일일이 만들어야한다는 번거로움
		- 둘째 부가적인 기능이 여러 메소드에 반복적으로 나타나게 돼서 코드 중복의 문제가 발생한다는 점
		- 위의 두가지 문제점을 프록시 팩토리 빈이 해결 해준다.
- 이 과정에서 스프링 DI는 매우 중요한 역할을 한다.
- 프록시를 사용하려면 DI가 필요한 것은 물론이고 효율적인 프록시 생성을 위한 다이내믹 프록시를 사용하려고 할 때도 팩토리 빈을 통한 DI는 필수다.

#### 프록시 팩토리 빈의 한계
- 프록시를 통해 타깃에 부가기능을 제공하는 것은 메소드 단위로 일어나는 일이다.
- 하나의 클래스 안에 존재하는 여러 개의 메소드에 부가기능을 한 번에 제공하는 건 어렵지 않게 가능했다.
- 하지만 한 번에 여러 개의 클래스에 공통적인 부가기능을 제공하는 일은 지금까지 살펴본 방법으로는 불가능하다.
- 트랜잭션과 같이 비즈니스 로직을 담은 많은 클래스의 메소드에 적용할 필요가 있다면 거의 비슷한 프록시 팩토리 빈의 설절이 중복되는 것을 막을 수 없다.
- 하나의 타깃에 여러 개의 부가기능을 적용하려고 할 때도 문제다.
	- ex)프록시, 보안, 기능 검사, 부가기능 등을 동시에 추가 하고 싶을때 문제가 발생
- 적용 대상이 증가 할 수록 그만큼 XML 빈 설정 코드가 늘어난다.
- XML 빈 설정 코드가 많아 질수록 유지보수가 힘들어 질 수 있다.
- 비슷한 빈설정들이 중복으로 되니 먼가 찜찜하다.
- 또 한 가지 문제점은 TransactionHandler 오브젝트가 프록시 팩토리 빈 개수만큼 만들어진다는 점이다.
- 트랜잭션 부가기능을 제공하는 동일한 코드임에도 불구하고 타깃 오브젝트가 달라지면 새로운 TransactionHandler 오브젝트를 만들어야한다.


#### ProxyFactoryBean
- 스프링은 일관된 방법으로 프록시를 만들 수 있게 도와주는 추상 레이어를 제공한다.
- 생성된 프록시는 스프링의 빈으로 등록돼야 한다.
- 스프링은 프록시 오브젝트를 생성해주는 기술을 추상화한 팩토리 빈을 제공해준다.
- 스프링의 ProxyFactoryBean은 프록시를 생성해서 빈 오브젝트를 등록하게 해주는 팩토리 빈이다.
- ProxyFactoryBean은 순수하게 프록시를 생성하는 작업만을 담당하고 프록시를 통해 제공해줄 부가기능은 별도의 빈에 둘 수 있다.
- ProxyFactoryBean이 생성하는 프록시에서 사용할 부가기능은 MethodInterceptor 인터페이스를 구현해서 만든다.
- InvocationHandler의 invoke() 메소드는 타깃 오브젝트에 대한 정보를 제공하지 않는다. 따라서 타깃은 InvocationHandler를 구현한 클래스가 직접 알고 있어야 한다.
- MehodInterceptor의 invoke() 메소드는 ProxyFactoryBean으로부터 타깃 오브젝트에 대한 정보까지도 함께 제공 받는다.
- MethodInterceptor 오브젝트는 타깃이 다른 여러 프록시에서 함께 사용할수 있고, 싱글톤 빈으로 등록 가능하다.

#### 어드바이스:타깃이 필요 없는 순수한 부가기능
- 
