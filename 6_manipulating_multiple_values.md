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
