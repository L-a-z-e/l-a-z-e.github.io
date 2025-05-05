---
title: Declaring an Aspect
description: 
author: laze
date: 2025-05-05 00:00:13 +0900
categories: [Dev, SpringBoot]
tags: [SpringBoot]
---
**Declaring an Aspect**

`@AspectJ` 지원이 활성화되면, `@AspectJ` 애스펙트인 클래스(`@Aspect` 어노테이션을 가짐)를 가진 애플리케이션 컨텍스트에 정의된 모든 빈은 스프링에 의해 자동으로 감지되고 스프링 AOP를 구성하는 데 사용됩니다.

다음 두 예제는 그다지 유용하지 않은 애스펙트에 필요한 최소 단계를 보여줍니다.

두 예제 중 첫 번째는 `@Aspect`로 어노테이션된 빈 클래스를 가리키는 애플리케이션 컨텍스트의 일반 빈 정의를 보여줍니다:

**Java**

```java
// ApplicationConfiguration.java
import org.springframework.context.annotation.Bean;
import org.springframework.context.annotation.Configuration;
// Assuming NotVeryUsefulAspect exists in the same package or imported

@Configuration
public class ApplicationConfiguration {

	@Bean
	public NotVeryUsefulAspect myAspect() { // Define the aspect as a bean
		NotVeryUsefulAspect myAspect = new NotVeryUsefulAspect();
		// 여기서 애스펙트의 속성 구성
		return myAspect;
	}
}
```

**Kotlin**

```kotlin
// ApplicationConfiguration.kt
import org.springframework.context.annotation.Bean
import org.springframework.context.annotation.Configuration
// Assuming NotVeryUsefulAspect exists in the same package or imported

@Configuration
class ApplicationConfiguration {

    @Bean
    fun myAspect(): NotVeryUsefulAspect { // Define the aspect as a bean
        val myAspect = NotVeryUsefulAspect()
        // 여기서 애스펙트의 속성 구성
        return myAspect
    }
}
```

**Xml**

```xml
<beans>
    <bean id="myAspect" class="org.xyz.NotVeryUsefulAspect">
        <!-- 여기서 애스펙트의 속성 구성 -->
    </bean>
</beans>
```

두 예제 중 두 번째는 `@Aspect`로 어노테이션된 `NotVeryUsefulAspect` 클래스 정의를 보여줍니다:

**Java**

```java
// NotVeryUsefulAspect.java
import org.aspectj.lang.annotation.Aspect;

@Aspect // Mark this class as an Aspect
public class NotVeryUsefulAspect {
    // No advice, pointcuts, etc. defined yet
}
```

**Kotlin**

```kotlin
// NotVeryUsefulAspect.kt
import org.aspectj.lang.annotation.Aspect

@Aspect // Mark this class as an Aspect
class NotVeryUsefulAspect {
    // No advice, pointcuts, etc. defined yet
}
```

애스펙트들(`@Aspect`로 어노테이션된 클래스들)은 다른 클래스와 마찬가지로 메소드와 필드를 가질 수 있습니다.

또한 포인트컷, 어드바이스 및 인트로덕션(타입 간) 선언을 포함할 수 있습니다.

*컴포넌트 스캐닝을 통한 애스펙트 자동 감지 (Autodetecting aspects through component scanning)*
다른 스프링 관리 빈과 마찬가지로, 스프링 XML 구성에서 일반 빈으로, `@Configuration` 클래스의 `@Bean` 메소드를 통해, 또는 클래스패스 스캐닝을 통해 스프링이 자동 감지하도록 애스펙트 클래스를 등록할 수 있습니다.

그러나 `@Aspect` 어노테이션만으로는 클래스패스에서의 자동 감지에 충분하지 않다는 점에 유의하십시오.

그 목적을 위해서는 별도의 `@Component` 어노테이션(또는 스프링의 컴포넌트 스캐너 규칙에 따라 자격이 되는 커스텀 스테레오타입 어노테이션)을 추가해야 합니다.

*다른 애스펙트로 애스펙트 어드바이스하기? (Advising aspects with other aspects?)*
스프링 AOP에서 애스펙트 자체는 다른 애스펙트의 어드바이스 대상이 될 수 없습니다.

클래스의 `@Aspect` 어노테이션은 그것을 애스펙트로 표시하고, 따라서 자동 프록시에서 제외합니다.
