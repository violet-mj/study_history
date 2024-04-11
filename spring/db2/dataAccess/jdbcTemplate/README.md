- JdbcTemplate는 JDBC를 편리하게 사용하도록 도와준다.

**장점**
- 설정이 편리함
	- JdbcTemplate은 `spring-jdbc`라이브러리에 포함되어 있는데 JDBC를 사용할 때 기본으로 사용되는 라이브러리
- 반복 문제 해결
	- 템플릿 콜백 패턴을 사용하여 반복 작업을 대신 해결
		- 커넥션 획득
		- `statement`를 준비하고 실행
		- 커넥션 종료, `statement`, `resultSet`종료
		- 트랜잭션을 다루기 위한 커넥션 동기화
		- 예외 발생시 스프링 예외 변환기 실행

**단점**
- **동적 SQL을 해결하기 어렵다**

# 구현
### 1. [JdbcTemplateRepositoryV1](./v1.md)
- `JdbcTemplate`
- JdbcTemplate을 활용하여 반복되는 Jdbc 코드, 트랜잭션 동기화를 제거할 수 있었다.
- 하지만 SQL을 작성하는 과정에서 파라미터를 **순서**대로 넣는 과정에서 실수할 가능성이 크다.
### 2. [JdbcTemplateRepositoryV2](./v2.md)
- `NamedParameterJdbcTemplate`
- 컬럼명으로 파라미터를 주입해주면서 SQL 파라미터를 넣는 것에 대한 실수가 줄어든다.
### 3. [JdbcTemplateRepositoryV3](./v3.md)
- `SimpleJdbcInsert`
- Insert를 편리하게 사용할 수 있는 util

**JdbcTemplate**

**단건 조회**
```java
int rowCount = jdbcTemplate.queryForObject("select count(*) from item");
```

**단건 조회 - 문자 조회**
```java
String lastName = jdbcTemplate.queryForObject("select last_name from t_actor where id = ?");
```

**단건 - 객체 조회**
```java
Actor actor = jdbcTemplate.queryForObject( "select first_name, last_name from t_actor where id = ?", (resultSet, rowNum) -> { Actor newActor = new Actor(); newActor.setFirstName(resultSet.getString("first_name")); newActor.setLastName(resultSet.getString("last_name")); return newActor; }, 1212L);
```
- 객체를 하나 조회 할때는 객체를 **매핑**해야하므로 `RowMapper`를 사용해야한다. 

**목록 조회 - 객체**
```java
private final RowMapper<Actor> actorRowMapper = (resultSet, rowNum) -> { Actor actor = new Actor(); actor.setFirstName(resultSet.getString("first_name")); actor.setLastName(resultSet.getString("last_name")); return actor; }; public List<Actor> findAllActors() { return this.jdbcTemplate.query("select first_name, last_name from t_actor", actorRowMapper); }
```
- 여러 row를 조회할 때는 `query()`를 사용하면 된다. 결과를 리스트로 반환한다.
- 결과를 객체로 매핑해야하므로 `RowMapper`를 사용해야한다.

**임의의 SQL 실행**
- `execute()`를 사용하면 된다.









