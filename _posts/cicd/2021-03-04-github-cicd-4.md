---
title: (CI/CD) argocd를 이용한 cd
comment: true
categories: [cicd]
toc: true
toc_sticky: true
---

**CI/CD Contents 순서**
1. [sphinx-autoapi 를 이용한 자동 api 문서 생성하기](https://aiden-jeon.github.io/cicd/sphinx-autoapi)
2. [github action을 이용한 ci](https://aiden-jeon.github.io/cicd/github-cicd-1)
3. [ghcr을 이용한 kubernetes deployment 만들기](https://aiden-jeon.github.io/cicd/github-cicd-2)
4. [helm을 이용한 deployment chart 만들기](https://aiden-jeon.github.io/cicd/github-cicd-3)
5. [argocd를 이용한 cd](https://aiden-jeon.github.io/cicd/github-cicd-4)

---



이번 포스트에서는 argocd를 이용해 helm chart를 cd(continuous delivery) 하는 법에 대해서 알아보겠습니다.

## 1. argocd 설치
argocd 설치는 [tutorial](https://argoproj.github.io/argo-cd/getting_started/)을 따라했습니다.

```bash
# 1. argocd namespace
kubectl create namespace argocd
kubectl apply -n argocd -f https://raw.githubusercontent.com/argoproj/argo-cd/stable/manifests/install.yaml

# access argo cd api server
kubectl patch svc argocd-server -n argocd -p '{"spec": {"type": "LoadBalancer"}}'

#port forward
kubectl port-forward svc/argocd-server -n argocd 8080:443

argocd login localhost:8080
```
기본 아이디는 admin입니다. 비밀번호는 아래 명령어로 확인할 수 잇습니다.
```bash
kubectl get pods -n argocd -l app.kubernetes.io/name=argocd-server -o name | cut -d'/' -f 2
```

로그인에 성공하면 https://localhost:8080 으로 들어갑니다.

처음 나오는 화면은 다음 처럼 아무런 app이 없습니다.
![img](/assets/imgs/github/cicd-8.png)

NEW APP 을 눌러 추가하겠습니다.
![img](/assets/imgs/github/cicd-9.png)
![img](/assets/imgs/github/cicd-10.png)

위와 같이 config를 입력하고 create를 해줍니다. 완성되면 아래와 같이 app이 생성됩니다.
![img](/assets/imgs/github/cicd-11.png)

app에 클릭해서 들어가면 다음과 같이 나옵니다. 여기서 SYNC 버튼을 눌러줍니다
![img](/assets/imgs/github/cicd-12.png)

SYNCHRONIZE 버튼을 누릅니다.
![img](/assets/imgs/github/cicd-13.png)

다음은 sync가 완료된 화면입니다.
![img](/assets/imgs/github/cicd-14.png)

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

다음 명령어로 접속할 ip와 포트를 얻습니다.
```bash
> minikube service -n sphinx-doc --url sphinx-api-doc-sphinx-doc

http://192.168.64.2:31058
```

정상적으로 실행이 되었는지 확인합니다.
