---
title: Spring Framework Using CustomAutowireConfigurer
description: 
author: laze
date: 2025-05-03 00:00:04 +0900
categories: [Dev, SpringBoot]
tags: [SpringBoot]
---
**Using CustomAutowireConfigurer**

`CustomAutowireConfigurer`는 `BeanFactoryPostProcessor`로서, 스프링의 `@Qualifier` 어노테이션으로 어노테이션되지 않은 경우에도 자신만의 커스텀 퀄리파이어(qualifier) 어노테이션 타입을 등록할 수 있게 해줍니다.

```xml
<bean id="customAutowireConfigurer"
		class="org.springframework.beans.factory.annotation.CustomAutowireConfigurer">
	<property name="customQualifierTypes">
		<set>
			<value>example.CustomQualifier</value>
		</set>
	</property>
</bean>
```

`AutowireCandidateResolver`는 다음을 통해 자동 와이어링 후보를 결정합니다:

- 각 빈 정의의 `autowire-candidate` 값
- `<beans/>` 요소에서 사용 가능한 모든 `default-autowire-candidates` 패턴
- `@Qualifier` 어노테이션 및 `CustomAutowireConfigurer`에 등록된 모든 커스텀 어노테이션의 존재 여부

여러 빈이 자동 와이어링 후보에 해당될 때, "기본(primary)" 결정은 다음과 같습니다: 만약 후보들 중 정확히 하나의 빈 정의에 `primary` 속성이 `true`로 설정되어 있다면, 해당 빈이 선택됩니다.

---

**전체 주제: `CustomAutowireConfigurer` 사용하기**

이 부분은 스프링의 `@Qualifier` 어노테이션을 **직접 사용하지 않고**, 개발자가 만든 **임의의 어노테이션**을 마치 퀄리파이어처럼 인식시켜 자동 와이어링 후보를 좁히는 데 사용할 수 있도록 **등록**하는 방법에 대한 내용입니다.

**핵심 아이디어:** 우리가 만든 `@MyCustomQualifier` 같은 어노테이션도 스프링의 `@Qualifier`처럼 동작하게 만들자! (단, 어노테이션 정의에 `@Qualifier`를 붙이지 않고)

---

**1. 배경 및 동기:**

- 앞서 배운 방법은 커스텀 퀄리파이어 어노테이션(`@Genre` 등)을 정의할 때, 그 정의 위에 **반드시 스프링의 `@Qualifier` 메타 어노테이션을 붙여야** 스프링이 그것을 퀄리파이어로 인식했습니다.
- 하지만 만약, 어떤 이유로든 커스텀 어노테이션 정의 자체를 수정할 수 없거나(예: 외부 라이브러리 어노테이션), 또는 `@Qualifier` 메타 어노테이션을 붙이고 싶지 않은 경우가 있을 수 있습니다.
- 이럴 때 `CustomAutowireConfigurer`를 사용하여 **"이 어노테이션(`example.CustomQualifier`)도 퀄리파이어로 취급해줘!"** 라고 스프링 컨테이너에 **명시적으로 알려줄 수 있습니다.**

---

**2. `CustomAutowireConfigurer` 사용법:**

- **`CustomAutowireConfigurer` 빈 등록:** 이 클래스는 `BeanFactoryPostProcessor`의 구현체입니다. 따라서 스프링 설정에 **빈으로 등록**해주면 됩니다.
- **`customQualifierTypes` 속성 설정:** 이 속성에 **퀄리파이어로 사용하고 싶은 커스텀 어노테이션들의 클래스 이름(정규화된 이름)을 Set 형태로 지정**합니다.

    ```xml
    <bean id="customAutowireConfigurer"
          class="org.springframework.beans.factory.annotation.CustomAutowireConfigurer">
        <property name="customQualifierTypes">
            <set>
                <!-- "example.CustomQualifier" 라는 어노테이션을 퀄리파이어로 등록 -->
                <value>example.CustomQualifier</value>
                <!-- 필요하다면 다른 커스텀 어노테이션도 추가 가능 -->
                <!-- <value>another.CustomAnnotation</value> -->
            </set>
        </property>
    </bean>
    
    ```

- **동작 방식:**
  1. 스프링 컨테이너가 시작되면서 `CustomAutowireConfigurer` 빈을 (다른 빈들보다 먼저) 생성하고 `BeanFactoryPostProcessor`로 인식합니다.
  2. `CustomAutowireConfigurer`는 `customQualifierTypes`에 등록된 어노테이션 타입들을 기억해둡니다.
  3. 나중에 `@Autowired` 의존성 주입 시 후보 빈을 결정할 때, 스프링의 자동 와이어링 메커니즘(`AutowireCandidateResolver`)은 `@Qualifier` 어노테이션뿐만 아니라, **`CustomAutowireConfigurer`에 등록된 커스텀 어노테이션들(`example.CustomQualifier` 등)이 주입 지점이나 빈 정의에 사용되었는지도 확인**합니다.
  4. 만약 등록된 커스텀 어노테이션이 발견되면, 이를 `@Qualifier`와 유사하게 취급하여 자동 와이어링 후보를 좁히는 데 사용합니다.

---

**3. 자동 와이어링 후보 결정 프로세스 요약 (`AutowireCandidateResolver`):**

스프링이 `@Autowired` 대상 빈을 최종 결정할 때 고려하는 요소들은 다음과 같습니다:

1. **타입 매칭:** 먼저 주입 지점의 타입과 일치하는 모든 빈을 찾습니다.
2. **`autowire-candidate` 속성 확인 (XML):** 빈 정의에 `autowire-candidate="false"`가 설정되어 있다면 해당 빈은 후보에서 제외됩니다.
3. **`default-autowire-candidates` 패턴 확인 (XML):** `<beans>` 요소에 설정된 패턴과 일치하지 않는 빈은 제외될 수 있습니다.
4. **퀄리파이어 매칭:**
  - 주입 지점에 `@Qualifier` 어노테이션이나 **`CustomAutowireConfigurer`에 등록된 커스텀 어노테이션**이 있는지 확인합니다.
  - 만약 있다면, 후보 빈들 중에서 해당 퀄리파이어 정보를 가진 빈들만 남깁니다. (XML에서는 `<qualifier>` 태그 또는 빈 이름 fallback)
5. **`@Primary` 확인:** 남은 후보들 중에 `@Primary`가 붙은 빈이 **정확히 하나** 있다면, 그 빈을 최종 선택합니다.
6. **`@Fallback` 확인:** 남은 후보들 중 `@Fallback`이 붙은 빈들을 제외하고 **정확히 하나**의 일반 빈이 남는다면, 그 빈을 최종 선택합니다.
7. **이름 매칭 (Fallback):** 위 단계들로도 후보가 여러 개 남는다면, 주입 지점의 이름(필드명/파라미터명)과 빈의 이름(ID)이 일치하는 후보가 있는지 확인하고 있다면 선택합니다.
8. **최종 결정:** 위 과정을 거쳐 후보가 정확히 하나로 결정되면 주입하고, 그렇지 않으면 오류(NoUniqueBeanDefinitionException 또는 NoSuchBeanDefinitionException)를 발생시킵니다.

---

**결론:**

`CustomAutowireConfigurer`는 스프링의 `@Qualifier` 메타 어노테이션을 사용하지 않고도, 개발자가 정의한 **임의의 어노테이션을 퀄리파이어처럼 등록**하여 자동 와이어링 후보를 선택하는 데 사용할 수 있게 해주는 `BeanFactoryPostProcessor`입니다. 이는 커스텀 어노테이션 정의를 수정할 수 없거나 특정 디자인 제약이 있을 때 유용할 수 있지만, 일반적으로는 커스텀 퀄리파이어를 만들 때 `@Qualifier` 메타 어노테이션을 붙이는 것이 더 직접적이고 권장되는 방법입니다.
