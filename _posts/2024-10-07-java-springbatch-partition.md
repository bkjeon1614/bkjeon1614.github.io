---
layout: post
title: "Spring Batch Partitioning 구현 (작성중)"
subtitle: "2024-10-07-java-springbatch-partition.md"
date: 2024-10-07 21:40:00
author: "전봉근"
header-img: "img/posts/android-bg.jpg"
comments: true
tags: [java, spring batch, jvm]
---

## Spring Batch Partitioning 구현

#### 예제코드
[예제코드](https://github.com/bkjeon1614/java-example-code/tree/main/java17/spring-batch-mybatis-codebase)

#### Spring Batch Partitioning 이란
파티셔닝은 매니저 Step 이 대량의 데이터 처리를 위해 지정된 수의 작업자 (Worker) Step 으로 병렬처리 하는 방식이다.
![java-springbatch-partition-1](/img/posts/language/java/springbatch/java-springbatch-partition-1.png)       

#### Multi Thread Step 과 비교
- 멀티스레드 Step 은 단일 Step 을 Chunk 단위로 스레드를 생성해 분할처리 한다.
  - `어떤 쓰레드에서 어떤 데이터들을 처리하게 할지 세밀한 조정이 불가능`
  - 해당 Step의 ItemReader/ItemWriter 등이 멀티스레드 환경을 지원하는지 유무가 굉장히 중요
- 파티셔닝은 독립적인 Step (Worker Step)을 구성하고, 그에 따른 각각 별도의 StepExecution 파라미터 환경을 가지게 하여 처리
  - 멀티스레드 Step과는 별개로 ItemReader/ItemWriter의 멀티쓰레드 환경 지원 여부가 중요하지 않음
  - `각 파티션은 독립적으로 실행되며, 서로 영향을 받지 않고 작업을 수행`

#### 주요 인터페이스
- Partitioner
  - 파티셔닝된 Step (Worker Step) 을 위한 Step Executions 을 생성하는 인터페이스
  - 인터페이스가 갖고 있는 메서드는 partition(int gridSzie) 이며 gridSize 는 몇 개의 StepExecution 을 생성할지 결정하는 설정 값이고 일반적으로는 StepExecution 당 1개의 Worker Step 을 매핑하기 때문에 Worker Step 의 수와 마찬가지로 보면 된다.
  - 해당 gridSzie 를 이용하여 각 Worker Step 마다 어떤 StepExecution 환경을 갖게 할지는 개발자의 몫이다.
- PartitionHandler
  - 매니저 (마스터) Step이 Worker Step를 어떻게 다룰지를 정의 (Ex: 병렬로 실행할지 실행한다면 스레드풀은 어떻게 관리할지 gridSize 는 몇으로 둘지 등등..)
  - 일반적으로는 Partitioner의 구현체는 개발자가 요구사항에 따라 별도 생성해서 사용하곤 하지만, 자신만의 PartitionHandler를 작성하지는 않는다고 한다.
  - 구현체 종류
    - TaskExecutorPartitionHandler
      - 단일 JVM 내에서 분할 개념을 사용할 수 있도록 같은 JVM 내에서 스레드로 분할 실행
    - MessageChannelPartitionHandler
      - 원격의 JVM에 메타 데이터를 전송

#### 예시
[MybatisSamplePartitionJobConfig.java]    
```
package com.bkjeon.job;

import com.bkjeon.core.listener.CommonChunkListener;
import com.bkjeon.core.listener.CommonStepListener;
import com.bkjeon.feature.entity.sample.Sample;
import com.bkjeon.feature.entity.sample.SampleIdRangePartitioner;
import com.bkjeon.feature.entity.sample.SampleOut;
import com.bkjeon.feature.mapper.sample.SampleMapper;
import java.util.HashMap;
import lombok.RequiredArgsConstructor;
import lombok.extern.slf4j.Slf4j;
import org.apache.ibatis.session.SqlSessionFactory;
import org.mybatis.spring.batch.MyBatisBatchItemWriter;
import org.mybatis.spring.batch.MyBatisPagingItemReader;
import org.mybatis.spring.batch.builder.MyBatisBatchItemWriterBuilder;
import org.mybatis.spring.batch.builder.MyBatisPagingItemReaderBuilder;
import org.springframework.batch.core.Job;
import org.springframework.batch.core.Step;
import org.springframework.batch.core.configuration.annotation.StepScope;
import org.springframework.batch.core.job.builder.JobBuilder;
import org.springframework.batch.core.partition.support.TaskExecutorPartitionHandler;
import org.springframework.batch.core.repository.JobRepository;
import org.springframework.batch.core.step.builder.StepBuilder;
import org.springframework.batch.item.ItemProcessor;
import org.springframework.beans.factory.annotation.Value;
import org.springframework.context.annotation.Bean;
import org.springframework.context.annotation.Configuration;
import org.springframework.core.task.TaskExecutor;
import org.springframework.scheduling.concurrent.ThreadPoolTaskExecutor;
import org.springframework.transaction.PlatformTransactionManager;

/**
 * --job.name=MYBATIS_SAMPLE_PARTITION_JOB requestDate=20240701
 * MYBATIS_SAMPLE_JOB(MybatisSampleJobConfig.class 파티셔닝 개선)
 */
@Slf4j
@Configuration
@RequiredArgsConstructor
public class MybatisSamplePartitionJobConfig {

    private static final String JOB_NAME = "MYBATIS_SAMPLE_PARTITION_JOB";
    private int chunkSize;

    @Value("${chunkSize:10}")
    public void setChunkSize(int chunkSize){
        this.chunkSize = chunkSize;
    }

    private int poolSize;

    @Value("${poolSize:5}")
    public void setPoolSize(int poolSize){
        this.poolSize = poolSize;
    }

    private final SqlSessionFactory sqlSessionFactory;
    private final SampleMapper sampleMapper;

    @Bean(name = JOB_NAME + "_PARTITION_HANDLER")
    public TaskExecutorPartitionHandler partitionHandler(JobRepository jobRepository,
        PlatformTransactionManager platformTransactionManager) {
        // 멀티 스레드로 수행이 가능하도록 TaskExecutorPartitionHandler 구현체를 사용
        TaskExecutorPartitionHandler partitionHandler = new TaskExecutorPartitionHandler();

        // Worker 로 실행할 Step 을 지정
        // Partitioner가 만들어준 StepExecutions 환경에서 개별적으로 실행
        partitionHandler.setStep(mybatisSamplePartitionJobStep(jobRepository, platformTransactionManager));

        // 멀티쓰레드로 실행하기 위해 TaskExecutor 를 지정
        partitionHandler.setTaskExecutor(executor());

        // 쓰레드 개수와 gridSize를 맞추기 위해서 poolSize를 gridSize로 등록
        partitionHandler.setGridSize(poolSize);
        return partitionHandler;
    }

    @Bean(name = JOB_NAME + "_TASK_POOL")
    public TaskExecutor executor() {
        // SimpleAsyncTaskExecutor 를 사용할수도 있지만 해당 구현체는 레드를 계속해서 생성할 수 있기 때문에 실제 운영 환경에서는 대형 장애를 발생시킬 수 있음
        // 그래서 스레드풀내에서 지정된 갯수만큼 스레드만 생성할 수 있도록 ThreadPoolTaskExecutor 사용
        ThreadPoolTaskExecutor executor = new ThreadPoolTaskExecutor();
        executor.setCorePoolSize(poolSize);
        executor.setMaxPoolSize(poolSize);
        executor.setThreadNamePrefix("partition-thread");
        executor.setWaitForTasksToCompleteOnShutdown(Boolean.TRUE);
        executor.initialize();
        return executor;
    }

    @Bean(name = JOB_NAME)
    public Job mybatisSamplePartitionJob(JobRepository jobRepository,
        PlatformTransactionManager platformTransactionManager) {
        return new JobBuilder(JOB_NAME, jobRepository)
            .start(mybatisSamplePartitionJobStepManager(jobRepository, platformTransactionManager))
            .build();
    }

    @Bean(name = JOB_NAME + "_STEP_MANAGER")
    public Step mybatisSamplePartitionJobStepManager(JobRepository jobRepository,
        PlatformTransactionManager platformTransactionManager) {
        return new StepBuilder(JOB_NAME + "_STEP.MANAGER", jobRepository)
            .partitioner(JOB_NAME + "_STEP", partitioner()) // step1에 사용될 Partitioner 구현체를 등록
            .step(mybatisSamplePartitionJobStep(jobRepository, platformTransactionManager)) // 파티셔닝될 Step 을 등록, step1 이 Partitioner 로직에 따라 서로 다른 StepExecutions를 가진 여러개로 생성
            .partitionHandler(partitionHandler(jobRepository, platformTransactionManager))  // 사용할 PartitionHandler 를 등록
            .build();
    }

    @Bean(name = JOB_NAME + "_PARTITIONER")
    @StepScope
    public SampleIdRangePartitioner partitioner() {
        return new SampleIdRangePartitioner(sampleMapper);
    }

    @Bean(name = JOB_NAME + "_STEP")
    public Step mybatisSamplePartitionJobStep(JobRepository jobRepository,
        PlatformTransactionManager platformTransactionManager) {
        return new StepBuilder(JOB_NAME + "_STEP", jobRepository)
            .<Sample, SampleOut>chunk(chunkSize, platformTransactionManager)
            .reader(mybatisSamplePartitionPagingItemReader(null, null))
            .processor(processor())
            .writer(mybatisSamplePartitionItemWriter(null, null))
            .listener(new CommonChunkListener())
            .listener(new CommonStepListener())
            .build();
    }

    @Bean(name = JOB_NAME + "_READER")
    @StepScope
    public MyBatisPagingItemReader<Sample> mybatisSamplePartitionPagingItemReader(
        @Value("#{stepExecutionContext[minId]}") Long minId,
        @Value("#{stepExecutionContext[maxId]}") Long maxId) {

        log.info("reader minId={}, maxId={}", minId, maxId);

        return new MyBatisPagingItemReaderBuilder<Sample>()
            .pageSize(chunkSize)
            .sqlSessionFactory(sqlSessionFactory)
            .parameterValues(new HashMap<>() {{
                put("minId", minId);
                put("maxId", maxId);
            }})
            .queryId("com.bkjeon.feature.mapper.sample.SampleMapper.selectSamplePartitionList")
            .build();
    }

    private ItemProcessor<Sample, SampleOut> processor() {
        return SampleOut::new;
    }

    @Bean(name = JOB_NAME + "_WRITER")
    @StepScope
    public MyBatisBatchItemWriter<SampleOut> mybatisSamplePartitionItemWriter(
        @Value("#{stepExecutionContext[minId]}") Long minId,
        @Value("#{stepExecutionContext[maxId]}") Long maxId) {
        log.info("stepExecutionContext minId={}", minId);
        log.info("stepExecutionContext maxId={}", maxId);
        return new MyBatisBatchItemWriterBuilder<SampleOut>()
            .sqlSessionFactory(sqlSessionFactory)
            .statementId("com.bkjeon.feature.mapper.sample.SampleMapper.insertSample")
            .build();
    }

    /*
    @Bean(name = JOB_NAME + "_WRITER")
    @StepScope
    public ItemWriter<SampleOut> mybatisSamplePartitionItemWriter(
        @Value("#{stepExecutionContext[minId]}") Long minId,
        @Value("#{stepExecutionContext[maxId]}") Long maxId) {
        return items -> {
            log.info("stepExecutionContext minId={}", minId);
            log.info("stepExecutionContext maxId={}", maxId);
        };
    }
     */

}
```      
      
[SampleIdRangePartitioner.java]
```
package com.bkjeon.feature.entity.sample;

import com.bkjeon.feature.mapper.sample.SampleMapper;
import java.util.HashMap;
import java.util.Map;
import lombok.RequiredArgsConstructor;
import lombok.extern.slf4j.Slf4j;
import org.springframework.batch.core.partition.support.Partitioner;
import org.springframework.batch.item.ExecutionContext;

/**
 * Partitioner 는 각 Worker Step들에게 어떤 Step Executions 변수를 가지게 할지를 결정하고, 그에 따라 생성할 Worker Step 수를 결정
 * 해당 Partitioner 는 데이터의 시작 PK 값과 끝 PK 값을 조회해 파티션별로 분할해서 할당하여 처리
 */
@Slf4j
@RequiredArgsConstructor
public class SampleIdRangePartitioner implements Partitioner {

    private final SampleMapper sampleMapper;

    @Override
    public Map<String, ExecutionContext> partition(int gridSize) {
        long min = sampleMapper.findMinId();
        long max = sampleMapper.findMaxId();
        long targetSize = (max - min) / gridSize + 1;

        Map<String, ExecutionContext> result = new HashMap<>();
        long number = 0;
        long start = min;
        long end = start + targetSize - 1;

        while (start <= max) {
            ExecutionContext value = new ExecutionContext();
            result.put("partition" + number, value);

            if (end >= max) {
                end = max;
            }

            value.putLong("minId", start); // 각 파티션마다 사용될 minId
            value.putLong("maxId", end); // 각 파티션마다 사용될 maxId
            start += targetSize;
            end += targetSize;
            number++;
        }

        return result;
    }

}
```     
      
[SampleIdRangePartitionerTest.java]     
```
package com.bkjeon.feature.entity.sample;

import static org.assertj.core.api.Assertions.assertThat;

import com.bkjeon.feature.mapper.sample.SampleMapper;
import java.util.Map;
import org.junit.jupiter.api.Test;
import org.junit.jupiter.api.extension.ExtendWith;
import org.mockito.Mock;
import org.mockito.Mockito;
import org.mockito.junit.jupiter.MockitoExtension;
import org.springframework.batch.item.ExecutionContext;

@ExtendWith(MockitoExtension.class)
public class SampleIdRangePartitionerTest {

    private static SampleIdRangePartitioner partitioner;

    @Mock
    private SampleMapper sampleMapper;

    @Test
    void gridSize에_맞게_id가_분할된다() throws Exception {
        // given
        // (1) findMinId(), findMaxId() 메서드가 호출되면 각각 1L, 10L을 반환하도록 설정
        Mockito.lenient()
            .when(sampleMapper.findMinId())
            .thenReturn(1L);

        Mockito.lenient()
            .when(sampleMapper.findMaxId())
            .thenReturn(10L);

        // (2) SampleIdRangePartitioner 인스턴스 생성
        partitioner = new SampleIdRangePartitioner(sampleMapper);

        // when
        // (3) gridSize가 5일 때 partition() 메서드 호출 (5개의 파티션으로 분할하면 각 파티션당 2개씩 할당)
        Map<String, ExecutionContext> executionContextMap = partitioner.partition(5);

        // then
        // (4) 첫번째 파티션에 등록된 minId, maxId를 검증 (예상결과: minId=1, maxId=2)
        ExecutionContext partition1 = executionContextMap.get("partition0");
        assertThat(partition1.getLong("minId")).isEqualTo(1L);
        assertThat(partition1.getLong("maxId")).isEqualTo(2L);

        // (5) 마지막 파티션에 등록된 minId, maxId를 검증 (예상결과: minId=9, maxId=10)
        ExecutionContext partition5 = executionContextMap.get("partition4");
        assertThat(partition5.getLong("minId")).isEqualTo(9L);
        assertThat(partition5.getLong("maxId")).isEqualTo(10L);
    }

}
```     
    
[sample.xml]       
```
...

<select id="selectSamplePartitionList" resultMap="selectSampleListMap">
    SELECT
        id,
        amount,
        tx_name,
        tx_date_time
    FROM sample
    WHERE id BETWEEN #{minId} AND #{maxId}
</select>

<insert id="insertSample" parameterType="com.bkjeon.feature.entity.sample.SampleOut">
    INSERT INTO sample_out (
        amount,
        tx_name,
        tx_date_time
    ) VALUES (
        #{amount},
        #{txName},
        #{txDateTime}
    )
</insert>
```

#### 설명
SampleIdRangePartitionerTest.java 기준으로 설명하자면 gridSize 가 5개 이므로 5개의 파티션이 생성되며 minId, maxId 는 아래와 같이 할당되며 쿼리가 수행된다.
```
partition0 (minId:1, maxId:2)
partition1 (minId:3, maxId:4)
partition2 (minId:5, maxId:6)
partition3 (minId:7, maxId:8)
partition4 (minId:9, maxId:10)
```       
![java-springbatch-partition-2](/img/posts/language/java/springbatch/java-springbatch-partition-2.png)       

#### 참고
- https://docs.spring.io/
- https://jojoldu.tistory.com/