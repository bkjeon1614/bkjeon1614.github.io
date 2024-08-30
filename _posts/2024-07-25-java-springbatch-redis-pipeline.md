---
layout: post
title: "Spring Batch + Redis Pipeline 으로 Aggregation 구현한 성능 개선"
subtitle: "2024-07-25-java-springbatch-redis-pipeline.md"
date: 2024-07-25 20:40:00
author: "전봉근"
header-img: "img/posts/android-bg.jpg"
comments: true
tags: [java, spring, spring batch, redis, db, mysql]
---

## Spring Batch + Redis Pipeline 으로 Aggregation 구현한 성능 개선 
코드 참고는 https://github.com/bkjeon1614/java-example-code/tree/develop/spring-batch-mybatis-codebase 에서 참고 부탁드립니다.

### Redis Pipeline 이란
Redis의 pipeline은 여러 개의 명령어를 한 번에 보내고, 그 결과를 한 번에 받아올 수 있는 메커니즘입니다. 이를 통해 네트워크 오버헤드를 줄이고 Redis 서버의 처리 성능을 최적화할 수 있다.
또한 주의해야할 점은, Redis 서버의 처리량(capacity)을 고려하여 pipeline의 chunk size를 결정해야 한다.     


### 주의사항
- Request Chunk Size: 먼저 요청하는 chunk size 에 대해 설명하자면 너무 작은 chunk size는 네트워크 오버헤드를 줄이지만 Redis 서버에 부하를 증가시킬 수 있고, 너무 큰 chunk size는 한 번에 처리해야 할 명령어 개수가 많아져서 오히려 성능을 저하시킬 수 있다.
공식문서 상에서는 명시적으로 정해진 값은 없지만, 일반적으로 100에서 1000개 사이의 명령어를 포함하는 것이 효율적인 경우가 많다고 한다.
- Command Timeout: 네트워크 레이턴시를 고려하자면 Pipeline을 사용할 때는 네트워크 지연에 의한 영향을 줄일 수 있지만, 여전히 Redis 서버의 응답 시간은 중요하다. 일반적으로 5초에서 10초 사이의 commandTimeout 값을 고려할 수 있습니다. (너무 짧게 설정하면 Pipeline 이 타임아웃이 될 수 있다.)
  - Error Example: org.springframework.data.redis.connection.RedisPipelineException: Pipeline contained one or more invalid commands...


### 실습
RedisTemplate 의 executePipelined 메소드와 RedisCallback을 사용하여 파이프라이닝을 사용할 수 있다.       
      
실습사양
- Java 17
- Gradle 8.8
- Spring Batch 5.1.2

1. table schema 정의
   ```
    create table sample (
      id         bigint not null auto_increment,
      amount     bigint,
      tx_name     varchar(255),
      tx_date_time datetime,
      primary key (id)
    ) engine = InnoDB;

    create table sample_out (
      id         bigint not null auto_increment,
      amount     bigint,
      tx_name     varchar(255),
      tx_date_time datetime,
      primary key (id)
    ) engine = InnoDB;
   ```      
     
2. RedisConfig 에 redisTemplate bean 추가
   ```
   ...

  @Bean
  public RedisTemplate<String, Object> redisTemplate() {
      RedisTemplate<String, Object> redisTemplate = new RedisTemplate<>();
      redisTemplate.setConnectionFactory(redisConnectionFactory());
      redisTemplate.setKeySerializer(new StringRedisSerializer());
      redisTemplate.setValueSerializer(new StringRedisSerializer());
      return redisTemplate;
  }   
   ```        
       
3. Job 생성      
   [MybatisSampleRedisJobConfig.java]    
   ```
    package com.bkjeon.job;

    import com.bkjeon.feature.entity.sample.Sample;
    import com.bkjeon.feature.entity.sample.SampleOut;
    import com.bkjeon.feature.mapper.sample.SampleMapper;
    import java.time.LocalDateTime;
    import java.util.ArrayList;
    import java.util.List;
    import lombok.RequiredArgsConstructor;
    import lombok.extern.slf4j.Slf4j;
    import org.apache.ibatis.session.SqlSessionFactory;
    import org.mybatis.spring.batch.MyBatisBatchItemWriter;
    import org.mybatis.spring.batch.MyBatisPagingItemReader;
    import org.mybatis.spring.batch.builder.MyBatisBatchItemWriterBuilder;
    import org.mybatis.spring.batch.builder.MyBatisPagingItemReaderBuilder;
    import org.springframework.batch.core.Job;
    import org.springframework.batch.core.Step;
    import org.springframework.batch.core.configuration.annotation.JobScope;
    import org.springframework.batch.core.job.builder.JobBuilder;
    import org.springframework.batch.core.repository.JobRepository;
    import org.springframework.batch.core.step.builder.StepBuilder;
    import org.springframework.batch.core.step.tasklet.Tasklet;
    import org.springframework.batch.item.ItemProcessor;
    import org.springframework.batch.repeat.RepeatStatus;
    import org.springframework.beans.factory.annotation.Value;
    import org.springframework.context.annotation.Bean;
    import org.springframework.context.annotation.Configuration;
    import org.springframework.data.redis.connection.StringRedisConnection;
    import org.springframework.data.redis.core.RedisCallback;
    import org.springframework.data.redis.core.StringRedisTemplate;
    import org.springframework.transaction.PlatformTransactionManager;

    /**
    * --job.name=MYBATIS_SAMPLE_REDIS_JOB requestDate=20240701
    */
    @Slf4j
    @Configuration
    @RequiredArgsConstructor
    public class MybatisSampleRedisJobConfig {

        private static final String JOB_NAME_PREFIX = "MYBATIS_SAMPLE_REDIS";
        private static final String REDIS_KEY_PREFIX = "SAMPLE_";
        private static final int CHUNK_SIZE = 1000;

        private final StringRedisTemplate redisTemplate;
        private final SqlSessionFactory sqlSessionFactory;

        @Bean
        public Job mybatisSampleRedisJob(JobRepository jobRepository,
            Step mybatisSampleRedisJobStep1, Step mybatisSampleRedisJobStep2) {
            return new JobBuilder(JOB_NAME_PREFIX + "_JOB", jobRepository)
                .start(mybatisSampleRedisJobStep1)
                .next(mybatisSampleRedisJobStep2)
                .build();
        }

        @Bean
        @JobScope
        public Step mybatisSampleRedisJobStep1(JobRepository jobRepository,
            Tasklet mybatisSampleDataToRedisTasklet, PlatformTransactionManager platformTransactionManager) {
            log.info(">>>>>>>>>>>>>>>>>>>>>>>>>>>>>>>>>>> mybatisSampleRedisJobStep1");
            return new StepBuilder(JOB_NAME_PREFIX + "_JOB_STEP2", jobRepository)
                .tasklet(mybatisSampleDataToRedisTasklet, platformTransactionManager).build();
        }

        /**
        * Redis Pipeline Test (pipeline 1000 단위로 처리)
        * [사양]
        * - version: 7.0.5
        * - set 명령기준으로 한거라 재 측정필요
        * 1. 10만건
        *    - asis: 117초
        *    - tobe: 7초(1000)
        * 2. 100만건
        *    - asis: 1225초
        *    - tobe: 64초(1000) - chunk size 를 늘려도 처리속도는 크게 변화가 없다. 레디스 스펙에 영향을 받나? -> Redis 사용량에 따라 성능이 달라질 수 있다.
        * * 약 19배 증가
        */
        @Bean
        public Tasklet mybatisSampleDataToRedisTasklet() {
            return ((contribution, chunkContext) -> {
                log.info(">>>>> This is mybatisSampleDataToRedisTasklet");

                // Mock Data
                List<Sample> sampleList = new ArrayList<>();
                for (long c=1; c <= 1000000; c++) {
                    sampleList.add(
                        Sample.builder().id(c).txName("TEST" + c).amount(c * 1000).txDateTime(LocalDateTime.now()).build());
                }

                long beforeTime = System.currentTimeMillis();
                int size = sampleList.size();
                for (int i = 0; i < size; i += CHUNK_SIZE) {
                    List<Sample> chunk = sampleList.subList(i, Math.min(size, i + CHUNK_SIZE));

                    redisTemplate.executePipelined(
                        (RedisCallback<Object>) connection -> {
                            StringRedisConnection stringRedisConn = (StringRedisConnection) connection;
                            for (Sample sample: chunk) {
                                stringRedisConn.set(REDIS_KEY_PREFIX + sample.getId(), "value" + sample.getTxName());
                            }
                            return null;
                        });
                }

                long afterTime = System.currentTimeMillis();
                long secDiffTime = (afterTime - beforeTime) / 1000;
                log.info(">>>>>>>>>>>>>>>>>>>>>>>>>> Redis 실행시간(s): {}", secDiffTime);
                return RepeatStatus.FINISHED;
            });
        }

        @Bean
        @JobScope
        public Step mybatisSampleRedisJobStep2(JobRepository jobRepository,
            PlatformTransactionManager platformTransactionManager,
            @Value("#{jobParameters[requestDate]}") String requestDate) {
            log.info(">>>>>>>>>>>>>>>>>>>>>>>>>>>>>>>>>>> mybatisSampleRedisJobStep2 requestDate:{} ", requestDate);
            return new StepBuilder(JOB_NAME_PREFIX + "_JOB_STEP2", jobRepository)
                .<Sample, SampleOut>chunk(CHUNK_SIZE, platformTransactionManager)
                .reader(mybatisSampleRedisToRdbPagingItemReader())
                .processor(mybatisSampleToSampleOutItemProcessor())
                .writer(mybatisSampleRedisToRdbItemWriter())
                .build();
        }

        @Bean
        public MyBatisPagingItemReader<Sample> mybatisSampleRedisToRdbPagingItemReader() {
            return new MyBatisPagingItemReaderBuilder<Sample>()
                .pageSize(CHUNK_SIZE)
                .sqlSessionFactory(sqlSessionFactory)
                .queryId("com.bkjeon.feature.mapper.sample.SampleMapper.selectZeroOffsetSampleList")
                .build();
        }

        @Bean
        public ItemProcessor<Sample, SampleOut> mybatisSampleToSampleOutItemProcessor() {
            return item -> SampleOut.builder()
                .id(item.getId())
                .amount(item.getAmount())
                .txName(redisTemplate.opsForValue().get(REDIS_KEY_PREFIX + item.getId()))
                .txDateTime(LocalDateTime.now())
                .build();
        }

        @Bean
        public MyBatisBatchItemWriter<SampleOut> mybatisSampleRedisToRdbItemWriter() {
            return new MyBatisBatchItemWriterBuilder<SampleOut>()
                .sqlSessionFactory(sqlSessionFactory)
                .statementId("com.bkjeon.feature.mapper.sample.SampleMapper.insertSample")
                .build();
        }

    }
   ```      

4. Mapper    
   [SampleMapper.java]
   ```
    import com.bkjeon.feature.entity.sample.Sample;
    import com.bkjeon.feature.entity.sample.SampleOut;
    import java.util.List;
    import org.apache.ibatis.annotations.Mapper;

    @Mapper
    public interface SampleMapper {
        ...

        List<Sample> selectZeroOffsetSampleList();        
        void insertSample(SampleOut sample);

    }
   ```

5. Mapper xml
   ```
    ...

    <select id="selectZeroOffsetSampleList" resultMap="selectSampleListMap">
        SELECT
            id,
            amount,
            tx_name,
            tx_date_time
        FROM sample
        WHERE 1=1
        AND id > #{_skiprows}
        LIMIT 0, #{_pagesize}
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

6. Entity 생성
   [Sample.java]     
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
   [SampleOut.java]    
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
      
> 완료 후 batch 를 동작시켜서 pipeline 을 사용한것과 안한것의 차이를 비교해보면 chunk size 1000 개 기준 약 19배의 성능이 증가한걸 확인할 수 있다.