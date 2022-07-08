---
layout: post
title : "Circuitbreaker-Resilience4j-Java 1편"
subtitle : "2022-07-04-circuitbreaker-resilience4j-java-2.md"
date: 2022-07-04 20:00:00
author: "전봉근"
header-img: "img/posts/android-bg.jpg"
comments: true
tags: [java, msa, springboot, circuitbreaker]
---


# 서킷브레이커(=Circuitbreaker) Resilience4j 적용 (Java + Spring Boot) 1편


## 서킷브레이커란
`Fault Tolerance(=장애 허용 시스템) 에서 사용되는 대표적인 패턴`으로써 서비스에서 타 서비스 호출 시 에러, 응답지연, 무응답, 일시적인 네트워크 문제 등을 요청이 무작위로 실패하는 경우에 Circuit를 오픈하여 메세지가 다른 서비스로 전파되지 못하도록 막고 미리 정의해놓은 Fallback Response를 보내어 서비스 장애가 전파되지 않도록 하는 패턴 (대표적으로 MSA 환경에서 사용)

- 상태가 정상
  - Client -> Service A -> Circuit Breaker (상태: 정상이므로 Bypass Traffic) -> Service B
- 상태가 장애상황
  - Client -> Service A <-> Circuit Breaker (상태: 장애상황이므로 Fallback Message 처리) | Service B (Circuit Breaker 에 의해서 Service B 까지 요청이 도달하지 않음)


## 서킷브레이커 종류
- Netflix Hystrix
  - 넷플릭스에서 만든 라이브러리로 MSA 환경에서 분산된 서비스간 통신이 원활하지 않은 경우에 각 서비스가 장애 내성과 지연 내성을 갖게하도록 도와주는 라이브러리
  - `현재는 지원종료 상태이며 Resilience4j 가 권장되는 상태`
    - [Resilience4j 권고](https://github.com/Netflix/Hystrix)
    - Spring Boot 2.4.X 부터는 더 이상 지원하지 않음 [[Link]](https://spring.io/blog/2018/12/12/spring-cloud-greenwich-rc1-available-now)
- `Resilience4j`
  - Resilience4j 는 `Netflix Hystrix` 로 부터 영감을 받은 `Fault Tolerance Libray`
  - 사용하기 가볍고 `다른 라이브러리 의존성이 없음`
  - Java 전용으로 개발된 경량화된 Fault Tolerance Libray
  

## 서킷브레이커 상태
![tool-vscode-1](/img/posts/msa/msa-circuitbreaker-java-1.png)
이미지 출처: https://jupiny.com/
- `CLOSE`: 초기 상태이며 모든 접속은 평소와 같이 실행된다.
- `OPEN`: 에러율이 임계치를 넘어서면 `OPEN` 상태가 되며 모든 접속은 차단(fail fast) 된다. (실제 요청을 날리지 않고 바로 에러를 발생시킴)
- `HALF_OPEN`: `OPEN` 상태 중간에 한번씩 요청을 날려 응답이 성공인지를 확인하는 상태이며 `OPEN` 후 일정 시간이 지나면 `HALF_OPEN` 상태가 된다. 접속을 시도하여 성공하면 `CLOSE`, 실패하면 `OPEN`으로 되돌아감


## Resilience4j 모듈 종류
- [Retry](https://resilience4j.readme.io/docs/retry#create-and-configure-retry)
  - 요청이 실패하였을 때, 재 시도하는 기능 제공
- [Circuit Breaker](https://resilience4j.readme.io/docs/circuitbreaker#create-and-configure-a-circuitbreaker)
  - 장애 감지 시 예상치 못한 시스템 장애가 지속적으로 반복되는 것을 방지
- [Bulkhead](https://resilience4j.readme.io/docs/bulkhead#create-and-configure-a-threadpoolbulkhead)
  - 동시 실행 횟수 제한 기능 제공
- [RateLimiter](https://resilience4j.readme.io/docs/ratelimiter#create-and-configure-a-ratelimiter)
  - 제한치를 넘어선 요청에 대한 요청을 거부하거나 Queue로 만들어 실행 기능 제공
- [TimeLimiter](https://resilience4j.readme.io/docs/timeout)
  - 실행 시간 제한 설정 기능 제공
- [Cache](https://resilience4j.readme.io/docs/cache)
  - 결과 캐싱 기능 제공

> 모듈 우선순위 (Retry 는 마지막에 적용, 그러나 우선순위는 변경이 가능하다.) <br /> Retry ( CircuitBreaker ( Function ) ) <br /> Retry ( CircuitBreaker ( RateLimiter ( TimeLimiter ( Bulkhead ( Function ) ) ) ) )

- 우선순위 변경방법 (Ex: Retry 처리가 끝난 후에 서킷브레이커를 시작하려면 "retryAspectOrder" 프로퍼티를 circuitBreakerAspectOrder 값보다 큰값으로 설정해야한다. 즉, `순번이 빠를수록 더 빨리 실행됨`)  
  [application.yml]
  ```
  ...
  resilience4j:
    circuitbreaker:
      circuitBreakerAspectOrder: 1
    retry:
      retryAspectOrder: 2  
  ...
  ```

> 여기서 우리는 Retry 와 Circuit breaker 를 적용해보자.

- 각 모듈에 대한 application.yml 에 설정할 프로퍼티 설명
  - Retry
    - `maxRetryAttempts`: 최대 재시도 수(최초 호출도 포함, 기본값 3)
    - `waitDuration`: 재시도 할 때마다 기다리는 고정시간 (1초[1000ms], 기본값: 0.5초[500ms])
    - intervalFunction
      - dynamic한 waitDuration을 만들고자 할 때 사용, input 시도 횟수, output waitDuration (1.7.x 버전에서 현재 bug로 남아있음, 기본값: 없음)
      - properties로 정할 수 없고 RetryConfigCustomizer를 이용해야 한다.
    - intervalBiFunction
      - dynamic한 waitDuration을 만들고자 할 때 사용, input 시도 횟수, output waitDuration (1.7.x 에서 버그가 발생하므로 1.6.x 버전으로 사용필요([Link](https://github.com/resilience4j/resilience4j/issues/1404)), 기본값: numOfAttempts -> waitDuration)
      - properties로 정할 수 없고 RetryConfigCustomizer를 이용해야 한다.
    - retryOnResultPredicate: 반환되는 결과에 따라서 retry를 할지 말지 결정하는 filter, true로 반환하면 retry하고 false로 반환하면 retry 하지 않습니다. (기본값: (numOfAttempts,Either<throwable, result) -> waitDuration)
    - retryExceptionPredicate: 예외(Exception)에 따라 재시도 여부를를 결정하기 위한 filter, 만약 예외에 따라 재시도해야 한다면 true를, 그 외엔 false를 리턴해야 한다. (기본값: result -> false)
    - retryExceptions: 실패로 기록되는 에러 클래스 리스트 즉, 재시도 되야하는 에러 클래스의 리스트이다. empty일 경우 모든 에러 클래스를 재시도 한다. (기본값: empty)
    - ignoreExceptions: 무시(=ignore)되야 하는 에러 클래스 리스트 즉, 재시도 되지 않아야 할 에러 클래스 리스트이다. (기본값: empty)
    - failAfterMaxRetries: 설정한 maxAttempts 만틈 재시도하고 나서도 결과가 여전히 retryOnResultPredicate를 통과하지 못했을 때 MaxRetriesExceededException  발생을 활성화/비활성화하는 boolean (기본값: false)
  - Circuit Breaker
    - `failureRateThreshold`: 실패비율 임계치를 백분율로 설정 해당 값을 넘어갈 시 Circuit Breaker 는 Open상태로 전환되며, 이때부터 호출을 차단한다 (기본값: 50)
    - `slowCallRateThreshold`: 임계값을 백분율로 설정, CircuitBreaker는 호출에 걸리는 시간이 slowCallDurationThreshold보다 길면 느린 호출로 간주, 해당 값을 넘어갈 시 Circuit Breaker 는 Open상태로 전환되며, 이때부터 호출을 차단한다 (기본값: 100)
    - `slowCallDurationThreshold`: 호출에 소요되는 시간이 설정한 임계치보다 길면 느린 호출로 계산한다. -> 응답시간이 느린것으로 판단할 기준 시간 (60초, 1000 ms = 1 sec) (기본값: 60000[ms])
    - permittedNumberOfCallsInHalfOpenState: HALF_OPEN 상태일 때, OPEN/CLOSE 여부를 판단하기 위해 허용할 호출 횟수를 설정 수 (기본값: 10)
    - maxWaitDurationInHalfOpenState: HALF_OPEN 상태로 있을 수 있는 최대 시간이다. 0일 때 허용 횟수 만큼 호출을 모두 완료할 때까지 HALF_OEPN 상태로 무한정 기다린다. (기본값: 0)
    - slidingWindowType: sliding window 타입을 결정한다. COUNT_BASED인 경우 slidingWindowSize만큼의 마지막 call들이 기록되고 집계됩니다. TIME_BASED인 경우 마지막 slidingWindowSize초 동안의 call들이 기록되고 집계됩니다.
    - slidingWindowSize: CLOSED 상태에서 집계되는 슬라이딩 윈도우 크기를 설정한다. (기본값: 100)
    - minimumNumberOfCalls: minimumNumbeerOfCalls 이상의 요청이 있을 때부터 faiure/slowCall rate를 계산한다. 예를들어, 해당값이 10이라면 최소한 호출을 10번을 기록해야 실패 비율을 계산할 수 있다. 기록한 호출 횟수가 9번뿐이라면 9번 모두 실패했더라도 circuitbreaker는 열리지 않는다. (기본값: 100)
    - waitDurationInOpenState: OPEN에서 HALF_OPEN 상태로 전환하기 전 기다리는 시간 (60초, 1000 ms = 1 sec) (기본값: 60000[ms])
    - automaticTransitionFromOpenToHalfOpenEnabled: true 라면 waitDurationInOpenState 기간이 지난 이후에 Open에서 Half-open으로 자동으로 상태가 변경된다. 하나의 쓰레드가 CircuitBreaker의 모든 인스턴스들을 모니터링하며 상태를 확인한다. false라면 요청이 있을 때만 상태가 Half-open으로 변경된다. 즉, waitDurationInOpenState 기간이 지난 후에 새로운 요청이 있으면 Half-open 상태로 변경된다. 모니터링을 위한 쓰레드가 필요없는 이점이 있다. (기본값: false)
    - recordExceptions: 실패로 기록할 Exception 리스트 (기본값: empty)
    - ignoreExceptions: 실패나 성공으로 기록하지 않을 Exception 리스트 (기본값: empty)
    - recordException: 실패로 기록할 Exception을 판단하는 Predicate<Throwable>을 설정 By default all exceptions are recored as failures. (커스터마이징, 기본값: throwable -> true)
    - ignoreException: 기록하지 않을 Exception을 판단하는 Predicate<Throwable>을 설정 (커스터마이징, 기본값: throwable -> true)
    - recordFailure: 어떠한 경우에 Failure Count를 증가시킬지 Predicate를 정의해 CircuitBreaker에 대한 Exception Handler를 재정의하는 것이다. true를 return할 경우, failure count를 증가시키게 된다 (기본값: false)


## 시작하기 (Spring Boot에 Resilience4j(Retry, Circuit Breaker) 적용)
resilience4j는 다양한 프레임워크에서 돌아갈 수 있는 모듈이다. 그러므로 Spring Boot에서 지원하는 전용 모듈을 추가해야 한다. [참고링크](https://resilience4j.readme.io/docs/getting-started-3)

- 의존성 추가  
  [bulid.gradle]
  ```
  ...
  dependencies {
      // resilience4j (actuator, aop 는 필수로 같이 선언되어야함)
      implementation 'org.springframework.boot:spring-boot-starter-actuator'
      implementation 'org.springframework.boot:spring-boot-starter-aop'
      implementation 'io.github.resilience4j:resilience4j-spring-boot2:1.7.0'

      // Retry, Circuit Breaker 관련 의존성 추가
      implementation 'io.github.resilience4j:resilience4j-retry:1.7.0'
      implementation 'io.github.resilience4j:resilience4j-circuitbreaker:1.7.0'      
  }
  ...
  ```
- application.yml에 resilience4j의 Retry, Circuit Breaker 적용 (설정값은 케이스에 맞게 수정 필요)   
  [application.yml]  
  ```
  ...
  resilience4j:
    retry:
      configs:
        default:
          maxRetryAttempts: 5 # 최대 재시도 수
          waitDuration: 5000  # 재시도 사이에 고정된 시간 [ms]
          #retryExceptions:
          #  - org.springframework.web.client.HttpServerErrorException
          #  - java.io.IOException
          #ignoreExceptions:
          #  - java.util.NoSuchElementException
      instances:
        retry-test-3000: # retry name
          baseConfig: default # 기본 config 지정 (Ex-retry.configs.{default})
          waitDuration: 3000
        retry-db-select-4000:
          baseConfig: default
          waitDuration: 4000
        retry-db-select-5000:
          baseConfig: default
          waitDuration: 5000
    circuitbreaker:
      configs:
        default:  # 기본 config 명
          registerHealthIndicator: true
          slidingWindowType: TIME_BASED
          slidingWindowSize: 10
          minimumNumberOfCalls: 10  # 최소한 호출을 10번을 기록해야 실패 비율을 계산할 수 있다.
          slowCallRateThreshold: 100
          slowCallDurationThreshold: 60000
          failureRateThreshold: 50
          permittedNumberOfCallsInHalfOpenState: 10
          waitDurationInOpenState: 10s  # 서킷의 상태가 Open 에서 Half-open 으로 변경되기전에 Circuit Break가 기다리는 시간 [s]
      instances:
        circuit-test-70000: # circuitbreaker name
          baseConfig: default # 기본 config 지정 (Ex-circuitbreaker.configs.{default})
          slowCallDurationThreshold: 70000 # 응답시간이 느린것으로 판단할 기준 시간 [ms]
        circuit-db-select-200:
          baseConfig: default
          slowCallDurationThreshold: 200
        circuit-db-select-300:
          baseConfig: default
          slowCallDurationThreshold: 300
  ```
- 모듈별 애노테이션 설정의 name 값에 일치시킬 상수선언   
  [Resilience4jCode.java]   
  ```
  // Resilience4j 상수 선언 (서킷브레이커 관련 모듈 네임 매핑 용도)
  public final class Resilience4jCode {

      public static final String RETRY_TEST_3000 = "retry-test-3000";
      public static final String RETRY_DB_SELECT_4000 = "retry-db-select-4000";
      public static final String RETRY_DB_SELECT_5000 = "retry-db-select-5000";
      public static final String CIRCUIT_TEST_70000 = "circuit-test-70000";
      public static final String CIRCUIT_DB_SELECT_200 = "circuit-db-select-200";
      public static final String CIRCUIT_DB_SELECT_300 = "circuit-db-select-300";

  }  
  ```
- @CircuitBreaker 설정
  - 호출 Method 에 name(yml에 선언한 값 -> 상수(=Resilience4jCode)), fallbackMethod: fallbackMethod method명을 지정    
    [CircuitBreakerController.java]   
    ```
    import org.springframework.web.bind.annotation.GetMapping;
    import org.springframework.web.bind.annotation.RestController;

    import lombok.RequiredArgsConstructor;

    @RestController
    @RequiredArgsConstructor
    public class CircuitBreakerController {

        private final CircuitBreakerService circuitBreakerService;

        @GetMapping("/hello")
        public String hello() {
          return circuitBreakerService.getHello();
        }

    }    
    ```
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
          throw new RuntimeException("failed");
        }
        private String getCircuitBreakerFallback(Throwable t) {
          return "getCircuitBreakerFallback! exception type: " + t.getClass() + "exception, message: " + t.getMessage();
        }

    }
    ```  
  - 결과  
    ```
    // waitDurationInOpenState: 10s 로 설정되어있어 10초동안 Circuit Break가 기다린다
    // minimumNumberOfCalls: 10 으로 설정되어 있으므로 최소한 호출을 10번을 기록해야 실패 비율을 계산할 수 있다. 서킷브레이커가 발동시 하기와 같은 메세지를 볼 수 있다 -> ex.getMessage (즉, getBoardListFallBack 메소드가 실행됨을 의미)
    getHelloFallback! exception type: class io.github.resilience4j.circuitbreaker.CallNotPermittedExceptionexception message: CircuitBreaker 'circuit-test-70000' is OPEN and does not permit further calls
    ```
- @Retry 설정
  - 호출 Method 에 name(yml에 선언한 값 -> 상수(=Resilience4jCode)), fallbackMethod: fallbackMethod method명을 지정    
    [CircuitBreakerController.java]   
    ```
    import org.springframework.web.bind.annotation.GetMapping;
    import org.springframework.web.bind.annotation.RequestMapping;
    import org.springframework.web.bind.annotation.RestController;

    import lombok.RequiredArgsConstructor;

    @RestController
    @RequestMapping("v1/circuitBreaker")
    @RequiredArgsConstructor
    public class CircuitBreakerController {

        private final CircuitBreakerService circuitBreakerService;

        @GetMapping("retryFail")
        public String retryFail() {
          return circuitBreakerService.getRetry();
        }

    }    
    ```
    [CircuitBreakerService.java]   
    ```  
    import org.springframework.stereotype.Service;

    import com.example.bkjeon.constants.Resilience4jCode;

    import io.github.resilience4j.circuitbreaker.annotation.CircuitBreaker;
    import io.github.resilience4j.retry.annotation.Retry;
    import lombok.extern.slf4j.Slf4j;

    @Slf4j
    @Service
    public class CircuitBreakerService {

        private void runtimeException() {
           throw new RuntimeException("failed");
        }

        @Retry(name = Resilience4jCode.RETRY_TEST_3000, fallbackMethod = "getRetryFallback")
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
  - 결과  
    ```
    // maxRetryAttempts: 5 이므로 실패시 최대 재시도 수 5번까지 한 후 Fallback 메소드 실행
    // waitDuration: 5000 이므로 5초에 한 번씩 재요청
    getRetryFallback! exception type: class java.lang.RuntimeExceptionexception, message: failed
    ```    

> 여기까지 적용하면 완료이다. 상황에 맞게 잘 활용하면 된다.

> 해당 적용된 부분을 모니터링을 하고 싶으면 다음 포스팅인 [서킷브레이커(=Circuitbreaker) Resilience4j 적용 (Java + Spring Boot) 2편](https://bkjeon1614.tistory.com/712) 을 참고하자.