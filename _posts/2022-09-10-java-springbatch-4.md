---
layout: post
title : "Spring Batch 4편 - Spring Batch Job Flow"
subtitle : "2022-09-10-java-springbatch-4.md"
date: 2022-09-10 19:50:00
author: "전봉근"
header-img: "img/posts/android-bg.jpg"
comments: true
tags: [spring, java, jvm, spring batch]
---

# Spring Batch Job Flow
이전글: https://bkjeon1614.tistory.com/739  
작업코드: [작업코드](https://github.com/bkjeon1614/java-example-code/tree/develop/spring-batch-study/spring-batch-study-jpa)

실제 비지니스 로직을 처리하는 기능은 `Step` 에 구현되어 있다. 즉, `Batch 로 실제 처리하고자 하는 기능과 설정을 모두 포함한다.` Job 내부의 Step 들 간에 순서 또는 처리 프로세스를 제어하기 위해 여러 Step 들이 어떻게 관리해야 하는지 차근차근 알아보자.

## Next
`next()` 는 순차적으로 Step 들을 연결시킬 때 사용 즉, step1 -> step2 -> ... 와 같이 하나씩 실행할 때 사용하면 좋다. 샘플코드를 작성해보자.   
[NextJobConfiguration.java]    
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

@Slf4j
@Configuration
@RequiredArgsConstructor
public class NextJobConfiguration {

    private final JobBuilderFactory jobBuilderFactory;
    private final StepBuilderFactory stepBuilderFactory;

    @Bean
    public Job nextJob() {
        return jobBuilderFactory.get("nextJob")
            .start(step1())
            .next(step2())
            .next(step3())
            .build();
    }

    @Bean
    public Step step1() {
        return stepBuilderFactory.get("step1")
            .tasklet((contribution, chunkContext) -> {
                log.info(">>>>> Step1");
                return RepeatStatus.FINISHED;
            })
            .build();
    }

    @Bean
    public Step step2() {
        return stepBuilderFactory.get("step2")
            .tasklet((contribution, chunkContext) -> {
                log.info(">>>>> Step2");
                return RepeatStatus.FINISHED;
            })
            .build();
    }

    @Bean
    public Step step3() {
        return stepBuilderFactory.get("step3")
            .tasklet((contribution, chunkContext) -> {
                log.info(">>>>> Step3");
                return RepeatStatus.FINISHED;
            })
            .build();
    }

}
```
그 다음 Program arguments 에서 `version=1` 로 변경한 다음    
![springbatch-24](/img/posts/language/java/springbatch-24.png)       

실행하게되면    
![springbatch-25](/img/posts/language/java/springbatch-25.png)       
기존에 작성하였던 `simpleJob` 또한 실행되었다는 것을 로그로 확인할 수 있다. 이 경우 지정한 Batch Job 만 실행되도록 설정해보자.

먼저 프로젝트의 src/main/resources/application.yml 의 설정을 추가하자     
[application.yml]      
```
spring:
  profiles:
    active: local
  batch:
    job:
      names: ${job.name:NONE}

...
```
해당 추가된 batch.job.names 설정값은 `Program arguments 로 job.name 값이 넘어오면 해당 값과 일치하는 Job 만 실행` 하겠다는 설정이다. 여기서 `:NONE` 는 `job.name` 가 있으면 job.name 을 할당하고 없으면 `NONE` 을 할당하겠다는 의미이다. 또한 해당 값이 `NONE` 이면 `어떤 배치도 실행하지 않겠다` 라는 의미이다 즉, `혹여나 값이 없을 때 모든 배치들이 실행하지 않도록 막는` 중요한 역할을 한다.

이제 IDE 의 Program arguments 에 `--job.name=nextJob` 을 입력하고 실행해보자. (`이전 실행에서 version1 이 이미 실행되었으니 version2 로 변경해야 한다.`)
![springbatch-26](/img/posts/language/java/springbatch-26.png)       
![springbatch-27](/img/posts/language/java/springbatch-27.png)         

> 위와 같이 필요한 job 만 변경해주면서 실행하면되고, 실제 운영에서는 `java -jar batch.jar --job.name=nextJob` 과 같이 실행해주면 된다.

## 조건별 흐름 제어
먼저 Next 가 순차적으로 Step 의 순서를 제어한다는 것을 알게 되었다. 그러나 `앞의 step 에서 오류가 발생하면 나머지 뒤에 step 들은 실행되지 못한다` 는 것이다. 이러한 상황을 대응하기 위하여 `정상일 때 Step B` 로 `오류일 때 Step C` 로 수행하도록 조건별로 Step 을 사용해보자.     
[StepNextConditionalJobConfiguration.java]    
```
package com.example.job;

import org.springframework.batch.core.ExitStatus;
import org.springframework.batch.core.Job;
import org.springframework.batch.core.Step;
import org.springframework.batch.core.configuration.annotation.JobBuilderFactory;
import org.springframework.batch.core.configuration.annotation.StepBuilderFactory;
import org.springframework.batch.repeat.RepeatStatus;
import org.springframework.context.annotation.Bean;
import org.springframework.context.annotation.Configuration;

import lombok.RequiredArgsConstructor;
import lombok.extern.slf4j.Slf4j;

@Slf4j
@Configuration
@RequiredArgsConstructor
public class StepNextConditionalJobConfiguration {

	private final JobBuilderFactory jobBuilderFactory;
	private final StepBuilderFactory stepBuilderFactory;

	@Bean
	public Job stepNextConditionalJob() {
		return jobBuilderFactory.get("stepNextConditionalJob")
            .start(conditionalJobStep1())
                .on("FAILED") // FAILED 일 경우
                .to(conditionalJobStep3()) // step3으로 이동한다.
                .on("*") // step3의 결과 관계 없이
                .end() // step3으로 이동하면 Flow가 종료한다.
            .from(conditionalJobStep1()) // step1로부터
                .on("*") // FAILED 외에 모든 경우
                .to(conditionalJobStep2()) // step2로 이동한다.
                .next(conditionalJobStep3()) // step2가 정상 종료되면 step3으로 이동한다.
                .on("*") // step3의 결과 관계 없이
                .end() // step3으로 이동하면 Flow가 종료한다.
            .end() // Job 종료
            .build();
	}

	@Bean
	public Step conditionalJobStep1() {
		return stepBuilderFactory.get("step1")
            .tasklet((contribution, chunkContext) -> {
                log.info(">>>>> This is stepNextConditionalJob Step1");

                /**
                ExitStatus를 FAILED로 지정한다.
                해당 status를 보고 flow가 진행된다.
                **/
                contribution.setExitStatus(ExitStatus.FAILED);

                return RepeatStatus.FINISHED;
            })
            .build();
	}

	@Bean
	public Step conditionalJobStep2() {
		return stepBuilderFactory.get("conditionalJobStep2")
            .tasklet((contribution, chunkContext) -> {
                log.info(">>>>> This is stepNextConditionalJob Step2");
                return RepeatStatus.FINISHED;
            })
            .build();
	}

	@Bean
	public Step conditionalJobStep3() {
		return stepBuilderFactory.get("conditionalJobStep3")
            .tasklet((contribution, chunkContext) -> {
                log.info(">>>>> This is stepNextConditionalJob Step3");
                return RepeatStatus.FINISHED;
            })
            .build();
	}

}
```
상기 코드의 프로세스는 step1 성공여부에 따라 달라진다
- step1 실패: step1 -> step3
- step1 성공: step1 -> step2 -> step3   

또한 상기 Flow 를 관리하는 코드의 기능을 살펴보면    
- .on()
  - catch 할 `ExitStatus 지정`
  - `*` 인 경우는 결과에 상관없이 모든 ExitStatus 가 지정
- .to()
  - 다음으로 이동할 Step 지정
- .from()
  - `Event Listener` 역할
  - 상태값이 일치하는 상태라면 `.to()` 에 지정된 `step` 을 호출
  - 만약, step1 의 이벤트 캐치가 FAILED 로 되어있는 상태일 때 `추가로 이벤트 캐치를 하려면 from 을 사용해야함` 
- .end()
  - end 는 FlowBuilder 를 반환하는 end 와 FlowBuilder 을 종료하는 end 로 총 2개의 end 가 있다.
  - `on("*")` 뒤에 있는 end 는 FlowBuilder 를 반환하는 end (해당 end 는 계속해서 `.from()` 을 이어갈 수 있음)
  - `build()` 앞에 있는 end 는 FlowBuilder 를 종료하는 end

이제 여기서 실행하면 `conditionalJobStep1` 의 분기처리를 위해 상태값 조정이 필요한 `ExitStatus.FAILED` 코드로 먼저 FAILED 를 발생시켜 step1 -> step3 flow 를 테스트를 해보자. (원하는 상황에 맞게 분기로직을 작성하려면 contribution.setExitStatus 의 값을 변경해주면 된다. 또한 여기서 중요한건 `.on()` 이 캐치하는 상태값이 BatchStatus 가 아닌 ExitStatus 라는걸 확인할 수 있다.)    
![springbatch-28](/img/posts/language/java/springbatch-28.png)        

그 다음 contribution.setExitStatus(ExitStatus.FAILED); 를 주석처리 후 실행해보자.     
![springbatch-29](/img/posts/language/java/springbatch-29.png)        
정상 Flow 인 step1 -> step2 -> step3 순으로 차례대로 수행된 것을 확인할 수 있다.

## Batch Status VS Exit Status
상기 내용에 사용된 `ExitStatus` 와 그리고 이번에 비교할 `BatchStatus` 의 차이를 알아보자. 

#### Batch Status   
- Job 또는 Step 의 실행 결과를 Spring 에서 기록할 때 사용하는 Enum 이다.
- 사용되는 값으로는 `COMPLETED, STARTING, STARTED, STOPPING, STOPPED, FAILED, ABANDONED, UNKNOWN` 이 있으며 단어와 동일한 뜻으로 이해하면 된다.

#### ExitStatus
`ExitStatus` 는 `Step 의 실행 후 상태` 를 의미하며, 상기 샘플코드중 `.on("FAILED").to(stepB())` 에서 `.on()` 메소드가 참조하는 것은 BatchStatus 로 생각할 수 있지만 실제 참조되는 값은 Step 의 `ExitStatus` 이다. (ExitStatus 는 Enum 이 아니다.) 또한 해석해보면 `exitCode 가 FAILED 로 끝나면 stepB 를 실행` 하라는 의미이다.    

또한 여기서 본인만의 커스텀한 exitCode 가 필요하면 어떻게 해야될까? 예제 코드를 하나 참고해보자.    
```
.start(step1())
    .on("FAILED")
    .end()
.from(step1())
    .on("COMPLETED WITH SKIPS")
    .to(errorPrint1())
    .end()
.from(step1())
    .on("*")
    .to(step2())
    .end()
```
상기 실행 결과는 아래와 같다.   
- step1 실패하며, job 또한 실패하게 된다.
- step1 성공적으로 수행되어 step2 가 수행
- step1 성공적으로 완료되며, `COMPLETED WITH SKIPS` 의 exit 코드로 종료 된다. (COMPLETED WITH SKIPS 는 ExitStatus 에 없으므로 커스텀한 코드이다. 그러므로 원하는대로 처리되길 위해서는 별도의 로직이 필요)   
  [별도로직 샘플코드]
  ```
  // StepExecutionListener 에서 먼저 Step 성공여부 확인 후 StepExecution의 skip 횟수가 0보다 클 경우 COMPLETED WITH SKIPS 의 exitCode를 갖는 ExitStatus를 반환합니다.
  public class CompleteSkipListener extends StepExecutionListenerSupport {
      public ExitStatus afterStep(StepExecution stepExecution) {
          String exitCode = stepExecution.getExitStatus().getExitCode();
          if (!exitCode.equals(ExitStatus.FAILED.getExitCode()) && stepExecution.getSkipCount() > 0) {
              return new ExitStatus("COMPLETED WITH SKIPS");
          } else {
              return null;
          }
      } 
  }
  ```

#### Decide
위의 진행했던 분기 처리 방식은 아래와 같은 문제가 존재한다.
- Step이 담당하는 역할이 2개 이상이 됨
  - 실제 해당 Step이 처리해야할 로직외에도 분기처리를 시키기 위해 ExitStatus 조작이 필요
- 다양한 분기 로직 처리의 어려움
  - ExitStatus를 커스텀하게 고치기 위해선 Listener를 생성하고 Job Flow에 등록하는 등 번거로움이 존재    
   
이러한 문제들 때문에 Spring Batch 에서는 Step 들의 Flow 속에서 `분기만 담당하는 타입`이 있다. `JobExecutionDecider` 이며 샘플코드를 참고해보자.   
[DeciderJobConfiguration.java]    
```
package com.example.job;

import java.util.Random;

import org.springframework.batch.core.Job;
import org.springframework.batch.core.JobExecution;
import org.springframework.batch.core.Step;
import org.springframework.batch.core.StepExecution;
import org.springframework.batch.core.configuration.annotation.JobBuilderFactory;
import org.springframework.batch.core.configuration.annotation.StepBuilderFactory;
import org.springframework.batch.core.job.flow.FlowExecutionStatus;
import org.springframework.batch.core.job.flow.JobExecutionDecider;
import org.springframework.batch.repeat.RepeatStatus;
import org.springframework.context.annotation.Bean;
import org.springframework.context.annotation.Configuration;

import lombok.RequiredArgsConstructor;
import lombok.extern.slf4j.Slf4j;

@Slf4j
@Configuration
@RequiredArgsConstructor
public class DeciderJobConfiguration {

    private final JobBuilderFactory jobBuilderFactory;
    private final StepBuilderFactory stepBuilderFactory;

    @Bean
    public Job deciderJob() {
      return jobBuilderFactory.get("deciderJob")
          .start(startStep())
          .next(decider()) // 홀수 | 짝수 구분
          .from(decider()) // decider 의 상태가
              .on("ODD") // ODD 일 경우
              .to(oddStep()) // oddStep 로 간다.
          .from(decider()) // decider 의 상태가
              .on("EVEN") // EVEN 일 경우
              .to(evenStep()) // evenStep 로 간다.
          .end() // builder 종료
          .build();
    }

    @Bean
    public Step startStep() {
      return stepBuilderFactory.get("startStep")
              .tasklet((contribution, chunkContext) -> {
                  log.info(">>>>> Start!");
                  return RepeatStatus.FINISHED;
              })
              .build();
    }

    @Bean
    public Step evenStep() {
      return stepBuilderFactory.get("evenStep")
              .tasklet((contribution, chunkContext) -> {
                  log.info(">>>>> 짝수입니다.");
                  return RepeatStatus.FINISHED;
              })
              .build();
    }

    @Bean
    public Step oddStep() {
      return stepBuilderFactory.get("oddStep")
              .tasklet((contribution, chunkContext) -> {
                  log.info(">>>>> 홀수입니다.");
                  return RepeatStatus.FINISHED;
              })
              .build();
    }

    @Bean
    public JobExecutionDecider decider() {
        return new OddDecider();
    }

    // JobExecutionDecider 인터페이스를 구현한 OddDecider
    public static class OddDecider implements JobExecutionDecider {

        // 랜덤하게 숫자를 생성하여 홀수/짝수인지에 따라 서로 다른 상태를 반환한다. (주의점은 여기선 Step으로 처리하는게 아니기 때문에 ExitStatus가 아닌 FlowExecutionStatus로 상태를 관리함)
        @Override
        public FlowExecutionStatus decide(JobExecution jobExecution, StepExecution stepExecution) {
            Random rand = new Random();

            int randomNumber = rand.nextInt(50) + 1;
            log.info("랜덤숫자: {}", randomNumber);

            // 여기서 EVEN, ODD 라는 상태는 from().on() 에 사용하고 있다.
            if (randomNumber % 2 == 0) {
                return new FlowExecutionStatus("EVEN");
            } else {
                return new FlowExecutionStatus("ODD");
            }
        }

    }

}
```   
해당 Batch 의 Flow 는 `startStep() -> addDecider 에서 홀수인지 짝수인지 구분 -> oddStep or evenStep 진행` 이다. decider 를 flow 사이에 넣는 로직은 아래와 같다.
- start()
  - 첫 번째 step 을 시작
- next()
  - `startStep()` 이후에 `decider()` 실행
- from()
  - from 은 이벤트 리스너 역할
  - `decider()` 의 상태값을 보고 일치하면 `to()` 에 포함된 `step` 을 호출

상기 코드를 보면 모든 조건에 대한 부분을 `OddDecider` 가 전담한다. 즉, `역할과 책임이 분리` 되어있다. 코드를 실행하면 홀수/짝수가 나오면서 서로 다른 step (oddStep, evenStep) 이 실행되는 것을 확인할 수 있다.   
![springbatch-30](/img/posts/language/java/springbatch-30.png)        


## 참고
- https://jojoldu.tistory.com/