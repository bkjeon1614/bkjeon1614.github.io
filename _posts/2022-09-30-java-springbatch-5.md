---
layout: post
title : "Spring Batch 5편 - Spring Batch Scope & Job Parameter"
subtitle : "2022-09-30-java-springbatch-5.md"
date: 2022-09-30 18:50:00
author: "전봉근"
header-img: "img/posts/android-bg.jpg"
comments: true
tags: [spring, java, jvm, spring batch]
---

# Scope & Job Parameter
이전글: https://bkjeon1614.tistory.com/740   
작업코드: [작업코드](https://github.com/bkjeon1614/java-example-code/tree/develop/spring-batch-study/spring-batch-study-jpa)

## Job Parameter, Scope
외부/내 에서 파라미터를 받아 처리할 때 해당 파라미터를 `Job Parameter` 라고 한다.   
Job Parameter 를 사용하려면 항상 Scope 를 선언해야 하는데 Scope 는 크게 `@StepScope` 와 `@JobScope` 가 있다. 사용법은 아래와 같다.        
```
@Value("#{jobParameters[파라미터명]}")
```   
> 그 외에도 `jobExecutionContext`, `stepExecutionContext` 등 에도 `SpEL` 로 사용할 수 있다. @JobScope 에선 `stepExecutionContext` 는 사용할 수 없고, `jobParameters` 와 `jobExecutionContext` 만 사용할 수 있다. (SpEL(=Spring Expression Language): 런타임에서 객체에 대한 쿼리와 조작을 지원하는 강력한 표현 언어)

### JobScope
@JobScope 는 Step 선언문에서 사용이 가능하다.
```
...

@Bean
public Job simpleJob() {
    return jobBuilderFactory.get("simpleJob")
        .start(simpleStep1(null))
        .next(simpleStep2())
        .build();

@Bean
@JobScope
public Step scopeStep1(@Value("#{jobParameters[requestDate]}") String requestDate) {
    return stepBuilderFactory.get("simpleStep1")
        .tasklet((contribution, chunkContext) -> {
            log.info(">>>>> This is Step1");
            log.info(">>>>> Request Data: {}", requestDate);
            return RepeatStatus.FINISHED;
        })
        .build();
}

...
```

### StepSocpe
@StepScope 는 Tasklet 이나 ItemReader, ItemWriter, ItemProcessor 에서 사용이 가능하다.
```
...

@Bean
public Job simpleStep2() {
    return stepBuilderFactory.get("simpleStep2")
        .tasklet(simpleStep2Tasklet(null))
        .build();

@Bean
@StepScope
public Tasklet simpleStep2Tasklet(@Value("#{jobParameters[requestDate]}") String requestDate) {
    return (contribution, chunkContext) -> {
        log.info(">>>>> This is simpleStep2Tasklet");
        return RepeatStatus.FINISHED;
    }
}

...
```

> 현재 Job Parameter 의 타입으로 사용할 수 있는건 Double, Long, Date, String 이 있다.

## @StepScope 와 @JobScope
Spring Bean 의 기본 Scope 는 Singleton 이지만 하기 코드처럼 Spring Batch 컴포넌트 (Ex: Tasklet, ItemReader, ItemWriter, ItemProcessor 등) 에 @StepScope 를 사용하면 
```
@Bean
@StepScope
public ListItemReader<Integer> test() {
    ....
}
```
Spring Batch 가 Spring 컨테이너를 통해 지정된 `Step 의 실행시점에 해당 컴포넌트를 Spring Bean 으로 생성 한다.` 마찬가지로 @JobScope 는 `Job 실행시점` 에 Bean 이 생성된다. 즉, `Bean 의 생성 시점을 지정된 Scope 가 실행되는 시점으로 지연` 시킨다. (JobScope, StepScope 는 Job 이 실행되고 끝날 때, Step 이 실행되고 끝날 때 생성/삭제가 이루어진다.)   

이렇게 Bean 의 생성시점을 애플리케이션 실행 시점이 아닌, Step 혹은 Job 의 실행시점으로 지연시키면서 얻는 장점은 크게 2가지가 있다.

1. `JobParameter` 의 `Late Binding` 이 가능하다. 
Job Parameter가 StepContext 또는 JobExecutionContext 레벨에서 할당시킬 수 있습니다.
꼭 Application이 실행되는 시점이 아니더라도 Controller나 Service와 같은 비지니스 로직 처리 단계에서 Job Parameter를 할당시킬 수 있다.

2. 동일한 컴포넌트를 병렬 혹은 동시에 사용할 때 유용하다.
Step 안에 Tasklet이 있고, 이 Tasklet은 멤버 변수와 해당 멤버 변수를 변경하는 로직이 있다고 가정해보면, 이 경우 `@StepScope` 없이 Step을 병렬로 실행시키게 되면 `서로 다른 Step에서 하나의 Tasklet을 두고 마구잡이로 상태를 변경하려고 할 것 이다.` 하지만 `@StepScope` 가 있다면 `각각의 Step 에서 별도의 Tasklet 을 생성하고 관리하기 때문에 서로의 상태를 침범할 일이 없다.`

## JobParameters 를 사용하기 위해선 반드시 @StepScope, @JobScope 로 Bean 을 생성해야한다.
JobParameters 는 @Value 를 통해서 가능하며 JobParameters 는 Step 이나, Tasklet, Reader 등 Batch 컴포넌트 Bean 의 생성 시점에 호출할 수 있지만, 정확히는 `Scope Bean 을 생성할때만 가능` 하다. 즉, `@StepScope, @JobScope 는 Bean 을 생성할때만 JobParameters 가 생성`되기 때문에 사용할 수 있다.    

예시로 하기 코드를 보자. (메소드를 통해 Bean 을 생성하지 않고, 클래스에 직접 Bean 을 생성, Job 과 Step 의 코드에서 @Bean 과 @Value("#{jobParameters[requestDate]}") 를 제거하고 SimpleJobTasklet 을 생성자 DI로 받도록 변경)     
```
...

// 생성자 DI
private final SimpleJobTasklet tasklet1;

@Bean
public Job simpleJob() {
    return jobBuilderFactory.get("simpleJob")
        .start(simpleStep1())
        .next(simpleStep2(null))
        .build();
}

//@Bean
//@JobScope
public Step simpleStep1() {
    return stepBuilderFactory.get("simpleStep1")
        .tasklet(tasklet1)  // 생성자 DI
        .build();
}

...
```

그 다음 SimpleJobTasklet 은 하기 코드와 같이 @Componet 와 @StepScope 로 Scope 가 Step 인 Bean 으로 생성하고 @Value("#{jobParameters[requestDate]}") 를 Tasklet 의 멤버변수로 할당한다.
```
package com.example.tasklet;

import org.springframework.batch.core.StepContribution;
import org.springframework.batch.core.configuration.annotation.StepScope;
import org.springframework.batch.core.scope.context.ChunkContext;
import org.springframework.batch.core.step.tasklet.Tasklet;
import org.springframework.batch.repeat.RepeatStatus;
import org.springframework.beans.factory.annotation.Value;
import org.springframework.stereotype.Component;

import lombok.extern.slf4j.Slf4j;

@Slf4j
@Component
@StepScope
public class SimpleJobTasklet implements Tasklet {

	@Value("#{jobParameters[requestDate]}")
	private String requestDate;

	public SimpleJobTasklet() {
		log.info(">>>>>>> Tasklet 생성");
	}

	@Override
	public RepeatStatus execute(StepContribution contribution, ChunkContext chunkContext) {
		log.info(">>>>>>>> This is Step1 {}", requestDate);
		return RepeatStatus.FINISHED;
	}

}
```

> 이렇게 하면 `메소드의 파라미터로 JobParameter를 할당받지 않고, 클래스의 멤버 변수로 JobParameter를 할당 받도록 해도 정상적으로 JobParameter를 받아` 사용할 수 있습니다. 왜냐하면 `SimpleJobTasklet Bean이 @StepScope로 생성되었기 때문`이다. (반면, SimpleJobTasklet Bean을 일반 singleton Bean으로 생성할 경우(=@StepScope 를 주석처리 할 경우) 아래와 에러가 발생한다.) 즉, JobParameters를 사용하기 위해선 꼭 `@StepScope, @JobScope` 로 Bean을 생성해야한다.

## 참고
- https://jojoldu.tistory.com/