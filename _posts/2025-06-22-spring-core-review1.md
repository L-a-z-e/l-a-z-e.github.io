---
title: Spring Core - Review(1)
description: 
author: laze
date: 2025-06-22 00:00:01 +0900
categories: [Dev, SpringWebMVC]
tags: [SpringWebMVC]
---
### **IoC, 스프링 IoC 컨테이너, 빈(Bean) 개념 정리**

**1. IoC(제어의 역전)란 무엇이며, 전통적인 방식과 어떻게 다른가?**

- 핵심은 **객체의 생성, 구성, 생명주기 관리의 "제어권"이 개발자에게서 프레임워크(여기서는 스프링 컨테이너)로 넘어갔다**는 것입니다.
- **전통적인 방식:** 개발자가 코드 내에서 `new MyService()` 와 같이 직접 객체를 생성하고, 생성된 객체들 간의 의존 관계도 직접 설정합니다.
  `MyController`가 `MyService`를 필요로 한다면, `MyController` 내부에서 `MyService`를 생성하거나 가져오는 코드를 작성해야 합니다.

    ```java
    // 전통적인 방식 예시 (의존성을 직접 해결)
    public class MyController {
        private MyService myService;
    
        public MyController() {
            this.myService = new MyServiceImpl(); // MyController가 MyService를 직접 생성하고 의존함
        }
    
        public void doSomething() {
            myService.performAction();
        }
    }
    ```

- **IoC 방식 (스프링):** 객체(빈)의 생성과 의존 관계 설정은 스프링 컨테이너가 담당합니다.
  개발자는 어떤 객체들이 필요하고, 그 객체들이 어떻게 연결되어야 하는지에 대한 "설정 정보"(XML, 어노테이션, Java Config 등)만 제공하면 됩니다.
  그러면 스프링 컨테이너가 이 정보를 바탕으로 객체들을 만들고, 필요한 곳에 "주입(Injection)"해줍니다.
  위 예시에서 `MyController`는 `MyService`를 직접 생성하지 않습니다.
  스프링 컨테이너가 `MyService`의 구현체 빈을 찾아 생성자의 인자로 주입해줍니다. 이것이 바로 "제어의 역전"입니다.

    ```java
    // 스프링 IoC 방식 예시 (의존성 주입)
    // (MyServiceImpl은 @Service, MyController는 @Controller로 등록되었다고 가정)
    @Controller
    public class MyController {
        private final MyService myService; // final로 선언하여 생성자 주입 강제 가능
    
        @Autowired // 스프링 컨테이너가 MyService 타입의 빈을 찾아 자동으로 주입
        public MyController(MyService myService) {
            this.myService = myService;
        }
    
        public void doSomething() {
            myService.performAction();
        }
    }
    ```


**2. 스프링 IoC 컨테이너의 주요 역할은 무엇인가?**

- 핵심 역할은 **빈(Bean)의 생명주기 관리**와 **의존성 주입(Dependency Injection, DI)**입니다.
- 조금 더 구체적으로 나열해보면 다음과 같습니다:
  - **객체(빈) 생성:** 설정 정보를 바탕으로 빈 인스턴스를 생성합니다.
  - **의존성 설정:** 빈들 간의 의존 관계를 설정합니다 (DI).
  - **생명주기 관리:** 빈의 생성부터 소멸까지의 전체 생명주기를 관리합니다 (초기화 콜백, 소멸 콜백 등).
  - **빈 제공:** 애플리케이션의 다른 부분에서 필요할 때 관리하고 있는 빈을 제공(조회)합니다.
  - **다양한 부가 기능 제공:** AOP, 트랜잭션 관리, 메시지 처리 등 스프링의 다른 기능들과 연동하여 빈에 부가 기능을 적용할 수 있는 기반을 제공합니다.

**3. 빈(Bean)은 일반적인 자바 객체와 어떤 차이점이 있는가?**

- **일반 자바 객체:** 개발자가 `new` 키워드를 사용하여 직접 생성하고, 소멸(가비지 컬렉션)도 JVM에 의해 관리되지만, 객체 간의 관계 설정이나 특별한 생명주기 관리는 개발자의 책임입니다.
- **스프링 빈:**
  - **스프링 IoC 컨테이너에 의해 생성되고 관리되는 객체**입니다.
  - 컨테이너는 빈의 **생성, 설정, 초기화, 사용, 소멸**까지의 전 과정을 관리합니다.
  - 개발자는 빈으로 등록될 클래스를 만들고, 스프링에게 어떻게 이 빈을 생성하고 구성할지(XML, 어노테이션, Java Config) 알려주기만 하면 됩니다.
  - **코드 예시 (어노테이션 기반):**

      ```java
      import org.springframework.stereotype.Component;
      
      @Component // 이 클래스의 객체를 스프링 빈으로 등록하라는 어노테이션
      public class MySampleBean {
          public void showMessage() {
              System.out.println("Hello from MySampleBean!");
          }
      }
      ```

    위와 같이 `@Component` 어노테이션을 붙이면, 스프링 컨테이너는 컴포넌트 스캔 과정에서 `MySampleBean` 클래스를 발견하고, 이 클래스의 인스턴스(빈)를 생성하여 내부적으로 관리합니다.

    그리고 다른 빈에서 `@Autowired` 등을 통해 이 `MySampleBean` 타입의 빈을 주입받아 사용할 수 있게 됩니다.

      ```java
      import org.springframework.beans.factory.annotation.Autowired;
      import org.springframework.stereotype.Service;
      
      @Service
      public class AnotherService {
          private final MySampleBean mySampleBean;
      
          @Autowired
          public AnotherService(MySampleBean mySampleBean) { // MySampleBean 빈이 주입됨
              this.mySampleBean = mySampleBean;
          }
      
          public void useBean() {
              mySampleBean.showMessage();
          }
      }
      ```


---

**1. `BeanFactory`와 `ApplicationContext`**

- 일반적으로 애플리케이션 개발 시에는 **`ApplicationContext`를 주로 사용**합니다.
- **관계:** `ApplicationContext`는 `BeanFactory` 인터페이스를 **상속**받습니다. 즉, `ApplicationContext`는 `BeanFactory`가 제공하는 모든 기능을 포함하면서, 추가적으로 더 많은 엔터프라이즈급 기능들을 제공합니다.
- **`BeanFactory`의 핵심 기능:**
  - 스프링 IoC 컨테이너의 **가장 기본적인 형태**입니다.
  - 주된 역할은 **빈(Bean)을 생성하고 관리하며, 의존성을 주입**하는 것입니다.
  - 빈을 요청할 때(지연 로딩, lazy-loading) 생성하는 것이 기본 동작인 경우가 많습니다. (설정에 따라 다를 수 있음)
- **`ApplicationContext`가 추가적으로 제공하는 주요 기능 (그래서 주로 사용되는 이유):**
  - **메시지 소스(MessageSource)를 이용한 국제화(i18n) 기능:** 다국어 지원을 쉽게 할 수 있습니다.
  - **이벤트 발행(Event Publishing) 메커니즘:** 애플리케이션 내에서 발생하는 이벤트를 발행하고 구독하여 처리할 수 있습니다. (예: `ApplicationEvent` 및 `ApplicationListener`)
  - **리소스 로딩(Resource Loading)의 편리한 방법:** 파일 시스템, 클래스패스, URL 등 다양한 위치의 리소스를 일관된 방식으로 로드할 수 있습니다. (예: `ResourceLoader` 기능)
  - **자동 `BeanPostProcessor` 등록, 자동 `BeanFactoryPostProcessor` 등록:** 컨테이너의 확장 기능을 더 쉽게 사용할 수 있도록 합니다.
  - **애플리케이션 시작 시 대부분의 싱글톤 빈 즉시 로딩(pre-instantiation):** 설정 오류 등을 애플리케이션 시작 시점에 빠르게 발견할 수 있게 해줍니다. (이것도 설정에 따라 변경 가능)
  - **다양한 `ApplicationContext` 구현체:** 웹 환경(`WebApplicationContext`), 테스트 환경 등 다양한 환경에 맞는 구현체들이 제공됩니다.

    ---

  **1. 메시지 소스(MessageSource)를 이용한 국제화(i18n) 기능**

  - **개념:** 사용자의 언어 설정(Locale)에 따라 다른 메시지를 보여줄 수 있게 합니다. 메시지를 코드에서 분리하여 관리합니다.
  - **코드 예시:**

    **메시지 프로퍼티 파일 준비:**

    - `messages_en.properties` (영어)

        ```
        greeting=Hello, {0}!
        ```

    - `messages_ko.properties` (한국어)

        ```
        greeting=안녕하세요, {0}님!
        ```


        **`MessageSource` 빈 설정 (Java Config 예시):**
        
        ```java
        import org.springframework.context.MessageSource;
        import org.springframework.context.annotation.Bean;
        import org.springframework.context.annotation.Configuration;
        import org.springframework.context.support.ReloadableResourceBundleMessageSource;
        
        @Configuration
        public class AppConfig {
            @Bean
            public MessageSource messageSource() {
                ReloadableResourceBundleMessageSource messageSource = new ReloadableResourceBundleMessageSource();
                messageSource.setBasename("classpath:messages"); // "messages"로 시작하는 프로퍼티 파일 사용
                messageSource.setDefaultEncoding("UTF-8");
                return messageSource;
            }
        }
        ```
        
        **`ApplicationContext`를 통해 `MessageSource` 사용:**
        
        ```java
        import org.springframework.context.ApplicationContext;
        import org.springframework.context.MessageSource;
        import org.springframework.context.annotation.AnnotationConfigApplicationContext;
        import java.util.Locale;
        
        public class I18nExample {
            public static void main(String[] args) {
                ApplicationContext context = new AnnotationConfigApplicationContext(AppConfig.class);
                MessageSource messageSource = context.getBean(MessageSource.class);
        
                // 또는 ApplicationContext 자체가 MessageSource를 구현하므로 바로 사용 가능
                // String greetingEn = context.getMessage("greeting", new Object[]{"Spring"}, Locale.ENGLISH);
                // String greetingKo = context.getMessage("greeting", new Object[]{"스프링"}, Locale.KOREAN);
        
                String greetingEn = messageSource.getMessage("greeting", new Object[]{"Spring"}, Locale.ENGLISH);
                String greetingKo = messageSource.getMessage("greeting", new Object[]{"스프링"}, Locale.KOREAN);
        
                System.out.println("English: " + greetingEn); // English: Hello, Spring!
                System.out.println("Korean: " + greetingKo);  // Korean: 안녕하세요, 스프링님!
            }
        }
        ```
        
    
    **2. 이벤트 발행(Event Publishing) 메커니즘**
    
    - **개념:** 애플리케이션 내의 한 부분에서 특정 이벤트가 발생했음을 알리면(발행), 다른 부분에서 그 이벤트를 받아(구독) 처리할 수 있게 하는 디자인 패턴입니다. 컴포넌트 간의 결합도를 낮춥니다.
    - **코드 예시:**
        
        **커스텀 이벤트 정의 (`ApplicationEvent` 상속):**
        
        ```java
        import org.springframework.context.ApplicationEvent;
        
        public class MyCustomEvent extends ApplicationEvent {
            private String message;
        
            public MyCustomEvent(Object source, String message) {
                super(source);
                this.message = message;
            }
        
            public String getMessage() {
                return message;
            }
        ```
        
        **이벤트 리스너 정의 (`ApplicationListener` 구현 또는 `@EventListener` 사용):**
        
        ```java
        import org.springframework.context.ApplicationListener;
        import org.springframework.stereotype.Component;
        // 또는 import org.springframework.context.event.EventListener;
        
        @Component
        public class MyCustomEventListener implements ApplicationListener<MyCustomEvent> {
            @Override
            public void onApplicationEvent(MyCustomEvent event) {
                System.out.println("Received custom event - Message: " + event.getMessage());
                System.out.println("Event source: " + event.getSource().getClass().getName());
            }
        }
        
        // @EventListener를 사용하는 더 간결한 방법
        @Component
        public class MyAnnotationEventListener {
            @EventListener
            public void handleCustomEvent(MyCustomEvent event) {
                System.out.println("Annotation Listener received custom event - Message: " + event.getMessage());
            }
        }
        ```
        
        **이벤트 발행자 (일반 빈):**
        
        ```java
        import org.springframework.beans.factory.annotation.Autowired;
        import org.springframework.context.ApplicationEventPublisher;
        import org.springframework.stereotype.Component;
        
        @Component
        public class MyEventPublisher {
            @Autowired
            private ApplicationEventPublisher applicationEventPublisher; // ApplicationContext가 주입해줌
        
            public void publishEvent(String message) {
                System.out.println("Publishing custom event with message: " + message);
                MyCustomEvent customEvent = new MyCustomEvent(this, message);
                applicationEventPublisher.publishEvent(customEvent);
            }
        }
        ```
        
        **실행 예시:**
        
        ```java
        // (ApplicationContext 설정 및 빈 스캔이 되어 있다고 가정)
        public class EventExample {
            public static void main(String[] args) {
                ApplicationContext context = new AnnotationConfigApplicationContext("com.example.eventpackage"); // 빈 스캔 패키지
                MyEventPublisher publisher = context.getBean(MyEventPublisher.class);
                publisher.publishEvent("Hello Spring Event!");
                // 콘솔에 "Received custom event..." 메시지가 출력됨
            }
        }
        ```
        
    
    **3. 리소스 로딩(Resource Loading)의 편리한 방법**
    
    - **개념:** `classpath:`, `file:`, `http:` 등 다양한 접두사를 사용하여 파일 시스템, 클래스패스, URL 등의 리소스를 일관된 방식으로 로드할 수 있습니다. `ApplicationContext`는 `ResourceLoader` 역할을 합니다.
    - **코드 예시:**
        
        ```java
        import org.springframework.context.ApplicationContext;
        import org.springframework.context.support.ClassPathXmlApplicationContext; // 예시로 XML 컨텍스트 사용
        import org.springframework.core.io.Resource;
        import java.io.BufferedReader;
        import java.io.InputStreamReader;
        
        public class ResourceLoadingExample {
            public static void main(String[] args) throws Exception {
                // ApplicationContext 자체가 ResourceLoader
                // 여기서는 간단히 ClassPathXmlApplicationContext를 사용했지만,
                // AnnotationConfigApplicationContext 등 다른 컨텍스트도 동일하게 ResourceLoader 기능 제공
                ApplicationContext context = new ClassPathXmlApplicationContext(); // 실제 XML 파일 없이도 ResourceLoader로 사용 가능
        
                // 클래스패스 리소스 로드
                Resource classpathResource = context.getResource("classpath:mytextfile.txt");
                // 파일 시스템 리소스 로드 (절대 경로 또는 상대 경로)
                // Resource fileResource = context.getResource("file:/path/to/your/file.txt");
                // URL 리소스 로드
                // Resource urlResource = context.getResource("<https://example.com/some.json>");
        
                if (classpathResource.exists()) {
                    System.out.println("Resource found: " + classpathResource.getFilename());
                    try (BufferedReader reader = new BufferedReader(new InputStreamReader(classpathResource.getInputStream()))) {
                        String line;
                        while ((line = reader.readLine()) != null) {
                            System.out.println(line);
                        }
                    }
                } else {
                    System.out.println("Resource not found.");
                }
            }
        }
        ```
        
    
    **4. 자동 `BeanPostProcessor` 등록, 자동 `BeanFactoryPostProcessor` 등록**
    
    - **개념:** `Bean(Factory)PostProcessor`는 스프링 컨테이너의 빈 생성 및 초기화 과정에 개입하여 빈을 커스터마이징하거나 컨테이너 설정을 변경할 수 있는 강력한 확장 메커니즘입니다. `ApplicationContext`는 이러한 후처리기들을 자동으로 감지하고 등록하여 활성화합니다. `BeanFactory`는 수동으로 등록해야 하는 경우가 많습니다.
    - **코드 예시 (`BeanPostProcessor`):**
    위 `MyBeanPostProcessor`는 `MyServiceType`의 빈이 초기화되기 전과 후에 메시지를 출력합니다. `ApplicationContext`는 `@Component`로 선언된 `MyBeanPostProcessor`를 자동으로 인식하고 모든 빈 생성 시 적용합니다.
        
        ```java
        import org.springframework.beans.BeansException;
        import org.springframework.beans.factory.config.BeanPostProcessor;
        import org.springframework.stereotype.Component;
        
        @Component // ApplicationContext가 자동으로 감지하고 등록
        public class MyBeanPostProcessor implements BeanPostProcessor {
            @Override
            public Object postProcessBeforeInitialization(Object bean, String beanName) throws BeansException {
                if (bean instanceof MyServiceType) { // 특정 타입의 빈에만 적용
                    System.out.println("BeanPostProcessor: Before Initialization of " + beanName);
                }
                return bean; // 항상 원본 빈 또는 수정된 빈을 반환해야 함
            }
        
            @Override
            public Object postProcessAfterInitialization(Object bean, String beanName) throws BeansException {
                if (bean instanceof MyServiceType) {
                    System.out.println("BeanPostProcessor: After Initialization of " + beanName);
                    // 여기서 빈을 프록시로 감싸거나 다른 로직 추가 가능
                }
                return bean;
            }
        }
        
        // 인터페이스 및 빈 예시
        interface MyServiceType { void doService(); }
        @Component
        class MyServiceImpl implements MyServiceType {
            public MyServiceImpl() { System.out.println("MyServiceImpl constructor called"); }
            public void doService() { System.out.println("MyServiceImpl is doing service"); }
            // @PostConstruct public void init() { System.out.println("MyServiceImpl init method"); }
        }
        ```
        
    
    **5. 애플리케이션 시작 시 대부분의 싱글톤 빈 즉시 로딩(pre-instantiation)**
    
    - **개념:** `ApplicationContext`는 기본적으로 싱글톤 스코프의 빈들을 컨텍스트 초기화 시점에 미리 생성하고 초기화합니다. 이를 통해 설정 오류(예: 잘못된 의존성 주입, 클래스 찾을 수 없음)를 애플리케이션 구동 초기에 발견할 수 있습니다. `BeanFactory`는 기본적으로 빈을 요청할 때 생성하는 지연 로딩(lazy loading) 방식을 따르는 경우가 많습니다.
    - **코드 예시 (특별한 코드는 없음, 동작 방식의 차이):**`ApplicationContext`는 시작 시점에 더 많은 작업을 수행하여 오류를 조기에 발견하는 장점이 있습니다. (물론, 빈 정의에 `@Lazy`를 붙이면 `ApplicationContext`에서도 지연 로딩이 가능합니다.)
        
        ```java
        // ApplicationContext 사용 시
        ApplicationContext context = new AnnotationConfigApplicationContext(AppConfig.class);
        // 이 시점에서 AppConfig에 정의된 싱글톤 빈들이 대부분 생성 및 초기화됨
        // 만약 빈 생성/초기화에 문제가 있다면 여기서 예외 발생
        
        // BeanFactory 사용 시 (예시)
        // DefaultListableBeanFactory factory = new DefaultListableBeanFactory();
        // XmlBeanDefinitionReader reader = new XmlBeanDefinitionReader(factory);
        // reader.loadBeanDefinitions(new ClassPathResource("beans.xml"));
        // MyBean bean = factory.getBean("myBean", MyBean.class); // 이때 myBean이 처음 생성될 수 있음
        ```
        
    
    **6. 다양한 `ApplicationContext` 구현체**
    
    - **개념:** 스프링은 다양한 실행 환경에 맞는 `ApplicationContext` 구현체들을 제공합니다.
    - **예시:**
    이러한 다양한 구현체 덕분에 개발자는 환경에 맞는 컨테이너를 쉽게 선택하여 사용할 수 있습니다.
        - `AnnotationConfigApplicationContext`: 자바 기반 설정(`@Configuration` 클래스)을 로드.
        - `ClassPathXmlApplicationContext`: 클래스패스에 있는 XML 설정 파일을 로드.
        - `FileSystemXmlApplicationContext`: 파일 시스템 경로의 XML 설정 파일을 로드.
        - **`AnnotationConfigServletWebServerApplicationContext` (스프링 부트 웹 애플리케이션):** 스프링 부트가 웹 애플리케이션을 실행할 때 내부적으로 사용하는 `WebApplicationContext`의 구현체입니다. 내장 웹 서버(Tomcat, Jetty 등)를 시작하고 웹 환경에 필요한 설정을 구성합니다.
            
            ```java
            // 스프링 부트 애플리케이션의 main 메소드 (간략화)
             public class MyApplication {
                 public static void main(String[] args) {
                     SpringApplication.run(MyApplication.class, args);
                     // 내부적으로 AnnotationConfigServletWebServerApplicationContext 등이 사용됨
                 }
             }
            ```
            
    
    ---


**2. 빈(Bean)을 정의하는 세 가지 주요 방법**

1. **XML 기반 설정 (XML-based configuration):**
  - **특징:** 스프링 초기부터 사용된 전통적인 방식입니다. 빈의 정의, 의존 관계, 라이프사이클 등을 XML 파일에 명시적으로 작성합니다.
  - **주요 태그:** `<beans>` 루트 태그 안에 `<bean>` 태그를 사용하여 각 빈을 정의합니다.
  - **코드 예시 (XML):**

      ```xml
      <beans xmlns="<http://www.springframework.org/schema/beans>"
             xmlns:xsi="<http://www.w3.org/2001/XMLSchema-instance>"
             xsi:schemaLocation="<http://www.springframework.org/schema/beans> <http://www.springframework.org/schema/beans/spring-beans.xsd>">
      
          <bean id="myService" class="com.example.MyServiceImpl">
              <!-- 생성자 주입 예시 -->
              <constructor-arg ref="myRepository"/>
          </bean>
      
          <bean id="myRepository" class="com.example.MyRepositoryImpl"/>
      
      </beans>
      
      ```

  - 최근에는 사용 빈도가 줄어들고 있지만, 레거시 시스템 유지보수나 일부 복잡한 설정에는 여전히 사용될 수 있습니다.
2. **어노테이션 기반 설정 (Annotation-based configuration):**
  - **특징:** 자바 클래스에 `@Component`, `@Service`, `@Repository`, `@Controller` 등의 스테레오타입 어노테이션을 붙여 해당 클래스가 스프링 빈으로 관리되도록 합니다. 의존성 주입은 `@Autowired`, `@Resource` 등을 사용합니다.
  - **코드 예시 (Java):**

      ```java
      package com.example;
      
      import org.springframework.stereotype.Component;
      
      @Component // 또는 @Service, @Repository, @Controller 등
      public class MyComponent {
          public void doWork() {
              System.out.println("MyComponent is working!");
          }
      }
      ```

      ```java
      package com.example;
      
      import org.springframework.beans.factory.annotation.Autowired;
      import org.springframework.stereotype.Service;
      
      @Service
      public class AnotherComponent {
          private final MyComponent myComponent;
      
          @Autowired
          public AnotherComponent(MyComponent myComponent) {
              this.myComponent = myComponent;
          }
      
          public void useMyComponent() {
              myComponent.doWork();
          }
      }
      ```

  - **활성화:** 이러한 어노테이션을 스프링 컨테이너가 인식하고 처리하려면, XML 설정에 `<context:component-scan base-package="com.example"/>`를 추가하거나, Java 설정 클래스에 `@ComponentScan(basePackages = "com.example")`을 사용해야 합니다.
3. **자바 기반 설정 (Java-based configuration):**
  - **특징:** XML을 전혀 사용하지 않고, 자바 클래스와 어노테이션만으로 스프링 설정을 구성합니다. `@Configuration` 어노테이션이 붙은 클래스 내에서 `@Bean` 어노테이션이 붙은 메소드를 통해 빈을 정의하고 의존 관계를 설정합니다.
  - **코드 예시 (Java):**

      ```java
      package com.example.config;
      
      import com.example.MyRepositoryImpl;
      import com.example.MyService;
      import com.example.MyServiceImpl;
      import org.springframework.context.annotation.Bean;
      import org.springframework.context.annotation.Configuration;
      
      @Configuration // 이 클래스가 스프링 설정 정보를 담고 있음을 나타냄
      public class AppConfig {
      
          @Bean // 이 메소드가 반환하는 객체를 스프링 빈으로 등록
          public MyRepositoryImpl myRepository() {
              return new MyRepositoryImpl();
          }
      
          @Bean
          public MyService myService() {
              // myRepository() 메소드를 호출하여 의존성 주입
              return new MyServiceImpl(myRepository());
          }
      }
      ```

  - 최근 스프링 부트(Spring Boot) 환경에서는 이 자바 기반 설정이 기본이자 가장 권장되는 방식입니다.

**3. 의존성 주입(DI) 방법 및 생성자 주입의 장점**

- **대표적인 DI 방법 세 가지:**
  1. **생성자 주입 (Constructor Injection):** 생성자의 매개변수를 통해 의존성을 주입받습니다.
  2. **세터 주입 (Setter Injection):** 세터(setter) 메소드를 통해 의존성을 주입받습니다. 필드에 `@Autowired`를 붙이거나, 세터 메소드에 `@Autowired`를 붙여 사용합니다.

      ```java
      // 세터 주입 예시
      @Service
      public class MySetterService {
          private MyDependency myDependency;
      
          @Autowired
          public void setMyDependency(MyDependency myDependency) {
              this.myDependency = myDependency;
          }
          // ...
      }
      ```

  3. **필드 주입 (Field Injection):** 클래스의 필드에 직접 `@Autowired` 어노테이션을 붙여 의존성을 주입받습니다.
     (참고: 필드 주입은 코드가 간결해 보이지만, 테스트 용이성 감소, 순환 참조 가능성, 객체 불변성 확보 어려움 등의 단점이 있어 최근에는 생성자 주입이 더 권장됩니다.)

      ```java
      // 필드 주입 예시
      @Service
      public class MyFieldService {
          @Autowired
          private MyDependency myDependency;
          // ...
      }
      ```

- **생성자 주입의 장점 (스프링 공식 문서에서도 권장하는 방식):**
  1. **의존성 불변성(Immutability) 확보:** `final` 키워드를 사용하여 필드를 선언하고 생성자에서 초기화하면, 해당 의존성이 객체 생성 이후 변경되지 않음을 보장할 수 있습니다. 이는 객체의 상태를 예측 가능하게 만들고 스레드 안전성을 높이는 데 도움이 됩니다.

      ```java
      @Service
      public class MyConstructorService {
          private final MyDependency myDependency; // final로 선언
      
          @Autowired // 생성자가 하나만 있다면 @Autowired 생략 가능 (스프링 4.3부터)
          public MyConstructorService(MyDependency myDependency) {
              this.myDependency = myDependency;
          }
      }
      ```

  2. **필수 의존성 명확화:** 생성자의 매개변수로 받는 의존성은 해당 객체가 생성될 때 **반드시 필요하다**는 것을 명확하게 나타냅니다. 만약 해당 의존성이 주입되지 않으면 객체 자체가 생성될 수 없으므로, 런타임에 `NullPointerException`이 발생하는 것을 컴파일 시점(또는 애플리케이션 시작 시점)에 더 가깝게 방지할 수 있습니다.
  3. **순환 참조 방지 (애플리케이션 시작 시 감지):** 두 빈이 서로 생성자 주입을 통해 순환적으로 의존하는 경우, 스프링 컨테이너는 빈을 생성하는 과정에서 이를 감지하고 `BeanCurrentlyInCreationException`과 같은 예외를 발생시켜 애플리케이션 시작을 실패시킵니다. 이를 통해 문제를 더 빨리 발견하고 수정할 수 있습니다. (세터 주입이나 필드 주입은 순환 참조 발생 시 애플리케이션은 일단 시작될 수 있지만, 실제 해당 빈을 사용하려 할 때 문제가 발생할 수 있습니다.)
  4. **테스트 용이성 향상:** 의존성을 외부에서 주입받으므로, 단위 테스트 시 실제 구현체 대신 모의(mock) 객체를 생성자에 쉽게 전달하여 테스트할 수 있습니다. 의존성 주입 프레임워크(스프링) 없이도 객체를 생성하고 테스트하기 용이합니다.

      ```java
      // 테스트 코드 예시
      MyDependency mockDependency = mock(MyDependency.class); // Mockito 사용
      MyConstructorService service = new MyConstructorService(mockDependency);
      // service.doSomething(); 테스트 진행
      ```


---

**필드 주입의 단점:**

1. **테스트 용이성 저하:**
  - **이유:** 필드 주입을 사용하면, 해당 클래스를 단위 테스트할 때 의존성을 외부에서 직접 주입하기가 어렵습니다. 테스트 코드에서 MyFieldInjectionService 객체를 new로 생성하면 myDependency 필드는 null 상태가 됩니다. 이를 해결하려면 리플렉션(Reflection)을 사용하여 강제로 값을 주입하거나, 스프링 테스트 컨텍스트를 로드하여 스프링 컨테이너가 주입해주도록 해야 하는데, 이는 순수한 단위 테스트의 목적을 벗어나고 테스트를 복잡하게 만듭니다.
  - **생성자 주입의 경우:** 테스트 코드에서 new MyConstructorService(mockDependency)와 같이 모의(mock) 객체를 생성자에 쉽게 전달하여 테스트할 수 있습니다. 의존성 주입 프레임워크 없이도 객체 생성이 가능합니다.
  - **세터 주입의 경우:** 테스트 코드에서 service.setMyDependency(mockDependency)와 같이 세터 메소드를 호출하여 모의 객체를 주입할 수 있습니다.
2. **순환 참조 가능성 (런타임 오류 발생 가능성):**
  - **이유:** 두 클래스 A와 B가 서로 필드 주입을 통해 의존하는 경우 (A가 B를, B가 A를 필드 주입), 스프링 컨테이너는 빈을 생성하는 과정에서 이 순환 참조를 감지하지 못하고 일단 객체들을 생성할 수 있습니다. (정확히는, 생성자 주입처럼 빈 생성 자체를 막지는 못하는 경우가 많습니다. 스프링은 똑똑해서 일부 순환 참조는 해결하기도 하지만, 복잡한 경우 문제가 됩니다.)
  - 문제는 실제 해당 빈들의 메소드가 호출되어 서로를 참조하려고 할 때, 아직 완전히 초기화되지 않은 프록시 객체 등을 참조하게 되어 StackOverflowError나 예기치 않은 동작이 발생할 수 있다는 것입니다. 즉, 애플리케이션 실행 중에 문제가 터질 수 있습니다.
  - **생성자 주입의 경우:** 순환 참조가 발생하면 스프링 컨테이너가 빈을 생성하는 시점에 BeanCurrentlyInCreationException을 발생시켜 애플리케이션 시작을 실패시킵니다. 이를 통해 개발 초기에 문제를 발견하고 수정할 수 있습니다.
3. **객체 불변성(Immutability) 확보 어려움:**
  - **이유:** 필드 주입을 사용하면 해당 필드를 final로 선언할 수 없습니다. final 필드는 반드시 생성자에서 초기화되어야 하기 때문입니다. final로 선언되지 않은 필드는 객체 생성 이후에도 (이론적으로는) 변경될 가능성이 있습니다.
  - 객체 불변성은 프로그램의 상태를 예측 가능하게 만들고, 동시성 환경에서 스레드 안전성을 높이는 데 중요한 요소입니다.
  - **생성자 주입의 경우:** final 키워드를 사용하여 의존성을 불변으로 만들 수 있습니다.
4. **DI 컨테이너에 대한 강한 의존성:**
  - **이유:** 필드 주입 방식은 해당 클래스가 스프링과 같은 DI 컨테이너 환경 밖에서는 정상적으로 동작하기 어렵게 만듭니다. 의존성을 주입할 방법이 없기 때문입니다.
  - 생성자 주입이나 세터 주입은 DI 컨테이너 없이도 개발자가 직접 의존성을 전달하여 객체를 생성하고 사용할 수 있는 여지를 남겨둡니다 (POJO 친화적).
5. **단일 책임 원칙(SRP) 위반 가능성 증가:**
  - 필드 주입은 의존성을 추가하기가 너무 쉽기 때문에, 개발자가 자신도 모르게 하나의 클래스에 너무 많은 의존성을 추가하게 될 가능성이 있습니다. 이는 단일 책임 원칙을 위반하고 클래스를 거대하게 만들 수 있습니다.
  - 생성자 주입의 경우, 생성자 매개변수가 너무 많아지면 "이 클래스가 너무 많은 책임을 지고 있는 것은 아닌가?"라는 경고 신호로 작용할 수 있습니다.

**결론적으로,**

- **생성자 주입은 불변성, 필수 의존성 명시, 순환 참조 조기 발견, 테스트 용이성 등의 장점**으로 인해 현재 가장 권장되는 의존성 주입 방식입니다.
- **세터 주입은 선택적 의존성**에 사용할 수 있지만, 생성자 주입으로 대부분의 상황을 해결할 수 있습니다. (@RequiredArgsConstructor 같은 롬복 어노테이션을 사용하면 생성자 코드도 간결하게 작성 가능)
- **필드 주입은 코드는 가장 간결해 보이지만, 위에 언급된 여러 단점** 때문에 특별한 경우가 아니라면 피하는 것이 좋습니다.
