## 9. 시계열 기반으로 데이터 집계하기
<br>

### 9-1. 날짜별 매출 집계하기
```sql
SELECT
  dt,
  COUNT(order_id) AS purchase_count,
  SUM(purchase_amount) AS total_amount,
  AVG(purchase_amount) AS avg_amount
FROM purchase_log
GROUP BY dt
ORDER BY dt;
```

* `COUNT(*)`와 `COUNT(order_id)`의 차이는?
<br>

### 9-2. 이동평균을 사용한 날짜별 추이 보기
```sql
SELECT
  dt,
  SUM(purchase_amount) AS total_amount,
  -- 최근 최대 7일 동안의 평균 계산
  AVG(SUM(purchase_amount))
    OVER(ORDER BY dt ROWS BETWEEN 6 PRECEDING AND CURRENT ROW)
  AS seven_day_avg_amount,
  -- 최근 최대 7일 동안의 평균, 7일이 모두 존재하는 경우만 계산
  CASE
    WHEN 7 = COUNT(*)
      OVER(ORDER BY dt ROWS BETWEEN 6 PRECEDING AND CURRENT ROW)
    THEN AVG(SUM(purchase_amount))
      OVER(ORDER BY dt ROWS BETWEEN 6 PRECEDING AND CURRENT ROW)
    ELSE ''
  END AS seven_day_avg_amount_strict,
FROM purchase_log
GROUP BY dt
ORDER BY dt;
```

```sql
--MySQL 8.0
SELECT
  p1.dt,
  p1.daily_amount AS total_amount,
  SUM(p2.daily_amount)/COUNT(DISTINCT p2.dt) AS seven_day_avg_amount,
  CASE
    WHEN COUNT(p2.dt) = 7
      THEN SUM(p2.daily_amount)/COUNT(DISTINCT p2.dt)
	  ELSE ''
  END AS seven_day_avg_amount_strict   
FROM (
      SELECT
        dt,
        SUM(purchase_amount) AS daily_amount
      FROM purchase_log
      GROUP BY dt
     ) p1
JOIN (
      SELECT
        dt,
        SUM(purchase_amount) AS daily_amount
      FROM purchase_log
      GROUP BY dt
     ) p2
  ON DATEDIFF(p1.dt, p2.dt) BETWEEN 0 AND 6
GROUP BY p1.dt
ORDER BY p1.dt;
```
<br>

### 9-3. 당월 매출 누계 구하기
날짜별 매출 및 해당 월에 누적된 매출을 동시에 확인할 수 있는 리포트
```sql
WITH
daily_purchase AS (
  SELECT
    dt,
    SUBSTR(dt,  1, 4) AS years,
    SUBSTR(dt, 6, 2) AS months,
    SUBSTR(dt, 9, 2) AS dates,
    SUM(purchase_amount) AS purchase_amount
  FROM purchase_log
  GROUP BY dt
  ORDER BY dt
)  
SELECT
  dt,
  CONCAT(years, '-', months) AS year_month,
  purchase_amount,
  SUM(purchase_amount)
    OVER (PARTITION BY years, months ORDER BY dt ROWS UNBOUNDED PRECEDING)
  AS agg_amount
FROM daily_purchase
ORDER BY dt;
```
* `date` 자료형에도 `SUBSTRING`이 가능하다.
* 책에서는 필요할 때마다 `SUBSTR(dt, 1, 7)` 또는 `DATE_FORMAT(dt, '%Y-%m')`로 `year_month`를 추출하는 대신, 가독성을 위해 임시 테이블을 만들면서 날짜를 `year`/`month`/`date`로 분할해 놓고, 다시 `CONCAT`로 결합해서 사용하고 있다.

"서비스를 운용, 개발하기 위해 사용하는 SQL과 비교했을 때, 빅데이터 분석 SQL은 성능이 조금 떨어지더라도 가독성과 재사용성을 중시해서 작성하는 경우가 많습니다."   - 147p.

<br>

### 9-4. 월별 매출의 전년 동월 대비 구하기
