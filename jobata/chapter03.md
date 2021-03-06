# 3장 템플릿
- OCP(개방 폐쇄 원칙)
  - 확장에는 자유롭게 열려 있고 변경에는 굳게 닫혀 있다.
    - 코드에서 어떤 부분은 변경을 통해 그 기능이 다양해지려고 확장하려는 성질이 있고
    - 어떤 부분은 고정되어 있고 변하지 않으려는 성질이 있다.
  - 변화의 특성이 다른 부분을 구분해주고, 각각 다른 목적과 다른 이유에 의해 다른 시점에 독립적으로 변경될 수 있는 효율적인 구조를 만들어 주는 것
- 템플릿
  - 변경이 거의 일어나지 않으며 일정한 패턴으로 유지되는 특성을 가진 부분을 독립시켜서 효과적으로 활용할 수 있도록 하는 방법

## 3.1 다시 보는 초난감 DAO

#### 3.1.1 예외처리 기능을 갖춘 DAO
- JDBC 코드 예외처리의 필요성
  - DB 커넥션이라는 제한적 리소스를 공유해 사용하는 서버에서는 사용한 리소스를 반드시 반환하도록 만들어야 한다.
    - 서버에서는 제한된 개수의 DB 커넥션을 만들어서 재사용 가능한 풀로 관리한다.
    - DB 풀은 매번 getConnection()으로 가져간 커넥션을 명시적으로 close()해서 돌려줘야지만 다시 풀에 넣었다가 재사용할 수 있다.
    - 오류가 날 때마다 미처 반환되지 못한 Connection이 계쏙 쌓이면 커넥션 풀에 여유가 없어져 리소스가 모자라는 오류가 발생하며 서버가 중단될 위험이 있다.
  - JDBC 코드에서는 어떤 상황에서도 가져온 리소스를 반환하도록 try/catch/finally 구문 사용을 권장하고 있다.
    - try 블록
      - 예외가 발생할 가능성이 있는 코드를 묶어준다.
    - catch 블록
      - 예외가 발생했을 때 부가적인 작업을 해줄 수 있도록 한다.
    - finally 블록
      - try 블록에서 예외가 발생했을 때나 안 했을 때나 모두 실행된다.

## 3.2 변하는 것과 변하지 않는 것

#### 3.2.1 JDBC try/catch/finally 코드의 문제점
- 아직 한숨이 나오는 코드
  - try/catch/finally 블록 2중 중첩, 모든 메소드마다 반복
- 이런 코드를 효과적으로 다룰 수 있는 방법은 없을까?
  - 잘 분리해내는 작업
    - 변하지 않는, 그러나 많은 곳에서 중복되는 코드
    - 로직에 따라 자꾸 확장되고 자주 변하는 코드

#### 3.2.2 분리와 재사용을 위한 디자인 패턴 적용
- 어떻게 개선할까?
  - 변하는 성격이 다른 것을 찾기
    - 변하지 않는 부분
    - 변하는 부분
  - 변하는 부분을 변하지 않는 나머지 코드에서 분리한다.
  - 변하지 않는 부분을 재사용한다.
- 접근 방법
  - 메소드 추출
    - 변하지 않는 부분이 변하는 부분을 감싸고 있어 변하지 않는 부분을 추출하기가 어려웠다.
    - 그래서 반대로 변하는 부분을 메소드로 독립시켰는데 재사용을 못한다.
    - but, 이건 접근 방법이 잘못됐다.
  - 템플릿 메소드 패턴의 적용
    - 상속을 통해 기능을 확장해서 사용하는 패턴
    - 변하지 않는 부분
      - 슈퍼클래스에
    - 변하는 부분
      - 추상 메소드로 정의하여 서브클래스에서 오버라이드하여 새롭게 정의해 쓰도록하는 것
    - but, 단점
      - 가장 큰 문제는 DAO 로직마다 상속을 통해 새로운 클래스를 만들어야 한다는 점
        - ![image](https://user-images.githubusercontent.com/36880294/55453556-57df4f80-5617-11e9-9885-623c91662cf0.png)
      - 확장구조가 이미 클래스를 설계하는 시점에서 고정되어 버린다는 점
        - 이미 클래스 레벨에서 컴파일일 시점에 이미 그 관계가 결정되어 있다. 따라서 그 관계에 대한 유연성이 떨어져 버린다.
  - 전략 패턴의 적용
    - 오브젝트를 아예 둘로 분리하고 클래스 레벨에서는 인터페이스를 통해서만 의존하도록 만드는 패턴
      - 확장에 해당하는 변하는 부분을 별도의 클래스로 만들어 추상화된 인터페이스를 통해 위임하는 방식
      - ![image](https://user-images.githubusercontent.com/36880294/55453730-071c2680-5618-11e9-8c4e-abc2b8cfdc23.png)
    - 개방 폐쇄 원칙을 잘 지키며 템플릿 메소드 패턴보다 유연하고 확장성이 뛰어남
    - but, 컨텍스트 안에서 이미 구체적인 전략 클래스를 사용하도록 고정되어 있다면 뭔가 이상하다.
      - 컨텍스트가 인터페이스뿐 아니라 특정 구현 클래스를 직접 알고 있다는 건 전략패턴에도 OCP에도 잘 들어맞는다고 볼 수 없음.
  - DI 적용을 위한 클라이언트/컨텍스트 분리
    - 전략 패턴에 따르면 Context가 어떤 전략을 사용하게 할 것인가는 Context를 시용하는 앞단의 Client가 결정하는 게 일반적
      - Client가 구체적인 전략의 하나를 선택하고 오브젝트로 만들어서 Context에 전달
      - Context는 전달받은 그 Strategy 구현 클래스의 오브젝트를 시용
      - ![image](https://user-images.githubusercontent.com/36880294/55454046-5878e580-5619-11e9-94c8-88b159ae8c3b.png)
    - 관심사를 분리하고 유연한 확장관계를 유지하도록 만드는 작업은 매우 중요
- 마이크로 DI
  - 작은 위의 코드와 메소드 사이에서 일어나는 DI
  - DI의 장점을 단순화해서 IoC 컨테이너의 도움 없이 코드 내에서 적용한 경우
- DI
  - 의존관계 주입
  - 제3자의 도움을 통해 두 오브젝트 사이의 유연한 관계가 설정되도록 만드는 것
  - 형태
    - 의존관계에 있는 두 개의 오브젝트와 이 관계를 다이내믹하게 설정해주는 오브젝트 팩토리(DI 컨테이너), 그리고 이를 사용하는 클라이언트라는 4개의 오브젝트 사이에서 일어난다.
    - 원시적인 전략 패턴 구조를 따라 클라이언트가 오브젝트 팩토리의 책임을 함께 지고 있을 수도 있다
    - 클라이언트와 전략(의존 오브젝트)이 걸합될 수도 있다.
    - 클라이언트와 DI 관계에 있는 두 개의 오브젝트가 모두 하나의 클래스 안에 담길 수도 있다.

## 3.3 JDBC 전략 패턴의 최적화

#### 3.3.1 전략 클래스의 추가 정보
- 전략과 컨텍스트

#### 3.3.2 전략과 클라이언트의 동거
- 불만
  - DAO 메소드마다 새로운 StatementStrategy 구현 클래스를 만들어야 한다.
    - 런타임 시에 다이내믹하게 DI 해준다는 점을 제외하면 로직마다 상속을 사용하는 템플릿 메소드 패턴을 적용했을 때 보다 그다지 나을 게 없다.
  - DAO 메소드에서 StatementStrategy에 전달할 User와 같은 부가적인 정보가 있는 경우 이를 위해 오브젝트를 전달받는 생성자와 이를 저장해둘 인스턴스 변수를 번거롭게 만들어야 한다.
    - 이 오브젝트가 사용되는 시점은 컨텍스트가 전략 오브젝트를 호출할 때이므로 잠시라도 어딘가에 다시 저장해둘 수밖에 없다.
- 해결 방법
  - 로컬 클래스
    - StatementStrategy 전략 클 래스를 매번 독립된 파일로 만들지 말고 UserDao 클래스 안에 내부 클래스로 정의해버리는 것
    - 클래스가 내부 클래스이기 때문에 자신이 선언된 곳의 정보에 접근할 수 있다.
      - 번거롭게 생성자를 통해 오브젝트를 전달해줄 펼요가 없다.
      - 다만，내부 클래스에서 외부의 변수를 사용할 때는 외부 변수는 반드시 final로 선언해줘야 한다.
  - 익명 내부 클래스
    - 좀 더 간결하게 클래스 이름도 제거할 수 있다.
- 중첩 클래스(nested calss)
  - 스태틱 클래스(static class)
    - 독립적으로 오브젝트로 만들어질 수 있음
  - 내부 클래스(inner class)
    - 자신이 정의된 클래스의 오브젝트 안에서만 만들어질 수 있음
    - 범위에 따른 구분
      - 멤버 내부 클래스(member inner class)
        - 오브젝트 레벨에 정의
      - 로컬 클래스(local class)
        - 메소드 레벨에 정의
      - 익명 내부 클래스(anonymous inner class)
        - 선언된 위치에 따라 다름
        - 클래스를 재사용힐 필요가 없고. 구현한 인터페이스 타입으로만 사용할 경우에 유용하다.

## 3.4 컨택스트와 DI

#### 3.4.1 JdbcContext의 분리
- 컨텍스트 메소드를 클래스 밖으로 독립시켜서 모든 DAO가 사용할 수 있게 해보자.
  - 클래스 분리
  - 빈 의존관계 변경
- DI
  - 인터페이스를 사이에 둬서 클래스 레벨에서는 의존관계가 고정되지 않게 하고， 런타임 시에 의존할 오브젝트와의 관계를 다이내믹하게 주입해줘야 한다.

#### 3.4.2 JdbcContext의 특별한 DI
- 런타임 시에 DI 방식으로 외부에서 오브젝트를 주입해주는 방식을 사용하긴 했지만, 의존 오브젝트의 구현 클래스를 변경할 수는 없다.
  - 스프링 빈으로 DI
  - 코드를 이용하는 수동 DI

## 3.5 템플릿과 콜백
- 템플릿
  - 고정된 작업 흐름을 가진 코드를 재사용한다는 의미
  - 템플릿 메소드 패턴은 고정된 틀의 로직을 가진 템플릿 메소드를 슈퍼클래스에 두고， 바뀌는 부분을 서브클래스의 메소드에 두는 구조로 이뤄진다.
- 콜백
  - 실행되는 것을 목적으로 다른 오브젝트의 메소드에 전달되는 오브젝트
  - 파라미터로 전달되지만 값을 참조하기 위한 것이 아니라 특정 로직을 담은 메소드를 실행 시키기 위해 시용
  - 자바에선 메소드 자체를 파라미터로 전달할 방법은 없기 때문에 메소드가 담긴 오브젝트를 전달해야 한다. 그래서 펑셔널 오브젝트(functional object)라고도 한다.

#### 3.5.1 템플릿/콜백의 동작원리
- 템플릿/콜백의 특징
- JdbcContext에 적용된 템플릿/콜백

#### 3.5.2 편리한 콜백의 재활용
- 콜백의 분리와 재활용

#### 3.5.3 템플릿/콜백의 응용
- 테스트와 try/catch/finally
- 중복의 제거와 템플릿/콜백 설계
- 템플릿/콜백의 재설계

## 3.6 스프링의 JDBCTEMPLATE

#### 3.6.1 update()

#### 3.6.2 queryForInt()

#### 3.6.3 queryForObject()

#### 3.6.4 query()

#### 3.6.5 재사용 가능한 콜백의 분리

## 3.7 정리
- 3징에서는 예외처리와 안전한 리소스 반환을 보장해주는 DAO 코드를 만들고 이를 객체지향 설계 원리와 디자인 패턴, DI 등을 적용해서 깔끔하고 유연하며 단순한 코드로 만드는 방법을 살펴봤다.
  - JDBC와 같은 예외가 발생할 가능성이 있으며 공유 리소스의 반환이 필요한 코드는 반드시 try/catch/finally 블록으로 관리해야 한다.
  - 일정한 작업 흐름이 반복되면서 그중 일부 기능만 바뀌는 코드가 존재한다연 전략 패턴을 적용한다. 바뀌지 않는 부분은 컨택스트로 바뀌는 부분은 전략으로 만들고 인터페이스를 통해 유연하게 전략을 변경할 수 있도록 구성한다.
  - 같은 애플리케이션 안에서 여러 가지 종류의 전략을 다이내믹하게 구성하고 시용해야 한다 변 컨텍스트를 이용하는 클라이언트 메소드에서 직접 전략을 정의하고 제공하게 만든다.
  - 클라이언트 메소드 안에 익명 내부 클래스를 사용해서 전략 오브젝트를 구현하면 코드도 간결해지고 메소드의 정보를 직접 사용할 수 있어서 편리하다.
  - 컨텍스트가 하나 이상의 클라이언트 오브젝트에서 사용된다면 클래스를 분리해서 공유하도록만든다.
  - 컨텍스트는 별도의 빈으로 등록해서 DI 받거나 클라이언트 클래스에서 직접 생성해서 사용 한다. 클래스 내부에서 컨텍스트를 사용할 때 컨텍스트가 의존하는 외부의 오브젝트가 있다면 코드를 이용해서 직접 DI 해줄 수 있다.
  - 단일 전략 메소드를 갖는 전략 패턴이면서 익명 내부 클래스를 사용해서 매번 전략을 새로 만들어 사용하고, 컨텍스트 호출과 동시에 전략 DI를 수행하는 방식을 템플릿/콜백 패턴이라고한다.
  - 콜백의 코드에도 일정한 패턴이 반복된다면 콜백을 템플릿에 넣고 재활용히는 것이 편리하다.
  - 템플릿과 콜백의 타입이 다양하게 바뀔 수 있다면 제네릭스를 이용한다.
  - 스프링은 JDBC 코드 작성을 위해 JdbcTemplate을 기반으로 하는 다양한 템플릿과 콜백을 제공한다.
  - 템플릿은 한 번에 하나 이상의 콜백을 사용할 수도 있고, 하나의 콜백을 여러 번 호출할 수도있다.
  - 템플릿/콜백을 설계할 때는 템플릿과 콜백 사이에 주고받는 정보에 관심을 둬야 한다.