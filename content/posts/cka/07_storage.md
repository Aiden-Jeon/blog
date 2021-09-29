---
title: (CKA) 07. Storage
comment: true
categories: [kubernetes]
tags: ["k8s", "cka"]
toc: true
date: 2021-05-23
author: Jongseob Jeon
---

CKA를 준비하면서 공부한 요약 내용입니다.
- [강의](https://www.udemy.com/course/certified-kubernetes-administrator-with-practice-tests/)
- [What is CKA?](https://www.cncf.io/certification/cka/)

## Docker Storage

### File system
![img-1](/imgs/cka/storage-1.png)

#### Layered architecture
- build
  - application 1
    ![img-2](/imgs/cka/storage-2.png)
  - application 2
    ![img-3](/imgs/cka/storage-3.png)
- run
  ![img-4](/imgs/cka/storage-4.png)
- copy-on-write
  ![img-5](/imgs/cka/storage-5.png)

#### Volumes
![img-6](/imgs/cka/storage-6.png)

#### Volume mounting
- create and run
  - create
    - `docker volume create data_volume`
  - run
    - `docker run -v data_volume:/var/lib/mysql mysql`
- create by run
  - `docker run -v data_volume2:/var/lib/mysql mysql`
  - automatically create data_volume

#### Bind mounting
- `docker run -v /data/mysql:/var/lib/mysql mysql`

#### Preferred
- old
  - `-v`
- new
  - `--mount`
  - `docker run --mount type=bind,source=/data/mysql,target=/var/lib/mysql mysql`

### Storage drivers
- AUFS
- ZFS
- BTFS
- Device Mapper
- Overlay
- Overlay2

### Volume Drivers
- Local
- Azure File Storage
- Convoy
- ...

## Volumes
- docker
  ![img-7](/imgs/cka/storage-7.png)
- k8s  
  ![img-8](/imgs/cka/storage-8.png)

### Volumes & Mounts
![img-9](/imgs/cka/storage-9.png)

- definition
  ```yaml
  apiVersion: v1
  kind: Pod
  metadat:
    name: random-number-generator
  spec:
    containers:
    - image: alpine
      name: alpine
      command: ["bin/sh", "-c"]
      args: ["shuf -i 0-100 -n 1 >> /opt/number.out;"]
      volumeMounts:
      - mountPath: /opt
        name: data-volume
    volumes:
    - name: data-volume
      hostPath:
        path: /data
        type: Directory
  ```

  - hostPath
    - → single node is possible
    - → not recommended in multi node
  - AWS
    ```yaml
      volumes:
      - name: data-volume
        awsElastricBlockStore:
          volumeID: <volume-id>
          fsType: ext4
    ```

## Persistent Volume

![img-10](/imgs/cka/storage-10.png)

### 생성
- definition
  ```yaml
  apiVersion: v1
  kind: PersistentVolume
  metadata:
    name: pv-vol1
  spec:
    persistentVolumeReclaimPolicy: Retain
    accessModes:
      - ReadWriteOnce
    capacity:
      storage: 1Gi
    hostPath:
      path: /tmp/data
  ```
  - `accessModes`
    - `ReadOnlyMany`
    - `ReadWriteOnce`
    - `ReadWriteMany`
  - `persistentVolumeReclaimPolicy`
    - `delete`
    - `recycle`
    - `retain`

### 상태
- `kubectl get persistentvolume`
- `kubectl get pv`

## Persistent Volume Claims
- Persistent Volume
  - administrator crates
- Persistent Volume Claims
  - users create to use persistent volume

### 생성

- definition
  ```yaml
  apiVersion: v1
  kind: PersistentVolumeClaim
  metadata:
    name: myclaim
  spec:
    accessModes:
      - ReadWriteOnce
    resources:
      requests:
        storage: 500Mi
  ```

### 상태

- `kubectl get persistentvolumeclaim`
- `kubectl get pvc`

### 삭제

- `kubectl delete persistentvolumne myclaim`

## Pod with PVC

### 생성

- definition w\ hostPath
  ```yaml
  apiVersion: v1
  kind: Pod
  metadata:
    name: webapp
  spec:
    containers:
    - name: event-simulator
      image: kodekloud/event-simulator
      env:
      - name: LOG_HANDLERS
        value: file
      volumeMounts:
      - mountPath: /log
        name: log-volume
    volumes:
    - name: log-volume
      hostPath:
        ## directory location on host
        path: /var/log/webapp
        ## this field is optional
        type: Directory
  ```
- definition w\ PVC
  ```yaml
  apiVersion: v1
  kind: Pod
  metadata:
    name: webapp
  spec:
    containers:
    - name: event-simulator
      image: kodekloud/event-simulator
      env:
      - name: LOG_HANDLERS
        value: file
      volumeMounts:
      - mountPath: /log
        name: log-volume
  
    volumes:
    - name: log-volume
      persistentVolumeClaim:
        claimName: claim-log-1
  ```

## Storage Class

### 상태
- `kubectl get storageclass`
- `kubectl get sc`