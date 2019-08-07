---
layout: post
title : "NodeJS VS Vert.x"
subtitle : "2019-08-07-nodejs-vertx-difference.md"
date: 2019-08-07 17:30:00
author: "전봉근"
header-img: "img/posts/android-bg.jpg"
comments: true
tags: [javascript, nodejs, vert.x, java]
---


## NodeJS vs Vert.x
----------------------------------------------------------------
해당 문서는 NodeJS와 Vert.x에 대한 대략적인 특징 및 성능 비교에 대해 간단명료하게 작성하였다. 해당 내용에 대해 더 자세한 정보를 보고싶다면 아래 링크에서 확인하자.
- [NodeJS](https://nodejs.org/ko)
- [Vert.x](http://vertx.io)


### NodeJS
----------------------------------------------------------------
NodeJS는 Google의 Chrome V8 Javascript 엔진 기반인 고성능의 비동기 IO를 지원하는 네트워크 서버이다. 
- Event - Driven 방식
- Async / Non Blocking IO 기반
- Single Thread Model

![javascript-nodejs-vertx-1](/img/posts/javascript/nodejs/javascript-nodejs-vertx-1.png)  
[(이미지 출처)](https://bcho.tistory.com/)

단점
  - Singgle Thread Model 기반이므로 하나의 request를 처리할 때 CPU를 많이 사용하면 다른 요청 처리가 지연되며 전체적인 응답시간 저하로 연결된다.
  - 에러가 나면 대부분 서버가 죽어버리기 때문에 운영 관점에서 트러블 슈팅등이 어렵다. (Ex: V8 Engine Memory leak)
    -> 보편적인 방법으로는 pm2와 같은 프로세스 관리자 툴로 restart 시키며 원인에 대해서는 특정 함수(=process.on('uncaughtException'))나 라이브러리를 사용하여 로그를 남긴다.


### Vert.x
----------------------------------------------------------------
NodeJS로부터 영향을 받은 프로젝트이며 NodeJS처럼 Event - Driven 방식인 비동기 소켓서버 프레임워크이다. 
- JVM 위에서 동작
- Non Blocking 방식 (Netty 기반)
- 다양한 언어 지원
- 멀티 스레드 지원
- Event Bus: 각각의 Verticle 들이 어떤 언어로 되어있든 상관하지 않고 통신이 가능하도록 도와준다.
- 클러스터링 지원: 한 하드웨어에 여러가지 vert.x 를 띄울 수 있음. (HazelCast 기반으로 데이터 공유가 가능하다.)
- 멀티 인스턴스 지원을 통한 성능 증대: 동시에 여러개의 Verticle을 실행할 수 있다. 여러 개의 Thread를 띄우더라도, 각 Thread는 독립적으로 동작하고 각 Thread간에 객체 공유나 자원의 공유가 없이 전혀 다르게 독립적으로 동작한다.

> Netty: 유지 관리가 용이한 고성능 프로토콜 서버와 클라이언트를 신속하게 개발하기 위한 비동기식 이벤트 기반 네트워크 애플리케이션 프레임워크

> Verticle: Vert.x의 하나의 애플리케이션(=Servlet과 같음)이며 독립된 Class Loader에서 독립된 Object로 존재하므로 Multi threading 문제가 발생하지 않음.

> Hazelcast: 분산 환경에서 데이터 공유를 위한 In-Memory Data Grid 오픈소스 솔루션

![javascript-nodejs-vertx-2](/img/posts/javascript/nodejs/javascript-nodejs-vertx-2.png)  
[(이미지 출처)](https://www.javacodegeeks.com/2012/07/osgi-case-study-modular-vertx.html)


### NodeJS와 Vert.x 비교
----------------------------------------------------------------
|                      | NodeJS               | Vert.x          |
| :------------------- | -------------------: |:---------------:|
| 사용가능 언어 | Javascript | Python,JavaScript,Java,Groovy,Scala |
| 지원 모듈 | 40,000개 이상 | 100개 이하 |
| 안전성 | Netty, HazleCast 등 안정된 엔진위에 개발됨 | V8 엔진 자체가 불안함 |
| 클러스터링 | 한 하드웨어에 여러개 NodeJS를 띄울 수 있음. NodeJS 인스턴스간 상태 Share 불가 | 한 하드웨어에 여러개의 vertx를 띄울 수 있음. node간의 상태 공유 메세징 가능 |
| 레퍼런스 | 매우 풍부 | 매우 적음(ex:공식 서적 2개) |
| 성능 | 열세 (한 node 인스턴스당 CPU 코어 1개 이상 사용 불가, 멀티스레딩 안됨) | 우세 (JVM기반으로 하나의 vert.x 인스턴스에 여러개의 Verticle 인스턴스를 띄워서 CPU의 사용 극대화) |
| 에러 처리 | Context 정보 없이 죽어서 추적이 어려움 | Stack을 출력하고 죽어서 추적에 용이함 |
| 비동기 Non-Blocking IO | Thread Pool을 이용한 Emulation | OS 수준의 Non-Blocking IO 사용(IO 처리에 유리) |


### 성능비교 [(이미지 출처)](https://vertxproject.wordpress.com/2012/05/09/vert-x-vs-node-js-simple-http-benchmarks/)
----------------------------------------------------------------

#### 200/OK 응답만 주었을 때의 성능 비교
![javascript-nodejs-vertx-3](/img/posts/javascript/nodejs/javascript-nodejs-vertx-3.png)

> 위의 그래프를 보면 Node.js 보다 Vert.x-Javascript의 성능이 좋다고 표시되었으나 이것은 Vert.x 제작자가 밝힌 성능이며 엄격한 환경에서 실시한 테스트가 아니라고 하여 상대적인 성능 격차에만 주목하는것이 좋다고 한다. 


### 사용 방향성
----------------------------------------------------------------
개인적으론 Vert.x와 Node.js중 선택을하자면 Node를 사용하는게 더 좋다고 생각한다.   
왜냐하면 vert.x는 2012년 5월에 첫 버전이 나왔지만 2009년에 첫 버전이 나온 Nodejs에 비하면 역사가 짧은 부분도 있으며 레퍼런스 또한 매우 부족한 상황이라  모듈인 경우에도 자체적으로 개발해야되는 상황이 많다.   
그러므로 서비스 초기엔 성능적인 이점은 조금 양보하여 안정적이며 레퍼런스 및 모듈들이 방대한 Node.js로 운영하다 향후에 Vert.x 진영의 발전 속도나 기술력을 고려하여 도입에 대한 여부를 결정하는것이 좋을 것 같다. ( 이미 왠만한 기업들도 NodeJS 사용을 많이하여 검증이 되었다고 생각한다. )

`위의 내용 중 NodeJS 트러블 슈팅이 어렵다고 되어있지만 현재 NodeJS 진영에서는 V8 Engine의 Memory Leak issue를 예로 들면 프로세스 관리자인 pm2로 관리를 하면서 로그는 process.on('uncaughtException') 과 같은 메소드로 예외처리를 한다거나 등의 여러가지 해결책들이 제시되고 있다고 한다.`


### 참고  
https://nodejs.org  
https://bcho.tistory.com  
https://d2.naver.com/helloworld  
https://asfirstalways.tistory.com  
https://118k.tistory.com  