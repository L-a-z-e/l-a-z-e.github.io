---
title: Tomcat - LibraryNotFoundError tcnative-2.dll 오류 해결
description: 
author: laze
date: 2025-08-11 00:00:01 +0900
categories: [Dev, SpringBoot]
tags: [SpringBoot, Tomcat, LibraryNotFoundError, tcnative]
---
# [Spring Boot 트러블슈팅] `tcnative-2.dll` 오류 해결: Tomcat Native로 성능 올리기

## 1. 서론: 오류 메시지를 '성능 개선의 기회'로 바라보기

Spring Boot 애플리케이션을 실행할 때, 특히 Windows 환경이나 Docker 빌드 과정에서 다음과 같은 `LibraryNotFoundError`를 마주칠 때가 있습니다.

```
org.apache.tomcat.jni.LibraryNotFoundError: Can't load library: bin\\tcnative-2.dll
```

많은 개발자들이 이 오류를 귀찮은 경고 정도로 여기고, 인터넷에서 `tomcat.apr.enabled=false` 설정을 찾아 급히 끄는 것으로 문제를 해결하곤 합니다. 하지만 이 오류는 단순히 무시해도 될 경고가 아니라, **"Tomcat의 성능을 한 단계 더 끌어올릴 수 있는 기회를 놓치고 있다"**는 중요한 신호입니다.

이 글의 목표는 이 오류를 단순히 '끄는 것'이 아니라, 근본 원인을 진단하고 **'해결하여'** Tomcat Native (APR)가 제공하는 강력한 성능 이점을 온전히 누리는 방법을 알아보는 것입니다.

## 2. 1단계: `LibraryNotFoundError`의 진짜 원인 추적하기

`tcnative` 라이브러리는 Apache Tomcat이 OS 수준의 네이티브 기능을 활용하여 성능을 극대화할 수 있도록 돕는 C언어 기반 라이브러리입니다. 특히 **HTTPS/SSL 처리 성능을 비약적으로 향상**시키므로, 트래픽이 많은 프로덕션 환경에서는 그 가치가 더욱 빛납니다.

이 라이브러리를 로드하지 못하는 이유는 다음 세 가지 용의자 중 하나일 가능성이 99%입니다. 체계적인 진단 절차를 통해 범인을 찾아봅시다.

### 진단 1: 아키텍처(Bitness) 일치 확인 - 가장 흔한 함정

가장 먼저 확인할 것은 **JVM의 아키텍처**와 **네이티브 라이브러리의 아키텍처**가 일치하는지 여부입니다. 이것이 가장 흔하면서도 가장 찾기 어려운 오류의 원인입니다.

- **핵심:** OS의 아키텍처(64비트 Windows)가 아니라, **그 위에서 실행되는 JVM의 아키텍처**가 기준입니다. 64비트 OS에서도 32비트 JDK를 설치하여 사용할 수 있기 때문에 주의가 필요합니다.

**1. 내 JVM 아키텍처 확인하기**
터미널을 열고 다음 명령어를 실행합니다.

```bash
java -version
```

출력된 내용에서 `64-Bit Server VM` 문구가 포함되어 있는지 확인합니다. 만약 이 문구가 없다면 32비트 JVM일 가능성이 높습니다.

**2. `tcnative` 라이브러리 아키텍처 확인하기**[Tomcat Native 공식 다운로드 페이지](https://tomcat.apache.org/download-native.cgi) 등에서 라이브러리를 다운로드 받을 때, 파일 이름에 아키텍처 정보가 포함되어 있습니다.

- `tomcat-native-x.x.x-win32-bin.zip`: **32비트**용
- `tomcat-native-x.x.x-win64-bin.zip`: **64비트**용

**64비트 JVM은 64비트 DLL만 로드할 수 있고, 32비트 JVM은 32비트 DLL만 로드할 수 있습니다.** 이 둘이 일치하지 않으면 `LibraryNotFoundError`가 발생합니다.

### 진단 2: 라이브러리 파일 위치 확인

아키텍처가 일치한다면, 다음은 파일이 올바른 위치에 있는지 확인할 차례입니다.

- **파일 이름:**
  - Windows: `tcnative-2.dll` (또는 버전에 따라 `tcnative-1.dll`)
  - Linux: `libtcnative-1.so` (또는 유사한 이름)
- **어디서 구하는가?:**
  - **Windows:** 위에서 언급한 Tomcat 공식 사이트에서 다운로드합니다.
  - **Linux:** `apt-get install libtcnative-1` (Debian/Ubuntu) 또는 `yum install tomcat-native` (CentOS/RHEL) 같은 패키지 매니저를 통해 설치하는 것이 가장 좋습니다.

### 진단 3: `java.library.path` 경로 확인

Java가 네이티브 라이브러리를 찾기 위해 스캔하는 경로를 `java.library.path`라고 합니다. 라이브러리가 위치한 디렉토리가 이 경로에 포함되어 있어야 합니다.

**1. 현재 경로 확인하기**
애플리케이션 코드에 다음 한 줄을 추가하여 현재 `java.library.path`를 출력해볼 수 있습니다.

```java
System.out.println(System.getProperty("java.library.path"));
```

**2. 경로에 추가하는 방법**

- **OS 환경 변수 설정:** Windows의 `PATH` 또는 Linux의 `LD_LIBRARY_PATH` 환경 변수에 라이브러리가 있는 디렉토리 경로를 추가합니다. (시스템 전역 설정)
- **JVM 시작 옵션 (권장):** `Djava.library.path=/path/to/lib` 옵션을 사용하여 애플리케이션 실행 시에만 경로를 지정합니다. 이 방법이 더 깔끔하고 프로젝트에 종속적인 해결책입니다.

## 3. 2단계: 시나리오별 구체적인 해결 가이드

진단이 끝났다면, 이제 환경에 맞는 해결책을 적용할 차례입니다.

### 시나리오 A: 로컬 Windows 개발 환경 (with IntelliJ)

1. `java -version`으로 내 JVM 아키텍처(e.g., 64비트)를 확인합니다.
2. Tomcat 공식 사이트에서 **64비트용** `tomcat-native-...-win64-bin.zip` 파일을 다운로드합니다.
3. 압축을 풀고 `bin/x64/` 디렉토리 안에 있는 `tcnative-2.dll` 파일을 찾습니다.
4. 프로젝트 루트에 `lib/native` 같은 폴더를 만들고 그 안에 `tcnative-2.dll` 파일을 복사합니다.
5. IntelliJ 상단의 'Run/Debug Configurations' (실행 구성 편집)으로 들어갑니다.
6. 사용하는 실행 구성의 'Modify options' -> 'Add VM options'를 선택합니다.
7. 'VM options' 입력란에 다음을 추가합니다. 프로젝트 루트를 기준으로 상대 경로를 사용할 수 있습니다.

    ```
    -Djava.library.path=lib/native
    ```


이제 애플리케이션을 실행하면, JVM은 `lib/native` 폴더를 `java.library.path`에 포함시키고, Tomcat은 성공적으로 `tcnative-2.dll`을 로드할 것입니다.

### 시나리오 B: Docker/Linux 환경

`Dockerfile`에서 패키지 매니저를 사용하여 네이티브 라이브러리를 설치하는 것이 가장 안정적이고 좋은 방법입니다.

```docker
# Dockerfile (Debian/Ubuntu 기반 예시)
FROM openjdk:17-jdk-slim

# Tomcat Native 라이브러리 설치
# RUN 명령어 하나로 묶어 Docker 이미지 레이어를 최소화하는 것이 좋다.
RUN apt-get update && \\
    apt-get install -y libtcnative-1 && \\
    rm -rf /var/lib/apt/lists/*

WORKDIR /app
COPY build/libs/*.jar app.jar
ENTRYPOINT ["java", "-jar", "app.jar"]
```

패키지 매니저로 설치하면 보통 `/usr/lib` 등 표준 라이브러리 경로에 설치되므로, 별도의 `java.library.path` 설정이 필요 없는 경우가 대부분입니다.

## 4. 3단계 (대안): APR 기능을 비활성화해야 할 때

이 글의 목표는 APR을 활성화하는 것이지만, 다음과 같은 특수한 경우에는 비활성화가 더 나은 선택일 수 있습니다.

- 네이티브 라이브러리 설치가 불가능한 제한적인 CI/CD 파이프라인 환경.
- HTTPS 트래픽이 거의 없고, 고성능 I/O가 중요하지 않은 간단한 내부 관리용 애플리케이션.

`application.properties`나 `main` 메서드에서의 설정은 Tomcat의 클래스 로딩 시점보다 늦게 적용되어 효과가 없을 수 있습니다. **가장 확실한 비활성화 방법은 JVM 시작 옵션을 사용하는 것입니다.**

```docker
# Dockerfile에서 비활성화
ENTRYPOINT ["java", "-Dorg.apache.catalina.useAprConnector=false", "-jar", "app.jar"]
```

IntelliJ에서는 'VM options'에 `-Dorg.apache.catalina.useAprConnector=false`를 추가하면 됩니다.

## 5. 결론: 오류 해결을 넘어 성능 최적화로

`tcnative-2.dll` `LibraryNotFoundError`는 단순히 해결해야 할 오류가 아니라, 내 애플리케이션의 성능을 한 단계 끌어올릴 수 있는 **최적화의 기회**입니다.

이 오류를 마주쳤을 때, 무조건 기능을 끄는 대신 다음 순서로 문제를 해결하는 습관을 들입시다.

1. **아키텍처(Bitness) 확인:** 내 JVM과 라이브러리의 비트가 일치하는가?
2. **파일 위치 확인:** 올바른 라이브러리 파일이 시스템에 존재하는가?
3. **라이브러리 경로 설정:** `java.library.path`에 해당 파일의 경로가 포함되어 있는가?

올바르게 설정된 Tomcat Native (APR)는 특히 트래픽이 많은 프로덕션 환경에서 눈에 띄는 응답 속도 개선과 리소스 사용량 감소를 가져다줄 수 있습니다.
