---
layout: post
title : "Portainer 설치"
subtitle : "2020-01-16-portainer.md"
date: 2020-01-16 17:30:00
author: "전봉근"
header-img: "img/posts/android-bg.jpg"
comments: true
tags: [docker, container]
---

## Portainer 설치
----------------------------------------------------------------

[Portainer](https://www.portainer.io/)는 Docker를 Web에서 관리할 수 있는 툴이다. 즉, UI를 통하여 쉽게 Container를 관리할 수 있다.

> 시작하기전에 Docker가 먼저 설치되어 있어야 한다.

먼저 Portainer에서 사용할 Volume를 생성
```
docker volume create portainer_data
```

컨테이너 생성 후 실행
```
// linux
docker run -d -p 9000:9000 -v /var/run/docker.sock:/var/run/docker.sock -v portainer_data:/data --restart=always portainer/portainer
```

> 위의 옵션 중 --restart=always 을 주면 추후 docker를 재시작 했을 경우에도 자동으로 구동된다.
   