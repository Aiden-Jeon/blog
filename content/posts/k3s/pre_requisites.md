---
title: Pre-Requisites for k3s Setup
categories: [k3s]
tags: ["k8s"]
toc: true
date: 2021-10-01
author: Jongseob Jeon
---

**Kubeflow 설치 시리즈**
1. [Pre-Requisites for k3s Setup]({{< relref "posts/k3s/pre_requisites" >}})
2. [k3s Installation]({{< relref "posts/k3s/install_k3s" >}})
3. [Kubeflow Installation]({{< relref "posts/k3s/install_kubeflow" >}})

---
이번 포스트에서는 k3s를 설정하기 전에 필요한 것들을 세팅하는 과정에 대해서 설명합니다.  
이번 포스트에서 진행하는 과정은 서버 데스크탑, 로컬 노트북 두 대의 장비가 있다고 가정하고 있습니다.

---

# On Server
사용하는 서버 스s펙은 다음과 같습니다.
```
OS: Ubuntu 20.04 LST
GPU: RTX 2060 x2
CPU: Intel(R) Core(TM) i7-9700K CPU @ 3.60GHz
MEM: 64G
```

서버에 설치할 것 들은 3가지 입니다.
1. Docker
2. Nvidia Driver
3. WireGuard VPN (사용할 경우)

## 1. Docker
### 1.1 Docker 설치  
k3s는 snap으로 설치된 docker를 사용할 수 없습니다.  

아래 링크를 통해 직접 빌드를 해야 합니다.  
https://docs.docker.com/engine/install/ubuntu/

### 1.2 설치 확인
링크를 통해 설치가 끝나면 sudo 권한이 있는지 확인해야 합니다.

우선 아래 명령어로 권한이 있는지 확인합니다.
```bash
docker ps
```

권한이 없는 경우 아래와 같은 에러를 출력합니다.
```bash
docker: Got permission denied while trying to connect to the Docker daemon socket at unix:///var/run/docker.sock: Post http://%2Fvar%2Frun%2Fdocker.sock/v1.35/containers/create: dial unix /var/run/docker.sock: connect: permission denied.
```

### 1.3 권한 문제 해결
권한이 없는 경우 그룹을 생성해 추가를 해주어야 합니다.  
자세한 내용은 [링크](https://docs.docker.com/engine/install/linux-postinstall/)를 통해 확인할 수 있습니다.

1. 우선 `docker` 그룹을 생성합니다.
    ```bash
    sudo groupadd docker
    ```
2. 유저를 `docker` 그룹에 추가합니다.
    ```bash
    sudo usermod -aG docker $USER
    ```
3. 권한을 확인합니다
    ```bash
    docker ps
    ```


다만 이 과정을 통해서도 권한이 없다는 에러가 나올 수 있습니다.
이 경우에는 위 과정이 끝난 후 reboot을 하면 해결되는 경우도 있습니다.
```bash
sudo reboot
```

## 2. Nvidia Driver
제가 사용하는 서버는 gpu를 이용하기 때문에 nvidia driver의 설치가 필요합니다.  
`ubuntu-drivers-common`를 이용하면 쉽게 설치할 수 있습니다.

### 2.1 `ubuntu-drivers-common` 설치
1. `ubuntu-drivers-common` 설치
    ```bash
    sudo apt install ubuntu-drivers-common
    ```
2. `auto install` 실행
    ```bash
    sudo ubuntu-drivers autoinstall
    ```
### 2.2 Nvidia Docker 세팅
Docker가 gpu를 사용할 수 있도록 설정해주어야 합니다.
자세한 내용은 [링크](https://github.com/NVIDIA/k8s-device-plugin)를 통해 확인할 수 있습니다.

1. Nvidia Docker 설치
    ```bash
    distribution=$(. /etc/os-release;echo $ID$VERSION_ID)
    curl -s -L https://nvidia.github.io/nvidia-docker/gpgkey | sudo apt-key add -
    curl -s -L https://nvidia.github.io/nvidia-docker/$distribution/nvidia-docker.list | sudo tee /etc/apt/sources.list.d/nvidia-docker.list

    sudo apt-get update && sudo apt-get install -y nvidia-docker2
    sudo systemctl restart docker
    ```
2. Docker Config 설정  
    sudo 권한으로 `/etc/docker/daemon.json` 파일을 열어줍니다.
    ```bash
    sudo vim /etc/docker/daemon.json
    ```
    원본의 내용을 지우고 아래 내용으로 수정합니다.
    ```json
    {
        "default-runtime": "nvidia",
        "runtimes": {
            "nvidia": {
                "path": "/usr/bin/nvidia-container-runtime",
                "runtimeArgs": []
            }
        }
    }
    ```
3. Reboot
    ```bash
    sudo reboot
    ```
4. `nvidia-smi` 확인  
   설치가 되었는지 확인합니다.
    ```bash
    nvidia-smi
    ```

## WireGuard VPN (사용할 경우)
저는 WireGuard VPN을 이용하고 있기 때문에 서버에 추가적인 VPN 작업을 해주었습니다.
1. WireGuard Config 발급  
    WireGuard에서 Ubuntu OS의 config를 생성합니다.
2. WireGuard 설치
   ```bash
   sudo apt-get install -y resolvconf wireguard
   ```
3. Config 복사  
   발급받은 config를 서버의 `/etc/wireguard/wg0.conf`에 옮겨 줍니다.
   저는 ssh를 이용해 전송 후 복사하였습니다.
   ```bash
   sudo cp wg0.conf /etc/wireguard/wg0.conf
   ```
    ssh가 불가능할 경우 `sudo vim /etc/wireguard/wg0.conf`을 이용해 직접 수정해도 됩니다.
4. Systemctl 설정
   ```bash
   sudo systemctl enable wg-quick@wg0
   sudo systemctl start wg-quick@wg0
   sudo systemctl status wg-quick@wg0
   ```
5. 인터페이스 확인
   ```bash
   ifconfig wg0
   ```
6. Ping 확인
   ```bash
   ping 172.25.0.1
   ```

WireGuard config를 수정할 경우 restart를 해줍니다.
```bash
sudo systemctl restart wg-quick@wg0
```

# On Local
로컬에서 설치 할 것 들은 2가지 입니다.
1. `kubectl`
2. `krew`

## 1. `kubectl`
각 OS에 맞는 `kubectl`을 설치합니다.  
https://kubernetes.io/docs/tasks/tools/

## 2. `krew`

1. `krew`설치  
   각 OS에 맞는 `krew`를 설치합니다.  
    https://krew.sigs.k8s.io/docs/user-guide/setup/install/
2. `ctx`설치
   ```bash
   kubectl krew install ctx
   ```
