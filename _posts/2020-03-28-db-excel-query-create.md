---
layout: post
title : "Excel 파일을 DB Insert 쿼리로 생성"
subtitle : "2020-03-28-db-excel-query-create.md"
date: 2020-03-28 21:12:00
author: "전봉근"
header-img: "img/posts/android-bg.jpg"
comments: true
tags: [db, data, excel]
---

## Excel 파일을 DB Insert 쿼리로 생성
----------------------------------------------------------------

보통 비 IT 부서에서 다량의 데이터를 등록하기 위하여 Excel과 같은 형태로 데이터를 보내주는 경우가 있다.  
이럴 경우 기존 DB 모델링 규격과 상이하여 쉽게 데이터를 삽입할 수 없을 때 Excel 함수를 이용한 쿼리를 생성하여 쉽게 하는 방법을 알아보자.

1. 아래 이미지는 타 부서에서 데이터를 삽입해 달라고 전달받은 Excel 파일이다.  
   ![db-excel-query-create-1](/img/posts/db/common/db-excel-query-create-1.PNG)

2. 아래 이미지와 같이 사원 관련 테이블이 있다. Excel의 "=" 연산을 사용하여 우선 한줄 쿼리를 만들자.  
   - 쿼리를 입력할 때 문자는 '"문자"' 와 같이 입력하고 숫자는 "숫자" 처럼 생성하자.  
     ![db-excel-query-create-2](/img/posts/db/common/db-excel-query-create-2.PNG) 
     ![db-excel-query-create-3](/img/posts/db/common/db-excel-query-create-3.PNG) 
   ```
    ="INSERT INTO MEMBER VALUES("&A2&", '"&B2&"', '"&C2&"', '"&D2&"')"
   ```
   
3. 해당 셀을 마우스로 끌어내리면 아래와 이미지와 같이 쿼리가 생성되어 있다. (완료)
   ![db-excel-query-create-4](/img/posts/db/common/db-excel-query-create-4.PNG) 


> 위와 같은 유형으로 UPDATE 같은 쿼리문도 생성이 가능하며 특히 건별로 입력하는 상기 유형같은 경우 시간이 오래걸리므로 VALUES (1, 2, 3), (4, 5, 6) ... 등과 같이 다량 데이터 삽입 쿼리를 이용하는 경우도 있는데 DB 패킷 사이즈에 따라 안되는 경우도 있으며 DB서버에 부하를 줄 수 있는 경우도 있으므로 적당하게 잘 사용할 필요가 있다.