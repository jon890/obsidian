
https://github.com/redis/redis


- Remote DIctionary Server

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
	- 