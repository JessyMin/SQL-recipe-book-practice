## 5-4. 날짜와 타임스탬프 다루기

참고 :
* https://dev.mysql.com/doc/refman/8.0/en/date-and-time-functions.html
* http://www.mysqlkorea.com/sub.html?mcode=manual&scode=01&lang=k&ver_name=5


### 현재 날짜와 타임스탬프 추출

```
SELECT
    CURRENT_DATE() AS dt,
    CURRENT_TIMESTAMP() AS stamp;
```

### 지정한 값의 날짜/시각 데이터 추출

```
SELECT
    STR_TO_DATE('2018-04-18 10:02:06', '%Y-%m-%d');
```

STR_TO_DATE(str, format)
str과 format의 형식이 같아야 함.

```
--잘못된 쿼리
SELECT
    STR_TO_DATE("2018-04-18 10:02:06", "%Y, %m, %d") AS dt;
```
--> 형식이 다르면 NULL을 리턴함


### 문자열을 날짜 자료형, 타임스탬프 자료형으로 변환
```
SELECT
    STR_TO_DATE("2018-04-18", "%Y-%m-%d") AS dt,
    STR_TO_DATE("2018-04-18 10:02:06", "%Y-%m-%d %H:%i:%s") AS stamp;


SELECT
    UNIX_TIMESTAMP(STR_TO_DATE("2018-04-18 10:02:06", "%Y-%m-%d %H:%i:%s")) AS stamp;
```

### 날짜/시각에서 특정 필드 추출하기

#### 1) 타임스탬프/날짜 자료형에서 특정 필드 추출
```
SELECT
    stamp,
    DATE_FORMAT(stamp, "%Y") AS year,
    DATE_FORMAT(stamp, "%m") AS month,
    DATE_FORMAT(Stamp, "%d") AS day,
    DATE_FORMAT(Stamp, "%H") AS hour,
    DATE_FORMAT(stamp, "%Y-%m") AS year__month
FROM access_log;
```

<br>


#### 2) 타임스탬프를 문자열로 취급해 필드 추출
```
SELECT
	stamp,
    SUBSTR(stamp, 1, 4) AS year,
    SUBSTR(stamp, 6, 2) AS month,
    SUBSTR(stamp, 9, 2) AS day,
    SUBSTR(stamp, 12, 2) AS hour,
    SUBSTR(stamp, 1, 7) AS year__month
FROM access_log;
```
