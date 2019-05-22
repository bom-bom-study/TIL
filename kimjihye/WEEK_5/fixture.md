예를 들어,
아래의 테스트에는

```java
class UserServiceTest{
  List<User> users;   // 테스트 픽스처 리스트

  @Before
  public void setUp(){
    users = Arrays.asList(
              new User("user1","사용자1","p1",Level.BASIC, 49, 0);
              new User("user2","사용자2","p2",Level.BASIC, 50, 0);
              new User("user3","사용자3","p3",Level.SILVER, 60, 0);
              new User("user4","사용자4","p4",Level.SILVER, 60, 0);
              new User("user5","사용자5","p5",Level.GOLD, 49, 0);
            );
  }
}
```
테스트 픽스처 개수가 5개다.
