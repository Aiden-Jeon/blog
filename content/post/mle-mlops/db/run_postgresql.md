---
title: DB-1) Run Postgresql Server
categories: [mle-mlops]
tags: ["postgres", "docker", "db"]
toc: true
date: 2022-09-15
author: Jongseob Jeon
---

[Github](https://github.com/Aiden-Jeon/mle-mlops/tree/main/01_db) 에서 해당 내용에 대해서 확인할 수 있습니다.

## Overview
### 목표

- 도커를 이용해 postgres 서버를 띄워봅시다.
- `psql` 을 이용해 생성된 username과 그 역할을 확인합니다.

### 요구사항

1. Postgresql User Requirements
    - Username: `postgres`
    - Role: `Superuser`
    - Password: `mypassword`
2. Docker Run Option
    - port forwarding: 5432:5432
3. psql cli tool 을 통해 생성한 데이터 베이스에 접속합니다.
    - 다음 option을 사용해 접속합니다.
        `-h, --host=HOSTNAME database server host or socket directory (default: "local socket")`
        `-U, --username=USERNAME database user name (default: "...")`
4. psql 내부에서 다음 내용을 확인합니다.
    - 생성한 Role name 과 role
---

## Docker

다음 명령어를 통해 기본적인 postgresql server를 실행시킬 수 있습니다.  
`-d` 옵션을 통해 실행된 도커와의 interaction을 주지 않습니다.
```bash
docker run -d postgres
```

### 1. Postgresql User Requirements
#### Username & Role

Postgresql 서버를 최초 실행할 경우 생성되는 username은 `postgres` 이며 그 role이 `Superuser` 입니다.  
그렇기 때문에 해당 요구사항은 자동적으로 만족됩니다.

#### Password

컨테이너를 run할 때 `-e` 옵션을 통해 필요한 environment를 제공할 수 있습니다.  
최초 `postgres` 유저의 비밀번호는 `POSTGRES_PASSWORD` 를 통해 설정할 수 있습니다.  
이제 위의 명령어에 다음 내용을 추가합니다.
```bash
docker run -e POSTGRES_PASSWORD=mypassword -d postgres
```

### 2. Docker Run Option

컨테이너를 run할 때 `-p` 옵션을 통해 지정한 port를 localhost 포트로 맵핑할 수 있습니다.
```bash
docker run -p 5432:5432 -e POSTGRES_PASSWORD=mypassword -d postgres
```

### 3. psql cli tool 을 통해 생성한 데이터 베이스에 접속합니다.

`psql -h <host> -p <port> -U <username> <database>`

- `-h` 를 통해 host를 지정하고 `-p`를 통해 포트를 지정합니다.이 때 host는 docker 에서 포트 포워딩을 했기 때문에 `localhost` 를 입력합니다.
- `-U` 옵션을 통해 접속해 시도할 유저 이름을 입력할 수 있습니다.  여기서는 `postgres`를 통해 접속합니다.  
- database 이름 또한 따로 지정하지 않았기 때문에 기본값인 `postgres`를 입력합니다.


위의 내용을 바탕으로 명령어를 작성하면 다음과 같습니다.
```bash
psql -h localhost -p 5432 -U postgres postgres
```

위 명령어를 실행하면 다음과 같이 비밀번호를 요구합니다.
```zsh
> psql -h localhost -p 5432 -U postgres postgres
Password for user postgres:
```

비밀번호를 입력하고 접속하면 다음과 같이 출력됩니다.
```zsh
> psql -h localhost -p 5432 -U postgres postgres
Password for user postgres:
psql (14.3, server 14.5 (Debian 14.5-1.pgdg110+1))
Type "help" for help.

postgres=#
```

### 4. psql 내부에서 다음 내용을 확인합니다.

`\du` 를 통해 현재 데이터베이스 내의 Role name과 role을 확인할 수 있습니다.

```
postgres=# \du
                                   List of roles
 Role name |                         Attributes                         | Member of
-----------+------------------------------------------------------------+-----------
 postgres  | Superuser, Create role, Create DB, Replication, Bypass RLS | {}
```

## Docker Compose

### docker-compose.yaml
`docker-compose.yaml` 파일을 생성하고 아래 항목을 작성합니다.

```yaml
services:
  db:
    image: postgres:latest
    ports:
      - "5432:5432"
    environment:
      POSTGRES_PASSWORD: "mypassword"
```

`image`는 `postgres` 의 `latest` tag를 사용합니다.  
`ports` 을 통해 port-forwarding 을 설정합니다.  
앞서 docker 에서 사용한 `-e` 옵션은 `environment`에 설정할 수 있습니다.
docker에서와 같이 `POSTGRES_PASSWORD: "mypassword"` 를 입력합니다.

### docker-compose up

아래 명령어를 이용해 컨테이너들을 실행할 수 있습니다.
```bash
docker-compose up
```

실행하면 아래와 같은 메세지가 출력됩니다.
```zsh
> docker-compose up
[+] Running 1/0
 ⠿ Container db-db-1  Created                                                                                                                                                                          0.0s
Attaching to db-db-1
db-db-1  | The files belonging to this database system will be owned by user "postgres".
db-db-1  | This user must also own the server process.
db-db-1  | 
db-db-1  | The database cluster will be initialized with locale "en_US.utf8".
db-db-1  | The default database encoding has accordingly been set to "UTF8".
db-db-1  | The default text search configuration will be set to "english".
db-db-1  | 
db-db-1  | Data page checksums are disabled.
db-db-1  | 
db-db-1  | fixing permissions on existing directory /var/lib/postgresql/data ... ok
db-db-1  | creating subdirectories ... ok
db-db-1  | selecting dynamic shared memory implementation ... posix
db-db-1  | selecting default max_connections ... 100
db-db-1  | selecting default shared_buffers ... 128MB
db-db-1  | selecting default time zone ... Etc/UTC
db-db-1  | creating configuration files ... ok
db-db-1  | running bootstrap script ... ok
db-db-1  | performing post-bootstrap initialization ... ok
db-db-1  | syncing data to disk ... initdb: warning: enabling "trust" authentication for local connections
db-db-1  | You can change this by editing pg_hba.conf or using the option -A, or
db-db-1  | --auth-local and --auth-host, the next time you run initdb.
db-db-1  | ok
db-db-1  | 
db-db-1  | 
db-db-1  | Success. You can now start the database server using:
db-db-1  | 
db-db-1  |     pg_ctl -D /var/lib/postgresql/data -l logfile start
db-db-1  | 
db-db-1  | waiting for server to start....2022-09-16 04:16:32.249 UTC [47] LOG:  starting PostgreSQL 14.5 (Debian 14.5-1.pgdg110+1) on aarch64-unknown-linux-gnu, compiled by gcc (Debian 10.2.1-6) 10.2.1 20210110, 64-bit
db-db-1  | 2022-09-16 04:16:32.251 UTC [47] LOG:  listening on Unix socket "/var/run/postgresql/.s.PGSQL.5432"
db-db-1  | 2022-09-16 04:16:32.254 UTC [48] LOG:  database system was shut down at 2022-09-16 04:16:32 UTC
db-db-1  | 2022-09-16 04:16:32.258 UTC [47] LOG:  database system is ready to accept connections
db-db-1  |  done
db-db-1  | server started
db-db-1  | 
db-db-1  | /usr/local/bin/docker-entrypoint.sh: ignoring /docker-entrypoint-initdb.d/*
db-db-1  | 
db-db-1  | waiting for server to shut down...2022-09-16 04:16:32.362 UTC [47] LOG:  received fast shutdown request
db-db-1  | .2022-09-16 04:16:32.364 UTC [47] LOG:  aborting any active transactions
db-db-1  | 2022-09-16 04:16:32.365 UTC [47] LOG:  background worker "logical replication launcher" (PID 54) exited with exit code 1
db-db-1  | 2022-09-16 04:16:32.366 UTC [49] LOG:  shutting down
db-db-1  | 2022-09-16 04:16:32.381 UTC [47] LOG:  database system is shut down
db-db-1  |  done
db-db-1  | server stopped
db-db-1  | 
db-db-1  | PostgreSQL init process complete; ready for start up.
db-db-1  | 
db-db-1  | 2022-09-16 04:16:32.478 UTC [1] LOG:  starting PostgreSQL 14.5 (Debian 14.5-1.pgdg110+1) on aarch64-unknown-linux-gnu, compiled by gcc (Debian 10.2.1-6) 10.2.1 20210110, 64-bit
db-db-1  | 2022-09-16 04:16:32.478 UTC [1] LOG:  listening on IPv4 address "0.0.0.0", port 5432
db-db-1  | 2022-09-16 04:16:32.478 UTC [1] LOG:  listening on IPv6 address "::", port 5432
db-db-1  | 2022-09-16 04:16:32.481 UTC [1] LOG:  listening on Unix socket "/var/run/postgresql/.s.PGSQL.5432"
db-db-1  | 2022-09-16 04:16:32.484 UTC [59] LOG:  database system was shut down at 2022-09-16 04:16:32 UTC
db-db-1  | 2022-09-16 04:16:32.488 UTC [1] LOG:  database system is ready to accept connections
```

다만 이렇게 실행할 경우 커널을 종료할 수 있는 방법이 없습니다.  
그렇기 때문에 `-d` 옵션을 통해 컨테이너만 실행되게 합니다.
```bash
docker-compose up -d
```
