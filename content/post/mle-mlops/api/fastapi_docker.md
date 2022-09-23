---
title: "[ Chapter 3. API ] 5) Build FastAPI Docker"
categories: [mle-mlops]
tags: ["model-registry", "api", "fastapi", "postgresql"]
toc: true
date: 2022-09-23 20:50:00+09:00
author: Jongseob Jeon
---

[Github](https://github.com/Aiden-Jeon/mle-mlops/tree/main/03_api) 에서 해당 내용에 대해서 확인할 수 있습니다.

## Overview
### 목표

- 앞서 작성한 스크립트를 실행할 수 있는 Dockerfile을 작성하고 이미지를 빌드합니다.

### 요구사항

1. 작성한 script를 build할 수 있는 dockerfile을 작성합니다.
2. 작성한 dockerfile을 이용해 이미지를 build 합니다.
3. 빌드된 이미지를 실행합니다.
4. api가 정상적으로 동작하는지 확인합니다.
---

## Dockerfile
작성한 app을 실행할 수 있는 `Dockerfile`을 작성합니다.

```Dockerfile
FROM python:3.9-slim

WORKDIR /app

RUN apt-get update && apt-get install -y \
    build-essential \
    software-properties-common \
    git \
    && rm -rf /var/lib/apt/lists/*

RUN pip install -U pip &&\
    pip install "fastapi[all]" sqlalchemy psycopg2-binary

COPY app/ .

ENTRYPOINT ["uvicorn", "crud:app", "--host", "0.0.0.0", "--reload"]
```

## docker-compose
db 와 app 이 함께 실행될 수 있도록 `docker-compose` 를 이용하도록 하겠습니다.  
우선 다음과 같은 `docker-compose.yaml`을 `Dockerfile`과 `app`과 같은 위치에 있도록 작성합니다.

```zsh
.
├── Dockerfile
├── app
│   ├── crud.py
│   ├── database.py
│   ├── models.py
│   └── schemas.py
└── docker-compose.yaml
```

```Dockerfile
services:
  db:
    image: postgres:latest
    ports:
      - "5432:5432"
    environment:
      POSTGRES_PASSWORD: "mypassword"
  app:
    platform: linux/amd64
    build: ./
    depends_on:
      - db
    ports:
      - "8000:8000"
    environment:
      - SQLALCHEMY_DATABASE_URL=postgresql://postgres:mypassword@db:5432/postgres

```

db 이름으로 database 관련된 정보를 작성합니다.  

app 이름으로 fastapi 관련된 정보를 작성합니다.  
이 때 build 옵션을 통해 docker 이미지를 빌드하고 실행할 수 있습니다. 같은 위치에 app과 dockerfile을 두었으므로 `./` 를 입력합니다.  
`depends_on` 옵션을 통해 app 이 db가 실행된 뒤에 실행되도록 합니다.  
`database.py` 에서 docker-compose가 띄운 db 정보를 알 수 있도록 `SQLALCHEMY_DATABASE_URL` 을 환경변수로 입력합니다.  
db의 주소는 service name을 통해 접근할 수 있습니다.
