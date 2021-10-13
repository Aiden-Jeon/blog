---
title: Oracle Cloud Public IP 사용하기
categories: [self-hosted]
tags: ["self-hosted", "oracle"]
toc: true
date: 2021-10-13
author: Jongseob Jeon
---

이번 포스트에서는 할당 받은 Public IP를 사용하는 방법에 대해서 설명합니다.

## 1. 인스턴스 생성하기
1. Instances 메뉴를 선택합니다.  
    ![instances](/imgs/self-hosted/expose_public_ip-0.png)
2. Create Instance를 선택합니다.
    ![create instance](/imgs/self-hosted/expose_public_ip-1.png)
3. Instances 이름을 입력합니다.
    ![instance name](/imgs/self-hosted/expose_public_ip-2.png)
4. 이미지를 변경합니다.
    ![change image](/imgs/self-hosted/expose_public_ip-3.png)
5. 사용할 이미지는 Canonical Ubuntu 입니다.
    ![select image](/imgs/self-hosted/expose_public_ip-4.png)
6. 저는 ssh key를 자체적으로 발급받고 사용하려고 합니다.
    이를 위해서 Private Key를 저장합니다.
    ![ssh](/imgs/self-hosted/expose_public_ip-5.png)
7. 볼륨은 따로 건들지 않겠습니다.
    ![volume](/imgs/self-hosted/expose_public_ip-6.png)

## 2. Public IP 할당하기
1. 생성된 인스턴스 화면에서 Attached VNCs 메뉴를 선택합니다.
    ![attached vcns](/imgs/self-hosted/expose_public_ip-7.png)
2. 생성한 instance 이름을 선택합니다.
    ![select instance](/imgs/self-hosted/expose_public_ip-8.png)
3. IPv4 Addrresses 를 선택합니다.
    ![ipv4](/imgs/self-hosted/expose_public_ip-9.png)
4. ...을 선택합니다.
    ![more](/imgs/self-hosted/expose_public_ip-10.png)
5. Edit을 선택합니다.
    ![edit](/imgs/self-hosted/expose_public_ip-11.png)
6. No Pulic IP를 선택하고 Update합니다.
    ![no publice ip](/imgs/self-hosted/expose_public_ip-12.png)
7. 다시 edit을 누릅니다.
    ![edit](/imgs/self-hosted/expose_public_ip-13.png)
8. Reserved public IP를 선택하고 생성된 IP를 선택한뒤 Update합니다.
    ![reserved public ip](/imgs/self-hosted/expose_public_ip-14.png)
9. 다음과 같이 IP가 할당된 것을 확인합니다.
    ![check ip](/imgs/self-hosted/expose_public_ip-15.png)

## 3. 접속하기
1. 이미지를 생성할 때 다운로드 받은 key를 `~/.ssh`로 옮깁니다.
    ```bash
    mv ssh-key-2021-10-13.key ~/.ssh
    cd ~/.ssh
    ```
2. 400 권한을 줍니다.
    ```bash
    chmod 400 ssh-key-2021-10-13.key
    ```
3. ssh를 이용해 접속합니다.
    ```bash
    ssh ubuntu@<PUBLIC IP> -i ssh-key-2021-10-13.key
    ```

