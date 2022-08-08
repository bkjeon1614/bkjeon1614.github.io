---
layout: post
title : "Spring Boot Graceful Shutdown"
subtitle : "2022-08-08-springboot-graceful-shutdown.md"
date: 2022-08-08 21:18:00
author: "전봉근"
header-img: "img/posts/android-bg.jpg"
comments: true
tags: [java, backend, spring, springboot]
---

# Spring Boot Graceful Shutdown
애플리케이션을 배포(ex: Rolling Deploy) 또는 특정 케이스로 종료할 때 여러가지 방법이 존재한다. 다만 케이스별로 정상적은 종료를 위하여 반드시 정상 종료 프로세스는 꼭 필수이다.
그러므로 애플리케이션을 정상적으로 종료할 수 있게 도와주는 Graceful Shutdown 에 대하여 알아보자.


## 리눅스 KILL 명령
리눅스 환경에서 프로세스를 종료할 때 KILL 을 사용하고 옵션에 따라 종료시키는 보편적으로 사용하는 동작은 아래와 같다.
> -9: 작업중인 모든 데이터를 저장하지 않고 프로세스를 종료하기 때문에 저장되지 않는 데이터는 소멸된다. (강제종료) 
> -15: 하던 작업을 모두 안전하게 저장한 후 프로세스를 종료한다. (정상종료)


## Graceful Shutdown 적용하기
Graceful Shutdown 은 Spring Boot 버전에 따라 적용하는 방식이 상이하다. 왠만하면 적용방법이 제일 간단한 `Spring Boot 2.3` 이상을 권장한다.
또한 Tomcat, Jetty, Undertow, Netty 모두에 대한 정상적인 종료 기능을 지원한다고 한다.


## Spring Boot 2.3 이상 적용
- 컨트롤러 생성   
  [TestController.java]  
  ```
  @RestController
  @Slf4j
  @RequestMapping("test")
  public class TestController {

      @GetMapping("excuteProcess/{processNum}")
      public String process(@PathVariable int processNum) throws InterruptedException {
          log.info("========================== Start Process -> Process Number: {}", processNum);
          Thread.sleep(20000);
          log.info("========================== End Process -> Process Number: {}", processNum);

          return "Process Success !!";
      }

  }
  ```  
- 실행하면 하기 이미지와 같이 20초 후에 정상적으로 로그를 남긴다.     
  ![springboot-graceful-shutdown-1](/img/posts/language/java/springboot-graceful-shutdown-1.png)   
- 그 다음 앱이 실행한 상태에서 작업 요청을 한 후 20초를 기다리지 않고 바로 kill -15 수행해보자.   
  ![springboot-graceful-shutdown-2](/img/posts/language/java/springboot-graceful-shutdown-2.png)   
  > End Process 관련 로그가 남겨지기 전에 종료되버린다. 로그를 확인해보면 애플리케이션에서는 kill -15 를 수신받긴 했지만 실제로 그 이후에 어떻게 종료해야될지 판단이 안되기 때문에 바로 종료가 되어버린 것이다.
- graceful 옵션을 적용해보자.   
- application.yml 설정   
  [application.yml]   
  ```
  spring:
    profiles:
      default: local
    lifecycle:
      timeout-per-shutdown-phase: 35s # 기본값은 30s 이다.

  server:
    port: 9090
    shutdown: graceful  # graceful shutdown 적용 (immediate 옵션도 존재하는 해당 옵션은 디폴트이며 말 그대로 즉시 종료이다.)
  ```  
- 설정 후 다시 작업 요청을 한 후 20초를 기다리지 않고 바로 kill -15 수행 (kill -15 명령을 내리고 그 도중에 요청하여 결과도 확인해보자.)   
  ![springboot-graceful-shutdown-3](/img/posts/language/java/springboot-graceful-shutdown-3.png)   
  > 정상적으로 End Process 까지 로그가 찍히고 정상적으로 종료되는것을 볼 수 있다. 또한 kill -15 명령을 내리고 요청을 다시 하였을 때 사이트에 연결할 수 없다고 네트워크 레벨에서 접근이 차단되어 앱에서 요청을 받지 못하여 종료진행시에 추가 유입되는 유저들이 요청하는 것을 방지할 수 있다.

> 해당 작업은 tomcat 환경 기준으로 진행된 것이며, undertow 는 동작이 다르다고하니 자세한 내용은 [https://www.baeldung.com/spring-boot-web-server-shutdown](https://www.baeldung.com/spring-boot-web-server-shutdown) 를 참고하자.


## Spring Boot 2.2 이하 적용
Spring Boot 2.2 이하는 개발자가 별도로 로직처리를 해줘야 한다.
- 컨트롤러 생성   
  [GracefulShutdownController.java]   
  ```
  import java.util.concurrent.TimeUnit;

  import org.springframework.http.HttpStatus;
  import org.springframework.http.ResponseEntity;
  import org.springframework.web.bind.annotation.GetMapping;
  import org.springframework.web.bind.annotation.RequestMapping;
  import org.springframework.web.bind.annotation.RestController;

  @RestController
  @RequestMapping("test")
  public class GracefulShutdownController {

      @GetMapping
      public ResponseEntity excuteShutdownTest() throws InterruptedException {
          TimeUnit.SECONDS.sleep(20);
          return new ResponseEntity<>("Graceful Shutdown Process Finished !!", HttpStatus.OK);
      }

  }  
  ```
- Graceful Shutdown Event Listener 생성   
  [GracefulShutdownEventListener.java]   
  ```
  import java.util.concurrent.ThreadPoolExecutor;
  import java.util.concurrent.TimeUnit;

  import org.springframework.context.ApplicationListener;
  import org.springframework.context.event.ContextClosedEvent;
  import org.springframework.stereotype.Component;

  import lombok.extern.slf4j.Slf4j;

  @Component
  @Slf4j
  public class GracefulShutdownEventListener implements ApplicationListener<ContextClosedEvent> {

      private final GracefulShutdownTomcatConnector gracefulShutdownTomcatConnector;

      public GracefulShutdownEventListener(GracefulShutdownTomcatConnector gracefulShutdownTomcatConnector) {
          this.gracefulShutdownTomcatConnector = gracefulShutdownTomcatConnector;
      }

      @Override
      public void onApplicationEvent(ContextClosedEvent event) {
          gracefulShutdownTomcatConnector.getConnector().pause();

          ThreadPoolExecutor threadPoolExecutor = (ThreadPoolExecutor) gracefulShutdownTomcatConnector.getConnector()
            .getProtocolHandler()
            .getExecutor();
          threadPoolExecutor.shutdown();

          try {
              threadPoolExecutor.awaitTermination(20, TimeUnit.SECONDS);
              log.info("Web Application Gracefully Stopped.");
          } catch (InterruptedException e) {
              Thread.currentThread().interrupt();
              e.printStackTrace();
              log.error("Web Application Graceful Shutdown Failed.");
          }
      }

  }  
  ```
- Graceful Shutdown Tomcat Connector 생성    
  [GracefulShutdownTomcatConnector.java]    
  ```
  import org.apache.catalina.connector.Connector;
  import org.springframework.boot.web.embedded.tomcat.TomcatConnectorCustomizer;
  import org.springframework.stereotype.Component;

  @Component
  public class GracefulShutdownTomcatConnector implements TomcatConnectorCustomizer {

      private volatile Connector connector;

      @Override
      public void customize(Connector connector) {
          this.connector = connector;
      }

      public Connector getConnector() {
          return connector;
      }

  }  
  ```
- Tomcat Config 생성    
  [TomcatConfig.java]    
  ```
  import org.springframework.boot.web.embedded.tomcat.TomcatServletWebServerFactory;
  import org.springframework.boot.web.servlet.server.ConfigurableServletWebServerFactory;
  import org.springframework.context.annotation.Bean;
  import org.springframework.context.annotation.Configuration;

  @Configuration
  public class TomcatConfig {

      private final GracefulShutdownTomcatConnector gracefulShutdownTomcatConnector;

      public TomcatConfig(GracefulShutdownTomcatConnector gracefulShutdownTomcatConnector) {
          this.gracefulShutdownTomcatConnector = gracefulShutdownTomcatConnector;
      }

      @Bean
      public ConfigurableServletWebServerFactory webServerFactory() {
          TomcatServletWebServerFactory factory = new TomcatServletWebServerFactory();
          factory.addConnectorCustomizers(gracefulShutdownTomcatConnector);

          return factory;
      }

  }  
  ```
- 실행하면 하기 이미지와 같이 20초 후에 정상적으로 로그를 남긴다.     
  ![springboot-graceful-shutdown-4](/img/posts/language/java/springboot-graceful-shutdown-4.png)   
- 그 다음 앱이 실행한 상태에서 작업 요청을 한 후 20초를 기다리지 않고 바로 kill -15 수행해보자.   
  ![springboot-graceful-shutdown-5](/img/posts/language/java/springboot-graceful-shutdown-5.png)    
  > End Process 관련 로그가 남겨지기 전에 종료되버린다. 로그를 확인해보면 애플리케이션에서는 kill -15 를 수신받긴 했지만 실제로 그 이후에 어떻게 종료해야될지 판단이 안되기 때문에 바로 종료가 되어버린 것이다.    
- 설정 후 다시 작업 요청을 한 후 20초를 기다리지 않고 바로 kill -15 수행 (kill -15 명령을 내리고 그 도중에 요청하여 결과도 확인해보자.)    
  ![springboot-graceful-shutdown-6](/img/posts/language/java/springboot-graceful-shutdown-6.png)    


## 참고
- https://www.baeldung.com/
- https://blog.naver.com/