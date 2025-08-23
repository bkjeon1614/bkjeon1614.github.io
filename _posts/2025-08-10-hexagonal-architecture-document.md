---
layout: post
title: "헥사고날 아키텍처의 이해 (작성중)"
subtitle: "2025-08-10-hexagonal-architecture-document.md"
date: 2025-08-10 20:40:00
author: "전봉근"
header-img: "img/posts/android-bg.jpg"
comments: true
tags: [cs, architecture]
---

#### 헥사고날 아키텍처란
헥사고날 아키텍처(Hexagonal Architecture) 는 클린 아키텍처의 구현 방식 중 하나로 포트와 어댑터 아키텍처(Ports and Adapters Architecture 는 소프트웨어 아키텍처 중 하나로, Alistair Cockburn에 의해 제안되었고, `애플리케이션 핵심 로직(Core Domain)` 과 `외부 의존성(데이터베이스, UI, 메시지 브로커 등)` 을 명확히 분리하는 것을 목표로 한다.


#### 장점
- 의존성 역전
   - Core 는 외부 기술에 의존하지 않고, 추상화(Port) 에만 의존
- 테스트 용이성
  - 외부 환경 없이도 Core 로직을 단위 테스트 가능
- 유연한 교체 가능성
  - DB, 메세지 브로커, 외부 API 등 변경 시 Adapter 만 수정하면 됨
- 관심사 분리
  - 비즈니스 로직과 인프라 코드의 경계를 명확히 구분


#### 단점
- 구조가 복잡해져서 작은 프로젝트에는 과도할 수 있음
- 추상화 계층이 많아져 초기 구현 비용이 증가


> 헥사고날 아키텍처는 비즈니스 로직 중심으로 설계하게 해주며, 외부 환경의 변화에 강한 구조를 제공하며 특히 대규모 서비스나 장기간 유지보수가 필요한 프로젝트에서 용이하다.





#### 참고
- https://alistair.cockburn.us/hexagonal-architecture
- https://en.wikipedia.org/wiki/Hexagonal_architecture_(software)
- https://www.baeldung.com/hexagonal-architecture
- https://www.thoughtworks.com/radar/techniques/ports-and-adapters