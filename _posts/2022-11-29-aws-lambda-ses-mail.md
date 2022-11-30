---
layout: post
title: "AWS Lambda 와 SES(=Simple Email Service) 를 이용한 간단한 메일 발송 (NodeJS)"
subtitle: "2022-11-29-aws-lambda-ses-mail.md"
date: 2022-11-29 17:50:00
author: "전봉근"
header-img: "img/posts/android-bg.jpg"
comments: true
tags: [js, javascript, nodejs, mail, aws, lambda, ses]
---

# AWS Lambda 와 SES(=Simple Email Service) 를 이용한 간단한 메일 발송 (NodeJS)
메일 발송 기능을 개발을 하기위해 AWS 의 Lambda 와 SES(=Simple Email Service) 를 선택하게 되었다.    
서비스의 역할 및 예제코드를 참고하여 간단하게 메일 발송 기능을 구축해보자. (NodeJS 를 사용)


## AWS Lambda
AWS Lambda 는 이벤트 기반 서버리스 컴퓨팅 플랫폼이며, 이벤트에 반응하여 Lambda 에 작성된 코드를 실행하는 서비스이다. 즉, 클라우드 제공업체에서 인프라에 대한 관리를 대신 처리해주기 때문에 개발자는 비즈니스 로직에만 집중할 수 있다. (Node, Java, .NET, Go, Python, Ruby 등의 다양한 언어를 지원한다.)      
또한 상기 내용처럼 Lambda 에다 원하는 기능들을 코드로 작성 및 사용할 수 있고 다른 AWS 서비스들과 연동이 편하다.    
     
### Lambda 장점
1. 필요할때만 함수를 호출하기 때문에 일반 서버처럼 켜놓는 시점에서 과금을 하지 않아서 비용을 절약할 수 있다. (그러나 Lambda 요청 수 및 실행시간에 따라 요금이 부과된다)
2. 서버관리 및 보안 등 인프라는 AWS 에서 관리하므로 개발자가 특별히 서버 관리를 하지 않아도 되므로 비즈니스 로직에만 집중할 수 있다.
3. AWS 의 다른 서비스와의 연동이 쉽고 Lambda 내부에 코드를 작성/수정만 해주면 되고 또한 배포방법도 단순하며 빠르다.

### Lambda 단점
1. 함수 하나가 호출될 때 AWS 에서는 최대 10GB 메모리와 처리시간은 최대 15분이라 성능을 잘 고려하여 선택해야 한다.
2. Lambda 는 함수가 호출될 때 컨테이너를 띄우는 방식이므로 Stateless (상태비저장) 이다. 그러므로 db connection 과 같은것을 유지하는 기능은 수행하지 못한다.
3. Lambda 는 리소스를 효율적으로 사용하기 위해 오랫동안 사용하지 않으면 컴퓨팅파워를 꺼두고 있으므로 다시 사용하려면 Lambda 컨테이너를 띄우기 위해 프로비저닝되는 시간이 소요된다. 즉, 어느정도의 레이턴시가 발생하게 된다. (ColdStart 문제발생)     
   ```
   [ColdStart 해결방법]
   - Lambda 를 지속적으로 호출한다. (호출 및 처리시간에 따라 비용이 산정되므로 잘 생각해서 결정하는게 좋다.)
   - Lambda 의 메모리 스펙을 높인다. (메모리를 더 할당하면 처리속도가 빨라지므로 지연시간을 줄일 수 있다.)
   - 프로비저닝된 동시성 활성화 (2019년 ColdStart 문제를 해결하기위해 나온 옵션이라고 한다.) 를 한다. (추가적인 비용 과금)
   ```
4. 동시성 제한이 있다. (리전별로 동시에 실행할 수 있는 Lambda 함수의 개수는 최대 1000개로 제한하고 있다.)      
   ```
   - 예약된 동시성을 통해 동시성 개수 높이기
   - Limit Increase 요청 (아무리 최적화를 해도 어쩔 수 없이 최대 동시성 수를 넘어갔을 경우, AWS 에 동시실행 수를 올려달라고 서비스 할당량 증가 요청을 할 수 있다.)
   ```

## AWS SES
SES 는 Simple Email Servcie 의 약자로 이메일 전송 서비스이며 마케팅 같은 대량의 이메일을 발송하기에 적절하고 발송한 이메일의 수와 데이터 전송에 대해 요금이 부과된다. (EC2 로부터의 아웃바운드 이메일일경우 62000 건 까지는 요금이 0 원이다. [요금정책](https://aws.amazon.com/ko/ses/pricing/)) 이외에도 전송 시도, 거부 메세지, 반송, 수신 거부등의 통계를 자동으로 수집하여 모니터링도 할 수 있으며 메일 전송 테스트도 해볼 수 있다. 또한 SMTP 인터페이스 또는 SES Query API 를 직접 호출할 수 있다. (직접 메일서버를 구축할 필요없이 저렴하게 사용이 가능하다.)

## 실습 (NodeJS)
1. AWS SES 메일 인증
   - Verified identities 메뉴 선택 후 우측의 Create identity 를 클릭     
     ![aws-lambda-ses-mail-1](/img/posts/aws/mail/aws-lambda-ses-mail-1.PNG)     
   - Email address 를 선택한 후 내용을 입력하고 하단의 Create identity 클릭    
     ![aws-lambda-ses-mail-2](/img/posts/aws/mail/aws-lambda-ses-mail-2.PNG)     
   - 다 완료하면 생성한 이메일로 인증메일이 발송하는데 인증링크를 클릭하면 하기 이미지의 아이콘이 초록색 체크로 변한다.      
     ![aws-lambda-ses-mail-3](/img/posts/aws/mail/aws-lambda-ses-mail-3.PNG)      
2. AWS SES SMTP 사용
   - SMTP settings 메뉴 선택 후 우측의 Create SMTP credentials 클릭     
     ![aws-lambda-ses-mail-4](/img/posts/aws/mail/aws-lambda-ses-mail-4.PNG)      
   - IAM User Name 은 잘 기억해놓자. (나중에 필요할 수 도 있음)      
     ![aws-lambda-ses-mail-5](/img/posts/aws/mail/aws-lambda-ses-mail-5.PNG)      
   - 위의 과정까지 완료하면 우측하단에 자격 증명 다운로드를 클릭하면 인증정보를 알 수 있는 파일이 다운된다.    
     ![aws-lambda-ses-mail-6](/img/posts/aws/mail/aws-lambda-ses-mail-6.PNG)      
3. 로컬환경에서의 테스트
   - 라이브러리 설치
     ```
     $ npm init
     $ npm install nodemailer
     $ npm install nodemailer-smtp-transport
     ```    
   - package.json 변경     
     ``` 
     ...
     "scripts": {
       "test": "echo \"Error: no test specified\" && exit 1",
       "start": "node index.js"   // 추가
     },     
     ...
     ```     
   - 메일전송 코드 추가 (index.js)    
     [index.js]    
     ```
      "use strict";
      const nodemailer = require("nodemailer");

      const smtpEndpoint = "email-smtp.ap-northeast-2.amazonaws.com"; // aws email region (가운데 ap-northeast-2 는 서울 리전임)
      const port = 587;
      const senderAddress = "bkjeon@gmail.com"; // 보낸사람
      const toAddresses = "bkjeon2@gmail.com"; // 받는사람
      const ccAddresses = ""; // 참조
      const bccAddresses = ""; // 숨은참조
      const smtpUsername = "AKIA5TZNI26SL6WSZFQM"; // aws ses smtp 인증완료시 발급되는 username 값
      const smtpPassword = "BM7FQ51XBIde4BelRcoEp2Legp086MKKhqlvfnmOriWM"; // aws ses smtp 인증완료시 발급되는 password 값

      // 구성세트값 (Optional)
      // 내 정책을 SES 에서 완전히 허용하도록 설정 (https://docs.aws.amazon.com/ses/latest/dg/control-user-access.html)
      const configurationSet = [
        {
          Effect: "Allow",
          Action: ["ses:*"],
          Resource: "*",
        },
      ];

      const subject = "Amazon SES test (Nodemailer) 제목"; // 메일제목
      const body_text = `Amazon SES Test (Nodemailer) 제목2
      ---------------------------------
      This email was sent through the Amazon SES SMTP interface using Nodemailer.
      `;
      // The body of the email for recipients whose email clients support HTML content.
      const body_html = `<html>
      <head></head>
      <body>
        <h1>Amazon SES Test (Nodemailer)</h1>
        <p>This email was sent with <a href='https://aws.amazon.com/ses/'>Amazon SES</a>
              using <a href='https://nodemailer.com'>Nodemailer</a> for Node.js.</p>
      </body>
      </html>`; // 메일내용
      const tag0 = "key0=value0";
      const tag1 = "key1=value1";

      async function main() {
        // Create the SMTP transport.
        let transporter = nodemailer.createTransport({
          host: smtpEndpoint,
          port: port,
          secure: false, // true for 465, false for other ports
          auth: {
            user: smtpUsername,
            pass: smtpPassword,
          },
        });

        // Specify the fields in the email.
        let mailOptions = {
          from: senderAddress,
          to: toAddresses,
          subject: subject,
          cc: ccAddresses,
          bcc: bccAddresses,
          text: body_text,
          html: body_html,
          // Custom headers for configuration set and message tags.
          headers: {
            "X-SES-CONFIGURATION-SET": configurationSet,
            "X-SES-MESSAGE-TAGS": tag0,
            "X-SES-MESSAGE-TAGS": tag1,
          },
        };

        // Send the email.
        let info = await transporter.sendMail(mailOptions);

        console.log("Message sent! Message ID: ", info.messageId);
      }

      main().catch(console.error);
     ```
5. Lambda 에 4번의 테스트 코드 적용
   - 함수 생성 클릭    
     ![aws-lambda-ses-mail-7](/img/posts/aws/mail/aws-lambda-ses-mail-7.png)      
   - 함수이름 및 Node 버전 입력 후 함수생성     
     ![aws-lambda-ses-mail-8](/img/posts/aws/mail/aws-lambda-ses-mail-8.png)      
   - API 호출 연동을 위한 트리거 추가 클릭
     ![aws-lambda-ses-mail-9](/img/posts/aws/mail/aws-lambda-ses-mail-9.png)      
   - API 게이트웨이 선택
     ![aws-lambda-ses-mail-10](/img/posts/aws/mail/aws-lambda-ses-mail-10.png)      
   - 하기 이미지와 같이 선택 후 생성한다. (API 이름 정도만 기재한다. 이외에 다른 옵션들은 따로 찾아보자.)     
     ![aws-lambda-ses-mail-11](/img/posts/aws/mail/aws-lambda-ses-mail-11.png)    
   - Test 실행하여 결과 확인   
     ![aws-lambda-ses-mail-12](/img/posts/aws/mail/aws-lambda-ses-mail-12.png)       
   - 상기 프로젝트때 만든 샘플코드를 압축하여 하기 이미지처럼 업로드한다.    
     ![aws-lambda-ses-mail-13](/img/posts/aws/mail/aws-lambda-ses-mail-13.png)    


## 참고
- https://inpa.tistory.com/
- https://likelionsungguk.github.io/