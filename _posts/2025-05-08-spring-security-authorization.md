---
title: Authorization
description: 
author: laze
date: 2025-05-08 00:00:03 +0900
categories: [Dev, SpringSecurity]
tags: [SpringSecurity]
---
# Authorization

## 권한 부여 (Authorization)

Spring Security는 권한 부여에 대한 포괄적인 지원을 제공합니다. 권한 부여는 특정 리소스에 누가 접근할 수 있도록 허용될 것인지 결정하는 것입니다. Spring Security는 요청 기반 권한 부여와 메소드 기반 권한 부여를 허용함으로써 심층 방어(defense in depth)를 제공합니다.

### 요청 기반 권한 부여

Spring Security는 서블릿(Servlet) 및 웹플럭스(WebFlux) 환경 모두에 대해 요청에 기반한 권한 부여를 제공합니다.

### 메소드 기반 권한 부여

Spring Security는 서블릿(Servlet) 및 웹플럭스(WebFlux) 환경 모두에 대해 메소드 호출에 기반한 권한 부여를 제공합니다.
