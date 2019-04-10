---
layout: post
title : "JAVA Garbage Collection (작성중)"
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
- 새롭게 생성한 객체의 대부분이 여기에 위치한다. 대부분의 객체가 금방 접근 불가능 상태가 되기 때문에 매우 많은 객체가 Young 영역에 생성되었다가 사라진다. 이 영역에서 객체가 사라질 때 Minor GC가 발생한고 말한다.
#### Old 영역(=Old Generation 영역)
- 접근 불가능 상태로 되지 않아 Young 영역에서 살아남은 객체가 여기로 복사된다. 대부분 Young 영역보다 크게 할당하며, 크기가 큰 만큼 Young 영역보다 GC는 적게 발생한다. 이 영역에서 객체가 사라질 때 Major GC(혹은 Full GC)가 발생한다고 말한다.

위의 영역의 데이터 흐름은 아래와 같다. (Perm 영역은 다음 내용에 참고하자.)
- ( Young 영역 할당 -> Old 영역 이동 ) | ( Perm 영역 )

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
##### 이미지 출처: https://d2.naver.com/helloworld/1329

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

### Serial GC (-XX:+UseSerialGC)
- Young 영역에서의 GC는 위의 내용에서 설명한 방식을 사용한다. Old 영역의 GC는 mark-sweep-compact라는 알고리즘을 사용한다 순서는 아래와 같다.
  1. Old 영역에 살아 있는 객체를 식별(mark)한다.
  2. heap의 앞 부분부터 확인하여 살아 있는 것만 남긴다(Sweep)
  3. 각 객체들이 연속되게 쌓이도록 heap의 가장 앞 부분부터 채워서 객체가 존재하는 부분과 객체가 없는 부분으로 나눈다.(Compoaction)
> Serial GC는 적은 메모리와 CPU 코어 개수가 적을 때 적합한 방식이다.

### Parallel GC (-XX:+UseParallelGC)
- Serial GC와 기본적인 알고리즘은 같지만 Serial GC는 GC를 처리하는 스레드가 하나인 것에 비해, Parallel GC는 GC를 처리하는 스레드가 여러개이다. 그러므로 Serial GC 보다 빠르게 객체를 처리할 수 있다.
> Serial GC는 메모리가 충분하고 코어의 개수가 많을 때 유리하다. 또한 Throughput GC라고도 불린다.

### Parallel Old GC(-XX:+UseParallelOldGC)
- Parallel Old GC는 JDK 5 update 6부터 제공한 GC 방식이다. 앞서 설명한 Parallel GC와 비교하여 Old 영역의 GC 알고리즘만 다르다. 이 방식은 Mark-Summary-Compaction 단계를 거친다. Summary 단계는 앞서 GC를 수행한 영역에 대해서 별도로 살아 있는 객체를 식별한다는 점에서 Mark-Sweep-Compaction 알고리즘의 Sweep 단계와 다르며, 약간 더 복잡한 단계를 거친다. (어떤건지만 알아두기만 하자)

### CMS GC (-XX:+UseConcMarkSweepGC)
![java-gc-2](/img/posts/language/java/java-gc-2.png)
##### 이미지 출처: https://d2.naver.com/helloworld/1329

- 초기 Initial Mark 단계에서는 클래스 로더에서 가장 가까운 객체 중 살아 있는 객체만 찾는 것으로 끝낸다. 따라서, 멈추는 시간은 매우 짧다. 그리고 Concurrent Mark 단계에서는 방금 살아있다고 확인한 객체에서 참조하고 있는 객체들을 따라가면서 확인한다. 이 단계의 특징은 다른 스레드가 실행 중인 상태에서 동시에 진행된다는 것이다.

그 다음 Remark 단계에서는 Concurrent Mark 단계에서 새로 추가되거나 참조가 끊긴 객체를 확인한다. 마지막으로 Concurrent Sweep 단계에서는 쓰레기를 정리하는 작업을 실행한다. 이 작업도 다른 스레드가 실행되고 있는 상황에서 진행한다.

이러한 단계로 진행되는 GC 방식이기 때문에 stop-the-world 시간이 매우 짧다. 모든 애플리케이션의 응답 속도가 매우 중요할 때 CMS GC를 사용하며, Low Latency GC라고도 부른다.

그런데 CMS GC는 stop-the-world 시간이 짧다는 장점에 반해 다음과 같은 단점이 존재한다.
- 다른 GC 방식보다 메모리와 CPU를 더 많이 사용한다.
- Compaction 단계가 기본적으로 제공되지 않는다.

> 따라서, CMS GC를 사용할 때에는 신중히 검토한 후에 사용해야 한다. 그리고 조각난 메모리가 많아 Compaction 작업을 실행하면 다른 GC 방식의 stop-the-world 시간보다 stop-the-world 시간이 더 길기 때문에 Compaction 작업이 얼마나 자주, 오랫동안 수행되는지 확인해야 한다.

### G1 GC (-XX:+UseG1GC)
- G1 GC를 이해하기 위해선 지금까지의 Young 영역과 Old 영역에 대해서는 잊는 것이 좋다. 다음 그림에서 보다시피, G1 GC는 바둑판의 각 영역에 객체를 할당하고 GC를 실행한다. 그러다가, 해당 영역이 꽉 차면 다른 영역에서 객체를 할당하고 GC를 실행한다. 즉, 지금까지 설명한 Young의 세가지 영역에서 데이터가 Old 영역으로 이동하는 단계가 사라진 GC 방식이라고 이해하면 된다. G1 GC는 장기적으로 말도 많고 탈도 많은 CMS GC를 대체하기 위해서 만들어 졌다.

![java-gc-3](/img/posts/language/java/java-gc-3.png)

> G1 GC의 가장 큰 장점은 성능이다. 지금까지 설명한 어떤 GC 방식보다도 빠르다. 하지만, JDK 6에서는 G1 GC를 early access라고 부르며 그냥 시험삼아 사용할 수만 있도록 한다. 그리고 JDK 7에서 정식으로 G1 GC를 포함하여 제공한다. 그러나 JDK 7을 실서비스에서 사용하려면 많은 검증 기간(1년은 필요하다는 생각이다)을 거쳐야 할 것으로 보이기 때문에, G1 GC를 당장 사용하고 싶어도 더 기다리는 것이 좋다는 것이 개인적인 생각이다. java6에서 G1 GC를 적용했다가 JVM Crash가 발생했다는 말도 몇 번 들었기에 더더욱 안정화될 때까지 기다리는 것이 좋겠다. (java7 u60 이상 버전에서 사용하고, java8에서는 class 영역이 클 경우 class unloading을 하는 gc 시간이 매우 길어질 수 있다. 아래와 같은 옵션으로 대부분 문제가 없어진다.)

```
-XX:+UseLargePagesInMetaspace
```

# References
- https://www.oracle.com/technetwork/java/index.html
- https://www.oracle.com/webfolder/technetwork/tutorials/obe/java/gc01/index.html
- https://docs.oracle.com/javase/8/docs/technotes/guides/vm/gctuning/sizing.html
- https://d2.naver.com/helloworld/1329
- https://www.holaxprogramming.com/2013/07/20/java-jvm-gc/