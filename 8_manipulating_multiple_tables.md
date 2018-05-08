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
