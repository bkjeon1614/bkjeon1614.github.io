---
layout: post
title : "관리 도구로서의 Jenkins 를 통한 Spring Batch 운영"
subtitle : "2022-11-09-java-springbatch-jenkins.md"
date: 2022-11-09 18:50:00
author: "전봉근"
header-img: "img/posts/android-bg.jpg"
comments: true
tags: [spring, java, jenkins, spring batch]
---
 
# 관리 도구로서의 Jenkins 를 통한 Spring Batch 운영
Spring Batch 는 아직까지 확실한 표준 관리 도구가 없다고 한다. 그래서 보통 클라우드 서버리스, 따로 API 기반 호출, Spring Batch Admin(더 이상 개선하지 않으며 Spring Cloud Data Flow 로 전환하라고함), Spring Quartz, Jenkins, Teamcity 등의 방법을 사용한다고한다.   
    
여기서 Spring Cloud Data Flow 가 러닝커브도 높으며 아직 국내에서 많이 사용되는 사례가 적으므로 그 전까지 보편적으로 활용되는 Jenkins 에 대해서 알아보기로 한다.


## 1. 장점

### 1-1. 관리기능
관리 도구의 대표 기능들에 대해 기본으로 지원된다. (Ex: Dashboard, 이력관리, 로그인, 계정권한 등)   
또한 Job 별로 실행 이력등을 관리할 수 있으며 세부적으로 Log 를 확인할 수 있다. 또한 파이프라인 구성도 가능하다. (Job 별로 파이프라인을 구성하는데 용이하다 -> 유지보수에 좋다.)

### 1-2. 통합
여러 소프트웨어와 통합 환경이 잘 구축되어 있다. github 의 경우 Repository 연동 뿐만 아니라 OAuth 로그인도 연동이 가능하며, Job 실행별로 나눠진 로그들을 logentries 에 모아줄 수 있다. 그리고 Elastic Stack (Elasticsearch + Logstash + Kibana) 에 로그를 보낼수도 있다. 그 외에 Slack, Email 등의 알람도 지정이 가능하므로 매우 다양한 통합환경을 지원하고 있다.

### 1-3. 다양한 Job 실행 환경
1. 스케줄링 (Crontab) 을 지정하여 실행
2. 직접 Jenkins 상에서 실행
3. HTTP API 로 실행 (사용하지 않는 계정은 삭제하여 Token 도 막을 수 있다.)
4. Job Trigger (현재 Job 의 상태에 맞춰 지정된 배치를 후속 실행이 가능하도록 할 수 있다.)   
   
> 이 중 한개를 선택하는 것이 아니라, 생성된 Job 들은 모든 실행 방법을 사용할 수 있다.   
    
### 1-4. 풍부한 생태계
역사가 오래되다보니 참고할 수 있는 정보나 여러 플러그인들이 많다.
- [Data Parameter](https://plugins.jenkins.io/date-parameter/): Java 8의 LocalDate, LocalDateTime 문법으로 날짜 파라미터를 사용할 수 있게 해주는 플러그인
- [Monitoring](https://plugins.jenkins.io/monitoring/): 모니터링 플러그인
- [커뮤니티 - 젠킨스 코리아 페이스북 그룹](https://www.facebook.com/groups/jenkinskorea/?mibextid=HsNCOg)

### 1-5. 파이프라인
최대 장점이라고 볼 수 있는 `파이프라인`은 배치 관리 도구의 가장 큰 장점이다. 또한 파이프라인 덕분에 배치의 설계도 유용하게 구성할 수 있다.    
예를 들어 1개의 Job 에 N 개의 Step 을 구성하는데 아래와 같은 요구사항이 생길 경우가 있다.   
- Step2 만 수행하고 싶다
- Step2, Step3 만 수행하고 싶다    
이렇게 되면 Job 하나의 조건별로 복잡한 Step 구성이 될 수 밖에 없다. 그러므로 `설계 초기에 고려된 사항이 아니므로 조건별 실행에 대한 모든 예외 상황을 검증`해야 하지만 파이프라인을 사용한다면 해당 문제를 해결할 수 있다. 	
     
> Job 을 최소 단위로 사용하자 (파이프라인으로 Job 을 묶어주는 것이 가능하므로 `Step 이 아닌 Job 을 최소 단위로 사용할 수 있다.`)   

Job 과 Step 이 1:1 관계로, Step 만큼 나누어진 Job 들은 파이프라인이 Job 을 묶어주는 구조로 변경    
또한 파이프라인은 각각의 Job 들에 대해 실행 순서, 동시 실행 등 여러 실행 유형들에 대해서 파이프라인에서 지정할 수 있으므로 본인의 Job 역할에만 충실하며 또한 기존에 잘 동작되는 배치 코드 수정을 안해도 되어 요구사항에 대응이 가능하다.    
      
만약, 지정된 날짜의 데이터를 가공하는 배치였는데, 한달치를 실행해야 할 경우 쿼리수정 및 인덱스 검토 / Job Parameter 변경 및 추가 / 테스트 등이 필요하지만 파이프라인에서는 하기 코드와 같이 `반복문으로 날짜를 변경하면서 Job 실행이 가능` 하다.   
```
// jenkins 설정 내용중 pipeline 에서 script 를 사용할 수 있음
node {
	def month = params.month
	def startDate = params.startDate.toInteger()
	def endDate = params.endDate.toInteger()

	for (i=startDate; i<=endDate; i++) {
		...
		build job: "exampleBatch",
		parameters: [
			string(name: 'date', value: "$month - $number"),
		]
	}
}
```    
또한 상기 파이프라인 관리 말고도 다양하게 존재한다. (Web GUI, Web Script, 지정된 위치의 Groovy File 등)

### 1-6. Slave Node 환경
한 대의 jenkins 서버로 운영하면서 자원을 많이 사용하는 Batch 를 수행하게 된다면 자원부족으로 다른 Batch 들을 수행하기 어렵다. 그로 인하여 jenkins 에서는 Master - Slave Node 환경을 지원한다. (원래는 분산 빌드 용도)    
서버만 SSH 로 열려 있다면 별도의 Jenkins 설치 없이 Slave 로 활용할 수 있다. (별도의 실행 환경 구성은 Master Jenkins 가 처리) 여분의 서버만 있으면 Batch Job 들을 격리해서 실행할 수 있고, 단일 작업을 쪼개서 실행하는 것도 가능하다. 예를 들어 Slave Node 서버가 3개면 특정일자를 3등분 하여 처리할 수 있다.  
- 1번 라벨에서는 0 ~ 6시
- 2번 라벨에서는 6시 ~ 12시
- 3번 라벨에서는 12시 ~ 18시   
등과 같이 처리가 가능하며 더 유용하게 사용하려면 [Node and Label parameter](https://plugins.jenkins.io/nodelabelparameter/) 가 필수이다.

### 1-7. 공통 설정 관리
하나의 애플리케이션에 job 이 여러개 설정되는 경우 하기 코드와 같이 공통 설정이 생기게 된다.  
```
java -jar \
-XX: +UseG1GC \
-DSpring.profiles.active=dev \
Application.jar \
--job.name={job name} \
jobParameter1=jobParameterValue1 \
jobParameter2=jobParameterValue2
```    
이 경우 jenkins 의 Global Properties 를 사용하여 공통설정들을 쉽게 주입이 가능하다.      
![springbatch-58](/img/posts/language/java/springbatch/springbatch-58.png)     


## 2. 단점

### 2-1. 소수의 배치를 관리하기에는 과함
몇 개 안되는 Batch 를 사용하는 경우에는 Jenkins 서버 자체로 증설해야되고 그에 따른 공수가 들어가게 된다. 차라리 기존 어드민에서 단순히 배치 코드만 추가하는게 더 유용한 것 같다.

### 2-2. 불편한 Web Editor
대부분 CI 도구에서는 자동 완성, 문법 지원 등을 지원하지만 Jenkins 는 그런것들이 안되며 문법 체크와 템플릿 외에는 지원하지 않는다.    

### 2-3. 신뢰할 수 없는 플러그인
특정 플러그인을 검색어로 조회할 경우 무수히 많은 플러그인이 조회된다. 이 중 어떤걸 써야될지도 애매하며 플러그인 검수가 철저하지 않아 특히 Jenkins 버전업시 사용 못하는 경우가 많다. 또한 플러그인 자체도 만들기 쉬운편이 아니다.

### 2-4. 파일 기반의 설정 정보
Jenkins 는 설정 정보와 Job 실행 이력 등과 같은 정보들이 모두 `파일로 관리` 된다. 해당 정보들이 궁금하면 Jenkins 제공하는 API 또는 직접 서버로 접속하여 서버내의 XML 파일로만 확인이 가능하다. 그러므로 백업 & 이중화가 어려우며 만약 하게 된다면 rsync 등과 같은 툴로 백업 서버에 지속적인 동기화를 하거나 주기적으로 외부파일서버에 디렉토리 통으로 업로드 하는 방법으로 관리해야 한다. (업로드 실패시 알림 등 별도의 공수가 계속 필요함, 또한 Jenkins 를 제외한 다른 도구들은 보편적으로 DB로 관리됨)

> 그 외에 Teamcity 나 Spring Cloud Data Flow 등이 있지만 여러 가지 이유(CI/CD 로만 발전, 유료, 특정 환경에서만 사용, 국내 레퍼런스 부족 등의 단점) 로 아직 사용하기엔 이른 것 같아 지금은 추천하지 않는다.


## 참고
- https://jojoldu.tistory.com/
- https://www.youtube.com/c/%EC%9A%B0%EC%95%84%ED%95%9CTech