---
layout: post
title: "Redis 관련 내용 정리"
subtitle: "2024-10-04-redis-document.md"
date: 2024-10-04 20:40:00
author: "전봉근"
header-img: "img/posts/android-bg.jpg"
comments: true
tags: [redis]
---

## Redis 관련 내용 정리

#### 대표적인 구조
- Look Aside Cache (보편적으로 사용)
  - Client -> Application -> Cache 에 데이터가 있으면 Cache 에서 가져옴 만약 없으면 DB 에서 데이터를 읽어오고 해당 데이터를 Cache 에 저장
- Write Back
  - Client -> Application -> Cache 에 먼저 데이터를 저장하고 특정 시점마다 DB 에 저장
  - 이렇게 하면 DB 에 저장될 때 건별로 Insert 쿼리를 날리는 것 보다 한 번에 쿼리를 날리다보니 성능에 용이

#### 사용사례
- 여러 서버들이 데이터를 공유할 때
- 인증 토큰 등을 저장
- Ranking(Sorted Set)
- API
- Queue  

#### Redis Collections
- Strings
  - 단일 Key
    - GET <Key>
    - SET <Key> <Value>
  - 멀티 Key
    - MSET <Key1> <Value1> <Key2> <Value2> ....
    - MGET <Key1> <Key2> ....
- List (Job Queue 용도로 많이 쓰임)
  - insert
    - LPUSH <Key>
    - RPUSH <Key>
  - pop
    - KEY
    - LPOP <Key>
    - RPOP <Key>
    - BRPOP <Key>
      - 누가 데이터를 Push 하기 전까지 대기
- Set (특정 유저를 Follow 하는 목록을 저장할 때 사용)
  - SADD
    - value 가 이미 Key 에 있으면 추가 X
  - SMEMBERS
    - 모든 Value 를 반환 (전체를 불러오는 것이므로 주의)
  - SISMEMBER
    - value 가 존재하면 1, 없으면 0
- Sorted Set (유저 랭킹 보드를 사용할 수 있음, Sorted set 의 score 는 double 이므로 값이 정확하지 않을 수 있다.)
  - ZADD <Key> <Score> <Value>
    - Value 가 이미 Key 에 있으면 해당 Score 로 변경
  - ZRANGE <Key> <StartIndex> <EndIndex>
    - 해당 Index 범위 값을 모두 돌려줌
    - Zrange testkey 0 -1
      - 모든 범위를 가져옴
  - 그 외 쿼리예시
    - 쿼리예시
      ```
      // zrange rank 50 70
      SELECT * FROM rank ORDER BY score LIMIT 50, 20;

      // zrevrange rank 50 70
      SELECT * FROM rank ORDER BY score desc LIMIT 50, 20;

      // zrangebyscore rank 70 100
      SELECT * FROM rank WHERE score >= 70 AND score < 100;

      // zrangebyscore rank(70 +inf) -> +inf 는 무한대 즉, infinity
      SELECT * FROM rank WHERE scroe > 70;
      ```
- Hash
  - HMSET <key> <subkey1> <value1> <subkey2> <value2>
  - HGETALL <key>
    - 해당 key 의 모든 subkey 와 value 를 가져옴
  - HGET <key> <subkey>
  - HMGET <key> <subkey1> <subkey2> ....

#### Collection 주의 사항
- 하나의 컬렉션에 너무 많은 아이템을 담으면 좋지 않음
  - 10000 개 이하 몇 천개 수준으로 유지하는게 좋음
- Expire 는 Collection 의 item 개별로 걸리지 않고 전체 Collection 에 대해서만 걸림

#### Redis 운영
- 메모리 관리
  - Physical Memory 이상을 사용하면 문제 발생
    - Swap 이 있다면 Swap 사용으로 해당 메모리 Page 접근시 마다 늦어짐
    - Swap 이 없다면? O(N) 이런 명령들은 죽을 것 이다.
  - Maxmemory 를 설정하더라도 이보다 더 사용할 가능성이 큼
  - RSS (Resident Set Size) 값을 모니터링 해야 함 (Resident Set Size: 프로세스가 차지하는 메모리)
  - 큰 메모리를 사용하는 instance 하나보다는 적은 메모리를 사용하는 instance 여러개가 안전함
    - 24 GB instance -> 8 GB instance * 3
    - master/slave 구성에서 Write 가 많은 Redis 는 최대 메모리를 2배까지 사용할 수 있다. 처음 fork 를 할 경우 copy on write 가 동작하면서 Read 일 경우는 복사를 안하나 Write 가 일어날 때 메모리를 복사해서 메모리를 더 사용해야 되기 때문
  - Redis 는 메모리 파편화가 발생할 수 있음. 4.x 부터 파편화를 줄이는 기능이 들어갔으나 jemalloc 버전에 따라서 다르게 동작할 수 있음
  - 3.x 버전의 경우 실제 used memory 는 2GB 로 보고가 되지만, 11GB 의 RSS 를 사용하는 경우가 자주 발생
  - `다양한 사이즈를 가지는 데이터 보다는 유사한 크기의 데이터를 가지는 경우가 유리`
- 메모리가 부족할 때
  - 좀 더 메모리 많은 장비로 Migration
  - 있는 데이터 줄이기 (다만 이미 Swap 을 사용중이라면, 프로세스를 재시작 해야함)
- 메모리를 줄이기 위한 설정
  - 기본적으로 Collection 들을 다음과 같은 자료구조를 사용
    - Hash -> HashTable 을 하나 더 사용
    - Sorted Set -> Skiplist 와 HashTable 을 이용
    - Set -> HashTable 사용
    - 해당 자료구조들은 메모리를 많이 사용함
  - 한 컬렉션에 여러 아이템 리스트를 사용한다면 Ziplist 를 사용하는게 속도는 느려지지만 메모리를 적게쓰는 방법이 있다.
    - 원래 쓰는 자료구조대신에 ziplist 를 사용하도록 설정만 바꿔서 사용할 수 있다.
    - ziplist 구조
      - In-Memory 특성 상, 적은 개수라면 선형 탐색을 하더라도 빠름
      - List, hash, sorted set 등을 ziplist 로 대체해서 처리를 하는 설정이 존재 (단, 설정값 보다 많은 아이템이 존재하면 원래 자료구조로 바뀌게 되어서 속도는 유지하지만 메모리를 더 많이 사용)
        - hash-max-ziplist-entries, hash-max-ziplist-value
        - list-max-ziplist-size, list-max-ziplist-value
        - zset-max-ziplist-entries, zset-max-ziplist-value
- O(N) 관련 명령을 주의
  - Redis 는 Single Thread 이고 단순한 get/set 의 경우 초당 10만 TPS 이상 가능 (CPU 속도에 따라 다름)
  - Redis 는 ProcessInputBuffer 에서 packet 을 하나로 command 를 만들어서 완성되었는지 확인하고 완성이되면 실행 즉, 해당 작업이 처리되는 동안에는 뒤에 packet 이 쌓일 것이다.
  - 주의할 명령
    - KEYS
      - Ex: 모니터링 스크립트가 초당 한번씩 keys 를 호출하는 경우
      - `scan` 명령으로 대체가 가능 (커서 방식이므로 짧게 여러번 돌리는걸로 구현)
        ```
        redis 127.0.0.1:6379> scan 0
        ```
    - FLUSHALL, FLUSHDB
    - Delete Collections
    - Get ALL Collections
      - 아이템이 몇만개 든 hash, sorted set, set 에서 모든 데이터를 가져오는 경우
        - Collection 의 일부만 가져옴
        - 큰 Collection 을 작은 여러개의 Collection 으로 나누어서 저장
          - Ex) Userranks -> Userranks1, Userranks2, Userranks3 (하나당 몇천개 안쪽으로 저장하는게 좋음)
    - `Spring Security Oauth RedisTokenStore` 예전 버전에서 문제가 발생했었다.
      - List(O(N)) 자료구조를 통해서 이루어지기 때문
        - 검색, 삭제시에 모든 item 을 매번 찾아야됨 (100 만개쯤 되면 전체 성능에 영향을 줌)
        - `현재 해결된 버전은 Set(O(1)) 을 이용해서 검색, 삭제를 하도록 수정되어 있음`

#### Redis Replication
- Async Replication
  - Replication Lag 이 발생할 수 있다.
- DBMS 로 보면 statement replication 과 유사 즉, 쿼리로 보냄
  - 주의할점은 만약 now() 라는 걸 사용하면 primary 와 secondary 와 저장되는 시점이 다르기 때문에 데이터가 다를 수 있다.
- Replication 설정 과정
  - secondary 에 replicaof or slaveof 명령을 전달
  - secondary 는 primary 에 sync 명령 전달
  - primary 는 현재 메모리 상태를 저장하기 위해 Fork
  - Fork 한 프로세서는 현재 메모리 정보를 disk 에 dump
  - 해당 정보를 secondary 에 전달
  - Fork 이후의 데이터를 secondary 에 계속 전달
- Replication 주의할 점



55:12