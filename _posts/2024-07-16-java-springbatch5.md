---
layout: post
title: "Spring Batch 5 + Mybatis (JdbcItem Reader/Writer) 구현"
subtitle: "2024-07-16-java-springbatch5.md"
date: 2024-07-16 18:40:00
author: "전봉근"
header-img: "img/posts/android-bg.jpg"
comments: true
tags: [java, spring, spring batch]
---

# Spring Batch 5 + Mybatis (JdbcItem Reader/Writer) 구현
코드는 [https://github.com/bkjeon1614/java-example-code/tree/develop/spring-batch-mybatis-codebase](https://github.com/bkjeon1614/java-example-code/tree/develop/spring-batch-mybatis-codebase) 참고 부탁드립니다.


## Spring Batch 4.x -> 5.x 대표적 변경 내용
[What’s New in Spring Batch 5.0](https://docs.spring.io/spring-batch/docs/5.0.4/reference/html/whatsnew.html)
- Java 17 Requirement
- Major dependencies upgrade
- Batch infrastructure configuration updates
- Batch testing configuration updates
- Job parameters handling updates
- Execution context serialization updates
- SystemCommandTasklet updates
- New features
- Pruning

[Migrate Application From Spring Boot 2 to Spring Boot 3](https://www.baeldung.com/spring-boot-3-migration#spring-batch)
- @EnableBatchProcessing 을 권장하지 않음
- @EnableBatchProcessing or DefaultBatchConfiguration 을 상속하여, 구현할 경우 autoconfiguration 기능을 사용할 수 없음
- spring.batch.job.name 속성을 사용하여, 단일 Batch Job이 어플리케이션 실행 시 동작함 (이전 버전처럼 여러 Batch Job 수행 불가능)
- JobBuilder(String name) or JobBuilderFactory 는 depreacted 처리됨
  - JobBuilder(String name, JobRepository jobRepository) 사용
- StepBuilder(string name) or StepBuilderFactory는 deprecated 처리됨
  - StepBuilder(String name, JobRepository jobRepository) 사용


## 실습
#### SQL
```
create table sample (
    id         bigint not null auto_increment,
    amount     bigint,
    tx_name     varchar(255),
    tx_date_time datetime,
    primary key (id)
) engine = InnoDB;

insert into sample (amount, tx_name, tx_date_time) VALUES (1000, 'trade1', '2018-09-10 00:00:00');
insert into sample (amount, tx_name, tx_date_time) VALUES (2000, 'trade2', '2018-09-10 00:00:00');
insert into sample (amount, tx_name, tx_date_time) VALUES (3000, 'trade3', '2018-09-10 00:00:00');
insert into sample (amount, tx_name, tx_date_time) VALUES (4000, 'trade4', '2018-09-10 00:00:00');
insert into sample (amount, tx_name, tx_date_time) VALUES (4000, 'trade5', '2018-09-10 00:00:00');
insert into sample (amount, tx_name, tx_date_time) VALUES (5000, 'trade6', '2018-09-10 00:00:00');
insert into sample (amount, tx_name, tx_date_time) VALUES (6000, 'trade7', '2018-09-10 00:00:00');
insert into sample (amount, tx_name, tx_date_time) VALUES (7000, 'trade8', '2018-09-10 00:00:00');
insert into sample (amount, tx_name, tx_date_time) VALUES (8000, 'trade9', '2018-09-10 00:00:00');
insert into sample (amount, tx_name, tx_date_time) VALUES (9000, 'trade10', '2018-09-10 00:00:00');
insert into sample (amount, tx_name, tx_date_time) VALUES (10000, 'trade11', '2018-09-10 00:00:00');
insert into sample (amount, tx_name, tx_date_time) VALUES (14000, 'trade12', '2018-09-10 00:00:00');
insert into sample (amount, tx_name, tx_date_time) VALUES (15000, 'trade13', '2018-09-10 00:00:00');
insert into sample (amount, tx_name, tx_date_time) VALUES (16000, 'trade14', '2018-09-10 00:00:00');
insert into sample (amount, tx_name, tx_date_time) VALUES (18000, 'trade15', '2018-09-10 00:00:00');
insert into sample (amount, tx_name, tx_date_time) VALUES (20000, 'trade16', '2018-09-10 00:00:00');

create table sample_out (
    id         bigint not null auto_increment,
    amount     bigint,
    tx_name     varchar(255),
    tx_date_time datetime,
    primary key (id)
) engine = InnoDB;
```

#### 의존성추가
```
...

dependencies {
    ...

	// spring
	implementation 'org.springframework.boot:spring-boot-starter-actuator'
	implementation 'org.springframework.boot:spring-boot-starter-batch'
	implementation 'org.springframework.boot:spring-boot-starter-web'
	runtimeOnly 'com.mysql:mysql-connector-j'

	// Lombok for reducing boilerplate code
	compileOnly 'org.projectlombok:lombok'
	annotationProcessor 'org.projectlombok:lombok'

	// Test
	testImplementation 'org.springframework.boot:spring-boot-starter-test'
	testImplementation 'org.springframework.batch:spring-batch-test'
	testRuntimeOnly 'org.junit.platform:junit-platform-launcher'    
}
```

#### 기본 Application 설정
```
package com.bkjeon;

import jakarta.annotation.PostConstruct;
import lombok.extern.slf4j.Slf4j;
import org.springframework.beans.factory.annotation.Value;
import org.springframework.boot.SpringApplication;
import org.springframework.boot.autoconfigure.SpringBootApplication;

@Slf4j
@SpringBootApplication
public class SpringBatchMybatisCodebaseApplication {

	@Value("${spring.batch.job.name:NONE}")
	private String jobName;

	public static void main(String[] args) {
		int exitCode = SpringApplication.exit(SpringApplication.run(SpringBatchMybatisCodebaseApplication.class, args));
		log.info("exitCode={}", exitCode);
		System.exit(exitCode);
	}

	@PostConstruct
	public void validateJobNames() {
		log.info("jobName: {}", jobName);
		if (jobName.isEmpty() || jobName.equals("NONE")) {
			throw new IllegalStateException("Job Name Empty !!");
		}
	}

}
```

#### Job 생성
```
package com.bkjeon.job;

import com.bkjeon.feature.entity.sample.Sample;
import com.bkjeon.feature.rowmapper.sample.JdbcSampleRowMapper;
import java.util.HashMap;
import java.util.Map;
import javax.sql.DataSource;
import lombok.RequiredArgsConstructor;
import lombok.extern.slf4j.Slf4j;
import org.springframework.batch.core.Job;
import org.springframework.batch.core.Step;
import org.springframework.batch.core.configuration.annotation.JobScope;
import org.springframework.batch.core.job.builder.JobBuilder;
import org.springframework.batch.core.repository.JobRepository;
import org.springframework.batch.core.step.builder.StepBuilder;
import org.springframework.batch.item.database.BeanPropertyItemSqlParameterSourceProvider;
import org.springframework.batch.item.database.JdbcBatchItemWriter;
import org.springframework.batch.item.database.JdbcPagingItemReader;
import org.springframework.batch.item.database.Order;
import org.springframework.batch.item.database.PagingQueryProvider;
import org.springframework.batch.item.database.builder.JdbcBatchItemWriterBuilder;
import org.springframework.batch.item.database.builder.JdbcPagingItemReaderBuilder;
import org.springframework.batch.item.database.support.SqlPagingQueryProviderFactoryBean;
import org.springframework.beans.factory.annotation.Value;
import org.springframework.context.annotation.Bean;
import org.springframework.context.annotation.Configuration;
import org.springframework.transaction.PlatformTransactionManager;

/**
 * --job.name=JDBC_SAMPLE_JOB requestDate=20240711
 */
@Slf4j
@Configuration
@RequiredArgsConstructor
public class JdbcSampleJobConfig {

    private static final String JOB_NAME_PREFIX = "JDBC_SAMPLE";
    private static final int CHUNK_SIZE = 10;

    private final DataSource dataSource;

    @Bean
    public Job jdbcSampleJob(JobRepository jobRepository, Step jdbcSampleJobStep) {
        return new JobBuilder(JOB_NAME_PREFIX + "_JOB", jobRepository)
            .start(jdbcSampleJobStep)
            .build();
    }

    @Bean
    @JobScope
    public Step jdbcSampleJobStep(JobRepository jobRepository, PlatformTransactionManager platformTransactionManager,
        @Value("#{jobParameters[requestDate]}") String requestDate) throws Exception {
        log.info(">>>>> requestDate: {}", requestDate);
        return new StepBuilder(JOB_NAME_PREFIX + "_JOB_STEP", jobRepository)
            .<Sample, Sample>chunk(CHUNK_SIZE, platformTransactionManager)
            .reader(jdbcSamplePagingItemReader())
            .writer(jdbcSamplePagingItemWriter())
            .build();
    }

    @Bean
    public JdbcPagingItemReader<Sample> jdbcSamplePagingItemReader() throws Exception {
        return new JdbcPagingItemReaderBuilder<Sample>()
            .pageSize(CHUNK_SIZE)
            .fetchSize(CHUNK_SIZE)
            .dataSource(dataSource)
            .rowMapper(new JdbcSampleRowMapper())
            .queryProvider(createQueryProvider(dataSource))
            .parameterValues(getParameterValues())
            .name(JOB_NAME_PREFIX + "_PAGING_ITEM_READER")
            .build();
    }

    @Bean
    public JdbcBatchItemWriter<Sample> jdbcSamplePagingItemWriter() {
        return new JdbcBatchItemWriterBuilder<Sample>()
            .dataSource(dataSource)
            .itemSqlParameterSourceProvider(new BeanPropertyItemSqlParameterSourceProvider<>())
            .sql("""
                    INSERT INTO sample_out (id, amount, tx_name, tx_date_time)
                    VALUES (:id, :amount, :txName, :txDateTime)
                """)
            .build();
    }

    private Map<String, Object> getParameterValues() {
        Map<String, Object> parameterValues = new HashMap<>();
        parameterValues.put("amount", 2000);
        return parameterValues;
    }

    @Bean
    public PagingQueryProvider createQueryProvider(DataSource dataSource) throws Exception {
        SqlPagingQueryProviderFactoryBean queryProvider = new SqlPagingQueryProviderFactoryBean();
        queryProvider.setDataSource(dataSource);
        queryProvider.setSelectClause("id, amount, tx_name, tx_date_time");
        queryProvider.setFromClause("from sample");
        queryProvider.setWhereClause("where amount >= :amount");

        Map<String, Order> sortKeys = new HashMap<>(1);
        sortKeys.put("id", Order.ASCENDING);

        queryProvider.setSortKeys(sortKeys);

        return queryProvider.getObject();
    }

}
```

#### Entity 생성
```
package com.bkjeon.feature.entity.sample;

import java.time.LocalDateTime;
import lombok.Builder;
import lombok.Getter;
import lombok.NoArgsConstructor;
import lombok.ToString;

@ToString
@Getter
@NoArgsConstructor
public class Sample {

    private Long id;
    private Long amount;
    private String txName;
    private LocalDateTime txDateTime;

    @Builder
    public Sample(Long id, Long amount, String txName, LocalDateTime txDateTime) {
        this.id = id;
        this.amount = amount;
        this.txName = txName;
        this.txDateTime = txDateTime;
    }

}
```
```
package com.bkjeon.feature.entity.sample;

import java.time.LocalDateTime;
import lombok.Builder;
import lombok.Getter;
import lombok.NoArgsConstructor;
import lombok.ToString;

@ToString
@Getter
@NoArgsConstructor
public class SampleOut {

    private Long id;
    private Long amount;
    private String txName;
    private LocalDateTime txDateTime;

    @Builder
    public SampleOut(Long id, Long amount, String txName, LocalDateTime txDateTime) {
        this.id = id;
        this.amount = amount;
        this.txName = txName;
        this.txDateTime = txDateTime;
    }

}
```

#### Row Mapper 생성
Row Mapper 란 JDBC 기본 인터페이스 ResultSet에서 원하는 객체로 타입을 변경해주고 그것을 count(=rowNum) 만큼 반복     
```
package com.bkjeon.feature.rowmapper.sample;

import com.bkjeon.feature.entity.sample.Sample;
import java.sql.ResultSet;
import java.sql.SQLException;
import org.springframework.jdbc.core.RowMapper;

public class JdbcSampleRowMapper implements RowMapper<Sample> {

    @Override
    public Sample mapRow(ResultSet rs, int rowNum) throws SQLException {
        return Sample.builder()
            .id(rs.getLong("id"))
            .amount(rs.getLong("amount"))
            .txName(rs.getString("tx_name"))
            .txDateTime(rs.getTimestamp("tx_date_time").toLocalDateTime())
            .build();
    }

}
```

#### 실행
```
Program Args 에 --job.name=JDBC_SAMPLE_JOB requestDate=20240711 을 넣고 실행해보자.
```

## 참고
- https://www.baeldung.com
- https://velog.io/@collenkim
- https://docs.spring.io