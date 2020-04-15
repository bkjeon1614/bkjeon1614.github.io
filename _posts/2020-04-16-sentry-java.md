---
layout: post
title : "Sentry Java 연동"
subtitle : "2020-04-16-sentry-java.md"
date: 2020-04-16 02:00:00
author: "전봉근"
header-img: "img/posts/android-bg.jpg"
comments: true
tags: [java, sentry]
---

## Sentry Java 연동
----------------------------------------------------------------

Sentry란 오류 모니터링을 제공하는 오픈소스 플랫폼이다. 클라우드(=sentry.io)는 무료 및 비용별 과금을 제공하지만 자체 구축하면 무료로 사용할 수 있다. 다양한 언어를 제공하므로 유용하게 사용할 수 있다.

1. sentry.io 에 접속하여 회원가입을 한다.

2. 회원가입을하면 DSN 번호를 확인하자.
   - Settings -> Client Keys (DSN) -> DSN 번호 확인

3. 이제 java 경로인 resources 폴더에 sentry.properties 파일을 생성하고 하기 내용을 입력하자.
   ```
      dsn={myDSN}
      servername={serverName}
      stacktrace.app.packages={Package}
   ```

4. 그 다음 resources 폴더경로에 logback-spring.xml 파일을 생성하고 하기 내용을 입력하자.
   ```
      <!-- sentry -->
      <appender name="Sentry" class="io.sentry.logback.SentryAppender">
         <filter class="ch.qos.logback.classic.filter.ThresholdFilter">
            <level>WARN</level>
         </filter>
      </appender>

      <root level="info">
            <appender-ref ref="FILE"/>
            <appender-ref ref="Sentry" />
      </root>
   ```

5. gradle 의존성에 추가
   ```
      ...

      // sentry logback
      compile 'io.sentry:sentry-logback:1.7.27'
   ```

6. 모든 설정을 다 마치고 에러를 발생시키면 사이트에 에러가 Tracking이 되고 있는 것을 확인할 수 있다. Noti 또한 원하는 방법으로 설정이 가능하다.