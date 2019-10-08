---
layout: post
title : "Vuejs를 선택한 이유"
subtitle : "2019-10-08-what-vuejs.md"
date: 2019-10-08 10:00:00
author: "전봉근"
header-img: "img/posts/android-bg.jpg"
comments: true
tags: [vuejs, vue, javascript]
---

# 선택조건
- 낮은 Learning Curve
- 성능
- 안정적
- 큰 커뮤니티 또는 큰 단체에서 지원하며 문서화가 잘되어 있거나 레퍼런스 또한 많아야 한다.


# Vue 특징
- 낮은 Learning Curve
  - 단순한 구성 요소
  - 기존 자바스크립트 지식만으로 충분
  - 컴포넌트 단위 관리
- [가능성](https://trends.google.com/trends/explore?date=today%205-y&q=angular.js,react.js,vue.js,angular2)
- github
   - 2017-06-30
     ![what-vuejs-1](/img/posts/javascript/vuejs/what-vuejs-1.jpg)
     ![what-vuejs-2](/img/posts/javascript/vuejs/what-vuejs-2.jpg)
     ![what-vuejs-3](/img/posts/javascript/vuejs/what-vuejs-3.jpg)
   - 현재 ( vuejs는 역사가 짧음에도 엄청난 성장을 보여주고 있다. )
     ![what-vuejs-4](/img/posts/javascript/vuejs/what-vuejs-4.png)
     ![what-vuejs-5](/img/posts/javascript/vuejs/what-vuejs-5.png)
     ![what-vuejs-6](/img/posts/javascript/vuejs/what-vuejs-6.png)
- [Model-View 양방향 바인딩 지원](https://velog.io/@skyepodium/Vue.js-v-model%EA%B3%BC-syntax-sugar-7ajrp44tnq)
  - 두 데이터 혹은 정보의 소스를 모두 일치시키는 기법
- Component-Based ( 단일 파일 Component -> 유지보수 용이하며 개발속도가 빠름 )
- Virtual DOM 을 사용하여 빠른 렌더링
  - DOM의 복사본을 메모리 내에 저장하여 사용하며 변경 사항을 "가상의" 위치에서 처리하므로 "실제DOM"의 조작을 최소화한다.
- SPA (Single Page Application)
  - 사용자 친화적(빠른 반응성, 화면전환 등)
  - 변경되는 부분만 데이터를 받아서 렌더링 하기 때문에 속도가 빠르며 서버는 불필요한 코드가 계속 요청되는 일이 없기 때문에 처리량과 트래픽이 적어짐

> 즉, Vue는 Angular의 양방향 데이터 바인딩 특성과 react의 가상 돔 기반 렌더링 장점을 적용  

  
<br><br>
## 참고 사이트
https://velog.io/@skyepodium
https://sungjk.github.io