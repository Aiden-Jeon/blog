---
title: Model Registry-1) Run Mlflow Server
categories: [mle-mlops]
tags: ["mlflow", "model-registry"]
toc: true
date: 2022-09-19 17:50:00+09:00
author: Jongseob Jeon
image: mlflow-ui.png
---

[Github](https://github.com/Aiden-Jeon/mle-mlops/tree/main/02_model_registry) 에서 해당 내용에 대해서 확인할 수 있습니다.

## Overview
### 목표

- 로컬에서 MLFLow Server를 띄어봅시다.

### 요구사항

1. 로컬에 mlflow server를 띄웁니다.
    - `mlflow server` 를 사용해 서버를 띄웁니다.
    - mlflow server는 "3.모델 학습 및 저장" 와 작성할 스크립트와 같은 디렉토리에서 실행되어야 합니다.
2. mlflow ui website를 확인합니다.
---

## Script

### Install
mlflow 를 설치합니다.

```bash
pip install mlflow
```

### Run
다음 명령어로 mlflow를 실행합니다.

```bash
mlflow server
```

실행되면 다음과 같이 출력됩니다.

```zsh
> mlflow server
[2022-09-20 11:55:20 +0900] [23908] [INFO] Starting gunicorn 20.1.0
[2022-09-20 11:55:20 +0900] [23908] [INFO] Listening at: http://127.0.0.1:5000 (23908)
[2022-09-20 11:55:20 +0900] [23908] [INFO] Using worker: sync
[2022-09-20 11:55:20 +0900] [23909] [INFO] Booting worker with pid: 23909
[2022-09-20 11:55:21 +0900] [23910] [INFO] Booting worker with pid: 23910
[2022-09-20 11:55:21 +0900] [23911] [INFO] Booting worker with pid: 23911
[2022-09-20 11:55:21 +0900] [23912] [INFO] Booting worker with pid: 23912
```

실행된 후에는 디렉토리에 `mlruns` 폴더가 생성됩니다.

```zsh
.
└── mlruns
    └── 0
        └── meta.yaml
```

## Web

[http://localhost:5000](http://localhost:5000) 에 접속합니다.

![mlflow ui](mlflow-ui.png)
