
- https://docs.spring.io/spring-framework/reference/testing/integration.html

## 통합 테스트

- 스프링의 통합 테스트는 다음 주요한 목표를 달성하도록 한다
	- Spring IoC 컨테이너를 테스트간에 캐싱할 수 있도록 한다
	- 테스트 픽스쳐 인스펀스 간의 의존성 주입을 제공한다
	- 통합 테스트 중에 적절한 트랜잭션 관리가 되도록 한다
	- 개발자가 통합테스트 작성하는데에 스프링에 적절한 기본 클래스를 제공한다

- 컨텍스트 관리 및 캐시
	- 