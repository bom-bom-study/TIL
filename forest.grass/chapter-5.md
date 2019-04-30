# toby-spring

## 04월 11일

### keyword

#### 서비스 추상화
- 사용방법과 형식은 다르지만 기능과 목적이 유사한 기술이 존재
- 자바의 표준 기술 중에서도 플랫폼과 컨텍스트에 차이가 있거나 발전한 역사가 다르기 때문에 목적이 유사한 여러 기술이 공존
  - 환경과 상황에 따라서 기술이 바뀌고, 그에 따른 다른 API를 사용하고 다른 스타일의 접근 방법을 따라야 한다는 건 매우 피곤할 일이다.
- 비슷한 여러 종류의 기술을 추상화하는 방법을 알아보자

#### 사용자 레벨 관리 기능 추가
- 기존 UserDAO에 비즈니스 로직을 추가
  - 사용자 관리 기능
  - 활동내역을 참고해서 레벨을 조정해주는 기능


#### enum
- 일정한 종류의 정보를 문자열로 넣는 것은 별로 좋아 보이지 않는다.
  - BASIC, SILVER, GOLD
- 정수값으로 치환하여 상수로 있다고 하여도 사이드 이펙트가 발생할 수 있다.
  - 상수로 정의 해놓은 정수값이 벗어날경우?
<pre><code>
public enum User {
  BASIC(1), SILVER(2), GOLD(3);

  private final int value;
 
  Level(int value) {
    this.value = value;
  }

  public int intValue() {
    return value;
  }

  public staic Level valueOf(int value) {
    switch(value) {
      case 1 : return BASIC;
      case 2 : return SILVER;
      case 3 : return GOLD;
      default : throw new AssertionError("Unknown value: " + value);
    }
  }
}
</code></pre>

#### 포괄적인 테스트
- 포괄적인 테스트를 만들면서 개발을 하자
- 코드 수정 및 개발후 검증을 거쳐 배포하여야한다.
- 테스트코드가 있으니 쉽게 검증 할 수 있다.

#### UPDATE SQL문 검증
- 첫번째 방법
  - JdbcTemplate의 update()가 돌려주는 리턴 값을 확인하는 것이다.
- 두번째 방법
  - 테스트를 보강해서 원하는 사용자 외의 정보는 변경되지 않았음을 직접 확인하는 것이다.

#### 서비스 로직 만들기
- UserService layer 추가
  - 등급업 서비스 로직 추가
    - 각등급문의 if문을 통해 flag를 설정 한다
    - flag가 true가 되면 DAO의 업데이트 메서드 호출
    - 픽스처를 만들어 각각의 경우 테스트코드 작성 및 검증
  - 유저 추가 서비스 로직 추가
  
#### 코드 개선
- 코드 작성후 자신에 물어보자
  - 코드에 중복된 부분은 없는가?
  - 코드가 무엇을 하는 것인지 이해하기 불편하지 않은가?
  - 코드가 자신이 있어야 할 자리에 있는가?
  - 앞으로 변경이 일어난다면 어떤 것이 있을 수 있고, 그 변화에 쉽게 대응할 수 있게 작성되어 있는가?
- 개선전
<pre><code>
if(user.getLevel() == Level.BASIC && user.getLogin() >= 50) {
  user .setLevel(Level.SILVER);
  changed = ture;
}
...
if(changed) { userDao.update(user); }

하나의 서비스 로직에 하는 일이 너무 많다.
코드를 보고 이해하기가 어렵다.(가독성 저하)
추후 Level이 추가 될 경우 새로운 if문이 추가 되어 더욱 복잡성이 올라간다.
</code></pre>
- 개선후
<pre><code>
1.전체적인 흐름을 가지는 서비스 로직
public void upgradeLevels() {
  List<User> users = userDao.getAll();
  for(User user : users) {
    if(canUpgradeLevel(user)) {
      upgradeLevel(user);
    }
  }
}

2.Level 체크를 위한 메서드
private boolean canUpgradeLevel(User user) {
  Level currentLevel = user.getLevel();
  switch(currentLevel) {
    case BASIC : return (user.getLogin() >= 50);
    case SILVER : return (user.getRecomment() >= 30);
    case GOLD : return false;
    default : throw new IllegalArgumentException("Unknown Level : " + currentLevel);
  }
}

3.Level 업그레이드를 위한 메서드
private void upgradeLevel(User user) {
  if(user.getLevel() == Level.BASIC) user.setLevel(Level.SILVER);
  else if (user.getLevel() == Level.SILVER) user.setLevle(Level.GOLD);
  userDao.update(user);
}
</code></pre>
- 3.Level 업그레이드를 위한 메서드에서 다음 Level이 무엇인지 확인하는 책임을 enum에게 넘겨주자
<pre><code>
1.User Class에 추가
public void upgradeLevel() {
  Level nextLevel = this.level.nextLevel();
  if (nextLevel == null) {
    throw new IllegalStateException(this.level + "은 업그레이드가 불가능합니다.");
  } else {
    this.level = nextLevel;
  }
}

2.UserService
private void upgradeLevel(User user) {
  user.upgradeLevel();
  userDao.update(user);
}
</code></pre>
- 리팩토링후 가독성이 더 좋아졌다.
- 각각의 메서드가 각자의 책임을 맞게 가지고 있다고 할 수 있다.
- 항상 코드를 더 깔금하고 유연하면서 변화에 대응하기 쉽고 테스트하기 좋게 만들려고 노력해야 함을 기억하자

#### UserServiceTest 개선
- 어떤 테스트를 할 것인지 코드만 보고 알 수 있게 하자
- 상수를 이용하자

#### 트랜잭션
- 롤백과 커밋
- TestCode를 통하여 트랜잭션 상황을 구현하자
- UserService를 상속시켜 예외상황 구현
- 트랜잭션 : 더 이상 나눌 수 없는 단위 작업

#### 트랜잭션 경계설정
- setAutoCommit(false)
- commit()
- rollback()
- 로컬 트랜잭션 : 하나의 DB 커넥션 안에서 만들어지는 트랜잭션

#### Connection Object를 직접 생성하여 넘겨주면 생기는 문제점들
- 1.DB 커넥션을 비롯한 리소스의 깔금한 처리를 가능하게 했던 JdbcTemplate을 더 이상 활용할 수 없다
- 2.DAO의 메소드와 비즈니스 로직을 담고 있는 UserService의 메소드에 Connection 파라미터가 추가돼야 한다.
- 3.Connection 파라미터가 UserDao 인터페이스 메소드에 추가되면 UserDao는 더이상 데이터 액세스 기술에 독립적일 수 없다.
- 4.DAO 메소드에 Connection 파라미터를 받게 하면 테스트 코드에도 영향을 미친다.

#### 해결법 - 트랜잭션 동기화
- UserService에서 트랜잭션을 시작하기 위해 만든 Connection 오브젝트를 특별한 저장소에 보관해두고, 이후에 호출되는 DAO의 메소드에서는 저장된 Connection을 가져다가 사용하게 하는 것이다.
- 트랜잭션 동기화 저장소는 작업 스레드마다 독립적으로 Connection 오브젝트를 저장하고 관리하기 때문에 다중 사용자를 처리하는 서버의 멀티스레드 환경에서도 충돌이 날 염려는 없다.

#### 스프링이 제공하는 트래잭션 동기화 관리 클래스
- TransactionSynchronizationManager
- 위 클래스를 이용해 먼저 트랜잭션 동기화 작업을 초기화 하도록 요청
- DataSourceUtils을 이용하여 Connection 오브젝트를 가져온다.
  - DataSource에서 Connection을 가져오지 않고, 스프링이 제공하는 유틸리티에서 가져오는 이유는
  - DataSourceUtils는 Connection 오브젝트를 생성해줄뿐만 아니라 트랜잭션 동기화에 사용하도록 저장소에 바인딩해주기 때문이다.
- 저장소에 바인딩되 Connection을 JdbcTemplate가 알아서 가져와 사용하게 된다.

#### JdbcTemplate과 트랜잭션 동기화
- DB커넥션이 없을 경우 알아서 생성하여 사용한다.
- 동기화된 커넥션이 있을 경우 알아서 저장소에 있는 커넥션을 들고와 사용한다.
- 3대 기능
  - try/catch/finally 작업 흐름 지원
  - SQLException의 예외 변환
  - 유연한 DB커넥션 사용

#### 글로벌 트랜잭션
- 하나의 DB가 아닌 여러개의 DB 사용시에 트랜잭션 처리는?
- 하나의 커넥션으로 여러개의 DB커넥션을 담을수가 없다.(로컬 트랜잭션으로 처리 불가능)
- 별도의 트랜잭션 관리자를 통해 트랜잭션을 관리하는 글로벌 트랜잭션 방식이 필요하다.
- 글로벌 트랜잭션을 적용해야 트랜잭션 매니저를 통해 여러 개의 DB가 참여하는 작업을 하나의 트랜잭션으로 처리 할 수 있다.
- 자바는 JTA를 통해 글로벌 트랜잭션을 지원한다.

#### 글로벌 트랜잭션 적용시 문제점
- UserService의 메소드 안에서 트랜잭션 경계설정 코드를 제거할 수는 없다.
- 특정 기술에 의존적인 Connection, UserTransaction, Session/Transaction API등에 종속되어 버린다.
- 유사한 코드를 추상화를 통해서 처리하자

#### 스프링의 트랜잭션 서비스 추상화
- PlatformTransactionManager를 통한 추상화
- 로컬 트랜잭션 : DataSourceTransactionManager
- 글로벌 트랜잭션 : JTATransactionManger
- JPA 트랜잭션 : HibernateTransactionManager
- 스프링에서 싱글톤으로 제공하기 있기때문에 bean으로 생성하여 DI로 유연하게 이용할 수 있다.

#### 서비스 추상화 단일 책임 원칙
- 스프링에서 제공하는 트랜잭션 추상화 기법을 이용하여 기술에 종속적이지 않게 개발할 수 있다
- 수평적인 분리 : UserDao와 UserService는 각각 담당하는 코드의 기능적인 관심에 따라 분리되고, 서로 불필요한 영향을 주지 않으면서 독자적으로 확장이 가능하도록 만든 것
- 수직적인 분리 : 애플리케이션의 비즈니스 로직과 그 하위에서 동작하는 로우레벨의 트랜잭션 기술이라는 아예 다른 계층의 특성을 갖는 코드를 분리(트랜잭션의 추상화)
- DI를 통하여 분리
  - 애플리케이션의 로직의 종류에 따른 수평적인 구분이든, 로직과 기술이라는 수직적인 구분인든 모두 결합도가 낮으며
  - 서로 영향을 주지 않고 자유롭게 확장될 수 있는 구조를 만들 수 있는 데는 스프링의 DI가 중요한 역할을 하고 있따.
  - DI의 가치는 이렇게 관심, 책임, 성격이 다른 코드를 깔끔하게 분리하는데에 있다.

#### 단일 책임 원칙
- 하나의 모듈은 한가지 책임을 가져야 한다는 의미이다.
- 하나의 모듈이 바뀌는 이유는 한가지여야 한다
- UserService
  - 사용자 레벨을 관리 할 것인가
  - 어떻게 트랜잭션을 관리할 것인가
  - 두가지의 책임
  - 트랜잭션 추상화 기법을 통해 한가지의 책임으로 수정

#### 단일 책임 원칙의 장점
- 단일 책임 원칙을 잘지키고 있다면, 어떤 변경이 필요할 때 수정 대상이 명확해진다.
- 서비스하나가 여러개의 DAO를 사용하게 된다면?
- DAO 하나를 여러군데 서비스에서 사용하게 될 경우 DAO를 수정하게 된다면?
- 기술적인 변경이 어렵게 된다면?
- 적절하게 책임과 관심이 다른 다른 코드를 분리하고, 서로 영향을 주지 않도록 다양한 추상화 기법을 도입하고,
- 어플리케이션 로직과 기술/환경을 분리하는 등의 작업은 갈수록 복잡해지는 엔터프라이즈 어플리케이션에서는 반드시 필요하다.
- 이를 위한 핵심적인 도구가 바로 스프링이 제공하는 DI다.
- 단일 책임 원칙을 잘 지키는 코드를 만들려면 인터페이스를 도입하고 이를 DI로 연결해야 하며, 그결과로 단일 책임 원칙뿐 아니라 개방 폐쇄 원칙도 잘 지키고,
- 모듈 간에 결합도가 낮아서 서로의 변경이 영향을 주지 않고, 같은 이유로 변경이 단일 책임에 집중되는 응집도 높은 코드가 나온다.
- 다양항 디자인 패턴이 자연스럽게 적용이 된다.
- 테스트하기에도 유용하다.
- 스프링에서 제공하는 DI와 싱글톤 레지스트리 덕분에 편리하게 자동화된 테스트를 만들 수 있다.

#### 메일 서비스 추상화
- 두가지 추가 서비스 개발 필요
  - 1. 유저의 메일 관리
  - 2. 유저에게 메일 발송
- 실제 메일 서버를 통한 테스트를 할 필요가 없을 경우
- 서비스layer에서 javaMail까지 넘어 갈 필요가 없을 경우
- 추상화를 통해 유연하게 할 수 있게 만들자
- 스프링에서 MailSender를 제공(추상화)
- try/catch을 코드에서 사라져 가독성이 올라감
- DI/IOC를 통해 생성 코드를 밖으로 빼주어 단일책임원칙을 실현 시켜주자

#### 테스트 대역
- 테스트용으로 사용되는 특별한 오브젝트들
- 대부분 테스트 대상인 오브젝트의 의존 오브젝트가 되는 것들이다.
- 테스트 환경을 만들어주기 위해, 테스트 대상이 되는 오브젝트의 기능에만 충실하게 수행하면서 빠르게, 자주 테스트를 실행할 수 있도록 사용하는 이런 오브젝트를 테스트 대역이라고 한다.

#### 테스트 스텁
- 테스트 대상 오브젝트의 의존객체로서 존재하면서 테스트 동안에 코드가 정상적으로 수행할 수 있도록 돕는 것

#### 목 오브젝트(Mock Object)
- 테스트 가 수행 될 수 있도록 의존 오브젝트에 간접적으로 입력값을 제공해주면서
- 간접적인 출력 값까지 확인이 가능하게 해주는 오브젝트
