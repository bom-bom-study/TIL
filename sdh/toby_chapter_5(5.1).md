# 서비스 추상화

자바에는 표준 스펙, 상용 제품, 오픈소스를 통틀어서 사용 방법과 형식은 다르지만 기능과 목적이 유사한 기술이 존재한다. 그에 따라 API를 사용하고 다른 스타일의 접근 방법을 따라야 하는 건 매우 피곤한 일이다. 스프링은 성격이 비슷한 여러 종류의 기술을 추상화하고 이를 일관된 방법으로 사용할 수 있도록 지원한다.

### 5.1 사용자 레벨 관리 기능 추가
지금까지의 UserDao는 User 오브젝트에 담겨 있는 사용자의 기초적인 CRUD 작업만 가능하다. 여기에 정기적으로 사용자의 활동 내역을 참고해서 레벨을 조정해주는 기능을 추가해주자.
* 사용자의 레벨은 BASIC, SILVER, GOLD 세 가지다.
* 처음에는 BASIC 레벨이 되며, 이후 활동에 따라 한 단계씩 업그레이드 된다.

### 필드 추가
DB에 varchar 타입으로 선언하고 문자열로 넣는 방법 대신 각 레벨을 코드화해서 숫자로 넣으면 DB 용량도 많이 차지하지 않고 가벼워서 좋다. 하지만 자바의 User에 추가할 프로퍼티 타입을 숫자로 하면 안전하지 않아서 위험할 수 있다.

```java
Class User {
    private static final int BASIC = 1;
    private static final int SILVER = 2;
    private static final int GOLD = 3;

    int level;

    public void setLevel(int level) {
        this.level = level;
    }
}
```

상수로 정의해서 깔끔하게 코드를 작성할 수 있고, DB에 저장될 때는 getLevel()이 돌려주는 숫자 값을 사용하면 된다.

문제는 level 타입이 int이기 때문에 1, 2, 3외에 값을 넣으면 기능은 문제없이 돌아가는 것처럼 보이겠지만 사실은 레벨이 엉뚱하게 바뀌는 심각한 버그가 만들어진다.

그래서 자바 5 이상에서 제공하는 Enum을 이용하는 게 안전하고 편리하다.
```java
public enum level {
    BAISC(1), SILVER(2), GOLD(3);

    private final int value;

    Level(int value) {
        this.value = value;
    }

    pubic int intValue() {
        return value;
    }

    public static Level valueOf(int value) {
        switch(value) {
            case 1: return BASIC;
            case 2: return SILVER;
            case 3: return GOLD;
            default: throw new AssertionError("Unknown value: " + value);
        }
    }
}
```

이렇게 만들어진 Level 이늄 내부에는 DB에 저장할 int 타입의 값을 갖고 있지만, 겉으로는 Level 타입의 오브젝트이기 때문에 안전하게 사용할 수 있다.

### User 필드 추가
이렇게 만든 Level 타입의 변수를 User 클래스에 추가하고, 사용자 레벨 관리 로직에 필요한 로그인 횟수와 추천수도 추가하자.
```java
public class User {
    ...
    Level level;
    int login;
    int recommend;

    // getter, setter
}
```

이제 DB의 USER 테이블에도 필드를 추가한다.
* Level - tinyint, Not Null
* Login - int, Not Null
* Recommend - int, Not Null

이렇게 User클래스에 변수와 DB에 필드를 추가하고나면, UserDaoTest를 수정해서 새로 추가된 변수에 대한 테스트도 성공하도록 만든다.

### upgradeLevels() 메소드
로직을 추가할 준비가 끝나면 사용자 레벨 관리 기능과 테스트를 만든다. 요구사항을 보고 구현해보면 다음과 같이 만들 수 있다.
```java
public void upgradeLevels() {
    List<User> users = userDao.getAll();
    for (User user : users) {
        Boolean changed = null;
        if (user.getLevel() == Level.BASIC && user.getLogin() >= 50) {
            user.setLevel(Level.SILVER);
            changed = true;
        }
        else if (user.getLevel() == LEVEL.SILVER && user.getRecommend() >= 30 {
            user.setLevel(Level.GOLD);
            changed = true;
        }
        else if (user.getLevel() == Level.GOLD) {
            changed = false;
        }
        else {
            changed = false;
        }

        if (chagned) {
            userDao.update(user);
        }
    }
}
```

모든 사용자를 가져와 한명씩 레벨 변경 작업을 수행한다.

### UserService.add()
사용자 관리 비즈니스 로직 중 처음 가입하는 사용자는 기본적으로 BASIC 레벨이어야 한다는 구현을 해야 한다. 이 로직은 UserDaoJdbc의 add() 메소드에서는 관심사가 다르기 때문에 적합하지 않다. User 클래스에서 처음부터 BASIC 레벨로 초기화하는 것은 처음 가입할 때를 제외하면 무의미한 정보이기 때문에 문제가 있어 보인다.

그렇다면 사용자 관리에 대한 비즈니스 로직을 담고 있는 UserService에 이 로직을 넣는게 맞는 것 같다.
```java
public void add(User user) {
    if (user.getLevel() == null)
        user.setLevel(Level.BASIC);
    userDao.add(user);
}
```

레벨이 이미 설정됐던 것은 그대로 유지되어 있어야 하고, 레벨이 없던 것은 디폴트인 BASIC으로 설정됐는지 확인해준다.

### 코드 개선
비즈니스 로직의 구현을 모두 마치고, 테스트도 만들어서 꼼꼼하게 점검해서 넘어갈 수도 있지만, 깔끔한 코드를 위해서 만들어진 코드를 다시 한번 검토해보자.
* 코드에 중복된 부분은 없는가?
* 코드가 무엇을 하는 것인지 이해하기 불편하지 않은가?
* 코드가 자신이 있어야 할 자리에 있는가?
* 앞으로 변경이 일어난다면 어떤 것이 있을 수 있고, 그 변화에 쉽게 대응할 수 있게 작성되어 있는가?

### upgradeLevels() 메소드 코드의 문제점
* for 루프 속에 들어 있는 if/else if/else 블록들이 읽기 불편하다.
* 레벨의 변화 단계와 업그레이드 조건, 조건이 충족됐을 떄 해야할 작업이 한데 섞여있어 이해하기 어렵다.
* 만약 새로운 레벨이 추가된다면 Level 이늄도 수정하고 if 조건식과 블록을 추가해 줘야 한다.

### upgradeLevels() 리팩토링
월래는 자주 변경될 가능성이 있는 구체적인 내용이 추상적인 로직의 흐름과 함께 섞여 있다.

추상적인 레벨에서 로직을 작성해보면 다음과 같다.
```java
public void upgradeLevels() {
    List<User> users = userDao.getAll();
    for (User user : users) {
        if (canUpgradeLvel(user)) {
            upgradeLevel(user);
        }
    }
}
```

```java
private boolean canUpgradeLevel(User user) {
    Level currentLevel = user.getLevel();
    switch(currentLevel) {
        case BASIC: return (user.getLogin >= 50);
        case SILVER: return (user.getRecommend() >= 30);
        case GOLD: return false;
        default: throw new IllegalArgumentException("Unknown Level: " + currentLevel);
    }
}
```

업그레이드가 가능한지 확인하는 방법은 User 오브젝트에서 레벨을 가져와서, switch문으로 구분하고, 각 레벨에 대한 업그레이드 조건을 만족하는지 확인해주면 된다.

업그레이드 조건을 만족했을 경우 구체적으로 무엇을 할 것인가를 담고 있는 코드는 다음과 같다.

```java
private void upgradeLevel(User user) {
    if (user.getLevel() == Level.BASIC)
        user.setLevel(Level.SILVER);
    else if (user.getLevel() == Level.SILVER)
        user.setLevel(Level.GOLD);
    
    userDaol.update(user);
}
```

그런데 다음 단게가 무엇인가 하는 로직과 그때 사용자 오브젝트의 level 필드를 변경해준다는 로직이 함께 있는데다, 너무 노골적으로 드러나 있고, 예외상황에 대한 처리가 없다.

잘못해서 GOLD 레벨인 사용자를 업그레이드를 할려고 하면 아무것도 처리하지 않고 DAO의 업데이트 메소드만 실행될 것이다. 레벨이 늘어나면 if 문이 점점 길어질 것이고, 레벨 변경 시 사용자 오브젝트에서 level 필드 외의 값도 변경해야 한다면 if 조건 뒤에 붙는 내용도 점점 길어질 것이다.

이것도 더 분리해보면 레벨의 순서와 다음 단계 레벨이 무엇인지를 결정하는 일은 Level에게 맡길 수 있다.

```java
public enum Level {
    GOLD(3, null), SILVER(2, GOLD), BASIC(1, SILVER);
    
    private final int value;
    private final Level next; // 다음 단계의 레벨 정보를 스스로 갖고 있도록 한다.

    Level(int value, Level next) {
        this.value = value;
        this.next = next;
    }

    ...

    public Level nextLevel() {
        return this.next;
    }

    ...
}
```

이렇게 하면 다음 레벨이 무엇인지 nextLevel() 메소드를 호출해서 알 수 있기 떄문에 레벨의 업그레이드 순서는 비즈니스 로직의 if 문에서가 아니라 Level 이늄 안에서 관리할 수 있다. 

이번엔 사용자 정보가 바뀌는 부분을 UserService 메소드에서 User로 옮겨보자.

User의 내부 정보가 변경되는 것은 UserService보다는 User가 스스로 다루는 게 적절하다. User는 사용자 정보를 담고 있는 단순한 자바빈이긴 하지만 User도 엄연히 자바 오브젝트이고 내부 정보를 다루는 기능이 있을 수 있다.

```java
public void upgradeLevel() {
    Level nextLevel = this.level.nextLevel();
    if (nextLevel == null) {
        throw new IllegalStateException(this.level + "은 업그레이드가 불가능합니다");
    }
    else {
        this.level = nextLevel;
    }
}
```

User에 업그레이드 작업을 담당하는 메소드를 두고 사용할 경우, 업그레이드 시 기타 정보도 변경이 필요해지면 User의 upgradeLevel() 메소드에 추가를 하면 된다.

이렇게 하면 UserService의 upgradeLevel()는 간결해진다.

```java
private void upgradeLevel(User user) {
    user.upgradeLevel();
    userDao.update(user);
}
```

> 객체지향적인 코드는 다른 오브젝트의 데이터를 가져와서 작업하는 대신 데이터를 갖고 있는 다른 오브젝트에게 작업을 해달라고 요청한다. 오브젝트에게 데이터를 요구하지 말고 작업을 요청하라는 것이 객체지향 프로그래밍의 가장 기본이 되는 원리이기도 하다.