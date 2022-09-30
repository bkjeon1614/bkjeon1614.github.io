---
layout: post
title : "Spring Batch 3편 - Spring Batch Meta Table 설명"
subtitle : "2022-09-09-java-springbatch-3.md"
date: 2022-09-09 18:50:00
author: "전봉근"
header-img: "img/posts/android-bg.jpg"
comments: true
tags: [spring, java, jvm, spring batch]
---

# Spring Batch Meta Table
이전글: https://bkjeon1614.tistory.com/738
작업코드: [작업코드](https://github.com/bkjeon1614/java-example-code/tree/develop/spring-batch-study/spring-batch-study-jpa)

하기 이미지는 Spring Batch 를 사용하기 위해 필요한 메타 테이블이며 이것들이 각각 어떤 데이터를 저장하는지에 대해 알아보자.
![springbatch-3](/img/posts/language/java/springbatch-3.png)    

## BATCH_JOB_INSTANCE
먼저 `BATCH_JOB_INSTANCE` 테이블을 조회해보면 실행한 Job 이 존재하는걸 볼 수 있다.   
![springbatch-11](/img/posts/language/java/springbatch-11.png)    
> 해당 테이블은 `Job Parameter` 에 따라 생성되므로, `Job Parameter` 가 다를 경우에만 기록되며 `Job Parameter 같다면 기록되지 않는다.`

확인하기 위하여 먼저 이전에 작성하였던 SimpleJobConfiguration.java 를 수정하여 실행해보자.   
[SimpleJobConfiguration.java]    
```
package com.example.job;

import org.springframework.batch.core.Job;
import org.springframework.batch.core.Step;
import org.springframework.batch.core.configuration.annotation.JobBuilderFactory;
import org.springframework.batch.core.configuration.annotation.JobScope;
import org.springframework.batch.core.configuration.annotation.StepBuilderFactory;
import org.springframework.batch.repeat.RepeatStatus;
import org.springframework.beans.factory.annotation.Value;
import org.springframework.context.annotation.Bean;
import org.springframework.context.annotation.Configuration;

import lombok.RequiredArgsConstructor;
import lombok.extern.slf4j.Slf4j;

@Slf4j
@RequiredArgsConstructor
@Configuration
public class SimpleJobConfiguration {

    private final JobBuilderFactory jobBuilderFactory;
    private final StepBuilderFactory stepBuilderFactory;

    @Bean
    public Job simpleJob() {
      return jobBuilderFactory.get("simpleJob")
        .start(simpleStep1(null))
        .build();
    }

    @Bean
    @JobScope
    public Step simpleStep1(@Value("#{jobParameters[requestDate]}") String requestDate) {
      return stepBuilderFactory.get("simpleStep1")
        .tasklet((contribution, chunkContext) -> {
          log.info(">>>>> This is Step1");
          log.info(">>>>> Request Data: {}", requestDate);
          return RepeatStatus.FINISHED;
        })
        .build();
    }

}
```    
![springbatch-12](/img/posts/language/java/springbatch-12.png)    
![springbatch-13](/img/posts/language/java/springbatch-13.png)    
![springbatch-14](/img/posts/language/java/springbatch-14.png)    

> 여기까지 Job Parameter 가 로그에 잘 찍힌것을 볼 수 있고, 만약 동일한 파라미터로 실행하였을 때 하기 이미지와 같이 에러메세지가 표시되는 것을 확인할 수 있다.
![springbatch-15](/img/posts/language/java/springbatch-15.png)    

> 그 다음 requestDate=20220915 로 변경하여 실행하면 하기 이미지와 같이 정상적으로 수행되었음을 확인할 수 있다.
![springbatch-16](/img/posts/language/java/springbatch-16.png)    
![springbatch-17](/img/posts/language/java/springbatch-17.png)    


## BATCH_JOB_EXECUTION
`BATCH_JOB_EXECUTION` 은 `JOB_INSTANCE` 가 `성공 or 실패 했었던 모든 내역을 갖고 있다.`   
먼저 SimpleJobConfiguration.java 코드를 에러가 발생하게끔 변경해보자.   
[SimpleJobConfiguration.java]    
```
package com.example.job;

import org.springframework.batch.core.Job;
import org.springframework.batch.core.Step;
import org.springframework.batch.core.configuration.annotation.JobBuilderFactory;
import org.springframework.batch.core.configuration.annotation.JobScope;
import org.springframework.batch.core.configuration.annotation.StepBuilderFactory;
import org.springframework.batch.repeat.RepeatStatus;
import org.springframework.beans.factory.annotation.Value;
import org.springframework.context.annotation.Bean;
import org.springframework.context.annotation.Configuration;

import lombok.RequiredArgsConstructor;
import lombok.extern.slf4j.Slf4j;

@Slf4j
@RequiredArgsConstructor
@Configuration
public class SimpleJobConfiguration {

    private final JobBuilderFactory jobBuilderFactory;
    private final StepBuilderFactory stepBuilderFactory;

    @Bean
    public Job simpleJob() {
        return jobBuilderFactory.get("simpleJob")
            .start(simpleStep1(null))
            .next(simpleStep2(null))
            .build();
    }

    @Bean
    @JobScope
    public Step simpleStep1(@Value("#{jobParameters[requestDate]}") String requestDate) {
        return stepBuilderFactory.get("simpleStep1")
            .tasklet((contribution, chunkContext) -> {
                throw new IllegalArgumentException("Step1 실패 !!");
            })
            .build();
    }

    @Bean
    @JobScope
    public Step simpleStep2(@Value("#{jobParameters[requestDate]}") String requestDate) {
        return stepBuilderFactory.get("simpleStep2")
          .tasklet((contribution, chunkContext) -> {
              log.info(">>>>> This is Step2");
              log.info(">>>>> Request Data: {}", requestDate);
              return RepeatStatus.FINISHED;
          })
          .build();
    }

}
```   
그 다음 Job Parameter 을 `requestDate=20200101` 로 변경 후 실행하면 에러 및 에러로그가 테이블에 저장되어있는것을 확인할 수 있다.
![springbatch-18](/img/posts/language/java/springbatch-18.png)    
![springbatch-19](/img/posts/language/java/springbatch-19.png)    
![springbatch-20](/img/posts/language/java/springbatch-20.png)     
   
이제 코드를 수정하여 JOB 을 성공시켰을때 결과를 확인해보자.   
[SimpleJobConfiguration.java]
```
package com.example.job;

import org.springframework.batch.core.Job;
import org.springframework.batch.core.Step;
import org.springframework.batch.core.configuration.annotation.JobBuilderFactory;
import org.springframework.batch.core.configuration.annotation.JobScope;
import org.springframework.batch.core.configuration.annotation.StepBuilderFactory;
import org.springframework.batch.repeat.RepeatStatus;
import org.springframework.beans.factory.annotation.Value;
import org.springframework.context.annotation.Bean;
import org.springframework.context.annotation.Configuration;

import lombok.RequiredArgsConstructor;
import lombok.extern.slf4j.Slf4j;

@Slf4j
@RequiredArgsConstructor
@Configuration
public class SimpleJobConfiguration {

    private final JobBuilderFactory jobBuilderFactory;
    private final StepBuilderFactory stepBuilderFactory;

    @Bean
    public Job simpleJob() {
        return jobBuilderFactory.get("simpleJob")
            .start(simpleStep1(null))
            .next(simpleStep2(null))
            .build();
    }

    @Bean
    @JobScope
    public Step simpleStep1(@Value("#{jobParameters[requestDate]}") String requestDate) {
        return stepBuilderFactory.get("simpleStep1")
            .tasklet((contribution, chunkContext) -> {
                log.info(">>>>> This is Step1");
                log.info(">>>>> Request Data: {}", requestDate);
                return RepeatStatus.FINISHED;
            })
            .build();
    }

    @Bean
    @JobScope
    public Step simpleStep2(@Value("#{jobParameters[requestDate]}") String requestDate) {
        return stepBuilderFactory.get("simpleStep2")
            .tasklet((contribution, chunkContext) -> {
                log.info(">>>>> This is Step2");
                log.info(">>>>> Request Data: {}", requestDate);
                return RepeatStatus.FINISHED;
            })
            .build();
    }

}
```   
![springbatch-21](/img/posts/language/java/springbatch-21.png)   
![springbatch-22](/img/posts/language/java/springbatch-22.png)       

> 여기서 `JOB_INSTANCE_ID` 컬럼을 보면 같은 ID 인 `5` 를 가진 2개의 ROW 를 볼 수 있다. 하나는 FAILED 이고 나머지 하나는 COMPLETED 인데 해당 Job 은 `동일한 Job Parameter 으로 2번 실행했는데 같은 파라미터로 실행되었다는 에러가 발생되지 않은것을 확인할 수 있다.` 즉, `동일한 Job Parameter 로 성공한 기록이 없을 때 재수행이 된다` 는 것을 확인할 수 있다. 

즉, 상기 내용을 간단하게 정리하자면 아래와 같다.
- Job: simpleJob
- Job Instance: Job Parameter 를 requestDate=20200101 으로 지정하여 실행한 simpleJob (Job Parameter 단위로 생성)
- Job Execution: Job Parameter 를 requestDate=20200101 으로 지정하여 실행한 simpleJob 의 1번째 시도 or 다음번 시도 (`부모-자식 관계 => job Instance: 부모, Job Excution: 자식`)

마지막으로 `BATCH_JOB_EXECUTION_PARAMS` 테이블은 BATCH_JOB_EXECUTION 테이블이 생성될 당시에 입력 받은 `Job Parameter` 데이터를 가지고 있다.   
![springbatch-23](/img/posts/language/java/springbatch-23.png)       


## 참고
- https://docs.spring.io/
- https://jojoldu.tistory.com/