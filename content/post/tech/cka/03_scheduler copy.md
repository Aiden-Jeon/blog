---
title: \[CKA\] 03. Scheduler
comment: true
categories: [kubernetes]
tags: ["k8s", "cka"]
toc: true
date: 2021-05-16
author: Jongseob Jeon
---

CKA를 준비하면서 공부한 요약 내용입니다.
- [강의](https://www.udemy.com/course/certified-kubernetes-administrator-with-practice-tests/)
- [What is CKA?](https://www.cncf.io/certification/cka/)

## Node name
- default: `kubectl-scheduler`
- 위 작업이 기본적으로 pod을 node에 할당해 줌

### 상태 확인

- `kubectl get node`

- `node` = `no`

  ```yaml
  > k get no
  NAME           STATUS   ROLES                  AGE     VERSION
  controlplane   Ready    control-plane,master   11m     v1.20.0
  node01         Ready    <none>                 9m48s   v1.20.0
  ```

### 수동 배정

- To schedule node manually
- `NodeName`
  ```yaml
  ---
  apiVersion: v1
  kind: Pod
  metadata:
    name: nginx
  spec:
    containers:
    -  image: nginx
       name: nginx
    nodeName: node01
  ```

## Label & Selectors

### Label

- definition
  ```yaml
  apiVersion: ~~
  kind: ~~
  metadata:
      name: ~~
      labels:
          app: App1
          function: Front-end
  sepc: ~~
  ```
  → on `metadata: labels`

- imperative
  ```yaml
  kubectl label node node01 key=value
  ```

### Select

- `kubectl get pods --selector app=App1`
- several selector field
  - `kubectl get pods --selector app=App1,env=dev`

### Replicaset
- `replica-definition.yaml`
  ```yaml
  apiVersion: apps/v1
  kind: Replcaset
  metadata:
      name: simple-webapp
      labels:
          app: App1
          function: Front-end
  spec:
      replicas: 3
      selector:
          matchLabels:
              app: App1
      template:
          metadata:
              labels:
                  app: App1
                  function: Front-end
          spec:
              containers:
              - name: simple-webapp
                  image: simple-webapp
  ```
- labels at the top are the labels of the replicaset itself.
  ```yaml
  apiVersion: apps/v1
  kind: Replcaset
  metadata:
      name: simple-webapp
      labels:
          app: App1
          function: Front-end
  ```
- in order to connect the replicaset to the pod, configure the selector field
  ```yaml
      selector:
          matchLabels:
              app: App1
  ```

### Annotations

- used to record other detail for informatively purpose
- eg) tool details

## Taints, Tolerations

- taints
  - set on the nodes
- toleartions
  - set on the pods
- taints and toleration does not tell the pod to go to a particular node
- node to only accept pod with certain tolerations

### Taints

#### 설정

- `kubectl taint nodes node-name key=value:taint-effect`
- `taint-effect` :
  - `NoSchedule`
    - not to be scheduled, if they do not tolerate the taint
  - `PreferNoSchedule`
    - system will be try to avoid placing a pod on the node, not guaranteed
  - `NoExecute`
    - new pods will not be scheduled on the node
    - existing pods on the node will be evicted if the do not tolertae the taint

#### 확인

- describe & grep
  ```yaml
  k describe node node01 |grep -i taints
  Taints:             <none>
  ```

#### 제거

- `kubectl taint nodes node-name key=value:taint-effect-`
  ```bash
  kubectl taint nodes master/controlplane node-role.kubernetes.io/master:NoSchedule-
  ```

### Tolerations

#### 설정

```yaml
apiVersion:
kind: Pod
metadata:
    name: myapp-pod
spec:
    containers:
    - name: nginx-container
    - image: nginx
    tolerations:
    - key: "app"
        operator: "Equal"
        value: "blue"
        effect: "NoSchedule"
```

## PODs to Node

1. Node Selector
2. Node Affinity

### Node Selector

- Label nodes
- use `nodeSelector` with label
  ```yaml
  apiVersion:
  kind:
  metadata:
      name:
  spec:
      containers:
      - name:
          image:
      nodeSelector:
          size: Large
  ```

- limitations
  - only use single label
  - there should be complex constraints

### Node Affinity

- `nodeSelector` vs `affinity`
  - two are working equally, schedule pods to `Large` node.
  ![img-0](/imgs/cka/scheduler-0.png)

#### 설정

```yaml
apiVersion:
kind:
metadata:
  name:
spec:
  containers:
  - name:
    image:
  affinity:
    nodeAffinity:
      requireDuringSchedulingIgnoredDuringExecution:
        nodeSelectorTerms:
        - matchExpressions:
          - key:
            operator:
            values:
```

- `operator`
  - `In`
      - guarantee to schedule in values
  - `NotIn`
      - guarantee not to schedule in values
  - `Exists`
      - given key exists or not

#### Node Affinity Types

- available
  - `requireDuringSchedulingIgnoredDuringExecution`
  - `preferredDuringSchedulingIgnoredDuringExecution`
- planned
  - `requireDuringSchedulingRequiredDuringExecution`
- two states in the life cycle of pod
  - `DuringScheduling`
    - `Required`
      - must be
    - `Preferred`
      - best to
  - `DuringExecution`
    - `Ignored`
      - if they are scheduled, will not impact them.
    - `Required`
      - pod will be evicted

## Resource Limits

### Resource Requests

#### 설정

```yaml
apiVersion:
kind:
metadata:
    name:
spec:
    containers:
    - name:
        image:
    resources:
        requests:
            memory: "1Gi"
            cpu: 1
```

- cpu:
  - 0.1 == 100mi
  - lower: 1mi
- memory
  ![img-1](/imgs/cka/scheduler-.1png)

### Resources Limits

#### 설정

```yaml
apiVersion:
kind:
metadata:
    name:
spec:
    containers:
    - name:
        image:
    resources:
        requests:
            memory: "1Gi"
            cpu: 1
        limits:
            memory: "2Gi"
            cpu: 2
```

- default
  - 1v CPU
  - 512Mi

## Daemon Sets

![img-2](/imgs/cka/scheduler-2.png)

- one copy of the pod is always present in all nodes in the cluster

### use case

- monitoring
  ![img-3](/imgs/cka/scheduler-3.png)
- kube-proxy
  ![img-4](/imgs/cka/scheduler-4.png)
- networking
  ![img-5](/imgs/cka/scheduler-5.png)

### 생성

- `DaemonSet` vs `ReplicaSet`
  ![img-6](/imgs/cka/scheduler-6.png)

- definition
  ```yaml
  apiVersion: apps/v1
  kind: DaemonSet
  metadata:
      name: monitoring-daemon
  spec:
      selector:
          matchLabels:
              app: monitoring-daemon
      template:
          metadata:
              labels:
                  app: monitoring-daemon
          spec:
              contaiers:
              - name: monitoring-agent
                  image: monitoring-agent
  ```

### 상태

- `kubectl get daemonsets`

## Static PODs

### Static PODs vs DaemonSets

![img-7](/imgs/cka/scheduler-7.png)

- static pod
  - created directly by kubelet

### Config

1. `--pod-manifeset-path=/etc/Kubernetes/manifest`
2. `--config=kubeconfig.yaml`
   ```yaml
   # kubeconfig.yaml
   staticPodPath: /etc/Kubernetes/manifest
   ```

#### 파일 위치

First idenity the kubelet config file:

```
root@controlplane:~# ps -aux | grep /usr/bin/kubelet
root      3668  0.0  1.5 1933476 63076 ?       Ssl  Mar13  16:18 /usr/bin/kubelet --bootstrap-kubeconfig=/etc/kubernetes/bootstrap-kubelet.conf --kubeconfig=/etc/kubernetes/kubelet.conf --config=/var/lib/kubelet/config.yaml --network-plugin=cni --pod-infra-container-image=k8s.gcr.io/pause:3.2
root      4879  0.0  0.0  11468  1040 pts/0    S+   00:06   0:00 grep --color=auto /usr/bin/kubelet
root@controlplane:~#
```

From the output we can see that the kubelet config file used is `/var/lib/kubelet/config.yaml`

Next, lookup the value assigned for `staticPodPath`:

```
root@controlplane:~# grep -i staticpod /var/lib/kubelet/config.yaml
staticPodPath: /etc/kubernetes/manifests
root@controlplane:~#
```

As you can see, the path configured is the `/etc/kubernetes/manifests` directory.

### 상태

- `kubectl get po -A`
  - `-controlplane` appended pods
  ```yaml
  > k get po -A
  NAMESPACE     NAME                                   READY   STATUS    RESTARTS   AGE
  kube-system   coredns-74ff55c5b-578df                1/1     Running   0          25m
  kube-system   coredns-74ff55c5b-nnjdp                1/1     Running   0          25m
  kube-system   etcd-controlplane                      1/1     Running   0          25m
  kube-system   kube-apiserver-controlplane            1/1     Running   0          25m
  kube-system   kube-controller-manager-controlplane   1/1     Running   0          25m
  kube-system   kube-flannel-ds-q7plw                  1/1     Running   0          24m
  kube-system   kube-flannel-ds-w5rnm                  1/1     Running   0          25m
  kube-system   kube-proxy-5dg8f                       1/1     Running   0          24m
  kube-system   kube-proxy-ld2qq                       1/1     Running   0          25m
  kube-system   kube-scheduler-controlplane            1/1     Running   0          25m
  ```
  - `grep -i controlplane`
  ```yaml
  > k get po -A |grep -i controlplane
  kube-system   etcd-controlplane                      1/1     Running   0          27m
  kube-system   kube-apiserver-controlplane            1/1     Running   0          27m
  kube-system   kube-controller-manager-controlplane   1/1     Running   0          27m
  kube-system   kube-scheduler-controlplane            1/1     Running   0          27m
  ```

### 삭제

First, let's identify the node in which the pod called **static-greenbox** is created. To do this, run:

```
root@controlplane:~# kubectlget pods --all-namespaces -o wide  | grep static-greenbox
defaultstatic-greenbox-node01                 1/1Running   0          19s     10.244.1.2   node01       <none>           <none>
root@controlplane:~#
```

From the result of this command, we can see that the pod is running on node01.

Next, SSH to node01 and identify the path configured for static pods in this node.

`Important`: The path need not be `/etc/kubernetes/manifests`.

Make sure to check the path configured in the kubelet configuration file.

```
root@controlplane:~# ssh node01
root@node01:~# ps -ef |  grep /usr/bin/kubelet
root       752   654  0 00:30 pts/0    00:00:00 grep --color=auto /usr/bin/kubelet
root     28567     1  0 00:22 ?        00:00:11 /usr/bin/kubelet --bootstrap-kubeconfig=/etc/kubernetes/bootstrap-kubelet.conf --kubeconfig=/etc/kubernetes/kubelet.conf --config=/var/lib/kubelet/config.yaml --network-plugin=cni --pod-infra-container-image=k8s.gcr.io/pause:3.2
root@node01:~# grep -i staticpod /var/lib/kubelet/config.yaml
staticPodPath: /etc/just-to-mess-with-you
root@node01:~#
```

Here the staticPodPath is `/etc/just-to-mess-with-you`

Navigate to this directory and delete the YAML file:

```
root@node01:/etc/just-to-mess-with-you# ls
greenbox.yaml
root@node01:/etc/just-to-mess-with-you# rm -rf greenbox.yaml
root@node01:/etc/just-to-mess-with-you#
```

Exit out of node01 using `CTRL + D` or type `exit`. You should return to the `controlplane` node. Check if the `static-greenbox` pod has been deleted:

```
root@controlplane:~# kubectl get pods --all-namespaces -o wide  | grep static-greenbox
root@controlplane:~#
```

## Multiple Schedulers

### 생성

- cli
  ![img-8](/imgs/cka/scheduler-8.png)
- kubeadm
  ![img-9](/imgs/cka/scheduler-9.png)
  - `--leader-elect`
    - used when multiple copies of the scheduler running on different master nodes
    - who will lead activity
  - `--scheduler-name`
    - name of scheduler
  ![img-10](/imgs/cka/scheduler-10.png)
  - `--lock-object-name`

### 사용
- definition
  ```yaml
  apiVersion: v1
  kind: Pod
  metadata:
      name: nginx
  spec:
      containers:
      - image: nginx
          name: nginx
      schedulerName: my-custom-scheduler
  ```

### 상태
- `kubectl get events`
- `kubectl logs my-custom-scheduler --n kube-system`
