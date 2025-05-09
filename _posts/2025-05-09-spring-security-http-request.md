---
title: Protection Against Exploits - Http Request
description: 
author: laze
date: 2025-05-09 00:00:02 +0900
categories: [Dev, SpringSecurity]
tags: [SpringSecurity]
---
## HTTP

정적 리소스를 포함한 모든 HTTP 기반 통신은 TLS를 사용하여 보호되어야 합니다.

프레임워크로서 Spring Security는 HTTP 연결을 직접 처리하지 않으므로 HTTPS를 직접 지원하지는 않습니다.

그러나 HTTPS 사용에 도움이 되는 여러 가지 기능을 제공합니다.

### HTTPS로 리디렉션

클라이언트가 HTTP를 사용하는 경우, 서블릿 및 웹플럭스 환경 모두에서 Spring Security를 구성하여 HTTPS로 리디렉션하도록 할 수 있습니다.

### 엄격한 전송 보안 (Strict Transport Security)

Spring Security는 엄격한 전송 보안(Strict Transport Security)을 지원하며 기본적으로 활성화합니다.

### 프록시 서버 구성

프록시 서버를 사용하는 경우 애플리케이션을 올바르게 구성했는지 확인하는 것이 중요합니다.

예를 들어, 많은 애플리케이션에는 `https://example.com/`에 대한 요청에 응답하여 `http://192.168.1.107`의 애플리케이션 서버로 요청을 전달하는 로드 밸런서가 있습니다.

적절한 구성이 없으면 애플리케이션 서버는 로드 밸런서가 존재한다는 것을 알 수 없으며 클라이언트가 `http://192.168.1.107:8080`을 요청한 것처럼 요청을 처리합니다.

이를 해결하기 위해 RFC 7239를 사용하여 로드 밸런서가 사용 중임을 지정할 수 있습니다.

애플리케이션이 이를 인식하도록 하려면 `X-Forwarded` 헤더를 인식하도록 애플리케이션 서버를 구성해야 합니다.

예를 들어 Tomcat은 `RemoteIpValve`를 사용하고 Jetty는 `ForwardedRequestCustomizer`를 사용합니다.

또는 Spring 사용자는 서블릿 스택에서 `ForwardedHeaderFilter`를 사용하거나 리액티브 스택에서 `ForwardedHeaderTransformer`를 사용할 수 있습니다.

Spring Boot 사용자는 `server.forward-headers-strategy` 속성을 사용하여 애플리케이션을 구성할 수 있습니다.

---

### 🌐 인터넷 통신, 안전하게 주고받자! (HTTP와 HTTPS 이해하기)

우리가 웹사이트를 볼 때, 우리 컴퓨터(클라이언트)와 웹사이트가 있는 컴퓨터(서버)는 서로 정보를 주고받아요.

이때 사용하는 약속이 바로 **HTTP (HyperText Transfer Protocol)**입니다. 그런데 HTTP는 마치 우리가 엽서를 보내는 것과 같아요.

중간에 누가 훔쳐보면 내용을 다 볼 수 있죠.

즉, **암호화되지 않아서 보안에 취약**합니다.

그래서 등장한 것이 **HTTPS (HTTP Secure)**! HTTPS는 HTTP에 **TLS (Transport Layer Security)**라는 암호화 기술을 더한 거예요.

이건 마치 우리가 비밀 편지를 암호 봉투에 넣어 보내는 것과 같아서, 중간에 누가 가로채도 내용을 알 수 없게 만들죠.

**로그인 정보, 개인 정보, 금융 정보 등 중요한 데이터를 주고받을 때는 반드시 HTTPS를 사용해야 합니다!**

### 1. Spring Security와 HTTPS의 관계 🤝

- **Spring Security는 직접 HTTPS 연결을 만들지 않아요.**
  - HTTPS 연결 설정(인증서 설치, 웹 서버 설정 등)은 웹 서버(Tomcat, Nginx 등)나 클라우드 서비스(AWS, Azure 등)의 역할입니다. Spring Security는 애플리케이션 레벨의 보안 프레임워크이기 때문이죠.
- **하지만 HTTPS 사용을 적극적으로 도와줘요!**
  - Spring Security는 애플리케이션이 HTTPS를 안전하고 올바르게 사용하도록 여러 가지 편리한 기능을 제공합니다.

### 2. "얘야, HTTP 말고 HTTPS로 오렴!" (HTTPS로 리디렉션 ↪️)

사용자가 실수로 `http://mybank.com` (HTTP)으로 접속하려고 할 때, Spring Security는 자동으로 `https://mybank.com` (HTTPS) 주소로 바꿔서 다시 접속하도록 만들 수 있어요.

이걸 **HTTPS 리디렉션**이라고 합니다.

- **왜 필요할까요?** 사용자가 항상 `https://`를 붙여서 주소를 입력하리란 보장이 없으니까요. 이 기능을 통해 모든 접속을 안전한 HTTPS로 유도할 수 있습니다.
- Spring Security는 서블릿 환경과 웹플럭스 환경 모두에서 이 기능을 설정할 수 있도록 지원합니다.

### 3. "우리 사이트는 무조건 HTTPS만 써!" (엄격한 전송 보안 - HSTS 🛡️)

`Strict-Transport-Security` 헤더
Spring Security는 이 HSTS 기능을 **기본적으로 활성화**해서, 한번 HTTPS로 접속했던 사용자의 브라우저가 다음부터는 무조건 HTTPS로만 우리 사이트에 접속하도록 강제합니다.

- **왜 중요할까요?** HTTPS 리디렉션만으로는 첫 번째 HTTP 요청이 중간자 공격에 노출될 수 있는 아주 짧은 순간이 존재해요. HSTS는 이 위험까지 줄여줍니다.

### 4. "저 앞에 경비원(프록시) 있는데요?" (프록시 서버 환경 설정 🚦)

요즘 웹사이트들은 대부분 **로드 밸런서**나 **리버스 프록시** 같은 중간 장비를 사용해요. 이 장비들이 사용자 요청을 받아서 내부의 여러 웹 서버(애플리케이션 서버)로 나눠주는 역할을 하죠.

- **일반적인 구성:**
  - 사용자 ➡️ `https://example.com` (로드 밸런서, HTTPS) ➡️ `http://192.168.1.100` (내부 앱 서버, HTTP)
  - 즉, 외부 사용자와 로드 밸런서 사이는 암호화된 HTTPS 통신을 하지만, 로드 밸런서와 내부 앱 서버 사이는 암호화되지 않은 HTTP 통신을 하는 경우가 많습니다. (내부망은 비교적 안전하다고 가정하고, 암호화/복호화 부하를 줄이기 위해)
- **문제점:**
  - 이렇게 되면, 내부 앱 서버는 사용자가 `http://192.168.1.100`으로 직접 요청한 것처럼 착각할 수 있어요.
  - 그러면 앱 서버는 "어? 사용자가 HTTP로 요청했네? 그럼 나도 HTTP로 응답해야지!" 하면서, HTTPS로 리디렉션해야 할 상황에서도 안 하거나, HSTS 헤더를 보내지 않거나, CSRF 토큰의 `secure` 플래그를 잘못 설정하는 등 보안상 문제가 생길 수 있습니다.
- **해결책: "X-Forwarded" 헤더 사용!**
  - 로드 밸런서 같은 프록시 장비는 원래 클라이언트가 어떤 프로토콜(`X-Forwarded-Proto: https`), 어떤 호스트(`X-Forwarded-Host: example.com`), 어떤 IP(`X-Forwarded-For: 클라이언트IP`)로 요청했는지 알려주는 특별한 HTTP 헤더들(`X-Forwarded-*` 헤더)을 내부 앱 서버로 전달해줍니다. (RFC 7239 표준)
  - **앱 서버(또는 Spring)가 이 헤더들을 이해하도록 설정**해주면, 앱 서버는 "아! 원래 사용자는 `https://example.com`으로 요청했었구나!" 하고 올바르게 상황을 인지하고 HTTPS 관련 보안 기능들을 제대로 작동시킬 수 있습니다.
- **Spring에서 설정하는 방법:**
  - **서블릿 환경:** `ForwardedHeaderFilter` 사용
  - **리액티브 환경:** `ForwardedHeaderTransformer` 사용
  - **Spring Boot:** `application.properties` 또는 `application.yml` 파일에 `server.forward-headers-strategy=NATIVE` (웹 서버 자체 기능 사용) 또는 `server.forward-headers-strategy=FRAMEWORK` (Spring의 필터/트랜스포머 사용) 같은 설정을 추가하면 쉽게 해결할 수 있습니다.
  - 웹 서버 자체(Tomcat의 `RemoteIpValve`, Jetty의 `ForwardedRequestCustomizer`)에서 이 헤더를 처리하도록 설정할 수도 있습니다.

**핵심은, 프록시를 사용한다면 애플리케이션이 실제 클라이언트 요청 정보를 올바르게 알 수 있도록 반드시 설정을 해줘야 한다는 것입니다!**

---

**HSTS는 "HTTP 엄격한 전송 보안"**의 약자입니다. 이름에서 알 수 있듯이, 웹사이트가 브라우저에게 **"앞으로는 무조건 HTTPS로만 나랑 통신해야 해! HTTP는 절대 안 돼!"** 라고 강력하게 지시하는 보안 기능입니다.

**HSTS가 필요한 이유: HTTPS 리디렉션의 빈틈**

보통 웹사이트는 사용자가 `http://example.com` (HTTP)으로 접속하면, `https://example.com` (HTTPS)으로 자동으로 바꿔주는 "HTTPS 리디렉션" 기능을 사용합니다. 이건 좋은 방법이지만, 한 가지 작은 빈틈이 있어요.

1. **사용자:** 주소창에 `example.com` 입력 (또는 HTTP 링크 클릭)
2. **브라우저:** `http://example.com`으로 요청 시도
3. **중간자 공격자 (해커):** 이 첫 번째 HTTP 요청을 가로챔! 😈
  - 해커는 사용자에게 가짜 로그인 페이지를 보여주거나, 악성 코드를 심거나, 다른 위험한 사이트로 보내버릴 수 있습니다.
4. **(만약 공격이 없다면) 서버:** "어? HTTP로 왔네? HTTPS로 다시 와!" 하면서 `https://example.com`으로 리디렉션 응답을 보냄.
5. **브라우저:** `https://example.com`으로 다시 요청 (이제부터 안전한 HTTPS 통신 시작)

문제는 바로 **3번 단계**입니다. 아주 짧은 순간이지만, **첫 번째 HTTP 요청이 암호화되지 않은 상태로 전송되기 때문에 중간자 공격(Man-in-the-Middle Attack, MITM Attack)에 노출**될 수 있다는 것입니다.

**HSTS의 작동 원리: "기억해! 다음부터는 무조건 HTTPS야!"**

HSTS는 이 빈틈을 메워줍니다.

1. **첫 방문 (또는 HSTS 설정 후 첫 HTTPS 방문):**
  - 사용자가 `https://example.com` (HTTPS)으로 정상적으로 접속합니다.
  - 이때 서버는 HTTP 응답 헤더에 **`Strict-Transport-Security`** 라는 특별한 헤더를 포함해서 보냅니다.

      ```
      Strict-Transport-Security: max-age=31536000; includeSubDomains
      
      ```

    - `max-age=31536000`: "앞으로 31536000초(약 1년) 동안 이 규칙을 기억해!" (이 시간 동안은 무조건 HTTPS)
    - `includeSubDomains` (선택 사항): "이 도메인뿐만 아니라 모든 하위 도메인(예: `blog.example.com`, `shop.example.com`)도 이 규칙을 적용해!"
2. **브라우저의 행동:**
  - 브라우저는 이 `Strict-Transport-Security` 헤더를 보고, "아하! `example.com`은 앞으로 1년 동안 무조건 HTTPS로만 접속해야 하는구나!" 라고 **기억**합니다. (브라우저 내부에 HSTS 정책 목록을 저장)
3. **다음 방문 (max-age 기간 내):**
  - 사용자가 다시 `example.com` (또는 `http://example.com`)을 주소창에 입력하거나 HTTP 링크를 클릭합니다.
  - 브라우저는 "잠깐! `example.com`은 HSTS 정책이 설정된 곳이잖아? 무조건 HTTPS로 바꿔서 요청해야 해!" 라고 판단하고, **자동으로 내부에서 `http://`를 `https://`로 변경**하여 `https://example.com`으로 **처음부터 HTTPS 요청**을 보냅니다.
  - 따라서, **첫 번째 HTTP 요청 자체가 발생하지 않으므로** 중간자 공격의 위험이 사라집니다!

**HSTS의 장점:**

- **중간자 공격(SSL Stripping 등) 방어 강화:** 사용자가 실수로 HTTP로 접속하려는 시도를 브라우저 단에서 차단하고 강제로 HTTPS로 전환하여 보안을 강화합니다.
- **사용자 편의성:** 사용자는 `https://`를 매번 입력할 필요 없이 사이트 주소만 입력해도 안전하게 접속됩니다.
- **성능 약간 향상:** HTTP로 접속 후 HTTPS로 리디렉션되는 과정이 생략되므로 아주 약간의 성능 향상 효과도 있을 수 있습니다.

**HSTS Preloading (사전 로딩): 더 강력한 보호**

`Strict-Transport-Security` 헤더에 `preload` 지시문을 추가하고, [hstspreload.org](https://hstspreload.org/) 와 같은 사이트를 통해 신청하면, 해당 도메인을 **브라우저 자체에 내장된 HSTS 목록**에 포함시킬 수 있습니다.

- **장점:** 사용자가 해당 웹사이트를 **생애 처음 방문할 때부터** 브라우저는 HSTS 정책을 알고 있어, 첫 HTTPS 연결을 통해 헤더를 받기 전에도 안전하게 HTTPS로만 접속합니다. 가장 강력한 HSTS 보호 방법입니다.
- **주의점:** 한번 Preload 목록에 등록되면 제거하기가 매우 어렵고 시간이 오래 걸리므로, 사이트 전체가 HTTPS로 완벽하게 운영될 준비가 되었을 때 신중하게 결정해야 합니다.

**요약하자면, HSTS는:**

- 웹사이트가 브라우저에게 "앞으로 지정된 기간 동안은 무조건 HTTPS로만 접속하라"고 알리는 보안 메커니즘입니다.
- 첫 HTTP 요청 시 발생할 수 있는 중간자 공격의 위험을 줄여줍니다.
- `Strict-Transport-Security` HTTP 응답 헤더를 통해 설정됩니다.
- `max-age` (필수), `includeSubDomains` (선택), `preload` (선택, 강력) 등의 지시문을 가집니다.
