---
title: 밑바닥부터 시작하는 Seldon-Core - Custom Server
categories: [mlops]
tags: ["k8s", "seldon"]
toc: true
date: 2021-06-28
lastmod: 2021-10-01
author: Jongseob Jeon
---


# Pre-requisites
이번 포스트에서는 작성하는 Custom Server를 사용하기 위해서는 아래와 같은 Secret이 필요합니다.
1. GHCR secrets
2. aws secrets

## 1. GHCR Secrets
작성한 Custom Server의 Docker는 GHCR에서 관리합니다.  
k8s에서 image를 pull 하기 위해서는 ghcr에 접근할 수 있는 권한이 필요합니다.
```bash
kubectl -n seldon create secret docker-registry mrx-ghcr \\
   --docker-server=ghcr.io \\
   --docker-username=<github id> \\
   --docker-password=<secret key> \\
   --docker-email=<github mail>
```

## 2. aws secrets
이번 포스트에서는 모델을 S3에 저장한다고 가정합니다.  
k8s에서 S3에 있는 모델을 가져오기 위한 aws에 접근할 수 있는 권한이 필요합니다.  
이를 위해서 아래와 같은 ``aws-secret.yaml``을 작성합니다.
```yaml
apiVersion: v1
kind: Secret
metadata:
 name: seldon-init-container-secret
 namespace: seldon
type: Opaque
data:
 AWS_ACCESS_KEY_ID: <base64 | key_id>
 AWS_SECRET_ACCESS_KEY: <base64 | access_key>
 AWS_ENDPOINT_URL: <base64 | endpoint>
 USE_SSL: <base64 | true>
```

여기서 주의해야할 점은 data에 value 값은 모두 base64 인코딩 되어 있어야 합니다.  
이는 bash에서 `base64` command로 쉽게 변환할 수 있습니다.
```bash
echo 'hello, world!' | base64
```
변환이 되면 다음과 같이 출력됩니다.
```
aGVsbG8sIHdvcmxkIQo=
```

apply로 secret을 생성합니다.
```bash
kubectl apply -f aws-secret.yaml
```

# Custom Server
Seldon-core에서 제공하는 pre-packaged inference server 다음과 같습니다.
- SKLearn Server
- MLflow Server
- Triton Inference Server
- Tensorflow Serving
- XGBoost Server

대부분의 경우는 제공하는 서버를 사용하면 됩니다. 하지만 필요한 경우 Custom 서버를 직접 만들어서 사용할 수 도 있습니다.  
이번 포스트에서는 MLFlowServer를 기본으로 하지만 Conda 환경이 아닌 Pip 환경을 이용하는 방법에 대해서 설명합니다.

# Model
## Docker
### Content

seldon-core 에서 제공하는 서버들의 디렉토리 구성은 다음과 같습니다.

```bash
mlflowserver
├── MLFlowServer.py
├── before-run
├── conda_env_create.py
├── image_metadata.json
└── requirements.txt
```

파일 중 `before-run` 과 `conda_env_create.py` 가 inference server의 환경을 구성하는 파일 입니다.

`conda_env_create.py`  에서 conda 환경을 구성하는 부분을 빼고 pip 설치만 하도록 수정 후 `pip_env_create.py` 를 생성하겠습니다.

1. `pip_env_create.py`
    ```python
    import argparse
    import json
    import logging
    import os
    import shutil
    import tempfile
    from typing import Any
    
    import yaml
    from pip._internal.operations import freeze
    from seldon_core import Storage
    from seldon_core.microservice import PARAMETERS_ENV_NAME, parse_parameters
    
    log = logging.getLogger()
    log.setLevel("INFO")
    
    parser = argparse.ArgumentParser()
    parser.add_argument(
        "--parameters",
        type=str,
        default=os.environ.get(PARAMETERS_ENV_NAME, "[]"),
    )
  
    # This is already set on the environment_rest and environment_grpc files, but
    # we'll define a default just in case.
    DEFAULT_CONDA_ENV_NAME = "mlflow"
    BASE_REQS_PATH = os.path.join(
        os.path.dirname(os.path.abspath(__file__)),
        "requirements.txt",
    )
    
    def setup_env(model_folder: str) -> None:
        """Sets up a pip environment.
        This methods creates the pip environment described by the `MLmodel` file.
        Parameters
        --------
        model_folder : str
            Folder where the MLmodel files are stored.
        """
        mlmodel = read_mlmodel(model_folder)
    
        flavours = mlmodel["flavors"]
        pyfunc_flavour = flavours["python_function"]
        env_file_name = pyfunc_flavour["env"]
        env_file_path = os.path.join(model_folder, env_file_name)
        env_file_path = copy_env(env_file_path)
        copy_pip(env_file_path)
        install_base_reqs()
    
    def read_mlmodel(model_folder: str) -> Any:
        """Reads an MLmodel file.
        Parameters
        ---------
        model_folder : str
            Folder where the MLmodel files are stored.
        Returns
        --------
        obj
            Dictionary with MLmodel contents.
        """
        log.info("Reading MLmodel file")
        mlmodel_path = os.path.join(model_folder, "MLmodel")
        return _read_yaml(mlmodel_path)
    
    def _read_yaml(file_path: str) -> Any:
        """Reads a YAML file.
        Parameters
        ---------
        file_path
            Path to the YAML file.
        Returns
        -------
        dict
            Dictionary with YAML file contents.
        """
        with open(file_path, "r") as file_reader:
            return yaml.safe_load(file_reader)
    
    def copy_env(env_file_path: str) -> str:
        """Copy conda.yaml to temp dir
        to prevent the case where the existing file is on Read-only file system.
        Parameters
        ----------
        env_file_path : str
            env file path to copy.
        Returns
        -------
        str
            tmp file directory.
        """
        temp_dir = tempfile.mkdtemp()
        new_env_path = os.path.join(temp_dir, "conda.yaml")
        shutil.copy2(env_file_path, new_env_path)
    
        return new_env_path
    
    def copy_pip(new_env_path: str) -> None:
        """Copy pip packages from conda.yaml to requirements.txt
        Parameters
        ----------
        new_env_path : str
            requirements.txt path.
        """
        conda = _read_yaml(new_env_path)
        pip_packages = conda["dependencies"][-1]["pip"]
        freezed = freeze.freeze()
        freezed = map(lambda x: x.split("==")[0], freezed)
        package_to_install = []
        for package in pip_packages:
            name = package.split("==")[0]
            if name not in freezed:
                package_to_install += [package]
    
        with open(BASE_REQS_PATH, "a") as file_writer:
            file_writer.write("\\n".join(package_to_install))
    
    def install_base_reqs() -> None:
        """Install additional requirements from requirements.txt.
        If the variable is not defined, it falls back to `mlflow`.
        """
        log.info("Install additional package from requirements.txt")
        cmd = f"pip install -r {BASE_REQS_PATH}"
        os.system(cmd)
    
    def main(arguments: argparse.Namespace) -> None:
        """main algorithm.
        Parameters
        ----------
        arguments : argparse.Namespace
        """
        parameters = parse_parameters(json.loads(arguments.parameters))
        model_uri = parameters.get("model_uri", "/mnt/model/")
    
        log.info("Downloading model from %s", model_uri)
        model_folder = Storage.download(model_uri)
        setup_env(model_folder)
    
    if __name__ == "__main__":
        args = parser.parse_args()
        main(args)
    ```

2. `before-run`
    ```bash
    #!/bin/bash -e
    
    echo "---> Creating environment with pip..."
    python ./pip_env_create.py
    ```

3. `MLFlowServer.py`
    ```python
    import logging
    import os
    from typing import Dict, List, Union
    
    import numpy as np
    import pandas as pd
    import requests
    import yaml
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
    
        def load(self):
            logger.info(f"Downloading model from {self.model_uri}")
            model_folder = Storage.download(self.model_uri)
            self._model = pyfunc.load_model(model_folder)
            self.ready = True
    
        def predict(
            self, X: np.ndarray, feature_names: List[str] = [], meta: Dict = None
        ) -> Union[np.ndarray, List, Dict, str, bytes]:
            logger.debug(f"Requesting prediction with: {X}")
    
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
    
            def class_names(self) -> Iterable[str]:
            output_schema = self._model.metadata.get_output_schema()
            if output_schema is not None:
                columns = [schema["name"] for schema in output_schema.to_dict()]
            return columns
    ```

4. `image_metadata.json`
    ```json
    {"labels": [{"name": "Seldon MLFlow Server"}, {"vendor": "Seldon Technologies"}, {"version": "1.7.0"}, {"release": "1"}, {"summary": "An MLFlow Model Server for Seldon Core"}, {"description": "The model server for MLFlow models"}]}
    ```

5. `requirements.txt`
   빈 파일을 생성합니다.
    ```
    ```


위와 같이 수정하면 디렉토리 구성은 다음과 같이 됩니다.
```bash
mlflowserver
├── Dockerfile
└── mrxflowserver
    ├── MLFlowServer.py
    ├── before-run
    ├── image_metadata.json
    ├── pip_env_create.py
    └── requirements.txt
```


### Dockerfile

`Dockerfile` 을 작성합니다.

```docker
ARG BASE_CONTAINER=ghcr.io/<github-owner>/python:3.8-minimal
FROM $BASE_CONTAINER

RUN pip3 install seldon_core

ENV LANG C.UTF-8
COPY mlflowserver/ /app/

RUN chmod +r /app/MLFlowServer.py
RUN chmod +x /app/before-run

WORKDIR /app
EXPOSE 5000
EXPOSE 9000

ENV MODEL_NAME MLFlowServer
ENV SERVICE_TYPE MODEL
ENV PERSISTENCE 0

CMD bash before-run &&\\
    exec seldon-core-microservice $MODEL_NAME --service-type $SERVICE_TYPE --persistence $PERSISTENCE
```


Docker를 build하고 push 하겠습니다.
```docker
docker build . -t ghcr.io/<github-owner>/serving-tutorial:latest
docker push ghcr.io/<github-owner>/serving-tutorial:latest
```

## Seldon-deploy
다음과 같은 `model.yaml` deploy 파일을 작성합니다.

- container의 image가 build 될 때  `requirements.txt` 에 접근 권한이 있어야 하므로 `securityContext` 에 `privileged` , `runsAsUser`, `runAsGroup` 을 작성합니다.

- `model.yaml`
    ```yaml
    apiVersion: machinelearning.seldon.io/v1
    kind: SeldonDeployment
    metadata:
        name: graph-iris-model
        namespace: seldon
    spec:
        name: model
        predictors:
        - name: model
        componentSpecs:
        - spec:
            volumes:
            - name: model-provision-location
                emptyDir: {}
            containers:
            - image: ghcr.io/<github-owner>/serving-tutorial:latest
                name: model
                imagePullPolicy: Always
                securityContext:
                privileged: true
                runAsUser: 0
                runAsGroup: 0
                volumeMounts:
                - mountPath: /mnt/models
                name: model-provision-location
                readOnly: true
            imagePullSecrets:
            - name: mrx-ghcr
            initContainers:
            - name: model-initializer
                image: gcr.io/kfserving/storage-initializer:v0.4.0
                imagePullPolicy: IfNotPresent
                args:
                -   s3://mlflow/mlflow/artifacts/17/b84160df245441fa8d5ad7c5b62a424d/artifacts/iris_svm_model/
                - /mnt/models
                envFrom:
                - secretRef:
                    name: seldon-init-container-secret
                volumeMounts:
                - mountPath: /mnt/models
                name: model-provision-location
        graph:
            name: model
            type: MODEL
            children: []
            parameters:
            - name: xtype
            type: STRING
            value: DataFrame
            - name: model_uri
            type: STRING
            value: /mnt/models
            - name: predict_method
            type: STRING
            value: decision_function
        replicas: 1
    ```

- apply 합니다.
    ```bash
    kubectl apply -f model.yaml
    ```

- predict를 요청합니다.
    ```bash
    curl -X POST <http://10.102.231.216/seldon/seldon/graph-iris-model/api/v1.0/predictions> \\
        -H 'Content-Type: application/json' \\
        -d '{
                "data": {
                "ndarray": [
                    [6.8,  2.8,  4.8,  1.4],
                    [6.0,  3.4,  4.5,  1.6]
                ],
                "names":["sepal length (cm)", "sepal width (cm)", "petal length (cm)", "petal width (cm)"]
                }
            }'
    ```

- 다음과 같은 결과를 받을 수 있습니다.
    ```bash
    {
        "data": {
        "names": [
            "Class_0",
            "Class_1",
            "Class_2"
        ],
        "ndarray": [
            [
            0.9663949459143529,
            1.0137844346775422,
            1.0215099592994648
            ],
            [
            0.9660782736674745,
            1.0133498258843965,
            1.0222645232705032
            ]
        ]
        },
        "meta": {
        "requestPath": {
            "model": "ghcr.io/<github-owner>/serving-tutorial:latest"
        }
        }
    }
    ```

# Preprocessor

이번에는 데이터 전처리를 할 수 있는 preprocessor를 작성해보겠습니다.

## Docker

앞서 사용했던 파일에서 `MLFLowServer.py` 를 수정하겠습니다. sklearn의 StandardScaler는 predict 함수를 갖고 있지 않습니다. 그래서 pyfunc을 이용해서 load할 수 없습니다. 이를 위해서 pyfunc을 새로 생성하고 transform을 할 수 있는 `transform_input` 를 작성합니다.

1. `pyfunc.py`

    ```python
    import importlib
    import os
    from typing import Any, Dict, List, Union
    
    import mlflow
    import numpy as np
    import pandas
    import yaml
    from mlflow.exceptions import MlflowException
    from mlflow.models import Model
    from mlflow.models.model import MLMODEL_FILE_NAME
    from mlflow.protos.databricks_pb2 import RESOURCE_DOES_NOT_EXIST
    from mlflow.pyfunc import (
        _enforce_schema,
        _warn_potentially_incompatible_py_version_if_necessary,
    )
    from mlflow.tracking.artifact_utils import _download_artifact_from_uri
    
    FLAVOR_NAME = "python_function"
    MAIN = "loader_module"
    CODE = "code"
    DATA = "data"
    ENV = "env"
    PY_VERSION = "python_version"
    
    PyFuncInput = Union[pandas.DataFrame, np.ndarray, List[Any], Dict[str, Any]]
    PyFuncOutput = Union[pandas.DataFrame, pandas.Series, np.ndarray, list]
    
    class PyFuncModel:
        """
        MLflow 'python function' model.
        Wrapper around model implementation and metadata. This class is not meant to be constructed
        directly. Instead, instances of this class are constructed and returned from
        :py:func:`load_model() <mlflow.pyfunc.load_model>`.
        ``model_impl`` can be any Python object that implements the `Pyfunc interface
        <https://mlflow.org/docs/latest/python_api/mlflow.pyfunc.html#pyfunc-inference-api>`_, and is
        returned by invoking the model's ``loader_module``.
        ``model_meta`` contains model metadata loaded from the MLmodel file.
        """
    
        # Customized from mlflow==1.16.0
        def __init__(self, model_meta: Model, model_impl: Any):
            if not model_meta:
                raise MlflowException("Model is missing metadata.")
            self._model_meta = model_meta
            self._model_impl = model_impl
    
        def predict(self, data: PyFuncInput) -> PyFuncOutput:
            """
            Generate model predictions.
            If the model contains signature, enforce the input schema first before calling the model
            implementation with the sanitized input. If the pyfunc model does not include model schema,
            the input is passed to the model implementation as is. See `Model Signature Enforcement
            <https://www.mlflow.org/docs/latest/models.html#signature-enforcement>`_ for more details."
            :param data: Model input as one of pandas.DataFrame, numpy.ndarray, or
                        Dict[str, numpy.ndarray]
            :return: Model predictions as one of pandas.DataFrame, pandas.Series, numpy.ndarray or list.
            """
            input_schema = self.metadata.get_input_schema()
            if input_schema is not None:
                data = _enforce_schema(data, input_schema)
            return self._model_impl.predict(data)
    
        # Customized from mlflow==1.16.0
        def transform(self, data: PyFuncInput) -> PyFuncOutput:
            """transform method"""
            input_schema = self.metadata.get_input_schema()
            if input_schema is not None:
                data = _enforce_schema(data, input_schema)
            return self._model_impl.transform(data)
    
        @property
        def metadata(self) -> Model:
            """Model metadata."""
            if self._model_meta is None:
                raise MlflowException("Model is missing metadata.")
            return self._model_meta
    
        def __repr__(self) -> Any:
            info = {}
            if self._model_meta is not None:
                if (
                    hasattr(self._model_meta, "run_id")
                    and self._model_meta.run_id is not None
                ):
                    info["run_id"] = self._model_meta.run_id
                if (
                    hasattr(self._model_meta, "artifact_path")
                    and self._model_meta.artifact_path is not None
                ):
                    info["artifact_path"] = self._model_meta.artifact_path
    
                info["flavor"] = self._model_meta.flavors[FLAVOR_NAME]["loader_module"]
    
            return yaml.safe_dump(
                {"mlflow.pyfunc.loaded_model": info},
                default_flow_style=False,
            )
    
    def load_model(model_uri: str, suppress_warnings: bool = True) -> PyFuncModel:
        local_path = _download_artifact_from_uri(artifact_uri=model_uri)
        model_meta = Model.load(os.path.join(local_path, MLMODEL_FILE_NAME))
    
        conf = model_meta.flavors.get(FLAVOR_NAME)
        if conf is None:
            raise MlflowException(
                f'Model does not have the "{FLAVOR_NAME}" flavor',
                RESOURCE_DOES_NOT_EXIST,
            )
        model_py_version = conf.get(PY_VERSION)
        if not suppress_warnings:
            _warn_potentially_incompatible_py_version_if_necessary(
                model_py_version=model_py_version,
            )
        if CODE in conf and conf[CODE]:
            code_path = os.path.join(local_path, conf[CODE])
            mlflow.pyfunc.utils._add_code_to_system_path(code_path=code_path)
        data_path = os.path.join(local_path, conf[DATA]) if (DATA in conf) else local_path
        model_impl = importlib.import_module(conf[MAIN])._load_pyfunc(data_path)  # type: ignore
        return PyFuncModel(model_meta=model_meta, model_impl=model_impl)
    ```

2. `MLFlowServer.py`
     - `transform_input` 구현

     ```python
     import logging
     import os
     from typing import Dict, List, Union
     
     import numpy as np
     import pandas as pd
     import requests
     import yaml
     from seldon_core import Storage
     from seldon_core.user_model import SeldonComponent
     
     from pyfunc import load_model
     
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
     
         def load(self):
             logger.info(f"Downloading model from {self.model_uri}")
             model_folder = Storage.download(self.model_uri)
             self._model = load_model(model_folder)
             self.ready = True
     
         def predict(
             self, X: np.ndarray, feature_names: List[str] = [], meta: Dict = None
         ) -> Union[np.ndarray, List, Dict, str, bytes]:
             logger.debug(f"Requesting prediction with: {X}")
     
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
     
             logger.debug(f"Prediction result: {result}")
             return result
     
         def transform_input(
             self, X: np.ndarray, feature_names: List[str] = [], meta: Dict = None
         ) -> Union[np.ndarray, List, Dict, str, bytes]:
             logger.info(f"Requesting transformation with: {X}")
             if not self.ready:
                 raise requests.HTTPError("Model not loaded yet")
     
             if self.xtype == "ndarray":
                 result = self._model.transform(X)
             else:
                 if feature_names is not None and len(feature_names) > 0:
                     df = pd.DataFrame(data=X, columns=feature_names)
                 else:
                     df = pd.DataFrame(data=X)
                 result = self._model.transform(df)
     
             logger.debug(f"transformation result: {result}")
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
     
     		def class_names(self) -> Iterable[str]:
             output_schema = self._model.metadata.get_output_schema()
             if output_schema is not None:
                 columns = [schema["name"] for schema in output_schema.to_dict()]
             return columns
     ```

위와 같이 수정하면 디렉토리 구성은 다음과 같이 됩니다.

```bash
mlflowserver
├── Dockerfile
└── mrxflowserver
    ├── MLFlowServer.py
    ├── before-run
    ├── image_metadata.json
    ├── pip_env_create.py
    ├── pyfunc.py
    └── requirements.txt
```

## Seldon-deploy
`preprocessor.yaml` deploy 파일을 작성합니다.

- `preprocessor.yaml`
    ```yaml
    apiVersion: machinelearning.seldon.io/v1
    kind: SeldonDeployment
    metadata:
        name: graph-iris-preprocessor
        namespace: seldon
    spec:
        name: preprocessor
        predictors:
        - name: preprocessor
        componentSpecs:
        - spec:
            volumes:
            - name: model-provision-location
                emptyDir: {}
            containers:
            - image: ghcr.io/<github-owner>/serving-tutorial:latest
                name: preprocessor
                imagePullPolicy: Always
                securityContext:
                privileged: true
                runAsUser: 0
                runAsGroup: 0
                volumeMounts:
                - mountPath: /mnt/models
                name: model-provision-location
                readOnly: true
            imagePullSecrets:
            - name: mrx-ghcr
            initContainers:
            - name: preprocessor-initializer
                image: gcr.io/kfserving/storage-initializer:v0.4.0
                imagePullPolicy: IfNotPresent
                args:
                - s3://mlflow/mlflow/artifacts/17/b84160df245441fa8d5ad7c5b62a424d/artifacts/iris_preprocessor_model/
                - /mnt/models
                envFrom:
                - secretRef:
                    name: seldon-init-container-secret
                volumeMounts:
                - mountPath: /mnt/models
                name: model-provision-location
        graph:
            name: preprocessor
            type: TRANSFORMER
            children: []
            parameters:
            - name: xtype
            type: STRING
            value: DataFrame
            - name: model_uri
            type: STRING
            value: /mnt/models
        replicas: 1
    ```

- apply 합니다.
    ```bash
    kubectl apply -f preprocessor.yaml
    ```

- predict를 요청합니다.
    ```bash
    curl -X POST <http://10.102.231.216/seldon/seldon/graph-iris-preprocessor/api/v1.0/predictions> \\
        -H 'Content-Type: application/json' \\
        -d '{
                "data": {
                "ndarray": [
                    [6.8,  2.8,  4.8,  1.4],
                    [6.0,  3.4,  4.5,  1.6]
                ],
                "names":["sepal length (cm)", "sepal width (cm)", "petal length (cm)", "petal width (cm)"]
                }
            }'
    ```

- 다음과 같은 결과를 받을 수 있습니다.
    ```bash
    {
        "data": {
        "names": [
            "sepal length (cm)",
            "sepal width (cm)",
            "petal length (cm)",
            "petal width (cm)"
        ],
        "ndarray": [
            [
            1.1591726272442633,
            -0.5923730118389178,
            0.5922459884397444,
            0.26414191647586993
            ],
            [
            0.18982966369505322,
            0.7888075857129604,
            0.4217337076989351,
            0.5274062850564719
            ]
        ]
        },
        "meta": {
        "requestPath": {
            "preprocessor": "ghcr.io/<github-owner>/serving-tutorial:latest"
        }
        }
    }
    ```

# Postprocessor

이번에는 score에 후처리를 하는 postprocessor를  작성해보겠습니다.

## Docker

seldon-core 에서 제공하는 서버들의 디렉토리 구성은 다음과 같습니다.

```bash
mlflowserver
├── MLFlowServer.py
├── before-run
├── conda_env_create.py
├── image_metadata.json
└── requirements.txt
```

앞서 사용했던 파일에서 `MLFLowServer.py` 를 수정하겠습니다. 이 예제에서는 postprocessor 또한 sklearn의 scaler를 사용하겠습니다. sklearn의 scaler는 predict 함수를 갖고 있지 않습니다. 그래서 pyfunc을 이용해서 load할 수 없습니다. 이를 위해서 pyfunc을 새로 생성하고 transform을 할 수 있는 `transform_output` 를 작성합니다.

1. `pyfunc.py`
    ```python
    import importlib
    import os
    from typing import Any, Dict, List, Union
    
    import mlflow
    import numpy as np
    import pandas
    import yaml
    from mlflow.exceptions import MlflowException
    from mlflow.models import Model
    from mlflow.models.model import MLMODEL_FILE_NAME
    from mlflow.protos.databricks_pb2 import RESOURCE_DOES_NOT_EXIST
    from mlflow.pyfunc import (
        _enforce_schema,
        _warn_potentially_incompatible_py_version_if_necessary,
    )
    from mlflow.tracking.artifact_utils import _download_artifact_from_uri
    
    FLAVOR_NAME = "python_function"
    MAIN = "loader_module"
    CODE = "code"
    DATA = "data"
    ENV = "env"
    PY_VERSION = "python_version"
    
    PyFuncInput = Union[pandas.DataFrame, np.ndarray, List[Any], Dict[str, Any]]
    PyFuncOutput = Union[pandas.DataFrame, pandas.Series, np.ndarray, list]
    
    class PyFuncModel:
        """
        MLflow 'python function' model.
        Wrapper around model implementation and metadata. This class is not meant to be constructed
        directly. Instead, instances of this class are constructed and returned from
        :py:func:`load_model() <mlflow.pyfunc.load_model>`.
        ``model_impl`` can be any Python object that implements the `Pyfunc interface
        <https://mlflow.org/docs/latest/python_api/mlflow.pyfunc.html#pyfunc-inference-api>`_, and is
        returned by invoking the model's ``loader_module``.
        ``model_meta`` contains model metadata loaded from the MLmodel file.
        """
    
        # Customized from mlflow==1.16.0
        def __init__(self, model_meta: Model, model_impl: Any):
            if not model_meta:
                raise MlflowException("Model is missing metadata.")
            self._model_meta = model_meta
            self._model_impl = model_impl
    
        def predict(self, data: PyFuncInput) -> PyFuncOutput:
            """
            Generate model predictions.
            If the model contains signature, enforce the input schema first before calling the model
            implementation with the sanitized input. If the pyfunc model does not include model schema,
            the input is passed to the model implementation as is. See `Model Signature Enforcement
            <https://www.mlflow.org/docs/latest/models.html#signature-enforcement>`_ for more details."
            :param data: Model input as one of pandas.DataFrame, numpy.ndarray, or
                        Dict[str, numpy.ndarray]
            :return: Model predictions as one of pandas.DataFrame, pandas.Series, numpy.ndarray or list.
            """
            input_schema = self.metadata.get_input_schema()
            if input_schema is not None:
                data = _enforce_schema(data, input_schema)
            return self._model_impl.predict(data)
    
        # Customized from mlflow==1.16.0
        def transform(self, data: PyFuncInput) -> PyFuncOutput:
            """transform method"""
            input_schema = self.metadata.get_input_schema()
            if input_schema is not None:
                data = _enforce_schema(data, input_schema)
            return self._model_impl.transform(data)
    
        @property
        def metadata(self) -> Model:
            """Model metadata."""
            if self._model_meta is None:
                raise MlflowException("Model is missing metadata.")
            return self._model_meta
    
        def __repr__(self) -> Any:
            info = {}
            if self._model_meta is not None:
                if (
                    hasattr(self._model_meta, "run_id")
                    and self._model_meta.run_id is not None
                ):
                    info["run_id"] = self._model_meta.run_id
                if (
                    hasattr(self._model_meta, "artifact_path")
                    and self._model_meta.artifact_path is not None
                ):
                    info["artifact_path"] = self._model_meta.artifact_path
    
                info["flavor"] = self._model_meta.flavors[FLAVOR_NAME]["loader_module"]
    
            return yaml.safe_dump(
                {"mlflow.pyfunc.loaded_model": info},
                default_flow_style=False,
            )
    
    def load_model(model_uri: str, suppress_warnings: bool = True) -> PyFuncModel:
        local_path = _download_artifact_from_uri(artifact_uri=model_uri)
        model_meta = Model.load(os.path.join(local_path, MLMODEL_FILE_NAME))
    
        conf = model_meta.flavors.get(FLAVOR_NAME)
        if conf is None:
            raise MlflowException(
                f'Model does not have the "{FLAVOR_NAME}" flavor',
                RESOURCE_DOES_NOT_EXIST,
            )
        model_py_version = conf.get(PY_VERSION)
        if not suppress_warnings:
            _warn_potentially_incompatible_py_version_if_necessary(
                model_py_version=model_py_version,
            )
        if CODE in conf and conf[CODE]:
            code_path = os.path.join(local_path, conf[CODE])
            mlflow.pyfunc.utils._add_code_to_system_path(code_path=code_path)
        data_path = os.path.join(local_path, conf[DATA]) if (DATA in conf) else local_path
        model_impl = importlib.import_module(conf[MAIN])._load_pyfunc(data_path)  # type: ignore
        return PyFuncModel(model_meta=model_meta, model_impl=model_impl)
    ```

2. `MLFlowServer.py`
     - `transform_output` 구현

    ```python
    import logging
    import os
    from typing import Dict, List, Union
    
    import numpy as np
    import pandas as pd
    import requests
    import yaml
    from seldon_core import Storage
    from seldon_core.user_model import SeldonComponent
    
    from pyfunc import load_model
    
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
    
        def load(self):
            logger.info(f"Downloading model from {self.model_uri}")
            model_folder = Storage.download(self.model_uri)
            self._model = load_model(model_folder)
            self.ready = True
    
        def predict(
            self, X: np.ndarray, feature_names: List[str] = [], meta: Dict = None
        ) -> Union[np.ndarray, List, Dict, str, bytes]:
            logger.debug(f"Requesting prediction with: {X}")
    
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
    
            logger.debug(f"Prediction result: {result}")
            return result
    
        def transform_output(
            self, X: np.ndarray, feature_names: List[str] = [], meta: Dict = None
        ) -> Union[np.ndarray, List, Dict, str, bytes]:
            logger.info(f"Requesting transformation with: {X}")
            if not self.ready:
                raise requests.HTTPError("Model not loaded yet")
    
            if self.xtype == "ndarray":
                result = self._model.transform(X)
            else:
                if feature_names is not None and len(feature_names) > 0:
                    df = pd.DataFrame(data=X, columns=feature_names)
                else:
                    df = pd.DataFrame(data=X)
                result = self._model.transform(df)
    
            logger.debug(f"transformation result: {result}")
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
    
        def class_names(self) -> Iterable[str]:
            output_schema = self._model.metadata.get_output_schema()
            if output_schema is not None:
                columns = [schema["name"] for schema in output_schema.to_dict()]
            return columns
    ```

위와 같이 수정하면 디렉토리 구성은 다음과 같이 됩니다.
```bash
mlflowserver
├── Dockerfile
└── mrxflowserver
    ├── MLFlowServer.py
    ├── before-run
    ├── image_metadata.json
    ├── pip_env_create.py
    ├── pyfunc.py
    └── requirements.txt
```

## seldon-deploy
다음과 같은 `postprocessor.yaml` deploy 파일을 작성합니다.

- `postprocessor.yaml`
    ```yaml
    apiVersion: machinelearning.seldon.io/v1
    kind: SeldonDeployment
    metadata:
        name: graph-iris-postprocessor
        namespace: seldon
    spec:
        name: postprocessor
        predictors:
        - name: postprocessor
        componentSpecs:
        - spec:
            volumes:
            - name: model-provision-location
                emptyDir: {}
            containers:
            - image: ghcr.io/<github-owner>/serving-tutorial:latest
                name: postprocessor
                imagePullPolicy: Always
                securityContext:
                privileged: true
                runAsUser: 0
                runAsGroup: 0
                volumeMounts:
                - mountPath: /mnt/models
                name: model-provision-location
                readOnly: true
            imagePullSecrets:
            - name: mrx-ghcr
            initContainers:
            - name: postprocessor-initializer
                image: gcr.io/kfserving/storage-initializer:v0.4.0
                imagePullPolicy: Always
                args:
                - s3://mlflow/mlflow/artifacts/17/b84160df245441fa8d5ad7c5b62a424d/artifacts/iris_postprocessor_model/
                - /mnt/models
                envFrom:
                - secretRef:
                    name: seldon-init-container-secret
                volumeMounts:
                - mountPath: /mnt/models
                name: model-provision-location
        graph:
            name: postprocessor
            type: OUTPUT_TRANSFORMER
            parameters:
            - name: xtype
            type: STRING
            value: DataFrame
            - name: model_uri
            type: STRING
            value: /mnt/models
            children: []
        replicas: 1
    ```

- apply 합니다.
    ```bash
    kubectl apply -f postprocessor.yaml
    ```

- predict를 요청합니다.
    ```bash
    curl -X POST <http://10.102.231.216/seldon/seldon/graph-iris-postprocessor/api/v1.0/predictions> \\
        -H 'Content-Type: application/json' \\
        -d '{
            "data": {
                "ndarray": [
                [
                    -0.22568508012717417,
                    2.236410242744829,
                    0.9149322555240071
                ]
                ],
                "names": [
                "Class_0",
                "Class_1",
                "Class_2"
                ]
            }
        }'
    ```

- 다음과 같은 결과를 받을 수 있습니다.
    ```bash
    {
        "data": {
        "names": [
            "Class_0",
            "Class_1",
            "Class_2"
        ],
        "ndarray": [
            [
            -0.7139912744706434,
            1.41185905424711,
            -0.064123229924972
            ]
        ]
        },
        "meta": {
        "requestPath": {
            "postprocessor": "ghcr.io/<github-owner>/serving-tutorial:latest"
        }
        }
    }
    ```
