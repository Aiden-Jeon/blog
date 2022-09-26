---
title: "[ Chapter 4. Model API ] 2) Write Data to DB"
categories: [mle-mlops]
tags: ["model-registry", "api", "db"]
toc: true
date: 2022-09-26 19:00:00+09:00
author: Jongseob Jeon
---

[Github](https://github.com/Aiden-Jeon/mle-mlops/tree/main/04_model_api) 에서 해당 내용에 대해서 확인할 수 있습니다.

## Overview
### 목표

- `predict` 로 요청받은 데이터와 그 결과를 DB에 저장합니다.

### 요구사항

1. `database.py` 를 작성해 db에 연결합니다.
2. `models.py` 를 작성합니다.
    - `raw_data` table columns:
        - 입력으로 받은 column들
        - `timestamp`: 요청받은 시간
    - `prediction` table columns:
        - `iris_class` 모델의 결과와
        - `timestamp`: 요청받은 시간
3. 작성한 내용으로 `crud.py` 를 수정합니다.
4. request를 날린후 `raw_data` table 과 `prediction` table을 확인합니다.
5. query문을 작성해 input 과 prediction 을 같이 확인합니다.

### 질문사항

1. 모델을 디버깅하기 위해서 입력값과 결과값을 머지하기 위해서는 어떻게 해야할까요?
    - `timestamp` or `uuid` ?
2. 왜 input_schema 와 output_schema가 필요할까요?

---

## Database
앞서 작성했던 자료를 참고해 다음과 같이 `database.py`를 작성합니다.

```python
import os
from sqlalchemy import create_engine
from sqlalchemy.ext.declarative import declarative_base
from sqlalchemy.orm import sessionmaker


SQLALCHEMY_DATABASE_URL = os.getenv("SQLALCHEMY_DATABASE_URL", "postgresql://postgres:mypassword@localhost:5432/postgres")

engine = create_engine(SQLALCHEMY_DATABASE_URL)
SessionLocal = sessionmaker(autocommit=False, autoflush=False, bind=engine)

Base = declarative_base()
```

## Models
Database에서 사용할 `models.py` 를 정의합니다.

### Input Model
입력으로 받은 데이터에서 필요한 추가 column인 timestamp를 추가해 작성합니다.

```python
from sqlalchemy import Column, Integer, DateTime, Float

from database import Base


class DataIn(Base):
    __tablename__ = "raw_data"

    id = Column(Integer, primary_key=True, index=True)
    timestamp = Column(DateTime)
    sepal_width = Column(Float)
    sepal_length = Column(Float)
    petal_width = Column(Float)
    petal_length = Column(Float)
```


### Output Model
모델의 예측결과를 `iris_class` column에 저장합니다.

```python
class DataOut(Base):
    __tablename__ = "prediction"

    id = Column(Integer, primary_key=True, index=True)
    timestamp = Column(DateTime)
    iris_class = Column(Integer)
```

## Crud
database 와 models 내용을 app 에 추가해 작성합니다.

필요한 패키지를 불러옵니다.
```python
import os
from datetime import datetime

import mlflow
import pandas as pd
from fastapi import Depends, FastAPI
from sqlalchemy.orm import Session

from database import SessionLocal, engine
from models import Base, DataIn, DataOut
from schemas import PredictIn, PredictOut
```

다음으로 db 의 table을 자동으로 생성합니ㅏㄷ.
```python
Base.metadata.create_all(bind=engine)
```

app을 선언하고 db를 불러올 수 있는 코드를 작성합니다.
```python
app = FastAPI()


def get_db():
    db = SessionLocal()
    try:
        yield db
    finally:
        db.close()
```

사용할 모델을 전역 변수로 선언합니다.
```python
MODEL = mlflow.pyfunc.load_model("../model")
```

predict 함수를 작성합니다.

```python
@app.post("/predict", response_model=PredictOut)
def predict(data: PredictIn, db: Session = Depends(get_db)):
    now = datetime.utcnow()

    df = pd.DataFrame([data.dict()])
    pred = int(MODEL.predict(df))

    data_in = DataIn(timestamp=now, **data.dict())
    db.add(data_in)
    db.commit()
    db.refresh(data_in)

    data_out = DataOut(timestamp=now, iris_class=pred)
    db.add(data_out)
    db.commit()
    db.refresh(data_out)
    return PredictOut(iris_class=pred)
```

request 를 요청받은 시간을 `now` 에 저장합니다.  
그 이후 모델의 예측을 수행합니다.

예측을 수행한 후 입력으로 들어온 데이터를 commit 합니다.  
다음으로 모델의 결과를 commit 합니다.

## 실행
uvicorn 을 이용해 app을 실행합니다.

```bash
uvicorn crud:app --reload
```

web에서 예제를 실행합니다.

`psql`을 이용해 database를 확인합니다.
- raw_data
    ```psql
    postgres=# SELECT * FROM raw_data;
    id |         timestamp          | sepal_width | sepal_length | petal_width | petal_length
    ----+----------------------------+-------------+--------------+-------------+--------------
    1 | 2022-09-26 05:47:41.517851 |           0 |            0 |           0 |            0
    (1 rows)
    ```
- prediction
    ```psql
    postgres=# SELECT * FROM prediction;
    id |         timestamp          | iris_class
    ----+----------------------------+------------
    1 | 2022-09-26 05:47:41.517851 |          2
    (1 rows)
    ```
