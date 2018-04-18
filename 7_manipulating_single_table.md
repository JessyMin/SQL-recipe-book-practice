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

* GROUP BY에 지정한 컬럼 또는 집약함수만 SELECT 구문의 컬럼으로 사용 가능
* GROUP BY 구문에 지정한 컬럼을 유니크 키로 새로운 테이블을 생성하게 됨.    
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
