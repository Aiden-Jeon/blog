---
title: "[ Chapter 4. Model API ] 3) Docker with model"
categories: [mle-mlops]
tags: ["model-registry", "api", "docker"]
toc: true
date: 2022-09-26 19:30:00+09:00
author: Jongseob Jeon
---

[Github](https://github.com/Aiden-Jeon/mle-mlops/tree/main/04_model_api) 에서 해당 내용에 대해서 확인할 수 있습니다.

## Overview
### 목표

- 컨테이너에 모델 정보를 복사해 API를 띄우는 Dockerfile을 작성합니다.

### 요구사항

1. 작성한 API에 필요한 모델을 이미지에 복사하는 Dockerfile을 작성합니다.
    - 이 이미지를 실행하면 별도의 모델 정보를 요구하지 않습니다.
2. 빌드된 이미지를 실행합니다.
3. example을 통해 정상적으로 예측하는지 확인합니다.
4. DB에 데이터가 쌓이고 있는지 확인합니다.
---

## Crud
앞서 작성한 crud 파일에서 모델을 불러오는 부분을 다음과 같이 수정합니다.  
모델의 경로가 전역 변수로 선언되어 있지 않는 경우 `/mnt/model` 에 있는 모델을 load 하도록 합니다.

```python
MODEL_PATH = os.getenv("MODEL_PATH", "/mnt/model")
MODEL = mlflow.pyfunc.load_model(MODEL_PATH)
```


## Dockerfile

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

COPY app/ .
COPY model /mnt/model

ENTRYPOINT ["uvicorn", "crud:app", "--host", "0.0.0.0", "--reload"]
```

모델을 불러오는데 필요한 패키지들을 설치합니다.  
API를 실행하는데 필요한 코드를 복사합니다.  
그리고 모델 폴더를 `/mnt/model`에 복사합니다.

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
docker build . -t docker_with_model
```

## Run
실행 후 swagger 페이지를 확인합니다.

```bash
docker run docker_with_model
```
