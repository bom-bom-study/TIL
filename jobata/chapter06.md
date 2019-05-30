# 6장 AOP

## 6.1 트랜잭션 코드의 분리
- PSA를 적용한 UserService
  - but, 트랜잭션 경계설정을 위해 넣은 코드가 많은 자리를 차지하고 있어 깔끔하지 못하다.

#### 6.1.1 메소드 분리
- upgradeLevels()
  - 메소드 구성
    - 트랜잭션 시작
    - 비즈니스 로직
    - 트랜잭션 종료
  - 메소드 특징
    - 트랜잭션 경계설정 코드와 비즈니스 로직 간에 서로 주고 받는 정보가 없다.
      - 비즈니스 로직에서 직접 DB를 사용하지 않기 때문에 트랜잭션 준비 과정에서 만드러진 DB 커넥션 정보 등을 직접 참조할 필요가 없기 때문
    - 트랜잭션 정보는 트랜잭션 동기화 방법을 통해 DAO가 알아서 활용한다.
  - 그렇다면?
    - 트랜잭션 경계설정과 비즈니스 로직은 서로 독립적인 코드이므로 두 개의 메소드로 분리할 수 있지 않을까?
- 우선 비즈니스 로직 부분만 따로 메소드로 분리하여 보았음.
  - 하지만 아직 트랜잭션을 담당하는 기술적인 코드가 버젓이 UserServcie 안에 자리 잡고 있다.

#### 6.1.2 DI를 이용한 클래스의 분리
- UserService를 인터페이스로 만들고 기존 코드의 비즈니스 로직과 트랜잭션 경계설정을 UserService 인터페이스의 구현 클래스를 만들어 넣도록 한다.
  - 클라이언트와의 결합은 낮아지고
  - 직접 구현 클래스에 의존하고 있지 않기 때문에 유연한 확장이 가능
  - ![image](https://user-images.githubusercontent.com/36880294/58157973-71665480-7cb4-11e9-80f2-d86d099ebccb.png)
- UserService 인터페이스
  ```java
  public interface UserService {
    void add(User user);
    void upgradeLevels();
  }
  ```
- UserServiceImpl 클래스
  ```java
  public class UserServiceImpl implements UserService {
    UserDao userDao;
    MailSender mailSender;

    public void upgradeLevels() {
      List<User> users = userDao.getAll();
      for (User user : users {
        if (canUpgradeLevel(user)) {
          upgradeLevel(user);
        }
      })
    }
    /*
      ...
    */
  }
  ```
- UserServiceTx 클래스
  ```java
  public class UserServiceTx implements UserService {
    UserService userService;
    PlatfromTransactionManager transactionManager;

    public void setUserService(UserService userService) {
      this.userService = userService;
    }

    public void SetTransactionManager(PlatformTransactionManager transaction Manager) {
      this.transactionManager = transactionManager;
    }

    public void add(User user) {
      userService.add(user);
    }

    public void upgradeLevels() {
      TransactionStatus status = this.transactionManager.getTransaction(new DefaultTransactionDefinition());
      try {
        userService.upgradeLevels();
        this.transactionManager.commit(status);
      } catch (RuntimeException e) {
        this.transactionManager.rollback(status);
        throw e;
      }
    }
  }
  ```
- 클라이언트가 UserService 인터페이스를 통해 사용자 관리 로직을 이용하려고 할 때 먼저 트랜잭션을 담당하는 오브젝트가 사용돼서 트랜잭션에 관련된 작업을 진행해주고, 실제 사용자 관리 로직을 담은 오브젝트가 이후에 호출돼서 비즈니스 로직에 관련된 작업을 수행하도록 만든다.
  - ![image](https://user-images.githubusercontent.com/36880294/58159114-cefba080-7cb6-11e9-983c-406aaa32b376.png)
- 트랜잭션 경계설정 코드 분리의 장점
  - 비즈니스 로직을 담당하고 있는 UserServicelmpl의 코드를 작성할 때는 트랜잭션과 같은 기술적인 내용에는 전혀 신경 쓰지 않아도 된다.
  - 비즈니스 로직에 대한 테스트를 손쉽게 만들어낼 수 있다는 것

## 6.2 고립된 단위 테스트
- 작은 단위 테스트의 장점
  - 테스트가 실패했을 때 원인을 찾기 쉽다.
  - 테스트의 의도나 내용이 분명해진다.
  - 만들기 쉽다.

#### 6.2.1 복잡한 의존관계 속의 테스트
- UserService 테스트 동작
  - ![image](https://user-images.githubusercontent.com/36880294/58159631-e8511c80-7cb7-11e9-8cc0-a361f304cc93.png)
  - 배보다 배꼽이 더 큰 작업이다.
    - UserServcie를 테스트하는 것처럼 보이지만 의존관계 때문에 사실은 그 뒤에 존재하는 더 많은 오브젝트와 환경, 서비스, 서버, 심지어 네트워크까지 함께 테스트하는 셈이 된다.

#### 6.2.2 테스트 대상 오브젝트 고립시키기
- 테스트의 대상을 고립시켜야 한다.
  - 테스트 스텁이나 목 오브젝트를 사용하여 테스트의 대상이 환경이나 외부 서버, 다른 클래스의 코드에 종속되고 영향을 받지 않도록해야한다.
- 테스트를 위한 UserServiceImpl 고립
  - ![image](https://user-images.githubusercontent.com/36880294/58160315-2e5ab000-7cb9-11e9-927a-b50657929827.png)
  - 사전에 테스트를 위해 준비된 동작만 하도록 만든 두 개의 목 오브젝트에만 의존하는 완벽하게 고립된 테스트 대상으로 만들 수 있다.
  - but, 의존 오브젝트나 외부 서비스에 의존하지 않는 고립된 테스트 방식으로 만든 UserServicelmpl은 아무리 그 기능이 수행돼도 그 결과가 DB 등을 통해서 남지 않으니, 기존의 방법으로는 작업 결과를 검증하기 힘들다.
    - UserDao에게 어떤 요청을 했는지 확인하는 작업이 필요
    - UserDao와 같은 역할을 하면서 UserServicelmpl과의 사이에서 주고받은 정보를 저장해뒀다가, 태스트의 검증 에 사용할 수 있게 하는 목 오브젝트를 만들 필요가 있다.
- 고립된 단위 테스트 활용
  - 기존 upgaradeLevels() 테스트 
    - DB 테스트 데이터 준비
    - 메일 발송 여부 확인을 위해 목 오브젝트 DI
    - 테스트 대상 실행
    - DB에 저장된 결과 확인
    - 목 오브젝트를 이용한 결과 확인
  - MockUserDao를 사용해서 만든 고립된 테스트
    - 테스트 대상 오브젝트를 직접 생성한다.
    - 목 오브젝트로 만든 UserDao를 직접 DI 해준다.
    - MockUserDao로부터 업데이트 결과를 가져온다.
    - 업데이트 횟수와 정보를 확인한다.
- 테스트 수행 성능의 향상
  - 고립된 테스트를 하면 태스트가 다른 의존 대상에 영향을 받을 경우를 대비해 복잡하게 준비할 펼요가 없을 뿐만 아니라, 테스트 수행 성능도 크게 향상된다.

#### 6.2.3 단위 테스트와 통합 테스트
- 단위 테스트
  - 태스트 대상 클래스를 목 오브젝트 등의 태스트 대역을 이용해 의존 오브젝트나 외부의 리소스를 사용하지 않도록 고립시켜서 태스트하는 것
- 통합 테스트
  - 두 개 이상의, 성격이나 계층이 다른 오브젝트가 연동하도록 만들어 테스트하거나, 또는 외부의 DB나 파일, 서비스 등의 리소스가 참여하는 태스트
- 단위 테스트와 통합 테스트 중 어떤 방법을 쓸 것인가?
  - 항상 단위 테스트를 먼저 고려한다.
  - 하나의 클래스나 성격과 목적이 같은 긴밀한 클래스 몇 개를 모아서 외부와의 의존관계를 모두 차단하고 필요에 따라 스텁이나 목 오브젝트 등의 테스트 대역을 이용하도록 태스트를 만든다.
  - 외부 리소스를 사용해야만 가능한 테스트는 통합 테스트로 만든다.
  - DAO는 DB까지 연동하는 태스트로 만드는 편이 효과적이다.
  - DAO를 태스트를 통해 충분히 검증해두면 DAO를 이용하는 코드는 DAO 역할을 스텁이나 목 오브젝트로 대체해서 테스트할 수 있다.
  - 여러 개의 단위가 의존관계를 가지고 동작할 때를 위한 통합 테스트는 필요하다.
  - 단위 테스트를 만들기가 너무 복잡하다고 판단되는 표드는 처음부터 통합 테스트를 고려해 본다.
    - 이때도 통합 테스트에 참여하는 코드 중에서 가능한 한 많은 부분을 미리 단위 테스트로 검증해두는게 유리하다.
  - 스프링 태스트 컨텍스트 프레임워크를 이용하는 테스트는 통합 테스트다.
- 테스트에 대한 생각
  - 단위 테스트와 통합 테스트 모두 개발자가 스스로 자신이 만든 코드를 테스트하기 위해 만드는 개발자 테스트다.
  - 코드를 작성하면서 테스트는 어떻게 만들 수 있을까를 생각해보는 것은 좋은 습관이다
  - 테스트하기 편하게 만들어진 코드는 깔끔하고 좋은 코드가 될 가능성이 높다.
  - 스프링이 지지하고 권장하는 깔끔하고 유연한 코드를 만들다보면 테스트도 그만큼 만들기 쉬워지고, 테스트는 다시 코드의 품질을 높여주고, 리팩토링과 개선에 대한 용기를 주기도 할 것이다.

#### 6.2.4 목 프레임워크
- 번거로운 목 오브젝트를 편리하게 작성하도록 도와주는 다양한 목 오브젝트 지원 프레임워크가 있다.
- Mockito 프레임워크
  - 인터페이스를 이용해 목 오브젝트를 만든다.
  - 목 오브젝트가 리턴할 값이 있으면 이를 지정해준다.
    - 메소드가 호출되면 예외를 강제로 던지게 만들 수도 있다.
  - 테스트 대상 오브젝트에 DI 해서 목 오브젝트가 테스트 중에 사용되도록 만든다.
  - 테스트 대상 오브젝트를 사용한 후에 목 오브젝트의 특정 메소드가 호출됐는지, 어떤 값을 가지고 몇 번 호출됐는지를 검증한다.

## 6.3 다이내믹 프록시와 팩토리 빈

#### 6.3.1 프록시와 프록시 패턴, 데코레이터 패턴
- 전략 패턴 적용
  - ![image](https://user-images.githubusercontent.com/36880294/58164458-da53c980-7cc0-11e9-895d-5ab5d1ce3abf.png)
  - 전략 패턴으로는 트랜잭션 기능의 구현 내용을 분리해냈을 뿐이다.
  - 트랜잭션을 적용한다는 사실은 그대로 코드에 남아 있다.
  - 구체적인 구현 코드는 제거했을지라도 위임을 통해 기능을 사용하는 핵심 코드와 함께 남아 있다.
- 부가기능과 핵심 기능의 분리
  - ![image](https://user-images.githubusercontent.com/36880294/58164604-1e46ce80-7cc1-11e9-8b30-4f36ee9f7e77.png)
  - UserService 인터페이스를 통해 UserServiceTx와 UserServiceImpl 구현체 클래스를 만들었다.
  - 부가기능 전부를 핵심 코드가 담긴 클래스에서 독립시킬 수 있다.
  - 부가기능 외의 나머지 모든 기능은 원래 핵심기능을 가진 클래스로 위임해줘야 한다.
    - 핵심기능은 부가기능을 가진 클래스의 존재 자체를 모른다.
    - 따라서 부가기능이 핵심기능을 사용하는 구조가 된다.
  - 문제는 클라이언트가 핵심기능을 가진 클래스를 직접 사용한다면?
    - 부가기능이 적용될 기회가 없다.
    - 그래서 부가기능은 마치 자신이 핵심 기능을 가진 클래스인 것처럼 꾸며서 자신을 거쳐서 핵심 기능을 사용하도록 만들어야 한다.
- 핵심기능 인터페이스의 적용
  - ![image](https://user-images.githubusercontent.com/36880294/58164961-c6f52e00-7cc1-11e9-8fce-99b92af6bbf6.png)
  - 클라이언트는 인터페이스를 통해서만 핵심기능을 사용하게 하고, 부가기능 자신도 같은 인터페이스를 구현한 뒤에 자신이 그 사이에 끼어들어야 한다.
  - 클라이언트는 인터페이스만 보고 사용을 하기 때문에 자신은 핵심기능을 가진 클래스를 사용할 것이라고 기대하지만 사실은 부가기능을 통해 핵심기능을 이용하게 되는 것이다.
  - 부가기능 코드에서는 핵심기능으로 요청을 위임해주는 과정에서 자신이 가진 부가적인 기능을 적용해 줄 수 있다.
    - 비즈니스 로직 코드에 트랜잭션 기능을 부여해주는 경우가 예시
- 프록시(Proxy)
  - 마치 자신이 클라이언트가 사용하려고 하는 실제 대상인 것처럼 위장해서 클라이언트의 요청을 받아주는 것을 대리자, 대리인과 같은 역할을 한다고 해서 프록시라고 부른다.
- 타깃(target) or 실체(real subject)
  - 프록시를 통해 최종적으로 요청을 위임받아 처리하는 실제 오브젝트
- 프록시와 타깃을 사용하는 구조
  - ![image](https://user-images.githubusercontent.com/36880294/58165220-4e42a180-7cc2-11e9-83ce-4e3bab292b61.png)
- 프록시의 특징
  - 타깃과 같은 인터페이스를 구현했다는 것
  - 프록시가 타깃을 제어할 수 있는 위체이 있다는 것
- 프록시 사용 목적
  - 클라이언트가 타깃에 접근하는 방법을 제어하기 위해서
  - 타깃에 부가적인 기능을 부여해주기 위해서
- 데코레이터 패턴
  - 타깃에 부가적인 기능을 런타임 시 다이내믹하게 부여해주기 위해 프록시를 사용하는 패턴
    - 다이내믹하게 기능을 부가한다는 의미?
      - 컴파일 시점 즉 코드상에서는 어떤 방법과 순서로 프록시와 타깃이 연결되어 사용되는지 정해져 있지 않다는 뜻
  - 데코레이터라고 불리는 이유?
    - 제품이나 케익 등을 여러 겹으로 포장하고 그 위에 장식을 부팅는 것처럼 실제 내용물은 동일하지만 부가적인 효과를 부여해줄 수 있기 때문이다.
  - 프록시가 한개로 제한되지 않는다.
    - 프록시가 직접 타깃을 사용하도록 고정시킬 필요가 없다.
    - 같은 인터페이스를 구현한 타겟과 여러 개의 프록시를 사용할 수 있다.
      - 여러 개인 만큼 순서를 정해서 단계적으로 위임하는 구조로 만들면 된다.
  - 소스코드를 출력하는 기능을 가진 핵심기능에 적용한 예
    - ![image](https://user-images.githubusercontent.com/36880294/58165598-0c662b00-7cc3-11e9-866c-12a0780f80c4.png)
  - 데코레이터 패턴은 인터페이스를 통해 위임하는 방식이다.
    - 각 데코레이터는 위임하는 대상에도 인터페이스로 접근하기 때문에 자신이 최종 타깃으로 위임하는지, 아니면 다음 단계의 데코레이터 프록시로 위임하는지 알지 못한다.
    - 데코레이터 다음 위임 대상은 인터페이스로 선언하고 생성자나 수정자 메소드를 통해 위임 대상을 외부에서 런타임 시에 주입받을 수 있도록 만들어야 한다.
  - 데코레이터 패턴은 타깃의 코드를 손대지 않고, 클라이언트가 호출하는 방법도 변경하지 않은 채로 새로운 기능을 추가할 때 유용한 방법이다.
- 프록시 패턴
  - 용어의 구분
    - 일반적으로 사용하는 프록시
      - 클라이언트와 사용 대상 사이에 대리 역할을 맡은 오브젝트를 두는 방법
    - 프록시 패턴
      - 프록시를 사용하는 방법 중에서 타깃에 대한 접근 방법을 제어하려는 목적을 가진 경우
  - 프록시 패턴의 프록시는 타깃의 기능을 확장하거나 추가하지 않는다.
  - 프록시 패턴 적용 예
    - 타깃 오브젝트에 대한 레퍼런스가 미리 필요할 수 있을 때
      - 클라이언트에게 타깃에 대한 레퍼런스를 넘겨야 하는데, 실제 타깃 오브젝트는 만드는 대신 프록시를 넘겨주는 것이다.
      - 그리고 프록시의 메소드를 통해 타깃을 사용하려고 시도하면, 그때 프록시가 타깃 오브젝트를 생성하고 요청을 위임해주는 식이다.
      - 만약 레퍼런스는 갖고 있지만 끝까지 사용하지 않거나, 많은 작업이 진행된 후에 사용되는 경우라면 이렇게 프록시를 통해 생성을 최대한 늦춤으로써 얻는 장점이 많다.
    - 원격 오브젝트를 이용하는 경우
      - RMI나 EJB, 또는 각종 리모팅 기술을 이용해 다른 서벼에 존재하는 오브젝트를 사용해야 한다면, 원격 오브젝트에 대한 프록시를 만들어두고, 클라이언트는 마치 로컬에 존재하는 오브젝트를 쓰는 것처럼 프록시를 시용하게 할 수 있다.
      - 프록시는 클라이언트의 요청을 받으면 네트워크를 통해 원격의 오브젝트를 실행하고 결과를 받아서 클라이언트에게 돌려준다.
      - 클라이언트로 하여금 원격 오브젝트에 대한 접근 방법을 제공해주는 프록시 패턴의 예라고 볼 수 있다.
    - 특별한 상황에서 타깃에 대한 접근권한을 제어하기 위해
      - 수정 가능한 오브젝트가 있는데, 특정 레이어로 넘어가서는 읽기전용으로만 동작하게 강제해야 한다고 하자.
      - 이럴때는 오브젝트의 프록시를 만들어서 사용할 수 있다. 프록시의 특정 메소드를 사용하려고 하면 접근이 불가능하다고 예외를 발생시키면 된다.
  - 클라이언트가 타깃에 접근하는 방식을 변경해준다.
  - 타깃의 기능 자체에는 관여하지 않으면서 접근하는 방법을 제어해주는 프록시를 이용하는 것이다.
  - 프록시 패턴과 데코레이터 패턴의 혼용
    - ![image](https://user-images.githubusercontent.com/36880294/58167746-a203b980-7cc7-11e9-9676-3e229328163e.png)
    - 프록시는 코드에서 자신이 만들거나 접근할 타깃 클래스 정보를 알고 있는 경우가 많다.
      - 생성을 지연하는 프록시라면 구체적인 생성 방법을 알아야 하기 때문에 타깃 클래스에 대한 직접적인 정보를 알아야 한다.
      - 물론 인터페이스를 통해 위임하도록 만들 수도 있다.
    - 접근제어를위한프록시를두는프록시 패턴과 컬러, 페이징 기능을 추가하기 위한 프록시를 두는 데코레이터 패턴을함께 적용한 예다.
    - 두가지 모두 프록시의 기본 원리대로 타깃과 같은 인터페이스를 구현해두고 위임하는 방식으로 만들어져 있다.
- 앞으로는 타깃과 통일한 인터페이스를 구현하고 클라이언트와 타깃 사이에 존재하면서 기능의 부가 또는 접근 제어를 담당하는 오브젝트를 모두 프록시라고 부르겠다.
  - 그때마다 사용의 목적이 기능의 부가인지, 접근 제어인지를 구분해보면 각각 어떤 목적으로 프록시가 사용됐는지 그에 따라 어떤 패턴이 적용됐는지 알 수 있을 것이다.

#### 6.3.2 다이내믹 프록시
- 프록시도 일일이 모든 인터페이스를 구현해서 클래스를 새로 정의하지 않고도 편리하게 만들어서 사용할 방법은 없을까?
  - java.lang.reflect 패키지 안에 프록시를 손쉽게 만들 수 있도록 지원해주는 클래스들이 있음.
- 프록시의 구성
  - 타깃과 같은 메소드를 구현하고 있다가 메소드가 호출되면 타깃 오브젝트로 위임한다.
  - 지정된 요청에 대해서는 부가기능을 수행한다.
- 프록시를 만들기 번거로운 이유?
  - 타깃의 인터페이스를 구현하고 위임하는 코드를 작성하기가 번거롭다.
  - 부가기능 코드가 중복될 가능성이 많다.
- 이러한 번거로움은 JDK의 다이내믹 프록시로 해결 가능하다.
- 리플렉션
  - 자바의 코드 자체를 추상화해서 접근하도록 만든 것
  - (java.lang.reflect 패키지)[https://docs.oracle.com/javase/8/docs/api/java/lang/reflect/package-summary.html]
- 프록시 클래스
  - Hello 인터페이스
    ```java
    public interface Hello {
      String sayHello(String name);
      String sayHi(String name);
      String sayThankYou(String name);
    }
    ```
  - 타깃 클래스
    ```java
    public class HelloTarget implements Hello {
      public String sayHello(String name) {
        return "Hello " + name;
      }

      public String SayHi(String name) {
        return "Hi " + name;
      }

      public String SayThankYou(String name) {
        return "Thank You " + name;
      }
    }
    ```
  - 클라이언트 역할의 테스트
    ```java
    @Test
    public void simpleProxy() {
      Hello hello = new HelloTarget();  // 타깃은 인터페이스를 통해 접근하는 습관을 들이자.
      assertThat(hello.sayHello("Toby"), is("Hello Toby"));
      assertThat(hello.sayHi("Toby"), is("Hi Toby"));
      assertThat(hello.sayThankYou("Toby"), is("Thank You Toby"));
    } 
    ```
  - 프록시 클래스
    ```java
      public class HelloUppercase implements Hello {
        Hello hello;  // 다른 프록시를 추가할 수도 있으니 인터페이스로 접근

        public helloUppercase(Hello hello) {
          this.hello = hello;
        }

        public String sayHello(String name) {
          return hello.sayHello(name).toUpperCase();  // 위임과 부가기능 적용
        }

        public String SayHi(String name) {
          return hello.sayHi(name).toUpperCase();
        }

        public String SayThankYou(String name) {
          return hello.sayThankYou(name).toUpperCase();
        }
      }
    ```
  - 프록시 적용의 일반적인 문제점 두 가지를 모두 갖고 있다.
    - 인터페이스의 모든 메소드를 구현해 위임하도록 코드를 만들어야 한다.
    - 부가기능이 모든 메소드에 중복돼서 나타난다.
- 다이내믹 프록시 적용
  - 다이내믹 프록시 동작 방식
    - ![image](https://user-images.githubusercontent.com/36880294/58169265-4dfad400-7ccb-11e9-89ba-fee43bfea0a2.png)
    - 다이내믹 프록시는 프록시 팩토리에 의해 런타임 시 다이내믹하게 만들어지는 오브젝트
    - 클라이언트는 다이내믹 프록시 오브젝트를 타깃 인터페이스를 통해 사용할 수 있다.
      - 프록시를 만들 때 인터페이스를 모두 구현해가면서 클래스를 정의하는 수고를 덜 수 있다.
      - 프록시 팩토리에게 인터페이스 정보만 제공해주면 해당 인터페이스를 구현한 클래스의 오브젝트를 자동으로 만들어 준다.
    - 구현 클래스의 오브젝트는 만들어주지만, 프록시로서 필요한 부가기능 제공 코드는 직접 작성해야 한다.
      - 부가기능은 프록시 오브젝트와 독립적으로 InvocationHandler를 구현한 오브젝트에 담는다.
  - InvocationHandler를 통한 요청 처리 구조
    - ![image](https://user-images.githubusercontent.com/36880294/58418881-140f4080-80c4-11e9-9b2b-207ec38e29fb.png)
  - InvocationHandler 구현 클래스
    ``` java
    public class UppercaseHandler implements InvocationHandler {
      Hello target;  // 다이내믹 프록시부터 전달받은 요청을 다시 타깃 오브젝트에 위임해야 하기 때문에 타깃 오브젝트를 주입받아 둔다.

      public UppercaseHandler(Hello target) {
        this.target = target;
      }

      public Object invoke(Object proxy, Method method, Object[] args) throws Throwable {
        String ret = (String)method.invoke(target, args);  // 타깃으로 위임, 인터페이스의 메소드 호출에 모두 적용한다.
        return ret.toUpperCase();  // 부가기능 제공  
      }
    }
    ```
      - 다이내믹 프록시가 클라이언트로부터 받는 모든 요청은 invoke() 메소드로 전달된다.
      - 다이내믹 프록시를 통해 요청이 전달되면 리플렉션 API를 이용해 타깃 오브젝트의 메소드를 호출한다.
      - 타깃 오브젝트는 생성자를 통해 미리 전달받아 둔다.
  - 프록시 생성
    ```java
    Hello proxiedHello = (Hello)Proxy.newProxyInstance(  // 생성된 당이내믹 프록시 오브젝트는 Hello 인터페이스를 구현하고 있으므로 Hello 타입으로 캐스팅해도 안전하다.
      getClass().getClassLoader(),  // 동적으로 생성되는 다이내믹 프록시 클래스의 로딩에 사용할 클래스 로더
      new Class[] { Hello.class },  // 구현할 인터페이스
      new UppercaseHandler(new HelloTarget()));  // 부가기능과 위임 코드를 담은 InvocationHandler
    ```
- 다이내믹 프록시의 확장
  - 인터페이스의 메소드가 늘어나도 매번 코드를 추가할 필요가 없다.
    - 다이내믹 프록시가 만들어질 때 추가된 메소드가 자동으로 포함될 것이고 부가기능은 invoke() 메소드에서 처리된다.
  - 타깃의 종류에 상관없이도 적용이 가능하다.
    - 리플렉션의 Method 인터페이스를 이용해 타깃의 메소드를 호출하기 때문
  - 확장된 UppercaseHandler
    ```java
    public class UppercaseHandler implements InvocationHandler {
      Object target;  // 어떤 종류의 인터페이스를 구현한 타깃에도 적용 가능하도록 Object 타입으로 수정
      private UppercaseHandler(Object target) {
        this.target = target;
      }

      public Object invoke(Object proxy, Methos method, Object[] args) throws Throwable {
        Object ret = method.invoke(target, args);
        if (ret instanceof String) {
          return ((String)ret).toUpperCase();
        } else {
          return ret;
        }
      }
    }
    ```
  - 메소드를 선별해서 부가기능을 적용하는 invoke()
    ```java
    public Object invoke(Object proxy, Methos method, Object[] args) throws Throwable {
        Object ret = method.invoke(target, args);
        if (ret instanceof String && method.getName().startsWith("say")) {
          return ((String)ret).toUpperCase();
        } else {
          return ret;
        }
      }
    ```

#### 6.3.3 다이내믹 프록시를 이용한 트랜잭션 부가기능
- 트랜잭션이 필요한 클래스와 메소드가 증가하면 UserServiceTx처럼 프록시 클래스를 일일이 구현하는 것은 큰 부담이다.
  - 트랜잭션 부가기능을 제공하는 다이내믹 프록시를 만들어 적용하자.
  - 다이내믹 프록시와 연동해서 트랜잭션 기능을 부가해주는 InvocationHandler는 한개만 정의해도 충분하다.
- 트랜잭션 InvocationHandler
  - 다이내믹 프록시를 위한 트랜잭션 부가기능
    ```java
    public class TransactionHandler implements InvocationHandler {
      private Object target;  // 부가기능을 제공할 타깃 오브젝트, 어떤 타입의 오브젝트에도 적용 가능하다.
      private PlatformTransactionManager transactionManager;  // 트랜잭션 기능을 제공하는 데 필요한 트랜잭션 매니저
      private String pattern;  // 트랜잭션을 적용할 메소드 이름 패턴

      public void setTarget(Object target) {
        this.target = target;
      }

      public void setTransactionManager(PlatformTransactionManager transactionManager) {
        this.transactionManager = transactionManager;
      }

      public void setPattern(String pattern) {
        this.pattern = pattern;
      }

      public Object invoke(Object proxy, Methos method, Object[] args) throws Throwable {
        if (method.getName().startsWith(pattern)) {  // 트랜잭션 적용 대상 메소드를 선별해서 트랜잭션 경계설정 기능을 부여해준다.
          return invokeInTransaction(method, args);
        } else {
          return method.invoke(target, args);
        }
      }

      private Object invokeInTransaction(Method method, Object[] args) throws Throwable {
        TransactionStatus status = this.transactionManager.getTransaction(new DefaultTransactionDefinition());
        try {  // 트랜잭션을 시작하고 타깃 오브젝트의 메소드를 호출한다. 예외가 발생하지 않았다면 커밋한다.
          Object ret = method.invoke(target, args);
          this.transactionManager.commit(status);
          return ret;
        } catch (InvocationTargetException e) {  // 예외가 발생하면 트랜잭션을 롤백한다.
          this.transactionManager.rollback(status);
          throw e.getTargetException();
        }
      }
    }
    ```
- TransactionHandler와 다이내믹 프록시를 이용하는 테스트
  - 다이내믹 프록시를 이용한 트랜잭션 테스트
    ```java
    @Test
    public void upgaradeAllOrNothing() throws Exception {
      // ...
      TransactionHandler txHandler = new TransactionHandler();
      // 트랜잭션 핸들러가 필요한 정보와 오브젝트를 DI 해준다.
      txHandler.setTraget(testUserService);
      txHandler.setTransactionManager(transactionManager);
      txHandler.setPattern("upgradeLevels");
      UserService txUserService = (UserService)Proxy.newProxyInstance(getClass().getClassLoader().new Class[] { UserService.class }, txHandler);  // UserService 인터페이스 타입의 다이내믹 프록시 생성
      // ...
    }
    ```

#### 6.3.4 다이내믹 프록시를 위한 팩토리 빈
- TransactionHandler와 다이내믹 프록시를 스프링의 DI를 통해 사용할 수 있도록 만들 차례이다.
  - but, DI의 대상이 되는 다이내믹 프록시 오브젝트는 스프링의 빈으로는 등록할 방법이 없다.
    - 클래스 자체도 내부적으로 다이내믹하게 새로 정의해서 사용하기 때문에 사전에 프록시 오브젝트의 클래스 정보를 미리 알아내서 스프링의 빈에 정의할 방법이 없다.
    - 다이내믹 프록시는 Proxy 클래스의 newProxyInstance()라는 스태틱 팩토리 메소드를 통해서만 만들 수 있다.
- 스프링의 빈 오브젝트 생성 방법
  - 스프링 빈은 기본적으로 클래스 이름과 프로퍼티로 정의된다.
  - 스프링은 지정된 클래스 이름을 가지고 리플랙션을 이용해서 해당 클래스의 오브젝트를 만든다.
    ```java
    Date now = (Date)Class.forName("jave.util.Date").newInsatnace();
    ```
    - 클래스의 이름을 갖고 있다면 다음과 같은 방법으로 새로운 오브젝트를 생성할 수 있다.
    - Class의 newInstance() 메소드는 해당 클래스의 파라미터가 없는 생성자를 호출하고, 그 결과 생성되는 오브젝트를 돌려주는 리플렉션 API다.
  - 스프링은 내부적으로 리플렉션 API를 이용해서 빈 정의에 나오는 클래스 이름을 가지고 빈 오브젝트를 생성한다.
- 사실 스프링은 클래스 정보를 가지고 디폴트 생성자를 통해 오브젝트를 만드는 방법 외에도 빈을 만들 수 있는 여러 가지 방법을 제공한다.
  - 대표적으로 팩토리 빈을 이용한 빈 생성방법을 들 수 있다.
- 팩토리 빈
  - 스프링을 대신해서 오브젝트의 생성로직을 담당하도록 만들어진 특별한 빈
  - 만드는 방법
    - 스프링의 FactoryBean이라는 인터페이스를 구현하는 것
      ```java
      package org.springframework.beans.factory;

      public interface FactoryBean<T> {
        T getObject() throws Exception;  // 빈 오브젝트를 생성해서 돌려준다.
        Class<? extends T> getObjectType();  // 생성되는 오브젝트의 타입을 알려준다.
        boolean isSingleton();  // getObject()가 돌려주는 오브젝트가 항상 같은 싱글톤 오브젝트인지 알려준다.
      }
      ```
    - FactoryBean 인터페이스를 구현한 클래스를 스프링의 빈으로 등록하면 팩토리 빈으로 동작한다.
  - 생성자를 제공하지 않는 클래스
    ```java
    public class Message {
      String text;

      private Message(String text) {  // 생성자가 private으로 선언되어 있어서 외부에서 생성자를 통해 오브젝트를 만들 수 없다.
        this.text = text;
      }

      public static Message newMessage(String text) {  // 생성자 대신 사용할 수 있는 스태틱 팩토리 메소드를 제공한다.
        return new Message(text);
      }
    }
    ```
    - newMessage()라는 스태틱 메소드를 사용해야하기 때문에 직접 스프링 빈으로 등록해서 사용할 수 없다.
      ```xml
      <bean id="m" class="springbook.learningtest.spring.factorybean.Message">
      ```
      - 사실 스프링은 private 생성자를 가진 클래스도 빈으로 등록해주면 리플렉션을 이용해 오브젝트를 만들어준다.
        - 리플렉션은 private으로 선언된 접근 규약을 위반할 수 있는 강력한 기능이 있다.
      - but, 생성자를 private으로 만들었다는 것은 스태틱 메소드를 통해 오브젝트가 만들어져야 하는 중요한 이유가 있기 때문이므로 이를 무시하고 오브젝트를 강제로 생성하면 위험하다.
  - Message의 팩토리 빈 클래스
    ```java
    public class MessageFactoryBean implements FactoryBean<Message> {
      String text;  // 오브젝트를 생성할 때 필요한 정보를 팩토리 빈의 프로퍼티로 설정해서 대신 DI 받을 수 있게 한다. 주입된 정보는 오브젝트 생성 중에 사용된다.

      public void setText(String text) {
        this.text = text;
      }

      public Message getObject() throws Exception {  // 실제 빈으로 사용할 오브젝트를 직접 생성한다. 코드를 이용하기 때문에 복잡한 방식의 오브젝트 생성과 초기화 작업도 가능하다.
        return Message.newMessage(this.text);
      }

      public Class<? extends Message> getObjectType() {
        return Message.class;
      }

      public boolean isSingleton() {  // getObject() 메소드가 돌려주는 오브젝트가 싱글톤인지를 알려준다. 이 팩토리 빈은 매번 요청할 때마다 새로운 오브젝트를 만들므로 false로 설정한다. 이것은 팩토리 빈의 동작방식에 관한 설정이고 만들어진 빈 오브젝트는 싱글톤으로 스프링이 관리해줄 수 있다.
        return false;
      }
    }
    ```
    - 팩토리 빈은 전형적인 팩토리 메소드를 가진 오브젝트
    - 스프링은 FactoryBean 인터페이스를 구현한 클래스가 빈의 클래스로 지정되면, 팩토리 빈 클래스의 오브젝트의 getObject() 메소드를 이용해 오브젝트를 가져오고, 이를 빈 오브젝트로 사용한다.
    - 빈의 클래스로 등록된 팩토리 빈은 빈 오브젝트를 생성하는 과정에서만 사용될 뿐이다.
- 다이내믹 프록시를 만들어주는 팩토리 빈
  - 팩토리 빈을 이용한 트랜잭션 다이내믹 프록시의 적용
    - ![image](https://user-images.githubusercontent.com/36880294/58430517-990a5200-80e4-11e9-9976-faf1d6fcd035.png)
    - 스프링 빈에는 팩토리 빈과 UserServiceImpl만 빈으로 등록한다.
    - 팩토리 빈은 다이내믹 프록시가 위임할 타깃 오브젝트인 UserServiceImpl에 대한 레퍼런스를 프로퍼티를 통해 DI 받아둬야 한다.
      - 다이내믹 프록시와 함께 생성할 TransactionHandler에게 타깃 오브젝트를 전달해줘야 하기 때문
      - 다이내믹 프록시나 TransactionHandler를 만들 때 필요한 정보는 팩토리 빈의 프로퍼티로 설정해뒀다가 다이내믹 프록시를 만들면서 전달해줘야한다.

#### 6.3.5 프록시 팩토리 빈 방식의 장점과 한계
- 프록시 팩토리 빈의 재사용
  - TransactionHandler를 이용하는 다이내믹 프록시를 생성해주는 TxProxyFactoryBean은 코드의 수정 없이도 다양한 클래스에 적용할 수 있다.
  - 타깃 오브젝트에 맞는 프로퍼티 정보를 설정해서 빈으로 등록해주기만 하면 된다.
  - 설정 변경을 통한 트랜잭션 기능 부가
    - ![image](https://user-images.githubusercontent.com/36880294/58546730-f0bdd000-8240-11e9-8e35-8b9d80d91db4.png)
- 프록시 팩토리 빈 방식의 장점
  - 다이내믹 프록시를 이용하면 타깃 인터페이스를 구현하는 클래스를 일일이 만드는 번거로움을 제거할 수 있다.
  - 하나의 핸들러 메소드를 구현하는 것만으로도 수많은 메소드에 부가기능을 부여해줄 수 있으니 부가기능 코드의 중복 문제도 사라진다.
  - 다이내믹 프록시에 팩토리 빈을 이용한 DI까지 더해주면 다이내믹 프록시 생성 코드도 제거할 수 있다.
  - DI 설정만으로 다양한 타깃 오브젝트에 적용도 가능하다.
- 프록시 팩토리 빈의 한계
  - 한 번에 여러 개의 클래스에 공통적인 부가기능을 제공하는 일을 지금까지 살펴본 방법으로는 불가능하다.
    - 프록시를 통해 타깃에 부가기능을 제공하는 것은 메소드 단위로 일어나는 일이다.
      - 하나의 클래스 안에 존재하는 여러 개의 메소드에 부가기능을 한 번에 제공하는 건 어렵지 않게 가능하다.
    - 하나의 타깃 오브젝트에만 부여되는 부가기능이라면 상관 없겠지만 트랙잰션과 같이 비즈니스 로직을 담은 많은 클래스의 메소드에 적용할 필요가 있다면 거의 비슷한 프록시 팩토리 빈의 설정이 중복되는 것을 막을 수 없다.
  - 하나의 타깃에 여러 개의 부가기능을 적용하려고 할 때도 문제다.
    - 프록시 팩토리 빈 설정이 부가기능의 개수만큼 따라 붙어야 한다.
    - XML 설정이 사람이 손으로 편집할 수 있는 한계를 벗어난다.
  - TransactionHandler 오브젝트가 프록시 팩토리 빈 개수만큼 만들어진다.

## 6.4 스프링의 프록시 팩토리 빈

#### 6.4.1 ProxyFactoryBean
- 스프링은 프록시 오브젝트를 생성해주는 기술을 추상화한 팩토리 빈을 제공해준다.
- ProxyFactoryBean
  - 서비스 추상화를 프록시 기술에 동일하게 적용
  - 프록시를 생성해서 빈 오브젝트로 등록하게 해주는 팩토리 빈이다.
  - 순수하게 프록시를 생성하는 작업만을 담당하고 프록시를 통해 제공해줄 부가기능은 별도의 빈에 둘 수 있다.
  - ProxyFactoryBean이 생성하는 프록시에서 시용할 부가기능은 Methodlnterceptor 인터페이스를 구현해서 만든다.
    - Methodlnterceptor의 invoke() 메소드는 ProxyFactoryBean으로부터 타깃 오브젝트에 대한 정보까지도 함께 제공받는다.
    - Methodlnterceptor 오브젝트는 타깃이 다른 여러 프록시에서 함께 사용할 수 있고, 싱글톤 빈으로 등록 기능하다.
- ProxyFactoryBean vs JDK 다이내믹 프록시
  - Methodlnvocation 구현 클래스는 일종의 공유 가능한 뱀플릿처럼 동작한다.
    - ProxyFactoryBean은 작은 단위의 빔플릿/콜백 구조를 응용해서 적용했기 때문에 뱀플릿 역할을 하는 Methodlnvocation을 싱글톤으로 두고 공유할 수 있다.
  - 아무리 많은 부가기능을 적용 하더라도 ProxyFactoryBean 하나로 충분하다.
    - ProxyFactoryBean에 이 Methodlnterceptor를 설정해줄 때는 일반적인 DI 경우 처럼 수정자 메소드를 사용하는 대신 addAdvice()라는 메소드를 사용한다는 점도 눈 여겨봐야 한다.
    - add라는 이름에서 알 수 있듯이 ProxyFactoryBean에는 여러 개의 Methodlnterceptor를 추가할 수 있다.
- 스프링 ProxyFactoryBean을 이용한 방식
  - ![image](https://user-images.githubusercontent.com/36880294/58548376-2a440a80-8244-11e9-91fd-ba605465f10b.png)
  - 어드바이스와 포인트컷은 모두 프록시에 DI로 주입돼서 사용된다.
  - 두가지 모두 여러 프록시에서 공유가 가능하도록 만들어지기 때문에 스프링의 싱글톤 빈으로 등록이 가능하다.
  - 재사용 가능한 기능을 만들어두고 바뀌는 부분(콜백 오브젝트와 메소드 호출정보)만 외부에서 주입해서 이를 작엽 흐름(부가기능부여) 중에 사용하도록 하는 전형적인 템플릿/콜백 구조다.
    - 어드바이스가 템플릿
    - 타깃을 호출하는 기능을 갖고 있는 MethodInvocation 오브젝트가 콜백
  - 템플릿은 한 번 만들면 재사용이 가능하고 여러 빈이 공유해서 사용할 수 있듯이 어드바이스도 독립적인 싱글톤 빈으로 등록하고 DI를주입해서 여러 프록시가 사용하도록 만들 수 있다.
  - 프록시로부터 어드바이스와 포인트컷을 독립시키고 DI를 사용하게 한 것은 전형적인 전략 패턴 구조다.
  - 프록시와 ProxyFactoryBean 등의 변경 없이도 기능을 자유롭게 확장할 수 있는 OCP를 충실히 지키는 구조가 되는 것이다.
- 어드바이스
  - 타깃 오브젝트에 종속되지 않는 순수한 부가기능을 담은 오브젝트
- 포인트컷
  - 메소드 선정 알고리즘을 담은 오브젝트
  - 타깃 오브젝트의 메소드 중에서 어떤 메소드에 부가기능을 적용할지를 선정해주는 역할
- 어드바이저
  - 어드바이스와 포인트컷을 묶은 오브젝트
  - 포인트컷(메소드 선정 알고리즘) + 어드바이스(부가기능)

#### 6.4.2 ProxyFactoryBean 적용
- TransactionAdvice
  ```java
  package springbook.learningtest.jdk.proxy;
  /// ...
  public class TransactionAdvice implements MethodInterceptor {  // 스프링의 어드바이스 인터페이스 구현
    PlatformTransactionManager transactionManager;

    public void setTransactionManager(PlatformTransactionManager transactionManager) {
      this.transactionManager = transactionManager;
    }

    public Object invoke(MethodInvocation invocation) throws Throwable {  // 타깃을 호출하는 기능을 가진 콜백 오브젝트를 프록시로부터 받는다. 덕분에 어드바이스는 특정 타깃에 의존하지 않고 재사용 가능하다.
      TransactionStatus status = this.transactionManager.getTransaction(new DefaultTransactionDefinition());
      try {
        Object ret = invocation.proceed();  // 콜백을 호출해서 타깃 메소드를 실행한다. 타깃 메소드 호출 전후로 필요한 부가기능을 넣을 수 있다. 경우에 따라서 타깃이 아예 호출되지 않게 하거나 재시도를 위한 반복적인 호출도 가능하다.
        this.transactionManager.commit(status);
        return ret;
      } catch (RuntimeException e) {  // JDK 다이내믹 프록시가 제공하는 Method와는 달리 스프링의 MethodInvocation을 통한 타깃 호출은 예외가 포장되지 않고 타깃에서 보낸 그대로 전달된다.
        this.transactionManager.rollback(status);
        throw e;
      }
    }
  }
  ```
- 트랜잭션 어드바이스 빈 설정
  ```xml
  <bean id="transactionAdvice" class="springbook.user.service.TransactionAdvice">
    <property name="transactionManager" ref="transactionManager"/>
  </bean>
  ```
- 포인트컷 빈 설정
  ```xml
  <bean id="transactionPointcut" class="org.springframework.aop.support.NameMatchMethodPointcut">
    <property name="mappedName" value="upgrade*"/>
  </bean>
  ```
- 어드바이저 빈 설정
  ```xml
  <bean id="transactionAdvisor" class="org.springframework.aop.support.DefaultPointcutAdvisor">
    <property name="advice" ref="transactionAdvice"/>
    <property name="pointcut" ref="transactionPointcut"/>
  </bean>
  ```
- ProxyFactoryBean 설정
  ```xml
  <bean id="userService" class="org.springframework.aop.framework.ProxyFactoryBean">
    <property name="target" ref="userServicelmpl"/>
    <property name="interceptorNames">
      <list>
        <value>transactionAdvisor</value>
      </list>
    </property>
  </bean>
  ```
  - 어드바이저는 interceptorNames라는 프로퍼티를 통해 넣는다.
  - 프로퍼티 이름 이 advisor가 아닌 이유는 어드바이스와 어드바이저를 혼합해서 설정할 수 있도록 하기 위해서다.
  - property 태그의 ref 애트리뷰트를 통한 설정 대신 list와 value 태그를 통해 여러 개의 값을 넣을 수 있도록 하고 있다.
  - value 태그에는 어드바 이스 또는 어드바이저로 설정한 빈의 아이디를 넣으면 된다.
  - 한 개 이상을 넣을 수 있 다.
  - 타깃의 모든 메소드에 적용해도 좋기 때문에 포인트컷의 적용이 필요 없다면 transactionAdvice라고 넣을 수 있다.
- ProxyFactoryBean을 이용한 트랜잭션 테스트
  ```java
  @Test
  @DirtiesContext  // 컨텍스트 설정을 변경하기 때문에 여전히 필요하다.
  public void upgradeAllorNothing() {
    TestUserService testUserService = new TestUserService(users.gert(3).getId());
    testUserService.setUserDao(userDao);
    testUserService.setMailSender(mailSender);

    ProxyFactoryBean txProxyFactoryBean = context.getBean("&userService", ProxyFactoryBean.class);  // userService 빈은 이제 스프링의 ProxyFactoryBean이다.
    txProxyFactoryBean.setTarget(testUserService);
    UserService txUserService = (UserSErvice) txProxyFactoryBean.getObject();  // FactoryBean 타입이므로 동일하게 getObejct()로 프록시를 가져온다.
  }
  ```
- 어드바이스와 포인트컷의 재사용
  - ProxyFactoryBean은 스프링의 DI와 템플릿/콜백 패턴, 서비스 추상화 등의 기법이 모두 적용된 것이다.
    - 덕분에 독립적이며 여러 프록시가 공유할 수 있는 어드바이스와 포인트컷으로 확장 기능을 분리할 수 있었다.
  - 이제 UserService 외에 새로운 비즈니스 로직을 담은 서비스 클래스가 만들어져도 이미 만들어둔 TransactionAdvice를 그대로 재시용할 수 있다.
  - 메소드의 선정을 위한 포인트컷이 필요하면 이름 패턴만 지정해서 ProxyFactoryBean에 등록해주면 된다.
    - 트랜잭션을 적용할 메소드의 이름은 일관된 명명 규칙을 정해두면 하나의 포인트컷으로 충분할 수도 있다.
  - ProxyFactoryBean, Advice, Pointcut을 적용한 구조
    - ![image](https://user-images.githubusercontent.com/36880294/58550068-efdc6c80-8247-11e9-8d02-6ba1e778f242.png)
    - 트랜잭션 부가기능을 담은 TransactionAdvice는 하나만 만들어서 싱글톤 빈으로 등록해주면, DI 설정을 통해 모든 서비스에 적용이 기능하다.
    - 메소드 선정 방식이 달라지는 경우만 포인트컷의 설정을 따로 등록하고 어드바이저로 조합해서 적용해주면 된다.

## 6.5 스프링 AOP

#### 6.5.1 자동 프록시 생성
- 중복 문제의 접근 방법
  - 반복적인 ProxyFactoryBean 설정 문제는 설정 자동등록 기법으로 해결할 수 없을까?
  - 실제 빈 오브젝트가 되는 것은 ProxyFactoryBean을 통해 생성되는 프록시 그 자체이므로 프록시가 자동으로 빈으로 생성되게 할 수는 없을까?
- 빈 후처리기를 이용한 자동 프록시 생성기
  - BeanPostProcessor 인터페이스를 구현해서 만드는 빈 후처리기
    - 스프링 빈 오브젝트로 만들어지고 난 후에 빈 오브젝트를 다시 가공할 수 있게 해준다.
  - DefaultAdvisorAutoProxyCreator
    - 어드바이저를 이용한 자동 프록시 생성기
  - 빈 후처리기를 스프링에 적용하는 방법
    - 빈 후처리기 자체를 빈으로 등록한다.
  - 스프링은 빈 후처리기가 빈으로 등록되어 있으면 빈 오브젝트가 생성될 때마다 빈 후처리기에 보내서 후처리 작업을 요청한다.
    - 스프링이 설정을 참고해서 만든 오브젝트가 아닌 다른 오브젝트를 빈으로 등록시키는 것이 가능하다.
  - 자동 프록시 생성 빈 후처리기
    - 스프링이 생성하는 빈 오브젝트의 일부를 프록시로 포장하고 프록시를 빈으로 대신 등록한다.
  - 빈 후처리기를 이용한 프록시 자동생성
    - ![image](https://user-images.githubusercontent.com/36880294/58552775-2ddc8f00-824e-11e9-9899-92424797948a.png)
- 확장된 포인트컷
  - 두 가지 기능을 정의한 Pointcut 인터페이스
    ```java
    public interface Pointcut {
      ClassFilter getClassFilter();  // 프록시를 적용할 클래스인지 확인해준다.
      MethodMatcher getMethodMatcher();  // 어드바이스를 적용할 메소드인지 확인해준다.
    }
    ```
  - 빈 오브젝트 자체를 선택한다.
  - 오브젝트 내의 메소드를 선택한다.

#### 6.5.2 DefaultAdvisorAutoProxyCreator의 적용
- 클래스 필터를 적용한 포인트컷 작성
  ```java
  package springbook.learningtest.jdk.proxy;
  // ...
  public class NameMatchClassMethodPointcut extends NameMatchMethodPointcut {
    public void setMappedClassName(String mappedClassName) {
      this.setClassFilter(new SimpleClassFilter(mappedClassName));  // 모든 클래스를 다 허용하던 디폴트 클래스 필터로 프로퍼티로 받은 클래스 이름을 이용해서 필터를 만들어 덮어씌운다.
    }

    static class SimpleClassFilter implements ClassFilter {
      String mappedName;

      private SimpleClassFilter(String mappedName) {
        this.mappedName = mappedname;
      }

      public boolean matches(Class<?> clazz) {
        return PatternMatchUtils.simpleMatch(mappedName, clazz.getSimpleName());  // 와일드카드(*)가 들어간 문자열 비교를 지원하는 스프링의 유틸리티 메소드다. "name, name", "name" 세가지 방식을 모두 지원한다.
      }
    }
  }
  ```
- 어드바이저를 이용하는 자동 프록시 생성기 등록
  - DefaulAdvisorAutoProxyCreator
    - 등록된 빈 중에서 Advisor 인터페이스를 구현한 것을 모두 찾는다.
    - 등록 방법
      ```xml
      <bean class="org.springframework.aop.framework.autoproxy.DefaultAdvisorAutoProxyCreator"/>
      ```
- 포인트컷 등록
  ```xml
  <bean id="transactionPointcut" class="springbook .service.NameMatchCl assMethodPointcut">
    <property name="mappedClassName" value="*Servicelmpl"/>
    <property name="mappedName" value="upgrade*"/>
  </bean>
  ```
  - ServiceImpl로 이름이 끝나는 클래스와 upgrade로 시작하는 메소드를 선정해주는 포인트컷
- 어드바이스와 어드바이저
  - 어드바이저를 이용하는 자동 프록시 생성기인 DefaultAdvisorAutoProxyCreator에 의해 자동수집되고, 프록시 대상 선정 과정에 참여하며 자동 생성된 프록시에 다이내믹하게 DI 돼서 동작하는 어드바이저가 된다.
- ProxyFactoryBean 제거와 서비스 빈의 원상복구
  - 프록시 팩토리 빈을 제거한 후의 빈 설정
    ```xml
    <bean id="userService" class="springbook.service.UserServicelmpl">
      <property name="userDao" ref="userDao"/>
      <property name="mailSender" ref="mailSender"/>
    </bean>
    ```
    - 더 이상 명시적인 프록시 팩토리 빈을 등록하지 않는다.

#### 6.5.3 포인트컷 표현식을 이용한 포인트컷
- 포인트컷 표현식
  - 정규식이나 JSP의 EL과 비슷한 일종의 표현식 언어를 사용해서 포인트컷을 작성할 수 있도록 하는 방법
  - AspectJExpressionPointcut 클래스를 사용하면 된다.
    - 클래스와 메소드의 선정 알고리즘을 포인트컷 표현식을 이용해 한 번에 지정할 수 있게 해준다.
    - 자바의 RegEx 클래스가 지원하는 정규식처럼 간단한 문자열로 복잡한 선정조건을 쉽게 만들어낼 수 있는 강력한 표현식을 지원한다.
  - 문법
    - ![image](https://user-images.githubusercontent.com/36880294/58554207-5914ad80-8251-11e9-91d5-6de5c4954e50.png)
  - AspectJ 포인트컷 표현식의 메소드 선정 결과
    - ![image](https://user-images.githubusercontent.com/36880294/58554347-adb82880-8251-11e9-8e17-4bcd7c1cc078.png)
- 포인트컷 표현식을 이용하는 포인트컷 적용
  - 포인트컷 표현식을 사용한 빈 설정
    ```xml
    <bean id="transactionPointcut" class="org.springframework.aop.aspectj.AspectJExpressionPointcut">
      <property name="expression" value="execution(* *..*Servicelmpl.upgrade*(..))"/>
    </bean>
    ```
- 타입 패턴과 클래스 이름 패턴
  - 포인트컷 표현식의 클래스 이름에 적용되는 패턴은 타입 패턴이다.

#### 6.5.4 AOP란 무엇인가?
- 트랜잭션 서비스 추상화
  - 트랜잭션 경계설정 코드를 비즈니스 로직을 담은 코드에 넣으면서 맞닥뜨린 첫 번째 문제는 특정 트랜잭션 기술에 종속되는 코드가 돼버린다는 것이었다.
  - 서비스 추상화 기법을 적용한 덕분에 비즈니스로직 코드는 트랜잭션을 어떻게 처리해야 한다는 구체적인 방법과 서버환경에서 종속되지 않는다.
  - 구체적인 구현 내용을 담은 의존 오브젝트는 런타임 시에 다이내믹하게 연결해 준다는 DI를 활용한 전형적인 접근 방법이었다.
  - 트랜잭션 추상화란 결국 인터페이스와 DI를 통해 무엇을 하는지는 남기고, 그것을 어떻게 하는지를 분리한 것이다.
- 프록시와 데코레이터 패턴
  - DI를 이용해 데코레이터 패턴을 적용하는 방법으로 비즈니스 로직을 담은 클래스의 코드에는 전혀 영향을 주지 않으면서 트랜잭션이라는 부가기능을 지유롭게 부여할 수 있는 구조를 만들었다.
    - 트랜잭션을 처리하는 코드는 일종의 데코레이터에 담겨서 클라이언트와 비즈니스 로직을 담은 타깃 클래스 사이에 존재하도록 만들었다.
    - 클라이언트가 일종의 대리자인 프록시 역할을 하는 트랜잭션 데코레이터를 거쳐서 타깃에 접근할 수 있게 됐다.
- 다이내믹 프록시와 프록시 팩토리 빈
  - 프록시 클래스 없이도 프록시 오브젝트를 런타임 시에 만들어주는 JDK 다이 내믹 프록시 기술을 적용한 덕분에 프록시 클래스 코드 작성의 부담도 덜고, 부가 기능부여 코드가 여기저기 중복돼서 나타나는 문제도 일부 해결할수 있었다
    - but, 통일한 기능의 프록시를 여러 오브젝트에 적용할 경우 오브젝트 단위로 중복이 일어나는 문제는 해결하지 못했다.
  - JDK 다이내믹 프록시와 같은 프록시 기술을 추상화한 스프링의 프록시 팩토리 빈을 이용해서 다이내믹 프록시 생성 방법에 DI를 도입했다.
    - 내부적으로 템플릿/콜백 패턴을 활용하는 스프링의 프록시 팩토리 빈 덕분에 부가기능을 담은 어드바이스와 부가기능 선정 알고리즘을 담은 포인트컷은 프록시에서 분리될 수 있었고 여러 프록시에서 공유해서 사용할 수 있게 됐다.
- 자동 프록시 생성 방법과 포인트컷
  - 트랜잭션 적용 대상이 되는 빈마다 일일이 프록시 팩토리 빈을 설정해줘야 한다는 부담이 남아있었다.
  - 스프링 컨테이너의 빈 생성 후처리 기법을 활용해 컨테이너 초기화 시점에서 지동으로 프록시를 만들어주는 방법을 도입했다.
    - 프록시를 적용할 대상을 일일이 지정하지 않고 패턴을 이용해 자동으로 선정할 수 있도록, 클래스를 선정 하는 기능을 담은 확장된 포인트컷을 사용했다.
  - 최종적으로는 포인트컷 표현식이라는 좀 더 편리하고 깔끔한 방법을 활용해서 간단한 설정만으로 적용 대상을 손쉽게 선택할 수 있게 됐다.
- 부가기능의 모듈화
  - 트랜잭션 적용 코드는 기존에 써왔던 방법으로는 간단하게 분리해서 독립된 모듈로 만들 수가 없었다.
    - 트랜잭션 경계설정 기능은 다른 모율의 코드에 부가적으로 부여되는 기능 이라는 특징이 있기 때문이다.
    - 트랜잭션 코드는 한데 모을 수 없고, 애플리케이션 전반에 여기저기 흩어져 있다.
  - 트랜잭션과 같은 부가기능은 핵심기능과 같은 방식으로는 모듈화하기가 매우 힘들다.
    - 이름 그대로 부가기능이기 때문에 스스로는 독립적인 방식으로 존재해서는 적용되기 어렵기 때문이다.
    - 트랜잭션 부가기능이란 트랜잭션 기능을 추가해줄 다른 대상, 즉 타깃이 존재해야만 의미가 있다.
    - 각 기능을 부가할 대상인 각 타깃의 코드 안에 침투하거나 긴밀하게 연결되어 있지 않으면 안된다.
  - 부가기능을 모듈화하기 위해서?
    - DI, 데코레이터 패턴, 다이내믹 프록시, 오브젝트 생성 후처리, 자동 프록시 생성, 포인트컷과 같은 기법을 사용하여 모듈화 하였다.
- AOP(애스펙트 지향 프로그래밍)
  - 애스펙트(aspect)
    - 그 자체로 애플리케이션의 핵심기능을 담고 있지는 않지만, 애플리케이션을 구성하는 중요한 한 가지 요소이고 핵심기능에 부가되어 의미를 갖는 특별한 모듈
    - 부가될 기능을 정의한 코드인 어드바이스와, 어드바이스를 어디에 적용할지를 결정하는 포인트컷을 함께 갖고 있다.
    - 지금 사용하고 있는 어드바이저는 아주 단순한 형태의 애스펙트
  - 독립 애스펙트를 이용한 부가기능의 분리와 모듈화
    - ![image](https://user-images.githubusercontent.com/36880294/58555805-6b90e600-8255-11e9-9f05-a5432c0b98ff.png)
    - 2차원적인 평면 구조에서는 어떤 설계 기법을 동원해도 해결 할 수 없었던 것을, 3차원의 다면체 구조로 가져가면서 각각 성격이 다른 부가기능은 다른 면에 존재하도록 만들었다.
    - 독립된 측면에 존재하는 애스펙트로 분리한 덕에 핵심기능은 순수하게 그 기능을 담은 코드로만 존재하고 독립적으로 살펴볼 수 있도록 구분된 면에 존재하게 된 것이다.
    - 물론 애플리케이션의 여러 다른 측면에 존재하는 부가기능은 결국 핵심기능과 함께 어우러져서 동작하게 되어 있다.
    - 결국 런타임 시에는 왼쪽의 그림처럼 각 부가기능 애스펙트는 자기가 필요한 위치에 다이내믹하게 참여하게 될 것이다.
    - 하지만 설계와 개발은 오른쪽 그림처럼 다른 특성을 띤 애스펙트들을 독립적인 관점으로 작성하게 할 수 있다.
  - 애플리케이션의 핵심적인 기능에서 부가적인 기능을 분리해서 애스펙트라는 독특한 모듈로 만들어서 설계하고 개발하는 방법을 AOP(Aspect Oriented Programming)이라고 부른다.
    - AOP는 OOP를 돕는 보조적인 기술이다.
      - 애스펙트를 분리함으로써 핵심기능을 설계하고 구현할 때 객체지향적인 가치를 지킬 수 있도록 도와주는 것
    - 애플리케이션을 특정한 관점을 기준으로 바라볼 수 있게 해준다는 의미에서 AOP를 관점 지향 프로그래밍이라고도 한다.

#### 6.5.5 AOP 적용기술
- 프록시를 이용한 AOP
  - 스프링은 IoC/DI 컨테이너와 다이내믹 프록시, 데코레이터 패턴, 프록시 패턴, 자동 프록시 생성 기법, 빈 오브젝트의 후처리 조작 기법 등의 다양한 기술을 조합해 AOP를 지원하고 있다.
  - 그 중 가장 핵심은 프록시를 이용했다는 것이다.
  - 프록시로 만들어서 DI로 연결된 빈 사이에 적용해 타깃의 메소드 호출 과정에 참여해서 부가기능을 제공해주 도록 만들었다.
  - 스프링 AOP는 자바의 기본 JDK와 스프링 컨테이너 외에는 특별한 기술이나 환경을 요구하지 않는다.
  - 독립적으로 개발한 부가기능 모듈을 다양한 타깃 오브젝트의 메소드에 다이내믹하게 적용해주기 위해 가장 중요한 역할을 맡고 있는 게 바로 프록시다.
  - 그래서 스프링 AOP는 프록시 방식의 AOP라고 할 수 있다.
- 바이트코드 생성과 조작을 통한 AOP
  - AspectJ
  - 타깃 오브젝트를 뜯어고쳐서 부가 기능을 직접 넣어주는 직접적인 방법을 사용한다.
    - 부가기능을 넣는다고 타깃 오브젝트의 소스코드를 수정할 수는 없으니, 컴파일된 타깃의 클래스 파일 자체를 수정하거나 클래스가 JVM에 로딩되는 시점을 가로채서 바이트코드를 조작하는 복잡한 방법을 사용한다.
  - 스프링과 같은 컨테이너가 사용되지 않는 환경에서도 손쉽게 AOP의 적용이 가능해진다.
  - 프록시 방식보다 훨씬 강력하고 유연한 AOP가 가능하다.
    - 바이트코드를 직접 조작해서 AOP를 적용 하면 오브젝트의 생성, 펼드 값의 조회와 조작, 스태틱 초기화 등의 다양한 작업에 부가기능을 부여해줄 수 있다.
    - 타깃 오브젝트가 생성되는 순간 부가기능을 부여해주고 싶을 수도 있다.
  - 바이트코드 조작을 위해 JVM의 실행 옵션을 변경하거나, 별도의 바이트코드 컴파일러를 사용하거나, 특별한 클래스 로더를 사용하게하는 등의 번거로운 작업이 필요하다.

#### 6.5.6 AOP의 용어
- 타깃
  - 부가기능을 부여할 대상
  - 핵심기능을 담은 클래스일 수도 있지만 경우에 따라서는 다른 부가기능을 제공하는 프록시 오브젝트일 수도 있다.
- 어드바이스
  - 타깃에게 제공할 부가기능을 담은 모듈
  - 오브젝트로 정의하기도 하지만 메소드 레벨에서 정의할 수도 있다.
- 조인 포인트
  - 어드바이스가 적용될 수 있는 위치
  - 스프링의 프록시 AOP에서 조인 포인트는 메소드의 실행 단계뿐이다.
  - 타깃 오브젝트가 구현한 인터페이스의 모든 메소드는 조인 포인트가 된다.
- 포인트컷
  - 어드바이스를 적용할 조인 포인트를 선별하는 작업 또는 그 기능을 정의한 모듈
  - 스프링 AOP의 조인 포인트는 메소드의 실행이므로 스프령의 포인트컷은 메소드를 선정하는 기능을 갖고 있다.
- 프록시
  - 클라이언트와 타깃 사이에 투명하게 존재하면서 부가기능을 제공히는 오브젝트
  - DI를 통해 타깃 대신 클라이언트에게 주입되며 클라이언트의 메소드 호출을 대신 받아서 타깃에 위임해주면서, 그 과정에서 부가기능을 부여한다.
  - 스프링은 프록시를 이용해 AOP를 지원한다.
- 어드바이저
  - 포인트컷과 어드바이스를 하나씩 갖고 있는 오브젝트
  - 어떤 부가기능(어드바이스)을 어디에(포인트컷) 전달할 것인가를 알고 있는 AOP의 가장 기본이 되는 모듈
  - 스프링은 자동 프록시 생성기가 어드바이저를 AOP 작업의 정보로 활용한다.
  - 어드바이저는 스프링 AOP에서만 사용되는 특별한 용어이고, 일반적인 AOP에서는 시용되지 않는다.
- 애스펙트
  - OOP의 클래스와 마찬가지로 애스펙트는 AOP의 기본 모듈
  - 한 개 또는 그 이상 의 포인트컷과 어드바이스의 조합으로 만들어지며 보통 싱글톤 형태의 오브젝트로 존재한다.
    - 따라서 클래스와 같은 모듈 정의와 오브젝트와 같은 실제(인스턴스)의 구분이 특별히 없다.

#### 6.5.7 AOP 네임스페이스
- 스프링의 프록시 방식 AOP를 적용하려면 최소한 네 가지 빈을 등록해야 한다.
  - 자동 프록시 생성기
    - 스프링의 DefaultAdvisorAutoProxyCreator 클래스를 빈으로 등록한다.
    - 다른 빈을 DI 하지도 않고 자신도 DI 되지 않으며 독립적으로 존재한다.
    - 애플리케이션 컨텍스트가 빈 오브젝트를 생성하는 과정에 빈 후처리기로 참여한다.
    - 빈으로 등록된 어드바이저를 이용해서 프록시를 자동으로 생성하는 기능을 담당
  - 어드바이스
    - 부가기능을 구현한 클래스를 빈으로 등록한다.
    - TransactionAdvice는 AOP 관련 빈 중에서 유일하게 직접 구현한 클래스를 사용한다.
  - 포인트컷
    - AspectJExpressionPointcut을 빈으로 등록하고 expression 프로퍼티에 포인트컷 표현식을 넣어주면 된다.
    - 코드 작성 필요 X
  - 어드바이저
    - 스프링의 DefaultPointcutAdvisor 클래스를 빈으로 등록해서 사용한다.
    - 어드바이스와 포인트컷을 프로퍼티로 참조하는 것 외에는 기능은 없다.
    - 자동 프록시 생성기에 의해 자동 검색되어 사용된다.
  - 이런 빈들은 스프링 컨테이너에 의해 자동으로 인식돼서 특별한 작업을 위해 사용된다.
- AOP 네임스페이스
  - 스프링에서 AOP를 위해 기계적으로 적용하는 빈들을 간편한 방법으로 등록할 수 있다.
  - AOP와 관련된 태그를 정의해둔 aop 스키마를 제공한다
  - aop 스키마에 정의된 태그는 별도의 네임스페이스를 지정해서 디폴트 네임스페이스의 <bean> 태그와 구분해서 사용할 수 있다.
  - aop 네임스페이스 선언을 설정파일에 추가해줘야 태그를 사용 가능하다.
  - aop 네임스페이스 선언
    - ![image](https://user-images.githubusercontent.com/36880294/58558460-e0671e80-825b-11e9-8eda-f8affba76103.png)
  - aop 네임스페이스를 적용한 AOP 설정 빈
    - ![image](https://user-images.githubusercontent.com/36880294/58558506-ff65b080-825b-11e9-9653-ec4df00264cf.png)
- 어드바이저 내장 포인트컷
  - 포인트컷을 내장한 어드바이저 태그
    - ![image](https://user-images.githubusercontent.com/36880294/58558565-2328f680-825c-11e9-8292-dcef984e34df.png)
