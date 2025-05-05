---
title: Spring Framework Resources
description: 
author: laze
date: 2025-05-05 00:00:01 +0900
categories: [Dev, SpringBoot]
tags: [SpringBoot]
---
**Resources**

이 장에서는 스프링이 리소스를 어떻게 처리하는지, 그리고 스프링에서 리소스를 어떻게 사용할 수 있는지 다룹니다.

- 소개
- `Resource` 인터페이스
- 내장 `Resource` 구현체
- `ResourceLoader` 인터페이스
- `ResourcePatternResolver` 인터페이스
- `ResourceLoaderAware` 인터페이스
- 의존성으로서의 리소스
- 애플리케이션 컨텍스트와 리소스 경로

**소개 (Introduction)**

자바의 표준 `java.net.URL` 클래스와 다양한 URL 접두사에 대한 표준 핸들러는 불행히도 저수준 리소스에 대한 모든 접근에 충분하지 않습니다.

예를 들어, 클래스패스나 `ServletContext`에 상대적으로 얻어야 하는 리소스에 접근하는 데 사용할 수 있는 표준화된 URL 구현이 없습니다.

전문화된 URL 접두사(예: `http:`와 같은 기존 핸들러와 유사)에 대한 새로운 핸들러를 등록하는 것이 가능하지만,

이는 일반적으로 매우 복잡하며 URL 인터페이스에는 여전히 지정된 리소스의 존재 여부를 확인하는 메소드와 같은 몇 가지 바람직한 기능이 부족합니다.

**`Resource` 인터페이스 (The Resource Interface)**

스프링의 `Resource` 인터페이스(`org.springframework.core.io` 패키지에 위치)는 저수준 리소스에 대한 접근을 추상화하기 위한 더 유능한 인터페이스가 되도록 의도되었습니다.

다음 목록은 `Resource` 인터페이스의 개요를 제공합니다. 자세한 내용은 `Resource` javadoc을 참조하십시오.

```java
public interface Resource extends InputStreamSource {

	boolean exists();

	boolean isReadable();

	boolean isOpen();

	boolean isFile();

	URL getURL() throws IOException;

	URI getURI() throws IOException;

	File getFile() throws IOException;

	ReadableByteChannel readableChannel() throws IOException;

	long contentLength() throws IOException;

	long lastModified() throws IOException;

	Resource createRelative(String relativePath) throws IOException;

	String getFilename();

	String getDescription();
}
```

`Resource` 인터페이스 정의에서 보여주듯이, 이는 `InputStreamSource` 인터페이스를 확장합니다. 다음 목록은 `InputStreamSource` 인터페이스의 정의를 보여줍니다:

```java
public interface InputStreamSource {

	InputStream getInputStream() throws IOException;
}
```

`Resource` 인터페이스의 가장 중요한 메소드 중 일부는 다음과 같습니다:

- `getInputStream()`: 리소스를 찾아 열고, 리소스에서 읽기 위한 `InputStream`을 반환합니다. 각 호출은 새로운 `InputStream`을 반환할 것으로 예상됩니다. 스트림을 닫는 것은 호출자의 책임입니다.
- `exists()`: 이 리소스가 실제로 물리적 형태로 존재하는지 여부를 나타내는 부울 값을 반환합니다.
- `isOpen()`: 이 리소스가 열린 스트림을 가진 핸들(handle)을 나타내는지 여부를 나타내는 부울 값을 반환합니다. `true`이면 `InputStream`을 여러 번 읽을 수 없으며 한 번만 읽고 리소스 누수(resource leaks)를 피하기 위해 닫아야 합니다. `InputStreamResource`를 제외한 모든 일반적인 리소스 구현에 대해 `false`를 반환합니다.
- `getDescription()`: 리소스로 작업할 때 오류 출력에 사용될 이 리소스에 대한 설명을 반환합니다. 이는 종종 정규화된 파일 이름이나 리소스의 실제 URL입니다.

다른 메소드들은 리소스를 나타내는 실제 `URL` 또는 `File` 객체를 얻을 수 있게 합니다 (기본 구현이 호환되고 해당 기능을 지원하는 경우).

`Resource` 인터페이스의 일부 구현체는 쓰기를 지원하는 리소스에 대해 확장된 `WritableResource` 인터페이스도 구현합니다.

스프링 자체는 리소스가 필요할 때 많은 메소드 시그니처에서 인수 타입으로 `Resource` 추상화를 광범위하게 사용합니다.

일부 스프링 API의 다른 메소드(예: 다양한 `ApplicationContext` 구현체의 생성자)는 꾸밈없거나(unadorned) 단순한 형태의 `String`을 받으며,

이는 해당 컨텍스트 구현에 적합한 `Resource`를 생성하는 데 사용되거나, 문자열 경로의 특수 접두사를 통해 호출자가 특정 `Resource` 구현을 생성하고 사용해야 함을 지정할 수 있도록 합니다.

`Resource` 인터페이스는 스프링과 함께 그리고 스프링에 의해 많이 사용되지만, 코드가 스프링의 다른 부분을 알거나 신경 쓰지 않더라도 리소스 접근을 위해 자체 코드에서 일반 유틸리티 클래스로 사용하는 것이 실제로는 매우 편리합니다.

이는 코드를 스프링에 결합시키지만, 실제로는 `URL`에 대한 더 유능한 대체 역할을 하고 이 목적을 위해 사용할 다른 라이브러리와 동등하다고 간주될 수 있는 이 작은 유틸리티 클래스 집합에만 결합시킵니다.

`*Resource` 추상화는 기능을 대체하지 않습니다.*

*가능한 경우 래핑(wraps)합니다. 예를 들어, `UrlResource`는 URL을 래핑하고 래핑된 URL을 사용하여 작업을 수행합니다.*

**내장 `Resource` 구현체 (Built-in Resource Implementations)**

스프링은 여러 내장 `Resource` 구현체를 포함합니다:

- `UrlResource`
- `ClassPathResource`
- `FileSystemResource`
- `PathResource`
- `ServletContextResource`
- `InputStreamResource`
- `ByteArrayResource`

*UrlResource*`UrlResource`는 `java.net.URL`을 래핑하며, 파일, HTTPS 대상, FTP 대상 등과 같이 일반적으로 URL로 접근 가능한 모든 객체에 접근하는 데 사용될 수 있습니다.

모든 URL에는 표준화된 `String` 표현이 있으며, 적절한 표준화된 접두사가 사용되어 한 URL 타입을 다른 타입과 구별합니다.

이는 파일 시스템 경로 접근을 위한 `file:`, HTTPS 프로토콜을 통한 리소스 접근을 위한 `https:`, FTP를 통한 리소스 접근을 위한 `ftp:` 등을 포함합니다.

`UrlResource`는 명시적으로 `UrlResource` 생성자를 사용하여 자바 코드로 생성되지만, 경로를 나타내기 위한 `String` 인수를 받는 API 메소드를 호출할 때 암묵적으로 생성되는 경우가 많습니다.

후자의 경우, JavaBeans `PropertyEditor`가 궁극적으로 어떤 타입의 `Resource`를 생성할지 결정합니다.

경로 문자열에 (즉, 프로퍼티 에디터에게) 잘 알려진 접두사(예: `classpath:`)가 포함된 경우, 해당 접두사에 적합한 특수화된 `Resource`를 생성합니다.

그러나 접두사를 인식하지 못하면 문자열이 표준 URL 문자열이라고 가정하고 `UrlResource`를 생성합니다.

*ClassPathResource*
이 클래스는 클래스패스에서 얻어야 하는 리소스를 나타냅니다.

리소스를 로드하기 위해 스레드 컨텍스트 클래스 로더, 주어진 클래스 로더 또는 주어진 클래스 중 하나를 사용합니다.

이 `Resource` 구현은 클래스패스 리소스가 파일 시스템에 있는 경우 `java.io.File`로서의 해석을 지원하지만, jar에 있고 파일 시스템으로 확장되지 않은(서블릿 엔진이나 환경이 무엇이든) 클래스패스 리소스에 대해서는 지원하지 않습니다.

이를 해결하기 위해 다양한 `Resource` 구현체는 항상 `java.net.URL`로서의 해석을 지원합니다.

`ClassPathResource`는 명시적으로 `ClassPathResource` 생성자를 사용하여 자바 코드로 생성되지만, 경로를 나타내기 위한 `String` 인수를 받는 API 메소드를 호출할 때 암묵적으로 생성되는 경우가 많습니다.

후자의 경우, JavaBeans `PropertyEditor`는 문자열 경로의 특수 접두사인 `classpath:`를 인식하고 해당 경우에 `ClassPathResource`를 생성합니다.

*FileSystemResource*
이것은 `java.io.File` 핸들에 대한 `Resource` 구현입니다. 또한 `java.nio.file.Path` 핸들도 지원하며, 스프링의 표준 문자열 기반 경로 변환을 적용하지만 모든 작업은 `java.nio.file.Files` API를 통해 수행합니다.

순수한 `java.nio.path.Path` 기반 지원을 위해서는 대신 `PathResource`를 사용하십시오.

`FileSystemResource`는 `File` 및 `URL`로서의 해석을 지원합니다.

*PathResource*
이것은 `java.nio.file.Path` 핸들에 대한 `Resource` 구현이며, 모든 작업과 변환을 `Path` API를 통해 수행합니다.

`File` 및 `URL`로서의 해석을 지원하며 확장된 `WritableResource` 인터페이스도 구현합니다.

`PathResource`는 다른 `createRelative` 동작을 가진 `FileSystemResource`에 대한 효과적으로 순수한 `java.nio.path.Path` 기반 대안입니다.

*ServletContextResource*
이것은 관련 웹 애플리케이션의 루트 디렉토리 내에서 상대 경로를 해석하는 `ServletContext` 리소스에 대한 `Resource` 구현입니다.

항상 스트림 접근 및 URL 접근을 지원하지만, 웹 애플리케이션 아카이브가 확장되고 리소스가 물리적으로 파일 시스템에 있는 경우에만 `java.io.File` 접근을 허용합니다.

파일 시스템에 확장되어 있는지 또는 JAR이나 데이터베이스와 같은 다른 곳에서 직접 접근되는지(상상 가능함) 여부는 실제로 서블릿 컨테이너에 따라 다릅니다.

*InputStreamResource*`InputStreamResource`는 주어진 `InputStream`에 대한 `Resource` 구현입니다.

특정 `Resource` 구현이 적용 가능하지 않은 경우에만 사용해야 합니다. 특히, 가능한 경우 `ByteArrayResource` 또는 파일 기반 `Resource` 구현 중 하나를 선호하십시오.

다른 `Resource` 구현과 대조적으로, 이것은 이미 열린 리소스에 대한 기술자(descriptor)입니다.

따라서 `isOpen()`에서 `true`를 반환합니다.

리소스 기술자를 어딘가에 보관해야 하거나 스트림을 여러 번 읽어야 하는 경우에는 사용하지 마십시오.

*ByteArrayResource*
이것은 주어진 바이트 배열에 대한 `Resource` 구현입니다.

주어진 바이트 배열에 대해 `ByteArrayInputStream`을 생성합니다.

일회용 `InputStreamResource`에 의존하지 않고 주어진 바이트 배열에서 콘텐츠를 로드하는 데 유용합니다.

**`ResourceLoader` 인터페이스 (The ResourceLoader Interface)**

`ResourceLoader` 인터페이스는 `Resource` 인스턴스를 반환(즉, 로드)할 수 있는 객체에 의해 구현되도록 의도되었습니다.

```java
public interface ResourceLoader {

	Resource getResource(String location);

	ClassLoader getClassLoader();
}
```

모든 애플리케이션 컨텍스트는 `ResourceLoader` 인터페이스를 구현합니다.

따라서 모든 애플리케이션 컨텍스트를 사용하여 `Resource` 인스턴스를 얻을 수 있습니다.

특정 애플리케이션 컨텍스트에서 `getResource()`를 호출하고 지정된 위치 경로에 특정 접두사가 없는 경우, 해당 특정 애플리케이션 컨텍스트에 적합한 `Resource` 타입을 반환합니다.

예를 들어, 다음 코드 스니펫이 `ClassPathXmlApplicationContext` 인스턴스에 대해 실행되었다고 가정해 봅시다:

```java
// Java
Resource template = ctx.getResource("some/resource/path/myTemplate.txt");
```

```kotlin
// Kotlin
val template: Resource = ctx.getResource("some/resource/path/myTemplate.txt")
```

`ClassPathXmlApplicationContext`에 대해 해당 코드는 `ClassPathResource`를 반환합니다.

동일한 메소드가 `FileSystemXmlApplicationContext` 인스턴스에 대해 실행되면 `FileSystemResource`를 반환합니다.

`WebApplicationContext`의 경우 `ServletContextResource`를 반환합니다.

각 컨텍스트에 대해 유사하게 적절한 객체를 반환합니다.

결과적으로 특정 애플리케이션 컨텍스트에 적합한 방식으로 리소스를 로드할 수 있습니다.

반면에 다음 예제와 같이 특수 `classpath:` 접두사를 지정하여 애플리케이션 컨텍스트 타입에 관계없이 `ClassPathResource` 사용을 강제할 수도 있습니다:

```java
// Java
Resource template = ctx.getResource("classpath:some/resource/path/myTemplate.txt");
```

```kotlin
// Kotlin
val template: Resource = ctx.getResource("classpath:some/resource/path/myTemplate.txt")
```

유사하게, 표준 `java.net.URL` 접두사 중 하나를 지정하여 `UrlResource` 사용을 강제할 수 있습니다.

다음 예제는 `file` 및 `https` 접두사를 사용합니다:

```java
// Java
Resource template = ctx.getResource("file:///some/resource/path/myTemplate.txt");
```

```kotlin
// Kotlin
val template1: Resource = ctx.getResource("file:///some/resource/path/myTemplate.txt")
```

```java
// Java
Resource template = ctx.getResource("<https://myhost.com/resource/path/myTemplate.txt>");
```

```kotlin
// Kotlin
val template2: Resource = ctx.getResource("<https://myhost.com/resource/path/myTemplate.txt>")
```

다음 표는 `String` 객체를 `Resource` 객체로 변환하는 전략을 요약합니다:

표 1. 리소스 문자열

| 접두사 | 예제 | 설명 |
| --- | --- | --- |
| `classpath:` | `classpath:com/myapp/config.xml` | 클래스패스에서 로드됩니다. |
| `file:` | `file:///data/config.xml` | 파일 시스템에서 URL로 로드됩니다. `FileSystemResource` 주의사항도 참조하십시오. |
| `https:` | `https://myserver/logo.png` | URL로 로드됩니다. |
| (없음) | `/data/config.xml` | 기본 `ApplicationContext`에 따라 다릅니다. |

**`ResourcePatternResolver` 인터페이스 (The ResourcePatternResolver Interface)**

`ResourcePatternResolver` 인터페이스는 위치 패턴(예: Ant 스타일 경로 패턴)을 `Resource` 객체로 해석(resolving)하는 전략을 정의하는 `ResourceLoader` 인터페이스의 확장입니다.

```java
public interface ResourcePatternResolver extends ResourceLoader {

	String CLASSPATH_ALL_URL_PREFIX = "classpath*:";

	Resource[] getResources(String locationPattern) throws IOException;
}
```

위에서 볼 수 있듯이, 이 인터페이스는 또한 클래스 경로에서 일치하는 모든 리소스에 대한 특수 `classpath*:` 리소스 접두사를 정의합니다.

리소스 위치는 이 경우 플레이스홀더 없는 경로여야 한다는 점에 유의하십시오 - 예를 들어, `classpath*:/config/beans.xml`.

JAR 파일이나 클래스 경로의 다른 디렉토리는 동일한 경로와 동일한 이름을 가진 여러 파일을 포함할 수 있습니다.

전달된 `ResourceLoader`(예: `ResourceLoaderAware` 시맨틱을 통해 제공됨)가 이 확장 인터페이스도 구현하는지 확인할 수 있습니다.

`PathMatchingResourcePatternResolver`는 `ApplicationContext` 외부에서 사용할 수 있는 독립 실행형 구현이며 `Resource[]` 빈 속성을 채우기 위해 `ResourceArrayPropertyEditor`에서도 사용됩니다. `PathMatchingResourcePatternResolver`는 지정된 리소스 위치 경로를 하나 이상의 일치하는 `Resource` 객체로 해석할 수 있습니다. 소스 경로는 대상 `Resource`에 대한 일대일 매핑을 가진 단순 경로일 수도 있고, 또는 특수 `classpath*:` 접두사 및/또는 내부 Ant 스타일 정규 표현식(스프링의 `org.springframework.util.AntPathMatcher` 유틸리티를 사용하여 매치됨)을 포함할 수도 있습니다.

후자의 두 가지 모두 효과적으로 와일드카드입니다.

*모든 표준 `ApplicationContext`의 기본 `ResourceLoader`는 실제로 `ResourcePatternResolver` 인터페이스를 구현하는 `PathMatchingResourcePatternResolver`의 인스턴스입니다.*

`*ApplicationContext` 인스턴스 자체도 `ResourcePatternResolver` 인터페이스를 구현하고 기본 `PathMatchingResourcePatternResolver`에 위임하므로 동일합니다.*

**`ResourceLoaderAware` 인터페이스 (The ResourceLoaderAware Interface)**

`ResourceLoaderAware` 인터페이스는 `ResourceLoader` 참조가 제공될 것으로 기대하는 컴포넌트를 식별하는 특수 콜백 인터페이스입니다.

다음 목록은 `ResourceLoaderAware` 인터페이스의 정의를 보여줍니다:

```java
public interface ResourceLoaderAware {

	void setResourceLoader(ResourceLoader resourceLoader);
}
```

클래스가 `ResourceLoaderAware`를 구현하고 애플리케이션 컨텍스트(스프링 관리 빈으로)에 배포되면, 애플리케이션 컨텍스트에 의해 `ResourceLoaderAware`로 인식됩니다.

그러면 애플리케이션 컨텍스트는 `setResourceLoader(ResourceLoader)`를 호출하여 자체를 인수로 제공합니다 (스프링의 모든 애플리케이션 컨텍스트는 `ResourceLoader` 인터페이스를 구현합니다).

`ApplicationContext`는 `ResourceLoader`이므로, 빈은 `ApplicationContextAware` 인터페이스를 구현하고 제공된 애플리케이션 컨텍스트를 직접 사용하여 리소스를 로드할 수도 있습니다.

그러나 일반적으로 필요한 것이 그것뿐이라면 특수화된 `ResourceLoader` 인터페이스를 사용하는 것이 좋습니다.

코드는 리소스 로딩 인터페이스(유틸리티 인터페이스로 간주될 수 있음)에만 결합되고 전체 스프링 `ApplicationContext` 인터페이스에는 결합되지 않습니다.

애플리케이션 컴포넌트에서는 `ResourceLoaderAware` 인터페이스를 구현하는 대신 `ResourceLoader`의 자동 와이어링(autowiring)에 의존할 수도 있습니다.

전통적인 `constructor` 및 `byType` 자동 와이어링 모드(협력자 자동 와이어링(Autowiring Collaborators)에서 설명됨)는 각각 생성자 인수 또는 세터 메소드 파라미터에 대해 `ResourceLoader`를 제공할 수 있습니다.

더 많은 유연성(필드 및 여러 파라미터 메소드를 자동 와이어링하는 기능 포함)을 원하면 어노테이션 기반 자동 와이어링 기능을 고려하십시오.

이 경우, 해당 필드, 생성자 또는 메소드에 `@Autowired` 어노테이션이 있는 한 `ResourceLoader`는 `ResourceLoader` 타입을 기대하는 필드, 생성자 인수 또는 메소드 파라미터에 자동 와이어링됩니다.

*와일드카드를 포함하거나 특수 `classpath*:` 리소스 접두사를 사용하는 리소스 경로에 대해 하나 이상의 `Resource` 객체를 로드하려면,*

`*ResourceLoader` 대신 `ResourcePatternResolver` 인스턴스를 애플리케이션 컴포넌트에 자동 와이어링하는 것을 고려하십시오.*

**의존성으로서의 리소스 (Resources as Dependencies)**

빈 자체가 어떤 종류의 동적 프로세스를 통해 리소스 경로를 결정하고 제공하려는 경우, 빈이 리소스를 로드하기 위해 `ResourceLoader` 또는 `ResourcePatternResolver` 인터페이스를 사용하는 것이 합리적일 수 있습니다.

예를 들어, 필요한 특정 리소스가 사용자의 역할에 따라 달라지는 어떤 종류의 템플릿 로딩을 고려해 봅시다.

리소스가 정적인 경우, `ResourceLoader` 인터페이스(또는 `ResourcePatternResolver` 인터페이스) 사용을 완전히 제거하고, 빈이 필요한 `Resource` 속성을 노출하고 주입되기를 기대하는 것이 합리적입니다.

그러면 이러한 속성을 주입하는 것을 간단하게 만드는 것은 모든 애플리케이션 컨텍스트가 `String` 경로를 `Resource` 객체로 변환할 수 있는 특수 JavaBeans `PropertyEditor`를 등록하고 사용한다는 것입니다.

예를 들어, 다음 `MyBean` 클래스는 `Resource` 타입의 `template` 속성을 가집니다.

```java
// Java
public class MyBean {

	private Resource template;

	public void setTemplate(Resource template) {
		this.template = template;
	}

	// ...
}
```

```kotlin
// Kotlin
class MyBean {
    var template: Resource? = null
        set(value) {
            field = value
        }
    // ...
}
```

XML 구성 파일에서 `template` 속성은 다음 예제와 같이 해당 리소스에 대한 간단한 문자열로 구성될 수 있습니다:

```xml
<bean id="myBean" class="example.MyBean">
	<property name="template" value="some/resource/path/myTemplate.txt"/>
</bean>
```

리소스 경로에 접두사가 없다는 점에 유의하십시오. 결과적으로, 애플리케이션 컨텍스트 자체가 `ResourceLoader`로 사용될 것이기 때문에,

리소스는 애플리케이션 컨텍스트의 정확한 타입에 따라 `ClassPathResource`, `FileSystemResource` 또는 `ServletContextResource`를 통해 로드됩니다.

특정 `Resource` 타입 사용을 강제해야 하는 경우 접두사를 사용할 수 있습니다.

다음 두 예제는 `ClassPathResource`와 `UrlResource`(후자는 파일 시스템의 파일에 접근하는 데 사용됨)를 강제하는 방법을 보여줍니다:

```xml
<property name="template" value="classpath:some/resource/path/myTemplate.txt"/>
```

```xml
<property name="template" value="file:///some/resource/path/myTemplate.txt"/>
```

`MyBean` 클래스가 어노테이션 기반 구성을 사용하도록 리팩토링되면, `myTemplate.txt` 경로는 `template.path`라는 키 아래에 저장될 수 있습니다

- 예를 들어, 스프링 환경(Environment)에서 사용 가능한 프로퍼티 파일(환경 추상화(Environment Abstraction) 참조). 그런 다음 템플릿 경로는 `@Value` 어노테이션을 사용하여 속성 플레이스홀더(Using @Value 참조)를 통해 참조될 수 있습니다.

스프링은 템플릿 경로 값을 문자열로 검색하고, 특수 `PropertyEditor`가 문자열을 `Resource` 객체로 변환하여 `MyBean` 생성자에 주입합니다. 다음 예제는 이를 달성하는 방법을 보여줍니다.

```java
// Java
@Component
public class MyBean {

	private final Resource template;

	public MyBean(@Value("${template.path}") Resource template) {
		this.template = template;
	}

	// ...
}
```

```kotlin
// Kotlin
@Component
class MyBean(
    @Value("\\${template.path}") // Use \\ escape for $
    private val template: Resource
) {
    // ...
}
```

클래스패스의 여러 위치(예: 클래스패스의 여러 jar)에서 동일한 경로 아래 발견된 여러 템플릿을 지원하려면,

특수 `classpath*:` 접두사와 와일드카드를 사용하여 `templates.path` 키를 `classpath*:/config/templates/*.txt`로 정의할 수 있습니다.

`MyBean` 클래스를 다음과 같이 재정의하면, 스프링은 템플릿 경로 패턴을 `MyBean` 생성자에 주입될 수 있는 `Resource` 객체 배열로 변환합니다.

```java
// Java
@Component
public class MyBean {

	private final Resource[] templates;

	public MyBean(@Value("${templates.path}") Resource[] templates) {
		this.templates = templates;
	}

	// ...
}
```

```kotlin
// Kotlin
@Component
class MyBean(
    @Value("\\${templates.path}") // Expects a pattern like classpath*:/config/templates/*.txt
    private val templates: Array<Resource>
) {
    // ...
}
```

**애플리케이션 컨텍스트와 리소스 경로 (Application Contexts and Resource Paths)**

이 섹션에서는 XML과 함께 작동하는 단축키, 와일드카드 사용 방법 및 기타 세부 정보를 포함하여 리소스로 애플리케이션 컨텍스트를 생성하는 방법을 다룹니다.

*애플리케이션 컨텍스트 생성하기 (Constructing Application Contexts)*
애플리케이션 컨텍스트 생성자(특정 애플리케이션 컨텍스트 타입용)는 일반적으로 컨텍스트 정의를 구성하는 XML 파일과 같은 리소스의 위치 경로인 문자열 또는 문자열 배열을 받습니다.

이러한 위치 경로에 접두사가 없는 경우, 해당 경로에서 빌드되고 빈 정의를 로드하는 데 사용되는 특정 `Resource` 타입은 특정 애플리케이션 컨텍스트에 따라 달라지며 적합합니다.

예를 들어, `ClassPathXmlApplicationContext`를 생성하는 다음 예제를 고려하십시오:

```java
// Java
ApplicationContext ctx = new ClassPathXmlApplicationContext("conf/appContext.xml");
```

```kotlin
// Kotlin
val ctx = ClassPathXmlApplicationContext("conf/appContext.xml")
```

`ClassPathResource`가 사용되기 때문에 빈 정의는 클래스패스에서 로드됩니다. 그러나 `FileSystemXmlApplicationContext`를 생성하는 경우

```java
// Java
ApplicationContext ctx =
	new FileSystemXmlApplicationContext("conf/appContext.xml");
```

```kotlin
// Kotlin
val ctx = FileSystemXmlApplicationContext("conf/appContext.xml")
```

이제 빈 정의는 파일 시스템 위치(이 경우 현재 작업 디렉토리에 상대적)에서 로드됩니다.

위치 경로에 특수 `classpath` 접두사 또는 표준 URL 접두사를 사용하면 빈 정의를 로드하기 위해 생성되는 `Resource`의 기본 타입을 오버라이드한다는 점에 유의하십시오.

```java
// Java
ApplicationContext ctx =
	new FileSystemXmlApplicationContext("classpath:conf/appContext.xml");
```

```kotlin
// Kotlin
val ctx = FileSystemXmlApplicationContext("classpath:conf/appContext.xml")
```

`FileSystemXmlApplicationContext`를 사용하면 클래스패스에서 빈 정의를 로드합니다.

그러나 여전히 `FileSystemXmlApplicationContext`입니다.

이후 `ResourceLoader`로 사용되는 경우 접두사 없는 경로는 여전히 파일 시스템 경로로 처리됩니다.

`*ClassPathXmlApplicationContext` 인스턴스 생성하기 - 단축키 (Constructing ClassPathXmlApplicationContext Instances — Shortcuts)*`ClassPathXmlApplicationContext`는 편리한 인스턴스화를 가능하게 하는 여러 생성자를 노출합니다.

기본 아이디어는 XML 파일 자체의 파일 이름만 포함하는(선행 경로 정보 없음) 문자열 배열과 `Class`를 제공하기만 하면 된다는 것입니다.

그러면 `ClassPathXmlApplicationContext`는 제공된 클래스에서 경로 정보를 파생합니다.

```
com/
  example/
    services.xml
    repositories.xml
    MessengerService.class
```

다음 예제는 `services.xml` 및 `repositories.xml`(클래스패스에 있음)이라는 이름의 파일에 정의된 빈으로 구성된 `ClassPathXmlApplicationContext` 인스턴스를 인스턴스화하는 방법을 보여줍니다:

```java
// Java
ApplicationContext ctx = new ClassPathXmlApplicationContext(
	new String[] {"services.xml", "repositories.xml"}, MessengerService.class);
```

```kotlin
// Kotlin
val ctx = ClassPathXmlApplicationContext(
    arrayOf("services.xml", "repositories.xml"), // Use arrayOf
    MessengerService::class.java // Use ::class.java
)
// Assuming MessengerService is in com.example package
```

*애플리케이션 컨텍스트 생성자 리소스 경로의 와일드카드 (Wildcards in Application Context Constructor Resource Paths)*
애플리케이션 컨텍스트 생성자 값의 리소스 경로는 (앞서 보여준 것처럼) 대상 `Resource`에 대한 일대일 매핑을 가진 단순 경로일 수도 있고,

또는 특수 `classpath*:` 접두사 또는 내부 Ant 스타일 패턴(스프링의 `PathMatcher` 유틸리티를 사용하여 매치됨)을 포함할 수도 있습니다. 후자의 두 가지 모두 효과적으로 와일드카드입니다.

이 메커니즘의 한 가지 용도는 컴포넌트 스타일 애플리케이션 어셈블리가 필요할 때입니다.

모든 컴포넌트는 잘 알려진 위치 경로에 컨텍스트 정의 조각을 게시할 수 있으며, 최종 애플리케이션 컨텍스트가 `classpath*:` 접두사가 붙은 동일한 경로를 사용하여 생성될 때 모든 컴포넌트 조각이 자동으로 선택됩니다.

이 와일드카드는 애플리케이션 컨텍스트 생성자의 리소스 경로 사용(또는 `PathMatcher` 유틸리티 클래스 계층 구조를 직접 사용할 때)에 특정하며 생성 시점에 해결된다는 점에 유의하십시오.

이는 `Resource` 타입 자체와는 아무 관련이 없습니다. 리소스는 한 번에 하나의 리소스만 가리키므로 실제 `Resource`를 생성하기 위해 `classpath*:` 접두사를 사용할 수 없습니다.

*Ant 스타일 패턴 (Ant-style Patterns)*
다음 예제와 같이 경로 위치는 Ant 스타일 패턴을 포함할 수 있습니다:

```
/WEB-INF/*-context.xml
com/mycompany/**/applicationContext.xml
file:C:/some/path/*-context.xml
classpath:com/mycompany/**/applicationContext.xml
```

경로 위치에 Ant 스타일 패턴이 포함된 경우, 해석기(resolver)는 와일드카드를 해석하기 위해 더 복잡한 절차를 따릅니다.

마지막 비-와일드카드 세그먼트까지의 경로에 대한 `Resource`를 생성하고 그로부터 URL을 얻습니다.

이 URL이 `jar:` URL 또는 컨테이너 특정 변형(예: WebLogic의 `zip:`, WebSphere의 `wsjar` 등)이 아닌 경우, 그로부터 `java.io.File`을 얻어 파일 시스템을 탐색하여 와일드카드를 해석하는 데 사용됩니다.

jar URL의 경우, 해석기는 그로부터 `java.net.JarURLConnection`을 얻거나 수동으로 jar URL을 구문 분석한 다음 jar 파일의 내용을 탐색하여 와일드카드를 해석합니다.

*이식성에 대한 영향 (Implications on Portability)*
지정된 경로가 이미 파일 URL인 경우(기본 `ResourceLoader`가 파일 시스템이거나 명시적으로), 와일드카딩은 완전히 이식 가능한 방식으로 작동함이 보장됩니다.

지정된 경로가 클래스패스 위치인 경우, 해석기는 `Classloader.getResource()` 호출을 통해 마지막 비-와일드카드 경로 세그먼트 URL을 얻어야 합니다.

이것은 경로의 노드일 뿐이므로(끝의 파일이 아님), 이 경우 어떤 종류의 URL이 반환되는지는 실제로 정의되지 않았습니다(`ClassLoader` javadoc에서).

실제로는 항상 파일 시스템 위치로 해석되는 디렉토리를 나타내는 `java.io.File`이거나, 클래스패스 리소스가 jar 위치로 해석되는 어떤 종류의 jar URL입니다. 여전히 이 작업에는 이식성 문제가 있습니다.

마지막 비-와일드카드 세그먼트에 대해 jar URL이 얻어진 경우, 해석기는 jar의 내용을 탐색하고 와일드카드를 해석할 수 있도록 그로부터 `java.net.JarURLConnection`을 얻거나 수동으로 jar URL을 구문 분석할 수 있어야 합니다.

이것은 대부분의 환경에서 작동하지만 다른 환경에서는 실패하며, jar에서 오는 리소스의 와일드카드 해석이 의존하기 전에 특정 환경에서 철저히 테스트될 것을 강력히 권장합니다.

`*classpath*:` 접두사 (The `classpath*:` Prefix)*
XML 기반 애플리케이션 컨텍스트를 생성할 때, 다음 예제와 같이 위치 문자열은 특수 `classpath*:` 접두사를 사용할 수 있습니다:

```java
// Java
ApplicationContext ctx =
	new ClassPathXmlApplicationContext("classpath*:conf/appContext.xml");
```

```kotlin
// Kotlin
val ctx = ClassPathXmlApplicationContext("classpath*:conf/appContext.xml")
```

이 특수 접두사는 주어진 이름과 일치하는 모든 클래스패스 리소스를 얻어야 함(내부적으로는 본질적으로 `ClassLoader.getResources(…)` 호출을 통해 발생함)을 지정하고, 그런 다음 병합하여 최종 애플리케이션 컨텍스트 정의를 형성합니다.

*와일드카드 클래스패스는 기본 `ClassLoader`의 `getResources()` 메소드에 의존합니다.*

*대부분의 애플리케이션 서버는 요즘 자체 `ClassLoader` 구현을 제공하므로, 특히 jar 파일을 다룰 때 동작이 다를 수 있습니다.*

`*classpath*`가 작동하는지 확인하는 간단한 테스트는 `ClassLoader`를 사용하여 클래스패스의 jar 내에서 파일을 로드하는 것입니다: `getClass().getClassLoader().getResources("<someFileInsideTheJar>")`.*

*이름은 같지만 다른 위치에 있는 파일로 이 테스트를 시도해보십시오 - 예를 들어, 이름과 경로는 같지만 클래스패스의 다른 jar에 있는 파일.*

*부적절한 결과가 반환되는 경우, `ClassLoader` 동작에 영향을 미칠 수 있는 설정에 대해 애플리케이션 서버 문서를 확인하십시오.*

위치 경로의 나머지 부분에서 `classpath*:` 접두사를 `PathMatcher` 패턴과 결합할 수도 있습니다 (예: `classpath*:META-INF/*-beans.xml`).

이 경우 해석 전략은 상당히 간단합니다: 클래스 로더 계층 구조에서 일치하는 모든 리소스를 얻기 위해 마지막 비-와일드카드 경로 세그먼트에서 `ClassLoader.getResources()` 호출이 사용된 다음,

각 리소스에서 앞서 설명한 동일한 `PathMatcher` 해석 전략이 와일드카드 하위 경로에 사용됩니다.

*와일드카드 관련 기타 참고 사항 (Other Notes Relating to Wildcards)*`classpath*:`는 Ant 스타일 패턴과 결합될 때, 실제 대상 파일이 파일 시스템에 있지 않는 한,

패턴이 시작되기 전에 적어도 하나의 루트 디렉토리가 있어야만 안정적으로 작동한다는 점에 유의하십시오.

이는 즉, `classpath*:*.xml`과 같은 패턴은 jar 파일의 루트에서 파일을 검색하지 못하고 확장된 디렉토리의 루트에서만 검색할 수 있음을 의미합니다.

*스프링의 클래스패스 항목 검색 기능은 JDK의 `ClassLoader.getResources()` 메소드에서 비롯되며, 이 메소드는 빈 문자열(검색할 잠재적 루트 표시)에 대해서만 파일 시스템 위치를 반환합니다.*

*스프링은 `URLClassLoader` 런타임 구성과 jar 파일의 `java.class.path` 매니페스트도 평가하지만, 이것이 이식 가능한 동작으로 이어질 것이라고 보장되지는 않습니다.*

*클래스패스 패키지 스캔은 클래스패스에 해당 디렉토리 항목이 있어야 합니다. Ant로 JAR을 빌드할 때 JAR 태스크의 `files-only` 스위치를 활성화하지 마십시오.*

*또한 일부 환경의 보안 정책에 따라 클래스패스 디렉토리가 노출되지 않을 수 있습니다*

*모듈 경로(Java Module System)에서 스프링의 클래스패스 스캔은 일반적으로 예상대로 작동합니다.*

*여기에 리소스를 전용 디렉토리에 넣는 것이 매우 권장되며, 이는 jar 파일 루트 레벨 검색과 관련된 앞서 언급한 이식성 문제를 피합니다.*

`classpath:` 리소스가 있는 Ant 스타일 패턴은 검색할 루트 패키지가 여러 클래스패스 위치에서 사용 가능한 경우 일치하는 리소스를 찾는 것을 보장하지 않습니다.

다음 리소스 위치를 참고하세요:`com/mycompany/package1/service-context.xml`

이제 누군가가 해당 파일을 찾기 위해 사용할 수 있는 Ant 스타일 경로를 고려하십시오:

`classpath:com/mycompany/**/service-context.xml`

이러한 리소스는 클래스패스의 한 위치에만 존재할 수 있지만, 앞의 예제와 같은 경로를 사용하여 해석하려고 시도하면 해석기는 `getResource("com/mycompany");`가 반환하는 (첫 번째) URL에서 작동합니다.

이 기본 패키지 노드가 여러 `ClassLoader` 위치에 존재하는 경우 원하는 리소스가 발견된 첫 번째 위치에 존재하지 않을 수 있습니다.

따라서 이러한 경우, `com/mycompany` 기본 패키지를 포함하는 모든 클래스패스 위치를 검색하는 동일한 Ant 스타일 패턴과 함께 `classpath*:`를 사용하는 것을 선호해야 합니다: `classpath*:com/mycompany/**/service-context.xml`.

`*FileSystemResource` 주의사항 (FileSystemResource Caveats)*`FileSystemApplicationContext`에 연결되지 않은 `FileSystemResource`

(즉, `FileSystemApplicationContext`가 실제 `ResourceLoader`가 아닐 때)는 절대 경로와 상대 경로를 예상대로 처리합니다.

상대 경로는 현재 작업 디렉토리에 상대적이며, 절대 경로는 파일 시스템의 루트에 상대적입니다.

그러나 하위 호환성(역사적) 이유로, `FileSystemApplicationContext`가 `ResourceLoader`일 때 이것이 변경됩니다.

`FileSystemApplicationContext`는 연결된 모든 `FileSystemResource` 인스턴스가 선행 슬래시(`/`)로 시작하는지 여부에 관계없이 모든 위치 경로를 상대적으로 처리하도록 강제합니다.

실제로 이는 다음 예제가 동일하다는 것을 의미합니다:

```java
// Java
ApplicationContext ctx =
	new FileSystemXmlApplicationContext("conf/context.xml");
```

```kotlin
// Kotlin
val ctx1 = FileSystemXmlApplicationContext("conf/context.xml")
```

```java
// Java
ApplicationContext ctx =
	new FileSystemXmlApplicationContext("/conf/context.xml");
```

```kotlin
// Kotlin
val ctx2 = FileSystemXmlApplicationContext("/conf/context.xml") // Treated as relative within FileSystemXmlApplicationContext
```

다음 예제도 동일합니다 (한 경우는 상대적이고 다른 경우는 절대적이므로 다를 것이 합리적이지만):

```java
// Java
FileSystemXmlApplicationContext ctx = ...;
ctx.getResource("some/resource/path/myTemplate.txt");
```

```kotlin
// Kotlin
val ctx: FileSystemXmlApplicationContext = ...
val resource1 = ctx.getResource("some/resource/path/myTemplate.txt") // Relative to working directory
```

```java
// Java
FileSystemXmlApplicationContext ctx = ...;
ctx.getResource("/some/resource/path/myTemplate.txt");
```

```kotlin
// Kotlin
val ctx: FileSystemXmlApplicationContext = ...
val resource2 = ctx.getResource("/some/resource/path/myTemplate.txt") // Also treated as relative
```

실제로 진정한 절대 파일 시스템 경로가 필요한 경우, `FileSystemResource` 또는 `FileSystemXmlApplicationContext`와 함께 절대 경로 사용을 피하고 `file:` URL 접두사를 사용하여 `UrlResource` 사용을 강제해야 합니다.

다음 예제는 이를 수행하는 방법을 보여줍니다:

```java
// Java
// 실제 컨텍스트 타입은 중요하지 않음, Resource는 항상 UrlResource가 됨
ctx.getResource("file:///some/resource/path/myTemplate.txt");
```

```kotlin
// Kotlin
// 실제 컨텍스트 타입은 중요하지 않음, Resource는 항상 UrlResource가 됨
val resource = ctx.getResource("file:///some/resource/path/myTemplate.txt")
```

```java
// Java
// 이 FileSystemXmlApplicationContext가 UrlResource를 통해 정의를 로드하도록 강제
ApplicationContext ctx =
	new FileSystemXmlApplicationContext("file:///conf/context.xml");
```

```kotlin
// Kotlin
// 이 FileSystemXmlApplicationContext가 UrlResource를 통해 정의를 로드하도록 강제
val ctx = FileSystemXmlApplicationContext("file:///conf/context.xml")
```

---

**전체 주제: 리소스 (Resources)**

이 장은 스프링이 **파일 시스템, 클래스패스, URL 등 다양한 위치에 있는 외부 자원(리소스)에 일관되고 유연하게 접근**하기 위해 사용하는 **추상화 메커니즘**과 관련 인터페이스 및 구현체들을 상세하게 설명합니다.

**핵심 아이디어:** 자바의 기본 리소스 접근 방식의 한계를 극복하고, 어떤 위치의 리소스든 간편하고 통일된 방식으로 접근할 수 있도록 스프링이 제공하는 강력한 `Resource` 추상화에 대해 알아보자.

---

**첫 번째 파트: 소개 (Introduction)**

- **문제 제기:** 자바의 표준 `java.net.URL` 클래스는 유용하지만, 모든 종류의 저수준 리소스(특히 클래스패스나 서블릿 컨텍스트 내부 리소스)에 접근하기에는 **부족하거나 표준화된 방법이 없다**는 한계가 있습니다. 새로운 URL 프로토콜 핸들러를 등록하는 것은 복잡하고, URL 인터페이스 자체에도 리소스 존재 여부 확인 같은 기본적인 기능이 부족합니다.
- **스프링의 목표:** 이러한 자바 표준의 한계를 극복하고, **더 강력하고 유연하며 일관된 리소스 접근 방법**을 제공하기 위해 스프링은 자체적인 `Resource` 추상화를 만들었습니다.

---

**두 번째 파트: `Resource` 인터페이스 (The Resource Interface)**

이 부분은 스프링 리소스 추상화의 **핵심**인 `org.springframework.core.io.Resource` 인터페이스를 상세히 설명합니다.

- **목적:** 저수준 리소스에 대한 접근을 **추상화**하는 더 유능한 인터페이스입니다.
- **주요 메소드 (복습 및 추가):**
  - `getInputStream()`: 리소스 내용을 읽기 위한 새로운 `InputStream` 반환 (호출자가 닫아야 함).
  - `exists()`: 리소스가 실제로 존재하는지 확인.
  - `isReadable()`: 리소스 내용을 읽을 수 있는지 확인.
  - `isOpen()`: 리소스가 이미 열려 있는 스트림을 나타내는지 확인 (주로 `InputStreamResource`만 `true`). `true`이면 스트림은 한 번만 읽고 닫아야 함.
  - `isFile()`: 리소스가 파일 시스템의 파일인지 확인.
  - `getURL()`: 리소스의 `java.net.URL` 객체 반환 (가능한 경우).
  - `getURI()`: 리소스의 `java.net.URI` 객체 반환 (가능한 경우).
  - `getFile()`: 리소스의 `java.io.File` 객체 반환 (파일 시스템 리소스인 경우). 클래스패스 리소스 등은 파일 객체로 직접 접근 불가능할 수 있으므로 예외 발생 가능.
  - `readableChannel()`: NIO 기반의 `ReadableByteChannel` 반환.
  - `contentLength()`: 리소스의 크기 (바이트 단위) 반환.
  - `lastModified()`: 마지막 수정 시간 반환.
  - `createRelative(String relativePath)`: 현재 리소스를 기준으로 상대 경로의 새로운 `Resource` 객체 생성.
  - `getFilename()`: 리소스의 파일 이름 반환.
  - `getDescription()`: 리소스에 대한 설명 문자열 반환 (오류 메시지 등에 유용).
- **확장 인터페이스:**
  - `InputStreamSource`: `Resource`의 부모 인터페이스로, `getInputStream()` 메소드만 정의.
  - `WritableResource`: `Resource`를 확장하며, 리소스에 내용을 쓸 수 있는 기능(`getOutputStream()` 등)을 추가로 제공하는 구현체들이 구현.
- **스프링에서의 활용:** 스프링 내부 API 곳곳에서 리소스가 필요한 경우 파라미터 타입으로 `Resource` 인터페이스를 광범위하게 사용합니다. `ApplicationContext` 생성자 등에서는 리소스 경로 문자열을 받아 내부적으로 적절한 `Resource` 객체를 생성하여 사용합니다.
- **독립적 사용:** `Resource` 인터페이스와 관련 구현체들은 스프링의 다른 부분과 독립적으로도 **일반적인 리소스 접근 유틸리티**로 매우 유용하게 사용할 수 있습니다.

---

**세 번째 파트: 내장 `Resource` 구현체 (Built-in Resource Implementations)**

스프링은 다양한 종류의 리소스를 처리하기 위해 미리 정의된 여러 `Resource` 구현 클래스를 제공합니다.

- **`UrlResource`:**
  - `java.net.URL` 객체를 래핑합니다.
  - `http:`, `https:`, `ftp:`, `file:` 등 표준 URL 프로토콜을 사용하여 접근 가능한 모든 리소스를 다룹니다.
  - 직접 생성하거나, 접두사가 없는 경로 문자열이 주어질 때 스프링이 기본적으로 생성할 수 있습니다.
- **`ClassPathResource`:**
  - **클래스패스** 상의 리소스를 나타냅니다. (가장 많이 사용됨)
  - 클래스 로더를 사용하여 리소스를 찾습니다.
  - 리소스가 파일 시스템에 실제 파일로 존재하면 `getFile()`을 지원하지만, JAR 파일 내부에 있다면 지원하지 않습니다. (`getURL()`, `getInputStream()`은 항상 가능)
  - `classpath:` 접두사가 붙은 경로 문자열이 주어지면 스프링이 자동으로 생성합니다.
- **`FileSystemResource`:**
  - `java.io.File` 또는 `java.nio.file.Path` 핸들을 사용하여 **파일 시스템** 상의 리소스를 나타냅니다.
  - `getFile()`, `getURL()`을 지원하며 `WritableResource`도 구현합니다.
- **`PathResource`:**
  - `java.nio.file.Path` 핸들을 사용하여 파일 시스템 리소스에 접근하며, 모든 작업을 **NIO Path API**를 통해 수행합니다. (`FileSystemResource`의 NIO 기반 대안)
  - `WritableResource`를 구현합니다.
- **`ServletContextResource`:**
  - **웹 애플리케이션의 루트 디렉토리**를 기준으로 리소스를 찾습니다. (`ServletContext` 필요, 웹 환경 전용)
  - 웹 애플리케이션 아카이브(WAR)가 압축 해제되어 파일 시스템에 있을 때만 `getFile()`을 지원합니다.
- **`InputStreamResource`:**
  - 이미 열려 있는 **`InputStream` 객체 자체**를 `Resource`로 래핑합니다.
  - 다른 `Resource` 구현체를 사용할 수 없을 때 임시로 사용합니다.
  - `isOpen()`이 `true`를 반환하며, 스트림은 한 번만 읽고 닫아야 합니다. 재사용이 불가능합니다.
- **`ByteArrayResource`:**
  - 주어진 **바이트 배열(`byte[]`)** 을 `Resource`로 래핑합니다.
  - 내부적으로 `ByteArrayInputStream`을 생성하여 내용을 제공합니다.
  - 메모리에 있는 바이트 데이터를 리소스로 다루고 싶을 때 유용합니다.

---

**네 번째 파트: `ResourceLoader` 인터페이스 (The ResourceLoader Interface)**

- **역할 (복습):** 리소스 **위치 문자열(location)** 을 받아 해당하는 **`Resource` 객체를 반환(로드)** 하는 역할을 하는 인터페이스입니다.

    ```java
    public interface ResourceLoader {
        Resource getResource(String location); // 핵심 메소드
        ClassLoader getClassLoader(); // 연관된 클래스 로더 반환
    }
    ```

- **`ApplicationContext`와의 관계:** 모든 `ApplicationContext` 구현체는 `ResourceLoader`를 구현합니다. 따라서 `ApplicationContext` 객체를 통해 `getResource()` 메소드를 호출할 수 있습니다.
- **컨텍스트별 기본 리소스 타입:** `getResource()` 호출 시 위치 문자열에 **접두사가 없으면**, 반환되는 `Resource`의 구체적인 타입은 `ApplicationContext`의 종류에 따라 달라집니다.
  - `ClassPathXmlApplicationContext` -> `ClassPathResource` 반환
  - `FileSystemXmlApplicationContext` -> `FileSystemResource` 반환
  - `WebApplicationContext` -> `ServletContextResource` 반환
- **접두사 사용:** `classpath:`, `file:`, `https:` 등의 접두사를 사용하면 `ApplicationContext` 타입과 관계없이 **명시적으로 특정 타입의 `Resource`** (예: `ClassPathResource`, `UrlResource`)를 사용하도록 강제할 수 있습니다.

---

**다섯 번째 파트: `ResourcePatternResolver` 인터페이스 (The ResourcePatternResolver Interface)**

- **역할:** `ResourceLoader` 인터페이스를 **확장**하여, **패턴(와일드카드)** 이 포함된 위치 문자열을 받아 **여러 개의 `Resource` 객체 배열**로 해석(resolve)하는 기능을 추가한 인터페이스입니다.

    ```java
    public interface ResourcePatternResolver extends ResourceLoader {
        String CLASSPATH_ALL_URL_PREFIX = "classpath*:"; // 특별 접두사 정의
        Resource[] getResources(String locationPattern) throws IOException; // 핵심 메소드
    }
    ```

- **`classpath*:` 접두사:** 이 특별한 접두사는 클래스패스 상의 **모든 위치**에서 해당 패턴과 일치하는 리소스를 찾으라는 의미입니다. (예: 여러 JAR 파일에 같은 경로/이름의 파일이 있을 때 모두 찾음)
- **Ant 스타일 패턴:** `/WEB-INF/*-context.xml`, `com/mycompany/**/applicationContext.xml` 처럼  (하나의 경로 세그먼트 내 임의 문자), `*` (여러 경로 세그먼트) 와일드카드를 사용할 수 있습니다. 스프링 내부의 `AntPathMatcher` 유틸리티를 사용하여 패턴을 매칭합니다.
- **구현체:** `PathMatchingResourcePatternResolver`가 대표적인 독립 실행형 구현체입니다.
- **`ApplicationContext`와의 관계:** 모든 표준 `ApplicationContext`는 내부적으로 `PathMatchingResourcePatternResolver`를 사용하며, `ApplicationContext` 객체 자체가 `ResourcePatternResolver` 인터페이스를 구현하므로, `context.getResources("패턴")` 처럼 직접 사용할 수 있습니다.

---

**여섯 번째 파트: `ResourceLoaderAware` 인터페이스 (The ResourceLoaderAware Interface)**

- **역할 (복습):** 빈이 자신이 속한 `ApplicationContext`(즉, `ResourceLoader`)에 대한 참조를 **주입**받고 싶을 때 구현하는 콜백 인터페이스입니다.

    ```java
    public interface ResourceLoaderAware {
        void setResourceLoader(ResourceLoader resourceLoader);
    }
    ```

- **동작:** 빈이 이 인터페이스를 구현하면, 스프링 컨테이너가 `setResourceLoader()` 메소드를 호출하여 `ApplicationContext` 자신을 전달해줍니다.
- **대안:** `ApplicationContextAware`를 사용하거나 (하지만 `ResourceLoader`만 필요하다면 이것이 더 좋음), `@Autowired`를 사용하여 `ResourceLoader` 또는 `ResourcePatternResolver`를 직접 주입받는 것이 더 선호됩니다.

---

**일곱 번째 파트: 의존성으로서의 리소스 (Resources as Dependencies)**

이 부분은 빈의 **속성(property)** 으로 `Resource` 타입을 직접 사용하여 의존성을 주입하는 가장 편리한 방법을 설명합니다.

- **정적인 리소스:** 빈이 필요로 하는 리소스의 경로가 고정되어 있다면, `ResourceLoader`를 직접 사용하는 대신 빈의 **멤버 변수 타입을 `Resource`** 로 선언하고 설정(XML, Java Config, `@Value`)에서 **리소스 경로 문자열**을 값으로 주입하는 것이 가장 좋습니다.
- **자동 변환:** 스프링 컨테이너는 빈 속성을 설정할 때, 주입되는 값이 **문자열**이고 대상 속성 타입이 **`Resource`** 이면, 내부적으로 `PropertyEditor`를 사용하여 **문자열 경로를 자동으로 적절한 `Resource` 객체로 변환**하여 주입해줍니다. 접두사가 없으면 컨텍스트에 따라, 접두사가 있으면 해당 타입으로 변환됩니다.

    ```java
    // 빈 클래스
    public class MyBean {
        private Resource template;
        public void setTemplate(Resource template) { this.template = template; }
    }
    
    // XML 설정
    <bean id="myBean" class="example.MyBean">
        <property name="template" value="classpath:some/resource/path/myTemplate.txt"/>
    </bean>
    
    // Java Config + @Value
    @Component
    public class MyBean {
        private final Resource template;
        public MyBean(@Value("${template.path}") Resource template) { // @Value로 경로 주입 -> Resource 변환
            this.template = template;
        }
    }
    ```

- **리소스 배열 주입:** 와일드카드나 `classpath*:` 패턴이 포함된 경로 문자열을 `Resource[]` (리소스 배열) 타입의 속성에 주입하면, 스프링은 패턴과 일치하는 **모든 `Resource` 객체들을 배열로 만들어 주입**해줍니다.

---

**여덟 번째 파트: 애플리케이션 컨텍스트와 리소스 경로 (Application Contexts and Resource Paths)**

이 부분은 `ApplicationContext` 생성 시 리소스 경로를 어떻게 지정하고, 와일드카드 사용 시 주의할 점 등을 상세히 설명합니다.

- **컨텍스트 생성자:** `ClassPathXmlApplicationContext`, `FileSystemXmlApplicationContext` 등의 생성자는 리소스 경로 문자열(또는 배열)을 인수로 받습니다.
- **기본 경로 해석 (복습):** 접두사가 없으면 컨텍스트 타입에 따라 기본 경로(클래스패스, 파일 시스템)가 결정됩니다. 접두사(`classpath:`, `file:`)를 사용하면 기본 동작을 오버라이드할 수 있습니다.
- **`ClassPathXmlApplicationContext` 단축키:** 생성자에 클래스 객체를 함께 전달하면, XML 파일 이름을 해당 클래스와 같은 패키지 경로에서 찾도록 경로 정보를 자동으로 완성해주는 편의 기능을 제공합니다.
- **와일드카드 (, `*`) 및 `classpath*:`:**
  - 컨텍스트 생성자의 경로 문자열에도 Ant 스타일 패턴과 `classpath*:` 접두사를 사용할 수 있습니다.
  - 이를 통해 여러 위치에 흩어져 있는 설정 파일 조각들을 자동으로 모아서 컨텍스트를 구성할 수 있습니다 (컴포넌트 기반 조립).
  - **주의:** 와일드카드나 `classpath*:`의 동작 방식은 `ClassLoader` 구현이나 실행 환경(파일 시스템 vs JAR 파일)에 따라 약간의 차이나 제약이 있을 수 있으므로, 특히 JAR 파일 내 리소스를 다룰 때는 주의하고 테스트해야 합니다. (예: `classpath*:*.xml`은 JAR 루트의 파일을 못 찾을 수 있음)
  - 패턴 사용 시 루트 패키지가 여러 클래스패스 위치에 존재하면 `classpath:` 대신 `classpath*:`를 사용하여 모든 위치를 검색하도록 하는 것이 안전합니다.
- **`FileSystemResource` 주의사항:** `FileSystemXmlApplicationContext` 내에서 사용될 때, 하위 호환성 때문에 접두사 없는 경로는 항상 **현재 작업 디렉토리 기준의 상대 경로**로 처리됩니다 (경로 앞에 `/`가 있더라도). 진정한 절대 경로를 사용하려면 반드시 `file:` 접두사를 붙여 `UrlResource`를 사용하도록 강제해야 합니다.

**요약 (전체):**

스프링의 `Resource` 추상화는 파일 시스템, 클래스패스, URL 등 다양한 위치의 리소스에 **일관되고 강력한 방식**으로 접근할 수 있게 해줍니다. `ResourceLoader`(주로 `ApplicationContext`)가 위치 문자열을 해석하여 `Resource` 객체를 로드하며, `ResourcePatternResolver`는 와일드카드 패턴을 처리합니다. 빈에서는 `Resource` 타입 속성에 경로 문자열을 직접 주입하는 것이 가장 편리하며, 스프링이 자동으로 변환해줍니다. 컨텍스트 생성 시에도 경로 문자열과 패턴을 유연하게 사용하여 설정을 로드할 수 있습니다.

---

**1. IO vs NIO (java.io vs java.nio)**

- **IO (Input/Output):**
  - `java.io` 패키지에 해당합니다.
  - **스트림 기반(Stream-based):** 데이터를 **단방향(입력 또는 출력)** 으로 흐르는 **바이트(byte) 또는 문자(char)의 연속적인 흐름(stream)** 으로 다룹니다. 한 번 읽거나 쓰면 되돌릴 수 없습니다.
  - **블로킹(Blocking) 방식:** 기본적으로 데이터 읽기/쓰기 작업이 완료될 때까지 해당 스레드가 **멈춰서 기다리는(blocking)** 방식입니다. 스레드가 대기하는 동안 다른 작업을 할 수 없어 효율성이 떨어질 수 있습니다.
  - **버퍼 사용 제한적:** 직접 버퍼를 관리하거나 특정 스트림 클래스(예: `BufferedInputStream`)를 사용해야 효율적인 처리가 가능합니다.
  - **간단함:** 사용법이 비교적 간단하고 직관적입니다. 파일 처리 등 간단한 입출력에 많이 사용됩니다.
- **NIO (New IO / Non-blocking IO):**
  - `java.nio` 패키지에 해당합니다. JDK 1.4부터 도입되었습니다.
  - **채널 기반(Channel-based) & 버퍼 기반(Buffer-based):** 데이터는 **채널(Channel)** 이라는 양방향 통로를 통해 **버퍼(Buffer)** 라는 메모리 공간으로 읽어들여 처리합니다. 버퍼 내에서 데이터를 자유롭게 이동하며 읽고 쓸 수 있습니다.
  - **논블로킹(Non-blocking) 지원:** 하나의 스레드가 여러 개의 채널에 대한 입출력 작업을 **기다리지 않고(non-blocking)** 동시에 처리할 수 있는 **셀렉터(Selector)** 기반의 메커니즘을 제공합니다. 요청한 작업이 준비되었는지 확인하고 준비된 작업만 처리하여 스레드 효율성을 높일 수 있습니다. (네트워크 프로그래밍 등에서 매우 중요)
  - **버퍼 중심:** 데이터를 항상 버퍼 단위로 읽고 쓰므로 효율적인 데이터 처리가 가능합니다.
  - **복잡함:** IO보다 개념(채널, 버퍼, 셀렉터)이 많고 사용법이 더 복잡합니다. 대용량 파일 처리나 고성능 네트워크 통신에 주로 사용됩니다.
- **`PathResource`와 NIO:** 스프링의 `PathResource`는 파일 시스템 접근 시 `java.nio.file.Path`와 NIO API를 사용하므로, 더 현대적이고 성능 이점이 있을 수 있는 방식으로 파일을 다룹니다.

**2. PropertyEditor는 무엇인가?**

- **개념:** `java.beans.PropertyEditor`는 **문자열(String) 형태의 텍스트 값**을 **다른 자바 객체 타입**으로 변환하거나, 반대로 자바 객체를 문자열로 변환하는 방법을 정의하는 **자바 표준 인터페이스**입니다.
- **스프링에서의 역할:** 스프링은 빈의 속성(property) 값을 설정할 때 (XML의 `value` 속성, 프로퍼티 파일의 값, `@Value` 어노테이션의 문자열 등) 이 `PropertyEditor` 메커니즘을 광범위하게 사용합니다.
  - **예시:** XML에 `<property name="port" value="8080"/>` 이라고 썼을 때, `port` 속성의 타입이 `int`라면, 스프링은 내부적으로 `String` "8080"을 `int` 8080으로 변환해주는 `PropertyEditor`(예: `CustomNumberEditor`)를 사용합니다.
  - **`Resource` 변환:** 마찬가지로, `<property name="template" value="classpath:my.txt"/>` 나 `@Value("classpath:my.txt")` 처럼 **문자열 경로**를 `Resource` 타입의 속성에 주입할 때, 스프링은 **`ResourceEditor`** 라는 특별한 `PropertyEditor`를 사용하여 해당 문자열 경로를 적절한 `Resource` 객체(예: `ClassPathResource`)로 **자동 변환**해주는 것입니다.
- **핵심:** "문자열과 특정 자바 타입 간의 **변환기** 역할을 하는 표준 인터페이스"이며, 스프링은 이를 활용하여 설정 파일의 텍스트 값을 실제 객체 타입으로 자동 변환해줍니다.

**3. UrlResource 프로토콜별 동작 (`http:`, `file:`)**

`UrlResource`는 내부적으로 `java.net.URL` 객체를 사용합니다. 각 프로토콜 접두사는 `URL` 객체가 해당 리소스를 어떻게 찾고 접근할지를 결정합니다.

- **`http:` / `https:`:**
  - **형식:** `http://호스트명[:포트]/경로/파일명`, `https://호스트명[:포트]/경로/파일명`
  - **동작:** 지정된 URL 주소로 **HTTP 또는 HTTPS 네트워크 요청**을 보내서 웹 서버로부터 해당 리소스(HTML 페이지, 이미지, JSON 데이터 등)의 내용을 **다운로드** 받으려고 시도합니다. `getInputStream()`을 호출하면 웹 서버와의 연결이 열리고 데이터 스트림을 받게 됩니다.
  - **사용 예:** 웹상의 설정 파일이나 데이터를 직접 읽어올 때 사용합니다.

      ```java
      Resource webResource = ctx.getResource("<https://example.com/config/settings.json>");
      // webResource.getInputStream() 호출 시 네트워크 통신 발생
      ```

- **`file:`:**
  - **형식:**
    - `file:/절대/경로/파일명` (Unix/Linux 계열)
    - `file:///C:/절대/경로/파일명` (Windows, 슬래시 3개 주의)
    - `file:상대/경로/파일명` (현재 작업 디렉토리 기준 상대 경로)
  - **동작:** 로컬 **파일 시스템**에서 지정된 경로의 파일을 직접 찾아서 접근합니다. `getInputStream()`을 호출하면 해당 파일에 대한 `FileInputStream`이 열립니다. `getFile()` 메소드를 사용하여 `java.io.File` 객체를 얻을 수도 있습니다.
  - **사용 예:** 파일 시스템의 특정 위치에 있는 설정 파일이나 데이터 파일을 읽을 때 사용합니다.

      ```java
      // 절대 경로 (Linux/Mac)
      Resource fileResource1 = ctx.getResource("file:/home/user/data.csv");
      // 절대 경로 (Windows)
      Resource fileResource2 = ctx.getResource("file:///C:/apps/config.xml");
      // 상대 경로 (현재 실행 위치 기준)
      Resource fileResource3 = ctx.getResource("file:input/mydata.txt");
      ```


**4. 왜 이렇게 `ApplicationContext` 종류가 많은가? (용도와 필요성)**

각 `ApplicationContext` 구현체는 **어떤 종류의 환경**에서 실행되고, **어떤 형식의 설정 정보를 기본으로 사용하는지**에 따라 특화되어 있습니다. 다양한 사용 환경과 설정 방식을 지원하기 위해 여러 종류가 존재하는 것입니다.

- **`ClassPathXmlApplicationContext`:**
  - **용도:** **클래스패스(Classpath)** 에 있는 **XML 설정 파일**을 로드하여 컨테이너를 생성합니다.
  - **주 사용 환경:** 일반적인 독립 실행형 자바 애플리케이션, 테스트 코드 등 클래스패스 기반으로 설정을 관리하는 환경.
  - **기본 리소스 경로:** 접두사 없는 경로는 클래스패스로 간주.
- **`FileSystemXmlApplicationContext`:**
  - **용도:** **파일 시스템** 상의 특정 경로에 있는 **XML 설정 파일**을 로드하여 컨테이너를 생성합니다.
  - **주 사용 환경:** 파일 시스템의 특정 디렉토리에 설정 파일을 두고 관리하는 환경.
  - **기본 리소스 경로:** 접두사 없는 경로는 파일 시스템 경로로 간주 (단, 컨텍스트 로더로 사용될 때는 상대 경로처럼 동작하는 주의사항 있음).
- **`AnnotationConfigApplicationContext`:**
  - **용도:** **`@Configuration` 어노테이션이 붙은 자바 클래스**나 **`@Component` 스캔**을 통해 설정을 로드하여 컨테이너를 생성합니다. **XML 없는 설정**을 위한 핵심 구현체입니다.
  - **주 사용 환경:** Java Config 기반 또는 어노테이션 기반의 스프링 애플리케이션 (현대적인 방식).
  - **기본 리소스 경로:** 경로 해석 방식은 덜 중요하며, 주로 클래스 기반 설정에 집중합니다. (`getResource` 호출 시 기본 동작은 `ClassPathXmlApplicationContext`와 유사할 수 있음).
- **`WebApplicationContext` (인터페이스):**
  - **용도:** **웹 애플리케이션 환경**에 특화된 기능을 추가로 제공하는 `ApplicationContext` 인터페이스입니다. (`ServletContext` 접근 기능 등)
  - **구현체:**
    - `XmlWebApplicationContext`: `web.xml`에서 **XML 설정 파일**을 로드하는 기본 웹 컨텍스트.
    - `AnnotationConfigWebApplicationContext`: `web.xml`에서 **Java Config 클래스**나 스캔할 **패키지**를 로드하는 웹 컨텍스트.
  - **주 사용 환경:** 서블릿 컨테이너(Tomcat 등) 위에서 실행되는 웹 애플리케이션. `ContextLoaderListener`나 `DispatcherServlet`에 의해 생성됩니다.
  - **기본 리소스 경로:** 접두사 없는 경로는 웹 애플리케이션 루트(ServletContext 기준)로 간주.

**결론 (Context 종류):** 각 `ApplicationContext`는 **설정 정보를 어디서(클래스패스, 파일 시스템, 자바 클래스) 어떤 형식으로(XML, Java/Annotation) 읽어올 것인가** 와 **어떤 실행 환경(일반 앱, 웹 앱)에 최적화되어 있는가** 에 따라 나뉩니다. 개발자는 자신의 프로젝트 환경과 설정 방식에 맞는 구현체를 선택하여 사용하면 됩니다.

**5. Ant 스타일 패턴이란?**

- **개념:** 파일 경로를 지정할 때 사용하는 **와일드카드(Wildcard) 패턴 규칙** 중 하나입니다. 원래는 자바 빌드 도구인 **Ant**에서 사용되던 경로 매칭 방식인데, 스프링을 포함한 많은 곳에서 널리 사용됩니다.
- **주요 와일드카드:**
  - `?`: 한 글자와 일치합니다. (예: `myfile?.txt`는 `myfile1.txt`, `myfileA.txt` 등과 일치)
  - : **하나의 경로 세그먼트(디렉토리 또는 파일명)** 내에서 0개 이상의 글자와 일치합니다. (슬래시(`/`)는 포함하지 않음)
    - 예: `com/t?st.jsp` -> `com/test.jsp`, `com/tast.jsp` 등
    - 예: `com/*.jsp` -> `com` 디렉토리 바로 아래의 모든 `.jsp` 파일
  - `*`: **0개 이상의 디렉토리**와 일치합니다. (가장 강력하고 많이 사용됨)
    - 예: `com/**/test.jsp` -> `com` 디렉토리 아래의 모든 하위 디렉토리(없어도 됨)에 있는 `test.jsp` 파일 (예: `com/test.jsp`, `com/foo/test.jsp`, `com/foo/bar/test.jsp`)
    - 예: `classpath*:/**/*.xml` -> 클래스패스 상의 모든 위치에서 모든 하위 디렉토리에 있는 `.xml` 파일
- **스프링에서의 사용:** 리소스 경로 지정(`ResourcePatternResolver`), 컴포넌트 스캔의 패키지 지정, AOP 포인트컷 표현식 등 다양한 곳에서 이 Ant 스타일 패턴을 사용하여 여러 개의 대상(파일, 클래스, 메소드)을 편리하게 지정할 수 있습니다.

**6. 컨텍스트 생성 과정에 대한 혼란 해소**

- **프로세스:** 스프링 애플리케이션 실행 시 **가장 먼저 해야 할 일** 중 하나가 바로 **`ApplicationContext` (스프링 컨테이너)를 생성하고 초기화하는 것**입니다. 이 과정에서 스프링은 설정 정보(XML, Java Config, 스캔 등)를 읽어 빈 정의를 등록하고, 빈 객체들을 생성하며 의존성을 주입합니다.
- **생성 시점:**
  - **독립 실행형 앱:** 개발자가 `main` 메소드 등에서 **직접 `new ...ApplicationContext(...)` 코드를 호출**하여 컨텍스트를 생성합니다.
  - **웹 앱 (전통 방식):** `web.xml`에 `ContextLoaderListener`를 등록하면, **웹 서버가 앱을 시작할 때 이 리스너를 통해 자동으로** 컨텍스트를 생성합니다.
  - **스프링 부트:** `SpringApplication.run(...)`을 호출하면 **스프링 부트가 내부적으로 자동으로** 적절한 컨텍스트를 생성합니다.
- **결론:** 이전 파트에서 다양한 컨텍스트를 사용한 것은 **"이런 종류의 컨텍스트는 이런 식으로 설정하고 사용할 수 있다"** 는 예시를 보여준 것이고, 마지막 파트("애플리케이션 컨텍스트 생성하기")는 **"그래서 실제로 이 컨텍스트 객체를 언제, 어떻게 만드는가?"** 에 대한 **구체적인 방법**들을 설명한 것입니다. 즉, 컨텍스트의 종류와 특징을 먼저 배우고, 그것을 실제로 만드는 방법을 나중에 배운 흐름입니다. 컨텍스트는 항상 어딘가에서 **생성되는 시점**이 반드시 있습니다.

**7. `@Component`와 `@Value("${template.path}")`의 관계**

```java
// Java
@Component // ① 이 클래스를 스프링 빈으로 등록하도록 함
public class MyBean {
    private final Resource template;

    // ② 생성자 파라미터에 @Value 사용
    public MyBean(@Value("${template.path}") Resource template) {
        this.template = template;
    }
}
```

- **`@Component` (①):** 이 어노테이션은 클래스패스 스캐닝(`@ComponentScan`)이 활성화되었을 때, 스프링 컨테이너에게 **"이 `MyBean` 클래스를 찾아서 빈으로 등록하고 관리해줘!"** 라고 알려줍니다. 즉, 스프링이 `MyBean` 객체를 생성하고 생명주기를 관리하게 됩니다.
- **`@Value("${template.path}")` (②):** 이 어노테이션은 `MyBean`의 **생성자 파라미터** `template`에 값을 주입하라는 지시입니다.
  - `"${template.path}"`: 스프링 `Environment`에서 `"template.path"`라는 키를 가진 프로퍼티 값을 찾으라는 의미입니다. (이 값은 `application.properties` 등에 정의되어 있어야 합니다. 예: `template.path=classpath:myTemplate.txt`)
  - `Resource template`: 주입받을 파라미터의 타입이 `Resource`입니다.
- **자동 주입 과정:**
  1. 스프링 컨테이너는 `@Component`를 보고 `MyBean` 빈을 생성하려고 합니다.
  2. 생성자를 호출해야 하는데, 생성자에 `@Value` 어노테이션이 붙은 파라미터가 있는 것을 발견합니다.
  3. `Environment`에서 `"template.path"` 프로퍼티 값을 찾습니다. (예: "classpath:myTemplate.txt" 문자열을 찾음)
  4. 찾은 **문자열 값**("classpath:myTemplate.txt")을 파라미터 타입인 **`Resource`** 로 **자동 변환**합니다. (스프링의 내장 `ResourceEditor` 사용)
  5. 변환된 `Resource` 객체(예: `ClassPathResource` 인스턴스)를 생성자의 `template` 파라미터로 전달하면서 생성자를 호출합니다.
  6. `MyBean` 객체 생성이 완료되고, 내부에 `Resource` 객체가 주입된 상태가 됩니다.
- **결론:** `@Component`는 빈 등록을 위한 것이고, `@Value`는 해당 빈(또는 그 생성자/메소드)에 외부 설정 값을 주입하기 위한 것입니다. 컴포넌트 스캔이 활성화되어 있다면, 스프링은 `@Component`를 발견하고 빈을 생성하는 과정에서 `@Value`를 처리하여 값을 자동으로 주입해줍니다.
