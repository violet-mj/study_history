
### 문제점

![](/spring/static/png/applicationStructure.png)

**순수한 서비스 계층**
- 가장 중요한 계층은 서비스 계층이다. ui나 db 접근 로직은 변경해도 비즈니스 로직은 최대한 그대로 유지해야한다.
- 이렇게 하려면 최대한 서비스 계층을 특정 기술에 종속적이지 않게 만들어야한다.
	- 계층을 나눈 이유도 서비스 계층을 최대한 순수하게 유지하기 위한 목적, 기술 적인 부분은 프레젠테이션 계층, 데이터 접근 계층이 가져간다.
	- 프레젠테이션 계층
		- 클라이언트가 접근하는 UI와 관련된 기술인 웹, 서블릿, HTTP같은 부분을 담당. 그래서 서비스 계층을 이런 UI와 관련된 기술로부터 보호. 
		- 예를 들어 HTTP API를 사용하다가 GRPC로 변경해도 프레젠테이션 계층의 코드만 변경하면 된다.
	- 데이터 접근 계층
		- 데이터를 저장하고 관리하는 기술 담당
		- JDBC, JPA와 같은 구체적인 데이터 접근 기술로부터 서비스 계층을 보호해준다.
		- 예를 들어, JDBC를 사용하다가 JPA로 변경해도 서비스 계층은 변경하지 않아도 된다.

**서비스 계층이 특정 기술에 종속되지 않게 구현하자**


#### 기존 코드의 문제점
```java
@RequiredArgsConstructor  
@Slf4j  
public class MemberServiceV2 {  
  private final MemberRepositoryV2 memberRepository;  
  private final DataSource dataSource;  
  public void accountTransfer(String fromId, String toId, int money) throws SQLException {  
  
    Connection con = dataSource.getConnection();  
  
    try {  
      con.setAutoCommit(false);  
      bisLogic(con, fromId, toId, money);  
      // 성공 시 커밋  
      con.commit();  
    } catch (Exception e) {  
      con.rollback();  
      throw new IllegalStateException(e);  
    } finally {  
      if(con != null) {  
        try {  
          release(con);  
        } catch (Exception e) {  
          log.info("error", e);  
        }  
      }  
    }  
  }  
  
  private void bisLogic(Connection con, String fromId, String toId, int money) throws SQLException {  
    Member fromMember = memberRepository.findById(con, fromId);  
    Member toMember = memberRepository.findById(con, toId);  
  
    memberRepository.update(con, fromId, fromMember.getMoney() - money);  
    validate(toMember);  
    memberRepository.update(con, toId, toMember.getMoney() + money);  
  }  
  
  private static void validate(Member toMember) {  
    if(toMember.getMemberId().equals("ex")) {  
        throw new IllegalStateException("송금 실패");  
    }  
  }  
  
  private static void release(Connection con) throws SQLException {  
    // close를 쓰면 풀로 돌아간다.  
    // autocommit이 false인 상태로 풀로 돌아간다.  
    con.setAutoCommit(true); // 커넥션 풀을 고려  
    con.close();  
  }  
}
```

- 트랜잭션은 비즈니스 로직이 있는 서비스 계층에서 시작하는 것이 좋다.
- 그러나 트랜잭션을 사용하기 위해서는 `javax.sql.DataSource`, `java.sql.Connection`, `java.sql.SQLException`과 같은 JDBC 기술에 의존해야한다. 결과적으로 비즈니스 로직보다 JDBC를 사용해서 트랜잭션을 처리하는 코드가 더 많다.

정리하면 3가지 문제가 있다
1. 트랜잭션 문제
	-  같은 트랜잭션을 유지하기 위해 커넥션을 파라미터로 넘겨야 한다.
1. 예외 누수 문제
	- 서비스 계층은 **순수**해야한다.
	- 서비스 계층은 특정 기술에 종속되지 않아야한다.
2. JDBC 반복 문제
	-  `try`, `catch`, `finally`와 같은 코드의 반복
#### **예외 누수**

- JDBC의 예외가 서비스 계층으로 전파
- `SQLException`을 잡아서 처리하거나 `throw`를 통해 밖으로 전파해야한다. 
- `SQLException`은 JDBC 전용 기술이다 기술이 바뀌면 그에 맞는 예외로 변경해야한다.

![](/spring/static/png/TransactionDiagram.png)
- 만약 JDBC에서 JPA로 바꾸기 위해서는 모든 `@Service`와 `@Repository`의 코드를 변경해야한다.
- 이를 해결해기 위해 추상화에 의존하게 코드를 수정하자

![](/spring/static/png/transactionAbsDiagram.png)

하지만 스프링은 이 추상체와 구현체 모두 제공한다.
![](/spring/static/png/transactionSpringDiagram.png)

```java
public interface PlatformTransactionManager extends TransactionManager {
	TransactionStatus getTransaction(@Nullable TransactionDefinition definition)
	throws TransactionException;
	void commit(TransactionStatus status) throws TransactionException;
	void rollback(TransactionStatus status) throws TransactionException;
}
```

- `getTransaction()` 트랜잭션을 시작한다.
	- 이미 진행중인 트랜잭션이 있는 경우 해당 트랜잭션에 참여할 수 있기 때문이다.
- `commit()` 트랜잭션을 커밋
- `rollback()` 트랜잭션을 롤백한다.

#### 트랜잭션 동기화

스프링이 제공하는 트랜잭션은 2가지 역할을 한다.
1. 트랜잭션 추상화
2. 리소스 동기화

**리소스 동기화**
- 트랜잭션을 유지하려면 트랜잭션의 시작부터 끝까지 같은 데이터베이스 커넥션을 유지해야한다. 결국 같은 커넥션을 동기화하기 위해서 이전에는 파라미터로 커넥션을 전달하는 방법을 사용했다.
	- 파라미터로 커넥션을 전달하는 방법은 코드가 지저분해진다.
	-  커넥션을 넘기는 메서드와 넘기지 않는 메서드를 중복해서 만들어야한다.

![](/spring/static/png/transactionManager.png)

- 스프링은 **트랜잭션 동기화 매니저**를 제공한다. 이것을 쓰레드 로컬(`ThreadLocal`)을 사용해서 커넥션을 동기화해준다. 트랜잭션 매니저는 내부에서 이 트랜잭션 동기화 매니저를 사용한다.
- 트랜잭션 동기화 매니저는 쓰레드 로컬을 사용하기 때문에 멀티쓰레드 상황에 안전하게 커넥션을 동기화 할 수 있다. 따라서 커넥션이 필요하면 트랜잭션 동기화 매니저를 통해 커넥션을 획득하면 된다. 따라서 이전처럼 파라미터로 커넥션을 전달하지 않아도 된다.

**동작 방식**
1. 트랜잭션을 시작하려면 커넥션이 필요함. 트랜잭션 매니저는 데이터소스를 통해 커넥션을 만들고 트랜잭션을 시작한다.
2. 트랜잭션 매니저는 트랜잭션이 시작된 커넥션을 트랜잭션 동기화 매니저에 보관한다.
3. 레포지토리는 트랜잭션 동기화 매니저에 보관된 커넥션을 꺼내서 사용한다. 따라서 파라미터로 커넥션을 전달하지 않아도 된다.
4. 트랜잭션이 종료되면 트랜잭션 매니저는 트랜잭션 동기화 매니저에 보관된 커넥션을 통해 트랜잭션을 종료하고, 커넥션도 닫는다.

#### **구현**

```java
private Connection getConnection() throws SQLException {  
  // 주의!!! 트랜잭션 동기화를 사용하려면 DataSourceUtils를 사용해야 한다.  
  Connection connection = DataSourceUtils.getConnection(dataSource);  
  log.info("get connection={}, class={}", connection, connection.getClass());  
  return connection;  
}
```
- 그림 참조! **트랜잭션 동기화 매니저**에서 커넥션을 꺼내는 과정이다. `DataSourceUtils.getConnection()`은
	- 동기화 매니저가 커넥션을 가지고 있으면 커넥션을 꺼낸다.
	- 동기화 매니저가 커넥션을 조회할 수 없으면 커넥션을 생성해서 반환한다.

```java
// 트랜잭션 시 잘못된 코드
private void close(Connection con, Statement stmt, ResultSet rs) throws SQLException {  
  JdbcUtils.closeResultSet(rs);  
  JdbcUtils.closeStatement(stmt);  
  // *** 주의 *** 트랜잭션 사용시 커넥션을 종료하면 안된다.
  JdbcUtils.closeConnection(con);  
}

// 올바른 예시
private void close(Connection con, Statement stmt, ResultSet rs) throws SQLException {  
  JdbcUtils.closeResultSet(rs);  
  JdbcUtils.closeStatement(stmt);  
  // 아래 코드를 사용하여 
  DataSourceUtils.releaseConnection(con, dataSource);  
}
```

  `DataSourceUtils.releaseConnection(con, dataSource)`은 커넥션을 바로 닫는 것이 아니다.
  - **트랜잭션을 사용하기 위해 동기화된 커넥션은 커넥션을 닫지 않고 그대로 유지한다**
  - 트랜잭션 동기화 매니저가 관리하는 커넥션이 없는 경우 해당 커넥션을 닫는다.

```java
@Slf4j  
@RequiredArgsConstructor  
public class MemberServiceV3_1 {  
  
  private final PlatformTransactionManager transactionManager;  
  private final MemberRepositoryV3 memberRepository;  
  
  public void accountTransfer(String fromId, String toId, int money) throws SQLException {  
  
    TransactionStatus status = transactionManager.getTransaction(new DefaultTransactionDefinition());  
  
    try {  
      bisLogic(fromId, toId, money);  
      transactionManager.commit(status); // 성공 시 커밋  
    } catch (Exception e) {  
      transactionManager.rollback(status); // 실패 시 롤백  
      throw new IllegalStateException(e);  
    }  
  }  
  
  private void bisLogic(String fromId, String toId, int money) throws SQLException {  
    Member fromMember = memberRepository.findById(fromId);  
    Member toMember = memberRepository.findById(toId);  
  
    memberRepository.update(fromId, fromMember.getMoney() - money);  
    validate(toMember);  
    memberRepository.update(toId, toMember.getMoney() + money);  
  }  
  
  private static void validate(Member toMember) {  
    if(toMember.getMemberId().equals("ex")) {  
      throw new IllegalStateException("송금 실패");  
    }  
  }  
  
  private static void release(Connection con) throws SQLException {  
    // close를 쓰면 풀로 돌아간다.  
    // autocommit이 false인 상태로 풀로 돌아간다.  
    con.setAutoCommit(true); // 커넥션 풀을 고려  
    con.close();  
  }  
}
```

  `private final PlatformTransactionManager transactionManager`
  - 트랜잭션 매니저를 주입 받는다. 지금은 JDBC를 사용하기 때문에 `DataSourceTransactionManager` 구현체를 주입 받아야 한다.

`transactionManager.getTransaction()`
- 트랜잭션을 시작한다.
- `TransactionStatus`를 반환한다. 현재 트랜잭션의 상태 정보가 포함되어 있다. 이후 트랜잭션을 커밋, 롤백할 때 필요

`new DeafaultTransactionDefinition()`
- 트랜잭션과 관련된 옵션을 지정할 수 있다.

`transactionManager.commit(status)`
- 트랜잭션이 성공하면 이 로직 호출

`transactionManager.rollback(status)`
- 문제가 발생하면 이 메서드 호출하여 트랜잭션 롤백.

**위 두 메소드 호출 시 release가 자동으로 호출**


#### 트랜잭션 매니저 2

![](/spring/static/png/transactionManager2.png)
1. 서비스 계층에서 `transactionManager.getTransaction()`을 호출해서 트랜잭션을 시작한다.
2. 트랜잭션을 시작하려면 먼저 데이터베이스 커넥션이 필요하다. 트랜잭션 매니저는 내부에서 데이터소스를 사용해서 커넥션을 생성한다.
3. 커넥션을 수동 커밋 모드로 변경해서 실제 데이터베이스 트랜잭션을 시작한다.
4. 커넥션을 트낼잭션 동기화 매니저에 보관한다.
5. 트랜잭션 동기화 매니저는 쓰레드 로컬에 커넥션을 보관한다. 따라서 멀티 쓰레드 환경에 안전하게 커넥션을 보관할 수 있다.

![](/spring/static/png/transactionManager3.png)
6. 서비스는 비즈니스 로직을 실행하면서 리포지토리의 메서드들을 호출한다. **커넥션을 파라미터로 전달하지 않음**
7. 리포지토리 메서드들은 트랜잭션이 시작된 커넥션이 필요하다. 리포지토리는 `DataSourceUtils.getConnection()`을 사용해서 트랜잭션 동기화 매니저에 보관된 커넥션을 꺼내서 사용한다. 커넥션, 트랜잭션이 유지
8. 커넥션을 이용해 sql을 데이터베이스에 전달하여 실행

![](/spring/static/png/transactionManager4.png)
9. 비즈니스 로직이 끝나고 트랜잭션을 종료한다. 트랜잭션은 커밋하거나 롤백하면 종료한다.
10. 트랜잭션을 종료하려면 동기화된 커넥션이 필요하다. 트랜잭션 동기화 매니저를 통해 동기화된 커넥션을 획득한다.
11. 획득한 커넥션을 통해 트랜지션을 커밋하거나 롤백한다.
12. 전체 리소스 정리
	- 트랜잭션 동기화 매니저를 정리한다. 쓰레드 로컬은 사용 후 꼭 정리하자
	- `con.setAutoCommit(true)`로 되돌린다. 커넥션 풀 고려!!
	- `con.close()`를 호출해서 커넥션 종료한다. 커넥션 풀을 사용하는 경우 `con.close()`를 호출하면 커넥션 풀에 반환된다.


#### 트랜잭션 템플릿

```java
TransactionStatus status = transactionManager.getTransaction(new DefaultTransactionDefinition());  
  
try {  
  bisLogic(fromId, toId, money);  
  transactionManager.commit(status); // 성공 시 커밋  
} catch (Exception e) {  
  transactionManager.rollback(status); // 실패 시 롤백  
  throw new IllegalStateException("송금을 실패하였습니다.");  
}
```
- 트랜잭션을 시작하고, 비즈니스 로직을 실행하고, 성공하면 커밋하고 , 예외가 발생해서 실패하면 롤백한다.
-  다른 서비스에서 트랜잭션을 시작하려면 `try`, `catch`, `finally`를 포함한 성공시 커밋, 실패시 롤백 코드가 반복될 것이다.
- 이런 형태는 각각의 서비스에서 반복 된다. 달라지는 부분은 비즈니스 로직 뿐이다.
- 리럴 때 템플릿 콜백 패넌을 활용하면 이런 반복 문제를 깔끔하게 해결할 수 있다.

### `TransactionTemplate`
```java
public class TransactionTemplate {
	private PlatformTransactionManager transactionManager;
	public <T> T excute(TransactionCallback<T> action) {};
	void executeWithoutResult(Consumer<TransactionStatus> action) {};
}
```
`execute`: 응답 값이 있을 때 사용
`executeWithoutResult`: 응답 값이 없을 때 사용한다.

```java
txTemplate.executeWithoutResult((status) -> {  
  try {  
    bisLogic(fromId, toId, money);  
  } catch(SQLException e) {  
    throw new IllegalStateException(e);  
  }  
});
```

- 위 코드로 트랜잭션 커밋, 롤백 코드가 사라졌다.
- 동작 과정은 다음과 같다
	- 비즈니스 로직이 정상 수행되면 커밋
	- 언체크 예외가 발생하면 롤백한다.
- 코드에서 예외를 처리하기 위해 `try~catch`를 사용했는데, `bisLogic()`을 사용하면 `SQLException`을 넘기는데, 람다에서 체크를 예외를 밖으로 던질 수 없기 때문에 예외로 바꾸어 던지도록 예외를 전환
**체크예외? 언체크예외?**

- 하지만 서비스 로직인데 비즈니즈 로직 뿐만 아니라 트랜잭션을 처리하는 기술 로직이 함께 포함되어 있다.
- 비즈니스 로직과 트랜잭션을 처리하는 기술 로직이 한 곳에 있으면 유지보수가 어려워진다.

#### 트랜잭션 - 트랜잭션 AOP 이해

이전
- 트랜잭션 추상화, 트랜잭션 템플릿을 적용했다.

문제점
- 트랜잭션 템플릿을 통해 트랜잭션을 처리하는 반복 코드는 해결했지만. 서비스 계층에 순수한 비즈니스 로직만 남기는 목표는 달성하지 못함

해결방법
- 스프링 AOP를 사용하자!

**프록시 도입 전**

![](/spring/static/png/transaction_AOP1.png)
- 프록시를 도입하기 전에는 서비스 로직에서 트랜잭션을 수행한다.

**프록시 도입 후**

![](/spring/static/png/transaction_AOP2.png)
- 프록시를 사용하면 트랜잭션을 처리하는 객체와 비즈니스 로직을 처리하는 서비스 객체를 명확하게 분리할 수 있다.

**스프링이 제공하는 트랜잭션 AOP**
- 스프링이 제공하는 AOP 기능을 사용하면 프록시를 매우 편리하게 적용할 수 있다.
- `@Aspect`, `@Advice`, `@Pointcut`을 사용해서 머리속으로 구상해보자
- 개발자는 트랜잭션 처리가 필요한 곳에 `@Transactional`을 붙여주면 된다. 스프링의 트랜잭션 AOP는 이 애노테이션을 인식해서 트랜잭션 프록시를 적용해준다.

```java
@SpringBootTest // 스프링을 하나 띄우고 스프링 빈을 전부 등록하고 테스트한다.  
class MemberServiceV3_3Test {  
  public static final String MEMBER_A = "memberA";  
  public static final String MEMBER_B = "memberB";  
  public static final String MEMBER_EX = "ex";  
  
  @Autowired  
  private MemberRepositoryV3 memberRepository;  
  
  @Autowired  
  private MemberServiceV3_3 memberService;  
  
  @TestConfiguration  
  static class TestConfig {  
    @Bean  
    DataSource dataSource() {  
      return new DriverManagerDataSource(URL, USERNAME, PASSWORD);  
    }  
  
    @Bean  
    PlatformTransactionManager transactionManager() {  
      return new DataSourceTransactionManager(dataSource());  
    }  
  
    @Bean  
    MemberRepositoryV3 memberRepository() {  
      return new MemberRepositoryV3(dataSource());  
    }  
  
    @Bean  
    MemberServiceV3_3 memberService() {  
      return new MemberServiceV3_3(memberRepository());  
    }  
  }
```

`@SpringBootTest`
- 스프링 AOP를 적용하려면 스프링 컨테이너가 필요하다. 이 애노테이션이 있으면 테스트시 스프링 부트를 통해 스프링 컨테이너를 생성한다. 그리고 테스트에서 `@Autowired`를 통해 스프링 컨테이너가 관리하는 빈들을 사용할 수 있다.
`@TestConfiguration`
- 테스트 안에서 내부 설정 클래스를 만들어서 사용하면서 이 애노테이션을 붙이면, 스프링 부트가 자동으로 만들어주는 빈들에 추가로 필요한 스프링 빈들을 등록하고 테스트를 수행할 수 있다.

`TestConfig`
- `DataSource`
	- 스프링에서 기본으로 사용할 데이터소스를 스프링 빈으로 등록한다. 추가로 트랜잭션 매니저에도 사용한다.
- `DataSourceTransactionManager`
	- 스프링이 제공하는 트랜잭션 AOP는 스프링 빈에 등록된 트랜잭션 매니저를 찾아서 사용하기 때문에 트랜잭션 매니저를 스프링 빈으로 등록해두어야 한다.

### **정리**
![](/spring/static/png/transactionAopSummary.png)

**`SQLException`이 아직 종속되어 있다** 

