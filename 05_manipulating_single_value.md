### 5-2. URL에서 요소 추출하기


### 1) 레퍼러에서 어떤 웹페이지를 거쳐 넘어왔는지 판별하기

```sql
SELECT
    stamp,
    SUBSTRING_INDEX(SUBSTRING_INDEX(referrer, '//', -1), '/', 1) AS referrer_host
FROM access_log;
```
<br>


### 2) URL에서 경로와 요청 매개변수 값 추출하기

이런 경우 접근방법은 두 가지다.
1) 구분자로 잘라서 해당되는 내용만 가져온다. (mysql : REGEXP_SUBSTR)
2) 불필요한 부분을 ''로 replace한다. (mysql : REGEXP_REPLACE)
3) 필요한 부분을 extract한다.

```sql
SELECT
    stamp,
    url,
    SUBSTR(SUBSTRING_INDEX(SUBSTRING_INDEX(url, '?id=', 1), '#', 1), 23) AS path,
    CASE
        WHEN LENGTH(SUBSTRING_INDEX(url, '?id=', -1))=3
            THEN SUBSTRING_INDEX(url, '?id=', -1)
        ELSE ''
    END AS id
FROM access_log;
```

* MySQL에는 REGEXP_SUBSTR 함수가 없다.  
 The MySQL REGEXP can be used for matching strings, but not for transforming them.  
Unfortunately, MySQL's regular expression function return true, false or null depending if the expression exists or not.  
(참고 : https://dba.stackexchange.com/questions/34724/how-to-use-substring-using-regexp-in-mysql)

* substring_index 함수  
: SELECT SUBSTRING_INDEX(str, delim, count)  
: str 문자열의 delim 구분자를 기준으로 count 수만큼 반환받는다. 음수이면 뒤에서부터 카운터한다.  

위의 쿼리에서 아래와 같이 추출하면, id가 없는 경우 url값이 그대로 들어간다.  
substring_index(url, '?id=', -1)

<br>

### 5-3. 문자열을 배열로 분해하기
split string to array

MySQL에는 SPLIT 함수가 없다. 따라서 SUBSTRING_INDEX를 적절히 활용하였다.

```sql
SELECT
    stamp,
    url,
    SUBSTRING_INDEX(SUBSTRING_INDEX(SUBSTRING_INDEX(SUBSTR(url, 24), '?', 1), '#', 1), '/', 1) AS path1,
    CASE
        WHEN SUBSTRING_INDEX(SUBSTRING_INDEX(SUBSTR(url, 24), '?', 1), '#', 1) RLIKE '\/'
            THEN SUBSTRING_INDEX(SUBSTRING_INDEX(SUBSTRING_INDEX(SUBSTR(url, 24), '?', 1), '#', 1), '/', -1)
		    ELSE ''
    END AS path2
FROM access_log;
```
<br>

### 5-4. 날짜와 타임스탬프 다루기
* `timestamp`는 시간대를 포함한 시간 정보를 저장하는 데이터 타입
* 시간대를 변경하면 `timestamp` 데이터 타입은 영향을 받는다.
* 참고 : <a href="http://mysqldba.tistory.com/279">MySQL에서의 시간 정보 관리하기</a>


#### 현재 날짜와 타임스탬프 추출하기
```sql
-- MySQL 8.0
SELECT
  CURDATE() AS dt,
  CURRENT_TIMESTAMP() AS stamp;
```

* 미들웨어에 따라 자료형/함수의 차이가 크며, 같은 함수라도 리턴값이 다름
* `CURRENT_TIMESTAMP()`는 타임존을 고려 필요

<br>

#### 문자열로 지정한 값의 날짜/시각 데이터 추출하기

``` sql
-- MySQL 8.0

SELECT
  -- 문자열을 날짜로 변환
  CAST('2018-05-04' AS date) AS dt1,
  STR_TO_DATE('2018-05-04 00:38:58', '%Y-%m-%d') AS dt2,
  -- 문자열을 타임스탬프로 변환
  UNIX_TIMESTAMP('2018-05-04 00:38:58') AS stamp,
  -- 타임스탬프를 일반적 날짜/시간 표현으로 변환
  FROM_UNIXTIME(UNIX_TIMESTAMP('2018-05-04 00:38:58'))
    AS stamp_to_dt;

```

* 문자열을 날짜/타임스탬프 자료형으로 변환할 때는 `CAST`가 가장 범용적임
* 그러나 MySQL은 `CAST`로 date, time, datetime만 형변환 가능
  * MySQL 8.0 Reference Manual :
<a href="https://dev.mysql.com/doc/refman/8.0/en/cast-functions.html"> CAST Functions and Operators</a>
* 따라서 `UNIX_TIMESTAMP()` 사용
  * 입력값은 완벽한 날짜 형식이어야 함.
  * 필요 시 STR_TO_DATE()를 이용해 문자열을 올바른 형식으로 바꾼 뒤 타임스탬프로 변환
  * 참고 : <a href="https://stackoverflow.com/questions/38896445/converting-date-time-string-to-unix-timestamp-in-mysql
">Converting date/time string to unix timestamp in MySQL
</a>


### 5-5. 결손 값을 디폴트 값으로 대치하기
* NULL과 문자열을 결합해도, 숫자를 사칙연산해도 NULL이 됨.
  그러므로 NULL을 반드시 다른 값으로 바꿔줘야 함.
* `COALESCE()` : 지정한 값이 NULL이면 다른 값으로 반환해 줌
* `COALESCE(column, value)`

```sql
-- 구매액에서 쿠폰 값을 제외한 매출 금액을 구하는 쿼리
SELECT
    purchase_id,
    amount,
    coupon,
    amount - COALESCE(coupon, 0) AS discount_amount
--쿠폰 사용 여부가 있는 구매로그 테이블
FROM purchase_log_with_coupon;
```
