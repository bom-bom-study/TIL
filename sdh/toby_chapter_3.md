# 템플릿

개방 폐쇄 원칙(OCP)은 코드에서 어떤 부분은 변경을 통해 그 기능이 다양해지고 확장하려는 성질이 있고, 어떤 부분은 고정되어 있고 변하지 않으려는 성질이 있음을 말해준다.

변화의 특성이 다른 부분을 구분해주고, 각각 다른 목적과 다른 이유에 의해 다른 시점에 독립적으로 변경될 수 있는 효율적인 구조를 만들어주는 것이 OCP이다.

템플릿이란 이렇게 바뀌는 성질이 다른 코드 중에서 변경이 거의 일어나지 않으며 일정한 패턴으로 유지되는 특성을 가진 부분을 자유롭게 변경되는 성질을 가진 부분으로부터 독립시켜서 효과적으로 활용할 수 있도록 하는 방법이다.

### DB 커넥션
일반적으로 서버에서는 제한된 개수의 DB 커넥션을 만들어서 재사용 가능한 풀로 관리한다. DB 풀은 매번 getConnection()으로 가져간 커넥션을 명시적으로 close()해서 돌려줘야지만 다시 풀에 넣었다가 다음에 재사용할 수 있다. 그런데 오류가 날 때마다 반환하지 못한 커넥션이 계속 쌓이면 커넥션 풀에 여유가 없어지고 서버가 중단된다.

### 리소스 반환과 close()
Connection이나 PreparedStatement에는 close() 메소드가 있다. 이름으로 보면 닫는다는 의미지만 만들어진 걸 종료하는 것이 아니라 리소스를 반환한다는 의미로 이해하는게 좋다. Connection과 PreparedStatement는 보통 풀(pool) 방식으로 운영된다. 미리 정해진 풀 안에 제한된 수의 리소스를 만들어두고 사용이 필요할 때 이를 할당하고, 반환하는 방식으로 운영된다.

요청이 매우 많은 서버 환경에서는 매번 새로운 리소스를 생성하는 대신 풀에 만들어둔 리소스를 돌려가며 사용하는 편이 훨씬 유리하다. 대신 사용한 리소스는 빠르게 반환해야 한다. 그렇지 않으면 풀에 있는 리소스가 고갈되고 문제가 발생한다.

### JDBC try/catch/finally 코드의 문제점
JDBC DAO에는 복잡한 try/catch/finally 블록이 2중으로 중첩되어 나오고, 모든 메소드마다 반복된다. 이런 코드를 작성할 때 가장 쉬운 방법은 Copy&Paste이다. 하지만, finally 블록에서 close()를 빼먹는 것과 같은 실수를 해도 컴파일 에러가 나지 않기 때문에 테스트도 통과되고 별 문제가 없어보인다. 그러다가 커넥션이 하나씩 반환되지 않고 쌓여가다간 언젠간 리소스가 꽉 차서 서비스가 중단되는 상황이 발생한다.

이런 코드를 효과적으로 다룰 수 있는 방법의 핵심은 변하지 않는, 그러나 많은 곳에서 중복되는 코드와 로직에 따라 자꾸 확장되고 자주 변하는 코드를 잘 분리해내는 작업이다.

```
Connection c = null;
PreparedStatement ps = null;

try {
    c = dataSource.getConnection();

    ps = c.prepareStatement("delte from users"); // 변하는 부분

    ps.executeUpdate();
} catch (SQLException e) {
    throw e;
} finally {
    if (ps != null) { try { ps.close(); } catch (SQLException e) {} )
    if (c != null) { try { c.close(); } catch (SQLException e) {} )
}
```

### 메소드 추출
먼저 생각해볼 수 있는 방법은 변하는 부분을 메소드로 빼는 것이다. 하지만 메소드 추출 리팩토링을 적용하는 경우에는 분리시킨 메소드를 다른 곳에서 재사용 할 수 있어야 하는데, 이건 반대로 분리시키고 남은 메소드가 재사용이 필요한 부분이고, 분리된 메소드는 새롭게 만들어서 확장돼야 하는 부분이기 때문에 반대로 됐다.

### 템플릿 메소드 패턴의 적용
템플릿 메소드 패턴은 상속을 통해 기능을 확장해서 사용하는 부분이다.
추출해서 별도의 메소드로 독립시킨 makeStatement() 메소드를 추상 메소드 선언으로 변경한다. 물론 Dao도 추상 클래스가 돼야 한다.

```
abstract protected PreparedStatement makeStatement(Connection c) throws SQLException;
```

그리고 상속받는 서브클래스에서 이 메소드를 구현한다. 고정된 JDBC try/catch/finally 블록을 가진 슈퍼클래스 메소드와 필요에 따라 상속을 통해 PreparedStatement를 바꿔서 사용할 수 있게 만드는 서브 클래스로 깔끔하게 분리할 수 있다.

이제 Dao 클래스의 기능을 확장하고 싶을 떄마다 상속을 통해 확장할 수 있고 상위 Dao클래스는 불필요한 변화가 생기지 않도록 할 수 있으니 OCP을 잘 지키는 구조를 만들어 낼 수 있는 것 같다.

하지만 템플릿 메소드 패턴은 Dao 로직마다 상속을 통해 새로운 클래스를 만들어야 한다. 또 확장구조가 이미 클래스를 설계하는 시점에서 고정되어 버리고, 컴파일 시점에 이미 그 관계가 결정되어 있어서 유연성이 떨어져 버린다.

### 전략 패턴의 적용
OCP를 잘 지키는 구조이면서 템플릿 메소드 패턴보다 유연하고 확장성이 뛰어난 것이, 오브젝트를 아예 둘로 분리하고 클래스 레벨에서는 인터페이스를 통해서만 의존하도록 만드는 전략 패턴이다. 전략 패턴은 OCP 관점에서 보면 확장에 해당하는 변하는 부분을 별도의 클래스로 만들어 추상화된 인터페이스를 통해 위임하는 방식이다. deleteAll()은 JDBC를 이용해 DB를 업데이트하는 작업이라는 변하지 않는 맥락(컨텍스트)를 갖는다. deleteAll()의 컨텍스트를 정리해보면 다음과 같다.
* DB 커넥션 가져오기
* PreparedStatement를 만들어줄 외부 기능 호출하기
* 전달받은 PreparedStatement 실행하기
* 예외가 발생하면 이를 다시 메소드 밖으로 던지기
* 모든 경우에 만들어진 PreparedStatement와 Connection을 적절히 닫아주기

두 번째 작업에서 사용하는 PreparedStatement를 만들어주는 외부 기능이 바로 전략 패턴에서 말하는 전략이라고 볼 수 있다. PreparedStatement를 생성하는 전략을 호출할 때는 이 컨텍스트 내에서 만들어둔 DB 커넥션을 전달해야 한다. 커넥션이 없으면 PreparedStatement도 만들 수가 없을 것이다. 이 내용을 인터페이스로 정의하면 다음과 같다

```
public interface StatementStrategy {
    PreparedStatement makePreparedStatement(Connection c) throws SQLException;
}
```

인터페이스를 상속해서 실제 전략, 즉 바뀌는 부분인 PreparedStatement를 생성하는 클래스인 deleteAllStatement 클래스를 만들면 다음과 같다.
```
public class DeleteAllStatement implements StatementStrategy {
    public PreparedStatement makePreparedStatement(Connection c) {
        PreparedStatement ps = c.prepareStatement("delete from users");
        return ps;
    }
}
```

이제 확장해서 만든 DeleteAllStatement 클래스를 사용해서 UserDao의 deleteAll() 메소드에서 다음과 같이 사용하면 그럭저럭 전략패턴을 사용했다고 볼 수 있다.

```
public void deleteAll() throws SQLException {
    ...
    try {
        c = dataSource.getConnection();

        StatementStrategy strategy = new DeleteAllStatement();
        ps = Strategy.makePreparedStatement(c);

        ps.executeUpdate();
    } catch (SQLException e) {
    ...
}
```
하지만 전략 패턴은 필요에 따라 컨텍스트는 그대로 유지되면서(OCP의 폐쇄 원칙) 전략을 바꿔 쓸수 있다(OCP의 개방 원칙)는 것인데, 위와 같이 컨텍스트 안에서 이미 구체적인 전략 클래스인 deleteAllStatement를 사용하도록 고정되어 특정 구현 구현 클래스를 알고 있다는 건, 전략패턴에도 OCP에도 잘 들어 맞는다고 볼 수 없다.

### DI 적용을 위한 클라이언트/컨텍스트 분리
전략 패턴에서 어떤 전략을 사용하게 할 것인가는 Context를 사용하는 앞단의 Client가 결정하는 게 일반적이다. 컨텍스트에 해당하는 JDBC try/catch/finally 코드를 클라이언트 코드인 StratementStrategy를 만드는 부분에서 독립시켜야 한다는 것이다. 현재 deleteAll() 메소드에서 다음 코드는 클라이언트에 들어가야 할 코드다. deleteAll()의 나머지 코드는 컨텍스트 코드이므로 분리해야 한다.
```
StatementStrategy strategy = new DeleteAllStatement();
```

컨텍스트에 해당하는 부분을 별도의 메소드로 독립시키면, 클라이언트는 DeleteAllStatement 오브젝트와 같은 전략 클래스의 오브젝트를 컨텍스트의 메소드를 호출하며 전달해야 한다. 이를 위해 전략 인터페이스인 StatementStrategy를 컨텍스트 메소드 파라미터로 지정할 필요가 있다.

```
public void jdbcContextWithStatementStrategy(StatementStrategy stmt) {
    Connection c = null;
    PreparedStatement ps = null;

    try {
        c = dataSource.getConnection();    

        ps = stmt.makePreparedStatement(c);

        ps.executeUpdate();
    } catch (SQLException e) {
        throw e;
    } finally {
        ...
    }
}
```

컨텍스트를 별도의 메소드로 분리했으니 deleteAll() 메소드가 클라이언트가 된다. deleteAll()은 전략 오브젝트를 만들고 컨텍스트를 호출하는 책임을 지고 있다. 사용할 전략 클래스는 DeleteAllStatement이므로 이 클래스의 오브젝트를 생성하고, 컨텍스트로 분리한 jdbcContextWithStatementStrategy() 메소드를 호출해주면 된다.

```
public void deleteAll() throw SQLException {
    StatementStrategy st = new DeleteAllStatement(); // 선정한 전략 클래스의 오브젝트를 생성
    jdbcContextWithStatementStrategy(st); // 컨텍스트 호출, 전략 오브젝트 전달
}
```

### 마이크로 DI
DI의 가장 중요한 개념은 제 3자의 도움을 통해 두 오브젝트 사이의 유연한 관계가 설정되도록 만든다는 것이다. 이 개념만 따른다면 DI를 이루는 오브젝트와 구성요소의 구조나 관계는 다양하게 만들 수 있다.

일반적으로 DI는 의존관계에 있는 두 개의 오브젝트와 이 관계를 다이내믹하게 설정해주는 오브젝트 팩토리(DI 컨테이너), 그리고 이를 사용하는 클라이언트라는 4개의 오브젝트 사이에서 일어난다. 하지만 때로는 원시적인 전략 패턴 구조를 따라 클라이언트가 오브젝트 팩토리의 책임을 함께 지고 있을 수도 있다. 또는, 클라이언트와 전략이 결합되거나 모두 하나의 클래스 안에 담길 수도 있다. 이런 경우도 DI가 이뤄지고 있기 때문에 DI의 장점을 단순화해서 IoC 컨테이너의 도움 없이 코드 내에서 적용한 경우를 마이크로 DI 또는 수동 DI라고 부를 수도 있다.

### 익명 내부 클래스
익명 내부 클래스(anonymous inner class)는 이름을 갖지 않는 클래스다. 클래스 선언과 오브젝트 생성이 결합된 형태로 만들어지며, 상속할 클래스나 구현할 인터페이스를 생성자 대신 사용해서 만든다. 클래스를 재사용할 필요가 없고, 구현한 인터페이스 타입으로만 사용할 경우에 유용하다.
```
new 인터페이스() { 클래스 본문 };
```

### 컨텍스트와 DI
jdbcContextWithStatementStrategy() 메소드는 여러 DAO에서 사용할 수 있는 컨텍스트이다. 그렇기 때문에 JdbcContext라는 클래스로 따로 분리하고, DataSource타입 빈을 DI받으면 된다.

그리고 UserDao에서는 JdbcContext 빈을 DI받도록 하면 된다. JdbcContext를 분리해서 DI받도록 했지만 인터페이스를 의존하는게 아니고 구체 클래스를 의존하게 되어 있다.

스프링 DI의 기본 의도에 맞게 JdbcContext의 메소드를 인터페이스로 뽑아내어 정의해두고 사용해도 상관은 없지만 꼭 그럴 필요는 없다.

### 스프링 빈으로 DI
인터페이스를 사용해서 클래스를 자유롭게 변경할 수 있게 하지는 않았지만, JdbcContext를 UserDao와 DI구조로 만들어야 할 이유는 다음과 같다.
* JdbcContext가 스프링 컨테이너의 싱글톤 레지스트리에서 관리되는 싱글톤 빈이 되기 때문이다.
    * JdbcContext는 그 자체로 변경되는 상태정보를 갖고 있지 않고, DataSource도 읽기 전용이기 때문에 싱글톤이 되는데 아무런 문제가 없다.
    * JDBC 컨텍스트 메소드를 제공해주는 일종의 서비스 오브젝트이기 때문에, 싱글톤으로 등록되서 여러 오브젝트에서 공유해 사용되는 것이 이상적이다.
* JdbcContext가 DI를 통해 다른 빈에 의존하고 있기 때문이다.
    * DI를 위해서는 주입되는 오브젝트와 주입받는 오브젝트 양쪽 모두 스프링 빈으로 등록돼야 한다.
    * 스프링이 생성하고 관리하는 IoC대상이어야 DI에 참여할 수 있기 떄문이다.

UserDao가 JDBC 방식 대신 JPA나 하이버네이트 같은 ORM으로 바꾼다면 JdbcContext를 통째로 바꿔야 겠지만, 다른 구현으로 대체해서 사용할 이유가 없다면 이런 경우는 굳이 인터페이스를 두지 말고 강력한 결합을 가진 관계를 허용하면서 스프링 빈으로 등록해서 DI 되도록 만들어도 좋다.

단, 이런 클래스를 바로 사용하는 코드 구성을 DI에 적용하는 것은 가장 마지막단계에서 고려해볼 사항이다. 인터페이스를 만들기 귀찮아서 그냥 클래스를 사용하는 건 잘못된 생각이다. 굳이 원한다면 JdbcContext에 인터페이스를 두고 UserDao에서 인터페이스를 사용하도록 만들어도 문제 될 것은 없다.

### 코드를 이용하는 수동 DI
JdbcContext를 스프링의 빈으로 등록해서 UserDao에 DI하는 대신 UserDao 내부에서 직접 DI를 적용하는 방법도 있다.

이 방법을 쓰려면 JdbcContext를 스프링의 빈으로 등록해서 싱글톤으로 만드는 것은 포기해야 한다. 스프링 빈으로 등록하지 않았으므로 jdbcContext의 생성과 초기화는 UserDao가 갖는 것이 적당하다.

JdbcContext는 DataSource의 타입의 빈을 다이내믹하게 주입받는 것에 대해서는 UserDao가 대신 DataSource빈을 주입받아서 JdbcContext에 직접적으로 넣어주면 된다.

```
public class UserDao {
    ...
    private JdbcContext jdbcContext;

    public void setDataSource(DataSource dataSource) {
        this.jdbcContext = new JdbcContext(); // JdbcContext 생성(IoC)
        this.jdbcContext.setDataSource(dataSource); // 의존 오브젝트 주입(DI)
    }
}
```

이 방법은 굳이 인터페이스를 두지 않아도 될 만큼 긴밀한 관계를 갖는 DAO클래스와 JdbcContext를 어색하게 따로 빈으로 분리하지 않고 내부에서 직접 만들어 사용하면서도 다른 오브젝트에 대한 DI를 적용할 수 있다는 점이다. 이렇게 수정자 메소드에서 다르 오브젝트를 초기화하고 코드를 이용해 DI하는 것은 스프링에서도 종종 사용되는 기법이다.

인터페이스를 사용하지 않고 DI를 받는 두가지 방법의 장단점
* 스프링의 DI를 이용해 빈으로 등록해서 사용하는 방법
    * 오브젝트 사이의 실제 의존관계가 설정파일에 명확하게 드러난다는 장점
    * DI의 근본적인 원칙에 부합하지 않는 구체적인 클래스와의 관계가 설정에 직접 노출된다는 단점
* 코드를 이용해 수동으로 DI를 하는 방법
    * JdbcContext가 UserDao의 내부에서 만들어지고 사용되면서 그 관계를 외부에는 드러내지 않는다는 장점
    * 필요에 따라 내부에서 은밀히 DI를 수행하고 그 전략을 외부로 감출수 있는 장점
    * JdbcContext를 여러 오브젝트가 사용하더라도 싱글톤으로 만들 수 없는 단점
    * DI 작업을 위한 부가적인 코드가 필요하는 단점

상황에 따라 적절하다고 판단되는 방법을 선택해서 사용하면 된다. 다만 그렇게 선택한 이유에 대해서는 분명한 이유와 근거가 있어야 한다. 분명하게 설명한 자신이 없다면 차라리 인터페이스를 만들어서 평범한 DI 구조로 만드는게 나을 수도 있다.

### 템플릿과 콜백
전략 패턴은 복잡하지만 바뀌지 않는 일정한 패턴을 갖는 작업 흐름이 존재하고 그중 일부분만 자주 바꿔서 사용해야 하는 경우에 적합한 구조다.

전략 패턴의 기본 구조에 익명 내부 클래스를 활용한 방식을 스프링에서는 템플릿/콜백 패턴이라고 부른다. 전략 패턴의 컨텍스트를 템플릿이라 부르고, 익명 내부 클래스로 만들어지는 오브젝트를 콜백이라고 부른다.

> 템플릿(template)은 어떤 목적을 위해 미리 만들어둔 모양이 있는 틀을 가르킨다. JSP는 HTML이라는 고정된 부분에 EL과 스크립릿이라는 변하는 부분을 넣은 일종의 템플릿 파일이다. 템플릿 메소드 패턴은 고정된 틀의 로직을 가진 템플릿 메소드를 슈퍼클래스에 두고, 바뀌는 부분을 서브클래스의 메소드에 두는 구조로 이뤄진다.

> 콜백
콜백(callback)은 실행되는 것을 목적으로 다른 오브젝트의 메소드에 전달되는 오브젝트를 말한다. 파라미터로 전달되지만 값을 참조하기 위한 것이 아니라 특정 로직을 담은 메소드를 실행시키기 위해 사용한다. 자바에선 메소드 자체를 파라미터로 전달할 방법은 없기 때문에 메소드가 담긴 오브젝트를 전달해야 한다. 그래서 functional object라고도 한다.

### 템플릿/콜백의 특징
여러 개의 메소드를 가진 일반적인 인터페이스를 사용할 수 있는 전략 패턴의 전략과 달리 템플릿/콜백 패턴의 콜백은 보통 단일 메소드 인터페이스를 사용한다. 템플릿의 작업 흐름 중 특정 기능을 위해 한 번 호출되는 경우가 일반적이기 때문이다. 

콜백 인터페이스의 메소드에는 보통 파라미터가 있다. 이 파라미터는 템플릿의 작업 흐름 중에 만들어지는 컨텍스트 정보를 전달 받을 때 사용된다. jdbcContext에서는 Connection 오브젝트를 넘겨주고 PreparedStatement를 돌려받는다.

<img src="img\template_callback_workflow.png">

조금 복잡해 보이지만 DI 방식의 전략 패턴 구조라고 보면 간단하다. 클라이언트가 템플릿 메소드를 호출하면서 콜백 오브젝트를 전달하는 것은 메소드 레벨에서 일어나는 DI다. 템플릿이 사용할 콜백 인터페이스를 구현한 오브젝트를 메소드를 통해 주입해주는 DI 작업이 클라이언트가 템플릿의 기능을 호출하는 것과 동시에 일어난다.

일반적인 DI라면 인스턴스 변수에 주입받아서 사용하지만, 템플릿/콜백 방식에는 매번 메소드 단위로 사용할 오브젝트를 새롭게 전달받는다는 것이 특징이다.

### 편리한 콜백의 재활용
템플릿/콜백 방식은 템플릿에 담긴 코드를 여기저기서 반복적으로 사용하는 원시적인 방법에 비해 많은 장점이 있다. 예를 들어 클라이언트인 DAO의 메소드는 간결해지고 최소한의 데이터 액세스 로직만 갖고 있게 된다. 그런데 템플릿/콜백 방식은 매번 익명 내부 클래스를 사용하기 때문에 상대적으로 작성하고 읽기가 조금 불편하다.

익명 내부 클래스로 만든 콜백 오브젝트의 구조를 다시 보면 다음과 같다.
```
public void deleteAll() throws SQLException {
    this.jdbcContext.workWithStatementStrategy(
        new StatementStrategy() { 
            public PreparedStatement makePreparedStatement(Connection c) throws SQLException {
                return c.preparedStatement("delete from user"); // 변하는 SQL 문장
            }
        }
    );
}
```

deleteAll() 메소드를 통틀어서바뀔 수 있는 것은 오직 "delete from users"라는 문자열 뿐이기 때문에 다음과 같이 별도의 메소드로 분리할 수 있다.

```
public void deleteAll() throws SQLException {
    executeSql("delete from users"); -> 변하는 SQL 문장
}

private void executeSql(final String query) throws SQLException {
    this.jdbcContext.workWithStatementStrategy(
        new StatementStrategy() { 
            public PreparedStatement makePreparedStatement(Connection c) throws SQLException {
                return c.preparedStatement(query); // 변하는 SQL 문장
            }
        }
    );
}
```

### 콜백과 템플릿의 결합
executeSql() 메소드는 UserDao만 사용하기는 아깝다. 재사용 가능한 콜백을 담고 있는 메소드라면 DAO가 공유할 수 있는 템플릿 클래스 안으로 옮겨도 된다. 템플릿은 JdbcContext 클래스가 아니라 workWithStatementStrategy() 메소드이므로 JdbcContext 클래스로 콜백 생성과 템플릿 호출이 담긴 executeSql() 메소드를 옮겨도 된다.

```
public class JdbcContext {
    ...
    public void executeSql(final String query) throws SQLException {
        workWithStatementStrategy(
            ...
        );
    }
}
```

UserDao의 메소드에서도 jdbcContext를 통해 executeSql() 메소드를 호출하도록 수정해야 한다.
```
public void deleteAll() throws SQLException {
    this.jdbcContext.executeSql("delete from users");
}
```

### 템플릿/콜백의 응용
스프링의 많은 API나 기능을 살펴보면 템플릿/콜백 패턴을 적용한 경우를 많이 발견할 수 있다.

DI는 순수한 스프링의 기술이 아니다. 기본적으로 객체지향의 장점을 잘 살려서 설계하고 구현하도록 도와주는 여러 가지 원칙과 패턴의 활용 결과일 뿐이다. 스프링은 단지 이를 편리하게 사용할 수 있도록 도와주는 컨테이너를 제공하고, 이런 패턴의 사용 방법을 지지해주는 것뿐이다. 템플릿/콜백 패턴도 DI와 객체지향 설계를 적극적으로 응용한 결과다. 스프링에는 다양한 자바 엔터프라이즈 기술에서 사용할 수 있도록 미리 만들어져 제공되는 수십 가지의 템플릿/콜백 클래스와 API가 있다.

고정된 작업 흐름을 갖고 있으며서 여기저기서 자주 반복되는 코드가 있다면, 중복되는 코드를 분리할 방법을 생각해보자. 중복된 코드는 먼저 메소드로 분리하는 간단한 시도를 해본다. 그 중 일부 작업을 필요에 따라 바꾸어 사용해야 한다면 인터페이스를 사이에 두고 분리해서 전략 패턴을 적용하고 DI로 의존관계를 관리하도록 만든다. 그런데 바뀌는 부분이 한 애플리케이션 안에서 동시에 여러 종류가 만들어질 수 있다면 템플릿/콜백 패턴을 적용하는 것을 고려해볼 수 있다. 가장 전형적인 템플릿/콜백 패턴의 후보는 try/catch/finally 블록에서 리소스를 가져와 작업하는 코드이다.

템플릿과 콜백을 찾아낼 때는, 변하는 코드의 경계를 찾고 그 경계를 사이에 두고 주고받는 일정한 정보가 있는지 확인을 하면 된다.

템플릿/콜백 패턴은 다양한 작업에 손쉽게 활용할 수 있다. 콜백이라는 이름이 의미하는 것처럼 다시 불려지는 기능을 만들어서 보내고 템플릿과 콜백, 클라이언트 사이에 정보를 주고받는 일이 처음에는 조금 복잡하게 느껴질지도 모른다. 하지만 코드의 특성이 바뀌는 경계를 잘 살피고 그것을 인터페이스를 사용해 분리한다는, 가장 기본적인 객체지향 원칙에만 충실하면 어렵지 않게 템플릿/콜백 패턴을 만들어 활용할 수 있다.

### 스프링의 JdbcTemplate
스프링은 JDBC를 이용하는 DAO에서 사용할 수 있도록 준비된 다양한 템플릿과 콜백을 제공한다. 거의 모든 종류의 JDBC 코드에 사용 가능한 템플릿과 콜백을 제공할 뿐만 아니라, 자주 사용되는 패턴을 가진 콜백은 다시 템플릿에 결합시켜서 간단한 메소드 호출만으로 사용이 가능하도록 만들어져 있다.

### update()
JdbcTemplate의 콜백은 PreparedStatementCreator 인터페이스의 createPreparedStatement() 메소드다. 템플릿으로부터 Connection을 제공받아서 PreparedStatement를 만들어 돌려준다는 면에서 구조는 동일하다.

```
public void deleteAll() {
    this.jdbcTemplate.update(
        new PreparedStatementCreator() {
            public PreparedStatement createPreparedStatement(Connection con)
                throws SQLException {
                return con.prepareStatement("delete from users");
            }
        }
    );
}
```

앞에서의 executeSql()은 SQL 문장만 전달하면 미리 준비된 콜백을 만들어서 템플릿을 호출하는 것까지 한번에 해줬는데 JdbcTemplate에도 기능이 비슷한 메소드가 존재한다. 메소드 이름은 동일하고 SQL 문장을 전달한다는 것만 다르다.
```
public void deleteAll() {
    this.jdbcTemplate.update("delete from users");
}
```
add()와 같이 파라미터를 바인딩해주는 기능을 가진 update() 메소드는 SQL과 함께 가변인자로 선언된 파라미터를 제공해주면 된다.
```
this.jdbcTemplate.update("insert into users(id, name, password) values(?, ?, ?)", user.get(), user.getName(), user.getPassword());
```

### queryForInt()
getCount()는 SQL쿼리를 실행하고 ResultSet을 통해 결과 값을 가져오는 코드다. 이런작업 흐름을 가진 코드에서 사용할 수 있는 템플릿은 PreparedStatementCreator콜백과 ResultSetExtractor 콜백을 파라미터로 받는 query()메소드다.

### queryForObject()
getCount()처럼 단순한 값이 아니라 복잡한 User같은 오브젝트를 만들 때 사용한다. 이를 위해, ResultSetExtractor 콜백 대신 RowMapper콜백을 사용한다. 

ResultSetExtractor와 RowMapper가 다른 점은 ResultSetExtractor은 ResultSet을 한 번 전달받아서 알아서 추출 작업을 모두 진행하고 최종 결과만 리턴해주면 되는 데 반해, RowMapper는 ResultSet의 로우 하나를 매핑하기 위해 여러번 호출될 수 있다는 점이다.
```
public User get(String id) {
    return this.jdbcTemplate.queryForObject("select * from users where id = ?",
        new Object[] {id}, // SQL에 바인딩 할 파라미터 값, 가변인자 대신 배열을 사용한다.
        new RowMapper<User>() {
            public User mapRow(ResultSet rs, int rowNum) throws SQLException {
                User user = new User();
                user.setId(rs.getString("id"));
                user.setName(rs.getString("name"));
                user.setPassword(rs.getString("password"));
                return user;
            }
        });
}
```
첫 번째 파라미터는 PreparedStatement를 만들기 위한 SQL이고, 두 번째는 SQL문에 바인딩할 값들이다. 가변인자를 사용하지 않은 이유는 뒤에 다른 파라미터가 있기 때문에 Object 타입의 배열을 사용해야 한다.

조회 결과가 없는 예외상황은 queryForObject()가 SQL을 실행해서 받은 로우의 개수가 하나가 아니라면 EmptyResultDataAccessException예외를 발생시킨다.