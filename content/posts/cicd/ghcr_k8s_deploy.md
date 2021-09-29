---
title: ghcr을 이용한 kubernetes deployment 만들기
categories: [cicd]
tags: ["k8s", "ghcr"]
toc: true
date: 2021-03-04
author: Jongseob Jeon
---

**CI/CD Contents 순서**
1. [sphinx-autoapi 를 이용한 자동 api 문서 생성하기]({{< relref "posts/cicd/sphinx_autoapi" >}})
2. [github action을 이용한 ci]({{< relref "posts/cicd/github_action_ci" >}})
3. [ghcr을 이용한 kubernetes deployment 만들기]({{< relref "posts/cicd/ghcr_k8s_deploy" >}})
4. [helm을 이용한 deployment chart 만들기]({{< relref "posts/cicd/helm_deployment_chart" >}})
5. [argocd를 이용한 cd]({{< relref "posts/cicd/argocd_cd" >}})

---


이번 포스트에서는 minikube를 이용해 [이전 포스트](https://aiden-jeon.github.io/cicd/github-cicd-2)에서 만든 ghcr package를 이용해 deployment를 만드는 법에 대해서 알아보겠습니다.

## 0. requirements
본 포스트에서는 local환경에서 minikube를 사용하고 있습니다.

## 1. `deployment.yaml`
`docker/deploy` 폴더 밑에 `deployment.yaml` 파일을 작성하겠습니다.
```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: sphinx-doc
spec:
  selector:
    matchLabels:
      type: app
      service: sphinx-doc
  template:
    metadata:
      labels:
        type: app
        service: sphinx-doc
    spec:
      containers:
      - name: sphinx-doc
        image: ghcr.io/aiden-jeon/sphinx-api:latest
        imagePullPolicy: Always
        ports:
        - containerPort: 80
      imagePullSecrets:
        - name: test-ghcr
```

이전 포스트에서 작성한 sphinx-api를 띄우는 deployment 입니다.
apply하기에 앞서 secrets를 설정해주어야 합니다. 자세한 방법은 [포스트]({{< relref "posts/kubernetes/kubernetes_ghcr" >}})를 참고해주세요.

secrets를 설정했다면 deployment.yaml 파일을 apply 해줍니다.
```bash
> kubectl apply -f deployment.yaml

deployment.apps/sphinx-doc created
```

pod이 정상적으로 image를 pull 했는지 확인해봅니다.
```bash
> k describe po

Name:         sphinx-doc-7cdd4b7c55-cqlk4
Namespace:    default
Priority:     0
Node:         minikube/192.168.64.2
Start Time:   Thu, 04 Mar 2021 15:37:04 +0900
Labels:       pod-template-hash=7cdd4b7c55
              service=sphinx-doc
              type=app
Annotations:  <none>
Status:       Running
IP:           172.17.0.4
IPs:
  IP:           172.17.0.4
Controlled By:  ReplicaSet/sphinx-doc-7cdd4b7c55
Containers:
  sphinx-doc:
    Container ID:   docker://26991f0eed6db1208c3a3cd92d5f00bbe5ba37934baed29e964d021e92603315
    Image:          ghcr.io/aiden-jeon/sphinx-api:latest
    Image ID:       docker-pullable://ghcr.io/aiden-jeon/sphinx-api@sha256:f35c4c0852edaab2bfd617f2a42cf3fe141a08cd3692ac3105f1cb6be3591135
    Port:           80/TCP
    Host Port:      0/TCP
    State:          Running
      Started:      Thu, 04 Mar 2021 15:37:08 +0900
    Ready:          True
    Restart Count:  0
    Environment:    <none>
    Mounts:
      /var/run/secrets/kubernetes.io/serviceaccount from default-token-d7kw7 (ro)
Conditions:
  Type              Status
  Initialized       True 
  Ready             True 
  ContainersReady   True 
  PodScheduled      True 
Volumes:
  default-token-d7kw7:
    Type:        Secret (a volume populated by a Secret)
    SecretName:  default-token-d7kw7
    Optional:    false
QoS Class:       BestEffort
Node-Selectors:  <none>
Tolerations:     node.kubernetes.io/not-ready:NoExecute op=Exists for 300s
                 node.kubernetes.io/unreachable:NoExecute op=Exists for 300s
Events:
  Type     Reason       Age    From               Message
  ----     ------       ----   ----               -------
  Normal   Scheduled    5m10s  default-scheduler  Successfully assigned default/sphinx-doc-7cdd4b7c55-cqlk4 to minikube
  Warning  FailedMount  5m9s   kubelet            MountVolume.SetUp failed for volume "default-token-d7kw7" : failed to sync secret cache: timed out waiting for the condition
  Normal   Pulling      5m8s   kubelet            Pulling image "ghcr.io/aiden-jeon/sphinx-api:latest"
  Normal   Pulled       5m6s   kubelet            Successfully pulled image "ghcr.io/aiden-jeon/sphinx-api:latest" in 1.246291235s
  Normal   Created      5m6s   kubelet            Created container sphinx-doc
  Normal   Started      5m6s   kubelet            Started container sphinx-doc
```
맨 마지막의 Events 항목을 보면 정상적으로 pull 했음을 확인할 수 있습니다.

정상적으로 띄우고 있는지 확인하기 위해서 curl pod을 이용해 접근해 보겠습니다.
```bash
kubectl run curl -it --image=curlimages/curl sh
```

deployment에서 띄운 pod의 ip는 172.17.0.4 입니다. 다음 명령어로 정상적으로 띄우고 있는지 확인합니다.
```bash
curl 172.17.0.4
```
정상적으로 띄우고 있다면 아래와 같이 출력됩니다.
```bash

<!DOCTYPE html>
<html class="writer-html5" lang="en" >
<head>
  <meta charset="utf-8" />
  
  <meta name="viewport" content="width=device-width, initial-scale=1.0" />
  
  <title>Welcome to example’s documentation! &mdash; example 0.1 documentation</title>
...
```

## 2. `service.yaml`
위에서는 pod을 이용해 cluster 안에 있는 pod의 내용을 확인할 수 있었습니다. 이번에는 service를 이용해 로컬에서도 볼 수 있도록 하겠습니다.

`docker/deploy/`에서 `service.yaml`을 작성해줍니다.
```yaml
apiVersion: v1
kind: Service
metadata:
  name: sphinx-doc-svc
spec:
  type: NodePort
  ports:
  - port: 8000
    targetPort: 80
    protocol: TCP
  selector:
    type: app
    service: sphinx-doc
```

apply를 합니다.
```bash
> kubectl apply -f service.yaml

service/sphinx-api-svc created
```

정상적으로 됐는지 확인합니다.
```bash
> kubectl get svc

NAME             TYPE        CLUSTER-IP      EXTERNAL-IP   PORT(S)          AGE
kubernetes       ClusterIP   10.96.0.1       <none>        443/TCP          44h
sphinx-doc-svc   NodePort    10.105.108.18   <none>        8000:30511/TCP   43s
```

다음 명령어로 ip로 접근할 수 있습니다.
```
> minikube service -n default --url sphinx-doc-svc

http://192.168.64.2:30511
```

정상적으로 document를 보여주는 것을 확인할 수 있습니다.

## 3. `ingress.yaml`
다음으로 ip 주소를 domain 으로 바꾸기 위한 ingress를 작성해 보겠습니다.

`docker/deploy/`에서 `ingress.yaml`을 작성해줍니다.
```yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: sphinx-doc-ingress
spec:
  rules:
    - host: docs.sphinx.com
      http:
        paths:
          - path: /
            pathType: Prefix
            backend:
              service:
                name: sphinx-doc-svc
                port:
                  number: 8000
```

apply 해줍니다.
```bash
> kubectl apply -f ingress.yaml

ingress.networking.k8s.io/sphinx-doc-ingress created
```

정상적으로 됐는지 확인합니다.
```bash
> kubectl get ingress

NAME                 CLASS    HOSTS             ADDRESS        PORTS   AGE
sphinx-doc-ingress   <none>   docs.sphinx.com   192.168.64.2   80      49s
```

`/etc/hosts`를 열어줍니다. (환경에 따라 `/etec/host` 일 수 도 있습니다.)
```bash
sudo vim /etc/hosts
```
파일을 열고 가장 마지막에 minikube ip와 ingress에 작성한 도메인을 입력합니다.
```vim
192.168.64.2 docs.sphinx.com
```
위와 같이 적고 저장 후 닫습니다.
http://docs.sphinx.com/ 를 들어가서 정상적으로 api document가 나오는지 확인합니다.
