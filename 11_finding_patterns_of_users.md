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

#### 액션과 관련된 지표 집계하기

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
