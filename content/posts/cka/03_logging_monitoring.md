---
title: (CKA) 03. Logging & Monitoring
comment:   
    enable: true
categories: [kubernetes]
tags: ["k8s", "cka"]
toc: true
date: 2021-05-17
author: Jongseob Jeon
---

CKA를 준비하면서 공부한 요약 내용입니다.
- [강의](https://www.udemy.com/course/certified-kubernetes-administrator-with-practice-tests/)
- [What is CKA?](https://www.cncf.io/certification/cka/)

## Metrics Server

- In-memory
- does not store

#### Install

- minikube
  - `minikube addons enable metrics-server`
- others
  - `git clone <https://github.con/kubernetes-incubator/metrics-server`>
  - `kubectl create -f deploy/1.8+`

#### View

- `kubeclt top node`
- `kubeclt top pod`

## Managing Application Logs

### Logs - k8s

#### 로그 보기 w\ defintion

- `kubectl create -f sample.yaml`
- `kubectl logs -f sample.yaml`

#### 여러 container

- `kubectl logs -f sample.yaml specific-container`
