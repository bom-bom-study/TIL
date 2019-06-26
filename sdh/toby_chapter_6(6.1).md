# 스프링 AOP

스프링에 적용된 가장 인기 있는 AOP의 적용 대상은 바로 선언적 트랜잭션 기능이다.

### 6.1 트랜잭션 코드의 분리
* 서비스 추상화 기법을 적용해 트랜잭션 기술에 독립적으로 만들어줬지만, 트랜잭션 경계설정을 위해 넣은 코드 때문에 찜찜하다.
* 비즈니스 로직이 주인이어야 할 메소드 안에 트랜잭션 코드가 더 많은 자리를 차지하고 있다.

### 메소드 분리
``` java
public void upgradeLevels() throws Exception {
    TransactionStatus status = this.transactionManager.getTransaction(new DefaultTransactionDefinition());
    try {

        List<User> users = userDao.getAll();
        for (User user : users) {
            if (canUpgradeLevel(user)) {
                upgradeLevel(user);
            }
        }

        this.transactionManager.commit(status);
    } catch (Exception e) {
        this.transactionManager.rollback(status);
        throw e;
    }
}
```

* 자세히 살펴보면 두 가지 종류의 코드가 구분되어 있음을 알 수 있다.
* 트랜잭션 경계설정의 코드와 비즈니스 로직 코드 간에 서로 주고받는 정보가 없다.
* 트랜잭션 정보는 트랜잭션 동기화 방법을 통해 DAO가 알아서 활용한다.
* 코드 성격이 다를 뿐 아니라 서로 주고받는 것도 없는, 완벽하게 독립적인 코드다.

비즈니스 로직을 담당하는 코드를 메소드로 추출해서 독립시킬 수 있다.
* 코드는 깔끔하게 분리 되서 보기 좋다.
* 하지만 트랜잭션을 담당하는 기술적인 코드가 UserService 안에 자리 잡고 있다.
* UserService에서는 보이지 않게 트랜잭션 코드를 클래스 밖으로 뽑아내면 된다.

### DI 적용을 이용한 트랜잭션 분리
* 구체적인 구현 클래스를 직접 참조하는 경우의 전형적인 단점
    * 트랜잭션 코드를 UserService밖으로 빼버리면 UserService를 직접 사용하는 클라 코드에선 트랜잭션 기능이 빠진 UserService를 사용하게 될 것이다.
* 직접 사용이 문제가 되면 간접적으로 사용하면 된다.
    * DI의 기본 아이디어는 실제 사용할 오브젝트의 클래스 정체는 감춘 채 인터페이스를 통해 간적접으로 접근하는 것이다.
    * 덕분에 구현 클래스는 얼마든지 외부에서 변경할 수 있다.
* 한 번에 두개의 UserService 인터페이스 구현 클래스를 동시에 이용하면 된다.

### UserService 인터페이스 도입
* 클라이언트가 사용할 로직을 담은 핵심 메소드만 UserService 인터페이스로 만든 후 구현하도록 만든다.
    * add(User user);
    * upgradeLevels();
* UserServiceImpl은 기존 내용을 유지하고 트랜잭션과 관련된 코드는 모두 제거한다.

### 분리된 트랜잭션 기능
* 트랜잭션 처리를 담은 UserServiceTx를 만든다.
    * 그리고 같은 인터페이스를 구현한 다른 오브젝트에게 고스란히 작업을 위임하게 만들면 된다.
    * 적어도 비즈니스 로직에 대해서는 UserServiceTx가 아무런 관여도 하지 않는다.

``` java
public class UserServiceTx implements UserService {
    UserService userService;

    ...

    public void add(User user) {
        userService.add(user);
    }

    ...
}
```

* 비즈니스 로직을 담은 UserService 오브젝트를 DI 받을 수 있도록 만든다.
* 경계설정이라는 부가작업을 추가하면 다음과 같다.
``` java
public class UserServiceTx implements UserService {
    UserService userService;
    PlatformTransactionManager transactionManager;

    ...

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

### 트랜잭션 적용을 위한 DI 설정
클라이언트가 UserService라는 인터페이스를 통해 사용자 관리 로직을 이용하려고 할 때 UserServiceTx가 먼저 작업을 진행해주고, 그 이후에 UserServiceImpl가 작업을 진행한다.

### 트랜잭션 경계설정 코드 분리의 장점
* 비즈니스 로직을 담당하고 있는 UserServiceImpl의 코드를 작성할 때는 트랜잭션과 같은 기술적인 내용에 전혀 신경 쓰지 않아도 된다.
    * 트랜잭션의 적용이 필요한지도 신경 쓰지 않아도 된다.
    * DI를 이용해 UserServiceTx와 같은 트랜잭션 기능을 가진 오브젝트가 먼저 실행되도록 만들기만 하면 된다.
* 비즈니스 로직에 대한 테스트를 손쉽게 만들어낼 수 있다.