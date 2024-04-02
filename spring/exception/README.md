![load failed](ExceptionLayer.png)

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