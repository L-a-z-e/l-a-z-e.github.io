---
title: Spring Resource 로딩 ClasspathResource vs. FileSystemResource, 언제 무엇을 써야 할까?
description: 
author: laze
date: 2025-07-07 00:00:03 +0900
categories: [Dev, Java]
tags: [Java]
---
# Spring Resource 로딩: ClasspathResource vs. FileSystemResource, 언제 무엇을 써야 할까?

## 1. 서론: 리소스 로딩의 혼란

Spring 기반의 애플리케이션을 개발할 때, 설정 파일이나 템플릿 같은 리소스(Resource)를 로딩하는 작업은 필수적이다. 하지만 이 과정에서 많은 개발자들이 혼란을 겪는다.

- `ClassPathResource`는 언제 쓰고, `FileSystemResource`는 언제 쓰는가?
- 경로 앞에 `classpath:`는 왜 붙이는가?
- `new File(...)`과 무엇이 다른가?

본 포스팅은 Spring의 `Resource` 추상화 개념을 명확히 이해하고, 다양한 상황에 맞는 올바른 리소스 처리 방법을 선택하는 기준을 정립하는 것을 목표로 한다.

## 2. Spring의 해답: `Resource` 인터페이스로 모든 리소스를 추상화하다

Java의 저수준 API(`FileInputStream`, `URL.openStream()` 등)는 리소스의 위치(파일 시스템, 클래스패스, URL 등)에 따라 사용법이 달라 코드의 복잡성을 높인다.

Spring은 이 문제를 해결하기 위해 `org.springframework.core.io.Resource`라는 추상 인터페이스를 도입했다.

`Resource` 인터페이스는 리소스의 물리적 위치에 상관없이, `getInputStream()`과 같은 일관된 메서드로 리소스에 접근할 수 있는 단일 창구를 제공한다.

## 3. 핵심 `Resource` 구현체 분석: 언제 무엇을 쓰는가?

`Resource` 인터페이스는 다양한 구현체를 통해 여러 종류의 리소스를 지원한다. 각 구현체의 용도를 이해하는 것이 리소스 로딩의 첫걸음이다.

### 3.1. `ClassPathResource`: 애플리케이션 내장 리소스

- **용도:** `src/main/resources` 디렉토리처럼 **애플리케이션과 함께 빌드되어 JAR/WAR 내부에 포함되는 리소스**를 다룰 때 사용한다. 설정 파일, 템플릿, 정적 데이터 파일 등이 여기에 해당한다.
- **특징:** 클래스패스(Classpath)를 기준으로 리소스를 찾으므로, **배포 환경에 상관없이 가장 안정적으로 동작**한다. 실무에서 가장 빈번하게 사용되는 구현체다.
- **주의사항:** JAR 파일 내부의 리소스는 실제 파일 시스템의 파일이 아니므로, `resource.getFile()` 메서드는 예외를 발생시킨다. 항상 `resource.getInputStream()`으로 접근해야 한다.

### 예시 코드

프로젝트 구조:

```
src/main/resources/
└── config/
    └── application.yml
```

```java
import org.springframework.core.io.ClassPathResource;
import org.springframework.core.io.Resource;
import java.io.InputStream;
import java.nio.charset.StandardCharsets;

public class ClasspathExample {
    public void readFromClasspath() throws Exception {
        // 클래스패스 루트를 기준으로 경로 지정
        Resource resource = new ClassPathResource("config/application.yml");

        if (resource.exists()) {
            try (InputStream inputStream = resource.getInputStream()) {
                // InputStream을 사용하여 파일 내용 읽기
                String content = new String(inputStream.readAllBytes(), StandardCharsets.UTF_8);
                System.out.println(content);
            }
        }
    }
}
```

### 3.2. `FileSystemResource`: 애플리케이션 외부 파일

- **용도:** 애플리케이션 외부, 즉 **서버의 특정 파일 시스템 경로에 존재하는 파일**을 다룰 때 사용한다. 외부에서 주입해야 하는 설정 파일, 동적으로 생성되는 로그 파일, 업로드된 파일 등이 대상이다.
- **특징:** OS의 파일 시스템 경로를 직접 사용하므로, 특정 서버 환경에 종속된다. 이로 인해 애플리케이션의 이식성이 저하될 수 있다.

### 예시 코드

```java
import org.springframework.core.io.FileSystemResource;
import org.springframework.core.io.Resource;
import java.io.File;
import java.io.InputStream;

public class FileSystemExample {
    public void readFromFileSystem() throws Exception {
        // OS의 절대 경로 또는 상대 경로를 사용
        String filePath = "/etc/app/external-config.properties";
        Resource resource = new FileSystemResource(filePath);

        // FileSystemResource는 getFile() 호출이 가능
        if (resource.exists()) {
            File file = resource.getFile();
            System.out.println("File path: " + file.getAbsolutePath());

            try (InputStream inputStream = resource.getInputStream()) {
                // ... 내용 읽기 ...
            }
        }
    }
}
```

### 3.3. `UrlResource`: URL 기반 리소스

- **용도:** `http://`, `https://`, `ftp://`, `file://` 등의 프로토콜을 사용하여 접근 가능한 원격지 또는 로컬의 리소스를 다룰 때 사용한다.
- **특징:** 웹상의 파일을 다운로드하거나, 파일 시스템의 파일을 URL 형식으로 표현할 때 유용하다.

### 예시 코드

```java
import org.springframework.core.io.Resource;
import org.springframework.core.io.UrlResource;
import java.io.InputStream;
import java.net.URL;

public class UrlResourceExample {
    public void readFromUrl() throws Exception {
        Resource resource = new UrlResource("https://start.spring.io");

        if (resource.exists()) {
            URL url = resource.getURL();
            System.out.println("URL: " + url.toString());

            try (InputStream inputStream = resource.getInputStream()) {
                // ... 내용 읽기 ...
            }
        }
    }
}
```

## 4. 최고의 실무 전략: `ResourceLoader`에게 위임하기

실무에서는 위 구현체들을 직접 `new` 키워드로 생성하기보다, Spring의 **`ResourceLoader`**를 사용하는 것이 훨씬 유연하고 편리하다. `ApplicationContext` 자체가 `ResourceLoader`이므로, 개발자는 리소스의 위치를 나타내는 문자열만 전달하면 된다. `ResourceLoader`는 해당 문자열의 **접두어(prefix)**를 분석하여 자동으로 최적의 `Resource` 구현체를 반환해준다.

| 접두어 | 선택되는 `Resource` 구현체 | 예시 |
| --- | --- | --- |
| `classpath:` | `ClassPathResource` | `classpath:config/app.yml` |
| `file:` | `FileSystemResource` | `file:/etc/app/config.properties` |
| `http:`, `https://:` | `UrlResource` | `https://example.com/data.json` |

### 4.1. `@Value` 어노테이션 활용 (가장 간편한 방법)

`@Value` 어노테이션에 접두어를 포함한 경로를 지정하면, Spring이 자동으로 해당 리소스를 주입해준다.

```java
import org.springframework.beans.factory.annotation.Value;
import org.springframework.core.io.Resource;
import org.springframework.stereotype.Component;

@Component
public class ValueResourceLoader {

    @Value("classpath:config/application.yml")
    private Resource classpathResource;

    @Value("file:/etc/app/external-config.properties")
    private Resource fileSystemResource;

    public void printResources() {
        System.out.println("Classpath resource exists: " + classpathResource.exists());
        System.out.println("File system resource exists: " + fileSystemResource.exists());
    }
}
```

### 4.2. `ResourceLoader` 직접 주입

더 동적인 로직이 필요할 경우 `ResourceLoader`를 직접 주입받아 사용할 수 있다.

```java
import org.springframework.beans.factory.annotation.Autowired;
import org.springframework.core.io.Resource;
import org.springframework.core.io.ResourceLoader;
import org.springframework.stereotype.Service;

@Service
public class DynamicResourceLoader {

    @Autowired
    private ResourceLoader resourceLoader;

    public Resource load(String location) {
        // location 문자열에 따라 적절한 Resource가 반환된다.
        // 예: "classpath:data.csv", "file:c:/temp/data.csv"
        return resourceLoader.getResource(location);
    }
}
```

## 5. `classpath:`와 `classpath*:`의 차이

- `classpath:`: 경로와 일치하는 **첫 번째** 리소스를 찾는다. 여러 개의 JAR 파일에 동일한 경로의 리소스가 존재하더라도, 클래스패스 순서상 가장 먼저 발견된 하나만 반환한다.
- `classpath*:`: 경로와 일치하는 **모든** 리소스를 찾아 `Resource` 배열로 반환한다. 여러 모듈이나 라이브러리에서 동일한 이름의 설정 파일을 모두 로드해야 할 때 유용하다.

```java
// 여러 JAR에 흩어져 있는 my-plugin.xml 파일을 모두 찾는다.
@Value("classpath*:META-INF/my-plugin.xml")
private Resource[] pluginFiles;
```

## 6. 결론: 리소스 로딩을 위한 의사결정 가이드

복잡해 보이지만, 리소스 로딩의 기준은 명확하다.

1. **애플리케이션과 함께 배포되어야 하는가?** (90%의 경우)
  - **Yes** → `classpath:` 접두어를 사용한다. (내부적으로 `ClassPathResource` 동작)
2. **애플리케이션 외부 파일 시스템에 존재하는가?**
  - **Yes** → `file:` 접두어를 사용한다. (내부적으로 `FileSystemResource` 동작)
3. **URL을 통해 접근해야 하는가?**
  - **Yes** → `http:`, `https://` 등 URL 전체를 경로로 사용한다. (내부적으로 `UrlResource` 동작)
4. **코드를 통해 동적으로 위치를 지정해야 하는가?**
  - **Yes** → `ResourceLoader`를 직접 주입받아 `getResource(location)` 메서드를 활용한다.

이 가이드를 따르면, 어떤 상황에서든 혼란 없이 최적의 방법으로 Spring `Resource`를 다룰 수 있다.

---
