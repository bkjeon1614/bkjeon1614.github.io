---
layout: post
title : "Javascript var, let, const 차이"
subtitle : "2022-08-09-Javascript-var-let-const"
date: 2022-08-09 22:06:00
author: "전봉근"
header-img: "img/posts/android-bg.jpg"
comments: true
tags: [javascript]
---

Javascript var, let, const 개념 및 차이점
=========

## Key Point
- var, let: 변수를 선언하는 키워드이다.
- const: 상수를 선언하는 키워드이다.

Example Code
```
    // 1. var는 값을 재 할당 가능하다.
    var name = "test";
    name = "test2";

    // 2. let은 var처럼 재 할당이 가능하다.
    let score = "1";
    score = "2";

    // 3. const는 값 재할당이 불가능하며 선언과 동시에 리터럴 값을 할당해야 한다.
    const PI = "3.14";
```

> let, const는 ECMA6에 도입된 키워드이며 var 타입으로 인한 혼동을 방지하기 위하여 만들어 졌다. var 타입의 혼동이 일어나는 이유에 대해선 아래를 참고하자.

1. var는 변수명을 재 선언해도 아무런 문제가 발생하지 않는다.
   ```
       var title = "테스트타이틀";

       // 위에 title을 선언했지만 깜박하여 재 선언해도 아무런 문제가 생기지 않는 크리티컬한 문제점이 있다.
       var title = "재선언타이틀"; 
   ```

2. 1번과 다르게 let은 변수명을 재 선언시에 에러를 발생시킨다.
   ```
       var title = "테스트타이틀";

       // SyntaxError를 발생 즉, 이미 선언된 변수라는것을 알 수 있다.
       var title = "재선언타이틀"; 
   ```   


## var, let, const 차이점 및 호이스팅이란
- function-scoped: var
- block-scoped: let, const
- 호이스팅: 인터프리터가 변수와 함수의 메모리 공간을 선언 전에 미리 할당하는 것을 의미
  ```
  concatStr();

  function concatStr(str1, str2) {
    return str1 + str2;
  }
  ```
  > 쉽게말해 선언된적이 없는 것으 참조하려 할 때 즉, `concatStr`을 실행 시점에 `function concatStr()` 이 선언되어 있지 않으므로 에러가 나야되지만 정상동작한다. 왜냐하면 `function concatStr` 함수를 상단으로 올려서 참조할 수 있게 해줬기 때문이다. 이를 `호이스팅` 이라고 한다. 또한 함수 선언은 동시에 초기화가 이루어지기 때문에 참조 뿐만 아니라 실행도 가능하다.


### var (function-scoped)
```
// var는 function-scope 이기 때문에 for문이 끝난다음에 j를 호출하면 값이 출력이 잘 된다.
// 이건 var가 hoisting이 되었기 때문이다.
for(var j=0; j<10; j++) {
  console.log('j', j)
}
console.log('after loop j is ', j) // after loop j is 10


// 아래의 경우에는 에러가 발생한다.
function counter () {
  for(var i=0; i<10; i++) {
    console.log('i', i)
  }
}
counter()
console.log('after loop i is', i) // ReferenceError: i is not defined
```

호이스팅 문제를 해결하기 위해서는 항상 function 을 만들어서 호출해야 되는건가? 그건 아니고 javascript 에서는 [IIFE(=immediately-invoked function expression)](https://ko.wikipedia.org/wiki/%EC%A6%89%EC%8B%9C_%EC%8B%A4%ED%96%89_%ED%95%A8%EC%88%98) 로 function-scope 인 것 처럼 만들 수 있다.

```
// IIFE를 사용하면
// i is not defined가 뜬다.
(function() {
  // var 변수는 여기까지 hoisting이 된다.
  for(var i=0; i<10; i++) {
    console.log('i', i)
  }
})()
console.log('after loop i is', i) // ReferenceError: i is not defined
```

상기 코드가 이상한 부분이 있다. 뭐냐면 `IIFE`는 `function-scope` 처럼 보이게 만들어주지만 결과는 같지 않다.
하기 코드를 참고하면 에러없이 출력이 된다.

```
// 이 코드를 실행하면 에러없이 after loop i is 10이 호출된다.
(function() {
  for(i=0; i<10; i++) {
    console.log('i', i)
  }
})()
console.log('after loop i is', i) // after loop i is 10
```
상기 코드가 에러없이 출력되는 이유는 i가 키워드 없이 선언되었기 때문에 해당 변수는 암시적으로 전역 객체에 프로퍼티로 할당되기 때문이다.

상기 코드처럼 `IIFE`를 사용해도 `hoisting` 이 된다면 사용하는 의미가 없어진다. 그러므로 이런 `hoisting` 을 막기 위해 `use strict` 를 사용한다.
```
// 아까랑 다르게 실행하면 i is not defined라는 에러가 발생한다.
(function() {
  'use strict'
  for(i=0; i<10; i++) {
    console.log('i', i)
  }
})()
console.log('after loop i is', i) // ReferenceError: i is not defined
```
> 이러한 예시들을 보았을 때 변수선언 하나로 인하여 너무 많은 작업을 한다고 생각하게 될 것이다. 그래서 등장한게 `es2015` 에서 `let`, `const` 가 추가되었다.


### let, const (block-scoped)
`ES2015` 에서는 `let`, `const` 가 추가되었다. 기존 javascript 에서는 `var` 만 존재했기 때문에 아래와 같은 문제점이 존재했다.
```
// 이미 만들어진 변수이름으로 재선언했는데 아무런 문제가 발생하지 않는다.
var a = 'test'
var a = 'test2'

// hoisting으로 인해 ReferenceError에러가 안난다.
c = 'test'
var c
```
상기 코드와 같은 문제점이 있었으나 `let`, `const` 를 사용하면 `var`와 다르게 `변수 재선언`이 불가능하다.
또한 `let` 과 `const` 의 차이점은 변수의 `immutable` 여부이다. `let` 은 변수에 재 할당이 가능하지만 `const` 는 변수 재선언, 재할당 모두 불가능하다.
```
// let
let a = 'test'
let a = 'test2' // Uncaught SyntaxError: Identifier 'a' has already been declared
a = 'test3'     // 가능

// const
const b = 'test'
const b = 'test2' // Uncaught SyntaxError: Identifier 'a' has already been declared
b = 'test3'    // Uncaught TypeError:Assignment to constant variable.
```

여기서 `let`, `const` 가 `hoisting` 이 발생하지 않는건 아니며 `var`가 `function-scoped` 로 `hoisting` 이 되었다면 `let`, `const` 는 `block-scoped` 단위로 `hoisting` 이 일어나는데 아래 코드에서 `ReferenceError` 에러가 발생한 이유는 [tdz(temporal dead zone)](https://github.com/ajzawawi/js-interview-prep/blob/master/answers/es6/temporal-dead-zone.md) 때문이다. 즉, `let 은 값을 할당하기전에 변수가 선언 되어있어야 하는데 그렇지 않기 때문에 에러 발생`
```
c = 'test' // ReferenceError: c is not defined
let c
```
> TDZ (Temporal Dead Zone): 변수 선언 및 초기화 하기 전 사이의 사각지대 즉, TDZ에 영향을 받지 않으려면 초기화한 다음 변수 값을 출력하는 순서를 지켜야됨

또한 `const` 도 마찬가지이나 좀 더 엄격하다.
```
// let은 선언하고 나중에 값을 할당이 가능하지만
let dd
dd = 'test'

// const 선언과 동시에 값을 할당 해야한다.
const aa // Missing initializer in const declaration
```


## 참고
- https://github.com/ajzawawi/
- https://2ality.com/
- https://gist.github.com/LeoHeo