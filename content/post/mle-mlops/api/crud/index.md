---
title: "[ Chapter 3. API ] 2) FastAPI CRUD"
categories: [mle-mlops]
tags: ["model-registry", "api", "fastapi"]
toc: true
date: 2022-09-20 16:50:00+09:00
author: Jongseob Jeon
image: swagger.png
---

[Github](https://github.com/Aiden-Jeon/mle-mlops/tree/main/03_api) 에서 해당 내용에 대해서 확인할 수 있습니다.

## Overview
### 목표

- FastAPI를 이용해 CRUD가 되는 api를 작성합니다.
- Path Parameter 와 Query Parameter의 차이점을 알아봅니다.

### 요구사항

1. 다음의 CRUD를 수행하는 fastapi의 명세서를 작성합니다.
    - Create: 이름과 별명을 입력할 수 있습니다.
    - Read: 이름을 입력하면 별명을 얻을 수 있습니다.
        - 만약 입력된 이름이 없다면 `400 code` 와 함께 `Name not found` 메세지를 반환합니다.
        - 이를 위해서 `fastapi.HTTPException` 를 사용합니다. [[FastAPI Handling Errors](https://fastapi.tiangolo.com/tutorial/handling-errors/)]
    - Update: 이름과 새로운 별명을 입력하면 이름의 별명이 업데이트 됩니다.
        - 만약 입력된 이름이 없다면 `400 code` 와 함께 `Name not found` 메세지를 반환합니다.
    - Delete: 이름을 입력하면 해당 이름과 별명을 삭제합니다.
        - 만약 입력된 이름이 없다면 `400 code` 와 함께 `Name not found` 메세지를 반환합니다.
2. 작성한 명세서를 구현합니다.
    - API에 필요한 정보들은 메모리에 저장합니다.
3. fastapi의 swagger에서 다음 시나리오를 확인합니다.
    1. `{"name": "hello2", "nickname": "world"}` 로 create 합니다.
    2. `{"name": "hello2", "nickname": "world"}` 로 update 합니다.
        - 에러가 발생하는지 확인합니다.
    3. `{"name": "hello"}` 로 read 합니다.
        - 정상적으로 `“world"` 를 반환하는 지 확인합니다.
    4. `{"name": "hello", "nickname": "world"2}` 로 update 합니다.
    5. 주어`{"name": "hello"}` 로 read 합니다.
        - 정상적으로 `“world2"` 를 반환하는 지 확인합니다.
    6. `{"name": "hello"}` 로 delete 합니다.
    7. `{"name": "hello"}` 로 read합니다.
4. Path Parameter 와 Query Parameter의 차이점은 무엇인가요?


### 명세서 예시

이름과 별명을 입력받는 명세서

- Path Parameter
    - Request Header
        - `POST /example_1`
    - Request Body
        
        ```json
        {
          "name": "hello",
          "nickname": "world"
        }
        ```
        
    - Response
        
        ```json
        {
          "status": "success"
        }
        ```
        
- Query Parameter
    - Request Header
        - `POST /example_2/name/{name}/nickname/{nickname}`
    - Request Body
        
        ```json
        {}
        ```
        
    - Response
        
        ```json
        {
          "status": "success"
        }
        ```
---

## API Specification
주어진 기능에 따른 명세서를 작성하면 다음 표들과 같습니다.

### 1. Create
이름과 별명을 입력할 수 있습니다

| Parameter | Request Head                                     | Request Body                             | Response Body           |
| --------- | ------------------------------------------------ | ---------------------------------------- | ----------------------- |
| Path      | `POST /nickname`                                 | `{"name": "hello", "nickname": "world"}` | `{"status": "success"}` |
| Query     | `POST /nickname/name/{name}/nickname/{nickname]` | `{}`                                     | `{"status": "success"}` |

### 2. Read
이름을 입력하면 별명을 얻을 수 잇습니다.

| Parameter | Request Head                | Request Body        | Response Body           |
| --------- | --------------------------- | ------------------- | ----------------------- |
| Path      | `GET /nickname`             | `{"name": "hello"}` | `{"nickname": "world"}` |
| Query     | `GET /nickname/name/{name}` | `{}`                | `{"nickname": "world"}` |


### 3. Update
- 이름과 새로운 별명을 입력하면 이름의 별명이 업데이트 됩니다.
    

| Parameter | Request Head                                    | Request Body                             | Response Body           |
| --------- | ----------------------------------------------- | ---------------------------------------- | ----------------------- |
| Path      | `PUT /nickname`                                 | `{"name": "hello", "nickname": "dlrow"}` | `{"status": "success"}` |
| Query     | `PUT /nickname/name/{name}/nickname/{nickname}` | `{}`                                     | `{"status": "success"}` |


### 4. Delete
| Parameter | Request Head                   | Request Body        | Response Body           |
| --------- | ------------------------------ | ------------------- | ----------------------- |
| Path      | `DELETE /nickname`             | `{"name": "hello"}` | `{"status": "success"}` |
| Query     | `DELETE /nickname/name/{name}` | `{}`                | `{"status": "success"}` |


## 구현
위에 적힌 명세서 내용을 구현합니다.

### Memory
우선 입력된 key:value 를 저장하기 위해서 dict 를 선업하고 app을 생성합니다.

```python
from fastapi import FastAPI, HTTPException


KEY_VALUE_STORE = {}
app = FastAPI()
```

### Response Fail
만약 없는 name에 대한 요청을 할 경우 발생할 에러인 `HTTPException` 선업합니다.  
status code는 정의된 것처럼 400을 메세지는 Name not found. 를 입력합니다.

```python
NAME_NOT_FOUND = HTTPException(status_code=400, detail="Name not found.")
```

### Create
입력받은 name과 nickname을 `KEY_VALUE_STORE` 에 저장합니다.

#### path
```python
@app.post("/nickname")
def create_path(name: str, nickname: str):
    KEY_VALUE_STORE[name] = nickname
    return {"status": "success"}
```

#### query
```python
@app.post("/nickname/name/{name}/nickname/{nickname}")
def create_query(name: str, nickname: str):
    KEY_VALUE_STORE[name] = nickname
    return {"status": "success"}
```

### Read
입력받은 name이 `KEY_VALUE_STORE`에 없다면 `HTTPException` 를 raise 합니다.

#### path
```python
@app.get("/nickname")
def get_path(name: str):
    if name not in KEY_VALUE_STORE:
        raise NAME_NOT_FOUND
    return {"nickname": KEY_VALUE_STORE[name]}
```

#### query
```python
@app.get("/nickname/name/{name}")
def get_query(name: str):
    if name not in KEY_VALUE_STORE:
        raise NAME_NOT_FOUND
    return {"nickname": KEY_VALUE_STORE[name]}
```

### Uodate
#### path
```python
@app.put("/nickname")
def update_path(name: str, nickname: str):
    if name not in KEY_VALUE_STORE:
        raise NAME_NOT_FOUND
    KEY_VALUE_STORE[name] = nickname
    return {"status": "success"}
```

#### query
```python
@app.put("/nickname/name/{name}/nickname/{nickname}")
def update_query(name: str, nickname: str):
    if name not in KEY_VALUE_STORE:
        raise NAME_NOT_FOUND
    KEY_VALUE_STORE[name] = nickname
    return {"status": "success"}
```

### Delete
#### path
```python
@app.delete("/nickname")
def delete_path(name: str):
    if name not in KEY_VALUE_STORE:
        raise NAME_NOT_FOUND
    del KEY_VALUE_STORE[name]
    return {"status": "success"}
```

#### query
```python
@app.delete("/nickname/name/{name}")
def delete_query(name: str):
    if name not in KEY_VALUE_STORE:
        raise NAME_NOT_FOUND
    del KEY_VALUE_STORE[name]
    return {"status": "success"}
```

## 실행
### 전체코드

위의 코드를 `tutorial.py` 로 작성합니다.
```python
# tutorial.py
from fastapi import FastAPI
from enum import Enum

app = FastAPI()


@app.get("/")
async def root():
    return {"message": "Hello World"}


@app.get("/items/{item_id}")
async def read_item(item_id: int):
    return {"item_id": item_id}


@app.get("/users/me")
async def read_user_me():
    return {"user_id": "the current user"}


@app.get("/users/{user_id}")
async def read_user(user_id: str):
    return {"user_id": user_id}


@app.get("/users")
async def read_users():
    return ["Rick", "Morty"]


@app.get("/users")
async def read_users2():
    return ["Bean", "Elfo"]


class ModelName(str, Enum):
    alexnet = "alexnet"
    resnet = "resnet"
    lenet = "lenet"


@app.get("/models/{model_name}")
async def get_model(model_name: ModelName):
    if model_name is ModelName.alexnet:
        return {"model_name": model_name, "message": "Deep Learning FTW!"}

    if model_name.value == "lenet":
        return {"model_name": model_name, "message": "LeCNN all the images"}

    return {"model_name": model_name, "message": "Have some residuals"}


@app.get("/files/{file_path:path}")
async def read_file(file_path: str):
    return {"file_path": file_path}
```

그리고 `uvicorn` 명령어로 실행합니다.

```bash
uvicorn run tutorial:app --reload
```

실행후 [http://127.0.0.1:8000/docs](http://127.0.0.1:8000/docs)에 접속하면 다음과 같이 작성된 swagger를 확인할 수 있습니다.

![swagger ui](swagger.png)
