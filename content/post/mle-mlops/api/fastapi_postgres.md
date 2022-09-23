---
title: "[ Chapter 3. API ] 4) FastAPI with Postgresql"
categories: [mle-mlops]
tags: ["model-registry", "api", "fastapi", "postgresql"]
toc: true
date: 2022-09-23 20:40:00+09:00
author: Jongseob Jeon
---

[Github](https://github.com/Aiden-Jeon/mle-mlops/tree/main/03_api) 에서 해당 내용에 대해서 확인할 수 있습니다.

## Overview
### 목표

- 앞서 작성한 fastapi의 정보를 postgesql db에 작성합니다.

### 요구사항

1. Postgresql 서버를 실행합니다.
    - 데이터가 저장되고 있는 도커와는 다른 서버를 실행해주세요.
    - 포트가 사용중이라면 5433 으로 포워딩합니다.
2. 다음 과정에 따라 create 함수로 입력 받는 값을 postgresql 에 저장하게 합니다.
    1. ORM(Object Relational Mapping) 에 대해서 알아봅니다. [[ORMs](https://fastapi.tiangolo.com/tutorial/sql-databases/?h=pydantic#orms)]
    2. `database.py` 파일을 생성합니다.[[File Structure](https://fastapi.tiangolo.com/tutorial/sql-databases/?h=pydantic#file-structure)]
        - 튜토리얼을 따라 postgresql 과 연결할 수 있는 session_maker를 작성합니다. [[Create the SQLAlchemy parts](https://fastapi.tiangolo.com/tutorial/sql-databases/?h=pydantic#create-the-sqlalchemy-parts)]
    3. `models.py` 파일을 생성합니다.
        - tablename은 users로 합니다.
        - 튜토리얼을 따라 postgresql 에서 사용하는 모델을 생성합니다. [[Create the database model](https://fastapi.tiangolo.com/tutorial/sql-databases/?h=pydantic#create-the-database-models)]
        
        | Column | Type | Key |
        | --- | --- | --- |
        | id | Integer | Primary, Index |
        | name | String | Unique, Index |
        | nickname | String |  |
    4. `schemas.py` 파일을 생성합니다. [[create-pydantic-models-schemas-for-reading-returning](https://fastapi.tiangolo.com/tutorial/sql-databases/?h=pydantic#create-pydantic-models-schemas-for-reading-returning)]
    5. `crud.py` 파일을 생성 후 아래 함수를 작성합니다. [[Crud Utils](https://fastapi.tiangolo.com/tutorial/sql-databases/?h=pydantic#crud-utils)]
        - `get_user`
        - `creat_user`

---

## 구현
### database
[[Create the SQLAlchemy parts](https://fastapi.tiangolo.com/tutorial/sql-databases/?h=pydantic#create-the-sqlalchemy-parts)]을 따라서 `database.py` 파일을 작성합니다.

```python
# database.py
import os
from sqlalchemy import create_engine
from sqlalchemy.ext.declarative import declarative_base
from sqlalchemy.orm import sessionmaker


SQLALCHEMY_DATABASE_URL = os.getenv("SQLALCHEMY_DATABASE_URL", "postgresql://postgres:mypassword@localhost:5432/postgres")

engine = create_engine(SQLALCHEMY_DATABASE_URL)
SessionLocal = sessionmaker(autocommit=False, autoflush=False, bind=engine)

Base = declarative_base()
```
이 중 `SQLALCHEMY_DATABASE_URL` 값은 나중에 docker에서 값을 변경해서 전달 할 수 있도록 `os.getenv`를 통해 가져오도록 합니다.  
로컬에서 진행하는 튜토리얼에서는 기본 값으로 사용하도록 합니다.

### models
 [[Create the database model](https://fastapi.tiangolo.com/tutorial/sql-databases/?h=pydantic#create-the-database-models)]을 따라서 `models.py` 파일을 작성합니다.

tablename은 요구사항에 맞춰 `users`로 정합니다.
나머지 column들도 주어진 요구사항에 맞춰서 작성합니다.

| Column | Type | Key |
| --- | --- | --- |
| id | Integer | Primary, Index |
| name | String | Unique, Index |
| nickname | String |  |

- `id` : `Column(Integer, primary_key=True, index=True)`
- `name`: `Column(String, unique=True, index=True)`
- `nickname`: `Column(String)`


```python
# models.py
from sqlalchemy import Column, Integer, String

from database import Base


class User(Base):
    __tablename__ = "users"

    id = Column(Integer, primary_key=True, index=True)
    name = Column(String, unique=True, index=True)
    nickname = Column(String)
```

### schemas
[[create-pydantic-models-schemas-for-reading-returning](https://fastapi.tiangolo.com/tutorial/sql-databases/?h=pydantic#create-pydantic-models-schemas-for-reading-returning)]을 따라서 `schemas.py` 파일을 작성합니다.

이 전 포스트에서 작성한 pydantic model을 활용합니다.  
우선 `UseCreateIn`, `UserCreateOut` 모두에서 사용할 `UserBase`를 작성합니다.  
`UseCreateIn`, `UserCreateOut`는 작성한 `UserBase` 를 상속받습니다  
`UserCreateOut` 는 추가로 `status` 와 `id`를 입력합니다.  
마지막으로 database 에서 사용할 `User` class를 작성합니다. db에서 사용해야 하니 `orm_mode` 를 `True` 로 입력합니다.

```python
# schemas.py
from pydantic import BaseModel


class UserBase(BaseModel):
    name: str
    nickname: str


class UserCreateIn(UserBase):
    pass


class UserCreateOut(UserBase):
    status: str
    id: int


class User(UserBase):
    id: int

    class Config:
        orm_mode = True

```

### crud
[[Crud Utils](https://fastapi.tiangolo.com/tutorial/sql-databases/?h=pydantic#crud-utils)]을 따라서 `crud.py` 를 작성합니다.

```python
# crud.py
from fastapi import Depends, FastAPI

from sqlalchemy.orm import Session

from models import User, Base
from schemas import UserCreateIn, UserCreateOut
from database import SessionLocal, engine

Base.metadata.create_all(bind=engine)

app = FastAPI()

# Dependency
def get_db():
    db = SessionLocal()
    try:
        yield db
    finally:
        db.close()


@app.get("/user")
def get_user(user_id: int, db: Session = Depends(get_db)):
    return db.query(User).filter(User.id == user_id).first()


@app.post("/user", response_model=UserCreateOut)
def create_user(user: UserCreateIn, db: Session = Depends(get_db)):
    db_user = User(name=user.name, nickname=user.nickname)
    db.add(db_user)
    db.commit()
    db.refresh(db_user)
    return UserCreateOut(name=db_user.name, nickname=db_user.nickname, status="success", id=db_user.id)
```

## 실행
작성한 app을 실행하고 databse에 저장되는지 확인합니다.

우선 database 가 실행되지 않았다면 실행합니다.
```bash
docker run -p 5432:5432 -e POSTGRES_PASSWORD=mypassword -d postgres
```

다음으로 uvicorn을 이용해 app을 실행합니다.

```bash
uvicorn crud:app --reload
```

`POST /user` 을 통해 유저를 생성합니다.  
생성 후 `psql` 로 생성되었는지 확인합니다.

```bash
psql -h localhost -p 5432 -U postgres postgres
```

정상적으로 추가가 된 것을 확인할 수 있습니다.

```psql
postgres=# SELECT * FROM users;
 id |  name  | nickname
----+--------+----------
  1 | string | string
(1 row)
```
