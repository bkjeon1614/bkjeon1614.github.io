---
layout: post
title : "Spring Boot에서 에러 페이지 처리하기"
subtitle : "2019-05-22-springBoot-exception-page.md"
date: 2019-05-22 17:35:00
author: "전봉근"
header-img: "img/posts/android-bg.jpg"
comments: true
tags: [spring]
---

Spring Boot에서 에러 페이지 처리하기
=========
에러가 발생할 때 웹 페이지에 에러에 대한 내용을 바로 출력하는 경우가 있다. 이와 같은 경우를 방지하기 위하여 에러페이지를 커스터마이징을 할 수 있는 컨트롤러를 만들어보자.

## 컨트롤러 생성
ErrorController를 Implements하여 커스텀 에러 컨트롤러를 생성한다.

[CustomErrorController.java]
```
    import java.util.Date;
    import javax.servlet.RequestDispatcher;
    import javax.servlet.http.HttpServletRequest;
    import lombok.extern.slf4j.Slf4j;
    import org.springframework.boot.web.servlet.error.ErrorController;
    import org.springframework.http.HttpStatus;
    import org.springframework.stereotype.Controller;
    import org.springframework.ui.Model;
    import org.springframework.web.bind.annotation.RequestMapping;
    import org.springframework.web.bind.annotation.RequestMethod;

    @Slf4j
    @Controller
    public class CustomErrorController implements ErrorController {

        private static final String ERROR_PATH = "/error";

        @Override
        public String getErrorPath() {
            return ERROR_PATH;
        }

        @RequestMapping(value = "/error", method = RequestMethod.GET)
        public String handleError(HttpServletRequest request, Model model) {
            Object status = request.getAttribute(RequestDispatcher.ERROR_STATUS_CODE);
            HttpStatus httpStatus = HttpStatus.valueOf(Integer.valueOf(status.toString()));

            log.info("httpStatus : "+httpStatus.toString());

            model.addAttribute("code", status.toString());
            model.addAttribute("msg", httpStatus.getReasonPhrase());
            model.addAttribute("timestamp", new Date());
            return "error/error";
        }

    }
```

## View 생성
[error/error.html]
```
    <!DOCTYPE html>
    <html lang="en" xmlns:th="http://www.w3.org/1999/xhtml">
        <head>
            <meta charset="UTF-8">
            <title>Product Admin</title>
            <h1>Product Admin Error Page</h1>
            error code : <span th:text="${code}"></span>
            <br>error msg : <span th:text="${msg}"></span>
            <br>timestamp : <span th:text="${timestamp}"></span>
        </head>
        <body></body>
    </html>
```