
## 저자

Vlad

## 1. 시작하며

- 많은 애플리케이션에서 Data Access Layer 성능에서 많은 지연이 발생함
- Data Access Layer에서 필요한 응답시간은 여러가지를 포함한다
	1. Connection 획득
	2. Statement 실행
	3. ResultSet 추출
	4. Business Logic Idle TIme


### 1. 3 로깅

- hibernate.show_sql 
	- 위 속성은 사용하지 말자
- hibernate_format_sql
	- hibernate statement logging에만 사용됨
- hibernate_use_sql_comments
	- entity pacakge name, 조인을 선택한 이유? 등 다양한 정보를 statement에 포함하여 출력해 줌
		- 한 번 사용해보는 것도 좋을듯..?
		- jpa를 깊게 이해하고 있지 못하므로..
	- 운영환경에서는 사용하지 말자
	- 주석 내용을 jdbc를 통해, DB에 전달하기 떄문에 불필요한 오버헤드가 발생할 수 있음

- hibernate logging을 사용하는 것 보다는, 다른 statement logging 라이브러리가 선호된다
	- JDBC Datasource를 Proxy하여 다양한 횡단 관심사를 처리할 수 있기 때문
		- slow query 추적
		- 실행 중인, statement수 확인 등..
	- P6Spy, Datasource-Proxy
		- batch 처리인지, statement 처리인지 확인 가능 
			- **batch 처리에 대해 잘 아는게 없다.. 이 부분 공부 필요!**
			- jdbc에서의 배치 처리, hibernate에서의 배치처리 => db전송 어떻게 되는거지?
				- hibernate.jdbc.batch_size 변경시 어떻게 동작하는가?


- 자바의 로깅은 어떻게 동작하는걸까? 또 로깅 프레임워크는 왜 필요한걸까?
	- log4j, logback, slf4j의 용어의 뜻은 무엇인가?


## 1.4 스키마 관리

- 스키마 마이그레이션을 수동으로 하지 말라
	- CI, CD, DevOps에 반한다.
	- 자동으로 동작하게 할 것 (flyway, liquibase 등)
	- 운영환경에서도 자동으로 동작해야하는가?
		- QA 환경에서 동작했다면, 운영환경에서도 동작해야할 것..
		- 하지만 현재 회사에서는 자동화 하지는 않음 => 자동화 할 순 없는걸까?


### 1.5 통합 테스트