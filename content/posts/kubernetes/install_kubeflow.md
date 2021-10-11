---
title: Kubeflow Installation
categories: [kubernetes]
tags: ["k8s", "k3s"]
toc: true
date: 2021-10-01
author: Jongseob Jeon
---

**Kubeflow 설치 시리즈**
1. [Pre-Requisites for k3s Setup]({{< relref "posts/kubernetes/k3s_prerequisite" >}})
2. [k3s Installation]({{< relref "posts/kubernetes/install_k3s" >}})
3. [Kubeflow Installation]({{< relref "posts/kubernetes/install_kubeflow" >}})

---

이번 포스트에서는 서버에 kubeflow를 설치하는 과정에 대해서 설명합니다.  
필요한 준비 과정을 확인 후 진행해 주세요.  
1. [Pre-Requisites for k3s Setup]({{< relref "posts/kubernetes/k3s_prerequisite" >}})
2. [k3s Installation]({{< relref "posts/kubernetes/install_k3s" >}})

---

이번 포스트에서 사용하는 kubeflow의 버전은 `1.3.0` 입니다.  
버전이 바뀔 경우 해당 설치 과정과 달라질 수 있습니다.  
자세한 내용은 [링크](https://github.com/kubeflow/manifests/tree/v1.3.0)를 확인해 주세요.


# Kustomize
Kubeflow를 설치하기 위해서는 kustmoize와 kubeflow manifest가 필요합니다.  

1. Clone kubeflow/manifest   
    Clone 받는 위치는 편한 곳에 하면 됩니다.
    ```bash
    git clone https://github.com/kubeflow/manifests
    ```

2. kustomize 다운로드  
    kubelfow manifest를 지원하는 kustomize를 다운 받습니다.
    포스트를 작성하는 기준으로는 3.2.0을 지원합니다.
    > kustomize (version 3.2.0)  
    > ⚠️ Kubeflow 1.3.0 is not compatible with the latest versions of of kustomize 4.x. This is due to changes in the order resources are sorted and printed. Please see kubernetes-sigs/kustomize#3794 and kubeflow/manifests#1797. We know this is not ideal and are working with the upstream kustomize team to add support for the latest versions of kustomize as soon as we can.

    [https://github.com/kubernetes-sigs/kustomize/releases/tag/v3.2.0](https://github.com/kubernetes-sigs/kustomize/releases/tag/v3.2.0)
    
3. kubeflow/manifest 경로로 이동
    다운받은 kustomize를 `kubeflow/manifest`로 옮겨줍니다.  
    이 때 cli에서 편하게 사용하기 위해서 이름을 `kustomize` 로 수정하면 좋습니다.
    ```bash
    mv ~/Downloads/kustomize_3.2.0_darwin_amd64 manifests/kustomize
    ```

4. 실행 권한 주기  
    옮긴 kustomize에 실행 권한을 줍니다.  
    ```bash
    cd manifests
    chmod a+x kustomize
    ```

5. Checkout to Version 1.3.0  
   manifest를 1.3.0 버전으로 맞춰 줍니다.
   ```bash
   git checkout v1.3.0
   ```

# Kubeflow Manifest 설치하기
Kubeflow를 설치하기 위한 Manifest를 설치합니다.
1. Cert-Manager
    ```bash
    kustomize build common/cert-manager/cert-manager-kube-system-resources/base | kubectl apply -f -
    kustomize build common/cert-manager/cert-manager-crds/base | kubectl apply -f -
    kustomize build common/cert-manager/cert-manager/overlays/self-signed | kubectl apply -f -
    ```
2. Istio
    ```bash
    kustomize build common/istio-1-9-0/istio-crds/base | kubectl apply -f -
    kustomize build common/istio-1-9-0/istio-namespace/base | kubectl apply -f -
    kustomize build common/istio-1-9-0/istio-install/base | kubectl apply -f -
    ```
3. Dex
    ```bash
    kustomize build common/dex/overlays/istio | kubectl apply -f -
    ```
    
4. OIDC AuthService
    authoservice의 경우 spec의 변경이 필요합니다.
    1. authservice spec 다운 받기
        ```bash 
        kustomize build common/oidc-authservice/base > oidc-authservice.yaml
        ```
    2. spec 수정  
        authservice 의 spec을 변경합니다.
        ```yaml
        apiVersion: apps/v1
        kind: StatefulSet
        metadata:
        name: authservice
        namespace: istio-system
        spec:
        replicas: 1
        selector:
            matchLabels:
            app: authservice
        serviceName: authservice
        template:
            metadata:
            annotations:
                sidecar.istio.io/inject: "false"
            labels:
                app: authservice
            spec:
            containers:
            - envFrom:
                - secretRef:
                    name: oidc-authservice-client
                - configMapRef:
                    name: oidc-authservice-parameters
                image: gcr.io/arrikto/kubeflow/oidc-authservice:28c59ef
                imagePullPolicy: Always
                name: authservice
                ports:
                - containerPort: 8080
                name: http-api
                readinessProbe:
                httpGet:
                    path: /
                    port: 8081
                volumeMounts:
                - mountPath: /var/lib/authservice
                name: data
            securityContext:
                runAsUser: 0  # 추가 하기
                # fsGroup: 111  # 지우기
            volumes:
            - name: data
                persistentVolumeClaim:
                claimName: authservice-pvc
        ```
    3. 설치
        ```bash
        kubectl apply -f oidc-authservice.yaml
        ```

5. Knative
    ```bash
    kustomize build common/knative/knative-serving-crds/base | kubectl apply -f -
    kustomize build common/knative/knative-serving-install/base | kubectl apply -f -
    kustomize build common/istio-1-9-0/cluster-local-gateway/base | kubectl apply -f -
    ```
    Optionally, you can install Knative Eventing which can be used for inference request logging:
    ```bash    
    kustomize build common/knative/knative-eventing-crds/base | kubectl apply -f -
    kustomize build common/knative/knative-eventing-install/base | kubectl apply -f -
    ```

6. Kubeflow Namespace
    ```bash
    kustomize build common/kubeflow-namespace/base | kubectl apply -f -
    ```
    
7. Kubeflow Roles
    ```bash
    kustomize build common/kubeflow-roles/base | kubectl apply -f -
    ```
    
8. Kubeflow Istio Resources
    ```bash
    kustomize build common/istio-1-9-0/kubeflow-istio-resources/base | kubectl apply -f -
    ```
    
9. Kubeflow Pipelines using Docker
    ```bash
    kustomize build apps/pipeline/upstream/env/platform-agnostic-multi-user | kubectl apply -f -
    ```
    

# Microservice
Kubeflow에서 사용하는 Microservice들을 설치합니다.

1. KFServing
    ```bash
    kustomize build apps/kfserving/upstream/overlays/kubeflow | kubectl apply -f -
    ````
    
2. Katib
    ```bash
    kustomize build apps/katib/upstream/installs/katib-with-kubeflow | kubectl apply -f -
    ````
    
3. Central Dashboard
    ```bash
    kustomize build apps/centraldashboard/upstream/overlays/istio | kubectl apply -f -
    ````
    
4. Admission Webhook
    ```bash
    kustomize build apps/admission-webhook/upstream/overlays/cert-manager | kubectl apply -f -
    ````
    
5. Notebooks
    ```bash
    kustomize build apps/jupyter/notebook-controller/upstream/overlays/kubeflow | kubectl apply -f -
    kustomize build apps/jupyter/jupyter-web-app/upstream/overlays/istio | kubectl apply -f -
    ````
    
6. Profiles + KFAM
    ```bash
    kustomize build apps/profiles/upstream/overlays/kubeflow | kubectl apply -f -
    ````
    
7. Volumes Web App
    ```bash
    kustomize build apps/volumes-web-app/upstream/overlays/istio | kubectl apply -f -
    ````
    
8. Tensorboard
    ```bash
    kustomize build apps/tensorboard/tensorboards-web-app/upstream/overlays/istio | kubectl apply -f -
    kustomize build apps/tensorboard/tensorboard-controller/upstream/overlays/kubeflow | kubectl apply -f -
    ````
    
9.  User Namespace
    ```bash
    kustomize build common/user-namespace/base | kubectl apply -f -
    ````

# 설치 확인하기
다음 명령어를 통해 모든 pod들이 정상적으로 생성되고 있는지 확인합니다.
```bash
kubectl get pods -n cert-manager
kubectl get pods -n istio-system
kubectl get pods -n auth
kubectl get pods -n knative-eventing
kubectl get pods -n knative-serving
kubectl get pods -n kubeflow
kubectl get pods -n kubeflow-user-example-com
```

## Port-forwarding
port-forwarding을 통해 kubeflow 에 접속합니다.
```bash
kubectl port-forward svc/istio-ingressgateway -n istio-system 8080:80
```

1. http://localhost:8080 로 접속합니다.
2. 기본 ID/PW는 다음과 같습니다.
   - ID: user@example.com
   - PW: 12341234

> 포트 포워딩후 연결이 되지 않을 경우 [포스트]({{< relref "posts/kubernetes/issue_with_auth" >}})를 참고해보시기 바랍니다.

## Reverse-proxy
[다음 글]({{< relref "posts/kubernetes/install_k3s#on-server" >}})에서 설정한 reverse-proxy 도커를 이용해 접속할 수 있습니다.

우선 ingress-gateway의 연결된 포트를 확인합니다.
```bash
kubectl -n istio-system get service istio-ingressgateway
```

다음과 같이 포트들이 포워딩된 것을 확인할 수 있습니다.
```bash
NAME                   TYPE       CLUSTER-IP    EXTERNAL-IP   PORT(S)                                                                      AGE
istio-ingressgateway   NodePort   10.43.244.7   <none>        15021:31160/TCP,80:32455/TCP,443:32293/TCP,31400:32132/TCP,15443:32123/TCP   3d21h
```

이중 80번 포트의 포워딩 포트를 확인합니다. 저는 32455 였습니다.  
만약 다를 경우 [다음 글]({{< relref "posts/kubernetes/install_k3s#on-server" >}}) 에서 `kubeflow.conf`의 값을 수정하면 됩니다.

local에 host를 추가해줍니다.
```bash
sudo sh -c "echo '<VPN_IP> kubeflow.k3s.cluster.local' >> /etc/hosts"
```

http://kubeflow.k3s.cluster.local/ 에 접속합니다.


## 마무리
설치가 정상적으로 되었다면 다음과 같은 화면이 출력됩니다.
![kubeflow_home](/imgs/k3s/kubeflow_home.png)
