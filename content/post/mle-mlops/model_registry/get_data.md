---
title: "[ Chapter 2. Model Registry ] 2) Get Data from DB"
categories: [mle-mlops]
tags: ["model-registry", "db"]
toc: true
date: 2022-09-20 13:10:00+09:00
author: Jongseob Jeon
---

[Github](https://github.com/Aiden-Jeon/mle-mlops/tree/main/02_model_registry) 에서 해당 내용에 대해서 확인할 수 있습니다.

## Overview
### 목표

- 앞선 챕터에서 만든 DB에서 데이터를 추출합니다.

### 요구사항

1. `pandas.read_sql` 함수를 이용합니다.
2. `id` column을 기준으로 최신 데이터 100개를 추출하는 쿼리문을 작성합니다.
3. 데이터를 추출합니다.

---

## 1. `pandas.read_sql` 함수를 이용합니다.

`pandas.read_sql` 는 입력 argument로 Query문과 DB Connection을 받습니다.

## 2. `id` column을 기준으로 최신 데이터 100개를 추출하는 쿼리문을 작성합니다.

요구사항을 Query문으로 작성하면 다음과 같습니다.

```sql
SELECT * FROM iris_data ORDER BY id DESC LIMIT 10;
```

`psql` 에서 해당 쿼리문을 입력하면 다음과 같이 출력됩니다.

```psql
postgres=# SELECT * FROM iris_data ORDER BY id DESC LIMIT 10;
  id  | sepal_width | sepal_length | petal_width | petal_length | target
------+-------------+--------------+-------------+--------------+--------
 5789 |         4.4 |            3 |         1.3 |          0.2 |      0
 5788 |         5.1 |          3.7 |         1.5 |          0.4 |      0
 5787 |           5 |          3.4 |         1.6 |          0.4 |      0
 5786 |         5.3 |          3.7 |         1.5 |          0.2 |      0
 5785 |           5 |            3 |         1.6 |          0.2 |      0
 5784 |         6.1 |            3 |         4.9 |          1.8 |      2
 5783 |         6.7 |            3 |           5 |          1.7 |      1
 5782 |         6.9 |          3.1 |         5.4 |          2.1 |      2
 5781 |         6.8 |          3.2 |         5.9 |          2.3 |      2
 5780 |         5.9 |          3.2 |         4.8 |          1.8 |      1
(10 rows)
```

## 3. 데이터를 추출합니다.
postgres 에 연결할 수 있는 db connection을 생성 후 쿼리문과 db 커넥션을 이용해 데이터를 불러옵니다.

```python
import pandas as pd
import psycopg2


db_connect = psycopg2.connect(host="localhost", database="postgres", user="postgres", password="mypassword")
df = pd.read_sql("SELECT * FROM iris_data ORDER BY id DESC LIMIT 10", db_connect)
```

추출된 데이터를 확인하면 다음과 같습니다.

```python
>>> df
     id  sepal_width  sepal_length  petal_width  petal_length  target
0  5824          6.0           2.2          5.0           1.5       2
1  5823          5.7           4.4          1.5           0.4       0
2  5822          6.0           2.7          5.1           1.6       1
3  5821          6.3           2.3          4.4           1.3       1
4  5820          4.8           3.4          1.9           0.2       0
5  5819          4.5           2.3          1.3           0.3       0
6  5818          6.7           3.3          5.7           2.1       2
7  5817          5.2           4.1          1.5           0.1       0
8  5816          5.7           4.4          1.5           0.4       0
9  5815          5.4           3.0          4.5           1.5       1
```
