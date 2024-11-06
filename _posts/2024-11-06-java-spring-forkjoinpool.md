---
layout: post
title: "Java Fork Join Pool (작성중)"
subtitle: "2024-11-06-java-spring-forkjoinpool"
date: 2024-11-06 21:40:00
author: "전봉근"
header-img: "img/posts/android-bg.jpg"
comments: true
tags: [java, spring, spring boot, jvm]
---

## Java Fork Join Pool

#### 예제코드
[예제코드](https://github.com/bkjeon1614/java-example-code/tree/main/java11/bkjeon-mybatis-codebase)

#### Fork Join Pool 이란
ForkJoinPool 은 Java7부터 사용가능하며 동일한 작업을 여러개의 Sub Task로 분리(Fork)하여 각각 처리하고, 이를 최종적으로 합쳐서(Join) 결과를 만들어내는 방식이다. 즉, 대규모 작업을 빠르게 처리하는데 용이하다.      
![java-spring-forkjoinpool-1](/img/posts/language/java/spring/java-spring-forkjoinpool-1.png)               

#### 핵심 컴포넌트
- ForkJoinPool
  - ForkJoinTask 를 실행하는데 사용되는 스레드 풀로, 알고리즘 작업의 균형을 동적으로 조정하는 역할
- ForkJoinTask
  - 작업의 기본 단위, 작업에는 결과를 반환하지 않는 작업(RecursiveAction) 과 결과를 반환하는 작업(RecursiveTask) 의 두 가지 형태가 존재
- Work-Stealing Algorithm
  - ForkJoinPool 의 핵심 알고리즘 즉, 스레드가 `다른 작업의 작업을 훔치는 알고리즘` 이고, 그럼으로써 모든 스레드가 유휴시간 없이 계속해서 작업을 처리할 수 있도록 한다.    
  - 또한 이를 통하여 스레드 간의 작업 부하를 균형잡히게 유지하고, 모든 스레드가 계속 작업을 수행할 수 있도록 한다.

스레드풀의 큐(inbound queue) 에 작업이 할당되면 pool 내의 스레드들이 해당 작업을 가져와서 수행하며 각 스레드들은 다시 본인의 작업 큐(work queue) 를 가지고 가
![java-spring-forkjoinpool-2](/img/posts/language/java/spring/java-spring-forkjoinpool-2.png)               
     
#### 주의점
작업 분할 및 작업 스틸링에는 일정한 오버헤드가 있다. 상황에 따라 단순 threadPoolExecutor 를 사용하는 경우가 더 성능이 좋을 수 있다. (예를 들어 스레드가 4개인 경우 4개의 스레드가 동일한 일을 하게 된다면 오히려 독이지만 하나의 스레드가 굉장히 오래 걸리고 나머지 3개의 스레드가 금방 끝이나는 경우에 적합하다.)    

#### 사용법
- ParellelStream
  - parellelStream 내부적으로 ForkJoinPool 사용 (default: commonPool 사용, commonPool 이란 애플리케이션 코드에서 직접 사용하지 않고 JVM이 자동으로 생성)
  - parellelStream을 수행할 thread의 개수를 정해주고 싶은 경우 (단, 사용 완료 후 shutdown 호출해야 memory leak 방지)
    ```
    ForkJoinPool forkJoinPool = new ForkJoinPool(6);
    forkJoinPool.submit(() -> {   
        integerList.parallelStream().forEach((integer) -> { ...})
    }
    forkJoinPool.shutdown();    
    ```

## 참고
- https://kkang-joo.tistory.com/
- https://upcurvewave.tistory.com/
- https://junghyungil.tistory.com/