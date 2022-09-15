---
title: kubeflow 설치 중 auth dex 이슈 해결하기
categories: [mlops]
tags: ["k8s", "k3s"]
toc: true
date: 2021-10-08
author: Jongseob Jeon
---

Kubeflow 설치 중 auth의 dex pod이 동작하지 않는 것을 확인했습니다.
우선, auth namespace를 확인해봤습니다.
```bash
kubectl get all -n auth
```
다음과 같은 결과를 얻을 수 있었습니다.
```bash
NAME                       READY   STATUS   RESTARTS   AGE
pod/dex-6f4f4fd769-wdfhp   0/1     Error    197        16h

NAME          TYPE       CLUSTER-IP    EXTERNAL-IP   PORT(S)          AGE
service/dex   NodePort   10.43.69.43   <none>        5556:32000/TCP   16h

NAME                  READY   UP-TO-DATE   AVAILABLE   AGE
deployment.apps/dex   0/1     1            0           16h

NAME                             DESIRED   CURRENT   READY   AGE
replicaset.apps/dex-6f4f4fd769   1         1         0       16h
```

dex pod을 확인해 보니 다음과 같았습니다.
```bash
kubectl describe -n auth po dex-6f4f4fd769-wdfhp
```

CrashLoopBackOff 상태임을 확인했습니다.
```
Name:         dex-6f4f4fd769-wdfhp
Namespace:    auth
Priority:     0
Node:         mrx-desktop3/10.160.121.43
Start Time:   Thu, 07 Oct 2021 17:56:55 +0900
Labels:       app=dex
              pod-template-hash=6f4f4fd769
Annotations:  <none>
Status:       Running
IP:           10.42.0.36
IPs:
  IP:           10.42.0.36
Controlled By:  ReplicaSet/dex-6f4f4fd769
Containers:
  dex:
    Container ID:  docker://5d1ec3cba7b0630e7fd254fbd5c5372a5459235c696884e209606a0e0facc856
    Image:         quay.io/dexidp/dex:v2.24.0
    Image ID:      docker-pullable://quay.io/dexidp/dex@sha256:c9b7f6d0d9539bc648e278d64de46eb45f8d1e984139f934ae26694fe7de6077
    Port:          5556/TCP
    Host Port:     0/TCP
    Command:
      dex
      serve
      /etc/dex/cfg/config.yaml
    State:          Waiting
      Reason:       CrashLoopBackOff
    Last State:     Terminated
      Reason:       Error
      Exit Code:    2
      Started:      Fri, 08 Oct 2021 10:20:28 +0900
      Finished:     Fri, 08 Oct 2021 10:20:28 +0900
    Ready:          False
    Restart Count:  197
    Environment Variables from:
      dex-oidc-client  Secret  Optional: false
    Environment:       <none>
    Mounts:
      /etc/dex/cfg from config (rw)
      /var/run/secrets/kubernetes.io/serviceaccount from kube-api-access-r69m8 (ro)
Conditions:
  Type              Status
  Initialized       True
  Ready             False
  ContainersReady   False
  PodScheduled      True
Volumes:
  config:
    Type:      ConfigMap (a volume populated by a ConfigMap)
    Name:      dex
    Optional:  false
  kube-api-access-r69m8:
    Type:                    Projected (a volume that contains injected data from multiple sources)
    TokenExpirationSeconds:  3607
```

좀 더 자세한 내용을 확인하기 위해서 로그도 확인해봤습니다.
```bash
kubectl log -n auth po dex-6f4f4fd769-wdfhp
```
로그 결과는 아래와 같았습니다.
```bash
time="2021-10-08T01:20:28Z" level=info msg="config using log level: debug"
time="2021-10-08T01:20:28Z" level=info msg="config issuer: http://dex.auth.svc.cluster.local:5556/dex"
failed to initialize storage: failed to inspect service account token: jwt claim "kubernetes.io/serviceaccount/namespace" not found
```

제일 마지막에 있는 이슈를 기준으로 찾아보니 쿠버네티스 v1.21 이후 dex에서 jwt 파싱에 관한 문제였습니다.  
https://github.com/dexidp/dex/issues/2082

해결 방법은 dex container에 `KUBERNETES_POD_NAMESPACE` env을 직접 추가해주면 됩니다.

저는 helm차트를 이용하지 않았기 때문에 직접 deployment를 수정하였습니다.
```bash
kubectl edit deployments -n auth dex
```

```bash
spec:
  template:
    spec:
      containers:
        - name:dex
          env:
          - name: KUBERNETES_POD_NAMESPACE
            value: <your-namespace-name>
```

추가 후 확인하면 다음과 같이 됩니다.
```bash
k get all -n auth
```
```bash
NAME                      READY   STATUS    RESTARTS   AGE
pod/dex-9bfcdd65c-9pt7c   1/1     Running   0          70s

NAME          TYPE       CLUSTER-IP    EXTERNAL-IP   PORT(S)          AGE
service/dex   NodePort   10.43.69.43   <none>        5556:32000/TCP   16h

NAME                  READY   UP-TO-DATE   AVAILABLE   AGE
deployment.apps/dex   1/1     1            1           16h

NAME                             DESIRED   CURRENT   READY   AGE
replicaset.apps/dex-6f4f4fd769   0         0         0       16h
replicaset.apps/dex-9bfcdd65c    1         1         1       70s
replicaset.apps/dex-58c689dffb   0         0         0       8m32s
```