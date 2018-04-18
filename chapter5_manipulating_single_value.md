## 5-2.URL에서 요소 추출하기


### 1) 레퍼러에서 어떤 웹페이지를 거쳐 넘어왔는지 판별하기
```
SELECT
		stamp,
    substring_index(
		substring_index(referrer, '//', -1), '/', 1) AS referrer_host
FROM access_log;
```
<br>


### 2) URL에서 경로와 요청 매개변수 값 추출하기

이런 경우 접근방법은 두 가지다.
1) 구분자로 잘라서, 해당되는 내용만 가져온다. (mysql : REGEXP_SUBSTR)
2) 불필요한 부분을 ''로 replace한다. (mysql : REGEXP_REPLACE)
3) 필요한 부분을 extract한다.

```
SELECT
    stamp,
    url,
    substr(substring_index(substring_index(url, '?id=', 1), '#', 1), 23) AS path,
    CASE
			WHEN length(substring_index(url, '?id=', -1))=3 THEN substring_index(url, '?id=', -1)
      ELSE ''
		END AS id
FROM access_log;
```

* MySQL에는 REGEXP_SUBSTR 함수가 없다.
 The MySQL REGEXP can be used for matching strings, but not for transforming them.
Unfortunately, MySQL's regular expression function return true, false or null depending if the expression exists or not.
(참고 : https://dba.stackexchange.com/questions/34724/how-to-use-substring-using-regexp-in-mysql)

* substring_index 함수
: SELECT SUBSTRING_INDEX(str, delim, count);
: str 문자열의 delim 구분자를 기준으로 count 수만큼 반환받는다. 음수이면 뒤에서부터 카운터한다.

위의 쿼리에서 아래와 같이 추출하면, id가 없는 경우 url값이 그대로 들어간다.
substring_index(url, '?id=', -1)

<br>

## 5-2. 문자열을 배열로 분해하기
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

## 5-5. 결손 값을 디폴트 값으로 대치하기
* NULL과 문자열을 결합하면 NULL이 되고, 숫자를 사칙연산해도 NULL이 됨.
* COALESCE() : 지정한 값이 NULL이면 다른 값으로 반환해 줌
  COALESCE(column, value)

```
-- 구매액에서 쿠폰 값을 제외한 매출 금액을 구하는 쿼리
SELECT
    purchase_id,
    amount,
    coupon,
    amount - COALESCE(coupon, 0) AS discount_amount
--쿠폰 사용 여부가 있는 구매로그 테이블
FROM purchase_log_with_coupon;
```
