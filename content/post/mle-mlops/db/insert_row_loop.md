---
title: "[ Chapter 1. Database ] 3) Insert Row Loop"
categories: [mle-mlops]
tags: ["postgres", "db"]
toc: true
date: 2022-09-19 14:30:00+09:00
author: Jongseob Jeon
---

[Github](https://github.com/Aiden-Jeon/mle-mlops/tree/main/01_db) 에서 해당 내용에 대해서 확인할 수 있습니다.

## Overview
### 목표

- postgres에 계속해서 데이터를 insert 하는 스크립트를 작성합니다.

### 요구사항

1. 앞서 작성한 스크립트를 기반으로 5초마다 iris 데이터 하나를 삽입하는 python script를 작성합니다.
2. `psql` 등을 이용해 데이터가 계속해서 삽입되고 있는지 확인합니다.

---

## Script

앞선 포스트에서 작성한 `insert_row` 함수에 while문을 추가합니다.  
그리고 요구사항에 맞춰서 `time` 패키지의 `sleep` 함수를 이용해 5초간 멈추게합니다.

```python
import time

def insert_row_loop(db_connect, df):
    while True:
        insert_row(db_connect, df.sample(1))
        time.sleep(5)
```


전체 코드는 아래와 같습니다.

```python
# insert_data_loop.py
import time
import pandas as pd
import psycopg2
from sklearn.datasets import load_iris


def get_data():
    X, y = load_iris(return_X_y=True, as_frame=True)
    df = pd.concat([X, y], axis="columns")
    rename_rule = {
        "sepal length (cm)": "sepal_width",
        "sepal width (cm)": "sepal_length",
        "petal length (cm)": "petal_width",
        "petal width (cm)": "petal_length",
    }
    df = df.rename(columns=rename_rule)
    return df


def insert_row(db_connect, data):
    insert_row_query = f"""
    INSERT INTO iris_data
        (sepal_width, sepal_length, petal_width, petal_length, target)
        VALUES (
            {data.sepal_width.values[0]},
            {data.sepal_length.values[0]},
            {data.petal_width.values[0]},
            {data.petal_length.values[0]},
            {data.target.values[0]}
        );
    """
    print(insert_row_query)
    with db_connect.cursor() as cur:
        cur.execute(insert_row_query)
        db_connect.commit()


def insert_row_loop(db_connect, df):
    while True:
        insert_row(db_connect, df.sample(1))
        time.sleep(5)


if __name__ == "__main__":
    db_connect = psycopg2.connect(host="localhost", database="postgres", user="postgres", password="mypassword")
    df = get_data()
    insert_row_loop(db_connect, df)
```

## Run

스크립트를 실행합니다.

```python
python insert_data_loop.py
```

`psql`을 이용해 계속해서 데이터가 삽입중인지 확인합니다.

```psql
postgres=# SELECT * FROM iris_data;
 id | sepal_width | sepal_length | petal_width | petal_length | target
----+-------------+--------------+-------------+--------------+--------
  1 |         5.9 |            3 |         5.1 |          1.8 |      2
  2 |         5.7 |            3 |         4.2 |          1.2 |      1
  3 |         5.7 |            3 |         4.2 |          1.2 |      1
(3 rows)

postgres=# SELECT * FROM iris_data;
 id | sepal_width | sepal_length | petal_width | petal_length | target
----+-------------+--------------+-------------+--------------+--------
  1 |         5.9 |            3 |         5.1 |          1.8 |      2
  2 |         5.7 |            3 |         4.2 |          1.2 |      1
  3 |         5.7 |            3 |         4.2 |          1.2 |      1
  4 |         5.7 |            3 |         4.2 |          1.2 |      1
(4 rows)
```
