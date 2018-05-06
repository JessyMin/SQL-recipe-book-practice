## 11. 사용자 전체의 특징과 경향 찾기

액션 로그(action_log) 테이블
- view : 페이지 열람
- favorite : 관심상품 등록
- add_cart : 장바구니 추가
- purchase : 구매
- review : 상품 리뷰
- user_id : 로그인한 사용자, 비로그인 사용 시 해당 레코드 비어 있음

<p>

### 11-1. 사용자의 액션 수 집계하기

### 액션과 관련된 지표 집계하기

사용률 = 특정 액션 unique users / 전체 액션 unique users  

```sql  
WITH
  stats AS (
  SELECT COUNT(DISTINCT session) AS total_unique_users
  FROM action_log
  )
SELECT
  l.action,
  -- 액션 유니크 유저
  COUNT(DISTINCT l.session) AS action_unique_users,
  -- 액션 수
  COUNT(l.action) AS action_count,
  -- 전체 유니크 유저
  s.total_unique_users,
  -- 사용률 = 100 * (액션별 유니크 유저) / (전체 유니크 유저)
  100 * COUNT(DISTINCT l.session) / s.total_unique_users AS usage_rate,
  -- 1인당 액션 수 = (액선 수) / (액션별 유니크 유저)
  COUNT(l.session) / COUNT(DISTINCT l.session) AS count_per_user  
FROM action_log l
CROSS JOIN stats s
GROUP BY l.action, s.total_unique_users;
```

<p>

### 로그인 사용자/비로그인 사용자 구분해서 집계하기

- 서비스 충성도가 높은 사용자와 낮은 사용자의 행동 패턴을 비교할 수 있음
- user_id 값이 비어 있으면 비로그인 사용자

아래는 먼저 로그인/비로그인 상태를 판별하는 쿼리이다.

```sql  
-- 로그인 상태를 판별하는 쿼리  
SELECT
  session,
  user_id,
  action,
  -- user_id가 NULL 또는 빈 문자가 아닌 경우 login으로 판정
  CASE
    WHEN COALESCE(user_id, ' ') <> ' ' THEN 'login'
    ELSE 'guest'
  END AS login_status
FROM action_log;

--시행착오 쿼리 : NULL만 고려하고 빈 문자를 고려하지 않음
SELECT
  session,
  user_id,
  action,
  CASE
    WHEN user_id IS NOT NULL THEN 'login'
    ELSE 'guest'
  END AS login_status
FROM action_log;
```
<p>

위에서 산출한 login_status를 기반으로 액션 수와 unique users를 집계한다.

__1) ROLLUP 사용__
```sql
WITH
login_status AS (    
  SELECT
    session,
    user_id,
    action,
    CASE
      WHEN user_id IS NOT NULL THEN 'login'
      ELSE 'guest'
    END AS login_status
  FROM action_log
)
SELECT
  COALESCE(action, 'all') AS action,
  COALESCE(login_status, 'all') AS login_status,
  COUNT(DISTINCT session) AS action_unique_users,
  COUNT(action) AS action_count
FROM login_status
GROUP BY action, login_status WITH ROLLUP;
```

<br>

__2. UNION 사용(한 삽질)__
...MySQL에 `ROLLUP`이 있는 걸 뒤늦게 알았다.

```sql  
WITH
login_status AS (    
  SELECT
    session,
    user_id,
    action,
    CASE
      WHEN COALESCE(user_id, ' ') <> ' ' THEN 'login'
      ELSE 'guest'
    END AS login_status
  FROM action_log
)
SELECT *
FROM
(
  -- (1)액션별 로그인/비로그인 상태에 따른 집계
  (
  SELECT
    action,
    login_status,
    COUNT(DISTINCT session) AS action_unique_users,
    COUNT(action) AS action_count
  FROM login_status
  GROUP BY action, login_status
  )
  UNION
  -- (2)액션별 전체 집계
  (
  SELECT
    action,
    'all' AS login_status,
    COUNT(DISTINCT session) AS action_unique_users,
    COUNT(action) AS action_count
  FROM login_status
  GROUP BY action
  )
ORDER BY action
) tmp

UNION
-- (3)전체 집계
(
SELECT
  'all' AS action,
  'all' AS login_status,
  COUNT(DISTINCT session) AS action_unique_users,
  COUNT(action) AS action_count
FROM action_log
);

```

* `UNION`한 결과 전체에 `ORDER BY`를 적용할 때는, 각각의 `SELECT`문을 괄호로 감싼 뒤 마지막 구문 뒤에 `ORDER BY`를 붙이면 된다.
(참고
https://stackoverflow.com/questions/13261113/mysql-order-by-and-union-and-parentheses)

* 그런데 `SELECT`문 각각에 서로 다른 정렬 기준을 적용해야 하는 경우도 있다. 위의 쿼리는 (1)+(2)를 action으로 ORDER BY하고, 가장 아래 row에 전체 집계값인 (3)을 붙이고 싶은 경우다. 이 때는 위와 같이 **서브쿼리** 를 사용해 준다.
(참고 : https://dev.mysql.com/doc/refman/8.0/en/union.html
  3번째 코멘트, Posted by Phil McCarley on February 28, 2006)


```sql  
--시행착오 쿼리 : 서브쿼리가 아닌 괄호만을 사용

WITH
login_status AS (    
  SELECT
    session,
    user_id,
    action,
    CASE
      WHEN COALESCE(user_id, ' ') <> ' ' THEN 'login'
      ELSE 'guest'
    END AS login_status
  FROM action_log
)

(
  -- (1)액션별 로그인/비로그인 상태에 따른 집계
  (
  SELECT
    action,
    login_status,
    COUNT(DISTINCT session) AS action_unique_users,
    COUNT(action) AS action_count
  FROM login_status
  GROUP BY action, login_status
  )
  UNION
  -- (2)액션별 전체 집계
  (
  SELECT
    action,
    'all' AS login_status,
    COUNT(DISTINCT session) AS action_unique_users,
    COUNT(action) AS action_count
  FROM login_status
  GROUP BY action
  )
ORDER BY action
)
UNION
-- (3)전체 집계
(
SELECT
  'all' AS action,
  'all' AS login_status,
  COUNT(DISTINCT session) AS action_unique_users,
  COUNT(action) AS action_count
FROM action_log
);

```

* 위는 시행착오 쿼리다. 1차적으로 (1)+(2)를 `ORDER BY`한 결과에 괄호를 사용한 뒤 (3) `UNION`했다. 하지만 `ORDER BY`의 위치나 괄호 사용여부와 상관없이 (1)+(2)+(3)을 `UNION`한 결과 전체에 `ORDER BY`가 적용된다.

<br>

### 회원과 비회원을 구분해서 집계하기

* 이전에 한번이라도 로그인했다면 현재 로그인하지 않은 상태여도 회원으로 계산하기

```sql

SELECT
  session,
  user_id,
  action,
  CASE
    WHEN COALESCE(MAX(user_id)
      OVER(PARTITION BY session ORDER BY stamp
           ROWS BETWEEN UNBOUNDED PRECEDING AND CURRENT ROW)
      , '') <> ''
      THEN 'member'
    ELSE 'none'
  AS member_status,
  stamp
FROM action_log;

```

<br>

### 11-2. 연령별 구분 집계하기
* 나이는 DB에 저장된 생일을 기반으로 리포팅 시점에 집계함

__성별과 연령대로 사용자 카테고리 구분하기__
```sql  
-- 사용자 생일/현재 날짜를 정수 표현으로 변환
WITH
  mst_users_with_int_birth_date AS (
  SELECT
    *,
    REPLACE(CURDATE(), '-', '') AS int_cur_date,
    REPLACE(birth_date, '-', '') AS int_birth_date
FROM mst_users
),
-- 현재 시점의 나이 산출
mst_users_with_age AS (
  SELECT
    *,
    FLOOR((int_cur_date - int_birth_date) / 10000) AS age
  FROM mst_users_with_int_birth_date
),
-- 성별/나이를 기준으로 카테고리 구분
mst_users_with_category AS (
  SELECT
    user_id,
    sex,
    age,
    CONCAT(
      CASE
        WHEN age >= 20 THEN CONCAT(sex, ' ')
        ELSE ''
      END,
      CASE
        WHEN age BETWEEN 4 AND 12 THEN 'Child'
        WHEN age BETWEEN 13 AND 19 THEN 'Teenager'
        WHEN age BETWEEN 20 AND 34 THEN '2034'
        WHEN age BETWEEN 35 AND 49 THEN '3549'
        WHEN age >= 50 THEN '5060'
      END
    ) AS category
  FROM mst_users_with_age
)

SELECT *
FROM mst_users_with_category;

```

<br>

__연령별 사람 수 계산하기__

```sql
-- 괄호 안은 위의 코드 재사용
WITH mst_users_with_int_birth_date(
  ...  
),
WITH mst_users_with_age(
  ...
),
WITH mst_users_with_category(
  ...
),

SELECT
  category,
  COUNT(user_id) AS user_count
FROM mst_users_with_category
GROUP BY category
ORDER BY category;
```

<br>

### 11-3. 연령별 구분의 특징 추출하기

__연령대별로 구매 상품 카테고리 집계하기__
```sql


SELECT
  u.category AS user_category,
  p.category AS product_category,
  COUNT(p.user_id) AS purchase_count
FROM action_log AS p
JOIN mst_users_with_category AS u
  ON p.user_id = u.user_id
		-- 구매 로그만 선택
        AND p.action = 'purchase'
GROUP BY u.category, p.category
ORDER BY u.category, p.category;

```
__[질문]__ 왜 JOIN ON이 아닌 WHERE로 했을까?
* JOIN과 WHERE을 같이 쓰면 먼저 JOIN 후 WHERE 적용된다고 한다.
구매로그만 먼저 필터링한 뒤 JOIN하는 게 효율적이지 않은가?
* 아니면 저자들이 쓰는 미들웨어는 알아서 WHERE 먼저 적용해줘서 그런가?

### 11-4. 사용자의 방문 빈도 집계하기

* '서비스를 한 주 동안 며칠 사용하는 사용자가 몇 명인지' 집계
```sql
WITH
action_log_with_dt AS (
  SELECT
    *,
    STR_TO_DATE(stamp, '%Y-%m-%d') AS dt
  FROM action_log
),

action_day_count_per_user AS (
SELECT
  user_id,
  COUNT(DISTINCT dt) AS action_day_count
FROM action_log_with_dt  
-- 한 주 동안의 데이터만 필터링
WHERE dt BETWEEN '2016-11-01' AND '2016-11-07'
GROUP BY user_id
)

SELECT
  action_day_count,
  -- 카운트
  COUNT(user_id) AS user_count
  -- 구성비
  100
    * COUNT(user_id)
    / SUM(COUNT(user_id)) OVER()
    AS composition_ratio,
  -- 구성비 누계
  100
    * SUM(COUNT(user_id))
      OVER(ORDER BY action_day_count
        ROWS BETWEEN UNBOUNDED PRECEDING AND CURRENT ROW)
    / SUM(COUNT(user_id)) OVER()
    AS cumulative_ratio
FROM action_day_count_per_user
GROUP BY action_day_count
ORDER BY action_day_count;

```

__[질문]__ 책에서는 COUNT(DISTINCT user_id)했는데 그럴 필요가 있을까? 바로 전의 WITH문에서 이미 한 번  GROUP BY 했는데..왜 DISTINCT 했을까?
