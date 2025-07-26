---
title: Stream flatMap & boxed 데이터 타입 흐름 추적하기
description: 
author: laze
date: 2025-07-26 00:00:01 +0900
categories: [Dev, Java]
tags: [Java, File, Files, Path, Paths]
---
# [Java] `File`, `Path`, `Files`  (파일 API 정리)

## 1. 서론: 아직도 `new File()` 쓰세요?

Java로 파일이나 디렉토리를 다룰 때, 많은 개발자들이 습관적으로 `java.io.File` 클래스를 사용합니다. 하지만 Java 7부터 도입된 **NIO.2(New I/O 2)** API인 `Path`, `Paths`, `Files`는 기존 `File` 클래스의 여러 문제점을 해결한, 훨씬 더 강력하고 현대적인 대안입니다.

기존 `File` 클래스는 다음과 같은 문제점을 안고 있었습니다.

- **모호한 역할:** 이름은 `File`이지만, 파일과 디렉토리를 모두 다루어 역할이 명확하지 않습니다.
- **불안정한 오류 처리:** 실패 시 `null`이나 `false`를 반환하여, 개발자가 오류 처리를 놓치기 쉽습니다.
- **제한적인 기능:** 파일 속성(권한, 소유자 등)에 대한 상세한 접근이 어렵고, 심볼릭 링크를 제대로 처리하지 못합니다.

이 글에서는 더 이상 `File` 클래스로 고통받지 않도록, 모던 Java 파일 API의 삼총사인 `Path`, `Paths`, `Files`의 역할과 관계를 명확히 정리하고, 실무에서 어떻게 활용하는지 구체적인 코드로 알아보겠습니다.

## 2. 역할 분담: 주소(`Path`), 주소 검색(`Paths`), 택배 기사(`Files`)

`Path`, `Paths`, `Files`는 각자 명확한 역할을 가지고 유기적으로 동작합니다. 이들의 관계를 '주소'와 '택배'에 비유하면 쉽게 이해할 수 있습니다.

| 클래스 | 역할 | 비유 |
| --- | --- | --- |
| **`Path` (인터페이스)** | **경로(Path) 그 자체**를 나타내는 불변(Immutable) 객체. | **지도 위의 주소**. (주소 정보만 있고, 실제 집은 없음) |
| **`Paths` (클래스)** | `Path` 객체를 **생성하는 공장(Factory)**. | **주소 검색 서비스**. (문자열 주소를 입력하면, 주소 객체 반환) |
| **`Files` (클래스)** | `Path`를 이용하여 **실제 파일/디렉토리를 조작**하는 유틸리티 클래스. | **택배 기사**. (주소를 보고, 실제 집에 가서 물건을 놓거나 가져옴) |

이들의 핵심적인 작업 흐름은 다음과 같습니다.

1. *`Paths.get("경로 문자열")`*을 사용해서 **`Path` 객체(주소)**를 만든다.
2. 만들어진 **`Path` 객체(주소)**를 **`Files` 클래스(택배 기사)**의 `static` 메서드에 전달하여, 실제 파일 조작을 수행한다.

## 3. 실전 예제: `Path`와 `Files`로 할 수 있는 모든 것

이제 실무에서 자주 사용하는 파일 처리 시나리오를 `Files`의 강력한 메서드들과 함께 살펴보겠습니다.

### 파일/디렉토리 생성

```java
// 1. Path 객체 생성
Path dirPath = Paths.get("./my-directory");
Path filePath = dirPath.resolve("my-file.txt"); // 기존 경로에 하위 경로를 추가

try {
    // 2. Files를 이용한 실제 조작
    if (Files.notExists(dirPath)) {
        Files.createDirectory(dirPath);
        System.out.println("디렉토리 생성 완료");
    }
    if (Files.notExists(filePath)) {
        Files.createFile(filePath);
        System.out.println("파일 생성 완료");
    }
} catch (IOException e) {
    e.printStackTrace();
}
```

- **핵심:** `Files`의 메서드들은 실패 시 `IOException`을 던지므로, 명시적인 예외 처리가 강제되어 더 안정적인 코드를 작성할 수 있습니다.

### 파일 쓰기 (간단한 텍스트)

```java
Path filePath = Paths.get("./my-directory/my-file.txt");
String content = "Hello, Modern Java File I/O!";

try {
    // Java 11+
    Files.writeString(filePath, content);

    // Java 8+
    // Files.write(filePath, content.getBytes(StandardCharsets.UTF_8));
    System.out.println("파일 쓰기 완료");
} catch (IOException e) {
    e.printStackTrace();
}
```

### 파일 읽기 (간단한 텍스트)

```java
Path filePath = Paths.get("./my-directory/my-file.txt");

try {
    // Java 11+
    String content = Files.readString(filePath);

    // Java 8+
    // byte[] bytes = Files.readAllBytes(filePath);
    // String content = new String(bytes, StandardCharsets.UTF_8);

    System.out.println("파일 내용: " + content);
} catch (IOException e) {
    e.printStackTrace();
}
```

- **주의:** `readString`, `readAllBytes`는 파일 전체를 메모리에 한 번에 올리므로, 작은 크기의 파일에만 적합합니다.

### 대용량 파일 스트리밍 (매우 중요)

대용량 파일을 처리할 때는 `Files.lines()`를 사용하여 메모리 문제를 피해야 합니다.

```java
Path largeFilePath = Paths.get("path/to/large-file.log");

try (Stream<String> lines = Files.lines(largeFilePath)) { // try-with-resources로 자동 close
    long errorCount = lines
            .filter(line -> line.contains("ERROR"))
            .count();
    System.out.println("에러 로그 수: " + errorCount);
} catch (IOException e) {
    e.printStackTrace();
}
```

- `Files.lines()`는 파일 전체를 메모리에 올리지 않고, `Stream` 파이프라인이 요청할 때마다 한 줄씩 **게으르게(lazily)** 읽어옵니다. 이 덕분에 메모리 사용량이 매우 효율적입니다.

### 파일 복사/이동/삭제

```java
Path source = Paths.get("./my-directory/my-file.txt");
Path copyTarget = Paths.get("./my-directory/my-file-copy.txt");
Path moveTarget = Paths.get("./my-directory-2/my-file-moved.txt");

try {
    // 복사 (StandardCopyOption.REPLACE_EXISTING: 대상 파일이 있으면 덮어쓰기)
    Files.copy(source, copyTarget, StandardCopyOption.REPLACE_EXISTING);

    // 이동 (이름 변경 포함)
    Files.createDirectory(Paths.get("./my-directory-2"));
    Files.move(copyTarget, moveTarget);

    // 삭제
    Files.delete(moveTarget);
    Files.deleteIfExists(Paths.get("./my-directory-2")); // 디렉토리 삭제
} catch (IOException e) {
    e.printStackTrace();
}
```

## 4. 레거시와의 공존: `File`과 `Path` 변환하기

오래된 라이브러리나 레거시 코드가 `File` 객체를 요구하거나 반환할 수 있습니다. 다행히 `File`과 `Path`는 서로 쉽게 변환할 수 있습니다.

```java
// File -> Path (가장 흔한 경우)
File legacyFile = new File("legacy.txt");
Path modernPath = legacyFile.toPath();

// Path -> File
Path newPath = Paths.get("new-file.txt");
File newFile = newPath.toFile();

// 이제 modernPath로 Files의 강력한 기능들을 사용할 수 있다.
if (Files.exists(modernPath)) {
    // ...
}
```

## 5. 결론: 이제는 `Path`와 `Files`를 사용하자

Java의 모던 파일 I/O API는 더 이상 선택이 아닌 필수입니다. 다음의 Best Practice를 기억하고 새로운 파일 처리 코드를 작성해 봅시다.

1. **새로운 코드는 무조건 `Path`와 `Files`를 사용하라.**
2. **역할을 명확히 기억하라:** `Paths`로 `Path`(경로 객체)를 만들고, `Files`로 실제 작업을 한다.
3. **대용량 파일은 반드시 `Files.lines()`를 사용하라.** 메모리 효율성이 비교할 수 없을 정도로 좋다.
4. **`File`은 오직 하위 호환성을 위해 필요할 때만, `toPath()`로 변환하여 사용하라.**

`File` 클래스의 시대는 저물었습니다. `Path`와 `Files`가 제공하는 명확성, 안정성, 그리고 강력한 기능들을 적극적으로 활용하여 더 나은 코드를 작성하시기 바랍니다.
