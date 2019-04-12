---
layout: post
title : "JAVA Garbage Collection"
subtitle : "2019-04-05-JAVA Garbage Collection"
date: 2019-04-05 19:00:00
author: "전봉근"
header-img: "img/posts/android-bg.jpg"
comments: true
tags: [java]
---

# Garbage Collection 이란
Java Application에서 사용하지 않는 메모리를 자동으로 수거하는 기능을 말한다.

## Garbage Collection 과정
GC에 대해 알아보기 전에 알아야할 용어인 "Stop-the-world"를 참고하면서 읽자.

### Stop-the-world
GC를 실행하기 위해 JVM이 애플리케이션 실행을 멈추는 것이다. 이것이 발생하면 GC를 실행하는 쓰레드를 제외한 나머지 쓰레드는 모두 작업을 멈춘다. GC 작업을 완료한 이후에야 중단했던 작업을 다시 시작한다.  
어떤 GC 알고리즘을 사용하더라고 stop-the-world는 발생한다. 즉, GC 튜닝이란 stop-the-world 시간을 줄이는 것이다.

Java는 프로그램 코드에서 메모리를 명시적으로 지정하여 해제하지 않는다. 그렇기 때문에 GC가 더 이상 필요 없는 쓰레기 객체를 찾아 지우는 작업을 한다. 하지만 명시적으로 해제하려고 해당 객체를 null로 지정하거나 System.gc() 메서드를 호출하는 개발자가 있다. Null로 지정하는 것은 큰 문제가 안 되지만, System.gc() 메서드를 호출하는 것은 시스템의 성능에 영향을 야기하므로 절대로 사용하면 안된다.

#### 아래의 2가지 가정에 의하여 만들어진 것이 GC이다.
- 대부분의 객체는 금방 접근 불가능한 상태가 된다.
- 오래된 객체에서 젊은 객체로의 참조는 아주 적게 존재한다.  

이 가정의 장점을 최대한 살리기 위하여 HotSpot VM(=자바엔진이름)에서는 크게 2개로 물리적 공간을 나누었다. 그것이 Young 영역과 Old 영역이다.

### 영역
#### Young 영역(=Young Generation 영역)
새롭게 생성한 객체의 대부분이 여기에 위치한다. 대부분의 객체가 금방 접근 불가능 상태가 되기 때문에 매우 많은 객체가 Young 영역에 생성되었다가 사라진다. 이 영역에서 객체가 사라질 때 Minor GC가 발생한고 말한다.

#### Old 영역(=Old Generation 영역)
접근 불가능 상태로 되지 않아 Young 영역에서 살아남은 객체가 여기로 복사된다. 대부분 Young 영역보다 크게 할당하며, 크기가 큰 만큼 Young 영역보다 GC는 적게 발생한다. 이 영역에서 객체가 사라질 때 Major GC(혹은 Full GC)가 발생한다고 말한다.

위의 영역의 데이터 흐름은 아래와 같다. (Perm 영역은 다음 내용에 참고하자.)
> ( Young 영역 할당 -> Old 영역 이동 ) | ( Perm 영역 )

Permanent Generation 영역(이하 Perm 영역)
- 해당 영역은 Method Area라고도 불린다. 객체나 억류된 문자열 정보를 저장하는 곳이며, Old 영역에서 살아남은 객체가 영원히 남아 있는 곳이 절대 아니며 여기서 GC가 발생해도 Major GC의 횟수에 포함된다.

Old 영역의 객체가 Young 영역의 객체를 참조하는 경우
- 이 경우 Old 영역에는 512바이트의 덩어리로 되어 있는 Card Table이 존재한다. 여기에선 Old 영역에 있는 객체가 Young 영역의 객체를 참조할 때마다 정보가 표시된다. Young 영역의 GC를 실행할 때에는 Old 영역에 있는 모든 객체의 참조를 확인하지 않고, 이 Card Table만 뒤져서 GC 대상인지 아닌지 식별한다.
> Card Table은 Write Barrier(=Minor GC를 빠르게 할 수 있도록 하는 장치)를 사용하여 관리하므로 약간의 오버헤드가 발생하지만 전반적인 GC 시간은 줄어들게 된다. 

## Young 영역의 구성

위의 내용되로 GC에서 제일 먼저 할당되는 Young의 영역은 Eden 영역, Survivor 영역(2개)로 총 3개의 영역으로 구성된다.

위의 총 3개의 영역을 각 영역의 처리 절차를 순서대로 아래에 기술하면 다음과 같다.
- 새로 생성한 대부분의 객체는 Eden 영역에 위치한다.
- Eden 영역에서 GC가 한 번 발생한 후 살아남은 객체는 Survivor 영역 중 하나로 이동
- Eden 영역에서 GC가 발생하면 이미 살아남은 객체가 존재하는 Survivor 영역으로 객체가 계속 쌓인다.
- 하나의 Survivor 영역이 가득 차게 되면 그 중에서 살아남은 객체를 다른 Survivor 영역으로 이동한다. 그리고 가득 찬 Survivor 영역은 아무 데이터도 없는 상태로 된다.
- 이 과정을 반복하다가 계속해서 살아남아 있는 객체는 Old 영역으로 이동하게 된다.

위와 같이 Young 영역에서 Minor GC를 통하여 Old 영역까지 데이터가 쌓인 것을 아래 그림을 보면 알 수 있다.
![java-gc-1](/img/posts/language/java/java-gc-1.png)
이미지 출처: https://d2.naver.com/helloworld/1329

> Young 영역에서는 Eden 영역에 최초로 객체가 만들어지고, Survivor 영역을 통해서 Old 영역으로 오래 살아남은 객체가 이동한다는 사실은 꼭 기억하자.

### 추가적으로 아래는 알고만 있어도 된다.
- bump-the-pointer: Eden 영역에 마지막 객체를 캐싱 해두고, 그 다음에 생성되는 객체가 있으면 Eden에 넣기에 적당한지 판단한다.
- TLABs: 멀티 스레드 환경을 고려해 Eden영역에 lock-contention으로 인한 성능 이슈를 개선한 것이다.

## Old 영역에 대한 GC
Old 영역은 데이터가 가득 차면 GC를 실행한다. GC 방식에 따라서 처리 절차가 달라지므로, 어떤 GC 방식이 있는지는 아래를 살펴보자. (JDK 7 기준)
- Serial GC
- Parallel GC
- Parallel Old GC(Parallel Compacting GC)
- Concurrent Mark & Sweep GC(이하 CMS)
- G1(Garbage First) GC

> 이 중 운영서버에서 절대 사용하면 안 되는 방식이 Serial GC이다. Serial GC는 데스크톱의 CPU 코어가 싱글일 경우에만 사용하기 위하여 만든 방식이다. (성능 저하 발생)

아래는 java 버전마다 default로 사용되는 gc이다.
- java 7 - Parallel GC
- java 8 - Parallel GC
- java 9 - G1 GC
- java 10 - Parallel GC


### Serial GC (-XX:+UseSerialGC)
Young 영역과 Old 영역이 시리얼하게(연속적으로) 처리되며 하나의 CPU를 사용합니다. 이 처리를 수행할 때를 Stop-the-world라고 표현합니다. 다시 말하면, 콜렉션이 수행될 때 애플리케이션 수행이 정지됩니다.  
  ![java-gc-1](/img/posts/language/java/java-gc-1.png)
  이미지 출처: https://d2.naver.com/helloworld/1329

  1. 일단 살아있는 객체는 Eden 영역에 존재한다.
  2. Eden 영역이 꽉차게 되면 To Survivor 영역(비어있는 영역)으로 살아 있는 객체가 이동합니다. 이때 Survivor 영역에 들어가기에 너무 큰 객체는 바로 Old 영역으로 이동합니다. 그리고 From Survivor 영역에 있는 살아 있는 객체는 To Survivor 영역으로 이동합니다.
  3. To Survivor 영역이 꽉 찼을 경우, Eden 영역이나 From Survivor 영역에 남아 있는 객체들은 Old 영역으로 이동합니다.

이 이후에 Old 영역이나 Perm 영역에 있는 객체들은 Mark-sweep-compact 콜렉션 알고리즘을 따릅니다. 이 알고리즘에 대해서 간단하게 말하면, 안 쓰는 거 표시해서 삭제하고 한 곳으로 모으는 알고리즘입니다.
  1. Old 영역으로 이동된 객체들 중 살아 있는 개체를 식별합니다. (Mark)
  2. Old 영역의 객체들을 훑는 작업을 수행하여 쓰레기 객체를 식별합니다. (Sweep)
  3. 필요 없는 객체들을 지우고 살아 있는 객체들을 한 곳으로 모은다 (Compact)

> Serial GC는 적은 메모리와 CPU 코어 개수가 적을 때 적합한 방식이다. 하지만 Serial GC는 절대 사용하면 안되는 GC이다. 데스크톱의 CPU 코어가 싱글일 경우에만 사용하기 위하여 만든 방식이기 때문이다. (성능 저하 발생)


### Parallel GC (-XX:+UseParallelGC)
Throughput Collector 라고도 불리며 이 방식은 다른 CPU가 대기 상태로 남아 있는 것을 최소화하는 것입니다. 시리얼 콜렉터와 달리 Young 영역에서의 콜렉션을 병렬(Parallel)로 처리합니다. Old 영역의 GC는 시리얼 콜렉터와 마찬가지로 Mark-Sweep-Compact 콜렉션 알고리즘을 사용 합니다.

  ![java-gc-new-1](/img/posts/language/java/java-gc-new-1.png)
  이미지 출처: https://www.oracle.com/technetwork/java/javase/tech/memorymanagement-whitepaper-1-150020.pdf

> 많은 CPU 를 사용하기 때문에 GC의 부하를 줄이고 애플리케이션의 처리량을 증가시킬 수 있습니다. 즉, 순간적으로 트래픽이 몰려도 일시 중단을 견딜 수 있고 GC에 의해 야기된 CPU 오버 헤드에 대해 최적화할 수 있는 애플리케이션에 가장 적합하다.


### Parallel Old GC(-XX:+UseParallelOldGC)
병렬 콜렉터와 다른 점은 Old 영역 GC에서 새로운 알고리즘을 사용합니다. 그러므로 Young 영역에 대한 GC는 병렬 콜렉터와 동일하지만, Old 영역의 GC는 다음의 3단계를 거치게 됩니다. 
  1. Mark 단계 : 살아 있는 객체를 식별하여 표시해 놓는 단계 
  2. Sweep 단계 : 이전에 GC를 수행하여 컴팩션된 영역에 살아 있는 객체의 위치를 조사하는 단계 
  3. Compact 단계 : 컴팩션을 수행하는 단계. 수행 이후에는 컴팩션된 영역과 비어 있는 영역으로 나뉩니다. 

> 병렬 콜렉터와 동일하게 이 방식도 여러 CPU를 사용하는 서버에 적합합니다. GC를 사용하는 스레드 개수는 -XX:ParallelGCThreads=n 옵션으로 조정할 수 있습니다. 


### CMS GC (-XX:+UseConcMarkSweepGC)
이 방식은 low-latency collector로도 알려져 있으며, 힙 메모리 영역의 크기가 클 때 적합합니다. Young 영역에 대한 GC는 병렬 콜렉터와 동일합니다. Old 영역의 GC는 다음 단계를 거칩니다.
  1. Mark 단계 : 매우 짧은 대기 시간으로 살아 있는 객체를 찾는 단계
  2. Sweep 단계 : 서버 수행과 동시에 살아 있는 객체에 표시를 해 놓는 단계
  3. Remark 단계 : Concurrent 표시 단계에서 표시하는 동안 변경된 객체에 대해서 다시 표시하는 단계
  4. Concurrent Sweep 단계 : 표시되어 있는 쓰레기를 정리하는 단계

  ![java-gc-2](/img/posts/language/java/java-gc-2.png)
  이미지 출처: https://d2.naver.com/helloworld/1329

초기 Initial Mark 단계에서는 클래스 로더에서 가장 가까운 객체 중 살아 있는 객체만 찾는 것으로 끝낸다. 따라서, 멈추는 시간은 매우 짧다. 그리고 Concurrent Mark 단계에서는 방금 살아있다고 확인한 객체에서 참조하고 있는 객체들을 따라가면서 확인한다. 이 단계의 특징은 다른 스레드가 실행 중인 상태에서 동시에 진행된다는 것이다.

그 다음 Remark 단계에서는 Concurrent Mark 단계에서 새로 추가되거나 참조가 끊긴 객체를 확인한다. 마지막으로 Concurrent Sweep 단계에서는 쓰레기를 정리하는 작업을 실행한다. 이 작업도 다른 스레드가 실행되고 있는 상황에서 진행한다.

이러한 단계로 진행되는 GC 방식이기 때문에 stop-the-world 시간이 매우 짧다. 모든 애플리케이션의 응답 속도가 매우 중요할 때 CMS GC를 사용하며, Low Latency GC라고도 부른다.

그런데 CMS GC는 stop-the-world 시간이 짧다는 장점에 반해 다음과 같은 단점이 존재한다.
- 다른 GC 방식보다 메모리와 CPU를 더 많이 사용한다.
- Compaction 단계가 기본적으로 제공되지 않는다.

CMS는 컴팩션 단계를 거치지 않기 때문에 왼쪽으로 메모리를 몰아 놓는 작업을 수행하지 않습니다. 그래서 GC 이후에 그림과 같이 빈 공간이 발생하므로, -XX:CMSInitiatingOccupancyFraction=n 옵션을 사용하여 Old 영역의 %를 n 값에 지정합니다. (기본값 : 68)

> CMS 콜렉터 방식은 2개 이상의 프로세서를 사용하는 서버에 적당합니다. 가장 적당한 대상으로는 웹서버가 있습니다. CMS 콜렉터는 추가적인 옵션으로 점진적 방식을 지원합니다. 이 방식은 Young 영역의 GC를 더 잘게 쪼개어 서버의 대기 시간을 줄일 수 있습니다. CPU가 많지 않고 시스템의 대기 시간이 짧아야 할 때 사용하면 좋습니다. 점진적은 GC를 수행하려면 -XX:+CMSIncrementalMode 옵션을 지정하면 됩니다. JVM에 따라서는 -Xingc라는 옵션을 지정해도 같은 의미가 됩니다. 하지만 이 옵션을 지정할 경우 예기치 못한 성능 저하가 발생할 수 있으므로, 충분한 테스트를 한 후에 운영 서버에 적용해야 합니다.


### G1 GC (-XX:+UseG1GC)
G1 GC를 이해하기 위해선 지금까지의 Young 영역과 Old 영역에 대해서는 잊는 것이 좋다. 다음 그림에서 보다시피, G1 GC는 바둑판의 각 영역에 객체를 할당하고 GC를 실행한다. 그러다가, 해당 영역이 꽉 차면 다른 영역에서 객체를 할당하고 GC를 실행한다. 즉, 지금까지 설명한 Young의 세가지 영역에서 데이터가 Old 영역으로 이동하는 단계가 사라진 GC 방식이라고 이해하면 된다. G1 GC는 장기적으로 말도 많고 탈도 많은 CMS GC를 대체하기 위해서 만들어 졌다.

![java-gc-3](/img/posts/language/java/java-gc-3.png)

> G1 GC의 가장 큰 장점은 성능이다. 지금까지 설명한 어떤 GC 방식보다도 빠르다. 하지만, JDK 6에서는 G1 GC를 early access라고 부르며 그냥 시험삼아 사용할 수만 있도록 한다. 그리고 JDK 7에서 정식으로 G1 GC를 포함하여 제공한다. 그러나 JDK 7을 실서비스에서 사용하려면 많은 검증 기간(1년은 필요하다는 생각이다)을 거쳐야 할 것으로 보이기 때문에, G1 GC를 당장 사용하고 싶어도 더 기다리는 것이 좋다는 것이 개인적인 생각이다. java6에서 G1 GC를 적용했다가 JVM Crash가 발생했다는 말도 몇 번 들었기에 더더욱 안정화될 때까지 기다리는 것이 좋겠다. (java7 u60 이상 버전에서 사용하고, java8에서는 class 영역이 클 경우 class unloading을 하는 gc 시간이 매우 길어질 수 있다. 아래와 같은 옵션으로 대부분 문제가 없어진다.)

```
-XX:+UseLargePagesInMetaspace
```

# 마무리
개발자인 경우에는 GC에 대해 이해만 하고 있으면 된다. 단, 시스템 오픈 전 성능테스트 또는 서버 세팅시 알맞는 GC 방식을 개발한 시스템에 적용하자.

# References
- https://www.oracle.com/technetwork/java/javase/tech/memorymanagement-whitepaper-1-150020.pdf
- https://www.oracle.com/technetwork/java/index.html
- https://www.oracle.com/webfolder/technetwork/tutorials/obe/java/gc01/index.html
- https://docs.oracle.com/javase/8/docs/technotes/guides/vm/gctuning/sizing.html
- https://d2.naver.com/helloworld/1329
- https://12bme.tistory.com/57
- https://www.holaxprogramming.com/2013/07/20/java-jvm-gc/