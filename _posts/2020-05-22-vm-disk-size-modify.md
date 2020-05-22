---
layout: post
title : "VirtualBox 에서 디스크 크기 변경하기(동적할당 or 고정할당)"
subtitle : "2020-05-22-vm-disk-size-modify.md"
date: 2020-05-22 22:00:00
author: "전봉근"
header-img: "img/posts/android-bg.jpg"
comments: true
tags: [virtualbox, vm]
---

## VirtualBox 에서 디스크 크기 변경하기
----------------------------------------------------------------

VirtualBox 에서 초기 디스크 용량을 적게 설정할 때 증가를 시키기위해 VirtualBox에서 기본적으로 제공하는 매니징툴을 사용하여 사이즈를 늘려보자.

1. VirtualBox가 설치된 폴더에서 VBoxManage.exe 를 실행하여 size를 변경해보자.
   ```
      // size단위는 MB로 입력하자.
      c:\{vm경로}>VBoxManage.exe modifyhd "{사용자경로(*.vdi가 생성되는경로)}" --resize 51400
   ```
   ![tool-vm-1](/img/posts/tool/vm/tool-vm-1.PNG)

   > 위의 명령으로 만약 동적할당된 디스크면 변경이 가능하지만 고정할당된 디스크면 위의 이미지와 같은 에러가 표시된다.

2. "파일" -> "가상 미디어 관리자" -> "복사(C)" 를 하게되면 동적 확장 저장소가 생성된다. (속성에서도 사이즈 변경이 가능하다.)
   ![tool-vm-2](/img/posts/tool/vm/tool-vm-2.PNG)

3. 동적 할당 저장소가 생성되고 사이즈 변경을 실행하면 아래와 같이 완료된다.
   ![tool-vm-3](/img/posts/tool/vm/tool-vm-3.PNG)

4. 이제 기존에 연결되어있떤 저장소를 제거해보자.
   ![tool-vm-4](/img/posts/tool/vm/tool-vm-4.PNG)

5. 새로운 저장소를 연결해보자.
   ![tool-vm-5](/img/posts/tool/vm/tool-vm-5.PNG)

6. 이전에 복사로 생성된 동적 할당 저장소를 선택하자.
   ![tool-vm-6](/img/posts/tool/vm/tool-vm-6.PNG)   

7. 연결 완료 후 실행하면 용량이 변경됨을 확인할 수 있다.
   ![tool-vm-7](/img/posts/tool/vm/tool-vm-7.PNG)   

8. 이제 가상 디스크는 사이즈가 커졌지만 하드 디스크 내의 파티션은 그대로 이기 때문에 직접 변경해 주어야한다. ubuntu 설치시 사용하였던 iso 파일로 부팅해서 가상 하드 디스크 파티션을 변경해보자. iso 파일로 부팅 후 설치 창이 표시되면 설치하는 것이 아니므로 "Try Ubuntu"를 클릭하자.
   ![tool-vm-8](/img/posts/tool/vm/tool-vm-8.PNG)

9. 그러면 기존 ubuntu를 부팅한 것 처럼 화면이 표시된다. 이제 파티션을 설정하는 프로그램을 실행해보자.
   ```
      sudo gparted
   ```
   ![tool-vm-9](/img/posts/tool/vm/tool-vm-9.PNG)

10. /dev/sda1(주 파티션)을 제외!! 하고 나머지 파티션들을 swap은 오른쪽 버튼을 눌러 swap-off 로 해주고 파티션을 삭제 한다. /dev/sda1 에서 오른쪽 버튼을 눌러 resize 한다. (2GB 정도는 swap 할 파티션을 지정 -> 필자는 그냥 6GB정도로 세팅)
   ![tool-vm-10](/img/posts/tool/vm/tool-vm-10.PNG)

11. 재부팅 후 아래 명령을 이용해 용량이 변경된걸 확인할 수 있다.
   ![tool-vm-11](/img/posts/tool/vm/tool-vm-11.PNG)