---
layout: post
title: "Redis 성능 개선을 위한 GZIP 압축 로직을 적용시 Native OOM 이슈에 대한 분석"
subtitle: "2023-07-25-java-gzip-oom-issue.md"
date: 2023-07-25 18:40:00
author: "전봉근"
header-img: "img/posts/android-bg.jpg"
comments: true
tags: [java, jvm, spring, spring boot, os, k8s, kubernetes, pod]
---

# Redis 성능 개선을 위한 GZIP 압축 로직을 적용시 Native OOM 이슈에 대한 분석

## 원인 
원인은 Redis 의 Packet Loss 가 발생하였고 해당 애플리케이션의 전체 Pod 에서 OOM 이 발생하면서 재시작되는 현상이였고 Pod 의 Heap 메모리 사용량은 여유가 있는 상태였습니다.    
왜 이러한 문제가 발생하였는지 알아보겠습니다.     

## 컨테이너에서 OOM 발생
먼저 서버로그를 확인해본 결과 Gzip 압축 적용후에 점진적으로 메모리 사용량이 증가한 내용이 확인되었다.
![app-oom-1](/img/posts/language/java/gzip/app-oom-1.png)          
![app-oom-2](/img/posts/language/java/gzip/app-oom-2.png)          
     
또한 컨테이너에 OOM Kill 도 발생한 것을 확인할 수 있었다.
![oom-kill](/img/posts/language/java/gzip/oom-kill.png)          
         
그러나 이상한점은 Heap Memory 는 정상적인 수치로 확인되었다.     
![app-heap-1](/img/posts/language/java/gzip/app-heap-1.png)          
![app-heap-2](/img/posts/language/java/gzip/app-heap-2.png)          

## 원인분석
Native Memory 에서 OOM 이 발생하였고, Heap Memory 는 정상 적이라 애플리케이션 코드에 문제가 있다고 판단 하였고 관련하여 상기 언급 되었던  배포 항목중 gzip 데이터 압축 로직에 대해 검토하였다.     
     
1. try-catch 구문 에서의 Stream 객체 들의 메모리 반환 부분 의심
![code-1](/img/posts/language/java/gzip/code-1.png)        
     
2. GZIPOutputStream 외 다른 Stream 객체들도 finish() 와 close() 로 메모리 반환을 하고 있었지만, 정상 종료되지 않았을 경우 close() 하지 않는 것이 원인
   - GZIPOutputStream 의 finish() 메소드를 확인해보면 기본 stream 을 닫지 않고 출력 스트림에 압축 데이터 쓰기를 완료하게끔 작성 되어있다.
     ![code-2](/img/posts/language/java/gzip/code-2.png)        

3. 여기서 더 내부적으로 파악해보면 2번 이미지에 def 라는 객체는 `Deflater` 의 메소드를 활용한 것인데 `Deflater 는 네이티브 영역의 메모리(C heap) 을 사용한다.` 즉, 특정 케이스로 인하여 정상적으로 종료되지 않았을 경우 스트림이 닫히질 않아 네이티브 영역의 메모리가 쌓이게 되고 그로 인하여 컨테이너 전체의 OOM 이 발생하여 [OOM Killer](https://kubernetes.io/ko/docs/tasks/configure-pod-container/assign-memory-resource/)  에 의해 POD 가 자동으로 재 기동 된 것이다. 네이티브 메모리 영역 관련해서는 [JDK8 의 Metaspace 영역](https://bkjeon1614.github.io/blog/java-memory-metaspace) 을 참고하자.          
   ![code-3](/img/posts/language/java/gzip/code-3.png)        
   ![code-4](/img/posts/language/java/gzip/code-4.png)        
   - [Deflater 메모리 누수 관련](https://bugs.java.com/bugdatabase/view_bug?bug_id=4797189)

## 해결방법
[GZIPOutputStream의 디플레이터로 인한 기본 메모리 누수 해결](https://www.ibm.com/support/pages/apar/IZ97009) 을 참고해보면 기본 Deflater 의 close() 메소드를 선언하지 않아서 문제가 발생한다고 되어있다. 그러므로 GZIPOutputStream 의 finish() 호출 후 내부의 기본 Deflater 를 close 하도록 코드를 수정하였다.

1. try-with-resources 구문으로 대체 ([아이템 9. try-finally보다는 try-with-resources를 사용하라](https://recepinanc.medium.com/til-18-prefer-try-with-resources-to-try-catch-finally-afc8c0dc9c05) (with. Autocloseable))      
     
2. 코드예시 (Autocloseable 로 인하여 gzipOutputStream.finish() > GZIPOutputStream close > ByteArrayOutputStream Close 순으로 자원을 반납)    
   ![code-5](/img/posts/language/java/gzip/code-5.png)       

## 결과
적용 후 안정적으로 메모리가 확보된 것을 확인할 수 있다.   
![app-oom-finish-1](/img/posts/language/java/gzip/app-oom-finish-1.png)        

## 참고
- https://stackoverflow.com/questions/55790261/closing-gzipoutputstream-after-bytestream-copy-in-finally-block-break-zip
- https://recepinanc.medium.com
- https://bugs.java.com
- https://kubernetes.io
- https://www.ibm.com/
- https://devguli.com/