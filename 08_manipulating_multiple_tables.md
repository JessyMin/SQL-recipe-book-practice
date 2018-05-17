## 8. 여러 개의 테이블 조작하기

* 여러 개의 테이블을 조작하면 SQL이 복잡해지기 쉬움
* 간단하고 가독성 있게 작성하는 방법
* 부족한 데이터를 보완하는 방법

로그 데이터 분석
하나의 거대하 로그파일이 한 테이블에 저장되어 있어도, 여러 테이블을 다루는 것처럼 처리해야 할 경우가 있음.
  * 셀프조인 등을 통해 레코드를 비교

### 8-1. 여러 개의 테이블을 세로로 결합하기

* 두 테이블의 컬럼이 완전히 일치해야 함
* 한 쪽에만 존재하는 컬럼
  * SELECT 구문으로 제외시킴
  * 디폴트 값을 부여

```sql
SELECT
  'app1' AS app_name,
  user_id,
  name,
  email
FROM app1_mst_users

UNION ALL

SELECT
  'app2' AS app_name,
  user_id,
  name,
  '' AS email
FROM app2_mst_users;
```

<br>

### 8-2. 여러 개의 테이블을 가로로 결합하기

```sql
SELECT
  m.category_id,
  m.name,
  s.sales,
  r.product_id
FROM mst_categories m
-- 카테고리별 매출액 결합
LEFT JOIN category_sales s
  ON m.category_id = s.category_id
-- 카테고리별 최고매출 상품만 결합
LEFT JOIN product_sale_ranking r
  ON m.category_id = r.category_id
  AND r.ranks = 1;
```

* MySQL에서는 `rank`를 컬럼명으로 사용할 수 없음. Error Code 1064 발생.

```sql  

-- 상관 서브쿼리를 사용하는 방법
SELECT
  m.category_id,
  m.name,
  -- 카테고리별 매출액 추출
  (SELECT s.sales
   FROM category_sales s
   WHERE m.category_id = s.category_id
  ) AS sales,
  -- 카테고리별 최고매출 상품 1개 추출
  (SELECT r.product_id
   FROM product_sale_ranking r
   WHERE m.category_id = r.category_id
   ORDER BY sales DESC
   LIMIT 1
  ) AS top_sale_product
FROM mst_categories m;

```

[의문] 상관 서브쿼리의 개념은? 어떨 때 사용하면 유용한 건지?

<br>

### 8-4. 계산한 테이블에 이름붙여 재사용하기(CTE)

* CTE란?
  * SQL99에서 도입된 공통 테이블 식(Common Table Expression)
  * 일시적인 테이블에 이름을 붙여 재사용
  * 코드의 가독성이 높아짐

* 예시
  * 카테고리별 매출 1/2/3위 제품을 추출한 테이블 작성

```sql
WITH
product_sales_ranking AS (
  SELECT
    category_name,
    product_id,
    sales,
    ROW_NUMBER()
      OVER(PARTITION BY category_name ORDER BY sales DESC)
      AS ranks
  FROM product_sales
)
, mst_rank AS(
  SELECT DISTINCT ranks AS ranks
  FROM product_sales_ranking
  LIMIT 3
)
SELECT
  m.ranks
, b.product_id AS book
, b.sales AS book_sales
, c.product_id AS cd
, c.sales AS cd_sales
, d.product_id AS dvd
, d.sales AS dvd_sales
FROM mst_rank m
LEFT JOIN product_sales_ranking b
  ON b.ranks = m.ranks
     AND b.category_name = 'book'
LEFT JOIN product_sales_ranking c
  ON c.ranks = m.ranks
     AND c.category_name = 'cd'
LEFT JOIN product_sales_ranking d
  ON d.ranks = m.ranks
     AND d.category_name = 'dvd'
;

```
