# Chapter7. 스프링 핵심 기술의 응용
## 7.1 SQL과 DAO의 분리
- DAO : 데이터를 가져오고 조작하는 작업의 인터페이스 역할
- SQL 변경이 필요한 상황이 발생하면 SQL을 담고 있는 DAO 코드가 수정될 수 밖에 없으므로 SQL과 DAO의 분리가 필요함
### 7.1.1 XML 설정을 이용한 분리
#### 개별 SQL 프로퍼티 방식
- SQL을 외부로 빼서 프로퍼티로 정의하
- add() 에서는 외부로부터 DI 받은 SQL 문장을 담은 메소드 사용
- XML 설정은 userDao 빈에 프로퍼티 추가
- 매번 새로운 SQL이 필요할 때마다 프로터피를 추가하고 DI를 위한 변수와 수정자 메소드를 만들어야 하

#### SQL 맵 프로퍼티 방식
- SQL을 하나의 컬렉션으로 담아두는 방식
- 맵을 이용해 키 값을 이용하면 프로퍼티는 하나만 만들어도 됨
- Map은 하나 이상의 복잡한 정보를 담고 있기 때문에 XML 설정시 `<property>` 대신 `<map>`을 사용해야 함
- 맵으로 만들어 두면 새로운 SQL이 필요할 때 설정에 `<entry>`만 추가해주면 됨

### 7.1.2 SQL 제공 서비스
- SQL과 설정정보를 함께 두는 것은 바람직하지 못함
- SQL을 따로 분리해둬야 독립적으로 SQL문의 리뷰나 튜닝 작업을 수행하기도 편함  

#### SQL 서비스 인터페이스
```java
public interface SqlService {
    String getSql(String key) throw SqlRetrievalFailureExcepiton;
}
```
```java
public class SqlRetrievalFailureException extends RuntimeException{ 
    public SqlRetrievalFailureException(String message) {
        super(message);
    }

    public SqlRetrievalFailureException(String message, Throwable cause) {
        super(message, cause);
    }
}
```
```java
public class UserDaoJdbc implements UserDao {
    ...
    private SqlService sqlService;

    public void setSqlService(SqlService sqlService) {
        this.sqlService = sqlService;
    }
}
```
#### 스프링 설정을 사용하는 단순 SQL 서비스
- SqlService 인터페이스에는 어떤 기술적인 조건이나 제약사항도 담겨 있지 않다.
- 어떤 방법을 사용하든 DAO가 요구하는 SQL을 돌려주기만 하면 된다. 
- SqlService 인터페이스를 구현하는 클래스에서 Map타입 프로퍼티를 추가하고 맵에서 SQL을 읽어서 돌려주도록 구현

## 7.2 인터페이스의 분리와 자기참조 빈
### 7.2.1 XML 파일 매핑
#### JAXB
- 장점 
    - XML 문서정보를 거의 동일한 구조의 오브젝트로 직접 매핑해줌
    - XML 정보를 그대로 담고 있는 오브젝트 트리 구조로 만들어주기 때문에 XML 정보를 오브젝트처럼 다룰 수 있음
    - XML 문서의 구조를 정의한 스키마를 이용해서 매핑할 오브젝트의 클래스까지 자동으로 만들어주는 컴파일러도 제공

#### SQL 맵을 위한 스키마 작성과 컴파일
- SQL 정보는 키와 SQL 목록으로 구성된 맵 구조로 만들어두면 편리함
```xml
<sqlmap>
    <sql key="userAdd">insert into users(...)...</sql>
    <sql key="userGet">select * from users ...</sql>
    ...
```
- XML 문서의 구조를 정의하는 스키마도 필요함
- 만든 스키마 파일은 sqlmap.xsd라는 이름으로 프로젝트 루트에 저장하고 JAXB 컴파일러로 컴파일
- 컴파일 하면 두 개의 바인딩용 자바 클래스와 팩토리 클래스가 만들어진다. 
- XML 문서 바인딩용 클래스를 살펴보면 `<sqlmap>`이 바인딩 될 SqlmapType 클래스와 `<sql>` 태그의 정보를 담을 SqlType 클래스가 있다. 

#### 언마샬링
**언마샬링** : XML 문서를 읽어서 자바의 오브젝트로 변환하는 것  
**마샬링** : 바인딩 오브젝트를 XML 문서로 변환하는 것  
cf) **직렬화** : 자바 오브젝트를 바이트 스트림으로 바꾸는 것  

### 7.2.2 XML 파일을 이용하는 SQL 서비스
#### SQL 맵 XML 파일
- SQL은 DAO 로직의 일부라고 볼 수 있으므로 같은 패키지에 두는 게 좋다. 
#### XML SQL 서비스
- XML 파일로부터 익은 내용은 어딘가에 저장해두고 DAO에서 요청이 올 때 사용해야 함
- 처음 SQL을 읽어들이는 방법  
    : 생성자에서 SQL을 읽어와 내부에 저장  
    - JAXB 컴파일러가 생성해준 XML 문서 바인딩용 클래스들을 이용해 XML 문서를 언마샬링하면 SQL 문장들은 Sql 클래스의 오브젝트에 하나씩 담긴다. 
    - 매번 검색을 위해 리스트를 모두 검사해야 하므로 비효율적 
    - 상대적으로 검색 속도가 빠른 Map 타입 오브젝트에 저장하는 것이 나음
    - 생성자에서 XML 파일을 읽어서 맵에 저장하고, SQL을 맵에서 찾아서 돌려주는 getSql()을 구현하면 됨

### 7.2.3 빈의 초기화 작업
- 생성자에서 예외가 발생할 수도 있는 복잡한 초기화 작업을 다루는 것은 좋지 않음 
- 오브젝트 생성 중에 생성자에서 발생하는 예외의 문제
    - 다루기 힘듦
    - 상속하기 불편
    - 보안 문제 발생 가능
- 초기 상태를 가진 오브젝트를 만들고 별도의 초기화 메소드를 사용하는 방법이 바람직
- 읽어들일 파일의 위치와이름이 코드에 고정되는 것도 좋지 않음
- 바뀔 가능성이 있는 내용은 외부에서 DI로 설정해줄 수 있게 만들어야 함
- XmlSqlService 클래스 수정
    1. 프로퍼티 추가
        ```java
        private String sqlmapFile;

        public void setSqlmapFile(String sqlmapFile) {    
            this.sqlmapFile = sqlmapFile;
        }
        ```
    2. 생성자 대신 별도의 초기화 메소드 사용
        ```java
        public void loadSql() {
            String contextPath =Sqlmap.class.getPackage().getName(); 
            try {
                ...
                // 프로퍼티로 설정을 통해 제공받은 파일 이름 사용
                InputStream is = UserDao.class.getResourceAsStream(this.sqlmapFile);            
                ...
            }
        }
        ```
    3. 오브젝트를 만드는 시점에서 초기화 메소드 호출
        ```java
        XmlSqlService sqlProvider =new Xm1SqlService(); sq1Provider.setSq1mapFile( 닝q띠1뻐mn뻐l떠ap .xm1"); sqlProvider.loadSql();
        ```
- XmlSqlService 오브젝트는 빈이므로 제어권이 스프링에 있어서 생성과 초기화도 스프링에 맡겨야 한다. 
- **스프링의 빈 후처리기** : 스프링 컨테이너가 빈을 생성한 뒤 부가적인 작업을 수행할 수 있게 해주는 특별한 기능
- AOP를 위한 프록시 자동생성기가 대표적인 빈 후처리기
- context 네임 스페이스를 사용해서 `<context:annotation-config/>` 태그를 만들어 설정파일에 넣어주면 빈 설정 기능에 사용할 수 있는 특별한 어노테이션 기능을 부여해주는 빈 후처리기들이 등록됨
- `@PostConstruct`
    - 빈 오브젝트의 초기화 메소드를 지정하는 데 사용
    - 초기화 작업을 수행할 메소드에 부여해주면 XmlSqlService 클래스로 등록된 빈의 오브젝트를 생성하고 DI 작업을 마친 뒤 @PostConstruct가 붙은 메소드를 자동으로 실행해줌
- 마지막으로 sqlmapFile 프로퍼티 값은 sqlService 빈의 설정에 넣어준다.

### 7.2.4 변화를 위한 준비: 인터페이스 분리
#### 책임에 따른 인터페이스 정의
- 분리 가능한 관심사 구분
    1. SQL 정보를 외부의 리소스로부터 읽어옴  
    (어플리케이션에서 활용 가능하도록 메모리에 읽어 들이는 것)
    2. 읽어온 SQL을 보관해두고 있다가 필요할 때 제공   
    (SQL에 대한 어플리케이션 내의 저장소 제공)
- 부가적인 책임
    - 서비스를 위해 한 번 가져온 SQL을 필요에 따라 수정할 수 있게 하는 것
- 두 가지 책임 조합 방법
    - SqlService를 구현해서 DAO에 서비스를 제공해주는 오브젝트가 이 두 가지 책임을 가진 오브젝트와 협력해서 동작하도록 만들어야 함
    - 변경 가능한 기능은 전략 패턴을 적용해 별도의 오브젝트로 분리  
    - SqlService의 구현 클래스가 변경 가능한 책임을 가진 SqlReader와 SqlRegistry 두 가지 타입의 오브젝트를 사용하도로고 만듦
    - SqlRegistry의 일부 인터페이스는 SqlService가 아닌 다른 오브젝트가 사용할 수도 있다. 
![image](https://user-images.githubusercontent.com/44107947/61287260-756baa00-a7ff-11e9-9cbf-ed3354a4cde5.png)
- SqlService가 일단 SqlReader에서 정보를 전달받은 뒤, SqlRegistry에 다시 전달해야 할 필요는 없다. 
- 두 오브젝트 사이의 정보를 전달하는 것이 전부라면 SqlService에게 SqlRegistry 전략을 제공해주면서 이를 이용해 SQL 정보를 SqlRegistry에 저장하라고 요청하는 편이 낫다.
```java
//변경된 SqlService 코드
//SQL을 저장할 대상인 sqlRegistry 오브젝트를 전달한다.
sqlReader.readSql(sqlRegistry);
```
```java
// SqlReader는 읽어들인 SQL을 이 메소드를 이용해 레지스트리에 저장
interface SqlRegistry{
    void registerSql(String key, String sql);
}
```
- 자바의 오브젝트는 자신이 가진 데이터를 이용해 어떤 작업을 할지 알고 있으므로 오브젝트 내부의 데이터를 외부로 노출 시킬 필요는 없다. 
- SqlReader는 내부에 가지고 있는 SQL 정보를 협력관계에 있는 의존 오브젝트인 SqlRegistry에게 필요에 따라 등록을 요청할 때만 활용하면 된다.
![image](https://user-images.githubusercontent.com/44107947/61290280-781dcd80-a806-11e9-858c-f77dd26ba03e.png)
- SqlReader가 사용할 SqlRegistry 오브젝트를 제공해주는 건 SqlService의 코드가 담당한다. (SqlRegistry가 일종의 콜백 오브젝트처럼 사용됨)
- SqlReader 입장에서는 SqlRegistry 인터페이스를 구현한 오브젝트를 런타임 시에 메소드 파라미터로 제공받아서 사용하는 구조이니 일종의 코드에 의한 수동 DI라고 볼 수도 있다.
- 동시에 SqlRegistry는 SqlService에게 등록된 SQL을 검색해서 돌려주는 기능을 제공하고 있기도 하므로 SqlService의 의존 오브젝트이기도 함
- SqlService의 구현 클래스는 두 개의 인터페이스 타입 프로퍼티를 갖고 있어서 각각의 구현 오브젝트를 DI 받도록 설정해주면 됨

#### SqlRegistry 인터페이스 
- SQL을 제공받아 등록해뒀다가 키로 검색해서 돌려주는 기능을 담당
```java
public interface SqlRegistry{
    void registerSql(String key, String sql);

    String findSql(String key) throws SqlNotFoundException;
}
```

#### SqlReader 인터페이스
- SqlRegistry 오브젝트를 메소드 파라미터로 DI 받아서 읽어들인 SQL을 등록하는데 사용하도록 만들어야 함
```java
public interface SqlReader{
    void read(SqlRegistry sqlRegistry);
}
```

### 7.2.5 자기참조 빈으로 시작하기
#### 다중 인터페이스 구현과 간접 참조
- SqlService의 구현 클래스는 SqlReader와 SqlRegistry 두 개의 프로퍼티를 DI 받을 수 있는 구조로 만들어야 함
![image](https://user-images.githubusercontent.com/44107947/61291714-427ae380-a80a-11e9-8fc4-c500cd04401e.png)
- XmlSqlprovider는 3개의 인터페이스와 이를 구현한 3개의 클래스로 이뤄짐
- XmlSqlService 클래스는 다형성을 활용하여 위 세 개의 인터페이스를 구현하도록 만든다.
![image](https://user-images.githubusercontent.com/44107947/61292164-799dc480-a80b-11e9-97c0-217e2b558706.png)

#### 인터페이스를 이용한 분리
- XmlSqlService는 SqlService 만을 구현한 독립적인 클래스라고 가정한다면 두 개의 인터페이스 타입 오브젝트에 의존하는 구조로 만들어야 한다.
- DI를 통해 이 두 개의 인터페이스를 구현한 오브젝트를 주입받을 수 있도록 프로퍼티 정의(생성자 주입)
- XmlSqlService 클래스가 SqlRegistry를 구현하도록 만든다.
    - 기존의 HashMap을 사용하는 코드는 그대로 유지하지만 SqlRegistry 인터페이스를 구현하는 메소드로 만들면, sqlMap은 SqlRegistry 구현의 일부가 된다.
    - 따라서 sqlMap은 독립적인 오브젝트라고 생각하고 SqlRegistry의 메소드를 통해 접근해야 함
- XmlSqlService 클래스가 SqlReader를 구현하도록 만든다.
    - SqlReader를 구현한 코드에서 XmlSqlService 내의 다른 변수와 메소드를 직접 참조하거나 사용하면 안됨
- @PostConstruct가 달린 빈 초기화 메소드와 SqlService 인터페이스에 선언된 메소드인 getFinder()를 sqlReader와 sqlRegistry를 이용하도록 변경

#### 자기참조 빈 설정
- 빈 설정을 통해 실제 DI가 일어나도록 해야 함
- 빈은 sqlService 하나만 선언하므로 실제 빈 오브젝트도 한 개만 만들어짐
- 책임이 다르다면 클래스를 구분하고 각기 다른 오브젝트로 만들어지는 것이 자연스러움
- 자기참조 빈은 책임과 관심사가 복잡하게 얽혀 있어 확장이 힘들고 변경에 취약한 구조의 클래스를 유연한 구조로 만들려고 할 때 처음 시도해 볼 수 있는 방법임
- 이를 통해 기존의 복잡하게 얽혀 있던 코드를 책임을 가진 단위로 구분해 낼 수 있음
- 당장 확장구조를 이용해 구현을 바꿔 사용하지 않더라도 확장구조를 만들어두는 게 좋다고 생각될 때 가장 간단히 접근할 수 있는 방법 

### 7.2.6 디폴트 의존관계
확장 가능한 인터페이스를 정의하고 인터페이스에 따라 메소드를 구분해서 DI가 가능하도록 이를 완전히 분리하고 DI로 조합해서 사용하게 만들어야 한다.
#### 확장 가능한 기반 클래스
- 자기참조가 가능한 빈으로 만들었던 XmlSqlService 코드에서 의존 인터페이스와 그 구현 코드를 제거하기만 하면 된다. (BaseSqlService)
- BaseSqlService를 sqlService 빈으로 등록하고 SqlReader와 SqlRegistry를 구현한 클래스 역시 빈으로 등록해서 DI 해주면 됨
- DI를 적용했기 때문에 SqlReader와 SqlRegistry의 구현 클래스는 자유롭게 변경해서 기능 확장 가능
- JAXB를 이용해 XML 파일에서 SQL 정보를 읽어오는 코드를 SqlReader 인터페이스의 구현 클래스로 독립시킨다.
- 분리한 클래스를 각각 빈으로 등록

#### 디폴트 의존관계를 갖는 빈 만들기
- **디폴트 의존관계** : 외부에서 DI 받지 않는 경우 기본적으로 자동 적용되는 의존관계
- DI 설정이 없을 경우 디폴트로 적용하고 싶은 의존 오브젝트를 생성자에서 넣어준다.
- DI란 클라이언트 외부에서 의존 오브젝트를 주입해주는 것이지만, 자신이 사용할 디폴트 의존 오브젝트를 스스로 DI 하는 방법도 있다. (DefaultSqlService)
- 이 경우 테스트에 실패하는데, DefaultSqlService의 경우 내부에서 생성하는 JaxbXmlSqlReader의 sqlmapFile 프로퍼티가 비어 있기 때문이다.
- 해결 방법 
    - sqlmapFile을 DefaultSqlService의 프로퍼티로 정의 
        - DefaultSqlService가 sqlmapFile을 받아서 내부적으로 JaxbXmlSqlReader를 만들면서 다시 프로퍼티로 넣어줌
        - JaxbXmlSqlReader는 디폴트 의존 오브젝트이므로 외부 클래스의 프로퍼티로 정의해서 전달받는 것은 적절하지 않다.
    - 관례적으로 사용할 만한 이름을 정해서 디폴트로 넣어줌
        - JaxbXmlSqlReader는 DefaultSqlService의 sqlReader로서 대부분 그대로 사용해도 좋기 때문에 디폴트 의존 오브젝트로 만든 것
        - SQL 파일 이름을 매번 바꿔야할 필요가 없으므로 sqlmapFile의 경우도 JaxbXmlSqlReader에 의해 기본적으로 사용될 만한 디폴트 값을 가질 수 있음
- DI를 한다고 새서 항상 모든 프로퍼티 값을 설정에 넣고 모든 의존 오브젝트를 빈으로 일일이 지정할 필요는 없다.
- 자주 사용되는 의존 오브젝트는 미리 지정한 디폴트 의존 오브젝트를 설정 없이 사용할 수 있게 만드는 것도 좋은 방법
- 단점 : 설정을 통해 다른 구현 오브젝트를 사용하게 해도 DefaultSqlService는 생성자에서 일단 디폴트 의존 오브젝트를 다 만들어버린다. 

## 7.3 서비스 추상화 적용
### 7.3.1 OXM 서비스 추상화
- XML 자바오브젝트 매핑 기술
    - Castor XML
    - JiBX
    - XmlBeans
    - Xstream
- 스프링이 제공하는 OXM 추상 계층의 API를 이용해 XML 문서와 오브젝트 사이의 변환을 처리하면, 코드 수정 없이도 OXM 기술을 자유롭게 바꿔서 적용할 수 있다.

#### OXM 서비스 인터페이스
- **Marshaller** : Java Object -> XML
- **Unmarshaller** : XML -> Java Object
- SqlReader는 Unmarshaller를 이용하면 됨
- Unmarshaller 인터페이스
    - XML 파일에 대한 정보를 담은 Source 차입의 오브젝트를 주면 설정에서 지정한 OXM 기술을 이용해 자바 오브젝트 트리로 변환하고 루트 오브젝트 반환
    - Unmarshaller 인터페이스를 구현한 5개 클래스 존재
    - 각 클래스는 해당 기술에서 필요로 하는 추가 정보를 빈 프로퍼티로 지정 가능

#### JAXB 구현 테스트
- Jaxb2Marsharller 클래스 : Unmarsharller, Marshaller 인터페이스를 모두 구현하고 있음
- Jaxb2Marshaller 클래스를 빈으로 등록하고 바인딩 클래스의 패키지 이름을 지정하는 contextPath 프로퍼티만 넣어주면 됨

#### Castor 구현 테스트
- XML 매핑 파일 이용
- 매핑 정보만 적절히 만들어주면 어떤 클래스와 필듣로도 매핑이 가능하다.
- 설정 파일의 unmarrshaller 빈의 클래스를 Castor 용 구현클래스로 변경하고, mappingLocation 프로퍼티에는 Castor 용 매핑 파일의 위치를 지정한다. 

### 7.3.2 OXM 서비스 추상화 적용
#### 멤버 클래스를 참조하는 통합 클래스
- OxmSqlService는 SqlReader 타입의 의존 오브젝트를 사용하되 이를 스태틱 멤버 클래스로 내장학고 자신만이 사용할 수 있도록 만듦
- 의존 오브젝트를 자신만이 사용하도록 독점하는 구조로 만드는 방법
- 내부적으로 낮은 결합도를 유지한 채 응집도가 높은 구현을 만들 때 유용하게 쓸 수 있는 방법
- OxmSqlService와 OxmSqlReader는 구조적으로는 강하게 결합되어 있지만 논리적으로 명확하게 분리되는 구조
-> 자바의 스태틱 멤버 클래스를 쓰기에 적합
- OxmSqlReader는 private 멤버 클래스이므로 외부에서 접근하거나 사용할 수 없음
- OxmSqlService는 이를 final로 선언하고 직접 오브젝트를 생성하기 때문에 OxmSqlReader를 DI 하거나 변경할 수 없음
- 두 개의 클래스를 강하게 결합하고 더 이상의 확장이나 변경을 제한함으로써 서비스 구조로 최적화할 수 있고, 하나의 클래스로 만들어두기 때문에 빈의 등록과 설정은 단순해지고 쉽게 사용할 수 있다.
- OxmSqlReader는 외부에 노출되지 않기 때문에 OxmSqlService에 의해서만 만들어지고, 스스로 빈으로 등록될 수 없다.
- 따라서 자신이 DI를 통해 제공받아야 하는 프로퍼티가 있다면 이를 OxmSqlService의 공개된 프로퍼티를 통해 간접적으로 DI 받아야 한다.
- OxmSqlReader의 경우 두 개의 프로퍼티가 필요하고, 스프링이 어떤 순서로 프로퍼티를 설정해줄지 알 수 없기 때문에 어느 수정자 메소드에서 오브젝트를 생성해야 할 지 모른다. 
- 따라서 미리 오브젝트를 만들어두고 각 수정자 메소드에서는 DI 받은 값을 넘겨주기만 한다.

#### 위임을 이용한 BaseSqlService의 재사용
- loadSql()과 getSql()이 중복됨
- 위임 구조를 이용해 코드의 중복 제거 가능 
- loadSql()과 getSql()의 구현 로직은 BaseSqlService에만 두고, OxmSqlService는 일종의 설정과 기본 구성을 변경해주기 위한 어댑터 같은 개념으로 BaseSqlService의 앞에 두는 설계가 가능함
- OxmSqlService의 외형적인 틀은 유지한 채로 SqlService의 기능 구현은 BaseSqlService로 위임히는 
- 위임을 위해서는 두 개의 빈을 등록하고 클라이언트의 요청을 직접 받는 빈이 주요한 내용은 뒤의 빈에게 전달해주는 구조여야 함
- 하지만 특화된 서비스를 위해 한 번만 사용할 것이므로 두 클래스를 하나로 묶어도 됨
- OxmSqlService는 OXM 기술에 특화된 SqlReader를 멤버로 내장하고 있고, 그에 필요한 설정을 한 번에 지정할 수 있는 확장구조를 가지곡 있음
- SqlReader와 SqlService를 이용해 SqlService의 기능을 구현하는 일은 내부에 BaseSqlService를 만들어서 위임
![image](https://user-images.githubusercontent.com/44107947/61356461-60e7ea00-a8b1-11e9-9bcb-7afc47706fd4.png)

### 7.3.3 리소스 추상화
- 문제점 : XML 파일 이름을 프로퍼티로 외부에서 지정할 수는 있지만, 클래스패스에 존재하는 파일로 제한된다.
- 다양한 위치에 존재하는 리소스에 대해 단일화된 접근 인터페이스를 제공해주는 클래스가 없음

#### 리소스
- Resouce 인터페이스
    ```java
    public interface Resourrce extends InputStreamSource{
        boolean exists();
        boolean isReadable();
        boolean isOpen();
        
        URL getURL() throws IOException;
        URI getURI() throws IOException;
        File getFile() throws IOException;

        Resouce createRelatives(String relativePath) throws IOException;

        long lastModified() throws IOException;
        String getFilename();
        String getDescrirption();
    }

    public interface InputStreamSource{
        InputStream getInputStream() throws IOException;
    }
    ```
- 리소스는 다른 서비스 추상화의 오브젝트와 달리 빈이 아니라 값으로 취급된다.
- 따라서 추상화를 적용하는 방법이 문제가 됨
- Resource는 빈으로 등록하지 않으므로 외부에서 지정한다고 해도 `<property>`의 value 애트리뷰트에 넣는 방법밖에 없지만 value에는 단순 문자열만 들어갈 수 있다.

#### 리소스 로더
- URL 클래스와 유사하게 접두어를 이용해 Resource 오브젝트를 선언할 수 있음
- 문자열 안에 리소스 종류와 리소스 위치를 함께 표현하고 문자열로 정의된 리소스를 실제 Resource 타입 오브젝트로 변환해주는 ResourceLoader를 제공함
- 접두어가 없는 경우 리소스 로더의 구현 방식에 따라 리소스를 가져오는 방식이 달라짐
- 접두어를 붙여주면 리소스 로더의 종류와 상관없이 접두어가 의미하는 위치와 방법을 이용해 리소스를 읽어옴

#### Resource를 이용해 XML 파일 가져오기
- sqlmapFile 프로퍼티를 모두 Resource 타입으로 바꾸고 이름도 sqlmap으로 변경
- Resource 타입은 실제 소스가 어떤 것이든 상관없이 getInputStream()을 이용해 스트림으로 가져올 수 있음
- 이를 StreamSource 클래스를 이용해서 OXM 언마샬러가 필요로 하는 Source 타입으로 만들어주면 됨
- Resource는 단지 리소스에 접근할 수 있는 추상화 된 핸들러일 뿐이므로 Resource 타입의 오브젝트가 만들어졌다고 해도 실제로 리소스가 존재하지 않을 수 있음

## 7.4 인터페이스 상속을 통한 안전한 기능확장
어플리케이션을 새로 시작하지 않고 특정 SQL의 내용만을 변경하고 싶은 경우
### 7.4.1 DI와 기능의 확장
#### DI를 의식하는 설계
- DI를 적용하려면 최소한 두 개 이상의, 의존관계를 가지고 서로 협력하는 오브젝트가 필요하다.
- 따라서 적절한 책임에 따라 오브젝트를 분리해줘야 한다.
- 그리고 항상 의존 오브젝트는 자유롭게 확장될 수 있다는 점을 염두에 두어야 한다.
- DI의 목적은 런타임 시에 의존 오브젝트를 다이내믹하게 연결해줘서 유연한 확장을 가능하게 하는 것

#### DI와 인터페이스 프로그래밍
- DI를 적용할 때는 가능한 인터페이스를 사용하도록 해야 한다.
    - 다형성
    - 인터페이스 분리 원칙을 통해 클라이언트와 의존 오브젝트 사이의 관계를 명확하게 해줄 수 있음
- 인터페이스는 하나의 오브젝트가 여러 개를 구현할 수 있으므로, 하나의 오브젝트를 바라보는 창이 여러 가지 일 수 있음
- 각기 다른 관심과 목적을 가지고 어떤 오브젝트에 의존하고 있을 수 있다는 의미
- 오브젝트가 그 자체로 충분히 응집도가 높은 단위로 설계됐더라도, 목적과 관심이 각기 다른 클라이언트가 있다면 인터페이스를 통해 이를 적절하게 분리해줄 필요가 있다. (**인터페이스 분리 원칙**)

### 7.4.2 인터페이스 상속 
- 인터페이스 분리 원칙의 장점  
    - 모든 클라이언트가 자신의 관심에 따른 접근 방식을 불필요한 간섭 없이 유지할 수 있어서 기존 클라이언트에 영향을 주지 않은 채로 오브젝트의 기능을 확장하거나 수정할 수 있음
    - 인터페이스의 구현클래스가 또 다른 제 3의 클라이언트를 위한 인터페이스를 가질 수 있다.
- 인터페이스를 추가하거나 상속을 통해 확장하는 방식을 잘 활용하면 이미 기존의 인터페이스를 사용하는 클라이언트가 있는 경우에도 유연한 확장 가능
- 인터페이스를 적절하게 분리하고 확장하는 방법을 통해 오브젝트 사이의 의존관계를 명확히 해주고, 기존 의존관계에 영향을 주지 않으면서 유연한 확장성을 얻는 방법에 대해 고민해야 함
- DI와 객체지향 설계는 서로 밀접한 관계를 맺고 있음
