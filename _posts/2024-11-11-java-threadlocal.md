---
layout: post
title: "Java ThreadLocal"
subtitle: "2024-11-11-java-threadlocal"
date: 2024-11-11 20:40:00
author: "전봉근"
header-img: "img/posts/android-bg.jpg"
comments: true
tags: [java, spring, spring boot, jvm, thread]
---

## Java ThreadLocal
[참고코드](https://github.com/bkjeon1614/java-example-code/tree/main/java11/bkjeon-mybatis-codebase/base-api)

### Java ThreadLocal 이란
Spring Bean 은 싱글톤으로 등록이 된다. 이러한 상황에서 하나의 인스턴스에 여러개의 스레드가 동시에 접근하면 동시성 문제가 발생하게 된다.      
이러한 경우 스레드 세이프하게 하려면 ThreadLocal 을 사용하면 된다. `ThreadLocal 은 해당 Thread 만 접근할 수 있는 특별한 저장소를 말한다. 즉, 각 Thread 마다 별도의 내부 저장소를 제공`       
          
### 동시성 문제 예제
하기 코드를 실행해보면 동시성 문제가 발생한다.     
[ThreadLocalServiceTest.java]
```
package com.example.bkjeon.base.services.api.v1.thread.service;

import lombok.extern.slf4j.Slf4j;
import org.springframework.stereotype.Service;

@Slf4j
@Service
public class ThreadLocalServiceTest {

    private String nameStore;

    // 매개변수를 통해 넘어온 name을 전역 변수인 nameStore에 저장 후, 1초 뒤 조회 로그를 찍어주는 메소드
    public String logic(String name) {
        log.info("저장 name={} -> nameStore={}", name, nameStore);
        nameStore = name;
        sleep(1000);
        log.info("조회 nameStore={}",nameStore);
        return nameStore;
    }

    private void sleep(int millis) {
        try {
            Thread.sleep(millis);
        } catch (InterruptedException e) {
            e.printStackTrace();
        }
    }

}
```
[ThreadController.java]
```
...

private final ThreadLocalServiceTest threadLocalServiceTest;

@ApiOperation("ThreadLocal 테스트(동시성 문제 발생 테스트)")
@GetMapping("/isThreadLocalTest")
public void isThreadLocal() {
    log.info("main start");
    Runnable userA = () -> {
        threadLocalServiceTest.logic("userA");
    };
    Runnable userB = () -> {
        threadLocalServiceTest.logic("userB");
    };

    Thread threadA = new Thread(userA);
    threadA.setName("thread-A");
    Thread threadB = new Thread(userB);
    threadB.setName("thread-B");
    threadA.start(); //A실행

    sleep(2000); //동시성 문제 발생X
    // sleep(100); //동시성 문제 발생O

    threadB.start(); //B실행
    sleep(3000); //쓰레드B가 끝날 때까지 메인 쓰레드 종료 대기
    log.info("main exit");
}

private void sleep(int millis) {
    try {
        Thread.sleep(millis);
    } catch (InterruptedException e) {
        e.printStackTrace();
    }
}
```

- Thread 2개를 생성 후 2초 간격으로 두 스레드 실행
  - 동시성 문제 발생 X
    ```
    저장 name=userA -> nameStore=userB
    조회 nameStore=userA
    저장 name=userB -> nameStore=userA
    조회 nameStore=userB
    ```
- Thread 2개를 생성 후 0.1초 간격 두 스레드 실행
  - 동시성 문제 발생 O
    ```
    저장 name=userA -> nameStore=null
    저장 name=userB -> nameStore=userA
    조회 nameStore=userB
    조회 nameStore=userB
    ```
  - 동시성 문제가 발생하는 이유
    - 스프링 빈은 싱글톤으로 생성되기 때문에 인스턴스 1개를 공유해서 함께 사용한다.
    - 스레드A가 nameStore 에 저장 후 조회를 하는데는 1초의 지연시간이 있다. 그 지연시간동안 스레드B가 nameStore 에 접근해서 값을 바꿔버리게 되면 스레드A는 변경된 값을 얻게 되므로 문제가 생긴다.

### ThreadLocal 사용
상기 동시성 문제가 발생하였던 코드를 수정해보자.     
[ThreadLocalService.java]
```
package com.example.bkjeon.base.services.api.v1.thread.service;

import lombok.extern.slf4j.Slf4j;
import org.springframework.stereotype.Service;

@Slf4j
@Service
public class ThreadLocalService {

    private ThreadLocal<String> nameStore = new ThreadLocal<>();

    public String logic(String name) {
        log.info("저장 name={} -> nameStore={}", name, nameStore.get());
        nameStore.set(name);
        sleep(1000);
        log.info("조회 nameStore={}", nameStore.get());
        return nameStore.get();
    }

    private void sleep(int millis) {
        try {
            Thread.sleep(millis);
        } catch (InterruptedException e) {
            e.printStackTrace();
        }
    }

}
```    

### 주의할점
`ThreadLocal 은 ThreadLocal Pool 환경에서 사용시에 반드시 remove() 를 호출하여 관리해야 한다.` 그렇지 않으면 해당 스레드를 부여받게 되는 다른 사용자가 기존에 세팅된 ThreadLocal 의 데이터를 공유하게 될 수도 있기 때문이다.

### 참고
- https://velog.io/@gmtmoney2357/
- https://velog.io/@wken5577/