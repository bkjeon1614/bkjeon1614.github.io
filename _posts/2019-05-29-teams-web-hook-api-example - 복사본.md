---
layout: post
title : "Teams Web Hook API를 이용한 Message Sending 처리"
subtitle : "2019-05-29-teams-web-hook-api-example.md"
date: 2019-05-29 19:35:00
author: "전봉근"
header-img: "img/posts/android-bg.jpg"
comments: true
tags: [spring, teams]
---

Teams Web Hook API를 이용한 Message Sending 처리
=========
Teams 메신저를 사용할 경우 애플리케이션에 알람이 필요한 경우의 예제 코드이다.

## 메세지를 수신받을 Teams API 생성
(1) Teams 좌측 메뉴에서 "팀"을 클릭한다.  
![etc-teams-api-1](/img/posts/lib/teams/etc-teams-api-1.png)

(2) "채널 추가"를 클릭한다.
![etc-teams-api-2](/img/posts/lib/teams/etc-teams-api-2.png)

(3) 입력란에 입력 후 "추가"를 클릭한다.
![etc-teams-api-3](/img/posts/lib/teams/etc-teams-api-3.png)

(4) "커넥터"를 선택한다.
![etc-teams-api-4](/img/posts/lib/teams/etc-teams-api-4.png)

(5) "Incoming Webhook" 우측의 "구성" 버튼을 클릭한다.
![etc-teams-api-5](/img/posts/lib/teams/etc-teams-api-5.png)

(6) 이름을 입력하고 하단의 "만들기" 버튼 클릭.
![etc-teams-api-6](/img/posts/lib/teams/etc-teams-api-6.png)

(7) 제공하는 URL을 복사한다. (반드시 기억하고 있어야 한다.)
![etc-teams-api-7](/img/posts/lib/teams/etc-teams-api-7.png)

(8) 좌측 구성됨을 선택하여 확인할 수 있다.
![etc-teams-api-8](/img/posts/lib/teams/etc-teams-api-8.png)

(9) "이 채널 팔로우"를 선택하면 팀즈 채널로 메세지가 전송될 때 마다 알림이 표시된다.
![etc-teams-api-9](/img/posts/lib/teams/etc-teams-api-9.png)

(10) 해당 URL과 Contents-Type을 설정하고 json형태로 request를 보내어 테스트를 한다. (완료되면 1을 리턴)
```
    {
        "themeColor": "BCF7DA",
        "title": "테스트제목",
        "text": "테스트내용2"
    }
```
![etc-teams-api-10](/img/posts/lib/teams/etc-teams-api-10.png)

## 코드 작성
(1) 설정파일 내용 추가
   [application.yml]
   ```
        ...
            teams:
              error-url: {your URL}
        ...
   ```

(2) 클래스 작성
   [TeamsWebHookMessage.java]
   ```
        import com.fasterxml.jackson.annotation.JsonProperty;
        import lombok.Getter;
        import lombok.Setter;

        @Getter
        @Setter
        public class TeamsWebHookMessage {

            @JsonProperty("themeColor")
            private String color = "BCF7DA";

            @JsonProperty("title")
            private String title = "수신로그";

            @JsonProperty("text")
            private String text;

        }

   ```

   [TeamsWebHookApiUtil.java]
    ```     
        import com.fasterxml.jackson.core.JsonProcessingException;
        import com.fasterxml.jackson.databind.ObjectMapper;
        import com.wmp.admin.common.entity.TeamsWebHookMessage;
        import java.nio.charset.Charset;
        import javax.annotation.PostConstruct;
        import lombok.extern.slf4j.Slf4j;
        import org.apache.http.client.HttpClient;
        import org.apache.http.impl.client.HttpClientBuilder;
        import org.springframework.beans.factory.annotation.Value;
        import org.springframework.http.HttpEntity;
        import org.springframework.http.HttpHeaders;
        import org.springframework.http.HttpMethod;
        import org.springframework.http.MediaType;
        import org.springframework.http.ResponseEntity;
        import org.springframework.http.client.HttpComponentsClientHttpRequestFactory;
        import org.springframework.stereotype.Component;
        import org.springframework.web.client.RestTemplate;

        @Slf4j
        @Component
        public class TeamsWebHookApiUtil {

            private RestTemplate restTemplate;
            private HttpHeaders headers;

            @Value("${teams.error-url}")
            private String sendMessageErrorUrl;

            private ObjectMapper mapper = new ObjectMapper();

            @PostConstruct
            public void init() {
                HttpComponentsClientHttpRequestFactory factory = new HttpComponentsClientHttpRequestFactory();
                factory.setReadTimeout(5000); // 읽기시간초과, ms
                factory.setConnectTimeout(3000); // 연결시간초과, ms
                HttpClient httpClient = HttpClientBuilder.create()
                    .setMaxConnTotal(100) // connection pool 적용
                    .setMaxConnPerRoute(5) // connection pool 적용
                    .build();
                factory.setHttpClient(httpClient); // 동기실행에 사용될 HttpClient 세팅
                restTemplate = new RestTemplate(factory);
                headers = new HttpHeaders();
                headers.setContentType(MediaType.APPLICATION_JSON);
                Charset utf8 = Charset.forName("UTF-8");
                MediaType mediaType = new MediaType("application", "json", utf8);
                headers.setContentType(mediaType);
            }

            public String sendErrorMessage(String text, String color) {
                return requestHttpWookCall(text, color, sendMessageErrorUrl);
            }

            private String requestHttpWookCall(String text, String color, String sendMessageErrorUrl) {
                TeamsWebHookMessage message = new TeamsWebHookMessage();
                message.setText(text);
                message.setColor(color);
                HttpEntity<String> request = null;
                try {
                    request = new HttpEntity<>(mapper.writeValueAsString(message), headers);
                    ResponseEntity<String> result = restTemplate
                        .exchange(sendMessageErrorUrl, HttpMethod.POST, request, String.class);
                    log.debug("Telegram sendMessage result={}", result);
                    return result.getBody();
                } catch (JsonProcessingException e) {
                    e.printStackTrace();
                    log.error("Error found During sending telegram message-parse {}", e.getMessage());
                    return null;
                }
            }
        }
    ```

(3) 컨트롤러에서 사용 예시
   [TestController.java]
   ```
        public class TestController {

            @Autowired
            private TeamsWebHookApiUtil teamsWebHookApiUtil;

            @GetMapping("test")
            public void sendTest() {
                try {
                    new Thread(() -> {
                        System.out.println(teamsWebHookApiUtil.sendErrorMessage("테스트", "BCF7DA"));
                    }).start();
                } catch (Exception e) {
                    log.error("Teams Web Hook Api Error {}", e.getMessage());
                }
            }

        }
   ```

## 결과
![etc-teams-api-11](/img/posts/lib/teams/etc-teams-api-11.png) 