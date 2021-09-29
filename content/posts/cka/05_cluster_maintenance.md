---
title: (CKA) 05. Cluster Maintenance
comment:   
    enable: true
categories: [kubernetes]
tags: ["k8s", "cka"]
toc: true
date: 2021-05-22
author: Jongseob Jeon
---

CKA를 준비하면서 공부한 요약 내용입니다.
- [강의](https://www.udemy.com/course/certified-kubernetes-administrator-with-practice-tests/)
- [What is CKA?](https://www.cncf.io/certification/cka/)

## Operating System Upgrade

- upgrading base software
- applying security patches

### Pod Eviction Timeout
- waiting pod healthy
  - `pod-eviction-timeout=5m0s`

### Drain
Do not use this node, and all pods are out.

- `kubectl drain node01`
- `kubectl drain node01 --ignore-daemonsets`
  ```bash
  root@controlplane:~# kubectl drain node01 --ignore-daemonsets
  node/node01 cordoned
  WARNING: ignoring DaemonSet-managed Pods: kube-system/kube-flannel-ds-x6dgs, kube-system/kube-proxy-jfmxw
  evicting pod default/blue-746c87566d-wn2qg
  evicting pod default/blue-746c87566d-kt266
  evicting pod default/blue-746c87566d-9w9zq
  pod/blue-746c87566d-kt266 evicted
  pod/blue-746c87566d-wn2qg evicted
  pod/blue-746c87566d-9w9zq evicted
  node/node01 evicted
  ```
- `kubectl drain node01 --ignore-daemonsets --force`  
  if a pod in node has no replicaset, use `--force` option, but a pod will be lost.
  ```bash
  root@controlplane:~# kubectl drain node01 --ignore-daemonsets --force
  node/node01 already cordoned
  WARNING: deleting Pods not managed by ReplicationController, ReplicaSet, Job, DaemonSet or StatefulSet: default/hr-app; ignoring DaemonSet-managed Pods: kube-system/kube-flannel-ds-x6dgs, kube-system/kube-proxy-jfmxw
  evicting pod default/hr-app
  pod/hr-app evicted
  node/node01 evicted
  ```

### Cordon
Do not use this node, but exist pods will be running.

- `kubectl cordon node01`
  - This will ensure that no new pods are scheduled on this node.
  - The existing pods will not be affected by this operation.

### Uncordon
Can use this node.

- `kubectl uncordon node01`

## Cluster Upgrade Process

### Supported versions

![img-1](/imgs/cka/cluster_maintenance-1.png)

#### gcp

![img-2](/imgs/cka/cluster_maintenance-2.png)

#### kubeadm

- `kubectl upgrade plan`
  ```bash
  root@controlplane:~## kubeadm upgrade plan
  [upgrade/config] Making sure the configuration is correct:
  [upgrade/config] Reading configuration from the cluster...
  [upgrade/config] FYI: You can look at this config file with 'kubectl -n kube-system get cm kubeadm-config -oyaml'
  [preflight] Running pre-flight checks.
  [upgrade] Running cluster health checks
  [upgrade] Fetching available versions to upgrade to
  [upgrade/versions] Cluster version: v1.19.0
  [upgrade/versions] kubeadm version: v1.19.0
  I0522 08:54:38.911002   21647 version.go:252] remote version is much newer: v1.21.1; falling back to: stable-1.19
  [upgrade/versions] Latest stable version: v1.19.11
  [upgrade/versions] Latest stable version: v1.19.11
  [upgrade/versions] Latest version in the v1.19 series: v1.19.11
  [upgrade/versions] Latest version in the v1.19 series: v1.19.11
  
  Components that must be upgraded manually after you have upgraded the control plane with 'kubeadm upgrade apply':
  COMPONENT   CURRENT       AVAILABLE
  kubelet     2 x v1.19.0   v1.19.11
  
  Upgrade to the latest version in the v1.19 series:
  
  COMPONENT                 CURRENT   AVAILABLE
  kube-apiserver            v1.19.0   v1.19.11
  kube-controller-manager   v1.19.0   v1.19.11
  kube-scheduler            v1.19.0   v1.19.11
  kube-proxy                v1.19.0   v1.19.11
  CoreDNS                   1.7.0     1.7.0
  etcd                      3.4.9-1   3.4.9-1
  
  You can now apply the upgrade by executing the following command:
  
          kubeadm upgrade apply v1.19.11
  
  Note: Before you can perform this upgrade, you have to update kubeadm to v1.19.11.
  
  _____________________________________________________________________
  
  The table below shows the current state of component configs as understood by this version of kubeadm.
  Configs that have a "yes" mark in the "MANUAL UPGRADE REQUIRED" column require manual config upgrade or
  resetting to kubeadm defaults before a successful upgrade can be performed. The version to manually
  upgrade to is denoted in the "PREFERRED VERSION" column.
  
  API GROUP                 CURRENT VERSION   PREFERRED VERSION   MANUAL UPGRADE REQUIRED
  kubeproxy.config.k8s.io   v1alpha1          v1alpha1            no
  kubelet.config.k8s.io     v1beta1           v1beta1             no
  _____________________________________________________________________
  ```

- `kubectl upgrade apply`

### Kubeadm

#### Strategy
- strategy-1
  - all node down and up
- strategy-2
  - upgrade node one by one
- strategy-3
  - add new version node and remove old node
  - easy in cluster

#### Procedure
- master node
  - ver 1
    - `apt-get upgrade -y kuebadm=1.12.0-00`
    - `kubeadm upgrade apply v1.12.0`
    - `apt-get upgrade -y kubelet=1.12.0-00`
    - `systemctl restart kubelet`
  - ver 2
    - `apt update`
    - `apt install kubeadm=1.20.0-00`
    - `kubeadm upgrade apply v1.20.0`
    - `apt install kubelet=1.20.0-00`
    - `systemctl restart kubelet`
- worker node
  - ver 1
    - `kubectl drain node01`
    - `apt-get upgrade -y kuebadm=1.12.0-00`
    - `apt-get upgrade -y kubelet=1.12.0-00`
    - `kubeadm upgrade node config --kubelet-version v1.12.0`
    - `systemctl restart kubelet`
    - `kubectl uncordon node01`
  - ver 2
    - `apt update`
    - `apt install kubeadm=1.20.0-00`
    - `kubeadm upgrade node`
    - `apt install kubelet=1.20.0-00`
    - `systemctl restart kubelet`

## Backup and Restore

### Backup Candidates
- Resource Configuration
- ETCD Cluster
- Persistent Volumes

#### Resource Configuration

- kube-apiserver
  - `kubectl get all -A -o yaml > all-deploy-svc.yaml`
  - too many resource to do
  - → opensource like VELERO

#### ETCD
- method 1
  - `ExecStart= ... \\ --data-dir=/var/lib/etct`
- method 2
  - `ETCDCTL_API=3 etcdctl snapshot save snapshot.db`
  - `servcie kube-apiserver stop`
  - `ETCDCTL_API=3 etcdctl snapshot --data-dir /var/lib/etcd-from-backup snapshot restore snapshot.db`
  - `systemctl daemon-reload`
  - `service etcd restart`
  - `service kube-apiserver start`

- backup practice
  ```bash
  ETCDCTL_API=3 etcdctl --endpoints=https://[127.0.0.1]:2379 \\
  --cacert=/etc/kubernetes/pki/etcd/ca.crt \\
  --cert=/etc/kubernetes/pki/etcd/server.crt \\
  --key=/etc/kubernetes/pki/etcd/server.key \\
  snapshot save /opt/snapshot-pre-boot.db
  ```
- restore practice
  - Restore snapshot
    ```bash
    ETCDCTL_API=3 etcdctl  --data-dir /var/lib/etcd-from-backup \\
    snapshot restore /opt/snapshot-pre-boot.db
    ```
  - update the `/etc/kubernetes/manifests/etcd.yaml`
    ```bash
    volumes:
      - hostPath:
          path: /var/lib/etcd-from-backup
          type: DirectoryOrCreate
        name: etcd-data
    ```
