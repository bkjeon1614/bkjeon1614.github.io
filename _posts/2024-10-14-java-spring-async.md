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

## Java Fork Join Pool

#### 예제코드
[예제코드](https://github.com/bkjeon1614/java-example-code/tree/main/java11/bkjeon-mybatis-codebase)

#### Fork Join Pool 이란
ForkJoinPool 은 Java7부터 사용가능한 Java Concurrency 툴이며, 동일한 작업을 여러개의 Sub Task로 분리(Fork)하여 각각 처리하고, 이를 최종적으로 합쳐서(Join) 결과를 만들어내는 방식이다. ???

## 참고
- https://upcurvewave.tistory.com/653
- https://junghyungil.tistory.com/103