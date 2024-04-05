
#### 체크 예외와 인터페이스
- 서비스 계층은 가급적 기존 기술에 의존하지 않고 순수하게 유지해야한다.

#### 인터페이스 도입
![](/spring/static/png/repositoryInterface.png)
- 이렇게 인터페이스를 도입하면 `MemberService`는 `MemberRepository`인테페이스에만 의존하면 된다.
- 구현 기술을 변경하고 싶으면 dI를 이용해서 `MemberService`코드의 변경 없이 구현 기술을 변경할 수 있다.

**MemberRepository 인터페이스**
```java
public interface MemberRepository {
	Member save(Member member);
	Member findById(String memberId);
	void update(String memberId, int money);
	void delete(String memberId);
}
```

**체크 예외 코드에 인터페이스 도입 시 문제점**
- 인터페이스의 구현체가 체크 예외를 던지려면, 인터페이스 메서드에 체크 예외를 던지는 부분이 선언되어 있어야한다.
- 구현 클래스의 메서드에 선언할 수 있는 예외는 부모 타입에서 던진 예외와 같거나 하위 타입이된다.

**잘못된 인터테이스 예제**
```java
public interface MemberRepository {
	Member save(Member member) throws SQLException;
	Member findById(String memberId) throws SQLException;
	void update(String memberId, int money) throws SQLException;
	void delete(String memberId) throws SQLException;
}
```
- 구현 기술을 쉽게 변경하기 위해서 인터페이스를 도입하는 것인데 인터페이스가 `SQLException`과 같은 구현 기술에 의존해버리면 인터페이스의 목적 자체가 오염되어 버린다.
- 언체크 예외를 사용하여 이러한 인터페이스를 수정하자

```java
/**  
 * 예외 누수 문제 해결  
 * SQLException 제거  
 * MemberRepository 인터페이스에 의존  
 */  
@Slf4j  
public class MemberServiceV4 {  
  
  private final MemberRepository memberRepository;  
  
  public MemberServiceV4(MemberRepository memberRepository) {  
    this.memberRepository = memberRepository;  
  }  
  
  @Transactional  
  public void accountTransfer(String fromId, String toId, int money) throws SQLException {  
    bisLogic(fromId, toId, money);  
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
}
```
- `MemberRepository` 인터페이스에 의존하도록 코드 변경했다.
- `MemberRepositoryV3_3`와 비교해서 보면 메서드에서 `throws SQLException`을 제거하였다.

**정리**
- 체크 예외를 런타임 예외로 변환하여 인터페이스와 서비스 계층의 순수성을 유지할 수 있었다.
- 덕분에 향후 JDBC에서 다른 구현 기술로 변경하더라도 서비스 계층의 코드를 변경하지 않고 유지할 수 있다

**문제점**
- 리포지토리에서 넘어오는 특정한 예외는 복구 시도 가능하다. 그러나 `MyDbException`과 같은 런타임 예외만 넘어오기 때문에 예외 구분이 불가능하다. 예외를 잡아서 복구하고 싶다면 어떻게 할까?

#### 데이터 접근 예외 직접 만들기
- 데이터 오류에 따라서 특정 예외는 복구하고 싶을 때
![](/spring/static/png/sqlexceptioncode.png)
- `SQLException`내부에 들어있는 `errorCode`를 활용하면 데이터베이스에서 어떤 문제가 발생했는지 알 수 있다.
- 데이터베이스마다 코드가 다르니 주의


하지만 `SQLException`을 서비스에 넘겨서 이를 해결하는 것은 다시 JDBC 기술에 의존하게 되면서 서비스 계층의 순수성이 무너진다.
- 리포지토리에서 예외를 변환해서 던지자! `SQLException` -> `MyDuplicateKeyException`

```java
public class MyDuplicatekeyException extends MyDbException{  
  public MyDuplicatekeyException() {  
  }  
  
  public MyDuplicatekeyException(String message) {  
    super(message);  
  }  
  
  public MyDuplicatekeyException(String message, Throwable cause) {  
    super(message, cause);  
  }  
  
  public MyDuplicatekeyException(Throwable cause) {  
    super(cause);  
  }  
}
```
- `MyDbException`을 상속받아 의미 있는 계층 형성

```java
public void create(String memberId) {  
  try {  
    repository.save(new Member(memberId, 0));  
    log.info("saveId={}", memberId);  
  } catch (MyDuplicatekeyException e) {  
    log.info("키 중복, 복구 시도");  
    String retryId = generateNewId(memberId);  
    repository.save(new Member(retryId, 0));  
    log.info("resaveId={}", retryId);  
  } catch (MyDbException e) {  
    log.info("데이터 접근 계층 예외");  
  }
```
- 저장을 시도하고 `MyDuplicateKeyException`이 올라오면 이 예외를 잡는다.
- 예외를 잡아서 새로운 아이디 생성을 시도한다.
- 복구할 수 없는 예외(`MyDbException`)면 로그를 남기고 예외를 던진다.

#### 문제점
- 데이터베이스마다 SQL `ErrorCode`는 다르다
- 데이터베이스가 전달하는 오류는 키 중복 뿐만 아니라 락이 걸린 경우, 문법 오류인 경우 다양하다 모든 솽황에 맞는 예외를 만들어야 할까?
![](/spring/static/png/springExceptionStructure.png)
- 스프링은 데이터 접근 계층에 대한 예외를 정리해서 일관된 예외 계층 제공
- 예외의 최고 계층은 `org.springframework.dao.DataAccessException`.
	- `RuntimeException`을 상속받기 때문에 모든 예외는 런타임 예외이다.
- `DataAccessException`은 `NonTransient`와 `Transient`예외이다.
	- `Transient`는 일시적이라는 뜻이며, 동일한 SQL을 다시 시도했을 때 성공할 가능성이 있다.
		- 쿼리 타임아웃, 락과 관련된 오류
	- `NonTransient`는 일시적이지 않다는 뜻.  동일한 SQL을 다시 시도하면 실패한다.
		- SQL 문법오류, 데이터베이스 제약 조건 위배

```java
@Test  
void exceptionTranslator() {  
  String sql = "select bad grammer";  
  
  try {  
    Connection con = dataSource.getConnection();  
    PreparedStatement stmt = con.prepareStatement(sql);  
    stmt.execute();  
  } catch (SQLException e) {  
    SQLErrorCodeSQLExceptionTranslator exTranslator = new SQLErrorCodeSQLExceptionTranslator(dataSource);  
    DataAccessException resultEx = exTranslator.translate("", sql, e);  
    log.info("resultEx", resultEx);  
    assertThat(resultEx.getClass()).isEqualTo(BadSqlGrammerException.class);
```
- `translate()`의 첫번째 파라미터는 읽을 수 있는 설명, 두번째는 sql 쿼리, 세번째는 발생한 에러이다.
	- 반환 타입은 최상위 타입인 `DataAccessException`이지만 실제로는 `BadSqlGrammerException`이 반환된다.
- 각각의 DB마다 SQL ErrorCode는 다르다. 그런데 스프링은 어떻게 각각의 DB가 제공하는 SQL ErrorCode까지 고려해서 예외를 반환할까?
	- `org.springframework.jdbc.support.sql-error-codes.xml`
	- 스프링 SQL 예외 반환기는 SQL ErrorCode를 이 파일에 대입해서 어떤 스프링 데이터 접근 예외로 전환해야 할지 찾아낸다.

```java
try {  
  con = getConnection();  
  pstmt = con.prepareStatement(sql);  
  pstmt.setString(1, memberId);  
  pstmt.executeUpdate();  
} catch (SQLException e) {  
  throw exTranslator.translate("delete", sql, e);  
} finally {  
  close(con, pstmt, null);  
}
```
- `translate`를 사용함으로써 서비스 계층은 특정 리포지토리의 구현 기술과 예외에 종속적이지 않게 되었다. 서비스 계층은 특정 구현 기술이 변경되어도 그대로 유지할 수 있다.
- 서비스 계층에서 예외를 복구하는 경우, 예외가 스프링에서 제공하는 데이터 접근 예외로 변경 되어 서비스 계층에 넘어오기 때문에 필요한 경우 예외를 잡아서 복구하면 된다.

#### JDBC Template
```java
private final JdbcTemplate template;  
  
@Autowired  
public MemberRepositoryV5(DataSource dataSource)  {  
  this.template = new JdbcTemplate(dataSource);  
}  
  
public Member save(Member member) {  
  String sql = "insert into member(member_id, money) values (?, ?)";  
  template.update(sql, member.getMemberId(), member.getMoney());  
  return member;  
}  
  
public Member findById(String memberId) {  
  String sql = "select * from member where member_id = ?";  
  return template.queryForObject(sql, memberRowMapper(), memberId);  
}
```

- `JdbcTemplate`으로 기존의 connection 동기화, 예외 처리(스프링) 등 모든 중복을 제거할 수 있었다.  