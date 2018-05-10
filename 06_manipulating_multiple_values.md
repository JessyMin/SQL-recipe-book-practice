### 6. 여러 개의 값에 대한 조작

### 6-1. 문자열 연결하기
* CONCAT() 함수 사용
* PostgreSQL, Redshift는 || 연산자 사용 가능

```sql  
SELECT
  user_id
  , CONCAT(pref_name, ' ', city_name)
    AS pref_city
  -- PostgreSQL, Redshift
  , pref_name || city_name
    AS pref_city2
FROM
  mst_user_location
;
```

<br>

### 6-2. 여러 개의 값 비교하기

#### 2개 컬럼 비교
__: 분기별 매출 증감 판정하기__
```sql
SELECT
  years
  , q1
  , q2
  , CASE
      WHEN q2-q1 > 0 THEN '+'
      WHEN q2 - q1 < 0 THEN '-'
      ELSE ''
    END AS judge_q1_q2
  , q2 - q1 AS diff_q2_q1
  , SIGN(q2 - q1) AS sign_q2_q1
FROM
  quarterly_sales
ORDER BY
  years
;
```
__[의문]__ 원본 테이블이 연도별로 정렬되어 있는데, 왜 `ORDER BY`를 다시 해주는 걸까?

<br>

#### 다수 컬럼 비교
__: 매출 최대/최소인 분기 찾기__


* 3개 이상의 컬럼을 비교해 최대값/최대값을 찾을 때는 `GREATEST()`, `LEAST()` 함수를 사용한다.
  * SQL 표준은 아니지만 대부분의 쿼리 엔진에 구현됨
  * 책의 쿼리는 아래와 같이 깔끔

```sql
SELECT
    years
  , GREATEST(q1, q2, q3, q4) AS greatest_sales
  , LEAST(q1, q2, q3, q4) AS least_sales
FROM
  quarterly_sales
;
```

* 그런데 책의 예제 데이터는 3/4분기 매출이 확정되지 않아 `NULL`인 상태다.

* MySQL에서는 `NULL`이 포함된 연산에서 `NULL`이 반환되고, 공백으로 치환하면 `BLOB`이 반환된다.
* 따라서 아래와 같은 꼼수로...해결해보았다.

```sql
SELECT
    years
  , GREATEST(
      q1  
    , q2
    , COALESCE(q3, 0)
    , COALESCE(q4, 0))
    AS greatest_sales
  , LEAST(
      q1    
    , q2
    , COALESCE(q3, 10000000)
    , COALESCE(q4, 10000000))
    AS least_sales
FROM quarterly_sales
;
```

<br>
#### 다수 컬럼 연산
__: 4분기 매출 평균 구하기__

```sql
WITH
remove_nulls AS (
  SELECT
      years
    , q1
    , q2
    , COALESCE(q3, 0) AS q3
    , COALESCE(q4, 0) AS q4
  FROM
    quarterly_sales
)
SELECT
    year
  , (q1 + q2 + q3 + q4)
    / (SIGN(q1) + SIGN(q2) + SIGN(q3) + SIGN(q4))
    AS avg_sales
FROM
  remove_nulls
;
```
* 책에서는 `CTE`를 사용하지 않았으나, 재사용할 일이 많아서 적용해보았다.

<br>

### 6-3. 2개의 값 비율 계산하기

* 책에는 값을 0으로 나누면 오류가 발생한다고 되어 있으나, MySQL은 그냥 `NULL`을 반환한다.

```sql
SELECT
    dt
  , ad_id
  , 100 * clicks / impressions AS ctr_as_percent
FROM
  advertising_stats
;
```

<br>

* 0으로 나누면 에러가 발생하는 엔진일 경우, `NULLIF`를 이용해 0을 null로 치환해준다.

* `NULLIF(expr1, expr2)`
  * 두 값이 같으면 null을 리턴, 다르면 expr1을 리턴함
  * 0을 null로 치환할 때 사용

```sql
SELECT NULLIF(0, 0);  
-- NULL을 리턴
SELECT NULLIF(5, 0);
-- 5를 리턴
```

<br>

* 비슷한 함수도 정리해본다.
* `IFNULL(expr1, expr2)`
  * expr1이 null이면 expr2를 리턴, 아니면 원래값인 expr1을 리턴한다.
  * null을 0으로 치환할 때 사용

```sql
SELECT IFNULL(NULL, 0);
-- 0을 리턴
SELECT IFNULL(3, 0);
-- 원래값인 3을 리턴
```

<br>

### 6-4. 두 값의 거리 계산하기

#### 절대값, 제곱평균 제곱근 계산하기

* 제곱평균 제곱근(RMS: Root Mean Square) : 두 값의 차이를 제곱한 뒤 제곱근을 적용한 값
```sql  
SELECT
    x1
  , x2
  , ABS(x1 - x2) AS abs
  , SQRT(POWER((x1 - x2), 2)) AS rms
FROM
  location_1d
;
```

#### 유클리드 거리 계산

* 책에서는 'xy평면 위에 있는 두 점의 유클리드 거리 계산'으로 소개하고 있지만, 다차원 공간에 존재하는 두 점에도 적용 가능함.

```sql
SELECT
    x1, y1
  , x2, y2
  , SQRT(
      POWER((x1 - x2), 2) + POWER((y1 - y2), 2)
    ) AS dist
FROM
  location_2d
;
```
* PostgreSQL은 point 자료형과 거리 연산자 <-> 사용 가능

<br>

### 6-5. 날짜/시간 계산하기

* 기본적인 날짜/시간 데이터 계산 방법

```sql  
-- MySQL 8.0

SELECT
  user_id,
  -- 회원가입 날짜/시간
  register_stamp,
  -- 가입 1시간 뒤
  register_stamp + interval 1 hour AS after_1_hour,
  -- 가입 30분 전
  register_stamp - interval 30 minute AS before_30_minutes,
  -- 회원가입 날짜
  DATE_FORMAT(register_stamp, '%Y-%m-%d') AS register_date,
  -- 가입 1일 후
  DATE_FORMAT(register_stamp, '%Y-%m-%d') + interval 1 day AS after_1_day,
  -- 가입 1달 전
  DATE_FORMAT(register_stamp, '%Y-%m-%d') - interval 1 month AS before_1_month
FROM mst_users_with_birthday;

```

<br>

### 날짜 데이터들의 차이 계산하기

* 날짜 차이 계산
DATEDIFF(day1, day2) : day1에서 day2를 뺀 값 반환
* ex) SELECT DATEDIFF('2018-05-03', '2018-05-01') : 2

```sql
SELECT
  user_id,
  CURDATE() AS today,
  DATE_FORMAT(register_stamp, '%Y-%m-%d') AS register_date,
  DATEDIFF(
      CURDATE(), DATE_FORMAT(register_stamp, '%Y-%m-%d'))
    AS diff_days
FROM mst_users_with_birthday;
```

<br>

### 사용자의 생년월일로 나이 계산하기

* 한국식 나이는 간단하게 계산 가능

```sql
-- MySQL 8.0
-- 한국식 나이 계산
SELECT
  user_id,
  CURDATE() AS today,
  DATE_FORMAT(register_stamp, '%Y-%m-%d') AS register_date,
  birth_date,
  -- 현재 나이
  DATE_FORMAT(CURDATE(), '%Y')
    - DATE_FORMAT(birth_date, '%Y') + 1
    AS current_age
  -- 가입 당시 나이
  DATE_FORMAT(register_stamp, '%Y')
    - DATE_FORMAT(birth_date, '%Y') + 1
    AS register_age
FROM mst_users_with_birthday;

```

* 외국식의 만 나이를 계산하는 방법은 복잡함
  * 해당 년의 생일이 지났는지, 윤년 등을 고려해야 함
* 전용함수가 없다면, 날짜를 정수로 표현해서 계산하는 방법이 있다.
  * 타임스탬프에서 날짜만 추출 -> 문자열에서 하이픈 제거 -> 정수로 캐스트

```sql  
-- MySQL 8.0

SELECT
  user_id,
  DATE_FORMAT(register_stamp, '%Y-%m-%d') AS register_date,
  birth_date,
  -- 가입 당시 나이
  FLOOR(
    (CAST(REPLACE(SUBSTR(register_stamp, 1, 10), '-', '') AS unsigned)
     - CAST(REPLACE(birth_date, '-', '') AS unsigned)
    ) / 10000
  ) AS register_age,
  -- 현재 나이
  FLOOR(
    (CAST(REPLACE(CURDATE(), '-', '') AS unsigned)
     - CAST(REPLACE(birth_date, '-', '') AS unsigned)
    ) / 10000
  ) AS current_age
FROM mst_users_with_birthday;

```
