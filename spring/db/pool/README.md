
[pool](/spring/db/pool/ConnectionPool.md)

### Summary

1. 커넥션 생성을 통한 커넥션 획득

![이미지 로드 실패](/spring/static/png/connection.png)
- database 연결과정
	1. 클라이언트 요청
	2. 커넥션 
	3. db에 tcp/ip와 id, pw .. 등등을 보낸다.
	4. db에서 커넥션을 생성하고 반환한다.
- 문제점 
	- 커넥션이 필요할 때 마다 커넥션을 생성한다. 커넥션을 생성하는 것은 굉장히 많은 비용이 든다.

2. 커넥션 풀 사용
	- 커넥션을 미리 만들어 두고 사용하자
![이미지 로드 실패](/spring/static/png/connectionPool.png)
- 커넥션을 새로 생성하지 않고 커넥션 풀에서 커넥션을 가져온다.