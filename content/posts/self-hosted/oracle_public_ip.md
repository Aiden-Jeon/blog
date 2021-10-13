---
title: Oracle Cloud Public IP 할당 받기
categories: [self-hosted]
tags: ["self-hosted", "oracle"]
toc: true
date: 2021-10-13
author: Jongseob Jeon
---

이번 포스트에서는 Oracle Cloud 무료 버전을 이용해 Public IP를 할당 받는 방법에 대해서 설명합니다.

## 1. 구획 생성하기
1. Compartments 메뉴를 선택합니다.
    ![compartments](/imgs/self-hosted/self-hosted-0.png)
2. Create Compartment를 누른 후 사용할 이름과 설명을 적어줍니다
    ![create compartments](/imgs/self-hosted/self-hosted-1.png)

## 2. Networking
1. Virtual Cloud Networks 메뉴를 선택합니다.
    ![virtual cloud networks](/imgs/self-hosted/self-hosted-2.png)
2. Compartment에서 생성한 구획을 선택하고 Start VCN Wizard 버튼을 눌러줍니다.
    ![start vcn wizard](/imgs/self-hosted/self-hosted-3.png)
3. 다음 항목들을 클릭하고 넘어갑니다.
    ![next vcn wizard](/imgs/self-hosted/self-hosted-4.png)
4. vcn 이름을 작성하고 다음으로 넘어갑니다.
    ![vcn name](/imgs/self-hosted/self-hosted-5.png)
5. 생성을 누릅니다.$$
    ![create vcn](/imgs/self-hosted/self-hosted-6.png)
6. 생성이 완료되면 다음과 같이 나옵니다.
   View Virtual Cloud Network를 선택합니다.
    ![view vcn](/imgs/self-hosted/self-hosted-7.png)
7. 구획 이름과 vcn 이름을 확인합니다.
    ![check vcn](/imgs/self-hosted/self-hosted-8.png)

## 3. Security 
### 3.1 http, https
1. security list를 선택하고 Default Security List를 선택합니다.
    ![security list](/imgs/self-hosted/self-hosted-9.png)
2. Add Ingress Rules를 선택합니다.
    ![add ingress rule](/imgs/self-hosted/self-hosted-10.png)
3. 다음과 같이 http와 https를 위한 포트를 추가합니다.
    ![http https port](/imgs/self-hosted/self-hosted-11.png)
4. 다음과 같이 룰이 추가된 것을 확인합니다.
    ![check http https port](/imgs/self-hosted/self-hosted-12.png)

### 3.2 nginx proxy
1. 다음과 같이 nginx proxy를 위한 81 포트를 추가합니다.
    ngix설정 후 해당 포트는 삭제합니다.
    ![nginx proxy port](/imgs/self-hosted/self-hosted-13.png)
2. 다음과 같이 룰이 추가된 것을 확인합니다.
    ![checkt nginx proxy port](/imgs/self-hosted/self-hosted-14.png)

## 4. Public IP
1. IP Management 메뉴를 선택합니다.
    ![ip management](/imgs/self-hosted/self-hosted-15.png)
2. Public 구획에서 Reserve Public IP Address를 선택합니다.
    ![reserve public ip address](/imgs/self-hosted/self-hosted-16.png)

3. ip 이름을 설정한 후 Reserve Public IP Address를 선택합니다.  
    ![security list](/imgs/self-hosted/self-hosted-17.png)
4. 다음과 같이 IP가 추가된 것을 확인합니다.
    ![security list](/imgs/self-hosted/self-hosted-18.png)
