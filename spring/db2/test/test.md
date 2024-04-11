# 테스트 - 데이터베이스 연동
- `@SpringBootTest`는 `@SpringBootApplication`을 찾는다.
	- 아래 정보를 설정으로 사용한다.
```java
@Import(JdbcTemplateV3Config.class)  
@SpringBootApplication(scanBasePackages = "hello.itemservice.web")  
public class ItemServiceApplication {
...
```

- `@SpringBootApplication`이 과거에는 `MemoryConfig.class`를 사용하다가 이제는 `JdbcTemplateV3Config.class`를 사용하도록 변경되었다. 따라서 테스트도 `JdbcTemplate`를 통해 실제 데이터베이스를 호출하게 된다.

```java
@Test  
void findItems() {  
    //given  
    Item item1 = new Item("itemA-1", 10000, 10);  
    Item item2 = new Item("itemA-2", 20000, 20);  
    Item item3 = new Item("itemB-1", 30000, 30);  
  
    itemRepository.save(item1);  
    itemRepository.save(item2);  
    itemRepository.save(item3);  
  
    //둘 다 없음 검증  
    test(null, null, item1, item2, item3);  
    test("", null, item1, item2, item3);  
  
    //itemName 검증  
    test("itemA", null, item1, item2);  
    test("temA", null, item1, item2);  
    test("itemB", null, item3);  
  
    //maxPrice 검증  
    test(null, 10000, item1);  
  
    //둘 다 있음 검증  
    test("itemA", 10000, item1);  
}
```
- 이 테스트는 실패한다. `MemoryConfig`와 다르게 기존 데이터베이스의 데이터가 누적되어 있기 때문이다.

**원인**
- 과거에 저장했던 데이터들이 모두 남아있다.
	- 이 데이터들이 현재 테스트에 영향을 미친다.
	- 빈 데이터베이스를 가정하고 테스트 했기때문에 문제가 생긴다.
- 테스트 전용 db가 필요하다

**테스트 실행**
- `findItems`를 단독으로 실행해보면
	- 처음에는 성공한다.
	- 그러나 다시 실행하면 실패함
- 데이터가 축적된다.
	- 다른 테스트에서 이미 데이터를 추가했기 때문에 테스트 데이터가 오염된 것이다.
- 테스트가 끝날 때마다 테스트에서 추가한 데이터를 삭제해야한다.

**원칙**
1. 테스트는 다른 테스트와 격리해야한다.
2. 테스트는 반복해서 실행할 수 있어야한다.


#### 데이터 롤백

**트랜잭션과 롤백 전략**
- 트랜잭션을 활용하여 테스트 후 데이터를 원래 상태로 돌리자

**`@Transactional`
- 스프링에서 로직이 성공적으로 수행되면 커밋하도록 동작한다
- 그러나 테스트에서 사용하면 테스트가 끝나면 트랜잭션을 자동으로 롤백시킨다.
- 테스트에서 트랜잭션을 실행하면 테스트 실행이 종료될 때 까지 테스트가 실행하는 모든 코드가 같은 트랜잭션 범위에 들어간다고 이해하면 된다.

**`@Commit`**
- 테스트가 끝난 후 강제로 커밋 시키고 싶으면 이 애노테이션을 사용하면 된다.


### 테스트 - 임베디드 모드 DB
- H2 데이터베이스는 JVM안에서 메모리 모드로 동작하는 특별한 기능을 제공한다. 
- DB를 애플리케이션에 내장해서 실행한다고 해서 임베디드 모드라고 부른다. 
- 애플리케이션이 종료되면 같이 종료되고 데이터도 사라진다.

```java
@Bean  
@Profile("test")  
public DataSource dataSource() {  
  log.info("메모리 데이터베이스 초기화");  
  DriverManagerDataSource dataSource = new DriverManagerDataSource();  
  dataSource.setDriverClassName("org.h2.Driver");  
  dataSource.setUrl("jdbc:h2:mem:db;DB_CLOSE_DELAY=-1");  
  dataSource.setUsername("sa");  
  dataSource.setPassword("");  
  return dataSource;  
}
```

- `@Profile("test")`
	- 프로필이 `test`인 경우에만 데이터소스를 스프링 빈으로 등록한다.
	- 테스트 케이스에서만 이 데이터 소스를 사용하겠다는 것
- `jdbc:h2:mem:db` : 데이터 소스를 만들 때 이렇게만 적으면 임베디드 모드로 동작하는 h2 데이터베이스를 사용할 수 있다.
- `DB_CLOSE_DELAY=-1`: 임베디드 모드에서는 데이터베이스 커넥션 연결이 모두 끊어지면 데이터베이스도 종료되는데, 그것을 방지하는 설정

- 하지만 이렇게 설정을 해도 테스트는 `Item`테이블이 없어서 예외를 뱉는다.
	- 실제로 임베디드 db에 테이블의 정의하지 않았다!!
	- `/test/resources/schema.sql`에 테이블을 정의하자

### 테스트 - 스프링 부트와 임베디드 모드

- 모든 설정 정보를 제거하자
```properties
spring.profiles.active=test  
logging.level.org.springframework.jdbc=debug
#spring.datasource.url=jdbc:h2:tcp://localhost/~/testcase
#spring.datasource.username=sa
```
- 스프링은 아무 설정 정보가 없으면 임베디드 db를 생성한다.


