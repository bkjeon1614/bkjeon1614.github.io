---
layout: post
title : "Log4J 에서 isDebugEnabled() 을 사용한 효율적인 리소스 관리"
subtitle : "2020-05-08-log4j-logging-enabled.md"
date: 2020-05-08 10:00:00
author: "전봉근"
header-img: "img/posts/android-bg.jpg"
comments: true
tags: [java, log4j, spring]
---

## Log4J 에서 isDebugEnabled() 을 사용한 효율적인 리소스 관리
----------------------------------------------------------------

일반적으로 log4j를 사용하는 코드를 보자.  
```
   log.debug("error message example")
```
  
위와 같은 방식으로 사용하는 경우가 있으며 아래 코드를 보자  
```
   if (log.isDebugEnabled()) {
      log.debug("error message example");
   }
```  
  
이렇게 되면 두번이나 체크하게 될텐데 효과적일까? 라는 의문을 갖게 된다.  
예를 들어 log.debug("Entry Number: " + i + ", Value: " + String.valueOf(entry[i])) 이런식으로 디버깅을 사용한다고 하자.  
이러면 메세지 파라미터를 생성할 때 String 연산들이 일어나게 되며 해당 작업은 메세지 로깅여부에 상관없이 항상 발생하게 되어 파라미터 생성 비용을 발생시킬 수 있다.

> log4j 에서는 왠만하면 항상 enabled를 체크하는 로직을 넣는것이 좋다. 