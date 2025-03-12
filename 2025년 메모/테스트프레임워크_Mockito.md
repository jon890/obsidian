
https://github.com/mockito/mockito/wiki/Mockito-features-in-Korean

- 테스트 대역 (스턴트맨, Test Double)
	- 테스트를 위해 실제 객체를 대체하는 것
- 모의 객체 (목 객체, Mock)
	- 호출했을 때 사전에 정의된 명세대로 결과를 돌려주도록 프로그래밍 된 테스트용 객체
- Stubbing
	- 모의 객체 생성 및 모의 객체 동작 지정
- Verification
	- 테스트하고자 하는 메서드가 의도한 대로 동작하는지 검증
	- Mockito.verify(mock).action() 등으로 사용

- 유닛 테스트에 잘 활용해보기