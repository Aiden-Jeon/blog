---
title: 밑바닥부터 시작하는 Seldon-Core - 설치 [Ambassador Ver.]
comment: true
categories: [seldon]
tags: ["k8s", "seldon", "ambassador"]
toc: true
date: 2021-05-14
author: Jongseob Jeon
---

---
**밑바닥부터 시작하는 Seldon-Core 순서**
1. 설치  
   1) [밑바닥부터 시작하는 Seldon-Core - 설치 [Istio Ver.]]({{< relref "post/tech/seldon/from-scratch/install-istio" >}})  
   2) [밑바닥부터 시작하는 Seldon-Core - 설치 [Ambassador Ver.]]({{< relref "post/tech/seldon/from-scratch/install-ambassador" >}})  

---

## Pre-requisites
1. Minikube
2. Helm
3. Ingress
4. Istio

### 1. Minikube
사용한 minikube version은 다음과 같습니다.
```bash
> minikube version
minikube version: v1.20.0
commit: c61663e942ec43b20e8e70839dcca52e44cd85ae
```

minikube의 default config는 memory의 경우 2048mb 입니다. 이 경우 실습 중 메모리가 부족해서 OOM이슈가 생겨서 정삭적으로 진행할 수 없을 수 도 있습니다. OOM을 방지하기 위해서 minikube의 메모리를 4096mb로 늘려줍니다.
```bash
> minikube config set memory 4096
❗  These changes will take effect upon a minikube delete and then a minikube start
```

minikube를 실행시킵니다.
```bash
> minikube start
😄  minikube v1.20.0 on Ubuntu 18.04
✨  Automatically selected the docker driver. Other choices: virtualbox, ssh
👍  Starting control plane node minikube in cluster minikube
🚜  Pulling base image ...
🔥  Creating docker container (CPUs=6, Memory=4096MB) ...
🐳  Preparing Kubernetes v1.20.2 on Docker 20.10.6 ...
    ▪ Generating certificates and keys ...
    ▪ Booting up control plane ...
    ▪ Configuring RBAC rules ...
🔎  Verifying Kubernetes components...
    ▪ Using image gcr.io/k8s-minikube/storage-provisioner:v5
🌟  Enabled addons: storage-provisioner, default-storageclass
🏄  Done! kubectl is now configured to use "minikube" cluster and "default" namespace by default
```


### 2. Helm
[helm 공식 홈페이지](https://helm.sh/docs/intro/install/)의 방법을 따라 합니다. 이 때 버전은 3 이상이어야 합니다.  
여러 방법중 스크립트 방법을 이용해 설치하겠습니다.
```bash
curl -fsSL -o get_helm.sh <https://raw.githubusercontent.com/helm/helm/master/scripts/get-helm-3>
chmod 700 get_helm.sh
./get_helm.sh
```

### 3. Ingress
minikube에서 ingress addon 을 사용합니다.
```bash
> minikube addons enable ingress
   ▪ Using image k8s.gcr.io/ingress-nginx/controller:v0.44.0
   ▪ Using image docker.io/jettech/kube-webhook-certgen:v1.5.1
   ▪ Using image docker.io/jettech/kube-webhook-certgen:v1.5.1
🔎  Verifying ingress addon...
🌟  The 'ingress' addon is enabled
```

### 4. Ambassador
helm chart를 이용해 설치합니다.
```bash
kubectl create namespace ambassador || echo "namespace ambassador exists"

helm repo add datawire <https://www.getambassador.io>
helm install ambassador datawire/ambassador \\
 --set image.repository=quay.io/datawire/ambassador \\
 --set enableAES=false \\
 --set crds.keep=false \\
 --namespace ambassador
```

상태를 확인합니다.
```bash
> kubectl get all -n ambassador
NAME                                    READY   STATUS    RESTARTS   AGE
pod/ambassador-5bf7689fc6-9hlrd         1/1     Running   0          116s
pod/ambassador-5bf7689fc6-bk86c         1/1     Running   0          116s
pod/ambassador-5bf7689fc6-r6v9q         1/1     Running   0          116s
pod/ambassador-agent-8585f84d86-vbczf   1/1     Running   0          116s

NAME                       TYPE           CLUSTER-IP       EXTERNAL-IP   PORT(S)                      AGE
service/ambassador         LoadBalancer   10.102.231.216   <pending>     80:30904/TCP,443:32156/TCP   116s
service/ambassador-admin   ClusterIP      10.99.228.85     <none>        8877/TCP,8005/TCP            116s

NAME                               READY   UP-TO-DATE   AVAILABLE   AGE
deployment.apps/ambassador         3/3     3            3           116s
deployment.apps/ambassador-agent   1/1     1            1           116s

NAME                                          DESIRED   CURRENT   READY   AGE
replicaset.apps/ambassador-5bf7689fc6         3         3         3       116s
replicaset.apps/ambassador-agent-8585f84d86   1         1         1       116s
```

## Seldon-core

### Install seldon-core

- namespace 생성
  ```bash
  > kubectl create namespace seldon-system
  namespace/seldon-system created
  ```

- helm chart 생성
  ```bash
  helm install seldon-core seldon-core-operator \\
      --repo <https://storage.googleapis.com/seldon-charts> \\
      --set usageMetrics.enabled=true \\
      --namespace seldon-system \\
      --set ambassador.enabled=true
  ```

- 상태 확인
  ```bash
  > kubectl get all -n seldon-system
  NAME                                            READY   STATUS    RESTARTS   AGE
  pod/seldon-controller-manager-559c567c9-jxbgf   1/1     Running   0          27s
  
  NAME                             TYPE        CLUSTER-IP     EXTERNAL-IP   PORT(S)   AGE
  service/seldon-webhook-service   ClusterIP   10.97.194.20   <none>        443/TCP   27s
  
  NAME                                        READY   UP-TO-DATE   AVAILABLE   AGE
  deployment.apps/seldon-controller-manager   1/1     1            1           27s
  
  NAME                                                  DESIRED   CURRENT   READY   AGE
  replicaset.apps/seldon-controller-manager-559c567c9   1         1         1       27s
  ```

### Sample deploy

- namespace 생성
  ```bash
  kubectl create namespace seldon
  ```

- pod 생성
  ```bash
  kubectl apply -f - << END
  apiVersion: machinelearning.seldon.io/v1
  kind: SeldonDeployment
  metadata:
    name: iris-model
    namespace: seldon
  spec:
    name: iris
    predictors:
    - graph:
        implementation: SKLEARN_SERVER
        modelUri: gs://seldon-models/sklearn/iris
        name: classifier
      name: default
      replicas: 1
  END
  ```

- pod 생성 확인
  ```bash
  > kubectl get all -n seldon
  NAME                                                   READY   STATUS    RESTARTS   AGE
  pod/iris-model-default-0-classifier-546fb8bfff-cd5gm   2/2     Running   0          5m47s
  
  NAME                                    TYPE        CLUSTER-IP       EXTERNAL-IP   PORT(S)             AGE
  service/iris-model-default              ClusterIP   10.98.254.42     <none>        8000/TCP,5001/TCP   71s
  service/iris-model-default-classifier   ClusterIP   10.101.118.187   <none>        9000/TCP,9500/TCP   5m47s
  
  NAME                                              READY   UP-TO-DATE   AVAILABLE   AGE
  deployment.apps/iris-model-default-0-classifier   1/1     1            1           5m47s
  
  NAME                                                         DESIRED   CURRENT   READY   AGE
  replicaset.apps/iris-model-default-0-classifier-546fb8bfff   1         1         1       5m47s
  ```

- minikube tunnel
  ```bash
  > minikube tunnel
  Status:
          machine: minikube
          pid: 117019
          route: 10.96.0.0/12 -> 192.168.49.2
          minikube: Running
          services: [istio-ingressgateway]
      errors: 
                  minikube: no errors
                  router: no errors
                  loadbalancer emulator: no errors
  ```

- ambassador external ip 확인하기
  ```bash
  > k get svc -n ambassador
  NAME               TYPE           CLUSTER-IP       EXTERNAL-IP      PORT(S)                      AGE
  ambassador         LoadBalancer   10.102.231.216   10.102.231.216   80:30904/TCP,443:32156/TCP   5m6s
  ambassador-admin   ClusterIP      10.99.228.85     <none>           8877/TCP,8005/TCP            5m6s
  ```
  → 이 경우 10.102.231.216

- predict 요청 날려보기
  ```bash
  curl -X POST <http://10.102.231.216/seldon/seldon/iris-model/api/v1.0/predictions> \\
      -H 'Content-Type: application/json' \\
      -d '{ "data": { "ndarray": [[1,2,3,4]] } }'
  ```

  - format
    - `http://<ambassador_url>/seldon/<namespace>/<model-name>/api/v1.0/doc/`

- 다음과 같은 결과를 얻을 수 있습니다.
  ```bash
  {"data":{"names":["t:0","t:1","t:2"],"ndarray":[[0.0006985194531162841,0.003668039039435755,0.9956334415074478]]},"meta":{"requestPath":{"classifier":"seldonio/sklearnserver:1.7.0"}}}
  ```

## Seldon-analytics

### Install
- helm chart 설치
  ```bash
  helm install seldon-core-analytics seldon-core-analytics \\
     --repo <https://storage.googleapis.com/seldon-charts> \\
     --namespace seldon-system
  ```

### Usage
- 설치된 seldon-core-analytics port-forward
  ```bash
  kubectl port-forward svc/seldon-core-analytics-grafana 3000:80 -n seldon-system
  ```

- http://localhost:3000 에 접속합니다.
  - admin / password 로 로그인 합니다.
  - dashboard에서 prediction analytics를 클릭합니다
    ![img-0](/imgs/seldon/from-scratch-install-istio-0.png)
  - 다음과 같은 dashboard를 볼 수 있습니다.
    ![img-1](/imgs/seldon/from-scratch-install-istio-1.png)
