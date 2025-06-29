---
title: Spring Core - AOP (AuthAspect)
description: 
author: laze
date: 2025-06-29 00:00:03 +0900
categories: [Dev, SpringBoot]
tags: [SpringBoot]
---
Custom Annotation 생성

```java
package com.laze.aopboard.aop.annotation;

import java.lang.annotation.ElementType;
import java.lang.annotation.Retention;
import java.lang.annotation.RetentionPolicy;
import java.lang.annotation.Target;

@Target(ElementType.METHOD)
@Retention(RetentionPolicy.RUNTIME)
public @interface AuthRequired {
}
```

Custom Exception 생성

```java
package com.laze.aopboard.exception;

public class AuthException extends RuntimeException {
    public AuthException(String message) {
        super(message);
    }
}
```

**AuthAspect.java**

```java
package com.laze.aopboard.aop;

import com.laze.aopboard.exception.AuthException;
import jakarta.servlet.http.HttpServletRequest;
import lombok.extern.slf4j.Slf4j;
import org.aspectj.lang.annotation.Aspect;
import org.aspectj.lang.annotation.Before;
import org.springframework.stereotype.Component;
import org.springframework.util.StringUtils;
import org.springframework.web.context.request.RequestContextHolder;
import org.springframework.web.context.request.ServletRequestAttributes;

@Aspect
@Component
@Slf4j
public class AuthAspect {
    
    @Before("@annotation(com.laze.aopboard.aop.annotation.AuthRequired)")
    public void validateAuthorization() {
        HttpServletRequest request = ((ServletRequestAttributes) RequestContextHolder.currentRequestAttributes()).getRequest();
        
        String authToken = request.getHeader("Authorization");
        log.info("Authorization Token: {}", authToken);
        
        if (!StringUtils.hasText(authToken)) {
            throw new AuthException("Authorization token is required.");
        }
    }
}
```

1. **@Before("@annotation(...)")**:
  - @annotation() 지시자를 사용하여 com.laze.aopboard.aop.annotation.AuthRequired 어노테이션이 붙은 메소드 실행 **직전**에 validateAuthorization 로직을 실행하도록 합니다. 포인트컷을 별도로 정의하지 않고 @Before에 바로 표현식을 작성
2. **RequestContextHolder.currentRequestAttributes()**: 스프링이 제공하는 유틸리티로, 현재 요청의 HttpServletRequest 객체에 접근할 수 있게 해줍니다. AOP 어드바이스는 일반 컨트롤러와 달리 HttpServletRequest를 직접 매개변수로 받기 어려우므로, 이 방법을 사용하면 편리합니다.
3. **request.getHeader("Authorization")**: HTTP 요청 헤더에서 "Authorization" 헤더의 값을 가져옵니다.
4. **if (!StringUtils.hasText(authToken))**: 스프링 유틸리티인 StringUtils.hasText()를 사용하여 토큰 값이 실제로 존재하는지(null, "", " "가 아닌지) 확인합니다.

   Apache의 `StringUtils`는 **풍부하고 안정적인 문자열 유틸리티 메서드**를 제공

   | 메서드 | 설명 |
       | --- | --- |
   | `isEmpty(String)` | `null` 또는 `""`인지 확인 |
   | `isBlank(String)` | `null` 또는 공백 문자열인지 확인 (`"  "` 포함) |
   | `defaultIfEmpty(str, "default")` | `str`이 비어있으면 기본값 반환 |
   | `join(array, ",")` | 문자열 배열을 구분자로 연결 |
   | `split(str, ",")` | 구분자로 문자열 나누기 |
   | `capitalize("abc")` | `Abc`로 변경 |
   | `containsIgnoreCase(...)` | 대소문자 무시하고 포함 여부 확인 |
   | `strip(...)` | 앞뒤 공백 제거 (`trim`보다 범용적) |

Controller → AuthRequired Annotation 추가

```java
package com.laze.aopboard.controller;

import com.laze.aopboard.aop.annotation.AuthRequired;
import com.laze.aopboard.dto.ArticleDto;
import com.laze.aopboard.service.ArticleService;
import jakarta.validation.Valid;
import lombok.RequiredArgsConstructor;
import org.springframework.http.ResponseEntity;
import org.springframework.web.bind.annotation.*;

import java.util.List;

@RestController // @Controller + @ResponseBody
@RequiredArgsConstructor
@RequestMapping("/api/articles") // 이 컨트롤러의 모든 메소드는 /api/articles 로 시작하는 URL을 가짐
public class ArticleController {

    private final ArticleService articleService;

    // 1. 모든 게시글 조회 API
    @GetMapping
    public ResponseEntity<List<ArticleDto>> getAllArticles() {
        List<ArticleDto> articles = articleService.getArticles();
        return ResponseEntity.ok(articles); // 200 OK와 함께 게시글 목록 반환
    }

    // 2. ID로 특정 게시글 조회 API
    @GetMapping("/{id}")
    public ResponseEntity<ArticleDto> getArticleById(@PathVariable Long id) {
        ArticleDto article = articleService.getArticleById(id);
        if (article != null) {
            return ResponseEntity.ok(article); // 200 OK와 함께 게시글 반환
        } else {
            return ResponseEntity.notFound().build(); // 404 Not Found
        }
    }

    // 3. 새 게시글 생성 API
    @PostMapping
    @AuthRequired
    public ResponseEntity<ArticleDto> createArticle(@Valid @RequestBody ArticleDto articleDto) {
        // @Valid: articleDto에 대한 유효성 검사 수행
        // @RequestBody: 요청 본문의 JSON을 ArticleDto 객체로 변환
        ArticleDto createdArticle = articleService.createArticle(articleDto);
        return ResponseEntity.ok(createdArticle); // 200 OK와 함께 생성된 게시글 정보 반환
    }

    // 4. 특정 게시글 삭제 API
    @DeleteMapping("/{id}")
    @AuthRequired
    public ResponseEntity<Void> deleteArticle(@PathVariable Long id) {
        articleService.deleteArticle(id);
        return ResponseEntity.noContent().build(); // 204 No Content (성공적으로 처리했으나 반환할 내용 없음)
    }
}
```

ExceptionHandler

```java
package com.laze.aopboard.exception;

import lombok.Getter;
import org.springframework.http.HttpStatus;
import org.springframework.http.ResponseEntity;
import org.springframework.validation.FieldError;
import org.springframework.web.bind.MethodArgumentNotValidException;
import org.springframework.web.bind.annotation.ExceptionHandler;
import org.springframework.web.bind.annotation.RestControllerAdvice;

import java.util.HashMap;
import java.util.Map;

@RestControllerAdvice
public class GlobalExceptionHandler {

    @ExceptionHandler(AuthException.class)
    public ResponseEntity<ErrorResponse> hadnleAuthException(AuthException e) {
        ErrorResponse errorResponse = new ErrorResponse("AUTH_ERROR", e.getMessage());
        return new ResponseEntity<>(errorResponse, HttpStatus.UNAUTHORIZED); // 401 Unauthorized
    }

    @ExceptionHandler(MethodArgumentNotValidException.class)
    public ResponseEntity<Map<String, String>> handleValidationException(MethodArgumentNotValidException e) {
        Map<String, String> errors = new HashMap<>();
        e.getBindingResult().getAllErrors().forEach(error -> {
           String fieldName = ((FieldError) error).getField();
           String errorMessage = error.getDefaultMessage();
           errors.put(fieldName, errorMessage);
        });

        return new ResponseEntity<>(errors, HttpStatus.BAD_REQUEST);
    }

    @ExceptionHandler(Exception.class)
    public ResponseEntity<ErrorResponse> handleGenericException(Exception e) {
        ErrorResponse errorResponse = new ErrorResponse("INTERNAL_SERVER_ERROR", "An unexpected error occurred.");
        e.getStackTrace();
        return new ResponseEntity<>(errorResponse, HttpStatus.INTERNAL_SERVER_ERROR);
    }

    @Getter
    public static class ErrorResponse {
        private String code;
        private String message;

        public ErrorResponse(String code, String message) {
            this.code = code;
            this.message = message;
        }
    }
}
```
