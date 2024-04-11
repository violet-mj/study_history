
```java
@EventListener(ApplicationReadyEvent.class)  
public void initData() {  
    log.info("test data init");  
    itemRepository.save(new Item("itemA", 10000, 10));  
    itemRepository.save(new Item("itemB", 20000, 20));  
}
```
- 애플리케이션 실행할 때 초기 데이터를 저장한다.
- 리스트에서 데이터가 잘 나오는지 편리하게 확인할 용도로 사용한다.
- `@EventListener(ApplicationReadyEvent.class)`는 스프링 컨테이너가 완전히 초기화를 다 끝내고, 실행준비가 되었을 때 발생하는 이벤트이다.
	- `@PostConstruct`를 사용할 경우 AOP와 같은 부분이 아직 다 처리되지 않은 시점에 호출될 수 있기 때문에 간혹 문제가 발생할 수 있다.
	-`@EventListener(ApplicationReadyEvent.class)`는 AOP를 포함한 스프링 컨테이너가 완전히 초기화 된 이후에 호출된다.

**ItemServiceApplication**
```java
@Import(MemoryConfig.class)  
@SpringBootApplication(scanBasePackages = "hello.itemservice.web")  
public class ItemServiceApplication {  
  
  public static void main(String[] args) {  
   SpringApplication.run(ItemServiceApplication.class, args);  
  }  
  
  @Bean  
  @Profile("local")  
  public TestDataInit testDataInit(ItemRepository itemRepository) {  
   return new TestDataInit(itemRepository);  
  }  
  
}
```
- `@Import(MemoryConfig.class)` MemoryConfig을 설정 파일로 사용한다.
- `scanBasePackages = "hello.itemservice.web"`을 지정하지 않으면 현재 이 파일 위치와 그 하위 폴더들이 모두 컴포넌트 스캔 대상이 된다.
- `@Profile("local")`은 특정 프로필의 경우에만 해당 스프링 빈을 등록한다. 여기서 `local`이라는 이름의 프로필이 사용되는 경우에만 `testDataInit`을 스프링 빈으로 등록한다.


### 프로필
- 스프링은 로딩 시점에 `application.properties`의 `spring.profiles.active`의 속성을 읽어서 프로필로 사용한다.
- **main 프로필**
```properties
spring.profiles.active=local
```

- **test 프로필**
	- `/src/test/resources` 하위의 `application.properties`
```java
spring.profiles.active=test
```
- 이 경우 `@Profile("local")`이기 때문에 위 테스트 데이터 설정이 반영되지 않는다. 
- `TestDataInit`은 빈으로 등록되지 않는다.

**인터페이스를 테스트하자**
- 구현체가 바뀌더라도 같은 코드로 테스트할 수 있다.

데이터베이스 기본 키의 3가지 조건
1. `null`을 허용하지 않는다.
2. 유일해야한다.
3. 변경이 불가능하다.

**기본 키를 선택하는 전략 2가지**
1. 자연 키 (nature key)
	- 비즈니스에 의미가 있는 키
	- 예: 주민등록번호, 이메일, 전화번호
2. 대리 키 (surrogate key)
	- 비즈니스와 관련 없는 임의로 만들어진 키, 대체 키
	- auto_increment, identity, 키생성

**자연 키보다는 대리 키를 권장한다**
- 비즈니스 규칙은 생각보다 쉽게 변한다.

**비즈니스 환경은 언젠가 변한다**






