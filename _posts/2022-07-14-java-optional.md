---
layout: post
title : "Java Optional"
subtitle : "2022-07-14-java-optional.md"
date: 2022-07-13 18:00:00
author: "전봉근"
header-img: "img/posts/android-bg.jpg"
comments: true
tags: [java]
---


# Optional
먼저 Optional 을 설명하기전에 NPE(=NullPointerException) 에 대해서 알아보자.

## NPE(=NullPointerException)
개발시 예외처리중 가장 많이 발생하는 오류가 NPE(=NullPointerException) 이다. NPE 를 방어하려면 null 여부를 검사해야 하는데 변수가 많을시에 코드의 가독성과 유지보수성이 떨어지기 때문에 null 대신 초기값을 사용하길 권장하기도 한다.


## Optional?
`Java8 에서 처음 도입`되었으며 Optional<T> 클래스를 사용하여 NPE를 방지할 수 있도록 도와준다. Optional 이란 `Null 이 될 가능성을 가진 값`을 객체로 감싸는 래퍼 클래스이다.
또한 하기 코드처럼 value 에 값을 저장하기 때문에 값이 null 이더라도 바로 NPE가 발생하지 않으며, 클래스이므로 각종 메소드를 제공해준다.
```
public final class Optional<T> {
  ...
  // If non-null, the value; if null, indicates no value is present
  private final T value;
  ...
}
```
> 사용법에 대해서는 [공식문서](https://docs.oracle.com/javase/8/docs/api/java/util/Optional.html) 를 참고해도 좋다.


## Optional 활용
- 





