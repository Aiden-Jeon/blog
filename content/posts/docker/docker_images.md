---
title: Docker Images
comment: true
categories: [docker]
tags: ["docker"]
toc: true
date: 2021-05-27
author: Jongseob Jeon
---

## Docker Images
### How to create my own image?
#### Order
1. OS - Ubuntu
2. Update apt repo
3. Install dependencies using apt
4. Install Python dependencies using pip
5. Copy source code to /opt folder
6. Run the web server using "flask" command

#### Dockerfile
```docker
FROM Ubuntu

RUN apt-get update && apt-get -y install python

RUN pip install flask flask-mysql

COPY . /opt/source-code

ENTRYPOINT FLASK_APP=/opt/source-code/app.py flask run
```
- `INSTRUCTION ARGUMENT`

#### Build
- `docker build . -f Dockerfile -t <image name>`

#### Push
- `docker push <image name>`

### Layered architecture

![layered-architecture](/imgs/docker/docker-image-1.png)

- each layer only store the changes from the previous layer
- `docker history <image name>`

## Environment Variables

- `-e`
  - `docker run -e <KEY>=<value> <image name>`

### Inspect Environment Variable

- `docker inspect <container name | id>`

## Command vs Entrypoint

### 설명

#### Command
- `CMD`
- instruction the command line parameters passed will get replaced entirely.

#### Entrypoint
- `ENTRYPOINT`
- command line parameters will get appended.
- if parameter is not given, it will get error that the operand is missing
  - How to give and default value for entrypoint?
    ```Docker
    ENTRYPOINT ["command"]
    CMD ["param1"]
    ```
- how to override entrypoint?
  - `docker run --entrypoint <executable> <image-name> <param>`

### 명령어
- `CMD`
  - `CMD command param1`
  - `CMD ["command", "param1"]`
- `ENTRYPOINT`
  - `ENTRYPOINT ["command"]`