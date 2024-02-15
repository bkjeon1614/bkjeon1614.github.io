---
layout: post
title: "SpringBoot 에서 FeignClient 를 사용하여 Rest API 를 간편하게 사용하자"
subtitle: "2022-12-17-java-springboot-feignclient.md"
date: 2022-12-17 18:30:00
author: "전봉근"
header-img: "img/posts/android-bg.jpg"
comments: true
tags: [java, spring, springboot]
---

# Feign 이란
`Feign` 이란 Netflix 에서 개발된 오픈소스이며 선언적 방식으로 REST 기반 호출을 추상화해서 제공한다. 즉, interface 와 annotaion 만으로 간편하게 HTTP API Client 를 구현할 수 있으며 RestTemplate 을 만들 필요없이 사용할 수 있어 코드의 복잡성이 낮아진다.     
   

# 샘플코드는 하기 링크로 확인
[샘플코드](https://github.com/bkjeon1614/java-example-code/tree/develop/bkjeon-mybatis-codebase)   


# 사용법
## Dependency    
- maven
[pom.xml]   
```
 <dependencyManagement>
     <dependencies>
         <dependency>
            <groupId>org.springframework.cloud</groupId>
            <artifactId>spring-cloud-dependencies</artifactId>
            <version>${spring-cloud.version}</version>
            <type>pom</type>
            <scope>import</scope>
        </dependency>
    </dependencies>
</dependencyManagement>

...

<dependency>
    <groupId>org.springframework.cloud</groupId>
    <artifactId>spring-cloud-starter-openfeign</artifactId>
</dependency>
```

- gradle      
[build.gradle]   
```
...
// https://spring.io/projects/spring-cloud 에서 Spring Boot 버전에 맞게 세팅한다. (해당 포스팅은 Spring Boot 2.2.2.RELEASE 로 진행)
ext {
    set('springCloudVersion', "Hoxton.SR3")
}

dependencyManagement {
    imports {
        mavenBom "org.springframework.cloud:spring-cloud-dependencies:${springCloudVersion}"
    }
}

dependencies {
  ...
  implementation 'org.springframework.cloud:spring-cloud-starter-openfeign'
}
```

## SpringApplication.java 에 @EnableFeignClients 을 선언
[ApiApplication.java]    
```
@EnableFeignClients
@SpringBootApplication
...
public class ApiApplication {

    public static void main(String[] args) {
        SpringApplication.run(ApiApplication.class, args);
    }

}
```

## Feign 사용
먼저 패키지를 생성하자. 패키지는 ApiApplication.java 와 같은 레벨인 Root Package 에 존재하여야 하며, 그렇지 않으면 basePackages 또는 basePackageClasses 를 지정해야 한다. feign 이라는 패키지를 생성하고 그 안에 user 패키지 생성 후 또 그 안에 FeignUserClient.java 라는 클래스를 생성하자. (혹시 수동으로 구현체를 생성하고 싶으면 [수동 구현체 생성](https://cloud.spring.io/spring-cloud-netflix/multi/multi_spring-cloud-feign.html#_creating_feign_clients_manually) 링크를 참고)     
[../feign/user/FeignUserClient.java]     
```
package com.example.bkjeon.base.feign.user;

import java.util.List;

import org.springframework.cloud.openfeign.FeignClient;
import org.springframework.cloud.openfeign.SpringQueryMap;
import org.springframework.web.bind.annotation.GetMapping;
import org.springframework.web.bind.annotation.PathVariable;
import org.springframework.web.bind.annotation.RequestParam;

import com.example.bkjeon.base.feign.user.model.FeignUserRequest;

@FeignClient(
    configuration = FeignUserClientConfig.class,
    fallbackFactory = FeignUserClientFallback.class,
    name = "example",
    url = "${external-api.url}"
)
public interface FeignUserClient {

    @GetMapping("users")
    List<FeignUser> getUserList(@SpringQueryMap FeignUserRequest feignUserRequest);

    @GetMapping("users/{userId}")
    FeignUser getUser(@PathVariable("userId") Integer userId, @RequestParam("testCode") String testCode);

}
```      
     
API 를 호출할 도메인 및 토큰 입력
[application.yml]   
```
...

external:
  api:
    user:
      url: https://jsonplaceholder.typicode.com
      token: eyJhbGciOiJIUzUxMiJ9.eyJzdWIiOiJ7XCJwb3NpdERlcHRJZFwiOjMsXCJkbW5DZFwiOlwiRUxUXCIsXCJhcGlrZXlJZFwiOlwiRUxUXzE1NTY1MjM
```     

결과값을 받을 클래스를 생성한다.     
[../feign/user/entity/FeignUser.java]  
```
package com.example.bkjeon.base.feign.user.entity;

import lombok.Getter;
import lombok.NoArgsConstructor;

@Getter
@NoArgsConstructor
public class FeignUser {

    private Integer id;
    private String name;
    private String username;
    private String email;
    private Address address;
    private String phone;
    private String website;
    private Company company;

    @Getter
    public static class Address {
        private String street;
        private String suite;
        private String city;
        private String zipcode;
        private Geo geo;

        @Getter
        public static class Geo {
            private String lat;
            private String lng;
        }
    }

    @Getter
    public static class Company {
        private String name;
        private String catchPhrase;
        private String bs;
    }

}
```     
     
요청시 필요한 파라미터 클래스를 생성한다.   
[../feign/user/FeignUserRequest.java]
```
package com.example.bkjeon.base.feign.user.model;

import lombok.Builder;
import lombok.ToString;

@ToString
public class FeignUserRequest {

	private String testCode;

	@Builder
	public FeignUserRequest(String testCode) {
		  this.testCode = testCode;
	}

}
```    
             
실패할 경우 예외처리할 클래스를 생성한다.   
[../feign/user/FeignUserClientFallback.java]    
```
package com.example.bkjeon.base.feign.user;

import java.util.List;

import org.springframework.stereotype.Component;

import com.example.bkjeon.base.feign.user.model.FeignUserRequest;

import feign.hystrix.FallbackFactory;
import lombok.extern.slf4j.Slf4j;

/**
 * Exception
 */
@Component
@Slf4j
public class FeignUserClientFallback implements FallbackFactory<FeignUserClient>  {

    @Override
    public FeignUserClient create(Throwable throwable) {
        return new FeignUserClient() {
            @Override
            public List<FeignUser> getUserList(FeignUserRequest feignUserRequest) {
                log.error("param: {}, error ==> {}", feignUserRequest.toString(), throwable.toString());
                return null;
            }

            @Override
            public FeignUser getUser(Integer userId, String testCode) {
                log.error("param: userId ==> {}, testCode ==> {}, error ==> {}", userId, testCode, throwable.toString());
                return new FeignUser();
            }
        };
    }

}
```     
         
API 요청시 헤더에 세팅할 인터셉터를 구현한다.     
[../feign/user/FeignClientInterceptor.java]    
```
package com.example.bkjeon.base.feign.user;

import com.google.common.base.Preconditions;

import feign.RequestInterceptor;
import feign.RequestTemplate;

/**
 * Header Setting Interceptor
 */
public class FeignClientInterceptor implements RequestInterceptor {

    private String apiToken;

    public FeignClientInterceptor(String apiToken) {
        Preconditions.checkNotNull(apiToken, "Api Token Should Not Be Null");
        this.apiToken = apiToken;
    }

    @Override
    public void apply(RequestTemplate template) {
        template.header("Authorization", "Bearer " + apiToken);
        template.header("Content-Type", "application/json");
    }

}
```     
             
Header 값 등 세팅할 Configuration 을 생성      
[../feign/user/FeignUserClientConfig.java]
```
package com.example.bkjeon.base.feign.user;

import org.springframework.beans.factory.annotation.Value;
import org.springframework.context.annotation.Bean;

import feign.RequestInterceptor;

/**
 * Feign Configuration
 */
public class FeignUserClientConfig {

    @Bean
    public RequestInterceptor requestHeader(@Value("${external-api.token}") String apiToken) {
        return new FeignUserClientInterceptor(apiToken);
    }

}
```      
        
## Feign 을 사용할 Controller 생성
[FeignExampleController.java]    
```
package com.example.bkjeon.base.services.api.v1.feign;

import java.util.List;

import org.springframework.web.bind.annotation.GetMapping;
import org.springframework.web.bind.annotation.PathVariable;
import org.springframework.web.bind.annotation.RequestMapping;
import org.springframework.web.bind.annotation.RestController;

import com.example.bkjeon.base.feign.user.FeignUserClient;
import com.example.bkjeon.base.feign.user.entity.FeignUser;
import com.example.bkjeon.base.feign.user.model.FeignUserRequest;

import io.swagger.annotations.ApiOperation;
import lombok.RequiredArgsConstructor;

@RestController
@RequiredArgsConstructor
@RequestMapping("v1/feign")
public class FeignExampleController {

    private final FeignUserClient feignUserClient;

    @ApiOperation("Feign 을 통한 User 목록 조회")
    @GetMapping("users")
    public List<FeignUser> getUserList() {
        FeignUserRequest feignUserRequest = FeignUserRequest.builder().testCode("bkjeon").build();
        return feignUserClient.getUserList(feignUserRequest);
    }

    @ApiOperation("Feign 을 통한 User 상세 조회")
    @GetMapping("users/{userId}")
    public FeignUser getUser(@PathVariable Integer userId) {
        return feignUserClient.getUser(userId, "bkjeon");
    }

}
```     

## 결과확인
```
http://localhost:9090/api/v1/feign/users
또는
http://localhost:9090/api/v1/feign/users/1
```

## Feign 패키지 구조
![springboot-feign-1](/img/posts/language/java/springboot-feign-1.png)     