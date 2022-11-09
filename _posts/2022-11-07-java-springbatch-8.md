---
layout: post
title : "Spring Batch 8편 - ItemWriter"
subtitle : "2022-11-07-java-springbatch-8.md"
date: 2022-11-07 18:50:00
author: "전봉근"
header-img: "img/posts/android-bg.jpg"
comments: true
tags: [spring, java, jvm, spring batch]
---

# ItemWriter
이전글: [Spring Batch 7편 - ItemReader](https://bkjeon1614.tistory.com/745)   
작업코드: [작업코드](https://github.com/bkjeon1614/java-example-code/tree/develop/spring-batch-study/spring-batch-study-jpa)


## 1. ItemWriter 란
ItemWriter 는 Spring Batch 에서 사용하는 `출력` 기능이며 Spring Batch 초기에는 ItemReader 와 마찬가지로 item 을 하나씩 다루었지만 Spring Batch2 와 Chunk 기반 처리의 도입으로 인하여 ItemWriter 는 item 하나를 작성하지 않고 `Chunk 단위로 묶인 item list` 를 다룬다.   
![springbatch-47](/img/posts/language/java/springbatch/springbatch-47.png)     
상기 코드를 보면 ItemReader 의 read() 는 item 을 하나를 반환하지만, Writer 의 write() 는 인자로 Item List 를 받는다.    
    
하기 이미지를 참고하자.  
![springbatch-48](/img/posts/language/java/springbatch/springbatch-48.png)     
- ItemReader 를 통해 각 항목을 개별적으로 읽고 이를 처리하기 위해 ItemProcessor 에 전달
- 해당 프로세스는 Chunk 의 Item 개수 만큼 처리 될 때까지 계속된다.
- Chunk 단위만큼 처리가 완료되면 Writer 에 전달되어 Writer 에 명시되어있는대로 일괄처리 한다.
> 즉, Reader 와 Processor 를 거쳐 처리된 Item 을 Chunk 단위 만큼 쌓은 뒤 이를 Writer 에 전달


## 2. Database Writer
Writer 는 Chunk 단위의 마지막 단계이다. 그러므로 Database 의 영속성과 관련해서는 `항상 마지막에 Flush 를 해야 한다.` 예를 들어 영속성을 사용하는 JPA, Hibernate 의 경우 ItemWriter 구현체에서는 JPA 는 flush(), Hibernate 는 session.clear() 가 따라온다. Writer 가 받은 모든 Item 이 처리 된 후, Spring Batch 는 현재 트랜잭션을 커밋한다.  
      
Database 와 관련된 Writer 는 아래와 같다.  
- JdbcBatchItemWriter
- HibernateItemWriter
- JpaItemWriter

### 2-1. JdbcBatchItemWriter
ORM 을 사용하지 않는 경우 대부분 JdbcBatchItemWriter 를 사용한다. JdbcBatchItemWriter 는 `JDBC 의 Batch 기능을 사용하여 한번에 Database 로 전달하여 Database 내부에서 쿼리들이 실행` 되도록 한다.  
1. JdbcBatchItemWriter 에서 Query 모은다. (각 Query 들은 ChunkSize 만큼 쌓는다.)   
2. 모아놓은 Query 들을 한번에 Database 로 전송   
3. Database 에서 받은 쿼리들을 실행    
이렇게 처리하는 이유는 애플리케이션과 데이터베이스 간에 데이터를 주고 받는 회수를 최소화 하여 성능 향상을 시키기 위함이다. ([업데이트를 일괄 처리로 그룹화하면 DB와 애플리케이션간 왕복 횟수가 줄어들어 성능이 향상 된다.](https://docs.spring.io/spring-framework/docs/3.0.0.M4/reference/html/ch12s04.html))   
> 실제로 JdbcBatchItemWriter 의 write() 를 확인하면 일괄처리 하는 것을 확인할 수 있다. (Ex: namedParameterJdbcTemplate.batchUpdate(), ps.addBatch())    
         
JdbcBatchItemWriter 를 사용한 간단한 Batch Job 및 예제를 위한 파일들을 생성해보자.  
[application.yml]  
```
...
spring:
  profiles:
    default: mysql
  datasource:
    hikari:
      jdbc-url: jdbc:mysql://localhost:3306/spring_batch
      username: root
      password: wjsqhdrms
      driver-class-name: com.mysql.jdbc.Driver
  jpa:	# 추가 (로그확인용도)
    show-sql: true
...
```    
[sql]
```
create table product_new (
 id         bigint not null auto_increment,
 amount     bigint,
 tx_name     varchar(255),
 tx_date_time datetime,
 primary key (id)
) engine = InnoDB;
```    
[ProductNew.java]
```
package com.example.entity;

import java.time.LocalDateTime;
import java.time.format.DateTimeFormatter;

import javax.persistence.Entity;
import javax.persistence.GeneratedValue;
import javax.persistence.GenerationType;
import javax.persistence.Id;

import lombok.Getter;
import lombok.NoArgsConstructor;
import lombok.Setter;
import lombok.ToString;

@ToString
@Getter
@Setter
@NoArgsConstructor
@Entity(name = "product_new")
public class ProductNew {

	private static final DateTimeFormatter FORMATTER = DateTimeFormatter.ofPattern("yyyy-MM-dd hh:mm:ss");

	@Id
	@GeneratedValue(strategy = GenerationType.IDENTITY)
	private Long id;
	private Long amount;
	private String txName;
	private LocalDateTime txDateTime;

	public ProductNew(Long id, Long amount, String txName, String txDateTime) {
		this.id = id;
		this.amount = amount;
		this.txName = txName;
		this.txDateTime = LocalDateTime.parse(txDateTime, FORMATTER);
	}

}
```    
[JdbcBatchItemWriterJobConfiguration.java]   
```
package com.example.job;

import javax.sql.DataSource;

import org.springframework.batch.core.Job;
import org.springframework.batch.core.Step;
import org.springframework.batch.core.configuration.annotation.JobBuilderFactory;
import org.springframework.batch.core.configuration.annotation.StepBuilderFactory;
import org.springframework.batch.item.database.JdbcBatchItemWriter;
import org.springframework.batch.item.database.JdbcCursorItemReader;
import org.springframework.batch.item.database.builder.JdbcBatchItemWriterBuilder;
import org.springframework.batch.item.database.builder.JdbcCursorItemReaderBuilder;
import org.springframework.context.annotation.Bean;
import org.springframework.context.annotation.Configuration;
import org.springframework.jdbc.core.BeanPropertyRowMapper;

import com.example.entity.Product;

import lombok.RequiredArgsConstructor;
import lombok.extern.slf4j.Slf4j;

@Slf4j
@RequiredArgsConstructor
@Configuration
public class JdbcBatchItemWriterJobConfiguration {
	private final JobBuilderFactory jobBuilderFactory;
	private final StepBuilderFactory stepBuilderFactory;
	private final DataSource dataSource; // DataSource DI

	private static final int chunkSize = 10;

	@Bean
	public Job jdbcBatchItemWriterJob() {
		return jobBuilderFactory.get("jdbcBatchItemWriterJob")
			.start(jdbcBatchItemWriterStep())
			.build();
	}

	@Bean
	public Step jdbcBatchItemWriterStep() {
		return stepBuilderFactory.get("jdbcBatchItemWriterStep")
			.<Product, Product>chunk(chunkSize)
			.reader(jdbcBatchItemWriterReader())
			.writer(jdbcBatchItemWriter())
			.build();
	}

	@Bean
	public JdbcCursorItemReader<Product> jdbcBatchItemWriterReader() {
		return new JdbcCursorItemReaderBuilder<Product>()
			.fetchSize(chunkSize)
			.dataSource(dataSource)
			.rowMapper(new BeanPropertyRowMapper<>(Product.class))
			.sql("SELECT id, amount, tx_name, tx_date_time FROM Product")
			.name("jdbcBatchItemWriter")
			.build();
	}

	/**
	 * reader 에서 넘어온 데이터를 하나씩 출력하는 writer
	 */
	@Bean // beanMapped()을 사용할때는 필수
	public JdbcBatchItemWriter<Product> jdbcBatchItemWriter() {
		return new JdbcBatchItemWriterBuilder<Product>()
			.dataSource(dataSource)
			.sql("insert into product_new(amount, tx_name, tx_date_time) values (:amount, :txName, :txDateTime)")
			.beanMapped()
			.build();
	}

}
```      
JdbcBatchItemWriterBuilder 설정값은
- assertUpdates
  - Parameter Type: boolean
  - 설명: 적어도 하나의 항목이 행을 업데이트하거나 삭제하지 않을 경우 예외를 throw 할지 여부를 결정 Exception: EmptyResultDataAccessException (기본값은 true)
- columnMapped
  - Parameter Type: 없음
  - 설명: Key, Value 기반으로 Insert SQL 의 Values 를 매핑한다. (ex: Map<String, Object>)
- beanMapped
  - Parameter Type: 없음
  - 설명: POJO 기반으로 Insert SQL 의 Values 를 매핑한다.   
   
여기서 columnMapped 와 beanMapped 차이를 알아보자면 상기 JdbcBatchItemWriterJobConfiguration.java 코드는 beanMapped 로 작성되었으며 만약, columnMapped 로 변경하면 아래와 같은 코드로 변한다.  
```
new JdbcBatchItemWriterBuilder<Map<String, Object>>() // Map 사용
	.columnMapped()
	.dataSource(this.dataSource)
	.sql("insert into pay2(amount, tx_name, tx_date_time) values (:amount, :txName, :txDateTime)")
	.build();
```   
차이는 Reader 에서 Writer 로 넘겨주는 타입이 Map<String, Object> 또는 Product.class 와 같은 POJO 타입의 차이다.   
    
또한 values (:field) 는 DTO 의 Getter 혹은 Map의 Key 에 매핑되어 값이 할당된다.   
그리고 JdbcBatchItemWriter 제네릭 타입은 Reader 에서 넘겨주는 값의 타입이며, 상기 예제 코드중 product_new 테이블에 데이터를 넣는것은 Writer 이지만 선언된 제네릭 타입은 Reader/Processor 에서 넘겨준 Product 클래스이다.  
  
마지막 이외에도 추가로 알고 있어야 할 메소드는 afterPropertiesSet 이다. 해당 메소드는 InitializingBean 인터페이스에서 갖고 있는 메소드이며 ItemWriter 의 구현체들은 모두 InitializingBean 인터페이스를 구현하고 있다. afterPropertiesSet 의 역할은 각각의 Writer 들이 실행되기 위해 필요한 필수값들이 제대로 세팅되어 있는지를 체크한다. (어느 값이 누락되었는지 명확하게 인지할 수 있어서 보편적으로 사용하는 옵션)  
```
...

@Override
public void afterPropertiesSet() {
	// 체크로직
	...
}
```   

상기 샘플 코드를 실행하면 product_new 테이블에 데이터가 적재된걸 확인할 수 있다.  
![springbatch-49](/img/posts/language/java/springbatch/springbatch-49.png)     


## 3. JpaItemWriter
ORM 을 사용할 수 있는 JpaItemWriter 이다. Writer 에 전달하는 데이터가 Entity 클래스라면 JpaItemWriter 를 사용하면 된다. 아래 샘플코드를 작성해보자.  
[JpaItemWriterJobConfiguration.java]
```
package com.example.job;

import javax.persistence.EntityManagerFactory;

import org.springframework.batch.core.Job;
import org.springframework.batch.core.Step;
import org.springframework.batch.core.configuration.annotation.JobBuilderFactory;
import org.springframework.batch.core.configuration.annotation.StepBuilderFactory;
import org.springframework.batch.item.ItemProcessor;
import org.springframework.batch.item.database.JpaItemWriter;
import org.springframework.batch.item.database.JpaPagingItemReader;
import org.springframework.batch.item.database.builder.JpaPagingItemReaderBuilder;
import org.springframework.context.annotation.Bean;
import org.springframework.context.annotation.Configuration;

import com.example.entity.Product;
import com.example.entity.ProductNew;

import lombok.RequiredArgsConstructor;
import lombok.extern.slf4j.Slf4j;

@Slf4j
@RequiredArgsConstructor
@Configuration
public class JpaItemWriterJobConfiguration {
	private final JobBuilderFactory jobBuilderFactory;
	private final StepBuilderFactory stepBuilderFactory;
	private final EntityManagerFactory entityManagerFactory;

	private static final int chunkSize = 10;

	@Bean
	public Job jpaItemWriterJob() {
		return jobBuilderFactory.get("jpaItemWriterJob")
			.start(jpaItemWriterStep())
			.build();
	}

	@Bean
	public Step jpaItemWriterStep() {
		return stepBuilderFactory.get("jpaItemWriterStep")
			.<Product, ProductNew>chunk(chunkSize)
			.reader(jpaItemWriterReader())
			.processor(jpaItemProcessor())
			.writer(jpaItemWriter())
			.build();
	}

	@Bean
	public JpaPagingItemReader<Product> jpaItemWriterReader() {
		return new JpaPagingItemReaderBuilder<Product>()
			.name("jpaItemWriterReader")
			.entityManagerFactory(entityManagerFactory)
			.pageSize(chunkSize)
			.queryString("SELECT p FROM Product p")
			.build();
	}

	// Processor 추가: Product Entity 를 읽어서 Writer 에는 ProductNew Entity 를 전달하기 위함
	@Bean
	public ItemProcessor<Product, ProductNew> jpaItemProcessor() {
		return product -> new ProductNew(product.getAmount(), product.getTxName(), product.getTxDateTime());
	}

	@Bean
	public JpaItemWriter<ProductNew> jpaItemWriter() {
		JpaItemWriter<ProductNew> jpaItemWriter = new JpaItemWriter<>();
		jpaItemWriter.setEntityManagerFactory(entityManagerFactory);
		return jpaItemWriter;
	}

}
```   
> Processor 추가: Product Entity 를 읽어서 Writer 에는 ProductNew Entity 를 전달하기 위함 (Reader 에서 읽은 데이터를 가공해야할 때 Processor 가 필요)       
JpaItemWriter 는 JPA 를 사용하므로 영속성 관리를 위해 EntityManager 를 할당해야 한다. 또한 JpaItemWriter는 JdbcBatchItemWriter와 달리 넘어온 Entity를 데이터베이스에 반영한다. 즉, `JpaItemWriter는 Entity 클래스를 제네릭 타입으로 받아야만 한다.` (JdbcBatchItemWriter의 경우 DTO 클래스를 받더라도 sql로 지정된 쿼리가 실행되니 문제가 없지만, JpaItemWriter 의 내부 메소드은 doWrite() 에서는 넘어온 Item을 그대로 entityManger.merge()로 테이블에 반영을 하기 때문)   
       
> spring-boot-starter-data-jpa 가 의존성에 등록되어있다면 Entity Manager 가 Bean 으로 자동생성되어 DI 코드만 추가하면 된다. 대신 필수로 설정해야할 값이 Entity Manager 뿐이다. (즉, JdbcBatchItemWriter 에 비해 필수값이 Entity Manager 뿐이라 체크할 요소가 적다는 것이 장점)   
   
그리고 필수값 체크 메소드인 afterPropertiesSet 에 EntityManager 만 set 하여 설정을 마무리하자.   
```
// JpaItemWriterJobConfiguration.java 에 추가
public void afterPropertiesSet() throws Exception {
	Assert.notNull(entityManagerFactory, "EntityManagerFactory is required");
}
```    
    
실제로 실행하여 결과를 확인해보자.   
![springbatch-50](/img/posts/language/java/springbatch/springbatch-50.png)     


## 4. Custom ItemWriter
Spring Batch 에서 공식적으로 지원하지 않는 Writer 를 사용하고 싶을 때 `ItemWriter 인터페이스를 구현` 하면 된다.
- Reader에서 읽어온 데이터를 RestTemplate으로 외부 API로 전달해야할때
- 임시저장을 하고 비교하기 위해 싱글톤 객체에 값을 넣어야할때
- 여러 Entity를 동시에 save 해야할때  
샘플 코드를 작성해보자.  
[CustomItemWriterJobConfiguration.java]   
```
package com.example.job;

import javax.persistence.EntityManagerFactory;

import org.springframework.batch.core.Job;
import org.springframework.batch.core.Step;
import org.springframework.batch.core.configuration.annotation.JobBuilderFactory;
import org.springframework.batch.core.configuration.annotation.StepBuilderFactory;
import org.springframework.batch.item.ItemProcessor;
import org.springframework.batch.item.ItemWriter;
import org.springframework.batch.item.database.JpaPagingItemReader;
import org.springframework.batch.item.database.builder.JpaPagingItemReaderBuilder;
import org.springframework.context.annotation.Bean;
import org.springframework.context.annotation.Configuration;

import com.example.entity.Product;
import com.example.entity.ProductNew;

import lombok.RequiredArgsConstructor;
import lombok.extern.slf4j.Slf4j;

@Slf4j
@RequiredArgsConstructor
@Configuration
public class CustomItemWriterJobConfiguration {
	private final JobBuilderFactory jobBuilderFactory;
	private final StepBuilderFactory stepBuilderFactory;
	private final EntityManagerFactory entityManagerFactory;

	private static final int chunkSize = 10;

	@Bean
	public Job customItemWriterJob() {
		return jobBuilderFactory.get("customItemWriterJob")
			.start(customItemWriterStep())
			.build();
	}

	@Bean
	public Step customItemWriterStep() {
		return stepBuilderFactory.get("customItemWriterStep")
			.<Product, ProductNew>chunk(chunkSize)
			.reader(customItemWriterReader())
			.processor(customItemWriterProcessor())
			.writer(customItemWriter())
			.build();
	}

	@Bean
	public JpaPagingItemReader<Product> customItemWriterReader() {
		return new JpaPagingItemReaderBuilder<Product>()
			.name("customItemWriterReader")
			.entityManagerFactory(entityManagerFactory)
			.pageSize(chunkSize)
			.queryString("SELECT p FROM Product p")
			.build();
	}

	@Bean
	public ItemProcessor<Product, ProductNew> customItemWriterProcessor() {
		return product -> new ProductNew(product.getAmount(), product.getTxName(), product.getTxDateTime());
	}

	@Bean
	public ItemWriter<ProductNew> customItemWriter() {
		return items -> {
			for (ProductNew item : items) {
				System.out.println(item);
			}
		};
	}

}
```
> 코드와 같이 write() 만 @Override 하면 된다.


## 참고
- https://jojoldu.tistory.com/