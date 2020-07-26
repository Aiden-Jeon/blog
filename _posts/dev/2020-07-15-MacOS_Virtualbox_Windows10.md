---
title: MacOS에 Virtualbox Windows10 설치하기
comment: true
categories: [dev]
toc: true
toc_sticky: true
---
## 준비
우선 버츄얼 박스와 윈도우10을 다운로드 받습니다.
- [Virtual box 다운로드](https://www.virtualbox.org/wiki/Downloads)
- [Window10 다운로드](https://www.microsoft.com/ko-kr/software-download/windows10ISO)

## 설치하기
Virtualbox를 실행시키면 아래와 같은 화면이 나옵니다.
(2020.07 기준 virutalbox 버전은 6.1.12 입니다.)

![2CB89C19-07FC-43A4-956D-63DF859B4239](/assets/imgs/windows/windows_1.png)

여기서 새로 만들기 버튼을 눌러줍니다.

![C1B4CED5-5CA7-4945-8F03-9CA46F4BF834](/assets/imgs/windows/windows_2.png)

이름에 Windows 10 이라고 입력해줍니다. 그러면 밑의 버전도 이름에 맞추어서 자동으로 변경됩니다.

![58CE287C-919A-4A2C-BDBF-4BCD66D6AC75](/assets/imgs/windows/windows_3.png)

추천 메모리 크기는 기본값인 2048로 둡니다.

![1A563FFA-D88B-4D51-901A-D1E5E8CAB4D3](/assets/imgs/windows/windows_4.png)

계속하기를 누르면 하드 디스크 크기를 설정해주어야 합니다.

![04EC17F0-6A11-4FE4-9CE2-DBF4A18469DE](/assets/imgs/windows/windows_5.png)

VMDK를 골라줍니다.

![20E7DBBF-8382-4792-A9BC-4A9CC323D4DC](/assets/imgs/windows/windows_6.png)



![786AED0C-E1C6-4A57-900F-35219368E990](/assets/imgs/windows/windows_7.png)

저는 32gb 로 설정해주었습니다.

![E4960960-3549-4E34-85A6-8528E19EC706](/assets/imgs/windows/windows_8.png)

설치가 완료되면 다음과 같이 나옵니다.

![F685527D-D908-46AA-8436-DDB0BB6D7B44](/assets/imgs/windows/windows_9.png)



이제 여기서 설정 버튼을 눌러줍니다.

![7CFC4AE9-3401-434F-A842-00E48CBDE776](/assets/imgs/windows/10.png)

우선 시스템에서 광 디스크를 하드 디스크 밑으로 내려 줍니다. 이는 윈도우가 CD로 설치 된 후 하드 디스크로 부팅시키기 위함입니다.

![A59A7058-9541-4CB2-B452-C3DD44EB26C3](/assets/imgs/windows/11.png)

그리고 저장소에서 비어있음을 누르고 빨간 네모가 쳐진 시디 모양의 아이콘을 눌러줍니다.

![B2006D54-592C-4F5B-89F8-021EE17ABC61](/assets/imgs/windows/12.png)

그리고 가상 광학 디스크 선택/만들기를 눌러줍니다.

![A50A569F-843B-426A-A277-2D3384CD817A](/assets/imgs/windows/13.png)

추가를 누르고

![AC69DC40-5E2A-4C4A-8D26-28EE774FC882](/assets/imgs/windows/14.png)

아까 다운로드 받은 windows10.ios 를 선택해줍니다.

![A564F171-514B-419F-98E5-89B2C718C4F6](/assets/imgs/windows/15.png)

그러면 아래와 같이 win10.ios 가 생기게 됩니다. 선택을 누르고 확인을 누르면 됩니다.

![637F6A71-E478-4F48-8BAB-D5303F198285](/assets/imgs/windows/16.png) 

이제 시작을 눌러줍니다.

![1377F26C-A9A3-4AE6-A2C5-009C9B8CFEF5](/assets/imgs/windows/17.png)


## 윈도우 설치
시작을 누르면 윈도우10 설치 과정을 따라 가면 됩니다.

윈도우를 설치하실 때에는 제품 키가 없음을 눌러서 진행하시면 됩니다.

![743C2297-AB10-441A-82B2-B058D5236BD2](/assets/imgs/windows/18.png)

윈도우 버전은 본인의 필요에 따라 선택하면 됩니다. 가장 기본적인 기능은 Windows10 Home 입니다.

![5359A096-9DB4-41F4-A929-CB7B70792E76](/assets/imgs/windows/19.png)

약관에 동의하시고 사용자 지정 설치로 진행합니다.

![3BCF3F77-D130-4CF6-AB61-743CC0DAFFDD](/assets/imgs/windows/20.png)

이후에는 본인의 취향에 맞추어서 설정을 선택하면서 진행하면 됩니다.
