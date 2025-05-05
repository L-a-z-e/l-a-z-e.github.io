---
title: AspectJ
description: 
author: laze
date: 2025-05-05 00:00:12 +0900
categories: [Dev, SpringBoot]
tags: [SpringBoot]
---
**Enabling @AspectJ Support**

스프링 구성에서 `@AspectJ` 애스펙트(aspects)를 사용하려면,

`@AspectJ` 애스펙트를 기반으로 스프링 AOP를 구성하고 해당 애스펙트에 의해 어드바이스(advised)되는지 여부에 따라 빈을 자동 프록시(auto-proxying)하는 스프링 지원을 활성화해야 합니다.

자동 프록시란, 스프링이 어떤 빈이 하나 이상의 애스펙트에 의해 어드바이스된다고 판단하면, 메소드 호출을 가로채고 필요에 따라 어드바이스(advice)가 실행되도록 해당 빈에 대한 프록시를 자동으로 생성한다는 것을 의미합니다.

`@AspectJ` 지원은 프로그래밍 방식 또는 XML 구성으로 활성화할 수 있습니다.

어떤 경우든, AspectJ의 `org.aspectj:aspectjweaver` 라이브러리가 애플리케이션의 클래스패스(버전 1.9 이상)에 있는지 확인해야 합니다.

**Java**

```java
import org.springframework.context.annotation.Configuration;
import org.springframework.context.annotation.EnableAspectJAutoProxy;

@Configuration
@EnableAspectJAutoProxy // Enables @AspectJ support
public class ApplicationConfiguration {
    // ... other bean definitions
}
```

**Kotlin**

```kotlin
import org.springframework.context.annotation.Configuration
import org.springframework.context.annotation.EnableAspectJAutoProxy

@Configuration
@EnableAspectJAutoProxy // Enables @AspectJ support
class ApplicationConfiguration {
    // ... other bean definitions
}
```

**Xml**

```xml
<beans xmlns="<http://www.springframework.org/schema/beans>"
    xmlns:xsi="<http://www.w3.org/2001/XMLSchema-instance>"
    xmlns:aop="<http://www.springframework.org/schema/aop>"
    xsi:schemaLocation="
        <http://www.springframework.org/schema/beans> <https://www.springframework.org/schema/beans/spring-beans.xsd>
        <http://www.springframework.org/schema/aop> <https://www.springframework.org/schema/aop/spring-aop.xsd>">

    <!-- Enables @AspectJ support -->
    <aop:aspectj-autoproxy/>

    <!-- ... other bean definitions -->

</beans>
```
