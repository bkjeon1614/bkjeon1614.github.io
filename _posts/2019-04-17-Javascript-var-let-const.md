---
layout: post
title : "Javascript var, let, const 차이"
subtitle : "2019-04-17-Javascript-var-let-const"
date: 2019-04-17 22:06:00
author: "전봉근"
header-img: "img/posts/android-bg.jpg"
comments: true
tags: [javascript]
---

Javascript var, let, const 차이점
=========

## Key Point
- var, let: 변수를 선언하는 키워드이다.
- const: 상수를 선언하는 키워드이다.

Example Code
```
    // 1. var는 값을 재 할당 가능하다.
    var name = "test";
    name = "test2";

    // 2. let은 var처럼 재 할당이 가능하다.
    let score = "1";
    score = "2";

    // 3. const는 값 재할당이 불가능하며 선언과 동시에 리터럴 값을 할당해야 한다.
    const PI = "3.14";
```

> let, const는 ECMA6에 도입된 키워드이며 var 타입으로 인한 혼동을 방지하기 위하여 만들어 졌다. var 타입의 혼동이 일어나는 이유에 대해선 아래를 참고하자.

1. var는 변수명을 재 선언해도 아무런 문제가 발생하지 않는다.
   ```
       var title = "테스트타이틀";

       // 위에 title을 선언했지만 깜박하여 재 선언해도 아무런 문제가 생기지 않는 크리티컬한 문제점이 있다.
       var title = "재선언타이틀"; 
   ```

2. 1번과 다르게 let은 변수명을 재 선언시에 에러를 발생시킨다.
   ```
       var title = "테스트타이틀";

       // SyntaxError를 발생 즉, 이미 선언된 변수라는것을 알 수 있다.
       var title = "재선언타이틀"; 
   ```   