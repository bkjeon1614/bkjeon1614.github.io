---
layout: post
title: "@Async 를 활용한 비동기 호출 구현 (작성중)"
subtitle: "2024-10-14-java-spring-async"
date: 2024-10-14 21:40:00
author: "전봉근"
header-img: "img/posts/android-bg.jpg"
comments: true
tags: [java, spring, spring boot, jvm]
---

## @Async 를 활용한 비동기 호출 구현

#### 예제코드
[예제코드](https://github.com/bkjeon1614/java-example-code/tree/main/java11/bkjeon-mybatis-codebase)

#### @Async 를 사용해야 하는 이유
Spring @Async 를 사용하지 않으면 Thread 를 관리할 수 없어서 매우 위험하다. 예를 들어 동시에 1000 개의 호출이 요청된다면 짧은 시간 안에 Thread 를 1000 개를 생성해야 되므로 Thread 를 생성하는 비용은 적지 않기 때문에 프로그램 성능에 악영향을 끼칠 수 있다고 한다.

#### @Async
@EnableAsync 어노테이션을 설정 클래스에 추가해야 합니다. 이 설정을 통해 스프링은 비동기 작업을 위한 설정을 활성화하고, AsyncTaskExecutor 인터페이스의 구현체를 사용하여 비동기 작업을 처리한다. (@EnableAsync 어노테이션을 통해 스프링이 비동기 처리를 위한 프록시 객체를 생성하고, 이를 통해 실제 메소드 호출을 비동기적으로 처리하기 때문)     
    
기본적으로 스프링은 SimpleAsyncTaskExecutor를 사용하지만, ThreadPoolTaskExecutor와 같은 다른 Executor를 사용하여 성능을 향상시킬 수 있다. 왜냐하면 SimpleAsyncTaskExecutor는 매 요청마다 새로운 스레드를 생성하는 반면, ThreadPoolTaskExecutor는 미리 생성된 스레드 풀을 사용하기 때문에 오버헤드를 줄이고 성능을 향상시킬 수 있다.

#### Java 비동기 구현에 대한 정의
- 리턴 값이 있는 경우 Future, ListenableFuture, CompletableFuture 을 사용하며 비동기 메소드의 반환 형태를 new AsyncResult() 로 리턴하면 된다.
  - Future: future.get() 은 블로킹을 통해 요청 결과가 올 때까지 기다리는 역할을 한다. 그래서 비동기 `블로킹 방식이 되어버려 성능이 좋지 않다.` 또한 여러 Future 를 조합하는 것도 어렵고 보통 Future는 사용하지 않는다고 한다.
    ```
    ...

    for (int i = 1; i <= 5; i++) {
        Future<String> future = asyncService.isFeatureReturn(i + "");
        log.info("============= isAsyncAnnotationReturn : {}", future.get());
    }        
    ```          
    ```
    ...
    @Async("threadPoolTaskExecutor")
    public Future<String> isFeatureReturn(String message) throws InterruptedException {
        log.info("============= Feature Task Start - {}", message);
        Thread.sleep(3000);
        return new AsyncResult<>("bkjeon-" + message);
    }    
    ```     
  - ListenableFuture: 콜백을 통해 논블로킹 방식으로 작업을 처리할 수 있다. 그러나 콜백 메소드의 단점이 생긴다. 즉, 중첩 구조를 가지기 때문에 코드 가독성이 떨어진다. (`ListenableFuture 는 spring 6 부터 deprecate 되었다고 한다.`)
    ```
    ...

    /**
        * future.get() 은 블로킹을 통해 요청 결과가 올 때까지 기다리는 역할을 한다.
        * 그러므로 비동기 블로킹 방식이 되어버려 성능이 좋지 않다. 보통 Future 는 사용하지 않는다.
        * [결과]
        * ============= Feature Task Start - 1
        * ============= isAsyncAnnotationReturn : bkjeon-1
        * ============= Feature Task Start - 2
        * ============= isAsyncAnnotationReturn : bkjeon-2
        * ============= Feature Task Start - 3
        * ============= isAsyncAnnotationReturn : bkjeon-3
        * ============= Feature Task Start - 4
        * ============= isAsyncAnnotationReturn : bkjeon-4
        * ============= Feature Task Start - 5
        * ============= isAsyncAnnotationReturn : bkjeon-5
    */    
    for (int i = 1; i <= 5; i++) {
        ListenableFuture<String> listenableFuture = asyncService.isListenableFutureReturn(i + "");
        listenableFuture.addCallback(System.out::println, error -> System.out.println(error.getMessage()));
    }    
    ```     
    ```
    ...

    /**
        * ListenableFuture 은 콜백을 통해 논블로킹 방식으로 작업을 처리할 수 있다.
        * addCallback() 메소드의 첫 번째 파라미터는 작업 완료 콜백 메소드, 두 번째 파라미터는 작업 실패 콜백 메소드를 정의하면 된다.
        * 스레드 풀의 core 스레드를 3개로 설정했으므로 "Task Start" 메시지가 처음에 3개 찍힘
        * [결과]
        * ============= Feature Task Start - 1
        * ============= Feature Task Start - 2
        * ============= Feature Task Start - 3
        * bkjeon-2
        * bkjeon-1
        * bkjeon-3
        * ============= Feature Task Start - 4
        * ============= Feature Task Start - 5
        * bkjeon-5
        * bkjeon-4
    */    
    @Async("threadPoolTaskExecutor")
    public ListenableFuture<String> isListenableFutureReturn(String message) throws InterruptedException {
        log.info("============= ListenableFuture Task Start - {}", message);
        Thread.sleep(3000);
        return new AsyncResult<>("bkjeon-" + message);
    }    
    ```     
  - `CompletableFuture`: Future, ListenableFuture 을 대신해 해당 클래스를 사용하는것을 권장한다.
    - 해당 기능의 샘플은 다음 Step 에서 확인하자.

#### CompletableFuture 
CompletableFuture 는 Future 와 다르게 Non-blocking 에 여러 연산을 함께 연결/결합 하기가 쉽고 예외처리에 용이하다.
       
- supplyAsync() 호출 시 ForkJoinPool.commonPool()에서 전달된 Supplier를 비동기적으로 호출한 뒤 CompleteableFuture 인스턴스를 반환
  ```
  
  ```
https://11st-tech.github.io/2024/01/04/completablefuture/







#### 참고
- https://steady-coding.tistory.com/611
- https://f-lab.kr/insight/understanding-spring-async-annotation-and-thread-pool
- https://velog.io/@brucehan/CompletableFuture-%EC%A0%95%EB%A6%AC
- https://11st-tech.github.io/2024/01/04/completablefuture/