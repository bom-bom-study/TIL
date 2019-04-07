## 3장 템플릿
### 3.1 다시 보는 초난감 DAO
---
#### 3.1.1 예외처리 기능을 갖춘 DAO
1. JDBC 수정 기능의 예외처리 코드  
ex) deleteAll()
``` java
public void deleteAll() throws SQLException {
    Connection c = dataSource.getConnection();

    PreparedSatement ps = c.preparedStatement("delete from users");
    ps.executeUpdate();

    ps.close();
    c.close();
}
```
메소드에서 Connection과 PreparedStatement라는 두 개의 공유 리소스를 가져와서 사용하는데, PreparedStatement 처리 중에 예외가 발생하면 메소드 실행을 끝내지 못하고 메소드를 빠져나가게 된다. 이 때 close()가 실행되지 않아 제대로 리소스가 반환되지 않을 수 있다는 문제가 발생한다. 리소스가 반환되지 못하고 Connection이 계속 쌓이면 커넥션 풀에 여유가 없어지고 리소스가 모자라게 된다. 따라서 어떤 상황에서도 가져온 리소스를 반환하도록 try/catch/finally 구문을 사용할 것을 권장하고 있다. 
try/catch를 사용하면 어느 시점에 예외가 발생하는가에 따라 Connection과 Preparedstatement 중 어떤 close()를 호출해야할지 달라지기 때문에 finally에서 반드시 c와 ps가 null이 아닌지 확인한 후 close()를 호출해야 한다.  

``` java
public void deleteAll() throws SQLException {
    Connection c = null;
    PreparedSatement ps = null;

    try {
        c = dataSource.getConnection();
        ps = c.preparedStatement("delete from users");
        ps.executeUpdate();
    } catch(SQLException e) {
        throw e;
    } finally {
        if(ps != null) {
            try {
                ps.close();
            } catch (SQLException e){}
        }
        if(c != null) {
            try {
                c.close();
            } catch (SQLException e){}
        }
    }
}
```

2. JDBC 조회 기능의 예외처리  
조회를 위한 JDBC코드에는 Connection, PreparedStatement, ResultSet이 필요하다. ResultSet도 반환해야 하는 리소스이기 때문에 예외상황에서도 ResultSet의 close()가 호출되도록 해야 한다.

``` java
public void deleteAll() throws SQLException {
    Connection c = null;
    PreparedSatement ps = null;
    ResultSet rs = null;

    try {
        c = dataSource.getConnection();
        ps = c.preparedStatement("select count(*) from users");
        rs = ps.executeQuery();
        rs.next();

        return rs.getInt(1);
    } catch(SQLException e) {
        throw e;
    } finally {
        if(rs != null) {
            try {
                rs.close();
            } catch (SQLException e){}
        }
        if(ps != null) {
            try {
                ps.close();
            } catch (SQLException e){}
        }
        if(c != null) {
            try {
                c.close();
            } catch (SQLException e){}
        }
    }
}
```


### 3.2 변하는 것과 변하지 않는 것
---
##### 3.2.1 JDBC try/catch/finally 코드의 문제점
메소드마다 적용해야 하기 때문에 코드 상에 빠진 부분이 있을 수 있고, 테스트 코드를 만들더라도 그 양이 매우 늘어날 것이라는 문제가 있다.

##### 3.2.2 분리와 재사용을 위한 디자인 패턴 적용
1. 메소드 추출  
변하는 부분을 메소드로 빼낸다.  
자주 바뀌는 부분을 메소드로 독립시킨 경우 분리시키고 남은 메소드가 재사용이 필요하고 분리된 메소드는 DAO 로직마다 새롭게 만들어서 확장되어야 하는 부분이 된다.  

2. 템플릿 메소드 패턴의 적용  
템플릿 메소드 패턴은 상속을 통해 기능을 확장해서 사용하는 부분. 변하지 않는 부분은 슈퍼클래스에 두고 변하는 부분은 추상 메소드로 정의해 서브클래스에서 오버라이드하여 새롭게 정의해 쓰도록 하는 것.  
하지만 DAO 로직마다 상속을 통해 새로운 클래스를 만들어야 한다는 단점이 있다. 또한 확장구조가 이미 클래스 설계 시점에서 고정되기 때문에 서브클래스들이 이미 클래스 레벨에서 컴파일 시점에 관계가 설정되어 있다.

3. 전략 패턴의 적용  
* 전략 패턴  
: 오브젝트를 아예 둘로 분리하고, 클래스 레벨에서는 인터페이스를 통해서만 의존하도록 만드는 것. 템플릿 메소드 패턴보다 유연하고 확장성이 뛰어남.
OCP(개방 폐쇄 원칙)에서 보면 확장에 해당하는 변하는 부분을 별도의 클래스로 만들어 추상화된 인터페이스를 통해 위임하는 방식.  

* deleteAll()의 컨텍스트
    - DB 커넥션 가져오기
    - PreparedStatement를 만들어줄 외부 기능 호출하기 => 전략
    - 전달받은 PreparedStatement 실행하기
    - 예외가 발생하면 이를 다시 메소드 밖으로 던지기
    - 모든 경우에 만들어진 PreparedStatement와 Connection 적절히 닫아주기  

PrepardStatement를 만드는 전략의 인터페이스는 컨텍스트가 만들어둔 Connection을 전달받아서 PreparedStatement를 만들고 만들어진 PreparedStatement 오브젝트를 돌려준다.

##### 인터페이스
```java
public interface StatementStrategy {
    PreparedStatement makePreparedStatement(Connection c) throws SQLException;
}
```
##### 전략 클래스
``` java
public class DeleteAllStatement implements StatementStrategy {
    public PreparedStatement makePreparedStatement(Connection c) throws SQLException {
        PreparedStatement ps = c.prepareStatement("delete from users");
        return ps;
    }
}
```
##### deleteAll()
``` java
public void deleteAll() throws SQLException {
    ...
    try {
        c = dataSource.getConnection();

        StatementStrategy strategy = new DeleteAllStatement();
        ps = strategy.makePreparedStatement(c);

        ps.executeUpdate();
    } catch (SQLException e) { ... }
}
```
전략 패턴은 필요에 따라 컨텍스트는 그대로 유지되면서 전략을 바꿔쓸 수 있어야 하지만 이미 구체적인 전략 클래스인 DeleteAllStatement를 사용하도록 고정되어 있고 컨텍스트가 인터페이스뿐 아니라 특정 구현 클래스를 직접 알고 있다는 것은 전략패턴과 OCP에도 맞지 않는다.

4. DI 적용을 위한 클라이언트/컨텍스트 분리
전략 패턴에 의하면 Context가 어떤 전략을 사용하게 할 것인지는 Client가 구체적인 전략을 선택하고 오브젝트로 만들어서 Context에 전달한다.  
이 구조에서 전략 오브젝트 생성과 컨텍스트로의 전달을 담당하는 책임을 ObjectFactory로 분리시켰고, 이를 일반화 한 것이 DI이다.
컨텍스트에 해당하는 부분은 별도의 메소드로 독립시켜야 하고, 클라이언트는 전략 클래스의 오브젝트를 컨텍스트의 메소드를 호출하며 전달해야 한다.   

        **마이크로 DI**  
        의존관계 주입은 다양한 형태로 적용할 수 있다. DI의 가장 중요한 개념은 제3자의 도움을 통해 두 오브젝트 사이의 유연한 관계가 설정되도록 만드는 것이다.  
        일반적으로 의존관계에 있는 두 개의 오브젝트와 이 관계를 동적으로 설정해주는 오브젝트 팩토리(DI 컨테이너), 클라이언트 4개의 오브젝트 사이에서 일어난다. 하지만 클라이언트가 오브젝트 팩토리의 책임을 함께 지고 있거나 클라이언트와 전략이 결합될 수도 있다.  
        이런 경우 DI가 매우 작은 단위의 코드와 메소드 사이에서 일어나기도 한다. 이렇게 DI의 장점을 단순화해서 IoC 컨테이너의 도움 없이 코드 내에서 적용한 경우를 마이크로 DI라고 한다.

    
## 3.3 JDBC 전략 패턴의 최적화
---
#### 3.3.1 전략 클래스의 추가 정보
add() 메소드에 전략 패턴을 추가하는 경우 add()에서는 PreparedStatement를 만들 때 userf라는 부가적인 정보가 필요하다. 이러한 경우 클라이언트로부터 User 타입 오브젝트를 받을 수 있도록 AddStatement의 생성자를 통해 제공받아야 한다. 

#### 3.3.2 전략과 클라이언트의 동거
* 문제점
    - DAO 메소드마다 새로운 StatementStrategy 구현 클래스를 만들어야 하기 때문에 클래스 파일 개수가 많아진다.
    - DAO 메소드에서 StatementStrategy에 전달할 부가적인 정보가 있는 경우, 이를 위해 오브젝트를 전달받는 생성자와 이를 저장해둘 인스턴스 변수를 만들어야 한다.


1. 로컬 클래스 
클래스 파일이 많아지는 문제에 대한 해결 방법. StatementStrategy 전략 클래스를 매번 독립된 파일로 만들지 말고 UserDao 클래스 안에 내부 클래스로 정의한다.   

        **중첩 클래스의 종류**
        중첩 클래스 : 다른 클래스 내부에 정의되는 클래스
            - static class : 독립적으로 오브젝트로 만들어 질 수 있음
            - inner class : 자신이 정의된 오브젝트 안에서만 만들어 질 수 있음
        inner class는 범위에 따라 세 가지로 구분됨
            - member inner class : 멤버 필드처럼 오브젝트 레벨에 정의됨
            - local class : 메소드 레벨에 정의됨
            - anonymous inner class : 이름을 갖지 않음

    AddStatement 클래스를 로컬 클래스로 add() 메소드 안에 넣는다. 로컬 클래스는 선언된 메소드 내에서만 사용할 수 있다. 또한 로컬 클래스는 클래스가 내부 클래스이기 때문에 자신이 선언된 곳의 정보에 접근할 수 있다. 따라서 생서자를 통해 오브젝트를 전달해줄 필요가 없게 된다. 

2. 익명 내부 클래스  
AddStatement 클래스는 add() 메소드에서만 사용할 용도로 만들어졌기 때문에 클래스 이름을 제거할 수 있다. 익명 내부 클래스는 선언과 동시에 오브젝트를 선언한다. 이름이 없기 때문에 자신의 타입을 가질 수 없고, 구현한 인터페이스 타입의 변수에만 저장할 수 있다.


### 3.4 컨텍스트와 DI
---
#### 3.4.1 JdbcContext의 분리
전략패턴의 구조에서 보면 UserDao의 메소드가 클라이언트, 익명 내부 클래스로 만들어지는 것이 개별적인 전략, jdbcContextWithStatementStrategy()는 컨텍스트. jdbcContextWithStatementStrategy()는 다른 DAO에서도 사용 가능하므로 UserDao 클래스 밖으로 독립시켜 모든 DAO가 사용할 수 있도록 한다.  

1. 클래스 분리
##### JDBC 작업 흐름을 분리해서 만든 JdbcContext 클래스
```java
public class JdbcContext {
    private DataSource dataSource;

    public void setDataSource(DataSource dataSource) {
        this.dataSource = dataSource;
    }

    public void workWithStatementStrategy(StatementStrategy stmt) throws SQLException {
        Connecton c = null;
        PreparedStatement ps = null;

        try {
            c = this.dataSource.getConnection();
            ps = stmt.makePreparedStatement(c);
            ps.executeUpdate();
        } catch (SQLException e) { throw e; }
        finally {
            if(ps != null) {
                try{ ps.close(); } catch (SQLException e) {}
            }
            if(c != null) {
                try{ c.close(); } catch (SQLException e) {}
            }
        }
    }
}
````   
##### UserDao
```java
public class UserDao {
    private JdbcContext jdbcContext;

    public void setJdbcContext(JdbcContext jdbcContext) {
        this.jdbcContext = jdbcContext;
    }

    public void add(final User user) throws SQLException {
        this.jdbcContext.workWithStatementStrategy (
            new StatementStrategy() { ... }
        );
    }
    ...
}
```

2. 빈 의존관계 변경  
    JdbcContext는 그 자체로 독립적인 JDBC 컨텍스트를 제공해주는 서비스 오브젝트로서의 의미가 있을 뿐이고 구현 방법이 바뀔 가능성은 없으므로 인터페이스를 구현하도록 하지 않았기 때문에 UserDao와 인터페이스를 사이에 두지 않고 DI를 적용하는 구조가 된다.

#### 3.4.2 JdbcContext의 특별한 DI
1. 스프링 빈으로 DI  
DI 개념에 따르면 인터페이스를 사이에 둬서 클래스 렙벨에서는 의존관계가 고정되지 않게 하고, 런타임 시에 의존할 오브젝트와의 관계를 다이내믹하게 주입해주어야 한다.  
하지만 스프링의 DI는 넓게 보자면 객체의 생성과 관계 설정에 대한 제어 권한을 오브젝트에서 제거하고 외부로 위임했다는 IoC라는 개념을 포괄한다.    

* JdbcContext가 UserDao와 DI 구조로 만들어야 할 이유
    - JdbcContext가 스프링 컨테이너의 싱글톤 레지스트리에서 관리되는 싱글톤 빈이기 때문. JdbcContext는 그 자체로 변경되는 상태정보를 갖고 있지 않기 때문에 싱글톤이 되는데 아무 문제가 없다.
    - JdbcContext가 DI를 통해 다른 빈에 의존하고 있다. dataSource 프로퍼티를 통해 DataSource 오브젝트를 주입받도록 되어 있다. DI를 위해서는 주입되는 오브젝트와 주입받는 오브젝트 양쪽 모두 스프링 빈으로 등록되어야 한다. 스프링이 생성하고 관리하는 IoC 대상이어야 하기 때문이다.
    - 인터페이스를 사용하지 않은 이유는 UserDao와 JdbcContext가 강한 응집도를 갖고 있기 때문이다.

2. 코드를 이용하는 수동 DI  
UserDao 내부에서 직접 DI를 적용하는 경우 DAO 마다 하나의 JdbcContext 오브젝트를 만들어야 한다. JdbcContext에는 상태정보가 없기 때문에 수백개가 만들어진다고 해도 메모리에 주는 부담은 거의 없고, 자주 생성되고 제거되지 않기 때문에 GC에 대한 부담도 없다.  
JdbcContext를 스프링 빈으로 등록하지 않았기 때문에 UserDao가 JdbcContext의 생성과 초기화를 책임져야 한다.  
JdbcContext는 다른 빈을 인터페이스를 통해 간접접으로 의존하고 있다. 의존 오브젝트를 DI를 통해 제공받기 위해서는 자신도 빈으로 등록되어야 하기 때문에 JdbcContext에 대한 제어권을 갖고 생성과 관리를 담당하는 UserDao에 DI를 맡겨야 한다. userDao 빈에 DataSource 타입 프로퍼티를 지정해서 dataSouce 빈을 주입받도록 하면 JdbcContext 오브젝트를 만들면서 DI 받은 DataSource 오브젝트를 JdbcContext의 수정자 메소드로 주입해준다. 만들어진 JdbcContext 오브젝트는 UserDao의 인스턴스 변수에 저장해두고 사용한다. 


### 3.5 템플릿과 콜백
---
**템플릿/콜백 패턴** 
    
    * 템플릿 : 어떤 목적을 위해 미리 만들어둔 모양이 있는 틀. (전략 패턴의 컨텍스트)

    * 콜백 : 실행되는 것을 목적으로 다른 오브젝트 메소드에 전달되는 오브젝트. 파라미터로 전달되지만 값을 참조하기 위한 것이 아닌 특정 로직을 담은 메소드를 실행하기 위해 사용. (익명 내부 클래스로 만들어지는 오브젝트)

#### 3.5.1 템플릿/콜백의 동작 원리
1. 템플릿/콜백의 특징  
템플릿/콜백 패턴에서는 템플릿의 작업 흐름 중 특정 기능을 위해 한 번 호출되는 경우가 일반적이기 때문에 콜백은 보통 단일 메소드 인터페이스를 사용한다. 콜백은 일반적으로 하나의 메소드를 가진 인터페이스를 구현한 익명 내부 클래스로 만들어진다.  
콜백 인터페이스 메소드에는 보통 템플릿 작업 흐름 중에 만들어지는 컨텍스트 정보를 전달받을 때 사용되는 파라미터가 있다.  

* 템플릿/콜백의 작업 흐름
    - 클라이언트가 템플릿 안에서 실행될 로직을 담은 콜백 오브젝트를 만들고 콜백이 참조할 정보를 제공한다. 만들어진 콜백은 클라이언트가 템플릿의 메소드를 호출할 때 파라미터로 전달된다.  
    - 템플릿은 정해진 작업 흐름을 따라 작업을 진행하다가 내부에서 생성한 참조정보를 가지고 콜백 오브젝트의 메소드를 호출한다. 콜백은 클라이언트 메소드에 있는 정보와 템플릿이 제공한 참조 정보를 이용해 작업을 수행하고 결과를 템플릿에 돌려준다.
    - 템플릿은 콜백이 돌려준 정보를 사용해서 작업을 수행하고, 최종 결과를 클라이언트에 다시 돌려주기도 한다.

    이는 DI 방식의 전략 패턴 구조와 비슷한데, 메소드 단위로 사용할 오브젝트를 수정자 메소드가 아닌 새롭게 전달받고, 콜백 오브젝트가 내부 클래스로 자신을 생성한 클라이언트 메소드 내의 정보를 직접 참조한다는 것은 템플릿/콜백의 고유한 특징이다. 또한 클라이언트와 콜백이 강하게 결합된다. 

2. JdbcContext에 적용된 템플릿/콜백  
JdbcContext의 workWithStatementStrategy() 템플릿은 리턴 값이 없는 단순한 구조이다. 조회작업에서는 템플릿의 작업 결과를 클라이언트에 리턴하고, 작업 흐름이 복잡한 경우 한 번 이상의 콜백을 호출하거나 여러 개의 콜백을 클라이언트로부터 받아서 사용하기도 한다. 

#### 3.5.2 편리한 콜백의 재활용  
1. 콜백의 분리와 재활용  
익명 내부 클래스 사용을 최소화 할 수 있는 방법은 자주 바뀌지 않는 부분을 분리하는 것이다.
StatementStrategy 인터페이스의 makePreparedStatement()에서는 SQL만 다르고 중복되므로 SQL 문장을 파라미터로 받아서 바꿀 수 있게 하고, 별도의 메소드로 만들면 된다.

    ``` java
    public void deleteAll() throws SQLException {
        executeSql("delete from users");
    }
    ----------------------------------------------
    puvlic void executeSql(final String query) throws SQLException {
        this.jdbcContext.workWithStatementStrategy(
            new StatementStrategy() {
                public PreparedStatement makePreparedStatement(Connetion c) throws SQLException {
                    return c.preparedStatement(query);
                }
            }
        )
    }
    ```

2. 콜백과 템플릿의 결합  
    executeSql()은 재사용 가능한 콜백을 담고 있기 때문에 DAO가 공유할 수 있는 템플릿 클래스 안으로 옮길 수 있다. 템플릿은 workWithStatementStrategy() 메소드이므로 JDBCContext 클래스로 콜백 생성과 템플릿 호출이 담긴 executeSql()을 옮겨도 문제가 되지 않는다.  
    또한 UserDao 메소드에서도 jdbcContext를 통해 executeSql()을 호출하도록 수정해야 한다.  

    ##### JdbcContext로 옮긴 executeSql()
    ``` java
    public class JdbcContext {
        ...
        // 외부에서 바로 접근이 가능하도록 public으로 선언
        public void executeSql(final String query) throws SQLException {
            workWithStatementStrategy(
                new StatementStrategy() {
                    public PreparedStatement makePreparedStatement(Connection c) throws SQLException {
                        return c.preparedStatement(query);
                    }
                }
            );
        }
    }
    ```
    ##### executeSql()을 사용하는 deleteAll()
    ``` java
    public void deleteAll() throws SQLException {
        this.jdbcContext.executeSql("delete from users");
    }
    ```
    일반적으로 성격이 다른 코드들은 가능한 분리하는 편이 낫지만, 하나의 목적을 위해 서로 긴밀하게 연관되어 동작하는 응집력이 강한 코드들이기 때문에 한 군데 모여 있는 것이 유리하다. 구체적인 구현과 내부의 전략 패턴, 코드에 의한 DI, 익명 내부 클래스 등의 기술은 최대한 감춰두고 외부에는 꼭 필요한 기능을 제공하는 단순한 메소드만 노출한다.

#### 3.5.3 템플릿/콜백의 응용
1. 테스트와 try/catch/finally  
파일을 열어서 모든 라인의 숫자를 더한 합을 돌려주는 코드를 테스트하는 테스트 코드.  
파일을 읽거나 처리하다가 예외가 발생하면 파일이 정상적으로 닫히지 않고 메소드를 빠져나가는 문제가 발생할 수 있으므로 try/finally를 이용해서 어떠한 경우에도 파일을 닫아줄 수 있도록 해야 한다.
``` java
public Integer calcSum(String filepath) throws IOException {
    BufferedReader br =null;
    try {
        br =new BufferedReader(new FileReader(filepath)); Integer sum =0;
        String line =null;
        while((line = br.readLine()) != null) {
            sum += Integer.valueOf(line); 
        }
        return sum;
    } catch(IOException e) { 
        System.out.println(e.getMessage()); 
        throw e;
    } finally {
        if(br != null) {
            try {br.close();}
            catch(IOException e) {System.out.println(e.getMessage());}
        }
    }
```

2. 중복의 제거와 템플릿/콜백 설계  
파일을 읽어서 처리하는 비슷한 기능이 필요한 경우 템플릿 콜백 패턴을 적용할 수 있다.  
콜백의 인터페이스 정의를 위해 템플릿이 콜백에게 전달해줄 내부 정보는 무엇이고, 콜백이 템플릿에 돌려줄 내용은 무엇인지, 템플릿이 작업을 마친 뒤 돌려줄 내용인 무엇인지에 대해 파악해야 한다.  
```java
public Integer fileReadTemplate(String filepath, BufferedReaderCallback callback) throws IOException {
    BufferedReader br =null; 
    try {
        br =new BufferedReader(new FileReader(filepath));
        int ret = callback.doSomethingWithReader(br);
        return ret;
    } catch(IOException e) {
        System.out.println(e.getMessage());
        throw e;
    } finally {
        if(br != null) {
            try {br.close();}
            catch(IOException e) {System.out.println(e.getMessage());}
        }
    }
```

BufferReader를 만들어서 넘겨주는 것과 그 외의 작업에 대한 작업 흐름은 템플릿에서 진행하고 준비된 BufferReader를 이용해 작업을 수행하는 부분은 콜백을 호출해서 처리하도록 한다.  
템플릿으로 분리한 부분을 제외한 나머지 코드를 BufferReaderCallback 인터페이스로 만든 익명 내부 클래스에 담는다. 처리할 파일의 경로와 함께 준비된 익명 내부 클래스의 오브젝트를 템플릿에 전달한다.
```java
public Integer calcSum(String filepath) throws IOException{
    BufferedReaderCallback sumCallback = new BufferedReaderCallback() {
        public Integer doSomethingWithReader(BufferedReader br) throws IOException {
            Integer sum =0;
            String line =null;
            while((line = br.readLine()) != null) {
                sum += Integer.valueOf(line); 
            }
            return sum;
        }
    };
    return fileReadTemplate(filepath, sumCallback);
}
```

테스트 메소드에서 사용할 클래스의 오브젝트와 파일이름이 공유되는 경우 @Before 메소드에서 미리 픽스처로 만들어 두는 것이 좋다.
```java
public class CalSumTest {
    Calculator calculator;
    String numFilepath;

    @Before 
    public void setUp(){
        this.calculator = new Calculator();
        this.numFilepath = getClass().getResource("number.txt").getPath();
    }

    @Test
    public void sumOfNumbers() throws IOException {
        assertThat(calculator.calcSum(this.numFilepath), is(10));
    }
    
    @Test
    public void multiplyOfNumbers() throws IOException {
        assertThat(calculator.calcMultiply(this.numFilepath), is(24));
    }
}
```

3. 템플릿/콜백의 재설계  
템플릿과 콜백을 찾아낼 때는, 변하는 코드의 경계를 찾고 그 경계를 사이에 두고 주고받는 일정한 정보가 있는지 확인하면 된다.  

4. 제네릭스를 이용한 콜백 인터페이스  
제네릭스를 이용하면 다양한 오브젝트 타입을 지원하는 인터페이스나 메소드를 정의할 수 있다. 
``` java
public interface LineCallback<T>{
    T doSthWithLine(String line, T value);
}
```
``` java
LineCallback<Integer> sumCallback = new LineCallback<Integer>() {...};
```


### 3.6 스프링의 JdbcTemplate
##### JdbcTemplate 초기화를 위한 코드
``` java
public class UserDao {
    ...
    private JdbcTemplate jdbcTemplate;

    public void setDataSource(DataSource dataSource) {
        this.jdbcTemplat = new JdbcTemplate(dataSource);

        this.dataSource = dataSource;
    }
}
```

#### 3.6.1 update()
파라미터로 SQL 문장을 전달한다.
``` java
public void deleteAll() {
    this.jdbcTemplate.update("delete from users");
}

public int add() {
    ... 
    PreparedStatement ps = c.prepareStatement("insert into users(id, name, password) values(?,?,?)");
    ps.setString(1, user.getId());
    ps.setString(2, user.getName());
    ps.setString(3, user.getPassword());
    ...
}
```

#### 3.6.2 queryForInt()
ResultSet을 통해 결과값을 가져오는 경우 PreparedStatementCreator 콜백과 ResultSetExtractor 콜백 템플릿을 사용할 수 있다. ResultSetExtractor 콜백은 템플릿이 제공하는 ResultSet을 이용해 원하는 값을 추출해서 템플릿에 전달하면, 템플릿은 나머지 작업을 수행한 후 그 값을 query() 메소드의 리턴 값으로 돌려준다.  
queryForInt()는 이러한 기능을 가진 콜백을 내장하고 있다. 
```java
public int getCount(){
    return this.jdbcTemplate.queryForInt("select count(*) from users");
}
```

#### 3.6.3 queryForObject()
RowMapper 콜백은 템플릿으로부터 ResultSet을 전달받고, 필요한 정보를 추출해서 리턴하는 방식으로 동작한다. RowMapper는 ResultSet의 로우 하나를 매핑하기 위해 사용되므로 여러 번 호출될 수 있다.  
queryForObject는 SQL을 실행하면 한 개의 로우만 얻을 것이라고 기대하고, ResultSet의 next()를 실행해서 첫 번째 로우로 이동시킨 후에 RowMapper 콜백을 호출한다. RowMapper가 호출되는 시점에서 ResultSet은 첫번째 로우를 가리키고 있으므로 rs.next()를 호출할 필요가 없고, RowMapper에서는 현재 ResultSet이 가리키고 있는 로우의 내용을 User 오브젝트에 그대로 담아서 리턴해준다.  
queryForObject()를 이용할 때는 조회 결과가 없는 예외 상황에서는 **EmptyResultDataAccessException** 예외가 발생한다. 


#### 3.6.4 query()
1. 기능 정의와 테스트 작성  
1. query() 템플릿을 이용하는 getAll() 구현  
2. 테스트 보완

#### 3.6.5 재사용 가능한 콜백의 분리
1. DI를 위한 코드 정리  
필요 없어진 DataSource 인스턴스 변수를 제거하고, JdbcTemplate을 생성하면서 직접 DI 해주기 위해 필요한 DataSource를 전달받기 위한 setter는 남겨둔다.
```java
private JdbcTemplate jdbcTemplate;

public void setDataSource(DataSource dataSource) {
    this.jdbcTemplate = new JdbcTemplate(dataSource);
}
```

2. 중복 제거
RowMapper가 중복되므로 콜백은 하나만 만들어서 공유한다.  
userMapper라는 이름으로 인스턴스 변수를 만들고 사용할 매핑용 콜백 오브젝트를 초기화하도록 만든다. 익명 내부 클래스는 클래스 안에서라면 어디서든 만들 수 있다.  

3. 템플릿/콜백 패턴과 UserDao  
UserDao에는 User 정보를 DB에 넣거나 가져오거나 조작하는 방법에 대한 로직만 담겨 있다. 자바 오브젝트와 USER 테이블 사이에 어떻게 정보를 주고받을지, DB와 커뮤니케이션 하기 위한 SQL 문장이 무엇인지에 대한 최적화된 코드를 가지고 있다.  
JDPC API를 사용하는 방식, 예외처리, 리소스 반납, DB 연결을 가져오는 방법에 관한 책임과 관심은 모두 JdbcTemplate에 있다. 따라서 변경이 일어나도 UserDao 코드에는 아무런 영향을 주지 않는다.