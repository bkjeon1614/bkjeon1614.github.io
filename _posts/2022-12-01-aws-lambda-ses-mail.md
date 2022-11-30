---
layout: post
title: "AWS Lambda 와 SES(=Simple Email Service) 를 이용한 간단한 메일 발송 (NodeJS)"
subtitle: "2022-12-01-aws-lambda-ses-mail.md"
date: 2022-12-01 18:50:00
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
     ![aws-lambda-ses-mail-7](/img/posts/aws/mail/aws-lambda-ses-mail-7.PNG)      
   - 함수이름 및 Node 버전 입력 후 함수생성     
     ![aws-lambda-ses-mail-8](/img/posts/aws/mail/aws-lambda-ses-mail-8.PNG)      
   - API 호출 연동을 위한 트리거 추가 클릭
     ![aws-lambda-ses-mail-9](/img/posts/aws/mail/aws-lambda-ses-mail-9.PNG)      
   - API 게이트웨이 선택
     ![aws-lambda-ses-mail-10](/img/posts/aws/mail/aws-lambda-ses-mail-10.PNG)      
   - 하기 이미지와 같이 선택 후 생성한다. (API 이름 정도만 기재한다. 이외에 다른 옵션들은 따로 찾아보자.)     
     ![aws-lambda-ses-mail-11](/img/posts/aws/mail/aws-lambda-ses-mail-11.PNG)    
   - Test 실행하여 결과 확인   
     ![aws-lambda-ses-mail-12](/img/posts/aws/mail/aws-lambda-ses-mail-12.PNG)       
   - 상기 프로젝트때 만든 샘플코드를 압축하여 하기 이미지처럼 업로드한다.    
     ![aws-lambda-ses-mail-13](/img/posts/aws/mail/aws-lambda-ses-mail-13.PNG)    


## 참고
- https://inpa.tistory.com/
- https://likelionsungguk.github.io/

















# Mocking 라이브러리를 활용하자
프론트엔드 개발시에 가장 이상적인 프로세스는 기획, 분석, 백엔드 개발, 프론트 개발들의 작업들이 서로 겹치지 않고 각각 진행되는 것이 좋다.

하지만 진행하는 과정에서 나오는 이슈 또는 요구사항의 변경등으로 인하여 번거롭게 작업을 진행해야되는 경우가 많다. 또한 프론트 엔드 영역과 백엔드 영역에 대한 일정이 서로 겹치게 동시 진행하는 경우도 많으므로 백엔드의 API 의 의존적이게 작업해야하는 프론트엔드의 경우는 백엔드 API 기능들이 개발서버에 배포되기 전까지 프론트엔드에서는 진행하기에 번거롭다. (물론 백엔드쪽에서 굳이 개발서버 배포를 안해도 로컬에서도 바로 확인 할 수 있는 다른 방법들이 있지만 인원이 많을 경우 변동성이 적은 개발서버 기준으로 작업하는것이 안정적이다.)

상기와 같은 이슈들이 존재하여 백엔드쪽의 API 가 개발되기전에 프론트엔드에서 먼저 개발을 할 수 있는 환경을 만들기 위해 Mocking 라이브러리를 활용하는 것이다.     
     

## Mocking 라이브러리 선택
먼저 Mocking 라이브러리는 sinon, nock, msw 등 여러가지들을 사용하고 있다고 한다.   
![moking-msw-1](/img/posts/javascript/msw/moking-msw-1.png)     
상기 이미지의 라이브러리들을 대표적으로 사용하고 있으며 이 중 가장 러닝커브가 낮고, 디버깅에 용이하며 GraphQL 또한 적용이 가능한 MSW(=Mock Service Worker) 를 선택하게 되었다.    
MSW 는 별도 환경없이 네트워크 레벨에서 요청을 가로채도록 설계되어 있어 API 에 종속성 없이 높은 수준의 작업이 가능하다. (실제 사용하는 방식처럼 테스트가 가능함)


## VS Code 에 Prettier 적용
일관적인 코드 스타일을 유지하기 위하여 Prettier 를 적용하자 (대부분 알고 있는 내용이라 간단히만 작성하였다.)   
1. VS Code 에 Prettier 플러그인 설치       
2. Code > Preferences > Settings > Text Editor > Default Formatter > Prettier - Code formatter 로 변경      
   ![tool-vscode-prettier-1](/img/posts/tool/tool-vscode-prettier-1.png)    
   ![tool-vscode-prettier-2](/img/posts/tool/tool-vscode-prettier-2.png)    
3. 검색창에 format on save 입력 후 체크    
   ![tool-vscode-prettier-3](/img/posts/tool/tool-vscode-prettier-3.png)    
* Prettier 예외 설정 방법은 - ROOT 경로에 .prettierignore 파일 생성 (ex: 내용에 *.md 추가)


## 1. MSW.js 란?
[MSW (Mock Service Worker)](https://mswjs.io/) 는 Service Worker API 를 사용하여 실제 요청을 가로채는 API Mocking 라이브러리 이다. (지원되는 API 는 REST, GraphQL 가능)  
MSW 는 이벤트를 통해 애플리케이션의 나가는 요청을 수신 대기하는 서비스 워커를 등록하고 해당 요청을 클라이언트 측 라이브러리로 보내고 응답할 작업자에게 서비스 워커가 모의 응답을 전송한다.    
![moking-msw-2](/img/posts/javascript/msw/moking-msw-2.png)    

> 샘플 코드는 필요한 내용에 대해서만 간단하게 작성하므로 필요한 부분이 있으면 알아서 개선시키면 된다.
       
[샘플코드](https://github.com/bkjeon1614/vuejs-example-code/tree/main/vue3-typescript-msw)
           
## 2. MSW 설치
### 2-1. Vue Project 설치
이미 설치되어 있는 사람은 해당 내용을 건너 뛰어도 된다. 3번 부터 시작하자.      
1. Vue 프로젝트 설치 명령 실행
   ```
   $ vue create hello-world
   ```
2. Please pick a preset
   ```
   [ ] Default ([Vue 2] babel, eslint)
   [ ] Default (Vue 3 Preview) ([Vue 3] babel, eslint)
   [x] Manually select features
   ```
3. Check the features needed for your project
   ```
   (*) Choose Vue version
   (*) Babel
   (*) TypeScript
   ( ) Progressive Web App (PWA) Support
   (*) Router
   (*) Vuex
   (*) CSS Pre-processors
   (*) Linter / Formatter
   ( ) Unit Testing
   ( ) E2E Testing
   ```
4. Choose a version of Vue.js that you want to start the project with
   ```
   [ ] 2.x
   [x] 3.x (Preview)
   ```
5. Use class-style component syntax? (y/N)
   ```
   N
   ```
6. Use Babel alongside TypeScript (required for modern mode, auto-detected polyfills, transpiling JSX)?
   ```
   Y
   ```
7. Use history mode for router? (Requires proper server setup for index fallback in production)
   ```
   Y
   ```
8. Pick a CSS pre-processor (PostCSS, Autoprefixer and CSS Modules are supported by default)
   ```
   [x] Sass/SCSS (with dart-sass)
   [ ] Less
   [ ] Stylus
   ```
9. Pick a linter / formatter config: (Use arrow keys)
   ```
   [ ] ESLint with error prevention only
   [ ] ESLint + Airbnb config
   [ ] ESLint + Standard config
   [*] ESLint + Prettier
   [ ] TSLint (deprecated)
   ```
10. Pick additional lint features
    ```
    (*) Lint on save
    (*) Lint and fix on commit
    ```
11. Where do you prefer placing config for Babel, ESLint, etc.?
    ```
    [] In dedicated config files
    [x] In package.json
    ```
    > 이건 각 설정파일들을 따로 관리할 수 있는 `In dedicated config files` 를 보편적으로 사용하지만 개인적으로 package.json 이 좀 더 익숙하므로 선택함
12. Save this as a preset for future projects? (y/N)
    ```
    N
    ```

### 2-2. Vue Project 설치 및 실행
1. NPM 설치 및 실행
   ```
   $ npm install
   $ npm run serve
   ```
2. 브라우저 접속 http://localhost:8080 (하단에 표시됨)

### 2-3. Axios 설치
1. Axios 설치
   ```
   $ npm install axios
   ```

### 2-4. Axios 통신 샘플코드 작성 및 실행
#### 2-4-1. 프로젝트 구조    
![moking-msw-3](/img/posts/javascript/msw/moking-msw-3.png)     
   
#### 2-4-2. 샘플코드 생성
[package.json]
```
...
  "eslintConfig": {
    ...
    "rules": {
      "@typescript-eslint/no-namespace": "off",
      "@typescript-eslint/no-var-requires": 0,
      "@typescript-eslint/no-explicit-any": [
        "off"
      ]
    }
  },
...    
```    
    
[src/api/index.ts]   
```
import axios, {
  AxiosInstance,
  AxiosError,
  AxiosResponse,
  AxiosRequestConfig,
} from "axios";
import { ResultEnum } from "@/enums/httpEnum";
import { checkStatus } from "./helper/checkStatus";

const axiosConfig = {
  timeout: 10000,
};

class RequestHttp {
  service: AxiosInstance;
  public constructor(axiosConfig: AxiosRequestConfig) {
    this.service = axios.create(axiosConfig);

    this.service.interceptors.request.use(
      (axiosConfig: AxiosRequestConfig) => {
        return {
          ...axiosConfig,
          headers: { ...axiosConfig.headers, "x-access-token": "bkjeon" },
        };
      },
      (error: AxiosError) => {
        return Promise.reject(error);
      }
    );
    this.service.interceptors.response.use(
      (response: AxiosResponse) => {
        const { data } = response;

        if (data.code && data.code !== ResultEnum.SUCCESS) {
          console.log("Application Error !!: " + data.msg);
          return Promise.reject(data);
        }

        return data;
      },
      async (error: AxiosError) => {
        const { response } = error;
        if (error.message.indexOf("timeout") !== -1) {
          console.log("Request timed out! Please try again later");
        }

        if (response) checkStatus(response.status);
        return Promise.reject(error);
      }
    );
  }
  get<T>(url: string, params?: object, _object = {}): Promise<T> {
    return this.service.get(url, { params, ..._object });
  }
  post<T>(url: string, params?: object, _object = {}): Promise<T> {
    return this.service.post(url, params, _object);
  }
  put<T>(url: string, params?: object, _object = {}): Promise<T> {
    return this.service.put(url, params, _object);
  }
  delete<T>(url: string, params?: any, _object = {}): Promise<T> {
    return this.service.delete(url, { params, ..._object });
  }
}

export default new RequestHttp(axiosConfig);
```     
      
[src/api/helper/checkStatus.ts]    
```
export const checkStatus = (status: number): void => {
  switch (status) {
    case 400:
      console.log("Request failed! Please try again later");
      break;
    case 500:
      console.log("Service exception!");
      break;
    default:
      console.log("Request failed!");
  }
};
```     
     
[src/api/interface/index.ts]     
```
export interface Result {
  code: string;
  msg: string;
}

export interface ResultData<T = any> extends Result {
  data: T;
}

export interface ResPage<T = any> extends Result {
  pageNum: number;
  pageSize: number;
  totalCnt: number;
  data: T[];
}

export interface ReqPage {
  pageNum?: number;
  pageSize?: number;
}

export namespace Sample {
  export interface ReqGetSampleParams extends ReqPage {
    title?: string;
    description?: string;
  }
  export interface ResSampleList {
    title: string;
    description: string;
    ingredients: string[];
    image: string;
    id: number;
  }
}
```    
    
[src/api/modules/samples.ts]    
```   
import { Sample } from "@/api/interface/index";

import http from "@/api";

export const getSampleList = (params: Sample.ReqGetSampleParams) => {
  return http.get<Sample.ResSampleList[]>(
    `https://api.sampleapis.com/coffee/hot`,
    params
  );
};
```       
    
[src/enums/httpEnums.ts]      
```
export enum ResultEnum {
  SUCCESS = 200,
  ERROR = 500,
  TIMEOUT = 10000,
  TYPE = "SUCCESS",
}

export enum RequestEnum {
  GET = "GET",
  POST = "POST",
  PATCH = "PATCH",
  PUT = "PUT",
  DELETE = "DELETE",
}

export enum ContentTypeEnum {
  JSON = "application/json;charset=UTF-8",
  TEXT = "text/plain;charset=UTF-8",
  FORM_URLENCODED = "application/x-www-form-urlencoded;charset=UTF-8",
  FORM_DATA = "multipart/form-data;charset=UTF-8",
}
```     
    
[src/views/SampleView.vue]    
```
<template>
  <div v-if="sampleList">
    <ul v-for="(item, index) in sampleList" :key="index">
      <li>{{ item.id }}</li>
      <li>{{ item.title }}</li>
      <li>{{ item.description }}</li>
      <li>{{ item.image }}</li>
      <li>{{ item.ingredients }}</li>
    </ul>
  </div>
</template>

<script lang="ts">
import { onBeforeMount, reactive, ref } from "vue";
import { Sample } from "@/api/interface";
import { getSampleList } from "@/api/modules/sample";

export default {
  setup() {
    const sampleList = ref<Sample.ResSampleList[]>([]);
    const requestSampleParam = reactive<Sample.ReqGetSampleParams>({});

    onBeforeMount(() => {
      getSamples();
    });

    const getSamples = async () => {
      await getSampleList(requestSampleParam).then((response) => {
        sampleList.value = response;
      });
    };

    return {
      sampleList,
    };
  },
};
</script>

<style></style>
```    
    
[src/router/index.ts]    
```
import { createRouter, createWebHistory, RouteRecordRaw } from "vue-router";
import HomeView from "../views/HomeView.vue";

const routes: Array<RouteRecordRaw> = [
  ...
  {
    path: "/sample",
    name: "sample",
    component: () => import("../views/sample/SampleView.vue"),
  },
];

...
```         
     
#### 2-4-3. 샘플코드 실행
`npm run serve` 실행하면 하기 이미지와 같이 데이터를 출력하는 것을 볼 수 있다.    
![moking-msw-4](/img/posts/javascript/msw/moking-msw-4.png)    

#### 2-4-4. MSW 적용을 위한 샘플코드 작성
[src/router/index.ts]    
```
import { createRouter, createWebHistory, RouteRecordRaw } from "vue-router";
import HomeView from "../views/HomeView.vue";

const routes: Array<RouteRecordRaw> = [
  ...
  {
    path: "/user",
    name: "user",
    component: () => import("../views/user/UserView.vue"),
  },
];

...
```   

[src/api/modules/user.ts]
``` 
import { ResPage, ResultData, User } from "@/api/interface/index";

import http from "@/api";

export const getUserList = (params: User.ReqGetUserParams) => {
  return http.get<ResPage<User.ResUser>>(`/api/users`, params);
};

export const getUserData = (params: { id: string }) => {
  return http.get<ResultData<User.ResUser>>(`/api/users/` + params.id);
};
```      
    
[src/views/user/UserView.scss]   
```
/* 필요시 작성 */
```

[src/views/user/UserView.vue]   
```
<template>
  <div v-if="userData">
    <ul>
      <li>{{ userData.id }}</li>
      <li>{{ userData.username }}</li>
      <li>{{ userData.age }}</li>
      <li>{{ userData.email }}</li>
    </ul>
  </div>
  <br /><br /><br /><br />
  <div v-if="userList">
    <div>{{ userListCnt }}</div>
    <ul v-for="(item, index) in userList" :key="index">
      <li>{{ item.id }}</li>
      <li>{{ item.username }}</li>
      <li>{{ item.age }}</li>
      <li>{{ item.email }}</li>
    </ul>
  </div>
</template>

<script lang="ts">
import { onBeforeMount, reactive, ref } from "vue";
import { User } from "@/api/interface";
import { getUserList, getUserData } from "@/api/modules/user";

export default {
  setup() {
    const userListCnt = ref<number>(0);
    const userList = ref<User.ResUser[]>([]);
    const userData = ref<User.ResUser>();
    const requestUserParam = reactive<User.ReqGetUserParams>({
      pageNum: 1,
      pageSize: 10,
      username: "bkjeon",
      email: "bkjeon@gmail.com",
    });
    const requestUserId = "M221114EEE";

    onBeforeMount(() => {
      getUsers();
      getUser();
    });

    const getUsers = async () => {
      await getUserList(requestUserParam).then((response) => {
        userListCnt.value = response.totalCnt;
        userList.value = response.data;
      });
    };

    const getUser = async () => {
      await getUserData({ id: requestUserId }).then((response) => {
        userData.value = response.data;
      });
    };

    return {
      userListCnt,
      userList,
      userData,
    };
  },
};
</script>

<style scoped lang="scss">
@import "./UserView.scss";
</style>
```    
  
### 2-5. MSW 적용
#### 2-5-1. MSW 패키지 설치
```
$ npm install msw --save-dev    // or yarn add msw --dev
```

#### 2-5-2. Service Worker 를 위한 JS 생성
```
$ npx msw init public --save    // 경로는 public 폴더 안으로 지정
```    
MSW CLI 를 통하여 init 명령어와 함께 사용하고자 하는 프로젝트의 public directory 를 지정해서 실행하면 mockServiceWorker.js 가 생성된다.

#### 2-5-3. Mocking 할 API handler / mock data 생성
[src/mock/modules/user_handler.ts]  
```
import { rest } from "msw";
import { users, user } from "@/mock/modules/data/users";

const handlers = [
  rest.get("/api/users", (req, res, ctx) => {
    return res(ctx.status(200), ctx.delay(1000), ctx.json(users));
  }),
  rest.get("/api/users/M221114EEE", (req, res, ctx) => {
    return res(ctx.status(200), ctx.delay(1000), ctx.json(user));
  }),
];

export default handlers;
```

[src/mock/data/users.ts]  
```
export const user = {
  data: {
    id: "M221114EEE",
    username: "bkjeon",
    age: 33,
    email: "gcijdfdo@gmail.com",
  },
};

export const users = {
  pageNum: 1,
  pageSize: 10,
  totalCnt: 3,
  data: [
    {
      id: "M221114EEE",
      username: "bkjeon",
      age: 33,
      email: "gcijdfdo@gmail.com",
    },
    {
      id: "M220014EEE",
      username: "bkjeon2",
      age: 31,
      email: "gcijdfdo2@gmail.com",
    },
    {
      id: "M221114EAA",
      username: "bkjeon3",
      age: 23,
      email: "gcijdfdo3@gmail.com",
    },
  ],
};
```

#### 2-5-4. 핸들러를 MSW 에서 사용할 수 있도록 import
[src/mock/browser.ts]
```
import { setupWorker } from "msw";
import user_handler from "@/mock/modules/user_handler";

export const worker = setupWorker(...user_handler);
```

#### 2-5-5. MSW 실행을 위한 코드 추가
[src/main.ts]
```
...

// MSW
if (process.env.NODE_ENV === "development") {
  const { worker } = require("@/mock/browser");
  worker.start();
}

...
```


## 3. 실행결과 확인
![moking-msw-5](/img/posts/javascript/msw/moking-msw-5.png)    
> 상기 이미지와 같이 `[MSW] Mocking enabled.` 메세지를 확인할 수 있다. (중지는 src/main.ts 의 worker.stop() 으로 하면 된다.)


## 참고
- https://mswjs.io/
- https://tech.kakao.com/2021/09/29/mocking-fe/