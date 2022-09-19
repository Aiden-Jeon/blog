---
title: (CKA) 06. Security
categories: [kubernetes]
tags: ["k8s", "cka"]
toc:
  auto: true
date: 2021-05-23 08:00:00
author: Jongseob Jeon
---

CKA를 준비하면서 공부한 요약 내용입니다.
- [강의](https://www.udemy.com/course/certified-kubernetes-administrator-with-practice-tests/)
- [What is CKA?](https://www.cncf.io/certification/cka/)


## Kubernetes Security Primitives
### Secrure Hosts
- Password based authentication disabled
- SSH Key based authentication

### Secure kubernetes
- kube-apiserver
  - Who can access?
  - What can they do?

### Authentication
#### Who can access?
- Files - Username and Passwords
- Fiels - Username and Tokens
- Certificates
- External Authenication providers - LDAP
- Service Accounts

### Authorization
#### What can they do?
- RBAC Authorization
- ABAC Authorization
- Node  Authorization
- Webhook Mode

### TLS Certificates
![tls-certificates-example](/imgs/cka/security-1.png)

### Network Policies
![networking-policies-example](/imgs/cka/security-2.png)

## Authentication
Users
- Admins
- Developers
- Application End Users
- Bots

### Accounts
- User
  - Admins
  - Developers
- Servcie Accounts
  - Bots
  - not supported officialy in k8s

### Auth Mechanisms
- kube-apiserver
  - Static Password File
  - Static Token File
  - Certificates
  - Identity Service

#### Basic - file
- `user-details.csv`
  ```
  password123,user1,u0001
  password123,user2,u0002
  ```
- `kube-apiserver.service`
  ```bash
  ExecStart=/usr/local/bin/kube-apiserver \
        --basic-auth-file=user-details.csv \
        ...
  ```
- kubeadm
  ```yaml
  apiVersion: v1
  kind: Pod
  metadata:
    creationTimestamp: null
    name: kube-apiserver
    namespace: kube-system
  spec:
    containers:
    - command:
      - kube-apiserver
      - --authorization-mode=Node,RBAC
        <content-hidden>
      - --basic-auth-file=/tmp/users/user-details.csv
  ```
- pod
  ```yaml
  apiVersion: v1
  kind: Pod
  metadata:
    name: kube-apiserver
    namespace: kube-system
  spec:
    containers:
    - command:
      - kube-apiserver
        <content-hidden>
      image: k8s.gcr.io/kube-apiserver-amd64:v1.11.3
      name: kube-apiserver
      volumeMounts:
      - mountPath: /tmp/users
        name: usr-details
        readOnly: true
    volumes:
    - hostPath:
        path: /tmp/users
        type: DirectoryOrCreate
      name: usr-details
  ```
- Authenticate User
  - `curl -v -k <https://master-node-ip:6443/api/v1/pods> -u "user1:password123"`

#### Basic - token
- `user-token-details.csv`
  ```
  asdfbasdhjasdhjkfhjk12312,user1,u0001
  1234g23rgy8werfg89asdfasd,user2,u0002
  ```
- `kube-apiserver.service`
  ```bash
  ExecStart=/usr/local/bin/kube-apiserver \
        --token-auth-file=user-details.csv \
        ...
  ```
- kubeadm
  - `kubectl edit`
- Authenticate User
  - `curl -v -k <https://master-node-ip:6443/api/v1/pods> --header "Authorization: Bearer asdfbasdhjasdhjkfhjk12312`


## TLS
- master node
- worker nodes

### Server Certificates for Servers
![server-certificates-for-server](/imgs/cka/security-3.png)

#### Kube-Api server
- exposes a service that other components as well as external users use to manage the k8s cluster
- server
  - certificates to secure all communications with its clients
  - `apiserver.crt`
  - `apiserver.key`

#### ETCD Server
- `etcdserver.crt`
- `etcdserver.key`

#### KUBELET Server
- `kubelet.crt`
- `kubelet.key`

### Client Certificates for Clients
![client-certificates-for-clients](/imgs/cka/security-4.png)

#### Admin
- `admin.crt`
- `admin.key`

#### Kube-Scheduler
- `scheduler.crt`
- `scheduler.key`

#### Kube-Controller-Manager
- `controll-manager.crt`
- `controll-manager.key`

#### Kube-Proxy
- `kube-proxy.crt`
- `kube-proxy.key`


## Certification Creation
### OpenSSL to Create Client Certification
- Certificate Authority(CA)
  - Generate Keys (ca.key)
    - `openssl genrsa -out ca.key 2048`
  - Certificate Signing Requests (ca.csr)
    - `openssl req -new -key ca.key -subj "/CN=KUBERNETES-CA" -out ca.csr`
  - Sign Certificates (ca.crt)
    - `openssl x509 -req -in ca.csr -signkey ca.key -out ca.crt`
- Admin User
  - Generate Keys (admin.key)
    - `openssl genrsa -out admin.key 2048`
  - Certificate Signing Requests (admin.csr)
    - `openssl req -new -key admin.key -subj "/CN=kube-admin/O=system:masters" -out admin.csr`
    - `"/CN=kube-admin/O=system:masters"` ← admin user
  - Sign Certificates (admin.crt)
    - `openssl x509 -req -in admin.csr -signkey ca.key -out admin.crt`
- Kube Scheduler
- Kube Controller Manager
- kube Proxy

### How to use
- `curl <https://kube-apisever:6443/api/v1/pods> --key admin.key --cert admin.crt --cacert ca.crt`
- definition
  ```yaml
  apiVersion: v1
  clusters:
  - cluster:
      certificate-authority: ca.crt
      server: <https://kube-apiserver:6443>
    name: kubernetes
  kind: Config
  users:
  - name: kubernetes-admin
    user:
      client-certificate: admin.crt
      clinet-key: admin.key
  ```

### OpenSSL to Create Server Certification
#### ETCD Servers
```yaml
- etcd
  --key-file=
  --cert-file=
  --peer-cert-file=
  --peer-client-cert-auth=
  --peer-key-file=
  --peer-trusted-ca-cile
  --trusted-ca-file= 
```

#### Kube-Api Server
![server-certificate-example](/imgs/cka/security-5.png)

- Generate Keys (apiserver.key)
  - `openssl genrsa -out apiserver.key 2048`
- Certificate Signing Requests (apiserver.csr)
  - `openssl.cnf`
    ```
    [req]
    req_extensions = v3_req
    [ v3_req ]
    basicConstraints = CA:FALSE
    keyUsage = nonRpudiation,
    subjectAltName = @alt_names
    [alt_names]
    DNS.1 = kubernetes
    DNS.2 = kubernetes.default
    DNS.3 = kubernetes.default.svc
    DNS.4 = kubernetes.default.svc.cluster.local
    IP.1 = 10.96.0.1
    IP.2 = 172.17.0.87
    ```
  - `openssl req -new -key apiserver.key -subj "/CN=kube-apiserver" -out apiserver.csr -config openssl.cnf`
- Sign Certificates (apiserver.crt)
  - `openssl x509 -req -in apiserver.csr -CA ca.crt -CAkey ca.key -out apiserver.crt`

#### Kubelet Server
- server cert
  ![server-cert-examples](/imgs/cka/security-6.png)
  - definition
    ```yaml
    apiVersion: kubelet.config.k8s.io/v1beta1
    kind: KubeletConfiguration
    authentication:
      x509:
        clientCAFile: "/var/lib/kubernetes/ca.pem"
    authorization:
      mode: Webhook
    clusterDomain: "cluster.local"
    clusterDNS:
      - "10.32.0.10"
    podCIDR: "$(POD_CIDR)"
    resolveConf: "/run/systemd/resolve/resolv.conf"
    reuntimeRequrestTimeout: "15m"
    tlsCertFile: "/var/lib/kubelet/kubelet-node01.crt"
    tlsPrivateKeyFile: "/var/lib/kubelet/kubelet-node01.key"
    ```
- client cert
  ![client-crt-examples](/imgs/cka/security-7.png)


## View Certificate Details
- The Hard Way
  - `cat /etc/systemd/system/kube-apiserver.service`
- kubeadm
  - `cat /etc/kubernetes/manifest/kube-apiserver.yaml`

### Inspect Service Logs
- `journalctl -u etcd.service`

### View Logs
- `kubectl logs etcd-master`


## Certificates API
### Certificate Signing Request
#### Order
1. Create `CertificateSigningReqeust` Object
2. Review Requests
3. Approve Requests
4. Share Certs to Users

#### Request
- `openssl genrsa -out jan.key 2048`
- `openssl req -new -key jane.key -subj "/CN=jane" -out jan.csr`
- send to the administrator
- administrator takes the key and creates a `CertifacteSigningRequest` obejct
  ```yaml
  apiVersion: certificates.k8s.io/v1beta1
  kind: CeritifcateSigningRequest
  metadata:
    name: jane
  spec:
    groups:
    - system:authenticated
    usage:
    - digital signature
    - key encipherment
    - server auth
    request:
      <cat jane.csr | base64>
  ```
  - `cat akshay.csr | base64 | tr -d "\\n"`

#### Approve
- `kubectl get csr`
  ```bash
  > k get csr
  NAME     AGE   SIGNERNAME                            REQUESTOR          CONDITION
  akshay   10s   kubernetes.io/kube-apiserver-client   kubernetes-admin   Pending
  ```
- `kubectl certificate approve jane`

#### 상세 요청
- `kubectl get csr jane -0 yaml`

#### Deny
- `kubectl certificate deny agent-smith`

#### Delete
- `kubectl delete csr agent-smith`

### Contoller Manager
- CSR-APPROVING
- CSR-SIGNING
- `cat /etc/kubernetes/manifests/kube-controller-manager.yaml`


## Kubeconfig
### Using kubectl with crt
```bash
kubectl get pods
  --server my-kube-playground:6443
  --client-key admin.key
  --client-certificated admin.crt
  --certificate-authority ca.crt
```
- this is tedious task
  - → move it to config file

### Using kubectl with config file
- config
  ```bash
    --server my-kube-playground:6443
    --client-key admin.key
    --client-certificated admin.crt
    --certificate-authority ca.crt
  ```
- kubectl
  - `kubectl get pods --kubeconfig config`
  - default config file path
    - `.kube/config`

### KubeConfig File
- clusters
  - development
  - production
  - google
  - `--server my-kube-playground:6443`
- contexts
  - match cluster and users
    - admin@production
    - dev@google
    - mykubeadmin@mykubeplayground
- users
  - admin
  - dev user
  - prod user
  ```bash
    --client-key admin.key
    --client-certificated admin.crt
    --certificate-authority ca.crt
  ```
- defintion
  ```yaml
  apiVersion: v1
  kind: Config
  current-context: dev-user@google
  clusters:
  - name: my-kube-playground
    cluster:
      certificate-authority: ca.crt
      server: my-kube-playground:6443
  - name: ...
  contexts:
  - name: my-kube-admin@my-kube-playground
    context:
      clusters: my-kube-playground
      user: my-kube-admin
  - name: ...
  users:
  - name: my-kube-admin
    user:
      client-certificate: admin.crt
      client-key: admin.key
  - name: ...
  ```
  - `current-context`
    - default context
  - `certificate-authority`
    - instead file path using data directly
    - `  certificate-authority-data`
      - with base64 encoding

#### Command Line Tool
- view
  - `kubectl config view`
  - `kubectl config view --kubeconfig=<config-file-name>`
- update
  - `kubectl config use-context prod-user@production`
- helo
  - `kubectl config -h`


## API Groups
### Example
- `curl <https://kube-master:6443/version`>
- `curl <https://kube-master:6443/api/v1/pods`>

### Types
- `/metrics`
- `/healthz`
- `/version`
- `/api`
- `/apis`
- `/logs`

### API, APIS
- `api`
  - core group
  ![core-grup](/imgs/cka/security-8.png)
- `apis`
  - named group
  ![named-group](/imgs/cka/security-9.png)
- CLT
  - `curl <http://localhost:6443> -k`
  - `curl <http://localhost:6443> -k | grep "name"`
  - need certificates file
- `kubectl proxy`
  - to see proxy
  - `kube proxy` ≠ `kubectl proxy`
  - `kube proxy`
    - enable connectivity betewwnd pods and services across different nodes in the cluster.
  - `kubectl proxy`
    - HTTP proxy service created by kubectl utility to access the kube-api sever


## Authorization
### Authorization Mechanisms
#### Types
- Node
- ABAC
- RBAC
- Webhook

#### Node
- Node Authorizer

#### ABAC
User Attribute Based Authorization

- dev-user
  - → Can view PODs
  - → Can create PODs
  - → Can delete PODs
  ```json
  {"kind": "Policy", "spec": {"user": "dev-user", "namespace": "*", "resource": "pods", "apiGroup": "*"}}
  ```

#### RBAC
Rule Based Authorizaion

- Developer
  - → Can view PODs
  - → Can create PODs
  - → Can delete PODs
- definition
  ```yaml
  apiVersion: rbac.authorization.k8s.io/v1
  kind: Role
  metadata:
    name: developer
  rules:
  - apiGroups: [""]
    resources: ["pods"]
    verbs: ["list", "get", "create", "update", "delete"]
    resourceNames: ["blue", "orange"]
  ```
  - rules have 3 sections
    - `apiGroups`
    - `resources`
    - `verbs`
- link users to rule
  ```yaml
  apiVersion: rbac.authorization.k8s.io/v1
  kind: RoleBinding
  metadata:
    name: devuser-developer-binding
  subjects:
  - kind: User
    name: dev-user
    apiGroup: rbac.authorization.k8s.io
  roleRef:
    kind: Role
    name: developer
    apiGroup: rbac.authorization.k8s.io
  ```
- view
  - `kubectl get roles`
  - `kubectl get rolebindings`
- Check Access
  - `kubectl auth can-i create deployments`
  - `kubectl auth can-i delete nodes`
  - `kubectl auth can-i create deployments --as dev-user`

#### Webhook
- outsource all the mechanism

#### AlwaysAllow
- always allow

#### AlwaysDeny
- always deny

### Setting
```bash
ExecStart = ...\\
  --authorization-mode=AlwaysAllow
```
- default : `AlwaysAllow`
- multiple mode: 
  ```
  Node,RBAC,Webhook
  ```
  - if one's requiest is denied, it pass to next chain
  - if approved no more checks.

### Inspect
- ```
  kubectl describe pod kube-apisever-controlplane -n kube-system
  ```
  - `--authorizastion mode`


## Cluster Roles
### Types
- namespace ← role
  - pods
  - replicasets
  - ...
  - `kubectl api-resources --namespaced=true`
- cluster  scoped
  - nodes
  - pv
  - ...
  - `kubectl api-resources --namespaced=false`

### Clusterroles
- Cluster admin
  - Can view Nodes
  - can create Nodes
  - Can delete Nodes
- definition
  ```yaml
  apiVersion: rbac.authorizastion.k8s.io/v1
  kind: ClusterRole
  metadata:
    name: cluster-administrator
  rules:
  - apiGroups: [""]
    resources: ["nodes"]
    verbs: ["list", "get", "create", "delete"]
  ```

### Clusterrolebining
- definition
  ```yaml
  apiVersion: rbac.authorization.k8s.io/v1
  kind: ClusterRoleBinding
  metadata:
    name: cluster-admin-role-binding
  subjects:
  - kind: User
    name: cluster-admin
    apiGroup: rbac.authorization.k8s.io
  roleRef:
    kind: ClusterRole
    name: cluster-administrator
    apiGroup: rbac.authorization.k8s.io
  ```


## Image Security
### Private Repository
#### Login to private repository
- `docker login private-registry.io`
- `docker run private-registry.io/apss/internal-app`

#### Create secret
```yaml
kubectl create secrete docker-registry regcred  \\
  --docker-server=private-registry.io  \\
  --docker-username=registry-user  \\
  --docker-password=registry-password  \\
  --docker-email=registry-user@org.com
```

#### Create pod
- definition
  ```yaml
  apiVersion: v1
  kind: Poid
  metadata:
    name: nginx-pod
  spec:
    containers:
    - name: nginx
      image: private-registry.op/apps/internal-app
    imagePullSecrets:
    - name: regcred
  ```


## Security Context
- definition for pod
  ```yaml
  apiVersion: v1
  kind: Pod
  metadata:
    name: web-pod
  spec:
    securityContext:
      runAsUser: 1000
    containers:
      - name: ubuntu
        image: ubuntu
        command: ["sleep", "3600"]
  ```
- definition for container
  ```yaml
  apiVersion: v1
  kind: Pod
  metadata:
    name: web-pod
  spec:
    containers:
      - name: ubuntu
        image: ubuntu
        command: ["sleep", "3600"]
        securityContext:
          runAsUser: 1000
          capabilities:
            add: ["AMC_ADMIN"]
  ```
  - `capabilities`
    - only supported at the container level


## Network Policy
### Ingress & Egress
![ingres-egress-outline](/imgs/cka/security-10.png)
- ingress
  - for a web serever the incoming traffic from the users is an ingress
- egress
  - outgoing request to the app server
  - allow external calls

### traffic
![traffic-outline](/imgs/cka/security-11.png)

### Network Security
- Network Policy
  - eg) Allow Ingress Raffic From API pod on Port 3306

#### Selectors
- definition
  ```yaml
  apiVersions: networking.k8s.io/v1
  kind: NetworkPolicy
  metadata:
    name: db-policy
  spec:
    podSelector:
      mathLabels:
        role: db
    policyTpes:
    - Ingress
    ingress:
    - from:
      - podSelector:
          matchLabels:
            name: api-pod
      ports:
      - protocol: TCP
        port: 3306
  ```


## Network Policies
- network policy to pod to protect
  - defintion
    ```yaml
    apiVersions: networking.k8s.io/v1
    kind: NetworkPolicy
    metadata:
      name: db-policy
    spec:
      podSelector:
        mathLabels:
          role: db
    ```
- what type of policies?
  - ingress or egress or both ?
  - allow incoming request → ingress
  - if incoming request is allowed, response is allowed automatically
  - definition
    ```yaml
      policyTpes:
      - Ingress
    ```
- define specific rules
  ```yaml
    ingress:
    - from:
      - podSelector:
          matchLabels:
            name: api-pod
      ports:
      - protocol: TCP
        port: 3306
  ```
- allow only in prod namespace
  - definition (and)
    ```yaml
      ingress:
      - from:
        - podSelector:
            matchLabels:
              name: api-pod
          namespaceSelector:
            matchLabels:
              name: prod
        ports:
        - protocol: TCP
          port: 3306
    ```
    - if only `namespaceSelector` is defined
      - allow or request in same namespace
  - definition (or)
    ```yaml
      ingress:
      - from:
        - podSelector:
            matchLabels:
              name: api-pod
        - namespaceSelector:
            matchLabels:
              name: prod
        ports:
        - protocol: TCP
          port: 3306
    ```
    - use array
- to backup in server
  ```yaml
    - ipBlock:
      - cidr: 192.168.5.10/32
  ```
- to push to bakcup server
  ```yaml
  apiVersions: networking.k8s.io/v1
  kind: NetworkPolicy
  metadata:
    name: db-policy
  spec:
    podSelector:
      mathLabels:
        role: db
    policyTpes:
    - Ingress
    - Egress
    ingress:
    - from:
      - podSelector:
          matchLabels:
            name: api-pod
      ports:
      - protocol: TCP
        port: 3306
    egress:
    - to:
      - ipBlock:
          cidr: 192.168.5.10/32
      ports:
      - protocol: TCP
        port: 80
  ```
