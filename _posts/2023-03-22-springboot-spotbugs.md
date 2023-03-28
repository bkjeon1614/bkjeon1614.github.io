---
layout: post
title: "Spring Boot + Gradle 을 활용한 정적 코드 분석 도구 Spotbugs 적용"
subtitle: "2023-03-22-springboot-spotbugs.md"
date: 2023-03-22 18:40:00
author: "전봉근"
header-img: "img/posts/android-bg.jpg"
comments: true
tags: [devops, springboot, static analysis]
---

# Spring Boot + Gradle 을 활용한 정적 코드 분석 도구 Spotbugs 적용
해당 내용은 [Spotbugs 4.7.3](https://spotbugs.github.io/) 기준으로 작성하였고, [Spotbugs Gradle Plugin 5.0.14](https://github.com/spotbugs/spotbugs-gradle-plugin/blob/master/README.md) 를 활용하였다.    
    

## 정적 코드 분석 도구란
`정적 분석 도구` 는 코드를 검사하여 메모리 누수 또는 버퍼 오버플로우 등 일반적으로 알려진 오류 및 취약점을 파악하고 코딩 표준 적용이 가능하다. 즉, `코드의 정확도, 스타일, 성능 등 코드 품질에 관련된 패턴을 분석해서 알려준다.` 또한 [GNU Lesser General Public License](https://www.gnu.org/licenses/lgpl-3.0.html) 조건에 따라 배포되는 자유 소프트웨어이다. 도구중에서 해당 내용에선 Intellij 에서도 사용이 가능한 `Spotbugs` 를 다룰것이다.  


## 정적분석과 동적분석
코드분석이 꼭 정적분석만 존재하는것이 아니다. 동적분석도 존재하며 목적은 취약점을 찾는다는거에 동일하다. 차이점은 분석시 개발 싸이클에서 어느 시점에서 수행되는지이며, `동적 분석은 애플리케이션 테스팅 및 수행 단계에서 취약점을 발견`하고 `정적 분석은 실행 시점 이전에 수행되므로 개발 초기부터 사용할 수 있다.`


## Spotbugs 란
[Spotbugs](https://spotbugs.github.io/) 는 자바 바이트 코드를 분석하여 버그 패턴을 발견하는 정적분석 공개 소프트웨어다. (상기 언급된 내용처럼 GNU LGPL 라이센스를 적용)    
원래 기존에 존재했던 정적분석도구인 [Findbugs](https://findbugs.sourceforge.net/) 를 계승한 프로젝트이며 미국의 메릴랜드 대학에서 2004 년에 개발하였고 Java 프로그램에서 발생가능한 100여개의 잠재적인 에러에 대해 등급별로 구분하여 탐지하고 그 결과를 XML 로 저장할 수 있도록 지원한다. 또한 룰셋을 커스터마이징을 통해 적용이 가능하다.


## Spotbugs 에서 제공하는 탐지유형 (4.7.3 기준)
- Bad practice: 클래스 명명규칙, null 처리 실수 등 개발자의 나쁜 습관을 탐지
- Correctness: 잘못된 상수, 무의미한 메소드 호출 등 문제의 소지가 있는 코드를 탐지
- Experimental: 메소드에서 생성된 stream이나 리소스가 해제하지 못한 코드를 탐지
- Internationalization: Default 인코딩을 지정하지 않은 경우 등 지역특성을 고려하지 않은 코드 탐지
- Malicious code vulnerability: 보안 코드에 취약한 가변적인 배열이나 콜렉션, Hashtable 탐지
- Multithreaded correctness: 멀티쓰레드에 안전하지 않은 객체 사용 등을 탐지
- Bogus random noise: 소프트웨어에서 실제 버그를 찾는 것이 아닌 데이터 마이닝 실험에서 컨트롤로 유용하게 사용하기 위한 것
- Performance: 미사용 필드, 비효율적 객체생성 등 성능에 영향을 주는 코드를 탐지
- Security: CSS, DB 패스워드 누락 등 보안에 취약한 코드를 탐지
- Dodgy code: int의 곱셈결과를 long으로 변환하는 등 부정확하거나 오류를 발생시킬 수 있는 코드를 탐지


## 사용이유
개발자마다 서로 다른 방식의 코딩으로 인하여 오류를 발생하거나 보안을 위협할 수 있는 코드를 방치하게 된다면 이후 프로그램의 고질적인 문제로 남아있다거나 유지보수또한 어려움을 겪에되는 상황이 발생한다. 그러므로 코드가 규칙에 맞게 잘 작성되었는지를 점검하고 수정해야 한다. 하지만 매번 직접 코드를 사람이 직접 하나하나 검수하기는 힘드므로 정적 코드 분석 도구를 활용하는것이다. 종류로는 CheckStyle, PMD, SpotBugs(FindBugs 를 잇는 분석도구) 등이 있다.     
    
`Spotbugs` 를 선택한 이유는 [Coverity](https://scan.coverity.com/), [SonarQube](https://www.sonarsource.com/products/sonarqube/) 와 같은 다른 정적 분석 도구에 추가로 붙여서 사용이 가능하고 (Ex: Spotbugs + SonarQube 를 사용하여 검토 후 SonarQube Dashboard 에서 종합적으로 확인이 가능), 새로운 패턴을 추가하면 다른 도구에서도 쉽게 해당 패턴을 적용하여 결과를 볼 수 있다.    
     
또한 각종 빌드 도구(Maven, Gradle 등) 와, 개발 도구(Intellij, Eclipse 등) 쉽게 연동이 가능하고 다양한 운영체제(linux, windows, macOSX) 운영체제를 지원하며 또한 바이트코드를 분석하므로 소스코드없이 jar 파일만으로도 분석이 가능하다.      


## Spotbugs 장점 및 단점
- 실제 결함을 잘 찾아준다
- 찾은 결함이 엉뚱한 결함일 확률이 낮다
- 바이트 코드를 읽음으로 인하여 속도가 빠르다 (컴파일된 클래스 파일에서 바이트 코드를 읽어서 사용해야하므로 빌드과정이 필수)


## Spotbugs 실습 (with. Spring Boot + Gradle)
### Intellij
1. 플러그인 설치   
   - Preferences.. > Plugins > SpotBugs 검색 후 Install      
     ![springboot-spotbugs-1](/img/posts/language/java/tool/springboot-spotbugs-1.png)       
2. 검사
   - 원하는 패키지 우측 클릭 후 `Analyze Package(S) Files` 클릭    
     ![springboot-spotbugs-2](/img/posts/language/java/tool/springboot-spotbugs-2.png)       

> 미리 Intellij 플러그인을 통하여 검증을해야 빌드 단계에서 실패가 날 확률을 줄일 수 있다.

### Spring Boot + Gradle
1. 의존성추가    
   [build.gradle]   
   ```
   plugins {
       ...
       id "com.github.spotbugs" version "5.0.14"
   }
   
   ...

   // ---------------- Static Application Security Testing (SAST) Start
   // https://github.com/spotbugs/spotbugs-gradle-plugin/blob/master/README.md
   tasks.withType(com.github.spotbugs.snom.SpotBugsTask) {
      spotbugs {
         ignoreFailures = true   // 오류무시여부
         showStackTraces = true  // 스택트래이스 표시 여부
         showProgress = true  // 프로그레스바 표시 여부
      }

      reports {
         xml.required.set(false) // XML 파일 생성여부
         html {
            required = true
            outputLocation = file("src/main/resources/spotbugs.html")   // Report File 추출경로
         }
      }
   }
   // ---------------- Static Application Security Testing (SAST) End

   ...

   task buildStep {
      ...
      // SpringBoot Application 에서 화면을 표시하기위해 정적경로에 추출한 HTML 파일을 복사
      copy {
            from projectDir.toString() + "/src/main/resources/"
            include "spotbugs.html"
            into serverFolderResourcesDir + "static/"         
      }
   }
   ```     
2. 동작여부 테스트     
   ```
   $ ./gradlew spotbugsMain
   ```   
3. resources/static 경로에서 spotbugs.html 파일이 생성된걸 확인할 수 있다.   
4. http://localhost:9090/spotbugs.html 에서 확인    
   ![springboot-spotbugs-3](/img/posts/language/java/tool/springboot-spotbugs-3.png)       


### 작업순서
1. Intellij Spotbugs Plugin 을 통하여 코드를 수정   
2. repoting 된 파일을 참고하여 코드 커버리지 확인 (실제 빌드시 Spotbugs Ignore 설정을 통하여 주기적으로 웹 페이지에서 Spotbugs 의 Report 를 주기적으로 확인할 수 있게함, 실제 빌드시 확인 경로는 build/reports/spotbugs/main.html 이며 해당 글에서는 html 파일로 추출하나 SonarQube 와 연동시에는 XML 파일로도 사용하기도 한다.)


# 참고
- https://www.jetbrains.com/
- https://engineering.linecorp.com/
- https://keichee.tistory.com/
- https://camelsource.tistory.com/
- 공개SW를 활용한 소프트웨어 개발보안 점검가이드 일부 참조
- IGLOO_PLUS 일부 참조
- ChatGPT