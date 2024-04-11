
스프링 부트를 이용하고 있다면 데이터소스나 트랜잭션 매니저를 등록한적이 없다고 생각할 것이다.

**직접 등록**
```java
@Bean
DataSource dataSource() {
	return new DriverManagerDataSource(URL, USERNAME, PASSWORD);
}

@Bean
PlatformTransactionManager transactionManager() {
	return new DataSourceTransactionManager(dataSource());
}
```


**데이터소스 - 자동등록**

```properties
spring.datasource.url=jdbc:h2:tcp://localhost/~/test
spring.datasource.username=sa
spring.datasource.password=...
```
- 스프링 부트가 기본으로 생성하는 데이터소스는 커넥션풀을 제공하는 `HikariDataSource`이다. 커넥션 풀과 관련된 설정도 `application.properties`를 통해 지정할 수 있다.
- `spring.datasource.url`속성이 없으면 내장 데이터베이스를 생성하려고 시도한다.

**트랜잭션 매니저 - 자동등록**
- 스프링 부트는 적절한 트랜잭션 매니저(`PlatformTransactionManager`)를 자동으로 스프링 빈에 등록한다
- 자동으로 등록되는 스프링 빈 이름: `transactionManager`
- 개발자가 직접 트랜잭션 매니저를 빈으로 등록하면 스프링 부트는 트랜잭션 매니저를 자동으로 등록하지 않는다
	- 트랜잭션 매니저 선택기준
		- JDBC `DataSourceTransactionManager`
		- JPA `JpaTransactionManager`

**스프링은 `transactionManager`와 `dataSource`를 자동등록한다.**


