---
title: seldon-core와 prometheus local에서 사용해보기
comment: true
categories: [seldon]
tags: ["seldon", "prometheus"]
toc: true
date: 2021-05-10
author: Jongseob Jeon
---

이번 포스트에서는 local에서 실행한 seldon-core의 metric을 prometheus로 가져오는 방법에 대해서 설명합니다.

## Install Prometheus
mac에서는 brew를 통해서 쉽게 설치할 수 있습니다.

```bash
brew install prometheus
```

그 외 다른 os 에서는 아래 링크를 참조해서 설치하면 됩니다.
- https://prometheus.io/download/


정상적으로 설치되었는지 확인합니다.

```bash
❯ prometheus --version

prometheus, version 2.26.0 (branch: non-git, revision: non-git)
  build user:       brew@BigSur
  build date:       20210331-18:20:11
  go version:       go1.16.2
  platform:         darwin/amd64
```

## Run Seldon-core
다음과 같은 `MLFlowServer.py`를 작성합니다.
```python
import yaml
import os
import logging
import requests
import random
import time
from typing import Dict, List, Union

import numpy as np
import pandas as pd
from mlflow import pyfunc
from seldon_core import Storage
from seldon_core.user_model import SeldonComponent

logger = logging.getLogger()

MLFLOW_SERVER = "model"


class MLFlowServer(SeldonComponent):
    def __init__(self, model_uri: str, xtype: str = "ndarray"):
        super().__init__()
        logger.info(f"Creating MLFLow server with URI {model_uri}")
        logger.info(f"xtype: {xtype}")
        self.model_uri = model_uri
        self.xtype = xtype
        self.ready = False
        self.bucket = 0

    def load(self):
        logger.info(f"Downloading model from {self.model_uri}")
        model_folder = Storage.download(self.model_uri)
        self._model = pyfunc.load_model(model_folder)
        self.ready = True

    def predict(
        self, X: np.ndarray, feature_names: List[str] = [], meta: Dict = None
    ) -> Union[np.ndarray, List, Dict, str, bytes]:
        logger.debug(f"Requesting prediction with: {X}")
        self.bucket = len(X)
        tic = time.time()
        time.sleep(random.choice([0, 1, 2, 3, 4]))
        if not self.ready:
            raise requests.HTTPError("Model not loaded yet")

        if self.xtype == "ndarray":
            result = self._model.predict(X)
        else:
            if feature_names is not None and len(feature_names) > 0:
                df = pd.DataFrame(data=X, columns=feature_names)
            else:
                df = pd.DataFrame(data=X)
            result = self._model.predict(df)
        toc = time.time()
        self.time = toc - tic
        logger.debug(f"Prediction result: {result}")
        return result

    def init_metadata(self):
        file_path = os.path.join(self.model_uri, "metadata.yaml")

        try:
            with open(file_path, "r") as f:
                return yaml.safe_load(f.read())
        except FileNotFoundError:
            logger.debug(f"metadata file {file_path} does not exist")
            return {}
        except yaml.YAMLError:
            logger.error(
                f"metadata file {file_path} present but does not contain valid yaml"
            )
            return {}

    def metrics(self) -> List[Dict]:
        return [
            {
                "type": "COUNTER",
                "key": "call_count",
                "value": 1,
            },  # a counter which will increase by the given value
            {
                "type": "COUNTER",
                "key": "data_count",
                "value": self.bucket,
            },  # a gauge which will be set to given value
            {
                "type": "GAUGE",
                "key": "data_gauge",
                "value": self.bucket,
            },  # a gauge which will be set to given value
            {
                "type": "TIMER",
                "key": "inference_time",
                "value": self.time,
            },  # a timer which will add sum and count metrics - assumed millisecs
        ]
```

다음 명령어를 이용해 seldon-core를 실행시킵니다.
사용되는 모델은 [이전 포스트]({{< relref "post/tech/seldon/local" >}})를 참고하면 됩니다.
```bash
seldon-core-microservice MLFlowServer --parameters '[{"name": "model_uri", "type":"STRING", "value": "file://mlruns/0/d96ad56315364063901ace2df62dfbc2/artifacts/iris_model/"}]'
```

## Prometheus
prometheus에서 사용할 config 파일 `prometheus.yaml`을 작성합니다.
이 때 target port는 6000입니다.
```yaml
global:
  scrape_interval:     15s
  evaluation_interval: 15s

rule_files:
  # - "first.rules"
  # - "second.rules"

scrape_configs:
  - job_name: prometheus
    static_configs:
      - targets: ['localhost:6000']
```

prometheus를 실행합니다.
```bash
prometheus --config.file=prometheus.yaml
```

predict 를 합니다.
```bash
curl -H "Content-Type: application/json" http://localhost:9000/predict -d @./input.json
```
다음과 같은 결과를 얻습니다.
```bash
{"data":{"names"****:[],"ndarray":[1,1]},"meta":{"metrics":[{"key":"call_count","type":"COUNTER","value":1},{"key":"data_count","type":"COUNTER","value":2},{"key":"data_gauge","type":"GAUGE","value":2},{"key":"inference_time","type":"TIMER","value":4.001783132553101}]}}
```

[http://localhost:9090](http://localhost:9090) 으로 접속하면 prometheus ui를 볼 수 있습니다.
![img-1](/imgs/seldon/prometheus-1.png)

다음과 같이 볼 metric을 정하면 결과를 볼 수 잇씁니다.
![img-2](/imgs/seldon/prometheus-2.png)

이제 predict를 몇 번 더 실행 시켜보겠습니다.
```bash
❯ curl -H "Content-Type: application/json" http://localhost:9000/predict -d @./input.json
{"data":{"names":[],"ndarray":[1,1]},"meta":{"metrics":[{"key":"call_count","type":"COUNTER","value":1},{"key":"data_count","type":"COUNTER","value":2},{"key":"data_gauge","type":"GAUGE","value":2},{"key":"inference_time","type":"TIMER","value":4.003461837768555}]}}
❯ curl -H "Content-Type: application/json" http://localhost:9000/predict -d @./input.json
{"data":{"names":[],"ndarray":[1,1]},"meta":{"metrics":[{"key":"call_count","type":"COUNTER","value":1},{"key":"data_count","type":"COUNTER","value":2},{"key":"data_gauge","type":"GAUGE","value":2},{"key":"inference_time","type":"TIMER","value":3.003340005874634}]}}
```

다음과 같이 그래프가 변하는 것을 확인할 수 있습니다.
![img-3](/imgs/seldon/prometheus-3.png)
