---
title: Spring Boot Getting Started
description: 
author: laze
date: 2025-04-29 00:00:02 +0900
categories: [Dev, SpringBoot]
tags: [SpringBoot]
---
# Getting Started

### Required

- java 17 +
- Gradle 7.5+ or Maven 3.5+
- STS or IntelliJ IDEA or VSCode

```
git clone https://github.com/spring-guides/gs-spring-boot.git
```

**few example of automatic configuration Spring boot provides.**

- Spring MVC 가 classpath에 있을 때 필수로 필요한 bean들을 Spring Boot에서 자동으로 추가해 줌
- jetty가 classpath에 있다면 Tomcat을 사용하고 싶지 않을 것이므로 이를 핸들링해줌
- Thymeleaf 가 classpath에 있을 때 필수로 필요한 bean들을 Spring Boot에서 자동으로 추가해줌

Spring Boot는 코드나 파일을 생성해 주는 것이 아닌 애플리케이션 실행 시점에서 동적으로 bean 들과 설정을 application context에 적용함

### Spring Initializr

1. Navigate to [https://start.spring.io](https://start.spring.io/). This service pulls in all the dependencies you need for an application and does most of the setup for you.
2. Choose either Gradle or Maven and the language you want to use. This guide assumes that you chose Java.
3. Click **Dependencies** and select **Spring Web**.
4. Click **Generate**.
5. Download the resulting ZIP file, which is an archive of a web application that is configured with your choices.

**Create Simple Web Application**

```java
package com.example.springboot;

import org.springframework.web.bind.annotation.GetMapping;
import org.springframework.web.bind.annotation.RestController;

@RestController
public class HelloController {

	@GetMapping("/")
	public String index() {
		return "Greetings from Spring Boot!";
	}

}
```

- @RestController - Spring MVC가 웹 요청을 처리할 준비가 됐다는 의미의 flag
  - Controller + ResponseBody(결과로 html같은 view가 아닌 data를 전달하는 특징) → index() → text를 return
- @GetMapping(”/”)  -/ to indxe() 맵핑

**Create an Application calss**

```java
package com.example.springboot;

import java.util.Arrays;

import org.springframework.boot.CommandLineRunner;
import org.springframework.boot.SpringApplication;
import org.springframework.boot.autoconfigure.SpringBootApplication;
import org.springframework.context.ApplicationContext;
import org.springframework.context.annotation.Bean;

@SpringBootApplication
public class Application {

	public static void main(String[] args) {
		SpringApplication.run(Application.class, args);
	}

	@Bean
	public CommandLineRunner commandLineRunner(ApplicationContext ctx) {
		return args -> {

			System.out.println("Let's inspect the beans provided by Spring Boot:");

			String[] beanNames = ctx.getBeanDefinitionNames();
			Arrays.sort(beanNames);
			for (String beanName : beanNames) {
				System.out.println(beanName);
			}

		};
	}

}
```

- @SpringBootApplication 아래 전부를 추가해주는 convention
  - @Configuration - 해당 클래스를 애플리케이션 컨텍스트를 위한 빈(Bean) 설정 정보의 출처로 지정
  - @EnableAutoConfiguration - 스프링 부트에게 클래스패스 설정(프로젝트에 포함된 라이브러리들), 다른 빈(Bean)들, 그리고 다양한 속성 설정(application.properties 등)을 기반으로 필요한 빈들을 자동으로 추가하기 시작하라고 지시함.
    예를 들어, 만약 spring-webmvc 라이브러리가 클래스패스에 있다면, 이 어노테이션은 해당 애플리케이션을 웹 애플리케이션으로 인식하도록 표시하고, DispatcherServlet을 설정하는 것과 같은 핵심 동작들을 활성화
  - @ComponentScan - 스프링에게 해당 클래스가 위치한 패키지(com/example 같은)와 그 하위 패키지들을 탐색해서, 다른 컴포넌트(@Component), 설정(@Configuration), 서비스(@Service), 컨트롤러(@Controller) 등을 찾으라고 지시하여, 스프링이 컨트롤러 같은 것들을 찾을 수 있게 함
- XML 설정, web.xml 설정 같은 것이 필요없이 SpringApplication.run() method로 실행됨
  - 컴포넌트간의 연결이나, 인프라 구조 설정 등의 설정을 개발자가 직접 다룰 필요 없어짐
- 여기서 자주 나오는 classpath 란? → **자바 가상 머신(JVM)이나 자바 컴파일러가 실행될 때, 필요한 클래스 파일(.class)이나 라이브러리 파일(.jar)들을 어디서 찾아야 하는지를 알려주는 경로(path) 목록**

Run Application

```java
// when use gradle
./gradlew bootRun

// when use maven
./mvnw spring-boot:run
```

실행 결과

```
Let's inspect the beans provided by Spring Boot:
application
beanNameHandlerMapping
defaultServletHandlerMapping
dispatcherServlet
embeddedServletContainerCustomizerBeanPostProcessor
handlerExceptionResolver
helloController
httpRequestHandlerAdapter
messageSource
mvcContentNegotiationManager
mvcConversionService
mvcValidator
org.springframework.boot.autoconfigure.MessageSourceAutoConfiguration
org.springframework.boot.autoconfigure.PropertyPlaceholderAutoConfiguration
org.springframework.boot.autoconfigure.web.EmbeddedServletContainerAutoConfiguration
org.springframework.boot.autoconfigure.web.EmbeddedServletContainerAutoConfiguration$DispatcherServletConfiguration
org.springframework.boot.autoconfigure.web.EmbeddedServletContainerAutoConfiguration$EmbeddedTomcat
org.springframework.boot.autoconfigure.web.ServerPropertiesAutoConfiguration
org.springframework.boot.context.embedded.properties.ServerProperties
org.springframework.context.annotation.ConfigurationClassPostProcessor.enhancedConfigurationProcessor
org.springframework.context.annotation.ConfigurationClassPostProcessor.importAwareProcessor
org.springframework.context.annotation.internalAutowiredAnnotationProcessor
org.springframework.context.annotation.internalCommonAnnotationProcessor
org.springframework.context.annotation.internalConfigurationAnnotationProcessor
org.springframework.context.annotation.internalRequiredAnnotationProcessor
org.springframework.web.servlet.config.annotation.DelegatingWebMvcConfiguration
propertySourcesBinder
propertySourcesPlaceholderConfigurer
requestMappingHandlerAdapter
requestMappingHandlerMapping
resourceHandlerMapping
simpleControllerHandlerAdapter
tomcatEmbeddedServletContainerFactory
viewControllerHandlerMapping
```

- getmapping

```
curl http://localhost:8080
> Greetings from Spring Boot!%                                                                                    
```

Unit test 추가

```
// build.gradle (gradle)
testImplementation('org.springframework.boot:spring-boot-starter-test')

// pom.xml (maven)
<dependency>
	<groupId>org.springframework.boot</groupId>
	<artifactId>spring-boot-starter-test</artifactId>
	<scope>test</scope>
</dependency>
```

```java
package com.example.springboot;

import org.junit.jupiter.api.Test;
import org.springframework.beans.factory.annotation.Autowired;
import org.springframework.boot.test.autoconfigure.web.servlet.AutoConfigureMockMvc;
import org.springframework.boot.test.context.SpringBootTest;
import org.springframework.http.MediaType;
import org.springframework.test.web.servlet.MockMvc;
import org.springframework.test.web.servlet.request.MockMvcRequestBuilders;

import static org.hamcrest.Matchers.equalTo;
import static org.springframework.test.web.servlet.result.MockMvcResultMatchers.content;
import static org.springframework.test.web.servlet.result.MockMvcResultMatchers.status;

@SpringBootTest
@AutoConfigureMockMvc
public class HelloControllerTest {

    @Autowired
    private MockMvc mvc;

    @Test
    public void getHello() throws Exception {
        mvc.perform(MockMvcRequestBuilders.get("/").accept(MediaType.APPLICATION_JSON))
                .andExpect(status().isOk())
                .andExpect(content().string(equalTo("Greetings from Spring Boot!")));
    }
}

```

- MockMvc - Spring Test에서 제공되고 편리한 builder 클래스들을 통해 http 요청을 DispatcherServlet 에 보내고 결과에 대해 검증할 수 있게 해줌
- @SpringBootTest → 전체 애플리케이션 컨텍스트가 생성되도록 요청
- @WebMvcTest → 컨텍스트의 웹 계층만 생성하도록 요청할 수 있음
- 스프링부트는 기본적으로 메인 애플리케이션 클래스를 자동적으로 찾으려하지만 어노테이션을 통해 이를 override 하거나 범위를 좁힐 수 있음

```java
package com.example.springboot;

import org.junit.jupiter.api.Test;
import org.springframework.beans.factory.annotation.Autowired;
import org.springframework.boot.test.context.SpringBootTest;
import org.springframework.boot.test.web.client.TestRestTemplate;
import org.springframework.http.ResponseEntity;

import static org.assertj.core.api.Assertions.assertThat;

@SpringBootTest(webEnvironment = SpringBootTest.WebEnvironment.RANDOM_PORT)
public class HelloControllerITest {
    
    @Autowired
    private TestRestTemplate template;
    
    @Test
    public void getHello() throws Exception {
        ResponseEntity<String> response = template.getForEntity("/", String.class);
        assertThat(response.getBody()).isEqualTo("Greetings from Spring Boot!");
    }
}
```

```

2025-04-30T00:03:45.452+09:00  INFO 71527 --- [    Test worker] c.e.springboot.HelloControllerITest      : Starting HelloControllerITest using Java 17.0.14 with PID 71527 (started by laze in /Users/laze/Laze/Dev/SpringBoot/gs-spring-boot/gs-spring-boot/initial)
2025-04-30T00:03:45.452+09:00  INFO 71527 --- [    Test worker] c.e.springboot.HelloControllerITest      : No active profile set, falling back to 1 default profile: "default"
2025-04-30T00:03:45.710+09:00  INFO 71527 --- [    Test worker] o.s.b.w.embedded.tomcat.TomcatWebServer  : Tomcat initialized with port 0 (http)
2025-04-30T00:03:45.715+09:00  INFO 71527 --- [    Test worker] o.apache.catalina.core.StandardService   : Starting service [Tomcat]
2025-04-30T00:03:45.715+09:00  INFO 71527 --- [    Test worker] o.apache.catalina.core.StandardEngine    : Starting Servlet engine: [Apache Tomcat/10.1.24]
2025-04-30T00:03:45.736+09:00  INFO 71527 --- [    Test worker] o.a.c.c.C.[Tomcat].[localhost].[/]       : Initializing Spring embedded WebApplicationContext
2025-04-30T00:03:45.736+09:00  INFO 71527 --- [    Test worker] w.s.c.ServletWebServerApplicationContext : Root WebApplicationContext: initialization completed in 278 ms
2025-04-30T00:03:45.844+09:00  INFO 71527 --- [    Test worker] o.s.b.w.embedded.tomcat.TomcatWebServer  : Tomcat started on port 64657 (http) with context path '/'
2025-04-30T00:03:45.847+09:00  INFO 71527 --- [    Test worker] c.e.springboot.HelloControllerITest      : Started HelloControllerITest in 0.468 seconds (process running for 0.818)
2025-04-30T00:03:46.197+09:00  INFO 71527 --- [o-auto-1-exec-1] o.a.c.c.C.[Tomcat].[localhost].[/]       : Initializing Spring DispatcherServlet 'dispatcherServlet'
2025-04-30T00:03:46.197+09:00  INFO 71527 --- [o-auto-1-exec-1] o.s.web.servlet.DispatcherServlet        : Initializing Servlet 'dispatcherServlet'
2025-04-30T00:03:46.197+09:00  INFO 71527 --- [o-auto-1-exec-1] o.s.web.servlet.DispatcherServlet        : Completed initialization in 0 ms
```

- random port 를 사용해 포트 임의로 설정
- TestRestTemplate 객체가 알아서 **실제 할당된 포트 번호를 포함한 기본 URL(예: http://localhost:51234)을 가지도록 자동으로 구성**

**actuator module 추가**

```
// gradle
implementation 'org.springframework.boot:spring-boot-starter-actuator'

// maven
<dependency>
	<groupId>org.springframework.boot</groupId>
	<artifactId>spring-boot-starter-actuator</artifactId>
</dependency>
```

추가 이후 실행

```
// gradle
./gradlew bootRun

// maven
./mvnw spring-boot:run
```

- actuator를 통해 endpoints 들이 추가 되었는지 확인 할 수 있음
  - http://localhost:8080/actuator
  - curl -X POST http://localhost:8080/actuator
