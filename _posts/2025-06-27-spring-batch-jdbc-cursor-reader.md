---
title: Spring Batch - JdbcCursorItemReader
description: 
author: laze
date: 2025-06-27 00:00:02 +0900
categories: [Dev, SpringBatch]
tags: [SpringBatch]
---
### Schema

```sql
-- 기존 테이블이 있으면 삭제 (테스트 시 매번 깨끗한 상태에서 시작하기 위함)
DROP TABLE IF EXISTS CUSTOMER;
DROP TABLE IF EXISTS PURCHASE_HISTORY;
DROP TABLE IF EXISTS VIP_CUSTOMER_REPORT; -- 리포트 테이블도 만든다고 가정

-- 고객 정보 테이블
CREATE TABLE CUSTOMER (
    id BIGINT PRIMARY KEY,
    name VARCHAR(255),
    grade VARCHAR(255) DEFAULT 'BRONZE'
);

-- 월별 구매 이력 테이블
CREATE TABLE PURCHASE_HISTORY (
    id BIGINT PRIMARY KEY AUTO_INCREMENT,
    customer_id BIGINT,
    price DECIMAL(19, 2),
    created_at TIMESTAMP
);

-- (참고) 만약 Writer를 DB로 한다면, 이런 결과 테이블이 있을 수 있음
CREATE TABLE VIP_CUSTOMER_REPORT (
    customer_id BIGINT,
    customer_name VARCHAR(255),
    total_amount DECIMAL(19, 2),
    new_grade VARCHAR(255)
);
```

### Domain

```java
package com.laze.springbatchpractice.domain;

import lombok.AllArgsConstructor;
import lombok.Data;
import lombok.NoArgsConstructor;

import java.math.BigDecimal;

@Data
@NoArgsConstructor
@AllArgsConstructor
public class VipCustomer {
    private Long customerId;
    private String customerName;
    private BigDecimal totalAmount;
    private String newGrade;
}
```

```java
package com.laze.springbatchpractice.domain;

import lombok.Data;
import lombok.NoArgsConstructor;

import java.math.BigDecimal;

@Data
@NoArgsConstructor
public class CustomerPurchaseSummary {
    private Long customerId;
    private String customerName;
    private BigDecimal totalAmount;
}
```

### JobConfig

```java
package com.laze.springbatchpractice.config;

import com.laze.springbatchpractice.domain.CustomerPurchaseSummary;
import lombok.RequiredArgsConstructor;
import org.springframework.batch.core.configuration.annotation.StepScope;
import org.springframework.batch.item.database.JdbcCursorItemReader;
import org.springframework.batch.item.database.builder.JdbcCursorItemReaderBuilder;
import org.springframework.beans.factory.annotation.Value;
import org.springframework.context.annotation.Bean;
import org.springframework.context.annotation.Configuration;
import org.springframework.jdbc.core.BeanPropertyRowMapper;

import javax.sql.DataSource;

@Configuration
@RequiredArgsConstructor
public class VipCustomerJobConfig {
    private final DataSource dataSource;

    @Bean
    @StepScope
    public JdbcCursorItemReader<CustomerPurchaseSummary> vipCustomerReader(
            @Value("#{jobParameters['yearMonth']}") String yearMonth
    ) {
        String sql = "SELECT C.id as customerId, C.name as customerName, SUM(P.price) as totalAmount " +
                "FROM CUSTOMER C " +
                "INNER JOIN PURCHASE_HISTORY P ON C.id = P.customer_id " +
                "WHERE FORMATDATETIME(P.created_at, 'yyyy-MM') = ? " + // H2 DB의 함수 사용
                "GROUP BY C.id, C.name " +
                "HAVING SUM(P.price) > 0 " + // 구매금액이 있는 고객만 대상
                "ORDER BY C.id";

        return new JdbcCursorItemReaderBuilder<CustomerPurchaseSummary>()
                .name("vipCustomerReader")
                .dataSource(this.dataSource)
                .sql(sql)
                .preparedStatementSetter(pstmt -> pstmt.setString(1, yearMonth))
                .rowMapper(new BeanPropertyRowMapper<>(CustomerPurchaseSummary.class))
                .build();
    }
}
```

### TestCode

```java
package com.laze.springbatchpractice.reader;

import com.laze.springbatchpractice.config.VipCustomerJobConfig;
import com.laze.springbatchpractice.domain.CustomerPurchaseSummary;
import org.junit.jupiter.api.AfterEach;
import org.junit.jupiter.api.BeforeEach;
import org.junit.jupiter.api.Test;
import org.springframework.batch.item.ExecutionContext;
import org.springframework.batch.item.database.JdbcCursorItemReader;
import org.springframework.batch.test.context.SpringBatchTest;
import org.springframework.beans.factory.annotation.Autowired;
import org.springframework.boot.autoconfigure.EnableAutoConfiguration;
import org.springframework.boot.test.context.SpringBootTest;
import org.springframework.jdbc.core.JdbcTemplate;
import org.springframework.test.context.ContextConfiguration;

import javax.sql.DataSource;

import static org.assertj.core.api.Assertions.assertThat;

@SpringBatchTest
@SpringBootTest
@EnableAutoConfiguration
@ContextConfiguration(classes = VipCustomerJobConfig.class)
public class VipCustomerReaderTest {

    @Autowired
    private DataSource dataSource;

    private JdbcTemplate jdbcTemplate;

    // 테스트할 ItemReader를 직접 가져오려면 Job Config Class에서 가져와야 함
    // 여기서는 테스트 메서드 내에서 직접 생성하여 테스트하는 방식
    // 또는 @Autowired로 주입받으려면 Job 설정 클래스에서 Bean 이름을 명확히 해야 함.
    private JdbcCursorItemReader<CustomerPurchaseSummary> reader;

    @BeforeEach
    void setUp() {
        this.jdbcTemplate = new JdbcTemplate(dataSource);

        // 테스트 데이터 준비
        jdbcTemplate.update("INSERT INTO CUSTOMER (id, name, grade) VALUES (1, 'John Doe', 'BRONZE')");
        jdbcTemplate.update("INSERT INTO CUSTOMER (id, name, grade) VALUES (2, 'Jane Smith', 'BRONZE')");
        jdbcTemplate.update("INSERT INTO CUSTOMER (id, name, grade) VALUES (3, 'Peter Jones', 'VIP')");

        // 2023년 7월 데이터
        jdbcTemplate.update("INSERT INTO PURCHASE_HISTORY (customer_id, price, created_at) VALUES (1, 1500000, '2023-07-10T10:00:00')");
        jdbcTemplate.update("INSERT INTO PURCHASE_HISTORY (customer_id, price, created_at) VALUES (2, 500000, '2023-07-15T11:00:00')");
        // 2023년 6월 데이터 (이 데이터는 조회되면 안 됨)
        jdbcTemplate.update("INSERT INTO PURCHASE_HISTORY (customer_id, price, created_at) VALUES (1, 200000, '2023-06-05T12:00:00')");
    }

    @AfterEach
    void tearDown() {
        jdbcTemplate.update("DELETE FROM PURCHASE_HISTORY");
        jdbcTemplate.update("DELETE FROM CUSTOMER");
    }

    @Test
    void should_read_customer_purchase_summary_for_given_month() throws Exception {
        // given
        String yearMonth = "2023-07";
        // JobParameters를 생성하여 yearMonth 값을 전달 (StepScope 빈에 값을 주입하기 위해)
        // StepScope 빈을 테스트하려면 StepExecution 컨텍스트가 필요함
        // 여기서는 ItemReader를 직접 생성하고 파라미터를 설정하여 테스트
        // StepScopeTestUtils를 활용할 수 있음

        // ItemReader를 직접 생성하는 방식
        VipCustomerJobConfig config = new VipCustomerJobConfig(dataSource);
        this.reader = config.vipCustomerReader(yearMonth);
        reader.afterPropertiesSet(); // 빌더로 생성 시 내부적으로 호출되나 수동 생성 시 명시적 호출 (recommend)

        ExecutionContext executionContext = new ExecutionContext();
        reader.open(executionContext);

        // when
        CustomerPurchaseSummary summary1 = reader.read();
        CustomerPurchaseSummary summary2 = reader.read();

        // then
        assertThat(summary1.getCustomerId()).isEqualTo(1L);
        assertThat(summary1.getCustomerName()).isEqualTo("John Doe");
        assertThat(summary1.getTotalAmount().intValue()).isEqualTo(1500000);

        assertThat(summary2.getCustomerId()).isEqualTo(2L);
        assertThat(summary2.getCustomerName()).isEqualTo("Jane Smith");
        assertThat(summary2.getTotalAmount().intValue()).isEqualTo(500000);

        // 더이상 읽을 데이터가 없어야 함 (3번 고객 7월 구매 내역 없음)
        assertThat(reader.read()).isNull();

        reader.close();

    }
}
```
