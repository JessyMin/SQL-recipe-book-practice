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

#### 로그인 사용자/비로그인 사용자 구분해서 집계하기

- 서비스 충성도가 높은 사용자와 낮은 사용자의 행동 패턴을 비교할 수 있음
- user_id 값이 비어 있으면 비로그인 사용자

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

위에서 산출한 login_status를 기반으로 액션 수와 unique users를 집계
```sql  

```
