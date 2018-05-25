## 12. 시계열에 따른 사용자 전체의 상태 변화 찾기

* 사용자의 서비스 사용을 시계열로 수치화, 변화를 시각화하는 방법을 살펴봄

### 12-1. 등록 수의 추이와 경향

* 날짜별 등록 수의 추이

```sql
SELECT
  register_date
  , COUNT(DISTINCT user_id) AS register_count
FROM
  mst_users
GROUP BY
  register_date
ORDER BY
  register_date
;
```

* 회원 테이블이므로 user_id의 중복은 없을 것이지만, 책에서는 아래와 같이 언급하며 `DISTINCT`를 사용하고 있다.
  * '이 책에서는 등록 수도 등록한 사용자 수로 다룬다고 정의하고, 정의에 따라 중복을 생략하고 집계하겠습니다'
  * 무슨 뜻인지 지금은 이해가 되지 않는다. 나중을 위해 기록해둔다. 실무를 해봐야 알 수 있을 듯해서다.

#### 월별 등록 수 추이

* 매달 등록 수와 전월비
```sql
SELECT
  SUBSTR(register_date, 1, 7) AS register_month
  , COUNT(DISTINCT user_id) AS register_count
  , COUNT(DISTINCT user_id)
    / LAG(COUNT(DISTINCT user_id))
      OVER(ORDER BY SUBSTR(register_date, 1, 7))
    * 100
    AS month_over_month_ratio
FROM
  mst_users
GROUP BY
  register_month
ORDER BY
  register_month
;
```

#### 등록 디바이스별 등록 수 추이
```sql
SELECT
  SUBSTR(register_date, 1, 7) AS register_month
  , COUNT(DISTINCT user_id) AS register_count
  , COUNT(DISTINCT CASE
      WHEN register_device = 'pc' THEN user_id END)
    AS register_pc
  , COUNT(DISTINCT CASE
      WHEN register_device = 'mobile' THEN user_id END)
    AS register_mobile
  , COUNT(DISTINCT CASE
      WHEN register_device = 'tablet' THEN user_id END)
    AS register_tablet
FROM
  mst_users
GROUP BY
  register_month
ORDER BY
  register_month
;  
```

* 책의 쿼리를 통해 `CASE ~ WHEN` 앞에 `DISTINCT`를 바로 붙여서 `COUNT`하는 문법을 처음 보았다.
* 아래는 나의 시행착오 쿼리다. 물론 회원 테이블에서 user_id가 unique하게 정의되어 있다면 아래 쿼리도 이론상 문제없다.

```sql
SELECT
  SUBSTR(register_date, 1, 7) AS register_month
  , COUNT(DISTINCT user_id) AS register_count
  , SUM(CASE
      WHEN register_device = 'pc' THEN 1 END)
    AS register_pc
  , SUM(CASE
      WHEN register_device = 'mobile' THEN 1 END)
    AS register_mobile
  , SUM(CASE
      WHEN register_device = 'tablet' THEN 1 END)
    AS register_tablet
FROM
  mst_users
GROUP BY
  register_month
ORDER BY
  register_month
;  
```

<br>

### 12-2. 지속률과 정착률 산출하기

* 등록 시점을 기준으로 일정 기간동안 사용자가 지속해서 사용하고 있는지를 파악
  * 지속률 :
  * 정착률 :

* 서비스의 목적과 용도 등 특성을 고려해 적절한 지표를 선택한다. 
