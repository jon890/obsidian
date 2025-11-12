
- https://docs.spring.io/spring-framework/reference/testing/integration.html

## 통합 테스트

- 스프링의 통합 테스트는 다음 주요한 목표를 달성하도록 한다
	- Spring IoC 컨테이너를 테스트간에 캐싱할 수 있도록 한다
	- 테스트 픽스쳐 인스펀스 간의 의존성 주입을 제공한다
	- 통합 테스트 중에 적절한 트랜잭션 관리가 되도록 한다
	- 개발자가 통합테스트 작성하는데에 스프링에 적절한 기본 클래스를 제공한다

- 컨텍스트 관리 및 캐시
	- ApplicationContext, WebApplicationContext 인스턴스는 잘 캐시할 수 있음
	- 이 startup 하는 시간은 꽤나 크게 걸리므로 캐쉬하는 것이 좋음
		- 50 ~ 100개의 하이버네이트 매핑 파일은 로드되는데 10-20초까지 걸릴 수 있음
	- 기본적으로, 한번 로드되면, 구성된 ApplicationContext는 각 테스트에서 재사용 됨
		- 빈 정의를 수정하거나, 애플리케이션 상태 객체등을 오염하지 않는 이상 재사용 됨
	- `test suite` 라는 의미는 같은 JVM에서 수행되는 모든 테스트를 의미함

## Spring TextContext Framework

- org.springframework.test.context 패키지에 위치
- 테스팅 프레임워크 (junit4, junit5) 등에 관계 없이 동작할 수 있도록, 제너릭, 애노테이션 기반으로 동작


### TestContext

- 테스트가 동작하는 컨텍스트를 캡슐화 한 것 (어떤 테스팅 프레임워클르 쓰는지와 관계없이)
- 테스트 인스턴스에 대한 컨텍스트 관리 캐시를 지원
- ApplicationContext 가 요청되면 SmartContextLoader에게 위임하여 요청


### TestContextManager

- 