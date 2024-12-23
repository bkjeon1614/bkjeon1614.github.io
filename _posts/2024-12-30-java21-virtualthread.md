---
layout: post
title: "Java 21 Virtual Thread (작성중)"
subtitle: "2024-12-30-java21-virtualthread"
date: 2024-12-30 20:40:00
author: "전봉근"
header-img: "img/posts/android-bg.jpg"
comments: true
tags: [java, jvm]
---

## Java 21 Virtual Thread

### Virtual Thread
JDK 21(LTS) 에 추가된 경량 스레드 OS 스레드를 그대로 사용하지 않고 JVM 내부 스케줄링을 통하여 수십 ~ 수백만개의 스레드를 동시에 사용할 수 있게 한다.

### 전통적인 Java 의 Thread
- Java 의 Thread 는 OS Thread 를 Wrapping 한 것 (=Platform Thread)
- Java 애플리케이션에서 Thread 를 사용하면 실제로는 OS Thread 를 사용한 것
- OS Thread 는 생성 갯수가 제한적이고 생성, 유지하는 비용이 비쌈

> 이러한 내용 떄문에 플랫폼 스레드를 효율적으로 사용하기 위해 Thread Pool 을 사용

![java21-virtualthread-1](/img/posts/language/java/java21-virtualthread-1.png)               




### 참고
- https://www.youtube.com/watch?v=vQP6Rs-ywlQ