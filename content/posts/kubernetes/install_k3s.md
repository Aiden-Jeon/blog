---
title: k3s Installation
categories: [kubernetes]
tags: ["k8s", "k3s"]
toc: true
date: 2021-10-01
author: Jongseob Jeon
weight: 99
---

**Kubeflow 설치 시리즈**
1. [Pre-Requisites for k3s Setup]({{< relref "posts/kubernetes/k3s_prerequisite" >}})
2. [k3s Installation]({{< relref "posts/kubernetes/install_k3s" >}})
3. [Kubeflow Installation]({{< relref "posts/kubernetes/install_kubeflow" >}})

---

이번 포스트에서는 서버에 k3s를 설치하는 과정에 대해서 설명합니다.  
필요한 [준비 과정]({{< relref "posts/kubernetes/k3s_prerequisite" >}})을 확인 후 진행해 주세요.  

---

# k3s Installation
1. k3s 설치  
   k3s는 gpu를 사용하기 위해서 도커를 백엔드로 사용하겠습니다.  
   docker 백엔드는 `--docker`를 추가하면 됩니다.
   ```bash
   curl -sfL https://get.k3s.io | sh -s - server --disable traefik --disable servicelb --disable local-storage --docker
   ```
2. k3s config 확인
   k3s를 설치후 k3s config를 확인합니다
   ```bash
   cat /etc/rancher/k3s/k3s.yaml
   ```
3. k3s config를 kubeconfig로 복사
   ```bash
   mkdir .kube
   sudo cp /etc/rancher/k3s/k3s.yaml .kube/config
   sudo chown mrx:mrx .kube/config
   ```
4. local로 kube config 옮기기  
   local에서 `~/.kube/config`로 설정합니다.  
   multi-context인 경우 multi-context에 맞게 설정합니다.  
   multi-context는 `kubectl ctx` 를 통해 쉽게 변경할 수 있습니다.
5. local에서 서버 확인하기
    ```bash
    kubectl get nodes
    ```

# Kubernetes Setup
k3s를 설치 후에는 서버로 사용할 수 있도록 써드파티들을 설치해주어야 합니다.  
위의 k3s installation 과정이 끝났다면 로컬에서 서버를 `kubectl` 을 통해 관리할 수 있습니다.  
아래 과정은 모두 local에서 이루어집니다. (`kubectl`의 context를 잘 확인 후 진행하시기 바랍니다.)

## 1. Nvidia Device Plugin
```bash
helm repo add nvdp https://nvidia.github.io/k8s-device-plugin
helm repo update
helm install \
    --version=0.9.0 \
    --generate-name \
    nvdp/nvidia-device-plugin
```

## 2. Ingress
```bash
helm repo add ingress-nginx https://kubernetes.github.io/ingress-nginx
helm repo update
helm install ingress-nginx/ingress-nginx -g -n ingress-nginx --create-namespace --set controller.service.type='NodePort' --set controller.service.nodePorts.http=32080 --set controller.service.nodePorts.https=32443
```

## 3. Longhorn
Longhorn은 k3s에서 StorageClass를 생성을 해줍니다.

### On Local
1. 설치
    ```bash
    helm repo add longhorn https://charts.longhorn.io
    helm repo update
    helm install longhorn longhorn/longhorn --namespace longhorn-system --set csi.kubeletRootDir=/var/lib/kubelet --create-namespace
    ```
2. VPN IP 설정하기
   ```bash
   sudo sh -c "echo '<VPN_IP> longhorn.k3s.cluster.local' >> /etc/hosts"
   ```
3. auth & ingress 생성
   ```bash
    USER=admin; PASSWORD=adminadmin; echo "${USER}:$(openssl passwd -stdin -apr1 <<< ${PASSWORD})" >> auth
    kubectl -n longhorn-system create secret generic basic-auth --from-file=auth
    cat <<EOF | kubectl apply -f -
    apiVersion: networking.k8s.io/v1
    kind: Ingress
    metadata:
      name: longhorn-ingress
      namespace: longhorn-system
      annotations:
        ingress.kubernetes.io/rewrite-target: /
        kubernetes.io/ingress.class: "nginx"
        # type of authentication
        nginx.ingress.kubernetes.io/auth-type: basic
        # prevent the controller from redirecting (308) to HTTPS
        nginx.ingress.kubernetes.io/ssl-redirect: 'false'
        # name of the secret that contains the user/password definitions
        nginx.ingress.kubernetes.io/auth-secret: basic-auth
        # message to display with an appropriate context why the authentication is required
        nginx.ingress.kubernetes.io/auth-realm: 'Authentication Required '
        # custom max body size for file uploading like backing image uploading
        nginx.ingress.kubernetes.io/proxy-body-size: 10000m
    spec:
      rules:
      - host: longhorn.k3s.cluster.local
        http:
          paths:
          - pathType: Prefix
            path: "/"
            backend:
              service:
                name: longhorn-frontend
                port:
                  number: 80
    EOF
   ```
### On Server
로컬에서 확인할 수 있는 reverse-proxy를 설정해주는 작업입니다.
이 작업은 서버에서 이루어집니다.

1. reverse-proxy 폴더 생성
   ```bash
    mkdir reverse-proxy
    cd reverse-proxy
    ```
2. reverse-proxy 폴더 아래 다음 파일들 생성 합니다.    
    생성할 파일은 총 3개 입니다.      
    - `Dockerfile`
        ```bash
        FROM nginx
        COPY ./longhorn.conf /etc/nginx/conf.d/longhorn.conf
        COPY ./kubeflow.conf /etc/nginx/conf.d/kubeflow.conf
        ```
    - `longhorn.conf`
        ```bash
        server {
            listen 80;
        
            server_name longhorn.k3s.cluster.local;
            location / {
                proxy_redirect off;
                proxy_pass_header Server;
                proxy_set_header Host $http_host;
                proxy_set_header X-Real-IP $remote_addr;
                proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
                proxy_set_header X-Forwarded-Proto $scheme;
                proxy_set_header X-NginX-Proxy true;
                proxy_set_header X-Scheme $scheme;
                proxy_pass http://localhost:32080;
            }
        }
        ```
    - `kubeflow.conf`
        ```bash
        server {
            listen 80;
            server_name kubeflow.k3s.cluster.local;
            location / {
                proxy_redirect off;
                proxy_pass_header Server;
                proxy_set_header Host $http_host;
                proxy_set_header X-Real-IP $remote_addr;
                proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
                proxy_set_header X-Forwarded-Proto $scheme;
                proxy_set_header X-NginX-Proxy true;
                proxy_set_header X-Scheme $scheme;
                proxy_pass http://localhost:30807;
                proxy_http_version 1.1;
                proxy_set_header Upgrade $http_upgrade;
                proxy_set_header Connection "upgrade";
                add_header X-Frame-Options SAMEORIGIN;
            }
        }
        ```
3. reverse proxy docker를 빌드합니다.
    ```bash
    docker build -t reverse-proxy .
    ```
4. 빌드한 docker image를 실행합니다.
    ```bash
    docker run -d --net host reverse-proxy
    ```

### On local
다음 주소로 접속이 되는지 확인합니다.  
http://longhorn.k3s.cluster.local/  

이 때 기본 ID/PWD는 다음과 같습니다.
- id: admin
- pw: adminadmin

로그인이 되면 다음과 같은 화면이 나옵니다.
![longhorn](/imgs/k3s/longhorn.png)
