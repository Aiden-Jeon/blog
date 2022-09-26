---
title: "[ Chapter 4. Model API ] 4) Docker without model"
categories: [mle-mlops]
tags: ["model-registry", "api", "docker"]
toc: true
date: 2022-09-26 19:50:00+09:00
author: Jongseob Jeon
---

[Github](https://github.com/Aiden-Jeon/mle-mlops/tree/main/04_model_api) 에서 해당 내용에 대해서 확인할 수 있습니다.

## Overview
### 목표

- 모델 경로를 받아 API를 띄우는 Dockerfile을 작성합니다.

### 요구사항

1. 작성한 API에 필요한 모델을 `/mnt/models` 경로에서 가져오도록 코드를 수정합니다.
2. 빌드된 이미지를 실행합니다.
    - 모델이 있는 경로를 `/mnt/models` 로 volume mapping 합니다.
3. example을 통해 정상적으로 예측하는지 확인합니다.
4. DB에 데이터가 쌓이고 있는지 확인합니다.
---

## Dockerfile
앞서 작성한 dockerfile 에서 모델을 copy하는 부분을 삭제합니다.  
그리고 전역 변수로 모델의 경로를 입력합니다.

```Dockerfile
FROM python:3.9-slim

WORKDIR /app

RUN apt-get update && apt-get install -y \
    build-essential \
    software-properties-common \
    git \
    && rm -rf /var/lib/apt/lists/*

RUN pip install -U pip &&\
    pip install "fastapi[all]" sqlalchemy psycopg2-binary scikit-learn mlflow

ENV MODEL_PATH="/mnt/models"
COPY app/ .

ENTRYPOINT ["uvicorn", "crud:app", "--host", "0.0.0.0", "--reload"]
```

## Build

다음과 같이 폴더를 갖고 있는 경로에서 이미지를 빌드합니다.
```bash
.

├── Dockerfile
├── app
│   ├── __init__.py
│   ├── crud.py
│   ├── database.py
│   ├── models.py
│   └── schemas.py
└── model
    ├── MLmodel
    ├── conda.yaml
    ├── model.pkl
    ├── python_env.yaml
    └── requirements.txt
```

```bash
docker build . -t docker_without_model
```

## Run
도커에 volume mapping을 한 후 실행합니다.

```bash
docker run -p $(pwd)/model:/mnt/models docker_without_model
```

실행 후 swagger 페이지를 확인합니다.
