---
layout: post
title : "Java Optional"
subtitle : "2022-07-22-java-optional.md"
date: 2022-07-22 18:00:00
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


## Optional 을 잘못 사용하는 경우 발생하는 문제
- NPE 대신 NoSuchElementException 가 발생
  ```
  Optional<Test> optionalTest = ...;
  
  // Optional 이 갖는 value 가 없으면 NoSuchElementException 가 발생
  ```
- Optional 은 직렬화를 할 수 없다. (즉, 선택형 반환값을 지원하는 용도로만 사용해야 한다.)
  - 클래의 내부를 살펴보면 java.io.Serializable 인터페이스를 구현하지 않는다. 그러므로 NotSerializableException 예외가 발생한다.
  - 또한 클래스 필드 형식으로 사용할 것을 가정하지 않았기 때문에, Serializable 인터페이스를 구현하지 않는다. 따라서 도메인 모델에 Optional을 사용한다면 직렬화시 문제가 생길 수 있다.
  - 그래서 `별도로 Optional 메소드를 도메인에 추가하는 방식을 권장한다.`
    ```
    public class Bong {
        // 직렬화 용도
        private Test test;

        // Optional 용도
        public Optional<Test> getTestAsOptional() {
            return Optional.ofNullable(test);
        }
    }
    ```
- 코드의 가독성 하락
  ```
  // 해당 test 메소드는 optionalTest 값이 비어있으면 NoSuchElementException가 발생한다. 그 부분을 Optional 로 대응했다고 하더라도 optionalTest 객체 자체가 null 이면 NPE 가 발생할 수 있다. 그러므로 하기 test2 메소드와 같이 수정해야한다.
  public void test(Optional<Test> optionalTest) {
      Test test = optionalTest.orElseThrow(IllegalStateException::new);
      // 후 처리 로직
  }

  // 이렇게되면 검사가 더 추가되므로 코드가 복잡해지며 코드 글자수까지 증가했다. 즉, Optional 을 남용하면 가독성이 떨어질 수 있음을 보여주는 예시이다.
  public void test2(Optional<Test> optionalTest) {
      if (optionalTest != null && optionalTest.isPresent()) {
          // 후 처리 로직
      }
      
      throw new IllegalStateException();
  }
  ```
- 오버헤드 증가
  - Optional은 객체를 감싸는 컨테이너이므로 Optional 객체 자체를 저장하기 위한 메모리가 추가로 필요하다. (공간적 비용)
  - Optional 안에 있는 객체를 얻기 위해서는 Optional 객체를 통해 접근해야 하므로 접근 비용이 증가한다. (시간적 비용)


## 올바른 Optional 사용법
- Optional 변수에 Null을 할당하지 말자
  ```
  // Optional 변수에 null을 할당하는 것은 Optional 변수 자체가 null인지 또 검사해야 하는 문제를 야기한다. 그러므로 값이 없는 경우라면 Optional.empty()로 초기화하도록 하자.
  public Optional<Test> getTest() {
      Optional<Test> emptyTest = null;
      ...
  }
  ```
- 값이 없을 때 Optional.orElseGet()과 같은 함수로 기본 값을 반환하라
  ```
  // Optional의 장점 중 하나는 함수형 인터페이스를 통해 가독성좋고 유연한 코드를 작성할 수 있다는 것이다. 가급적이면 isPresent()로 검사하고 get()으로 값을 꺼내기 보다는 orElseGet 등을 활용해 처리하도록 하자.
  // orElseGet 은 값이 준비되어 있지 않은 경우, orElse는 값이 준비되어 있는 경우에 사용하면 된다. 만약 null을 반환해야 하는 경우라면 orElse(null)을 활용하도록 하자. 만약 값이 없어서 throw 해야하는 경우라면 orElseThrow를 사용하면 되고 그 외에도 다양한 메소드들이 있으니 적당히 활용하면 된다.
  public String getTest(Long id) {
      String test = "";
      return Optional.ofNullable(test).orElse("test empty");
  }
  ```
- `단순히 값을 얻으려는 목적으로만 Optional을 사용하지 마라`
  ```
  // 단순히 값을 얻으려고 Optional을 사용하는 것은 Optional을 남용하는 대표적인 경우이다. 이러한 경우에는 굳이 Optional을 사용해 비용을 낭비하는 것 보다는 직접 값을 다루는 것이 좋다.

  // Optional을 남용하는 나쁜코드
  public String getTest(Long id) {
      String name = ... ;
      
      return Optional.ofNullable(name).orElse("default value");
  }

  // 좋은 코드
  public String getTest(Long id) {
      String name = ... ;
      
      return name == null 
        ? "default value" 
        : name;
  }
  ```
- 생성자, 수정자, 메소드 파라미터 등으로 Optional을 넘기지 마라
  - Optional을 파라미터로 넘기는 것은 상당히 의미없는 행동이다. 왜냐하면 넘겨온 파라미터를 위해 자체 null체크도 추가로 해주어야 하고, 코드도 복잡해지는 등 상당히 번거로워지기 때문이다. Optional은 반환 타입으로 대체 동작을 사용하기 위해 고안된 것임을 명심해야 하며, 앞서 설명한대로 Serializable을 구현하지 않으므로 필드 값으로 사용하지 않아야 한다.
  - Optional을 접근자에 적용하는 경우도 마찬가지이다. 하기 코드에서 name을 얻기 위해 Optional.ofNullable()로 반환하고 있는데, Getter에 Optional을 얹어 반환하는 것을 두고 남용하는 것이다.
    ```
    ...
    private final String name;

    public Optional<String> getName() {
        return Optional.ofNullable(name);
    }
    ```
- Collection의 경우 Optional이 아닌 빈 Collection을 사용하라
  ```
  // testList 에 Null 이 올수 있다고 가정하자.

  // 잘못된 코드
  public Optional<List<User>> getTestList() {
      List<User> testList = ...;

      return Optional.ofNullable(testList);
  }

  // 좋은 코드
  public List<User> getTestList() {
      List<User> testList = ...;

      return testList == null 
        ? Collections.emptyList() 
        : testList;
  }  
  ```
  ```
  // 하기 케이스라면 map에 getOrDefault 메소드가 있으니 이걸 활용하는 것이 훨씬 좋다.

  // 잘못된 코드
  public Map<String, Optional<String>> getTestMap() {
      Map<String, Optional<String>> items = new HashMap<>();
      items.put("bong1", Optional.ofNullable(...));
      items.put("bong2", Optional.ofNullable(...));
      
      Optional<String> item = items.get("bong1");
      
      if (item == null) {
          return "default value"
      } else {
          return item.orElse("default value");
      }
  }

  // 좋은 코드
  public Map<String, String> getTestMap() {
      Map<String, String> items = new HashMap<>();
      items.put("bong1", ...);
      items.put("bong2", ...);
      
      return items.getOrDefault("bong1", "default value");
  }  
  ```
- 반환 타입으로만 사용하라
  - Optional은 `반환 타입으로써 에러가 발생할 수 있는 경우에 결과 없음을 명확히 드러내기 위해` 만들어졌으며, `Stream API와 결합되어 유연한 체이닝 api를 만들기 위해 탄생`한 것이다. 언어를 만드는 사람의 입장에서는 Null을 반환하는 것보다 값의 유무를 나타내는 객체를 반환하는 것이 합리적일 것이다.


참고
- https://mangkyu.tistory.com/
- https://madplay.github.io/

