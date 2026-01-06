---
title: Docker 환경에서 Spring OAuth2 Resource Server의 JWT 검증 문제 해결기
description: 
author: laze
date: 2026-01-06 00:00:01 +0900
categories: [Dev, Docker]
tags: [OAuth2, JWT, Docker]
---
# Docker 환경에서 Spring OAuth2 Resource Server의 JWT 검증 문제 해결기

## 들어가며

마이크로서비스 아키텍처로 프로젝트를 진행하면서 로컬에서는 잘 동작하던 인증이 Docker 환경에서는 계속 401 에러를 뱉어냈습니다. 몇 시간을 삽질한 끝에 얻은 교훈을 공유합니다.

## 문제 상황

Portal Universe라는 프로젝트를 진행 중입니다. Auth Service에서 JWT를 발급하고, Blog Service 같은 다른 마이크로서비스들이 이 토큰을 검증하는 구조입니다. 로컬에서는 문제없이 동작했는데, `docker-compose up`으로 올리니 Blog API 호출 시 401 Unauthorized가 발생했습니다.

### 에러 로그

```
org.springframework.security.oauth2.jwt.JwtDecoderInitializationException: 
Failed to lazily resolve the supplied JwtDecoder instance

Caused by: org.springframework.web.client.ResourceAccessException: 
I/O error on GET request for 
"https://portal-universe:30000/auth-service/.well-known/openid-configuration": 
Connection refused
```

처음에는 "JWT 해석 실패구나" 정도로만 생각했는데, 자세히 보니 JwtDecoder 자체를 초기화하지 못하는 문제였습니다.[1]

## 근본 원인: OIDC Discovery의 함정

Spring Security OAuth2 Resource Server는 똑똑한 자동 설정을 제공합니다. `application.yml`에 issuer-uri만 설정하면:

```yaml
spring:
  security:
    oauth2:
      resourceserver:
        jwt:
          issuer-uri: https://portal-universe:30000/auth-service
```

내부적으로 이런 흐름을 타게 됩니다:

1. `{issuer-uri}/.well-known/openid-configuration` 호출
2. Discovery Document에서 `jwks_uri` 찾기
3. Public Key 다운로드
4. JwtDecoder 생성

문제는 Blog Service **컨테이너 내부**에서 `https://portal-universe:30000`에 접근하려 한다는 점입니다.

### 왜 문제가 될까?

Docker 네트워크의 관점을 이해해야 합니다:[2][3]

**브라우저 관점** (호스트 머신):
- `portal-universe:30000`은 `/etc/hosts`에서 `127.0.0.1`로 매핑됨
- API Gateway의 NodePort로 정상 접속 가능

**Blog Service 컨테이너 관점**:
- `portal-universe:30000`을 DNS로 해석할 수 없음
- 설령 해석되더라도 포트가 맞지 않음 (Auth Service는 내부적으로 8081 사용)
- 결과: Connection refused

## 해결책: jwk-set-uri 직접 지정

핵심은 **Public Key를 가져오는 경로**와 **JWT의 issuer 검증**을 분리하는 것입니다.

### application-docker.yml 수정

```yaml
spring:
  security:
    oauth2:
      resourceserver:
        jwt:
          # Docker 네트워크 내부 주소로 Public Key 다운로드
          jwk-set-uri: http://auth-service:8080/oauth2/jwks
          # JWT의 iss 클레임 검증용 (외부 주소)
          issuer-uri: https://portal-universe:30000/auth-service
```

이렇게 하면:
- Blog Service가 시작할 때 `http://auth-service:8080/oauth2/jwks`에서 Public Key 다운로드 (컨테이너 내부 통신)
- JWT를 받으면 캐시된 Public Key로 서명 검증
- JWT의 `iss` 클레임이 `https://portal-universe:30000/auth-service`인지 확인

### 또는 JwtDecoder Bean 직접 생성

더 세밀한 제어가 필요하다면:

```java
@Bean
public JwtDecoder jwtDecoder() {
    NimbusJwtDecoder decoder = NimbusJwtDecoder
        .withJwkSetUri("http://auth-service:8080/oauth2/jwks")
        .build();
    
    OAuth2TokenValidator<Jwt> withIssuer = 
        JwtValidators.createDefaultWithIssuer(
            "https://portal-universe:30000/auth-service"
        );
    
    decoder.setJwtValidator(withIssuer);
    return decoder;
}
```

## 이해를 돕기 위한 개념 정리

### issuer의 두 가지 역할

1. **외부 접근**: 브라우저가 로그인할 때 사용하는 URL
2. **신뢰 검증**: JWT의 `iss` 클레임으로 "이 토큰이 우리가 신뢰하는 Auth Service에서 온 게 맞나?" 확인

### jwks_uri의 역할

Public Key를 배포하는 엔드포인트입니다. 중요한 점은 "검증을 대신 해주는 서버"가 아니라 **"도장 대조본을 다운로드하는 곳"**이라는 점입니다.

비대칭 암호화의 마법:
- Auth Service: Private Key로 JWT 서명
- Resource Service: Public Key로 서명 검증 (로컬에서 수행, 네트워크 호출 없음)
- Public Key는 공개되어도 안전 (읽기 전용 검증만 가능)

## 추가로 해결한 것들

### SecurityConfig 수정

`oauth2ResourceServer` 설정이 주석 처리되어 있거나 커스텀 Converter가 연결되지 않은 경우도 있었습니다:

```java
@Bean
@Order(2)
public SecurityFilterChain defaultSecurityFilterChain(
    HttpSecurity http,
    JwtDecoder jwtDecoder
) throws Exception {
    http
        .authorizeHttpRequests(authorize -> authorize
            .requestMatchers("/api/admin").hasRole("ADMIN")
            .anyRequest().authenticated()
        )
        .oauth2ResourceServer(server -> server
            .jwt(jwt -> jwt
                .decoder(jwtDecoder)
                .jwtAuthenticationConverter(
                    JwtAuthenticationConverterAdapter.createDefault()
                )
            )
        );
    return http.build();
}
```

### JWT에서 roles 추출

기본 설정은 `scope` 클레임만 읽습니다. 커스텀 권한 체계를 사용한다면 Converter가 필요합니다:[1]

```java
public static JwtAuthenticationConverter createDefault() {
    JwtGrantedAuthoritiesConverter grantedAuthoritiesConverter =
        new JwtGrantedAuthoritiesConverter();
    grantedAuthoritiesConverter.setAuthoritiesClaimName("roles");
    grantedAuthoritiesConverter.setAuthorityPrefix(""); // 이미 ROLE_ 포함됨
    
    JwtAuthenticationConverter converter = new JwtAuthenticationConverter();
    converter.setJwtGrantedAuthoritiesConverter(grantedAuthoritiesConverter);
    return converter;
}
```

## 배운 점

1. **에러 로그 제대로 읽기**: `JwtDecoderInitializationException`은 JWT 해석 실패가 아니라 초기화 실패입니다.

2. **Lazy Initialization 이해**: Spring Security는 첫 요청 시 JwtDecoder를 초기화합니다. 애플리케이션이 시작되었다고 설정이 올바른 건 아닙니다.

3. **네트워크 관점 분리**: 브라우저 접근 경로 ≠ 컨테이너 간 통신 경로. 이 차이를 이해해야 issuer와 jwk-set-uri를 올바르게 설정할 수 있습니다.[4]

4. **OIDC Discovery는 만능이 아니다**: 간단한 환경에선 편리하지만, 복잡한 네트워크 구조에선 수동 설정이 더 명확합니다.

## 마치며

로컬에서 잘 되던 게 Docker에서 안 되는 경험, 다들 한 번쯤 겪어보셨을 겁니다. 특히 인증/인가 관련은 네트워크 경로, 포트, 도메인 등 변수가 많아서 더 까다롭습니다.

이 글이 비슷한 문제로 고민하시는 분들께 도움이 되길 바랍니다. 질문이나 더 나은 해결책이 있다면 댓글로 알려주세요!

***

**참고 자료**:
- Spring Security OAuth2 Resource Server 공식 문서
- Docker 네트워크 가이드
- Zipkin/Prometheus Docker 설정

**관련 코드**: [GitHub - portal-universe](https://github.com/L-a-z-e/portal-universe)

[1](https://docs.spring.io/spring-security/reference/servlet/oauth2/resource-server/jwt.html)
[2](https://docs.docker.com/engine/network/port-publishing/)
[3](https://www.baeldung.com/ops/assign-port-docker-container)
[4](https://stackoverflow.com/questions/74877968/zipkin-not-working-in-docker-conncection-refused)
