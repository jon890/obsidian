
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


## 6. 상속

### 6.1 JPA 상속의 기본 개념

- 단일 테이블 상속 -> InheritanceType.SINGLE_TABLE
- 클래스 테이블 상속 -> InheritanceType.JOINED
- concrete 테이블 상속 -> InheritanceType.TABLE_PER_CLASS

### 6.2 단일 테이블 상속

- SingleTableTest
- 호율적인 statement 생성, 성능면에서 우수하다
- subclass 컬럼에 not null validation을 사용할 수 없다?
	- 반드시 하위 클래스에서 설정되어야 함으로
	- 이 무결성은 다른 애플리케이션에서 꺠질 수 있음으로 반드시 지켜져야 하도록 check, trigger를 통해 DB 레벨에서 체크되어야 함
	- SingleTableMySQLTriggerTest

### 6.3 JPA 상속구분자 컬럼

- `@DiscriminatorColumn`
	- String (default), Char, Integer 사용 가능
	- dType에 대해서 인덱스를 사용할 것이므로, 값이 짧은게 성능면으로 유리하긴하다 (얼마나 차이날까?)
	- string 사용하는게 나아보임 ㅎㅎ 대신, 적절히 줄인정도로 사용하면 좋을듯?

## 6.4 Joined 상속

- subClass 수가 많지 않을 떄 사용하면 좋을듯..?

### 6.5 Table per Class 상속 

- entity 간의 상속된 데이터들이, 각 테이블에 모두 존재하게끔 상속하는 것
	- 다형성 쿼리 시, union all로 가져오게 됨으로 성능상 불리함이 있다. 

### 6.6 MappedSuperclass 상속

- Table per class 상속보다 더 나은 전략


## 7. 영속성 컨텍스트 그리고 플러싱

### 7.1 Basics

- JPA - EntityManager
- Hibernate - Session
- 엔티티 식별자를 키로, 엔티티 객체를 값으로 가지는 맵으로 생각할 수 있다 
- hibernate가 먼저 개발되고 이를 인터페이스화 한 것이 JPA임

- hiberante version > 5.2
- EntityManager (interface) <- SessionImpl (실제 구현체)
	- ㄴ Session (interface) --------⏌

- 트랜잭션 write-behind cahce
	- Application ----- Cache ----- Cache Store ----- Database
	- application logic에서 데이터 변경시 cache.put() -> cacheStroe.enque() 됨
	- 이후 flush를 호출해야 실제 DB에 save 됨
	- 이 시스템은 널리 사용되고 있음
		- 관계형 DB의 버퍼풀, OS 등.. 
	- 쓰기 작업을 큐에 넣고, 한 번에 처리할 수 있다.

- EntityManager, Session은 일반적으로 1차 캐쉬라고 불림
	- 관리되는 모든 entity는 map에 저장되고, EntityManager에 의해 다시 꺼내 쓸 수 있음
	- Managed Entity 상태로 만드는법
		- EntityManager
			- find(entityClass, primaryKey)
			- find(entityClass, primaryKey, lockMode)
			- getReference(entityClass, primaryKey)
			- createQuery().getResultList()
			- createQuery().getSingleResult()
		- Session
			- get(entityClass, primaryKey)
			- load(entityClass, primaryKey)
			- createQuery/createFilter().list()
			- createQuery/createFilter().uniqueReesult()
			- byId/byNaturalId(entityClass).load(primaryKey)
			- byId/byNaturalId(entityClass).getReference(primaryKey)

- Flsuhing
	- write-behind 캐쉬 처럼, 영속성 컨텍스트는 메모리에 저장된 상태와 DB 간의 데이터를 동기화 하기위해 플러시가 필요하다.
	- entity 변경과정을 dirty checking이라 한다.
	- EntityManager와 Hibernate native Session 인터페이스는 flush() 메소드를 정의하고 있고 이는 메모리의 도메인 모델과, DB 구조를 동기화하는 것을 트리거하는 메소드이다. 

- JPA FlushModeType
	- AUTO 
		- 모든 쿼리 수행 전에 flush를 트리거하는 모드
		- 트랜잭션 커밋보다 우선적으로 일어남 
	- COMMIT
		- 트랜잭션이 커밋될 때만 트리거 됨

- Hibernate FlushMode
	- AUTO
	- ALWAYS
	- COMMIT
	- MANUAL 

### 7.2 Action Queue  

- flush 상태에서 사용하는 동작을 action으로 정의하여 queue로 보관하고 수행
	- EntityInsertAction
	- EntityUpdateAction
	- EntityDeleteAction 

- 영속성 컨텍스트 플러시가 끝나기전, 모든 엔티티 액션이 엄격한 순서에 따라서 실행 됨
		- OrphanRemovalAction
		- EntityInsertAction and EntityIdentityInsertAction
		- EntityUpdateAction
		- CollectionremoveAction
		- CollectionUpdateAction
		- CollectionRecreateAction
		- EntityDeleteAction 
	- FlushOrderTest
		- **위와 같은 순서 떄문에, unqiue 제약이 걸린 행을 제거 후, insert하는 로직이 있다면 수행되지 않음**
			- 물론 위 로직이 일반적이라고 할 순 없음, 고유한 식별자를 update 하고, insert하는 것이 바람직한 로직임

### 7.3 FlushModeType.AUTO

- JPA
	- 모든 jpql or native SQL query 실행 시, 트리거 됨
- Hibernate
	- 반드시 모든 쿼리 수행에서 트리거되지는 않음 
	- 스마트 플러시 메커니즘을 사용함
	- 엔티티 상태 전이가 pending되어있는 상태일 때만 flush를 트리거함
	- 위와 같은 최적화로 인해, flush call을 줄이고, 1차 캐시의 동기화를 가능한 지연시키는 것을 목표로 함
	- but.. native query SQL에는 적용되지 않음 
	- **hibernate가 언제 flush해야하는지 판단하고 최적화 함**


### 7.4 더티체킹 메커니즘

- entity 상태가 transient -> managed로 될 떄, hibernate는 INSERT문을 실행한다. 
- entity가 removed로 마킹되면, SQL DELETE가 실행된다.
- UPDATE는 연관된 전이 상태가 없다.
- entity가 managed가 되면, 영속성 컨텍스트는 초기 상태와, 플러시 시점의 상태를 비교해 SQL UPDATE문을 수행한다. 

- 기본 메커니즘
	- PersistenceContext ----- Session ----- DefaultFlushEventListener ----- DefaultFlushEntityEventListener ----- AbstractEntityPersister ----- TypeHelper
	- AbstractEntityPersister.findDirty() 
		- 여기에서 엔티티의 변경점을 찾음 
	- 더티 체크 횟수는, 엔티티 x 엔티티 속성 수
	- final Object[] loadedState = persister.hydrate(resultSet, id, object, rootPersister, cols, fetchAllPropertiesRequested, session)
	- 로딩 시점의 상태도 보관해야함으로, 매니지드 엔티티를 저장하기 위해 2배의 메모리가 필요하다. 
	- 엔티티를 업데이트할 필요가 없다면, read-only 모드로 가져오는 것이 효율적이다.

- 배치 프로세싱
	- 위 떄문에 배치 작업 시, 영속성 컨텍스트의 범위를 잘 사용하는 것이 매우 중요
	- 주기적으로 flush and clear
	- 배치 프로세싱에서 제일 효과적인 방법은 flush-clear-commit 

### 7.5 바이트코드 향상 더티체킹

- 동작 원리
	- hibernate enhance plugin을 사용
	- enableDirtyTracking : true로 설정
- reflection vs bytecode enhancement
	- 개선효과가 엄청나진 않은듯..


## 8 . Statements

- 실행 계획
	- MySQL
		- EXPLAIN EXTENDED
		- EXPLAIN FORMAT=JSON
	- ExecutionPlanTest를 확인해보면 애플리케이션에서 createNativeQuery를 활용하여 실행계획도 보는 것을 확인할 수 있다. 
	- 각 DB 클라이언트에서 제공해주는 실행계획을 확인하면 더 이해할 수 있도록 정보를 많이 제공해줌 (visual explain)

- 자바의 PreparedStatement에 대해서 더 깊게 공부해보면 재밌어 보인다. 
	- jdbc driver 측에서도 캐쉬할 수 있다고 함 
	- connection.prepareStatement()를 호출하여 PrepraredStatement 객체를 반환받고
	- 위를 통해 connection.executeQuery()를 수행하는 것 같았는데
	- 이 때, prepareStatement()도 DB를 왕복하는 동작일 듯 하다.. 
	- MySQL
		- cachePrepStmts (default false)
		- prepStmtCacheSize (default 25)
		- prepStmtCacheSqlLimit (default 256)
	- 깊게 공부하다보면 여기까지 최적화 하게 되는거군..

## 9. Batching

### 9.1 Batch Updates with JDBC and Hibernate

- Statement batching
	- statement.addBatch(INSERT ~ )
	- statement.executeBatch()
	- DB 왕복시간이 줄어듬
	- JDBC도 어떻게 구성되어있는지 좀 더 볼필요가 있어보이네

- MySQL Statement batching
	- 기본적으로 MySQL JDBC driver는 지원하지 않음
		- 지금도 그럴까?
	-  rewriteBatchedStatements 속성을 설정하면, 단일 문자열로 배치 실행
		- 다중값 insert 정도로

- 적절한 배치 사이즈
	- 수학적 공식은 없음
	- 애플리케이션에서 튜닝해보면서 적절한 값을 찾는게 최적임
	- 낮은 배치 사이즈도 트랜잭션 응답시간이 좋아짐으로 (10 - 30) 정도로 시작해서 사용하면 좋음

- Hibernate batching - 기본 동작
	- 기본적으로 JDBC 배치를 사용하지 않음
	- 명시적으로 설정해야함
	- hibernate.jdbc.batch_size
	- SessionFactory, EntityManagerFactory 수준에서 설정 됨
	- GenerationType.IDENTITY 에서는 사용할 수 없음
		- 영속성 컨텍스트는 1차 캐쉬를 사용하기 위해 식별자를 알아야 함 
			- 배치 처리 테이블은 sequence를 사용해보면 더 좋을 수도..?
			- **spring, jdbc, mysql에 최적 배치를 찾아보는 것도 재밌을 듯**

### 9.2 Batching Cascade Operations

- Post -> PostComments (one to many)
	- cascade = CascadeType.ALL
	- orphanRemovel = true
	- 엔티티 상태 전이에 따라서, 배치 처리는 어떻게 될까?
	- hibernate.jdbc.batch_size = 5로 해보자
- Post와 PostComments를 한 번에 **INSERT**
	- 개별적으로 실행 됨
	- Post, PostComment의 다른 엔티티를 바라보기 떄문에, 매 post.persist 마다 배치 작업을 flush 해야 함
	-  이를 해결할 수 있는 법?
		- hibernate.order_inserts = true
		- jdbc statement를 정렬하면서도 참조 무결성이 꺠지지 않도록 유지하도록 보장함
- **UPDATE**
	- hibernate.order_updates 를 설정
- Batching versioned data
	- hibernate.jdbc.batch_versioned_data = true
	- 기본으로 화럿ㅇ화 되어있음
- DELETE
	- 삭제 순서 설정은 없음
	- 해결 법 - 1
		- 자식 엔티티를 제거 후, flush 
		- 이렇게 되면 statement가 교차되지 않기 떄문에 괜찮음
	- 해결 법 - 2
		- cascade 에서 CascadeType.PERSIST, MERGE만 사용, orphanRemoval도 제거
		- 해당 게시글로 먼저 댓글을 모두 삭제하도록 쿼리 수행
	- 해결 법 - 3
		- DB 레벨에서의 처리 (on delete cascade)

### 9.3 Update 연산배치 처리

- default vs dynamic update
	- 기본 update의 단점
		- 컬럼 사이즈
			- 큰 컬럼이 변경되지 않더라도 업데이트
		- 인덱스
			- 인덱스 재조정
		- 레플리케이션
			- 업데이트된 정보가 모든 래플리케이션에 전달 됨
		- 언두 리두 로그
			- 로그 크기 증가
		- 트리거
			- 실제 컬럼이 변경되지 않더라도 트리거가 동작할 수 있음


## 10. Fetching

### 10.1 JDBC Statement Fetch Size

-  statement.setFetchSize(fetchSize) 
- 기본값
	- Oracle - 10
	- SQL Server - 128
	- PostgreSQL - 모든 데이터를 가져옴
	- MySQL - 전체 결과 집합을 가져옴
		- 스트리밍을 위해 다음과 같이 할 수 있음
			- 1개씩 로딩 setFetchSize(Integer.MIN_VALUE)
			- connection property에 useCursorFetch 설정 후 , setFetchSize(fetchSize)
- hibernate.jdbc.fetch_size 

### 10.2 JDBC ResultSet size

- 왜 ResultSet 크기를 제한해야 하는가?
	- 데이터는 시간에 따라 증가한다.  
	- Top-N 쿼리를 사용하는 것이 바람직함
	- 실제 필요한 컬럼만 조회하는 것이 바람직함

- SQL:2008 표준 페이징
	- FETCH FIRST N ROWS ONLY
	- OFFSET M ROWS
	- 지원되는 데이터베이스
		- Oracle 12c
		- SQL Server 2012
		- PostreSQL 8.4

- JPQL Query - 페이징
	- setFirstResult()
	- setMaxResults()

- JDBC Statement max rows
	- statement.setMaxRows(maxRows)
	- **SQL 실행 계획에는 영향을 주지 않음**
		- DB에서 optimizer에 영향을 주지 않음으로, table full scan 할 수 도 있음
	- 문서에 따르면, 제한이 초과되면, 초과된 행은 조용히 버려진다고 함

### 10.3 JPA Query Projections 

- Default projection
	- 기본적으론 `Object[]`로 프로젝션 된다.
	- ```java
	  List<Object[]> tuples = entityManager.createNativeQuery("""
		  SELECT p.id AS id,
				 p.title AS title
		  FROM post p
		  """).getResultList();
	  ```
- Tuple projection
	- ```java
	  List<Tuple> tuples = entityManager.createNativeQuery("""
		  SELECT p.id AS id,
				 p.title AS title
	  """).getResultList();
	  ```
	  java의 map처럼 작동하는 인터페이스, 더 안전해지긴했지만, 타입 안전하지 않아 여전히 불편하다!
- DTO projection
	- ClassImportIntegrator를 통해 단순 클래스 목록을 제공 가능하다
		- hibernate.integrator_provider 속성에 제공하면 됨
	- SqlResultSetMapping
		- NamedNativeQuery 
		- 귀찮다..!!
	- Java 14 Records

### 10.4 Hibernate Query Projections

- AliasToBeanResultTransformer
	- 기본 생성자로 새로운 인스턴스 생성, setter를 통해 값을 채워넣음
- ```java
  List<PostDTO> postDTOs = entityManager.createNativeQuery("""
	  SELECT p.id AS "id", // column 명이 대문자로 반환될 수 있기 때문에 alias 사용
		     p.title AS "title"
	  FROM post p
	  """)
  .unwrap(org.hibernate.query.Query.class) // hibernate Query 객체로 변환
  .setResultTransformer(Transformers.aliasToBean(PostDTO.class))
  .getResultList();
  ```
- Hibernate에서는 parent - child 간의 DTO 매핑도 지원한다.
	- PostDTO에서 `List<PostCommentDTO>`를 가질 수 있다는 뜻
	- HibernateDTOProjectionTest.testParentChildDTOProjectionNativeQueryTupleTransformer에 정의된 테스트를 참조해보자


### 10.5 엔티티 패치(fetch)

- Entity identifier direct fetching - JPA
	- entityManager.find(Post.class, 1L);
		- entity POJO를 가져온다
	- entityManager.getReference(Post.class, 1L);
		- entity Proxy를 가져온다
		- 식별자 외의 값들은 지연 로딩된다.
	- FindVsGetReferenceTest로 비교해보자

- 궁금한 점
	- 2차 캐쉬는 어떻게 구성되어있을까?
	- org.hibernate.annotations.Cache(useage = CacheConcurrencyStrategy.READ_WRITE)

### 10.6 연관관계(Associations) 패치

- @ManyToOne @OneToOne
	- default - FetchType.EAGER
- @OneToMany, @ManyToMany
	- default - FetchType.LAZY 

- FetchType.EAGER
	- 엔티티를 직접 조회할 떄, LEFT JOIN으로 가져온다
	- JPQL or Criteria API (Entity Query)로 조회할 때, FetchType.EAGER 연관관계를 명시적으로 JON FETCH로 지정하지 않으면 2가지 쿼리로 분리되어 조회 됨

- FetchType.LAZY
	- ManyToOne, OneToOne에서 즉시 로딩하지 않기 위해 사용
		- LEFT JOIN으로 조회하지 않는다
		- proxy 객체로 지연 로딩 된다.
		- 연관된 데이터를 필요할 떄 가져올 수 있게 한다.
	- 그러나 Post - PostComments 조회 시, 게시물이 여러개 있을 떄, 코멘트가 N+1로 쿼리 되는 문제가 있을 수 있음

- N+1 쿼리 문제는, LAZY 연관관계 때문만은 아님
	- JPQL 쿼리를 수행할 때, FetchType.EAGER 연관관계에서, PostComment를 조회하면, 댓글 목록에 모든 Post가 N개 조회될 수 있음
	- 어떻게 회피할 수 있나?
		- JOIN FETCH 절을 사용하여, 여러 연관관계 엔티티들을 한 번에 조회

- Open Session in View anti-pattern
	- View까지 영속성 컨텍스트가 계속 열려있어, lazy 로딩을 가능하게 함
	- but DB관점에서 보면 문제가 있음
	- 서비스 레이어에서 외부까지 트랜잭션이 열려있다면, N+1 문제가 발생할 수 있음
	- DB Connection이 오래 점유될 수 있음
	- 원래 영속성 컨텍스트가 닫히기 전까지 연관관계 엔티티를 조회하거나, 엔티티와 연관없는 DTO로 변환하여 적절한 범위에서 조회하는 것이 바람직함

## 11. 트랜잭션과 동시성 제어 패턴

### 11.2 DB 트랜잭션

- 트랜잭션 실행 시, 얼마나 많은 statement가 포함 될지를 고민해야한다.
- ACID
	- Atomicity : 원자성
		- 롤백이 반드시 필요
	- Consistency
		- 일관성
		- 컬럼 타입, 길이, 널
	- Isolation
	- Durability

- DB는 어떻게 동작하는가?
	- Undo Log
		- 롤백 시 사용되는 로그
		- undo log를 다시 적용하면 이전 상태로 돌아갈 수 있음
	- Storage Engine
	- Redo Log
		- 동기화 전에 DB가 중단되었다면, 완료된 트랜잭션 이후를 Redo Log에서 읽어서 동기화를 진행햐야 함

### 11.3 Isolation

- Locking
	- Lock만으로는 충돌을 피하기는 부족하다.
	- Read/Shared Lock 읽기는 허용, 쓰기 불가
	- WRite/Exclusive Lock 읽기 불가, 쓰기 불가

### 11.4 Pessimistic Locking (비관적인 잠금)

- 명시적인 물리적 잠금 모드
	- Oracle
		- Shared - FOR UPDATE
		- Exclusive - FOR UPDATE
			- 해당 트랜잭션을 커밋할 때 까지, 다른 사용자가 읽기를 제한함
	- MySQL
		- Shared - LOCK IN SHARED MODE
		- Exclusive - FOR UPDATE
			- for update로 행 단위의 잠금을 걸 때, 해당 테이블에 insert가 불가능
	- PostgreSQL
		- Shared - FOR SHARE
		- Exclusive - FOR UPDATE

- 락 옵션
	- LockOption.NO_WAIT
		- 기다리지 않고, 빠른 예외
	- LockOption.SKIP_LOCKED
		- 이미 락이 걸린 로우는 넘어가고, 락이 걸리지 않은 로우만 락

### 11.5 Optimistic Locking (낙관적 잠금)

- timestamp
- version 등으로 잠금

- OS timestamp issues
	- 시간으로 관리할 경우 문제가 발생할 수 있음
	- 시간은 네트워크와 동기화 되기 때문에, 이전 시간을 반환할 수도 있음

- Version 기반 잠금
	- short 타입으로 해도 괜찮을까?
		- 65536번 수정하고, 수정되지 않은 것처럼 동작할 수 있긴 함

- 사용사례
	- 이미 다른 사람이 물품을 구매해감 version : 1 => 2로 변경
	- 나도 version : 1에서 수량이 남은 것을 보고 구매시도
		- 이미 물품이 판매되었고, version : 2로 변경되었기 때문에, 예외 발생 (OptimisticLockException)
- 게시굴 - 게시글 조회수
	- 게시글 조회수는 자주 변경되는 데이터임으로 테이블 분리
	- 조회수에만 락을 걸어서, 업데이트하는 방식으로
	- 궁금한 점 
		- 실패해도 예외를 던지면 별로 좋지 않을 것 같다..
		- retry?
			- 이 것도 엄청나게 조회를 시도하면, 계속 실패할 수 있지 않을까..


## 12. 최적의 캐싱 방법

### 12.1 DB 캐싱

- 캐시 동기화 전략 - Read through
	- 애플리케이션 - 캐시 - 캐시 저장소 - 데이터베이스
	- 예를 들어, 애플리케이션은 cache.get()만 호출하면, 캐시에서 가져오든 DB에서 가져오든 엔티티를 가져올 수 있게 된다.
- Write-behind
	- Cache.update()를 호출한 결과를 queue에 쌓아둠
	- Cache.flush() 호출 시, 버퍼링한 결과들을 모두 DB에 업데이트

- 데이터베이스와 OS 캐시
	- 많은 DB 엔진들은 읽기, 쓰기동작의 속도를 위해 내부 캐시 매커니즘을 사용한다.
	- 가장 흔한 DB 캐시 컴포넌트로는 인메모리 버퍼 풀이 있다.
		- 다른 캐시 컴포넌트로는 실행 계획 캐시, 쿼리 결과 버퍼등이 있다.
	- DB에서 쓰기가 발생할 때 마다 디스크에 쓴다면, I/O 작업이 엄청나게 될 것이다.
		- 5분마다 주기적으로 flush
		- 이 정보를 redo log가 보관하여, flush
		- DB 시스템이 다운되어도, redo log를 통해 복원
	- Oracle
		- PGA(Program Global Area)
			- ORDER BY
			- Window Functions
			- Hash Join
		- SGA(System Global Area)
	- PostgreSQL
		- 운영체제 캐시에 의존
		- Shared Buffer - OS Cache - Disk Drive
		- 공유버퍼와, OS Cache 모두에 저장될 수도 있음
		- 그래서 Shared Buffer가 너무 클 수 없음
			- shared buffer를 OS RAM의 15 ~ 25% 정도만 사용
		- vacuum
			- 사용되지 않은 오래된 페이지, 데이터를 수집하여 재할당 될 수 있도록 함
			- MVCC 때문에 이 것이 필요함
			- 가비지 컬렉터처럼 동작
	- MySQL
		- Oracle과 비슷하게 동작
		- Direct I/O 사용
		- 버퍼풀을 80 ~ 90% 까지 사용 가능
		- SHOW ENGINE INNODB STATUS
			- InnoDB에 할당된 버퍼풀 사이즈를 알 수 있음
			- 캐시 히트율이 높은지 파악도 해볼 수 있다

### 12.2 애플리케이션 캐시

- 버퍼 풀은 데이터 페이지만 캐쉬 됨
	- 실행 계획등은 버퍼 풀에 캐시 되지 않으므로
	- 여러 테이블이 조인되는 쿼리는 CPU등의 자원을 많이 소모할 수 있음
	- 이를 위해 DB 캐시만으로는 충분하지 않다는 뜻도 됨
	- 애플리케이션 캐시를 사용!

- 스택 오버 플로우는 모든 데이터를 캐시
	- 읽기 전용으로 웹 사이트 운영 가능
	- DB 업그레이드 등 유지보수 가능

- Redis, Memcached, Hazelcast, Etcd, Ehcache, Aerospike


### 12. 3 하이버네이트 2차 캐시

- 왜 사용할까?
	- 단일 Primary Node, 4개의 복제 노드
	- 읽기 트래픽이 더 증가하면, 복제 노드를 더 증가시킴
	- 쓰기 부하가 커지면 단일 Primary Node의 부하가 계속 커질 수 있음
	- 단일 Primary Node의 하드웨어를 좋게하는 수직 확장으로 확장해야 함
	- 또 다른 방법으로 부하를 분산하기 위해 2차 캐시를 사용할 수 있음

- 하이버네이트는 다양한 2차 캐시 옵션을 제공함
	- 엔티티 캐시
	- 컬렉션 캐시
	- 쿼리 캐시
	- NaturalId 캐시

- 간단한 동작방식
	- 엔티티 조회
		- 1. Session 캐시에서 엔티티를 찾음
		- 2. 없다면 2차 캐시에서 엔티티를 찾음
		- 3. 없다면 데이터 소스에서 엔티티를 찾음
		- 4. 엔티티 반환

- 2차 캐시는 기본적으로 활성화 되어있음
	- 그런데 NoOp 처럼 동작함
	- RegionFactory를 명시적으로 제공해야 동작함
	- hibernate.cache.region.factory_class
		- ehcache
		- jcache

- 엔티티가 자동으로 캐시되지는 않음
	- org.hibernate.annotations.cache 애노테이션으로 캐시 전략을 제공해야 동작함

- 엔티티 캐시의 키
	- primary key가 KEY가 됨
	- 값은 loadedState, Object[], StandardCacheEntryImpl, 분해된 상태로 저장

- 객체 참조도 저장할 수 있음
	- hibernate.cache.use_reference_entries : true
	- 객체를 힙에 저장된 그대로 저장할 수 있음
	- 여러 제한 사항이 있음
		- 읽기 전용으로만 동작
		- @Cache(useage = CacheConcurrencyStrategy.READ_ONLY)
		- 엔티티 연관관계를 사용할 수 없고, 기본 속성만 사용할 수 있음
		- 이러한 제한 사항때문에 크게 유용하지는 않음

- 컬렉션 캐시
	- 연관관계의 컬렉션에 캐쉬 애노테이션을 사용할 수 있음

- 쿼리 캐시
	- hibernate.cache.use_query_cache : true
	- 기본값은 false
	- 모든 쿼리가 자동으로 캐시되지는 않음
	- JPA를 사용한다면 힌트를 사용해야함 
		- setHint("org.hibernate.cacheable", true)
	- DB에서 실행되는 캐시와 정확히 일치함
	- timestamp와 함께 저장
	- 엔티티 식별자를 저장
	- DTO 프로젝션에도 사용할 수 있음
		- 캐시값에는 로드된 상태들이 저장 됨

- NaturalID 캐시
	- 자연 ID = 비즈니스 키, 특정 컬럼으로 특정 레코드를 식별할 수 있음 값
	- 주민번호, UUID 등
	- NatrualID 로 Fetching 하면 2개의 쿼리가 호출 됨
		- 자연ID로 엔티티를 불러옴
		- 엔티티가 캐쉬되어 있지 않다면, Entity ID로 한 번더 조회 후, 캐시
			- 이 2번째 쿼리가 실행되지 않도록, 2차 캐시를 사용할 수 있음
	- ```java
	  @Entity(name = "post")
	  @Table(name = "post)
	  @org.hibernate.annotations.Cache(usage = CacheConcurrencyStrategy.READ_WRITE)
	  @NatrualIdCache
	  public class Post {
	  }
	  ```
	  - @NaturalIdCache 애노테이션을 사용하면, 첫 번째 쿼리(자연ID로 조회하는 쿼리)를 캐시할 수 있음

- 2차 캐시 - 캐시 동시성 전략
	- 이 솔루션을 사용하는 이유 => 강력한 일관성을 제공함
	- 