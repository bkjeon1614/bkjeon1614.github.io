---
layout: post
title : "Spring Batch 2편 - Spring Batch 활용"
subtitle : "2022-09-08-java-springbatch-2.md"
date: 2022-09-08 12:50:00
author: "전봉근"
header-img: "img/posts/android-bg.jpg"
comments: true
tags: [spring, java, jvm, spring batch]
---

# Spring Batch 활용


## Spring Batch 프로젝트 생성
- 프로젝트 환경
  - IntelliJ IDEA 2022. 1
  - Spring Boot 2.7.3
  - Java 11
  - Gradle

- 프로젝트 생성
  - 프로젝트 유형 설정   
    ![springbatch-4](/img/posts/language/java/springbatch-4.png)    
  - 프로젝트 관련 설정   
    ![springbatch-5](/img/posts/language/java/springbatch-5.png)    
  - 생성된 build.gradle 확인   
    [build.gradle]    
    ```
    plugins {
        id 'org.springframework.boot' version '2.7.3'
        id 'io.spring.dependency-management' version '1.0.13.RELEASE'
        id 'java'
    }

    group = 'com.example'
    version = '0.0.1-SNAPSHOT'
    sourceCompatibility = '11'

    configurations {
        compileOnly {
            extendsFrom annotationProcessor
        }
    }

    repositories {
        mavenCentral()
    }

    dependencies {
        implementation 'org.springframework.boot:spring-boot-starter-batch'
        implementation 'org.springframework.boot:spring-boot-starter-data-jpa'
        compileOnly 'org.projectlombok:lombok'
        runtimeOnly 'com.h2database:h2'
        runtimeOnly 'mysql:mysql-connector-java'
        annotationProcessor 'org.projectlombok:lombok'
        testImplementation 'org.springframework.boot:spring-boot-starter-test'
        testImplementation 'org.springframework.batch:spring-batch-test'
    }

    tasks.named('test') {
        useJUnitPlatform()
    }
    ```
  - `Spring Batch Enable 설정` (SpringBatchStudyJpaApplication.java 에 @EnableBatchProcessing 추가)    
    [SpringBatchStudyJpaApplication.java]    
    ```
    package com.example;

    import org.springframework.batch.core.configuration.annotation.EnableBatchProcessing;
    import org.springframework.boot.SpringApplication;
    import org.springframework.boot.autoconfigure.SpringBootApplication;

    @EnableBatchProcessing	//	Spring Batch Enable
    @SpringBootApplication
    public class SpringBatchStudyJpaApplication {

        public static void main(String[] args) {
           SpringApplication.run(SpringBatchStudyJpaApplication.class, args);
        }

    }
    ```    


## Spring Batch 프로젝트 생성 및 실습
- com.example 에 `job` 패키지 생성 후 그 안에 `SimpleJobConfiguration.java` 파일 생성   
  [SimpleJobConfiguration.java]    
  ```
  package com.example.job;

  import org.springframework.batch.core.Job;
  import org.springframework.batch.core.Step;
  import org.springframework.batch.core.configuration.annotation.JobBuilderFactory;
  import org.springframework.batch.core.configuration.annotation.StepBuilderFactory;
  import org.springframework.batch.repeat.RepeatStatus;
  import org.springframework.context.annotation.Bean;
  import org.springframework.context.annotation.Configuration;

  import lombok.RequiredArgsConstructor;
  import lombok.extern.slf4j.Slf4j;

  @Slf4j  // Log
  @RequiredArgsConstructor  // 생성자 인젝션을 위한 Lombok 어노테이션 사용
  @Configuration  // Spring Batch 의 모든 Job 은 @Configuration 으로 등록해서 사용
  public class SimpleJobConfiguration {
    
      // 생성자 인젝션 주입
      private final JobBuilderFactory jobBuilderFactory;
      private final StepBuilderFactory stepBuilderFactory;

      @Bean
      public Job simpleJob() {
        return jobBuilderFactory.get("simpleJob") // simplejob 이란 이름의 batch job 생성 (job의 이름은 별도로 지정하지 않고, 이렇게 Builder를 통해 이름을 지정)
          .start(simpleStep1())
          .build();
      }

      @Bean
      public Step simpleStep1() {
        return stepBuilderFactory.get("simpleStep1")  // simpleStep1 이란 이름의 Batch Step 을 생성 (jobBuilderFactory.get("simpleJob") 과 마찬가지로 Builder를 통해 이름을 지정)
          // Step 안에서 수행될 기능들을 명시
          // tasklet 은 Step 안에서 단일로 수행될 커스텀한 기능들을 선언할 때 사용
          // 여기서 Batch 가 수행되면 하기 log 가 출력됨
          .tasklet((contribution, chunkContext) -> {
            log.info(">>>>> This is Step1");
            return RepeatStatus.FINISHED;
          })
          .build();
      }

  }
  ```   

> `Job 은 하나의 Batch 작업 단위`를 의미하며 `Job` 안에는 여러 `Step` 이 존재하고, `Step` 안에 `Tasklet` 혹은 `Reader & Processor & Writer` 묶음이 존재 (즉, `Tasklet 하나와 Reader & Processor & Writer 한 묶음이 같은 레벨` 이므로 만약 Reader & Processor 가 끝나고 Tasklet 으로 마무리 짓는 등으로 만들 수 없다는것을 유의하자!)


## Spring Batch 실습 (MySQL)
Spring Batch 를 사용하기 위해서는 `메타 데이터 테이블` 들이 필요하다. 
> 메타 데이터란, `데이터를 설명하는 데이터` 이다. [나무위키](https://namu.wiki/w/%EB%A9%94%ED%83%80%EB%8D%B0%EC%9D%B4%ED%84%B0)   

- 메타 데이터 테이블들은 하기 내용들과 그 외 여러가지 정보를 담고 있다.
  - 이전에 실행한 Job 의 정보 및 매개변수들
  - 실패한 Batch Parameter 가 어떤것이며 성공한 Job 은 어떤것들이 있는지
  - 다시 실행한다면 어디서 부터 시작하면 될지
  - 어떤 Job 에 어떤 Step 들이 있고, Step 들 중 성공한 Step 과 실패하면 Step 들은 어떤것들이 있는지

- 메타 데이터 테이블 생성
  - IDE 에서 파일 검색으로 `schema-` 를 하면 메타 테이블들의 스키마가 DBMS 에 맞춰 각각 존재하는것을 볼 수 있다. (원하는 스키마를 선택하여 해당 DBMS 에 생성하고 Spring Batch 를 실행해보자)

- MySQL 연결
  - application.yml 에 datasource 추가   
    [application.yml]
    ```
    spring:
      profiles:
        active: local

    ---
    spring:
      profiles:
        default: local
      datasource:
        hikari:
          jdbc-url: jdbc:h2:mem:testdb;DB_CLOSE_DELAY=-1;DB_CLOSE_ON_EXIT=FALSE
          username: sa
          password:
          driver-class-name: org.h2.Driver
    ---
    spring:
      profiles:
        default: mysql
      datasource:
        hikari:
          jdbc-url: jdbc:mysql://localhost:3306/spring_batch
          username: bkjeon
          password: wjsqhdrms
          driver-class-name: com.mysql.jdbc.Driver 
    ```
  - application profile 세팅 후 구동   
    ![springbatch-6](/img/posts/language/java/springbatch-6.png)    
    ![springbatch-7](/img/posts/language/java/springbatch-7.png)    
    > 메타 테이블 데이터인 `BATCH_JOB_INSTANCE` 가 존재하지 않아서 배치가 실패됨을 확인할 수 있다. 메타 테이블을 생성하지 않아서 표시되는 에러이므로 테이블들을 생성하자.   
  - 메타 테이블 생성
    - schema-mysql.sql 파일 검색 후 mysql 에 명령실행 후 확인   
      ![springbatch-8](/img/posts/language/java/springbatch-8.png)    
      ![springbatch-9](/img/posts/language/java/springbatch-9.png)    
    - 다시 배치를 실행해보자   
      ![springbatch-10](/img/posts/language/java/springbatch-10.png)    


## 참고
- https://docs.spring.io/
- https://jojoldu.tistory.com/