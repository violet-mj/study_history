
## 트랜잭션
- 데이터를 단순히 파일에 저장해도 되는데, 데이터베이스에 저장하는 이유는 트랜잭션을 지원하기 때문이다
- A의 5000원을 B에게 이체한다고 생각해보면
	1. A의 잔고는 5000원 감소한다.
	2. B의 잔고는 5000원 증가한다.
	- 1번은 성공했지만 2번은 실패했다면 심각한 문제가 발생한다
	- 2가지 둘다 성공해야 저장하고, 중간에 하나라도 실패하면 거래 전의 상태로 돌아간다.

- 모든 작업이 성공해서 데이터를 데이터베이스에 저장하는 것을 `Commit`이라 하고 하나라도 실패해서 돌아가는 것을 `Rollback`이라고 한다.

### ACID

**원자성**: 트랜잭션 내에서 실행한 작업은 마치 **하나의 작업인 것처럼 모두 성공하거나 모두 실패**해야한다
**일관성**: 모든 트랙잭션은 **일관성 있는 데이터 상태를 유지**해야한다.
**격리성**: 동시에 실행되는 트랜잭션들이 **서로에게 영향을 미치지 않도록 격리**한다.
**지속성**: 트랜잭션을 성공적으로 끝내면 그 결과가 **항상 기록**되어야 한다. 중간에 시스템에 문제가 발생해도 데이터베이스 로그 등을 사용해서 성공한 트랜잭션 내용을 복구해야한다.
- 트랜잭션 간에 격리성을 완벽히 보장하려면 트랜잭션을 거의 순서대로 실행해야한다. 이렇게 하면 도시 처리 성능이 매우 나빠진다. 이런 문제로 인해 ANSI 표준은 트랜잭션의 격리 수준을 4단계로 나누어 정의함

### Isolation level
- READ UNCOMMITED(커밋되지 않은 읽기)
	- commit되지 않은 데이터를 읽었을 때 만약 rollback이 일어나면 데이터가 다른 현상이 발생할 수 있다. 
- READ COMMITTED(커밋된 읽기)
	- Non-Repeatable read가 발생할 수 있다.
		-  한 트랜잭션 내에서 update연산이 발생했을 때 같은 쿼리 2번을 보냈을 때 다른 결과가 나올 수 있다.
		- update쿼리에 대해서는 한 트랜잭션 내에서 일관성을 보장하지 않는다.
- REPEATABLE READ(반복 가능한 읽기)
	- Phantom-Read가 발생할 수 있다.
		- insert나 delete연산이 발생했을 때 2가지 쿼리에 대해서 없어지거나 추가된 row를 읽을 수 있다.
		- insert, update에 대해 일관성을 보장하지 않는다.
- SERIALIZABLE(직렬화 가능)
[mysql doc](https://dev.mysql.com/doc/refman/8.0/en/innodb-transaction-isolation-levels.html)
- mysql의 InnoDB 경우 REATABLE READ가 기본 옵션이다.


### 데이터베이스 연결 구조와 DB 세션
![](/spring/static/png/connectionStructure.png)
- 클라이언트는 커넥션을 받아와서 커넥션을 맺게된다. db서버는 내부에 세션이라는 것을 만들고 해당 커넥션은 세션을 통해서 실행하게 된다.


#### 트랜잭션 사용법
- 데이터 변경 쿼리를 실행하고 결과를 반영하려면 `commit`을 호출하고, 결과를 반영하고 싶지 않으면 롤백 명령어인 `rollback`을 호출한다.
- **커밋을 호출하기 전까진 임시로 데이터를 저장**하는 것이다. 해당 사용자만 변경 사항을 볼 수 있고 다른 세션의 사람들은 볼 수 없다.
![](/spring/static/png/transaction1.png)
커밋하지 않은 데이터를 다른 요청이 조회한다면 무슨 문제가 발생할까?
- 커밋하지 않은 데이터가 보이면, 세션2는 데이터를 조회했을 때 회원2, 3이 보일 것이다. 하지만 세션 1이 롤백하면 데이터 정합성에 문제가 생긴다.
![](transaction2.png)
- commit을 하면 세션2도 변경된 결과를 볼 수 있다.
- 하지만 롤백 시 세션1이 변경한 모든 데이터가 처음 상태로 복구된다.


자동 커밋
- 각각의 쿼리 실행 직후에 자동으로 커밋을 호출한다.

```sql
set autocommit true;
insert into member(member_id, money) values ('data1', 10000); // 자동 커밋
insert into member(member_id, money) values ('data2', 20000); // 자동 커밋
```

`commit`, `rollback`을 직접 호출하면서 트랜잭션 기능을 제대로 수행하려면 자동 커밋을 끄고 수동 커밋을 사용해야한다.

수동 커밋
- 쿼리 실행 직후에 수동으로 커밋이나 롤백을 호출해야한다.

```sql
set autocommit false;
insert into member(member_id, money) values ('data1', 10000); // 자동 커밋
insert into member(member_id, money) values ('data2', 20000); // 자동 커밋
commit;
```

- 수동 커밋 모드로 설정하는 것을 트랜잭션을 시작한다고 표현한다.


### DB Lock


![](/spring/static/png/dblock1.png)
1. 세션 1은 트랜잭션을 시작한다.
2. 세션 1은 `memberA`의 `money`를 500으로 변경을 시도한다. 이 때 해당 row의 lock을 먼저 획득해야한다. 해당 로우에 락이 있으므로 획득한다.
3. 획득 후 해당 row에 update sql 수행

![](/spring/static/png/dblock2.png)
4. 세션 2가 트랜잭션을 시작.
5. 세션 2도 `memberA`의 `money`를 변경하려고 시도하지만. 먼저 락을 획득해야한다. 하지만 락이 없으므로 대기한다.
	- 무한정 대기하지는 않고 락 대기 시간을 넘어가면 락 타임아웃 오류가 발생

![](/spring/static/png/dblock3.png)
6. 세션 1이 커밋을 수행하고 커밋으로 트랜잭션이 종료 되었으므로 락도 반납한다.

![](/spring/static/png/dblock4.png)
7. 세션 2가 락을 획득하고 update sql를 수행한다.

![](/spring/static/png/dblock5.png)
8. 세션 2도 커밋 후 락을 반환한다.


### DB lock - lookup

**일반적인 조회는 락을 사용하지 않는다**
- 데이터베이스마다 다르지만, 보통 데이터를 조회 할 때는 락을 획득하지 않고 바로 데이터를 조회할 수 있다. 예를 들어서 세션1이 락을 획득하고 데이터를 변경하고 있어도, 세션2에서 데이터를 조회는 할 수 있다. 물론 데이터를 변경하려면 락이 필요하다.

**조회와 락**
- 데이터를 조회할 때도 락을 획득하고 싶다면. `select for update`구문을 사용하면 된다.
- 이렇게 하면 세션1이 조회 시점에 락을 가져가버리기 때문에 다른 세션에서 해당 데이터를 변경할 수 없다.
- 트랜잭션을 커밋하면 락을 반납

**조회 시점에 락이 필요한 시점**
- 트랜잭션 종료 시점까지 해당 데이터를 다른 곳에서 변경하지 못하도록 강제로 막아야할 때
- 돈과 관련된 매우 중요한 계산이어서 계산을 완료할 때 까지 `memberA`의 금액을 다른곳에서 변경하면 안된다.


### 적용
```java
@Test  
@DisplayName("정상 이체")  
void accountTransfer() throws SQLException {  
  // given  
  Member memberA = new Member(MEMBER_A, 10000);  
  Member memberB = new Member(MEMBER_B, 10000);  
  memberRepository.save(memberA);  
  memberRepository.save(memberB);  
  
  // when  
  memberService.accountTransfer(memberA.getMemberId(), memberB.getMemberId(), 2000);  
  
  // then  
  Member findMemberA = memberRepository.findById(memberA.getMemberId());  
  Member findMemberB = memberRepository.findById(memberB.getMemberId());  
  
  assertThat(findMemberA.getMoney()).isEqualTo(8000);  
  assertThat(findMemberB.getMoney()).isEqualTo(12000);  
}
```
- given
	- 데이터를 저장해서 테스트 준비
- when
	- 계좌이체 로직 실행
- then 
	- 계좌이체 실패. `memberA`의 돈만 2000원 줄어듦


### 적용 2

- 트랜잭션은 비즈니스 로직이 있는 서비스 계층에서 시작해야 한다. 비즈니스 로직이 잘못되면 해당 비즈니스 로직으로 인해 문제가 되는 부분을 함께 롤백해야하기 때문
- 트랜잭션을 시작하려면 커넥션이 필요함
- 애플리케이션에서 DB 트랜잭션을 사용하려면 **트랜잭션을 사용하는 동안 같은 커넥션을 유지**해야한다.
	- 같은 커넥션을 유지하려면 커넥션을 파라미터로 전달해서 같은 커넥션이 유지되도록 하는 것이다.
![](/spring/static/png/connectionStructure.png)
1. `con = getConnection()`과 같은 커넥션을 얻는 코드가 있으면 안된다.
2. `con`은 종료되면 안된다.

### 애플리케이션 적용

```java
@RequiredArgsConstructor  
@Slf4j  
public class MemberServiceV2 {  
  private final MemberRepositoryV1 memberRepository;  
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
    Member fromMember = memberRepository.findById(fromId);  
    Member toMember = memberRepository.findById(toId);  
  
    memberRepository.update(con, fromId, fromMember.getMoney() - money);  
    validate(toMember);  
    memberRepository.update(con, toId, toMember.getMoney() - money);  
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

`Connection con = dataSource.getConnection()`
- 트랜잭션을 시작하려면 커넥션이 필요하다.
`con.setAutoCommit(false)`
- 트랜잭션 시작하기 위해 자동 커밋을 꺼야한다.
`bizLogic(con, fromId, toId, money)`
- 트랜잭션이 시작된 커넥션을 전달하면서 비즈니스 로직을 수행한다.
- 이렇게 분리한 이유는 트랜잭션을 관리하는 로직과 실제 비즈니스 로직을 구분하기 위함이다.
`con.commit()`
- 비즈니스 로직이 정상 수행되면 트랜잭션을 커밋함
`con.rollback()`
- `catch(Ex)..` 을 사용해서 예외 발생 시 트랜잭션 롤백
`release(con)`
- `finally`를 이용하여 커넥션을 사용하고 종료한다. 하지만 `autocommit`이 `false`상태로 풀에 반환한다. 그렇기 때문에 autocommit을 `true`로 변환한 후 반환한다.
	- 풀을 사용하기 때문에 발생하는 것 driverManager처럼 커넥션을 끝내버리면 다시 커넥션을 생성하기 때문에 문제가 되지 않는다;.


**반드시 자원을 사용하고 나면 종료해주어야 한다**