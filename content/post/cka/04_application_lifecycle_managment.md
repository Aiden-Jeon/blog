---
title: (CKA) 04. Application Lifecycle Management
categories: [kubernetes]
tags: ["k8s", "cka"]
toc:
  auto: true
date: 2021-05-18
author: Jongseob Jeon
---

CKA를 준비하면서 공부한 요약 내용입니다.
- [강의](https://www.udemy.com/course/certified-kubernetes-administrator-with-practice-tests/)
- [What is CKA?](https://www.cncf.io/certification/cka/)

## Rollout and Versioning
![img-1](/imgs/cka/05-1.png)

### Rollout Command
- `kubectl rollout status deployment/myapp-deployment`
- `kubectl rollout history deployment/myapp-deployment`

### Deployment Strategy

#### Recreate

![img-2](/imgs/cka/05-2.png)
- destruction 5
- create 5

![img-3](/imgs/cka/05-3.png)

#### Rolling Update
→ default strategy

![img-4](/imgs/cka/05-4.png)
- destruction and create one by one

### Revision

- `kubectl apply -f deployment.yaml`
- `kubectl set image deployment/myapp-deployment nginx=nginx:1.9.1`

### Upgrades

![img-5](/imgs/cka/05-5.png)
- make new replicaset when upgrade deploy

### Rollback

![img-6](/imgs/cka/05-6.png)
- `kubectl rollout undo deployment/myapp-deployment`
- before rollback vs after rollback
  ![img-7](/imgs/cka/05-7.png)

## Commands and Arguments

- docker
  ```docker
  From Ubuntu
  Entrypoint ["sleep"]
  cmd ["5"]
  ```

- definition
  ```yaml
  apiVersion: v1
  kind: Pod
  metadata:
    name: ubuntu-sleeper-pod
  spec:
    containers:
      - name: ubuntu-sleeper
        image: ubuntu-sleeper
        command: ["sleep2.0"]
        args: ["10"]
  ```

## Environment

- plain Key Value
  ```yaml
  env:
    - name: APP_COLOR
      value: pink
  ```

- configMap
  ```yaml
  env:
    - name: APP_COLOR
      valueFrom:
        configMapKeyRef:
  ```

- Secrets
  ```yaml
  env:
    - name: APP_COLOR
      valueFrom:
        secretKeyRef:
  ```

## ConfigMaps

- create ConfigMap
- Inject into pod

### create

- imperative
  ```bash
  kubectl create configmap\\
    app-config --from-literal=APP_COLOR=blue \\
              --from-literal=APP_MODE=prod
  ```
  - `kubectl create configmap app-config --from-file=<path-to-file>`

- declartive
  ```yaml
  apiVersion: v1
  kind: ConfigMap
  metadata:
    name: app-config
  data:
    APP_COLOR: blue
    APP_MODE: prod
  ```

### view

- `kubectl get configmaps`
- `kubectl describe configmaps`

### ConfigMap in Pods

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: simple-webapp-color
spec:
  containers:
    - name: simple-webapp-color
      image: simple-webapp-color
      envFrom:
        - configMapRef:
            name: app-config
```

## Secrets

- create secret
- inject into pod

### create

- imperative
  ```bash
  kubectl create secret generic\\
    <secret-name> --from-literal=<key>=<value>
  ```
  - `kubectl create secret <secret-name> --from-file=<path-to-file>`

- declartive
  ```yaml
  apiVersion: v1
  kind: Secret
  metadata:
    name: app-secret
  data:
    DB_Host: mysql
    DB_User: root
    DB_Passwird: paswrd
  ```
  - data → encoded format for safe
  - `echo -n 'mysql' | base64`

### view

- `kubectl get secrets`
- `kubectl describe secrets`

#### decode

- `echo -n 'abalksdfas=' | base54 --decode`

### Secrets in Pods

- defintion
  ```yaml
  apiVersion: v1
  kind: Pod
  metadata:
    name: simple-webapp-color
  spec:
    containers:
      - name: simple-webapp-color
        image: simple-webapp-color
        envFrom:
          - configMapRef:
              name: app-secret
  ```

- env
  ```yaml
  envFrom:
    - secretRef:
        name: app-config
  ```

- single env

  ```yaml
  env:
    - name: DB_Password
      valueFrom:
        secretKeyRef:
          name: app-secret
          key: DB_Password
  ```

- volume
  ```yaml
  volumes:
  - name: app-secret-volumne
    secret:
      secretName: app-secret
  ```

  - inside the container
    - list
      ```bash
      ls /opt/app-secret-volumes
      ```
    - content
      ```bash
      cat /opt/app-secret-volumes/DB_Password
      ```

## InitContainers

In multi-container pod, want to run a process that runs to completion in a container

- initContainer
  ```yaml
  apiVersion: v1
  kind: Pod
  metadata:
    name: myapp-pod
    labels:
      app: myapp
  spec:
    containers:
    - name: myapp-container
      image: busybox:1.28
      command: ['sh', '-c', 'echo The app is running! && sleep 3600']
    initContainers:
    - name: init-myservice
      image: busybox
      command: ['sh', '-c', 'git clone <some-repository-that-will-be-used-by-application> ; done;']
  ```

- multiple initContainers
  ```yaml
  apiVersion: v1
  kind: Pod
  metadata:
    name: myapp-pod
    labels:
      app: myapp
  spec:
    containers:
    - name: myapp-container
      image: busybox:1.28
      command: ['sh', '-c', 'echo The app is running! && sleep 3600']
    initContainers:
    - name: init-myservice
      image: busybox:1.28
      command: ['sh', '-c', 'until nslookup myservice; do echo waiting for myservice; sleep 2; done;']
    - name: init-mydb
      image: busybox:1.28
      command: ['sh', '-c', 'until nslookup mydb; do echo waiting for mydb; sleep 2; done;']
  ```
