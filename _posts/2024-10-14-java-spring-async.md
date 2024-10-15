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
  - Future: Future 인터페이스는 java5 부터 비동기 연산을 위해 추가되었다. 그리고 값을 리턴받기 위해선 future.get() 은 블로킹을 통해 요청 결과가 올 때까지 기다리는 역할을 한다. 그래서 비동기 `블로킹 방식이 되어버려 성능이 좋지 않다.` 또한 여러 Future 를 조합하는 것도 어렵고 보통 Future는 사용하지 않는다고 한다. 또한 여러 연산을 결합하기 어렵고 예외처리가 힘들어 잘 사용하지 않는다고 한다.
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
  - ListenableFuture: 콜백을 통해 논블로킹 방식으로 작업을 처리할 수 있고 값을 리턴받을 때 Future 처럼 블로킹 방식으로 받지 않아서 성능에 용이하다. 그러나 콜백 메소드의 단점이 생긴다. 즉, 중첩 구조를 가지기 때문에 코드 가독성이 떨어진다. (`ListenableFuture 는 spring 6 부터 deprecate 되었다고 한다.`)
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
###### CompletableFuture 기본 사용 (여러 메서드가 존재하지만 보편적으로 사용되는 메서드 위주로 설명)
CompletableFuture 는 Java8 에 등장하였으며 Future 와 다르게 Non-blocking 에 여러 연산을 함께 연결/결합 하기가 쉽고 예외처리에 용이하다.      
     
CompletableFuture 는 Future, CompleteStage 인터페이스를 구현하고 있다.
- Future: java5에서 비동기 연산을 위해 추가된 인터페이스
- CompleteStage: 여러 연산을 결합할 수 있도록 연산이 완료되면 다음 단계의 작업을 수행하거나 값을 연산하는 비동기식 연산 단계를 제공하는 인터페이스     
          
먼저 CompletableFuture 클래스의 정적 메서드인 supplyAsync() 메서드를 통해 CompletableFuture 인스턴스를 생성할 수 있다. 또한 Supplier 인수로 upplyAsync() 호출 시 ForkJoinPool.commonPool()에서 전달된 Supplier를 비동기적으로 호출한 뒤 CompleteableFuture 인스턴스를 반환한다.       
```
public void isCompletableFutureReturn(String message) throws ExecutionException, InterruptedException {
    CompletableFuture<String> testInfoFuture = CompletableFuture
        .supplyAsync(() -> getAsyncTestInfo(message));
    log.info("============= CompletableFuture Return - {}", testInfoFuture.get());
}

private String getAsyncTestInfo(String message) {
    try {
        Thread.sleep(3000);
    } catch (InterruptedException ie) {
        log.error("InterruptedException", ie);
    }

    return "bkjeon async";
}
```      
      
###### 순차적 연산 처리
복잡한 연산을 비동기로 순차적으로 처리해야 할 때 하기 메소드들을 참고       
- thenApply()
  - 이전 단계의 결괏값을 인수로 사용하고, 전달한 Function 을 다음 연산으로 사용    
    ```
    public void isCompletableFutureReturnThenApply(String message) throws ExecutionException, InterruptedException {
        CompletableFuture<String> testInfoFuture = CompletableFuture
            .supplyAsync(() -> getAsyncTestInfo(message));
        CompletableFuture<String> testInfoFuture2 = testInfoFuture.thenApply(this::getAsyncTestInfo2);
        log.info("============= CompletableFuture Then Apply Return - {}", testInfoFuture2.get());  // 이전 연산 완료 후(2초 후) 다음 연산 처리
    }

    private String getAsyncTestInfo(String message) {
        try {
            Thread.sleep(3000);
        } catch (InterruptedException ie) {
            log.error("InterruptedException", ie);
        }

        return "bkjeon async";
    }

    private String getAsyncTestInfo2(String message) {
        try {
            Thread.sleep(5000);
        } catch (InterruptedException ie) {
            log.error("InterruptedException", ie);
        }

        return "bkjeon async2";
    }    
    ```     
- thenAccept()    
  - Consumer를 인자로 받고, 결과를 CompletableFuture<Void>를 반환하며, 리턴 타입이 없는 로직을 호출할 때 사용할 수 있다.
    ```
    public void isCompletableFutureReturnThenAccept(String message) throws ExecutionException, InterruptedException {
        CompletableFuture<String> testInfoFuture = CompletableFuture
            .supplyAsync(() -> getAsyncTestInfo(message));
        CompletableFuture<Void> thenAccept = testInfoFuture.thenAccept(this::voidTest);
        thenAccept.get();   // Completed: bkjeon async
    }    

    private String getAsyncTestInfo(String message) {
        try {
            Thread.sleep(3000);
        } catch (InterruptedException ie) {
            log.error("InterruptedException", ie);
        }

        return "bkjeon async";
    }    

    private void voidTest(String message) {
        log.info("Completed: " + message);
    }    
    ```   
- thenRun()        
  - Runnable를 인자로 받고, thenAccept() 메서드와 동일하게 결과를 CompletableFuture<Void>로 반환된다. 댠, thenAccept() 메서드와 다르게 CompletableFuture<Void>.get() 호출 없이 연산이 실행    
    ``` 
    public void isCompletableFutureReturnThenRun(String message) {
        CompletableFuture<String> testInfoFuture = CompletableFuture
            .supplyAsync(() -> getAsyncTestInfo(message));
        testInfoFuture.thenRun(() -> voidTest(message));    // Completed: test thenRun()
    }    

    private void voidTest(String message) {
        log.info("Completed: " + message);
    }    
    ```      

###### 연산 결합하기
복잡한 연산을 결합할 때 하기 메소드를 참고
- thenCompose()
  - CompletableFuture 인스턴스를 결합하여 연산을 처리 (두 개의 Future를 순차적으로 연결 -> 이전 단계의 결과(CompletionStage)를 다음 CompletableFuture 안에서 사용)      
    ```
    public void isCompletableFutureReturnThenCompose(String message) throws ExecutionException, InterruptedException {
        CompletableFuture<String> testInfoFuture = CompletableFuture
            .supplyAsync(() -> getAsyncTestInfo(message)).thenCompose(s -> CompletableFuture.supplyAsync(() -> getAsyncTestInfo2(s)));
        log.info("============= CompletableFuture Then Compose Return - {}", testInfoFuture.get());
    }    

    private String getAsyncTestInfo(String message) {
        try {
            Thread.sleep(3000);
        } catch (InterruptedException ie) {
            log.error("InterruptedException", ie);
        }

        return "bkjeon async";
    }

    private String getAsyncTestInfo2(String message) {
        try {
            Thread.sleep(5000);
        } catch (InterruptedException ie) {
            log.error("InterruptedException", ie);
        }

        return "bkjeon async2";
    }    
    ```
- thenCombine()    
  - 두 개의 독립적인 Future를 처리하고 두 결과를 결합하여 추가적인 작업을 수행 (Future와 Functional Interface인 BiFunction를 파라미터로 받아서 두 결과를 결합한 추가적인 처리가 가능)    
    ```
    public void isCompletableFutureReturnThenCombine(String message) throws ExecutionException, InterruptedException {
        CompletableFuture<String> testInfoFuture = CompletableFuture.supplyAsync(() -> getAsyncTestInfo(message))
            .thenCombine(CompletableFuture.supplyAsync(() -> getReturnStr("message1")), this::getCombineReturnStr);

        // ============= CompletableFuture Then Combine Return - bkjeon async. [옵션] Test!!!!
        log.info("============= CompletableFuture Then Combine Return - {}", testInfoFuture.get());
    }
    
    private String getAsyncTestInfo(String message) {
        try {
            Thread.sleep(3000);
        } catch (InterruptedException ie) {
            log.error("InterruptedException", ie);
        }

        return "bkjeon async";
    }       

    private String getReturnStr(String message) {
        return "Test!!!!";
    }

    private String getCombineReturnStr(String message1, String message2) {
        return message1 + ". [옵션] " + message2;
    }     
    ```     
- thenAcceptBoth()
  - thenCombine() 메서드와 유사하지만 thenAcceptBoth() 메서드는 결괏값을 전달할 필요가 없으면 간단하게 사용할 수 있다. (주로 thenCombine() 는 데이터 조회 후 추가적인 정제 작업에 쓰이고, thenAcceptBoth() 는 insert, update 와 같은 작업에 유용하다.)
    ```
    public void isCompletableFutureReturnThenAcceptBoth(String message) {
        // 결과: bkjeon async. [옵션] Test!!!!
        CompletableFuture.supplyAsync(() -> getAsyncTestInfo(message))
            .thenAcceptBoth(CompletableFuture.supplyAsync(() -> getReturnStr("message1")), this::getUpdateInfo);
    }

    private String getAsyncTestInfo(String message) {
        try {
            Thread.sleep(3000);
        } catch (InterruptedException ie) {
            log.error("InterruptedException", ie);
        }

        return "bkjeon async";
    }

    private String getReturnStr(String message) {
        return "Test!!!!";
    }    

    private void getUpdateInfo(String message1, String message2) {
        log.info(message1 + ". [옵션] " + message2);
    }
    ```     

###### thenApply or thenCompose
보다보면 두 개의 사용기준이 모호한 경우가 있다. 그럴 경우 하기 내용을 참고하자.
- 각 CompletableFuture의 호출 결과가 필요할 경우, thenApply() 메서드를 사용하는 것이 적합
- 각 CompletableFuture의 결과를 결합한 최종 연산 결과만 필요한 경우, thenCompose() 메서드를 사용하는 것이 적합

#### 병렬처리










#### 참고
- https://steady-coding.tistory.com/611
- https://f-lab.kr/insight/understanding-spring-async-annotation-and-thread-pool
- https://velog.io/@brucehan/CompletableFuture-%EC%A0%95%EB%A6%AC
- https://11st-tech.github.io/2024/01/04/completablefuture/