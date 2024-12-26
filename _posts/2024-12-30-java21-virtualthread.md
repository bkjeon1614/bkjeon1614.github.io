---
layout: post
title: "Java 21 Virtual Thread"
subtitle: "2024-12-30-java21-virtualthread"
date: 2024-12-30 20:40:00
author: "전봉근"
header-img: "img/posts/android-bg.jpg"
comments: true
tags: [java, jvm]
---

## Java 21 Virtual Thread
해당 글은 [kakao tech meet 발표](https://www.youtube.com/watch?v=vQP6Rs-ywlQ) 을 참고하여 필요한 부분만 요약하여 정리하였다.


### Virtual Thread
JDK 21(LTS) 에 추가된 경량 스레드 OS 스레드를 그대로 사용하지 않고 JVM 내부 스케줄링을 통하여 수십 ~ 수백만개의 스레드를 동시에 사용할 수 있게 한다.


### 해결하고자 하는 문제
1. 애플리케이션의 높은 처리량(throughput) 확보
   - Blocking 발생시 내부 스케줄링을 통해 다른 작업을 처리
2. 자바 플랫폼 디자인과 조화를 이루는 코드 생성
   - 기존 스레드 구조 그대로 사용
     ```
     sealed abstract class BaseVirtualThread extends Thread
        permits VirtualThread, ThreadBuilders.BoundVirtualThread {
            ...
        }
     ```


### Virtual Thread 구조
![java-virtualthread-1](/img/posts/language/java/java-virtualthread-1.png)       
![java-virtualthread-2](/img/posts/language/java/java-virtualthread-2.png)       
- 기존 Platform Thread 와 비슷하다.
- Virtual Thread 가 어떤 작업을 할당 받아서 Carrier Thread 와 연결되어 있는 구조이다.
- 여기서 Blocking 이 발생한다면 Virtual Thread 는 Carrier Thread 에서 Unmount 되며 비어있는 Carrier Thread 에 다른 Virtual Thread 가 할당된다.


### 기존 Platform Thread 와 Virtual Thread 비교
![java-virtualthread-3](/img/posts/language/java/java-virtualthread-3.png)       


### 사용방법
- Java     
  ```
  // 방법 1
  Thread.startVirtualThread(() => {
    System.out.println("Hello Virtual Thread");
  });

  // 방법 2
  Runnable runnable = () => System.out.println("Hi Virtual Thread");
  Thread virtualThread1 = Thread.ofVirtual().start(runnable);

  // 방법 3
  Thread.Builder builder = Thread.ofVirtual().name("JVM-Thread");
  Thread virtualThread2 = builder.start(runnable);

  // 스레드가 Virtual Thread 인지 확인하여 출력
  System.out.println("Thread is Virtual? " + virtualThread2.isVirtual());

  // ExecutorService 사용
  try (final ExecutorService executorService = Executors.newVirtualThreadPerTaskExecutor()) {
      for (int i=0; i<3; i++) {
          executorService.submit(runnable);
      }
  }
  ```       
  ```
  # 결과
  Hello Virtual Thread
  Hi Virtual Thread
  Hi Virtual Thread
  Thread is Virtual? true
  Hi Virtual Thread
  Hi Virtual Thread
  Hi Virtual Thread
  ```
- Spring Boot 적용 (3.2 이상)   
  [application.yml]         
  ```
  # virtual enabled 를 true 로 지정하면 was 에 대한 처리를 virtual thread 가 감당하게끔 알아서 해준다.
  spring:
    threads:
      virtual:
        enabled: true
  ```
- 만약 Spring Boot 3.2 이상이 아닌 3.X 버전을 사용할 경우 직접 Bean 을 생성해야 함
  ```
  // Web Request 를 처리하는 Tomcat 이 Virtual Thread 를 사용하여 유입된 요청을 처리하도록 한다.
  @Bean
  public TomcatProtocolHandlerCustomizer<?> protocolHandlerVirtualThreadExecutorCustomizer() {
      return protocolHandler -> {
          protocolHandler.setExecutor(Executors.newVirtualThreadPerTaskExecutor());
      };
  }

  // Async Task 에 Virtual Thread 사용
  @Bean
  public AsyncTaskExecutor asyncTaskExecutor(TaskExecutionAutoConfiguration.APPLICATION_TASK_EXECUTOR_BEAN_NAME) {
      return new TaskExecutorAdapter(Executors.newVirtualThreadPerTaskExecutor());
  }
  ```


### 유의사항
- 단순히 기존 Platform Thread 의 코드를 Virtual Thread 의 코드로 바꾸기만 하면 성능상의 이점을 누릴 수 없으므로 개별 Task 를 Virtual Thread 로 할당한다는 개념으로 접근해야 한다.
- Thread Local 사용주의
  - Virtual Thread 는 수십 수백만까지도 생성이 가능하며 필요시마다 Heap 을 사용하기 때문에 이를 남발하면 메모리 사용이 늘어남
- synchronized 사용시 주의
  - synchronized 사용시 Virtual Thread 에 연결된 Carrier Thread 가 Blocking 될 수 있으니 주의 (이런 경우를 pinning 이라고 한다.)
    - 해결방법
      - ReentrantLock 을 사용하면 우회하게 된다. (순차적 접근 보장)
        ```
        // pinning 발생
        public synchronized String accessResource() {
            return access();
        }

        // ReentrantLock 사용 (pinning 발생하지 않음)
        private static final ReentrantLock LOCK = new ReentrantLock();

        public String accessResource() {
            LOCK.lock();
            try {
                return access();
            } finally {
                LOCK.unlock();
            }
        }
        ```    


### 성능/이슈
I/O Blocking 이 발생하는 경우 Virtual Thread 가 더 좋은 처리량을 보여준다. 단, I/O 에 대한 처리를 넘길 때 뒷부분에 까지 넘어가서 그 부분이 압도되서 뒤에서 처리를 못하는 이슈가 발생할 수 있음    
    

### 요약      
- 적합한 사용처
  - I/O Blocking 이 발생하는 경우 Virtual Thread 가 적합
  - CPU Intensive 작업에서는 적합하지 않다. (Ex: 이미지 프로세싱, 동영상 인코딩 등..)
  - Spring MVC 기반 Web API 제공시 편리하게 사용할 수 있다. (높은 throughput 을 위해서 Webflux 를 고려중이라면 대안이 될 수 있다.)
- Virtual Thread 에 대한 오해 
  - Virtual Thread 는 기존 (Platform) Thread 를 대체하는 것이 목적이 아니다. (도입한다고 무조건 처리량이 높아지지 않으며 둘 다 사용 가능하다.)
  - Virtual Thread 는 기다림에 대한 개선, 그리고 플랫폼 디자인과의 조화
  - Virtual Thread 는 그 자체로 Java 의 동시성을 완전히 개선했다고 보기 어렵다. (제약이 있음)
- Virtual Thread 의 제약
  - Thread Pool 에 적합하지 않다. Task 별로 Virtual Thread 르 할당해야 한다.
  - Thread Local 사용시 메모리 사용이 늘어날 수 있다.
  - synchronized 사용시 주의가 필요하다. (carrier thread 가 blocking 될 가능성이 있다. -> ReentrantLock 사용필요)
  - 제한된 리소스의 경우 semaphore 를 사용
  

### 참고
- https://www.youtube.com/watch?v=vQP6Rs-ywlQ