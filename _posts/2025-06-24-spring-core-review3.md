---
title: Spring Core - Review(3)
description: 
author: laze
date: 2025-06-24 00:00:01 +0900
categories: [Dev, SpringBoot]
tags: [SpringBoot]
---
스프링은 XML 설정 파일 대신 자바 클래스를 사용하여 빈(Bean)을 정의하고 의존성을 설정하는 방법을 제공합니다.

1. 자바 기반 설정에서 특정 클래스가 **스프링 설정 정보를 담고 있음을 나타내는 주요 어노테이션**은 무엇이며, 이 어노테이션이 붙은 클래스 내에서 **개별 빈(Bean)을 정의하는 메소드에 사용하는 어노테이션**은 무엇인가요?
2. 자바 기반 설정 클래스 내에서 @Bean 어노테이션이 붙은 메소드를 통해 빈을 정의할 때, **해당 빈의 이름은 기본적으로 어떻게 결정**되며, 만약 **개발자가 직접 빈의 이름을 지정하고 싶다면 @Bean 어노테이션에 어떤 속성을 사용**할 수 있나요? 간단한 코드 예시도 보여주시면 좋겠습니다.
3. @Configuration 클래스 내에서 다른 @Bean 메소드를 호출하여 의존성을 주입하는 것은 일반적인 자바 메소드 호출과 어떤 차이점이 있을까요? (힌트: 스프링 컨테이너의 역할, 프록시) 그리고 이 방식이 보장하는 중요한 특징은 무엇일까요?

**1. 스프링 설정 정보 클래스 및 빈 정의 메소드 어노테이션**

- **`@Configuration`:** 이 어노테이션이 붙은 클래스는 스프링 컨테이너에게 **"이 클래스는 빈 정의와 의존성 설정을 포함하는 설정 정보를 제공합니다"** 라고 알려주는 역할을 합니다. `@Configuration` 클래스 자체도 `@Component`를 포함하므로 스프링 빈으로 관리됩니다.
- **`@Bean`:** `@Configuration` 클래스 내의 **메소드에 사용**되며, 해당 메소드가 반환하는 객체를 스프링 컨테이너가 관리하는 **빈(Bean)으로 등록**하라는 의미입니다.

**2. `@Bean` 메소드를 통해 정의된 빈의 이름 결정 방식 및 커스터마이징**

- **기본 이름 결정:** `@Bean` 어노테이션이 붙은 **메소드의 이름**이 기본적으로 해당 빈의 이름(ID)으로 사용됩니다.
- **이름 커스터마이징:** `@Bean` 어노테이션의 `name` 속성 (또는 `value` 속성, `name`이 별칭임)을 사용하여 빈의 이름을 명시적으로 지정할 수 있습니다. 여러 이름을 지정할 수도 있습니다 (`name = {"customName", "anotherName"}`).
- **코드 예시:**

    ```java
    import org.springframework.context.annotation.Bean;
    import org.springframework.context.annotation.Configuration;
    
    // 예시 서비스 인터페이스 및 구현체
    interface MyDataService {}
    class DefaultDataService implements MyDataService {}
    class SpecialDataService implements MyDataService {}
    
    @Configuration
    public class AppConfigForBeanName {
    
        @Bean // 빈 이름은 "defaultDataService" (메소드 이름)
        public MyDataService defaultDataService() {
            return new DefaultDataService();
        }
    
        @Bean(name = "customDataService") // 빈 이름을 "customDataService"로 명시적으로 지정
        public MyDataService specialDataService() {
            return new SpecialDataService();
        }
    
        @Bean({"anotherService", "serviceAlias"}) // 여러 이름 지정 가능
        public AnotherService anotherService() {
            return new AnotherService();
        }
    }
    class AnotherService {}
    ```


**3. `@Configuration` 클래스 내 `@Bean` 메소드 호출의 특별함 (CGLIB 프록시와 싱글톤 보장)**

- **일반 자바 클래스에서의 메소드 호출:**
  만약 `@Configuration`이 아닌 일반 자바 클래스에서 다른 메소드를 여러 번 호출하면, 그 메소드는 호출될 때마다 실행되어 **매번 새로운 객체를 반환**할 수 있습니다 (메소드 구현에 따라).
- **`@Configuration` 클래스에서의 `@Bean` 메소드 호출 (핵심 차이):**
  - 스프링 컨테이너는 `@Configuration` 어노테이션이 붙은 클래스를 특별하게 처리합니다. 이 클래스는 **CGLIB 라이브러리를 사용하여 프록시(proxy) 객체로 감싸져서 빈으로 등록**됩니다. (프록시는 원본 객체 대신 호출을 가로채서 부가 기능을 수행하거나 동작을 변경하는 대리 객체입니다.)
  - 이 프록시 객체 덕분에, `@Configuration` 클래스 내에서 다른 `@Bean` 메소드를 호출하여 의존성을 주입하려고 할 때, 해당 `@Bean` 메소드가 단순히 여러 번 실행되어 매번 새로운 객체를 만드는 것이 아닙니다.
  - 대신, 프록시화된 `@Bean` 메소드는 **스프링 컨테이너에게 이미 해당 빈이 생성되어 있는지 확인**합니다.
    - **싱글톤 스코프 (기본값):** 만약 해당 빈이 이미 컨테이너에 싱글톤으로 존재한다면, 프록시는 메소드를 실제로 실행하지 않고 **컨테이너에 있는 기존의 싱글톤 빈 인스턴스를 반환**합니다.
    - 만약 아직 생성되지 않았다면, 그때 메소드를 실행하여 빈을 생성하고 컨테이너에 등록한 후 그 빈을 반환합니다.
  - **중요한 특징 (이것이 보장하는 것):** 이 메커니즘 덕분에 `@Configuration` 클래스 내에서 `@Bean` 메소드를 여러 번 호출하더라도, 해당 빈이 싱글톤 스코프라면 **항상 동일한 인스턴스가 반환되는 것을 보장**합니다. 이는 빈들 간의 의존 관계를 설정할 때 싱글톤 객체의 일관성을 유지하는 데 매우 중요합니다.
- **코드 예시:**

    ```java
    import org.springframework.context.annotation.Bean;
    import org.springframework.context.annotation.Configuration;
    
    class DataSourceBean {
        public DataSourceBean() { System.out.println("DataSourceBean created: " + this); }
    }
    
    class RepositoryBean {
        private DataSourceBean dataSource;
        public RepositoryBean(DataSourceBean dataSource) {
            this.dataSource = dataSource;
            System.out.println("RepositoryBean created with DataSource: " + dataSource);
        }
    }
    
    class ServiceBean {
        private DataSourceBean dataSource;
        public ServiceBean(DataSourceBean dataSource) {
            this.dataSource = dataSource;
            System.out.println("ServiceBean created with DataSource: " + dataSource);
        }
    }
    
    @Configuration
    public class AppConfigWithProxy {
    
        @Bean // dataSourceBean 메소드는 한 번만 실행되어 싱글톤 객체 생성
        public DataSourceBean dataSourceBean() {
            return new DataSourceBean();
        }
    
        @Bean
        public RepositoryBean repositoryBean() {
            // dataSourceBean()을 호출하지만, 프록시가 개입하여
            // 컨테이너의 싱글톤 dataSourceBean 인스턴스를 반환
            return new RepositoryBean(dataSourceBean());
        }
    
        @Bean
        public ServiceBean serviceBean() {
            // 여기서도 dataSourceBean()을 호출하지만,
            // repositoryBean에 주입된 것과 동일한 싱글톤 인스턴스를 반환
            return new ServiceBean(dataSourceBean());
        }
    }
    
    // 실행 예시
    // ApplicationContext context = new AnnotationConfigApplicationContext(AppConfigWithProxy.class);
    // RepositoryBean repo = context.getBean(RepositoryBean.class);
    // ServiceBean service = context.getBean(ServiceBean.class);
    // System.out.println("Is DataSource same in repo and service? " + (repo.dataSource == service.dataSource)); // true 출력 예상
    ```

  위 코드에서 `AppConfigWithProxy`가 `@Configuration`으로 선언되었기 때문에, `repositoryBean()`과 `serviceBean()` 메소드 내에서 `dataSourceBean()`을 호출해도 `DataSourceBean`은 딱 한 번만 생성되고, `repositoryBean`과 `serviceBean`은 동일한 `DataSourceBean` 인스턴스를 주입받게 됩니다. 콘솔에는 "DataSourceBean created" 메시지가 한 번만 찍힐 것입니다.

  만약 `AppConfigWithProxy`에서 `@Configuration` 어노테이션을 제거하고 (또는 `@Configuration(proxyBeanMethods = false)`로 설정하고) 실행하면, `dataSourceBean()` 메소드는 일반 자바 메소드처럼 호출될 때마다 새로운 `DataSourceBean` 인스턴스를 생성하게 되어, "DataSourceBean created" 메시지가 여러 번 찍히고 `repositoryBean`과 `serviceBean`은 서로 다른 `DataSourceBean` 인스턴스를 가지게 됩니다.

- **`proxyBeanMethods` 속성:** `@Configuration(proxyBeanMethods = false)`로 설정하면, CGLIB 프록시를 사용하지 않고 `@Configuration` 클래스를 일반 빈처럼 취급합니다. 이 경우 `@Bean` 메소드 간 호출은 일반 메소드 호출처럼 동작하여 싱글톤 보장이 깨질 수 있습니다.

### **환경 추상화 및 프로파일 (`@Profile`, `Environment`) 피드백 및 상세 설명**

**1. 스프링에서 프로파일(Profile)이란 무엇이며, 어떤 목적으로 사용될까요?**

- **프로파일(Profile)이란?** 애플리케이션의 설정을 여러 환경(예: 개발-`dev`, 테스트-`test`, 운영-`prod`, 로컬-`local`)에 따라 **그룹화**하고, 특정 환경에서는 특정 그룹의 설정만 활성화하도록 하는 메커니즘입니다.
- **주된 목적:**
  - **환경별 설정 분리:** 데이터베이스 연결 정보, 외부 API 키, 로깅 레벨, 특정 기능 활성화 여부 등 환경에 따라 달라져야 하는 설정들을 분리하여 관리할 수 있게 합니다.
  - **배포 용이성:** 동일한 애플리케이션 코드를 빌드한 후, 실행 시 활성화할 프로파일만 지정함으로써 각 환경에 맞는 설정으로 쉽게 배포할 수 있습니다. (코드 변경 없이 설정만으로 환경 대응)
  - **테스트 용이성:** 테스트 환경에서는 실제 DB 대신 인메모리 DB를 사용하거나, 모의(mock) 서비스를 사용하도록 프로파일을 통해 설정할 수 있습니다.
- **`application-{profile}.yml` (또는 `.properties`) 파일 로딩:** 스프링 부트(Spring Boot) 환경에서는 이처럼 프로파일별 프로퍼티 파일을 자동으로 로드하는 편리한 기능을 제공합니다. `spring.profiles.active` 프로퍼티를 통해 활성화할 프로파일을 지정하면 해당 파일의 설정이 우선적으로 적용됩니다. (스프링 프레임워크 자체에서도 유사한 방식으로 프로퍼티 파일을 프로파일별로 관리할 수 있습니다.)

**2. 특정 프로파일에서만 설정/빈을 활성화하는 어노테이션 및 코드 예시**

- 특정 프로파일이 활성화되었을 때만 해당 설정 클래스, `@Bean` 메소드, 또는 `@Component` 계열 어노테이션이 붙은 클래스를 활성화하는 데 사용되는 주요 어노테이션은 바로 **`@Profile`** 입니다.
- **`@Profile` 어노테이션 사용법:**
  - `@Profile("프로파일이름")`: 지정된 프로파일이 활성화될 때만 해당 요소가 활성화됩니다.
  - `@Profile({"프로파일1", "프로파일2"})`: 여러 프로파일 중 하나라도 활성화되면 해당 요소가 활성화됩니다 (OR 조건).
  - `@Profile("!프로파일이름")`: 지정된 프로파일이 활성화되지 *않았을 때만* 해당 요소가 활성화됩니다 (NOT 조건).
  - `@Profile("프로파일1 & 프로파일2")`: 프로파일1과 프로파일2가 *모두* 활성화될 때만 (AND 조건, 스프링 5.1부터 지원).
  - `@Profile("프로파일1 | 프로파일2")`: 프로파일1 *또는* 프로파일2가 활성화될 때 (OR 조건, 스프링 5.1부터 지원).
- **코드 예시 ("dev" 프로파일에서만 활성화되는 `@Bean` 메소드):**
  위 예제에서 만약 애플리케이션 실행 시 활성 프로파일을 "dev"로 지정하면 (`Dspring.profiles.active=dev` 또는 `application.properties`에 `spring.profiles.active=dev` 설정), `devDataSource()` 메소드만 호출되어 `DevDataSourceConfig` 빈이 생성됩니다.

    ```java
    import org.springframework.context.annotation.Bean;
    import org.springframework.context.annotation.Configuration;
    import org.springframework.context.annotation.Profile;
    
    // 예시 데이터 소스 인터페이스 및 구현체
    interface DataSourceConfig { String getUrl(); }
    
    class DevDataSourceConfig implements DataSourceConfig {
        @Override public String getUrl() { return "jdbc:h2:mem:devdb"; }
    }
    
    class ProdDataSourceConfig implements DataSourceConfig {
        @Override public String getUrl() { return "jdbc:mysql://prod-server/maindb"; }
    }
    
    @Configuration
    public class DataSourceConfiguration {
    
        @Bean
        @Profile("dev") // "dev" 프로파일이 활성화될 때만 이 빈이 생성됨
        public DataSourceConfig devDataSource() {
            System.out.println("Creating DEV DataSource bean");
            return new DevDataSourceConfig();
        }
    
        @Bean
        @Profile("prod") // "prod" 프로파일이 활성화될 때만 이 빈이 생성됨
        public DataSourceConfig prodDataSource() {
            System.out.println("Creating PROD DataSource bean");
            return new ProdDataSourceConfig();
        }
    
        @Bean
        @Profile("!prod & !dev") // "prod"도 "dev"도 아닐 때 (예: "default" 프로파일)
        public DataSourceConfig defaultDataSource() {
            System.out.println("Creating DEFAULT DataSource bean (In-memory H2)");
            return new DevDataSourceConfig(); // 기본으로 개발용 H2 사용
        }
    }
    
    ```


**3. 활성 프로파일 및 프로퍼티 값 접근 인터페이스와 코드 예시**

- **피드백 및 추가 설명:**
  - 스프링 애플리케이션의 현재 환경 정보(활성 프로파일, 프로퍼티 값 등)에 접근하기 위한 핵심 인터페이스는 바로 **`org.springframework.core.env.Environment`** 입니다.
  - `ApplicationContext`는 `EnvironmentCapable` 인터페이스를 구현하므로, `ApplicationContext`를 통해 `Environment` 객체를 얻을 수 있습니다. 또는, 빈에 `@Autowired`를 사용하여 `Environment` 객체를 직접 주입받을 수도 있습니다.
  - **`Environment` 인터페이스의 주요 기능:**
    - **활성 프로파일 확인:** `String[] getActiveProfiles()` (활성 프로파일 목록 반환), `boolean acceptsProfiles(Profiles profiles)` (특정 프로파일 표현식이 현재 활성 프로파일과 맞는지 확인)
    - **프로퍼티 값 접근:** `String getProperty(String key)`, `String getProperty(String key, String defaultValue)`, `T getProperty(String key, Class<T> targetType)` 등 다양한 메소드로 프로퍼티 값을 읽어올 수 있습니다. 이 프로퍼티는 시스템 프로퍼티, 환경 변수, `application.properties`/`yml` 파일 등 다양한 소스에서 올 수 있습니다.
  - **코드 예시 (`Environment`를 사용하여 프로퍼티 값 읽기):**

    **`application.properties` (또는 `application-dev.properties` 등):**

      ```
      myapp.service.url=http://dev-service.example.com
      myapp.service.timeout=5000
      ```

    **`Environment`를 주입받아 사용하는 서비스 클래스:**

      ```java
      import org.springframework.beans.factory.annotation.Autowired;
      import org.springframework.core.env.Environment;
      import org.springframework.stereotype.Service;
      import jakarta.annotation.PostConstruct;
      
      @Service
      public class MyConfigurableService {
      
          @Autowired
          private Environment environment; // Environment 객체 주입
      
          private String serviceUrl;
          private int serviceTimeout;
      
          @PostConstruct // 빈 초기화 시 프로퍼티 값 로드
          public void init() {
              this.serviceUrl = environment.getProperty("myapp.service.url");
              // getProperty는 String을 반환하므로, int로 변환 필요. 기본값 제공도 가능.
              this.serviceTimeout = environment.getProperty("myapp.service.timeout", Integer.class, 10000); // 기본값 10000ms
      
              System.out.println("Service URL from properties: " + serviceUrl);
              System.out.println("Service Timeout from properties: " + serviceTimeout);
      
              // 활성 프로파일 확인 예시
              if (environment.acceptsProfiles(org.springframework.core.env.Profiles.of("dev"))) {
                  System.out.println("Currently running with DEV profile.");
              } else if (environment.acceptsProfiles(org.springframework.core.env.Profiles.of("prod"))) {
                  System.out.println("Currently running with PROD profile.");
              }
          }
      
          public void callExternalService() {
              System.out.println("Calling external service at: " + serviceUrl + " with timeout: " + serviceTimeout);
              // ... 실제 서비스 호출 로직 ...
          }
      }
      ```

    위 `MyConfigurableService`는 `Environment` 객체를 주입받아, `init()` 메소드에서 `application.properties` (또는 활성 프로파일에 맞는 프로퍼티 파일)에 정의된 `myapp.service.url`과 `myapp.service.timeout` 값을 읽어와 필드에 저장합니다.

    > 참고: 프로퍼티 값을 직접 필드에 주입받는 더 간편한 방법으로 @Value("${property.key}") 어노테이션도 있습니다. Environment는 좀 더 포괄적인 환경 정보 접근 및 프로그래밍 방식의 조건 처리에 유용합니다.
>

---
