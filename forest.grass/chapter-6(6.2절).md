# toby-spring

## 05월 14일

### keyword

#### AOP
- Aspect Oriented Programming
- 스프링의 대표적인 3가지 기술 중 하나이다.
- 대표적인예로는 선언적 트랜잭션 기능이다.

#### 트랜잭션 코드의 분리
- 서비스 추상화 기법을 적용해 트랜잭션 기술에 독립적으로 만들어줬지만 아직까지 코드에는 트랜잭션 코드들이 많이 남아 있다.
- 개선 할 수 있는 방법이 필요하다

#### 메서드 분리
```{.java}
public void upgradeLevels() throws Exception {
  TransactionStatus status = this.transactionManager
          .getTransaction(new DefaultTransactionDefinition());
  try {

    List<User> users = userDao.getAll();
    for(User user : users) {
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
- 트랜잭션 경계설정 코드와 비즈니스 로직코드로 구분되어 있는걸 알 수 있다.
- 트랜잭션 경계설정의 코드와 비즈니스 로직 코드간에 서루 주고 받는 정보가 없다는 점을 확인 할 수 있다.
- 이 두가지 코드는 성격이 다를 뿐 아니라 서로 주고 받는 것도 없는, 완벽하게 독립적인 코드다.
- 비즈니스 로직을 담당하는 코드가 트랜잭션의 시작과 종료 작업 사이에서 수행돼야 한다는 사항만 지켜지면 된다.
- 비즈니스 로직만 따로 뽑아 내서 사용한다.
```{.java}
public void upgradeLevels() throws Exception {
  TransactionStatus status = this.transactionManager
          .getTransaction(new DefaultTransactionDefinition());
  try {
    upgradelevelIsInternal();
    this.transactionManager.commit(status);
  } catch (Exception e) {
    this.transactionManager.rollback(status);
    throw e;
  }
}

private void upgradelevelIsInternal() {
    List<User> users = userDao.getAll();
    for(User user : users) {
      if (canUpgradeLevel(user)) {
        upgradeLevel(user);
      }
    }
}
```
- 트랜잭션 코드는 여전히 UserService에 남아 있다.

#### DI를 이용한 클래스 분리
- 트랜잭션 코드를 클래스 밖으로 뽑아 내자
- DI 적용을 이용한 트랜잭션 분리
- DI의 기본 아이디어는 실제 사용할 오브젝트의 클래스 정체는 감춘 채 인터페이스를 통해 간접으로 접근합는 것이다.
- 그 덕분에 구현 클래스는 얼마든지 외부에서 변경할 수 있다. 바로 이런 개념이 DI다
- UserService 인턴페이스를 두고 비즈니스 로직을 담담하는 구현체 클래스와 트랜잭션을 당담하는 구현체 클래스 두개로 분리 시켜보자.
![AOP-1](/forest.grass/img/AOP-1.png)
- 클라이언트가 UserService라는 인터페이스를 통해 사용자 관리 로직을 이용하려고 할 때 먼저 트랜잭션 담담하는 오브젝트가 사용돼서 트랜잭션을 관련된 작업을 진행시켜주자.
- 실세 잣용자 관리 로직을 담은 오브젝트가 이후에 호출돼서 비즈니스 로직에 관련된 작업을 수행하도록 만들자.

#### 트랜잭션 분리에 따른 테스트 수정
- Test에서는 @Autowired 어노테이션을 이용하여 빈을 주입 시켰다.
- UserService의 두개의 구현체가 생겼기 때문에 어떤걸 선택해 의존성을 주입 시킬까?
- @Autowired는 기본적으로 타입을 이용해 빈을 찾지만 만약 타입으로 하나의 빈을 결정할 수 없는 경우에는 필드 이름을 이용해 빈을 찾는다.
- 타입이 같은 구현체가 있다면 변수명과 동일한 빈 아이디를 갖는 빈을 주입 시켜준다.
- UserServcieImpl과 UserService로 분리 시켜 줬으니 기존에 있는 테스트 코드의 목오브젝트 주입과정을 수정 할 필요가 생겼다.
- test code 중 upgradeAllOrNothing() 테스트를 수정 할 필요가 생겼다.
- 기존에는 userServcie에서 하였지면 현재는 트랜잭션과 서비스로직을 클래스로 분리 시켰으니 트랜잭션 클래스에 추가적으로 빈을 주입시켜 검증하여야 한다.
```{.java}
@Test
	public void upgradeAllOrNothing() {
		TestUserService testUserService = new TestUserService(users.get(3).getId());
		testUserService.setUserDao(userDao);
		testUserService.setMailSender(mailSender);
		
		UserServiceTx txUserService = new UserServiceTx();
		txUserService.setTransactionManager(transactionManager);
		txUserService.setUserService(testUserService);
		 
		userDao.deleteAll();			  
		for(User user : users) userDao.add(user);
		
		try {
			txUserService.upgradeLevels();   
			fail("TestUserServiceException expected"); 
		}
		catch(TestUserServiceException e) { 
		}
		
		checkLevelUpgraded(users.get(1), false);
	}

  static class TestUserService extends UserServiceImpl {
		private String id;
		
		private TestUserService(String id) {  
			this.id = id;
		}

		protected void upgradeLevel(User user) {
			if (user.getId().equals(this.id)) throw new TestUserServiceException();  
			super.upgradeLevel(user);  
		}
	}
	
	static class TestUserServiceException extends RuntimeException {
	}
```

#### 트랜잭션 경계설정 코드 분리의 장점
- 첫째, 비즈니스 로직을 담당하고 있는 UserServiceImpl의 코드를 작성할 때는 트랜잭션과 같은 기술적인 내용에는 전혀 신경 쓰지 않아도 된다. 트랜잭션의 적용이 필요한지도 신경 쓰지 않아도 된다.
  - ex) 스프링의 JDBC, JTA 같은 로우레벨의 트랜잭션 API는 물론이고 스프링의 트랜잭션 추상화 API
- 둘째, 비즈니스 로직에 대한 테스를 손쉽게 만들어낼 수 있다

#### 고립된 단위 테스트
- 가장 편하고 좋은 테스트 방법은 가능한 한 작은 단위로 쪼개서 테스트하는 것이다.
- 작은 단위 테스타가 좋은 이유는 테스트가 실패햇을 때 그원인을 찾기 쉽기 때문이다.
- 테스트 단위가 작아야 테스트의 의도나 내용이 분명해지고, 만들기도 쉬워진다.
- 테스트 대상이 다른 오브젝트와 환경에 의존하고 있다면 작은 단위의 테스트가 주는 장점을 얻기 힘들다

#### 복잡한 의존관계 속의 테스트
- UserService를 동작하려면 세 가지 타입의 의존 오브젝트가 필요하다
- UserDao, MainSender, PlatforTransactionManager
- 세 오브젝트에 대하여 의존 관계를 갖고 있다.
- 따라서 세 가지 의존관계를 갖는 오브젝트들이 테스트가 진행되는 동안에 같이 실행된다.
- 더 큰 문제는 그 세가지 의존 오브젝트도 자신의 코드만 실행하고 마는 게 아니라는 점이다.
- UserService라는 테스트 대상이 테스트 단위인 것처럼 보이지만 사실은 그 뒤의 의존관계를 따라 등장하는 오브젝트와 서비스, 환경 등이 모두 합쳐져 테스트 대상이 되는 것이다.
- ex)UserService는 문제가 없는데도 누군가 UserDao의 코드를 잘못 수정해서, 그 오류 때문에 UserService의 테스트가 실패할 수 도 있다.
- DAO에서 복잡한 쿼리 또는 로직이 있다고 가정하에 UserService를 테스트 할 경우 배보다 배꼽이 더 커져버린다.

#### 테스트 대상 오브젝트 고립시키기
- 테스트의 대상이 환경이나, 외부 서버, 다른 클래스의 코드에 종속되고 영향을 받지 않도록 고립시킬 필요가 있다.
- 테스트를 위한 대역을 사용하자
- 테스트 스텁과 목 오브젝트를 이용하자
- 트랜잭션 로직과 서비스 로직을 분리 시켜 놨으니 서비스 로직을 검증하기 위해서는 DAO, MailSender만 빈으로 주입 시켜주면 된다.
- 두 의존 오브젝트를 목 오브젝트로 주입시켜 테스트 대상 오브젝트를 고립 시켜주자.
- 현재까지의 방식은 DB에 정보를 직접 수정하여 검색하여 검증하였다.
- 의존 오브젝트나 외부 서비스에 의존하지 않는 고립된 테스트 방식으로 만든 UserServiceImpl은 아무리 그 기능이 수행돼도 그 결과가 DB 등을 통해서 남지 않으니, 기존의 방법으로는 작업 결과를 검증하기 힘들다.
- 테스트 대상이 협력 오브젝트에게 어떤 요청을 했는지를 확인 하는 작업 추가하여 테스트를 검증하자.
- 테스트 대상과 협력 오브젝트 사이에서 주고받은 정보를 저장해둿다가, 테스트의 검증에 사용할 수 있게 하는 목 오브젝트를 만들자.

#### 고립된 단위 테스트 활용
- 기존 UserServiceTest의 upgradeLevels() 테스트
```{.java}
	@Test
	public void upgradeLevels() {
		userDao.deleteAll();
		for(User user : users) userDao.add(user);
		
		MockMailSender mockMailSender = new MockMailSender();  
		userService.setMailSender(mockMailSender);  
		
		userService.upgradeLevels();
		
		checkLevelUpgraded(users.get(0), false);
		checkLevelUpgraded(users.get(1), true);
		checkLevelUpgraded(users.get(2), false);
		checkLevelUpgraded(users.get(3), true);
		checkLevelUpgraded(users.get(4), false);
		
		List<String> request = mockMailSender.getRequests();  
		assertThat(request.size(), is(2));  
		assertThat(request.get(0), is(users.get(1).getEmail()));  
		assertThat(request.get(1), is(users.get(3).getEmail()));  
	}

  private void checkLevelUpgraded(User user, boolean upgraded) {
		User userUpdate = userDao.get(user.getId());
		if (upgraded) {
			assertThat(userUpdate.getLevel(), is(user.getLevel().nextLevel()));
		}
		else {
			assertThat(userUpdate.getLevel(), is(user.getLevel()));
		}
	}
```
- 위 테스트는 다섯 단계로 나뉜다.
  - 1.테스트 실행 중에 UserDao를 통해 가져올 테스트용 정보르 DB에 넣는다.
  - 2.메일 발송 여부를 확인하기 위해 MailSender 목 오브젝트르 DI 해준다.
  - 3.실제 테스트 대상인 userService의 메소드를 실행한다.
  - 4.결과가 DB에 반영됐는지 확인하기 위해서 UserDao를 이용해 DB에서 데이터를 가져와 결과를 확인한다.
  - 5.목 오브젝트를 통해 UserService에 의한 메일 발송이 있었는지를 확인하면 된다.

#### UserDao 목 오브젝트
- UserDao도 목오브젝트로 대체하여 사용해보자
- 목 오브젝트는 기본적으로 스텁과 같은 방식으로 테스트 대상을 통해 사용될 때 필요한 기능을 지원해줘야 한다.
- getAll()에 대해서는 스텁으로서, update()에 대해서는 목 오브젝트로서 동작하는 UserDao 타입의 테스트 대역이 필요하다.
```{.java}
static class MockUserDao implements UserDao { 
		private List<User> users;  
		private List<User> updated = new ArrayList(); 
		
		private MockUserDao(List<User> users) {
			this.users = users;
		}

		public List<User> getUpdated() {
			return this.updated;
		}

		public List<User> getAll() {  
			return this.users;
		}

		public void update(User user) {  
			updated.add(user);
		}
		
		public void add(User user) { throw new UnsupportedOperationException(); }
		public void deleteAll() { throw new UnsupportedOperationException(); }
		public User get(String id) { throw new UnsupportedOperationException(); }
		public int getCount() { throw new UnsupportedOperationException(); }
	}
```
- 사용하는 메서드 스텁과 목오브젝트로 구현
- 사용하지 않는 메서드 excetpion 처리
- getAll() 메소드가 호출되면 DB에서 가져온 것처럼 돌려주는 용도이다.
- update() 메소드를 실행하면서 넘겨준 업그레이드 대상 User 오브젝트를 저장해뒀다가 검증을 위해 돌려주기 위한 것이다.
```{.java}
@Test 
	public void upgradeLevels() throws Exception {
		UserServiceImpl userServiceImpl = new UserServiceImpl(); 
		
		MockUserDao mockUserDao = new MockUserDao(this.users);  
		userServiceImpl.setUserDao(mockUserDao);

		MockMailSender mockMailSender = new MockMailSender();
		userServiceImpl.setMailSender(mockMailSender);
		
		userServiceImpl.upgradeLevels();

		List<User> updated = mockUserDao.getUpdated();  
		assertThat(updated.size(), is(2));  
		checkUserAndLevel(updated.get(0), "joytouch", Level.SILVER); 
		checkUserAndLevel(updated.get(1), "madnite1", Level.GOLD);
		
		List<String> request = mockMailSender.getRequests();
		assertThat(request.size(), is(2));
		assertThat(request.get(0), is(users.get(1).getEmail()));
		assertThat(request.get(1), is(users.get(3).getEmail()));
	}

	private void checkUserAndLevel(User updated, String expectedId, Level expectedLevel) {
		assertThat(updated.getId(), is(expectedId));
		assertThat(updated.getLevel(), is(expectedLevel));
	}
```
- 테스트 대역 오브젝트를 이용해 완전히 고립된 테스트로 만들기 전에의 테스트의 대상은 스프링 컨테이너에서 @Autowired를 통해 가져온 UserService 타입의 빈이었다.
- 컨테이너에서 가져온 UserService 오브젝트는 DI를 통해서 많은 의존 오브젝트와 서비스, 외부 환경에 의존하고 있었다.
- 이제는 완전히 고립돼서 테스트만을 위해 독립적으로 동작하는 테스트 대상을 사용할 것이기 때문에 스프링 컨테이너에서 빈을 가져올 필요가 없다.
- upgradeLevels() 테스트만 있었다면 스프링의 테스트 컨텍스트를 이용하기 위해 도입한 @RunWith 등은 제거 할 수 있다.
- 사용자 정보를 모두 삭제하고 테스트용 사용자 정보를 DB에 등록하는 등의 번거로운 준비 작업은 필요 없다.
- 준비 해둔 MockUserDao 오벶ㄱ트를 사용하도록 수동 DI 해주기만 하면 된다. 이미 고립된 테스트가 가능하도록 MockMailSender도 수정자 메소드를 이용해 DI를 해주자.
- 이제 고립된 오브젝트를 테스트 하여 검증해보자

#### 테스트 수행 성능의 향상
- 테스트 수행시간이 이전보다 빨라졌다.
- 고립된 테스트를 하면 테스트가 다른 의존 대상에 영향을 받을 경우를 대비해 복잡하게 준비할 필요가 없을 뿐만 아니라, 테스트 수행 성능도 크게 향상된다.
- 테스트가 빨리 돌아가면 부담 없이 자주 테스트를 돌려볼 수 있다.

#### 단위 테스트와 통합 테스트
- 단위 테스트의 단위는 정하기 나름이다. 사용자 관리 기능 전체를 하나의 단위로 볼 수 도 있고 하나의 클래스나 하나의 메소드를 단위로 볼 수도 있다.
- 중요한 것은 하나의 단위에 초점을 맞춘 테스트라는 점이다.
- 토비에서 부르는 단위테스트란
  - 테스트 대상 클래스를 목 오브젝트 등의 테스트 대역을 이용해 의존 오브젝트나 외부의 리소스를 사용하지 않도록 고립시켜서 테스트하는 것
- 토비에서 부르는 통합테스트란
  - 반면에 두 개 이상의, 성격이나 계층이 다른 오브젝트가 연동하도록 만들어 테스트하거나, 또는 외부의 DB나 파일, 서비스 등의 리소스가 참여하는 테스트
  - 스프링의 테스트 컨텍스트 프레임워크를 이용해서 컨텍스트에서 생성되고 DI된 오브젝트를 테스트하는 것도 통합 테스트다.

#### 단위 테스트를 위한 가이드라인
- 항상 단위 테스트를 먼저 고려한다.
- 하나의 클래스나 성격과 목적이 같은 긴밀한 클래스 몇 개를 모아서 외부와의 의존관계를 모두 차단하고 필요에 따라 스텁이나 목 오브젝트 등의 테스트 대역을 이용하도록 테스트를 만든다.
- 외부 리소스를 사용해야만 가능한 테스트는 통합 테스트로 만든다.
- 단위 테스트로 만들기가 어려운 코드도 있다. 대표적인게 DAO다. DAO는 그 자체로 로직을 담고 있기보다는 DB를 통해 로직을 수행하는 인터페이스와 같은 역할을 한다. DB와 연동을 하여 테스트하자
- DAO 테스트는 DB라는 외부 리소스를 사용하기 때문에 통합 테스트로 분류된다. 하지만 코드에서 보자면 하나의 기능 단위를 테스트하는 것이기도 하다. DAO를 테스트를 통해 충분히 검증해두면, DAO를 이용하는 코드는 DAO 역할을 스텁이나 목 오브젝트로 대체해서 테스트할 수 있다.
- 여러 개의 단위가 의존관계를 가지고 동작할 때를 위한 통합 테스트는 필요하다. 다만, 단위 테스트를 충분히 거쳤다면 통합 테스트의 부담은 상대적으로 줄어든다.
- 단위 테스트를 만들기가 너무 복잡하다고 판단되는 코드는 처음부터 통합 테스트를 고려해본다.
- 스프링 테스트 컨텍스트 프레임워크를 이용하는 테스트는 통합 테스트다. 가능하면 스프링의 지원 없이 직접 코드 레벨의 DI를 사용하면서 단위 테스트를 하는 게 좋겠지만 스프링의 설정 자체도 테스트 대상이고, 스프링을 이용해 좀 더 추상적인 레벨에서 테스트해야 할 경우도 종 종 있다.

#### 테스트 주의점
- 단위 테스트와 통합 테스트 모두 개발자가 스스로 자신이 만든 코드를 테스트하기 위해 만드는 개발자 테스트다.
- TDD는 코드를 만들자마자 바로 테스트가 가능하다는 장점이 있다.
- 테스트를 코드를 작성한 후에 만드는 경우에도 강능한 한 빨리 작성하도록 해야 한다.
- 코드를 만들고 나서 오랜 시간이 지난 뒤에 작성하는 테스트는 테스트 대상 코드에 대한 이해가 떨어지기 때문에 불완전해지기 쉽고 작성하기도 번거롭다.
- 코드를 작성하면서 테스트는 어떻게 만들 수 있을까를 생각해보는 것은 좋은 습관이다.
- 테스는 다시 코드의 품질을 높여주고, 리팩토링과 개선에 대한 용기를 주기도 할 것이다.

#### 목 프레임워크
- 단위 테스트를 만들기 위해서는 스텁이나 목 오브젝트의 사용이 필수적이다.
- 대부분 의존 오브젝트를 필요로 하는 코드를 테스트하게 되기 때문이다.
- 테스트시 사용 될 목 오브젝트를 만드는 일의 큰 짐이 된다.
- 인터페이스의 모든 메서드 재정의와 검증이 가능한 메서드를 작성 해야 하기 때문이다.
- 번거로운 목 오브젝트를 편리하게 작성하도록 도와주는 다양한 목 오브젝트 지원 프레임워크가 있다.

#### Mockito 프레임워크
- 사용하기도 편리하고, 코드도 직관적이다.
- 목 프레임워크의 특징은 목 클래스를 일일이 준비해둘 필요가 없다.
- 간단한 메소드 호출만으로 다이내믹하게 특정 인터페이스를 구현한 테스트용 목 오브젝트를 만들수 있다.
- 사용방법
```{.java}
 //org.mockito.Matchers 클래스에 정의된 스태틱 메소드
 UserDao mockUserDao = mock(UserDao.class);
 
 //getAll() 메소드가 불려올 때 사용자 목록을 리턴하도록 스텁 기능을 추가
 when(mockUserDao.getAll()).thenReturn(this.users);

 //update() 호출이 있었는지 검증 하는 부분
 //update() 메소드가 두 번 호출됐는지 확인
 verify(mockUserDao, times(2)).update(any(User.class));
 
```
- 인터페이스를 구현한 클래스를 만들 필요도 없고 리턴 값을 생성자를 통해 넣어줬다가 메소드 호출 시 리턴하도록 코드를 만들 필요도 없다.
- 특정 메소드의 호출이 있었는지, 어떤 값을 가지고 호출했는지를 일일이 기록해뒀다가 반환하는 기능을 만들지 않아도 된다.
- 편리하게 작성된 메소드 몇 개로 목 오브젝트를 사용할 수 있게 해준다.

#### Mockito 목 오브젝트의 네 단계
- 1.인터페이스를 이용해 목 오브젝트를 만든다.
- 2.목 오브젝트가 리턴할 값이 있으면 이를 지정해준다. 메소드가 호출되면 예외를 강제로 던지게 만들 수도 있다.
- 3.테스트 대상 오브젝트에 DI해서 목 오브젝트가 테스트 중에 사용되도록 만든다.
- 4.테스트 대상 오브젝트를 사용한 후에 목 오브젝트의 특정 메소드가 호출됐는지, 어떤 값을 가지고 몇 번 호출 됐는지를 검증한다.
- 2번과 4번은 필요시에만 사용한다.

#### Mockito를 이용한 테스트 코드 작성
```{.java}
@Test
	public void mockUpgradeLevels() throws Exception {
		UserServiceImpl userServiceImpl = new UserServiceImpl();

		UserDao mockUserDao = mock(UserDao.class);	    
		when(mockUserDao.getAll()).thenReturn(this.users);
		userServiceImpl.setUserDao(mockUserDao);

		MailSender mockMailSender = mock(MailSender.class);  
		userServiceImpl.setMailSender(mockMailSender);

		userServiceImpl.upgradeLevels();

		verify(mockUserDao, times(2)).update(any(User.class));				  
		verify(mockUserDao, times(2)).update(any(User.class));
		verify(mockUserDao).update(users.get(1));
		assertThat(users.get(1).getLevel(), is(Level.SILVER));
		verify(mockUserDao).update(users.get(3));
		assertThat(users.get(3).getLevel(), is(Level.GOLD));

		ArgumentCaptor<SimpleMailMessage> mailMessageArg = ArgumentCaptor.forClass(SimpleMailMessage.class);  
		verify(mockMailSender, times(2)).send(mailMessageArg.capture());
		List<SimpleMailMessage> mailMessages = mailMessageArg.getAllValues();
		assertThat(mailMessages.get(0).getTo()[0], is(users.get(1).getEmail()));
		assertThat(mailMessages.get(1).getTo()[0], is(users.get(3).getEmail()));
	}
```