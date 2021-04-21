---
title: minikube를 이용해 seldon 사용해보기
comment: true
categories: [kubernetes]
tags: ["k8s", "seldon"]
toc: true
date: 2021-04-21
author: Jongseob Jeon
---

이번 포스트는 Seldon Core를 설치하는 법에 대해서 설명합니다.

사용한 k8s 환경은 minikube 입니다.

## 1. Install
우선, minikube를 시작합니다
```bash
minikube start
```

그리고 helm을 이용해 seldon-core를 설치합니다.
```bash
> kubectl create ns seldon-system
> helm install seldon-core seldon-core-operator \
    --repo https://storage.googleapis.com/seldon-charts \
    --set usageMetrics.enabled=true \
    --namespace seldon-system
```

상태를 확인해 봅니다.

```bash
> kubectl get all -n seldon-system
NAME                                            READY   STATUS    RESTARTS   AGE
pod/seldon-controller-manager-559c567c9-9tj9w   1/1     Running   0          11s

NAME                             TYPE        CLUSTER-IP    EXTERNAL-IP   PORT(S)   AGE
service/seldon-webhook-service   ClusterIP   10.101.37.1   <none>        443/TCP   11s

NAME                                        READY   UP-TO-DATE   AVAILABLE   AGE
deployment.apps/seldon-controller-manager   1/1     1            1           11s

NAME                                                  DESIRED   CURRENT   READY   AGE
replicaset.apps/seldon-controller-manager-559c567c9   1         1         1       11s
```

## 2. Sklearn Model
deploy에 이용할 모델을 만들어 보겠습니다.
모델을 생성하는 코드입니다.

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

    joblib.dump(clf, "model.joblib")

if __name__ == "__main__":
    train()
```

실행 시켜서 `model.joblib` 을 생성합니다.

## 3. pvc

pod에 모델 파일을 옮기기 위해 pvc를 생성합니다.

우선 minikube에 접속해서 폴더를 생성합니다.

```bash
> minikube ssh
> docker@minikube:~$ sudo mkdir -p /mnt/models/sklearn
```

그리고 생성된 경로를 바라보는 pvc를 생성합니다.

```yaml
# sklearn-pv-volume.yaml
apiVersion: v1
kind: PersistentVolume
metadata:
  name: sklearn-pv-volume
  labels:
    type: local
spec:
  storageClassName: manual
  capacity:
    storage: 1Gi
  accessModes:
    - ReadWriteOnce
  hostPath:
    path: "/mnt/models/sklearn"
---
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: sklearn-pv-volume
  namespace: seldon
spec:
  storageClassName: manual
  accessModes:
    - ReadWriteOnce
  resources:
    requests:
      storage: 1Gi
```


생성 후 상태를 확인합니다.

```bash
> kubectl create ns seldon
> kubectl apply -f sklearn-pv-volume.yaml
persistentvolume/sklearn-pv-volume created

> kubectl get pv sklearn-pv-volume
NAME                CAPACITY   ACCESS MODES   RECLAIM POLICY   STATUS   CLAIM                      STORAGECLASS   REASON   AGE
sklearn-pv-volume   1Gi        RWO            Retain           Bound    seldon/sklearn-pv-volume   manual                  53s
```

## 4. Deploy

이제 SeldonDeployment를 생성하겠습니다.

간단하게 SKLEARN_SERVER를 사용하는 deployment를 띄우겠습니다. 우선 아래의 내용을 `deploy.yaml` 파일에 작성합니다.

```bash
# seldom-deploy.yaml
apiVersion: machinelearning.seldon.io/v1
kind: SeldonDeployment
metadata:
  name: iris-model
  namespace: seldon
spec:
  name: iris-model
  predictors:
  - graph:
      children: []
      implementation: SKLEARN_SERVER
      modelUri: pvc://sklearn-pv-volume/
      name: classifier
      parameters:
        - name: method
          type: STRING
          value: predict
    name: default
    replicas: 1
```

그리고 apply를 합니다.

```bash
kubectl apply -f deploy.yaml
```

상태를 확인합니다.

```bash
> k get po -n seldon
NAME                                               READY   STATUS    RESTARTS   AGE
iris-model-default-0-classifier-6bb64c9588-5q8ck   0/2     Running   0          18s
```

## 5. test

정상적으로 작동하는지 확인하기 위한 테스트를 해보겠습니다.

우선 서비스에 어떤 것들이 있는지 확인해보겠습니다.

```bash
kubectl get svc -n seldon
NAME                            TYPE        CLUSTER-IP     EXTERNAL-IP   PORT(S)             AGE
iris-model-default              ClusterIP   10.96.166.74   <none>        8000/TCP,5001/TCP   89s
iris-model-default-classifier   ClusterIP   10.102.2.217   <none>        9000/TCP,9500/TCP   114s
```

이 중 `classifier` 가 붙어있는 svc 가 predict를 담당합니다.

로컬에서 사용할 수 있게 포트포워딩을 해줍니다.

```bash
> kubectl port-forward -n seldon svc/iris-model-default-classifier 9000
Forwarding from 127.0.0.1:9000 -> 9000
Forwarding from [::1]:9000 -> 9000
```

inference할 데이터를 생성해보겠습니다.

다음 파일을 input.json에 저장합니다.

```json
{
    "data": {
      "ndarray": [
        [6.8,  2.8,  4.8,  1.4],
        [6.0,  3.4,  4.5,  1.6]
      ]
    }
  }
```

다음 명령어를 이용해 predict를 해봅니다.

```bash
> curl -H "Content-Type: application/json" http://localhost:9000/predict -d @./input.json
{"data":{"names":[],"ndarray":[1,1]},"meta":{"requestPath":{"classifier":"seldonio/sklearnserver:1.7.0"}}}
```

`[1, 1]` 로 예측되어서 나오는 것을 확인할 수 있습니다.
