---
layout: post
title: "SCDF(=Spring Cloud Data Flow) - 1편"
subtitle: "2022-12-29-java-spring-cloud-data-flow-1.md"
date: 2022-12-29 18:30:00
author: "전봉근"
header-img: "img/posts/android-bg.jpg"
comments: true
tags: [java, spring, springboot, springcloud, scdf]
---

# SCDF(=Spring Cloud Data Flow) - 1편
## SCDF 란
Spring Cloud Data Flow 는 Cloud Foundry 및 Kubernetes에서 스트리밍 및 일괄 데이터 처리 파이프라인을 구축하기 위한 마이크로서비스 기반 툴킷이다. [공식문서](https://docs.spring.io/spring-cloud-dataflow/docs/current/reference/htmlsingle/)      
      
우선 SCDF 로 검색해보면 아래 3가지의 단어를 많이 볼 수 있다. 내용은 아래와 같다.
- Data MicroService
  - 데이터를 Function 단위의 비즈니스 로직이 들어 있는 Application 으로 DataPipeLine 을 구성
- Orchestrator
  - Stream 을 구성하는 Application 과 Application 간의 Channel 을 관리(Ex: Kafka topic, group, partition..) 즉, Message MiddleWare 의 Channel 을 알아서 관리
- Deployer
  - 앱을 통해 만든 Stream 이나 Task 를 배포를 해줘야 동작한다. 즉, Target Platform 에 따라 Application 을 배포
         

## SCDF 특징
- 웹 대시보드, REST API, JAVA DSL, console shell 다양한 인터페이스로 제공
- 로컬, Cloud Foundry 및 Kubernetes 위에서 설치 및 운영 가능
- 스케줄링 기능은 Cloud Foundry 및 Kubernetes에서만 사용 가능, 일반 서버에서는 X
- SCDF의 상태 및 데이터 관리는 관계형 DB 에서 관리 및 다양한 DB 지원 (CDC 기술도 지원)

## SCDF 는 왜 사용하나?
- 빠른 개발과 빠른 수정
  - 익숙한 Spring Boot 로 Function 단위의 Application 을 만들어 유지보수가 용이하고 복잡하지 않는 구조로 Stream 을 생성
  - Application 과 Application 간의 Message 통신을 Message Middleware 에서 담당
- 무중단 배포
  - Cloud Platform 에서 제공하는 배포정책으로 Downtime 과 데이터 유실없이 Stream 을 배포
- 분산처리
  - Application 분산처리를 Cloudplatform 에 맞게 Intance 를 늘리고 MessageBroker 의 Partition 을 조정
- 확장성
  - 실행되고 있는 Stream 에 다른 Stream 을 연결하여 Stream 의 확장이 쉽다.


## Require Platform
- Docker
- SCDF Server (SCDF Dataflow Dashboard)
- Skipper (SCDF Deployer)
- DataBase (Metadata Save)     
![scdf-1](/img/posts/language/java/scdf/scdf-1.png)      
         
- MessageMiddleware (`Kafka, RabbitMQ`, Kafka Streams, Amazon Kinesis, Google Pub/Sub, Solace PubSub+, Azure Event Hubs)
  - Kafka, RabbitMQ 가 문서가 제일 많고 스타트업을 제공해준다.
  - 또한 처리할 내용들이 Message Middleware 계속 쌓이므로 Consumer 는 Application 이 죽었다 다시 살아나면 누락된 데이터까지 처리하는 이점이 있음
- Monitoring (Optional)
  - Monitoring Metric Data 를 수집하는 시계열 데이터베이스인 TSDB(=Time Series DB) 필요 (Prometheus, InfluxDB)
  - Dashboard (Grafana, Wavefront)
![scdf-2](/img/posts/language/java/scdf/scdf-2.png)     


## 설치 환경별 설명
### Local 환경
Local 환경에서는 docker-compose 파일을 다운받아서 DATAFLOW_VERSION, SKIPPER_VERSION 버전들을 명시하고 docker-compose up 명령을 통하면 패키지 설치가 가능하다. (꼭 이 방법이 아니래도 검색하여 나오는 방법으로 해도 된다.)    
```
$ wget -O docker-compose.yml https://raw.githubusercontent.com/spring-cloud-dataflow/v2.7.1/spring-cloud-dataflow-server/docker-compose.yml;
$ DATAFLOW_VERSION=2.7.1 SKIPPER_VERSION=2.6.1
$ docker-compose up
```
> Local 버전은 Cloud Platform 환경과 사용방법이 많이 틀려서 결국은 Local 버전을 사용하다가 Cloud Platform 으로 변경하게 된다. 즉, 처음부터 Cloud Platform 사용을 권장한다.

### Cloud Platform 환경
Cloud Platform 에서는 Helm 을 통해서 설치할 수 있다.   
```
// Helm Chart: https://bitnami.com/stack/spring-cloud-dataflow/helm
// Reference: https://dataflow.spring.io/docs/installation/kubernetes/helm
$ helm repo add bitnami https://charts.bitnami.com/bitnami
$ helm install my-release bitnami/spring-cloud-dataflow
```

## SCDF Dashboard
SCDF 는 크게 Stream 과 Task 로 크게 둘로 나뉘어진다.
- Stream
  - 마이크로한 앱을 통해 Data Pipeline 으로 구성하는 것
- Task
  - Spring Batch 로 만든 Batch 관리에 대한 기능을 제공
          
SCDF Dashboard 좌측 메뉴의 각 의미하는 내용을 설명한다.     
![scdf-3](/img/posts/language/java/scdf/scdf-3.png)   

### Applications
#### Application 목록
- Stream 이나 Task 에서 사용할 Application 을 등록, 삭제, 변경
- 등록된 Application 목록 확인
- 같은 이름의 버전이 다른 Application 을 등록하면 버전관리도 가능하다.       
![scdf-4](/img/posts/language/java/scdf/scdf-4.png)     
          
#### Application 등록
- Application 의 name 과 type 을 작성하고 Source Repository 를 지정하면 Application 을 등록할 수 있다. (그 외에도 다른 방법들이 있으며 어렵지 않는 내용이니 직접 해보자.)
![scdf-5](/img/posts/language/java/scdf/scdf-5.png)      

#### StarterApp (Stream or Task)
![scdf-20](/img/posts/language/java/scdf/scdf-20.png)      
- SCDF 에서는 Message Middleware 를 Kafka 와 RabbitMQ 를 사용할 경우 Starter App 을 제공(Source, Processor, Sink) 해주므로 잘 사용하면은 코딩없이 Stream 을 빠르게 만들 수 있다.   
    
### Streams
#### Stream 목록
- 기 등록된 Stream 목록을 확인하고 해당 Stream 을 Deploy, Undeploy, Destroy 를 할 수 있다.
- 간략하게 Stream 의 구성정보나 status 정보도 확인이 가능하다.     
![scdf-6](/img/posts/language/java/scdf/scdf-6.png)      
         
#### Stream Deploy 화면
- 아래 이미지는 Stream 을 실제 배포하기전에 Properties 를 구성한 화면이다. 해당 화면에서 Properties 를 작성하고 Deploy Stream 을 하면 실제로 Stream 이 배포된다. (원하는 Stream 을 클릭 후 상세 페이지에서 DEPLOY STREAM 을 클릭한 화면이다.)    
![scdf-7](/img/posts/language/java/scdf/scdf-7.png)      
           
#### Stream 등록화면
- 하기 이미지의 좌측을 보면 Application 에 이미 등록된 내용들을 확인할 수 있다. 해당 Application 을 드래그 앤 드롭해서 Stream 을 생성할 수 있다.   
> Stream 을 생성할 때, `Source` 와 `Sink` 는 필수 요소이고 Processor 는 Optional 이다.      
![scdf-8](/img/posts/language/java/scdf/scdf-8.png)      
      
### Runtime
#### Runtime 이란
- Stream 에서 실행하고 있는 Application 의 디테일한 정보를 조회할 수 있는 메뉴
![scdf-9](/img/posts/language/java/scdf/scdf-9.png)      
상기 화면에서 `VIEW DETAILS` 를 클릭하면 팝업에 정보가 표시되는데 내용은 container.restartCount, service.name, host.ip, pop.name 등의 Instance 에 대한 정보를 제공한다. (여기서 확인하기 보다는 Platform 의 Monitoring 툴을 통해서 확인하는걸 추천한다.)    
      
### Tasks / Jobs
#### Task 란
- Task 는 Spring Batch 로 만든 Batch 를 의미하는 것이며 또한 `Batch 를 만들 때 Spring Batch 에 Spring Cloud Task 의 Dependency 가 추가되어야 한다.` (Dependency 에 추가되지 않으면 동작 X)   
- 해당 화면은 등록된 Task 의 목록을 확인하는 화면이고 Status 정보 확인이 가능하다.      
![scdf-10](/img/posts/language/java/scdf/scdf-10.png)      

#### 등록된 Task 화면
![scdf-11](/img/posts/language/java/scdf/scdf-11.png)      
- 등록된 Task 에 대한 정보를 조회하고, Task 를 실행, 삭제할 수 있는 화면이다. 또한 Schdule 을 설정이 가능하다. (실제 Schduling 을 SCDF 에서 관리해 주는 건 아니고 Kubernetes 기준으로 Kubernetes 의 CronJob 으로 등록되어서 관리가 된다.)
- LAST EXECUTION 에서 Task 에 대한 실행내역 확인이 가능하다. (Arguments 나 Properties 에 대한 값을 확인할 수 있다.)        
      
#### Task 생성 화면
![scdf-12](/img/posts/language/java/scdf/scdf-12.png)      
- 좌측에 Task 로 등록된 Application 목록을 확인할 수 있고, 우측으로 드래그 앤 드롭을 하여 Task 를 생성할 수 있다.

#### Task Executions    
![scdf-13](/img/posts/language/java/scdf/scdf-13.png)      
- `Task Executions` 에서는 Task 실행에 대한 History 를 조회하는 화면이며 시작시간, 종료시간 등을 확인할 수 있으다.       
        
#### Jobs Executions         
![scdf-14](/img/posts/language/java/scdf/scdf-14.png)      
![scdf-15](/img/posts/language/java/scdf/scdf-15.png)      
- `Jobs Executions` 에서는 Batch 에서 실행된 Job 에 대한 실행 내용을 조회할 수 있다.    
         
#### Schedules
![scdf-16](/img/posts/language/java/scdf/scdf-16.png)      
![scdf-17](/img/posts/language/java/scdf/scdf-17.png)      
- 해당 페이지는 Task 를 Scheduling 하는 화면인데 한 가지 이슈가 존재한다고 한다. (해결되었다고 한다. 그래도 혹시 모르니 테스트로 확인해보자.)
  - CronExpression 이슈
    - SCDF 의 Server 와 Skipper 의 Timezone 이 한국시간이더라도 Cron Expression 은 Target Platform(k8s) 의 Timezone 을 적용
    - SCDF 의 Timezone 이 UTC+9 이고 K8S 의 Timezone 이 UTC 면 SCDF 에서 UTC 기준으로 CronExpression 을 적용      
      
#### Audit Records
![scdf-18](/img/posts/language/java/scdf/scdf-18.png)      
- 해당 화면은 Stream 이나 Task 에서 사용한 Application 을 등록, 삭제, 변경 이력에 대한 History 를 확인할 수 있는 화면이다.      
     
#### Tools
![scdf-19](/img/posts/language/java/scdf/scdf-19.png)      
- 해당 화면은 기존에 등록된 Stream 이나 Task 를 Import / Export 를 할 수 있는 기능이 있다. (JSON File 형태)
- 또한 Task 및 Job 에 대한 이력들을 Clean up 하는 기능도 있다.       


# 참고
- https://docs.spring.io/spring-cloud-dataflow/docs/current/reference/htmlsingle/
- https://www.devkuma.com/docs/spring-cloud-data-flow/
- https://www.youtube.com/@workerksug7409
- https://sgitario.github.io/introduction-spring-data-flow/
- https://www.confluent.io/blog/spring-for-apache-kafka-deep-dive-part-3-apache-kafka-and-spring-cloud-data-flow/