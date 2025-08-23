---
layout: post
title: "Java Mutex(뮤텍스) 와 Semaphore(세마포어) 그리고 Spinlock(스핀락)"
subtitle: "2024-12-11-java-mutex-semaphore"
date: 2024-12-11 20:40:00
author: "전봉근"
header-img: "img/posts/android-bg.jpg"
comments: true
tags: [java, spring, spring boot, jvm, thread]
---

## Java Mutex(뮤텍스) 와 Semaphore(세마포어) 그리고 Spinlock(스핀락)
[참고코드](https://github.com/bkjeon1614/java-example-code/tree/main/java11/bkjeon-mybatis-codebase/base-api)


### Mutex(뮤텍스)   
Mutex 는 Mutual Exclusion(상호 배제)의 약자이며 Lock(락) 이라고도 한다. Mutex 는 한 번에 하나의 스레드만이 특정 리소스나 코드 섹션에 접근할 수 있도록 한다. `즉, 한 시점에 단 하나의 스레드만이 리소스를 사용할 수 있게 된다.` (Lock/UnLock)        
        
또한 Java 에서는 ReentrantLock 라는 Lock 인터페이스가 존재하는데 이걸 활용해서 Lock 의 획득과 해제에 대한 부분을 제어할 수 있다.        

[LockController.java]       
```
// 예제코드
@RestController
@RequestMapping("v1/lock")
@RequiredArgsConstructor
public class LockController {

    private final LockService lockService;

    @ApiOperation("Mutex 예제")
    @GetMapping("/mutex")
    public void isMutex() throws InterruptedException {
        ExecutorService executor = Executors.newFixedThreadPool(5); // 동시에 5개의 스레드를 실행
        for (int i = 0; i < 5; i++) {
            final int threadId = i;
            executor.submit(() -> lockService.isMutex(threadId));
        }

        executor.shutdown();
        boolean finished = executor.awaitTermination(10, TimeUnit.SECONDS);
        assert finished;
    }

}
```            

[LockService.java]     
```
@Slf4j
@Service
public class LockService {

    private final Lock lock = new ReentrantLock();

    public void isMutex(int threadId) {
        // 자원 진입 시도
        log.info("Thread {} 자원 진입 시도", threadId);
        lock.lock();
        try {
            // 자원 진입!
            log.info("Thread {} 자원 진입 완료!", threadId);
            // 자원에 대한 작업을 수행하는 동안 지연을 추가하여 로그 출력 관찰
            Thread.sleep(1000);
        } catch (InterruptedException e) {
            Thread.currentThread().interrupt();
        } finally {
            // 자원 사용 완료!
            log.info("Thread {} 자원 사용 완료!", threadId);
            lock.unlock();
        }
    }

}
```       

[실행결과]
```
2024-12-12 11:10:32  INFO [,,,] 37376 --- [pool-4-thread-1] c.e.b.b.s.api.v1.lock.LockService        : Thread 0 자원 진입 시도
2024-12-12 11:10:32  INFO [,,,] 37376 --- [pool-4-thread-2] c.e.b.b.s.api.v1.lock.LockService        : Thread 1 자원 진입 시도
2024-12-12 11:10:32  INFO [,,,] 37376 --- [pool-4-thread-1] c.e.b.b.s.api.v1.lock.LockService        : Thread 0 자원 진입 완료!
2024-12-12 11:10:32  INFO [,,,] 37376 --- [pool-4-thread-3] c.e.b.b.s.api.v1.lock.LockService        : Thread 2 자원 진입 시도
2024-12-12 11:10:32  INFO [,,,] 37376 --- [pool-4-thread-4] c.e.b.b.s.api.v1.lock.LockService        : Thread 3 자원 진입 시도
2024-12-12 11:10:32  INFO [,,,] 37376 --- [pool-4-thread-5] c.e.b.b.s.api.v1.lock.LockService        : Thread 4 자원 진입 시도
2024-12-12 11:10:33  INFO [,,,] 37376 --- [pool-4-thread-1] c.e.b.b.s.api.v1.lock.LockService        : Thread 0 자원 사용 완료!
2024-12-12 11:10:33  INFO [,,,] 37376 --- [pool-4-thread-2] c.e.b.b.s.api.v1.lock.LockService        : Thread 1 자원 진입 완료!
2024-12-12 11:10:34  INFO [,,,] 37376 --- [pool-4-thread-2] c.e.b.b.s.api.v1.lock.LockService        : Thread 1 자원 사용 완료!
2024-12-12 11:10:34  INFO [,,,] 37376 --- [pool-4-thread-3] c.e.b.b.s.api.v1.lock.LockService        : Thread 2 자원 진입 완료!
2024-12-12 11:10:35  INFO [,,,] 37376 --- [pool-4-thread-3] c.e.b.b.s.api.v1.lock.LockService        : Thread 2 자원 사용 완료!
2024-12-12 11:10:35  INFO [,,,] 37376 --- [pool-4-thread-4] c.e.b.b.s.api.v1.lock.LockService        : Thread 3 자원 진입 완료!
2024-12-12 11:10:36  INFO [,,,] 37376 --- [pool-4-thread-4] c.e.b.b.s.api.v1.lock.LockService        : Thread 3 자원 사용 완료!
2024-12-12 11:10:36  INFO [,,,] 37376 --- [pool-4-thread-5] c.e.b.b.s.api.v1.lock.LockService        : Thread 4 자원 진입 완료!
2024-12-12 11:10:37  INFO [,,,] 37376 --- [pool-4-thread-5] c.e.b.b.s.api.v1.lock.LockService        : Thread 4 자원 사용 완료!
```      
     
> 실행결과를 보면 하나의 스레드가 종료되어야 다른 스레드가 진입할 수 있는걸 확인할 수 있다. Ex) Thread 0 자원 사용 완료 전까지 다른 Thread 들은 자원 진입시도 또는 진입완료 에서 대기하다가 Thread 0 의 작업이 끝나야 다음 작업에 할당된 Thread1 이 실행하는걸 확인할 수 있다. 즉, 상호 배제를 통한 단일 리소스 접근 제어를 한다.

     
### Semaphore(세마포어)    
세마포어는 리소스에 `동시에 접근할 수 있는 스레드의 수를 제한`합니다. 세마포어는 특정 수의 `허가증(permits)` 을 가지고 있으며, 자바에서 Semaphore 클래스의 acquire() 메서드가 허가증을 주는 역할을 한다고 보시면 된다. 즉, 스레드가 리소스에 접근하기 위해서는 허가증을 획득해야 합니다. 모든 허가증이 사용 중일 때 추가 스레드는 허가증이 반환될 때까지 대기합니다. (Acquire/Release)

멀티프로그래밍 환경에서 공유 자원에 대한 접근을 제한하는 방법으로 사용       

[LockController.java]     
```
...

    @ApiOperation("Semaphore 예제")
    @GetMapping("/semaphore")
    public void isSemaphore() throws InterruptedException {
        ExecutorService executor = Executors.newFixedThreadPool(10); // 동시에 10개의 스레드를 실행

        for (int i = 0; i < 10; i++) {
            final int threadId = i;
            executor.submit(() -> lockService.isSemaphore(threadId));
        }

        executor.shutdown();
        boolean finished = executor.awaitTermination(20, TimeUnit.SECONDS);
        assert finished;
    }
```       

[LockService.java]     
```
...

    public void isSemaphore(int threadId) {
        try {
            // 자원 진입 시도
            log.info("Thread {} 자원 진입 시도", threadId);

            // 허가증 획득 시도
            semaphore.acquire();

            // 자원 진입!
            log.info("Thread {} 자원 진입 완료!", threadId);

            // 자원에 대한 작업을 수행하는 동안 일부 지연을 추가
            Thread.sleep(1000);
        } catch (InterruptedException e) {
            Thread.currentThread().interrupt();
        } finally {
            log.info("Thread {} 자원 사용 완료!", threadId);
            semaphore.release();
        }
    }
```     

[실행결과]
```
2024-12-12 13:15:58  INFO [,,,] 48755 --- [pool-4-thread-1] c.e.b.b.s.api.v1.lock.LockService        : Thread 0 자원 진입 시도
2024-12-12 13:15:58  INFO [,,,] 48755 --- [pool-4-thread-1] c.e.b.b.s.api.v1.lock.LockService        : Thread 0 자원 진입 완료!
2024-12-12 13:15:58  INFO [,,,] 48755 --- [pool-4-thread-2] c.e.b.b.s.api.v1.lock.LockService        : Thread 1 자원 진입 시도
2024-12-12 13:15:58  INFO [,,,] 48755 --- [pool-4-thread-2] c.e.b.b.s.api.v1.lock.LockService        : Thread 1 자원 진입 완료!
2024-12-12 13:15:58  INFO [,,,] 48755 --- [pool-4-thread-3] c.e.b.b.s.api.v1.lock.LockService        : Thread 2 자원 진입 시도
2024-12-12 13:15:58  INFO [,,,] 48755 --- [pool-4-thread-4] c.e.b.b.s.api.v1.lock.LockService        : Thread 3 자원 진입 시도
2024-12-12 13:15:58  INFO [,,,] 48755 --- [pool-4-thread-3] c.e.b.b.s.api.v1.lock.LockService        : Thread 2 자원 진입 완료!
2024-12-12 13:15:58  INFO [,,,] 48755 --- [pool-4-thread-5] c.e.b.b.s.api.v1.lock.LockService        : Thread 4 자원 진입 시도
2024-12-12 13:15:58  INFO [,,,] 48755 --- [pool-4-thread-6] c.e.b.b.s.api.v1.lock.LockService        : Thread 5 자원 진입 시도
2024-12-12 13:15:58  INFO [,,,] 48755 --- [pool-4-thread-7] c.e.b.b.s.api.v1.lock.LockService        : Thread 6 자원 진입 시도
2024-12-12 13:15:58  INFO [,,,] 48755 --- [pool-4-thread-8] c.e.b.b.s.api.v1.lock.LockService        : Thread 7 자원 진입 시도
2024-12-12 13:15:58  INFO [,,,] 48755 --- [pool-4-thread-9] c.e.b.b.s.api.v1.lock.LockService        : Thread 8 자원 진입 시도
2024-12-12 13:15:58  INFO [,,,] 48755 --- [ool-4-thread-10] c.e.b.b.s.api.v1.lock.LockService        : Thread 9 자원 진입 시도
2024-12-12 13:15:59  INFO [,,,] 48755 --- [pool-4-thread-1] c.e.b.b.s.api.v1.lock.LockService        : Thread 0 자원 사용 완료!
2024-12-12 13:15:59  INFO [,,,] 48755 --- [pool-4-thread-2] c.e.b.b.s.api.v1.lock.LockService        : Thread 1 자원 사용 완료!
2024-12-12 13:15:59  INFO [,,,] 48755 --- [pool-4-thread-3] c.e.b.b.s.api.v1.lock.LockService        : Thread 2 자원 사용 완료!
2024-12-12 13:15:59  INFO [,,,] 48755 --- [pool-4-thread-4] c.e.b.b.s.api.v1.lock.LockService        : Thread 3 자원 진입 완료!
2024-12-12 13:15:59  INFO [,,,] 48755 --- [pool-4-thread-5] c.e.b.b.s.api.v1.lock.LockService        : Thread 4 자원 진입 완료!
2024-12-12 13:15:59  INFO [,,,] 48755 --- [pool-4-thread-6] c.e.b.b.s.api.v1.lock.LockService        : Thread 5 자원 진입 완료!
2024-12-12 13:16:00  INFO [,,,] 48755 --- [pool-4-thread-4] c.e.b.b.s.api.v1.lock.LockService        : Thread 3 자원 사용 완료!
2024-12-12 13:16:00  INFO [,,,] 48755 --- [pool-4-thread-6] c.e.b.b.s.api.v1.lock.LockService        : Thread 5 자원 사용 완료!
2024-12-12 13:16:00  INFO [,,,] 48755 --- [pool-4-thread-5] c.e.b.b.s.api.v1.lock.LockService        : Thread 4 자원 사용 완료!
2024-12-12 13:16:00  INFO [,,,] 48755 --- [pool-4-thread-7] c.e.b.b.s.api.v1.lock.LockService        : Thread 6 자원 진입 완료!
2024-12-12 13:16:00  INFO [,,,] 48755 --- [pool-4-thread-8] c.e.b.b.s.api.v1.lock.LockService        : Thread 7 자원 진입 완료!
2024-12-12 13:16:00  INFO [,,,] 48755 --- [pool-4-thread-9] c.e.b.b.s.api.v1.lock.LockService        : Thread 8 자원 진입 완료!
2024-12-12 13:16:01  INFO [,,,] 48755 --- [pool-4-thread-8] c.e.b.b.s.api.v1.lock.LockService        : Thread 7 자원 사용 완료!
2024-12-12 13:16:01  INFO [,,,] 48755 --- [pool-4-thread-7] c.e.b.b.s.api.v1.lock.LockService        : Thread 6 자원 사용 완료!
2024-12-12 13:16:01  INFO [,,,] 48755 --- [pool-4-thread-9] c.e.b.b.s.api.v1.lock.LockService        : Thread 8 자원 사용 완료!
2024-12-12 13:16:01  INFO [,,,] 48755 --- [ool-4-thread-10] c.e.b.b.s.api.v1.lock.LockService        : Thread 9 자원 진입 완료!
2024-12-12 13:16:02  INFO [,,,] 48755 --- [ool-4-thread-10] c.e.b.b.s.api.v1.lock.LockService        : Thread 9 자원 사용 완료!
```      
       
> 실행결과를 보면 동시에 3개의 Thread 만 자원 진입을 하고 있는 내용을 볼 수 있다. 즉, 제한된 수의 리소스 동시 접근제어를 한다.    


### 결론
자바에서 동시성을 제어하는데 필수적인 동기화 메커니즘이며 이러한 동기화 메커니즘을 통해 멀티스레딩 환경에서 데이터의 일관성과 무결성을 보장할 수 있다.      
또한 뮤텍스와 세마포어를 쉽게 간추려서 화장실로 예시를 들어 설명하자면       
- 뮤텍스: 화장실을 이용하는 사람은 프로세스 or 스레드이며 화장실은 공유자원, 화장실 키는 공유자원에 접근하기 위한 오브젝트이다.
  - Toilet <- 사람(화장실 키 O) <- 대기열 <- 대기하는 사람(화장실 키 X)    
- 세마포어: 화장실이 공유자원이면 사람들이 스레드 or 프로세스 그리고 화장실 빈 칸의 개수는 현재 공유자원에 접근할 수 있는 스레드 or 프로세스의 개수    
  - Toilet 3개 <- 대기하는 사람들이 빈 칸으로 들어가는 구조


### 스핀락 (Spinlock)은 무엇이며 뮤텍스와 스핀락의 차이점?
스핀락 (Spinlock) 이란 임계영역이 언락되어 진입이 가능해질 때까지 루프를 돌면서 재시도하여 스레드가 CPU를 점유하고 있는 상태이다. (Busy Waiting 상태) 또한 스핀락은 운영체제 스케쥴링 지원을 받지 않기 때문에, 해당 스레드에 대한 문맥교환(Context Switch) 이 일어나지 않는다.       
      
임계영역이 짧은 시간 안에 언락되어 진입이 가능하면 Context Switch 가 일어나질 않기 때문에 효율적이나 임계영역이 오랜 시간동안 연락되지 않으면 그 시간동안 계속 CPU 를 점유하게 되어 다른 스레드가 사용하지 못해 오버헤드가 발생한다. 그래서 스핀락은 Context Switching 가 일어나지 않아 멀티 프로세서 시스템에서만 사용이 가능하다.

뮤텍스는 상태가 오직 획득(Lock) / 해제(Unlock)만 존재한다는 점은 스핀락과 동일하다. 하지만 스핀락이 임계영역이 언락되어 권한을 획득하기까지 Busy Waiting 상태를 유지한다면, 뮤텍스는 Sleep 상태로 들어갔다 Wakeup 되면 다시 권한 획득을 시도한다. 뮤텍스의 경우에는 Locking 메커니즘으로 락을 걸은 스레드만이 임계영역을 나갈 때 락을 해제할 수 있습니다. 시스템 전반의 성능에 영향을 주고 싶지 않고 길게 처리해야하는 작업인 경우에 주로 사용된다. 주로 스레드 작업에서 많이 사용된다. (Busy Waiting 이란 원하는 자원을 얻기 위해 기다리는 것이 아니라 권한을 얻을 때까지 계속 확인하는 것 즉, 스핀락은 Busy Waiting 으로 인하여 시스템에 행이 걸릴 수 있다. 이것은 멀티 프로세서 환경에서 사용하는 것이 좋으며, 짧은 시간의 연산에 대해 사용할 경우 성능이 좋다. 그 외 뮤텍스나 세마포어는 오버헤드가 큰 상황에서 더 좋다.)


### 참고
- https://ko.wikipedia.org
- https://medium.com/@kwoncharles/
- https://notavoid.tistory.com
- https://velog.io/@deannn/
- https://ko.wikipedia.org/