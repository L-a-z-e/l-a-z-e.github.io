---
title: Spring Core - AOP (LoggingAspect)
description: 
author: laze
date: 2025-06-29 00:00:02 +0900
categories: [Dev, SpringBoot]
tags: [SpringBoot]
---
### 기본 구조 생성

**ArticleController.java**

```java
package com.laze.aopboard.controller;

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
    public ResponseEntity<ArticleDto> createArticle(@Valid @RequestBody ArticleDto articleDto) {
        // @Valid: articleDto에 대한 유효성 검사 수행
        // @RequestBody: 요청 본문의 JSON을 ArticleDto 객체로 변환
        ArticleDto createdArticle = articleService.createArticle(articleDto);
        return ResponseEntity.ok(createdArticle); // 200 OK와 함께 생성된 게시글 정보 반환
    }

    // 4. 특정 게시글 삭제 API
    @DeleteMapping("/{id}")
    public ResponseEntity<Void> deleteArticle(@PathVariable Long id) {
        articleService.deleteArticle(id);
        return ResponseEntity.noContent().build(); // 204 No Content (성공적으로 처리했으나 반환할 내용 없음)
    }
}
```

**ArticleService.java**

```java
package com.laze.aopboard.service;

import com.laze.aopboard.dto.ArticleDto;
import com.laze.aopboard.repository.ArticleRepository;
import lombok.RequiredArgsConstructor;
import org.springframework.stereotype.Service;

import java.util.List;

@Service
@RequiredArgsConstructor
public class ArticleService {

    private final ArticleRepository articleRepository;

    public List<ArticleDto> getArticles() {
        return articleRepository.findAll();
    }

    public ArticleDto getArticleById(Long id) {
        return articleRepository.findById(id).orElse(null);
    }

    public ArticleDto createArticle(ArticleDto articleDto) {
        articleDto.setId(null);
        return articleRepository.save(articleDto);
    }

    public void deleteArticle(Long id) {
        articleRepository.deleteById(id);
    }

}
```

**ArticleRepository.java**

```java
package com.laze.aopboard.repository;

import com.laze.aopboard.dto.ArticleDto;
import org.springframework.stereotype.Repository;

import java.util.*;
import java.util.concurrent.ConcurrentHashMap;

@Repository
public class ArticleRepository {

    private final Map<Long, ArticleDto> articleStore = new HashMap<>();
    private long sequence = 0L;

    public List<ArticleDto> findAll() {
        return new ArrayList<>(articleStore.values());
    }

    public Optional<ArticleDto> findById(Long id) {
        return Optional.ofNullable(articleStore.get(id));
    }

    // 게시글 저장 (생성 및 수정)
    public ArticleDto save(ArticleDto articleDto) {
        if (articleDto.getId() == null || articleDto.getId() == 0L) {
            // 새 게시글 생성
            articleDto.setId(++sequence);
            articleStore.put(articleDto.getId(), articleDto);
            return articleDto;
        } else {
            // 기존 게시글 수정
            articleStore.put(articleDto.getId(), articleDto);
            return articleDto;
        }
    }

    // 게시글 삭제
    public void deleteById(Long id) {
        articleStore.remove(id);
    }
}
```

**ArticleDto.java**

```java
package com.laze.aopboard.dto;

import jakarta.validation.constraints.NotBlank;
import jakarta.validation.constraints.Size;
import lombok.*;

@Getter
@Setter
@AllArgsConstructor
@NoArgsConstructor
@ToString
public class ArticleDto {

    private Long id;

    @NotBlank(message = "Title cannot be blank")
    @Size(min = 3, message = "Title must be at least 3 characters long")
    private String title;

    @NotBlank(message = "Content cannot be blank")
    private String content;

}
```

**LoggingAspect.java**

```java
package com.laze.aopboard.aop;

import lombok.extern.slf4j.Slf4j;
import org.aspectj.lang.JoinPoint;
import org.aspectj.lang.annotation.Aspect;
import org.aspectj.lang.annotation.Before;
import org.aspectj.lang.annotation.Pointcut;
import org.springframework.stereotype.Component;

import java.util.Arrays;
import java.util.stream.Collectors;

@Aspect
@Component
@Slf4j
public class LoggingAspect {
    
    @Pointcut("within(com.laze.aopboard.controller..*)")
    public void controllerLayer() {}
    
    @Before("controllerLayer()")
    public void logBefore(JoinPoint joinPoint) {
        String className = joinPoint.getTarget().getClass().getName();
        String methodName = joinPoint.getSignature().getName();
        
        String args = Arrays.stream(joinPoint.getArgs())
                .map(Object::toString)
                .collect(Collectors.joining(", ")
                );

        log.info("[Request Log] Controller: {}, Method: {}, Arguments: [{}]", className, methodName, args);
    }

}
```

- Pointcut
  - within ↔ execution
  - **`execution`:** 가장 일반적으로 사용되며, **메소드 실행 조인 포인트**를 매칭합니다. (스프링 AOP의 핵심)
    - 형식: `execution(수식어패턴? 리턴타입패턴 선언타입패턴?메소드이름패턴(파라미터패턴) 예외패턴?)`
    - 예: `execution(public * com.example.service.*.*(..))`
      - `public`: public 접근 제어자 (생략 가능, 생략 시 모든 접근 제어자)
      - : 모든 리턴 타입
      - `com.example.service.*`: `com.example.service` 패키지의 모든 클래스
      - `.*`: 모든 메소드 이름
      - `(..)`: 0개 이상의 모든 파라미터
  - `within`: 특정 타입 내의 모든 조인 포인트를 매칭합니다.
    - 예: `within(com.example.service.*)`
- Advice
  - **@Before("controllerLayer()")**: 컨트롤러의 메소드가 실행되기 **직전**에 logBefore 메소드를 실행하도록 합니다.
- JoinPoint
  - Advice의 매개변수로 JoinPoint를 선언하면, 스프링 AOP가 이 객체를 통해 현재 조인 포인트(실행되는 메소드)에 대한 다양한 정보(메소드 시그니처, 인자 등)를 전달해 줍니다.

```
2025-06-29T14:08:57.758+09:00  INFO 23614 --- [aop-board] [io-50001-exec-1] com.laze.aopboard.aop.LoggingAspect      : [Request Log] Controller: com.laze.aopboard.controller.ArticleController, Method: getAllArticles, Arguments: []
2025-06-29T14:11:02.607+09:00  INFO 23614 --- [aop-board] [io-50001-exec-5] com.laze.aopboard.aop.LoggingAspect      : [Request Log] Controller: com.laze.aopboard.controller.ArticleController, Method: createArticle, Arguments: [ArticleDto(id=null, title=My First AOP Article, content=This is a test article for AOP logging.)]
2025-06-29T14:11:14.659+09:00  INFO 23614 --- [aop-board] [io-50001-exec-6] com.laze.aopboard.aop.LoggingAspect      : [Request Log] Controller: com.laze.aopboard.controller.ArticleController, Method: getAllArticles, Arguments: []
```
