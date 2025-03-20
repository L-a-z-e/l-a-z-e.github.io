---
title: 자바의 정석 Chapter 16
description: 모르는 내용 위주 학습 기록
author: laze
date: 2025-02-27 00:00:16 +0900
categories: [Dev, Java]
tags: [Java]
---
# Chapter 16

- client
    - service user
- server
    - service provider
    - file server / mail server / application server
    - 전용 서버 - server-based model
    - p2p모델 - 각 클라이언트가 서버 역할 수행
- IP
    - 4byte 정수 4개의 정수가 .를 구분자로 표현됨 0~255 사이 정수
    - ip address - 192.168.10.100
        - 192.168.10 - 네트워크 주소
        - 100 - 호스트 주소
    - subnet mask
        - 255.255.255.0
        - ip 주소와 & 연산 하면 네트워크 주소만 뽑아낼 수 있음
        - 같은 네트워크 주소 상에 존재하는지 확인
- InetAddress
    
    
    | byte[] getAddress() | IP주소를 byte 배열로 반환 |
    | --- | --- |
    | static InetAddress getAllByName(String host) | 도메인명(host)에 지정된 모든 호스트의 IP주소를 배열에 담아 반환 |
    | static InetAddress getByAddress(byte[] addr) | byte배열을 통해 IP주소를 얻는다 |
    | String getCanonicalHostName() | FQDN(fully qualified domain name) 반환 |
    | String getHostAddress() | 호스트 IP 주소 반환 |
    | String getHostName() | 호스트 이름 반환 |
    | static InetAddress getLocalHost() | 지역호스트의 IP주소 반환 |
    | boolean isMulticastAddress() | IP주소가 멀티캐스트 주소인지 알려줌 |
    | boolean isLoopbackAddress() | IP주소가 loopback 주소 (127.0.0.1)인지 알려 줌 |
- URL(Uniform Resource Locator)
    - 자원에 접근할 수 있는 주소
    - `프로토콜://호스트명:포트번호/경로명/파일명?쿼리스트링#참조`
        - 프로토콜 - 자원에 접근하기 위해 서버와 통신하는데 사용되는 통신규약 (http)
        - 호스트명 - 자원을 제공하는 서버의 이름(www.google.com)
        - 포트 번호 - 통신에 사용되는 서버의 포트 번호 (80)
        - 경로명 - 접근하려는 자원이 저장된 서버상의 위치 (/sample/)
        - 쿼리 - URL에서 ? 이후 부분
        - 참조(anchor) - URL에서 #이후의 부분
    
    | URL(String spec) | 지정된 문자열 정보의 URL 객체 생성 |
    | --- | --- |
    | URL(String protocol, String host, int port, String file) | 지정된 값으로 구성된 URL 객체  생성 |
    | String getAuthority() | 호스트명과 포트를 문자열로 반환 |
    | Object getContent()
    Object getContent(Class[] classes) | URL의 Content 객체 반환 |
    | URLConnection openConnection()
    URLConnection openConnection(Proxy proxy) | URL과 연결된 URLConnection을 얻는다 |
    | InputStream openStream() | URL과 연결된 URLConnection의 InputStream을 얻는다. |
    | String toExternalForm() | URL을 문자열로 변환하여 반환한다. |
    | URI toURI() | URL을 URI로 변환하여 반환한다. |
- URLConnection
    - 어플리케이션과 URL간의 통신 연결을 나타내는 클래스의 최상위 클래스
    - HttpURLConnection / JarURLConnection 등이 이를 상속받아 구현한 클래스
    - content 객체 반환 / header 필드 값 / 캐시 사용여부 등
    
    ```java
    public class URLConnectionSample {
    	public static void miain(String[] args) {
    		URL url = null;
    		BufferedReader input = null;
    		String address = "http://www.sample/board/list.html";
    		String line = "";
    		
    		try {
    			url = new URL(address);
    			input = new BufferedReader(new InputStreamReader(url.openStream()));
    			
    			while((line=input.readLine() != null) {
    				System.out.println(line);
    			}
    			input.close();
    		}
    		catch (Exception e) {
    			e.printStackTrace();
    		}
    	}
    }
    ```
    
- 소켓 프로그래밍
    - 소켓 - 프로세스간의 통신에 사용되는 양쪽 끝단(endpoint)
- TCP/IP 프로토콜
    - 이기종 시스템간의 통신을 위한 표준
    - 전송계층(transport layer)에 해당하는 프로토콜
    - TCP
        - 연결 기반, 1:1 통신방식
        - 데이터의 경계 구분 안함 (byte stream)
        - 데이터 전송순서 / 수신여부 / 손실시 재전송 / UDP에 비해 전송속도 느림
        - Socket , ServerSocket
    - UDP
        - 비연결기반, n:n 통신방식
        - 데이터의 경계 구분 (datagram)
        - 신뢰성없는 데이터 전송(전송순서, 수신여부), 패킷 관리 필요
        - DatagramSocket, DatagrramPacket, MulticastSocket
    - TCP 소켓 프로그래밍
        1. 서버 프로그램에서는 서버 소켓을 사용해 특정 포트에서 클라이언트의 연결요청을 처리할 준비
        2. 클라이언트 프로그램에서는 IP주소, 포트 정보를 가지고 소켓 생성해서 서버에 연결 요청
        3. 연결 요청 받은 서버에서 새로운 소켓 생성해서 클라이언트 소켓과 연결
        4. 클라이언트 소켓 ↔ 서버의 새로 생성된 소켓 1:1 통신 (서버 소켓과 관계없음)
        - Socket - InputStream, OutputStream을 가지고있음 / 프로세스간 통신(입출력) 이루어짐
        - ServerSocket - 연결 요청되면 Socket 생성해서 통신을 이루어지도록 함. 한 포트당 하나의 ServerSocket만 연결 가능
    - UDP 소켓 프로그래밍
        - DatagramSocket 사용
        - DatagramPacket - 헤더(호스트 주소 + 포트)와 데이터로 구성 → DatagramSocket으로 보냄