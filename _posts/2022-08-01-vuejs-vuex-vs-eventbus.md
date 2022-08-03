---
layout: post
title : "[Vuejs] Vuex VS Eventbus"
subtitle : "2022-08-01-vuejs-vuex-vs-eventbus.md"
date: 2022-08-01 20:00:00
author: "전봉근"
header-img: "img/posts/android-bg.jpg"
comments: true
tags: [javascript, vuejs, frontend, client]
---

# Vuex VS Eventbus

## Props? Emit? 그리고 Vuex, Eventbus
Vue 에서 컴포넌트간 통신은 `props` 와 `emit` 을 통하여 전달한다.
먼저 비교하기전에 props 와 emit 에 대해 모르는 사람들이 있을 수 있으니 간단하게 정리하자면 아래와 같다.
- props: 상위 컴포넌트의 데이터를 하위 컴포넌트에 전달하는 특성이며 하위 컴포넌트에서 전달받기 위해서는 props 를 명시적으로 선언해야 한다.
- emit: 최상위 컴포넌트가 하나 이상인 경우, 이벤트를 직접 컴포넌트에 할당하는 것을 의미

앞서 말했듯이 props 와 emit 이 많아지면 관리가 복잡해져 사이드 이펙트가 발생할 확률이 높아지므로 이러한 문제점을 해결하기 위하여 주로 상태 관리 패턴 라이브러리를 활용하며 Vue 에서는 보편적으로 `Vuex` 나 `Eventbus` 를 활용한다.
주로 전역으로 데이터가 전달이 가능하며 수신 방법이 간단하다는 대표적인 장점이 있지만 무분별한 선언으로 리소스를 낭비한다던가 전역 범위로 사용되므로 데이터 충돌 등의 이슈가 발생할 수 있어 주의하여 사용해야 한다.

## 되도록 Vuex 를 사용하는편이 좋다.
Vuex 와 Eventbus 중 왠만하면 Vuex 를 사용하는것이 좋다.
보통 이러한 관련 포스팅을 찾아보면 애플리케이션 복잡성에 따라 규모가 크면 vuex, 작으면 event bus 를 사용하는 것이 좋다고 나와있는데 ([https://v2.vuejs.org/v2/style-guide/?redirect=true#Non-flux-state-management-use-with-caution](https://v2.vuejs.org/v2/style-guide/?redirect=true#Non-flux-state-management-use-with-caution)) 이러한 내용이 틀린 내용은 아니지만
왜 Vuex 를 사용해야 되는지 알아보자.
- [https://github.com/vuejs/rfcs/blob/master/active-rfcs/0020-events-api-change.md](https://github.com/vuejs/rfcs/blob/master/active-rfcs/0020-events-api-change.md) 참고하면 Vue3 에서는 공식적으로 EventBus 를 지원하지 않는다고 되어있다.
- 또한 [Vue 공식 사이트(영문)](https://vuejs.org/) 에서 EventBus 에 대한 기록이 없다. (한글문서에서는 아직 존재함)
- 상기 내용의 애플리케이션의 복잡성에 따라 사용여부를 선택하는 부분에 의하면 결국 이 부분에 의하여 특별히 큰 이익을 얻는 것도 아니며 또한 시작은 작은규모여도 결국 마지막은 큰 규모의 애플리케이션이 될 수 있으며 그 때 포팅하기엔 이미 $emit, $on, $off 로 떡칠이 되어있을 것이므로 초반부터 vuex 로 잡고 가는것이 좋을 것 같다고 생각한다.
- Vuex 는 `Flux 패턴` 의 단방향 데이터 흐름의 구조이므로 한 눈에 파악하기 쉬우며 Event Bus 는 listener 제거를 안했을 때의 문제나 또한 어디서 호출하는지 흩어져 있으므로 가독성과 데이터 흐름을 파악하기 어려워 많아질 경우 복잡하게 된다.

> 결론적으론 Event Bus 는 권장하지않으며 공식문서에도 존재하지 않는다. 즉, Vuex 를 활용하는게 탁월한 선택일 것 같다.

참고
- https://kdydesign.github.io/
- https://pozafly.github.io/