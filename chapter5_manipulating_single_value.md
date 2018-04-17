## 5-2.URL에서 요소 추출하기

DROP TABLE IF EXISTS access_log ;
CREATE TABLE access_log (
    stamp    varchar(255)
  , referrer text
  , url      text
);

INSERT INTO access_log
VALUES
    ('2016-08-26 12:02:00', 'http://www.other.com/path1/index.php?k1=v1&k2=v2#Ref1', 'http://www.example.com/video/detail?id=001')
  , ('2016-08-26 12:02:01', 'http://www.other.net/path1/index.php?k1=v1&k2=v2#Ref1', 'http://www.example.com/video#ref'          )
  , ('2016-08-26 12:02:01', 'https://www.other.com/'                               , 'http://www.example.com/book/detail?id=002' )
;


### 1) 레퍼러에서 어떤 웹페이지를 거쳐 넘어왔는지 판별하기
SELECT
		stamp,
    substring_index(
		substring_index(referrer, '//', -1), '/', 1) AS referrer_host
FROM access_log;

### 2) URL에서 경로와 요청 매개변수 값 추출하기

이런 경우 접근방법은 두 가지다.
1) 구분자로 잘라서, 해당되는 내용만 가져온다. (mysql : REGEXP_SUBSTR)
2) 불필요한 부분을 ''로 replace한다. (mysql : REGEXP_REPLACE)
3) 필요한 부분을 extract한다.


SELECT
		stamp,
    url,
    substr(substring_index(substring_index(url, '?id=', 1), '#', 1), 23) AS path,
    CASE
			WHEN length(substring_index(url, '?id=', -1))=3 THEN substring_index(url, '?id=', -1)
      ELSE ''
		END AS id
FROM access_log;

* MySQL에는 REGEXP_SUBSTR 함수가 없다.
 The MySQL REGEXP can be used for matching strings, but not for transforming them.

* substring_index 함수
: SELECT SUBSTRING_INDEX(str, delim, count);
: str 문자열의 delim 구분자를 기준으로 count 수만큼 반환받는다. 음수이면 뒤에서부터 카운터한다.

위의 쿼리에서 아래와 같이 추출하면, id가 없는 경우 url값이 그대로 들어간다.
substring_index(url, '?id=', -1)


## 5-2. 문자열을 배열로 분해하기
