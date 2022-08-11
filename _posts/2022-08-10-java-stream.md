---
layout: post
title : "Java Stream 이란"
subtitle : "2022-08-10-java-stream.md"
date: 2022-08-10 19:50:00
author: "전봉근"
header-img: "img/posts/android-bg.jpg"
comments: true
tags: [java]
---

Java Stream 이란
=========

개념은 알고 있지만 한 번 정리가 필요할 것 같아서 해당 포스팅을 작성하게 되었다.


## Stream
우리는 수 많은 데이터를 다룰 때, 컬렉션이나 배열에 데이터를 담고 원하는 결과를 얻기위해 for 문이나 Iterator 를 이용하여 코드를 작성해왔다. 그러나 이러한 코드는 가독성과 재사용성이 떨어진다. 또한 Collection 이나 Iterator 와 같은 인터페이스를 활용하여 컬렉션을 다루는 방식을 표준화하기는 하였지만. 각 컬렉션 클래스에는 같은 기능의 메서드들이 중복해서 정의되어 있다. 예를 들면 List 를 정렬하려면 Collections.sort() 를 사용하거나 배열은 Array.sort() 를 사용해야 한다.

이러한 문제점들을 해결하기 위해서 Java8 부터 추가된것이 `스트림(Stream)` 이다. `스트림` 은 데이터 소스를 추상화하고, 데이터를 다루는데 자주 사용되는 메서드들을 정의해 놓았다. 즉, 코드의 `재사용성`이 높아진다는 것을 의미한다.

예를 들어, 문자열 배열과 같은 내용의 문자열을 저장하는 컬렉션 List 가 있다고 가정하자.
```
String[] strArr = {"a", "d", "c"};
List<String> strList = Arrays.asList(strArr);
```

상기 데이터를 기반으로 스트림을 생성하자.
```
Stream<String> strArrStream = Arrays.stream(strArr);
Stream<String> strListStream = strList.stream();
```

생성된 두 개의 스트림을 통하여 정렬하고 화면에 출력하자. (데이터 소스가 정렬되는 것은 아니라는 것에 유의하자)
```
/**
  * 결과값
  * a
  * c
  * d
  * ===========================
  * a
  * c
  * d
  */
strArrStream.sorted().forEach(System.out::println);
System.out.println("===========================");
strListStream.sorted().forEach(System.out::println);
```

만약, 위의 스트림을 사용하지 않는다면 하기 코드와 같이 작성을 했어야 하므로 `스트림을 사용하면 코드가 간결하고 이해하기 쉬우며 재사용성이 높아지는 것을 알 수 있다.`
```
Arrays.sort(strArr);
Collections.sort(strList);

for (String str: strArr) {
    System.out.println(str);
}

for (String str: strList) {
    System.out.println(str);
}
```


## Stream 특징

#### 스트림은 데이터 소스를 변경하지 않는다.
스트림은 데이터 소스로 부터 데이터를 읽기만할 뿐, 데이터 소스를 변경하지 않으며 필요시 정렬된 결과를 컬렉션이나 배열에 담아서 반활할 수도 있다.
```
// 정렬된 겨로가를 새로운 List에 담아서 변환
List<String> sortedList = strArrStream.sorted().collect(Collections.toList());
```

#### 스트림은 일회용이다.
스트림은 Iterator 처럼 일회용이다. 한 번 사용하면 닫혀서 다시 사용할 수 없다. 필요하다면 스트림을 다시 생성해야한다.
```
strListStream.sorted().forEach(System.out::println);
int strListNum = strListStream.count(); // 에러발생!! 스트림이 이미 닫혀있음
```

#### 스트림은 작업을 내부 반복으로 처리한다.
내부반복은 반복문을 메서드의 내부에 숨길 수 있다는 것을 의미한다. 즉, forEach() 는 스트림에 정의된 메서드 중의 하나로 매개변수에 대입된 람다식을 데이터 소스의 모든 요소에 적용한다.
```
// 스트림이 아닐 때
for (String str: strArr) {
    System.out.println(str);
}

// 스트림일 때
stream.forEach(System.out::println);
```
> 즉, forEach() 는 메서드 안으로 for 문을 넣은 것이다.


## Stream 연산

#### 중간연산과 최종연산
스트림이 제공하는 다양한 연산을 이용하여 복잡한 작업들을 간단히 처리할 수 있다. 연산은 `중간연산` 과 `최종연산` 으로 분류할 수 있는데, `중간연산`은 연산결과를 스트림으로 반환하기 때문에 `중간연산`을 연속해서 연결할 수 있지만 `최종연산` 은 스트림의 요소를 소모하면서 연산을 수행하므로 단 한번만 연산이 가능
```
// distinct, limit, sorted 는 [중간연산]
// forEach 는 [최종연산]
stream.distinct().limit(5).sorted().forEach(System.out::println);
```


## 병렬 스트림

#### 병렬 스트림 처리
스트림으로 데이터를 다룰 때의 장점 중 하나가 바로 병렬 처리가 쉽다는 것이다. 개발자가 할일이라고 그저 스트림에 `parallel()` 이라는 메서드를 호출해서 병렬로 연산을 수행하도록 지시하면 될 뿐이다. 반대로 병렬로 처리되지 않게 하려면 sequential() 을 호출하면 되지만 이것은 스트림이 기본적으로 병렬 스트림이 아니므로 호출할 필요가 없다.
```
int sum = strListStream.parallelStream().mapToInt(s -> s.length()).sum();
```

#### ParallelStream 사용 시 주의사항
- 데이터가 많을수록 유리하며 적을수록 수행속도가 느려진다.
- `박싱(=Boxing)을 주의하자.`
  - 박싱은 성능을 크게 저하시킬 수 있는 요소이기 때문에 기본형 특화 스트림(IntStream, LongStream .. 등) 을 제공한다.
- `순서 연산에 주의하자` (Ex: limit(), findFirst())
  - 요소의 순서와 상관없이 연산하는 `findAny()` 를 사용하자. (요소의 순서에 관련있는 findFirst 보다 성능이 좋다.)
  - 정렬된 스트림에 `unordered()` 를 호출하면 비정렬된 스트림을 얻을 수 있으므로 해당 스트림은 요소의 순서가 상관 없으므로 `limit()` 를 호출할시 효율적이다.
- 스트림에서 수행하는 전체 파이프라인 연산 비용을 고려해야 한다.
  - 전체 스트림 파이프 라인 처리 비용
    - N(처리해야할 요소 수) * Q(하나의 요소를 처리하는데 드는 비용)
    - Q가 높아진다는 것은 병렬 스트림으로 성능을 개선할 수 있는 가능성이 있음을 의미
- 스트림의 특성과 파이프라인의 중간 연산이 스트림의 특성을 어떻게 바꾸는지에 따라 분해 과정의 성능이 달라질 수 있다.
  - 사이즈가 명확한 스트림은 정확히 같은 크기의 두 스트림으로 분할 할 수 있으므로 효과적으로 병렬처리가 가능하다.
  - 만약 필터 연산이 있으면 스트림의 사이즈를 예측할 수 없으므로 병렬처리가 효율적인지 예측이 어렵다.
- 최종 연산의 병합 과정 비용을 살펴보자. (병합과정이 비싸면 서브스트림의 부분결과를 합치는 과정에서 상쇄될 수 있다.)
  - 매우 좋음
    - ArrayList, IntStream.range
  - 좋음
    - HashSet, TreeSet
  - 나쁨
    - LinkedList, Stream.iterate


## 참고
- 자바의 정석 (남궁성 지음)
- https://yongho1037.tistory.com/
- https://n1tjrgns.tistory.com/
- https://tweety1121.tistory.com/