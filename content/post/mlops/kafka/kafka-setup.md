---
title: Kafka Setup
categories: [kafka]
tags: ["kafka"]
toc: true
date: 2022-01-02
author: Jongseob Jeon
---

## Kafka CLI
카프카 명령어를 사용할 수 있는 CLI를 설정하는 과정에 대해서 설명합니다.  
kafka_2.13-2.8.1 를 다운로드 받습니다. (버전은 작성일 기준 도커로 이용할 수 있는 최신 버전입니다.)
```bash
wget https://dlcdn.apache.org/kafka/2.8.1/kafka_2.13-2.8.1.tgz
```

압축을 풉니다.
```bash
tar -xzf kafka_2.13-2.8.1.tgz
```

압축이 해제된 폴더를 홈 디렉토리로 이동합니다.

```bash
mv kafka_2.13-2.8.1 ~/
cd ~/
```

아래 항목이 실행되는지 확인합니다.

```bash
cd kafka_2.13-2.8.1
bin/kafka-topics.sh
```

kafka는 백엔드로 java를 사용하는데 자바가 설치되어 있지 않다면 다음 에러 메세지가 나옵니다.

```
The operation couldn’t be completed. Unable to locate a Java Runtime.
Please visit http://www.java.com for information on installing Java.
```

또한 카프카를 구동하기 위해 필요한 자바 버전은 8 로 본인의 자바 버전이 맞는지 확인합니다.

```bash
java -version
```

만약 호환되는 자바또는 자바 버전이 없을 경우 각 OS별로 자바를 설치합니다.  
Mac OS의 경우 brew를 통해 쉽게 설치할 수 있습니다.

```bash
brew tap homebrew/cask-versions
brew install --cask adoptopenjdk8
```

설치 후 자바 버전을 확인합니다.

```bash
java -version
```

다음과 같이 출력 되면 정상적으로 설치되었습니다.

```bash
openjdk version "1.8.0_292"
OpenJDK Runtime Environment (AdoptOpenJDK)(build 1.8.0_292-b10)
OpenJDK 64-Bit Server VM (AdoptOpenJDK)(build 25.292-b10, mixed mode
```

다시 아래 명령어를 실행합니다.

```bash
bin/kafka-topics.sh
```

정상적으로 실해오디면 다음과 같이 나옵니다.

```bash
Create, delete, describe, or change a topic.
Option                                   Description                            
------                                   -----------                            
--alter                                  Alter the number of partitions,        
                                           replica assignment, and/or           
                                           configuration for the topic.         
--at-min-isr-partitions                  if set when describing topics, only    
                                           show partitions whose isr count is   
                                           equal to the configured minimum. Not 
                                           supported with the --zookeeper       
                                           option.                              
...(생략)
```

이제 bin 파일에 있는 명령어를 CLI로 사용할 수 있도록 설정합니다.
사용하고 있는 shell profile 파일에 경로를 추가합니다.

```bash
cd ~/kafka_2.13-2.8.1/bin
echo $(pwd)
```

위 명령어를 통해 출력되는 경로를 아래 명령어의 `<YOUR-KAFKA-BIN-PATH>`를 수정하여서 실행합니다.


```bash
echo 'export PATH="$PATH:<YOUR-KAFKA-BIN-PATH>"' >> ~/.zshrc
source ~/.zshrc
```

다음 명령어가 정상적으로 수행되는지 확인합니다.

```bash
kafka-topics.sh
```

## Docker-compose

다음과 같은 docker-compose.yaml 파일을 작성합니다.

```yaml
version: '3'
services:
  zookeeper:
    container_name: zookeeper
    image: wurstmeister/zookeeper
    ports:
      - "2181:2181"
  kafka:
    image: wurstmeister/kafka:2.13-2.8.1
    ports:
      - "9092:9092"
    environment:
      KAFKA_ADVERTISED_HOST_NAME: localhost
      KAFKA_ZOOKEEPER_CONNECT: zookeeper:2181
    volumes:
      - /var/run/docker.sock:/var/run/docker.sock
```

docker-compose를 실행합니다.

```bash
docker-compose up -d
```

정상적으로 실행되면 다음과 같이 출력됩니다.

```bash
Creating network "kafka_default" with the default driver
Creating kafka_kafka_1 ... done
Creating zookeeper     ... done
```

## Kafka Topic

### Create

테스트 토픽을 생성합니다.

```bash
kafka-topics.sh --create --bootstrap-server localhost:9092 --replication-factor 1 --partitions 1 --topic test
```

정상적으로 수행되면 다음과 같이 출력됩니다.

```bash
Created topic test.
```

### List

생성된 토픽을 확인해 보겠습니다.

```bash
kafka-topics.sh --list --bootstrap-server localhost:9092
```

다음과 같이 출력됩니다.

```bash
test
```

## Message

### 전송

```bash
kafka-console-producer.sh --broker-list localhost:9092 --topic test
```

위 명령어를 입력하면 console창이 실행됩니다.
실행된 콘솔창에서 다음과 같이 입력합니다.

```Bash
> This is test message.
```

입력 후 단축키를 이용해 콘솔창을 종료합니다.

### 읽기

이제 위에서 작성한 메세지를 확인해 보겠습니다.

```bash
kafka-console-consumer.sh --bootstrap-server localhost:9092 --topic test --from-beginning
```

정상적으로 실행되면 다음과 같이 출력됩니다.

```bash
This is test message.
```

단축키를 이용해 콘솔창을 종료합니다.
콘솔창이 종료되면 다음과 같은 메세지가 출력됩니다.

```bash
Processed a total of 1 messages
```
