---
layout: post
title: "Spring Boot + Jacoco 를 활용한 코드 커버리지 관리"
subtitle: "2023-06-01-springboot-jacoco.md"
date: 2023-06-01 18:40:00
author: "전봉근"
header-img: "img/posts/android-bg.jpg"
comments: true
tags: [test, springboot, java, jacoco]
---

# Spring Boot + Jacoco 를 활용한 코드 커버리지 관리
해당 포스팅에 대한 코드는 [https://github.com/bkjeon1614/java-example-code](https://github.com/bkjeon1614/java-example-code) 를 참고     
   
## Jacoco 란?
Java 코드 커버리지를 측정하는 도구이다. (코드 커버리지란 소프트웨어 테스트를 논할 때 얼마나 테스트가 충분한가를 나타내는 지표중 하나이다. 또한 코드 커버리지는 소스코드 기반으로 수행하는 `화이트 박스 테스트`를 통하여 측정한다.)    
`Jacoco` 를 사용할 경우 장점은 아래와 같다.    
- 소프트웨어의 `안정성`을 높여준다.
- 사이드 이펙트가 발생할 확률이 높아진다.
- 간결하고 재사용성이 좋은 코드를 작성할 수 있게 해준다.
 

## 코드 커버리지 측정기준
코드 커버리지의 측정기준은 구문(Statement), 조건(Condition), 결정(Decision) 의 구조로 이루어져 있다.    
- 구문(Statement)
  - 테스트에 의해 모든 문장들이 최소 한 번은 실행될 수 있는 입력 데이터를 선정하는 기준이다. Line 커버리지 라고도 불리며 코드가 한 줄 이상 실행됐을 경우 조건을 충족하게 된다.
  - `구문 커버리지(%) = (수행된 구문의 수 / 전체 구문의 수) x 100`
  - `코드 한 줄이 한 번이상 실행된다면 충족`     
    ```
    public void getTest(int i) {
        System.out.println("Start Line");   // 1번

        if (i > 0) {    // 2번
            System.out.println("Middle Line");  // 3번
        }

        System.out.println("End Line"); // 4번
    }
    ```     
    > 상기 코드에서 i = -1 이라고 가정하면 if문을 통과하지 못하기 때문에, 3번 라인의 코드는 실행되지 못하며 즉, 총 4개의 라인에서 1,2,4 번의 라인만 실행되므로 `구문 커버리지는 3 / 4 * 100 = 75(%)` 가 된다.      
- 조건(Condition)     
  - `전체 조건의 결과와 관계없이 각 개별 조건이 True/False 모두 갖도록 조합하는 것` 입니다. 결정 커버리지 보다 강력한 형태의 커버리지입니다. 조건 커버리지는 모든 결정 커버리지에 대한 보장을 주지는 않는다.
  - `조건 커버리지(%) = (수행된 조건의 수 / 전체 조건의 수) x 100`
  - 모든 조건식의 내부 조건이 true/false 를 가지게 되면 충족한다. (`내부조건`이란 조건식 각각의 조건이다.)     
    ```
    public void getTest(int i, int j) {
        System.out.println("Start Line");   // 1번

        if (i > 0 && j < 0) {    // 2번
            System.out.println("Middle Line");  // 3번
        }

        System.out.println("End Line"); // 4번
    }
    ```     
    > 위의 코드를 테스트한다고 가정해보자. 조건 커버리지를 만족하는 테스트 케이스로는 (x = 1, y = 1), (x = -1, y = -1) 이 있다. 이는 x > 0 내부 조건에 대해 true/false를 만족하고, y < 0 내부 조건에 대해 false/true를 만족한다. 그러나 테스트 케이스는 두 조건 모두 성립할 수 없으므로 if 문은 조건에 대해 false만 반환한다. 즉, 내부 조건 x > 0, y < 0에 대해서는 각각 true와 false 모두 나왔지만 if 조건문의 관점에서 보면 false에 해당하는 결과만 발생했다. 이는 조건 커버리지는 만족했을지 몰라도 if 문의 내부 코드(=3번) 을 실행하지 못하므로 라인 커버리지를 만족하지 못하고 또한 if 문의 false 에 해당하는 시나리오만 체크되었기 때문에 만약 조건 커버리지를 만족하도록 테스트를 작성할 경우, 구문 커비리지와 결정 커버리지를 만족하지 못하는 경우가 존재한다.       
- 결정(Decision)     
  - 브랜치(Branch) 커버리지라고 부르기도 한다.
  - `결정 커버리지(%) = (수행된 분기의 수 / 전체 분기의 수) x 100`
  - `모든 조건식이 true/false` 를 가지게 되면 충족한다.     
    ```
    public void getTest(int i, int j) {
        System.out.println("Start Line");   // 1번

        if (i > 0 && j < 0) {    // 2번
            System.out.println("Middle Line");  // 3번
        }

        System.out.println("End Line"); // 4번
    }    
    ```     
    > if 문 조건에 대해 true/false 모두 가질 수 있는 테스트 케이스로는 (x = 1, y = -1), (x = -1, y = 1) 이 있다. 해당 테스트 케이스에 모두 각각 true or false 이므로 결정 커버리지를 충족한다.


## Spring Boot 에 Jacoco 플러그인 추가
[build.gradle]    
```
plugins {
    ...
    id 'jacoco' // jacoco 플러그인 추가
}

...

test {
    finalizedBy 'jacocoTestReport'  // gradle test 실행시에 jacoco 실행

    ...
}

...

// jacoco start
jacocoTestReport {
    dependsOn test

    reports {
        xml.required = false    // xml 파일 저장 X
        csv.required = false    // csv 파일 저장 X
        html.outputLocation = file("jacoco")    // html 파일 저장 O
    }

    finalizedBy 'jacocoTestCoverageVerification'    // rule 수행
}

jacocoTestCoverageVerification {
    violationRules {
        rule {
            enabled = true // 활성화
            element = 'CLASS' // 클래스 단위로 커버리지 체크
            // includes = []

            // 라인 커버리지 제한을 80%로 설정
            limit {
                counter = 'LINE'
                value = 'COVEREDRATIO'
                minimum = 0.80
            }

            // 브랜치 커버리지 제한을 80%로 설정
            limit {
                counter = 'BRANCH'
                value = 'COVEREDRATIO'
                minimum = 0.80
            }

            // 빈 줄을 제외한 코드의 라인수를 최대 200라인으로 제한합니다.
            limit {
                counter = 'LINE'
                value = 'TOTALCOUNT'
                maximum = 200
            }

            excludes = []
        }

        // 여러 개의 rule 정의 가능
        rule {
        }
    }
}
// jacoco end
// ---------------- Test End

...
```


## jacocoTestCoverageVerification 상세설명 
상기 jacocoTestCoverageVerification Task 설정을 자세하게 알아보자.       
     
- enable: rule 의 활성여부를 나타낸다. default 는 true 이다.
- [element](https://docs.gradle.org/current/javadoc/org/gradle/testing/jacoco/tasks/rules/JacocoViolationRule.html#getElement--): 커버리지를 체크할 기준을 정할 수 있다.
  - BUNDLE (default): 패키지 번들(프로젝트 모든 파일을 합친 것)
  - CLASS: 클래스
  - METHOD: 메소드
  - PACKAGE: 패키지
  - SOURCEFILE: 소스파일
- [counter](https://docs.gradle.org/current/javadoc/org/gradle/testing/jacoco/tasks/rules/JacocoLimit.html#getCounter--)
  - LINE: 빈 줄을 제외한 실제 코드의 라인 수
  - BRANCH: 조건문 등의 분기 수
  - CLASS: 클래스 수
  - METHOD: 메소드 수
  - INSTRUCTION (default): [Java 바이트코드 명령 수](https://en.wikipedia.org/wiki/List_of_Java_bytecode_instructions)
  - COMPLEXITY: [복잡도](https://www.eclemma.org/jacoco/trunk/doc/counters.html)
- [value](https://docs.gradle.org/current/javadoc/org/gradle/testing/jacoco/tasks/rules/JacocoLimit.html#getValue--)
  - TOTALCOUNT: 전체 개수
  - MISSEDCOUNT: 커버되지 않은 개수
  - COVEREDCOUNT: 커버된 개수
  - MISSEDRATIO: 커버되지 않은 비율. 0부터 1 사이의 숫자로, 1이 100%입니다.
  - COVEREDRATIO (default): 커버된 비율. 0부터 1 사이의 숫자로, 1이 100%입니다.
     
- Jacoco 테스트에서 제외     
  ```
  ...

    jacocoTestCoverageVerification {
    violationRules {
        rule {
            element = 'CLASS'

            limit {
                counter = 'LINE'
                value = 'TOTALCOUNT'
                maximum = 8
            }

            // 커버리지 체크를 제외할 클래스들
            excludes = [
        //      '*.test.*',
        //      '*', 모든파일 제외
                'com/example/bkjeon/base/*.class'   // 패키지+클래스명 으로 해줘야한다!
            ]
        }
    }
    }

  ...
  ```


## 실행
./gradlew test 명령을 실행하면 /build/jacoco (default) 또는 지정한 경로에 폴더가 생성된걸 확인할 수 있다 해당 폴더안에 index.html 을 실행하면    
![springboot-jacoco](/img/posts/language/java/tool/springboot-jacoco.png)        
결과를 볼 수 있다.
    
   
## 참고
- https://ko.wikipedia.org/wiki/
- https://seller-lee.github.io/
- https://m.blog.naver.com/suresofttech/
- https://velog.io/@33bini/
- https://tecoble.techcourse.co.kr/
- https://techblog.woowahan.com/
