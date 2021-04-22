---
title: helm을 이용한 deployment chart 만들기
comment: true
categories: [cicd]
tags: ["helm"]
toc: true
date: 2021-03-04
author: Jongseob Jeon
---

**CI/CD Contents 순서**
1. [sphinx-autoapi 를 이용한 자동 api 문서 생성하기]({{< relref "post/tech/cicd/sphinx_autoapi" >}})
2. [github action을 이용한 ci]({{< relref "post/tech/cicd/github_action_ci" >}})
3. [ghcr을 이용한 kubernetes deployment 만들기]({{< relref "post/tech/cicd/ghcr_k8s_deploy" >}})
4. [helm을 이용한 deployment chart 만들기]({{< relref "post/tech/cicd/helm_deployment_chart" >}})
5. [argocd를 이용한 cd]({{< relref "post/tech/cicd/argocd_cd" >}})

---



이번 포스트에서는 helm 을 이용해 [이전 포스트](https://aiden-jeon.github.io/cicd/github-cicd-2) 에서 작성한 파일들을 자동화 하는 법에 대해서 알아 보겠습니다.

## 1. helm start
helm 을 시작하려는 repo 최상단에서 다음과 같이 입력해줍니다.
```bash
> helm create sphinx-doc

Creating sphinx-doc
```

그러면 아래와 같은 파일들이 생성됩니다.
```bash
> cd sphinx-doc
> ls

Chart.yaml  charts      templates   values.yaml
```

## 2. `deployment.yaml`
생성된 `deployment.yaml` 을 다음과 같이 수정합니다.
```yaml
{% raw %}
apiVersion: apps/v1
kind: Deployment
metadata:
  name: {{ template "sphinx-doc.fullname" . }}
  namespace: sphinx-doc
  labels:
    app: {{ template "sphinx-doc.name" . }}
    chart: {{ template "sphinx-doc.chart" . }}
    release: {{ .Release.Name }}
    heritage: {{ .Release.Service }}
spec:
  replicas: {{ .Values.replicaCount }}
  revisionHistoryLimit: 3
  selector:
    matchLabels:
      app: {{ template "sphinx-doc.name" . }}
      release: {{ .Release.Name }}
  template:
    metadata:
      labels:
        app: {{ template "sphinx-doc.name" . }}
        release: {{ .Release.Name }}
    spec:
      imagePullSecrets:
        - name: "{{ .Values.image.pullSecrets }}"
      containers:
        - name: {{ .Chart.Name }}
          image: "{{ .Values.image.repository }}:{{ .Values.image.tag }}"
          imagePullPolicy: {{ .Values.image.pullPolicy }}
          ports:
            - name: http
              containerPort: 80
              protocol: TCP
          livenessProbe:
            httpGet:
              path: /
              port: http
          readinessProbe:
            httpGet:
              path: /
              port: http
          resources:
{{ toYaml .Values.resources | indent 12 }}
    {{- with .Values.nodeSelector }}
      nodeSelector:
{{ toYaml . | indent 8 }}
    {{- end }}
    {{- with .Values.affinity }}
      affinity:
{{ toYaml . | indent 8 }}
    {{- end }}
    {{- with .Values.tolerations }}
      tolerations:
{{ toYaml . | indent 8 }}
    {{- end }}
{% endraw %}
```

## 3. `service.yaml`
생성된 `service.yaml` 을 다음과 같이 수정합니다.

```yaml
{% raw %}
apiVersion: v1
kind: Service
metadata:
  namespace: sphinx-doc
  name: {{ template "sphinx-doc.fullname" . }}
  labels:
    app: {{ template "sphinx-doc.name" . }}
    chart: {{ template "sphinx-doc.chart" . }}
    release: {{ .Release.Name }}
    heritage: {{ .Release.Service }}
spec:
  type: {{ .Values.service.type }}
  ports:
    - port: {{ .Values.service.port }}
      targetPort: http
      protocol: TCP
      name: http
  selector:
    app: {{ template "sphinx-doc.name" . }}
    release: {{ .Release.Name }}
{% endraw %}
```

## 4. `ingress.yaml`
생성된 `ingress.yaml` 을 다음과 같이 수정합니다.

```yaml
{% raw %}
{{- if .Values.ingress.enabled -}}
{{- $fullName := include "sphinx-doc.fullname" . -}}
{{- $ingressPath := .Values.ingress.path -}}
{{- $svcPort := .Values.service.port -}}
{{- if semverCompare ">=1.14-0" .Capabilities.KubeVersion.GitVersion -}}
apiVersion: networking.k8s.io/v1beta1
{{- else -}}
apiVersion: extensions/v1beta1
{{- end }}
kind: Ingress
metadata:
  namespace: mrxflow
  name: {{ template "sphinx-doc.fullname" . }}
  labels:
    app: {{ template "sphinx-doc.name" . }}
    chart: {{ template "sphinx-doc.chart" . }}
    release: {{ .Release.Name }}
    heritage: {{ .Release.Service }}
  {{- with .Values.ingress.annotations }}
  annotations:
    {{- toYaml . | nindent 4 }}
  {{- end }}
spec:
{{- if .Values.ingress.tls }}
  tls:
  {{- range .Values.ingress.tls }}
    - hosts:
      {{- range .hosts }}
        - {{ . | quote }}
      {{- end }}
      secretName: {{ .secretName }}
  {{- end }}
{{- end }}
  rules:
  {{- range .Values.ingress.hosts }}
    - host: {{ .host }}
      http:
        paths:
          - path: {{ $ingressPath }}
            backend:
              serviceName: {{ $fullName }}
              servicePort: {{ $svcPort }}
  {{- end }}
{{- end }}
{% endraw %}
```

## 5. `values-local.yaml`
local에서 확인을 위한 value로 `values-local.yaml` 을 만들겠습니다.

```yaml
replicaCount: 1
image:
  repository: ghcr.io/aiden-jeon/sphinx-api
  tag: dc3c4be
  pullPolicy: Always
  pullSecrets: test-ghcr
service:
  type: NodePort
  port: 80
ingress:
  enabled: false
  annotations:
    kubernetes.io/ingress.class: nginx
    ingress.kubernetes.io/rewrite-target: /
  path: /
  hosts:
    - host: chart-example.local
  tls: []

resources: {}

nodeSelector: {}
tolerations: []
affinity: {}
```

## 3. helm install
이제 helm 을 이용해 install 해보겠습니다.

우선 kubectl namespace를 생성합니다.
```bash
> kubectl create ns sphinx-doc

namespace/sphinx-doc created
```

새 namespace에 secret을 추가해줍니다.
```bash
kubectl -n sphinx-doc create secret docker-registry test-ghcr \
    --docker-server=ghcr.io \
    --docker-username=aiden-jeon \
    --docker-password=<secret_key> \
    --docker-email=ells2124@gmail.com
```

사용하지 않는 `serviceaccount.yaml` `hpa.yaml` 을 삭제합니다.
```bash
rm sphinx-doc/templates/serviceaccount.yaml
rm sphinx-doc/templates/hpa.yaml
```

그리고 helm install을 합니다.
```bash
> helm install sphinx-doc sphinx-doc/ --values sphinx-doc/values-local.yaml

NAME: sphinx-doc
LAST DEPLOYED: Thu Mar  4 16:59:41 2021
NAMESPACE: default
STATUS: deployed
REVISION: 1
NOTES:
1. Get the application URL by running these commands:
  export NODE_PORT=$(kubectl get --namespace default -o jsonpath="{.spec.ports[0].nodePort}" services sphinx-doc)
  export NODE_IP=$(kubectl get nodes --namespace default -o jsonpath="{.items[0].status.addresses[0].address}")
  echo http://$NODE_IP:$NODE_PORT
```

정상적으로 설치되면 아래와 같이 자동으로 service 들이 뜨게 됩니다.
```bash
> kubectl get all -n sphinx-doc

NAME                              READY   STATUS    RESTARTS   AGE
pod/sphinx-doc-6f5b76f46c-tzt7d   1/1     Running   0          11s

NAME                 TYPE       CLUSTER-IP      EXTERNAL-IP   PORT(S)        AGE
service/sphinx-doc   NodePort   10.103.77.217   <none>        80:32164/TCP   11s

NAME                         READY   UP-TO-DATE   AVAILABLE   AGE
deployment.apps/sphinx-doc   1/1     1            1           11s

NAME                                    DESIRED   CURRENT   READY   AGE
replicaset.apps/sphinx-doc-6f5b76f46c   1         1         1       11s
```

연결을 위한 ip를 확인합니다.
```bash
> minikube service -n sphinx-doc --url sphinx-doc

http://192.168.64.2:32164
```

정상적으로 api document 가 보이는지 확인합니다.
