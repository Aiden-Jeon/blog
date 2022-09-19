---
title: DB-4) Insert Row Loop Dockerfile
categories: [mle-mlops]
tags: ["postgres", "docker", "db"]
toc: true
date: 2022-09-19 14:50:00+08:00
author: Jongseob Jeon
---

[Github](https://github.com/Aiden-Jeon/mle-mlops/tree/main/01_db) 에서 해당 내용에 대해서 확인할 수 있습니다.

## Overview
### 목표
- 앞서 작성한 스크립트를 실행할 수 있는 Dockerfile을 작성하고 이미지를 빌드합니다.

### 요구사항
1. 작성한 script를 build할 수 있는 dockerfile을 작성합니다.
2. 작성한 dockerfile을 이용해 이미지를 build 합니다.
3. 빌드된 이미지를 실행합니다.
    - 이미지가 실행될 때의 host에 대해서 생각해 봅니다.
4. `psql` 등을 이용해 DB에 데이터가 계속해서 쌓이고 있는지 확인합니다.

---

## Docker
### Dockerfile

앞서 작성한 스크립트를 실행할 수 있는 Dockerfile을 작성합니다.  

```Dockerfile
FROM python:3.9-slim

WORKDIR /app

RUN apt-get update && apt-get install -y \
    build-essential \
    software-properties-common \
    git \
    && rm -rf /var/lib/apt/lists/*

RUN pip install -U pip &&\
    pip install scikit-learn pandas psycopg2-binary

COPY insert_row_loop.py insert_row_loop.py

ENTRYPOINT ["python", "insert_row_loop.py"]
```

1. base가 되는 이미지는 `python:3.9-slim`을 사용하도록 하겠습니다.
2. 복사할 파일을 같은 곳에 묶어 두기 위해서 `WORKDIR` 은 `/app` 으로 설정합니다.  
3. 다음으로, pip install 할 수 있는 apt pacakge들을 설치합니다.  
4. 앞서 만든 스크립트에서 필요한 패키지인 `scikit-learn`, `pandas`와 `psycopg2-binary` 를 설치합니다.  
5. `insert_row_loop.py` 를 복사합니다.
6. 이미지가 실행될 때 실행할 명령어를 입력합니다.

### Build Image
다음 명령어를 통해 docker 이미지를 build합니다.

```bash
docker build . -t insert_row_loop
```

### Run Image
Build된 이미지를 실행합니다.
```bash
docker run insert_row_loop
```

하지만, 위 이미지를 실행하면 다음과 같은 에러가 출력되면 실행되지 않습니다.

```zsh
Traceback (most recent call last):
  File "/app/insert_row_loop.py", line 45, in <module>
    db_connect = psycopg2.connect(host="localhost", database="postgres", user="postgres", password="mypassword")
  File "/usr/local/lib/python3.9/site-packages/psycopg2/__init__.py", line 122, in connect
    conn = _connect(dsn, connection_factory=connection_factory, **kwasync)
psycopg2.OperationalError: could not connect to server: Connection refused
	Is the server running on host "localhost" (127.0.0.1) and accepting
	TCP/IP connections on port 5432?
could not connect to server: Cannot assign requested address
	Is the server running on host "localhost" (::1) and accepting
	TCP/IP connections on port 5432?
```

이런 이유가 나오는 이유는 도커의 네트워킹과 관련되어 있습니다.  
기존의 스크립트를 실행할 때는 `localhost` 라는 네트워크를 공유하고 있습니다. 그래서 `localhost:5432` 로의 통신이 가능했습니다. 
하지만 이미지가 실행되면 독립된 네트워크를 갖게 됩니다. 즉, 실행된 컨테이너 입장에서는 `localhost:5432`란 없는 주소입니다.  
이를 위해서는 실행되는 두 컨테이너 사이에 네트워크를 연결시켜주어야 합니다.

## Docker Network
### Create Network

아래 명령어를 실행하면 현재 docker에서 사용할 수 있는 목록들이 나옵니다.
```bash
docekr network ls
```

따로 설정한 적이 없다면 아래와 같이 3개의 네트워크 리스트가 나옵니다.
```zsh
> docker network ls
NETWORK ID     NAME         DRIVER    SCOPE
b28719341e4b   bridge       bridge    local
0d5dc37e7a9d   host         host      local
703a7cd222a5   none         null      local
```

위 3개의 네트워크들은 도커가 실행되면 기본으로 생성되는 네트워크들입니다. 도커에서는 디폴트 네트워크를 이용하는 것 보다는 사용자가 직접 네트워크를 생성해 사용하는 것을 권장하고 있습니다.

아래 명령어를 통해 네트워크를 생성합니다.
```bash
docker network create my-network
```

네트워크가 생성되었는지 확인해봅니다.
```zsh
> docker network ls
NETWORK ID     NAME         DRIVER    SCOPE
b28719341e4b   bridge       bridge    local
0d5dc37e7a9d   host         host      local
a6bc214efdb5   my-network   bridge    local
703a7cd222a5   none         null      local
```

`docker network inspect <NETWORK_NAME>` 명령어를 통해 생성한 네트워크를 확인해 봅니다.
```zsh
> docker network inspect my-network
[
    {
        "Name": "my-network",
        "Id": "a6bc214efdb582d2e28f25be9f7ed835767757be0f20c15659461b683829b3af",
        "Created": "2022-09-19T07:16:58.08159655Z",
        "Scope": "local",
        "Driver": "bridge",
        "EnableIPv6": false,
        "IPAM": {
            "Driver": "default",
            "Options": {},
            "Config": [
                {
                    "Subnet": "172.22.0.0/16",
                    "Gateway": "172.22.0.1"
                }
            ]
        },
        "Internal": false,
        "Attachable": false,
        "Ingress": false,
        "ConfigFrom": {
            "Network": ""
        },
        "ConfigOnly": false,
        "Containers": {},
        "Options": {},
        "Labels": {}
    }
]
```
생성한 직후에는 연결된 컨테이너가 없어서 `Containers` 가 비어있는 걸 확인할 수 있습니다.

### Connect Conatiner to Network

`docker network connect <NETWORK_NAME> <CONTAINER_NAME>` 을 이용해 기존에 실행한 DB 컨테이너를 생성한 네트워크에 연결합니다.

실행중인 db 컨테이너의 이름을 확인합니다.
```bash
docker ps
CONTAINER ID   IMAGE             COMMAND                  CREATED      STATUS      PORTS                    NAMES
972d4b7e0c6e   postgres:latest   "docker-entrypoint.s…"   3 days ago   Up 3 days   0.0.0.0:5432->5432/tcp   db-db-1
```

실행한 DB 컨테이너의 이름인 `db-db-1` 를 `my-network`에 연결합니다.
```bash
docker network connect my-network db-db-1
```

연결이 되었는지 확인해 봅니다.
```zsh
docker network inspect my-network
[
    {
        "Name": "my-network",
        "Id": "a6bc214efdb582d2e28f25be9f7ed835767757be0f20c15659461b683829b3af",
        "Created": "2022-09-19T07:16:58.08159655Z",
        "Scope": "local",
        "Driver": "bridge",
        "EnableIPv6": false,
        "IPAM": {
            "Driver": "default",
            "Options": {},
            "Config": [
                {
                    "Subnet": "172.22.0.0/16",
                    "Gateway": "172.22.0.1"
                }
            ]
        },
        "Internal": false,
        "Attachable": false,
        "Ingress": false,
        "ConfigFrom": {
            "Network": ""
        },
        "ConfigOnly": false,
        "Containers": {
            "972d4b7e0c6e8505ba6e8df798b35999ca8320b52777d9840bff0d86e44068e3": {
                "Name": "db-db-1",
                "EndpointID": "054692d99556026e28d56eee40f45a839dbbdeab2bd5e853bef5618d25a86c49",
                "MacAddress": "02:42:ac:16:00:02",
                "IPv4Address": "172.22.0.2/16",
                "IPv6Address": ""
            }
        },
        "Options": {},
        "Labels": {}
    }
]
```

`Container`에 `db-db-1이 추가된 것을 확인할 수 있습니다.`

### Run docker with network

이제 데이터를 삽입하는 이미지를 실행합니다.  
`--network` 옵션을 이용해 실행하는 이미지에 네트워크를 추가할 수 있습니다.

```bash
docker run --network "my-network" insert_row_loop
```

위 명령어를 실행해도 여전히 에러가 발생합니다.
```zsh
> docker run --network "my-network" insert_row_loop
Traceback (most recent call last):
  File "/app/insert_row_loop.py", line 45, in <module>
    db_connect = psycopg2.connect(host="localhost", database="postgres", user="postgres", password="mypassword")
  File "/usr/local/lib/python3.9/site-packages/psycopg2/__init__.py", line 122, in connect
    conn = _connect(dsn, connection_factory=connection_factory, **kwasync)
psycopg2.OperationalError: could not connect to server: Connection refused
	Is the server running on host "localhost" (127.0.0.1) and accepting
	TCP/IP connections on port 5432?
could not connect to server: Cannot assign requested address
	Is the server running on host "localhost" (::1) and accepting
	TCP/IP connections on port 5432?
```

이는 연결된 네트워크가 `localhost`가 아닌 `my-network`로 연결되기 때문입니다.
script 파일에서 db 연결을 위한 host 부분을 아래와 같이 수정합니다.
이 때 사용하는 host 이름은 연결하려는 컨테이너의 이름입니다.

```python
db_connect = psycopg2.connect(host="my-network", database="postgres", user="postgres", password="mypassword")
```

다시 이미지를 빌드하고 실행합니다.  

```bash
docker build . -t insert_row_loop
docker run --network "my-network" insert_row_loop
```
***주의**: m1 맥북은 `--platform=linux/amd64` 를 입력해야 정성작으로 실행됩니다.*

이제 제대로 실행되는 것을 확인할 수 있습니다. `psql` 명령어를 이용해 계속해서 데이터가 들어오고 있는지 확인해 봅니다.

또한 network도 제대로 연결되었는지 확인해봅니다.
```zsh
> docker network inspect my-network
[
    {
        "Name": "my-network",
        "Id": "a6bc214efdb582d2e28f25be9f7ed835767757be0f20c15659461b683829b3af",
        "Created": "2022-09-19T07:16:58.08159655Z",
        "Scope": "local",
        "Driver": "bridge",
        "EnableIPv6": false,
        "IPAM": {
            "Driver": "default",
            "Options": {},
            "Config": [
                {
                    "Subnet": "172.22.0.0/16",
                    "Gateway": "172.22.0.1"
                }
            ]
        },
        "Internal": false,
        "Attachable": false,
        "Ingress": false,
        "ConfigFrom": {
            "Network": ""
        },
        "ConfigOnly": false,
        "Containers": {
            "490a5ae2948f82caf6df7d5147e83763b600629701de905dcb8eb732ed737ce9": {
                "Name": "vigorous_mirzakhani",
                "EndpointID": "fdec99903513900ae49864079da5d2fb3cb3c09d031a4880cefc2c551a586c1a",
                "MacAddress": "02:42:ac:16:00:03",
                "IPv4Address": "172.22.0.3/16",
                "IPv6Address": ""
            },
            "972d4b7e0c6e8505ba6e8df798b35999ca8320b52777d9840bff0d86e44068e3": {
                "Name": "db-db-1",
                "EndpointID": "054692d99556026e28d56eee40f45a839dbbdeab2bd5e853bef5618d25a86c49",
                "MacAddress": "02:42:ac:16:00:02",
                "IPv4Address": "172.22.0.2/16",
                "IPv6Address": ""
            }
        },
        "Options": {},
        "Labels": {}
    }
]
```

container가 두개로 늘어난 것을 확인할 수 있습니다.

### Docker Entrypoint

그런데 db 컨테이너의 이름이 변경이 될 때 마다 docker 를 매번 빌드할 수 없으므로 이를 argument로 받아서 실행될 수 있게 변경해 보겠습니다.

우선 파이썬 스크립트에서 다음과 같이 argument를 받을 수 있게 수정합니다.
```python
if __name__ == "__main__":
    from argparse import ArgumentParser

    parser = ArgumentParser()
    parser.add_argument("--host")
    args = parser.parse_args()
    db_connect = psycopg2.connect(host=args.host, database="postgres", user="postgres", password="mypassword")
    df = get_data()
    insert_row_loop(db_connect, df.sample(1))
```

도커 파일에서 argument를 받을 수 있도록 수정합니다.
```Dockerfile
FROM python:3.9-slim

WORKDIR /app

RUN apt-get update && apt-get install -y \
    build-essential \
    software-properties-common \
    git \
    && rm -rf /var/lib/apt/lists/*

RUN pip install -U pip &&\
    pip install psycopg2-binary scikit-learn pandas

COPY insert_row_loop.py insert_row_loop.py

ENTRYPOINT ["python", "insert_row_loop.py", "--host"]
```

새로운 이미지를 빌드하고 실행합니다. 이 때 argument로 db 컨테이너의 이름을 입력합니다.

```bash
docker build . -t insert_row_loop
docker run --network "my-network" insert_row_loop "db-db-1"
```
