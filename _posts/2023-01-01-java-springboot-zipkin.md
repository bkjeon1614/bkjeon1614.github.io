---
layout: post
title: "SpringBoot + Zipkin 을 활용한 트레이스 환경 구성"
subtitle: "2023-01-01-java-springboot-zipkin.md"
date: 2023-01-01 18:40:00
author: "전봉근"
header-img: "img/posts/android-bg.jpg"
comments: true
tags: [java, spring, springboot, zipkin]
---

# SpringBoot + Zipkin 을 활용한 트레이스 환경 구성
## MSA 환경과 OpenTracing
모놀리식 아키텍쳐의 경우 하나의 서버가 서비스의 전반적이 기능을 모두 제공하므로 클라이언트의 요청을 받으면 하나의 스레드에서 모든 요청을 실행하여 로그를 확인하기 쉽다. 반면에 MSA 는 여러개의 마이크로 서비스 간에 통신이 발생하기 때문에 로그를 확인하기 어렵다.    
     
이와 같은 문제를 해결하기 위한 방법으로는 [OpenTracing](https://medium.com/opentracing/towards-turnkey-distributed-tracing-5f4297d1736) 이 알려져 있다. `OpenTracing` 는 애플리케이션 간 분산 추적을 위한 표준이라고 정의할 수 있으며, 이 표준의 대표적인 구현체로 [Jaeger](https://www.jaegertracing.io/) 와 [Zipin](https://zipkin.io/) 이 있다. (다른 방법으로는 APM (Ex: Jennifer, Pinpoint 등..) 도구를 사용한 방법이 있지만 상세한 추적이 어렵다고 한다)     
      
그러므로 Java 와 Spring 프레임워크 환경에서 쉽게 연동할 수 있는 `Zipkin` 으로 선택한 내용을 보고 정리하기로 하였다.    


## Zipkin
Zipkin 이란 분산 환경에서 로그 트레이싱하는 오픈소스로 트위터에서 개발되었으며, 현재 가장 활성화된 오픈소스라고 한다. (플러그인 또는 부가적인 도구가 많음). Zipkin 으로 추적할 수 있는 분산 트랜잭션은 HTTP, gRPC 가 있다.       
       
Zipkin 의 아키텍쳐는 아래와 같다.      
![springboot-zipkin-1](/img/posts/language/java/zipkin/springboot-zipkin-1.PNG)        
1. `Reporter` 가 Transport 를 통해서 Collector 에 트레이스 정보를 전달한다.
   - Reporter
     - 각 서버는 계측(instrumented) 라이브러리를 사용해야 Reporter 로서 동작할 수 있다고 한다. Zipkin 에서는 다양한 언어에 대한 라이브러리를 [제공](https://zipkin.io/pages/tracers_instrumentation.html)하며 Java 환경이므로 [Brave](https://github.com/openzipkin/brave) 를 사용할 수 있으며 또한 Spring 프레임워크에서는 [Spring Cloud Sleuth](https://github.com/spring-cloud/spring-cloud-sleuth) 의 BraveTracer 를 통하여 트레이스 데이터를 관리하기 위한 기능들을 제공하고 있어 쉽게 적용할 수 있다.     
2. 전달된 트레이스 정보는 Database 에 저장된다.
   - Database
     - Zipkin 에서 몇 가지 [Storage](https://github.com/openzipkin/zipkin#storage-component) 를 지원하며, 그 중 ElasticSearch 를 선택하여 진행하기로 하였다.     
3. Zipkin UI 에서 API 를 통해 해당 정보를 시각화해서 제공한다. (하기 명령으로 간단하게 구성할 수 있다.)     
   ```
   $ curl -sSL https://zipkin.io/quickstart.sh | bash -s
   $ java -jar zipkin.jar --STORAGE_TYPE=elasticsearch --ES_HOSTS=http://127.0.0.1:9200
   ```
   ![springboot-zipkin-2](/img/posts/language/java/zipkin/springboot-zipkin-2.PNG)        


## Spring Cloud Sleuth
Spring Boot 환경에서 `Spring Cloud Sleuth` 를 사용해보자.     
Zipkin 은 [B3-Propagation](https://github.com/openzipkin/b3-propagation) 을 통하여 OpenTracing 을 구현하고 있다.   Client 에서 HTTP Header 를 통하여 다른 서버로 전달하고, 예를 들어 Kafka 메세지를 통해 전달하는 경우에는 Kafka 헤더를 통해서 전달하면 Header 의 `X-B3-` 으로 시작하는 X-B3-TraceId, X-B3-ParentSpanId, X-B3-SpanId, X-B3-Sampled 총 4개 값을 전달하여 트레이스 정보를 관리한다.
      
### Spring Cloud Sleuth 설정 
### 의존성 추가
build.gradle 에 의존성 추가 (각 서비스에 추가)    
```
...
dependencies {
    ...
    implementation group: 'org.springframework.cloud', name: 'spring-cloud-starter-zipkin', version: '2.2.8.RELEASE'    
    implementation group: 'org.springframework.cloud', name: 'spring-cloud-starter-sleuth', version: '3.1.5'
}
```     
       
application.yml 에 zipkin 과 sleuth 설정 추가 (각 서비스에 추가)      
```
spring:
  application:
    name: [Zipkin UI 에서 확인할 수 있는 서비스명]
  sleuth:
    sampler:
      probability: 1.0  # Zipkin 에 트랜잭션을 어느정도 비율로 보낼지에 대한 설정 값이며 기본값은 10%(0.1) 이며, 1.0 이면 트랜잭션을 100% 보내게 된다.
    zipkin:
      base-url: http://[Zipkin 실행 호스트 ip]:9411
```

### 테스트


# 참고
- https://zipkin.io/
- https://engineering.linecorp.com/