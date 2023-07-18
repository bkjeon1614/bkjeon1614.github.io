---
layout: post
title: "JDK8 에서 Perm 영역 삭제와 Metaspace 영역의 추가"
subtitle: "2023-07-18-java-memory-metaspace.md"
date: 2023-07-17 18:40:00
author: "전봉근"
header-img: "img/posts/android-bg.jpg"
comments: true
tags: [java, jvm, spring, spring boot, os]
---

# JDK8 에서 Perm 영역 삭제와 Metaspace 영역의 추가

## 시작에 앞서
먼저 Java8 부터 삭제되는 영역인 Permanent Generation 부터 알아보자면 `Class 혹은 Method Code 가 저장되는 영역`이다. 줄여서 PermGen 이라고 하며, Heap 영역에 속한다.   
PermGen 에는 로드된 클래스의 정보, 정적 변수, 상수 정보 등 변하지 않을 것이라고 어느 정도 보증되는 데이터가 저장된다고 한다.  

## PermGen -> Metaspace 영역으로 변경된 이유
PermGen 은 메모리가 제한되기 때문에 OOM(=OutOfMemoryError) 이 발생하게 된다. 그래서 해당 문제를 해결하기 위해 Native 메모리를 사용하는 Metaspace 로 변경되었으며 `Metaspace 영역은 Native 메모리를 이용함으로써 개발자는 영역 확보의 상한을 크게 의삭할 필요가 없어지게 된다.`    
![java-jvm-1](/img/posts/language/java/java-jvm-1.png)          

## PermGen VS Metaspace
||Java 7|Java 8|
|------|---|---|
|Class 메타 데이터|저장|저장|
|Method 메타 데이터|저장|저장|
|Static Objecvt 변수, 상수|저장|Heap 영역으로 이동|
|메모리 튜닝|Heap, Perm 영역 튜닝|Heap 튜닝, Native 영역은 OS가 동적 조정|
|메모리 옵션|-XX:PermSize, -XX:MaxPermSize|-XX:MetaspaceSize, -XX:MaxMetaspaceSize|

## 참고
- https://johngrib.github.io
- https://blog.voidmainvoid.net










