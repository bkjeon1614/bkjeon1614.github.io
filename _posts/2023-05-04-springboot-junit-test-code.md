---
layout: post
title: "Spring Boot + Junit5 를 활용한 테스트 코드 분리"
subtitle: "2023-05-04-springboot-junit-test-code.md"
date: 2023-05-04 18:40:00
author: "전봉근"
header-img: "img/posts/android-bg.jpg"
comments: true
tags: [tdd, test, springboot, java]
---

# Spring Boot + Junit5 를 활용한 테스트 코드 분리
해당 포스팅에 대한 코드는 [https://github.com/bkjeon1614/java-example-code](https://github.com/bkjeon1614/java-example-code) 를 참고


## 테스트 분리를 왜 하는가?
각각의 테스트 규칙에 따라 테스트 코드를 작성하게 될 경우 통합 테스트가 단위 테스트보다는 아무래도 전체적으로 하다보니 수행속도가 느릴 수 밖에 없다. 즉, 개발과정에서 통합 테스트를 지속적으로 수행하게 된다면 개발생산성이 매우 저하 될 것이다. 그러므로 테스트 분리에 따라 테스트 수행을 하는것이 더 효과적일 수 있다.


## Given - When - Then Pattern
Given - When - Then Pattern 은 테스트 코드를 접해본 개발자들은 한 번쯤은 보았을거라고 생각한다. 해당 패턴은 [Given(준비), When(실행), Then(검증)](https://martinfowler.com/bliki/GivenWhenThen.html) 이라고 한다.
- Given
  - 테스트를 위해 준비하는 과정이며 변수와 입력 값 등을 정의하며 Mock 객체를 정의하는 구문도 포함
- When
  - 실제 기능을 동작하여 테스르를 실행하는 과정
- Then
  - 테스트를 검증하는 과정 즉, 실행을 통해 리턴된 값을 검증


## 테스트 종류
- 단위 테스트(Unit Test)
  - 시스템에서 테스트 가능한 가장 작은 기능들을 테스트
  - 단위의 크기가 작을수록 단위의 복잡성이 낮아지기 때문에 동작을 표현하기가 용이함
  - `시스템 내부 구조나 구현 방법을 고려하여 개발자 관점에서 테스트하므로 내부 코드에 관련한 지식을 반드시 알고 있어야 하는 화이트 박스 테스트이며 TDD 와 함께 할 때 더 강력해진다.`
  - 메소드 상단의 `@Test` 로 실행   
    ```
    ...

    @DisplayName("Unit Test")
    @Test
    void getMainUnit() {
        // given
        Test test = new Test("test");

        // when
        test.move(1);

        // then
        assertThat(test.getPosition()).isEqualTo(1);
    } 

    ...
    ```
- 통합 테스트(Integration Test)
  - 단위 테스트보다 더 큰 동작을 달성하기 위해 여러 모듈들을 모아 이들이 의도대로 협력하는지 확인하는 테스트 (=블랙박스 테스트)
  - `개발자가 변경할 수 없는 부분 즉, 외부 라이브러리와 같은 기능들을 묶어 검증할 때 사용 또한 DB에 접근하거나 전체 코드와 다양한 환경이 제대로 작동하는지 확인하는데 필요한 모든 작업을 수행`
  - 단위 테스트보다 발견하기 어려운 버그를 찾을 수 있다. (Ex: CPU 의 코어 개수에 따라 잘 실행되는지에 대한 테스트를 할 수 있음)
  - 단, 단위 테스트보다 더 많은 코드를 테스트하기 때문에 신뢰성이 떨어질 수 있다. 또한 어디서 에러가 발생했는지 확인하기 쉽지 않아 유지보수하기에 힘들다.
  - 클래스 상단의 `@SpringBootTest` 로 실행   
    ```
    @SpringBootTest
    class MainControllerTest {

        @Test
        void getMain() {

        }

    }
    ```
- 인수 테스트(Acceptance Test)
  - `사용자 스토리에 맞춰 수행하는 테스트`
  - 통합 테스트와 단위 테스트와 달리 비즈니스 쪽에 초점을 둠
  - `누가, 언제 목적으로, 무엇을 하는가` 라는 시나리오가 정상적으로 동작하는지를 테스트 한다.
  - `RestAssured`, `MockMvc` 와 같은 도구를 활용하여 인수 테스트를 작성한다.
    ```
    ...

    @SpringBootTest(webEnvironment = WebEnvironment.RANDOM_PORT)
    @Transactional
    public class BoardControllerTest {

        private final static String VERSION_NAME = "api/v1";
        private final static String BASE_URI_PATH = "/" + VERSION_NAME + "/boards";

        @LocalServerPort
        private int port;

        ...

        @Test
        @DisplayName("게시글 상세 조회 테스트")
        void get_board_detail() {
             Header header = new Header("key", "val");

             // statusCode 가 200 인지, body 안의 statusCode 스코프가 200 값이 일치하는지 확인
             RestAssured
                   .given().port(port).log().all()
                       .header(header)
                   .when()
                       .get(BASE_URI_PATH + "/1")
                   .then().log().all()
                       .assertThat().statusCode(HttpStatus.OK.value())
                       .body("statusCode", Matchers.equalTo(200));
         }
    }
    ...
    ```


## Junit5 @Tag 기능 적용을 통한 테스트 분리
테스트는 통합 테스트인 integration test 와 단위 테스트인 unit test 로 분리하는 방식으로 진행

1. build.gradle 추가
   [build.gradle]     
   ```
   ...

   dependencies {
      ...

      // 의존성 추가 (인수 테스트를 하기 위한 Rest-Assured 사용)
      testImplementation group: 'io.rest-assured', name: 'rest-assured', version: '3.0.0'
      testImplementation 'org.mockito:mockito-core:3.9.0'

      ...
   }

   // ---------------- Test Start
   task integrationTest(type: Test) {
      useJUnitPlatform {
         includeTags 'integration'  // 통합 테스트
      }
   }

   task unitTest(type: Test) {
      useJUnitPlatform {
         includeTags 'unit'   // 단위 테스트
      }
   }

    task acceptanceTest(type: Test) {
        useJUnitPlatform {
            includeTags 'acceptance'  // 인수 테스트
        }
    }   
   // ---------------- Test End

   ...
   ```    
2. @Tag 를 이용하여 테스트 분리   
   [IntergrationTest.java]    
   ```
   package com.example.bkjeon.base.api.helper;

   import java.lang.annotation.ElementType;
   import java.lang.annotation.Retention;
   import java.lang.annotation.RetentionPolicy;
   import java.lang.annotation.Target;

   import org.junit.jupiter.api.Tag;
   import org.junit.jupiter.api.Tags;
   import org.junit.jupiter.api.Test;

   // 통합테스트를 (통합 + 단위 테스트로 전부 실행하게 하였다. 이게 정답이 아니니 상황에 맞게 알아서 태그를 세팅하면된다.)
   @Target(ElementType.METHOD)
   @Retention(RetentionPolicy.RUNTIME)
   @Tag("integration")
   @Test
   public @interface IntergrationTest {
   }
   ```     
   [UnitTest.java]     
   ```
   package com.example.bkjeon.base.api.helper;

   import java.lang.annotation.ElementType;
   import java.lang.annotation.Retention;
   import java.lang.annotation.RetentionPolicy;
   import java.lang.annotation.Target;

   import org.junit.jupiter.api.Tag;
   import org.junit.jupiter.api.Test;

   @Target(ElementType.METHOD)
   @Retention(RetentionPolicy.RUNTIME)
   @Tag("unit")
   @Test
   public @interface UnitTest {
   }
   ```       
   [AcceptanceTest.java]    
   ```
   package com.example.bkjeon.base.api.helper;

   import java.lang.annotation.ElementType;
   import java.lang.annotation.Retention;
   import java.lang.annotation.RetentionPolicy;
   import java.lang.annotation.Target;

   import org.junit.jupiter.api.Tag;
   import org.junit.jupiter.api.Test;

   @Target(ElementType.METHOD)
   @Retention(RetentionPolicy.RUNTIME)
   @Tag("acceptance")
   @Test
   public @interface AcceptanceTest {
   }
   ```    
3. 테스트 코드 작성     
   [BoardControllerTest.java]   
   ```
   ...
   @AcceptanceTest
   @DisplayName("[Acceptance] 게시글 상세 조회 테스트")
   void get_board_detail() {
       RestAssured
           .given().port(port).log().all()
               .header(header)
           .when()
               .get(BASE_URI_PATH + "/1")
           .then().log().all()
               .assertThat().statusCode(HttpStatus.OK.value())
               .body("statusCode", Matchers.equalTo(200));
   }

   @UnitTest
   @DisplayName("[Unit] 게시글 상세 조회")
   void get_service_board_detail() {
       // given
       Long boardNo = 1L;

       // when
       ApiResponse<BoardResponseDTO> board = boardService.getBoard(boardNo);

       // then
       assertThat(board.getData()).isNotNull();
   }   
   ```        
   [FeignExampleControllerTest.java]   
   ```
   ...
   @IntergrationTest
   @DisplayName("[Intergration] Feign 을 통한 User 목록 조회")
   void board_insert_test() {
       // given
       FeignUserRequest feignUserRequest = FeignUserRequest.builder().testCode("bkjeon").build();

       // when
       List<FeignUser> feignUserList = feignUserClient.getUserList(feignUserRequest);

       // then
       assertThat(feignUserList).isNotNull();
   }   
   ```     
4. 테스크 실행
   ```
   // integration test 
   $ ./gradlew gradle :base-api:integrationTest

   // unit test
   $ ./gradlew gradle :base-api:unitTest

   // acceptance test
   $ ./gradlew gradle :base-api:acceptanceTest
   ```


# 참고
- https://tecoble.techcourse.co.kr
- https://brunch.co.kr/@springboot/