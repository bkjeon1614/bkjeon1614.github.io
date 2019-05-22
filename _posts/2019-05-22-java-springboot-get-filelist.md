---
layout: post
title : "Spring Boot에서 특정 디렉토리에 있는 파일 목록 읽기"
subtitle : "2019-05-22-java-springboot-get-filelist.md"
date: 2019-05-22 20:41:00
author: "전봉근"
header-img: "img/posts/android-bg.jpg"
comments: true
tags: [java]
---

Spring Boot에서 특정 디렉토리에 있는 파일 목록 읽기
=========
- 설정 파일에 파일이 저장되어있는 경로를 설정(MAC 기준, 외장 tomcat 구성 기준)
[application.yml]
```
    ...

    # dev
    local-server:
        local-file-save-path: ${HOME}

    # live
    local-server:
        local-file-save-path: /var/lib/tomcat8/webapps

    ...
```

[FileServerProperties.java]
// application.yml에 설정된 파일 경로를 가져온다.
```
    import lombok.Getter;
    import lombok.Setter;
    import org.springframework.boot.context.properties.ConfigurationProperties;
    import org.springframework.context.annotation.Configuration;

    @Setter
    @Getter
    @Configuration
    @ConfigurationProperties(prefix = "local-server")
    public class FileServerProperties {
        private String localFileSavePath;   // /Users/we
    }
```

[Main.java]
```
    public static void main(String[] args) {
        File dirFile = new File(fileServerProperties.getLocalFileSavePath());
        File[] fileList = dirFile.listFiles();
        Arrays.sort(fileList, LastModifiedFileComparator.LASTMODIFIED_REVERSE);
        List<String> retList = new ArrayList<String>();

        for(File file: fileList) {
            if (file.getName().contains("upload_")) {   // upload_ 문자열이 있는지 체크
                retList.add(file.getName());    // 이름출력
            }
        }
    }
```