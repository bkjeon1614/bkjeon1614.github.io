---
layout: post
title : "Spring Batch"
subtitle : "2022-09-07-java-springbatch.md"
date: 2022-09-07 13:50:00
author: "전봉근"
header-img: "img/posts/android-bg.jpg"
comments: true
tags: [spring, java, jvm, spring batch]
---

# Spring Batch
Spring Batch 를 사용하면서 내부적으로 어떻게 동작하는지 어떤 특징이 있는지 등에 대해서 한 번 정리가 다시할 필요성을 느껴서 해당 포스팅을 작성하게 되었다.


## Batch?
### 정의
개발자가 정의한 작업을 한번에 일괄 처리하는 애플리케이션이며 즉, 데이터를 실시간으로 처리하는게 아니라, 일괄적으로 모아서 처리하는 작업을 의미한다. (Ex: 하루동안 쌓인 데이터를 배치작업을 통해 특정 시간에 한꺼번에 처리하는 경우)

### 예시
- 상품 키워드 검색 수에 대한 일/주/월 키워드 검색순위 데이터 집계
- 매출 데이터를 사용한 일매출 데이터 집계 (실시간 집계 쿼리로 해결하면 조회 시간이나 서버 부하가 많으므로 매일 새벽 전날에 매출 집계 데이터를 미리 생성)
- 내부 및 외부 시스템에서 우리에게 맞는 데이터로 가공하여 수신이 주기적으로 필요한 경우

### Batch 가 필요한 경우
- 일정한 주기로 실행이 필요할 때
- 실시간 처리에 어려운 대량의 데이터를 처리할 때 (이와 같은 작업을 하나의 애플리케이션에서 처리하게 된다면 심각한 성능 저하를 유발할 수 있으므로 배치 애플리케이션을 구현하여 작업 리소스를 분산한다.)

### Batch 애플리케이션의 조건
- 대용량 데이터: 배치 어플리케이션은 `대량의 데이터`를 가져오거나, 전달하거나, 계산하는 등의 처리를 할 수 ​​있어야 함
- 자동화: 배치 어플리케이션은 심각한 문제 해결을 제외하고는 `사용자 개입 없이` 실행되어야 함
- 견고성: 배치 어플리케이션은 잘못된 데이터를 충돌/중단 없이 처리할 수 있어야 함
- 신뢰성: 배치 어플리케이션은 무엇이 잘못되었는지를 추적할 수 있어야 합니다. (로깅, 알림)
- 성능: 배치 어플리케이션은 `지정한 시간 안에 처리를 완료`하거나 동시에 실행되는 `다른 어플리케이션을 방해하지 않도록` 수행되어야 함

### Reader & Writer
Spring Batch 에서 지원하는 Reader & Writer 아래와 같다. (Spring Batch 4.0(Spring Boot 2.0) 기준)
```
[DataSource]     [기술]                 [설명]
Database          JDBC                 페이징, 커서, 일괄 업데이트 등 사용 가능
Database          Hibernate            페이징, 커서 사용 가능
Database          JPA                  페이징 사용 가능 (상기 기준 버전에서는 커서 X)
File              Flat file            지정한 구분자로 파싱 지원
FIle              XML                  XML 파싱 지원
```


## Scheduling?
### 정의
일정한 시간간격 또는 일정한 시각(설정가능)에 특정 로직을 돌리기 위해서 사용 즉, 매 시간/지정한 시간에 지정한 동작을 수행함

### Spring Batch VS Quartz
- [Spring Batch](https://docs.spring.io/spring-batch/docs/4.2.x/reference/html/index-single.html#domainLanguageOfBatch) 에서는 Scheduling 기능을 제공하지 않는다. (보통 Quartz + Batch, Jenkins + Batch 와 같은 방법으로 조합하여 사용)
- Scheduling 프레임워크는 Batch 의 특성대로 잘 실행될 수 있게 도와주는 역할은 한다.


## Spring Batch?
Spring Batch 프로젝트는 Accenture와 Spring Source의 공동 작업으로 2007년에 탄생하였다. 즉, Accenture의 배치 노하우 & 기술력과 Spring 프레임워크가 합쳐져 만들어진 것이 `Spring Batch` 이다. 또한 Spring 의 특성을 그대로 가져왔으므로 `DI, AOP, 서비스 추상화` 등의 Spring 프레임워크의 대표적인 3대 요소를 모두 사용할 수 있다.

### Batch 의 Domain 용어
![springbatch-1](/img/posts/language/java/springbatch-1.png)    
이미지 출처: https://docs.spring.io/
- 각 `Job` 들을 실행하기 위한 `JobLauncher` 구현
- 하나의 `Job` 은 여러개의 `Step` 으로 구성될 수 있다.
- 각 `Step` 은 `ItemReader, ItemProcessor, ItemWriter` 를 딱 한개씩 가지고 있다.
- `JobLauncher, Job, Step` 즉, 현재 실행중인 프로세스의 메타정보는 `JobRepository` 에 저장된다.

### Job
`Job` 은 단순히 `Step` 인스턴스의 컨테이너 개념이다. 논리적으로 한 플로우에 속한 여러 `Step` 을 결합하고, 재시작 같은 속성을 전역으로 구성할 수 있다.
```
Job -> JobInstance -> JobExecution
```
- 간단한 Job 의 이름
- `Step` 인스턴스의 정의 및 순서
- `Job` 의 재시작 가능 여부

### JobInstance
`JobInstance` 는 논리적으로 Job 을 실행하는것이다. JobParameters 를 이용하게되면 개발자는 효율적으로 `JobInstance` 를 정의할 수 있으며, 거기 사용될 파라미터도 제어할 수 있다.
(Job 은 하나지만 JobParameter 가 두개가 있는 경우 => JobInstance = Job + 식별용 JobParameters) 또한 `JobInstance` ID 에 관여하지 않는 파라미터를 사용할 수도 있다.

### JobExecution
`JobExecution` 은 작업을 실행하려는 단일 시도이며 실패했던 `JobInstance` 에 대해 새로운 실행을 하면 새로운 `JobExecution` 이 생성
- JobExecution Properties (필요한 부분만 발췌)
  - Status: 실행 상태를 나타내는 `BatchStatus` 개체이며 실행중이면 STARTED, 실패면 FAILED, 완료되면 COMPLETED
  - exitStatus: `ExitStatus` 실행 결과를 나타낸다. 호출자에게 반환되는 종료코드가 포함되어 있으므로 가장 중요!! (작업이 완료되지 않으면 해당 필드는 비어있음)
  - executionContext: 실행간에 유지되어야 하는 사용자 데이터가 포함된 속성집합소

### Step
![springbatch-2](/img/posts/language/java/springbatch-2.png)    
이미지 출처: https://docs.spring.io/
- batch 작업의 독립적이고 순차적인 단계를 캡슐화하는 도메인 객체
- 모든 `Job` 은 하나 or 여러개의 `Step` 으로 구성
- `Step` 의 내용은 개발자의 재량이므로 원하는 만큼 간단하거나 복잡하게 구현이 가능
- StepExecution (필요한 부분만 발췌)
  - Status: 실행 상태를 나타내는 `BatchStatus` 개체이며 실행중이면 STARTED, 실패면 FAILED, 완료되면 COMPLETED
  - executionContext: 실행간에 유지되어야 하는 사용자 데이터가 포함된 속성집합소
  - `ReadCount, WriteCount, CommitCount, RollbackCount, FilterCount, readSkipCount, processSkipCount 등` 다양한 실행에 대한 정보를 담고 있다.

### JobRepository
`JobRepository` 는 위에서 언급한 개념에 대한 지속성 메커니즘이다. `JobLauncher, Job, Step` 구현에 대한 CRUD 작업을 제공

### JobLauncher
`JobLauncher` 는 `Job` 을 시작하기 위한 간단한 인터페이스며 구현 시 `JobRepository` 에서 유효한 `JobExecution` 을 획득하고 `Job` 을 실행

### Spring Batch 의 메타데이터
`Spring Batch` 에서는 `메타 데이터 테이블` 들이 필요하다. 해당 테이블들은 상기 설명되었던 내용들을 담고 있다. (실행된 Job 목록, 실패한 Job 또는 Batch Parameter, 다시 실행한다면 어디서 부터 시작하면 될지, 어떤 Job 의 어떤 Step 있고 Step 들 중 성공/실패한 것들이 어떤것들이 있는지 등..) `즉, 이러한 메타데이터를 직접 개발자가 구현하지 않고 제공받는것을 이용하여 구현할 수 있기 떄문에 개발자는 비즈니스로직에 집중할 수 있다.`
- Meta-Data Schema   
  ![springbatch-3](/img/posts/language/java/springbatch-3.png)    
  > 해당 테이블들이 있어야 정상동작하므로 개발자가 직접 생성해야한다. 본인 IDE에서 파일검색으로 `schema-` 를 검색해보면 메타 테이블들의 스키마가 DBMS에 맞춰 각각 존재하는것을 참고하여 create table 해주면 된다. 

### Item
- Item: 작업에 사용하는 데이터
- Chunk: 데이터 덩어리로 작업할 때 각 커밋 사이에 처리되는 row 수 즉, Chunk 지향 처리는 한번에 하나씩 데이터를 읽어, Chunk 라는 덩어리를 만들고, Chunk 단위로 트랜잭션을 다루는 것을 의미
- ItemReader: `Step` 에서 `한 항목씩 검색`한다. 제공할 수 있는 항목이 소진되면 null 을 반환
- ItemWriter: Spring Batch 에서 사용하는 `출력` 기능
- ItemProcessor: `비즈니스 처리를 담당` 하며 항목이 유효하지 않다고 판단하는 경우 null 을 반환



## 참고
- https://docs.spring.io/
- https://godekdls.github.io/
- https://jojoldu.tistory.com/
- https://www.youtube.com/c/%EC%9A%B0%EC%95%84%ED%95%9CTech
- https://devboi.tistory.com/