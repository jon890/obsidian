
## 저자

Vlad

## 시작하며

- 많은 애플리케이션에서 Data Access Layer 성능에서 많은 지연이 발생함
- Data Access Layer에서 필요한 응답시간은 여러가지를 포함한다
	1. Connection 획득
	2. Statement 실행
	3. ResultSet 추출
	4. Business Logic Idle TIme


### 로깅

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
	- P6Spy
