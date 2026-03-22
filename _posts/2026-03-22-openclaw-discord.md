---
layout: post
title: "Openclaw 와 discord 연동을 활용한 AI 비서 만들기 - 2"
subtitle: "2026-03-22-openclaw-discord.md"
date: 2026-03-22 20:40:00
author: "전봉근"
header-img: "img/posts/android-bg.jpg"
comments: true
tags: [ai]
---

#### Openclaw 에 Discord 를 연동하여 AI 비서를 만들자
https://bkjeon1614.tistory.com/842 이전 게시글을 참고하여 먼저 openclaw 를 설치합니다.     


#### Openclaw 를 AI Agent 를 사용할 때 주의사항
Openclaw 를 AI Agent 를 사용하기 위해선 Claude, ChatGPT 등과 같은 AI 모델이 필요합니다.     
단, 성능이 좋을 수록 비용이 많이 사용되므로(ex-claude의 opus 등..) 질문 용도에 맞는 모델을 선택하면 됩니다.      
      
현재 claude, gemini 는 검열에 걸려 계정이 제재당할 수 있습니다.     
그러므로 chatgpt의 codex를 사용하여 진행할 것 입니다. 이전 게시글을 참고하여 codex로 다시 세팅하시면 됩니다.    
    

#### Openclaw AI 모델 선택
하기 명령을 실행하고 이미지와 같이 차례로 넘어갑니다.      
```
$ openclaw config
```       
![openclaw-11](/img/posts/ai/openclaw/openclaw-11.PNG)       
![openclaw-12](/img/posts/ai/openclaw/openclaw-12.PNG)       
![openclaw-13](/img/posts/ai/openclaw/openclaw-13.PNG)       
![openclaw-14](/img/posts/ai/openclaw/openclaw-14.PNG)       
      
하다보면 https://auth.openai.com/oauth~ 로 링크가 출력되는데 복사하여 브라우저로 접속후 로그인을하고 "Authentication successful...." 라는 문구가 나오면 인증이 됩니다.     
![openclaw-15](/img/posts/ai/openclaw/openclaw-15.PNG)       
![openclaw-16](/img/posts/ai/openclaw/openclaw-16.PNG)       
![openclaw-17](/img/posts/ai/openclaw/openclaw-17.PNG)       
![openclaw-18](/img/posts/ai/openclaw/openclaw-18.PNG)        
         
```
// 해당 명령을 통하여 openclaw 를 실행하고 커맨드를 입력하면 AI가 대답을 하는것을 확인할 수 있습니다.
$ openclaw tui
```        
      
![openclaw-19](/img/posts/ai/openclaw/openclaw-19.PNG)        


#### Discord Bot 생성
https://discord.com/developers/applications 접속하여 좌측탭에 Application 클릭 후 우측 상단에 New Application 을 클릭하여 디스코드봇의 이름을 정의하고 생성합니다.   
      
생성 후 해당 Application 에 들어가서 좌측 메뉴에 Overview > Bot 에서 하기 이미지와 같은 항목을 허용합니다.    
![openclaw-20](/img/posts/ai/openclaw/openclaw-20.PNG)        
       
그리고 아래와 같이 토큰 초기화를 클릭 후 Discord Bot Token을 Copy 후 별도의 메모장에 저장합니다.      
![openclaw-21](/img/posts/ai/openclaw/openclaw-21.PNG)        
       
좌측 메뉴에 OAuth2를 클릭하면 openclaw discord bot이 가질 권한을 부여할 수 있습니다.        
![openclaw-22](/img/posts/ai/openclaw/openclaw-22.PNG)        
![openclaw-23](/img/posts/ai/openclaw/openclaw-23.PNG)        
       
그리고 상기 이미지 마지막에 Generated URL을 복사한 뒤 브라우저에서 접속하여 생성한 디스코드봇을 디스코드 채널로 초대합니다.      
![openclaw-24](/img/posts/ai/openclaw/openclaw-24.PNG)      
![openclaw-25](/img/posts/ai/openclaw/openclaw-25.PNG)      
       

#### 생성한 Discord Bot을 Openclaw와 연동
터미널에서 `openclaw config` 명령어 입력 후 하기 이미지 순서대로 진행합니다.      
![openclaw-26](/img/posts/ai/openclaw/openclaw-26.PNG)      
![openclaw-27](/img/posts/ai/openclaw/openclaw-27.PNG)      
![openclaw-28](/img/posts/ai/openclaw/openclaw-28.PNG)      
       
Enter Discord bot token 란에 아까 별도의 메모장에 복사하였던 Token을 입력하고 엔터를 누르고 다음 질문에 Yes를 선택합니다. (해당 토큰값 외부 노출 금지)     
![openclaw-29](/img/posts/ai/openclaw/openclaw-29.PNG)      
      
이제 Discord로 이동하여 프로필 우측에 환경설정 버튼을 클릭합니다.    
![openclaw-30](/img/posts/ai/openclaw/openclaw-30.PNG)      
       
고급메뉴를 클릭 후 개발자 모드를 활성화 시킵니다.     
![openclaw-31](/img/posts/ai/openclaw/openclaw-31.PNG)      
      
이제 디스코드 봇을 초대한 채널의ID와 하단의 채팅채널의 일반을 우클릭하여 채널ID도 복사하여 별도의 메모장에 복사합니다.     
![openclaw-32](/img/posts/ai/openclaw/openclaw-32.PNG)      
       
그 다음 Discord channels allowlist (comma-separated) 란에 아까 별도 메모장에 복사하였던 `서버ID/일반채널ID` 의 형태로 입력하고 엔터를 누른 후 Finished (Done) 을 선택합니다.       
![openclaw-33](/img/posts/ai/openclaw/openclaw-33.PNG)      
![openclaw-34](/img/posts/ai/openclaw/openclaw-34.PNG)      
      
마지막으로 DM 규칙은 기본 Pairing이므로 변경하지 않고 No를 선택합니다.     
![openclaw-35](/img/posts/ai/openclaw/openclaw-35.PNG)      
        
이제 터미널에서 하기 명령을 입력하여 게이트웨이를 재시작합니다.     
```
$ openclaw gateway restart
```             
        
모두 완료되었으므로 @{디스코드봇} 메세지를 보내면 AI가 답변을 하는것을 볼 수 있습니다.      
만약 개인 DM을 보내면 초기 접근 설정이 안되어 있다고 메세지 답변을 받는데 내용 가장 하단에 `openclaw pairing approve discord` 부분을 복사하여 터미널에 입력하면 페어링이 되어 봇과 연동되어 보안에 이점이 있습니다.       

> 여기까지 연동하는 과정을 마무리 하겠습니다.      

    
#### 참고
- https://goddaehee.tistory.com/
- https://blog.naver.com/ryurime88/