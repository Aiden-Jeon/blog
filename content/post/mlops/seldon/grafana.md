---
title: prometheus와 grafana 연결하기
categories: [mlops]
tags: ["seldon", "prometheus"]
toc: true
draft: true
date: 2021-05-10
author: Jongseob Jeon
---

이번 포스트에서는 local에서 실행한 seldon-core의 metric을 prometheus의 metric을 grafana에서 확인하는 방벙에 대해서 설명합니다.

# Grafana

grafana를 설치합니다.

```bash
brew install grafana
```

grafana를 실행하기 위해선 아래 명령어를 입력합니다.

```bash
grafana-server --config=/usr/local/etc/grafana/grafana.ini --homepath /usr/local/share/grafana --packaging=brew cfg:default.paths.logs=/usr/local/var/log/grafana cfg:default.paths.data=/usr/local/var/lib/grafana cfg:default.paths.plugins=/usr/local/var/lib/grafana/plugins
```

실행 후  [http://localhost:3000/](http://localhost:3000/) 에 들어갑니다.

![img-1](/imgs/seldon/grafana-1.png)
아이디와 비밀번호는 기본적으로 `admin` / `admin` 입니다.

로그인 후 톱니바퀴에서 data source를 선택합니다.

![img-2](/imgs/seldon/grafana-2.png)

prometheus를 선택합니다.

![img-3](/imgs/seldon/grafana-3.png)

http 의 url에 [http://localhost:9090/](http://localhost:9090/) 를 입력합니다

![img-4](/imgs/seldon/grafana-4.png)

Save & Test를 눌러서 저장합니다.

이제 새로운 대쉬보드를 만듭니다.

![img-5](/imgs/seldon/grafana-5.png)

add an empty panel 을 클릭합니다.

![img-6](/imgs/seldon/grafana-6.png)

데이터 소스를 Prometheus1로 바꿉니다.
![img-7](/imgs/seldon/grafana-7.png)

metric이 활성화됩니다.
![img-8](/imgs/seldon/grafana-8.png)

보고 싶은 metric들을 추가합니다.
![img-9](/imgs/seldon/grafana-9.png)
