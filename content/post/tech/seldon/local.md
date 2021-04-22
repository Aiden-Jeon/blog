---
title: seldon-core local에서 사용해보기
comment: true
categories: [kubernetes]
tags: ["seldon"]
toc: true
date: 2021-04-22
author: Jongseob Jeon
---

이번 포스트에서는 seldon core를 local에서 사용하는 법에 대해서 설명합니다.

## Install selon-core

현재 seldon-core를 로컬에서 사용하기 위해서는 multiprocessing issue로 인해 3.7에서만 가능합니다.

```bash
pyenv virtualenv 3.7.9 seldon
pyenv activate seldon
pip install seldon-core scikit-learn mlflow pandas
```

seldon-core repo를 clone 합니다.

```bash
git clone https://github.com/SeldonIO/seldon-core
```

## Sklearn server

sklearn server 를 사용할 경로 위치로 이동합니다.

```bash
cd seldon-core/servers/sklearnserver/sklearnserver
```

모델을 저장할 스크립트를 작성합니다.

```python
# sklearn_model.py
import joblib
from sklearn import datasets
from sklearn import svm

def train():
    df = datasets.load_iris()
    X, y = df.data, df.target

    clf = svm.SVC()
    clf.fit(X, y)

    joblib.dump(clf, "model/model.joblib")

if __name__ == "__main__":
    train()
```

모델을 저장할 경로를 생성한 후 스크립트를 실행해 `model.joblib`을 생성합니다.

```bash
mkdir model
python sklearn_model.py
```

이제 서버를 실행시켜 보겠습니다. SVC 모델은 predict 함수만 있기 때문에 parameter에서 predict method를 주겠습니다.

```bash
seldon-core-microservice SKLearnServer --parameters '[{"name": "method", "type": "STRING", "value": "predict"}, {"name": "model_uri", "type":"STRING", "value": "file://model/"}]'
```

이제 predict를 해보겠습니다.

```bash
curl -H "Content-Type: application/json" http://localhost:9000/predict -d '{
    "data":{
        "ndarray": [[0.6745011455,-0.5923730118, 1.0469454037, 1.1855672065]],
        "names": ["sepal length (cm)", "sepal width (cm)","petal length (cm)","petal width (cm)"]
    },
    "meta" : {}
}'
```

## MLflow server

mlflow server 가 위치한 경로로 이동합니다.

```bash
cd seldon-core/servers/sklearnserver/sklearnserver
```

모델을 저장할 스크립트를 작성합니다.

```python
# mlflow_model.py
import mlflow
from mlflow.models.signature import infer_signature
from sklearn import datasets
from sklearn import svm

def train():
    df = datasets.load_iris()
    X, y = df.data, df.target

    clf = svm.SVC()
    clf.fit(X, y)
    signature = infer_signature(X, clf.predict(X))
    mlflow.sklearn.log_model(clf, "iris_model", signature=signature)

if __name__ == "__main__":
    train()
```

생성된 mlruns 폴더에서 artifact의 모델 경로를 찾습니다.

```bash
> cd mlruns/0/d96ad56315364063901ace2df62dfbc2/artifacts/iris_model
> ls
MLmodel    conda.yaml model.pkl
```

이제 서버를 실행시켜 보겠습니다. SVC 모델은 predict 함수만 있기 때문에 parameter에서 predict method를 주겠습니다.

```bash
seldon-core-microservice MLFlowServer --parameters '[{"name": "model_uri", "type":"STRING", "value": "file://mlruns/0/d96ad56315364063901ace2df62dfbc2/artifacts/iris_model/"}]'
```

이제 predict를 해보겠습니다.

```bash
curl -H "Content-Type: application/json" http://localhost:9000/predict -d '{
    "data":{
        "ndarray": [[0.6745011455,-0.5923730118, 1.0469454037, 1.1855672065]],
        "names": ["sepal length (cm)", "sepal width (cm)","petal length (cm)","petal width (cm)"]
    },
    "meta" : {}
}'
```
