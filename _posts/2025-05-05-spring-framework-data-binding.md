---
title: Spring Framework Data Binding
description: 
author: laze
date: 2025-05-05 00:00:03 +0900
categories: [Dev, SpringBoot]
tags: [SpringBoot]
---
**Data Binding**

데이터 바인딩은 사용자 입력을 대상 객체에 바인딩하는 데 유용하며, 여기서 사용자 입력은 JavaBeans 관례를 따르는 속성 경로를 키로 가지는 맵입니다.

`DataBinder`는 이를 지원하는 주요 클래스이며, 사용자 입력을 바인딩하는 두 가지 방법을 제공합니다:

1. **생성자 바인딩 (Constructor binding)** - 사용자 입력을 public 데이터 생성자에 바인딩하며, 생성자 인수 값을 사용자 입력에서 조회합니다.
2. **속성 바인딩 (Property binding)** - 사용자 입력을 세터(setter)에 바인딩하며, 사용자 입력의 키를 대상 객체 구조의 속성과 일치시킵니다.

생성자 바인딩과 속성 바인딩 모두를 적용하거나 하나만 적용할 수 있습니다.

*생성자 바인딩 (Constructor Binding)*
생성자 바인딩을 사용하려면:

1. 대상 객체로 `null`을 사용하여 `DataBinder`를 생성합니다.
2. `targetType`을 대상 클래스로 설정합니다.
3. `construct`를 호출합니다.

대상 클래스는 단일 public 생성자 또는 인수가 있는 단일 비-public 생성자를 가져야 합니다. 여러 생성자가 있는 경우, 기본 생성자(있는 경우)가 사용됩니다.

기본적으로 인수 값은 생성자 파라미터 이름을 통해 조회됩니다. 스프링 MVC 및 WebFlux는 생성자 파라미터 또는 필드(있는 경우)의 `@BindParam` 어노테이션을 통해 커스텀 이름 매핑을 지원합니다.

필요한 경우, 사용할 인수 이름을 사용자 정의하기 위해 `DataBinder`에 `NameResolver`를 구성할 수도 있습니다.

사용자 입력을 변환하기 위해 필요에 따라 타입 변환이 적용됩니다.

생성자 파라미터가 객체인 경우, 중첩된 속성 경로를 통해 동일한 방식으로 재귀적으로 생성됩니다.

이는 즉, 생성자 바인딩이 대상 객체와 그것이 포함하는 모든 객체 모두를 생성한다는 것을 의미합니다.

생성자 바인딩은 단일 문자열(예: 쉼표로 구분된 목록)에서 변환되거나 `accounts[2].name` 또는 `account[KEY].name`과 같은 인덱싱된 키를 기반으로 하는 `List`, `Map` 및 배열 인수를 지원합니다.

바인딩 및 변환 오류는 `DataBinder`의 `BindingResult`에 반영됩니다.

대상이 성공적으로 생성되면, `construct` 호출 후 `target`은 생성된 인스턴스로 설정됩니다.

**BeanWrapper를 사용한 속성 바인딩 (Property Binding with BeanWrapper)**

`org.springframework.beans` 패키지는 JavaBeans 표준을 준수합니다.

JavaBean은 기본 인수 없는 생성자를 가지고 있으며 (예를 들어) `bingoMadness`라는 이름의 속성이 `setBingoMadness(..)` 세터 메소드와 `getBingoMadness()` 게터 메소드를 가지는 명명 규칙을 따르는 클래스입니다.

`beans` 패키지에서 매우 중요한 클래스 중 하나는 `BeanWrapper` 인터페이스와 해당 구현체(`BeanWrapperImpl`)입니다.

`BeanWrapper`는 속성 값 설정 및 가져오기(개별적 또는 일괄적), 속성 기술자(property descriptors) 가져오기, 속성이 읽기 가능 또는 쓰기 가능한지 확인하기 위한 쿼리 기능을 제공합니다.

또한 `BeanWrapper`는 중첩된 속성을 지원하여 무한 깊이까지 하위 속성의 속성 설정을 가능하게 합니다.

`BeanWrapper`는 또한 대상 클래스에 지원 코드 없이 표준 JavaBeans `PropertyChangeListener` 및 `VetoableChangeListener`를 추가하는 기능을 지원합니다.

마지막으로, `BeanWrapper`는 인덱싱된 속성 설정을 지원합니다.

`BeanWrapper`는 일반적으로 애플리케이션 코드에서 직접 사용되지 않고 `DataBinder` 및 `BeanFactory`에서 사용됩니다.

`BeanWrapper`가 작동하는 방식은 이름에 부분적으로 나타납니다: 빈을 래핑(wrap)하여 속성 설정 및 검색과 같은 작업을 해당 빈에서 수행합니다.

*기본 및 중첩 속성 설정 및 가져오기 (Setting and Getting Basic and Nested Properties)*
속성 설정 및 가져오기는 `BeanWrapper`의 오버로드된 `setPropertyValue` 및 `getPropertyValue` 메소드 변형을 통해 수행됩니다.

| 표현식 | 설명 |
| --- | --- |
| `name` | `getName()` 또는 `isName()` 및 `setName(..)` 메소드에 해당하는 속성 `name`을 나타냅니다. |
| `account.name` | (예를 들어) `getAccount().setName()` 또는 `getAccount().getName()` 메소드에 해당하는 속성 `account`의 중첩된 속성 `name`을 나타냅니다. |
| `accounts[2]` | 인덱싱된 속성 `account`의 세 번째 요소를 나타냅니다. 인덱싱된 속성은 배열, 리스트 또는 기타 자연 순서 컬렉션 타입일 수 있습니다. |
| `accounts[KEY]` | `KEY` 값으로 인덱싱된 맵 항목의 값을 나타냅니다. |

다음 두 예제 클래스는 속성을 가져오고 설정하기 위해 `BeanWrapper`를 사용합니다:

```java
// Java
public class Company {

	private String name;
	private Employee managingDirector;

	public String getName() { return this.name; }
	public void setName(String name) { this.name = name; }
	public Employee getManagingDirector() { return this.managingDirector; }
	public void setManagingDirector(Employee managingDirector) { this.managingDirector = managingDirector; }
}
```

```kotlin
// Kotlin
data class Company(var name: String? = null, var managingDirector: Employee? = null)
```

```java
// Java
public class Employee {

	private String name;
	private float salary;

	public String getName() { return this.name; }
	public void setName(String name) { this.name = name; }
	public float getSalary() { return salary; }
	public void setSalary(float salary) { this.salary = salary; }
}
```

```kotlin
// Kotlin
data class Employee(var name: String? = null, var salary: Float = 0.0f)
```

다음 코드 스니펫은 인스턴스화된 `Companys` 및 `Employees`의 일부 속성을 검색하고 조작하는 방법의 몇 가지 예를 보여줍니다:

```java
// Java
BeanWrapper company = new BeanWrapperImpl(new Company());
// 회사 이름 설정..
company.setPropertyValue("name", "Some Company Inc.");
// ... 다음과 같이 할 수도 있습니다:
PropertyValue value = new PropertyValue("name", "Some Company Inc.");
company.setPropertyValue(value);

BeanWrapper jim = new BeanWrapperImpl(new Employee());
jim.setPropertyValue("name", "Jim Stravinsky");
company.setPropertyValue("managingDirector", jim.getWrappedInstance());

// 회사를 통해 managingDirector의 급여 검색
Float salary = (Float) company.getPropertyValue("managingDirector.salary");
```

```kotlin
// Kotlin
val company = BeanWrapperImpl(Company())
// 회사 이름 설정..
company.setPropertyValue("name", "Some Company Inc.")
// ... 다음과 같이 할 수도 있습니다:
val value = PropertyValue("name", "Some Company Inc.")
company.setPropertyValue(value)

val jim = BeanWrapperImpl(Employee())
jim.setPropertyValue("name", "Jim Stravinsky")
company.setPropertyValue("managingDirector", jim.wrappedInstance) // Use wrappedInstance property

// 회사를 통해 managingDirector의 급여 검색
// Use safe cast and Elvis operator for null safety
val salary = company.getPropertyValue("managingDirector.salary") as? Float ?: 0.0f
```

**PropertyEditor들 (PropertyEditor's)**

스프링은 `Object`와 `String` 간의 변환을 수행하기 위해 `PropertyEditor` 개념을 사용합니다.

객체 자체와 다른 방식으로 속성을 표현하는 것이 편리할 수 있습니다. 예를 들어, `Date`는 사람이 읽을 수 있는 방식(문자열: '2007-14-09')으로 표현될 수 있지만,

사람이 읽을 수 있는 형식을 원래 날짜로 다시 변환할 수 있습니다 (또는 더 좋게는, 사람이 읽을 수 있는 형식으로 입력된 모든 날짜를 다시 `Date` 객체로 변환).

이 동작은 `java.beans.PropertyEditor` 타입의 커스텀 편집기(custom editors)를 등록하여 달성할 수 있습니다.

`BeanWrapper` 또는 (이전 장에서 언급했듯이) 특정 IoC 컨테이너에 커스텀 편집기를 등록하면 속성을 원하는 타입으로 변환하는 방법을 알게 됩니다.

`PropertyEditor`에 대한 자세한 내용은 Oracle의 `java.beans` 패키지 javadoc을 참조하십시오.

스프링에서 속성 편집이 사용되는 몇 가지 예:

- 빈에 대한 속성 설정은 `PropertyEditor` 구현을 사용하여 수행됩니다. XML 파일에서 선언하는 일부 빈의 속성 값으로 `String`을 사용할 때, 스프링은 (해당 속성의 세터가 `Class` 파라미터를 가지는 경우) `ClassEditor`를 사용하여 파라미터를 `Class` 객체로 해석하려고 시도합니다.
- 스프링의 MVC 프레임워크에서 HTTP 요청 파라미터 파싱은 `CommandController`의 모든 하위 클래스에서 수동으로 바인딩할 수 있는 모든 종류의 `PropertyEditor` 구현을 사용하여 수행됩니다.

스프링에는 삶을 편하게 해주는 여러 내장 `PropertyEditor` 구현이 있습니다.

모두 `org.springframework.beans.propertyeditors` 패키지에 위치합니다.

대부분(다음 표에 표시된 대로 전부는 아님)은 기본적으로 `BeanWrapperImpl`에 의해 등록됩니다. 속성 편집기가 어떤 방식으로든 구성 가능한 경우, 기본 편집기를 오버라이드하기 위해 자신만의 변형을 여전히 등록할 수 있습니다.

다음 표는 스프링이 제공하는 다양한 `PropertyEditor` 구현을 설명합니다:

표 2. 내장 `PropertyEditor` 구현

| 클래스 | 설명 |
| --- | --- |
| `ByteArrayPropertyEditor` | 바이트 배열용 편집기. 문자열을 해당 바이트 표현으로 변환합니다. 기본적으로 `BeanWrapperImpl`에 의해 등록됩니다. |
| `ClassEditor` | 클래스를 나타내는 문자열을 실제 클래스로 파싱하고 그 반대도 가능합니다. 클래스를 찾을 수 없는 경우 `IllegalArgumentException`이 발생합니다. 기본적으로 `BeanWrapperImpl`에 의해 등록됩니다. |
| `CustomBooleanEditor` | `Boolean` 속성에 대한 사용자 정의 가능한 속성 편집기. 기본적으로 `BeanWrapperImpl`에 의해 등록되지만, 커스텀 인스턴스를 커스텀 편집기로 등록하여 오버라이드할 수 있습니다. |
| `CustomCollectionEditor` | 컬렉션용 속성 편집기, 모든 소스 `Collection`을 주어진 대상 `Collection` 타입으로 변환합니다. |
| `CustomDateEditor` | `java.util.Date`에 대한 사용자 정의 가능한 속성 편집기, 커스텀 `DateFormat`을 지원합니다. 기본적으로 등록되지 않습니다. 필요에 따라 적절한 형식으로 사용자가 등록해야 합니다. |
| `CustomNumberEditor` | `Integer`, `Long`, `Float` 또는 `Double`과 같은 모든 `Number` 하위 클래스에 대한 사용자 정의 가능한 속성 편집기. 기본적으로 `BeanWrapperImpl`에 의해 등록되지만, 커스텀 인스턴스를 커스텀 편집기로 등록하여 오버라이드할 수 있습니다. |
| `FileEditor` | 문자열을 `java.io.File` 객체로 해석합니다. 기본적으로 `BeanWrapperImpl`에 의해 등록됩니다. |
| `InputStreamEditor` | 문자열을 받아 (중간 `ResourceEditor` 및 `Resource`를 통해) `InputStream`을 생성할 수 있는 단방향 속성 편집기이므로 `InputStream` 속성을 문자열로 직접 설정할 수 있습니다. 기본 사용법은 `InputStream`을 자동으로 닫지 않는다는 점에 유의하십시오. 기본적으로 `BeanWrapperImpl`에 의해 등록됩니다. |
| `LocaleEditor` | 문자열을 `Locale` 객체로 해석하고 그 반대도 가능합니다 (`Locale`의 `toString()` 메소드와 동일한 `[language]_[country]_[variant]` 문자열 형식). 밑줄의 대안으로 공백을 구분 기호로 허용합니다. 기본적으로 `BeanWrapperImpl`에 의해 등록됩니다. |
| `PatternEditor` | 문자열을 `java.util.regex.Pattern` 객체로 해석하고 그 반대도 가능합니다. |
| `PropertiesEditor` | (`java.util.Properties` 클래스의 javadoc에 정의된 형식으로 포맷된) 문자열을 `Properties` 객체로 변환할 수 있습니다. 기본적으로 `BeanWrapperImpl`에 의해 등록됩니다. |
| `StringTrimmerEditor` | 문자열을 트리밍(trim)하는 속성 편집기. 선택적으로 빈 문자열을 `null` 값으로 변환할 수 있습니다. 기본적으로 등록되지 않습니다 - 사용자가 등록해야 합니다. |
| `URLEditor` | URL의 문자열 표현을 실제 `URL` 객체로 해석할 수 있습니다. 기본적으로 `BeanWrapperImpl`에 의해 등록됩니다. |

스프링은 필요할 수 있는 속성 편집기에 대한 검색 경로를 설정하기 위해 `java.beans.PropertyEditorManager`를 사용합니다.

검색 경로에는 `Font`, `Color` 및 대부분의 기본 타입과 같은 타입에 대한 `PropertyEditor` 구현을 포함하는 `sun.bean.editors`도 포함됩니다.

또한 표준 JavaBeans 인프라는 처리하는 클래스와 동일한 패키지에 있고 해당 클래스와 이름이 같고 `Editor`가 추가된 `PropertyEditor` 클래스(명시적으로 등록할 필요 없이)를 자동으로 발견한다는 점에 유의하십시오.

예를 들어, 다음 클래스 및 패키지 구조가 있으면 `SomethingEditor` 클래스가 `Something` 타입 속성에 대한 `PropertyEditor`로 인식되고 사용되기에 충분합니다.

```
com
  chank
    pop
      Something
      SomethingEditor // Something 클래스에 대한 PropertyEditor
```

표준 `BeanInfo` JavaBeans 메커니즘도 여기서 사용할 수 있다는 점에 유의하십시오 (여기서 어느 정도 설명됨). 다음 예제는 `BeanInfo` 메커니즘을 사용하여 연관된 클래스의 속성에 하나 이상의 `PropertyEditor` 인스턴스를 명시적으로 등록합니다:

```
com
  chank
    pop
      Something
      SomethingBeanInfo // Something 클래스에 대한 BeanInfo
```

참조된 `SomethingBeanInfo` 클래스에 대한 다음 자바 소스 코드는 `Something` 클래스의 `age` 속성에 `CustomNumberEditor`를 연관시킵니다:

```java
// Java
import java.beans.*;

public class SomethingBeanInfo extends SimpleBeanInfo {

	@Override
	public PropertyDescriptor[] getPropertyDescriptors() {
		try {
			final PropertyEditor numberPE = new CustomNumberEditor(Integer.class, true);
			PropertyDescriptor ageDescriptor = new PropertyDescriptor("age", Something.class) {
				@Override
				public PropertyEditor createPropertyEditor(Object bean) {
					return numberPE; // Return the specific editor for the 'age' property
				}
			};
			return new PropertyDescriptor[] { ageDescriptor };
		}
		catch (IntrospectionException ex) {
			throw new Error(ex.toString());
		}
	}
}
```

```kotlin
// Kotlin
import java.beans.*

class SomethingBeanInfo : SimpleBeanInfo() {

    override fun getPropertyDescriptors(): Array<PropertyDescriptor>? {
        return try {
            val numberPE: PropertyEditor = CustomNumberEditor(Integer::class.java, true)
            val ageDescriptor = object : PropertyDescriptor("age", Something::class.java) { // Assuming Something exists
                override fun createPropertyEditor(bean: Any?): PropertyEditor {
                    return numberPE // Return the specific editor for the 'age' property
                }
            }
            arrayOf(ageDescriptor)
        } catch (ex: IntrospectionException) {
            throw Error(ex.toString())
        }
    }
}
```

**커스텀 PropertyEditor들 (Custom PropertyEditor's)**

빈 속성을 문자열 값으로 설정할 때, 스프링 IoC 컨테이너는 궁극적으로 표준 JavaBeans `PropertyEditor` 구현을 사용하여 이러한 문자열을 속성의 복잡한 타입으로 변환합니다.

스프링은 여러 커스텀 `PropertyEditor` 구현(예: 문자열로 표현된 클래스 이름을 `Class` 객체로 변환)을 사전 등록합니다.

또한 자바의 표준 JavaBeans `PropertyEditor` 룩업 메커니즘은 클래스에 대한 `PropertyEditor`가 적절하게 명명되고 지원을 제공하는 클래스와 동일한 패키지에 배치되어 자동으로 찾을 수 있도록 합니다.

다른 커스텀 `PropertyEditors`를 등록해야 하는 경우 여러 메커니즘을 사용할 수 있습니다.

가장 수동적인 접근 방식(일반적으로 편리하거나 권장되지 않음)은 `BeanFactory` 참조가 있다고 가정하고 `ConfigurableBeanFactory` 인터페이스의 `registerCustomEditor()` 메소드를 사용하는 것입니다.

또 다른 (약간 더 편리한) 메커니즘은 `CustomEditorConfigurer`라는 특수 빈 팩토리 후처리기를 사용하는 것입니다.

`BeanFactory` 구현과 함께 빈 팩토리 후처리기를 사용할 수 있지만, `CustomEditorConfigurer`에는 중첩된 속성 설정이 있으므로 다른 빈과 유사한 방식으로 배포하고 자동으로 감지 및 적용될 수 있는 `ApplicationContext`와 함께 사용하는 것을 강력히 권장합니다.

모든 빈 팩토리와 애플리케이션 컨텍스트는 속성 변환을 처리하기 위해 `BeanWrapper`를 사용하므로 자동으로 여러 내장 속성 편집기를 사용한다는 점에 유의하십시오.

`ApplicationContext`는 특정 애플리케이션 컨텍스트 타입에 적합한 방식으로 리소스 룩업을 처리하기 위해 추가 편집기를 오버라이드하거나 추가합니다.

표준 JavaBeans `PropertyEditor` 인스턴스는 문자열로 표현된 속성 값을 속성의 실제 복잡한 타입으로 변환하는 데 사용됩니다.

빈 팩토리 후처리기인 `CustomEditorConfigurer`를 사용하여 추가 `PropertyEditor` 인스턴스에 대한 지원을 `ApplicationContext`에 편리하게 추가할 수 있으며, 그러면 필요에 따라 사용할 수 있습니다.

`ExoticType`이라는 사용자 클래스와 `ExoticType`을 속성으로 설정해야 하는 `DependsOnExoticType`이라는 다른 클래스를 정의하는 다음 예제를 고려하십시오:

```java
// Java
package example;

public class ExoticType {
	private String name;
	public ExoticType(String name) { this.name = name; }
    public String getName() { return name; }
}

public class DependsOnExoticType {
	private ExoticType type;
	public void setType(ExoticType type) { this.type = type; }
    public ExoticType getType() { return type; }
}
```

```kotlin
// Kotlin
package example

data class ExoticType(var name: String)

data class DependsOnExoticType(var type: ExoticType? = null)
```

제대로 설정되면, `PropertyEditor`가 실제 `ExoticType` 인스턴스로 변환할 문자열로 `type` 속성을 할당할 수 있기를 원합니다. 다음 빈 정의는 이 관계를 설정하는 방법을 보여줍니다:

```xml
<bean id="sample" class="example.DependsOnExoticType">
	<property name="type" value="aNameForExoticType"/>
</bean>
```

`PropertyEditor` 구현은 다음과 유사할 수 있습니다:

```java
// Java
package example;

import java.beans.PropertyEditorSupport;

// 문자열 표현을 ExoticType 객체로 변환
public class ExoticTypeEditor extends PropertyEditorSupport {

	@Override
	public void setAsText(String text) {
		setValue(new ExoticType(text.toUpperCase()));
	}
}
```

```kotlin
// Kotlin
package example

import java.beans.PropertyEditorSupport

// 문자열 표현을 ExoticType 객체로 변환
class ExoticTypeEditor : PropertyEditorSupport() {

    override fun setAsText(text: String?) {
        // Handle potential null text if needed
        value = text?.let { ExoticType(it.uppercase()) }
    }
}
```

마지막으로, 다음 예제는 `CustomEditorConfigurer`를 사용하여 새로운 `PropertyEditor`를 `ApplicationContext`에 등록하는 방법을 보여주며, 그러면 필요에 따라 사용할 수 있습니다:

```xml
<bean class="org.springframework.beans.factory.config.CustomEditorConfigurer">
	<property name="customEditors">
		<map>
			<entry key="example.ExoticType" value="example.ExoticTypeEditor"/>
		</map>
	</property>
</bean>
```

*PropertyEditorRegistrar*
스프링 컨테이너에 속성 편집기를 등록하는 또 다른 메커니즘은 `PropertyEditorRegistrar`를 생성하고 사용하는 것입니다.

이 인터페이스는 여러 다른 상황에서 동일한 속성 편집기 세트를 사용해야 할 때 특히 유용합니다.

해당 등록기관(registrar)을 작성하고 각 경우에 재사용할 수 있습니다.

`PropertyEditorRegistrar` 인스턴스는 스프링 `BeanWrapper`(및 `DataBinder`)에 의해 구현되는 인터페이스인 `PropertyEditorRegistry` 인터페이스와 함께 작동합니다.

`PropertyEditorRegistrar` 인스턴스는 `setPropertyEditorRegistrars(..)`라는 속성을 노출하는 `CustomEditorConfigurer`(여기서 설명됨)와 함께 사용할 때 특히 편리합니다.

이러한 방식으로 `CustomEditorConfigurer`에 추가된 `PropertyEditorRegistrar` 인스턴스는 `DataBinder` 및 스프링 MVC 컨트롤러와 쉽게 공유될 수 있습니다.

더욱이, 커스텀 편집기에 대한 동기화 필요성을 피합니다: `PropertyEditorRegistrar`는 각 빈 생성 시도에 대해 새로운 `PropertyEditor` 인스턴스를 생성할 것으로 예상됩니다.

다음 예제는 자신만의 `PropertyEditorRegistrar` 구현을 생성하는 방법을 보여줍니다:

```java
// Java
package com.foo.editors.spring;

import example.ExoticType; // Assuming ExoticType is in example package
import example.ExoticTypeEditor; // Assuming ExoticTypeEditor is in example package
import org.springframework.beans.PropertyEditorRegistrar;
import org.springframework.beans.PropertyEditorRegistry;

public final class CustomPropertyEditorRegistrar implements PropertyEditorRegistrar {

	@Override
	public void registerCustomEditors(PropertyEditorRegistry registry) {

		// 새로운 PropertyEditor 인스턴스가 생성될 것으로 예상됨
		registry.registerCustomEditor(ExoticType.class, new ExoticTypeEditor());

		// 여기서 필요한 만큼 많은 커스텀 속성 편집기를 등록할 수 있습니다...
	}
}
```

```kotlin
// Kotlin
package com.foo.editors.spring

import example.ExoticType // Assuming ExoticType is in example package
import example.ExoticTypeEditor // Assuming ExoticTypeEditor is in example package
import org.springframework.beans.PropertyEditorRegistrar
import org.springframework.beans.PropertyEditorRegistry

class CustomPropertyEditorRegistrar : PropertyEditorRegistrar {

    override fun registerCustomEditors(registry: PropertyEditorRegistry) {

        // 새로운 PropertyEditor 인스턴스가 생성될 것으로 예상됨
        registry.registerCustomEditor(ExoticType::class.java, ExoticTypeEditor())

        // 여기서 필요한 만큼 많은 커스텀 속성 편집기를 등록할 수 있습니다...
    }
}
```

`registerCustomEditors(..)` 메소드 구현에서 각 속성 편집기의 새 인스턴스를 생성하는 방식에 주목하십시오.

다음 예제는 `CustomEditorConfigurer`를 구성하고 `CustomPropertyEditorRegistrar` 인스턴스를 주입하는 방법을 보여줍니다:

```xml
<bean class="org.springframework.beans.factory.config.CustomEditorConfigurer">
	<property name="propertyEditorRegistrars">
		<list>
			<ref bean="customPropertyEditorRegistrar"/>
		</list>
	</property>
</bean>

<bean id="customPropertyEditorRegistrar"
	class="com.foo.editors.spring.CustomPropertyEditorRegistrar"/>
```

마지막으로 (그리고 이 장의 초점에서 약간 벗어나서) 스프링의 MVC 웹 프레임워크를 사용하는 분들에게는 데이터 바인딩 웹 컨트롤러와 함께 `PropertyEditorRegistrar`를 사용하는 것이 매우 편리할 수 있습니다.

다음 예제는 `@InitBinder` 메소드 구현에서 `PropertyEditorRegistrar`를 사용합니다:

```java
// Java
import org.springframework.beans.PropertyEditorRegistrar;
import org.springframework.stereotype.Controller;
import org.springframework.web.bind.WebDataBinder;
import org.springframework.web.bind.annotation.InitBinder;

@Controller
public class RegisterUserController {

	private final PropertyEditorRegistrar customPropertyEditorRegistrar;

	// Inject the registrar (e.g., via constructor)
	public RegisterUserController(PropertyEditorRegistrar propertyEditorRegistrar) {
		this.customPropertyEditorRegistrar = propertyEditorRegistrar;
	}

	@InitBinder // Method to initialize the WebDataBinder
	void initBinder(WebDataBinder binder) {
		this.customPropertyEditorRegistrar.registerCustomEditors(binder);
	}

	// 사용자 등록 관련 다른 메소드들
}
```

```kotlin
// Kotlin
import org.springframework.beans.PropertyEditorRegistrar
import org.springframework.stereotype.Controller
import org.springframework.web.bind.WebDataBinder
import org.springframework.web.bind.annotation.InitBinder

@Controller
class RegisterUserController( // Inject the registrar via constructor
    private val customPropertyEditorRegistrar: PropertyEditorRegistrar
) {

    @InitBinder // Method to initialize the WebDataBinder
    fun initBinder(binder: WebDataBinder) {
        this.customPropertyEditorRegistrar.registerCustomEditors(binder)
    }

    // 사용자 등록 관련 다른 메소드들
}
```

이 스타일의 `PropertyEditor` 등록은 간결한 코드( `@InitBinder` 메소드 구현이 한 줄뿐임)로 이어질 수 있으며, 공통 `PropertyEditor` 등록 코드를 클래스에 캡슐화한 다음 필요한 만큼 많은 컨트롤러 간에 공유할 수 있게 합니다.

---

**전체 주제: 데이터 바인딩 (Data Binding)**

이 부분은 주로 **사용자 입력(예: 웹 폼 데이터, API 요청 본문 등)** 을 애플리케이션의 **자바 객체(주로 JavaBeans)** 에 **자동으로 채워 넣는(바인딩)** 과정과 관련된 스프링의 기능을 설명합니다.

또한 JavaBeans의 속성을 다루는 기본적인 메커니즘도 함께 다룹니다.

**핵심 아이디어:** 사용자가 보낸 문자열 형태의 데이터를 일일이 꺼내서 자바 객체의 필드에 수동으로 넣는 대신, 스프링이 알아서 적절한 타입으로 변환하여 객체의 속성에 자동으로 바인딩(연결)해주도록 하자!

---

**1. 데이터 바인딩이란?**

- **개념:** 소스(Source, 예: HTTP 요청 파라미터 Map, 프로퍼티 파일)에 있는 데이터를 타겟(Target, 예: `User` 객체, `OrderForm` 객체) 객체의 **속성(property)** 에 자동으로 **매핑하고 설정**하는 프로세스입니다.
- **필요성:** 웹 애플리케이션에서 사용자가 폼에 입력한 수많은 필드 값들을 받아서, 이를 처리할 자바 객체(커맨드 객체, DTO 등)의 각 필드에 일일이 `setUser(request.getParameter("user"))`, `setEmail(request.getParameter("email"))` 처럼 코딩하는 것은 매우 번거롭고 오류가 발생하기 쉽습니다. 데이터 바인딩은 이 과정을 자동화해줍니다.
- **핵심 요소:**
  - **소스 데이터:** 일반적으로 **키-값** 형태로 제공됩니다 (예: `Map<String, String>`, `HttpServletRequest` 파라미터). 키는 보통 타겟 객체의 **속성 경로**(예: "name", "address.city")와 일치합니다.
  - **타겟 객체:** 데이터를 담을 자바 객체 (주로 JavaBeans 규약을 따름).
  - **데이터 바인더 (`DataBinder`):** 소스 데이터를 타겟 객체에 실제로 바인딩하는 작업을 수행하는 스프링의 핵심 클래스입니다. 타입 변환, 검증(Validation) 등의 기능도 함께 처리할 수 있습니다.

---

**2. `DataBinder`와 두 가지 바인딩 방식**

스프링의 `org.springframework.validation.DataBinder`는 데이터 바인딩을 수행하는 주요 클래스입니다. 두 가지 주요 바인딩 방식을 제공합니다.

- **(1) 생성자 바인딩 (Constructor Binding):**
  - **동작:** 소스 데이터를 타겟 객체의 **생성자(Constructor)** 파라미터에 바인딩하여 **새로운 객체를 생성**합니다.
  - **사용법:**
    1. `DataBinder`를 생성할 때 타겟 객체 자리에 `null`을 넣고, `targetType` 속성에 생성할 클래스를 지정합니다.
    2. `construct(PropertyValue...)` 메소드를 호출하여 바인딩을 수행합니다.
  - **요구사항:** 대상 클래스는 단일 public 생성자 또는 단일 non-public 생성자를 가져야 합니다. (여러 생성자가 있으면 기본 생성자 사용 시도)
  - **값 매핑:** 기본적으로 생성자 **파라미터 이름**과 소스 데이터의 **키**를 매칭하여 값을 찾습니다. (스프링 MVC 등에서는 `@BindParam`으로 이름 커스터마이징 가능)
  - **특징:** 객체와 그 내부의 중첩된 객체까지 **모두 생성**하면서 바인딩합니다. 불변(immutable) 객체를 만드는 데 유용할 수 있습니다. `List`, `Map`, 배열 형태의 인수 바인딩도 지원합니다.
- **(2) 속성 바인딩 (Property Binding) - `BeanWrapper` 사용:**
  - **동작:** 소스 데이터를 **이미 존재하는** 타겟 객체의 **세터(setter) 메소드**를 통해 속성에 바인딩합니다.
  - **핵심 도구 (`BeanWrapper`):** `DataBinder`는 내부적으로 **`BeanWrapper`** 라는 것을 사용하여 속성 바인딩을 수행합니다.
  - **사용법:** `DataBinder`를 생성할 때 타겟 객체를 직접 전달하고, `bind(PropertyValues)` 등의 메소드를 호출합니다.
  - **값 매핑:** 소스 데이터의 **키**와 타겟 객체의 **속성 이름**(JavaBeans 규약에 따른 getter/setter 기준)을 매칭합니다.
  - **특징:** 가장 일반적인 데이터 바인딩 방식입니다. 기존 객체의 상태를 업데이트하는 데 사용됩니다.
- **혼합 사용:** 필요에 따라 생성자 바인딩과 속성 바인딩을 함께 사용할 수도 있습니다.

---

**3. `BeanWrapper` 인터페이스: JavaBeans 속성 조작의 핵심**

`BeanWrapper` (`org.springframework.beans.BeanWrapper`)는 스프링에서 **자바 빈(JavaBean) 객체의 속성을 다루는 핵심적인 로우 레벨(low-level) 인터페이스**입니다. `DataBinder`와 `BeanFactory`가 내부적으로 이를 사용합니다.

- **JavaBeans 표준:** `BeanWrapper`는 기본 인수 없는 생성자, getter/setter 명명 규칙(예: `getName()`, `setName()`)을 따르는 JavaBeans 표준을 기반으로 동작합니다.
- **주요 기능 (`BeanWrapperImpl` 구현체):**
  - **속성 값 설정/가져오기:** `setPropertyValue(String propertyName, Object value)`, `getPropertyValue(String propertyName)` 메소드를 사용하여 개별 속성 값을 설정하거나 가져올 수 있습니다.
  - **일괄 설정:** 여러 속성 값을 한 번에 설정할 수도 있습니다 (`setPropertyValues(...)`).
  - **중첩 속성 지원:** `"address.city"` 와 같이 점(`.`)으로 연결된 경로를 사용하여 객체 내부의 객체 속성까지 접근하고 설정/가져오기가 가능합니다. (무한 깊이 지원)
  - **인덱싱된 속성 지원:** 배열이나 리스트 속성의 특정 요소(`accounts[2]`), 맵 속성의 특정 키에 해당하는 값(`accounts[KEY]`)에 접근하고 설정할 수 있습니다.
  - **속성 정보 조회:** 특정 속성이 읽기 가능한지(`isReadableProperty`), 쓰기 가능한지(`isWritableProperty`), 속성의 타입이 무엇인지(`getPropertyType`) 등을 조회할 수 있습니다.
  - (부가 기능) `PropertyChangeListener`, `VetoableChangeListener` 등록 지원.
- **래핑(Wrapping):** 이름처럼 대상 빈 객체를 **"감싸서(wrap)"** 리플렉션(reflection) 등을 통해 내부적으로 해당 빈의 속성에 접근하고 조작하는 작업을 대신 수행해줍니다.
- **직접 사용:** 일반적인 애플리케이션 코드에서는 `BeanWrapper`를 직접 사용할 일이 거의 없습니다. 주로 `DataBinder`나 스프링 컨테이너 내부에서 사용됩니다.

---

**4. PropertyEditor (속성 편집기) - 타입 변환의 마법사**

- **역할:** 소스 데이터(주로 **문자열**)를 타겟 객체의 속성 타입(예: `int`, `Date`, `Boolean`, `Resource`, 커스텀 객체)으로 **변환**하거나, 그 반대로 변환하는 역할을 합니다.
- **필요성:** 사용자 입력은 대부분 문자열 형태인데, 자바 객체의 속성은 다양한 타입을 가집니다. 데이터 바인딩 시 이 **타입 불일치를 해결**해야 합니다.
- **스프링의 활용:**
  - `BeanWrapper` (그리고 이를 사용하는 `DataBinder`, `BeanFactory`)는 속성 값을 설정할 때, **자동으로 해당 속성 타입에 맞는 `PropertyEditor`를 찾아서 사용**합니다.
  - 예를 들어, 문자열 "true"를 `boolean` 타입 속성에 설정해야 할 때 `CustomBooleanEditor`를 사용하고, 문자열 "2023-01-01"을 `Date` 타입 속성에 설정해야 할 때 `CustomDateEditor`를 사용합니다. (단, `CustomDateEditor`는 기본 등록되지 않아 사용자가 포맷과 함께 등록해야 함)
- **내장 PropertyEditor:** 스프링은 다양한 기본 타입 및 자주 사용되는 타입(Class, File, URL, Locale 등)에 대한 `PropertyEditor` 구현체들을 **미리 내장하고 자동으로 등록**하여 사용합니다.
- **자동 검색:** JavaBeans 표준에 따라, `com.example.Something` 클래스에 대한 `PropertyEditor`는 같은 패키지의 `SomethingEditor` 라는 이름으로 만들면 스프링(정확히는 `PropertyEditorManager`)이 자동으로 찾아서 사용할 수도 있습니다. `SomethingBeanInfo` 클래스를 통해 명시적으로 등록할 수도 있습니다.

---

**5. 커스텀 PropertyEditor 등록하기**

만약 스프링이 기본 제공하지 않는 커스텀 타입(예: `ExoticType`)에 대한 문자열 변환 규칙이 필요하다면, 직접 `PropertyEditor`를 만들고 스프링 컨테이너에 등록해야 합니다.

- **커스텀 `PropertyEditor` 구현:** `java.beans.PropertyEditorSupport` 클래스를 상속받아 `setAsText(String text)` 메소드를 오버라이드하는 것이 일반적입니다. 이 메소드 안에서 입력된 문자열(`text`)을 파싱하여 실제 객체로 변환하고 `setValue(객체)`를 호출합니다.

    ```java
    public class ExoticTypeEditor extends PropertyEditorSupport {
        @Override
        public void setAsText(String text) {
            // 문자열 text를 기반으로 ExoticType 객체 생성
            setValue(new ExoticType(text.toUpperCase()));
        }
    }
    ```

- **등록 방법:**
  1. **`CustomEditorConfigurer` 사용 (권장):**
    - `CustomEditorConfigurer`는 `BeanFactoryPostProcessor`의 일종으로, 커스텀 `PropertyEditor`들을 컨테이너에 등록하는 역할을 합니다.
    - XML 설정에서 `CustomEditorConfigurer` 빈을 정의하고, `customEditors` 속성(Map 형태)에 `<entry key="타겟클래스이름" value="에디터클래스이름"/>` 형식으로 등록합니다.

        ```xml
        <bean class="org.springframework.beans.factory.config.CustomEditorConfigurer">
            <property name="customEditors">
                <map>
                    <entry key="example.ExoticType" value="example.ExoticTypeEditor"/>
                </map>
            </property>
        </bean>
        
        ```

  2. **`PropertyEditorRegistrar` 사용:**
    - 여러 곳에서 동일한 에디터들을 등록해야 할 때 유용합니다.
    - `PropertyEditorRegistrar` 인터페이스를 구현한 클래스를 만듭니다 (`registerCustomEditors` 메소드 구현).
    - 이 `Registrar` 빈을 `CustomEditorConfigurer`의 `propertyEditorRegistrars` 속성에 등록합니다.
    - 스프링 MVC 컨트롤러의 `@InitBinder` 메소드에서도 `PropertyEditorRegistrar`를 활용하여 해당 컨트롤러에서 사용할 커스텀 에디터를 편리하게 등록할 수 있습니다.
  3. `ConfigurableBeanFactory.registerCustomEditor()` 직접 호출 (비추천): `BeanFactory`를 직접 다룰 때 사용 가능하지만 번거롭습니다.

**요약:**

데이터 바인딩은 사용자 입력(주로 문자열 키-값)을 자바 객체의 속성에 자동으로 설정하는 과정입니다. 스프링의 `DataBinder`가 이 역할을 하며, 내부적으로 `BeanWrapper`를 사용하여 JavaBeans 속성을 조작합니다. 이 과정에서 **문자열 값을 실제 속성 타입으로 변환하기 위해 `PropertyEditor`가 핵심적인 역할**을 합니다. 스프링은 많은 내장 에디터를 제공하며, 개발자는 커스텀 에디터를 만들어 `CustomEditorConfigurer` 등을 통해 등록하여 사용할 수 있습니다. 생성자 바인딩과 속성(세터) 바인딩 두 가지 방식이 있습니다.

---

**데이터 바인딩과 커스텀 PropertyEditor의 실제 사용 시나리오 (주로 웹 애플리케이션)**

데이터 바인딩, 특히 커스텀 `PropertyEditor`가 가장 빛을 발하는 곳은 **스프링 MVC 웹 애플리케이션**입니다. 사용자가 웹 브라우저를 통해 폼(Form) 데이터를 제출하는 과정을 예시로 들자면

**상황:** 사용자가 상품 등록 폼에서 상품 정보와 함께, 특별한 형식의 "상품 코드"(예: "PRODUCT-A123")를 입력하여 제출한다고 가정해 봅시다. 우리는 이 문자열 "PRODUCT-A123"을 단순 문자열이 아닌, 특별한 로직을 가진 `ProductCode`라는 **커스텀 객체**로 변환하여 `Product` 객체의 `productCode` 필드에 저장하고 싶습니다.

**단계별 실제 사용법:**

**1단계: 커스텀 타입 정의 (`ProductCode.java`)**

```java
package com.example.domain;

// 상품 코드를 나타내는 커스텀 클래스
public class ProductCode {
    private final String prefix;
    private final String code;

    public ProductCode(String prefix, String code) {
        this.prefix = prefix;
        this.code = code;
    }

    public String getFullCode() {
        return prefix + "-" + code;
    }

    @Override
    public String toString() { // 디버깅 편의를 위해
        return getFullCode();
    }
    // Getter 등 필요에 따라 추가
}
```

**2단계: 커스텀 PropertyEditor 구현 (`ProductCodeEditor.java`)**

문자열 "PRODUCT-A123"을 `ProductCode` 객체로 변환하는 로직을 구현합니다.

```java
package com.example.editor;

import com.example.domain.ProductCode;
import java.beans.PropertyEditorSupport;

public class ProductCodeEditor extends PropertyEditorSupport {

    @Override
    public void setAsText(String text) throws IllegalArgumentException {
        if (text == null || !text.contains("-")) {
            // 간단한 유효성 검사 (실제로는 더 정교하게)
            throw new IllegalArgumentException("Invalid product code format: " + text);
        }
        String[] parts = text.split("-", 2);
        // 문자열을 파싱하여 ProductCode 객체 생성 후 설정
        setValue(new ProductCode(parts[0], parts[1]));
    }

    @Override
    public String getAsText() {
        // ProductCode 객체를 다시 문자열로 변환 (필요한 경우)
        ProductCode code = (ProductCode) getValue();
        return (code != null) ? code.getFullCode() : "";
    }
}
```

**3단계: 데이터 바인딩될 커맨드 객체 정의 (`ProductForm.java`)**

웹 폼 데이터를 담을 객체입니다. `ProductCode` 타입의 필드를 가집니다.

```java
package com.example.form;

import com.example.domain.ProductCode;

public class ProductForm {
    private String productName;
    private int price;
    private ProductCode productCode; // ★ 커스텀 타입 필드 ★

    // Getters and Setters
    public String getProductName() { return productName; }
    public void setProductName(String productName) { this.productName = productName; }
    public int getPrice() { return price; }
    public void setPrice(int price) { this.price = price; }
    public ProductCode getProductCode() { return productCode; }
    public void setProductCode(ProductCode productCode) { this.productCode = productCode; }

    @Override
    public String toString() { // 결과 확인용
        return "ProductForm{" +
               "productName='" + productName + '\\'' +
               ", price=" + price +
               ", productCode=" + productCode + // ProductCode 객체의 toString() 호출됨
               '}';
    }
}
```

**4단계: 커스텀 PropertyEditor 등록 및 사용 (스프링 MVC 컨트롤러)**

스프링 MVC 컨트롤러에서 사용자가 제출한 폼 데이터를 `ProductForm` 객체에 **자동으로 바인딩**하도록 하고, 이때 **우리가 만든 `ProductCodeEditor`를 사용**하도록 설정합니다. `@InitBinder` 어노테이션을 사용하는 것이 일반적입니다.

```java
package com.example.controller;

import com.example.domain.ProductCode;
import com.example.editor.ProductCodeEditor; // 에디터 임포트
import com.example.form.ProductForm;
import org.springframework.stereotype.Controller;
import org.springframework.ui.Model;
import org.springframework.validation.BindingResult;
import org.springframework.web.bind.WebDataBinder;
import org.springframework.web.bind.annotation.GetMapping;
import org.springframework.web.bind.annotation.InitBinder; // ★ InitBinder 임포트 ★
import org.springframework.web.bind.annotation.PostMapping;
import javax.validation.Valid; // (선택) JSR-303 검증 사용 시

@Controller
public class ProductController {

    // ★★★ @InitBinder: 이 컨트롤러 내에서의 데이터 바인딩 설정을 위한 메소드 ★★★
    @InitBinder
    public void initBinder(WebDataBinder binder) {
        // WebDataBinder 객체에 커스텀 에디터를 등록합니다.
        // "ProductCode" 타입에 대해서는 "ProductCodeEditor"를 사용하라고 알려줍니다.
        binder.registerCustomEditor(ProductCode.class, new ProductCodeEditor());
        System.out.println("ProductCodeEditor가 WebDataBinder에 등록됨!");
    }

    @GetMapping("/product/add")
    public String showAddForm(Model model) {
        model.addAttribute("productForm", new ProductForm());
        return "product-add-form"; // 상품 등록 폼 뷰 이름 (예: product-add-form.html/jsp)
    }

    @PostMapping("/product/add")
    // ★ ProductForm 객체를 파라미터로 받으면 스프링 MVC가 자동으로 폼 데이터 바인딩 수행 ★
    public String addProduct(@Valid ProductForm productForm, BindingResult bindingResult, Model model) {

        System.out.println("제출된 폼 데이터 바인딩 결과: " + productForm);

        if (bindingResult.hasErrors()) {
            System.out.println("폼 데이터 검증 오류 발생!");
            // 오류가 있으면 다시 폼으로
            return "product-add-form";
        }

        // 바인딩 및 검증 성공 시 서비스 로직 호출 등
        // productForm.getProductName(), productForm.getPrice(), productForm.getProductCode() 사용 가능
        // productForm.getProductCode()는 이미 ProductCode 객체 상태!
        System.out.println("Product Code Prefix: " + productForm.getProductCode().getPrefix()); // 예시
        System.out.println("Product Code Code: " + productForm.getProductCode().getCode()); // 예시

        model.addAttribute("message", "상품이 성공적으로 등록되었습니다!");
        return "product-success"; // 성공 뷰 이름
    }
}
```

**5단계: 웹 폼 (`product-add-form.html` - 예시, Thymeleaf 사용)**

```html
<!DOCTYPE html>
<html xmlns:th="<http://www.thymeleaf.org>">
<head>
    <title>상품 등록</title>
</head>
<body>
    <h1>상품 등록</h1>
    <form th:action="@{/product/add}" th:object="${productForm}" method="post">
        <div>
            <label for="productName">상품명:</label>
            <input type="text" id="productName" th:field="*{productName}" />
            <span th:if="${#fields.hasErrors('productName')}" th:errors="*{productName}"></span>
        </div>
        <div>
            <label for="price">가격:</label>
            <input type="number" id="price" th:field="*{price}" />
            <span th:if="${#fields.hasErrors('price')}" th:errors="*{price}"></span>
        </div>
        <div>
            <label for="productCode">상품 코드 (예: PROD-123):</label>
            <!-- ★ 사용자는 문자열로 입력 ★ -->
            <input type="text" id="productCode" th:field="*{productCode}" />
            <!-- ★ 오류 발생 시 ProductCodeEditor의 변환 오류 또는 Validator 오류 메시지 표시 ★ -->
            <span th:if="${#fields.hasErrors('productCode')}" th:errors="*{productCode}"></span>
        </div>
        <button type="submit">등록</button>
    </form>
</body>
</html>
```

**동작 흐름 요약:**

1. 사용자가 웹 폼에 상품명, 가격, 상품 코드("PRODUCT-A123")를 입력하고 제출합니다.
2. 요청이 `/product/add` POST 매핑이 있는 `ProductController`의 `addProduct` 메소드로 전달됩니다.
3. 스프링 MVC는 `@PostMapping` 메소드의 `ProductForm productForm` 파라미터를 보고, **자동으로 데이터 바인딩을 시도**합니다.
4. `productName`과 `price`는 문자열/숫자이므로 스프링의 기본 변환기로 처리됩니다.
5. `productCode` 필드에 값을 바인딩하려고 할 때, 스프링 MVC는 **`@InitBinder`로 등록된 `ProductCodeEditor`를 발견**합니다.
6. `ProductCodeEditor`의 `setAsText("PRODUCT-A123")` 메소드가 호출됩니다.
7. `ProductCodeEditor`는 문자열을 파싱하여 `new ProductCode("PRODUCT", "A123")` 객체를 생성하고 `setValue()` 합니다.
8. 이 **`ProductCode` 객체**가 `productForm` 객체의 `productCode` 필드에 최종적으로 설정됩니다.
9. 만약 `@Valid`와 함께 JSR-303 검증 어노테이션(`@NotNull` 등)이나 스프링 `Validator`가 설정되어 있다면, 바인딩 이후 검증 과정이 수행됩니다.
10. 컨트롤러 메소드 본문에서는 `productForm.getProductCode()`를 호출하면 이미 변환된 `ProductCode` 객체를 바로 사용할 수 있습니다.

**결론:**

데이터 바인딩, 특히 커스텀 `PropertyEditor`는 주로 **스프링 MVC 환경**에서 사용자의 **문자열 입력을 컨트롤러의 커맨드 객체(Form 객체, DTO)의 특정 타입 필드로 자동 변환**하는 데 매우 유용하게 사용됩니다. `@InitBinder`를 사용하여 특정 컨트롤러 또는 전역적으로 커스텀 에디터를 등록하고, 스프링 MVC는 요청 처리 시 이 에디터를 자동으로 활용하여 데이터 바인딩을 수행합니다. 이를 통해 개발자는 번거로운 타입 변환 코드를 직접 작성할 필요 없이 비즈니스 로직에 집중할 수 있습니다.
