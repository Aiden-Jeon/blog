---
title: argocd를 이용한 cd
categories: [cicd]
tags: ["argocd"]
toc: true
date: 2021-03-04
author: Jongseob Jeon
---

**CI/CD Contents 순서**
1. [sphinx-autoapi 를 이용한 자동 api 문서 생성하기]({{< relref "posts/mlops/cicd/sphinx_autoapi" >}})
2. [github action을 이용한 ci]({{< relref "posts/mlops/cicd/github_action_ci" >}})
3. [ghcr을 이용한 kubernetes deployment 만들기]({{< relref "posts/mlops/cicd/ghcr_k8s_deploy" >}})
4. [helm을 이용한 deployment chart 만들기]({{< relref "posts/mlops/cicd/helm_deployment_chart" >}})
5. [argocd를 이용한 cd]({{< relref "posts/mlops/cicd/argocd_cd" >}})

---



이번 포스트에서는 argocd를 이용해 helm chart를 cd(continuous delivery) 하는 법에 대해서 알아보겠습니다.

## 1. 설치
argocd 설치는 [tutorial](https://argoproj.github.io/argo-cd/getting_started/)을 따라했습니다.


## 2. 실행
```bash
# 1. argocd namespace
kubectl create namespace argocd
kubectl apply -n argocd -f https://raw.githubusercontent.com/argoproj/argo-cd/stable/manifests/install.yaml

# access argo cd api server
kubectl patch svc argocd-server -n argocd -p '{"spec": {"type": "LoadBalancer"}}'

#port forward
kubectl port-forward svc/argocd-server -n argocd 8080:80
```

argocd 를 실행시켰으면 https://localhost:8080 으로 들어갑니다.  
기본으로 설정되어 있는 아이디는 admin이며 비밀번호는 아래 명령어로 확인할 수 있습니다.

v1.8.0 이하는 아래 명령어로 확인할 수 있습니다.
```bash
kubectl get pods -n argocd -l app.kubernetes.io/name=argocd-server -o name | cut -d'/' -f 2
```

v1.9.0 이상은 아래 명령어로 확인할 수 있습니다.
```bash
kubectl -n argocd get secret argocd-initial-admin-secret -o jsonpath="{.data.password}" | base64 -d
```

## 3. APP 생성
처음 나오는 화면은 다음과 같이 아무런 app이 없습니다.

![그림-1](/imgs/github/cicd-8.png)

NEW APP 을 눌러 추가하겠습니다.

![그림-2](/imgs/github/cicd-9.png)
![그림-3](/imgs/github/cicd-10.png)

그림-3과 같이 config를 입력하고 create를 해줍니다. 완성되면 아래와 같이 app이 생성됩니다.

![그림-4](/imgs/github/cicd-11.png)

app에 클릭해서 들어가면 다음과 같이 나옵니다. 여기서 SYNC 버튼을 눌러줍니다

![그림-5](/imgs/github/cicd-12.png)

SYNCHRONIZE 버튼을 누릅니다.

![그림-6](/imgs/github/cicd-13.png)

다음은 sync가 완료된 화면입니다.

![그림-7](/imgs/github/cicd-14.png)

command창에서 정상적으로 떴는지 확인해봅니다.
```bash
> kubectl get all -n sphinx-doc
NAME                                             READY   STATUS    RESTARTS   AGE
pod/sphinx-api-doc-sphinx-doc-5576ff677b-d4xzl   1/1     Running   0          89s

NAME                                TYPE       CLUSTER-IP      EXTERNAL-IP   PORT(S)        AGE
service/sphinx-api-doc-sphinx-doc   NodePort   10.105.22.158   <none>        80:31058/TCP   89s

NAME                                        READY   UP-TO-DATE   AVAILABLE   AGE
deployment.apps/sphinx-api-doc-sphinx-doc   1/1     1            1           89s

NAME                                                   DESIRED   CURRENT   READY   AGE
replicaset.apps/sphinx-api-doc-sphinx-doc-5576ff677b   1         1         1       89s
```


## 4. 확인
다음 명령어로 접속할 ip와 포트를 얻습니다.
```bash
> minikube service -n sphinx-doc --url sphinx-api-doc-sphinx-doc

http://192.168.64.2:31058
```

정상적으로 실행이 되었는지 확인합니다.
