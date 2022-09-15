---
title: kubernetes에서 ghcr package pull 하기
categories: [mlops]
tags: ["k8s"]
toc: true
date: 2021-03-03
author: Jongseob Jeon
---



kubernetes 에서 사용하는 image는 기본적으로 dockerhub에서 pull 된다. 기본적으로 Local에 있는 패키지는 사용하지 않는다. 
회사에서는 도커 이미지를 dockerhub와 같은 곳에  public 하게 공유할 수가 없다. 
그래서 github 에서 제공하는 (GHCR) GitHub Container Registry for Docker images 을 이용해서 도커 이미지를 관리하고 있다.

GHCR 을 이용하기 위해서는 github access key 가 필요하다. 우선 github access token을 발급 받아 보자.

## 1. ghcr 을 위한 github access token 발급 받기
우선 github setting에서 deplotver settings / personal access tokens 로 들어간다.

![그림-1](/imgs/k8s/ghcr/0.png)
여기서 generate new token을 누른다.

![그림-2](/imgs/k8s/ghcr/1.png)
토큰 이름으로 ghcr-token을 입력하고 packages와 관련된 권한을 주고 token을 생성한다.

![그림-3](/imgs/k8s/ghcr/2.png)
다음과 같이 토큰이 생성된다. 이제 이 토큰을 잘 저장해둔다.


## 2. kubernetes secret
다음으로 kubernetes 에 secret 을 생성해준다. 
```bash
kubectl -n <k8s-namespace> create secret docker-registry <k8s-docker-registry-secret-name> \
    --docker-server=ghcr.io \
    --docker-username=<github-username> \
    --docker-password=<github-personal-access-token> \
    --docker-email=<email-address>
```
- `k8s-docker-registry-secret-name`: secret 이름 
- `github-username`: github 유저 네임
- `github-personal-access-token`: 발급받은 토큰
- `email-address`: github 이메일


예를 들면 아래와 같이 사용할 수 있다.

```bash
> kubectl -n default create secret docker-registry test-ghcr \
    --docker-server=ghcr.io \
    --docker-username=aiden-jeon \
    --docker-password=<github-personal-access-token> \
    --docker-email=ells2124@gmail.com
```
정상적으로 실행될 경우 아래와 같이 출력된다.
```bash
secret/test-ghcr created
```

## 3. yaml 파일 작성
이제 deployment를 위한 yaml 파일을 작성한다.
```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: test-deploy
spec:
  selector:
    matchLabels:
      type: app
      service: test-deploy
  template:
    metadata:
      labels:
        type: app
        service: test-deploy
    spec:
      containers:
      - name: test-deploy
        image: ghcr.io/{user-name}/{package-name}:{package-tag}
        imagePullPolicy: Always
        ports:
        - containerPort: 80
      imagePullSecrets:
        - name: test-ghcr
```
image에 ghcr 의 도커 이미지를 입력해주면 된다.
그리고 imagePullSecrets에 좀 전에 생성한 secret 의 이름을 입력하면 된다.
