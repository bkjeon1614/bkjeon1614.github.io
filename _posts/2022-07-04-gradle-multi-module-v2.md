---
layout: post
title : "Gradle Multi Project (Gradle 7.1.1 기준 대응)"
subtitle : "2022-07-04-gradle-multi-module-v2.md"
date: 2021-07-04 23:00:00
author: "전봉근"
header-img: "img/posts/android-bg.jpg"
comments: true
tags: [java, gradle, springboot]
---

# Gradle Multi Project (Gradle 7.1.1 기준 대응)

## 필수참고
- https://bkjeon1614.tistory.com/38 포스팅을 먼저 참고하고오자. 해당 포스팅은 Gradle 업그레이드로 인한 Deprecate 대응이므로 일부 코드수정만 확인할 수 있다. ([참고 Github Repo](https://github.com/bkjeon1614/java-example-code/tree/develop/sample-multi-module))

## 시작하기전에
- 이전 포스팅에서 작성했던 Gradle 기능중 일부 삭제된 부분이 존재한다. 삭제된 내용은 하기 내용을 참고하자.
- Could not find method compile() for arguments 오류해결
  - `compile`, `runtime`, `testCompile`, `testRuntime` 은 Gradle 4.10 (2018. 8. 27) 이래로 Deprecate 되었으며 Gradle 7.0 (2021. 4. 9) 부터 삭제되었다. 그러므로 해당 명령수정이 필요하다.
    ```
    compile -> implementation
    runtime -> runtimeOnly
    testCompile -> testImplementation
    testRuntime -> testRuntimeOnly
    ```
> 위의 내용을 기반으로 코드를 수정해보자.

## 시작하기
- gradle (4.10.3 -> 7.1.1), java (8 -> 11), springboot (2.0.6-RELEASE -> 2.6.7) 버전 업그레이드  
  [build.gradle]  
  ```
  ...
  buildscript {
    ext {
      // spring boot version 변경
      springBootVersion = '2.6.7'
    }

    ...
    
    // java version 변경
    sourceCompatibility = 11
  }
  ```
- 생성자 인젝션으로 변경  
  [TestController.java]  
  ```
  // @Autowired 제거 (필드 인젝션 제거)
  @RestController
  @RequestMapping("test")
  @RequiredArgsConstructor
  public class TestController {

      private final ApiSampleService apiSampleService;

      ...
  }
  ```  
  [ApiSampleService.java]  
    ```
    // @Autowired 제거 (필드 인젝션 제거)
    @Service
    @RequiredArgsConstructor
    public class ApiSampleService {
      
        private final SampleService sampleService;

        ...
    }
    ```    
- 일부 오작동 코드들 수정
  - [커밋내용 참고](https://github.com/bkjeon1614/java-example-code/commit/7d214409c89a4a4a9be39a83f34d6827802a9004)
- gradle 버전 업그레이드 대응 (Deprecate 대응)  
  [build.gradle]  
  ```
  // 기존 지정해놓은 프로젝트 의존성 세팅값 제거 (해당 코드 모두 제거 -> :admin-api, :admin-web)
  project(':admin-api') {
      dependencies {
          compile project(':admin-common')
      }
  }  

  project(':admin-web') {
      dependencies {
          compile project(':admin-common')
      }
  }  
  ```  
  [admin-api/build.gradle]  
  ```
  ...
  // compile -> implementation 로 변경 (compile project(':admin-common') 제거)
  dependencies {
    implementation project(':admin-common') // implementation 로 변경
    ...
  ```
  [admin-web/build.gradle]  
  ```
  ...
  // compile -> implementation 로 변경 (compile project(':admin-common') 제거)
  dependencies {
    implementation project(':admin-common') // implementation 로 변경
    ...
  ```  

> 이렇게 되면 멀티모듈 적용이 완료된다.