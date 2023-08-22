---
layout: post
title: "[Redis 시리즈 2편] Lettuce 를 사용한 Read / Write 분리"
subtitle: "2023-08-09-java-redis-lettuce-readwrite.md"
date: 2023-07-30 18:40:00
author: "전봉근"
header-img: "img/posts/android-bg.jpg"
comments: true
tags: [java, spring, spring boot, redis, cache, lettuce]
---

# [Redis 시리즈 2편] Lettuce 를 사용한 Read / Write 분리
코드는 [https://github.com/bkjeon1614/java-example-code/tree/develop/bkjeon-mybatis-codebase](https://github.com/bkjeon1614/java-example-code/tree/develop/bkjeon-mybatis-codebase) 참고 부탁드립니다.    
   
> 원활한 세팅을 위해 기존 1편에서 설치된 Redis 를 제거 후 진행하자.
  

## 환경 (Reader DNS 를 통한 Replica Node 들에 분산하는 방법으로 가정)
- Redis Master 1대 Slave 2대 Docker Compose 로 설치 (with. Docker)    

```
$ docker network create app-tier --driver bridge
```
     
[docker-compose.yml]     

```
version: '2'

networks:
  app-tier:
    driver: bridge

services:
  redis:
    image: 'bitnami/redis:latest'
    environment:
      - REDIS_REPLICATION_MODE=master
      - ALLOW_EMPTY_PASSWORD=yes
    networks:
      - app-tier
    ports:
      - 6379:6379
  redis-slave-1:
    image: 'bitnami/redis:latest'
    environment:
      - REDIS_REPLICATION_MODE=slave
      - REDIS_MASTER_HOST=redis
      - ALLOW_EMPTY_PASSWORD=yes
    ports:
      - 6479:6379
    depends_on:
      - redis
    networks:
      - app-tier
  redis-slave-2:
    image: 'bitnami/redis:latest'
    environment:
      - REDIS_REPLICATION_MODE=slave
      - REDIS_MASTER_HOST=redis
      - ALLOW_EMPTY_PASSWORD=yes
    ports:
      - 6579:6379
    depends_on:
      - redis
    networks:
      - app-tier
```

```
// 컨테이너 생성
$ docker-compose up -d
```

## 적용
- yml 에 Replica Node 정보 추가   
  [application.yml]   
  ```
  ...
  spring:
    ...
    redis:
      base:
        host: localhost
        port: 6379
        database: 0
        timeout: 50
        replicas:
          - host: localhost
            port: 6479
            database: 0
            timeout: 50
          - host: localhost
            port: 6579
            database: 0
            timeout: 50

  ...
  ```

- RedisProperties 를 상속받는 RedisReplicaProperties 생성      
  [RedisReplicaProperties.java]    

  ```
  package com.example.bkjeon.base.config.redis;

  import java.util.List;

  import org.springframework.boot.autoconfigure.data.redis.RedisProperties;
  import org.springframework.boot.context.properties.ConfigurationProperties;
  import org.springframework.context.annotation.Configuration;

  import lombok.Getter;
  import lombok.Setter;

  @Configuration
  @ConfigurationProperties(prefix = "spring.redis.base")
  @Getter
  @Setter
  public class RedisReplicaProperties {

      private String host;
      private int port;
      private RedisProperties main;
      private List<RedisProperties> replicas;

  }
  ```

- RedisConnectionFactory 수정 (ReadFrom 적용 및 Replica 정보추가)     
  [RedisConfig.java]      

  ```
  @Bean
  public RedisConnectionFactory redisConnectionFactory() {
 	  LettuceConnectionFactory connectionFactory;

	  LettuceClientConfiguration clientConfig = LettuceClientConfiguration
	  	  .builder()
		    .readFrom(ReadFrom.REPLICA_PREFERRED)
		    .commandTimeout(Duration.ofMillis(500))
		    .build();
	  RedisStaticMasterReplicaConfiguration staticMasterReplicaConfiguration =
	  	  new RedisStaticMasterReplicaConfiguration(redisProperties.getHost(), redisProperties.getPort());

	  redisProperties.getReplicas()
	  	  .forEach(replica -> staticMasterReplicaConfiguration.addNode(replica.getHost(), replica.getPort()));
	  connectionFactory = new LettuceConnectionFactory(staticMasterReplicaConfiguration, clientConfig);

	  return connectionFactory;
  }  
  ```    


## 테스트

```
$ redis-cli -h localhost -p 6379 monitor | grep foo
$ redis-cli -h localhost -p 6479 monitor | grep foo
$ redis-cli -h localhost -p 6579 monitor | grep foo
```
    
![java-redis-lettuce-2](/img/posts/language/java/redis/java-redis-lettuce-2.png)       