# 회고

## String, JDBCTemplate - query

### 1. 김지혜
<pre><code>
String str1 = new String("spring");
String str2 = new String("spring");
str1 == str2 // flase 개별 객체로 생성됨

String str1 = "spring";
String str2 = "spring";
str1 == str2 // ture string constant pool을 통하여 공유
</code></pre>
- 참조 : https://brunch.co.kr/@kd4/1

### 2. 조현우, 이해은, 서동현
- JDBC, Mybatis(mapper), JPA등 객체, 데이터 구분없이 들고 올 수 있다.
- 참조
  - https://www.java2novice.com/spring/jdbctemplate-single-query/
  - https://www.mkyong.com/spring/jdbctemplate-queryforint-is-deprecated/