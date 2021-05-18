---
title: seldon-core local에서 사용해보기
comment: true
categories: [seldon]
tags: ["seldon"]
toc: true
date: 2021-04-22
author: Jongseob Jeon
draft: true
---

이번 포스트에서는 seldon-core를 local에서 사용하는 법에 대해서 설명합니다.

## 1. Install
### 1.1) seldon-core
현재 seldon-core를 로컬에서 사용하기 위해서는 multiprocessing issue로 인해 3.7에서만 가능합니다.

```bash
pyenv virtualenv 3.7.9 seldon
pyenv activate seldon
pip install seldon-core scikit-learn mlflow pandas
```

### 1.2) git clone
local에서 사용하기 위해서는 git repo를 clone해야 합니다.
```bash
git clone https://github.com/SeldonIO/seldon-core
```

## 2. Sklearn server

clone받은 repo에서 sklearn server가 위치한 경로로 이동합니다.

```bash
cd seldon-core/servers/sklearnserver/sklearnserver
```

간단한 iris를 분류하는 SVC 모델을 생성할 스크립트를 작성합니다.

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

모델 파일을 읽기 위해서는 경로가 필요합니다.
현재 위치를 경로로 설정할 수 없기 때문에 모델을 저장할 경로를 만듭니다.

```bash
mkdir model
```

그리고 위에서 작성한 스크립트를 실행해 `model.joblib`을 생성합니다.

```bash
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

## 3. MLflow server

clone받은 repo에서 mlflow server가 위치한 경로로 이동합니다.

```bash
cd seldon-core/servers/mlflowserver/mlflowserver
```

mlflow를 이용해 모델을 저장할 스크립트를 작성합니다.

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

스크립트를 실행시키면 mlrus 폴더가 생성됩니다.
생성된 mlruns 폴더에서 artifact의 모델 경로를 찾습니다.

```bash
cd mlruns/0/d96ad56315364063901ace2df62dfbc2/artifacts/iris_model
```
```bash
> ls
MLmodel    conda.yaml model.pkl
```


이제 서버를 실행시켜 보겠습니다. mlflow 서버는 predict 함수만을 지원합니다.

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
