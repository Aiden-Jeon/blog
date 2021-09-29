---
title: (CKA) 08. Networking
categories: [kubernetes]
tags: ["k8s", "cka"]
toc: true
date: 2021-06-06
author: Jongseob Jeon
---


## POD Networking
### Networking Model
CKA를 준비하면서 공부한 요약 내용입니다.
- [강의](https://www.udemy.com/course/certified-kubernetes-administrator-with-practice-tests/)
- [What is CKA?](https://www.cncf.io/certification/cka/)


- Every POD should have and IP Address
- Every POD should be able to communicate with every other POD in the same node
- Every POD should be able to communicate with every other POD on other nodes without NAT

## CNI in kubernetes
### Configuring CNI
- kubelete.service
- `ps -aux | grep kubelet`
  - `--cni-conf-dir=/etc/cni/net.d`
  - `ls /opt/cni/bin`
  - `ls /etc/cni/net.d`


## CNI WeaveWorks
### Script
Networking solution has a routing table which mapped what networks are on what hosts
- when a packet is sent from pod to the other, it goes out
  - to the network,
  - to the router,
  - find its way to the node that hosts that pod
→ works for a small environment and in a simple network
→ Larger environment, this is not practical
- routing table may not support so many entries

### WeaveWorks
- deply an agent or service on each node
- they communicate with each other to exchange information withing them
  - nodes
  - networks
  - PODs

#### How it works
- Send
  1. a packet is sent from one pod to another on another node
  2. weave intercepts the packet and identifies that it's on a seperate network
  3. encapsulates this packet into a new one with new source and destination
  4. sends it across the network
- Recieve
  - other weave agent retrieves the packet
  - decapsulates
  - routes the packet to the right pod

#### Deploy
- weave and weave peers
  - deployed as service or daemons on each node
- `kubectl apply -f "<https://cloud.weave.works/k8s/net?k8s-version=$>(kubectl version | base64 | tr -d '\\n')"`
  - deploys daemonset

#### 상태
- `kubectl get pods -n kubesystem | grep weave-net`


## IP Adress Management
- how the tool brutes networks
- the nodes assign an IP subset
- how are the pods assigned an IP
- where is this information stored
- who is responsible for ensuring there are no duplicate IP is assigned

### Script
#### Who
- CNI
  - who define the standards CNI
  - it is responsibility of the CNI plugin, the network provider to take care of assigning IP to the containers

#### How
- get free IP list from file
  - placed on the each host
  - `ip = get_free_ip_from_host_local()`
- `cat /etc/cni/net.d/net-script.conf`
  ```yaml
  "ipam": {
    "type": ...,
    "subnet": ...,
    "routes": ...,
  }
  ```

### WeaveWorks
- entire network
  - 10.32.0.0/12
  - 10.32.0.1 ~ 10.47.255.254


## Service Networking
### Type of service
#### Cluster IP
- if created it is accessible from all pods of the cluster, irrespective of what nodes the pods are on
- hosted across the cluster
- only aceescible from within ther cluster

#### NodePort
- works as cluster IP
- in addtion, it also exposes the applications on a port on all nodes in ther cluster
  - external user or applications have access to the service

### Question
- How are the services getting these IP addresses
- How are they made avaliable acorss all the nodes in the cluster
- How is service made avalible to external users through a port on each node
- Who is doing that, and how and where do we see it

### Answer
#### kubelet process
- which is responsible for creating PODs
- each kubelet service on each node
- whatches the changes in the cluster through the kube-api server
  1. everty time a new pod is to be created, it creates the pod on the nodes
  2. invokes the CNI plugin to configure networking for that pod

#### kube-proxy
- runs on each node
- watches the changes in the cluster through kube-api server
- every time a new service is to be crated
- service are not created on each node or assigned to each node
  - service is the cluster wide concept
- virtual object

#### Service IP
1. create service obejct in k8s
2. it is assigned an IP address from a pre-defined range
3. kube-proxy components running on each node, get's that IP address
4. creates forwarding rulse on each node in ther clsuter
   - any traffic coming in to service IP, should go the IP of the pod

#### Rules of service IP
- kube-proxy supports serveral-way
  - userspace
  - ipvs
  - iptables
- setting
  - `kube-proxy --proxy-mode [userspaces | iptables | ipvs] ...`
    - default: ip tables
- ip tables
  1. pod is created it has ip 10.244.1.2
  2. create a ClusterIP service
     - k8s assigns an IP address to it, 10.103.132.104
     - range is specified in kube-api-server option
       - `kube-api-server --service-cluster-ip-range ipNet`
       - default : 10.0.0.0/24
       - `ps aux | grep kube-api-server`
     - pod ip range and service ip range shoulde be not duplicat ed
  3. see the tables
     - `iptables -L -t net | grep db-service`

#### Logs
- `cat /var/log/kube-proxy.log`
  - what proxy it uses
  - IP tables and Add and entry whe new serviec is created


## DNS in kubernetes
### Objectives
- what names are assigned to what obejcts?
- Service DNS records
- POD DNS records

### Kube DNS
- service is created
- k8s DNS service creates a record for the service
  - it maps the service name to the IP address
- any pod can reach the service using its service name
  - `<service-name>.<namesapce>.<type>.<Root>`
  - subdomain = namespace
  - type = svc
  - root = cluster.local
  - eg) `curl <http://web-service.apps.svc.cluster.local`>
- pod
  - hostname
    - 10.224.2.5 → 10-224-2-5
  - type → pod
  - eg) `curl <http://10-224-2-5.apps.pod.cluster.local`>



## How k8s implements DNS?
### CoreDNS
- `cat /etc/coredns/Corefile`
- `kubectl get configmap -n kube-system`

### DNS Server
- node
  - `cat /etc/resolv.conf`
    - `nameserver 10.96.0.10`
    - k8s do automatically



## Ingress
### Ingress Controller
![ingress-controller](/imgs/cka/networking-1.png)
- nginx deployment
  ```yaml
  apiVersion: extensrions/v1beta1
  kind: Deployment
  metadata:
    name: nginx-ingress-controller
  spec:
    replicas: 1
    selector:
      matchLabels:
        name: nginx-ingress
    template:
      metadata:
        labels:
          name: nginx-ingress
      spec:
        containers:
            - name: nginx-ingress-controller
              image: ...
        args:
          - /nginx-ingress-controller
          - --configmap=$(POD_NAMESPACE)/nginx-configuration
        env:
          - name: POD_NAME
            valueFrom:
              fieldRef:
                fieldPath: metadata.name
          - name: POD_NAMESPACE
            valueFrom:
              fieldRef:
                fieldPath: metadata.namesapce
        ports:
          - name: http
            contrainerPort: 80
          - name: https
            contrainerPort: 443
  ```
- configmap
  - feed nginx configuration data
  ```yaml
  kind: ConfigMap
  apiVersion: v1
  metadata:
    name: nginx-configuration
  ```
- service
  - expose
  ```yaml
  apiVersion: v1
  kind: Service
  metadata:
    name: nginx-ingress
  spec:
    type: NodePort
    ports:
      - port: 80
        targetPort: 80
        protocol: TCP
        name: http
      - port: 443
        targetPort: 443
        protocol: TCP
        name: https
    selector:
      name: nginx-ingress
  ```
- service account
  - right permissions to access
  - Roles
  - ClusterRoles
  - RoleBindings
  ```yaml
  apiVersion: v1
  kind: ServiceAccount
  metadata:
    name: nginx-ingress-serviceaacount
  ```

### Ingress Resource
- simple one application
- single domain and several subdomains
- several domains
- 상태
  - `kubectl get ingress`

#### Rules
![ingress-rules](/imgs/cka/networking-2.png)

#### Single
![ingress-single](/imgs/cka/networking-3.png)
- definition
  ```yaml
  apiVersion: extensrions/v1beta1
  kind: Ingress
  metadata:
    name: ingress -wear
  spec:
    backend:
      serviceName: web-service
      servicePort:  80
  ```

#### Several rules
![ingress-serveral-rules](/imgs/cka/networking-4.png)
- definition
  ```yaml
  apiVersion: extensrions/v1beta1
  kind: Ingress
  metadata:
    name: ingress -wear
  spec:
    rules:
    - http:
        paths:
        - path: /wear
          backend:
            serviceName: wear-service
            servicePort: 80
        - path: /watch
          backend:
            serviceName: watch-service
            servicePort: 80
  ```
  - 상태
    - `kubectl describe ingress ingress-wear-watch`
    - default backend
      - if user access to not specified server

#### Several domains
![ingress-serveral-domains](/imgs/cka/networking-5.png)
- definition
  ```yaml
  apiVersion: extensrions/v1beta1
  kind: Ingress
  metadata:
    name: ingress -wear
  spec:
    rules:
    - host: wear.my-online-store.com
      http:
        paths:
        - backend:
            serviceName: wear-service
            servicePort: 80
    - host: watch.my-online-store.com
      http:
        paths:
        - backend:
            serviceName: watch-service
            servicePort: 80
  ```
