---
layout: post
title: "[Redis 시리즈 1편] Spring Boot 에 Redis 라이브러리인 Lettuce 적용"
subtitle: "2023-07-30-java-redis-lettuce.md"
date: 2023-07-30 18:40:00
author: "전봉근"
header-img: "img/posts/android-bg.jpg"
comments: true
tags: [java, spring, spring boot, redis, cache]
---

# [Redis 시리즈 1편] Spring Boot 에 Redis 라이브러리인 Lettuce 적용
코드는 [https://github.com/bkjeon1614/java-example-code/tree/develop/bkjeon-mybatis-codebase](https://github.com/bkjeon1614/java-example-code/tree/develop/bkjeon-mybatis-codebase) 참고 부탁드립니다.


## Lettuce
Lettuce 란 Netty(비동기 이벤트 기반 고성능 네트워크 프레임워크) 기반이며 고성능, 확장가능, 스레드세이프등을 지원한다.     


## Lettuce 를 선택한 이유
주로 java 쪽에서는 Lettuce 말고도 Jedis 도 많이 사용한다고 한다. 그러나 Jedis 에 비해 몇 배 이상의 성능, 빠른 피드백, 심플하게 디자인된 코드, [잘 기재된 공식문서](https://lettuce.io/core/release/reference/) 등의 이유로 Lettuce 를 많이 사용하고 있는 추세라고 한다.    


## 환경
1. Redis 설치 (with. Docker)
```
$ sudo docker run --name redis-primary -d -p 6379:6379 redis

// 해당 컨테이너에 접속
$ docker exec -it redis-primary /bin/bash

// 모니터링
$ redis-cli -h localhost -p 6379 monitor | grep foo
```


## 적용
1. 의존성 추가 (Lettuce 는 spring-data-redis 만 추가하면 기본의존성으로 사용이 가능)
```
implementation('org.springframework.boot:spring-boot-starter-data-redis')
```

2. RedisConnectionFactory 인터페이스를 통해 LettuceConnectionFactory를 생성하여 반환하는 코드를 작성한다.    
[RedisConfig.java]     
```
package com.example.bkjeon.base.config.redis;

import java.time.Duration;

import org.springframework.boot.autoconfigure.data.redis.RedisProperties;
import org.springframework.context.annotation.Bean;
import org.springframework.context.annotation.Configuration;
import org.springframework.context.annotation.Primary;
import org.springframework.data.redis.cache.RedisCacheConfiguration;
import org.springframework.data.redis.cache.RedisCacheManager;
import org.springframework.data.redis.connection.RedisConnectionFactory;
import org.springframework.data.redis.connection.RedisStaticMasterReplicaConfiguration;
import org.springframework.data.redis.connection.lettuce.LettuceClientConfiguration;
import org.springframework.data.redis.connection.lettuce.LettuceConnectionFactory;
import org.springframework.data.redis.core.RedisTemplate;
import org.springframework.data.redis.repository.configuration.EnableRedisRepositories;
import org.springframework.data.redis.serializer.GenericJackson2JsonRedisSerializer;
import org.springframework.data.redis.serializer.RedisSerializationContext;
import org.springframework.data.redis.serializer.StringRedisSerializer;

import io.lettuce.core.ReadFrom;
import lombok.RequiredArgsConstructor;

@Configuration
@EnableRedisRepositories
@RequiredArgsConstructor
public class RedisConfig {

	private final RedisProperties redisProperties;

	@Bean
	public RedisTemplate<String, Object> redisTemplate() {
      RedisTemplate<String, Object> redisTemplate = new RedisTemplate<>();
      redisTemplate.setConnectionFactory(redisConnectionFactory());
      redisTemplate.setKeySerializer(new StringRedisSerializer());
      redisTemplate.setValueSerializer(new StringRedisSerializer());
      return redisTemplate;
	}

	@Bean
	public RedisConnectionFactory redisConnectionFactory() {
      LettuceConnectionFactory connectionFactory;

      LettuceClientConfiguration clientConfig = LettuceClientConfiguration
          .builder()
          .readFrom(ReadFrom.SLAVE_PREFERRED)
          .commandTimeout(Duration.ofMillis(500))
          .build();
      RedisStaticMasterReplicaConfiguration staticMasterReplicaConfiguration =
          new RedisStaticMasterReplicaConfiguration(
              redisProperties.getHost(),
              redisProperties.getPort()
          );
        
      connectionFactory = new LettuceConnectionFactory(staticMasterReplicaConfiguration, clientConfig);

      return connectionFactory;
	}

	@Bean
	@Primary
	public RedisCacheManager isLettuceCacheManager() {
      RedisCacheConfiguration redisCacheConfiguration = RedisCacheConfiguration
          .defaultCacheConfig()
          .entryTtl(Duration.ofSeconds(60))
          .serializeKeysWith(
            RedisSerializationContext.SerializationPair.fromSerializer(
                new StringRedisSerializer()))
          .serializeValuesWith(
              RedisSerializationContext.SerializationPair.fromSerializer(new GenericJackson2JsonRedisSerializer()));

      return RedisCacheManager.RedisCacheManagerBuilder
          .fromConnectionFactory(redisConnectionFactory())
          .cacheDefaults(redisCacheConfiguration)
          .build();
	}

}
```

3. yaml 을 작성 (RedisProperties를 통해서 properties에 저장한 host, port를 가지고 와서 연결한다.)
```
...

spring:
  redis:
    host: localhost
    port: 6379

...
```

4. 테스트용 컨트롤러 작성   
[RedisController.java]    
```
package com.example.bkjeon.base.services.api.v1.cache;

import java.util.List;

import org.springframework.boot.autoconfigure.EnableAutoConfiguration;
import org.springframework.boot.autoconfigure.data.redis.RedisAutoConfiguration;
import org.springframework.boot.autoconfigure.data.redis.RedisReactiveAutoConfiguration;
import org.springframework.web.bind.annotation.GetMapping;
import org.springframework.web.bind.annotation.RequestMapping;
import org.springframework.web.bind.annotation.RequestParam;
import org.springframework.web.bind.annotation.RestController;

import com.example.bkjeon.entity.cache.CacheExampleData;

import io.swagger.annotations.ApiOperation;
import io.swagger.annotations.ApiParam;
import lombok.RequiredArgsConstructor;

@RestController
@RequiredArgsConstructor
@RequestMapping("v1/redis")
public class RedisController {

    private final CacheService cacheService;

    @ApiOperation("Cache Example Data List")
    @GetMapping("examples/cache")
    public List<CacheExampleData> getRedisExampleList(
        @ApiParam(
            value = "bkjeon: bkjeon관련 데이터, example:example 관련 데이터",
            name = "exampleType",
            required = true
        ) @RequestParam String exampleType
    ) {
        return cacheService.getRedisExampleList(exampleType);
    }

}
```     

5. 테스트용 서비스 작성    
[CacheService.java]
```
...

@Cacheable(value = "foo", key = "#exampleType", cacheManager = "isLettuceCacheManager")
public List<CacheExampleData> getRedisExampleList(String exampleType) {
    List<CacheExampleData> exampleList = new ArrayList<>();

    for (int i=1; i<7; i++) {
        CacheExampleData cacheExampleData = CacheExampleData.builder()
            .exampleNo(i)
            .writer(exampleType)
            .build();
        exampleList.add(cacheExampleData);
    }

    return exampleList;
}

...
```

6. Entity 클래스 작성
```
package com.example.bkjeon.entity.cache;

import lombok.AllArgsConstructor;
import lombok.Builder;
import lombok.Getter;
import lombok.NoArgsConstructor;

@Builder
@Getter
@NoArgsConstructor
@AllArgsConstructor
public class CacheExampleData {

    private Integer exampleNo;
    private String writer;

}
```

7. 테스트    
```
$ redis-cli -h localhost -p 6379 monitor | grep foo
```
![java-redis-lettuce-1](/img/posts/language/java/redis/java-redis-lettuce-1.png)       


## 참고
- https://jojoldu.tistory.com
- https://lettuce.io/core/release/reference/