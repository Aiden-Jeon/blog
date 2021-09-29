---
title: (CKA) 08. Networking Pre-requisites
categories: [kubernetes]
tags: ["k8s", "cka"]
toc: true
date: 2021-06-06
author: Jongseob Jeon
---

CKA를 준비하면서 공부한 요약 내용입니다.
- [강의](https://www.udemy.com/course/certified-kubernetes-administrator-with-practice-tests/)
- [What is CKA?](https://www.cncf.io/certification/cka/)


## Pre-Requisites
- Switching and Routing
  - Switching
  - Routing
  - Default Gateway
- DNS
  - DNS Configurations on Linux
  - CoreDNS Introduction
- Network Namespaces
- Docker Networking

## Switching Routing
### Switching
![switching](/imgs/cka/networking-pre-1.png)

#### 상황
- There are two computers and want to communicate to each other
  - → use switch

#### 방법
1. see the interface of host
   - `ip link`
   - look at the interface named `eth0`
2. assume ip address
   - switch: 192.168.1.0
3. give ip to hosts
   - `ip addr add <ip>/<port> <hostname> <interface>`
   - A: 192.168.1.10
     - `ip addr add 192.168.1.10/24 dev eth0`
   - B: 192.168.1.11
     - `ip addr add 192.168.1.11/24 dev eth0`
4. Two computers can communicate each other through switch
   - `ping 192.16.1.11`

#### 특징
- switch can only enable communications within network
- it can receive packets from a host on the network and deliver it to other system within the same network

### Routing
![routing](/imgs/cka/networking-pre-2.png)

#### 상황
- New system exists
  - Two computer name with C, D
  - Ip address
    - C: 192.168.2.10
    - D: 192.168.2.11
- How does computer A reach to computer c?
- → Router
  - helps connect two networks together.

#### 방법
1. Connecting two networks, it need two ip assigned.
   - first network (A&B): 192.168.1.1
   - second network (C&D): 192.168.2.1
2. Send A to C
   - how does know where is C?
   - → Gateway

### Gateway
![gateway](/imgs/cka/networking-pre-3.png)

#### 상황
- network ← room
- gateway ← a door to the outside world to other networks

#### 방법
1. systems need to know where that door is to go through
   - to see the existing routing configuration on a system
   - `route`
   - displays the kernels routing table
2. Configure a gateway
   1. `ip route add 192.168.2.0/24 via 192.168.1.1`
      - → you can reach the 192.168.2.0 network through the door or gateway at 192.168.1.11
   2. `route`
      - see the route added
3. Configure on all system
   1. `ip route add 192.168.1.0/24 via 192.168.2.1`
   2. `route`

### Internet
#### 상황
- System need acces to the internet

#### 방법
- `ip route add 172.217.194.0/24 via 192.168.2.1`
  - add a new road in routing table to road all traffic to the network 172.217.194.0

### Default gateway
![default-gateway](/imgs/cka/networking-pre-4.png)

#### 상황
- any request to any network outside of existing network goes to this particular router

#### 방법
- `ip route add default via 192.168.2.1`
- also can called 0.0.0.0
  - `ip route add 0.0.0.0 via 192.168.2.1`
- if default gateway is defined, in same network there is no need to add 192.168.1.0
- in sepreated network
  - one entry for private network
  - `ip addr add 192.168.1.0/24 via 192.168.2.2`

### Linux as a host
![linux-host](/imgs/cka/networking-pre-5.png)

#### 상황
- 3 hosts, A, B, C
- host B is connected to both networks using eth0 and eth1
  - A&B are connected with 192.168.1.0
  - B&C are connected with 192.168.2.0
- how to reach from A to C
  - → need to tell host A that the door or gateway to network C is through host B

#### 방법
1. add a routing table entry
   - `ip route add 192.168.2.0/24 via 192.168.1.6`
2. when packet is reached to host C, host C will have to send back response to host A
   - `ip route add 192.168.1.0/24 via 192.168.2.6`
3. in Linux, packets are not forwarded from one interface to the next
   - eg) pacets receivced on eht0 on host B, are not forwarded to through eth1
   - eth0: private, eth1: public
4. allow forward
   - default: denied
     ```bash
     > cat /proc/sys/net/ipv4/ip_forward
     0
     ```
   - set allowed
     ```bash
     > echo 1> /proc/sys/net/ipv4/ip_forward
     1
     ```
   - this is not persistant, changed if reboot.
   - to make persistent, modify the same value in the `/etc/sysctl.conf`
     ```bash
     net.ipv4.ip_forward=1
     ```

### Take Aways
#### ip
- `ip link`
  - to list and modify interfactes on the host
- `ip addr`
  - to see the ip addresses assigned to those interface
- `ip addr ad`
  - add IP addresses on the interfaces

## DNS
### Name Resolution
![name-resolution](/imgs/cka/networking-pre-6.png)

#### 상황
- two computers A and B
  - same network
  - assigned IP
    - A: 192.168.1.0
    - B: 192.168.1.11
- able to ping one computer to others with IP
  - `ping 192.168.1.11`
- B has a database service
  - give a name DB
- want to ping to B, using DB instead of IP address
  - `ping db`
  - → err: `ping :unkown host db`

#### 방법
1. When i say db, it means IP 192.161.1.11
   ```bash
   cat >> /etc/hosts
   192.168.1.11 db
   ```
2. `ping db`
   - successful

#### 특징
- doesn't matter real host name
- can have many names
- need to all specify other system
  ![name-resoltuions](/imgs/cka/networking-pre-7.png)

### DNS Server
![before-dns-server](/imgs/cka/networking-pre-8.png)
![after-dns-server](/imgs/cka/networking-pre-9.png)

#### 상황
- until the enviroment grew and these files got filled with too many entries
- managing these files too hard
- move all these files in one server
- → DNS Server
  - point all hosts to look up tahat server
  - ip: 192.168.1.100

#### 방법
![how-dns-server](/imgs/cka/networking-pre-10.png)
1. add entry
   ```bash
   cat /etc/resolv.conf
   nameserver   192.168.1.100
   ```
2. configure in all host

#### 특징
- also can use `/etc/hosts`
  ```bash
  cat >> /etc/hosts
  192.168.1.115    test
  ```
  - `ping test`
- if an entry in both places
  - host first looks in the local `/etc/hosts`
  - and then look at the name sever
- order can change
  - `etc/nsswitch.conf`
- multiple name server is possible

### Domain Names
#### Structure
![domain-name-structure](/imgs/cka/networking-pre-11.png)
- tree structure
  - .com
    - top level domain
  - google
    - domain name
  - www
    - subdomain
      - helping further grouping things together
      - eg) maps, drive, ..

#### In public
![domain-name-works-public](/imgs/cka/networking-pre-12.png)
1. request hits organization's internal DNS server
2. forwards request to the Internet
   - point to a serving .com
3. forwards to Google
4. Google DNS server provides IP of ther serving the apps applications
5. to speed up all future results, organization's DNS server choose to cache this IP

### In private
![domain-names-works-private](/imgs/cka/networking-pre-13.png)
- within in company do not wany to use domain name
  ```bash
  cat >> /etc/resolv.conf
  nameserver   192.168.1.100
  search       mycompany.com
  ```

### Record Types
![record-types](/imgs/cka/networking-pre-14.png)

### Tools
#### nslookup
![nslookup](/imgs/cka/networking-pre-15.png)
- to query hostname from a DNS server
- does not consider the entries in the local `/etc/hosts` file
  - entry for web application has to be present in DNS server.

#### dig
![dig](/imgs/cka/networking-pre-16.png)
- give more information


## Network Namespaces
### Process Namespace
![process-name](/imgs/cka/networking-pre-17.png)
- container host
  - create a container, want to make sure that it is isolated
  - create a special room, called namespace
  - only sees the processes run by container
- underlying host
  - has visibility in to all of the proceeses
    - including those running inside containers

### Network namespace
![network-namespace](/imgs/cka/networking-pre-18.png)
- host has its own intercates that connect to the local area network
- own routing table, arp table
  - information about rest of the network

#### 방법
1. create a container
2. create a network namespace for it
   - it has no visibility to any network related infromation on the host within its namespace
3. the container have its own virtual interfaces, routing and arp tables

#### Create network namespace
![create-ns](/imgs/cka/networking-pre-19.png)
- 생성
  - `ip netns add red`
  - `ip netns add blue`
- 상태
  - `ip netns`

#### Exec in network ns
![exec-ns](/imgs/cka/networking-pre-20.png)
- interfaces
  - to list interfaces on my host
    - `ip link`
  - to list interfaces on network namespace
    - `ip netns exec red ip link`
    - `ip -n red link`
- arp & route table
  - no entries in namespace
  - these namespaces has no connectivity to other network

### Connect
![connect-ns](/imgs/cka/networking-pre-21.png)
- connect namespaces together using a virtual ehternet pair or a virtual cable

#### 방법
1. create the cable
   - `ip link add veth-red type veth peer name veth-blue`
2. attach each interface to the appropriate namespace
   - `ip link set veth-red netns red`
   - `ip link set veth-blue netns blue`
3. assign IP addresses to each of these namespaces
   - `ip -n red addr add 192.168.15.1 dev veth-red`
   - `ip -n blue addr add 192.168.15.2 dev veth-blue`
4. bring up the interface
   - `ip -n red link set veth-red up`
   - `ip -n blue link set veth-blue up`

#### 확인
- arp table is created
  ![arp-table](/imgs/cka/networking-pre-22.png)

### Virtual network
- many pods → need a switch
- create a virtual switch
- how to create virtual networks?
  - 여러 open source들
  - 여기선 linux bridge

#### 생성
![create-vns](/imgs/cka/networking-pre-23.png)
1. add a new interface to the host
   - `ip link add v-net-0 type bridge`
2. check
   - `ip link`
3. turn up
   - `ip link set de v-net-0 up`
4. connect the namespaces to new virtual network switch

#### 연결
![connect-vns](/imgs/cka/networking-pre-24.png)
1. create a cable
   - `ip link add veth-red type veth peer name veth-red-br`
   - `ip link add veth-blue type veth peer name veth-blue-br`
2. attach one end of this interface to the red namespace
   - `ip link set veth-red netns red`
   - `ip link set veth-red-br master v-net-0`
3. same job with 2 in blue
   - `ip link set veth-blue netns blue`
   - `ip link set veth-blue-br master v-net-0`
4. assign ip address
   - `ip -n red addr add 192.168.15.1 dev veth-red`
   - `ip -n blue addr add 192.168.15.2 dev veth-blue`
5. turn devices up
   - `ip -n red link set veth-red up`
   - `ip -n blue link set veth-blue up`

#### Host and namespace
![host-vns](/imgs/cka/networking-pre-25.png)
- try trired to reach one of these interfaces in this namespaces from my host
  - → doesn't work
- connect interface with host
  - `ip addr add 192.168.15.5/25 dev net-0`

### Localhost
#### 상황
![localhost-status](/imgs/cka/networking-pre-26.png)
- can't reach the outside world and no one from outside world can not reach hosted inside
- how to configure this bridge to reach the network through the internet port?
  - → need to add an entry in to the routing table to provide a gateway or door to the outside world
- localhost
  - has all these namesapces
  - has an interface to attach to the private network
  - a gateway that connects the two networks toghter

#### 방법
![localhost-method](/imgs/cka/networking-pre-27.png)
1. add localhost to router
   - `ip netns exec blue ip route add 192.168.1.0/24 via 192.168.15.5`
   - host has two ip addresses
     - 192.168.15.5
     - external network: 192.168.1.2
2. forward
   - `iptables -t nat -A POSTROUT -s 192.168.15.0/25 -j MASQUERADE`
3. connect to internet
   - `ip netns exec blue ip route add default via 192.168.15.5`

## Docker Networking
### Single Docker host
- Has an ethernet interface at eth0 that connects to the local network with the IP address 192.168.1.10
- run a container, there is different networking option to choose from
  - none network
  - host network
  - bridge

#### None network
![docker-none](/imgs/cka/networking-pre-28.png)
- docker container is not attached to any network

#### Host network
![docker-host](/imgs/cka/networking-pre-29.png)
- container is attached to the host's network
- sharing host network, cannot use same port at same time

#### Bridge
![docker-bridge](/imgs/cka/networking-pre-30.png)

- internal private network is created which the docker host and containers attach to
- ip address: 172.17.0.0

### Bridge
#### 상태
![bridge-status](/imgs/cka/networking-pre-31.png)
- `docker network ls`
  - docker calls the network by the name bridge
- `ip link`
  - host the name of network is created with docker0
  - interface is currently down
- `ip addr`

#### 생성

![bridge-crate](/imgs/cka/networking-pre-32.png)
- container is created
- docker creates a network namepsace for it
- `ip netns`
- `docker inspect ~~`

### Attach container to the bridge
![bridge-attach](/imgs/cka/networking-pre-33.png)
1. creates a virtual cable
   - `ip link` on docker host
     - end of the interface which is attached to the local bridge docker0
   - `ip -n ~~ link`
2. check the ip
   - `ip -n ~~ addr`

### Port Mapping
![port-mapping](/imgs/cka/networking-pre-34.png)
- port publishing or port mapping options
  - to allow to access the applications hosted on containers
  - `docker run -p 8080:80 nginx`

#### 방법
![port-mapping-how](/imgs/cka/networking-pre-35.png)
- requirements
  - forward traffic coming in one port to another port on the server

1. add rules to docker chain
2. set destination to include the containers IP
3. see the rule
   - `iptables -nvL -t nat`
   ![port-mapping-rule](/imgs/cka/networking-pre-36.png)

## Cluster Networking
### IP & FQDN
1. k8s cluster consists of master and worker nodes
2. each node must have at least 1 interface connected to a network
3. each interface must have an address configured
4. the host must have a unique hostname set, unique MAC address
   - note this if you created the VMs by cloning from existing ones.

### PORTS
There are some ports to be opened.

#### Master
- `kube-api` - 6443
- `kubelet` - 10250
- `kube-scheduler` - 10251
- `kube-controller-manager` - 10252
- `etcd` 
  - 2379
  - 2380, if multiple master nodes

#### Worker
- `kubelet` - 10250
- `services` - 30000~32767

### Practice
#### What is the network interface configured for cluster connectivity on the master node? node-to-node communication
Run: `kubectl get nodes -o wide` to see the IP address assigned to the controlplane node.
```
root@controlplane:~# kubectl get nodes controlplane -o wide
NAME           STATUS   ROLES                  AGE     VERSION   INTERNAL-IP   EXTERNAL-IP   OS-IMAGE             KERNEL-VERSION   CONTAINER-RUNTIME
controlplane   Ready    control-plane,master   4h46m   v1.20.0   10.3.116.12   <none>        Ubuntu 18.04.5 LTS   5.4.0-1041-gcp   docker://19.3.0
```
In this case, the internal IP address used for node for node to node communication is `10.3.116.12`.

**Important Note** : The result above is just an example, the node IP address will vary for each lab.

Next, find the network interface to which this IP is assigned by making use of the `ip a` command:
```
root@controlplane:~# ip a | grep -B2 10.3.116.12
16476: eth0@if16477: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 1450 qdisc noqueue state UP group default
    link/ether 02:42:0a:03:74:0c brd ff:ff:ff:ff:ff:ff link-netnsid 0
    inet 10.3.116.12/24 brd 10.3.116.255 scope global eth0
root@controlplane:~#
```
Here you can see that the interface associated with this IP is eth0 on the host.

#### What is the MAC address assigned to node01?

```bash
> arp node01
Address                  HWtype  HWaddress           Flags Mask            Iface
10.153.142.8             ether   02:42:0a:99:8e:07   C                     eth0
```
