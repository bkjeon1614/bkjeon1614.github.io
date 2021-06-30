---
layout: post
title : "AWS RDS MySQL VS Aurora"
subtitle : "2021-06-30-aws-rds-mysql-vs-aurora.md"
date: 2021-06-30 10:00:00
author: "전봉근"
header-img: "img/posts/android-bg.jpg"
comments: true
tags: [aws, db, mysql, aurora]
---

# AWS RDS MySQL VS Aurora
----------------------------------------------------------------

Aurora란 AWS가 MySQL과 Postgresql을 호환해서 만든 RDBMS이다. 둘의 가장 큰 차이점은 Storage이며 `Aurora는 Shared Storage를 사용`하며 `MySQL은 Binary Log 기반의 Replication 기반이 아닌 Storage와 Page 기반의 Replication을 사용`
- Aurora
  - 장점
    - 기본적으로 MySQL 및 PostresSQL 호환 가능하다.
    - 스토리지 용량이 64TB까지 자동 증가된다. (RDS MySQL은 EBS 볼륨 할당을 직접 해야함)
    - 3개의 AZ (가용 영역)에 대해서 6방향 복제를 지원한다 (6개의 스토리지 지원)
    - Read Replica (읽기 전용 복제본)을 15개까지 지원한다
    - Read Replica에 대한 지연 시간이 RDS MySQL 대비 짧다
  - 단점
    - 비용이 RDS MySQL 대비, 약 20~30% 정도 비쌈 (항상 그렇진 않고 케이스별로 다름)
    - MySQL 버전은 최신 버전이 아니다. (RDS MySQL 버전이 더 최신 버전 지원)
    - `Patch로 인한 Downtime 발생`
      - OS, Security Patch가 발생할 경우 Downtime 불가피하다 Patch는 강제사항이며 Patch시간을 선택하여 진행하여야 한다
      - Shared Storage 라는 특성 때문에 Writer 인스턴스가 재시작되고 나서 곧바로 Reader 인스턴스들도 동시에 재시작 되므로 `Failover 가 발생하지 않고 재시작만 되기 때문에 Downtime이 발생한다` (Amazon Aurora MySQL 5.7부터 제로 다운타임 패치 지원한다고 한다.)

- 차이점
  - Storage: RDS MySQL은 자체 EBS로 운영하지만 Aurora MySQL은 Shared Storage를 사용
  - 관리주체: RDS MySQL은 관리자가 RDS MySQL의 버전을 올리면서 사용하지만 Aurora MySQL은 AWS가 개발해서 버전 업그레이드를 주기적으로 하기 때문에 optional 또는 mandatory가 AWS에 의해 정해질 수 있다.
  - Read Replica 구성: RDS MySQL은 standby와 read replica 만들때 binary log를 사용하지만 Aurora의 경우 내부 storage 및 redo log 전송을 통해 빠른 동기화가 가능하며 bandwidth를 줄일 수 있다.