# 0. 테이블 생성
## 0.1
```SQL
CREATE EXTERNAL TABLE users (
    User_ID INT,
    Location STRING,
    Age INT
)
ROW FORMAT SERDE 'org.apache.hadoop.hive.serde2.OpenCSVSerde'
WITH SERDEPROPERTIES (
   "separatorChar" = ";",
   "quoteChar"     = "\""
)
STORED AS TEXTFILE
LOCATION '/user/ubuntu/input/users'
TBLPROPERTIES ("skip.header.line.count"="1");
```
## 0.2 
- users의 age > int로 변경하기
- users의 user_id > int로 변경하기
```SQL
-- 결측값 확인
SELECT user_id, COUNT(*)
FROM users
WHERE user_id IS NULL
OR TRIM(user_id) = ''
GROUP BY user_id;

-- 새로운 테이블 생성
CREATE TABLE users_cleaned AS
SELECT
    CASE
        WHEN TRIM(User_ID) = '' OR NOT User_ID RLIKE '^[0-9]+$' THEN NULL
        ELSE CAST(User_ID AS INT)
    END AS User_ID,
    Location,
    CASE
        WHEN TRIM(age) = '' OR NOT age RLIKE '^[0-9]+$' THEN NULL
        ELSE CAST(age AS INT)
    END AS age
FROM users;

--테이블 덮어쓰기
DROP TABLE users;
ALTER TABLE users_cleaned RENAME TO users;
```

## 0.3 books의 year_of_publication > int로 변경
- 데이터 확인
```SQL
-- 1. 네 자리 숫자가 아닌 값 확인
SELECT year_of_publication, COUNT(*)
FROM books
WHERE NOT year_of_publication RLIKE '^[0-9]{4}$'
GROUP BY year_of_publication;

-- 2. 빈 값(null 또는 빈 문자열) 확인
SELECT year_of_publication, COUNT(*)
FROM books
WHERE year_of_publication IS NULL OR TRIM(year_of_publication) = ''
GROUP BY year_of_publication;
```

- 데이터 변환 및 기존값 대체
```SQL
CREATE TABLE books_cleaned AS
SELECT
    ISBN,
    Book_Title,
    Book_Author,
    CASE
        WHEN Year_Of_Publication RLIKE '^[0-9]{4}$' THEN CAST(Year_Of_Publication AS INT)
        ELSE NULL
    END AS Year_Of_Publication,
    Publisher
FROM books;

DROP TABLE books;

ALTER TABLE books_cleaned RENAME TO books;
```

# 1. 데이터의 기초 정보 확인
## 1.1 중복 데이터 확인
- Books 테이블에서 중복된 ISBN 확인
```SQL
SELECT
    books.isbn AS isbn,
    COUNT(books.isbn) AS _c1
FROM books
GROUP BY books.isbn
HAVING COUNT(books.isbn) > 1;
```

```Plain Text
+-------+------+
| isbn  | _c1  |
+-------+------+
+-------+------+
```
- Ratings 테이블에서 중복된 사용자-책 평가 확인
```SQL
select
    ratings.user_id, ratings.isbn, COUNT(ratings.user_id)
from ratings
group by
    ratings.user_id, ratings.isbn
having
    COUNT(ratings.user_id) > 1;
```
```Plain Text
+----------+-------+------+
| user_id  | isbn  | _c2  |
+----------+-------+------+
+----------+-------+------+
```
## 1.2 결측값 확인
- Users 테이블에서 Age의 결측값 확인
```SQL
SELECT COUNT(users.user_id)
FROM users
WHERE users.age IS NULL;
```
```Plain Text
+---------+
|   _c0   |
+---------+
| 110762  |
+---------+
```
- Books 테이블에서 Year_Of_Publication의 결측값 확인
```SQL
select COUNT(isbn)
from books
where books.year_of_publication is null;
```
```Plain Text
+------+
| _c0  |
+------+
| 0    |
+------+
```
# 2. 데이터의 기초 통계 확인
## 2.1 사용자 연령 통계 확인
- 사용자 연령의 기초 통계(최소, 최대, 평균)를 확인합니다.
```SQL
select
    MIN(users.age) as min_age,
    MAX(users.age) as max_age,
    (SUM(users.age) / COUNT(users.age)) as avg_age
from users;
```
```Plain Text
+----------+----------+--------------------+
| min_age  | max_age  |      avg_age       |
+----------+----------+--------------------+
| 0        | 244      | 34.75143370454978  |
+----------+----------+--------------------+
```
## 2.2 출판 연도 통계 확인
-
```SQL
select
    MIN(year_of_publication) as min_year,
    MAX(year_of_publication) as max_year,
    (SUM(year_of_publication) / COUNT(year_of_publication)) as avg_year
from books;
```
```Plain Text
+-----------+-----------+---------------------+
| min_year  | max_year  |      avg_year       |
+-----------+-----------+---------------------+
| 0         | 2050      | 1959.7560496574902  |
+-----------+-----------+---------------------+
```

## 2.3 평점의 분포 확인
```SQL
SELECT book_rating, COUNT(*)
From ratings
GROUP BY book_rating;
```
```Plain Text
+--------------+---------------+
| book_rating  | rating_count  |
+--------------+---------------+
| 0            | 716109        |
| 1            | 1770          |
| 2            | 2759          |
| 3            | 5996          |
| 4            | 8904          |
| 5            | 50974         |
| 6            | 36924         |
| 7            | 76457         |
| 8            | 103736        |
| 9            | 67541         |
| 10           | 78610         |
+--------------+---------------+
```
# 3. 데이터의 주요 패턴 탐색
## 3.1 출판사별 책 수 및 평균 평점
- 출판사별로 얼마나 많은 책이 있는지, 그리고 그 책들의 평균 평점이 어떤지 확인합니다.

```SQL
SELECT b.publisher, count(b.book_title) as book_count, (sum(r.book_rating) / count(book_rating)) as avg_rating 
from books b
join ratings r
on b.isbn = r.isbn
group by b.publisher
order by book_count desc
limit 10;
```
```Plain Text
+---------------------------+-------------+---------------------+
|        b.publisher        | book_count  |     avg_rating      |
+---------------------------+-------------+---------------------+
| Ballantine Books          | 34724       | 2.8012037783665478  |
| Pocket                    | 31989       | 2.4986714183000407  |
| Berkley Publishing Group  | 28614       | 2.4248270077584397  |
| Warner Books              | 25506       | 2.6712930290911943  |
| Harlequin                 | 25029       | 1.4398098206080947  |
| Bantam Books              | 23600       | 2.327627118644068   |
| Bantam                    | 20007       | 2.8771929824561404  |
| Signet Book               | 19155       | 2.6849908640041766  |
| Avon                      | 17352       | 2.447498847395113   |
| Penguin Books             | 17033       | 3.203898315035519   |
+---------------------------+-------------+---------------------+
```
## 3.2 가장 많이 평가된 책과 그 평점
- 가장 많이 평가된 책이 무엇인지 확인합니다
```SQL
SELECT
    b.book_title, COUNT(r.isbn) AS rating_count,
    SUM(r.book_rating) / COUNT(r.book_rating) AS avg_ratings
FROM books b
JOIN ratings r ON b.isbn = r.isbn
GROUP BY b.book_title
ORDER BY rating_count DESC
LIMIT 10;
```
```Plain Text
+--------------------------------------------------+---------------+---------------------+
|                   b.book_title                   | rating_count  |     avg_rating      |
+--------------------------------------------------+---------------+---------------------+
| Wild Animus                                      | 2502          | 1.0195843325339728  |
| The Lovely Bones: A Novel                        | 1295          | 4.468725868725869   |
| The Da Vinci Code                                | 898           | 4.642538975501114   |
| A Painted House                                  | 838           | 3.231503579952267   |
| The Nanny Diaries: A Novel                       | 828           | 3.5301932367149758  |
| Bridget Jones's Diary                            | 815           | 3.5276073619631902  |
| The Secret Life of Bees                          | 774           | 4.447028423772609   |
| Divine Secrets of the Ya-Ya Sisterhood: A Novel  | 740           | 3.437837837837838   |
| The Red Tent (Bestselling Backlist)              | 723           | 4.334716459197787   |
| Angels & Demons                                  | 670           | 3.708955223880597   |
+--------------------------------------------------+---------------+---------------------+
```

# 4. 데이터 관계 분석
## 4.1 책 평점과 출판 연도 간의 관계
책의 출판 연도와 평점 간의 관계를 확인합니다.
```SQL
select b.year_of_publication, (sum(r.book_rating) / count(book_rating)) as avg_rating 
from books b
join ratings r
on b.isbn = r.isbn
group by b.year_of_publication
order by b.year_of_publication;
```
```Plain Text
+------------------------+---------------------+
| b.year_of_publication  |     avg_rating      |
+------------------------+---------------------+
| 0                      | 3.1328336902212706  |
| 1376                   | 4.0                 |
| 1378                   | 10.0                |
| 1806                   | 5.0                 |
| 1897                   | 0.0                 |
| 1900                   | 1.75                |
| 1901                   | 4.571428571428571   |
| 1902                   | 1.8                 |
| 1904                   | 10.0                |
| 1906                   | 5.0                 |
| 1908                   | 10.0                |
| 1909                   | 0.0                 |
| 1910                   | 0.0                 |
| 1911                   | 2.1153846153846154  |
| 1914                   | 0.0                 |
| 1917                   | 0.0                 |
| 1919                   | 0.0                 |
| 1920                   | 3.15                |
...
...
| 2021                   | 5.333333333333333   |
| 2024                   | 0.0                 |
| 2026                   | 4.8                 |
| 2030                   | 3.03125             |
| 2037                   | 10.0                |
| 2038                   | 2.375               |
| 2050                   | 4.857142857142857   |
+------------------------+---------------------+

```

## 4.2 사용자 위치별 평점 차이
- 위치에 따라 평균 평점을 출력합니다. 적어도 10개 이상의 평가를 한 경우만 출력합니다.
```SQL
select u.location,
    (sum(r.book_rating) / count(book_rating)) as avg_rating,
    count(r.book_rating) as rating_count
from users u
join ratings r
on u.user_id = r.user_id
group by u.location
having rating_count > 10
order by avg_rating desc
limit 10;
```
```Plain Text
+----------------------------------------+--------------------+---------------+
|               u.location               |     avg_rating     | rating_count  |
+----------------------------------------+--------------------+---------------+
| centerville, georgia, usa              | 9.894736842105264  | 19            |
| mountain city, tennessee, usa          | 9.833333333333334  | 12            |
| m�xico, d.f., disrito federal, mexico  | 9.588235294117647  | 17            |
| bryant, alabama, usa                   | 9.448275862068966  | 29            |
| pillager, minnesota, usa               | 9.4                | 15            |
| fairview, new jersey, usa              | 9.264150943396226  | 53            |
| lakeland, tennessee,                   | 9.243243243243244  | 37            |
| w.m., pennsylvania, usa                | 9.21311475409836   | 61            |
| tulsa, tasmania, usa                   | 9.0                | 11            |
| san francsico, california, usa         | 9.0                | 14            |
+----------------------------------------+--------------------+---------------+
```

## 4.3 책 저자별 평균 평점
- 각 저자별로 평균 평점이 어떻게 다른지 확인합니다. 적어도 10개 이상의 평가를 한 경우만 출력합니다.
```SQL
select b.book_author, (sum(r.book_rating) / count(book_rating)) as avg_rating, count(r.book_rating) as rating_count
from books b
join ratings r
on b.isbn = r.isbn
group by b.book_author
having count(r.book_rating) >= 10
order by avg_rating desc
limit 10;
```
```Plain Text
+-------------------+--------------------+---------------+
|   b.book_author   |     avg_rating     | rating_count  |
+-------------------+--------------------+---------------+
| Michiro Ueyama    | 10.0               | 13            |
| Mitsumasa Anno    | 9.636363636363637  | 11            |
| Penny Frank       | 9.4375             | 32            |
| Peter Abrams      | 9.416666666666666  | 12            |
| Daisaku Ikeda     | 9.161290322580646  | 31            |
| Jonathan Pearce   | 8.833333333333334  | 12            |
| Hayao Miyazaki    | 8.476190476190476  | 21            |
| Wataru Yoshizumi  | 8.461538461538462  | 13            |
| Naoko Takeuchi    | 8.13953488372093   | 43            |
| Andi Watson       | 8.083333333333334  | 12            |
+-------------------+--------------------+---------------+
```

## 4.4 평점 상위 10개의 책이 출판된 연도 분포
- 평점 상위 10개의 책이 어느 연도에 많이 출판되었는지 확인합니다.
    - 평점이 높은 책 찾기
    - 평점을 이용해서...
```SQL
-- 평점 상위 10개 책 찾기 (평가가 10개 이상인 경우)
select b.book_title,
    b.year_of_publication,
    (sum(r.book_rating) / count(r.book_rating)) as avg_rating
from books b
join ratings r
on b.isbn = r.isbn
group by b.book_title, b.year_of_publication
having count(r.book_rating) >= 10
order by avg_rating desc
limit 10;

-- 이 책들을 연도별로 묶어서 연도 개수 출력
SELECT b.year_of_publication, COUNT(b.book_title) AS cnt_book_title
FROM books b
JOIN 
    (SELECT a.isbn, 
            SUM(c.book_rating) / COUNT(c.book_rating) AS avg_rating
     FROM ratings c
     JOIN books a
     ON a.isbn = c.isbn
     GROUP BY a.isbn
     ORDER BY avg_rating DESC
     LIMIT 10) r
ON b.isbn = r.isbn
GROUP BY b.year_of_publication
ORDER BY cnt_book_title DESC;
```
```Plain Text
+----------------------+------------------+
| year_of_publication  | number_of_books  |
+----------------------+------------------+
| 1996                 | 3                |
| 2003                 | 2                |
| 2001                 | 1                |
| 1999                 | 1                |
| 1998                 | 1                |
| 1990                 | 1                |
| 1988                 | 1                |
+----------------------+------------------+
```
- 기타
```SQL
-- 테이블 제거
DROP TABLE IF EXISTS ratings;
-- 테이블 정보 확인
DESCRIBE books;
```