![load failed](/spring/static/png/ExceptionLayer.png)

**`Object`**
- 예외도 객체이다. 모든 객체의 최상위 부모는 `Object`이므로 예외의 최상위 부모도 `Object`이다.
**`Throwable`**
- 최상위 예외. 하위에 `Excectpion`, `Error`가 있다.
**`Error`**
- 메모리 부족이나 심각한 시스템 오류와 같이 애플리케이션에서 복구 불가능한 시스템 예외이다. 애플리케이션 개발자는 이 예외를 잡으려고 해서는 안된다.
**`Exception`**
- 체크 예외
- 애플리케이션 로직에서 사용할 수 있는 실질적인 최상위 예외
- 단 `RuntimeException`은 예외이다.
**`RuntimeException`**
- 컴파일러가 체크 하지 않는 언체크 예외이다.
- `RuntimeException`과 그 자식 예외는 모두 언체크 예외이다. 

**예외 처리**
![](/spring/static/png/exceptionThrow1.png)
- 5번에서 예외를 처리하면 이후에는 애플리케이션 로직이 정상 흐름으로 동작

**예외 던짐**
![](/spring/static/png/exceptionThrow2.png)
- 예외를 처리하지 못하면 호출한 곳으로 예외를 계속 던짐

**2가지 기본 규칙**
1. 예외는 잡아서 처리하거나 던져야 한다.
2. 예외를 잡거나 던질 때 지정한 예외분만 아니라 그 예외의 자식들도 함께 처리된다.
	- 예를 들어 `Exception`을 `catch`로 잡으면 그 하위 예외들도 모두 잡을 수 있다.
	- `Exception`을 `throws`로 던지면 그 하위 예외들도 모두 던질 수 있다.

**예외를 처리하지 못하고 계속 던지면?**
- 자바 `main()`쓰레드의 경우 예외 로그를 출력하면서 시스템 종료
- 웹 애플리케이션의 경우 여러 사용자의 요청을 처리하기 때문에 하나의 예외 때문에 시스템이 종료되면 안된다. WAS는 해당 예외를 받아서 사용자에게 지정한 오류페이지를 보여준다.

### **체크 예외**
- `Exception`과 

```java
static class Repository {  
  public void call() throws MyCheckedException {  
    throw new MyCheckedException("ex");  
  }

static class Service {  
  Repository repository = new Repository();  

  // 체크 예외는 예외를 던지지 않으면 컴파일 에러
  public void callCatch() {  
    repository.call();  
  }  

  // 예외를 던지거나 처리해주자
  // 던지기
  public void callCatch() throws MyCheckedException {  
    repository.call();  
  }  

  // 처리해주기
  public void callCatch() {  
	try {
	    repository.call();  
	} catch (Exception e) {
		....
	}
  }   
	  
}

```

**체크 예외의 장단점**
- 체크 예외는 예외를 잡아서 처리할 수 없을 때, 예외를 밖으로 던지는 `throws`예외를 필수로 선언해야한다. 그렇지 않으면 컴파일 오류가 발생한다.
- 장점
	- 개발자가 실수로 예외를 누락하지 않도록 컴파일러를 통해 문제를 잡아주는 훌륭한 안전 장치이다.
- 단점
	- 하지만 실제로는 개발자가 모든 체크 예외를 반드시 잡거나 던지도록 처리해야 하기 때문에, 너무 번거로운 일이 된다. 크게 신경쓰고 싶지 않은 예외까지 모두 챙겨야 한다. 
	- 의존관계에 따른 단점도 있다.


### 언체크 예외
- `RuntimeException`과 그 하위 예외는 언체크 예외로 분류된다
- 언체크 예외는 말 그대로 컴파일러가 예외를 체크하지 않는다는 뜻이다.
- 언체크 예외는 체크 예와와 기본적으로 동일하다. 차이가 있다면 예외를 던지는 `throws`를 선언하지 않을 수 있다.

**언체크 예외의 장단점**
- 언체크 예외는 예외를 잡아서 처리할 수 없을 때, 예외를 밖으로 던지는 `throw 예외`를 생략할 수 있다.
- 장점
	- 신경쓰고 싶지 않은 언체크 예외를 무시할 수 있다. 체크 예외의 경우 처리할 수 없는 예외를 밖으로 던지려면 항상 `throw 예외`를 선언해야 하지만, 언체크 예외는 이 부분을 생략할 수 있다. 이후에 설명하겠지만, 신경쓰고 싶지 않은 예외의 의존관계를 참조하지 않아도 되는 장점이 있다.
- 단점
	- 언체크 예외는 개발자가 실수로 예외를 누락할 수 있다. 반면에 체크 예외는 컴파일러를 통해 예외 누락을 잡아준다.


### **체크 예외 활용**
- 언제 체크 예외를 사용하고 언제 언체크 예외를 사용하면 좋을까?

**기본 원칙 2가지**
1. 기본적으로 언체크(런타임) 예외를 사용하자.
2. 체크 예외는 비즈니스 로직상 의도적으로 던지는 예외에만 사용하자.
	- 이 경우 해당 예외를 잡아서 반드시 처리해야 하는 문제일 때만 체크 예외를 사용해야 한다.
	- 예시
		- 계좌 이체 실패 예외
		- 결제시 포인트 부족 예외
		- 로그인 ID, PW 불일치 예외

**체크 예외의 문제점**
- 체크 예외는 컴파일러가 예외 누락을 체크해주기 때문에 개발자가 실수로 예외를 놓치는 것을 막아준다. 그래서 항상 명시적으로 예외를 잡아서 처리하거나, 처리할 수 없을 때는 예외를 던지도록 `method() throws`예외로 선언해야한다.
- 체크 예외가 더 안전해 보이는데 왜 기본으로 사용하는 것이 문제가 될까

![](/spring/static/png/checkedException.png)
- 리포지토리는 DB에 접근해서 데이터를 저장하고 관리한다. 여기서는 `SQLException` 체크 예외를 던진다
- `NetworkClient`는 외부 네트워크에 접속해서 어떤 기능을 처리하는 객체이다. 여기서 `ConnectException`체크 예외를 던진다.
- 서비스는 리포지토리와 `NetworkClient`를 둘다 호출한다.
	- 따라서 두 곳에서 올라오는 체크 예외인 `SQLException`과 `ConnectException`을 처리해야한다.
	- 서비스는 이 둘을 처리할 방법이 없다.
- 처리할 방법이 없으므르 둘다 밖으로 던진다.
	- `method() throws SQLException, ConnectException`
- 컨트롤러도 두 예외를 처리할 방법이 없다
- 웹 애플리케이션은 스프링 MVC가 제공하는 `ControllerAdvice`에서 이런 예외를 공통으로 처리한다. 
	- 이런 문제들은 보통 사용자에게 어떤 문제가 발생했는지 설명이 어렵다
	- API는 HTTP 상태코드 500을 사용한다
	- 해결이 불가능한 공통 예외는 별도의 오류로그를 남기고, 개발자가 오류를 빨리 인지할 수 있도록 메일, 알림을 통해 전달 받아야 한다.

**2가지 문제**
##### 1. **복구 불가능한 예외**
- 대부분의 예외는 복구가 **불가능**하다. 일부 복구가 가능한 예외도 있지만 아주 적다.
- `SQLException`의 경우 데이터베이스에 무언가 문제가 있어서 발생하는 예외이다. 데이터 베이스 관련 문제는 복구가 불가능하다. 따라서 이런 문제는 일관성 있게 공통으로 처리해야한다. 서블릿 필터, 스프링 인터셉터, 스프링의 `ControllerAdvice`를 사용하면 이런 부분을 깔끔하게 공통으로 해결할 수 있다.
##### 2. 의존 관계에 대한 문제
- 대부분의 예외는 복구 불가능한 예외이다. 그런데 체크 예외이기 때문에 컨트롤러나 서비스 입장에서는 본인이 처리할 수 없어도 어쩔 수 없이 `throws`를 통해 던지는 예외를 선언해야한다.

```java
static class Service {  
  public void logic() throws ConnectException, SQLException {  
    repository.call();  
    networkClient.call();  
  }  
}  
  
static class Controller {  
  Service service = new Service();  
  // throws SQLException을 사용한다는 것은 
  // Controller가 JDBC에 의존한다는 의미이다.
  public void request() throws SQLException, ConnectException {  
    service.logic();  
  }  
}
```
- 즉 컨트롤러 입장에서 어차피 본인이 처리할 수 없는 예외를 의존해야하는 문제가 발생하게 된다.
- 결과적으로 OCP, DI를 통해 클라이언트 코드의 변경 없이 대상 구현체를 변경할 수 있다는 장점이 체크 예외 때문에 발목을 잡게된다.
![](/spring/static/png/checkedException2.png)
- JDBC -> JPA 같은 기술로 변경하면 예외도 함께 변경해야한다. 그리고 해당 예외를 던지는 모든 다음 부분도 함께 변경해야한다. 
	- `logic() throws SQLException` -> `logic() throws JPAException`

**정리**
- 데이터베이스나 네트워크 통신에서 올라오는 예외는 복구가 불가능하다.
- 이런 복구 불가능한 예외를 서비스, 컨트롤러 같은 각각의 클래스가 모두 알고 있어야(의존해야)한다. 불필요한 의존관계 문제가 발생한다.
	- 의존 관계 문제를 해결하기 위해 `throws Exception`을 사용하는 것은 다른 체크 예외를 체크할 수 있는 기능이 무효화 된다. 이러한 방식은 그리 좋은 것이 아니다.

#### 언체크 예외 활용
![](/spring/static/png/unCheckedException.png)

- `SQLException`을 런타임 예외인 `RuntimeSQLException`으로 변환했다
- `ConnectException` 대신에 `RuntimeConnectException`을 사용하도록 바꾸었다
- 런타임 예외이기 때문에 서비스, 컨트롤러는 해당 예외들을 처리할 필요 없다.

#### **예외 포함과 스택 트레이스**
- 예외를 전환할 때는 꼭 **기존 예외**를 포함해야한다.
	- 스택 트레이스를 확인할 때 심각한 문제가 된다.
```java
@Test  
public void printEx() {  
  Controller controller = new Controller();  
  try {  
    controller.request();  
  } catch(Exception e) {  
    log.info("ex", e);  
  }  
}
```
- 로그를 출력할 때 마지막 파라미터에 예외를 넣어주면 로그에 스택 트레이스를 출력할 수 있다.

```java
static class Repository {  
  public void call() {  
    try {  
      runSQL();  
    } catch(SQLException e) {  
      throw new RuntimeSQLException(e);  
      // throw new RuntimeSQLException();  
    }  
  }
}
```
- `throw new RuntimeSQLException()`을 사용한다면 `SQLException`일 때의 스택 트레이스가 모두 사라진다. `SQLException`에는 이전 오류 정보가 다 들어있다.
- `Throwable`에 이전 정보들을 다 담아주는 메커니즘이 있다.