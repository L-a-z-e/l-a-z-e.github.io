---
title: Spring Core - Review(2)
description: 
author: laze
date: 2025-06-23 00:00:01 +0900
categories: [Dev, SpringBoot]
tags: [SpringBoot]
---
### **빈 스코프 (Bean Scopes) 피드백 및 상세 설명**

**1. 스프링의 기본 빈 스코프와 그 특징**

- 스프링의 기본 빈 스코프는 **싱글톤(Singleton)**입니다.
- **싱글톤 스코프의 특징:**
  - **단 하나의 인스턴스:** 스프링 IoC 컨테이너 내에서 해당 빈 정의에 대해 **오직 하나의 인스턴스만 생성**됩니다.
  - **공유:** 컨테이너는 이 단일 인스턴스를 캐싱해두고, 해당 빈을 필요로 하는 모든 곳(다른 빈이나 애플리케이션 코드)에 **동일한 인스턴스를 공유하여 제공**합니다.
  - **생명주기:** 컨테이너가 시작될 때 생성되어(기본적으로 즉시 로딩), 컨테이너가 종료될 때까지 유지됩니다.

**2. 기본 스코프 외 다른 주요 빈 스코프와 사용 사례**

- **피드백 및 추가 설명:**
  - **프로토타입(Prototype) 스코프**
    - **특징:** 빈을 요청할 때마다 (주입받거나 `getBean()`으로 조회할 때마다) **매번 새로운 인스턴스가 생성**됩니다.
    - **사용 사례:**
      - 상태를 가지는(stateful) 빈을 사용해야 할 때. 각 사용자나 각 요청마다 독립적인 상태를 유지해야 하는 경우에 적합합니다.
      - 매번 새로운 객체가 필요한 경우 (예: 특정 작업 수행 후 폐기되는 객체).
    - **주의점:** 프로토타입 빈은 스프링 컨테이너가 생성 및 의존성 주입까지만 관여하고, 그 이후의 생명주기(예: 소멸 콜백 호출)는 관리하지 않습니다. 개발자가 직접 관리해야 합니다.
  - **웹 환경에서 사용되는 주요 스코프 (추가적으로 알아두면 좋은 것들):**
    - **`request` 스코프:**
      - **특징:** 각 HTTP 요청마다 새로운 빈 인스턴스가 생성됩니다. 동일한 HTTP 요청 내에서는 같은 인스턴스가 공유되지만, 다른 요청에서는 다른 인스턴스가 사용됩니다.
      - **사용 사례:** HTTP 요청과 관련된 상태 정보를 저장하고 싶을 때 (예: 현재 요청의 사용자 정보, 요청별 캐시 데이터).
    - **`session` 스코프:**
      - **특징:** 각 HTTP 세션마다 새로운 빈 인스턴스가 생성됩니다. 동일한 사용자의 세션 동안에는 같은 인스턴스가 공유됩니다.
      - **사용 사례:** 사용자의 로그인 정보, 장바구니 정보 등 세션 동안 유지되어야 하는 상태 정보를 저장할 때.
    - **`application` 스코프 (또는 `servletContext` 스코프):**
      - **특징:** 웹 애플리케이션 전체에 대해 단 하나의 빈 인스턴스만 생성됩니다 (싱글톤과 유사하지만 웹 애플리케이션 컨텍스트 범위). `ServletContext` 생명주기와 같습니다.
      - **사용 사례:** 웹 애플리케이션 전반에 걸쳐 공유되어야 하는 설정 정보나 공용 데이터.
    - `websocket` 스코프 (웹소켓 사용 시) 등도 있습니다.

**3. 빈 스코프 지정 어노테이션 및 코드 예시**

- 어노테이션 기반 설정에서 빈의 스코프를 지정할 때는 **`@Scope` 어노테이션**을 사용합니다.
- **코드 예시:**

  **싱글톤 스코프 (기본값이므로 `@Scope("singleton")`은 보통 생략):**

    ```java
    import org.springframework.stereotype.Component;
    // import org.springframework.context.annotation.Scope; // 필요시 명시적 선언
    
    @Component
    // @Scope("singleton") // 기본값이므로 생략 가능
    public class MySingletonBean {
        public MySingletonBean() {
            System.out.println("MySingletonBean instance created: " + this);
        }
        // ...
    }
    ```

  **프로토타입 스코프:**

    ```java
    import org.springframework.context.annotation.Scope;
    import org.springframework.stereotype.Component;
    
    @Component
    @Scope("prototype") // 스코프를 "prototype"으로 지정
    public class MyPrototypeBean {
        public MyPrototypeBean() {
            System.out.println("MyPrototypeBean instance created: " + this);
        }
        // ...
    }
    ```

  **Request 스코프 (웹 환경에서만 유효):**

    ```java
    import org.springframework.context.annotation.Scope;
    import org.springframework.stereotype.Component;
    import org.springframework.web.context.annotation.RequestScope; // 더 명시적인 어노테이션
    
    @Component
    // @Scope("request") // 일반적인 방법
    @RequestScope // 스프링 4.3부터 제공되는 더 명시적인 어노테이션
    public class MyRequestScopedBean {
        public MyRequestScopedBean() {
            System.out.println("MyRequestScopedBean instance created: " + this);
        }
        // ...
    }
    ```

  (마찬가지로 `@SessionScope`, `@ApplicationScope`도 존재합니다.)

  **사용 예시 (프로토타입 빈 주입):**

    ```java
    import org.springframework.beans.factory.annotation.Autowired;
    import org.springframework.stereotype.Service;
    
    @Service
    public class BeanUserService {
        @Autowired
        private MySingletonBean singletonBean1;
        @Autowired
        private MySingletonBean singletonBean2;
    
        @Autowired
        private MyPrototypeBean prototypeBean1;
        @Autowired
        private MyPrototypeBean prototypeBean2;
    
        public void printBeans() {
            System.out.println("SingletonBean1: " + singletonBean1);
            System.out.println("SingletonBean2: " + singletonBean2); // singletonBean1과 동일한 인스턴스
            System.out.println("PrototypeBean1: " + prototypeBean1);
            System.out.println("PrototypeBean2: " + prototypeBean2); // prototypeBean1과 다른 새 인스턴스
        }
    }
    ```

  위 `BeanUserService`를 실행해보면, `singletonBean1`과 `singletonBean2`는 같은 객체 주소를 출력하지만, `prototypeBean1`과 `prototypeBean2`는 서로 다른 객체 주소를 출력하는 것을 확인할 수 있습니다. 이것이 바로 스코프의 차이입니다.


---

### **빈 생명주기 콜백 및 `BeanPostProcessor` 피드백 및 상세 설명**

**1. 빈 생명주기 콜백(Lifecycle Callback) 방법 두 가지**

1. **어노테이션 사용 (`@PostConstruct`, `@PreDestroy`):**
  - **`@PostConstruct`:** 의존성 주입이 완료된 후, 빈이 실제로 사용되기 전에 **초기화 작업**을 수행하기 위해 호출될 메소드에 붙입니다.
  - **`@PreDestroy`:** 빈이 컨테이너에서 제거되기 직전에 **자원 정리 작업** 등을 수행하기 위해 호출될 메소드에 붙입니다. (주로 싱글톤 빈이 컨테이너 종료 시 호출됨)
  - **장점:** 코드가 간결하고, 특정 인터페이스에 대한 의존성이 없어 POJO 친화적입니다. **가장 권장되는 방식**입니다.
  - **코드 예시:**

      ```java
      import jakarta.annotation.PostConstruct; // 또는 javax.annotation.PostConstruct (Java EE 시절)
      import jakarta.annotation.PreDestroy;  // 또는 javax.annotation.PreDestroy
      import org.springframework.stereotype.Component;
      
      @Component
      public class MyServiceWithAnnotations {
      
          public MyServiceWithAnnotations() {
              System.out.println("MyServiceWithAnnotations: Constructor called");
          }
      
          // 의존성 주입이 완료된 후 실행될 초기화 메소드
          @PostConstruct
          public void initialize() {
              System.out.println("MyServiceWithAnnotations: @PostConstruct initialize method called.");
              // 예: 리소스 로딩, 기본값 설정, 외부 시스템 연결 등
          }
      
          public void doWork() {
              System.out.println("MyServiceWithAnnotations: Doing work...");
          }
      
          // 빈이 소멸되기 직전에 실행될 정리 메소드
          @PreDestroy
          public void cleanup() {
              System.out.println("MyServiceWithAnnotations: @PreDestroy cleanup method called.");
              // 예: 리소스 해제, 연결 종료 등
          }
      }
      ```

2. **인터페이스 구현 (`InitializingBean`, `DisposableBean`):**
  - **`InitializingBean`:** `afterPropertiesSet()`이라는 단일 메소드를 가집니다. 이 메소드는 빈의 모든 프로퍼티(의존성 포함)가 설정된 후 호출됩니다. `@PostConstruct`와 유사한 시점입니다.
  - **`DisposableBean`:** `destroy()`라는 단일 메소드를 가집니다. 이 메소드는 빈이 소멸될 때 호출됩니다. `@PreDestroy`와 유사한 시점입니다.
  - **단점:** 스프링 프레임워크 인터페이스에 대한 의존성이 생깁니다. 코드가 스프링에 종속적으로 됩니다.
  - **코드 예시:**

      ```java
      import org.springframework.beans.factory.DisposableBean;
      import org.springframework.beans.factory.InitializingBean;
      import org.springframework.stereotype.Component;
      
      @Component
      public class MyServiceWithInterfaces implements InitializingBean, DisposableBean {
      
          public MyServiceWithInterfaces() {
              System.out.println("MyServiceWithInterfaces: Constructor called");
          }
      
          // InitializingBean 인터페이스의 메소드
          @Override
          public void afterPropertiesSet() throws Exception {
              System.out.println("MyServiceWithInterfaces: afterPropertiesSet (InitializingBean) method called.");
              // 초기화 로직
          }
      
          public void doWork() {
              System.out.println("MyServiceWithInterfaces: Doing work...");
          }
      
          // DisposableBean 인터페이스의 메소드
          @Override
          public void destroy() throws Exception {
              System.out.println("MyServiceWithInterfaces: destroy (DisposableBean) method called.");
              // 정리 로직
          }
      }
      ```


**실행 순서 (한 빈에 여러 방식이 정의된 경우):**
만약 한 빈에 여러 초기화/소멸 방식이 정의되어 있다면, 일반적으로 다음과 같은 순서로 실행됩니다 (하지만 특정 방식만 사용하는 것이 좋습니다):

- **초기화:** `@PostConstruct` -> `InitializingBean.afterPropertiesSet()` -> (XML 설정의 `init-method` 속성)
- **소멸:** `@PreDestroy` -> `DisposableBean.destroy()` -> (XML 설정의 `destroy-method` 속성)

**2. `BeanPostProcessor` 인터페이스**

- 빈의 **초기화(initialization) 과정 전후에 개입**하여 빈 자체를 변경하거나 추가적인 작업을 수행하는 역할을 합니다.
- **`BeanPostProcessor`의 역할:**
  - 스프링 IoC 컨테이너가 **빈(Bean) 인스턴스를 생성하고 의존성 주입을 완료한 후, 실제 초기화 콜백(`@PostConstruct` 또는 `InitializingBean.afterPropertiesSet()`)이 호출되기 *전*과 *후*에** 추가적인 처리를 할 수 있는 "가로채기(interception)" 또는 "후킹(hooking)" 메커니즘을 제공합니다.
  - 컨테이너에 등록된 **모든 빈**에 대해 이 과정이 적용됩니다 (물론, 조건에 따라 특정 빈에만 적용하도록 만들 수도 있습니다).
- **개입 시점:**
  1. 빈 인스턴스 생성
  2. 의존성 주입 (예: `@Autowired` 처리)
  3. `BeanPostProcessor`의 **`postProcessBeforeInitialization(Object bean, String beanName)`** 메소드 호출
  4. 빈의 초기화 콜백 실행 (예: `@PostConstruct` 메소드, `InitializingBean.afterPropertiesSet()`)
  5. `BeanPostProcessor`의 **`postProcessAfterInitialization(Object bean, String beanName)`** 메소드 호출
  6. 빈 사용 준비 완료
- **`BeanPostProcessor`를 사용한 부가 기능 예시:**
  - **프록시(Proxy) 객체 생성:** 특정 빈을 프록시 객체로 감싸서 AOP(관점 지향 프로그래밍) 기능을 적용할 수 있습니다. 예를 들어, 특정 어노테이션이 붙은 빈을 찾아 트랜잭션 처리 프록시나 로깅 프록시로 대체할 수 있습니다. (스프링의 많은 내부 기능들이 `BeanPostProcessor`를 통해 구현됩니다. 예를 들어 `@Autowired` 어노테이션 처리, `@Async` 처리 등)
  - **빈 프로퍼티 수정:** 초기화 전후에 빈의 프로퍼티 값을 변경할 수 있습니다.
  - **커스텀 어노테이션 처리:** 개발자가 직접 만든 커스텀 어노테이션을 빈에서 찾아서 특정 로직을 수행하도록 할 수 있습니다.
  - **빈 래핑(Wrapping):** 원본 빈을 다른 객체로 감싸서 반환할 수 있습니다.
- **코드 예시 (`BeanPostProcessor`):**

    ```java
    import org.springframework.beans.BeansException;
    import org.springframework.beans.factory.config.BeanPostProcessor;
    import org.springframework.stereotype.Component;
    
    // MyServiceWithAnnotations 빈이 있다고 가정 (위 예제 사용)
    
    @Component // 스프링 빈으로 등록되어야 자동으로 동작
    public class MyCustomBeanPostProcessor implements BeanPostProcessor {
    
        @Override
        public Object postProcessBeforeInitialization(Object bean, String beanName) throws BeansException {
            if ("myServiceWithAnnotations".equals(beanName)) { // 특정 빈에만 적용
                System.out.println("MyCustomBeanPostProcessor: Before Initialization of '" + beanName + "'. Bean: " + bean);
            }
            return bean; // 항상 원본 빈 또는 수정된 빈을 반환해야 함
        }
    
        @Override
        public Object postProcessAfterInitialization(Object bean, String beanName) throws BeansException {
            if ("myServiceWithAnnotations".equals(beanName)) {
                System.out.println("MyCustomBeanPostProcessor: After Initialization of '" + beanName + "'. Bean: " + bean);
                // 예시: 여기서 MyServiceWithAnnotations 타입이라면 특정 메소드를 호출하거나,
                // 프록시로 감싸서 반환할 수도 있음.
                // if (bean instanceof MyServiceWithAnnotations) {
                //     // ((MyServiceWithAnnotations) bean).doWork(); // 예시일 뿐, 실제로는 이렇게 직접 호출 잘 안함
                // }
            }
            return bean;
        }
    }
    ```

  **위 코드를 실행하면 콘솔 출력 순서 (예상):**

  1. `MyServiceWithAnnotations: Constructor called` (빈 인스턴스 생성)
  2. (의존성 주입 완료 - 해당 예제에는 없지만)
  3. `MyCustomBeanPostProcessor: Before Initialization of 'myServiceWithAnnotations'. Bean: ...` (`postProcessBeforeInitialization` 호출)
  4. `MyServiceWithAnnotations: @PostConstruct initialize method called.` (초기화 콜백 실행)
  5. `MyCustomBeanPostProcessor: After Initialization of 'myServiceWithAnnotations'. Bean: ...` (`postProcessAfterInitialization` 호출)
  6. (이후 `MyServiceWithAnnotations` 빈 사용 가능)
  7. (컨테이너 종료 시) `MyServiceWithAnnotations: @PreDestroy cleanup method called.` (소멸 콜백 실행)

---

### **어노테이션 기반 설정 및 컴포넌트 스캔 피드백 및 상세 설명**

**1. 스프링 빈 자동 등록 어노테이션 및 스테레오타입 어노테이션**

- **`@Component`:** 스프링에게 해당 클래스가 **관리해야 할 컴포넌트(빈)**임을 나타내는 가장 일반적이고 기본적인 어노테이션입니다. 컴포넌트 스캔의 대상이 됩니다.
- **스테레오타입(Stereotype) 어노테이션:** `@Component`의 특별한 형태로, 특정 역할이나 계층을 나타내어 코드의 가독성을 높이고, 경우에 따라 추가적인 기능(예: `@Repository`의 예외 변환)을 제공하기도 합니다.
  - **`@Service`:** **서비스 계층(Service Layer)**의 컴포넌트에 사용됩니다. 비즈니스 로직을 담당하는 클래스에 주로 붙입니다.
  - **`@Repository`:** **데이터 접근 계층(Data Access Layer, Persistence Layer)**의 컴포넌트에 사용됩니다. 데이터베이스 연동과 관련된 예외를 스프링의 `DataAccessException`으로 변환해주는 기능도 포함합니다.
  - **`@Controller` (또는 `@RestController`):** **프레젠테이션 계층(Presentation Layer)**의 컴포넌트에 사용됩니다. 웹 요청을 처리하는 컨트롤러 클래스에 붙입니다. (이전 Web MVC 챕터에서 학습)
- **`@Configuration`:** 이 어노테이션은 해당 클래스가 **스프링의 설정 정보(주로 `@Bean` 메소드를 포함)를 담고 있는 클래스**임을 나타냅니다. `@Configuration` 클래스도 내부적으로 `@Component`를 포함하고 있어 컴포넌트 스캔의 대상이 됩니다.

**2. 어노테이션 기반 빈 자동 등록 메커니즘 및 활성화 어노테이션**

- **메커니즘:** **클래스패스 스캐닝(Classpath Scanning)**. 스프링 컨테이너는 지정된 기본 패키지(base package)부터 시작하여 그 하위 패키지들을 탐색하면서 `@Component` (및 그 스테레오타입 어노테이션들)가 붙은 클래스들을 찾아 자동으로 빈으로 등록합니다.
- **활성화 어노테이션 (Java 기반 설정 시):** **`@ComponentScan`** 어노테이션을 `@Configuration` 클래스에 사용하여 컴포넌트 스캔을 활성화하고 탐색할 기본 패키지를 지정합니다.

    ```java
    import org.springframework.context.annotation.ComponentScan;
    import org.springframework.context.annotation.Configuration;
    
    @Configuration
    @ComponentScan(basePackages = "com.example.myapp") // "com.example.myapp" 패키지부터 하위까지 스캔
    // 또는 @ComponentScan(basePackageClasses = MyAppMainClass.class) // 특정 클래스가 위치한 패키지 기준
    public class AppConfig {
        // 여기에 다른 @Bean 정의 등이 올 수 있음
    }
    ```

- **XML 기반 설정 시:** `<context:component-scan base-package="com.example.myapp"/>` 태그를 사용합니다.

**3. `@Autowired`의 의존성 주입 방식 및 모호성 해결**

- 가장 먼저 **타입(Type)을 기준**으로 주입할 빈을 찾습니다. 예를 들어, `MyService` 타입의 필드에 `@Autowired`를 붙이면, 스프링 컨테이너에서 `MyService` 타입으로 등록된 빈을 찾습니다.
- 만약 해당 타입의 빈이 **하나만 존재**하면 그 빈을 주입합니다.
- 만약 해당 타입의 빈이 **여러 개 존재**하면, 그때 **필드 이름(또는 파라미터 이름)과 빈의 이름(id)**이 일치하는 빈을 찾으려고 시도합니다.
- **모호성 해결 방법:**
  - **`@Primary`:**
    - 여러 개의 동일 타입 빈 중에서 **우선적으로 주입될 빈**을 지정할 때 사용합니다. `@Component`나 `@Bean` 어노테이션과 함께 사용하여 해당 빈이 기본 선택지가 되도록 합니다.

      ```java
      interface MessageService {}
      
      @Component
      @Primary // 이 빈이 MessageService 타입의 기본 주입 대상이 됨
      class EmailService implements MessageService {}
      
      @Component
      class SmsService implements MessageService {}
      
      // 사용하는 곳
      @Service
      class NotificationService {
          @Autowired
          private MessageService messageService; // EmailService가 주입됨
      }
      ```

  - **`@Qualifier("빈이름")`:**
    - 여러 개의 동일 타입 빈 중에서 **특정 이름을 가진 빈을 명시적으로 지정**하여 주입받고 싶을 때 사용합니다. `@Autowired` 어노테이션과 함께 사용됩니다.

      ```java
      interface MessageService {}
      
      @Component("emailMessageService") // 빈의 이름을 "emailMessageService"로 지정
      class EmailService implements MessageService {}
      
      @Component("smsMessageService") // 빈의 이름을 "smsMessageService"로 지정
      class SmsService implements MessageService {}
      
      // 사용하는 곳
      @Service
      class NotificationService {
          @Autowired
          @Qualifier("smsMessageService") // "smsMessageService"라는 이름의 빈을 주입
          private MessageService messageService; // SmsService가 주입됨
      }
      ```


### **`@Inject` 어노테이션 (JSR-330)**

- **역할:** 스프링의 **`@Autowired`와 거의 동일한 역할**을 합니다. 의존성을 주입받기 위해 필드, 생성자, 또는 메소드에 사용할 수 있습니다.
- **빈 탐색 방식:**
  - 기본적으로 **타입(Type)을 기준**으로 주입할 빈을 찾습니다.
  - 만약 해당 타입의 빈이 여러 개 존재하여 모호성이 발생하면, 스프링의 `@Autowired`처럼 필드 이름으로 매칭을 시도하지는 않습니다. 이때는 `@Named` 어노테이션을 함께 사용해야 합니다.
- **`required` 속성 부재:** 스프링의 `@Autowired(required = false)`와 같이 주입 대상이 없어도 오류를 발생시키지 않도록 하는 `required` 속성이 `@Inject` 자체에는 없습니다. 대신, 주입받는 타입으로 `java.util.Optional<MyType>`을 사용하거나, `javax.inject.Provider<MyType>`를 사용하여 지연(lazy) 주입 및 선택적 주입을 구현할 수 있습니다. (스프링 환경에서는 `@Autowired`의 `required` 속성이 더 직관적일 수 있습니다.)
- **사용 예시:**

    ```java
    import javax.inject.Inject; // 또는 jakarta.inject.Inject (Jakarta EE 9+)
    import org.springframework.stereotype.Service;
    
    // MyDependency 빈이 이미 등록되어 있다고 가정
    // @Component
    // class MyDependency { /* ... */ }
    
    @Service
    public class MyServiceWithInject {
    
        private final MyDependency myDependency;
    
        // 생성자 주입
        @Inject // @Autowired와 유사하게 동작
        public MyServiceWithInject(MyDependency myDependency) {
            this.myDependency = myDependency;
        }
    
        public void doWork() {
            myDependency.performAction();
        }
    }
    ```

    ```java
    @Service
    public class AnotherServiceWithInject {
    
        @Inject // 필드 주입
        private MyDependency myDependency;
    
        // ...
    }
    ```


---

### **`@Named` 어노테이션 (JSR-330)**

- **역할:** 스프링의 **`@Qualifier`와 매우 유사한 역할**을 합니다. 특정 이름을 가진 빈을 지정하여 주입받고 싶을 때 사용합니다. 또한, `@Component`처럼 클래스를 빈으로 등록하는 역할도 할 수 있습니다 (이름을 지정하여).
- **`@Inject`와 함께 사용 (의존성 주입 시):**
  - 주입하려는 타입의 빈이 여러 개 있을 때, `@Named("빈이름")`을 `@Inject`와 함께 사용하여 특정 빈을 지정합니다.
- **클래스 레벨에서 사용 (빈 정의 시):**
  - 클래스에 `@Named("빈이름")`을 붙이면, 해당 클래스가 "빈이름"이라는 이름을 가진 빈으로 등록됩니다. 이는 스프링의 `@Component("빈이름")`과 유사합니다. 이름을 지정하지 않으면 기본 이름 규칙을 따릅니다.
- **사용 예시:**

  **빈 정의 시:**

    ```java
    import javax.inject.Named; // 또는 jakarta.inject.Named
    
    @Named("emailService") // "emailService"라는 이름의 빈으로 등록
    public class EmailServiceImpl implements MessageService {
        // ...
    }
    
    @Named("smsService") // "smsService"라는 이름의 빈으로 등록
    public class SmsServiceImpl implements MessageService {
        // ...
    }
    ```

  **의존성 주입 시:**

    ```java
    import javax.inject.Inject;
    import javax.inject.Named;
    import org.springframework.stereotype.Service;
    
    @Service
    public class NotificationServiceWithNamed {
    
        private final MessageService messageService;
    
        @Inject
        public NotificationServiceWithNamed(@Named("smsService") MessageService messageService) {
            // "smsService"라는 이름을 가진 MessageService 구현체를 주입받음
            this.messageService = messageService;
        }
    
        public void sendNotification(String message) {
            messageService.sendMessage(message);
        }
    }
    ```


---

**`@Autowired`/`@Qualifier` vs. `@Inject`/`@Named` 요약**

| 스프링 고유 어노테이션 | JSR-330 표준 어노테이션 | 주요 역할 및 차이점 |
| --- | --- | --- |
| `@Autowired` | `@Inject` | 의존성 주입. `@Autowired`는 `required` 속성이 있고, 타입 매칭 실패 시 필드명으로 추가 매칭 시도. `@Inject`는 표준이며, `required` 속성 대신 `Optional`이나 `Provider` 사용. |
| `@Qualifier("이름")` | `@Named("이름")` | 특정 이름의 빈을 지정하여 주입. `@Named`는 클래스 레벨에서 빈 이름 정의에도 사용 가능 (`@Component("이름")`과 유사). |

**왜 JSR-330 표준 어노테이션을 사용할까요?**

- **프레임워크 독립성 향상 (이론상):** JSR-330은 자바 표준이므로, 이론적으로는 코드가 특정 DI 프레임워크(스프링)에 대한 의존성을 줄일 수 있습니다. 다른 JSR-330 호환 DI 컨테이너로 코드를 이전할 때 더 용이할 수 있습니다.
- **표준 준수:** 프로젝트나 팀에서 자바 표준을 따르는 것을 선호하는 경우 선택할 수 있습니다.

**하지만 현실적으로 스프링 프로젝트에서는...**

- 대부분의 스프링 프로젝트에서는 스프링 고유의 `@Autowired`, `@Qualifier`, `@Component` 등을 사용하는 것이 더 일반적입니다. 왜냐하면 스프링 프레임워크의 다른 기능들과 더 긴밀하게 통합되어 있고, 스프링 개발자들에게 더 익숙하며, 스프링이 제공하는 추가적인 기능(예: `@Autowired`의 `required` 속성)을 활용할 수 있기 때문입니다.
- 스프링 부트를 사용하면 의존성 관리가 매우 잘 되어 있어, JSR-330 어노테이션을 사용하기 위해 별도의 라이브러리(`javax.inject:javax.inject:1` 또는 `jakarta.inject:jakarta.inject-api`)를 추가해야 할 수도 있습니다.
