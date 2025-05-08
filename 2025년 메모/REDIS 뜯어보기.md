
https://github.com/redis/redis


- Remote DIctionary Server
	- 싱글 스레드 기반
		- 락 프리 구조
		- 원자성 보장
		- 컨텍스트 스위칭 최소화
	- 이벤트 루프 아키텍쳐
	- 비동기 I/O, I/O 멀티플렉싱, 논블로킹 소켓
		- 레디스의 핵심 성능 비결
		- 여러 I/O 스트림 (소켓, 파일 등)을 하나의 스레드에서 효율적으로 처리할 수 있게 해주는 기술
		- 커널레벨에서 시스템 호출을 사용해 여러 I/O 이벤트를 감지하고, 준비된 것만 처리
			- 서버 소켓을 감시 대상으로 등록
			- 준비된 이벤트가 발생할 때 까지 대기
		- 수천 개 연결을 한 스레드로 처리가능 => 스레드 오버헤드 없음
		- ae.c (이벤트 루프), ae_epoll.c (epoll 백엔드)
		- Redis뿐 아니라, node.js, nginx, mysql 8, kafka 등에서도 사용한다.
		- 자바에서는 java.nio 패키지를 사용하여 구현할 수 있음 (non-blocking i/o)
	- https://medium.com/better-programming/internals-workings-of-redis-718f5871be84
	- 모든 데이터를 메모리에 올리고 있음, 주기적으로 DISK에 백업

- 주로 사용되는 use-case
	- 캐쉬
	- 분산 락
	- 휘발성 데이터 (1일만 보관해야하는 데이터 등)
	- 잭팟 (왜 여기에 보관해야하는가..?)
		- RDBMS에 비해서 성능이 우수해서..? 단지?
	- AI 데이터 저장소

- 소스코드 분석
	- c로 작성..

- 주의해야할 점
	- 메모리에 데이터를 저장하기 때문에 물리메모리 용량보다 더 많이 사용하게 될 경우 swap이 발생할 수 있다.
		- 보통 몇 % 사용해야하는가?
		- OS의 swap에 대해서 잘 알고있는가?
		- max-memory 설정
		- max-memory policy
			- LRU, LFU, Random의 3가지 방식 제공

- Redis 코드를 이해하여, java 버전으로 작성해볼까?