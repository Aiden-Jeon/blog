---
title: (CI/CD) github action을 이용한 ci
comment: true
categories: [cicd]
toc: true
toc_sticky: true
---

**CI/CD Contents 순서**  
1. [sphinx-autoapi 를 이용한 자동 api 문서 생성하기](https://aiden-jeon.github.io/cicd/sphinx-autoapi)  
2. [github action을 이용한 ci](https://aiden-jeon.github.io/cicd/github-cicd-1)  
3. [ghcr을 이용한 kubernetes deployment 만들기](https://aiden-jeon.github.io/cicd/github-cicd-2)  
4. [helm을 이용한 deployment chart 만들기](https://aiden-jeon.github.io/cicd/github-cicd-3)  
5. [argocd를 이용한 cd](https://aiden-jeon.github.io/cicd/github-cicd-4)  

---



이번 포스트에서는 github action을 이용해 CI(Continuous Integreation)를 하는 법에 대해서 알아보겠습니다.
이번 포스트에서 사용하는 Dockerfile은 [github](https://github.com/Aiden-Jeon/github-cicd) 를 이용합니다.
내용은 [이전 포스트](https://aiden-jeon.github.io/cicd/sphinx-autoapi)를 확인해주세요.


## 1. github package 사용 설정하기
이번 포스트에서는 ghcr(GitHub Container Registry for Docker images) 을 이용할 예정입니다. 개인 repo에서 ghcr을 사용하기 위해서는 따로 설정할 부분이 있습니다.

github 페이지에서 오른쪽 위에 있는 프로필을 누른 후 Feature Preview를 열어줍니다.
![img](/assets/imgs/github/cicd-0.png)

Improved continer support enable을 눌러줍니다. 그러면 아래와 같이 바뀌게 됩니다.
![img](/assets/imgs/github/cicd-1.png)

## 2. secrets 설정해주기
github action 으로 자동으로 push 하기 위해서는 github packages에 접근할 수 있는 key가 필요합니다.

우선 github setting에서 deplotver settings / personal access tokens 로 들어 갑니다.

![img](/assets/imgs/k8s/ghcr/0.png)
여기서 generate new token을 누릅니다.

![img](/assets/imgs/k8s/ghcr/1.png)
토큰 이름으로 ghcr-token을 입력하고 packages와 관련된 권한을 주고 token을 생성합니다.

![img](/assets/imgs/k8s/ghcr/2.png)
다음과 같이 토큰이 생성됩니다. 이 토큰을 메모장에 잘 적어둡니다.

그리고 ci를 진행할 github repo의 settings에 들어갑니다.
secrets를 누른 후 New repository secret을 눌러줍니다.
![img](/assets/imgs/github/cicd-2.png)

Name에는 CR_PAT을 value 에는 위에서 생성한 key를 입력합니다.
![img](/assets/imgs/github/cicd-3.png)

정상적으로 생성되면 아래와 같이 나오게 됩니다.
![img](/assets/imgs/github/cicd-4.png)


## 3. github action 작성하기
이제 준비가 끝났으니 github action을 작성할 차례입니다.
`.github/workflows` 폴더를 생성합니다. 그리고 아래와 같은 파일을 작성합니다. 파일명은 `docker-image-push.yml`으로 하겠습니다.
```yaml
name: Docker Image CI/CD

on:
  push:
    branches: [main]

jobs:
  docker-image-ci:
    runs-on: ubuntu-18.04
    steps:
      - name: Checkout
        uses: actions/checkout@v2
        with:
          submodules: recursive
          fetch-depth: 0
      - name: Create shot SHA
        uses: benjlevesque/short-sha@v1.2
        id: short-sha
      - name: Set up Docker Buildx
        uses: docker/setup-buildx-action@v1
      - name: Login to GitHub Container Registry
        uses: docker/login-action@v1
        with:
          registry: ghcr.io
          username: ${{ github.repository_owner }}
          password: ${{ secrets.CR_PAT }}
      - name: Build and push
        uses: docker/build-push-action@v2
        with:
          push: true
          context: ./
          file: docker/Dockerfile
          tags: |
            ghcr.io/<github-name>/<package-name>:latest
            ghcr.io/<github-name>/<package-name>:${{ steps.short-sha.outputs.sha }}
```

내용에서 수정해야 할 부분은 아래 부분입니다.
```yaml
tags: |
ghcr.io/<github-name>/<package-name>:latest
ghcr.io/<github-name>/<package-name>:${{ steps.short-sha.outputs.sha }}
```
- `github-name`: pacakge를 저장할 github name을 적어줍니다.
- `package-name`: 저장할 package 이름을 적어줍니다.

예를 들어 저는 다음과 같이 적겠습니다.
```yaml
tags: |
ghcr.io/aiden-jeon/sphinx-api:latest
ghcr.io/aiden-jeon/sphinx-api:${{ steps.short-sha.outputs.sha }}
```

이제 작성한 workflow를 github에 push 해줍니다. github에 들어가보면 자동으로 action을 실행합니다.
![img](/assets/imgs/github/cicd-5.png)

package에 가보면 아래과 같이 생성된 걸 확인할 수 있습니다.
![img](/assets/imgs/github/cicd-6.png)


## 4. local ghcr pull
local에서 ghcr을 pull해서 정상적으로 build가 되었는지 확인해보겠습니다.

아래 명령어로 ghcr docker에 로그인합니다.
```bash
export CR_PAT=<MY_TOKEN>
echo $CR_PAT | docker login ghcr.io -u <github-username> --password-stdin
```
- MY_TOKEN: 위에서 발급받은 key를 입력해줍니다.
- github-username: ghcr package가 있는 username을 입력합니다.

로그인을 한후 아까 만든 docker를 pull합니다. pull 명령어는 package를 누르면 확인할 수 있습니다.
![img](/assets/imgs/github/cicd-7.png)

```bash
docker pull ghcr.io/aiden-jeon/sphinx-api:dc3c4be
```

정상적으로 run 되는지 확인해보겠습니다.
```bash
docker run -p 8000:80 --rm ghcr.io/aiden-jeon/sphinx-api:dc3c4be
```
http://localhost:8000 을 들어가면 정상적으로 나오는 것을 확인할 수 있습니다.
