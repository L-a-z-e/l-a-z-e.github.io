---
title: Spring Batch - Database
description: 
author: laze
date: 2025-06-19 00:00:02 +0900
categories: [Dev, SpringBatch]
tags: [SpringBatch]
---
### 데이터베이스 (Database)

대부분의 엔터프라이즈 애플리케이션 스타일과 마찬가지로, 데이터베이스는 배치의 중앙 저장 메커니즘입니다.

그러나 배치는 시스템이 처리해야 하는 데이터셋의 엄청난 크기 때문에 다른 애플리케이션 스타일과 다릅니다.

만약 SQL 문이 100만 행을 반환한다면, 결과 집합(result set)은 모든 행이 읽힐 때까지 반환된 모든 결과를 메모리에 보관할 가능성이 높습니다.

Spring Batch는 이 문제에 대해 두 가지 유형의 솔루션을 제공합니다:

- 커서 기반 `ItemReader` 구현체 (Cursor-based `ItemReader` Implementations)
- 페이징 `ItemReader` 구현체 (Paging `ItemReader` Implementations)

### 커서 기반 ItemReader 구현체 (Cursor-based ItemReader Implementations)

데이터베이스 커서를 사용하는 것은 일반적으로 대부분의 배치 개발자의 기본 접근 방식입니다.

왜냐하면 이것이 관계형 데이터를 '스트리밍'하는 문제에 대한 데이터베이스의 솔루션이기 때문입니다.

Java `ResultSet` 클래스는 본질적으로 커서를 조작하기 위한 객체 지향 메커니즘입니다.

`ResultSet`은 현재 데이터 행에 대한 커서를 유지합니다.

`ResultSet`에서 `next`를 호출하면 이 커서가 다음 행으로 이동합니다.

Spring Batch 커서 기반 `ItemReader` 구현은 초기화 시 커서를 열고, `read`를 호출할 때마다 커서를 한 행씩 앞으로 이동시켜 처리할 수 있는 매핑된 객체를 반환합니다.

그런 다음 `close` 메서드가 호출되어 모든 리소스가 해제되도록 합니다.

Spring 코어의 `JdbcTemplate`은 콜백 패턴을 사용하여 `ResultSet`의 모든 행을 완전히 매핑하고 메서드 호출자에게 제어를 반환하기 전에 닫음으로써 이 문제를 해결합니다.

그러나 배치에서는 스텝이 완료될 때까지 기다려야 합니다.

다음 이미지는 커서 기반 `ItemReader`가 작동하는 방식에 대한 일반적인 다이어그램을 보여줍니다.

예제에서는 SQL이 널리 알려져 있기 때문에 SQL을 사용하지만, 어떤 기술이라도 기본 접근 방식을 구현할 수 있습니다.

**커서 예제**
![Desktop View](assets/img/batch-database-1.png){: width="972" height="589" }

이 예제는 기본 패턴을 보여줍니다.

ID, NAME, BAR 세 개의 열을 가진 'FOO' 테이블이 주어졌을 때, ID가 1보다 크고 7보다 작은 모든 행을 선택합니다.

이것은 커서의 시작(첫 번째 행)을 ID 2에 둡니다. 이 행의 결과는 완전히 매핑된 `Foo` 객체여야 합니다.

`read()`를 다시 호출하면 커서가 다음 행, 즉 ID가 3인 `Foo`로 이동합니다.

이러한 읽기의 결과는 각 `read` 후에 기록되어 객체가 가비지 컬렉션될 수 있도록 합니다(인스턴스 변수가 이에 대한 참조를 유지하지 않는다고 가정).

### JdbcCursorItemReader

`JdbcCursorItemReader`는 커서 기반 기술의 JDBC 구현입니다.

`ResultSet`과 직접 작동하며 `DataSource`에서 얻은 연결에 대해 실행할 SQL 문이 필요합니다.

다음 데이터베이스 스키마가 예제로 사용됩니다:

```sql
CREATE TABLE CUSTOMER (
   ID BIGINT IDENTITY PRIMARY KEY, -- IDENTITY는 DB 종류에 따라 AUTO_INCREMENT 등일 수 있음
   NAME VARCHAR(45),
   CREDIT FLOAT
);
```

많은 사람들이 각 행에 대해 도메인 객체를 사용하는 것을 선호하므로, 다음 예제에서는 `RowMapper` 인터페이스의 구현을 사용하여 `CustomerCredit` 객체를 매핑합니다:

```java
public class CustomerCreditRowMapper implements RowMapper<CustomerCredit> {

    public static final String ID_COLUMN = "id";
    public static final String NAME_COLUMN = "name";
    public static final String CREDIT_COLUMN = "credit";

    public CustomerCredit mapRow(ResultSet rs, int rowNum) throws SQLException {
        CustomerCredit customerCredit = new CustomerCredit();

        customerCredit.setId(rs.getInt(ID_COLUMN));
        customerCredit.setName(rs.getString(NAME_COLUMN));
        customerCredit.setCredit(rs.getBigDecimal(CREDIT_COLUMN)); // FLOAT 대신 BigDecimal 사용 권장

        return customerCredit;
    }
}
```

`JdbcCursorItemReader`는 `JdbcTemplate`과 주요 인터페이스를 공유하므로, `ItemReader`와 대조하기 위해 `JdbcTemplate`으로 이 데이터를 읽어들이는 예제를 보는 것이 유용합니다.

이 예제의 목적상, CUSTOMER 데이터베이스에 1,000개의 행이 있다고 가정합니다.

첫 번째 예제는 `JdbcTemplate`을 사용합니다:

```java
//단순화를 위해 dataSource가 이미 확보되었다고 가정
JdbcTemplate jdbcTemplate = new JdbcTemplate(dataSource);
List<CustomerCredit> customerCredits = jdbcTemplate.query("SELECT ID, NAME, CREDIT from CUSTOMER",
                                          new CustomerCreditRowMapper());
```

앞의 코드 스니펫을 실행한 후, `customerCredits` 리스트에는 1,000개의 `CustomerCredit` 객체가 포함됩니다.

`query` 메서드에서는 `DataSource`에서 연결을 얻고, 제공된 SQL을 실행하며, `ResultSet`의 각 행에 대해 `mapRow` 메서드가 호출됩니다.

이를 다음 예제에 표시된 `JdbcCursorItemReader`의 접근 방식과 대조해 보십시오:

```java
JdbcCursorItemReader<CustomerCredit> itemReader = new JdbcCursorItemReader<>(); // 제네릭 타입 명시
itemReader.setDataSource(dataSource);
itemReader.setSql("SELECT ID, NAME, CREDIT from CUSTOMER");
itemReader.setRowMapper(new CustomerCreditRowMapper());
int counter = 0;
ExecutionContext executionContext = new ExecutionContext();
itemReader.open(executionContext); // ItemStream 생명주기
CustomerCredit customerCredit; // 타입 명시, 초기화는 루프 내에서
while(true){ // customerCredit != null 조건은 루프 내에서 처리
    customerCredit = itemReader.read();
    if (customerCredit == null) { // null이면 루프 종료
        break;
    }
    counter++;
}
itemReader.close(); // ItemStream 생명주기
```

앞의 코드 스니펫을 실행한 후, `counter`는 1,000과 같습니다.

위 코드가 반환된 `customerCredit`을 리스트에 넣었다면 결과는 `JdbcTemplate` 예제와 정확히 동일했을 것입니다.

그러나 `ItemReader`의 큰 장점은 아이템을 '스트리밍'할 수 있다는 것입니다.

`read` 메서드를 한 번 호출하고, 아이템을 `ItemWriter`로 기록한 다음, `read`로 다음 아이템을 얻을 수 있습니다.

이를 통해 아이템 읽기 및 쓰기를 '청크'로 수행하고 주기적으로 커밋할 수 있으며, 이것이 고성능 배치 처리의 본질입니다. 또한, Spring Batch `Step`에 주입하기 위한 구성이 용이합니다.

다음 예제는 Java에서 `ItemReader`를 `Step`에 주입하는 방법을 보여줍니다:

**Java 설정 (Java Configuration)**

```java
@Bean
public JdbcCursorItemReader<CustomerCredit> itemReader(DataSource dataSource) { // DataSource 주입
	return new JdbcCursorItemReaderBuilder<CustomerCredit>()
			.dataSource(dataSource) // 주입받은 dataSource 사용
			.name("creditReader")
			.sql("select ID, NAME, CREDIT from CUSTOMER")
			.rowMapper(new CustomerCreditRowMapper())
			.build();
}
```

### 추가 속성들 (Additional Properties)

Java에서 커서를 열기 위한 매우 다양한 옵션이 있기 때문에, `JdbcCursorItemReader`에는 설정할 수 있는 많은 속성이 있으며, 다음 표에 설명되어 있습니다:

**표 1. JdbcCursorItemReader 속성**

| 속성 | 설명 |
| --- | --- |
| `ignoreWarnings` | SQLWarning이 기록되는지 또는 예외를 발생시키는지 여부를 결정합니다. 기본값은 true입니다 (즉, 경고가 기록됨). |
| `fetchSize` | `ItemReader`가 사용하는 `ResultSet` 객체에 더 많은 행이 필요할 때 데이터베이스에서 가져와야 하는 행 수에 대한 힌트를 JDBC 드라이버에 제공합니다. 기본적으로 힌트는 제공되지 않습니다. |
| `maxRows` | 기본 `ResultSet`이 한 번에 보유할 수 있는 최대 행 수를 제한합니다. |
| `queryTimeout` | `Statement` 객체가 실행되기를 드라이버가 기다리는 시간(초)을 설정합니다. 제한을 초과하면 `DataAccessException`이 발생합니다. (자세한 내용은 드라이버 공급업체 설명서 참조). |
| `verifyCursorPosition` | `ItemReader`가 보유한 동일한 `ResultSet`이 `RowMapper`에 전달되므로 사용자가 직접 `ResultSet.next()`를 호출할 수 있으며, 이는 리더의 내부 개수에 문제를 일으킬 수 있습니다. 이 값을 true로 설정하면 `RowMapper` 호출 후 커서 위치가 이전과 동일하지 않은 경우 예외가 발생합니다. |
| `saveState` | `ItemStream#update(ExecutionContext)`에서 제공하는 `ExecutionContext`에 리더의 상태를 저장해야 하는지 여부를 나타냅니다. 기본값은 true입니다. |
| `driverSupportsAbsolute` | JDBC 드라이버가 `ResultSet`에서 절대 행을 설정하는 것을 지원하는지 여부를 나타냅니다. `ResultSet.absolute()`를 지원하는 JDBC 드라이버의 경우 이 값을 true로 설정하는 것이 좋습니다. 특히 대용량 데이터셋으로 작업하는 동안 스텝이 실패할 경우 성능이 향상될 수 있습니다. 기본값은 false입니다. |
| `setUseSharedExtendedConnection` | 커서에 사용되는 연결을 다른 모든 처리에서도 사용해야 하는지, 즉 동일한 트랜잭션을 공유해야 하는지 여부를 나타냅니다. 이 값을 false로 설정하면 커서는 자체 연결로 열리고 스텝 처리의 나머지 부분에 대해 시작된 어떤 트랜잭션에도 참여하지 않습니다. 이 플래그를 true로 설정하면 각 커밋 후 연결이 닫히고 해제되는 것을 방지하기 위해 `DataSource`를 `ExtendedConnectionDataSourceProxy`로 래핑해야 합니다. 이 옵션을 true로 설정하면 커서를 여는 데 사용되는 문은 'READ_ONLY' 및 'HOLD_CURSORS_OVER_COMMIT' 옵션으로 생성됩니다. 이를 통해 스텝 처리에서 수행되는 트랜잭션 시작 및 커밋을 넘어 커서를 열어 둘 수 있습니다. 이 기능을 사용하려면 이를 지원하는 데이터베이스와 JDBC 3.0 이상을 지원하는 JDBC 드라이버가 필요합니다. 기본값은 false입니다. |

### StoredProcedureItemReader

때로는 저장 프로시저를 사용하여 커서 데이터를 얻어야 합니다.

`StoredProcedureItemReader`는 `JdbcCursorItemReader`처럼 작동하지만, 커서를 얻기 위해 쿼리를 실행하는 대신 커서를 반환하는 저장 프로시저를 실행한다는 점이 다릅니다.

저장 프로시저는 세 가지 다른 방식으로 커서를 반환할 수 있습니다:

1. 반환된 `ResultSet`으로 (SQL Server, Sybase, DB2, Derby, MySQL에서 사용).
2. out 매개변수로 반환된 ref-cursor로 (Oracle, PostgreSQL에서 사용).
3. 저장 함수 호출의 반환 값으로.

```java
@Bean
public StoredProcedureItemReader<CustomerCredit> reader(DataSource dataSource) { // 제네릭 타입 및 DataSource 주입
	StoredProcedureItemReader<CustomerCredit> reader = new StoredProcedureItemReader<>();

	reader.setDataSource(dataSource);
	reader.setProcedureName("sp_customer_credit");
	reader.setRowMapper(new CustomerCreditRowMapper());

	return reader;
}
```

앞의 예제는 반환된 결과로 `ResultSet`을 제공하는 저장 프로시저에 의존합니다 (이전의 옵션 1).

저장 프로시저가 ref-cursor를 반환한 경우(옵션 2), 반환된 ref-cursor인 out 매개변수의 위치를 제공해야 합니다.

```java
@Bean
public StoredProcedureItemReader<CustomerCredit> reader(DataSource dataSource) {
	StoredProcedureItemReader<CustomerCredit> reader = new StoredProcedureItemReader<>();

	reader.setDataSource(dataSource);
	reader.setProcedureName("sp_customer_credit");
	reader.setRowMapper(new CustomerCreditRowMapper());
	reader.setRefCursorPosition(1); // 첫 번째 파라미터가 ref-cursor임을 명시

	return reader;
}
```

커서가 저장 함수에서 반환된 경우(옵션 3), "function" 속성을 true로 설정해야 합니다. 기본값은 false입니다.

다음 예제는 Java에서 속성을 true로 설정하는 것을 보여줍니다:

```java
@Bean
public StoredProcedureItemReader<CustomerCredit> reader(DataSource dataSource) {
	StoredProcedureItemReader<CustomerCredit> reader = new StoredProcedureItemReader<>();

	reader.setDataSource(dataSource);
	reader.setProcedureName("sp_customer_credit");
	reader.setRowMapper(new CustomerCreditRowMapper());
	reader.setFunction(true); // 저장 함수 호출임을 명시

	return reader;
}
```

이 모든 경우에, `RowMapper`뿐만 아니라 `DataSource` 및 실제 프로시저 이름도 정의해야 합니다.

저장 프로시저나 함수가 매개변수를 받는 경우, `parameters` 속성을 사용하여 선언하고 설정해야 합니다.

Oracle에 대한 다음 예제는 세 개의 매개변수를 선언합니다.

첫 번째는 ref-cursor를 반환하는 out 매개변수이고, 두 번째와 세 번째는 `INTEGER` 유형의 값을 받는 in 매개변수입니다.

다음 예제는 Java에서 매개변수로 작업하는 방법을 보여줍니다:

```java
@Bean
public StoredProcedureItemReader<CustomerCredit> reader(DataSource dataSource, RowMapper<CustomerCredit> rowMapper,
                                                     PreparedStatementSetter parameterSetter) { // 의존성 주입
	List<SqlParameter> parameters = new ArrayList<>();
	parameters.add(new SqlOutParameter("newId", OracleTypes.CURSOR)); // Oracle의 경우 Types.REF_CURSOR 또는 OracleTypes.CURSOR
	parameters.add(new SqlParameter("amount", Types.INTEGER));
	parameters.add(new SqlParameter("custId", Types.INTEGER));

	StoredProcedureItemReader<CustomerCredit> reader = new StoredProcedureItemReader<>();

	reader.setDataSource(dataSource);
	reader.setProcedureName("spring.cursor_func"); // 프로시저 이름 예시
	reader.setParameters(parameters.toArray(new SqlParameter[0])); // SqlParameter 배열로 변환
	reader.setRefCursorPosition(1); // 또는 SqlParameter 정의 순서에 따라 조정
	reader.setRowMapper(rowMapper); // 주입받은 RowMapper 사용
	reader.setPreparedStatementSetter(parameterSetter); // 주입받은 PreparedStatementSetter 사용

	return reader;
}

// rowMapper()와 parameterSetter()에 대한 빈 정의 필요
// @Bean public RowMapper<CustomerCredit> rowMapper() { return new CustomerCreditRowMapper(); }
// @Bean public PreparedStatementSetter parameterSetter() { /* ... 구현 ... */ }
```

매개변수 선언 외에도, 호출에 대한 매개변수 값을 설정하는 `PreparedStatementSetter` 구현을 지정해야 합니다. 이것은 위의 `JdbcCursorItemReader`와 동일하게 작동합니다. 추가 속성 목록에 나열된 모든 추가 속성은 `StoredProcedureItemReader`에도 적용됩니다.

### 페이징 ItemReader 구현체 (Paging ItemReader Implementations)

데이터베이스 커서를 사용하는 대안은 각 쿼리가 결과의 일부를 가져오는 여러 쿼리를 실행하는 것입니다. 이

부분을 페이지라고 합니다.

각 쿼리는 시작 행 번호와 페이지에서 반환하려는 행 수를 지정해야 합니다.

### JdbcPagingItemReader

페이징 `ItemReader`의 한 가지 구현은 `JdbcPagingItemReader`입니다.

`JdbcPagingItemReader`에는 페이지를 구성하는 행을 검색하는 데 사용되는 SQL 쿼리를 제공하는 `PagingQueryProvider`가 필요합니다.

각 데이터베이스는 페이징 지원을 제공하기 위한 자체 전략을 가지고 있으므로, 지원되는 각 데이터베이스 유형에 대해 다른 `PagingQueryProvider`를 사용해야 합니다.

또한 사용 중인 데이터베이스를 자동 감지하고 적절한 `PagingQueryProvider` 구현을 결정하는 `SqlPagingQueryProviderFactoryBean`도 있습니다.

이것은 구성을 단순화하며 권장되는 모범 사례입니다.

`SqlPagingQueryProviderFactoryBean`을 사용하려면 select 절과 from 절을 지정해야 합니다.

선택적으로 where 절을 제공할 수도 있습니다.

이러한 절과 필수 `sortKey`는 SQL 문을 작성하는 데 사용됩니다.

실행 간에 데이터 손실이 없도록 `sortKey`에 고유 키 제약 조건이 있는 것이 중요합니다.

리더가 열린 후에는 다른 `ItemReader`와 마찬가지로 `read`를 호출할 때마다 하나의 아이템을 반환합니다. 추가 행이 필요할 때 페이징은 백그라운드에서 발생합니다.

다음 Java 예제 구성은 이전에 표시된 커서 기반 `ItemReader`와 유사한 'customer credit' 예제를 사용합니다:

```java
@Bean
public JdbcPagingItemReader<CustomerCredit> itemReader(DataSource dataSource, PagingQueryProvider queryProvider,
                                                    RowMapper<CustomerCredit> customerCreditMapper) { // 의존성 주입
	Map<String, Object> parameterValues = new HashMap<>();
	parameterValues.put("status", "NEW"); // 쿼리 파라미터 예시

	return new JdbcPagingItemReaderBuilder<CustomerCredit>()
           				.name("creditReader")
           				.dataSource(dataSource)
           				.queryProvider(queryProvider) // PagingQueryProvider 설정
           				.parameterValues(parameterValues) // 쿼리 파라미터 설정
           				.rowMapper(customerCreditMapper) // RowMapper 설정
           				.pageSize(1000) // 한 페이지당 읽어올 아이템 수
           				.build();
}

@Bean
public SqlPagingQueryProviderFactoryBean queryProvider(DataSource dataSource) { // DataSource 주입 필요
	SqlPagingQueryProviderFactoryBean provider = new SqlPagingQueryProviderFactoryBean();

  provider.setDataSource(dataSource); // DataSource 설정 필수
	provider.setSelectClause("select id, name, credit");
	provider.setFromClause("from customer");
	provider.setWhereClause("where status=:status");
	provider.setSortKey("id"); // 정렬 기준 키 (고유해야 함)

	return provider;
}

// customerCreditMapper()에 대한 빈 정의 필요
// @Bean public RowMapper<CustomerCredit> customerCreditMapper() { return new CustomerCreditRowMapper(); }
```

이 구성된 `ItemReader`는 지정해야 하는 `RowMapper`를 사용하여 `CustomerCredit` 객체를 반환합니다.

'pageSize' 속성은 각 쿼리 실행에 대해 데이터베이스에서 읽어들이는 엔티티 수를 결정합니다.

'parameterValues' 속성은 쿼리에 대한 매개변수 값의 `Map`을 지정하는 데 사용할 수 있습니다.

where 절에서 명명된 매개변수를 사용하는 경우 각 항목의 키는 명명된 매개변수의 이름과 일치해야 합니다.

전통적인 '?' 플레이스홀더를 사용하는 경우 각 항목의 키는 1부터 시작하는 플레이스홀더의 번호여야 합니다.

### JpaPagingItemReader

페이징 `ItemReader`의 또 다른 구현은 `JpaPagingItemReader`입니다.

JPA에는 Hibernate `StatelessSession`과 유사한 개념이 없으므로 JPA 사양에서 제공하는 다른 기능을 사용해야 합니다.

JPA는 페이징을 지원하므로 배치 처리에 JPA를 사용할 때 이것은 자연스러운 선택입니다.

각 페이지를 읽은 후 엔티티는 분리(detached)되고 영속성 컨텍스트(persistence context)는 지워져 페이지가 처리되면 엔티티가 가비지 컬렉션될 수 있도록 합니다.

`JpaPagingItemReader`를 사용하면 JPQL 문을 선언하고 `EntityManagerFactory`를 전달할 수 있습니다.

그런 다음 다른 `ItemReader`와 마찬가지로 `read`를 호출할 때마다 하나의 아이템을 반환합니다. 추가 엔티티가 필요할 때 페이징은 백그라운드에서 발생합니다.

다음 Java 예제 구성은 이전에 표시된 JDBC 리더와 유사한 'customer credit' 예제를 사용합니다:

```java
@Bean
public JpaPagingItemReader<CustomerCredit> itemReader(EntityManagerFactory entityManagerFactory) { // EntityManagerFactory 주입
	return new JpaPagingItemReaderBuilder<CustomerCredit>()
           				.name("creditReader")
           				.entityManagerFactory(entityManagerFactory) // 주입받은 EntityManagerFactory 사용
           				.queryString("select c from CustomerCredit c") // JPQL 쿼리
           				.pageSize(1000) // 한 페이지당 읽어올 아이템 수
           				.build();
}
```

이 구성된 `ItemReader`는 `CustomerCredit` 객체에 올바른 JPA 어노테이션 또는 ORM 매핑 파일이 있다고 가정하고 위에서 `JdbcPagingItemReader`에 대해 설명한 것과 정확히 동일한 방식으로 `CustomerCredit` 객체를 반환합니다.

'pageSize' 속성은 각 쿼리 실행에 대해 데이터베이스에서 읽어들이는 엔티티 수를 결정합니다.

### 데이터베이스 ItemWriters (Database ItemWriters)

플랫 파일과 XML 파일 모두 특정 `ItemWriter` 인스턴스를 가지고 있지만, 데이터베이스 세계에는 정확히 동등한 것이 없습니다.

이는 트랜잭션이 필요한 모든 기능을 제공하기 때문입니다.

`ItemWriter` 구현은 파일에 필요합니다.

왜냐하면 파일은 트랜잭션처럼 작동해야 하며, 기록된 아이템을 추적하고 적절한 시기에 플러시하거나 지워야 하기 때문입니다.

데이터베이스는 쓰기가 이미 트랜잭션에 포함되어 있으므로 이러한 기능이 필요하지 않습니다.

사용자는 `ItemWriter` 인터페이스를 구현하는 자체 DAO를 만들거나 일반적인 처리 문제를 위해 작성된 사용자 정의 `ItemWriter` 중 하나를 사용할 수 있습니다.

어느 쪽이든 문제없이 작동해야 합니다.

한 가지 주의할 점은 출력을 일괄 처리(batching)함으로써 제공되는 성능 및 오류 처리 기능입니다.

이것은 Hibernate를 `ItemWriter`로 사용할 때 가장 일반적이지만 JDBC 배치 모드를 사용할 때도 동일한 문제가 발생할 수 있습니다.

데이터베이스 출력 일괄 처리는 데이터를 플러시하고 데이터에 오류가 없는지 신중하게 확인한다면 본질적인 결함이 없습니다.

그러나 쓰는 동안 발생하는 모든 오류는 어떤 개별 아이템이 예외를 일으켰는지 또는 어떤 개별 아이템이 책임이 있었는지 알 수 없기 때문에 혼란을 야기할 수 있으며, 다음 이미지에 설명되어 있습니다:

**플러시 시 오류**
[이미지 설명: 여러 아이템(Item 1, Item 2, ..., Item N)이 버퍼에 쌓여 있다가, Session.flush() 시점에 한꺼번에 DB로 전송될 때, 그중 특정 아이템(예: Item 15)에서 DataIntegrityViolationException이 발생하면, 어떤 아이템이 문제였는지 알기 어렵고 전체 트랜잭션이 롤백됨을 보여주는 다이어그램.]
*그림 2. 플러시 시 오류*

![Desktop View](assets/img/batch-database-2.png){: width="972" height="589" }

아이템이 기록되기 전에 버퍼링되면, 커밋 직전에 버퍼가 플러시될 때까지 오류가 발생하지 않습니다.

예를 들어, 청크당 20개의 아이템이 기록되고 15번째 아이템이 `DataIntegrityViolationException`을 발생시킨다고 가정합니다.

`Step`의 관점에서 보면, 실제로 기록될 때까지 오류가 발생하는지 알 수 있는 방법이 없으므로 20개의 아이템 모두 성공적으로 기록된 것으로 간주됩니다.

`Session#flush()`가 호출되면 버퍼가 비워지고 예외가 발생합니다.

이 시점에서 `Step`이 할 수 있는 일은 아무것도 없습니다.

트랜잭션은 롤백되어야 합니다. 일반적으로 이 예외는 아이템을 건너뛰게 할 수 있으며(skip/retry 정책에 따라 다름), 그런 다음 다시 기록되지 않습니다. 그러나 일괄 처리 시나리오에서는 어떤 아이템이 문제를 일으켰는지 알 수 있는 방법이 없습니다.

실패가 발생했을 때 전체 버퍼가 기록되고 있었습니다. 이 문제를 해결하는 유일한 방법은 다음 이미지와 같이 각 아이템 후에 플러시하는 것입니다:

**쓰기 시 오류**
[이미지 설명: 각 아이템(Item 1, Item 2, ...)이 개별적으로 DB에 쓰여지고(Write), 각 쓰기 작업 후 즉시 플러시(Flush)될 때, Item 15에서 오류가 발생하면 해당 아이템만 문제가 되고 다른 아이템들은 영향을 덜 받거나, 문제 아이템을 특정하기 쉬움을 보여주는 다이어그램.]
*그림 3. 쓰기 시 오류*

![Desktop View](assets/img/batch-database-3.png){: width="972" height="589" }

이것은 특히 Hibernate를 사용할 때 일반적인 사용 사례이며, `ItemWriter` 구현에 대한 간단한 지침은 `write()`를 호출할 때마다 플러시하는 것입니다.

그렇게 하면 아이템을 안정적으로 건너뛸 수 있으며, Spring Batch는 내부적으로 오류 후 `ItemWriter` 호출의 세분성을 처리합니다.

---

### **학습 목표 제시**

이번 "Database" 챕터는 내용이 매우 방대하여 여러 핵심 목표를 가집니다:

1. **데이터베이스 기반 `ItemReader`의 두 가지 주요 접근 방식(커서, 페이징) 이해:** 대용량 데이터베이스 처리를 위한 커서 방식과 페이징 방식의 차이점, 장단점, 그리고 각각의 사용 시나리오를 파악합니다.
2. **`JdbcCursorItemReader` 및 `StoredProcedureItemReader` 사용법 숙지:** JDBC를 사용하여 데이터베이스 커서로부터 데이터를 읽어오는 방법과 저장 프로시저를 통해 데이터를 읽어오는 방법을 이해하고, `RowMapper`의 역할과 주요 설정 속성들을 학습합니다.
3. **`JdbcPagingItemReader` 및 `JpaPagingItemReader` 사용법 숙지:** JDBC 또는 JPA를 사용하여 데이터베이스로부터 페이징 방식으로 데이터를 읽어오는 방법을 이해하고, `PagingQueryProvider` (JDBC의 경우) 또는 JPQL (JPA의 경우) 설정 및 `pageSize`의 의미를 학습합니다.
4. **데이터베이스 `ItemWriter`의 특징 및 고려사항 이해:** 데이터베이스 쓰기 시 별도의 전용 `ItemWriter`가 없는 이유(트랜잭션 활용)와, 일괄 처리(batching) 시 발생할 수 있는 오류 처리 문제 및 플러시 전략의 중요성을 파악합니다.

---

### **핵심 개념 설명**

이 챕터의 핵심은 **"대용량의 데이터베이스 데이터를 어떻게 효율적이고 안정적으로 읽고 쓸 것인가?"** 입니다.

이를 위해 Spring Batch는 크게 **커서(Cursor) 기반** 방식과 **페이징(Paging) 기반** 방식의 `ItemReader`를 제공하며, `ItemWriter`에 대한 특별한 고려사항도 다룹니다.

### 1. 왜 데이터베이스 처리가 특별한가? (대용량 데이터의 도전)

- 일반적인 웹 애플리케이션과 달리, 배치 애플리케이션은 수백만, 수천만 건의 데이터를 한 번에 처리해야 할 수 있습니다.
- 만약 SQL 쿼리의 결과를 한 번에 메모리로 다 가져오려고 한다면? (`JdbcTemplate`의 일반적인 `query()` 메서드처럼) -> **메모리 부족 (Out Of Memory Error, OOME)** 발생!
- Spring Batch는 이러한 문제를 해결하기 위해 데이터를 "스트리밍" 방식으로 조금씩 가져와 처리하는 방법을 제공합니다.

### 2. 데이터베이스 `ItemReader`: 두 가지 주요 전략

Spring Batch는 데이터베이스에서 데이터를 읽어오는 `ItemReader`에 대해 크게 두 가지 접근 방식을 제공합니다.

**A. 커서 기반 `ItemReader` (Cursor-based ItemReaders)**

- **개념:** 데이터베이스의 **커서(Cursor)**를 사용하여 한 번에 모든 결과를 메모리에 로드하는 대신, 한 번에 한 행(row)씩 데이터를 가져오는 방식입니다.
  - **커서란?** SQL 쿼리 결과 집합 내의 특정 위치를 가리키는 포인터와 같습니다. `ResultSet.next()`를 호출할 때마다 이 커서가 다음 행으로 이동하며 데이터를 가져옵니다.
- **장점:**
  - **메모리 효율성:** 한 번에 하나의 행(또는 JDBC 드라이버의 `fetchSize`만큼의 작은 묶음)만 메모리에 유지하므로 대용량 데이터 처리에 매우 적합합니다.
  - **간단함:** 전통적인 JDBC 프로그래밍 방식과 유사하여 이해하기 쉽습니다.
- **단점:**
  - **데이터베이스 연결 유지:** 커서가 열려 있는 동안에는 데이터베이스 연결이 계속 유지되어야 합니다. 매우 긴 시간 동안 실행되는 잡의 경우 DB 연결 유지에 부담이 될 수 있습니다.
  - **상태 저장 및 재시작 복잡성 (경우에 따라):** 스크롤 가능한 커서(scrollable cursor)를 사용하지 않는 한, 일반적으로 앞으로만 이동합니다. 재시작 시 정확히 중단된 위치부터 다시 시작하려면 추가적인 고려가 필요할 수 있습니다. (Spring Batch는 `saveState=true`일 때 이를 지원하려고 노력합니다.)
- **동작 원리 (그림 1. Cursor Example 참고):**
  1. `ItemReader` 초기화 시(`open()`) DB에 연결하고 SQL 쿼리를 실행하여 커서를 엽니다.
  2. `read()` 메서드가 호출될 때마다 `ResultSet.next()`를 호출하여 커서를 다음 행으로 이동시키고, 해당 행의 데이터를 `RowMapper`를 통해 객체로 변환하여 반환합니다.
  3. 더 이상 읽을 행이 없으면 `read()`는 `null`을 반환합니다.
  4. 스텝 종료 시(`close()`) 커서와 DB 연결을 닫습니다.
- **핵심은 "스트리밍"**: `JdbcTemplate`이 모든 결과를 List로 받아오는 것과 달리, `JdbcCursorItemReader`는 한 번에 하나씩 데이터를 흘려보내 줍니다.

**B. 페이징 기반 `ItemReader` (Paging ItemReaders)**

- **개념:** 전체 결과를 한 번에 가져오는 대신, 여러 번의 쿼리를 실행하여 각각의 쿼리가 결과의 일부, 즉 **페이지(page)** 단위로 데이터를 가져오는 방식입니다.
  - 각 쿼리는 "몇 번째 페이지부터 몇 개의 데이터를 가져와라" (예: "1번 페이지부터 1000개", "2번 페이지부터 1000개") 와 같이 동작합니다.
- **장점:**
  - **상태 비저장(Stateless)에 가까움:** 각 페이지를 가져올 때마다 새로운 쿼리가 실행되므로, 커서처럼 장시간 DB 연결을 유지할 필요가 없습니다. (물론, 페이지 정보를 기억해야 하므로 완전히 상태 비저장은 아님)
  - **병렬 처리 용이성 (경우에 따라):** 각 페이지를 독립적으로 처리하는 병렬 처리 시나리오에 더 적합할 수 있습니다.
  - **재시작 용이성:** 마지막으로 성공한 페이지 번호만 기억하면 되므로 재시작 로직이 상대적으로 간단할 수 있습니다.
- **단점:**
  - **여러 번의 쿼리 실행:** 데이터를 가져오기 위해 DB에 여러 번 쿼리를 날려야 하므로, 쿼리 실행 비용이 누적될 수 있습니다. (DB 부하 증가 가능성)
  - **정렬 키(Sort Key)의 중요성:** 페이지 간 데이터 중복이나 누락을 방지하려면, 쿼리 결과가 일관되게 정렬되어야 하고, 이 정렬 기준이 되는 키(`sortKey`)는 **고유(unique)**해야 합니다. 그렇지 않으면 데이터 유실 또는 중복 발생 가능성이 있습니다.
- **동작 원리:**
  1. `ItemReader` 초기화 시 첫 페이지를 가져오기 위한 정보를 설정합니다.
  2. `read()` 메서드가 호출될 때, 현재 페이지에 아직 읽지 않은 데이터가 있으면 해당 데이터를 `RowMapper` (JDBC) 또는 JPA 매핑을 통해 객체로 변환하여 반환합니다.
  3. 현재 페이지의 모든 데이터를 다 읽으면, 다음 페이지를 가져오기 위한 새로운 쿼리를 실행하여 다음 페이지 데이터를 로드합니다.
  4. 더 이상 가져올 페이지가 없으면 `read()`는 `null`을 반환합니다.

### 3. 커서 기반 `ItemReader` 구현체들

**A. `JdbcCursorItemReader<T>`**

- **역할:** 표준 JDBC API를 사용하여 데이터베이스 커서로부터 데이터를 읽어오는 가장 기본적인 커서 기반 리더입니다.
- **주요 설정:**
  - `dataSource` (DataSource): DB 연결을 위한 `DataSource`.
  - `sql` (String): 실행할 SQL 쿼리 문자열.
  - `rowMapper` (RowMapper<T>): `ResultSet`의 각 행을 `T` 타입 객체로 변환하는 `RowMapper` 구현체.
    - `CustomerCreditRowMapper` 예제: `ResultSet`에서 "id", "name", "credit" 컬럼 값을 읽어 `CustomerCredit` 객체를 생성하고 필드를 채웁니다.
- **`JdbcTemplate`과의 비교:**
  - `JdbcTemplate.query()`: 모든 결과를 `List`로 반환 (메모리 부담).
  - `JdbcCursorItemReader.read()`: 한 번에 하나의 객체만 반환 (스트리밍, 메모리 효율적).
- **추가 속성들 (문서 표 1. JdbcCursorItemReader Properties 참고):**
  - `fetchSize`: JDBC 드라이버에게 한 번에 DB에서 가져올 행 수에 대한 힌트. 성능에 영향.
  - `maxRows`: ResultSet이 가질 수 있는 최대 행 수 제한.
  - `queryTimeout`: 쿼리 실행 시간 제한.
  - `verifyCursorPosition`: `RowMapper` 내에서 `ResultSet.next()`를 임의로 호출하는 것을 방지하기 위한 검증.
  - `saveState`: 재시작을 위해 `ExecutionContext`에 현재 상태(읽은 행 수 등)를 저장할지 여부 (기본값 `true`).
  - `driverSupportsAbsolute`: JDBC 드라이버가 `ResultSet.absolute()` (특정 행으로 직접 이동)를 지원하는지 여부. 재시작 성능 향상에 도움.
  - `setUseSharedExtendedConnection`: 커서용 연결을 스텝의 다른 작업과 공유할지, 아니면 별도 연결을 사용할지 결정. 복잡한 트랜잭션 관리 시 고려.

**B. `StoredProcedureItemReader<T>`**

- **역할:** SQL 쿼리 대신 **저장 프로시저(Stored Procedure)**를 실행하여 그 결과로 반환되는 커서(ResultSet)로부터 데이터를 읽어옵니다.
- **커서 반환 방식 (DB마다 다름):**
  1. 저장 프로시저가 직접 `ResultSet`을 반환 (SQL Server, Sybase, DB2, Derby, MySQL 등).
  2. OUT 파라미터를 통해 **REF CURSOR** 반환 (Oracle, PostgreSQL 등).
  3. 저장 **함수(Stored Function)**의 반환 값으로 커서 반환.
- **주요 설정:**
  - `dataSource` (DataSource)
  - `procedureName` (String): 호출할 저장 프로시저/함수 이름.
  - `rowMapper` (RowMapper<T>)
  - `refCursorPosition` (int): REF CURSOR 방식일 때, 몇 번째 OUT 파라미터가 커서인지를 지정 (1부터 시작).
  - `function` (boolean): 저장 함수를 호출하는 경우 `true`로 설정 (기본값 `false`).
  - `parameters` (SqlParameter[]): 저장 프로시저/함수에 전달할 IN/OUT 파라미터 정보 (`SqlParameter`, `SqlOutParameter` 객체 사용).
  - `preparedStatementSetter` (PreparedStatementSetter): IN 파라미터에 실제 값을 설정하는 로직 구현.

### 4. 페이징 기반 `ItemReader` 구현체들

**A. `JdbcPagingItemReader<T>`**

- **역할:** JDBC를 사용하여 여러 번의 쿼리를 통해 페이지 단위로 데이터를 읽어옵니다.
- **주요 설정:**
  - `dataSource` (DataSource)
  - `queryProvider` (PagingQueryProvider): 각 페이지를 가져오기 위한 SQL 쿼리를 생성하는 역할. DB 종류별로 페이징 쿼리 구문이 다르므로 중요.
    - **`SqlPagingQueryProviderFactoryBean`**: DB 타입을 자동 감지하여 적절한 `PagingQueryProvider` 구현체를 생성해주는 팩토리 빈 (권장).
      - `setSelectClause(String)`: `SELECT id, name, ...` 부분.
      - `setFromClause(String)`: `FROM customer` 부분.
      - `setWhereClause(String)`: (선택) `WHERE status = :status` 부분.
      - `setSortKey(String)`: **페이징의 기준이 되는 정렬 키 컬럼명. 반드시 고유(unique)해야 데이터 누락/중복 방지.**
  - `parameterValues` (Map<String, Object>): `WHERE` 절 등에 사용될 파라미터 값.
  - `rowMapper` (RowMapper<T>)
  - `pageSize` (int): 한 페이지당 읽어올 아이템(행)의 수.

**B. `JpaPagingItemReader<T>`**

- **역할:** JPA(Java Persistence API)를 사용하여 페이지 단위로 엔티티를 읽어옵니다.
- **주요 설정:**
  - `entityManagerFactory` (EntityManagerFactory): JPA `EntityManager`를 생성하기 위한 팩토리.
  - `queryString` (String): 실행할 JPQL(Java Persistence Query Language) 쿼리 문자열. (예: `SELECT c FROM CustomerCredit c ORDER BY c.id`)
  - `pageSize` (int): 한 페이지당 읽어올 엔티티 수.
  - `parameterValues` (Map<String, Object>): (선택) JPQL 쿼리에 전달할 파라미터 값.
- **특징:** 각 페이지를 읽은 후, 읽어온 엔티티들은 영속성 컨텍스트에서 분리(detach)되고 컨텍스트는 비워집니다. 이는 가비지 컬렉션이 이전 페이지의 엔티티들을 수거할 수 있도록 하여 메모리 사용량을 관리합니다.

### 5. 데이터베이스 `ItemWriter`에 대한 고찰

- **전용 `ItemWriter`가 없는 이유?:** 플랫 파일이나 XML과는 달리, 데이터베이스 쓰기를 위한 `JdbcBatchItemWriter` 같은 "전용" `ItemWriter`가 Spring Batch 코어에 풍부하게 제공되지는 않습니다. (Spring Batch Extensions 등에는 일부 존재)
  - **이유:** 데이터베이스 작업은 이미 **트랜잭션**이라는 강력한 메커니즘 위에서 동작하기 때문입니다. `ItemWriter`의 주된 역할 중 하나는 파일 쓰기 등을 "마치 트랜잭션처럼" 동작하게 만드는 것(상태 추적, 플러시 등)인데, DB는 이미 트랜잭션을 지원합니다.
- **일반적인 접근 방식:**
  - 개발자가 직접 `ItemWriter<T>` 인터페이스를 구현하는 DAO(Data Access Object)를 만듭니다. (예: `MyCustomerDao implements ItemWriter<CustomerCredit>`)
  - 또는, Spring Data JPA의 `JpaRepository` 등을 활용한 `RepositoryItemWriter` (Spring Batch 제공) 등을 사용합니다.
  - Spring JDBC의 `JdbcBatchItemWriter` (Spring Batch 제공)를 사용하여 JDBC 배치 업데이트를 수행할 수도 있습니다.
- **일괄 처리(Batching) 시 오류 처리 문제 (그림 2. Error On Flush, 그림 3. Error On Write 참고):**
  - Hibernate나 JDBC 배치 모드를 사용하여 여러 아이템을 버퍼에 모았다가 한 번에 `flush` (DB로 전송)할 때, 만약 그중 일부 아이템에서 오류(예: `DataIntegrityViolationException`)가 발생하면,
    - **어떤 아이템이 문제였는지 정확히 알기 어렵습니다.** (버퍼 전체가 한꺼번에 처리되다가 실패)
    - 전체 트랜잭션이 롤백되어야 합니다.
    - Spring Batch의 skip 정책을 적용하기 어렵습니다 (어떤 아이템을 스킵해야 할지 모르므로).
  - **해결책 (또는 완화책):**
    - `ItemWriter`의 `write()` 메서드 내에서 각 아이템을 개별적으로 DB에 쓰고 즉시 `flush` 하거나 (성능 저하 가능성 있음),
    - 또는, 오류 발생 가능성이 적도록 데이터를 철저히 검증하거나,
    - 아니면 작은 크기의 배치 업데이트를 사용하고, 오류 발생 시 해당 배치 전체를 재처리하거나 실패 처리하는 전략을 사용합니다.
  - **핵심 가이드라인:** 안정적인 skip 처리를 원한다면, `ItemWriter` 구현 시 각 `write()` 호출마다 (또는 매우 작은 단위마다) 플러시하는 것을 고려해야 합니다.

---

### **주요 용어 해설**

- **커서 (Cursor - DB):** SQL 쿼리 결과 집합 내의 특정 레코드를 가리키는 포인터.
- **`ResultSet` (Java JDBC):** DB 쿼리 결과를 담고 있으며, 커서를 통해 각 행에 접근할 수 있는 객체.
- **`RowMapper<T>` (Spring JDBC):** `ResultSet`의 한 행 데이터를 `T` 타입 객체로 변환하는 인터페이스.
- **페이징 (Paging - DB):** 대량의 데이터를 작은 페이지(묶음) 단위로 나누어 가져오는 기법.
- **`PagingQueryProvider` (Spring Batch):** DB 종류별로 다른 페이징 쿼리 구문을 생성해주는 인터페이스.
- **정렬 키 (Sort Key - 페이징 시):** 페이징 쿼리 시 결과를 정렬하는 기준이 되는 컬럼. 페이지 간 데이터 일관성을 위해 고유해야 함.
- **JPQL (Java Persistence Query Language):** JPA에서 사용하는 객체 지향 쿼리 언어.
- **영속성 컨텍스트 (Persistence Context - JPA):** 엔티티의 생명주기를 관리하는 JPA의 환경.
- **DAO (Data Access Object):** 데이터베이스 접근 로직을 캡슐화한 객체.

---

### **코드 예제 분석**

### `JdbcCursorItemReader` 설정 (Java Configuration)

```java
@Configuration
public class DbReaderConfig {
    @Autowired // 또는 생성자 주입
    private DataSource dataSource; // 데이터베이스 연결을 위한 DataSource

    @Bean
    public JdbcCursorItemReader<CustomerCredit> customerCreditJdbcCursorReader() {
        return new JdbcCursorItemReaderBuilder<CustomerCredit>()
                .name("customerCreditJdbcCursorReader") // 리더 이름 (메타데이터 및 재시작 시 사용)
                .dataSource(this.dataSource)            // 사용할 DataSource
                .sql("SELECT ID, NAME, CREDIT FROM CUSTOMER WHERE CREDIT > ? ORDER BY ID") // 실행할 SQL
                .preparedStatementSetter(args -> args.setBigDecimal(1, new BigDecimal("1000"))) // SQL의 ?에 값 바인딩 (선택적)
                .rowMapper(new CustomerCreditRowMapper()) // ResultSet 행을 CustomerCredit 객체로 변환
                // .fetchSize(100) // (선택) DB에서 한 번에 가져올 행 수 힌트
                // .saveState(true) // (선택) 재시작을 위해 상태 저장 (기본값 true)
                .build();
    }

    // CustomerCreditRowMapper는 이전에 정의된 것 사용
}
```

- `dataSource`: DB 연결 제공.
- `sql`: 데이터를 가져올 SQL 쿼리. `?` 플레이스홀더 사용 가능.
- `preparedStatementSetter`: SQL의 `?`에 실제 값을 설정하는 로직 (선택 사항).
- `rowMapper`: 각 행을 `CustomerCredit` 객체로 변환.

### `JdbcPagingItemReader` 설정 (Java Configuration)

```java
@Configuration
public class DbPagingReaderConfig {
    @Autowired
    private DataSource dataSource;

    @Bean
    public SqlPagingQueryProviderFactoryBean customerCreditQueryProvider() {
        SqlPagingQueryProviderFactoryBean provider = new SqlPagingQueryProviderFactoryBean();
        provider.setDataSource(this.dataSource); // 어떤 DB인지 파악하기 위해 DataSource 필요
        provider.setSelectClause("SELECT ID, NAME, CREDIT");
        provider.setFromClause("FROM CUSTOMER");
        provider.setWhereClause("WHERE CREDIT > :minCredit"); // 명명된 파라미터 사용
        provider.setSortKey("ID"); // 페이징 기준이 될 고유한 정렬 키!
        return provider;
    }

    @Bean
    public JdbcPagingItemReader<CustomerCredit> customerCreditJdbcPagingReader(
            PagingQueryProvider customerCreditQueryProvider) throws Exception { // queryProvider 주입

        Map<String, Object> parameterValues = new HashMap<>();
        parameterValues.put("minCredit", new BigDecimal("500")); // WHERE 절의 :minCredit 파라미터 값

        return new JdbcPagingItemReaderBuilder<CustomerCredit>()
                .name("customerCreditJdbcPagingReader")
                .dataSource(this.dataSource)
                .queryProvider(customerCreditQueryProvider) // 위에서 정의한 쿼리 프로바이더
                .parameterValues(parameterValues)           // 쿼리 파라미터
                .rowMapper(new CustomerCreditRowMapper())   // RowMapper
                .pageSize(100)                             // 한 페이지당 아이템 수
                .build();
    }
}
```

- `SqlPagingQueryProviderFactoryBean`: DB 종류에 맞는 페이징 쿼리를 생성. `select`, `from`, `where`, `sortKey` 절을 조합.
- `JdbcPagingItemReader`: `queryProvider`로부터 페이징 쿼리를 받아 실행. `pageSize`만큼씩 데이터를 가져옴.

---

### **"왜?" 라는 질문에 대한 답변**

- **왜 커서 방식과 페이징 방식, 두 가지나 필요할까?**
  - **상황에 따른 트레이드오프:**
    - **커서:** 한 번의 쿼리, DB 연결 유지, 스트리밍에 최적화. 매우 큰 데이터셋을 순차적으로 한 번만 읽을 때 유리.
    - **페이징:** 여러 번의 쿼리, DB 연결 짧게 사용, 상태 관리 용이. DB 부하 분산, 병렬 처리, 특정 페이지 직접 접근 등에 유리할 수 있음.
  - 데이터베이스 종류, 데이터 크기, 네트워크 환경, 재시작 요구사항 등에 따라 더 적합한 방식이 다를 수 있습니다. Spring Batch는 선택지를 제공하여 유연성을 높입니다.
- **페이징 방식에서 `sortKey`가 왜 고유해야 할까?**
  - 페이지를 나누는 기준이 되는 `sortKey`가 중복된 값을 가지면, 한 페이지의 마지막 데이터와 다음 페이지의 첫 데이터가 같거나, 특정 데이터가 누락될 수 있습니다.
  - 예를 들어, `ORDER BY name`으로 정렬하는데 동명이인이 많고 `pageSize=10`이라면, "홍길동"이 10명 있을 때 첫 페이지에 "홍길동" 10명, 다음 페이지에도 "홍길동"이 또 나올 수 있습니다. 만약 `name`이 아닌 고유한 `id`로 정렬하면 이런 문제가 없습니다. `WHERE id > [이전 페이지 마지막 id]` 와 같은 방식으로 다음 페이지를 가져오기 때문에 `sortKey`는 페이지를 구분하는 경계 역할을 하며, 이 경계가 명확해야(고유해야) 데이터 중복/누락이 없습니다.

---

### **주의사항 및 Best Practice**

1. **커서 vs. 페이징 선택:** 처리할 데이터의 양, DB 부하, 트랜잭션 시간, 재시작의 복잡성 등을 고려하여 적절한 방식을 선택해야 합니다. 일반적으로 매우 큰 데이터셋의 단순 순차 처리에는 커서가, 그렇지 않은 경우에는 페이징이 더 유연할 수 있습니다.
2. **`RowMapper` 구현:** `RowMapper`는 `ResultSet`의 컬럼 이름이나 인덱스를 정확히 사용하고, 적절한 `getXXX()` 메서드를 호출하며, `SQLException`을 잘 처리해야 합니다.
3. **페이징 시 `sortKey`의 고유성 확보:** `JdbcPagingItemReader` 사용 시 `sortKey`는 반드시 고유한 값을 가지는 컬럼(들)으로 지정해야 데이터 무결성을 보장할 수 있습니다. (보통 PK)
4. **`fetchSize` (커서) / `pageSize` (페이징) 튜닝:** 이 값들은 성능에 큰 영향을 미칩니다. 너무 작으면 DB 왕복이 잦아지고, 너무 크면 메모리 사용량이 늘어날 수 있습니다. 적절한 값은 테스트를 통해 찾아야 합니다.
5. **데이터베이스 `ItemWriter`의 트랜잭션과 오류 처리:** 대량 쓰기 시 배치 업데이트 기능을 활용하고, 오류 발생 시 어떻게 처리할지(개별 아이템 skip 가능 여부, 롤백 범위 등) 신중하게 설계해야 합니다. Hibernate 사용 시에는 세션 `flush` 시점에 유의해야 합니다.
6. **`ItemStream` 구현:** 대부분의 데이터베이스 `ItemReader`는 `ItemStream`을 구현하므로, 재시작을 위한 상태 저장/복원이 가능합니다. `saveState=true` (보통 기본값)인지 확인하세요.

---

### **이전 학습 내용과의 연관성**

- **`ItemReader` / `ItemWriter` / `ItemStream` 인터페이스:** 이 챕터의 모든 DB 관련 리더/라이터는 이 기본 인터페이스들을 구현하거나 활용합니다.
- **`ExecutionContext`:** 재시작 시 상태 복원을 위해 사용됩니다.
- **`Resource`:** (직접 사용되진 않지만) DB 연결 정보를 담은 `DataSource`가 `Resource`와 유사한 역할을 합니다.
- **트랜잭션 관리:** 특히 `ItemWriter` 부분에서 청크 단위 트랜잭션과 DB 트랜잭션의 관계가 중요하게 다뤄집니다.

---
