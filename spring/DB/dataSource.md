커넥션을 얻는 방법은 JDBC `DriverManager`를 사용하거나, 커넥션 풀을 사용하는 방법이 존재
![failed](/Excalidraw/png/driverManager.png)

- driverManager로 커넥션을 획득하다가 커넥션 풀을 사용하는 방법으로 변경하려면 어떻게 해야할까?
	- **커넥션을 획득하는 방법을 추상화**

### dataSource
![](/Excalidraw/png/dataSource.png)
- 자바는 `jakarta.sql.DataSource` interface를 제공
- `DataSource`는 **커넥션을 획득 하는 방법을 추상화**하는 인터페이스
- `DataSource`는 **커넥션 조회**를 위한 인터페이스
##### 주의
- `DriverManager`는 `DataSource`인터페이스를 사용하지 않는다 `DriverManager`는 직접 사용해야한다.
	- Spring은 `DriverManagerDataSource`라는 `DataSource`를 구현한 클래스를 제공한다.

**DriverManager**를 사용한 방법
```java
// 호출할 때마다 URL, USERNAME, PASSWORD가 필요함  

@Test  
void driverManager() throws SQLException {  
  // 커넥션을 획득할 때마다 설정 정보가 필요하다.
  Connection con1 = DriverManager.getConnection(URL,USERNAME,PASSWORD);  
  Connection con2 = DriverManager.getConnection(URL,USERNAME,PASSWORD);  
  log.info("con1 = {}, con2 = {}", con1, con2);  
}
```

**DataSource**를 사용한 방법
```java
// 인터페이스를 통해 커넥션을 획득  
@Test  
void dataSourceDriverManager() throws SQLException {  
  DriverManagerDataSource dataSource = new DriverManagerDataSource(URL,USERNAME,PASSWORD);  
  useDataSource(dataSource);  
}  
  
private void useDataSource(DataSource dataSource) throws SQLException {  
  Connection conn1 = dataSource.getConnection();  
  Connection conn2 = dataSource.getConnection();  
  log.info("connection = {}, class = {}", conn1, conn1.getClass());  
  log.info("connection = {}, class = {}", conn2, conn2.getClass());  
}
```
- 처음에만 URL, USERNAME, PASSWORD를 넘겨주고 이후 커넥션을 획득할 때는 단순히 `dataSource.getConnection()`을 입력하면 된다.

> **설정과 사용의 분리**
- 설정
	- `DataSource`를 만든고 필요한 속성들을 사용해서 URL, USERNAME, PASSWORD 같은 부분을 입력하는 것을 말함
		- 이를 통해 속성들은 한 곳에 존재하게 된다. 유연하게 대처 가능하다
- 사용
	- 설정은 신경쓰지 않고 `dataSource.getConnection()`만을 사용하면 된다.

- URL, USERNAME, PASSWORD에 의존하지 않고 커넥션을 얻을 수 있다. 
- 객체를 **설정**하는 부분과 **사용**하는 부분을 분리할 수 있다.


#### Connection Pool

```java
@Test  
void dataSourceConnectionPool() throws SQLException, InterruptedException {  
  // 커넥션 풀링  
  HikariDataSource dataSource = new HikariDataSource();  
  dataSource.setJdbcUrl(URL);  
  dataSource.setUsername(USERNAME);  
  dataSource.setPassword(PASSWORD);  
  dataSource.setMaximumPoolSize(10);  
  dataSource.setPoolName("MyPool");  
  useDataSource(dataSource);  
  // 쓰레드가 끝나면 useDataSource의 로그가 보이지 않는다.
  Thread.sleep(1000);  
}
```

```text
com.zaxxer.hikari.pool.HikariPool -- MyPool - Pool stats (total=4, active=2, idle=2, waiting=0)
23:05:07.411 [MyPool connection adder] DEBUG com.zaxxer.hikari.pool.HikariPool -- MyPool - Added connection conn4: url=jdbc:h2:tcp://localhost/~/test user=SA
23:05:07.435 [MyPool connection adder] DEBUG com.zaxxer.hikari.pool.HikariPool -- MyPool - Added connection conn5: url=jdbc:h2:tcp://localhost/~/test user=SA
23:05:07.456 [MyPool connection adder] DEBUG com.zaxxer.hikari.pool.HikariPool -- MyPool - Added connection conn6: url=jdbc:h2:tcp://localhost/~/test user=SA
23:05:07.482 [MyPool connection adder] DEBUG com.zaxxer.hikari.pool.HikariPool -- MyPool - Added connection conn7: url=jdbc:h2:tcp://localhost/~/test user=SA
23:05:07.501 [MyPool connection adder] DEBUG com.zaxxer.hikari.pool.HikariPool -- MyPool - Added connection conn8: url=jdbc:h2:tcp://localhost/~/test user=SA
23:05:07.519 [MyPool connection adder] DEBUG com.zaxxer.hikari.pool.HikariPool -- MyPool - Added connection conn9: url=jdbc:h2:tcp://localhost/~/test user=SA
```

- 스레드를 이용하여 모든 커넥션을 만들어 둔 것을 볼 수 있다.
- `Pool stats (total=4, active=2, idle=2, waiting=0)`2개를 사용 중이고 나머지는 사용되고 있지 않다


```java

private void useDataSource(DataSource dataSource) throws SQLException {  
  Connection conn1 = dataSource.getConnection(); // Pool 1개  
  Connection conn2 = dataSource.getConnection(); // Pool 1개 얻을 때까지 기다린다.  
  Connection conn3 = dataSource.getConnection(); 
  Connection conn4 = dataSource.getConnection(); 
  Connection conn5 = dataSource.getConnection(); 
  Connection conn6 = dataSource.getConnection(); 
  Connection conn7 = dataSource.getConnection(); 
  Connection conn8 = dataSource.getConnection(); 
  Connection conn9 = dataSource.getConnection(); 
  Connection conn10 = dataSource.getConnection();
  Connection conn11 = dataSource.getConnection(); 
  log.info("connection = {}, class = {}", conn1, conn1.getClass());  
  log.info("connection = {}, class = {}", conn2, conn2.getClass());  
}
```

- 풀이 차면 설정에 따라 얼마나 기다릴지 설정할 수 있다.
- 풀이 차고 설정한 시간이 지나면 time out 된다.