# 서비스 추상화

### 5.2 트랜잭션 서비스 추상화
정기 사용자 레벨 관리 작업을 하는 도중에 네트워크가 끊기거나 서버에 장애가 생겨서 작업을 완료할 수 없다면, 그때까지 변경된 사용자의 레벨을 모두 취소시키고 시스템 장애가 있어서 레벨 변경 작업을 못했다고 공지하고 다음 날 다시 시도하는 편이 낫다.

### 트랜잭션 경계설정
DB는 그 자체로 완벽한 트랜잭션을 지원한다. SQL을 이용해 다중 로우의 수정이나 삭제를 위한 요청을 했을 때 일부만 되고, 나머지는 안되고 실패로 끝나는 경우는 없다.
* 트랜잭션 롤백
    * 두 가지 작업이 하나의 트랜잭션이 되려면, 두 번째 SQL이 성공적으로 DB에서 수행되기 전에 문제가 발생할 경우 앞에서 처리한 SQL 작업도 취소시키는 작업
* 트랜잭션 커밋
    * 여러 개의 SQL을 하나의 트랜잭션으로 처리하는 경우에 모든 SQL 수행작업이 다 성공적으로 마무리됐다고 알려주는 것

### JDBC 트랜잭션의 트랜잭션 경계설정
애플리케이션 내에서 트랜잭션이 시작되고 끝나는 위치를 트랜잭션의 경계라고 부른다. 복잡한 로직의 흐름 사이에서 정확하게 트랜잭션 경계를 설정하는 일은 매우 중요한 작업이다.

``` java
Connection c = dataSource.getConnection(); // DB 커넥션 시작

c.setAutoCommit(false); // 트랜잭션 시작
try {
    PreparedStatement st1 = c.prepareStatement("update users ...");
    st1.executeUpdate();

    PreparedStatement st2 = c.prepareStatement("update users ...");
    st2.executeUpdate();

    c.commit(); // 트랜잭션 커밋
}
catch (Exception e) {
    r.rollback() // 트랜잭션 롤백
}

c.close(); // DB 커넥션 종료
```

JDBC의 트랜잭션은 하나의 Connection을 가져와 사용하다가 닫는 사이에 일어난다. 트랜잭션의 시작과 종료는 Connection 오브젝트를 통해 이뤄지기 때문이다. JDBC에서 기본 설정은 DB 작업을 수행한 직후에 자동 커밋이 되도록 되어 있기 때문에 트랜잭션을 시작하려면 자동커밋 옵션을 false로 만들어주면 된다.

트랜잭션이 한 번 시작되면 commit() 또는 rollback() 메소드가 호출될 때까지의 작업이 하나의 트랜잭션으로 묶인다.

이렇게 setAutoCommit(false)로 트랜잭션의 시작을 선언하고 commit() 또는 rollback()으로 트랜잭션을 종료하는 작업을 트랜잭션의 경계설정(transaction  demarcation)이라고 한다. 트랜잭션의 경계는 하나의 Connection이 만들어지고 닫히는 범위 안에 존재한다.

하나의 DB 커넥션 안에서 만들어지는 트랜잭션을 로컬 트랜잭션(local transaction)이라고도 한다.

### UserService와 UserDao의 트랜잭션 문제
하나의 템플릿 메소드 안에서 DataSource의 getConnection() 메소드를 호출해서 Connection 오브젝트를 가져오고, 작업을 마치면 Connection을 확실하게 닫아주고 템플릿 메소드를 빠져나온다. 결국 템플릿 메소드 호출 한 번에 한 개의 DB 커넥션이 만들어지고 닫히는 일까지 일어나는 것이다. 결국 jdbcTemplate의 메소드를 사용하는 UserDao는 각 메소드마다 하나의 독립적인 트랜잭션으로 실행될 수 밖에 없다.

DAO를 분리해놓았을 경우는 DAO의 메소드를 호출할 때마다 하나의 새로운 트랜잭션이 만들어지는 구조가 될 수밖에 없다.

### 비즈니스 로직 내의 트랜잭션 경계설정
UserService와 UserDao를 그대로 둔 채로 트랜잭션을 적용하려면 트랜잭션의 경계설정 작업을 UserService 쪽으로 가져와야 한다. 결국 다음과 같은 구조로 만들어야 한다.

``` java
public void upgradeLevels() throw Exception {
    DB Connection 생성
    트랜잭션 시작
    try {
        DAO 메소드 호출
        트랜잭션 커밋
    }
    catch (Exception e) {
        트랜잭션 롤백
        throw e;
    }
    finally {
        DB Connection 종료
    }
}
```

여기서 생성된 Connection 오브젝트를 가지고 데이터를 액세스 하는 작업을 진행하는 코드는 DAO에 있어야 하기 때문에 DAO 메소드로 Connection을 다음과 같이 파라미터로 전달해줘야 한다.

``` java
public interface UserDao {
    public void add(Connection c, User user);
    ...
    public void update(Connection c, User user);
}
```

여기서 UserService의 upgradeLevels()는 UserDao의 update()를 직접 호출하지 않고 upgradeLevel() 메소드를 통해 호출한다. 결국 upgradeLevel()에도 Connection 오브젝트를 사용하도록 전달해야 한다.

``` java
protected void upgradeLevel(Connection c, User user) {
    user.upgradeLevel();
    userDao.update(c, user);
}
```

### UserService 트랜잭션 경계설정의 문제점
위와 같이 수정하면 트랜잭션 문제는 해결할 수 있지만 여러 가지 새로운 문제가 발생한다.
* DB 커넥션을 비록한 리소스의 깔끔한 처리를 가능하게 했던 JdbcTemplate을 더 이상 활용할 수 없다는 점
    * 결국 JDBC API를 직접 사용하는 초기 방식으로 돌아가서 try/catch/finally 블록이 UserService 내에 존재하게 된다.
* DAO의 메소드와 비즈니스 로직을 담고 있는 UserService의 메소드에 Connection 파라미터가 추가돼야 한다는 점
* Connection 파라미터가 인터페이스 메소드에 추가되면 UserDao는 더 이상 데이터 액세스 기술에 독립적일 수가 없다.
    * JPA나 하이버네이트로 변경하려고 하면 EntityManager나 Session 오브젝트를 전달받아야 하게 된다.
* 테스트 코드에도 영향을 미친다.
    * DB 커넥션은 전혀 신경 쓰지 않고 있다가 Connection 오브젝트를 일일이 만들어서 DAO를 호출하도록 모두 변경해야 한다.

### 트랜잭션 동기화
Connection 파라미터 문제를 해결하기 위해서는 스프링이 제안하는 방법인 트랜잭션 동기화(transaction synchronization)방식을 사용한다.

트랜잭션 동기화란 UserService에서 트랜잭션을 시작하기 위해 만든 Connection 오브젝트를 특별한 저장소에 보관해두고, 이후에 호출되는 DAO의 메소드에서는 저장된 Connection을 가져다가 사용하게 하는 것이다. 정확히는 JdbcTemplate이 트랜잭션 동기화 방식을 이용하도록 하는 것이다.

트랜잭션 동기화 저장소는 작업 스레드마다 독립적으로 Connection 오브젝트를 저장하고 관리하기 때문에 다중 사용자를 처리하는 서버의 멀티스레드 환경에서도 충돌이 날 염려는 없다.

### JdbcTemplate과 트랜잭션 동기화
DAO를 사용할 때 트랜잭션이 굳이 필요 없다면 바로 호출해서 사용해도 되고, DAO 외부에서 트랜잭션을 만들고 이를 관리할 필요가 있다면 미리 DB 커넥션을 생성한 다음 트랜잭션 동기화를 해주고 사용하면 된다. 동기화를 해주면 DAO에서 사용하는 JdbcTemplate은 자동으로 트랜잭션 안에서 동작할 것이다.

JDBC 코드의 try/catch/finally 작업 흐름 지원, SQLException의 예외 변환과 함께 JdbcTemplate이 제공해주는 세 가지 유용한 기능 중 하나다.

### 트랜잭션 서비스 추상화
깔끔하게 책임과 성격에 따라 데이터 액세스 부분과 비즈니스 로직을 분리했지만 문제가 있다.

### 기술과 환경에 종속되는 트랜잭션 경계설정 코드
업체별 DB 연결 방법은 DI를 적용한 덕분에 자유롭게 변경할 수 있지만, 하나의 트랜잭션 안에서 여러 개의 DB에 데이터를 넣는 작업을 해야 할 필요가 있을 수 있다. 한 개 이상의 DB로의 작업을 하나의 트랜잭션으로 만드는 건 JDBC의 Connection을 이용한 트랜잭션 방식인 로컬 트랜잭션으로는 불가능하다.
> 로컬 트랜잭션은 하나의 DB Connection에 종속되기 때문이다.

따라서 각 DB와 독립적으로 만들어지는 Connection을 통해서가 아니라, 별도의 트랜잭션 관리자를 통해 트랜잭션을 관리하는 글로벌 트랜잭션 방식을 사용해야 한다.

자바는 JDBC 외에 이런 글로벌 트랜잭션을 지원하는 트랜잭션 매니저를 지원하기 위한 API인 JTA(Java Transaction API)를 제공하고 있다. JTA를 이용한 방법으로 바꿔도 트랜잭션 경계설정을 위한 구조는 JDBC를 사용했을 때와 비슷하다.

문제는 JDBC 로컬 트랜잭션을 JTA를 이용하는 글로벌 트랜잭션으로 바꾸려면 UserService의 코드를 수정해야 한다는 점이다. UserService는 자신의 로직이 바뀌지 않았음에도 기술환경에 따라서 코드가 바뀌는 코드가 돼버리고 말았다. 또 하이버네이트를 이용해 UserDao를 구현해 사용하게 된다면 JDBC나 JTA의 코드와는 또 다르게 코드가 바뀌게 된다.

### 트랜잭션 API의 의존관계 문제와 해결책
UserService의 메소드 안에서 트랜잭션 경계설정 코드를 제거할 수는 없다. 특정 기술에 의존적인 Connection(Jdbc), UserTransaction(JTA), Session/Transaction(하이버네이트) API 등에 종속되지 않게 할 수 있는 방법은 있다.

트랜잭션 경계설정를 담당하는 코드가 일정한 패턴을 갖는 유사한 구조이기 때문에 추상화를 생각해볼 수 있다.

### 스프링의 트랜잭션 서비스 추상화
스프링은 트랜잭션 기술의 공통점을 담은 트랜잭션 추상화 기술을 제공하고 있다.

``` java
public void upgradeLevels() {
    PlatformTransactionManager transactionManager = new DataSourceTransactionManager(dataSource);

    TransactionStatus status = transactionManager.getTransaction(new DefaultTransactionDefinition());
    try {
        List<User> users = userDao.getAll();
        for (User user : users) {
            if (canUpgradeLevel(user)) {
                upgradeLevel(user);
            }
        }
        transactionManager.commit(status); // 트랜잭션 커밋
    } catch (RuntimeException e) {
        transactionManager.rollback(status) // 트랜잭션 롤백
        throw e;
    }
}
```

PlatformTransactionManager
* 트랜잭션 경계설정을 위한 추상 인터페이스
DataSourceTransactionManager
* JDBC의 로컬 트랜잭션을 이용하기 위한 구현체
* 사용할 DB의 dataSource를 생성자 파라미터로 넣으면 오브젝트를 만들어 준다.
transactionManager.getTransaction()
* 트랜잭션을 가져오는 요청만 하면 트랜잭션 매니저가 Connection을 생성하고 트랜잭션을 시작해준다.
DefaultTransactionDefinition
* 트랜잭션에 대한 속성을 담고 있다.

PlatformTransactionManager로 시작한 트랜잭션은 트랜잭션 동기화 저장소에 저장된다.

### 트랜잭션 기술 설정의 분리
UserService 코드를 JTA를 이용하는 글로벌 트랜잭션으로 변경하기 위해서는 PlatformTransactionManager 구현 클래스를 JTATransactionManager로 바꿔주기만 하면 된다.

JTATransactionManager는 주요 자바 서버에서 제공하는 JTA 정보를 JNDI를 통해 자동으로 인식하는 기능을 갖고 있다. 따라서 별 다른 설정 없이 트랜잭션 매니저/서비스와 연동해서 동작한다.

하이버네이트, JPA로 UserDao를 구현했다면 그에 해당하는 구현체를 사용하면 된다. getTransaction(), commit(), rollback() 메소드는 손댈 필요가 없지만, 어떤 트랜잭션 매니저 구현 클래스를 사용할지 Service 코드가 알고 있는 것은 DI 원칙에 위배된다.

구현체를 스스로 결정하지 말고 외부 컨테이너를 통해 외부에서 제공받게 하는 스프링 DI의 방식으로 바꾸자. 그럼 빈으로 등록하고 DI 방식으로 사용하게 해야 한다. 
> 어떤 클래스든 스프링의 빈으로 등록할 때 먼저 검토해야 할 것은 싱글톤으로 만들어져 여러 스레드에서 동시에 사용해도 괜찮은가 하는 점이다. 상태를 갖고 있고, 멀티스레드 환경에서 안전하지 않은 클래스를 빈으로 등록하면 심각한 문제가 발생한다.

userService에는 PlatformTransactionManager 인터페이스 타입의 인스턴스 변수를 선언하고, 수정자 메소드를 추가해서 DI가 가능하게 해준다.