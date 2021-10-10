---
title:  (CKA) 01. Core Concepts
categories: [kubernetes]
tags: ["k8s", "cka"]
toc:
  auto: true
date: 2021-05-10
author: Jongseob Jeon
---

CKA를 준비하면서 공부한 요약 내용입니다.
- [강의](https://www.udemy.com/course/certified-kubernetes-administrator-with-practice-tests/)
- [What is CKA?](https://www.cncf.io/certification/cka/)

# 0. Tips

1. ailias `kubectl` to `k`
```bash
alias k="kubectl"
k get po
```

2. shortcuts
- pod = po
- service = svc
- namespace = ns
- replicasets = rs

3. do not write all config file
- use dry run
  - `kubectl run nginx --image=nginx --dry-run=client -o yaml`
  - `kubectl create deployment --image=nginx nginx --dry-run=client -o yaml`
- export it to yaml file
  - `kubectl create deployment --image=nginx nginx --dry-run=client -o yaml > nginx-deployment.yaml`


## 1. Pods
### 생성
1. yaml
```yaml
# pod.yaml
apiVersion: v1
kind: pod
metadata:
  name: nginx
spec:
  containers:
    - name: nginx-container
      image: nginx
```
```bash
kubectl apply -f pod.yaml
```

2. run
```bash
kubectl run nginx --image=nginx
```

### 상태

- 기본
  ```bash
  > k get po
  NAME            READY   STATUS             RESTARTS   AGE
  newpods-6nl8r   1/1     Running            0          3m
  newpods-9sp8p   1/1     Running            0          3m
  newpods-tplx9   1/1     Running            0          3m
  nginx           1/1     Running            0          3m11s
  webapp          1/2     ImagePullBackOff   0          54s
  ```

  - READY means
    - running_containers in pod / total_containers in pod

- wide
  ```bash
  > k get po -o wide
  NAME            READY   STATUS              RESTARTS   AGE   IP       NODE           NOMINATED NODE   READINESS GATES
  newpods-6nl8r   0/1     ContainerCreating   0          14s   <none>   controlplane   <none>           <none>
  newpods-9sp8p   0/1     ContainerCreating   0          14s   <none>   controlplane   <none>           <none>
  newpods-tplx9   0/1     ContainerCreating   0          14s   <none>   controlplane   <none>           <none>
  nginx           0/1     ContainerCreating   0          25s   <none>   controlplane   <none>           <none>
  ```

  - NODE

- describe
  ```bash
  > k describe po webapp
  Name:         webapp
  Namespace:    default
  Priority:     0
  Node:         controlplane/10.77.67.9
  Start Time:   Mon, 10 May 2021 03:54:38 +0000
  Labels:       <none>
  Annotations:  <none>
  Status:       Pending
  IP:           10.244.0.8
  IPs:
    IP:  10.244.0.8
  Containers:
    nginx:
      Container ID:   docker://f9580f7e4b7c51b53f0cb0d94ff913b643706a65a103a58614c3c254cf26043f
      Image:          nginx
      Image ID:       docker-pullable://nginx@sha256:75a55d33ecc73c2a242450a9f1cc858499d468f077ea942867e662c247b5e412
      Port:           <none>
      Host Port:      <none>
      State:          Running
        Started:      Mon, 10 May 2021 03:54:41 +0000
      Ready:          True
      Restart Count:  0
      Environment:    <none>
      Mounts:
        /var/run/secrets/kubernetes.io/serviceaccount from default-token-pz92t (ro)
    agentx:
      Container ID:   
      Image:          agentx
      Image ID:       
      Port:           <none>
      Host Port:      <none>
      State:          Waiting
        Reason:       ImagePullBackOff
      Ready:          False
      Restart Count:  0
      Environment:    <none>
      Mounts:
        /var/run/secrets/kubernetes.io/serviceaccount from default-token-pz92t (ro)
  Conditions:
    Type              Status
    Initialized       True 
    Ready             False 
    ContainersReady   False 
    PodScheduled      True 
  Volumes:
    default-token-pz92t:
      Type:        Secret (a volume populated by a Secret)
      SecretName:  default-token-pz92t
      Optional:    false
  QoS Class:       BestEffort
  Node-Selectors:  <none>
  Tolerations:     node.kubernetes.io/not-ready:NoExecute op=Exists for 300s
                   node.kubernetes.io/unreachable:NoExecute op=Exists for 300s
  Events:
    Type     Reason     Age                From               Message
    ----     ------     ----               ----               -------
    Normal   Scheduled  17s                default-scheduler  Successfully assigned default/webapp to controlplane
    Normal   Pulling    15s                kubelet            Pulling image "nginx"
    Normal   Pulled     15s                kubelet            Successfully pulled image "nginx" in 170.242045ms
    Normal   Created    15s                kubelet            Created container nginx
    Normal   Started    14s                kubelet            Started container nginx
    Normal   Pulling    14s                kubelet            Pulling image "agentx"
    Warning  Failed     13s                kubelet            Failed to pull image "agentx": rpc error: code = Unknown desc = Error response from daemon: pull access denied for agentx, repository does not exist or may require 'docker login': denied: requested access to the resource is denied
    Warning  Failed     13s                kubelet            Error: ErrImagePull
    Normal   BackOff    12s (x2 over 13s)  kubelet            Back-off pulling image "agentx"
    Warning  Failed     12s (x2 over 13s)  kubelet            Error: ImagePullBackOff
  ```

### 삭제
```bash
> k delete po webapp
pod "webapp" deleted
```

### 수정
```bash
k edit po redis
```

## 2. Replication Controller
What is replica and why need controller?  
-> replication controller runs multiple instances of a single pod in the k8s cluster


### 특징
1. high availability
   - multiplie pod or single pod
   - replication controller ensures that the specified number of pods are running at all times.
2. Load Balancing & Scaling
   - balance the load
   - spans across multiple nodes
 
### 종류
- replication controller
  - old technology
- replica set
  - recommended

### 생성
- defintion
  ```yaml
  # ReplicaSet
  apiVersion: apps/v1
  kind: ReplicaSet
  metadata:
    name: myapp-replicaset
    labels:
      app: myapp
      type: front-end
  spec:
    template:
      metadata: myapp-pod
        name: myapp-pod
        labels:
          app: myapp
          type: front-end
        spec:
          containers:
            - name: nginx-container
              image: nginx
    replicas: 3
    selector:
      matchLabels:
        type: front-end
  ```

### 상태
```bash
> k get replicasets
NAME              DESIRED   CURRENT   READY   AGE
new-replica-set   4         4         0       11s
```

### 수정
```bash
> k edit replicaset            
replicaset.apps/new-replica-set edited
```

### 삭제
```bash
> k delete po --all
```

-> 자동으로 다시 pod 생성 됨
```bash
ot@controlplane:~# k get po
NAME                    READY   STATUS              RESTARTS   AGE
new-replica-set-7mx8k   0/1     Terminating         0          4m22s
new-replica-set-7p95n   0/1     ContainerCreating   0          17s
new-replica-set-dcl5m   0/1     Terminating         0          4m22s
new-replica-set-hhpm9   0/1     ContainerCreating   0          17s
new-replica-set-hr54x   0/1     ContainerCreating   0          17s
new-replica-set-m6dmb   0/1     Terminating         0          4m22s
new-replica-set-v2vr5   0/1     ContainerCreating   0          17s
new-replica-set-vmts5   0/1     Terminating         0          4m22s
```

### scale
1. replace w\ file
   ```bash
   kubectl replace -f ~~.yaml
   ```
2. scale w\ file
   ```bash
   kubectl scale --replicas=6 -f ~~.yaml
   ```
3. scale w\ resource
   ```bash
   kubectl scale --replicas=6 replicaset myapp-replicaset
   ```
4. edit
   ```bash
   kubectl edit replicaset
   ```

- 실습
  ```bash
  > k scale --replicas=5 rs new-replica-set 
  replicaset.apps/new-replica-set scaled
  ```

## 3. Deployments
- rolling updates
- rolling back

![img-1](/imgs/cka/core_concepts-1.png)

## 4. Namespaces
- user의 실수부터 보호하기 위해서 처음 3개의 namespace가 생성됨
  - Default
  - kube-system
  - kube-public
- isolate resources between namespace
- own policies
  - each namespace is guaranteed certain amount and does not to use more.
- to reach in same namespace
  - `mysql.connect("db-service")`
- to reach in other namespace
  - `mysql.connect(db-service.dev.svc.cluster.local)`
  - `{service-name}.{namespace}.{service}-{domain-name}`

### 생성
```bash
k create ns dev-ns
namespace/dev-ns created
```

### 목록
```bash
> k get ns

NAME              STATUS   AGE
default           Active   88s
dev               Active   34s
finance           Active   34s
kube-node-lease   Active   93s
kube-public       Active   94s
kube-system       Active   95s
manufacturing     Active   33s
marketing         Active   34s
prod              Active   33s
research          Active   33s
```

### 상태
1. `--namespace`
   ```bash
   kubectl get pods --namespace=kube-system
   ```
2. `-n`
   ```bash
   kubectl get pods -n kube-system
   ```

### change default

```bash
kubectl config set-context $(kubectl config current-context) --namespace=dev
```

### pod 생성

```bash
k run redis --image=redis --namespace=finance
pod/redis created
```

### 전체 확인

```bash
k get po --all-namespaces
```

- 하나의 값만 찾고 싶을 때

  ```bash
  k get po --all-namespaces | grep -i blue
  ```

## 5. Services

allow to communicate w\ people, backend, frontend, extra data source

**types**

- node port
  - make an internal pod accessible on a port on node
- cluster ip
  - creates a virtual IP inside the cluster to enable communication b2n different services
- load balancer
  - provisions a load balancer for service for cloud provider

**NodePort**

![img-2](/imgs/cka/core_concepts-2.png)

- target port
- port
  - port of service itself
- node port
  - valid range: 30000 ~ 32767

```yaml
apiVersion: v1
kind: SErvice
metadata:
  name: myapp-service
spec:
  type: NodePort
  ports:
    - targetPort: 80
      port: 80
      nodePort: 30008
```

- if `targetPort` is not given
  - same as `port`
- if `nodePort` is not given
  - automatically allocated
- randomly distributed if same port is exists

**ClusterIP**
![img-3](/imgs/cka/core_concepts-3.png)

**LoadBalancer**

### 상태

```bash
> k get svc

NAME         TYPE        CLUSTER-IP   EXTERNAL-IP   PORT(S)   AGE
kubernetes   ClusterIP   10.96.0.1    <none>        443/TCP   3m46s
```

### 명령어

```bash
> kubectl expose pod redis --port=6379 --name redis-service
```

### expose

```bash
> kubectl run custom-nginx --image=nginx --port=8080
kubectl run httpd --image=httpd:alpine --port=80 --expose
```

## 6. Imperative vs Declaritive
![img-4](/imgs/cka/core_concepts-4.png)

### Infrastructure as Code
![img-5](/imgs/cka/core_concepts-5.png)

### Kubernetes
![img-6](/imgs/cka/core_concepts-6.png)
**Imperative Commands**

- create objects
- update objects
→ hard to keep track

Imperative Configuration Files
- create objects
  - `kubectl create -f ~.yaml`
- update obejcts
  - `kubectl edit ~~ ~~`
  - `kubectl replace -f ~.yaml`

**Declaritve**
- create objects
  - `kubectl apply -f ~.yaml`
  - `kubectl apply -f /path/to/config-files`
- update objects
  - `kubectl apply -f ~.yaml`
