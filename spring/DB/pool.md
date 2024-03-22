
## Connection Pool
![이미지 로드 실패](/Excalidraw/png/connection.png)
1. 애플리케이션 로직은 DB 드라이버를 통해 커넥션 조회
2. DB 드라이버는 `TCP/IP` 커넥션을 연결. 물론 이 과정에서 `3 way handshake` 같은 `TCP/IP` 연결을 위한 네트워크 동작 발생
3. DB 드라이버는 `TCP/IP` 커넥션이 연결되면 ID, PW와 기타 부가정보를 DB에 전달
4. DB는 ID, PW를 통해 내부 인증을 완료하고, 내부에 DB session을 생성.
5. DB는 커넥션 생성이 완료되었다는 응답 보냄.
6. DB 드라이버는 커넥션 객체를 생성해서 클라이언트에 반환

### 문제점
- 이렇게 커넥션을 새로 만드는 것은 **과정도 복잡**하고 **시간이 매우 많이 소모**되는 일이다.
### Connection Pool
![이미지 로드 실패](/Excalidraw/png/connectionPool.png)
- 위 문제를 해결하기 위해 커넥션을 미리 생성해두고 사용하는 방법이다.
	- 기본 값은 10이지만 서버 스펙에 따라 숫자가 다름

![이미지 로드 실패](/Excalidraw/png/connectingPool.png)
- 애플리케이션 로직에서 이는 DB드라이버를 통해서 새로운 커넥션을 획득하는 것이 아님
- 이미 생성되어 있는 객체 참조로 그냥 가져다 쓰면 된다.
- 커넥션 풀에 커넥션을 요청하면 커넥션 풀은 자신이 가지고 있는 커넥션 중에 하나를 반환한다.
![image load failed](/Excalidraw/useConnectionPool)
- 애플리케이션 로직은 요청을 받으면 커넥션 풀에서 커녁션을 사용해 SQL을 db에 전달한다.
- 커넥션을 사용한 후 커넥션을 커넥션 풀에 반환한다.

### 정리
- 적절한 커넥션 풀 숫자는 성능테스트를 통해 정하자
- `hikariCP`를 주로 사용한다.




