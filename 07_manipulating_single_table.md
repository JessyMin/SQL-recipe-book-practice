## 7. 하나의 테이블에 대한 조작


### 7-1. 그룹의 특징 잡기

* Aggregation 함수 : 여러 레코드를 기반으로 하나의 값을 리턴하는 함수

#### 테이블 전체의 특징량 계산하기
```sql
SELECT
    COUNT(*) AS total_count,
    COUNT(DISTINCT(user_id)) AS user_count,
    COUNT(DISTINCT(product_id)) AS product_count,
    SUM(score) AS sum,
    AVG(score) AS average,
    MAX(score) AS max,
    MIN(score) AS min
FROM review;
```
<br>

#### 그룹핑한 데이터의 특징량 계산하기
```sql
SELECT
    user_id,
    COUNT(score) AS total_count,
    COUNT(product_id) AS product_count,
    SUM(score) AS sum,
    AVG(score) AS average,
    MAX(score) AS max,
    MIN(score) AS min
FROM review
GROUP BY user_id;
```

* `GROUP BY`에 지정한 컬럼 또는 집약함수만 `SELECT` 구문의 컬럼으로 사용 가능
* `GROUP BY` 구문에 지정한 컬럼을 유니크 키로 새로운 테이블을 생성하게 됨.    
이 과정에서 그 외의 컬럼은 사라짐. 따라서 집약 전/후의 값은 동시에 사용할 수 없는 것임.

<br>

#### 집약함수 적용한 값과 집약 전의 값을 동시에 다루기
```sql
SELECT
    user_id,
    product_id,
    score,
    AVG(score) OVER( ) AS avg_score,
    AVG(score) OVER(PARTITION BY user_id) AS user_avg_score,
    score - AVG(score) OVER(PARTITION BY user_id) AS user_avg_score_diff
FROM review;
```
<br>

* MySQL 윈도 함수 : https://dev.mysql.com/doc/refman/8.0/en/window-functions-usage.html

<br>
<br>

### 7-2. 그룹 내부의 순서

#### ORDER BY 구문으로 순서 정의하기
윈도우 함수의 OVER 구문 내부에 ORDER BY 구문을 사용한다.
```sql
SELECT
    product_id,
    score,
    ROW_NUMBER() OVER(ORDER BY score DESC) AS row_number,
    RANK() OVER(ORDER BY score DESC) AS rank,
    DENSE_RANK() OVER(ORDER BY score DESC) AS dense_rank,
    LAG(product_id) OVER(ORDER BY score DESC) AS lag1,
    LAG(product_id, 2) OVER(ORDER BY score DESC) AS lag2,
    LEAD(product_id) OVER(ORDER BY score DESC) AS lead1,
    LEAD(product_id, 2) OVER(ORDER BY score DESC) AS lead2
FROM popular_products
ORDER BY row;
```

그러나 MySQL에는 윈도우 함수가 없다.
사용자 정의 변수나 다른 방법을 이용해서 구현할 수 있다.

```sql
-- MySQL 8.0
SELECT
  product_id,
  score,
  -- ROW_NUMBER()
  @number := @number + 1 AS row_num,
  -- RANK()
  (SELECT COUNT(score) + 1
   FROM popular_products p1
   WHERE p1.score > p2.score
  ) AS ranking,
  -- DENSE_RANK()
  (SELECT COUNT(DISTINCT score) + 1
   FROM popular_products p1
   WHERE p1.score > p2.score
  ) AS ranking_dense
FROM popular_products p2, (SELECT @number := 0) AS n
ORDER BY score DESC;

-- Alias로 rank, row_number 등은 사용 불가
```
참고 : http://nodapiseverywhere.blogspot.kr/2016/11/mysql-ranking-query.html  

<br>

#### ORDER BY 구문과 집약함수 조합하기

__윈도 프레임 지정 구문__
현재 레코드 위치를 기반으로 상대적인 윈도를 정의함. `ROWS BETWEEN start AND end`  
- 현재 행 : `CURRENT ROW`  
- 현재 행 + 이전 행 전부 : `UNBOUNDED PRECEDING`
- 현재 행 + 이후 행 전부 : `UNBOUNDED FOLLOWING`  
- n행 앞 : `n PRECEDING`
- n행 뒤 : `n FOLLOWING`
<br>

__1) 순위 상위로부터 누계 점수 계산하기__
```sql
SELECT
  product_id,
  score,
  SUM(score)
    OVER(ORDER BY score DESC
      ROWS BETWEEN UNBOUNDED PRECEDING AND CURRENT ROW)
  AS cum_score
FROM popular_products
ORDER BY score;
```


```sql  
-- MySQL 8.0
SELECT
  product_id,
  score,
  COALESCE(
    (SELECT SUM(score)
     FROM popular_products p1
     WHERE p1.score > p2.score
    ), 0
  ) + p2.score
  AS cum_score
FROM popular_products p2
ORDER BY score DESC;
```

<br>

__2) 현재 행/앞뒤 행이 가진 값으로 로컬 평균 계산하기__
```sql  
SELECT
  product_id,
  score,
  AVG(score)
    OVER(ORDER BY score DESC
      ROWS BETWEEN 1 PRECEDING AND 1 FOLLOWING)
  AS local_avg
FROM popular_products
ORDER BY score;
```

```sql
-- MySQL 8.0
SELECT
  p1.product_id,
  p1.score,
  AVG(p2.score) AS local_avg
FROM (
      SELECT
        product_id,
        score,
        @rownum1 := @rownum1 + 1 AS row_num1
      FROM popular_products, (SELECT @rownum1 := 0) AS n
      ORDER BY score DESC
     ) p1
LEFT JOIN (
      SELECT
        score,
        @rownum2 := @rownum2 + 1 AS row_num2
      FROM popular_products, (SELECT @rownum2 := 0) AS n
      ORDER BY score DESC
      ) p2
  ON p1.row_num1 =p2.row_num2
    OR p1.row_num1 = p2.row_num2 + 1
    OR p1.row_num1 = p2.row_num2 - 1
GROUP BY p1.product_id, p1.score
ORDER BY p1.score DESC;
```
<br>

__3) 윈도 내부의 가장 첫 번째/마지막 레코드 추출__
```sql
SELECT
  product_id, score,
  FIRST_VALUE(product_id)
    OVER(ORDER BY score DESC
      ROWS BETWEEN UNBOUNDED PRECEDING AND UNBOUNDED FOLLOWING)
  AS first_value
FROM popular_products
ORDER BY score DESC;    
```
```sql
--Mysql 8.0

```
<br>

#### PARTITION BY와 ORDER BY 조합하기

__1) 각 카테고리별로 순위 계산__
```sql
SELECT
  category,
  product_id,
  score,
  ROW_NUMBER()
    OVER(PARTITION BY category ORDER BY score DESC)
  AS row_num
FROM popluar_products
ORDER BY category, row_num;
```
```sql
--Mysql 8.0
SELECT
  category,
  product_id,
  score,
  CASE
    WHEN category = 'action' THEN @r1 := @r1 + 1
    WHEN category = 'drama' THEN @r2 := @r2 + 1
    ELSE ''
   END AS row_num
FROM popular_products, (SELECT @r1 := 0, @r2 := 0) AS n
ORDER BY category, score DESC;
```
<br>

__2) 각 카테고리별 상위 n개 상품 추출__
```sql
SELECT
  category,
  product_id,
  score,
  ranks
FROM (
      SELECT
        category,
        product_id,
        score,
        ROW_NUMBER()
          OVER(PARTITION BY category ORDER BY score DESC)
        AS ranks
      FROM popluar_products    
     ) tmp
WHERE ranks <= 2
ORDER BY category, rank;
```
```sql

```
__3) 각 카테고리별 최상위 순위 상품 추출__
```sql

```
```sql
```

### 7-3. Pivot(1) : 세로 기반 데이터를 가로 기반으로 변환하기

#### 행을 열로 변환하기

목표하는 피봇 테이블의 열의 종류와 수를 알고 있는 경우

```sql  
SELECT
  dt
  , SUM(CASE WHEN indicator = 'impressions' THEN val END) AS impressions
  , SUM(CASE WHEN indicator = 'sessions' THEN val END) AS sessions
  , SUM(CASE WHEN indicator = 'users' THEN val END) AS users
FROM daily_kpi
GROUP BY dt
;
```
* CASE 식의 결과는 리스트이다. 여기서 값 하나를 추출하기 위해 일반적으로 MAX/MIN을 사용한다고 한다.

아래는 나의 시행착오 쿼리다. CASE문 적용 후 다시 GROUP BY로 집계해, 불필요한 단계를 거쳤다.
```sql
WITH
tmp AS (
SELECT
  dt
  , CASE WHEN indicator = 'impressions' THEN val END AS impressions
  , CASE WHEN indicator = 'sessions' THEN val END AS sessions
  , CASE WHEN indicator = 'users' THEN val END AS users
FROM
  daily_kpi
)
SELECT
  dt
  , SUM(impressions) AS impressions
  , SUM(sessions) AS sessions
  , SUM(users) AS users
FROM tmp
GROUP BY dt
;
```

#### 행을 쉼표로 구분한 문자열로 집약하기

목표하는 피봇 테이블의 열의 종류와 수를 모르는 경우

예) 상품 구매로그를 구매 ID 기반으로 집계할 때, 한 주문에 몇 개의 상품을 주문했는지 알 수 없는 경우

```sql
-- MySQL
SELECT
  purchase_id
  , GROUP_CONCAT(product_id) AS product_ids
  , SUM(price) AS amount
FROM
  purchase_detail_log
GROUP BY
  purchase_id
;
```

* MySQL의 `GROUP_CONCAT()` 함수는 구분자의 디폴트 값이 쉼표로 되어 있다.
* 변경하고 싶은 경우 `GROUP_CONCAT(column separator '|')` 이렇게 해준다. (<a href="http://fruitdev.tistory.com/16">참고</a>)


* 레코드가 세로로 쌓여 있는 세로 기반 테이블은 시스템이 다루기 쉬운 반면, 가로 기반 테이블은 사람이 직감적으로 이해하기 쉽다.

<br>

### 7-4. 가로 기반 데이터를 세로 기반으로 변환하기

#### 열로 표현된 값을 행으로 변환하기

```sql
SELECT
    years
  , CASE
      WHEN idx = 1 THEN 'q1'
      WHEN idx = 2 THEN 'q2'
      WHEN idx = 3 THEN 'q3'
      WHEN idx = 4 THEN 'q4'
    END AS quarter
  , CASE
      WHEN idx = 1 THEN q1
      WHEN idx = 2 THEN q2
      WHEN idx = 3 THEN q3
      WHEN idx = 4 THEN q4
    END AS sales    
FROM quarterly_sales q
CROSS JOIN
  (
             SELECT 1 AS idx
   UNION ALL SELECT 2 AS idx
   UNION ALL SELECT 3 AS idx
   UNION ALL SELECT 4 AS idx
  ) i
ORDER BY years, quarter
;
```


#### 임의의 길이를 가진 배열을 행으로 전개하기
