---
layout: post
title : "Spinnaker"
subtitle : "2022-08-26-spinnaker.md"
date: 2022-08-26 18:50:00
author: "전봉근"
header-img: "img/posts/android-bg.jpg"
comments: true
tags: [devops, cicd, infra]
---

# Spinnaker
## Spinnaker 란?
Spinnaker 는 `넷플릭스`에서 개발하고 구글에서 확장한 `오픈 소스화한 멀티 클라우드를 지원`하는 `CD(=Continuous Delivery) 플랫폼`이다. 구글 클라우드, 아마존, 마이크로소프트 등 `대부분의 메이져 기업의 클라우드를 지원하며 Kubernetes 또는 Openstack 과 같은 오픈소스 기반의 클라우드 또는 컨테이너 플랫폼을 동시 지원한다.`

### Spinnaker 아키텍처  
![spinnaker-3](/img/posts/devops/spinnaker-3.png)    
출처: https://www.tothenew.com/
- Deck: Deck 컴포넌트는 UI 컴포넌트로, Spinnaker의 UI 웹사이트 컴포넌트이다.
- Gate: Spinnaker는 MSA 구조로, 모든 기능을 API 로 Expose 한다, Gate는 API Gateway로, Spinnaker의 기능을 API로 Expose 하는 역할을 한다.
- Clouddriver: 여러 클라우드 플랫폼에 명령을 내리기 위한 어댑터 역할을 한다.
- Orca: Oraca는 이 모든 컴포넌트를 오케스트레이션하여, 파이프라인을 관리해주는 역할을 한다. 
- Rosco: Rosco는 Bakering 컴포넌트로, Spinnaker는 VM또는 Docker 이미지 형태로 배포하는 구조를 지원하는데, 이를 위해서 VM이나 도커 이미지를 베이커링(굽는) 단계가 필요하다. Spinnaker는 Packer를 기반으로 하여 VM이나 도커 이미지를 베이커링 할 수 있는 기능을 가지고 있으며, Rosco가 이 기능을 담당 한다.
- Front50: 파이프라인이나 기타 메타 정보를 저장하는 스토리지 컴포넌트이다.
- Igor: Spinnaker는 Jenkins CI 툴과 연동이 되는데, Jenkins에서 작업이 끝나면, Spinnaker Pipeline을 Invoke 하는데, 이를 위해서 Jenkins의 작업 상태를 Polling을 통해서 체크한다. Jenkins의 작업을 Polling으로 체크 하는 컴포넌트가 Igor이다.
- Echo: 외부 통신을 위한 Event Bus로, 장애가 발생하거나 특정 이벤트가 발생했을때, SMS, Email 등으로 notification을 보내기 위한 Connector라고 생각하면 된다.
- Rush: Spinnaker에서 사용되는 스크립트를 실행하는 스크립트 엔진이다.

### Spinnaker 구성요소  
![spinnaker-4](/img/posts/devops/spinnaker-4.png)    
출처: https://www.tothenew.com/
> 상기 이미지는 Spinnaker 가 리소스를 관리하기 위해서, 리소스에 대한 계층구조를 정의하고 있으며 가장 최상위에는 Project, 다음은 Application, Application 마다 Cluster Service 를 가지며 각 Cluster Service 는 Server Group 으로 구성된다.
- 계층별 각각의 개념을 파악해보자.
  - Server Group: 동일한 서버(같은 VM과 애플리케이션)으로 이루어진 서버군이다. (웹 서버 그룹이나 이미지 업로드 서버 그룹식으로 그룹을 잡을 수도 있고, 이미지 서버 그룹에 따라 버전별로 잡는등 유연하게 서버 군집의 구조를 정의할 수 있다.)
  - 이러한 Server Group 들을 Cluster 라는 단위로 묶일 수 있다.


## Spinnaker 장점
- release pipeline 또는 deployment pipeline 에 대해 시각화가 잘 되어있다.
- pipeline 에 대한 프로그래밍이 쉽다.
- 배포전략을 다양하게 지원 
  - `Red/Black (Blue/Green 와 동일): 테스트해보고 트래픽을 한번에 전환함. 롤백이 빠름`
     - 동일한 양의 instance로 이루어진 새로운 Server Group을 생성
     - 신규 Server Group이 정상상태가 되면 LB는 신규 Server Group에 트래픽을 분산함. 
     > Blue 구역은 실제 운영서버이고, Green 구역에서 신규버전 테스트를 하고, 문제가 없으면 Blue 구역에 향하던 Request 를 Green 구역으로 향하게함. 그러면 이제 Green 구역이 실 운영서버가 되고 Blue 구역은 테스트서버가 됨. (구역 swap)
  - `Rolling Red/Black: Blue/Green 과 동일하나 단, 인스턴스별 또는 그룹별로 Rolling`
  - `Canary: 트래픽을 분산시켜 전환함`
    - 가장 작은 개수의 인스턴스를 교체시키고
    - 새로운 버전으로 트래픽을 분산시킨다 (1~5%)
    - 새로운 버전에 이슈가 없을때까지 테스트를 진행하고
    - 특정시간까지 이슈가 없으면 배포를 늘려간다.    


## Spinnaker 사용
### Clusters
![spinnaker-1](/img/posts/devops/spinnaker-1.png)    
> Spinnaker 를 실행하면 상기 이미지와 같이 배포된 Cluster 환경을 한 눈에 볼 수 있다. (초록색 칸 하나가 Pod 하나라고 보면 된다.)

### Piplines
![spinnaker-2](/img/posts/devops/spinnaker-2.png)    
출처: https://www.tothenew.com/
> Pipline 은 상기 이미지와 같이 workflow 형태로 구성이 가능하다.

파이프라인에서 지원하는 단계에 대해 알아보자.
- Bake: VM 이미지를 생성
- Find: 기존 클러스터에서 이전에 구운 이미지를 찾을 수 있다.
- Deploy: VM 이미지 또는 컨테이너를 클러스터에 배포한다.
- Destroy Server Group: 인스턴스 및 auto-scaling-groups 등을 종료하여 서버 그룹을 파괴
- Disable Server Group: 기존 활성된 서버 그룹을 비활성화 한다.
- Enable Server Group: 비활성화된 서버 그룹 활성화
- Jenkins: Jenkins Job 을 실행
- Manual Judgement: 사용자한테 입력을 받아 파이프라인 실행 여부를 결정
- Pipeline: 다른 파이프라인을 수행
- WebHook: HTTP 로 다른 시스템을 호출한다. 통상적으로 HTTP REST API 를 호출함
- Resize server group: 서버 그룹 크기를 조정
- Script: 쉘 스크립트 실행
- Shrink cluster: 선택한 클러스터를 축소
- Wait: 지정된 시간 동안 대기를 할 수 있다.


## 참고
- https://bcho.tistory.com/
- https://berrrrr.github.io/
- https://spinnaker.io/
- https://www.tothenew.com/