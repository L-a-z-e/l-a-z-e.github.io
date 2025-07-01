---
title: 오라클 복잡한 집계 및 그룹핑
description: 
author: laze
date: 2025-07-01 00:00:02 +0900
categories: [Dev, Oracle]
tags: [Oracle]
---
### 1. **복잡한 집계 및 그룹핑**

- `GROUP BY CUBE`, `ROLLUP`, `GROUPING SETS`
- 예시: 매출 데이터를 연도/월/제품별, 전체 집계까지 한번에 구하기
- 배울 것: 다차원 분석, OLAP 스타일 리포트 구조
- 예제

  **주제: 복잡한 집계 및 그룹핑**

**OLAP(온라인 분석 처리)** 쿼리나 **보고서 데이터 요약**에 자주 쓰임
    
  ---

  ## ✅ 1. 복잡한 집계 및 그룹핑

  ### 📌 기본 개념 요약

  | 문법 | 설명 |
      | --- | --- |
  | `GROUP BY` | 일반적인 그룹핑 |
  | `GROUP BY ROLLUP` | 위계 구조로 누적 집계 |
  | `GROUP BY CUBE` | 가능한 모든 조합으로 집계 |
  | `GROUPING SETS` | 명시적으로 여러 그룹 기준을 지정 |
    
  ---

  ## ✅ 예제 데이터: 월별 지역별 판매 테이블

    ```sql
    CREATE TABLE sales (
        year     NUMBER,
        month    NUMBER,
        region   VARCHAR2(20),
        amount   NUMBER
    );
    
    -- 예시 데이터 삽입
    INSERT INTO sales VALUES (2024, 1, '서울', 100);
    INSERT INTO sales VALUES (2024, 1, '부산', 200);
    INSERT INTO sales VALUES (2024, 2, '서울', 150);
    INSERT INTO sales VALUES (2024, 2, '부산', 250);
    
    ```
    
  ---

  ## 📘 1-1. 기본 `GROUP BY`

    ```sql
    SELECT year, month, region, SUM(amount) AS total
    FROM sales
    GROUP BY year, month, region;
    ```

  📌 각 `(연도, 월, 지역)` 조합별로 매출을 집계
    
  ---

  ## 📘 1-2. `ROLLUP`: 위계적 누적 집계

    ```sql
    SELECT year, month, region, SUM(amount) AS total
    FROM sales
    GROUP BY ROLLUP (year, month, region);
    ```

  🔍 생성되는 그룹:

  | year | month | region | 의미 |
      | --- | --- | --- | --- |
  | 2024 | 1 | 서울 | 월별/지역별 |
  | 2024 | 1 | 부산 | - |
  | 2024 | 1 | null | 해당 월의 전체 |
  | 2024 | 2 | 서울 | - |
  | 2024 | 2 | 부산 | - |
  | 2024 | 2 | null | 해당 월의 전체 |
  | 2024 | null | null | 연도 전체 합계 |

  🔸 즉, **하위 항목이 `null`로 채워지며 누적적으로 집계**
    
  ---

  ## 📘 1-3. `CUBE`: 모든 조합으로 집계

    ```sql
    SELECT year, month, region, SUM(amount) AS total
    FROM sales
    GROUP BY CUBE (year, month, region);
    
    ```

  🔍 생성되는 그룹 (ROLLUP보다 많음):

  - 연도+월+지역
  - 연도+월
  - 연도+지역
  - 월+지역
  - 연도만
  - 지역만
  - 월만
  - 전체

  → **모든 가능한 조합**을 다 집계
    
  ---

  ## 📘 1-4. `GROUPING SETS`: 직접 조합 지정

    ```sql
    SELECT year, month, region, SUM(amount) AS total
    FROM sales
    GROUP BY GROUPING SETS (
        (year, month, region),
        (year, month),
        (year),
        ()
    );
    
    ```

  🔍 내가 원하는 조합만 명시적으로 집계:

  - 연도+월+지역
  - 연도+월
  - 연도
  - 전체

  → 불필요한 집계 조합 없이 필요한 것만!
    
  ---

  ## 🎓 추가 팁: `GROUPING()` 함수

  `null`이 진짜 null인지, 집계 때문에 만들어진 null인지 구분하는 함수

    ```sql
    SELECT year, month, region,
           GROUPING(year) AS g_year,
           GROUPING(month) AS g_month,
           GROUPING(region) AS g_region,
           SUM(amount) AS total
    FROM sales
    GROUP BY ROLLUP (year, month, region);
    ```
    
  ---

  ## ✅ 정리

  | 기능 | 설명 | 대표 목적 |
      | --- | --- | --- |
  | `ROLLUP` | 누적 요약 | 연도→월→전체 식 요약 |
  | `CUBE` | 전방위 조합 | 다차원 분석 |
  | `GROUPING SETS` | 선택적 조합 | 보고서 용도 최적화 |
  | `GROUPING()` | NULL 판별 | 표시 제어, 필터링 |
    
  ---
