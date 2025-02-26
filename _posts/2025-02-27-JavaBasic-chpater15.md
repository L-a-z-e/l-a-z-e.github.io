---
title: 자바의 정석 Chapter 15
description: 모르는 내용 위주 학습 기록
author: laze
date: 2025-02-27 00:06:00 +0900
categories: [Dev, Java]
tags: [Java]
---
# Chapter 15

I/O

- 입출력 → 컴퓨터 내부 또는 외부의 장치와 프로그램간의 데이터를 주고받는 것

스트림

- 데이터를 운반하는데 사용되는 연결 통로
- 단방향 통신만 가능하기때문에 하나의 스트림으로 입력과 출력을 동시에 처리할 수 없다
- input stream ↔ output stream
- 먼저 보낸 데이터를 먼저 받게 되어있고 중간에 건너뜀 없이 연속적으로 데이터를 주고받음

바이트 기반 스트림 - InputStream, OutputStream

- 바이트 단위로 데이터 전송

| 입력스트림 | 출력스트림 | 입출력 대상 종류 |
| --- | --- | --- |
| FileInputStream | FileOutputStream | 파일 |
| ByteArrayInputStream | ByteArrayOutputStream | 메모리(byte배열) |
| PipedInputStream | PipedOutputStream | 프로세스(프로세스간의 통신) |
| AudioInputStream | AudioOutputStream | 오디오장치 |
- 입출력 표준화 방법 제공
- 추상메서드는 반드시 구현되어야함 그외는 구현한 추상메서드를 사용하기때문에 선행적으로 구현 필수

| 입력스트림 | 출력스트림 |
| --- | --- |
| abstract int read() | abstract void write(int b) |
| int read(byte[] b) | void write(byte[] b) |
| int read(byte[] b, int off, int len) | void write(byte[] b, int off, int len) |
- InputStream, OutputStream 메서드
    - InputStream
        
        
        | 메서드명 | 설명 |
        | --- | --- |
        | int available() | 스트림으로부터 읽어 올 수 있는 데이터의 크기 반환 |
        | void close() | 스트림을 닫아 사용하고 있던 자원을 반환 |
        | void mark(int readlimit) | 현재위치 표시. 후에 reset()에 의해 표시해놓은 위치로 돌아갈 수 있음 readlimit은 되돌아갈 수 있는 byte의 수 |
        | boolean markSupported() | mark()와 reset() 지원 여부 |
        | abstract int read() | 1byte를 읽어온다 (0~255 사이 값) 읽을 값이 없으면 -1 반환 |
        | int read(byte[] b) | 배열b 의 크기만큼 읽어서 배열을 채우고 읽어온 데이터의 수를 반환 |
        | int read(byte[] b, int off, int len) | 최대 len 개의 byte를 읽어서 배열 b의 off 위치부터 저장 |
        | void reset() | 스트림에서의 위치를 마지막으로 mark() 가 호출된 곳으로 되돌린다. |
        | long skip(long n) | 스트림에서 주어진 길이만큼 건너 뜀 |
    - OutputStream
    
    | 메서드명 | 설명 |
    | --- | --- |
    | void close() | 입력 소스를 닫고 사용하고 있던 자원 반환 |
    | void flush() | 스트림의 버퍼에 있는 모든 내용을 출력소스에 쓴다. |
    | abstarct void write(int b) | 주어진 값을 출력 소스에 쓴다. |
    | void write(byte[] b) | 주어진 배열 b에 저장된 모든 내용을 출력 소스에 쓴다. |
    | void write(byte[] b, int off, int len) | 주어진 배열 b에 저장된 내용 중에서 off부터 len개 만큼을 읽어서 출력 소스에 쓴다. |
    - 스트림 입출력 샘플 코드
    
    ```java
    public static void main(String[] args) {
    	byte[] inSrc = {0, 1, 2, 3, 4, 5, 6, 7, 8, 9};
    	byte[] outSrc = null;
    	
    	ByteArrayInputStream input = null;
    	ByteArrayOutputStream output = null;
    	
    	input = new ByteArrayInputStream(inSrc);
    	output = new ByteArrayOutputStream();
    	
    	int data = 0;
    	
    	while((data = input.read)) != -1) {
    		output.write(data);
    	}
    	
    	outSrc = output.toByteArray();
    	
    	System.out.println("Input Source :" + Arrays.toString(inSrc));
    	System.out.println("Output Source :" + Arrays.toString(outSrc));
    }
    ```
    
    - FileInputStream / FileOutputStream
    
    | 생성자 | 설명 |
    | --- | --- |
    | FileInputStream(String name) | 지정된 파일이름(name)을 가진 실제 파일과 연결된 FileInputStream 생성 |
    | FileInputStream(File file) | 파일의 이름이 String이 아닌 File 인스턴스로 지정해주어야한다는 점이 차이 |
    | FileInputStream(FileDescriptor fdObj) | 파일 디스크립터로 FileInputStream 생성 |
    | FileOutputStream(String name) | 지정된 파일이름(name)을 가진 실제 파일과 연결된 FileOutputStream 생성 |
    | FileOutputStream(String name, boolean append) | 두번째 인자를 true로 하면 출력시 기존의 파일 내용의 마지막에 덧붙임
    false면 기존의 파일 내용을 덮어씀 |
    | FileOutputStream(File file)
    FileOutputStream(File file, boolean append) | String → File 차이만 존재 |
    | FileOutputStream(FileDescriptor fdObj) | 파일 디스크립터로 FileOutputStream 생성 |
    - String, File, FileDescriptor
        
        `FileInputStream`과 `FileOutputStream` 생성자에서 사용되는 `String name`, `File file`, `FileDescriptor fdObj`의 차이점을 설명해 드리겠습니다. 핵심은 파일을 지정하는 방식과 그에 따른 유연성, 그리고 운영체제 수준에서의 파일 접근 방식의 차이입니다.
        
        **1. `String name` (문자열 경로):**
        
        - **가장 일반적인 방법:** 파일의 경로를 문자열(String)로 직접 지정합니다. 상대 경로(relative path) 또는 절대 경로(absolute path)를 사용할 수 있습니다.
            - 상대 경로: 현재 작업 디렉터리(current working directory)를 기준으로 파일의 위치를 나타냅니다. 예) `"data/input.txt"`
            - 절대 경로: 파일 시스템의 루트(root) 디렉터리부터 시작하여 파일의 전체 경로를 나타냅니다. 예) `"C:/Users/MyUser/Documents/input.txt"` (Windows), `"/home/myuser/documents/input.txt"` (Linux/macOS)
        - **간편함:** 사용하기 가장 쉽고 직관적입니다.
        - **내부 동작:** `FileInputStream` 또는 `FileOutputStream` 생성자는 내부적으로 이 문자열 경로를 사용하여 운영체제 API를 호출하여 파일을 열고, 파일 디스크립터(File Descriptor)를 얻습니다.
        
        **2. `File file` (File 객체):**
        
        - **`File` 객체 사용:** `java.io.File` 클래스의 인스턴스를 사용하여 파일을 나타냅니다. `File` 객체는 파일 또는 디렉터리에 대한 정보를 담고 있으며, 파일 시스템과 관련된 다양한 작업을 수행할 수 있는 메서드를 제공합니다.
            - `File` 객체는 생성 시점에 파일이 실제로 존재하지 않아도 됩니다. 파일 존재 여부 확인, 생성, 삭제, 속성 변경 등 파일 관련 작업을 수행할 때 유용합니다.
        - **유연성:** `File` 객체를 사용하면 파일 경로뿐만 아니라 파일의 존재 여부, 디렉터리 여부, 읽기/쓰기 권한 등을 미리 확인하고 조작할 수 있습니다. 파일과 관련된 여러 단계를 거쳐야 하는 경우 (예: 파일이 없으면 생성하고, 디렉터리가 없으면 생성하고, 권한을 확인하는 등)에 유용합니다.
        - **예시:**
            
            ```java
            File file = new File("data/input.txt"); // File 객체 생성 (파일이 없어도 됨)
            
            if (!file.exists()) { // 파일이 존재하는지 확인
                file.createNewFile(); // 파일 생성 (필요한 경우)
            }
            
            FileInputStream fis = new FileInputStream(file); // FileInputStream 생성
            
            ```
            
        - **내부동작:** `FileInputStream` 생성자는 `File`객체로부터 파일의 경로명을 추출하여, `String name`을 사용하는 경우와 동일한 방식으로 파일을 엽니다.
        
        **3. `FileDescriptor fdObj` (파일 디스크립터):**
        
        - **가장 낮은 수준의 접근:** 운영체제가 파일, 소켓 등 I/O 리소스를 식별하기 위해 사용하는 정수 값인 파일 디스크립터(File Descriptor)를 직접 사용합니다.
        - **특수한 경우에 사용:** 일반적인 파일 입출력에서는 거의 사용되지 않습니다. 다음과 같은 특수한 상황에서 사용될 수 있습니다.
            - 이미 다른 방법(예: `FileDescriptor.in`, `FileDescriptor.out`, `FileDescriptor.err` (표준 입출력), 다른 네이티브 코드)을 통해 얻은 파일 디스크립터를 `FileInputStream` 또는 `FileOutputStream`에서 사용해야 하는 경우.
            - 매우 낮은 수준의 I/O 제어가 필요한 경우 (성능 최적화 등)
            - 표준 입/출력 스트림을 다른 스트림으로 리다이렉션(redirection) 할 때
        - **주의사항**: 파일 디스크립터는 운영체제에 의해 관리되므로, 직접 생성하거나 조작하는 것은 매우 주의해야 합니다. 잘못 사용하면 시스템 자원 누수, 보안 문제 등이 발생할 수 있습니다. 일반적인 애플리케이션 개발에서는 `FileDescriptor`를 직접 사용할 일이 거의 없습니다.
        - **예시**:
            
            ```java
            // 표준 입력(stdin)을 나타내는 파일 디스크립터를 얻음
            FileDescriptor fd = FileDescriptor.in;
            
            FileInputStream fis = new FileInputStream(fd); // 표준 입력을 FileInputStream으로 사용
            
            // ... fis를 사용하여 표준 입력에서 데이터를 읽음
            
            ```
            
        
        **요약 및 비교:**
        
        | 방식 | 설명 | 장점 | 단점 | 사용 빈도 |
        | --- | --- | --- | --- | --- |
        | `String name` | 파일 경로를 문자열로 직접 지정 | 간편하고 직관적 | 파일 관련 추가 작업(존재 여부 확인, 생성 등)을 하려면 별도 코드가 필요 | 높음 |
        | `File file` | `java.io.File` 객체를 사용하여 파일을 나타냄. | 파일 관련 다양한 작업을 수행할 수 있고, `FileInputStream`/`FileOutputStream` 생성 전에 파일 상태를 제어할 수 있음 | `File` 객체를 생성해야 하는 추가 단계가 필요 | 중간 |
        | `FileDescriptor` | 운영체제 수준의 파일 디스크립터를 사용. | 매우 낮은 수준의 I/O 제어 가능, 이미 존재하는 파일 디스크립터를 활용 가능 | 사용이 복잡하고 위험하며, 일반적인 파일 입출력에서는 거의 사용되지 않음 | 낮음 |
        
        대부분의 경우 `String name` 또는 `File file`을 사용하면 충분하며, `FileDescriptor`는 특수한 경우에만 제한적으로 사용됩니다. `File` 객체는 파일 관련 작업을 좀 더 유연하게 처리할 수 있다는 장점이 있습니다.
        

보조 스트림

- 스트림의 기능 향상 혹은 새로운 기능 추가를 위해 사용
    - FileInputStream의 성능 향상을 위해 BufferedInputStream 사용

| 입력 | 출력 | 설명 |
| --- | --- | --- |
| FilterInputStream | FilterOutputStream | 필터를 이용한 입출력 처리 |
| BufferedInputStream | BufferedOutputStream | 버퍼를 이용한 입출력 성능 향상 |
| DataInputStream | DataOutputStream | int, float 같은 기본형 단위로 데이터 처리 |
| SequenceInputStream |  | 두 개의 스트림을 하나로 연결 |
| LineNumberInputStream |  | 읽어 온 데이터의 라인 번호 카운트 |
| ObjectInputStream | ObjectOutputStream | 데이터를 객체 단위로 읽고 쓰는데 사용
주로 파일을 이용하며 객체 직렬화와 관련 |
|  | PrintStream | 버퍼를 이용, print 관련 기능 |
| PushbackInputStream |  | 버퍼를 이용해 읽어 온 데이터를 다시 되돌리는 기능 |
- 바이트 기반의 보조 스트림
    - FilterInput/OutputStream → 모든 보조 스트림의 조상
    - 상속을 통해 오버라이딩해서 사용해야함
        - BufferedInputStream, DataInputStream, PushbackInputStream 등
        - 버퍼를 이용해서 한 번에 여러 바이트를 입출력하는 것이 빠르기 때문에 사용함
        - BufferedInputStream(InputStream in, int size) - default 8192(8K)
        - OutputStream → flush() 출력소스에 버퍼 내용 출력하고 버퍼 비움
        close() → flush() 호출하고 BufferedOutputStream 인스턴스가 사용하던 자원 반환
        - DataInput/OutputStream → 각 타입에 맞게 값 읽어옴
            - `char readChar()` / `void writeChar(int c)`
            - `double readDouble()` / `void writeDouble(double d)`
            - `String readUTF()` / `void writeUTF(String s)`
            - 이때 저장은 binary data로 저장되기때문에 ByteArrayOutputStream 사용해야 이진 데이터 확인 가능 혹은 DataInputStream의 read 메서드 사용
    - try-with-resource
        
        `try-with-resources` 구문에 대해 다시 한번 명확하게 정리해 드리겠습니다. 기존 `try-catch-finally`와 비교하여, 특히 `try` 괄호 안에 리소스를 선언하는 부분과 리소스 처리 순서에 초점을 맞춰 설명하겠습니다.
        
        **1. `try-with-resources` 구조와 기존 `try-catch-finally` 비교**
        
        | 구분 | `try-with-resources` | `try-catch-finally` |
        | --- | --- | --- |
        | **기본 구조** | `try (리소스 선언) { ... } catch (예외 타입 e) { ... }` | `try { ... } catch (예외 타입 e) { ... } finally { 리소스 해제 }` |
        | **리소스 선언 위치** | `try` 키워드 바로 뒤 괄호 `()` 안에 | `try` 블록 안, 또는 `try` 블록 이전에 별도로 선언 |
        | **리소스 해제** | 자동. `try` 블록 종료 시 (정상/예외 모두) 선언된 역순으로 리소스의 `close()` 메서드 자동 호출 | 수동. `finally` 블록에서 명시적으로 `close()` 메서드 호출해야 함. `null` 체크, `close()` 중 예외 처리 필요 |
        | **코드 간결성** | 간결함. `finally` 블록 불필요 | 복잡함. `finally` 블록에서 리소스 해제 코드 필요 |
        | **가독성** | 높음. 리소스 해제 코드가 분리되어 핵심 로직에 집중 가능 | 낮음. 리소스 해제 코드가 핵심 로직과 섞임 |
        | **안전성** | 높음. `close()` 호출 누락, `close()` 중 예외 처리 누락 방지 | 낮음. `close()` 호출 누락, `close()` 중 예외 처리 누락 가능성 있음 |
        | **AutoCloseable** | 리소스는 반드시 `java.lang.AutoCloseable` 인터페이스 구현해야 함 | 제약 없음 (하지만 `close()` 메서드를 가진 객체를 사용하는 것이 일반적) |
        
        **2. `try` 괄호 안 리소스 선언 방식**
        
        - **선언과 초기화:** `try` 괄호 안에는 리소스 객체를 *선언*하고 *초기화*하는 코드를 작성합니다. 일반적인 변수 선언과 동일한 문법을 사용합니다.
            - `타입 변수명 = new 생성자();`
            
            ```java
            try (FileInputStream fis = new FileInputStream("myFile.txt")) {
                // ...
            }
            
            ```
            
        - **다중 리소스 선언:** 여러 개의 리소스를 선언할 때는 세미콜론(`;`)으로 구분합니다.
            
            ```java
            try (FileInputStream fis = new FileInputStream("input.txt");
                 FileOutputStream fos = new FileOutputStream("output.txt")) {
                // ...
            }
            
            ```
            
        - **final 또는 effectively final** : `try`의 괄호 안에 선언된 변수는 `final`이거나, `effectively final`이어야 합니다. `effectively final`이란, 변수가 초기화된 후, 값이 변경되지 않는 변수를 말합니다.
        
        **3. 데이터 읽기 및 `close()` 시점, 순서**
        
        - **데이터 읽기:** `try` 블록 *안*에서 선언된 리소스를 사용하여 데이터를 읽거나 씁니다. `try` 괄호 안에 선언된 리소스는 해당 `try` 블록 내에서만 유효한 지역 변수(local variable)입니다.
            
            ```java
            try (BufferedReader reader = new BufferedReader(new FileReader("myFile.txt"))) {
                String line;
                while ((line = reader.readLine()) != null) { // reader를 사용하여 파일 읽기
                    System.out.println(line);
                }
            } catch (IOException e) {
                // 예외 처리
            }
            
            ```
            
        - **`close()` 시점:** `try` 블록이 *종료되는 시점*에 자동으로 리소스의 `close()` 메서드가 호출됩니다.
            - 정상 종료: `try` 블록의 코드가 모두 성공적으로 실행된 후
            - 예외 발생: `try` 블록 내에서 예외가 발생하여 `catch` 블록으로 제어가 넘어가는 시점 ( `catch` 블록 실행 *전* )
        - **`close()` 순서 (역순):** 여러 개의 리소스가 선언된 경우, `close()` 메서드는 *선언된 역순*으로 호출됩니다. 가장 나중에 선언된 리소스가 가장 먼저 닫힙니다. 이는 리소스 간의 의존성(dependency) 때문입니다.
            
            ```java
            try (ResourceA a = new ResourceA();
                 ResourceB b = new ResourceB(a); // b가 a에 의존
                 ResourceC c = new ResourceC()) {
                // ...
            } // c.close() -> b.close() -> a.close() 순서로 호출
            
            ```
            
            위의 예시에서, `ResourceB`는 `ResourceA`에 의존하므로, `ResourceB`가 먼저 닫히고 그 다음에 `ResourceA`가 닫혀야 안전합니다.
            
        
        **4. 예외 처리**
        
        - try 블록에서 예외가 발생하면, 아직 close()가 호출되지 않은 리소스들의 close()가 선언 역순으로 호출 된 후, 해당 예외가 catch 블록으로 전달된다.
        - close()메서드 내에서 예외가 발생하면, 이 예외는 억제된 예외로 처리된다. 원래 try블록에서 발생했던 예외(또는 다른 close()에서 발생한 억제된 예외)에 이 억제된 예외가 추가된다. Throwable.getSuppressed()메서드를 사용하여 억제된 예외들을 확인할 수 있다.
        
        **정리 (단계별 순서):**
        
        1. **리소스 선언 및 초기화:** `try` 괄호 안에 리소스들을 선언하고 초기화합니다 (여러 개일 경우 세미콜론으로 구분).
        2. **`try` 블록 실행:** `try` 블록 안의 코드를 실행합니다. 이 코드에서 선언된 리소스들을 사용하여 데이터를 읽거나 쓰는 등의 작업을 수행합니다.
        3. **`try` 블록 종료:**
            - **정상 종료:** `try` 블록의 코드가 모두 성공적으로 실행되면, 선언된 역순으로 리소스들의 `close()` 메서드가 자동으로 호출됩니다.
            - **예외 발생:** `try` 블록 내에서 예외가 발생하면, 아직 닫히지 않은 리소스들의 `close()` 메서드가 선언 역순으로 자동으로 호출된 *후*, `catch` 블록으로 제어가 넘어갑니다.
        4. **`catch` 블록 실행 (예외 발생 시):** `catch` 블록에서 예외를 처리합니다. `close()` 메서드에서 발생한 예외는 억제된 예외로 처리되어 원래 예외에 추가됩니다.
        5. **프로그램 계속 실행:** `try-catch` 블록 이후의 코드가 계속 실행됩니다.
        
        `try-with-resources`를 사용하면 리소스 관리가 자동화되어 코드가 더 간결해지고 안전해집니다.
        
    - SequnceInputStream → 여러 개의 스트림을 연속적으로 연결해 하나의 스트림처럼 처리
        - `SequenceInputStream(Enumeratione) / SequnceInputStream(InputStream s1, InputStream s2)`
    - 

문자 기반 스트림

- 입출력 단위 2byte
- InputStream → Reader
- OutputStream → Writer

| 바이트 기반 | 문자 기반 | 입출력 대상 종류 |
| --- | --- | --- |
| FileInputStream
FileOutputStream | FileReader
FileWriter | 파일 |
| ByteArrayInputStream
ByteArrayOutputStream | CharArrayReader
CharArrayWriter | 메모리(byte배열) |

| InputStream | Reader |
| --- | --- |
| abstract int read() | int read() |
| int read(byte[] b) | int read(char[] cbuf) |
| int read(byte[] b, int off, int len) | abstact int read(char[] cbuf, int off, int len) |

| OutputStream | Writer |
| --- | --- |
| abstract void write(int b) | void write(int c) |
| void write(byte[] b) | void write(char[] cbuf) |
| void write(byte[] b, int off, int len) | abstracct void wrtite(char[] cbuf, int off, int len) |
- Reader / Writer
    - ByteStream과 동일한 메서드들을 사용하지만 배열이 byte가 아닌 char 배열
    - close() / mark() / markSupported() / read() / ready() / reset()
    - FileReader / FileWriter
        - 파일로부터 텍스트 데이터를 읽고 쓰는데 사용
    - PipedReader / PipedWriter
        - 쓰레드 간에 데이터를 주고 받을 때 사용
        - connect()를 어느 한 쪽 쓰레드에서 호출해서 연결
        - 둘 중 하나가 닫히면 나머지도 자동으로 닫힘
    - StringReader / StringWriter
        - 입출력 대상이 메모리인 스트림
        - StringWriter에 출력되는 데이터는 내부의 StringBuffer 에 저장
            - StringBuffer getBuffer() / String toString() 으로 저장된 데이터를 얻을 수 있음
- 문자기반의 보조 스트림
    - BufferedReader / BufferedWriter
        - 버퍼를 이용해 입출력의 효율을  높임
        - readLine() 데이터를 라인 단위로 읽을 수 있음
        - newLine() 줄바꿈
    - InputStreamReader / OutputStreamWriter
        - 바이트 기반 스트림 → 문자 기반 스트림으로 연결 + 바이트 기반 스트림의 데이터를 지정된 인코딩의 문자데이터로 변환
        
        | 메서드 | 설명 |
        | --- | --- |
        | InputStreamReader(InputStream in) | OS에서 사용하는 기본 인코딩의 문자로 변환하는 InputStreamReader 생성 |
        | InputStreamReader(InputStream in, String encoding) | 지정된 인코딩을 사용하는 InputStreamReader 생성 |
        | String getEncoding() | InputStreamReader의 인코딩 반환 |
        | OutputStreamWriter(OutputStream out) | OS에서 사용하는 기본 인코딩의 문자로 변환하는 OutputStreamWriter 생성 |
        | OutputStreamWriter(OutputStream out, String encoding) | 지정된 인코딩을 사용하는 OutputStreamWriter 생성 |
        | String getEncodig() | OutputStreamWriter 의 인코딩 반환 |

- 표준 입출력
    - System.in - 콘솔로부터 데이터를 입력받는데 사용
    - System.out - 콘솔로부터 데이터를 출력하는데 사용
    - System.err - 콘솔로부터 데이터를 출력하는데 사용
    - System 클래스의 static 변수 / 입출력 스트림
    - 표준 입출력 대상 변경 - setOut(), setErr(), setIn() 으로 다른 입출력대상 사용 가능
- RandomAccesFile
    - 하나의 클래스로 파일에 대한 입력과 출력을 모두 할 수 있도록 설계됨
    - 파일의 어느 위치에나 읽기 / 쓰기가 가능함
    
    | RandomAccessFile(File file, String mode)
    RandomAccessFile(String fileName, String mode) | RandomAccessFile 인스턴스 생성
    r / rw / rws(출력내용 지연없이 파일에 씀) / rwd (rws + 메타정보 포함) |
    | --- | --- |
    | FileChannel getChannel() | 파일 채널 반환 |
    | FileDescriptor getFD() | 파일 디스크립터 반환 |
    | long getFilePointer() | 파일의 포인터 위치 반환 |
    | long length() | 파일의 크기 (byte) |
    | void seek(long pos) | 파일의 포인터의 위치 변경 |
    | void setLength(long newLength) | 파일의 크기를 지정된 길이로 변환 |
    | int skipBytes(int n) | 지정된 수 만큼의 byte를 건너 뜀 |

- File
    - 생성자 / 메서드
        
        
        | File(String fileName) | 주어진 문자열을 이름으로 갖는 File 인스턴스 생성, 주로 경로를 포함해서 지정하지만 파일 이름만 사용하는 경우 프로그램이 실행되는 위치가 path로 간주 됨 |
        | --- | --- |
        | File(String pathName, String fileName)
        File(File pathName, String fileName) | 파일의 경로와 이름을 따로 분리해서 지정할 수 있도록 한 생성자 |
        | File(URI uri) | 지정된 uri로 파일 생성 |
        | String getName() | 파일의 이름 반환 |
        | String getPath() | 파일의 경로 반환 |
        | String getAbsolutePath()
        File getAbsolutePath() | 파일의 절대경로 반환 |
        | String getParent()
        Fild getParent() | 파일의 조상 디렉토리 반환 |
        | String getCanonicalPath()
        File getCanonicalPath() | 파일의 정규 경로 반환 |
    - 멤버 변수
    
    | static String pathSeparator | OS에서 사용하는 경로(path) 구분자. 윈도우 “;”, 유닉스 “:” |
    | --- | --- |
    | static char pathSeparatorChar | OS에서 사용하는 경로(path) 구분자. 윈도우 “;”, 유닉스 “:” |
    | static String separator | OS에서 사용하는 이름 구분자. 윈도우 “\”, 유닉스 “/” |
    | static char separator | OS에서 사용하는 이름 구분자. 윈도우 “\”, 유닉스 “/” |
- 직렬화
    - 객체를 데이터 스트림으로 만드는 것을 뜻함
    - ObjectOutputStream - 스트림에 객체를 출력
- 역직렬화
    - 스트림으로부터 데이터를 읽어서 객체를 만드는 것
    - ObjectInputStream - 스트림으로부터 객체를 입력
- 직렬화 가능한 클래스 - Serializable, transient
    - Serializable - 직렬화를 고려하여 작성한 클래스인지 판단하는 기준
    - transient - 비밀번호등 직렬화 대상에서 제외할 수 있도록 함
    - serialVersionUID - 클래스의 이름이 같더라도 클래스의 내용이 변경된 경우 역직렬화는 실패하게 되므로 클래스의 버전을 관리함