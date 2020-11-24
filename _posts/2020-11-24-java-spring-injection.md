---
layout: post
title : "Field Injection이 아닌 Constructor Injection 사용하자"
subtitle : "2020-11-24-java-spring-injection.md"
date: 2020-11-24 10:00:00
author: "전봉근"
header-img: "img/posts/android-bg.jpg"
comments: true
tags: [java, springboot, spring]
---

# Field Injection이 아닌 Constructor Injection 사용하자
----------------------------------------------------------------

평소 개발시에 스프링쪽이라든지 타 포스팅을 참고하여 Field Injection이 아닌 Constructor Injection 을 지향하게 되었다.
하지만 그 당시 간략히 이해만하고 넘어간지라 최근에 해당 내용에 대해 설명을 해야할 일이 생겼을 때 간략한 내용만 전달하게 되어서 좀 더 자세한 내용을 다시 복습하고자 포스팅을 작성하게 된다.

## [의존성 주입의 종류]
- Setter Injection
  ```
    public class ExampleClass {
        @Autowired
        private ExampleService1 exampleService1;

        @Autowired
        private ExampleService2 exampleService2;
    }
  ```
- Field Injection
  ```
    public class ExampleClass {
        private ExampleService1 exampleService1;
        private ExampleService2 exampleService2;

        @Autowired
        public void setExampleService1(ExampleService1 exampleService1) {
            this.exampleService1 = exampleService1;
        }

        @Autowired
        public void setExampleService2(ExampleService2 exampleService2) {
            this.exampleService2 = exampleService2;
        }
    }
  ```
- Constructor Injection
  ```
    public class ExampleClass {
        private final ExampleService1 exampleService1;
        private final ExampleService2 exampleService2;

        @Autowired
        public ExampleClass(ExampleService1 exampleService1, ExampleService2 exampleService2) {
            this.exampleService1 = exampleService1;
            this.exampleService2 = exampleService2;
        }
    }
  ```

여기서 이전 회사들의 레거시 코드를 생각해보면 대부분 Field Injection을 사용 한 것으로 기억된다.

## [Field Injection 을 지양해야하는 이유]
- SOLID 원칙중 하나인 단일 책임의 원칙 위반(SRP - Single Responsiblity Principle)
  - `@Autowired 선언을 무분별하게 추가하게 된다.` 이 부분을 Constructor Injection을 사용한다면 생성자에 파라미터가 많아지므로 하나의 클래스가 많은 책임을 떠안게 된다는걸 인지하게 되어 리팩토링을 해야한다는 신호가 될 수 있다.
- 의존성이 숨는다.
  - DI(Dependency Injection)을 사용한다는건 클래스가 자신의 의존성만 책임지는게 아닌 제공된 의존성 또한 책임지는것이다. 즉, 클래스가 어떤 의존성을 책임지지 않을 때, 메서드나 생성자를 통해 확실히 커뮤니케이션이 되어야하지만 Field Injection은 숨은 의존성만 제공
- DI 컨테이너의 결합성과 테스트 용이성
  - Field Injection을 사용하면 필요한 의존성을 가진 클래스를 곧바로 인스턴스화 시킬 수 없으므로 테스트에 용이하지 않다.
- 불변성
  - Final을 선언할 수 없으므로 객체가 변할 수 있다.
- 순환 의존성
  - Constructor Injection에서 순환 의존성을 가질 경우 BeanCurrentlyCreationExeption을 발생시킴으로써 순환 의존성을 알 수 있다. (순환 의존성이란 여러개의 클래스가 서로 순환되어 참조되는 경우)

> Spring 3.X 에서는 Setter Injection을 추천했었고, 최근에는 본문 내용과 같이 Constructor Injection을 추천한다. 또한 Spring 4.3 이상의 버전부터는 @Autowired를 붙이지 않아도 된다.

참고: https://zorba91.tistory.com/