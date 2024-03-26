  

### JDBC(Java Database Connectivity)

- 자바에서 데이터베이스에 접속할 수 있도록 하는 자바 API다.

![이미지 로드 실패](jdbcStructure.png)

  

- java.sql.Connection
    
    ![image load failed](ConnectionInterface.png)


    
    ![이미지 로드 실패](spring/static/driverManager.png)

_
    

  

JDBC는 `java.sql.Connection` 표준 인터페이스를 정의

H2 데이터베이스 드라이버는 JDBC Connection 인터페이스를 구현한 `org.h2.jdbc.JdbcConnection` 을 제공

  

  

  

  

라이브러리에 등록된 드라이버를 모두 살펴본 후 커넥션을 획득할 수 있는지 확인한다.

  

- java.sql.Statement
- java.sql.ResultSet

  

개선점

1. 데이터베이스를 다른 종류의 데이터베이스로 변경하면 애플리케이션 사용 코드도 함께 병경해야하는 문제
2. 개발자가 각각의 데이터베이스마다 커넥션 연결, SQL 전달, 그리고 그 결과를 응답 받는 방법을 새로 학습해야하는 문제

  

문제점

- JBCD의 등장으로 많은 것이 편해졌지만, **각각의 데이터베이스마다 SQL, 데이터타입, 페이징 SQL등의 일부 사용이 다름**

  

PreparedStatement

- statement의 자식 타입, `?` 을 통한 파라미터 바인딩을 가능하게 해준다.
    - SQL Injection을 예방하기 위해 `?` 을 사용해야한다.
- executeQuery
    
    - 실행 후 ResultSet을 반환하는 메서드
    
      
    
- excuteUpdate
    
    - DML(INSERT, UPDATE, DELETE) 등의 데이터 조작어를 실행할 때 사용한다.
    - Return Value
        - DML 1
        - DDL 2