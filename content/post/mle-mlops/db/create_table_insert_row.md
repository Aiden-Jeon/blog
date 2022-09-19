---
title: DB-2) Create Table and Insert Row
categories: [mle-mlops]
tags: ["postgres", "docker", "db"]
toc: true
date: 2022-09-16
author: Jongseob Jeon
---

[Github](https://github.com/Aiden-Jeon/mle-mlops/tree/main/01_db) 에서 해당 내용에 대해서 확인할 수 있습니다.

## Overview
### 목표

- 앞서 띄운 postgres 서버에 테이블을 생성하고 row를 삽입해 봅시다.

### 요구사항

1. Python의 `psycopg2` 패키지를 이용합니다.
    - `pip install psycopg2-binary`
2. psycopg2 를 사용해 `iris_data` 이름을 가진 table을 만들어 봅시다. [[Postgres Create Table](https://www.postgresql.org/docs/current/sql-createtable.html)]
    - table은 다음과 같은 column 을 갖고 있어야 합니다.
    
    | id | sepal length (cm) | sepal width (cm) | petal length (cm) | petal width (cm) | target |
    | --- | --- | --- | --- | --- | --- |
    | serial (primary key) | float | float | float | float | int |
    - table을 생성할 때 아래 내용을 참고합니다.
        - 파이썬에서 표시되는 float64 는 어떻게 처리해야 할까요? [[Postgres Data Types](https://www.postgresql.org/docs/current/datatype-numeric.html)]
        - column 명은 어떻게 입력해야 할까요? [[Postgres Names](https://www.postgresql.org/docs/7.0/syntax525.htm)]
    - primary key는 어떤 역할을 하나요? 꼭 넣어야 할까요?
        - +) serial 타입은 어떤 역할을 할까요?
    - 이미 테이블이 있는데 다시 생성을 요청하면 어떻게 되나요? 어떻게 방지할 수 있을까요?
3. psycopg2 를 사용해 iris 데이터 하나를 삽입해 봅시다.
4. `psql` 등을 이용해 생성한 테이블과 삽입한 데이터를 확인합니다.
---

## 1.  Requirements

이번 목표를 진행하기 앞서 다음 패키지들을 설치합니다.
```bash
pip install pandas psycopg2-binary scikit-learn
```

## 2. psycopg2 를 사용해 `iris_data` 이름을 가진 table을 만들어 봅시다. 

### 2.1 DB Connect
`psycopg2` 를 이용하기 위해서는 `connect` 함수를 이용해 database에 접근해야 합니다. database에 접근할 수 있는 connector를 `db_connect`에 선언합니다.

```python
import psycopg2

db_connect = psycopg2.connect(host="localhost", database="postgres", user="postgres", password="mypassword")
```

### 2.2 Create Table Query

삽입할 iris 데이터는 scikit-learn 패키지의 `load_iris`을 이용합니다.

```python
import pandas as pd
from sklearn.datasets import load_iris


X, y = load_iris(return_X_y=True, as_frame=True)
df = pd.concat([X, y], axis="columns")
```

iris 데이터를 확인하면 다음과 같습니다.
```python
>>> df.dtypes
sepal length (cm)    float64
sepal width (cm)     float64
petal length (cm)    float64
petal width (cm)     float64
target                 int64
dtype: object
```
X 의 data type은 float64로 target은 int64로 표기됩니다. 그런데 이 데이터 타입들은 postgres에서 사용할 수 없기에 각각 float8, int로 선언해야 합니다. 또한 column 이름은 `sepal length (cm)` 에 포함되어 있는 `(` 때문에 이용할 수 없기 때문에 해당 부분을 제거합니다.

위의 내용을 반영한 query문은 다음과 같이 작성할 수 있습니다.

```python
create_table_query = """
CREATE TABLE iris_data (
    id SERIAL PRIMARY KEY,
    sepal_width float8,
    sepal_length float8,
    petal_width float8,
    petal_length float8
    target int
);"""
```

### 2.3 Execute Query

이제 선언한 query문을 db에 전달합니다.

```python
with db_connect.cursor() as cur:
    cur.execute(create_table_query)
    db_connect.commit()
```

`excute` 함수를 통해 query를 전달할 수 있습니다. 작성 후 `commit` 까지 해야 정상적으로 query문이 수행됩니다.

### 2.4 Full Code

아래 코드는 위의 내용을 정리한 전체 코드입니다.

```python
# create_table.py
import psycopg2


def create_table(db_connect):
    create_table_query = """
    CREATE TABLE iris_data (
        id SERIAL PRIMARY KEY,
        sepal_width float8,
        sepal_length float8,
        petal_width float8,
        petal_length float8,
        target int
    );"""
    print(create_table_query)
    with db_connect.cursor() as cur:
        cur.execute(create_table_query)
        db_connect.commit()


if __name__ == "__main__":
    db_connect = psycopg2.connect(host="localhost", database="postgres", user="postgres", password="mypassword")
    create_table(db_connect)
```


위 코드를 최초 실행하면 다음과 같이 출력됩니다.

```zsh
> python create_table.py

    CREATE TABLE iris_data (
        id SERIAL PRIMARY KEY,
        sepal_width float8,
        sepal_length float8,
        petal_width float8,
        petal_length float8,
        target int
    );
```

하지만 두 번째 실행하면 다음과 같이 에러가 납니다.

```zsh
> python create_table_insert_row.py

    CREATE TABLE iris_data (
        id SERIAL PRIMARY KEY,
        sepal_width float8,
        sepal_length float8,
        petal_width float8,
        petal_length float8,
        target int
    );
Traceback (most recent call last):
  ...
psycopg2.errors.DuplicateTable: relation "iris_data" already exists
```

이를 해결하기 위해서는 query에 `IF NOT EXISTS` 옵션을 추가해야 합니다. 추가한 전체 코드는 아래와 같습니다.

```python
# create_table.py
import psycopg2


def create_table(db_connect):
    create_table_query = """
    CREATE TABLE IF NOT EXISTS iris_data (
        id SERIAL PRIMARY KEY,
        sepal_width float8,
        sepal_length float8,
        petal_width float8,
        petal_length float8,
        target int
    );"""
    print(create_table_query)
    with db_connect.cursor() as cur:
        cur.execute(create_table_query)
        db_connect.commit()


if __name__ == "__main__":
    db_connect = psycopg2.connect(host="localhost", database="postgres", user="postgres", password="mypassword")
    create_table(db_connect)
```

### 2.5 psql

psql cli 에 접속해서 생성된 테이블을 확인합니다.
```psql
postgres=# SELECT * FROM iris_data;
 id | sepal_width | sepal_length | petal_width | petal_length | target
----+-------------+--------------+-------------+--------------+--------
(0 rows)
```

## 3. psycopg2 를 사용해 iris 데이터 하나를 삽입해 봅시다.
### 3.1 Load iris data

iris 데이터를 생성하는 코드와 column name을 앞서 생성한 db table의 column name이 일치하도록 `rename` 함수를 이용해 수정합니다.

```python
import pandas as pd
from sklearn.datasets import load_iris


X, y = load_iris(return_X_y=True, as_frame=True)
df = pd.concat([X, y], axis="columns")
rename_rule = {
    "sepal length (cm)": "sepal_width",
    "sepal width (cm)": "sepal_length",
    "petal length (cm)": "petal_width",
    "petal width (cm)": "petal_length",
}
df = df.rename(columns=rename_rule)
```

### 3.2 Insert Query

다음으로 data를 삽입할 수 있는 query 문을 작성합니다. 데이터를 insert 하는 query문의 포맷은 아래와 같습니다.

```bash
INSRT INTO {table_name} (col_1, col_2, ...) VALUES (val_1, val_2, ...)
```

우선 `sample` 함수를 이용해 데이터 하나를 추출합니다.
```python
data = df.sample(1)
```

추출된 데이터를 이용해 query문을 작성합니다.
```python
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
```

### 3.3 Full Code

위의 내용을 종합해 전체 코드를 작성하면 아래와 같습니다.

```python
# insert_row.py
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


if __name__ == "__main__":
    db_connect = psycopg2.connect(host="localhost", database="postgres", user="postgres", password="mypassword")
    df = get_data()
    insert_row(db_connect, df.sample(1))

```

위 스크립트를 실행하면 아래와 같이 출력됩니다.

```zsh
> python insert_row.py

    INSERT INTO iris_data
        (sepal_width, sepal_length, petal_width, petal_length, target)
        VALUES (
            5.9,
            3.0,
            5.1,
            1.8,
            2
        );
```

### 3.4 psql
`psql`을 이용해 삽입한 데이터를 확인합니다.

```psql
postgres=# SELECT * FROM iris_data;
 id | sepal_width | sepal_length | petal_width | petal_length | target
----+-------------+--------------+-------------+--------------+--------
  1 |         5.9 |            3 |         5.1 |          1.8 |      2
(1 row)
```
