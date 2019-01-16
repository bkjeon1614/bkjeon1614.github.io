---
layout: post
title : "VS Code 에디터 플러그인을 이용하여 Markdown문서를 HTML로 변환"
subtitle : "2019-01-10-VSCode-Markdown-To-HTML"
date: 2019-01-10 10:51:00
author: "전봉근"
header-img: "img/posts/android-bg.jpg"
comments: true
tags: [tool]
---

VS Code 에디터 플러그인을 이용하여 Markdown문서를 HTML로 변환
=========
티스토리 블로그나 Markdown 형식의 문서를 동시에 관리해야하는 문제가 생겨 어떻게 하면 쉽게 양쪽다 글을 올릴 수 있을까? 라는 생각에 방법을 찾아 해당 포스팅을 작성하게 되었다.

### 준비
- OS: MacOS
- [VS CODE Editor](https://code.visualstudio.com)

### 과정
- VS Code에서 Markdown Preview 확인
  - VS Code에서 Markdown 파일을 실행하고 우측 상단에 창 2개에 돋보기 모양의 버튼을 클릭하면 된다.
    ![tool-vscode-1](/img/posts/tool/tool-vscode-1.png)
    
    그럼 아래와 같이 Preview를 확인할 수 있다.
    ![tool-vscode-2](/img/posts/tool/tool-vscode-2.png)  

- Markdown 문서를 HTML로 변환
  - Copy Markdown As HTML Plugin 설치
    ![tool-vscode-3](/img/posts/tool/tool-vscode-3.png)  
    
    설치가 완료되면 install 버튼이 변경된다.
    ![tool-vscode-4](/img/posts/tool/tool-vscode-4.png)  

  - F1키를 눌러 검색창을 열고 "Mark" or "Copy"의 키워드를 입력하고 Markdown: Copy as HTML을 선택하면 현재 마크다운 문서가 HTML코드로 변환되어 클립보드에 저장된다.
    ![tool-vscode-5](/img/posts/tool/tool-vscode-5.png)  

  - 티스토리에 글 작성 부분에 HTML모드로 전환하여 Ctrl + V를 누르면 아래와 이미지와 같이 표시된다.
    ![tool-vscode-6](/img/posts/tool/tool-vscode-6.png)  

  - 글 작성을 눌러서 완료하자. (단, 이미지 같은경우는 직접 티스토리 사진등록을 통하여 등록해야하는 번거로움이 있다.)
    ![tool-vscode-7](/img/posts/tool/tool-vscode-7.png)