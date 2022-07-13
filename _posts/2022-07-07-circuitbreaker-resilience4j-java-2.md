---
layout: post
title : "Circuitbreaker-Resilience4j-Java 2편"
subtitle : "2022-07-04-circuitbreaker-resilience4j-java-2.md"
date: 2022-07-04 20:00:00
author: "전봉근"
header-img: "img/posts/android-bg.jpg"
comments: true
tags: [java, msa, springboot, circuitbreaker]
---


# 서킷브레이커(=Circuitbreaker) Resilience4j 적용 (Java + Spring Boot) 2편 
해당 포스팅은 1편인 이전 포스팅인 [https://bkjeon1614.tistory.com/711](https://bkjeon1614.tistory.com/711)을 참고하여 사전작업 후 진행하는것이 좋다. (모니터링 하는 방법에 대해서만 설명이 나오기 때문)


## 서킷브레이커 테스트를 위한 코드수정
먼저 모니터링을 위하여 [https://bkjeon1614.tistory.com/711](https://bkjeon1614.tistory.com/711) 의 `CircuitBreakerService.java` 코드를 일부 변경하자.   
[CircuitBreakerService.java]   
```
import java.util.Random;
import org.springframework.stereotype.Service;
import com.example.bkjeon.constants.Resilience4jCode;
import io.github.resilience4j.circuitbreaker.annotation.CircuitBreaker;
import lombok.extern.slf4j.Slf4j;

@Slf4j
@Service
public class CircuitBreakerService {

    @CircuitBreaker(name = Resilience4jCode.CIRCUIT_TEST_70000, fallbackMethod = "getCircuitBreakerFallback")
    public String getCircuitBreaker() {
        runtimeException();
        return "hello world!";
    }

    private void runtimeException() {
        // as-is
        // throw new RuntimeException("failed");

        // to-be
		int randomInt = new Random().nextInt(10);

		if(randomInt <= 7) {
			throw new RuntimeException("failed");
		}        
    }
    private String getCircuitBreakerFallback(Throwable t) {
        return "getCircuitBreakerFallback! exception type: " + t.getClass() + "exception, message: " + t.getMessage();
    }

	@Retry(name = Resilience4jCode.RETRY_TEST_3000 , fallbackMethod = "getRetryFallback")
	public String getRetry() {
		log.info("=============== getRetry Request !!");
		runtimeException();
		return "hello world!";
	}
	private String getRetryFallback(Throwable t) {
		return "getRetryFallback! exception type: " + t.getClass() + "exception, message: " + t.getMessage();
	}    

}
```


## 모니터링을 위한 설정파일 수정
- application.yml 수정    
  [application.yml]   
  ```
  ...
  resilience4j:
    resilience4j:
      retry:
        configs:
        default:
          registerHealthIndicator: true # actuator 정보 노출을 위한 설정  
    ...
    circuitbreaker:
      configs:
        default:
          registerHealthIndicator: true # actuator 정보 노출을 위한 설정
  ...
  # actuator
  management:
    endpoint:
      health:
        show-details: always
    endpoints:
      web:
        exposure:
          include:
            - "*" # 테스트를 위해 actuator 전체 노출
    health:
      circuitbreakers:
        enabled: true # circuitbreakers 정보 노출
      retryevents:
        enabled: true # retryevents 정보 노출  
  ```


## Actuator 를 통한 모니터링
- {domain}/actuator/health 을 통하여 상태를 확인할 수 있다. (서킷의 status or state 로 상태를 확인할 수 있다. 자세한 로그를 보려면 하기 내용을 참고하자.)   
  ![msa-circuitbreaker-actuator-1](/img/posts/msa/msa-circuitbreaker-actuator-1.png)
- {domain}/actuator 다양한 모니터링 endpoint 목록 확인    
  ![msa-circuitbreaker-actuator-2](/img/posts/msa/msa-circuitbreaker-actuator-2.png)
- 각 모듈의 모니터링 로그 확인 -> 상세 로그 확인 시 (Ex - Retry, Circuitbreaker)
  - 1편에서 작성한 `@Retry` 관련 API를 호출할 때 모니터링 ({domain}/actuator/retryevents)   
    - @Retry 가 실행 되지 않을 때 (초기상태)   
      ![msa-circuitbreaker-actuator-3](/img/posts/msa/msa-circuitbreaker-actuator-3.png)
    - 설정해놓은 임계치를 넘은 재시도에 의해서 @Retry 실행 되었을 때 (type: ERROR 인 객체가 표시된다.)    
      ![msa-circuitbreaker-actuator-4](/img/posts/msa/msa-circuitbreaker-actuator-4.png)
    - 설정해놓은 임계치 이전에 성공을 하였을때 (type: SUCCESS 인 객체가 표시된다. 성공하였으므로 `numberOfAttempts`는 증가하지 않음)   
      ![msa-circuitbreaker-actuator-5](/img/posts/msa/msa-circuitbreaker-actuator-5.png)   
  - 1편에서 작성한 `@Circuitbreaker` 관련 API를 호출할 때 모니터링 ({domain}/actuator/circuitbreakerevents)   
    - @Circuitbreaker 가 실행되지 않을 때 (초기상태)   
      ![msa-circuitbreaker-actuator-6](/img/posts/msa/msa-circuitbreaker-actuator-6.png)
    - 설정해놓은 임계치를 넘은 실패에 의해서 @Circuitbreaker 실행되어 Circuit이 OPEN 되었을 때   
      ![msa-circuitbreaker-actuator-7](/img/posts/msa/msa-circuitbreaker-actuator-7.png)
    - Circuit이 HALF_OPEN 으로 되었을 때    
      ![msa-circuitbreaker-actuator-8](/img/posts/msa/msa-circuitbreaker-actuator-8.png)   


> 해당 적용된 부분을 Prometheus + Grafana 모니터링을 적용하고 싶으면 다음 포스팅인 [서킷브레이커(=Circuitbreaker) Resilience4j 적용 (Java + Spring Boot) 3편](https://bkjeon1614.tistory.com/714) 을 참고하자.