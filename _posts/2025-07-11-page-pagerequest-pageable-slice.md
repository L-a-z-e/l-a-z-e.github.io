---
title: Page, Pageable, Slice 완벽 정복 - 페이징 API 실전 가이드
description: 
author: laze
date: 2025-07-11 00:00:05 +0900
categories: [Dev, SpringBoot]
tags: [SpringBoot, Page, Paging, Pageable, Slice]
---
# [Spring Data JPA] Page, Pageable, Slice 완벽 정복: 페이징 API 실전 가이드

## 1. 서론: `findAll()`의 함정과 페이징의 필요성

데이터베이스에 100만 건의 게시글 데이터가 저장되어 있다고 상상해보자.

이 모든 게시글 목록을 클라이언트에게 보여주기 위해, `postRepository.findAll()`을 호출하면 어떤 일이 벌어질까?

애플리케이션은 100만 개의 `Post` 객체를 모두 메모리에 올리려다 `OutOfMemoryError`를 뿜으며 비참하게 종료될 것이다.

이처럼 대용량 데이터를 한 번에 조회하는 것은 매우 위험하다. **페이징(Paging)**은 이 문제를 해결하기 위해, 전체 데이터를 안전하고 효율적으로 분할하여 **"지금 당장 필요한 '한 페이지' 만큼만"** 가져오는 필수적인 기술이다.

이 글에서는 Spring Data JPA가 제공하는 강력한 페이징 기능의 3요소(`Pageable`, `PageRequest`, `Page`)를 알아보고, 성능 최적화를 위한 `Slice`의 활용법, 그리고 실제 API 개발에 적용하는 전체 과정을 구체적인 코드로 알아본다.

## 2. 페이징의 3요소: `Pageable`, `PageRequest`, `Page`

Spring Data JPA의 페이징은 '요청 규격', '요청서', '결과 보고서'라는 명확한 역할 분담을 통해 이루어진다.

### 2.1. `Pageable`: 페이지 요청의 '명세서' (인터페이스)

`Pageable`은 "페이지를 어떻게 가져올 것인가"에 대한 모든 **요청 정보의 규격**을 담은 인터페이스다. 이 자체로는 객체를 만들 수 없으며, 단지 약속일 뿐이다.

```java
public interface Pageable {
    int getPageNumber(); // 조회할 페이지 번호 (0부터 시작)
    int getPageSize();   // 한 페이지에 보여줄 데이터 개수
    Sort getSort();      // 정렬 조건
    // ... 기타 편의 메서드
}
```

### 2.2. `PageRequest`: 페이지 요청의 '구현체' (클래스)

`PageRequest`는 `Pageable` 인터페이스를 구현한, 우리가 실제로 만들어서 사용할 **'요청서' 객체**다.

`PageRequest.of(...)` 라는 정적 팩토리 메서드를 통해 손쉽게 생성할 수 있다.

```java
import org.springframework.data.domain.PageRequest;
import org.springframework.data.domain.Pageable;
import org.springframework.data.domain.Sort;

// 1. 가장 기본적인 요청: 0번째 페이지, 10개씩
Pageable pageable = PageRequest.of(0, 10);

// 2. 정렬 조건 추가: 0번째 페이지, 10개씩, ID 기준 내림차순
Pageable sortedPageable = PageRequest.of(0, 10, Sort.by("id").descending());

// 3. 여러 정렬 조건 추가: 이름은 오름차순, 생성일은 내림차순
Sort multiSort = Sort.by("name").ascending()
                     .and(Sort.by("createdAt").descending());
Pageable multiSortedPageable = PageRequest.of(0, 20, multiSort);
```

이렇게 생성된 `PageRequest` 객체를 Repository 메서드의 파라미터로 전달하면 된다.

### 2.3. `Page<T>`: 페이지 결과의 '종합 보고서' (반환 타입)

`Page<T>`는 Repository가 페이징 요청을 처리한 후 반환하는 **'결과물'**이다. 단순히 데이터 목록(`List<T>`)만 담고 있는 것이 아니라, 페이지네이션 UI를 구현하는 데 필요한 모든 메타데이터를 포함하고 있다.

`Page<T>`의 가장 큰 특징은 **전체 데이터 개수를 알아내기 위한 `count` 쿼리가 추가로 실행**된다는 점이다.

```java
public interface Page<T> extends Slice<T> {
    int getTotalPages();       // 전체 페이지 수 (e.g., 15)
    long getTotalElements();    // 전체 데이터 개수 (e.g., 142)
    <U> Page<U> map(Function<? super T, ? extends U> converter); // DTO 변환 등
}
```

## 3. 실전 예제: 게시판 목록 API 만들기

이제 Controller, Service, Repository 각 계층에서 페이징이 어떻게 연동되는지 전체 흐름을 코드로 살펴보자.

### 3.1. Repository

`JpaRepository`를 상속받기만 하면, `findAll(Pageable pageable)` 메서드를 바로 사용할 수 있다. Spring Data JPA가 알아서 페이징 쿼리(e.g., `LIMIT`, `OFFSET`)를 생성해준다.

```java
// PostRepository.java
import org.springframework.data.jpa.repository.JpaRepository;
import org.springframework.data.domain.Page;
import org.springframework.data.domain.Pageable;

public interface PostRepository extends JpaRepository<Post, Long> {
    // 기본 제공되는 페이징 메서드
    Page<Post> findAll(Pageable pageable);

    // 커스텀 쿼리에도 Pageable 적용 가능
    Page<Post> findByTitleContaining(String title, Pageable pageable);
}
```

### 3.2. Service

Service 계층에서는 Repository로부터 `Page<Entity>`를 받아, `Page<DTO>`로 변환하여 Controller에게 전달하는 역할을 주로 수행한다.

`Page` 객체의 `map()` 메서드를 사용하면 이 변환 과정을 매우 간결하게 처리할 수 있다.

```java
// PostService.java
@Service
@RequiredArgsConstructor
public class PostService {
    private final PostRepository postRepository;

    @Transactional(readOnly = true)
    public Page<PostDto> findPosts(Pageable pageable) {
        Page<Post> postPage = postRepository.findAll(pageable);
        // Page.map()을 사용하여 각 Post 엔티티를 PostDto로 변환한다.
        // 페이지 메타데이터는 그대로 유지된다.
        return postPage.map(PostDto::from);
    }
}
```

여기서 `PostDto::from`은 `Post` 엔티티를 `PostDto`로 변환하는 정적 메서드를 의미한다.

### 3.3. Controller

Controller에서는 클라이언트로부터 오는 HTTP 요청 파라미터(e.g., `?page=1&size=20&sort=id,desc`)를 `Pageable` 객체로 자동 변환하여 받아야 한다.

`@PageableDefault` 어노테이션을 사용하면 기본값을 편리하게 지정할 수 있다.

```java
// PostController.java
@RestController
@RequiredArgsConstructor
public class PostController {
    private final PostService postService;

    @GetMapping("/api/posts")
    public Page<PostDto> getPosts(
        // 클라이언트 요청 파라미터가 없으면 이 기본값을 사용한다.
        @PageableDefault(size = 10, sort = "createdAt", direction = Sort.Direction.DESC)
        Pageable pageable
    ) {
        // 클라이언트가 ?page=0&size=5 요청 시, pageable에는 해당 정보가 담겨 넘어온다.
        return postService.findPosts(pageable);
    }
}
```

클라이언트는 이제 이 API를 통해 페이지 단위로 데이터를 요청하고, 전체 페이지 수와 같은 메타데이터까지 함께 응답받을 수 있다.

## 4. 성능 최적화: `Page` vs `Slice`

"더보기"나 모바일 앱의 '무한 스크롤' 기능을 구현할 때는 전체 페이지 수가 필요 없다. 오직 **"다음 페이지가 있는지 없는지"**만 알면 된다. `Page`는 불필요한 `count` 쿼리를 추가로 실행하여 성능 저하를 유발할 수 있다.

이럴 때 사용하는 것이 바로 `Slice<T>`다.

- **`Slice<T>`:** `count` 쿼리를 실행하지 않는다. `getTotalElements()`, `getTotalPages()` 메서드가 없다. 대신 `hasNext()` 메서드를 통해 다음 페이지의 존재 여부만 알려준다.

| 기능 | `Page<T>` | `Slice<T>` |
| --- | --- | --- |
| `count` 쿼리 실행 | O | X |
| `getContent()` | O | O |
| `getPageable()` | O | O |
| `hasNext()` | O | O |
| `getTotalElements()` | O | **X** |
| `getTotalPages()` | O | **X** |

### `Slice` 사용 예제

`Slice`를 사용하려면 Repository 메서드의 반환 타입을 `Slice<T>`로 선언하기만 하면 된다.

```java
// PostRepository.java
public interface PostRepository extends JpaRepository<Post, Long> {
    // 반환 타입을 Slice로 변경
    Slice<Post> findByTitleContaining(String title, Pageable pageable);
}

// PostService.java
@Transactional(readOnly = true)
public Slice<PostDto> searchPosts(String title, Pageable pageable) {
    Slice<Post> postSlice = postRepository.findByTitleContaining(title, pageable);
    return postSlice.map(PostDto::from);
}

// PostController.java
@GetMapping("/api/posts/search")
public Slice<PostDto> searchPosts(
    @RequestParam String title,
    @PageableDefault(size = 20) Pageable pageable
) {
    return postService.searchPosts(title, pageable);
}
```

이제 클라이언트는 응답받은 `Slice` 객체의 `hasNext` 필드 값을 보고 '더보기' 버튼을 보여줄지 말지를 결정할 수 있다.

## 5. 반드시 알아야 할 기본 지식: Q&A

- **Q: 페이지 번호는 왜 0부터 시작하나요?**
  - A: `offset` 계산(`page * size`)의 편의성 등 기술적인 관례 때문이다. 클라이언트에게는 1부터 시작하는 페이지 번호를 보여주되, 서버에 요청을 보낼 때는 `page - 1` 값을 보내도록 처리하는 것이 일반적이다.
- **Q: `@PageableDefault`는 무엇인가요?**
  - A: 클라이언트가 `page`, `size`, `sort` 파라미터를 명시적으로 보내지 않았을 때, 서버에서 사용할 기본값을 지정하는 편리한 어노테이션이다. API의 안정성을 위해 설정해두는 것이 좋다.

## 6. 결론: 역할에 맞는 도구를 선택하라

Spring Data JPA의 페이징 기능은 매우 강력하고 유연하다. 그 핵심은 역할에 맞는 도구를 선택하는 것이다.

- **요청 정보:** `Pageable` 인터페이스와 `PageRequest` 구현체.
- **결과 정보:** `Page` 또는 `Slice`.

**"전체 페이지 수가 필요한 전통적인 페이지네이션 UI를 만들 때는 `Page`를, '더보기'나 '무한 스크롤'처럼 다음 데이터의 존재 여부만 확인하면 될 때는 성능상 이점이 있는 `Slice`를 사용한다."**

이 원칙만 기억하면, 어떤 페이징 요구사항에도 자신감 있게 대응할 수 있을 것이다.
