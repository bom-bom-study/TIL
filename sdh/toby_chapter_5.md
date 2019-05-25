# 서비스 추상화

자바에는 표준 스펙, 상용 제품, 오픈소스를 통틀어서 사용 방법과 형식은 다르지만 기능과 목적이 유사한 기술이 존재한다. 그에 따라 API를 사용하고 다른 스타일의 접근 방법을 따라야 하는 건 매우 피곤한 일이다. 스프링은 성격이 비슷한 여러 종류의 기술을 추상화하고 이를 일관된 방법으로 사용할 수 있도록 지원한다.

### 사용자 레벨 관리 기능 추가
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