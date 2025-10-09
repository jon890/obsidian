
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


- 궁금한점
	- 자바의 로깅은 어떻게 동작하는걸까? 또 로깅 프레임워크는 왜 필요한걸까?
	- log4j, logback, slf4j의 용어의 뜻은 무엇인가?


## 1.4 스키마 관리

- 스키마 마이그레이션을 수동으로 하지 말라
	- CI, CD, DevOps에 반한다.
	- 자동으로 동작하게 할 것 (flyway, liquibase 등)
	- 운영환경에서도 자동으로 동작해야하는가?
		- QA 환경에서 동작했다면, 운영환경에서도 동작해야할 것..
		- **하지만 현재 회사에서는 자동화 하지는 않음 => 자동화 할 순 없는걸까?**


### 1.5 통합 테스트

- 많은 애플리케이션에서 테스트 속도를 위해, 메모리 DB로 통합테스트를 진행한다.
	- 적절한 테스트가 수행되지 못 할 수 있음
	- DB의 기능을 많이 사용하는 경우, store procedure 등..
	- **그런데 내 생각에는 많은 data access library에서 테스트를 진행해주고 있으므로, 비즈니스 로직에 대한 테스트를 더 강화하는게 좋지 않을까**
- 더 빠르게 테스트하기 위해서, docker, testcontainers등의 실제 DB 구현체를 사용해보기
- 도커 tmpfs 볼륨을 사용해서, 빠르게 테스트하기 등의 방법을 사용하여 테스트 진행


## 2. 커넥션

### 2.1 커넥션 관리 및 하이버네이트 커넥션 제공자

- JTA는 뭘까?
- jpa 2.0부터는 jdbc driver 기반의 커넥션 관리를 지원함
	- javax.persistence.jdbc.url
- 애플리케이션 트랜잭션 응답시간 중, 커넥션을 획득하는 과정도 빠르게 처리되어야 함
	- jdbc driver를 통해 얻기
		- DriverManager.getConnection()
		- -> RDBMS와 소켓을 열어 TCP 통신 시작
		- 꽤나 많은 시간이 소요될 수 있음
	- Connection Pool을 사용해서 시간을 줄이는 것이 좋음
		- HikariCP => Connection이 확보만 되어있다면 매우빠름
- 하이버네이트가 Connection이 필요하면? `ConnectionProvider` 인터페이스를 통해 커넥션을 얻거나, 반환한다
	- HikraiConnectionProvider

- 궁금한점
	- Connection Pool을 사용하면, 각 애플리케이션이 커넥션 몇 개를 **점유(혹은 소유)**하게 되는 것 인가? 
		- 그렇다면 이 때, 오토 스케일링으로 애플리케이션이 늘어나거나 줄어들 떄, 커넥션 개수는 어떻게 되는 거지?
		- 커넥션풀 모니터링?

### 2.2 하이버네이트 커넥션 라이프사이클

- hibernate.cinnection.handling_mode
	- DELAYED_ACCQUISITION_AND_RELEASE_AFTER_STATEMENT
		- 커넥션 획득을 지연
		- 각 statement 수행 후 해제 (JTA)
	- DELAYED_ACCQUISITION_AND_RELEASE_AFTER_TRANSACTION
		- 커넥션 획득 지연
		- 트랜잭션 수행 후 해제 (RESOURCE_LOCAL)
- hibernate.connection.proider_disables_autocommit
 - dataSource가 connection을 가져올 떄, autocommit이 비활성화 되어있다면 hibernate는 getAutoCommit으로 값을 가져올 필요가 없음
	- 이를 위한 최적화
	- ResourceLocalDelayConnectionAcquisitionTest 로 확인해보기

- 궁금한 점
	- ResourceLocal, JTA는 무엇이 다른거지?

### 2.3 커넥션 모니터링

- 커넥션 풀 사이즈 관리법
	- Little's Law, Erlang B, Erlang C 등
	- 공식으로만은 적절하지 않음, 다른 외부 변수가 많기 떄문에
		- 네트워크 지연, jvm, gc 등..
	- 이를 위해 모니터링 및 failover 구성이 필요하다.
	- flexy-pool
		- 풀 연결시간을 알 수 있는 오픈소스 라이브러리
		- 커넥션 수를 유동적으로 사용할 수도 있게 해줌
		- 다른 모니터링 도구로 데이터를 전달할 수 있음
		- FlexyPoolTest
			- 이 테스트를 통해 쉽게 설정할 수 있음을 보여 줌


### 2.4 하이버네이트 통계

- StatisticsImplementor
	- openSession
	- closeSession
	- flush
	- connect
	- prepareStatement
	- closeStatement
	- endTransaction
	- optimisticFailure
	- loadEntity
	- fetchEntity
	- insertEntity
	- updateEntity
	- deleteEntity...
- hibernate.generate_statistics를 true로 하면 통계수집을 활성화 할 수 있다
- Dropwizard Metrics를 통해 모니터링하면 좀 더 쉽게 모니터링 가능


## 3. 타입

### 3.1 JPA와 하이버네이트의 타입

- db에서의 타입은 최대한 compact한게 좋다
	- db에서의 데이터 단위인 페이지에 여러 행들이 존재하는 것 이 좋기 떄문에 컴팩트하게 유지하는 것이 좋음
	- codeEnum을 사용해서 데이터 크기를 줄이면 좋겠다.

### 3.2 하이버네이트 커스텀 타입

- ip와 같은 정보를 저장하기 위한 컬럼 타입으로 PostgreSQL의 inet 타입을 사용할 수 있다
- 이 타입을 지원하기 위해, 하이버네이트 커스텀 타입을 구현해야한다.
- IPv4TypeTest를 참조해보자
- hypersistence 패키지?


## 4. 식별자

- Identity
	- mysql AUTO_INCREMENT 전략등으로 증가하는 DB 내부 카운터 값
	- 경량화된 잠금 매커니즘을 사용하여 빠름
	- 락을 짧게 가져가기 떄문에, 트랜잭션 롤백 시, 중간 값이 비는 경우가 존재할 수 있음 (버그 아님)
	- 실제 insert statement가 수행되어야 id값을 알 수 있기 떄문에, jdbc 배치로는 처리가 불가능
		- persist -> insert statement -> 트랜잭션 기반 쓰기지연 캐싱 시맨틱이 꺠짐으로 불가능
- sequence
	- database sequence
	- hibernate_sequence를 통해 값을 가져오고, 값을 사용
	- 경량화된 잠금 매커니즘을 사용
	- 카운터값만 증가되고 락이 풀리는 구조
- table
	- JPA에서 지원해주는 기능
	- sequence를 관리히는 테이블, 행 수준의 잠금으로 관리
	- 별도의 트랜잭션을 통해 시퀀스 값을 가져와야함으로 오버헤드 발생
- auto

- 이식성을 위해 xml을 사용하여 별도로 전략을 지정해줄 수 도 있음
	- xml > annotation
	- 애노테이션 설정을 덮어쓸 수 있음

- PostgreSQLIdentifierTest
- MySQLIdentifierTest


### 4.2 하이버네이트 식별자 최적화

- hi/lo algorithm
	- 식별자를 여러 개 미리 생성해놓는 전략? 
	-  다른 애플리케이션이 다른 전략을 사용해버리면 문제가 생김
- the pooled optimizer
	- jpa 표준 애노테이션으로 작동
	- 시퀀스는 1, 4, 7, 10과 같이 생성
		- 다른 스레드가 동일한 시퀀스를 생성할 수 없다는 것을 알고
		- 1, 2, 3, 4의 식별자를 사용할 수 있다와 같이 동작
- 한 트랜잭션에서 여러 insert를 수행할 떄 사용하면 좋은 건가? 어떨 떄 사용해야하는지 아직은 잘 와닿지 않네..


## 5. 관계

### 5.1 JPA와 하이버네이트의 관계

- hibernate에서, fk를 사용하지 않고, 연관관계를 사용할 수 있나?
	- fk의 오버헤드 때문에, fk를 잘 안걸고 개발하는 것 같은데..
	- jpa의 이점을 많이 누리지 못하고 있는 것 같기도하다. 
	- 근데 확장성 떄문에 fk를 사용하지 않는게 좋으려나? 
	- sql 쿼리로 조회하는 것이 더 나아보이기도한다.
		- 데이터의 양이 얼마나 될지 알 수 없음
		- 페이징 처리 가능

- equals and hashcode를 올바르게 구현하기
	- AbstractEqualityCheckTest
		-  모든 상태 변경에서 일관성이 있는지 테스트할 수 있는 클래스
	- 제일 쉬운 법은 자연 식별자나 비즈니스키를 사용하는 것이다
		- 자연 식별자 - isbn 등..
			- NaturalIdEqualityTest

### 5.4 OneToOne

- Post - PostDetails
	- 단방향
		- JoinColumn을 통해 명시
			- 기본키와 외래키가 분리
		- 기본키와 외래키를 합치도록하려면 MapsId를 사용하는 것이 좋음
	- 양방향
		- OneToOne 애노테이션 속성에 mappedBy를 추가
		- 부모에 보통 자식을 설정할 수 있는 유틸 메소드를 추가하는 것이 관행
		- N+1 문제가 발생할 수 있음
			- 바이트코드 향상을 통해 발생하지 않도록 할 수 있음
			- `@LazyToOne(LazyToOneOption.NO_PROXY)`
				- 하이버네이트에서는 MapsId와 같이 동작하지 않음
				- 따라서 MapsId를 사용하는 것이 더 선호됨
