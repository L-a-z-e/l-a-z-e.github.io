---
title: '@PathVariable, @RequestParam, @RequestPart, @Param'
description: 
author: laze
date: 2025-06-29 00:00:01 +0900
categories: [Dev, Java]
tags: [Java]
---
Spring MVC에서 자주 사용되는 주요 애노테이션들인 `@PathVariable`, `@RequestParam`, `@RequestPart`, `@Param`에 대한 **정리와 차이점**, 그리고 언제 어떤 상황에 쓰는 게 맞는지에 대한 **실전 위주 설명**입니다.

---

## ✅ 1. `@PathVariable`

### 📌 개념

- URL 경로(path)에 포함된 값을 변수로 바인딩할 때 사용
- RESTful API 경로에서 자주 사용됨

### 📘 사용 예시

```java
@GetMapping("/users/{id}")
public ResponseEntity<User> getUserById(@PathVariable Long id) {
    return userService.findById(id);
}
```

### 🔍 정리

| 항목 | 설명 |
| --- | --- |
| 데이터 출처 | URL 경로 (`/users/{id}`) |
| 바인딩 방식 | 경로에서 변수 추출 |
| 사용 예 | 리소스 조회 `/products/{productId}` |

---

## ✅ 2. `@RequestParam`

### 📌 개념

- 쿼리 스트링 (URL?name=value) 또는 폼 데이터에서 값을 가져올 때 사용

### 📘 사용 예시

```java
@GetMapping("/search")
public ResponseEntity<?> search(@RequestParam String keyword, @RequestParam(defaultValue = "1") int page) {
    return service.search(keyword, page);
}
```

**호출 예:** `/search?keyword=phone&page=2`

### 🔍 정리

| 항목 | 설명 |
| --- | --- |
| 데이터 출처 | URL 쿼리 파라미터 또는 `application/x-www-form-urlencoded` |
| 바인딩 방식 | `?key=value` 형식 |
| 사용 예 | 검색, 필터링, 페이징 |

---

## ✅ 3. `@RequestPart`

### 📌 개념

- `multipart/form-data` 요청에서 파일과 JSON 등 다른 파트를 분리하여 받을 때 사용
- 파일 업로드와 함께 JSON 객체를 동시에 받을 때 유용

### 📘 사용 예시

```java
@PostMapping("/upload")
public ResponseEntity<?> upload(
    @RequestPart("file") MultipartFile file,
    @RequestPart("metadata") UploadMetadata metadata
) {
    return service.handleUpload(file, metadata);
}
```

### 🔍 정리

| 항목 | 설명 |
| --- | --- |
| 데이터 출처 | `multipart/form-data` 요청 바디 |
| 바인딩 방식 | 각 파트를 이름 기준으로 바인딩 |
| 사용 예 | 파일 + JSON 같이 받을 때 |

---

## ✅ 4. `@Param` (주의!)

### 📌 개념

- `Spring MVC`에서는 **거의 사용 안 함**
- *JPA (특히 @Query)**에서 **JPQL 내에서 쿼리 파라미터 바인딩**할 때 사용

### 📘 사용 예시 (JPA)

```java
@Query("SELECT u FROM User u WHERE u.name = :name")
List<User> findByName(@Param("name") String name);
```

### 🔍 정리

| 항목 | 설명 |
| --- | --- |
| 데이터 출처 | JPQL 쿼리 내 변수 |
| 바인딩 방식 | `:name` 과 Java 메서드 파라미터를 연결 |
| 사용 예 | Spring Data JPA의 @Query 내 파라미터 |

---

## ✅ 비교 요약표

| 애노테이션 | 요청 형식 예시 | 용도 | 주로 쓰는 곳 |
| --- | --- | --- | --- |
| `@PathVariable` | `/users/5` | URL 경로에서 값 추출 | RESTful API |
| `@RequestParam` | `/search?keyword=test&page=2` | 쿼리 스트링 값 받기 | 검색, 필터 등 |
| `@RequestPart` | `multipart/form-data` | 파일 + JSON 파트 분리 | 파일 업로드 + 데이터 전달 |
| `@Param` | JPQL: `@Query("... WHERE x = :x")` | JPA 쿼리 파라미터 바인딩 | Spring Data JPA (@Query) |

---

## 💡 언제 어떤 걸 써야 할까?

| 상황 | 추천 애노테이션 | 이유 |
| --- | --- | --- |
| RESTful 리소스 조회 | `@PathVariable` | 리소스를 URL로 식별 |
| 검색 조건 전달 | `@RequestParam` | 가변적인 필터/조건 전달 |
| 파일 업로드 + JSON | `@RequestPart` | multipart에서 JSON+파일 분리 |
| JPA 직접 쿼리 | `@Param` | `@Query` 내 파라미터 필요 |

---

## 💬 보너스: 혼용 가능?

- `@PathVariable`, `@RequestParam`, `@RequestPart`는 **서로 혼용 가능하나 목적에 맞게 구분해서 써야** 한다.
- 예를 들어:

    ```java
    @PostMapping("/users/{id}/profile")
    public ResponseEntity<?> updateProfile(
        @PathVariable Long id,
        @RequestPart("image") MultipartFile file
    )
    ```


---
