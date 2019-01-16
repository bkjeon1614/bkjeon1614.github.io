---
layout: post
title : "Jquery를 활용한 Loading Bar 구현"
subtitle : "2019-01-16-Jquery-Loding-Bar"
date: 2019-01-16 18:50:00
author: "전봉근"
header-img: "img/posts/android-bg.jpg"
comments: true
tags: [javascript]
---

Jquery를 활용한 Loading Bar 구현
=========
Jquery를 활용하여 Ajax와 같은 통신을 할 때 loading bar 표시와 hide를 할 수 있는 기능을 구현

### 로딩바 표시
```
function showLoadingBar() {
    var maskHeight = $(document).height();
    var maskWidth = window.document.body.clientWidth;

    var mask = "<div id='mask' style='position:absolute; z-index:9000; background-color:#000000; display:none; left:0; top:0;'></div>";
    var loadingImg = '';

    loadingImg += "<div id='loadingImg' style='position:absolute; left:50%; top:40%; display:none; z-index:10000;'>";
    loadingImg += "    <img src='./img/loading.gif'/>";
    loadingImg += "</div>";

    $('body').append(mask).append(loadingImg);

    $('#mask').css({
        'width' : maskWidth
        , 'height': maskHeight
        , 'opacity' : '0.3'
    });

    $('#mask').show();
    $('#loadingImg').show();
}
```

### 로딩바 숨김
```
function hideLoadingBar() {
    $('#mask, #loadingImg').hide();
    $('#mask, #loadingImg').remove();
}
```

### 이미지 표시
- 1번 이미지  
  ![tool-vscode-1](/img/posts/javascript/jquery/jquery-loading-1.gif)

- 2번 이미지  
  ![tool-vscode-1](/img/posts/javascript/jquery/jquery-loading-2.gif)