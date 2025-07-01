---
title: Oracle 윈도우 함수
description: 
author: laze
date: 2025-07-01 00:00:03 +0900
categories: [Dev, Oracle]
tags: [Oracle]
---
### 2. **윈도우 함수 (Analytic Function)**

- `ROW_NUMBER()`, `RANK()`, `DENSE_RANK()`, `SUM() OVER`, `LEAD()`, `LAG()`
- 예시: 고객별 구매 순위, 직전 구매일과 차이 계산
- 배울 것: 행 간 비교, 누적 합계, 변화 추적 등l
- 예제

  partition by

  ### ✅ 누적하려는 경우:

    ```sql
    
    SELECT emp_id, month, SUM(sales_amt) OVER (PARTITION BY emp_id ORDER BY month) AS cum_sum
    FROM sales;
    ```

  > 출력 순서를 바꾸는 게 아니라,
  >
  >
  > Oracle이 **“month 순으로 정렬하면서 이전 값들을 계속 합쳐라”**라는 누적 로직을 수행한다.
  >
    
  ---

  ## ✅ 1. 누적 합계 – `SUM() OVER (...)`

    ```sql
    SELECT emp_id, emp_name, month, sales_amt,
           SUM(sales_amt) OVER (PARTITION BY emp_id ORDER BY month) AS cumulative_sales
    FROM sales;
    
    ```

  | emp_id | emp_name | month | sales_amt | cumulative_sales |
      | --- | --- | --- | --- | --- |
  | 1 | 홍길동 | 1 | 100 | 100 |
  | 1 | 홍길동 | 2 | 150 | 250 |
  | 1 | 홍길동 | 3 | 180 | 430 |
  | 2 | 김영희 | 1 | 200 | 200 |
  | 2 | 김영희 | 2 | 250 | 450 |
  | 2 | 김영희 | 3 | 240 | 690 |
    
  ---

  ## ✅ 2. `ROW_NUMBER()` / `RANK()` / `DENSE_RANK()`

  > 동일한 데이터가 있을 때 각각 어떻게 순위를 매기는지 비교해보자.
  >

  ### 📘 예제 데이터

  | emp_id | emp_name | month | sales_amt |
      | --- | --- | --- | --- |
  | 1 | 홍길동 | 1 | 100 |
  | 1 | 홍길동 | 2 | 150 |
  | 1 | 홍길동 | 3 | 150 |
  | 2 | 김영희 | 1 | 200 |
  | 2 | 김영희 | 2 | 250 |
  | 2 | 김영희 | 3 | 250 |

  ### 👇 쿼리:

    ```sql
    SELECT emp_id, emp_name, month, sales_amt,
           ROW_NUMBER() OVER (PARTITION BY emp_id ORDER BY sales_amt DESC) AS row_num,
           RANK()       OVER (PARTITION BY emp_id ORDER BY sales_amt DESC) AS rank,
           DENSE_RANK() OVER (PARTITION BY emp_id ORDER BY sales_amt DESC) AS dense_rank
    FROM sales
    ORDER BY emp_id, month;
    
    ```

  ### 🔍 결과 예시:

  | emp_id | month | sales_amt | row_num | rank | dense_rank |
      | --- | --- | --- | --- | --- | --- |
  | 1 | 2 | 150 | 1 | 1 | 1 |
  | 1 | 3 | 150 | 2 | 1 | 1 |
  | 1 | 1 | 100 | 3 | 3 | 2 |
  | 2 | 2 | 250 | 1 | 1 | 1 |
  | 2 | 3 | 250 | 2 | 1 | 1 |
  | 2 | 1 | 200 | 3 | 3 | 2 |

  ✅ **차이 요약**

  | 함수 | 동점자 처리 | 순위 건너뜀 여부 |
      | --- | --- | --- |
  | `ROW_NUMBER` | 무조건 1, 2, 3, ... | X |
  | `RANK` | 동점자 동일 순위 | O (건너뜀) |
  | `DENSE_RANK` | 동점자 동일 순위 | X (연속적) |
    
  ---

  ## ✅ 3. `LAG()` / `LEAD()` – 전/후 행 참조

  > 전월 대비 증감 구하기
  >

    ```sql
    SELECT emp_id, emp_name, month, sales_amt,
           LAG(sales_amt)  OVER (PARTITION BY emp_id ORDER BY month) AS prev_month_amt,
           LEAD(sales_amt) OVER (PARTITION BY emp_id ORDER BY month) AS next_month_amt,
           sales_amt - LAG(sales_amt) OVER (PARTITION BY emp_id ORDER BY month) AS diff_from_prev
    FROM sales
    ORDER BY emp_id, month;
    
    ```

  ### 🔍 결과 예시:

  | emp_id | month | sales_amt | prev_month_amt | next_month_amt | diff_from_prev |
      | --- | --- | --- | --- | --- | --- |
  | 1 | 1 | 100 | null | 150 | null |
  | 1 | 2 | 150 | 100 | 150 | 50 |
  | 1 | 3 | 150 | 150 | null | 0 |
    
  ---

  ## ✅ 4. `FIRST_VALUE()` / `LAST_VALUE()`

  > 직원별 월별 매출 중, 가장 첫 월/마지막 월의 매출을 비교하고 싶을 때
  >

    ```sql
    SELECT emp_id, emp_name, month, sales_amt,
           FIRST_VALUE(sales_amt) OVER (PARTITION BY emp_id ORDER BY month) AS first_month_amt,
           LAST_VALUE(sales_amt) OVER (PARTITION BY emp_id ORDER BY month
                                        ROWS BETWEEN UNBOUNDED PRECEDING AND UNBOUNDED FOLLOWING) AS last_month_amt
    FROM sales
    ORDER BY emp_id, month;
    
    ```

  > LAST_VALUE는 반드시 ROWS BETWEEN ...을 지정해야 진짜 마지막 값이 나옴
  >
    
  ---

  ## 🔚 요약

  | 함수 | 설명 |
      | --- | --- |
  | `ROW_NUMBER()` | 순번 |
  | `RANK()` | 순위 (건너뜀 있음) |
  | `DENSE_RANK()` | 순위 (건너뜀 없음) |
  | `LAG()` / `LEAD()` | 이전/다음 값 참조 |
  | `FIRST_VALUE()` / `LAST_VALUE()` | 첫 번째 / 마지막 값 참조 |
  | `SUM() OVER` | 누적합, 이동 평균 등 |
    
  ---
