
1. `@Test`의 메서드는 `public`이어야 한다.
```java

/**
 * org.junit.runners.model.InvalidTestClassError: Invalid test class 'hello.jdbc.exception.basic.UncheckedTest':
 * 1. Method uncheckedCatch() should be public
 * 2. Method uncheckedThrow() should be public
*/
@Test  
void uncheckedCatch() {  
  Service service = new Service();  
  service.callCatch();  
}

// correct
@Test  
public void uncheckedCatch() {  
  Service service = new Service();  
  service.callCatch();  
}
```
- "테스트 러너가 리플렉션을 사용하기 때문에 `public`을 사용하여 접근할 수 있게 해주어야 한다."
	- [Why Junit test cases(Methods) should be public? - stackoverflow](https://stackoverflow.com/questions/37019972/why-junit-test-casesmethods-should-be-public)
- "어쨌든 JUnit은 전통을 유지해서 JUnit 4.x에서도 여전히 public 메소드만 테스트로 허용하고 있습니다."
	- [JUnit의 테스트 메소드는 왜 public이어야 할까요? - groups.google](https://groups.google.com/g/ksug/c/xpJpy8SCrEE)
