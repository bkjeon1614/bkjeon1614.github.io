---
layout: post
title : "JAVA Garbage Collection (작성중)"
subtitle : "2019-04-05-JAVA Garbage Collection"
date: 2019-04-05 19:00:00
author: "전봉근"
header-img: "img/posts/android-bg.jpg"
comments: true
tags: [java]
---

# Garbage Collection 이란
작성필요

# Garbage Collection 과정
## Stop-the-world
GC를 실행하기 위해 JVM이 애플리케이션 실행을 멈추는 것이다. 이것이 발생하면 GC를 실행하는 쓰레드를 제외한 나머지 쓰레드는 모두 작업을 멈춘다. GC 작업을 완료한 이후에야 중단했던 작업을 다시 시작한다.  
어떤 GC 알고리즘을 사용하더라고 stop-the-world는 발생한다. 즉, GC 튜닝이란 stop-the-world 시간을 줄이는 것이다.

Java는 프로그램 코드에서 메모리를 명시적으로 지정하여 해제하지 않는다. 그렇기 때문에 GC가 더 이상 필요 없는 쓰레기 객체를 찾아 지우는 작업을 한다. 하지만 명시적으로 해제하려고 해당 객체를 null로 지정하거나 System.gc() 메서드를 호출하는 개발자가 있다. Null로 지정하는 것은 큰 문제가 안 되지만, System.gc() 메서드를 호출하는 것은 시스템의 성능에 영향을 야기하므로 절대로 사용하면 안된다.

### 아래의 2가지 가정에 의하여 만들어진 것이 GC이다.
- 대부분의 객체는 금방 접근 불가능한 상태가 된다.
- 오래된 객체에서 젊은 객체로의 참조는 아주 적게 존재한다.  

이 가정의 장점을 최대한 살리기 위하여 HotSpot VM(=자바엔진이름)에서는 크게 2개로 물리적 공간을 나누었다. 그것이 Young 영역과 Old 영역이다.

- **Young 영역(=Young Generation 영역)**: 새롭게 생성한 객체의 대부분이 여기에 위치한다. 대부분의 객체가 금방 접근 불가능 상태가 되기 때문에 매우 많은 객체가 Young 영역에 생성되었다가 사라진다. 이 영역에서 객체가 사라질 때 Minor GC가 발생한고 말한다.
- **Old 영역(=Old Generation 영역)**: 접근 불가능 상태로 되지 않아 Young 영역에서 살아남은 객체가 여기로 복사된다. 대부분 Young 영역보다 크게 할당하며, 크기가 큰 만큼 Young 영역보다 GC는 적게 발생한다. 이 영역에서 객체가 사라질 때 Major GC(혹은 Full GC)가 발생한다고 말한다.


작성중...