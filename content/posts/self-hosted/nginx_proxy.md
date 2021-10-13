---
title: Nginx Proxy Manager 설정하기
categories: [self-hosted]
tags: ["self-hosted", "nginx"]
toc: true
date: 2021-10-13
author: Jongseob Jeon
---

이번 포스트에서는 생성한 인스턴스를 nginx proxy로 관리하는 방법에 대해서 설명합니다.

## Ubuntu Instance 기본 설정
### 1. 기본 패키지 업데이트
```bash
sudo apt update && sudo apt -y upgrade
```

### 2. Swap Memory 생성
Free tier의 메모리는 1G 이므로 스왑 메모리를 생성하도록 하겠습니다.
```bash
sudo fallocate -l 2G /swapfile
sudo chmod 600 /swapfile
sudo mkswap /swapfile
```

생성된 스왑 메모리를 확인합니다.
```bash
free -h
```

재부팅해도 스왑 메모리가 유지되도록 수정합니다.
```bash
sudo vim /etc/fstab
```

아래 내용을 추가합니다.
```
/swapfile swap swap defaults 0 0
```

추가한 후의 모습은 다음과 같습니다.
```sh
LABEL=cloudimg-rootfs   /        ext4   defaults        0 0
LABEL=UEFI      /boot/efi       vfat    defaults        0 0
/swapfile swap swap defaults 0 0
```

### 3. 시간 설정
```bash
sudo timedatectl set-timezone Asia/Seoul
```

## Nginx Proxy Manager
### 1. iptable 초기화
oracle에서 기본적으로 설치한 iptable 규칙을 초기화합니다.
```bash
sudo iptables -F
sudo iptables -X
sudo netfilter-persistent save
sudo netfilter-persistent reload
```

### 2. docker 설치
[공식 홈페이지](https://docs.docker.com/engine/install/ubuntu/)를 참고해 docker를 설치합니다.
1. Setup repository
    ```bash
    sudo apt-get update
    sudo apt-get install \
        apt-transport-https \
        ca-certificates \
        curl \
        gnupg \
        lsb-release
    ```
2. Add Docker's official GPG key
    ```bash
    curl -fsSL https://download.docker.com/linux/ubuntu/gpg | sudo gpg --dearmor -o /usr/share/keyrings/docker-archive-keyring.gpg
    ```
    ```bash
     echo \
        "deb [arch=$(dpkg --print-architecture) signed-by=/usr/share/keyrings/docker-archive-keyring.gpg] https://download.docker.com/linux/ubuntu \
        $(lsb_release -cs) stable" | sudo tee /etc/apt/sources.list.d/docker.list > /dev/null
    ```
3. Install docker
    ```bash
    sudo apt-get update
    sudo apt-get install docker-ce docker-ce-cli containerd.io
    ```
4. 실행 확인
    ```bash
    docker ps
    ```
    - 실행이 안될 경우 [포스트]({{< relref "posts/kubernetes/k3s_prerequisite###1.3 권한 문제 해결" >}})를 참고하세요.

### 3. docker-compose 설치
[공식 홈페이지](https://docs.docker.com/compose/install/)를 참고해 docker-compose를 설치합니다.
1. Download Docker Compose
    ```bash
    sudo curl -L "https://github.com/docker/compose/releases/download/1.29.2/docker-compose-$(uname -s)-$(uname -m)" -o /usr/local/bin/docker-compose
    ```
2. Apply permission
    ```bash
    sudo chmod +x /usr/local/bin/docker-compose
    ```
3. Add Path
    ```bash
    sudo ln -s /usr/local/bin/docker-compose /usr/bin/docker-compose
    ```
4. Test
    ```bash
    docker-compose --version
    ```

### 4. `docker-compose.yaml` 작성
1. 폴더 생성
    ```bash
    mkdir ~/npm; cd ~/npm
    ```
2. `docker-compose.yaml` 작성
```bash
cat > docker-compose.yml << EOF
version: "3"
services:
  app:
    image: 'jc21/nginx-proxy-manager:latest'
    restart: always
    network_mode: "host"
    environment:
      # These are the settings to access your db
      DB_MYSQL_HOST: localhost
      DB_MYSQL_PORT: 3306
      DB_MYSQL_USER: "npm"
      DB_MYSQL_PASSWORD: "npm"
      DB_MYSQL_NAME: "npm"
      # If you would rather use Sqlite uncomment this
      # and remove all DB_MYSQL_* lines above
      # DB_SQLITE_FILE: "/data/database.sqlite"
      # Uncomment this if IPv6 is not enabled on your host
      # DISABLE_IPV6: 'true'
    volumes:
      - /home/ubuntu/npm/data:/data
      - /home/ubuntu/npm/letsencrypt:/etc/letsencrypt
    depends_on:
      - db
  db:
    image: 'jc21/mariadb-aria:latest'
    restart: always
    network_mode: "host"
    environment:
      MYSQL_ROOT_PASSWORD: 'npm'
      MYSQL_DATABASE: 'npm'
      MYSQL_USER: 'npm'
      MYSQL_PASSWORD: 'npm'
    volumes:
      - /home/ubuntu/npm/mysql:/var/lib/mysql
EOF
```
3. docker-compose 실행
    ```bash
    docker-compose up -d
    ```
4. 실행 확인
    ```bash
    docker-compose logs
    ```
    로그의 제일 마지막에 다음과 같은 출력이 있다면 정상적으로 실행된 것입니다.
    ```bash
    app_1  | [10/13/2021] [9:53:02 AM] [SSL      ] › ℹ  info      Renewing SSL certs close to expiry...
    app_1  | [10/13/2021] [9:53:02 AM] [IP Ranges] › ℹ  info      IP Ranges Renewal Timer initialized
    app_1  | [10/13/2021] [9:53:02 AM] [Global   ] › ℹ  info      Backend PID 246 listening on port 3000 ...
    app_1  | [10/13/2021] [9:53:03 AM] [Nginx    ] › ℹ  info      Reloading Nginx
    app_1  | [10/13/2021] [9:53:03 AM] [SSL      ] › ℹ  info      Renew Complete
    ```

### 5. DNS 설정
무료로 이용할 수 있는 [duckdns](https://www.duckdns.org/)를 사용해 DNS를 연결하겠습니다.

1. 로그인 한 뒤 사용할 dns를 추가합니다.
    ![add dns](/imgs/self-hosted/nginx_proxy-0.png)
2. oracle에서 받은 public ip를 입력합니다.
    ![update current ip](/imgs/self-hosted/nginx_proxy-1.png)

### 6. Nginx Proxy 연결
`http://<Public IP>:81`로 접속합니다.

다음과 같이 화면이 출력됩니다.
![login](/imgs/self-hosted/nginx_proxy-2.png)

기본 아이디와 비밀번호는 다음과 같습니다.
- id: admin@example.com
- pw: changeme

접속 후 아이디와 비밀번호를 바꿔줍니다.

### 7. Nginx DNS 연결
1. Proxy Hosts를 선택합니다.
    ![select proxy hosts](/imgs/self-hosted/nginx_proxy-3.png)
2. Add Proxy Host를 선택합니다.
    ![add proxy hosts](/imgs/self-hosted/nginx_proxy-4.png)
3. Nginx Proxy Host를 접속한 dns를 입력해줍니다.
    저는 npm을 subdomain으로 생성했습니다.
    ![select dns](/imgs/self-hosted/nginx_proxy-5.png)
4. 설정한 dns로 접속되는지 확인합니다.
5. https 접속을 위한 ssl를 설정해주어야 합니다.
    ![edit](/imgs/self-hosted/nginx_proxy-6.png)
6. 다음과 같은 내용으로 작성합니다.   
    ![ssl](/imgs/self-hosted/nginx_proxy-7.png) 
7. oracle cloud에서 81번 포트를 삭제합니다.
    ![erase port](/imgs/self-hosted/nginx_proxy-8.png) 
